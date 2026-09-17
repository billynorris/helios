---
title: Amazon DynamoDB — Multi-Region
service: dynamodb
tags: [service, multi-region, dynamodb]
status: researched
replication: native
rpo_achievable: "sub-second typical (async, last-writer-wins); zero with MRSC"
rto_achievable: "< 1 min for the data layer, if capacity is pre-warmed"
meets_targets: yes
updated: 2026-09-17
---

# Amazon DynamoDB — Multi-Region

> Prerequisite reading: [[dynamodb-table-naming-migration]]. None of this is
> reachable until the per-region table names are reconciled — Global Tables
> require every replica to share one name.

## TL;DR

- **Global Tables version 2019.11.21 is the answer and it comfortably beats
  both targets.** Replication is typically sub-second, so RPO 2h is met with
  ~7,000× margin, and the standby table is live and readable at all times, so
  the data layer contributes ~0 to RTO.
- **But it is multi-active, and you asked for active/passive.** Every replica
  is writable. There is no "read-only replica" setting. A stray write in the
  standby replicates *back* to the primary and can silently win under
  last-writer-wins. This is the single biggest conceptual mismatch between what
  the service does and what this project wants, and it needs an IAM deny to
  contain — carefully, because a careless deny breaks replication permanently
  after 20 hours.
- **Capacity is your real RTO threat, not replication.** A replica autoscaled
  down to idle takes 100% of traffic at failover and throttles within seconds.
  Autoscaling reacts in minutes. Use on-demand and pre-warm.
- **Streams fire in every region.** A Lambda trigger deployed in both regions
  processes every change twice. This is the classic Global Tables production
  bug. Fix with a `writeRegion` attribute plus a Lambda event filter — and know
  that the fix does not cover deletes.
- **`TransactWriteItems` is region-local.** Atomicity holds only in the region
  where the transaction was invoked; other replicas can observe partially
  applied transactions. Nobody expects this.

---

## Does this service cross regions at all?

Yes, natively and well — DynamoDB is one of the few AWS data services where
multi-region is a first-class, fully managed feature rather than a pattern you
assemble. A *global table* is a set of replica tables in different Regions that
DynamoDB keeps in sync. Per AWS:

> *"A replica table (or replica) is a DynamoDB table that functions as part of
> a global table. A global table consists of two or more replica tables across
> different AWS Regions. Each global table can have only one replica per AWS
> Region. All replicas in a global table share the same table name, primary key
> schema, and item data."*

Global tables carry a **99.999% availability SLA**, against 99.99% for
single-region tables.

Global tables are available in **all AWS Regions where DynamoDB is available**,
including `ca-west-1` — see the `ca-west-1` section below, which has a real
caveat.

### Consistency modes: MREC vs MRSC

Two modes, chosen at creation and **immutable thereafter**:

| | MREC (multi-Region eventual consistency) | MRSC (multi-Region strong consistency) |
|---|---|---|
| Default? | Yes | No, opt-in |
| Replicas | 2+, any Regions | **Exactly 3** — 3 replicas, or 2 replicas + 1 witness |
| RPO | Sub-second typical, non-zero | **Zero** |
| Conflict resolution | Last-writer-wins on item timestamp | None needed — strongly ordered |
| Transactions | Region-local atomicity only | **Not supported — returns an error** |
| TTL | Supported, synchronised, deletes replicated | **Not supported** |
| Streams | On by default, cannot be disabled (it *is* the replication mechanism) | Off by default, optional; records identical across replicas |
| `ReplicationLatency` metric | Published | **Not published** |
| Region availability | All Regions | Three fixed Region sets (below) |
| Account model | Same-account or multi-account | **Same-account only** |

MRSC's Region sets, verbatim from
[AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/dynamodb-global-tables/faq.html):

- **US set:** us-east-1, us-east-2, us-west-2
- **EU set:** eu-west-1, eu-west-2, eu-west-3, eu-central-1
- **AP set:** ap-northeast-1, ap-northeast-2, ap-northeast-3

Read that against our pairs:

| Pair | MRSC possible? |
|---|---|
| EU — eu-west-1 / eu-west-2 | **Yes** — both in the EU set. Third member: eu-west-3 or eu-central-1 as a witness. |
| US — us-east-1 / us-west-2 | **Yes** — both in the US set. Third member: us-east-2 as a witness. |
| CA — ca-central-1 / ca-west-1 | **No.** There is no Canadian MRSC Region set. |

**Recommendation: use MREC.** The RPO target is 2 hours. MRSC buys you zero
RPO — a capability you were not asked for — and charges for it in write
latency, in losing transactions entirely, in losing TTL entirely, in a
mandatory third Region, and in an architecture that cannot be applied to the CA
pair, breaking the "one pattern, three pairs" property that keeps this estate
manageable. A 2-hour RPO does not need consensus writes. Note the MRSC option
in [[open-decisions]] and move on.

It also matters for data residency that the CA pair *cannot* use MRSC without
placing a witness outside Canada — see [[data-residency]].

---

## Global Tables versions: 2019.11.21 vs 2017.11.29

If any table in the estate is already a legacy global table, it must be
upgraded before this work proceeds. Check with:

```bash
# Legacy check: success = it IS legacy. GlobalTableNotFoundException = it is not.
aws dynamodb describe-global-table --global-table-name orders-eu-prod --region eu-west-1

# Current check: look for "GlobalTableVersion": "2019.11.21"
aws dynamodb describe-table --table-name orders-eu-prod --region eu-west-1
```

| | 2017.11.29 (Legacy) | 2019.11.21 (Current) |
|---|---|---|
| Write cost, `PutItem` 1 KB | 2 rWRU **per region** | 1 rWRU |
| Write cost, `UpdateItem` 1 KB | 2 rWRU source + 1 rWRU per destination | 1 rWRU both |
| Write cost, `DeleteItem` 1 KB | 1 rWRU source + 2 rWRU per destination | 1 rWRU both |
| Region availability | **11 Regions only** | All Regions |
| Creation | Create empty tables in each Region, then `CreateGlobalTable` | `UpdateTable` to add a replica to an existing table |
| Add a replica to a table with data | **No — all replicas must be emptied first** | Yes |
| Control plane API | `CreateGlobalTable`, `UpdateGlobalTable`, `DescribeGlobalTable`, … | `DescribeTable` / `UpdateTable` |
| Stream records per write | **Two** | One |
| `aws:rep:*` attributes | Populated (`aws:rep:deleting`, `aws:rep:updateregion`, `aws:rep:updatetime`) | Not populated |
| TTL settings synced across replicas | No | Yes |
| TTL deletes replicated | **No** | Yes |
| Auto scaling settings synced | No | Yes |
| GSI settings synced | No | Yes |
| Encryption-at-rest settings synced | No | Yes |
| `PendingReplicationCount` metric | **Published** | **Not published** |

That last row matters and contradicts a lot of blog content: **on the current
version there is no `PendingReplicationCount` metric.** Anything you read that
tells you to alarm on it is describing the legacy version. On 2019.11.21 you
have `ReplicationLatency` and nothing else. See the monitoring section.

Note also the legacy version's inability to add a replica to a non-empty table.
That restriction is the entire reason Alex DeBrie's widely-cited 2019 migration
article (linked in [[dynamodb-table-naming-migration]]) is as elaborate as it
is. It no longer applies.

### Terraform resource mapping

| Global Tables version | Terraform resource |
|---|---|
| 2019.11.21 (Current) | `aws_dynamodb_table` with `replica` blocks — **use this** |
| 2019.11.21 (Current), alternative | `aws_dynamodb_table_replica` as a separate resource |
| 2017.11.29 (Legacy) | `aws_dynamodb_global_table` — legacy only, avoid |

The provider docs are explicit: `aws_dynamodb_global_table` *"manages DynamoDB
Global Tables V1 (version 2017.11.29)"* and directs you to the
`aws_dynamodb_table` `replica` block for V2. It is not formally deprecated with
a deprecation warning, but it is the wrong resource for anything new.

**`replica` block vs `aws_dynamodb_table_replica` — they are mutually
exclusive.** The provider states plainly: *"Do not use `replica` configuration
blocks of `aws_dynamodb_table` together with `aws_dynamodb_table_replica`."*

| | `replica` block | `aws_dynamodb_table_replica` |
|---|---|---|
| Where the replica lives in state | Inside the primary table resource | Its own resource, with its own `provider` alias |
| Provider alias for the standby | Not used — the block only takes `region_name` | Yes, `provider = aws.standby` |
| Fits a per-region-stack layout | Poorly | Well |
| Fits a single-stack-owns-the-global-table layout | Well | Adequately |
| Requires `lifecycle { ignore_changes = [replica] }` on the table | No | **Yes** |

**Recommendation: `replica` blocks.** They keep the global table as one
Terraform object, which matches how AWS models it (one `UpdateTable` call
against the source Region). It also matches the CloudFormation guidance —
*"you should choose one Region as the reference Region for deploying your
global tables and define all of your global table's replicas in that Region's
stack"* — and the same logic applies to Terraform state. `aws_dynamodb_table_replica`
is the right choice only if your repo structure genuinely cannot have one
stack own two Regions, which is the subject of
[[provider-aliases-vs-separate-stacks]].

### Upgrading from legacy to current

The upgrade is **online**: *"All global table replicas will continue to process
read and write traffic while upgrading."* It takes *"between a few minutes to
several hours depending on the table size and number of replicas."* Table
status goes `ACTIVE` → `UPDATING` → `ACTIVE`.

Requirements and traps:

- You need `dynamodb:UpdateGlobalTableversion` **in every Region with a
  replica** (note the lowercase `v` in the action name — it is not a typo in
  this note, it is how AWS spells it).
- *"Auto scaling will not adjust the provisioned capacity settings for a global
  table while the table is being upgraded. We strongly recommend that you set
  the table to on-demand capacity mode during the upgrade."* If you stay on
  provisioned, raise the autoscaling minimums first or you will throttle.
- `ReplicationLatency` *"can temporarily report latency spikes or stop
  reporting metric data during the upgrade process"* — so your replication-lag
  alarm will fire, or worse, go blind. Suppress it deliberately, don't discover
  it.
- Stream behaviour changes mid-flight: two records per write before, one after.
  **Any stream consumer must be idempotent before you start the upgrade**, not
  after.
- There is no Terraform resource for the upgrade. It is a console action or an
  `UpdateGlobalTableVersion` API call, after which you restructure the HCL from
  `aws_dynamodb_global_table` to `replica` blocks and reconcile state with
  `import`/`removed` (see [[dynamodb-table-naming-migration]]).

---

## Multi-active by design — and you asked for active/passive

This is the section to read twice.

DynamoDB global tables are marketed and engineered as **multi-active**: *"Any
global table replica can serve reads and writes."* There is **no configuration
flag that makes a replica read-only.** The standby table in `eu-west-2` will
accept a `PutItem` from anything with credentials and network access, and that
write will replicate back to `eu-west-1` and become the truth.

You get more than active/passive asked for, and the surplus is a liability.

### The stray-write failure mode, concretely

1. Someone runs a data-fix script and their `AWS_REGION` is `eu-west-2` because
   that is what was in their shell from last week's failover test.
2. The script writes item `ORDER#123` in the standby.
3. That write replicates to `eu-west-1` within a second.
4. Last-writer-wins compares item-level timestamps. The stray write is newer.
   **It wins.** The real order state is gone.
5. Nothing alarms. `ReplicationLatency` is fine. The write succeeded. There is
   no error anywhere. You find out from a customer.

The same shape occurs with: a Lambda whose environment variable region was
never updated; a CI job with a hard-coded region; a canary left running after a
DR game day; a read-only analytics job that turns out not to be read-only.

### Last-writer-wins and what it actually costs you

AWS describes MREC conflict resolution as *"last-writer-wins conflict
resolution strategy based on item-level timestamps."* Two properties matter:

- **It is item-level, not attribute-level.** A concurrent write to a *different
  attribute* of the same item still loses the whole item's other changes. This
  surprises people who expect a merge.
- **The losing write is discarded silently.** No error, no metric, no DLQ. The
  data is simply not there. There is no "conflicts detected" counter to alarm
  on.

For a genuinely active/passive workload — one region taking all writes — LWW
never fires, because there are no concurrent cross-region writes. That is
precisely why the discipline of *keeping* it active/passive matters: LWW is
harmless until the moment it isn't, and when it isn't, it is silent.

### Blocking writes to the standby with IAM

Yes, do it — and this is the genuinely tricky part, because a naive deny does
permanent damage.

DynamoDB replication is performed by a **service-linked role**,
`AWSServiceRoleForDynamoDBReplication`. If your deny policy catches that
principal:

> *"If you deny required SLR permissions, replication to and from affected
> replicas will stop, and the replica table status will change to
> `REPLICATION_NOT_AUTHORIZED`."*
>
> *"For multi-Region eventual consistency (MREC) global tables, if a replica
> remains in the `REPLICATION_NOT_AUTHORIZED` state for more than 20 hours, the
> replica is irreversibly converted to a single-Region DynamoDB table."*

**Irreversibly.** A misfired SCP on a Friday evening leaves you, on Monday,
with two unrelated single-region tables and no replication — and re-establishing
it means a full re-backfill. That is a 20-hour fuse on a mistake you cannot see
without looking at `ReplicaStatus`.

AWS publishes the exact guard condition. Attach it to any broad deny:

```json
"Condition": {
  "StringNotEquals": {
    "aws:PrincipalArn": [
      "arn:aws:iam::123456789012:role/aws-service-role/replication.dynamodb.amazonaws.com/AWSServiceRoleForDynamoDBReplication",
      "arn:aws:iam::123456789012:role/aws-service-role/dynamodb.application-autoscaling.amazonaws.com/AWSServiceRoleForApplicationAutoScaling_DynamoDBTable"
    ]
  }
}
```

The Application Auto Scaling SLR is in there for the same reason — deny it and
autoscaling stops, which given the capacity section below is its own outage.

A workable standby-write-fence, as an SCP on the account (not an identity
policy — identity policies get forgotten on new roles):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyDynamoDBWritesInStandbyRegion",
    "Effect": "Deny",
    "Action": [
      "dynamodb:PutItem",
      "dynamodb:UpdateItem",
      "dynamodb:DeleteItem",
      "dynamodb:BatchWriteItem",
      "dynamodb:TransactWriteItems",
      "dynamodb:ExecuteStatement",
      "dynamodb:ExecuteTransaction",
      "dynamodb:BatchExecuteStatement"
    ],
    "Resource": "*",
    "Condition": {
      "StringEquals": { "aws:RequestedRegion": "eu-west-2" },
      "StringNotEquals": {
        "aws:PrincipalArn": [
          "arn:aws:iam::123456789012:role/aws-service-role/replication.dynamodb.amazonaws.com/AWSServiceRoleForDynamoDBReplication",
          "arn:aws:iam::123456789012:role/aws-service-role/dynamodb.application-autoscaling.amazonaws.com/AWSServiceRoleForApplicationAutoScaling_DynamoDBTable",
          "arn:aws:iam::123456789012:role/FailoverBreakGlassRole"
        ]
      }
    }
  }]
}
```

Note `ExecuteStatement` / `ExecuteTransaction` / `BatchExecuteStatement` —
PartiQL writes are separate actions and a deny list that omits them is a deny
list with a hole in it.

**The critical design constraint: this fence must be removable in well under 15
minutes, by an on-call engineer, at 3am, possibly while the primary region's
control plane is impaired.** Options:

| Approach | Removal time | Risk |
|---|---|---|
| SCP edited by hand at failover | Minutes, if the right person is awake and has Org access | Human step in the critical path; Organizations is a control-plane dependency |
| SCP with a condition on an SSM parameter | N/A — IAM cannot read SSM | Doesn't work |
| Pre-provisioned `FailoverBreakGlassRole` excluded from the deny, which the failover automation assumes | **Seconds** — no policy change needed | The role exists permanently and is a standing write path |
| No fence; rely on process | Zero | Back to the stray-write failure mode |

**Recommendation: exclude a dedicated, heavily-audited `FailoverBreakGlassRole`
from the deny, and have the application in the standby region run as that
role.** The standby's compute is scaled to zero while the primary is healthy
(see [[aws-eks]]), so nothing is *using* the role; failing over means scaling up
pods that already have it. No IAM change is needed during the incident, so
IAM's eventual consistency and Organizations' availability are both off the
critical path — which is exactly the "rely on data plane, not control plane"
principle AWS states for failover. Alarm hard on any use of that role while the
primary is healthy, via CloudTrail. See [[aws-iam]] and
[[split-brain-and-fencing]].

> **`aws:RequestedRegion` caveat:** it is a global condition key evaluated
> against the endpoint's region, and it is reliable for DynamoDB's regional
> endpoints. It does *not* fence a cross-region call that DynamoDB makes on
> your behalf — but that is the SLR, which you have already excluded.

---

## Replication lag and monitoring

### `ReplicationLatency`

> *"The elapsed time between when an updated item appears in the DynamoDB
> stream for one replica table, and when that item appears in another replica
> in the global table. `ReplicationLatency` is expressed in milliseconds and is
> emitted for every source- and destination-Region pair."*

Properties that matter for alarm design:

- **Per source/destination Region pair.** With two replicas you get two
  directional metrics, and you should alarm on both. eu-west-1→eu-west-2 being
  healthy tells you nothing about eu-west-2→eu-west-1.
- **Latency is distance-dependent.** AWS's own example contrasts us-west-1→us-west-2
  with us-west-1→af-south-1. Do not copy a threshold from a blog; baseline your
  own pair for a fortnight first, then set the threshold off *your* p99.
- **Alarm on it dropping to zero as well as rising.** The prescriptive guidance
  says so explicitly: *"You might also watch for `ReplicationLatency` dropping
  to 0, which indicates stalled replication."* A missing-data alarm is the
  other half — the metric only emits when there are writes, so a
  `treat_missing_data` choice of `breaching` on a low-traffic table will page
  you all weekend. Use `notBreaching` for low-traffic tables and rely on a
  synthetic canary write instead.
- **MRSC does not publish it at all.** Another reason MRSC changes your
  monitoring story wholesale.

Published thresholds, both real, both from AWS, and usefully far apart:

| Source | Threshold | Notes |
|---|---|---|
| [AWS Prescriptive Guidance checklist](https://docs.aws.amazon.com/prescriptive-guidance/latest/dynamodb-global-tables/checklist.html) | *"generate an alert if the recent average exceeds 180,000 milliseconds"* | 3 minutes. A "something is badly wrong" alarm. |
| [AWS Database Blog, Global Tables best practices Part 1](https://aws.amazon.com/blogs/database/best-practices-for-amazon-dynamodb-global-tables-part-1-operational-readiness/) | Warning at 3,000 ms sustained 5 min; critical at 5,000 ms sustained 3 min | Seconds. A "we are degrading" alarm. |

They are not in conflict — they are different alarm tiers. Use both.

### RPO breach alarming

Your RPO is 2 hours = 7,200,000 ms. `ReplicationLatency` at 7.2M ms would be an
extraordinary event; the 180,000 ms alarm gives you a **40× margin** before RPO
is threatened. Practically:

- **Warning** at your baselined p99 × 3, or 3,000 ms, whichever is higher.
- **Critical** at 180,000 ms — this is the "RPO is now genuinely at risk"
  alarm and should page.
- **RPO breach** at 7,200,000 ms is not worth an alarm as such; if you are
  there, the 180,000 ms alarm fired two hours ago and was ignored.

The more useful RPO instrument is a **canary**: write a timestamped item to the
primary every 30 seconds, read it from the standby, and emit the observed
delta as a custom metric. That measures the thing you actually care about
(end-to-end staleness of the standby) rather than a service-internal latency,
and it keeps emitting on a table with no organic traffic. See
[[observability-multi-region]].

### `PendingReplicationCount` — does not exist on the current version

Worth repeating because it is a widely-repeated error. AWS:
*"version 2017.11.29 (Legacy) publishes the `PendingReplicationCount` metric.
version 2019.11.21 (Current) does not publish this metric."* If your monitoring
IaC references it on a current-version table, the alarm will sit in
`INSUFFICIENT_DATA` forever and everyone will assume it is fine.

### Chaos testing

Both MREC and MRSC integrate with **AWS Fault Injection Service**, which can
*"pause replication to and from a selected replica"* to simulate Region
isolation. That is the right way to prove your `ReplicationLatency` alarms
actually fire, and it belongs in [[dr-testing-and-gamedays]].

---

## Requirements to be replica-ready

| Requirement | Detail |
|---|---|
| **Identical table name** | *"All replicas in a global table share the same table name."* The blocker — see [[dynamodb-table-naming-migration]]. |
| **Identical primary key schema** | Partition key and sort key must match exactly. `ForceNew` in Terraform. |
| **Identical GSIs** | GSI names and key schemas must match. On the current version, creating a GSI in one Region auto-creates and backfills it in the others. |
| **Streams** | Enabled by default on all MREC replicas and **cannot be disabled** — streams *are* the replication mechanism. In Terraform, set `stream_enabled = true` and `stream_view_type = "NEW_AND_OLD_IMAGES"` explicitly so your config matches reality. |
| **PITR** | Not strictly required for a global table, but required for export, required for the migration, and required by AWS Backup's continuous backup. Turn it on per replica — the `replica` block has its own `point_in_time_recovery` argument. AWS notes PITR *"for one replica in a global table might be sufficient"* for DR, which is a real cost lever. |
| **Capacity mode** | Must be consistent in effect across replicas. On-demand write capacity is *"automatically synchronized across all replicas."* In provisioned mode, autoscaling settings sync, but read capacity can be overridden per replica via `ProvisionedThroughputOverride`. |
| **Autoscaling (provisioned only)** | *"When configuring a global table for provisioned capacity mode, auto scaling must be configured."* Not optional. |
| **KMS** | *"All replicas in a global table must be configured with the same type of KMS key (AWS owned key, AWS managed key, or Customer managed key)."* Same *type*, not the same key — each Region needs its own regional key, or a multi-region key. See [[aws-kms]] and [[kms-when-to-use-multi-region-keys]]. |
| **Throughput quotas must match** | *"The table-level write throughput limit must be the same in every Region that has a replica."* If you have raised it above the 40,000 default in the primary, raise it in the standby **first** or replica creation fails. |
| **Deletion protection** | Per-replica. *"You must enable deletion protection on each replica."* |
| **Tags** | Do not propagate by default. Terraform's `replica` block has `propagate_tags` (defaults to `false`) — set it `true`. |

---

## Adding a replica to an existing table

### In-place or forced replacement in Terraform?

**In-place.** In the provider schema, `replica` is a plain `TypeSet` with no
`ForceNew`, while `name`, `hash_key` and `range_key` all carry
`ForceNew: true`. So adding a `replica` block to an existing
`aws_dynamodb_table` produces an **update in place**, which maps onto a single
`UpdateTable` call. There is no destroy/recreate and no downtime.

Read the plan anyway. Specifically:

- If the same PR *also* changes `name` (because someone bundled the rename from
  [[dynamodb-table-naming-migration]] into the same change), the plan is a
  **destroy**, and the replica block is irrelevant because the table is being
  deleted. Never bundle these two changes.
- If autoscaling is in play, there is a known provider limitation: AWS requires
  autoscaling to be enabled *before* global tables, so Terraform *"cannot create
  a new DynamoDB table with both features enabled during a single apply."* This
  affects table *creation*, not adding a replica to an existing table — but it
  is why greenfield tables in this estate should be created on-demand and
  converted later, if at all.
- The apply will sit there. Adding a replica is asynchronous and the provider
  waits for `ACTIVE`. Set a generous `timeouts { update = "3h" }` or your CI
  job will time out mid-backfill and leave you reasoning about a half-applied
  state. (The backfill itself continues server-side regardless; it is only
  Terraform that gives up.)
- *"Some global tables operations (for example, adding a replica) are
  asynchronous, and require that the IAM Principal is valid until they
  complete."* A one-hour CI role session will expire during a large backfill.

### Backfill: how it works, how long, how much

> *"When you add an AWS Region to your table, DynamoDB begins populating a new
> replica using a snapshot of your existing table. You can continue writing to
> the originating region while DynamoDB builds the new replica, and DynamoDB
> replicates all in-flight updates automatically to the new replica."*

So it is online and safe. The constraints are quota and cost, not correctness.

**Quota — the hard limit:**

| Quota | Default | Adjustable |
|---|---|---|
| Backfilled data for new replicas, per account, per Region, per day | **10 TB** | Yes, via Support |
| Throughput per MREC table | 40,000 RCU/WCU (or request units) | Yes |
| Table-level write throughput limit | **Must match across all replica Regions** | Yes |

If you are adding replicas totalling more than 10 TB into one destination
Region in 24 hours, **raise the quota before you start**, not when the apply
fails. Same for the WCU quota if the table exceeds 40,000 WCU. And the mismatch
error is explicit and quotable:

> `Cannot create a replica of table 'example_table' in region 'example_region_A' because its exceeds your current account limit in region 'example_region_B'.`

**Time:** AWS does not publish a bytes-per-hour backfill rate, and I could find
no credible public benchmark. **No public data found** — treat any number you
see in a blog as that person's table, not yours. What you *can* do: run the
replica-add on your smallest table first and measure GB/hour on your own data,
then extrapolate. The 10 TB/day quota implies AWS expects at least that order
of magnitude to be achievable, which is a floor, not a promise.

**Cost:**

> *"When you add a new Region to a global table, DynamoDB bootstraps the new
> Region automatically and charges you as if it were a table restore, based on
> the GB size of the table. It also charges cross-Region data transfer fees."*

So: a restore-priced one-off on table size, plus inter-region data transfer
out, plus — from then on — **every write is an rWRU/rWCU charged in every
Region**. That steady-state doubling of write cost is the real number for the
cost model, not the one-off backfill. See [[cost-model]].

**Capacity trap during backfill:** *"the provisioned capacity settings of the
source Region — including auto scaling maximum capacity — are applied to the new
replica when it is created, regardless of the capacity values specified in your
CloudFormation template."* The same is true of Terraform, because it is AWS
behaviour, not a template behaviour. If the source's autoscaling max exceeds
the destination's table-level quota, replica creation fails with
`insufficient TableMaxReadCapacityUnits limits`. After creation you can adjust
the replica's *read* capacity independently, and that adjustment does not
propagate back.

---

## Capacity at failover — the direct RTO threat

**This is where a Global Tables DR plan actually fails**, and it fails at the
worst moment.

The failure mode: the standby replica has been sitting idle for months. Its
autoscaled provisioned capacity has ratcheted down to near its minimum, because
that is what autoscaling is for. You fail over. 100% of production traffic
arrives in one step. Autoscaling responds in **minutes**; throttling starts in
**seconds**. The AWS Database Blog is blunt about the mechanism: the provisioned
capacity *"can change at any time"* and reflects recent traffic, not failover
readiness, and *"auto scaling takes minutes to react, which is too slow for a
sudden failover surge."*

Your RTO is 15 minutes. A five-minute autoscaling ramp during which the
application is returning `ProvisionedThroughputExceededException` is not
"recovered" — it is an outage with the lights on.

### Mitigation 1 — on-demand capacity mode

The blog's recommendation for planned events: *"This gives all replicas
identical capacity behavior and removes the need to manage auto scaling across
Regions during a high-stress incident."*

On-demand absorbs step changes without a scaling policy. It is the single
biggest RTO risk reduction available here, and it removes an entire class of
cross-region configuration drift.

**But on-demand is not infinitely instant.** A table has a *warm throughput*
value — the throughput it can serve immediately — and for a **new on-demand
table that value is 4,000 writes/sec and 12,000 reads/sec**. Beyond that,
on-demand scales, but it scales; it does not teleport. If your production
traffic exceeds those figures, on-demand alone still throttles you at failover.

**Cost:** on-demand is roughly 5–7× the per-request cost of fully-utilised
provisioned capacity at published `us-east-1` rates ($0.625 per million WRU /
$0.125 per million RRU on-demand, vs $0.00065/WCU-hour and $0.00013/RCU-hour
provisioned). Whether that is expensive depends entirely on your utilisation:
a table provisioned for peak and running at 15% average utilisation is already
paying more than on-demand would. **Measure utilisation before assuming
provisioned is cheaper.** Verify current rates at
[DynamoDB pricing](https://aws.amazon.com/dynamodb/pricing/on-demand/).

### Mitigation 2 — warm throughput (this does exist, and it is the right tool)

Launched November 2024, available in all commercial Regions. Two parts:

- A **warm throughput value** on every table and index, visible for free, that
  tells you *"the minimum throughput that your table is prepared to handle
  instantaneously."*
- A **pre-warm** operation that raises it. Asynchronous and non-blocking; you
  can do other table updates concurrently.

And, decisively for this project:

> Warm throughput is *"fully compatible with DynamoDB Global Tables"* and
> *"requests to update warm throughput value are automatically synchronized
> across all replicas for both read and write operations."*

So pre-warming the primary pre-warms the standby. That is the missing piece
that turns "on-demand standby" into "on-demand standby that can actually take
the traffic in second one".

**Cost:** a **one-time** charge on the *difference* between the new and current
warm throughput, priced at provisioned rates — $0.00065 per WCU and $0.00013
per RCU in `us-east-1`. Pre-warming to 20,000 WCU / 60,000 RCU is therefore a
one-off of roughly $13 + $7.80 ≈ $21. **That is nothing**, and it directly
addresses the largest RTO risk in this note. Do it.

Duration: *"depends on the requested warm throughput values and the storage
size of the table or index"* — AWS gives no figure. Pre-warm as part of the
standby build-out, not at failover time.

**Terraform support:** `aws_dynamodb_table` exposes `warm_throughput`
(*"Sets the number of warm read and write units for the specified table"*) and
`on_demand_throughput` (*"Sets the maximum number of read and write units for
the specified on-demand table"*). Both are first-class arguments, so this is
declarative, not a runbook step.

### Mitigation 3 — if you must stay provisioned

Raise the **autoscaling minimum** in the standby to match the primary's current
provisioned capacity, permanently. You then pay for idle capacity in the
standby at all times — which is the honest price of provisioned-mode warm
standby, and should be in [[cost-model]] as such. Note that autoscaling
settings *synchronise across replicas* on the current version, so per-replica
divergence is limited to read capacity via `ProvisionedThroughputOverride`.

**Recommendation: on-demand plus pre-warmed warm throughput.** It is the
lowest-operations answer, it eliminates cross-region autoscaling drift, and the
pre-warm cost is a rounding error. Revisit only if a cost review shows a
specific table with high, flat, predictable throughput where provisioned wins
decisively — and then carry the idle-capacity cost in the standby deliberately.

---

## Streams and Lambda: the double-processing bug

**Every replica produces its own stream, containing every write regardless of
where it originated.**

> *"Each global table produces an independent stream based on all its write
> operations, wherever they started from."*

So a Lambda trigger deployed in both regions — which is exactly what
"prerequisites first, mirror everything" produces — processes every change
**twice**. If that Lambda sends an email, charges a card, or increments a
counter in another system, you have shipped a duplicate-side-effects bug into
production, and it will not appear in any pre-production environment that only
exists in one region.

It is worse than a simple duplicate: replica streams are not identical.
*"The MREC replication process might combine multiple changes in a short period
of time into a single replicated write, resulting in each replica's Stream
containing slightly different records."* Ordering is guaranteed per item, but
*"the relative ordering of changes to different items might vary across
replicas."* So you cannot even dedupe by sequence number across regions.

### How to make a consumer run in one region only

Four approaches, worst to best:

**1. Don't deploy the consumer in the standby.** Simplest, and wrong for this
project — the whole prerequisites-first strategy is about having things
pre-provisioned. A Lambda that has to be created at failover time is a
control-plane operation in your 15-minute budget.

**2. Deploy the Lambda but disable the event source mapping.** The function,
its role, its layers, its VPC config, its concurrency reservation all exist
warm; only the ESM is disabled. Enabling an ESM is one API call, and it is
fast. In Terraform: `aws_lambda_event_source_mapping` with `enabled = false`,
flipped by the failover automation.

Caveat: an ESM starting from `LATEST` skips everything buffered while it was
off — which is what you want for "run only in the active region", but means the
standby consumer does **not** catch up on the backlog. Starting from
`TRIM_HORIZON` means replaying up to 24 hours. Neither is obviously right;
decide per consumer and write it down.

**3. AWS's documented pattern — a region attribute plus a Lambda event
filter.** AWS states it directly:

> *"If you want to process local but not replicated write operations, you can
> add your own `Regionattribute` to each item to identify the writing Region.
> You can then use a Lambda event filter to call the Lambda function only for
> write operations in the local Region. This helps with insert and update
> operations, but not delete operations."*

The application stamps `writeRegion: "eu-west-1"` on every item; each region's
ESM filters for its own value. Both consumers stay enabled permanently; only
one ever fires, and it follows the writes automatically at failover with **zero**
control-plane action. This is the best fit for a 15-minute RTO.

```hcl
resource "aws_lambda_event_source_mapping" "orders_stream" {
  provider          = aws.standby
  event_source_arn  = aws_dynamodb_table.orders.replica[0].stream_arn
  function_name     = aws_lambda_function.orders_processor.arn
  starting_position = "LATEST"

  filter_criteria {
    filter {
      # Only process writes that originated in THIS region.
      pattern = jsonencode({
        dynamodb = {
          NewImage = {
            writeRegion = { S = ["eu-west-2"] }
          }
        }
      })
    }
  }
}
```

**The delete gap is real and AWS says so.** A `REMOVE` event has no `NewImage`,
so the filter cannot match on it. Options: use `OldImage.writeRegion` in a
second filter (works if the item carried the attribute, but `OldImage`
reflects the *original writer*, not the deleter — so a deletion in eu-west-1 of
an item originally written in eu-west-2 is misattributed); or use soft deletes
(a `deleted: true` flag plus TTL) so deletes are updates and carry a fresh
`writeRegion`; or handle deletes idempotently and accept double-processing for
that one event type.

**Recommendation: soft deletes where the consumer's delete handling has side
effects; idempotency everywhere else.** Note that TTL-driven hard deletes are
also replicated on the current version and will appear in both streams.

**4. A distributed lock / leader election** so only one consumer processes.
More moving parts than the problem deserves. Skip it unless (3) genuinely
cannot work.

### Regardless of which you choose: be idempotent

Lambda's DynamoDB Streams integration is **at-least-once**, so a single
consumer in a single region already sees occasional duplicates. Idempotency is
not a global-tables tax; it is table stakes that global tables make
unavoidable. See [[aws-lambda]] and [[messaging-in-flight-data-loss]].

Also note the shard-reader quota: *"For global tables, we recommend you limit
the number of simultaneous readers to one to avoid request throttling."* Half
the normal allowance. If you have a Lambda *and* a Kinesis adapter *and* an
analytics consumer on the same stream, that is already too many.

---

## `TransactWriteItems` is region-local

Quoting AWS directly, because this one gets argued about:

> *"On a global table configured for MREC, DynamoDB transaction operations
> (`TransactWriteItems` and `TransactGetItems`) are only atomic within the
> Region where the operation was invoked. Transactional writes are not
> replicated as a unit across Regions, meaning only some of the writes in a
> transaction might be returned by read operations in other replicas at a given
> point in time."*

And the worked example: a `TransactWriteItems` in us-east-2 *"might observe
partially completed transactions in the US West (Oregon) Region as changes are
replicated."*

Why it surprises people: transactions are sold as an atomicity guarantee, and
replication is sold as transparent. Compose them and you assume atomicity
replicates. It does not. The transaction commits atomically in its own Region,
then its constituent writes replicate as **independent items**, arriving in an
arbitrary order.

**What breaks in practice:**

- A standby-region read that sees a debit without the matching credit.
- Any invariant enforced by a transaction ("these two items are always
  consistent") is **not** enforced in the standby, at any instant.
- A stream consumer in another region that reconstructs transactions from
  stream records — it cannot; the grouping is gone.

**For an active/passive posture this is mostly benign**, because nothing reads
the standby while the primary is healthy. It becomes real in exactly two
moments: (1) at failover, where the standby may be mid-transaction-tear for a
sub-second window — covered by your RPO of 2h with enormous margin, but worth
knowing the shape of; and (2) if anyone ever adds a "read from the nearest
region" optimisation, which converts a benign property into a correctness bug.
Write that constraint into [[open-decisions]] before someone discovers it as a
performance idea.

And note again: **MRSC does not support transactions at all** — they return an
error. A codebase using `TransactWriteItems` cannot move to MRSC without being
rewritten, which quietly closes off that option for good.

---

## PITR + cross-region restore: the cheaper alternative

Given RPO is only 2 hours, it is worth asking whether global tables are
necessary at all. They are not the only option.

**How it works:** PITR maintains continuous backups. You can *"restore your
DynamoDB table data across AWS Regions such that the restored table is created
in a different Region from where the source table resides."* The restore target
is always a **new table**. Cross-Region restore works between commercial
Regions (with documented exclusions — Asia Pacific (Hong Kong) and Middle East
(Bahrain) — neither of which is in scope here). The CLI requires
`--source-table-arn` and the default region set to the destination.

| | Global Tables | PITR cross-region restore |
|---|---|---|
| RPO | Sub-second | Up to your restore granularity; PITR is continuous, so effectively minutes |
| **RTO** | **~0 — table is already live** | **Hours. Restores are not fast, and a 50 TB concurrent-restore quota applies.** |
| Standby storage cost | Full second copy | PITR at $0.20/GB-month, no second table |
| Steady-state write cost | **Every write billed in both Regions** | Single-region writes |
| Cross-region data transfer | Yes, continuous | Only at restore |
| Ops complexity | Low (managed) | Restore runbook, plus everything the restore doesn't recreate |
| Meets RTO 15m? | **Yes** | **No** |

**Verdict: PITR cross-region restore fails the RTO target, decisively.** A
restore of any meaningful table takes far longer than 15 minutes, and the
restored table arrives without autoscaling policies, without CloudWatch alarms,
without IAM policies attached, and without tags — so even after the data lands,
there is reassembly work. It is a *backup* strategy, not a *failover* strategy.

**Where it does belong:** as the second line of defence against the failure
mode global tables cannot help with — **logical corruption**. A bad deploy that
writes garbage replicates the garbage to every replica in under a second.
Global tables are not a backup. Keep PITR on (AWS: enabling it on *one* replica
*"might be sufficient to meet your disaster recovery objectives"*, which is a
genuine cost lever) and keep AWS Backup cross-region copies for the regulatory
retention case. See [[aws-backup]].

So: **both**, for different threats. Global tables for regional failure, PITR
for "we deleted the wrong thing".

---

## RPO / RTO analysis

**RPO 2h: met with enormous margin.** Replication is typically sub-second. Even
a sustained degradation triggering the 180,000 ms critical alarm leaves you at
3 minutes of exposure — 40× inside budget. The only realistic path to breaching
a 2-hour RPO is a replica stuck in `REPLICATION_NOT_AUTHORIZED` or
`INACCESSIBLE_ENCRYPTION_CREDENTIALS`, both of which are configuration
failures, not capacity failures, and both of which you alarm on by watching
`ReplicaStatus` rather than `ReplicationLatency`.

**RTO 15m: met, conditionally.** Where the time goes:

| Step | Time | Pre-provisioned? |
|---|---|---|
| DynamoDB replica becomes writable | **0 s** — it always was | Yes |
| Application in standby starts pointing at the local endpoint | Seconds, if the region is resolved from config/env at startup | Depends on [[eks-workload-delivery]] |
| Removing the standby write-fence | **0 s** with a pre-excluded break-glass role; minutes if it needs an SCP edit | Design choice, above |
| Capacity absorbing 100% of traffic | **0 s if on-demand and pre-warmed; 2–10 min of throttling otherwise** | **This is the whole risk** |
| Enabling stream consumers | 0 s with region-attribute filters; seconds to enable an ESM | Design choice, above |
| Everything else (DNS, EKS, ALB) | Dominates | See [[failover-orchestration]] |

DynamoDB itself contributes essentially **zero** to RTO. Every minute at risk
comes from capacity warmth and from fences you put in place yourself. Both are
solvable at build time, not at incident time — which is the correct place to
solve them.

---

## Warm standby shape

What exists in `eu-west-2` while `eu-west-1` is healthy:

| Thing | State | Costs money? |
|---|---|---|
| Replica table | **ACTIVE, fully populated, readable and writable** | Storage ($0.25/GB-month standard) + the rWRU on every primary write |
| Table data | Complete, sub-second stale | Yes — full second copy |
| Stream | Enabled, cannot be disabled | Stream read requests only if consumed |
| Lambda consumers | Deployed, filtered to no-op (or ESM disabled) | Near zero |
| PITR | Optional per replica | $0.20/GB-month if enabled |
| Warm throughput | Pre-warmed to primary's peak | One-off only |
| Write fence | SCP denying writes except break-glass | Free |
| Alarms/dashboards | Deployed and active | Cents |

**Nothing here is scaled to zero and nothing can be.** A DynamoDB replica is
either there and complete, or it is not there. That is the cost floor, and it
is why the cost conversation for DynamoDB is about the *steady-state write
doubling*, not about the standby's idle footprint.

---

## Terraform implementation

### Provider aliases

```hcl
provider "aws" {
  alias  = "primary"
  region = var.primary_region     # eu-west-1
  default_tags { tags = local.common_tags }
}

provider "aws" {
  alias  = "standby"
  region = var.standby_region     # eu-west-2
  default_tags { tags = local.common_tags }
}
```

Note that for `replica` blocks you barely use `aws.standby` for the table
itself — the replica is created by an `UpdateTable` against the primary. You
need the alias for everything *around* the table: alarms, the stream consumer's
ESM, the break-glass role.

### The module

```hcl
# modules/dynamodb-global-table/main.tf

resource "aws_dynamodb_table" "this" {
  provider = aws.primary

  name         = var.name          # pair-scoped, no region token
  billing_mode = var.billing_mode  # PAY_PER_REQUEST
  hash_key     = var.hash_key
  range_key    = var.range_key

  dynamic "attribute" {
    for_each = var.attributes
    content {
      name = attribute.value.name
      type = attribute.value.type
    }
  }

  dynamic "global_secondary_index" {
    for_each = var.global_secondary_indexes
    content {
      name            = global_secondary_index.value.name
      hash_key        = global_secondary_index.value.hash_key
      range_key       = try(global_secondary_index.value.range_key, null)
      projection_type = global_secondary_index.value.projection_type
    }
  }

  # Required for MREC replication; also what stream consumers read.
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  point_in_time_recovery { enabled = true }
  deletion_protection_enabled = true

  server_side_encryption {
    enabled     = true
    kms_key_arn = var.primary_kms_key_arn   # see [[aws-kms]]
  }

  dynamic "ttl" {
    for_each = var.ttl_attribute == null ? [] : [1]
    content {
      attribute_name = var.ttl_attribute
      enabled        = true
    }
  }

  # Pre-warm so a failover does not meet a cold table. One-off cost.
  dynamic "warm_throughput" {
    for_each = var.warm_throughput == null ? [] : [var.warm_throughput]
    content {
      read_units_per_second  = warm_throughput.value.read
      write_units_per_second = warm_throughput.value.write
    }
  }

  # THE replica. Adding this block is an in-place update, not a replacement.
  dynamic "replica" {
    for_each = var.replica_regions
    content {
      region_name            = replica.value.region
      kms_key_arn            = replica.value.kms_key_arn   # regional key
      point_in_time_recovery = replica.value.pitr
      propagate_tags         = true                        # default is false
      # consistency_mode intentionally omitted -> EVENTUAL (MREC).
    }
  }

  timeouts {
    # Backfilling a large table takes hours. Do not let CI give up mid-apply.
    create = "3h"
    update = "3h"
    delete = "1h"
  }

  lifecycle {
    prevent_destroy = true
    # Only if provisioned + autoscaling; harmless to omit on PAY_PER_REQUEST.
    ignore_changes = [read_capacity, write_capacity]
  }
}
```

### Variable surface worth exposing

```hcl
variable "name" {
  description = "Pair-scoped table name. Must NOT contain a region token."
  type        = string
  validation {
    condition     = !can(regex("(eu|us|ca|ap|sa|me|af)-[a-z]+-[0-9]", var.name))
    error_message = "Global Tables replicas share one name; remove the region from the table name."
  }
}

variable "replica_regions" {
  description = "Replica regions. Empty list produces a plain single-region table."
  type = list(object({
    region      = string
    kms_key_arn = optional(string)
    pitr        = optional(bool, true)
  }))
  default = []
}

variable "billing_mode" {
  type    = string
  default = "PAY_PER_REQUEST"
  validation {
    condition     = contains(["PAY_PER_REQUEST", "PROVISIONED"], var.billing_mode)
    error_message = "billing_mode must be PAY_PER_REQUEST or PROVISIONED."
  }
}

variable "warm_throughput" {
  description = "Pre-warm target. Set to the primary's peak. Synced to all replicas."
  type        = object({ read = number, write = number })
  default     = null
}
```

The point of `replica_regions` defaulting to `[]` is that **the same module
serves single-region and multi-region tables**, which is what a cookiecutter
monorepo needs. Going multi-region for a given environment is a one-line change
to a `.tfvars`, not a different module. That is the shape of change the brief
asks for: evolve the templating, don't replace it.

### Where to put the alarms

```hcl
resource "aws_cloudwatch_metric_alarm" "replication_latency_critical" {
  provider = aws.primary
  for_each = toset(var.replica_regions[*].region)

  alarm_name          = "${var.name}-replication-latency-to-${each.value}"
  namespace           = "AWS/DynamoDB"
  metric_name         = "ReplicationLatency"
  statistic           = "Average"
  period              = 60
  evaluation_periods  = 3
  comparison_operator = "GreaterThanThreshold"
  threshold           = 180000   # 3 min. AWS Prescriptive Guidance figure.

  dimensions = {
    TableName      = aws_dynamodb_table.this.name
    ReceivingRegion = each.value
  }

  # Metric only emits when there are writes. Breaching on missing data
  # will page you every quiet Sunday.
  treat_missing_data = "notBreaching"
  alarm_actions      = [var.alarm_topic_arn]
}
```

Verify the dimension name (`ReceivingRegion`) against
[DynamoDB metrics and dimensions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/metrics-dimensions.html)
before shipping — dimension naming for this metric is easy to get wrong and the
alarm fails silently into `INSUFFICIENT_DATA` if you do.

Alarms belong in the **primary**'s provider for the outbound direction and the
standby's for the inbound; deploy both. See [[observability-multi-region]] for
why the alarm itself must also exist in a region that is not the one failing.

---

## Migration path from single-region

Assuming the naming work in [[dynamodb-table-naming-migration]] is done:

1. **Pre-flight the quotas.** Table-level write throughput limit must match in
   both Regions. Backfill volume under 10 TB/day per destination Region, or
   quota raised. Do this days ahead — Support tickets are not a 15-minute
   operation either.
2. **Create the KMS key in the standby** (or decide on a multi-region key) and
   confirm the *type* matches the primary's. See
   [[kms-when-to-use-multi-region-keys]].
3. **Confirm `stream_enabled = true`** in the config. It is about to become
   true whether you declare it or not; declaring it prevents a confusing diff.
4. **Add the `replica` block.** `terraform plan` must show
   `~ update in-place`. **Any `# forces replacement` is a stop-the-line
   event** — go back and find what else changed in the PR.
5. **Apply with a long timeout.** Watch `ReplicaStatus` go
   `CREATING` → `ACTIVE`. Watch `ConsumedWriteCapacityUnits` in the primary
   for backfill-induced read pressure.
6. **Pre-warm** to the primary's observed peak. One API call, one-off cost,
   synced to the replica automatically.
7. **Apply the write fence** (SCP) and immediately test it: assume a normal
   application role, attempt a `PutItem` in the standby, confirm
   `AccessDenied`. Then assume the break-glass role and confirm it *succeeds*.
   A fence you have not tested from both sides is a fence you do not have.
8. **Check `ReplicaStatus` is not `REPLICATION_NOT_AUTHORIZED`** after the
   fence lands. You have 20 hours before the damage is irreversible; check
   within 20 minutes.
9. **Deploy stream consumers in the standby** with region-attribute filters,
   enabled. Verify they do not fire.
10. **Deploy alarms in both Regions.** Baseline `ReplicationLatency` for two
    weeks before setting the warning threshold.
11. **Run an FIS region-isolation experiment** to prove the alarms fire and the
    application degrades the way you think it does.

**Nothing in this sequence requires downtime, and nothing forces replacement.**

---

## Failover procedure

The DynamoDB-specific steps, which are mercifully few. The full picture is
[[failover-runbook-template]].

1. **Decide.** Human. DynamoDB will not tell you to fail over — the replica is
   healthy by definition.
2. **Check replication drained.** Read `ReplicationLatency` primary→standby for
   the last 15 minutes. If the primary is truly gone, this metric is your only
   estimate of how much data is in flight and will be lost. Record it — it is
   your actual RPO for this incident, and it goes in the post-incident report.
3. **Do not remove the replica.** The instinct to "promote" is wrong and
   dangerous. There is no promotion. Removing the replica from the global table
   is a control-plane operation that permanently severs replication and makes
   failback a full re-backfill. The standby is already writable. Do nothing to
   the table.
4. **Release the write fence** — zero-touch if you used the break-glass role
   pattern.
5. **Shift traffic** (Route 53 / ARC). See [[aws-route53]] and
   [[failover-orchestration]].
6. **Watch for throttling** in the standby: `ThrottledRequests`,
   `ReadThrottleEvents`, `WriteThrottleEvents`. If they appear despite
   on-demand and pre-warming, your peak exceeded the warm value and you need to
   pre-warm higher next time. Record the number.
7. **Expect stale reads.** AWS Part 2: *"your application might encounter stale
   reads post-failover"*; design operations to be idempotent and carry version
   attributes.
8. **Expect elevated `ReplicationLatency`** once the old primary returns, as
   the backlog drains standby→primary. This is normal and your alarm will fire.
   Do not panic-remove the replica.

AWS's own guidance, worth internalising: rely on **data plane** operations
during failover, not control plane, *"because some control plane operations
might be degraded during Region failures."* Every step above except "release
the fence" is data plane, and that step is data plane too if you used the
break-glass role.

### Failover strategy options

Part 2 of the AWS best-practices series names two:

| | Route 53 ARC (Region switch + routing controls) | Route 53 DNS failover + health checks |
|---|---|---|
| RTO | *"Seconds to low minutes"* | *"Several minutes"* (alarm evaluation + DNS propagation) |
| Complexity | High upfront | Low |
| Safety rules | Yes — prevents dangerous state changes | No |
| Recommended for | Mission-critical | Simpler architectures |

Given a **15-minute RTO that must hold end-to-end across EKS, ALB, RDS and
DynamoDB**, "several minutes" of DNS propagation eats a large fraction of the
budget before anything else moves. **Recommendation: ARC.** Cost it in
[[cost-model]] and design it in [[failover-orchestration]] — this is a
cross-cutting decision, not a DynamoDB one, but DynamoDB is the service where
ARC's routing controls are easiest to reason about because the data layer needs
no action at all.

---

## Failback

Structurally easy, which is unusual and worth appreciating.

Because the global table is multi-active, there is no "reverse the replication
direction" step. The old primary is **still a replica**. While you were serving
from the standby, every write replicated back to it (assuming the region
recovered and the SLR was never denied). It is current.

Failback is therefore: **re-apply the fence to the standby, release it on the
primary, shift traffic back.** That is it.

The traps:

- **Do not remove and re-add the replica.** If someone "cleaned up" during the
  incident by removing the failed region's replica, failback is a full
  re-backfill — hours, plus the restore-priced bootstrap charge again.
- **The 20-hour clock.** If the primary Region was *disabled* (not merely
  impaired), *"those replicas are permanently converted to single-Region tables
  20 hours after the Region is disabled."* Likewise `REPLICATION_NOT_AUTHORIZED`
  and `INACCESSIBLE_ENCRYPTION_CREDENTIALS` both irreversibly convert the
  replica after 20 hours on MREC. **During a long incident, someone must be
  watching `ReplicaStatus`.** Put it in the incident checklist, because at hour
  18 of an outage nobody is thinking about a replica state field.
- **Writes made in the standby during the incident win LWW on failback** — which
  is correct, since they are newer. But if the old primary had in-flight writes
  that never replicated out before it failed, and the standby subsequently wrote
  the same items, those primary writes are **lost on reconciliation**. That is
  your RPO being spent, exactly as designed, but it is worth stating plainly so
  nobody expects a merge.
- **Verify before shifting back.** `ReplicationLatency` standby→primary at
  baseline, backlog drained, `ReplicaStatus` `ACTIVE` on both.

See [[failback]].

---

## Gotchas

- **`PendingReplicationCount` does not exist on the current version.** Any
  alarm on it sits in `INSUFFICIENT_DATA` forever, looking healthy.
- **The 20-hour irreversibility clock** applies to `REPLICATION_NOT_AUTHORIZED`
  (denied SLR) *and* `INACCESSIBLE_ENCRYPTION_CREDENTIALS` (revoked CMK) on
  MREC. Seven days on MRSC, with a worse outcome. Monitor `ReplicaStatus`, not
  just latency.
- **Denying the replication SLR breaks replication permanently.** Always attach
  the `aws:PrincipalArn` `StringNotEquals` guard to broad denies. Same for the
  Application Auto Scaling SLR.
- **KMS: all replicas must use the same *type* of key**, and DynamoDB needs key
  access *to delete a replica* — so delete the replica before you delete the
  key, never the other way round.
- **Table-level write throughput quotas must match across Regions** or replica
  creation fails with a cross-region limit error.
- **10 TB/account/Region/day backfill quota** on adding replicas.
- **Source Region's capacity settings, including autoscaling maximum, are
  applied to the new replica on creation**, overriding your template, and can
  fail with `insufficient TableMaxReadCapacityUnits limits`.
- **Tags do not propagate by default.** Set `propagate_tags = true` or your
  standby table escapes every cost allocation and tag-based backup selection.
- **On-demand's initial warm throughput is 4,000 writes/sec and 12,000
  reads/sec.** "On-demand scales automatically" is true and insufficient.
- **You cannot delete a table used to seed a replica for 24 hours.**
- **DAX is invisible to replication:** *"Global tables bypass DAX by updating
  DynamoDB directly, so DAX isn't aware that it's holding stale data."* If DAX
  is in use anywhere, a replicated write leaves a stale DAX entry until TTL.
- **Global tables are not a backup.** A bad deploy's corrupt writes replicate
  in under a second. Keep PITR and AWS Backup.
- **Stream shard reader limit is one for global tables**, versus two for
  single-region tables.
- **Transactions do not replicate atomically** — see above.
- **MRSC is a one-way door:** consistency mode cannot be changed after
  creation, and MRSC forbids transactions and TTL entirely.
- **Terraform applies can outlive CI role sessions** during a large backfill.
- **`aws_dynamodb_global_table` is for legacy 2017.11.29 only.** Reaching for
  it because the name sounds right is an easy and costly mistake.

---

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Consistency mode | MREC (eventual) | MRSC (strong, zero RPO) | **MREC.** RPO 2h does not need zero RPO, and MRSC costs transactions, TTL, a third Region, and cannot serve the CA pair at all. |
| Terraform resource | `replica` blocks on `aws_dynamodb_table` | `aws_dynamodb_table_replica` | **`replica` blocks.** One object, matches AWS's own model and CloudFormation guidance. Revisit only if [[provider-aliases-vs-separate-stacks]] forces per-region stacks. |
| Capacity mode | On-demand | Provisioned + autoscaling | **On-demand**, plus pre-warmed warm throughput. Removes the largest RTO risk for ~$20 one-off. Revisit per-table if utilisation is high and flat. |
| Standby writes | IAM/SCP deny fence | Rely on process and routing | **Fence**, with a pre-excluded break-glass role so removal is zero-touch at failover. |
| Stream consumers in standby | `writeRegion` attribute + Lambda event filters | Disabled ESM, enabled at failover | **Filters.** Zero control-plane action at failover, which is what a 15-minute RTO needs. Handle deletes via soft-delete or idempotency. |
| PITR | Every replica | One replica only | **One replica** (the primary) as the default — AWS says it *"might be sufficient"* — and both for tables where a regional loss during a corruption incident is intolerable. Real cost lever at $0.20/GB-month. |
| Backup strategy | Global tables only | Global tables + PITR + AWS Backup cross-region copy | **All three.** They defend different threats: regional failure, logical corruption, regulatory retention. |
| Failover orchestration | Route 53 ARC | DNS health checks | **ARC**, on RTO grounds. Cross-cutting — see [[failover-orchestration]]. |

---

## Cost

Per table, steady state, standby idle. Published `us-east-1` rates — **verify
for eu-west-1 / eu-west-2 / ca-central-1 / ca-west-1 before budgeting**, they
differ.

| Line | Rate | Note |
|---|---|---|
| Storage, both Regions | $0.25/GB-month each | You now pay twice for the same data |
| Writes | rWRU billed **in every Region** the item is written to or replicated to | **Effectively 2× write cost.** The dominant line. |
| Reads in standby | ~0 while passive | The standby serves no reads |
| Cross-region data transfer | Per-GB, continuous | Proportional to write volume, not table size |
| PITR | $0.20/GB-month per replica where enabled | Enable on one to halve it |
| Warm throughput pre-warm | $0.00065/WCU + $0.00013/RCU, **one-off** | ~$21 to pre-warm 20k WCU / 60k RCU |
| Replica bootstrap | Priced "as if it were a table restore," on table size | One-off, plus transfer |

**The levers, in order of size:**

1. **Which tables actually need a replica.** Not all of them do. A table
   holding cached derived data, or one rebuildable from an event log, does not
   need to double its write cost. Go table by table; the default should be *no*
   replica until someone argues for one.
2. **Capacity mode**, once you have measured real utilisation.
3. **PITR on one replica, not both.**
4. **`STANDARD_INFREQUENT_ACCESS` table class** for large, cold tables — the
   `replica` block supports a per-replica table class override.

Note there is **no reserved capacity for rWCUs or rWRUs**: *"There is no
reserved capacity available for rWCUs or rWRUs at this time."* So the usual
commit-discount lever is unavailable for the replicated write cost, which is
the largest line. Budget accordingly.

---

## `ca-west-1` (Calgary) — parity check

- **DynamoDB is available in `ca-west-1`.** Confirmed by the region launch
  announcement (DynamoDB is in the core service set present at every region
  launch).
- **Global tables state they are available in all Regions where DynamoDB is
  available**, which includes `ca-west-1`. I did **not** find a `ca-west-1`-specific
  global tables announcement to corroborate this beyond the general statement —
  **verify with a throwaway table before committing the CA pair's design.** It
  is a ten-minute check and it removes an assumption.
- **MRSC is not available for the CA pair.** The published Region sets are US,
  EU and AP only. If strong consistency ever becomes a requirement, the CA pair
  cannot satisfy it without a witness outside Canada, which has
  [[data-residency]] consequences.
- **Cross-Region restore exclusions** (Hong Kong, Bahrain) do not affect
  `ca-west-1`.

Feed these into [[region-pair-selection]].

---

## Open questions

1. **Are there any existing 2017.11.29 legacy global tables in the estate?** If
   yes, upgrading them is a separate workstream with its own stream-behaviour
   change, and must happen before the naming migration lands.
2. **Which tables genuinely need a replica?** The default answer should be
   "none until argued". Doubling the write bill for a rebuildable cache is a
   pure loss.
3. **Does any table use `TransactWriteItems`?** If so, MRSC is permanently off
   the table for that table, and any future "read from the nearest region"
   optimisation is a correctness bug waiting to happen.
4. **Do any stream consumers have non-idempotent side effects?** Answer before
   deploying a consumer into the standby, not after.
5. **Is DAX in use anywhere?** It silently holds stale data on replicated
   writes.
6. **What is the current peak WCU/RCU per table?** Needed to set the warm
   throughput target, and unknowable at 3am.
7. **Who can edit the SCP at 3am, and is the break-glass role acceptable to
   security?** The answer determines whether the fence costs you minutes of RTO.
8. **Is `ReplicaStatus` monitored anywhere today?** The 20-hour irreversibility
   clock is the sharpest edge in this note and nothing else surfaces it.
9. **RTO definition** — 15 minutes from incident start, or from decision to
   fail over? Flagged in [[CLAUDE]] as unresolved estate-wide. For DynamoDB it
   barely matters (the data layer contributes ~0 either way), but it changes
   the acceptable throttling window materially.

---

## Sources

- [Global tables — multi-active, multi-Region replication](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html) — the multi-active framing, MREC/MRSC modes, same-account vs multi-account, immutability of the consistency mode.
- [Global tables core concepts](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/globaltables-CoreConcepts.html) — the same-name requirement, 99.999% SLA, TTL and streams behaviour, the transactions-are-region-local statement with worked example, throughput sync rules, `ReplicationLatency` description, the 24-hour and 20-hour management rules, and FIS integration.
- [DynamoDB global tables versions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/V2globaltables_versions.html) — the full legacy-vs-current comparison, the `PendingReplicationCount` removal, version detection commands, and the complete upgrade procedure including the on-demand recommendation and stream-behaviour table.
- [DynamoDB global tables security](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/globaltables-security.html) — `AWSServiceRoleForDynamoDBReplication`, the `REPLICATION_NOT_AUTHORIZED` 20-hour irreversibility, AWS's own SLR-exclusion condition block, the KMS same-key-type rule and `INACCESSIBLE_ENCRYPTION_CREDENTIALS`, and per-operation IAM permission lists.
- [Quotas in Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/ServiceQuotas.html) — the 10 TB/day replica backfill quota, the matching-write-throughput-quota requirement with its verbatim error message, the 40,000 defaults, and the one-simultaneous-stream-reader recommendation for global tables.
- [Best practices for global tables](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/globaltables-bestpractices.html) — per-replica deletion protection, PITR-on-one-replica guidance, and the source-Region-capacity-overrides-your-template gotcha with its `insufficient TableMaxReadCapacityUnits limits` error.
- [Global tables FAQ — AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/dynamodb-global-tables/faq.html) — rWCU/rWRU pricing model, no reserved capacity for rWCUs, the MRSC Region sets, the region-attribute + Lambda-event-filter pattern including the delete caveat, DAX staleness, tags not propagating, and the replica-bootstrap-priced-as-a-restore statement.
- [Preparation checklist for global tables](https://docs.aws.amazon.com/prescriptive-guidance/latest/dynamodb-global-tables/checklist.html) — the 180,000 ms `ReplicationLatency` alert example, watching for it dropping to zero, and the data-plane-not-control-plane failover principle.
- [Best practices for Amazon DynamoDB Global Tables – Part 1: Operational readiness](https://aws.amazon.com/blogs/database/best-practices-for-amazon-dynamodb-global-tables-part-1-operational-readiness/) — the failover capacity gap, "auto scaling takes minutes to react", the on-demand recommendation, and the 3,000 ms / 5,000 ms alarm tiers.
- [Best practices for Amazon DynamoDB Global Tables – Part 2: Failover strategies](https://aws.amazon.com/blogs/database/best-practices-for-amazon-dynamodb-global-tables-part-2-failover-strategies/) — ARC vs DNS failover with RTO figures, post-failover stale reads and idempotency guidance, throttling and connectivity troubleshooting.
- [Build resilient applications with Amazon DynamoDB global tables: Part 4](https://aws.amazon.com/blogs/database/part-4-build-resilient-applications-with-amazon-dynamodb-global-tables/) — observability, deployment pipelines and runbooks for multi-region DynamoDB; the data-plane-during-recovery principle.
- [Pre-warming Amazon DynamoDB tables with warm throughput](https://aws.amazon.com/blogs/database/pre-warming-amazon-dynamodb-tables-with-warm-throughput/) — the 4,000 writes/sec and 12,000 reads/sec on-demand defaults, one-off pricing at provisioned rates, asynchronous non-blocking behaviour, and that warm throughput updates synchronise across all global table replicas.
- [Amazon DynamoDB introduces warm throughput for tables and indexes](https://aws.amazon.com/about-aws/whats-new/2024/11/amazon-dynamodb-warm-throughput-ondemand-provisioned-tables) — launch date (13 Nov 2024) and availability in all commercial Regions.
- [Monitoring global tables](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/globaltables_monitoring.html) — full `ReplicationLatency` and `PendingReplicationCount` definitions. **Note: this page is explicitly flagged as legacy-version documentation**, which is exactly why the `PendingReplicationCount` confusion persists.
- [Point-in-time backups for DynamoDB — before you begin](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/pointintimerecovery_beforeyoubegin.html) — cross-Region restore support, the `--source-table-arn` requirement, Hong Kong / Bahrain exclusions, and that restored tables leave the global table configuration behind.
- [hashicorp/aws — `aws_dynamodb_table`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/dynamodb_table) — `replica` block arguments (`region_name`, `kms_key_arn`, `point_in_time_recovery`, `deletion_protection_enabled`, `propagate_tags`, `consistency_mode`), `warm_throughput`, `on_demand_throughput`, `global_table_witness`, and the mutual-exclusivity warning.
- [hashicorp/aws — `aws_dynamodb_table_replica`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/dynamodb_table_replica) — the alternative model, its `stream_enabled` prerequisite, provider-alias example and `table-name:main-region` import form.
- [hashicorp/aws — `aws_dynamodb_global_table`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/dynamodb_global_table) — confirms it manages V1 (2017.11.29) and redirects to `replica` blocks for V2.
- [hashicorp/aws — `internal/service/dynamodb/table.go`](https://github.com/hashicorp/terraform-provider-aws/blob/main/internal/service/dynamodb/table.go) — the provider schema proving `replica` is not `ForceNew` while `name`, `hash_key` and `range_key` are.
- [DynamoDB pricing](https://aws.amazon.com/dynamodb/pricing/on-demand/) — on-demand, provisioned, storage, PITR, export/import and warm-throughput rates. Regional variation applies.
- [Capture DynamoDB global table stream with Lambda — AWS re:Post](https://repost.aws/knowledge-center/dynamodb-global-table-stream-lambda) — the cross-region duplicate-invocation problem stated as a known issue. *(Note: this page returned HTTP 403 to automated fetching; it is cited from the search result summary rather than from the full text, and should be read directly before being relied on.)*
