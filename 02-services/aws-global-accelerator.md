---
title: AWS Global Accelerator — Multi-Region
service: global-accelerator
tags: [service, multi-region, global-accelerator, anycast, failover, edge, dt-premium, cost]
status: partial
replication: global — nothing to replicate; the accelerator and its IPs are already everywhere
rpo_achievable: N/A — traffic steering, not data
rto_achievable: "traffic-shift step: seconds-to-~30 s and no DNS TTL at all. Detection for ALB/NLB endpoints is the ALB's own health-check timing, which AWS does not publish as a single number."
meets_targets: yes — for the traffic-shift step, more cleanly than any other mechanism in this vault. The decision is cost, not capability.
updated: 2026-09-21
---

# AWS Global Accelerator — Multi-Region

> **Scope.** This note is the **head-to-head challenger to [[aws-route53]]**. That
> note documents where the 15-minute RTO is won or lost on DNS: ~90 s of the 900 s
> budget goes to DNS, and a JVM DNS cache can silently extend it to an hour.
> Global Accelerator's whole claim is that it **deletes that problem**, not that it
> mitigates it. [[aws-alb-nlb]] has a shorter GA section reaching the same
> conclusion; this note is the deep version, and it also settles the three-way
> question the vault has never answered: **Route 53 DNS failover vs CloudFront
> origin groups vs Global Accelerator.** The verdict is at
> [[#The verdict — the three-way decision table]].

## TL;DR

- **Two static anycast IPv4 addresses that never change for the life of the accelerator.** Because the client-facing address is constant, **failover removes the DNS TTL problem and the client-side DNS-cache problem entirely** — not "reduces", *removes*. There is no wrong IP to cache. [[aws-route53]]'s hardest, least-controllable risk (the JVM `networkaddress.cache.ttl` set to "cache forever", resolvers overstaying TTLs) becomes structurally impossible rather than merely improbable. **This is the entire argument for GA and it is a strong one.**
- **It does not fix connection pools, and AWS is explicit about the number.** GA's idle timeout is **340 seconds for TCP, 30 seconds for UDP, not customisable**, and *"Global Accelerator continues to direct traffic for established connections to an endpoint until the idle timeout is met, even if the endpoint is marked as unhealthy or if it is removed from the accelerator."* GA fixes *resolution*; the ALB `client_keep_alive = 300` change from [[aws-alb-nlb]] is still mandatory. Anyone selling GA as "instant failover for all traffic" has not read that sentence.
- **`ca-west-1` is supported, with no AZ exception — and `ca-central-1` has one.** Calgary has failed the parity check for Cognito, OpenSearch CCR, Managed Grafana and Backup Audit Manager ([[region-pair-selection]]). **It passes here.** The surprise is the other way round: **`ca-central-1` is listed "except AZ `cac1-az3`"** — the *primary*, not the standby, carries the constraint. Verify your Montreal subnets' AZ **IDs**. See [[#ca-west-1 and ca-central-1 — the parity check]].
- **The cost verdict turns on one number and this estate happens to be in the cheap band.** The accelerator fee is trivial (**$0.025/hour ≈ $18.25/month**, three accelerators ≈ **$54.75/month**). The killer is normally **DT-Premium**, a per-GB surcharge *on top of* normal data transfer out. But DT-Premium is priced by *source Region → destination edge*, and all three pairs here serve customers in their own continent, which is the **$0.015/GB** band — not the $0.105/GB Australia band that kills the idea elsewhere. At 10 TB/month per pair the whole estate is **≈ $515/month**. See [[#Cost]].
- **The thing that will bite: fail-open, and it is a data-residency bug.** If no healthy weighted endpoint is found after trying the three closest endpoint groups, GA *"routes traffic to a random endpoint in the endpoint group that is closest to the client"* — **ignoring the traffic dial**. One accelerator containing all six regions will cross-route EU traffic into `us-east-1` under sufficient failure. **Three accelerators, one per pair, two endpoint groups each.** $36/month buys the compliance story ([[data-residency]]).

## Does this service cross regions at all?

Wrong question, and worth saying plainly before anyone opens a ticket called
"replicate the accelerator to eu-west-2".

**Global Accelerator is a global service.** It sits in the same AWS
fault-isolation category as Route 53, CloudFront and IAM — see the
[Fault Isolation Boundaries whitepaper, Appendix B](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/appendix-b---edge-network-global-service-guidance.html),
already cited in [[aws-acm]], [[aws-route53]] and [[aws-regional-outages]].

| Thing | Scope |
|---|---|
| Accelerator (`aws_globalaccelerator_accelerator`) | **Global.** No region attribute. |
| The two static IPv4 addresses (four if dual-stack) | **Global anycast**, advertised from the AWS edge network. |
| Listener (`aws_globalaccelerator_listener`) | **Global**, a child of the accelerator. |
| **Endpoint group** | **Regional** — this is the *only* regional object, and it is the one you create twice. |
| Endpoints (ALB, NLB, EC2, EIP) | **Regional**, owned by [[aws-alb-nlb]] / [[aws-eks]]. |
| **Global Accelerator control plane** | **`us-west-2` only.** See [[#The control plane — and why us-west-2 is the interesting answer]]. |
| Data plane (the anycast advertisement, edge TCP termination, health evaluation) | **Global**, every AWS edge location. |

Three practical consequences for the Terraform estate:

1. **There is exactly one accelerator per region pair, for all time.** It already
   survives the loss of `eu-west-1`. There is nothing to mirror. The multi-region
   work is *one extra `aws_globalaccelerator_endpoint_group` resource* pointed at
   the standby ALB — which, compared with everything else in this vault, is an
   embarrassingly small diff.
2. **The accelerator stack is a global stack, not a per-region module.** It
   consumes outputs (ALB ARNs) from both regional stacks. Structurally identical
   to the "one DNS stack, not one per region" recommendation in [[aws-route53]],
   and it is probably literally the same stack. See
   [[#Terraform implementation]] and [[provider-aliases-vs-separate-stacks]].
3. **Route 53 does not go away.** You still need a name. `app.example.com`
   becomes an **alias to the accelerator**, using the fixed Global Accelerator
   hosted zone ID `Z2BJ6XQ5FK7U4H`
   ([provider docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/globalaccelerator_accelerator)).
   The difference is that **this record never changes again**, so its TTL stops
   mattering — which is the entire point.

> [!important] Say this to the team once, clearly
> Global Accelerator is not "another thing to make multi-region". It is a
> *replacement for the failover mechanism*. Adopting it **deletes work** from
> [[aws-route53]] and [[failover-orchestration]] — it does not add a workstream.

## The anycast thesis — the head-to-head with Route 53

This is the point of the note, so it goes near the top rather than buried in a
comparison table at the end.

### What [[aws-route53]] actually costs you

From that note's own arithmetic, the DNS layer contributes **~90–120 seconds** to
the 900-second budget: ~30 s of health-check detection (10 s interval × threshold
3), a few seconds of internal propagation, then **60 s of record TTL**. That is
fine. Ninety seconds out of nine hundred is not the problem.

**The problem is the sentence after it.** That note's own honest statement:

> Route 53 will be serving the new answer in ~90 seconds, and some fraction of
> your traffic will not notice for much longer than that.

Because the mechanism is "change which IP the name resolves to", every cache
between you and the customer gets a vote:

| Cache | Who controls it | Worst case |
|---|---|---|
| Recursive resolvers that overstay the TTL | Nobody. Measured in [Moura et al., IMC 2019](https://ant.isi.edu/~johnh/PAPERS/Moura19b.html) | Minutes to hours |
| **JVM `networkaddress.cache.ttl` negative value** | Your base image, *if* someone set it | **Forever, until process restart** |
| ALB HTTP client keepalive, default **3600 s** | You (one attribute) | 1 hour |
| Connection pools, HTTP/2 sessions, WebSockets | Your app | Until the pool recycles |
| CDNs / reverse proxies with their own origin-resolution cache | The CDN vendor | Vendor-defined |

Four of those five are **someone else's software**. You can *audit* them; you
cannot *guarantee* them; and the audit is invisible to every test that does not
actually move an IP.

### What Global Accelerator does to that table

**It deletes the top two rows and leaves the bottom three.**

The client resolves `app.example.com` → `75.2.x.x, 99.83.x.x` (or whatever your
two addresses are). Those addresses are **the same before, during and after
failover**. From the [GA how-it-works page](https://docs.aws.amazon.com/global-accelerator/latest/dg/introduction-how-it-works.html),
verbatim:

> The static IP addresses remain assigned to your accelerator for as long as it
> exists, even if you disable the accelerator and it no longer accepts or routes
> traffic.

So:

- A resolver overstaying the TTL is **serving the correct answer anyway**.
- A JVM that cached the IP at process start and will never refresh it is
  **holding the correct IP**. The legendary failover-breaker in [[aws-route53]]
  is, against an anycast entry point, *harmless*.
- A firewall allow-list at a B2B customer, an IoT device with a hard-coded IP, a
  mobile client with an aggressive DNS cache — all fine.

The failover decision moves from "what does this name resolve to?" (a distributed
cache-coherency problem you do not own) to "which endpoint does the AWS edge
forward this connection to?" (a routing decision AWS makes, at the edge, on every
new connection).

> [!important] This is the single strongest architectural argument in this vault
> Every other mitigation for the DNS-cache problem is a *discipline*: set the TTL
> to 60, set `client_keep_alive`, set the JVM security property, audit it
> annually, hope nobody ships a base image that undoes it. GA makes the problem
> **not exist**. In a 15-minute RTO, replacing a discipline with a structural
> guarantee is worth real money.

### What it does **not** fix — read this before celebrating

From the same page, verbatim, and this is the sentence that gets skipped:

> Global Accelerator continues to direct traffic for established connections to
> an endpoint until the idle timeout is met, even if the endpoint is marked as
> unhealthy or if it is removed from the accelerator. Global Accelerator selects
> a new endpoint, if needed, only when a new connection starts or after an idle
> timeout.

And the timeouts themselves:

> The timeout is 340 seconds for TCP connections.
> The timeout is 30 seconds for UDP connections.
>
> The idle timeout periods are not customizable.

Three consequences, all of which belong in [[failover-runbook-template]]:

1. **An idle-but-alive TCP connection can stay pinned to the dead region for up
   to 340 seconds** — nearly 6 minutes, 38% of the RTO budget — *and you cannot
   tune it*. "Idle" here is strict: AWS says *"you cannot use TCP keep-alive
   packets to maintain an open connection"*, so a connection sending real data
   every few seconds never idles out at all and stays pinned **indefinitely**.
2. **`client_keep_alive = 300` on the ALB is still mandatory.** GA does not
   recycle client connections for you; the ALB does. This is the same one-line
   change [[aws-alb-nlb]] calls the highest-leverage item in that note, and GA
   does not let you skip it.
3. **The honest comparison is therefore:** Route 53 has a *resolution* problem
   **and** a *connection* problem; GA has only the *connection* problem. That is
   a large improvement, not a total one.

### Head to head

| | **Route 53 DNS failover** | **Global Accelerator** |
|---|---|---|
| Client-facing address | A name resolving to **changing** IPs | **Two static anycast IPs, permanent** |
| Detection | Route 53 health checkers, 10/30 s × threshold 3 ≈ 30–90 s, **18% checker quorum** | The ALB's own target health (GA does not add a check for LB endpoints) |
| Propagation to clients | **TTL + resolver behaviour + client DNS cache** | **Edge-side, no DNS involved** |
| Documented shift figure | ~90 s total DNS contribution (vault's own arithmetic) | *"about 30 seconds or so"* — but see [[#Health checks and what "30 seconds" actually means]]; that is AWS's **recovery** figure |
| **Does the TTL/DNS-cache problem exist?** | **Yes, and it is the dominant uncontrolled risk** | **No. Structurally impossible.** |
| Connection-pool problem | Yes | **Yes — unchanged, plus a hard 340 s TCP idle timeout you cannot tune** |
| Manual switch | Weighted 100/0 (needs the **`us-east-1`** control plane) or ARC (**$1,825/mo**) | **Traffic dial, `us-west-2` control plane, $0 extra** |
| Gradual / canary shift | Weighted records, TTL-limited, coarse | **Traffic dial 10 → 25 → 50 → 100, edge-side, immediate** |
| Control plane during a `us-east-1` impairment | **Unavailable** (accelerated recovery restores it after ~60 min) | **Available — `us-west-2`** |
| Control plane during a `us-west-2` impairment | Available (`us-east-1`) | **Unavailable — see [[#The control plane — and why us-west-2 is the interesting answer]]** |
| Cross-region fail-open risk | No | **Yes.** Must be bounded by one accelerator per pair |
| Standby with zero healthy targets | Reported **healthy** by an alias with `evaluate_target_health` | Reported **unhealthy** — GA will not fail over to it |
| Protocols | Anything (it is just a name) | **TCP and UDP.** No HTTP semantics, no caching |
| Blast radius of a misconfiguration | One record, bounded by TTL, rolls out gradually | **Every client, instantly.** No TTL means no gradual rollout of a mistake either |
| Cost | ~$25/month per pair | **$18.25/month per accelerator + DT-Premium on every GB** |

**The two rows that decide it** are "does the TTL problem exist" (GA wins,
decisively) and "cost" (depends entirely on egress volume and geography — see
[[#Cost]], where this estate turns out to be in the cheap band).

## Traffic dials and endpoint weights — the failover switch

GA gives you two independent knobs, and the distinction matters because only one
of them is your failover switch.

From the [how-it-works page](https://docs.aws.amazon.com/global-accelerator/latest/dg/introduction-how-it-works.html),
verbatim:

> You configure traffic dials for *endpoint groups*. The traffic dial lets you
> cut off a percentage of traffic—or all traffic—to the group, by "dialing down"
> traffic that the accelerator has already directed to it based on other factors,
> such as proximity.

> You use weights, on the other hand, to set values for *individual endpoints*
> within an endpoint group.

And on the dial's exact semantics:

> The traffic dial limits the portion of traffic that an endpoint group accepts,
> expressed as a percentage of traffic directed to that endpoint group. For
> example, if you set the traffic dial for an endpoint group in `us-east-1` to 50
> (that is, 50%) and the accelerator directs 100 user requests to that endpoint
> group, only 50 requests are accepted by the group. The accelerator directs the
> remaining 50 requests to endpoint groups in other Regions.

> [!warning] The dial is a *percentage of traffic already routed here*, not a global share
> With **two** endpoint groups this is intuitive: primary at 100 / standby at 0
> means "everything that geo-routing sent to the primary, the primary keeps".
> With **three or more** it is not intuitive at all, because the denominator is
> whatever geo-proximity already decided. One more reason for
> **exactly two endpoint groups per accelerator**.

### The mapping to active/passive

| Posture | Primary endpoint group dial | Standby endpoint group dial |
|---|---|---|
| Normal running | **100** | **0** |
| Canary / rehearsal | 90 | 10 |
| Failed over | **0** | **100** |

**And here is the property that makes GA better at active/passive than Route 53's
weighted 0.** From [How failover works for unhealthy endpoints](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoints-endpoint-weights.unhealthy-endpoints.html),
verbatim:

> If there are no healthy endpoints in an endpoint group that have a weight
> greater than zero, Global Accelerator tries to fail over to a healthy endpoint
> with a weight greater than zero in another endpoint group. **Note that for this
> failover, Global Accelerator ignores the traffic dial setting.** So if, for
> example, an endpoint group has a traffic dial set to zero, Global Accelerator
> still includes that endpoint group in the failover attempt.

Read that against [[aws-route53]]'s gotcha #2, where a Route 53 weight of 0 is
served *only if everything else is dead* — a behaviour that note describes as
counter-intuitive and a trap. **GA gives you the semantics people actually want
and Route 53 half-delivers:** dial 0 means *"no traffic under normal
circumstances, but you remain my failover target."* That is precisely the
definition of a warm standby, expressed as one integer.

### Two switches, two very different meanings

| You want | Set | Effect |
|---|---|---|
| "Standby takes over **now**, because I decided" | Primary group dial **0**, standby dial **100** | Deliberate, human, ~seconds |
| "Standby takes over **if** the primary dies" | Leave dials 100/0 | Automatic, on endpoint health, unattended |
| "Standby is out of service entirely" (e.g. mid-deploy, DB not ready) | Standby endpoint **weight 0** | Weight 0 removes it from the failover attempt too — **the real off switch** |

That third row is the one people miss and it matters enormously for an
async-replica standby: **the traffic dial does not stop an automatic failover, only
a weight of 0 does.** If your standby's database is 2 hours behind and you do not
want an unattended failover into it, **dial 0 is not enough**.

> [!danger] For an RPO of 2 hours, "automatic" is a data-loss event
> [[aws-route53]] makes this argument and it applies identically here: with dials
> at 100/0 and both endpoints weighted, **GA will fail your production traffic
> into a 2-hour-stale replica at 3am with nobody in the loop**, based on the
> ALB's opinion of its own targets — and then **fail it back automatically** when
> the primary recovers, *"in about 30 seconds or so"*, into a region whose
> database is now the wrong one. Split-brain by health check. See
> [[split-brain-and-fencing]].
>
> **Recommendation: run the standby endpoint at weight 0 and dial 0**, so GA
> cannot move traffic on its own, and make failover an explicit two-call
> operation (set weight, set dials). You lose automatic failover — which you did
> not want — and keep every other GA benefit.

### How a shift is actually executed

It is an `UpdateEndpointGroup` call against the `us-west-2` endpoint. No
Terraform, no DNS, no `ChangeResourceRecordSets`:

```bash
# GA's control plane is global but addressed via us-west-2.
# Endpoint group ARNs are hard-coded in the runbook, exactly as
# [[aws-route53]] says to hard-code ARC's five cluster endpoints.
aws globalaccelerator update-endpoint-group \
  --region us-west-2 \
  --endpoint-group-arn "$STANDBY_EG_ARN" \
  --traffic-dial-percentage 100 \
  --endpoint-configurations EndpointId="$STANDBY_ALB_ARN",Weight=128,ClientIPPreservationEnabled=true

aws globalaccelerator update-endpoint-group \
  --region us-west-2 \
  --endpoint-group-arn "$PRIMARY_EG_ARN" \
  --traffic-dial-percentage 0 \
  --endpoint-configurations EndpointId="$PRIMARY_ALB_ARN",Weight=0
```

Order matters: **bring the standby up before taking the primary down**, or you
briefly have no weighted healthy endpoint anywhere and GA fails open (below).

> [!note] Two calls is a real weakness versus ARC
> ARC sells `UpdateRoutingControlStates` (plural) as an **atomic multi-control
> transaction**, so you are never half-switched. GA has no equivalent: two
> endpoint groups means two API calls and a window between them. In practice the
> window is a second or two and GA's fail-*open* behaviour means the failure mode
> during it is "traffic goes somewhere" rather than "traffic goes nowhere" — but
> it is a genuine point for ARC, and the reason the order above is not optional.
> Compare [[route53-application-recovery-controller]].

## Health checks and what "30 seconds" actually means

**This is the most commonly mis-cited figure in the whole GA/DNS argument, so it
is worth being precise.** The "about 30 seconds" number is real and it is AWS's,
but it is a statement about **recovery**, not about **detection**.

### The verbatim source

From [How failover works for unhealthy endpoints](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoints-endpoint-weights.unhealthy-endpoints.html):

> When recovery occurs, that is, Regions are healthy again, Global Accelerator
> returns to regular routing behavior. This means that, typically, routing will
> start back to healthy endpoints with traffic dials that aren't set to zero in
> about 30 seconds or so. However, note that established active connections are
> not moved. They continue to route to the zero weight Region until the connection
> is reset by the client or the server, or until the client makes a new
> connection.

That is the only "30 seconds" AWS publishes on this page, and it is about
**failing back after recovery**. It is reasonable to read it as indicative of the
data plane's general reaction time, and this note does — but **it is not a
documented failover SLA and should not be quoted as one.**

### Detection time, decomposed

| Endpoint type | Who runs the check | Configurable timing | Detection time |
|---|---|---|---|
| **ALB** | **The ALB's own target-group health checks.** GA's settings are ignored | ELB target-group settings ([[aws-alb-nlb]]) | **Not published by AWS as a single figure** |
| **NLB** | The NLB's own target-group health checks | ELB target-group settings | **Not published** |
| EC2 instance / Elastic IP | **Global Accelerator**, via the Route 53 health-checker fleet | `health_check_interval_seconds` (**10 or 30, default 30**) × `threshold_count` (**default 3**) | **30 s – 90 s** |

The per-endpoint-type rule, verbatim from the
[health check options page](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoint-groups-health-check-options.html):

> The health check options that you choose in Global Accelerator do not affect
> Network Load Balancers or Application Load Balancers that you've added as
> endpoints. That is, health check options that you specify in Global Accelerator
> are used for Amazon EC2 and Elastic IP address health checks, but not for health
> checks on load balancer endpoints.

And the two health *definitions*, which differ and both matter:

> Global Accelerator considers an Application Load Balancer healthy if every
> target group has at least one healthy target.

> Global Accelerator considers a Network Load Balancer healthy if there is at
> least one healthy Availability Zone. An Availability Zone is healthy if it has a
> healthy target in all load balancer target groups that it is in.

> [!warning] GA is stricter than Route 53, and it breaks scale-to-zero standbys
> A Route 53 alias with `evaluate_target_health = true` reports an ALB with **no
> registered targets** as **healthy** ([[aws-route53]] gotcha #4). **GA reports
> the same ALB as unhealthy** — "every target group has at least one healthy
> target". GA is more correct. But it means **GA will not fail over into a standby
> that has scaled to zero**, and worse, a standby with *any* empty target group
> (a rarely-used service, a canary target group with nothing in it) makes the
> whole region unhealthy to GA. **Every target group in the standby needs at least
> one healthy pod.** That is a hard constraint on the warm-standby shape
> ([[aws-eks]]) and it is not obvious.

The NLB rule has a second-order effect flagged in [[aws-alb-nlb]]: with
cross-zone load balancing enabled, *"if every target group contains a healthy
target, Global Accelerator considers the Network Load Balancer to be healthy"* —
so enabling cross-zone changes how GA judges regional health. Also note GA
*"only considers a Network Load Balancer healthy if the number of healthy targets
meets the Network Load Balancer minimum healthy target count setting,
`minimum_healthy_targets`."*

### EC2/EIP checks use the Route 53 health-checker fleet

From the same page:

> To ensure access for health checks to complete successfully for EC2 instance or
> Elastic IP address endpoints, make sure that your router and firewall rules
> allow inbound traffic from the IP addresses associated with Amazon Route 53
> health checkers.

So the managed-prefix-list work in [[aws-route53]]
(`com.amazonaws.<region>.route53-healthchecks`) is **reused, not replaced**, if
you ever use EC2/EIP endpoints. For a pure ALB-endpoint design it is not needed
for GA — but you probably still want Route 53 health checks for *alarming*.

### Fail-open — the behaviour that must be bounded

Two verbatim statements, from the failover page and the health-check page:

> If Global Accelerator doesn't find a healthy endpoint with a weight greater than
> zero after trying the three closest endpoint groups (that is, AWS Regions), it
> routes traffic to a random endpoint in the endpoint group that is closest to the
> client. That is, it *fails open*.

> Note that if there aren't any healthy endpoints to route traffic to, Global
> Accelerator routes incoming client requests to *all* endpoints in the endpoint
> group.

> [!danger] One accelerator per pair. Never one accelerator for all six regions.
> A single accelerator holding all six endpoint groups will, under sufficient
> failure, route **European customer traffic into `us-east-1` or `ca-central-1`**.
> In this estate that is not a performance quirk — the three deployments **share
> no data** ([[research-brief]]), so it is simultaneously a "your data isn't
> here" bug **and** a data-residency breach ([[data-residency]]).
>
> **Three accelerators × $0.025/hour = $54.75/month**, versus $18.25 for one.
> **Pay the $36.50 and bound the blast radius to the pair.** With exactly two
> endpoint groups, "the three closest endpoint groups" can only ever mean "the
> two in this pair", and fail-open degrades to "send it to the pair's own
> regions", which is correct behaviour.

## The control plane — and why `us-west-2` is the interesting answer

Verbatim, from the [Region availability page](https://docs.aws.amazon.com/global-accelerator/latest/dg/preserve-client-ip-address.regions.html):

> AWS Global Accelerator is a global service. However, you must specify the US
> West (Oregon) Region (that is, specify the parameter `--region us-west-2`) in
> Regional Global Accelerator AWS CLI commands. That is, when you create
> resources, such as accelerators.

Confirmed independently by the
[AWS General Reference](https://docs.aws.amazon.com/general/latest/gr/global_accelerator.html),
which lists exactly **one** service endpoint — `globalaccelerator.amazonaws.com`
in **`us-west-2`** (plus a FIPS variant) — and by the
[Fault Isolation Boundaries whitepaper](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/appendix-b---edge-network-global-service-guidance.html),
which [[aws-regional-outages]] already summarises as "Global Accelerator:
`us-west-2`".

### Why this is good news for the US pair

The US pair is **`us-east-1` → `us-west-2`**, and [[aws-route53]] calls the
control-plane trap out as the single scariest thing in that note:

> A `us-east-1` regional impairment is simultaneously the event you are failing
> away from and the event that removes your ability to change DNS.

**Global Accelerator does not have that problem.** During a `us-east-1`
impairment you can still call `UpdateEndpointGroup` and move traffic, because the
API is served from Oregon. Route 53 accelerated recovery restores DNS
changeability — **after about 60 minutes**, four times the RTO. GA gives you the
switch immediately.

### Why it is worse news for the EU and CA pairs

Here is the honest half, and it is the same class of question
[[route53-application-recovery-controller]] asks about ARC's five-region data
plane:

| Scenario | Route 53 record change | GA traffic-dial change | ARC routing control |
|---|---|---|---|
| `us-east-1` impaired | ❌ blocked (60 min via accelerated recovery) | ✅ works (`us-west-2`) | ✅ works (5-region quorum) |
| `us-west-2` impaired | ✅ works (`us-east-1`) | ❌ **blocked** | ✅ works (quorum survives 2 endpoint losses) |
| Both impaired | ❌ | ❌ | ✅ (3 of 5) |
| **US pair failing over** (`us-east-1` → `us-west-2`) | ❌ the worst case | ✅ | ✅ |
| **EU pair, during an unrelated `us-west-2` event** | ✅ | ❌ **you cannot shift EU traffic** | ✅ |

**This is a real and under-appreciated coupling.** Adopting GA moves the EU and CA
pairs' failover switch from a dependency on `us-east-1` to a dependency on
`us-west-2`. Neither is a dependency on their own regions, and **both are single
regions**. ARC's five-region quorum is genuinely better than either — that is
exactly what the $1,825/month buys and
[[route53-application-recovery-controller]] is right to say so.

**But weigh it honestly:**

1. `us-west-2` is a mature region and has had its own event (15 December 2021 —
   see [[aws-regional-outages]]), so this is not hypothetical.
2. **Only the *control plane* is affected.** The GA **data plane** — the anycast
   advertisement and the automatic health-based failover — is global. If
   `us-east-1` dies and you cannot reach the GA API, GA's *automatic* failover
   still works, because that decision is made at the edge. What you lose is the
   ability to make a *deliberate* shift.
3. **Static stability is the mitigation, and it is free.** Pre-create both
   endpoint groups. Keep both endpoints weighted. Then the only thing you need
   the control plane for is the *deliberate* switch — and you can fall back to
   letting the automatic health-based failover do it, or to a Route 53 change,
   depending on which control plane survives.

> [!tip] The cheap resilience play: keep both switches wired
> Route 53's control plane is `us-east-1`. GA's is `us-west-2`. Keeping
> **`app.example.com` as a Route 53 record you *could* repoint** alongside the GA
> traffic dial gives you **two independent control planes in two different
> regions for $0**. It is strictly more resilient than either alone and
> considerably cheaper than ARC's five. Write it into
> [[failover-runbook-template]] as "primary switch: GA dial; fallback switch:
> repoint the Route 53 record at the standby ALB directly (accept the TTL)".

## Endpoint types — what can and cannot sit behind it

From [Requirements for resources you add as accelerator endpoints](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoints-caveats.html):

| Endpoint type | Supported? | Notes |
|---|---|---|
| **Application Load Balancer** | ✅ | *"can be internet-facing or internal"*. Dual-stack ALBs OK. **Not in a Local Zone.** |
| **Network Load Balancer** | ✅ | *"can be internet-facing or internal"*. Client IP preservation only on NLBs that support security groups, and **not with TLS termination**. |
| **EC2 instance** | ✅ | Excludes a list of older instance families (C1, CC1, CC2, CG1, CG2, CR1, CS1, G1, G2, HI1, HS1, M1, M2, M3, T1). |
| **Elastic IP address** | ✅ | **Dual-stack EIPs cannot be added.** |
| VPC subnet | ✅ — **custom routing accelerators only** | See [[#Custom routing accelerators]]. |
| **API Gateway** | ❌ **Not an endpoint type** | Verified: the supported list is ALB, NLB, EC2, EIP (and subnets for custom routing). See below. |
| CloudFront distribution | ❌ | Not in the list. GA sits *beside* CloudFront, not behind it. See [[#Global Accelerator vs CloudFront]]. |
| S3 bucket, Lambda, ECS service directly | ❌ | Front them with an ALB/NLB. |

### API Gateway — verified negative, and what to do instead

**API Gateway is not a supported Global Accelerator endpoint type.** The
endpoint-requirements page enumerates ALB, NLB, EC2 and Elastic IP for standard
accelerators and VPC subnets for custom routing accelerators; API Gateway does not
appear, and there is no `endpoint_id` form that accepts an API Gateway ARN in the
[`aws_globalaccelerator_endpoint_group` provider docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/globalaccelerator_endpoint_group)
either (*"If the endpoint is a Network Load Balancer or Application Load
Balancer, this is the ARN of the resource. If the endpoint is an Elastic IP
address, this is the Elastic IP address allocation ID."*).

**This is a real constraint for this estate**, because [[aws-api-gateway]] is a
service in scope. If any public entry point is an API Gateway REST/HTTP API, then
**that entry point cannot use GA as its failover mechanism** and must fall back to
Route 53 (API Gateway's own custom-domain failover / regional endpoints) or
CloudFront. Practical options:

1. **Put the API behind an ALB instead** (e.g. an ALB → private API Gateway via
   VPC link, or skip API Gateway for that route). Then GA works.
2. **Use a split design**: GA for the ALB-fronted services, Route 53 failover for
   the API Gateway custom domains. Two mechanisms, two runbooks — a real
   operational cost, and an argument for consolidating on Route 53 if API Gateway
   carries most of the traffic.
3. **CloudFront in front of API Gateway**, with origin groups doing the failover
   ([[aws-cloudfront]]) — subject to that note's `GET`/`HEAD`/`OPTIONS`
   limitation.

**Get the answer to "how much of the public surface is API Gateway?" before
committing to GA estate-wide.** Raised in [[#Open questions]].

### Internal load balancers are supported — a genuinely useful detail

*"An Application Load Balancer endpoint can be internet-facing or internal."* This
means GA can front an **internal** ALB, so the anycast IPs become the *only*
public entry point and the ALBs themselves never need public addresses. For a
security posture that wants no internet-facing regional load balancers
([[security-posture-of-the-standby]]), this is a clean answer that CloudFront
only matches via VPC Origins.

### Client IP preservation — and the Terraform trap

`client_ip_preservation_enabled` is per-endpoint and **defaults to `false`** in
the provider. With it off, your ALB access logs and WAF rules see *Global
Accelerator's* address, not the client's — which breaks IP-based WAF rules, rate
limiting and geo rules ([[aws-waf-shield]]). Turn it on. But note the warning in
the [provider docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/globalaccelerator_endpoint_group),
verbatim:

> When client IP address preservation is enabled, the Global Accelerator service
> creates an EC2 Security Group in the VPC named `GlobalAccelerator` that must be
> deleted (potentially outside of Terraform) before the VPC will successfully
> delete. If this EC2 Security Group is not deleted, Terraform will retry the VPC
> deletion for a few minutes before reporting a `DependencyViolation` error. This
> cannot be resolved by re-running Terraform.

**That will break every ephemeral/sandbox environment that `terraform destroy`s
its VPC.** Flag it in the cookiecutter template alongside the
accelerated-recovery zone-deletion trap from [[aws-route53]]. Also note from the
endpoint-requirements page: *"To add an endpoint to a dual-stack accelerator, the
endpoint must have client IP address preservation enabled."*

### Do not also serve the same endpoints directly

> When you add resources as endpoints behind Global Accelerator, we recommend
> that you don't also send traffic directly to the same endpoints over the
> internet. Sending direct traffic can lead to connection collision issues.

Relevant here because a **migration** naturally runs both paths at once (see
[[#Migration path from single-region]]). AWS also recommends *"disable
cross-zone traffic for [NLBs] to avoid connection collisions"* — which directly
contradicts the NLB health-evaluation advice above. That tension is real; for ALB
endpoints it does not arise.

## Client affinity, and what happens to in-flight connections

From [How client affinity works](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-listeners-client-affinity.html),
verbatim:

> By default, client affinity for a standard listener is set to **None** and
> Global Accelerator distributes traffic equally between the endpoints in the
> endpoint groups for the listener.

> If you configure client affinity for your Global Accelerator resource to be
> **None**, Global Accelerator uses the 5-tuple properties—source IP, source port,
> destination IP, destination port, and protocol—to select the hash value.

> If you want to maintain client affinity by routing a specific user—identified by
> their source IP address—to the same endpoint each time they connect, set client
> affinity to **Source IP**. When you specify this option, Global Accelerator uses
> the 2-tuple properties—source IP and destination IP—to select the hash value and
> route the user to the same endpoint whenever they connect. Additionally, Global
> Accelerator honors client affinity by routing all connections with the same
> source IP address to the same endpoint group.

And the caveat that matters for DR:

> On occasion, network maintenance or disruptions created by variations in
> internet traffic routing can cause client traffic to shift to different Global
> Accelerator edge locations. When this happens, if the edge location that now
> serves the client traffic prefers a different AWS Region, then client affinity
> is not guaranteed to be maintained.

### What to set here

**`client_affinity = "NONE"` (the default) for this architecture.** Reasoning:

- With **two** endpoint groups and the standby dialled to 0, there is only ever
  one region receiving traffic, so affinity buys nothing in normal running.
- During a **canary shift** (dial 90/10), `SOURCE_IP` affinity means a given
  client consistently lands in one region rather than bouncing — which sounds
  good, but with **no data shared between regions** and a 2-hour-stale standby,
  *you do not want a subset of users pinned to the stale replica for a sustained
  period*. A short, uniformly-sampled canary is safer than a sticky one.
- `SOURCE_IP` hashes on **client IP**, so any large NAT'd corporate customer
  lands entirely in one region. That is lumpy load, not affinity.
- **If the application is stateful** (in-memory sessions, sticky WebSockets), set
  `SOURCE_IP` — but the real fix is [[aws-elasticache-redis]]-backed sessions,
  not edge hashing.

### In-flight connections at a shift — the precise answer

| Connection state at the moment you change the dial | What GA does |
|---|---|
| **New** connection | Routed by the new dial/weights. Immediate. |
| **Established and active** (data flowing) | **Stays on the old endpoint.** Not moved. Indefinitely, as long as it does not idle out. |
| **Established and idle** | Stays on the old endpoint until the idle timeout: **340 s TCP / 30 s UDP**, not tunable. |
| Endpoint marked **unhealthy** | Established connections **still** continue to it until idle timeout. |
| Endpoint **removed from the accelerator entirely** | Same — *"even if the endpoint is marked as unhealthy or if it is removed from the accelerator"*. |

**Two operational consequences:**

1. **You cannot drain clients by deleting the endpoint.** Removing it from the
   endpoint group does not hang up existing connections. To actually stop the old
   region serving, you must fence at the region — stop the pods, revoke the
   security group, deregister the targets ([[split-brain-and-fencing]]).
2. **"Traffic moved" is not the same as "the old primary stopped writing".**
   GA gives you a fast *traffic* switch and **zero** help with fencing. That is
   the same gap [[split-brain-and-fencing]] identifies for every other mechanism;
   GA does not close it and must not be sold as if it does.

## `ca-west-1` and `ca-central-1` — the parity check

**Explicitly checked.** From the
[AWS Region availability page](https://docs.aws.amazon.com/global-accelerator/latest/dg/preserve-client-ip-address.regions.html),
which lists supported endpoint Regions with AZ exceptions noted inline:

| Region | In scope as | Listed? | AZ exception |
|---|---|---|---|
| `eu-west-1` Ireland | EU primary | ✅ | none |
| `eu-west-2` London | EU standby | ✅ | none |
| `us-east-1` N. Virginia | US primary | ✅ | none |
| `us-west-2` Oregon | US standby | ✅ | none |
| **`ca-central-1` Montreal** | **CA primary** | ✅ | **"except AZ `cac1-az3`"** |
| **`ca-west-1` Calgary** | **CA standby** | ✅ | **none** |

**`ca-west-1` passes.** Calgary was added as a supported endpoint Region on
[25 April 2024](https://aws.amazon.com/about-aws/whats-new/2024/04/aws-global-accelerator-endpoints-calgary-region).
Given the register in [[region-pair-selection]] — Cognito MRR, OpenSearch CCR,
Managed Grafana and Backup Audit Manager all missing in Calgary — **this is one of
the few clean passes in the CA pair's whole design, and it should be recorded as
such.** If the CA pair needs a failover mechanism that Calgary actually supports,
GA is available to it.

> [!warning] The exception is on the *primary*, and that is the one to check
> **`ca-central-1` is supported "except AZ `cac1-az3`".** If the live Montreal
> deployment has subnets in `cac1-az3`, resources in that AZ **cannot** be GA
> endpoints. This is an **AZ ID**, not an AZ name — `cac1-az3` maps to a
> different letter (`ca-central-1a/b/d`) in every account, so you cannot answer
> this from the console's letter labels.
>
> ```bash
> aws ec2 describe-availability-zones --region ca-central-1 \
>   --query 'AvailabilityZones[].[ZoneName,ZoneId]' --output table
> ```
>
> For an **ALB** endpoint the practical impact is limited — the ALB is the
> endpoint, not its subnets, and an ALB spanning `cac1-az1`/`az2` is fine. It
> bites hardest for **EC2 instance or Elastic IP endpoints**. Confirm before the
> CA pair's design is signed off. Also note `ca-central-1` has historically been
> a 3-AZ region where `cac1-az3` is not offered to all accounts at all — check
> what you actually have.

**Note the contrast with [[aws-cognito]]:** Calgary's problem is usually
"feature X is not a supported Region". Here Calgary is fine and **Montreal**
carries the asterisk. Worth adding to [[region-pair-selection]] because it breaks
the pattern everyone has internalised.

## Still to research

This note is `partial`. Written so far: the anycast thesis and head-to-head,
traffic dials/weights, health checks and the real failover-speed figures, the
`us-west-2` control plane, endpoint types (including the verified API Gateway
negative), client affinity and in-flight connections, and the `ca-west-1` /
`ca-central-1` parity check.

Still to write:

- **Cost** — full DT-Premium model for this estate at several egress volumes, on
  one scale with the ~$0.75/month Route 53 STOP pattern and the $1,825/month ARC
  cluster.
- **Global Accelerator vs CloudFront** — the overlap, and when both is right.
- **BYOIP** — whether bringing your own IPs changes the calculus.
- **Custom routing accelerators** — brief.
- **WAF and Shield Advanced** — the verified answer on whether WAF works in front
  of GA.
- **Terraform implementation** — module signature for the cookiecutter monorepo.
- **Migration path, failover procedure, failback, gotchas.**
- **The verdict — the three-way decision table.**

## Sources

- [How AWS Global Accelerator works](https://docs.aws.amazon.com/global-accelerator/latest/dg/introduction-how-it-works.html) — anycast static IPs and the "remain assigned for as long as it exists" guarantee; **idle timeout 340 s TCP / 30 s UDP, not customisable**; "continues to direct traffic for established connections… even if the endpoint is marked as unhealthy or if it is removed from the accelerator"; traffic-dial vs endpoint-weight semantics verbatim; standard vs custom routing accelerators; TCP termination at the edge; "if Global Accelerator doesn't have any healthy endpoints… it routes requests to all endpoints"; ICMP and MTU behaviour; two IPv4 addresses, four for dual-stack.
- [How failover works for unhealthy endpoints](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoints-endpoint-weights.unhealthy-endpoints.html) — the traffic-dial-is-ignored-during-failover rule verbatim; the "three closest endpoint groups" then **fail open** behaviour; and the **"about 30 seconds or so"** figure, which is a statement about *recovery*, plus "established active connections are not moved".
- [Ensure health check access for your accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoint-groups-health-check-options.html) — GA health-check settings **do not apply to ALB/NLB endpoints**; the ALB definition ("every target group has at least one healthy target"); the NLB definition (at least one healthy AZ, plus `minimum_healthy_targets`); EC2/EIP checks come from the **Route 53 health-checker IP ranges**; health check port/protocol/interval/threshold options.
- [Requirements for resources you add as accelerator endpoints](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoints-caveats.html) — the definitive supported-endpoint list (ALB, NLB, EC2, EIP) and therefore the **verified absence of API Gateway**; internal ALB/NLB both supported; no Local Zones; excluded EC2 instance families; dual-stack requires client IP preservation; dual-stack EIPs unsupported; the "don't also send traffic directly to the same endpoints" connection-collision warning; cross-account endpoint rules.
- [How client affinity works in Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-listeners-client-affinity.html) — `None` (5-tuple) is the default, `Source IP` (2-tuple) verbatim, and the caveat that affinity is not guaranteed when edge-location routing shifts.
- [AWS Region availability for AWS Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/preserve-client-ip-address.regions.html) — the endpoint-Region table used for the parity check: **`ca-west-1` supported with no AZ exception**, **`ca-central-1` "except AZ `cac1-az3`"**, all four EU/US regions clean; and the verbatim *"you must specify the US West (Oregon) Region… in Regional Global Accelerator AWS CLI commands"*.
- [AWS Global Accelerator endpoints and quotas (General Reference)](https://docs.aws.amazon.com/general/latest/gr/global_accelerator.html) — confirms a **single global service endpoint in `us-west-2`**; quotas: 20 standard accelerators per account (adjustable), 10 custom routing accelerators, 42 endpoint groups per accelerator (not adjustable), 10 endpoints per endpoint group, 10 listeners per accelerator, 10 port ranges per listener.
- [AWS Global Accelerator pricing](https://aws.amazon.com/global-accelerator/pricing/) — **$0.025 per accelerator-hour**, charged "for every full or partial hour… until it is deleted"; the DT-Premium definition verbatim (rate depends on **source Region** and **destination edge location**); "You will only be charged DT-Premium in the dominant data transfer direction"; the full per-GB rate matrix used in the cost model.
- [AWS Global Accelerator now supports endpoints in the Canada West (Calgary) Region](https://aws.amazon.com/about-aws/whats-new/2024/04/aws-global-accelerator-endpoints-calgary-region) — the 25 April 2024 date for `ca-west-1` support.
- [AWS Fault Isolation Boundaries — Appendix B](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/appendix-b---edge-network-global-service-guidance.html) — control-plane home Regions: Route 53 and CloudFront in `us-east-1`, **Global Accelerator in `us-west-2`**.
- [`aws_globalaccelerator_accelerator` (Terraform Registry)](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/globalaccelerator_accelerator) — `ip_address_type` (`IPV4`/`DUAL_STACK`), `ip_addresses` for BYOIP ("1 or 2 IPv4 addresses"), `enabled`, the `attributes` flow-logs block, exported `dns_name`/`dual_stack_dns_name`/`ip_sets`, and the fixed **`hosted_zone_id` = `Z2BJ6XQ5FK7U4H`** used for Route 53 alias records.
- [`aws_globalaccelerator_listener` (Terraform Registry)](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/globalaccelerator_listener) — `client_affinity` (`NONE` default / `SOURCE_IP`), `protocol` (`TCP`/`UDP`), `port_range`.
- [`aws_globalaccelerator_endpoint_group` (Terraform Registry)](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/globalaccelerator_endpoint_group) — `traffic_dial_percentage` (default 100), `health_check_interval_seconds` (**10 or 30, default 30**), `threshold_count` (**default 3**), `endpoint_configuration.weight`, `client_ip_preservation_enabled` (default `false`) **and the `GlobalAccelerator` security group that blocks VPC deletion**, `port_override`, `attachment_arn` for cross-account.
- [Moura, Heidemann, Schmidt, Hardaker — "Cache Me If You Can: Effects of DNS Time-to-Live", ACM IMC 2019](https://ant.isi.edu/~johnh/PAPERS/Moura19b.html) — peer-reviewed measurement that real resolver cache lifetimes diverge from configured TTLs; the evidence behind "the TTL is advisory", which is the problem GA removes.

## Related notes

[[aws-route53]] · [[aws-alb-nlb]] · [[aws-cloudfront]] · [[aws-api-gateway]] · [[aws-acm]] · [[aws-waf-shield]] · [[aws-eks]] · [[route53-application-recovery-controller]] · [[failover-orchestration]] · [[split-brain-and-fencing]] · [[observability-multi-region]] · [[region-pair-selection]] · [[provider-aliases-vs-separate-stacks]] · [[module-patterns]] · [[cost-model]] · [[aws-regional-outages]] · [[lessons-and-antipatterns]] · [[data-residency]]
