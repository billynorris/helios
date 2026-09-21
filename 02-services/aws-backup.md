---
title: AWS Backup — Multi-Region
service: aws-backup
tags: [service, multi-region, aws-backup, dr, compliance, ransomware]
status: researched
replication: native (scheduled cross-Region copy jobs) — not continuous
rpo_achievable: "2h achievable for continuous-backup resources copied hourly; 4–8h realistic for snapshot-based resources"
rto_achievable: "hours to days — fails RTO 15m for every resource type"
meets_targets: conditional — meets RPO 2h with effort, fails RTO 15m by design
updated: 2026-09-21
---

# AWS Backup — Multi-Region

## TL;DR

- **AWS Backup is not the RTO mechanism and should never be sold internally as
  one.** Restoring a meaningful RDS Postgres instance, EFS file system or S3
  bucket from a recovery point takes hours, sometimes days. There is no
  configuration of AWS Backup that reaches 15 minutes. The warm standby
  documented in [[aws-rds-postgres]], [[aws-dynamodb]], [[aws-s3]] and
  [[aws-efs-ebs]] is the RTO mechanism. AWS Backup sits underneath it.
- **What AWS Backup is for: the RPO floor, the compliance evidence layer, and
  the corruption/ransomware answer.** Per-service replication faithfully
  replicates a `DROP TABLE`, a bad migration, an `aws s3 rm --recursive`, and an
  encryption payload. It replicates them in seconds, to the standby, correctly.
  A backup does not. That asymmetry is the entire reason this note exists.
- **Cross-Region copy is a first-class feature of a backup plan** (a
  `copy_action` on a rule) and works for almost every resource type — but
  **plain DynamoDB is the glaring gap**: DynamoDB recovery points cannot be
  copied cross-Region or cross-account at all unless you opt into *Advanced
  DynamoDB backup*. Verified against AWS's own feature-availability matrix.
- **Vault Lock compliance mode is irreversible.** After the cooling-off period
  (`ChangeableForDays`, minimum 3 days) expires, neither you, nor your root
  user, nor AWS can remove the lock or delete a recovery point before its
  lifecycle completes. Teams enable this because it sounds like the responsible
  option and then discover they have committed to paying for storage for the
  full retention period. Use governance mode unless a regulator is asking.
- **`ca-west-1` (Calgary) fails the parity check again, twice.** AWS Backup
  Audit Manager is **not available in Calgary**, and **logically air-gapped
  vaults are not available in Calgary** — so the CA pair gets neither the
  compliance-evidence layer nor the strongest ransomware control that the EU and
  US pairs get. See [[region-pair-selection]].

---

## The framing: replication is not backup

This note is positioned differently from the rest of `02-services/`. Every other
service note asks "how do I get this data into the standby region fast enough."
This one asks "what happens when the thing you replicated was wrong."

Set out plainly:

| Failure mode | Cross-Region replication | AWS Backup |
|---|---|---|
| Region loses power / AZ-wide failure | **Protects you.** This is what it's for. | Too slow to help. |
| Instance/cluster hardware failure | Protects you. | Too slow. |
| `DROP TABLE users;` on the primary | **Replicates the drop.** Standby is now also missing the table, in seconds. | **Protects you.** Yesterday's recovery point still has the table. |
| Bad schema migration deployed to prod | **Replicates the migration.** | **Protects you.** |
| `aws s3 rm s3://bucket --recursive` | S3 CRR replicates deletes if configured to; even with delete-marker replication off, the source objects are gone. | **Protects you.** |
| Ransomware encrypts the data in place | **Replicates the ciphertext.** Both regions now hold encrypted garbage. | **Protects you — if the backup vault is out of the blast radius.** |
| Credential compromise, attacker deletes the RDS instance *and* the standby | Standby is in the same account, same IAM trust boundary. Gone. | **Protects you only if backups live in a separate account with a vault lock.** |

Rows 3–7 are the reason to run AWS Backup at all. If the only threat model were
"a region falls over," the per-service replication already documented in this
vault would be sufficient and AWS Backup would be pure cost. It is not the only
threat model, and the bottom two rows are the ones that end companies rather
than merely embarrassing them. See [[lessons-and-antipatterns]] and
[[aws-regional-outages]].

The corollary matters as much as the claim: **because AWS Backup answers a
different question, it is not in competition with the replication work.** Do not
let a "we have AWS Backup copying to the standby region, do we still need Aurora
Global Database?" conversation happen. The answer is yes, and this note exists to
make that argument in writing, once, so it doesn't have to be re-litigated.

---

## Does this service cross regions at all?

Yes, natively, and this is one of the better-designed cross-Region stories in
AWS — but with a specific shape you need to internalise.

**Regional resources.** A backup vault is a regional resource. Its ARN is
`arn:aws:backup:eu-west-1:123456789012:backup-vault:my-vault`. A backup plan is
regional. A recovery point lives in exactly one vault in exactly one region.
There is no "global vault."

**Crossing regions is a copy, not a replica.** Unlike Secrets Manager (declare a
replica region, AWS mirrors it) or DynamoDB Global Tables (continuous
multi-master replication), AWS Backup crosses regions by running a discrete
**copy job** that takes a recovery point in region A and produces a *new,
separate* recovery point in region B. Two recovery points, two ARNs, two
lifecycles, two retention clocks, two KMS keys. The copy is a first-class
recovery point in the destination vault, not a pointer back to the source.

That has a consequence people miss: **deleting the source recovery point does
not delete the copy, and the copy's retention is set independently** by the
`lifecycle` block inside the `copy_action`. This is good — it's what lets you
keep 7 days locally and 35 days in the DR region — but it means your retention
policy is expressed in two places and they will drift.

**The copy is a *push* from the source region.** The copy job is orchestrated by
AWS Backup in the source region, and it is triggered by the backup plan that
lives in the source region. This is the single most important operational fact in
this note and it is developed in full under
[If the primary region is gone, so is your copy scheduler](#if-the-primary-region-is-gone-so-is-your-copy-scheduler).

**First copy is full, subsequent copies are incremental — mostly.** From the AWS
docs: "When you copy a backup to a new AWS Region for the first time, AWS Backup
copies the backup in full. In general, if a service supports incremental
backups, subsequent copies of that backup in the same AWS Region will be
incremental." There is an explicit exception: **Amazon EBS**, where copying a
snapshot into a vault using a *different* KMS key produces a **full, not
incremental, copy** every time. Since a cross-Region copy by definition lands in
a destination vault with a different key (a KMS key is regional — see
[[aws-kms]]), the naive reading is "EBS cross-Region copies are always full."
The doc's wording is "If you consistently copy to the same vault with the same
encryption key, subsequent copies remain incremental" — in practice, once the
destination vault's key is fixed, copies to it stabilise as incremental. Treat
the *first* copy of every EBS volume into the standby as a full-volume transfer
and budget the data-transfer bill and the wall-clock time accordingly.

**Cold storage cannot be copied cross-Region.** Direct from the docs: "AWS Backup
does not support cross-Region copies for storage in cold tiers." If your
lifecycle transitions a recovery point to cold storage on day 8, you cannot
initiate a cross-Region copy of it on day 20. Copy first, tier later — and note
that the copy's own lifecycle governs when *the copy* tiers to cold, independent
of the source.

---

## Which resource types actually support cross-Region copy

This is the section to verify rather than assume, and the gaps are the finding.
Reproduced from AWS's own
[feature availability by resource](https://docs.aws.amazon.com/aws-backup/latest/devguide/backup-feature-availability.html)
matrix, restricted to the resource types plausibly in this estate:

| Resource type | Cross-Region copy | Cross-account copy | Incremental | Continuous backup / PITR | Full AWS Backup management | Cold storage | Restore testing |
|---|---|---|---|---|---|---|---|
| Amazon EC2 | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| Amazon EBS | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ |
| Amazon S3 | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Amazon RDS DB instance | ✅ ¹ | ✅ ¹ | ✅ | ✅ | ❌ | ❌ | ✅ |
| Amazon RDS **Multi-AZ cluster** | ✅ ¹ | ✅ ¹ | ✅ | ❌ ² | ❌ | ❌ | ✅ |
| Amazon Aurora | ✅ ¹ | ✅ ¹ | ✅ ³ | ✅ | ❌ | ❌ | ✅ |
| Amazon EFS | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| **Amazon DynamoDB (plain)** | **❌** | **❌** | ❌ | ❌ | ❌ | ❌ | ❌ |
| DynamoDB **with Advanced DynamoDB backup** | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Amazon EKS | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | ✅ ⁴ |
| AWS CloudFormation | ✅ ⁵ | **❌** | ✅ ⁵ | ❌ | ❌ | ✅ | ❌ |
| Amazon DocumentDB | ✅ ¹ | ✅ ¹ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Amazon Neptune | ✅ ¹ | ✅ ¹ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Amazon Redshift Serverless | **❌** | **❌** | ❌ | ❌ | ❌ | ❌ | ❌ |
| Amazon FSx (all flavours) | ✅ | ✅ | ✅ | ❌ | ❌ | partial | ✅ |
| Amazon Timestream | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ |

¹ RDS, Aurora, DocumentDB and Neptune support cross-Region **and** cross-account
snapshot copy in a **single action** — you do not have to chain two copies.
² AWS Backup does not support continuous backup / PITR for RDS **Multi-AZ
clusters** (as distinct from Multi-AZ *instances*, which are supported).
³ Aurora snapshots are full; incremental is offered through PITR.
⁴ EKS restore testing is not available in Middle East (Bahrain) or (UAE).
⁵ CloudFormation stack backups: nested resources keep their own feature support,
but resources inside the stack **lose PITR**.

### The gaps, called out

**1. Plain DynamoDB cannot be copied anywhere. At all.** This is the big one and
it is easy to miss because the matrix has a separate row for the advanced
variant. AWS's own note on the matrix is unambiguous: *"If a resource type does
not have a checkmark in the Cross-Region backup or Cross-account backup columns,
then copy operations for that resource type are not supported in any scenario,
including same-Region and same-account copies to a different vault."* So without
Advanced DynamoDB backup you cannot even copy a DynamoDB recovery point into an
isolated vault **in the same region and same account**. The ransomware
blast-radius control is unavailable. Since the brief says DynamoDB is already
in-flight toward Global Tables, and Global Tables is replication (which
replicates deletion), **enabling Advanced DynamoDB backup is a prerequisite, not
a nice-to-have** — otherwise DynamoDB is the one datastore in the estate with
replication and no independent backup copy. See [[aws-dynamodb]] and
[[dynamodb-table-naming-migration]].

Advanced DynamoDB backup is opted into per-region via
`aws_backup_region_settings` (`resource_type_management_preference` for
`DynamoDB`). Note the trade: it gives you cross-Region/cross-account copy, cold
storage tiering, independent KMS encryption and restore testing, but the matrix
shows **no incremental** — DynamoDB backups are full each time under AWS Backup.
Cost implication under [Cost](#cost).

**2. CloudFormation stack backups cannot cross accounts.** Cross-Region ✅,
cross-account ❌. If the isolated-account pattern is your control, CloudFormation
stack backups are outside it.

**3. Redshift Serverless supports neither.** Not believed to be in this estate;
flagged for completeness.

**4. Several things in scope for this project are simply not AWS Backup resource
types at all.** SQS, SNS, EventBridge, Lambda, API Gateway, ACM, Route 53, ALB/
NLB, CloudFront, ECR, KMS, Secrets Manager, SSM Parameter Store, IAM, VPC. AWS
Backup does not back these up. Their "backup" is Terraform state plus the
per-service notes ([[aws-sqs]], [[aws-eventbridge]], [[aws-secrets-manager]],
[[aws-ssm-parameter-store]], [[aws-ecr]], [[aws-iam]], [[aws-vpc-networking]]).
**This is worth saying out loud to anyone who believes "AWS Backup covers us":
it covers the data plane of a specific list of storage services and nothing
else.** Configuration drift in the standby region is not an AWS Backup problem;
it is a [[module-patterns]] problem.

### `ca-west-1` (Calgary) parity — checked explicitly

From the same AWS page's *Feature availability by AWS Region* table:

| Feature | `eu-west-1` | `eu-west-2` | `us-east-1` | `us-west-2` | `ca-central-1` | **`ca-west-1`** |
|---|---|---|---|---|---|---|
| Opt-in required | No | No | No | No | No | **Yes** |
| Cross-Region backup copy | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Cross-account management | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Cross-account backup copy | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **AWS Backup Audit Manager + Jobs dashboard** | ✅ | ✅ | ✅ | ✅ | ✅ | **❌** |
| Restore testing | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Logically air-gapped vault (as copy target)** | ✅ | ✅ | ✅ | ✅ | ✅ | **❌** ⁶ |
| Backup search | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Backup tiering | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Malware Protection | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Backup access points | ✅ | ✅ | ✅ | ✅ | ✅ | **❌** |

⁶ The region table shows a checkmark for "logically air-gapped vault" in
Calgary, but footnote 1 on that table states: *"Cross-Region and cross-account
copy to a logically air-gapped vault is not currently available in Asia Pacific
(Malaysia), **Canada West (Calgary)**, Mexico (Central), …"* — and separately the
[logically air-gapped vault page](https://docs.aws.amazon.com/aws-backup/latest/devguide/logicallyairgappedvault.html)
lists Calgary under "Not currently available." Since a LAG vault's entire purpose
here is to be a cross-Region/cross-account copy target, **treat LAG vaults as
unavailable for the CA pair.**

Also missing in Calgary from the *Supported services by Region* table:
**Amazon DocumentDB**, **SAP HANA on EC2**, **VMware/Backup gateway**,
**Amazon Timestream**, **Amazon Redshift Serverless**, and **Amazon RDS Multi-AZ
cluster** backups. (`ca-central-1` has all of those except Timestream.)
DocumentDB's absence is the one to check against the actual estate.

**The finding:** Calgary can receive cross-Region backup copies of the resource
types that matter here (EBS, EC2, RDS instances, Aurora, EFS, S3, DynamoDB-
advanced, EKS), and can do cross-account copy, and supports restore testing. What
it cannot do is **produce compliance evidence** (no Audit Manager, no Jobs
dashboard) or **be the target of the logically air-gapped vault pattern**. For a
project whose Canadian deployment presumably exists *because of* Canadian data
residency and therefore has the most regulator attention, losing the audit layer
in exactly that region is a genuinely awkward result. Options:

- **Run Audit Manager from `ca-central-1` only** and accept that the standby
  region's backup posture is unaudited by AWS-native tooling. Cheapest; the
  evidence gap is real but the *primary* region is where the protected resources
  live, so most controls still evaluate correctly there.
- **Substitute AWS Config + Security Hub in `ca-west-1`** for the subset of
  controls you care about. More work, but Config is available in Calgary and can
  express "resource is in a backup plan"-shaped rules.
- **Change the CA pair.** Moving to `us-*` breaks Canadian data residency
  outright (see [[data-residency]]). There is no second Canadian region other
  than `ca-central-1`/`ca-west-1`, so the pair is forced. **This is not a
  reason to change the pair — it's a reason to accept an evidence gap and
  document it.**

**Recommendation:** run Audit Manager in `ca-central-1`, add a small AWS
Config-based compensating control in `ca-west-1` covering "every recovery point
in the DR vault is encrypted" and "the DR vault has a lock," and write the gap
into the [[regulatory-drivers]] register rather than hiding it. Feed this into
[[region-pair-selection]] — it is the third Calgary failure recorded in this
vault, after Cognito multi-region replication, OpenSearch cross-cluster
replication, and Amazon Managed Grafana.

Note also: **Calgary is an opt-in region.** Cross-account *management* in opt-in
regions is degraded — from the docs, "delegated administrator accounts can launch
policies but do not have access to the monitoring functions," and if a child
account opts in before its management account there is a delay of **up to 24
hours** before cross-account monitoring shows job statuses. Budget for that
during enablement rather than debugging it.

---

## RPO analysis: can a scheduled copy meet 2 hours?

The target is RPO 2h. Do the arithmetic honestly, because the marketing answer
("AWS Backup copies to another region, so you're covered") is not the answer.

### Where the time actually goes

A cross-Region recovery point becomes usable only after **four** sequential
delays:

```
  t0   scheduled backup time (cron in the plan)
   +   START WINDOW          AWS Backup may start anywhere in this window
   +   BACKUP JOB DURATION   snapshot/continuous flush, source region
   +   COPY JOB DURATION     cross-Region transfer + re-encrypt
   =   t1   recovery point is available in the standby region
```

Your effective RPO for a given resource is **`t1 − (time of the last write
included in that recovery point)`**, which at worst is the full interval between
scheduled backups *plus* all three delays above.

**Term by term:**

**1. Backup frequency (cron).** `aws_backup_plan` takes a `schedule` cron
expression. Hourly is expressible: `cron(0 * * * ? *)`. This is the term you
control most directly and the one that dominates.

**2. Start window (`start_window`, minutes).** AWS Backup does not begin the job
at the cron instant; it begins it somewhere inside the start window, and it
randomises within that window to spread load. The console's "use backup window
defaults - recommended" hides this. **For an RPO-critical plan you want the start
window as small as the service permits, not the default**, because every minute
of start window is a minute of RPO you're donating to job scheduling. Check the
[AWS Backup quotas page](https://docs.aws.amazon.com/aws-backup/latest/devguide/aws-backup-limits.html)
for the current minimum before setting it — **not verified here**.

**3. Backup job duration.** Snapshot-based backups (EBS, RDS, Aurora) return
quickly at the API level but the snapshot is not *complete* (and therefore not
copyable) until the underlying service finishes it. For a large, busy RDS
instance this is minutes to tens of minutes. **AWS publishes no duration SLA or
typical-duration figure for backup jobs.** This is a measurement you must take in
your own estate, not a number you can look up. `completion_window` sets when AWS
Backup gives up on the job — it is a failure threshold, not a promise.

**4. Copy job duration.** The dominant unknown. It is a function of changed-bytes
since the last copy, cross-Region bandwidth, and AWS-side concurrency limits on
copy jobs. **AWS publishes no SLA, no typical duration, and no throughput figure
for cross-Region copy jobs.** Searching for a published benchmark returned
nothing authoritative — **no public data found**. Anyone quoting you a number for
this is guessing.

### The arithmetic, for real

Take an hourly plan with a 60-minute start window (a common default), against a
resource whose backup takes 10 minutes and whose cross-Region copy takes 30
minutes:

```
worst-case RPO = 60 (interval) + 60 (start window) + 10 (backup) + 30 (copy)
               = 160 minutes = 2h 40m        ❌ MISSES RPO 2h
```

Tighten the start window to 15 minutes:

```
worst-case RPO = 60 + 15 + 10 + 30 = 115 minutes = 1h 55m    ✅ just makes it
```

Now make the copy take 90 minutes instead of 30 — entirely plausible after a
heavy write day, or on the *first* copy of an EBS volume, or when AWS-side copy
concurrency is saturated because every plan in the account fires at the top of
the hour:

```
worst-case RPO = 60 + 15 + 10 + 90 = 175 minutes = 2h 55m    ❌ MISSES
```

**Conclusion: a scheduled AWS Backup cross-Region copy can meet RPO 2h, but only
with a tight start window, hourly scheduling, and measured copy durations — and
it has no headroom.** It is not a mechanism you should *rely* on for RPO 2h. It
is a mechanism that will usually meet RPO 2h and will silently miss it on the
days you most need it to hit.

**This is why AWS Backup is the RPO *floor* and not the RPO *mechanism*.** The
RPO mechanism is per-service async replication:

| Service | Replication RPO (per that note) | AWS Backup cross-Region copy RPO |
|---|---|---|
| RDS Postgres (cross-Region read replica) | seconds to low minutes | 2–3 hours |
| Aurora Global Database | typically ~1s | 2–3 hours |
| DynamoDB Global Tables | sub-second to seconds | 2–3 hours (advanced only) |
| S3 CRR (with RTC) | minutes | 2–3 hours |
| EFS replication | minutes | 2–3 hours |

Replication wins on RPO by two to three orders of magnitude. AWS Backup's job is
to be the *worst case you can still survive* when replication is the thing that
failed you — which, per the framing section, is exactly the corruption case.

### Continuous backup / PITR and what it does and doesn't buy you

For S3, RDS instances, Aurora, SAP HANA and Aurora DSQL, AWS Backup supports
**continuous backup** (`enable_continuous_backup = true` on the rule), giving
point-in-time restore to any moment inside the retention window (which for
continuous backup is capped at 35 days).

This looks like it solves the RPO problem. It does — **in the source region
only**. The critical sentence from the feature-availability page:

> "When a cross-Region or cross-account copy of a continuous backup is made, the
> copied recovery point (backup) becomes a snapshot (periodic) backup. PITR
> (Point-in-Time Restore) is not available for these copies."

So the shape is:

- **In `eu-west-1`:** PITR to any second in the last 35 days. Excellent RPO for
  the corruption case *as long as the primary region is healthy*. This is the
  configuration that actually answers "someone ran a bad migration at 14:07,
  restore to 14:06."
- **In `eu-west-2`:** discrete snapshots at whatever cadence your copy rule runs.
  No PITR. RPO is the copy interval.

Both are useful and they answer different questions. **Enable continuous backup
on the source plan *and* schedule cross-Region copies.** The continuous backup
handles logical corruption caught quickly (the overwhelmingly common case); the
cross-Region snapshot copies handle the region-plus-corruption case.

Also note this interaction with Vault Lock: a `min_retention_days` on a locked
vault is evaluated against a job's lifecycle retention, and continuous backup
caps at 35 days. A vault lock with `min_retention_days = 90` will cause
continuous-backup jobs into that vault to **fail**, not degrade. See
[Vault Lock](#backup-vaults-and-vault-lock).

---

## RTO analysis: why AWS Backup cannot meet 15 minutes

State this plainly and early in any internal conversation: **there is no
resource type for which restore-from-AWS-Backup meets a 15-minute RTO.**

### Where the restore time goes

A restore is not a data copy into an existing resource. For every resource type
AWS Backup supports, a restore **creates a brand new resource** and then
hydrates it:

1. **Control-plane provisioning.** A new RDS instance, a new EFS file system, a
   new EBS volume, a new S3 bucket. This alone is minutes to tens of minutes and
   is exactly the "provisioning from cold at failover time" that the research
   brief says the 15-minute RTO rules out.
2. **Data hydration.** For snapshot-backed block storage this is lazy — the
   volume is available quickly but reads fault in from S3 on first touch, so the
   restored resource is *available* long before it is *performant*. For S3 and
   file systems the restore is object-by-object or file-by-file and scales with
   object count as much as with bytes.
3. **Post-restore work that is yours, not AWS's.** DNS repointing, security
   group and subnet group attachment, parameter group reattachment, application
   configuration pointing at the new endpoint, connection pool restarts, and for
   databases any replication or extension re-setup. The restored resource has a
   **new identifier and a new endpoint**; nothing in your estate knows about it
   until you tell it.

Step 3 is routinely omitted from restore-time estimates and is frequently the
largest term. [[failover-orchestration]] should own it.

### Published figures, honestly

**AWS publishes essentially no restore-duration figures.** This is a genuine
finding, not a research gap — restore time depends on data volume, object count,
instance class and lazy-loading behaviour, so AWS declines to publish numbers it
would be held to.

| Resource type | AWS-published restore time | What actually dominates |
|---|---|---|
| Amazon EBS | **No published figure.** | Volume is usable quickly (snapshot-backed volumes hydrate lazily from S3); full performance requires initialisation or Fast Snapshot Restore. FSR is a *pre-warming* feature with its own cost and per-region, per-snapshot enablement — see [[aws-efs-ebs]]. |
| Amazon EC2 | **No published figure.** | AMI registration + instance launch + boot + config management convergence. |
| Amazon RDS / Aurora | **No published figure.** | New instance provisioning, then lazy-load from S3 for RDS; performance is degraded until blocks are touched. Large instances: tens of minutes to hours. See [[aws-rds-postgres]]. |
| Amazon EFS | **No published figure.** | Restores into a *new* or existing file system, file-by-file. Dominated by **file count**, not bytes. Millions of small files is the pathological case and can run into days. |
| Amazon S3 | **No published figure.** | Object-by-object restore; dominated by object count. A bucket with 10⁸ small objects is a very different restore from one with 10³ large ones. |
| Amazon DynamoDB | **No published figure.** | Table restore creates a new table; throughput of the restore is AWS-side. |
| Amazon FSx | **No published figure.** | New file system creation from backup. |

**The one place AWS gives you a number is the number you measure yourself.**
Restore testing (below) records restore job duration, and Audit Manager has a
control — "Restore time for resources meet target" — that evaluates measured
restore durations against a target you set. **That control is the only
trustworthy source of restore-time data for this estate, and nobody has run it.**
That is the direct answer to the question raised in [[lessons-and-antipatterns]]
about whether the company has ever timed an end-to-end restore: the tooling to
answer it exists, is cheap, and has not been turned on.

### The conclusion, stated for the record

> **AWS Backup does not and cannot meet RTO 15m. It is not a failover mechanism.
> The failover mechanism is the pre-provisioned warm standby described in
> [[failover-orchestration]]. AWS Backup is the mechanism for the class of
> incident where failing over to the standby would fail over to corrupted data.**

The RTO for a restore-from-backup scenario should be documented separately, as a
**second, much longer RTO for a different class of incident** — call it "RTO-C"
(corruption) versus "RTO-R" (region loss). Presenting a single 15-minute RTO
number to the business while the corruption scenario is a multi-hour or multi-day
restore is the kind of thing that ends up in a postmortem. Getting the business
to sign off on two different RTOs for two different incident classes is a
worthwhile political project and this note is the ammunition.

Related: the 15-minute RTO ambiguity flagged in `CLAUDE.md` (15 minutes from
*incident start* vs from *decision to fail over*) applies doubly here. For a
corruption incident, detection time alone routinely exceeds 15 minutes — you
cannot fail over from a `DROP TABLE` you haven't noticed yet.

---

## Backup vaults and Vault Lock

### Vaults

A **backup vault** is the container for recovery points. It has a name, a KMS
key, an optional resource-based **vault access policy**, and an optional **vault
lock**. It is regional. In a multi-region design you will have at least one vault
per region per account — commonly:

| Vault | Region | Purpose |
|---|---|---|
| `helios-<env>-primary` | `eu-west-1` | Where scheduled backups land. Short retention (7–14d). Governance lock. |
| `helios-<env>-dr` | `eu-west-2` | Cross-Region copy target. Longer retention (35d). Governance lock. |
| `helios-<env>-isolated` | `eu-west-2` (separate account) | Cross-account + cross-Region copy target. Long retention. **Compliance lock.** |

The vault access policy is worth using and usually isn't. It is a resource
policy on the vault and it is the *only* place you can express "nobody in this
account may delete recovery points from this vault," short of a vault lock. A
`Deny` on `backup:DeleteRecoveryPoint` for everything except a break-glass role
is a cheap, reversible first step before anyone reaches for compliance mode. See
[[security-posture-of-the-standby]].

### Vault Lock: the section to read twice

AWS Backup Vault Lock gives WORM semantics over a vault. Two modes, and the
difference between them is the difference between a control and a commitment.

| | **Governance mode** | **Compliance mode** |
|---|---|---|
| Who can remove the lock | Anyone with sufficient IAM permissions (`backup:DeleteBackupVaultLockConfiguration`) | **Nobody. Not you, not your root user, not AWS.** |
| Cooling-off period | N/A — removable at any time | `ChangeableForDays`, **minimum 3 days (72h)**, maximum **36,500 days** |
| Can you shorten retention later | Yes | No |
| Can you delete recovery points early | Yes (with permissions) | No — not until their lifecycle completes |
| Can you delete the vault | Yes | Only if it is **empty** |
| Terraform | `aws_backup_vault_lock_configuration` without `changeable_for_days` | `aws_backup_vault_lock_configuration` **with** `changeable_for_days` |
| Reversible mistake | Yes | **No** |

The API distinction is subtle and dangerous: `PutBackupVaultLockConfiguration`
creates a **governance** lock if you omit `ChangeableForDays`, and a
**compliance** lock if you include it. There is no `mode` parameter. In
Terraform, the presence or absence of a single optional argument —
`changeable_for_days` — is the difference between a reversible setting and a
permanent one. **A copy-pasted example, an over-eager `for_each`, or a variable
defaulting to a non-null value flips a vault into compliance mode, and after 72
hours nobody on earth can undo it.**

#### What "irreversible" actually means, spelled out

Quoting the AWS documentation directly, because paraphrase softens it:

> "Once the grace time expires, the vault and its lock are immutable and cannot
> be changed or deleted by any user or by AWS."

> "If any user (including the root user) attempts to delete a backup or change
> the lifecycle properties in a locked vault, AWS Backup will deny the
> operation."

> "Backups within a locked vault cannot be deleted until their lifecycle
> completes, resulting in persistent costs if you are not careful. For example,
> ensure there are no recovery points with a retention period set to 'Always' —
> once the grace time expires, these recovery points will be retained forever and
> cannot be altered or deleted."

Read that last one again. **A recovery point with retention set to "Always" in a
compliance-locked vault is a permanent, unbounded, unremovable line item on your
AWS bill.** You cannot delete it. You cannot delete the vault, because the vault
can only be deleted when empty. Your options are to pay the storage charge
forever, or to close the AWS account — and closing the account is the one
documented escape hatch:

> "When you close an AWS account that contains a backup vault, AWS and AWS Backup
> suspend your account for 90 days with your backups intact. If you do not reopen
> your account during those 90 days, AWS deletes the contents of your backup
> vault, even if AWS Backup Vault Lock was in place."

So the true statement is: *compliance mode is irreversible short of destroying
the entire AWS account and waiting 90 days.* If that vault is in the same account
as production, that is not an option. **This is a concrete argument for putting
compliance-locked vaults in a dedicated account that does nothing else** — see
the isolated-account pattern below. It is also, uncomfortably, a note that vault
lock does not protect against an attacker who can close your account.

#### Retention semantics

- `min_retention_days` — backup and copy jobs into this vault whose lifecycle
  retention is **shorter** than this value **fail**. Minimum settable: 1 day.
- `max_retention_days` — jobs whose retention is **longer** than this value
  **fail**. Maximum settable: 36,500 days (~100 years).
- Neither applies retroactively. From the docs: *"Recovery points already saved
  in the vault prior to the vault lock's creation are not affected"* and follow
  their previous lifecycle settings. So locking a vault does not retroactively
  extend the retention of what is already in it — a common misreading. If you
  need existing recovery points protected, the lock protects them from
  *deletion*, but their existing expiry dates still fire.
- These are **validation rules that cause job failures**, not clamps. A plan with
  `delete_after = 30` writing into a vault with `min_retention_days = 90` does
  not silently get 90-day retention; **the backup job fails**. This is a superb
  way to silently stop backing up production. See
  [the silent-gap problem](#backup-selections-and-the-silent-gap-problem).

#### The Cohasset assessment

AWS states that Backup Vault Lock "has been assessed by Cohasset Associates for
use in environments that are subject to SEC 17a-4, CFTC, and FINRA regulations,"
and links a Cohasset Associates Compliance Assessment. If a regulator or auditor
asks for third-party attestation that the WORM control is genuine, that document
is the artefact to hand over. It is linked from the
[Vault Lock page](https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html)
as a downloadable ZIP. Relevant to [[regulatory-drivers]] — note this is
US-regulator framing; whether it carries weight for UK/EU/Canadian regulators is
**not verified** and is a question for legal, not for this note.

#### A gotcha AWS documents and nobody reads

Vault Lock stops deletion. It does not stop a resource being made
*unrestorable*:

> "Mechanisms that render resources inactive can impact the ability to restore
> them. While they still cannot be deleted in a locked vault, they can be in a
> state other than active. For instance, the Amazon Elastic Compute Cloud setting
> that allows you to disable an AMI can temporarily block the ability to restore
> backups of EC2 instances. This affects all EC2 recovery points, even backups
> affected by a vault lock or a legal hold."

AWS's own mitigation: deny `ec2:DisableImage` via SCP or IAM policy. If EC2
recovery points are part of the DR story, add that to the SCP set alongside the
backup-protection SCPs.

#### Recommendation on modes

| Vault | Mode | Why |
|---|---|---|
| Primary-region operational vault | **Governance** | You will need to delete recovery points during normal operations — test data, decommissioned resources, mistakes. Governance mode plus a tight vault access policy gives you the control without the commitment. |
| DR-region copy vault | **Governance** | Same reasoning. The value here is geographic separation, not immutability. |
| Isolated-account vault | **Compliance**, with a deliberately chosen `changeable_for_days` and a hard-capped `max_retention_days` | This is the one vault whose entire purpose is to be un-deletable by a compromised production account. Compliance mode is the point. |

**Do not enable compliance mode anywhere until someone has computed the maximum
storage cost at the maximum retention and signed it off in writing.** Set
`max_retention_days` explicitly — never leave it unset — so that no future plan
can write an "Always" recovery point into the vault. Setting `max_retention_days`
is the single highest-value line in the whole configuration, because it is the
only thing standing between a misconfigured lifecycle and a permanent bill.

Set `changeable_for_days` longer than 3 if you can tolerate it. Three days is the
floor, not a recommendation. A 14- or 30-day cooling-off gives a real chance for
someone to notice a mistake before it becomes permanent — and there is no
downside beyond a slightly later start to immutability.

---

## The KMS story on cross-Region copy

A backup you cannot decrypt is not a backup. This section is the one that
produces an unrecoverable failure if it is got wrong, and it is got wrong often
because KMS keys are regional and people reason about them as if they were not.
Read [[aws-kms]] and [[kms-when-to-use-multi-region-keys]] alongside this.

### The mechanism

From the AWS encryption documentation:

> "A copy of a backup to another AWS Region is encrypted using the key of the
> destination vault."

So: the source recovery point is decrypted (in the source region, with the source
key) and the copy is **re-encrypted with the destination vault's key**. The two
recovery points are encrypted under two different keys. There is no shared key
requirement and — importantly — **no need for a KMS multi-Region key here.** The
standard answer in [[kms-when-to-use-multi-region-keys]] applies: use
independent, per-region customer-managed keys unless something forces otherwise,
and nothing here forces otherwise.

AWS also states that the copy is encrypted **even if the original backup was
not**: *"AWS Backup automatically encrypts those copies for most resource types,
even if the original backup is unencrypted."* Useful, but note the "most" — the
documented exception is that *"copies of unencrypted Amazon Aurora, Amazon
DocumentDB, and Amazon Neptune clusters are also unencrypted."*

### Independent encryption vs inherited encryption — and why it matters

The critical split is whether a resource type is **fully managed by AWS Backup**:

| Encrypted with the **vault's** key (independent encryption) | Encrypted with the **source resource's** key (inherited) |
|---|---|
| S3, EFS, DynamoDB *with* Advanced DynamoDB backup, Timestream, CloudFormation, VMware VMs, SAP HANA on EC2 | **RDS**, **Aurora**, **EBS**, **EC2/AMI**, DynamoDB *without* advanced backup, DocumentDB, Neptune, Redshift, Storage Gateway, FSx |

The right-hand column is where the danger lives. **An RDS snapshot in your vault
is encrypted with the key that encrypts the RDS instance, not with the vault's
key.** If someone rotates, disables, or schedules deletion of that key, the
recovery points in the vault become undecryptable — and the vault lock will
cheerfully keep the ciphertext for you for the full retention period.

Specifically:

- **Disabling a KMS key** makes every recovery point encrypted under it
  unrestorable until it is re-enabled. Recoverable mistake.
- **Scheduling a KMS key for deletion** starts a 7–30 day window. If it elapses,
  every recovery point under that key is **permanently unreadable**. This is not
  recoverable. Not by AWS. The ciphertext in the vault becomes noise.
- **Revoking a grant** or adding a `Deny` to the key policy causes copy jobs and
  restore jobs to fail. AWS documents both causes: *"Failed jobs can occur due to
  either one or more Deny statements applied to the KMS key or due to a grant
  revoked for the key."*

**Control to apply:** a `Deny` on `kms:ScheduleKeyDeletion` and `kms:DisableKey`
for any key used to encrypt backup-protected resources, scoped to everyone except
a break-glass role, applied as an SCP. This pairs with the vault lock — a vault
lock without KMS key protection protects the container but not the contents'
readability. Note there is no KMS equivalent of Vault Lock; AWS re:Post has an
open question on exactly this
([Prevent deletion of a CMK even by the root user](https://repost.aws/questions/QUcluGN53vSt-YyQuzmKK0Bw/prevent-deletion-of-a-cmk-even-by-the-root-user-kms-equivalent-to-backup-vault-lock)).
An SCP is the answer available.

### Required key policy permissions

AWS documents the minimum permissions the key policy must grant:

- `kms:CreateGrant`
- `kms:GenerateDataKey`
- `kms:Decrypt`

Plus, for a cross-Region copy specifically, AWS notes: *"the key associated with
the role initiating a cross-Region copy job must have
`"kms:ResourceAliases": "alias/aws/backup"` in the `DescribeKey` permission."*

The `CreateGrant` permission is the one people strip out during a security review
because it looks over-broad. AWS's own recommended form scopes it tightly:

```json
{
  "Sid": "KmsCreateGrantPermissions",
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::123456789012:root" },
  "Action": ["kms:CreateGrant"],
  "Resource": "*",
  "Condition": {
    "ForAnyValue:StringEquals": { "kms:EncryptionContextKeys": "aws:backup:backup-vault" },
    "Bool": { "kms:GrantIsForAWSResource": true },
    "StringLike": { "kms:ViaService": "backup.*.amazonaws.com" }
  }
}
```

Note the `kms:ViaService` value is `backup.*.amazonaws.com` — **wildcarded across
regions**. If your key policy hardcodes `backup.eu-west-1.amazonaws.com` (a
perfectly reasonable single-region hardening), **cross-Region copy jobs will
fail** once you introduce the second region. That is a concrete, findable
migration blocker: grep the estate's KMS key policies for hardcoded regional
`kms:ViaService` values before enabling any copy rule.

### Cross-account copy and AWS managed keys

A hard constraint, quoted:

> "Cross-account copy with AWS managed keys isn't supported for resources that
> aren't fully managed by AWS Backup. The key policy of an AWS managed key is
> immutable, which prevents copying the key across accounts."

Translation: **if your RDS instances are encrypted with `aws/rds`, you cannot
copy their backups into another account.** The isolated-account pattern is
unavailable to you until you re-encrypt those instances under a customer-managed
key. For RDS that means a snapshot-copy-with-new-key and a restore — i.e. a
migration, not a setting. AWS links a KB article for
[changing the encryption key on RDS](https://repost.aws/knowledge-center/update-encryption-key-rds)
and a storage blog post,
[Protecting encrypted Amazon RDS instances with cross-account and cross-Region backups](https://aws.amazon.com/blogs/storage/protecting-encrypted-amazon-rds-instances-with-cross-account-and-cross-region-backups/),
describing how to keep using AWS managed keys via an intermediate re-encryption
step.

**Audit action for this estate:** enumerate every RDS instance, Aurora cluster,
EBS volume and DynamoDB table and record whether it is encrypted with a CMK or an
AWS managed key. Anything on an AWS managed key is excluded from the
isolated-account control and needs a re-key plan. Do this **before** designing the
isolated account, because it determines whether the design is a week of work or a
quarter.

Also note the destination-vault key constraint for non-fully-managed resources:
*"the key associated to the destination vault must be a CMK or the managed key of
the service that owns the underlying resource. For example, if you are copying an
EC2 instance, a Backup managed key cannot be used. Instead, a CMK or Amazon EBS
KMS key (`aws/ebs`) must be used to avoid copy job failure."* So the DR vault
cannot simply use `aws/backup` if EC2/EBS are in scope.

### What happens if the source region's key is unavailable

The scenario that matters: `eu-west-1` is having a bad day, and you want to
restore in `eu-west-2` from the copied recovery point.

**Good news:** the copy in `eu-west-2` is encrypted under the *destination*
vault's `eu-west-2` key. It does not need the source key, the source vault, or
AWS Backup in `eu-west-1` to be restorable. This is the correct and reassuring
answer, and it is a direct consequence of re-encryption on copy.

**Bad news:** this is only true for the recovery points that were *already
copied*. Copies in flight at the moment the region degraded are lost, and — see
below — no new copies will be produced.

**Also verify:** the restore role in `eu-west-2` must have `kms:Decrypt` on the
`eu-west-2` key, and the `eu-west-2` key policy must permit it. This is easy to
get wrong because the standby region's IAM is rarely exercised. It is precisely
the class of thing a restore test would catch and nothing else would. See
[[dr-testing-and-gamedays]].

### If the primary region is gone, so is your copy scheduler

The backup plan, its rules, and its copy actions all live in the **source**
region. If `eu-west-1` is unavailable:

- No new backups are taken (the plan is in `eu-west-1`).
- No new cross-Region copies are produced (the copy job is initiated from
  `eu-west-1`).
- **Already-copied recovery points in `eu-west-2` are fully usable** — they are
  independent recovery points under a `eu-west-2` key in a `eu-west-2` vault, and
  restoring them requires nothing from `eu-west-1`.

So during a regional outage your DR-region backup set is **frozen at the last
successful copy** and starts ageing. That is fine for a short outage. For a long
one it means: after you fail over to `eu-west-2` and start serving traffic from
there, **the new primary has no backup plan.** Nothing is backing up the data you
are now writing.

**This is a real, easily-missed gap and it belongs in the failover runbook.** The
fix is to pre-provision a *disabled or unattached* backup plan in the standby
region — Terraform it, apply it, leave its selection empty — so that step N of
the failover runbook is "attach the standby backup selection" rather than "write
and apply new Terraform during an incident." Cross-ref [[failover-orchestration]]
and the reverse-copy discussion under [Failback](#failback).

---

## Organizations, cross-account, and the isolated-account pattern

Cross-Region protects you from a region. **Cross-account protects you from
yourself** — and from anyone holding your credentials. Given that the top
threats AWS Backup exists to answer are corruption, deletion and ransomware, and
that all three are usually executed with valid credentials inside the production
account, **cross-account is the more important axis of the two.**

### The blast-radius argument

If the backup vault lives in the same AWS account as the production resource,
then a principal with sufficient permissions in that account can delete the
resource *and* the backups. Vault Lock in compliance mode is a partial answer
(it survives an account-internal attacker), but recall the account-closure escape
hatch: an attacker with root can close the account and, 90 days later, AWS
deletes the vault contents regardless of the lock.

The pattern that actually holds:

```
  ┌─────────────────────────────────────┐     ┌──────────────────────────────┐
  │  PRODUCTION ACCOUNT                 │     │  BACKUP / VAULT ACCOUNT      │
  │  eu-west-1 (primary)                │     │  (no humans, no workloads)   │
  │                                     │     │                              │
  │  resources ──► backup plan          │     │  eu-west-2                   │
  │                  │                  │     │   ┌──────────────────────┐   │
  │                  ├─ vault (primary) │     │   │ isolated vault       │   │
  │                  │   governance lock │────┼──►│ COMPLIANCE LOCK      │   │
  │                  │                  │  copy  │ CMK, long retention  │   │
  │  eu-west-2 (standby, same account)  │     │   └──────────────────────┘   │
  │    └─ vault (dr) governance lock ◄──┘     │                              │
  └─────────────────────────────────────┘     └──────────────────────────────┘
        ▲                                                ▲
        │ prod credentials reach here                    │ prod credentials do NOT reach here
```

Three copies, three blast radii:

| Copy | Survives | Does not survive |
|---|---|---|
| Primary vault, prod account, `eu-west-1` | resource deletion, corruption | region loss, account compromise |
| DR vault, prod account, `eu-west-2` | resource deletion, corruption, **region loss** | account compromise |
| Isolated vault, backup account, `eu-west-2` | all of the above, **account compromise**, **ransomware** | AWS-wide catastrophe |

The third row is the one worth paying for. The first two are cheap by-products
of the plan you were writing anyway.

**Can you do both hops in one copy action?** For RDS, Aurora, DocumentDB and
Neptune, **yes** — AWS explicitly supports "cross-Region and cross-account
snapshot copying in a single action." For other resource types, the safe
assumption is chained copies (copy to DR region, then copy cross-account), which
means an extra copy job and extra latency. This is a per-resource-type detail
worth confirming against the estate's actual mix before designing the plan.

### Setting it up

1. **AWS Organizations must exist**, and every account involved must be in the
   same organization. This is a hard prerequisite for both cross-account copy and
   cross-account management.
2. **The management account enables** *Backup policies*, *Cross-account
   monitoring*, *Cross-account backup*, and (optionally) *Delegated
   administrator* under AWS Backup → Settings. These are opt-ins, not defaults.
3. **Register a delegated administrator.** Up to 5 accounts. This is the move
   that keeps day-to-day backup administration out of the management account,
   which is what any half-decent security review will insist on. Two separate
   steps are required and the docs are emphatic that people miss the second:
   - `aws organizations register-delegated-administrator --account-id <id>
     --service-principal "backup.amazonaws.com"`
   - **and** attach the `AWSBackupOrganizationAdminAccess` managed policy to the
     roles in that account, **and** create a resource-based delegation policy in
     the management account. From the docs: *"Registering an account as a
     delegated administrator alone does not enable backup policy management."*
4. **The destination vault needs a vault access policy** permitting the source
   account to copy into it. Cross-account copy is a push from the source, but the
   destination vault must consent.
5. **Every member account needs at least one vault and an IAM role** before an
   org-level backup policy will work against it. From the docs: *"Backup policies
   that reference accounts that lack the required information will not work as
   expected."* This is a bootstrapping order-of-operations trap — the policy
   applies cleanly and then silently does nothing in accounts that weren't
   prepared. In a cookiecutter monorepo, the per-account baseline stack must
   create the vault and role *before* the org policy is attached.

### Backup policies in Organizations vs local backup plans

An Organizations **backup policy** is a JSON document attached to the root, an
OU, or an account. AWS Backup materialises it as a real backup plan inside each
affected account.

**Inheritance and merging.** Policies attached at different levels **merge**, but
— and this is the sharp edge — *"Merging is done only for backup policies that
share the same backup plan name."* Same name → merged, with the more specific
level's values overriding. Different name → both plans coexist independently,
and **you are now taking and paying for two sets of backups.** The AWS example
is worth internalising: policy A at the org root (daily, 7-day retention), policy
B on the Finance OU (30-day retention); because they share a plan name, Finance
accounts get one plan with 30-day retention rather than two plans.

**Hard limitation, quoted:**

> "Backup policies support resource selection by resource types (for example,
> `arn:aws:ec2:*:*:volume/*`) or by tags, but not by individual resource ARNs.
> Use backup policies for broad application across multiple accounts in an
> organization or OU, not for targeting individual resources. To back up
> individual resources by ARN, use a local backup plan resource assignment
> instead."

So the architecture is necessarily two-tier: **org policies for the broad,
tag-driven baseline; local plans (Terraform `aws_backup_plan` +
`aws_backup_selection`) for anything that needs ARN-level targeting or unusual
retention.**

**Opt-in override.** For plans created by an org-level policy, *"the AWS Backup
opt-in settings for the Organizations management account will override the opt-in
settings in that member account, but only for that backup plan."* Local plans in
the member account follow the member account's own opt-in settings. So
`aws_backup_region_settings` means two different things depending on which kind
of plan you're reasoning about. This trips people up when Advanced DynamoDB
backup appears enabled in the console but a local plan still produces
non-copyable recovery points, or vice versa.

**Delegated administrators are not management accounts.** From the docs:
*"Delegated administrator accounts are member accounts with enhanced features and
cannot override settings like a management account can."* And, in opt-in regions
like `ca-west-1`, *"delegated administrator accounts can launch policies but do
not have access to the monitoring functions."* So the CA pair loses part of the
delegated-admin story on top of losing Audit Manager. Another entry for
[[region-pair-selection]].

### Terraform vs the org policy

The fork here is real and worth documenting for [[provider-aliases-vs-separate-stacks]]:

| | **Org backup policy** (`aws_organizations_policy`, type `BACKUP_POLICY`) | **Per-account Terraform plans** (`aws_backup_plan` + `aws_backup_selection`) |
|---|---|---|
| Coverage of new accounts | **Automatic.** A new account in the OU inherits the policy the moment it joins. | Manual — someone must add the account to the monorepo. |
| Targeting | Resource type and tag only. No ARNs. | Full ARN targeting, `not_resources`, conditions. |
| Where it lives in the repo | A single org-level stack in the management/delegated-admin account | Per-env, per-region, alongside the resources |
| Drift | Policy JSON is opaque to `terraform plan` diffs; a change rewrites the whole doc | Native Terraform diffs, readable plan output |
| Failure mode | Silent no-op in unprepared accounts | Loud `terraform apply` error |
| Blast radius of a mistake | Organisation-wide | One environment |

**Recommendation: use both, with a clear division.** Put the *floor* in an org
backup policy — "every EBS volume, RDS instance and EFS file system in every
account gets a daily backup with 35-day retention, copied cross-Region." That
floor is what closes the silent-gap problem below, because it applies to accounts
and resources nobody remembered to put in the monorepo. Then use Terraform local
plans for anything that needs tighter RPO (hourly), continuous backup, ARN-level
selection, or the cross-account copy into the isolated vault. Give the local
plans **different plan names** from the org policy so they coexist rather than
merge, and accept the (small) duplicate-backup cost as the price of a floor that
cannot be forgotten.

---

## Backup selections and the silent-gap problem

This is the failure mode that makes a backup programme worthless while looking
healthy on every dashboard.

### The mechanism

`aws_backup_selection` binds resources to a plan. It offers three ways to do it:

1. **`resources`** — explicit ARNs or ARN wildcards.
2. **`selection_tag`** — key/value tag match (`STRINGEQUALS`).
3. **`condition`** — richer matching: `string_equals`, `string_like`,
   `string_not_equals`, `string_not_like`, supporting wildcards like `prod*`.

Tag-based selection is the only one that scales, so essentially every real
deployment uses it. And tag-based selection has a property that is obvious once
stated and invisible in practice:

> **A resource created without the tag is simply never backed up. No job fails.
> No alarm fires. No dashboard turns red. There is no error, because from AWS
> Backup's point of view nothing happened at all.**

A failed backup job is loud — it appears in the jobs list, it can fire an SNS
notification via `aws_backup_vault_notifications`, it shows up in the Audit
Manager report. **A backup that was never scheduled is silent.** Every monitoring
instinct you have is tuned to detect the first and blind to the second.

### The realistic ways it happens

- A new microservice ships with a new RDS instance. The Terraform module that
  creates it doesn't inherit the shared `default_tags`, or someone overrode
  `tags` at the resource level and clobbered the backup tag.
- Someone fixes a typo: `Backup = "daily"` becomes `backup = "daily"`. Tag keys
  are case-sensitive. The resource silently leaves the plan.
- A resource is created outside Terraform — console, a migration script, an
  incident.
- A whole new AWS account is onboarded and the monorepo's per-account stack is
  applied a sprint later than the workload.
- The vault gets a `min_retention_days` lock and the plan's `delete_after` is
  below it, so **every job fails** — this one is loud, but only if anyone is
  listening to the notification topic.
- `max_retention_days` on a locked vault is below the plan's `delete_after` —
  same, loud but unheard.

### Detection

There are three mechanisms and you should run at least two.

**1. AWS Backup Audit Manager: "Backup resources are included in at least one
backup plan."** This is the purpose-built control. It *"evaluates if resources
are included in at least one backup plan,"* runs **automatically every 24 hours**,
and can be scoped by **resource type** — which is the crucial part, because
scoping by *tag* would reintroduce the very blind spot you are trying to close.
**Scope this control by resource type, never by tag.** A tag-scoped control
answers "are the tagged resources backed up," which is tautological. A
type-scoped control answers "are there any EBS volumes in this account that
nothing is backing up," which is the question.

**2. AWS Config.** A Config rule over resource types with a remediation or
notification is the alternative where Audit Manager is unavailable — which, per
the parity table, is exactly the `ca-west-1` situation. AWS Config is available in
Calgary. This is the compensating control referenced earlier.

**3. Tag policies in Organizations, plus an SCP.** Organizations tag policies can
enforce that a tag key exists and constrain its values, and an SCP can deny
`rds:CreateDBInstance` / `ec2:CreateVolume` without the required tag key via
`aws:RequestTag` conditions. This is prevention rather than detection and is the
strongest option, but it is also the one most likely to break somebody's
deployment on a Friday. Introduce it in report-only form first.

**Recommendation:** Audit Manager type-scoped control in the five regions that
support it, AWS Config equivalent in `ca-west-1`, and an org-level backup policy
providing a type-based (not tag-based) floor so that even an untagged resource
gets *something*. Prevention via SCP is the right end state; sequence it after
the detection is in place and quiet.

### The related control worth enabling

**"Last recovery point was created"** — evaluates whether a recovery point exists
within a time frame you set (1–744 hours, or 1–31 days). This catches the case
where the selection is correct but jobs are silently failing. Set it to roughly
2× your backup interval.

**One gotcha, quoted from the AWS control documentation:**

> "This control evaluates only recovery points created within the same account
> and Region. Recovery points copied from another account are not evaluated by
> this control."

So this control **cannot** be used in the isolated backup account to verify that
copies are arriving. Running it there will produce nothing useful. To monitor the
isolated vault you need either cross-account monitoring in the management/
delegated-admin account, or the "Cross-account backup copy is scheduled" control
in the *source* account, or EventBridge on copy-job state changes. Cross-ref
[[observability-multi-region]].

Note also that the **Resources in a logically air-gapped vault** control has a
recommended minimum of 7 days / 168 hours and warns that setting the window more
frequent than your copy cadence produces spurious `NON_COMPLIANT` results. And
per the parity table, that control is moot in `ca-west-1` anyway.

---

## Restore testing: the feature that answers the question nobody has asked

[[lessons-and-antipatterns]] raises the question of whether this company has ever
timed an end-to-end restore. AWS Backup has a feature whose entire purpose is to
answer that question on a schedule, and it is almost certainly not enabled.

### What it is

A **restore testing plan** (`aws_backup_restore_testing_plan` +
`aws_backup_restore_testing_selection`) does, on a cron schedule:

1. Selects protected resources matching the selection (by ARN — up to 30 — or by
   tag conditions).
2. For each, picks **at most one** eligible recovery point from the configured
   vaults, using a `latest`-or-`random` algorithm over a selection window of 1
   to 365 days.
3. Infers the restore metadata (subnet, instance class, subnet group, security
   group, encryption key) from the recovery point, with per-parameter overrides
   available.
4. Runs a real `StartRestoreJob` — *"Restore testing runs restore jobs in the
   same way as on-demand restores and uses the same recovery points."* You see
   the calls in CloudTrail. It is not a simulation.
5. **Measures and records the restore duration.**
6. Optionally holds the restored resource for 1–168 hours so you can validate it.
7. Deletes the restored resource.

Step 5 is the whole point. That number is the only honest input to an RTO
conversation and this estate does not currently have it for any resource type.

### What it validates

- **That the recovery point is restorable at all.** Not theoretically — actually.
  This catches KMS key problems, missing option groups, revoked grants, and
  corrupt recovery points.
- **That the restore IAM role works.** The troubleshooting section of the AWS doc
  is dominated by IAM and KMS failures, which tells you what breaks in practice.
- **That the restore fits in your target window.** Via the Audit Manager control
  *"Restore time for resources meet target"* — `NON_COMPLIANT` if
  `LatestRestoreExecutionTimeMinutes` exceeds your `maxRestoreTime` parameter.
- **That the destination region's plumbing exists.** Run restore tests in
  `eu-west-2` against copied recovery points and you are continuously proving the
  standby region's subnet groups, security groups, KMS key policies and IAM roles
  are correct — the exact set of things that rot silently in a warm standby.
  This is the highest-value configuration of the feature for this project.

### What it does NOT validate — and this is the important half

- **It does not validate that the data is correct.** Restore testing confirms a
  resource was created from a recovery point. It does not run a query, check a
  row count, verify a checksum, or open a file. Optional **restore testing
  validation** exists but *"validation can be run programmatically but not from
  the AWS Backup console"* — you write the validation. Without that, a restore
  test passing means "AWS produced a database," not "the database has your data
  in it."
- **It does not validate the application.** No connection strings repointed, no
  migrations run, no service brought up against the restored data. The gap
  between "RDS instance exists" and "the product works" is the gap that owns your
  real RTO, and restore testing does not touch it. That gap belongs to
  [[dr-testing-and-gamedays]] and a real gameday.
- **It does not validate every resource type.** Assignable types are: Aurora,
  DocumentDB, DynamoDB, EBS, EC2, EFS, FSx (Lustre/ONTAP/OpenZFS/Windows),
  Neptune, RDS, S3. Notably **EKS is absent** from that list, so the EKS backup
  story ([[eks-stateful-workloads]]) has no automated restore validation.
- **It does not test the resource you care about most unless you say so.**
  Recovery point selection uses `latest` or `random`; tag conditions apply to
  **protected resource** selection, not recovery point selection. Quoted:
  *"Tag conditions apply only to protected resource selection… As a result, the
  restored recovery point might not match the tag conditions used for protected
  resource selection."*
- **It does not measure the restore from cold storage.** Recovery points in cold
  tiers have their own retrieval latency, which is a separate and larger number.

### Operational traps

- **The cleanup depends on a tag.** Restored resources are tagged
  `awsbackup-restore-test`. *"If a user removes this tag, AWS Backup cannot
  delete the resource at the end of the testing period and the user will have to
  delete it manually."* An over-zealous tag-enforcement automation that strips or
  rewrites unknown tags will leave you paying for orphaned RDS instances. And for
  DynamoDB, S3, SAP HANA, VMs and Timestream — which don't support tag-on-restore
  — AWS deletes **by name**, so anything that renames a restored resource orphans
  it too.
- **S3 cleanup is slow and visible on the bill.** *"The deletion of a
  restore-tested Amazon S3 bucket takes longer than other resource types, because
  it can take a few days for lifecycle policies to delete all objects."*
- **Cron is evaluated 00:00–23:59.** *"If you create a restore testing plan for
  'every 12 hours' but provide a start time of later than 11:59, it will only run
  once per day."*
- **`SelectionWindowDays` interacts with `StartWindowHours`.** AWS's guidance:
  *"The selection window is calculated from the actual job execution time, not
  the plan's scheduled start time… set `SelectionWindowDays` to be greater than
  your backup frequency interval plus your `StartWindowHours` value to avoid
  edge-case recovery point exclusions."* Get this wrong and tests silently skip
  resources, which is the silent-gap problem wearing a different hat.
- **Use a dedicated account.** AWS's own recommendation: *"Recommended best
  practice is to designate an account to be used for restore tests."* Restores
  create real, billable, network-attached resources; you do not want them
  appearing in production VPCs.
- **Quotas:** 100 restore testing plans, 30 selections per plan, 30 protected
  resource ARNs per selection, 30 conditions per selection, 30 vault selectors
  per selection, max selection window 365 days, start window 1–168 hours.

### Recommendation

Enable restore testing in **both** regions of each pair, from day one of the
backup work, before any of the fancier controls:

- In the **primary**, weekly, `latest` algorithm, covering one instance of each
  resource type. Proves the backups are good.
- In the **standby**, weekly, against the **copied** recovery points. Proves the
  cross-Region copy chain, the destination KMS key policy, the destination IAM
  role, and the standby region's VPC plumbing all work. This is the continuous
  integration test for the DR posture.
- Turn on the **"Restore time for resources meet target"** control with a
  deliberately generous `maxRestoreTime` at first — say 240 minutes — and let it
  tell you the truth before you argue about what the target should be.
- Write at least one **programmatic validation** (row count, object count,
  checksum of a canary file) for the most important datastore. Without it the
  test proves less than people will believe it proves.

The cost is real but small — see [Cost](#cost) — and it is the cheapest piece of
genuine assurance available anywhere in this project.

---

## Compliance value: Audit Manager and evidence for auditors

The second reason AWS Backup exists in this design is that it is the only part of
the DR story that generates **evidence an auditor will accept**. Replication
produces no artefact. "We have Aurora Global Database" is an assertion. A daily
Audit Manager report showing 100% of in-scope resources covered by a plan, with
retention above the required minimum, copied cross-Region, encrypted, and stored
in a locked vault — that is a document.

### The controls, verified

Reproduced from AWS's
[controls and remediation](https://docs.aws.amazon.com/aws-backup/latest/devguide/controls-and-remediation.html)
page. Limit: **50 controls per account per Region**; the same control in two
frameworks counts twice.

| Control | What it evaluates | Evaluation cadence |
|---|---|---|
| Backup resources are included in at least one backup plan | Coverage — the silent-gap detector | Every 24h |
| Backup plan minimum frequency and minimum retention | Plan has a rule with frequency ≥ N and retention ≥ N days (defaults: 1 day / 35 days) | On config change |
| Vaults prevent manual deletion of recovery points | Vault access policy denies deletion except by up to 5 named IAM role ARNs | On config change |
| Recovery points are encrypted | Encryption at rest on recovery points | On config change |
| Minimum retention established for recovery point | Per-recovery-point retention ≥ N (default 35 days) | On config change |
| **Cross-Region backup copy is scheduled** | Resource is configured to copy backups to another Region (optionally: which Region) | Every 24h |
| **Cross-account backup copy is scheduled** | Resource is configured to copy to another account (up to 5 accounts; must be same Organization) | Every 24h |
| Resources are in a backup plan with an AWS Backup Vault Lock | Resource's backups land in a locked vault, optionally with min/max retention bounds | Every 24h |
| Last recovery point was created | A recovery point exists within N hours (1–744) or days (1–31). **Same account and Region only.** | Every 24h |
| **Restore time for resources meet target** | `LatestRestoreExecutionTimeMinutes` ≤ `maxRestoreTime` | Every 24h |
| Resources in a logically air-gapped vault | ≥1 recovery point copied to a LAG vault within a window (24–2184 hours / 1–91 days) | Every 24h |

For a multi-region DR programme, **"Cross-Region backup copy is scheduled" is the
control that turns the whole project into an auditable claim.** It is scoped by
resource type or tag and can be pinned to a specific destination Region — so you
can literally produce a report asserting "every production RDS instance in
`eu-west-1` is configured to copy its backups to `eu-west-2`," refreshed every 24
hours, with a history.

Important honesty note that AWS itself attaches to the restore-time control:

> "AWS Backup does not provide any service-level agreements (SLAs) for a restore
> time. Restore times can vary based upon system load and capacity, even for
> restores containing the same resources."

That sentence should be quoted verbatim in any internal document that states an
RTO for a restore-based recovery. It is AWS declining, in writing, to promise the
number your business continuity plan depends on.

Also relevant: only **active** resources are evaluated. A stopped EC2 instance is
not evaluated by "Last recovery point was created." A resource that is stopped
for a quarter can silently fall out of compliance reporting while still
containing data you care about.

### Frameworks and reports

`aws_backup_framework` bundles controls; `aws_backup_report_plan` emits reports
to an S3 bucket on a schedule (or on demand) in CSV/JSON. That S3 bucket is the
artefact you hand to an auditor. Put it somewhere with its own retention and its
own Object Lock — an audit trail that the audited party can rewrite is not an
audit trail. See [[aws-s3]].

Org-level aggregated reporting requires **both** cross-account management **and**
Audit Manager in the region. Per the parity table, `ca-west-1` has the former but
not the latter, so the Canadian standby cannot participate in aggregated org
reporting. State that in the compliance register rather than letting a report
quietly show fewer rows than expected.

### `ca-west-1` and the compliance gap, restated

It bears repeating in this section because it is where it hurts most: the region
most likely to be subject to a data-residency-driven audit is the one region in
the estate where AWS Backup Audit Manager does not run. See
[[regulatory-drivers]] and [[data-residency]].

---

## Data residency: a cross-Region backup copy is a data transfer

Flag this loudly, because it is the section that gets skipped and then gets
found.

**A cross-Region backup copy moves customer data across a border.** It is not a
metadata operation, it is not "just DR," and it is not exempt because it is
encrypted. For the three pairs:

| Pair | Copy crosses | Assessment |
|---|---|---|
| `eu-west-1` → `eu-west-2` | Ireland (EU) → United Kingdom | **Post-Brexit third-country transfer.** Relies on the UK adequacy decision. Adequacy decisions are renewable and have been challenged before. This is a live legal dependency, not a settled fact. |
| `us-east-1` → `us-west-2` | Virginia → Oregon | Domestic. No cross-border issue. State-level considerations only. |
| `ca-central-1` → `ca-west-1` | Montreal → Calgary | Domestic (Canada). **Clean — and a strong argument for keeping the CA pair as-is despite the service gaps.** |

The EU pair is the one to escalate. If the company has contractual commitments to
EU customers about data remaining in the EU — and the fact that there is a
separate Canadian deployment suggests residency commitments are taken seriously —
then **copying backups to London may breach them even though the primary data
never leaves Ireland.** Backups are frequently overlooked in residency reviews
precisely because they feel like infrastructure rather than data.

Options if the EU pair's residency is a problem:

- **Change the EU pair** to an EU-resident second region — Frankfurt, Paris,
  Stockholm, Spain, Milan, Zurich. All of these show full AWS Backup feature
  support in the region table, including Audit Manager and LAG vaults, so from
  *this* note's perspective any of them is a straight upgrade over London. The
  latency and the rest of the estate decide it. Feed to [[region-pair-selection]].
- **Keep London for compute failover but keep backups in-EU.** AWS Backup lets a
  plan have multiple `copy_action` blocks, so you can copy to Frankfurt for
  compliance and to London for operational convenience. Costs double the copy
  storage and transfer.
- **Accept the transfer under the UK adequacy decision**, documented, with legal
  sign-off, and with a note in the risk register about adequacy renewal.

**This note does not have enough information to recommend one.** The question is
"what have we told customers and regulators about where their data lives," and
that is an inside-the-company question. It is logged under
[Open questions](#open-questions) as a blocker, not a nicety.

One further wrinkle: **logically air-gapped vaults store backups in an AWS
Backup service-owned account.** From the AWS docs: *"a logically air-gapped vault
stores its backups in an AWS Backup service owned account (which results in
backups shown as shared outside your organization in modify attribute items in
AWS CloudTrail logs)."* A security team watching CloudTrail for
shared-outside-the-organization events will see these and escalate. Pre-brief
them, and be ready to explain to an auditor what "service-owned account" means
for custody of the data.

---

## Warm standby shape

What exists in `eu-west-2` while `eu-west-1` is healthy:

| Component | State while primary is healthy | Cost while idle |
|---|---|---|
| DR backup vault | Exists, receiving copies continuously | Vault itself: free. Storage: charged per GB-month. |
| DR vault KMS key | Exists, in use | ~$1/month per key + request charges — see [[aws-kms]] |
| DR vault lock (governance) | Applied | Free |
| Copied recovery points | Accumulating | **The main cost.** Per-GB-month at the resource type's rate. |
| DR backup plan | **Pre-provisioned but with no selection attached** | Free — a plan with no selection does nothing |
| DR restore testing plan | Active, running weekly | Per-test charge + restored resource charges for the test duration |
| DR IAM roles (backup + restore) | Exist | Free |
| Isolated-account vault | Exists, receiving copies | Storage charges |
| Audit Manager framework | Active in primary (and DR where supported) | Per-evaluation charges |

The important line is the **pre-provisioned DR backup plan with no selection**.
It costs nothing, it is a one-line Terraform resource, and it converts "write and
apply Terraform during an incident" into "apply one variable change" at the
moment you fail over. See [Failover procedure](#failover-procedure).

Nothing here is scaled to zero, because nothing here has capacity. AWS Backup's
idle cost is storage, and storage is the thing you cannot avoid — see
[Cost](#cost) and [[cost-model]].

---

## Terraform implementation

Real HCL against `hashicorp/aws` v5.x/v6.x, shaped for a cookiecutter-templated
multi-env monorepo.

### Provider aliases

Because the plan lives in the primary and the destination vault lives in the
standby, this module needs **two providers** — and, if you use the
isolated-account pattern, a third with an assume-role into the backup account.
See [[provider-aliases-vs-separate-stacks]] for the general argument; AWS Backup
is a clear case for **aliases in one stack**, because `copy_action` requires the
destination vault ARN and having Terraform compute it is far better than passing
it through a remote state lookup or, worse, a hardcoded string.

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.60, < 7.0"
    }
  }
}

provider "aws" {
  alias  = "primary"
  region = var.primary_region        # e.g. "eu-west-1"
  default_tags { tags = local.common_tags }
}

provider "aws" {
  alias  = "standby"
  region = var.standby_region        # e.g. "eu-west-2"
  default_tags { tags = local.common_tags }
}

# Only used when var.isolated_vault_enabled = true
provider "aws" {
  alias  = "backup_account"
  region = var.standby_region
  assume_role {
    role_arn = var.backup_account_role_arn
  }
  default_tags { tags = local.common_tags }
}
```

### Module signature

The variable surface to expose from a cookiecutter-templated module. The design
goal is that an environment's `terraform.tfvars` says *what* it wants protected
and *how tightly*, and never says *how*.

```hcl
# modules/aws-backup/variables.tf

variable "name_prefix" {
  description = "Resource name prefix, e.g. helios-prod. Must be stable — renaming a vault forces replacement."
  type        = string
}

variable "primary_region" { type = string }
variable "standby_region" { type = string }

variable "backup_tag_key" {
  description = "Tag key used for selection. Resources without this key are NOT backed up."
  type        = string
  default     = "BackupPolicy"
}

variable "tiers" {
  description = <<-EOT
    Backup tiers keyed by the value of var.backup_tag_key. A resource tagged
    BackupPolicy=critical gets the "critical" tier. Each tier is one rule in
    one plan.
  EOT
  type = map(object({
    schedule                 = string       # cron, e.g. "cron(0 * * * ? *)" for hourly
    start_window_minutes     = number       # keep small for RPO-critical tiers
    completion_window_minutes = number
    enable_continuous_backup = optional(bool, false)
    primary_delete_after     = number       # days
    primary_cold_after       = optional(number)  # days, null = never tier
    copy_to_standby          = optional(bool, true)
    standby_delete_after     = optional(number)
    copy_to_isolated_account = optional(bool, false)
    isolated_delete_after    = optional(number)
  }))

  # Guardrail: cold storage must be >= 90 days after warm, and cold recovery
  # points cannot be copied cross-Region.
  validation {
    condition = alltrue([
      for k, t in var.tiers :
      t.primary_cold_after == null || t.primary_delete_after >= t.primary_cold_after + 90
    ])
    error_message = "delete_after must be at least 90 days after cold_storage_after (AWS minimum cold-tier duration)."
  }

  validation {
    condition = alltrue([
      for k, t in var.tiers :
      !(coalesce(t.copy_to_standby, true) && t.primary_cold_after != null && t.primary_cold_after < 1)
    ])
    error_message = "Cross-Region copy is not supported for recovery points in cold storage. Do not tier before copying."
  }
}

variable "vault_lock" {
  description = <<-EOT
    Vault lock configuration. changeable_for_days = null -> GOVERNANCE mode
    (removable). Setting it to a number -> COMPLIANCE mode, which is
    IRREVERSIBLE after that many days. Read 02-services/aws-backup.md before
    setting it.
  EOT
  type = object({
    enabled             = bool
    min_retention_days  = number
    max_retention_days  = number          # NEVER leave this null on a compliance lock
    changeable_for_days = optional(number) # null = governance
  })
  default = {
    enabled            = true
    min_retention_days = 7
    max_retention_days = 400
  }

  validation {
    condition     = var.vault_lock.max_retention_days != null
    error_message = "max_retention_days must be set explicitly. An unbounded compliance-locked vault is a permanent bill."
  }
}

variable "isolated_vault_enabled" { type = bool, default = false }
variable "backup_account_role_arn" { type = string, default = null }

variable "restore_testing" {
  type = object({
    enabled              = bool
    schedule             = string          # cron
    selection_window_days = number
    algorithm            = string          # "LATEST_WITHIN_WINDOW" | "RANDOM_WITHIN_WINDOW"
    resource_types       = list(string)
    retain_hours         = optional(number, 1)
  })
  default = { enabled = true, schedule = "cron(0 3 ? * SUN *)", selection_window_days = 7, algorithm = "LATEST_WITHIN_WINDOW", resource_types = ["RDS", "EBS", "EFS"] }
}
```

### Vaults and keys

```hcl
# --- Primary region ---

resource "aws_kms_key" "primary_vault" {
  provider                = aws.primary
  description             = "${var.name_prefix} AWS Backup vault key (${var.primary_region})"
  enable_key_rotation     = true
  deletion_window_in_days = 30
  policy                  = data.aws_iam_policy_document.vault_key_primary.json
}

resource "aws_kms_alias" "primary_vault" {
  provider      = aws.primary
  name          = "alias/${var.name_prefix}-backup-vault"
  target_key_id = aws_kms_key.primary_vault.key_id
}

resource "aws_backup_vault" "primary" {
  provider    = aws.primary
  name        = "${var.name_prefix}-primary"
  kms_key_arn = aws_kms_key.primary_vault.arn
}

# --- Standby region: separate key, NOT a multi-region key ---

resource "aws_kms_key" "standby_vault" {
  provider                = aws.standby
  description             = "${var.name_prefix} AWS Backup vault key (${var.standby_region})"
  enable_key_rotation     = true
  deletion_window_in_days = 30
  policy                  = data.aws_iam_policy_document.vault_key_standby.json
}

resource "aws_backup_vault" "standby" {
  provider    = aws.standby
  name        = "${var.name_prefix}-dr"
  kms_key_arn = aws_kms_key.standby_vault.arn
}
```

The KMS key policy is the piece that must not hardcode a region. Note the
wildcard in `kms:ViaService`:

```hcl
data "aws_iam_policy_document" "vault_key_primary" {
  statement {
    sid     = "RootAccountAccess"
    effect  = "Allow"
    principals {
      type        = "AWS"
      identifiers = ["arn:${data.aws_partition.current.partition}:iam::${data.aws_caller_identity.current.account_id}:root"]
    }
    actions   = ["kms:*"]
    resources = ["*"]
  }

  statement {
    sid    = "AllowBackupServiceCreateGrant"
    effect = "Allow"
    principals {
      type        = "AWS"
      identifiers = ["arn:${data.aws_partition.current.partition}:iam::${data.aws_caller_identity.current.account_id}:root"]
    }
    actions   = ["kms:CreateGrant"]
    resources = ["*"]

    condition {
      test     = "ForAnyValue:StringEquals"
      variable = "kms:EncryptionContextKeys"
      values   = ["aws:backup:backup-vault"]
    }
    condition {
      test     = "Bool"
      variable = "kms:GrantIsForAWSResource"
      values   = ["true"]
    }
    condition {
      # MUST be wildcarded across regions. Hardcoding
      # "backup.eu-west-1.amazonaws.com" breaks cross-Region copy jobs.
      test     = "StringLike"
      variable = "kms:ViaService"
      values   = ["backup.*.amazonaws.com"]
    }
  }
}
```

### Vault lock — with the compliance-mode footgun made explicit

```hcl
resource "aws_backup_vault_lock_configuration" "primary" {
  count    = var.vault_lock.enabled ? 1 : 0
  provider = aws.primary

  backup_vault_name = aws_backup_vault.primary.name
  min_retention_days = var.vault_lock.min_retention_days
  max_retention_days = var.vault_lock.max_retention_days

  # PRESENCE OF THIS ARGUMENT = COMPLIANCE MODE = IRREVERSIBLE.
  # Absence = governance mode = removable with IAM permissions.
  changeable_for_days = var.vault_lock.changeable_for_days

  lifecycle {
    precondition {
      condition     = var.vault_lock.changeable_for_days == null || var.vault_lock.max_retention_days <= 2557
      error_message = "Refusing a compliance-mode lock with retention over 7 years without an explicit override."
    }
  }
}
```

Two things worth stealing from this snippet:

- The `precondition` is a cheap tripwire. It does not stop a determined engineer,
  but it makes "I didn't realise" an unavailable excuse.
- Consider also adding `prevent_destroy = true` on compliance-locked vaults, not
  because Terraform could destroy them (AWS will refuse) but so that
  `terraform plan` fails *early and legibly* rather than producing an apply error
  in the middle of a pipeline.

### The plan, with cross-Region and cross-account copy actions

```hcl
resource "aws_backup_plan" "main" {
  provider = aws.primary
  name     = "${var.name_prefix}-plan"

  dynamic "rule" {
    for_each = var.tiers
    content {
      rule_name         = "tier-${rule.key}"
      target_vault_name = aws_backup_vault.primary.name

      schedule                     = rule.value.schedule
      schedule_expression_timezone = "UTC"   # never leave this to chance across DST
      start_window                 = rule.value.start_window_minutes
      completion_window            = rule.value.completion_window_minutes

      # PITR in the source region only. Cross-Region copies of a continuous
      # backup degrade to periodic snapshots and lose PITR.
      enable_continuous_backup = rule.value.enable_continuous_backup

      lifecycle {
        cold_storage_after = rule.value.primary_cold_after
        delete_after       = rule.value.primary_delete_after
      }

      recovery_point_tags = merge(local.common_tags, {
        Tier          = rule.key
        SourceRegion  = var.primary_region
      })

      # --- cross-Region copy into the standby ---
      dynamic "copy_action" {
        for_each = coalesce(rule.value.copy_to_standby, true) ? [1] : []
        content {
          destination_vault_arn = aws_backup_vault.standby.arn
          lifecycle {
            delete_after = coalesce(rule.value.standby_delete_after, rule.value.primary_delete_after)
          }
        }
      }

      # --- cross-account (+ cross-Region) copy into the isolated vault ---
      dynamic "copy_action" {
        for_each = (var.isolated_vault_enabled && coalesce(rule.value.copy_to_isolated_account, false)) ? [1] : []
        content {
          destination_vault_arn = aws_backup_vault.isolated[0].arn
          lifecycle {
            delete_after = coalesce(rule.value.isolated_delete_after, 365)
          }
        }
      }
    }
  }

  # Required for application-consistent Windows backups; harmless otherwise.
  # Only include if Windows EC2 is actually in scope.
  dynamic "advanced_backup_setting" {
    for_each = var.windows_vss_enabled ? [1] : []
    content {
      resource_type  = "EC2"
      backup_options = { WindowsVSS = "enabled" }
    }
  }
}
```

### Selection — and the explicit acknowledgement of the silent gap

```hcl
resource "aws_backup_selection" "by_tier" {
  provider = aws.primary
  for_each = var.tiers

  name         = "${var.name_prefix}-${each.key}"
  plan_id      = aws_backup_plan.main.id
  iam_role_arn = aws_iam_role.backup.arn

  # Wildcard over everything, then narrow by tag. This is the standard shape.
  # NOTE: a resource without var.backup_tag_key is silently excluded. The
  # Audit Manager coverage control below is what detects that.
  resources = ["*"]

  condition {
    string_equals {
      key   = "aws:ResourceTag/${var.backup_tag_key}"
      value = each.key
    }
  }

  # Never back up restore-test artefacts or scratch resources.
  condition {
    string_not_like {
      key   = "aws:ResourceTag/Name"
      value = "awsbackup-restore-test*"
    }
  }
}
```

`selection_tag` (the older, simpler block) is equivalent for a plain
`STRINGEQUALS` match; `condition` is preferred because it supports
`string_like` / `string_not_like` and multiple predicates. Note the key format
differs: `condition` blocks take the full `aws:ResourceTag/<key>` form,
`selection_tag` takes the bare key.

### Advanced DynamoDB backup — the opt-in that unlocks copy

```hcl
# Without this, DynamoDB recovery points cannot be copied ANYWHERE —
# not cross-Region, not cross-account, not even to another vault in the
# same region and account.
resource "aws_backup_region_settings" "primary" {
  provider = aws.primary

  resource_type_opt_in_preference = {
    "Aurora"          = true
    "DocumentDB"      = false
    "DynamoDB"        = true
    "EBS"             = true
    "EC2"             = true
    "EFS"             = true
    "FSx"             = false
    "Neptune"         = false
    "RDS"             = true
    "S3"              = true
    "Storage Gateway" = false
    "VirtualMachine"  = false
  }

  resource_type_management_preference = {
    "DynamoDB" = true   # <-- Advanced DynamoDB backup
    "EFS"      = true
  }
}
```

Apply the same in the standby region. **Note that `resource_type_opt_in_preference`
is a per-region, per-account setting that Terraform will fight with anything else
that manages it** — including an Organizations backup policy, which overrides it
for org-managed plans. Own it in exactly one place.

### Restore testing

```hcl
resource "aws_backup_restore_testing_plan" "standby" {
  count    = var.restore_testing.enabled ? 1 : 0
  provider = aws.standby     # test the COPIES, in the region you'd fail over to

  name                         = replace("${var.name_prefix}_restore_test", "-", "_")
  schedule_expression          = var.restore_testing.schedule
  schedule_expression_timezone = "UTC"
  start_window_hours           = 2

  recovery_point_selection {
    algorithm             = var.restore_testing.algorithm
    include_vaults        = [aws_backup_vault.standby.arn]
    recovery_point_types  = ["SNAPSHOT"]
    selection_window_days = var.restore_testing.selection_window_days
  }
}

resource "aws_backup_restore_testing_selection" "standby" {
  for_each = var.restore_testing.enabled ? toset(var.restore_testing.resource_types) : toset([])
  provider = aws.standby

  name                      = replace("${var.name_prefix}_${each.key}", "-", "_")
  restore_testing_plan_name = aws_backup_restore_testing_plan.standby[0].name
  protected_resource_type   = each.key
  iam_role_arn              = aws_iam_role.restore_standby.arn

  # Hold the restored resource so programmatic validation can run against it.
  validation_window_hours = var.restore_testing.retain_hours

  protected_resource_arns = ["*"]

  protected_resource_conditions {
    string_equals {
      key   = "aws:ResourceTag/${var.backup_tag_key}"
      value = "critical"
    }
  }
}
```

Note the naming constraint the AWS docs call out: restore testing plan and
selection names *"must consist of only alphanumeric characters and underscores"* —
hence the `replace(..., "-", "_")`. A name that works everywhere else in the
monorepo will fail here, which is exactly the kind of thing that breaks a
cookiecutter template.

### Audit Manager framework

```hcl
resource "aws_backup_framework" "dr" {
  provider    = aws.primary
  name        = replace("${var.name_prefix}_dr_framework", "-", "_")
  description = "Coverage, retention and cross-Region copy controls for ${var.name_prefix}"

  control {
    name = "BACKUP_RESOURCES_PROTECTED_BY_BACKUP_PLAN"
    # Scope by RESOURCE TYPE, not tag. A tag-scoped coverage control is
    # circular: it can only ever find resources that already have the tag.
    scope {
      compliance_resource_types = ["RDS", "EBS", "EFS", "DynamoDB", "S3"]
    }
  }

  control {
    name = "BACKUP_RECOVERY_POINT_MINIMUM_RETENTION_CHECK"
    input_parameter {
      name  = "requiredRetentionDays"
      value = "35"
    }
  }

  control {
    name = "BACKUP_RESOURCES_PROTECTED_BY_CROSS_REGION"
    input_parameter {
      name  = "crossRegionList"
      value = var.standby_region
    }
  }

  control {
    name = "BACKUP_RECOVERY_POINT_ENCRYPTED"
  }

  control {
    name = "RESTORE_TIME_FOR_RESOURCES_MEET_TARGET"
    input_parameter {
      name  = "maxRestoreTime"
      value = "240"     # minutes. Start generous; tighten once you have data.
    }
  }
}

resource "aws_backup_report_plan" "dr" {
  provider    = aws.primary
  name        = replace("${var.name_prefix}_dr_report", "-", "_")
  description = "Daily DR backup compliance evidence"

  report_delivery_channel {
    s3_bucket_name = var.evidence_bucket
    formats        = ["CSV", "JSON"]
  }

  report_setting {
    report_template = "CONTROL_COMPLIANCE_REPORT"
    framework_arns  = [aws_backup_framework.dr.arn]
  }
}
```

Guard both of these behind a `var.audit_manager_supported` flag in the
cookiecutter, defaulting to `true` and set to `false` for `ca-west-1`. That is
the cleanest way to express a regional capability gap in a templated monorepo —
better than a conditional on region name buried in a `locals` block, because the
variable is self-documenting in the tfvars file.

### What NOT to put in this module

`aws_organizations_policy` with `type = "BACKUP_POLICY"` belongs in the
organisation-level stack owned by the management or delegated-admin account, not
in a per-environment module. Mixing organisation-scoped and account-scoped
resources in one state file is how you end up unable to apply an environment
change because the org stack is locked. See [[state-management]].

---

## Migration path from single-region

The company is live in `eu-west-1` today. AWS Backup is unusual among the
services in this vault in that **the migration is genuinely low-risk** — nothing
here touches a running workload, and the destructive operations are all opt-in.

### Step 0 — Audit before you build

Three inventories, none of which take long, all of which change the design:

1. **Encryption inventory.** For every RDS instance, Aurora cluster, EBS volume
   and DynamoDB table: CMK or AWS managed key? Anything on an AWS managed key is
   **excluded from cross-account copy** and needs a re-key plan. Do this first;
   it is the only finding that can turn a two-week project into a quarter.
2. **Tag inventory.** What proportion of in-scope resources already carry a
   usable backup tag? If the answer is "we don't have a convention," the tag
   convention is the first deliverable, not the backup plan.
3. **KMS key policy grep.** Any key policy with a hardcoded regional
   `kms:ViaService` (e.g. `backup.eu-west-1.amazonaws.com`) will break
   cross-Region copy. Fix to `backup.*.amazonaws.com` before enabling copy rules.

### Step 1 — Vaults and keys in both regions, no plans

Apply `aws_kms_key`, `aws_backup_vault` in `eu-west-1` and `eu-west-2`. Zero
impact on anything. Zero cost beyond two KMS keys.

**Do not apply any vault lock yet.**

### Step 2 — A plan with no selection

Apply `aws_backup_plan` with the rules and copy actions. A plan with no
`aws_backup_selection` attached takes no backups and costs nothing. This lets you
review the rendered plan in the console and argue about retention before any data
moves.

### Step 3 — Selection against one non-production resource

Attach a selection scoped to a single ARN in a non-production environment. Watch:

- the backup job complete, and **time it**;
- the copy job complete, and **time it**;
- the recovery point appear in the `eu-west-2` vault with the `eu-west-2` key.

These two timings are the inputs to the RPO arithmetic earlier in this note. You
now have real numbers instead of the placeholder ones. **Do not size the schedule
before this step.**

### Step 4 — Restore test the copy immediately

Before widening the selection, prove a restore works from `eu-west-2`. This
catches the KMS, IAM, subnet-group and option-group problems while the blast
radius is one test resource. This is the step everyone skips and it is the step
that finds the bugs.

### Step 5 — Widen by tag, environment by environment

Non-prod first, then prod. Watch the bill after each widening — the **first**
copy of every resource is a full transfer and the first month's data-transfer
line will not resemble steady state.

### Step 6 — Audit Manager and detection

Turn on the coverage control **scoped by resource type**. Expect it to be red.
The red is the point: it is the first honest measurement of how much of the
estate is unprotected. Work it down.

### Step 7 — Governance locks

Apply `aws_backup_vault_lock_configuration` **without** `changeable_for_days` to
both vaults. Reversible. Cheap. Satisfies most of the "immutable backups"
conversation.

### Step 8 — Organizations, delegated admin, isolated account

The biggest step and the one with real prerequisites (Organizations enabled,
cross-account features opted in, a new account, CMKs everywhere). Do it last and
separately.

### Step 9 — Compliance lock on the isolated vault only

After — and only after — someone has computed and signed off the maximum cost at
`max_retention_days`. Use a `changeable_for_days` longer than the 3-day minimum.

### ForceNew / replacement hazards

| Change | Effect |
|---|---|
| `aws_backup_vault.name` | **ForceNew.** Destroys and recreates the vault. Terraform will refuse to destroy a vault containing recovery points, so in practice `terraform apply` **errors out** mid-run and you are left half-applied. Pick vault names carefully and never let a cookiecutter naming-convention change touch them. |
| `aws_backup_vault.kms_key_arn` | **ForceNew.** Same hazard. Choosing the vault key is a one-way door in practice. |
| `aws_backup_plan.name` | ForceNew on the plan. Selections are children of the plan and are recreated with it. A brief window with no active selection; low risk but real. |
| `aws_backup_vault_lock_configuration` with `changeable_for_days` set | **Not a Terraform replacement — an AWS-level one-way door.** After the grace period, Terraform cannot remove it, and a `terraform destroy` on that vault will fail forever. |
| `aws_backup_restore_testing_plan.name` | ForceNew, and the name has a restricted character set. |
| Adding a `copy_action` to an existing rule | In-place update. Safe. Applies to future recovery points only — **existing recovery points are not retroactively copied.** |
| Changing `lifecycle.delete_after` | In-place update. Applies to future recovery points only. Existing recovery points keep their original expiry. |

That last pair is a recurring confusion: **backup plan changes are prospective,
never retrospective.** Changing retention from 7 to 35 days does not extend the
recovery points you already have. If you need the old ones retained, copy them
on demand.

Nothing here requires downtime on any production resource, and no existing
resource is modified — `aws_backup_selection` is a pointer to resources, not a
change to them.

---

## Failover procedure

AWS Backup's role in the 3am runbook is smaller than people expect and it is
mostly about what happens *after* the failover, not during it.

**During the failover (region loss):**

1. **AWS Backup is not consulted.** You promote the replicated datastores per
   [[failover-orchestration]]. Do not restore from backup. Restoring from backup
   during a region-loss failover means discarding the replicated data in favour
   of data that is up to 2–3 hours older, and taking hours to do it. It is
   strictly worse on both axes.
2. **Note the time of the last successful cross-Region copy.** It is your
   worst-case fallback position and it stops advancing the moment the primary
   goes dark. Someone should write it on the incident timeline.

**Immediately after traffic is serving from `eu-west-2`:**

3. **Attach the standby backup selection.** The new primary currently has no
   backup plan. Everything written since failover is unprotected. This is the
   step the pre-provisioned, selection-less standby plan exists for: flip one
   variable, apply, done. Put it in the runbook with an explicit owner, because
   it will otherwise be forgotten for weeks.
4. **Decide where the standby's copies go.** There is nowhere to copy to — the
   paired region is down. Options: copy in-region to a second vault (better than
   nothing, no geographic separation), copy to the isolated account (which is the
   argument for putting the isolated account's vault in a *third* region), or
   accept single-region backups for the duration.
5. **Confirm the restore role in `eu-west-2` can decrypt.** If you have been
   running restore tests in the standby, you already know. If you have not, find
   out now rather than during the next incident.

**During a corruption or ransomware incident — the scenario AWS Backup owns:**

1. **Stop replication first.** This is the single most important and most
   counter-intuitive step. Before anything else, break or pause the replication
   that is currently copying corrupted data to the standby. Every second you
   deliberate, the standby becomes less useful. See [[split-brain-and-fencing]].
2. **Identify a clean recovery point.** For continuous-backup resources, PITR to
   just before the bad event. This is the fast path and it only exists in the
   **source** region.
3. **Restore to a new resource, not over the old one.** Never restore in place.
   The corrupted resource is evidence and it may contain data the backup does
   not.
4. **Accept the RTO.** This is hours. Communicate it early and honestly.
5. **Repoint the application.** New endpoint, new identifier. This is the part
   that is not automated and is the largest term in the real RTO.

---

## Failback

**After a region-loss failover and failback to `eu-west-1`:**

The backup plan in `eu-west-1` is still there and still configured, so it
resumes. The problems are subtler:

- **The recovery points in `eu-west-1` are now stale** — they stopped at the
  moment of the outage and describe a version of the data that is hours or days
  behind. They are not wrong, but they are not a restore target either. Do not
  let an automated "restore to the most recent recovery point" runbook step reach
  for them.
- **Copy direction must be reversed and then re-reversed.** While `eu-west-2` is
  primary, its plan should copy to `eu-west-1`. After failback, back the other
  way. This is two variable flips, and **both should be in the runbook and both
  should be Terraform, not console clicks** — a console-configured copy action
  will be silently reverted by the next `terraform apply`, which is a genuinely
  nasty way to lose your DR copies. Cross-ref [[state-management]].
- **The retention asymmetry is now backwards.** If `eu-west-1` keeps 7 days and
  `eu-west-2` keeps 35, then after a long period running from `eu-west-2` your
  short-retention vault is the one holding the authoritative history. Consider
  symmetric retention on both sides purely to remove a class of failback bug —
  the extra storage cost is usually smaller than the cost of getting this wrong.
- **Incremental copy chains reset.** After a long gap, expect the first copy
  after failback to be full for at least some resource types. Budget the transfer.

**After a corruption incident:** the restored resource is a *new* resource with a
new ARN. **The backup selection will not match it unless it carries the backup
tag.** A restored RDS instance that nobody re-tagged is a resource with no
backups, created during an incident, at exactly the moment everyone's attention
is elsewhere. This is the silent-gap problem with the worst possible timing.
Add "re-tag the restored resource" as an explicit runbook step, and rely on the
24-hour Audit Manager coverage control as the backstop.

---

## Gotchas

The list that makes this note worth reading.

1. **Plain DynamoDB recovery points cannot be copied anywhere** — not
   cross-Region, not cross-account, not even to another vault in the same region
   and account. Advanced DynamoDB backup (`resource_type_management_preference`)
   is required. Easy to miss because the matrix has two DynamoDB rows.
2. **Vault Lock compliance mode is created by the *presence of an argument*, not
   by a mode flag.** `changeable_for_days` set = compliance = irreversible after
   ≥3 days. A stray non-null variable default is enough.
3. **`max_retention_days` unset on a compliance-locked vault is an unbounded
   permanent bill.** A recovery point with "Always" retention can never be
   deleted, and the vault can never be deleted because it is not empty. The only
   documented escape is closing the AWS account and waiting 90 days.
4. **A hardcoded regional `kms:ViaService` in a key policy breaks cross-Region
   copy.** AWS's own example uses `backup.*.amazonaws.com`. Single-region
   hardening becomes a multi-region outage.
5. **RDS, Aurora, EBS and EC2 recovery points are encrypted with the *source
   resource's* key, not the vault's key.** Deleting that KMS key makes the
   backups permanently unreadable, and the vault lock will faithfully preserve
   the undecryptable ciphertext for the full retention period.
6. **Cross-account copy is impossible for resources encrypted with AWS managed
   keys** (`aws/rds`, `aws/ebs`). The key policy of an AWS managed key is
   immutable. This blocks the isolated-account pattern until you re-key.
7. **Cold recovery points cannot be copied cross-Region.** Copy first, tier
   later. A lifecycle that tiers to cold on day 3 and a copy rule that runs on
   day 4 produce nothing and no obvious error.
8. **Cold storage has a 90-day minimum.** `delete_after` must be at least 90 days
   greater than `cold_storage_after`, or the plan is invalid.
9. **RDS cross-Region copy does not carry the option group.** From the docs: AWS
   Backup *"copies the default option group, even if you have configured a custom
   option group,"* and if your custom group uses persistent options the copy job
   **fails** unless you have manually pre-created a matching option group in the
   destination region. Silent until the day it isn't. Pre-create the option group
   in the standby as part of the [[aws-rds-postgres]] work.
10. **Cross-Region copies of continuous backups lose PITR.** The copy becomes a
    periodic snapshot. You have PITR in the primary and discrete snapshots in the
    standby, and people will assume otherwise.
11. **A tag-based selection silently excludes untagged resources.** No job, no
    error, no alarm. Detect with the type-scoped coverage control, not a
    tag-scoped one.
12. **A vault lock's `min_retention_days` makes non-conforming backup jobs
    *fail*, not clamp.** A lock with `min_retention_days = 90` plus a plan with
    `delete_after = 30` stops all backups into that vault. Similarly a continuous
    backup (35-day cap) into a vault with `min_retention_days = 90` fails.
13. **The "Last recovery point was created" control ignores cross-account
    copies.** Quoted: *"Recovery points copied from another account are not
    evaluated by this control."* It cannot monitor the isolated vault.
14. **Only active resources are evaluated by Audit Manager controls.** A stopped
    EC2 instance falls out of compliance reporting while still holding data.
15. **Restore-test cleanup depends on the `awsbackup-restore-test` tag** (or, for
    DynamoDB/S3/SAP HANA/VMs/Timestream, on the resource *name*). Tag-enforcement
    automation that strips unknown tags will orphan billable resources.
16. **Restore testing cron is evaluated 00:00–23:59.** "Every 12 hours" with a
    start time after 11:59 runs once a day.
17. **Concurrency quotas are real and shared.** 100 concurrent cross-Region copy
    jobs per account per destination region (then only 5 more per vault); **1
    concurrent backup or copy job per resource**. That last one caps your backup
    frequency: if a resource's backup job takes 70 minutes, an hourly schedule
    cannot keep up and jobs will be skipped.
18. **Max 5 `copy_action` blocks per rule, 10 rules per plan, 10 tag-based
    selections per plan, 100 resource assignments per plan.** These bind a
    tier-driven `for_each` design faster than you'd think.
19. **Org backup policies merge only when the plan names match.** Different names
    = two coexisting plans = two sets of backups = double storage cost, silently.
20. **Org backup policies cannot target individual ARNs** — resource type or tag
    only. And they silently no-op in member accounts that lack a vault or an IAM
    role.
21. **Registering a delegated administrator does not grant policy management.**
    You must also attach `AWSBackupOrganizationAdminAccess` and create a
    resource-based delegation policy in the management account.
22. **`ca-west-1` is an opt-in region**, which degrades cross-account management:
    delegated admins can launch policies but lose monitoring, and there can be a
    24-hour lag on cross-account job visibility.
23. **Vault Lock does not prevent a resource being made unrestorable.**
    `ec2:DisableImage` blocks restores of EC2 recovery points even in a locked
    vault. Deny it via SCP.
24. **LAG vault backups live in an AWS service-owned account** and appear in
    CloudTrail as shared outside your organisation. Pre-brief the security team.
25. **The copy scheduler lives in the source region.** Lose the primary and you
    lose the ability to produce new copies — and the new primary has no backup
    plan until you attach one.
26. **Backup plan changes are prospective only.** Extending retention does not
    extend existing recovery points; adding a `copy_action` does not copy
    existing ones.
27. **Renaming a vault is ForceNew, and the destroy will fail** because the vault
    contains recovery points — leaving a half-applied state. Vault names are
    effectively permanent.

---

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| **Is AWS Backup part of the RTO story?** | Yes — treat restore-from-backup as a failover path | No — AWS Backup is RPO floor + corruption recovery only; replication owns RTO | **B, unambiguously.** No resource type restores in 15 minutes. Document a separate, longer RTO for the corruption incident class. |
| **Vault lock mode, operational vaults** | Compliance (irreversible) | Governance (removable with IAM) | **B.** Compliance mode on a vault you operate daily is a trap. Governance + a vault access policy gets you 90% of the control with 0% of the commitment. |
| **Vault lock mode, isolated-account vault** | Governance | Compliance, with explicit `max_retention_days` and a >3-day grace | **B.** This vault's purpose *is* immutability. But compute the maximum cost first, and never leave `max_retention_days` unset. |
| **Isolated backup account** | Skip it — cross-Region copy in the same account is enough | Separate account, cross-account copy, compliance lock | **B**, but sequence it last. It is the only control that survives production account compromise. Blocked on the CMK audit — AWS-managed-key resources cannot participate. |
| **Backup plan authoring** | Org backup policies only | Per-account Terraform only | **Both.** Org policy provides a type-based floor that cannot be forgotten when a new account appears; Terraform provides ARN-level precision and readable diffs. Use different plan names so they coexist. |
| **Selection method** | Tag-based | ARN-based | **Tag-based** (it is the only thing that scales) **plus** a type-scoped Audit Manager coverage control to detect the gap it creates. Tag-based without the coverage control is negligence. |
| **Backup frequency for critical data** | Daily (AWS default-ish) | Hourly with a tight start window | **Hourly for the tier that has an RPO commitment, daily for everything else.** Hourly barely meets RPO 2h and only with measured copy durations. Do not run hourly across the whole estate — the storage and job-concurrency cost is real. |
| **Continuous backup (PITR)** | Off — snapshots are simpler | On for S3, RDS, Aurora | **On.** It is the single best answer to logical corruption caught quickly, which is the most common real incident. Accept that it does not survive to the standby region. |
| **Restore testing** | Defer until the backups are bedded in | Enable from day one, in both regions | **B, from day one.** It is cheap, it is the only source of real restore-time data, and testing against the *copied* recovery points in the standby continuously validates the DR region's plumbing. |
| **EU pair backup destination** | `eu-west-2` (London) — matches the compute pair | An EU-resident region (Frankfurt/Paris/Stockholm) for backups specifically | **Blocked on legal.** If EU residency is contractually committed, copying backups to the UK is a third-country transfer. Technically Frankfurt is a straight upgrade (full feature parity). Escalate before building. |
| **`ca-west-1` audit gap** | Change the CA pair | Accept the gap, compensate with AWS Config | **B.** There is no other Canadian region, and leaving Canada breaks residency. Compensate with Config, document the gap in the compliance register. |
| **KMS keys for vaults** | Multi-Region keys | Independent per-region CMKs | **B.** Copies are re-encrypted with the destination vault's key; MRKs buy nothing here and add operational weight. Consistent with [[kms-when-to-use-multi-region-keys]]. |
| **Cold storage tiering** | Tier aggressively to save money | Keep everything warm | **Warm for the DR copy, tier the long-retention isolated copy.** Cold recovery points cannot be copied cross-Region, and cold retrieval adds to an already-failing RTO. Tier only where the data is pure compliance archive. |

---

## Cost

**Verification status:** the figures below marked ✅ were read from AWS's own
[AWS Backup pricing page](https://aws.amazon.com/backup/pricing/). Figures marked
⚠️ come from third-party sources and are **not verified against an AWS page** —
the AWS pricing tables are JavaScript-rendered and did not yield complete
per-resource rates to automated fetching. **Treat ⚠️ figures as indicative only
and re-check in the AWS Pricing Calculator before putting a number in a budget.**

### Storage

| Item | Rate | Verified |
|---|---|---|
| EBS backup warm storage, `us-east-1` | $0.05 / GB-month | ✅ |
| EFS backup warm storage, `us-east-1` | $0.05 / GB-month | ✅ |
| RDS backup warm storage, `us-east-1` | ~$0.095 / GB-month | ⚠️ |
| DynamoDB backup warm storage, `us-east-1` | ~$0.10 / GB-month | ⚠️ |
| EFS cold storage, `us-east-1` | ~$0.01 / GB-month | ⚠️ |
| `eu-west-1` / `eu-west-2` / `ca-central-1` / `ca-west-1` rates | **Not verified.** Non-US regions are generally higher. | — |

Cold storage is materially cheaper but carries a **90-day minimum retention**
and cannot be cross-Region copied. Also note that for EFS, S3, VMware, SAP HANA
and Timestream, storage is billed **per GB-day** — a backup that exists for part
of a day is billed for the whole day, which makes high-frequency short-retention
plans on those resource types more expensive than the monthly rate suggests.
(⚠️ third-party reported.)

### Cross-Region data transfer

| Item | Rate | Verified |
|---|---|---|
| EBS backup copy, `us-east-1` → `eu-west-1` | $0.02 / GB | ✅ |
| EFS backup copy, `us-east-1` → `eu-west-1` | $0.04 / GB | ✅ |
| Rates for the EU/US/CA pairs used in this project | **Not verified** | — |

Remember: **the first copy of every resource is full.** If the estate holds 20 TB
of EBS, the initial seed into the standby region is a 20 TB transfer. At $0.02/GB
that is roughly $400 one-off — modest — but the *steady-state* number depends
entirely on daily change rate, which nobody has measured. That measurement is a
prerequisite for a credible [[cost-model]] entry.

### Restore and restore testing

| Item | Rate | Verified |
|---|---|---|
| Restore testing, per recovery point evaluated | $1.50 | ✅ |
| Standard restore | charged per GB restored; AWS's worked example shows $0.02/GB for EFS | ✅ (as an example rate) |
| EBS restore | free in AWS's worked example | ✅ (as an example) |
| Resources created during a restore test | charged at normal rates for their lifetime | ✅ |

Restore testing at $1.50 per recovery point is cheap enough that cost is not a
reason to skip it. Weekly tests across 10 resources in 2 regions is ~$130/month
plus the short-lived restored resources — and it is the only genuine assurance in
the entire programme. **The restored resources are the larger cost**, especially
S3 (buckets take days to lifecycle-delete) and anything with a large instance
class inferred from the recovery point. Override the instance class down in the
restore testing selection where the resource type allows it.

### Free

Vault Lock is explicitly free: *"AWS Backup Vault Lock is available at no
additional charge. Standard AWS Backup storage charges apply to backups stored in
a locked vault."* Vaults themselves are free. Backup plans are free.

### The levers

1. **Retention, not frequency, dominates storage cost** for incremental resource
   types. Going from 7 to 35 days is roughly 5× the retained data; going from
   daily to hourly adds far less than 24× because the increments are small.
   **Except for DynamoDB with advanced backup, which the matrix shows as
   non-incremental — every backup is full. Hourly DynamoDB backups are
   catastrophically expensive. Do not do it.**
2. **Asymmetric retention.** 7 days warm in the primary, 35 in the DR region, 365
   in the isolated vault, tiered to cold. Set per-`copy_action` `lifecycle`.
3. **Don't back up what you can regenerate.** Derived data, caches, search
   indexes ([[aws-opensearch]]) and anything rebuildable from a source of truth
   should not be in a backup plan.
4. **Tier the archive copy, not the DR copy.** Cold cannot be copied
   cross-Region and slows an already-slow restore.
5. **Compliance-locked vaults have no cost lever at all.** Once locked, retention
   cannot be shortened and recovery points cannot be deleted. This is why
   `max_retention_days` is the most important number in the configuration.

---

## Open questions

Things that need an answer from inside the company before this design is final.

1. **Has anyone ever timed an end-to-end restore of production data?** If not,
   that is the single highest-value action arising from this note, and restore
   testing makes it a one-week task rather than a project. Feeds
   [[lessons-and-antipatterns]] and [[dr-testing-and-gamedays]].
2. **What have we told customers and regulators about where data lives?**
   Specifically: does copying backups from Ireland to London breach any EU
   residency commitment? This blocks the EU pair decision and cannot be resolved
   from documentation. Feeds [[data-residency]] and [[regulatory-drivers]].
3. **Are production RDS/Aurora/EBS resources encrypted with CMKs or AWS managed
   keys?** Determines whether the isolated-account pattern is available at all.
4. **Does an AWS Organization exist, with the accounts arranged usefully?** Most
   of the cross-account section is blocked on this.
5. **Is there a tagging convention with enforcement?** Tag-based selection is
   only as good as the tagging discipline behind it.
6. **What is the actual daily change rate per datastore?** Required for any
   credible cross-Region transfer cost estimate.
7. **Who is accountable for the corruption-incident RTO?** The 15-minute RTO is
   owned; the multi-hour restore RTO currently is not, because nobody has stated
   it exists.
8. **Is there an existing backup mechanism today** (RDS automated backups, EBS
   Data Lifecycle Manager, ad-hoc scripts) that AWS Backup would duplicate or
   should replace? Running both is a common and expensive accident.
9. **Does the estate use RDS custom option groups?** If so, they must be
   pre-created in the standby region or every cross-Region copy job fails.
10. **What is the acceptable maximum spend on a compliance-locked vault?** This
    number must exist, in writing, before anyone sets `changeable_for_days`.

---

## Sources

All URLs verified as reachable and read during research on 2026-09-21.

**AWS documentation (primary sources — the support matrix, Vault Lock semantics
and quotas in this note are taken directly from these):**

- [AWS Backup feature availability](https://docs.aws.amazon.com/aws-backup/latest/devguide/backup-feature-availability.html)
  — the authoritative support matrix. Source of the cross-Region/cross-account
  copy table, the DynamoDB gap, and every `ca-west-1` parity finding.
- [Creating backup copies across AWS Regions](https://docs.aws.amazon.com/aws-backup/latest/devguide/cross-region-backup.html)
  — copy mechanics, first-copy-is-full, the EBS full-copy exception, the
  no-cold-storage-copy rule, and the RDS option group failure.
- [AWS Backup Vault Lock](https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html)
  — governance vs compliance, the 3-day/36,500-day `ChangeableForDays` bounds,
  min/max retention semantics, the "cannot be changed or deleted by any user or
  by AWS" statement, the account-closure escape hatch, and the
  `ec2:DisableImage` gotcha.
- [Encryption for backups in AWS Backup](https://docs.aws.amazon.com/aws-backup/latest/devguide/encryption.html)
  — the independent-vs-inherited encryption table, re-encryption on copy, the
  minimum KMS key policy permissions, and the AWS-managed-key cross-account
  restriction.
- [Restore testing](https://docs.aws.amazon.com/aws-backup/latest/devguide/restore-testing.html)
  — what restore testing does and does not validate, quotas, the
  `awsbackup-restore-test` tag cleanup dependency, and the troubleshooting list.
- [AWS Backup Audit Manager controls and remediation](https://docs.aws.amazon.com/aws-backup/latest/devguide/controls-and-remediation.html)
  — every control in the compliance table, including the "no SLA for restore
  time" statement and the same-account-and-Region limitation on "Last recovery
  point was created."
- [Managing AWS Backup resources across multiple AWS accounts](https://docs.aws.amazon.com/aws-backup/latest/devguide/manage-cross-account.html)
  — Organizations backup policies, policy merging by plan name, delegated
  administrator setup and its two-step trap, opt-in override rules.
- [Logically air-gapped vault](https://docs.aws.amazon.com/aws-backup/latest/devguide/logicallyairgappedvault.html)
  — always-compliance-locked, 7-day minimum retention, RAM sharing,
  service-owned-account storage, and the Calgary unavailability.
- [AWS Backup quotas](https://docs.aws.amazon.com/aws-backup/latest/devguide/aws-backup-limits.html)
  — the concurrency limits that bound backup frequency, plus the per-plan rule,
  copy-action and selection limits that bound the Terraform design.
- [AWS Backup pricing](https://aws.amazon.com/backup/pricing/) — the ✅ figures in
  the cost section. Note the page is JS-rendered; the full per-region rate tables
  did not yield to automated fetching and are marked unverified above.
- [Protecting encrypted Amazon RDS instances with cross-account and cross-Region backups](https://aws.amazon.com/blogs/storage/protecting-encrypted-amazon-rds-instances-with-cross-account-and-cross-region-backups/)
  — AWS Storage blog, linked from the encryption docs as the workaround for
  AWS-managed-key resources.
- [Managing backups at scale in your AWS Organizations using AWS Backup](https://aws.amazon.com/blogs/storage/managing-backups-at-scale-in-your-aws-organizations-using-aws-backup/)
  — AWS Storage blog, linked from the cross-account docs for org-scale patterns.
- [Update the encryption key on an RDS instance](https://repost.aws/knowledge-center/update-encryption-key-rds)
  — AWS re:Post KB, the re-key path for resources on AWS managed keys.

**Terraform provider documentation:**

- [`aws_backup_plan`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/backup_plan)
  — argument reference verified from the provider's source documentation; source
  of the `rule` / `lifecycle` / `copy_action` / `advanced_backup_setting`
  argument names used in the HCL above.
- [`aws_backup_selection`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/backup_selection)
  — `selection_tag` vs `condition`, and the four condition operators.
- [`aws_backup_vault_lock_configuration`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/backup_vault_lock_configuration)
  — confirms `changeable_for_days` is the compliance-mode switch.
- [`aws_backup_logically_air_gapped_vault`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/backup_logically_air_gapped_vault)
  — confirms the resource exists with `min_retention_days` / `max_retention_days`.

**Community / secondary (used only where flagged, never for a factual claim in
this note):**

- [AWS re:Post — Prevent deletion of a CMK even by the root user](https://repost.aws/questions/QUcluGN53vSt-YyQuzmKK0Bw/prevent-deletion-of-a-cmk-even-by-the-root-user-kms-equivalent-to-backup-vault-lock)
  — cited only to establish that there is no KMS equivalent of Vault Lock.
- Third-party pricing write-ups (N2W, Eon, Cloudchipr, NetApp) — the source of
  the ⚠️-marked rates. **Not authoritative.** Listed for traceability, not as
  evidence.

**Explicitly not found:**

- **No published AWS figure for cross-Region copy job duration or throughput.**
  Searched; nothing authoritative exists. The RPO arithmetic in this note
  therefore uses clearly-labelled illustrative durations and says so.
- **No published AWS restore-duration figures** for any resource type. AWS
  states directly that it provides "no service-level agreements (SLAs) for a
  restore time."
- **No public case study or postmortem found** describing a company's real
  measured AWS Backup cross-Region restore time. If one exists it did not surface
  in this research.
