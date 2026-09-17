---
title: Amazon SQS — Multi-Region
service: sqs
tags: [service, multi-region, sqs, messaging, eventing]
status: researched
replication: none
rpo_achievable: "0 for the queue *resource*; for messages in the queue, loss is bounded by queue latency (seconds), not by the 2h RPO"
rto_achievable: "< 1 min — an empty mirror queue is instant to create and free to leave sitting"
meets_targets: yes
updated: 2026-09-16
---

# Amazon SQS — Multi-Region

> **The uncomfortable truth this note exists to state plainly.**
>
> **SQS queues do not replicate. There is no cross-region SQS replication — no
> API, no console toggle, no Terraform argument, no AWS-managed background
> process. Messages sitting in an `eu-west-1` queue when `eu-west-1` fails are
> unreachable until `eu-west-1` comes back.** Not "delayed". Not "eventually
> consistent". Unreachable. The queue URL resolves to a regional endpoint that
> is down.
>
> Every design in this note is a way of arranging your system so that this fact
> doesn't matter very much. None of them make it untrue.

This is corroborated by AWS's own DR workshop, which says of message queues:
*"Features such as message queues have no auto-replication feature of messages
to another region"* and instructs you to *"account for the possible loss of
messages in transit"* and to make consumers idempotent
([disaster-recovery.workshop.aws](https://disaster-recovery.workshop.aws/en/services/app_integration/sqs/active-active.html)).

There is no AWS "what's new" post announcing native SQS cross-region
replication, and searching for one finds only third-party articles describing
DIY approaches. **If you read a blog that describes "SQS cross-region
replication" as a feature you configure on the queue, it is wrong.** (Several
of the top search results for that phrase are AI-generated SEO pages that
describe a feature that does not exist. Do not let one of them into a design
review.)

## TL;DR

- **SQS is regional and has zero native replication.** The mirror is a second,
  empty queue you create with Terraform. It costs nothing while empty.
- **That is almost always the right answer.** The exposure is not "everything in
  the queue" — it is "everything that arrived and hasn't been consumed yet",
  which on a healthy queue is *seconds* of traffic, not hours. Measure
  `ApproximateAgeOfOldestMessage` on the real queue and you will find your
  actual RPO exposure is far better than 2h. **Queue backlog is not two hours of
  business data; it's two seconds of it.** See [[messaging-in-flight-data-loss]]
  for the full argument, including whether unprocessed messages count as "data"
  at all.
- **If you need better than that, the only good primitive is SNS.** A topic in
  the primary can deliver natively to an SQS queue in the standby — see
  [[aws-sns]]. Fan-out at publish time is the one option that doesn't require
  you to write and operate code. The drain-forwarder Lambda is the option
  everyone reaches for first and it is the worst of the four.
- **FIFO does not survive a cross-region relay.** Deduplication scope is
  per-queue. Relaying a FIFO message into another region gives it a fresh
  dedup window and a fresh ordering context. Anything you tell your auditors
  about exactly-once stops being true at the region boundary. Say this out loud
  before anyone builds a relay.
- **The thing that will bite:** the DLQ. `RedrivePolicy` requires the DLQ to be
  in the *same account and region* as the source queue, and
  `StartMessageMoveTask` (the redrive API) is same-region only. So the standby
  needs its own full DLQ topology — and the primary's DLQ contents, which are by
  definition the messages you most wanted to keep, are the least recoverable
  thing in the estate.

## Does this service cross regions at all?

No, in every sense that matters.

| Thing | Regional or global? | Consequence |
|---|---|---|
| Queue URL | Regional — `https://sqs.<region>.amazonaws.com/<acct>/<name>` | The URL is the identity. A standby queue has a different URL. |
| Queue ARN | Regional — `arn:aws:sqs:<region>:<acct>:<name>` | Any IAM policy, SNS subscription or EventBridge target that names a queue names a region. |
| Messages | Regional, replicated across AZs **within** the region only | AZ-redundant, region-fragile. |
| Queue name | Unique per account **per region** | You *may* use the same name in both regions. See the naming decision below. |
| `RedrivePolicy` / DLQ | Same account **and region** as source | No cross-region DLQ. |
| `StartMessageMoveTask` | Same region | No cross-region redrive. |

SQS gives you 11-nines-style durability *within* a region by storing each
message redundantly across multiple Availability Zones. That is an excellent
answer to "a data centre burned down" and no answer at all to "the region's
control plane is returning 500s". The AZ redundancy is why people mistake SQS
for durable-in-the-general-sense; it is durable in exactly one region.

The one genuinely cross-region thing you can do *to* an SQS queue is have
something else write to it: SNS can deliver cross-region, EventBridge can route
cross-region to a bus that then targets a local queue, and any Lambda or pod
with credentials can call `SendMessage` against another region's endpoint. All
of these are **producers reaching in**, not replication.

> **Terraform note.** There is an open provider issue —
> [hashicorp/terraform-provider-aws#44777](https://github.com/hashicorp/terraform-provider-aws/issues/44777)
> — about not being able to import an SQS queue from a region other than the
> provider's. This is a symptom of the same regionality: the queue URL embeds
> the region, so cross-region addressing is awkward everywhere, including in the
> tooling.

## Replication / mirroring options

Four real options. They are not equally good and the note will not pretend
otherwise, but each has a workload shape where it is correct.

---

### Option (a) — Empty mirror queue in the standby, accept in-flight loss

Create the queue in `eu-west-2` with Terraform. Leave it empty. Nothing consumes
it while the primary is healthy (or a scaled-to-near-zero consumer polls it —
see [[messaging-in-flight-data-loss]] for that sub-decision). At failover,
producers start publishing to the standby queue and the standby consumers drain
it. Whatever was in the primary queue is stranded until the primary returns.

**What you actually lose.** This is the number the whole decision turns on, and
almost nobody computes it. The exposure is:

```
messages_at_risk  ≈  arrival_rate (msg/s)  ×  time_in_queue (s)
```

`time_in_queue` is exactly what `ApproximateAgeOfOldestMessage` measures. On a
healthy, properly-scaled consumer fleet that number is small — often under a
second, typically a few seconds. Worked examples, to make it concrete:

| Arrival rate | Steady-state queue age | Messages stranded at failover |
|---|---|---|
| 10 msg/s | 1 s | ~10 |
| 100 msg/s | 2 s | ~200 |
| 1,000 msg/s | 5 s | ~5,000 |
| 100 msg/s | **300 s (consumers already struggling)** | ~30,000 |

The bottom row is the real risk, and it is not a replication problem — it is a
*capacity* problem that becomes a data-loss problem at failover. The same
incident that makes you fail over (region degradation) is likely to have already
backed the queue up, so the number you lose in an outage is **not** the number
you measured on a calm Tuesday. Take your p99 of `ApproximateAgeOfOldestMessage`
over the last 90 days, not the mean.

**Do this before committing to option (a):** put a CloudWatch dashboard on
`ApproximateNumberOfMessagesVisible` and `ApproximateAgeOfOldestMessage` for
every queue in the estate for one month. If the p99 age of every queue is single
-digit seconds, option (a) is defensible in writing to a risk committee. If some
queue routinely sits at ten minutes deep, that queue specifically needs one of
the other options — or, more likely, needs more consumers.

**Cost:** zero. An SQS queue with no API calls against it generates no requests,
and SQS is billed per request. An empty standby queue is genuinely free.

**Verdict: this is the default and it is usually correct.** It is also the only
option with no moving parts, which at 3am is worth more than a small RPO
improvement.

---

### Option (b) — Dual-write / fan-out from SNS to queues in both regions

Instead of the producer writing to SQS directly, the producer publishes to an
SNS topic in the primary region. The topic has two subscriptions: the local
`eu-west-1` queue, and — natively, no code — the `eu-west-2` standby queue.
Both queues receive every message. At failover, the standby queue already
contains everything the primary queue contained.

This works because SNS *does* support cross-region delivery to SQS, officially
and documented
([docs](https://docs.aws.amazon.com/sns/latest/dg/sns-cross-region-delivery.html)).
It is the single most important fact in this whole family of notes. Full detail,
including the exact queue policy, is in [[aws-sns]]; there is also an official
AWS sample of the pattern at
[aws-samples/sample-sns-sqs-multi-region](https://github.com/aws-samples/sample-sns-sqs-multi-region).

**What this actually gives you:** an RPO for queued work of *seconds* — SNS
delivery latency — rather than "whatever was in flight". That is a real
improvement and it requires no bespoke code.

**What it costs you, and this is the part people skip:**

1. **Every message is now processed twice, or nearly.** If the standby consumer
   is running, both regions do the work. If it isn't, the standby queue grows
   unboundedly until it hits the 14-day retention wall and starts silently
   discarding — which produces a queue that *looks* full of recoverable work and
   is in fact a rolling 14-day window with an invisible tail. You must either
   (i) run standby consumers and make every consumer globally idempotent, or
   (ii) not run them and accept that the standby queue is a buffer with a
   deliberate 14-day horizon.
2. **The topic is in the primary region.** This is the fatal flaw for DR. If
   `eu-west-1` is the thing that's broken, the topic you publish to is in
   `eu-west-1`. Fan-out does not help you *during* the outage; it helps you
   recover the backlog that existed *before* it. To close that you need topics in
   both regions and producer-side failover, which is [[aws-sns]]'s problem.
3. **Double the cross-region data transfer.** Every byte of every message crosses
   the region boundary. Inter-region transfer within Europe is on the order of
   $0.02/GB, charged on the sending side (verify current rates against the AWS
   pricing page — third-party trackers agree on $0.02/GB EU↔EU but AWS's own page
   is the authority). At high message volume with large payloads this is a real
   line item, unlike the queue itself.
4. **You have to move producers onto SNS.** If producers currently call
   `SendMessage` directly, this is an application change across every producing
   service — the single biggest cost of this option, and it is a people cost, not
   an AWS one.

**Verdict: correct for a small number of high-value queues** — payment
instructions, order placements, anything where a stranded message is a customer
calling you. Wrong as an estate-wide default, because you'd be paying
cross-region transfer and double-processing risk on queues whose messages are
worth nothing an hour later.

---

### Option (c) — Drain-forwarder Lambda relaying cross-region

A Lambda in the primary region (or the standby) polls the primary queue and
`SendMessage`s each message into the standby queue. Continuous, or triggered
only at failover time to drain what's left.

This is the option that gets designed on a whiteboard in four minutes and then
owned forever. Be honest about it:

**If it runs continuously**, it is not replication, it is *consumption*. A
Lambda that receives a message and deletes it has taken the message out of the
primary queue; if it instead receives-forwards-and-doesn't-delete, the message
comes back after the visibility timeout and gets forwarded again, and again,
forever. There is no "peek without consuming" in SQS. So a continuous forwarder
must either drain the primary (breaking the primary's own consumers — now the
forwarder *is* the consumer and must re-publish locally too) or duplicate
without bound. **Neither is a replication primitive.** If you want both regions
to see every message, use option (b); SNS already does exactly this, correctly,
for free, with no code.

**If it runs only at failover** — "drain the leftovers into the standby once the
primary comes back" — it is much more defensible, and it is genuinely useful.
This is a *recovery* tool, not a *replication* tool, and framing it that way is
what makes it sane:

- It runs **after** the primary region is reachable again, which may be hours
  later. That is fine: SQS retention is up to 14 days (see below), so the
  messages are still there.
- It is a one-shot batch job, not a standing system. It can be a Lambda, or
  honestly it can be a Python script an engineer runs from a laptop with two
  boto3 clients. A thing you run once per incident does not need to be
  serverless infrastructure.
- It needs a kill switch and a dry-run mode, because the failure mode is
  "replayed 400,000 stale messages into a live system".

**The ordering and dedup problem.** Relayed messages arrive in the standby queue
with new message IDs, at a new time, possibly out of order, and — for FIFO —
with a dedup window that knows nothing about the source queue. See the FIFO
section below. For standard queues this is survivable if consumers are
idempotent. For FIFO it silently voids the guarantee.

**The "the leftovers are stale" problem.** Messages drained hours after the
outage may be instructions that no longer make sense — "charge this card",
"send this email", "reserve this inventory". Replaying them blind can be worse
than losing them. The drain script needs a business-level filter, and deciding
that filter is a product decision, not an infra one. Put it in the runbook now,
not at 3am.

**Verdict: build the failover-time drainer, don't build the continuous
forwarder.** And build it as a documented script with a dry-run, not as
always-on infrastructure. It is the cheapest insurance in this note: it costs
nothing while idle and recovers option (a)'s losses *most* of the time, since
most region events are degradations that end, not craters.

---

### Option (d) — Rearchitect so producers are idempotent and replayable

Make the queue a transport, not a store of record. The message's content exists
somewhere durable and cross-region-replicated *before* it is enqueued — a
DynamoDB Global Table row, an Aurora Global Database row, an outbox table. The
queue carries a pointer and a "please process this" signal. If the queue's
contents evaporate, you re-derive the work by scanning the source of truth for
records in a non-terminal state and re-enqueuing them.

This is the transactional outbox pattern with the region boundary drawn around
it, and it is **the only option on this list that makes the stranded-message
problem structurally go away** rather than mitigating it.

**How it works at failover:**

1. Fail over. Standby queue is empty, standby consumers are running.
2. Run a reconciliation query against the (already replicated) source of truth:
   `SELECT * FROM orders WHERE status = 'PENDING_FULFILMENT' AND updated_at > now() - interval '2 hours'`.
3. Re-enqueue those. Consumers are idempotent, so anything the primary already
   processed before it died is a no-op.
4. The stranded primary-region messages are now irrelevant. When the primary
   comes back you *delete* them rather than draining them.

**The prerequisites are real:**

- Consumers must be idempotent. Truly idempotent, not "we set a flag at the end
  and hope". This is the expensive part and it is application work.
- The source of truth must itself be cross-region replicated with an RPO inside
  2h — which is what [[aws-dynamodb]] (Global Tables) and
  [[aws-aurora-global-database]] are for. **This option is only available to you
  once the database work lands**, which fits the estate's prerequisites-first
  strategy exactly: it's a reason to sequence the database notes ahead of a
  bespoke queue-replication project.
- Somebody has to write and *test* the reconciliation query per workload. A
  reconciliation job that has never been run is a reconciliation job that
  doesn't work.

**Verdict: this is the right long-term answer for anything that matters, and it
is free of AWS-specific machinery.** It is also the only option here that also
fixes single-region bugs (poison messages, consumer crashes, a bad deploy that
ate a batch). Recommend it as the target state for the handful of workloads
where message loss is genuinely unacceptable, and recommend option (a) for
everything else. The combination — (a) everywhere, (d) for the crown jewels,
(c) as a recovery script — is the design this note lands on.

---

### Option comparison

| | (a) Empty mirror | (b) SNS fan-out | (c) Drain-forwarder | (d) Idempotent + replayable |
|---|---|---|---|---|
| Native AWS feature? | Yes (just Terraform) | Yes (SNS x-region) | No — you own it | No — app design |
| Code to write | None | None (producer change) | A script | Significant |
| RPO for queued work | Queue latency (secs) | Seconds | Recoverable post-incident | Zero — re-derivable |
| Works *during* primary outage? | Yes (new traffic) | Topic is in primary — no | No | Yes |
| Breaks FIFO semantics? | No | No (SNS FIFO→SQS FIFO holds) | **Yes** | No |
| Ongoing cost | £0 | x-region transfer + double processing | £0 idle | £0 |
| Ongoing ops burden | None | Low | Medium | Low once built |
| Recommend | **Default** | High-value queues only | As a recovery script | Target state for critical paths |

## SQS FIFO across regions

FIFO is where cross-region gets genuinely dangerous, because the failure is
silent. Standard queues degrade honestly — you get duplicates, you knew you
might. FIFO degrades by continuing to *claim* a guarantee it is no longer
providing.

**Deduplication is scoped to the queue.** A FIFO queue's
`DeduplicationScope` is either `queue` (default) or `messageGroup`, and in both
cases the scope is *inside one queue*. There is no account-level or global dedup
namespace. The dedup window is 5 minutes.

Consequences for every cross-region pattern:

- **Relay (option c) voids dedup.** Send a message to the primary FIFO queue,
  relay it to the standby FIFO queue. The standby queue has never seen that
  `MessageDeduplicationId` and accepts it. Run the relay twice, or run it after a
  partial earlier run, and the standby gets the message twice — because the two
  runs may be more than 5 minutes apart, outside the dedup window. **The dedup
  window is 5 minutes; your failover is 15. Dedup cannot help you across a
  failover, by construction.**
- **Relay voids ordering.** FIFO's ordering guarantee is per `MessageGroupId`
  within one queue. A relay that reads a batch and re-sends it can reorder within
  a group (batches are not necessarily received in order across multiple
  `ReceiveMessage` calls, and a partial failure mid-batch reorders trivially). If
  you relay, you must relay strictly serially per message group and handle
  partial failure by restarting the group — at which point you're building a
  replication protocol, and you should stop and use option (b) or (d).
- **Fan-out (option b) preserves both, conditionally.** SNS FIFO topics deliver
  to SQS FIFO queues preserving order and dedup per message group, because SNS
  does the ordering, not the relay. This is the one cross-region FIFO pattern
  that holds. **But:** it is subject to whatever regional availability SNS FIFO
  has, and I did not find an AWS statement explicitly confirming that an SNS
  *FIFO* topic may have a *cross-region* SQS FIFO subscriber. The SNS
  cross-region delivery doc discusses SQS and Lambda destinations without
  distinguishing FIFO. **This is an unverified assumption — test it in a sandbox
  before designing on it.** ([[aws-sns]] carries this open question too.)
- **`fifo_throughput_limit` and `deduplication_scope` must match in both
  regions.** If the standby queue is `perQueue`/`queue` and the primary is
  `perMessageGroupId`/`messageGroup`, the standby will throttle differently under
  the failover surge — exactly when you can least afford it. Drive both from the
  same Terraform variable.

**Recommendation for FIFO queues specifically:** do not relay. Either accept
option (a) — which for FIFO is especially defensible, because FIFO queues are
usually low-volume and consumers usually keep them near-empty by design — or go
to option (d). A FIFO queue whose ordering guarantee is honoured in one region
and not the other is worse than no queue, because the downstream system was
written to trust it.

Note also: AWS markets FIFO as "exactly-once processing", and that phrase does a
lot of unearned work. It means SQS won't *introduce* a duplicate inside the
dedup window. It does not make your `DeleteMessage` and your database write
atomic. A consumer that crashes after committing and before deleting will
reprocess. **You need idempotent consumers regardless of FIFO**, which means
option (d)'s prerequisite is one you should be paying anyway.

## DLQs in the standby

Three facts, and they compose badly:

1. A `RedrivePolicy` names a DLQ by ARN, and **the DLQ must be in the same
   account and region as the source queue**.
2. `StartMessageMoveTask` — the redrive API — moves messages between queues
   **in the same region**. There is no cross-region redrive.
3. `StartMessageMoveTask` only accepts DLQs whose sources are other SQS queues.
   DLQs fed by Lambda or SNS failures aren't supported as sources
   ([API docs](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_StartMessageMoveTask.html)).

So:

- **The standby needs its own DLQs**, created and wired by the same Terraform
  module that creates the main queues. This is mechanical and cheap — just don't
  forget them, because a standby queue with `redrive_policy = null` silently
  drops poison messages after `maxReceiveCount` instead of parking them, and you
  won't notice until an incident review six weeks later asks where the messages
  went.
- **The primary's DLQ contents are the least recoverable thing in the estate.**
  A DLQ is, by definition, full of messages that already failed and that someone
  intended to look at. They have no source-of-truth re-derivation path (that's
  why they're in a DLQ). They can't be redriven cross-region. And they may be
  *old* — DLQs are exactly where the 14-day retention wall matters.
- **Set DLQ retention to the full 14 days** (`message_retention_seconds =
  1209600`) on every DLQ in both regions, unconditionally. The default is 4 days.
  The cost of a longer retention is zero (SQS bills requests, not storage-time),
  and the benefit is that a DLQ stranded by a regional outage survives a
  ten-day incident. **This is the single highest-value one-line change in this
  note.** There is a widely-repeated cautionary story of teams losing data to the
  4-day default without realising the default existed; whether or not any
  specific retelling is verifiable, the default is real and it is wrong for a
  DLQ.
- **Alarm on `ApproximateNumberOfMessagesVisible > 0` on every DLQ, in both
  regions.** In the standby that alarm should be silent forever — which makes it
  a free canary for "something is consuming in the standby that shouldn't be".
  See [[split-brain-and-fencing]].

## The 14-day retention as a natural recovery window

`MessageRetentionPeriod` ranges from 60 seconds to **1,209,600 seconds (14
days)**, default 345,600 (4 days). This is the quiet hero of the whole design.

Reframe it: **retention is the amount of time the primary region has to come
back before option (c) stops being possible.** Almost every real AWS regional
event is a degradation measured in hours. Very few are measured in days. None
in the public record have been measured in weeks. So:

- If retention is 14 days, a stranded message is recoverable for 14 days. The
  primary comes back, you run the drain script, and option (a)'s loss turns out
  to have been a *delay*, not a loss, for the overwhelming majority of incidents.
- If retention is the 4-day default, you have four days.
- If someone set retention to 1 hour on a queue because "these are real-time
  events, stale ones are useless" — which is a perfectly reasonable single-region
  decision — that queue has an RPO of "gone" and nobody wrote it down.

**Recommendation: audit `MessageRetentionPeriod` across the estate as part of
this work, and set it to 14 days everywhere unless a queue has an explicit,
documented business reason for shorter.** Longer retention has no cost (SQS
bills per request; a message sitting in a queue is not a request). The
counter-argument — "short retention protects us from processing stale
instructions" — is real but is better handled by a timestamp check in the
consumer than by trusting SQS to garbage-collect your correctness for you.

The interaction with RPO 2h is neat and worth stating: **the 2h RPO is a
statement about how much data you may lose. A 14-day retention means stranded
queue messages are usually not lost at all, just deferred past the RTO.** That
is a different kind of failure — a latency failure, not a durability one — and
it is a much easier conversation with the business.

## Queue depth as a failover health signal

Two metrics, and the estate should treat them as first-class failover inputs,
not just capacity alerts:

- **`ApproximateNumberOfMessagesVisible`** — depth. How much work is waiting.
- **`ApproximateAgeOfOldestMessage`** — age of the oldest undeleted message,
  updated roughly once a minute. **This is the one that matters.** Depth without
  age is meaningless: 1,000 messages draining in 200ms is healthy, 100 messages
  where the oldest is an hour old is an outage.

Why they belong in the failover decision:

1. **Age is your live RPO exposure meter.** At any moment, "how much would we
   lose if we failed over right now" is approximately the integral under the
   arrival curve over `ApproximateAgeOfOldestMessage`. Put that on the failover
   dashboard next to the Route 53 health checks. The on-call engineer deciding
   whether to fail over should be able to see "we would strand ~4 seconds of
   work" or "we would strand ~40 minutes of work" — those are different
   decisions.
2. **Age climbing on multiple unrelated queues is a strong regional-degradation
   signal**, often earlier than an ALB health check notices, because it detects
   "consumers can't make progress" rather than "the front door is down". Two
   unrelated queues both backing up at once is rarely two coincidental
   application bugs.
3. **Depth on the *standby* queue should be zero, always.** Any non-zero depth on
   a standby queue while the primary is healthy means either a misrouted producer
   (see the naming decision below) or an un-fenced consumer situation. Alarm on
   it at threshold 1. This is cheap and catches the exact class of bug that the
   suffixed-naming option is designed to prevent.
4. **After failover, age on the standby queue is your "are we actually
   recovered" signal.** A standby that is receiving traffic but whose oldest
   message keeps ageing means the standby consumers are under-scaled — which is
   the single most likely way a warm standby fails on the day. Bake this into the
   post-failover verification steps in [[failover-runbook-template]].

One wrinkle worth knowing: `ApproximateAgeOfOldestMessage` can jump around when
a queue contains a stuck poison message — the poison message stays the "oldest"
and pins the metric high even though everything else is draining fine. Alarm on
it, but pair it with depth so you can tell "backed up" from "one bad message".

## The naming decision: same name in both regions, or suffixed

This is a genuine fork with no obviously-correct answer, and it recurs across
this whole vault (it is the same argument as [[dynamodb-table-naming-migration]],
except that Global Tables *force* the answer and SQS leaves it open).

### Branch A — identical queue name in both regions

`orders-prod` in `eu-west-1` and `orders-prod` in `eu-west-2`.

**For:**
- **Configuration is portable.** An app config that says `QUEUE_NAME=orders-prod`
  works unchanged in both regions. The consumer resolves the URL via
  `GetQueueUrl` against the local regional endpoint and gets the local queue.
  Nothing in the deployment artefact is region-aware.
- **This is the only shape that survives the standby becoming the new primary
  permanently.** After a real failover where you never fail back, a suffixed
  scheme leaves you running production out of a queue called `orders-prod-dr`
  forever, and every dashboard, runbook and alert name lies.
- **It matches what the estate already does elsewhere.** Secrets Manager replicas
  keep the same name (and even the same ARN suffix), and DynamoDB Global Tables
  *require* the same name. Consistency with the direction of travel is worth
  real money in a templated monorepo.
- **The Terraform is trivially templatable:** one module, two provider aliases,
  same `name` variable. No conditional naming logic in the cookiecutter.

**Against:**
- **Nothing structurally prevents cross-wiring.** A misconfigured
  `AWS_REGION`/`AWS_DEFAULT_REGION` in a standby pod makes it silently consume
  from the *primary's* queue with no error — same name, valid credentials,
  perfectly plausible. This is a genuinely nasty failure mode because it looks
  like nothing: messages disappear from the primary queue and are processed by a
  region that shouldn't be processing them. It is a split-brain in miniature; see
  [[split-brain-and-fencing]].
- Screenshots, ARNs in tickets, and CLI output are ambiguous unless you read the
  region field carefully. At 3am people do not read the region field carefully.

### Branch B — suffixed names

`orders-prod-euw1` and `orders-prod-euw2` (or `-primary`/`-standby`).

**For:**
- **Accidental cross-wiring becomes impossible.** A standby workload configured
  with `orders-prod-euw2` cannot resolve that name in `eu-west-1`;
  `GetQueueUrl` returns `QueueDoesNotExist`. **A loud, immediate, unambiguous
  failure instead of a silent wrong-region success.** This is a large safety
  benefit and should not be dismissed.
- Unambiguous in every log line, alarm name, ARN and screenshot.
- You can enforce it with an IAM condition or SCP (`sqs:*` on
  `arn:aws:sqs:eu-west-2:*:*-euw2` only), turning a convention into a control.

**Against:**
- **Config is no longer portable**, so every consuming service needs
  region-derived naming — either templated at deploy time, or derived at runtime
  from the instance metadata region. That's a small amount of logic replicated
  into every service, and it is exactly the kind of thing that is right in 40
  services and wrong in the 41st.
- **Failback and permanent-promotion both get ugly.** Running prod forever out of
  `-euw2`-suffixed queues is a documentation debt that never gets paid.
- Slightly more template complexity in cookiecutter.

### Recommendation

**Branch A — same name in both regions — with the cross-wiring risk closed by
controls rather than by naming.**

The reasoning: the portability benefit is structural and permanent, while the
cross-wiring risk is a specific, enumerable failure that has specific,
cheap mitigations:

1. **IAM boundary.** The standby workload's role gets `sqs:*` only on
   `arn:aws:sqs:eu-west-2:<acct>:*`. A cross-region call from a standby pod then
   fails with `AccessDenied` — loud, immediate, unambiguous. This recovers
   *exactly* the safety property Branch B was buying, without paying for it in
   config portability. Do the mirror-image thing for the primary's role.
2. **The alarm above:** depth > 0 on any standby queue while primary is healthy.
3. **VPC endpoint policy.** If the standby VPC has an SQS interface endpoint and
   no route to the internet for SQS traffic, cross-region SQS calls simply don't
   have a path. See [[aws-vpc-networking]].

Control (1) alone is enough, costs one line of Terraform, and is more robust
than a naming convention because it's enforced by IAM rather than by everyone
remembering the convention.

**But note the exception:** if the estate cannot guarantee tight per-region IAM
roles — e.g. if workloads share a broad role across regions, or if there are
human operators with account-wide `sqs:*` — then Branch B's mechanical safety is
worth more than Branch A's tidiness, and you should suffix. Check this before
committing. It is a five-minute question to the IAM owner and it flips the
recommendation. See [[aws-iam]].

## RPO / RTO analysis

### Against RTO 15m: passes easily

Nothing about SQS takes time at failover.

| Step | Time | Pre-provisioned? |
|---|---|---|
| Standby queue exists | 0 s | **Yes** — Terraform, always |
| Standby DLQ exists and is wired | 0 s | **Yes** |
| Queue policy allows the standby workload | 0 s | **Yes** |
| KMS key for SSE available in standby | 0 s | **Yes** — see gotchas + [[aws-kms]] |
| Consumers start polling | seconds–minutes | Depends on [[aws-eks]] / [[aws-lambda]], not SQS |
| Producers point at standby queue | seconds | Config/DNS — the actual work |

The SQS contribution to RTO is effectively zero. **Creating a queue at failover
time would also be fast** (`CreateQueue` is a single fast API call) — but do not
do it, because (i) `CreateQueue` after a `DeleteQueue` of the same name requires
a 60-second wait, and more importantly (ii) creating it at failover time means
the queue policy, DLQ wiring, encryption config and tags are also being created
at failover time, and that is four more things to get wrong at 3am. Pre-provision.

The real RTO risk is the standby consumer fleet, not the queue. A queue with no
consumers meets its RTO and delivers nothing. That's [[aws-eks]]'s problem and
[[eks-workload-delivery]]'s problem.

### Against RPO 2h: passes, but the framing needs care

The honest statement is: **SQS's contribution to RPO is not measured in hours at
all. It's measured in seconds, and it is a different *kind* of loss than the 2h
RPO was written to describe.**

- RPO 2h is a statement about *committed state* — "we may lose up to two hours of
  database writes".
- Unprocessed queue messages are *pending work*, not committed state. Whether
  they count against the RPO budget at all is the central argument of
  [[messaging-in-flight-data-loss]], and the answer materially changes how much
  engineering this deserves.
- Numerically it doesn't matter much either way: a healthy queue holds seconds of
  work, which is two orders of magnitude inside a 2h budget. **SQS is not the
  thing that will blow your RPO.** [[aws-rds-postgres]] and
  [[aws-elasticache-redis]] are far more likely candidates.
- The exception is a *sick* queue. A queue that has been backing up for 90
  minutes when the region dies strands 90 minutes of work — still inside 2h, but
  uncomfortably close, and it's the only realistic path by which SQS threatens
  the target. Which is another argument for alarming hard on
  `ApproximateAgeOfOldestMessage` with a threshold far below 2 hours.

## Warm standby shape

While the primary is healthy, the standby region contains:

| Resource | State | Cost while idle |
|---|---|---|
| Main queues | Created, empty | **£0** — SQS bills per request; no requests, no bill |
| DLQs | Created, empty, 14-day retention | **£0** |
| Queue policies | Applied | £0 |
| KMS key / alias for SSE | Present (see [[aws-kms]]) | ~$1/month per CMK, or £0 with SSE-SQS |
| CloudWatch alarms on depth/age | Active, in OK state | Pennies |
| Consumers | **Open decision** — see [[messaging-in-flight-data-loss]] | Depends |

**An empty SQS queue is one of the cheapest warm-standby resources in the entire
estate.** There is no per-queue-per-month charge, no storage charge, no minimum.
This is worth stating explicitly in [[cost-model]] because it means there is no
cost argument whatsoever for *not* pre-provisioning every queue in the standby,
including ones you think you'll never need. Create them all.

The only meaningful idle cost is a customer-managed KMS key if you use SSE-KMS.
If the queue's payloads don't require a CMK, `sqs_managed_sse_enabled = true`
(SSE-SQS) is free and removes a whole class of cross-region KMS grant problems.
See the gotchas.

## Terraform implementation

### The module

A single module, instantiated once per region via provider alias. The module
itself is region-agnostic — it inherits whatever provider it's given — which is
the property that makes it fit a cookiecutter monorepo.

```hcl
# modules/sqs-queue/variables.tf

variable "name" {
  description = "Queue name. Identical in both regions — see the naming decision."
  type        = string
}

variable "fifo" {
  description = "FIFO queue. Adds the required .fifo suffix automatically."
  type        = bool
  default     = false
}

variable "visibility_timeout_seconds" {
  type    = number
  default = 30
}

variable "message_retention_seconds" {
  description = "Default 14 days. Do not lower without a documented reason."
  type        = number
  default     = 1209600 # 14 days — the maximum, and the estate default
}

variable "max_receive_count" {
  description = "Deliveries before a message is parked in the DLQ."
  type        = number
  default     = 5
}

variable "kms_master_key_id" {
  description = "CMK ARN or alias. Null selects SSE-SQS (AWS-owned key, free, no cross-region grant problems)."
  type        = string
  default     = null
}

variable "deduplication_scope" {
  description = "FIFO only: queue | messageGroup. MUST match across regions."
  type        = string
  default     = null
}

variable "fifo_throughput_limit" {
  description = "FIFO only: perQueue | perMessageGroupId. MUST match across regions."
  type        = string
  default     = null
}

variable "producer_principals" {
  description = "IAM principal ARNs allowed to SendMessage. Include the *other* region's producer role if you use SNS cross-region fan-out."
  type        = list(string)
  default     = []
}

variable "alarm_actions" {
  type    = list(string)
  default = []
}

variable "is_standby" {
  description = "True in the standby region. Adds the 'this queue should be empty' canary alarm."
  type        = bool
  default     = false
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

```hcl
# modules/sqs-queue/main.tf

locals {
  suffix     = var.fifo ? ".fifo" : ""
  queue_name = "${var.name}${local.suffix}"
  dlq_name   = "${var.name}-dlq${local.suffix}"
}

resource "aws_sqs_queue" "dlq" {
  name       = local.dlq_name
  fifo_queue = var.fifo

  # DLQs always get maximum retention. A DLQ stranded by a regional outage
  # must survive a multi-day incident.
  message_retention_seconds = 1209600

  kms_master_key_id                 = var.kms_master_key_id
  kms_data_key_reuse_period_seconds = var.kms_master_key_id == null ? null : 300
  sqs_managed_sse_enabled           = var.kms_master_key_id == null ? true : null

  deduplication_scope   = var.fifo ? var.deduplication_scope : null
  fifo_throughput_limit = var.fifo ? var.fifo_throughput_limit : null

  tags = merge(var.tags, { Role = "dlq" })
}

resource "aws_sqs_queue" "main" {
  name       = local.queue_name
  fifo_queue = var.fifo

  visibility_timeout_seconds = var.visibility_timeout_seconds
  message_retention_seconds  = var.message_retention_seconds
  receive_wait_time_seconds  = 20 # long polling; always on, never a reason not to

  kms_master_key_id                 = var.kms_master_key_id
  kms_data_key_reuse_period_seconds = var.kms_master_key_id == null ? null : 300
  sqs_managed_sse_enabled           = var.kms_master_key_id == null ? true : null

  deduplication_scope   = var.fifo ? var.deduplication_scope : null
  fifo_throughput_limit = var.fifo ? var.fifo_throughput_limit : null

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dlq.arn # same region, enforced by AWS
    maxReceiveCount     = var.max_receive_count
  })

  tags = var.tags
}

# Only the paired main queue may redrive into this DLQ.
resource "aws_sqs_queue_redrive_allow_policy" "dlq" {
  queue_url = aws_sqs_queue.dlq.id

  redrive_allow_policy = jsonencode({
    redrivePermission = "byQueue"
    sourceQueueArns   = [aws_sqs_queue.main.arn]
  })
}

data "aws_iam_policy_document" "main" {
  count = length(var.producer_principals) > 0 ? 1 : 0

  statement {
    sid     = "AllowNamedProducers"
    effect  = "Allow"
    actions = ["sqs:SendMessage"]
    resources = [aws_sqs_queue.main.arn]

    principals {
      type        = "AWS"
      identifiers = var.producer_principals
    }
  }
}

resource "aws_sqs_queue_policy" "main" {
  count     = length(var.producer_principals) > 0 ? 1 : 0
  queue_url = aws_sqs_queue.main.id
  policy    = data.aws_iam_policy_document.main[0].json
}
```

```hcl
# modules/sqs-queue/alarms.tf

resource "aws_cloudwatch_metric_alarm" "oldest_message_age" {
  alarm_name  = "sqs-${local.queue_name}-oldest-message-age"
  namespace   = "AWS/SQS"
  metric_name = "ApproximateAgeOfOldestMessage"
  dimensions  = { QueueName = aws_sqs_queue.main.name }

  statistic           = "Maximum"
  period              = 60
  evaluation_periods  = 5
  comparison_operator = "GreaterThanThreshold"

  # 900s = the RTO. If the oldest message is older than our entire RTO budget,
  # our live RPO exposure has stopped being "seconds" and someone must know.
  threshold = 900

  alarm_description = "Live RPO exposure for this queue exceeds the 15m RTO budget. See 02-services/aws-sqs.md."
  alarm_actions     = var.alarm_actions
  ok_actions        = var.alarm_actions
  treat_missing_data = "notBreaching"
}

resource "aws_cloudwatch_metric_alarm" "dlq_not_empty" {
  alarm_name  = "sqs-${local.dlq_name}-not-empty"
  namespace   = "AWS/SQS"
  metric_name = "ApproximateNumberOfMessagesVisible"
  dimensions  = { QueueName = aws_sqs_queue.dlq.name }

  statistic           = "Maximum"
  period              = 300
  evaluation_periods  = 1
  comparison_operator = "GreaterThanThreshold"
  threshold           = 0

  alarm_actions      = var.alarm_actions
  treat_missing_data = "notBreaching"
}

# Standby canary: this queue should never have anything in it while the
# primary is healthy. Non-zero depth here means a misrouted producer or an
# un-fenced consumer. See 04-operations/split-brain-and-fencing.md.
resource "aws_cloudwatch_metric_alarm" "standby_should_be_empty" {
  count = var.is_standby ? 1 : 0

  alarm_name  = "sqs-${local.queue_name}-standby-unexpectedly-non-empty"
  namespace   = "AWS/SQS"
  metric_name = "ApproximateNumberOfMessagesVisible"
  dimensions  = { QueueName = aws_sqs_queue.main.name }

  statistic           = "Maximum"
  period              = 300
  evaluation_periods  = 1
  comparison_operator = "GreaterThanThreshold"
  threshold           = 0

  alarm_description  = "Standby queue is not empty while primary is active. Suspect misrouted producer or cross-wired region."
  alarm_actions      = var.alarm_actions
  treat_missing_data = "notBreaching"
}
```

### Calling it for a pair

```hcl
# environments/prod-eu/providers.tf

provider "aws" {
  alias  = "primary"
  region = "eu-west-1"
  default_tags { tags = local.common_tags }
}

provider "aws" {
  alias  = "standby"
  region = "eu-west-2"
  default_tags { tags = local.common_tags }
}
```

```hcl
# environments/prod-eu/queues.tf

locals {
  queues = {
    orders     = { fifo = true,  visibility_timeout_seconds = 60, deduplication_scope = "messageGroup", fifo_throughput_limit = "perMessageGroupId" }
    thumbnails = { fifo = false, visibility_timeout_seconds = 300 }
    audit      = { fifo = false, visibility_timeout_seconds = 30 }
  }
}

module "queues_primary" {
  source   = "../../modules/sqs-queue"
  for_each = local.queues
  providers = { aws = aws.primary }

  name                       = "${each.key}-${var.environment}"
  fifo                       = try(each.value.fifo, false)
  visibility_timeout_seconds = try(each.value.visibility_timeout_seconds, 30)
  deduplication_scope        = try(each.value.deduplication_scope, null)
  fifo_throughput_limit      = try(each.value.fifo_throughput_limit, null)

  producer_principals = [aws_iam_role.app_primary.arn]
  alarm_actions       = [aws_sns_topic.alerts_primary.arn]
  is_standby          = false
  tags                = { Region = "primary" }
}

module "queues_standby" {
  source   = "../../modules/sqs-queue"
  for_each = local.queues
  providers = { aws = aws.standby }

  # Identical name. Identical everything. That is the entire point.
  name                       = "${each.key}-${var.environment}"
  fifo                       = try(each.value.fifo, false)
  visibility_timeout_seconds = try(each.value.visibility_timeout_seconds, 30)
  deduplication_scope        = try(each.value.deduplication_scope, null)
  fifo_throughput_limit      = try(each.value.fifo_throughput_limit, null)

  producer_principals = [aws_iam_role.app_standby.arn]
  alarm_actions       = [aws_sns_topic.alerts_standby.arn]
  is_standby          = true
  tags                = { Region = "standby" }
}
```

The `for_each` over one `local.queues` map is the load-bearing bit for a
cookiecutter estate: **there is exactly one list of queues, and both regions are
derived from it.** Drift between regions becomes structurally impossible rather
than a review checklist item. If someone adds a queue and forgets the standby,
`terraform plan` adds it to both. That is the whole design goal.

### A note on `aws_sqs_queue_redrive_allow_policy`

Newer provider versions split the redrive-allow policy out of the queue
resource. If the estate is on an older 5.x, check the provider version pin
before copying the above — on very old versions this was a `redrive_allow_policy`
attribute on `aws_sqs_queue` itself. It is not a functional difference, just a
schema one.

## Migration path from single-region

**There is nothing to migrate.** This is the easiest note in the vault on this
axis, and the reason is worth stating: because SQS has no replication, there is
no "convert this queue into a replicated queue" operation that could force
replacement. You are purely *adding* resources in a new region.

### Step by step

1. **Import or confirm the existing primary queues are under Terraform
   management.** If any production queue was clicked into existence, `terraform
   import` it first. Do this *before* refactoring into the module, so the
   refactor's plan is readable.

2. **Refactor the primary into the module — carefully.** This is the only step
   with replacement risk, and it's a Terraform risk, not an AWS one. Moving
   `aws_sqs_queue.orders` into `module.queues_primary["orders"].aws_sqs_queue.main`
   is an address change. Use `moved` blocks:

   ```hcl
   moved {
     from = aws_sqs_queue.orders
     to   = module.queues_primary["orders"].aws_sqs_queue.main
   }

   moved {
     from = aws_sqs_queue.orders_dlq
     to   = module.queues_primary["orders"].aws_sqs_queue.dlq
   }
   ```

   **Read the plan. `terraform plan` must show zero destroys.** If it shows a
   destroy of a live queue, stop — you are about to delete production messages,
   and SQS additionally imposes a 60-second cooldown before a queue of the same
   name can be recreated, so the outage is not instantaneous-and-recovered.

3. **Watch for the `name` attribute.** `name` on `aws_sqs_queue` is ForceNew —
   changing it destroys and recreates. If the module's computed name differs
   from the live queue's name by even one character (a `.fifo` suffix, a hyphen,
   an environment prefix), the plan will destroy the live queue. This is the
   single most dangerous line in the migration. **Verify the computed name
   string-by-string against the live queue name before applying.**

4. **Also watch `fifo_queue`, `deduplication_scope` and `fifo_throughput_limit`**
   — `fifo_queue` is likewise ForceNew (a standard queue cannot become FIFO), and
   the FIFO tuning attributes may be ForceNew depending on the provider version.
   Confirm against your pinned provider's docs before assuming an in-place update.

5. **Add the standby module instantiation.** This is pure creation: new queues,
   new DLQs, new policies, new alarms in `eu-west-2`. No effect on the primary.
   `terraform plan` shows only adds. Apply it.

6. **Bump retention to 14 days on the primary queues.**
   `message_retention_seconds` is an in-place update — no replacement, no
   disruption, no message loss. Do it as its own small PR so it's visible.

7. **Add the IAM boundary** (per-region `sqs:*` scoping on the app roles) that
   the naming recommendation depends on. See [[aws-iam]].

8. **Test.** Send a message to the standby queue by hand. Receive it by hand.
   Confirm it's encrypted as expected, confirm the DLQ wiring by setting
   `maxReceiveCount` low on a scratch queue and letting a message fail. Then
   delete the test messages. A standby queue that has never had a message
   through it is not a tested standby queue.

### Zero-downtime guarantee

Steps 2–8 are all either metadata operations on the primary or creations in the
standby. **No step drains, pauses, or reconfigures a live consumer.** The only
downtime risk in the whole migration is a botched `moved` block, which is caught
by reading the plan.

## Failover procedure

Assumes option (a) + option (c) + option (d) for critical paths, which is the
recommended combination.

### At the moment of failover

1. **Record the exposure before you cut over.** Screenshot or
   `aws cloudwatch get-metric-statistics` for
   `ApproximateNumberOfMessagesVisible` and `ApproximateAgeOfOldestMessage` on
   every primary queue. If the primary is fully dark you can't — which is
   itself why you should be recording these continuously to a cross-region
   destination. See [[observability-multi-region]]. **You cannot reconstruct
   what you stranded after the fact if your metrics died with the region.**
2. **Fence the primary consumers.** If the primary region is only *degraded*, not
   dark, its consumers may still be draining the queue while the standby starts
   processing the same logical work from the source of truth. That is
   double-processing. If the primary is reachable enough to act on, disable its
   consumers — Lambda event source mappings disabled, EKS deployments scaled to
   zero. AWS now ships a purpose-built primitive for exactly this: **ARC Region
   Switch's Lambda event source mapping execution block**, which enables/disables
   ESMs (SQS, Kinesis, DynamoDB Streams, MSK) as a step in a failover plan, with
   an "ungraceful" mode for when the primary can't be reached
   ([announcement](https://aws.amazon.com/about-aws/whats-new/2026/05/region-switch-lambda-esm-execution-block/)).
   This is directly relevant and worth evaluating in [[failover-orchestration]].
3. **Point producers at the standby queue.** With Branch A naming, this is a
   region change in configuration, not a queue-name change. Mechanically it's
   whatever [[failover-orchestration]] decides — environment variable,
   Parameter Store value, or simply "the standby pods were always configured for
   `eu-west-2` and we're just scaling them up".
4. **Scale up standby consumers.** Not an SQS action.
5. **Reconcile from the source of truth** for any workload on option (d). This is
   the step that actually recovers the stranded work, and it should be a
   documented, tested, per-workload script. If it doesn't exist for a given
   workload, that workload is on option (a) and you've accepted the loss —
   which should also be written down, per workload, *before* the incident.
6. **Verify:** standby `ApproximateAgeOfOldestMessage` rises and then falls. If
   it rises and keeps rising, the standby consumer fleet is under-scaled. This is
   the most likely way the failover fails, and it is fixable in minutes if
   someone is watching the right graph.

### Once the primary region returns

7. **Do not let the primary's consumers come back up automatically.** If the EKS
   nodes rejoin and the deployments are still at their pre-incident replica
   count, they will start draining the primary queue and processing stale work
   into a system that has moved on. This is the highest-risk moment of the whole
   incident and it happens *after* everyone has declared victory. Whatever
   fencing mechanism you use must be persistent across a region recovery, not
   just a runtime flag that resets. See [[split-brain-and-fencing]].
8. **Then, deliberately, decide what to do with the stranded messages.** Three
   options, per queue, decided by a human:
   - **Drain them** into the (now-active) standby queue with the drain script.
     Correct when the messages are still semantically valid.
   - **Discard them** (`PurgeQueue`). Correct when option (d) reconciliation has
     already recovered the work, or when the messages are time-sensitive and now
     meaningless.
   - **Archive them** — drain them into a holding queue or an S3 object for
     later analysis. Correct when you're not sure, which is most of the time.
     This is the safe default and should be the runbook's default.

   **`PurgeQueue` deletes everything and cannot be undone.** Whoever runs it
   should have to type the queue name, and it should be a two-person action.

## Failback

Failback is where the option (b) fan-out design pays a debt, and where option
(a) is almost free.

**Option (a) failback:** trivially, the mirror image of failover. Producers point
back at `eu-west-1`, `eu-west-2`'s consumers drain the residue and stop,
`eu-west-2`'s queues return to empty. The only real work is the stranded-message
decision above, and this time you get to make it calmly because nothing is on
fire. **Failback should be scheduled, in business hours, with the same runbook as
failover run in reverse** — and the queues are the easy part of it.

**Option (b) failback is harder.** If you built primary→standby SNS fan-out, that
subscription is one-directional. After failover you are publishing to the
`eu-west-2` topic, which has no subscription pointing back to `eu-west-1`. You
have two sub-choices, and this must be decided at build time not at failback
time:

- **Symmetric topics** — both regions have a topic, each with a local queue
  subscription and a cross-region subscription to the other region's queue.
  Failback is then free and the design is region-agnostic. The cost is that you
  are always paying double cross-region transfer and every message lands in four
  places. It also creates an obvious loop hazard if anyone ever subscribes topic
  A to topic B.
- **Asymmetric, reconfigured at failback** — only primary→standby exists
  normally, and you add standby→primary as a Terraform change during failback.
  Cheaper in steady state, but it means failback has a Terraform apply in its
  critical path, which is a thing to be nervous about.

**Recommendation: symmetric.** The cost difference is small relative to the
operational cost of a Terraform apply during an incident, and symmetry means the
"which region is primary" question stops being encoded in the infrastructure —
which is the property that makes failback boring, and boring is the goal.

**A queue-specific failback trap:** after failing back, `eu-west-2`'s queues
should return to empty and *stay* empty, and the standby canary alarm above will
tell you if they don't. A non-empty standby queue two days after failback means
some producer never got pointed back. That producer is writing into a void. The
canary alarm is what catches it.

## Gotchas

1. **There is no cross-region SQS replication.** Restated because it is the whole
   note and because at least one person in every design review will have read a
   blog post claiming otherwise.

2. **`PurgeQueue` cannot be undone, and can only be called once every 60
   seconds.** In the post-incident cleanup, when someone is tired, this is the
   command that turns a recoverable incident into an unrecoverable one.

3. **Deleting a queue imposes a 60-second wait before a queue of the same name
   can be created.** Relevant if a Terraform refactor accidentally schedules a
   destroy-then-create: the apply will fail partway, leaving you with no queue at
   all for a minute — which is a real production outage caused by a rename.

4. **KMS grants do not cross regions.** If the queue uses SSE-KMS with a
   customer-managed key, the standby queue needs a key *in the standby region*
   — either a separate regional key or a multi-region key replica — and the
   standby workload's role needs `kms:Decrypt`/`kms:GenerateDataKey` on *that*
   key. A standby queue configured with the primary's key ARN is simply broken
   and will fail at first use, which you will discover during the failover.
   **Test the standby queue with a real encrypted message before you need it.**
   See [[aws-kms]] and [[kms-when-to-use-multi-region-keys]].
   *Corollary:* if the payloads don't actually require a CMK, use
   `sqs_managed_sse_enabled = true` and this entire class of problem disappears
   for free.

5. **The DLQ must be in the same account and region as its source queue**, and
   `StartMessageMoveTask` is same-region. There is no cross-region DLQ and no
   cross-region redrive. The standby needs its own full DLQ topology.

6. **`StartMessageMoveTask` only supports DLQs whose source is another SQS
   queue.** DLQs fed by SNS subscription failures or Lambda async failures are
   not supported as move sources — so the "just redrive it" recovery plan
   silently doesn't apply to your SNS DLQs. You'll be writing a script.

7. **The 4-day default retention is wrong for a multi-region estate.** It is not
   wrong *by AWS's design*; it's wrong *for this use case*. Audit and set to 14
   days.

8. **`name` and `fifo_queue` are ForceNew.** A rename is a destroy. In a
   cookiecutter estate where names are computed from templates, a change to the
   naming template is a change to production queue identity. Treat naming-template
   PRs as production-affecting changes, with the plan output in the PR
   description.

9. **FIFO dedup windows are 5 minutes and per-queue.** They provide no protection
   across a failover, which takes 15. Do not put "FIFO handles duplicates" in a
   failover design document.

10. **In-flight message limits are 120,000 for both standard and FIFO queues**
    (FIFO was raised from 20,000 to 120,000 in November 2024). This matters
    during failover recovery specifically: if you scale the standby consumer
    fleet aggressively to chew through a backlog, you can hit the in-flight cap,
    and SQS does not return an error — processing just quietly degrades. Know
    the number before you scale to 500 pods.

11. **`ApproximateAgeOfOldestMessage` is pinned high by a single poison
    message**, so an age alarm can fire for "one bad message" rather than "we're
    backed up". Pair it with depth. Otherwise the alarm gets muted, and then
    it isn't there when you need it as an RPO gauge.

12. **`ca-west-1`.** SQS itself is available in Calgary, but this note's
    surrounding machinery is not uniformly available there — notably EventBridge
    archive/replay and global endpoints are **not** available in `ca-west-1`
    (see [[aws-eventbridge]]). SQS-only designs are fine for the CA pair; designs
    that lean on EventBridge replay are not. Feed this into
    [[region-pair-selection]].

13. **`ca-west-1` is an opt-in region.** It must be explicitly enabled on the
    account before any Terraform can target it, and opt-in status changes the
    required SNS service principal in cross-region queue policies (see
    [[aws-sns]]). This is a prerequisite step, not a footnote — a provider block
    pointing at a non-enabled region fails at plan time.

14. **Cross-region data transfer is charged on the sending side.** An SNS
    fan-out design's transfer bill lands in the primary region's cost centre,
    not the DR one, which makes it easy to misattribute in
    [[cost-model]].

15. **A queue you never tested is not a standby.** The failure modes above —
    wrong KMS key, missing queue policy, absent DLQ, mismatched FIFO settings —
    are all invisible on an empty, unused queue and all fatal on the day. Put
    "send and receive one message through every standby queue" into
    [[dr-testing-and-gamedays]] as a recurring check.

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Queue naming across regions | Identical names — config portable, survives permanent promotion | Region-suffixed — cross-wiring impossible, unambiguous in logs | **A**, with per-region IAM resource scoping to recover B's safety. **Flip to B if the estate can't guarantee tight per-region roles** — check with [[aws-iam]] first. |
| Mirroring mechanism (estate default) | Empty mirror queue, accept in-flight loss | SNS cross-region fan-out on everything | **A.** Exposure is seconds, cost is zero, moving parts are zero. |
| Mirroring for crown-jewel queues | Same as estate default | SNS fan-out, or idempotent+replayable producers | **Idempotent + replayable (option d)**, falling back to SNS fan-out where the app can't be changed. Requires the database work to land first. |
| Continuous cross-region forwarder Lambda | Build it | Don't | **Don't.** It isn't replication — SQS has no peek — and SNS already does the job natively. |
| Failover-time drain script | Build it now | Improvise on the day | **Build it now**, with a dry-run flag and a documented staleness filter. Costs nothing idle; converts most "loss" into "delay". |
| Message retention | Keep per-queue values as they are | 14 days everywhere unless documented otherwise | **14 days everywhere.** Zero cost, materially widens the recovery window. Absolutely non-negotiable on DLQs. |
| Standby consumers running? | Poll an empty queue continuously | Sit at zero | See [[messaging-in-flight-data-loss]] — it's argued in full there. |
| FIFO cross-region strategy | Relay at failover | Never relay; accept loss or use option (d) | **Never relay.** A silently-void ordering guarantee is worse than a known gap. |
| Post-incident stranded messages | Drain by default | Archive by default, drain on decision | **Archive by default.** Draining stale instructions into a recovered system is how a recovered incident becomes a second incident. |

## Cost

**The queues themselves: effectively zero while idle.**

- SQS bills per *request*. Every API action is a request; payloads are chunked at
  64 KB per billable request (a 1 MiB message is 16 requests). An empty queue
  with no API calls against it generates no requests and therefore no charge.
  There is no per-queue, per-month or per-GB-stored charge.
- The first **1 million requests per month are free**, permanently, across all
  regions (excluding GovCloud) — per AWS's pricing page.
- Rate beyond the free tier: third-party pricing trackers consistently state
  **$0.40 per million requests for standard queues and $0.50 per million for
  FIFO** in the first tier. *I was not able to get AWS's own pricing page to
  render its rate table, so treat these two numbers as unverified and confirm at
  [aws.amazon.com/sqs/pricing](https://aws.amazon.com/sqs/pricing/) before they
  go into [[cost-model]].* The free-tier and 64 KB-chunking facts above **are**
  from the AWS page.

**What actually costs money in a multi-region SQS design:**

| Item | When it applies | Rough magnitude |
|---|---|---|
| Empty standby queues + DLQs | Always | **£0** |
| Standby consumers polling an empty queue | If you choose to poll | Long polling at 20s = ~130k requests/month per consumer per queue. Cheap, but multiply by (queues × consumers × regions) before dismissing it. |
| Customer-managed KMS key in standby | SSE-KMS only | ~$1/month per key + per-request KMS charges. **£0 with SSE-SQS.** |
| SNS cross-region fan-out — transfer | Option (b) | ~$0.02/GB EU↔EU, charged to the *sending* region. Verify against AWS. |
| SNS cross-region fan-out — double processing | Option (b) with live standby consumers | Duplicates the compute cost of the workload. **Usually the largest line item, and usually forgotten.** |
| CloudWatch alarms | Always | Pennies per alarm per month |

**The levers, in order of size:**

1. **Don't run standby consumers** unless the analysis in
   [[messaging-in-flight-data-loss]] says you should. Duplicated compute dwarfs
   every other cost here.
2. **Use SSE-SQS instead of SSE-KMS** wherever the data classification allows.
   Free, and deletes an entire failure mode.
3. **Restrict SNS fan-out to the handful of queues that need it.** The cost is
   proportional to bytes crossing the region boundary; applying it estate-wide is
   how a messaging bill becomes a transfer bill.
4. **Long polling (`receive_wait_time_seconds = 20`) everywhere.** Short polling
   on an idle queue generates constant empty-receive requests for nothing. This
   is a single-region optimisation too, but it becomes twice as valuable when
   you have twice as many regions.

**Compared to the rest of the estate, SQS is a rounding error.** The interesting
cost question in this family is [[aws-eventbridge]]'s (archives are billed on
GB processed and GB stored, and cross-region events carry a per-million
replication charge) and [[aws-eks]]'s standby capacity. SQS is not where the
money is, which is another reason to pre-provision generously and not
over-engineer the mirroring.

## Open questions

1. **What is the p99 of `ApproximateAgeOfOldestMessage` on each production
   queue?** This single number decides whether option (a) is defensible per
   queue, and nobody has measured it. It is a one-day piece of work with a
   month of history and it should happen before any of this design is committed
   to.
2. **Which queues carry messages that are not re-derivable from a database?**
   That list *is* the list of queues that need option (b) or (d). If the answer
   is "none", this note's recommendation collapses to "create empty mirrors,
   done", and the messaging workstream is a week rather than a quarter.
3. **Are consumers idempotent today?** Option (d), and the safe version of every
   other option, depends on it. The honest answer is usually "some are, nobody
   knows which".
4. **Is any queue's retention set below 4 days?** A queue with 1-hour retention
   has an undocumented RPO of zero and its owner probably doesn't know.
5. **Can an SNS FIFO topic have a cross-region SQS FIFO subscriber?** Not
   confirmed in AWS docs — needs a sandbox test. It gates the only viable
   cross-region FIFO design.
6. **Do the app roles already scope `sqs:*` per region?** This flips the naming
   recommendation. Five-minute question to the IAM owner.
7. **Does the estate want to adopt ARC Region Switch** for consumer fencing, or
   roll its own? The Lambda ESM execution block is purpose-built for the
   double-processing problem in step 2 of the failover procedure. Worth a
   spike — decide in [[failover-orchestration]].
8. **Who owns the staleness policy?** "Is a 6-hour-old `charge-card` message
   still valid?" is a product question that infra cannot answer, and the drain
   script cannot be written without it.

## Sources

- [Amazon SQS — Active-Active, AWS Disaster Recovery Workshop](https://disaster-recovery.workshop.aws/en/services/app_integration/sqs/active-active.html)
  — AWS's own statement that queues have no auto-replication to another region,
  and the instruction to plan for message loss and idempotent consumers. The
  primary citation for this note's central claim.
- [Sending Amazon SNS messages to an SQS queue or Lambda function in a different Region](https://docs.aws.amazon.com/sns/latest/dg/sns-cross-region-delivery.html)
  — the official basis for option (b), including the opt-in-region service
  principal rule that affects the CA pair.
- [aws-samples/sample-sns-sqs-multi-region](https://github.com/aws-samples/sample-sns-sqs-multi-region)
  — AWS's own reference implementation of SNS fan-out to queues in two regions.
  Note it assumes manual producer failover and does not address ordering or
  duplicate semantics.
- [Amazon SQS quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-quotas.html)
  and [message quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html)
  — retention range (60s–14 days, default 4 days) and the 120,000 in-flight cap.
- [Amazon SQS increases in-flight limit for FIFO queues from 20K to 120K](https://aws.amazon.com/about-aws/whats-new/2024/11/amazon-sqs-increases-in-flight-limit-fifo-queues)
  — dates the FIFO in-flight change; relevant if you read older capacity docs.
- [Exactly-once processing in Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-exactly-once-processing.html)
  — what FIFO's dedup actually guarantees and the 5-minute window.
- [StartMessageMoveTask API reference](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_StartMessageMoveTask.html)
  — same-region-only redrive, and the restriction to SQS-sourced DLQs.
- [Using dead-letter queues in Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
  — the same-account-and-region constraint on DLQs.
- [Amazon SQS pricing](https://aws.amazon.com/sqs/pricing/)
  — free tier (1M requests/month), 64 KB payload chunking, no charge for
  same-region data transfer. The rate table did not render for me; the $0.40/$0.50
  figures in the Cost section are third-party and unverified.
- [ARC Region Switch adds Lambda event source mapping execution block](https://aws.amazon.com/about-aws/whats-new/2026/05/region-switch-lambda-esm-execution-block/)
  — AWS's purpose-built mechanism for toggling SQS/Kinesis/DDB-Streams/MSK
  consumers during an active-passive failover, including "ungraceful" mode when
  the primary is unreachable. Directly addresses the double-processing risk.
- [hashicorp/terraform-provider-aws #44777 — cannot import SQS queue from another region](https://github.com/hashicorp/terraform-provider-aws/issues/44777)
  — evidence that cross-region SQS addressing is awkward in the tooling too.
- [aws_sqs_queue — Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/sqs_queue.html)
  — the resource schema. **Check your pinned provider version** for whether
  `redrive_allow_policy` is a separate resource and which FIFO attributes are
  ForceNew; the registry page did not render fully for me.

### Searched for and did not find

- **Any AWS announcement of native SQS cross-region replication.** It does not
  exist. Several high-ranking third-party articles describe it as though it
  does; they are wrong.
- **A published engineering postmortem or case study of a company losing SQS
  messages in a regional failover.** No public example found. The absence is
  itself informative: either it happens rarely, or — more likely — the losses are
  small enough that nobody writes them up, which supports this note's central
  argument that the exposure is seconds rather than hours.
- **AWS confirmation that SNS FIFO topics support cross-region SQS FIFO
  subscribers.** Not stated either way in the cross-region delivery docs.
