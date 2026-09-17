---
title: Amazon EventBridge — Multi-Region
service: eventbridge
tags: [service, multi-region, eventbridge, messaging, eventing]
status: researched
replication: native (cross-Region event bus targets; global endpoints) + manual (archive/replay)
rpo_achievable: "seconds with a cross-Region bus target; ~360s (max 420s) with global endpoints; up to the archive's coverage window if you replay"
rto_achievable: "< 1 min for the bus itself if pre-provisioned; replay-based recovery is measured in *hours*, not minutes"
meets_targets: conditional — RTO yes, RPO yes, but **not in `ca-west-1`**, which has no archive, no replay, no global endpoints and no Pipes
updated: 2026-09-17
---

# Amazon EventBridge — Multi-Region

> **Read [[aws-sqs]] and [[messaging-in-flight-data-loss]] first.** They establish
> the estate's position on messaging: queues do not replicate, the exposure is
> *seconds of pending work* rather than hours of committed state, and the right
> default is an empty mirror plus idempotent, replayable producers. This note
> does not repeat that argument. It covers the one thing EventBridge has that
> SQS and SNS do not: **a genuinely first-class, AWS-native cross-region
> primitive**, and a second-order one (archive/replay) that people reach for as
> an RPO tool and usually misunderstand.

## TL;DR

- **EventBridge is the only service in the messaging family with a real
  cross-region primitive.** A rule in `eu-west-1` can name an event bus in
  `eu-west-2` as a target, natively, with no code and no relay. Latency is
  seconds. This is the strongest option in the whole [[aws-sqs]] /
  [[aws-sns]] / EventBridge triangle and it is the one to build on.
- **But only event buses cross regions.** Every other target type — SQS, Lambda,
  Step Functions, API destinations — must be in the same region as the rule. The
  pattern is therefore always *bus → remote bus → local rule → local target*, a
  two-hop shape you must design for. And **you cannot chain a third hop**: AWS
  explicitly will not forward an event received from another bus on to a third
  bus.
- **Archive and replay is a real RPO tool, with two hard constraints that decide
  the whole design.** (1) *"Archive events can only be replayed to the source
  event bus"* — so an archive in `eu-west-1` is useless to you when `eu-west-1`
  is down. **You must archive on the standby bus, fed by the cross-region
  target.** (2) Replay is slow, unordered, minute-bucketed, and re-delivers with
  a `replay-name` metadata field that changes the event shape. It is a
  **recovery** tool, not a failover tool: it will not fit inside a 15-minute RTO
  and should never be on the RTO critical path.
- **Global endpoints still exist and are not deprecated**, but they are a
  different product from what this estate needs: they fail over *ingestion*
  automatically on a Route 53 health check with an RTO/RPO AWS states as
  **360 seconds, max 420**. They demand SDK changes (CRT + SigV4A +
  `endpointId`), identically-named custom buses, and — fatally for the CA pair —
  **they are not available in `ca-west-1`**.
- **The thing that will bite:** `eu-west-2`'s EventBridge quotas are roughly
  **one eighth** of `eu-west-1`'s. `PutEvents` is 10,000 TPS in Ireland and
  1,200 TPS in London; invocations are 18,750/s vs 2,250/s. Fail over at full
  production volume and you will throttle in the standby. Both are adjustable;
  neither is adjusted by default; **raising a quota is a support-ticket-shaped
  lead time, not a 15-minute one.** Raise them now. (`ca-west-1` is worse:
  400 TPS `PutEvents`, 750/s invocations.)

---

## Does this service cross regions at all?

Yes — more than anything else in the messaging family, and it is worth being
precise about exactly which pieces do.

| Thing | Regional or global? | Crosses regions? |
|---|---|---|
| Event bus | Regional (`arn:aws:events:<region>:<acct>:event-bus/<name>`) | **Yes, as a target.** This is the whole feature. |
| Rule | Regional, attached to one bus | No. Rules are per-bus, per-region. Mirror them with Terraform. |
| Target — event bus | — | **Yes** |
| Target — SQS, Lambda, Step Functions, ECS, API destination, everything else | Regional | **No.** Must be in the rule's region. |
| Archive | Regional, bound to one source bus | **No.** Replay is to the source bus only. |
| Replay | Regional | **No** |
| Global endpoint | A regional resource with a global DNS name | Spans exactly two regions |
| Scheduler schedule | Regional | Schedule is regional; see the Scheduler section |
| Pipe | Regional | Source and target both regional; not available in `ca-west-1` at all |
| Schema registry | Regional | No |

The `eb-targets` documentation states the rule plainly for cross-account, and
the same regionality governs cross-region: *"If another account is in the same
Region and has granted you permission, then you can send events to that
account."* The cross-region escape hatch is bus-to-bus and nothing else.

### The no-chaining rule

This one catches people building hub-and-spoke designs. From the AWS docs on
bus-to-bus routing:

> *"EventBridge can't route events received from a sender event bus to a third
> event bus."*

and from the cross-account page, said twice for emphasis:

> *"If a receiver account sets up a rule that sends events received from a
> sender account on to a third account, these events are not sent to the third
> account."*
>
> *"If you have three event buses in the same account, and set up a rule on the
> first event bus to forward events from the second event bus to a third event
> bus, those events are not sent to the third event bus."*

**Consequence for this estate:** the standby bus is a terminus. It can fan out to
local targets, but it cannot forward on. You cannot build
`app-bus → central-bus → standby-bus`. If you want events on the standby bus,
the *originating* bus must target it directly. In a multi-bus estate that means
**every bus that matters needs its own cross-region rule**, which is a
`for_each` in Terraform, not an architecture.

---

## Replication / mirroring options

### Option (a) — Do nothing: an empty mirror bus and mirrored rules

Create the bus, the rules and the local targets in `eu-west-2` with the same
Terraform. Nothing flows through it while the primary is healthy. At failover,
producers `PutEvents` against the standby bus and the standby rules fire.

**Cost: £0.** EventBridge bills per event ingested and per invocation. A bus with
no traffic generates no bill. There is no per-bus, per-rule or per-month charge.

**What you lose:** events that were in flight in the primary at the moment it
died — between `PutEvents` and target delivery. EventBridge retries target
delivery for up to `MaximumEventAgeInSeconds`, whose **maximum and effective
default is 86,400 seconds (24 hours)**, with up to **185 retry attempts**. So an
event that made it onto the bus and whose target is a *local* resource in the
failed region is retried into a region that is down, for a day, and then
dropped — unless the target had a DLQ, in which case it lands in a DLQ in the
failed region, which is [[aws-sqs]]'s "least recoverable thing in the estate"
problem verbatim.

**Verdict: this is the default, exactly as for SQS.** It is correct for
everything where the event is a signal rather than a record. See
[[messaging-in-flight-data-loss]]'s three-row classification — apply it per bus,
per rule.

### Option (b) — Cross-region event bus target (the recommended primitive)

Add a rule on the primary bus whose target is the standby bus's ARN. Every
matching event lands on both buses within seconds. This is the option this note
exists to recommend, and it gets its own section below.

### Option (c) — Archive on the standby bus, replay after failover

Combine (b) with an archive attached to the **standby** bus. The standby bus
receives every event via the cross-region rule; the archive captures them; the
standby's rules are either absent or disabled so nothing is *processed*. At
failover you enable the standby rules and replay the archive for the window you
need to recover.

This is the "archive and replay as an RPO tool" design and it is genuinely good
— with caveats large enough to need their own section. Also below.

### Option (d) — Global endpoint

Let AWS fail over ingestion for you based on a Route 53 health check, with
optional event replication to the secondary. Covered in full below. Short
version: a real feature, wrong shape for this estate, and unavailable in
`ca-west-1`.

### Option (e) — Producer-side dual publish

The application calls `PutEvents` against both regions. Mentioned only to
dismiss it: it doubles producer latency on the critical path, it has no
transactional guarantee across the two calls (you can succeed in one region and
fail in the other with no way to reconcile), and option (b) does the same job
server-side for the same $1.00/M. **Do not build this.** The only case where it
wins is when you need the *producer* to survive the primary region being
unreachable — and that case is better served by a global endpoint or by routing
the producer itself, not by the event layer.

---

## Cross-region event bus targets, in depth

### What it is

Launched **April 2021** with three supported destination regions (N. Virginia,
Oregon, Ireland), expanded **November 2021** to a list of 21 commercial regions.
The AWS announcement names them:

> *"US East (Ohio), US East (N. Virginia), US West (Oregon), US West (N.
> California), Canada (Central), Europe (Stockholm), Europe (Paris), Europe
> (Ireland), Europe (Frankfurt), Europe (London), Europe (Milan), Middle East
> (Bahrain), Africa (Cape Town), Asia Pacific (Mumbai), Asia Pacific (Tokyo),
> Asia Pacific (Seoul), Asia Pacific (Singapore), Asia Pacific (Hong Kong), Asia
> Pacific (Osaka), Asia Pacific (Sydney), and South America (São Paulo)."*

**For the three pairs in this estate:**

| Pair | Source | Destination | Cross-region bus target supported? |
|---|---|---|---|
| EU | `eu-west-1` | `eu-west-2` | **Yes** — London is on the list |
| US | `us-east-1` | `us-west-2` | **Yes** — Oregon is on the list |
| CA | `ca-central-1` | `ca-west-1` | **Unverified — see below** |

**`ca-west-1` is the open question.** Calgary launched in December 2023, two
years after that announcement, so it cannot be on the 2021 list. The current
docs page (`eb-cross-account`) only says *"as long as the destination Region is
a supported cross-Region destination Region"* and links to a page that does not
itself carry a region list. **I could not find a current, authoritative,
enumerated list of supported cross-region destination regions on any AWS page.**
Some secondary sources summarise it as "all regions except GovCloud and China";
that phrasing does not appear verbatim on the AWS pages I fetched.

**Action: test it.** `aws events put-targets` with a `ca-west-1` bus ARN from
`ca-central-1` takes five minutes and settles it. If it fails you will get
`ValidationException: Cross-region api call is not allowed` (see the Terraform
gotcha below for what that error looks like). Record the answer in
[[region-pair-selection]] — and note that it is moot for any design that also
needs archive/replay, because `ca-west-1` has neither.

### IAM: the role is on the *sender*, the policy is on the *receiver*

Two distinct grants, and both are needed even for same-account cross-region.

**1. An IAM role that EventBridge assumes, in the sender's region**, with
`events:PutEvents` on the destination bus ARN, trusted by `events.amazonaws.com`.
This is passed as `role_arn` on the target. For same-account cross-region this
role is the *only* thing that is strictly required; the resource policy matters
for cross-account.

**2. A resource-based policy on the receiving bus** allowing the sending
account/org/principal to `events:PutEvents`. In a single-account estate you can
usually rely on the default bus's implicit same-account permission, but **be
explicit anyway** — an explicit policy is what makes the grant survive an
account-boundary change later, and it's the thing you'll wish existed when
someone splits prod into its own account.

Note AWS's dated behavioural change: *"EventBridge now requires all new cross
account event bus targets to add IAM roles. This only applies to event bus
targets created after March 2, 2023."* If you inherit an old cross-account
target with no role, do not assume you can recreate it as-is.

### Ordering and duplication semantics

This is the part that is under-documented and that you must not assume away.

- **No ordering guarantee.** EventBridge does not order events on a bus, does
  not order delivery to targets, and therefore does not order events across a
  cross-region hop. Two events published 1ms apart may arrive on the standby bus
  in either order. **I found no AWS statement offering any ordering guarantee for
  event buses.** The only ordering statement in the EventBridge FAQ is about
  *Pipes*: *"EventBridge Pipes will maintain the order of events received from an
  event source when sending those events to a destination service"* — that is a
  Pipes property, from an ordered source, and it does not transfer to buses.
- **At-least-once, therefore duplicates.** The FAQ states at-least-once delivery
  explicitly for Scheduler. For buses, the retry policy (up to 185 attempts over
  24h) is itself a duplicate generator: a target that succeeds but whose response
  is lost gets retried. **Assume duplicates on both sides of the region
  boundary.** Consumers must be idempotent — which is the same conclusion
  [[messaging-in-flight-data-loss]] reaches for every other reason.
- **The delivered event is unchanged.** The cross-region routing blog states:
  *"The delivered event is identical to the original event, and does not contain
  any additional metadata or attributes."* This is a double-edged fact. Good:
  your standby rules' event patterns work unmodified. Bad: **there is no marker
  on the event saying "this arrived from the other region"**, so a standby rule
  cannot distinguish a replicated event from a natively-published one, and you
  cannot write a pattern that excludes replicated events. The `source` and
  `region` fields still say the *original* region, so you can match on
  `region` if you need to — that is the only discriminator you get, and it
  changes meaning after a failover (post-failover, events genuinely originate in
  the standby region and carry its `region` value). **Do not build fencing logic
  on the `region` field.** Fence with rule state, per [[split-brain-and-fencing]].
- **Duplication across a failover is near-certain.** During the window where both
  regions are briefly live, the primary's rule is still replicating to the
  standby *and* producers have started publishing to the standby directly. Every
  event in that window exists twice on the standby bus, with different event IDs.
  Idempotency is not optional.

### The two-hop shape

Because only buses cross regions, the delivered architecture is always:

```
  eu-west-1                               eu-west-2
  ---------                               ---------
  producer --PutEvents--> [default bus]
                               |
                          rule "replicate-to-standby"
                          target = arn:aws:events:eu-west-2:...:event-bus/default
                          role_arn = <replication role>
                               |
                               +-------(seconds, AWS-managed)-------> [default bus]
                                                                          |
                                                                     rules (mirrored,
                                                                     state configurable)
                                                                          |
                                                        +-----------------+-----------------+
                                                        |                 |                 |
                                                   local SQS         local Lambda      archive
```

Three design consequences:

1. **The standby's rules are the fencing mechanism.** Mirror every rule into the
   standby, and control whether the standby *acts* by setting rule state
   (`ENABLED` / `DISABLED`) rather than by controlling replication. Flipping a
   rule's state is a single, fast, idempotent control-plane call that is easy to
   automate and easy to verify. This is the EventBridge analogue of [[aws-sqs]]'s
   "run the consumers, disable the subscriptions".
   *Caveat:* rule state is a control-plane operation and AWS warns *"When you add
   targets to a rule and that rule runs soon after, any new or updated targets
   might not be immediately invoked. Allow a short period of time for changes to
   take effect."* Budget seconds, not zero, and verify rather than assume.
2. **You pay twice.** Every cross-region event is billed as a custom event to the
   sending account ($1.00/M) *plus* the standard ingestion on the receiving bus,
   *plus* inter-region data transfer at standard AWS rates. See Cost.
3. **A replicated event with no matching rule on the standby is silently
   discarded.** That is not a bug — EventBridge discards unmatched events by
   design — but it means a drifted rule set in the standby fails *silently*.
   Alarm on this: compare `MatchedEvents` against ingested events on the standby
   bus, or simply alarm on the standby's `TriggeredRules` being zero when
   replication is known to be flowing. A standby bus receiving thousands of
   events and matching none of them looks perfectly healthy in the console.

### Should you replicate continuously, or only archive?

A genuine fork.

**Branch A — replicate and process (standby rules enabled).** Both regions do
the work. This is active/active for the event tier, which the brief explicitly
does not want, and it doubles the compute cost and the side-effect blast radius.
**Reject**, except for idempotent, side-effect-free targets such as an audit
archive or a metrics sink.

**Branch B — replicate but do not process (standby rules disabled, except an
archive rule).** The standby bus receives everything, an archive captures
everything, nothing is invoked. At failover you enable the rules. This gets you:
a warm, continuously-exercised replication path (which proves the IAM role, the
bus policy and the KMS grants every single day — the same continuous-validation
argument [[messaging-in-flight-data-loss]] makes for standby consumers), a
recoverable event history in the surviving region, and zero double-processing.

**Recommendation: Branch B.** The cost is $1.00/M events plus transfer plus
archive storage, and the benefit is that the cross-region path is the one part
of your DR design that is *proven working right now* rather than proven working
at a game day six months ago.

---

## Archive and replay as an RPO tool

This is the most interesting idea in this note and the one most likely to be
implemented wrongly. Two constraints decide everything.

### Constraint 1: replay is to the source bus only

From the AWS archive documentation, unambiguously:

> *"Archive events can only be replayed to the source event bus."*

and

> *"Each archive receives events from a single source event bus. You cannot
> change the source event bus once an archive is created."*

**Therefore: an archive attached to the `eu-west-1` bus can only ever replay into
the `eu-west-1` bus.** When `eu-west-1` is the thing that has failed, that
archive is unreachable and, even if reachable, useless — replaying into a dead
region's bus achieves nothing.

**The only correct design is to archive on the standby bus.** The cross-region
rule feeds the standby bus; an archive on the *standby* bus captures the stream;
after failover you replay into the standby bus, which is now the live bus, and
the (now-enabled) standby rules process the replayed events.

This single fact inverts the naive design. Almost everyone's first instinct is
"archive in the primary, ship the archive to the standby". That is not a thing
AWS supports. There is no archive export, no cross-region archive copy, and no
way to re-point an archive.

*(A second archive on the primary bus is still worth having, for the ordinary
single-region debugging use case. Just do not confuse it with a DR asset.)*

### Constraint 2: replay is slow, unordered, and changes the event

The AWS "considerations when replaying" section is short and every line matters:

- **Ordering.** *"Events aren't necessarily replayed in the same order that they
  were added to the archive. A replay processes events to replay based on the
  time in the event, and replays them on one minute intervals. If you specify an
  event start time and an event end time that covers a 20 minute time range, the
  events are replayed from the first minute of that 20 minute range first. Then
  the events from the second minute are replayed."* So: **ordered to the minute,
  arbitrary within the minute.** If your consumers care about order, replay
  breaks them. (Which is another reason not to put anything order-sensitive on a
  bus in the first place — see the FIFO discussion in [[aws-sqs]].)
- **The replayed event is not the original event.** *"When EventBridge sends an
  event from an archive to the source event bus during a replay, it adds a
  metadata field to the event, `replay-name`, which contains the name of the
  replay."* This is useful — you can match on it, and you should log it — but it
  also means **anything that hashes or signs the whole event payload will see a
  different event.** Check your idempotency-key derivation: if it is computed
  over the whole event JSON, replayed events will produce different keys and your
  idempotency will silently fail exactly when you need it.
- **Archives are not instantly consistent with the bus.** *"There may be a delay
  between an event being received on an event bus and the event arriving in the
  archive. We recommend you delay replaying archived events for 10 minutes to
  make sure all events are replayed."* **That ten minutes is two-thirds of your
  entire 15-minute RTO.** It is a decisive reason replay cannot be on the RTO
  critical path.
- **You cannot trust `DescribeArchive` counts in the moment.** *"The `EventCount`
  and `SizeBytes` values of the `DescribeArchive` operation have a reconciliation
  period of 24 hours."* So during an incident you cannot ask the archive "how
  many events do you have?" and get a trustworthy answer. Plan your replay by
  *time window*, never by count.
- **Concurrency cap:** *"You can have a maximum of ten active concurrent replays
  per account per AWS Region."* Ten. Across the whole account. If your recovery
  plan is "replay each of our forty buses in parallel", it isn't.
- **Replays are garbage-collected.** *"EventBridge deletes replays after 90
  days."* The replay *records* go, not the archived events — but your audit trail
  of what you replayed during an incident has a 90-day life unless you capture it
  yourself. Log `StartReplay` calls to a durable store.

### How fast is a replay, really?

**I could not find a published AWS throughput figure for replay, nor any
third-party benchmark. No public data found.** What the docs give you is the
governing constraint:

> *"Events are replayed based on, but separate from, the `PutEvents` transactions
> per second limit for the AWS account."*

So the ceiling is the destination region's `PutEvents` quota — and this is
exactly where the standby-region quota asymmetry bites. Working the arithmetic
from the published quotas (these are the *ceilings*, and real replay throughput
will be lower, possibly much lower — treat these as optimistic bounds, not
measurements):

| Destination | `PutEvents` TPS quota | Optimistic ceiling for replaying 2h of events at 500 eps (3.6M events) |
|---|---|---|
| `eu-west-1` | 10,000/s | ~6 min |
| `eu-west-2` | **1,200/s** | **~50 min** |
| `us-east-1` | 10,000/s | ~6 min |
| `us-west-2` | 10,000/s | ~6 min |
| `ca-central-1` | 600/s | ~100 min |
| `ca-west-1` | 400/s (*"each of the other supported Regions"*) | ~150 min — **and archive/replay is not available there at all** |

Read the `eu-west-2` row carefully. **The standby region — the one you replay
into — has one eighth of the primary's ingestion quota.** Replaying two hours of
Ireland-volume traffic into London is, at absolute best, an hour. In practice,
add the mandatory 10-minute archive lag, the minute-bucketing, and the fact that
replay shares capacity with the live traffic you have just failed over. **Replay
is an hours-scale recovery, not a minutes-scale failover.**

### The verdict on replay as an RPO tool

**It is a legitimate and, for a 2-hour RPO, well-matched tool — but it belongs
after the RTO clock stops, not before it.** Frame it exactly as [[aws-sqs]]
frames its drain script:

> Failover restores *service* in 15 minutes. Replay restores *completeness* in
> the following hours.

That framing is defensible to a risk committee and it is honest. What is not
defensible is a runbook that says "fail over, then replay the last two hours"
inside a 15-minute RTO. It will not happen.

**Concretely, the recommended shape:**

1. Cross-region rule replicates every event from the primary bus to the standby
   bus, continuously. (Option b.)
2. An archive on the **standby** bus captures everything, with retention set to
   at least 30 days — long enough that a post-incident "actually, we need to
   replay Tuesday afternoon too" is possible.
3. Standby rules exist but are `DISABLED`. Nothing is processed.
4. **At failover:** enable the standby rules. Point producers at the standby bus.
   Service is restored. *Do not replay yet.* New events flow and are processed
   normally. The RTO clock stops here.
5. **After the dust settles** (and at least 10 minutes after the last event the
   primary ingested): start a replay bounded to the window between "last event we
   know was processed successfully" and "first event the standby processed
   natively". Consumers are idempotent, so overlap is safe; err on the side of
   overlapping.
6. Use `DescribeReplay`'s `EventLastReplayedTime` to track progress. Expect it
   to take a while.

The hard part of step 5 is knowing the start timestamp, and the honest answer is
that you usually won't know it precisely. **Overlap generously and rely on
idempotency.** If you cannot rely on idempotency, replay is not available to you
as a tool at all — which is one more entry on the long list of things in this
vault that reduce to "make consumers idempotent".

### An archive is also your evidence

[[messaging-in-flight-data-loss]] makes the point that losing a region often
means losing the evidence of what you lost. An archive on the standby bus is a
**cross-region, durable, queryable record of every event the primary ingested**,
which survives the primary's destruction. Even if you never replay it, it lets
you answer "what did we drop?" at the post-incident review. That alone is worth
$0.023/GB-month.

---

## Global endpoints: current status

### Does it still exist?

**Yes.** The feature page
(`docs.aws.amazon.com/eventbridge/latest/userguide/eb-global-endpoints.html`) is
live and current, global endpoints appear as a column in the current EventBridge
feature-availability-by-region table, and `aws_cloudwatch_event_endpoint` is a
current Terraform resource. **I found no deprecation notice, no end-of-support
announcement and no "no longer accepting new customers" language.** I also found
no new-feature announcements for it since its April 2022 GA, which is a
reasonable signal that it is stable-and-unloved rather than actively invested in
— but that is an inference, not a finding.

### What it actually does

You create a *global endpoint* spanning exactly two buses with **identical
names** in two regions in the **same account**. You attach a Route 53 health
check to the primary. Producers call `PutEvents` against the endpoint's URL
(passing `endpointId`) rather than a regional endpoint. When the health check
goes unhealthy, ingestion is routed to the secondary bus automatically. With
*event replication* enabled, every custom event is sent to **both** buses via
AWS-managed rules.

AWS's stated objectives:

> *"With global endpoints, if you follow our prescriptive guidance for alarm
> configuration, you can expect the RTO and RPO to be 360 seconds with a maximum
> of 420 seconds."*

Six minutes, worst case seven. Against a 15-minute RTO that is comfortable —
**for the ingestion layer only.** It does nothing for your compute, your
database or your DNS.

### Why it is the wrong fit here, in order of severity

1. **`ca-west-1` does not support global endpoints.** The feature-availability
   table lists Canada (Central) with a tick and Canada West (Calgary) without
   one. The CA pair cannot use this feature at all. Since the estate wants one
   pattern across three pairs, that is close to disqualifying on its own.
2. **It requires a producer SDK change.** *"You'll need to have the AWS Common
   Runtime (CRT) library installed for your specific SDK"*, *"you'll need to add
   the `endpointId` and `EventBusName` to any `PutEvents` calls"*, and it signs
   with **SigV4A**. There is a real trap in the docs: *"If you request temporary
   credentials from the global AWS STS endpoint (sts.amazonaws.com), AWS STS
   vends credentials which, by default, do not support SigV4A."* That is a
   silent, environment-dependent failure waiting in an IRSA/EKS setup. C++
   support is still listed as "coming soon", four years after GA.
3. **It fails over ingestion independently of everything else.** A Route 53
   health check going unhealthy for 6 minutes will move your *events* to
   `eu-west-2` while your *API*, your *database writes* and your *DNS* are still
   in `eu-west-1`. You now have a split system that nobody decided to split.
   For an active/passive estate with a deliberate, human-decided failover — which
   is what the brief describes — **automatic, independent, per-service failover
   is a liability, not a feature.** This is the deepest objection and it is
   architectural, not a gap.
4. **Replication is all-or-nothing and only covers custom events.** You cannot
   select which events replicate. Costs scale with total event volume.
5. **It requires custom buses with identical names.** *"If you're using custom
   buses, you'll need a custom bus in each Region with the same name and in the
   same account for failover to work properly."* Fine if you already name buses
   identically across regions (which [[aws-sqs]]'s naming recommendation would
   have you do anyway), annoying if you don't.
6. **Recovery needs replication on.** *"If you don't have event replication
   enabled, you'll have to manually reset the Route 53 health check to 'healthy'
   before events will go back to the primary Region."* So the cheap
   (replication-disabled) configuration has a manual failback step.

### Where it *would* be right

If the estate ever wants an **automatic, sub-7-minute, ingestion-only** failover
for a specific high-value event stream — say, inbound payment webhooks where
losing six minutes of events is worse than a brief split — a global endpoint on a
dedicated bus for that one stream is a good, cheap answer. It's a scalpel, not
the estate-wide pattern.

**Recommendation: do not adopt global endpoints as the estate pattern. Use
option (b) + (c): cross-region bus target plus a standby-side archive.** Revisit
global endpoints only for a narrowly-scoped stream in the EU and US pairs, never
for CA. Record the decision in [[open-decisions]].

---

## EventBridge Scheduler across regions

Scheduler is available in **every region in the feature table, including
`ca-west-1`** — it is one of only three EventBridge features Calgary has (buses,
Scheduler, schema registry).

The multi-region question for Scheduler is not "how do I replicate it" but
**"what happens if both copies fire?"** — and it's the sharper question because
Scheduler's whole job is to cause side effects on a timer.

- A schedule is regional. There is no cross-region schedule and no native
  replication. You mirror it with Terraform, same as a rule.
- Scheduler targets are regional too; the universal-target mechanism calls an AWS
  API, and those calls are region-local. To hit something in another region you
  again go via a bus.
- **Delivery is at-least-once.** The AWS FAQ: *"EventBridge Scheduler provides
  at-least-once event delivery to targets, meaning that at least one delivery
  succeeds with a response from the target."* A schedule that is enabled in both
  regions will fire in both. For a "send the weekly invoice run" schedule that is
  a duplicate-billing incident.

**Therefore: every mirrored schedule must be created with `state = "DISABLED"` in
the standby, and enabling them must be an explicit, ordered step in the failover
runbook.** This is a different default from rules, where you might reasonably
leave a replication rule enabled. For schedules the default must be off.

Two further notes:

- **There is no "catch-up" on enable.** A schedule that was disabled through its
  fire window does not fire retroactively when you enable it. So a failover that
  happens at 02:55 and a schedule that fires at 03:00 daily is fine; a failover
  at 03:05 means the 03:00 run happened in neither region (the primary was dead,
  the standby was disabled) and **nobody will tell you.** Enumerate every
  schedule's fire time and add "which scheduled jobs did we miss?" as an explicit
  runbook step in [[failover-runbook-template]]. This is a real, silent,
  guaranteed gap and it is not on anyone's list.
- Scheduler has its own quota page separate from the EventBridge quotas page;
  check the standby region's limits there too.

## EventBridge Pipes across regions

- **Pipes is not available in `ca-west-1`.** Blank cell in the feature table.
  Any design using Pipes is EU/US-only.
- Concurrent pipe executions per account: **3,000** in `us-east-1`, `us-west-2`
  and `eu-west-1`; **1,000** in every other listed region including `eu-west-2`
  and `ca-central-1`. Another 3× standby asymmetry to pre-raise.
- Pipes preserves source ordering (*"EventBridge Pipes will maintain the order of
  events received from an event source when sending those events to a destination
  service"*), which makes it the only ordered primitive in this family — and
  therefore the one whose semantics you lose hardest if you try to span regions
  with it. **Don't.** Keep a pipe entirely within one region; mirror the pipe
  definition into the standby with Terraform, pointing at that region's own
  source and target.
- A Pipe whose source is an SQS queue is a consumer, so everything in
  [[messaging-in-flight-data-loss]] about fencing standby consumers applies to it.
  A pipe in the standby, enabled, polling a queue, is a live consumer.

---

## RPO / RTO analysis

### Against RTO 15m

| Step | Time | Pre-provisioned? |
|---|---|---|
| Standby bus exists | 0 s | **Yes** — Terraform |
| Standby rules exist (disabled) | 0 s | **Yes** |
| Enabling standby rules | seconds (control plane; allow propagation) | No — a failover action |
| Enabling standby schedules | seconds | No — a failover action |
| Producers repointed to standby bus | seconds–minutes | Config; the actual work |
| Cross-region replication path | already flowing | **Yes** |
| **Replay of the archive** | **tens of minutes to hours** | **Explicitly NOT on the RTO path** |

**Passes, comfortably, provided replay is excluded from the RTO definition.**
The EventBridge contribution to RTO is a handful of `EnableRule` calls. Nothing
is provisioned from cold.

The quota asymmetry is the one thing that can turn a passing RTO into a failing
one: if you fail `eu-west-1`'s 3,000 events/s into `eu-west-2`'s 1,200 TPS
`PutEvents` limit, producers get throttled and the failover *looks* successful
while dropping load. **Raise the standby quotas before you need them.**

### Against RPO 2h

| Design | RPO for events |
|---|---|
| (a) Empty mirror bus | Everything in flight in the primary at the moment of failure; unbounded if the primary is gone permanently |
| (b) Cross-region bus target | **Seconds** — replication latency |
| (b)+(c) Plus standby archive | **Seconds**, and the events are also *recoverable and reprocessable* rather than merely delivered |
| (d) Global endpoint w/ replication | **360 s, max 420 s** (AWS's own figure) |

Option (b) is two to three orders of magnitude inside the 2h budget. **The 2h
RPO is not the binding constraint on EventBridge; it is comically loose for
it.** The binding constraints are RTO (fine) and correctness (duplicates and
ordering — not fine unless consumers are idempotent).

---

## Warm standby shape

| Resource in the standby | State while primary is healthy | Idle cost |
|---|---|---|
| Event bus(es) | Created, receiving replicated events | £0 for the bus; $1.00/M for ingested replicated events |
| Rules mirroring the primary's | Created, **`is_enabled = false`** | £0 |
| The archive rule / archive | **Active** | $0.10/GB processed + $0.023/GB-month stored |
| Targets (SQS queues, Lambdas) | Created, wired to disabled rules | See [[aws-sqs]] / [[aws-lambda]] |
| Target DLQs | Created, empty | £0 |
| Schedules | Created, **`state = "DISABLED"`** | £0 |
| Pipes | Created, **stopped** | £0 (EU/US only) |
| KMS key for archive encryption | Present in-region | ~$1/month per CMK |
| CloudWatch alarms | Active | Pennies |

**The standby EventBridge footprint is cheap.** The two real costs are the
replicated event volume and the archive. Both are proportional to traffic, both
are knobs (filter the replication rule's event pattern; shorten archive
retention), and both are small relative to [[aws-eks]] standby capacity.

---

## Terraform implementation

### The module

```hcl
# modules/eventbridge-bus/variables.tf

variable "name" {
  description = "Bus name. Identical in both regions — required if you ever want global endpoints, and consistent with the SQS naming decision."
  type        = string
}

variable "is_standby" {
  description = "True in the standby region. Disables rules and schedules, and turns on the standby-specific alarms."
  type        = bool
  default     = false
}

variable "replicate_to_bus_arn" {
  description = "ARN of an event bus in the OTHER region to replicate matching events to. Null disables replication. Only set this on the primary."
  type        = string
  default     = null
}

variable "replication_event_pattern" {
  description = "Event pattern deciding which events cross the region boundary. Defaults to everything with a `source`, which is every custom event. Narrow this to control cost."
  type        = string
  default     = null
}

variable "archive_retention_days" {
  description = "0 = indefinite (AWS default). Set on the STANDBY bus; an archive on the primary bus is a debugging tool, not a DR asset."
  type        = number
  default     = 30
}

variable "create_archive" {
  type    = bool
  default = false
}

variable "rules" {
  description = "Rule definitions, shared verbatim by both regions."
  type = map(object({
    event_pattern = string
    description   = optional(string)
    targets = map(object({
      arn      = string
      role_arn = optional(string)
      dlq_arn  = optional(string)
    }))
  }))
  default = {}
}

variable "kms_key_arn" {
  description = "CMK for archive encryption. MUST be a key in this region. See aws-kms."
  type        = string
  default     = null
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

```hcl
# modules/eventbridge-bus/main.tf

resource "aws_cloudwatch_event_bus" "this" {
  name = var.name
  tags = var.tags
}

# ---------------------------------------------------------------------------
# Cross-region replication. Set only on the primary.
# The role lives in the SENDER's region; the permission targets the RECEIVER's
# bus ARN. This is the single most important resource in this note.
# ---------------------------------------------------------------------------

resource "aws_iam_role" "replication" {
  count = var.replicate_to_bus_arn == null ? 0 : 1

  name = "eventbridge-xregion-${var.name}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "events.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })

  tags = var.tags
}

resource "aws_iam_role_policy" "replication" {
  count = var.replicate_to_bus_arn == null ? 0 : 1

  role = aws_iam_role.replication[0].id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = "events:PutEvents"
      Resource = var.replicate_to_bus_arn # cross-region ARN; region is in the ARN
    }]
  })
}

resource "aws_cloudwatch_event_rule" "replicate" {
  count = var.replicate_to_bus_arn == null ? 0 : 1

  name           = "${var.name}-replicate-cross-region"
  description    = "Replicate events to the paired standby region. See 02-services/aws-eventbridge.md"
  event_bus_name = aws_cloudwatch_event_bus.this.name

  # Default: every event that has a `source`, i.e. every custom event.
  # Narrow this to cut cross-region transfer and per-event charges.
  event_pattern = coalesce(
    var.replication_event_pattern,
    jsonencode({ source = [{ exists = true }] })
  )

  tags = var.tags
}

resource "aws_cloudwatch_event_target" "replicate" {
  count = var.replicate_to_bus_arn == null ? 0 : 1

  rule           = aws_cloudwatch_event_rule.replicate[0].name
  event_bus_name = aws_cloudwatch_event_bus.this.name
  target_id      = "cross-region-bus"

  arn      = var.replicate_to_bus_arn
  role_arn = aws_iam_role.replication[0].arn # REQUIRED for a bus target in another region

  # Retry into a region that may be unreachable. 24h is the max and the right
  # value here: a transient cross-region problem should self-heal.
  retry_policy {
    maximum_event_age_in_seconds = 86400
    maximum_retry_attempts       = 185
  }

  # DLQ is same-region (it lives with the SENDING rule), so this captures
  # events that could not be replicated. Alarm on it — it is your replication
  # health signal.
  dead_letter_config {
    arn = aws_sqs_queue.replication_dlq[0].arn
  }
}

resource "aws_sqs_queue" "replication_dlq" {
  count = var.replicate_to_bus_arn == null ? 0 : 1

  name                      = "${var.name}-xregion-replication-dlq"
  message_retention_seconds = 1209600 # 14 days, per the aws-sqs note

  # NOTE: EventBridge does not support SQS queues encrypted with an AWS *owned*
  # key as targets or as target DLQs. Use SSE-KMS with a CMK here.
  kms_master_key_id = var.kms_key_arn

  tags = var.tags
}

data "aws_iam_policy_document" "replication_dlq" {
  count = var.replicate_to_bus_arn == null ? 0 : 1

  statement {
    effect    = "Allow"
    actions   = ["sqs:SendMessage"]
    resources = [aws_sqs_queue.replication_dlq[0].arn]
    principals {
      type        = "Service"
      identifiers = ["events.amazonaws.com"]
    }
    condition {
      test     = "ArnEquals"
      variable = "aws:SourceArn"
      values   = [aws_cloudwatch_event_rule.replicate[0].arn]
    }
  }
}

resource "aws_sqs_queue_policy" "replication_dlq" {
  count     = var.replicate_to_bus_arn == null ? 0 : 1
  queue_url = aws_sqs_queue.replication_dlq[0].id
  policy    = data.aws_iam_policy_document.replication_dlq[0].json
}

# ---------------------------------------------------------------------------
# Bus resource policy. Explicit even in a single account — it is what survives
# a future account split, and it is what a security reviewer will ask for.
# ---------------------------------------------------------------------------

data "aws_caller_identity" "current" {}

resource "aws_cloudwatch_event_bus_policy" "this" {
  event_bus_name = aws_cloudwatch_event_bus.this.name

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowSameAccountCrossRegionPutEvents"
      Effect    = "Allow"
      Principal = { AWS = data.aws_caller_identity.current.account_id }
      Action    = "events:PutEvents"
      Resource  = aws_cloudwatch_event_bus.this.arn
    }]
  })
}

# ---------------------------------------------------------------------------
# Archive. On the STANDBY bus this is the DR asset; on the primary it is a
# debugging convenience. Replay is to the source bus only, which is the entire
# reason it must live on the standby.
# ---------------------------------------------------------------------------

resource "aws_cloudwatch_event_archive" "this" {
  count = var.create_archive ? 1 : 0

  name             = "${var.name}-archive"
  description      = "DR archive. Replay target is THIS bus and only this bus."
  event_source_arn = aws_cloudwatch_event_bus.this.arn
  retention_days   = var.archive_retention_days

  kms_key_identifier = var.kms_key_arn

  # Archive everything that has a source. Do not filter this down without
  # understanding that unfiltered events are simply not recoverable later.
  event_pattern = jsonencode({ source = [{ exists = true }] })
}

# ---------------------------------------------------------------------------
# Business rules — identical in both regions, DISABLED in the standby.
# Rule state is the fencing mechanism.
# ---------------------------------------------------------------------------

resource "aws_cloudwatch_event_rule" "this" {
  for_each = var.rules

  name           = "${var.name}-${each.key}"
  description    = try(each.value.description, null)
  event_bus_name = aws_cloudwatch_event_bus.this.name
  event_pattern  = each.value.event_pattern

  # THE fence. Standby rules exist, are fully wired, and do nothing.
  state = var.is_standby ? "DISABLED" : "ENABLED"

  tags = var.tags
}

resource "aws_cloudwatch_event_target" "this" {
  for_each = merge([
    for rule_key, rule in var.rules : {
      for target_key, target in rule.targets :
      "${rule_key}/${target_key}" => merge(target, { rule_key = rule_key, target_key = target_key })
    }
  ]...)

  rule           = aws_cloudwatch_event_rule.this[each.value.rule_key].name
  event_bus_name = aws_cloudwatch_event_bus.this.name
  target_id      = each.value.target_key
  arn            = each.value.arn
  role_arn       = try(each.value.role_arn, null)

  dynamic "dead_letter_config" {
    for_each = try(each.value.dlq_arn, null) == null ? [] : [1]
    content {
      arn = each.value.dlq_arn
    }
  }

  retry_policy {
    maximum_event_age_in_seconds = 86400
    maximum_retry_attempts       = 185
  }
}
```

```hcl
# modules/eventbridge-bus/alarms.tf

# Replication is broken. This is the single most important EventBridge alarm
# in the estate: it is the only thing that tells you your DR event stream has
# silently stopped.
resource "aws_cloudwatch_metric_alarm" "replication_dlq_not_empty" {
  count = var.replicate_to_bus_arn == null ? 0 : 1

  alarm_name  = "eventbridge-${var.name}-xregion-replication-failing"
  namespace   = "AWS/SQS"
  metric_name = "ApproximateNumberOfMessagesVisible"
  dimensions  = { QueueName = aws_sqs_queue.replication_dlq[0].name }

  statistic           = "Maximum"
  period              = 300
  evaluation_periods  = 1
  comparison_operator = "GreaterThanThreshold"
  threshold           = 0

  alarm_description  = "Cross-region event replication is failing. The standby archive is going stale. See 02-services/aws-eventbridge.md"
  treat_missing_data = "notBreaching"
}

# Standby is receiving events but matching none of them => rule drift.
# EventBridge discards unmatched events silently; this is the only way to see it.
resource "aws_cloudwatch_metric_alarm" "standby_matching_nothing" {
  count = var.is_standby ? 1 : 0

  alarm_name  = "eventbridge-${var.name}-standby-matched-zero-events"
  namespace   = "AWS/Events"
  metric_name = "MatchedEvents"
  dimensions  = { EventBusName = aws_cloudwatch_event_bus.this.name }

  statistic           = "Sum"
  period              = 3600
  evaluation_periods  = 2
  comparison_operator = "LessThanOrEqualToThreshold"
  threshold           = 0

  alarm_description  = "Standby bus matched zero events in an hour while replication should be flowing. Suspect rule drift or replication failure."
  treat_missing_data = "breaching"
}
```

### Calling it for a pair

```hcl
# environments/prod-eu/providers.tf
provider "aws" {
  alias  = "primary"
  region = "eu-west-1"
}

provider "aws" {
  alias  = "standby"
  region = "eu-west-2"
}
```

```hcl
# environments/prod-eu/eventbridge.tf

locals {
  # ONE definition. Both regions derive from it. Drift is structurally
  # impossible — which is the whole point in a cookiecutter monorepo.
  rules = {
    order-placed = {
      event_pattern = jsonencode({ source = ["com.acme.orders"], detail-type = ["OrderPlaced"] })
      targets       = {} # filled per region below in a real estate
    }
  }
}

# Standby FIRST — the primary's replication target needs its ARN.
module "bus_standby" {
  source    = "../../modules/eventbridge-bus"
  providers = { aws = aws.standby }

  name                   = "acme-${var.environment}"
  is_standby             = true
  create_archive         = true   # THE DR archive. Replay lands here.
  archive_retention_days = 30
  kms_key_arn            = aws_kms_key.events_standby.arn
  rules                  = local.rules
  tags                   = { Region = "standby" }
}

module "bus_primary" {
  source    = "../../modules/eventbridge-bus"
  providers = { aws = aws.primary }

  name                 = "acme-${var.environment}"
  is_standby           = false
  replicate_to_bus_arn = module.bus_standby.bus_arn
  create_archive       = true   # local debugging archive; NOT a DR asset
  kms_key_arn          = aws_kms_key.events_primary.arn
  rules                = local.rules
  tags                 = { Region = "primary" }
}
```

Note the **ordering dependency**: the standby bus must exist before the primary's
replication target can reference it. Terraform resolves this automatically
through the ARN reference, but it means the standby module cannot depend on the
primary module — the graph must flow standby → primary. If you ever want
symmetric replication (for failback, per [[aws-sqs]]'s symmetric-topics
recommendation), you have a dependency cycle and must break it by constructing
the ARN as a string rather than referencing the resource:

```hcl
replicate_to_bus_arn = "arn:aws:events:eu-west-1:${data.aws_caller_identity.current.account_id}:event-bus/acme-${var.environment}"
```

This is ugly and it bypasses Terraform's dependency tracking, which is exactly
the kind of thing [[provider-aliases-vs-separate-stacks]] exists to decide. If
the estate goes with separate stacks per region, the cycle disappears and you
pass the ARN through a remote state data source instead. **Flag this to whoever
owns that decision — cross-region EventBridge is an argument for separate
stacks.**

### Global endpoint, if you ever want one

```hcl
# Included for completeness. NOT recommended as the estate pattern — see above.
# Note: requires identically-named buses, same account, and ca-west-1 is
# unsupported.
resource "aws_cloudwatch_event_endpoint" "this" {
  provider = aws.primary

  name     = "acme-${var.environment}-global"
  role_arn = aws_iam_role.global_endpoint_replication.arn

  event_bus { event_bus_arn = module.bus_primary.bus_arn }
  event_bus { event_bus_arn = module.bus_standby.bus_arn }

  replication_config { state = "ENABLED" }

  routing_config {
    failover_config {
      primary   { health_check = aws_route53_health_check.primary.arn }
      secondary { route        = "eu-west-2" }
    }
  }
}
```

---

## Migration path from single-region

**Nothing here forces replacement of a live resource, and nothing interrupts the
primary.** This is a pure-additive migration, which makes it one of the safer
workstreams in the vault.

1. **Get the existing primary bus and rules under Terraform.** If the default
   bus is in use with click-created rules, `terraform import` them first. Note
   that the *default* event bus always exists and is not created by Terraform —
   `aws_cloudwatch_event_bus` with `name = "default"` is a no-op/import target,
   not a create.

2. **Refactor the primary into the module with `moved` blocks.**

   ```hcl
   moved {
     from = aws_cloudwatch_event_rule.order_placed
     to   = module.bus_primary.aws_cloudwatch_event_rule.this["order-placed"]
   }
   ```

   **`name` on `aws_cloudwatch_event_rule` is ForceNew.** If the module's computed
   rule name (`"${var.name}-${each.key}"`) differs from the live rule's name by a
   character, the plan destroys and recreates the rule. There is a window between
   destroy and create where events matching that rule are **silently discarded**
   — EventBridge does not queue unmatched events. **Read the plan. Zero
   destroys.**

3. **Create the standby bus and rules (disabled).** Pure additions. No effect on
   the primary.

4. **Create the standby archive.** Also pure addition. From this moment you are
   accumulating a cross-region event history, which is valuable even before any
   of the rest lands.

5. **Add the replication role, rule and target on the primary.** This is the only
   step with a cost impact: every replicated event is a new $1.00/M charge plus
   transfer. **Roll it out with a narrow `event_pattern` first** — one low-volume
   `source` — confirm events are arriving on the standby bus (`MatchedEvents` on
   the standby, or just look at the archive), then widen.

6. **Raise the standby region's quotas.** `PutEvents` TPS and invocations TPS,
   via Service Quotas, to match the primary's. **Do this early** — quota
   increases are not instant and you do not want the request in flight during an
   incident. Also raise `Number of rules` if you are near 300 per bus.

7. **Verify end to end.** Publish a synthetic event in the primary, confirm it
   lands in the standby archive. Then temporarily enable one harmless standby
   rule and confirm it fires. Then disable it again. **A replication path that
   has never delivered an event is not a replication path.**

8. **Add the alarms** (replication DLQ, standby `MatchedEvents`).

### What forces replacement

| Attribute | ForceNew? | Consequence |
|---|---|---|
| `aws_cloudwatch_event_bus.name` | Yes | Destroying a bus destroys its rules and archives. Never rename a bus. |
| `aws_cloudwatch_event_rule.name` | Yes | Brief window of silently-discarded events |
| `aws_cloudwatch_event_rule.event_bus_name` | Yes | Moving a rule between buses is a recreate |
| `aws_cloudwatch_event_archive.event_source_arn` | Yes | *"You cannot change the source event bus once an archive is created"* — and recreating the archive **loses the archived events**. Get this right first time. |
| `aws_cloudwatch_event_target.target_id` | Yes | Cheap; just a re-registration |

**The archive one is the dangerous one.** An archive is the only stateful
EventBridge resource, its state is your DR asset, and its source bus is
immutable. A Terraform refactor that recreates the archive silently deletes
your event history. Put a `lifecycle { prevent_destroy = true }` on the standby
archive and mean it.

---

## Failover procedure

1. **Decide.** Human. See [[failover-orchestration]].
2. **Enable the standby rules.** `aws events enable-rule --name X --event-bus-name Y`,
   or flip `is_standby` and apply — but **do not put a Terraform apply on the
   failover critical path.** Script the API calls. Allow a few seconds for
   control-plane propagation and then *verify* with `describe-rule`, do not
   assume.
3. **Enable the standby schedules** — but only the ones that should run. Review
   the list; a schedule enabled at the wrong point in its cycle is a duplicate
   job. Consider a per-schedule decision in the runbook rather than a blanket
   enable.
4. **Repoint producers** to the standby bus. Same bus name, different region, so
   this is a region configuration change, not a name change — provided you took
   the identical-naming branch, which you should have.
5. **Disable the primary's replication rule** *if the primary is reachable.*
   Otherwise you get a loop hazard the moment you add reverse replication for
   failback. If the primary is dark, note it and do it during recovery — and put
   it as a blocking pre-condition on any failback step.
6. **Verify.** `MatchedEvents` and `TriggeredRules` rising on the standby bus;
   `FailedInvocations` at zero; the replication DLQ not growing. Also watch
   `ThrottledRules` — this is where the quota asymmetry shows up, and it shows up
   as *degradation*, not as an error anyone will page on.
7. **Stop the RTO clock.** Service is restored.
8. **Only then**, and at least 10 minutes after the primary's last ingested
   event, start the replay to recover the gap. Bounded by time window.
   Overlapping is safe; gaps are not.

**What must be scripted, not manual:** step 2. Enabling forty rules by hand at
3am is how you enable thirty-nine.

## Failback

Harder than failover, for one specific reason: **the replication rule is
one-directional.**

After failing over, `eu-west-2` is live and nothing replicates back to
`eu-west-1`. Two branches, and this must be decided at build time:

- **Symmetric replication** — both buses have a replication rule pointing at the
  other. Failback is free. Costs: double the cross-region charges and transfer,
  and **you must be certain about loop prevention.** AWS's no-chaining rule
  (*"EventBridge can't route events received from a sender event bus to a third
  event bus"*) prevents A→B→C, but A→B→A is two buses, not three. **I could not
  find an AWS statement confirming that a bus will not echo a received event back
  to its sender.** *Do not assume it is safe.* Test it in a sandbox before
  enabling symmetric replication in production — an event loop between two buses
  at $1.00/M each is an expensive way to find out.
- **Asymmetric, reconfigured at failback** — add the reverse rule during
  failback. Cheaper and provably loop-free, but it puts a Terraform apply in the
  failback path.

**Recommendation: asymmetric, with the reverse rule pre-written and held in a
branch**, until someone has tested the loop behaviour. This differs from
[[aws-sqs]]'s recommendation of symmetric SNS topics, and deliberately: SNS
subscriptions cannot loop (a topic doesn't re-publish what it receives), but two
event buses replicating to each other plausibly can. **Prove it before you build
it.** If the sandbox test shows no echo, switch to symmetric and align with the
SQS note.

The other failback asymmetry: **the archive.** After failover, your DR archive is
attached to the bus that is now *primary*. You now want an archive on
`eu-west-1`, which is now the standby. If you followed the recommendation of
archiving on both buses, you already have one and there is nothing to do. **That
is the reason to archive on both, despite the duplicate cost** — it makes the
design symmetric and removes a step from the worst-remembered runbook in the
estate.

---

## Gotchas

1. **Only event buses can be cross-region targets.** Every other target type must
   be same-region. If someone's design diagram shows a rule in Ireland targeting
   a Lambda in London, it does not work.

2. **No chaining.** *"EventBridge can't route events received from a sender event
   bus to a third event bus."* Hub-and-spoke event architectures do not survive
   contact with this rule.

3. **An archive can only replay to its own source bus, and the source bus is
   immutable.** This inverts the naive design. **Archive on the standby.**

4. **Replay needs a 10-minute settling delay** and replays in one-minute buckets,
   unordered within the bucket. Two-thirds of your RTO, gone, before the first
   event moves.

5. **Replayed events carry a `replay-name` field the original did not.** If your
   idempotency key is derived from the whole event body, replayed events produce
   different keys and your idempotency silently stops working — during a
   recovery, which is the worst possible time.

6. **The standby region's quotas are a fraction of the primary's.**
   `eu-west-1` 10,000 TPS `PutEvents` vs `eu-west-2` 1,200. Invocations 18,750/s
   vs 2,250/s. Pipes concurrency 3,000 vs 1,000. **The US pair is the exception**
   — `us-west-2` matches `us-east-1` on all three. Raise EU and CA quotas
   explicitly, well in advance, and re-check after any account change.

7. **Unmatched events are discarded silently.** A standby with drifted or missing
   rules looks completely healthy: events arrive, nothing happens, no metric goes
   red. The `MatchedEvents`-is-zero alarm above is the only detection.

8. **EventBridge cannot target an SQS queue encrypted with an AWS *owned* key** —
   *"This includes targets, as well as Amazon SQS queues specified as dead-letter
   queues for targets."* This is in tension with [[aws-sqs]]'s advice to prefer
   `sqs_managed_sse_enabled` to avoid cross-region KMS grant problems. **For any
   queue that is an EventBridge target or an EventBridge target DLQ, you need a
   CMK**, and therefore you need a key in the standby region, and therefore you
   are back in [[aws-kms]] / [[kms-when-to-use-multi-region-keys]] territory.
   Reconcile these two notes before writing the queue module's default.

9. **The KMS key policy must allow `events.amazonaws.com`** `kms:Decrypt` and
   `kms:GenerateDataKey` for encrypted targets, and **that grant must exist on
   the key in the standby region**, not just the primary's. A standby target
   whose key policy was never mirrored fails at first use, i.e. during the
   failover.

10. **Terraform provider issue
    [hashicorp/terraform-provider-aws#31444](https://github.com/hashicorp/terraform-provider-aws/issues/31444)**
    — users report `ValidationException: Cross-region api call is not allowed`
    when creating a cross-region event bus target via Terraform, while the same
    configuration works from the console. **Verify against your pinned provider
    version before committing to this design.** If you hit it, the failure is at
    apply time, not plan time, which is the worst place to discover it. (The
    reported region in that issue is `af-south-1`; whether the problem is
    provider-side or a genuine destination-region restriction is not resolved in
    the thread — which is itself a reason to run the five-minute CLI test for
    `ca-west-1` rather than trusting either.)

11. **`aws_cloudwatch_event_rule.name` and `aws_cloudwatch_event_bus.name` are
    ForceNew.** A rename is a destroy-then-create, and during that window
    matching events are dropped with no error and no metric.

12. **Global endpoints need SigV4A, and the global STS endpoint doesn't vend
    SigV4A-capable credentials by default.** *"If you request temporary
    credentials from the global AWS STS endpoint (sts.amazonaws.com), AWS STS
    vends credentials which, by default, do not support SigV4A."* A latent,
    environment-specific failure in anything using IRSA or assumed roles.

13. **`ca-west-1` has no archive, no replay, no global endpoints, no Pipes and no
    API destinations.** It has event buses, Scheduler and the schema registry.
    **Every EventBridge DR mechanism in this note except "an empty mirror bus" is
    unavailable for the CA pair.** This is a significant, concrete parity failure
    and it belongs in [[region-pair-selection]] as evidence.

14. **`ca-west-1` is opt-in.** It must be enabled on the account before any
    provider block can target it.

15. **A missed scheduled job is completely silent.** A schedule disabled in the
    standby and dead in the primary simply does not fire, and nothing reports it.
    Enumerate schedules and their fire times in the runbook.

16. **`DescribeArchive`'s event count lags by up to 24 hours.** Do not plan a
    replay by count. Plan by time window.

17. **Ten concurrent replays per account per region, total.** A fan-out recovery
    plan across many buses serialises whether you intended it to or not.

---

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Cross-region mechanism | Cross-region event bus target | Global endpoint | **A.** Global endpoints fail over ingestion independently of the rest of the estate, need SDK changes, and don't exist in `ca-west-1`. |
| Do we replicate events at all? | Empty mirror bus, accept loss | Replicate continuously to the standby bus | **B for anything with a real event stream.** It's $1.00/M plus transfer, it's continuously self-testing, and it enables the archive design. A for low-value or purely-signal buses. |
| Standby rules | Enabled (both regions process) | Disabled (fenced) | **B.** Rule state is the cleanest fence in the estate: one API call, verifiable, persists across a region recovery. |
| Where does the DR archive live? | Primary bus | **Standby bus** | **B — this is not really a choice.** Replay is to the source bus only. An archive on the primary is a debugging tool. Build both; only the standby one is a DR asset. |
| Archive retention | AWS default (indefinite) | A fixed window | **30–90 days.** Indefinite is a slowly-growing bill nobody owns. Long enough to cover "we need to replay last month" and short enough to bound cost. |
| Is replay in the RTO? | Yes — fail over then replay | No — replay is post-RTO recovery | **B, explicitly and in writing.** The 10-minute settling delay alone rules out A, and the standby quota makes it worse. |
| Failback replication | Symmetric (both directions always) | Asymmetric, added at failback | **B until the A→B→A loop behaviour is tested.** Then reconsider. Differs deliberately from [[aws-sqs]]'s SNS recommendation. |
| Standby schedules | Enabled | Disabled | **Disabled, no exceptions.** At-least-once + two enabled copies = duplicate jobs. |
| `ca-west-1` | Same pattern as EU/US | Different pattern | **Different, and this is forced.** Empty mirror bus only. Feed into [[region-pair-selection]]. |

---

## Cost

From the EventBridge pricing page:

| Item | Price |
|---|---|
| Custom events ingested | **$1.00 per million** |
| Delivery to a target in the same account | **$0.00** |
| Delivery to a different account | **$1.00 per million** |
| Archive processing | **$0.10 per GB** |
| Archive storage | **$0.023 per GB-month** |
| Replay | Billed as custom events — **$1.00 per million** |
| Scheduler | 14,000,000 invocations/month free, then **$1.00 per million** |
| Pipes | **$0.40 per million requests** (64 KB chunks = one request) |
| Cross-region data transfer | *"You may incur additional data transfer charges between regions billed at standard AWS Data Transfer Charges."* |

**Worked example.** 500 events/second sustained, 2 KB average event.

- 500 eps ≈ **1.3 billion events/month**.
- Primary ingestion: already being paid today. ~$1,300/month.
- **Cross-region replication adds ingestion on the standby bus: ~$1,300/month.**
  (The pricing page prices cross-*account* delivery at $1.00/M and is not
  explicit about cross-*region* same-account delivery; at minimum the receiving
  bus ingests the event and is billed for it. **Verify this on a real bill before
  committing a budget number** — this is the single biggest uncertainty in this
  section.)
- Inter-region transfer: 1.3B × 2 KB ≈ **2.6 TB/month**. At roughly $0.02/GB
  EU↔EU that is ~$52/month. Small. (Verify the rate against AWS's data transfer
  pricing; [[aws-sqs]] flags the same number as third-party-sourced.)
- Archive processing: 2.6 TB × $0.10/GB ≈ **$260/month**, one-time per GB
  processed.
- Archive storage at 30 days retention: ~2.6 TB stored × $0.023 ≈ **$60/month**.
- **Total incremental: roughly $1,670/month at 500 eps**, dominated by the
  per-event replication charge.

**The levers, largest first:**

1. **Narrow the replication rule's event pattern.** You almost certainly do not
   need every event in the standby. Replicate the ones that are *records*, per
   [[messaging-in-flight-data-loss]]'s classification, and drop the ones that are
   *signals*. This is a 10× lever, not a 10% one — and it is the only lever that
   matters at high volume.
2. **Shorten archive retention.** Linear in storage cost, and 90 days vs
   indefinite is the difference between a bounded and an unbounded bill.
3. **Filter the archive's event pattern separately from the replication rule's.**
   You can replicate broadly for processing and archive narrowly for recovery.
4. **Batch at the producer.** `PutEvents` accepts up to 10 entries, but billing is
   per event, so batching saves TPS quota, not money. Worth doing for the quota
   headroom in the standby.

At low volume (< 10M events/month) the whole thing is under $20/month and none of
this matters. **Compute the estate's actual monthly event count before designing
for cost** — it is a one-line CloudWatch query against the bus's `MatchedEvents`
and it will probably show this is a rounding error, exactly as [[aws-sqs]] found.

---

## Open questions

1. **Is `ca-west-1` a supported cross-region event bus *destination*?** Not
   answerable from the docs. Five-minute CLI test. Gates whether the CA pair gets
   anything at all beyond an empty mirror bus. → [[region-pair-selection]]
2. **Does bus A → bus B → bus A loop?** The no-chaining rule covers three buses,
   not two. Must be tested before symmetric replication is built. Gates the
   failback design.
3. **What is the estate's actual monthly event volume per bus?** Decides whether
   the cost section is a rounding error or a budget line.
4. **Which events are records and which are signals?** Same classification
   [[messaging-in-flight-data-loss]] demands for queues, applied to event
   patterns. It is the replication-rule event pattern, and therefore the cost
   lever.
5. **Are any EventBridge target queues currently using SSE-SQS / an AWS-owned
   key?** If so they are already broken as EventBridge targets, or they are
   about to be when someone "simplifies" the KMS config. Audit against gotcha 8.
6. **Is `terraform-provider-aws` #31444 reproducible on our pinned version?**
   Gates whether this design is buildable in Terraform at all, or needs a
   console/CLI escape hatch.
7. **Does anything derive an idempotency key from the whole event body?** If yes,
   replay is unsafe until that changes (gotcha 5).
8. **What are the standby regions' current quota values, as opposed to the
   documented defaults?** Someone may have raised them already. Check Service
   Quotas per region and per account before raising a ticket.
9. **What scheduled jobs exist, and what is the blast radius of one silently not
   running during a failover window?** Nobody has this list.

---

## Sources

- [Sending and receiving events between AWS Regions in Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-cross-region.html)
  — the cross-region feature's home page. Thin; mostly links to the blog.
- [Sending and receiving events between AWS accounts in Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-cross-account.html)
  — the "destination Region must be a supported cross-Region destination Region"
  statement, the no-forwarding-to-a-third-account rule, and the March 2 2023
  IAM-role requirement for new cross-account bus targets.
- [Sending events between event buses in the same account and Region](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-bus-to-bus.html)
  — *"EventBridge can't route events received from a sender event bus to a third
  event bus."* The no-chaining rule.
- [Introducing cross-Region event routing with Amazon EventBridge — AWS Compute Blog](https://aws.amazon.com/blogs/compute/introducing-cross-region-event-routing-with-amazon-eventbridge/)
  — the IAM role shape, and *"The delivered event is identical to the original
  event, and does not contain any additional metadata or attributes."*
- [Amazon EventBridge introduces support for cross-Region event bus targets (Apr 2021)](https://aws.amazon.com/about-aws/whats-new/2021/04/amazon-eventbridge-introduces-support-cross-region-event-bus-targets)
  — the original three destination regions.
- [Amazon EventBridge cross-Region support now expands to more Regions (Nov 2021)](https://aws.amazon.com/about-aws/whats-new/2021/11/amazon-eventbridge-cross-region-expands)
  — the enumerated 21-region destination list quoted above. **The most recent
  authoritative list I could find**, and it predates `ca-west-1`.
- [Archiving and replaying events in Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-archive.html)
  — *"Archive events can only be replayed to the source event bus"*, the
  `replay-name` field, the 10-minute settling recommendation, minute-bucket
  ordering, ten concurrent replays, 90-day replay deletion, the 24-hour
  `DescribeArchive` reconciliation period, and the PutEvents-bound replay rate.
  **The single most important source in this note.**
- [Making applications Regional-fault tolerant with global endpoints in EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-global-endpoints.html)
  — the 360s/420s RTO/RPO figure, the CRT/SigV4A/`endpointId` requirements, the
  STS SigV4A caveat, the identical-bus-name requirement, the replication-required
  -for-auto-recovery behaviour, and the available-regions list (no `ca-west-1`).
- [Amazon EventBridge feature availability by AWS Region](https://docs.aws.amazon.com/eventbridge/latest/userguide/feature-availability.html)
  — the definitive parity table. Canada West (Calgary) has event buses, Scheduler
  and schema registry only: **no Pipes, no archive and replay, no schema
  discovery, no global endpoints, no API destinations.**
- [Amazon EventBridge quotas](https://docs.aws.amazon.com/eventbridge/latest/userguide/cloudwatch-limits-eventbridge.html)
  — per-region `PutEvents` and invocation TPS (the 10,000 vs 1,200 EU asymmetry),
  300 rules per bus, 5 targets per rule, and Pipes concurrency by region.
- [Event bus targets in Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-targets.html)
  — the full target list, the target IAM/trust policy shape, the KMS key policy
  statement for encrypted targets, and *"EventBridge does not support using
  Amazon SQS queues that are encrypted with an AWS owned key. This includes
  targets, as well as Amazon SQS queues specified as dead-letter queues for
  targets."*
- [RetryPolicy — EventBridge API Reference](https://docs.aws.amazon.com/eventbridge/latest/APIReference/API_RetryPolicy.html)
  — `MaximumEventAgeInSeconds` 60–86,400 and `MaximumRetryAttempts` 0–185.
- [Amazon EventBridge pricing](https://aws.amazon.com/eventbridge/pricing/)
  — $1.00/M custom events, $0.10/GB archive processing, $0.023/GB-month archive
  storage, replay billed as custom events, Scheduler free tier, Pipes $0.40/M.
- [Amazon EventBridge FAQs](https://aws.amazon.com/eventbridge/faqs/)
  — Pipes ordering guarantee and Scheduler's at-least-once statement. Note what
  it does *not* say: there is no ordering or exactly-once claim for event buses.
- [aws_cloudwatch_event_endpoint — Terraform provider docs](https://github.com/hashicorp/terraform-provider-aws/blob/main/website/docs/r/cloudwatch_event_endpoint.html.markdown)
  — the global endpoint resource schema used above.
- [aws_cloudwatch_event_archive — Terraform provider docs](https://github.com/hashicorp/terraform-provider-aws/blob/main/website/docs/r/cloudwatch_event_archive.html.markdown)
  — `retention_days`, `event_source_arn`, `kms_key_identifier`.
- [hashicorp/terraform-provider-aws#31444 — aws_cloudwatch_event_target cross-region](https://github.com/hashicorp/terraform-provider-aws/issues/31444)
  — the `ValidationException: Cross-region api call is not allowed` report.
- [Multi-Region event-driven failover architecture with Amazon EventBridge and Route 53 — AWS Compute Blog](https://aws.amazon.com/blogs/compute/multi-region-event-driven-failover-architecture-with-amazon-eventbridge-and-route-53/)
  — an AWS reference architecture for active/passive event-driven failover:
  independent regional buses, Route 53 health checks (30s interval, 3-failure
  threshold ≈ 90s detection), DynamoDB global tables for state. **Note it does
  not use archive/replay or cross-region bus targets** — worth knowing that
  AWS's own published pattern here is "two independent stacks plus DNS", which
  is closer to this note's option (a) than to its recommendation.

### Searched for and did not find

- **A current, authoritative, enumerated list of supported cross-Region
  *destination* regions.** The Nov 2021 announcement is the newest enumeration I
  found. Whether `ca-west-1` (launched Dec 2023) is a valid destination is
  **unresolved** and must be tested.
- **Any published throughput figure or benchmark for EventBridge replay.** No
  public data found, from AWS or third parties. The only quantitative anchor is
  the docs' statement that replay is bounded by the account's `PutEvents` TPS
  limit; the estimates in this note are derived from that, and are ceilings
  rather than measurements.
- **Any statement about whether two event buses replicating to each other will
  loop.** The no-chaining rule is stated for three buses only. Untested.
- **Any deprecation notice for global endpoints.** None found — the feature
  appears current. Equally, no announcements of any kind for it since GA in
  April 2022.
- **A published case study of an organisation using EventBridge archive/replay
  as the recovery mechanism in a real regional failover.** No public example
  found. The idea is sound and the primitives are documented, but nobody
  appears to have written up doing it in anger.
- **An AWS statement on event ordering for event buses.** None exists, in either
  direction. Treat buses as unordered.
