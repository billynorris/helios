---
title: Amazon RDS for PostgreSQL — Multi-Region
service: rds-postgres
tags: [service, multi-region, rds, postgres, database, stateful]
status: researched
replication: native (cross-Region read replica) | native (cross-Region automated backups) | manual (logical/DMS)
rpo_achievable: "seconds–minutes with a live cross-Region read replica; ~5 min with cross-Region automated backups (log shipping interval)"
rto_achievable: "~5–15 min with a warm promoted replica (tight, needs rehearsal); 1–several hours from a cross-Region backup restore (FAILS RTO)"
meets_targets: conditional
updated: 2026-09-16
---

# Amazon RDS for PostgreSQL — Multi-Region

## TL;DR

- **RDS Postgres has exactly one mechanism that can meet a 15-minute RTO: a running cross-Region read replica that you `PromoteReadReplica`.** Everything else (snapshot copy, cross-Region automated backups, AWS Backup copy jobs) requires *creating a DB instance at failover time*, and that fails RTO. [[research-brief]] calls this out generically; for RDS it is the whole story.
- **Promotion is irreversible and AWS will not tell you how long it takes.** The docs say "several minutes or longer to complete, depending on the size of the read replica" and that RDS *reboots* the instance as part of it. There is **no public, reproducible benchmark** for RDS Postgres promotion duration — see [[#How long does promotion actually take]]. Budget 5–10 minutes of the 15 and measure it yourself in a game day.
- **Because RPO is only 2 hours, there is a genuinely cheaper branch**: `aws_db_instance_automated_backups_replication` ships snapshots *and* transaction logs to the standby Region continuously (logs every 5 minutes), for roughly the price of storage, with **no standby instance running at all**. It hits RPO 2h with 24x margin. It cannot hit RTO 15m. Both branches are documented in [[#Decisions to make]]; the recommendation is **both**, not either.
- **The KMS interaction is mandatory, not optional.** An encrypted cross-Region replica *must* be given a KMS key that lives in the destination Region, because "KMS keys are specific to the AWS Region that they are created in". In Terraform, `storage_encrypted` is **ignored** on a cross-Region replica and `kms_key_id` is what actually does the work. See [[#KMS and the destination-region key]] and [[aws-kms]].
- **The thing that will bite:** for PostgreSQL specifically, if the source instance is deleted, the cross-Region replica is **not** promoted — its replication status goes to `terminated` and it just sits there, read-only, until a human promotes it. This differs from MySQL/MariaDB/Oracle/SQL Server/Db2, where deleting the source *does* auto-promote. Do not carry a mental model over from a MySQL estate.

## Does this service cross regions at all?

An RDS DB instance is a **regional** resource and its endpoint is a regional DNS name. Nothing about it is global. But RDS does give you three native cross-Region primitives, all of which are real AWS-managed replication rather than scripts you write:

| Primitive | What crosses the Region boundary | Continuous? |
|---|---|---|
| Cross-Region read replica | Physical WAL stream over a managed replication slot | Yes, continuously |
| Cross-Region automated backups | Snapshots + transaction logs into the destination Region | Yes, snapshots daily, logs ~5 min |
| Snapshot copy (`aws_db_snapshot_copy`) | One snapshot, on demand | No |

A DB **subnet group**, **parameter group**, **option group** and **security group** are all regional and none of them cross. They must be mirrored, and version drift between the two copies is a classic way to turn a failover into an outage — see [[#Parameter, option, subnet and security groups]].

### Region availability for the three pairs

Cross-Region read replicas for RDS for PostgreSQL are documented as "available in **all Regions**" for PostgreSQL 10 through 18. So all three pairs in [[research-brief]] work: `eu-west-1→eu-west-2`, `us-east-1→us-west-2`, `ca-central-1→ca-west-1`.

Cross-Region **automated backups** are *not* available for every pair — AWS publishes an explicit source→destination matrix. All three of our pairs are on it:

| Source | Destination available? | Source |
|---|---|---|
| Europe (Ireland) | Europe (London) — yes | AWS docs matrix |
| US East (N. Virginia) | US West (Oregon) — yes | AWS docs matrix |
| Canada (Central) | **Canada West (Calgary) — yes** | AWS docs matrix |
| Canada West (Calgary) | Canada (Central) — yes (so failback backup replication also works) | AWS docs matrix |

That Canada row is worth noting: `ca-central-1 ↔ ca-west-1` is one of the *narrowest* entries in the whole table (Calgary's only listed destination is Montreal), but it exists in both directions, which is exactly what an active/passive pair with failback needs.

### `ca-west-1` — what actually differs

This is the Region [[research-brief]] asks us to be suspicious of. Findings:

- **Aurora and RDS Postgres both exist there.** `ca-west-1` launched with ~70 services including "Amazon Aurora", "Aurora PostgreSQL" and "Amazon RDS". Aurora Global Database is listed as supported in Canada West (Calgary) for Aurora PostgreSQL 11 through 18, same version floors as every other Region — see [[aws-aurora-global-database]]. **The CA pair is not invalidated by a Global Database gap.**
- **`ca-west-1` is an opt-in Region.** Your account is not enabled there and your IAM identity is not replicated there until you opt in. This has a concrete, painful consequence for cross-Region replica creation — see [[#The STS opt-in-Region trap]].
- **Instance class availability is the real gap.** The launch announcement lists EC2 families `C5, M5, M5d, R5, C6g, C6gn, C6i, C6id, M6g, M6gd, M6i, M6id, R6g, R6i, R6id, I4i, I3en, T3, T4g`. That is **no 7th-generation anything** — no M7g/R7g (Graviton3), no M7i/R7i. RDS instance-class availability tracks EC2 availability but is not identical to it, and the Region has been open long enough that this may have moved. **Do not assume.** Verify before you write the standby module:

  ```bash
  aws rds describe-orderable-db-instance-options \
    --region ca-west-1 --engine postgres --engine-version 16.4 \
    --query 'OrderableDBInstanceOptions[].DBInstanceClass' --output text | tr '\t' '\n' | sort -u
  ```

  Run the identical command against `ca-central-1` and diff. If the primary runs `db.r7g.*` and Calgary only offers `db.r6g.*`, the standby is on a different class — which is allowed (AWS only *recommends* same-or-larger) but changes your cost model and your post-failover capacity headroom. This is the single most likely thing to invalidate a naive "same module, different provider alias" approach. Same check applies to `--engine aurora-postgresql`.
- **Engine minor versions can lag.** `describe-db-engine-versions --region ca-west-1 --engine postgres` vs `ca-central-1`. If the primary is pinned to a minor version that does not exist in Calgary, `create-db-instance-read-replica` fails outright. This is a build-time failure, not a failover-time one, so you find it early — but it constrains your upgrade ordering forever after (see [[#Gotchas]]).

## Replication / mirroring options

### Option 1 — Cross-Region read replica (the only one that meets RTO 15m)

**Mechanics.** RDS creates the replica in four phases, and the docs are explicit that "this process can take **hours** to complete" depending on data volume:

1. Source instance goes to `modifying` while RDS configures it as a replication source.
2. RDS takes an automated snapshot of the source in the source Region (named `rds:<InstanceID>-<timestamp>`).
3. RDS copies that snapshot cross-Region.
4. RDS loads the replica from the copied snapshot, then starts streaming.

Once running, RDS Postgres cross-Region replicas use **physical streaming replication with a replication slot** on the source (not `archive_command`/`wal_keep_size` WAL shipping). The slot is what makes this robust: WAL accumulates on the *source* during a network partition instead of the replica falling off a cliff. It is also what makes it dangerous — see the storage-exhaustion gotcha below.

**Lag characteristics.** AWS's own best-practices post gives the one number that surprises people: with **no active workload**, expect **up to 5 minutes** of reported lag on a cross-Region replica. That is an artefact of how `ReplicaLag` is computed when there is nothing to replicate, not a real data gap. Under write load, lag tracks the write rate and the inter-Region RTT. Documented lag drivers:

- Replica instance class smaller than source (replica replays the same write volume *and* serves reads).
- Different storage type / lower provisioned IOPS on the replica.
- Bursts of WAL from bulk operations.
- Exclusive locks on the source (`ALTER TABLE`, `TRUNCATE`) serialising replay.
- Long-running queries on the replica conflicting with recovery (`hot_standby_feedback`, `max_standby_streaming_delay`).

Watch `ReplicaLag`, `OldestReplicationSlotLag`, `TransactionLogsDiskUsage` and `FreeStorageSpace` — the last two on the **source**, because slot-retained WAL eats source storage. AWS's documented remedy when a slot has retained too much WAL is brutal: "drop the existing cross-Region replica and create a new one."

**RPO delivered.** Seconds to low minutes under normal conditions. Against a 2-hour RPO target this is enormous headroom — the replica is bought for RTO, not RPO. Worth saying out loud in the cost conversation.

**Limits.** RDS "can't guarantee more than five cross-Region read replica DB instances" per source (ACL entry limits on the source VPC), and 20 concurrent create-replica requests per destination Region per account. Neither binds us at one replica per pair.

### Option 2 — Cross-Region automated backups (`aws_db_instance_automated_backups_replication`)

Genuinely underused, and given **RPO 2h** it deserves a serious look rather than a footnote.

You enable it on the source instance and RDS "initiates a cross-Region copy of all snapshots and transaction logs as soon as they are ready on the DB instance." RDS uploads transaction logs to S3 **every five minutes**, so the destination Region holds a continuously-advancing PITR window. Restore with `restore-db-instance-from-db-snapshot` or PITR against the *replicated* backup in the destination Region.

- **RPO: ~5 minutes.** 24x inside the 2-hour target.
- **RTO: fails.** You are creating a DB instance from scratch. Instance provisioning plus lazy-loading means the docs warn that after restore "its volumes continue to load data blocks from Amazon S3 in the background... performance might not be at its fullest until initialization completes" (track `StorageOperationStatus` / `StorageOperationPercentProgress` on `DescribeDBInstances`). Plus PITR replays transaction logs on top of the base snapshot. For anything past a few hundred GB this is comfortably outside 15 minutes.
- **Cost: storage + transfer only.** No standby compute. This is the cheap option.
- **Limits:** not supported for Multi-AZ **DB clusters** (it *is* supported for Multi-AZ DB *instances*). Default quota 20 cross-Region automated backups per account. Encryption needs a destination-Region KMS key, same rule as replicas.

**This is not a competitor to the replica, it is a complement.** The replica protects against Region loss. The replicated backup protects against the thing a replica cannot protect against: **logical corruption**, which replicates faithfully and instantly. A `DELETE FROM` with a bad `WHERE` clause is on the replica before you finish reading the error. Backups are your only rewind. Run both.

### Option 3 — PostgreSQL logical replication / `pglogical`

Native logical replication (publication/subscription, PG10+) or the `pglogical` extension. On RDS you set `rds.logical_replication = 1` in the parameter group, which requires a reboot.

Choose logical over physical when:

- **The two ends must run different major versions.** Physical replication cannot cross a major version; logical can. This is the standard mechanism for a near-zero-downtime major upgrade, and it works cross-Region.
- **You only need a subset of tables.** Physical replicates the entire instance, byte for byte, including the tables you would rather not ship across a border — relevant if [[data-residency]] constrains what may leave a Region.
- **You are moving between RDS and Aurora** (in either direction). Physical replication cannot cross the RDS↔Aurora boundary; Aurora replicas of an RDS Postgres source are not a thing the way they are for MySQL. Logical replication is the bridge. See [[rds-vs-aurora-decision]].
- **You want the target writable** while replication is running (e.g. for a phased cutover with reverse replication armed).

Costs of logical replication, and they are real:

- **Sequences are not replicated.** Logical replication carries the *values* in serial columns but not the sequence objects' state. After a switchover you must `setval()` every sequence or the new primary starts handing out duplicate primary keys. (Contrast: a **physical** replica has correct sequences because they are in the data files, in the WAL stream.)
- **DDL is not replicated.** Schema changes must be applied to both ends, in the right order, forever.
- **No TRUNCATE before PG11, no large objects, ever.**
- You are now operating replication yourself. Slot lag, subscription state, conflict resolution — all yours.

For an active/passive DR standby of the *same* major version, logical replication is strictly worse than the native cross-Region read replica. Use it for **migrations**, not for **DR**.

### Option 4 — AWS DMS

DMS gives you what logical replication gives you plus: heterogeneous engines, transformation rules, table-level filtering with a managed control plane, and validation. Choose it over hand-rolled logical replication when you want the ongoing-replication task monitored and restartable by AWS rather than by you, or when the source is not a PostgreSQL you control.

DMS as a *steady-state DR mechanism* is a bad idea — it is a migration tool with a replication instance you now have to size, patch and monitor, and CDC tasks fail in ways that are silent until you check. See [[aws-dms]]. Use it for the one-time move, tear it down after.

### Option 5 — Do nothing, deploy an empty second copy

Explicitly wrong for RDS. A stateless service can do this (see [[aws-lambda]], [[aws-api-gateway]]); a database cannot. Listed only so the note doesn't silently skip the template's question.

## RPO / RTO analysis

Against **RPO 2h / RTO 15m**:

| Mechanism | RPO | RTO | Meets targets? |
|---|---|---|---|
| Cross-Region read replica, promoted | seconds–minutes | ~5–15 min (see below) | **Conditional yes** |
| Cross-Region automated backups, PITR restore | ~5 min | 1 h → many hours | RPO yes, **RTO no** |
| Manual snapshot copy on a schedule | = copy interval | hours | RPO maybe, **RTO no** |
| AWS Backup cross-Region copy | = backup frequency | hours | **RTO no** |
| Logical replication to a live standby | seconds | ~minutes (target already writable) | yes, but high operational cost |

### Where the 15 minutes actually goes

Assume a warm cross-Region read replica, already running, already correctly sized.

| Step | Time | Automatable? |
|---|---|---|
| Detect + human decision to fail over | 0–? | The decision is human. Excluded from RTO by definition in [[research-brief]] ("from decision-to-fail-over"). |
| `aws rds promote-read-replica` API call returns | seconds | Yes |
| RDS stops replication, reboots the instance, instance returns to `available` | **"several minutes or longer"** | Automatic, but you must poll |
| Postgres crash-recovery / redo of unreplayed WAL on the new primary | seconds–minutes, proportional to unreplayed WAL | Automatic |
| DNS cutover to the new writer endpoint | TTL-bound, see [[#Endpoint management at failover]] | Yes |
| Application connection pools notice and reconnect | 0 → 15 min, see [[#The connection pool trap]] | Only if configured for it |
| Standby compute (EKS etc.) scaled up and serving | out of scope here, see [[aws-eks]] | — |

Two of those rows are unbounded by default, and both are avoidable. The DNS and connection-pool rows are where most real failovers lose their RTO, not the database.

### How long does promotion actually take

**No public, reproducible benchmark was found.** This is a real gap, not a lazy search. AWS says only:

> "When you promote a read replica, RDS reboots the DB instance before making it available. The promotion process can take several minutes or longer to complete, depending on the size of the read replica."

and, in the step-by-step section:

> "The promotion process takes a few minutes to complete. When you promote a read replica, RDS stops replication and reboots the read replica."

AWS's own DR blog comparing backups / snapshots / read replicas ranks read replicas "Best" for RTO but publishes **no numbers at all** — only Good/Better/Best. The DR workshop page for RDS cross-Region replication likewise gives no timing.

**Action:** this must be measured in-house before the RTO is signed off. It is a 20-minute experiment: build the replica, promote it, record wall-clock from API call to `available` plus time-to-first-successful-write. Record the number per environment and per data volume, because it scales with size. Logged in [[#Open questions]].

### Promotion is irreversible — and the consequences are larger than they look

From the AWS docs, verbatim: "After you promote the read replica, it ceases to function as a read replica and becomes a standalone DB instance... **You can't use the DB instance as a replication target because it is no longer a read replica.**"

There is no `demote-db-instance`. Chew on what that means operationally:

1. **There is no "undo" on a false alarm.** If you promote because you think `eu-west-1` is gone, and `eu-west-1` comes back five minutes later with an intact, still-writable primary, you now have **two independent writable Postgres instances that have diverged**. Any write that landed on the old primary after the replica's last received WAL is now on a database that is no longer anybody's source of truth. Reconciling that is a manual data-forensics exercise, not a runbook step.
2. **This makes the failover decision a genuine one-way door**, and your runbook must treat it as such. The go/no-go criteria need to be written down *in advance*, agreed by someone who can be woken up, and biased towards waiting — because the cost of a wrong promote is worse than the cost of five more minutes of outage. See [[failover-runbook]].
3. **Failback is a rebuild, not a flip.** The old primary cannot become a replica of the new one. See [[#Failback]].
4. **RDS Postgres has no switchover equivalent.** RDS for *Oracle* has a Data Guard switchover (source becomes the replica, replica becomes the source — reversible, planned). RDS for PostgreSQL does not. Aurora Global Database **does** have a managed planned switchover, and this is one of the strongest arguments in [[rds-vs-aurora-decision]].
5. **A rehearsal costs you the replica.** You cannot test promotion non-destructively on the real standby. Every game day either (a) uses a throwaway replica built for the test, or (b) burns the real one and triggers a full reseed. Budget for (a).

### Post-promotion: what the new primary has and hasn't got

Because an RDS cross-Region replica is a **physical** replica, most of the scary stuff is fine:

| Thing | State after promotion | Why |
|---|---|---|
| **Sequences** | Correct. No `setval()` needed. | Sequence state is in the data files and travels in the WAL stream. (This is *only* true for physical replication — with logical replication you must fix them.) |
| **Extensions** | Present, same versions. | `CREATE EXTENSION` is DDL captured in WAL. |
| **Users / roles / passwords** | Present. | Cluster-wide catalogs are physical. |
| **Data** | Everything the replica had received and replayed. Anything still in flight is **lost**. | Async replication. This is the RPO. |
| **Replication slots** | **Absent / not carried.** | A promoted replica has no downstream slots for logical consumers unless you are on PG17+ with `sync_replication_slots` (RDS supports logical slot synchronisation from PG17). Anything consuming CDC off the old primary — Debezium, DMS, a data pipeline — breaks and needs re-bootstrapping from a new slot. |
| **Parameter group / option group** | **Retained** from the pre-promotion replica. | Docs: "The standalone DB instance retains the option group and the parameter group of the pre-promotion read replica." This is why a drifted standby parameter group is so dangerous — it silently becomes the production config. |
| **Automated backups / PITR window** | You set retention *at promotion time* (the console asks). The PITR history of the old primary does not follow. | Enable backups on the replica **before** the incident — AWS recommends "enable backups and complete at least one backup" pre-promotion, and a replica in `backing-up` status **cannot be promoted at all**. |
| **Multi-AZ** | Whatever the replica had. If the replica was single-AZ to save money, your new production primary is single-AZ. | Modifying to Multi-AZ post-promotion is a separate, slow operation. |
| **Read replicas of its own** | None. | Create them after. |

> [!danger] The `backing-up` interaction
> "Make sure that your read replica doesn't have the `backing-up` status. You can't promote a read replica when it is in this state." If you enable automated backups on the standby replica (you should) and its backup window happens to overlap the incident, **promotion is blocked until the backup finishes**. Set the standby's `backup_window` to a time you would never fail over in, and make sure it does not collide with the primary's. This is a one-line Terraform change that buys you an RTO you would otherwise lose at the worst possible moment.

## Warm standby shape

While the primary is healthy, the standby Region holds:

| Resource | State | Costs money? |
|---|---|---|
| `aws_db_instance` (the replica) | Running, read-only, streaming | **Yes — full instance-hour price, 24/7.** The dominant line item. |
| Storage (gp3/io1/io2) | Allocated, same size as source | Yes |
| Cross-Region data transfer | Continuous, proportional to WAL volume | Yes |
| DB subnet group | Exists, empty of cost | No |
| DB parameter group | Exists | No |
| Security group | Exists | No |
| KMS CMK in the standby Region | Exists | ~$1/month/key + requests ([[aws-kms]]) |
| Replicated automated backups (if enabled) | Accumulating | Yes — snapshot storage |
| Secrets Manager replica of the DB credential | Exists (already done per [[research-brief]]) | Small |

**Nothing here can be scaled to zero.** Unlike EKS node groups or Lambda, the RDS replica must run continuously to be warm. `aws rds stop-db-instance` explicitly does not work on read replicas or on instances that have read replicas. That is the cost floor of meeting RTO 15m on RDS, and it is why [[#Decisions to make]] presents the backups-only branch honestly rather than dismissing it.

**Sizing lever:** AWS *recommends* the replica be same-class-or-larger, but does not enforce it. A smaller replica is cheaper and will usually keep up with a read-light DR workload, at the cost of (a) more lag under write bursts and (b) a post-promotion primary that cannot carry production load. If you undersize, the runbook must include "modify instance class" — which is another reboot and blows the RTO. **Recommendation: size the standby replica the same as the primary.** The cost saving from undersizing is not worth spending it back at 3am. If cost pressure is real, take the backups-only branch instead of an undersized replica — an undersized replica is the worst of both worlds.

## Terraform implementation

Provider v5/v6. The shape below assumes a cookiecutter-templated monorepo where each environment instantiates a stack and the stack fans out to per-Region modules.

### Provider aliases

```hcl
# providers.tf — generated by cookiecutter per environment
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.70"
    }
  }
}

provider "aws" {
  alias  = "primary"
  region = var.primary_region   # eu-west-1 | us-east-1 | ca-central-1
  default_tags { tags = local.common_tags }
}

provider "aws" {
  alias  = "standby"
  region = var.standby_region   # eu-west-2 | us-west-2 | ca-west-1

  # REQUIRED when standby_region is an opt-in Region (ca-west-1).
  # See "The STS opt-in-Region trap" below.
  sts_region_endpoints = "regional"   # or set AWS_STS_REGIONAL_ENDPOINTS=regional in CI

  default_tags { tags = local.common_tags }
}
```

> The exact attribute name for regional STS endpoints has moved between provider major versions; `AWS_STS_REGIONAL_ENDPOINTS=regional` as an environment variable in CI is version-proof and is what we actually recommend. Pin it in the CI job, not in HCL.

### Module signature

One module, called twice — once per Region — with a `replicate_source_db_arn` that is `null` for the primary. This keeps a single code path and avoids a "primary module" and a "standby module" drifting apart, which is the failure mode we are trying to avoid.

```hcl
# modules/rds-postgres/variables.tf
variable "name"                    { type = string }
variable "environment"             { type = string }
variable "engine_version"          { type = string }   # e.g. "16.4"
variable "instance_class"          { type = string }
variable "allocated_storage"       { type = number, default = 100 }
variable "max_allocated_storage"   { type = number, default = 1000 }
variable "storage_type"            { type = string, default = "gp3" }
variable "multi_az"                { type = bool,   default = true }
variable "vpc_id"                  { type = string }
variable "subnet_ids"              { type = list(string) }
variable "ingress_security_group_ids" { type = list(string), default = [] }
variable "kms_key_arn"             { type = string }   # MUST be a key in THIS module's region
variable "parameter_group_family"  { type = string }   # e.g. "postgres16"
variable "db_parameters"           { type = map(string), default = {} }
variable "backup_retention_period" { type = number, default = 14 }
variable "backup_window"           { type = string, default = "03:00-04:00" }
variable "maintenance_window"      { type = string, default = "sun:05:00-sun:06:00" }
variable "deletion_protection"     { type = bool,   default = true }

# Null on the primary. Source instance ARN on the standby.
variable "replicate_source_db_arn" { type = string, default = null }

# Only meaningful on the primary: turn on cross-Region automated backup replication.
variable "backup_replication_destination" {
  description = "Region to replicate automated backups into. Null to disable."
  type        = string
  default     = null
}
variable "backup_replication_kms_key_arn" { type = string, default = null }
```

### The instance

```hcl
# modules/rds-postgres/main.tf
locals {
  is_replica = var.replicate_source_db_arn != null
}

resource "aws_db_subnet_group" "this" {
  name       = "${var.name}-${var.environment}"
  subnet_ids = var.subnet_ids
}

resource "aws_db_parameter_group" "this" {
  name        = "${var.name}-${var.environment}-${replace(var.parameter_group_family, ".", "")}"
  family      = var.parameter_group_family
  description = "Managed by Terraform — MUST be identical in primary and standby regions"

  dynamic "parameter" {
    for_each = var.db_parameters
    content {
      name         = parameter.key
      value        = parameter.value
      apply_method = "pending-reboot"
    }
  }

  lifecycle { create_before_destroy = true }
}

resource "aws_security_group" "this" {
  name_prefix = "${var.name}-${var.environment}-rds-"
  vpc_id      = var.vpc_id
  lifecycle { create_before_destroy = true }
}

resource "aws_vpc_security_group_ingress_rule" "from_app" {
  for_each                     = toset(var.ingress_security_group_ids)
  security_group_id            = aws_security_group.this.id
  referenced_security_group_id = each.value
  from_port                    = 5432
  to_port                      = 5432
  ip_protocol                  = "tcp"
}

resource "aws_db_instance" "this" {
  identifier = "${var.name}-${var.environment}"

  # --- replica-vs-primary fork -------------------------------------------
  # Cross-Region replica: replicate_source_db takes the source *ARN*.
  # (Same-Region replicas take the bare identifier; cross-Region needs the ARN.)
  replicate_source_db = var.replicate_source_db_arn

  # engine/version/credentials are inherited from the source on a replica and
  # MUST NOT be set, or the provider will try to modify them post-create.
  engine         = local.is_replica ? null : "postgres"
  engine_version = local.is_replica ? null : var.engine_version
  db_name        = local.is_replica ? null : replace(var.name, "-", "_")
  username       = local.is_replica ? null : "app_admin"
  manage_master_user_password = local.is_replica ? null : true
  master_user_secret_kms_key_id = local.is_replica ? null : var.kms_key_arn
  # -----------------------------------------------------------------------

  instance_class        = var.instance_class
  allocated_storage     = var.allocated_storage
  max_allocated_storage = var.max_allocated_storage
  storage_type          = var.storage_type

  # CRITICAL: on a cross-Region replica `storage_encrypted` is IGNORED by the
  # provider; `kms_key_id` is what does the work and it must name a key that
  # lives in THIS region. See [[aws-kms]].
  storage_encrypted = local.is_replica ? null : true
  kms_key_id        = var.kms_key_arn

  db_subnet_group_name   = aws_db_subnet_group.this.name
  parameter_group_name   = aws_db_parameter_group.this.name
  vpc_security_group_ids = [aws_security_group.this.id]

  multi_az = var.multi_az

  # Enable backups on the REPLICA too. Two reasons:
  #  1. AWS recommends at least one completed backup before promotion.
  #  2. It gives the standby its own PITR window the moment it is promoted.
  backup_retention_period = var.backup_retention_period
  backup_window           = var.backup_window
  maintenance_window      = var.maintenance_window

  performance_insights_enabled = true
  monitoring_interval          = 60
  monitoring_role_arn          = aws_iam_role.enhanced_monitoring.arn
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]

  auto_minor_version_upgrade = false   # see Gotchas: version drift across regions
  deletion_protection        = var.deletion_protection
  skip_final_snapshot        = false
  final_snapshot_identifier  = "${var.name}-${var.environment}-final-${formatdate("YYYYMMDDhhmmss", timestamp())}"
  apply_immediately          = false

  lifecycle {
    ignore_changes = [
      final_snapshot_identifier,
      # During an incident a human may promote out-of-band. Do NOT ignore
      # replicate_source_db here — you want the drift to be visible.
      # See "Terraform state when you promote out-of-band".
    ]
  }
}
```

### Cross-Region automated backups

The resource is `aws_db_instance_automated_backups_replication` (provider v4.9.0+), and it is declared **in the destination Region** pointing at the source ARN:

```hcl
# Declared in the STANDBY region's provider, referencing the PRIMARY's instance.
resource "aws_db_instance_automated_backups_replication" "this" {
  count = var.backup_replication_destination != null ? 1 : 0

  source_db_instance_arn = aws_db_instance.this.arn
  kms_key_id             = var.backup_replication_kms_key_arn  # key in the DESTINATION region
  retention_period       = var.backup_retention_period
}
```

Because the resource lives in the destination Region but references a source in another Region, in a two-Region stack it is cleanest to declare it **outside** the per-Region module, at the stack level, where both providers are in scope:

```hcl
# stacks/<env>/rds.tf
module "db_primary" {
  source    = "../../modules/rds-postgres"
  providers = { aws = aws.primary }

  name           = var.service_name
  environment    = var.environment
  engine_version = var.pg_version
  instance_class = var.db_instance_class
  vpc_id         = module.vpc_primary.vpc_id
  subnet_ids     = module.vpc_primary.database_subnet_ids
  kms_key_arn    = module.kms_primary.rds_key_arn
  parameter_group_family = "postgres${split(".", var.pg_version)[0]}"
  db_parameters = local.pg_parameters      # <-- SAME map for both regions
}

module "db_standby" {
  source    = "../../modules/rds-postgres"
  providers = { aws = aws.standby }
  count     = var.enable_standby_replica ? 1 : 0

  name           = var.service_name
  environment    = var.environment
  instance_class = var.db_standby_instance_class
  vpc_id         = module.vpc_standby.vpc_id
  subnet_ids     = module.vpc_standby.database_subnet_ids
  kms_key_arn    = module.kms_standby.rds_key_arn        # DESTINATION-region key
  parameter_group_family = "postgres${split(".", var.pg_version)[0]}"
  db_parameters = local.pg_parameters      # <-- SAME map, single source of truth

  replicate_source_db_arn = module.db_primary.instance_arn
  multi_az                = var.standby_multi_az
  deletion_protection     = true
}

# Cheap DR floor — keep this ON even when enable_standby_replica is true.
resource "aws_db_instance_automated_backups_replication" "standby" {
  provider = aws.standby

  source_db_instance_arn = module.db_primary.instance_arn
  kms_key_id             = module.kms_standby.rds_key_arn
  retention_period       = var.backup_retention_period
}
```

The single `local.pg_parameters` map passed to both modules is the whole trick for parameter-group drift. Do not let the standby have its own list.

### Variable surface to expose in cookiecutter

| Variable | Why it is a knob |
|---|---|
| `enable_standby_replica` | Lets non-prod environments skip the expensive warm replica entirely while keeping backup replication. This is where most of the cost saving lives. |
| `db_standby_instance_class` | Separate from primary so you *can* undersize, but defaults to the primary's class. |
| `standby_multi_az` | Standby Multi-AZ doubles standby cost; defaults to `false` in non-prod, `true` in prod. |
| `backup_retention_period` | Drives both local and replicated backup retention. |
| `primary_region` / `standby_region` | The pair. Defaulted per environment from the [[region-pair-selection]] table. |

### The STS opt-in-Region trap

Relevant to the **CA pair specifically**. The RDS docs state:

> "Session tokens from the global AWS Security Token Service (AWS STS) endpoint are valid only in AWS Regions that are enabled by default (commercial Regions). If you use credentials from the `assumeRole` API operation in AWS STS, use the regional endpoint if the source Region is an opt-in Region. Otherwise, the request fails... your credentials must be valid in both Regions."

`ca-west-1` is an opt-in Region. Terraform in CI almost always uses `AssumeRole` (OIDC from GitHub Actions/GitLab, or an execution role). If those credentials come from the **global** STS endpoint, cross-Region calls involving `ca-west-1` can fail with an opaque auth error that looks nothing like "you used the wrong STS endpoint".

Fixes, in order of laziness:

1. Set `AWS_STS_REGIONAL_ENDPOINTS=regional` in the CI job environment. One line. (This is the default in modern SDKs but *not* in every runner image or older CLI.)
2. Set the account's STS **global endpoint token version** to "Valid in all AWS Regions" (v2 tokens). Account-wide setting, fixes it everywhere.
3. Same applies to the `--pre-signed-url` path if you ever drop to the CLI during an incident.

Also note: the requester's IAM policy must allow `rds:CreateDBInstanceReadReplica` on **both** the source and the replica ARNs, and must **not** deny `aws:ViaAWSService` — RDS calls the source Region on your behalf. An `aws:RequestedRegion` condition scoped to a single Region will silently break replica creation. This bites estates with tight SCPs.

### Terraform state when you promote out-of-band

This will happen. At 3am somebody runs `aws rds promote-read-replica` (or clicks Promote) because that is faster than a Terraform apply, and it is the right call. Here is what you are left with.

**What the provider does on the next plan.** The provider reads the instance and finds `ReadReplicaSourceDBInstanceIdentifier` is now empty, while the config still has `replicate_source_db` set. `replicate_source_db` is **not** `ForceNew` in the AWS provider — the docs state explicitly:

> "Removing the `replicate_source_db` attribute from an existing RDS Replicate database managed by Terraform will promote the database to a fully standalone database."

So the attribute is *updatable*, which is both good and bad:

- **Good:** you can promote *through Terraform* by flipping `replicate_source_db_arn` to `null` and applying. That is the clean path and it keeps state correct.
- **Bad:** if state still says "replica" and reality says "standalone", the provider may attempt to (re)establish replication, and RDS answers with `cannot elect new source database for replication` (the error in [hashicorp/terraform-provider-aws#16054](https://github.com/hashicorp/terraform-provider-aws/issues/16054), which was closed as stale/not-planned). At that point people reach for `taint`, and **tainting a promoted production database is a catastrophe** — it destroys your only surviving copy of the data.

**Runbook rule, write it in bold in [[failover-runbook]]:** after an out-of-band promotion, **nobody runs `terraform apply` against that stack until the config has been reconciled**. Specifically:

1. Immediately after promoting, disable the CI pipeline for that stack (protected branch / manual gate).
2. Set `enable_standby_replica`-equivalent inputs so that the promoted instance is now the *primary* in config: `replicate_source_db_arn = null`, and add back the attributes that were `null` on a replica (`backup_retention_period` must be `> 0`, which it already is if you followed the module above).
3. `terraform plan` and read every line. Expect: `replicate_source_db` removal (in-place), possibly `backup_retention_period`, possibly `engine_version` now appearing. **Confirm there is no `-/+ destroy and then create replacement`** before applying. If there is, stop and work out why.
4. The old primary's `aws_db_instance` resource in state now refers to something that may be gone or may be a zombie. `terraform state rm` it rather than letting a plan try to delete it — you may want the zombie for forensics.

This is the strongest practical argument in [[terraform-repo-structure]] for keeping the two Regions in **separate state files** with the role (primary vs standby) as an input rather than baked into resource addresses. When the roles swap, you want to change a variable, not move resources between states.

### KMS and the destination-region key

The rule, stated by AWS three different ways:

> "To copy an encrypted snapshot from one AWS Region to another, you must specify the KMS key in the destination AWS Region. **This is because KMS keys are specific to the AWS Region that they are created in.**"

> "If the primary DB instance and read replica are in different AWS Regions, you encrypt the read replica using the KMS key for that AWS Region."

> Terraform: `storage_encrypted` — "if you are creating a cross-region read replica this field is ignored and you should instead declare `kms_key_id` with a valid ARN."

**Why it is mandatory, mechanically.** Creating the replica goes through a cross-Region encrypted snapshot copy (step 3 of the four-phase process). A single-Region KMS key's key material never leaves its Region and the KMS endpoint in the destination Region cannot call `Decrypt` against a key ARN in another Region. So RDS uses **envelope encryption**: it decrypts the snapshot's data key in the source Region, re-wraps it under the destination key, and the destination Region's storage is encrypted under the destination key from then on. Without a destination key there is nothing to re-wrap under and the operation is rejected.

**Consequences worth internalising:**

- **The primary and the standby are encrypted under different keys, permanently.** Two keys, two key policies, two sets of grants, two rotation schedules. Both must allow the RDS service principal and both must allow whatever CI principal creates the instance ("During the creation of a DB instance, Amazon RDS checks if the calling principal has access to the KMS key and generates a grant from the KMS key that it uses for the entire lifetime of the DB instance").
- **You cannot change a DB instance's KMS key after creation.** "Once you have created an encrypted DB instance, you can't change the KMS key used by that DB instance." Getting the standby key wrong means destroying and rebuilding the replica — which means re-seeding from a fresh snapshot copy, which is hours.
- **Multi-Region KMS keys (MRKs) do not remove the requirement**, they just make it tidier. An MRK replica key in the standby Region is *still a different key ARN* and you still pass it as `kms_key_id`. What an MRK buys you is a shared key *policy* and shared key material, so the two Regions' keys cannot drift and a ciphertext produced in one is decryptable in the other. For RDS specifically that last property matters less than it does for, say, Secrets Manager or S3 SSE-KMS — but the policy-drift protection alone is worth it. See [[aws-kms]] for the MRK-vs-two-independent-CMKs decision; that decision belongs in that note, not this one.
- **Disabling a KMS key takes the database down**, and the "recoverable" grace state does **not** apply to read replicas or to instances that have read replicas: "This recoverable state is not applicable to instances that can't stop, such as read replicas and instances with read replicas." A fat-fingered key disable in the standby Region takes out your DR capability with no 7-day grace period. Guard `kms:DisableKey` with an SCP.

## Migration path from single-region

Today: one live, encrypted, Multi-AZ RDS Postgres instance per Region, managed by Terraform. Target: the same plus a warm cross-Region replica and replicated automated backups. Nothing below requires downtime, and nothing below forces a replacement of the existing instance — **if** you go in this order.

**Step 0 — pre-flight, do this first and do it for real.**
- `aws rds describe-orderable-db-instance-options --region <standby> --engine postgres --engine-version <current>` and confirm your instance class exists. **Especially for `ca-west-1`.**
- `aws rds describe-db-engine-versions --region <standby> --engine postgres` and confirm your exact minor version exists.
- Confirm the source instance is **encrypted**. "To create an encrypted read replica in a different AWS Region from the source DB instance, the source DB instance must be encrypted." If your primary is unencrypted, you have a bigger problem: you cannot encrypt in place. The path is snapshot → encrypted copy → restore → cutover, which *is* downtime. Find this out now, not in month three.
- Confirm the source has `backup_retention_period > 0`. A replica source needs automated backups on.
- Check the source instance's `storage_type` and IOPS; plan to match.

**Step 1 — standby networking and KMS.** VPC, subnets, DB subnet group, security groups, and the standby-Region CMK. No RDS resources yet. See [[aws-vpc-networking]] and [[aws-kms]]. This step is free-ish and reversible.

**Step 2 — turn on cross-Region automated backup replication.** One resource, `aws_db_instance_automated_backups_replication`. This is non-disruptive to the source, gives you an immediate RPO improvement, and is the cheap insurance that stays switched on forever regardless of what you decide about the replica. **Do this before the replica.** If the replica project stalls, you still walked away with cross-Region DR at ~5 minute RPO.

**Step 3 — extract the parameter group into a shared local.** Before creating any standby instance, refactor so both Regions read the same `local.pg_parameters` map. If today's primary uses a hand-crafted parameter group created outside Terraform, import it first. Drift that exists before you mirror is drift you will mirror.

**Step 4 — create the replica.** `terraform apply` with `enable_standby_replica = true`. Watch for:
- The source goes to `modifying` briefly. This is normal and non-disruptive, but it will trip a naive "instance status != available" alarm. Silence that alarm first.
- Creation "can take hours". Terraform will sit in `Still creating...`. Raise the `timeouts { create = ... }` block on the resource if your CI has a job timeout. Default provider create timeout for `aws_db_instance` is generous but your *pipeline's* timeout may not be.
- Data transfer charges begin (initial snapshot copy, then ongoing WAL).

**Step 5 — verify.** `ReplicaLag` settling to its baseline. Connect to the replica endpoint and confirm it is in recovery (`SELECT pg_is_in_recovery();` → `t`). Row counts on a couple of large tables. Extension list matches.

**Step 6 — endpoint indirection.** Introduce the Route 53 CNAME layer *now*, while nothing is on fire, and repoint the application at the CNAME rather than the raw RDS endpoint. See [[#Endpoint management at failover]]. Doing this during an incident is how you lose the RTO.

**Step 7 — game day.** Build a *throwaway* second replica, promote it, time it, throw it away. Record the number.

**Things that force replacement — watch for these in plan:**

| Change | Effect |
|---|---|
| Adding `replicate_source_db` to an *existing* standalone instance | Not supported. You cannot turn a standalone instance into a replica. The standby must be *created* as a replica. |
| Changing `kms_key_id` on an existing instance | Replacement. And you cannot change a DB's key at all — plan will try, API will refuse or destroy. |
| Changing `storage_encrypted` | Replacement. |
| Changing `db_subnet_group_name` to a group in a different VPC | Replacement. |
| `identifier` change | Replacement. Cookiecutter naming changes are dangerous here. |
| `restore_to_point_in_time`, `backup_target`, `nchar_character_set_name` | Documented as ForceNew. |

## Failover procedure

Assumes a running cross-Region read replica, DNS indirection in place, and an out-of-band (CLI) promotion because that is faster and more reliable than a Terraform apply during an incident.

**Human decision gate — not automatable.** Promotion is a one-way door (see above). Someone with authority decides. The runbook should state the criteria, e.g. "primary Region RDS control plane unreachable AND primary endpoint unreachable from two independent probes for > N minutes".

```bash
# 0. Freeze the pipeline for this stack. Do this FIRST.
#    (A `terraform apply` mid-failover is a worse outage than the one you have.)

# 1. Record the replica's last-received WAL position, for the post-mortem
#    and for any data-loss reconciliation.
psql -h $STANDBY_ENDPOINT -c "SELECT pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn(), now();"

# 2. Confirm the replica is not mid-backup — promotion is BLOCKED if it is.
aws rds describe-db-instances --region $STANDBY_REGION \
  --db-instance-identifier $STANDBY_ID --query 'DBInstances[0].DBInstanceStatus'

# 3. Promote. Irreversible from here.
aws rds promote-read-replica --region $STANDBY_REGION \
  --db-instance-identifier $STANDBY_ID \
  --backup-retention-period 14 --preferred-backup-window 03:00-04:00

# 4. Wait for available. This includes a reboot. TIME THIS.
time aws rds wait db-instance-available --region $STANDBY_REGION \
  --db-instance-identifier $STANDBY_ID

# 5. Confirm it is genuinely writable.
psql -h $STANDBY_ENDPOINT -c "SELECT pg_is_in_recovery();"   # expect f
psql -h $STANDBY_ENDPOINT -c "CREATE TABLE _failover_probe(t timestamptz); DROP TABLE _failover_probe;"

# 6. Repoint DNS. (Should be a single Route 53 UPSERT — see below.)
aws route53 change-resource-record-sets --hosted-zone-id $ZONE_ID \
  --change-batch file://cutover.json

# 7. Bounce application connection pools. Do not wait for them to notice.
kubectl -n $NS rollout restart deployment/api    # or equivalent
```

Steps 3–7 are scriptable end-to-end. Step 0 and the decision are not. See [[failover-orchestration]] for where this script should live (an SSM Automation document or a Step Functions state machine in a *third* place, not in either Region's own account/stack — a runbook that lives in the dead Region is not a runbook).

### Endpoint management at failover

The application must not hold the raw RDS endpoint (`db-prod.abc123.eu-west-1.rds.amazonaws.com`). At failover that hostname either disappears or, worse, keeps resolving to a database nobody is writing to.

**Option A — Route 53 CNAME (recommended).**

```hcl
resource "aws_route53_record" "db_writer" {
  zone_id = var.private_zone_id
  name    = "db.${var.environment}.internal.example.com"
  type    = "CNAME"
  ttl     = 5                       # see TTL trap below
  records = [module.db_primary.endpoint_address]
}
```

Application connects to `db.prod.internal.example.com`. Failover is one `UPSERT`. Keep the record in a **private hosted zone associated with both VPCs**, or the standby's workloads can't resolve it. Note a private hosted zone is a global Route 53 resource but its VPC associations are per-VPC — associating a zone with a VPC in another Region is supported and is the piece people forget. See [[aws-route53]].

**The DNS TTL trap.** A CNAME with a 300-second TTL adds up to 5 minutes to your RTO — a third of the budget — and that is the *best* case. Set the TTL to **5 seconds** and set it now, months before you need it, because lowering a TTL only takes effect after the old TTL expires everywhere. Lowering it during an incident does nothing for the first 300 seconds.

And TTL is a lie anyway:
- The JVM historically caches DNS **forever** (`networkaddress.cache.ttl = -1` under a security manager). Set `networkaddress.cache.ttl=5` explicitly. This is still, in 2026, the single most common cause of a failover where "DNS changed but the app didn't".
- Node's default resolver does not honour TTL for `dns.lookup()`; it calls `getaddrinfo` and relies on the OS cache.
- `nscd`/`systemd-resolved` on the node, and CoreDNS in the cluster, each add their own caching layer. CoreDNS's default `cache 30` is on top of your record TTL.
- Go's `net.Resolver` does not cache, which for once is what you want.

**The connection pool trap, which is worse than DNS.** A connection pool that already holds 50 established TCP sockets to the old primary's IP **will never do a DNS lookup again**. Repointing DNS changes nothing for those connections. They will either:
- hang until the OS TCP retransmit timeout (on Linux, `tcp_retries2 = 15` ≈ **~15 minutes** by default) if the old Region is black-holing packets rather than sending RSTs, or
- error immediately if something sends a RST.

The black-hole case is the realistic one in a Region failure, and ~15 minutes of hung connections *is your entire RTO*, spent doing nothing. Mitigations, in order of effectiveness:

1. **Restart the application pods at failover.** Crude, instant, always works. Put it in the runbook as step 7. This is the lazy answer and it is the right one.
2. Set aggressive JDBC/pgx socket timeouts (`socketTimeout`, `tcp_user_timeout`) so hung connections die in seconds rather than minutes. `TCP_USER_TIMEOUT` is the one that actually fixes the black-hole case; `socketTimeout` only helps if a query is in flight.
3. Pool-level `maxLifetime` (HikariCP defaults to 30 min — too long) and connection validation on borrow.

**Option B — RDS Proxy.** Gives the app a stable proxy endpoint. But RDS Proxy is **regional** and must be pre-provisioned in the standby Region, pointed at the standby instance, and its target registration has to survive the promotion. It does not solve cross-Region failover on its own — you still need DNS or config to move traffic between the two proxies. It *does* help with the connection-pool problem in general. Worth it if you already have pool exhaustion issues; not worth adding solely for DR.

**Option C — application-side service discovery / config reload.** The app reads the writer endpoint from SSM Parameter Store or a Secrets Manager secret and re-reads on failure. Since [[research-brief]] says Secrets Manager replication is already done, the credential secret is already in both Regions — putting the host in the same secret is nearly free. Downside: you have now built a config-propagation system and its cache TTL is the new DNS TTL. **Recommendation: Route 53 CNAME with a 5s TTL, plus mandatory pod restart at failover.** Do not build option C.

## Failback

This is the section people skip and then discover at the worst time. **Failback from a promoted RDS replica is a full rebuild in the reverse direction.**

Because promotion is irreversible and there is no demote, the old primary (assume it comes back healthy and intact) **cannot** be attached as a replica of the new primary. Your only options:

**Path 1 — Reverse the replica (recommended).**
1. Delete the old primary in the original Region entirely. It holds stale, diverged data; keep a final snapshot for forensics and then get rid of it, because leaving a writable zombie Postgres with the old DNS name around is how you get split-brain twice.
2. Create a **new** cross-Region read replica in the *original* Region, sourced from the now-primary in the standby Region. This is the same four-phase snapshot-copy-and-seed process as the original build: **"this process can take hours to complete."** For a multi-TB database over an inter-Region link, plan on hours, not minutes. The whole database moves across the Region boundary again, at cross-Region transfer rates.
3. Wait for lag to settle.
4. Schedule a **planned** failback: quiesce writes, confirm zero lag, promote the new replica in the original Region, repoint DNS, restart pools.
5. Rebuild the replica in the standby direction again. **Another full seed, another few hours.**

So a round trip costs you **two full reseeds**. And note that step 4 is still a promotion — still irreversible, still a reboot, still ~5–15 minutes of write unavailability. **There is no zero-downtime failback for RDS Postgres.** Even the planned, calm, daytime version of it takes an outage window.

**Path 2 — Logical replication for the failback leg.** Set up native logical replication from the new primary back to a freshly-restored instance in the original Region. Advantages: the target is writable during replication (so you can pre-warm), you can do it in stages, and you avoid one of the reboots. Disadvantages: sequences need fixing (`setval()` on every sequence), DDL doesn't replicate, and you've added a hand-operated replication system to an already stressful week. Only worth it if the database is large enough that the reseed window is genuinely intolerable.

**Path 3 — Don't fail back.** Genuinely consider this. If the two Regions are symmetric (same instance classes available, same data-residency posture, same latency to users), then after a failover the *standby becomes the primary permanently* and you build the new standby in the old primary Region. You still pay one reseed instead of two, and you skip the second promotion outage entirely.

This only works if the Terraform is written so that "which Region is primary" is a **variable**, not a structural property of the code. It is a strong argument for the module shape above (one module, `replicate_source_db_arn` as an input) and against having `modules/rds-primary/` and `modules/rds-standby/`. See [[terraform-repo-structure]].

**Recommendation: Path 3 for the CA and EU pairs where the Regions are broadly symmetric; Path 1 for the US pair if there is a latency or cost reason to live in `us-east-1`.** Whichever you pick, write it down before the incident — "do we fail back?" is not a question to answer at 4am.

## Parameter, option, subnet and security groups

All four are regional. All four must be mirrored. Each has its own way of ruining a failover.

| Object | Failure mode |
|---|---|
| **DB parameter group** | The docs say "In most cases, the read replica uses the **default** DB parameter group" and lists only Db2 (mandatory) and MySQL/Oracle (optional) as engines where you can pass `--db-parameter-group-name` to `create-db-instance-read-replica` — **PostgreSQL is not in that list.** Terraform sets `parameter_group_name` on the resource regardless; whether RDS honours it at create time or the provider applies it as a subsequent `ModifyDBInstance` is worth verifying in your account (see [[#Open questions]]). Either way: **after creating the replica, assert that its parameter group is yours and not `default.postgres16`.** A standby running default `shared_buffers`, default `work_mem`, default `max_connections` will fall over the moment it takes production load — and it will do so *after* you have irreversibly promoted it. |
| **Option group** | N/A for PostgreSQL in practice (option groups matter for Oracle/SQL Server). The promoted instance "retains the option group... of the pre-promotion read replica", so if you ever do need one, it must be mirrored. |
| **DB subnet group** | Must exist in the standby VPC with subnets in ≥2 AZs. The docs note that a `--db-subnet-group-name` for a cross-Region replica "must specify a DB subnet group from the same VPC" (i.e. the destination VPC). AZ count differs between Regions — `ca-west-1` has **3** AZs; check your standby has enough for Multi-AZ. |
| **Security group** | The docs state the cross-Region read replica "uses the **default** security group" unless you pass one. Terraform does pass `vpc_security_group_ids`, but verify. A replica in the default SG is either unreachable (default SG allows only intra-SG traffic) or, if someone has "fixed" the default SG, over-exposed. |

**The drift problem is the real one.** Two parameter groups, maintained by hand, in two Regions, for two years. Someone bumps `max_connections` in production during an incident, doesn't mirror it, and nobody notices for eighteen months — until the failover. Mitigations:

1. **One `local.pg_parameters` map, consumed by both module calls.** As in the Terraform above. This makes drift impossible *through Terraform*.
2. **Forbid console/CLI parameter changes.** SCP-deny `rds:ModifyDBParameterGroup` to everyone except the CI role. This is the only thing that actually works.
3. **A scheduled diff.** A trivial Lambda or CI job that calls `describe-db-parameters` in both Regions and alarms on any difference. Cheap, and it catches the case where someone breaks rule 2.
4. **Pin `auto_minor_version_upgrade = false`** on both. Otherwise the two Regions' maintenance windows independently drift the engine version, and physical replication across differing minor versions is at best unsupported. (You then owe yourself a deliberate, ordered upgrade process: **upgrade the replica first, then the primary** — a replica may be on a *higher* minor version than its source but generally not a lower one.)

## Gotchas

1. **Postgres does not auto-promote when the source is deleted.** "For PostgreSQL DB instances, when the source DB instance for a cross-Region read replica is deleted, the replication status of the read replica is set to `terminated`. The read replica isn't promoted." Every other RDS engine auto-promotes. If your runbook was written against a MySQL estate, it is wrong.
2. **A replication slot on the source can fill the source's disk.** Cross-Region replicas use slots. If the replica is unreachable for long enough, WAL piles up on the *primary* until `FreeStorageSpace` hits zero and the **primary** goes down. Your DR mechanism took out production. Alarm on `OldestReplicationSlotLag` and `TransactionLogsDiskUsage` on the primary, not just on the replica. AWS's documented fix once you're in the hole is "drop the existing cross-Region replica and create a new one" — i.e. lose your DR capability for several hours.
3. **`backing-up` blocks promotion.** Covered above. Set non-overlapping backup windows.
4. **`storage_encrypted` is silently ignored on a cross-Region replica.** If you think you've set encryption and haven't set `kms_key_id`, you will find out at apply time — or worse, get an unencrypted replica of an... no, actually, you can't: "You can't have an encrypted read replica of an unencrypted DB instance or an unencrypted read replica of an encrypted DB instance." So it fails loudly. Fine. But the Terraform reads as if it's doing something it isn't.
5. **You can't change a DB instance's KMS key, ever.** Get the standby key right first time or rebuild.
6. **Disabling the standby KMS key kills the replica with no grace period**, because the `inaccessible-encryption-credentials-recoverable` state doesn't apply to read replicas.
7. **The STS opt-in-Region trap for `ca-west-1`.** Covered above. This one produces an error message that does not mention STS.
8. **IAM conditions that break replica creation.** `aws:RequestedRegion` scoped to one Region, or a deny on `aws:ViaAWSService`, or `aws:SourceVpc`/`aws:SourceVpce` conditions. All documented by AWS as causes of failure. Mature estates with SCPs hit this.
9. **The source goes to `modifying` when you first attach a cross-Region replica.** Non-disruptive, but it will page you if you alarm on instance status.
10. **Replica creation "can take hours".** Set `timeouts { create = "4h" }` and raise your CI job timeout, or your pipeline dies mid-create and leaves a half-built replica that Terraform has no state for.
11. **Instance class availability differs by Region.** `ca-west-1` launched with no 7th-gen families. Verify with `describe-orderable-db-instance-options`.
12. **Engine minor version availability differs by Region.** A version that exists in `ca-central-1` may not exist in `ca-west-1` on the same day.
13. **Cross-Region data transfer is billed on every write.** WAL volume, not row count. A write-heavy workload with lots of index churn transfers far more than people estimate. Model it from actual `TransactionLogsGeneration` / WAL bytes, not from table size.
14. **Restored instances get *default* parameter and option groups** unless you explicitly pass yours. Relevant to the backups-only branch: your restore script must pass `--db-parameter-group-name`.
15. **Lazy loading after a restore.** A PITR-restored instance is "fully operational" but slow until blocks are faulted in from S3. If you take the backups branch, your RTO includes not just "instance available" but "instance performing acceptably", and those are different times.
16. **Baseline `ReplicaLag` of up to 5 minutes with no workload** will scare whoever builds the dashboard. Document it so nobody chases it.
17. **`rds.logical_replication = 1` needs a reboot.** If you might ever want logical replication (for a major upgrade or an Aurora move), turn it on during a *planned* maintenance window now rather than needing a reboot later. It costs a small amount of extra WAL. Cheap insurance.
18. **Terraform `taint` on a promoted production database destroys your data.** Say it out loud in the runbook.

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| **DR mechanism** | Warm cross-Region read replica: RPO seconds, RTO ~5–15 min, costs a full second instance 24/7 | Cross-Region automated backups only: RPO ~5 min, RTO hours, costs ~storage | **Both.** Backups are cheap insurance against logical corruption (which a replica happily replicates) and are the fallback if the replica is broken. The replica is what buys RTO. Prod gets both; non-prod gets backups only. |
| **Standby instance class** | Same as primary | Smaller, to save money | **Same as primary.** An undersized standby lags more and cannot carry load after promotion, and resizing costs another reboot inside the RTO. If cost is the driver, drop to backups-only rather than undersizing. |
| **Standby Multi-AZ** | Multi-AZ replica (2x standby cost) | Single-AZ replica | **Single-AZ in the standby while it is a standby**, with "modify to Multi-AZ" as a *post*-failover, out-of-RTO step. The standby's job is to survive the primary Region dying, not to survive an AZ dying while idle. Revisit if [[compliance]] mandates it. |
| **Promotion trigger** | Fully automated on health-check failure | Human-gated, scripted | **Human-gated.** Promotion is irreversible and a false positive costs you a diverged database. Automate everything *after* the decision. |
| **Endpoint discovery** | Route 53 CNAME, 5s TTL | RDS Proxy / service discovery / config store | **Route 53 CNAME with a 5s TTL**, plus mandatory app restart at failover. Simplest thing that works; the pool restart is what actually delivers the cutover, not the DNS. |
| **Failback policy** | Always fail back to the original primary Region | Treat the pair as symmetric; the standby becomes the new primary permanently | **Symmetric (don't fail back)** for EU and CA. One reseed instead of two, one outage instead of two. Requires the Terraform to treat "primary" as a variable. |
| **Parameter group management** | Separate groups per Region, reviewed | One shared `locals` map consumed by both | **Shared map + SCP-deny on manual `ModifyDBParameterGroup` + a scheduled diff job.** |
| **RDS vs Aurora** | Stay on RDS Postgres | Move to Aurora Global Database | See [[rds-vs-aurora-decision]] — this is the big one and it deserves its own note. |

## Cost

Rough shape, not quoted figures — check the [RDS pricing page](https://aws.amazon.com/rds/pricing/) for current numbers, and note **`ca-west-1` and `eu-west-2` are more expensive per instance-hour than `ca-central-1` and `eu-west-1`**, so "double the database bill" understates it slightly.

| Line | Warm replica branch | Backups-only branch |
|---|---|---|
| Standby instance-hours | ~100% of primary instance cost (more if the standby Region is pricier) | **$0** |
| Standby storage | ~100% of primary storage cost | $0 (snapshot storage only) |
| Standby Multi-AZ | +100% again if enabled | n/a |
| Replicated snapshot storage | Optional but recommended | Snapshot storage, roughly proportional to DB size × retention |
| Cross-Region data transfer | Initial full copy, then continuous WAL. Proportional to write volume. | Initial full copy, then incremental snapshots + 5-minutely logs. **Usually less than WAL streaming.** |
| KMS | ~$1/key/month + request charges | Same |
| Performance Insights / enhanced monitoring on standby | Small but non-zero | $0 |

**Levers, in order of size:**

1. **`enable_standby_replica = false` in non-production.** Dev and staging do not need a 15-minute RTO. This is most of the saving and it costs nothing but a variable.
2. **Reserved Instances / Savings Plans on the standby.** The standby runs 24/7 forever — it is the most reservable workload you have. (Check RI availability in `ca-west-1` specifically; newer Regions have historically had thinner RI coverage.)
3. **Single-AZ standby.** Halves the standby instance cost.
4. **gp3 over io1/io2 on the standby** if the primary uses provisioned IOPS purely for production write throughput the standby doesn't see — but note this increases lag risk under write bursts. Measure before doing this.
5. **Backups-only for any environment where RTO can be relaxed.**

## Open questions

1. **How long does promotion actually take on our data volume?** No public number exists. Must be measured. Blocks sign-off on the 15-minute RTO.
2. **Is the production primary encrypted?** If not, mirroring requires a downtime migration and the whole plan changes. Check today.
3. **Does RDS honour a custom parameter group at cross-Region replica *create* time for PostgreSQL?** The docs list only Db2/MySQL/Oracle. Verify empirically what the Terraform provider produces, and add a post-create assertion either way.
4. **What instance classes and engine versions are actually orderable in `ca-west-1` today?** Run the two `describe-*` commands. This is the one finding that could change the CA pair's design.
5. **Is `ca-west-1` already opted into in every account that needs it?** Opt-in is per-account and needs doing in the management account first.
6. **What is the actual WAL generation rate per environment?** Drives the cross-Region transfer bill and the lag profile. `TransactionLogsGeneration` in CloudWatch.
7. **Does anything consume CDC from the primary** (Debezium, DMS, a warehouse pipeline)? Those replication slots do not survive promotion pre-PG17 and their re-bootstrap is not currently in anyone's runbook.
8. **Does [[data-residency]] permit Canadian data in Calgary and Irish data in London?** Assumed yes (same country / same-ish jurisdiction) but it must be confirmed in writing before the replica ships a single byte.
9. **Which team owns the go/no-go on an irreversible promotion, and are they on a rota?**

## Sources

- [Creating a read replica in a different AWS Region — RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.XRgn.html) — the four-phase creation process, "can take hours", the five-replica ACL limit, the PostgreSQL-doesn't-auto-promote-on-source-delete rule, the default-parameter-group and default-security-group behaviour, the IAM conditions that break replica creation, and the STS opt-in-Region warning.
- [Promoting a read replica to be a standalone DB instance — RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.Promote.html) — "several minutes or longer", the reboot, the `backing-up` block, retention of parameter/option group, and the explicit statement that a promoted instance can no longer be a replication target.
- [Best practices for Amazon RDS for PostgreSQL cross-Region read replicas — AWS Database Blog](https://aws.amazon.com/blogs/database/best-practices-for-amazon-rds-for-postgresql-cross-region-read-replicas/) — physical streaming with replication slots, the "up to 5 minutes of lag with no workload" baseline, the lag drivers, the CloudWatch metrics to watch, and the "drop and recreate the replica" remedy for slot-driven storage exhaustion.
- [Encrypting Amazon RDS resources — RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html) — "KMS keys are specific to the AWS Region that they are created in", the cross-Region destination-key requirement, "you can't change the KMS key used by that DB instance", envelope encryption during snapshot copy, and the `inaccessible-encryption-credentials-recoverable` behaviour (and its non-applicability to read replicas).
- [Replicating automated backups to another AWS Region — RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReplicateBackups.html) — the full source→destination Region matrix (including `ca-central-1 ↔ ca-west-1`), the Multi-AZ DB cluster exclusion, and the 20-per-account quota.
- [Restoring a DB instance to a specified time — RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PIT.html) — "RDS uploads transaction logs for DB instances to Amazon S3 every five minutes", post-restore lazy loading from S3, `StorageOperationStatus`, and restored instances defaulting to the default parameter group.
- [Supported Regions and DB engines for cross-Region read replicas — RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RDS_Fea_Regions_DB-eng.Feature.CrossRegionReadReplicas.html) — cross-Region read replicas available in **all** Regions for PostgreSQL 10–18, which clears all three pairs.
- [The AWS Canada West (Calgary) Region is now available — AWS News Blog](https://aws.amazon.com/blogs/aws/the-aws-canada-west-calgary-region-is-now-available/) — three AZs, ~70 services at launch including Aurora PostgreSQL and RDS, and the launch EC2 instance family list with no 7th-generation families.
- [terraform-provider-aws `aws_db_instance` docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_instance) — `replicate_source_db` takes the ARN cross-Region; `storage_encrypted` "is ignored" for cross-Region replicas; "Removing the `replicate_source_db` attribute... will promote the database to a fully standalone database".
- [hashicorp/terraform-provider-aws#16054](https://github.com/hashicorp/terraform-provider-aws/issues/16054) — the `cannot elect new source database for replication` error and the `taint` workaround; closed as stale/not-planned, so this is unfixed provider behaviour you must design around.
- [Implementing a disaster recovery strategy with Amazon RDS — AWS Database Blog](https://aws.amazon.com/blogs/database/implementing-a-disaster-recovery-strategy-with-amazon-rds/) — AWS's own RTO/RPO comparison of backups vs snapshots vs read replicas, notable for publishing only Good/Better/Best and **no numbers**, which is why [[#Open questions]] item 1 exists.
- [Replicating AWS RDS automated backups to a different Region — Xebia](https://xebia.com/blog/cross-region-automated-rds-backups/) — third-party write-up of `aws_db_instance_automated_backups_replication` (provider v4.9.0+) with the destination-Region KMS key requirement.
- No public benchmark of RDS for PostgreSQL cross-Region replica **promotion duration** was found. Searched AWS docs, AWS Database Blog, the AWS DR workshop, and general web. **This is a genuine gap** and the reason for the in-house game-day recommendation.
