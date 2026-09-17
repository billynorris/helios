---
title: RDS Postgres vs Aurora Global Database vs Backups-Only — The Decision
service: rds-postgres, aurora-postgresql
tags: [service, multi-region, decision, rds, aurora, postgres, database, cost]
status: researched
replication: n/a — this is a decision note
rpo_achievable: "all three candidates clear RPO 2h by 24x or more; RPO is not the deciding variable"
rto_achievable: "backups-only: hours (fails). RDS replica: ~5-15 min (unverified). Aurora Global DB: ~1-2 min (vendor-claimed, one AWS-published customer test at 2 min)"
meets_targets: conditional
updated: 2026-09-16
---

# RDS Postgres vs Aurora Global Database vs Backups-Only — The Decision

> This note sits **on top of** [[aws-rds-postgres]] and [[aws-aurora-global-database]]. It does not
> re-explain either mechanism. If you have not read those, read them first — everything here is a
> comparison, not a description. Read [[research-brief]] for the targets.

## TL;DR

- **RPO is not the deciding variable and you should stop treating it as one.** RPO 2h is met with
  1000x+ margin by an RDS cross-Region replica (seconds), by Aurora Global Database (sub-second),
  and with 24x margin even by cross-Region automated backups alone (~5 min). Every argument in this
  note is about **RTO, rehearsability and failback**.
- **Steady-state cost is roughly a wash.** Worked from live AWS pricing below: for a 500 GB EU-pair
  database, the warm standby costs ~**$259/month** on RDS and ~**$272/month** on Aurora. Anyone
  claiming "Aurora is the expensive option" or "Aurora saves money" for *this* workload shape is
  guessing. See [[#Cost, worked properly]].
- **The whole Aurora case rests on migration cost, and that cost is real and non-trivial.** Moving
  RDS Postgres → Aurora Postgres is a database migration project: a write-stopping promotion window,
  a Terraform resource-type change that no `moved` block can express, a parameter-group re-homing,
  an instance-class floor that kills `db.t*`, and — the one that actually bites — **you cannot
  create an Aurora read replica of an RDS instance that already has a cross-Region read replica**,
  so you must tear down your DR to build your DR. See [[#The migration cost, honestly]].
- **Recommendation: build Option A + C now (RDS cross-Region replica plus cross-Region automated
  backups). Do not make Aurora a prerequisite for the multi-region programme.** The prerequisites-
  first strategy in [[research-brief]] is the right one and an engine migration is not a
  prerequisite — it is a parallel, separately-funded workstream. Full reasoning and the exact
  conditions that flip this in [[#Recommendation]] and [[#What flips the recommendation]].
- **The thing that will bite:** the recommendation is contingent on a number **nobody has measured**
  — how long `promote-read-replica` actually takes on your data volume. AWS publishes none and no
  public benchmark exists ([[aws-rds-postgres#How long does promotion actually take]]). If that
  number comes back at 12 minutes, Option A has no margin and the recommendation flips to Aurora.
  **Measure it before you spend anything.**

## The three candidates, stated precisely

| | **A — RDS + cross-Region read replica** | **B — Aurora PostgreSQL Global Database** | **C — Cross-Region automated backups only** |
|---|---|---|---|
| Resource | `aws_db_instance` with `replicate_source_db` | `aws_rds_global_cluster` + two `aws_rds_cluster` | `aws_db_instance_automated_backups_replication` |
| What crosses the Region | Physical WAL stream over a managed replication slot | Storage-volume replication on dedicated infrastructure | Snapshots + transaction logs into the destination Region |
| Failover primitive | `promote-read-replica` (irreversible) | `switchover-global-cluster` / `failover-global-cluster` | `restore-db-instance-from-db-snapshot` / PITR |
| Prerequisite | None — you are already on RDS | **A full RDS → Aurora migration** | None |
| Detail | [[aws-rds-postgres#Option 1 — Cross-Region read replica (the only one that meets RTO 15m)]] | [[aws-aurora-global-database#Option 1 — Global database with a warm secondary (recommended for prod)]] | [[aws-rds-postgres#Option 2 — Cross-Region automated backups (aws_db_instance_automated_backups_replication)]] |

C is listed as a candidate because [[research-brief]] asks for the cheap branch to be taken
seriously wherever RPO 2h is the binding constraint. It is not the answer here — but it is not a
strawman either, and it belongs in the estate alongside whichever of A or B wins. See
[[#C is not a competitor, it is a floor]].

## The decision table

Scores are against **RPO 2h / RTO 15m**, active/passive, three independent pairs.

| Dimension | **A — RDS cross-Region replica** | **B — Aurora Global Database** | **C — Backups only** |
|---|---|---|---|
| **Achievable RPO** | Seconds to low minutes. Baseline `ReplicaLag` reports up to 5 min with **no** workload (metric artefact). **Passes with ~1000x margin.** | Sub-second (`AuroraGlobalDBRPOLag` in ms; AWS blog shows an illustrative 483 ms). RPO **0** on a planned switchover. **Passes with ~10⁷x margin.** | ~5 min — RDS ships transaction logs to the destination Region every 5 minutes. **Passes with 24x margin.** |
| **Achievable RTO** | ~5–15 min. API call is instant; the cost is "several minutes or longer" of RDS reboot + crash recovery, then DNS, then connection pools. **No public benchmark exists.** | ~1–2 min. AWS: managed failover "in typically a minute"; one AWS-published customer measured **2 minutes** end to end for cross-Region recovery. | **1 hour → many hours.** You are creating a DB instance from cold and replaying logs, then waiting for lazy loading from S3 to finish before performance is acceptable. **Fails.** |
| **Margin inside 15 min** | **Thin and unquantified.** The database may eat 5–10 of the 15, leaving the app layer nothing. | **Comfortable.** ~13 min of the budget left for [[aws-eks]], DNS and pools. | None. |
| **Cost (steady-state standby)** | ~$259/mo for the 500 GB EU example. Standby instance-hours dominate and **cannot be scaled to zero** — `stop-db-instance` refuses on replicas. | ~$272/mo for the same example, plus replicated write I/O. Headless (0 instances) is far cheaper but **fails RTO** — see [[aws-aurora-global-database#Option 2 — Global database with a headless secondary (fails our RTO)]]. | Snapshot storage only. Roughly an order of magnitude cheaper than A or B. |
| **Migration effort from today's RDS** | **Low.** Additive. One new resource, no downtime, no replacement in plan, no engine change. Weeks, not quarters. | **High.** A database migration per pair (×3), a write-stopping promotion window, a Terraform resource-type change, parameter-group re-homing, and a DR gap during the cutover. Quarters. | **Lowest.** One resource, non-disruptive, reversible. Days. |
| **Operational complexity, steady state** | Medium. You own the replication-slot failure mode: a stalled replica retains WAL **on the primary** until `FreeStorageSpace` hits zero and takes production down. Alarm on `OldestReplicationSlotLag`. | Lower. Replication is off-engine, on dedicated infrastructure — **the slot failure mode does not exist**. But Aurora adds its own: no stop/start, no auto-scaling on secondaries, awkward Terraform. | Lowest. Nothing to monitor but the copy job. |
| **Rehearsability** | **Poor, and this is underrated.** Promotion is irreversible and there is no demote. Every rehearsal either burns the real standby (forcing a multi-hour reseed) or requires building a throwaway replica. Teams therefore rehearse rarely, which is exactly how RTOs rot. | **Good.** `switchover-global-cluster` is RPO 0, ~1 min, preserves topology and is explicitly designed for "regional rotation". Quarterly production rehearsal is realistic. | Trivially rehearsable (restore into a scratch VPC) — and you should, because restore time is the whole risk. |
| **Failback difficulty** | **Bad.** Promotion is one-way. The old primary cannot become a replica. Failback = delete old primary, build a fresh cross-Region replica in the reverse direction ("can take hours"), then a **second** promotion with a **second** outage. A round trip costs **two full reseeds**. | **Good.** Old primary Region auto-rejoins as a secondary when it recovers. Failback is a switchover: RPO 0, ~1 minute, no rebuild. | N/A — you are restoring from backup in whichever direction you need, at hours each way. |
| **Blast radius of DR on production** | Real. The replication slot is a documented path from "standby is unreachable" to "primary is down". | Minimal — **unless** someone sets `rds.global_db_rpo`, which makes the primary block commits. Don't. | None. |
| **Protects against logical corruption?** | **No.** A bad `DELETE` replicates in milliseconds. | **No.** Same. | **Yes.** This is C's unique value. |

### Reading the table

Three honest observations that the table makes hard to miss:

1. **RPO is a non-question.** All three pass. Any conversation that starts "but what about data loss"
   is already solved by the cheapest option on the list.
2. **A and B cost about the same to run.** The difference between them is concentrated in three
   cells: *migration effort*, *rehearsability* and *failback*. That is the actual decision.
3. **A's RTO cell contains an unmeasured number.** Everything downstream of it is provisional.

## Cost, worked properly

All figures pulled on 2026-09-16 from the live pricing feed that backs the public AWS pricing pages
(`b0.p.awsstatic.com/pricing/2.0/meteredUnitMaps/...`, publication date 2026-09-11). On-demand, USD,
no RIs, no Savings Plans. **Re-verify before quoting to finance** — see [[#Sources]].

### Verified unit rates

**RDS for PostgreSQL, `db.r6g.large`, on-demand $/hour:**

| Region | Single-AZ | Multi-AZ |
|---|---|---|
| `eu-west-1` Ireland | $0.252 | $0.504 |
| `eu-west-2` London | $0.264 | $0.529 |
| `us-east-1` N. Virginia | $0.225 | $0.450 |
| `us-west-2` Oregon | $0.225 | $0.450 |
| `ca-central-1` Montreal | $0.243 | $0.486 |
| `ca-west-1` Calgary | $0.243 | $0.486 |

**Aurora PostgreSQL, `db.r6g.large`, on-demand $/hour:**

| Region | Aurora Standard | Aurora I/O-Optimized |
|---|---|---|
| `eu-west-1` | $0.286 | $0.372 |
| `eu-west-2` | $0.304 | $0.395 |
| `us-east-1` | $0.260 | $0.338 |
| `us-west-2` | $0.260 | $0.338 |
| `ca-central-1` | $0.286 | $0.372 |
| `ca-west-1` | $0.286 | $0.372 |

> Note `ca-west-1` is priced **identically to `ca-central-1`** for both engines. Calgary is not a
> cost penalty. `eu-west-2` is ~5–6% above `eu-west-1`. `us-west-2` is identical to `us-east-1`.
> This is a genuinely useful finding for [[region-pair-selection]]: **the standby Region choice is
> not a material cost lever in any of the three pairs.**

**Storage, $/GB-month:**

| | `eu-west-1` | `eu-west-2` | `us-east-1` | `us-west-2` | `ca-central-1` | `ca-west-1` |
|---|---|---|---|---|---|---|
| RDS gp3, Single-AZ | $0.127 | $0.133 | $0.115 | $0.115 | $0.127 | $0.127 |
| RDS gp3, Multi-AZ | $0.254 | $0.266 | $0.230 | $0.230 | $0.254 | $0.254 |
| Aurora Standard storage | $0.110 | $0.100 | $0.100 | $0.100 | $0.110 | $0.110 |
| Aurora I/O-Optimized storage | $0.248 | $0.225 | $0.225 | $0.225 | $0.248 | $0.248 |
| Aurora snapshot storage | $0.021 | $0.022 | $0.021 | $0.021 | $0.023 | $0.023 |

**Aurora I/O, $ per million:**

| | `eu-west-1` | `eu-west-2` | `us-east-1` | `us-west-2` | `ca-central-1` | `ca-west-1` |
|---|---|---|---|---|---|---|
| Aurora I/O operations | $0.22 | $0.20 | $0.20 | $0.20 | $0.22 | $0.22 |
| Global Database replicated write I/O | $0.22 | $0.20 | $0.20 | $0.20 | $0.22 | $0.22 |

**Inter-Region data transfer out** — verified $0.02/GB for **all three pairs, in both directions**;
inbound is free. Full detail and the other two pairs in [[cross-region-connectivity]].

> **Not verified:** the RDS *backup storage* rate (the charge that drives Option C) is published on
> the RDS pricing page under "Backup storage costs" but is not present in the machine-readable feed
> and could not be extracted programmatically. **Look it up before costing Option C.** Do not let
> anyone put a number on C from memory — including this note.

### Worked example: EU pair, 500 GB, `db.r6g.large`, 730 h/month

Primary today: RDS Postgres `db.r6g.large`, Multi-AZ, 500 GB gp3, in `eu-west-1`.

| Line | **A — RDS + replica** | **B — Aurora Global DB** |
|---|---|---|
| Primary compute | Multi-AZ `db.r6g.large`: $0.504 × 730 = **$367.92** | Writer + 1 in-Region reader: 2 × $0.286 × 730 = **$417.56** |
| Primary storage | 500 GB Multi-AZ gp3 @ $0.254 = **$127.00** | 500 GB @ $0.110 = **$55.00** |
| Primary I/O | included in gp3 baseline | **workload-dependent**, $0.22/million |
| **Primary subtotal** | **$494.92** | **$472.56 + I/O** |
| Standby compute | `db.r6g.large` Single-AZ `eu-west-2`: $0.264 × 730 = **$192.72** | 1 reader `db.r6g.large` `eu-west-2`: $0.304 × 730 = **$221.92** |
| Standby storage | 500 GB Single-AZ gp3 `eu-west-2` @ $0.133 = **$66.50** | 500 GB @ $0.100 = **$50.00** |
| Replication charge | WAL volume × $0.02/GB | replicated write I/O × $0.20/million, **plus** $0.02/GB transfer |
| **Standby subtotal (the DR delta)** | **~$259.22 + WAL transfer** | **~$271.92 + replicated I/O + transfer** |

**The DR delta differs by about 5%.** That is inside the noise of instance-class choice, RI coverage
and a month's WAL volume. Two consequences:

1. **Do not choose Aurora to save money on DR, and do not reject it because it "costs more".**
   Neither is true at this shape.
2. **The primary-side lines move more than the standby lines.** Aurora trades cheaper storage
   ($0.110 vs $0.254/GB-month, because Aurora has no Multi-AZ storage multiplier) for more expensive
   compute and a brand-new, unbounded I/O line. Whether B is cheaper or dearer *overall* is decided
   by your `VolumeWriteIOPs`, not by anything in the DR design. A write-heavy workload can make
   Aurora Standard much worse and Aurora I/O-Optimized much better — model it from real metrics
   before anyone signs anything. See [[aws-aurora-global-database#Cost]].

### Where the cost levers actually are

| Lever | Saving | Applies to |
|---|---|---|
| **No warm standby in non-production** (`enable_standby_replica = false`, or headless secondary) | ~100% of the standby compute in those environments | A and B |
| Single-AZ standby rather than Multi-AZ | 50% of standby compute | A (B's secondary is single-instance by design) |
| Reserved Instances / Savings Plans on the standby | The standby runs 24/7 forever. It is the most reservable workload in the estate. **Check RI availability in `ca-west-1` specifically.** | A and B |
| Backups-only for any tier whose RTO can be relaxed | ~90% | C |

Note what is *not* on that list: choosing the cheaper standby Region. There isn't one.

## The migration cost, honestly

This section exists because the Aurora option is usually costed as "the price difference between two
instance types" and it is not. Option B's real price is a migration project, repeated three times.

### The mechanism, and its two hard blockers

The supported near-zero-downtime path is **RDS → Aurora via an Aurora read replica**: `create-db-cluster`
with `--replication-source-identifier` pointing at the RDS instance ARN, then `create-db-instance`,
then `promote-read-replica-db-cluster`.

> **Blocker 1 — you cannot build Aurora while your cross-Region DR exists.** AWS: "You can't create
> an Aurora read replica if your RDS for PostgreSQL DB instance already has an Aurora read replica
> **or if it has a cross-Region read replica**."
>
> Read that again in the context of the sequencing question. If you build Option A first (which this
> note recommends) and later decide to move to Aurora, **you must delete the cross-Region read
> replica before you can start the Aurora migration**, and you do not get it back until the Aurora
> global database is built on the other side. You are choosing to run without cross-Region DR for
> the duration of a database migration. That window has to be planned, risk-accepted and probably
> covered by keeping Option C (replicated automated backups) switched on throughout.

> **Blocker 2 — the global database is a second, separate phase.** From
> [[aws-aurora-global-database#B. On RDS for PostgreSQL today, want Aurora Global Database]]: "If the
> primary DB cluster of your global database is based on a replica of an Amazon RDS PostgreSQL
> instance, you can't create a secondary cluster... Attempts to do so **time out**." You must fully
> promote and detach from RDS first, *then* attach the secondary Region. Two phases, and the DR gap
> spans both.

### The line items nobody puts in the estimate

| Cost | Why it is not free |
|---|---|
| **Write-stopping promotion window** | AWS's own procedure is: "Stop all database write workload on the source", compare `pg_current_wal_lsn()` to `pg_last_wal_replay_lsn()`, then promote — and the console step warns "This may take a few minutes and **can cause downtime**." This is a real maintenance window per environment, per Region, with writes stopped. |
| **Seeding time** | "It can take **several hours per terabyte** of data for the migration to complete." Plan the window around your actual size, and note the WAL retained on the source during seeding eats `FreeStorageSpace` — the same slot mechanics as [[aws-rds-postgres#Gotchas]] item 2, now applied to a migration. |
| **Terraform resource-type change** | `aws_db_instance` → `aws_rds_cluster` + `aws_rds_cluster_instance` + `aws_rds_global_cluster`. Different resource types, so **no `moved` block and no `terraform state mv` will do it**. It is create-new / import / `state rm` old, with the cookiecutter module's entire variable surface changing shape. In a heavily templated monorepo that is a template change that touches every environment. See [[terraform-repo-structure]]. |
| **Parameter-group re-homing** | `aws_db_parameter_group` becomes **two** objects: `aws_rds_cluster_parameter_group` (cluster-level) and `aws_db_parameter_group` (instance-level). Every tuned parameter has to be placed at the correct level. It is not a rename; get it wrong and a parameter silently reverts to default. |
| **Instance-class floor** | Global Database "requires DB instance classes that are optimized for memory-intensive applications... we recommend that you use a **db.r5 or higher**". Any environment on `db.t3`/`db.t4g` to save money needs a class change — and that is a cost *increase* in exactly the environments where the DR requirement is weakest. |
| **Secrets Manager integration must be turned off** | "When you add a Region to a global database, you must first turn off Secrets Manager integration for the DB instance." If you use `manage_master_user_password = true` you need a replacement credential-rotation design — and it has to be reconciled with the Secrets Manager replication work [[research-brief]] says is already complete. |
| **Endpoint change** | Aurora's writer endpoint is a new hostname. Every connection string, every secret, every config. If you have already introduced the Route 53 CNAME indirection recommended in [[aws-rds-postgres#Endpoint management at failover]] this is one `UPSERT`; if you have not, it is a code change across every service. **Do the CNAME work first regardless of which option wins** — it is useful under A and it de-risks B. |
| **Unlogged tables** | AWS explicitly flags them: "it's important to identify and handle unlogged tables appropriately", and on Aurora they are readable **only from the writer**. Converting `UNLOGGED` → `LOGGED` rewrites the table, which is expensive on large tables. Audit for them before you plan the window. |
| **Performance re-qualification** | Aurora's storage and I/O behaviour is different. Every latency SLO, every slow-query budget, every autovacuum tuning decision has to be re-validated. Extensions are at parity, but *behaviour* is not. |
| **Double-running** | Both engines live simultaneously for the bake period, in every environment you migrate. |
| **× 3** | Three independent deployments per [[research-brief]]. Three migrations, three windows, three bake periods. |

### The honest summary

Option A is **additive and reversible**: a new resource, no downtime, no `ForceNew`, and if it
doesn't work out you delete it. Option B is a **one-way engine migration** with a planned write
outage, a temporary loss of cross-Region DR, and a Terraform refactor that lands in every
environment of a templated monorepo.

That asymmetry is the decision. It is not that Aurora is worse — on failover, failback and
rehearsability Aurora is clearly better, and [[aws-aurora-global-database]] makes that case well. It
is that Aurora's benefits are bought with a project, and A's benefits are bought with a resource
block.

## C is not a competitor, it is a floor

Option C cannot meet RTO 15m and is not being proposed as the answer. It should nonetheless be
switched on **under whichever of A or B you choose**, permanently, for three reasons:

1. **It is the only one of the three that protects against logical corruption.** A bad migration, a
   `DELETE` without a `WHERE`, a compromised credential — all three replicate faithfully and
   instantly under A and B. Backups are the only rewind.
2. **It is the DR you still have when the DR is broken.** Replica reseeding after a slot blowout
   takes hours. Aurora refusing a managed failover on a version mismatch drops you to a
   topology-destroying manual detach. In both cases the replicated backup is what is left.
3. **It is the DR posture during the Aurora migration**, if you ever do it — see Blocker 1 above.

It is also the correct *sole* answer for non-production, where a 15-minute RTO is not a real
requirement. Make it a cookiecutter variable, not a debate.

## Recommendation

> **Build Option A + Option C now. Do not put an Aurora migration on the critical path of the
> multi-region programme.**

Reasoning, in order of weight:

1. **RTO 15m is met by A on current evidence, and A is additive.** A cross-Region read replica is one
   resource, no downtime, no replacement in `terraform plan`, and it fits the prerequisites-first
   strategy in [[research-brief]] exactly — the database is warm and populated in the standby Region
   before EKS ever lands there. That is precisely the shape the programme is already executing for
   Secrets Manager and DynamoDB.
2. **The cost argument is neutral, so it cannot justify a migration.** ~5% apart. If Aurora were 40%
   cheaper the answer would be different.
3. **An engine migration is not a prerequisite.** It is a database project that happens to also
   change the DR mechanism. Coupling it to the multi-region programme means the programme cannot
   ship the standby Region until a migration lands in three environments. That is a scheduling
   disaster and it buys nothing that A does not already buy on the RPO axis.
4. **The known weaknesses of A are known, and two of the three are mitigable today.** Slot-driven
   storage exhaustion → alarm on `OldestReplicationSlotLag` and `TransactionLogsDiskUsage` on the
   *primary*. `backing-up` blocking promotion → set non-overlapping backup windows. The third —
   irreversible promotion and expensive failback — is **not** mitigable, and is the strongest single
   argument for B. See below.
5. **The decision is reversible in the right direction.** Choosing A now and migrating to Aurora in
   twelve months costs you the DR gap described in Blocker 1 and nothing else. Choosing B now and
   discovering Aurora's I/O charges are ruinous on your write profile costs you a migration back.

**What to do in the next four weeks, in order:**

1. **Measure `promote-read-replica`** on a throwaway replica at production data volume, per pair.
   One number, ~20 minutes of work, and it is the input to everything else. This is
   [[aws-rds-postgres#Open questions]] item 1 and it is blocking.
2. **Settle the RTO definition** — 15 minutes from *incident start* or from *decision to fail over*?
   Flagged as an open thread in the vault's `CLAUDE.md` and it materially changes this note's
   answer. From decision-to-failover (the [[research-brief]] reading), A has margin. From
   incident-start, detection and human decision eat 5–10 minutes and A very likely does not.
3. **Turn on cross-Region automated backups (Option C) immediately**, everywhere, prod and non-prod.
   Cheap, non-disruptive, and it is the only logical-corruption protection in the estate.
4. **Introduce the Route 53 CNAME endpoint indirection** before anything else. It is required under
   A, it de-risks B, and doing it during an incident is how RTOs are lost.
5. **Then** build the replica in prod, and run a game day.

## What flips the recommendation

Write these down as explicit triggers. The point of a decision note is that the decision can be
re-opened by evidence rather than by opinion.

**Flip to B (Aurora Global Database) if any of these is true:**

| Trigger | Why it flips |
|---|---|
| **Measured promotion + reboot exceeds ~10 minutes** at production volume | Leaves under 5 minutes for DNS, connection pools and the entire application tier. That is not an RTO, it is a hope. Aurora's ~1–2 min gives the app layer 13 minutes. |
| **The RTO is defined from incident start, not from decision** | Detection + human go/no-go realistically costs 5–10 minutes. A's margin disappears entirely; B's survives. |
| **Failback within the same day is a business or compliance requirement** | A's failback is two full reseeds ("can take hours" each) plus a second write outage. B's is a ~1-minute switchover at RPO 0 with the old Region auto-rejoining. This is the single largest capability gap between the two and it is not closable. |
| **A regulator or customer contract requires a rehearsed, non-destructive DR test** | You cannot non-destructively rehearse an RDS promotion — it is irreversible and burns the standby. Aurora's `switchover-global-cluster` is designed for exactly this and can be scheduled quarterly like a deploy. |
| **The database grows to where a reseed is measured in days** | Both the initial replica build and every failback reseed scale with data volume. At multi-TB, A's failback stops being a runbook and becomes a project. Aurora's storage-layer replication removes the reseed entirely. |
| **The slot failure mode bites once in production** | If a stalled standby has already taken the primary down by filling its disk with retained WAL, the argument is over. That failure mode does not exist on Aurora. |
| **Aurora is being adopted anyway for non-DR reasons** | Read scaling, Serverless v2, Blue/Green deployments, or I/O-Optimized economics on a write-heavy profile. If the migration is happening regardless, its cost is no longer attributable to DR and B wins on every remaining axis. |

**Flip to C-only for a given environment or tier if:**

- It is non-production. Default this in cookiecutter rather than discussing it per environment.
- The tier's RTO is genuinely hours. Then the ~$260/month standby is pure waste and C's ~5-minute
  RPO already clears the 2-hour target with 24x margin.

**Do not flip to B for:**

- Cost. It is a wash (±5%) at this shape.
- RPO. Both pass by orders of magnitude. Sub-second versus seconds is a difference with no business
  meaning when the target is two hours.
- "Aurora is the modern/strategic choice." That is not a DR argument. If it is the real driver, make
  it explicitly and fund it as a database modernisation project, not as a multi-region prerequisite.

## Things that are true of both A and B and are therefore not differentiators

Worth listing so they stop appearing in the argument:

- **Both need a destination-Region KMS key.** Keys are regional, full stop
  ([[aws-rds-postgres#KMS and the destination-region key]], [[aws-kms]]). Neither is easier.
- **Both are hurt by parameter-group drift**, and both have the same fix: one shared `locals` map
  consumed by both Regions, an SCP denying manual modification, and a scheduled diff job.
- **Both hit the `ca-west-1` opt-in / STS trap.** `AWS_STS_REGIONAL_ENDPOINTS=regional` in CI, or v2
  global tokens on the account ([[aws-rds-postgres#The STS opt-in-Region trap]]).
- **Both are supported in all three pairs**, including `ca-central-1 → ca-west-1`, and `ca-west-1` is
  priced identically to `ca-central-1`. The CA pair is not the constraint anyone expected it to be.
- **Both leave the application layer as the long pole.** Neither the database nor the DNS is what
  loses a 15-minute RTO — it is a connection pool holding 50 established sockets to a black-holed IP
  for `tcp_retries2` ≈ 15 minutes. Restart the pods. See
  [[aws-rds-postgres#The connection pool trap, which is worse than DNS]] and [[aws-eks]].
- **Neither protects against logical corruption.** Only C does.
- **Both need the Region-role to be a Terraform *variable*, not a structural property of the code.**
  Otherwise failback (or "don't fail back") requires moving resources between state files during an
  incident.

## Terraform: making the choice a variable rather than a fork

The worst outcome is a repo with a `modules/rds-postgres/` and a `modules/aurora-global/` that drift
apart while someone decides. In a cookiecutter monorepo, model the choice as **one input on the
stack**, with the engine modules behind it, so that a future migration is a variable change plus a
state operation rather than a rewrite of every environment.

```hcl
# stacks/<env>/variables.tf
variable "db_engine_strategy" {
  description = <<-EOT
    Which cross-Region database strategy this environment uses.
      "rds_replica"     - RDS Postgres primary + cross-Region read replica  (Option A)
      "aurora_global"   - Aurora PostgreSQL Global Database                 (Option B)
      "backups_only"    - RDS Postgres primary, cross-Region automated backups only (Option C)
    See [[rds-vs-aurora-decision]]. Default is deliberately the cheap one so that a new
    environment does not silently acquire a $260/month standby.
  EOT
  type    = string
  default = "backups_only"

  validation {
    condition     = contains(["rds_replica", "aurora_global", "backups_only"], var.db_engine_strategy)
    error_message = "db_engine_strategy must be rds_replica, aurora_global or backups_only."
  }
}
```

```hcl
# stacks/<env>/database.tf
locals {
  is_rds    = var.db_engine_strategy != "aurora_global"
  is_aurora = var.db_engine_strategy == "aurora_global"
  warm_standby = var.db_engine_strategy != "backups_only"

  # ONE source of truth for tuning, consumed by whichever engine is active and by
  # BOTH regions. This is the only thing that reliably prevents parameter drift.
  pg_parameters = {
    "log_min_duration_statement" = "1000"
    "shared_preload_libraries"   = "pg_stat_statements"
  }
}

module "db_rds" {
  source = "../../modules/rds-postgres"
  count  = local.is_rds ? 1 : 0

  providers = {
    aws.primary = aws.primary
    aws.standby = aws.standby
  }

  name        = var.service_name
  environment = var.environment

  # The warm cross-Region replica is Option A. Option C omits it but keeps
  # backup replication, which is ON in every case.
  enable_standby_replica       = local.warm_standby
  enable_backup_replication    = true

  pg_parameters        = local.pg_parameters
  primary_kms_key_arn  = module.kms_primary.rds_key_arn
  standby_kms_key_arn  = module.kms_standby.rds_key_arn   # MUST be a standby-region key
}

module "db_aurora_global" {
  source = "../../modules/aurora-global"
  count  = local.is_aurora ? 1 : 0

  providers = {
    aws.primary = aws.primary
    aws.standby = aws.standby
  }

  name        = var.service_name
  environment = var.environment

  # Headless secondary is the non-prod cost lever and FAILS RTO 15m.
  # Production must be >= 1.
  secondary_instance_count = var.environment == "prod" ? 1 : 0

  pg_parameters        = local.pg_parameters
  primary_kms_key_arn  = module.kms_primary.rds_key_arn
  standby_kms_key_arn  = module.kms_standby.rds_key_arn
}

# The application NEVER holds a raw engine endpoint. This record is the seam that
# makes an engine migration (and a failover) a one-line UPSERT rather than a code change.
# Build this FIRST, whichever option wins.
resource "aws_route53_record" "db_writer" {
  provider = aws.primary
  zone_id  = var.private_zone_id
  name     = "db.${var.environment}.internal.example.com"
  type     = "CNAME"
  ttl      = 5     # set it now; lowering a TTL during an incident does nothing for 300s
  records  = [
    local.is_aurora
      ? module.db_aurora_global[0].global_writer_endpoint
      : module.db_rds[0].primary_endpoint_address
  ]
}
```

Three things this shape buys you, and they are the reason it is worth the `count` gymnastics:

1. **The strategy is a variable per environment**, so non-prod can sit on `backups_only` while prod
   sits on `rds_replica`, from one template. That is most of the cost saving in the whole programme.
2. **`local.pg_parameters` is consumed by whichever engine is live and by both Regions**, so drift is
   impossible through Terraform. (It is still possible through the console — SCP-deny
   `rds:ModifyDBParameterGroup` and `rds:ModifyDBClusterParameterGroup` to everyone but CI.)
3. **The CNAME is engine-agnostic**, so the eventual A → B migration does not touch a single
   application config.

What it does **not** buy you: flipping `db_engine_strategy` from `rds_replica` to `aurora_global` is
**not** an apply. It plans a destroy of your production database and a create of a new cluster. Put a
loud comment on the variable saying so, and treat the migration as the scripted, out-of-band
procedure it is — Terraform reconciles state *after* the migration, it does not perform it.

## Open questions

1. **How long does `promote-read-replica` take on our production data volume, per pair?** Blocking.
   No public number exists. 20 minutes of work to answer.
2. **Is the RTO measured from incident start or from decision-to-fail-over?** Flagged in the vault's
   `CLAUDE.md`. Changes the answer.
3. **What is our actual `VolumeWriteIOPs` / WAL generation rate?** Decides whether Aurora is cheaper
   or dearer overall, and sizes the cross-Region transfer bill under both options.
4. **Is failback a business requirement, and on what timescale?** If "same day", the recommendation
   flips to Aurora. If "eventually, or never — the pair is symmetric", A is fine.
5. **Are we contractually or contractually-adjacent obliged to demonstrate a DR test?** If a
   non-destructive rehearsal is required, A cannot provide one.
6. **What is the current RDS backup storage rate in each standby Region?** Needed to cost Option C.
   Not machine-extractable; read it off the pricing page.
7. **Is the production primary encrypted?** If not, neither A nor B works as described and the whole
   plan changes ([[aws-rds-postgres#Migration path from single-region]], step 0).
8. **Does anything consume logical replication / CDC from the primary?** Those slots survive neither
   an RDS promotion (pre-PG17) nor an engine migration, and re-bootstrapping them is not currently
   in anyone's runbook.

## Sources

- [Creating a read replica in a different AWS Region — RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.XRgn.html) — the mechanism behind Option A; four-phase creation, "can take hours", the PostgreSQL-specific no-auto-promote-on-source-delete behaviour.
- [Promoting a read replica to be a standalone DB instance — RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.Promote.html) — "several minutes or longer", the reboot, the `backing-up` block, and the explicit statement that a promoted instance can no longer be a replication target (the root of A's failback problem).
- [Migrating data from an RDS for PostgreSQL DB instance to an Aurora PostgreSQL DB cluster using an Aurora read replica — Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Migrating.RDSPostgreSQL.Replica.html) — **the single most important source for the migration-cost section.** Same-Region-and-account only; "You can't create an Aurora read replica if your RDS for PostgreSQL DB instance already has an Aurora read replica or if it has a cross-Region read replica" (Blocker 1); "several hours per terabyte"; the stop-writes / compare-LSN / promote procedure; "This may take a few minutes and can cause downtime"; the `FreeStorageSpace` / `OldestReplicationSlotLag` / `RDSToAuroraPostgreSQLReplicaLag` metrics to watch during seeding; and the pointer to handling unlogged tables before migrating.
- [Using Amazon Aurora Global Database — Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html) — the restriction that a global database primary derived from an RDS replica cannot take a secondary cluster (Blocker 2), and the Secrets Manager incompatibility.
- [Configuration requirements of an Amazon Aurora global database — Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.configuration.requirements.html) — the "db.r5 or higher" memory-optimised instance-class floor that excludes `db.t3`/`db.t4g`.
- [Using switchover or failover in Amazon Aurora Global Database — Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html) — switchover RPO 0 / "within a few minutes" for managed failover, the old primary Region auto-rejoining, and the `rds.global_db_rpo` two-Region warning. The basis for B's failback and rehearsability advantage.
- [How a large financial AWS customer implemented HA and DR for Amazon Aurora PostgreSQL using Global Database and Amazon RDS Proxy — AWS Database Blog](https://aws.amazon.com/blogs/database/how-a-large-financial-aws-customer-implemented-ha-and-dr-for-amazon-aurora-postgresql-using-global-database-and-amazon-rds-proxy/) — the only real customer case study with numbers: cross-Region recovery measured at 2 minutes.
- [Working with unlogged tables in Aurora PostgreSQL — Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-postgresql-unlogged-tables.html) — unlogged tables are readable only from the writer on Aurora; relevant to the migration audit.
- [Amazon RDS for PostgreSQL pricing](https://aws.amazon.com/rds/postgresql/pricing/) and [Amazon Aurora pricing](https://aws.amazon.com/rds/aurora/pricing/) — the public pages. The tables in [[#Cost, worked properly]] were extracted from the machine-readable feeds these pages render from: `https://b0.p.awsstatic.com/pricing/2.0/meteredUnitMaps/rds/USD/current/rds-postgresql-ondemand.json`, `.../rds-aurora-ondemand.json`, `.../rds-aurora-storage.json`, `.../rds-storage.json` and `https://b0.p.awsstatic.com/pricing/2.0/meteredUnitMaps/datatransfer/USD/current/datatransfer.json`, publication date 2026-09-11.
- **Not found:** any public benchmark of RDS for PostgreSQL cross-Region read replica **promotion duration**. AWS's own DR comparison blog publishes only Good/Better/Best with no numbers. This is the number the recommendation hinges on and it must be measured in-house.
- **Not verified:** the RDS backup storage $/GiB-month rate. Present on the pricing page under "Backup storage costs" but absent from the machine-readable feed and not extractable by automated fetch. Option C cannot be costed until someone reads it off the page.
