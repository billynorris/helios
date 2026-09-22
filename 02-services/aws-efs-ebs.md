---
title: EFS & EBS — Multi-Region
service: efs-ebs
tags: [service, multi-region, efs, ebs, storage, block-storage, nfs, stateful]
status: researched
replication: EFS — native (EFS Replication, AWS-published RPO 15 min); EBS — none, cross-Region snapshot copy only
rpo_achievable: "EFS: 15 min (AWS-published). EBS: ~3-4h realistic — DLM's floor is a 1h interval plus up to 1h of scheduler jitter, so 2h is not reliably achievable"
rto_achievable: "EFS: minutes, if mount targets + access points are pre-provisioned. EBS: fails 15m — restores are lazily hydrated, and FSR is capped at 5 snapshots/Region at $540/AZ/month and does not survive a cross-Region copy"
meets_targets: conditional — EFS meets both; EBS meets neither reliably
ca_west_1: PASS — EFS is in ca-west-1 (Feb 2024) and replication ships with EFS everywhere; EBS/FSR have no parity gap
key_finding: "EFS failover = DeleteReplicationConfiguration; in Terraform, destroying aws_efs_replication_configuration IS the failover"
updated: 2026-09-22
---

# EFS & EBS — Multi-Region

## TL;DR

- **EBS volumes are AZ-scoped. Full stop.** A volume lives in exactly one
  Availability Zone and can only be attached to an instance in that same AZ.
  There is no cross-AZ attach, no cross-Region attach, no "multi-Region EBS"
  feature and no roadmap item for one. **The only mechanism that moves EBS data
  between Regions is the snapshot.** Every design decision in the EBS half of
  this note is downstream of that one sentence.
- **EBS cannot meet RTO 15m for anything meaningful.** A volume created from a
  snapshot is *lazily hydrated* — blocks are pulled from S3 on first touch, so
  the volume is attachable in seconds but slow for minutes-to-hours afterwards.
  Fast Snapshot Restore fixes that, but **FSR is per-snapshot-per-AZ, capped at
  5 snapshots per Region, and costs $0.75/hour — $540/month per snapshot per
  AZ** ([AWS docs](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-fast-snapshot-restore.html#fsr-pricing)).
  And an FSR-enabled snapshot's copy is **not** FSR-enabled, so you pay it again
  in the standby.
- **EBS also misses RPO 2h, which nobody expects.** Data Lifecycle Manager's
  fastest interval is 1 hour, and AWS states runs "start **within one hour of**
  their scheduled time"
  ([docs](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-ami-policy.html#snapshot-considerations)).
  One hour of interval plus up to an hour of jitter plus copy time means the
  loose-looking 2h RPO is not reliably met at DLM's tightest setting. Two hours
  is loose for continuous replication and tight for a batch scheduler.
- **EFS Replication is the good news and AWS publishes a number: "Amazon EFS
  maintains a Recovery Point Objective (RPO) of 15 minutes for most file
  systems"** ([AWS docs](https://docs.aws.amazon.com/efs/latest/ug/efs-replication.html#efs-replication-performance)).
  That clears RPO 2h with 8x headroom. The destination is **read-only until you
  fail over, and "failing over" literally means deleting the replication
  configuration** — an irreversible-feeling API call that is the single most
  surprising thing in this note.
- **`ca-west-1` (Calgary) finally passes one.** AWS states "Replication is
  available in all AWS Regions in which Amazon EFS is available"
  ([AWS docs](https://docs.aws.amazon.com/efs/latest/ug/efs-replication.html)),
  and EFS is available in `ca-west-1`. Calgary has failed five parity checks in
  this vault (Cognito MRR, OpenSearch CCR, Managed Grafana, Backup Audit
  Manager, Origin Shield) and passed for Aurora Global Database; EFS Replication
  is the second pass. See [[region-pair-selection]].
- **The recommendation, stated up front: EBS-backed state should not exist in a
  15-minute-RTO estate.** Push it into managed services ([[aws-rds-postgres]],
  [[aws-elasticache-redis]], [[aws-dynamodb]]) or onto EFS. Where it must exist,
  it should be *derived* state — caches, scratch, build artefacts — that the
  standby can rebuild rather than restore. Full argument in
  [Recommendation](#recommendation-should-ebs-backed-state-exist-at-all).

---

## Does this service cross regions at all?

Two services, two completely different answers, which is why they share a note —
the decision is usually "which of these two do I put this workload on", and
answering that requires both halves side by side.

| | EBS | EFS |
|---|---|---|
| Scope of the resource | **One AZ** | **One Region**, mountable from every AZ in the VPC |
| Attach from another AZ | No | Yes (via a mount target in that AZ) |
| Attach from another Region | No | No — but see cross-VPC/cross-Region mount below |
| Native cross-Region replication | **None** | **Yes — EFS Replication** |
| Cross-Region data movement | Snapshot copy only | Replication, or AWS Backup copy, or DataSync |
| Published RPO | N/A (snapshot cadence) | **15 minutes** |
| Meets RPO 2h | Yes, with DLM | Yes, comfortably |
| Meets RTO 15m | **No** | Conditionally yes |

### EBS: the AZ constraint in detail

An EBS volume is created in a single Availability Zone and
["can only be attached to instances in that Availability Zone"](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volumes.html).
This is not a soft limit or a quota — it is what an EBS volume *is*. The volume
ID is not addressable from outside its AZ, the API call to attach it will fail,
and there is no peering, endpoint, or gateway that changes this.

The knock-on effects for a warm standby:

1. **A pod or instance with an EBS PVC is pinned to an AZ.** If that AZ goes
   away, the pod cannot be rescheduled elsewhere until a volume exists in the
   new AZ — which means a snapshot restore. This is the single biggest
   operational complaint about EBS-backed Kubernetes state and is covered from
   the Kubernetes side in [[eks-stateful-workloads]].
2. **Cross-Region is strictly worse than cross-AZ.** Cross-AZ at least has
   `zone-aware` scheduling and a same-Region snapshot restore (which is fast to
   *initiate*). Cross-Region adds a copy that must be scheduled, must complete,
   and must be tracked.
3. **There is no "warm" EBS volume.** You cannot keep a standby volume
   continuously fed with the primary's writes. The closest thing is a recent
   snapshot copy sitting in the standby Region, which is a cold artefact, not a
   warm resource.

> **The one mechanism.** Cross-Region EBS = `CopySnapshot`. Everything below —
> DLM, AWS Backup, EBS Snapshots Archive, AMI copy — is a wrapper around
> `CopySnapshot` or a scheduler for it. Understanding `CopySnapshot`'s
> behaviour is therefore the whole game.

---

## EBS — cross-Region snapshot copy

### Is a cross-Region copy incremental or full? (AWS's own docs disagree — resolved)

This matters enormously: a full copy of a 500 GiB volume every 2 hours is a
completely different cost and duration profile from a 5 GiB delta. It is also
the question AWS's documentation answers **twice, differently, on the same
page**. Both quotes are from
[Copy an Amazon EBS snapshot](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-copy-snapshot.html):

Under **Considerations for copying snapshots**:

> "If you copy a snapshot to a new Region, a full (non-incremental) copy is
> created. This results in additional storage costs."

Under **Incremental snapshot copying**, a few paragraphs later:

> "When you copy a snapshot across Regions or accounts, the copy is an
> incremental copy if the following conditions are met:
> - The snapshot was copied to the destination Region or account previously.
> - The most recent snapshot copy still exists in the destination Region or account.
> - The most recent snapshot copy has not been archived.
> - All copies of the snapshot in the destination Region or account are either
>   unencrypted or were encrypted using the same KMS key."

**Resolution: the "Considerations" bullet means the *first* copy to a new
Region.** The Incremental section is the operative rule. In steady state,
cross-Region copies **are** incremental, provided you satisfy all four
conditions. The word "new" in "a new Region" is doing load-bearing work that the
bullet does not make obvious, and this bullet has misled more than one capacity
plan.

**What this means in practice, and it is a trap worth naming:**

- The **first** copy of each volume to the standby Region transfers the entire
  used size of the volume. Budget for it, schedule it off-peak, and do not do it
  for 400 volumes on the same afternoon.
- Steady-state copies are deltas — cheap and fast.
- **Any of the four conditions breaking silently reverts you to full copies.**
  The most common way this happens: a lifecycle/retention policy in the standby
  Region deletes the previous copy before the next one lands, so condition 2
  fails, so the next copy is full, so it takes longer, so it overruns the
  window, and now your RPO is blown *and* your bill tripled. **Retention in the
  destination must always be longer than the copy interval.** This is the
  single highest-value sentence in the EBS half of this note.
- **Archiving to EBS Snapshots Archive breaks incrementality** (condition 3)
  and restoring from archive takes up to 72 hours. Never archive the standby
  Region's most recent copy.
- **Re-keying breaks incrementality** (condition 4): "if you encrypt the
  snapshot copy using a different KMS key, the copy is a full copy." See
  [KMS](#ebs--kms-on-cross-region-copy) below — this collides directly with the
  fact that a single-Region KMS key does not exist in the standby Region.

**How to verify it is actually happening:** AWS provides a CloudWatch event.
"To see whether your snapshot copies are incremental, check the `copySnapshot`
CloudWatch event"
([docs](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-copy-snapshot.html#ebs-incremental-copy)).
Alarm on this. A copy that has silently gone full is invisible until the bill
arrives or the window overruns. Cross-ref [[observability-multi-region]].

### Copy duration is the RPO lever

AWS does not publish a copy-duration SLA. The default is best-effort. The one
lever AWS *does* give you is **time-based copies**: when copying, you can
specify a completion duration "in 15-minute increments" and AWS will prioritise
the copy to hit it
([docs](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-copy-snapshot.html)).
Without it, "the snapshot copy is completed on a best-effort basis" — AWS's own
words, and "best-effort" is not a thing you can put in an RPO commitment.

**This is the correct lever for this estate and it should be switched on.** An
unbounded best-effort copy makes the RPO arithmetic below unanswerable; a copy
with a declared completion duration makes it arithmetic again. Set the duration
to a value comfortably inside your snapshot interval.

### Concurrent copy limits

> "There is a limit of `20` concurrent snapshot copy requests per destination.
> If you exceed this quota, you receive a `ResourceLimitExceeded` error."
> — [AWS docs](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-copy-snapshot.html#snapshot-copy-consids)

Note the wording: **per *destination***, not per source. Three primary Regions
all copying into the same standby would contend. In this estate each pair has a
dedicated standby, so the practical reading is 20 concurrent copies into
`eu-west-2`, 20 into `us-west-2`, 20 into `ca-west-1`.

**Do the arithmetic before committing to a schedule.** If the estate has 60
EBS volumes to protect and copies take ~10 minutes each, 20-at-a-time means 3
waves ≈ 30 minutes. Fine for a 2h RPO. If it has 400 volumes, 20 waves at 10
minutes is 200 minutes — **RPO 2h is already blown by queueing alone**, before
any copy is slow. This is the arithmetic that decides whether EBS state is
viable at all, and it is why the recommendation at the end of this note is what
it is.

### Other named gotchas on copy

- **"User-defined tags are not copied from the source snapshot to the snapshot
  copy."** ([docs](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-copy-snapshot.html#snapshot-copy-consids))
  If your standby's automation, cost allocation, or retention logic keys off
  tags — and in a cookiecutter monorepo it certainly does — the copies arrive
  **untagged**. You must re-apply tags during the copy call. DLM does this for
  you; a hand-rolled Lambda does not unless you remember.
- **Copies get an arbitrary volume ID** such as `vol-ffffffff`. Do not build
  anything that correlates copies back to source volumes by volume ID. AWS's own
  advice: "We recommend that you tag your snapshot copies with the volume ID and
  creation time so that you can keep track of the most recent snapshot copy of a
  volume in the destination Region."
- **FSR does not survive a copy.** "If you copy a snapshot that is enabled for
  fast snapshot restore, the snapshot copy is not automatically enabled for fast
  snapshot restore. You must explicitly enable fast snapshot restore for the
  snapshot copy." This is the sentence that makes the FSR cost double.
- **Cross-Region data transfer charges apply**, and "if you delete any snapshots
  after initiation, you are still charged for the data that has already been
  transferred."
- **Encrypted-copy failures are silent.** "if you attempt to copy an encrypted
  snapshot without having permissions to use the encryption key, the operation
  fails silently and the snapshot copy receives the 'Given key ID is not
  accessible' status message." Silent failure + a status message nobody reads =
  a standby that is quietly months out of date. Alarm on snapshot age in the
  destination Region, not on copy-job success.

---

## EBS — KMS on cross-Region copy

Deep background in [[aws-kms]] and [[kms-when-to-use-multi-region-keys]]; this
section covers only what is EBS-specific.

The structural problem: **a single-Region KMS key does not exist in the standby
Region**, so an encrypted snapshot copied cross-Region must be re-encrypted
under a key that does. That re-encryption is not optional and it has a cost:
per condition 4 above, **changing the KMS key makes the copy full, not
incremental**.

That produces a genuine fork:

| | Option A: separate per-Region CMKs | Option B: multi-Region KMS key |
|---|---|---|
| Copy incrementality | **Broken** — every copy re-keys, so every copy is full | **Preserved** — same key ID material both sides, copies stay incremental |
| Blast radius | Smaller — compromise of one Region's key doesn't cross | Larger — same key material in both Regions |
| Terraform | Two `aws_kms_key` resources, one per provider alias | One primary + `aws_kms_replica_key` |
| Cost | Two keys, plus full-copy transfer and storage every cycle | Two key charges, incremental transfer |
| Compliance posture | Easier story for "keys never leave the Region" | Needs an argument for the auditor |

**Recommendation: Option B, a multi-Region key, specifically for EBS snapshot
copy.** The incrementality argument is decisive — Option A means paying full
transfer and full storage on every single copy cycle forever, which for an
hourly cadence is unaffordable and for a 2-hourly cadence is merely bad. This
is one of the clearest cases in the estate for a multi-Region key and should be
recorded as such in [[kms-when-to-use-multi-region-keys]]. Note this conclusion
is EBS-specific and does not automatically generalise to other services.

**Required IAM/KMS permissions on the copying principal** (from the
[copy docs](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-copy-snapshot.html#creating-encrypted-snapshots)):
`kms:DescribeKey`, `kms:CreateGrant`, `kms:GenerateDataKey`,
`kms:GenerateDataKeyWithoutPlaintext`, `kms:ReEncrypt`, `kms:Decrypt`.

**`kms:CreateGrant` is the one that bites.** Grants are regional and do not
replicate. A key policy that works in the primary will not produce a working
grant in the standby unless the standby key's policy also permits the EBS
service principal and the DLM role to create grants. Symptom: the silent copy
failure described above.

---

## EBS — restore time, lazy hydration, and FSR

### Lazy hydration: the RTO trap

A volume created from a snapshot is available almost immediately — but it is
**not hydrated**. Blocks live in S3 and are fetched on first access. AWS's own
framing of what FSR fixes:

> "Amazon EBS fast snapshot restore (FSR) enables you to create a volume from a
> snapshot that is fully initialized at creation. This eliminates the latency of
> I/O operations on a block when it is accessed for the first time. Volumes that
> are created using fast snapshot restore instantly deliver all of their
> provisioned performance."
> — [AWS docs](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-fast-snapshot-restore.html)

Read that inverted and you have the trap: **without FSR, a restored volume does
not deliver its provisioned performance until every block has been touched
once.** The `terraform apply` succeeds, the instance boots, the pod goes Ready,
the health check passes — and the database on that volume is crawling, because
every page fault is a round trip to S3. Your dashboards say the failover worked.
Your users say it didn't.

This is the classic way an EBS-based DR plan passes a tabletop exercise and
fails a real one, and it is why [[dr-testing-and-gamedays]] must measure
*application* latency after failover, not just resource existence.

### FSR: the fix, and why it is usually unaffordable

From the [FSR considerations](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-fast-snapshot-restore.html#fsr-considerations):

- "Each snapshot and Availability Zone pair refers to one fast snapshot restore."
  — the unit is **snapshot × AZ**, so covering 3 AZs costs 3×.
- "You can enable up to **5 snapshots for fast snapshot restore per Region**."
  A hard quota, and it counts snapshots shared with you. **Five.** That number
  alone disqualifies FSR as a general-purpose answer for an estate with dozens
  of stateful volumes.
- "Fast snapshot restore can be enabled on snapshots with a size of 16 TiB or less."
- FSR is not supported on Outposts, Local Zones, or Wavelength Zones.
- Volumes above 64,000 IOPS or 1,000 MiB/s don't get the full benefit and should
  be explicitly initialized anyway.
- The number of volumes you can restore at full benefit is governed by **volume
  creation credits** — FSR is not an unlimited tap; restore many volumes at once
  and later ones fall back to lazy hydration.

**Pricing, quoted exactly** from
[Pricing and Billing](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-fast-snapshot-restore.html#fsr-pricing):

> "You are billed for each minute that fast snapshot restore is enabled for a
> snapshot in a particular Availability Zone. Charges are pro-rated with a
> minimum of one hour.
>
> For example, if you enable fast snapshot restore for one snapshot in
> `US-East-1a` for one month (30 days), you are billed **$540** (1 snapshot x 1
> AZ x 720 hours x **$0.75 per hour**). If you enable fast snapshot restore for
> two snapshots in `us-east-1a`, `us-east-1b`, and `us-east-1c` for the same
> period, you are billed **$3240** (2 snapshots x 3 AZs x 720 hours x $0.75 per
> hour)."

**Work the number for this estate.** Warm standby, 3 AZs in the standby Region,
and you want FSR on your standby copies so the restore is actually fast:

| Snapshots with FSR | AZs | Monthly cost (at $0.75/hr) |
|---|---|---|
| 1 | 1 | $540 |
| 1 | 3 | $1,620 |
| 5 (the per-Region cap) | 3 | **$8,100** |

$8,100/month **per standby Region**, and that is the *maximum* you are allowed
to buy — five snapshots. Across three standby Regions that is $24,300/month to
accelerate at most 15 snapshots. Feed that into [[cost-model]].

**And it is worse than it looks**, for three compounding reasons:

1. FSR must be enabled on the **copy in the standby Region**, not the source —
   copies don't inherit it. So this is net-new spend, not a re-use.
2. FSR is enabled on **a specific snapshot**. Your standby's newest snapshot
   changes every copy cycle. To keep FSR on "the latest snapshot" you must
   enable FSR on each new copy and disable it on the old one — and
   **enabling FSR is not instantaneous**. AWS publishes the rate:

   > "When fast snapshot restore is enabled for a snapshot, **it takes 60
   > minutes per TiB to optimize the snapshot.** We recommend that you configure
   > your schedules so that each snapshot is fully optimized before Amazon Data
   > Lifecycle Manager creates the next snapshot."
   > — [DLM snapshot policy considerations, AWS docs](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-ami-policy.html#snapshot-considerations)

   **Do that arithmetic and FSR collapses for this estate.** 60 min/TiB means a
   1 TiB volume's snapshot takes an hour to optimize. If your cross-Region copy
   cadence is hourly (which the RPO section below argues for), a 1 TiB volume's
   newest snapshot is *never* fully optimized before it is superseded — you are
   paying $540/AZ/month for a snapshot that is permanently mid-optimization.
   **FSR and a tight snapshot cadence are mutually exclusive above about 500
   GiB.** AWS's recommendation is explicitly to slow the schedule down to fit
   the optimization — i.e. to trade RPO for RTO. You cannot have both.
3. Because of (2), the cost is continuous. This is not "pay during the
   disaster" — you pay $540/snapshot/AZ/month permanently for a capability you
   hope never to use.
4. **FSR leaks cost after you stop using it.** "A snapshot that is enabled for
   fast snapshot restore remains enabled even if you delete or disable the
   policy, disable fast snapshot restore for the policy, or disable fast
   snapshot restore for the Availability Zone. **You must disable fast snapshot
   restore for these snapshots manually.**" Tearing down a DLM policy in
   Terraform does *not* stop the $540/month. Add an explicit teardown step to
   any runbook or module that enables FSR, and put a Cost Anomaly Detection
   monitor on the EBS FSR usage type. See [[cost-model]].
5. **Exceeding the cap fails silently, again.** "If you enable fast snapshot
   restore for a policy and you exceed the maximum number of snapshots that can
   be enabled for fast snapshot restore, Amazon Data Lifecycle Manager creates
   snapshots as scheduled but **does not enable them for fast snapshot
   restore**." No error, no alarm — just a DR plan that has quietly reverted to
   lazy hydration. Alarm on the `EBS fast snapshot restore` CloudWatch state
   events that AWS emits when FSR state changes.

**Verdict on FSR: it is a targeted tool, not a DR strategy.** Legitimate use is
one or two genuinely critical, genuinely irreplaceable volumes. Using it to make
a general EBS-backed estate meet RTO 15m is not financially or numerically
possible — the 5-per-Region cap settles it before the price does.

---

## EBS — Data Lifecycle Manager and the RPO 2h arithmetic

DLM is the native scheduler for "snapshot these volumes on a cadence and copy
the snapshots to another Region." Prefer it over a hand-rolled EventBridge +
Lambda for two reasons: it re-applies tags to copies (which raw `CopySnapshot`
does not), and AWS charges nothing for it — "Amazon Data Lifecycle Manager
provides a complete backup solution for Amazon EC2 instances and EBS volumes
**at no additional cost**"
([docs](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-lifecycle.html)).

### The shape of a DLM policy

Verified against
[Create a DLM custom policy for EBS snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-ami-policy.html):

| Thing | Limit / values |
|---|---|
| Schedules per policy | **4** (schedule 1 mandatory, 2–4 optional) |
| Interval values (`IntervalUnit: HOURS`) | **1, 2, 3, 4, 6, 8, 12, 24** — 1-hour added [March 2020](https://aws.amazon.com/about-aws/whats-new/2020/03/amazon-data-lifecycle-manager-adds-support-for-1-hour-backup-interval) |
| Cron expressions | Supported, "an interval of up to one year" |
| Cross-Region copy destinations per schedule | **3** ("up to three additional Regions or Outposts"), one copy rule each |
| Custom lifecycle policies per Region | 100 |
| Retention (count-based) | 1–1000 |
| Retention (age-based) | 1 day – 100 years |
| Retention when cross-Region copy is on | Must be **≥1**, or ≥1 day |
| Targeting | Tag-based, **same Region as the policy only** |

Two structural notes for a templated monorepo. First, **"All schedules must have
the same retention type (age-based or count-based). You can specify the
retention type for Schedule 1 only. Schedules 2, 3, and 4 inherit the retention
type from Schedule 1."** Your module's variable surface has to reflect that —
retention *type* is a policy-level input, retention *value* is per-schedule.
Second, **DLM targets by tag and only in its own Region**, so the policy is a
primary-Region resource that reaches into the standby; it is not a resource you
instantiate twice behind a provider alias.

### The ±1 hour that breaks the arithmetic

This is the finding that decides the EBS half of this note, and it is buried in
a considerations bullet:

> "The first snapshot creation operation starts **within one hour after** the
> specified start time. Subsequent snapshot creation operations start **within
> one hour of** their scheduled time."
> — [DLM considerations, AWS docs](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-ami-policy.html#snapshot-considerations)

DLM does not fire on time. It fires within an hour of its scheduled time, and
AWS publishes no tighter bound. So the real RPO is:

```
RPO = snapshot interval
    + up to 1h  DLM scheduling jitter        <-- uncontrollable, AWS's scheduler
    + time for the snapshot to reach `completed`
    + queueing behind the 20-concurrent-copy-per-destination limit
    + cross-Region copy duration
```

**Work it for a 1-hour interval — the tightest DLM offers:**

| Component | Best case | Worst case |
|---|---|---|
| Interval since last successful snapshot | 1h | 1h |
| DLM scheduling jitter | ~0 | **1h** |
| Snapshot reaches `completed` | minutes | tens of minutes |
| Copy queueing (20 concurrent per destination) | 0 | grows with volume count |
| Cross-Region copy | minutes (incremental) | bounded only if time-based copy is on |
| **Total** | **~1h 10m** | **comfortably over 2h** |

**Conclusion: DLM at its fastest setting does not reliably meet RPO 2h for EBS,
and there is no faster setting.** The interval floor is 1 hour and the jitter is
another hour on top of it. You can shave the tail with time-based copies, but
you cannot shave the jitter — it is AWS's scheduler, not yours.

This is a genuinely surprising result worth stating plainly, because "RPO 2h is
the loose target, surely block snapshots are fine" is the intuition everyone
arrives with. They are not fine. **Two hours is loose for continuous replication
and tight for a batch scheduler**, and DLM is a batch scheduler.

**If you must have EBS snapshots inside a 2h RPO, the options are:**

| Option | Mechanism | Verdict |
|---|---|---|
| A. DLM, 1h interval | Native, free, tag-driven | **Does not reliably meet 2h.** Meets 3–4h comfortably. |
| B. EventBridge Scheduler + Lambda calling `CreateSnapshot` / `CopySnapshot` | You own the cron, so no jitter; sub-hour intervals possible | Meets 2h. Costs you tag propagation, concurrency limiting, retention reaping and failure alerting — all of which DLM gave you free. |
| C. AWS Backup with cross-Region copy | See [[aws-backup]] | Same class of scheduler problem, and AWS Backup is explicitly not the RTO mechanism. |
| D. **Stop replicating block storage** | Move the state to EFS, RDS, DynamoDB or S3 | **Recommended.** See [Recommendation](#recommendation-should-ebs-backed-state-exist-at-all). |

**Recommendation: D, and B only where D is genuinely impossible.** Option B is
the classic "we rebuilt DLM, badly" trap — you re-implement four things DLM does
correctly, purely to remove an hour of jitter from a mechanism that *still*
cannot meet the 15-minute RTO. If a volume's data matters enough to need a
sub-2h RPO, it matters enough not to live on a lone EBS volume.

Where 2h is genuinely not required for a given volume — scratch, caches, CI
workspaces, log spool — **use DLM at a 12- or 24-hour interval and stop worrying
about it.** That is the right answer for most EBS volumes in most estates, and
saying so out loud saves a lot of pointless engineering.

### Other DLM behaviours that matter here

- **Tag copying is a per-copy-rule boolean.** For each destination "you can
  choose whether to copy all tags or no tags" — all or nothing, no selection.
  Given that raw `CopySnapshot` copies none, set it to all.
- **Cross-Region copies are never archived by DLM.** "Snapshots must be archived
  in the same Region in which they were created. If you enabled cross-Region
  copy and snapshot archiving, Amazon Data Lifecycle Manager does not archive
  the snapshot copy." This is good news — DLM will not accidentally break your
  standby's copy incrementality by archiving the latest copy. Manual archiving
  still can.
- **Archiving requires a ≥28-day creation frequency and a 90-day minimum archive
  retention**, and "when a snapshot is archived, it is converted to a full
  snapshot." Archive is a retention/compliance tool, not a DR tool. Keep it away
  from the standby's recent copies.
- **A policy in the `error` state stops deleting (count-based) or retains
  indefinitely (age-based).** Either way you accumulate cost silently. Alarm on
  DLM policy state and on the `SnapshotsCreateFailed` CloudWatch metric.
- **Deleting the source volume orphans its snapshots.** "If you delete a volume
  or terminate an instance targeted by a policy with a count-based retention
  schedule, Amazon Data Lifecycle Manager no longer manages snapshots … You must
  manually delete those earlier snapshots." In an estate that recycles nodes
  constantly — which an EKS estate does — this is a steady cost leak in *both*
  Regions.
- **Snapshots are crash-consistent, not application-consistent**, unless you use
  DLM's pre/post scripts. For a database on an EBS volume that distinction is
  the difference between a restore and a recovery.

---

## EFS — Replication

### Published RPO, quoted

> "After the initial replication is finished, Amazon EFS maintains a Recovery
> Point Objective (RPO) of 15 minutes for most file systems. However, if the
> source file system has files that change very frequently and has either more
> than 100 million files or files that are larger than 100 GB, replication may
> take longer than 15 minutes."
> — [Replication performance, AWS docs](https://docs.aws.amazon.com/efs/latest/ug/efs-replication.html#efs-replication-performance)

**15 minutes against a 2-hour target is 8x headroom.** EFS Replication is one of
the few services in this estate that clears its RPO without any tuning at all.

**But read the exception carefully and check it against your actual data.** The
qualifier is a conjunction of conditions: files that change *very frequently*
**and** (>100 million files **or** files >100 GB). A media-processing or
data-science workload with a few huge, frequently-rewritten files can land in
the exception. A workload with a million small config files that rarely change
does not. Measure before you assume — the `TimeSinceLastSync` CloudWatch metric
is the ground truth and must be alarmed. AWS points at it explicitly.

### Published RTO

**AWS does not publish an RTO figure on the replication documentation pages.**
The pages describe the failover *mechanism* but give no time bound for it. Do
not let anyone cite "EFS RTO is X minutes" without a source — as of this
research, no such published figure was found on the EFS replication or failover
pages. What *is* documented is that failover is a single control-plane
operation (delete the replication configuration), which is fast; the RTO is
therefore dominated by your own orchestration and by mount-target availability,
not by EFS.

**Write it in the note as a finding, not a gap: AWS publishes an RPO for EFS
Replication and does not publish an RTO.** That asymmetry is itself informative
— it tells you AWS considers the failover a control-plane operation whose
duration is your problem, not theirs. Plan the RTO from the parts you control
(see [Warm standby shape](#efs--warm-standby-shape-and-the-rto-15m-question)).

### Consistency model — not point-in-time

> "Changes made to the source file system are **not transferred to the
> destination file system in a point-in-time consistent manner**. Instead they're
> transferred based on the **Last synced time** for the replication."
> — [AWS docs](https://docs.aws.amazon.com/efs/latest/ug/efs-replication.html)

This is important and under-appreciated. The replica is **not a crash-consistent
snapshot of the source at an instant**. Files written close together are not
guaranteed to arrive together. If your application writes a data file and then a
marker/index file that references it, the replica can legitimately contain the
marker without the data. Applications that assume write-ordering across files
must be checked. This is a correctness concern, not a performance one, and it
belongs in the failover runbook's validation step.

### The destination is read-only, and failover means *deleting* the config

> "In the event of a disaster or when performing game day exercises, you can
> fail over to your replica file system **by deleting its replication
> configuration**. After the replication configuration is deleted, the replica
> becomes writeable."
> — [Using the replica, AWS docs](https://docs.aws.amazon.com/efs/latest/ug/replication-fail-over.html)

The API call is `DeleteReplicationConfiguration`. Three consequences that need
to be in the runbook:

1. **The failover action looks destructive.** At 3am, under pressure, the person
   holding the runbook is being asked to run a `delete` against the DR
   configuration. This *is* the correct action, but it reads like the wrong one.
   Write it in the runbook in exactly those words, with the reassurance that it
   deletes the *configuration*, not the file system or its data. See
   [[failover-orchestration]].
2. **It is one-way in the moment.** Once deleted, the replica is writeable and
   divergence from the primary begins. There is no "pause" — EFS Replication has
   no read-write-but-still-replicating mode. Failing over is a commitment.
3. **Terraform will fight you.** The replication configuration is a Terraform
   resource. Deleting it out-of-band during a failover means the next
   `terraform plan` wants to recreate it — which would immediately start
   replicating the *stale primary* over your now-live standby. See
   [Terraform](#terraform-implementation).

### Failback — reversible, but by recreate, not by reverse

This is the hard part and AWS's documentation on it is short. Quoted in full
because the detail matters:

> "During the failback process, you can choose to discard the changes made to
> your replica file system or preserve them by copying them back to your primary.
> - To **discard** the changes made to your replica during failover, re-create
>   the original replication configuration on your primary file system, where
>   the replica file system is the replication destination. During replication,
>   Amazon EFS synchronizes the file systems by updating your replica file
>   system's data to match that of your primary.
> - To **replicate** the changes made to your replica during failover, create a
>   replication configuration on the replica file system, where the primary file
>   system is the replication destination. During replication, Amazon EFS
>   identifies and transfers the differences from your replica file system back
>   to the primary file system. Once the replication is complete, you can resume
>   replicating the primary file system by re-creating the original replication
>   configuration or creating a new configuration."
> — [AWS docs](https://docs.aws.amazon.com/efs/latest/ug/replication-fail-over.html)

**Answering the question directly: replication direction cannot be "reversed" in
place. There is no reverse or swap API. You delete one replication
configuration and create another pointing the other way.** Practically that is
reversal, and AWS's own performance page acknowledges it as a first-class
scenario ("When you create new replications **or reverse the direction of
existing replications during the failback process**, Amazon EFS performs an
initial sync"). But it is delete-and-create at the API and Terraform level, not
an attribute flip.

**The expensive detail is that word "initial sync":**

> "When you create new replications or reverse the direction of existing
> replications during the failback process, Amazon EFS performs an initial sync,
> which includes a series of one-time setup actions to support the replication.
> **Replicated data is accessible in the destination file system only after the
> initial sync completes.** The amount of time that the initial sync takes to
> finish depends on factors such as the size of the source file system and the
> number of files in it."
> — [AWS docs](https://docs.aws.amazon.com/efs/latest/ug/efs-replication.html#efs-replication-performance)

So the failback sequence is: reverse-replicate standby → primary (initial sync,
duration proportional to file system size and file count, not published), cut
traffic back, then reverse *again* to restore the original direction (**another**
initial sync). **Two full initial syncs to complete a round trip.** For a large
file system this is the multi-hour part of the DR lifecycle, and it is why
failback should be planned as a scheduled maintenance event rather than an
emergency action. Nothing in the 15-minute RTO applies to failback.

AWS does not publish initial-sync throughput figures. Measure yours in a
gameday ([[dr-testing-and-gamedays]]) and record the number — it is the input
to every future failback decision, and nobody else can give it to you.

---

## EFS — `ca-west-1` parity check (PASS)

Calgary has failed five parity checks elsewhere in this vault and passed one.
**EFS Replication is the second pass.** Two facts chain together:

1. **EFS itself is in `ca-west-1`.** AWS announced it on 2024-02-05:
   [Amazon EFS is now available in the AWS Canada West (Calgary) region](https://aws.amazon.com/about-aws/whats-new/2024/02/amazon-efs-aws-canada-west-calgary-region/).
2. **Replication follows EFS everywhere.** "Replication is available in **all
   AWS Regions in which Amazon EFS is available**."
   ([docs](https://docs.aws.amazon.com/efs/latest/ug/efs-replication.html))

Unlike Cognito MRR, OpenSearch CCR or Origin Shield, EFS Replication is not a
separately-rolled-out feature with its own region list. It ships with EFS. That
makes this a durable pass rather than a "true today" pass, and it is worth
recording that distinction in [[region-pair-selection]] — some of Calgary's
failures are timing, this one cannot regress without EFS itself regressing.

**EBS has no parity question at all.** EBS, snapshots, snapshot copy and DLM are
universal. FSR is excluded only on Outposts, Local Zones and Wavelength Zones —
not on any commercial Region. The CA pair has no EBS-specific gap.

> Not verified: whether `ca-west-1` has the same *default* EFS throughput quota
> ceilings as `ca-central-1`. AWS documents per-Region throughput maximums in
> [Amazon EFS quotas](https://docs.aws.amazon.com/efs/latest/ug/limits.html) and
> says throughput above a Region's maximum "requires a throughput quota
> increase" considered "on a case-by-case basis". A young Region plausibly has
> lower ceilings. Left as an open question below rather than guessed at.

---

## EFS — mount targets in the standby

A file system is Regional; **access to it is not.** Access is via mount targets,
and mount targets are the thing that must be pre-provisioned in the standby.

From [Managing mount targets](https://docs.aws.amazon.com/efs/latest/ug/accessing-fs.html):

- "For EFS file systems that use Regional storage classes, you can create a
  mount target in **each Availability Zone in an AWS Region**."
- "You can create **only one mount target per Availability Zone**. If an
  Availability Zone has multiple subnets … you create a mount target in only one
  of the subnets. As long as you have one mount target in an Availability Zone,
  the EC2 instances launched in any of its subnets can share the same mount
  target."
- A mount target lives in a **VPC**. "If you want to modify the VPC for mount
  targets, then you need to first delete the existing mount targets."
- "You can't change the IP address of an existing mount target. To change an IP
  address, you need to delete the mount target and create a new one."

**The consequences for this estate:**

1. **Mount targets are per-AZ *and* per-VPC, and the standby has its own VPC, so
   the standby needs its own mount targets.** They are not replicated by EFS
   Replication — replication copies data, not network plumbing. The destination
   file system created by a replication configuration can be created *with* mount
   targets, but they are yours to define. Cross-ref [[aws-vpc-networking]].
2. **Mount targets are cheap and must be pre-provisioned.** AWS does not charge
   for a mount target itself; it consumes an ENI and an IP in the chosen subnet.
   There is no reason to create them at failover time and every reason not to —
   creating one is a control-plane operation with a propagation delay, and the
   RTO budget is 15 minutes. **Create all three standby mount targets on day
   one.** This is the single cheapest RTO win in the whole note.
3. **Security groups are attached to the mount target, not the file system.**
   The standby's mount-target SGs must already allow NFS (TCP 2049) from the
   standby's node/pod security groups. A standby with mount targets but no
   ingress rule is a standby that fails at 3am with a hang, not an error — NFS
   mounts do not fail fast on a blocked port, they retry. Bake the SG rule into
   the module and test it in a gameday ([[dr-testing-and-gamedays]]).
4. **You cannot mount an EFS file system from another Region over the public
   internet**, and cross-VPC mounting requires VPC peering or Transit Gateway
   plus DNS handling — the `fs-xxxx.efs.<region>.amazonaws.com` name resolves
   via the VPC's DNS. In an active/passive design you should not want this
   anyway: the standby mounts the *replica*, not the primary. Anything that
   mounts across the Region boundary has made the standby dependent on the
   primary being alive, which defeats the entire exercise. See
   [[cross-region-connectivity]] for the cases where cross-Region NFS genuinely
   is wanted (it is mostly migration, not DR).
5. **A One Zone replica has exactly one mount target**, in one AZ. That is the
   cost-saving option discussed below, and this is its price: the standby loses
   AZ redundancy.

---

## EFS — throughput modes and the cold-standby burst-credit question

This is the question that sounds like it should be a problem and, when you do
the arithmetic, is a problem **only in one specific configuration**. Worth
working through, because the wrong conclusion in either direction costs money.

### The three modes

From [Amazon EFS performance specifications](https://docs.aws.amazon.com/efs/latest/ug/performance.html):

| Mode | How throughput is determined | Burst credits? | Per-file-system read / write max |
|---|---|---|---|
| **Elastic** (default, AWS-recommended) | Scales automatically; "you pay only for the amount of metadata and data read or written" | **"you don't accrue or consume burst credits while using Elastic throughput"** | 20–60 GiBps read / 1–5 GiBps write |
| **Provisioned** | You declare a MiBps figure, billed on provisioned-in-excess-of-baseline | Falls back to Bursting if baseline exceeds provisioned | 3–10 GiBps read / 1–3.33 GiBps write |
| **Bursting** | Scales with *stored* data | **Yes** | 3–5 GiBps read / 1–3 GiBps write |

AWS's own guidance: use Elastic "when you have spiky or unpredictable workloads
… or when your application drives throughput at an average-to-peak ratio of 5%
or less"; use Provisioned "if you know your workload's performance requirements,
or when your application drives throughput at an average-to-peak ratio of 5% or
more."

**A warm standby that does nothing and then suddenly does everything is the
textbook definition of "spiky and unpredictable", average-to-peak ratio
approximately zero.** By AWS's own decision rule, the standby should be Elastic.

### The burst-credit arithmetic, since the question was asked

For Bursting mode, quoting
[Understanding Amazon EFS burst credits](https://docs.aws.amazon.com/efs/latest/ug/performance.html#efs-burst-credits):

- "the base throughput is proportionate to the file system's size in the
  **Standard storage class**, at a rate of **50 KiBps per each GiB** of storage."
- "When burst credits are available, a file system can drive throughput up to
  **100 MiBps per TiB** in Standard storage … with a minimum of 100 MiBps. If no
  burst credits are available, a file system can drive up to **50 MiBps per
  TiB** of storage, with a minimum of 1 MiBps."
- "A file system accumulates burst credits whenever it is **inactive** or
  driving throughput below its baseline metered rate."
- "File systems can earn credits up to a maximum credit balance of **2.1 TiB**
  for file systems smaller than 1 TiB, or 2.1 TiB per TiB stored for file
  systems larger than 1 TiB. This behavior means that file systems can
  accumulate enough credits to **burst for up to 12 hours continuously**."

**So the intuitive fear is backwards.** An idle standby is not credit-starved —
it is credit-*saturated*. It has been inactive since the day it was created, so
it sits pegged at the maximum balance, which AWS says is enough to burst
continuously for 12 hours. Twelve hours of full burst is far more than a
15-minute RTO needs, and more than enough to cover the window in which you would
either restore the primary or provision more throughput.

AWS's worked example, quoted: "a file system with 100 GiB of metered data in the
Standard storage class has a baseline throughput of 5 MiBps. Over a 24-hour
period of inactivity, the file system earns 432,000 MiB worth of credit (5 MiB ×
86,400 seconds = 432,000 MiB), which can be used to burst at 100 MiBps for 72
minutes."

**But look closely at that example and the actual trap appears.** Baseline and
burst are both proportional to **size in the Standard storage class**
(`ValueInStandard` from `DescribeFileSystems`). So:

> **The real cold-standby throughput trap is not burst credits. It is the
> lifecycle policy.** If the standby file system has a lifecycle policy that has
> quietly moved 95% of its data to IA or Archive, then `ValueInStandard` is 5%
> of what you think it is, and a Bursting-mode standby has 5% of the baseline
> throughput you sized for — with a floor of 1 MiBps. Your credits are full and
> irrelevant; you will exhaust them and then fall off a cliff onto a baseline
> computed from almost no Standard data.

That is a real, non-obvious failure mode, it is specific to the
cold-standby-with-lifecycle-policy combination, and it is created by exactly the
cost optimisation AWS recommends for DR replicas (below). **Elastic throughput
removes it entirely**, because Elastic does not derive throughput from stored
size at all.

### The 24-hour lock-in

> "after switching the throughput mode to Provisioned throughput or changing the
> Provisioned throughput amount, the following actions are restricted for a
> **24-hour period**: switching from Provisioned throughput mode to Elastic or
> Bursting throughput mode; decreasing the Provisioned throughput amount."
> — [AWS docs](https://docs.aws.amazon.com/efs/latest/ug/performance.html#switch-throughput-mode)

**This rules out "bump the standby to Provisioned during failover" as an RTO
manoeuvre in the direction that matters.** Going *to* Provisioned is allowed
immediately, but you are then locked in for 24 hours — you cannot come back down
or reduce the amount. As an emergency lever it works once and then bills you for
a day. More importantly, it is a control-plane change during a failover, which
is exactly the kind of thing the 15-minute RTO does not have room for. Do not
put it in the runbook as a planned step; keep it as a documented escalation.

### Recommendation on throughput mode

| | Option A: Bursting on the standby | Option B: Elastic on the standby | Option C: Provisioned on the standby |
|---|---|---|---|
| Idle cost | Nothing beyond storage | Nothing beyond storage — you pay per byte read/written, and an idle standby reads/writes almost nothing | **Pays for provisioned MiBps 24/7 while doing nothing** |
| Behaviour at cutover | Full credit balance, ~12h of burst, then falls to a baseline derived from `ValueInStandard` | Scales automatically, no cliff | Flat, predictable, already paid for |
| Interaction with IA/Archive lifecycle | **Broken** — baseline collapses with `ValueInStandard` | Unaffected | Unaffected |
| Max throughput | 3–5 GiBps read | **20–60 GiBps read** | 3–10 GiBps read |

**Recommendation: Option B, Elastic, for both the primary and the standby.** It
is AWS's default and AWS's recommendation, it has no idle cost for a standby
that is genuinely idle, it is immune to the lifecycle/`ValueInStandard`
interaction, and it has the highest ceiling of the three by a wide margin. The
only reason to choose Provisioned is a primary with a known, sustained,
high average-to-peak ratio — and even then the standby should stay Elastic,
because the standby's ratio is zero by definition. **Do not mirror the primary's
throughput mode into the standby reflexively; they have opposite workloads.**

Note that the source and destination file systems are configured independently —
AWS states you can "select the destination file system's lifecycle management
policy, backup policies, provisioned throughput, mount targets, and access
points independent of the source file system"
([AWS Storage Blog](https://aws.amazon.com/blogs/storage/use-cases-for-amazon-efs-replication/)),
so choosing different throughput modes per side is explicitly supported, not a
hack.

---

## EFS — storage classes, lifecycle, and whether transitions replicate

### The classes

From [Managing storage lifecycle](https://docs.aws.amazon.com/efs/latest/ug/lifecycle-management-efs.html)
and the performance page:

| Class | For | First-byte latency |
|---|---|---|
| **Standard** | Frequently accessed | ~1 ms read, ~2.7 ms write (SSD) |
| **Infrequent Access (IA)** | "accessed only a few times each quarter" | "tens of milliseconds" |
| **Archive** | "accessed only a few times each year or less" | "tens of milliseconds" |

Lifecycle is three policies, applied to the **whole file system** (not per
directory, not per access point): transition into IA (default 30 days since last
access in Standard), transition into Archive (default 90 days), and transition
back into Standard on access (**default: off** — "By default, files are not
moved back to the Standard storage class, and they remain in the IA or Archive
storage class when they are accessed").

Three behaviours that matter for a standby:

- **"File metadata, including file names, ownership information, and file system
  directory structure, is always stored in Standard."** So a `find` or a
  directory walk on a fully-archived standby is fast; only reading file
  *contents* is slow. That is a meaningful mitigation — an application that
  starts by scanning the tree will not stall.
- **"All write operations to files in the file system's IA or Archive storage
  classes are first written to Standard storage classes, and are then eligible
  to be transitioned to the applicable storage class after 24 hours."** A
  promoted standby's writes land in Standard, so it self-warms as it is used.
- **"File system operations for lifecycle management have a lower priority than
  operations for EFS file system workloads."** Transitions are best-effort and
  slow; do not build timing assumptions on them.

### Does the lifecycle configuration replicate?

**No — and this is the answer to the question as asked.** The lifecycle
configuration is a property of the destination file system and is set
independently. AWS is explicit that the destination's "lifecycle management
policy, backup policies, provisioned throughput, mount targets, and access
points" are chosen "independent of the source file system"
([AWS Storage Blog, Use cases for Amazon EFS Replication](https://aws.amazon.com/blogs/storage/use-cases-for-amazon-efs-replication/)),
and that "lifecycle management is not enabled on the destination file system by
default, but after the destination file system is created, you can enable it."

So: **storage-class *transitions* do not replicate; storage-class *policy* is
per-file-system.** Replicated data arrives and is then aged by the destination's
own policy, on the destination's own last-access clock — which, for a standby
nobody reads, means *everything* eventually ages out to IA and then Archive if
you turn lifecycle on.

### The cost fork this creates

AWS positions aggressive lifecycle on the replica as the cost play: you can
"cost-optimize and save up to 75% on your disaster recovery storage costs by
using low-cost EFS One Zone storage classes and a 7-day age-off lifecycle
management policy for your destination file system"
([AWS Storage Blog](https://aws.amazon.com/blogs/storage/use-cases-for-amazon-efs-replication/)).

That is a real saving and it is also a direct trade against the 15-minute RTO:

| | Option A: standby in Standard, no lifecycle | Option B: standby Regional + IA/Archive lifecycle | Option C: standby One Zone + 7-day lifecycle |
|---|---|---|---|
| Storage cost | Full | Lower | **Lowest — AWS cites up to 75% off** |
| First-byte latency after promotion | ~1 ms | **Tens of ms until re-warmed** | Tens of ms |
| Bursting baseline after promotion | Correct | **Collapses** (see above) | Collapses |
| AZ redundancy in standby | 3 AZs | 3 AZs | **1 AZ** |
| Mount targets in standby | Up to 3 | Up to 3 | **Exactly 1** |
| Meets RTO 15m | Yes | Conditional — depends on whether tens-of-ms latency is acceptable for your app | Conditional, plus an AZ-failure exposure |

**Recommendation: Option A for anything on the RTO-15m critical path; Option C
for file systems that are archival in nature and not part of the promotion
path.** The specific reasoning: Option B and C do not make the standby *fail*,
they make it *slow*, and "slow" is the failure mode that passes a tabletop test
and fails a real incident — identical to the EBS lazy-hydration trap earlier in
this note. A warm standby that must serve production traffic 15 minutes from now
should not be holding its data in a class whose whole premise is that you rarely
read it.

Option C's One Zone caveat deserves its own line: **a One Zone replica puts your
entire DR copy in a single AZ of the standby Region.** You have built a
multi-Region DR plan whose recovery target has no AZ redundancy. For genuinely
cold archival data that is a reasonable trade. For the thing you fail over to,
it is not.

If you take Option A, **explicitly set the lifecycle policy to disabled on the
destination in Terraform rather than relying on the default.** The default is
off today; a future AWS default change or a copied module block should not
silently archive your DR copy.

---

## EFS — access points and POSIX across Regions

Access points are the application-facing entry point: they "can enforce a user
identity, including the user's POSIX groups, for all file system requests that
are made through the access point" and "can also enforce a different root
directory"
([AWS docs](https://docs.aws.amazon.com/efs/latest/ug/efs-access-points.html)).

What matters across Regions:

- **Access points depend on mount targets, not the other way round.** "You must
  create at least one mount target in your VPC before using access points …
  Access points inherit the mount target's Availability Zone placement" and
  "Access points are available in all Availability Zones where you have mount
  targets." So in the standby: mount targets first, then access points.
- **Access points do not replicate.** They are a property of the destination
  file system, configured independently — AWS lists "access points" among the
  destination properties you choose "independent of the source file system"
  ([AWS Storage Blog](https://aws.amazon.com/blogs/storage/use-cases-for-amazon-efs-replication/)).
  **You must create the standby's access points yourself, in Terraform, with
  identical POSIX settings.**
- **Access point IDs are different in the standby.** The mount command embeds
  the access point ID: `mount -t efs -o tls,iam,accesspoint=fsap-… fs-…`. The
  standby's `fsap-` ID will not match the primary's. **Anything that hardcodes
  an `fsap-` or `fs-` ID — a Helm value, a PersistentVolume manifest, a
  `volumeHandle`, a mount unit, a config map — is a failover blocker.** These
  IDs must come from config, per-region, not from a literal. This is the same
  class of problem as ARNs embedded in policies and is worth a grep across the
  estate before anything else in this note is implemented.
- **POSIX UIDs/GIDs are numbers and they replicate.** The *data* carries its
  ownership; the *enforcement* is the access point's `posix_user` /
  `creation_info` settings, which you re-declare in the standby. If the two
  sides disagree — a copy-paste error, a drifted UID — the standby will present
  the same files under a different effective identity and the application will
  fail with permission errors that look nothing like a DR problem. **Assert the
  access point configuration matches across Regions in CI**, not at 3am.
- **Security groups are applied at the mount target level, not the access point
  level.** Repeated here because it is the thing people get wrong: locking down
  an access point does not lock down the network path.
- **`elasticfilesystem:AccessedViaMountTarget`** is the IAM condition key that
  forces access to go through a mount target. If you use it in the primary's
  policies, the standby needs the equivalent — and IAM policies referencing
  file-system ARNs are region-specific. Cross-ref [[aws-iam]].

---

## EFS vs EBS for Kubernetes PVCs

The full Kubernetes-side treatment is in [[eks-stateful-workloads]]; this
section is the storage-side view of the same decision, because it is the one
that actually determines whether the estate can hit RTO 15m.

| | EBS CSI (`ebs.csi.aws.com`) | EFS CSI (`efs.csi.aws.com`) |
|---|---|---|
| Access modes | `ReadWriteOnce` (and `ReadWriteOncePod`) | **`ReadWriteMany`** |
| Attach scope | One node at a time; **AZ-pinned** | Any node in the Region with a mount target |
| Scheduling constraint | PV carries AZ node affinity — the pod follows the volume | None |
| Cross-AZ failure | Pod is stuck until a snapshot is restored into the new AZ | Pod reschedules and mounts immediately |
| Cross-Region DR | Snapshot copy → restore → new PV, manually reconciled | Replica file system, new PV pointing at the replica's `fs-` ID |
| Performance | Local block device, lowest latency | NFS over the network; higher latency, especially small-file metadata ops |
| Cost | Provisioned GiB, whether used or not | Metered on stored bytes (+ per-byte I/O on Elastic throughput) |

**The structural point.** An EBS-backed PVC pins a pod to an AZ *and* to a
Region, and nothing about Kubernetes changes that — the `StorageClass`,
`WaitForFirstConsumer` binding, topology-aware scheduling and the CSI driver all
work *around* the AZ constraint, none of them remove it. In the standby Region,
an EBS-backed StatefulSet has no volumes at all until someone restores
snapshots and hand-builds PersistentVolumes with the right `volumeHandle`. That
is a multi-step, error-prone, human procedure that does not fit inside 15
minutes for more than a handful of volumes.

**EFS sidesteps AZ pinning at a performance cost, and for a warm standby that
trade is usually correct.** An EFS-backed StatefulSet in the standby can be
scaled to zero replicas with its PVCs already bound to a PV pointing at the
replica file system. At failover you scale up. That is a `kubectl scale`, and it
fits in the RTO.

**The performance cost is real and should not be waved away.** EFS is NFS v4.1.
Per-operation latency is ~1 ms read / ~2.7 ms write at best
([EFS performance specs](https://docs.aws.amazon.com/efs/latest/ug/performance.html)),
versus microseconds for a local NVMe-backed EBS volume, and "every NFS request
is accounted for as 4 kilobyte (KB) of throughput, or its actual request and
response size, whichever is larger." Workloads that do many small operations —
Postgres, most embedded databases, anything with a write-ahead log, Git
checkouts, `node_modules` — will be dramatically slower on EFS and in some cases
will not work correctly at all, because POSIX file locking over NFS is not the
same as local locking.

**So the decision is not "EFS or EBS", it is "which of these three buckets is
this workload in":**

| Bucket | Example | Storage | Standby story |
|---|---|---|---|
| **Shared, moderate-I/O, genuinely file-shaped** | Uploaded assets, shared config, ML feature files, plugin directories, Grafana/Jenkins home | **EFS** | Replica + scaled-to-zero workload. Meets RTO. |
| **Transactional, high-I/O, database-shaped** | Postgres, Redis persistence, Kafka log dirs, Elasticsearch data | **Neither — use the managed service** | [[aws-rds-postgres]], [[aws-elasticache-redis]], [[aws-opensearch]], MSK. They have their own replication. |
| **Derived / disposable** | Build caches, scratch, node-local temp, log spool | **EBS, or instance store, and do not replicate it** | Standby rebuilds it. Zero RPO concern because there is no RPO. |

There is no fourth bucket in which "an EBS volume we snapshot-copy to the
standby Region and restore at failover" is the best available answer. That is
the central claim of this note.

---

## FSx — is it relevant here?

Briefly, because the answer is "probably not for this estate, but know the
number."

**FSx for NetApp ONTAP is the strongest cross-Region file DR story on AWS, and
it beats EFS Replication on paper.** AWS's own Storage Blog states: "SnapMirror
enables you to configure replication with an **RPO of as low as five minutes,
and an RTO in single digit minutes**," and "Replication can be scheduled as
frequently as every five minutes"
([Cross-region disaster recovery with Amazon FSx for NetApp ONTAP, AWS Storage
Blog](https://aws.amazon.com/blogs/storage/cross-region-disaster-recovery-with-amazon-fsx-for-netapp-ontap/)).
Failover is breaking the SnapMirror relationship, which "makes the destination
volume writable and does not impact the primary ONTAP volume" — and AWS is
explicit that in a regional outage "failover to the secondary AWS Region …
would be manual and would have to be completed by operational teams."

**Should this estate use it? No, and here is the honest reasoning, not a
dismissal.** RPO 5 min versus RPO 15 min is a distinction without a difference
against a **2-hour** target — both clear it by an order of magnitude. What FSx
for ONTAP would buy you is a better *failback* story (SnapMirror resync is a
first-class operation, versus EFS's delete-and-recreate-with-a-full-initial-sync)
and genuine multiprotocol/SMB support. What it costs you is a second storage
platform to operate, ONTAP-specific expertise, a Terraform surface the team
doesn't have, and a migration off EFS. **For a 2h/15m target that EFS
Replication already meets, that is not a trade worth making.** Revisit only if
(a) failback duration measured in a gameday turns out to be operationally
unacceptable, or (b) an SMB/Windows workload appears.

FSx for Windows File Server and FSx for OpenZFS were not researched in depth
here because no Windows or ZFS workload is in scope per the research brief. **No
published cross-Region RPO/RTO figures for those two were verified in this
pass** — do not quote any without checking.

---

## EBS — volume types on the standby

A commonly-asked question with a slightly surprising answer: **does the standby
need the same volume type as the primary?**

**Technically, no.** A snapshot is type-agnostic. You can restore a snapshot
taken from an `io2` volume into a `gp3` volume, and vice versa. Nothing about
the snapshot encodes the source volume type.

**Practically, it depends on whether the volume is idle or pre-provisioned:**

- **If the standby volume does not exist until failover** (the normal warm-standby
  shape for EBS — you hold snapshots, not volumes), there is no idle cost and no
  decision to make. You choose the type at restore time, in the launch template
  / `StorageClass` / Terraform, and it should match the primary because the
  moment it exists it is serving production traffic.
- **If you pre-provision standby volumes to dodge the lazy-hydration problem**,
  then you are paying for provisioned capacity 24/7 and the type absolutely
  matters. `io2` bills provisioned IOPS separately from storage; an idle `io2`
  volume with 30,000 provisioned IOPS is burning money to do nothing. `gp3`
  includes a free baseline of "3,000 provisioned IOPS and 125 provisioned MB/s
  throughput" with additional IOPS and throughput "provisioned independently"
  ([EBS pricing](https://aws.amazon.com/ebs/pricing/)), which makes it far more
  sensible for an idle standby.

**Recommendation: gp3 for anything pre-provisioned and idle; match the primary's
type only for volumes created at failover time.** And note that pre-provisioning
standby EBS volumes does not actually solve anything on its own — an empty
pre-provisioned volume still has to be populated from a snapshot, which is the
lazily-hydrated operation you were trying to avoid. Pre-provisioning helps only
with the control-plane time to *create* the volume, which was never the
bottleneck.

**Two quota asymmetries to check before assuming the standby can hold what the
primary holds** (from
[Quotas for Amazon EBS](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-resource-quotas.html)):

- **`io2` storage is capped at 20 TiB per Region by default** — far lower than
  gp2/gp3/io1/st1/sc1, which default to 50 TiB in most Regions and **300 TiB in
  `ca-west-1`** and other newer Regions. If the primary runs a lot of `io2`, the
  standby needs a quota increase raised *in advance*, not during an incident.
- **`ca-west-1` actually has *higher* default volume-storage quotas than
  `ca-central-1`** for gp2/gp3/io1/st1/sc1/standard: 300 TiB vs 50 TiB. Calgary
  being a newer Region works in your favour here. A rare win.
- **Aggregate `io2` IOPS default to 100,000 per Region** vs 300,000 for `io1`.

---

## EBS — snapshot quotas

All verified against
[Quotas for Amazon EBS](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-resource-quotas.html):

| Quota | Default | Adjustable? | Why it matters here |
|---|---|---|---|
| **Snapshots per Region** | **100,000** | Yes | Generous, but a 1-hour DLM cadence × N volumes × retention adds up faster than people expect. 60 volumes hourly with 7-day retention = ~10,000 snapshots **in each Region**. |
| **Concurrent snapshot copies per destination Region** | **20** | **Not via Service Quotas** — "you can request an increase … by contacting AWS Support" | The throughput ceiling on your cross-Region copy fan-out. Drives the RPO arithmetic above. |
| **Time-based snapshot copy throughput per destination Region** | **2,000 MiB/s** | Yes | ≈ 7 TiB/hour of copy bandwidth into a standby Region, account-wide, when using time-based copies. **This is the number to do capacity planning against.** |
| Concurrent snapshots per gp2/gp3/io1/io2/standard volume | 5 | No | Fine for any sane cadence. |
| Concurrent snapshots per st1/sc1 volume | **1** | No | Throughput-optimised HDD volumes can only have one snapshot in progress. A slow snapshot on a large st1 volume serialises the next one — a real risk for big data volumes. |
| Archived snapshots per volume | 25 | Yes | Retention, not DR. |
| In-progress snapshot archives per account | 25 | Yes | Retention, not DR. |
| In-progress snapshot **restores from archive** per account | **5** | Yes | If your DR plan ever depends on restoring from archive, five at a time, at up to 72 hours each, is the ceiling. It is not a DR mechanism. |
| **Fast snapshot restore** | **5 per Region** — explicitly listed as 5 for `ca-west-1`, `ca-central-1`, `eu-west-1`, `eu-west-2`, `us-east-1`, `us-west-2` and every other Region | Yes | The cap that kills FSR as a general strategy. |
| EBS volumes per instance launch request | 2,500 in `us-east-1`/`us-west-2`/`eu-west-1`/`ap-northeast-1`; **500 everywhere else** | No | **`eu-west-2`, `ca-central-1` and `ca-west-1` are all in the 500 bucket while `eu-west-1` and `us-east-1` are in the 2,500 bucket.** A large-scale mass launch in the standby can hit a limit the primary never hits. |

Note the useful detail AWS adds: "Amazon EBS constantly monitors your
provisioned storage and IOPS usage within each Region and **might automatically
increase your quotas**, on a per-Region basis, based on your usage." Your
primary Region has been silently raised over years of use. **Your brand-new
standby Region has not.** Do not assume parity; check Service Quotas in the
standby explicitly and raise anything that matters before you need it. This is
a general multi-region lesson and belongs in [[lessons-and-antipatterns]].

---

## RPO / RTO analysis against 2h / 15m

### EFS

| Phase | Time | Pre-provisioned? |
|---|---|---|
| Data currency at the moment of the incident | **≤15 min** (AWS-published RPO) | — |
| Decide to fail over | Human | — |
| `DeleteReplicationConfiguration` on the replica | Seconds — one API call | — |
| Mount targets exist in standby VPC | **0 — must already exist** | **Yes, mandatory** |
| Access points exist in standby | **0 — must already exist** | **Yes, mandatory** |
| Security groups permit NFS 2049 from standby nodes | **0 — must already exist** | **Yes, mandatory** |
| Workload scales up and mounts | Minutes — pod start + mount | Pods pre-created at 0 replicas |
| **Total** | **Minutes. Meets RTO 15m.** | |

**EFS meets both targets, conditionally on pre-provisioning the network layer.**
The condition is cheap: mount targets and access points cost nothing to hold.
There is no excuse for not having them.

The two things that can still blow the RTO are not EFS's fault: (a) an
`fsap-`/`fs-` ID hardcoded somewhere, and (b) a standby whose data is all in
Archive and therefore answering at tens of milliseconds.

### EBS

| Phase | Time | Pre-provisioned? |
|---|---|---|
| Data currency | **1h 10m – 2h 40m+** with DLM at 1h (see arithmetic above) | — |
| Identify the correct snapshot copy in the standby | Minutes, and only if you tagged copies — IDs and tags do not carry over | No |
| `CreateVolume` from snapshot | Seconds–minutes | No |
| **Volume reaches full performance (lazy hydration)** | **Minutes to hours, unbounded** | Only with FSR — capped at 5 snapshots/Region, $540/AZ/month each |
| Attach, mount, reconstruct PVs / launch templates | Minutes, human-driven | No |
| **Total** | **Well beyond 15 minutes for anything non-trivial.** | |

**EBS fails RTO 15m and cannot be made to pass it at any reasonable cost.** The
FSR arithmetic is the proof: the only mechanism that fixes lazy hydration is
capped at five snapshots per Region, costs $540 per snapshot per AZ per month,
must be re-enabled on each new copy, takes 60 minutes per TiB to optimize, and
silently no-ops when you exceed the cap.

---

## Warm standby shape

### EFS — warm standby shape and the RTO 15m question

Standing in the standby Region while the primary is healthy:

| Resource | State | Costs money? |
|---|---|---|
| Destination file system | Exists, **read-only**, continuously synced | **Yes — storage for the full data set**, plus ~12 MiB of replication metadata |
| Mount targets (one per AZ) | Exist | No charge for the mount target; consumes an ENI + IP |
| Security groups | Exist, allow 2049 from node SGs | No |
| Access points | Exist, POSIX config mirrored | No |
| Throughput mode | **Elastic** | Effectively nothing while idle (metered per byte read/written) |
| Lifecycle policy | **Disabled** for RTO-critical file systems | — |
| Backup plan on the replica | Optional; see [[aws-backup]] | Yes if enabled |
| Workload consuming it | Deployed at 0 replicas, PVCs bound | See [[aws-eks]] |

**The dominant idle cost is simply a second full copy of the data**, which is
irreducible — that is what a replica is. Everything else in the standby EFS
footprint is free or negligible. Compared to the other services in this estate,
EFS's warm standby is remarkably cheap, and the temptation to optimise it with
One Zone + aggressive lifecycle should be resisted on the RTO-critical path.

> AWS's EFS pricing page renders its rate tables client-side and **this research
> pass could not read the per-GB figures from it**. No EFS per-GB prices are
> quoted in this note rather than risk inventing them. Two figures AWS does
> state in prose on that page: **$0.01/GB for cross-AZ data transfer**, and a
> "TCO as low as $0.0315/GB" headline claim. Get real per-Region rates from the
> [AWS Pricing Calculator](https://calculator.aws/) before building
> [[cost-model]].

### EBS — warm standby shape

| Resource | State | Costs money? |
|---|---|---|
| Snapshot copies in the standby Region | Exist, incremental after the first | **Yes — snapshot storage, per GB of changed blocks** |
| Cross-Region transfer | Per copy cycle | **Yes — EC2 data transfer rates** |
| Volumes | **Do not exist** | No |
| FSR | Off (recommended) | $540/snapshot/AZ/month if on |
| DLM policy | In the **primary** Region | Free |

Note the asymmetry with EFS: EFS's standby is a live file system you can
promote; EBS's "standby" is a pile of snapshots that is not a resource anyone
can use until someone builds volumes out of it. **That difference is the entire
RTO argument.**

> EBS per-GB prices (gp3 storage, gp3 IOPS/throughput, io2 storage and tiered
> IOPS, snapshot standard storage, snapshot archive, and archive restore) are
> also rendered client-side on the
> [EBS pricing page](https://aws.amazon.com/ebs/pricing/) and **were not
> readable in this pass.** The one EBS price quoted anywhere in this note —
> **FSR at $0.75 per snapshot per AZ per hour** — is stated in prose in the
> [FSR documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-fast-snapshot-restore.html#fsr-pricing)
> and is safe to rely on. Everything else: pricing calculator, before you commit
> a number to [[cost-model]].

---

## Terraform implementation

### The single most important Terraform fact in this note

```
Deleting aws_efs_replication_configuration IS the failover.
```

The provider documentation states it plainly: creating the resource "initiates
replication to a new read-only destination file system. Upon deletion,
replication stops and **the destination file system loses its read-only
status, though it remains intact**"
([aws_efs_replication_configuration](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/efs_replication_configuration)).

That is elegant and extremely dangerous in equal measure:

- **Elegant**: your failover is `terraform destroy -target=...` or a
  `terraform apply` with the replication module toggled off. It is auditable,
  reviewable, and in version control.
- **Dangerous**: a careless refactor, a renamed module, a `count = 0` typo, a
  moved resource without a `moved` block — **any of these silently breaks
  replication and makes your DR replica writable while production carries on
  obliviously.** There is no confirmation prompt, no `prevent_destroy` by
  default, and nothing about the plan output screams "you are about to disable
  disaster recovery."

**Mitigations, all of which should be applied:**

```hcl
resource "aws_efs_replication_configuration" "this" {
  source_file_system_id = aws_efs_file_system.this.id

  destination {
    region = var.standby_region
    # kms_key_id defaults to the AWS-managed /aws/elasticfilesystem key in the
    # destination Region. Set it explicitly to a CMK you control if the estate's
    # KMS policy requires customer-managed keys. See [[aws-kms]].
    kms_key_id = var.standby_efs_kms_key_arn
  }

  lifecycle {
    # Failover must be a deliberate act, not a plan artefact.
    prevent_destroy = true
  }
}
```

With `prevent_destroy = true`, failover becomes: comment out the lifecycle
block, apply. That is one extra deliberate step at 3am and a very large amount
of protection the rest of the time. **Alternatively — and this is the better
answer for a 15-minute RTO — do not fail over with Terraform at all.** Use
`aws efs delete-replication-configuration --source-file-system-id fs-…` in the
runbook, and reconcile Terraform afterwards. Terraform is a poor tool for
time-critical operations; see [[failover-orchestration]].

### The destination file system is not a Terraform resource

This is the second thing that surprises people. When `destination.file_system_id`
is omitted, **AWS creates the destination file system and Terraform does not
manage it.** You get its ID back as an attribute:

```hcl
output "standby_file_system_id" {
  value = aws_efs_replication_configuration.this.destination[0].file_system_id
}
```

Consequences:

- The destination file system is **not in state**. `terraform destroy` of the
  whole stack leaves it behind, billing you. Add it to the decommissioning
  runbook.
- Its tags, backup policy and `protection` settings are **not managed by
  Terraform** unless you import it or pre-create it.
- Mount targets, access points and security groups for the standby must
  reference that attribute, which means they are in the same apply and
  implicitly depend on the replication configuration existing.

**Recommendation: pre-create the destination file system explicitly** with an
`aws_efs_file_system` resource behind the standby provider alias, and pass its
ID via `destination.file_system_id`. You give up nothing and you gain a fully
managed, fully tagged, fully configurable standby file system whose throughput
mode and lifecycle policy you control in code — which, per the throughput and
lifecycle sections above, you very much want to control.

### Module shape for a cookiecutter monorepo

The estate is a mature, cookiecutter-templated, multi-env Terraform monorepo and
the goal is to evolve it. The shape that fits is a module that takes **two
providers** and owns both sides of the pair — consistent with
[[provider-aliases-vs-separate-stacks]] and [[module-patterns]].

```hcl
# modules/efs-replicated/versions.tf
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = ">= 5.40, < 7.0"
      configuration_aliases = [aws.primary, aws.standby]
    }
  }
}
```

```hcl
# modules/efs-replicated/variables.tf
variable "name"        { type = string }
variable "environment" { type = string }

variable "standby_region" {
  type        = string
  description = "Paired standby Region, e.g. eu-west-2 for the EU pair."
}

variable "primary_subnet_ids" {
  type        = list(string)
  description = "One subnet per AZ in the primary VPC. EFS allows exactly one mount target per AZ."
}

variable "standby_subnet_ids" {
  type        = list(string)
  description = "One subnet per AZ in the standby VPC. Mount targets are per-AZ AND per-VPC; the standby needs its own."
}

variable "primary_mount_target_security_group_ids" { type = list(string) }
variable "standby_mount_target_security_group_ids" { type = list(string) }

variable "throughput_mode" {
  type        = string
  default     = "elastic"
  description = "elastic | provisioned | bursting. Elastic is recommended for both sides; see note. A cold standby has an average-to-peak ratio of zero, which is precisely the Elastic use case."
  validation {
    condition     = contains(["elastic", "provisioned", "bursting"], var.throughput_mode)
    error_message = "throughput_mode must be elastic, provisioned or bursting."
  }
}

variable "standby_throughput_mode" {
  type        = string
  default     = null
  description = "Override for the standby. Defaults to var.throughput_mode. Deliberately separable: the primary and standby have opposite workload shapes."
}

variable "lifecycle_transition_to_ia" {
  type        = string
  default     = null
  description = "e.g. AFTER_30_DAYS. Null = disabled."
}

variable "standby_lifecycle_transition_to_ia" {
  type        = string
  default     = null
  description = "Deliberately separate from the primary's. Leave null for anything on the RTO-critical path: an archived standby answers in tens of milliseconds, and a Bursting standby's baseline throughput collapses with ValueInStandard."
}

variable "access_points" {
  description = "Created identically on BOTH file systems. IDs differ per Region; consumers must read them from output, never hardcode fsap- IDs."
  type = map(object({
    root_directory_path = string
    posix_uid           = number
    posix_gid           = number
    permissions         = string
  }))
  default = {}
}

variable "kms_key_arn_primary" {
  type    = string
  default = null
}

variable "kms_key_arn_standby" {
  type    = string
  default = null
}
```

```hcl
# modules/efs-replicated/main.tf
locals {
  standby_throughput_mode = coalesce(var.standby_throughput_mode, var.throughput_mode)
  common_tags = {
    Environment = var.environment
    Module      = "efs-replicated"
    # Tag both sides identically so the standby is discoverable by the same
    # tooling as the primary. Note this is EFS, so tags are ours to set; EBS
    # snapshot copies do NOT inherit tags (see the EBS section).
  }
}

resource "aws_efs_file_system" "primary" {
  provider = aws.primary

  creation_token  = "${var.name}-${var.environment}"
  encrypted       = true
  kms_key_id      = var.kms_key_arn_primary
  throughput_mode = var.throughput_mode

  dynamic "lifecycle_policy" {
    for_each = var.lifecycle_transition_to_ia == null ? [] : [1]
    content { transition_to_ia = var.lifecycle_transition_to_ia }
  }

  tags = merge(local.common_tags, { Name = "${var.name}-${var.environment}", Role = "primary" })
}

# Pre-created rather than letting the replication configuration conjure one,
# so that throughput mode, lifecycle and tags are managed in code.
resource "aws_efs_file_system" "standby" {
  provider = aws.standby

  creation_token  = "${var.name}-${var.environment}-standby"
  encrypted       = true
  kms_key_id      = var.kms_key_arn_standby
  throughput_mode = local.standby_throughput_mode

  dynamic "lifecycle_policy" {
    for_each = var.standby_lifecycle_transition_to_ia == null ? [] : [1]
    content { transition_to_ia = var.standby_lifecycle_transition_to_ia }
  }

  tags = merge(local.common_tags, { Name = "${var.name}-${var.environment}-standby", Role = "standby" })
}

resource "aws_efs_replication_configuration" "this" {
  provider              = aws.primary
  source_file_system_id = aws_efs_file_system.primary.id

  destination {
    region         = var.standby_region
    file_system_id = aws_efs_file_system.standby.id
  }

  lifecycle {
    prevent_destroy = true # failover must be deliberate
  }
}

# Mount targets: exactly one per AZ, per VPC. Both sides.
resource "aws_efs_mount_target" "primary" {
  provider        = aws.primary
  for_each        = toset(var.primary_subnet_ids)
  file_system_id  = aws_efs_file_system.primary.id
  subnet_id       = each.value
  security_groups = var.primary_mount_target_security_group_ids
}

resource "aws_efs_mount_target" "standby" {
  provider        = aws.standby
  for_each        = toset(var.standby_subnet_ids)
  file_system_id  = aws_efs_file_system.standby.id
  subnet_id       = each.value
  security_groups = var.standby_mount_target_security_group_ids
}

# Access points do not replicate. Create them identically on both sides.
resource "aws_efs_access_point" "primary" {
  provider       = aws.primary
  for_each       = var.access_points
  file_system_id = aws_efs_file_system.primary.id

  posix_user {
    uid = each.value.posix_uid
    gid = each.value.posix_gid
  }
  root_directory {
    path = each.value.root_directory_path
    creation_info {
      owner_uid   = each.value.posix_uid
      owner_gid   = each.value.posix_gid
      permissions = each.value.permissions
    }
  }
  tags = merge(local.common_tags, { Name = "${var.name}-${each.key}" })
}

resource "aws_efs_access_point" "standby" {
  provider       = aws.standby
  for_each       = var.access_points
  file_system_id = aws_efs_file_system.standby.id

  posix_user {
    uid = each.value.posix_uid
    gid = each.value.posix_gid
  }
  root_directory {
    path = each.value.root_directory_path
    creation_info {
      owner_uid   = each.value.posix_uid
      owner_gid   = each.value.posix_gid
      permissions = each.value.permissions
    }
  }
  tags = merge(local.common_tags, { Name = "${var.name}-${each.key}-standby" })
}
```

```hcl
# modules/efs-replicated/outputs.tf
output "primary_file_system_id" { value = aws_efs_file_system.primary.id }
output "standby_file_system_id" { value = aws_efs_file_system.standby.id }

output "access_point_ids" {
  description = "Per-region maps. Consumers MUST select by the region they are running in. Hardcoding an fsap- ID is a failover blocker."
  value = {
    primary = { for k, v in aws_efs_access_point.primary : k => v.id }
    standby = { for k, v in aws_efs_access_point.standby : k => v.id }
  }
}

output "replication_status" {
  value = aws_efs_replication_configuration.this.destination[0].status
}
```

Note `aws_efs_replication_configuration` has `create` and `delete` timeouts
defaulting to **20 minutes** each — worth raising for large file systems, since
create waits on control-plane setup and the initial sync can be long.

### EBS side: DLM in the primary only

```hcl
# modules/ebs-snapshot-dr/main.tf  —  primary provider only
resource "aws_dlm_lifecycle_policy" "this" {
  provider           = aws.primary
  description        = "${var.name}-${var.environment} EBS snapshots + cross-Region copy"
  execution_role_arn = aws_iam_role.dlm.arn
  state              = "ENABLED"

  policy_details {
    resource_types = ["VOLUME"]

    # DLM targets by tag, in its own Region only. It is NOT instantiated twice.
    target_tags = {
      Backup      = "true"
      Environment = var.environment
    }

    schedule {
      name = "hourly-with-cross-region-copy"

      create_rule {
        interval      = 1          # valid: 1,2,3,4,6,8,12,24
        interval_unit = "HOURS"
        times         = ["00:00"]
        # NOTE: DLM fires "within one hour of" the scheduled time. This schedule
        # therefore cannot guarantee an RPO of 2h once copy time is added.
      }

      # Retention here MUST exceed the copy interval, or the previous copy is
      # deleted before the next one lands and every copy reverts to a FULL copy.
      retain_rule { count = 24 }

      copy_tags = true

      cross_region_copy_rule {
        target    = var.standby_region
        encrypted = true

        # Use a multi-Region KMS key replica so copies stay INCREMENTAL.
        # A different key ID forces a full copy every cycle. See [[aws-kms]].
        cmk_arn = var.standby_kms_key_arn

        copy_tags = true # raw CopySnapshot copies NO tags; DLM can.

        retain_rule {
          interval      = 7
          interval_unit = "DAYS"
        }
      }

      # Deliberately NOT set. FSR is capped at 5 snapshots/Region, costs
      # $0.75/snapshot/AZ/hour, takes 60 min/TiB to optimize, does not survive a
      # cross-Region copy, and stays enabled (and billing) after this policy is
      # destroyed. See the FSR section.
      # fast_restore_rule { ... }
    }
  }

  tags = { Environment = var.environment }
}
```

The IAM role needs the KMS permissions listed in the
[KMS section](#ebs--kms-on-cross-region-copy) **against the destination
Region's key**, including `kms:CreateGrant`. A role that works for same-Region
snapshots will fail silently on the cross-Region copy.

---

## Migration path from single-region

### EFS: additive, no downtime, no replacement — with one caveat

**Adding `aws_efs_replication_configuration` to a live file system is purely
additive.** The source file system is not modified; replication is a separate
configuration object. There is no `ForceNew` on `aws_efs_file_system` triggered
by adding replication, and no client-visible interruption.

Steps:

1. **Pre-flight: check the source file system's immutable attributes.** The
   provider docs for `aws_efs_file_system` do not annotate which arguments force
   replacement, but the EFS API has **no modify operation** for
   `performance_mode`, `encrypted`, `kms_key_id` or `availability_zone_name` —
   they are set at creation. **Run `terraform plan` and read it** before
   assuming a refactor is safe; a plan that shows `# forces replacement` on a
   production file system is a data-loss event, not a deployment.
   *If the live file system is unencrypted, there is no in-place fix* — you
   would need a new encrypted file system and a DataSync/`rsync` migration. Find
   this out now, not in month three.
2. **Stand up the standby VPC, subnets and security groups** in the paired
   Region ([[aws-vpc-networking]]).
3. **Create the standby file system explicitly** (per the recommendation above),
   with Elastic throughput and lifecycle disabled.
4. **Create standby mount targets and access points.** Cheap, and they must
   exist before anything can mount.
5. **Add the replication configuration** pointing at the pre-created standby.
   The initial sync begins; duration is proportional to size and file count and
   is **not published by AWS**. Watch `TimeSinceLastSync`. Run this step
   deliberately, not as a side effect of a Friday deploy.
6. **Wait for the initial sync to complete.** "Replicated data is accessible in
   the destination file system only after the initial sync completes."
7. **Alarm on `TimeSinceLastSync`** before declaring done. A replication that
   has silently stalled looks identical to one that is working.
8. **Grep the estate for hardcoded `fs-` and `fsap-` IDs** and move them to
   per-region config. This is the step everyone skips and it is the one that
   breaks the failover.
9. **Gameday it** ([[dr-testing-and-gamedays]]) — and time the failback, because
   that number is not available anywhere else.

**If the file system already exists outside Terraform**, import it
(`terraform import aws_efs_file_system.primary fs-…`) before adding replication,
so that step 1's plan is meaningful.

### EBS: there is no migration, because there is nothing to migrate

Adding a DLM policy to a live estate changes nothing about the running volumes —
it is a new, independent resource that snapshots them on a schedule. No
downtime, no replacement, no risk to the workload. The only things to get right:

1. **Tag the volumes you want backed up** (`Backup = "true"` or whatever the
   estate's convention is). DLM targets by tag and **target tags are case
   sensitive**.
2. **Create the destination KMS key first**, ideally a multi-Region replica, and
   grant the DLM role access to it in the destination Region.
3. **Expect the first copy of every volume to be full.** Stagger the rollout
   across days. Do not enable a 1-hour cross-Region policy for 300 volumes on a
   Tuesday morning and then wonder about the data-transfer bill.
4. **Set destination retention longer than the copy interval** so incrementality
   survives.
5. Then — per the recommendation below — **treat this as a stopgap and start
   moving the state off EBS.**

---

## Failover procedure

### EFS, at 3am

1. **Decide.** Human. The RTO clock definition matters here — see the open
   question about "15 minutes from what" in `CLAUDE.md` and
   [[failover-orchestration]].
2. **Check `TimeSinceLastSync` on the replication configuration** and record it.
   This is your actual data loss. Do it *before* step 3, because step 3 destroys
   the configuration that reports it.
3. **Make the replica writable:**
   ```
   aws efs delete-replication-configuration \
     --source-file-system-id fs-<PRIMARY_ID> \
     --region <PRIMARY_REGION>
   ```
   **This is the failover. It is a `delete` command and that is correct.** It
   deletes the *replication configuration*, not the file system and not the
   data. Write it in the runbook in exactly these words, because the person
   running it will hesitate.
   > If the primary Region's control plane is unavailable, this call may fail.
   > **This is an untested failure mode in this research** — the API is scoped to
   > the source file system, which lives in the failing Region. Verify in a
   > gameday whether the call succeeds when the primary Region is degraded.
   > It is the single biggest open risk in the EFS failover path.
4. **Scale up the standby workload.** PVCs are already bound to PVs pointing at
   the replica's `fs-` ID; access points already exist. `kubectl scale` or
   equivalent. See [[aws-eks]] and [[eks-stateful-workloads]].
5. **Validate consistency, not just availability.** Because replication is
   explicitly **not point-in-time consistent**, check for the specific
   marker-without-data patterns your application can produce. A generic "can I
   read a file" check will pass on a corrupt data set.
6. **Shift traffic.** [[aws-route53]], [[failover-orchestration]].

### EBS, at 3am

There is no good version of this, which is the point. For completeness:

1. Find the most recent snapshot copy per volume in the standby Region — by the
   tags you hopefully applied, since IDs and volume IDs do not correlate.
2. `CreateVolume` from each, in the correct AZ, with the correct type and size.
3. Attach to instances, or construct PersistentVolumes with the right
   `volumeHandle` and node affinity.
4. Mount, `fsck` if the snapshot was crash-consistent rather than
   application-consistent, start the application.
5. **Accept degraded performance for an unbounded period** while the volumes
   lazily hydrate, unless the snapshot was one of your five FSR-enabled ones and
   had finished optimizing.

Steps 1–4 are scriptable. Step 5 is not fixable. **Do not write this runbook;
write the migration plan instead.**

---

## Failback

### EFS failback — the genuinely hard part

Failover is one API call. Failback is a project. Sequence, per
[AWS's documented options](https://docs.aws.amazon.com/efs/latest/ug/replication-fail-over.html):

**If you want to keep the writes that happened while failed over** (almost
always — that is production data):

1. Create a replication configuration on the **former replica**, with the
   **former primary** as the destination. AWS "identifies and transfers the
   differences from your replica file system back to the primary."
2. **Wait for the initial sync.** This is a full one-time setup sync, not a
   delta — "Replicated data is accessible in the destination file system only
   after the initial sync completes." Duration: proportional to size and file
   count, **not published by AWS**.
3. Quiesce writes in the standby and cut traffic back to the original Region.
4. Delete that replication configuration to make the original primary writable.
5. **Re-create the original replication configuration** (original primary →
   original replica) to restore the normal posture. **This triggers a second
   initial sync.**

**Two full initial syncs to complete one round trip.** That is the number that
should govern how you think about failback, and it is why failback is a planned
maintenance window, not an emergency action. Nothing in the 15-minute RTO
applies.

**If you are willing to discard the writes made while failed over** — a real
option for read-mostly file systems — you skip steps 1–3 entirely: just
re-create the original replication configuration and let EFS overwrite the
replica back to match the primary. One initial sync instead of two, and far
faster. **Decide in advance, per file system, which of these two failback modes
applies**, and write it in the runbook. Deciding at 3am, after the incident,
with a director asking when it will be done, is how the wrong one gets chosen.

**Gameday obligation:** AWS publishes no initial-sync throughput figure. **You
cannot plan a failback window without measuring your own.** Make "time a full
failback" an explicit objective of the first EFS gameday and record the number
in [[dr-testing-and-gamedays]]. No other source can give you this.

### EBS failback

Symmetrical to failover and equally unpleasant: snapshot the volumes in the
(now-primary) standby, copy them back, restore, re-hydrate. Note that the
**first copy back is a full copy** — the reverse direction has no prior copy to
be incremental against. Plan for full transfer of the entire data set in the
failback direction.

---

## Gotchas

The list that makes this note worth re-reading.

**EBS**

1. **AZ-scoped, always.** Not a quota, not a setting. There is no cross-AZ
   attach and there never will be.
2. **The first cross-Region copy of every volume is full.** AWS's
   "Considerations" bullet says cross-Region copies are always full; the
   "Incremental snapshot copying" section two paragraphs later says they are
   incremental if four conditions hold. The second is operative; the first means
   "the first one".
3. **Destination retention shorter than the copy interval silently reverts you
   to full copies, forever.** Cost triples, copies slow, RPO blows.
4. **Re-keying to a different KMS key forces a full copy.** Use a multi-Region
   key.
5. **Archiving the latest standby copy breaks incrementality** and takes up to
   72 hours to reverse.
6. **User-defined tags are NOT copied by `CopySnapshot`.** Your standby fills up
   with untagged snapshots your automation cannot see.
7. **Copies get an arbitrary volume ID (`vol-ffffffff`).** You cannot correlate
   a copy to its source volume without tagging it yourself.
8. **Encrypted-copy permission failures are silent** — "the operation fails
   silently and the snapshot copy receives the 'Given key ID is not accessible'
   status message." Alarm on snapshot *age* in the destination, not job success.
9. **`kms:CreateGrant` is regional.** A key policy that works in the primary
   does not make the copy work in the standby.
10. **FSR does not survive a snapshot copy.** You pay for it twice or not at all.
11. **FSR is capped at 5 snapshots per Region** — including `ca-west-1`, which
    AWS lists explicitly at 5.
12. **FSR stays enabled (and billing) after you delete the DLM policy.** Manual
    teardown required. $540/snapshot/AZ/month leak.
13. **FSR silently no-ops when the cap is exceeded** — snapshots still get
    created, just not accelerated.
14. **FSR takes 60 minutes per TiB to optimize**, so it is incompatible with a
    tight snapshot cadence above ~500 GiB.
15. **DLM fires "within one hour of" its scheduled time.** This alone makes a 2h
    RPO unreliable at DLM's fastest (1h) interval.
16. **Restored volumes are lazily hydrated** and slow until warm. Health checks
    pass; users suffer. The classic gameday-passes / incident-fails trap.
17. **st1 and sc1 volumes allow only ONE concurrent snapshot.** Large HDD
    volumes serialise.
18. **20 concurrent snapshot copies per destination Region**, not adjustable via
    Service Quotas — you must contact Support.
19. **`io2` regional storage quota defaults to 20 TiB** vs 50–300 TiB for other
    types. Check the standby before assuming it can hold the primary's data.
20. **EBS volumes per instance launch request is 500 outside
    `us-east-1`/`us-west-2`/`eu-west-1`/`ap-northeast-1`** — which puts
    `eu-west-2`, `ca-central-1` and `ca-west-1` in the lower bucket. A mass
    launch in the standby can hit a limit the primary never does.
21. **AWS silently raises quotas in Regions you use heavily.** Your years-old
    primary has been raised. Your new standby has not.
22. **Deleting a source volume orphans its snapshots** from DLM management, in
    both Regions. A slow cost leak in any estate that recycles nodes.
23. **Snapshots are crash-consistent, not application-consistent**, unless you
    use DLM pre/post scripts.

**EFS**

24. **Failover is `DeleteReplicationConfiguration`.** The correct action reads
    like the wrong one. Put that reassurance in the runbook.
25. **Deleting `aws_efs_replication_configuration` in Terraform IS a failover.**
    A `count = 0`, a rename without a `moved` block, or a botched refactor
    silently disables DR and makes the replica writable. Use
    `prevent_destroy`.
26. **The destination file system created implicitly by the replication
    configuration is not managed by Terraform** and survives
    `terraform destroy`, billing you. Pre-create it explicitly instead.
27. **Replication is NOT point-in-time consistent.** Files written together are
    not guaranteed to arrive together. Applications that rely on cross-file
    write ordering need checking.
28. **Mount targets, access points, security groups and lifecycle policies do
    not replicate.** Only data does. All of the above must be built in the
    standby by you.
29. **One mount target per AZ, and changing a mount target's VPC or IP requires
    delete-and-recreate.** Get the subnet choice right the first time.
30. **Access point IDs differ between Regions.** Any hardcoded `fsap-` or `fs-`
    ID is a failover blocker. Grep for them.
31. **A blocked NFS port hangs, it does not error.** A standby with mount
    targets but no SG ingress rule fails as a hang at 3am.
32. **Lifecycle on the standby collapses Bursting baseline throughput**, because
    baseline is computed from `ValueInStandard`. Credits will be full and
    useless.
33. **Elastic throughput does not accrue or consume burst credits at all** —
    which is why it sidesteps gotcha 32 entirely.
34. **Switching to Provisioned throughput locks you out of switching back, or
    reducing, for 24 hours.** Not an emergency lever.
35. **Failback requires two full initial syncs** if you preserve the replica's
    writes. AWS publishes no throughput figure for initial sync; you must
    measure it.
36. **There is no reverse or swap API.** "Reversing" replication is
    delete-and-create.
37. **Replication adds ~12 MiB of metered metadata** to the destination. Trivial
    in cost, but it means the destination is never reported as byte-identical to
    the source — don't build a monitoring check on size equality.
38. **`DeleteReplicationConfiguration` targets the source file system, which
    lives in the Region you are failing away from.** Whether it succeeds during
    a primary-Region control-plane outage is **not verified here** and is the
    most important thing to test in a gameday.
39. **Replication does not support tags for attribute-based access control
    (ABAC)** — stated explicitly in the AWS docs. If the estate's IAM strategy
    is ABAC-based, replication configurations need a different authorisation
    path. Cross-ref [[aws-iam]].

---

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| **EBS-backed state in a 15m-RTO estate** | Keep it; snapshot-copy to standby; accept RTO failure for those workloads | Push state into managed services or EFS; keep EBS only for derived/disposable data | **B.** No configuration of EBS meets 15m, and the FSR arithmetic proves it is not buyable. |
| **KMS key for EBS snapshot copies** | Separate per-Region CMKs | Multi-Region key with a replica in the standby | **B.** A different key ID makes every copy a full copy, forever. |
| **EBS snapshot scheduler** | DLM (free, native, tag-aware, ±1h jitter) | EventBridge Scheduler + Lambda (no jitter, you own everything) | **A** for the 3–24h cadences that are appropriate for EBS data. B only if a sub-2h RPO on a block volume is genuinely non-negotiable — and ask why first. |
| **Fast Snapshot Restore** | Enable on standby copies to fix lazy hydration | Do not use it | **B**, except for at most 1–2 irreplaceable volumes. Capped at 5/Region, $540/AZ/month each, 60 min/TiB to optimize, doesn't survive a copy, leaks cost on teardown. |
| **EFS throughput mode on the standby** | Mirror the primary's mode | Elastic on the standby regardless | **B.** The standby's average-to-peak ratio is zero, which is AWS's own definition of the Elastic use case. It is also immune to the lifecycle/`ValueInStandard` trap. |
| **EFS lifecycle policy on the standby** | Aggressive IA/Archive (AWS cites up to 75% saving with One Zone + 7-day age-off) | Disabled | **Split by workload.** Disabled on the RTO-critical path; aggressive for archival file systems that aren't part of promotion. Never rely on the default — set it explicitly. |
| **EFS standby storage class** | Regional (3 AZs) | One Zone (1 AZ, cheapest) | **Regional** for anything you fail over to. One Zone puts your entire DR copy in a single AZ. |
| **Destination EFS file system ownership** | Let the replication configuration create it implicitly | Pre-create it with `aws_efs_file_system` behind the standby alias | **B.** Otherwise it is unmanaged, untagged, undestroyable and you cannot control its throughput mode or lifecycle. |
| **Who performs EFS failover** | `terraform apply` with the replication module toggled off | `aws efs delete-replication-configuration` from the runbook, reconcile Terraform after | **B** for the 15-minute RTO; keep `prevent_destroy` on the Terraform resource so an accidental plan can't do it. |
| **EFS failback mode** | Preserve replica writes (reverse-replicate, two initial syncs) | Discard replica writes (one initial sync, much faster) | **Decide per file system, in advance, and write it down.** Read-mostly file systems should choose discard. |
| **Kubernetes PVCs** | EBS CSI (`ReadWriteOnce`, AZ-pinned) | EFS CSI (`ReadWriteMany`, Region-wide) | **EFS for shared/file-shaped state; managed services for database-shaped state; EBS only for disposable state.** See [[eks-stateful-workloads]]. |
| **FSx for NetApp ONTAP** | Adopt for its 5-min RPO and first-class SnapMirror failback | Stay on EFS | **B.** A 5-min RPO versus 15-min is meaningless against a 2h target, and the cost is a whole second storage platform. Revisit only if measured failback time proves unacceptable. |

---

## Cost

**What is honestly known, and what is not.** AWS renders the rate tables on both
[EBS pricing](https://aws.amazon.com/ebs/pricing/) and
[EFS pricing](https://aws.amazon.com/efs/pricing/) client-side, and this
research pass could not read the per-GB figures from either page. **No per-GB
storage, IOPS, throughput or snapshot prices are quoted in this note**, because
inventing them would be worse than omitting them. Pull real per-Region rates
from the [AWS Pricing Calculator](https://calculator.aws/) before building
[[cost-model]].

**What AWS does state in prose, and is therefore safe to use:**

| Item | Figure | Source |
|---|---|---|
| **Fast Snapshot Restore** | **$0.75 per snapshot per AZ per hour**; "$540 (1 snapshot x 1 AZ x 720 hours x $0.75 per hour)" for a 30-day month; "$3240" for 2 snapshots × 3 AZs | [FSR pricing](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-fast-snapshot-restore.html#fsr-pricing) |
| **Data Lifecycle Manager** | **No additional cost** | [DLM docs](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-lifecycle.html) |
| **EFS replication metadata** | **~12 MiB** of metered data on the destination | [EFS replication docs](https://docs.aws.amazon.com/efs/latest/ug/efs-replication.html#efs-replication-costs) |
| **EFS cross-AZ data transfer** | **$0.01/GB** | [EFS pricing](https://aws.amazon.com/efs/pricing/) |
| **EFS DR cost optimisation** | "save **up to 75%** on your disaster recovery storage costs by using low-cost EFS One Zone storage classes and a 7-day age-off lifecycle management policy for your destination file system" | [AWS Storage Blog](https://aws.amazon.com/blogs/storage/use-cases-for-amazon-efs-replication/) |
| **EBS mount target** | No charge for the mount target itself | [EFS pricing](https://aws.amazon.com/efs/pricing/) |

**The FSR arithmetic, which is the one number that decides an architecture:**

| Scenario | Monthly cost |
|---|---|
| 1 snapshot, 1 AZ | $540 |
| 1 snapshot, 3 AZs | $1,620 |
| **5 snapshots (the per-Region cap), 3 AZs** | **$8,100 per standby Region** |
| Three standby Regions at the cap | **$24,300/month** to accelerate at most 15 snapshots |

**The cost shape, qualitatively:**

- **EFS standby**: dominated by a second full copy of the data. Irreducible.
  Everything else (mount targets, access points, replication metadata) is free
  or negligible. Elastic throughput on an idle standby costs essentially nothing
  because an idle standby reads and writes essentially nothing.
- **EBS standby**: snapshot storage for changed blocks, plus cross-Region data
  transfer per copy cycle, plus the first full copy of every volume. **Plus
  whatever full copies you are accidentally paying for** because retention is
  too short or the key changed. No compute cost, because no volumes exist.
- **Levers, in order of size**: (1) don't use FSR; (2) keep copies incremental
  (multi-Region KMS key, destination retention > copy interval, no archiving of
  recent copies); (3) lengthen the snapshot cadence for data that doesn't need
  2h; (4) One Zone + lifecycle on non-critical EFS replicas; (5) delete the
  EBS-backed state entirely, which is the recommendation below and also the
  largest lever by far.

---

## Recommendation: should EBS-backed state exist at all?

**Short answer: no, not on the RTO-critical path, and the arithmetic in this
note is the argument.**

The case is not aesthetic. It is four verified facts that compound:

1. **EBS is AZ-scoped**, so cross-Region means snapshots, full stop.
2. **DLM's fastest interval is 1 hour and it fires within ±1 hour of schedule**,
   so the RPO floor is unreliable at 2h before copy time is even counted.
3. **Restored volumes are lazily hydrated**, so a "successful" failover serves
   degraded performance for an unbounded period.
4. **The only fix for (3) is FSR, which is capped at 5 snapshots per Region,
   costs $540 per snapshot per AZ per month, takes 60 minutes per TiB to
   optimize, does not survive a cross-Region copy, and silently no-ops past the
   cap.** You cannot buy your way out. The cap settles it before the price does.

Any one of these is survivable. Together they mean there is no configuration of
EBS that meets RTO 15m for a meaningful number of volumes.

### Per workload type

| Workload | Today (likely) | Target | Why | Cost direction |
|---|---|---|---|---|
| **Relational database** (Postgres) | EBS volume on EC2/EKS | **RDS or Aurora** — [[aws-rds-postgres]], [[aws-aurora-global-database]] | Aurora Global Database gives cross-Region replication with a promotion path that fits 15m. EBS snapshots do not. `ca-west-1` passes for Aurora Global DB. | **Up** on instance cost, **down** on engineering and incident cost. The RTO is not otherwise purchasable. |
| **Cache / session store** (Redis) | EBS-backed self-managed Redis | **ElastiCache** — [[aws-elasticache-redis]] | Cache state should be rebuildable; where it isn't, ElastiCache has its own cross-Region story. | Neutral to down; you stop paying for EC2 + EBS + operator time. |
| **Search / analytics** | EBS data nodes | **OpenSearch Service** — [[aws-opensearch]] | Note `ca-west-1` fails the OpenSearch CCR parity check; the CA pair needs a different answer. | Up on licence-equivalent cost, down on DR engineering. |
| **Shared application files** (uploads, assets, shared config, plugin dirs) | EBS + rsync, or EFS already | **EFS with Replication** | RPO 15 min published, RTO minutes, `ca-west-1` passes. This is the good path. | A second full copy of the data. Irreducible and modest. |
| **Object-shaped data** (anything accessed by key, not by path) | EBS or EFS | **S3 with CRR** — [[aws-s3]] | Cheapest, simplest, best cross-Region story on AWS. If it can be an object, make it an object. | **Down**, usually substantially. |
| **Queue / event state** | EBS-backed broker | **SQS / SNS / EventBridge / MSK** — [[aws-sqs]], [[aws-sns]], [[aws-eventbridge]] | See [[messaging-in-flight-data-loss]] for what is genuinely lost at cutover. | Down. |
| **Build caches, scratch, CI workspaces, log spool** | EBS | **EBS — and keep it.** Do not replicate it. | Derived data has no RPO. The standby rebuilds it, slower, once. | **Down** — you stop paying for cross-Region snapshot copy on data that didn't need it. |
| **Boot / root volumes** | EBS | **EBS, via AMIs** | Rebuild from an AMI copied to the standby, not from a data snapshot. Immutable infrastructure is the mechanism. See [[aws-eks]]. | Neutral. |

### The honest caveat

**This recommendation costs money in the short term and saves it in the long
term.** Moving a self-managed Postgres to Aurora is not free, and "migrate your
databases" is a bigger programme than "add a DLM policy". Someone will propose
the DLM policy as the pragmatic interim, and they will be right — **do that
too**, at a 12- or 24-hour cadence, as a safety net for the corruption and
ransomware cases that replication cannot address at all ([[aws-backup]]).

But it must be labelled correctly. **A cross-Region EBS snapshot copy is a
backup, not a warm standby.** The failure mode this note is really guarding
against is an estate that ships a DLM policy, ticks "multi-region storage: done"
on the programme plan, and discovers during the first real incident that the
box was ticked against a mechanism that was never capable of 15 minutes. Write
the DLM policy. Do not let anyone call it DR.

### What to do first

1. **Inventory EBS volumes by bucket** (transactional / shared-file / derived).
   Most estates find the derived bucket is far larger than expected, and it
   needs no work at all.
2. **Move the shared-file bucket to EFS with Replication.** It is the cheapest
   win available and it meets both targets.
3. **Put a 12–24h DLM policy with cross-Region copy on everything else** as a
   backup, not as DR, with a multi-Region KMS key and destination retention
   longer than the copy interval.
4. **Book the managed-service migrations** for the transactional bucket as
   actual projects with actual dates.
5. **Grep for hardcoded `fs-`, `fsap-` and `vol-` IDs** before any of the above
   is declared done.

---

## Open questions

Things that need an answer from inside the company or from a gameday, and that
no amount of further desk research can resolve:

1. **How much EBS-backed state actually exists, and in which bucket?** The
   recommendation above is only actionable against an inventory.
2. **What is the EFS initial-sync throughput for our file systems?** AWS
   publishes no figure. This is the input to every failback plan and can only be
   measured. **Highest-value gameday objective in this note.**
3. **Does `DeleteReplicationConfiguration` succeed when the primary Region's
   control plane is degraded?** The call targets the source file system, in the
   failing Region. If it does not, the documented failover path does not work in
   the scenario it exists for. **Highest-value gameday risk in this note.**
4. **Are any file systems in the >100M-files / >100GB-files exception** to the
   15-minute RPO? Check `TimeSinceLastSync` distributions on the real workloads
   rather than assuming.
5. **Does any application rely on cross-file write ordering?** EFS Replication is
   explicitly not point-in-time consistent; this is a correctness question, not
   a performance one.
6. **Are the live EFS file systems encrypted?** If not, there is no in-place
   remediation — it forces a new file system and a data migration. Find out now.
7. **Does `ca-west-1` have the same EFS throughput quota ceilings as
   `ca-central-1`?** AWS documents per-Region maximums and grants increases
   "on a case-by-case basis". A young Region plausibly has lower defaults.
   **Not verified in this pass** — check Service Quotas directly in both.
8. **What are the actual EBS and EFS per-GB rates in `eu-west-2`, `us-west-2`
   and `ca-west-1`?** Not readable from AWS's pricing pages in this pass.
9. **15 minutes from incident start, or from decision-to-fail-over?** The
   standing question from `CLAUDE.md`. For EFS it is the difference between
   comfortable and tight. For EBS it makes no difference — it fails either way.
10. **Where are `fs-`, `fsap-` and `vol-` IDs hardcoded?** One grep, potentially
    the single biggest failover blocker in the estate.

---

## Sources

Every URL below was fetched during this research. Where AWS's own documentation
was self-contradictory (the incremental-copy question) both statements are
quoted in the body and the contradiction is resolved explicitly rather than
smoothed over.

**EBS**

- [Copy an Amazon EBS snapshot — AWS docs](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-copy-snapshot.html)
  — the incremental-vs-full contradiction and its four conditions, the
  20-concurrent-copy limit, tags not copied, arbitrary `vol-ffff` IDs, the six
  KMS permissions needed for copy, the silent failure on inaccessible keys,
  time-based copies in 15-minute increments, FSR not surviving a copy.
- [Amazon EBS fast snapshot restore — AWS docs](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-fast-snapshot-restore.html)
  — lazy-hydration framing, the 5-snapshots-per-Region cap, 16 TiB limit, volume
  creation credits, and the **$0.75/hour → $540/month** worked pricing example.
- [Quotas for Amazon EBS — AWS docs](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-resource-quotas.html)
  — 100,000 snapshots per Region, FSR = 5 in every Region **including
  `ca-west-1`**, time-based copy throughput of 2,000 MiB/s per destination
  Region, st1/sc1 limited to 1 concurrent snapshot, `io2` capped at 20 TiB,
  `ca-west-1`'s higher 300 TiB volume quotas, the 500-vs-2,500 volumes-per-launch
  split, and AWS's note that it silently raises quotas in Regions you use.
- [Automate backups with Amazon Data Lifecycle Manager — AWS docs](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-lifecycle.html)
  — DLM is free; 100 custom policies per Region.
- [Create a DLM custom policy for EBS snapshots — AWS docs](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-ami-policy.html)
  — **the "within one hour of their scheduled time" jitter**, 4 schedules per
  policy, 3 cross-Region copy destinations, shared retention type across
  schedules, all-or-nothing tag copying, FSR at **60 minutes per TiB** to
  optimize, FSR staying enabled after policy deletion, FSR silently no-opping
  past the cap, archive rules and the 90-day minimum, orphaned snapshots on
  volume deletion.
- [Amazon DLM adds support for 1 hour backup interval — AWS What's New, March 2020](https://aws.amazon.com/about-aws/whats-new/2020/03/amazon-data-lifecycle-manager-adds-support-for-1-hour-backup-interval)
  — confirms the interval floor is 1 hour and the valid set 1/2/3/4/6/8/12/24.
- [Amazon EBS increases concurrent snapshot copy limits to 20 per destination Region — AWS What's New](https://aws.amazon.com/about-aws/whats-new/2020/04/amazon-ebs-increases-concurrent-snapshot-copy-limits-to-20-snapshots-per-destination-region/)
  — provenance for the 20-copy figure.
- [Amazon EBS volumes — AWS docs](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volumes.html)
  — the AZ-scoping constraint that the whole EBS half of this note rests on.
- [Amazon EBS pricing](https://aws.amazon.com/ebs/pricing/)
  — gp3's included 3,000 IOPS / 125 MB/s baseline, and the structure of io2
  tiered IOPS billing. **Per-GB rate tables are rendered client-side and were
  not readable**; no per-GB EBS price is quoted in this note.

**EFS**

- [Replicating EFS file systems — AWS docs](https://docs.aws.amazon.com/efs/latest/ug/efs-replication.html)
  — **the published RPO of 15 minutes**, the >100M-files / >100GB exception, the
  explicit "not … point-in-time consistent" statement, **"Replication is
  available in all AWS Regions in which Amazon EFS is available"** (the
  `ca-west-1` answer), initial-sync behaviour on both creation and reversal, the
  ~12 MiB metadata charge, required IAM permissions, `TimeSinceLastSync`, and
  the ABAC limitation.
- [Using the replica (EFS failover / failback) — AWS docs](https://docs.aws.amazon.com/efs/latest/ug/replication-fail-over.html)
  — failover by **deleting the replication configuration**, the destination being
  read-only until then, and both failback branches (discard vs preserve replica
  changes) quoted in full.
- [Amazon EFS performance specifications — AWS docs](https://docs.aws.amazon.com/efs/latest/ug/performance.html)
  — Elastic/Provisioned/Bursting comparison and per-mode maximums, "you don't
  accrue or consume burst credits while using Elastic throughput", the burst
  credit formula (50 KiBps/GiB baseline, 100 MiBps/TiB burst, **2.1 TiB max
  balance ≈ 12 hours of continuous burst**), the `ValueInStandard` dependency
  that creates the lifecycle trap, and the **24-hour lock-out** after switching
  to Provisioned.
- [Managing storage lifecycle — AWS docs](https://docs.aws.amazon.com/efs/latest/ug/lifecycle-management-efs.html)
  — the three lifecycle policies and their 30/90-day defaults, "Transition into
  Standard" defaulting to off, metadata always staying in Standard, writes to
  IA/Archive files landing in Standard first, and lifecycle operations being
  lower priority than workload operations.
- [Managing mount targets — AWS docs](https://docs.aws.amazon.com/efs/latest/ug/accessing-fs.html)
  — one mount target per AZ, mount targets being per-VPC, VPC and IP changes
  requiring delete-and-recreate, and One Zone's single mount target.
- [Working with access points — AWS docs](https://docs.aws.amazon.com/efs/latest/ug/efs-access-points.html)
  — POSIX identity and root-directory enforcement, access points requiring mount
  targets and inheriting their AZ placement, security groups applying at the
  mount target level, the `accesspoint=fsap-…` mount syntax, and the
  `elasticfilesystem:AccessedViaMountTarget` condition key.
- [Use cases for Amazon EFS Replication — AWS Storage Blog](https://aws.amazon.com/blogs/storage/use-cases-for-amazon-efs-replication/)
  — that the destination's lifecycle policy, backup policy, provisioned
  throughput, mount targets and access points are configured **independent of
  the source**, that lifecycle is off on the destination by default, and the
  **"up to 75%"** DR saving with One Zone + a 7-day age-off policy.
- [Amazon EFS is now available in the AWS Canada West (Calgary) Region — AWS What's New, Feb 2024](https://aws.amazon.com/about-aws/whats-new/2024/02/amazon-efs-aws-canada-west-calgary-region/)
  — the first half of the `ca-west-1` parity proof.
- [Amazon EFS pricing](https://aws.amazon.com/efs/pricing/)
  — $0.01/GB cross-AZ data transfer and the "TCO as low as $0.0315/GB" headline.
  **Per-GB storage-class rate tables are rendered client-side and were not
  readable**; no EFS per-GB price is quoted in this note.

**FSx**

- [Cross-region disaster recovery with Amazon FSx for NetApp ONTAP — AWS Storage Blog](https://aws.amazon.com/blogs/storage/cross-region-disaster-recovery-with-amazon-fsx-for-netapp-ontap/)
  — "SnapMirror enables you to configure replication with an **RPO of as low as
  five minutes, and an RTO in single digit minutes**", the 5-minute minimum
  schedule, failover being manual, and breaking the SnapMirror relationship
  making the destination writable.

**Terraform**

- [`aws_efs_replication_configuration` — Terraform AWS provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/efs_replication_configuration)
  ([source markdown](https://github.com/hashicorp/terraform-provider-aws/blob/main/website/docs/r/efs_replication_configuration.html.markdown))
  — **"Upon deletion, replication stops and the destination file system loses
  its read-only status, though it remains intact"** (i.e. destroy = failover),
  the `destination` block arguments (`region`, `availability_zone_name`,
  `file_system_id`, `kms_key_id`), the KMS default of
  `/aws/elasticfilesystem`, the exported `destination[0].file_system_id` and
  `status`, and the 20-minute create/delete timeouts.
- [`aws_dlm_lifecycle_policy` — Terraform AWS provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/dlm_lifecycle_policy)
  ([source markdown](https://github.com/hashicorp/terraform-provider-aws/blob/main/website/docs/r/dlm_lifecycle_policy.html.markdown))
  — `create_rule` valid intervals (1/2/3/4/6/8/12/24, `HOURS` only),
  `cross_region_copy_rule` arguments (`target`, `encrypted`, `cmk_arn`,
  `copy_tags`, required `retain_rule`), and `fast_restore_rule`.
- [`aws_efs_file_system` — Terraform AWS provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/efs_file_system)
  — the argument surface. **Note: the provider docs do not annotate which
  arguments force replacement**, which is why this note tells you to read the
  plan rather than asserting a ForceNew list.

**Kubernetes**

- [`kubernetes-sigs/aws-efs-csi-driver` — GitHub](https://github.com/kubernetes-sigs/aws-efs-csi-driver)
  — the `ReadWriteMany` EFS CSI driver.
- [Use Kubernetes volume storage with Amazon EBS — Amazon EKS docs](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)
  — the EBS CSI driver as an EKS managed add-on, and its `ReadWriteOnce` /
  AZ-pinned characteristics.

**Not found — recorded as findings, not gaps**

- **No published RTO figure for EFS Replication.** AWS publishes an RPO (15
  minutes) and documents the failover mechanism, but states no RTO on either the
  replication or the failover page. Do not let anyone cite one.
- **No published EFS initial-sync throughput or duration figure.** AWS says only
  that it "depends on factors such as the size of the source file system and the
  number of files in it."
- **No published EBS cross-Region snapshot copy duration or SLA.** AWS describes
  copies as "best-effort" unless time-based copy is enabled.
- **No public postmortem or case study found** describing a real production
  cross-Region EFS failover or an EBS-snapshot-based regional recovery with
  measured timings. Searched; nothing citable located. The timings in this note
  are therefore derived from AWS's own published mechanics and arithmetic, never
  from an unsourced anecdote.
- **No verified cross-Region RPO/RTO figures for FSx for Windows File Server or
  FSx for OpenZFS.** Out of scope per the research brief and not researched;
  do not quote any.
