---
title: Messaging — What Is Actually Lost At Failover
service: sqs, sns, eventbridge, lambda
tags: [service, multi-region, messaging, rpo, analysis, data-loss]
status: researched
replication: none
rpo_achievable: "seconds of pending work, on a healthy queue — but see the argument below about whether that counts against RPO at all"
rto_achievable: "N/A — this note is about RPO, not RTO"
meets_targets: conditional — depends entirely on the acknowledgement boundary, argued below
updated: 2026-09-16
---

# Messaging — What Is Actually Lost At Failover

> This is not a service note. It is the argument that decides how much
> engineering [[aws-sqs]], [[aws-sns]] and [[aws-eventbridge]] deserve. If the
> conclusion is "unprocessed queue messages are not RPO-relevant", the messaging
> workstream is a week of Terraform. If it is "they are", it is a quarter of
> application work. Nobody should build either until this question has an
> agreed answer.
>
> Template sections that only apply to a provisioned service (warm standby
> shape, migration path, failback, cost) are marked N/A with a line on why.

## TL;DR

- **The crux: an unprocessed queue message is usually not "data" for RPO
  purposes — it is *pending work*. But the distinction is not about queues, it's
  about acknowledgement.** The test is: *did we tell someone outside the system
  that we had accepted this?* If yes, the message is the only record of a
  promise, and losing it is a data loss in the fullest sense. If no, it is a
  work item that can be re-derived or re-requested, and it should not consume
  RPO budget.
- **The numbers make this mostly academic, and that is the most useful finding
  in this note.** A healthy queue holds *seconds* of work. RPO is 2 hours. Even
  the pessimistic case — a queue that has been quietly backing up for an hour
  before the region dies — is inside budget. **SQS is not the thing that will
  blow the 2h RPO.** [[aws-rds-postgres]] and [[aws-elasticache-redis]] are far
  more credible threats and should get the attention.
- **Where it stops being academic is the acknowledgement boundary at the front
  door.** An API that returns `202 Accepted` after `SendMessage` and before any
  durable write has made the queue the system of record for that request. That
  is a real, unbudgeted RPO exposure, and it is fixed by moving the durable
  write before the acknowledgement — not by replicating the queue.
- **Standby consumers: recommendation is to have them deployed, running, and
  polling — but with their SQS event source mappings / consumer loops
  *disabled*.** Warm process, cold subscription. That gets the RTO benefit of a
  warm fleet without the double-processing risk of a live subscription, and AWS
  now ships a purpose-built primitive to flip the subscription during failover
  (ARC Region Switch's Lambda ESM execution block). Both branches argued in
  full below.
- **The thing that will bite:** it is not the stranded messages. It is the
  *un-fenced primary consumers* coming back online after the region recovers and
  processing stale work into a system that has already moved on, in parallel
  with the standby. See [[split-brain-and-fencing]] — that is the incident that
  costs money, not the one this note is nominally about.

---

## The crux: do unprocessed messages count as "data" for RPO?

### What AWS's definition actually says

AWS's Well-Architected Reliability Pillar defines RPO as:

> *"RPO is the maximum acceptable amount of time since the last data recovery
> point. This determines what is considered an acceptable loss of data between
> the last recovery point and the interruption of service."*
> — [Disaster Recovery (DR) objectives](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/disaster-recovery-dr-objectives.html)

Read it carefully, because the wording is doing something specific. RPO is
framed around a **recovery point** — a snapshot, a replication checkpoint, a
transaction log position. It is a property of a *store*. The mental model is a
database and its replica: the replica is behind by some amount of time, and that
amount is the RPO.

A queue does not have a recovery point. It has a *state*, and that state is
"work that has been handed over but not yet done". The definition does not
cleanly extend to it, and pretending it does — "the queue is a store, therefore
its contents are data, therefore RPO applies" — smuggles in an assumption that
deserves to be argued rather than assumed.

Note also what AWS's own guidance says about this specific case: the DR workshop
tells you to *"account for the possible loss of messages in transit"* and to
make consumers idempotent
([source](https://disaster-recovery.workshop.aws/en/services/app_integration/sqs/active-active.html)).
That is not the language you use about something you consider protected data.
**AWS's prescriptive guidance for a multi-region messaging architecture is to
plan for message loss, not to prevent it.** That is a strong signal about the
intended mental model, and it is worth putting in front of anyone who wants to
build a queue replicator.

### The argument that they are NOT data

**A queue message is an instruction, not a record.** "Resize this image."
"Send this email." "Recalculate this total." The authoritative fact — that an
image was uploaded, that a user requested an email, that a total needs
recalculating — lives in a database that *does* have a recovery point and *is*
covered by the RPO. The message is a transient pointer to that fact.

If the database says `image_id=123, thumbnail_status=PENDING`, then the
instruction "resize image 123" is fully recoverable by querying the database. It
was never unique information. Losing it costs you a query, not a fact.

Under this view, counting queue messages against RPO **double-counts**: you'd be
budgeting for the loss of the record *and* the loss of the pointer to the
record, when losing the pointer alone costs nothing.

There is a second, subtler argument. RPO is a budget you spend on replication
engineering. Every pound spent protecting pointers is a pound not spent
protecting records. **If you allocate RPO budget to queue contents, you will
under-invest in the database replication that actually determines whether the
business survives.** Misallocation is a real cost, not a philosophical one.

### The argument that they ARE data

Sometimes the message *is* the record, and the honest version of this note has
to concede how often that's true in practice.

1. **The API that acknowledged before it persisted.** A common, entirely
   reasonable pattern: `POST /orders` → validate → `SendMessage` → return
   `202 Accepted` with a tracking ID. At that instant, the *only* durable record
   of that order is a message in an SQS queue. The customer has a tracking ID and
   a screenshot. If the region dies, the order does not exist anywhere. **The
   queue is the system of record for the window between acknowledgement and the
   consumer's first write.** That is unambiguously data loss and it is
   unambiguously in scope for RPO.

2. **The message carries information not present anywhere else.** Enriched
   payloads, computed values, upstream third-party webhook bodies, event bodies
   with fields that the producer didn't persist. Re-derivation is impossible
   because the inputs are gone. This is common with inbound webhooks from
   payment providers and shipping carriers, where the provider will not resend.

3. **The DLQ.** A dead-letter queue is by construction full of messages that
   failed and that someone intended to investigate. There is no re-derivation
   path — that's *why* they're in a DLQ. And they can't be redriven cross-region
   ([[aws-sqs]] covers the `StartMessageMoveTask` same-region restriction).
   **DLQ contents are the most data-like, least recoverable messages in the
   estate**, and they are the ones nobody thinks about when discussing queue DR.

4. **Legal and regulatory records.** An audit event sitting in a queue is a
   record you are obliged to retain, whether or not it has been "processed".
   See [[data-residency]] and [[regulatory-drivers]].

### The test that actually resolves it

Not "is it a queue?" but:

> **Has the system acknowledged receipt of this work to anything it cannot ask
> again?**

Three outcomes, and they should be recorded **per queue**, not per estate:

| Category | Description | RPO-relevant? | What to do |
|---|---|---|---|
| **Re-derivable** | Content is reconstructible from committed, replicated state | **No** | Empty mirror queue. Write a reconciliation query. Option (a)+(d) in [[aws-sqs]]. |
| **Acknowledged & unique** | We told an external party we had it, and it exists nowhere else | **Yes** | Fix the architecture: persist *before* acknowledging. Replication is the wrong fix. |
| **Ephemeral signal** | Cache invalidations, metrics, "poke the scheduler" messages | **No** — arguably not even work | Ignore entirely. Losing them is self-healing. |

The middle row is the whole problem, and **the right response to it is not to
replicate the queue.** It is to move the durable write in front of the
acknowledgement:

```
  Before:  validate → SendMessage → 202 Accepted
                                    ^ queue is now the system of record

  After:   validate → write to DynamoDB Global Table / Aurora → 202 Accepted
                    → SendMessage (fire and forget, or via outbox)
                                    ^ queue is now just a pointer
```

This is the transactional outbox with the region boundary drawn around it. It
costs one database write on the request path and it **moves the message from row
2 to row 1** — converting a hard replication problem into a solved one, because
the database layer's cross-region replication is already on the roadmap
([[aws-dynamodb]], [[aws-aurora-global-database]]).

**That is this note's central recommendation.** Do not build a queue replicator.
Find the places where the queue is the system of record and stop them being the
system of record. There are probably fewer than ten of them, they are probably
all at the API edge, and fixing each is a day of work.

### The verdict

**Unprocessed queue messages are not, in general, data for RPO purposes — with a
specific and enumerable set of exceptions, each of which is better fixed by
moving the durable write earlier than by replicating the queue.**

State it that way in the DR policy document, with the three-row table above, and
require every queue to be classified. The classification exercise is more
valuable than any replication mechanism this vault could recommend: it will
surface the two or three genuinely dangerous queues, and it will let you say
"everything else is an empty mirror" with a straight face and a written
justification.

---

## Relating this to the 2-hour RPO specifically

Here is the part that should calm everyone down.

**RPO is measured in time. Queue exposure is also measurable in time —
`ApproximateAgeOfOldestMessage` is literally "how far back does the unprocessed
work go".** The two are directly comparable without any modelling. That is a
genuinely convenient property and it means you can answer "are we inside RPO?"
for messaging with a single CloudWatch metric.

| Queue state | `ApproximateAgeOfOldestMessage` | Fraction of the 2h budget |
|---|---|---|
| Healthy, well-scaled consumers | 0–5 s | **< 0.07%** |
| Busy but coping | 30 s | 0.4% |
| Consumers degraded, alarms firing | 15 min | 12.5% |
| Consumers down for an hour before the region died | 60 min | 50% |
| Consumers down since yesterday and nobody noticed | 24 h | **Blown — 12× over** |

Read the table as a statement about *monitoring*, not about *replication*. The
only rows that threaten the RPO are rows where the queue was already unhealthy —
which means:

> **The most effective RPO control for messaging is not replication. It is an
> alarm on `ApproximateAgeOfOldestMessage`.**

[[aws-sqs]] sets that alarm's threshold at 900 seconds — the RTO — on the logic
that if the oldest unprocessed message is older than your entire failover
budget, your live RPO exposure has stopped being "seconds" and a human needs to
know. That threshold is arbitrary but defensible, and far more useful than any
number derived from the 2h RPO (2h is so loose that an alarm set there would
fire only after the damage was done).

**Three caveats before anyone gets comfortable:**

1. **The outage that makes you fail over is correlated with the backlog.** A
   degrading region backs queues up *first* and goes dark *second*. So your
   exposure at the moment of failover is systematically worse than your steady
   -state measurement. Use the p99 over 90 days, not the mean over a week, and
   assume the real number is worse than the p99.
2. **A stranded message is not necessarily a lost message.** With 14-day
   retention (which [[aws-sqs]] recommends everywhere), a message stranded by a
   region outage is recoverable for two weeks. Most regional events end in hours.
   **For most incidents, option (a)'s "loss" is actually a delay** — the work is
   late, not gone. That is a latency failure, not a durability failure, and it is
   a different and much easier conversation with the business. It is also
   something the RPO metric, as defined, does not capture at all.
3. **RPO 2h is loose enough that it is the wrong tool here.** With a 2h budget,
   almost nothing about messaging is binding. The real constraints on messaging
   are RTO (can the standby consume fast enough?) and correctness (are consumers
   idempotent?). Do not let a comfortable RPO number create false confidence
   about a messaging tier that has never been failed over.

---

## What else is actually lost at failover

Queues get the attention because they're visible in the console. They are not
the largest exposure. The honest inventory, roughly in order of how much work is
at risk:

| Thing in flight | Where it lives | Lost at failover? | Notes |
|---|---|---|---|
| **HTTP requests being served** | ALB/target connections | **Yes, all of them** | Almost certainly a bigger volume than the queue backlog. Clients retry; that's the mitigation. See [[aws-alb-nlb]]. |
| **Database writes not yet replicated** | RDS/Aurora replica lag | Yes, up to the lag | **The real RPO consumer.** See [[aws-rds-postgres]], [[aws-aurora-global-database]]. |
| **In-memory session/cache state** | ElastiCache | Yes, unless replicated | See [[aws-elasticache-redis]]. Often silently load-bearing. |
| **Unprocessed SQS messages** | SQS | Yes, until the region returns | This note. Seconds of work, usually. |
| **In-flight (received, not deleted) SQS messages** | SQS | Yes — and they were *mid-processing* | Worse than queued messages: a consumer was partway through. Half-done work is why you need idempotency, not just replay. |
| **SNS messages in retry backoff** | SNS internal | **Mostly no — see below** | For SQS/Lambda destinations SNS retries up to 100,015 times over 23 days. A message queued for a dead endpoint survives an astonishingly long outage. |
| **EventBridge events between `PutEvents` and target delivery** | EventBridge internal | Partly | EventBridge retries target delivery for up to 24 hours. With an **archive**, the event is recoverable and replayable — see [[aws-eventbridge]]. |
| **Lambda async invocation queue** | Lambda internal | Yes | Lambda's internal async queue is regional and not inspectable. You cannot drain it, count it, or replay it. **It is the least visible loss in the estate.** |
| **Kinesis / DynamoDB Streams positions** | Regional | Yes | Out of scope here; note that the ARC Region Switch ESM block covers these too. |
| **CloudWatch metrics/logs not yet flushed** | Agent buffers | Yes | Matters because it means **you lose the evidence of what you lost.** See [[observability-multi-region]]. |

Two things fall out of this table that are worth saying loudly:

**First — the SNS retry window is a hidden safety net.** SNS retries delivery to
SQS and Lambda destinations up to 100,015 times over 23 days
([docs](https://docs.aws.amazon.com/sns/latest/dg/sns-message-delivery-retries.html)).
If your architecture is producer → SNS → SQS, and the SQS queue's region is
unavailable, SNS holds and retries for **longer than SQS's own maximum
retention**. An SNS-fronted queue is meaningfully more durable across a regional
event than a directly-written queue, for free, with no replication. That is an
under-appreciated argument for putting a topic in front of queues even when you
don't need fan-out. [[aws-sns]] develops this.

**Second — record what you lost, before you lose the ability to.** If your
metrics pipeline dies with the region, you cannot answer "how many orders were
stranded?" at the post-incident review. Ship SQS depth/age metrics
cross-region continuously. This is a small piece of work that pays out entirely
during the one hour of the year you need it, and it belongs in
[[observability-multi-region]].

---

## The standby-consumer question

Should the standby region's consumers be running and polling the (empty) standby
queue while the primary is healthy?

This is a real fork with real money and real risk on both sides, and it
interacts with everything in [[split-brain-and-fencing]].

### Branch A — standby consumers run and poll continuously

Deployments at some non-zero replica count in `eu-west-2`, Lambda event source
mappings enabled, long-polling an empty queue forever.

**For:**

- **Warm is what the 15-minute RTO demands.** A consumer fleet that has to cold
  -start — pull images, JIT-warm, establish connection pools, populate caches,
  scale out from zero — can easily eat the entire RTO on its own. A fleet that
  is already running and already connected starts draining the instant the first
  message arrives. **This is the single strongest argument and it maps directly
  onto the brief's constraint** that nothing may be provisioned from cold at
  failover time.
- **It is continuously self-testing.** A consumer polling an empty queue is
  exercising: IAM permissions, the queue policy, KMS decrypt grants, VPC
  endpoints and security groups, DNS, credential rotation. **Every one of those
  is a thing that silently breaks and is invisible on an unused queue.** A
  standby you never touch is a standby whose IAM policy drifted six months ago.
  This benefit is easy to undervalue and it is large.
- **It's nearly free.** Long polling with `receive_wait_time_seconds = 20` is
  3 `ReceiveMessage` calls per minute per poller — roughly 130,000 requests per
  month, against a 1-million-request free tier. AWS's own cost guidance
  identifies empty receives as the main driver of surprise SQS bills and long
  polling as the fix, cutting empty receives by 50–90%
  ([re:Post](https://repost.aws/knowledge-center/sqs-high-charges)). *But do the
  multiplication before calling it free:* requests × queues × pollers × regions.
  Forty queues with ten pollers each is 52 million requests a month, which is no
  longer free — it's a small but real line item, and it is pure waste when the
  queue is permanently empty.
- **Consumers stay observably healthy.** You get real metrics from the standby
  fleet rather than hoping.

**Against:**

- **Double-processing, if both regions are ever briefly live.** This is the
  serious one. With option (b) SNS fan-out ([[aws-sqs]]), *both* queues receive
  every message, so live standby consumers process everything twice, always —
  not just during an incident. Even without fan-out, a partial failover, a
  botched failback, or a producer that was never repointed puts messages into
  both queues while both are being consumed.
- **It makes the standby capable of side effects at all times.** A running
  consumer with production credentials can charge cards, send emails and mutate
  the primary's database (via Global Tables, which replicate both ways). The
  blast radius of a misrouted message is "a real customer got a real email",
  not "a message sat in a queue". See [[security-posture-of-the-standby]].
- **Compute cost.** Not the SQS requests — the pods. In EKS that's nodes, and
  nodes are not free. This is typically the largest single line in the standby
  cost model and it belongs in [[cost-model]], not here.
- **Failing consumers in the standby generate alarm noise** that on-call learns
  to ignore, which is how you arrive at a standby that has been broken for weeks.

### Branch B — standby consumers scaled to zero

Deployments at zero replicas, event source mappings disabled. Scale up at
failover.

**For:**

- **Double-processing is structurally impossible.** Not "prevented by
  configuration" — impossible, because nothing is reading. This is the strongest
  possible form of fencing and it needs no coordination, no leader election, no
  Route 53 health check to be correct. **For a payments or fulfilment workload
  this can be the whole argument.**
- **Cheapest.** No compute, no requests.
- **Clean, auditable posture:** the standby cannot act. Easy to explain to a
  security reviewer.

**Against:**

- **It threatens the 15-minute RTO** and this is not hypothetical. Scaling a
  deployment from 0 to N in EKS means: scheduler decisions, possibly Cluster
  Autoscaler / Karpenter provisioning nodes (minutes), image pulls (minutes on a
  cold node, unless [[aws-ecr]] pull-through caching and pre-warmed nodes are in
  place), container start, health-check grace periods, connection pool
  establishment. **Five to fifteen minutes is a realistic range, and your entire
  RTO is fifteen.** The queue was instant; the consumers were not.
- **Everything is untested.** The IAM/KMS/VPC/DNS chain that Branch A exercises
  continuously is exercised for the first time during the incident. This is where
  "the standby queue was encrypted with the primary's KMS key" is discovered.
- **Lambda event source mappings are not instant to enable.** Enabling an ESM is
  a control-plane operation with propagation delay. Usually under a minute, but
  it is not zero and it is not something you want to discover the timing of at
  3am.

### Recommendation

**Neither pure branch. Run the consumers, disable the subscriptions.**

Concretely:

| Layer | State while primary is healthy | Why |
|---|---|---|
| EKS deployments / pods | **Running**, at reduced replica count (not zero) | Images pulled, nodes warm, connection pools alive, RTO risk removed |
| SQS polling loop / Lambda ESM | **Disabled** | Nothing is consumed; double-processing impossible |
| A dedicated low-privilege canary consumer | **Running and polling** a dedicated `*-canary` queue | Continuously proves IAM + KMS + VPC + DNS without touching real messages |
| Standby DLQ alarms | **Active** | Free split-brain detector |

This gets Branch A's warmth and continuous validation *and* Branch B's absolute
fencing. The cost is the standby compute, which you are paying anyway for a warm
standby posture — the brief's target is explicitly warm standby, not pilot
light.

The canary queue deserves emphasis: **a real consumer, polling a real queue, with
the real IAM role and the real KMS key, on a queue that only ever receives
synthetic messages.** Publish a heartbeat to it every five minutes from the
standby region and alarm if it isn't consumed. That single construct catches
essentially every silent-standby-rot failure mode — expired policy, revoked KMS
grant, deleted VPC endpoint, rotated credential — and it cannot process a real
message because real messages never go there. It is perhaps fifty lines of
Terraform and it is the highest-leverage thing in this note.

**The flip at failover** is then a single, well-defined, automatable action:
enable the event source mappings / start the polling loops. AWS ships a
primitive for exactly this — **ARC Region Switch's Lambda event source mapping
execution block**, which enables and disables ESMs for SQS, Kinesis, DynamoDB
Streams and MSK as ordered steps in a failover plan, including an "ungraceful"
mode that skips the disable step when the primary is unreachable
([announcement](https://aws.amazon.com/about-aws/whats-new/2026/05/region-switch-lambda-esm-execution-block/)).
AWS's own framing of the problem is that these mappings *"must be toggled during
failover to avoid duplicate processing — a manual, error-prone step."* That is
this exact decision, and it is worth a spike in [[failover-orchestration]]
before anyone writes a bespoke script.

**Where the recommendation flips:** if the workload's consumers are on EKS with
Karpenter, a warm node pool, and a proven sub-two-minute scale-from-zero, then
Branch B (true zero) becomes viable and is cheaper and safer. Measure the actual
0→N time in a game day before assuming either way. **The recommendation above is
a hedge against an unmeasured scale-up time; measure it and you may not need the
hedge.** See [[dr-testing-and-gamedays]].

---

## The failure mode that actually costs money

It is not stranded messages. It is this:

1. Primary degrades. You fail over. Standby consumers start.
2. Standby processes work, writes to the database (which replicates both ways
   via Global Tables).
3. **Primary region recovers.** EKS nodes rejoin. Deployments are still at their
   pre-incident replica count. Lambda ESMs are still enabled — nobody disabled
   them, because the region was unreachable when the runbook said to.
4. Primary consumers start draining the stranded backlog: hours-old instructions,
   processed into a system that has already moved on.
5. Both regions are now consuming and writing. Standard queues are at-least-once
   by design. Consumers that are merely "mostly idempotent" now produce duplicate
   charges, duplicate emails, duplicate shipments.

**This happens after the incident is declared over**, which is when attention is
lowest. It is the single most expensive messaging failure mode in an
active/passive design and it has nothing to do with replication.

Requirements it imposes:

- **Fencing must survive a region recovery.** A runtime feature flag stored in
  the primary region's Parameter Store is not fencing — it comes back with the
  region. The fence must live in the surviving region, or in a global control
  plane, or be a control-plane state (ESM disabled) that persists across node
  restarts. This is [[split-brain-and-fencing]]'s core problem and this note
  simply hands it over.
- **Idempotency must be real.** Not "we check a flag at the end". A durable
  idempotency key checked-and-claimed atomically before any side effect.
- **The runbook must have an explicit "primary has returned" step** that is
  separate from, and later than, the "we have failed over" step — with fencing
  verification as its first action. Most runbooks stop at "traffic is serving
  from the standby" and that is exactly one step too early.

---

## Sections from the template that don't apply

- **Warm standby shape** — N/A here; this note is an analysis, not a resource.
  The standby shape for queues is in [[aws-sqs]]; the consumer-fleet shape is the
  standby-consumer decision above.
- **Terraform implementation** — N/A. The only Terraform implied by this note is
  the canary queue + heartbeat, which belongs in the [[aws-sqs]] module, and the
  `ApproximateAgeOfOldestMessage` alarm, which is already there.
- **Migration path** — N/A. Nothing to migrate. The work implied here is
  *classification* (which queues are re-derivable?) and, where classification
  comes back "acknowledged & unique", an application change to persist before
  acknowledging.
- **Failback** — covered per-service in [[aws-sqs]] and [[aws-sns]]. The
  messaging-specific failback risk is the un-fenced-primary-consumers scenario
  above, which is [[split-brain-and-fencing]].
- **Cost** — the only cost decision here is the standby-consumer question, and
  its cost is compute (EKS nodes), not messaging. Belongs in [[cost-model]].

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Do unprocessed queue messages count against the 2h RPO? | Yes — budget for them, build replication | No — they are pending work, not committed state | **No, with exceptions.** Classify every queue into re-derivable / acknowledged-&-unique / ephemeral. Only the middle category is RPO-relevant, and it should be fixed at the API edge, not by replication. |
| How to fix "acknowledged & unique" queues | Replicate the queue (SNS fan-out) | Persist to a replicated store *before* acknowledging | **B.** Cheaper, removes the exposure rather than mitigating it, and reuses database replication that's already on the roadmap. |
| Standby consumers | Running and polling | Scaled to zero | **Neither: running but with subscriptions disabled**, plus a dedicated canary consumer on a canary queue. Revisit if measured 0→N scale-up is under two minutes. |
| How to flip consumers at failover | Bespoke script | ARC Region Switch ESM execution block | **Evaluate ARC first** — AWS built it for this exact problem, including an ungraceful mode. Decide in [[failover-orchestration]]. |
| Primary RPO control for messaging | Replicate queues | Alarm on `ApproximateAgeOfOldestMessage` | **The alarm.** It is the only thing that catches the scenarios that actually threaten the 2h RPO. |
| Post-recovery stranded messages | Drain automatically | Archive, then decide per queue | **Archive by default.** Replaying stale instructions is how a closed incident reopens. |

## Open questions

1. **Which queues sit behind an API that acknowledges before persisting?** This
   is *the* question. It is answerable by reading the handful of endpoints that
   return `202`. Until it's answered, nobody knows the estate's true messaging
   RPO exposure.
2. **Are consumers genuinely idempotent, per workload?** Everything in this note
   — every branch of every fork — degrades badly if the answer is no. And the
   honest answer is usually "some are, and nobody knows which".
3. **What is the measured 0→N scale-up time for the standby consumer fleet?**
   It decides the standby-consumer question. Measure it in a game day; don't
   estimate it.
4. **Is the RTO clock from incident start or from decision-to-fail-over?** Flagged
   in `CLAUDE.md` as an open thread for the whole vault, and it bites hardest
   here: if the clock starts at incident start, a cold consumer fleet is
   disqualified outright and Branch B is off the table.
5. **Does the DR policy document define RPO in terms of "data" or "state"?** If
   it was written against a database-shaped mental model — which the AWS
   definition is — then this note's argument needs to be added to it explicitly
   rather than assumed.
6. **Who signs off on the per-queue classification?** It is a business judgement
   ("is a lost cache-invalidation acceptable?"), not an infrastructure one, and
   without a named owner the classification exercise will stall.

## Sources

- [Disaster Recovery (DR) objectives — AWS Well-Architected Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/disaster-recovery-dr-objectives.html)
  — AWS's exact RPO/RTO definitions. The "last data recovery point" framing is
  the textual basis for this note's argument that the definition is
  store-shaped and doesn't cleanly cover queues.
- [REL13-BP01 Define recovery objectives for downtime and data loss](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_objective_defined_recovery.html)
  — the best-practice guidance for setting RPO/RTO per workload, which supports
  per-queue classification over a single estate-wide number.
- [Amazon SQS — Active-Active, AWS DR Workshop](https://disaster-recovery.workshop.aws/en/services/app_integration/sqs/active-active.html)
  — AWS telling you to plan for message loss and build idempotent consumers.
  The clearest signal that AWS does not treat queue contents as protected data.
- [Amazon SNS message delivery retries](https://docs.aws.amazon.com/sns/latest/dg/sns-message-delivery-retries.html)
  — the 100,015 retries over 23 days figure for SQS/Lambda destinations, which
  is why SNS-fronted queues survive regional events better than direct writes.
- [Amazon SQS standard queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues.html)
  — at-least-once delivery and possible out-of-order delivery; the basis for the
  duplicate-processing risk in the both-regions-live scenario.
- [Understand Amazon SQS billing and how to reduce costs — AWS re:Post](https://repost.aws/knowledge-center/sqs-high-charges)
  — empty receives as the dominant cost driver, and long polling as the
  mitigation. Underpins the "polling an empty queue is nearly free — but do the
  multiplication" analysis.
- [How Lambda processes records from stream and queue-based event sources](https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventsourcemapping.html)
  — event source mappings are at-least-once and AWS explicitly recommends
  idempotent function code.
- [ARC Region Switch adds Lambda event source mapping execution block](https://aws.amazon.com/about-aws/whats-new/2026/05/region-switch-lambda-esm-execution-block/)
  — AWS's own statement that ESMs "must be toggled during failover to avoid
  duplicate processing", and the managed primitive that does it, including
  ungraceful mode. The strongest external validation of this note's
  standby-consumer recommendation.

### Searched for and did not find

- **An authoritative statement — from AWS or from a recognised DR standard —
  on whether in-flight queue messages count toward RPO.** No public source
  addresses the question directly. Industry RPO definitions (F5, TechTarget,
  Imperva, Veeam and others all surfaced) are uniformly framed around backups
  and database recovery points and simply do not contemplate queues. **The
  argument in this note is therefore reasoned from the definitions, not cited
  from a source that settles it** — which is exactly why it needs an explicit
  internal decision rather than an appeal to authority.
- **A published postmortem of an organisation losing queued messages in a
  regional failover.** No public example found.
