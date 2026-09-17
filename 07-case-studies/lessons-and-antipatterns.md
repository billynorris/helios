---
title: Lessons and Anti-Patterns — Including the Case Against This Project
tags: [case-study, lessons, anti-patterns, counter-argument, regulatory, dora, fca]
status: researched
updated: 2026-09-17
---

# Lessons and Anti-Patterns — Including the Case Against This Project

> **What this note is for.** Two jobs. First, the distilled lessons from real,
> cited incidents — each one traceable to a source, not to folklore. Second, and
> more importantly, **the honest counter-argument**: the serious case that
> multi-region is not worth it. Someone will eventually ask "is this actually
> worth doing?" and the vault should contain the real answer rather than only
> the case for the programme.
>
> Factual record of the AWS events referenced here lives in
> [[aws-regional-outages]].

## TL;DR

- **The canonical framework is AWS's own four DR strategies** — backup and restore, pilot light, warm standby, multi-site active/active. Helios sits at **warm standby**, and AWS's description of warm standby matches the brief almost word for word. See [[#The canonical framework]].
- **AWS itself says most workloads don't need this.** From the DR whitepaper: "For a disaster event based on disruption or loss of one physical data center for a well-architected, highly available workload, you may only require a backup and restore approach." The multi-region case has to be made on *Region-scoped* failure or *regulation*, not on generic resilience.
- **The strongest counter-argument is not "it costs too much".** It is Corey Quinn's, and it is specific: *"you cannot have a multi-region failover strategy on AWS that features AWS's us-east-1 region."* The US pair's primary is `us-east-1`. That argument has to be answered, not deflected.
- **The second-strongest counter-argument is Salesforce NA14:** they had a tested-on-paper standby, failed over to it successfully, and the standby then corrupted its own database and **replicated the corruption back to the primary**. Replication is a two-way blast-radius amplifier. A standby is a new failure mode, not only a mitigation.
- **The thing that settles the argument for this company is probably not engineering at all.** DORA applies in the EU from 17 Jan 2025, and the UK operational resilience regime (FCA PS21/3 / PRA SS1/21) became fully binding on 31 Mar 2025. If the company is in scope, "is it worth it" is not the question being asked. See [[#Regulatory drivers]].

---

## The canonical framework

**Source:** [AWS, *Disaster Recovery of Workloads on AWS: Recovery in the Cloud* — "Disaster recovery options in the cloud"](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html). First-party. This is the document the whole Helios programme sits inside and everyone on it should have read this page.

Four strategies, "ranging from the low cost and low complexity of making backups
to more complex strategies using multiple active Regions."

| Strategy | What exists in the recovery Region | Trade-off |
|---|---|---|
| **Backup and restore** | Data backups (and IaC). Nothing running. | Cheapest. You must redeploy infrastructure, config and code at failover. Without IaC, "it may be complex to restore workloads in the recovery Region, which will lead to increased recovery times and possibly exceed your RTO." |
| **Pilot light** | Data replicated continuously; core infrastructure provisioned; **application servers switched off**. | "your core infrastructure is always available and you always have the option to quickly provision a full scale production environment by switching on and scaling out your application servers." |
| **Warm standby** | "a scaled down, but fully functional, copy of your production environment in another Region." Always on. | AWS's own distinction: "pilot light cannot process requests without additional action taken first, whereas warm standby can handle traffic (at reduced capacity levels) immediately." |
| **Multi-site active/active** (and its active/passive twin, **hot standby**) | Full capacity, both Regions. | "the most complex and costly approach … but it can reduce your recovery time to near zero." |

> **AWS deliberately does not publish hard RPO/RTO numbers per strategy in the
> body text.** The figures people quote ("RTO < 1 hour", "RPO seconds") come
> from Figure 6, a diagram. Do not put invented numbers in this vault. The
> numbers that matter are the Helios ones — RPO 2h / RTO 15m — and the analysis
> belongs in [[rpo-rto-analysis]].

**Four statements from this whitepaper that should be quoted at stakeholders:**

1. **The data-plane rule.** "For maximum resiliency, you should use only data plane operations as part of your failover operation. This is because the data planes typically have higher availability design goals than the control planes." Everything in [[aws-regional-outages]] is an illustration of this sentence.
2. **When multi-region is justified.** "For a disaster event based on disruption or loss of one physical data center for a well-architected, highly available workload, you may only require a backup and restore approach to disaster recovery. If your definition of a disaster goes beyond the disruption or loss of a physical data center to that of a Region **or if you are subject to regulatory requirements that require it**, then you should consider Pilot Light, Warm Standby, or Multi-Site Active/Active." (emphasis added)
3. **Against automatic failover.** "Automatically initiated failover based on health checks or alarms should be used with caution. Even using the best practices discussed here, recovery time and recovery point will be greater than zero … If you fail over when you don't need to (false alarm), then you incur those losses. Manually initiated failover is therefore often used. In this case, you should still automate the steps for failover, so that the manual initiation is like the push of a button."
4. **The warm-standby cost/resilience dial.** "Because Auto Scaling is a control plane activity, taking a dependency on it will lower the resiliency of your overall recovery strategy. It is a trade-off. You can choose to provision sufficient capacity such that the recovery Region can handle the full production load as deployed. This statically stable configuration is called *hot standby*. Or you may choose to provision fewer resources which will cost less, but take a dependency on Auto Scaling."

**Statement 4 is the single most important cost decision in the programme.** A
15-minute RTO plus the Ably evidence (control plane degraded → cannot scale;
see [[aws-regional-outages]]) argues that Helios's "warm standby" needs to be
closer to hot than the word "warm" implies. That is where the money goes. It
belongs in [[cost-model]] as an explicit slider, not a default.

One more AWS line that is uncomfortable for an active/passive programme:

> "Most customers find that if they are going to stand up a full environment in
> the second Region, it makes sense to use it active/active. Alternatively, if
> you do not want to use both Regions to handle user traffic, then Warm Standby
> offers a more economical and operationally less complex approach."

i.e. AWS's position is roughly: *if you're going to pay for hot, use it.* The
Helios brief has already chosen active/passive; this is the argument that will
be raised against it, and the answer is the one in
[[research-brief]] — the three deployments are data-isolated per geography, so
"use both regions" would mean splitting a residency boundary, not just splitting
traffic. See [[data-residency]].

---

## The lessons, each traced to a source

### 1. Every real AWS regional failure was a control-plane failure, not a physical one

Six public AWS post-event summaries, six `us-east-1` events, zero "the Region
was destroyed". Design for *"the Region is up but frozen"*, which is harder,
because the tools you would use to leave are the ones that are frozen.
→ [[aws-regional-outages]].

### 2. Do not put a control-plane call in the failover path

AWS's own anti-pattern list, verbatim from the [Fault Isolation Boundaries whitepaper](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/global-services.html):

- "Making changes to Route 53 records … to perform failover."
- "Creating or updating IAM resources, including IAM roles and policies, during a failover. This typically isn't intentional, but might be a result of an untested failover plan."
- "Making changes to AGA traffic dial weights to manually perform a Regional failover."
- "Updating a CloudFront distribution's origin configuration to fail away from an impaired origin."
- "Provisioning disaster recovery (DR) resources, like ELBs and RDS instances during a failure event, that depend on creating DNS records in Route 53."

Empirical backing: the Route 53 control plane was "impaired from 7:30 AM PST
until 2:30 PM PST preventing customers from making changes" on 7 December 2021
([AWS PES](https://aws.amazon.com/message/12721/)).

**Anti-pattern:** a runbook whose step 1 is `aws route53 change-resource-record-sets`.
**Pattern:** every record pre-created with a failover routing policy; failover
flips a health check (ideally an ARC routing control, a data-plane operation).
→ [[aws-route53]], [[failover-orchestration]].

### 3. Your standby can corrupt your primary — Salesforce NA14, May 2016

**Source:** *Salesforce Takes a Dive*, Availability Digest (Sombers Associates / W. H. Highleyman), July 2016 — [PDF](https://availabilitydigest.com/public_articles/1107/salesforce.pdf). Third-party analysis drawing on Salesforce's own "RCM for NA14 Disruption of Service" document and contemporaneous ZDNet/InformationWeek reporting. Corroborated by [The Register](https://www.theregister.com/2016/05/13/salesforcecom_crash_caused_data_loss/).

The sequence, which is the best cautionary tale in this vault:

1. 9 May 2016, 17:46 PDT: a **redundant** intelligent circuit breaker failed in the Washington DC (WAS) data centre. The backup breaker "could not confirm the state of the problem breaker, and this led to the redundant breaker not closing to activate the backup feed." *Redundancy that cannot determine whether it should engage is not redundancy.*
2. Salesforce did the right thing: **site switch to Chicago (CHI), completed 19:39.** Service restored. So far, a textbook failover.
3. 10 May, 05:41: performance degradation at CHI; 06:31, a database cluster failure. A **latent storage-array firmware bug** — exposed only by the backlog of traffic that had built up during the first disruption — slowed writes enough to cause database timeouts and file discrepancies.
4. **Those discrepancies replicated from CHI back to WAS, corrupting it too.** "Salesforce was now in a position that it could not use the NA14 instance in either data center."
5. Recovery was restore-from-backup, with permanent loss of everything written since the switchover. Salesforce's status page: "data written to the NA14 instance between 9:53 UTC and 14:53 on May 10, 2016 could not be restored." Full service was not restored until **15 May — almost a week.**

The Availability Digest's conclusion, verbatim:

> "The outage should have lasted no more than the two hours required to bring up
> the CHI data center."

> "you can't count on the success of a failover unless you have thoroughly
> tested it. This includes testing the backup system **under full load**. This
> clearly was not done by Salesforce; otherwise, they would have found the
> latent firmware bug."

**Three lessons for Helios, all direct:**
- **Bidirectional replication is a bidirectional blast radius.** Anything that replicates — Aurora Global Database, DynamoDB Global Tables, S3 CRR — propagates logical corruption as faithfully as it propagates good data. This is exactly why [[aws-backup]] and point-in-time recovery are *not* made redundant by replication. AWS says the same thing in the DR whitepaper: continuous replication "may not protect against disaster events such as data corruption or malicious attack".
- **Test the standby under production load, not smoke-test load.** The bug only appeared under backlog volume. A DR gameday that sends 5% of traffic to the standby proves almost nothing. → [[dr-testing-and-gamedays]].
- **Failing over is the start of the incident, not the end.** The standby inherits a traffic backlog that the primary never had to absorb cold.

### 4. Recovery is the second outage

In October 2025, two of the three cascading failures were caused by *recovering*
from the previous one: EC2's DropletWorkflow Manager went into **congestive
collapse** re-establishing hundreds of thousands of leases after DynamoDB came
back, and NLB's health-check subsystem was overwhelmed by newly-launched
instances whose network state had not propagated — which then **triggered
automatic AZ DNS failover that removed healthy capacity**
([AWS PES](https://aws.amazon.com/message/101925/)). AWS's remediation included
"velocity controls" to limit how fast capacity can be removed, and queue-size-
aware throttling.

**For Helios:** failback is where the thundering herd lives. A standby that has
been serving for six hours has six hours of divergence, a cold cache, and a
queue backlog to replay. → [[failback]], [[messaging-in-flight-data-loss]].

### 5. Automated failover is a liability as often as an asset

NLB's *automatic* failover removed healthy capacity in Oct 2025. AWS's DR
whitepaper explicitly advises caution and recommends manual initiation of a
fully-automated procedure. Slack's own root-cause note from the June 2021 AZ
incident: **"detecting failure in distributed systems is a hard problem"** —
gray failures gave different components inconsistent views of health
([Slack Engineering](https://slack.engineering/slacks-migration-to-a-cellular-architecture/)).

**For Helios:** a 15-minute RTO measured from *decision to fail over* is
compatible with a human in the loop. Measured from *incident start*, it almost
certainly is not — which is the RTO-definition question already flagged in
`CLAUDE.md` and it needs answering before [[failover-orchestration]] is
designed.

### 6. Your observability, identity and comms are inside the blast radius

- CloudWatch alarms went to `INSUFFICIENT_DATA` in Nov 2020 ([PES](https://aws.amazon.com/message/11201/)).
- AWS could not update its own status dashboard in 2017 (hosted on S3) or 2020 ("the tool we use to post these updates itself uses Cognito, which was impacted by this event").
- STS and the Console were collateral damage in Dec 2021, Jun 2023 and Oct 2025; in Oct 2025, federated console login via `signin.aws.amazon.com` failed **in Regions other than `us-east-1`**.
- AWS Support case creation was impaired in Dec 2021, Jun 2023 and Oct 2025.

→ [[observability-multi-region]], [[aws-iam]], [[security-posture-of-the-standby]].

### 7. Five backups, zero restores — GitLab, 31 January 2017

**Source:** GitLab's own public post-mortem, [*Postmortem of database outage of January 31*](https://about.gitlab.com/blog/postmortem-of-database-outage-of-january-31/) and the live [incident writeup](https://about.gitlab.com/blog/gitlab-dot-com-database-incident). First-party, unusually candid, written in public in a Google Doc as it happened.

An engineer, debugging replication, ran `rm -rf` against the wrong host and
removed the primary's data directory. Then, per GitLab's own account, **out of
five backup/replication mechanisms, none was usable**: `pg_dump` to S3 was
silently failing (version mismatch between the 9.2 client and the 9.6 server) and
the failure notifications were being silently dropped by DMARC; the secondary's
contents had just been deleted; disk snapshots were not enabled on the database
servers; the staging LVM snapshot was six hours old; WAL segments had been
purged. They were saved only by an ad-hoc snapshot an engineer had happened to
take six hours earlier, and still lost roughly six hours of data — "roughly
5,000 projects, 5,000 comments and 700 new user accounts".

**For Helios:** this is not an AWS story and it is not a multi-region story. It
is in this note because it is the cleanest public evidence for a single claim:
**an untested recovery mechanism should be assumed not to work.** Applied here:
if the standby has never actually served production traffic, its RTO is unknown,
not 15 minutes. → [[dr-testing-and-gamedays]].

### 8. The standby Region fails too

`us-west-2` — the US pair's standby — had its own event on 15 December 2021
([ThousandEyes](https://www.thousandeyes.com/blog/aws-outage-analysis-december-15-2021), third-party; AWS published no PES). It also hosts the control planes for
Amazon Application Recovery Controller and Global Accelerator
([AWS Fault Isolation Boundaries](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/global-services.html)).

**Anti-pattern:** treating the standby as infinitely available because it is
idle. Its patch level, quotas, AMI ages and certificate expiries all drift, and
nothing tells you, because nothing is using it.

### 9. Warm standby only helps if it can absorb the load *without* scaling

Ably's October 2025 account is explicit: their `us-east-1` data plane never
broke. What broke was the **control plane**, which stopped them adding capacity
as traffic rose. They shifted new connections to `us-east-2` by DNS, at a
measured cost of 3–12 ms
([Paddy Byers, Ably](https://ably.com/blog/multi-region-resilience-aws-outage)).

**For Helios:** the failure mode to design against is not "the standby is
missing", it is "the standby exists but cannot grow at the moment you need it
to". That pushes towards pre-provisioned capacity (AWS's *hot standby* end of
the dial) and away from "Auto Scaling will sort it out".

### 10. Cellular / AZ-level work may buy more resilience per pound than a second Region

Slack's response to a six-hour, gray-failure AZ incident in June 2021 was to
build **AZ siloing and fast, fine-grained traffic draining** inside `us-east-1`
— Envoy with dynamic weights over RTDS, 1% granularity, graceful drains,
"propagation through the control plane is on the order of seconds"
([Cooper Bethea, Slack Engineering, Aug 2023](https://slack.engineering/slacks-migration-to-a-cellular-architecture/)).

**Be honest about what this does and does not prove.** Slack does not argue
against multi-region and does not say they rejected it. What the post
demonstrates is that a company of Slack's size, with a very public uptime
obligation, chose to spend its resilience budget on intra-Region cellularisation.
AWS reached the same conclusion internally — the 2017 S3 remediation included
"refactoring the index subsystem into smaller cells to reduce blast radius"
([AWS PES](https://aws.amazon.com/message/41926/)).

---

## The counter-argument, presented fairly

This section exists because the vault should be able to survive the question
"is this worth it?" being asked in bad faith by someone holding the budget.

### C1. The `us-east-1` argument — the strongest one

**Source:** Corey Quinn, [*Lessons in Trust from us-east-1*](https://www.lastweekinaws.com/blog/lessons-in-trust-from-us-east-1/), Last Week in AWS, 15 December 2021. Opinion, clearly labelled as such, from an AWS-focused consultant.

> "you cannot have a multi-region failover strategy on AWS that features AWS's
> us-east-1 region"

His reasoning: too many services single-track through `us-east-1`, so a
control-plane failure there is total. He also argues the financial incentive is
not neutral — "data from the internet into AWS is free; moving data between
availability zones and regions starts at 2 cents per gigabyte" — and concludes
"AWS is going to milk customers like cows to achieve significant reliability."

**Is he right?** Partly, and the part he is right about is exactly the Helios US
pair. AWS's own [Fault Isolation Boundaries](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/global-services.html)
confirms the factual premise: IAM, Organizations, Account Management, Route 53
public **and** private DNS, CloudFront, ACM-for-CloudFront, WAF-for-CloudFront
and Shield Advanced all have their control plane in `us-east-1`.

**The honest rebuttal** is that the premise supports a narrower conclusion than
the slogan. AWS's guidance is that the *data planes* of these services are
regionally isolated and remain available; the failure is confined to CRUDL
operations. A failover plan that touches **zero control-plane operations** is
not defeated by a `us-east-1` control-plane outage — and the October 2025 event
is partial evidence for that, since Ably's data plane worked throughout. But
that plan is significantly harder to build than the one most teams write, and
Quinn's real point stands: **a failover plan that has not been audited for
control-plane calls is a plan that does not work when it matters.**

**Where this leaves the programme:** it is an argument for *how* to build the US
pair (pre-provision everything, ARC routing controls, regional STS, break-glass
users, hard-coded endpoints), and an argument for putting the **US pair last**,
not for cancelling it. It is also a genuine argument for re-examining whether
`us-east-1` should remain the US *primary* — reversing the pair so `us-west-2`
is primary is a real option and should be considered in
[[region-pair-selection]].

### C2. The cost argument

AWS's own framing of the ladder is that each step up costs more. Warm standby
means a "scaled down, but fully functional, copy of your production environment"
running continuously, and the 15-minute RTO pushes that copy towards full size.
Realistically the standing bill is compute + storage + replication traffic +
duplicated managed-service baselines, plus per-GB inter-Region data transfer on
everything replicated.

**This vault should not quote a multiplier it has not computed.** The figures
circulating online ("warm standby costs 60–70% of active/active", "multi-region
doubles your bill") come from secondary blogs with no methodology and should not
be cited. → [[cost-model]] must build this from the actual estate and verified
AWS price-list figures, and [[cost-levers]] must identify where RPO 2h (loose)
lets you buy cheaper replication than RTO 15m (tight) lets you buy capacity.

**The strongest honest form of the cost argument:** the money is not the
problem; the *recurring* money is. A second Region is a permanent ~2× on
infrastructure headcount attention — every change, every migration, every
upgrade, forever — and that cost is paid in engineering time on every single
future project, not once.

### C3. The complexity-causes-outages argument

The general form: the failure multi-region guards against (a Region-scoped AWS
failure, historically a handful of multi-hour events over nine years) may be
rarer than the failures the added complexity itself causes — split-brain,
replication lag surprises, config drift between Regions, botched failovers,
corrupted standbys.

**Real evidence for it in this vault:**
- **Salesforce NA14** is the canonical instance: the failover mechanism turned a two-hour outage into a six-day one and destroyed data.
- **AWS's own remediation history** shows each layer of resilience machinery adding a new failure mode — Kinesis was re-architected into cells after 2020, and in July 2024 the **cell management system** itself misjudged host health and caused a seven-hour event ([AWS PES](https://aws.amazon.com/message/073024/)).
- **Slack** chose intra-Region cellularisation over a second Region.

**Real evidence against it:** Ably rode out the largest AWS event on record with
"zero service disruption" *because* they were multi-region
([Ably](https://ably.com/blog/multi-region-resilience-aws-outage)). Authress
reports the same — DNS-based dynamic routing between a primary and a failover
Region, custom health checks rather than Route 53 defaults, and deliberate
avoidance of AWS control-plane dependencies (Sergio De Simone, [*How Authress Designed for Resilience and Survived a Major AWS Outage*](https://www.infoq.com/news/2025/12/infrastructure-resilience-aws/), InfoQ, 28 Dec 2025 — third-party reporting of a first-party account). Authress's CTO's stated philosophy is
notable and cuts *with* the complexity argument even while defending
multi-region: keep the infrastructure "simple and small", repeat patterns rather
than abstracting them, minimise change frequency.

**The synthesis, which is the defensible position:** complexity is the enemy,
and the way to have multi-region without paying the complexity tax in outages is
to make the standby **as identical as possible** to the primary and to make
failover **as few actions as possible**. That is an argument about *how*, and it
is exactly what the prerequisites-first strategy in [[research-brief]] already
does — mirroring one boring building block at a time, with the same Terraform,
rather than building a bespoke DR system.

### C4. "Multi-AZ is enough" — and where it stops being enough

The honest version: AWS's own whitepaper says so, for the loss of one data
centre. Multi-AZ is dramatically cheaper, is already in place, and handles the
most common physical failure (the Dec 22 2021 `USE1-AZ4` power loss was exactly
this).

**Where it stops:** every single one of the six AWS post-event summaries in
[[aws-regional-outages]] was *Region-scoped*. Multi-AZ would have helped with
none of them. That is the fact that decides this argument, and it should be the
headline slide: **multi-AZ protects against the failures AWS has mostly stopped
having; multi-region protects against the failures AWS keeps having.**

### C5. What the counter-argument does not survive

If the company is in scope for DORA or the UK operational resilience regime,
C1–C4 are interesting but not decisive. See below.

---

## Regulatory drivers

Where regulation applies, the "is it worth it" conversation changes shape: the
question becomes "can you evidence that you can stay within your stated impact
tolerance", and a single-Region deployment with a multi-day rebuild time usually
cannot.

### EU — DORA (Regulation (EU) 2022/2554)

Applies from **17 January 2025**. Relevant to the **EU pair (`eu-west-1` →
`eu-west-2`)** and, note, to the Ireland→London pair specifically because
London is outside the EU — which is a [[data-residency]] question as well as a
resilience one.

- **Article 12 — redundant ICT capacities.** Financial entities other than microenterprises "shall maintain redundant ICT capacities equipped with resources, capabilities and functions that are adequate to ensure business needs."
- **Article 12 — geographical separation.** The explicit secondary-site requirement (drafted for central securities depositories, but widely used as the benchmark) is that the secondary processing site must be "at a geographical distance from the primary processing site to ensure that it bears a distinct risk profile and to prevent it from being affected by the event which has affected the primary site", must provide "continuity of critical or important functions identically to the primary site", and must be "immediately accessible to the financial entity's staff".
- **Article 12 — testing.** Backup, restoration and recovery procedures must be **periodically tested**.
- **Switchover testing.** Testing plans must include "scenarios of cyber-attacks and switchovers between the primary ICT infrastructure and the redundant capacity, backups and redundant facilities."
- **Concentration risk and exit plans.** The oversight framework is explicitly aimed at "systemic and concentration risks arising from the financial sector's reliance on a limited number of ICT providers", and firms need documented, credible, tested exit plans for critical ICT third-party providers.

Sources: [DORA Article 12 text](https://www.digital-operational-resilience-act.com/Article_12.html); [EIOPA DORA overview](https://www.eiopa.europa.eu/digital-operational-resilience-act-dora_en).

**Two things worth flagging honestly:** (a) the "distinct risk profile" phrasing
is the clause that makes `eu-west-1` → `eu-west-2` defensible and would make
`eu-west-1` → `eu-west-3` equally so; (b) **a second AWS Region does not by
itself address ICT *concentration* risk under DORA**, because the provider is
still AWS. That is the regulatory argument for the multi-cloud "lifeboat"
pattern rather than a second Region, and it is a real gap in this programme's
story. Monzo's publicly-described "Stand-in" — a simplified banking system on
Google Cloud running at ~1% of the primary platform's cost, processing real
transactions daily — is the best-known worked example of that pattern; it was
presented at AWS re:Invent 2025 (HMC201) with Monzo's Andrew Lawson. Treat the
re:Invent framing as vendor-adjacent, but the existence and shape of Stand-in is
Monzo's own public claim. → [[regulatory-drivers]].

### UK — FCA PS21/3 and PRA SS1/21

Relevant to the **EU pair's London standby** and to any UK-regulated entity in
the group.

- [FCA PS21/3, *Building operational resilience*](https://www.fca.org.uk/publications/policy-statements/ps21-3-building-operational-resilience) — Handbook rules. Firms must identify **important business services**, "set impact tolerances for the maximum tolerable disruption" for each, map the people/processes/technology/third parties that support them, carry out **scenario testing** "to a level of sophistication necessary", and "make the necessary investments to enable them to operate consistently within their impact tolerances."
- [PRA SS1/21, *Operational resilience: Impact tolerances for important business services*](https://www.bankofengland.co.uk/prudential-regulation/publication/2021/march/operational-resilience-impact-tolerances-for-important-business-services-ss) — the PRA's parallel expectations for dual-regulated firms.
- **Dates:** identification, impact tolerances and initial mapping by **31 March 2022**; full compliance — mapping and testing complete, investments made — by **31 March 2025**. The transitional period is over.

**Why this matters more than the engineering argument:** an impact tolerance is
a *number the firm itself has published to its regulator*. If a firm has told
the FCA that its important business service can tolerate, say, four hours of
disruption, and its architecture's honest worst-case regional recovery is
"rebuild from backups over two days", the gap is a regulatory finding, not an
engineering preference. **Someone inside the company already knows what those
numbers are, and they should be the source of truth for the Helios RPO/RTO
rather than the 2h/15m working assumption.** → [[open-decisions]],
[[rpo-rto-analysis]].

### Canada

No equivalent finding was made for the **CA pair**. OSFI has guidance in this
space (B-13 on technology and cyber risk, E-21 on operational resilience) but
**this note has not verified its contents against a primary source and makes no
claim about what it requires.** That is a genuine gap —
[[regulatory-drivers]] should close it.

---

## The anti-pattern list

The short version, for a review checklist.

| # | Anti-pattern | Source |
|---|---|---|
| 1 | Failover step that writes a Route 53 record | AWS Fault Isolation Boundaries |
| 2 | Creating/updating IAM resources during failover | AWS Fault Isolation Boundaries |
| 3 | Creating ALBs, RDS instances, API Gateway endpoints or S3 buckets at failover time (all take a `us-east-1` Route 53/S3 control-plane dependency) | AWS Fault Isolation Boundaries |
| 4 | Relying on the default **global** STS endpoint | AWS Fault Isolation Boundaries; STS impaired 2021/2023/2025 |
| 5 | Relying on console or federated login during the incident | Oct 2025 PES — federated login failed outside `us-east-1` |
| 6 | Driving failover off CloudWatch alarms in the primary Region | Nov 2020 PES — alarms went `INSUFFICIENT_DATA` |
| 7 | Fully automatic failover with no human gate | AWS DR whitepaper; NLB auto-failover made Oct 2025 worse |
| 8 | Treating replication as a backup | Salesforce NA14; AWS DR whitepaper |
| 9 | Testing the standby at token load rather than production load | Salesforce NA14 |
| 10 | Never restoring a backup end-to-end | GitLab 2017 |
| 11 | Sizing the standby on the assumption Auto Scaling will work during the incident | Ably Oct 2025; AWS DR whitepaper (Auto Scaling is a control plane action) |
| 12 | Runbooks, credentials and comms hosted in the primary Region | AWS's own 2017/2020 status-page failures |
| 13 | Assuming the standby Region is always healthy | `us-west-2`, 15 Dec 2021 |
| 14 | Hard-coding an ARC endpoint discovery call instead of the endpoints themselves | AWS guidance: "hardcode or bookmark your five Regional cluster endpoints" |

---

## Open questions for the company

- **Is the company in scope for DORA and/or FCA/PRA operational resilience?** If yes, the impact tolerances already filed with the regulator supersede the 2h/15m working assumption.
- **Does "RTO 15 minutes" mean from incident start or from decision-to-failover?** Every lesson in this note bends on the answer.
- **Should `us-east-1` remain the US pair's primary?** C1 is a real argument for flipping the US pair. → [[region-pair-selection]].
- **Does a second AWS Region satisfy the concentration-risk limb of DORA, or only the continuity limb?** If only the latter, the programme has a scope gap that no amount of Terraform closes.
- **Has the company ever restored a production database from backup, end to end, and timed it?** GitLab's answer to that question was discovered on the day.

## Still to research

- OSFI B-13 / E-21 against a primary source, for the CA pair.
- A first-party post-mortem from a regulated EU or UK firm describing an actual AWS regional failover. Extensive searching found none; the sector does not publish these.
- Verified AWS inter-Region data-transfer pricing for the three specific pairs — **deliberately not quoted here** because it could not be confirmed against an AWS price page during this research. → [[cost-model]].

## Sources

**First-party AWS guidance (the framework)**
- [Disaster Recovery of Workloads on AWS — Disaster recovery options in the cloud](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html) — the four strategies; the data-plane rule; the caution against automatic failover; the warm-vs-hot standby cost dial; the admission that many workloads only need backup and restore.
- [AWS Fault Isolation Boundaries — Global services](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/global-services.html) — control-plane locations and AWS's own anti-pattern list.

**First-party AWS post-event summaries** (detail in [[aws-regional-outages]])
- [Feb 2017 S3](https://aws.amazon.com/message/41926/) · [Nov 2020 Kinesis](https://aws.amazon.com/message/11201/) · [Dec 2021 network](https://aws.amazon.com/message/12721/) · [Jun 2023 Lambda](https://aws.amazon.com/message/061323/) · [Jul 2024 Kinesis](https://aws.amazon.com/message/073024/) · [Oct 2025 DynamoDB DNS](https://aws.amazon.com/message/101925/)

**First-party company post-mortems and accounts**
- GitLab, [*Postmortem of database outage of January 31*](https://about.gitlab.com/blog/postmortem-of-database-outage-of-january-31/) and [*GitLab.com database incident*](https://about.gitlab.com/blog/gitlab-dot-com-database-incident) — five recovery mechanisms, none working.
- Paddy Byers (Ably), [*AWS us-east-1 outage: How Ably's multi-region architecture held up*](https://ably.com/blog/multi-region-resilience-aws-outage) — multi-region working, with the control plane still the binding constraint.
- Cooper Bethea (Slack), [*Slack's Migration to a Cellular Architecture*](https://slack.engineering/slacks-migration-to-a-cellular-architecture/) — AZ cells and fast drains inside one Region.

**Third-party analysis and reporting**
- Availability Digest, [*Salesforce Takes a Dive*](https://availabilitydigest.com/public_articles/1107/salesforce.pdf), July 2016 — the NA14 failover that corrupted the primary. Draws on Salesforce's own RCM document.
- [The Register, *Salesforce.com crash caused data loss*](https://www.theregister.com/2016/05/13/salesforcecom_crash_caused_data_loss/) — corroborates the data-loss window.
- Sergio De Simone, [*How Authress Designed for Resilience and Survived a Major AWS Outage*](https://www.infoq.com/news/2025/12/infrastructure-resilience-aws/), InfoQ, Dec 2025.
- [ThousandEyes, AWS Outage Analysis: December 15, 2021](https://www.thousandeyes.com/blog/aws-outage-analysis-december-15-2021).

**Opinion / counter-argument (labelled)**
- Corey Quinn, [*Lessons in Trust from us-east-1*](https://www.lastweekinaws.com/blog/lessons-in-trust-from-us-east-1/), 15 Dec 2021 — "you cannot have a multi-region failover strategy on AWS that features AWS's us-east-1 region."

**Regulatory**
- [DORA Article 12](https://www.digital-operational-resilience-act.com/Article_12.html) — redundant ICT capacities, secondary site at geographical distance with a distinct risk profile, periodic testing of restoration.
- [EIOPA — Digital Operational Resilience Act](https://www.eiopa.europa.eu/digital-operational-resilience-act-dora_en) — scope and application date.
- [FCA PS21/3, *Building operational resilience*](https://www.fca.org.uk/publications/policy-statements/ps21-3-building-operational-resilience) — important business services, impact tolerances, scenario testing, 31 Mar 2022 / 31 Mar 2025 dates.
- [PRA SS1/21](https://www.bankofengland.co.uk/prudential-regulation/publication/2021/march/operational-resilience-impact-tolerances-for-important-business-services-ss) — PRA expectations for dual-regulated firms.

**Vendor-adjacent (labelled: AWS conference/marketing surface)**
- AWS re:Invent 2025 session HMC201, *Architecting resilient multicloud operations, feat. Monzo Bank* — Monzo "Stand-in" on Google Cloud at ~1% of primary platform cost. Listed in the [AWS Cloud Operations Blog re:Invent 2025 resilience session guide](https://aws.amazon.com/blogs/mt/guide-to-aws-cloud-resilience-sessions-at-reinvent-2025/).
