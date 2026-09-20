---
title: Amazon OpenSearch Service — Multi-Region
service: opensearch
tags: [service, multi-region, opensearch, search, ccr]
status: researched
replication: native (cross-cluster replication) | manual (snapshot to S3) | none (rebuild from source)
rpo_achievable: "sub-minute with CCR; ~1h with hourly snapshot+restore; 0 with rebuild-from-source (the index is derived data)"
rto_achievable: "< 5 min with a warm follower domain + DNS flip; 30-90+ min if the standby has to restore from snapshot; minutes-to-hours if rebuilding"
meets_targets: conditional — yes for the EU and US pairs, **no via CCR for the CA pair** (see the opt-in Region blocker below), and never for OpenSearch Serverless
updated: 2026-09-20
---

# Amazon OpenSearch Service — Multi-Region

> Read [[messaging-in-flight-data-loss]] first if you have not. The argument it
> makes about queues — that a queue message is usually a *pointer* to a fact
> rather than the fact itself, and therefore should not consume RPO budget —
> applies with even more force to a search index. **An OpenSearch index is,
> for most workloads, derived data.** This note takes that seriously and gives
> "rebuild it" its own section rather than treating replication as the default.

## TL;DR

- **Cross-Cluster Replication (CCR) works, is genuinely near-real-time (AWS
  says "typical delivery times are less than a minute"), and comfortably meets
  RPO 2h — but it is a one-way door.** Followers are read-only; promoting one
  means `POST _plugins/_replication/<index>/_stop`, which permanently unfollows
  the index. You cannot restart replication afterwards, and failback requires
  **deleting the index on the old primary and re-bootstrapping from scratch**.
  Plan the failback before you build the failover.
- **HEADLINE, CA PAIR: CCR is not available between `ca-central-1` and
  `ca-west-1`.** AWS's own limitation list states cross-cluster replication "is
  not supported between default and opt-in Regions. Both domains must be either
  in default Regions or in opt-in Regions." `ca-central-1` is a **default**
  Region; `ca-west-1` (Calgary) is an **opt-in** Region. This is a hard
  product-level blocker, not a quota. The CA pair must use snapshot/restore or
  rebuild-from-source. Feed this into [[region-pair-selection]].
- **HEADLINE, SERVERLESS: OpenSearch Serverless has no cross-region story at
  all.** The service limitations page says flatly: "Cross-Region search and
  replication aren't supported." Manual snapshots are also not supported — only
  automated ones. And Serverless **is not available in `ca-west-1` at all**.
  Since Bedrock Knowledge Bases commonly use a Serverless vector collection as
  its vector store, **any Knowledge Base built on AOSS is single-region by
  construction** and its DR story is "re-ingest from the source S3 bucket".
  See [[aws-bedrock]].
- **Everything that is not index data is the part everyone forgets.** Dashboards
  saved objects live *inside* the cluster in the `.kibana*` indexes; the
  fine-grained-access-control user database lives inside the cluster in
  `.opendistro_security` — AWS states it "is stored in an OpenSearch index, so
  you can't share it with other clusters"; ISM policies live in
  `.opendistro-ism-config`. **CCR replicates user indexes, mappings and
  metadata. It does not replicate any of those.** Nor can you restore them from
  a snapshot cleanly — AWS's own restore guidance tells you to *exclude*
  `.kibana*` and `.opendistro*`. All of it has to be mirrored by Terraform, by
  a CI job, or by hand.
- **The thing that will bite:** the domain endpoint is region-specific and
  baked into every client. `search-<domain>-<hash>.eu-west-1.es.amazonaws.com`
  is not a name you can CNAME your way out of at 3am unless you already put a
  **custom endpoint** in front of it. Do that on day one — it is the single
  cheapest RTO improvement in this note and it costs one ACM cert and one
  Route 53 record per region. See [[aws-route53]] and [[aws-acm]].

---

## Does this service cross regions at all?

Partially, and the answer differs sharply by flavour.

| Flavour | Regional or global? | Native cross-region mechanism | Verdict |
|---|---|---|---|
| **OpenSearch Service (managed domain)** | Regional | **Cross-Cluster Replication** (leader → read-only follower) and **Cross-Cluster Search** | Real, supported, but constrained — see the opt-in Region blocker |
| **OpenSearch Serverless (collections)** | Regional | **None.** "Cross-Region search and replication aren't supported." | No DR story. Rebuild or re-ingest. |
| **OpenSearch Ingestion (OSI pipelines)** | Regional | Pipelines can write to a remote-region sink; AWS publishes a cross-region-resilience pattern using OSI + S3 CRR | Works, but **OSI is absent from `ca-west-1`** |
| **Snapshots** | Repository is an S3 bucket | Manual snapshots to *your* S3 bucket; the bucket can be replicated cross-region | The universal fallback. Works everywhere. |

Three region-parity facts, each verified against the AWS General Reference
endpoints page:

1. **OpenSearch Service managed domains exist in `ca-west-1`** —
   `es.ca-west-1.amazonaws.com` is a published endpoint. So the standby domain
   itself is buildable in Calgary.
2. **OpenSearch Serverless does not exist in `ca-west-1`.** The AOSS endpoint
   table lists `ca-central-1` but not `ca-west-1`. If any part of the Canadian
   deployment uses a Serverless collection — a vector store, a log-analytics
   collection — **there is nowhere in Canada to fail it over to**, and moving it
   out of Canada has data-residency consequences ([[data-residency]]).
3. **OpenSearch Ingestion does not exist in `ca-west-1`** either. The OSI
   endpoint table lists `ca-central-1`, `eu-west-1`, `eu-west-2`, `us-east-1`,
   `us-west-2` — but not Calgary. So the AWS-blessed OSI-based cross-region
   resilience pattern is available to the EU and US pairs and **not** to CA.

All three pairs otherwise have managed-domain parity. The EU pair
(`eu-west-1` → `eu-west-2`) and the US pair (`us-east-1` → `us-west-2`) are
both default-Region-to-default-Region, so CCR is available to both.

### The opt-in Region blocker, stated precisely

This is worth spelling out because it is easy to miss and it changes the CA
plan entirely.

From [Cross-cluster replication for Amazon OpenSearch
Service](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/replication.html),
under Limitations:

> "Cross-cluster replication is not supported between default and opt-in
> Regions. Both domains must be either in default Regions or in opt-in Regions."

From [Enable or disable AWS Regions in your
account](https://docs.aws.amazon.com/general/latest/gr/rande-manage.html):

- `ca-central-1` (Canada Central) — **Default Region**.
- `ca-west-1` (Canada West, Calgary) — **Opt-in Region**, GA.

Therefore `ca-central-1` → `ca-west-1` CCR is **not possible**. The same
limitation applies to cross-cluster *search* implicitly (the connection
machinery is shared — see below), although the cross-cluster-search limitation
list does not repeat the sentence, so treat CCS across that boundary as
unverified rather than known-broken.

Two other pairings are worth noting for [[region-pair-selection]]:
- `eu-west-1` → `eu-west-2`: both default. CCR available.
- `us-east-1` → `us-west-2`: both default. CCR available.
- `ca-central-1` → any other default Region (e.g. `us-east-1`): CCR available,
  but leaves Canada. Not acceptable if residency binds.
- `ca-west-1` → another opt-in Region: technically allowed by the rule, but
  there is no other opt-in Region in Canada.

**Conclusion for CA: the Canadian pair's OpenSearch DR is snapshot-based or
rebuild-based. Full stop.** That is not a disaster — see the RPO analysis, an
hourly snapshot meets a 2h RPO with room to spare — but it means the CA
runbook is structurally different from the EU and US runbooks, and a
cookiecutter template that assumes CCR everywhere will produce a broken CA
environment. Make the replication mechanism a module variable, not a constant.

---

## Replication / mirroring options

### Option A — Cross-Cluster Replication (CCR)

The native mechanism. A follower index on the standby domain pulls from a
leader index on the primary domain. Active-passive by construction: **the
follower is read-only and will not accept writes.**

**Prerequisites** (all mandatory, all verified from the AWS docs):

| Requirement | Detail | Consequence for a live estate |
|---|---|---|
| Engine version | Elasticsearch 7.10, or OpenSearch 1.1 or later | Anything older must be upgraded first, and an upgrade is a blue/green deployment |
| **Fine-grained access control enabled** | Required on both domains | If FGAC is off today, enabling it is a **blue/green deployment** and **cannot be undone** — "After you enable fine-grained access control, you can't disable it" |
| **Node-to-node encryption enabled** | Required on both domains | Enabling is also blue/green; in Terraform, *disabling* it later is `ForceNew` |
| `index.soft_deletes.enabled = true` on leader indexes | Default since ES 7.0 / OpenSearch 1.0 | Indexes created on ES 6.x and upgraded retain `soft_deletes=false` and **must be reindexed** before they can be replicated. Check this early. |
| Instance type | **Not M3, not T2, not T3** | A burstable dev/test domain cannot use CCR at all |
| Storage tier | Hot only — **no UltraWarm, no cold** | If the log-analytics domain uses UltraWarm, the warm tier is not replicated |
| Version ordering | At connection setup, leader must be on the **same or higher** version than follower | Constrains the upgrade order; see failback |

**Other stated limitations worth knowing:**

- Max 20 connected domains per domain (inbound + outbound combined).
- You cannot chain: follower → follower replication is not allowed.
- **Deleting an index on the leader does not delete it on the follower.** The
  standby will silently accumulate orphaned indexes. Budget disk for it and
  add a reconciliation job, or you will discover it as a disk-full alarm on
  the standby during an incident.
- **You cannot use CloudFormation to connect domains.** (Terraform *can* — see
  `aws_opensearch_outbound_connection` below — but note that AWS's own IaC
  cannot, which is a hint about how young this control plane is.)
- A connection previously created for cross-cluster *search* is marked
  `SEARCH_ONLY` and **cannot be reused for replication**. You must delete and
  recreate it. If someone set up CCS between these domains last year, that is
  a deletion-and-recreation step in your migration plan.

**What CCR actually replicates:** "user indexes, mappings, and metadata". Read
that narrowly. It does **not** carry:

- Dashboards saved objects (`.kibana*` — system indexes)
- The FGAC security configuration (`.opendistro_security`)
- ISM policies (`.opendistro-ism-config`)
- Index *templates* (a cluster-level object, not an index)
- Alerting monitors, notification channels, anomaly detectors
- Snapshot repository registrations
- Custom packages / dictionaries (`aws_opensearch_package` associations)

See "The stuff CCR doesn't carry" below — it is the longest gotcha in the note.

**Auto-follow** is the piece that makes CCR operationally viable. A replication
rule on the follower matches a pattern (`logs-*`, or `*` for everything) and
automatically creates follower indexes for matching leader indexes, including
ones created later. Without auto-follow, every new daily index is a manual
`_start` call and your standby silently falls behind the day someone adds an
index. **Use auto-follow. Alarm on `ReplicationNumSyncingIndices` diverging
from the leader's index count.**

**Replication lag.** AWS's Big Data blog states "typical delivery times are
less than a minute" and repeats "small lag (less than a minute)" when
discussing failover. Monitor it with the `LeaderCheckPoint` and
`FollowerCheckPoint` CloudWatch metrics — if they are equal, the index is in
sync; their divergence *is* your RPO exposure, in operations rather than
seconds. `ReplicationRate` gives ops/sec. **No independent public benchmark of
CCR lag under sustained heavy indexing on Amazon OpenSearch Service was found**
— the numbers above are AWS's own and should be treated as marketing-adjacent
until you measure your own in a game day ([[dr-testing-and-gamedays]]).

**The 12-hour rule.** Replication can be paused, but "you can't resume
replication after it's been paused for more than 12 hours. You must stop
replication, delete the follower index, and restart replication of the
leader." This matters more than it looks: a network partition, an IAM policy
change that breaks the connection, or a leader domain stuck in a long
blue/green can silently exceed 12 hours over a weekend. **You then have to
re-bootstrap the entire index**, which for a large index is hours of transfer
and a period during which the standby is *not* a valid failover target.
Alarm on replication status leaving `SYNCING` with a threshold well under 12
hours — one hour is a sane page threshold.

**Promotion.** The procedure is two commands and a DNS change, and it is fast:

```
POST _plugins/_replication/<follower-index>/_stop
{}
```

"When you stop replication completely, the follower index unfollows the leader
and becomes a standard index. **You can't restart replication after you stop
it.**" The index becomes writable immediately. For a handful of indexes this is
seconds; for a few hundred it is a scripted loop over the `_cat/indices`
output and still well inside a minute or two. **Promotion is not the slow part
of the RTO. DNS and application config are.**

### Option B — Cross-Cluster Search (CCS)

The alternative that people reach for and that is almost always wrong for DR.

CCS lets a *source* domain query a *destination* domain and merge results.
Unidirectional. No data is copied. Requirements mirror CCR's (FGAC,
node-to-node encryption, ES 7.10+/OpenSearch for cross-region, no T2/T3/M3,
max 20 connections each way), plus a networking one that CCR's docs do not
state as plainly:

> "If either domain is in a VPC, the domains must be connected via VPC Peering
> or Transit Gateway, and the security groups must allow traffic between them."

That is a real dependency on [[aws-vpc-networking]] — inter-region VPC peering
or a Transit Gateway peering attachment, with routes and SG rules, before any
of this works. Note also that **OpenSearch's managed VPC endpoints (PrivateLink)
are same-region only**, so `connection_mode = "VPC_ENDPOINT"` does not solve
the cross-region case; cross-region VPC domains need peering/TGW.

**Why CCS is not a DR mechanism:** it does not copy data. If the primary region
is gone, the destination domain is gone with it, and a CCS query against it
returns nothing (or fails the whole request, unless `skip_unavailable` is
`ENABLED`, in which case it silently returns partial results — which is
arguably worse, because your search results quietly get less complete and
nobody notices).

**Where CCS genuinely helps:** during a *planned* migration or a *partial*
failure, a standby-region application can query the primary's data without
having a copy of it. It is a latency-and-cost trade, not a durability one. And
`skip_unavailable = ENABLED` plus an alarm on `_clusters.skipped > 0` is a
decent degraded-mode posture for a non-critical search surface. Also note the
Dashboards implication: index patterns must be written as
`connection-alias:index`, so a dashboard built for CCS is not the same
dashboard as one built for local indexes.

**Recommendation: do not use CCS for DR.** Use it, if at all, for the
read-during-migration window in the migration path below.

### Option C — Snapshot to S3, restore into the standby

The cheap path, and **the only path available to the CA pair**.

Two kinds of snapshot, and the distinction is load-bearing:

| | Automated snapshots | Manual snapshots |
|---|---|---|
| Frequency | **Hourly**, 336 retained (14 days) | Whenever you trigger them (ISM `snapshot` action, Snapshot Management, or an external scheduler) |
| Repository | `cs-automated`, or `cs-automated-enc` if the domain is encrypted at rest | Your own S3 bucket, registered as a repository |
| Cost | Free (AWS-managed bucket) | Standard S3 charges |
| **Restorable to a different domain?** | **No** — "Restore the snapshot to a different OpenSearch Service domain (only possible with manual snapshots)" | **Yes** |

**This is the trap.** The domain already takes hourly snapshots for free, which
sounds like it satisfies a 2-hour RPO out of the box. It does not, because
**automated snapshots cannot be restored into another domain** — they exist
only to repair *this* domain. For cross-region DR you must run **manual**
snapshots into a bucket you own.

The cross-region mechanics:

1. Register an S3 repository on the primary domain pointing at a bucket in the
   primary region, with an IAM role (`TheSnapshotRole`) trusted by
   `es.amazonaws.com` and granting `s3:ListBucket`, `s3:GetObject`,
   `s3:PutObject`, `s3:DeleteObject`. The principal registering the repository
   needs `iam:PassRole` on that role plus `es:ESHttpPut`. With FGAC on, the
   `manage_snapshots` role must additionally be mapped to that IAM role.
2. Replicate the bucket to the standby region with **S3 Cross-Region
   Replication** ([[aws-s3]]). S3 Replication Time Control gives the 15-minute
   / 99.99% SLA that the OSI blog quotes; plain CRR gives no time guarantee.
3. Register the *replica* bucket as a repository on the standby domain, with a
   standby-region `TheSnapshotRole`.
4. Restore on demand.

Three gotchas in that chain:

- **Do not apply a Glacier lifecycle rule to the snapshot bucket.** AWS states
  "Manual snapshots don't support the Amazon Glacier storage class." A
  well-meaning cost-optimisation lifecycle policy will silently destroy your
  DR capability.
- **KMS.** If the bucket is encrypted with a customer-managed key, the standby
  region needs a key it can use. A single-region KMS key in `eu-west-1` is
  useless in `eu-west-2`. This is exactly the case [[kms-when-to-use-multi-region-keys]]
  and [[aws-kms]] exist to answer. S3 CRR can re-encrypt with a destination-region
  key, which is usually the better answer than a multi-region key.
- **Restore is not instant and it is not free of index conflicts.** "You can't
  restore a snapshot of your indexes to an OpenSearch cluster that already
  contains indexes with the same names." So the standby either holds no copy
  of those indexes (and restore takes as long as it takes) or you restore with
  `rename_pattern`/`rename_replacement` and re-alias — which is the trick that
  makes a *pre-warmed* standby possible. See "Warm standby shape".

**Restore excludes system indexes, by AWS's own instruction:**

```
POST _snapshot/<repo>/<snapshot>/_restore
{ "indices": "-.kibana*,-.opendistro*,-.opensearch-observability*,-.plugins-ml-config*" }
```

AWS says "Due to special permissions on the OpenSearch Dashboards and
fine-grained access control indexes, attempts to restore all indexes might
fail." **So snapshot/restore does not move your dashboards or your users
either.** Same gap as CCR. Section below.

### Option D — Rebuild the index from the source of truth

The option that most notes skip and that is, for a large fraction of
workloads, the correct answer. It gets its own section below because it
deserves an argument, not a bullet.

### Option E — OpenSearch Ingestion (OSI) dual-sink / S3-first

AWS published a pattern ([Achieve cross-Region resilience with Amazon
OpenSearch Ingestion](https://aws.amazon.com/blogs/big-data/achieve-cross-region-resilience-with-amazon-opensearch-ingestion/))
in which OSI pipelines in each region write to S3 and to the local collection,
with S3 CRR carrying data across, giving an **active-active** posture that
avoids CCR's manual leader/follower re-establishment. Notably it "applies to
both OpenSearch Service managed clusters and Amazon OpenSearch Serverless
collections" — which is currently the *only* published route to a
cross-region Serverless posture, and it is a pipeline pattern, not a service
feature.

Two reasons this is not the recommendation here:

1. **OSI is not available in `ca-west-1`**, so it cannot be the estate-wide
   standard.
2. It is an active-active ingest architecture. The brief's target is
   active/passive warm standby. Adopting OSI for DR means adopting a second
   ingest path in front of a service that currently has one, which is a
   larger change than the problem warrants.

It is, however, the right answer if the ingest path is *already* streaming
(Kafka/Kinesis) — see [[aws-msk-kafka]] and [[aws-kinesis]], where the same
"replay from the stream into the standby" idea shows up as the dominant
recommendation. If the log pipeline already exists in both regions, pointing
its standby copy at a standby domain is close to free and makes the search
tier's DR a property of the ingest tier rather than a separate mechanism.

---

## The honest question: should this be replicated at all?

**For a search index the answer is very often no, and the note would be
dishonest not to lead with that.**

An OpenSearch index is almost always a *projection* of state that lives
somewhere else — a Postgres table ([[aws-rds-postgres]]), a DynamoDB table
([[aws-dynamodb]]), an S3 data lake ([[aws-s3]]), a log stream. The index
exists because querying the source of truth with a `LIKE '%foo%'` or a vector
similarity search would be slow or impossible. **The index contains no
information that the source does not.**

If that is true of an index, then:

- Its RPO is **zero by definition** — nothing is lost, because nothing unique
  was stored. This is the same argument [[messaging-in-flight-data-loss]] makes
  about queue messages, and it is stronger here, because a search index almost
  never sits at an acknowledgement boundary.
- Its RTO is **however long a reindex takes**, which is entirely a function of
  source-data volume and indexing throughput, and which you can measure today.
- Its cost in the standby region is **zero while idle** if you run no domain
  at all, or the cost of a small domain if you want somewhere to reindex into.

### When rebuild-from-source is right

| Signal | Why it points to rebuild |
|---|---|
| The index is a projection of a relational/NoSQL table that is itself replicated | The data will be in the standby region anyway; only the projection needs recreating |
| Full reindex takes less than ~10 minutes at standby-region capacity | Fits inside RTO with room for the DNS flip |
| Search is a *degradable* feature (site search, filters, autocomplete) | The application can serve a degraded experience from the database while the index rebuilds |
| The index is log/observability data | See below — this is the special case |
| The domain is Serverless | You have no choice; there is no replication |
| The index is a Bedrock Knowledge Base vector store | You have no choice; re-ingest from the S3 source |

### When rebuild-from-source is wrong

| Signal | Why replication wins |
|---|---|
| **The index is the source of truth** | Rare but real: free-text-only content, ingested documents never stored elsewhere, security/audit logs written only to OpenSearch |
| Reindex takes hours | Blows the 15-minute RTO outright |
| The source of truth is a third party you cannot re-query | Same failure mode as the "acknowledged & unique" queue category in [[messaging-in-flight-data-loss]] |
| Expensive enrichment at index time | Embeddings, ML inference, external API lookups — the reindex cost is not just I/O, it is money and third-party rate limits |
| The index feeds a legal/regulatory retention obligation | See [[data-residency]] |

### The log/observability special case

A log-analytics domain deserves a separate decision from a product-search
domain, and conflating them is the most common mistake here.

Logs from the primary region, sitting in a search index in the primary region,
when the primary region is on fire, are exactly the data you most want and are
least able to reach. **Replicating them to the standby is not a DR measure for
the logging platform — it is a DR measure for your ability to run the
incident.** [[messaging-in-flight-data-loss]] makes this point in one line:
"you lose the evidence of what you lost." That is a strong argument for
replicating (or dual-shipping) the observability domain even when you would
not replicate a product-search domain.

But the cheaper version of that argument is: **don't replicate the domain,
ship the logs to both regions from the agent.** If the collection pipeline
already fans out ([[observability-multi-region]]), the standby domain is
populated continuously with no CCR, no opt-in-Region restriction, and it works
identically for the CA pair. That is a better design than CCR for logs, and it
sidesteps the whole of this note.

### The recommendation

**Classify each domain, exactly as [[messaging-in-flight-data-loss]] classifies
each queue.** Three buckets:

| Class | Mechanism | Standby cost |
|---|---|---|
| **Derived & fast to rebuild** | No replication. A small standby domain, or none. A tested reindex job. | Near zero |
| **Derived but slow to rebuild** | CCR (EU/US) or hourly manual snapshot + restore (CA) | A warm domain |
| **Source of truth** | CCR where possible; otherwise snapshot at a cadence that meets RPO, and open a ticket to stop OpenSearch being a system of record | A warm domain |

The classification exercise is worth more than the mechanism. In a typical
estate most domains land in row 1, and the budget saved there is what pays for
doing row 3 properly.

---

## RPO / RTO analysis

### Against RPO 2h

Every mechanism clears it, which is the comfortable part.

| Mechanism | Realistic RPO | Inside 2h? |
|---|---|---|
| CCR, healthy | < 1 minute (AWS's figure; verify yours) | Yes, by two orders of magnitude |
| CCR, degraded but syncing | Whatever `LeaderCheckPoint − FollowerCheckPoint` says | Yes, until it isn't — alarm on it |
| CCR, paused > 12h | **Infinite — the standby is invalid and must be re-bootstrapped** | **No.** This is the CCR failure mode that blows RPO. |
| Manual snapshot hourly + S3 CRR | ~1h + replication time (minutes with RTC) | Yes, with ~50% headroom |
| Manual snapshot every 4h | ~4h | **No** |
| Rebuild from source | 0 (nothing unique is lost) | Yes, trivially |

**The RPO risk with CCR is not lag. It is silent stoppage.** A `SYNCING` status
that has quietly become `PAUSED` or `FAILED` is an RPO of "however long since
anyone looked". Two alarms are mandatory and cheap:

- `ReplicationNumSyncingIndices` below the expected index count.
- `LeaderCheckPoint` − `FollowerCheckPoint` above a threshold, sustained.

With snapshot-based replication the equivalent alarm is on snapshot age in the
standby bucket — an S3 event or a scheduled Lambda that checks the newest
object's `LastModified` and pages if it is older than 90 minutes. That alarm is
the entire CA-pair RPO control and it should exist before the standby domain
does.

### Against RTO 15m

Here the mechanisms diverge hard.

| Step | CCR path | Snapshot path | Rebuild path |
|---|---|---|---|
| Decide to fail over | human | human | human |
| Promote / make writable | `_stop` per index — **seconds to ~2 min** | Restore snapshot — **minutes to hours**, proportional to index size | Start reindex — **minutes to hours** |
| Repoint clients | DNS / config — **1–5 min**, see below | same | same |
| Dashboards usable | Only if saved objects were pre-loaded | Only if pre-loaded | Only if pre-loaded |
| **Total** | **~5 min. Meets RTO.** | **Depends entirely on index size. Often fails RTO.** | **Depends. Often fails RTO for large indexes; trivially passes for small ones.** |

**Where the time actually goes, in order of how much it hurts:**

1. **Client reconfiguration.** The auto-generated endpoint
   `search-<domain>-<hash>.<region>.es.amazonaws.com` contains the region.
   Applications that hold it in config need a deploy, a config-map rollout, or
   a restart to change it. On EKS that is a rollout of every consuming
   deployment ([[aws-eks]]) and it can easily be ten minutes.
   **Mitigation, and do this first: enable a custom endpoint.** OpenSearch
   Service supports a custom domain endpoint backed by an ACM certificate in
   the same account, fronted by a CNAME (or, since April 2024, a Route 53
   **alias** record, which requires the domain to use the dual-stack IP address
   type). Give each domain `search.<env>.internal.example.com`, point it at the
   primary, and failover becomes a Route 53 record change — seconds, and
   automatable in [[failover-orchestration]]. Note the certificate must be in
   the **same region as the domain**, so each region needs its own ACM cert
   for the same name ([[aws-acm]]).
2. **Restore time**, if you are on the snapshot path. Unbounded and
   proportional to data. Mitigated by keeping the standby *continuously
   restored* — see warm standby shape.
3. **Promotion**, if you are on CCR. Small. Script it anyway; a `for` loop over
   `_cat/indices?h=index` issuing `_stop` is ten lines and removes a whole
   class of 3am error.
4. **Dashboards.** Not on the critical path for serving traffic, but very much
   on the critical path for *running the incident*. If the on-call's dashboards
   only exist in the dead region, the RTO for "we can see what's happening" is
   infinite. Pre-load saved objects.

**Verdict:** CCR meets RTO 15m for the EU and US pairs. **The CA pair cannot
meet RTO 15m via restore-at-failover** for any index of meaningful size, and
must use the continuously-restored standby pattern below, or accept rebuild.

---

## Warm standby shape

What exists in the standby region while the primary is healthy.

### EU and US pairs (CCR available)

| Component | State while primary healthy | Costs money? |
|---|---|---|
| Standby domain | **Running**, sized at or near primary's data-node count | **Yes — this is the bulk of the cost** |
| Dedicated master nodes | Running (3, if the primary has them) | Yes |
| Follower indexes | `SYNCING`, read-only | Storage cost |
| Cross-cluster connection | `Active`, outbound from follower | No |
| Auto-follow rule | Active, pattern `*` or per-workload | No |
| Dashboards saved objects | **Pre-loaded** via CI (see below) | No |
| FGAC roles / role mappings | **Pre-created** via CI or Terraform | No |
| ISM policies + index templates | **Pre-created** via CI | No |
| Custom endpoint + ACM cert | Present, DNS not yet pointed here | Cert free; Route 53 pennies |
| Snapshot repository registration | Registered against the replica bucket | No |
| Standby-region clients | Deployed, scaled per [[messaging-in-flight-data-loss]]'s "running but not subscribed" pattern | EKS cost, counted elsewhere |

**Sizing the standby is the real cost lever.** Two defensible positions:

- **Same size as primary.** Simplest, gives full performance immediately,
  and is the only option if query load at failover is equal to today's. Costs
  ~100% of the primary's domain bill.
- **Reduced data-node count, scale up at failover.** CCR needs enough nodes to
  hold the data, so you cannot go below the storage requirement — but you can
  run fewer, larger-storage-per-node instances, or the same count at a cheaper
  instance family. **Do not plan to change instance type at failover**: that is
  a blue/green deployment, which "usually" doubles the node count temporarily
  and copies all shards. It is not a 15-minute operation and it will happen at
  the worst possible moment. **Scaling the data node *count* up does not
  normally require blue/green** ("Changing the data node or UltraWarm node
  count" is on the no-blue/green list), so count is the safe lever and
  instance type is not.

**Recommendation: match the primary's instance type exactly, and allow a lower
node count if storage permits. Never plan an instance-type change into the
failover runbook.**

### CA pair (no CCR)

The warm shape has to be built differently, and the trick is to keep the
standby *continuously restored* rather than restoring at failover:

1. Manual snapshot on the primary every hour (ISM `snapshot` action or
   Snapshot Management), into a primary-region bucket.
2. S3 CRR (with Replication Time Control) to a `ca-west-1` bucket.
3. A scheduled job on the standby domain that restores the newest snapshot
   into **renamed** indexes (`rename_pattern` / `rename_replacement`, e.g.
   `orders` → `restore-orders`) — because you cannot restore over an existing
   index name.
4. At failover: delete the live-named aliases/indexes if any, and flip an
   **alias** from `restore-orders` to `orders`. Alias switching is atomic and
   instant.

That converts "restore at failover" (unbounded) into "restore continuously,
alias-flip at failover" (seconds). It costs a standby domain plus the restore
I/O, and it is more moving parts than CCR — but it is the only way the CA pair
hits 15 minutes.

Heed AWS's warning on aliases: if you delete an index that has an alias, the
alias goes with it, and an errant write to the now-missing alias will create a
fresh index of that name and permanently block you from re-creating the alias.
**Stop writes before you touch aliases.** That is a fencing problem —
[[split-brain-and-fencing]].

---

## Terraform implementation

### The shape for a cookiecutter monorepo

The estate is templated per environment; the multi-region change should be a
*variable*, not a fork. The design that fits:

- One `opensearch` module, instantiated twice by the environment stack — once
  per region, via provider aliases ([[provider-aliases-vs-separate-stacks]]).
- A `role` variable (`"primary"` / `"standby"`) that drives the small number of
  behavioural differences.
- A `replication_mode` variable (`"ccr"` / `"snapshot"` / `"none"`) so the CA
  environment can differ without a separate module. **This is the concrete
  consequence of the opt-in-Region blocker** and the reason not to hardcode CCR.

```hcl
# environments/<env>/opensearch.tf

module "opensearch_primary" {
  source = "../../modules/opensearch"
  providers = { aws = aws.primary }

  name_prefix      = local.name_prefix          # e.g. "acme-prod"
  role             = "primary"
  engine_version   = "OpenSearch_2.17"

  instance_type    = var.opensearch_instance_type       # e.g. "r6g.large.search"
  instance_count   = var.opensearch_instance_count
  dedicated_master = true
  ebs_volume_gb    = var.opensearch_volume_gb

  vpc_id             = module.vpc_primary.vpc_id
  subnet_ids         = module.vpc_primary.private_subnet_ids
  kms_key_arn        = module.kms_primary.opensearch_key_arn

  custom_endpoint          = "search.${var.env}.${var.internal_zone}"
  custom_endpoint_cert_arn = module.acm_primary.search_cert_arn

  snapshot_bucket_arn = module.s3_primary.opensearch_snapshots_bucket_arn
}

module "opensearch_standby" {
  source = "../../modules/opensearch"
  providers = { aws = aws.standby }

  name_prefix      = local.name_prefix
  role             = "standby"
  engine_version   = "OpenSearch_2.17"   # must be <= leader at connection time

  instance_type    = var.opensearch_instance_type       # MUST match primary
  instance_count   = var.opensearch_standby_instance_count
  dedicated_master = true
  ebs_volume_gb    = var.opensearch_volume_gb

  vpc_id             = module.vpc_standby.vpc_id
  subnet_ids         = module.vpc_standby.private_subnet_ids
  kms_key_arn        = module.kms_standby.opensearch_key_arn   # DIFFERENT key

  custom_endpoint          = "search.${var.env}.${var.internal_zone}"
  custom_endpoint_cert_arn = module.acm_standby.search_cert_arn  # region-local cert

  snapshot_bucket_arn = module.s3_standby.opensearch_snapshots_bucket_arn
}
```

### The domain resource, with the constraints CCR imposes

```hcl
# modules/opensearch/main.tf

resource "aws_opensearch_domain" "this" {
  domain_name    = "${var.name_prefix}-search"   # ForceNew — see migration
  engine_version = var.engine_version

  cluster_config {
    instance_type            = var.instance_type   # NOT t2/t3/m3 — CCR forbids them
    instance_count           = var.instance_count
    zone_awareness_enabled   = true
    zone_awareness_config { availability_zone_count = 3 }

    dedicated_master_enabled = var.dedicated_master
    dedicated_master_type    = var.dedicated_master_type
    dedicated_master_count   = var.dedicated_master ? 3 : null
  }

  ebs_options {
    ebs_enabled = true
    volume_type = "gp3"
    volume_size = var.ebs_volume_gb
  }

  vpc_options {                                   # block itself is ForceNew
    subnet_ids         = var.subnet_ids
    security_group_ids = [aws_security_group.this.id]
  }

  # --- CCR prerequisites. All three are required on BOTH domains. ---
  encrypt_at_rest {
    enabled    = true
    kms_key_id = var.kms_key_arn                  # ForceNew. Get this right first time.
  }

  node_to_node_encryption { enabled = true }

  domain_endpoint_options {
    enforce_https                   = true
    tls_security_policy             = "Policy-Min-TLS-1-2-PFS-2023-10"
    custom_endpoint_enabled         = true
    custom_endpoint                 = var.custom_endpoint
    custom_endpoint_certificate_arn = var.custom_endpoint_cert_arn
  }

  advanced_security_options {
    enabled                        = true         # FGAC — required for CCR
    internal_user_database_enabled = false        # prefer IAM master user
    master_user_options {
      master_user_arn = var.master_user_role_arn
    }
  }

  log_publishing_options {
    log_type                 = "INDEX_SLOW_LOGS"
    cloudwatch_log_group_arn = aws_cloudwatch_log_group.index_slow.arn
  }

  access_policies = data.aws_iam_policy_document.domain.json

  tags = merge(var.tags, { Role = var.role })
}
```

The domain access policy on the **leader** must grant `es:ESCrossClusterGet`
on the domain ARN *without* the `/*` suffix — AWS calls this out explicitly and
it is a classic half-hour of debugging:

```hcl
data "aws_iam_policy_document" "domain" {
  statement {
    effect  = "Allow"
    actions = ["es:ESHttp*"]
    principals { type = "AWS" identifiers = [data.aws_caller_identity.current.account_id] }
    resources = ["${aws_opensearch_domain.this.arn}/*"]
  }

  dynamic "statement" {
    for_each = var.role == "primary" && var.replication_mode == "ccr" ? [1] : []
    content {
      effect  = "Allow"
      actions = ["es:ESCrossClusterGet"]
      principals { type = "AWS" identifiers = [data.aws_caller_identity.current.account_id] }
      resources = [aws_opensearch_domain.this.arn]   # NO trailing /*
    }
  }
}
```

### The cross-cluster connection

Terraform *can* do this, unlike CloudFormation. The connection is initiated
**from the follower** (CCR is a pull model), so the resource lives in the
standby stack with the primary as `remote_domain_info`:

```hcl
# environments/<env>/opensearch-ccr.tf
# Only instantiated when replication_mode == "ccr" (i.e. NOT the CA pair)

resource "aws_opensearch_outbound_connection" "standby_to_primary" {
  count    = var.replication_mode == "ccr" ? 1 : 0
  provider = aws.standby

  connection_alias  = "${var.env}-primary"
  connection_mode   = "DIRECT"
  accept_connection = true          # auto-accept; same account both sides

  local_domain_info {              # the FOLLOWER
    owner_id    = data.aws_caller_identity.current.account_id
    region      = var.standby_region
    domain_name = module.opensearch_standby.domain_name
  }

  remote_domain_info {             # the LEADER
    owner_id    = data.aws_caller_identity.current.account_id
    region      = var.primary_region
    domain_name = module.opensearch_primary.domain_name
  }
}
```

Notes:
- `connection_mode` is `DIRECT` or `VPC_ENDPOINT`. **`VPC_ENDPOINT` is
  same-region only** (OpenSearch's managed VPC endpoints do not cross regions),
  so cross-region must be `DIRECT`, and for VPC domains that means VPC peering
  or Transit Gateway must already exist between the two VPCs
  ([[aws-vpc-networking]]).
- `connection_properties.cross_cluster_search.skip_unavailable` is the CCS
  knob; it is irrelevant for replication.
- If the connection is in the same account both ways, `accept_connection = true`
  saves you the `aws_opensearch_inbound_connection_accepter` dance.
- **`connection_alias` is what you reference as `leader_alias` in the
  replication API call.** Output it from the stack; the runbook needs it.

### What Terraform cannot do

**There is no Terraform resource for starting replication.** The
`_plugins/_replication/_autofollow` call is an OpenSearch REST API on the data
plane, not an AWS control-plane API. Options:

- `null_resource` + `local-exec` with `awscurl`. Works. Ugly. Not idempotent in
  any useful sense, and it puts a signed HTTP call in your plan/apply cycle.
- The community `opensearch` Terraform provider (`opensearch-project/opensearch`)
  manages data-plane objects (roles, role mappings, ISM policies, index
  templates, Dashboards objects). **This is the better answer and it also
  solves the "stuff CCR doesn't carry" problem below**, which is the bigger
  prize. It means a second provider in the stack, configured against each
  domain's endpoint — which in a VPC domain means Terraform must run somewhere
  with network reach to the domain. That is a real constraint on a CI runner
  and it needs deciding early ([[state-management]]).
- A bootstrap job (Lambda or a CI step) that idempotently ensures the
  auto-follow rule exists. Pragmatic, and it can run on a schedule to
  self-heal a replication that stopped.

**Recommendation: the community provider for the config-as-data objects, and a
scheduled idempotent job for the auto-follow rule.** Do not put a `local-exec`
in the main apply path.

### Variable surface worth exposing

```hcl
variable "replication_mode" {
  description = <<-EOT
    How the standby domain is kept current.
      "ccr"      — cross-cluster replication. NOT available for the CA pair:
                   AWS forbids CCR between default and opt-in Regions, and
                   ca-central-1 is default while ca-west-1 is opt-in.
      "snapshot" — hourly manual snapshot to S3 + CRR + continuous restore.
      "none"     — derived index, rebuilt from source at failover.
  EOT
  type    = string
  default = "ccr"
  validation {
    condition     = contains(["ccr", "snapshot", "none"], var.replication_mode)
    error_message = "replication_mode must be ccr, snapshot or none."
  }
}

variable "role"                          { type = string }  # primary | standby
variable "standby_instance_count"        { type = number }
variable "snapshot_interval_hours"       { type = number, default = 1 }
variable "ccr_autofollow_patterns"       { type = list(string), default = ["*"] }
variable "saved_objects_ndjson_path"     { type = string }   # Dashboards export in git
```

---

## Migration path from single-region

The estate is live in one region. Getting to a mirrored pair without downtime
and without a destroy/recreate.

### Step 0 — classify the domain

Derived-and-fast-to-rebuild? Stop here, set `replication_mode = "none"`, build
a reindex job and a game day, and spend the money somewhere else. This step
takes an afternoon and frequently deletes the rest of the project.

### Step 1 — the ForceNew audit, before anything else

These are the things that will replace your **live, primary** domain if you
change them. From the provider source (`internal/service/opensearch/domain.go`,
`hashicorp/aws`):

| Attribute | ForceNew? | Notes |
|---|---|---|
| `domain_name` | **Yes, unconditionally** | Renaming the domain destroys it. If multi-region naming conventions tempt you to rename, **don't** — unlike [[aws-dynamodb]] Global Tables, CCR does **not** require matching names across regions, so there is no reason to rename. See [[dynamodb-table-naming-migration]] for the contrast. |
| `encrypt_at_rest.kms_key_id` | **Yes, unconditionally** | You cannot rotate to a different CMK without replacing the domain. Choose the key deliberately, now. [[aws-kms]] |
| `vpc_options` block present/absent | **Yes** (block-level `ForceNew`) | Moving a public domain into a VPC — or out — replaces it. Changing `subnet_ids` *within* an existing block is an in-place update (which AWS executes as blue/green). |
| `encrypt_at_rest.enabled` | **Conditionally** | `true → false` always ForceNew. `false → true` is in-place on any OpenSearch engine version; on Elasticsearch it is in-place only at 6.7+. |
| `node_to_node_encryption.enabled` | **Conditionally** | Same rule as above. |
| `advanced_security_options.enabled` (FGAC) | **Conditionally** | `true → false` is ForceNew — **and AWS does not allow disabling FGAC at all**, so this is a trap door. `false → true` is in-place (but a blue/green at the AWS layer). |
| `ip_address_type` | **Conditionally** | ForceNew when moving *away from* `dualstack`. |
| `engine_version` | **Conditionally** | ForceNew if the target is not in `GetCompatibleVersions` for the domain — i.e. a *downgrade* or a non-upgrade-path jump replaces the domain. |

**The single most dangerous line in a CCR migration is
`node_to_node_encryption { enabled = true }` being added to a domain where it
is currently false.** That is in-place for OpenSearch engines, so it plans
clean — but it is a **blue/green deployment** at the AWS layer, which doubles
the node count, copies every shard, and degrades latency while it runs. Do it
in a maintenance window, on a healthy cluster, at low traffic, and run the
**dry run** first (`DryRun: true, DryRunMode: "Verbose"` on
`UpdateDomainConfig`, or the console's "Dry run analysis") to confirm what
deployment type you are about to trigger. Terraform's default update timeout
for this resource is **180 minutes**, which tells you what AWS thinks the
worst case looks like.

Also note the trap door: **fine-grained access control cannot be disabled once
enabled.** Enabling it to satisfy CCR's prerequisite is permanent, and it
changes how every client authenticates. If clients use IP-based or open access
policies today, use the 30-day migration period (`AnonymousAuthEnabled: true`)
to give yourself a transition window — **AWS auto-disables it after 30 days and
it cannot be re-enabled**, so that is a hard deadline, not a soft one.

### Step 2 — prepare the primary in place

In order, each verified with a dry run:

1. Enable node-to-node encryption (blue/green) if not already on.
2. Enable encryption at rest with the chosen CMK (blue/green) if not already on.
3. Enable FGAC with an IAM master user, using the migration period. Create
   role mappings for every client. Disable the migration period early.
4. Upgrade the engine to OpenSearch 1.1+ if on anything older (blue/green).
5. Add the custom endpoint + ACM cert, and move clients onto the custom name.
   **Do this before anything else that matters** — it is non-disruptive
   ("Modifying the custom endpoint" is on the no-blue/green list) and it is
   what converts failover from a deploy into a DNS change.
6. Check `soft_deletes` on every index you intend to replicate. Reindex any
   that were created on ES 6.x.

Step 5 deserves emphasis. **Migrating clients from the auto-generated endpoint
to a custom endpoint is the highest-value, lowest-risk piece of work in this
entire note**, and it is worth doing even if you later decide not to replicate
at all.

### Step 3 — stand up the standby

New module instantiation against `aws.standby`. Nothing in the primary's state
changes; `terraform plan` on the primary should be empty. Verify that.

Standby domain must be on the **same or lower** engine version than the leader
at connection time.

### Step 4 — connect and start replicating

`aws_opensearch_outbound_connection` from follower to leader. Then the
auto-follow rule via the bootstrap job. Watch `ReplicationNumSyncingIndices`
climb to the expected count.

Create a dedicated replication user on **both** domains with **identical
usernames** — AWS is explicit that "the usernames must be identical" — and map
them, rather than using `all_access` as the docs' examples do for brevity.

### Step 5 — mirror the config-as-data

The long pole. See the next section.

### Step 6 — prove it

A game day that promotes the follower, flips DNS, serves real queries, and
then **fails back**, which is the part that will find the problems. Budget a
full day and expect the first attempt to fail on something in Step 5.

---

## The stuff CCR doesn't carry (the gotcha that deserves its own section)

CCR replicates "user indexes, mappings, and metadata". Everything below lives
in the cluster and does **not** cross. Snapshots don't rescue you either — AWS's
restore guidance explicitly tells you to exclude these indexes.

### 1. Dashboards saved objects

Index patterns, visualisations, dashboards, saved searches, and (with
multi-tenancy) a separate `.kibana` index per tenant. They are documents in a
system index. **They are not replicated, not restorable, and not in your
Terraform.**

The supported route is the **saved objects export/import API**, which produces
an `.ndjson` file. So:

- Export saved objects from the primary in CI (`GET /api/saved_objects/_export`
  via the Dashboards API, signed).
- **Commit the `.ndjson` to the Terraform repo.** Dashboards become
  version-controlled artefacts, which is a good thing independent of DR.
- Import into the standby on every apply (`POST /api/saved_objects/_import`).

Two real hazards:

- **Exporting only the top-level dashboard produces an import with broken
  panels.** Export with `includeReferences`, or export the whole object graph.
- **Saved-object migrations are version-sensitive**: an export from a *newer*
  Dashboards version fails to import into an older one. Since the standby must
  be at the same or lower engine version than the leader at CCR connection
  time, **you can end up with exports the standby cannot import.** Keep the two
  domains on the same engine version except during the upgrade window, and
  upgrade the follower *first* (which is also what AWS's CCR upgrade guidance
  requires: "upgrade the follower domain first and then the leader domain").
- **Do not copy `.kibana*` documents directly between clusters** as a shortcut.
  It bypasses the migration machinery and produces a Dashboards install that
  half-works in ways that are hard to diagnose.

### 2. The fine-grained access control configuration

AWS is unambiguous about the internal user database: it "is stored in an
OpenSearch index, so you can't share it with other clusters."

So per-region you must recreate:
- FGAC **roles** (cluster/index/document/field permissions, field masking)
- **Role mappings** (users, backend roles, IAM role ARNs)
- Tenants, if using multi-tenancy
- Internal users and their passwords, if using the internal user database

**Recommendation: do not use the internal user database.** Use an **IAM master
user** and map IAM role ARNs as backend roles. IAM roles are global
([[aws-iam]]), so the *identities* exist in both regions for free and only the
*mappings* need duplicating. AWS's own guidance points the same way: "We
recommend IAM if you want to use the same users on multiple clusters." If you
must have human logins to Dashboards, use **SAML** (`aws_opensearch_domain_saml_options`)
against a central IdP rather than per-domain passwords.

The mappings themselves are data-plane objects. Manage them with the community
`opensearch` provider so both regions are driven from the same HCL, or with an
idempotent bootstrap job. Either way: **an IAM role ARN embedded in a role
mapping is region-agnostic, but the mapping document is not — it must exist in
both clusters.**

Worth noting for [[security-posture-of-the-standby]]: a standby domain with no
role mappings is not "secure", it is *broken*, and it will fail at 3am with
`no permissions for [indices:data/read/search]` — a message that looks like an
application bug, not a DR gap.

### 3. ISM policies and index templates

ISM policies live in `.opendistro-ism-config`; index templates are cluster
state. Neither is replicated.

**And there is a correctness hazard, not just a gap:** if an ISM policy with a
rollover action is attached on the *follower* while replication is active, you
have two systems trying to manage the same index lifecycle. Do not attach
rollover policies to follower indexes. Attach the standby's ISM policies at
failover, or write follower-specific policies that omit rollover. (This is
widely stated in community documentation; **no AWS first-party statement of
the rule was found**, so treat the mechanism as reasoned rather than cited, and
test it.)

Practically: keep ISM policies and index templates as JSON in the Terraform
repo and apply them to both domains from CI. They are small, they change
rarely, and they are exactly the kind of thing that silently drifts.

### 4. Everything else that lives in the cluster

- Alerting monitors, triggers, destinations, notification channels
- Anomaly detectors
- Snapshot repository registrations (`_snapshot/<repo>`) — the standby needs
  its own, pointed at the replica bucket, with a standby-region IAM role
- Custom packages / dictionaries (`aws_opensearch_package` + association)
- SQL/PPL saved queries
- The `.plugins-ml-config` index if ML plugins are in use

**Rule of thumb: if you configured it by talking to the cluster rather than to
the AWS API, CCR will not carry it and neither will a snapshot restore.**
Inventory those objects before designing anything else. That inventory is a
better use of the first week than any Terraform.

---

## Failover procedure

Assume CCR (EU/US). Annotated with what is automatable.

| # | Step | Automatable? | Time |
|---|---|---|---|
| 1 | Declare the failover | **Human.** Always. | — |
| 2 | Fence the primary: stop writers in the primary region | Automated, but the fence must survive the primary coming back — [[split-brain-and-fencing]] | < 1 min |
| 3 | Check `FollowerCheckPoint` vs last known `LeaderCheckPoint`; record the delta as the realised RPO | Automated (script), **read it before step 4** | seconds |
| 4 | `POST _plugins/_replication/<index>/_stop` for every follower index | Automated (loop over `_cat/indices`) | seconds–2 min |
| 5 | Delete the auto-follow rule so it does not recreate followers | Automated | seconds |
| 6 | Import Dashboards saved objects if not pre-loaded | Should be pre-loaded; if not, minutes | 0–5 min |
| 7 | Verify FGAC role mappings resolve for the application role | Automated smoke test | seconds |
| 8 | Flip the Route 53 record for the custom endpoint to the standby | Automated — [[aws-route53]], [[failover-orchestration]] | 1–2 min inc. TTL |
| 9 | Smoke test: index a document, search it, check a dashboard renders | Automated | 1 min |
| 10 | Enable standby writers | Automated | 1 min |

**Total: comfortably under 15 minutes if steps 6 and 7 were done in advance.**
If they were not, they are the steps that blow the RTO, and they are the two
steps that are invisible until the day you need them. That is the argument for
the game day.

**Step 3 is the one people skip and the one that matters afterwards.** Record
the checkpoint delta *before* you stop replication; once you stop, the evidence
of how far behind you were is gone, and "how much did we lose?" becomes
unanswerable at the post-incident review.

**The ungraceful case.** If the primary region is unreachable, step 2 cannot
execute against the primary and step 4's `_stop` still works (it is issued to
the *follower*, which is alive). Good. But note that the cross-cluster
connection will be in a failed state and the follower may need the
force-resume/stop path; test this specific case, because the happy-path runbook
is written against a reachable leader.

### CA-pair variant (snapshot path)

Steps 3–5 become: verify the newest snapshot in the standby bucket is inside
RPO, confirm the continuously-restored shadow indexes are current, then **flip
the aliases** from `restore-*` to the live names. Stop writes first (the alias
hazard above). Everything else is identical.

---

## Failback

Harder than failover, and with CCR it is genuinely nasty. **Read this before
choosing CCR, not after.**

The problem: `_stop` is irreversible. The promoted follower is now a standalone
writable index with new data in it. The old primary still holds the old leader
index. There is no "resync" operation. AWS's own blog states the fix plainly:
reversing replication requires **deleting the original index** and setting up
replication in the opposite direction, which "will bootstrap the index and
start the replication from scratch" and "may take time depending upon the size
of the index."

So failback is:

1. Old primary region recovers.
2. **Fence it.** Its old indexes are stale and its clients may still be
   configured against it. This is the [[messaging-in-flight-data-loss]]
   "un-fenced primary comes back" failure mode, applied to search: stale search
   results served to real users, which is quieter and therefore worse than an
   outage.
3. Delete the stale indexes on the old primary.
4. Create a cross-cluster connection in the **reverse** direction
   (`old-primary` follows `new-primary`). Note the version rule: the leader
   must be at the same or higher version than the follower.
5. Start replication / auto-follow. **Wait for a full bootstrap** — the entire
   dataset transfers. Hours for a large index.
6. Once `SYNCING` and checkpoints converge, repeat the failover procedure in
   reverse: fence, `_stop`, flip DNS.
7. Rebuild the original direction of replication. Which means deleting the
   indexes in the *other* region and bootstrapping again.

**That is two full bootstraps to get back to where you started.** For a
multi-terabyte index that is a multi-day operation with two write-freeze
windows.

**Implications, and they are strategic:**

- **Do not treat failback as an afterthought or as "the reverse of failover".**
  Put it in the runbook with its own duration estimate and its own maintenance
  window.
- **Consider not failing back.** If the two regions are symmetrical — same
  Terraform, same sizing, same everything — then "the standby is now the
  primary" is a legitimate end state, and you just rebuild replication in the
  new direction once. This flips the region roles in Terraform (a variable
  change, if the module is written as above) and avoids one bootstrap entirely.
  **This is the recommendation**, and it is a strong argument for making the
  primary/standby designation a variable rather than a structural property of
  the repo. [[provider-aliases-vs-separate-stacks]] and [[module-patterns]].
- **The snapshot path fails back more gracefully**, which is an underrated
  point in its favour: reverse the snapshot direction, restore, flip aliases.
  No irreversible operation, no double bootstrap. For the CA pair, the enforced
  snapshot path is not purely a downgrade.
- The rebuild-from-source path fails back most gracefully of all: reindex in
  the other direction. Nothing is irreversible because nothing was ever
  authoritative.

---

## Gotchas

1. **CCR between `ca-central-1` and `ca-west-1` is impossible.** Default vs
   opt-in Region. Verified against AWS docs. This is the one to put in front of
   whoever owns [[region-pair-selection]].
2. **OpenSearch Serverless has no cross-region anything.** "Cross-Region search
   and replication aren't supported." Manual snapshots aren't supported either
   — only automated ones, which can't be restored elsewhere. And it doesn't
   exist in `ca-west-1`. If a Bedrock Knowledge Base uses an AOSS vector
   collection, that Knowledge Base is single-region, full stop
   ([[aws-bedrock]]).
3. **Automated snapshots cannot be restored into another domain.** Only manual
   ones. The free hourly snapshots are not a DR asset.
4. **A Glacier lifecycle rule on the snapshot bucket silently breaks DR.**
   Manual snapshots don't support the Glacier storage class.
5. **Dashboards, users, roles, ISM policies and index templates do not
   replicate** — not by CCR, not by snapshot restore (AWS tells you to exclude
   those indexes). This is the gap that turns a "successful" failover into a
   cluster nobody can log into.
6. **The internal user database cannot be shared between clusters** (AWS's
   words). Use an IAM master user.
7. **`encrypt_at_rest.kms_key_id` is `ForceNew`.** You cannot change the CMK
   without destroying the domain. Decide the key strategy before you create the
   standby. [[aws-kms]]
8. **`domain_name` is `ForceNew`.** Do not rename domains for multi-region
   tidiness. CCR does not need matching names.
9. **FGAC cannot be disabled once enabled**, and enabling it is a blue/green
   deployment, and the 30-day open-access migration period is
   auto-disabled and **cannot be re-enabled**.
10. **Replication paused for more than 12 hours cannot be resumed.** You must
    stop, delete the follower index, and re-bootstrap. Alarm well before 12h.
11. **`_stop` is irreversible.** Promotion is a one-way door and failback needs
    a full re-bootstrap in each direction.
12. **Deleting an index on the leader does not delete it on the follower.** The
    standby accumulates orphans and eventually fills its disk.
13. **A connection created for cross-cluster *search* is `SEARCH_ONLY`** and
    cannot be used for replication — delete and recreate it.
14. **No CCR on T2/T3/M3 instances, and no CCR of UltraWarm or cold indexes.**
    A dev domain on `t3.small.search` cannot be used to rehearse this, which
    means your rehearsal environment costs real money.
15. **`VPC_ENDPOINT` connection mode is same-region only.** Cross-region VPC
    domains need VPC peering or Transit Gateway.
16. **The domain endpoint contains the region.** Without a custom endpoint,
    failover is an application deploy. With one, it is a DNS record.
17. **The ACM certificate for a custom endpoint must be in the same region as
    the domain** — so the same hostname needs a cert in each region
    ([[aws-acm]]).
18. **Blue/green deployments roughly double the node count temporarily** and
    increase latency and rejections. Never plan an instance-type change into a
    failover runbook. Use the dry-run API before any config change on a live
    domain.
19. **Terraform's update timeout on `aws_opensearch_domain` is 180 minutes.**
    That is the vendor telling you how long a bad day looks.
20. **You cannot delete a domain with active cross-cluster connections.** Tear
    down the connection first — otherwise a `terraform destroy` of a dev
    environment hangs.
21. **Multi-AZ with Standby imposes replica-count constraints on restore.** A
    snapshot whose indexes have incompatible replica counts will fail to restore
    unless you override with `index_settings`.

---

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Is this domain in DR scope at all? | Replicate it | Rebuild from source at failover | **Classify per domain.** Most product-search indexes are derived data and should be rebuilt. Spend the saved budget on the one or two that aren't. |
| EU/US replication mechanism | CCR | Hourly manual snapshot + continuous restore | **CCR.** Sub-minute RPO, seconds-long promotion, and it is the AWS-blessed path. Accept the painful failback. |
| CA pair replication mechanism | — | Snapshot + CRR + continuous restore into renamed indexes, alias-flip at failover | **Forced.** CCR is unavailable. Build the alias-flip pattern or accept rebuild-from-source. |
| Standby sizing | Match primary exactly | Fewer nodes, scale count at failover | **Match instance type exactly; allow lower node count if storage permits.** Node-count changes avoid blue/green; instance-type changes do not. |
| Client endpoint | Auto-generated `search-*.region.es.amazonaws.com` | Custom endpoint + ACM + Route 53 | **Custom endpoint, immediately.** Converts a failover deploy into a DNS change. Do this even if you never replicate. |
| FGAC identity source | Internal user database | IAM master user (+ SAML for humans) | **IAM.** The internal database is explicitly not shareable between clusters. |
| Config-as-data (roles, ISM, templates, dashboards) | Manual / console | Community `opensearch` Terraform provider + `.ndjson` in git | **In git, applied by CI to both regions.** This is the failure that ruins an otherwise-good failover. |
| Failback strategy | Fail back to the original primary | **Stay** in the new region; treat primary/standby as a variable | **Stay.** Failback via CCR costs two full bootstraps. Symmetric Terraform makes staying free. |
| Logs / observability domain | CCR the domain | Dual-ship from the collection agents | **Dual-ship.** Cheaper, works in `ca-west-1`, and keeps your incident telemetry alive when you need it most. [[observability-multi-region]] |
| Bedrock Knowledge Base vector store | AOSS Serverless | OpenSearch managed cluster as vector store | **Managed cluster if DR matters.** Bedrock Knowledge Bases gained support for OpenSearch Managed Cluster vector storage in March 2025; AOSS has no cross-region path. Verify current region support before committing. |

---

## Cost

**No verified per-instance hourly price for `eu-west-2` or `eu-west-1` was
retrievable from the pricing page in this research pass** — the AWS pricing
page renders its tables client-side and the fetched content did not include
them. Do not take a number from this note; price it in the AWS Pricing
Calculator against the actual instance type and node count. Flagging this
rather than estimating, per the vault's standards.

What *was* verified:

- **OpenSearch Serverless is billed at $0.24 per OCU-hour** (from the pricing
  page's worked example). **Classic collections bill a minimum of 2 OCUs**
  (1 OCU indexing including a standby, 1 OCU search including a replica) — so a
  Classic collection has a non-trivial floor cost even when idle. NextGen
  collections have "no minimum OCU requirement" and scale to zero after 10
  minutes of inactivity. A dev/test option at half cost with redundancy
  disabled exists.
- **You are not double-billed during blue/green deployments** in the general
  case: "If you don't change the instance type, you're charged only for the
  largest cluster for the first hour." If you *do* change instance type, you pay
  for both clusters for the first hour only.
- **Cross-cluster search has no additional charge.** Cross-cluster replication
  costs **standard inter-region data transfer** for the replicated bytes.
- **Automated snapshots are free** (AWS-managed bucket). **Manual snapshots
  cost standard S3.**

The cost shape, qualitatively:

| Line | Driver | Lever |
|---|---|---|
| Standby data nodes | Instance type × count × hours | Node *count* (safe to change), not type |
| Standby dedicated masters | 3 × master instance | Only omit if the primary omits them |
| Standby EBS | GB × gp3 rate | Same as primary — CCR needs the storage |
| Inter-region data transfer | Replicated bytes | The real variable cost of CCR on a high-ingest domain. **Model this for a log domain before choosing CCR** — it can dominate. |
| S3 snapshots + CRR | Snapshot size × frequency × retention, plus CRR transfer | Snapshot frequency; RTC on/off |
| Orphaned follower indexes | Leader deletes not propagating | A reconciliation job |

**The cheapest thing in this note is deciding a domain doesn't need
replicating.** The second cheapest is the custom endpoint. Do both before
buying a standby domain.

---

## Open questions

1. **How many OpenSearch domains are there, and what is each one for?** The
   classification in "should this be replicated at all" cannot be done without
   this, and it changes the cost by an order of magnitude.
2. **Is fine-grained access control already enabled on the live domains?** If
   not, enabling it is a blue/green deployment, an irreversible decision, and a
   client-authentication change — a project in its own right, and a hard
   prerequisite for CCR.
3. **Do any indexes predate Elasticsearch 7.0?** If so they may have
   `soft_deletes=false` and need reindexing before CCR will touch them.
4. **Is anything on T2/T3 instances?** Those domains cannot use CCR at all.
5. **Is UltraWarm in use?** Warm and cold indexes cannot be replicated.
6. **Does anything in the estate use OpenSearch Serverless?** Specifically: is
   there a Bedrock Knowledge Base? If yes, its vector store is single-region and
   somebody needs to decide whether that is acceptable or whether to move to a
   managed cluster.
7. **How big is the largest index, and how long does a full reindex from source
   take?** This one number decides rebuild-vs-replicate for most domains.
8. **Are dashboards business-critical or convenience?** If an operations team
   depends on them, their absence in the standby is an RTO failure even though
   no customer traffic depends on them.
9. **Who owns the Dashboards saved objects today, and would they accept them
   becoming a git artefact?** The export/import-from-CI pattern is a workflow
   change for whoever builds dashboards.
10. **Does the CA deployment have a residency constraint that forbids pairing
    `ca-central-1` with a US region?** If not, the CCR blocker evaporates (at
    the cost of leaving Canada). If it does, the CA pair is snapshot-only.
    [[data-residency]]
11. **Can the CI runner reach a VPC domain's endpoint?** Required for the
    community `opensearch` provider and for any config-as-data automation.

---

## Sources

- [Cross-cluster replication for Amazon OpenSearch Service](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/replication.html)
  — the authoritative limitation list, including the default-vs-opt-in Region
  restriction that blocks the CA pair, the T2/T3/M3 exclusion, the UltraWarm
  exclusion, the FGAC and node-to-node-encryption prerequisites, the
  `soft_deletes` requirement, the 12-hour pause limit, the irreversibility of
  `_stop`, and the `es:ESCrossClusterGet` policy shape.
- [Enable or disable AWS Regions in your account](https://docs.aws.amazon.com/general/latest/gr/rande-manage.html)
  — the definitive default/opt-in Region tables. `ca-west-1` is listed as
  opt-in; `ca-central-1`, `eu-west-1`, `eu-west-2`, `us-east-1`, `us-west-2` are
  all default. This is what makes the CA blocker a fact rather than a guess.
- [Amazon OpenSearch Service endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/opensearch-service.html)
  — region parity, verified per flavour: managed domains **are** in `ca-west-1`;
  OpenSearch **Serverless is not**; OpenSearch **Ingestion is not**. Also the
  Serverless OCU quotas.
- [What is Amazon OpenSearch Serverless?](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-overview.html)
  — the limitations list containing "Cross-Region search and replication aren't
  supported" and "Manual snapshots are not supported". The headline Serverless
  finding.
- [Creating index snapshots in Amazon OpenSearch Service](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/managedomains-snapshots.html)
  — hourly automated snapshots, 336 retained over 14 days, `cs-automated` /
  `cs-automated-enc`, the `TheSnapshotRole` IAM shape, and the Glacier
  prohibition.
- [Restoring data from snapshots](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/managedomains-snapshot-restore.html)
  — "Restore the snapshot to a different OpenSearch Service domain (only
  possible with manual snapshots)", the `-.kibana*,-.opendistro*` exclusion
  pattern, the alias-deletion hazard, and the Multi-AZ-with-Standby replica
  constraint.
- [Fine-grained access control in Amazon OpenSearch Service](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/fgac.html)
  — "The internal user database is stored in an OpenSearch index, so you can't
  share it with other clusters", the IAM-vs-internal master user trade-off, the
  irreversibility of enabling FGAC, and the 30-day migration period that AWS
  auto-disables.
- [Making configuration changes in Amazon OpenSearch Service](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/managedomains-configuration-changes.html)
  — which changes trigger blue/green (instance type, enabling FGAC, enabling
  encryption, changing subnets) and which don't (data node count, custom
  endpoint, access policy), the node-count doubling, the dry-run API, and the
  billing rules during a deployment.
- [Cross-cluster search in Amazon OpenSearch Service](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/cross-cluster-search.html)
  — the CCS limitation list, the VPC-peering/Transit-Gateway requirement for
  VPC domains, `skip_unavailable` semantics, and the `connection-alias:index`
  Dashboards implication.
- [Ensure availability of your data using cross-cluster replication with Amazon OpenSearch Service](https://aws.amazon.com/blogs/big-data/ensure-availability-of-your-data-using-cross-cluster-replication-with-amazon-opensearch-service/)
  — AWS's "typical delivery times are less than a minute" lag figure, the
  two-step promotion, the acknowledgement of small data loss at failover, and
  the statement that reversing replication requires deleting the index and
  bootstrapping from scratch. The primary source for the failback section.
- [AOSREL04-BP04 Employ cross-cluster replication to achieve higher availability](https://docs.aws.amazon.com/wellarchitected/latest/amazon-opensearch-service-lens/aosrel04-bp04.html)
  — the OpenSearch Well-Architected Lens position: CCR is the recommended
  availability mechanism, rated "Medium" risk if absent. Useful for justifying
  the work internally.
- [Achieve cross-Region resilience with Amazon OpenSearch Ingestion](https://aws.amazon.com/blogs/big-data/achieve-cross-region-resilience-with-amazon-opensearch-ingestion/)
  — the OSI-based active-active pattern, explicitly stated to apply to both
  managed domains and Serverless collections; the only published cross-region
  path for AOSS. Quotes the S3 "99.99% of objects replicated within 15 minutes"
  RTC figure.
- [Creating a custom endpoint for Amazon OpenSearch Service](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/customendpoint.html)
  — the ACM-certificate-plus-CNAME mechanism that makes DNS-based failover
  possible.
- [Amazon OpenSearch Service now supports Amazon Route 53 alias record for domain endpoint](https://aws.amazon.com/about-aws/whats-new/2024/04/amazon-opensearch-service-route-53-alias-record-custom-endpoint)
  — alias records as an alternative to CNAME, requiring the dual-stack IP
  address type.
- [Amazon OpenSearch Service pricing](https://aws.amazon.com/opensearch-service/pricing/)
  — the $0.24/OCU-hour Serverless figure, the 2-OCU Classic minimum, NextGen's
  scale-to-zero, and the no-additional-charge statement for cross-cluster
  search.
- [`aws_opensearch_outbound_connection` — Terraform AWS provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/opensearch_outbound_connection)
  — the argument reference used in the HCL above: `connection_alias`,
  `connection_mode` (`DIRECT` / `VPC_ENDPOINT`), `accept_connection`,
  `local_domain_info` / `remote_domain_info`.
- [`hashicorp/aws` provider source, `internal/service/opensearch/domain.go`](https://github.com/hashicorp/terraform-provider-aws/blob/main/internal/service/opensearch/domain.go)
  — read directly for the `ForceNew` table: `domain_name` and
  `encrypt_at_rest.kms_key_id` unconditionally; the `vpc_options` block at
  block level; conditional ForceNew via `CustomizeDiff` for
  `engine_version`, `encrypt_at_rest.enabled`,
  `node_to_node_encryption.enabled`, `advanced_security_options.enabled` and
  `ip_address_type`. Also the 120/180/90-minute create/update/delete timeouts.
- [Amazon Bedrock Knowledge Bases now supports Amazon OpenSearch Managed Cluster for vector storage](https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-bedrock-knowledge-bases-opensearch-cluster-vector-storage)
  — the escape hatch from the AOSS single-region trap for Knowledge Bases.
- [Export and import Kibana dashboards with Amazon OpenSearch Service](https://aws.amazon.com/blogs/big-data/export-and-import-kibana-dashboards-with-amazon-opensearch-service/)
  — AWS's own saved-objects export/import walkthrough; the basis for the
  `.ndjson`-in-git recommendation.
- [Cross-cluster replication — OpenSearch project documentation](https://docs.opensearch.org/latest/tuning-your-cluster/replication-plugin/index/)
  — the upstream plugin docs, including the permissions model and the
  leader/follower cluster role mapping requirement that AWS's docs defer to.

### Searched for and did not find

- **A published postmortem or case study of an organisation failing over an
  Amazon OpenSearch Service domain cross-region in anger.** No public example
  found. The material that exists is vendor documentation and walkthroughs; the
  operational reality of promotion-under-pressure is undocumented.
- **An independent benchmark of CCR replication lag on Amazon OpenSearch
  Service under sustained heavy indexing.** Not found. AWS's "less than a
  minute" is the only figure available and it is first-party. Measure your own.
- **An AWS first-party statement that ISM rollover policies must not be
  attached to follower indexes.** The hazard is widely described in community
  material and follows logically from followers being read-only, but no AWS
  documentation stating it was located. Treat as reasoned, and test it.
- **Per-region on-demand instance pricing for OpenSearch Service.** The pricing
  page's tables did not render in the fetched content. Deliberately not
  estimated.
- **Any AWS statement on whether cross-cluster *search* (as opposed to
  replication) is blocked between default and opt-in Regions.** The restriction
  is stated only on the replication page. Unverified for CCS — assume it may
  apply and test before relying on it for the CA pair.
