---
title: Failover Orchestration — the 15-minute RTO, decomposed
tags: [operations, failover, orchestration, rto, arc, ssm, step-functions, runbook]
status: researched
replication: N/A — this is a process note, not a service note
rpo_achievable: N/A
rto_achievable: "9–14 min from decision, if the sequence is parallelised. 25–45 min from incident start with a human decision gate."
meets_targets: conditional — depends entirely on which RTO definition the business is holding you to
updated: 2026-09-17
---

# Failover Orchestration

> This is the operational half of the programme. [[aws-route53]], [[aws-rds-postgres]],
> [[aws-eks]] and the rest establish that the standby *can* serve. This note is
> about the fact that somebody has to pull eleven levers, in order, under
> pressure, at 3am — and that the 15-minute RTO is won or lost in the four
> minutes before anyone touches a lever, not in the levers themselves.

## TL;DR

- **The most important open question in this entire vault is which RTO you are held to.** "15 minutes from incident start" and "15 minutes from the decision to fail over" differ by 15–30 minutes of detection, paging, triage and deliberation. [[research-brief]] assumes the second. **Nobody has confirmed that with the business.** Go and confirm it before designing anything else — see [[#The RTO definition problem]].
- **Detection + human deliberation routinely cost more than the mechanical steps.** Realistic budget: ~14 minutes of detect/page/triage/decide, against ~9–14 minutes of machine work. AWS says this explicitly in the DR whitepaper: *"It is critical to factor in incident detection, notification, escalation, discovery and declaration into your planning and objectives."*
- **The mechanical sequence only fits inside 900 seconds if you parallelise it.** Serialised, it is ~20 minutes. Fence → promote DB *in parallel with* scale compute → enable consumers → shift traffic → validate is ~14. There is no slack for a surprise.
- **Recommendation: manual trigger, fully automated execution.** AWS's own words: *"Manually initiated failover is therefore often used. In this case, you should still automate the steps for failover, so that the manual initiation is like the push of a button."* The [GitHub October 2018 incident](https://github.blog/news-insights/company-news/oct21-post-incident-analysis/) is the canonical case for why unattended cross-region promotion is dangerous — a 43-second network blip cost 24 hours.
- **The orchestrator must not live in the region that is failing, and "not in the primary" is not enough.** The `us-east-1` primary hosts the IAM, Route 53 and CloudFront control planes for the *whole account*, so a `us-east-1` event degrades the failover path for the EU and CA pairs too. Recommendation: **ARC Region switch** (AWS-managed, *"a data plane in each AWS Region, so that you can execute your Region switch plan without taking a dependency on the Region that you're deactivating"*) with **SSM Automation documents in a third region** as the self-built fallback. See [[#Where the orchestrator lives]].

## The RTO definition problem

**Read this section to the stakeholder who owns the number.**

[[research-brief]] states the target as "RTO: 15 minutes. From decision-to-fail-over to serving traffic." That is a *defensible* definition and it is the one the rest of this vault has been written against. It is also **not the definition most business stakeholders, auditors or regulators have in their heads**, and it is not the definition AWS uses.

AWS's [Detection](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/detection.html) page in the DR whitepaper is unambiguous, verbatim:

> If your recovery time objective is one hour, then you need to detect the incident, notify appropriate personnel, engage your escalation processes, evaluate information (if you have any) on expected time to recovery (without executing the DR plan), declare a disaster and recover **within an hour**.

> It is critical to factor in incident detection, notification, escalation, discovery and declaration into your planning and objectives to provide realistic, achievable objectives that provide business value.

And, in a note on the same page that is worth reading twice:

> If stakeholders decide not to invoke DR even though the RTO would be at risk, then re-evaluate DR plans and objectives. The decision not to invoke DR plans may be because the plans are inadequate or there is a lack of confidence in execution.

So AWS's position is: **RTO starts when the incident starts.** Under that clock, a 15-minute RTO with a human decision gate is *arithmetically impossible* and you should stop pretending otherwise. Under the clock-starts-at-decision definition it is achievable but tight.

This is not pedantry. The two definitions produce different architectures:

| | **Clock starts at incident** | **Clock starts at decision** |
|---|---|---|
| Achievable at 15 min? | **No, with a human in the loop.** Only with fully automatic failover on a fast, deep health signal. | **Yes**, barely, if the sequence is parallelised. |
| Failover trigger | Must be automated. Composite alarm → orchestrator, no approval step. | Human approval step is affordable. |
| Database choice | Forces [[aws-aurora-global-database]] (managed failover, topology survives, reversible-ish). An irreversible [[aws-rds-postgres]] promotion fired by a robot is not acceptable. | RDS cross-region replica is viable. |
| Data-loss posture | You accept that a false positive will sometimes destroy the primary's last minutes of writes. | A named human accepts the loss each time. |
| Cost | Higher — deep health checks, canaries, tighter alarms, probably Aurora. | Lower. |
| What you tell the auditor | "Automated failover, tested quarterly, MTTR measured." | "15-minute execution RTO; total recovery objective is 45 minutes." |

> [!danger] The action for the user
> Go to whoever owns the 15-minute number and ask one question: **"Does the 15 minutes include the time to notice and decide, or does it start when we press the button?"**
>
> If the answer is "it includes detection," you have two honest responses: (a) renegotiate to a **total recovery objective of 45 minutes with a 15-minute execution objective inside it**, or (b) commit to fully automatic failover and accept false positives, which means Aurora and not RDS. There is no third option where a human deliberates and you still hit 15 minutes from incident start.
>
> This is raised as an open thread in [[CLAUDE]] and belongs in [[open-decisions]] and [[rpo-rto-analysis]].

### Suggested vocabulary, so the ambiguity cannot recur

Adopt three named numbers and put them in the runbook header:

| Term | Definition | Proposed value |
|---|---|---|
| **TTD** — time to detect | Incident start → a human is looking at it | Target ≤ 5 min |
| **TTDecide** — time to decide | Human looking → "we are failing over", spoken by a named person | Target ≤ 5 min |
| **RTO(exec)** — execution RTO | Decision → standby validated and serving | **15 min — the vault's target** |
| **RTO(total)** | Incident start → serving | TTD + TTDecide + RTO(exec) ≈ **25 min best case, 45 min realistic** |

Publishing all four numbers is more honest than publishing one, and it makes the improvement work legible: shaving TTD is a monitoring project ([[observability-multi-region]]), shaving TTDecide is a *decision-criteria* project (this note), shaving RTO(exec) is an engineering project (everything else in the vault).

## The time budget

Two tables. The first is the one people build. The second is the one that happens.

### Budget A — RTO(exec): decision → serving, 900 seconds

Assumes warm standby per [[research-brief]]: RDS cross-region replica running, EKS system node group up with workload nodes at floor, queues and ESMs pre-created and disabled, ARC routing controls pre-created, DNS records pre-created at TTL 60.

| # | Step | Serial cost | Can run in parallel with | Source / note |
|---|---|---|---|---|
| 0 | Freeze the Terraform pipeline for both regions | 30 s | everything | [[aws-rds-postgres#Failover procedure]] — an `apply` mid-failover is worse than the outage |
| 1 | **Fence the primary** — stop writes (scale app to zero / SG lockdown) | 60–120 s | — | [[split-brain-and-fencing]] |
| 2 | **Promote the database** | **RDS: 300 s+ ("several minutes or longer")**<br>Aurora managed failover: 60–180 s | step 3 | [[aws-rds-postgres#How long does promotion actually take]] · [[aws-aurora-global-database#Switchover vs failover]] |
| 3 | **Scale standby compute** — EKS node group, then workload replicas | 240–300 s | step 2 | [[aws-eks#Failover procedure]] |
| 4 | **Enable consumers** — SQS consumers, Lambda event source mappings, EventBridge rules, schedulers | 60–120 s | — (must follow 2) | [[aws-lambda]], [[aws-sqs]], [[aws-eventbridge]] |
| 5 | **Shift traffic** — ARC routing control flip + DNS propagation + TTL 60 | 90–120 s | — | [[aws-route53#The arithmetic against the 15-minute budget]] |
| 6 | **Validate** — synthetic transaction passes, error rate normal, write probe succeeds | 180–300 s | — | [[dr-testing-and-gamedays]] |
| | **Serialised total** | **~16–22 min — FAILS** | | |
| | **Parallelised total** (2 ∥ 3) | **≈ 9.5–14 min — PASSES, with no slack** | | |

**Read the two totals again.** The difference between meeting and missing the RTO is entirely whether steps 2 and 3 are kicked off concurrently. A human working down a numbered list will serialise them, because numbered lists are serial. **This is the single strongest argument for an orchestrator over a wiki page** — a state machine runs branches in parallel; a tired human does not.

Note also that **validation is 180–300 seconds and it is not optional**. Teams routinely leave it out of the budget and then "meet RTO" by declaring victory when pods are `Running`. Declaring success on liveness rather than on a real customer transaction is how you discover at 09:00 that the standby has been serving 500s for five hours.

### Budget B — RTO(total): incident start → serving

| # | Phase | Realistic | Best achievable | What drives it |
|---|---|---|---|---|
| 1 | **Detection** — from first customer impact to an alarm firing | 60–300 s | 30–40 s | CloudWatch alarm at 1-min period × 3 datapoints = 180 s. A Route 53 fast health check (10 s × 3) = 30 s + aggregation. See [[#Detection is a budget line, not a given]]. |
| 2 | **Notification + acknowledgement** — page delivered, human awake and at a laptop | **300 s** | 120 s | On-call ack targets are typically ~5 min. At 3am this is a person finding a laptop. Not compressible by engineering. |
| 3 | **Triage** — "is this a regional event, or did we just deploy something?" | **300–900 s** | 120 s | The dominant cost, and the one nobody budgets. See below. |
| 4 | **Decision** — a named person says "we are failing over" and accepts the data loss | 120–300 s | 60 s | If the criteria are written down in advance this is fast. If they are not, it is a committee. |
| 5 | **Execution + validation** (Budget A) | 570–840 s | 570 s | |
| | **Total** | **≈ 25–45 minutes** | **≈ 15 minutes** | |

**Phases 1–4 sum to 12–25 minutes. The mechanical work sums to 9.5–14.** Detection and deliberation are *the larger half of the incident*, and they are where the cheap wins are:

- **Phase 3 (triage) is where the time actually goes, and it is a documentation problem, not a technology problem.** The question "regional event or bad deploy?" is answerable in 60 seconds if you have (a) a deploy timeline on the same dashboard as the error rate, (b) the AWS Health Dashboard open *from a machine outside the affected region*, and (c) a synthetic canary running from a **third** region that distinguishes "the region is unreachable" from "the app is broken". See [[dr-testing-and-gamedays#Continuous verification]].
- **Phase 4 is near-zero if the go/no-go criteria are pre-written and pre-agreed.** "Two independent external probes have failed for > 5 minutes AND the AWS Health Dashboard shows a regional event AND replica lag is inside RPO" is a decision a single on-call engineer can make at 3am. "Use your judgement" is a 20-minute conference call.
- **Phase 2 is not an engineering problem** and no amount of Terraform will fix it. Budget the five minutes honestly.

### Detection is a budget line, not a given

AWS's guidance, verbatim from the whitepaper's [Detection](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/detection.html) page:

> For the most stringent RTO requirements, you can implement automated failover based on health checks. Design health checks that are representative of user experience and based on Key Performance Indicators. Deep health checks exercise key functionality of your workload and go beyond shallow heartbeat checks. Use deep health checks based on multiple signals. **Use caution with this approach so that you do not trigger false alarms because failing over when there is no need to can in itself introduce availability risks.**

Two consequences for the design:

1. **A `/health` liveness endpoint is not a detection mechanism.** It will return 200 while 40% of real requests fail. [[aws-route53]] documents the constraint that makes this hard: a Route 53 HTTP health check must get a response within **2 seconds** after a 4-second connect, so the deep check cannot be *too* deep.
2. **Detection latency trades directly against false-positive rate**, and the trade is the same trade as automated-vs-manual failover. Three consecutive failures at a 10-second interval is 30 seconds and a meaningful false-positive rate. Three datapoints at a 1-minute period is 3 minutes and a much lower one. Pick deliberately, write the number in the runbook, and **measure your historical false-positive rate before you wire anything to an irreversible action.** [[aws-route53]]'s migration step 6 — "create health checks against both regions, attached to nothing, watch them for a week" — is exactly this reconnaissance and it should be done before this note's recommendation is implemented.

## The ordered failover sequence

Six steps. The order is not stylistic; five of the six orderings you could pick cause data loss or a second outage.

```
  0. FREEZE the Terraform pipeline (both regions)
         │
  1. FENCE the primary ──────────────────► writes stop
         │
         ├──────────────┬──────────────────┐
         ▼              ▼                  │
  2. PROMOTE the DB   3. SCALE standby     │  (2 and 3 in parallel)
         │              compute            │
         └──────┬───────┘                  │
                ▼                          │
  4. ENABLE consumers / event source mappings
                │
                ▼
  5. SHIFT traffic (ARC routing control → Route 53)
                │
                ▼
  6. VALIDATE (synthetic transaction, not a liveness probe)
```

### Why each edge in that graph exists

**0 before everything.** A `terraform apply` running against either region during a failover will fight you. Worse, it may *undo* you: [[aws-eks]] gotcha #4 notes that `desired_size` in Terraform will scale your newly-failed-over region back down on the next unrelated apply. Freeze first, in both regions. This is a 30-second step that prevents a category of self-inflicted second outage.

**1 before 2 — fence before you promote.** If you promote the standby while the old primary is still accepting writes, you have two writable databases and a divergence you will be reconciling by hand for a week. This is the whole of [[split-brain-and-fencing]] and it is the step most likely to be skipped, because when the primary looks dead it feels redundant. It is redundant exactly when it is unnecessary and essential exactly when it is not.

**2 before 4 — promote before you enable consumers.** A queue consumer or Lambda event source mapping that starts in the standby while the database there is still a read-only replica does not fail quietly. It dequeues messages, fails to write, and — depending on your retry and DLQ configuration — either burns the redrive count and drops them to a DLQ, or poison-pills the queue. You will have *consumed and lost* real work while believing you were still failed over cleanly. Keep every standby consumer disabled (`aws_lambda_event_source_mapping.enabled = false`, EventBridge rules `DISABLED`, Kubernetes consumer deployments at 0 replicas) and enable them as an explicit, ordered step. See [[messaging-in-flight-data-loss]].

**2 before 5 — promote before you shift traffic. This is the one the brief asks to be spelled out, so here it is in full.**

Suppose you flip DNS first. Traffic lands on the standby ALB, hits pods that are already running, and those pods connect to the standby database — which is still a **read-only replica**. What happens next:

- **Every read succeeds.** The application looks healthy. Dashboards go green. The standby ALB's request count climbs. Somebody says "failover complete".
- **Every write fails** with `ERROR: cannot execute INSERT in a read-only transaction` (Postgres `25006`). In a typical request mix that is 10–30% of requests — enough to be a catastrophe, not enough to make the region look down.
- **The failures are not clean.** Depending on the framework, a write failure mid-request can leave a partially-committed workflow: the Stripe charge succeeded, the order row did not. You are now generating *reconciliation debt* at production traffic rates.
- **Retries make it worse.** Clients retry the failed writes. Your queue consumers retry. You get a write-amplification storm against a database that cannot accept any of it.
- **The rollback is not free.** Flipping DNS back sends traffic to a primary that you have just fenced. You now have two regions that cannot serve writes, and the shortest path out is forward — promote the database anyway, having spent your entire RTO budget discovering this.

The inverse ordering error is cheaper but still bad: promoting the database *long* before shifting traffic means the standby is writable while all traffic still goes to the fenced primary, so customers see a hard outage for the whole gap. That is merely slow. Traffic-before-promotion is *lossy*, and lossy beats slow every time in a postmortem's severity rating.

**3 can run in parallel with 2, and must.** Scaling an EKS node group is ~4–5 minutes ([[aws-eks]]) and RDS promotion is ~5 minutes; run them serially and you have spent 10 of your 15 minutes on two things that do not depend on each other. The only coupling is that workload *pods* should not become Ready (and therefore ALB-healthy) before the database is writable — handle that with a readiness probe that checks `pg_is_in_recovery()` on the writer endpoint, not by serialising the whole step.

**5 last-but-one.** Traffic shift is the commit point. Everything before it is reversible-ish; after it, customers are on the new region.

**6 is not optional and must be a real transaction.** "Pods are Running" is not validation. The validation step must execute a write that touches the database, the queue and the cache, and assert the result — the same synthetic the canary runs (see [[dr-testing-and-gamedays]]). If validation fails, you are in the rollback branch, and the runbook must say what that is *for each step* (see [[failover-runbook-template]] and the skeleton in [[dr-testing-and-gamedays#A usable runbook skeleton]]).

### The rollback points, stated plainly

| After step | Reversible? | How |
|---|---|---|
| 0 freeze | Yes, trivially | Unfreeze |
| 1 fence | **Yes** — and this is the important one | Un-fence: restore the SG rules, scale the app back up. Nothing has been destroyed. **Step 1 is the last cheap abort point.** |
| 2 promote (RDS) | **NO. One-way door.** | There is no `demote-db-instance`. See [[aws-rds-postgres#Promotion is irreversible]] and [[split-brain-and-fencing]]. |
| 2 promote (Aurora managed failover) | Partially | Topology survives; the old primary rejoins as a secondary and you can `switchover` back later. Still lost the writes in flight. |
| 3 scale | Yes | Scale back down. Costs money, not correctness. |
| 4 enable consumers | Mostly | Disable them again, but messages already consumed-and-failed may be in DLQs. |
| 5 shift traffic | Yes, mechanically — **no, practically** | You can flip the routing control back in seconds. But you would be sending traffic to a fenced region with a demoted database. Rolling back step 5 without rolling back step 2 is not a thing. |

**The decision gate belongs between step 1 and step 2**, not at the top. Fencing is cheap and reversible; promotion is not. A runbook that puts the approval before the freeze-and-fence wastes the two minutes in which fencing would have limited the damage.

## Automated vs manual failover

This is a genuine fork and both branches deserve a fair hearing. The vault's rule is to document both and recommend.

### Branch A — automated

A composite CloudWatch alarm (or ARC Region switch trigger) fires and the orchestrator runs the whole sequence with no human.

**For:**
- It is the *only* way to meet 15 minutes measured from incident start. Removes phases 2–4 of Budget B entirely, i.e. 12–25 minutes.
- It is consistent. The robot never forgets step 4 or gets the order wrong.
- It works at 3am on a bank holiday when the on-call rota has a gap.
- For technologies where failover is cheap and reversible — Aurora Global Database managed failover, DynamoDB Global Tables, stateless compute — the cost of a false positive is small.

**Against — the false-positive problem, which is the whole argument:**

The failure mode is not "the automation breaks". It is "the automation works perfectly, on a signal that was wrong." A transient blip — a 43-second fibre cut, a bad deploy of your own health endpoint, a checker-side network partition — triggers an **irreversible** database promotion, and you wake up to two writable databases with divergent data.

The canonical public case is **GitHub, 21–22 October 2018**. Their [post-incident analysis](https://github.blog/news-insights/company-news/oct21-post-incident-analysis/) is one of the best-written in the industry and every sentence of it is on point here, verbatim:

> Connectivity between these locations was restored in 43 seconds, but this brief outage triggered a chain of events that led to **24 hours and 11 minutes of service degradation**.

> The US West Coast data center and US East Coast public cloud Orchestrator nodes were able to establish a quorum and start failing over clusters to direct writes to the US West Coast data center.

> The database servers in the US East Coast data center contained a brief period of writes that had not been replicated to the US West Coast facility. Because the database clusters in both data centers now contained writes that were not present in the other data center, **we were unable to fail the primary back over to the US East Coast data center safely.**

> We decided that the 30+ minutes of data written to the US West Coast data center prevented us from considering options other than failing-forward in order to keep user data safe.

And their remediation, which is the design recommendation this note is making, in their words:

> **Adjust the configuration of Orchestrator to prevent the promotion of database primaries across regional boundaries.**

Note the shape of it precisely: **43 seconds of network fault, 24 hours of degradation, caused by automation doing exactly what it was configured to do.** The automated failover did not fail. It succeeded, on a signal that turned out to be transient, and the *irreversibility of the promotion* converted a 43-second problem into a day-long one. That is the exact risk profile of an automated `promote-read-replica` against an [[aws-rds-postgres]] cross-region replica, which has no demote and no switchover.

**AWS's own position**, verbatim from the [DR whitepaper](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html):

> This failover operation can be initiated either automatically or manually. **Automatically initiated failover based on health checks or alarms should be used with caution.** Even using the best practices discussed here, recovery time and recovery point will be greater than zero, incurring some loss of availability and data. **If you fail over when you don't need to (false alarm), then you incur those losses.** Manually initiated failover is therefore often used. In this case, you should still automate the steps for failover, so that the manual initiation is like the push of a button.

That last sentence is the recommendation of this note, almost word for word.

### Branch B — manual trigger, automated execution

A human decides; a machine executes. The approval is a step in the orchestrator, not a Slack thread.

**For:**
- Eliminates the false-positive class entirely. A 43-second blip does not promote anything.
- Matches the data model. RPO 2h means asynchronous replication, which means failover *loses data by design*. Losing data should be a decision a named person makes, not an emergent property of a threshold. [[aws-route53]] reaches the same conclusion independently.
- Matches the irreversibility. Where the action has no undo (RDS promotion), a human gate is proportionate.
- Auditable. An `aws:approve` step produces `ApproverDecisions` — a JSON map of who approved and when — which is exactly the artefact [[dr-testing-and-gamedays#What an auditor needs]] and DORA want.

**Against:**
- Costs 12–25 minutes of Budget B. Fails an incident-start-clock RTO of 15 minutes.
- Depends on a human being reachable, awake, and authorised. The authorisation list is a real risk: if only two named people may approve and both are asleep or on a plane, your RTO is unbounded.
- Humans hesitate, especially on one-way doors, especially when the criteria are vague. This is a feature until it is 25 minutes long.

### What real organisations actually do

Honest accounting of what is and is not public:

| Organisation | What they do | Source quality |
|---|---|---|
| **GitHub** | Ran automated cross-region MySQL failover via Orchestrator; after the Oct 2018 incident, **explicitly configured it not to promote across regional boundaries.** Automatic within a region, not across. | **Strong** — first-party public postmortem |
| **Netflix** | Region evacuation is a **deliberate, operator-initiated** exercise, executed by tooling. [Project Nimble](https://netflixtechblog.com/project-nimble-region-evacuation-reimagined-d0d0568254d4) reduced evacuation from ~50 minutes to ~8 by keeping hot-standby instance pools per microservice and injecting them into clusters at failover. They also run it *regularly* (Chaos Kong) rather than only in emergencies. | **Strong** — first-party engineering blog. Note Netflix is active/active, so "failover" for them means traffic evacuation, not database promotion — the decision is far cheaper than ours. |
| **AWS (official guidance)** | "Automatically initiated failover ... should be used with caution ... Manually initiated failover is therefore often used." | **Strong** — vendor guidance, quoted above |
| **A large financial AWS customer** (unnamed, via AWS Database Blog) | Aurora Global Database + RDS Proxy + Route 53 CNAME weighting + Lambda canaries probing every 10 seconds requiring **2+ consecutive failures** before triggering + ARC. Measured cross-region recovery at **2 minutes in testing**. | **Medium** — AWS-published customer case study, anonymised. Already captured in [[aws-aurora-global-database#Real-world reports]]. |
| **athenahealth** (via AWS Architecture Blog, [Validating multi-Region DR for Terraform Enterprise with AWS FIS](https://aws.amazon.com/blogs/architecture/validating-multi-region-dr-for-terraform-enterprise-with-aws-fis/)) | **Operator-triggered** four-step failover runbook, validated with FIS. Measured **12–14 minutes from operator trigger to full recovery**, RPO < 1 minute. | **Strong for our purposes** — a real, named company, a real measured number, and a number that is *almost exactly our RTO(exec) budget*. |

> [!note] What is not public
> **No public postmortem was found describing an unplanned, full-region AWS failover of an Aurora Global Database or an RDS cross-region replica at a named company.** This matches the finding in [[aws-aurora-global-database#Real-world reports]] and is not a search failure — full-region evacuations are rare and the companies that do them (Netflix, Amazon itself) are active/active, so their public writing is about traffic steering, not database promotion. **Every cross-region-promotion timing number available to you is a vendor claim or a vendor-published customer test. Measure your own.**

### Recommendation

**Manual trigger, fully automated execution, with one automated exception.**

1. **Default: a human presses one button.** The button is `StartAutomationExecution` on an SSM Automation document, or `StartPlanExecution` on an ARC Region switch plan. Everything after the button is machine-driven, parallelised, and identical every time. This is AWS's "manual initiation is like the push of a button" and it is the right answer for a 2-hour-RPO async-replicated estate where the database promotion is a one-way door.
2. **The go/no-go criteria are written in advance and are boolean**, not judgemental. Draft: *fail over if (external probe from a third region has failed continuously for ≥ 5 min) AND (AWS Health Dashboard reports an event in the primary region OR the RDS/EKS control plane in the primary is unreachable) AND (replica lag < 2 h) AND (a named approver from the on-call list has said so).* Four conditions, all checkable in under a minute.
3. **The one automated exception: in-region, reversible actions.** Aurora *in-region* AZ failover, ALB target draining, EKS node replacement — automate freely, they are cheap and reversible. The rule is: **automate what has an undo; gate what does not.**
4. **Revisit if and only if the RTO is redefined to start at incident.** If the business insists on 15 minutes from incident start, then automation is mandatory, and the database decision changes — you must move to [[aws-aurora-global-database]], whose managed failover preserves the topology and therefore has a cheap-ish undo. **Do not automate an RDS `promote-read-replica`.** That combination — robot trigger, one-way door — is the GitHub incident with the serial numbers filed off.
5. **Build a "stage-1 only" mode.** The orchestrator should support executing steps 0–1 (freeze + fence) automatically on alarm, then stopping for approval before step 2. Fencing is reversible and buys you the split-brain protection *before* the human arrives, which directly shrinks the divergence window that ruined GitHub's day. This is the highest-value hybrid available and it costs one extra `aws:approve` step in the right place.

## The switch: Route 53 ARC routing controls

[[aws-route53]] covers the mechanism in full — routing controls, the five-region quorum data plane, the `RECOVERY_CONTROL` health check type, the $2.50/hour cluster price, the `us-west-2` config plane, the hard-coded-endpoints rule, and the Terraform. **Read that note; this section only covers what the orchestrator needs to know.**

The three things that matter to orchestration:

**1. `UpdateRoutingControlStates` (plural) is atomic.** Flip primary-off and standby-on in a single call, from the orchestrator, and you never occupy the half-switched state. Do not issue two singular calls.

**2. Safety rules are the orchestrator's guardrails, and they are the strongest argument for ARC over a homegrown switch.** Two kinds:
- **Assertion rules** — "at least one of {primary, standby} must be On." Prevents a bug or a fat-fingered override from turning the product off entirely.
- **Gating rules** — "you may not turn the standby On unless the `db-promoted` control is also On." **This encodes step 2-before-step-5 as an enforced invariant rather than a numbered list item.** Model the database promotion as its own routing control, have the orchestrator set it On after promotion succeeds, and gate the traffic control behind it. Now the ordering error described above is *structurally impossible*, not merely documented. This is the single most valuable thing ARC gives you that a shell script does not.

Safety rules can be overridden explicitly during a genuine emergency, which is the right escape hatch.

**3. Readiness checks are effectively unavailable.** [[aws-route53]] records the ARC pricing-page notice that readiness checks are not available to new customers from **30 April 2026**, a date that has passed. Do not design the orchestrator's pre-flight gate around them. Build it from CloudWatch composite alarms and a third-region canary instead — covered in [[dr-testing-and-gamedays#Continuous verification]]. If the company is an existing ARC customer, confirm and revisit.

## Where the orchestrator lives

**The constraint: the orchestrator must not have a dependency on the region it is failing away from.** Obvious when stated; violated constantly in practice, because the natural place to put the automation is next to the thing it automates.

The athenahealth case study above found exactly this, and it is worth quoting because it is a *different* flavour of the dependency than the one people expect — their failover scripts read infrastructure identifiers from Terraform state in S3, and when S3 in the primary became unreachable the scripts could not run:

> a circular dependency: you can't run the failover without access to the infrastructure you're trying to recover from

Their fix was to **hard-code the identifiers into the failover scripts** — the same advice [[aws-route53]] gives about hard-coding the five ARC cluster endpoints rather than calling `DescribeCluster`. Generalise it: **the orchestrator must carry every input it needs as a literal, or read it from a store whose data plane is in a region that is definitely up.**

### The dependency checklist for any orchestrator

Before accepting a placement, walk this list. Each item is a way the orchestrator can be dead exactly when you need it:

| Dependency | Why it bites | Mitigation |
|---|---|---|
| **Compute region** | Lambda/Step Functions/SSM in `eu-west-1` cannot run when `eu-west-1` is impaired | Deploy in the standby *and* a third region |
| **Terraform state in S3** | State bucket in the primary region; the athenahealth failure exactly | Hard-code inputs; never read state at failover time |
| **Secrets / parameters** | SSM Parameter Store has **no native cross-region replication** ([[aws-ssm-parameter-store]]) | Replicate explicitly, or use Secrets Manager replicas ([[aws-secrets-manager]], already done in prod) |
| **Container image in ECR** | Orchestrator packaged as a container, image in the primary's ECR | ECR cross-region replication ([[aws-ecr]]), or no containers in the recovery path |
| **IAM control plane (`us-east-1`)** | Any orchestrator step that *creates or modifies* an IAM role or policy — including IAM-based fencing | **Pre-create every role and policy.** See below and [[split-brain-and-fencing]] |
| **Route 53 control plane (`us-east-1`)** | `ChangeResourceRecordSets` to flip weights | Use ARC routing controls (data plane) |
| **Identity provider / SSO** | If the IdP is AWS-hosted or federates through an impaired region, nobody can log in to approve anything | Break-glass long-lived credentials in a vault — AWS explicitly recommends this |
| **Observability** | The dashboard you use to make the decision is served from the dead region | Cross-region or third-party observability ([[observability-multi-region]]) |

> [!danger] The `us-east-1` problem is worse than "the US pair's primary is `us-east-1`"
> The [AWS Fault Isolation Boundaries whitepaper](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/global-services.html) lists the partitional (global) services and their control-plane home regions, verbatim:
>
> > + AWS IAM (`us-east-1`)
> > + AWS Organizations (`us-east-1`)
> > + AWS Account Management (`us-east-1`)
> > + Route 53 Application Recovery Controller (ARC) (`us-west-2`)
> > + AWS Network Manager (`us-west-2`)
> > + Route 53 Private DNS (`us-east-1`)
>
> plus the edge services: Route 53 Public DNS, CloudFront, WAF for CloudFront, ACM for CloudFront and Shield Advanced in `us-east-1`; Global Accelerator in `us-west-2`.
>
> And the whitepaper's own list of anti-patterns, verbatim, includes:
>
> > **Creating or updating IAM resources, including IAM roles and policies, during a failover. This typically isn't intentional, but might be a result of an untested failover plan.**
>
> So: a `us-east-1` impairment is simultaneously (a) the US pair's primary-region disaster, (b) the loss of your ability to change DNS or IAM for **all three pairs**, and (c) the loss of any fencing technique that depends on modifying an IAM policy. The EU and CA pairs are not failing over, but their failover *capability* is degraded by an event in a region they do not use. **This is the argument for ARC-based, data-plane-only orchestration, stated as strongly as it can be stated.**

### The four candidate orchestrators

#### Option 1 — ARC Region switch (recommended)

AWS's managed answer. From the [Region switch docs](https://docs.aws.amazon.com/r53recovery/latest/dg/region-switch.html), verbatim, the line that settles the placement question:

> **A data plane in each AWS Region, so that you can execute your Region switch plan without taking a dependency on the Region that you're deactivating.**

A *plan* contains *workflows* made of *steps* containing *execution blocks*, run in sequence or in parallel. Relevant properties:

- **Graceful vs ungraceful modes.** From the [components doc](https://docs.aws.amazon.com/r53recovery/latest/dg/components-rs.html): *"When your environment is healthy, you can use the graceful workflow to run all steps for an orderly plan execution. The ungraceful workflow mode uses only the necessary steps and actions."* This maps precisely onto the switchover-vs-failover distinction in [[aws-aurora-global-database#Switchover vs failover]] — **one plan, two modes, so you rehearse the graceful path routinely and still have the ungraceful path defined.** Neither Step Functions nor SSM gives you this for free.
- **Manual Approval execution block** exists, so the human gate lives inside the plan.
- **Triggers**: *"you specify one or more Amazon CloudWatch alarms and define which alarm conditions ... should initiate plan execution."* So you can move from manual to automatic later without rebuilding — which makes the recommendation above reversible if the RTO definition changes.
- **It measures your RTO for you.** *"Region switch calculates an actual recovery time value for each plan execution, to help you evaluate if the plan is meeting your objectives."* Application health alarms per region feed this. **This is the auditor artefact, generated automatically** — see [[dr-testing-and-gamedays#What an auditor needs]].
- **Post-recovery workflows** run *"in the Region that was previously impaired"* and support an **RDS Create Cross-Region Replica** block — i.e. AWS has modelled the failback reseed as a first-class step. See [[split-brain-and-fencing#Failback]].
- **Practice mode** for rehearsal.
- **$70 per plan per month** ([ARC pricing](https://aws.amazon.com/application-recovery-controller/pricing/)), versus $1,825/month for a routing control cluster. Three plans (one per pair) is $210/month.
- **Terraform provider support** was announced in the [December 2025 Region switch update](https://aws.amazon.com/about-aws/whats-new/2025/12/amazon-application-recovery-controller-region-switch-new-capabilities), alongside post-recovery workflows and native RDS execution blocks.

Its own best-practices page adds a caveat you must plan around, verbatim:

> If you use these execution blocks in a plan, **Region switch does not guarantee that the desired compute capacity with be attained.** If you have a critical application and need to guarantee access to capacity, we recommend that you reserve the capacity.

Which is the same warning [[aws-eks#EC2 capacity in the standby during a real regional disaster]] gives, and points at On-Demand Capacity Reservations.

**Why this is the recommendation:** the 15-minute problem is *orchestration under a control-plane-hostile failure*, and Region switch is the only option where AWS operates the orchestrator's data plane in every region. Everything else on this list requires you to solve that yourself, correctly, and then keep solving it.

**Caveats, stated honestly:** it is a young service (GA August 2025, capabilities added December 2025). Terraform provider coverage should be verified against the version you are pinned to before committing. And it is one more AWS service in the recovery path — though one specifically engineered not to be.

#### Option 2 — SSM Automation documents (recommended as the fallback, and as the artefact regardless)

Runbooks-as-code. An `aws_ssm_document` of `document_type = "Automation"` is a YAML file in the same Terraform monorepo, versioned, reviewed in a PR, and — crucially — **it is simultaneously the executable and the documentation**. That is a real advantage over a state machine: the thing the auditor reads and the thing that runs are the same file.

The `aws:approve` action is the human gate. From the [docs](https://docs.aws.amazon.com/systems-manager/latest/userguide/automation-action-approve.html), verbatim:

> Temporarily pauses an automation until designated principals either approve or reject the action. After the required number of approvals is reached, the automation resumes.

> The default timeout for this action is 7 days (604800 seconds) and the maximum value is 30 days (2592000 seconds).

**A 7-day default timeout on a step inside a 15-minute RTO is absurd. Set `timeoutSeconds` explicitly — 600 is a defensible choice** — and set `onFailure: Abort` so a timed-out approval stops rather than proceeds.

> [!warning] Two gotchas on `aws:approve`
> 1. **"This action doesn't support multi-account and Region automations."** Verbatim from the docs. If you were planning to use SSM's multi-account/multi-Region automation feature to drive both regions from one execution, **you cannot have an approval step in it.** Structure it as: approval in a single-account/single-region document, which then invokes the cross-region work. This will surprise you at design time if you don't know it now.
> 2. **`NotificationArn` requires the SNS topic name to be prefixed with `Automation`.** Verbatim: *"The title of the Amazon SNS topic must be prefixed with 'Automation'."* A one-character naming problem that silently breaks your page.

Output is `ApprovalStatus` and `ApproverDecisions` — *"a JSON map that includes the approval decision of each approver"* — which, together with the execution history, is your audit trail.

**Real Terraform.** The shape below is what you'd actually want in a cookiecutter monorepo: one module, instantiated once per *orchestrator region*, with the region pair as variables so the same document drives EU, US and CA.

```hcl
# modules/failover-runbook/variables.tf
variable "pair_name"        { type = string }  # "eu" | "us" | "ca"
variable "primary_region"   { type = string }
variable "standby_region"   { type = string }
variable "db_standby_id"    { type = string }  # RDS replica identifier in the standby
variable "eks_standby_name" { type = string }
variable "eks_nodegroup"    { type = string }
variable "nodegroup_target_size" { type = number }

# Hard-coded at plan time on purpose. Do NOT look these up at failover time —
# DescribeCluster is an ARC CONTROL-PLANE call and defeats the entire design.
variable "arc_cluster_endpoints" {
  type        = list(string)
  description = "All five ARC regional cluster endpoints, as literals."
  validation {
    condition     = length(var.arc_cluster_endpoints) == 5
    error_message = "ARC clusters expose exactly five regional endpoints; bake in all five."
  }
}
variable "arc_routing_control_primary_arn" { type = string }
variable "arc_routing_control_standby_arn" { type = string }
variable "arc_routing_control_db_arn" {
  type        = string
  description = "Gating control: the standby traffic control cannot be turned On unless this is On."
}

variable "approver_arns" {
  type        = list(string)
  description = "IAM principals authorised to accept the data-loss decision. Max 10 (SSM limit)."
  validation {
    condition     = length(var.approver_arns) >= 3 && length(var.approver_arns) <= 10
    error_message = "At least 3 approvers, or your RTO is unbounded when two people are asleep. SSM caps at 10."
  }
}
variable "approval_timeout_seconds" {
  type    = number
  default = 600 # NOT the 7-day default. A failover approval that takes 10 minutes has already failed.
}
```

```hcl
# modules/failover-runbook/main.tf
#
# NOTE ON PROVIDER: this module is instantiated with a provider pinned to the
# ORCHESTRATOR region, which is neither the primary nor (ideally) the standby.
# See "Where the orchestrator lives". The document ACTS on other regions via
# the aws:executeAwsApi `Service`/`Api` inputs, which accept a region override
# through the assumed role's endpoint configuration; where that is awkward, the
# aws:executeScript steps below call the CLI/boto with an explicit region.

resource "aws_sns_topic" "approval" {
  # The "Automation" prefix is MANDATORY for aws:approve NotificationArn.
  name = "Automation-failover-${var.pair_name}"
}

resource "aws_ssm_document" "failover" {
  name            = "helios-failover-${var.pair_name}"
  document_type   = "Automation"
  document_format = "YAML"

  content = yamlencode({
    schemaVersion = "0.3"
    description   = "Fail over the ${var.pair_name} pair from ${var.primary_region} to ${var.standby_region}."
    assumeRole    = aws_iam_role.failover.arn

    parameters = {
      Reason = {
        type        = "String"
        description = "Incident ticket reference. Recorded for the auditor."
      }
      SkipFence = {
        type           = "Boolean"
        default        = false
        description    = "Only true if the primary region is confirmed unreachable. Read split-brain-and-fencing first."
      }
    }

    mainSteps = [
      # ---------- 0. FREEZE ----------
      {
        name        = "FreezePipeline"
        action      = "aws:executeScript"
        onFailure   = "Abort"
        timeoutSeconds = 60
        inputs = {
          Runtime = "python3.11"
          Handler = "handler"
          Script  = file("${path.module}/scripts/freeze_pipeline.py")
        }
      },

      # ---------- 1. FENCE (reversible — do it BEFORE the approval gate) ----------
      {
        name           = "FencePrimary"
        action         = "aws:executeScript"
        onFailure      = "Continue" # the primary may already be unreachable; that is fine
        timeoutSeconds = 180
        inputs = {
          Runtime = "python3.11"
          Handler = "handler"
          Script  = file("${path.module}/scripts/fence_primary.py")
          InputPayload = {
            region   = var.primary_region
            skip     = "{{ SkipFence }}"
          }
        }
      },

      # ---------- THE ONE-WAY DOOR ----------
      {
        name           = "ApprovePromotion"
        action         = "aws:approve"
        timeoutSeconds = var.approval_timeout_seconds
        onFailure      = "Abort" # timing out must NOT mean "proceed"
        inputs = {
          NotificationArn     = aws_sns_topic.approval.arn
          MinRequiredApprovals = 1
          Approvers           = var.approver_arns
          Message = join("\n", [
            "FAILOVER APPROVAL — ${var.pair_name} pair",
            "Reason: {{ Reason }}",
            "",
            "Approving PROMOTES the standby database in ${var.standby_region}.",
            "For RDS Postgres this is IRREVERSIBLE — there is no demote.",
            "You are accepting up to 2 hours of data loss (RPO).",
            "",
            "Confirm before approving:",
            "  [ ] External probe from a THIRD region failing >= 5 min",
            "  [ ] AWS Health Dashboard shows an event in ${var.primary_region}",
            "  [ ] Replica lag < 2h",
            "  [ ] The primary is FENCED (step FencePrimary succeeded, or region unreachable)",
          ])
        }
      },

      # ---------- 2 and 3 IN PARALLEL ----------
      # SSM Automation runs mainSteps SEQUENTIALLY. To parallelise, fan out from
      # one executeScript that starts both and polls. This is the single biggest
      # ergonomic weakness of SSM vs Step Functions — see the comparison table.
      {
        name           = "PromoteDbAndScaleComputeInParallel"
        action         = "aws:executeScript"
        onFailure      = "Abort"
        timeoutSeconds = 600
        inputs = {
          Runtime = "python3.11"
          Handler = "handler"
          Script  = file("${path.module}/scripts/promote_and_scale.py")
          InputPayload = {
            standby_region    = var.standby_region
            db_instance_id    = var.db_standby_id
            eks_cluster       = var.eks_standby_name
            eks_nodegroup     = var.eks_nodegroup
            nodegroup_size    = var.nodegroup_target_size
          }
        }
        outputs = [
          { Name = "WalLsnAtPromotion", Selector = "$.Payload.wal_lsn", Type = "String" },
          { Name = "PromotionSeconds",  Selector = "$.Payload.promote_secs", Type = "Integer" },
        ]
      },

      # ---------- 2b. Flip the GATING routing control, unlocking step 5 ----------
      {
        name           = "MarkDatabasePromoted"
        action         = "aws:executeScript"
        onFailure      = "Abort"
        timeoutSeconds = 60
        inputs = {
          Runtime = "python3.11"
          Handler = "handler"
          Script  = file("${path.module}/scripts/arc_update.py")
          InputPayload = {
            endpoints = var.arc_cluster_endpoints
            updates   = [{ arn = var.arc_routing_control_db_arn, state = "On" }]
          }
        }
      },

      # ---------- 4. CONSUMERS ----------
      {
        name           = "EnableConsumers"
        action         = "aws:executeScript"
        onFailure      = "Abort"
        timeoutSeconds = 180
        inputs = {
          Runtime = "python3.11"
          Handler = "handler"
          Script  = file("${path.module}/scripts/enable_consumers.py")
          InputPayload = { region = var.standby_region }
        }
      },

      # ---------- 5. TRAFFIC — atomic, both controls in ONE call ----------
      {
        name           = "ShiftTraffic"
        action         = "aws:executeScript"
        onFailure      = "Abort"
        timeoutSeconds = 120
        inputs = {
          Runtime = "python3.11"
          Handler = "handler"
          Script  = file("${path.module}/scripts/arc_update.py")
          InputPayload = {
            endpoints = var.arc_cluster_endpoints
            updates = [
              { arn = var.arc_routing_control_primary_arn, state = "Off" },
              { arn = var.arc_routing_control_standby_arn, state = "On"  },
            ]
          }
        }
      },

      # ---------- 6. VALIDATE — a real transaction, not a liveness probe ----------
      {
        name           = "ValidateStandby"
        action         = "aws:executeScript"
        onFailure      = "Abort"
        timeoutSeconds = 300
        isEnd          = true
        inputs = {
          Runtime = "python3.11"
          Handler = "handler"
          Script  = file("${path.module}/scripts/validate_synthetic_write.py")
          InputPayload = { region = var.standby_region }
        }
      },
    ]
  })

  tags = {
    Pair    = var.pair_name
    Purpose = "dr-failover"
  }
}

# Alarm-triggered STAGE 1 ONLY: freeze + fence, then stop at the approval gate.
# This shrinks the split-brain divergence window without automating the one-way door.
resource "aws_cloudwatch_event_rule" "stage1_on_regional_alarm" {
  name        = "helios-${var.pair_name}-stage1-fence"
  description = "On a composite regional-health alarm, freeze and fence. Promotion still needs a human."
  event_pattern = jsonencode({
    source      = ["aws.cloudwatch"]
    detail-type = ["CloudWatch Alarm State Change"]
    detail = {
      alarmName = ["helios-${var.pair_name}-primary-region-composite"]
      state     = { value = ["ALARM"] }
    }
  })
}
```

The `arc_update.py` helper must implement the [ARC best practice](https://docs.aws.amazon.com/r53recovery/latest/dg/route53-arc-best-practices.regional.html) of picking a random endpoint and retrying across all five — [[aws-route53]] has the shell version of exactly this loop.

**The honest weakness of SSM Automation: `mainSteps` run sequentially.** There is no native parallel construct, and the whole 15-minute budget depends on running promote and scale-out concurrently. The workaround above — one `aws:executeScript` that fans out with threads and polls both — works but puts orchestration logic inside a Python blob, which is precisely the thing runbooks-as-code was supposed to avoid. **This is the real reason Step Functions is on the list.**

#### Option 3 — Step Functions

Natively parallel (`Parallel` state), natively retryable, natively observable, and `.waitForTaskToken` gives you a clean human-approval gate. The [AWS networking blog on orchestrating DR with ARC and Step Functions](https://aws.amazon.com/blogs/networking-and-content-delivery/orchestrate-disaster-recovery-automation-using-amazon-route-53-arc-and-aws-step-functions/) is the reference implementation and it solves the placement problem the way you would have to solve it yourself, verbatim:

> we store the Route 53 ARC regional cluster endpoints, control panel ARN, and the order of Routing Controls in the global DynamoDB tables. Therefore, if the AWS console is inaccessible, the failover and failback sample step functions **deployed in any other AWS Regions** can access the ARC parameters.

and it draws the control-plane/data-plane line explicitly:

> any 'creation & deletion' of ARC Routing Controls are Control Plane operations, and any 'updates' to ARC Routing Controls are a Data Plane operation.

Their ordering is worth comparing to ours — deactivate the primary's controls first, then fail over the database, then activate the standby's — which is the same fence-then-promote-then-shift shape, expressed in routing controls.

**Against:** you deploy the state machine in every region and keep them in sync; the definition is JSON/ASL, which is less readable-as-a-runbook than SSM YAML; and it has no equivalent of ARC's safety rules. It is the right answer only if you reject Region switch *and* find SSM's sequential execution intolerable.

#### Option 4 — a Lambda on a composite alarm

The cheapest thing that could work: a CloudWatch composite alarm → SNS → Lambda that runs the whole sequence.

**Against, decisively:** no approval gate without building one; no step-level visibility; no retry semantics you don't write; a 15-minute Lambda timeout that your 14-minute sequence runs uncomfortably close to; and you must deploy and version it in multiple regions yourself. **Also: this is the shape that automates the one-way door by default**, which is the GitHub failure mode. It has one legitimate use — as the *stage-1 fence-only* responder shown in the Terraform above, where the action is cheap and reversible.

#### Option 5 — a CI pipeline

Tempting: the Terraform monorepo already has pipelines, they have approvals, they have audit trails, engineers know them.

**Against, decisively, three ways:**
1. **It is probably not in AWS.** GitHub Actions / GitLab.com are third-party dependencies in your recovery path. That is not automatically bad — being outside AWS is arguably *good* isolation — but it must be a deliberate decision, and self-hosted runners in the primary region silently reintroduce the dependency.
2. **Pipelines run `terraform apply`, and a failover must not.** Every note in this vault that discusses failover says the same thing: promote out-of-band with the CLI, freeze Terraform, reconcile state afterwards ([[aws-rds-postgres#Terraform state when you promote out-of-band]]). A pipeline whose only verb is `apply` is the wrong tool.
3. **Terraform needs its state backend**, which is an S3 bucket in a region. See athenahealth.

Use the pipeline to *deploy* the orchestrator. Do not use it *as* the orchestrator.

### Comparison

| | ARC Region switch | SSM Automation | Step Functions | Lambda on alarm | CI pipeline |
|---|---|---|---|---|---|
| Regional independence | **AWS-managed data plane in every region** | You deploy per region | You deploy per region | You deploy per region | External, or a runner in a region |
| Parallel steps | Yes, native | **No — sequential only** | Yes, native | You write it | Depends |
| Human approval | Manual Approval block | `aws:approve` (7-day default timeout — override it) | `.waitForTaskToken` | Build it | Native |
| Readable as a runbook | Console/plan view | **Best — it *is* the runbook** | ASL JSON, poor | No | Poor |
| Terraform-declarable | Provider support added Dec 2025 — **verify** | `aws_ssm_document` — mature | `aws_sfn_state_machine` — mature | Mature | N/A |
| Ordering enforced structurally | **Yes, via ARC gating rules** | No | No | No | No |
| Measures your RTO | **Yes, automatically** | No | No | No | No |
| Rehearsal mode | **Practice runs; graceful/ungraceful modes** | Run it in non-prod | Run it in non-prod | No | No |
| Cost | $70/plan/month | ~$0 | ~$0 | ~$0 | existing |
| Maturity | New (GA Aug 2025) | Very mature | Very mature | Mature | Mature |

### Recommended placement

**Primary: ARC Region switch**, one plan per pair, executed through the data plane from whichever region is reachable. This is the only option where regional independence is somebody else's problem.

**Fallback and companion: SSM Automation documents deployed to three regions** — the standby, plus a third region that is in neither pair. Concretely:

| Pair | Primary | Standby | Orchestrator regions |
|---|---|---|---|
| EU | `eu-west-1` | `eu-west-2` | `eu-west-2` + `eu-central-1` |
| US | `us-east-1` | `us-west-2` | `us-west-2` + `us-east-2` |
| CA | `ca-central-1` | `ca-west-1` | `ca-west-1` + `ca-central-1`… **problem** |

> [!warning] The CA pair has nowhere to put a third orchestrator
> With only two Canadian regions, a "third region" for the CA pair is either **outside Canada** — which collides with data residency ([[data-residency]]) — or does not exist. Note carefully that the orchestrator holds *identifiers and control-plane credentials*, not customer data, so hosting it in `us-east-2` is probably a residency non-issue — **but "probably" is not a compliance answer and this needs a legal ruling, not an engineering one.** It is also a strong practical argument for ARC Region switch on the CA pair specifically, since AWS runs the data plane and you take no placement decision at all. Logged in [[#Open questions]] and cross-referenced to [[region-pair-selection]].

**Why the standby region *and* a third:** the standby is the obvious host (it is up, by definition, in the scenario you care about) and covers 95% of cases. The third region covers the case you cannot otherwise cover — a **bilateral** event, or an event in the standby that makes you want to *not* fail over, or a `us-east-1` global-service impairment that degrades every region's control plane at once. Two extra deployments of a YAML document cost approximately nothing.

**What the orchestrator must carry as literals, never look up:**
- the five ARC cluster endpoints and all routing control ARNs
- the standby DB instance identifier and its writer endpoint
- the standby EKS cluster name, node group name and target size
- the hosted zone ID and record names
- the list of event source mapping UUIDs to enable

If any of these are resolved at runtime from Terraform state, SSM Parameter Store in the primary, or a `Describe*` call against an impaired control plane, **the orchestrator has a dependency on the region it is failing away from** and the whole design is decorative.

## Migration path — how to get here from nothing

Additive, and each step is independently useful.

1. **Ask the RTO definition question.** Everything else is contingent on the answer. One conversation, zero engineering.
2. **Write the go/no-go criteria down** and get them agreed. Four boolean conditions. Zero engineering, biggest single reduction in TTDecide.
3. **Build the third-region canary** ([[dr-testing-and-gamedays]]). This is what makes triage fast and makes the first criterion checkable.
4. **Write the sequence as an SSM Automation document and run it in non-prod.** Non-prod has a throwaway replica, so you can actually promote it. Measure every step. This produces your real numbers, which is what Budget A above is currently guessing at.
5. **Add ARC routing controls with the gating rule** (`standby-traffic` gated on `db-promoted`). Now the ordering is enforced.
6. **Evaluate ARC Region switch against the working SSM document.** By this point you know what the sequence is and what it costs, so the evaluation is concrete rather than speculative. This is the [[aws-route53#Decisions to make]] open item.
7. **Add the stage-1 fence-only alarm responder.** Cheap, reversible, shrinks the split-brain window.
8. **Production game day.** [[dr-testing-and-gamedays]].

Nothing here requires a Terraform refactor and nothing forces a resource replacement.

## Gotchas

1. **`aws:approve` has a 7-day default timeout.** Inside a 15-minute RTO. Set `timeoutSeconds` and `onFailure: Abort` explicitly.
2. **`aws:approve` does not work in multi-account/multi-Region SSM automations.** Documented limitation. Structure around it.
3. **The `NotificationArn` SNS topic name must start with `Automation`.** Silent failure otherwise.
4. **SSM `mainSteps` are sequential.** Your 15-minute budget depends on parallelism. Fan out inside a script, or use Step Functions / Region switch.
5. **`DescribeCluster` on an ARC cluster is a control-plane call.** Hard-code the five endpoints. Repeated from [[aws-route53]] because it is the single most common way a "data-plane-only" design turns out not to be.
6. **Creating or updating IAM roles or policies during a failover is an AWS-documented anti-pattern** and depends on `us-east-1`. Pre-create everything the orchestrator's role needs, in both regions, months in advance. This also constrains [[split-brain-and-fencing]].
7. **The default global STS endpoint is `us-east-1`.** If your orchestrator's SDK/CLI has not been configured for regional STS endpoints, it cannot get credentials during a `us-east-1` event. One config line ([AWS STS regionalized endpoints](https://docs.aws.amazon.com/sdkref/latest/guide/feature-sts-regionalized-endpoints.html)) between a working and a decorative failover plan. Verify this.
8. **If your IdP is the only way in, a failover may be unauthenticable.** AWS's ARC best practices recommend break-glass long-lived credentials *"in an on-premises physical safe or a virtual vault."* Contradicts normal security guidance; correct here; needs sign-off, not assumption. See [[aws-iam]] and [[security-posture-of-the-standby]].
9. **Auto Scaling is a control-plane activity.** Verbatim from the whitepaper: *"Because Auto Scaling is a control plane activity, taking a dependency on it will lower the resiliency of your overall recovery strategy."* Your step 3 depends on the EKS/ASG control plane in the standby region. Acceptable — the standby is healthy — but it is a dependency, and ARC's own docs warn capacity is not guaranteed.
10. **The orchestrator's own alarms must not be in the primary region.** The composite alarm that triggers stage 1, and the SNS topic that pages the approver, must live somewhere that survives. Easy to get wrong because CloudWatch alarms feel regional-and-therefore-fine.
11. **A freshly created Route 53 health check starts *healthy*.** Never create health checks as part of the failover. From [[aws-route53]].
12. **Two orchestrators, one outcome.** If you deploy SSM documents to the standby *and* a third region, two people can start two executions. Add a mutual-exclusion guard — a conditional write to a DynamoDB Global Table lock row as the first step — or accept that step 1's idempotency has to carry it.
13. **The runbook will drift from reality faster than the infrastructure does.** An SSM document in the same repo as the Terraform at least fails a plan when a referenced resource disappears. A Confluence page does not.

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| **Which RTO are we held to?** | 15 min from incident start | 15 min from decision to fail over | **Go and ask.** This is not an engineering decision and it cannot be defaulted. If A, automation becomes mandatory and the database decision changes to Aurora. If B, proceed as this vault assumes. **Blocks everything else in this note.** |
| **Failover trigger** | Fully automatic on a composite alarm | Human approval, automated execution | **B — manual trigger, automated execution.** AWS: *"Manually initiated failover is therefore often used. In this case, you should still automate the steps."* GitHub Oct 2018 is the cost of getting this wrong with an irreversible promotion. **Plus a stage-1 automatic fence**, which is reversible and shrinks the divergence window. |
| **Orchestrator** | ARC Region switch, $70/plan/month | Self-built SSM Automation documents, ~$0 | **A, with B built first as the fallback and the readable artefact.** Region switch is the only option where regional independence is AWS's problem, and it measures your RTO for you. Build the SSM version first anyway — it forces you to write the sequence down and gives you something to evaluate Region switch *against*. |
| **Orchestrator placement** | Standby region only | Standby + a third region | **B.** Two YAML deployments. Covers bilateral events and `us-east-1` global-service impairment. **Except the CA pair, where the third region is a residency question — see Open questions.** |
| **Ordering enforcement** | Document the order in the runbook | ARC **gating rule**: standby traffic control cannot go On unless the db-promoted control is On | **B.** Converts the most expensive ordering mistake in the sequence from a documentation problem into a structural impossibility. This is worth more than the routing control cluster's other features combined. |
| **Approver list size** | 2 named people | 5+ from the on-call rota | **B.** Two approvers is an unbounded RTO the first time both are unreachable. SSM caps at 10. |
| **Where the decision gate sits** | Before step 0 | Between step 1 (fence) and step 2 (promote) | **B.** Fencing is free and reversible; do it while the human is waking up. |

## Cost

| Item | Monthly |
|---|---|
| ARC Region switch plans, 3 pairs | **$210** |
| SSM Automation | $0 (no charge for automation steps in the free tier band; `executeScript` runs are billed as Automation steps beyond it — negligible at failover volumes) |
| Step Functions (if used instead) | ~$0 at this execution volume |
| CloudWatch composite alarms, ~6 | ~$3 |
| SNS | negligible |
| ARC routing control cluster (**only if** you take the routing-control path rather than Region switch's Route 53 health check block) | **$1,825** — see [[aws-route53#Cost]] |

**The lever:** if Region switch's Route 53 health check execution block removes the need for a routing control cluster, the orchestration layer costs **$210/month instead of $2,035/month**. That is the single largest cost question in the operations half of this programme and it is unresolved — [[aws-route53]] flags it as its highest-value open investigation, and this note concurs. Take it to [[cost-model]].

## Open questions

1. **Which RTO definition are we held to?** The blocking question. Everything in this note branches on it.
2. **What is our historical false-positive rate on a regional health signal?** Cannot make the automated-vs-manual call responsibly without it. Cheap to get: create the health checks, attach them to nothing, watch for a month ([[aws-route53]] migration step 6).
3. **How long does an RDS `promote-read-replica` actually take on our data volume?** [[aws-rds-postgres]] says no public benchmark exists. It is a 20-minute experiment and it determines whether Budget A holds.
4. **Does ARC Region switch have adequate Terraform provider coverage on the version we pin?** Announced December 2025. Verify against the actual provider version before committing.
5. **Where does the CA pair's third-region orchestrator live, and is that a residency problem?** Needs a legal answer, not an engineering one.
6. **Is break-glass long-lived credential storage acceptable to security?** If not, the failover path has an IdP dependency. Same question [[aws-route53]] raises; it needs one answer, not two.
7. **Are our SDKs/CLI configured for regional STS endpoints?** A one-line config that silently invalidates the whole plan during a `us-east-1` event.
8. **Who, by name, is authorised to accept up to 2 hours of data loss at 3am?** If this list does not exist, TTDecide is unbounded regardless of tooling.
9. **Is there a deploy-freeze mechanism we can call from a runbook?** Step 0 assumes one exists. If freezing the pipeline is a manual PR, it is not a 30-second step.

## Sources

- [Disaster Recovery of Workloads on AWS — Detection](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/detection.html) — the verbatim statement that detection, notification, escalation, discovery and declaration all fall inside the RTO; the deep-health-check guidance and the explicit false-alarm caution; the note about stakeholders declining to invoke DR. **The primary citation for the RTO-definition argument.**
- [Disaster Recovery of Workloads on AWS — Disaster recovery options in the cloud](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html) — the verbatim automated-vs-manual paragraph ("Automatically initiated failover ... should be used with caution ... manual initiation is like the push of a button"); "For maximum resiliency, you should use only data plane operations as part of your failover operation"; the warm standby definition; the warning that Auto Scaling is a control-plane dependency; ARC health checks as on/off switches; weighted-record flips being a control-plane operation.
- [AWS Fault Isolation Boundaries — Global services](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/global-services.html) — the verbatim list of partitional services and control-plane home regions (IAM/Organizations/Account Management in `us-east-1`, ARC and Network Manager in `us-west-2`); the edge-service list; **the anti-pattern list including "Creating or updating IAM resources ... during a failover"**; the STS global-endpoint default; break-glass user guidance; the recommendation to cache control-plane data in a data-plane-readable store.
- [GitHub — October 21 post-incident analysis](https://github.blog/news-insights/company-news/oct21-post-incident-analysis/) — 43-second partition → 24h11m degradation; Orchestrator establishing quorum and failing over across regions; the divergent-writes statement; "we were unable to fail the primary back over ... safely"; and the remediation "Adjust the configuration of Orchestrator to prevent the promotion of database primaries across regional boundaries." **The canonical automated-failover-false-positive case study.**
- [Netflix TechBlog — Project Nimble: Region Evacuation Reimagined](https://netflixtechblog.com/project-nimble-region-evacuation-reimagined-d0d0568254d4) — hot-standby instance pools per microservice, shadow clusters, evacuation reduced from ~50 minutes to ~8. Operator-initiated, regularly exercised.
- [AWS Architecture Blog — Validating multi-Region DR for Terraform Enterprise with AWS FIS](https://aws.amazon.com/blogs/architecture/validating-multi-region-dr-for-terraform-enterprise-with-aws-fis/) — a named company (athenahealth), an operator-triggered four-step runbook, **measured 12–14 minutes from trigger to full recovery**, RPO < 1 min, and the circular-dependency finding where failover scripts read Terraform state from an unreachable S3.
- [Orchestrate disaster recovery automation using Amazon Route 53 ARC and AWS Step Functions](https://aws.amazon.com/blogs/networking-and-content-delivery/orchestrate-disaster-recovery-automation-using-amazon-route-53-arc-and-aws-step-functions/) — state machines deployed in both regions; ARC endpoints and control ARNs stored in DynamoDB global tables so any region can execute; the control-plane/data-plane split on routing controls ("creation & deletion" vs "updates"); the ordered deactivate-primary → fail over DB → activate-standby sequence; and the honest admission that it does not handle data reconciliation.
- [`aws:approve` — Pause an automation for manual approval](https://docs.aws.amazon.com/systems-manager/latest/userguide/automation-action-approve.html) — full input surface, the **7-day default / 30-day max timeout**, the **"doesn't support multi-account and Region automations"** limitation, the `Automation`-prefix requirement on the SNS topic name, the 10-approver cap, and the `ApprovalStatus` / `ApproverDecisions` outputs.
- [Region switch in ARC](https://docs.aws.amazon.com/r53recovery/latest/dg/region-switch.html) — plans/workflows/steps/execution blocks; CloudWatch-alarm triggers; cross-account support; and the decisive line: **"A data plane in each AWS Region, so that you can execute your Region switch plan without taking a dependency on the Region that you're deactivating."**
- [Region switch components](https://docs.aws.amazon.com/r53recovery/latest/dg/components-rs.html) — graceful vs ungraceful configurations; triggers vs application health alarms; post-recovery workflows running "in the Region that was previously impaired" and supporting an RDS Create Cross-Region Replica block; parent/child plans.
- [Best practices for Region switch in ARC](https://docs.aws.amazon.com/r53recovery/latest/dg/best-practices.region-switch.html) — break-glass long-lived credentials; TTL 60–120 s; **"Region switch does not guarantee that the desired compute capacity with be attained"** and the recommendation to reserve capacity; use the data plane, not the console; test regularly; and the comparison of Region switch DNS failover against Route 53 accelerated recovery's 60-minute target.
- [Amazon Application Recovery Controller pricing](https://aws.amazon.com/application-recovery-controller/pricing/) — $70/plan/month for Region switch; $2.50/hour per routing control cluster.
- [REL13-BP03 Test disaster recovery implementation](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_dr_tested.html) — "the only error recovery that works is the path you test frequently"; and note that *"Never exercise failovers in production"* is listed as an **anti-pattern**, not advice. Expanded in [[dr-testing-and-gamedays]].

## Related notes

[[aws-route53]] · [[aws-rds-postgres]] · [[aws-aurora-global-database]] · [[aws-eks]] · [[split-brain-and-fencing]] · [[dr-testing-and-gamedays]] · [[failover-runbook-template]] · [[failback]] · [[observability-multi-region]] · [[aws-iam]] · [[aws-lambda]] · [[aws-sqs]] · [[aws-eventbridge]] · [[messaging-in-flight-data-loss]] · [[cost-model]] · [[open-decisions]] · [[rpo-rto-analysis]] · [[region-pair-selection]] · [[data-residency]] · [[security-posture-of-the-standby]] · [[terraform-repo-structure]]
