---
title: ElastiCache (Redis OSS / Valkey) — Multi-Region
service: elasticache
tags: [service, multi-region, elasticache, redis, valkey, cache]
status: researched
replication: native (Global Datastore, async) | manual (snapshot→S3→restore) | none (cold start)
rpo_achievable: "sub-second with Global Datastore; ~hours with snapshot export; N/A (irrelevant) for a pure cache"
rto_achievable: "< 1 min promotion once the decision is made, IF the secondary cluster is pre-provisioned; 20-40 min if you restore from a snapshot"
meets_targets: conditional
updated: 2026-09-20
---

# ElastiCache (Redis OSS / Valkey) — Multi-Region

> This note covers the **managed** side. The self-hosted-on-EKS side, and the
> head-to-head decision, live in [[redis-self-managed-vs-elasticache]]. The
> durable-datastore variant lives in [[aws-memorydb]]. Read all three; this
> estate runs Redis both ways today and the right answer differs per workload.

## TL;DR

- **ElastiCache Global Datastore is available in `ca-west-1` (Calgary).** AWS
  added it on 2025-04-29 along with 14 other regions, and the current
  prerequisites page lists "Canada — Canada Central and Canada West (Calgary)".
  All three Helios pairs (`eu-west-1`→`eu-west-2`, `us-east-1`→`us-west-2`,
  `ca-central-1`→`ca-west-1`) are supported. This was the thing most likely to
  fail the parity check and it does not. Feed this into
  [[region-pair-selection]].
- **Global Datastore hits the targets comfortably — on paper.** AWS states RPO
  "typically under one second" and RTO "typically under a minute" for the
  cross-region promotion. Against RPO 2h / RTO 15m that is ~7000x and ~15x of
  headroom respectively. The RTO number is only the *promotion*; the 15 minutes
  is consumed by detection, decision, DNS/endpoint change and client reconnect,
  not by ElastiCache.
- **The secondary is read-only until you promote it, and that is the design
  constraint that actually bites.** An application deployed warm in the standby
  region cannot write to its local cache. It either sits idle, or it writes
  cross-region to the primary (adding 60–100 ms to every `SET`), or you accept
  that the standby app is not really running until failover. Read the AWS
  session-store blog's advice of 200–500 ms cross-region write timeouts as the
  tell: this is not a cheap thing to do casually.
- **For a pure cache the honest answer is: do not replicate it.** Deploy an
  empty cluster in the standby, let it start cold, and spend the effort on
  making the *database* survive the thundering herd
  ([[aws-rds-postgres]], [[aws-aurora-global-database]]). Global Datastore costs
  ~$0.19/node-hour in the standby plus $0.02/GiB replication egress to protect
  data that is by definition reconstructible. The decision hinges entirely on
  **cache-as-cache vs cache-as-datastore** — sessions, rate limits, distributed
  locks and Redis-as-a-queue are not caches and must not be treated as one.
- **The one that will bite:** you cannot add an *existing* cluster as a
  secondary. AWS wipes it. Bootstrapping a global datastore is
  "existing cluster becomes primary, new empty cluster becomes secondary", which
  is fine for the migration — but it means the Terraform for the standby is a
  *create*, and a careless `terraform apply` that flips
  `global_replication_group_id` on an existing standby replication group is a
  data-loss event. Second bite: **Global Datastore and ElastiCache's new
  durability feature are mutually exclusive.** You get one or the other.

---

## Does this service cross regions at all?

ElastiCache is a **regional** service in every respect that matters:

| Thing | Scope | Consequence for the standby |
|---|---|---|
| Replication group / cluster | Regional | Must be created separately in `eu-west-2` etc. |
| Cache subnet group | Regional, references regional subnet IDs | Per-region resource. No cross-region reference possible. |
| Parameter group | Regional | Must be duplicated. Global Datastore *propagates parameter edits* to member clusters (see Gotchas) but the group object itself is regional. |
| Security group | Regional (it's a VPC SG) | Per-region. See [[aws-vpc-networking]]. |
| KMS key for at-rest encryption | Regional | A key in `eu-west-1` cannot encrypt a cluster in `eu-west-2`. See [[aws-kms]] and [[kms-when-to-use-multi-region-keys]]. |
| AUTH token / RBAC users & user groups | Regional | **Not replicated by Global Datastore.** See below. |
| Endpoint DNS name | Regional, includes region token (`…usw2.cache.amazonaws.com`) | Application config must be region-aware or fronted by your own CNAME. See [[aws-route53]]. |
| Snapshot (backup) | Regional | Exportable to S3 in the *same* region only; you move it with S3 replication. |
| **Global Datastore (global replication group)** | **Multi-region construct** | The one native cross-region primitive. |

Two clusters in two regions are otherwise completely unrelated objects. There is
no "global endpoint". Nothing resolves across regions for you. Even with a
global datastore in place, each member cluster keeps its own regional endpoint
and your application picks one.

### The Global Datastore object model

A **global replication group** is a thin control-plane object that owns 1
primary regional replication group and up to 2 secondary regional replication
groups. It is created in the primary's region but is addressable from any member
region's ElastiCache API. Its ID is `<aws-generated-prefix>-<your-suffix>` —
AWS prepends a random 5-character prefix you do not control, which matters for
Terraform (`global_replication_group_id_suffix` is what you set;
`global_replication_group_id` is what you get back).

Data flows one way: primary region → secondary regions, asynchronously, over the
AWS backbone.

---

## Engine reality check: Redis OSS vs Valkey vs the licence mess

This changed twice in two years and getting it wrong in a design doc is
embarrassing, so here is the verified state.

**Timeline (all verified):**

1. **2024-03-20** — Redis Ltd relicensed Redis from the 3-clause BSD licence to a
   dual RSALv2 / SSPLv1 model, from Redis 7.4 onward. The explicit target was
   managed-service providers.
2. **2024-03-28** — the Linux Foundation announced **Valkey**, a fork of the last
   BSD-licensed Redis (7.2.4), backed by AWS, Google, Oracle and Ericsson among
   others.
3. **2024-10** — AWS shipped **ElastiCache for Valkey** as a first-class engine,
   alongside Redis OSS and Memcached.
4. **2025-05-01** — Redis Ltd added **AGPLv3** as a third licence option from
   Redis 8.0, partially reversing course (Redis is "open source again", in
   antirez's words). AGPLv3 is OSI-approved; it is still not BSD and it is still
   copyleft, so it does not restore the status quo ante for a cloud provider.

**What AWS actually offers today (verified against the ElastiCache pricing page
and the public AWS pricing API):**

- ElastiCache for **Valkey** — the strategic engine. AWS prices it **20% below**
  other node-based engines and **33% below** Redis OSS on Serverless. Verified in
  the pricing API: `cache.r7g.large` in `eu-west-1` is **$0.243/hr** for Redis
  and **$0.1944/hr** for Valkey — exactly 20%.
- ElastiCache for **Redis OSS** — still offered, capped at the pre-relicence
  versions (4.0.10 through 7.1). AWS does not and cannot ship Redis 7.4+ or
  Redis 8 under the OSS engine.
- **Extended Support pricing for Redis OSS now exists in the pricing API.** For
  `cache.r7g.large` in `eu-west-1` there are SKUs reading
  `$0.194 per Hr for ExtendedSupportYr1_Yr2-NodeUsage` and
  `$0.389 per Hr for ExtendedSupportYr3-NodeUsage`. That is an on-top surcharge
  that roughly **doubles** the node rate by year 3. Read this as AWS pricing
  Redis OSS users toward Valkey.
- Existing **Redis OSS reserved nodes automatically apply to Valkey nodes** in
  the same family and region, and because Valkey is 20% cheaper you get 20% more
  value out of an existing RI. There is no reservation reason to stay on Redis
  OSS.

**Global Datastore supports both engines.** The docs phrase it as "Valkey or
Redis OSS". Engine version must match across all member clusters — the global
replication group controls versioning, and adding a cluster to a global
datastore **automatically disables auto-minor-version-upgrade and you cannot
re-enable it** while it is a member. You now own patching manually for the whole
global set.

**Recommendation for Helios:** standardise on **Valkey 8.x or later** before
building the global datastores. Doing the engine migration and the
region migration in one change is tempting but doubles the blast radius; do the
engine first (it is an in-place engine change on a single-region cluster, much
lower risk), then build the standby. Rationale: 20% cheaper on every node
including the idle standby, no Extended Support cliff, and it is the engine AWS
is actually investing in — durability (below) is Valkey-only.

---

## The new wrinkle: ElastiCache durability (Valkey 9.0+), and why it blocks Global Datastore

AWS shipped **durability for ElastiCache for Valkey** in June 2026. It gives a
node-based Valkey 9.0+ cluster a **Multi-AZ transactional log**, which is the
same architectural idea that made [[aws-memorydb]] a "database" rather than a
cache. Two modes:

- **Synchronous writes** — persisted to the log across ≥2 AZs *before* the client
  is acknowledged. Zero data loss. Write latency goes from microseconds to
  single-digit milliseconds.
- **Asynchronous writes** — persisted after acknowledgement. Microsecond writes,
  at risk of losing **up to 10 seconds** of uncommitted data. If the primary
  cannot persist to the log for >10 s it starts **rejecting writes** rather than
  silently accumulating loss.

It is priced as a separate line item on top of node usage. Verified in the
pricing API: `SyncDurability-NodeUsage:cache.r7g.large` in `eu-west-1` is
**$0.035/hr**, on top of the $0.1944/hr Valkey node — roughly an 18% uplift.

**The limitation that matters here, verbatim from the docs:**

> Durability is not supported with Global Datastores, Outposts, Local Zones, or
> data tiering.

So this is a genuine fork, and it is new enough that nobody in the company will
know about it:

| | Durability enabled | Global Datastore |
|---|---|---|
| Survives AZ loss with zero/≤10s data loss | Yes | No better than normal async replication |
| Survives **region** loss | **No** | Yes |
| Engine | Valkey 9.0+ only | Valkey or Redis OSS 5.0.6+ |
| Cluster mode | **Cluster-mode-enabled only** (CMD not supported) | Either |
| Serverless | No | No |
| Cost uplift | ~+18% (sync) on node rate | Second region's nodes + $0.02/GiB egress |
| Can you have both | **No** | **No** |

Other durability limitations worth writing down because they are traps:
durability can only be **enabled at cluster creation** and **cannot be disabled**
afterwards; you cannot enable it on an existing non-durable cluster; it requires
Multi-AZ with ≥1 replica per shard; it forces encryption-at-rest on; it caps
write throughput at 100 MiBps per primary node; and online migration from
self-hosted Valkey/Redis into a durable cluster is not supported (relevant to
[[redis-self-managed-vs-elasticache]] — you cannot lift-and-shift the EKS Redis
straight into a durable cluster).

For Helios the region requirement wins: **if a Redis needs to survive a region
loss, durability is off the table for that cluster.** If a Redis genuinely needs
both, that is a signal it should be [[aws-memorydb]] (which gives durability
*and* multi-region) or, more likely, a signal that the data in it belongs in
Postgres or DynamoDB.

---

## The question that comes before all of this: does the cache need to survive failover?

This is the section to read first and the one most likely to save money. Cache
is the only service family in this vault where **"do nothing" is frequently the
correct engineering answer**, and the vault should say so plainly rather than
reflexively reaching for a replication feature.

### Cache-as-cache

Properties: every key is derivable from a system of record. A miss costs latency,
not correctness. Typical contents: rendered fragments, query results, serialised
DTOs, reference data, feature flags.

**What happens if the standby starts cold:** on failover, hit rate goes from
~95% to 0% and every request falls through to the origin. The risk is **not** the
cache — it is that the database behind it takes the full unshielded read load at
the exact moment you are also failing the database over. That is a
[[lessons-and-antipatterns]] classic: the cache didn't fail, the cache's absence
killed the database.

**The correct mitigations are not replication.** In order of value for money:

1. **Request coalescing / singleflight** in the application so 5,000 concurrent
   misses on the same key become one origin read. This is the single highest-value
   change and it protects you against cold-start *and* eviction storms *and*
   node replacement in steady state.
2. **Pre-warming at failover.** Run a job in the standby that populates the top-N
   hot keys before you cut traffic over. If you know your hot set, this takes
   seconds and gets you most of the hit-rate back.
3. **Staged traffic shift** — 10% / 50% / 100% via weighted records in
   [[aws-route53]] rather than an instant 0→100 cutover, so the cache fills
   while the origin load ramps.
4. **Scale the standby database reads up before the cutover**, not after.
5. Only then: replicate the cache.

**Recommendation for cache-as-cache: no Global Datastore.** Deploy an empty,
correctly-sized cluster in the standby (it must be warm-provisioned to meet RTO
15m — you cannot create a replication group at failover time, it takes minutes
and you do not want to discover a capacity constraint at 3am) and accept a cold
start. Budget the saved replication egress into singleflight work.

### Cache-as-datastore

Properties: the key is the only copy. Losing it is a correctness or
user-visible-behaviour event. Typical contents:

| Pattern | What loss actually does | Does it need replication? |
|---|---|---|
| **Session store** | Every logged-in user is logged out mid-failover | Usually **no** — AWS's own session-store blog calls sessions "soft state": a failed session read means re-authentication, not data loss. Acceptable if and only if the business accepts a mass re-login on a regional DR event. If it doesn't, replicate. |
| **Rate limiting / quota counters** | Counters reset to zero; every client gets a fresh budget at the worst possible moment | **Usually no, but size for it.** The failure mode is a brief over-permit, not corruption. Dangerous only if the limiter is the thing protecting a fragile downstream. |
| **Distributed locks** (Redlock, `SET NX`) | **Locks vanish. Two holders. Corruption.** | **Neither cold start nor async replication is safe.** Async replication can *resurrect* a lock that was released, or lose one that was held. This is the [[split-brain-and-fencing]] problem and Redis is the classic wrong tool for it. Move to a fencing-token scheme backed by [[aws-dynamodb]] conditional writes. Do not solve this with Global Datastore — it does not solve it. |
| **Job queue** (Redis Lists / Streams, Sidekiq, BullMQ, Celery broker) | **Enqueued-but-unprocessed jobs are lost.** Orders, emails, webhooks, payments. | **Yes, or move it.** See [[messaging-in-flight-data-loss]]. The right answer is usually "this should have been [[aws-sqs]]", but within a 15-minute RTO you replicate it and plan the migration separately. |
| **Idempotency keys / dedup sets** | Duplicate side effects — double charge, double email | **Yes.** Or back them with DynamoDB conditional writes, which is cheaper and correct. |
| **Leaderboards / counters that are the SoR** | Silent data loss | **Yes** — and question why the system of record is a cache. |

**Recommendation for cache-as-datastore:** Global Datastore, *or* move the data
to a service designed for it. The second option is usually correct and is always
cheaper in the long run.

### The audit that has to happen first

Nobody can answer "does the cache need to survive failover" globally, because a
real estate has one ElastiCache cluster per service and they are all different.
The prerequisite work is a **keyspace audit per cluster**, and it is a
half-day of work:

```bash
# Per cluster, against a *replica* endpoint so you don't stall the primary.
# SCAN + TYPE + TTL sampled over a few thousand keys tells you almost everything.
redis-cli -h <replica-endpoint> --tls --scan --count 1000 | head -20000 > keys.txt
while read -r k; do
  printf '%s\t%s\t%s\n' "$k" "$(redis-cli -h <ep> --tls type "$k")" \
         "$(redis-cli -h <ep> --tls ttl "$k")"
done < keys.txt | awk -F'\t' '{split($1,p,":"); print p[1]"\t"$2"\t"($3==-1?"NO-TTL":"ttl")}' \
     | sort | uniq -c | sort -rn
```

The signal you are looking for: **key prefixes with no TTL (`-1`)**. A key with no
expiry is, almost by definition, not a cache. If a cluster is >20% no-TTL keys by
count, it is a datastore wearing a cache's name and it belongs in the
replicate-or-migrate column.

Record the result per cluster in a table and drive the rest of the decision from
it. Put the output in `05-cost/` alongside the standby cost model; the audit is
what turns an abstract "replicate everything" number into a real one.

---

## Replication / mirroring options

### Option 0 — Do nothing but deploy an empty cluster (cold standby cache)

- **RPO:** N/A — all data lost, by design.
- **RTO:** 0 for the cache itself (it is already there and serving); the cost is
  degraded hit rate for as long as it takes to refill.
- **Cost:** just the standby nodes. No cross-region egress, no Global Datastore
  coupling, no version lock-step, no promotion step in the runbook.
- **Correct for:** cache-as-cache, and for session stores where re-authentication
  is acceptable.
- Crucially, "do nothing" still means **pre-provisioned**. An absent cluster
  fails RTO 15m on its own — creating a multi-node replication group is a
  multi-minute control-plane operation and depends on capacity being available in
  the standby region's AZs at that moment.

### Option 1 — Global Datastore (native, async)

- **RPO:** AWS states "typically under one second". Monitor the actual figure
  with the CloudWatch metric AWS documents for this
  (`ReplicationLag` on the secondary; the re:Post article on monitoring
  cross-region replication lag in Global Datastore is the canonical reference).
- **RTO:** promotion "typically under a minute", via
  `aws elasticache failover-global-replication-group`.
- **Constraints:** every member cluster must have the **same node type, same
  engine version, same number of shards, and the same number of primary nodes**.
  Replica counts may differ per region — this is the cost lever (see Cost).
- **Direction:** strictly one-way. The secondary is read-only.
- **Bootstrapping:** existing cluster → primary. **You cannot attach an existing
  populated cluster as a secondary; AWS wipes it.**
- **Regions:** up to 2 secondaries. VPC required (no EC2-Classic, no Local
  Zones).
- **No IPv6.**
- **No cross-account.** Every member cluster must be in the same AWS account —
  relevant if the estate uses per-region or per-environment accounts, which many
  cookiecutter monorepos do. Check this early; it can invalidate the whole
  approach. See [[aws-iam]].
- **No auto-failover across regions.** AWS is explicit: "ElastiCache doesn't
  support autofailover from one AWS Region to another." Cross-region promotion
  is always a human or a runbook decision. For Helios that is *correct* — see
  [[failover-orchestration]] and [[split-brain-and-fencing]] — but it means the
  15-minute clock includes human detection and decision time.

### Option 2 — Snapshot export to S3, cross-region replicate the object, seed on failover

The cheap option that still clears RPO 2h, and it is genuinely underrated.

- ElastiCache backups produce `.rdb` files. `aws elasticache copy-snapshot`
  exports to **an S3 bucket in the same region as the backup**, then you use
  **S3 Cross-Region Replication** (or `aws s3 cp`) to land it in the standby
  region. See [[aws-s3]].
- You can take up to **20 manual backups per day** per the Well-Architected lens,
  which puts a comfortably sub-2-hour snapshot cadence within reach without
  touching automatic-backup windows. A snapshot every 60–90 minutes clears RPO
  2h with margin.
- Take backups **against a read replica**, not the primary — AWS's own
  Well-Architected guidance, because RDB fork/serialise on a busy primary causes
  latency spikes and memory pressure.
- **Restoring is not fast.** Seeding a cluster from an `.rdb` is a create-time
  operation (`snapshot_arns` on the replication group), so on failover you are
  creating a cluster and loading it. For a multi-GB dataset this is comfortably
  **20–40 minutes** and it **fails RTO 15m**.
- Therefore snapshot/restore is the right choice only where you would otherwise
  have chosen Option 0 anyway but want an "oh no, we actually needed that data"
  escape hatch. It is a data-preservation mechanism, not a failover mechanism.
- **Not available for data-tiering clusters** (`r6gd` etc.) — export to S3 is
  explicitly unsupported there.
- **IAM trap:** the caller needs `elasticache:CopySnapshot` *and*
  `s3:ListAllMyBuckets` scoped to `*`. AWS documents that scoping
  `ListAllMyBuckets` to a bucket ARN produces the unhelpful error "Elasticache
  was unable to validate the authenticated user has access on the S3 bucket". If
  a least-privilege policy generator tightened that, this is why the export
  fails. See [[aws-iam]].
- **S3 bucket policy trap:** the bucket policy principal is the *regional*
  service principal `<region>.elasticache-snapshot.amazonaws.com`. You need one
  policy statement per region, not one generic one.

### Option 3 — Dual-write from the application

Write to both regions' caches from the application. Occasionally proposed, almost
never right.

- Doubles write latency or requires async fire-and-forget (in which case you have
  reimplemented replication, badly, without ordering guarantees).
- Cross-region write from the standby is ~60–100 ms in the EU pair. The AWS
  session-store blog's recommendation of **200–500 ms cross-region write
  timeouts** is the honest measure of what you are signing up for.
- Divergence is silent and unmonitored.
- **Not recommended.** If you find this in the estate, treat it as tech debt and
  log it in [[lessons-and-antipatterns]].

### Option 4 — External replication tools (`riot`, `redis-shake`, MIGRATE)

Third-party or self-built continuous replication into the standby. Covered in
[[redis-self-managed-vs-elasticache]] because it is mostly relevant there. For
managed ElastiCache it is strictly worse than Global Datastore: you own the
tool, its EC2/EKS host, its monitoring, its lag, and its failure modes, to
achieve something AWS does natively in the same regions.

---

## Cluster mode enabled vs disabled, and whether it constrains Global Datastore

**It does not block Global Datastore either way.** Both cluster-mode-enabled
(CME) and cluster-mode-disabled (CMD) clusters can be members of a global
datastore. But the details differ and two of them are real:

1. **Shard count must match across regions** for CME. Scaling is done *at the
   global datastore level* (modify the global replication group and all regional
   members scale together, without interruption per the docs), which is actually
   nicer than it sounds — but it means you cannot run a 12-shard primary and a
   3-shard standby to save money. **The only per-region cost lever is replica
   count.**
2. **Pub/sub behaves differently.** This is documented and surprising:
   - CMD: pub/sub is **fully** propagated. Events published on the primary
     region's primary reach subscribers in secondary regions.
   - CME: **non-keyspace events are region-local only**. Only *keyspace*
     notifications propagate cross-region.

   If any service in the estate uses Redis pub/sub for cross-service fan-out
   (cache invalidation broadcasts are the usual one) and the cluster is CME,
   **standby-region subscribers will not receive those messages** while the
   primary is live. This will not show up in any test that only exercises the
   primary region. Flag it. The correct home for cross-region fan-out is
   [[aws-sns]] / [[aws-eventbridge]], not Redis pub/sub.
3. **Durability requires CME** (CMD is explicitly unsupported), whereas Multi-AZ
   auto-failover in CMD requires manual disabling to promote a replica. The
   general AWS Well-Architected recommendation is "in almost all cases deploy
   with cluster mode enabled", with **≥2 replicas per shard** and, for sharded
   workloads, **≥3 shards** so the Redis/Valkey cluster protocol can reach
   quorum on primaries during failover.

**Recommendation:** CME with ≥2 replicas per shard in the primary, ≥1 replica per
shard in the standby (the minimum Global Datastore allows when auto-failover is
on). If anything in the estate is CMD today, moving it is a separate migration
and should not be bundled with the multi-region work.

---

## ElastiCache Serverless: does it do cross-region?

**No.** Verified two ways:

- The ElastiCache pricing page states plainly that Global Datastore is **not
  currently available with ElastiCache Serverless**.
- The durability docs independently confirm the serverless gap ("Durability is
  not supported with ElastiCache Serverless").

Serverless caches *can* be backed up and exported to S3 (`export-serverless-cache-snapshot`),
so **Option 2 works for Serverless** — snapshot to S3, replicate the object,
restore into a standby Serverless cache. That is the only cross-region story
Serverless has.

Also note: **the public AWS pricing API index for ElastiCache in `ca-west-1`
contains no ECPU/serverless usage types at all** (nor do the `eu-west-1`,
`eu-west-2` or `ca-central-1` indexes — serverless appears to be priced under a
different product family, so absence here is *not* proof of regional
unavailability). Do not conclude from this that Serverless is missing in
Calgary; verify on the console or with
`aws elasticache describe-serverless-caches --region ca-west-1` before relying
on it. Recorded as an **open question**, not a finding.

**Recommendation:** if any workload that needs cross-region DR is on Serverless
today, it must move to node-based to get Global Datastore. Serverless is a good
fit for spiky dev/test caches and for caches you are happy to lose; it is not a
fit for the DR-relevant ones.

---

## RPO / RTO analysis

### Against RPO 2h

| Mechanism | Stated / measured RPO | Meets 2h? |
|---|---|---|
| Global Datastore | "typically under one second" (AWS FAQ) | Yes, with ~7200x margin |
| Snapshot every 60–90 min → S3 CRR | 60–90 min + S3 replication time | Yes, with margin |
| Automatic daily backup only | up to 24h | **No** |
| Cold standby (no replication) | Total loss | N/A for cache-as-cache; **fails** for cache-as-datastore |

RPO 2h is so loose relative to Global Datastore that **RPO is not the reason to
buy Global Datastore.** If the only requirement were RPO, hourly snapshots into
S3 would do. Global Datastore is bought for RTO and for not having to restore.

### Against RTO 15m — where the time actually goes

The 15 minutes is not consumed by ElastiCache. Budget for the EU pair:

| Step | Time | Automatable? |
|---|---|---|
| Detect the regional impairment | 2–5 min | Partly — alarms, but the "is this regional or is it us" judgement is human |
| Decision to fail over | 1–5 min | **No.** This is the human gate. See [[failover-orchestration]]. |
| `failover-global-replication-group` → secondary becomes writable | **< 1 min** (AWS-stated) | Yes |
| Endpoint/DNS change so applications point at the promoted cluster | 1–3 min incl. TTL | Yes — Route 53 record updated by a Lambda on the ElastiCache SNS event, which is exactly the pattern in the AWS session-store blog |
| Client libraries drop stale connections and reconnect | 10 s – 2 min | **Depends entirely on client config** — see below |
| Cache refill to useful hit rate | minutes to hours | Not on the RTO clock, but very much on the database's clock |

**Verdict: conditional yes.** ElastiCache's own contribution to RTO is under a
minute. The failure modes are (a) the human decision gate, (b) client libraries
that cache DNS or hold dead connections, and (c) the standby cluster not
existing yet.

**The client-config point deserves emphasis** because it is the most common cause
of a "successful" failover that the application does not notice:

- **Cluster-mode-disabled clients** must have a socket timeout so they notice the
  old primary is gone and re-resolve the endpoint. AWS says this explicitly in
  the Well-Architected lens.
- **Cluster-mode-enabled clients** are responsible for detecting topology change
  themselves, via the library's own topology-refresh setting (e.g. Lettuce's
  periodic and adaptive refresh, which is **off by default** — a notorious
  footgun).
- **JVM DNS caching**: `networkaddress.cache.ttl` defaults can pin a resolved IP
  for the life of the JVM in some configurations. If the application is JVM-based
  this must be set to something like 5–60 seconds.
- Validate all of this with `aws elasticache test-failover` in a lower
  environment *before* you need it. Note the limits: max 15 shards per rolling
  24 hours, and **AWS may block this API during large-scale operational events**
  — i.e. exactly during a real regional event. It is a test tool, not a DR tool.

### Within-region AZ failure vs region failure

Worth separating because people conflate them:

| | AZ / node failure | Region failure |
|---|---|---|
| Mechanism | Multi-AZ automatic failover | Global Datastore manual promotion |
| Trigger | Automatic, ElastiCache-detected | **Human** |
| Time to writable | AWS: "typically just a few seconds" for the promotion itself | AWS: "typically under a minute" |
| Endpoint change needed | **No** — ElastiCache propagates the DNS name of the promoted replica to the primary endpoint; the reader endpoint is auto-updated | **Yes** — different region, different endpoint |
| Data loss | "a small amount… due to replication lag" (async) — unless durability is on, in which case zero (sync) or ≤10s (async) | Sub-second of writes |
| Gotcha | A **customer-initiated reboot of the primary does not trigger failover**, and a rebooted primary comes back **empty**, causing replicas to clear their copies too — a documented total-data-loss path that has nothing to do with regions | Old primary becomes a secondary; see Failback |
| Gotcha | If the whole AZ is down, the replacement replica is **only created when the AZ returns** — you run with reduced redundancy for the duration of the AZ event | Standby capacity must already exist |

The AZ story is strong and automatic. The region story is strong but manual. Do
not let the quality of the AZ story create the impression that the region story
is also automatic — it is not, and AWS says so.

---

## Warm standby shape

What exists in `eu-west-2` while `eu-west-1` is healthy:

| Component | State while primary is healthy | Cost |
|---|---|---|
| Cache subnet group | Exists, references standby private subnets | Free |
| Security group | Exists, rules reference standby VPC CIDRs / SGs | Free |
| Parameter group | Exists, identical values to primary's | Free |
| KMS key (regional, or MRK replica) | Exists, key policy grants the standby's ElastiCache | ~$1/mo per key, + MRK replica cost |
| Secret holding the AUTH token / RBAC creds | Exists, replicated by Secrets Manager | ~$0.40/mo per replica |
| **Replication group (secondary)** | **Running, read-only, receiving replication** | **Full node price** |
| Route 53 private hosted zone record (`redis.<svc>.internal`) | Exists, pointing at the *primary* region's endpoint (or at the local one, per strategy) | Negligible |
| CloudWatch alarms on `ReplicationLag` | Armed | Negligible |

**There is no scale-to-zero for ElastiCache.** A node-based cluster costs the
same idle as loaded. The only levers are node type, node count (replicas per
shard), and reservations.

One decision to make explicitly: **which endpoint does the standby application
point at while the primary is healthy?**

- **Point at the local (read-only) secondary** — reads work, writes fail. Only
  viable if the standby app is genuinely passive (no traffic, health checks only).
  If it serves any real traffic, every write path 500s.
- **Point at the primary cross-region** — everything works, at 60–100 ms per
  operation. Sane for a warm standby doing occasional synthetic checks; not sane
  for real traffic.
- **Point at a CNAME you control** (`redis.<svc>.internal` in a private hosted
  zone per region) and flip it at failover — **recommended**. It is the only one
  that makes the failover a single DNS change rather than a config deploy, and it
  matches the SNS→Lambda→Route 53 pattern AWS demonstrates in its multi-region
  session-store post. See [[aws-route53]].

---

## Terraform implementation

Provider aliases for both regions, per [[provider-aliases-vs-separate-stacks]].
Module signature shaped for a cookiecutter multi-env monorepo, per
[[module-patterns]].

### The module

`modules/elasticache-valkey/variables.tf`:

```hcl
variable "name" {
  description = "Logical cluster name, e.g. \"sessions\". Region/env suffixes are added by the module."
  type        = string
}

variable "env" {
  description = "Environment slug from cookiecutter, e.g. \"prod\"."
  type        = string
}

# ---- The knob that decides everything in this note ----
variable "cross_region_strategy" {
  description = <<-EOT
    How this cache survives a regional failure.
      "none"            - standby cluster exists but starts cold. Correct for cache-as-cache.
      "global_datastore"- native async replication, secondary is read-only until promoted.
      "snapshot"        - periodic RDB export to S3 + CRR; restore-on-failover. Fails RTO 15m
                          but preserves data. Use where "none" is nearly right.
    See 02-services/aws-elasticache-redis.md for how to choose.
  EOT
  type        = string
  default     = "none"

  validation {
    condition     = contains(["none", "global_datastore", "snapshot"], var.cross_region_strategy)
    error_message = "cross_region_strategy must be one of: none, global_datastore, snapshot."
  }
}

variable "workload_class" {
  description = <<-EOT
    Documentation-and-policy field. One of: "cache", "session", "ratelimit",
    "lock", "queue", "system_of_record". Drives the guard rails below and,
    more importantly, forces a human to answer the question.
  EOT
  type        = string

  validation {
    condition = contains(
      ["cache", "session", "ratelimit", "lock", "queue", "system_of_record"],
      var.workload_class
    )
    error_message = "workload_class must be one of: cache, session, ratelimit, lock, queue, system_of_record."
  }
}

variable "node_type" {
  description = "Must be a Global Datastore-supported family if cross_region_strategy is global_datastore: M5/M6g/M7g/M8g/R5/R6g/R6gd/R7g/R8g/C7gn/C8gn, size large and above. No t-class."
  type        = string
  default     = "cache.r7g.large"
}

variable "num_node_groups" {
  description = "Shard count. Must match across all Global Datastore members."
  type        = number
  default     = 2
}

variable "replicas_per_node_group_primary" {
  type    = number
  default = 2
}

variable "replicas_per_node_group_standby" {
  description = "Cost lever: Global Datastore allows a different replica count per region. Minimum 1 when automatic_failover is enabled on the primary."
  type        = number
  default     = 1
}

variable "engine_version" {
  description = "Valkey. Pin the minor; Global Datastore membership disables auto minor upgrade and you own patching."
  type        = string
  default     = "8.1"
}

variable "primary_region"  { type = string }
variable "standby_region"  { type = string }
variable "primary_subnet_ids" { type = list(string) }
variable "standby_subnet_ids" { type = list(string) }
variable "primary_kms_key_arn" { type = string }
variable "standby_kms_key_arn" {
  description = "MUST be a key in the standby region. A primary-region key ARN will fail. See 02-services/aws-kms.md."
  type        = string
}
variable "auth_token_secret_arn" {
  description = "Secrets Manager secret (replicated to the standby region) holding the AUTH token. See 02-services/aws-secrets-manager.md."
  type        = string
}
```

`modules/elasticache-valkey/main.tf`:

```hcl
terraform {
  required_version = ">= 1.9"
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = ">= 5.70, < 7.0"
      configuration_aliases = [aws.primary, aws.standby]
    }
  }
}

locals {
  base = "${var.name}-${var.env}"
  gds  = var.cross_region_strategy == "global_datastore"

  # A lock or a queue must not be left to start cold. Fail the plan, loudly.
  _guard = (
    contains(["lock", "queue", "system_of_record"], var.workload_class)
    && var.cross_region_strategy == "none"
  ) ? tobool("workload_class=${var.workload_class} cannot use cross_region_strategy=none. See 02-services/aws-elasticache-redis.md.") : true
}

# ---------------------------------------------------------------- primary ----

resource "aws_elasticache_subnet_group" "primary" {
  provider   = aws.primary
  name       = "${local.base}-primary"
  subnet_ids = var.primary_subnet_ids
}

resource "aws_elasticache_parameter_group" "primary" {
  provider = aws.primary
  name     = "${local.base}-primary"
  family   = "valkey8"

  # Eviction policy is the single most important cache-vs-datastore signal in
  # the config. allkeys-lru == cache. noeviction == somebody is storing state.
  parameter {
    name  = "maxmemory-policy"
    value = var.workload_class == "cache" ? "allkeys-lru" : "noeviction"
  }
}

data "aws_secretsmanager_secret_version" "auth_primary" {
  provider  = aws.primary
  secret_id = var.auth_token_secret_arn
}

resource "aws_elasticache_replication_group" "primary" {
  provider = aws.primary

  replication_group_id = "${local.base}-euw1"
  description          = "${var.name} (${var.env}) — primary — class=${var.workload_class}"

  engine         = "valkey"
  engine_version = var.engine_version
  node_type      = var.node_type
  port           = 6379

  parameter_group_name = aws_elasticache_parameter_group.primary.name
  subnet_group_name    = aws_elasticache_subnet_group.primary.name
  security_group_ids   = var.primary_security_group_ids

  num_node_groups         = var.num_node_groups
  replicas_per_node_group = var.replicas_per_node_group_primary

  multi_az_enabled           = true
  automatic_failover_enabled = true

  at_rest_encryption_enabled = true
  kms_key_id                 = var.primary_kms_key_arn
  transit_encryption_enabled = true
  auth_token                 = jsondecode(data.aws_secretsmanager_secret_version.auth_primary.secret_string)["token"]
  auth_token_update_strategy = "ROTATE"

  # Snapshot strategy: these are ALSO the seed for cross_region_strategy="snapshot".
  snapshot_retention_limit = 7
  snapshot_window          = "03:00-04:00"

  apply_immediately = false

  lifecycle {
    # The global replication group owns versioning for member clusters; the
    # provider ignores engine/engine_version/parameter_group_name changes on a
    # member anyway, but being explicit stops diff noise in a templated repo.
    ignore_changes = [engine_version, num_cache_clusters]
  }
}

# --------------------------------------------------- global replication ------

resource "aws_elasticache_global_replication_group" "this" {
  count    = local.gds ? 1 : 0
  provider = aws.primary

  global_replication_group_id_suffix = local.base
  primary_replication_group_id       = aws_elasticache_replication_group.primary.id

  # NOTE: the real ID is <aws-random-5-char-prefix>-<suffix>. You do not control
  # the prefix. Consume the computed attribute, never reconstruct the string.
}

# ---------------------------------------------------------------- standby ----

resource "aws_elasticache_subnet_group" "standby" {
  provider   = aws.standby
  name       = "${local.base}-standby"
  subnet_ids = var.standby_subnet_ids
}

resource "aws_elasticache_parameter_group" "standby" {
  provider = aws.standby
  name     = "${local.base}-standby"
  family   = "valkey8"

  parameter {
    name  = "maxmemory-policy"
    value = var.workload_class == "cache" ? "allkeys-lru" : "noeviction"
  }
}

resource "aws_elasticache_replication_group" "standby" {
  provider = aws.standby

  replication_group_id = "${local.base}-euw2"
  description          = "${var.name} (${var.env}) — standby — class=${var.workload_class}"

  # When attached to a global replication group, engine/version/node_type are
  # inherited from the global group and MUST NOT be set here.
  global_replication_group_id = local.gds ? aws_elasticache_global_replication_group.this[0].global_replication_group_id : null

  # Only set these on the non-GDS ("none"/"snapshot") path.
  engine         = local.gds ? null : "valkey"
  engine_version = local.gds ? null : var.engine_version
  node_type      = local.gds ? null : var.node_type

  num_node_groups         = var.num_node_groups
  replicas_per_node_group = var.replicas_per_node_group_standby

  parameter_group_name = aws_elasticache_parameter_group.standby.name
  subnet_group_name    = aws_elasticache_subnet_group.standby.name
  security_group_ids   = var.standby_security_group_ids

  multi_az_enabled           = true
  automatic_failover_enabled = true

  at_rest_encryption_enabled = true
  kms_key_id                 = var.standby_kms_key_arn # regional key — NOT the primary's
  transit_encryption_enabled = true

  # The standby takes no backups of its own while it is a read-only secondary.
  snapshot_retention_limit = 0

  lifecycle {
    ignore_changes = [engine_version, num_cache_clusters]

    # Flipping global_replication_group_id on a live standby, or removing it,
    # is a data-affecting operation. Make Terraform refuse to do it silently.
    prevent_destroy = true
  }

  timeouts {
    create = "90m"
    update = "90m"
    delete = "40m"
  }
}

output "primary_endpoint" {
  value = aws_elasticache_replication_group.primary.configuration_endpoint_address
}

output "standby_endpoint" {
  value = aws_elasticache_replication_group.standby.configuration_endpoint_address
}

output "global_replication_group_id" {
  value = local.gds ? aws_elasticache_global_replication_group.this[0].global_replication_group_id : null
}
```

### Terraform-specific notes and traps

- **Default timeouts on `aws_elasticache_global_replication_group` are 60m
  create / 60m update / 20m delete.** These are not arbitrary — global datastore
  operations genuinely take tens of minutes. Your CI job timeout must exceed
  them or you will get a half-applied global datastore and a confusing state.
- **The provider deliberately ignores `engine`, `engine_version` and
  `parameter_group_name` changes on a replication group once it is a member of a
  global replication group**, because the global group owns versioning. Expect
  the plan to look "wrong" and do not fight it.
- **`parameter_group_name` on the global replication group is required when
  upgrading a major engine version.** Note this in the upgrade runbook —
  engine upgrades on a global datastore are driven from the global object, not
  the members.
- **`ForceNew` on the standby is the thing to watch.** The provider will replace
  an `aws_elasticache_replication_group` on changes to several immutable
  arguments; on the standby, replacement means the cluster is destroyed and a new
  empty one created and re-seeded from the primary. For a *secondary* that is
  survivable (it re-syncs) but it is a multi-hour window with no DR cover. Keep
  `prevent_destroy = true` on the standby and force these through a deliberate
  `terraform state` + targeted apply with a human present.
- **Known provider issue:** hashicorp/terraform-provider-aws issue
  [#40786](https://github.com/hashicorp/terraform-provider-aws/issues/40786) —
  "`aws_elasticache_replication_group` for valkey engine is stuck in resource
  replace loop". Check whether this is resolved in the provider version you pin
  **before** putting a Valkey global datastore in a pipeline that can apply
  without review. A perpetual-replace diff on a standby cluster in an automated
  pipeline is a DR outage waiting to happen.
- **Cookiecutter fit:** `cross_region_strategy` and `workload_class` are the two
  variables that belong in the per-service cookiecutter template, defaulted to
  `"none"` and *unset* respectively. Making `workload_class` required with no
  default forces every service owner to answer the cache-vs-datastore question
  once, at template time, in code review — which is exactly where you want that
  conversation to happen rather than at 3am.

---

## Migration path from single-region

The estate is live in one region. The good news: **making an existing cluster the
primary of a global datastore is non-destructive and does not replace the
resource.**

### Step 1 — Prerequisites (no production impact)

1. **Run the keyspace audit** (above) on every cluster. Classify each as
   `cache` / `session` / `ratelimit` / `lock` / `queue` / `system_of_record`.
   Only the last four force replication.
2. **Verify node type eligibility.** Global Datastore requires M5/M6g/M7g/M8g/
   R5/R6g/R6gd/R7g/R8g/C7gn/C8gn, **size `large` and above**, and **explicitly
   excludes burstable `t`-class**. If any production cluster is on `t3`/`t4g` —
   and in a mature estate at least one will be — it needs a node-type change
   first. That is an in-place scaling operation on ElastiCache, not a
   destroy/recreate, but it is a separate change with its own window.
   `cache.m4`/`cache.r4` are previous-generation and **not supported**.
3. **Verify family availability in the standby region.** Checked against the AWS
   pricing API: `ca-west-1` offers `c8gn, m5, m6g, m8g, r5, r6g, r7g, r8g, t3,
   t4g` for ElastiCache — note **no `m7g`, no `r6gd`, no `c7gn`, no `m4`/`r4`**.
   `ca-central-1` has `m7g` and `r6gd` but no `c7gn`. **If a `ca-central-1`
   cluster is on `m7g` or `r6gd` today, it cannot form a global datastore with
   `ca-west-1`** — the node type must match across members and `m7g`/`r6gd` are
   absent in Calgary. Move those to `r7g`/`r8g`/`m8g` first. This is the single
   most likely concrete blocker in this note and it is easy to miss.
4. **Standardise on Valkey** (see engine section). Do this before, not during.
5. **Pre-create the standby-region KMS key**, subnet group, SGs, parameter group,
   and replicate the AUTH-token secret. Cross-ref [[aws-kms]],
   [[aws-secrets-manager]], [[aws-vpc-networking]].
6. **Check the account model.** Global Datastore is single-account. If prod
   `eu-west-1` and prod `eu-west-2` are different accounts, stop and redesign.
7. **Confirm the primary has replication enabled.** "Replication must be enabled
   if you plan to use an existing single-node cluster" — a single-node cache
   cluster is not a replication group and cannot be a global datastore primary.

### Step 2 — Create the global replication group from the existing primary

```bash
aws elasticache create-global-replication-group \
  --global-replication-group-id-suffix sessions-prod \
  --primary-replication-group-id sessions-prod-euw1 \
  --region eu-west-1
```

In Terraform this is `aws_elasticache_global_replication_group` with
`count = 1`. **This does not replace `aws_elasticache_replication_group.primary`.**
Confirm on the plan output that the primary shows no changes, or only in-place
ones. If the plan proposes replacing the live primary, stop — something else is
wrong (usually a node type or engine attribute drifted at the same time).

### Step 3 — Add the standby as a *new, empty* secondary

This is the step where the one-way door is:

> To bootstrap from existing data, use an existing cluster as primary to create a
> global datastore. **We don't support adding an existing cluster as secondary.
> The process of adding the cluster as secondary wipes data, which may result in
> data loss.**

So: if there is already a Redis in `eu-west-2` with anything in it, it cannot
become the secondary. Create a new one. If some team has already stood up a
standby-region cache "to get ahead", it must be torn down and recreated as part
of the global datastore. Find this out before you plan the change window, not
during.

Initial sync copies the full dataset. For a multi-GB cache this is minutes to
tens of minutes and puts load on the primary. Do it off-peak.

### Step 4 — Endpoint indirection

Put a private hosted zone CNAME in front of both endpoints per region
(`redis.sessions.internal` → the local cluster's configuration endpoint) *before*
you need to fail over, and move applications onto the CNAME. Doing this while
healthy is a no-op deploy. Doing it during a failover is a config change under
pressure. See [[aws-route53]].

### Step 5 — Duplicate auth

**RBAC users and user groups are not replicated.** Per AWS re:Post, "user
management in ElastiCache Global Datastore is independently managed on each
cluster associated with a Global Datastore. Having User groups configured in one
region will not copy the configuration to the secondary region." You must create
matching `aws_elasticache_user` / `aws_elasticache_user_group` resources in the
standby region with the **same user IDs and the same passwords**, or the
application will fail to authenticate the moment it points at the promoted
cluster.

Because the passwords must match, they must come from one source of truth:
a Secrets Manager secret replicated to the standby region, consumed by both
regions' `aws_elasticache_user` resources. See [[aws-secrets-manager]]. Note
that ElastiCache RBAC user passwords are set, not read back — Terraform will
hold them in state, so the state backend encryption story in
[[state-management]] applies.

### Step 6 — Test

`aws elasticache test-failover` in a non-prod global datastore, plus a scheduled
game day promoting the real standby and failing back. If the runbook has never
been executed, the 15-minute RTO is aspirational. See [[failover-orchestration]].

### What forces replacement

| Change | Effect | Mitigation |
|---|---|---|
| Adding `global_replication_group_id` to an **existing populated** standby RG | **Data wipe** (AWS-documented) | Never do it. Create the secondary empty. |
| Changing `node_type` to a GD-eligible family | In-place scaling operation on ElastiCache | Do it as a separate, earlier change |
| `engine` Redis OSS → Valkey | In-place engine change | Separate, earlier change |
| `replication_group_id` | `ForceNew` | Never change it. Pin the naming convention before you start. |
| `subnet_group_name` | `ForceNew` | Get the standby subnet group right first time |
| `at_rest_encryption_enabled` / `kms_key_id` | `ForceNew` | Turn encryption on *before* building the global datastore |
| `transit_encryption_enabled` | Historically `ForceNew`; newer engine versions support in-place enablement | Verify against the provider version you pin, in a scratch environment |
| Removing a member from a global datastore then re-adding | Re-sync from scratch | Treat as a maintenance operation |

---

## Failover procedure

Assume `eu-west-1` is impaired and the decision to fail over has been made.

1. **Check `ReplicationLag`** on the secondary if the primary-region control
   plane is still answering. It tells you how much data you are about to lose.
   If the primary region's API is unreachable, proceed anyway — under RPO 2h the
   worst case here is seconds.
2. **Promote:**
   ```bash
   aws elasticache failover-global-replication-group \
     --global-replication-group-id abcde-sessions-prod \
     --primary-region eu-west-2 \
     --primary-replication-group-id sessions-prod-euw2 \
     --region eu-west-2
   ```
   Run this from the **standby region's** endpoint. If you run it against the
   impaired region's endpoint and that region's control plane is down, you are
   stuck. This is a frequently-missed detail in runbooks and it is the whole
   reason the runbook has to be tested.
3. **Wait for `available`.** AWS: typically under a minute. Poll
   `describe-global-replication-groups`.
4. **Flip DNS.** Update the private hosted zone CNAME to the `eu-west-2`
   configuration endpoint. Automate this off the ElastiCache SNS event with a
   Lambda — the pattern AWS demonstrates in its multi-region session-store post.
   Keep the record TTL low (30–60 s) *permanently*; lowering it during an
   incident does not help because the old TTL is already cached.
5. **Force client reconnection** if the client library does not do it — usually a
   rolling restart of the standby-region deployment. See
   [[eks-workload-delivery]].
6. **Verify writes.** `SET helios:failover:probe <ts>` then `GET` it.
7. **Watch the origin database.** If any cluster started cold, the database is
   now taking the full read load. This is the step that actually breaks.

**What is automated:** promotion, DNS flip, client restart.
**What is human:** the decision, and only the decision. Keep it that way —
automatic cross-region promotion is how you get split brain
([[split-brain-and-fencing]]). AWS agrees: it does not offer cross-region
autofailover at all.

---

## Failback

After the promotion, the **old primary becomes a secondary** of the same global
datastore, and once `eu-west-1` recovers it starts replicating *from*
`eu-west-2`. That is convenient and it is also a trap: the moment `eu-west-1`
comes back and re-syncs, **it discards its own contents** and takes `eu-west-2`'s
data. Anything written to `eu-west-1` in the seconds before the region went dark
and not yet replicated is gone permanently at that point. Under RPO 2h this is
fine; document it so nobody goes looking for it later.

Failback is the same operation in reverse:

1. Confirm `eu-west-1` is genuinely healthy — not just the API, the data plane.
2. Confirm `ReplicationLag` on the `eu-west-1` secondary is near zero.
3. `failover-global-replication-group --primary-region eu-west-1 …`.
4. Flip DNS back.
5. Rolling restart.

**Do this during business hours on a planned day, never as a rush.** The system
is running fine in `eu-west-2`; there is no urgency. The single most common
DR-programme mistake is treating failback as an emergency and doing it at 4am
with a tired team. See [[lessons-and-antipatterns]].

**Split-brain risk during failback is real** and specific to caches: if the old
primary was not fully fenced and some subset of application instances in
`eu-west-1` are still pointed at the old (now-secondary) endpoint, they will get
read-only errors on writes — noisy but safe. The dangerous variant is if
somebody "fixes" that by promoting `eu-west-1` while `eu-west-2` is still taking
writes. A global datastore has exactly one primary, so ElastiCache will not let
you have two writable members — which is a genuine safety property worth calling
out. Self-hosted Redis gives you no such guarantee; see
[[redis-self-managed-vs-elasticache]].

---

## Gotchas

1. **You cannot add an existing populated cluster as a secondary.** AWS wipes it.
   Documented, irreversible, and the most likely way to lose data during the
   migration.
2. **Node type must match across all members** — so the standby costs the same
   per node as the primary. Only replica *count* can differ.
3. **`ca-west-1` lacks `m7g`, `r6gd` and `c7gn`** for ElastiCache (verified
   against the AWS pricing API). A `ca-central-1` cluster on one of those
   families cannot pair with Calgary until it is moved.
4. **Burstable `t`-class node types cannot join a global datastore at all.**
   Neither can previous-generation `m4`/`r4`.
5. **Auto minor version upgrade is silently disabled** on global datastore
   members and cannot be re-enabled. You now own patching.
6. **Global Datastore and durability are mutually exclusive.** Region resilience
   or zero-data-loss-within-region. Pick one, or use [[aws-memorydb]].
7. **No cross-account.** All members in one AWS account.
8. **No IPv6.**
9. **No Local Zones, no Outposts.**
10. **RBAC users and user groups do not replicate.** Duplicate them, with
    matching passwords, in every region. AUTH tokens likewise.
11. **KMS keys are regional.** A cluster in `eu-west-2` needs a key in
    `eu-west-2`. Multi-Region Keys help with key *material* consistency but each
    replica is still a separate regional key with its own policy — see
    [[kms-when-to-use-multi-region-keys]].
12. **Snapshot export is same-region only** — export to S3 in the backup's
    region, then S3 CRR. And it does not work for data-tiering (`r6gd`)
    clusters.
13. **`s3:ListAllMyBuckets` must be scoped to `*`** for snapshot export, or you
    get a misleading permissions error.
14. **The S3 bucket policy principal is regional**:
    `<region>.elasticache-snapshot.amazonaws.com`.
15. **Cluster-mode-enabled pub/sub does not propagate non-keyspace events
    cross-region.** Silent, and invisible in single-region tests.
16. **Parameter-group edits propagate to all members.** Modifying a *local*
    parameter group on one member applies it to every cluster in the global
    datastore. A "just this region" parameter tweak is not a thing.
17. **A customer-initiated reboot of a primary does not trigger failover, and
    the rebooted primary comes back empty — then the replicas clear themselves
    to match.** Total data loss, within one region, with no region involved.
    Worth a runbook line of its own.
18. **After any replica promotion, the other replicas full-resync**, are briefly
    unavailable, and load the new primary. Normal Redis behaviour, but it means
    a failover has a visible tail.
19. **`test-failover` is capped at 15 shards per rolling 24h and AWS may block
    it during large-scale operational events.** It is a rehearsal tool only.
20. **Client-side topology refresh is off by default in several popular
    libraries** (Lettuce being the canonical example). A textbook-perfect
    failover that the application never notices is the most demoralising
    possible game-day outcome.
21. **Global replication group IDs carry an AWS-generated 5-character prefix.**
    Never construct the ID by hand in a script or a runbook; read it from the
    API or from Terraform output.
22. **Provider replace-loop bug on Valkey replication groups**
    ([#40786](https://github.com/hashicorp/terraform-provider-aws/issues/40786))
    — verify against your pinned provider version before enabling auto-apply.

---

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Replicate the cache at all? | Global Datastore for every cluster | Cold standby for cache-as-cache, Global Datastore only for cache-as-datastore | **B.** Run the keyspace audit and decide per cluster. Replicating a pure cache is paying cross-region egress to protect reconstructible data. |
| Engine | Stay on Redis OSS 7.1 | Move to Valkey 8.x+ | **Valkey.** 20% cheaper on every node including the idle standby, RIs carry over, Redis OSS now has Extended Support surcharges, and durability is Valkey-only. |
| Region resilience vs durability | Durability (Multi-AZ txn log, zero/≤10s loss) | Global Datastore (cross-region) | **Global Datastore** where region loss is the stated risk — they are mutually exclusive. If you genuinely need both, the workload belongs in [[aws-memorydb]] or in Postgres. |
| Cheap DR for borderline clusters | Hourly snapshot → S3 CRR | Global Datastore | **Snapshot** where the data is *nice to have* — it clears RPO 2h at a fraction of the cost. It fails RTO 15m for the restore, so only use it where a cold start was nearly acceptable anyway. |
| Cluster mode | CMD (simpler) | CME (shards) | **CME**, per AWS Well-Architected, with ≥2 replicas/shard primary and ≥3 shards for sharded workloads. But do not bundle a CMD→CME migration into the multi-region work. |
| Standby replica count | Match primary (2/shard) | 1/shard | **1/shard** in the standby. It is the only per-region cost lever Global Datastore gives you, and it is ~33% off the standby bill. Scale up as part of the failover runbook (online, no interruption) once you are serving. |
| Standby app endpoint | Point at local read-only secondary | Point at a per-region private-hosted-zone CNAME | **CNAME.** Makes failover a DNS change instead of a deploy. |
| Locks in Redis | Replicate them | Move to DynamoDB conditional writes + fencing tokens | **Move them.** Async cross-region replication of a lock is not safe under any configuration. [[split-brain-and-fencing]]. |
| Queues in Redis | Replicate them | Move to [[aws-sqs]] | **Move them**, on a separate track. Replicate in the meantime. |
| Serverless caches needing DR | Keep Serverless, snapshot to S3 | Move to node-based | **Node-based** for anything DR-relevant. Serverless has no Global Datastore. |

---

## Cost

All figures pulled from the **public AWS pricing API**
(`https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonElastiCache/current/<region>/index.json`),
on-demand, Valkey engine, retrieved 2026-09-20. These are real, not estimated.

### Node rates, `cache.r7g.large`, per node-hour

| Region | Valkey | Redis OSS | Delta |
|---|---|---|---|
| `us-east-1` | $0.1752 | $0.2190 | −20% |
| `us-west-2` | $0.1752 | $0.2190 | −20% |
| `eu-west-1` | $0.1944 | $0.2430 | −20% |
| `eu-west-2` | $0.2048 | $0.2560 | −20% |
| `ca-central-1` | $0.1912 | $0.2390 | −20% |
| `ca-west-1` | $0.1905 | (Redis OSS SKU present) | −20% |

Notable: **`ca-west-1` is marginally *cheaper* than `ca-central-1`** for this node
type ($0.1905 vs $0.1912). Calgary is not a cost penalty. `eu-west-2` is ~5%
more than `eu-west-1`, which is the worst of the three pairs and still trivial.

`cache.m7g.large` Valkey for comparison: `us-east-1` $0.1264, `eu-west-1`
$0.1392, `eu-west-2` $0.1456, `ca-central-1` $0.1376. **`m7g` is not offered in
`ca-west-1`.**

### Worked standby cost — EU pair, one cluster

Assume 2 shards, 2 replicas/shard in the primary (6 nodes), 1 replica/shard in
the standby (4 nodes), `cache.r7g.large`, Valkey, 730 h/month.

| Line | Calculation | Monthly |
|---|---|---|
| Primary `eu-west-1` (6 nodes) | 6 × $0.1944 × 730 | **$851.47** |
| Standby `eu-west-2` (4 nodes) | 4 × $0.2048 × 730 | **$598.02** |
| Standby at matched replica count (6 nodes) | 6 × $0.2048 × 730 | $897.02 |
| Global Datastore replication egress | $0.02/GiB (pricing page, Example 5) × GiB written/month | see below |

**The replica-count lever is worth $299/month per cluster** in the EU pair. Across
a dozen clusters and three pairs that is real money.

**Replication egress** is the line nobody models. The pricing page states Global
Datastore replication traffic OUT at **$0.02 per GiB**. This is charged on the
*write volume*, not the dataset size — a 4 GiB cache with a 20 MiB/s write rate
replicates ~50 TiB/month, which at $0.02/GiB is **~$1,024/month**, more than the
standby nodes. For a write-heavy cache Global Datastore's *egress* can be the
dominant cost.

**Action:** before committing to Global Datastore on any cluster, pull the
`BytesWrittenToCache`-family CloudWatch metrics (or `NetworkBytesIn` on the
primary as a proxy) for 30 days and multiply. A high-churn cache is exactly the
kind you should not be replicating anyway.

### Levers, in order of value

1. **Don't replicate pure caches.** Saves the egress entirely and lets you shrink
   or right-size the standby independently (no node-type match constraint once
   there is no global datastore). Biggest lever by a distance.
2. **Valkey over Redis OSS.** −20% on every node, everywhere, forever.
3. **1 replica/shard in the standby.** −33% of standby node cost.
4. **Reserved nodes on the standby.** The standby runs 24/7 at fixed size —
   textbook reservation candidate. Redis OSS reservations transfer to Valkey in
   the same family/region at 20% more value.
5. **Right-size the standby's node type** — only available on the no-global-
   datastore path, since Global Datastore mandates matching node types.
6. **Graviton.** `r7g`/`r8g`/`m8g` are already the default in most regions and
   AWS's Well-Architected guidance specifically calls out better replication and
   sync performance on Graviton2+ nodes.

### Durability cost, for reference

`SyncDurability-NodeUsage:cache.r7g.large` in `eu-west-1` is **$0.035/hr** on top
of the $0.1944 node rate — about **+18%**. Cheap for what it is, but remember it
locks out Global Datastore.

---

## Open questions

1. **How many ElastiCache clusters are in the estate, and what is in each one?**
   Nothing in this note can be costed or decided without the keyspace audit.
   This is the single highest-value next action.
2. **Are any clusters on `t3`/`t4g`, `m4`/`r4`, `m7g` or `r6gd`?** The first two
   cannot join a global datastore at all; `m7g`/`r6gd` cannot pair with
   `ca-west-1`.
3. **Is prod `eu-west-1` the same AWS account as prod `eu-west-2`?** Global
   Datastore is single-account. If the estate is multi-account per region, this
   note's primary recommendation does not apply as written.
4. **Does anything use Redis for distributed locking?** If so it is already
   unsafe within a single region under failover, never mind across regions.
   Treat as a correctness bug, not a DR item. [[split-brain-and-fencing]]
5. **Does anything use Redis as a job queue / broker** (Sidekiq, BullMQ, Celery,
   RQ)? These are the clusters where a cold start silently drops customer-facing
   work. [[messaging-in-flight-data-loss]]
6. **Does any service use Redis pub/sub for cross-service messaging?** If so, and
   the cluster is CME, standby-region subscribers will not receive non-keyspace
   events.
7. **What is the actual write byte rate per cluster?** Determines whether Global
   Datastore egress dwarfs the node cost.
8. **Is ElastiCache Serverless available in `ca-west-1`?** The public pricing API
   index does not carry serverless usage types for *any* of the regions checked,
   so absence proves nothing. Verify on the console before relying on it.
9. **Is the "15 minute RTO" measured from incident start or from decision?**
   Flagged estate-wide in `CLAUDE.md`; it matters acutely here because
   ElastiCache's own contribution is under a minute and everything else in the
   budget is detection and decision.
10. **Does the business accept a mass re-login on a regional failover?** That one
    answer decides the session-store cluster, which is usually the most expensive
    one to replicate.

---

## Sources

- <https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Redis-Global-Datastores-Getting-Started.html>
  — the authoritative prerequisites-and-limitations list: supported regions
  (**including Canada West (Calgary)**), supported instance families, the
  "cannot add an existing cluster as secondary" warning, no cross-account, no
  IPv6, auto-minor-upgrade disabled, no durability, and the cluster-mode pub/sub
  asymmetry.
- <https://aws.amazon.com/about-aws/whats-new/2025/04/amazon-elasticache-global-datastore-15-additional-regions>
  — dated 2025-04-29; the verbatim list of 15 added regions, which is where
  `ca-west-1` support came from.
- <https://aws.amazon.com/elasticache/faqs/> — AWS's own RPO ("typically under
  one second") and RTO ("typically under a minute") statements for Global
  Datastore, and the Valkey 20%/33% pricing claims.
- <https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/AutoFailover.html>
  — within-region Multi-AZ behaviour, "typically just a few seconds" promotion,
  the rebooted-primary-comes-back-empty data-loss path, the replica full-resync
  tail, `test-failover` limits including "AWS may block this API" during
  large-scale operational events.
- <https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/durability.html> and
  <https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Durability.Limitations.html>
  — the Valkey 9.0 durability feature, sync vs async write semantics, the 10-second
  async loss window, and the verbatim "Durability is not supported with Global
  Datastores" limitation.
- <https://aws.amazon.com/about-aws/whats-new/2026/06/durability-amazon-elasticache/>
  — the durability GA announcement (June 2026).
- <https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/ReliabilityPillar.html>
  — Well-Architected lens: the "some use cases are entirely ephemeral and don't
  require any DR strategy" framing that underpins the cache-vs-datastore section,
  the 20-manual-backups-per-day figure, backup-from-replica guidance,
  client-timeout and topology-refresh requirements, and the CME/≥2-replica/≥3-shard
  recommendations.
- <https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/backups-exporting.html>
  — snapshot export mechanics: same-region S3 requirement, the regional
  `<region>.elasticache-snapshot.amazonaws.com` bucket-policy principal, the
  `s3:ListAllMyBuckets` scoping trap, the data-tiering exclusion, and both the
  node-based (`copy-snapshot`) and serverless (`export-serverless-cache-snapshot`)
  CLI forms.
- <https://docs.aws.amazon.com/cli/latest/reference/elasticache/failover-global-replication-group.html>
  — the exact promotion command and its three required parameters.
- <https://aws.amazon.com/blogs/database/build-a-multi-region-session-store-with-amazon-elasticache-for-valkey-global-datastore/>
  — AWS's reference architecture for the session-store case: sessions as "soft
  state", secondary read-only, cross-region write timeouts of 200–500 ms, and
  the SNS→Lambda→Route 53 failover automation pattern.
- <https://repost.aws/questions/QUabGMT-CKRPWwxt6GRIZd8Q/for-aws-elastic-redis-global-cache-do-we-need-to-create-again-users-and-user-group-even-in-secondary-region-too-again>
  — AWS re:Post confirming RBAC users and user groups are **not** replicated
  across a global datastore and must be created per region.
- <https://repost.aws/knowledge-center/elasticache-replication-lag-datastore>
  — AWS's own guidance on monitoring cross-region replication lag in a global
  datastore; the metric to alarm on.
- <https://aws.amazon.com/elasticache/pricing/> — the Valkey discount
  percentages, the Redis-OSS-reservation-transfers-to-Valkey statement, the
  **$0.02/GiB Global Datastore replication traffic OUT** figure, and the explicit
  "Global Datastore is not currently available with ElastiCache Serverless".
- `https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonElastiCache/current/<region>/index.json`
  — the AWS public pricing API, queried directly for `eu-west-1`, `eu-west-2`,
  `ca-central-1`, `ca-west-1`, `us-east-1`, `us-west-2`. Source of every node
  price in the Cost section, the per-region instance-family availability
  (including **`m7g`/`r6gd`/`c7gn` absent in `ca-west-1`**), the
  `SyncDurability-NodeUsage` rates, and the **Redis OSS Extended Support**
  SKUs.
- <https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/elasticache_global_replication_group>
  — argument reference, the "provider ignores engine/engine_version/
  parameter_group_name on member groups" behaviour, `parameter_group_name`
  required for major-version upgrades, and the 60m/60m/20m default timeouts.
- <https://github.com/hashicorp/terraform-provider-aws/issues/40786>
  — open/known provider issue: Valkey `aws_elasticache_replication_group` stuck
  in a replace loop. Verify against your pinned version.
- <https://redis.io/blog/agplv3/> — Redis Ltd's own announcement of the AGPLv3
  addition in Redis 8 (2025-05-01).
- <https://antirez.com/news/151> — antirez's "Redis is open source again" post,
  the primary-source account of the licence reversal.
- <https://aws.amazon.com/blogs/database/amazon-elasticache-and-amazon-memorydb-announce-support-for-valkey/>
  — AWS's announcement of Valkey support across ElastiCache and MemoryDB.

### Related notes

[[redis-self-managed-vs-elasticache]] · [[aws-memorydb]] · [[aws-kms]] ·
[[kms-when-to-use-multi-region-keys]] · [[aws-secrets-manager]] ·
[[aws-route53]] · [[aws-vpc-networking]] · [[aws-iam]] · [[aws-sqs]] ·
[[aws-eventbridge]] · [[aws-dynamodb]] · [[aws-rds-postgres]] ·
[[aws-aurora-global-database]] · [[failover-orchestration]] ·
[[split-brain-and-fencing]] · [[messaging-in-flight-data-loss]] ·
[[lessons-and-antipatterns]] · [[region-pair-selection]] ·
[[provider-aliases-vs-separate-stacks]] · [[module-patterns]] ·
[[state-management]] · [[eks-workload-delivery]]
