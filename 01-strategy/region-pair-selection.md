---
title: Region Pair Selection — validating the three working pairs
tags: [strategy, multi-region, region-selection, ca-west-1, parity]
status: researched
replication: N/A — this note is about where, not how
rpo_achievable: N/A
rto_achievable: N/A
meets_targets: conditional — the CA pair has real, named gaps
updated: 2026-09-17
---

# Region Pair Selection

The [[research-brief]] carries three region pairs as a **working assumption**. This
note exists to confirm or break them. It is the note the rest of the vault is
downstream of: every other note has been written against `eu-west-2`,
`us-west-2` and `ca-west-1` as the standby regions, and if one of those is
wrong, a lot of HCL gets rewritten.

| Pair | Primary | Standby (assumed) |
|---|---|---|
| EU | `eu-west-1` Ireland | `eu-west-2` London |
| US | `us-east-1` N. Virginia | `us-west-2` Oregon |
| CA | `ca-central-1` Montreal | `ca-west-1` Calgary |

## TL;DR

- **EU pair: confirmed.** `eu-west-2` has full parity for everything in scope,
  has *more* AZs than Ireland (4 vs 3), and is ~12 ms away. It costs about 4%
  more on compute and, surprisingly, **less** on Aurora. The only real objection
  to London is legal, not technical — see [[data-residency]].
- **US pair: confirmed, with one caveat.** `us-west-2` is the most complete
  non-`us-east-1` region AWS runs, is priced **identically** to `us-east-1` on
  every line item checked, and is the natural pair. The caveat is AZ count:
  `us-east-1` has **6** AZs, `us-west-2` has **4**. If N. Virginia production
  spans more than four, the standby is not a 1:1 mirror.
- **CA pair: conditional — and this is the finding that changes the plan.**
  `ca-west-1` clears more of the parity bar than expected (Aurora Global
  Database, ElastiCache Global Datastore, EKS, API Gateway, Global Accelerator,
  zonal shift, AWS Backup cross-Region copy are all present). But it has
  **three hard gaps**: no **MemoryDB at all**, **no `m7i`/`m7g`/`m7a`/`m6a`/`c6a`/`r6a`/`r7i`
  and no GPU or high-memory families** (68 EC2 instance families present in
  Montreal are absent in Calgary), and **API Gateway throttle quotas at one
  quarter** of `ca-central-1`'s. Plus it is an **opt-in region**.
- **None of those gaps justify leaving Canada.** They are all addressable inside
  the CA deployment by changing what it runs on, and the alternative — standing
  the Canadian standby up in a US region — has data-residency consequences that
  are a far bigger problem than re-benchmarking a node group. **Recommendation:
  keep all three pairs; treat the CA pair's instance-family constraint as a
  design input, not a blocker.**
- **The one thing that will bite:** nobody will discover the missing instance
  families until `terraform apply` fails in Calgary — or worse, until a failover
  scales a node group that has no capacity to scale into. Run the verification
  command in [[#Verify before you commit]] this week.

---

## What "parity" was actually checked against

Service *presence* in a region is not parity. A region can have a service and
still be unable to host your deployment. Four separate things were checked, in
increasing order of how often they are forgotten:

1. **Does the service exist there?** (published regional endpoint)
2. **Does the *cross-region feature* of that service exist there?** Aurora
   Global Database, DynamoDB Global Tables, ElastiCache Global Datastore and
   MemoryDB Multi-Region each have their own, shorter region list than the base
   service.
3. **Are the quotas the same?** A region can have API Gateway and give you a
   quarter of the throughput.
4. **Are the underlying primitives the same?** Instance families, AZ count,
   AZ IDs. This is where `ca-west-1` actually fails.

Everything below is from an AWS-published page or the AWS Price List bulk API,
not from a third-party "supported regions" table. Several third-party lists
still quote `ca-west-1`'s December 2023 launch inventory and are wrong.

---

## Service parity matrix

✅ present · ⚠️ present but degraded/caveated · ❌ absent

| Capability | `eu-west-2` | `us-west-2` | `ca-west-1` | Source |
|---|---|---|---|---|
| EKS control plane | ✅ | ✅ | ✅ `eks.ca-west-1.amazonaws.com` | [EKS endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/eks.html) |
| EKS Pod Identity (`eks-auth`) | ✅ | ✅ | ✅ | same |
| EKS FIPS endpoint | ❌ | ✅ | ❌ | same |
| RDS PostgreSQL | ✅ | ✅ | ✅ | [[aws-rds-postgres]] |
| RDS cross-Region automated backup replication | ✅ | ✅ | ✅ `ca-central-1 ↔ ca-west-1` is in the matrix | [Replicating automated backups](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReplicateBackups.html) |
| Aurora PostgreSQL | ✅ | ✅ | ✅ | [[aws-aurora-global-database]] |
| **Aurora Global Database** | ✅ | ✅ | ✅ **Canada West listed for Aurora PG 11–18, same version floors as everywhere else** | [Supported Regions for Aurora global databases](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.Aurora_Fea_Regions_DB-eng.Feature.GlobalDatabase.html) |
| Aurora `db.r8g` instance class | ✅ | ✅ | ❌ **absent** (Price List: no `InstanceUsage:db.r8g.large` SKU in `ca-west-1`) | AWS Price List `AmazonRDS`, v20260910195514 |
| DynamoDB | ✅ | ✅ | ✅ | [[aws-dynamodb]] |
| DynamoDB Global Tables | ✅ | ✅ | ⚠️ *stated* available wherever DynamoDB is; **no `ca-west-1`-specific announcement found** | [[aws-dynamodb]] |
| DynamoDB MRSC (strong consistency) | ✅ (EU Region set) | ✅ (US Region set) | ❌ **no Canadian MRSC Region set exists** | [[aws-dynamodb]] |
| **ElastiCache Global Datastore** | ✅ London | ✅ Oregon | ✅ **Canada West (Calgary) listed** | [Global datastore prerequisites and limitations](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Redis-Global-Datastores-Getting-Started.html) |
| **MemoryDB (base service)** | ✅ | ✅ | ❌ **not in the supported-Regions endpoint table** | [MemoryDB Choosing Regions and AZs](https://docs.aws.amazon.com/memorydb/latest/devguide/regionsandazs.html) |
| **MemoryDB Multi-Region** | ✅ London | ✅ Oregon | ❌ **no Canadian region at all** — list is US East/West, EU Ireland/Frankfurt/London, AP Tokyo/Sydney/Mumbai/Seoul/Singapore | [MemoryDB Multi-Region](https://docs.aws.amazon.com/memorydb/latest/devguide/multi-region.html) |
| API Gateway (REST/HTTP, control + data plane) | ✅ | ✅ | ✅ incl. FIPS | [API Gateway endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/apigateway.html) |
| **API Gateway throttle quota** | ✅ 10,000 rps / 5,000 burst | ✅ 10,000 / 5,000 | ⚠️ **2,500 rps / 1,250 burst — one quarter** | same |
| Global Accelerator endpoints | ✅ | ✅ | ✅ since 25 Apr 2024 | [[aws-alb-nlb]]; [What's New](https://aws.amazon.com/about-aws/whats-new/2024/04/aws-global-accelerator-endpoints-calgary-region) |
| ARC zonal shift / autoshift | ✅ | ✅ | ✅ `arc-zonal-shift.ca-west-1.amazonaws.com` incl. FIPS | [Region availability for zonal shift](https://docs.aws.amazon.com/r53recovery/latest/dg/introduction-regions-zonal.html) |
| ARC routing controls | ✅ global | ✅ global | ✅ global | [Region availability for routing control](https://docs.aws.amazon.com/r53recovery/latest/dg/introduction-regions-routing.html) |
| **ARC readiness check** | ❌ **closed to new customers** | ❌ same | ❌ same | [ARC readiness check availability change](https://docs.aws.amazon.com/r53recovery/latest/dg/arc-readiness-availability-change.html) |
| EventBridge cross-Region bus target | ✅ | ✅ | ✅ (all commercial regions since Nov 2021) | [Cross-Region support expands](https://aws.amazon.com/about-aws/whats-new/2021/11/amazon-eventbridge-cross-region-expands) |
| AWS Backup — cross-Region copy | ✅ | ✅ | ✅ | [AWS Backup feature availability](https://docs.aws.amazon.com/aws-backup/latest/devguide/backup-feature-availability.html) |
| **AWS Backup Audit Manager + Jobs dashboard** | ✅ | ✅ | ❌ **blank in the Canada West row** | same |
| AWS Backup — logically air-gapped vault (x-region copy *into*) | ✅ | ✅ | ❌ explicitly excluded in footnote 1 | same |
| ACM (regional) | ✅ | ✅ | ✅ incl. FIPS | [[aws-acm]] |
| KMS | ✅ | ✅ | ✅ incl. FIPS | [[aws-kms]] |
| **KMS symmetric crypto-op quota** | ⚠️ 20,000/s | ✅ 100,000/s | ⚠️ 10,000/s (same as `ca-central-1`) | [KMS endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/kms.html) |
| Secrets Manager (+ replication) | ✅ | ✅ | ✅ incl. FIPS | [[aws-secrets-manager]] |
| SSM Parameter Store | ✅ | ✅ | ✅ incl. FIPS | [[aws-ssm-parameter-store]] |
| IAM | ✅ global | ✅ global | ✅ global — **but see opt-in, below** | [[aws-iam]] |
| **Region enablement** | ✅ default-on | ✅ default-on | ⚠️ **opt-in required** | [Enable or disable AWS Regions](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-regions.html) |
| EFS + EFS replication | ✅ | ✅ | ✅ | [[eks-stateful-workloads]] |
| WAFv2 `REGIONAL` scope | ✅ | ✅ | ⚠️ **unverified — check before sign-off** | [[aws-alb-nlb]] |
| Route 53 health-checker managed prefix list | ✅ | ✅ | ⚠️ **unverified** | [[aws-route53]] |
| CloudWatch cross-region metric centralisation | ✅ $0.05/GB | ✅ | ❌ no `CentralizedBytes` SKU in the `ca-west-1` price file | AWS Price List `AmazonCloudWatch` |

### Two findings that apply to *all three* pairs

**ARC readiness check is closed to new customers.** AWS: *"The readiness check
feature in Amazon Application Recovery Controller (ARC) is no longer open to new
customers. Existing customers can continue to use the service as normal."* This
company is a new customer, so readiness checks are **not available to it at
all** — in any region. AWS's stated migration path is **ARC Region switch**,
which has a "plan evaluation" capability that plays the same role. Anything in
[[failover-orchestration]] or [[dr-testing-and-gamedays]] that assumes readiness
checks needs rewriting against Region switch. Routing controls, zonal shift and
zonal autoshift are unaffected.

**MemoryDB Multi-Region is not a Canadian option and never has been.** This is
not a `ca-west-1` youth problem — MemoryDB Multi-Region has never listed *any*
Canadian region, including Montreal. If any deployment uses MemoryDB, the CA
deployment cannot use the same mirroring mechanism the EU and US deployments
can. See [[aws-elasticache-redis]].

---

## `ca-west-1` (Calgary) — the deep dive

This is the open thread [[CLAUDE]] names explicitly. Findings from
[[aws-eks]], [[aws-rds-postgres]], [[aws-aurora-global-database]],
[[aws-dynamodb]], [[aws-vpc-networking]], [[aws-iam]], [[aws-kms]],
[[aws-acm]], [[aws-alb-nlb]], [[aws-secrets-manager]] and
[[aws-ssm-parameter-store]] are collected here, plus new work.

### Region facts

`ca-west-1` launched **20 December 2023** with roughly 70 services, three
Availability Zones, on a stated ~CAD $4.3bn investment
([AWS News Blog](https://aws.amazon.com/blogs/aws/the-aws-canada-west-calgary-region-is-now-available/)).
It is the youngest region anywhere in this estate by seven years.

### The hypothesis that was wrong

The brief's stated worry was **AZ count** — that Calgary would have fewer AZs
than Montreal and break subnet-per-AZ templating. [[aws-vpc-networking]]
checked this properly and **it is false**: `ca-west-1` has three AZs, exactly
matching `ca-central-1`. A 3-AZ template mirrors cleanly.

The real AZ trap is in the *primary*: `ca-central-1`'s AZ IDs are
`cac1-az1`, `cac1-az2`, **`cac1-az4`** — there is no `cac1-az3` — while
`ca-west-1`'s are the contiguous `caw1-az1/2/3`. Any module that builds an AZ ID
by string-formatting an index blows up in Montreal, not Calgary. Full detail and
the working HCL are in [[aws-vpc-networking]].

### The gaps that are real

#### 1. Instance families — the big one

This is new work for this note. Parsed from the AWS Price List bulk API
(`AmazonEC2`, version `20260910195514`, Linux/shared-tenancy/on-demand SKUs),
comparing the instance families that actually have a price in each region:

**68 EC2 instance families exist in `ca-central-1` and do not exist in `ca-west-1`:**

```
c4, c5a, c5d, c5n, c6a, c6gd, c7gd, c7i-flex, c8gd, c8i, c8i-flex, c8id,
d2, d3, f2, g3, g4ad, g4dn, g5, g6, g6f, gr6, gr6f,
i3, i4g, i8g, i8ge, im4gn, inf1, is4gen,
m4, m5a, m5ad, m6a, m6idn, m6in, m7g, m7i, m7i-flex, m8gd, m8id,
p3, p4d, p5,
r4, r5a, r5ad, r5b, r5d, r5n, r6a, r6gd, r6idn, r6in, r7i, r8a, r8id,
t2, t3a,
u-3tb1, u-6tb1, u7i-6tb, x1, x1e, x2idn, x2iedn
```

Only one family goes the other way (`r6id` is in Calgary but not Montreal).

Read the shape of that list rather than the length:

- **No `m7i`, no `m7g`, no `m7i-flex`.** Calgary jumps `m6i`/`m6g` → `m8g`/`m8i`.
  If the Montreal node groups run 7th-generation general-purpose — an extremely
  common 2024–2025 choice — **there is no like-for-like instance in the standby.**
- **No AMD anywhere.** `m5a`, `m6a`, `c5a`, `c6a`, `r5a`, `r6a`, `r8a` all absent.
- **No GPU or accelerator instances at all.** `g4dn`, `g5`, `g6`, `p3`, `p4d`,
  `p5`, `inf1` — all absent. If any CA workload does inference, it cannot fail
  over.
- **No high-memory or `x`-family.** `x1`, `x1e`, `x2idn`, `x2iedn`, `u-*` absent.
- **No `t2`/`t3a`.** `t3` and `t4g` are present, so burstable is covered.

Verified prices for what *is* there (Linux, shared tenancy, on-demand,
`$/hour`):

| Instance | `ca-central-1` | `ca-west-1` |
|---|---|---|
| `m5.xlarge` | 0.214 | 0.214 |
| `m6i.xlarge` | 0.214 | 0.214 |
| `m6g.xlarge` | 0.1712 | 0.1712 |
| `m7i.xlarge` | 0.2247 | **absent** |
| `m7g.xlarge` | 0.1819 | **absent** |
| `m8g.xlarge` | 0.20012 | 0.20012 |
| `c7i.xlarge` | 0.1953 | 0.1953 |
| `c7g.xlarge` | 0.1581 | 0.1581 |
| `r7g.xlarge` | 0.2346 | 0.2346 |
| `t3.medium` | 0.0464 | 0.0464 |

**Where the family exists in both, the price is identical to the cent.** Calgary
is not a price premium — it is a catalogue restriction.

> [!danger] This is the highest-priority verification item in the whole vault
> If the `ca-central-1` EKS node groups, RDS instance classes or ElastiCache
> node types name a family in that 68-item list, the CA pair as designed
> **cannot be built**, and the discovery will happen at `terraform apply` or,
> far worse, at a failover that cannot scale.

#### 2. MemoryDB is absent

Not "Multi-Region is absent" — **the service is absent**. `ca-west-1` does not
appear in MemoryDB's own endpoint table. `ca-central-1` does
(`memory-db.ca-central-1.amazonaws.com`). If Montreal runs MemoryDB, there is no
Calgary equivalent and the workload has to move to ElastiCache (which *does*
have Global Datastore in Calgary) or be rebuilt in-region at failover. See
[[aws-elasticache-redis]].

#### 3. API Gateway quotas are a quarter

`ca-west-1` is in the reduced-quota tier: **throttle rate 2,500 rps, burst
1,250**, against `ca-central-1`'s 10,000 / 5,000. The rate is adjustable via
Service Quotas; the burst is **not**. For a warm standby that must absorb 100%
of Montreal's traffic within 15 minutes, a non-adjustable burst ceiling at 25%
of the primary's is a live RTO risk. **Raise the rate quota in Calgary now, as a
pre-provisioning task**, and make sure someone has modelled whether 1,250 burst
survives the thundering herd of a failover. See [[aws-api-gateway]].

#### 4. It is an opt-in region

`ca-west-1` requires `account:EnableRegion` from the management account.
Enablement is asynchronous, AWS says it "takes a few minutes for most accounts,
but can sometimes take several hours", is rate-limited to 6 in-flight per
account and 50 per Organization, and **has no Terraform resource**. STS v1
session tokens are rejected in opt-in regions. If Control Tower's Region deny
control or an equivalent SCP is in force, every principal is denied in Calgary
until someone edits the policy at the Organization level. Full treatment in
[[aws-iam]] — this is a *day-zero* blocker, not a day-two one, and it applies to
`eu-west-2` and `us-west-2` not at all.

#### 5. Smaller gaps worth knowing

- **Aurora `db.r8g` has no price in `ca-west-1`.** If the Montreal cluster is on
  Graviton4, the Global Database secondary cannot match its class. `db.r7g` and
  `db.r6g` are both present.
- **Aurora instances cost ~18% more in Calgary than Montreal** —
  `db.r6g.large` $0.286 vs $0.243, `db.r7g.large` $0.3040 vs $0.2582
  (I/O-Optimized is priced identically in both, at $0.3720 / $0.3950). This is
  the only place in the whole estate where the standby is materially more
  expensive than its primary. See [[cost-model]].
- **AWS Backup Audit Manager and the Jobs dashboard are not available.** Backups
  and cross-Region copy work; the *evidence* layer does not. If DR compliance
  reporting is a requirement, the CA pair cannot produce it from AWS Backup and
  needs a different control. See [[aws-backup]] and [[regulatory-drivers]].
- **No DynamoDB MRSC Region set for Canada.** Eventual consistency only across
  the CA pair, permanently. Fine for RPO 2h; fatal if anyone ever wants
  strongly-consistent multi-region reads. See [[aws-dynamodb]].
- **No EKS FIPS endpoint.** Irrelevant unless there is a FIPS obligation — but
  Canada is exactly where a FIPS obligation might exist. Worth a direct question.
- **No CloudWatch cross-region metric centralisation SKU.** Observability
  aggregation for the CA pair may need a different mechanism. See
  [[observability-multi-region]].
- **KMS crypto-op quota is 10,000/s** in both `ca-central-1` and `ca-west-1`, so
  no *delta* — but note that it is 10× lower than `us-east-1`/`us-west-2`/
  `eu-west-1` (100,000/s). See [[aws-kms]].
- **WAFv2 `REGIONAL` scope and the Route 53 health-checker managed prefix list
  are still unverified.** Both are named in [[aws-alb-nlb]] and [[aws-route53]]
  as CA-pair open items and both are ten-minute CLI checks.

### Verify before you commit

Four commands. Run them against the real accounts before the CA design is
signed off; each one closes an assumption this note could not close from public
documentation.

```bash
# 1. THE ONE THAT MATTERS. Do the families we actually run exist in Calgary?
#    Replace the Values list with the families in the ca-central-1 node groups.
aws ec2 describe-instance-type-offerings \
  --region ca-west-1 --location-type availability-zone \
  --filters Name=instance-type,Values='m7i.*','m7g.*','m6i.*','m8g.*','c7i.*','r7g.*' \
  --query 'InstanceTypeOfferings[].[InstanceType,Location]' --output table

# 2. Does the health-checker prefix list exist? Blocks the CA security-group module.
aws ec2 describe-managed-prefix-lists --region ca-west-1 \
  --filters Name=prefix-list-name,Values=com.amazonaws.ca-west-1.route53-healthchecks

# 3. Is regional WAF there? If not, the CA pair needs a CloudFront-fronted design.
aws wafv2 list-web-acls --scope REGIONAL --region ca-west-1

# 4. AZ IDs, in *your* account. The name↔ID mapping is per-account.
aws ec2 describe-availability-zones --region ca-central-1 \
  --query 'AvailabilityZones[].[ZoneName,ZoneId]' --output table
aws ec2 describe-availability-zones --region ca-west-1 \
  --query 'AvailabilityZones[].[ZoneName,ZoneId]' --output table
```

Plus one ten-minute experiment: **create a throwaway DynamoDB global table with
a `ca-central-1` / `ca-west-1` replica pair.** The docs only say global tables
are available wherever DynamoDB is; no Calgary-specific announcement exists.
Prove it rather than inherit it ([[aws-dynamodb]]).

---

## Availability Zones

Verified against the AWS AZ ID table via [[aws-vpc-networking]].

| Region | Role | AZ count | AZ IDs |
|---|---|---|---|
| `eu-west-1` | EU primary | 3 | `euw1-az1/2/3` |
| `eu-west-2` | EU standby | **4** | `euw2-az1/2/3/4` |
| `us-east-1` | US primary | **6** | `use1-az1` … `use1-az6` |
| `us-west-2` | US standby | **4** | `usw2-az1` … `usw2-az4` |
| `ca-central-1` | CA primary | 3 | `cac1-az1`, `cac1-az2`, **`cac1-az4`** |
| `ca-west-1` | CA standby | 3 | `caw1-az1/2/3` |

Three consequences:

1. **CA is fine.** 3 and 3. The subnet-per-AZ template mirrors.
2. **EU is fine, with slack.** London has one AZ more than Ireland. A standby
   bigger than its primary costs nothing extra if you only populate three of
   them — but `var.az_count` must be a variable, not a constant, or the template
   will either under-use London or try to find a fourth AZ in Ireland.
3. **US is the structural mismatch.** Six AZs in the primary, four in the
   standby. If `us-east-1` production genuinely spans five or six AZs, the
   standby **cannot be a 1:1 mirror** and the failover capacity plan has to fold
   six AZs of workload into four. Most estates run three and this is a
   non-issue — but it is a question for the team, not an assumption.

No available standby region fixes (3): no US region other than `us-east-1` has
six AZs. If `us-east-1` really is running six AZs wide, the answer is to narrow
the primary to four, not to widen the standby.

---

## Latency between the pairs

### The honest position on sources

**AWS does not publish an inter-region latency table.** The closest thing is the
per-account Infrastructure Performance view in the console (Network Manager),
which is not a citable public figure. Everything public is third-party
measurement. The figures below are from
[economize.cloud's AWS latency pages](https://www.economize.cloud/resources/aws/latency/),
which publish a single figure per pair but **do not publish their methodology,
measurement type or averaging window** — so treat them as *indicative order of
magnitude*, not as engineering inputs. A second source,
[latency.bluegoat.net](https://latency.bluegoat.net/), returns materially
different (higher) numbers for the same pairs, which is itself the argument for
not treating either as authoritative.

| Pair | Reported latency | Great-circle sanity check |
|---|---|---|
| `eu-west-1` → `eu-west-2` | **12.24 ms** | Dublin–London ~465 km. Plausible. |
| `us-east-1` → `us-west-2` | **68.23 ms** | N. Virginia–Oregon ~3,800 km. Plausible. |
| `ca-central-1` → `ca-west-1` | **48.09 ms** | Montreal–Calgary ~3,000 km. Plausible. |

If a real number is needed, **measure it**: stand up a `t4g.nano` in each region
of a pair and run a week of TCP round-trips. That is a half-day of work and it
produces a number you own. Recorded in [[#Open questions]].

### Why it matters — and why it matters less than you'd think

**RPO 2 hours is enormously loose relative to all three of these numbers.** That
is the single most important sentence in this section. At 12 ms or 68 ms, every
async replication mechanism in scope settles in seconds:

- **Aurora Global Database** targets typical cross-region replication lag well
  under a second and replicates at the storage layer, not through the engine, so
  the primary's write latency is unaffected by the distance. 68 ms of RTT
  consumes 0.001% of a 2-hour RPO budget.
- **DynamoDB Global Tables** propagate in the second-ish range under normal
  conditions. The `us-east-1` incident of October 2025 is the reminder that
  "normal conditions" is doing work in that sentence — during that event,
  customers with global tables *kept access to replicas in other regions* but
  saw **prolonged replication lag to and from the `us-east-1` replica**. Lag is
  a symptom of the incident, not of the distance.
- **ElastiCache Global Datastore** is async and AWS does not publish a lag SLA.
  Redis is a cache; if the standby's cache is cold or stale the workload should
  survive it. If it cannot, that is an application bug that the DR programme has
  usefully surfaced. See [[aws-elasticache-redis]].
- **S3 CRR** without Replication Time Control has **no SLA on replication
  latency at all**. With RTC, AWS commits to 99.99% of objects within 15
  minutes. 15 minutes against a 2-hour RPO means RTC is *not required* on RPO
  grounds — it is required, if at all, on RTO grounds, because an object that
  has not arrived when you cut over is an object the standby cannot serve. That
  distinction is worth real money; see [[cost-model]].

**Where latency does bite is the failover itself, not the steady state.** At
cutover you have a backlog to drain, and the drain rate is bandwidth-bound, not
latency-bound. And **none of these distances would support a synchronous design**
— which is fine, because the target posture is explicitly async
active/passive. If anyone proposes active/active later, 68 ms between
`us-east-1` and `us-west-2` is the number that kills it for any
read-modify-write path.

---

## Blast radius: how independent are these regions really?

This is the question a same-continent pair has to answer, and the honest answer
is **"independent enough for infrastructure failure, and not the thing you
should be worrying about."**

### The threat model that actually applies

AWS regional events are overwhelmingly **software and control-plane** events —
a bad deployment, a DNS race condition, a capacity-management bug — not
physical events. Geographic separation is close to irrelevant against that class
of failure; *regional isolation of the control plane* is what matters, and AWS
gives you that between any two regions, 400 km apart or 4,000.

The October 2025 `us-east-1` event is the canonical example: the root cause was
**a latent race condition in DynamoDB's automated DNS management**
([AWS post-event summary](https://aws.amazon.com/message/101925)), not anything
a second data centre in Virginia would have helped with. A `us-west-2` standby
was unaffected at the infrastructure level. Moving the US standby to Frankfurt
would have bought nothing extra and cost a great deal.

So: **for the EU and CA pairs, same-continent is sufficient.** Ireland and
London are in separate power grids, separate national jurisdictions post-Brexit
and separate AWS regional control planes. Montreal and Calgary are 3,000 km and
two time zones apart. There is no plausible correlated-failure story that takes
out both members of either pair and that a more distant pair would have
survived.

### The exception: `us-east-1`'s special status

`us-east-1` is not a region like the others, and the US pair therefore has a
correlation the other two pairs do not.

Per the AWS
[Fault Isolation Boundaries whitepaper](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/appendix-b---edge-network-global-service-guidance.html),
these global control planes are **hosted in `us-east-1`**:

| Service | Control plane | Data plane | What survives a control-plane impairment |
|---|---|---|---|
| **Route 53** | `us-east-1` | 200+ PoPs + every region | DNS resolution, health checks, and **routing changes driven by health-check state** |
| **CloudFront** | `us-east-1` | edge PoPs | content serving, **origin failover** |
| **ACM (for CloudFront)** | `us-east-1` | — | existing certs keep working, incl. auto-renewal |
| **AWS WAF (CloudFront scope)** | `us-east-1` | edge | configured web ACLs keep evaluating |
| **Shield Advanced** | `us-east-1` | — | existing protections keep protecting |
| **Global Accelerator** | **`us-west-2`** | anycast network | routing, health checks, traffic dials, endpoint weights |

Note the last row carefully: **Global Accelerator's control plane is in
`us-west-2`, not `us-east-1`.** For the US pair specifically, that means the
service you would reach for to shift traffic has its control plane co-located
with the standby. That is arguably a *point in the US pair's favour* — but it is
also its own correlation, and it is why [[failover-orchestration]] must not
depend on any CRUDL call at 3am.

**The design consequence is the same for all three pairs and it is not about
geography — it is about static stability.** The whitepaper is explicit: *"you
should not rely on the Route 53 control plane in your recovery path"*, and the
statically-stable pattern is to **drive failover from health-check state, not
from `ChangeResourceRecordSets`**. Concretely:

- Pre-create every DNS record, health check, ALB, Global Accelerator endpoint
  group, ACM certificate and Shield protection **before** the incident. Flipping
  a health check is a data-plane operation; creating a record is a
  `us-east-1` control-plane operation.
- **ARC routing controls** exist precisely to turn a failover into a data-plane
  operation: the cluster is five redundant regional endpoints, AWS guarantees at
  least three are reachable, and state converges in ≤15 s. That reliability is
  what the $2.50/hour is buying — see [[cost-model]].
- The failover runbook should contain **zero** create/update API calls against
  anything in the table above. If it contains one, it has a `us-east-1`
  dependency regardless of which pair is failing over. [[failover-orchestration]].

### Does `us-east-1`'s status argue against it as a *primary*?

It argues for a smaller thing than people usually claim. The global control
planes live there whether or not you run there. Hosting your US primary in
`us-east-1` adds one genuine correlation — a regional event in N. Virginia can
impair both your primary workload *and* your ability to make control-plane
changes to the global services you would use to escape it — and that is exactly
the correlation static stability neutralises. **Recommendation: keep
`us-east-1` as the US primary, and treat "no control-plane calls in the recovery
path" as a hard rule rather than a best practice.** Moving the primary is a far
larger programme than making the runbook statically stable, and it does not
remove the dependency.

---

## Cost delta per pair

Full model in [[cost-model]]. The headline, from the AWS Price List bulk API
(`20260910195514`), comparing **standby-region price to primary-region price for
the same resource**:

| Line item | EU (`eu-west-2` vs `eu-west-1`) | US (`us-west-2` vs `us-east-1`) | CA (`ca-west-1` vs `ca-central-1`) |
|---|---|---|---|
| EC2 `m6i.xlarge` | 0.222 vs 0.214 → **+3.7%** | 0.192 vs 0.192 → **0%** | 0.214 vs 0.214 → **0%** |
| EC2 `c7i.xlarge` | 0.2121 vs 0.19152 → **+10.7%** | 0.1785 vs 0.1785 → **0%** | 0.1953 vs 0.1953 → **0%** |
| ALB hourly | 0.02646 vs 0.0252 → **+5.0%** | 0.0225 vs 0.0225 → **0%** | 0.02475 vs 0.02475 → **0%** |
| ALB LCU-hour | 0.0084 vs 0.0080 → **+5.0%** | 0.0080 vs 0.0080 → **0%** | 0.0088 vs 0.0088 → **0%** |
| NAT Gateway hourly | 0.050 vs 0.048 → **+4.2%** | 0.045 vs 0.045 → **0%** | 0.050 vs 0.050 → **0%** |
| Interface VPC endpoint hourly | 0.011 vs 0.011 → **0%** | 0.010 vs 0.010 → **0%** | 0.011 vs 0.011 → **0%** |
| TGW attachment hourly | 0.06 vs 0.05 → **+20%** | 0.05 vs 0.05 → **0%** | 0.06 vs 0.06 → **0%** |
| RDS PG `db.m6g.large` | 0.184 vs 0.176 → **+4.5%** | 0.159 vs 0.159 → **0%** | 0.175 vs 0.175 → **0%** |
| **Aurora PG `db.r6g.large`** | 0.264 vs 0.286 → **−7.7%** | 0.225 vs 0.225 → **0%** | **0.286 vs 0.243 → +17.7%** |
| **Aurora storage GB-mo** | 0.225 vs 0.248 → **−9.3%** | 0.225 vs 0.225 → **0%** | 0.248 vs 0.248 → **0%** |
| **Aurora replicated write I/O** (per million) | 0.20 vs 0.22 → **−9.1%** | 0.20 vs 0.20 → **0%** | 0.22 vs 0.22 → **0%** |
| DynamoDB rWCU-hour | 0.000772 vs 0.000735 → **+5.0%** | 0.00065 vs 0.00065 → **0%** | 0.000715 vs 0.000715 → **0%** |
| S3 Standard GB-mo (first tier) | 0.024 vs 0.023 → **+4.3%** | 0.023 vs 0.023 → **0%** | 0.025 vs 0.025 → **0%** |
| EKS control plane, KMS key, Secrets Manager secret, ECR GB-mo | **0%** (flat everywhere) | **0%** | **0%** |
| Inter-region data transfer out | **$0.02/GB** | **$0.02/GB** | **$0.02/GB** |

Three things fall out of that table:

1. **The US pair is free of regional price delta.** Oregon and N. Virginia are
   priced identically on every line checked. Whatever the US standby costs, it
   is exactly what the same estate costs in the primary.
2. **The EU pair costs ~4–5% more in London for most things**, up to +20% on
   Transit Gateway attachments — **but London is cheaper than Ireland for
   Aurora**, on instances (−7.7%), storage (−9.3%) and replicated write I/O
   (−9.1%). If the EU deployment is Aurora-heavy, the standby may be cheaper
   per unit than the primary. Do not assume the mirror is a 1:1 duplicate in
   either direction.
3. **The CA pair is price-identical to Montreal on everything except Aurora,
   where Calgary is 18% more expensive.** If the CA deployment is Aurora-based,
   that is the one place a standby costs more than its primary anywhere in this
   estate.

**Inter-region data transfer is $0.02/GB in both directions for all three
pairs.** There is no cheap pair and no expensive pair on egress; distance does
not price into this.

---

## Recommendation

| Primary | **Recommended standby** | Runner-up | Deciding factor |
|---|---|---|---|
| `eu-west-1` Ireland | **`eu-west-2` London — confirm** | `eu-central-1` Frankfurt | London has complete service parity, 4 AZs, ~12 ms, and is the cheapest lift. Frankfurt's *only* advantage is that it keeps the data inside the EU/EEA, which is a **legal** question, not a technical one. If [[data-residency]] concludes the UK adequacy decision is too fragile to build on, **switch to Frankfurt** — the technical cost of doing so is close to zero and it should be decided *before* any HCL is written, not after. |
| `us-east-1` N. Virginia | **`us-west-2` Oregon — confirm** | `us-east-2` Ohio | Oregon is priced identically to N. Virginia, has near-total service parity (only `hpc7g` missing), is the secondary home of several AWS global services (Global Accelerator's control plane, ARC's mandatory API region) and is maximally distant. Ohio would be cheaper on inter-region transfer and lower latency, but shares a power/weather geography with N. Virginia and adds nothing else. **The AZ mismatch (6 → 4) is the open item, not the region choice.** |
| `ca-central-1` Montreal | **`ca-west-1` Calgary — confirm, conditionally** | *(no acceptable alternative inside Canada)* | Calgary clears the parity bar on everything that would have been expensive to work around — Aurora Global Database, ElastiCache Global Datastore, EKS, API Gateway, Global Accelerator, zonal shift, AWS Backup cross-Region copy, 3 AZs. Its gaps are **instance families, MemoryDB, API Gateway quotas and Backup Audit Manager**, and every one of those is cheaper to design around than to solve by leaving the country. |

### If `ca-west-1` fails the verification commands

It has not failed the *documented* parity check. It may still fail
[[#Verify before you commit]] step 1 — if the Montreal node groups run `m7i` or
`m7g`, there is no like-for-like instance in Calgary. Options, in the order they
should be considered:

| Option | What it costs | Verdict |
|---|---|---|
| **Re-target the CA deployment onto families Calgary has** (`m6i`/`m6g` → `m8g`/`m8i`, or `c7i`/`c7g`/`r7g`) | Re-benchmarking and a rolling node-group replacement in the *primary*. Days of engineering, no architectural change, no residency question. | **Recommended.** `m8g` is newer and cheaper per vCPU than `m7i`; this is likely an upgrade you would want anyway. |
| Run the CA standby on a different family from the primary | Free. Kubernetes does not care, as long as the architecture matches (both Graviton or both x86) and the capacity plan accounts for the performance delta. | **Acceptable fallback.** Requires that nothing in the workload is pinned to a CPU generation. Mixing `arm64` and `x86_64` across the pair is *not* acceptable — it doubles your container image matrix. |
| CA pair becomes multi-AZ only; no regional DR for Canada | Free, and honest. Canada gets AZ-level resilience and backup/restore, not 15-minute regional RTO. | **Viable if the CA deployment's risk profile genuinely differs.** Put it to the business explicitly rather than silently under-delivering. |
| Standby the CA deployment in a **US** region | Solves every technical gap instantly. | > [!danger] Hard blocker, almost certainly<br>Canadian customer data replicated to a US region is subject to US legal process, and PIPEDA-adjacent commitments and customer contracts very often prohibit it. The distinction that matters is **data residency** (where the bytes sit) versus **data sovereignty** (whose courts can compel access) — a US standby fails both. **This must be a legal decision, never an engineering one.** Route it through [[data-residency]] and [[regulatory-drivers]] before anyone models it. |
| Swap the CA pair's direction (`ca-west-1` primary, `ca-central-1` standby) | A full migration of the live Canadian deployment. | **Reject.** It moves the constrained catalogue into the *primary*, which is strictly worse, and it is the largest possible amount of work for a negative outcome. |

---

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| EU standby region | `eu-west-2` London — parity, 4 AZs, 12 ms | `eu-central-1` Frankfurt — stays in the EU/EEA | **London**, *conditional on [[data-residency]] signing off on UK adequacy*. Decide before writing HCL. |
| US standby region | `us-west-2` Oregon — max separation, identical pricing | `us-east-2` Ohio — lower latency, cheaper transfer | **Oregon.** Ohio's proximity is not a benefit here. |
| CA standby region | `ca-west-1` Calgary — stays in Canada, has real gaps | A US region — no gaps, fails residency | **Calgary.** Design around the gaps. |
| `us-east-1` as US primary | Keep | Move to a region without global control planes | **Keep**, and make the runbook statically stable instead. |
| Instance families in the CA pair | Match primary exactly | Let the standby differ | **Match**, by moving the *primary* onto a family Calgary has. Same family both sides is one less thing to reason about at 3am. |
| MemoryDB in the CA deployment | Keep it; rebuild at failover | Migrate CA to ElastiCache (Global Datastore works in Calgary) | **Migrate**, if MemoryDB is in use at all. Rebuilding a MemoryDB cluster is not a 15-minute operation. |
| ARC readiness checks | — | — | **Not a decision.** Closed to new customers. Use ARC Region switch. |

---

## Open questions

1. **Which EC2 instance families do the three primaries actually run today?**
   Everything about the CA pair turns on this one answer. Nothing else in this
   note is as urgent.
2. **Does `us-east-1` production span more than four AZs?** If yes, the US
   standby is not a mirror, and the right fix is narrowing the primary.
3. **Is MemoryDB in use anywhere?** If yes, the CA deployment needs a different
   Redis story from the EU and US ones.
4. **Is there a FIPS requirement on any deployment?** `ca-west-1` and `eu-west-2`
   have no FIPS EKS endpoint.
5. **Is the UK still covered by an EU adequacy decision the business is willing
   to build on?** This is the only thing that could break the EU pair, and it is
   a legal answer, not a technical one. [[data-residency]].
6. **Is DR compliance evidence required?** AWS Backup Audit Manager does not
   exist in `ca-west-1`.
7. **Is regional WAF needed in the CA pair, and does WAFv2 `REGIONAL` exist in
   `ca-west-1`?** Unverified. A "no" forces a CloudFront-fronted design.
8. **Who holds `account:EnableRegion`, and is `ca-west-1` already enabled?**
   Also: does a Control Tower Region deny control currently block it?
9. **Does anyone want a measured latency figure?** No AWS-published number
   exists. A week of `t4g.nano` pings per pair would produce one you own.
10. **RTO definition.** Carried forward from [[CLAUDE]] and still unanswered:
    15 minutes from *incident start* or from *decision to fail over*? The region
    choice does not change, but the runbook does. [[rpo-rto-analysis]].

---

## Sources

- [Amazon EKS endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/eks.html) — EKS and `eks-auth` endpoints in `ca-west-1`; absence of a FIPS EKS endpoint there.
- [Amazon API Gateway endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/apigateway.html) — **the quota finding**: `ca-west-1` throttle rate 2,500 rps / burst 1,250 vs 10,000 / 5,000 elsewhere; burst is non-adjustable. Confirms control- and data-plane endpoints incl. FIPS.
- [MemoryDB — Choosing Regions and Availability Zones](https://docs.aws.amazon.com/memorydb/latest/devguide/regionsandazs.html) — **the MemoryDB finding**: the supported-Regions endpoint table lists `ca-central-1` and omits `ca-west-1`.
- [MemoryDB Multi-Region](https://docs.aws.amazon.com/memorydb/latest/devguide/multi-region.html) — supported Regions are US East/West, EU Ireland/Frankfurt/London, AP Tokyo/Sydney/Mumbai/Seoul/Singapore. No Canadian region.
- [ElastiCache global datastore — prerequisites and limitations](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Redis-Global-Datastores-Getting-Started.html) — **Canada Central and Canada West (Calgary) both listed**; supported node families (M5, M6g, M7g, M8g, R5, R6g, R6gd, R7g, R8g, C7gn, C8gn, large and above); no cross-region autofailover; no IPv6.
- [Supported Regions and DB engines for Aurora global databases](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.Aurora_Fea_Regions_DB-eng.Feature.GlobalDatabase.html) — Canada West (Calgary) listed for Aurora PostgreSQL 11–18, same version floors as every other region.
- [AWS Backup feature availability](https://docs.aws.amazon.com/aws-backup/latest/devguide/backup-feature-availability.html) — the per-Region table: Canada West (Calgary) is opt-in, **has** cross-Region copy / cross-account management / restore testing / backup search, and **lacks** Backup Audit Manager and the Jobs dashboard; footnote 1 excludes Calgary from cross-Region copy into a logically air-gapped vault.
- [ARC readiness check availability change](https://docs.aws.amazon.com/r53recovery/latest/dg/arc-readiness-availability-change.html) — **closed to new customers**; Region switch is the recommended replacement; routing controls, zonal shift and zonal autoshift unaffected.
- [AWS Region availability for zonal shift (ARC)](https://docs.aws.amazon.com/r53recovery/latest/dg/introduction-regions-zonal.html) — `ca-west-1` listed with FIPS endpoints; all six in-scope regions present.
- [AWS Region availability for routing control (ARC)](https://docs.aws.amazon.com/r53recovery/latest/dg/introduction-regions-routing.html) — routing control is global, API calls must specify `--region us-west-2`; cluster is five redundant regional endpoints, ≥3 always reachable, state converges within 5 s average / 15 s max.
- [AWS Fault Isolation Boundaries — Appendix B](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/appendix-b---edge-network-global-service-guidance.html) — **the `us-east-1` control-plane table**: Route 53, CloudFront, ACM-for-CloudFront, WAF and Shield Advanced control planes in `us-east-1`; **Global Accelerator's control plane in `us-west-2`**; the static-stability rule for recovery paths.
- [Summary of the Amazon DynamoDB Service Disruption in N. Virginia](https://aws.amazon.com/message/101925) — the October 2025 event; latent race condition in DynamoDB's automated DNS management; global-table replicas in other regions stayed accessible but with prolonged replication lag to/from `us-east-1`.
- [Amazon EventBridge cross-Region support now expands to more Regions](https://aws.amazon.com/about-aws/whats-new/2021/11/amazon-eventbridge-cross-region-expands) — cross-Region event bus targets extended beyond the original `us-east-1`/`us-west-2`/`eu-west-1` to all commercial regions.
- [AWS Global Accelerator now supports endpoints in Canada West (Calgary)](https://aws.amazon.com/about-aws/whats-new/2024/04/aws-global-accelerator-endpoints-calgary-region) — 25 April 2024.
- [Enable or disable AWS Regions in your account](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-regions.html) — `ca-west-1` is opt-in; `eu-west-2` and `us-west-2` are default-enabled; enablement is asynchronous, "a few minutes … sometimes several hours"; 6-per-account / 50-per-Organization in-flight limits.
- [The AWS Canada West (Calgary) Region is now available](https://aws.amazon.com/blogs/aws/the-aws-canada-west-calgary-region-is-now-available/) — launch 20 Dec 2023, ~70 services, three AZs.
- [AWS KMS endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/kms.html) — per-region symmetric crypto-op quotas: `us-east-1`/`us-west-2`/`eu-west-1` 100,000/s, `eu-west-2` 20,000/s, `ca-central-1` and `ca-west-1` 10,000/s.
- **AWS Price List bulk API**, offer versions `20260910195514` (`AmazonEC2`, `AmazonRDS`) and current (`AmazonVPC`, `AWSELB`, `AmazonDynamoDB`, `AmazonS3`, `AmazonElastiCache`, `AmazonCloudWatch`, `AWSDataTransfer`), fetched 17 Sep 2026 — **the instance-family finding and every price in this note**. Public, no credentials required: `https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/<OfferCode>/current/region_index.json`.
- [economize.cloud AWS latency pages](https://www.economize.cloud/resources/aws/latency/) — the three inter-region latency figures, **methodology not published**, treat as indicative.
- [latency.bluegoat.net](https://latency.bluegoat.net/) — a second latency matrix that disagrees materially with the above; cited as the reason not to trust either.
- Internal: [[aws-vpc-networking]] (AZ counts and IDs, NAT pricing), [[aws-eks]] (Calgary endpoint/AZ/instance-family caution), [[aws-rds-postgres]], [[aws-aurora-global-database]], [[aws-dynamodb]] (MRSC region sets), [[aws-iam]] (opt-in region mechanics), [[aws-kms]], [[aws-acm]], [[aws-alb-nlb]], [[aws-secrets-manager]], [[aws-ssm-parameter-store]], [[aws-route53]].
