---
title: DynamoDB Table Naming — Migration to a Global-Tables-Compatible Name
service: dynamodb
tags: [service, multi-region, dynamodb, migration, live-blocker]
status: researched
replication: native (once names are reconciled)
rpo_achievable: "0 during the migration itself — the migration is online"
rto_achievable: "N/A — this is a migration note, not a failover note"
meets_targets: conditional
updated: 2026-09-17
---

# DynamoDB Table Naming — Migration to a Global-Tables-Compatible Name

> **This is the live blocker.** DynamoDB is the service currently in flight
> (see [[research-brief]]). Everything in [[aws-dynamodb]] is downstream of
> solving this first.

## TL;DR

1. **You cannot rename a DynamoDB table.** There is no API for it. There is no
   alias. `name` is `ForceNew` in the Terraform provider, so `terraform plan`
   will show a destroy/recreate the moment you change it. This is not
   negotiable and no amount of state surgery changes it.
2. The requirement is real and quotable: *"All replicas in a global table share
   the same table name, primary key schema, and item data."*
   ([AWS docs](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/globaltables-CoreConcepts.html))
   GSIs and LSIs must match on name and key schema too.
3. **The new name should be pair-scoped, not global.** You are building three
   independent active/passive pairs, not one mesh. `orders-eu-west-1-prod`
   becomes `orders-eu-prod`, not `orders`. Drop the *region* token, keep the
   *environment* token. That is the smallest rename that unblocks Global Tables
   and it preserves every environment boundary you already have.
4. **The safe path is dual-write + backfill + cutover reads + retire.** The
   cheap path is S3 export → `import_table` → stream-replay catch-up. Both are
   written out below as runbooks. Recommendation: **cheap path for the backfill
   mechanics, safe path for the cutover mechanics** — they compose.
5. **The thing that will bite:** your application almost certainly has the table
   name compiled in via an environment variable set at deploy time. The cutover
   is only atomic if the table name is *read at runtime from one place*. Move it
   to [[aws-ssm-parameter-store]] **before** you start, or your "instant
   rollback" is actually a redeploy.

---

## Why this is a blocker

Global Tables version 2019.11.21 adds a replica to an *existing* table via
`UpdateTable`. There is no "create a replica under a different name" parameter —
the replica is the same table, in another Region, addressed by the same name.
So a table called `orders-eu-west-1-prod` can only ever replicate to a table
called `orders-eu-west-1-prod` in `eu-west-2`. You would end up with a table in
London whose name asserts it lives in Ireland. That is not fatal, but it is a
lie baked into an immutable identifier, forever, in every IAM policy and every
log line.

Worse, it does not generalise: the moment a fourth region or a region swap
happens, the name is wrong again. You fix the naming once, now, while the
estate is still single-region and the blast radius is smallest.

### The option nobody mentions: live with the wrong name

Genuinely on the table. Adding a replica to `orders-eu-west-1-prod` in
`eu-west-2` works today, costs nothing extra, and requires **zero** migration.
The only cost is cosmetic confusion.

| | Rename properly | Live with the region token |
|---|---|---|
| Migration effort | Weeks per table | Zero |
| Risk of data loss | Non-zero (any data copy is) | Zero |
| Ops clarity at 3am | Correct | "Why is the Ireland table serving London?" |
| Survives a region swap | Yes | No — you rename anyway, later, under pressure |
| IAM policy ARNs | Rewritten once, cleanly | Already wrong, stay wrong |

**Recommendation: rename.** But rename *pair-scoped* (`orders-eu-prod`) and do
it table-by-table starting with the smallest, lowest-traffic table so the
runbook is proven on something that does not matter. If there are tables whose
migration cost genuinely exceeds the benefit — a 5 TB append-only audit table,
say — leaving the region token on *that one table* is a defensible local
exception. Write the exception down; don't let it be an accident.

### Naming: how much of the token has to go

| Current | Candidate | Works? |
|---|---|---|
| `orders-eu-west-1-prod` | `orders-eu-west-1-prod` | ✗ — the standby is not in `eu-west-1` |
| `orders-eu-west-1-prod` | `orders-eu-prod` | ✓ — pair-scoped, env-scoped |
| `orders-eu-west-1-prod` | `orders-prod` | ✓ *only if* the three deployments live in separate AWS accounts |
| `orders-eu-west-1-prod` | `orders` | ✗ unless every environment is a separate account too |

Table names must be unique per account **per Region**. So: if EU/US/CA are
separate AWS accounts, `orders-prod` is safe in all three. If they share an
account, you need the pair token. **Confirm the account topology before you
pick the name** — see [[open-questions]] at the end. Getting this wrong means
doing the migration twice.

---

## The five candidate mechanisms

### (a) New table + application dual-write + backfill + cutover reads + retire

**This is the safe path.** The application is the replication engine, so you
control ordering, you control conflict resolution, and you can stop at any
point.

- **Guarantees:** no downtime, no data loss if the conditional-write guard is
  correct, rollback available until the moment you delete the old table.
- **Costs:** application change (a dual-write shim), double write capacity for
  the duration, plus the backfill's write cost.
- **Risk:** the backfill racing live writes. Solved with a conditional write —
  the backfill writes only if the item is absent or older. See the runbook.
- **Where it fails:** if you cannot change the application (vendor code, a
  Lambda you don't own), skip to (b)+(e).

### (b) S3 export → import into a new table

DynamoDB's native export/import. Export is served from PITR, consumes **zero
read capacity**, and does not touch the table's hot path. Import **creates the
table** — this is the key constraint:

> *"During the Amazon S3 import process, DynamoDB creates a new target table
> that will be imported into. Import into existing tables is not currently
> supported by this feature."*
> ([AWS docs](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/S3DataImport.HowItWorks.html))

That constraint is a *feature* here: it is exactly what you want, because the
new table is created with the new name, and import consumes **no write
capacity** on the new table — so a 2 TB backfill does not cost you 2 TB of
WRUs. You can also create GSIs as part of the import (LSIs are not supported)
and add a global table replica once the import completes.

- **Guarantees:** a point-in-time-consistent snapshot of the source table, at
  a timestamp you choose.
- **Gap:** everything written after the export timestamp. You close that gap
  with dual-write (a) or stream replay (below).
- **Cost:** export $0.10/GB, import $0.15/GB of *uncompressed* source data,
  plus S3 storage and PUT requests. (Verify against
  [DynamoDB pricing](https://aws.amazon.com/dynamodb/pricing/on-demand/) for
  your region before you budget; the figures above are the published
  `us-east-1` rates.) Compare that with paying WRUs to write every item.
- **Caveat:** PITR must be enabled on the source table, and it must have been
  enabled long enough to cover your chosen `export_time`.

### (c) Glue / DMS

**DMS is out.** DynamoDB is a supported DMS *target* but **not** a supported
DMS *source* — the
[DMS sources list](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Introduction.Sources.html)
contains Oracle, SQL Server, MySQL, MariaDB, PostgreSQL, MongoDB, SAP ASE,
Db2, DocumentDB and S3. No DynamoDB. DMS-to-DynamoDB is for getting *out of* a
relational database, not for DynamoDB→DynamoDB. Do not spend a sprint
discovering this.

**Glue is in, and is a reasonable second choice.** Glue for Spark reads and
writes DynamoDB natively, in two flavours:

- **ETL connector** — `dynamodb.input.tableName`, parallelised by
  `dynamodb.splits`, throttled by `dynamodb.throughput.read.percent`. Consumes
  real RCUs off the live table.
- **Export connector** — `"dynamodb.export": "ddb"`, which internally issues
  `ExportTableToPointInTime` to S3 and reads from there. AWS states it
  *"performs better than the ETL connector when the DynamoDB table size is
  larger than 80 GB"* and requires PITR.

Writing back uses `dynamodb.output.tableName` with
`dynamodb.throughput.write.percent`. Cross-Region and cross-account are
supported via `dynamodb.sts.roleArn` / `dynamodb.sts.region`.

- **Why you'd choose Glue over plain export/import:** you need to *transform*
  during the move — renaming an attribute, re-keying, adding the
  `writeRegion` attribute that [[aws-dynamodb]] wants for stream filtering, or
  backfilling a `version` attribute for conditional writes. Export/import
  cannot transform; Glue can.
- **Why you wouldn't:** it costs DPU-hours and it writes through the data plane,
  so it consumes WCUs on the target. For a pure lift-and-shift, import-from-S3
  is strictly cheaper and simpler.
- **Gotcha:** *"The DynamoDB ETL reader does not support filters or pushdown
  predicates."* You always read the whole table.

### (d) `aws_dynamodb_table_export` + import

The Terraform-native expression of (b). Two resources, and they compose
properly:

```hcl
# Export the live table. Requires PITR on the source.
resource "aws_dynamodb_table_export" "orders_snapshot" {
  provider  = aws.primary
  table_arn = aws_dynamodb_table.orders_legacy.arn
  s3_bucket = aws_s3_bucket.migration.id
  s3_prefix = "orders/full"

  export_format = "DYNAMODB_JSON"
  export_type   = "FULL_EXPORT"
  # Pin the instant so a re-plan doesn't silently re-export.
  export_time   = "2026-09-20T02:00:00Z"
}
```

and the new table created *from* that export, in one resource:

```hcl
resource "aws_dynamodb_table" "orders" {
  provider     = aws.primary
  name         = "orders-eu-prod"          # the new, pair-scoped name
  billing_mode = "PAY_PER_REQUEST"         # see note on capacity below
  hash_key     = "pk"
  range_key    = "sk"

  attribute { name = "pk", type = "S" }
  attribute { name = "sk", type = "S" }

  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  point_in_time_recovery { enabled = true }
  deletion_protection_enabled = true

  import_table {
    input_format           = "DYNAMODB_JSON"
    input_compression_type = "GZIP"

    s3_bucket_source {
      bucket     = aws_s3_bucket.migration.id
      key_prefix = "orders/full/AWSDynamoDB/"
    }
  }

  lifecycle {
    # import_table is create-time only; never let a re-plan act on it.
    ignore_changes = [import_table]
  }
}
```

`aws_dynamodb_table` really does expose `import_table` —
*"Import Amazon S3 data into a new table"* — alongside `restore_source_name`,
`restore_source_table_arn` and `restore_date_time` for the PITR-restore route.
`aws_dynamodb_table_export` requires that *"Point-in-time Recovery must be
enabled on the target DynamoDB Table"*.

**The catch with `import_table` in a cookiecutter monorepo:** it is a
create-time-only argument that describes a one-off event, living inside a
resource that is meant to be permanent and templated. Six months from now
nobody will know why `orders-eu-prod` has an `import_table` block pointing at
a bucket that has been lifecycle-expired. Two ways out, pick one:

1. Do the import **out of band** with the CLI (`aws dynamodb import-table`),
   then `import` the resulting table into Terraform with a clean resource
   block. Cleanest long-term state, one manual step.
2. Keep `import_table` in HCL for the migration, then delete the block and
   `ignore_changes` in a follow-up PR once the table is `ACTIVE`. Fully
   auditable in git, at the cost of a second PR per table.

Recommendation: **(1) for production**, because the permanent HCL then looks
identical to a table that was never migrated, which is what you want a future
reader to see. **(2) for non-prod**, where the audit trail is worth more than
the tidiness.

### (e) In-place rename — **not possible. Full stop.**

There is no `RenameTable` API, no `UpdateTable` name parameter, no CLI flag, no
console button, and no Terraform escape hatch. The Terraform provider marks
`name` as `ForceNew`:

```go
names.AttrName: { Type: schema.TypeString, Required: true, ForceNew: true, ...
```

so changing it produces a **destroy-then-create** plan. If you `terraform apply`
that against a live table with `deletion_protection_enabled = false`, you delete
production. This is the single most dangerous line in this entire note.

`hash_key` and `range_key` are also `ForceNew` — if the rename is bundled with
a key-schema change, the same rule applies twice over.

**Defensive measure, do this today, before anything else:**

```hcl
resource "aws_dynamodb_table" "orders_legacy" {
  # ...
  deletion_protection_enabled = true

  lifecycle {
    prevent_destroy = true
  }
}
```

`deletion_protection_enabled` stops AWS from honouring the delete even if
Terraform asks. `prevent_destroy` stops Terraform asking. Belt and braces —
they fail in different ways (`prevent_destroy` is defeated by a `state rm`;
deletion protection is not).

And note the AWS-side trap: *"You can't delete a table used to add a new global
table replica until 24 hours have elapsed since the new replica was created."*
So even the retirement step has a mandatory waiting period.

---

## The runbook

Assumptions: one table, `orders-eu-west-1-prod` in `eu-west-1`, becoming
`orders-eu-prod`, later replicated to `eu-west-2`. Repeat per table. **Do the
smallest table first and treat it as a rehearsal, not a migration.**

Every phase names its rollback. If a phase has no rollback, it is called out.

---

### Phase 0 — Make the table name a runtime value (prerequisite, do this first)

If the application resolves its table name from an env var baked at deploy
time, "flip reads back" means "redeploy", which is minutes you do not have at
3am, and it is not atomic across pods.

Put the table name in SSM Parameter Store, read it at startup **and** on a
short refresh interval (or on a config-reload signal):

```hcl
resource "aws_ssm_parameter" "orders_table_name" {
  provider = aws.primary
  name     = "/${var.env}/dynamodb/orders/table_name"
  type     = "String"
  value    = aws_dynamodb_table.orders_legacy.name

  lifecycle {
    # The cutover changes this out of band; Terraform must not fight it.
    ignore_changes = [value]
  }
}
```

Also add two independent booleans, not one enum — `ORDERS_DUAL_WRITE` and
`ORDERS_READ_FROM_NEW`. They must be separately flippable, because the states
you need are (old only) → (dual-write, read old) → (dual-write, read new) →
(new only), and an enum tempts someone into skipping a step.

> **Rollback:** trivial, it is additive. Nothing reads the new values yet.
>
> **Note:** SSM Parameter Store has no native cross-region replication — see
> [[aws-ssm-parameter-store]], which is flagged in [[README]] as a real gap.
> For *this* migration that does not matter (it is all in the primary region),
> but the same parameter has to exist in the standby before failover.

### Phase 1 — Create the new table, empty, correctly named

Terraform, new resource address, same module. On-demand billing
(`PAY_PER_REQUEST`) for the whole migration — you do not know the backfill's
write shape, autoscaling will not keep up with it, and getting throttled
mid-backfill is a bad afternoon. Switch to provisioned later if the cost model
says so.

Turn on from day one: PITR (required for export, and you will want it),
streams with `NEW_AND_OLD_IMAGES`, deletion protection, and
`server_side_encryption` matching the old table.

> **Rollback:** `terraform destroy` the one resource. It is empty.
> **Verify before proceeding:** `DescribeTable` on the new table returns
> `ACTIVE` and a key schema *byte-identical* to the old one, including every
> GSI name and key. A GSI mismatch discovered in Phase 4 costs you the whole
> backfill.

### Phase 2 — Turn on dual-write

Flip `ORDERS_DUAL_WRITE=true`. The application writes to old (authoritative,
errors propagate) and new (best-effort, errors logged and counted, **never**
fail the request on the new table). Deletes must be dual too — a missed delete
resurrects an item.

The new-table write must be **conditional**, so that a slow backfill cannot
overwrite a newer live write:

```
ConditionExpression: attribute_not_exists(pk) OR #ver < :ver
```

If the items have no version or `updatedAt` attribute, add one in this phase
and let it populate naturally, or use Glue (option c) to backfill the attribute
first. Doing the backfill without a guard and "just running it at a quiet time"
is how tables lose writes.

> **Rollback:** flip the flag off. The new table drifts stale and is discarded.
> **Verify:** `SuccessfulRequestLatency`/`ThrottledRequests` on the new table
> are non-zero and healthy — i.e. the dual-write is actually firing. Emit a
> counter of dual-write failures and alarm on it at zero-tolerance.
> **Soak:** at least 24h. You want a full daily traffic cycle, including the
> nightly batch job that nobody remembered.

### Phase 3 — Backfill

Record `T_dual` = the timestamp dual-write went live and has been verified
error-free. Only items written **before** `T_dual` need backfilling.

**Backfill mechanism, in order of preference:**

1. **Export at `T_dual` → import.** But import creates the table, and you
   already created it in Phase 1. So either (i) reverse the order — do the
   export/import *first*, in Phase 1, using `import_table`, and let dual-write
   start after; or (ii) use Glue/a scan-writer to load into the existing table.
   Option (i) is cheaper and faster. Its cost is that the dual-write window
   must start before the export timestamp — which is fine, and is in fact the
   correct ordering:

   > **Corrected phase order for the cheap path:** Phase 2 (dual-write, writing
   > to a table that does not exist yet is impossible) — so instead: create the
   > table from the import *first*, then dual-write, then close the gap between
   > the export instant and the dual-write instant with **incremental export**
   > or **stream replay**. See the variant runbook below.

2. **Glue export-connector job**, reading a PITR export at `T_dual` and writing
   with conditional puts. Handles the transform case. Rate-limit with
   `dynamodb.throughput.write.percent`.

3. **Scan + `BatchWriteItem` worker** — the
   [thumbtack/dynamodb-rename](https://github.com/thumbtack/dynamodb-rename)
   shape: consistent `Scan` snapshot, `BatchWriteItem` to the destination,
   DynamoDB Streams to replay everything that changed during the copy, with
   client-side rate limiting on both sides. Apache-2.0, and useful as a
   *reference design* even if you write your own. Note it is a third-party tool
   and its current maintenance status should be checked before you depend on
   it in production.

> **Rollback:** stop the job. The new table has partial data; dual-write keeps
> it converging on the live set but historical items are missing, so you simply
> do not proceed to Phase 5. Nothing user-visible has happened.
> **This phase is fully reversible and can be re-run from scratch** — the
> conditional write makes it idempotent. That property is the whole point.

### Phase 4 — Verify. Do not skip. Do not eyeball it.

`DescribeTable`'s `ItemCount` is **not** a verification tool — it is updated
approximately every six hours and is explicitly documented as approximate. Item
counts also legitimately differ if the source had duplicate keys (import
overwrites: *"the number of items processed in the import table description
will not match the number of items in the target table"*).

Real verification, cheapest first:

1. **Export both tables to S3 at the same `export_time`** (zero read capacity
   consumed on either) and diff them with Athena or a Glue job. This is the
   only method that actually proves equality, and it costs $0.10/GB × 2.
2. **Sampled comparison** — pull N thousand random keys from the old table's
   export and `GetItem` both sides. Catches systemic breakage, not tail cases.
3. **Reconciliation counter** — a canary that writes a known item every minute
   and asserts it lands in both.

> **Rollback:** n/a — this phase is read-only.
> **Gate:** do not proceed on a "close enough" count. Proceed on a zero-diff
> export comparison, or on a documented, understood, accepted delta.

### Phase 5 — Cut over reads

Flip `ORDERS_READ_FROM_NEW=true` in SSM. Dual-write stays **on**. This is the
point of no confidence, not the point of no return: both tables are still
complete and current, so the rollback is a single parameter flip with no data
implications whatsoever.

Roll it out gradually if the app supports it: one pod, then one AZ, then all.
If it does not support gradual rollout, the flag flip is all-at-once and you
need the monitoring below to be already in place.

> **Rollback:** flip `ORDERS_READ_FROM_NEW=false`. Instant, lossless, because
> dual-write never stopped. **This is the single most valuable property of the
> whole design and it is why dual-write must not be turned off in this phase.**
> **Soak:** a week. Minimum. Through a full business cycle, a deploy, and a
> weekend.

### Phase 6 — Stop dual-write

Flip `ORDERS_DUAL_WRITE=false`. New table is now sole authority.

> **Rollback:** this is the first irreversible-ish step. Reverting reads to the
> old table now loses everything written since this flip. To restore
> reversibility you would have to reverse the dual-write direction (new →
> old), which is possible via a Lambda on the new table's stream, and is worth
> doing for a genuinely critical table. For most tables, a week of Phase 5 soak
> plus PITR on both tables is proportionate.
> **Keep PITR on the old table** for at least your longest plausible
> "we found a bug in last month's data" window.

### Phase 7 — Retire the old table

Not before: the Phase 6 soak has passed, a final export of the old table is
sitting in S3 (or Glacier), and CloudWatch confirms **zero** requests to the
old table for the full soak period.

Verify the last point properly — alarm on `ConsumedReadCapacityUnits` and
`ConsumedWriteCapacityUnits` > 0 on the old table, and check CloudTrail data
events if enabled. There is always one forgotten reporting script.

Then: remove `prevent_destroy`, set `deletion_protection_enabled = false`,
remove the resource from HCL, apply. Or — safer — use a `removed` block with
`destroy = false` to release it from Terraform first, and delete it by hand a
week later, so that the Terraform change and the destructive act are separate
events:

```hcl
removed {
  from = aws_dynamodb_table.orders_legacy

  lifecycle {
    destroy = false   # forget it, do not delete it
  }
}
```

`removed` is Terraform 1.7+. Prefer it to `terraform state rm` — it is
reviewable in a PR and it shows up in the plan.

> **Rollback:** none after the actual delete. Your rollback is the S3 export
> and an `import_table`, which is hours, not minutes. Hence the paranoia above.

### Phase 8 — *Now* add the replica

Only once the old table is gone (or at minimum, quiescent and past the 24h
rule) do you touch Global Tables. Adding the replica is a separate change, a
separate PR, and a separate blast radius. See [[aws-dynamodb]].

---

### Variant runbook — the cheap path (export/import first)

For a large table where paying WRUs for the whole backfill is the dominant
cost, reorder:

| Step | Action | Rollback |
|---|---|---|
| C0 | Enable PITR on the old table. Wait for it to be usable. | Disable |
| C1 | Turn on dual-write **to a temporary buffer**, not to the new table — e.g. the old table's stream feeding an SQS FIFO queue or an S3 journal. This starts the clock and captures every change from `T_start`. | Stop the consumer |
| C2 | `ExportTableToPointInTime` at `T_export` (> `T_start`). Zero read capacity consumed. | Delete the S3 objects |
| C3 | Create the new table **from the export** — `import_table` in Terraform, or `aws dynamodb import-table` out of band. No write capacity consumed. | `terraform destroy` the new table |
| C4 | Replay the buffer from `T_export` forward into the new table with conditional writes. Ordering is per-item, which is all DynamoDB guarantees anyway. | Truncate and re-import |
| C5 | Switch the buffer consumer to live tailing — now it is a real-time replicator, and you are in the same state as Phase 3-complete above. | Stop it |
| C6 | Continue from **Phase 4** (verify) above. | As above |

The 24-hour stream retention limit is the trap here. Elias Brange's write-up of
this exact pattern (cross-account, same mechanics) states it plainly:
*"DynamoDB streams can only store records for up to 24 hours. Thus, you must be
able to export all data, import it to a new table, and enable the stream within
24 hours."* If your export+import exceeds 24 hours — plausible for a multi-TB
table — you **must** buffer to durable storage (step C1), not rely on the
stream itself. That is why C1 exists as a separate step rather than "just turn
on a stream consumer at cutover time".

Alternatively, **incremental export** closes the gap without a stream at all:
export the delta between `T_export` and `T_now` from PITR and apply it. This is
supported by `aws_dynamodb_table_export` via `export_type =
"INCREMENTAL_EXPORT"` and an `incremental_export_specification` block with
`export_from_time` / `export_to_time`. It is the lowest-code option and it has
no 24-hour cliff — but it is batch, so you still need dual-write or a final
short stream tail for the last few minutes. Incremental export has a documented
minimum charge of 10 MB per export, so a tight polling loop is wasteful.

---

## Terraform: adopting the new table without a destroy

Three blocks matter, and they do different jobs. The most common mistake is
reaching for `moved` when you need `import`.

### `moved` — changes the *address*, never the *object*

`moved` rewrites which Terraform address owns an existing remote object. It
**cannot rename the remote object**. It is useless for renaming a DynamoDB
table, and reaching for it here is a category error.

It is extremely useful for the *other* half of this work: when the cookiecutter
module path changes because tables move from a per-region module to a
pair-scoped one, `moved` keeps the same physical table under the new address
with no plan churn:

```hcl
# The table did not change. Only where it lives in the config changed.
moved {
  from = module.dynamodb_tables["orders"].aws_dynamodb_table.this
  to   = module.dynamodb_global["orders"].aws_dynamodb_table.this
}
```

That is the correct and only use of `moved` in this migration.

### `import` — adopts an out-of-band table

If you created the new table with the CLI (recommended for production, above),
adopt it. `import` blocks accept `provider`, so this works cleanly with
aliases:

```hcl
import {
  to       = aws_dynamodb_table.orders
  id       = "orders-eu-prod"
  provider = aws.primary
}
```

and with `for_each` if you are migrating a set of tables in one go:

```hcl
import {
  for_each = toset(["orders", "customers", "inventory"])
  to       = aws_dynamodb_table.tables[each.key]
  id       = "${each.key}-eu-prod"
  provider = aws.primary
}
```

Then `terraform plan` and **read every line**. The expected plan is *no
changes*. Anything else — a `billing_mode` drift, a missing tag, a
`point_in_time_recovery` flip — means your HCL does not match reality, and you
fix the HCL, not the table. Any `# forces replacement` in that plan is a
stop-the-line event.

### `removed` — releases the old table without deleting it

Shown in Phase 7. `destroy = false` is the whole point: Terraform forgets the
table, the table survives, and the deletion is a separate, deliberate,
human act.

### State surgery you should *not* do

There is a folk remedy that goes: `terraform state rm` the old table, change
`name` in HCL, `terraform import` the new table into the same address. It
"works" in that it produces a clean plan. It is also a loaded gun — between the
`state rm` and the `import`, the old table is unmanaged and the new table is
unmanaged, and any concurrent apply from CI will happily create a *third*
table. If you must do it, do it with `removed` and `import` blocks in a single
PR so that the transition is one plan, one apply, one audit record — not two
CLI invocations separated by however long the coffee break was.

### The provider-alias skeleton for a cookiecutter monorepo

```hcl
# providers.tf — generated by cookiecutter per environment
provider "aws" {
  alias  = "primary"
  region = var.primary_region        # eu-west-1
  default_tags { tags = local.tags }
}

provider "aws" {
  alias  = "standby"
  region = var.standby_region        # eu-west-2
  default_tags { tags = local.tags }
}
```

```hcl
# modules/dynamodb-table/variables.tf — the surface worth exposing
variable "name" {
  description = "Pair-scoped table name, e.g. orders-eu-prod. NO region token."
  type        = string
  validation {
    # Catch the mistake mechanically instead of in review.
    condition     = !can(regex("(eu|us|ca|ap|sa|me|af)-[a-z]+-[0-9]", var.name))
    error_message = "Table name must not contain an AWS region; Global Tables replicas share one name."
  }
}

variable "replica_regions" {
  description = "Regions to replicate to. Empty list = single-region table."
  type        = list(string)
  default     = []
}

variable "billing_mode" {
  type    = string
  default = "PAY_PER_REQUEST"
}
```

The `validation` block is the cheap win. It makes the naming rule impossible to
violate by accident across every environment cookiecutter generates, which is
worth more than any amount of documentation.

---

## Gotchas

- **`name`, `hash_key` and `range_key` are all `ForceNew`.** Any plan that
  touches them on a live table is a destroy. Set `prevent_destroy` *and*
  `deletion_protection_enabled` before you start editing.
- **`ItemCount` from `DescribeTable` is approximate** (updated roughly every
  six hours). It is not a verification tool. Export-and-diff is.
- **Duplicate keys in the source silently collapse on import.** *"If any keys
  are not unique, the import will overwrite the associated items until only the
  last overwrite remains."* If the old table has a composite key and your export
  pipeline flattens it, you will lose rows and the item count will not match.
- **LSIs cannot be created by import-from-S3.** GSIs can. If the table has an
  LSI, import-from-S3 cannot reproduce it — and LSIs can only be created at
  table-creation time, so you are back to a scan-and-write backfill into a
  table you created with the LSI. Check for LSIs *before* choosing the path.
- **24-hour stream retention** is the hard deadline on any stream-replay
  catch-up. Buffer durably if the backfill might exceed it.
- **You cannot delete a table used to seed a global table replica for 24
  hours** after the replica is created. Plan the retirement window around it.
- **IAM policies contain table ARNs.** The rename invalidates every policy
  statement that names the old table, in every role, in every account,
  including ones you don't own (a partner's cross-account role, a SaaS
  integration). Grep the whole estate for the old name before Phase 5, not
  after. Same for resource policies, `Condition` keys, VPC endpoint policies
  and SCPs. See [[aws-iam]].
- **CloudWatch alarms, dashboards and Contributor Insights are per-table-name.**
  They will all silently go quiet at cutover. Recreate them on the new table
  in Phase 1, not Phase 5, so you have a baseline to compare against.
- **Application Auto Scaling targets are per-table-name too** and are separate
  resources. If you use provisioned mode, `aws_appautoscaling_target` /
  `aws_appautoscaling_policy` must be recreated for the new table. This is a
  strong argument for on-demand during the migration.
- **Backup plans and AWS Backup selections** that select by tag will pick up
  the new table automatically; ones that select by ARN will not. See
  [[aws-backup]].
- **Dual-write doubles your write cost** for the duration, and on a provisioned
  table can trip autoscaling into a step change you did not budget for.
- **Deletes are the hard case in dual-write.** An item deleted in the old table
  but not the new one resurrects on cutover. Tombstone or dual-delete
  explicitly; do not assume the backfill handles it.
- **TTL deletes bypass your dual-write shim entirely.** If the old table has a
  TTL attribute, items expiring during the migration disappear from the old
  table with no application write to mirror. The new table needs the same TTL
  configuration set in Phase 1 so the same items expire there independently.
  Verify the TTL attribute name matches exactly.

---

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Rename at all? | Rename to a pair-scoped name | Keep the region token, accept the lie | **Rename.** One-time cost, permanent clarity, survives a future region swap. Allow documented per-table exceptions for genuinely enormous tables. |
| New name shape | `orders-eu-prod` (pair token) | `orders-prod` (no pair token) | **Depends on account topology.** If EU/US/CA are separate accounts, B is cleaner. If one account, A is mandatory. *Resolve this before naming anything.* |
| Backfill mechanism | Export→`import_table` (no WCU, cheapest) | Glue job (transforms, costs WCU) | **Export→import** unless you need a transform during the move. Then Glue with the export connector. |
| Catch-up mechanism | Application dual-write | Stream-replay Lambda | **Dual-write** if you can change the app — it is the only option whose rollback is a flag flip. Stream replay when you cannot. |
| Gap closure for the cheap path | Durable buffer (SQS/S3) from `T_start` | Incremental export loop | **Durable buffer** for large tables (no 24h cliff, real-time). **Incremental export** for tables where a batch cutover window is acceptable — far less code. |
| `import_table` in HCL | Keep it in the permanent resource | CLI import + `import` block | **CLI + `import` block for prod**, HCL for non-prod. |
| Retiring the old table | `terraform destroy` | `removed` + manual delete later | **`removed { destroy = false }`**, then delete by hand after a soak. Separates the config change from the destructive act. |
| Billing mode during migration | Keep provisioned + autoscaling | On-demand for the duration | **On-demand.** Autoscaling cannot track a backfill's write shape, and throttling mid-backfill is avoidable pain. Re-evaluate after. |
| Table name resolution | Env var at deploy time | SSM parameter read at runtime | **SSM parameter.** It is what makes Phase 5's rollback instant instead of a redeploy. |

---

## Cost

Per table, roughly, for the export/import path. **Verify current rates for your
region at [DynamoDB pricing](https://aws.amazon.com/dynamodb/pricing/on-demand/)
before budgeting — figures below are published `us-east-1` rates and regional
variation is real.**

| Line item | Rate (us-east-1, published) | For a 500 GB table |
|---|---|---|
| PITR (prerequisite) | $0.20 / GB-month | ~$100/month while enabled |
| Full export to S3 | $0.10 / GB | ~$50 |
| Import from S3 | $0.15 / GB (uncompressed source) | ~$75 |
| S3 storage of the export | S3 standard rates | ~$12/month, delete when done |
| Write capacity for backfill | **$0** via import | — |
| Write capacity for backfill | vs. ~$0.625 per million WRUs if you scan-and-write | materially more for a large item count |
| Dual-write overhead | 2× WRU for the migration window | duration-dependent |

The headline: **import-from-S3 consumes no write capacity**, which is why it
beats every scan-and-write approach on cost for anything above a few GB. The
Glue path re-introduces the write cost, so only pay it when you need the
transform.

---

## Open questions

1. **Is each of EU / US / CA a separate AWS account?** This determines whether
   the new name needs a pair token. It must be answered before the first table
   is renamed, or the rename happens twice.
2. **How many tables, and what is the largest?** The runbook's shape changes at
   roughly the 80 GB mark (where AWS recommends the Glue export connector over
   the ETL connector) and again wherever export+import exceeds 24 hours.
3. **Do any tables have LSIs?** LSIs cannot be created by import-from-S3 and
   cannot be added after table creation. Those tables need a different path.
4. **Does the application read its table name at runtime or at deploy time?**
   Phase 0 is either an afternoon or a sprint depending on the answer.
5. **Do any tables have TTL enabled?** TTL deletes bypass the dual-write shim.
6. **Are there cross-account consumers** (partner roles, SaaS integrations,
   analytics jobs) with the old table ARN hard-coded in a policy they own and
   you don't? Lead time on those is external and can dominate the schedule.
7. **Is there an existing legacy (2017.11.29) global table anywhere?** If so it
   needs upgrading too, and that is a different procedure — see [[aws-dynamodb]].

---

## Sources

- [Global tables core concepts — Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/globaltables-CoreConcepts.html) — the definitive "all replicas share the same table name, primary key schema, and item data" statement, plus the 24-hour rules on deleting a seed table and on disabled Regions.
- [DynamoDB data import from Amazon S3: how it works](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/S3DataImport.HowItWorks.html) — import creates a new table only, consumes no write capacity, GSIs supported / LSIs not, duplicate-key overwrite behaviour.
- [Sources for AWS DMS](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Introduction.Sources.html) — the list that proves DynamoDB is not a supported DMS source.
- [AWS Glue — DynamoDB connections](https://docs.aws.amazon.com/glue/latest/dg/aws-glue-programming-etl-connect-dynamodb-home.html) — ETL vs export connector, the 80 GB guidance, all connection options, cross-Region via `dynamodb.sts.region`, and the "no pushdown predicates" limitation.
- [hashicorp/aws — `aws_dynamodb_table` docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/dynamodb_table) — `import_table`, `restore_source_name`, `restore_source_table_arn`, `restore_date_time`, replica block, the mutual exclusivity warning with `aws_dynamodb_table_replica`.
- [hashicorp/aws — `internal/service/dynamodb/table.go`](https://github.com/hashicorp/terraform-provider-aws/blob/main/internal/service/dynamodb/table.go) — the provider schema itself, showing `name`, `hash_key` and `range_key` as `ForceNew: true` and `replica` as not.
- [hashicorp/aws — `aws_dynamodb_table_export` docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/dynamodb_table_export) — full/incremental export arguments and the PITR prerequisite.
- [Terraform `import` block reference](https://developer.hashicorp.com/terraform/language/block/import) — confirms `provider` and `for_each` are supported inside `import` blocks.
- [Terraform `removed` block reference](https://developer.hashicorp.com/terraform/language/block/removed) — `destroy = false` to forget without deleting; why it beats `terraform state rm`.
- [Elias Brange — Migrate DynamoDB tables with zero downtime and no data loss](https://www.eliasbrange.dev/posts/migrate-dynamodb-with-zero-downtime/) — a real, worked export → import → stream-delta migration, and the source of the 24-hour stream-retention constraint framed as a hard deadline.
- [thumbtack/dynamodb-rename](https://github.com/thumbtack/dynamodb-rename) — Apache-2.0 reference implementation of scan-snapshot + `BatchWriteItem` + stream replay, with client-side rate limiting. Check maintenance status before depending on it.
- [Alex DeBrie — How to Migrate an existing DynamoDB Table to a Global Table](https://www.alexdebrie.com/posts/dynamodb-migrate-global-table/) — **historical context only.** Written against global tables 2017.11.29, when the target had to be empty. The article itself notes AWS later added in-place conversion. Read it for the backfill-with-conditional-writes pattern, not for the procedure.
- [DynamoDB pricing](https://aws.amazon.com/dynamodb/pricing/on-demand/) — export, import, PITR and request-unit rates. Regional variation applies.
- [Best practices for global tables — Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/globaltables-bestpractices.html) — the CloudFormation conversion procedure (retain → remove from stack → convert → re-import), which is the same shape as the Terraform `removed`/`import` dance recommended here.
