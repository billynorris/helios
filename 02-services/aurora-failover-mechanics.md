---
title: Aurora Global Database — Failover Mechanics
service: aurora-postgresql-global-database
tags: [service, multi-region, aurora, postgres, failover, runbook, operations, mechanics]
status: researched
replication: native (storage-layer, dedicated infrastructure)
rpo_achievable: "0 for switchover; seconds for managed failover; equal to AuroraGlobalDBRPOLag at the moment of failure"
rto_achievable: "'within a few minutes' (AWS, verbatim) for managed failover with a warm reader; the application's reconnect is the long pole, not the database"
meets_targets: yes — the database clears RTO 15m comfortably; endpoint/connection-pool handling is where the budget is actually spent
ca_west_1: supported — verified against AWS's supported-Regions table 2026-09-22, row identical to ca-central-1
sources_verified: 2026-09-22 — 16 first-party AWS pages plus kernel.org; 2 minor claims uncited (JVM/CoreDNS DNS caching), see Sources
updated: 2026-09-22
---

# Aurora Global Database — Failover Mechanics

> Companion note to [[aws-aurora-global-database]], which covers *whether* to use
> Aurora Global Database and how to build it. **This note covers what you
> actually do at 03:00 and what the machine actually does back.** It is the note
> a runbook is compiled from. Where the parent note states a fact, this note
> explains the mechanism behind it. Read [[failover-orchestration]] for the
> whole-estate sequence and [[split-brain-and-fencing]] for the fencing theory —
> this note is the database-shaped slice of both.

> [!check] Citation pass completed 2026-09-22
> This note previously carried an UNSOURCED banner. **It has now been checked
> claim by claim against first-party AWS documentation** — see [[#Sources]] for
> the 16 pages read and for an explicit "Could not verify" list.
>
> Results worth knowing before you read on:
> - **The `ca-west-1` claim held up.** Calgary is in the Aurora Global Database
>   supported-Regions table with a row identical to Montreal. The CA pair does
>   **not** need a different database DR design.
> - **All 21 RDS event IDs are real** and carry the messages quoted here.
> - **All five `AuroraGlobalDB*` CloudWatch metric names are real**, but one
>   emission Region was wrong and has been corrected.
> - **`tcp_retries2` was understated** and has been corrected upward.
> - Two claims about JVM and CoreDNS DNS caching remain uncited, and the
>   subscribable event category for the write-fencing events is genuinely
>   undocumented. Both are called out in place and in [[#Open questions]].
>
> That last item is the only thing in this note an automation would be built on
> that AWS does not pin down. Everything else here is safe to compile a runbook
> from.

## TL;DR

- **`ca-west-1` is supported — verified against the supported-Regions table.**
  AWS's [supported Regions and DB engines table for Aurora global
  databases](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.Aurora_Fea_Regions_DB-eng.Feature.GlobalDatabase.html)
  lists **Canada West (Calgary)** with a row *identical* to Canada (Central):
  Aurora PostgreSQL 18.3+, 17.4+, 16.1+, 15.2+, 14.3+, 13.4+, 12.8+ and
  "11.9 and version 11.13 and higher"; and Aurora MySQL 3.01.0+, 2.07.0+ and
  "all available versions" of 8.4. Checked 2026-09-22. Unlike Cognito MRR,
  OpenSearch CCR, Managed Grafana and Backup Audit Manager, **the CA pair does
  not need a different database DR design**. This is the one piece of good
  `ca-west-1` news in the vault and it should be recorded as such in
  [[region-pair-selection]]. (Caveats are instance-class and opt-in-Region
  shaped, not feature-shaped — see [[#What blocks a failover]].)
- **There are two failover paths and they are different operations, not two
  settings on one operation.** Managed failover (`FailoverGlobalCluster`)
  preserves the global cluster and auto-rejoins the old primary. Manual
  detach-and-promote (`RemoveFromGlobalCluster`) **destroys the global cluster**
  and leaves you rebuilding the topology by hand, including a full cross-Region
  reseed. The thing that silently moves you from the first path to the second is
  **engine patch-level drift between Regions**. See [[#Path B — manual failover]].
- **Switchover is a *maintenance* tool, not an outage tool.** It requires a
  healthy primary because it quiesces it first. In a real Region event it will
  refuse to run. Teams that rehearse only switchover have rehearsed the path they
  will not get to use. Rehearse both.
- **RPO 2h is not the constraint and never will be.** Aurora replicates at the
  storage layer with "latency typically under a second". You are four orders of
  magnitude inside the RPO target. **The whole DR conversation for this database
  is an RTO and operational-readiness conversation.** Say that out loud in the
  design review so nobody spends money buying RPO they already have.
- **The RTO is spent in the application, not the database.** Aurora promotes
  "within a few minutes" and the global writer endpoint re-points itself. What
  costs you fifteen minutes is a JVM with `networkaddress.cache.ttl=-1`, a
  HikariCP pool holding sixty established sockets to a dead IP, and a Linux
  `tcp_retries2` default that black-holes those sockets for roughly 15 minutes.
  **Plan to restart the application as a mandatory failover step.** See
  [[#Endpoint management — where the RTO is really spent]].
- **The nastiest single gotcha:** after an unplanned managed failover Aurora
  takes a point-of-failure snapshot of the old primary's storage volume named
  `rds:unplanned-global-failover-<cluster>-<timestamp>` — **and it is a system
  snapshot governed by the *old primary cluster's* backup retention period.**
  That snapshot is the only copy of the writes you lost. If retention is 7 days
  and reconciliation takes 8, the evidence deletes itself. Copy it to a manual
  snapshot in the first ten minutes of the incident.

## The two failover paths at a glance

Three operations, not two, because switchover has to be in the table to be
excluded from the outage path.

| | **Switchover** | **Managed failover** | **Manual failover (detach & promote)** |
|---|---|---|---|
| CLI | `aws rds switchover-global-cluster` | `aws rds failover-global-cluster --allow-data-loss` | `aws rds remove-from-global-cluster` |
| API | `SwitchoverGlobalCluster` | `FailoverGlobalCluster` | `RemoveFromGlobalCluster` |
| `--region` points at | **the primary's Region** | **the target secondary's Region** | the target secondary's Region |
| Former name | "managed planned failover" | — | the original way |
| Needs a healthy primary? | **Yes** | No | No |
| Waits for sync? | **Yes** | No — "doesn't wait for data to synchronize" | No |
| RPO | **0** | = `AuroraGlobalDBRPOLag` at failure, "typically measured in seconds" | same |
| Global cluster survives? | Yes | **Yes** | **No** |
| Old primary afterwards | Becomes a read-only secondary | Auto-rejoins as a secondary **with a brand-new storage volume** | Orphaned, still writeable, yours to deal with |
| Engine-version requirement | same major + minor; patch rules vary | same major + minor; patch rules vary | **none** — this is why it exists |
| Point-of-failure snapshot | N/A (no data loss) | **Yes**, `rds:unplanned-global-failover-*` | No — nothing takes one for you |
| Write fencing attempted? | N/A (primary quiesced) | **Yes, best-effort** | **No** |
| Use for | Maintenance, rotation, failback, game days | **A real Region outage** | Outage *and* version drift |

**The `--region` inversion in row 3 is a genuine 3am trap.** Switchover is
issued against the Region that currently holds the primary; failover is issued
against the Region you are promoting *into*. That is the correct design — in an
outage you cannot reach the primary's endpoint — but it means the two commands
in your runbook differ in a way that is easy to copy-paste wrong. Put the region
in an environment variable named for its *role in the command*, not for the pair.

---

## Path A — managed planned failover (switchover)

### What it is for

AWS lists exactly three use cases, verbatim, on [Using switchover or failover in
Amazon Aurora Global
Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html):

> For "regional rotation" requirements imposed on specific industries. For
> example, financial service regulations might want tier-0 systems to switch to a
> different Region for several months to ensure that disaster recovery procedures
> are regularly exercised.

> For multi-Region "follow-the-sun" applications. For example, a business might
> want to provide lower latency writes in different Regions based on business
> hours across different time zones.

> As a zero-data-loss method to fail back to the original primary Region after a
> failover.

Note what is *not* on that list: recovering from an outage. AWS is explicit:

> Switchovers are designed to be used on an Aurora global database where all the
> Aurora clusters and other services they interact with are in a healthy state.
> To recover from an unplanned outage, follow the appropriate procedure in
> Recovering an Amazon Aurora global database from an unplanned outage.

**So switchover is a testing, maintenance and failback tool.** For Helios that
makes it the mechanism behind the quarterly game day and the mechanism behind
failback — both genuinely valuable — and *not* the mechanism you use at 3am when
`eu-west-1` is on fire.

### The mechanism, step by step

What Aurora actually does, in order, from the AWS description:

1. **It waits.** "Before Aurora starts the switchover process, it waits for the
   target secondary Region clusters to be fully synchronized with the primary
   Region cluster." This is the step that makes RPO 0 possible and it is also the
   step whose duration you cannot predict without measuring lag first.
2. **The primary goes read-only.** "Then, the DB cluster in the primary Region
   becomes read-only." This is a real, complete write fence — the primary is
   quiesced by the platform before anything else happens. It is the only point in
   this entire note where fencing is a guarantee rather than a best effort.
3. **The chosen secondary promotes a reader.** "The chosen secondary cluster
   promotes one of its read-only nodes to full writer status."
4. **Roles swap; topology is identical.** "The switchover mechanism maintains
   your global database's existing replication topology: it still has the same
   number of Aurora clusters in the same Regions."

Downtime: AWS says only "Your database is unavailable for a short time while the
primary and selected secondary clusters are assuming their new roles." **AWS does
not publish a switchover duration figure on this page.** Treat any specific
number you have seen as an illustration from a blog, not a commitment.

### Prerequisites, all of which will bite you eventually

| Prerequisite | Source wording | What it means for you |
|---|---|---|
| Healthy primary and healthy dependencies | "all the Aurora clusters and other services they interact with are in a healthy state" | Unavailable in a disaster, by design |
| Same **major and minor** engine version | "only if the primary and secondary DB clusters have the same major and minor engine versions" | Version drift = no switchover |
| Patch level — depends on engine version | "Depending on the engine and engine versions, the patch levels might need to be identical or the patch levels can be different" | See [[#Patch-level compatibility]]. This is a *per-version* table, not a rule |
| Target must have a DB instance | "Before you can perform a switchover or failover to a headless secondary Aurora DB cluster, you must add a DB instance to it" | Headless secondaries cannot be switched over to. Non-prod is affected |
| Low lag preferred | "the larger the lag value, the longer the switchover will take" | Duration is proportional to `AuroraGlobalDBRPOLag`. Check it first |
| Low-write window preferred | "Perform this operation during nonpeak hours or at another time when writes to the primary DB cluster are minimal" | Schedule it like a deploy |

### The command

```bash
# NOTE: --region is the region of the CURRENT PRIMARY.
aws rds --region "$PRIMARY_REGION" switchover-global-cluster \
  --global-cluster-identifier "$GLOBAL_ID" \
  --target-db-cluster-identifier "$SECONDARY_CLUSTER_ARN"
```

`--target-db-cluster-identifier` takes the **ARN**, not the bare identifier.
Passing the identifier is one of the most common failed-command errors here,
because almost every other RDS CLI call accepts the short form.

### What switchover does NOT do for you

Five things, all of which are your job afterwards and all of which AWS states
explicitly on the switchover page:

1. **Parameter groups are not carried across.** "When you promote a secondary DB
   cluster to take over the primary role, the parameter group from the secondary
   might be configured differently than for the primary. If so, modify the
   promoted secondary DB cluster's parameter group to conform to your primary
   cluster's settings."
2. **Monitoring, alarms and CloudWatch Events are not inherited.** "configuration
   for these features isn't inherited from the primary during the switchover
   process." And the sharp edge: "Some CloudWatch metrics, such as replication
   lag, are only available for secondary Regions. Thus, a switchover changes how
   to view those metrics and set alarms on them, and could require changes to any
   predefined dashboards." **Your lag dashboard inverts.** Build both directions
   up front — see [[observability-multi-region]].
3. **Integrations with other AWS services are not reconfigured.** Secrets Manager,
   IAM, S3 and Lambda integrations "need to make sure these are configured as
   required for access from any secondary Regions."
4. **RDS Proxy is not repointed.** "If you're using RDS Proxy, make sure to
   redirect your application's write operations to the appropriate read/write
   endpoint of the proxy that's associated with the new primary cluster."
5. **Aurora PostgreSQL logical replication slots need managing.** The docs link
   out to "Managing logical slots for Aurora PostgreSQL" from both the switchover
   and the failover sections. If anything downstream consumes logical replication
   from this cluster — a CDC pipeline, a Debezium connector, an analytics sink —
   **that consumer's slot does not follow the primary across Regions** and is a
   separate, undocumented-in-your-runbook recovery task. Find out at design time
   whether any exist.

**Recommendation:** run a production switchover quarterly. It is RPO 0, it is the
only way to learn your real switchover duration, and it is the only rehearsal
that exercises items 1–5 above with real consequences. Schedule it in a change
window; treat the five items as a post-switchover checklist with named owners.

---

## Path B — unplanned failover: the path that matters

This is the outage path and it has **two sub-paths**, and which one you get is
decided months earlier by your patching discipline.

### B1 — managed failover (`FailoverGlobalCluster`)

**This is the recommended disaster path.** AWS: "Managed failover – This method
is recommended for disaster recovery. When you use this method, Aurora
automatically adds back the old primary Region to the global database as a
secondary Region when it becomes available again. Thus, the original topology of
your global cluster is maintained."

#### What AWS does for you

1. **Promotes a reader in the chosen secondary to writer.** "The chosen secondary
   cluster promotes one of its read-only nodes to full writer status."
2. **Does not wait for sync.** "Managed failover doesn't wait for data to
   synchronize between the chosen secondary Region and the current primary
   Region. Because Aurora Global Database replicates data asynchronously, it's
   possible that not all transactions replicated to the chosen secondary AWS
   Region before it's promoted."
3. **Guarantees transactional consistency of what survived.** "the recovery
   process that promotes a DB instance on the chosen secondary DB cluster to be
   the primary writer DB instance guarantees that the data is in a transactionally
   consistent state." **This matters and is under-appreciated:** you lose whole
   transactions off the tail, not halves of transactions. You will not find a
   half-applied order. That makes reconciliation a "which transactions are
   missing" problem rather than a "which rows are corrupt" problem, which is
   enormously cheaper.
4. **Attempts write fencing** at the storage layer. Best-effort, see
   [[#Split-brain — the old primary is still writeable]].
5. **Takes a point-of-failure snapshot** of the old primary's storage volume,
   "so that you can restore the snapshot and recover any of the missing data
   from it."
6. **Rebuilds the old primary Region on a fresh storage volume** when it
   recovers, and re-attaches it as a secondary automatically.
7. **Rebuilds every *other* secondary** so they match the new primary
   point-in-time. Not relevant to Helios — every pair has exactly one secondary,
   and it is the one being promoted — but if a third Region is ever added, this
   becomes a "few minutes to several hours" tail on the incident.
8. **Re-points the global writer endpoint** and emits an RDS Event when it
   observes the DNS change.

#### What AWS does NOT do for you

This is the list that makes the unplanned path uglier than the managed marketing
suggests. Everything in the switchover "does not do" list above still applies —
parameter groups, alarms, integrations, RDS Proxy, logical slots — **plus**:

- **It does not take your applications offline.** AWS's first recommendation
  before failing over is literally "To prevent writes from being sent to the
  primary cluster of Aurora Global Database, take applications offline." The
  platform will not do it. That is step 1 of your runbook and it is the only
  fence you control.
- **It does not preserve the lost writes anywhere you can query.** The
  point-of-failure snapshot is a *cluster snapshot*: to read it you must restore
  it into a new cluster, which costs time and money and is a post-incident
  project, not a failover step.
- **It does not tell you how much you lost.** There is no "N transactions
  discarded" output. You infer it from `AuroraGlobalDBRPOLag` *at the moment of
  failure*, which is why the metric has to be on a dashboard you can still reach
  — see [[#RPO and how to measure it before you commit]].
- **It does not size the new primary.** You are now serving production from
  whatever the standby was running. Aurora Auto Scaling is not supported on
  secondary clusters, so there is no elastic safety net; adding capacity is a
  scripted `create-db-instance` and a post-RTO activity.
- **It does not resume your consumers.** Anything you disabled in the standby —
  Lambda event source mappings, EventBridge rules, KEDA-scaled consumers — is
  still disabled. See [[failover-orchestration]] step 4.
- **It does not unfreeze your Terraform**, and you should not let it.

#### The command

```bash
# NOTE: --region is the region of the SECONDARY you are promoting.
# This is the opposite of switchover. Get it wrong and the call fails,
# which is the good outcome; the bad outcome is pasting the wrong ARN.
aws rds --region "$STANDBY_REGION" failover-global-cluster \
  --global-cluster-identifier "$GLOBAL_ID" \
  --target-db-cluster-identifier "$SECONDARY_CLUSTER_ARN" \
  --allow-data-loss
```

`--allow-data-loss` is not optional and not a safety flag you can omit for a
gentler failover — AWS describes it as the thing that "Explicitly make this a
failover operation instead of a switchover operation." **Without it you are
asking for a switchover, which will refuse to run against a dead primary.**

#### Timing

The only figure AWS publishes on this page, verbatim:

> Typically, the chosen secondary cluster assumes the primary role within a few
> minutes.

That is it. "Within a few minutes" is the committed-ish number for the promotion
itself. AWS's August 2023 launch announcement for the feature used the phrase
"typically a minute" for converting a secondary into the new primary. **Both are
vendor claims and neither is an SLA.** No independent, non-AWS measurement of an
unplanned Aurora Global Database cross-Region failover during a real Region event
was found — see the "Could not verify" list in [[#Sources]].

For the secondary-rebuild tail (irrelevant to a two-Region pair, relevant if a
third Region is ever added): "The duration of the complete rebuilding task can
take a few minutes to several hours, depending on the size of the storage volume
and the distance between the Regions."

### B2 — manual failover (detach and promote)

**This is the ugly path, and the reason to care about it is that you do not
choose it — patch drift chooses it for you.**

AWS's framing: "This alternative method can be used when managed failover isn't
an option, for example, when your primary and secondary Regions are running
incompatible engine versions."

#### The procedure, verbatim-derived

1. "Stop issuing DML statements and other write operations to the primary Aurora
   DB cluster in the AWS Region with the outage." *(Your fence. Nothing automates
   this.)*
2. "Identify an Aurora DB cluster from a secondary AWS Region to use as a new
   primary DB cluster. If you have two or more secondary AWS Regions... choose the
   secondary cluster that has the least replication lag."
3. **"Detach your chosen secondary DB cluster from the Aurora global database."**
   "Removing a secondary DB cluster from an Aurora global database immediately
   stops the replication from the primary to this secondary and promotes it to a
   standalone provisioned Aurora DB cluster with full read/write capabilities."
4. "Reconfigure your application to send all write operations to this now
   standalone Aurora DB cluster using its **new endpoint**." And the detail that
   makes this a code/config change rather than a DNS change: "you can change the
   endpoint by removing the `-ro` from the cluster's endpoint string" —
   `my-global.cluster-ro-xxxx.us-west-1.rds.amazonaws.com` becomes
   `my-global.cluster-xxxx.us-west-1.rds.amazonaws.com`.
5. "Add an AWS Region to the DB cluster. When you do this, the replication process
   from primary to secondary begins."
6. "Add more AWS Regions as needed to recreate the topology needed to support
   your application."

#### Why this is much worse than it looks

**1. The global writer endpoint is gone.** The detached cluster is a *standalone*
cluster. It has a cluster endpoint and a reader endpoint and no global writer
endpoint, because there is no longer a global cluster in front of it. Everything
[[aws-aurora-global-database]] says about "you don't have to change your
connection string" evaporates on this path. **Your application must be
re-pointed**, which means either a config change and redeploy, or a Route 53
CNAME you had the foresight to put in front of the endpoint. This alone is an
argument for the CNAME-indirection option in
[[#Endpoint management — where the RTO is really spent]] even though the parent
note recommends using the global endpoint directly.

**2. The old primary is not touched at all.** Nothing halts it, nothing fences
it, no snapshot is taken of its storage volume. If it is alive and reachable it
keeps accepting writes. Compare with managed failover, which at least *tries*.
**This path has zero platform-provided fencing.** See
[[split-brain-and-fencing]].

**3. The global cluster does not clean itself up.** After you detach everything,
per [Removing a cluster from an Amazon Aurora global
database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-detaching.html):
"The Aurora global database might remain in the **Databases** list, with zero
Regions and AZs." You now have a zombie `aws_rds_global_cluster` in your
Terraform state pointing at nothing. Note also that `remove-from-global-cluster`
takes the **ARN** for `--db-cluster-identifier` — AWS's own example passes
`{{secondary_cluster_ARN}}`.

**4. Rebuilding the topology is a full cross-Region reseed.** Step 5 —
"Add an AWS Region" — is a fresh secondary build: a complete copy of the storage
volume across the Region boundary. The parent note records AWS's "a few minutes
to several hours" for a rebuild of comparable shape. This happens *after* the
incident, unattended, and it costs cross-Region transfer.

**5. The cluster identifier is now wrong and cannot be fixed in place.** "You
can't rename a Regional Aurora DB cluster while it is a member of an Aurora
global database." (You *can* change the global cluster identifier and individual
instance identifiers.) So the cluster whose name says `...-eu-west-2` is now your
primary, forever, or until you rebuild it. Cosmetic until someone writes an
automation that parses the name.

**6. Terraform will be badly confused.** Managed failover leaves every resource
in existence with inverted roles, which the `ignore_changes` blocks in
[[aws-aurora-global-database#lifecycle blocks are load-bearing, not decoration]]
absorb. Detach-and-promote **removes cluster membership**, which those
`ignore_changes` cannot absorb because the resource's relationship to the global
cluster genuinely no longer exists. Expect to hand-edit state. Expect it to be
the worst two hours of the incident.

#### Recommendation

**Treat B2 as a failure of preparation, not a choice.** The single control that
keeps you on B1 is **strict engine-version parity between the two Regions**,
enforced in Terraform (one pinned `engine_version` variable feeding both
clusters) and verified by a scheduled check. Write the B2 procedure down anyway —
it is the only path that works when versions have drifted, and an incident is a
bad time to discover you never wrote it — but treat every execution of it as a
P2 incident review in its own right.

```bash
# The B2 path, for completeness. Read the consequences above before running this.
aws rds --region "$STANDBY_REGION" remove-from-global-cluster \
  --db-cluster-identifier "$SECONDARY_CLUSTER_ARN" \
  --global-cluster-identifier "$GLOBAL_ID"

# Then, and only then, re-point the application at the *cluster* endpoint
# (note: -ro removed) and start rebuilding the topology.
aws rds --region "$STANDBY_REGION" describe-db-clusters \
  --db-cluster-identifier "$SECONDARY_CLUSTER" \
  --query 'DBClusters[0].Endpoint' --output text
```

---

## RPO and how to measure it before you commit

### What AWS actually commits to

Two statements, both from the Aurora User Guide, and neither is an SLA:

> After any write operation, Aurora replicates data to the secondary AWS Regions
> using dedicated infrastructure, with latency **typically under a second**.

> For an Aurora global database, **RPO is typically measured in seconds**.

That is the whole of the public commitment. There is no published percentile, no
published bound, and no SLA credit attached to replication lag. **Your actual RPO
is whatever `AuroraGlobalDBRPOLag` reads at the instant the primary dies**, and
the only way to know your normal is to measure it on your own workload across
your own Region pair. Ireland↔London and Montreal↔Calgary are very different
distances; do not assume one pair's number generalises.

### The metrics, with the real names

All five names below were checked against AWS's [cluster-level CloudWatch metrics
table for Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMonitoring.Metrics.html#Aurora.AuroraMySQL.Monitoring.Metrics.clusters)
on 2026-09-22. All five exist, all five are spelled as shown, and — this is the
correction — **all five are emitted only in secondary Regions.** Descriptions in
quotes are AWS's.

| Metric | Emitted where | Units | AWS's description |
|---|---|---|---|
| **`AuroraGlobalDBRPOLag`** | **Secondary only** | Milliseconds | "the recovery point objective (RPO) lag time. This metric measures how far the secondary cluster is behind the primary cluster for user transactions." **This *is* your RPO.** |
| `AuroraGlobalDBReplicationLag` | Secondary only | Milliseconds | "the average time elapsed replicating updates between the primary cluster's replication server and the secondary cluster's replication server." The older, coarser metric |
| `AuroraGlobalDBDataTransferBytes` | **Secondary only** — *see correction below* | Bytes | "the amount of redo log data transferred from the source AWS Region to a secondary AWS Region" |
| `AuroraGlobalDBReplicatedWriteIO` | Secondary only | Count | "the number of write I/O operations replicated from the primary AWS Region to the cluster volume in a secondary AWS Region." AWS states billing for the primary Region uses this metric "to account for cross-Region replication within the global database" — so it is the **billing** metric as much as the operational one. [[cost-model]] |
| `AuroraGlobalDBProgressLag` | Secondary only | Milliseconds | "the measure of how far the secondary cluster's storage volume is behind the primary cluster's storage volume for both user transactions and system transactions" |

> [!warning] Correction made during the citation pass
> This note previously claimed `AuroraGlobalDBDataTransferBytes` was emitted on
> the **primary**. It is not. AWS's metric table says, for every one of the five
> metrics above, "This metric is available only in secondary AWS Regions."
> The practical effect is *in your favour*: the liveness signal used in the
> decision tree below lives in the Region that survives, not the one that died.

Which lag metric to read is version-dependent, and AWS says so on the
switchover/failover page verbatim: "For all versions of Aurora PostgreSQL-based
global databases, and for Aurora MySQL-based global databases starting with
engine versions 3.04.0 and higher or 2.12.0 and higher, use Amazon CloudWatch to
view the `AuroraGlobalDBRPOLag` metric for all secondary DB clusters. For lower
minor versions of Aurora MySQL-based global databases, view the
`AuroraGlobalDBReplicationLag` metric instead." **On Aurora PostgreSQL,
`AuroraGlobalDBRPOLag` is always the right one.**

> [!note] A wrinkle AWS does not reconcile
> The same paragraph ends "When you examine these metrics, do so from the current
> primary cluster" — while the metric table says the metrics exist *only* in
> secondary Regions. The reconciliation is almost certainly that you view the
> secondary's metrics from a console session scoped to the primary, but AWS does
> not say so. **Treat the metric table as authoritative on where the data lives**
> (secondary Region, secondary cluster ID), because that is the statement your
> alarms are actually built on.

> [!warning] The dashboard inverts at failover
> AWS, verbatim: *"Some CloudWatch metrics, such as replication lag, are only
> available for secondary Regions. Thus, a switchover changes how to view those
> metrics and set alarms on them, and could require changes to any predefined
> dashboards."* Every one of the metrics above is emitted **in the standby
> Region**, against the **standby cluster's** identifier. After a failover the
> roles swap, so the metric moves to the *other* Region and the *other* cluster
> ID. **Any alarm hard-coded to `DBClusterIdentifier = <standby>` silently stops
> evaluating** — it does not alarm, it goes `INSUFFICIENT_DATA` and most teams
> route that to nowhere. Create both directions' alarms at build time.
> [[observability-multi-region]].

### The SQL view — better than CloudWatch during an incident

CloudWatch has a publication delay and, in a Region event, a control plane you
may not love. The database will tell you directly:

```sql
-- Run against the PRIMARY cluster (per AWS's instructions).
SELECT * FROM aurora_global_db_status();
```

Columns, with AWS's own definitions:

| Column | Definition |
|---|---|
| `aws_region` | The Region this DB cluster is in |
| `highest_lsn_written` | "The highest log sequence number (LSN) currently written on this DB cluster" |
| `durability_lag_in_msec` | "The timestamp difference between the highest log sequence number written on a secondary DB cluster (`highest_lsn_written`) and the `highest_lsn_written` on the primary DB cluster" |
| `rpo_lag_in_msec` | "The recovery point objective (RPO) lag. This lag is the time difference between the most recent user transaction commit stored on a secondary DB cluster and the most recent user transaction commit stored on the primary DB cluster." |
| `last_lag_calculation_time` | When the two lag figures were last computed |
| `feedback_epoch`, `feedback_xmin` | Hot-standby feedback internals; ignore for DR |

In AWS's own example output the primary's row shows `-1` for both lag columns
(the value is meaningless on the primary) and the secondary's row carries the
real numbers.

And per-instance:

```sql
SELECT * FROM aurora_global_db_instance_status();
-- server_id, session_id, aws_region, durable_lsn, highest_lsn_rcvd,
-- feedback_epoch, feedback_xmin, oldest_read_view_lsn, visibility_lag_in_msec
```

`session_id = MASTER_SESSION_ID` identifies the current writer. That is a
one-query answer to "which instance is the writer right now", which is worth
knowing at 3am.

> [!note] `durability_lag` and `rpo_lag` are not the same thing and the
> difference is the whole point
> AWS's worked example: start a transaction, run an `INSERT` that takes an hour,
> then `COMMIT`. If the Regions partition after `BEGIN`, `durability_lag_in_msec`
> climbs to an hour — **but `rpo_lag_in_msec` stays at 0**, "because all the user
> data committed between the primary DB cluster and secondary DB cluster are still
> the same". It only jumps when the `COMMIT` lands.
>
> **Consequence for the runbook: alarm on `rpo_lag`, investigate on
> `durability_lag`.** A high durability lag with zero RPO lag means a long
> transaction is in flight, not that you are about to lose data. A team that
> alarms on durability lag will page itself every time the nightly batch job
> runs.

### How to decide: wait for lag to drain, or accept the loss?

This is a real decision gate and it should be pre-decided, not improvised.

```
                 read AuroraGlobalDBRPOLag (or rpo_lag_in_msec)
                              │
              ┌───────────────┴────────────────┐
              │                                │
       metric is FRESH                  metric is STALE or
       and readable                     the Region is gone
              │                                │
              ▼                                ▼
      lag < 60s?  ──yes──►  fail over now.   You have no RPO number.
              │             Loss is noise.   Assume worst case, fail over,
              no                             and start the reconciliation
              │                              workstream immediately.
              ▼
      Is the primary still
      replicating at all?
      (AuroraGlobalDBDataTransferBytes
       still non-zero?)
              │
      ┌───────┴────────┐
      │                │
     yes               no
      │                │
      ▼                ▼
  WAIT. Lag is     Replication is dead.
  draining.        Waiting buys nothing.
  Re-check every   Fail over now.
  60s, cap the
  wait at 5 min.
```

**The rule to write in the runbook:** *wait only if lag is actively falling and
the wait is bounded at five minutes.* Anything else is hope. And note the
asymmetry that makes this easy: with RPO 2h and a sub-second normal lag, **you
would have to be more than seven thousand times worse than normal before the
business-stated RPO is even at risk.** The reason to wait is not RPO compliance —
it is reducing the size of the reconciliation job. Which is a real reason, but a
much weaker one, and it should not be allowed to burn RTO.

### Getting the number when the primary is unreachable

The SQL view runs against the primary, which in a real outage is gone. Your
fallbacks, in order:

1. **`AuroraGlobalDBRPOLag` in the *standby* Region's CloudWatch.** The metric is
   published in the standby Region, so it survives the primary Region's loss.
   This is the single most important reason to have the lag dashboard hosted in
   the standby Region rather than the primary. [[observability-multi-region]].
2. **The last value scraped by whatever external monitoring you have.** A
   third-region Prometheus/Grafana scrape of the metric every 30 seconds gives
   you a last-known-good figure with a known staleness bound.
3. **Nothing.** If you have neither, you enter the incident without knowing your
   data loss, and you will be answering "how much did we lose?" with "we don't
   know" for the entire bridge call. Fixing this costs one dashboard.

---

## What is actually lost

### The three categories

| Category | Fate | Recoverable? |
|---|---|---|
| **Committed and replicated** | Present on the new primary | N/A — safe |
| **Committed but not yet replicated** | **Lost from the new primary.** Present only in the `rds:unplanned-global-failover-*` snapshot | Yes, by restoring the snapshot and diffing — a project, not a step |
| **In flight, uncommitted** | Lost. Aurora "guarantees that the data is in a transactionally consistent state" on the promoted cluster | No, and correctly so — they were never durable |

The middle row is the whole of the problem. Those are transactions your
application told the customer had succeeded.

### Why "transactionally consistent" is the most valuable sentence on the page

> the recovery process that promotes a DB instance on the chosen secondary DB
> cluster to be the primary writer DB instance guarantees that the data is in a
> transactionally consistent state

Aurora replicates the *storage volume* and promotes at a consistent point in the
redo stream. You therefore lose a **suffix of the commit history**, not a random
scatter of rows. Practically:

- No torn transactions. An order row and its line items are either both there or
  both gone.
- The loss boundary is a single LSN, not a set. `highest_lsn_written` on the
  promoted cluster *is* the boundary.
- Reconciliation reduces to "replay everything after LSN *X*", which is tractable
  **if and only if** the application has an idempotent replay path or an event
  log. See the reconciliation discussion in [[split-brain-and-fencing#Reconciling divergent writes]].

**Capture `highest_lsn_written` from the promoted cluster immediately after
promotion.** It is the Aurora analogue of the `pg_last_wal_receive_lsn()` capture
that [[aws-rds-postgres]] prescribes, it costs one query, and it is the only
mechanical record of the boundary you will ever have.

### The RPO target is not the constraint — say it plainly

The brief sets **RPO 2 hours**. Aurora's typical lag is **under one second**.

> **The RPO target is satisfied by roughly four orders of magnitude and is not a
> design input for this database.** Nothing in the Aurora design should be chosen
> to improve RPO, because there is nothing to buy. In particular: do **not** turn
> on `rds.global_db_rpo`; do **not** pay for synchronous anything; do **not**
> treat "reduce lag" as a project.

What *is* the constraint:

1. **RTO 15 minutes**, and specifically the application's ability to notice the
   endpoint moved. See the next section.
2. **Operational readiness** — whether the runbook exists, whether the team has
   run it, whether the standby has a running instance, whether versions match.
3. **Reconciliation capability** — whether you can replay the lost suffix at all.
   This is an application property, decided years before the incident, and it is
   the only thing on this list that money cannot fix at 3am.

### The one thing that can turn RPO into an availability problem

`rds.global_db_rpo` lets you enforce an RPO ceiling by **blocking commits on the
primary** when all secondaries exceed it. AWS: "Blocks the transaction if all
secondary DB clusters have RPO lag times that are larger than the RPO." Valid
range 20 s to 2,147,483,647 s; dynamic.

AWS's own warning for exactly our topology, verbatim:

> In a global database with only two AWS Regions, we recommend keeping the
> `rds.global_db_rpo` parameter's default value in the secondary Region's
> parameter group. Otherwise, performing a failover due to a loss of the primary
> AWS Region could cause Aurora to pause transactions.

With one secondary, "all secondaries are behind" is the same statement as "the
secondary is behind". Every Helios pair is two-Region. **Leave it at `-1`.** It
also blocks major version upgrades. AWS, verbatim, from the [upgrade
page](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-upgrade.html):
"With an Aurora global database based on Aurora PostgreSQL, you can't perform a
major version upgrade of the Aurora DB engine if the recovery point objective
(RPO) feature is turned on."

---

## Write forwarding

### What it actually is

Write forwarding lets a **secondary** cluster accept write statements and ship
them transparently to the primary Region's writer. AWS:

> With write forwarding enabled, secondary clusters in an Aurora global database
> forward SQL statements that perform write operations to the primary cluster.
> The primary cluster updates the source and then propagates resulting changes
> back to all secondary AWS Regions.

> Aurora handles the cross-Region networking setup. Aurora also transmits all
> necessary session and transactional context for each statement. The data is
> always changed first on the primary cluster and then replicated to the secondary
> clusters in the Aurora global database. **This way, the primary cluster is the
> source of truth and always has an up-to-date copy of all your data.**

That last sentence is the whole reason it is **not active/active**. There is
exactly one writer, in one Region, at all times. Write forwarding is a
*connection convenience*, not a topology change. It removes the need for your
application to hold a second connection to a far-away Region; it does not make
the second Region able to accept writes if the first one is gone.

**You connect to the secondary's *reader* endpoint** to use it — not its cluster
endpoint. AWS, verbatim: "On a secondary cluster, the cluster endpoint displays a
status of **inactive** because it doesn't handle write requests. You can still
connect to the cluster endpoint, but only for read queries."

### Version support (Aurora PostgreSQL)

Verbatim, from [Using write forwarding in an Aurora PostgreSQL global
database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-write-forwarding-apg.html)
(verified 2026-09-22): "In Aurora PostgreSQL version 16 and higher major
versions, global write forwarding is supported in all minor versions. For earlier
Aurora PostgreSQL versions, global write forwarding is supported with version
15.4 and higher minor versions, and version 14.9 and higher minor versions. Write
forwarding is available in every AWS Region where Aurora PostgreSQL-based global
databases are available."

So — **available in `ca-west-1`**, since the global-database Region list covers
it. This is an *inference from two verified AWS statements* (Calgary is in the
global-database Region table; write forwarding is available wherever that is),
not a directly stated fact. AWS publishes no per-Region write-forwarding table.
No parity gap here.

Enabling or disabling it "doesn't cause downtime or a reboot", and the console
note is worth knowing: if you pick "apply during the next maintenance window",
"Aurora ignores this setting and turns on write forwarding immediately."

Toggle: `--enable-global-write-forwarding` / `--no-enable-global-write-forwarding`
on `create-db-cluster` or `modify-db-cluster`; `EnableGlobalWriteForwarding` in
the API; `GlobalWriteForwardingStatus` (`enabled` / `disabled` / `enabling` /
`disabling` / `null`) to read it back. `null` means "not applicable" — either not
in a global database, or it *is* the primary.

### Consistency levels — `apg_write_forward.consistency_mode`

Session-scoped enum. Valid values `SESSION` (default), `EVENTUAL`, `GLOBAL`,
`OFF`. AWS's framing of the trade-off:

> As you increase the consistency level, your application spends more time
> waiting for changes to be propagated between AWS Regions.

| Mode | Semantics (AWS's words, condensed) | Latency cost | When you'd pick it |
|---|---|---|---|
| `EVENTUAL` | "might see data that is slightly stale due to replication lag. Results of write operations in the same session aren't visible until the write operation is performed on the primary Region and replicated to the current Region. **The query doesn't wait**" | Lowest — the forwarded write still costs a cross-Region round trip, but reads never block | Fire-and-forget writes where the same session does not read back |
| `SESSION` **(default)** | "All queries... see the results of all changes made in that session... regardless of whether the transaction is committed. If necessary, the query **waits** for the results of forwarded write operations to be replicated to the current Region. It doesn't wait for updated results from write operations performed in other Regions or in other sessions" | Read-after-your-own-write. Waits for *your* writes to come back | The sane default, and the one AWS chose |
| `GLOBAL` | "sees changes made by that session... **plus all committed changes from both the primary AWS Region and other secondary AWS Regions**. Each query might wait for a period that varies depending on the amount of session lag" | Highest. Every query blocks until the secondary is current as of query start | Correctness-critical reads that must not miss another Region's writes |
| `OFF` | "disables write forwarding in the session" | n/a | A per-session kill switch — useful in a runbook |

The waiting is observable. AWS publishes dedicated wait events:
`IPC:AuroraWriteForwardConsistencyPoint` (generated only under `SESSION` and
`GLOBAL`), `IPC:AuroraWriteForwardConnect`, `IPC:AuroraWriteForwardExecute`,
`IPC:AuroraWriteForwardGetGlobalConsistencyPoint` (only under `GLOBAL`),
`IPC:AuroraWriteForwardXactStart` / `XactCommit` / `XactAbort`. If write
forwarding is ever enabled, **these belong on the Performance Insights dashboard
from day one**, because "the app got slower after we enabled write forwarding"
is otherwise an unfalsifiable claim.

And the CloudWatch metrics, which are real and specific:

| Metric | Where | Meaning |
|---|---|---|
| `AuroraForwardingWriterDMLThroughput` | Primary writer | Forwarded DML statements/sec processed |
| `AuroraForwardingWriterOpenSessions` | Primary writer | Open sessions handling forwarded queries |
| `AuroraForwardingWriterTotalSessions` | Primary writer | Total forwarded sessions |
| `AuroraForwardingReplicaDMLLatency` | Secondary reader | "Average response time in milliseconds of forwarded DMLs on replica" — **the cost, in one number** |
| `AuroraForwardingReplicaDMLThroughput` | Secondary reader | "Number of forwarded DML statements processed on this replica each second" |
| `AuroraForwardingReplicaReadWaitLatency` | Secondary reader | "Average wait time in milliseconds that the replica waits to be consistent with the LSN of the primary cluster. The degree to which the reader DB instance waits depends on the `apg_write_forward.consistency_mode` setting." **This is the price of `SESSION`/`GLOBAL`, measured.** |
| `AuroraForwardingReplicaCommitThroughput` | Secondary reader | "Number of commits in sessions forwarded by this replica each second" |
| `AuroraForwardingReplicaOpenSessions` | Secondary reader | "The number of sessions that are using write forwarding on a replica instance" |
| `AuroraForwardingReplicaErrorSessionsLimit` | Secondary reader | "Number of sessions rejected by the primary cluster because the limit for **max connections or max write forward connections** was reached." Alarm on this; see below |

All eight names, units and descriptions above verified 2026-09-22 against the
write-forwarding page's two CloudWatch tables. AWS states the first three "are
all measured on the writer DB instance in the primary cluster" and the rest "are
measured on each reader DB instance in a secondary cluster with write forwarding
enabled."

### The connection-budget trap

`apg_write_forward.max_forwarding_connections_percent` is **global-scope**, type
`int`, defaults to **25**, valid range **1–100** (all four confirmed from AWS's
parameter table), and is "the upper limit on database connection slots that can
be used to handle queries forwarded from readers… expressed as a percentage of
the `max_connections` setting for the writer DB instance in the primary cluster."
AWS's own worked example, verbatim: "if `max_connections` is `800` and
`apg_write_forward.max_forwarding_connections_percent` is `10`, then the writer
allows a maximum of 80 simultaneous forwarded sessions. **These connections come
from the same connection pool managed by the `max_connections` setting.**"

Read that carefully: **forwarded sessions consume the primary writer's
`max_connections` budget.** Turning on write forwarding silently hands a quarter
of your primary's connection capacity to a remote Region. On a writer that is
already connection-pressured, this is a production incident waiting for a busy
Tuesday.

And it compounds with RDS Proxy. AWS, flagged as **Important** on [Using RDS
Proxy with Aurora global
databases](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-proxy-gdb.html):

> If the DB cluster is part of a global database with write forwarding turned on,
> reduce your proxy's `MaxConnectionsPercent` value by the quota that's allotted
> for write forwarding. The write forwarding quota is set in the DB cluster
> parameter `aurora_fwd_writer_max_connections_pct`.

> [!warning] That parameter name is the Aurora **MySQL** one
> `aurora_fwd_writer_max_connections_pct` is the Aurora MySQL parameter. The
> Aurora PostgreSQL equivalent is `apg_write_forward.max_forwarding_connections_percent`
> (same meaning, same default of 25). AWS's RDS Proxy page names only the MySQL
> parameter even though the page covers both engines. **The guidance applies; the
> parameter name in it does not, if you are on PostgreSQL.** Flagged here because
> someone will copy that name into a Terraform parameter group and it will be
> silently ignored.

### SQL you cannot use

Unsupported with write forwarding (Aurora PostgreSQL), verbatim list: DDL
statements, `ANALYZE`, `CLUSTER`, `COPY`, **cursors** ("make sure to close them
before using write forwarding"), `GRANT`/`REVOKE`/`REASSIGN OWNED`/`SECURITY
LABEL`, `LOCK`, `SAVEPOINT` statements, `SELECT INTO`, `SET CONSTRAINTS`,
`TRUNCATE`, `VACUUM`. Also: "user defined functions and user defined procedures
aren't supported."

Supported: DML (`INSERT`/`UPDATE`/`DELETE`), `SELECT FOR { UPDATE | NO KEY UPDATE
| SHARE | KEY SHARE }`, `PREPARE`/`EXECUTE`, and `EXPLAIN` over those.

> [!note] A documentation conflict — confirmed real, both sides cited
> **This was checked on 2026-09-22 and the contradiction is genuine, not a
> misreading.**
>
> [Using write forwarding in an Aurora PostgreSQL global
> database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-write-forwarding-apg.html)
> lists under "You can use the following kinds of SQL statements with write
> forwarding": "`SELECT FOR { UPDATE | NO KEY UPDATE | SHARE | KEY SHARE }`
> statements".
>
> [Connecting to Amazon Aurora Global
> Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-connecting.html)
> says: "write forwarding doesn't support certain MySQL or PostgreSQL operations,
> such as making data definition language (DDL) changes or `SELECT FOR UPDATE`
> statements."
>
> The engine-specific page is the more likely to be current and more specific,
> but **do not design around `SELECT FOR UPDATE` over write forwarding without
> testing it in your own account.** Carried to [[#Open questions]] rather than
> resolved, because guessing here would be inventing.

Isolation: `REPEATABLE READ` and `READ COMMITTED` work; **`SERIALIZABLE` is not
supported**. And "If the transaction access mode is set to read only, write
forwarding isn't used."

RDS Proxy adds one more: "When write forwarding is enabled on an Aurora
PostgreSQL Global Database secondary cluster, **RDS Proxy doesn't support the
`SESSION` value** for the `apg_write_forward.consistency_mode` parameter. Setting
this value can cause unexpected behavior." Since `SESSION` is the *default*,
**RDS Proxy + write forwarding + defaults = an explicitly unsupported
configuration.** That combination is easy to arrive at by accident.

### Why it makes failover worse, not better

From the connecting page, verbatim:

> However, with write forwarding, you do need to update your application code or
> configuration to connect to the **newly promoted primary Region's reader
> endpoint** after performing a cross-Region failover or switchover.

This is the part people miss. The global writer endpoint exists specifically so
you never change a connection string. Write forwarding routes through the
*secondary's reader endpoint* instead — which is a **regional** name. After a
failover, the Region whose reader endpoint you were using is the primary, and the
Region you now want to forward from is the other one. **You have reintroduced the
endpoint-change problem that the global writer endpoint removed**, and you have
reintroduced it into the path that only exists to save you a connection string.

### When it is genuinely useful: the partial failover

There is one case where write forwarding earns its keep in an active/passive
estate, and it is worth naming because it is not obvious.

**Scenario: the primary Region's *compute* is impaired but its *database* is
fine.** An EKS control-plane event, an AZ-wide capacity problem, a bad deploy
that cannot be rolled back, a network path issue between your users and the
primary Region's ALB. The Aurora cluster in the primary is healthy and writable.
The application in the primary Region is not serving.

Without write forwarding, your options are (a) fail the database over too —
accepting data loss and a one-way-ish door for a problem that is not a database
problem, or (b) route traffic to the standby Region's compute and have it reach
*across* to the primary's writer, which means VPC peering/TGW connectivity you
may not have and a cross-Region round trip on every write anyway.

With write forwarding enabled on the standby, option (c) exists: **shift traffic
to the standby Region's compute, which connects to its local reader endpoint,
reads locally and forwards writes to the still-healthy primary.** No database
failover. No data loss. No promotion. When the primary's compute recovers, you
shift traffic back and nothing needs undoing.

That is a genuinely valuable capability and it is the strongest argument for
turning write forwarding on. Cost of holding it: the 25% connection budget, a
config surface, the RDS Proxy incompatibility, and one more thing that behaves
differently after a real failover.

### Recommendation

**Leave write forwarding off by default, but build the standby so it can be
turned on in one `modify-db-cluster` call during a compute-only incident.**

Rationale: the steady-state value in active/passive is zero (no traffic in the
standby, nothing to forward); the failover-time value is negative (it
reintroduces an endpoint change); the *partial*-failover value is real but rare.
Since enabling it "doesn't cause downtime or a reboot" and applies immediately,
you do not need it on in advance to benefit from it. **Put it in the runbook as a
branch, not in the Terraform as a default.**

Expose it as `var.enable_write_forwarding` defaulting to `false`, document the
`max_forwarding_connections_percent` interaction next to the variable, and add a
runbook decision gate: *"Is the primary's database healthy and only its compute
impaired? → consider write forwarding instead of failover."* That gate alone may
be worth more than the feature.

---

## Endpoint management — where the RTO is really spent

The database promotes in "a few minutes". The fifteen-minute RTO is not at risk
from Aurora. It is at risk from everything between your application process and
that endpoint.

### What the global writer endpoint gives you

```
<global_cluster_id>.global-<unique_string>.global.rds.amazonaws.com
```

Retrieved from `describe-global-clusters` → `.GlobalClusters[].Endpoint`, or in
Terraform as `aws_rds_global_cluster.this.endpoint`.

> Each Aurora Global Database comes with a writer endpoint that is automatically
> updated by Aurora to route requests to the current writer instance of the
> primary DB cluster. With the writer endpoint, you don't have to modify your
> connection string after you change the location of the primary Region using the
> managed Aurora Global Database switchover and failover capabilities.

Three endpoint types exist and the distinction matters in a runbook:

| Endpoint | Scope | Survives failover? |
|---|---|---|
| **Global writer** | Global | **Yes — follows the primary** |
| Cluster (writer) | Regional | No. On a secondary it shows status `inactive` and accepts reads only |
| Reader | Regional | Yes as a name, but its *meaning* changes (it is now the primary's reader) |

**If the estate is currently connecting to the primary cluster's regional writer
endpoint, migrating to the global writer endpoint is the single highest-value,
lowest-risk change in this entire note.** AWS says so directly: "If you set up
your global database before the Aurora Global Database writer endpoint was
available, your application might connect to the cluster endpoint of the primary
cluster. In this case, we recommend switching your connection settings to use the
global writer endpoint instead."

It is a connection-string change. It is reversible. It can be done months before
any failover. Do it first.

### What it does not give you — the four documented caveats

**1. DNS caching, which AWS names as a *long* delay.** Verbatim:

> The global writer endpoint update after a global database failover or switchover
> **can take a long time depending upon your Domain Name Service (DNS) caching
> duration.**

And the prescription, from the failover page:

> If you are using the global writer endpoint and your application or networking
> layers cache DNS values, **reduce the time-to-live (TTL) of your DNS cache to a
> low value such as 5 seconds.**

> As an alternative, you can check for the RDS event that informs you when Aurora
> observed the DNS changes for the global writer endpoint. That way, you can
> validate that your application also registered the DNS change before restarting
> your application write traffic.

**Use that event as the automation trigger.** A fixed `sleep 60` in a runbook is
a guess; the RDS Event is a fact. See [[#The RDS events to key automation off]].

**2. Cross-VPC reachability.** Verbatim:

> Although the global writer endpoint lets you avoid changing the connection
> settings for your application, **your applications can't access the IP addresses
> in the newly promoted primary AWS Region's VPC until you set up networking
> between the two VPCs.**

For Helios this is a non-issue *provided* the standby Region runs its own copy of
the application in its own VPC — which is the entire point of the programme. It
becomes a real issue the moment somebody designs a "surviving service in the
primary Region reaches across to the promoted database" path. **Don't.** If that
path is ever needed, it needs peering or Transit Gateway built and tested in
advance, and it belongs in [[aws-vpc-networking]].

**3. Renaming the global cluster renames the endpoint.** "The first part of the
writer endpoint name is the name of your Aurora Global Database. Thus, if you
rename your Aurora Global Database, the writer endpoint name changes, and any
code that uses it must be updated with the new name."

In a cookiecutter monorepo this is a live hazard: a template change that alters a
name prefix is a **breaking production change**, not a refactor. Pin
`global_cluster_identifier` explicitly and add a `lifecycle { prevent_destroy }`
or a CI check that fails any plan touching it.

**4. Split-brain is still possible through it.** "Although Aurora attempts on
best-effort basis to block writes in the original primary AWS Region, failover
can be susceptible to split-brain issues."

### The cached-DNS / connection-pool problem, in detail

This is the same class of problem [[aws-route53]] documents for the web tier, and
it is worse for databases because database clients hold connections open for
hours by design.

**Layer 1 — the JVM.** Historically the most-hit trap in this entire space. The
JVM caches successful DNS lookups according to
`networkaddress.cache.ttl` in `$JAVA_HOME/jre/lib/security/java.security`. On
older JREs with a SecurityManager installed the default is **cache forever**
(`-1`); on modern JDKs without a SecurityManager the default is a small positive
number (30 seconds is the commonly cited value). **You must not rely on the
default** — pin it explicitly:

```java
// Do this at startup, before any connection is made.
java.security.Security.setProperty("networkaddress.cache.ttl", "5");
java.security.Security.setProperty("networkaddress.cache.negative.ttl", "3");
```

or in the container image's `java.security`, or via
`-Dsun.net.inetaddr.ttl=5` (the system-property form; the `Security` property
takes precedence where both are set). **Set it in the base image, not per
service**, or you will find the one service that missed it during the incident.

**Layer 2 — the OS / sidecar resolvers.** `nscd`, `systemd-resolved`, and in
Kubernetes **CoreDNS** (default cache TTL 30s) and **NodeLocal DNSCache** if
deployed. Each is an independent additive delay on top of the record's own TTL.
Audit them; each one is a multiplier on your RTO.

**Layer 3 — the connection pool, which is the one that actually hurts.** A
HikariCP/pgbouncer/psycopg pool holding 50 established TCP sockets to the old
writer's IP **will never perform another DNS lookup for those sockets**. DNS TTL
is irrelevant to an already-open connection. What happens instead:

- If the old writer is *gone* (Region hard-down, packets black-holed), the
  sockets do not error immediately. Linux retransmits according to
  `net.ipv4.tcp_retries2`. The kernel's own
  [ip-sysctl documentation](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt),
  verbatim: *"The default value of 15 yields a hypothetical timeout of **924.6
  seconds** and is a lower bound for the effective timeout. TCP will effectively
  time out at the first RTO which exceeds the hypothetical timeout."*
  **924.6 seconds is 15.4 minutes — and it is a lower bound.** That is your
  entire RTO budget, spent doing nothing, before the socket even reports an
  error. (This note previously said "roughly 13–15 minutes"; the documented
  figure is slightly worse than that.)
- If the old writer is *reachable but demoted*, the sockets stay healthy and
  writes fail with `ERROR: cannot execute INSERT in a read-only transaction`
  (SQLSTATE `25006`). Reads keep succeeding, so dashboards stay green while
  10–30% of requests fail. This is the same trap
  [[failover-orchestration#Why each edge in that graph exists]] describes for
  traffic-before-promotion, arriving from the other direction.

**Mitigations, in order of effectiveness:**

| Mitigation | Effect | Cost |
|---|---|---|
| **Restart the application at failover** (`kubectl rollout restart`) | Kills every socket, forces fresh DNS. **The only one that is reliable.** | ~30–90 s, and it is already in the sequence |
| Pool `maxLifetime` shorter than the incident (e.g. 10–15 min) | Bounds how long a stale socket can live in steady state | Slight connection churn |
| TCP keepalives on the JDBC/libpq connection (`tcp_keepalives_idle`, `keepalives_idle=10`) | Detects a black-holed peer in tens of seconds rather than ~15 min | Negligible |
| Lower `tcp_retries2` on the node (e.g. 5 → ~6 s) | Kernel gives up fast | Node-level setting; affects everything on the host. Test it |
| Connection validation query on borrow | Pool discards dead connections when used | Small per-borrow latency |
| RDS Proxy in front | Proxy absorbs the churn and **queues** through the failover (see below) | ~$ per vCPU-hour, plus an endpoint to repoint |

**Recommendation: mandatory application restart, plus keepalives, plus a bounded
`maxLifetime`.** Treat the restart as a required step in the runbook with no
"only if needed" qualifier — the cost of doing it unnecessarily is 60 seconds;
the cost of skipping it is the whole RTO.

### RDS Proxy

RDS Proxy is regional and does not follow the primary across Regions. AWS:

> Make sure to redirect your application's write operations to the appropriate
> read/write endpoint of the proxy that's associated with the new primary cluster.

But it does one genuinely valuable thing during the switch:

> RDS Proxy **queues all requests through read/write endpoints and sends them to
> the writer instance of the new primary cluster as soon as it's available**. It
> does so regardless of whether the switchover or failover operation has
> completed.

That is real buffering across the promotion gap — the behaviour the AWS financial
customer case study cited in [[aws-aurora-global-database#Real-world reports]]
was buying. And one sharp edge in the same paragraph:

> During switchover or failover, the default endpoint of the proxy for the old
> primary cluster **still accepts write operations**. However, as soon as that
> cluster becomes a secondary cluster, all of the write operations fail.

So a proxy in the old primary Region is a **route for split-brain-shaped writes**
during the switch, and then a source of hard failures afterwards. If you deploy
proxies, deploy one per Region, and make the endpoint your application uses a
name you control.

**Recommendation:** do not add RDS Proxy purely for DR. Add it if connection
churn is independently a problem (Lambda callers, high connection rates, IAM auth
at scale). If you do add it, you must build the "which proxy endpoint" indirection
anyway — which pushes you toward the CNAME option below.

### Route 53 CNAME indirection — the case for and against

**For:**
- One name you control, under your own zone, in front of whatever the truth
  currently is. Works for the global writer endpoint, an RDS Proxy endpoint, or a
  detached standalone cluster endpoint after a B2 manual failover.
- **It is the only endpoint strategy that survives path B2.** If you ever have to
  detach-and-promote, the global writer endpoint ceases to exist and every
  application must be re-pointed — unless there is a CNAME, in which case it is
  one `ChangeResourceRecordSets`.
- Lets you cut over gradually (weighted records) during migrations.

**Against:**
- Adds your TTL on top of Aurora's. Two caches, not one.
- `ChangeResourceRecordSets` is a **Route 53 control-plane call in `us-east-1`**,
  which [[split-brain-and-fencing]] and the AWS Fault Isolation Boundaries
  whitepaper both list as a failover anti-pattern. For the US pair that is the
  Region you are failing away from.
- It is a thing that can be forgotten. The global writer endpoint updates itself;
  a CNAME updates when your automation, which lives somewhere, runs.

**Recommendation: use the global writer endpoint directly as the primary path,
and additionally create a CNAME that points at it** — with a 5-second TTL, and
used by applications. Steady state costs you one extra 5-second cache layer. In
exchange you own the name, which means the B2 path, an RDS Proxy introduction,
and any future migration are all one DNS change instead of a redeploy of every
service. The control-plane objection is real but applies only to the *B2/proxy*
branches, where you have no alternative anyway.

Set the record TTL to 5 seconds to match AWS's own guidance, and accept the
slightly higher query cost — it is rounding error against an instance-hour.

### Global Accelerator

[[aws-global-accelerator]] is the right tool for the web tier's regional
failover. **It is not a tool for Aurora.** Global Accelerator's endpoint types
are Application Load Balancers, Network Load Balancers, EC2 instances and Elastic
IPs — an RDS or Aurora cluster endpoint is not an accelerator endpoint type, and
AWS publishes no Aurora integration. You could in principle put an NLB with IP
targets in front of the writer's private IPs and accelerate that, but the writer's
IP changes on every in-Region failover as well as every cross-Region one, so you
would be building an IP-tracking control loop to replace a DNS name that already
does the job. **Do not.**

Where Global Accelerator *does* belong in this picture is one layer up: it fails
the *application* over between Regions quickly and without DNS, while Aurora's
global writer endpoint fails the *database* over. They are complementary and
should not be conflated. See [[aws-global-accelerator]] and [[aws-route53]].

---

## The RDS events to key automation off

**This is the most useful under-publicised thing in this note.** Aurora emits
specific, documented RDS events through every stage of a switchover or failover.
Subscribing to them turns a runbook full of `sleep` statements into a runbook
driven by facts, and it is how you avoid the "we waited two minutes and hoped"
step that shows up in every first-draft DR procedure.

> [!check] Verified 2026-09-22
> Every event ID in the tables below was checked against [Amazon RDS event
> categories and event messages for
> Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_Events.Messages.html).
> **All 21 IDs exist with the messages and categories shown.** None were wrong,
> none were invented. The `Category` column is added from that page and is what
> you actually filter an `aws_db_event_subscription` on.

### Switchover lifecycle

All are **DB cluster** events in category **`global failover`**.

| Event ID | Message | Why you care |
|---|---|---|
| `RDS-EVENT-0181` | "Global switchover to DB cluster *X* in Region *Y* started." | Start of clock. AWS's note adds: "The process can be delayed because other operations are running on the DB cluster" |
| `RDS-EVENT-0183` | "Waiting for data synchronization across global cluster members. Current lags behind primary DB cluster: *…*" | **The sync wait, with the lag value in the message.** This is how you watch a switchover drain |
| `RDS-EVENT-0423` | "Waiting for data synchronization with the target DB cluster. Current target DB cluster lag behind the primary DB cluster: *…*" | Same, target-specific |
| `RDS-EVENT-0182` | "Old primary DB cluster *X* in Region *Y* successfully shut down." | **The write fence landed.** AWS's note: "The old primary instance in the global database isn't accepting writes. All volumes are synchronized." |
| `RDS-EVENT-0184` | "New primary DB cluster *X* in Region *Y* was successfully promoted." | New writer exists. AWS's note: "The volume topology of the global database is reestablished with the new primary volume" |
| `RDS-EVENT-0185` | "Global switchover to DB cluster *X* in Region *Y* finished." | Done — but see the note: "Replicas might take long to come online after the failover completes" |
| `RDS-EVENT-0186` / `RDS-EVENT-0187` | "…is cancelled." / "…failed." | Your abort signals |

### Failover lifecycle

Also **DB cluster** events in category **`global failover`**.

| Event ID | Message |
|---|---|
| `RDS-EVENT-0238` | "Global failover to DB cluster *X* in Region *Y* completed." |
| `RDS-EVENT-0239` | "Global failover to DB cluster *X* in Region *Y* failed." |
| `RDS-EVENT-0240` | "Started resynchronizing members of DB cluster *X* in Region *Y* after global failover." |
| `RDS-EVENT-0241` | "Finished resynchronizing members of DB cluster *X* in Region *Y* after global failover." |
| `RDS-EVENT-0519` | "Global database failover to DB cluster *X* in global database *Y* completed. **The DB cluster has no instances. Create a DB instance to access your data.**" |

`RDS-EVENT-0519` is worth staring at. Its existence implies a failover can
*complete* against a cluster with no DB instances — i.e. a headless secondary —
leaving you with a promoted primary you cannot connect to until you create an
instance. That sits awkwardly beside the docs' flat statement that "Before you
can perform a switchover or failover to a headless secondary Aurora DB cluster,
you must add a DB instance to it." **Do not resolve this by guessing.** The safe
reading is: headless secondaries are not a supported failover target, this event
exists because the platform can still end up there, and the recovery instruction
is in the event message. It is one more reason [[aws-aurora-global-database]]'s
recommendation of one warm reader in production is the right one.

### Write fencing — the two events that tell you whether you have split-brain

These are **DB instance** events, category `notification, global database` —
confirmed against the DB instance events table. Note that they are the *only*
global-database events emitted against instances rather than clusters, which is
why the Terraform below needs two subscriptions:

| Event ID | Message | Meaning |
|---|---|---|
| `RDS-EVENT-0390` | "Attempt to block writes for DB cluster *X* in Region *Y* **succeeded**." | AWS's note: "Aurora began blocking writes at the storage layer in preparation for switchover or failover of an Aurora global database." |
| `RDS-EVENT-0391` | "Attempt to block writes for DB cluster *X* in Region *Y* **timed out**." | AWS's note, verbatim and worth quoting in the runbook: "Aurora wasn't able to block writes at the storage layer… **The switchover or failover will proceed but you might need to recover recently written data from the snapshot of the original primary cluster.**" |

**`RDS-EVENT-0391` is the "you may have split-brain" alarm and it should page.**
It is the single most decision-relevant event in the list: it tells you, during
the incident, whether the point-of-failure snapshot is a formality or the thing
your reconciliation will be built on. And note where it lands — "If the old
primary cluster is reachable on the network, Aurora records these events there.
If not, Aurora records the events on the new primary cluster." **So your event
subscription must exist in both Regions**, or in the worst case the event is
recorded in the Region you cannot reach.

### The DNS event

| Event ID | Source type | Category | Message |
|---|---|---|---|
| `RDS-EVENT-0397` | DB cluster | `global failover` | "Aurora finished changing the DNS name that the global writer endpoint resolves to." |

This is the trigger AWS tells you to use instead of a timer: "you can check for
the RDS event that informs you when Aurora observed the DNS changes for the
global writer endpoint… you can validate that your application also registered
the DNS change before restarting your application write traffic."

**Runbook step: wait for `RDS-EVENT-0397`, then restart the application.** Not
`sleep 60`.

Also relevant at the instance level: `RDS-EVENT-0385` — "Cluster topology is
updated." AWS's note: "There are DNS changes to the DB cluster for the DB
instance. This includes when new DB instances are added or deleted, or there's a
failover."

### Two events that are really Terraform warnings in disguise

| Event ID | Category | Message | What it is really telling you |
|---|---|---|---|
| `RDS-EVENT-0518` | `maintenance` | "The engine version of DB cluster *X* has been changed from *A* to *B* **to align with the new primary cluster** *Y* **after a failover**." AWS: "No action required. This change is automatic and keeps your global database consistent." | **Aurora mutates `engine_version` on a member cluster, out of band, during a failover.** This is precisely why `lifecycle { ignore_changes = [engine_version] }` is load-bearing in [[aws-aurora-global-database#lifecycle blocks are load-bearing, not decoration]]. Without it, the first `terraform plan` after a failover proposes to change the engine version of your live database. |
| `RDS-EVENT-0517` | **`failure`** | "The *{{upgrade\_type}}* version upgrade for DB cluster *X* was canceled because **a failover occurred on the associated global database**. The DB cluster is now running engine version *V*. Retry the upgrade when the global database is available." | A failover during an upgrade **cancels the upgrade** and leaves the cluster on an indeterminate version. Which is exactly the state that disqualifies you from managed failover next time. Freeze upgrades during incidents |

Note the categories: `RDS-EVENT-0518` is `maintenance` and `RDS-EVENT-0517` is
`failure`, **neither of which is `global failover`**. A subscription filtered to
`global failover` alone will miss both of the events that matter most to
Terraform.

And one preventative event: `RDS-EVENT-0424`, category `maintenance`, DB cluster
— "The DB cluster *X* is running version *V*, which is higher than the target
upgrade version *W* for the global cluster. **We don't recommend having a
secondary cluster on a higher version than the global cluster, as it can cause
issues during failover or switchover.** Consider upgrading your global cluster to
match." Route this to the platform team's channel, not to nowhere. It is an early
warning that you are drifting off the managed-failover path.

### Wiring it up

```hcl
# One of these per Region. The subscription is regional; the events you need
# may be recorded in EITHER Region (see RDS-EVENT-0390/0391 above).
resource "aws_db_event_subscription" "global_db" {
  for_each = toset(["primary", "standby"])
  provider = each.key == "primary" ? aws.primary : aws.standby

  name      = "${var.name}-${var.environment}-gdb-events-${each.key}"
  sns_topic = aws_sns_topic.dr_events[each.key].arn

  source_type = "db-cluster"
  source_ids  = [each.key == "primary" ? aws_rds_cluster.primary.id : aws_rds_cluster.secondary[0].id]

  # "global failover" is the documented category covering RDS-EVENT-0181..0187,
  # 0238..0241, 0397, 0423, 0519.
  event_categories = ["global failover", "failover", "failure", "maintenance"]
}

# Write-fencing events are DB *instance* events, not cluster events.
resource "aws_db_event_subscription" "global_db_instances" {
  for_each = toset(["primary", "standby"])
  provider = each.key == "primary" ? aws.primary : aws.standby

  name        = "${var.name}-${var.environment}-gdb-inst-events-${each.key}"
  sns_topic   = aws_sns_topic.dr_events[each.key].arn
  source_type = "db-instance"
  # Deliberately no source_ids: catch every instance, including ones created
  # during the incident. A filtered subscription is a subscription that misses
  # the instance you made at 03:40.
  event_categories = ["notification", "availability", "failover", "recovery"]
}
```

**Do not filter the instance subscription by ID.** `RDS-EVENT-0390`/`0391` are
emitted against instances, and during an incident you will be creating
instances. A subscription scoped to the instances that existed at plan time will
miss exactly the ones you care about.

> [!warning] One thing in that snippet is *not* verified
> The `global failover` category string is confirmed — AWS's event table uses it
> verbatim for `RDS-EVENT-0181`–`0187`, `0238`–`0241`, `0397`, `0423` and `0519`.
> But `RDS-EVENT-0390`/`0391` are listed under the composite category
> **`notification, global database`**, and AWS's documentation does not state
> whether that is one subscribable category named `global database` plus
> `notification`, or a single string. **Resolve it in-account before trusting the
> instance subscription**, with:
> ```bash
> aws rds describe-event-categories --source-type db-instance
> aws rds describe-event-categories --source-type db-cluster
> ```
> This is the one place in this note where an automation would be built on a
> string that first-party docs do not pin down. Do not guess it.

---

## Failback

> Failover is a fifteen-minute event with executive attention. Failback is a
> multi-week project with none. [[split-brain-and-fencing#Failback]] makes the
> general argument; this section is the Aurora-specific mechanics.

### Aurora's failback is genuinely good, and it is the strongest single argument for Aurora

After a **managed** failover:

> As soon as that old primary Region is healthy and available again, **Aurora
> automatically adds it back to the global cluster as a secondary Region.** Thus,
> your Aurora global database's existing replication topology is maintained.

> After the original topology is restored, you can fail back your global database
> to the original primary Region by performing a **switchover** operation when it
> makes the most sense for your business and workload.

So the Aurora failback is: **wait, then run a switchover.** RPO 0. No reseed you
orchestrate. No second irreversible promotion. Compare with
[[aws-rds-postgres#Failback]], where a round trip costs two full cross-Region
reseeds and two planned outages.

**This is a change ticket, not a project.** That difference is worth restating in
[[rds-vs-aurora-decision]] because it is usually under-weighted next to the cost
comparison.

### But the "wait" is doing a lot of work — what actually gets rebuilt

The old primary does **not** re-attach with its existing data. Aurora throws the
volume away:

> To ensure the data is in a consistent state, **Aurora creates a new storage
> volume for the old primary Region after it recovers.** Before creating the new
> storage volume in the AWS Region, Aurora attempts to take a snapshot of the old
> storage volume at the point of failure.

This is a **full cross-Region copy of your entire database**, unattended, at
whatever rate AWS manages. The only published duration figure AWS gives for a
comparable rebuild is for *other* secondaries after a failover: "can take a few
minutes to **several hours**, depending on the size of the storage volume and the
distance between the Regions." **AWS publishes no separate figure for the old
primary's rebuild.** Assume the same order of magnitude and assume Montreal↔
Calgary is at the slow end.

Consequences:

1. **You are single-Region for the duration of the rebuild.** From the moment of
   failover until the old primary finishes rebuilding and catches up, there is no
   standby. If the *new* primary Region has an event in that window, you have no
   DR. Hours, not minutes. **This window belongs on the incident timeline and in
   the comms to the business**, because "we've failed over, we're fine" is not
   true yet.
2. **The old volume's data is gone unless you saved the snapshot.** Aurora
   replaces the volume. The `rds:unplanned-global-failover-*` snapshot is the
   only surviving copy, and it expires with the old cluster's retention period.
3. **Replicated-write I/O and cross-Region transfer charges spike** during the
   rebuild. A one-off cost, but a visible one — flag it to whoever watches the
   bill so it is not investigated as an anomaly. [[cost-model]].
4. **`RDS-EVENT-0240` / `RDS-EVENT-0241`** bracket the resynchronisation. Use
   them to know when you have a standby again, rather than guessing.

### The second data-loss window

Nobody plans for this one. The failback switchover is RPO 0 **with respect to the
new primary** — but it is still a **write outage**, and it has its own risks:

| Risk on the failback leg | Why |
|---|---|
| **Another short write outage.** "Your database is unavailable for a short time while the primary and selected secondary clusters are assuming their new roles." | You have to sell a second (planned) outage to a business that just absorbed an unplanned one |
| **Version drift accumulated during the incident.** `RDS-EVENT-0517` shows upgrades get cancelled by failovers; `RDS-EVENT-0518` shows Aurora rewrites engine versions to match. Post-incident, verify parity **before** attempting the failback switchover | If versions have drifted, the failback switchover is refused and you are on the detach-and-promote path *by choice*, which would be absurd |
| **Everything directional is still pointing the wrong way.** Parameter groups, alarms, dashboards, consumer enable/disable flags, SCPs, the write lease epoch, RI/Savings Plan coverage | [[split-brain-and-fencing]]'s step 6. Every one of these was inverted at failover and must be inverted back |
| **The reconciliation is not finished.** You may still be replaying lost writes into the new primary when you switch back | Switching primaries mid-reconciliation is how you lose the reconciliation |
| **Nobody is watching.** | The failback runs days later, off the bridge call, by whoever picked up the ticket |

**The genuine second data-loss window is not the switchover itself** — that is
RPO 0 — **it is the reconciliation.** If lost writes are being replayed into the
new primary while the old primary rebuilds, and the replay is not idempotent, and
the failback happens partway through, you can double-apply or drop records. The
mitigation is procedural, not technical: **finish the reconciliation before you
schedule the failback, and say so in the runbook.**

### Recommendation

**Fail back, deliberately, on a schedule you choose — but not immediately.**

Ordered:

1. Wait for `RDS-EVENT-0241` (resynchronisation finished) — you now have a
   standby again.
2. Verify engine version parity across both clusters (`describe-db-clusters`
   → `EngineVersion`, plus pending maintenance actions). Fix drift first.
3. Complete the reconciliation of anything recovered from the
   `rds:unplanned-global-failover-*` snapshot. **Do not skip to step 4.**
4. Invert every directional setting (parameter groups, alarms, consumer flags,
   fences) and verify with a `terraform plan` that is a no-op.
5. Schedule the switchover in a change window during low writes.
6. Run `switchover-global-cluster`, watching `RDS-EVENT-0181` → `0185`.
7. Post-switchover checklist: parameter groups, alarms, integrations, RDS Proxy,
   logical slots — the five things switchover does not carry across.

And a second-order recommendation that falls out of this: **because failback is
cheap on Aurora, run the switchover quarterly anyway.** It rehearses this exact
sequence with no incident attached, and it is the only way the checklist in step
7 stays true. AWS explicitly lists "regional rotation" as a switchover use case.

### Do not fail back on the manual (B2) path without rebuilding first

If you took the detach-and-promote path there is no global cluster to switch
over. AWS's instruction: "you add the old AWS Region to your new global database,
and then use the switchover process to switch its role." That means: build a new
global cluster around the promoted standalone cluster, attach the old primary
Region as a secondary (a full cross-Region seed), wait, **then** switch over.
Two cross-Region seeds total for the round trip — i.e. you have re-created the
RDS failback cost profile that Aurora exists to avoid. Another reason version
parity is a DR control, not a hygiene preference.

---

## Split-brain — the old primary is still writeable

[[split-brain-and-fencing]] covers the theory, the seven techniques and the
write-lease recommendation. This section is only the Aurora-specific facts.

### What Aurora does for you, exactly

> When you initiate a managed failover, Aurora also attempts to halt write
> traffic through the highly-available Aurora storage layer. We refer to this
> mechanism as "write fencing". If the process succeeds, Aurora emits an RDS
> Event letting you know that writes were stopped. In the unlikely event of
> multiple AZ failures in a Region, it's possible that the write fencing process
> doesn't succeed in a timely manner… **Because fencing writes is a best-effort
> attempt, it's possible that writes might be momentarily accepted in the old
> primary Region, causing split-brain issues.**

Three things to extract:

1. **It is real fencing, at the right layer.** The storage volume itself refuses
   writes. That is closer to Kleppmann's bar — enforcement at the protected
   resource — than anything else available in this estate. It does not depend on
   a security group rule propagating or an IAM policy converging.
2. **It is best-effort and it can time out**, and AWS tells you which event
   means which (`RDS-EVENT-0390` vs `RDS-EVENT-0391`).
3. **It only exists on the managed path.** Switchover quiesces the primary
   properly (`RDS-EVENT-0182`: "successfully shut down… isn't accepting writes").
   Managed failover attempts it. **Detach-and-promote does nothing at all.**

### Fencing by path

| Path | Fence | Quality |
|---|---|---|
| Switchover | Primary made read-only before anything else happens | **Guaranteed.** The only one |
| Managed failover | Storage-layer write fencing | **Best-effort**, observable via `RDS-EVENT-0390`/`0391` |
| Detach-and-promote | **None** | You own it entirely |

### What you must do regardless

AWS's own first recommendation, before any of its own machinery:

> To prevent writes from being sent to the primary cluster of Aurora Global
> Database, **take applications offline.**

The platform vendor's advice is that the fence you control beats the fence it
provides. That is [[split-brain-and-fencing]]'s Layer 1 (scale the primary's
writers to zero, opportunistically, `onFailure: Continue`) and Layer 0 (the
pre-armed application write lease, which is the only fence that works when the
primary is unreachable).

The Aurora-specific addition to that layering:

- **Reduce DNS TTL to 5 seconds** — AWS frames this explicitly as a *split-brain*
  control, not just an RTO one: "Although Aurora attempts to block writes in the
  old primary Region, the action is not guaranteed to succeed. **Reducing the DNS
  cache duration further reduces the likelihood of split-brain issues.**" A client
  that re-resolves quickly stops writing to the old primary quickly.
- **RDS Proxy in the old primary Region is a split-brain conduit** during the
  switch: "During switchover or failover, the default endpoint of the proxy for
  the old primary cluster still accepts write operations." If proxies exist, they
  need fencing too.
- **Watch for `RDS-EVENT-0391` and treat it as a data-integrity page**, not an
  informational event.

### The evidence-preservation step, which is time-critical

```bash
# DO THIS IN THE FIRST TEN MINUTES. The system snapshot expires with the OLD
# primary cluster's backup retention period.
SNAP=$(aws rds describe-db-cluster-snapshots --region "$PRIMARY_REGION" \
  --query "DBClusterSnapshots[?starts_with(DBClusterSnapshotIdentifier,'rds:unplanned-global-failover')] | [0].DBClusterSnapshotIdentifier" \
  --output text)

aws rds copy-db-cluster-snapshot --region "$PRIMARY_REGION" \
  --source-db-cluster-snapshot-identifier "$SNAP" \
  --target-db-cluster-snapshot-identifier "incident-${INCIDENT_ID}-point-of-failure" \
  --kms-key-id "$PRIMARY_KMS_KEY_ARN"
```

Two traps in that snippet:

- **The snapshot only exists if the primary Region's control plane is reachable
  enough for the copy call to succeed.** If it is not, retry on a loop for the
  duration of the incident. Put it in the orchestrator with
  `onFailure: Continue` and a retry, not as a manual "remember to do this".
- **The snapshot may not exist at all.** AWS says Aurora "*attempts* to take a
  snapshot… **If this operation is successful**, Aurora places this snapshot…".
  There is no guarantee. If it is absent, the lost writes are simply gone and the
  reconciliation is application-log-based or nothing.

Also copy it **cross-Region** if the old primary Region is expected to be
unavailable for a while — `copy-db-cluster-snapshot` into the standby Region
needs a destination-Region KMS key ([[aws-kms]]).

### And the epoch: capture the LSN

Immediately after promotion, against the new primary:

```sql
SELECT aws_region, highest_lsn_written FROM aurora_global_db_status();
```

That number is the divergence boundary. It is the Aurora analogue of
`pg_last_wal_receive_lsn()` and it is ten seconds of work that makes the
difference between a targeted reconciliation and a forensic one. See
[[split-brain-and-fencing#Reconciling divergent writes: the part with no tooling]].

---

## Limits, parity and what blocks a failover

### Topology limits

All verified 2026-09-22 against [Configuration requirements of an Amazon Aurora
global
database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.configuration.requirements.html).

| Limit | Value | AWS's wording |
|---|---|---|
| Secondary Regions | **10 maximum**, at least 1 required | "At least one secondary AWS Region is required, but an Aurora global database can have up to 10 secondary AWS Regions" |
| Writer instances | 1 in the primary, **0** in every secondary | Requirements table, "Writer instances" row |
| Readers per secondary cluster | **16 (total)**, vs 15 for a standalone cluster | Requirements table, "Read-only instances (Aurora replicas), per Aurora DB cluster": primary 15 (max), secondary 16 (total). The *why* is on the [Aurora Global Database overview page](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html): "The secondary cluster is read-only, so it can support up to 16 read-only DB instances rather than the usual limit of 15 for a single Aurora cluster" |
| Readers in the **primary** cluster | **15 − *s***, where *s* = number of secondary Regions | Requirements table row: "Read-only instances (max allowed, given actual number of secondary Regions) \| 15 - *s* \| *s* = total number of secondary AWS Regions" |
| Two clusters in the same Region | **Not allowed** | "no two Aurora DB clusters in an Aurora global database can be in the same AWS Region" |
| Cluster names | **Globally unique across Regions** | "The names you choose for each of your Aurora DB clusters must be unique, across all AWS Regions. You can't use the same name for different Aurora DB clusters even though they're in different Regions" |
| Instance class | **db.r5 or higher** | "An Aurora global database requires DB instance classes that are optimized for memory-intensive applications… We recommend that you use a db.r5 or higher instance class" |
| Serverless v2 floor | **8 ACUs in the primary Region** | "For a global database with Aurora serverless, the minimum recommended capacity for the DB cluster in the primary AWS Region is 8 ACUs" |

> [!note] The 15 − *s* rule is an obscure one worth knowing
> **Each secondary Region costs you one reader slot on the primary cluster.** With
> one secondary you can run 14 readers in the primary, not 15. Irrelevant at our
> scale, invisible in the docs unless you read the table carefully, and it will
> bite exactly one person, once, at the worst time.
>
> Note that the two sources sit on different pages: the *numbers* are in the
> configuration-requirements table, the *explanation* is in the "Advantages"
> section of the overview page. Both are cited above.

### Engine version parity — the control that decides which failover path you get

All of this section is verified against [Upgrading an Amazon Aurora global
database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-upgrade.html),
section "Patch level compatibility for managed cross-Region switchovers and
failovers" (checked 2026-09-22). Three tiers, and the difference between them is
the difference between B1 and B2:

| Operation | Version requirement |
|---|---|
| **Managed switchover / managed failover** | Same **major and minor**. Patch levels must be identical **unless** the version is on AWS's exception list |
| **Manual (detach-and-promote) failover** | Same **major and minor**. "In this case, the patch levels don't need to match" |
| Neither available | Different major or minor versions — you are restoring from a snapshot |

AWS's patch-level exception list for **Aurora PostgreSQL**, verbatim: version 15
or higher major versions; version 14.5 or higher minors; 13.8+; 12.12+; 11.17+.
On those, "you can perform managed cross-Region switchovers or failovers from a
primary DB cluster with one patch level to a secondary DB cluster with a different
patch level."

For **Aurora MySQL** the table says: "**No minor versions**. None of the Aurora
MySQL minor versions allow managed cross-Region switchovers or failovers with
differing patch levels between the primary and secondary DB clusters." If this
estate is ever on Aurora MySQL, patch parity is an absolute prerequisite.

> [!danger] The nastiest gotcha in this note, after the expiring snapshot
> AWS, verbatim, buried in the notes column of the patch-compatibility table:
>
> > **When you update a cluster in your global database to any of the following
> > patch versions, you won't be able to perform cross-Region switchovers or
> > failovers until all of the clusters in your global database are running one of
> > these patch versions or a newer one.**
> >
> > Patch versions 16.1.6, 16.2.4, 16.3.2, and 16.4.2
> > Patch versions 15.3.8, 15.4.9, 15.5.6, 15.6.4, 15.7.2, and 15.8.2
> > Patch versions 14.8.8, 14.9.9, 14.10.6, 14.11.4, 14.12.2, and 14.13.2
>
> **A routine patch applied to one Region silently disables managed failover for
> the whole global database until the other Region catches up.** No alarm fires.
> Nothing in the console says "DR is degraded". You find out when
> `failover-global-cluster` refuses, at 03:12, and you are on the B2 path.
>
> Mitigations, all of which you want:
> 1. **Patch the secondary first, then the primary, and never leave a gap
>    overnight.** AWS's own guidance for manual per-cluster upgrades: "upgrade all
>    secondary clusters before upgrading the primary cluster."
> 2. **Prefer the managed global minor upgrade** (Aurora PostgreSQL only):
>    `aws rds modify-global-cluster --global-cluster-identifier X --engine-version Y`,
>    which "orchestrates the upgrade across your primary cluster and all secondary
>    (mirror) clusters" and "automatically rolls back to the existing version" if
>    it fails. One operation, no window of mismatch. Note the caveat: "The managed
>    capability applies only to minor version upgrades. **Patch version upgrades
>    continue to use existing system-update maintenance actions**" — so the trap
>    above lives precisely in the category the managed upgrade does *not* cover.
> 3. **A daily automated parity check** that compares `EngineVersion` and pending
>    maintenance actions across both clusters and raises a ticket on divergence.
>    Ten lines of Lambda. This is the actual fix.
> 4. Subscribe to `RDS-EVENT-0424` ("secondary cluster on a higher version than
>    the global cluster… can cause issues during failover or switchover").

### What blocks a failover — the checklist

Ordered by how likely it is to be the thing that stops you:

| # | Blocker | How to detect it in advance | Fix |
|---|---|---|---|
| 1 | **Patch-level mismatch** between primary and secondary | Daily parity check; `describe-db-clusters --query 'DBClusters[].EngineVersion'` in both Regions; `describe-pending-maintenance-actions` | Align versions. Use `modify-global-cluster` for minors |
| 2 | **Headless secondary** — zero DB instances | `describe-db-clusters --query 'DBClusters[].DBClusterMembers'` | "you must add a DB instance to it" — and that wait is exactly what breaks a 15-minute RTO. Keep ≥1 warm reader in prod |
| 3 | **Switchover attempted against a dead primary** | n/a — it will simply refuse | Use `failover-global-cluster --allow-data-loss` |
| 4 | **`--allow-data-loss` omitted** | The call fails | It is mandatory on the failover path |
| 5 | **A Blue/Green deployment is mid-switchover** | `describe-blue-green-deployments` | "Global failover **is** supported during a blue/green switchover, but **Global switchover is not**." The target Region will "automatically roll back to the blue environment or roll forward to the green environment before the global failover occurs" — i.e. an extra, unbudgeted step |
| 6 | **An engine upgrade is in progress** | Cluster status; `RDS-EVENT-0177` | A failover cancels the upgrade (`RDS-EVENT-0517`) and leaves the version indeterminate. Freeze upgrades during incidents |
| 7 | **`rds.global_db_rpo` is set** on the *secondary's* parameter group | `describe-db-cluster-parameters --source user` | AWS: in a two-Region topology this "could cause Aurora to pause transactions" post-failover. Leave at `-1` |
| 8 | **Primary derived from an RDS for PostgreSQL replica** | Cluster lineage | "you can't create a secondary cluster… Attempts to do so time out". You never had a global database to fail over |
| 9 | **KMS key inaccessible** | Cluster status `inaccessible-encryption-credentials` | Terminal, with **no** `-recoverable` grace state on global databases. SCP-deny `kms:DisableKey` / `kms:ScheduleKeyDeletion`. [[aws-kms]] |
| 10 | **A target ARN typo / bare identifier instead of ARN** | Rehearsal | `--target-db-cluster-identifier` wants the ARN |

### Feature interactions

All quoted wording below was verified on 2026-09-22. Unless another page is
named, the source is the [Limitations of Amazon Aurora Global
Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html#aurora-global-database.limitations)
section.

| Feature | With Aurora Global Database | Consequence for failover |
|---|---|---|
| **Backtrack** | **Not supported.** Listed flatly under "Aurora Global Database currently doesn't support the following Aurora features: Backtracking in Aurora" | No "rewind 30 minutes" escape hatch after a bad deploy. PITR restore is your only in-place undo, and it is slow |
| **Cloning** | Aurora fast cloning is **Region-local**. [Cloning limitations](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Managing.Clone.html#Aurora.Managing.Clone.Limitations), verbatim: "You can't create a clone in a different AWS Region from the source Aurora DB cluster." | You cannot clone across Regions to seed or to test. Rehearsal clones live in one Region only |
| **Serverless v2** | Supported for readers. [Performance and scaling for Aurora serverless](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.setting-capacity.html), verbatim: "Aurora global databases – 8 ACUs (**applies only to the primary AWS Region**)". Same page lists "Aurora global databases" first among features that "can increase resource usage and **prevent the database from scaling down to minimum capacity**" | A Serverless v2 standby reader is warm (no instance-creation wait) but **will not be at production capacity the instant it is promoted**. AWS, verbatim: "The time it takes for an Aurora serverless DB instance to scale from its minimum capacity to its maximum capacity depends on the difference between its minimum and maximum ACU values… if you specify a relatively large maximum capacity and the DB instance spends most of its time near that capacity, consider increasing the minimum ACU setting." **AWS's own advice is to raise the floor.** Set it for the *post-promotion* load, not the idle load |
| **Blue/Green Deployments** | Supported, and it mirrors the whole topology. [Using Blue/Green Deployments for Amazon Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-bluegreen.html), verbatim: "including the primary cluster and all associated secondary regions across multiple AWS Regions"; switchover "with downtime typically under one minute" | **Verbatim: "Global failover is supported during a blue/green switchover, but Global switchover is not supported during a blue/green switchover."** And: "When you initiate a global failover during an RDS blue/green switchover, the target region automatically rolls back to the blue environment or rolls forward to the green environment before the global failover occurs." DR is maintained but the interaction adds an unbudgeted step |
| **Performance Insights** | Per-cluster, **not inherited**: "When you add a new secondary AWS Region to an Aurora global database that's already using Performance Insights, be sure that you enable Performance Insights in the newly added cluster" | After failover, if the standby never had PI enabled, you are debugging your new production database blind. Enable it on both clusters at build time. Note also "the associated Performance Insights URL is different for each DB instance" — bookmarked dashboards break |
| **Database Activity Streams** | "You start a database activity stream on each DB cluster separately. Each cluster delivers audit data to its own Kinesis stream within its own AWS Region" | Your audit trail moves Regions at failover. Compliance owners need to know. [[data-residency]] |
| **Aurora Auto Scaling** | **Not supported for secondary clusters** | No elastic ramp on the new primary. Adding capacity is a scripted `create-db-instance`, post-RTO |
| **Stop/start** | **Not available.** "You can't stop or start the Aurora DB clusters in your global database individually" | The non-prod overnight-shutdown saving disappears once a cluster joins a global database. Headless is the non-prod lever instead |
| **Secrets Manager integration** | **Unsupported.** "When you add a Region to a global database, you must first turn off Secrets Manager integration for the DB instance" | Collides with `manage_master_user_password = true`. Resolve before attaching a secondary. [[aws-secrets-manager]] |
| **Cluster cache management** | "isn't supported for Aurora PostgreSQL secondary DB clusters that are part of Aurora global databases" | The promoted cluster starts with a cold buffer cache. Expect a latency hump after promotion that is not a bug |
| **Renaming a member cluster** | **Not allowed while a member.** You *can* change the global cluster identifier and instance identifiers | After a B2 detach-and-promote your cluster is permanently named for the wrong Region |
| **Automatic minor version upgrade** | "the setting has no effect" on global database members | You own the patch calendar — which is exactly the thing that decides your failover path |

### `ca-west-1` parity, specifically

**Aurora Global Database itself: fully supported.** Canada West (Calgary) is in
the supported-Regions table for Aurora PostgreSQL 11 → 18 at identical version
floors to Canada (Central), and for Aurora MySQL 2/3/8.4. Write forwarding is
"available in every AWS Region where Aurora PostgreSQL-based global databases are
available", so that is covered too.

What still needs confirming in-account, none of which is a feature gap:

1. **Instance classes.** Global Database "requires DB instance classes that are
   optimized for memory-intensive applications… We recommend that you use a
   **db.r5 or higher** instance class." Burstable `db.t3`/`db.t4g` are therefore
   out, everywhere, including non-production. Verify what is orderable:
   ```bash
   aws rds describe-orderable-db-instance-options --region ca-west-1 \
     --engine aurora-postgresql --engine-version "$PINNED_VERSION" \
     --query 'OrderableDBInstanceOptions[].DBInstanceClass' --output text | tr '\t' '\n' | sort -u
   ```
   Run the same command against `ca-central-1` and diff. If the primary uses a
   class Calgary does not offer, the standby must use a different one — which is
   allowed (see below) but changes your post-promotion capacity story.
2. **Mixing instance classes across Regions is permitted.** Nothing in the
   configuration requirements forces the secondary to match the primary; the
   requirement is only "db.r5 or higher". **But a promoted standby serves
   production on whatever it is**, and Auto Scaling will not save you. Treat any
   deliberate undersizing as a documented, accepted post-RTO gap with a scripted
   `create-db-instance` to close it.
3. **`ca-west-1` is an opt-in Region.** Account enablement plus the STS
   credential-validity trap for cross-Region calls — set
   `AWS_STS_REGIONAL_ENDPOINTS=regional` in CI. See
   [[aws-rds-postgres#The STS opt-in-Region trap]].
4. **Reserved Instance / Savings Plan availability** for the standby reader,
   which runs 24/7 forever and is the most reservable thing in the estate.
   [[cost-model]].

**Headline for [[region-pair-selection]]: the CA pair passes the Aurora Global
Database parity check. The database is not the reason to reconsider Calgary.**

---

## A runbook skeleton

> Copy into [[failover-runbook-template]]. Decision gates are marked **◆**.
> Steps marked **∥** run concurrently with the step above them.
> Times are *allowances*, not measurements — replace them with your own game-day
> numbers before this is signed off.

### Header (fill in before the incident, not during)

```
PAIR:                eu | us | ca
GLOBAL_ID:           <global cluster identifier>
PRIMARY_REGION:      <region>            STANDBY_REGION:  <region>
PRIMARY_CLUSTER:     <identifier>        STANDBY_CLUSTER: <identifier>
STANDBY_CLUSTER_ARN: <full ARN — switchover/failover need the ARN>
GLOBAL_WRITER_ENDPOINT: <...>.global.rds.amazonaws.com
APP_CNAME:           db.<env>.internal.example.com   (TTL 5s)
FAILBACK POLICY:     ping-pong via switchover  (Aurora default — see below)
APPROVERS:           <≥3 named people>
```

### Phase 0 — pre-flight (≤ 60 s, and mostly pre-computed)

| # | Step | Gate |
|---|---|---|
| 0.1 | **Freeze the Terraform pipeline in both Regions.** | |
| 0.2 | Read `AuroraGlobalDBRPOLag` in the **standby** Region. Record the value and the timestamp. | |
| 0.3 | Confirm engine version parity: `describe-db-clusters --query 'DBClusters[].EngineVersion'` in both Regions. | **◆ If versions differ → you are on path B2. Say so out loud and switch runbooks.** |
| 0.4 | Confirm the standby has ≥1 `available` DB instance. | **◆ If headless → add an instance now and accept that RTO is blown.** |
| 0.5 | Confirm no Blue/Green switchover and no engine upgrade is in progress. | |

### Phase 1 — decide (◆ the one-way-ish door)

| # | Step | Gate |
|---|---|---|
| 1.1 | Is the primary Region's **database** actually the problem, or only its compute? | **◆ Compute-only → consider write forwarding + traffic shift instead of failover. No data loss, no promotion.** See [[#When it is genuinely useful: the partial failover]] |
| 1.2 | Is the primary healthy enough for a switchover? | **◆ Yes → `switchover-global-cluster` (RPO 0). This is almost never true in a real outage.** |
| 1.3 | Is lag actively falling and is the wait bounded at 5 minutes? | **◆ Yes → wait and re-check. No → proceed.** |
| 1.4 | **Named approver accepts the data loss**, quoting the figure from 0.2. | **◆ Record who, when, and the RPO number. This is the audit artefact.** |

### Phase 2 — fence (60–120 s, reversible — do this *before* 1.4 if you can)

| # | Step |
|---|---|
| 2.1 | **Take applications offline in the primary Region.** `kubectl --context $PRIMARY -n $NS scale deploy --all --replicas=0 --grace-period=0`. Run with "continue on failure" — the Region may be unreachable, which is fine |
| 2.2 | Set Lambda reserved concurrency to 0 for any writer functions in the primary |
| 2.3 | If a write lease is implemented: confirm expiry and **wait `L + skew`**. This is a wait, not an API call, and it cannot fail. [[split-brain-and-fencing]] |
| 2.4 | If RDS Proxy exists in the primary Region, fence it too — its default endpoint still accepts writes during the switch |

### Phase 3 — promote (the command)

```bash
# --region is the STANDBY's region.
aws rds --region "$STANDBY_REGION" failover-global-cluster \
  --global-cluster-identifier "$GLOBAL_ID" \
  --target-db-cluster-identifier "$STANDBY_CLUSTER_ARN" \
  --allow-data-loss
```

| # | Step |
|---|---|
| 3.1 | Issue the command above |
| 3.2 | **∥ Scale the standby's compute** — EKS node group and workload replicas. Do not wait for 3.1 |
| 3.3 | Watch for `RDS-EVENT-0390` (fence succeeded) or **`RDS-EVENT-0391` (fence timed out)**. **◆ If 0391 → raise a data-integrity workstream now; do not wait until after the incident** |
| 3.4 | Wait for `RDS-EVENT-0238` ("Global failover … completed") — **not** a fixed sleep |
| 3.5 | `aws rds wait db-cluster-available --region "$STANDBY_REGION" --db-cluster-identifier "$STANDBY_CLUSTER"` |

### Phase 4 — confirm and capture (≤ 60 s, and step 4.3 is time-critical)

| # | Step |
|---|---|
| 4.1 | `psql -h "$GLOBAL_WRITER_ENDPOINT" -c "SELECT pg_is_in_recovery();"` → expect `f` |
| 4.2 | **Capture the divergence boundary:** `SELECT aws_region, highest_lsn_written FROM aurora_global_db_status();` Paste into the incident channel |
| 4.3 | **Copy the point-of-failure snapshot to a manual snapshot.** `rds:unplanned-global-failover-*` expires with the OLD cluster's retention period. Retry on a loop if the primary Region is unreachable |

### Phase 5 — reconnect (this is where the RTO is actually spent)

| # | Step |
|---|---|
| 5.1 | Wait for **`RDS-EVENT-0397`** — "Aurora finished changing the DNS name that the global writer endpoint resolves to" |
| 5.2 | If using an app-owned CNAME, update it (TTL 5 s) |
| 5.3 | **Restart the application in the standby Region.** `kubectl --context $STANDBY -n $NS rollout restart deploy`. **Mandatory, not conditional.** Connection pools do not re-resolve DNS |
| 5.4 | Repoint RDS Proxy consumers to the new primary's proxy read/write endpoint, if proxies exist |
| 5.5 | Enable consumers: Lambda ESMs, EventBridge rules, queue workers — **after** 5.3, never before |

### Phase 6 — validate (◆ and it must be a real write)

| # | Step | Gate |
|---|---|---|
| 6.1 | Run the synthetic transaction: a real write that touches the database, the queue and the cache, and assert the result | **◆ Fail → rollback branch. "Pods are Running" is not validation** |
| 6.2 | Error rate and latency back to baseline | |
| 6.3 | Declare RTO met; record the elapsed time from the 1.4 approval | |

### Phase 7 — stabilise (post-RTO; none of this is inside the 15 minutes)

| # | Step |
|---|---|
| 7.1 | **Add capacity to the new primary.** No Auto Scaling on what was a secondary; script `create-db-instance` |
| 7.2 | Reconcile the promoted cluster's **DB cluster parameter group** against the old primary's |
| 7.3 | Re-point alarms and dashboards — the lag metrics have moved Region and cluster ID |
| 7.4 | Re-establish integrations: Secrets Manager, IAM, S3, Lambda |
| 7.5 | Re-establish any **logical replication slots** consumed by CDC/analytics |
| 7.6 | Confirm Performance Insights and Enhanced Monitoring are on in the new primary |
| 7.7 | Watch for `RDS-EVENT-0240` → **`RDS-EVENT-0241`**: the old Region has rebuilt and you have a standby again. **Until 0241, you have no DR.** Tell the business |
| 7.8 | Restore the Terraform pipeline. Run `plan` and read every line. Never `-auto-approve` |
| 7.9 | Reconcile the lost writes from the manual snapshot copy |
| 7.10 | Schedule the failback switchover — **after** 7.9 completes |

### The rollback points

| After phase | Reversible? |
|---|---|
| 0 freeze | Trivially |
| 2 fence | **Yes — the last cheap abort point.** Un-fence and carry on |
| 3 promote (managed) | **Partially.** The topology survives and you can switch back later — but the writes lost in the gap are lost, and the old primary's volume is being replaced |
| 3 promote (B2 detach) | **No.** The global cluster is destroyed |
| 5 reconnect | Mechanically yes, practically no |

## Open questions

These are genuine unknowns, not unfinished citation work. Each one is a thing the
docs do not answer and a game day or an in-account test would.

1. **Does `SELECT FOR UPDATE` actually work over write forwarding on Aurora
   PostgreSQL?** Two AWS pages contradict each other (both cited in
   [[#SQL you cannot use]]). Only an in-account test resolves it. Only matters if
   write forwarding is ever turned on.
2. **What is the exact subscribable event category for `RDS-EVENT-0390`/`0391`?**
   AWS's table shows the composite string "notification, global database" and
   does not say how to express that in an `aws_db_event_subscription`. Resolve
   with `aws rds describe-event-categories --source-type db-instance` before the
   fencing alarm is built. **This is the only automation input in this note that
   first-party docs do not pin down.**
3. **What is our actual switchover duration?** AWS publishes no figure — it says
   only "Your database is unavailable for a short time" and that duration is
   proportional to lag. The quarterly switchover is the only way to learn it.
4. **What is our actual managed-failover duration?** AWS says "within a few
   minutes" and its launch post says "typically a minute". Neither is an SLA and
   no independent measurement of a real Region event exists publicly (see
   the "Could not verify" list in [[#Sources]]). A game day is mandatory before the
   15-minute RTO is signed off.
5. **How long does the old primary's rebuild take for our data volume on the
   Montreal↔Calgary leg?** AWS publishes "a few minutes to several hours" for
   *other secondaries* and **no figure at all** for the old primary's rebuild.
   Until 0241 fires you have no DR, so this number sizes a real risk window.
6. **Are there any logical replication slots on the cluster today?** CDC,
   Debezium, or an analytics sink consuming from this database turns failover
   into a multi-system recovery. Find out at design time.
7. **Which instance classes are orderable for `aurora-postgresql` in
   `ca-west-1`?** Run the `describe-orderable-db-instance-options` diff in
   [[#`ca-west-1` parity, specifically]]. Not a feature gap, but it sizes the
   post-promotion capacity story.
8. **RTO definition** — 15 minutes from *incident start* or from *decision to
   fail over*? The runbook below assumes the latter (the clock starts at the 1.4
   approval). Flagged in [[research-brief]] as an open thread for the user.

## Still to research

The citation pass is **done** — see [[#Sources]]. What remains is new content,
not verification:

1. **Terraform implementation** for the event subscriptions and alarms in this
   note, beyond the snippet in [[#Wiring it up]]. The cluster/instance Terraform
   itself lives in [[aws-aurora-global-database#Terraform implementation]] and
   should not be duplicated here.
2. **Cost** — the standby reader's 24/7 instance-hours, `AuroraGlobalDBReplicatedWriteIO`,
   cross-Region transfer, and the one-off spike during an old-primary rebuild.
   Belongs in [[cost-model]]; this note should link, not restate.
3. **Migration path** — deliberately deferred to
   [[aws-aurora-global-database#Migration path from single-region]] and
   [[rds-vs-aurora-decision]]. This note is the *operational* slice and has no
   separate migration story.
4. **Decisions to make** — the three real forks in this note (write forwarding
   on/off, CNAME indirection or not, quarterly switchover or not) each carry a
   recommendation inline. Consolidating them into one table would help a reviewer
   but adds no new research.

## Sources

Every URL below was fetched and read on **2026-09-22**. Quotes in the note are
verbatim from these pages unless explicitly labelled as an inference or an
estimate.

### First-party AWS documentation

- [Supported Regions and DB engines for Aurora global databases](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.Aurora_Fea_Regions_DB-eng.Feature.GlobalDatabase.html)
  — **the `ca-west-1` finding, and the single most load-bearing check in this
  note.** Canada West (Calgary) is present in both the Aurora PostgreSQL and
  Aurora MySQL tables, with a row identical to Canada (Central).
- [Using switchover or failover in Amazon Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html)
  — **the primary source for this note.** The three switchover use cases; the
  "healthy state" restriction; the switchover mechanism step order; "Your
  database is unavailable for a short time"; managed failover's "doesn't wait for
  data to synchronize"; "guarantees that the data is in a transactionally
  consistent state"; the write-fencing paragraph including where the events are
  recorded; the `rds:unplanned-global-failover-{{name}}-{{timestamp}}` snapshot
  and the note that it "is a system snapshot that's subject to the backup
  retention period configured on the old primary cluster"; **"Typically, the
  chosen secondary cluster assumes the primary role within a few minutes"**; the
  secondary-rebuild "a few minutes to several hours"; the full manual
  detach-and-promote procedure; "take applications offline"; the DNS-TTL-5-seconds
  advice; the `--region` semantics for both commands; `--allow-data-loss`
  ("Explicitly make this a failover operation instead of a switchover
  operation"); the five post-promotion configuration items; and the complete
  `rds.global_db_rpo` behaviour including the two-Region warning and the
  20–2,147,483,647 second range.
- [Cluster-level CloudWatch metrics for Amazon Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMonitoring.Metrics.html#Aurora.AuroraMySQL.Monitoring.Metrics.clusters)
  — all five `AuroraGlobalDB*` metric names, units and descriptions. **Source of
  the one factual correction in this pass:** every one of them, including
  `AuroraGlobalDBDataTransferBytes`, "is available only in secondary AWS Regions".
- [Monitoring an Amazon Aurora global database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-monitoring.html)
  — the `aurora_global_db_status()` and `aurora_global_db_instance_status()`
  column lists and definitions; the `-1` on the primary's row in AWS's own
  example output; the hour-long-transaction worked example showing
  `durability_lag_in_msec` climbing while `rpo_lag_in_msec` stays at 0; the
  Performance Insights non-inheritance text; and the Database Activity Streams
  per-Region Kinesis statement.
- [Amazon RDS event categories and event messages for Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_Events.Messages.html)
  — **all 21 event IDs used in this note, verified individually.** Confirms
  categories: `global failover` for `RDS-EVENT-0181`–`0187`, `0238`–`0241`,
  `0397`, `0423`, `0519` (DB cluster); `notification, global database` for
  `RDS-EVENT-0390`/`0391` (DB **instance**); `maintenance` for `0518` and `0424`;
  `failure` for `0517`; `notification` for `0385` (DB instance).
- [Configuration requirements of an Amazon Aurora global database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.configuration.requirements.html)
  — the 10-secondary maximum, the writer/reader counts, **the 15 − *s* primary
  reader rule verbatim**, the same-Region prohibition, the globally-unique naming
  rule, "db.r5 or higher", and the 8-ACU Serverless v2 primary floor.
- [Using Amazon Aurora Global Database (overview and limitations)](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
  — "dedicated infrastructure, with latency typically under a second"; the
  16-vs-15 reader explanation; and the limitations list: Backtracking
  unsupported, Aurora Auto Scaling unsupported for secondary clusters, no
  individual stop/start, Secrets Manager unsupported, cluster cache management
  unsupported on APG secondaries, no renaming a member cluster, the terminal
  `inaccessible-encryption-credentials` state with no `-recoverable` variant, the
  RDS-PostgreSQL-replica-derived-primary restriction ("Attempts to do so time
  out"), and automatic minor version upgrade having "no effect".
- [Upgrading an Amazon Aurora global database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-upgrade.html)
  — the patch-level compatibility table (APG 15+/14.5+/13.8+/12.12+/11.17+;
  Aurora MySQL "No minor versions"); **the exact list of poisoned patch versions
  (16.1.6, 16.2.4, 16.3.2, 16.4.2; 15.3.8, 15.4.9, 15.5.6, 15.6.4, 15.7.2,
  15.8.2; 14.8.8, 14.9.9, 14.10.6, 14.11.4, 14.12.2, 14.13.2)**; "upgrade all
  secondary clusters before upgrading the primary cluster"; the
  `modify-global-cluster` managed minor upgrade, its automatic rollback, its
  Aurora-PostgreSQL-only scope, and the caveat that "Patch version upgrades
  continue to use existing system-update maintenance actions"; and the
  major-upgrade block when `rds.global_db_rpo` is on.
- [Using write forwarding in an Aurora PostgreSQL global database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-write-forwarding-apg.html)
  — the version/Region availability statement; the four
  `apg_write_forward.consistency_mode` values with AWS's description of each and
  the `SESSION` default; the full parameter table (scope, type, default, range)
  including `max_forwarding_connections_percent` = Global / int / 25 / 1–100 and
  the `max_connections = 800` worked example; the supported and unsupported SQL
  lists (including `SELECT FOR UPDATE` as **supported**); `SERIALIZABLE`
  unsupported; read-only access mode disabling forwarding; all eight
  `AuroraForwarding*` CloudWatch metrics; all seven `IPC:AuroraWriteForward*`
  wait events; "Enabling or disabling write forwarding doesn't cause downtime or
  a reboot"; the maintenance-window override; and the
  `GlobalWriteForwardingStatus` values and the meaning of `null`.
- [Connecting to Amazon Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-connecting.html)
  — the global writer endpoint format and auto-update behaviour; the
  recommendation to move off the primary's cluster endpoint; "can take a long
  time depending upon your Domain Name Service (DNS) caching duration"; the
  cross-VPC IP reachability caveat; the rename-breaks-the-endpoint warning; the
  best-effort/split-brain sentence; the secondary cluster endpoint showing status
  `inactive`; and **the write-forwarding statement that contradicts the
  engine-specific page on `SELECT FOR UPDATE`**.
- [Using RDS Proxy with Aurora global databases](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-proxy-gdb.html)
  — the request-queuing behaviour through the promotion gap; the old primary's
  proxy default endpoint still accepting writes during the switch and then
  failing; the `MaxConnectionsPercent` reduction guidance (which names only the
  Aurora **MySQL** parameter); and "RDS Proxy doesn't support the `SESSION` value
  for the `apg_write_forward.consistency_mode` parameter".
- [Removing a cluster from an Amazon Aurora global database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-detaching.html)
  — detaching makes a cluster "a standalone provisioned Aurora DB cluster with
  full read/write capabilities"; "The Aurora global database might remain in the
  **Databases** list, with zero Regions and AZs"; and the ARN form of
  `remove-from-global-cluster`.
- [Using Blue/Green Deployments for Amazon Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-bluegreen.html)
  — topology mirroring "including the primary cluster and all associated
  secondary regions across multiple AWS Regions"; "downtime typically under one
  minute"; and verbatim: "Global failover is supported during a blue/green
  switchover, but Global switchover is not supported during a blue/green
  switchover."
- [Performance and scaling for Aurora serverless](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.setting-capacity.html)
  — "Aurora global databases – 8 ACUs (applies only to the primary AWS Region)";
  global databases listed among features that "prevent the database from scaling
  down to minimum capacity"; and the min-to-max scaling-time paragraph with
  AWS's own advice to raise the minimum ACU setting.
- [Cloning a volume for an Amazon Aurora DB cluster](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Managing.Clone.html#Aurora.Managing.Clone.Limitations)
  — "You can't create a clone in a different AWS Region from the source Aurora DB
  cluster."
- API reference pages for the three operations named throughout:
  [`SwitchoverGlobalCluster`](https://docs.aws.amazon.com/AmazonRDS/latest/APIReference/API_SwitchoverGlobalCluster.html),
  [`FailoverGlobalCluster`](https://docs.aws.amazon.com/AmazonRDS/latest/APIReference/API_FailoverGlobalCluster.html),
  [`RemoveFromGlobalCluster`](https://docs.aws.amazon.com/AmazonRDS/latest/APIReference/API_RemoveFromGlobalCluster.html)
  — all three names confirmed by the CLI/API subsections of the
  switchover/failover and detaching pages.

### Inherited from the parent note

These were verified during the research for
[[aws-aurora-global-database]] and are cited there in full; this note relies on
them but did not re-fetch them in this pass:

- [Amazon Aurora Global Database supports failover — AWS What's New, Aug 2023](https://aws.amazon.com/about-aws/whats-new/2023/08/amazon-aurora-global-database-failover/)
  — the "typically a minute" phrasing quoted in [[#Timing]].
- [How a large financial AWS customer implemented HA and DR for Amazon Aurora PostgreSQL using Global Database and Amazon RDS Proxy — AWS Database Blog](https://aws.amazon.com/blogs/database/how-a-large-financial-aws-customer-implemented-ha-and-dr-for-amazon-aurora-postgresql-using-global-database-and-amazon-rds-proxy/)
  — the RDS Proxy buffering architecture referenced in [[#RDS Proxy]].

### Could not verify

Stated plainly rather than dressed up. Nothing in this list is presented as fact
in the note body.

- **No independent, non-AWS measurement of an unplanned Aurora Global Database
  cross-Region failover during a real Region event.** Every timing figure in
  public circulation originates with AWS. This was also the parent note's
  finding and nothing new was found. Recorded as a finding, not a gap in
  searching. The 15-minute RTO therefore rests on vendor claims plus whatever
  your own game day measures.
- **AWS publishes no switchover duration figure.** Only "Your database is
  unavailable for a short time" and the statement that duration is proportional
  to lag. Any specific number seen elsewhere is a blog illustration, not a
  commitment.
- **AWS publishes no duration for the old primary's rebuild after a managed
  failover.** The "a few minutes to several hours" figure is stated for
  *additional secondary* clusters. Applying it to the old primary is a **labelled
  inference** in [[#But the "wait" is doing a lot of work — what actually gets
  rebuilt]], not a quote.
- **The subscribable event-category string for `RDS-EVENT-0390`/`0391`.** AWS's
  table shows "notification, global database" and no page explains how to express
  it in a subscription. Flagged in-place and in [[#Open questions]]; must be
  resolved with `describe-event-categories` before the fencing alarm is trusted.
- **Write forwarding availability in `ca-west-1` is an inference**, not a stated
  fact. AWS says write forwarding is available wherever APG global databases are,
  and Calgary is in that table. Both halves are verified; the conjunction is
  ours. AWS publishes no per-Region write-forwarding table.
- **The `SELECT FOR UPDATE` contradiction is unresolved.** Both AWS pages are
  cited. Neither is dated. No third-party source was found that settles it.
- **Two of the non-Aurora claims in [[#The cached-DNS / connection-pool problem,
  in detail]]** remain uncited: the JVM `networkaddress.cache.ttl` defaults
  (stated in the note as "cache forever on older JREs with a SecurityManager,
  30 seconds commonly cited on modern JDKs") and the CoreDNS 30-second default
  cache TTL. Both are widely-documented general platform behaviour rather than
  Aurora facts, and the note's advice — pin the value explicitly rather than rely
  on the default — is correct regardless of which default applies. But they are
  **not verified here**. The third claim in that subsection, `tcp_retries2`, *is*
  now verified against the Linux kernel documentation and was **corrected** in
  the process (924.6 s lower bound, not "13–15 minutes").
