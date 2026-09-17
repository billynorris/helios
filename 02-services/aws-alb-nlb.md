---
title: ALB / NLB (Elastic Load Balancing) — Multi-Region
service: elbv2
tags: [service, multi-region, alb, nlb, elb, load-balancing, global-accelerator, edge]
status: researched
replication: none — strictly regional, deploy a second copy
rpo_achievable: N/A — stateless
rto_achievable: "< 1 min if pre-provisioned AND pre-warmed; 10-30 min of degraded capacity if not"
meets_targets: conditional — passes only if the standby ALB exists permanently and its cold-start scaling is addressed
updated: 2026-09-16
---

# ALB / NLB (Elastic Load Balancing) — Multi-Region

## TL;DR

- **Load balancers are strictly regional and there is no cross-region anything.** No replication, no global ALB, no way to reference an `eu-west-1` target group from `eu-west-2`. The only option is "deploy a second copy", and for a stateless service that is the correct answer, not a compromise.
- **The standby ALB must exist permanently — not be created at failover.** Not for creation speed (an ALB provisions in a couple of minutes) but for three compounding reasons: its DNS name and zone ID are inputs to your Route 53 records, its attached ACM certificate stops auto-renewing if it is detached ([[aws-acm]]), and a brand-new ALB has the smallest possible scaling footprint at exactly the moment you need the largest.
- **Idle cost is small and everyone over-estimates it.** An ALB is **$0.0225/hour** = **~$16.43/month** plus near-zero LCUs at zero traffic. Six standby load balancers across three pairs is about **$99/month**. This is not where the DR budget goes. Do not delete standby ALBs to save money — you will lose far more than $16 when the certificate silently expires.
- **Pre-warming is the real RTO threat, and the answer has changed.** The old answer was "open a support case". The current answer is **LCU Reservation** (GA November 2024): self-service, API-driven, reserve a minimum capacity floor. **Minimum reservation is 100 LCU**, which at $0.008/LCU-hour derives to **~$584/month per load balancer** — 35× the idle ALB cost. This is the genuine cost decision in this note. See [[#Pre-warming]].
- **The thing that will bite:** you will do everything right and still discover that **Global Accelerator would have been simpler**. Anycast IPs, no DNS TTL, failover in ~30 seconds, `$0.025/hour` per accelerator. It is not obviously the right answer — DT-Premium data charges are real — but it deserves a serious head-to-head, which this note gives it. See [[#Global Accelerator vs Route 53 failover]].

## Does this service cross regions at all?

No, in every sense that matters:

| Question | Answer |
|---|---|
| Can one ALB serve targets in two regions? | **No.** Targets must be in the load balancer's VPC, or reachable IPs — and cross-region IP targets are not supported. |
| Can a Route 53 alias in one region point at an ALB in another? | **Yes** — Route 53 is global, the alias just needs the right `zone_id`. This is the *only* cross-region relationship. |
| Is there a "global load balancer"? | **Not in ELB.** [[#Global Accelerator]] is the closest thing and it is a separate service with a separate control plane. |
| Can the certificate be shared? | **No.** Regional ACM cert per region. See [[aws-acm]]. |
| Is the DNS name stable across a rebuild? | **No.** Delete and recreate an ALB and you get a new `*.elb.amazonaws.com` name and new IPs. Another argument for permanence. |

The correct mental model: **the load balancer is part of the regional stack, like the VPC.** It gets templated, it gets deployed twice, and the multi-region work happens entirely in [[aws-route53]] or Global Accelerator above it.

That makes this one of the easier notes in the vault — until you get to capacity.

## Replication / mirroring options

### Option 1 — Deploy a second copy (the only real option, and it is fine)

The ALB is a stateless config object. There is genuinely nothing to replicate. What has to be *mirrored correctly* is the configuration, and the failure mode is **drift**, not replication lag:

- listener rules (path/host routing, weighted forward actions, fixed responses),
- target group attributes (`deregistration_delay`, stickiness, `slow_start`, health check settings),
- security groups (including the Route 53 health-checker prefix list — see [[aws-route53]]),
- WAF web ACL association,
- access log bucket, **which must be in the same region as the load balancer**,
- `client_keep_alive`, `idle_timeout`, `desync_mitigation_mode`, `routing.http.drop_invalid_header_fields`.

**The mitigation is that the second copy comes from the same module, driven by the same variables, with only the provider alias and a handful of region-scoped inputs differing.** In a cookiecutter monorepo that is nearly free. The estate should treat any per-region override of an ALB attribute as a smell requiring justification, and CI should assert the two configs are structurally identical.

> [!warning] Listener-rule drift is the silent killer
> Listener rules are frequently added by hand during incidents or by a CD pipeline that only targets the live region. Six months later the standby's rule set is missing three paths. **Nothing detects this** — the ALB is healthy, the certificate is valid, the targets pass health checks, and 12% of your API 404s the moment you fail over. Add a drift check: compare `describe-rules` output between regions weekly and alarm on difference. This belongs in [[failover-runbooks]] as a pre-flight, not a failover step.

### Option 2 — Create the ALB at failover time

Do not. The provisioning call returns in seconds and the ALB is usable in a couple of minutes, so it *looks* like it fits inside 15 minutes. It does not, because:

1. The new ALB has a **new DNS name**, so the Route 53 record change is now mandatory and you are back in the `us-east-1` control plane during an outage ([[aws-route53]]).
2. The certificate wasn't attached to anything, so per [[aws-acm]] it was **not eligible for managed renewal** and may have expired.
3. It starts at minimum capacity. See below.
4. It requires a `terraform apply` in your recovery path, which is the pattern [[research-brief]] rules out.

### Option 3 — NLB instead of ALB for the standby

Occasionally proposed on the theory that an NLB "scales instantly". It does scale differently, and NLB has genuinely better burst behaviour (800 new TCP connections/second per NLCU vs 25 new connections/second per LCU for ALB — see [[#Cost]]). But you lose path-based routing, host-based routing, WAF, OIDC auth, and HTTP-level observability. **Making the standby a different shape from the primary is the worst possible idea in a DR architecture**: you will have tested neither. If NLB is right, it is right in both regions.

## RPO / RTO analysis

**RPO: N/A.** No data.

**RTO:** the load balancer contributes ~0 seconds *if* it is pre-provisioned, and an unbounded amount of *degraded service* if it is not warm. Splitting that honestly:

| Phase | Pre-provisioned | Created at failover |
|---|---|---|
| ALB exists, DNS name known | 0 s | ~2–4 min (plus the Terraform run around it) |
| HTTPS listener with valid in-region cert | 0 s | 0 s if cert pre-exists; **up to 30 min–72 h if not** ([[aws-acm]]) |
| Targets registered and passing health checks | **depends on the standby's compute** — see below | same |
| Capacity to absorb 100% of production traffic | **0 s only with an LCU reservation** | **10s of minutes** |

### The target-registration window is not free

Even with a permanent ALB, targets have to be *healthy* before the ALB will route to them. With ALB target-group defaults:

- `HealthCheckIntervalSeconds` default **30 s**
- `HealthyThresholdCount` default **5**

**5 × 30 s = 150 seconds** before a newly-registered target is in service. If the warm standby runs its EKS deployments scaled to zero and scales up at failover, you pay pod start + 150 s of health checking before that pod takes traffic. **Tune these down in the standby** (`interval = 10`, `healthy_threshold = 2` → 20 s) or, better, keep a non-zero baseline of pods running so the registration has already happened. [[aws-eks]] owns that decision; this note owns the 150-second number.

Note the asymmetry in the defaults: `UnhealthyThresholdCount` defaults to **2** while `HealthyThresholdCount` defaults to **5**. ELB is fast to eject and slow to admit. Correct for steady state, wrong for a scale-from-zero failover.

### Fail-open, and why it matters here

From the [ALB target group health check docs](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html), verbatim:

> If a target group contains only unhealthy registered targets, the load balancer routes requests to all those targets, regardless of their health status. This means that if all targets fail health checks at the same time in all enabled Availability Zones, the load balancer fails open.

Failing open is usually a kindness. In a failover it is a **trap for your monitoring**: a standby whose targets are all unhealthy still *accepts and forwards* requests, so the ALB's own metrics look like it is serving. Alarm on `HealthyHostCount`, not on `RequestCount`.

## Pre-warming

This is the part of the note that directly threatens the 15-minute RTO, and the guidance found in most blog posts is out of date.

### The problem

An ALB is not a fixed-size box; it is a fleet of nodes that AWS scales based on observed traffic. An ALB that has served ~zero requests for six months is running at its minimum footprint. At failover you hand it 100% of production traffic in a single step function. The ALB *will* scale — but scaling takes minutes, and while it scales you get elevated latency, connection timeouts and `HTTP 503`/`Target.Timeout`, i.e. exactly the outage you failed over to avoid. **The standby is "up" and the customer experience is still broken.** That is an RTO failure by any honest definition.

This is the single most under-appreciated risk in a warm-standby design, because it is invisible in every test that ramps traffic gradually.

### The old answer: a support case

AWS's long-standing guidance was that for flash traffic, or load tests that can't ramp gradually, you contact AWS Support to **pre-warm** the load balancer, supplying the start/end dates, the expected request rate per second, and typical request/response sizes. This still exists as a mechanism.

**It is useless for unplanned DR.** You cannot open a support case at 03:12 and wait for a response as part of a 15-minute RTO. Pre-warming by support case only works for *planned* events — which does make it viable for a **scheduled failover test**, and you should use it for that.

### The current answer: LCU Reservation

Announced **20 November 2024** for both Application and Network Load Balancers. From the [AWS docs on capacity reservations](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/capacity-unit-reservation.html), verbatim:

> Load balancer Capacity Unit (LCU) reservations allow you to reserve a static minimum capacity for your load balancer. Application Load Balancers automatically scale to support detected workloads and meet capacity needs. When minimum capacity is configured, your load balancer continues scaling up or down based on the traffic received, but also prevents the capacity from going lower than the minimum capacity configured.

And the situations AWS lists for using it include, verbatim:

> You have an upcoming event that will have a sudden, unusual high traffic and want to ensure your load balancer can support the sudden traffic spike during the event.

> You are migrating workloads between load balancers and want to configure the destination to match the scale of the source.

That second bullet is, almost word for word, a regional failover.

**This is the answer to the pre-warming problem and it should be adopted.** It is self-service, API/Terraform-driven, and requires no human at AWS.

Sizing it — AWS gives a specific method:

> You can utilize the CloudWatch metric `PeakLCUs` to determine the level of capacity needed. The `PeakLCUs` metric accounts for peaks in your traffic pattern that the load balancer must scale across all scaling dimensions to support your workload. The `PeakLCUs` metric is different from the `ConsumedLCUs` metric, which only aggregates the billing dimensions of your traffic. Using the `PeakLCUs` metric is recommended to ensure your LCU reservation is adequate during load balancer scaling. When estimating capacity, use a per-minute `Sum` of `PeakLCUs`.

**So the standby's reservation should be sized from the primary's `PeakLCUs`, per minute, Sum.** That is a concrete, automatable input: a scheduled job reads the primary's `PeakLCUs` and sets the standby's reservation to match. Put it in [[cloudwatch-observability]] and [[failover-orchestration]].

Constraints, from the same page:

> The total reservation request must be at least 100 LCU. The maximum value is determined by the quotas for your account.

**The 100-LCU floor is the cost story.** At the standard $0.008 per LCU rate, 100 LCU × $0.008 × 730 h ≈ **$584/month per load balancer**, versus $16.43/month for the same ALB idle. Across three standby regions with one public ALB each, that is roughly **$1,750/month** to solve a problem that is invisible until the day it isn't.

> [!warning] Verify the reserved-LCU rate before committing this number to a budget
> The $584 figure above is **derived** from two separately-verified numbers — the 100-LCU minimum in the docs and $0.008/LCU on the [ELB pricing page](https://aws.amazon.com/elasticloadbalancing/pricing/) — not lifted from a single published line. The pricing page's LCU-reservation section describes the model as "the number of Load Balancer Capacity Units (LCU) reserved per minute, and additional number of LCUs used per minute beyond your reserved LCUs per hour". Confirm the exact reserved rate and region multiplier on the pricing page before anyone signs off. Raised in [[#Open questions]].

### The middle path, and the recommendation

You do not have to hold a reservation 24/7. **LCU reservation is a mutable attribute**, so it can be set *during* the failover — the reservation API call is fast and, critically, **regional to the standby**, so it does not depend on `us-east-1`.

| Approach | Idle cost | Risk |
|---|---|---|
| A: Permanent 100+ LCU reservation on the standby | ~$584/mo/LB | None. Capacity is there before you need it. |
| B: Set the reservation as failover step #1 | ~$16/mo/LB | Reservation provisioning is not instantaneous; you are racing the traffic. Adds an API call to the runbook. |
| C: Nothing | ~$16/mo/LB | The standby brownouts under the step-load. **This is the default and it is the one most estates are silently running.** |

**Recommendation: B for non-production, A for production**, at least for the EU and US pairs. Then — and this is the part that actually matters — **prove it**. Run a failover test that moves real production-scale traffic in one step and measure `TargetResponseTime`, `HTTPCode_ELB_5XX_Count` and `RejectedConnectionCount` on the standby. If B holds up in a real test, take the $584/month saving with evidence. If nobody ever runs that test, pay for A, because C is what you have by default.

> [!tip] Global Accelerator changes this calculus
> Traffic arriving via Global Accelerator still lands on the same ALB and still causes the same step-load. GA does **not** solve pre-warming. But GA's **traffic dial** lets you move traffic in increments — 10%, 25%, 50%, 100% — over a couple of minutes, which lets the ALB scale *with* the ramp. **That is a genuine, free alternative to an LCU reservation**, and it is one of the strongest arguments in GA's favour below.

## Target groups, health checks, deregistration, cross-zone

The regional mechanics, with the settings that actually differ in a standby.

### Health check settings (ALB target groups)

Verbatim defaults from the [ALB health check docs](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html):

| Setting | Range | Default |
|---|---|---|
| `HealthCheckIntervalSeconds` | 5–300 s | **30 s** (35 s for `lambda`) |
| `HealthCheckTimeoutSeconds` | 2–120 s | **5 s** (30 s for `lambda`) |
| `HealthyThresholdCount` | 2–10 | **5** |
| `UnhealthyThresholdCount` | 2–10 | **2** |
| `Matcher` | 200–499 | **200** |
| `HealthCheckPath` | — | `/` (`/AWS.ALB/healthcheck` for gRPC) |

In the standby, prefer `interval = 10`, `healthy_threshold = 2`, `unhealthy_threshold = 2`. That takes time-to-in-service from 150 s to 20 s at the cost of 3× the health check traffic — irrelevant load.

### Deregistration delay

`deregistration_delay.timeout_seconds` defaults to **300 seconds**. On failback (or on any rolling deploy in the standby) every target you remove holds the door open for five minutes. For failover that's mostly harmless; for **failback**, where you may be draining the standby deliberately, it is five minutes of the budget. Set it to something that matches your longest real request — typically 30–60 s — not the default.

### Cross-zone load balancing

- **ALB: on by default at the load balancer level and cannot be turned off there**, but *can* be disabled per target group (`load_balancing.cross_zone.enabled`). Cross-zone traffic on an ALB does **not** attract inter-AZ data transfer charges.
- **NLB: off by default.** Enable it with `load_balancing.cross_zone.enabled = true` — and note that NLB cross-zone traffic **does** attract inter-AZ data transfer charges. This is a real bill in a busy region.

Relevance to a warm standby: with cross-zone **off** on an NLB and compute scaled down to one AZ, clients resolving to the other two AZs' NLB IPs get nothing. **Turn cross-zone on for NLBs in the standby**, or ensure targets exist in every enabled AZ. This is an easy, quiet, total failure of a standby.

There is a second-order effect specific to Global Accelerator, quoted verbatim from the [GA health check docs](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoint-groups-health-check-options.html):

> Global Accelerator considers a Network Load Balancer healthy if there is at least one healthy Availability Zone. An Availability Zone is healthy if it has a healthy target in all load balancer target groups that it is in.
>
> Be aware that when you enable cross-zone load balancing, healthy targets in Network Load Balancer target groups contribute to the health of the target group in all Availability Zones.

So enabling NLB cross-zone also changes how Global Accelerator judges the whole region's health. Worth knowing before you flip it.

### The ALB health signal Global Accelerator uses

Also verbatim, and it directly contradicts Route 53's behaviour:

> Global Accelerator considers an Application Load Balancer healthy if every target group has at least one healthy target.

Compare with [[aws-route53]] gotcha #4, where a Route 53 alias with `evaluate_target_health = true` reports an ALB with **no** targets as **healthy**. Global Accelerator calls the same ALB **unhealthy**.

**Global Accelerator is stricter and more correct here.** A standby scaled to literally zero pods is unhealthy to GA and healthy to Route 53. If you use GA with a scale-to-zero standby, **GA will refuse to fail over to it**, or rather it will exhaust its endpoint groups and *fail open* (below). Keep a non-zero baseline of targets in the standby. This is a hard constraint on the warm-standby shape, and it is not obvious.

### `client_keep_alive` — the setting worth more than any DNS tuning

Default **3600 seconds**. ARC's own best-practice page ([[aws-route53]]) says:

> By default, Application Load Balancers set the HTTP client keepalive duration value to 3600 seconds, or 1 hour. We suggest that you lower the value to be inline with your recovery time goal for your application, for example, 300 seconds.

An hour of clients pinned to the failing region after DNS has already moved. **Set `client_keep_alive = 300` on every internet-facing ALB in the estate.** It is one line, it costs nothing, and it is probably the highest-leverage change in this entire note.

## Global Accelerator

The genuine alternative to DNS-based failover, and it deserves more than a footnote.

### What it is

Two **static anycast IPv4 addresses** (plus dual-stack IPv6), advertised from the AWS global edge network. Clients connect to the nearest edge; AWS carries the traffic over its own backbone to a regional endpoint. You register **endpoint groups** — one per region — each containing endpoints (ALB, NLB, EC2, EIP).

The failover-relevant controls:

- **Endpoint health.** For ALB/NLB endpoints, GA uses the **load balancer's own health checks** (quoted above) — you do not configure a separate check, and the GA health check options apply only to EC2/EIP endpoints.
- **Traffic dial**, 0–100 per endpoint group: the percentage of traffic *otherwise destined for that group* that it actually receives. Set the standby's dial to 0 and the primary's to 100 and you have an active/passive pair with a **manual, instant, DNS-free switch**.
- **Endpoint weights** within a group.

### How failover actually works — the verbatim rules

From [How failover works for unhealthy endpoints](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoints-endpoint-weights.unhealthy-endpoints.html):

> If there are no healthy endpoints in an endpoint group that have a weight greater than zero, Global Accelerator tries to fail over to a healthy endpoint with a weight greater than zero in another endpoint group. **Note that for this failover, Global Accelerator ignores the traffic dial setting.** So if, for example, an endpoint group has a traffic dial set to zero, Global Accelerator still includes that endpoint group in the failover attempt.

This is the key behaviour and it is excellent for active/passive: **traffic dial 0 on the standby means "no traffic normally, but you are still my failover target."** It is exactly the semantics people *want* from a Route 53 weight of 0, delivered properly.

> If Global Accelerator doesn't find a healthy endpoint with a weight greater than zero after trying the three closest endpoint groups (that is, AWS Regions), it routes traffic to a random endpoint in the endpoint group that is closest to the client. That is, it *fails open*.

**Fail-open is a trap in a three-independent-pairs architecture.** If both EU regions are unhealthy, GA will happily route European traffic to a US or Canadian endpoint group *if they are in the same accelerator*. That is a data-residency breach and a "your data isn't here" bug ([[data-residency-compliance]]).

> [!danger] Use one accelerator per region pair, never one accelerator for all six regions
> A single accelerator containing all six endpoint groups will, under sufficient failure, cross-route EU traffic into `us-east-1`. **Three separate accelerators — one per pair, each containing exactly two endpoint groups — bounds the blast radius to the pair.** Three accelerators × $0.025/hour = $54.75/month, versus $18.25 for one. Pay the $36 and keep the compliance story.

And on recovery:

> When recovery occurs, that is, Regions are healthy again, Global Accelerator returns to regular routing behavior. This means that, typically, routing will start back to healthy endpoints with traffic dials that aren't set to zero in about 30 seconds or so. However, note that established active connections are not moved. They continue to route to the zero weight Region until the connection is reset by the client or the server, or until the client makes a new connection.

Two things there. **"About 30 seconds or so"** is AWS's own number for the routing change — that is the figure to quote for GA's failover speed. And **established connections are not moved** — the same caveat as DNS. GA removes the *resolver* caching problem entirely; it does **not** remove the *connection pool* problem. `client_keep_alive` still matters.

Also note the automatic failback: GA returns to normal routing when the region recovers, *automatically*, in ~30 s. **That is the same flapping hazard as Route 53 failover routing**, with the same fix: at failover, set the primary endpoint group's traffic dial to 0 manually so it cannot come back on its own.

### Region availability — including `ca-west-1`

From the [GA region availability page](https://docs.aws.amazon.com/global-accelerator/latest/dg/preserve-client-ip-address.regions.html), all six regions in scope are supported for endpoints:

| Region | Supported | Note |
|---|---|---|
| `eu-west-1` (Ireland) | Yes | |
| `eu-west-2` (London) | Yes | |
| `us-east-1` (N. Virginia) | Yes | |
| `us-west-2` (Oregon) | Yes | |
| `ca-central-1` (Montreal) | Yes | **"except AZ cac1-az3"** |
| **`ca-west-1` (Calgary)** | **Yes** | Added [25 April 2024](https://aws.amazon.com/about-aws/whats-new/2024/04/aws-global-accelerator-endpoints-calgary-region) |

**No `ca-west-1` gap for Global Accelerator** — a pleasant surprise given the region's other gaps ([[region-pair-selection]]). The `ca-central-1` AZ exception is the thing to check: if the existing Montreal deployment has subnets in `cac1-az3`, those subnets cannot host GA endpoints. **AZ IDs, not AZ names** — `cac1-az3` maps to a different letter in different accounts. Verify with `aws ec2 describe-availability-zones --region ca-central-1 --query 'AvailabilityZones[].[ZoneName,ZoneId]'`. Raised in [[#Open questions]].

Also from that page, the control-plane quirk consistent with [[aws-acm]]:

> AWS Global Accelerator is a global service. However, you must specify the US West (Oregon) Region (that is, specify the parameter `--region us-west-2`) in Regional Global Accelerator AWS CLI commands.

**GA's control plane is `us-west-2`, not `us-east-1`.** For the US pair — whose primary is `us-east-1` — that is a meaningful advantage over Route 53: you can change a traffic dial during a `us-east-1` impairment. You cannot change a DNS record.

### Cost

From the [GA pricing page](https://aws.amazon.com/global-accelerator/pricing/):

> For every full or partial hour when an accelerator runs in your account, you are charged $0.025 until it is deleted.

That fee "applies whether the accelerator is enabled or disabled". **$0.025/hour = ~$18.25/month per accelerator.** Three accelerators (one per pair) ≈ **$54.75/month**. Trivial.

The real cost is **DT-Premium**, a per-GB surcharge on *all* traffic through the accelerator, charged **in addition to** normal EC2 data transfer out, varying by source region and destination edge location, and — importantly — "You will only be charged DT-Premium in the dominant data transfer direction". Published rates span roughly $0.007/GB to $0.105/GB depending on the route.

**This is the whole GA decision.** At 10 TB/month egress from Europe, DT-Premium at the lower end of the European range adds a few hundred dollars a month; at Asia-Pacific/Australia rates it could add several thousand. **You cannot evaluate Global Accelerator without knowing your monthly egress volume and geography.** Get that number from Cost Explorer before the design review — it is the first thing [[cost-modelling]] should produce.

## Global Accelerator vs Route 53 failover

The head-to-head, honestly.

| | **Route 53 failover** | **Global Accelerator** |
|---|---|---|
| Client-visible address | A DNS name resolving to changing IPs | **Two static anycast IPs that never change** |
| Detection | Route 53 health checkers, 10/30 s × threshold ≈ 30–90 s | ALB's own target health (the LB already knows) |
| Propagation | **Bounded by DNS TTL + resolver behaviour + client DNS cache** | **~30 s, edge-side. No DNS involved.** |
| **Does the TTL problem exist?** | **Yes, and it is the dominant risk.** JVM caching forever, resolvers overstaying TTLs, CDNs with their own origin cache | **No. Completely eliminated.** The IP does not change, so nothing can cache the wrong one. |
| Connection-pool problem | Yes | **Yes — unchanged.** GA fixes resolution, not established connections. |
| Manual switch | Weighted 100/0 (needs `us-east-1` control plane) or ARC (~$1,825/mo) | **Traffic dial, effective in seconds, `us-west-2` control plane** |
| Gradual traffic shift | Weighted records only, TTL-limited | **Traffic dial 10→25→50→100. Lets the standby ALB scale with the ramp.** |
| Control plane during a `us-east-1` outage | **Unavailable** (accelerated recovery restores it after ~60 min) | **`us-west-2` — available** |
| Fail-open risk | No | **Yes — can cross-route to distant regions.** Must be bounded with one accelerator per pair. |
| Works for non-HTTP (databases, MQTT, game traffic) | Yes | Yes, TCP and UDP |
| Clients that hard-code IPs / firewall allow-lists | Broken by design | **Solved — two fixed IPs, permanently** |
| Cost | ~$25/month per pair | **$18.25/month per accelerator + DT-Premium on every GB** |
| Standby must have healthy targets to be a failover candidate | No (alias reports empty ALB healthy) | **Yes — "every target group has at least one healthy target"** |
| Certificate handling | Per-region ACM on each ALB | Same — GA is L4, TLS still terminates on the ALB |
| Blast radius of a misconfiguration | One DNS record | **Every client at once, instantly.** No TTL means no gradual rollout of a mistake either. |

### Recommendation

**Use Global Accelerator as the failover mechanism for the customer-facing entry point of each pair — one accelerator per pair, standby endpoint group at traffic dial 0 — and keep Route 53 as a simple alias to the accelerator's static IPs.**

The reasoning, in order of weight:

1. **It deletes the hardest problem in [[aws-route53]].** The DNS TTL / JVM cache / resolver-lies problem is not solvable by you; it lives in other people's software. GA makes it structurally impossible rather than merely improbable. That is worth more than any amount of TTL tuning.
2. **The traffic dial is a free, instant, gradual switch with its control plane outside `us-east-1`.** It gives you ARC's "manual on/off switch that works when `us-east-1` doesn't" for **$18.25/month instead of $1,825/month**, and it gives you canary failover that ARC does not.
3. **Gradual ramping mitigates the ALB pre-warming problem**, potentially saving the $584/month LCU reservation per load balancer. The two savings compound.
4. It fixes the IP-allow-list problem that B2B integrations always eventually raise.

**The costs of choosing it, stated plainly:**

- **DT-Premium on every byte.** This can dwarf every other number in this note. If egress is large, GA may be flatly unaffordable, and then the answer is Route 53 + ARC-or-STOP. **This single number decides it.**
- **Fail-open must be bounded** by one accelerator per pair, or you have a compliance incident waiting.
- **The standby must keep healthy targets running** — no scale-to-absolute-zero — because GA won't fail over to an ALB with an empty target group.
- Another global service, another set of Terraform resources, another thing to learn.
- It does **not** remove the need for Route 53: you still need a name, and you still want health checks for *alarming*.

**If DT-Premium is prohibitive, fall back to: Route 53 failover records + ARC routing controls (or the STOP pattern) + a hard-enforced `client_keep_alive = 300` + a verified JVM DNS TTL.** That combination meets the RTO too; it just requires you to be right about a dozen client-side details instead of zero.

> [!note] They are not mutually exclusive
> A reasonable end state is **GA for the pair-level failover** and **Route 53 geolocation** above it to pick *which pair* a customer belongs to. Route 53 answers "which accelerator", GA answers "which region within the pair". The geolocation layer changes rarely and can have a long TTL; the part that changes during an incident has no TTL at all. This is the shape to aim for.

## WAF and Shield

### Where the WAF sits

| Placement | Web ACL scope | Region the ACL lives in |
|---|---|---|
| On a regional ALB | `REGIONAL` | **The ALB's own region** |
| On a CloudFront distribution | `CLOUDFRONT` | **`us-east-1` only** ([[aws-acm]], [[aws-cloudfront]]) |

A `REGIONAL` web ACL created in `us-east-1` is a **different object** from a `CLOUDFRONT`-scope ACL created in `us-east-1` and cannot be attached to a distribution. This trips people up constantly.

**For a mirrored pair you need two `REGIONAL` web ACLs — one per region — with identical rules.** They are separate resources with separate ARNs, separate rule-group references and separate IP sets.

### Mirroring WAF rules is harder than mirroring the ALB

Three things drift independently:

1. **Rule configuration** — solved by the same module, same variables. Easy.
2. **IP sets** — an `aws_wafv2_ip_set` is regional. Block lists updated by an incident-response process (or a Lambda reacting to abuse) are updated *in the live region only*. **After failover, your standby has a six-month-old block list.** Any automation that writes to an IP set must write to **both** regions. This is a real, specific, easily-missed bug.
3. **Rate-based rule state** — rate-based rules count requests **per web ACL**. The standby's counters start at zero. Immediately after failover, every attacker gets a fresh rate budget. Nothing to be done about it; just know it and don't be surprised by the traffic spike profile.

Also: **WAF logging destinations are regional** (Kinesis Firehose / S3 / CloudWatch Logs), so the standby needs its own logging pipeline or your security team loses visibility exactly when they need it. See [[aws-waf-shield]] and [[cloudwatch-observability]].

### Shield

- **Shield Standard** is automatic and free on ALB, NLB, CloudFront and Global Accelerator. Nothing to do.
- **Shield Advanced** is a per-organisation subscription with a **`us-east-1` control plane** (per the [Fault Isolation Boundaries whitepaper](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/appendix-b---edge-network-global-service-guidance.html) cited in [[aws-acm]]). **Protections are per-resource**, so the standby ALB needs its own `aws_shield_protection` — and if you only add it at failover time, you are making a `us-east-1` control-plane call during an incident. Pre-provision protections on standby ALBs. They cost nothing extra beyond the subscription.
- If Shield Advanced is in use, note that protecting a **Global Accelerator** gives you standard accelerator protection at the edge, which is generally a stronger position than protecting regional ALBs — another point in GA's favour.

## `ca-west-1` specifics

| Capability | `ca-west-1` |
|---|---|
| ALB / NLB | Available. `ca-west-1` has **3 AZs** (`ca-west-1a/b/c`). |
| Global Accelerator endpoints | **Supported** since 25 April 2024 |
| ACM | Available, regional endpoint exists ([[aws-acm]]) |
| Route 53 health-checker managed prefix list | **Verify** — see [[aws-route53]] |
| WAFv2 `REGIONAL` scope | **Verify before designing** — do not assume |
| Gateway Load Balancer / third-party appliance chains | **Verify** if the estate uses them |

The general rule for `ca-west-1`: **it is a newer region and service parity must be checked, not assumed.** The good news for this note specifically is that the two things that would have hurt most — ALB and Global Accelerator — are both present. Confirm WAF before the CA pair's design is signed off; a missing regional WAF would force the CA pair onto a CloudFront-fronted design ([[aws-cloudfront]]) to get any WAF at all.

## Terraform implementation

### Module signature

`modules/regional-alb/variables.tf` — one module, called twice with different provider aliases.

```hcl
variable "name_prefix" { type = string }
variable "vpc_id"      { type = string }
variable "subnet_ids"  { type = list(string) }

variable "certificate_arn" {
  type        = string
  description = "In-region ACM cert. See [[aws-acm]] — must be the standby's own cert, never the primary's ARN."
}

variable "is_standby" {
  type        = bool
  default     = false
  description = "Tunes health check timings and capacity reservation for a region that must absorb a step-load."
}

variable "min_lcu_reservation" {
  type        = number
  default     = 0
  description = <<-EOT
    0 = no reservation (ALB scales reactively; a cold standby will brownout
    under a sudden full-production step-load). Minimum non-zero value AWS
    accepts is 100. Size from the PRIMARY's PeakLCUs metric, per-minute Sum.
  EOT
  validation {
    condition     = var.min_lcu_reservation == 0 || var.min_lcu_reservation >= 100
    error_message = "LCU reservation must be 0 or at least 100."
  }
}

variable "client_keep_alive_seconds" {
  type        = number
  default     = 300 # NOT the AWS default of 3600. See the RTO discussion.
}

variable "waf_web_acl_arn" {
  type    = string
  default = null
}
```

`modules/regional-alb/main.tf`:

```hcl
resource "aws_lb" "this" {
  name               = "${var.name_prefix}-alb"
  load_balancer_type = "application"
  internal           = false
  subnets            = var.subnet_ids
  security_groups    = [aws_security_group.alb.id]

  # Default is 3600s. An hour of clients pinned to a dead region after
  # DNS or the traffic dial has already moved. This is the single
  # highest-leverage RTO setting on the load balancer.
  client_keep_alive = var.client_keep_alive_seconds

  idle_timeout               = 60
  drop_invalid_header_fields = true
  enable_deletion_protection = true

  access_logs {
    # NOTE: the bucket MUST be in the same region as the load balancer.
    bucket  = var.access_log_bucket
    prefix  = var.name_prefix
    enabled = true
  }

  # Pre-warming. Provider support for this attribute is recent; if your
  # pinned provider version lacks it, manage it with the awscc provider or
  # a scheduled API call and ignore_changes here.
  dynamic "minimum_load_balancer_capacity" {
    for_each = var.min_lcu_reservation > 0 ? [1] : []
    content {
      capacity_units = var.min_lcu_reservation
    }
  }
}

resource "aws_lb_target_group" "app" {
  name        = "${var.name_prefix}-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "ip" # EKS pods

  # 300s default is too slow for a failback drain.
  deregistration_delay = 60

  health_check {
    path     = "/health"
    protocol = "HTTP"
    matcher  = "200"

    # Defaults are interval 30 / healthy_threshold 5 = 150s before a
    # newly-scaled-up pod takes traffic. In a standby that is 150s of a
    # 900s RTO budget spent waiting for a health check to agree with you.
    interval            = var.is_standby ? 10 : 30
    timeout             = 5
    healthy_threshold   = var.is_standby ? 2 : 5
    unhealthy_threshold = 2
  }

  # Cross-zone is ON by default for ALB and free. Left explicit for clarity;
  # for an NLB this defaults to FALSE and a scaled-down standby will
  # blackhole traffic to AZs with no targets.
  load_balancing_cross_zone_enabled = true

  lifecycle { create_before_destroy = true }
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.this.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = var.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.this.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

resource "aws_wafv2_web_acl_association" "this" {
  count        = var.waf_web_acl_arn == null ? 0 : 1
  resource_arn = aws_lb.this.arn
  web_acl_arn  = var.waf_web_acl_arn
}

output "dns_name" { value = aws_lb.this.dns_name }
output "zone_id"  { value = aws_lb.this.zone_id }  # for Route 53 alias records
output "arn"      { value = aws_lb.this.arn }      # for Global Accelerator endpoints
```

### Calling it for a pair

```hcl
module "alb_primary" {
  source          = "../../modules/regional-alb"
  providers       = { aws = aws }
  name_prefix     = "${var.env}-${var.primary_region}"
  vpc_id          = module.vpc_primary.vpc_id
  subnet_ids      = module.vpc_primary.public_subnet_ids
  certificate_arn = module.cert_primary.certificate_arn
  waf_web_acl_arn = module.waf_primary.web_acl_arn
  is_standby      = false
}

module "alb_standby" {
  source          = "../../modules/regional-alb"
  providers       = { aws = aws.standby }
  name_prefix     = "${var.env}-${var.standby_region}"
  vpc_id          = module.vpc_standby.vpc_id
  subnet_ids      = module.vpc_standby.public_subnet_ids

  # The STANDBY's own certificate. Passing module.cert_primary here is a
  # classic copy-paste bug and the API rejects it: cross-region cert ARNs
  # are not accepted by ELB.
  certificate_arn = module.cert_standby.certificate_arn

  waf_web_acl_arn = module.waf_standby.web_acl_arn
  is_standby      = true

  # Production only. ~$584/month/LB derived from the 100-LCU minimum
  # at $0.008/LCU-hr. See the Pre-warming section.
  min_lcu_reservation = var.env == "prod" ? 100 : 0
}
```

### Global Accelerator — one accelerator per pair

```hcl
# GA is a global service but its CLI/API is pinned to us-west-2.
provider "aws" {
  alias  = "ga"
  region = "us-west-2"
}

resource "aws_globalaccelerator_accelerator" "pair" {
  provider        = aws.ga
  name            = "${var.env}-${var.pair_name}"   # eu | us | ca
  ip_address_type = "IPV4"
  enabled         = true

  attributes {
    flow_logs_enabled   = true
    flow_logs_s3_bucket = var.ga_flow_log_bucket
    flow_logs_s3_prefix = "${var.env}/${var.pair_name}/"
  }
}

resource "aws_globalaccelerator_listener" "https" {
  provider        = aws.ga
  accelerator_arn = aws_globalaccelerator_accelerator.pair.id
  protocol        = "TCP"
  client_affinity = "NONE"

  port_range {
    from_port = 443
    to_port   = 443
  }
}

resource "aws_globalaccelerator_endpoint_group" "primary" {
  provider              = aws.ga
  listener_arn          = aws_globalaccelerator_listener.https.id
  endpoint_group_region = var.primary_region
  traffic_dial_percentage = 100

  endpoint_configuration {
    endpoint_id = module.alb_primary.arn
    weight      = 128
    # Rewrites the source IP to the client's, so the ALB sees the real
    # client IP rather than a GA edge IP. Requires the ALB's security
    # group to allow the GA managed prefix list, not 0.0.0.0/0.
    client_ip_preservation_enabled = true
  }
}

resource "aws_globalaccelerator_endpoint_group" "standby" {
  provider              = aws.ga
  listener_arn          = aws_globalaccelerator_listener.https.id
  endpoint_group_region = var.standby_region

  # 0 = receives no traffic under normal operation, BUT remains a failover
  # target: "for this failover, Global Accelerator ignores the traffic dial
  # setting." This is the semantics a Route 53 weight of 0 only pretends to
  # have. Raise this to 10 / 25 / 50 / 100 to fail over gradually and let
  # the standby ALB scale with the ramp.
  traffic_dial_percentage = 0

  endpoint_configuration {
    endpoint_id                    = module.alb_standby.arn
    weight                         = 128
    client_ip_preservation_enabled = true
  }
}

# One simple alias. The IPs behind it never change, so TTL is irrelevant.
resource "aws_route53_record" "app" {
  zone_id = data.aws_route53_zone.public.zone_id
  name    = "app.${var.public_domain}"
  type    = "A"

  alias {
    name                   = aws_globalaccelerator_accelerator.pair.dns_name
    zone_id                = aws_globalaccelerator_accelerator.pair.hosted_zone_id
    evaluate_target_health = true
  }
}
```

The failover action is then a single regional call with no DNS and no `us-east-1`:

```bash
aws globalaccelerator update-endpoint-group --region us-west-2 \
  --endpoint-group-arn "$STANDBY_EG_ARN" --traffic-dial-percentage 100
aws globalaccelerator update-endpoint-group --region us-west-2 \
  --endpoint-group-arn "$PRIMARY_EG_ARN" --traffic-dial-percentage 0
```

Note the ordering: **raise the standby first, then drop the primary.** The reverse leaves a window with both dials effectively closed.

## Migration path from single-region

1. **Import the existing ALB** into the `regional-alb` module. Verify an empty plan. Expect the first plan to want to change `client_keep_alive` (3600 → 300), `deregistration_delay` (300 → 60) and health-check thresholds. **All are in-place updates, not replacements** — but apply them one at a time in the live region and watch, because shortening the keepalive changes connection churn.
2. **Deploy the standby ALB** with `is_standby = true`. Purely additive, touches nothing live.
3. **Allow the Route 53 health-checker prefix list** through the standby SG, and the GA prefix list if using GA with client IP preservation.
4. **Attach the standby certificate** ([[aws-acm]] step 5) and confirm `RenewalEligibility = ELIGIBLE`.
5. **Mirror the WAF**, including any IP sets, and point any IP-set-updating automation at both regions.
6. **Register a real (small) set of targets in the standby.** Not zero. Both because GA requires it and because an ALB with no targets is untestable.
7. **Then** choose the failover mechanism: if Global Accelerator, create the accelerator, add the **primary** endpoint group at dial 100 and the **standby** at 0, and **CNAME/alias the public name to the accelerator**. This is the only step with client-visible risk, and it is a DNS change with a TTL — so lower the TTL first, cut over, verify, then raise it. Once you are on the accelerator's static IPs, this is the last DNS change you ever need to make for failover.
8. **Set the LCU reservation** on the standby (or decide, with evidence from step 9, that you don't need one).
9. **Test with real step-load.** Move 100% of traffic to the standby in one action, during a low-traffic window, and watch `TargetResponseTime`, `HTTPCode_ELB_5XX_Count`, `RejectedConnectionCount` and `ActiveConnectionCount`. This test is the only thing that turns "we have a warm standby" into "we have a warm standby that works".

> [!warning] What forces replacement
> `aws_lb`: `name`, `internal`, `load_balancer_type`, `subnets` **for an NLB** (subnets can be *added* to an ALB in place but not removed), `ip_address_type` in some transitions.
> `aws_lb_target_group`: `name`, `port`, `protocol`, `vpc_id`, `target_type` — all `ForceNew`. Changing `target_type` from `instance` to `ip` (the usual EKS migration) **replaces the target group**, which detaches it from the listener. Always `create_before_destroy = true` on target groups.
> Renaming an ALB destroys it, and **the new one has a different DNS name and different IPs.** If anything has allow-listed those IPs, that is an outage. (Global Accelerator's static IPs make this class of problem go away permanently.)

## Failover procedure

Assuming Global Accelerator:

1. Confirm the standby's targets are healthy — `aws elbv2 describe-target-health` in the standby region. **GA will not fail over to an ALB whose target groups have no healthy targets.**
2. (Optional but recommended) Set the standby's LCU reservation to match the primary's recent `PeakLCUs`.
3. Raise the standby endpoint group's traffic dial: 10 → watch → 50 → watch → 100. Two to three minutes total, and the ramp lets the ALB scale.
4. Drop the primary endpoint group's dial to 0. **Do this explicitly** — otherwise GA will automatically route back to the primary about 30 seconds after it looks healthy again.
5. Watch `NewFlowCount` and `ProcessedBytesIn` per endpoint group in CloudWatch to confirm the shift.

Assuming Route 53:

1–2 as above, then flip the ARC routing control / weights / health check, and **disable the primary's health check** so it cannot fail back on its own. Then wait out the TTL, and watch the *primary's* `RequestCount` fall. If it doesn't fall, you have a client-caching problem ([[aws-route53]]).

## Failback

- **Failback is where the traffic dial earns its money.** 10 → 25 → 50 → 100 back to the primary, over as long as you like, watching error rates at each step. Route 53 weighted records can approximate this but every step is TTL-limited and resolver-dependent.
- The primary's ALB has now been idle for however long the incident lasted, so **it is now the cold one**. Everything in [[#Pre-warming]] applies in reverse. Set an LCU reservation on the primary before failing back, or ramp slowly.
- Remember to **re-enable whatever you disabled** — the primary's Route 53 health check, the primary endpoint group's dial, automatic failback behaviour.
- **Re-check WAF IP sets.** Anything added to the standby's block list during the incident must be copied back.

## Gotchas

1. **Cross-region certificate ARNs are rejected by ELB.** Passing the primary's cert to the standby listener fails at apply time. Loud, at least.
2. **`client_keep_alive` defaults to 3600 seconds.** One hour of clients pinned to a dead region. The highest-value one-line change in this note.
3. **`deregistration_delay` defaults to 300 seconds.** Five minutes per drain, which shows up on failback and on every rolling deploy.
4. **`HealthyThresholdCount` defaults to 5 with a 30 s interval = 150 s** before a scaled-up target takes traffic. Tune in the standby.
5. **An ALB with zero healthy targets fails open** and forwards to unhealthy targets anyway. Alarm on `HealthyHostCount`, never on `RequestCount`.
6. **Global Accelerator considers an ALB unhealthy if any target group has no healthy target** — the opposite of Route 53's alias behaviour, which reports an empty ALB as healthy. **A scale-to-zero standby is invisible to GA.**
7. **Global Accelerator fails open across endpoint groups**, potentially routing EU traffic to North America. **One accelerator per region pair** or you have a data-residency incident.
8. **Global Accelerator fails *back* automatically in ~30 s** when the region recovers. Same flapping hazard as Route 53 failover routing. Zero the primary's dial deliberately.
9. **Established connections are never moved** — not by DNS, not by GA. Connection pools and keepalives are the residual risk in both designs.
10. **NLB cross-zone load balancing is off by default** and NLB cross-zone traffic is billed as inter-AZ transfer. A scaled-down standby NLB with cross-zone off blackholes traffic to empty AZs.
11. **Access log buckets must be in the load balancer's region.** The standby needs its own bucket ([[aws-s3]]).
12. **WAF IP sets are regional and drift silently.** Any automation that writes block lists must write to both regions.
13. **Rate-based WAF rule counters reset on failover.** Attackers get a fresh budget.
14. **Renaming or recreating an ALB changes its DNS name and IPs**, breaking any client allow-list. GA's static IPs immunise you against this class of problem forever.
15. **`target_type` changes are `ForceNew`** and will detach the target group from the listener mid-apply without `create_before_destroy`.
16. **Global Accelerator's CLI requires `--region us-west-2`** even though it is a global service. Also means its control plane survives a `us-east-1` event.
17. **`ca-central-1` excludes AZ `cac1-az3` from Global Accelerator.** Check AZ **IDs**, not letters.
18. **LCU reservation has a 100-LCU floor**, so there is no "cheap small reservation". It is ~$584/month/LB or nothing.
19. **Pre-warming by support case still exists but is useless for unplanned failover.** Use it for scheduled failover tests only.
20. **`enable_deletion_protection` on the standby will block a `terraform destroy`** in ephemeral environments. Make it conditional on `env == "prod"`.

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| **Failover mechanism** | **A: Route 53 failover/ARC.** ~$25/mo per pair (or +$1,825/mo for ARC). Subject to DNS TTL, JVM caching, resolver behaviour. Control plane in `us-east-1`. | **B: Global Accelerator.** $18.25/mo per accelerator + **DT-Premium per GB**. Static anycast IPs, **no DNS TTL problem at all**, ~30 s failover, gradual traffic dial, `us-west-2` control plane. | **B, if DT-Premium is affordable.** It structurally eliminates the hardest failure mode and gives you a manual switch outside `us-east-1` for 1% of ARC's price. **Get the monthly egress figure first — that single number decides it.** If GA is too expensive, A with ARC-or-STOP plus hard-enforced `client_keep_alive` and JVM TTL. |
| **Standby ALB: keep or delete when idle** | A: Keep. ~$16.43/mo each. | B: Delete, recreate at failover. Saves ~$16/mo. | **A, emphatically.** B loses certificate auto-renewal ([[aws-acm]]), changes the DNS name, puts a `terraform apply` in the recovery path, and starts from minimum capacity. It saves the price of a couple of coffees. |
| **LCU reservation on the standby** | A: Permanent 100 LCU, ~$584/mo/LB. Capacity guaranteed. | B: Set it as failover step #1, ~$0/mo idle, racing the traffic. C: nothing (the silent default). | **A for production, B for non-production — then run a real step-load test and downgrade A to B if the evidence supports it.** Never C by accident. If GA's traffic dial is available to ramp gradually, B becomes much safer and the saving is real. |
| **One accelerator vs three** | A: One accelerator, six endpoint groups. $18.25/mo. | B: Three accelerators, two endpoint groups each. $54.75/mo. | **B.** A will cross-route EU traffic to North America when it fails open. $36/month for a bounded blast radius and a defensible compliance story. |
| **ALB vs NLB at the edge** | A: ALB — L7 routing, WAF, OIDC, HTTP metrics | B: NLB — better burst headroom (800 new conn/s per NLCU), static IPs, lower latency | **A**, unless the workload is non-HTTP. NLB's static IPs are attractive but Global Accelerator gives you static IPs *in front of an ALB*, which is strictly better. Never make the standby a different type from the primary. |
| **Health check tuning in the standby** | A: Same as primary (defaults) | B: `interval=10`, `healthy_threshold=2` | **B.** Saves 130 s of a 900 s budget for no meaningful cost. |

## Cost

Monthly, per standby region, at zero traffic.

| Item | Cost |
|---|---|
| ALB hourly ($0.0225 × 730) | **$16.43** |
| LCUs at zero traffic | ~$0 |
| **Idle standby ALB, total** | **≈ $16.43** |
| Six idle load balancers (3 pairs × 2) | **≈ $99** |
| LCU reservation, 100 LCU (derived: 100 × $0.008 × 730) | **≈ $584 per load balancer** |
| Global Accelerator, per accelerator ($0.025 × 730) | **$18.25** |
| Three accelerators | **$54.75** |
| Global Accelerator DT-Premium | **$0.007–$0.105 per GB**, route-dependent — *the number that decides everything* |
| WAF web ACL + rules, per region | See [[aws-waf-shield]] |
| NLB cross-zone inter-AZ data transfer | Per GB, only if cross-zone enabled on an NLB |

**The levers:**

1. **Idle ALBs are not a cost problem. Stop trying to optimise them.** $99/month for six is noise, and deleting them costs you certificate renewal.
2. **LCU reservations are the real number** — potentially $1,750/month across three production standbys. This is the one line item worth arguing about, and the argument is settled by a load test, not by opinion.
3. **DT-Premium is the GA decision.** Unknowable without the egress figure. Get it from Cost Explorer.
4. **Global Accelerator's fixed fee is irrelevant** at $54.75/month for three accelerators. Do not let anyone reject GA on the accelerator fee; reject it on DT-Premium or not at all.

## Open questions

1. **What is total monthly data transfer out, by region?** Decides Global Accelerator outright. Highest-priority question in this note.
2. **What is the primary's `PeakLCUs`, per-minute Sum, at peak?** Sizes the LCU reservation and quantifies the pre-warming risk. Available today from CloudWatch.
3. **What is the exact reserved-LCU price?** The ~$584/month figure is derived from the 100-LCU minimum and the $0.008/LCU rate, not read from a single published line. Verify before budgeting.
4. **Is `client_keep_alive` still 3600 on the live ALBs?** If yes, the current RTO is >1 hour regardless of anything else in this vault.
5. **Does `ca-central-1` use AZ `cac1-az3`?** If yes, those subnets can't host Global Accelerator endpoints. Check AZ IDs.
6. **Is WAFv2 `REGIONAL` scope available in `ca-west-1`?** Blocks the CA pair's design if not.
7. **Is Shield Advanced subscribed?** If so, standby ALBs need pre-provisioned protections, and GA becomes more attractive.
8. **Does anything (B2B partner, mobile app, corporate firewall) allow-list the ALB's IPs?** If yes, Global Accelerator moves from "nice" to "required".
9. **Can the standby run a non-zero baseline of pods?** Required for Global Accelerator to consider the standby a valid failover target. Interacts with [[aws-eks]] cost decisions.
10. **Who updates WAF IP sets during an incident, and does that process write to both regions?** Almost certainly not, today.

## Sources

- [Elastic Load Balancing pricing](https://aws.amazon.com/elasticloadbalancing/pricing/) — ALB $0.0225/hour and $0.008/LCU; NLB $0.0225/hour and $0.006/NLCU; the LCU dimensions (25 new connections/s, 3,000 active connections/min, 1 GB/hour processed, 1,000 rule evaluations/s) and the NLCU dimensions (800 new TCP connections/s, 100,000 active, 1 GB/hour); the "charged only on the dimension with highest usage" rule; the LCU-reservation billing model.
- [Capacity reservations for your Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/capacity-unit-reservation.html) — the verbatim definition of LCU reservation, the four listed use cases (including migrating between load balancers), the `PeakLCUs` vs `ConsumedLCUs` sizing guidance, and the **100-LCU minimum**.
- [Load Balancer Capacity Unit Reservation for Application and Network Load Balancers](https://aws.amazon.com/about-aws/whats-new/2024/11/load-balancer-capacity-unit-reservation-application-balancers) — the 20 November 2024 GA announcement; establishes that this is the modern replacement for pre-warming support cases.
- [Health checks for Application Load Balancer target groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html) — the full defaults table (interval 30 s, timeout 5 s, healthy threshold 5, unhealthy threshold 2, matcher 200) and the verbatim fail-open statement.
- [How failover works for unhealthy endpoints (Global Accelerator)](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoints-endpoint-weights.unhealthy-endpoints.html) — verbatim: failover ignores the traffic dial; the three-closest-endpoint-groups rule and fail-open behaviour; "about 30 seconds or so" to return to regular routing; established connections are not moved.
- [Ensure health check access for your accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoint-groups-health-check-options.html) — verbatim: GA health check options do not apply to ALB/NLB endpoints; "Global Accelerator considers an Application Load Balancer healthy if every target group has at least one healthy target"; the NLB/AZ and cross-zone interaction; the Route 53 health-checker IP requirement for EC2/EIP endpoints.
- [AWS Region availability for AWS Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/preserve-client-ip-address.regions.html) — the full supported-region table confirming `ca-west-1`, `eu-west-1`, `eu-west-2`, `us-east-1`, `us-west-2`, and the `ca-central-1` `cac1-az3` exception; plus the requirement to use `--region us-west-2` for GA CLI commands.
- [AWS Global Accelerator now supports endpoints in the Canada West (Calgary) Region](https://aws.amazon.com/about-aws/whats-new/2024/04/aws-global-accelerator-endpoints-calgary-region) — 25 April 2024, dates `ca-west-1` support.
- [AWS Global Accelerator pricing](https://aws.amazon.com/global-accelerator/pricing/) — verbatim $0.025/hour fixed fee charged whether the accelerator is enabled or disabled; DT-Premium per-GB model, the dominant-direction rule, and the rate range.
- [AWS Fault Isolation Boundaries — Appendix B](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/appendix-b---edge-network-global-service-guidance.html) — control-plane home regions, including Global Accelerator in `us-west-2` and Shield Advanced in `us-east-1`.
- [ELB Pre-warm feature (AWS re:Post)](https://repost.aws/questions/QUUNX-KyAHS0K6DOgnJeHejQ/elb-pre-warm-feature) and [ALB pre-warming (AWS re:Post)](https://repost.aws/questions/QUhvpfhsDsSZKRvoLOyWMRkQ/alb-pre-warming) — AWS's historical support-case pre-warming process (start/end dates, expected request rate, typical request/response size) and its supersession by LCU reservation.

## Related notes

[[aws-route53]] · [[aws-acm]] · [[aws-cloudfront]] · [[aws-waf-shield]] · [[aws-eks]] · [[aws-vpc-networking]] · [[aws-s3]] · [[cost-modelling]] · [[failover-runbooks]] · [[failover-orchestration]] · [[region-pair-selection]] · [[data-residency-compliance]] · [[cloudwatch-observability]] · [[terraform-repo-structure]]
