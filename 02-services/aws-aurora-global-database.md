---
title: Aurora PostgreSQL Global Database — Multi-Region
service: aurora-postgresql-global-database
tags: [service, multi-region, aurora, postgres, database, stateful, global-database]
status: researched
replication: native (dedicated storage-layer replication infrastructure)
rpo_achievable: "0 for a planned switchover; typically seconds (sub-second lag) for an unplanned failover"
rto_achievable: "~1–2 min managed failover with warm instances; 20+ min if the secondary is headless (FAILS RTO)"
meets_targets: yes — if the secondary cluster has at least one running DB instance
updated: 2026-09-16
---

# Aurora PostgreSQL Global Database — Multi-Region

## TL;DR

- **Aurora Global Database is the only AWS-managed Postgres option that comfortably clears a 15-minute RTO**, and it does so with an order of magnitude of margin. AWS claims managed failover converts a secondary into the new primary "in typically a minute". A published AWS customer case study measured **2 minutes** for cross-Region recovery end to end. See [[#Real-world reports]].
- **Switchover and failover are two different operations with two different RTOs and two different RPOs, and conflating them is the most common mistake in this space.** Switchover (`switchover-global-cluster`) is planned, waits for the secondary to fully catch up, and gives **RPO 0**. Failover (`failover-global-cluster --allow-data-loss`) is unplanned, does *not* wait, and gives **RPO measured in seconds**. Switchover requires a healthy primary and is therefore **unavailable to you in a real disaster**. See [[#Switchover vs failover]].
- **Replication is done by dedicated infrastructure at the storage layer, not by the database engine.** AWS: "Aurora replicates data to the secondary AWS Regions using dedicated infrastructure, with latency typically under a second." There is no WAL sender competing with your workload and no replication slot that can fill the primary's disk — which removes two of the nastiest failure modes in [[aws-rds-postgres]].
- **The cheap option — a "headless" secondary with zero DB instances — does not meet our RTO.** AWS's own blog frames headless as a DR strategy "if you have an RTO greater than the time it takes to add (and make available) instances in the secondary region", and a third-party write-up puts that threshold at "more than 20 minutes". With RTO 15m you must keep **at least one reader instance running** in the secondary. That instance is the price of admission.
- **The thing that will bite:** `aws_rds_global_cluster` and `aws_rds_cluster` have a genuinely awkward relationship in Terraform — circular references, "Provider produced inconsistent final plan" on version upgrades, and attributes with no read API. You will be writing `lifecycle { ignore_changes = [...] }` blocks, and if you get them wrong Terraform will try to detach your secondary Region. See [[#Terraform implementation]].

## Does this service cross regions at all?

Yes — more so than almost anything else in this vault. An **Aurora global database** is a first-class AWS resource (`aws_rds_global_cluster` / `GlobalCluster`) that *contains* regional DB clusters. The global cluster ARN has no Region component (`arn:aws:rds::123456789012:global-cluster:my-global`), which tells you it is a genuinely global control-plane object.

Topology: one primary Region with one writer, up to **10** secondary Regions, each read-only. For our active/passive pairs we want exactly one secondary.

### Region availability — including `ca-west-1`

**This is the headline `ca-west-1` finding and it is good news.** AWS's supported-Regions table for Aurora global databases with Aurora PostgreSQL lists **Canada West (Calgary)** with the same version floors as every other Region — Aurora PostgreSQL 11 (11.9 / 11.13+), 12.8+, 13.4+, 14.3+, 15.2+, 16.1+, 17.4+, 18.3+. Identical to the Canada (Central) row. **The CA pair is not invalidated by an Aurora Global Database gap.**

All three pairs in [[research-brief]] are covered:

| Pair | Primary | Secondary | Global DB supported? |
|---|---|---|---|
| EU | Europe (Ireland) | Europe (London) | Yes, all APG majors |
| US | US East (N. Virginia) | US West (Oregon) | Yes, all APG majors |
| CA | Canada (Central) | **Canada West (Calgary)** | **Yes, all APG majors** |

**What you still have to verify for `ca-west-1`:**

1. **Instance class availability.** Aurora Global Database "requires DB instance classes that are optimized for memory-intensive applications... We recommend that you use a **db.r5 or higher** instance class." `ca-west-1` launched with R5, R6g, R6i, R6id families (and **no** 7th-generation families). `db.r5` and `db.r6g` clear the bar, so this is workable — but check it rather than assume, and check whether the class you use in `ca-central-1` exists:
   ```bash
   aws rds describe-orderable-db-instance-options --region ca-west-1 \
     --engine aurora-postgresql --engine-version 16.4 \
     --query 'OrderableDBInstanceOptions[].DBInstanceClass' --output text | tr '\t' '\n' | sort -u
   ```
   Note the implication of the "db.r5 or higher" requirement in general: **burstable `db.t3`/`db.t4g` classes are out**. Any environment currently running Aurora on a `db.t4g.medium` to save money cannot join a global database without a class change. That hits non-production hardest.
2. **`ca-west-1` is an opt-in Region.** Your account is not enabled there until you opt in, and your IAM identity is not replicated there. Beyond the enablement itself, this creates an STS credential-validity trap for cross-Region API calls — documented in detail in [[aws-rds-postgres#The STS opt-in-Region trap]]. Same fix: `AWS_STS_REGIONAL_ENDPOINTS=regional` in CI, or v2 global tokens on the account.
3. **Serverless v2 minimum capacity.** If you were planning Aurora Serverless v2 to make the standby cheap, note "for a global database with Aurora serverless, the minimum recommended capacity for the DB cluster in the primary AWS Region is **8 ACUs**." That is a meaningful floor. And it is a *primary*-side requirement, so it constrains the live cluster, not just the standby.

## Replication / mirroring options

### How Global Database replication actually works

The key architectural fact, and the reason Aurora's numbers are so much better than RDS's: **Aurora replicates the storage volume, not the WAL stream, and it does it with infrastructure that is separate from the database engine.**

From the AWS docs:

> "After any write operation, Aurora replicates data to the secondary AWS Regions using **dedicated infrastructure**, with latency **typically under a second**."

> "Aurora uses the cluster storage volume and not the database engine for fast, low-overhead replication."

Consequences worth spelling out, because they eliminate specific RDS failure modes:

| RDS Postgres cross-Region replica | Aurora Global Database |
|---|---|
| WAL sender process on the primary competes with your workload | Replication offloaded to the storage layer; "the resources of the DB instances are fully devoted to serve application read and write workloads" |
| A replication slot on the primary can retain WAL until the **primary** runs out of disk and goes down | No engine-level slot for Global Database replication. This whole failure mode disappears. |
| Replica must replay WAL; a busy replica lags | Secondary readers read from a storage volume that is already up to date |
| Baseline lag of "up to 5 minutes" with no workload (metric artefact) | `AuroraGlobalDBRPOLag` in **milliseconds** |
| Promotion is irreversible with no managed path back | Managed switchover *and* managed failover; topology is preserved and the old primary auto-rejoins |

**Metrics to watch:** `AuroraGlobalDBRPOLag` (the one that matters — how much data you would lose, in ms) and `AuroraGlobalDBReplicationLag`. For Aurora PostgreSQL, `AuroraGlobalDBRPOLag` is available on all versions. You can also query the primary directly:

```sql
SELECT * FROM aurora_global_db_status();
-- durability_lag_in_msec, rpo_lag_in_msec, visibility_lag_in_msec
```

AWS's own DR blog shows an example value of **483 ms** for one of these lag figures. That is an illustration from a blog, not a benchmark of your workload — but it is the right order of magnitude to expect.

### Option 1 — Global database with a warm secondary (recommended for prod)

Secondary cluster with **at least one** `db.r*` reader instance running. This is the configuration that meets RTO 15m with room to spare.

### Option 2 — Global database with a headless secondary (fails our RTO)

A **headless** secondary is an Aurora DB cluster in the secondary Region with **zero DB instances**. Because Aurora decouples compute from storage, the storage volume still replicates; you pay only for storage and replicated-write I/O, not compute.

This is genuinely attractive — it is, in effect, "cross-Region replication at backup prices" — and for an RPO-driven requirement it would be the obvious answer. But:

> AWS docs: "Before you can perform a switchover or failover to a headless secondary Aurora DB cluster, **you must add a DB instance to it**."

> AWS Database Blog: "you can use this approach as a DR strategy if you have an **RTO greater than the time it takes to add (and make available) instances in the secondary region**."

So the failover path becomes: create instance → wait for it to become `available` → *then* fail over. AWS does not publish how long instance creation takes. A third-party analysis puts the practical threshold at "**more than 20 minutes**" of RTO tolerance. **Our RTO is 15 minutes. Headless is out for production.**

It is, however, exactly right for **non-production**. Dev and staging do not need a 15-minute RTO; they need the topology to exist so the Terraform is exercised. Headless gives you that for storage cost. Make it a variable — see [[#Terraform implementation]].

> [!note] Aurora Serverless v2 as a middle ground
> A Serverless v2 reader in the secondary scaling down to its minimum ACU is warm (so no instance-creation wait) and cheaper than an equivalently-sized provisioned instance. But the global-database minimum-capacity guidance (8 ACUs on the primary) and the general caveat that a cold-scaled ACU floor has to absorb a full production workload on promotion make this a "model it carefully" option rather than a free win. Covered properly in [[aurora-serverless-v2]].

### Option 3 — Aurora cross-Region read replica (the old, pre-Global-Database way)

Aurora also supports a classic cross-Region read replica cluster via `replication_source_identifier` (logical replication under the hood for Aurora MySQL; for Aurora PostgreSQL the supported cross-Region mechanism is Global Database). **Do not use this for Aurora Postgres DR.** Global Database supersedes it, is faster, and is the path AWS invests in.

### Option 4 — Write forwarding

**Almost certainly irrelevant to this project, and here is exactly why.**

Write forwarding lets a *secondary* cluster accept write statements and transparently ship them to the primary Region's writer. Its purpose is **active/active-ish read scaling**: your app runs in both Regions, reads locally, and the occasional write is forwarded rather than requiring the app to hold two connections.

[[research-brief]] specifies **active/passive**. In active/passive:

- While the primary is healthy, **no traffic is served from the standby Region at all**. There is nothing to forward.
- After a failover, the standby *is* the primary, so its writes are local. There is nothing to forward.
- Write forwarding adds cross-Region latency to every forwarded write and does not support DDL or `SELECT FOR UPDATE`.
- It actively makes failover *worse*: the docs state that "with write forwarding, you do need to **update your application code or configuration to connect to the newly promoted primary Region's reader endpoint** after performing a cross-Region failover or switchover" — i.e. it reintroduces exactly the endpoint-change problem that the global writer endpoint exists to remove.

**Recommendation: leave write forwarding off** (`enable_global_write_forwarding = false`, the default). Revisit only if the posture changes from active/passive to active/active, which is explicitly out of scope.

## RPO / RTO analysis

### Switchover vs failover

This distinction is the single most important thing in this note. They are different API calls, with different guarantees, and **only one of them is available to you during an actual disaster**.

| | **Switchover** | **Managed failover** | **Manual failover (detach & promote)** |
|---|---|---|---|
| CLI | `aws rds switchover-global-cluster` | `aws rds failover-global-cluster --allow-data-loss` | `aws rds remove-from-global-cluster` then reconfigure |
| API | `SwitchoverGlobalCluster` | `FailoverGlobalCluster` | `RemoveFromGlobalCluster` |
| Former name | "managed planned failover" | (new, Aug 2023) | the original way |
| Requires healthy primary? | **Yes** | No | No |
| Waits for secondary to catch up? | **Yes** — "waits for the target secondary Region clusters to be fully synchronized with the primary" | **No** — "doesn't wait for data to synchronize" | No |
| **RPO** | **0 — "RPO is 0 (no data loss)"** | Non-zero, "typically... measured in seconds"; equals the replication lag at the moment of failure | Same as managed failover |
| **RTO** | "Your database is unavailable for a short time"; a published AWS blog says switchover "takes about a minute" | "Typically, the chosen secondary cluster assumes the primary role **within a few minutes**". AWS's launch announcement: "typically a minute" | Blog: "Promotion process should take less than 1 minute" for the detach itself |
| Topology after | Preserved. Same clusters, same Regions, roles swapped. | **Preserved.** Old primary auto-rejoins as a secondary when its Region recovers. | **Destroyed.** The global cluster is gone; you rebuild it by hand. |
| Engine version requirement | Primary and secondary must have the same major *and* minor version (patch-level rules vary by engine version) | Same requirement | **None** — this is why manual failover exists |
| Split-brain risk | None (primary is quiesced first) | Real, mitigated by best-effort "write fencing" | Real, entirely on you |
| Use it for | Planned rotation, failback, regulatory DR exercises | **A real Region outage** | When engine versions are mismatched and managed failover is refused |

**The trap:** teams rehearse with `switchover` because it is safe and lossless, then assume that is what a real failover looks like. It is not. In a real outage the primary is unreachable, switchover will refuse to run, and you are on the failover path with non-zero RPO and a split-brain risk. **Rehearse both.**

**Write fencing**, which only exists on the managed-failover path, is worth understanding:

> "When you initiate a managed failover, Aurora also attempts to halt write traffic through the highly-available Aurora storage layer. We refer to this mechanism as 'write fencing'. If the process succeeds, Aurora emits an RDS Event letting you know that writes were stopped. In the unlikely event of multiple AZ failures in a Region, it's possible that the write fencing process doesn't succeed in a timely manner... **Because fencing writes is a best-effort attempt, it's possible that writes might be momentarily accepted in the old primary Region, causing split-brain issues.**"

Best-effort. Not a guarantee. Which is why AWS's first recommendation before failing over is "take applications offline" — the most reliable fence is the one you control.

And the data you lose is not silently lost: Aurora "attempts to take a snapshot of the old storage volume at the point of failure", named `rds:unplanned-global-failover-<cluster>-<timestamp>`. **That snapshot is subject to the old primary cluster's backup retention period** — so copy it to a manual snapshot immediately if you want it for reconciliation, or it will age out while you are still arguing about what to do with it.

### Where the 15 minutes goes (managed failover, warm secondary)

| Step | Time | Notes |
|---|---|---|
| Detect + human go/no-go | excluded from RTO per [[research-brief]] | Still needs a written decision rule |
| Take applications offline in the primary Region (recommended, reduces split-brain) | seconds–1 min | Scriptable |
| `failover-global-cluster --allow-data-loss` returns | seconds | |
| Secondary promotes a reader to writer | **"typically a minute"** / "within a few minutes" | Automatic |
| Global writer endpoint DNS updates | seconds at the record, but see TTL below | Aurora emits an RDS Event when it observes the DNS change |
| App connection pools pick up the new writer | 0 → minutes, see [[#Endpoint management at failover]] | This is the risky row |
| Secondary Regions rebuild (N/A for us — we have one secondary, which is the one being promoted) | "a few minutes to several hours" | Only matters with 2+ secondaries |
| Old primary Region rejoins as a secondary | whenever the Region recovers | Automatic, no action |

**Verdict: yes, this meets RTO 15m, with a comfortable margin — provided the secondary has a running instance and the application layer can actually follow the endpoint.** The database is not the long pole. The application is. See [[aws-eks]] for the compute side.

Against **RPO 2h**: Aurora's typical sub-second lag means RPO is a non-issue by roughly four orders of magnitude. This matters strategically — it means [[rds-vs-aurora-decision]] is an **RTO and operability** decision, not an RPO one.

### The `rds.global_db_rpo` parameter — a loaded gun

Aurora PostgreSQL has a genuinely unusual feature: you can enforce a hard RPO bound by making the **primary block commits** when all secondaries fall behind it.

> "Commits the transaction if at least one secondary DB cluster has an RPO lag time less than the RPO. **Blocks the transaction if all secondary DB clusters have RPO lag times that are larger than the RPO.**"

Valid range: 20 seconds to 2,147,483,647 seconds. Dynamic, so it can be reset without a reboot.

**Recommendation: do not set it.** Reasons:

1. Our RPO target is **2 hours**. Aurora's natural lag is sub-second. Setting a bound we exceed by 7000x buys nothing.
2. It converts a *replication* problem into a *production availability* problem. A network blip between Regions now stalls writes on the live primary. You have made your DR mechanism capable of taking down production — the exact thing we removed by moving off RDS replication slots.
3. AWS itself warns against it in our topology: "**In a global database with only two AWS Regions, we recommend keeping the `rds.global_db_rpo` parameter's default value** in the secondary Region's parameter group. Otherwise, performing a failover due to a loss of the primary AWS Region could cause Aurora to pause transactions." With one secondary, losing that secondary means *all* secondaries are behind, which means commits block. Every one of our pairs is a two-Region topology.
4. It blocks major version upgrades: "you can't perform a major version upgrade of the Aurora PostgreSQL DB engine if the recovery point objective (RPO) feature is turned on."

Document the parameter, leave it at `-1`, and make sure nobody turns it on because it sounded prudent.

## Warm standby shape

| Resource | State while primary is healthy | Cost |
|---|---|---|
| `aws_rds_global_cluster` | Exists | Free (it's a control-plane object) |
| Secondary `aws_rds_cluster` | Exists, read-only, storage continuously replicated | Storage + replicated write I/O |
| Secondary `aws_rds_cluster_instance` (≥1 reader) | **Running.** Required for RTO 15m. | Full instance-hour price |
| Secondary DB cluster parameter group | Exists, must mirror the primary's | Free |
| Secondary DB subnet group, security groups | Exist | Free |
| Secondary-Region KMS CMK | Exists | ~$1/month |
| Global writer endpoint | Exists, points at the primary | Free |

**What is different from RDS:** with Aurora you are paying for *storage once per Region* plus *compute per instance*, and the secondary's storage grows with the primary's automatically. You cannot "scale the secondary to zero" and still meet RTO, but you *can* run a single small-ish reader rather than a full mirror of the primary's fleet — a secondary with one `db.r6g.large` reader against a primary with two `db.r6g.4xlarge` instances is a legitimate configuration, as long as you accept that immediately after promotion you are serving production from one small writer and need to add capacity fast.

**Sizing recommendation:** one reader instance in the secondary, same class as the primary's writer. Adding more readers post-failover is fast (minutes) and can be scripted as a *post-RTO* step; having the writer itself undersized cannot be fixed without a failover of its own.

Note also: "Aurora Global Database currently doesn't support Aurora Auto Scaling for secondary DB clusters." You cannot lean on auto-scaling to grow the secondary at failover time. It has to be a scripted `create-db-instance` or a Terraform apply, both of which are post-RTO work.

## Terraform implementation

This is where Aurora Global Database is meaningfully harder than RDS, and it is worth being explicit because the failure modes are "Terraform destroys your DR Region".

### The three-resource dance

```
aws_rds_global_cluster           (global, no region)
  ├── aws_rds_cluster (primary)   provider = aws.primary
  │     └── aws_rds_cluster_instance × N
  └── aws_rds_cluster (secondary) provider = aws.standby
        └── aws_rds_cluster_instance × 1
```

The primary cluster references the global cluster by `global_cluster_identifier`. The global cluster's `global_cluster_members` then contains the primary. That is a cycle if you are not careful, and there are two open/closed provider issues about exactly this ([#34871](https://github.com/hashicorp/terraform-provider-aws/issues/34871) — cycle dependency between `source_db_cluster_identifier` and `global_cluster_identifier`; [#34203](https://github.com/hashicorp/terraform-provider-aws/issues/34203) — the global cluster resource "kicks out global_cluster_members when reapplied").

### Greenfield (new global database)

```hcl
# modules/aurora-global/main.tf

resource "aws_rds_global_cluster" "this" {
  global_cluster_identifier = "${var.name}-${var.environment}"
  engine                    = "aurora-postgresql"
  engine_version            = var.engine_version
  database_name             = replace(var.name, "-", "_")
  storage_encrypted         = true
  deletion_protection       = var.deletion_protection
  # Required if you ever need `terraform destroy` to remove member clusters.
  # Leave false in prod; true in ephemeral environments.
  force_destroy = var.force_destroy
}

# ---------- PRIMARY ----------
resource "aws_rds_cluster" "primary" {
  provider = aws.primary

  cluster_identifier        = "${var.name}-${var.environment}-${var.primary_region}"
  global_cluster_identifier = aws_rds_global_cluster.this.id
  engine                    = aws_rds_global_cluster.this.engine
  engine_version            = aws_rds_global_cluster.this.engine_version

  database_name   = aws_rds_global_cluster.this.database_name
  master_username = "app_admin"
  manage_master_user_password   = true
  master_user_secret_kms_key_id = var.primary_kms_key_arn

  db_subnet_group_name            = aws_db_subnet_group.primary.name
  vpc_security_group_ids          = [aws_security_group.primary.id]
  db_cluster_parameter_group_name = aws_rds_cluster_parameter_group.primary.name

  kms_key_id        = var.primary_kms_key_arn   # region-local key
  storage_encrypted = true

  backup_retention_period      = var.backup_retention_period
  preferred_backup_window      = var.backup_window
  preferred_maintenance_window = var.maintenance_window

  enabled_cloudwatch_logs_exports = ["postgresql"]
  deletion_protection             = var.deletion_protection
  skip_final_snapshot             = false
  final_snapshot_identifier       = "${var.name}-${var.environment}-primary-final"

  lifecycle {
    # The global cluster drives engine upgrades. Without this you get
    # "Provider produced inconsistent final plan" on every version bump.
    ignore_changes = [engine_version]
  }
}

resource "aws_rds_cluster_instance" "primary" {
  provider = aws.primary
  count    = var.primary_instance_count

  identifier           = "${var.name}-${var.environment}-${var.primary_region}-${count.index}"
  cluster_identifier   = aws_rds_cluster.primary.id
  instance_class       = var.instance_class      # db.r5 or higher — NOT db.t*
  engine               = aws_rds_cluster.primary.engine
  engine_version       = aws_rds_cluster.primary.engine_version
  db_subnet_group_name = aws_db_subnet_group.primary.name

  db_parameter_group_name      = aws_db_parameter_group.primary.name
  performance_insights_enabled = true
  monitoring_interval          = 60
  monitoring_role_arn          = var.monitoring_role_arn

  # No effect on a global database member, but harmless and documents intent.
  auto_minor_version_upgrade = false

  lifecycle { ignore_changes = [engine_version] }
}

# ---------- SECONDARY ----------
resource "aws_rds_cluster" "secondary" {
  provider = aws.standby
  count    = var.enable_secondary ? 1 : 0

  cluster_identifier        = "${var.name}-${var.environment}-${var.standby_region}"
  global_cluster_identifier = aws_rds_global_cluster.this.id
  engine                    = aws_rds_global_cluster.this.engine
  engine_version            = aws_rds_global_cluster.this.engine_version

  # NO database_name, NO master_username — inherited from the global cluster.

  db_subnet_group_name            = aws_db_subnet_group.secondary[0].name
  vpc_security_group_ids          = [aws_security_group.secondary[0].id]
  db_cluster_parameter_group_name = aws_rds_cluster_parameter_group.secondary[0].name

  kms_key_id        = var.standby_kms_key_arn   # DESTINATION-region key. Mandatory.
  storage_encrypted = true

  backup_retention_period = var.backup_retention_period
  deletion_protection     = var.deletion_protection
  skip_final_snapshot     = true   # the data lives in the primary

  # Off. See "Option 4 — Write forwarding".
  enable_global_write_forwarding = false

  lifecycle {
    ignore_changes = [
      replication_source_identifier,  # AWS sets this; Terraform must not fight it
      global_cluster_identifier,      # flips during failover
      engine_version,                 # driven by the global cluster
    ]
  }

  # The secondary cannot attach until the primary has a live writer.
  depends_on = [aws_rds_cluster_instance.primary]
}

resource "aws_rds_cluster_instance" "secondary" {
  provider = aws.standby
  # 0 => HEADLESS (cheap, RTO > 20 min, FAILS our target)
  # 1 => warm      (meets RTO 15m)
  count = var.enable_secondary ? var.secondary_instance_count : 0

  identifier           = "${var.name}-${var.environment}-${var.standby_region}-${count.index}"
  cluster_identifier   = aws_rds_cluster.secondary[0].id
  instance_class       = var.secondary_instance_class
  engine               = aws_rds_cluster.secondary[0].engine
  engine_version       = aws_rds_cluster.secondary[0].engine_version
  db_subnet_group_name = aws_db_subnet_group.secondary[0].name

  performance_insights_enabled = true
  monitoring_interval          = 60
  monitoring_role_arn          = var.standby_monitoring_role_arn

  lifecycle { ignore_changes = [engine_version] }
}

output "global_writer_endpoint" {
  description = "Stable endpoint that always points at the current primary writer."
  value       = aws_rds_global_cluster.this.endpoint
}
```

### Brownfield — promoting an existing single-Region Aurora cluster

This is the case that actually applies: there is a live Aurora cluster in `eu-west-1` today and it must become the primary of a global database **without being replaced**.

```hcl
resource "aws_rds_global_cluster" "this" {
  global_cluster_identifier    = "${var.name}-${var.environment}"
  source_db_cluster_identifier = aws_rds_cluster.existing.arn
  force_destroy                = true    # REQUIRED with source_db_cluster_identifier

  # Do NOT also set engine/engine_version here — they conflict with
  # source_db_cluster_identifier and are inherited from the source cluster.

  lifecycle {
    ignore_changes = [source_db_cluster_identifier]
    # There is no read API for this attribute, so Terraform cannot detect
    # drift on it and will show a permanent diff without this.
  }
}

resource "aws_rds_cluster" "existing" {
  # ... all existing arguments unchanged ...

  lifecycle {
    # Terraform will want to set this after the global cluster adopts the
    # cluster. Ignoring it avoids a circular reference.
    ignore_changes = [global_cluster_identifier, engine_version]
  }
}
```

The provider docs are explicit about both of these:

> `force_destroy` is required when using `source_db_cluster_identifier`, to enable removal of cluster members during deletion.

> After initial creation, `source_db_cluster_identifier` can be removed and replaced with `engine` and `engine_version`... the global cluster will inherit the engine and engine_version values from the source cluster.

> After importing, Terraform may report differences for `force_destroy` and `source_db_cluster_identifier`... for the latter, use `ignore_changes` since the API provides no read method for this value.

**Order of operations for the live migration:**

1. Apply the `aws_rds_global_cluster` with `source_db_cluster_identifier` pointing at the live cluster. This **adopts** the existing cluster as the primary. It is a control-plane change; the cluster keeps serving. Watch the plan output like a hawk to confirm it is `+ aws_rds_global_cluster` and `~ aws_rds_cluster` (in-place), **not** `-/+`.
2. Refactor the config to move `global_cluster_identifier` onto the cluster resource and drop `source_db_cluster_identifier` from the global cluster (per the docs' "can be removed and replaced with engine and engine_version"). Verify a no-op plan.
3. Apply the secondary cluster + instance. The initial storage replication is a full copy of the volume across Regions and takes a while; Terraform will wait.
4. Repoint applications from the cluster endpoint to the **global writer endpoint**.

### `lifecycle` blocks are load-bearing, not decoration

Every `ignore_changes` above exists for a specific documented reason. Summarised, because getting these wrong is how you lose a Region:

| Attribute | Resource | Why ignore |
|---|---|---|
| `engine_version` | both clusters, both instances | Upgrades are driven through the global cluster. Without this, `terraform apply` on a version bump fails with "Provider produced inconsistent final plan" because the global cluster mutates the member clusters underneath Terraform. |
| `replication_source_identifier` | secondary cluster | AWS populates it when the secondary attaches. Terraform sees an unmanaged value and wants to clear it. |
| `global_cluster_identifier` | secondary cluster (and the primary in the brownfield case) | **This is the dangerous one.** After a failover the roles swap and this attribute's meaning changes. If Terraform "corrects" it mid-incident, it can detach a cluster from the global database. |
| `source_db_cluster_identifier` | global cluster | No read API. Permanent diff without it. |

> [!danger] The failover/Terraform interaction
> After a managed failover, the secondary cluster is the primary and the old primary rejoins as a secondary. **Your Terraform still thinks the roles are the other way round.** Unlike RDS (see [[aws-rds-postgres#Terraform state when you promote out-of-band]]), Aurora's topology is preserved, so the resources all still exist — but their *roles* are inverted relative to config.
>
> This is survivable precisely *because* of the `ignore_changes` blocks: the attributes that encode role are ignored, so a plan after failover is mostly a no-op. It is still not something to find out by running `apply` during an incident. **Freeze the pipeline at failover. Reconcile deliberately afterwards.** And note that this argues strongly for a module where "which Region is primary" is a variable rather than a resource name.

### Variable surface for the cookiecutter monorepo

| Variable | Default (prod) | Default (non-prod) | Why |
|---|---|---|---|
| `enable_secondary` | `true` | `true` | Keep the topology exercised everywhere |
| `secondary_instance_count` | **`1`** | **`0` (headless)** | The single biggest cost lever. `0` fails RTO but costs only storage. |
| `secondary_instance_class` | same as primary writer | n/a when headless | Undersizing here is paid back after promotion |
| `primary_instance_count` | 2 (writer + reader) | 1 | |
| `instance_class` | `db.r6g.*` | `db.r6g.large` | **`db.t*` is not valid** for a global database |
| `engine_version` | pinned | pinned | Must match across Regions or managed failover is refused |
| `force_destroy` | `false` | `true` | Lets ephemeral envs actually be destroyed |
| `deletion_protection` | `true` | `false` | |

## Migration path from single-region

Two distinct starting points. Both are covered here; which one applies is a question for [[rds-vs-aurora-decision]].

### A. Already on Aurora PostgreSQL, single Region

Straightforward and non-destructive — this is the brownfield flow above. No downtime, no replacement, provided:

- The cluster's instance classes are `db.r5` or higher. **If you are on `db.t3`/`db.t4g`, you must modify the instance class first**, which is a failover-and-reboot in-Region. Do it in a maintenance window before starting.
- The engine version is one the Global Database table supports in **both** Regions.
- `storage_encrypted = true` with a region-local CMK, and a CMK exists in the standby Region. Same KMS logic as [[aws-rds-postgres#KMS and the destination-region key]]: keys are regional, the secondary cluster needs its own. See [[aws-kms]].
- **Secrets Manager integration must be turned off before adding a Region.** This is an explicit, easily-missed limitation: "Secrets Manager doesn't support Aurora Global Database. When you add a Region to a global database, you must first turn off Secrets Manager integration for the DB instance." If you use `manage_master_user_password = true` (RDS-managed master credentials in Secrets Manager) on the existing cluster, **you must disable it before attaching the secondary**. Plan a credential-management change as part of this migration — and note this interacts with the Secrets Manager replication work [[research-brief]] says is already done.

### B. On RDS for PostgreSQL today, want Aurora Global Database

This is a **database migration**, not a multi-Region project, and it should be sequenced and risk-assessed as one. See [[rds-vs-aurora-decision]] for whether to do it at all. Mechanically:

- The usual path is **RDS → Aurora via an Aurora read replica of the RDS instance**, then promote. But note the hard limitation: "**If the primary DB cluster of your global database is based on a replica of an Amazon RDS PostgreSQL instance, you can't create a secondary cluster.** Don't attempt to create a secondary from that cluster... Attempts to do so **time out**, and the secondary cluster isn't created." So you must fully detach from RDS *first*, then build the global database. Two phases, not one.
- Alternative: snapshot-restore RDS → Aurora (downtime), or logical replication RDS → Aurora (near-zero downtime, but sequences/DDL caveats — see [[aws-rds-postgres#Option 3 — PostgreSQL logical replication / pglogical]]).
- **Do phase 1 (RDS→Aurora, single Region) and let it bed in before starting phase 2 (add the secondary Region).** Doing both at once means that when something misbehaves you cannot tell which change caused it.

## Failover procedure

```bash
# ===== PLANNED SWITCHOVER (RPO 0, primary healthy) =====
# Use for: regional rotation, DR exercises, failback.
# Check lag first — switchover duration is proportional to lag.
aws cloudwatch get-metric-statistics --region $STANDBY_REGION \
  --namespace AWS/RDS --metric-name AuroraGlobalDBRPOLag \
  --dimensions Name=DBClusterIdentifier,Value=$SECONDARY_CLUSTER \
  --start-time $(date -u -d '-10 min' +%FT%TZ) --end-time $(date -u +%FT%TZ) \
  --period 60 --statistics Maximum

aws rds --region $PRIMARY_REGION switchover-global-cluster \
  --global-cluster-identifier $GLOBAL_ID \
  --target-db-cluster-identifier $SECONDARY_CLUSTER_ARN


# ===== UNPLANNED MANAGED FAILOVER (RPO seconds, primary gone) =====
# 0. FREEZE THE TERRAFORM PIPELINE.
# 1. Take applications offline in the primary Region. This is the only
#    reliable write fence — Aurora's own fencing is best-effort.
kubectl --context $PRIMARY_CTX -n $NS scale deploy/api --replicas=0 || true

# 2. Fail over. Note --region is the SECONDARY's region here, unlike switchover.
aws rds --region $STANDBY_REGION failover-global-cluster \
  --global-cluster-identifier $GLOBAL_ID \
  --target-db-cluster-identifier $SECONDARY_CLUSTER_ARN \
  --allow-data-loss

# 3. Wait for the new writer.
aws rds wait db-cluster-available --region $STANDBY_REGION \
  --db-cluster-identifier $SECONDARY_CLUSTER

# 4. Confirm writable via the GLOBAL WRITER ENDPOINT (unchanged hostname).
psql -h $GLOBAL_WRITER_ENDPOINT -c "SELECT pg_is_in_recovery();"   # expect f

# 5. Restart application pods in the standby Region so pools reconnect.
kubectl --context $STANDBY_CTX -n $NS rollout restart deploy/api

# 6. Preserve the point-of-failure snapshot before retention eats it.
aws rds describe-db-cluster-snapshots --region $PRIMARY_REGION \
  --query "DBClusterSnapshots[?starts_with(DBClusterSnapshotIdentifier,'rds:unplanned-global-failover')]"
# then copy-db-cluster-snapshot to a manual snapshot.

# 7. Add capacity to the new primary (post-RTO, not inside the 15 min).
aws rds create-db-instance --region $STANDBY_REGION ...
```

### Endpoint management at failover

**Aurora gives you something RDS does not: a stable global writer endpoint.**

```
<global_cluster_id>.global-<unique_string>.global.rds.amazonaws.com
```

From the docs: "Each Aurora Global Database comes with a writer endpoint that is **automatically updated by Aurora to route requests to the current writer instance of the primary DB cluster**. With the writer endpoint, you don't have to modify your connection string after you change the location of the primary Region."

This is a large operational win. It removes the Route 53 CNAME layer that [[aws-rds-postgres]] has to build by hand, and it removes an entire failure mode (the CNAME update that didn't happen because the automation lived in the dead Region).

Available as a Terraform output: `aws_rds_global_cluster.this.endpoint`.

**It is not magic, though. Four caveats, all from the AWS docs:**

1. **DNS caching still applies, and AWS says so plainly:** "The global writer endpoint update after a global database failover or switchover **can take a long time depending upon your Domain Name Service (DNS) caching duration**." AWS's recommendation for failover prep is to "reduce the time-to-live (TTL) of your DNS cache to a low value such as **5 seconds**." The JVM/Node/`nscd`/CoreDNS caching layers described in [[aws-rds-postgres#The DNS TTL trap]] apply identically here. Aurora emits an RDS Event when it observes the DNS change on the global writer endpoint — **use that event as your automation trigger** rather than a fixed sleep.
2. **The connection-pool problem is unchanged.** A pool holding established sockets to the old writer's IP will not re-resolve. Restarting the application at failover remains mandatory. See [[aws-rds-postgres#The connection pool trap]] for the `tcp_retries2` ≈ 15-minute black-hole detail — it eats your whole RTO and it is the same on Aurora.
3. **Cross-VPC reachability.** "Your applications can't access the IP addresses in the newly promoted primary AWS Region's VPC until you set up networking between the two VPCs." For our active/passive posture this is a non-issue *if* the standby Region runs its own copy of the application in its own VPC — which is the whole point of the programme. It **is** an issue if you imagined a surviving app in the primary Region reaching across to the promoted database. Don't design for that.
4. **Renaming the global cluster changes the endpoint name.** "If you rename your Aurora Global Database, the writer endpoint name changes, and any code that uses it must be updated." Cookiecutter-driven renames are therefore breaking changes. Pin the `global_cluster_identifier` and never let a template change touch it.

**Recommendation: use the global writer endpoint directly, with a 5-second DNS TTL on any CNAME you put in front of it, plus a mandatory application restart at failover.** If you want a vanity hostname (`db.prod.internal.example.com`), CNAME it to the global writer endpoint — but that adds your TTL on top of Aurora's, so keep it at 5s. See [[aws-route53]].

**RDS Proxy** is a supported middle layer (it has documented Global Database support, and the financial-customer case study below used it), and it genuinely helps with connection churn. But it is regional, has its own limitations with global databases, and needs its own endpoint redirection at failover. Add it if you have a connection-pooling problem; do not add it purely for DR.

## Failback

**This is where Aurora is dramatically better than RDS, and it may be the strongest single argument in [[rds-vs-aurora-decision]].**

Recall the RDS position: promotion is irreversible, the old primary cannot be demoted, and failback means two full cross-Region reseeds and two outage windows.

Aurora's managed failover preserves the topology:

> "As soon as that old primary Region is healthy and available again, **Aurora automatically adds it back to the global cluster as a secondary Region.** Thus, your Aurora global database's existing replication topology is maintained."

> "After the original topology is restored, you can fail back your global database to the original primary Region by performing a **switchover** operation when it makes the most sense for your business and workload."

So failback is:

1. Wait. Aurora rebuilds the old primary Region as a secondary automatically. Aurora "creates a new storage volume for the old primary Region after it recovers" — this is a full volume rebuild and AWS says rebuild time "can take a few minutes to **several hours**, depending on the size of the storage volume and the distance between the Regions". Not instant, but **automatic and unattended**, and it happens while you are serving traffic normally from the new primary.
2. When it has caught up, run `switchover-global-cluster` at a time of your choosing. **RPO 0. RTO about a minute.**

No reseed you have to orchestrate. No second irreversible promotion. No data-loss window on the failback leg. This is the difference between a failback being a project and a failback being a change ticket.

**Caveat — the manual (detach-and-promote) path does not give you this.** If you had to use manual failover because of an engine version mismatch, the global cluster is destroyed and you rebuild from scratch, including re-adding the old primary Region as a secondary (a full cross-Region copy). This is another reason to keep engine versions strictly aligned across Regions: **version drift silently downgrades you from the good failover path to the bad one.**

**Recommendation:** for Aurora, actually fail back. The switchover is cheap enough that the "don't fail back, treat the pair as symmetric" argument from [[aws-rds-postgres#Failback]] is much weaker here. Rotating the primary Region on a schedule (quarterly) via switchover is a legitimate, low-risk way to keep the DR path warm — and AWS explicitly lists "regional rotation" as a switchover use case.

## Real-world reports

[[research-brief]] asks for real accounts, so here is what exists and what does not.

**What AWS publishes:**

| Claim | Source | Type |
|---|---|---|
| "Latency typically under a second" for replication | AWS Aurora User Guide | Vendor claim |
| "RTO can be in the order of minutes"; "RPO is typically measured in seconds" | AWS Aurora User Guide | Vendor claim |
| Managed failover converts a secondary to primary "in typically a minute" | AWS What's New, Aug 2023 | Vendor claim |
| "Typically, the chosen secondary cluster assumes the primary role within a few minutes" | AWS Aurora User Guide | Vendor claim |
| Detach-and-promote: "Promotion process should take less than 1 minute" | AWS Database Blog (Aurora PostgreSQL cross-Region DR) | Vendor walkthrough |
| Example `rpo_lag_in_msec` of **483** ms | AWS Database Blog (same) | Illustrative output, not a benchmark |
| Secondary rebuild after failover: "a few minutes to several hours" | AWS Aurora User Guide | Vendor claim |

**The one real case study found — and it is a good one.** AWS Database Blog, "How a large financial AWS customer implemented high availability and fast disaster recovery for Amazon Aurora PostgreSQL using Global Database and Amazon RDS Proxy" (Sept 2024):

- The customer's stated objectives: "**Less than a minute for in-Region failover**" and "**5-minute RTO and 15-minute RPO for cross-Region recovery**". Note how close those targets are to ours — and that theirs were *looser* on RPO.
- Measured results: in-Region failover **"less than 10 seconds in our testing"**; cross-Region recovery in **single-digit minutes, 2 minutes in testing**.
- Architecture: Aurora PostgreSQL Global Database + **RDS Proxy** for connection pooling and request buffering during failover + **Route 53 with CNAME weighting** + **Lambda canaries probing every 10 seconds**, requiring **2+ consecutive failures** before triggering + **Route 53 Application Recovery Controller** for control-plane resilience.

Three things to take from that, beyond the numbers:

1. **A 2-minute cross-Region RTO is achievable in practice, by a real customer, and independently verified enough for AWS to publish it.** Our 15-minute target is not aggressive for Aurora Global Database. It *is* aggressive for everything else in this vault.
2. **They did not rely on the database alone.** RDS Proxy to buffer connections, Route 53 ARC so the DNS change works even during a Region event, and canaries with a 2-strike rule so a single blip doesn't trigger an irreversible-ish action. The database was the easy part.
3. **Automated triggering with a 2-failure threshold** is a middle path between "fully automatic" and "wake a human". Worth putting on the table in [[failover-orchestration]] — Aurora's managed failover is much safer to automate than an RDS promotion, precisely because the topology survives and you can switch back.

**What does not exist:** despite targeted searching, **no independent (non-AWS) engineering postmortem or blog describing a real, unplanned Aurora Global Database cross-Region failover during an actual AWS Region event was found.** Not a failure of searching — it reflects how rare full-Region Aurora failovers are. Every number above is either an AWS vendor claim or an AWS-published customer test. Treat them as *upper-bound optimistic* and rehearse in your own account. This gap is logged in [[#Open questions]].

## Gotchas

1. **`db.t3`/`db.t4g` are not usable.** Global Database "requires DB instance classes that are optimized for memory-intensive applications... We recommend that you use a db.r5 or higher instance class." Any environment on burstables needs a class change first.
2. **Secrets Manager integration must be disabled before adding a Region.** "Secrets Manager doesn't support Aurora Global Database. When you add a Region to a global database, you must first turn off Secrets Manager integration for the DB instance." This collides with `manage_master_user_password = true`, which is otherwise the modern best practice.
3. **Managed switchover/failover requires matching major AND minor engine versions** (patch-level tolerance varies by version). Version drift between Regions silently downgrades you to the manual detach-and-promote path, which destroys the topology and makes failback a rebuild.
4. **`auto_minor_version_upgrade` has no effect on global database members.** "Automatic minor version upgrade doesn't apply to Aurora MySQL and Aurora PostgreSQL clusters that are part of a global database. Note that you can specify this setting for a DB instance that is part of a global database cluster, but **the setting has no effect**." You own the upgrade calendar. Set it explicitly to `false` anyway so the config reads honestly.
5. **Cluster parameter groups are per-Region and are NOT inherited at failover.** "When you promote a secondary DB cluster to take over the primary role, the parameter group from the secondary might be configured differently than for the primary. If so, modify the promoted secondary DB cluster's parameter group to conform." Same drift discipline as [[aws-rds-postgres#Parameter, option, subnet and security groups]]: one shared `locals` map, SCP-deny manual modification, scheduled diff.
6. **Neither are CloudWatch alarms, dashboards or service integrations inherited.** "Configure the promoted DB cluster with the same logging ability, alarms, and so on... configuration for these features isn't inherited from the primary." And note: "Some CloudWatch metrics, such as replication lag, are **only available for secondary Regions**" — so your lag dashboard inverts at failover and any alarm keyed to the old secondary breaks. Build both Regions' alarms up front. See [[cloudwatch-observability]].
7. **You can't stop/start clusters in a global database.** "You can't stop or start the Aurora DB clusters in your global database individually." So the "stop the non-prod database overnight" cost trick is unavailable once a cluster joins a global database. For non-prod, headless is the cost lever instead.
8. **No Aurora Auto Scaling on secondaries.** You cannot auto-grow the standby at failover time.
9. **A primary-cluster reboot or failover restarts the secondary's readers.** "If the primary AWS Region's writer DB instance undergoes a restart or failover, reader DB instances in secondary Regions also restart. The secondary cluster is then unavailable until all reader DB instances are back in sync." An in-Region AZ failover in the primary therefore briefly costs you your standby's read capability. Don't alarm on it as if it were a DR event.
10. **No Backtrack.** Aurora Backtrack is unsupported on global databases.
11. **No `inaccessible-encryption-credentials-recoverable` grace state.** "Aurora Global Database currently doesn't support the `inaccessible-encryption-credentials-recoverable` status when Amazon Aurora loses access to the AWS KMS key for the DB cluster. In these cases, the encrypted DB cluster goes **directly into the terminal `inaccessible-encryption-credentials` state**." A disabled KMS key is unrecoverable, immediately. Guard `kms:DisableKey` and `kms:ScheduleKeyDeletion` with an SCP. See [[aws-kms]].
12. **Cluster names must be globally unique across Regions.** "You can't use the same name for different Aurora DB clusters even though they're in different Regions." Cookiecutter templates that name resources `${service}-${env}` will collide. Include the Region in the cluster identifier — and then accept that the identifier does *not* tell you which Region is currently primary.
13. **Renaming the global cluster changes the global writer endpoint hostname.**
14. **Terraform: "Provider produced inconsistent final plan" on engine upgrades** unless `ignore_changes = [engine_version]` is on the member clusters.
15. **Terraform: the global cluster resource has been reported to "kick out global_cluster_members when reapplied"** ([#34203](https://github.com/hashicorp/terraform-provider-aws/issues/34203)). Read every plan against a global database stack. Never `-auto-approve` one.
16. **`force_destroy` is mandatory with `source_db_cluster_identifier`** and is a foot-gun in production — it is precisely the flag that allows member clusters to be removed on destroy. Set `false` in prod and accept that `terraform destroy` will need a manual detach.
17. **You can't apply a custom cluster parameter group during a major version upgrade** of a global database; you create the groups per Region and apply them manually afterwards.
18. **`rds.global_db_rpo` blocks writes on the primary** if set and exceeded, and AWS explicitly recommends leaving it default in a two-Region topology. Leave it alone.
19. **The point-of-failure snapshot expires.** `rds:unplanned-global-failover-*` is a system snapshot governed by the old primary's retention period. Copy it to a manual snapshot on day one of the incident.
20. **Write fencing is best-effort.** Split-brain is possible. Take the app offline in the primary Region as your real fence.

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| **Secondary instance count** | `1` warm reader — meets RTO 15m, costs a full instance 24/7 | `0` headless — storage-only cost, RTO > 20 min | **`1` in production, `0` in non-production.** The headless threshold (>20 min RTO) is above our 15-min target, so headless cannot be the prod answer. It is the right non-prod answer. |
| **Failover trigger** | Fully automated (canary + 2-strike, as in the AWS case study) | Human-gated | **Human-gated initially; automate once you have run four successful game days.** Aurora is far safer to automate than RDS because the topology survives and switchover gives you a clean way back — but earn it first. |
| **Switchover rehearsal cadence** | Never (test in non-prod only) | Quarterly production switchover | **Quarterly.** RPO 0, ~1 min RTO, and it is the only way to know the path works. AWS lists "regional rotation" as a first-class switchover use case. Schedule it like a deploy. |
| **Endpoint strategy** | Global writer endpoint directly | Vanity CNAME → global writer endpoint | **Global writer endpoint directly** unless there is a strong naming requirement. Fewer layers, fewer TTLs, one less thing to update. |
| **Write forwarding** | On | Off | **Off.** Irrelevant to active/passive and it *reintroduces* an endpoint change at failover. |
| **`rds.global_db_rpo`** | Set to enforce an RPO bound | Leave default (`-1`) | **Leave default.** Our RPO is 2h; Aurora's lag is sub-second. Setting it lets a network blip stall production writes, and AWS advises against it in two-Region topologies. |
| **Failback** | Fail back via switchover once the old Region rejoins | Treat the pair as symmetric, stay put | **Fail back via switchover.** It is RPO 0 and about a minute. The "don't fail back" argument that applies to RDS does not apply here. |
| **RDS Proxy in front** | Yes | No | **Not initially.** Add it if connection churn at failover is measurably a problem — the financial-customer case study used it, but they were chasing a 5-minute RTO with automated triggering. |

## Cost

Aurora Global Database costs more than a single-Region Aurora cluster along three axes. Check the [Aurora pricing page](https://aws.amazon.com/rds/aurora/pricing/) for current figures — do not quote from memory, and remember `eu-west-2`/`ca-west-1`/`us-west-2` have different rates from their primaries.

| Line | Warm secondary (1 reader) | Headless secondary |
|---|---|---|
| Secondary instance-hours | One `db.r*` instance, 24/7 | **$0** |
| Secondary storage | Full copy of the volume, per-GB-month | Full copy of the volume, per-GB-month |
| **Replicated write I/O** | Billed per million replicated write I/Os to each secondary Region | Same — headless does **not** avoid this |
| Cross-Region data transfer | Billed | Billed |
| Secondary backups | Per retention | Per retention |
| KMS | ~$1/key/month | Same |

**Two Aurora-specific cost notes people miss:**

1. **Replicated write I/O is a real line item and headless does not avoid it.** The storage volume replicates whether or not an instance exists. Headless saves compute only. For a write-heavy workload the I/O charge can rival the instance charge — model it from actual `VolumeWriteIOPs`, not from a guess. (If the workload is I/O-heavy, price **Aurora I/O-Optimized** against standard; it flattens I/O charges into a higher instance/storage rate, which for a global database with a chatty write workload can come out ahead. Verify availability in the standby Region.)
2. **Aurora storage is billed on the high-water mark of *used* data, per Region.** Two Regions means two storage bills, and a large one-off data load inflates both permanently until the volume is rebuilt.

**Levers, in order:**

1. **Headless secondary in non-production.** Biggest saving, zero risk, one variable.
2. **One secondary reader, not a mirror of the primary's fleet.** Add capacity after promotion.
3. **Reserved Instances / Savings Plans on the secondary reader** — it runs 24/7 forever and is the most reservable thing you own. Check RI availability in `ca-west-1`.
4. **Aurora I/O-Optimized** if replicated write I/O dominates.
5. **Right-size the primary first.** Every cost above is roughly proportional to it.

## Open questions

1. **Are we on Aurora at all today, or on RDS?** This note assumes Aurora is an option. [[rds-vs-aurora-decision]] is the blocker.
2. **What instance classes are actually orderable for `aurora-postgresql` in `ca-west-1` today?** Run `describe-orderable-db-instance-options`. The `db.r5-or-higher` requirement plus Calgary's launch inventory should be fine, but confirm.
3. **Is `ca-west-1` opted into in every account?** And is CI using regional STS endpoints?
4. **Does anything currently use `manage_master_user_password` / Secrets Manager integration on the Aurora cluster?** If so, it must be disabled before adding a Region, and we need a replacement credential-rotation story that also fits the already-completed Secrets Manager replication work.
5. **What is the actual `AuroraGlobalDBRPOLag` on our workload?** Sub-second is the claim; measure it per pair, since inter-Region RTT differs (Montreal↔Calgary is a long way; Ireland↔London is not).
6. **How long does it actually take to add an instance to a headless secondary in our account?** This determines whether headless is viable for any tier above dev. AWS publishes no number.
7. **Who owns the quarterly switchover exercise, and is there a change window for it?**
8. **Have we independently verified any of AWS's failover timing claims?** Currently we have zero first-party data and no third-party postmortem exists publicly. A game day is mandatory before the RTO is signed off.

## Sources

- [Using Amazon Aurora Global Database — Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html) — "dedicated infrastructure, with latency typically under a second", storage-layer (not engine) replication, the 1-primary/10-secondary topology, and the full limitations list (no Backtrack, no auto-scaling on secondaries, no stop/start, Secrets Manager incompatibility, terminal KMS state, unique cluster naming, secondary readers restarting when the primary restarts, the RDS-replica-derived-primary restriction).
- [Using switchover or failover in Amazon Aurora Global Database — Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html) — **the single most important source for this note.** Switchover RPO 0 vs failover RPO-in-seconds, "within a few minutes", write fencing as best-effort, the `rds:unplanned-global-failover-*` snapshot and its retention, automatic re-addition of the old primary Region, the manual detach-and-promote procedure, the matching-engine-version requirement, and the full `rds.global_db_rpo` behaviour including the two-Region warning.
- [Supported Regions and DB engines for Aurora global databases — Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.Aurora_Fea_Regions_DB-eng.Feature.GlobalDatabase.html) — **the `ca-west-1` finding.** Canada West (Calgary) is listed for Aurora PostgreSQL 11–18 with the same version floors as every other Region.
- [Configuration requirements of an Amazon Aurora global database — Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.configuration.requirements.html) — the "db.r5 or higher" memory-optimised instance-class requirement (which excludes `db.t*`), the globally-unique cluster-name rule, and the 8-ACU Serverless v2 minimum for a global database primary.
- [Connecting to Amazon Aurora Global Database — Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-connecting.html) — the global writer endpoint format and behaviour, AWS's own "reduce DNS TTL to 5 seconds" advice, the RDS Event emitted when the endpoint's DNS changes, the cross-VPC reachability caveat, the rename-breaks-the-endpoint warning, and the write-forwarding trade-offs including the fact that write forwarding *requires* a connection change after failover.
- [How a large financial AWS customer implemented HA and DR for Amazon Aurora PostgreSQL using Global Database and Amazon RDS Proxy — AWS Database Blog (Sept 2024)](https://aws.amazon.com/blogs/database/how-a-large-financial-aws-customer-implemented-ha-and-dr-for-amazon-aurora-postgresql-using-global-database-and-amazon-rds-proxy/) — **the one real case study with numbers.** Targets of "5-minute RTO and 15-minute RPO for cross-Region recovery"; measured in-Region failover "less than 10 seconds in our testing" and cross-Region recovery in single-digit minutes (2 minutes in testing); the RDS Proxy + Route 53 CNAME weighting + 10-second Lambda canary + 2-consecutive-failure + Route 53 ARC architecture.
- [Introducing Aurora Global Database Failover — AWS Database Blog](https://aws.amazon.com/blogs/database/introducing-aurora-global-database-failover/) — why managed failover exists (the old manual path destroyed the global topology and invalidated the cluster name), the point-of-failure snapshot naming, and "a few minutes to a few hours" for secondary rebuild.
- [Amazon Aurora Global Database supports failover — AWS What's New, Aug 2023](https://aws.amazon.com/about-aws/whats-new/2023/08/amazon-aurora-global-database-failover/) — "convert a secondary region into the new primary region in **typically a minute** and also maintain the multi-region Global Database configuration".
- [Cross-Region disaster recovery using Amazon Aurora Global Database for Amazon Aurora PostgreSQL — AWS Database Blog](https://aws.amazon.com/blogs/database/cross-region-disaster-recovery-using-amazon-aurora-global-database-for-amazon-aurora-postgresql/) — the detach-and-promote walkthrough ("Promotion process should take less than 1 minute"), the `aurora_global_db_status()` function, and an illustrative 483 ms RPO lag figure.
- [Achieve cost-effective multi-Region resiliency with Amazon Aurora Global Database headless clusters — AWS Database Blog](https://aws.amazon.com/blogs/database/achieve-cost-effective-multi-region-resiliency-with-amazon-aurora-global-database-headless-clusters/) — the definition of a headless secondary and the decisive caveat: use it "if you have an RTO greater than the time it takes to add (and make available) instances in the secondary region". AWS publishes no figure for that time.
- [Creating a headless Aurora DB cluster in a secondary Region — Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-attach.console.headless.html) — and the switchover/failover doc's blunt statement that "before you can perform a switchover or failover to a headless secondary Aurora DB cluster, you must add a DB instance to it".
- [terraform-provider-aws `aws_rds_global_cluster` documentation](https://github.com/hashicorp/terraform-provider-aws/blob/main/website/docs/r/rds_global_cluster.html.markdown) — the "New Global Cluster From Existing DB Cluster" pattern, the mandatory `force_destroy` with `source_db_cluster_identifier`, the `ignore_changes = [global_cluster_identifier]` circular-reference workaround, the `engine_version` "Provider produced inconsistent final plan" problem and its `ignore_changes` fix, and the note that `source_db_cluster_identifier` has no read API.
- [hashicorp/terraform-provider-aws#34871](https://github.com/hashicorp/terraform-provider-aws/issues/34871) — cycle dependency between `source_db_cluster_identifier` and `global_cluster_identifier`.
- [hashicorp/terraform-provider-aws#34203](https://github.com/hashicorp/terraform-provider-aws/issues/34203) — `aws_rds_global_cluster` reported to kick out `global_cluster_members` on reapply.
- [The AWS Canada West (Calgary) Region is now available — AWS News Blog](https://aws.amazon.com/blogs/aws/the-aws-canada-west-calgary-region-is-now-available/) — three AZs, Aurora PostgreSQL listed at launch, and the launch EC2 instance families (R5/R6g/R6i/R6id present; no 7th-generation).
- **Not found:** no independent, non-AWS engineering postmortem of a real unplanned Aurora Global Database cross-Region failover during an actual Region event. Every timing figure available publicly originates with AWS. Recorded as a finding, not a gap in searching.
- **Not usable:** the [Amazon Aurora High Availability and Disaster Recovery Features for Global Resilience whitepaper (PDF)](https://d1.awsstatic.com/Amazon%20Aurora%20High%20Availability%20and%20Disaster%20Recovery%20Features%20for%20Global%20Resilience%20Whitepaper.pdf) exists and is likely relevant, but its text could not be extracted programmatically (subset-font encoding). Worth a human read before the design is finalised.
