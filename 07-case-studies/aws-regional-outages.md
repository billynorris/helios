---
title: AWS Regional Outages — What Actually Happened and What It Taught
tags: [case-study, outage, postmortem, us-east-1, resilience]
status: researched
updated: 2026-09-17
---

# AWS Regional Outages — What Actually Happened and What It Taught

> **Scope.** This note is the factual record: the large AWS Region-scoped
> failures that have public AWS post-event summaries (PES), what each one
> actually broke, and — the part that matters most to this programme — the
> repeatedly-demonstrated fact that **being multi-region did not, by itself,
> protect people**, because of dependencies on `us-east-1` that most teams did
> not know they had.
>
> Everything below is sourced. Where a claim comes from vendor marketing or a
> secondary blog rather than a first-party post-mortem, it is labelled.

## TL;DR

- **AWS publishes real post-event summaries and they are the best material in this vault.** Six major ones are directly relevant: [2017 S3](https://aws.amazon.com/message/41926/), [2020 Kinesis](https://aws.amazon.com/message/11201/), [Dec 2021 network](https://aws.amazon.com/message/12721/), [Jun 2023 Lambda](https://aws.amazon.com/message/061323/), [Jul 2024 Kinesis](https://aws.amazon.com/message/073024/), [Oct 2025 DynamoDB DNS](https://aws.amazon.com/message/101925/). **All six were `us-east-1`.**
- **Not one of these was a "Region fell into the sea" event.** Every single one was a *control-plane or internal-dependency* failure inside a Region that remained physically intact. That has a direct design consequence: the standby must be reachable and promotable using **data-plane operations only**. This is the single most important transferable lesson.
- **`us-east-1` is not just a Region, it is the control plane for the `aws` partition.** AWS states this itself in the [Fault Isolation Boundaries whitepaper](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/global-services.html): IAM, Organizations, Account Management, Route 53 Public DNS, Route 53 Private DNS, CloudFront, ACM-for-CloudFront, WAF-for-CloudFront and Shield Advanced all have their **control plane in `us-east-1`**. This is why "we're in eu-west-1, we were fine" was false for a lot of people in October 2025.
- **This directly threatens the US pair as currently scoped.** The US pair's primary is `us-east-1`. That means the US pair is the one pair where *the Region you are failing away from also hosts the control plane of the services you would use to fail away*. See [[#What this means for the Helios US pair]].
- **The thing that will bite:** in October 2025, Amazon Redshift customers **in every AWS Region** could not use IAM user credentials to run queries, because Redshift called an IAM endpoint in `us-east-1`. You do not get to audit that. Your only defence is static stability — needing no control-plane call to recover.

---

## The catalogue

### February 28, 2017 — S3 `us-east-1`

**Source:** [Summary of the Amazon S3 Service Disruption in the Northern Virginia (US-EAST-1) Region](https://aws.amazon.com/message/41926/) (AWS, first-party).

An authorised S3 engineer running an established playbook to debug the S3
billing system "entered incorrectly" one input to a capacity-removal command
and removed a much larger set of servers than intended. Two S3 subsystems lost
enough capacity that they required a **full restart**:

| Subsystem | Role |
|---|---|
| **Index** | "manages the metadata and location information of all S3 objects" — serves GET, LIST, PUT, DELETE |
| **Placement** | "manages allocation of new storage" for PUT |

Timeline: command at 09:37 PST; index serving GET/LIST/DELETE again at 12:26;
index fully recovered 13:18; placement recovered and S3 normal at 13:54. Roughly
**four hours**.

**The bit everyone quotes and nobody acts on:** the Service Health Dashboard
itself could not be updated, because it ran on S3 in `us-east-1`. AWS committed
to running the SHD "across multiple AWS regions". *Your own status page,
runbook wiki, and on-call tooling have the same problem — see
[[#Cross-cutting lessons]].*

AWS's committed changes were: slow down capacity-removal tooling and add
minimum-capacity safeguards; audit other operational tools for the same checks;
break the index subsystem into smaller **cells** to cut blast radius; restructure
the SHD to be multi-region. The cell work is the intellectual ancestor of the
cell-based architecture argument in [[#Slack — the counter-example]].

**What it taught:** blast radius is a design parameter, and human operational
tooling is part of the availability story. It did *not* teach anyone much about
multi-region, because other Regions genuinely were unaffected.

---

### November 25, 2020 — Kinesis Data Streams `us-east-1`

**Source:** [Summary of the Amazon Kinesis Event in the Northern Virginia (US-EAST-1) Region](https://aws.amazon.com/message/11201/) (AWS, first-party).

A small capacity addition to the Kinesis front-end fleet, started 02:44 PST and
finished 03:47 PST, pushed every server in the fleet past an **operating-system
thread limit**. AWS: "the new capacity had caused all of the servers in the fleet
to exceed the maximum number of threads allowed by an operating system
configuration."

Front-end servers build a **shard-map** cache — "membership details and shard
ownership for the back-end clusters". Thread exhaustion meant the maps could not
be built, so "front-end servers were ending up with useless shard-maps that left
them unable to route requests to back-end clusters." First alarms 05:15 PST; root
cause confirmed 09:39; first servers back 10:07; **full recovery 22:23 PST** —
roughly **17 hours**.

The cascade is the point:

| Service | How it failed |
|---|---|
| **Cognito** | Buffered failed Kinesis calls, exhausted memory, blocked webservers. Fixed by a code deploy at 14:18 PST. |
| **CloudWatch** | Metric and log processing failed; **alarms went to `INSUFFICIENT_DATA`**. Recovered 17:47–22:31 PST. |
| **Lambda** | Memory contention as function metrics backed up locally. Mitigated 10:36 PST. |
| **EventBridge, ECS, EKS** | Cluster provisioning and scaling delayed until 16:15 PST. |
| **EC2 Auto Scaling** | Policies that consume CloudWatch metrics stalled until 17:47 PST. |

**The two lessons that survive:**

1. **Your alarms are a dependency.** CloudWatch alarms flipping to
   `INSUFFICIENT_DATA` means an automated, health-check-driven failover may
   never fire — or may fire spuriously. Anything in [[failover-orchestration]]
   that keys off CloudWatch in the primary Region inherits this.
2. **AWS could not tell customers what was happening.** AWS: "the tool we use to
   post these updates itself uses Cognito, which was impacted by this event."
   They fell back to the Personal Health Dashboard.

---

### December 2021 — three separate `us-east-1`/US events in one month

**Source for the first:** [Summary of the AWS Service Event in the Northern Virginia (US-EAST-1) Region](https://aws.amazon.com/message/12721/) (AWS, first-party).

**December 7 (the big one).** An automated scaling activity on an internal AWS
service triggered "a large surge of connection activity that overwhelmed the
networking devices" between AWS's *internal* network — which hosts monitoring,
internal DNS, authorisation and parts of the EC2 control plane — and the main
AWS network. A **latent defect in internal client back-off logic** prevented the
clients from retreating: "This code path has been in production for many years
but the automated scaling activity triggered a previously unobserved behavior."
Classic congestion collapse.

Timeline: surge 07:30 PST; EC2 API errors 07:33; internal DNS traffic rerouted
09:28; congestion materially improved 13:34; devices recovered 14:22. Six to
seven hours.

The list of what broke is the list of things a failover plan touches:

| Impaired | Detail |
|---|---|
| **Route 53 control plane** | "impaired from 7:30 AM PST until 2:30 PM PST preventing customers from making changes" — **you could not change a DNS record for seven hours.** |
| **STS** | Elevated latency with OIDC providers; recovered 16:28 PST. |
| **AWS Console** | Login failures until 14:22 PST. |
| **Support Contact Center** | Could not open support cases 07:33–14:25 PST. |
| **Control planes** | EC2, RDS, EMR, WorkSpaces, ELB, Fargate/ECS/EKS all elevated error rates for create/manage operations. |

> **Read that Route 53 line again.** If your entire failover plan is
> "call `ChangeResourceRecordSets`", December 7 2021 is the day that plan did
> not work, for seven hours, in the Region that is the US pair's primary. This
> is the empirical basis for the control-plane warning in [[aws-route53]].

**December 15.** A separate, ~1-hour event affecting `us-west-1` and `us-west-2`,
attributed on the AWS status dashboard to network congestion between the AWS
backbone and a subset of ISPs, triggered by AWS traffic engineering. No PES was
published. ([ThousandEyes analysis](https://www.thousandeyes.com/blog/aws-outage-analysis-december-15-2021) — third-party.) **Note for this project: `us-west-2` is the US pair's standby. Standby Regions fail too.**

**December 22.** Loss of power in a single data centre within a single
Availability Zone (`USE1-AZ4`) in `us-east-1`, affecting EC2 instances in that
data centre. No PES. ([DCD coverage](https://www.datacenterdynamics.com/en/news/aws-has-another-east-coast-cloud-outage/) — third-party press.)

**What the month taught:** three unrelated failure mechanisms in 15 days, in
three different scopes (regional internal network, inter-region networking,
single-AZ power). Any model that treats "a Region outage" as one event type with
one probability is wrong.

---

### June 13, 2023 — Lambda `us-east-1`

**Source:** [Summary of the AWS Lambda Service Event in the Northern Virginia (US-EAST-1) Region](https://aws.amazon.com/message/061323/) (AWS, first-party).

A latent defect surfaced when the Lambda Frontend fleet crossed an unprecedented
capacity threshold during ordinary traffic growth. The defect caused execution
environments to be "successfully allocated for incoming requests, but never fully
utilized by the Lambda Frontend" — i.e. capacity was consumed without serving
anything.

Frontend scaling started 10:01 PDT; degradation began 11:49; defect identified
12:26; synchronous invocations fully recovered 13:45; normal operations 15:37
PDT.

Collateral: **STS** (11:49–14:10), **AWS Management Console** (11:48–14:02),
**EventBridge** (delivery latencies "of up to 801 seconds"), **EKS** new cluster
provisioning, **Amazon Connect**, and **AWS Support Center** (11:49–14:38).

**What it taught:** the recurring shape — *no customer changed anything, organic
growth crossed a threshold, a latent defect fired.* You cannot get ahead of this
with change freezes. Also: **STS and the Console are collateral damage in almost
every `us-east-1` event.** If your break-glass depends on console login or the
global STS endpoint, your break-glass depends on `us-east-1`.

---

### July 30, 2024 — Kinesis Data Streams `us-east-1`

**Source:** [Summary of the Amazon Kinesis Data Streams Service Event in the Northern Virginia (US-EAST-1) Region](https://aws.amazon.com/message/073024/) (AWS, first-party).

The newer, cell-based Kinesis architecture — built partly in response to 2020 —
failed in a new way. "One of the internal workloads on the impacted cell had a
very large number of very low-throughput shards which caused the cell management
system to behave incorrectly." A routine deployment at 09:09 PDT in one AZ
tripped it; the system "incorrectly determined that the healthy hosts were
unhealthy and began redistributing shards", overwhelming the component that
provisions secure connections.

Degradation 14:45 PDT; load-shedding mitigation 16:25; improvement 17:39; vast
majority normal 19:21; full recovery 21:37 PDT. Nearly seven hours.

Affected: CloudWatch Logs, Amazon Data Firehose, the S3 event publishing
framework, ECS, Lambda, Redshift, Glue.

**What it taught:** cellularisation reduces blast radius but introduces
*cell-management* as a new shared failure mode. Fixing a failure class does not
remove the class of "shared coordination component".

---

### October 19–20, 2025 — DynamoDB DNS `us-east-1`

**Source:** [Summary of the Amazon DynamoDB Service Disruption in the Northern Virginia (US-EAST-1) Region](https://aws.amazon.com/message/101925/) (AWS, first-party). **This is the most instructive post-event summary AWS has ever published and it should be read in full by anyone on this programme.**

**Root cause.** DynamoDB's automated DNS management splits into a **DNS Planner**
(watches load balancer health, produces plans) and a **DNS Enactor** (applies
plans, running independently in three AZs for redundancy). One Enactor stalled
while applying an older plan; a second Enactor raced ahead applying a newer plan
and then ran cleanup. The stale-timestamp check failed to stop the old plan
overwriting the new one, and cleanup then deleted the record entirely:

> "As this plan was deleted, all IP addresses for the regional endpoint were
> immediately removed. Additionally, because the active plan was deleted, the
> system was left in an inconsistent state that prevented subsequent plan
> updates from being applied."

`dynamodb.us-east-1.amazonaws.com` resolved to nothing. Not slow — *absent*.

**The cascade is the real lesson.** Three distinct, sequential failures, each
caused by the recovery of the previous one:

1. **DynamoDB DNS** (23:48 PDT Oct 19 → 02:25 PDT Oct 20). Manual DNS restore.
2. **EC2 launches.** DropletWorkflow Manager (DWFM) could not complete droplet
   lease checks against DynamoDB. When DynamoDB came back, DWFM went into
   **congestive collapse** — re-establishing leases with hundreds of thousands
   of droplets, timing out faster than it could converge. Engineers throttled
   work and restarted DWFM hosts at 04:14; leases done 05:28. Network Manager
   then had a huge backlog propagating network config to newly launched
   instances — normal at 10:36.
3. **NLB.** Health checks ran against newly launched EC2 instances whose network
   state had not propagated, producing alternating pass/fail, which increased
   load on the health-check subsystem and **triggered automatic AZ DNS failover
   that removed healthy capacity from service.** 05:30–14:09.

EC2 fully recovered 13:50; event concluded 14:20 PDT. **~15 hours end to end.**

**The cross-region blast radius — the part that matters here:**

> "Customers using their root credential, and customers using identity
> federation configured to use `signin.aws.amazon.com` experienced errors when
> trying to log into the AWS Management Console **in regions outside of the
> N. Virginia (us-east-1) Region**."

> "Amazon Redshift customers **in all AWS Regions** were unable to use IAM user
> credentials for executing queries" — because Redshift used an IAM API endpoint
> in `us-east-1`.

Also hit: Lambda invocations and SQS processing, ECS/EKS/Fargate task launches,
Amazon Connect (agents could not sign in), **STS API errors until 09:59**, the
**IAM Management Console**, Redshift (some clusters impaired until 04:05 on
Oct 21), and the **AWS Support Console**.

**AWS's committed changes:** disable the DynamoDB DNS Planner/Enactor automation
worldwide and fix the race before re-enabling; add **velocity controls to NLB**
so health-check failures cannot remove capacity too fast; new test suites for
DWFM recovery; queue-size-aware throttling in EC2.

---

## The hidden-`us-east-1` argument — the most important material in this note

The widely-made argument after October 2025 was not "AWS failed". It was
**"multi-region customers failed anyway"**. The mechanism is documented by AWS
itself, in the [AWS Fault Isolation Boundaries whitepaper, "Global services"](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/global-services.html). This is first-party AWS
documentation, not commentary, and it is the citation to use with stakeholders.

AWS's own framing:

> "Global AWS services still follow the conventional AWS design pattern of
> separating the control plane and data plane in order to achieve static
> stability. The significant difference for most global services is that their
> control plane is hosted in a *single* AWS Region, while their data plane is
> globally distributed."

### Where the control planes actually live (`aws` partition)

**Partitional services** — control plane in one Region, data plane isolated per Region:

| Service | Control plane Region |
|---|---|
| AWS IAM | `us-east-1` |
| AWS Organizations | `us-east-1` |
| AWS Account Management | `us-east-1` |
| **Route 53 Private DNS** | `us-east-1` |
| Amazon Application Recovery Controller (ARC) | **`us-west-2`** |
| AWS Network Manager | `us-west-2` |

**Edge-network global services:**

| Service | Control plane Region |
|---|---|
| Route 53 Public DNS | `us-east-1` |
| Amazon CloudFront | `us-east-1` |
| AWS WAF (and WAF Classic) for CloudFront | `us-east-1` |
| **ACM for CloudFront** | `us-east-1` |
| AWS Shield Advanced | `us-east-1` |
| AWS Global Accelerator | **`us-west-2`** |

AWS's recommendation, twice, verbatim:

> "Do not rely on the control planes of partitional services in your recovery
> path. Instead, rely on the data plane operations of these services."

> "Do not rely on the control plane of edge network services in your recovery
> path."

### The sneaky ones — "global single-Region operations"

These are the ones nobody audits, because they are *operations*, not services.

- **Anything that creates a DNS record creates it via the Route 53 control plane in `us-east-1`.** AWS lists: API Gateway REST/HTTP APIs, ELB load balancers, PrivateLink VPC endpoints, Lambda function URLs, ElastiCache, OpenSearch, CloudFront, MemoryDB, Neptune, DAX, Global Accelerator, ECS with DNS service discovery (via Cloud Map), and **the EKS Kubernetes control plane**. Creating an ALB in `eu-west-2` during a failover takes a dependency on `us-east-1`. (Exception AWS calls out explicitly: VPC DNS for EC2 instance hostnames like `ip-10-0-1-5.eu-west-2.compute.internal` is per-Region and does **not** depend on the Route 53 control plane.)
- **S3 bucket creation.** "All calls to the `CreateBucket` and `DeleteBucket` APIs depend on `us-east-1` … to ensure name uniqueness, even though the API call is directed at the specific Region." Plus ~24 `PutBucket*`/`DeleteBucket*` configuration APIs listed by AWS as having a `us-east-1` dependency — including `PutBucketPolicy`, `PutBucketReplication`, `PutBucketEncryption`, `PutBucketVersioning`, `PutBucketNotification`.
- **S3 Multi-Region Access Points** control plane is in `us-west-2` only, and itself depends on Global Accelerator in `us-west-2`, Route 53 in `us-east-1`, and ACM in each Region.
- **Edge-optimised API Gateway endpoints** need the CloudFront control plane in `us-east-1` to create.
- **STS.** "STS usage from the AWS SDK and CLI defaults to `us-east-1`." The global endpoint `sts.amazonaws.com` is served from a single Region with no automatic failover. Cross-reference [[aws-iam]].
- **SAML sign-in.** Works per-Region (`https://eu-west-2.signin.aws.amazon.com/saml`) but only if your role trust policies and IdP are configured for it. Most aren't. AWS explicitly recommends pre-provisioning **break-glass users**.

### AWS's own list of anti-patterns

Verbatim from the same whitepaper — this is the checklist to run the failover
runbook against:

- "Making changes to Route 53 records, like updating an A record's value or changing a weighted record set's weights, to perform failover."
- "Creating or updating IAM resources, including IAM roles and policies, during a failover. This typically isn't intentional, but might be a result of an untested failover plan."
- "Making changes to AGA traffic dial weights to manually perform a Regional failover."
- "Updating a CloudFront distribution's origin configuration to fail away from an impaired origin."
- "Provisioning disaster recovery (DR) resources, like ELBs and RDS instances during a failure event, that depend on creating DNS records in Route 53."

**Four of those five are what a naive Helios failover runbook would do.**

### One more trap: ARC's own control plane

AWS says ARC's control plane is in `us-west-2`, and separately advises:

> "Following the best practices for Route 53 Application Recovery Controller
> (ARC), you should hardcode or bookmark your five Regional cluster endpoints.
> During a failure event, you might not be able to access some API operations,
> including Route 53 ARC API operations that are not hosted on the extremely
> reliable data plane cluster."

So even the tool designed to solve this problem has a control plane you must not
depend on at 3am. The five regional cluster endpoints go in the runbook as
literal strings. See [[failover-orchestration]].

---

## What customers wrote afterwards

### Ably — multi-region worked, and they were still degraded

**Source:** Paddy Byers (co-founder & CTO), [*AWS us-east-1 outage: How Ably's multi-region architecture held up*](https://ably.com/blog/multi-region-resilience-aws-outage), 22 October 2025. First-party.

Ably runs a globally distributed system across multiple AWS Regions, each
scaling independently, with `us-east-1` normally carrying the most traffic.
During the October 2025 event:

- **Data plane held.** "error rates were negligibly low and unchanged
  throughout." Existing `us-east-1` infrastructure kept working.
- **Control plane did not.** Ancillary AWS services broke Ably's ability to
  **add capacity** in `us-east-1` as traffic rose. That is the constraint that
  forced their hand, not errors.
- **They shed the Region by DNS.** Around 12:00 UTC they routed *new*
  connections to `us-east-2`; existing connections were left alone.
- **Cost of the shift:** "Any additional round trip latency was limited to
  12ms … well below our 40–50ms global median", and across US monitoring
  locations the "measured latency difference averaged 3ms". Net: "zero service
  disruption".

**Why this is the single best case study for Helios.** Ably's failure mode was
*not* "the Region died". It was "the Region stopped letting me scale". A warm
standby sized for yesterday's traffic, in a Region whose control plane is
degraded, is a warm standby that cannot grow. Ably survived because the standby
was already *running and already taking traffic* — which is the argument for
sizing the Helios standby closer to hot than to pilot-light. See
[[rpo-rto-analysis]].

### HashiCorp — a documented single-API-call failover

**Source:** [*How HashiCorp made cross-Region switchover seamless with Amazon Application Recovery Controller*](https://aws.amazon.com/blogs/architecture/how-hashicorp-made-cross-region-switchover-seamless-with-amazon-application-recovery-controller), AWS Architecture Blog, 25 July 2025. **Label: this is AWS's own blog describing a customer using an AWS product — treat the enthusiasm as marketing, the mechanics as real.**

HashiCorp Cloud Platform's SRE team moved from manual regional failover — "DNS
record manipulation by specialized operators", single-Region control-plane
dependencies — to an ARC-based switchover: **a single API call, traffic shifting
within seconds, full propagation in about two minutes**, and, crucially, "disaster
recovery executable by regular on-call rotation" rather than by named experts.
Services discover the active Region via **TXT records** rather than by having
their endpoint mutated.

The article does **not** publish concrete RTO/RPO figures — it says only that
they "planned to not just reach these targets but make substantial improvements".
Do not quote numbers from it; there aren't any.

**Transferable:** the ~2-minute propagation figure is consistent with the DNS
arithmetic in [[aws-route53]], and "any on-call engineer can do it" is the right
acceptance criterion for a 15-minute RTO.

### Slack — the counter-example

**Source:** Cooper Bethea, [*Slack's Migration to a Cellular Architecture*](https://slack.engineering/slacks-migration-to-a-cellular-architecture/), Slack Engineering, 22 August 2023. First-party.

Slack runs a global multi-regional *edge* network, but keeps most core compute
in **multiple AZs within `us-east-1`**. After a 30 June 2021 incident in which a
faulty network link between AZs caused about six hours of degradation across two
incidents, Slack's response was **not** to go multi-region. It was to silo
services within an AZ and build the ability to drain an AZ: Envoy load balancers
with dynamic weights via RTDS, 1% granularity, graceful drains. "Propagation
through the control plane is on the order of seconds; Envoy load balancers will
apply new weights immediately."

Their stated root-cause insight is worth carrying: **"detecting failure in
distributed systems is a hard problem"** — gray failures gave different
components inconsistent views of what was healthy.

**Honest caveat:** the article does not argue against multi-region and does not
mention rejecting it. Do not claim Slack "rejected multi-region". What it *does*
demonstrate is that a very large, very visible company concluded that
**fine-grained, fast, well-practised traffic draining inside one Region** was the
highest-value resilience investment available to them. That is a real data point
against "multi-region or nothing", and it belongs in
[[lessons-and-antipatterns]].

---

## Cross-cutting lessons

1. **Every major event was a control-plane/dependency failure, not a physical one.** Design for "the Region is up but won't let you change anything", not "the Region is gone". That is a *harder* scenario, because it also breaks the tools you'd use to leave.
2. **Recovery is the second outage.** In Oct 2025, DWFM's congestive collapse and NLB's health-check storm were both caused by *recovering* from the prior failure. Thundering-herd on failback is a real risk for Helios — see [[failback]].
3. **Automated failover can hurt you.** NLB's automatic AZ DNS failover removed healthy capacity. AWS's own DR whitepaper says automatic failover "should be used with caution … If you fail over when you don't need to (false alarm), then you incur those losses", and recommends **manually initiated, fully automated** failover — the push of a button. This matches the Helios RTO definition question flagged in `CLAUDE.md`.
4. **Your observability is inside the blast radius.** CloudWatch alarms went `INSUFFICIENT_DATA` in 2020. AWS could not update its own status page in 2017 and 2020. [[observability-multi-region]] must assume the primary's telemetry is gone.
5. **Your identity path is inside the blast radius.** STS and the Console were collateral in 2021, 2023 and 2025; in 2025 federated console login failed *in other Regions*. Break-glass credentials, regional STS endpoints, and regional SAML endpoints are prerequisites, not polish. [[aws-iam]].
6. **Support is inside the blast radius.** Case creation was impaired in 2021, 2023 and 2025. Do not plan to escalate to AWS as step one.
7. **`us-east-1` is over-represented for reasons that are not bad luck.** It is the oldest, largest Region and hosts the partition's global control planes. Six of the six PESs above are `us-east-1`.
8. **Standby Regions fail too.** `us-west-2` had its own event on 15 Dec 2021 and hosts the control planes for ARC and Global Accelerator.

---

## What this means for the Helios US pair

The US pair is `us-east-1` → `us-west-2`. That pairing has two properties nobody
should discover during an incident:

- **The primary hosts the control planes you would use to escape it.** Route 53 public and private DNS, IAM, Organizations, CloudFront, ACM-for-CloudFront. Everything in the EU and CA pairs' failover paths *also* depends on `us-east-1` for those control planes — but for the US pair, the dependency and the disaster are the same Region.
- **The standby hosts ARC and Global Accelerator control planes.** That is mildly *good* (ARC's control plane is not co-located with the US primary) but it means an ARC-based plan for the EU and CA pairs has a `us-west-2` dependency.

**Concrete consequences for this programme:**

| Requirement | Why |
|---|---|
| Every record the standby needs is **pre-created**, with failover routing policies already in place. Failover flips a health check, never writes a record. | Route 53 control plane, Dec 2021: seven hours unavailable. |
| Every IAM role, policy, OIDC provider and trust relationship exists in advance. | AWS anti-pattern list; IAM control plane is in `us-east-1`. |
| SDKs and CLI configured for **regional STS endpoints**; role trust policies accept multi-Region SAML; break-glass IAM users exist. | STS impaired in 2021/2023/2025; federated console login failed globally in 2025. |
| No ALB, NLB, RDS instance, API Gateway endpoint, EKS cluster or S3 bucket is created at failover time. | All create Route 53 records via the `us-east-1` control plane. Also the direct reason the 15-minute RTO forbids cold provisioning. |
| Standby capacity sized to take the load **without** Auto Scaling completing first. | Auto Scaling is a control plane action; Ably's real constraint was inability to scale. |
| ARC cluster endpoints hard-coded in the runbook. | AWS's own guidance; ARC's control plane may be unavailable. |
| Runbooks, credentials and comms live outside the primary Region. | 2017 and 2020 SHD failures; Support Center impaired three times. |

## Still worth chasing

- A first-party post-mortem from a **regulated EU/UK financial institution** describing a real AWS regional failover. Nothing found; the sector publishes very little.
- AWS PESs for the Dec 15 and Dec 22 2021 events — **none appear to exist**; only status-dashboard text and third-party analysis.
- Whether AWS has published anything since Oct 2025 on regionalising the IAM/`signin.aws.amazon.com` dependency. Not found as of this note.

## Sources

**First-party AWS post-event summaries**
- [Summary of the Amazon S3 Service Disruption (Feb 28 2017)](https://aws.amazon.com/message/41926/) — typo'd capacity-removal command, index/placement restart, SHD hosted on the thing that broke.
- [Summary of the Amazon Kinesis Event (Nov 25 2020)](https://aws.amazon.com/message/11201/) — OS thread limit, shard-map cache, CloudWatch `INSUFFICIENT_DATA`, Cognito-dependent status tooling.
- [Summary of the AWS Service Event in US-EAST-1 (Dec 7 2021)](https://aws.amazon.com/message/12721/) — internal network congestion collapse; **Route 53 control plane unavailable for 7 hours**.
- [Summary of the AWS Lambda Service Event (Jun 13 2023)](https://aws.amazon.com/message/061323/) — latent defect at a capacity threshold; STS/Console/Support collateral.
- [Summary of the Amazon Kinesis Data Streams Service Event (Jul 30 2024)](https://aws.amazon.com/message/073024/) — cell management misjudged host health after a routine deploy.
- [Summary of the Amazon DynamoDB Service Disruption (Oct 19–20 2025)](https://aws.amazon.com/message/101925/) — DNS Planner/Enactor race; DWFM congestive collapse; NLB health-check storm; **cross-Region IAM/Redshift/console impact**.

**First-party AWS guidance**
- [AWS Fault Isolation Boundaries — Global services](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/global-services.html) — the authoritative list of which control planes live in which Region, plus AWS's own anti-pattern list. The single most citable document for this note.
- [Disaster Recovery of Workloads on AWS — Disaster recovery options in the cloud](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html) — the four-strategy framework; "For maximum resiliency, you should use only data plane operations as part of your failover operation."

**First-party customer accounts**
- Paddy Byers, Ably, [*AWS us-east-1 outage: How Ably's multi-region architecture held up*](https://ably.com/blog/multi-region-resilience-aws-outage) (22 Oct 2025) — data plane held, control plane blocked scaling, DNS shed to `us-east-2`, 3–12ms latency cost.
- Cooper Bethea, Slack, [*Slack's Migration to a Cellular Architecture*](https://slack.engineering/slacks-migration-to-a-cellular-architecture/) (22 Aug 2023) — AZ-level cells in `us-east-1`, Envoy/RTDS drains, gray-failure detection.

**Vendor blog describing a customer (labelled: AWS marketing surface)**
- [How HashiCorp made cross-Region switchover seamless with ARC](https://aws.amazon.com/blogs/architecture/how-hashicorp-made-cross-region-switchover-seamless-with-amazon-application-recovery-controller) (25 Jul 2025) — single-API-call switchover, ~2 min propagation, TXT-record active-Region discovery. **No RTO/RPO numbers published.**

**Third-party analysis (no AWS PES exists for these events)**
- [ThousandEyes — AWS Outage Analysis: December 15, 2021](https://www.thousandeyes.com/blog/aws-outage-analysis-december-15-2021) — `us-west-1`/`us-west-2` backbone/ISP congestion.
- [DCD — Data center power loss brings down AWS services](https://www.datacenterdynamics.com/en/news/aws-has-another-east-coast-cloud-outage/) — Dec 22 2021, `USE1-AZ4` power loss.
