---
title: Amazon SNS — Multi-Region
service: sns
tags: [service, multi-region, sns, messaging, eventing, fanout]
status: researched
replication: none
rpo_achievable: "0 for the topic *resource*. For messages: seconds, and better than SQS because SNS retries AWS-managed endpoints for 23 days. FIFO topics with an archive policy can do RPO 0 via replay, up to 365 days back."
rto_achievable: "< 1 min IF every standby subscription is pre-created AND pre-confirmed. Minutes-to-hours if any HTTP/S or email subscription is left in PendingConfirmation — that is the failure mode this note exists to prevent."
meets_targets: conditional
updated: 2026-09-21
---

# Amazon SNS — Multi-Region

> **The fact everything else follows from.**
>
> **An SNS topic is a regional resource with no native cross-region
> replication.** There is no replica topic, no `replica_regions` argument, no
> AWS-managed mirroring. The standby topic is a *second, separate resource with
> a different ARN*:
>
> ```
> arn:aws:sns:eu-west-1:111122223333:orders-prod
> arn:aws:sns:eu-west-2:111122223333:orders-prod
> ```
>
> Same name. Different resource. Different identity. Every publisher, every
> subscription, every IAM policy, every SQS queue policy, every Lambda
> permission and every alarm action that names one of those strings must be
> parameterised, because at failover the other string is the one that works.
>
> This is not a hardship — it is the same situation as [[aws-sqs]], and the
> answer is the same shape. But SNS has three properties SQS does not, and they
> change the design materially:
>
> 1. **SNS can deliver across regions** to SQS and Lambda, officially and
>    natively. SQS cannot pull across regions; SNS can push.
> 2. **SNS subscriptions have a confirmation handshake** for some protocols,
>    and an unconfirmed subscription is silently inert. This is the RTO trap.
> 3. **SNS FIFO topics have a built-in message archive with replay**, up to 365
>    days. That is a genuine recovery-point mechanism — the only one in the
>    messaging family.

Read [[aws-sqs]] first if you have not. This note deliberately does not restate
the queue-naming argument, the DLQ-retention argument, or the
"is a pending message even data" argument — those live in [[aws-sqs]] and
[[messaging-in-flight-data-loss]] and they apply unchanged here.

## TL;DR

- **SNS topics are regional and do not replicate.** The standby is a second
  topic with a different ARN. Pre-create it, pre-subscribe it, and — the part
  everyone forgets — **pre-confirm it**.
- **Cross-region subscriptions are real but narrow.** AWS supports cross-region
  delivery **to SQS queues and to Lambda functions only**. Those are the only
  two destination types the cross-region delivery page names. HTTP/S and email
  are not "cross-region" concepts at all — they are internet endpoints and work
  from any region — but that is a different claim and the note draws the
  distinction carefully below. **Firehose is not listed and should be assumed
  same-region until tested.**
- **Subscription confirmation is the 15-minute killer.** HTTP/S and email
  subscriptions sit in `PendingConfirmation` until the endpoint owner calls
  `ConfirmSubscription`. Confirmation tokens are valid for **two days**
  ([Subscribe API](https://docs.aws.amazon.com/sns/latest/api/API_Subscribe.html)),
  so you cannot create them once and forget them — an unconfirmed subscription
  ages out. A standby topic whose HTTPS subscriptions are all pending is a
  topic that publishes into a void, with **no error, no metric, no alarm**.
  Readiness-check `SubscriptionArn != "PendingConfirmation"` on every standby
  subscription, continuously.
- **The standby region's publish quota is up to 30× lower than the primary's,
  by default.** `eu-west-1` allows 9,000 messages/second; `eu-west-2` is in the
  "all other supported Regions" bucket at **300 messages/second**. Failing over
  from Ireland to London without raising that quota first means throttled
  publishes at exactly the wrong moment. This is the single most
  under-appreciated finding in this note and it applies to
  [[region-pair-selection]] generally.
- **`ca-west-1` passes the SNS parity check** — SNS is present, supports
  standard and FIFO topics, and FIFO archive/replay is available in "All
  Commercial Regions". **But Calgary is an opt-in region, which changes the
  service principal** you must put in cross-region SQS queue policies:
  `sns.ca-west-1.amazonaws.com`, not `sns.amazonaws.com`. Get that wrong and
  cross-region delivery fails silently. Feed this into
  [[region-pair-selection]] as a *pass* — a rare one for Calgary.

---

## Does this service cross regions at all?

Two different questions, and conflating them is how people get this wrong.

**Q1: Does the topic resource replicate?** No. Nothing about a topic —
attributes, subscriptions, filter policies, archive contents, access policy —
is mirrored anywhere by AWS.

**Q2: Can a topic in one region deliver a message to a resource in another?**
Yes, for two destination types, officially.

| Thing | Regional or global? | Consequence |
|---|---|---|
| Topic ARN | Regional | The ARN *is* the identity. Standby topic ≠ primary topic. |
| Topic name | Unique per account **per region** | You may reuse the name. Recommended — see [[aws-sqs]]'s naming argument, which applies verbatim. |
| Subscription ARN | Regional, derived from topic ARN | `arn:aws:sns:<region>:<acct>:<topic>:<uuid>`. Standby subscription UUIDs are different and unpredictable. |
| Subscription *endpoint* | **May be in another region** — SQS and Lambda only | This is the one genuinely cross-region capability. |
| Topic access policy | Regional | Contains the topic ARN. Must be templated per region. |
| Filter policy | Per-subscription, regional | **Drifts silently.** See below. |
| Subscription DLQ (`RedrivePolicy`) | **Same account and region as the subscription** | Explicitly stated by AWS. No cross-region SNS DLQ. |
| FIFO archive | Per-topic, regional | The archive in `eu-west-1` is unreachable when `eu-west-1` is. |
| KMS key for SSE | Regional | See [[aws-kms]]. |
| Platform applications (mobile push) | Regional | Device tokens registered in one region are not registered in the other. Out of scope here but worth a line in the runbook. |

### The API endpoint

`sns.<region>.amazonaws.com`. There is no global SNS endpoint. There is no
"multi-region topic". A client configured for `eu-west-1` publishing to an
`eu-west-2` topic ARN will fail — the SDK routes by client region, not by ARN
region, and the topic does not exist in `eu-west-1`. (This trips people up
because `Publish` takes an ARN and *looks* region-agnostic. It is not.)

---

## Cross-region subscriptions — the precise answer

This is the question the architecture turns on, so here is exactly what AWS
says, with nothing added.

From
[Sending Amazon SNS messages to an Amazon SQS queue or AWS Lambda function in a different Region](https://docs.aws.amazon.com/sns/latest/dg/sns-cross-region-delivery.html):

> *"Amazon SNS supports cross-Region deliveries, both for Regions that are
> enabled by default and for opt-in Regions."*
>
> *"Amazon SNS supports the cross-Region delivery of notifications to Amazon SQS
> queues and to AWS Lambda functions. When one of the Regions is an opt-in
> Region, you must specify a different Amazon SNS service principal in the
> subscribed resource's policy."*
>
> *"The Amazon SNS subscription command must be executed in the Region where
> Amazon SNS is hosted."*

### Per destination type

| Destination | Cross-region? | Basis |
|---|---|---|
| **SQS queue** | **Yes — officially supported.** | Named explicitly, with a support matrix for every opt-in/default combination. |
| **Lambda function** | **Yes — officially supported.** | Named explicitly, with its own support matrix. |
| **HTTP / HTTPS** | **Not a cross-region question.** | The endpoint is a URL on the public internet. SNS does not know or care which region (or cloud, or datacentre) serves it. There is no service principal to regionalise because there is no AWS resource policy involved. It "works cross-region" in the only sense the phrase can mean. |
| **Email / Email-JSON** | **Not a cross-region question.** | SMTP to an address. Same reasoning. |
| **SMS** | **Not a cross-region question**, but see the SMS section — the *origination* configuration and spend limits are per-region and do not replicate. |
| **Firehose delivery stream** | **Not stated.** AWS's cross-region page does not mention Firehose. Firehose subscriptions additionally require a `SubscriptionRoleArn`. **Treat as same-region-only until you have tested it.** No public source found either confirming or denying. |
| **Platform application endpoint (mobile push)** | **Not stated.** Platform applications are regional resources; assume same-region. |

**So: yes, an SNS topic in `eu-west-1` can deliver to an SQS queue in
`eu-west-2`.** This is the fact [[aws-sqs]] option (b) is built on, and it is
the only zero-code cross-region message-movement primitive AWS gives you.

### What you must actually configure

Three things, and two of them are easy to get wrong.

**1. The subscription must be created from the topic's region.**

AWS: *"The Amazon SNS subscription command must be executed in the Region where
Amazon SNS is hosted."* The Terraform provider documents the same constraint in
its own words:

> *"If the SNS topic and SQS queue are in different AWS regions, the
> `aws_sns_topic_subscription` must use an AWS provider that is in the same
> region as the SNS topic."*
> — [aws_sns_topic_subscription docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/sns_topic_subscription)

In a provider-alias monorepo this means: **the subscription resource takes the
provider of the topic, not of the queue.** That is counter-intuitive when the
queue is the "interesting" resource, and it is the single most common Terraform
error in this pattern. See [[provider-aliases-vs-separate-stacks]].

**2. The destination's resource policy must allow the SNS service principal,
with `aws:SourceArn` scoped to the topic.**

For the EU and US pairs (all four regions are enabled-by-default), the ordinary
principal works:

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "sns.amazonaws.com" },
  "Action": "sqs:SendMessage",
  "Resource": "arn:aws:sqs:eu-west-2:111122223333:orders-prod",
  "Condition": {
    "ArnLike": { "aws:SourceArn": "arn:aws:sns:eu-west-1:111122223333:orders-prod" }
  }
}
```

**3. If either region is an opt-in region, the principal must be regionalised.**
This is the CA-pair landmine and it gets its own section.

### The `ca-west-1` finding

`ca-west-1` (Calgary) **is an opt-in region.** Confirmed directly against
[Enable or disable AWS Regions](https://docs.aws.amazon.com/general/latest/gr/rande-manage.html),
whose opt-in table lists `Canada West (Calgary) | ca-west-1 | GA`. `ca-central-1`
is in the default-enabled table.

**SNS parity in `ca-west-1`: PASS.** From
[Amazon SNS endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/sns.html):

- `sns.ca-west-1.amazonaws.com` exists, plus a FIPS endpoint
  (`sns-fips.ca-west-1.api.aws`) — which not every region has.
- The endpoints table is prefaced *"The following endpoints support both
  standard and FIFO topics"*, so FIFO topics are available in Calgary.
- FIFO **message archiving and replay** is listed as available in *"All
  Commercial Regions"*. `ca-west-1` is a commercial region, so archive/replay is
  available there.

**This is a rare Calgary pass** and it matters, because Calgary has already
failed the parity check for Cognito multi-region replication, OpenSearch
cross-cluster replication and Amazon Managed Grafana. SNS is not a reason to
abandon the CA pair. Record that in [[region-pair-selection]].

**But the opt-in status has a concrete, breaking consequence.** From the
cross-region delivery page's support matrix:

| Direction | Required service principal in the queue policy |
|---|---|
| Default-enabled → opt-in | `sns.<queue-region>.amazonaws.com` |
| Opt-in → default-enabled | `sns.<topic-region>.amazonaws.com` |
| Opt-in → opt-in | `sns.<queue-region>.amazonaws.com` |

Apply that to the CA pair:

| Scenario | Topic | Queue | Principal |
|---|---|---|---|
| Steady state (primary fans out to standby) | `ca-central-1` (default) | `ca-west-1` (opt-in) | **`sns.ca-west-1.amazonaws.com`** |
| After failover (standby fans back) | `ca-west-1` (opt-in) | `ca-central-1` (default) | **`sns.ca-west-1.amazonaws.com`** |

Pleasingly, **both directions in the CA pair use `sns.ca-west-1.amazonaws.com`**
— the opt-in region's regionalised principal, either way. Write it once in the
module as a computed local and stop thinking about it.

The EU pair (`eu-west-1` ↔ `eu-west-2`) and US pair (`us-east-1` ↔ `us-west-2`)
are all default-enabled in both directions, so both use plain
`sns.amazonaws.com`. **Three pairs, two different principals — this is exactly
the kind of per-pair divergence that a cookiecutter template must compute
rather than hardcode.**

> **A documentation trap, stated plainly.** The SNS cross-region delivery page
> carries its own list of opt-in regions, and that list is **stale**. It names
> eleven regions and **does not include `ca-west-1`** (nor `ap-east-2`,
> `ap-southeast-5/6/7`, `mx-central-1`). The page also states the actual rule —
> *"Opt-in regions include any regions launched after March 20, 2019"* — which
> unambiguously covers Calgary (launched December 2023). **Trust the rule and
> the General Reference table, not the SNS page's enumeration.** An engineer who
> reads only the SNS page will conclude Calgary is default-enabled, write
> `sns.amazonaws.com` into the queue policy, and ship a cross-region
> subscription that never delivers. The subscription will show as `Confirmed`.
> Nothing will alarm.

### How the failure presents

This is worth knowing because it is genuinely nasty. If the queue policy is
wrong, the subscription still confirms (SQS subscriptions in the same account
auto-confirm regardless of whether delivery will work). Publishes succeed. SNS
records a delivery *failure* and retries — for 23 days. The queue stays empty.
`NumberOfNotificationsFailed` on the topic is the only signal, and nobody alarms
on it by default.

**Alarm on `NumberOfNotificationsFailed` for every topic in both regions.** It
is the SNS equivalent of [[aws-sqs]]'s DLQ-depth canary and it is the only thing
that catches a misconfigured cross-region subscription before an incident does.

---

## Subscription confirmation — the RTO trap

This section is why the note's `rto_achievable` says "conditional".

### The mechanic

From the [Subscribe API reference](https://docs.aws.amazon.com/sns/latest/api/API_Subscribe.html):

> *"If the endpoint type is HTTP/S or email, or if the endpoint and the topic
> are not in the same AWS account, the endpoint owner must run the
> `ConfirmSubscription` action to confirm the subscription. You call the
> `ConfirmSubscription` action with the token from the subscription response.
> **Confirmation tokens are valid for two days.**"*

And on the response:

> *"The ARN of the subscription if it is confirmed, or the string **"pending
> confirmation"** if the subscription requires confirmation."*

So confirmation is required when **either**:

- the protocol is `http`, `https`, `email` or `email-json`; **or**
- the endpoint and topic are in **different AWS accounts** (any protocol).

Note carefully what is *not* on that list: **different *regions* is not a
trigger.** A cross-region SQS or Lambda subscription within one account
auto-confirms. That is a genuine relief for the fan-out design and it should be
stated explicitly in the design doc, because people assume cross-region implies
cross-account implies confirmation, and it does not.

### Why it breaks a 15-minute RTO

Picture the standby topic, built by Terraform six months ago:

| Subscription | Protocol | State | Behaviour at failover |
|---|---|---|---|
| `orders-standby` queue | `sqs`, same account | `Confirmed` | Works. |
| `order-processor` fn | `lambda`, same account | `Confirmed` | Works. |
| Partner webhook | `https` | **`PendingConfirmation`** | **Silently drops every message.** |
| `ops-alerts@…` | `email` | **`PendingConfirmation`** | **Nobody gets paged.** |
| Vendor SIEM | `https`, cross-account | **`PendingConfirmation`** | **Silent.** |

At 03:00 you fail over, publishes succeed, `NumberOfMessagesPublished` looks
healthy — and three subscribers receive nothing. There is no error. A pending
subscription is not a failed delivery; it is simply *not a subscriber*. It does
not appear in `NumberOfNotificationsFailed`. It does not go to a DLQ. It
produces no metric at all.

Fixing it live means: get the endpoint owner (possibly a third party, possibly
asleep — see [[third-party-saas-dependencies]]) to re-trigger and accept a
confirmation POST, per subscription. That is not a 15-minute operation. **For
an HTTPS endpoint owned by another company, it is not even a same-day
operation.**

### The two-day token expiry makes "pre-confirm once" insufficient

This is the subtle part. You cannot simply confirm the standby subscriptions at
build time and consider the problem solved *if the subscription ever reverts to
pending*. A confirmed subscription stays confirmed indefinitely — that part is
fine. The two-day window applies to the **token**, i.e. to the gap between
`Subscribe` and `ConfirmSubscription`.

Where that bites:

- **Terraform creates the subscription; nobody confirms within two days; the
  token expires.** The subscription remains in `PendingConfirmation` forever
  and can never be confirmed — you must `Unsubscribe` and re-`Subscribe` to get
  a fresh token. Terraform will not do this for you: it sees the resource as
  existing and reports no drift.
- **Unconfirmed subscriptions cannot be deleted through the console** (the
  Delete button is disabled) and cannot be deleted by ARN through the API,
  because they have no ARN — they have the literal string
  `PendingConfirmation`. The provider docs confirm Terraform inherits this
  limitation: *"Unconfirmed subscriptions (via `email`, `email-json`, `http`,
  `https`) cannot be deleted by Terraform."* So a stuck pending subscription is
  **garbage you cannot collect**, occupying the 5,000-per-account
  pending-subscription quota, until AWS ages it out.
- **They count against a quota.** "Pending subscriptions: 5,000 per account"
  ([SNS quotas](https://docs.aws.amazon.com/general/latest/gr/sns.html)). An
  estate that repeatedly re-creates unconfirmable HTTPS subscriptions in a CI
  loop can exhaust this. Unlikely, but it is a real ceiling and it is not
  obvious.

> **Conflicting public sources on expiry, stated honestly.** The authoritative
> `Subscribe` API page says tokens are valid for **two days**. Several
> third-party posts and some AWS re:Post answers say three days, and describe
> SNS auto-removing pending subscriptions after two or three days. **I could not
> find an AWS documentation page that states the auto-removal period for pending
> subscriptions.** Use two days as the planning number — it is the one AWS's own
> API reference gives — and do not build anything that depends on the exact
> auto-removal behaviour. Verify in a sandbox if it matters.

### The readiness check — build this

The mitigation is not clever, it is just *present*. A continuous check that
every subscription on every standby topic is `Confirmed`.

```bash
# Every standby topic, every subscription, anything not confirmed.
aws sns list-topics --region eu-west-2 --query 'Topics[].TopicArn' --output text \
| tr '\t' '\n' \
| while read -r topic; do
    aws sns list-subscriptions-by-topic --region eu-west-2 --topic-arn "$topic" \
      --query 'Subscriptions[?SubscriptionArn==`PendingConfirmation`].[TopicArn,Protocol,Endpoint]' \
      --output text
  done
```

Any output at all is a failed readiness check. Run it on a schedule — a
five-minute Lambda in the standby region publishing a custom CloudWatch metric
`StandbyPendingSubscriptions`, alarmed at `> 0`. See
[[observability-multi-region]].

Three things to know when you build it:

1. **`ListSubscriptionsByTopic` is hard-throttled at 30 TPS** and cannot be
   increased. With a few hundred topics this is fine; with tens of thousands it
   needs pagination-aware backoff.
2. **Prefer `GetSubscriptionAttributes`** if you already know the ARNs — but
   note that in `eu-west-2`, `ca-central-1` and most non-tier-1 regions,
   `GetSubscriptionAttributes` sits in the **30 TPS** soft-quota bucket. In
   `eu-west-1` it is 900 and in `us-east-1` it is 3,000. Your readiness checker
   will be 30× slower in the standby than the primary. Build it to tolerate
   that.
3. **`aws_sns_topic_subscription` exposes `pending_confirmation` as an
   attribute.** You can surface it as a Terraform output and assert on it in CI
   — a cheap second line of defence that catches drift at plan time rather than
   at 3am.

### Making HTTPS subscriptions confirm themselves

For endpoints you control, the clean answer is to make the endpoint
auto-confirm: on receiving a `SubscriptionConfirmation` message type, validate
the signature and `GET` the `SubscribeURL`. **Validate the message signature
before following the URL** — an unauthenticated auto-confirm handler will
confirm subscriptions to topics you do not own, which is a genuine (if low
-impact) security bug.

Terraform has direct support:

```hcl
resource "aws_sns_topic_subscription" "webhook_standby" {
  provider  = aws.standby
  topic_arn = aws_sns_topic.standby.arn
  protocol  = "https"
  endpoint  = var.webhook_url

  # Terraform will poll for the subscription to leave PendingConfirmation.
  endpoint_auto_confirms         = true
  confirmation_timeout_in_minutes = 5   # default is 1 — too tight for a real endpoint
}
```

`endpoint_auto_confirms = true` tells the provider the endpoint will confirm
itself, and `confirmation_timeout_in_minutes` is how long Terraform waits before
giving up. **The default is 1 minute**, which is frequently too short for an
endpoint that has to be woken, so raise it. If the apply fails on timeout, the
subscription still exists in AWS in pending state — and per the deletion
limitation above, you cannot easily clean it up.

**For endpoints you do not control** (partner webhooks, vendor SIEMs, a
customer's callback URL), there is no technical fix. The only answers are
organisational:

- Get the standby subscription confirmed **at build time**, as part of
  onboarding that partner, and then verify it weekly with the readiness check
  above. Treat "confirmed in both regions" as an onboarding acceptance
  criterion.
- Or **remove the third party from the failover critical path entirely** by
  subscribing an SQS queue you own and having your own relay push to the
  partner. This converts a confirmation problem into a queue problem, and
  [[aws-sqs]] has already solved the queue problem. **This is the
  recommendation** for any partner endpoint whose availability you are held to.
  See [[third-party-saas-dependencies]].
- Or accept, in writing, that this subscriber is lost for the duration of a
  failover, and record it in the runbook.

### Email subscriptions specifically

Email subscriptions require a human to click a link in an email. There is no
API path to confirm one on someone else's behalf.

- An `ops-alerts@` distribution list subscribed to the primary alerting topic
  and *not* to the standby's is a **failover blind spot in your alerting
  itself** — the failure mode where you lose the region and simultaneously lose
  the ability to be told about it.
- **Do not subscribe humans to SNS topics for production alerting.** Subscribe a
  chat/paging integration that you can confirm programmatically, or route
  through a queue. Email subscriptions are for the console-clicking phase of a
  project and they quietly become load-bearing. See
  [[observability-multi-region]].
- Also relevant: **email delivery is hard-capped at 10 messages/second per
  subscription** and AWS states this *"is a hard limit and can't be increased"*.
  An alert storm during a failover will be shaped by that number.

---

## Replication / mirroring options — the three architectures

There are exactly three shapes. Every SNS multi-region design in the wild is one
of them or a hybrid. Each is described here with what it actually buys, what it
costs, and where it is correct.

Throughout, assume the EU pair: topic `orders-prod`, primary `eu-west-1`,
standby `eu-west-2`.

---

### Option (a) — Dual-publish: the application publishes to both regions

The producer holds two SNS clients and calls `Publish` twice, once against
`arn:aws:sns:eu-west-1:…:orders-prod` and once against
`arn:aws:sns:eu-west-2:…:orders-prod`. Each topic has its own local
subscribers. Nothing crosses a region boundary except the second `Publish`.

```
                    ┌──────────────── eu-west-1 ────────────────┐
                    │  topic orders-prod → local SQS, local λ   │
producer ──Publish──┤                                           │
    └────Publish────┤  eu-west-2                                │
                    │  topic orders-prod → standby SQS, λ (off) │
                    └───────────────────────────────────────────┘
```

**What it buys.** This is the only option on the list that **works during the
primary's outage**. Options (b) and (c) both route through a topic in the
primary region; if `eu-west-1`'s SNS control plane or data plane is degraded,
the publish itself fails. Dual-publish degrades to single-publish: the
`eu-west-1` call errors, the `eu-west-2` call succeeds, and the standby has the
message. **For a genuinely resilient publish path there is no substitute.**

**What it costs.**

1. **Every producer changes.** Two clients, two calls, a partial-failure policy.
   This is application work across every publishing service and it is the real
   price. See the failure-semantics problem below.
2. **Partial failure is now your problem.** What does the producer do when
   `eu-west-1` succeeds and `eu-west-2` fails? Three sub-options:
   - *Fire-and-forget the standby publish* (async, ignore errors). Simple.
     Means the standby silently misses messages whenever `eu-west-2` hiccups,
     and you will not know. **Add a metric on standby-publish failures or this
     is invisible rot.**
   - *Fail the request if either publish fails.* Now your primary's
     availability is the product of two regions' availabilities — you have made
     the system **less** available by adding a DR mechanism. Almost always
     wrong.
   - *Local outbox + async replicator.* Write the intent durably, have a
     background worker publish to both and retry. Correct, and it is
     [[aws-sqs]]'s option (d) wearing a different hat. If you are going to do
     this, you have the outbox, and the outbox is the better answer.
3. **Doubled publish cost and doubled downstream processing**, unless the
   standby's subscribers are inert (which is the recommended posture — see
   [[messaging-in-flight-data-loss]]).
4. **Two topics, two sets of subscriptions, two sets of filter policies,
   drifting independently.** Managed by the Terraform module below.
5. **Double-counted against the standby's publish quota** — which, as the quota
   section shows, is 30× smaller in `eu-west-2` than in `eu-west-1`. A 400
   msg/s steady-state workload publishes fine in Ireland and gets throttled in
   London **in steady state**, not just at failover. This is the failure that
   catches dual-publish designs out.

**Verdict: correct for the small set of publish paths that must survive the
primary's outage** — the ones where the publish *is* the acknowledgement to a
customer. Wrong as an estate default, because it is an application change
everywhere and it buys nothing for workloads whose producers are themselves in
the dead region (which is most of them: if your EKS pods are in `eu-west-1` and
`eu-west-1` is gone, there is no producer left to dual-publish).

> **That last point deserves emphasis because it undercuts a lot of DR
> thinking.** In an active/passive estate the producers live in the primary.
> When the primary dies, the producers die with it. Dual-publish protects the
> narrow window where SNS is degraded but compute is not — a real scenario, but
> a much narrower one than "the region is gone". Do not buy dual-publish
> expecting it to cover the full outage; it covers a partial one.

---

### Option (b) — Single primary topic with cross-region subscribers

One topic in `eu-west-1`. It has a subscription to the local `eu-west-1` queue
*and* a native cross-region subscription to the `eu-west-2` standby queue. No
code, no second publish.

```
                 ┌──────── eu-west-1 ────────┐
producer ──Pub──►│ topic orders-prod         │──► local SQS (consumed)
                 │                           │──► SQS in eu-west-2 (idle)
                 └───────────────────────────┘
```

**What it buys.**

- **Zero application change.** The producer publishes once, to one ARN, exactly
  as today. This is the reason the option exists and it is a big reason.
- **The standby queue is continuously populated** with everything the primary
  queue received. At failover the standby has the full backlog, not an empty
  queue. RPO for queued work drops from "whatever was in flight" to "SNS
  delivery latency", i.e. sub-second.
- **SNS retries AWS-managed endpoints 100,015 times over 23 days**
  ([delivery retries](https://docs.aws.amazon.com/sns/latest/dg/sns-message-delivery-retries.html)).
  If `eu-west-2` is the region that is down, SNS holds those messages and keeps
  trying for **longer than SQS's own 14-day maximum retention**. The
  cross-region subscription is more durable than the queue it feeds.
- AWS ships a reference implementation:
  [aws-samples/sample-sns-sqs-multi-region](https://github.com/aws-samples/sample-sns-sqs-multi-region),
  which builds exactly this across `us-east-1`/`us-west-2` and walks a regional
  failover.

**What it costs.**

1. **The topic is in the primary region. This is the fatal flaw and it is not
   fixable within the option.** If `eu-west-1` is what broke, the topic you
   publish to is gone. Option (b) recovers the backlog that existed *before* the
   outage; it does nothing *during* it. **Option (b) alone is not a failover
   design. It is a backlog-preservation design.** Anyone presenting it as the
   answer should be asked "where do we publish at 03:05?" and the honest answer
   is "the standby topic, which we also need, which is option (c)".
2. **Cross-region data transfer on every message**, charged to the sending side
   (see Cost).
3. **Filter policies now have to be right in two places** for what is logically
   one fan-out. See the filter-policy section.
4. **The subscription must be created with the topic's provider**, which in a
   two-provider module is a real source of confusion.
5. **Failback is asymmetric** unless you build it symmetric. Covered in
   [[aws-sqs]]'s failback section — the argument is identical and lands on
   *symmetric topics*.

**Verdict: correct as a *supplement*, never as the whole answer.** Use it for
the handful of queues where a preserved backlog is worth real money, on top of
option (c). It is also the only option that requires no application change,
which makes it the pragmatic first move while the application work for (a)/(d)
is scheduled.

---

### Option (c) — Independent topics per region; the publisher chooses at failover

Both regions have a full, identical topic with a full, identical set of *local*
subscriptions. Nothing crosses regions. The producer's configuration names one
region; at failover, the configuration changes.

```
  eu-west-1: topic orders-prod → local SQS, local λ   ◄── producers (today)
  eu-west-2: topic orders-prod → local SQS, local λ   ◄── producers (after failover)
```

**What it buys.**

- **The standby is a complete, self-contained copy.** No dependency on the
  primary for anything. This is what "warm standby" means and it is the shape
  the rest of the estate is already taking (Secrets Manager replicas, DynamoDB
  Global Tables).
- **Zero steady-state cost.** No cross-region transfer, no double publishing,
  no double processing. An idle SNS topic with idle subscriptions costs nothing
  — SNS bills per request, and a topic nobody publishes to generates no
  requests.
- **Failover is a configuration change, not an infrastructure change.** No
  `terraform apply` in the critical path. With same-name topics (recommended,
  per [[aws-sqs]]) the app's config is a region, not an ARN.
- **Failback is free and symmetric.** There is no directionality encoded
  anywhere.
- **It composes with everything.** (a) and (b) are both *additions* to (c). You
  need (c) regardless; the only question is whether you also buy (a) or (b) on
  top.

**What it costs.**

1. **Messages published to the primary before the failover are stranded** in
   the primary's topic-and-queue pipeline. This is exactly [[aws-sqs]] option
   (a)'s exposure, and [[messaging-in-flight-data-loss]] argues at length that
   it is seconds, not hours. **It is inside the 2h RPO by two orders of
   magnitude on a healthy system.**
2. **Everything must be kept in lockstep** — subscriptions, filter policies,
   delivery policies, DLQ wiring, KMS keys, access policies. This is a Terraform
   problem and the module below solves it with a single `for_each` over one
   list.
3. **The standby's subscriptions must be pre-confirmed.** See the confirmation
   section. This is the work item that makes (c) achievable in 15 minutes
   instead of not.

**Verdict: this is the baseline and it is non-negotiable.** Build (c) for every
topic in the estate.

---

### Recommendation

**(c) everywhere, as the foundation. (b) added selectively, for the queues
whose backlog is worth preserving. (a) only for publish paths that are
themselves the customer acknowledgement.**

| | (a) Dual-publish | (b) Cross-region subscribers | (c) Independent topics |
|---|---|---|---|
| Application change | **Yes, everywhere** | None | None (config only) |
| Works during primary outage | **Yes** (partially) | No | Yes (after cutover) |
| Preserves pre-outage backlog | Yes | **Yes** | No |
| Steady-state cost | 2× publish + 2× processing | x-region transfer | **£0** |
| Failback | Symmetric | Needs symmetric design | **Free** |
| Standby quota pressure | **Yes, continuously** | No | Only after failover |
| FIFO-safe | Yes (new dedup context) | Yes (SNS preserves order) | Yes |
| Recommend | Narrow, high-value publish paths | Selected queues | **Estate default** |

**Which workloads change the answer:**

- **A payments/order-intake API that returns `202` on publish** → you need (a),
  or better, you need to move the durable write in front of the acknowledgement
  ([[messaging-in-flight-data-loss]]'s central recommendation, which is cheaper
  and removes the exposure rather than mitigating it). **Prefer the outbox to
  dual-publish.**
- **A queue whose backlog is customer-visible work with no database
  re-derivation path** (inbound third-party webhooks, enriched payloads) → add
  (b) for that one topic.
- **Alerting and operational topics** → (c) only, but with the confirmation
  readiness check treated as a hard gate, because these are the ones with
  HTTPS and email subscribers.
- **Anything fan-out-heavy with expensive consumers** → (c) only. (b) doubles
  the consumer cost if the standby consumers run, and the standby queue grows
  unboundedly if they do not.
- **FIFO topics** → (c) only. See below.

---

## FIFO topics across regions

FIFO topics are a different product wearing the same name, and the differences
all matter for DR.

### What a FIFO topic can and cannot deliver to

From [Amazon SNS message delivery for FIFO topics](https://docs.aws.amazon.com/sns/latest/dg/fifo-message-delivery.html):

> *"Amazon SNS FIFO topics can't deliver messages to customer managed
> endpoints, such as email addresses, mobile apps, phone numbers for text
> messaging (SMS), or HTTP(S) endpoints. These endpoint types aren't guaranteed
> to preserve strict message ordering. Attempts to subscribe customer managed
> endpoints to Amazon SNS FIFO topics result in errors."*

| Subscriber | FIFO topic? | Note |
|---|---|---|
| SQS **FIFO** queue | Yes | The intended pairing. Order and dedup preserved end to end. |
| SQS **standard** queue | Yes | Allowed — gives best-effort ordering and at-least-once. Useful escape hatch. |
| Lambda (direct) | **No** | Fan out via an SQS queue, then trigger the function from the queue. |
| HTTP/S, email, SMS, mobile push | **No** | `Subscribe` errors. |
| Firehose | Not documented for FIFO. Assume no. | |

**The DR-relevant consequence is a good one:** because FIFO topics cannot have
HTTP/S or email subscribers, **FIFO topics are structurally immune to the
subscription-confirmation trap.** Every legal FIFO subscriber is an SQS queue
in your own account, which auto-confirms. If a topic is FIFO, you can stop
worrying about `PendingConfirmation` for it entirely. That is worth knowing
when triaging which topics need the readiness check.

### Cross-region FIFO delivery — answered

[[aws-sqs]] left this as open question #5: *"Can an SNS FIFO topic have a
cross-region SQS FIFO subscriber?"* There is now a citable answer, though it is
indirect. The
[SNS quotas page](https://docs.aws.amazon.com/general/latest/gr/sns.html)
states, in a footnote to the FIFO publish-throughput table:

> *"Amazon SNS FIFO topics can experience reduced throughput within a message
> group for cross Regional deliveries due to the added latency between Regions,
> and the need to maintain the strict order of messages."*

AWS would not document the throughput characteristics of a thing that cannot
happen. **Cross-regional delivery from a FIFO topic is supported**, and AWS is
warning you that it is slower — because strict ordering within a message group
means the next message cannot be delivered until the previous one is
acknowledged, and now that acknowledgement has a cross-region round trip in it.

**This is a genuine, quantifiable degradation and nobody models it.** Within a
message group, throughput is bounded by `1 / (inter-region RTT + processing)`.
London↔Ireland RTT is on the order of ~10 ms, so a single message group might
be bounded at something like tens of messages per second rather than hundreds —
but **I found no AWS-published number and no public benchmark, so do not take
that arithmetic as a measurement.** If a cross-region FIFO subscription is on
the design, benchmark it in a sandbox with your real message-group cardinality
before committing. Record the result in [[dr-testing-and-gamedays]].

*Update [[aws-sqs]]'s open question 5 to "answered, with a throughput caveat".*

### Deduplication scope — the thing that breaks

From [message deduplication for FIFO topics](https://docs.aws.amazon.com/sns/latest/dg/fifo-message-dedup.html):

> *"Message deduplication applies to an entire Amazon SNS FIFO topic when the
> topic attribute `FifoThroughputScope` is set to `Topic`. When the topic
> attribute `FifoThroughputScope` is set to `MessageGroup`, message
> deduplication applies to each individual message group."*
>
> *"If a message with a particular deduplication ID is successfully published
> to an Amazon SNS FIFO topic, any message published with the same
> deduplication ID, within the five minute deduplication interval, is accepted
> but not delivered."*

Two facts, both fatal to naive cross-region thinking:

1. **The dedup scope is the topic (or a message group within it). There is no
   account-wide or cross-region dedup namespace.** A message dual-published to
   the `eu-west-1` topic and the `eu-west-2` topic with the same
   `MessageDeduplicationId` is **two independent messages**. The standby topic
   has never heard of that ID. Both get delivered.
2. **The window is five minutes. Your RTO is fifteen.** Dedup cannot span a
   failover by construction. This is exactly the point [[aws-sqs]] makes about
   FIFO queues and it is equally true one layer up.

**So under option (a) dual-publish, a FIFO topic delivers every message twice —
once per region — and calls it exactly-once in both.** If the standby's
consumers are inert this is harmless. If they are live, you have duplicate
processing wearing an exactly-once label, which is worse than plain duplicate
processing because nobody looks for it.

### Filtering silently downgrades the delivery guarantee

Buried in the same page, and it is the best gotcha in this note:

> *"[Exactly-once delivery holds as long as] the Amazon SNS subscription topic
> has no message filtering. **When you configure message filtering, Amazon SNS
> FIFO topics support at-most-once delivery**, as messages can be filtered out
> based on your subscription filter policies."*

Adding a filter policy to a FIFO subscription moves it from **exactly-once** to
**at-most-once**. That is a downgrade in the guarantee, applied by editing a
JSON blob, with no warning anywhere in the console or in `terraform plan`.

Read it carefully — AWS is being precise, not sloppy. The point is that a
filter policy is a place where a message can be legitimately dropped, so "every
message arrives exactly once" stops being a statement you can make about the
subscription. If a downstream system was written on the assumption that it sees
every message in a group in order, and someone later adds a filter policy "just
to reduce noise", that assumption has been quietly voided.

**Put this in the PR template for filter-policy changes on FIFO subscriptions.**

### Other FIFO constraints that bite in a standby

| Constraint | Value | Why it matters in the standby |
|---|---|---|
| FIFO topics per account | **1,000** (vs 100,000 standard) | Per account *per region*, so the standby has its own 1,000. Unlikely to bind, but it is 100× tighter than standard — an estate that FIFO-ifies everything will hit it. |
| Subscriptions per FIFO topic | **100** (vs 12,500,000 standard) | A fan-out of more than 100 queues is impossible on FIFO. Discover this at design time, not at failover. |
| FIFO topic name | Must end `.fifo` | And `name` is **ForceNew**. See migration. |
| `fifo_topic` | **ForceNew** | A standard topic cannot become FIFO. Ever. |
| Per-message-group throughput | 300 msg/s max | Per AWS's quota footnote. |
| Per-topic throughput (`FifoThroughputScope = Topic`) | 3,000 msg/s **or 20 MB/s, whichever comes first** | The MB/s limit is the one people forget. |
| Publish quota, FIFO | **3,000 msg/s in "all other Regions"** | Note this is *higher* than standard's 300 in those regions. See quotas. |

That last row is genuinely surprising and worth stating plainly: **in
`eu-west-2`, `ca-central-1` and `ca-west-1`, the default FIFO publish quota
(3,000/s) is ten times the default standard publish quota (300/s).** If you are
choosing between standard and FIFO for a high-volume topic in a non-tier-1
region, the default quotas point the opposite way to intuition. (Both are soft
quotas; raise whichever you need. But the defaults are what you get at 3am if
nobody raised anything.)

### Recommendation for FIFO

**Option (c) only — independent FIFO topics per region, no cross-region
subscriptions, no dual-publish.** Reasons, in order:

1. Cross-region FIFO delivery works but degrades throughput by an unmeasured
   amount within each message group.
2. Dual-publish gives every message two independent dedup contexts, voiding the
   guarantee the topic exists to provide.
3. FIFO topics have no HTTP/S or email subscribers, so the main argument for
   pre-warming a cross-region path (confirmation) does not apply.
4. FIFO topics have the archive, which is a better recovery mechanism than any
   of them — see next.

---

## Message archiving and replay — the one real recovery point

This is the most consequential SNS feature for RPO in the entire messaging
family, and it exists only for FIFO topics.

### What it is

From
[Amazon SNS message archiving and replay for FIFO topics](https://docs.aws.amazon.com/sns/latest/dg/fifo-message-archiving-replay.html):

> *"Amazon SNS provides a no-code message archiving and replay feature,
> specifically designed for FIFO topics. This feature allows topic owners to
> store messages directly within the topic archive for up to 365 days and
> replay them to subscribers when needed. Message archiving and replay are
> essential for recovering lost messages and **synchronizing applications
> across regions** or systems by replicating states."*

AWS names cross-region state synchronisation as a headline use case. That is
worth reading twice: **this is the only AWS messaging primitive that gives you
a durable, addressable, time-indexed recovery point.** Everything else in
[[aws-sqs]] and [[messaging-in-flight-data-loss]] is about bounding loss;
this is about eliminating it.

### The mechanics

**Topic side — `ArchivePolicy`.** A topic attribute, settable at `CreateTopic`
or `SetTopicAttributes`:

```json
{ "ArchivePolicy": { "MessageRetentionPeriod": "30" } }
```

- Retention: **minimum 1 day, maximum 365 days.**
- **A2A FIFO topics only.** AWS: *"Amazon SNS message archiving and replay is
  only available for application-to-application (A2A) FIFO topics."*
- `GetTopicAttributes` returns **`BeginningArchiveTime`** — the oldest
  timestamp from which a replay can start. **This is literally your recovery
  point, exposed as an API field.** Export it as a metric.
- Metrics: `ApproximateNumberOfMessagesArchived` and
  `ApproximateNumberOfBytesArchived` (60-minute resolution),
  `NumberOfMessagesArchiveProcessing` and `NumberOfBytesArchiveProcessing`
  (1-minute resolution).

**Subscriber side — `ReplayPolicy`.** A subscription attribute:

```json
{
  "PointType": "Timestamp",
  "StartingPoint": "2026-09-21T02:00:00.000Z"
}
```

- `StartingPoint` required, `EndingPoint` optional, `PointType` must be
  `Timestamp` (the only supported value today).
- Replayed messages keep the **same content, `MessageId` and `Timestamp`** as
  the original, and gain a boolean **`Replayed`** attribute. Consumers can
  therefore tell a replay from live traffic — build that into the consumer now,
  because it is the hook you will want at 3am.
- `ReplayStatus`: `Pending` → `In progress` → `Completed` | `Failed`.
- Metrics: `NumberOfReplayedNotificationsDelivered` and
  `NumberOfReplayedNotificationsFailed`, 1-minute resolution.
- Filter policies apply to replays: SNS retrieves from the archive, then
  evaluates the subscription's `FilterPolicy`.

### The `EndingPoint` trap — read this before your runbook says otherwise

AWS, emphasis theirs:

> *"If an `EndingPoint` is specified, the service will replay messages from the
> `StartingPoint` up to the `EndingPoint` and then stop. **This action
> effectively pauses the subscription.** While the subscription is paused,
> newly published messages will not be delivered to the subscribed endpoint."*

**A bounded replay leaves the subscription dead.** It does not resume. You must
apply a *second* `ReplayPolicy` with no `EndingPoint`, starting from the
previous `EndingPoint`, to bring it back to live.

Imagine this at 03:10 during a failover: an engineer replays "just the last two
hours" into the standby subscription to catch up, it completes, everyone moves
on — and that subscription now receives **nothing** for the rest of the
incident, with no error and no alarm. It is the confirmation trap again in a
different costume: a silently inert subscription.

**Runbook rule: at failover, always replay with an open `EndingPoint`.** The
`StartingPoint` bounds what you recover; the absence of an `EndingPoint`
guarantees you rejoin live traffic. Use bounded replays only for deliberate,
non-urgent, forensic recovery, and only with a documented second step to
resume.

### What it does to the RPO answer

For a FIFO topic with an archive policy and a cross-region subscriber:

| Scenario | RPO without archive | RPO with archive |
|---|---|---|
| Standby queue missed messages while `eu-west-2` was degraded | Whatever SNS failed to deliver after 23 days of retries | **Zero**, replay from before the gap |
| Standby topic was never populated (option (c)) | Everything since cutover | Not applicable — the archive is on the *primary's* topic |
| Primary region gone entirely | N/A | **The archive is in the primary region and is unreachable** |

That last row is the limit, and it must be stated as loudly as the feature
itself: **the archive is a regional resource. It lives on the topic. If the
region hosting the topic is gone, so is the archive.** Archiving does not
replicate. It is not a cross-region backup.

**So what is it actually good for in this estate?**

1. **Recovering the standby after a *standby* outage.** Primary healthy,
   `eu-west-2` down for six hours, cross-region subscription failing. SNS
   retries for 23 days so you are probably fine anyway — but if you exceeded
   that, or if the subscription was misconfigured and messages were dropped
   rather than retried, replay closes the gap exactly.
2. **Recovering after a *partial degradation* of the primary**, where the topic
   survived and the consumers did not. This is the common case and replay is
   perfect for it.
3. **Failback.** After running in `eu-west-2` for a day, you need `eu-west-1`'s
   subscribers to catch up on what they missed. If the standby topic also has an
   archive policy, replay is the clean mechanism — better than draining queues
   or re-publishing.
4. **Game days.** Replaying yesterday's real traffic into a standby subscriber
   is the highest-fidelity DR test available, and it costs an API call. See
   [[dr-testing-and-gamedays]].

**Recommendation: enable `ArchivePolicy` on every FIFO topic in both regions,
at 7 days minimum.** It is an in-place attribute change (not ForceNew), it
requires no application change, and it converts a class of "we lost messages"
incidents into "we replayed them". Go longer than 7 days only where the data
justifies the storage bill. **It does not substitute for option (c)** — you
still need a standby topic — but it makes every other recovery step cheaper.

### Two Terraform-shaped landmines in the archive

Both from
[message archiving for FIFO topic owners](https://docs.aws.amazon.com/sns/latest/dg/message-archiving-and-replay-topic-owner.html):

> *"To avoid accidental message deletions, you can not delete a topic with an
> active message archive policy. The topic's message archive policy must be
> deactivated before the topic can be deleted. **When you deactivate a message
> archive policy, Amazon SNS deletes all of the archived messages.**"*

1. **`terraform destroy` on an archived topic fails.** The delete is rejected
   while the archive policy is active. In a cookiecutter monorepo that tears
   down ephemeral environments, this will break your teardown pipeline the
   first time someone adds an archive policy to a dev topic. The workaround is a
   two-step apply (set `archive_policy = "{}"`, then destroy), which is exactly
   the kind of thing that needs documenting before it bites.
2. **Setting `archive_policy` to `{}` irreversibly deletes the archive.** A
   Terraform change that removes the `archive_policy` argument — a one-line
   diff, shown in the plan as an innocuous attribute update, **not** as a
   destroy — wipes up to 365 days of messages. `terraform plan` will not warn
   you. **Treat archive-policy removals as destructive changes** and gate them
   the same way you gate a database drop.

### Encrypted archives need an explicit KMS grant

If the topic is encrypted, replay needs SNS to decrypt messages it archived
possibly months ago. AWS requires this on the key policy:

```json
{
  "Sid": "Allow SNS to decrypt archived messages",
  "Effect": "Allow",
  "Principal": { "Service": "sns.amazonaws.com" },
  "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
  "Resource": "*"
}
```

AWS: *"If encryption is enabled on a topic, and the KMS key is disabled or
deleted, or the KMS key policy is not correctly configured for Amazon SNS,
Amazon SNS cannot replay messages to your subscribers."*

**A rotated-out, disabled or deleted KMS key turns your archive into
ciphertext you own and cannot read.** Since the archive can hold 365 days of
messages, the key must outlive the retention period. This is a direct
constraint on key lifecycle — put it in [[aws-kms]] and in whatever runbook
governs key deletion. A 30-day KMS deletion window and a 365-day archive are
incompatible.

---

## Filter policies — the silent-drift surface

A filter policy is a **subscription attribute**, not a topic attribute. Two
regions, two topics, two sets of subscriptions, two sets of filter policies. If
they diverge, the standby delivers a different set of messages to the same
logical subscriber — and it does so silently, correctly, with no error.

**This is the most likely way a warm standby is wrong on the day and nobody
notices beforehand**, because a filter policy has no health signal. A queue
that receives 60% of the messages it should looks exactly like a queue that
receives 100%. There is no `FilteredOutMessages` alarm to set. (SNS does emit
`NumberOfNotificationsFilteredOut`, but a *correct* number is workload-specific,
so you cannot alarm on an absolute value — only on a sudden change, which is
after the fact.)

### The constraints you must mirror exactly

From [filter policy constraints](https://docs.aws.amazon.com/sns/latest/dg/subscription-filter-policy-constraints.html):

| Constraint | Limit |
|---|---|
| Total combination of values (product of array lengths) | **150** |
| Keys per filter policy | **5** (leaf keys only, for payload-based) |
| Maximum filter policy size | **256 KB** |
| Filter policies per topic | **200** (default, raisable via Service Quotas) |
| Filter policies per AWS account | **10,000** (default) |
| Numeric range | -10⁹ to 10⁹, five decimal places |
| String matching | Case-sensitive |
| Wildcard complexity | ≤ 100 points total; max 3 wildcards per pattern |
| Attribute-based filtering | **No nesting.** Only `String` and `String.Array` attribute types; `Binary` ignored |
| Payload-based filtering | Nesting allowed; nested level multiplies into the 150 combination budget |

Two practical notes:

- **Attribute-based filtering requires double-escaping `"` and `\`.** AWS:
  *"Failure to double-escape these characters will result in the filter policy
  not matching the attributes of a published message, and the notification
  won't be delivered."* In Terraform, where the policy is usually
  `jsonencode()`d, this is a real source of "works in one environment, silently
  drops in another".
- **The 10,000-per-account limit is documented without a region qualifier.**
  AWS service quotas are normally per-account-per-region, and the SNS quotas
  page lists it under regional quotas, so it is almost certainly per-region —
  meaning mirroring your topics does not consume the primary's budget.
  **I could not find an explicit AWS statement either way.** If the estate is
  anywhere near 10,000 filter policies, confirm with Support before assuming
  the standby is free.

### The 15-minute propagation delay — an RTO fact, not a footnote

> *"AWS services such as IAM and Amazon SNS use a distributed computing model
> called eventual consistency. **Additions or changes to a subscription filter
> policy require up to 15 minutes to fully take effect.**"*
> — [Amazon SNS subscription filter policies](https://docs.aws.amazon.com/sns/latest/dg/sns-subscription-filter-policies.html)

**Up to 15 minutes. The RTO is 15 minutes.** Changing a filter policy at
failover time consumes the entire failover budget on its own, and the change is
not observable while it propagates — you cannot tell whether it has taken
effect except by publishing test traffic and watching.

The rule that follows is absolute:

> **No filter policy may be changed as part of a failover. Ever. The standby's
> filter policies must be correct and settled before the incident starts.**

This also rules out a tempting design — "the standby's subscriptions have a
`{"env": ["never"]}` filter policy that drops everything, and we flip it at
failover to open the floodgates." It is an elegant-looking way to pre-create
subscriptions without them doing anything. **Do not build it.** It converts a
filter policy into a failover switch that takes up to 15 minutes to throw. Use
disabled consumers for fencing instead ([[messaging-in-flight-data-loss]]
argues this at length), not filter policies.

### Keeping them in lockstep in a cookiecutter monorepo

The only reliable mechanism is structural: **one definition, two
instantiations.** Do not maintain a `filter_policy` per region.

```hcl
locals {
  # THE list. Both regions derive from it. Drift is impossible by construction.
  topics = {
    orders = {
      fifo = false
      subscriptions = {
        fulfilment = {
          protocol      = "sqs"
          queue_key     = "fulfilment"
          filter_policy = { event_type = ["order.placed", "order.amended"] }
        }
        analytics = {
          protocol      = "sqs"
          queue_key     = "analytics"
          filter_policy = null            # receives everything
        }
      }
    }
  }
}
```

Then `for_each` the same map through both provider aliases (full module below).
A reviewer reading the diff sees one change to one filter policy and knows both
regions moved together. **There is no review checklist that achieves this and
no amount of discipline that substitutes for it.** See [[module-patterns]].

Two guard rails on top:

1. **A drift detector.** A scheduled job that fetches
   `GetSubscriptionAttributes` for every subscription in both regions, canonicalises
   the `FilterPolicy` JSON (sort keys, sort arrays — JSON key order is not
   stable), and alarms on any logical difference. This catches out-of-band
   console edits, which are the realistic cause of drift in a mature estate.
   Roughly the same construct as the confirmation readiness check, and it should
   be the same Lambda.
2. **A `terraform plan` in CI on a schedule**, not just on PRs. A nightly
   no-change plan that comes back non-empty is drift. Cheap, and it catches the
   same class of problem one layer up.

---

## The in-flight loss window, extended to fan-out

[[messaging-in-flight-data-loss]] establishes the frame: the question is not
"was it in a queue" but "did we acknowledge it to something we cannot ask
again". That analysis holds unchanged for SNS and is not repeated. What is
genuinely different about SNS is **fan-out multiplies the number of distinct
in-flight states for a single publish.**

### One publish, N delivery states

A message published to a topic with five subscribers is not in one state. It is
in five, independently:

```
Publish(m) → topic accepted, 200 OK to producer
              ├─ sub 1 (local SQS)        → delivered, in queue, unconsumed
              ├─ sub 2 (cross-region SQS) → delivered
              ├─ sub 3 (Lambda)           → invoked, mid-execution
              ├─ sub 4 (HTTPS partner)    → 503, retry #7, backoff phase
              └─ sub 5 (email)            → queued at SMTP
```

Kill the region here. Five different outcomes, and the producer got one `200`.

**The consequence that matters: at failover, "has this message been processed"
has no single answer.** The recovery question is per-subscriber, and a
reconciliation script that asks "did we handle order 1234" must ask it of five
systems. This is *not* an argument for replicating the topic; it is an argument
that **idempotency must be per-subscriber and reconciliation must be
per-subscriber**. [[messaging-in-flight-data-loss]]'s per-queue classification
exercise should therefore be a **per-subscription** exercise for anything
behind a topic.

### The retry window is a very large safety net — larger than most people think

| Endpoint class | Protocols | Retry schedule | Total |
|---|---|---|---|
| **AWS managed** | SQS, Lambda, Firehose | 3 immediate, 2 @ 1s, 10 exponential 1s→20s, then **100,000 @ 20s** | **100,015 attempts over 23 days** |
| **Customer managed** | SMTP (email), SMS, mobile push | 0 immediate, 2 @ 10s, 10 exponential 10s→600s, then 38 @ 600s | **50 attempts over 6 hours** |
| **HTTP/S** | http, https | Customer-definable | Default 3 retries; max `numRetries` 100; **total policy retry time capped at 3,600 s (hard)** |

This is the single most under-used fact about SNS in a DR context, and it is
worth spelling out what it means concretely:

**If your architecture is producer → SNS → SQS, and the SQS queue's region goes
dark, SNS holds those messages and retries for 23 days — longer than SQS's own
maximum retention of 14 days.** An SNS-fronted queue is strictly more durable
across a regional event than a directly-written queue, for free, with no
replication, no code, and no cost beyond the delivery itself.

**Therefore: putting a topic in front of a queue is a resilience improvement
even when you do not need fan-out.** [[aws-sqs]] flags this; here is the
recommendation in full:

> **For any queue whose messages are not re-derivable, front it with an SNS
> topic even if it has exactly one subscriber.** The cost is one extra publish
> request per message. The benefit is that a 23-day regional outage of the
> queue's region does not lose data. That is an excellent trade and it requires
> a one-line producer change (`Publish` instead of `SendMessage`).

**The three caveats:**

1. **Only for AWS-managed endpoints.** HTTP/S gets an hour at most; email and
   SMS get six hours. A partner webhook is not protected by this at all, and a
   6-hour regional event will exhaust every customer-managed subscription.
2. **The 23 days is for *server-side* errors.** Client-side errors get **no
   retries at all**. AWS: *"Amazon SNS doesn't retry the message delivery that
   fails because of a client-side error."* A deleted queue, a revoked queue
   policy, a wrong service principal on a cross-region subscription — these are
   client errors. **The misconfigured cross-region subscription described
   earlier does not benefit from the 23-day window; it fails permanently on the
   first attempt.** This is why the `NumberOfNotificationsFailed` alarm is not
   optional.
3. **The messages are held inside SNS and you cannot see them, count them,
   inspect them or drain them.** There is no `ApproximateNumberOfMessagesInRetry`.
   The only visibility is `NumberOfNotificationsFailed` ticking up. You are
   trusting a black box — a well-behaved one, but a black box. Do not build a
   recovery plan whose step 1 is "check what's in the retry buffer".

### Delivery status logging — turn it on in the standby

`aws_sns_topic` exposes `sqs_success_feedback_role_arn`,
`lambda_failure_feedback_role_arn`, `http_success_feedback_role_arn` and
siblings, plus `*_success_feedback_sample_rate`. These write per-delivery
status to CloudWatch Logs.

**Recommendation: enable failure feedback (100%) on every topic in both
regions, and success feedback at a low sample rate (1–5%) in the standby
only.** Rationale:

- Failure feedback is how you find out *why* a delivery failed, as opposed to
  merely that it did. During a cross-region misconfiguration it is the
  difference between a five-minute fix and an hour of guessing.
- Success feedback at a low sample rate in the standby is a cheap liveness
  signal: it proves the standby topic's delivery path works end to end, which
  the fully-idle option (c) design otherwise never exercises. It is the topic
  equivalent of [[aws-sqs]]'s canary queue.
- Success feedback at 100% on a busy primary is an expensive CloudWatch Logs
  bill for very little. Sample it or leave it off.

See [[observability-multi-region]] for where those logs should land, given that
logs written in the dying region are logs you lose.

---

## Delivery retry policies and DLQs

### The DLQ is attached to the subscription, not the topic

This is the right design — it tells you *which* subscriber failed — but it
means **N subscriptions need N DLQ wirings, in both regions.**

### The same-region constraint, stated by AWS

> *"The Amazon SNS subscription and Amazon SQS queue must be under the same AWS
> account and Region."*
>
> *"The ARN must point to an Amazon SQS queue in the same AWS account and
> Region as your Amazon SNS subscription."*
> — [Amazon SNS dead-letter queues](https://docs.aws.amazon.com/sns/latest/dg/sns-dead-letter-queues.html)

So:

- **There is no cross-region SNS DLQ.** Every region's topics need their own
  DLQs, created by the same module.
- **A cross-region subscription's DLQ lives in the *topic's* region**, because
  the subscription lives in the topic's region. Re-read that: under option (b),
  a `eu-west-1` topic delivering to a `eu-west-2` queue has a DLQ **in
  `eu-west-1`**. So the messages that failed to reach the standby are parked in
  the primary — **the exact region you were trying to escape.** If the primary
  then dies, those messages are stranded twice over.
- **FIFO topic subscriptions need FIFO DLQs**, standard need standard. AWS:
  *"FIFO topic subscriptions use FIFO queues, and standard topic subscriptions
  use standard queues."* A mismatched DLQ is a misconfiguration that only shows
  up when something fails — i.e. on the worst day.
- **Encrypted DLQs need a customer-managed key** whose policy grants the SNS
  service principal KMS access. AWS says so explicitly. An SSE-SQS (AWS-owned
  key) DLQ **will not work** as an SNS subscription DLQ. This is a departure
  from [[aws-sqs]]'s general "prefer SSE-SQS, it's free and simpler" advice and
  it is a real exception — note it in both places.

### Is a DLQ in the dead region recoverable?

Same answer as [[aws-sqs]], and it is worth restating because SNS DLQs are
worse:

- The DLQ is an ordinary SQS queue. Its contents survive in the region for up to
  the retention period. **Set it to 14 days** — AWS itself recommends the
  maximum for DLQs.
- `StartMessageMoveTask` (SQS redrive) is same-region **and** only accepts DLQs
  whose source is another SQS queue. **A DLQ fed by SNS subscription failures is
  explicitly not a supported redrive source.** So the "just redrive it" recovery
  does not apply. You will write a script.
- And unlike an SQS-fed DLQ, there is nowhere to redrive *to*: the original
  destination was a subscription, and subscriptions are not writable targets.
  Recovery means reading the DLQ and re-publishing to the topic (creating
  duplicates for every other subscriber) or writing directly to the intended
  destination queue (bypassing the topic). **Both are bespoke. Decide which,
  and write it down, before you need it.** Neither is a five-minute decision at
  3am.

**Recommendation: one DLQ per subscription, per region, 14-day retention,
alarmed at depth > 0, created by the same module as the subscription.** Plus a
documented recovery procedure per topic that states which of the two recovery
paths applies. The DLQ topology is mechanical; the recovery decision is not.

---

## KMS encryption across the region boundary

Cross-refer [[aws-kms]] for key strategy; this is the SNS-specific surface.

**The rule that governs everything: a KMS key is regional, and an SNS topic can
only use a key in its own region.** AWS states the corollary for IAM: *"AWS KMS
requires explicitly naming the full ARN of KMSs in specific regions in the
`Resource` section of an IAM policy."* A standby topic configured with the
primary's key ARN is simply broken, and — critically — **it is broken in a way
that does not surface until the first publish**. `terraform apply` succeeds.
The topic exists. It looks healthy in the console.

### What must exist in the standby

| Thing | Requirement |
|---|---|
| The key | A key **in the standby region**: either a separate regional CMK, or a **multi-region key replica** (see [[aws-kms]]). |
| `kms_master_key_id` on the standby topic | Must name the standby-region key or alias. |
| The publisher's IAM policy | `kms:GenerateDataKey*` and `kms:Decrypt` on **the standby key's ARN**, plus `sns:Publish` on the standby topic. |
| The key policy | SNS service principal allowed `kms:GenerateDataKey*` + `kms:Decrypt` if AWS services publish to the topic; plus the archive-decrypt statement if there is an `ArchivePolicy`. |
| The destination SQS queue's key | A **different** key again, in the queue's region, with `sns.amazonaws.com` granted. |

### Aliases are the load-bearing trick

`alias/aws/sns` (the AWS-managed key) is *"unique for each account and region"*
— so referring to the alias rather than the key ARN makes the configuration
region-portable automatically. The same applies to your own aliases: create
`alias/messaging` in both regions, pointing at each region's key, and the
module's `kms_master_key_id = "alias/messaging"` resolves correctly in both.

**This is the single cleanest way to avoid the wrong-key-ARN failure**, and it
costs nothing. One caveat AWS flags: if you control access via
`kms:ResourceAliases` conditions, the customer-managed key *must* have an alias
associated, or the condition never matches.

### Two SNS-specific KMS facts worth knowing

1. **SSE encrypts the message body only.** Not the topic name, not the topic
   attributes, not the subject, message ID, timestamp or message attributes,
   not the data protection policy, not per-topic metrics. **If you are putting
   anything sensitive in a message attribute — and filter policies encourage
   exactly that — it is not encrypted at rest.** Worth a line in
   [[data-residency]].
2. **Backlogged messages are not retroactively encrypted.** AWS: *"A message is
   encrypted only if it is sent after the encryption of a topic is enabled.
   Amazon SNS doesn't encrypt backlogged messages."* Enabling SSE on a live
   topic is safe and in-place, but it does not cover what is already in the
   archive or in retry.

### The cost model of SSE, which is not obvious

AWS gives the formula for KMS API calls per topic:

```
R = B / D * (2 * P)
```

where `B` is the billing period in seconds, `D` is the data-key reuse period
(SNS reuses a data key for up to **5 minutes** = 300 s), and `P` is the number
of publishing principals. AWS's own worked example: one topic, one publisher,
January → `2,678,400 / 300 * 2 = 17,856` KMS requests/month.

Two consequences for a multi-region estate:

- **The cost scales with publishing *principals*, not message volume.** A topic
  with 20 publishing roles costs 20× the KMS calls of a topic with one,
  regardless of throughput. A microservice estate where every service publishes
  to a shared topic is the expensive shape.
- **An idle standby topic makes no KMS calls at all**, because nothing
  publishes. **SSE on the standby costs only the key's monthly fee** (and
  nothing extra if you use a multi-region replica of an existing key — though
  MRK replicas are billed as separate keys; check [[aws-kms]]).

### Recommendation

**Use a per-region customer-managed key fronted by an identical alias in both
regions.** Reasons: it keeps the module region-agnostic, it avoids multi-region
key complexity where there is no need for the *same key material* (SNS never
needs to decrypt the other region's ciphertext — messages do not cross as
ciphertext), and it keeps the blast radius of a key compromise regional.

**Use a multi-region key only if** the archive's ciphertext needs to be
readable from the other region — which, per the archive section, it cannot be
anyway, since the archive is not accessible cross-region. **So: no multi-region
key is needed for SNS.** That is a useful negative finding for [[aws-kms]] and
for [[cost-model]]: the SNS workstream does not create MRK demand.

**And verify it works.** Publish one real, encrypted message through the
standby topic to the standby queue and read it back. The wrong-key failure is
invisible until you do.

---

## SMS and email — the delivery channels that do not fail over

SNS's customer-managed channels are the part of the service that behaves least
like infrastructure and most like a vendor relationship, and they are the part
most likely to be forgotten in a DR plan.

### The $1 spend limit will stop your SMS dead

From
[Requesting increases to your monthly SNS SMS spending quota](https://docs.aws.amazon.com/sns/latest/dg/channels-sms-awssupport-spend-threshold.html):

> *"We set the spending quota for all new accounts at **$1.00 (USD) per
> month**. This quota lets you test the message-sending features of Amazon SNS.
> To request an increase, open a quota increase case in the AWS Support
> Center."*

And the case form requires you to choose *"the Region from which you'll be
sending messages"* — twice, once in case details and once in the Requests
section, with an explicit note that you must include it in the Requests
section. **The spend quota is per-region.** Raising it in `eu-west-1` does
nothing for `eu-west-2`.

Then, after approval:

> *"You must complete the following steps or your SMS spend limit will not be
> increased."* — you have to go into the SNS console **in that region** and set
> the account spend limit manually.

Add it up:

| Step | Time |
|---|---|
| Open a Support case for the standby region | Minutes |
| AWS initial response | *"within 24 hours"* |
| Approval, possibly with follow-up questions | Longer if they need more info |
| Manually set the spend limit in the standby region's console | Minutes |

**That is a minimum of 24 hours of lead time on a 15-minute RTO.** If SMS is on
any customer-facing path — OTPs, delivery notifications, 2FA — and the standby
region's spend quota is still at the $1 default, **failover breaks SMS and you
cannot fix it during the incident.** Not "slowly": at all.

**Action, and it is a prerequisite not a nicety: raise the SMS spend quota in
every standby region now, to match the primary, and set the console value.**
Then verify it with `aws sns get-sms-attributes --region eu-west-2` and assert
on `MonthlySpendLimit` in the readiness check.

### Everything else about SMS that is regional and does not replicate

| Thing | Regional? | Consequence at failover |
|---|---|---|
| Account spend limit | **Yes** | Above. 24h+ lead time. |
| SMS sandbox status | **Yes** | An account out of the sandbox in `eu-west-1` may still be **in** the sandbox in `eu-west-2`, where only verified destination numbers receive messages. Silently drops everything else. |
| Origination identities (long codes, short codes, sender IDs, 10DLC) | **Yes** | Purchased and registered per region. A short code in one region does not exist in another. **Registration takes weeks, not days.** |
| Opt-out list | **Regional** | Customers who opted out in the primary have not opted out in the standby. **This is a compliance problem, not just a technical one** — messaging an opted-out recipient breaches carrier rules and potentially PECR/TCPA. |
| Default SMS type / delivery-status attributes | **Yes** (`SetSMSAttributes`) | Set per region; drift silently. |
| Delivery rate | 20 msg/s promotional, 20 msg/s transactional | Same default everywhere; confirm it is raised in both. |

The opt-out row is the one that should worry a compliance owner. Flag it in
[[data-residency]] and to whoever owns marketing consent.

### Recommendation on SMS: get it out of SNS's failover path

**If SMS matters, do not fail it over — centralise it.** Two credible options:

- **Send all SMS from a single, fixed region** (e.g. always `eu-west-1`)
  regardless of which region is serving traffic, accepting that an SMS outage
  is possible during a regional event. Simple, and it means one set of
  origination identities, one opt-out list, one spend quota.
- **Move SMS to Amazon Pinpoint / AWS End User Messaging**, which is the
  service AWS now points SMS users toward, and handle its multi-region story
  separately. Out of scope for this note, but it should be a tracked decision.

Either way: **SNS SMS should not be a thing your failover runbook has to think
about**, because every lever it needs has a multi-day lead time.

### Email subscriptions

Covered under the confirmation trap, but three additional facts:

1. **Delivery is capped at 10 messages/second per email/email-json
   subscription**, and AWS states this *"is a hard limit and can't be
   increased."* During a failover-triggered alert storm this is a queue, not a
   firehose.
2. **Retries are 50 attempts over 6 hours** (customer-managed endpoint policy),
   not 23 days. An email subscriber is far less protected than an SQS one.
3. **SNS email is not a transactional email service.** No custom From address,
   no templating, no bounce/complaint handling, no DKIM. If email matters, it
   belongs in [[aws-ses]] — and SES has its own, much more serious multi-region
   story (per-region sending quotas, per-region domain verification, per-region
   DKIM records, and a dedicated-IP-warming problem). **SNS email subscriptions
   should exist only for operational plumbing, and per the confirmation section
   they should not exist even for that.**

---

## Quotas — where the standby is quietly smaller than the primary

All figures from
[Amazon SNS endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/sns.html).
This section is the one to bring to a design review.

### The publish throttle, per account, per region

This is a **soft** quota, and its defaults vary enormously by region.

| Region tier | Standard topics | FIFO topics |
|---|---|---|
| `us-east-1` | **30,000 msg/s** | 30,000 msg/s |
| `us-west-2`, `eu-west-1` | **9,000 msg/s** | 9,000 msg/s |
| `us-east-2`, `us-west-1`, `ap-south-1`, `ap-northeast-1/2`, `ap-southeast-1/2`, `eu-central-1` | 1,500 msg/s | 3,000 msg/s |
| **All other supported Regions** | **300 msg/s** | 3,000 msg/s |

Now map the three pairs:

| Pair | Primary | Standby | Standard-topic ratio |
|---|---|---|---|
| **EU** | `eu-west-1` — 9,000/s | `eu-west-2` — **300/s** | **30× smaller** |
| **US** | `us-east-1` — 30,000/s | `us-west-2` — 9,000/s | 3.3× smaller |
| **CA** | `ca-central-1` — 300/s | `ca-west-1` — 300/s | **1× — symmetric** |

**The EU pair has a 30× publish-quota cliff at the exact moment you fail over.**
If `eu-west-1` publishes at a sustained 1,000 msg/s — entirely ordinary — then
`eu-west-2` will throttle at 300 and return errors to producers from the first
second of the failover. Every publisher sees `ThrottledException`. The failover
"succeeds" and the system does not work.

> **This is the highest-value single action in this note: raise the SNS publish
> quota in every standby region, via Service Quotas, to at least the primary's
> level, and do it now.** It is a soft quota, so it is a request rather than a
> purchase, and an unused raised quota costs nothing. It is also **slow to
> obtain** — quota increases are a Support workflow, not an API call — which is
> precisely why it cannot be left to failover time.

Note also what this says about [[region-pair-selection]]: the CA pair, which is
under doubt for other reasons, is **the only pair without a quota asymmetry**.
That is a small point in Calgary's favour and should be recorded alongside the
failures.

### The control-plane throttle, per region

`ConfirmSubscription`, `CreateTopic`, `SetSubscriptionAttributes`,
`GetSubscriptionAttributes`, `GetTopicAttributes`, `SetTopicAttributes` and
siblings share a soft quota:

| Region | TPS |
|---|---|
| `us-east-1` | 3,000 |
| `us-west-2`, `eu-west-1` | 900 |
| `us-east-2`, `us-west-1`, `ap-south-1`, `ap-northeast-1/2`, `ap-southeast-1/2`, `eu-central-1` | 150 |
| `ca-central-1`, `eu-west-2`, `eu-west-3`, `eu-north-1`, `af-south-1`, `ap-east-1`, `ap-south-2`, `ap-northeast-3`, `eu-south-1/2`, `il-central-1`, `me-south-1`, `sa-east-1` | **30** |

Three observations:

1. **`eu-west-2` is at 30 TPS versus `eu-west-1`'s 900** — another 30× gap, on
   the control plane this time. A mass `ConfirmSubscription` or
   `SetSubscriptionAttributes` operation at failover (which this note argues
   you should never need, but plans fail) would be 30× slower in London.
2. **`ca-west-1` does not appear in any tier of this table.** Nor do several
   other newer regions. **AWS does not publish a control-plane TPS figure for
   Calgary.** That is an honest gap, not an inference — record it as an open
   question and confirm with Support if the CA pair proceeds.
3. **`Subscribe` and `Unsubscribe` are hard-capped at 100 TPS everywhere** and
   cannot be increased. Likewise `ListSubscriptionsByTopic` and `ListTopics` at
   30 TPS. Any tooling that enumerates subscriptions estate-wide must paginate
   with backoff.

### Resource quotas

| Resource | Default |
|---|---|
| Standard topics | 100,000 per account |
| **FIFO topics** | **1,000 per account** |
| Subscriptions, standard topic | 12,500,000 per topic |
| **Subscriptions, FIFO topic** | **100 per topic** |
| Firehose subscriptions | 5 per topic per subscription owner |
| **Pending subscriptions** | **5,000 per account** |
| Filter policies | 200 per topic; 10,000 per account |
| Message size | 262,144 bytes (256 KiB); up to 2 GB via the Extended Client Library |
| Message header | 16,384 bytes (16 KiB) |
| `PublishBatch` entries | 10 |
| Email delivery rate | **10 msg/s per subscription — hard limit** |
| SMS delivery rate | 20 msg/s promotional; 20 msg/s transactional |
| SMS account spend threshold | **$1.00 USD** |

Assume all of these are per-account-**per-region** unless AWS says otherwise,
which means **mirroring into a standby does not consume the primary's
allowance** — a genuinely helpful property, and the reason "create every topic
in the standby" is free of quota risk.

### The readiness check should assert on quotas, not just resources

Add to the standby readiness Lambda:

```
servicequotas:GetServiceQuota  →  SNS publish rate     ≥ primary's value
sns:GetSMSAttributes           →  MonthlySpendLimit    ≥ primary's value
```

**A standby that has the right resources and the wrong quotas is not a
standby.** Quotas are invisible in `terraform plan`, invisible in the console's
resource views, and they are exactly the sort of thing that is discovered at
03:07.

---

## RPO / RTO analysis

### Against RPO 2h: passes comfortably, with one genuine improvement over SQS

SNS's contribution to data loss is not measured in hours.

| Loss scenario | Exposure | Inside 2h? |
|---|---|---|
| Published, not yet delivered to any subscriber | Sub-second (SNS delivery latency) | Yes, by ~4 orders of magnitude |
| Delivered to standby queue, unconsumed | = [[aws-sqs]]'s exposure | Yes |
| In SNS retry backoff to an SQS/Lambda endpoint | **Recoverable for 23 days** | Yes — not a loss at all |
| In SNS retry backoff to an HTTP/S endpoint | Up to 1 hour, then discarded unless a DLQ is attached | Yes, but *is* a real loss |
| In SNS retry backoff to email/SMS | 6 hours, then discarded unless DLQ | Yes |
| Published to the primary topic; primary region gone | Everything since the last subscriber delivery | Seconds |
| **FIFO topic with `ArchivePolicy`, region survives** | **Zero — replay from any point up to 365 days** | Trivially |

**The two findings worth carrying into [[cost-model]] and the DR policy:**

1. **SNS improves on SQS's RPO for free.** The 23-day retry window to AWS-managed
   endpoints means an SNS-fronted queue loses less than a directly-written one.
   Fronting a queue with a topic is an RPO improvement disguised as an extra
   hop.
2. **FIFO + archive is the only true zero-RPO messaging mechanism in the
   estate** — within the region. Cross-region it is zero-RPO for *standby*
   outages and no help for *primary* outages.

**SNS is not what will blow the 2h RPO.** As [[messaging-in-flight-data-loss]]
argues, [[aws-rds-postgres]] and [[aws-elasticache-redis]] are the credible
threats. Budget attention accordingly.

### Against RTO 15m: conditional — and the condition is not about SNS

Here is where the fifteen minutes actually goes.

| Step | Time | Pre-provisioned? |
|---|---|---|
| Standby topic exists | 0 s | **Yes** — Terraform |
| Standby topic access policy applied | 0 s | **Yes** |
| Standby subscriptions exist | 0 s | **Yes** |
| Standby subscriptions are **`Confirmed`** | 0 s **if pre-confirmed** / **hours-to-never if not** | **This is the gate** |
| Filter policies correct and settled | 0 s if unchanged / **up to 15 min if changed** | **Never change at failover** |
| DLQs exist and wired | 0 s | **Yes** |
| KMS key + alias present, publisher granted | 0 s | **Yes** |
| Publish quota raised | 0 s **if pre-raised** / **days if not** | **Prerequisite** |
| SMS spend limit raised | 0 s **if pre-raised** / **24h+ if not** | **Prerequisite** |
| Producers repoint to the standby ARN | seconds | Config — the actual work |
| Consumers start | seconds–minutes | [[aws-eks]] / [[aws-lambda]], not SNS |

**Read that table as a list of prerequisites, because that is what it is.**
Every row is either zero seconds or catastrophically more than fifteen minutes.
There is no middle. SNS has no slow-but-survivable failover path: either the
standby was prepared or the failover fails on that dimension.

**Creating a topic at failover time is technically fast** (`CreateTopic` is one
API call, and in `eu-west-2` you get 30 TPS of them). **Do not.** Creating the
topic means creating the policy, the subscriptions, the filter policies, the
DLQ wiring and the KMS config at failover time — and the subscriptions would
start life in `PendingConfirmation`, plus any filter policy would need up to 15
minutes to settle. **A topic created at failover time cannot meet a 15-minute
RTO even though creating a topic takes a second.**

**Verdict: `meets_targets: conditional`.** SNS meets RPO 2h easily and meets
RTO 15m **if and only if** the five prerequisite rows above are satisfied
before the incident. The conditionality is entirely about preparation, and the
readiness check is what converts "we think we prepared" into "we know".

---

## Warm standby shape

While `eu-west-1` is healthy, `eu-west-2` contains:

| Resource | State | Cost while idle |
|---|---|---|
| Topics (standard + FIFO) | Created, never published to | **£0** — SNS bills per request |
| Subscriptions | Created and **`Confirmed`** | £0 |
| Filter policies | Applied, settled | £0 |
| Subscription DLQs | Created, empty, 14-day retention | £0 |
| Topic access policies | Applied | £0 |
| KMS key + `alias/messaging` | Present | ~$1/month per CMK |
| `ArchivePolicy` on FIFO topics | Active, empty | £0 until messages exist |
| Delivery-status feedback roles | Present, failure feedback on | Pennies |
| `NumberOfNotificationsFailed` alarms | Active, in OK state | Pennies |
| Confirmation readiness Lambda | Running every 5 min | Pennies |
| Publish quota | **Raised to primary parity** | £0 |
| SMS spend limit | **Raised to primary parity** | £0 |
| Consumers of the standby queues | **Inert** — see [[messaging-in-flight-data-loss]] | Compute, decided there |

**An idle SNS topic is free.** There is no per-topic, per-subscription or
per-month charge. Like SQS, this means **there is no cost argument against
pre-provisioning every topic in the standby**, including ones you are sure you
will never need. Create them all.

The only non-trivial idle costs are the KMS key, the readiness Lambda, and —
under option (b) — cross-region data transfer, which is not idle at all.

---

## Terraform implementation

One module, instantiated once per region through a provider alias. The module
is region-agnostic and inherits whatever provider it is given. This is the
property that makes it fit a cookiecutter monorepo.

### Providers

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

See [[provider-aliases-vs-separate-stacks]] for the broader argument. For SNS
specifically, aliases in one state are *required* for option (b), because a
cross-region subscription resource needs the topic's provider while the queue
lives under the other one — a single plan has to see both.

### Module variable surface

```hcl
# modules/sns-topic/variables.tf

variable "name" {
  description = "Topic name, without the .fifo suffix. Identical in both regions."
  type        = string
}

variable "fifo" {
  description = "FIFO topic. Appends .fifo. ForceNew on change — see migration."
  type        = bool
  default     = false
}

variable "content_based_deduplication" {
  description = "FIFO only. In-place updatable."
  type        = bool
  default     = false
}

variable "fifo_throughput_scope" {
  description = "FIFO only: Topic | MessageGroup. Controls BOTH throughput and dedup scope. MUST match across regions."
  type        = string
  default     = null
}

variable "archive_days" {
  description = "FIFO only. 1-365. null disables. WARNING: moving this to null DELETES the archive."
  type        = number
  default     = null
  validation {
    condition     = var.archive_days == null || (var.archive_days >= 1 && var.archive_days <= 365)
    error_message = "archive_days must be between 1 and 365, or null."
  }
}

variable "kms_key_alias" {
  description = "Alias name, e.g. alias/messaging. Resolved per-region — this is what makes the module portable. null disables SSE."
  type        = string
  default     = null
}

variable "publisher_principals" {
  description = "IAM principal ARNs allowed sns:Publish."
  type        = list(string)
  default     = []
}

variable "subscriptions" {
  description = <<-EOT
    Map of logical name => subscription. ONE definition, both regions derive
    from it. This is the anti-drift mechanism; do not define per-region.
      protocol      : sqs | lambda | https | http | email | firehose
      endpoint      : ARN or URL. For cross-region subs, the OTHER region's ARN.
      filter_policy : object or null
      filter_scope  : MessageAttributes | MessageBody
      raw_delivery  : bool
      dlq_arn       : DLQ in the SUBSCRIPTION's region (= the topic's region)
  EOT
  type = map(object({
    protocol      = string
    endpoint      = string
    filter_policy = optional(any, null)
    filter_scope  = optional(string, "MessageAttributes")
    raw_delivery  = optional(bool, true)
    dlq_arn       = optional(string, null)
    auto_confirms = optional(bool, false)
  }))
  default = {}
}

variable "is_standby" {
  description = "True in the standby. Adds the pre-confirmation readiness outputs."
  type        = bool
  default     = false
}

variable "alarm_actions" {
  type    = list(string)
  default = []
}

variable "delivery_status_role_arn" {
  description = "IAM role for delivery status logging to CloudWatch Logs."
  type        = string
  default     = null
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

### The module

```hcl
# modules/sns-topic/main.tf

locals {
  topic_name = var.fifo ? "${var.name}.fifo" : var.name

  # Resolved per-region. The whole point of using an alias rather than an ARN.
  kms_key_id = var.kms_key_alias
}

resource "aws_sns_topic" "this" {
  name       = local.topic_name
  fifo_topic = var.fifo

  content_based_deduplication = var.fifo ? var.content_based_deduplication : null
  fifo_throughput_scope       = var.fifo ? var.fifo_throughput_scope : null

  # In-place updatable. Removing it DELETES the archive — treat as destructive.
  archive_policy = var.fifo && var.archive_days != null ? jsonencode({
    MessageRetentionPeriod = tostring(var.archive_days)
  }) : null

  kms_master_key_id = local.kms_key_id
  signature_version = 2 # SHA256. Default is 1 (SHA1) on older provider versions.

  # Failure feedback everywhere; success feedback sampled only in the standby,
  # where it is the only liveness signal an idle topic has.
  sqs_failure_feedback_role_arn    = var.delivery_status_role_arn
  lambda_failure_feedback_role_arn = var.delivery_status_role_arn
  http_failure_feedback_role_arn   = var.delivery_status_role_arn

  sqs_success_feedback_role_arn    = var.is_standby ? var.delivery_status_role_arn : null
  sqs_success_feedback_sample_rate = var.is_standby ? 5 : null

  tags = var.tags

  lifecycle {
    # An archive policy disappearing is data loss that plan renders as a
    # harmless attribute update. Make it impossible to do by accident.
    prevent_destroy = true
  }
}

data "aws_iam_policy_document" "topic" {
  count = length(var.publisher_principals) > 0 ? 1 : 0

  statement {
    sid       = "AllowNamedPublishers"
    effect    = "Allow"
    actions   = ["sns:Publish"]
    resources = [aws_sns_topic.this.arn]

    principals {
      type        = "AWS"
      identifiers = var.publisher_principals
    }
  }
}

resource "aws_sns_topic_policy" "this" {
  count  = length(var.publisher_principals) > 0 ? 1 : 0
  arn    = aws_sns_topic.this.arn
  policy = data.aws_iam_policy_document.topic[0].json
}

resource "aws_sns_topic_subscription" "this" {
  for_each = var.subscriptions

  # NOTE: inherits the module's provider, i.e. the TOPIC's region.
  # This is required, including when the endpoint is in the other region.
  topic_arn = aws_sns_topic.this.arn
  protocol  = each.value.protocol
  endpoint  = each.value.endpoint

  raw_message_delivery = contains(["sqs", "http", "https", "firehose"], each.value.protocol) ? each.value.raw_delivery : null

  filter_policy       = each.value.filter_policy == null ? null : jsonencode(each.value.filter_policy)
  filter_policy_scope = each.value.filter_policy == null ? null : each.value.filter_scope

  redrive_policy = each.value.dlq_arn == null ? null : jsonencode({
    deadLetterTargetArn = each.value.dlq_arn # same region as THIS subscription
  })

  # Only meaningful for http/https. 1 minute (the default) is too tight.
  endpoint_auto_confirms          = each.value.auto_confirms
  confirmation_timeout_in_minutes = each.value.auto_confirms ? 5 : null
}
```

```hcl
# modules/sns-topic/alarms.tf

# THE alarm. A misconfigured cross-region subscription, a revoked queue policy,
# a wrong opt-in service principal — all of them show up here and nowhere else.
resource "aws_cloudwatch_metric_alarm" "delivery_failures" {
  alarm_name  = "sns-${local.topic_name}-notifications-failed"
  namespace   = "AWS/SNS"
  metric_name = "NumberOfNotificationsFailed"
  dimensions  = { TopicName = aws_sns_topic.this.name }

  statistic           = "Sum"
  period              = 300
  evaluation_periods  = 1
  comparison_operator = "GreaterThanThreshold"
  threshold           = 0

  alarm_description  = "SNS delivery failing. Client-side errors get NO retries. See 02-services/aws-sns.md."
  alarm_actions      = var.alarm_actions
  treat_missing_data = "notBreaching"
}

# Standby canary: this topic should never be published to while primary is
# healthy. Non-zero means a misrouted producer. Mirror of the SQS depth canary.
resource "aws_cloudwatch_metric_alarm" "standby_unexpected_publish" {
  count = var.is_standby ? 1 : 0

  alarm_name  = "sns-${local.topic_name}-standby-unexpectedly-published"
  namespace   = "AWS/SNS"
  metric_name = "NumberOfMessagesPublished"
  dimensions  = { TopicName = aws_sns_topic.this.name }

  statistic           = "Sum"
  period              = 300
  evaluation_periods  = 1
  comparison_operator = "GreaterThanThreshold"
  threshold           = 0

  alarm_description  = "Standby topic received a publish while primary is active. Suspect misrouted producer."
  alarm_actions      = var.alarm_actions
  treat_missing_data = "notBreaching"
}
```

```hcl
# modules/sns-topic/outputs.tf

output "topic_arn" { value = aws_sns_topic.this.arn }
output "topic_name" { value = aws_sns_topic.this.name }

# Assert on this in CI. Any `true` is a standby that will silently drop
# messages at failover.
output "pending_confirmations" {
  description = "Subscriptions still awaiting confirmation. MUST be empty."
  value = {
    for k, s in aws_sns_topic_subscription.this : k => s.endpoint
    if s.pending_confirmation
  }
}
```

### Calling it for a pair

```hcl
# environments/prod-eu/topics.tf

locals {
  # ONE definition. Both regions derive. Drift is structurally impossible.
  topics = {
    orders = {
      fifo         = true
      archive_days = 7
      fifo_throughput_scope = "MessageGroup"
      subscriptions = {
        fulfilment = { protocol = "sqs", queue_key = "fulfilment",
                       filter_policy = { event_type = ["order.placed", "order.amended"] } }
        analytics  = { protocol = "sqs", queue_key = "analytics" }
      }
    }
    notifications = {
      fifo = false
      subscriptions = {
        dispatcher = { protocol = "sqs", queue_key = "dispatcher" }
      }
    }
  }
}

module "topics_primary" {
  source    = "../../modules/sns-topic"
  for_each  = local.topics
  providers = { aws = aws.primary }

  name                  = "${each.key}-${var.environment}"
  fifo                  = try(each.value.fifo, false)
  archive_days          = try(each.value.archive_days, null)
  fifo_throughput_scope = try(each.value.fifo_throughput_scope, null)
  kms_key_alias         = "alias/messaging"

  subscriptions = {
    for k, s in each.value.subscriptions : k => {
      protocol      = s.protocol
      endpoint      = module.queues_primary[s.queue_key].queue_arn
      filter_policy = try(s.filter_policy, null)
      dlq_arn       = module.queues_primary["sns-dlq"].queue_arn
    }
  }

  publisher_principals     = [aws_iam_role.app_primary.arn]
  delivery_status_role_arn = aws_iam_role.sns_delivery_status_primary.arn
  alarm_actions            = [aws_sns_topic.alerts_primary.arn]
  is_standby               = false
  tags                     = { Region = "primary" }
}

module "topics_standby" {
  source    = "../../modules/sns-topic"
  for_each  = local.topics
  providers = { aws = aws.standby }

  # Identical name, identical everything. That is the entire point.
  name                  = "${each.key}-${var.environment}"
  fifo                  = try(each.value.fifo, false)
  archive_days          = try(each.value.archive_days, null)
  fifo_throughput_scope = try(each.value.fifo_throughput_scope, null)
  kms_key_alias         = "alias/messaging"   # resolves to the STANDBY's key

  subscriptions = {
    for k, s in each.value.subscriptions : k => {
      protocol      = s.protocol
      endpoint      = module.queues_standby[s.queue_key].queue_arn
      filter_policy = try(s.filter_policy, null)   # SAME policy object
      dlq_arn       = module.queues_standby["sns-dlq"].queue_arn
    }
  }

  publisher_principals     = [aws_iam_role.app_standby.arn]
  delivery_status_role_arn = aws_iam_role.sns_delivery_status_standby.arn
  alarm_actions            = [aws_sns_topic.alerts_standby.arn]
  is_standby               = true
  tags                     = { Region = "standby" }
}
```

**The `for_each` over one `local.topics` map is the load-bearing line.** There
is exactly one list of topics, one set of subscriptions, one set of filter
policies, and both regions are derived from it. Add a topic and forget the
standby, and `terraform plan` adds it to both anyway. **That is the whole
design goal**, and it is the same pattern [[aws-sqs]] lands on — deliberately,
so the two modules compose. See [[module-patterns]].

### Option (b): the cross-region subscription

If you also want the primary's topic to feed the standby's queue:

```hcl
# The subscription uses the PRIMARY provider because the TOPIC is primary,
# even though the endpoint is a standby-region queue. This is the rule from
# both the AWS docs and the Terraform provider docs, and it is the #1 error.
resource "aws_sns_topic_subscription" "orders_to_standby_queue" {
  provider = aws.primary          # <-- topic's region. NOT aws.standby.

  topic_arn = module.topics_primary["orders"].topic_arn
  protocol  = "sqs"
  endpoint  = module.queues_standby["fulfilment"].queue_arn

  raw_message_delivery = true
  filter_policy        = jsonencode(local.topics.orders.subscriptions.fulfilment.filter_policy)

  # The DLQ must be in the SUBSCRIPTION's region — i.e. eu-west-1.
  redrive_policy = jsonencode({
    deadLetterTargetArn = module.queues_primary["sns-dlq"].queue_arn
  })
}

# And the queue policy, in the STANDBY region, with the region-correct principal.
data "aws_iam_policy_document" "standby_queue_allows_primary_topic" {
  statement {
    sid       = "AllowCrossRegionSNS"
    effect    = "Allow"
    actions   = ["sqs:SendMessage"]
    resources = [module.queues_standby["fulfilment"].queue_arn]

    principals {
      type = "Service"
      # THE opt-in region rule. Computed, never hardcoded.
      identifiers = [local.sns_service_principal]
    }

    condition {
      test     = "ArnLike"
      variable = "aws:SourceArn"
      values   = [module.topics_primary["orders"].topic_arn]
    }
  }
}
```

```hcl
# environments/prod-eu/locals.tf — or better, a shared module, since this is
# per-pair logic that a cookiecutter template must compute.

locals {
  # Regions launched after 2019-03-20 are opt-in. The SNS docs' own enumeration
  # is stale (it omits ca-west-1), so we keep our own list and review it.
  opt_in_regions = [
    "af-south-1", "ap-east-1", "ap-east-2", "ap-south-2", "ap-southeast-3",
    "ap-southeast-4", "ap-southeast-5", "ap-southeast-6", "ap-southeast-7",
    "ca-west-1", "eu-central-2", "eu-south-1", "eu-south-2", "il-central-1",
    "me-central-1", "me-south-1", "mx-central-1",
  ]

  topic_region = var.primary_region   # eu-west-1 / us-east-1 / ca-central-1
  queue_region = var.standby_region   # eu-west-2 / us-west-2 / ca-west-1

  topic_is_opt_in = contains(local.opt_in_regions, local.topic_region)
  queue_is_opt_in = contains(local.opt_in_regions, local.queue_region)

  # Per the AWS support matrix:
  #   default -> opt-in : sns.<queue-region>.amazonaws.com
  #   opt-in  -> default: sns.<topic-region>.amazonaws.com
  #   opt-in  -> opt-in : sns.<queue-region>.amazonaws.com
  #   default -> default: sns.amazonaws.com
  sns_service_principal = (
    local.queue_is_opt_in ? "sns.${local.queue_region}.amazonaws.com" :
    local.topic_is_opt_in ? "sns.${local.topic_region}.amazonaws.com" :
    "sns.amazonaws.com"
  )
}
```

For the three pairs this computes to:

| Pair | `sns_service_principal` |
|---|---|
| EU (`eu-west-1` → `eu-west-2`) | `sns.amazonaws.com` |
| US (`us-east-1` → `us-west-2`) | `sns.amazonaws.com` |
| **CA (`ca-central-1` → `ca-west-1`)** | **`sns.ca-west-1.amazonaws.com`** |

**Compute it; never hardcode it.** A cookiecutter template that bakes
`sns.amazonaws.com` into the queue policy works for two pairs out of three and
fails silently on the third.

### Provider version notes

- `fifo_throughput_scope` on `aws_sns_topic` and `replay_policy` on
  `aws_sns_topic_subscription` are relatively recent. **Check your pinned
  version** before copying; on older 5.x they may be absent, in which case the
  attribute has to be set out-of-band and will show as drift.
- `signature_version` defaults to 1 (SHA1) historically. Set it to 2
  explicitly.
- `archive_policy` arrived after the October 2023 feature launch; see
  [hashicorp/terraform-provider-aws#34150](https://github.com/hashicorp/terraform-provider-aws/issues/34150)
  for the request that tracked it.
- `optional()` in object type constraints requires Terraform ≥ 1.3.

---

## Migration path from single-region

Good news first: **there is nothing to replicate, so there is no
"convert this topic into a replicated topic" operation that could force
replacement.** You are purely adding resources in a new region. The risk in
this migration is entirely Terraform-shaped, and it concentrates in one place.

### 🚨 ForceNew — read this before you touch anything

Verified against the provider source
([`internal/service/sns/topic.go`](https://github.com/hashicorp/terraform-provider-aws/blob/main/internal/service/sns/topic.go),
[`topic_subscription.go`](https://github.com/hashicorp/terraform-provider-aws/blob/main/internal/service/sns/topic_subscription.go)):

| Resource | `ForceNew: true` attributes |
|---|---|
| `aws_sns_topic` | **`name`**, **`name_prefix`**, **`fifo_topic`** |
| `aws_sns_topic_subscription` | **`endpoint`**, **`protocol`**, **`topic_arn`** |

Now compose those two rows, because the composition is the danger:

> **`topic_arn` is ForceNew on the subscription. `name` is ForceNew on the
> topic. Therefore renaming a topic replaces the topic, which changes the ARN,
> which replaces every subscription on it — and every HTTP/S and email
> subscription comes back in `PendingConfirmation`.**

In a cookiecutter monorepo where topic names are *computed from templates*,
this means:

**A change to the naming template is a change to production topic identity, and
it will silently un-confirm every external subscriber.** The messages published
between the replacement and the (manual, possibly third-party) re-confirmation
are gone. There is no DLQ for them, because a pending subscription is not a
failed delivery — it is not a subscriber.

Treat naming-template PRs as production-affecting changes, with the full plan
output in the PR description, and a named reviewer. This is the same rule
[[aws-sqs]] gives for queue names, but the consequence here is strictly worse:
a replaced queue loses messages, a replaced topic loses *subscribers*.

Other ForceNew consequences:

- **`fifo_topic` can never be flipped.** A standard topic cannot become FIFO or
  vice versa. If a topic needs to become FIFO to get an archive, that is a new
  topic and a migration of every publisher and subscriber.
- **`protocol` and `endpoint` are ForceNew on the subscription.** Repointing an
  HTTPS subscription at a new URL is a destroy-and-create, so it goes back to
  pending. Partner URL changes are therefore confirmation events, not config
  events.

**What is safely in-place updatable** (useful, because it means most of the
hardening in this note is a no-downtime change):

`archive_policy`, `content_based_deduplication`, `fifo_throughput_scope`,
`kms_master_key_id`, `policy`, `delivery_policy`, `signature_version`, all the
`*_feedback_role_arn` and sample-rate attributes; and on subscriptions,
`filter_policy`, `filter_policy_scope`, `raw_message_delivery`,
`redrive_policy`, `replay_policy`, `confirmation_timeout_in_minutes`.

### Step by step

1. **Import or confirm every production topic is under Terraform management.**
   Anything clicked into existence gets `terraform import`ed *before* the
   refactor, so the refactor's plan is readable. Subscriptions too — a topic
   under management with unmanaged subscriptions is the worst of both worlds.

2. **Refactor the primary into the module using `moved` blocks.** This is the
   only step with replacement risk.

   ```hcl
   moved {
     from = aws_sns_topic.orders
     to   = module.topics_primary["orders"].aws_sns_topic.this
   }

   moved {
     from = aws_sns_topic_subscription.orders_fulfilment
     to   = module.topics_primary["orders"].aws_sns_topic_subscription.this["fulfilment"]
   }
   ```

   **`terraform plan` must show zero destroys and zero replacements.** If it
   shows a replacement of a live topic, stop. You are about to un-confirm your
   subscribers.

3. **Verify the computed name character by character** against the live topic
   name. A `.fifo` suffix, a hyphen, an environment prefix, a case difference —
   any of them replaces the topic. Diff the strings mechanically; do not eyeball
   them. **This is the single most dangerous line in the migration.**

4. **Add the standby module instantiation.** Pure creation: new topics, new
   subscriptions, new DLQs, new policies, new alarms in `eu-west-2`. No effect
   on the primary. Plan shows only adds. Apply.

5. **Confirm every standby subscription.** For SQS/Lambda in the same account
   this is automatic. For HTTP/S and email it is a deliberate, tracked task with
   a named owner per endpoint. **Do not close the migration ticket until
   `pending_confirmations` is empty for every standby topic.** Remember the
   two-day token window: if a subscription sat unconfirmed for three days it
   must be destroyed and recreated to get a fresh token.

6. **Raise the standby's publish quota** via Service Quotas to at least the
   primary's level. Lead time is days. **Start this at step 1, not step 6** —
   it is the long pole and it runs in parallel with everything else.

7. **Raise the standby's SMS spend limit** if SMS is used, and set the console
   value after approval. 24h+ lead time.

8. **Add `ArchivePolicy` to FIFO topics** in both regions. In-place, no
   disruption, no replacement.

9. **Add the `NumberOfNotificationsFailed` alarms and the readiness check.**

10. **Test.** Publish a real message to every standby topic and verify it lands
    in every standby subscriber. Verify it is encrypted with the standby's key.
    Verify a filter policy actually filters. Then clean up. **A topic that has
    never had a message through it is not a tested standby**, and every failure
    mode in the Gotchas list below is invisible until you publish.

### Zero-downtime guarantee

Steps 2–10 are either metadata operations on the primary or creations in the
standby. **No step pauses a publisher or drops a message.** The only downtime
risk in the whole migration is a botched `moved` block causing a topic
replacement, which is caught by reading the plan — which is why step 3 exists.

---

## Failover procedure

Assumes the recommended shape: option (c) everywhere, (b) on selected topics,
consumers inert in the standby per [[messaging-in-flight-data-loss]].

### Pre-flight (continuous, not at 3am)

These are the checks that must be green *before* an incident, and the whole
point of automating them is that nobody can run them during one:

- [ ] Every standby subscription is `Confirmed` (readiness Lambda, alarmed).
- [ ] Standby publish quota ≥ primary's.
- [ ] Standby SMS spend limit ≥ primary's (if SMS is in scope).
- [ ] Filter policies identical across regions (drift detector).
- [ ] `NumberOfNotificationsFailed` is zero in both regions.
- [ ] Standby DLQs are empty.

### At the moment of failover

1. **Record the exposure before you cut over.** `NumberOfMessagesPublished`,
   `NumberOfNotificationsDelivered` and `NumberOfNotificationsFailed` per topic.
   If the primary is dark you cannot — which is why these metrics must be
   shipped cross-region continuously. **You cannot reconstruct what you
   stranded once the region takes your metrics with it.** See
   [[observability-multi-region]].

2. **Repoint producers at the standby topic.** With same-name topics this is a
   *region* change in configuration, not an ARN change. Mechanically it is
   whatever [[failover-orchestration]] decides. **This is the only mandatory
   SNS step in the entire failover** — everything else was pre-provisioned.

3. **Do not touch filter policies.** Up to 15 minutes to propagate; your entire
   budget. If a filter policy is wrong, it is wrong for the duration of the
   incident and you work around it downstream.

4. **Do not create subscriptions.** Anything not already confirmed is not
   coming online inside the RTO. Note it as a degraded subscriber and move on.

5. **Fence the primary's consumers** if the primary is reachable — same
   argument, same ARC Region Switch mechanism, as [[aws-sqs]] step 2 and
   [[split-brain-and-fencing]]. Not an SNS action, but it belongs in the same
   runbook page.

6. **Verify delivery, not publication.** `NumberOfMessagesPublished` rising on
   the standby topic proves producers repointed. It does **not** prove anyone
   received anything. Check `NumberOfNotificationsDelivered` per subscription
   and the standby queue depths. **This is the step that catches a topic
   publishing into unconfirmed or misconfigured subscriptions**, and it is the
   step most likely to be skipped because the first graph looked green.

7. **Check for throttling.** If publishes start erroring, the publish quota was
   not raised. There is no fast fix — that is why step 6 of the migration
   exists. Mitigate by batching (`PublishBatch` carries 10 messages per
   request, so it buys up to 10× headroom against a *request*-rate limit — but
   note the quota is expressed in **messages** per second, so batching does not
   help against it; it helps only with API-request-rate ceilings). Realistically
   you shed load.

### Where replay fits

If the primary region is *degraded* rather than gone, and the topics are FIFO
with archives, replay is the cleanest catch-up mechanism:

```bash
aws sns set-subscription-attributes \
  --region eu-west-2 \
  --subscription-arn "$STANDBY_SUB_ARN" \
  --attribute-name ReplayPolicy \
  --attribute-value '{"PointType":"Timestamp","StartingPoint":"2026-09-21T02:00:00.000Z"}'
```

**No `EndingPoint`.** Ever, during an incident. See the trap above.

But note the limitation honestly: this replays from the **standby topic's**
archive, which only has messages if the standby topic was being published to.
In a pure option (c) design it was not. **Replay is a failback and
partial-degradation tool, not a failover tool.** Do not let it into the
failover runbook as though it recovers the primary's traffic — it does not.

---

## Failback

Failback is where the option you chose at build time presents its bill.

**Option (c) failback is trivial.** Repoint producers back to `eu-west-1`.
`eu-west-2`'s topics go quiet. The standby canary alarm
(`NumberOfMessagesPublished > 0` on a standby topic) tells you if some producer
never got repointed — **that alarm is the single most useful thing in a
failback**, because a forgotten producer publishing into a topic whose
consumers have stopped is a silent, ongoing data loss that can run for weeks.

**Option (b) failback is the asymmetry problem.** The cross-region subscription
you built runs primary → standby. After failover you are publishing to
`eu-west-2`, which has no subscription pointing back to `eu-west-1`. Two
sub-choices, and this must be decided at build time:

- **Symmetric** — both topics carry a cross-region subscription to the other
  region's queue. Failback is free, the design is region-agnostic, and "which
  region is primary" stops being encoded in infrastructure. Costs double
  cross-region transfer in steady state, and every message lands in four places.
- **Asymmetric, reconfigured at failback** — add standby → primary as a
  Terraform change during failback. Cheaper in steady state; puts a
  `terraform apply` in the failback critical path.

**Recommendation: symmetric**, for the same reason [[aws-sqs]] reaches it — the
transfer cost is small next to the operational cost of an apply during an
incident, and boring failback is the goal.

**One SNS-specific loop hazard that symmetric designs create:** if anyone ever
subscribes topic A to topic B *and* topic B to topic A, you have an infinite
message loop with no built-in cycle detection. SNS-to-SNS subscriptions are not
a thing SNS supports directly (there is no `sns` protocol), but the equivalent
is easy to build accidentally via a Lambda that republishes. **Put a hop-count
or origin-region message attribute on every message and have republishers drop
their own region's messages.** Cheap insurance; see
[[lessons-and-antipatterns]].

**FIFO failback has an extra step.** If the primary's subscribers need to catch
up on what happened while `eu-west-2` was serving, and the standby topic has an
archive, replay into the primary's subscriptions from the failover timestamp —
open-ended, as always. That is the clean mechanism and it is much better than
re-publishing, because replayed messages retain their original `MessageId` and
`Timestamp` and carry the `Replayed` flag, so consumers can tell.

**And the failback-specific readiness check:** after failback, re-run the
confirmation readiness check on `eu-west-2`. A subscription that was confirmed
six months ago is still confirmed, but a subscription that was *recreated*
during the incident (because someone repointed an endpoint) may not be. The
standby must return to a fully-confirmed state or the next failover inherits
the gap.

---

## Gotchas

1. **There is no cross-region SNS topic replication.** The standby topic is a
   second resource with a different ARN. Every ARN reference must be
   parameterised.

2. **A `PendingConfirmation` subscription is invisible.** No error, no metric,
   no DLQ, no alarm. It is the highest-severity silent failure in this note.
   The only detection is an explicit readiness check.

3. **Confirmation tokens expire in two days.** A subscription created by
   Terraform and never confirmed becomes permanently unconfirmable and cannot
   be deleted through the API or console. Terraform sees no drift.

4. **`topic_arn` is ForceNew on subscriptions and `name` is ForceNew on topics.**
   A topic rename cascades into replacing every subscription, which
   un-confirms every external subscriber. In a templated monorepo, a naming
   template change is a production incident waiting for an apply.

5. **`fifo_topic` is ForceNew.** Standard ⇄ FIFO is never an in-place change.

6. **The SNS docs' own opt-in region list is stale and omits `ca-west-1`.**
   Follow the "launched after 2019-03-20" rule and the General Reference table.
   Getting this wrong means `sns.amazonaws.com` in a CA-pair queue policy and a
   cross-region subscription that confirms, publishes, retries, and never
   delivers.

7. **Client-side delivery errors get zero retries.** The 23-day safety net only
   covers server-side errors. A wrong service principal, a deleted queue or a
   revoked policy fails permanently on the first attempt. Alarm on
   `NumberOfNotificationsFailed`.

8. **The standby's default publish quota can be 30× smaller than the
   primary's** (`eu-west-1` 9,000/s vs `eu-west-2` 300/s). Raise it before you
   need it; it is a Support workflow with days of lead time.

9. **Filter policy changes take up to 15 minutes to propagate** — the whole RTO.
   Never change one during a failover, and never design a failover switch out of
   one.

10. **A filter policy on a FIFO subscription downgrades the guarantee from
    exactly-once to at-most-once.** AWS says so explicitly. Nothing warns you.

11. **FIFO dedup is scoped to the topic (or message group) and lasts 5
    minutes.** It provides zero protection across a 15-minute failover, and
    dual-publishing gives each region an independent dedup context.

12. **FIFO topics cannot have HTTP/S, email, SMS or mobile subscribers** —
    `Subscribe` errors. Fan out via SQS. Silver lining: FIFO topics are immune
    to the confirmation trap.

13. **Setting `archive_policy` to `{}` permanently deletes the archive**, and
    `terraform plan` renders it as an ordinary attribute update, not a destroy.
    Up to 365 days of messages, gone in a one-line diff.

14. **You cannot delete a topic with an active archive policy.** Teardown
    pipelines for ephemeral environments will break the first time someone adds
    one.

15. **The archive is regional and does not replicate.** It is a recovery point
    for standby outages and degradations, not for the loss of the region that
    hosts it.

16. **A disabled or deleted KMS key makes the archive unreadable.** The key must
    outlive the retention period — a 365-day archive and a 30-day key deletion
    window are incompatible. See [[aws-kms]].

17. **SSE encrypts the message body only** — not message attributes, which is
    where filter-policy-driven designs put their routing metadata, and
    sometimes more.

18. **A cross-region subscription's DLQ lives in the topic's region**, i.e. the
    region you are trying to escape. Messages that failed to reach the standby
    are parked in the primary.

19. **An SNS-subscription DLQ is not a valid `StartMessageMoveTask` source.**
    SQS redrive explicitly does not support DLQs fed by SNS. Recovery is a
    bespoke script, and you must decide in advance whether it re-publishes to
    the topic (duplicating for all subscribers) or writes straight to the
    destination queue.

20. **An encrypted SNS-subscription DLQ requires a customer-managed key** whose
    policy grants the SNS service principal. SSE-SQS will not work — a
    deliberate exception to [[aws-sqs]]'s "prefer SSE-SQS" advice.

21. **A bounded replay (`EndingPoint` set) pauses the subscription
    permanently.** New messages stop arriving until a second, open-ended
    `ReplayPolicy` is applied. Another silent inert subscription.

22. **The SMS spend limit is $1/month by default and is per-region**, with a
    24h+ Support lead time and a mandatory manual console step afterwards. SMS
    cannot be fixed inside a 15-minute RTO.

23. **The SMS opt-out list is regional.** Failing over can mean messaging
    people who opted out. That is a compliance exposure, not just a bug.

24. **The SMS sandbox is per-region.** An account out of the sandbox in the
    primary may still be in it in the standby, silently dropping to unverified
    numbers.

25. **Attribute-based filter policies require double-escaping `"` and `\`.**
    Miss it and the policy silently never matches.

26. **A cross-region subscription must be created with the *topic's* provider**,
    not the endpoint's. The most common Terraform error in this pattern.

27. **`ca-west-1` has no published control-plane TPS quota.** It is absent from
    every tier of AWS's "other API throttling" table. Unknown, not zero —
    confirm with Support if the CA pair proceeds.

28. **A topic you never published to is not a standby.** Every failure above is
    invisible on an idle topic. Put "publish one message through every standby
    topic and verify every subscriber received it" in
    [[dr-testing-and-gamedays]] as a recurring check.

---

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Architecture (estate default) | Cross-region subscribers from one primary topic | Independent topics per region, publisher chooses | **B (option c).** A is not a failover design — the topic is in the dead region. Add A selectively on top. |
| Architecture (crown jewels) | Dual-publish from the app | Persist to a replicated store before acknowledging | **B.** [[messaging-in-flight-data-loss]]'s outbox argument applies unchanged and is cheaper than dual-publish. |
| Topic naming across regions | Identical names | Region-suffixed | **Identical**, with per-region IAM scoping to prevent cross-wiring. Same argument and same exception as [[aws-sqs]]. |
| Standby subscription confirmation | Confirm at failover | **Pre-confirm and verify continuously** | **Pre-confirm.** There is no other option that meets 15 minutes. |
| Third-party HTTPS subscribers | Subscribe them to both regions' topics | Subscribe a queue you own; relay to the partner | **B.** Removes a third party from the failover critical path and converts the problem into one [[aws-sqs]] already solved. |
| Email subscriptions for alerting | Keep them | Replace with a programmatically-confirmable integration | **Replace.** A human-confirmed subscription cannot be part of a tested standby. |
| Fencing the standby's subscribers | Filter policy that drops everything | Disabled consumers / ESMs | **Disabled consumers.** A filter policy takes 15 minutes to change — it is not a switch. |
| FIFO cross-region | Cross-region SQS FIFO subscriber | Independent FIFO topics per region | **Independent.** Cross-region works but degrades per-message-group throughput by an unpublished amount. |
| `ArchivePolicy` on FIFO topics | Skip it | Enable at 7+ days in both regions | **Enable.** In-place, cheap, converts a class of losses into replays. Mind the key lifetime. |
| KMS for topics | Multi-region key | Per-region key behind an identical alias | **Per-region + alias.** Messages never cross as ciphertext, so MRKs buy nothing here. Useful negative finding for [[aws-kms]]. |
| SSE at all | SSE-KMS everywhere | Only where classification requires | **Only where required**, *except* SNS-subscription DLQs, which need a CMK if encrypted at all. |
| SNS SMS | Fail it over with everything else | Pin it to one region, or move to End User Messaging | **Pin or move.** Every SMS lever has multi-day lead time; it cannot participate in a 15-minute failover. |
| Standby publish quota | Raise at failover | **Raise now** | **Now.** Soft quota, free while unused, days of lead time. Highest-value single action in this note. |
| Option (b) directionality | Asymmetric, reconfigure at failback | Symmetric both ways | **Symmetric.** Keeps `terraform apply` out of the failback critical path. |

---

## Cost

**Idle standby topics are free.** SNS bills per request. A topic nobody
publishes to, with subscriptions nobody delivers to, generates no requests and
no charge. There is no per-topic, per-subscription or per-month fee.

**I was unable to get AWS's pricing page to render its rate tables** — the
numbers are loaded client-side and did not come through. The same limitation
affected [[aws-sqs]]. **Rather than quote third-party figures, this note states
only what AWS's page does say in prose, and flags the rest for verification
before it enters [[cost-model]]:**

- *"Amazon SNS does not charge for per-message notification delivery when
  delivering messages to Amazon SQS and AWS Lambda, but does charge for the
  amount of data transferred."*
- Message archiving and replay is billed *"based on the amount of data you
  store and the length of time the data is stored for"*, minimum 1 day.
- Delivery charges vary by endpoint type; email, SMS, HTTP/S and mobile push
  are each priced differently.

**⚠️ Verify the per-million-request, per-GB-archived and per-GB-replayed rates
at [aws.amazon.com/sns/pricing](https://aws.amazon.com/sns/pricing/) before
building a cost model.** Do not carry unverified numbers into a budget.

### Where the money actually is, structurally

| Item | When | Magnitude |
|---|---|---|
| Idle standby topics + subscriptions | Always | **£0** |
| Idle standby DLQs | Always | **£0** (SQS bills requests) |
| Customer-managed KMS key per region | SSE enabled | ~$1/month per key, plus KMS request charges driven by **publishing principals**, not volume |
| **Option (b) cross-region data transfer** | Option (b) only | Per GB, **charged to the sending region** — so it lands in the primary's cost centre, not the DR one. Easy to misattribute; flag in [[cost-model]]. |
| **Option (b) double processing** | Option (b) with live standby consumers | **Duplicates the compute cost of the workload.** Usually the largest line and usually forgotten. |
| **Option (a) double publishing** | Option (a) only | 2× publish requests *and* 2× downstream delivery |
| FIFO archive storage | `ArchivePolicy` set | GB-stored × days. Linear in retention — this is the lever. |
| Replay | On use | GB replayed. Incident-only, so negligible in steady state. |
| Delivery status logging | If enabled | CloudWatch Logs ingestion. **Sample success feedback; 100% on a busy primary is a real bill.** |
| CloudWatch alarms + readiness Lambda | Always | Pennies |

### The levers, in order of size

1. **Do not run standby consumers** unless [[messaging-in-flight-data-loss]]
   says to. Duplicated compute dwarfs everything else here.
2. **Restrict option (b) to the handful of topics that need it.** Applied
   estate-wide it converts a messaging bill into a data-transfer bill.
3. **Set archive retention deliberately.** 7 days costs roughly 1/52 of 365
   days. Choose the number for a reason.
4. **Sample delivery-status success feedback**; keep failure feedback at 100%
   because it is low-volume and high-value.
5. **Skip SSE-KMS where classification allows.** Note the KMS cost scales with
   the number of publishing principals, so a shared topic with many publishers
   is the expensive shape — worth knowing before someone consolidates topics
   "to simplify".

**Compared to [[aws-eks]]'s standby capacity, SNS is a rounding error.** That
is another argument for pre-provisioning generously: there is no cost reason
not to create every topic, every subscription and every DLQ in the standby.

---

## Open questions

1. **Which topics have HTTP/S or email subscribers?** That list *is* the list
   of topics at risk from the confirmation trap. Everything else auto-confirms.
   It is a one-hour `ListSubscriptionsByTopic` sweep and it should happen before
   any of this design is committed to. **This is the single most valuable
   unanswered question in the note.**
2. **Are any of those endpoints owned by third parties?** If yes, the standby
   confirmation is an organisational task with a lead time, not an engineering
   one. See [[third-party-saas-dependencies]].
3. **What is the estate's actual peak publish rate per region?** It decides how
   urgent the quota increase is. If `eu-west-1` peaks at 50 msg/s, the 300/s
   standby default is fine and this note's loudest finding is a non-issue for
   you. If it peaks at 2,000, the EU failover is broken today.
4. **Is SMS on any customer-facing path?** If yes, the $1 standby spend limit
   is an open production risk right now, not a DR one, and the 24h lead time
   makes it a prerequisite.
5. **Does the CA pair proceed?** SNS passes the parity check (including FIFO and
   archive/replay), which is a point in Calgary's favour after Cognito,
   OpenSearch and Managed Grafana all failed. Feed into
   [[region-pair-selection]]. If it does proceed, confirm `ca-west-1`'s
   control-plane TPS quota with Support, since AWS does not publish it.
6. **Do any topics rely on FIFO's exactly-once guarantee downstream, *and* have
   filter policies?** Those subscriptions are at-most-once today and the owning
   team probably does not know.
7. **Is the 10,000-filter-policies-per-account quota per region?** Almost
   certainly yes, but not stated. Only matters if the estate is near the limit.
8. **Does any consumer already handle the `Replayed` message attribute?** If
   replay is to be a real recovery tool, consumers need to distinguish replayed
   from live traffic. Today they almost certainly do not.
9. **Which queues should be fronted by a topic purely for the 23-day retry
   window?** This is a cheap, high-value change (one-line producer edit) for any
   queue whose messages are not re-derivable, and nobody has made the list.
10. **Is the RTO clock from incident start or from decision-to-fail-over?**
    Flagged vault-wide in `CLAUDE.md`. It matters less for SNS than for compute,
    because SNS's steps are all zero-or-hours with nothing in between — but it
    changes how much slack the confirmation readiness process is allowed to
    consume.

## Sources

- [Sending Amazon SNS messages to an Amazon SQS queue or AWS Lambda function in a different Region](https://docs.aws.amazon.com/sns/latest/dg/sns-cross-region-delivery.html)
  — the authoritative statement that cross-region delivery is supported for SQS
  and Lambda **only**, the requirement to run `Subscribe` in the topic's region,
  and the opt-in-region service principal matrix. Its own opt-in region list is
  stale and omits `ca-west-1`.
- [Enable or disable AWS Regions — AWS General Reference](https://docs.aws.amazon.com/general/latest/gr/rande-manage.html)
  — the current opt-in/default region tables. Confirms `ca-west-1` is opt-in and
  `ca-central-1` is default-enabled.
- [Amazon SNS endpoints and quotas — AWS General Reference](https://docs.aws.amazon.com/general/latest/gr/sns.html)
  — SNS presence in `ca-west-1` incl. a FIPS endpoint; "both standard and FIFO
  topics"; the per-region publish throttling table; pending-subscription quota;
  email 10/s hard limit; archive/replay availability in all commercial regions.
- [Subscribe — Amazon SNS API Reference](https://docs.aws.amazon.com/sns/latest/api/API_Subscribe.html)
  — which protocols require confirmation, the cross-*account* (not cross-region)
  trigger, the two-day token validity, and the `"pending confirmation"` return
  value.
- [Amazon SNS message delivery retries](https://docs.aws.amazon.com/sns/latest/dg/sns-message-delivery-retries.html)
  — 100,015 retries over 23 days for SQS/Lambda; 50 attempts over 6 hours for
  SMTP/SMS/mobile push; HTTP/S custom delivery policies and the 3,600s cap.
- [Amazon SNS dead-letter queues](https://docs.aws.amazon.com/sns/latest/dg/sns-dead-letter-queues.html)
  — *"The Amazon SNS subscription and Amazon SQS queue must be under the same
  AWS account and Region"*; client vs server error handling; FIFO topics need
  FIFO DLQs.
- [Amazon SNS message archiving and replay for FIFO topics](https://docs.aws.amazon.com/sns/latest/dg/fifo-message-archiving-replay.html)
  — 365-day archive, replay via subscription `ReplayPolicy`, and AWS's own
  framing of it as a way of *"synchronizing applications across regions"*.
- [Securing Amazon SNS data with server-side encryption](https://docs.aws.amazon.com/sns/latest/dg/sns-server-side-encryption.html)
  — symmetric KMS keys only; `alias/aws/sns` is per-account-per-region; SSE
  encrypts the body only, not metadata; backlogged messages are not retroactively
  encrypted.
- [aws_sns_topic_subscription — Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/sns_topic_subscription)
  — *"the `aws_sns_topic_subscription` must use an AWS provider that is in the
  same region as the SNS topic"*; `endpoint_auto_confirms`;
  `confirmation_timeout_in_minutes` (default 1); unconfirmed subscriptions
  cannot be deleted by Terraform.
- [terraform-provider-aws `internal/service/sns/topic.go`](https://github.com/hashicorp/terraform-provider-aws/blob/main/internal/service/sns/topic.go)
  — provider source. `ForceNew: true` on `name`, `name_prefix`, `fifo_topic`;
  `archive_policy`, `fifo_throughput_scope` and `content_based_deduplication`
  are in-place updatable. Read from source because the website docs do not mark
  ForceNew.
- [terraform-provider-aws `internal/service/sns/topic_subscription.go`](https://github.com/hashicorp/terraform-provider-aws/blob/main/internal/service/sns/topic_subscription.go)
  — `ForceNew: true` on `endpoint`, `protocol`, `topic_arn` (the cascade that
  makes a topic rename un-confirm every subscriber); the 2-minute default
  confirmation wait; the provider skips waiting for `email` and for `http*`
  without `endpoint_auto_confirms`.
- [Amazon SNS message deduplication for FIFO topics](https://docs.aws.amazon.com/sns/latest/dg/fifo-message-dedup.html)
  — dedup scope follows `FifoThroughputScope` (topic or message group), the
  five-minute interval, and the statement that **filter policies downgrade FIFO
  from exactly-once to at-most-once**.
- [Amazon SNS message delivery for FIFO topics](https://docs.aws.amazon.com/sns/latest/dg/fifo-message-delivery.html)
  — FIFO topics cannot deliver to customer-managed endpoints (email, SMS,
  mobile, HTTP/S); `Subscribe` errors if you try.
- [Amazon SNS message archiving for FIFO topic owners](https://docs.aws.amazon.com/sns/latest/dg/message-archiving-and-replay-topic-owner.html)
  — `ArchivePolicy` JSON, 1–365 days, A2A-FIFO-only, the archive CloudWatch
  metrics, `BeginningArchiveTime`, the KMS decrypt grant, and the two landmines:
  a topic with an active archive policy cannot be deleted, and deactivating the
  policy deletes every archived message.
- [Amazon SNS message replay for FIFO topic subscribers](https://docs.aws.amazon.com/sns/latest/dg/message-archiving-and-replay-subscriber.html)
  — `ReplayPolicy` shape, `ReplayStatus` values, the `Replayed` attribute, and
  the critical statement that specifying an `EndingPoint` *"effectively pauses
  the subscription."*
- [Amazon SNS subscription filter policies](https://docs.aws.amazon.com/sns/latest/dg/sns-subscription-filter-policies.html)
  — *"Additions or changes to a subscription filter policy require up to 15
  minutes to fully take effect."* The basis for the "never change a filter
  policy during a failover" rule.
- [Filter policy constraints in Amazon SNS](https://docs.aws.amazon.com/sns/latest/dg/subscription-filter-policy-constraints.html)
  — 150 combinations, 5 keys, 256 KB, 200/topic, 10,000/account, the
  double-escaping requirement, and the wildcard complexity budget.
- [Managing Amazon SNS encryption keys and costs](https://docs.aws.amazon.com/sns/latest/dg/sns-key-management.html)
  — the publisher needs `kms:GenerateDataKey*` + `kms:Decrypt`; the KMS cost
  formula `R = B / D * (2 * P)` showing cost scales with publishing principals,
  not volume; the 5-minute data key reuse period; the requirement to name full
  regional key ARNs in IAM policies.
- [Requesting increases to your monthly Amazon SNS SMS spending quota](https://docs.aws.amazon.com/sns/latest/dg/channels-sms-awssupport-spend-threshold.html)
  — the $1.00/month default, the per-region Support case, the 24-hour initial
  response, and the mandatory manual console step afterwards.
- [aws-samples/sample-sns-sqs-multi-region](https://github.com/aws-samples/sample-sns-sqs-multi-region)
  — AWS's own reference implementation of cross-region SNS→SQS fan-out across
  `us-east-1`/`us-west-2`, including a regional failover walkthrough. Note it
  assumes manual producer failover.
- [Amazon SNS now supports in-place message archiving and replay for FIFO topics](https://aws.amazon.com/about-aws/whats-new/2023/10/amazon-sns-in-place-message-archiving-replay-fifo-topics)
  — the October 2023 launch announcement; dates the feature.
- [hashicorp/terraform-provider-aws #34150 — SNS FIFO topic message archiving](https://github.com/hashicorp/terraform-provider-aws/issues/34150)
  — the provider issue tracking `archive_policy` support; useful if the estate
  is on an older provider pin.

### Searched for and did not find

- **Any AWS statement that SNS Firehose subscriptions work cross-region.** The
  cross-region delivery page names SQS and Lambda only, and Firehose
  subscriptions additionally need a `SubscriptionRoleArn`. No public source
  either way. Treat as same-region-only until tested.
- **An AWS documentation page stating how long a `PendingConfirmation`
  subscription survives before AWS removes it.** Third-party sources say two or
  three days and disagree with each other. Only the *token* validity (two days)
  is documented, in the `Subscribe` API reference.
- **A published AWS number or third-party benchmark for cross-region FIFO
  per-message-group throughput.** AWS states the degradation exists
  (*"reduced throughput within a message group for cross Regional
  deliveries"*) but publishes no figure. The arithmetic in the FIFO section is
  reasoning from inter-region RTT, explicitly **not** a measurement. Benchmark
  it before designing on it.
- **A control-plane TPS quota for `ca-west-1`.** Calgary is absent from every
  tier of AWS's "other API throttling" table, as are several other recent
  regions. Unknown rather than zero.
- **SNS's own pricing rate tables.** The pricing page renders its numbers
  client-side and they did not come through. The Cost section deliberately
  quotes only the page's prose and flags the rates for verification rather than
  substituting third-party figures.
- **A published postmortem of an organisation losing messages because a standby
  SNS subscription was left unconfirmed.** No public example found. The failure
  mode is inferred from the documented mechanics (a pending subscription emits
  no metric and receives nothing), not from a case study — which is precisely
  why it is worth writing down: it is the kind of failure that produces no
  artefact to write up afterwards.
- **An explicit statement of whether the 10,000-filter-policies-per-account
  quota is per-region.** Listed among regional quotas, so almost certainly yes,
  but not stated.
