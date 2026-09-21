---
title: Amazon CloudFront — Multi-Region
service: cloudfront
tags: [service, multi-region, cloudfront, cdn, edge, failover, origin-groups]
status: researched
replication: global — nothing to replicate; the distribution is already everywhere
rpo_achievable: N/A — cache and configuration, not data
rto_achievable: "seconds (origin group failover, no DNS change) for GET/HEAD/OPTIONS; seconds (CloudFront Function + KeyValueStore origin selection) for all methods; 5–15 min if the plan is to edit the distribution"
meets_targets: conditional — yes as a failover *mechanism*, no if your runbook edits the distribution
updated: 2026-09-21
---

# Amazon CloudFront — Multi-Region

## TL;DR

- **CloudFront is global by nature. It is not something you mirror.** There is no "CloudFront in `eu-west-1`". One distribution is served from every edge location on earth. **Do not create a second distribution for the standby region** — that is the single most common wrong instinct on this note, and it costs you a second certificate, a second WAF ACL, a second cache, and a DNS layer you did not need. The question this note answers is not *"how do I make CloudFront multi-region"* but ***"how do I use CloudFront as the failover mechanism"***.
- **Origin groups are a genuinely strong RTO option and the best one in this vault for the read path.** Failover happens **at the edge, per request, in seconds**, with **no `ChangeResourceRecordSets` call, no Route 53 control plane, no TTL, and no client DNS cache problem at all**. Compare [[aws-route53]], where ~90 seconds of the 900-second budget is DNS and the JVM cache can silently extend that to an hour. Origin failover has none of that. See [[#Origin groups and origin failover]].
- **But origin failover only applies to `GET`, `HEAD` and `OPTIONS`.** AWS states this verbatim and there is no setting to change it. For a write-heavy API, **`POST`/`PUT`/`PATCH`/`DELETE` are returned to the client as errors and are never retried against the standby.** This one sentence decides whether origin groups are your failover mechanism or just a nice safety net on the read path. Read [[#The GET/HEAD/OPTIONS limitation — the decision point]] before anything else.
- **There is a way around it, and it is new: a CloudFront Function at *viewer request* that calls `cf.selectRequestOriginById()` or `cf.createRequestOriginGroup()`, reading an active-region flag out of CloudFront KeyValueStore.** That is *origin selection*, not *origin failover*, so it applies to **every** HTTP method, and a KVS key update propagates to all edges "in a few seconds" **without a distribution deployment**. This is the highest-value finding in this note. See [[#Edge functions as the real failover switch]].
- **The thing that will bite is not theoretical — it was measured during a real CloudFront incident.** Two things, both about the seconds after the switch. (1) **CloudFront does not remember that it failed over** — "CloudFront routes all incoming requests to the primary origin, even when a previous request failed over" — so *every* cache-miss request pays the full primary timeout (**up to 30 s by default**) before reaching the standby. (2) **Cold cache thundering herd**: the standby takes 100% of origin traffic with an empty edge cache, while it is scaling out and its database was just promoted. See [[#The cold-cache thundering herd]].

## Does this service cross regions at all?

**No, because it is already everywhere.** CloudFront sits in AWS's own *global services* fault-isolation category alongside Route 53, IAM and Global Accelerator — see the [Fault Isolation Boundaries whitepaper](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/appendix-b---edge-network-global-service-guidance.html), already quoted in [[aws-acm]] and [[aws-regional-outages]].

| Thing | Scope |
|---|---|
| Distribution (`aws_cloudfront_distribution`) | **Global.** No region attribute. Served from every POP. |
| Edge caches / regional edge caches | **Global**, AWS-operated, not yours to place. |
| Origin Shield | **Regional — you choose the AWS Region.** The one genuinely regional knob. See [[#The cold-cache thundering herd]]. |
| Cache / origin request / response headers policies, key groups, OAC, KeyValueStore, functions | **Global** CloudFront objects. |
| **CloudFront control plane** (`cloudfront.amazonaws.com`) | **`us-east-1`.** Every `UpdateDistribution`, every `CreateInvalidation`. |
| **Viewer certificate** | **`us-east-1` ACM only.** Cross-ref [[aws-acm]]. |
| **WAF web ACL** | **`CLOUDFRONT` scope, created in `us-east-1`.** Cross-ref [[aws-acm]] and [[aws-waf-shield]]. |
| **Origins** | **Regional.** This is the only part of the picture that has a region, and therefore the only part you mirror. |
| Lambda@Edge function (`aws_lambda_function`) | Authored in **`us-east-1`**, replicated by AWS to edge regions. |

Three practical consequences for the Terraform estate:

1. **There is exactly one distribution per public hostname, per environment — for all time.** It already survives the loss of `eu-west-1`. What does not survive is its *origin*. Mirroring work belongs to [[aws-alb-nlb]], [[aws-s3]] and [[aws-api-gateway]]; CloudFront's job is to choose between the two copies.
2. **The edge stack cannot be a per-region module.** It is a single global stack that consumes outputs from both regional stacks — structurally identical to the recommendation in [[aws-route53]] for the DNS stack, and probably the same stack. See [[#Terraform implementation]].
3. **The distribution is a shared fate you cannot mirror away.** On **16 July 2026** a CloudFront **VPC Origins** failure served global 5xx for **several hours** to every customer using that feature, and *no regional failover helped*, because the fault was in a global configuration-distribution system. **Reported durations vary between ~3 h 33 m and ~8 h depending on the source, and no durable first-party AWS post-event summary page was found** — see [[#Case study — the 16 July 2026 CloudFront VPC Origins event]] for exactly what is and is not verified.

> [!important] Say this to the team once, clearly
> "Replicate the CloudFront distribution to eu-west-2" is not a ticket. There is nothing to replicate. If such a ticket exists, close it and replace it with "add the standby ALB as a second origin and decide how we choose between them."

## Origin groups and origin failover

An **origin group** is a CloudFront object containing exactly **two** origins — a
primary and a secondary — plus a list of HTTP status codes that constitute
"failure". You then point a *cache behavior* at the origin group instead of at a
single origin. That is the whole feature. There is no third origin, no weighting,
no priority list beyond primary/secondary.

From [Optimize high availability with CloudFront origin failover](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/high_availability_origin_failover.html), verbatim, this is the full decision tree CloudFront runs per viewer request:

> + When there's a cache hit, CloudFront returns the requested object.
> + When there's a cache miss, CloudFront routes the request to the primary origin in the origin group.
> + When the primary origin returns a status code that is not configured for failover, such as an HTTP 2xx or 3xx status code, CloudFront serves the requested object to the viewer.
> + When any of the following occur:
>   + The primary origin returns an HTTP status code that you've configured for failover
>   + CloudFront fails to connect to the primary origin (when 503 is set as a failover code)
>   + The response from the primary origin takes too long (times out) (when 504 is set as a failover code)
>
>   Then CloudFront routes the request to the secondary origin in the origin group.

Read the second and third bullets of that last group carefully, because they are
the most commonly-missed sentence in the whole feature:

> [!danger] A connection failure is not automatically a failover
> **"CloudFront fails to connect to the primary origin (when 503 is set as a failover code)"** and **"The response from the primary origin takes too long (times out) (when 504 is set as a failover code)"**.
>
> If your origin group's failover status code list does **not** include `503`,
> then an origin that refuses TCP connections entirely — the exact signature of a
> dead region — **does not trigger failover**. If it does not include `504`, an
> origin that accepts the connection and then hangs — the signature of a brownout
> or a database that will not answer — **does not trigger failover either**.
>
> A regional outage is *precisely* the case of "can't connect" and "connects then
> hangs". **An origin group configured with only `500` and `502` will sit there
> and fail during the event it was built for.** Always include `500, 502, 503, 504`
> at minimum.

### Failover criteria — the complete list

From the same page, the status codes you may choose:

> You can choose any combination of the following status codes: 400, 403, 404, 416, 429, 500, 502, 503, or 504.

| Code | Include in the failover set? | Why |
|---|---|---|
| **500, 502, 503, 504** | **Yes, always** | The regional-failure codes. `503` covers "cannot connect", `504` covers "timed out". Without these two the feature does nothing during an outage. |
| 429 | Probably | Throttled primary. Sending the overflow to a warm standby is usually what you want — but see the thundering-herd warning below, you may just move the overload. |
| 400, 403, 404, 416 | **No** | These are *application* answers, not region failures. A 404 means the object is not there; it will not be in the standby either (unless S3 replication is lagging, in which case you are papering over a replication bug). **A 403 in the failover list is actively dangerous with S3 + OAC** — a permissions misconfiguration on the primary bucket silently doubles your origin traffic into the standby and hides the real fault. |

`400`/`403`/`404` in the failover list is the classic "we turned on everything"
mistake. It converts every bad request in the estate into two origin requests.

### Timeouts and attempts — where the seconds actually go

From [Origin settings → Connection attempts](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesOrigin.html), verbatim:

> You can set the number of times that CloudFront attempts to connect to the origin. You can specify 1, 2, or 3 as the number of attempts. The default number (if you don't specify otherwise) is 3.
>
> Use this setting together with **Connection timeout** to specify how long CloudFront waits before attempting to connect to the secondary origin or returning an error response to the viewer. **By default, CloudFront waits as long as 30 seconds (3 attempts of 10 seconds each) before attempting to connect to the secondary origin** or returning an error response.

And:

> If all the connection attempts fail and the origin is part of an origin group, CloudFront attempts to connect to the secondary origin. **If the specified number of connection attempts to the secondary origin fail, then CloudFront returns an error response to the viewer.**

So the **defaults for a hard-down primary are 30 s to the primary, then up to
another 30 s to the secondary, before the viewer gets an error.** Worst case, one
minute of edge-side patience per request.

The read path is worse, and this is the part people miss. From the same page,
**Response timeout** (default **30 s**, settable 1–120 s):

> `GET` and `HEAD` requests – If the origin doesn't respond or stops responding within the duration of the response timeout, CloudFront drops the connection. **CloudFront tries again to connect according to the value of Connection attempts.**
>
> `DELETE`, `OPTIONS`, `PATCH`, `PUT`, and `POST` requests – If the origin doesn't respond for the duration of the read timeout, CloudFront drops the connection and **doesn't try again** to contact the origin. The client can resubmit the request if necessary.

> [!warning] The 90-second `GET`
> Combine those two clauses. A `GET` against a primary that **accepts the TCP
> connection and then hangs** (the classic database-brownout shape — the ALB is
> up, targets are up, the query never returns) burns **response timeout ×
> connection attempts = 30 s × 3 = 90 seconds** before CloudFront even *starts*
> talking to the secondary. This is inference from the two verbatim clauses above
> rather than a single sentence AWS states outright, but it is the plain reading
> and it matches the shape of the retry description.
>
> Ninety seconds is **10% of the entire 900-second RTO budget**, spent by a single
> request, at every edge location, in parallel, for every cache-miss URL.
>
> **Mitigation: set `origin_read_timeout` on the primary to something aligned with
> your actual p99 origin latency — 5–10 s for a JSON API — and `connection_attempts = 1`
> or `2` on the primary.** Do not leave the defaults on a failover origin group.
> Note the constraint that the *secondary* wants the opposite: generous timeouts,
> because it is cold and slow in the first minutes after promotion.

The newer **Response completion timeout** setting gives you a hard ceiling on the
whole exchange, which is the cleanest way to bound this:

> Unlike **Response timeout**, which is the wait time for *individual* response packets, **Response completion timeout** is the *maximum* allowed amount of time that CloudFront waits for the response to complete... This maximum timeout includes what you specified for other timeout settings and the number of **Connection attempts** for each retry.

> [!note] Check provider support before writing HCL
> `response_completion_timeout` is a recent CloudFront origin attribute. Confirm
> the `hashicorp/aws` provider version in use exposes it before putting it in the
> module — see [[#Open questions]].

### The killer: CloudFront does not remember

Verbatim, from the origin failover page:

> **CloudFront routes all incoming requests to the primary origin, even when a previous request failed over to the secondary origin. CloudFront only sends requests to the secondary origin after a request to the primary origin fails.**

This is the single most important operational sentence about origin groups and it
is easy to read past. **There is no circuit breaker.** There is no "the primary is
marked down for 30 seconds" state. Origin failover is **stateless and
per-request**.

Consequences, in order of how much they hurt:

1. **Every cache-miss request pays the full primary timeout, forever.** Not the
   first one. Every one. With defaults that is 30 s (connect failure) or up to
   90 s (hang) of added latency on **100% of cache misses**, indefinitely, for the
   entire duration of the regional outage. Your p50 does not move (cache hits are
   fine); your p99 becomes the timeout value.
2. **It is not a 15-minute-RTO mechanism in the "and then we are healthy" sense.**
   It is a *"we are serving, degraded, while a human does the real failover"*
   mechanism. That is genuinely valuable — it converts a hard outage into a
   latency incident, which buys back the human decision time that
   [[aws-route53]]'s Scenario B spends — but do not write "RTO: seconds" on a
   slide and mean it.
3. **Your origin-request volume to the dead primary does not decrease.** If the
   primary is *degraded* rather than dead, you are still hammering it with 100% of
   cache-miss traffic. Origin groups will not let a struggling region recover;
   they will hold it under load while also loading the standby.
4. **Failback is automatic and instant, which is a feature and a hazard.** The
   moment the primary starts returning 2xx again, CloudFront uses it — with no
   damping, no soak, and no regard for whether the database in the primary is
   still the authoritative one. See [[#Failback]] and [[split-brain-and-fencing]].

The mitigation for (1) and (3) is to *stop relying on the origin group* as soon
as a human has decided: flip the KVS flag (see
[[#Edge functions as the real failover switch]]) so the standby becomes the
**primary** origin selection, and the timeouts disappear from the path entirely.
**Origin groups are the automatic first 90 seconds; edge-function origin
selection is the deliberate remainder.**

### Lambda@Edge fires twice

Also verbatim from the failover page, and a real bug source if you have any
origin-request logic:

> When you use a Lambda@Edge function with an origin group, the function can be triggered twice for a single viewer request.

If an origin-request Lambda@Edge function does anything non-idempotent — signing,
counting, incrementing a metric, writing an audit row — it will do it twice on
every failover request. Cross-ref [[aws-lambda]].

### The `OPTIONS` footgun

> CloudFront will not failover if `OPTIONS` are not set as a Cached HTTP methods in your cache behavior.

Most cache behaviors cache `GET, HEAD` only. If your SPA does CORS preflights
through the same distribution, those `OPTIONS` requests **will not fail over**
unless you explicitly add `OPTIONS` to *cached* methods (not merely allowed
methods). A failover where every preflight 504s is a failover where the browser
app is down while the API is up.

## The GET/HEAD/OPTIONS limitation — the decision point

**Verified verbatim.** AWS states it twice on the same page — once in the
behaviour description and once as a `Note` under the status-code selection step —
in identical words:

> CloudFront fails over to the secondary origin only when the HTTP method of the viewer request is `GET`, `HEAD`, or `OPTIONS`. CloudFront does not fail over when the viewer sends a different HTTP method (for example `POST`, `PUT`, and so on).

Source: [Optimize high availability with CloudFront origin failover](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/high_availability_origin_failover.html).

There is **no setting, no flag, no quota increase and no support ticket** that
changes this. It is not a default. It is the feature's boundary.

The second, corroborating statement is in the origin `Response timeout` docs and
says the same thing from the other direction — for write methods CloudFront
"drops the connection and doesn't try again to contact the origin. The client can
resubmit the request if necessary." AWS is consistent: **CloudFront will not
replay a request with a body against a different server.** Which is, to be fair,
correct behaviour — a silently retried `POST /payments` against a second region
is a duplicate-charge bug, and CloudFront cannot know whether your endpoint is
idempotent.

### So: mechanism, or safety net?

This is the fork the task exists to resolve, and the answer depends on one fact
about the estate that this note cannot see. State both branches.

| | If the distribution fronts a **static/read-heavy** surface | If the distribution fronts a **write-capable API** |
|---|---|---|
| What origin groups do during a regional outage | Essentially everything. `GET`s fail over at the edge in seconds. | Serve the GETs, and **return 5xx to every `POST`/`PUT`/`PATCH`/`DELETE` for the entire outage**. |
| Customer-visible result | Site loads, slightly slower on cache misses. | **Site loads and every button is broken.** Arguably worse than a clean outage, because monitoring looks half-green and users retry payments. |
| Verdict | **Origin groups are the failover mechanism.** | **Origin groups are a read-path safety net only.** They are not the mechanism and must not be described as one. |

**Recommendation for this estate: treat origin groups as a safety net, not as the
mechanism, and build the mechanism on edge-function origin selection.**

The reasoning is not close. The brief describes a live product with per-region
customer data ([[research-brief]]) — that is a stateful, write-capable
application, not a static site. A failover mechanism that covers reads and drops
writes leaves you in the worst quadrant of all: **a partial failover that
monitoring reports as success.** Error budgets look fine; `2xx` rate looks fine
because reads dominate volume; meanwhile every checkout, every settings save and
every upload has been 502ing for forty minutes.

**But keep origin groups anyway**, configured with `500,502,503,504`. They cost
nothing, and they convert the first 90 seconds of a regional failure from "total
outage" to "reads work, writes fail" *without any human involvement*. That is a
strictly better starting position for the human who is about to be paged. The
mistake is not *having* origin groups — it is *believing they are the plan*.

> [!important] The sentence to put in the design doc
> "Origin groups fail over `GET`, `HEAD` and `OPTIONS` only. Our write path is not
> covered by them. The write path fails over when the KeyValueStore active-region
> flag is flipped, or when DNS moves — and one of those two must be in the runbook
> as the actual mechanism."

## Edge functions as the real failover switch

This is the highest-value section in the note. It is also the one the earlier
draft asserted without evidence, so everything below has been checked against
first-party AWS pages and the verdict on the TL;DR is recorded at the end.

### The feature

CloudFront Functions running on the **viewer request** event can change which
origin the request goes to, using a helper module. From
[Helper methods for origin modification](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/helper-functions-origin-modification.html), verbatim:

> This section applies if you dynamically update or change the origin used on the request inside your CloudFront Functions code. **You can update the origin on *viewer request* CloudFront Functions only.** CloudFront Functions has a module that provides helper methods to dynamically update or change the origin.
>
> To use this module, create a CloudFront function using **JavaScript runtime 2.0** and include the following statement in the first line of the function code:
>
> ```
> import cf from 'cloudfront';
> ```

**All three API names asserted in the TL;DR are real and correctly spelled.**
There are three methods, and the difference between them matters:

| Method | What it does | Verbatim from AWS |
|---|---|---|
| `cf.selectRequestOriginById(origin_id, {origin_overrides})` | Switches to **an origin already defined on the distribution**, by its origin ID. | "Use `selectRequestOriginById()` to update an existing origin by selecting a different origin that's already configured in your distribution. This method uses all the same settings that are defined by the updated origin." |
| `cf.createRequestOriginGroup({origin_group_properties})` | Builds an **origin group at request time** from two origin IDs plus a `failoverCriteria` status-code array. | "Use `createRequestOriginGroup()` to define two origins to use as an origin group for failover in scenarios that require high availability." |
| `cf.updateRequestOrigin({origin properties})` | Overrides arbitrary origin properties, **including an origin not on the distribution at all**. | "The origin set by the `updateRequestOrigin()` method can be any HTTP endpoint and doesn't need to be an existing origin within your CloudFront distribution." |

Two footnotes from the same page that decide which one you use here:

- `updateRequestOrigin()` **cannot touch VPC origins**: "You can't use the `updateRequestOrigin()` method to update VPC origins. The request will fail." `selectRequestOriginById()` and `createRequestOriginGroup()` **can**. If the estate is heading toward private ALBs behind VPC origins, that alone selects the method.
- `updateRequestOrigin()` on a member of an origin group only changes the primary: "If you're updating an origin that is part of an origin group, only the *primary origin* of the origin group is updated. The secondary origin remains unchanged."

`createRequestOriginGroup()` is the most powerful of the three for this use case,
because it lets you **reverse the primary/secondary order per request** — the
failover is "the group is now `[standby, primary]` instead of `[primary, standby]`" —
which keeps origin-group failover semantics on the read path *and* points the
write path at the right region. Its `failoverCriteria.statusCodes` is required
and, note: "If you overwrite an existing origin group, this array will overwrite
all failover status codes that are set in the origin group's original
configuration."

### Does it really apply to every HTTP method?

**Partly verified — read this carefully.**

What AWS states outright: the `GET`/`HEAD`/`OPTIONS` restriction is documented as
a property of **origin failover**, on the origin-failover page. The
origin-modification helper page states **no method restriction at all** — the only
restriction it names is "*viewer request* CloudFront Functions only".

What AWS does **not** state anywhere I could find: an explicit sentence saying
"origin selection works for `POST`". The nearest supporting facts are that
CloudFront Functions on viewer request run before the cache lookup on every
request, and that the documented CloudFront Functions restriction list
([Restrictions on CloudFront Functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-function-restrictions.html))
contains no method filter — it says only "CloudFront Functions can't access the
body of the HTTP request", which implies functions *do* run on requests that have
bodies.

> [!warning] Verify this by test before it goes in a runbook
> The reasoning is sound and the absence of a documented restriction is
> meaningful, but "AWS did not say you can't" is weaker evidence than a quote.
> **Before this becomes the estate's write-path failover mechanism, run a
> five-minute test**: a distribution, two origins that echo which one they are,
> a viewer-request function calling `selectRequestOriginById()`, and a `curl -X POST`.
> Logged in [[#Open questions]]. This note will not claim it as verified fact.

The corresponding downside AWS *does* state plainly: a viewer-request function
"will run on every request when this function is used", where Lambda@Edge origin
logic "only runs on cache misses". **You pay CloudFront Functions invocation cost
on 100% of requests including cache hits.** See [[#Cost]].

### The flag: CloudFront KeyValueStore

A function cannot read environment variables — from the runtime docs, verbatim:
"There is no access to environment variables. Instead, you can use CloudFront
KeyValueStore to create a centralized datastore of key-value pairs for your
CloudFront Functions." So the active-region flag has to live in KVS (or be
hard-coded in the function, which would mean a function publish + distribution
deploy on every failover — see [[#The distribution deploy is not a failover mechanism]]).

From [Amazon CloudFront KeyValueStore](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/kvs-with-functions.html), verbatim:

> CloudFront KeyValueStore is a secure, global, low-latency key value datastore that allows **read access** from within CloudFront Functions...
>
> With CloudFront KeyValueStore, you make updates to function code and updates to the data associated with a function independently of each other. This separation simplifies function code and **makes it easy to update data without the need to deploy code changes.**

**Read access only.** A function can read the flag; it cannot write it. There is
no risk of an edge function flipping the estate's failover state, which is
correct — but it also means the flag must be flipped by an external caller using
the KeyValueStore API.

**Propagation time — the TL;DR's "a few seconds" claim is verified**, though not
from the developer guide. It comes from the AWS News Blog launch post,
[Introducing Amazon CloudFront KeyValueStore](https://aws.amazon.com/blogs/aws/introducing-amazon-cloudfront-keyvaluestore-a-low-latency-datastore-for-cloudfront-functions/):

> changes are propagated to all CloudFront edge locations in a few seconds

Treat that as AWS's *characterisation*, not an SLO. **AWS publishes no numeric
propagation SLO for KeyValueStore.** For an RTO calculation, budget "seconds, and
verify by observation during the failover rather than assuming".

**Consistency model.** The developer guide gives one precise statement, and it is
about invocation-level consistency rather than global consistency. From
[Work with key value data](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/kvs-with-functions-kvp.html), verbatim:

> When you invoke your CloudFront Functions by using CloudFront KeyValueStore, the values in the key value store aren't updated or changed during the invocation of the function. **Updates are processed in between invocations of a function.**

That guarantees a *single function run* sees a stable snapshot — it will not read
`region=eu-west-1` for one key and `region=eu-west-2` for another mid-execution.
It guarantees **nothing** about two different edge locations agreeing with each
other at the same instant.

> [!danger] The split-brain window is real and is measured in seconds
> Combine "propagated in a few seconds" with "no cross-edge consistency
> guarantee" and you get the only honest description: **for a few seconds after
> the flip, some edge locations send traffic to `eu-west-1` and others send it to
> `eu-west-2`, simultaneously.**
>
> For an active/passive pair with an **asynchronous replica**, that window is
> writes landing in two different databases. A few seconds of dual-writing is
> vastly better than the minutes DNS gives you, but it is not zero and it is
> **not a fencing mechanism**. The database must refuse writes in the old primary
> — see [[split-brain-and-fencing]]. Do not let "the KVS flip is atomic" into the
> design doc, because it is not.
>
> Also: the write API is all-or-nothing *per call* — `UpdateKeys` performs its
> deletes and puts "in one all-or-nothing operation" — so a multi-key flip is
> atomic **at the API**, not at the edges.

### The flip is a single API call, and its blast radius is one key

```bash
# Read current state (safe, run this first in the runbook)
aws cloudfront-keyvaluestore get-key \
  --kvs-arn "$KVS_ARN" --key active-region

# Get the ETag required for any write
ETAG=$(aws cloudfront-keyvaluestore describe-key-value-store \
  --kvs-arn "$KVS_ARN" --query ETag --output text)

# THE FAILOVER. This is the whole thing.
aws cloudfront-keyvaluestore put-key \
  --kvs-arn "$KVS_ARN" \
  --if-match "$ETAG" \
  --key active-region \
  --value eu-west-2
```

The `--if-match` ETag is mandatory on writes and is an optimistic-concurrency
guard: two operators racing to flip the same key, one wins, the other gets a
mismatch rather than a silent clobber. **That is a genuinely useful 3am property**
and it is the opposite of `ChangeResourceRecordSets`, which is last-write-wins.

Note that a write also needs a fresh `ETag`, and there are **two different
`DescribeKeyValueStore` operations returning two different, non-interchangeable
ETags** — the CloudFront one and the CloudFront KeyValueStore one. Verbatim: "Each
DescribeKeyValueStore operation returns a *different* `ETag`. The `ETags` aren't
interchangeable." Get this wrong in the failover script and you discover it during
the incident.

> [!danger] The credential trap that will break your failover script
> This one is not obvious and it is exactly the kind of thing that fails at 3am.
> From [Restrictions on CloudFront Functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-function-restrictions.html), verbatim:
>
> > The CloudFront KeyValueStore API is a global service that uses **Signature Version 4A (SigV4A)** for authentication. Using temporary credentials with SigV4A requires **version 2 session tokens**.
> >
> > To call the CloudFront KeyValueStore API, use a *Regional* endpoint in AWS STS to return a *version 2* session token. **If you use the *global* endpoint for AWS STS (`sts.amazonaws.com`), AWS STS will generate a *version 1* session token, which isn't supported by Signature Version 4A (SigV4A). As a result, you will receive an authentication error.**
>
> If the failover tooling assumes a role through the global STS endpoint — which
> plenty of older CI images and SDK configurations still do by default — **the
> failover command fails with an authentication error, during the incident, for a
> reason that looks nothing like the actual cause.**
>
> Fix it once, in advance, one of two ways:
> - Configure the CLI/SDK to use **regional STS endpoints** (AWS's recommendation:
>   "Regional endpoints provide higher availability and failover scenarios"), or
> - `aws iam set-security-token-service-preferences --global-endpoint-token-version v2Token`
>   at the account level.
>
> **Test the failover command with the exact credentials the runbook will use.**
> Cross-ref [[aws-iam]] and [[failover-orchestration]].

### The function

```javascript
import cf from 'cloudfront';

// Associated key value store. One KVS per function (hard quota).
const kvsHandle = cf.kvs();

async function handler(event) {
  const request = event.request;

  // Default to the primary. A KVS read failure MUST NOT take the site down,
  // so every failure path lands on the region we believe is healthy.
  // NOTE: get() has NO "default" option. Its only option is `format`
  // (string | json | bytes). A missing key THROWS. The try/catch is the
  // default, and it is mandatory, not defensive decoration.
  let active = 'primary';
  try {
    active = await kvsHandle.get('active-region', { format: 'string' });
  } catch (e) {
    console.log('kvs read failed, defaulting to primary: ' + e);
  }

  // Order the pair so the ACTIVE region is the origin-group primary.
  // Reads still get automatic edge failover; writes go to the active region.
  const ordered = active === 'standby'
    ? ['alb-eu-west-2', 'alb-eu-west-1']
    : ['alb-eu-west-1', 'alb-eu-west-2'];

  cf.createRequestOriginGroup({
    originIds: [{ originId: ordered[0] }, { originId: ordered[1] }],
    failoverCriteria: { statusCodes: [500, 502, 503, 504] }
  });

  return request;
}
```

Four things about that function, each of which is a way to cause an outage:

1. **`cf.kvs()`, `get()`, `exists()` and `meta()` are the documented helper
   names** — see [Helper methods for key value stores](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/functions-custom-methods.html).
   `get()` returns a Promise, hence `async`/`await`, hence **runtime 2.0 is
   mandatory** (runtime 1.0 has no `async`). If the distribution has an existing
   runtime-1.0 viewer-request function, this is a migration, not an addition.
   Also verbatim from that page, and a real trap if you read several keys:
   "Using promise combinators (for example, `Promise.all`, `Promise.any`, and
   promise chain methods (for example, `then` and `catch`) can require high
   function memory usage. If your function exceeds the maximum function memory
   quota, it will fail to execute." **Use sequential `await`, not `Promise.all`.**
   Function memory is **2 MB** ([Quotas on CloudFront Functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-limits.html)).
2. **Fail open, to the primary.** A function that throws returns a 5xx to the
   viewer. A KVS read failure must not be able to take down a healthy site, so
   every error path defaults to the current primary.
3. **It runs on every request**, including cache hits, and counts against
   **compute utilization** — a percentage of the maximum allowed run time, visible
   as a CloudWatch metric. Keep it this small. A function that exceeds the limit
   fails the request.
4. **It cannot make network calls.** "There is no support for network calls. For
   example, XHR, HTTP(S), and socket are not supported." The function cannot
   health-check anything. It reads a flag that a human or an orchestrator set.
   **That is the correct design** for a 2-hour-RPO async replica — the failover
   decision stays with a human, exactly as argued in [[aws-route53]] for weighted
   100/0 over automatic failover routing.

### The distribution deploy is not a failover mechanism

Worth stating plainly because it is the obvious-looking alternative: you could
fail over by calling `UpdateDistribution` to change the origin group's ordering,
or by changing the origin's `domainName`.

**Don't.** Three reasons:

1. It is a **`us-east-1` control-plane call** (see
   [[#Does this service cross regions at all?]]). For the US pair that is the
   region you are failing away from — the same circular dependency
   [[aws-route53]] documents for `ChangeResourceRecordSets`.
2. It has to propagate a configuration change to every edge location. AWS does
   document that the change is applied per-edge as it lands: "Until the
   distribution configuration is updated in a given edge location, CloudFront
   continues to forward requests to the previous origin." Historically minutes.
   Against a 900 s budget with a human decision gate already consuming several
   minutes, spending an unbounded few more is unnecessary when a KVS write costs
   seconds.
3. It means **running Terraform during an incident**, which every note in this
   vault argues against.

The KVS flip has none of those properties: no distribution deployment, no
function publish, no Terraform, seconds not minutes.

### Verdict on the TL;DR

**The TL;DR's claims in this area are correct and stand unmodified.**
`cf.selectRequestOriginById()` and `cf.createRequestOriginGroup()` are real
methods with those exact names; they are viewer-request-only; KeyValueStore is
the documented mechanism for the flag; AWS does say updates propagate "in a few
seconds"; and no distribution deployment is required. The two things the TL;DR
*understates* and which this section adds:

- **"Applies to every HTTP method" is a reasonable inference, not a documented
  guarantee.** Test it.
- **"Propagates in a few seconds" is not atomic.** There is a real,
  seconds-long window where different edges disagree, and for an async-replica
  pair that window is a dual-write window. It needs fencing, not faith.

## The cold-cache thundering herd

### The problem, stated properly

At the instant of failover, three things are true at once, and each makes the
other two worse:

1. **The standby's edge cache for it is empty.** This needs care, because the
   usual phrasing is wrong. CloudFront's cache key is a property of the *cache
   behavior*, not of the origin — so objects already cached at the edge are still
   served after the switch and do not re-fetch. What is empty is the **Origin
   Shield / regional-edge-cache layer in front of the standby** (a different
   Origin Shield region entirely, see below), and every object whose TTL expires
   from this moment on becomes an origin request to the *new* region. So it is
   less a wall and more a **step change followed by a rising tide**: cached
   objects keep serving, and then over the next TTL period, 100% of origin fetches
   land on the standby.
2. **The standby's compute is scaled down.** That is what "warm standby" means
   ([[research-brief]]). It is sized for a fraction of production, and EKS or ASG
   scale-out takes minutes you do not have inside 900 seconds. See [[aws-eks]].
3. **The standby's database was just promoted.** A freshly-promoted replica has a
   cold buffer pool. Every query is disk. The queries that ran in 5 ms in the
   primary run in 200 ms here, until the working set is resident. See
   [[aws-rds-postgres]] and [[aurora-failover-mechanics]].

Now add the CloudFront-specific multiplier from
[[#The killer: CloudFront does not remember]]: if you have *not* flipped the KVS
flag and are relying on the origin group, **every one of those requests first
spends up to 30 s failing against the dead primary**, so they arrive at the
standby in a bunched, retried, latency-shifted clump rather than smoothly.

The failure mode is circular and it is the classic one: slow database → requests
pile up → connection pool exhausted → 5xx → CloudFront's origin group treats 5xx
as failover criteria → **requests bounce back to the dead primary, time out, and
return** → clients retry → more load. **You can brown out the standby with the
traffic you just saved.**

### Origin Shield is the mitigation, and it is a good one

From [Use Amazon CloudFront Origin Shield](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/origin-shield.html), verbatim — this is precisely the thundering-herd property:

> **Reduced origin load** — Origin Shield can further reduce the number of simultaneous requests that are sent to your origin for the same object. **Requests for content that is not in Origin Shield's cache are consolidated with other requests for the same object, resulting in as few as one request going to your origin.** Handling fewer requests at your origin can preserve your origin's availability during peak loads or unexpected traffic spikes...

**Request collapsing is the feature.** Without Origin Shield, a cold object
requested simultaneously from twelve regional edge caches is twelve origin
requests. With Origin Shield, it is one. On a cold cache against a
just-promoted database, that difference is the difference between a slow minute
and an outage.

### Origin Shield is per-origin — which is exactly what failover needs

Verbatim:

> **Origin Shield is a property of the origin.** For each origin in your
> CloudFront distributions, you can separately enable Origin Shield in whichever
> AWS Region provides the best performance for that origin.

And, critically, its documented interaction with origin groups:

> Origin Shield is compatible with CloudFront origin groups. Because Origin Shield is a property of the origin, requests always travel through Origin Shield for each origin even when the origin is part of an origin group. **For a given request, CloudFront routes the request to the primary origin in the origin group through the primary origin's Origin Shield. If that request fails (according to the origin group failover criteria), CloudFront routes the request to the secondary origin through the secondary origin's Origin Shield.**

So the correct configuration writes itself, and it is **not** "one Origin Shield
for the distribution":

| Origin | Origin Shield region | Rationale |
|---|---|---|
| `alb-eu-west-1` (primary) | **`eu-west-1`** | Same region as the origin — AWS's own rule. |
| `alb-eu-west-2` (standby) | **`eu-west-2`** | Same region as the origin. Shields the standby the moment it takes traffic. |
| `alb-us-east-1` (primary) | **`us-east-1`** | |
| `alb-us-west-2` (standby) | **`us-west-2`** | |

AWS's rule, verbatim: "If your origin is in an AWS Region in which CloudFront
offers Origin Shield... **enable Origin Shield in the same Region as your
origin.**"

> [!warning] The CA pair has a real, specific Origin Shield problem
> The documented list of Origin Shield regions is: `us-east-2`, `us-east-1`,
> `us-west-2`, `ap-south-1`, `ap-northeast-2`, `ap-southeast-1`, `ap-southeast-2`,
> `ap-northeast-1`, `eu-central-1`, `eu-west-1`, `eu-west-2`, `sa-east-1`,
> `me-central-1`.
>
> **Neither `ca-central-1` nor `ca-west-1` is on it.** The EU and US pairs are
> fully covered — all four of `eu-west-1`, `eu-west-2`, `us-east-1`, `us-west-2`
> are supported. **The CA pair is covered by neither region.**
>
> Worse, AWS's fallback table maps **`ca-central-1` → `us-east-1`**. Taken
> literally, enabling Origin Shield for a Montreal origin puts a **caching layer
> for Canadian customer data in N. Virginia.** For a deployment whose entire
> premise is that "a customer in Canada lives entirely in the Canadian
> deployment" ([[research-brief]]), that is a **data-residency decision, not a
> performance tuning knob.** Escalate to [[data-residency]] before anyone ticks
> the box.
>
> And **`ca-west-1` (Calgary) does not appear in the fallback table at all** —
> AWS documents no recommended Origin Shield region for it. Add this to the
> `ca-west-1` parity findings in [[region-pair-selection]]; it is one more small
> gap in a young region.
>
> **Recommendation for the CA pair: do not enable Origin Shield.** Accept the
> thundering herd and mitigate it with pre-scaled standby capacity and longer
> TTLs instead. A residency breach is a worse outcome than a slow failover.

### The other mitigations, in order of value per effort

1. **Pre-scale the standby before you flip, not after.** This is the real answer
   and it is not a CloudFront control. Origin Shield reduces the herd; it does
   not create capacity. Scale EKS/ASG first, flip the flag second. Put the
   ordering in the runbook — see [[#Failover procedure]] and
   [[failover-orchestration]].
2. **Raise TTLs on anything that tolerates it.** Every second of TTL is a second
   the standby does not have to serve that object. Cheap, and it is a change you
   can make *before* the incident. Note the cost interaction below: a TTL under
   3600 s makes a `GET` count as a *dynamic* request for Origin Shield billing.
3. **`stale-if-error` — verified, supported, and underused.** CloudFront has
   honoured both `stale-while-revalidate` and `stale-if-error` since
   **17 May 2023** ([AWS What's New](https://aws.amazon.com/about-aws/whats-new/2023/05/amazon-cloudfront-stale-while-revalidate-stale-if-error-cache-control-directives/)).
   Verbatim, `stale-while-revalidate` lets CloudFront "immediately deliver stale
   responses to users while it revalidates caches in the background", and
   `stale-if-error` "defines how long CloudFront should reuse stale responses if
   there's an error".

   > [!tip] This is a genuinely good and nearly-free failover mitigation
   > `stale-if-error` is **exactly** the thundering-herd mitigation this section
   > needs: during the window where the primary is erroring and the standby is
   > cold and slow, CloudFront serves the last-known-good object instead of
   > hammering either origin. It is an **origin-side response header**, not a
   > distribution setting — `Cache-Control: max-age=60, stale-if-error=86400` — so
   > it is a change in the application, deployable today, with no CloudFront
   > change at all.
   >
   > **Caveat, and it is the one that catches people:** CloudFront serves stale
   > content up to the `stale-*` value **or the cache behavior's maximum TTL,
   > whichever is less.** A `stale-if-error=86400` against a cache policy whose
   > `max_ttl` is 3600 gives you one hour, not one day. Raise `max_ttl` on the
   > cache policy or the directive silently does much less than you think.

   Custom error pages with a long error-caching TTL are the complementary
   CloudFront-native control and are worth configuring anyway: they turn a 504
   storm into a cached, cheap, branded error rather than 30 seconds of nothing.
4. **Tune the primary's timeouts down** (see
   [[#Timeouts and attempts — where the seconds actually go]]) so that requests
   reach the standby in ~5 s rather than ~30–90 s. Compresses the clump.
5. **Flip the KVS flag early** so the standby becomes origin-group *primary* and
   the dead-primary timeout leaves the request path entirely.

### Origin Shield's own availability, and its Lambda@Edge side effect

Two more verbatim facts worth knowing before enabling it:

> Connections from CloudFront locations to Origin Shield also use active error tracking for each request to **automatically route the request to a secondary Origin Shield location if the primary Origin Shield location is unavailable.**

Good — enabling Origin Shield does not add a single point of failure in the
obvious way.

> When you use Origin Shield with Lambda@Edge, **origin-facing triggers (origin request and origin response) run in the AWS Region where Origin Shield is enabled.**

**This moves your Lambda@Edge execution region.** If any origin-request
Lambda@Edge function reads from a regional resource — a DynamoDB table, an SSM
parameter, a Secrets Manager secret — turning on Origin Shield silently relocates
where that read happens, and the IAM/KMS grants may not exist there. Cross-ref
[[aws-lambda]], [[aws-kms]] and [[aws-iam]]. A genuinely nasty one, because it
looks like a caching change.

Finally, for the runbook: Origin Shield hits are observable. "Cache hits from
Origin Shield appear as `OriginShieldHit` in the `x-edge-detailed-result-type`
field in CloudFront logs."

## Terraform implementation

### Where this stack lives

Per [[#Does this service cross regions at all?]], the edge layer is **one global
stack per public hostname/environment**, not a per-region module. It consumes
outputs from both regional stacks. Structurally this is the same stack shape
[[aws-route53]] recommends for DNS, and in a cookiecutter monorepo it should
probably *be* the same stack — `stacks/edge/` — because the CloudFront
distribution, the `us-east-1` ACM certificate, the `CLOUDFRONT`-scope WAF ACL and
the Route 53 records are all global objects that change together.

See [[provider-aliases-vs-separate-stacks]] for the general argument and
[[module-patterns]] for the module conventions this signature follows; this
note's contribution is that CloudFront **forces** the issue — there is no
per-region CloudFront resource to put in a per-region stack. Note the contrast
with genuinely regional prerequisites like [[aws-ecr]], where the cookiecutter
renders one module instance per region; the edge stack renders exactly once and
takes the region pair as *data*.

```hcl
terraform {
  required_version = ">= 1.9"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.0" }
  }
}

# CloudFront, ACM-for-CloudFront and CLOUDFRONT-scope WAF all live in us-east-1.
# This is the DEFAULT provider for the edge stack, which is deliberate: it makes
# the us-east-1 dependency visible in the file rather than hidden in an alias.
provider "aws" {
  region = "us-east-1"
  default_tags { tags = local.common_tags }
}

# Read-only aliases. The edge stack does not CREATE anything regional; it only
# needs to look things up if you are not passing them in via remote state.
provider "aws" {
  alias  = "primary"
  region = var.primary_region
}

provider "aws" {
  alias  = "standby"
  region = var.standby_region
}
```

### Module signature for a templated monorepo

The variable surface is the interesting part. It is deliberately **symmetric** —
primary and standby are the same shape — so the cookiecutter template renders one
block per region from the same loop, and so that failback is not a different code
path from failover.

```hcl
variable "env" {
  type        = string
  description = "Environment slug rendered by cookiecutter, e.g. prod, staging."
}

variable "public_domain" {
  type = string
}

variable "regions" {
  description = <<-EOT
    The region pair for this deployment. Order is NOT significant — which region
    is live is runtime state in KeyValueStore, not Terraform state. Terraform
    owns the possibility of serving from either; the KVS flag owns which one
    actually is. This separation is the whole design: failover must never be a
    terraform apply.
  EOT
  type = map(object({
    alb_dns_name        = string
    origin_shield_region = optional(string) # null = disabled (see CA pair)
  }))
  # Example:
  # {
  #   "eu-west-1" = { alb_dns_name = "...", origin_shield_region = "eu-west-1" }
  #   "eu-west-2" = { alb_dns_name = "...", origin_shield_region = "eu-west-2" }
  # }
}

variable "default_active_region" {
  description = <<-EOT
    Seed value for the KeyValueStore active-region key, applied ONLY at create
    time. Terraform must not manage this value thereafter — see the
    ignore_changes below. If Terraform owns the live failover state, then a
    routine apply during an incident fails you back into a dead region.
  EOT
  type = string
}

variable "origin_read_timeout_seconds" {
  description = "Default 30s is almost certainly wrong for an API. See the 16 July 2026 log analysis."
  type        = number
  default     = 10
}

variable "origin_connection_attempts" {
  type    = number
  default = 2 # 3 x 10s = 30s of dead-primary latency per cache miss. 2 is 20s.
}
```

### The distribution

```hcl
locals {
  # Deterministic, stable origin IDs. These strings are referenced BY NAME from
  # the CloudFront Function, so they are a public contract between the Terraform
  # and the JavaScript. Changing one silently breaks origin selection at the
  # edge with no Terraform error. Treat them as an interface.
  origin_ids = { for r, _ in var.regions : r => "alb-${r}" }
}

resource "aws_cloudfront_distribution" "this" {
  enabled         = true
  is_ipv6_enabled = true
  aliases         = ["app.${var.public_domain}"]
  web_acl_id      = aws_wafv2_web_acl.edge.arn # CLOUDFRONT scope, us-east-1

  dynamic "origin" {
    for_each = var.regions
    content {
      origin_id   = local.origin_ids[origin.key]
      domain_name = origin.value.alb_dns_name

      # Shared secret proving the request came through CloudFront. The ALB
      # listener rule rejects anything without it. Without this, both regional
      # ALBs are directly reachable and the WAF is decorative.
      custom_header {
        name  = "x-edge-shared-secret"
        value = var.edge_shared_secret # from Secrets Manager, not a tfvar
      }

      custom_origin_config {
        http_port                = 80
        https_port               = 443
        origin_protocol_policy   = "https-only"
        origin_ssl_protocols     = ["TLSv1.2"]
        origin_read_timeout      = var.origin_read_timeout_seconds
        origin_keepalive_timeout = 60
      }

      connection_attempts = var.origin_connection_attempts
      connection_timeout  = 5

      # Per-origin, in the origin's OWN region. Not one shield for the
      # distribution. Null for the CA pair - no supported region.
      dynamic "origin_shield" {
        for_each = origin.value.origin_shield_region == null ? [] : [1]
        content {
          enabled              = true
          origin_shield_region = origin.value.origin_shield_region
        }
      }
    }
  }

  # The STATIC origin group. This is the automatic read-path safety net that
  # applies before anyone is awake. The CloudFront Function overrides the
  # ordering per-request once a human has decided.
  origin_group {
    origin_id = "alb-pair"

    failover_criteria {
      # 503 = could not connect. 504 = timed out. Without BOTH of these, a dead
      # region does not trigger failover at all. Do NOT add 403/404.
      status_codes = [500, 502, 503, 504]
    }

    member { origin_id = local.origin_ids[var.default_active_region] }
    member {
      origin_id = one([for r, _ in var.regions : local.origin_ids[r]
                       if r != var.default_active_region])
    }
  }

  default_cache_behavior {
    target_origin_id       = "alb-pair"
    viewer_protocol_policy = "redirect-to-https"

    allowed_methods = ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"]

    # OPTIONS must be a CACHED method or CORS preflights will not fail over.
    # See "The OPTIONS footgun".
    cached_methods = ["GET", "HEAD", "OPTIONS"]

    cache_policy_id          = data.aws_cloudfront_cache_policy.caching_optimized.id
    origin_request_policy_id = data.aws_cloudfront_origin_request_policy.all_viewer.id

    function_association {
      event_type   = "viewer-request" # origin selection is viewer-request ONLY
      function_arn = aws_cloudfront_function.origin_selector.arn
    }
  }

  restrictions { geo_restriction { restriction_type = "none" } }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.edge.arn # MUST be us-east-1
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }
}
```

### The KeyValueStore and the function

```hcl
resource "aws_cloudfront_key_value_store" "failover" {
  name    = "${var.env}-active-region"
  comment = "Runtime failover state. Flipped by the runbook, NOT by Terraform."
}

# Seed the key once, then never manage it again.
resource "aws_cloudfrontkeyvaluestore_key" "active_region" {
  key_value_store_arn = aws_cloudfront_key_value_store.failover.arn
  key                 = "active-region"
  value               = var.default_active_region

  lifecycle {
    # THE MOST IMPORTANT FOUR LINES IN THIS NOTE.
    # Without this, a routine `terraform apply` during or after an incident
    # silently fails traffic BACK to the dead region, because Terraform state
    # says the primary is active and reality says otherwise. An unplanned,
    # unattended failback into a region whose database is no longer
    # authoritative is a split-brain event triggered by a pipeline.
    ignore_changes = [value]
  }
}

resource "aws_cloudfront_function" "origin_selector" {
  name    = "${var.env}-origin-selector"
  runtime = "cloudfront-js-2.0" # 2.0 is mandatory: KVS + async/await
  publish = true

  key_value_store_associations = [aws_cloudfront_key_value_store.failover.arn]

  code = templatefile("${path.module}/functions/origin-selector.js", {
    # Origin IDs injected at render time so the JS and the HCL cannot drift.
    primary_origin_id = local.origin_ids[var.default_active_region]
    standby_origin_id = one([for r, _ in var.regions : local.origin_ids[r]
                             if r != var.default_active_region])
  })
}
```

> [!note] Provider resource names — verified, with two caveats
> Checked against the `hashicorp/aws` registry docs. All confirmed:
> - **`aws_cloudfront_key_value_store`** — real. ([registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudfront_key_value_store))
> - **`aws_cloudfrontkeyvaluestore_key`** — real, note the **unusual
>   run-together name** (no underscores in `cloudfrontkeyvaluestore`). Required
>   arguments are exactly `key_value_store_arn`, `key`, `value`.
>   ([registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudfrontkeyvaluestore_key))
> - **`key_value_store_associations`** on `aws_cloudfront_function` — real, and
>   the provider docs carry the quota verbatim: "AWS limits associations to one
>   key value store per function." `runtime` valid values are exactly
>   `cloudfront-js-1.0` and `cloudfront-js-2.0`; `publish` defaults to `true`.
>
> **Caveat 1 — do not use `aws_cloudfrontkeyvaluestore_keys_exclusive` for the
> failover key.** The provider also offers an *exclusive* variant, which by
> convention manages the complete set of keys and **removes any key it does not
> know about**. Pointing that at the failover store means a `terraform apply`
> can delete `active-region` outright, which is strictly worse than the
> fail-back problem `ignore_changes` solves. Use the singular resource.
>
> **Caveat 2 — `response_completion_timeout` on the origin block is still
> unverified** against a pinned provider release. It is a recent CloudFront API
> attribute; check `terraform providers schema -json` before using it. Left in
> [[#Open questions]].

### Outputs the regional stacks need back

```hcl
output "distribution_domain_name" {
  value = aws_cloudfront_distribution.this.domain_name
}

# Regional stacks need this to write the ALB listener rule that rejects
# requests not carrying the shared secret.
output "edge_shared_secret_arn" {
  value = aws_secretsmanager_secret.edge_shared_secret.arn
}

# The failover runbook needs this. Print it; don't make someone look it up
# through a console that may be unavailable.
output "failover_kvs_arn" {
  value       = aws_cloudfront_key_value_store.failover.arn
  description = "aws cloudfront-keyvaluestore put-key --kvs-arn <this> --key active-region --value <region>"
}
```

## Case study — the 16 July 2026 CloudFront VPC Origins event

> [!important] Verification verdict, stated first
> **The incident is real.** It was reported first-hand by *The Register* on the
> day, quoting AWS's own Service Health Dashboard updates verbatim, and
> corroborated by an independent engineering write-up with access-log data.
>
> **Two claims in the earlier TL;DR did not survive contact with the sources:**
> 1. **The "~3 h 33 m" duration is one reported figure among several and is not
>    confirmed by the most credible source.** *The Register* reports a start of
>    0145 PDT and recovery by 1739 UTC — roughly **eight hours** of degradation.
>    Several secondary aggregator sites give an impact window of **07:45–11:18 UTC
>    (3 h 33 m)**, which is presumably the peak-impact window rather than the full
>    event. The TL;DR has been changed to "several hours" with the range recorded.
> 2. **No durable first-party AWS post-event summary was found.** AWS's statements
>    exist as Health Dashboard updates, which are ephemeral, and are preserved only
>    through third-party quotation. There is no `aws.amazon.com/message/...` page
>    for this event that I could locate. **Treat the AWS quotes below as
>    second-hand but well-attested**, not as citable first-party AWS text.
>
> Because the event is real and the architectural lesson is sound, the case study
> stays. Because the duration and the first-party sourcing are weaker than the
> earlier draft implied, both are now stated as such in the TL;DR and here.

### What happened

On **16 July 2026**, CloudFront returned widespread 5xx errors — **but only to
distributions using the VPC Origins feature.** Distributions using ordinary public
origins were unaffected.

AWS's Service Health Dashboard statements, as quoted by
[The Register](https://www.theregister.com/off-prem/2026/07/16/aws-cloudfront-outage-serves-errors-instead-of-websites/5272421):

> We are experiencing increased 5xx errors for CloudFront customers utilizing VPC Origins connectivity.

> Based on our investigation, we believe the root cause is related to a packet processing subsystem responsible for routing requests.

And the root-cause statement, as quoted by The Register and repeated consistently
across every secondary source:

> an internal constraint on the fleet that manages connections to private VPC origins. When this constraint was reached, the system responsible for distributing routing configuration to network processors failed to load the updated configuration data correctly, affecting routing of VPC Origin connections.

Customers named across reporting include Hugging Face, Instructure (Canvas),
Blackboard, Ubiquiti and the UK National Lottery. **AWS's advised workaround was
to change the origin type** — i.e. move off VPC Origins to a public origin — and
customers who did so were later told they could safely revert.

### Why this is in a multi-region vault

**Because a second region would not have helped, and that is the entire point.**

VPC Origins are a *regional* resource. The natural assumption is therefore that a
VPC Origins failure is a regional failure, and that a standby region fixes it. It
is not and it does not. The failing component was the **system that distributes
routing configuration to network processors** — a global CloudFront subsystem.
Every VPC Origin in every region failed together.

This is the concrete, dated illustration of the sentence in
[[#Does this service cross regions at all?]]: **the distribution is a shared fate
you cannot mirror away.** It belongs next to the fault-isolation argument in
[[aws-regional-outages]] and [[lessons-and-antipatterns]].

The honest conclusion is uncomfortable and should be written down rather than
argued away:

> **Active/passive multi-region reduces your exposure to regional events. It does
> not reduce your exposure to CloudFront itself.** A CloudFront global failure is
> an accepted risk of using CloudFront, exactly as a Route 53 global failure is an
> accepted risk of using Route 53. The only mitigation is not using it, or having
> a second CDN — and a second CDN means a second cache, a second WAF, a second
> certificate estate and a DNS layer to choose between them, for an event class
> that is rare. **Recommendation: accept the risk, and make sure the risk register
> says so explicitly rather than implying that "multi-region" covers it.**

There is one cheap, real mitigation the event does suggest, and it is worth a
ticket: **AWS's workaround was "change the origin type".** That is only possible
in minutes if you already *have* a public-origin path defined. Which argues for
keeping the standby region's ALB reachable as a **conventional custom origin**
rather than exclusively as a VPC Origin — the standby origin then doubles as an
escape hatch from a VPC-Origins-specific fault. Note the interaction with
[[#Edge functions as the real failover switch]]: `cf.updateRequestOrigin()`
**cannot** modify VPC origins, so a distribution built entirely on VPC Origins has
fewer edge-side escape routes than one built on custom origins. See
[[#Decisions to make]].

### The log evidence — and why it confirms the timeout arithmetic

This is the most useful part of the event for this note, and it comes from an
independent engineering analysis rather than from AWS: Suzuki Ryo,
["CloudFront VPC Origin Outage (2026-07-16): Confirming from logs that approximately 30-second timeout waits were occurring"](https://dev.classmethod.jp/en/articles/cloudfront-vpc-origin-incident-20260716-log-analysis/),
Classmethod DevelopersIO, 21 July 2026 — a log-level analysis of a real affected
distribution.

The findings, which line up exactly with
[[#Timeouts and attempts — where the seconds actually go]]:

| Measurement | Before (07:30–07:52) | During (07:54–08:29) |
|---|---|---|
| Mean `time-taken` | 0.374–0.535 s | **10–14 s** |
| p95 `time-taken` | ~1.2 s | **~30.3 s** |

**p95 settled at approximately 30.3 seconds — the configured `OriginReadTimeout`.**
That is the 30-second default response timeout, visible in production access logs,
as the dominant user experience of the incident. The dominant `x-edge-result-type`
values were `ClientCommError` (89.3%) and `ClientHungUpRequest` (10.7%) — i.e.
**users and clients gave up before CloudFront did.**

And the sentence that matters most for this vault: the analysis reports that
**origin failover did function, but users waited approximately 30 seconds before
receiving a degraded response, because "the secondary was attempted only after the
timeout elapsed."**

> [!important] This is the "CloudFront does not remember" behaviour, observed
> A real incident, a real distribution, real logs: **origin groups worked exactly
> as documented and the customer experience was still thirty seconds of nothing
> per cache miss, sustained.** Not for the first request. For every request.
>
> Three lessons, all actionable this week and none of them requiring a second
> region:
> 1. **Tune `origin_read_timeout` down.** The default 30 s is the thing users
>    experienced. For a JSON API, 5–10 s. This is a one-line Terraform change with
>    no replacement risk and it is the highest-value single edit in this note.
> 2. **Alarm on CloudFront `OriginLatency` p95, not just error rate.** In this
>    incident the error rate story was incomplete; the *latency* story was
>    unambiguous and arrived first. See [[observability-multi-region]].
> 3. **Put `time-taken` and `x-edge-result-type` in the standard log query.**
>    `ClientCommError` dominating is the signature of an origin that has stopped
>    answering. Have the query written before you need it.

## Choosing the failover mechanism — origin groups vs Route 53 vs Global Accelerator

This is the cross-service decision the estate actually has to make, and three
notes each own a third of it. Here is the whole thing in one place.

### The three candidates

| | **CloudFront origin group + KVS flag** | **[[aws-route53]] DNS failover** | **[[aws-global-accelerator]]** |
|---|---|---|---|
| Where the switch happens | At the edge, per request | In the DNS answer | In the AWS edge network, per connection |
| What the client has to do | **Nothing.** Same hostname, same IP, same TCP connection target | **Re-resolve.** And it won't, reliably | **Nothing.** Static anycast IPs never change |
| Switch latency (mechanism only) | **Seconds** (KVS propagation, "a few seconds", no SLO) | **~90 s** best case (30 s detection + propagation + 60 s TTL) | **Seconds**, but "the updated setting applies to only new connections" |
| Client DNS cache exposure | **None** | **The whole problem.** JVM caching forever, ALB keepalive default 3600 s | **None** |
| Control-plane dependency at failover | KeyValueStore API (global, SigV4A) | `ChangeResourceRecordSets` — **`us-east-1`** | Global Accelerator control plane |
| Covers `POST`/`PUT`/`DELETE`? | **Origin group: no. KVS origin selection: yes** (untested, see open questions) | Yes — it's just DNS | Yes — it's TCP/UDP |
| Works for non-HTTP (raw TCP, gRPC, WebSocket) | HTTP/HTTPS only; **Origin Shield unsupported for gRPC** | Yes | **Yes — its main advantage** |
| Automatic on health failure | Yes (origin group, read path only) | Yes (health checks) — **which is a liability at RPO 2h** | Yes |
| Deliberate, human-gated switch | **Yes — flip a KVS key** | Yes — weighted 100/0, or ARC | **Yes — traffic dial to 0** |
| Incremental cost | **~$0.** Functions $0.10/M invocations (2 M/mo free), KVS reads $0.03/M | ~$0.50–$1.50/mo per health check | **$0.025/accelerator/hour ≈ $18.25/mo** + DT-Premium $0.007–$0.105/GB |
| Already in the estate? | **Yes, if CloudFront already fronts the app** | **Yes** | **No — new service, new concept, new bill** |

### The argument

**Global Accelerator is the technically cleanest answer and the wrong one here.**
Its properties are genuinely excellent for this problem: two static anycast IPs
that never change, so there is no DNS cache to defeat; a traffic dial that is a
deliberate 0–100 number rather than a robot's opinion; and it works for protocols
CloudFront cannot carry. If the estate had no CDN and served raw TCP, this note
would recommend it.

It is the wrong one here for three reasons, in descending order of weight:

1. **The estate already has CloudFront in front of the application.** Global
   Accelerator would be a *fourth* layer (client → GA → CloudFront → ALB) or a
   parallel ingress path with its own WAF and TLS story. Adding a new global
   ingress service to solve a problem two existing global services already solve
   is the definition of the thing the brief warns against.
2. **The DT-Premium is a per-GB tax on all traffic, forever, to buy a failover
   property you can get for the price of a KVS write.** The $18.25/month
   accelerator fee is trivial; the data-transfer premium on production volume is
   not, and it is charged whether or not you ever fail over.
3. **It does not solve the hard part.** The hard part of this failover is not
   *moving traffic* — every one of these three does that inside the budget. It is
   *promoting the database, scaling the standby, and fencing the old primary*
   ([[aws-rds-postgres]], [[split-brain-and-fencing]]). Global Accelerator moves
   traffic faster than DNS and no faster than a KVS flip.

**Route 53 is the fallback, not the mechanism** — but it must stay wired up.
[[aws-route53]] measures the DNS layer at ~90 s of a 900 s budget and correctly
calls it "not the constraint". The reason not to lead with it is the one that note
also documents at length: **the ~90 s is what Route 53 does, not what your clients
do.** A JVM with `networkaddress.cache.ttl` negative caches the address for the
life of the process. CloudFront origin selection has *no such exposure*, because
the client's DNS answer never changes — `app.example.com` points at the same
distribution before and after. **That is the single strongest argument in this
note.**

**Recommendation, stated as a stack rather than a winner:**

> 1. **Primary mechanism: the CloudFront Function + KeyValueStore active-region
>    flag.** Covers all HTTP methods, no client DNS involvement, seconds, one API
>    call, optimistic-concurrency-guarded, no Terraform, no `us-east-1`
>    distribution deploy. **Verify the `POST` behaviour by test first.**
> 2. **Automatic safety net underneath it: a static origin group with failover
>    criteria `[500, 502, 503, 504]`.** Free, covers the read path in the first
>    seconds before anyone is paged, and turns a hard outage into a latency
>    incident.
> 3. **Keep Route 53 failover records pre-created and dormant**, weighted 100/0
>    with **no health checks attached** so 0 genuinely means 0. This is the escape
>    hatch for the one case CloudFront cannot serve: **CloudFront itself is the
>    fault** (see [[#Case study — the 16 July 2026 CloudFront VPC Origins event]]).
>    It is the only mechanism in the list that can route *around* CloudFront.
> 4. **Do not adopt Global Accelerator for this.** Revisit only if a non-HTTP
>    protocol enters scope.
> 5. **Do not buy an ARC routing-control cluster to drive any of this.**
>    [[route53-application-recovery-controller]] reaches the same conclusion
>    independently and costs it at **$1,825/month for roughly 25 seconds saved on
>    a 900-second budget**. A KVS `put-key` is a data-plane write with an ETag
>    precondition — it delivers the "deliberate, human-gated, not-a-robot" property
>    that was the main reason to want routing controls, for cents. What ARC
>    *would* have added is **safety rules**, and those must now be rebuilt as
>    guard-rails in the failover tool. Consider **ARC Region switch at $70/plan/
>    month** for the orchestration around the flip, per that note's verdict.

### RPO / RTO analysis

**RPO: N/A.** CloudFront holds cache and configuration, not data. Losing an edge
cache loses nothing durable. The estate's RPO is set by [[aws-rds-postgres]],
[[aws-dynamodb]] and [[aws-s3]].

**RTO: the CloudFront layer contributes seconds, not minutes** — *if* the
mechanism is the KVS flip. Where the time goes:

| Step | Time | Pre-provisioned? |
|---|---|---|
| Human decides to fail over | 2–5 min | The real cost. Not a CloudFront number. |
| Pre-scale the standby (do this **before** the flip) | 1–5 min | [[aws-eks]] |
| Promote the database | minutes | [[aws-rds-postgres]] |
| `put-key active-region = standby` | **< 1 s** | |
| KVS propagates to all edges | **"a few seconds"**, no SLO | |
| Cache-miss requests now reach the standby directly | immediate | |
| **CloudFront's contribution** | **≈ 5–15 s** | |

Against the [[aws-route53]] DNS figure of ~90 s, CloudFront origin selection is
roughly **an order of magnitude faster and removes the client-cache tail
entirely**. **Verdict: `meets_targets: conditional` — yes as a mechanism; no if
the runbook's step is "edit the distribution", which is a `us-east-1`
control-plane call plus edge propagation and cannot be bounded.**

### Warm standby shape

What exists at the edge while the primary is healthy, and what it costs:

| Thing | State while primary is healthy | Idle cost |
|---|---|---|
| The distribution | One, serving | Usage-based only |
| Standby ALB as a second origin | **Configured and receiving zero traffic** | **$0** — origins are not billed for existing |
| Standby Origin Shield | Enabled, zero requests | **$0** — billed per request |
| CloudFront Function | Running on 100% of requests | $0.10/M invocations |
| KeyValueStore | One key, read on every request | $0.03/M reads |
| `us-east-1` ACM cert, WAF ACL | Exist | WAF has a real monthly fee; cert is free |

**The edge layer of the warm standby is essentially free.** All the standby cost
in this programme is compute and database, not CloudFront. Worth saying out loud
in [[cost-model]], because it means there is no cost argument against configuring
the standby origin *today*, long before the standby region can serve anything.

## Migration path from single-region

The good news, stated first: **nothing in this migration forces a resource
replacement, and nothing requires downtime.** Adding an origin, adding an origin
group, adding a function association and changing a cache behavior's
`target_origin_id` are all in-place `UpdateDistribution` operations. There is no
`ForceNew` here. Contrast [[aws-acm]] and [[aws-dynamodb]], where there is.

The one real trap is **ordering**, and it is a `terraform plan` trap rather than
an AWS one.

**Step 0 — before touching CloudFront.** Tune the *existing* origin's timeouts:
`origin_read_timeout` from 30 s down to something near your p99, and
`connection_attempts` from 3 to 2. Zero-risk, standalone, and per the 16 July 2026
log analysis this is the highest-value single edit in the note. Ship it alone.

**Step 1 — add the standby origin, referenced by nothing.** An origin that is not
the target of any cache behavior receives no traffic. It is inert configuration.
Add it as soon as the standby ALB exists, even if the standby has no running
pods.

```
# terraform plan should show exactly one in-place update:
#   ~ resource "aws_cloudfront_distribution" "this" {
#       + origin { origin_id = "alb-eu-west-2" ... }
```

**Step 2 — add the origin group. Still referenced by nothing.**

> [!warning] The deletion-order trap
> "If you want to delete an origin, you must first edit or delete the cache
> behaviors that are associated with that origin." The same dependency applies in
> reverse and Terraform does not always order it correctly: a single apply that
> both *removes an origin* and *repoints a cache behavior* can fail mid-apply with
> a distribution in a half-updated state. **Never combine an origin removal with a
> behavior change in one apply.** Split it. This is the one place this migration
> can actually break.

**Step 3 — repoint the default cache behavior at the origin group.** This is the
moment origin failover becomes live. Nothing changes for viewers: the group's
primary is the origin they were already using. What changes is that a 5xx from
the primary now falls through to a standby that may not be ready — so **do not do
step 3 until the standby can serve a 200**, or you will convert clean 5xxs into
5xxs from a second region plus latency.

**Step 4 — add Origin Shield to both origins.** Independent of the above, safe to
do any time, skipped entirely for the CA pair (see the warning above).

**Step 5 — create the KeyValueStore, seed `active-region = <primary>`, publish
the function with the flag hard-wired to primary behaviour, and associate it.**
At this point the function runs on every request and selects the origin it was
already going to select. **This is the step to soak.** Watch the
`FunctionExecutionErrors` and `FunctionComputeUtilization` CloudWatch metrics for
a week before anyone believes the switch works.

**Step 6 — test the flip in a non-production environment, end to end, with
`curl -X POST`**, using the exact credentials the runbook will use (remember the
SigV4A/STS-v2 trap). **A failover mechanism that has never been executed is not a
failover mechanism.**

**Step 7 — add `ignore_changes = [value]` to the KVS key resource** — ideally in
the same commit that creates it, so there is never a window where an apply can
fail you back.

> [!note] Continuous deployment as a safer path for steps 3–5
> CloudFront supports **staging distributions** (20 per account) and continuous
> deployment policies, which let you send a small percentage of real traffic to a
> modified configuration. If the team is nervous about putting a function on 100%
> of viewer requests, this is the lower-risk route. Note the documented
> incompatibility: **"Response completion timeout doesn't support the continuous
> deployment feature."** You cannot use both.

## Failover procedure

Assumes the KVS mechanism from [[#Edge functions as the real failover switch]].
Times are the CloudFront-layer contribution only.

**Pre-flight (done in advance, verified quarterly — not at 3am):**

- `KVS_ARN` is written in the runbook as a **literal string**, not discovered via
  an API call. Same reasoning as [[route53-application-recovery-controller]]'s
  "bookmark or hard code your cluster endpoints": discovery during an incident is
  a dependency on the thing that is broken.
- The failover credentials have been proven to work against the KeyValueStore
  API, **with regional STS endpoints or account-level v2 tokens**.
- Both origins return 200 to a synthetic canary.

**The steps, in this order, and the order matters:**

1. **Confirm it is a regional event, not a bad deploy.** A deploy rollback is
   faster and lossless. Failing over for a bad deploy means accepting up to 2 h of
   data loss to fix something a `kubectl rollout undo` would have fixed.
2. **Decide to accept the RPO.** Human. Named. Logged. This is the expensive step
   and no tooling shortens it.
3. **Pre-scale the standby.** *Before* the flip, not after. Otherwise you have
   built the thundering herd on purpose.
4. **Promote the database and FENCE THE OLD PRIMARY.** See
   [[split-brain-and-fencing]]. The KVS propagation window means some edges will
   keep sending writes to the old region for seconds after the flip. **The fence,
   not the flip, is what makes that safe.**
5. **Flip the flag** — the single `put-key` with `--if-match`.
6. **Verify from outside.** `curl` the public hostname and assert on a response
   header identifying the serving region. Add one: a `Server-Region` response
   header set by the ALB is the cheapest failover observability in the estate, and
   it lets you watch propagation happen rather than assume it.
7. **Watch `OriginLatency` p95 and origin 5xx on the standby.** If the standby is
   browning out, step 3 was insufficient — scale further; do not flip back into a
   region you have just fenced.
8. **Do not run Terraform.** Not to fail over, not to check, not at all.

**What is automated:** the origin-group read-path failover (already happening,
automatically, before step 1). **What is a human decision:** everything else.
That asymmetry is deliberate and correct for a 2-hour-RPO async replica.

## Failback

Harder than failover, as always, and CloudFront adds one twist that is easy to
miss.

**The twist: the static origin group fails back on its own, instantly, with no
damping.** Recall the verbatim behaviour — CloudFront routes *all* requests to the
origin-group primary and only uses the secondary after a failure. So the moment
the old primary starts returning 200 again, **it starts serving cache misses
again**, regardless of whether its database is still the authoritative one.

> [!danger] The automatic failback is a split-brain generator
> If you failed over by flipping the KVS flag, the function reorders the group
> and this is fine — the promoted region is now the group's primary. **If you
> failed over by doing nothing and letting the origin group handle it**, then the
> instant the old region recovers, traffic returns to a database that has been
> stale since the incident began, while writes have been landing in the other
> region. There is no soak period and no manual gate.
>
> **This is the strongest single argument for flipping the flag rather than
> relying on the origin group**, even for a read-only workload.

**The failback procedure:**

1. **Rebuild replication in the opposite direction** and let it catch up. This is
   the long pole and it is not a CloudFront problem — see [[aws-rds-postgres]].
   Days, not minutes.
2. **Un-fence the old primary and re-fence the current one.** Same mechanism,
   opposite direction.
3. **Flip the KVS key back.** One command. Symmetric with failover *by design* —
   the `var.regions` map in the Terraform is deliberately unordered so there is no
   "failback code path" to be wrong.
4. **Do a planned failback in business hours**, not as the tail end of an
   incident at 5am. The whole point of a deliberate switch is that you get to
   choose when.
5. **Then, and only then, reconcile Terraform.** If `default_active_region` should
   change permanently, change it in a normal PR. Remember that `ignore_changes`
   means the KVS value will *not* be corrected by apply — which is the desired
   behaviour and also means **Terraform's view and reality can legitimately
   diverge.** Document that in the module README so the next engineer does not
   "fix" it.

**Cache warmth on failback:** the old primary's Origin Shield and regional edge
caches will have aged out. Failing back is a second thundering herd, in the other
direction, against a region that has been idle. Pre-scale before failing back too.

## Gotchas

The list that makes this note worth reading.

1. **`503` and `504` must be in the origin group's failover criteria or failover
   does not happen for a dead region.** "CloudFront fails to connect to the
   primary origin (**when 503 is set as a failover code**)". The most common
   misconfiguration and the one that silently nullifies the whole feature.
2. **`403`/`404` in the failover criteria doubles your origin traffic on every
   bad request** and masks S3/OAC permission faults by quietly serving from the
   other region.
3. **Origin failover is `GET`/`HEAD`/`OPTIONS` only.** No setting changes it.
4. **`OPTIONS` must be in *cached* methods**, not just allowed methods, or CORS
   preflights never fail over. A failover where the API works and the browser app
   is broken.
5. **CloudFront has no circuit breaker.** Every cache miss pays the primary
   timeout again, for the whole duration of the outage. Measured at p95 ≈ 30.3 s
   in the 16 July 2026 logs.
6. **A hung origin costs up to 90 s for a `GET`** (30 s response timeout × 3
   connection attempts), because response timeouts are retried per connection
   attempt for `GET`/`HEAD` — and are **not** retried for `POST`/`PUT`/etc.
7. **Lambda@Edge origin-request/response triggers fire twice on a failed-over
   request.** Anything non-idempotent runs twice.
8. **Enabling Origin Shield relocates Lambda@Edge origin-facing triggers to the
   Origin Shield region.** IAM and KMS grants in the old region will not follow.
   Cross-ref [[aws-kms]] — KMS grants and key policies are regional and this is
   exactly the class of failure that only appears at the edge.
9. **Origin Shield exists in neither `ca-central-1` nor `ca-west-1`**, and AWS's
   fallback for `ca-central-1` is `us-east-1` — a data-residency decision
   disguised as a caching setting. `ca-west-1` has no documented fallback at all.
10. **The KeyValueStore API needs SigV4A and version-2 STS session tokens.**
    Global STS endpoint → authentication error → your failover command fails for
    a reason that looks unrelated. Test the credentials in advance.
11. **There are two different `DescribeKeyValueStore` operations with two
    different, non-interchangeable ETags.** Use the wrong one in the failover
    script and the write fails.
12. **`kvs.get()` has no `default` option and throws on a missing key.** The only
    option is `format`. A function without a `try`/`catch` around it returns 5xx
    to every viewer the moment someone deletes the key.
13. **`Promise.all` in a CloudFront Function can exceed the 2 MB memory quota**
    and fail the request. Use sequential `await`.
14. **One key value store per function. Hard quota, not adjustable.** Plan the
    key namespace accordingly; you will not get a second store for the same
    function.
15. **`cf.updateRequestOrigin()` fails on VPC origins.** If the estate moves to
    VPC Origins, `selectRequestOriginById()` and `createRequestOriginGroup()` are
    the only edge-side options.
16. **`createRequestOriginGroup()`'s `failoverCriteria` overwrites, not merges:**
    "this array will overwrite all failover status codes that are set in the
    origin group's original configuration." Omit `504` in the function and you
    have silently removed it, no matter what Terraform says.
17. **The origin ID strings are an undeclared interface between HCL and
    JavaScript.** Rename an origin in Terraform, and the function's
    `selectRequestOriginById("alb-eu-west-1")` starts failing at the edge with no
    plan-time error. Render them into the function with `templatefile`.
18. **`terraform apply` can fail you back.** Without `ignore_changes = [value]` on
    the KVS key, any routine apply resets the active region to whatever
    `default_active_region` says. **This is the single most dangerous line of
    Terraform in the design.**
19. **Never combine removing an origin with repointing a cache behavior in one
    apply.** Split into two.
20. **Everything about the distribution is a `us-east-1` control-plane
    operation** — `UpdateDistribution`, `CreateInvalidation`, the ACM cert, the
    `CLOUDFRONT`-scope WAF ACL. For the US pair, `us-east-1` is the *primary*.
    A plan that says "edit the distribution to fail over" depends on the region it
    is failing away from.
21. **CloudFront itself is a shared fate.** 16 July 2026. No number of regions
    helps.
22. **Origin Shield is not supported with gRPC** — "the requests will be proxied
    directly to the gRPC origin without going through Origin Shield." If any
    service is gRPC-over-CloudFront, it gets no request collapsing.
23. **Response completion timeout is incompatible with continuous deployment.**
    Pick one.
24. **CloudFront Functions cannot see the request body and cannot make network
    calls.** The switch can never be self-healing, by design. It reads a flag
    someone set. (Argue this is a feature; see
    [[#The flip is a single API call, and its blast radius is one key]].)
25. **KVS helper calls do not appear in CloudTrail.** "Key value store helper
    method calls from CloudFront Functions don't trigger an AWS CloudTrail data
    event." You can audit *who flipped the flag* (the write API), but not *what
    the edge read*. Fine, but know it before an auditor asks.

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| **Primary failover mechanism** | CloudFront origin group only | CloudFront Function + KVS origin selection | **B, with A underneath as the automatic safety net.** A alone drops every write for the whole outage. |
| **Is the write path in scope at all?** | The distribution fronts only static/read content, so `GET`-only failover is complete | The distribution fronts a write-capable API | **Answer this first — it is the fork the whole note turns on.** Assume B until someone confirms A. |
| **What triggers the flip** | Automatic, from a CloudWatch alarm via Lambda | Human decision, runbook, one command | **B.** RPO 2h means failover is a data-loss decision. Automating it automates data loss. Same conclusion [[aws-route53]] reaches against failover routing. |
| **Origin timeouts** | Leave defaults (30 s read, 3 attempts) | Tune to p99 (~10 s, 2 attempts) | **B, and ship it this week independently of everything else.** The 16 July 2026 logs show the default *is* the user experience. |
| **Origin Shield** | On, per-origin, in each origin's own region | Off | **On for EU and US pairs. Off for the CA pair** — no supported region, and the `us-east-1` fallback is a residency breach. |
| **VPC Origins vs public ALB origins** | VPC Origins (private ALB, cleaner security) | Public ALB + shared-secret header + WAF | **B for now.** 16 July 2026 showed a VPC-Origins-specific global failure with "change the origin type" as the workaround, and `updateRequestOrigin()` cannot touch VPC origins. Revisit when the feature has more track record. Genuinely close — A is the better security posture. |
| **Global Accelerator** | Adopt as the failover mechanism | Don't | **Don't.** ~$18.25/mo plus a per-GB premium on all traffic, to buy a property CloudFront already gives you. Revisit if non-HTTP protocols enter scope. |
| **Route 53 records** | Remove, CloudFront handles it | Keep pre-created, weighted 100/0, no health checks | **B.** The only mechanism that can route *around* CloudFront when CloudFront is the fault. Costs nothing dormant. |
| **ARC routing-control cluster to drive the flip** | Buy it | Use the KVS ETag write as the deliberate switch | **B.** Agrees with [[route53-application-recovery-controller]]. Rebuild ARC's *safety rules* as guard-rails in the failover script — that is the real thing you give up. |
| **Second CDN for CloudFront-level failures** | Add one | Accept the risk, document it | **Accept and document.** The cost and complexity are large; the event class is rare; and the risk register should say so explicitly rather than implying multi-region covers it. |

## Cost

All figures from the [CloudFront pay-as-you-go pricing page](https://aws.amazon.com/cloudfront/pricing/pay-as-you-go/), verified rather than estimated. Regional variation applies; Europe and US shown.

| Item | Price | What it means here |
|---|---|---|
| **A second origin on the distribution** | **$0** | Origins are not billed for existing. **The standby origin can be configured today, free, before the standby can serve anything.** |
| **An origin group** | **$0** | |
| **CloudFront Functions** | **$0.10 per 1 M invocations**, first **2 M/month free** | Viewer-request means *every* request, including cache hits. At 500 M requests/month: **$50/month**. |
| **KeyValueStore reads** | **$0.03 per 1 M reads** | One read per request. At 500 M/month: **$15/month**. |
| **KeyValueStore writes / other API** | **$1 per 1,000 API requests** | A failover is a handful of calls. **Effectively $0** — but note it is ~$0.001 per call, so do not build a polling loop against it. |
| **Origin Shield** | **$0.0075 / 10,000 requests (US)**, **$0.0090 (Europe)** | Billed on requests that reach the shield as an incremental layer. **Zero while the standby is idle.** |
| **Lambda@Edge (the alternative)** | **$0.60 per 1 M requests** + $0.00005001/GB-s, **no free tier** | **6× the per-request price of CloudFront Functions** — but it runs only on cache misses. The crossover depends entirely on cache hit ratio. |
| Invalidations | First 1,000 paths/month free, then $0.005/path | Not a failover cost. |

**Two cost levers worth knowing:**

1. **CloudFront Functions vs Lambda@Edge is a cache-hit-ratio calculation, not a
   preference.** CloudFront Functions cost $0.10/M on *all* requests; Lambda@Edge
   costs $0.60/M on *cache misses only*. At a 90% hit ratio, Lambda@Edge is
   cheaper per unit of work — but it cannot do viewer-request origin selection,
   so for the write path it is not an option at any price. Model it in
   [[cost-model]] with real request volumes.
2. **Origin Shield billing punishes short TTLs.** Verbatim: "`GET` and `HEAD`
   requests that have a time to live (TTL) setting of less than 3600 seconds are
   considered dynamic requests", and dynamic requests are *always* an incremental
   billed layer. A write-heavy API with short TTLs pays Origin Shield on close to
   100% of traffic while getting relatively little collapsing benefit. **For a
   dynamic API, Origin Shield may not pay for itself** — AWS says as much: "Origin
   Shield may not be a good fit in other cases, such as dynamic content that is
   proxied to the origin, content with low cacheability, or content that is
   infrequently requested." Enable it where the content is cacheable; be
   sceptical on a pure API behaviour.

**Bottom line: the CloudFront layer of this programme costs tens of dollars a
month, not thousands.** It is the cheapest failover mechanism available to the
estate by a wide margin — cheaper than Global Accelerator's DT-Premium and three
orders of magnitude cheaper than an ARC routing-control cluster.

## Open questions

Things needing an answer from inside the company, or a test.

1. **Does the distribution front a write-capable API, or only static/read
   content?** [[#The GET/HEAD/OPTIONS limitation — the decision point]] turns
   entirely on this. Everything else in the note assumes "yes, writes".
2. **Does viewer-request origin selection actually apply to `POST`?** AWS
   documents no method restriction, in contrast to the explicit one on origin
   failover — but does not state it positively either. **Test it: two origins,
   one function, one `curl -X POST`.** Half an hour. Do it before this design is
   approved.
3. **Partly answered.** `aws_cloudfront_key_value_store`,
   `aws_cloudfrontkeyvaluestore_key` (arguments `key_value_store_arn`, `key`,
   `value`) and `key_value_store_associations` on `aws_cloudfront_function` are
   all **verified against the `hashicorp/aws` registry docs**. Still open:
   **which provider version is pinned in the monorepo**, and whether it exposes
   **`response_completion_timeout`** on the origin block — the one attribute in
   [[#Terraform implementation]] not confirmed against a release.
4. **Does the estate use, or plan to use, CloudFront VPC Origins?** Changes the
   16 July 2026 exposure, and removes `cf.updateRequestOrigin()` as an option.
5. **Is there one hostname per region pair or one global hostname?**
   [[aws-route53]] raises the same question. If geolocation routing steers
   EU/US/CA customers to three deployments, the CloudFront design needs to be
   three distributions (one per pair), not one — and failover must stay *inside*
   a pair for [[data-residency]] reasons.
6. **Is Origin Shield in `us-east-1` for a `ca-central-1` origin acceptable to
   legal?** If not — and it probably is not — the CA pair runs without Origin
   Shield and needs more standby headroom instead.
7. ~~**Does CloudFront honour `stale-while-revalidate`?**~~ **Answered: yes,
   and `stale-if-error` too, since 17 May 2023.** The open question that
   replaces it is narrower and is for the application teams: **what is
   `max_ttl` on the current cache policies?** `stale-if-error` is capped by it,
   so a large `stale-if-error` against a small `max_ttl` is a no-op. Audit the
   cache policies before promising this mitigation to anyone.
8. **Are the failover credentials SigV4A-capable?** Specifically: does the CI
   role or the break-glass role obtain tokens from a *regional* STS endpoint?
   Cross-ref [[aws-iam]].
9. **Is `us-east-1` the right primary for the US pair given that it also hosts
   the CloudFront, Route 53, ACM-for-CloudFront and WAF-CLOUDFRONT control
   planes?** Raised in [[aws-route53]] and [[aws-acm]]; it is a
   [[region-pair-selection]] question and this note adds a fourth vote to it.
10. **Who is allowed to flip the flag?** The IAM policy for
    `cloudfront-keyvaluestore:PutKey` on one ARN is the entire authorisation
    surface for production failover. That is elegantly small — and it needs a
    named owner. [[failover-orchestration]].

## Sources

Every URL below was fetched and read during the writing of this note. Verbatim
quotes in the body are from these pages.

**CloudFront origin failover and origins**
- [Optimize high availability with CloudFront origin failover](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/high_availability_origin_failover.html) — the primary source for this note. Origin group behaviour, the failover status-code list (400/403/404/416/429/500/502/503/504), the `GET`/`HEAD`/`OPTIONS` restriction verbatim (stated twice), "CloudFront routes all incoming requests to the primary origin, even when a previous request failed over", the 30 s default (3 × 10 s) connection budget, the `OPTIONS`-must-be-a-cached-method note, and Lambda@Edge firing twice.
- [Origin settings — all distribution settings reference](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesOrigin.html) — Connection attempts (1–3, default 3), Connection timeout (1–10 s, default 10), Response timeout (1–120 s, default 30 s) and the crucial split in retry behaviour between `GET`/`HEAD` (retried per connection attempts) and `DELETE`/`OPTIONS`/`PATCH`/`PUT`/`POST` (never retried); Response completion timeout as the hard ceiling; keep-alive timeout default 5 s.

**Edge functions and origin selection**
- [Helper methods for origin modification](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/helper-functions-origin-modification.html) — **the key source for the failover mechanism.** Verifies `cf.selectRequestOriginById()`, `cf.createRequestOriginGroup()` and `cf.updateRequestOrigin()` by name; "You can update the origin on *viewer request* CloudFront Functions only"; runtime 2.0 and `import cf from 'cloudfront'` required; `updateRequestOrigin()` fails on VPC origins; `createRequestOriginGroup()` takes `originIds` and a required `failoverCriteria.statusCodes`; the CloudFront Functions vs Lambda@Edge trade-off ("runs on every request" vs "runs only on cache misses").
- [Helper methods for key value stores](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/functions-custom-methods.html) — `cf.kvs()`, `get()`/`exists()`/`meta()`, the `format` option (there is **no** `default` option), `get()` throws on a missing key, and the warning that promise combinators can blow the function memory quota.
- [JavaScript runtime 2.0 features for CloudFront Functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/functions-javascript-runtime-20.html) — `async`/`await` support (runtime 2.0 only); and the restricted-features list: no network access, no timers, no environment variables, no `eval`, `Date` frozen at function start.
- [Restrictions on CloudFront Functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-function-restrictions.html) — **the SigV4A / version-2-session-token trap** on the KeyValueStore API verbatim, plus "CloudFront Functions can't access the body of the HTTP request" and the compute-utilization limit.

**CloudFront KeyValueStore**
- [Amazon CloudFront KeyValueStore](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/kvs-with-functions.html) — read-only from functions; "makes it easy to update data without the need to deploy code changes"; runtime 2.0 required.
- [Work with key value data](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/kvs-with-functions-kvp.html) — the consistency statement ("Updates are processed in between invocations of a function"), the mandatory `--if-match` ETag on writes, the two non-interchangeable `DescribeKeyValueStore` ETags, and the `put-key`/`update-keys` CLI shapes used in the runbook.
- [Introducing Amazon CloudFront KeyValueStore (AWS News Blog)](https://aws.amazon.com/blogs/aws/introducing-amazon-cloudfront-keyvaluestore-a-low-latency-datastore-for-cloudfront-functions/) — **the only source for "changes are propagated to all CloudFront edge locations in a few seconds"**, which is the number the TL;DR rests on. A blog characterisation, not an SLO.
- [CloudFront quotas](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-limits.html) — origin groups per distribution (10), origins per distribution (100), connection attempts (1–3), connection timeout (1–10 s), response timeout (1–120 s), keep-alive timeout (1–300 s), CloudFront Functions max size 10 KB / max memory 2 MB, KVS key 512 B / value 1 KB / store 5 MB / **1 key value store per function** / 100 functions per store / 200 stores per account.

**Dynamic origin modification in practice**
- [Part 2: Implementing dynamic origin modification in Amazon CloudFront](https://aws.amazon.com/blogs/networking-and-content-delivery/part-2-implementing-dynamic-origin-modification-in-amazon-cloudfront) — AWS Networking & Content Delivery blog, 10 February 2026, Nikhil Patne, Aish Gopalan and Deepak Garg. Worked example of KVS-driven origin selection at viewer request. **Note: its use case is subscription-tier routing, not failover** — AWS has not published a DR-flavoured version of this pattern, which is a gap worth knowing about.

**Origin Shield and caching**
- [Use Amazon CloudFront Origin Shield](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/origin-shield.html) — request collapsing ("resulting in as few as one request going to your origin"), "Origin Shield is a property of the origin", the origin-group interaction (each origin goes through *its own* Origin Shield), **the full list of Origin Shield regions — which excludes both `ca-central-1` and `ca-west-1`** — and the fallback table mapping `ca-central-1` to `us-east-1`. Also Origin Shield's own failover to a secondary shield location, the Lambda@Edge origin-facing-trigger region shift, `OriginShieldHit` in `x-edge-detailed-result-type`, and the dynamic-vs-cacheable billing rule (`PUT`/`POST`/`PATCH`/`DELETE`, and `GET`/`HEAD` with TTL < 3600 s, are always billed as an incremental layer).

**Pricing**
- [Amazon CloudFront pay-as-you-go pricing](https://aws.amazon.com/cloudfront/pricing/pay-as-you-go/) — Origin Shield $0.0075 per 10,000 requests (US) / $0.0090 (Europe); CloudFront Functions $0.10 per 1 million invocations with 2 million free per month; KeyValueStore **$0.03 per 1 million reads** and **$1 per 1,000 other API requests**; Lambda@Edge $0.60 per 1 million requests plus $0.00005001 per GB-second with no free tier; invalidations free for the first 1,000 paths/month then $0.005 per path.

**The 16 July 2026 VPC Origins event**
- [AWS CloudFront outage serves errors instead of websites](https://www.theregister.com/off-prem/2026/07/16/aws-cloudfront-outage-serves-errors-instead-of-websites/5272421) — *The Register*, 16 July 2026. **The most credible source for this event.** First-hand, same-day, and quotes AWS's Service Health Dashboard updates verbatim including the root-cause statement about "an internal constraint on the fleet that manages connections to private VPC origins". Gives a start of 0145 PDT and recovery by 1739 UTC (~8 h), which **conflicts with the ~3 h 33 m figure** repeated by aggregator sites.
- [CloudFront VPC Origin Outage (2026-07-16): Confirming from logs that approximately 30-second timeout waits were occurring](https://dev.classmethod.jp/en/articles/cloudfront-vpc-origin-incident-20260716-log-analysis/) — Suzuki Ryo, Classmethod DevelopersIO, 21 July 2026. **The best technical source on this event and the empirical backing for this note's timeout argument.** Access-log analysis of a real affected distribution: p95 `time-taken` settling at ~30.3 s matching `OriginReadTimeout`, `ClientCommError` at 89.3% of results, and the observation that origin failover worked but "the secondary was attempted only after the timeout elapsed".
- **Not found: a first-party AWS post-event summary.** Searched for an `aws.amazon.com/message/...` page and for an AWS Health Dashboard archive for this event; neither surfaced. AWS's statements survive only as third-party quotations of an ephemeral status page. This is a real gap and the note says so rather than implying first-party sourcing.

**Serving stale content**
- [Amazon CloudFront now supports stale-while-revalidate and stale-if-error cache control directives](https://aws.amazon.com/about-aws/whats-new/2023/05/amazon-cloudfront-stale-while-revalidate-stale-if-error-cache-control-directives/) — AWS What's New, 17 May 2023. Confirms both directives are honoured. `stale-if-error` is the cheapest thundering-herd mitigation in this note and is an origin response header, not a CloudFront setting.

**Terraform provider**
- [`aws_cloudfront_key_value_store`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudfront_key_value_store) and [`aws_cloudfrontkeyvaluestore_key`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudfrontkeyvaluestore_key) — verifies the resource names (note the run-together `cloudfrontkeyvaluestore`) and the required arguments `key_value_store_arn` / `key` / `value`.
- [`aws_cloudfront_function` resource docs (provider source)](https://raw.githubusercontent.com/hashicorp/terraform-provider-aws/main/website/docs/r/cloudfront_function.html.markdown) — verifies `key_value_store_associations` ("AWS limits associations to one key value store per function"), the `runtime` values `cloudfront-js-1.0` / `cloudfront-js-2.0`, and `publish` defaulting to `true`.

**The alternatives**
- [Use traffic dials to adjust traffic flow to Regions (AWS Global Accelerator)](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoint-groups-traffic-dial.html) — the traffic dial as a 0–100 deliberate switch, and the caveat that "when you change a traffic dial, the updated setting applies to only new connections. Existing connections are not terminated."
- [AWS Global Accelerator pricing](https://aws.amazon.com/global-accelerator/pricing/) — **$0.025 per accelerator per hour (≈ $18.25/month)** plus DT-Premium of $0.007–$0.105/GB depending on source region and destination edge. The per-GB premium, not the hourly fee, is why this note recommends against adopting it here.

**Fault isolation**
- [AWS Fault Isolation Boundaries — Appendix B, edge network and global services](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/appendix-b---edge-network-global-service-guidance.html) — places CloudFront in the global-services fault-isolation category alongside Route 53, IAM and Global Accelerator.
