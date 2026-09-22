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

## Cost

**This is the section that decides the note.** Everything above says GA is the
technically cleanest failover mechanism in the vault. The only question left is
whether it is worth paying for, and that turns on one number most people never
look up.

### The two charges

From the [Global Accelerator pricing page](https://aws.amazon.com/global-accelerator/pricing/):

**1. The fixed accelerator fee**, verbatim:

> For every full or partial hour when an accelerator runs in your account, you
> are charged $0.025 until it is deleted.

| | Hourly | Monthly (730 h) | Annual |
|---|---|---|---|
| One accelerator | $0.025 | **$18.25** | $219 |
| **Three** (one per pair — see the fail-open argument) | $0.075 | **$54.75** | $657 |

Note the words *"full or partial hour"* and *"until it is deleted"*. An
accelerator with `enabled = false` **still bills**. There is no scale-to-zero,
and disabling it during a change freeze saves nothing. Also note it is per
*accelerator*, not per listener or per endpoint group — the three-accelerator
recommendation costs exactly $36.50/month more than one.

**2. DT-Premium**, and this is the one that matters. Verbatim:

> This is a rate per gigabyte of data transferred over the AWS network. The
> DT-Premium rate depends on the AWS Region (source) that serves the request and
> the AWS edge location (destination) where the responses are directed.

> The DT-Premium fee is in addition to normal EC2 Data Transfer Out fees charged
> for your application endpoints running in AWS Region(s).

> You will only be charged DT-Premium in the dominant data transfer direction.

Three things to take from that:

- **It is a surcharge, not a replacement.** You still pay ordinary EC2/ELB data
  transfer out. DT-Premium is stacked on top.
- **It is billed on *all* traffic**, every month, whether or not you ever fail
  over. This is the structural difference from every other mechanism in the
  vault: Route 53 and CloudFront origin groups cost the same at 0 failovers and
  at 12; GA's bill scales with production volume.
- **The rate is a matrix**, not a number, and "which cell am I in" is the whole
  cost question.

### The rate matrix — and the cell this estate lands in

The published matrix is source-Region-group × destination-edge-group. The rows
and columns relevant here:

| Source Region group | → US / Canada / Mexico edges | → Europe / Israel / Turkey edges | → Australia / NZ edges |
|---|---|---|---|
| **US & Canada** | **$0.015/GB** | $0.015/GB | $0.105/GB |
| **Europe & Israel** | $0.015/GB | **$0.015/GB** | $0.105/GB |

Now map the estate onto it ([[research-brief]]):

| Pair | Source Regions | Customers served from it | Rate cell | **$/GB** |
|---|---|---|---|---|
| **EU** | `eu-west-1` / `eu-west-2` | European | Europe & Israel → Europe/Israel/Turkey | **$0.015** |
| **US** | `us-east-1` / `us-west-2` | North American | US & Canada → US/Canada/Mexico | **$0.015** |
| **CA** | `ca-central-1` / `ca-west-1` | Canadian | US & Canada → US/Canada/Mexico | **$0.015** |

> [!important] This estate is in the cheapest band of the matrix, and that is not luck — it is the architecture
> The reason GA is usually rejected on cost is the far right of that matrix:
> serving Australian users from a US region is **$0.105/GB, seven times** the
> rate here. This estate **never does that**. It runs three geographically
> self-contained deployments, each serving its own continent, with no data
> sharing between them. **The same property that makes the estate awkward
> (three separate products) makes GA cheap (three in-continent accelerators).**
>
> This is the direct, evidence-based answer to the objection in
> [[aws-cloudfront]], which cites the "$0.007–$0.105/GB" range and concludes the
> per-GB premium is disqualifying. **At the top of that range it would be. This
> estate sits at $0.015, near the bottom.** See
> [[#Engaging with the CloudFront note's argument]].

### The model

**There is no published or internal egress figure for this estate.** Nothing in
the vault states monthly data transfer out, so the honest thing is to model a
range and hand the team the multiplier rather than invent a volume. Raised in
[[#Open questions]]. Using 1 TB = 1,024 GB and 730 hours/month:

**Per pair, per month, DT-Premium only, at $0.015/GB:**

| Egress per pair / month | DT-Premium |
|---|---|
| 1 TB | $15.36 |
| 5 TB | $76.80 |
| **10 TB** | **$153.60** |
| 25 TB | $384.00 |
| 50 TB | $768.00 |
| 100 TB | $1,536.00 |

**Whole estate — three accelerators at $54.75/month plus DT-Premium on all three
pairs** (assuming equal volume, which it will not be — the EU and US pairs
almost certainly dwarf CA):

| Egress per pair | Accelerator fees | DT-Premium | **Total / month** | **Total / year** |
|---|---|---|---|---|
| 1 TB | $54.75 | $46.08 | **$100.83** | $1,210 |
| 5 TB | $54.75 | $230.40 | **$285.15** | $3,422 |
| **10 TB** | $54.75 | $460.80 | **$515.55** | **$6,187** |
| 25 TB | $54.75 | $1,152.00 | **$1,206.75** | $14,481 |
| 50 TB | $54.75 | $2,304.00 | **$2,358.75** | $28,305 |
| 100 TB | $54.75 | $4,608.00 | **$4,662.75** | $55,953 |

### On one scale with the other two mechanisms

This is the comparison the vault has been missing. All figures monthly, whole
estate, all three pairs.

| Mechanism | Fixed | Variable | **At 10 TB/pair/month** | Source |
|---|---|---|---|---|
| **Route 53 failover records, alias + `evaluate_target_health`** | $0 | $0.40/M queries | **≈ $0–2** | [[aws-route53]] |
| **Route 53 STOP pattern** (one inverted non-AWS-endpoint health check per pair) | $0.75/check/mo | — | **≈ $2.25** | [[aws-route53]], [[route53-application-recovery-controller]] |
| **Route 53 with fast (10 s) health checks**, 2 per pair | ~$1.50/check/mo | — | ≈ $9 | [[aws-route53]] |
| **CloudFront origin group + KeyValueStore flag** | $0 | Functions $0.10/M invocations (2 M/mo free), KVS reads $0.03/M | **≈ $0–5 incremental** | [[aws-cloudfront]] |
| **Global Accelerator** | **$54.75** | **$0.015/GB on all traffic** | **≈ $516** | this note |
| **ARC routing-control cluster** | **$1,825** (one cluster serves all three pairs) | — | **$1,825** | [[route53-application-recovery-controller]] |
| ARC Region switch plan | $70/plan/mo | — | $70–210 | [[route53-application-recovery-controller]] |

**The ordering is the finding: GA is roughly 200× the Route 53 pattern and
roughly 3.5× cheaper than an ARC cluster.** It is genuinely in between, not at
either extreme. That is a more interesting position than either the "GA is too
expensive" line in [[aws-cloudfront]] or the "GA is the clean answer" line in
this note's own TL;DR would suggest on its own.

### The crossover with ARC

At what volume does GA cost more than the $1,825/month ARC cluster?

```
($1,825 − $54.75) ÷ $0.015/GB = 118,017 GB ≈ 115 TB total across the estate
                                           ≈ 38 TB per pair per month
```

**Below ~38 TB/pair/month, GA is cheaper than an ARC routing-control cluster and
strictly better at the job** (ARC only flips a health check; GA also removes the
DNS cache problem, which ARC does nothing about). Above it, ARC is cheaper —
though by then you would be questioning both against a $2–5/month Route 53
pattern.

### Levers, if the number comes back too big

1. **Do not put static assets behind GA.** DT-Premium is per-GB on everything
   that traverses the accelerator. Images, video, JS bundles and downloads
   should go to CloudFront, which caches them at the edge and never touches the
   accelerator. **GA for the API, CloudFront for the assets** — this is not a
   compromise, it is the AWS-recommended split (see
   [[#Global Accelerator vs CloudFront]]) and it can remove the large majority
   of the byte volume from the GA bill.
2. **One accelerator for the pair that needs it, not three.** If only the US
   pair has the JVM-client problem, buy GA for the US pair and leave EU and CA
   on Route 53. $18.25 + US DT-Premium. Split mechanisms cost operational
   simplicity — see [[#The verdict — the three-way decision table]] — but it is a
   real option.
3. **Check the dominant direction.** *"You will only be charged DT-Premium in the
   dominant data transfer direction."* For an API that ingests more than it
   emits (log shipping, uploads, telemetry), the dominant direction is *inbound*
   — which is normally free on standard AWS data transfer but is **not** free on
   DT-Premium. Do not assume the billable volume equals your current egress
   bill; for an upload-heavy workload it could be larger.
4. **Compression, HTTP/2, and response-size hygiene** move the needle linearly
   here in a way they do not for a fixed-fee mechanism.

### What GA does *not* add to the bill

- **No charge per listener, endpoint group or endpoint.**
- **No charge for the static IPs** (unlike an idle Elastic IP).
- **No health-check charge** for ALB/NLB endpoints — GA uses the load balancer's
  own checks, which you already pay for ([[aws-alb-nlb]]).
- **No cross-region data transfer** between endpoint groups: GA sends a
  connection to *one* region, it does not mirror.
- **Route 53 does not get cheaper.** You keep the zone and the alias record;
  you may be able to delete the health checks, which is a saving of single-digit
  dollars.

## Global Accelerator vs CloudFront

These two are constantly confused because both are "AWS global edge service that
makes things faster and can fail over". They are not alternatives in the general
case — they are **different layers** — but in *this* estate they genuinely
compete for the same job, so the comparison has to be made properly.

### The structural difference in one line

**CloudFront is a layer-7 HTTP cache. Global Accelerator is a layer-4 TCP/UDP
proxy that does not cache anything.** From the AWS blog
[Well-Architecting online applications with CloudFront and AWS Global Accelerator](https://aws.amazon.com/blogs/networking-and-content-delivery/well-architecting-online-applications-with-cloudfront-and-aws-global-accelerator/),
verbatim:

> Customers use Amazon CloudFront for most HTTP(S) based Web applications.

> [Global Accelerator] operates at layer 4 of the OSI model, it can be used with
> any TCP/UDP application.

> AWS WAF rules are enforced in CloudFront PoPs, which allows HTTP(S) inspection
> at the scale of millions of requests per second.

That last quote is doing a lot of work and is picked up in
[[#Does AWS WAF work in front of Global Accelerator?]].

### Feature by feature

| | **CloudFront** | **Global Accelerator** |
|---|---|---|
| OSI layer | **7** — understands HTTP methods, headers, cookies, paths | **4** — TCP/UDP only, no HTTP semantics |
| Caching | **Yes, the whole point.** Cacheable bytes never reach the origin | **None.** Every byte traverses to the region and is billed DT-Premium |
| Client-facing address | A **name** (`d111.cloudfront.net`) resolving to changing edge IPs | **Two static anycast IPs, permanent** |
| Static IPs for firewall allow-lists | **No** (CloudFront anycast static IPs exist as a paid, limited-availability feature; not the default) | **Yes, by default, free** |
| Protocols | HTTP/HTTPS (plus WebSocket over HTTP upgrade, gRPC) | **Any TCP or UDP** |
| Edge TCP termination / congestion optimisation | Yes | Yes |
| AWS WAF attachable at the edge service itself | **Yes** — CloudFront is a WAF-protectable resource | **No** — see below |
| Shield Advanced protectable | Yes | **Yes** — standard accelerators |
| Failover mechanism | Origin group (**`GET`/`HEAD`/`OPTIONS` only**) or a CloudFront Function + KeyValueStore flag | Traffic dial / endpoint weight, all methods, all protocols |
| Failover granularity | Per request, per behaviour | Per **connection** |
| Cost shape | **Volume-priced but cheaper than origin egress**, and cache hits cost nothing at the origin | **Surcharge on top of origin egress**, every byte |
| API Gateway as a backend | **Yes**, as a custom origin | **No** — verified, see [[#Endpoint types — what can and cannot sit behind it]] |
| Already in this estate | **Yes** ([[aws-cloudfront]]) | No |

### When each is right

**CloudFront is right when:**

- The workload is HTTP(S) — which for this estate's web tier it is.
- There is cacheable content. Every cache hit is a byte that never leaves the
  region and never gets billed.
- You want AWS WAF evaluated at the edge, before traffic enters a Region.
- The failover switch can be an origin-selection decision (the KVS flag pattern
  in [[aws-cloudfront]]).

**Global Accelerator is right when:**

- The protocol is **not HTTP** — raw TCP, UDP, MQTT, game traffic, VoIP,
  database protocols, syslog. CloudFront simply cannot carry these.
- **Clients hold fixed IPs**: B2B customers with firewall allow-lists, IoT
  fleets, embedded devices, anything with a hard-coded address.
- **The client-side DNS cache is the failover risk you cannot control** — the
  JVM problem in [[aws-route53]]. This is the case that matters here.
- Content is **not cacheable at all** (a pure write-heavy API), so CloudFront's
  main economic advantage evaporates and its only contribution is TLS
  termination and routing.

### Using both is legitimate — and it is the cheapest version of adopting GA

**Nothing stops you running CloudFront for one hostname and GA for another.**
They are independent ingress paths onto the same regional ALBs. The natural
split for this estate:

| Hostname | Ingress | Failover mechanism | Why |
|---|---|---|---|
| `www.example.com`, `static.example.com`, assets | **CloudFront** | Origin group + KVS flag | Cacheable; keeps bulk bytes off the DT-Premium meter |
| `api.example.com` (non-cacheable, JVM SDK clients, B2B allow-lists) | **Global Accelerator** | Traffic dial | Kills the DNS-cache tail on exactly the traffic that has it |

This split is what makes the cost lever in [[#Cost]] real: **the DT-Premium bill
is only charged on the bytes that actually traverse the accelerator**, so moving
static assets to CloudFront can remove most of the volume from GA's bill while
keeping GA's anycast property where it counts.

> [!warning] Stacking them — client → GA → CloudFront — is not supported
> **A CloudFront distribution is not a valid Global Accelerator endpoint.** The
> supported list is ALB, NLB, EC2 and Elastic IP
> ([endpoint requirements](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoints-caveats.html)).
> You cannot put GA in front of CloudFront.
>
> The **other** order — client → CloudFront → GA (the accelerator's DNS name as a
> CloudFront custom origin) — is not prohibited by anything found in the docs, and
> the accelerator does get a public DNS name of the form
> `a1234567890abcdef.awsglobalaccelerator.com`
> ([DNS addressing](https://docs.aws.amazon.com/global-accelerator/latest/dg/dns-addressing-custom-domains.dns-addressing.html)),
> which satisfies CloudFront's custom-origin requirement on paper. **But no AWS
> documentation was found that endorses this configuration, and no public case
> study of it was found either.** It would also mean paying DT-Premium on
> origin-fetch traffic to buy an anycast property the client never sees — the
> client is talking to CloudFront. **Do not do it.** Recorded as a
> researched negative rather than a recommendation.

### Engaging with the CloudFront note's argument

[[aws-cloudfront]] closes with a three-candidate table and recommends a
**layered stack** — KVS flag as the switch, a static origin group as a free read-path
safety net, dormant Route 53 records to route around CloudFront itself — and
explicitly says **"Do not adopt Global Accelerator for this."** Its three reasons,
answered one at a time:

**1. "The estate already has CloudFront in front of the application, so GA would
be a fourth layer or a parallel ingress path."**

**Largely correct, and this is the strongest of the three.** Architectural
simplicity is a real asset at 3am and the brief explicitly warns against adding
services. But two qualifications:

- It assumes CloudFront fronts *everything*. **That is an assumption, not a
  verified fact about this estate** — [[aws-cloudfront]] raises "does CloudFront
  actually front the app today?" as an open question itself. If some ingress
  paths (an internal-facing API, a partner integration, an NLB for a non-HTTP
  protocol) do not go through CloudFront, those paths need a mechanism and
  CloudFront is not it.
- GA as a *parallel* path for one hostname is not a "fourth layer". It is a
  second front door, and the split above shows how to keep it narrow.

**2. "The DT-Premium is a per-GB tax on all traffic, forever, to buy a failover
property you can get for the price of a KVS write."**

**This is the reason to rebut, and the rebuttal is a number.** That note cites
the range **$0.007–$0.105/GB** and concludes the premium is disqualifying. The
range is accurate; the conclusion does not follow **for this estate**, because
all three pairs serve their own continent and therefore sit in the
**$0.015/GB** cell, near the bottom of the range and **seven times cheaper than
the Australia cell that makes the argument look decisive**
([[#The rate matrix — and the cell this estate lands in]]).

At 10 TB/pair/month that is **~$516/month for the whole estate** — more than
CloudFront's ~$0 incremental, certainly, but **less than a third of the ARC
cluster the vault has already discussed seriously**, and a small fraction of what
the standby compute in three regions will cost. *"Forever"* is right; *"tax"*
overstates it at this rate. **The honest framing is not "GA is too expensive", it
is "GA costs real money to buy a property CloudFront gives you free on HTTP
traffic" — which is a weaker but still sufficient argument.**

**3. "It does not solve the hard part — promoting the database, scaling the
standby, fencing the old primary."**

**Completely correct and this note agrees without reservation.** GA moves traffic
and does nothing else. [[#Client affinity, and what happens to in-flight connections]]
says the same thing about fencing, independently. Neither GA nor CloudFront nor
Route 53 is the reason this programme is hard.

**Verdict on the disagreement:** [[aws-cloudfront]]'s *recommendation* survives —
for the HTTP web tier fronted by CloudFront, GA is not worth adopting — but its
*reasoning* needs the correction in point 2, because the cost objection is much
weaker here than that note states and should not be the headline reason. The
right headline reason is point 1: **CloudFront already delivers the
no-client-DNS-cache property for HTTP traffic, free.** See
[[#The verdict — the three-way decision table]].

## Does AWS WAF work in front of Global Accelerator?

**No. Verified negative, and this answer is needed by [[aws-waf-shield]].**

### The verification

The [AWS WAF developer guide](https://docs.aws.amazon.com/waf/latest/developerguide/waf-chapter.html)
enumerates the protectable resource types, verbatim:

> You can protect the following resource types:
> - Amazon CloudFront distribution
> - Amazon API Gateway REST API
> - Application Load Balancer
> - AWS AppSync GraphQL API
> - Amazon Cognito user pool
> - AWS App Runner service
> - Amazon Bedrock AgentCore Gateway
> - AWS Verified Access instance
> - AWS Amplify

**Global Accelerator is not in that list, and there is no `WebACLAssociation`
form that takes an accelerator ARN.** This is a structural consequence of the
layer-4/layer-7 split: GA does not parse HTTP, so there is nothing for a web ACL
to inspect at the accelerator.

### What you do instead

Put the web ACL on the **ALB endpoint**. The accelerator forwards the connection
to the ALB, the ALB's web ACL evaluates the request, and the request is allowed
or blocked there. AWS describes exactly this pattern in the re:Post article
["Use AWS WAF with Global Accelerator to block Layer 7 HTTP method and headers from access to application"](https://repost.aws/knowledge-center/globalaccelerator-aws-waf-filter-layer7-traffic).

**This works, but it is materially worse than CloudFront's position, in three
specific ways — and [[aws-waf-shield]] should say all three:**

| | WAF on CloudFront | WAF on the ALB behind GA |
|---|---|---|
| Where blocking happens | **At the edge PoP**, before traffic enters a Region | **Inside the Region**, after the connection has crossed the AWS backbone |
| Who pays for blocked traffic | Nobody much | **You** — blocked requests still consumed DT-Premium and ALB LCUs on the way in |
| Scale | *"HTTP(S) inspection at the scale of millions of requests per second"* | Bounded by the ALB |
| Regional failover | One web ACL on the distribution | **Two web ACLs**, one per region, that must be kept identical or the standby behaves differently under attack |

That last row is a genuine multi-region gotcha and belongs in
[[aws-waf-shield]]: **with GA, the standby region needs its own regional-scope
web ACL, its own IP sets and its own rate-limit state.** Rate-based rules count
per web ACL, so **an attacker's rate-limit counter resets to zero the moment you
fail over.** IP sets must be replicated by Terraform (they are regional
resources with regional ARNs). CloudFront's single `CLOUDFRONT`-scope web ACL
has neither problem — though it buys that with a **`us-east-1` control-plane
dependency** ([[aws-route53]], [[aws-cloudfront]]), which GA's regional web ACLs
do not have. **That is a real, if small, point in GA's favour and it is the only
one in this section.**

### Client IP preservation is a hard prerequisite for any of it

If `client_ip_preservation_enabled = false`, the ALB — and therefore the web ACL
— sees **Global Accelerator's** address as the source, not the client's. Every
IP-based rule, every geo-match rule, every rate-based rule then operates on the
wrong address, and a rate-based rule will either block everyone or nobody. See
[[#Client IP preservation — and the Terraform trap]]. **Turning it on is not
optional if WAF is in the path**, and it drags in the `GlobalAccelerator`
security-group/`terraform destroy` trap documented there.

### Shield Advanced — verified positive

The [list of resources Shield Advanced protects](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary-protected-resources.html)
includes, verbatim:

> - Amazon CloudFront distributions...
> - Amazon Route 53 hosted zones.
> - **AWS Global Accelerator standard accelerators.**
> - Amazon EC2 Elastic IP addresses...
> - Amazon EC2 instances, through association to Amazon EC2 Elastic IP addresses.
> - The following Elastic Load Balancing (ELB) load balancers: Application Load
>   Balancers. Classic Load Balancers. Network Load Balancers, through
>   associations to Amazon EC2 Elastic IP addresses.

Two things follow:

1. **Standard accelerators are protectable; the wording says "standard", so do
   not assume custom routing accelerators are** ([[#Custom routing accelerators]]).
2. **Shield Standard protects GA regardless** — all AWS customers get Shield
   Standard at no charge, and GA's anycast footprint absorbs volumetric attacks
   across the whole edge network by design. Shield **Advanced** is a separate
   subscription decision that belongs in [[aws-waf-shield]], not here.

Also note, verbatim from the
[Shield Advanced overview](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary.html):

> Your Shield Advanced subscription covers the costs of using standard AWS WAF
> capabilities for resources that you protect with Shield Advanced.

So if Shield Advanced is already subscribed, the regional ALB web ACLs behind GA
are covered — which softens the "two web ACLs" cost, though not the operational
duplication.

## BYOIP — does bringing your own addresses change the calculus?

**For this estate: no. Note it, do not do it.** But it is the one GA feature that
can change the answer for a particular kind of customer, so it is worth
recording precisely.

From [Bring your own IP addresses (BYOIP) in Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/using-byoip.html),
verbatim:

> You can bring part or all of your public IPv4 address ranges from your
> on-premises network to your AWS account to use with AWS Global Accelerator. You
> continue to own the address ranges, but AWS advertises them on the internet.
> BYOIP with IPv6 is not supported at this time.

> You can't use the IP addresses that you bring to AWS for one AWS service with
> another service.

The requirements, verbatim from the
[requirements page](https://docs.aws.amazon.com/global-accelerator/latest/dg/using-byoip.requirements.html):

> You can bring up to two qualifying IP address ranges to AWS Global Accelerator
> per AWS account.

> The only address range that you can bring is /24.

> The IP address range must be registered with one of the following regional
> internet registries (RIRs): the American Registry for Internet Numbers (ARIN),
> Réseaux IP Européens Network Coordination Centre (RIPE), or Asia-Pacific
> Network Information Centre (APNIC). The address range must be registered to a
> business or institutional entity. It can't be registered to an individual.

> The IP addresses in the address range must have a clean history.

And the operational warning that is easy to miss:

> You must stop advertising your IP address range from other locations before you
> advertise it through AWS. If an IP address range is multihomed (that is, the
> range is advertised by multiple service providers at the same time), we can't
> guarantee that traffic to the address range will enter our network or that your
> BYOIP advertising workflow will complete successfully.

Plus one detail with a direct design consequence:

> When you create an accelerator, you can assign one IP address from your range
> to it. Global Accelerator assigns you a second static IP address from an Amazon
> IP address range. If you bring two IP address ranges to AWS, you can assign one
> IP address from each range to your accelerator. This restriction is because
> Global Accelerator assigns each address range to a different network zone, for
> high availability.

**So one BYOIP /24 buys you exactly one of your two addresses.** The second comes
from Amazon's pool unless you bring a *second* /24. If the entire point of BYOIP
was "our customers' firewalls already allow our /24", a half-BYOIP accelerator
does not deliver it — you need both ranges, and **two /24s is the per-account
maximum**, so **one accelerator can consume the whole quota**. With three
accelerators (one per pair) you **cannot** run all three fully on BYOIP.

### Why it does not apply here

1. There is no evidence in this vault that the company owns portable address
   space, and a `/24` from ARIN/RIPE is not something you acquire in a quarter.
2. The BYOIP *benefit* — "our published IPs never change, even if we leave AWS"
   — is a **vendor-independence** property, not a failover property. GA's
   Amazon-pool addresses are already permanent for the life of the accelerator,
   which is all the failover argument needs.
3. **It is separate from EC2 BYOIP**: *"You can't use the IP addresses that you
   bring to AWS for one AWS service with another service."* An existing EC2 BYOIP
   pool cannot be repurposed.
4. The two-range-per-account cap collides head-on with the three-accelerator
   design.

**Revisit only if** a regulator, a large B2B customer, or a partner integration
imposes a fixed-IP allow-list that must survive leaving AWS. Raised in
[[#Open questions]].

## Custom routing accelerators

**Briefly, because they are not applicable — but they are the other half of the
product and someone will ask.**

From [Working with custom routing accelerators](https://docs.aws.amazon.com/global-accelerator/latest/dg/work-with-custom-routing-accelerators.html),
verbatim:

> A custom routing accelerator lets you use application logic to directly map one
> or more users to a specific Amazon EC2 instance among many destinations, while
> gaining the performance improvements of routing your traffic through Global
> Accelerator. This is useful when you have an application that requires a group
> of users to interact with each other on the same session running on a specific
> EC2 instance and port, such as gaming applications or Voice over IP (VoIP)
> sessions.

> Endpoints for custom routing accelerators must be Amazon VPC (VPC) subnets, and
> a custom routing accelerator can only route traffic to Amazon EC2 instances in
> those subnets.

The mechanism is **deterministic port mapping**: you give the listener a large
port range and each (accelerator port) deterministically maps to one
(EC2 instance, destination port) pair. Your matchmaking service tells a client
"connect to `75.2.x.x:31427`" and that lands them on a specific instance.

| | Standard accelerator | Custom routing accelerator |
|---|---|---|
| Endpoints | ALB, NLB, EC2, Elastic IP | **VPC subnets only** (EC2 instances within them) |
| Routing decision | GA picks the closest healthy endpoint | **Your application picks**, via port mapping |
| Automatic health-based failover | **Yes** | **No — that is the point.** Traffic goes where the port says |
| Shield Advanced protectable | **Yes** (docs say "standard accelerators") | Not listed |
| Quota | 20 per account (adjustable) | **10 per account** |

> [!note] Not applicable to this estate, and the reason matters
> Custom routing exists to **defeat** automatic regional failover — it pins a
> session to a named instance. This estate wants the opposite. If anyone proposes
> it, they have the wrong product. The only realistic future trigger would be a
> stateful real-time feature (in-app voice, collaborative sessions) where users
> must share a process, and even then it would be a second accelerator alongside
> the standard one, not a replacement.

## Terraform implementation

### The single most useful fact, first

**You do not need a `us-west-2` provider alias.** Despite the CLI requiring
`--region us-west-2`, the `hashicorp/aws` provider resolves the Global
Accelerator endpoint to `us-west-2` **regardless of the provider's configured
region**. The provider's own generated endpoint tests assert exactly this —
[`internal/service/globalaccelerator/service_endpoints_gen_test.go`](https://github.com/hashicorp/terraform-provider-aws/blob/main/internal/service/globalaccelerator/service_endpoints_gen_test.go)
contains:

```go
expectedCallRegion = "us-west-2" //lintignore:AWSAT003
```

So `aws_globalaccelerator_accelerator` behaves like `aws_route53_zone` or
`aws_iam_role`: region-agnostic in HCL, globally scoped in reality. **Do not add
a fourth provider alias for it; do not create the accelerator from a `us-west-2`
root module "because that's where it lives".** That is a common and pointless
piece of ceremony.

`aws_globalaccelerator_endpoint_group` is the same — it is also a global API call
— but it carries `endpoint_group_region`, which is the *data*, not the provider.
This is the one place where "which region" is a string argument rather than a
provider alias, and it surprises people.

> [!warning] `endpoint_group_region` is `ForceNew`
> From the provider source, `endpoint_group_region` is declared
> `Optional: true, Computed: true, ForceNew: true`. Changing it **destroys and
> recreates the endpoint group**. Since it is also `Computed`, omitting it makes
> the provider infer the region from the endpoint ARN — which then *silently
> pins* to whatever the first apply saw. **Always set it explicitly.** Leaving it
> implicit is how a `plan` ends up proposing to recreate your live endpoint
> group because someone reordered a `for_each`.
>
> `listener_arn` is `ForceNew` too, so any listener change cascades.
> `ip_addresses` on the accelerator (the BYOIP argument) is **also `ForceNew`** —
> you cannot migrate an existing accelerator onto BYOIP addresses in place.
> `name` and `ip_address_type` are **not** `ForceNew` and update cleanly.

### Where this stack lives

GA is a **pair object**, not a regional object: one global accelerator, one
listener, and exactly two regional endpoint groups. That is the textbook case for
**Shape A — the pair wrapper module** from [[module-patterns]], and unusually it
is a case where Shape A is unambiguously correct rather than a judgement call:
the accelerator is a *single resource that spans the pair*, so there is nothing
to split into two root modules.

It also belongs in the **same stack as the Route 53 records**, for the reason
[[aws-route53]] gives — the alias record to the accelerator, the dormant fallback
records and the traffic dials are all "the failover switch", and the failover
switch should have exactly one blast radius and one set of IAM permissions. See
[[provider-aliases-vs-separate-stacks]].

### The module contract

```hcl
# modules/pair-accelerator/versions.tf

terraform {
  required_version = ">= 1.11"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 6.0"

      # The accelerator, listener and endpoint groups are all GLOBAL API calls
      # that the provider routes to us-west-2 on its own. We still take both
      # aliases, because the module reads regional data (ALB ARNs are supplied
      # as inputs, but the WAF association and the Route 53 alias want a
      # provider) and because the caller's mental model should stay uniform.
      configuration_aliases = [aws.primary, aws.standby]
    }
  }
}
```

### The variable surface

```hcl
# modules/pair-accelerator/variables.tf

variable "pair_name" {
  description = "Logical name of the region pair, e.g. prod-eu. One accelerator per pair — see the fail-open argument, never one accelerator for all six regions."
  type        = string
}

variable "primary_region" {
  description = "Must be set explicitly. endpoint_group_region is ForceNew."
  type        = string
}

variable "standby_region" {
  type = string
}

variable "primary_endpoint_arn" {
  description = "ARN of the primary region's ALB or NLB. Supplied by the regional root module via SSM/remote state — this module never does a cross-region data lookup."
  type        = string
}

variable "standby_endpoint_arn" {
  description = "ARN of the standby region's ALB or NLB. null until the standby exists."
  type        = string
  default     = null
}

# ---- the failover switch itself -------------------------------------------
# These four numbers ARE the runbook. They are deliberately exposed as plain
# variables so a human can see the current posture in a tfvars file, but see
# the lifecycle ignore_changes below: Terraform must NOT be the thing that
# moves them at 3am.

variable "primary_traffic_dial" {
  description = "0-100. Normal running: 100."
  type        = number
  default     = 100
}

variable "standby_traffic_dial" {
  description = "0-100. Normal running: 0."
  type        = number
  default     = 0
}

variable "primary_weight" {
  description = "0-255. Normal running: 128."
  type        = number
  default     = 128
}

variable "standby_weight" {
  description = <<-EOT
    0-255. Normal running: 0.

    THIS IS THE REAL OFF SWITCH, NOT THE TRAFFIC DIAL. GA ignores the traffic
    dial when failing over away from an unhealthy group, so dial 0 does NOT
    prevent an unattended automatic failover into a 2-hour-stale replica.
    Weight 0 does. Leave this at 0 for an async-replica standby.
  EOT
  type        = number
  default     = 0
}

variable "standby_enabled" {
  description = "Build the standby endpoint group. False leaves the primary untouched — this is what makes the migration a two-step, zero-downtime apply."
  type        = bool
  default     = false
}

variable "listener_ports" {
  description = "Client-facing ports. TCP 443 only for an HTTPS API; add 80 only if you actually redirect."
  type        = list(number)
  default     = [443]
}

variable "protocol" {
  description = "TCP or UDP. TCP unless you have a genuine UDP workload."
  type        = string
  default     = "TCP"
}

variable "client_affinity" {
  description = "NONE (default) or SOURCE_IP. See the client-affinity section — NONE is right here."
  type        = string
  default     = "NONE"
}

variable "flow_logs_bucket" {
  description = "S3 bucket for accelerator flow logs. null disables them. Worth having: flow logs are the only visibility into which endpoint group actually served a connection."
  type        = string
  default     = null
}
```

### The module body

```hcl
# modules/pair-accelerator/main.tf

locals {
  # One accelerator per PAIR. Never one for the whole estate — see the
  # fail-open argument: an accelerator holding all six regions will route EU
  # traffic into us-east-1 under sufficient failure, which is a data-residency
  # breach as well as a correctness bug. $36.50/month buys the blast radius.
  name = "${var.pair_name}-ingress"
}

# GLOBAL resource. No provider alias needed — the provider routes
# globalaccelerator API calls to us-west-2 on its own, whatever region the
# default provider is set to.
resource "aws_globalaccelerator_accelerator" "this" {
  name            = local.name
  ip_address_type = "IPV4"
  enabled         = true

  # NOT SET: ip_addresses. That argument is BYOIP-only and it is ForceNew.
  # Leave it unset and let AWS allocate. The two addresses it allocates are
  # permanent for the life of the accelerator, which is the whole point.

  dynamic "attributes" {
    for_each = var.flow_logs_bucket == null ? [] : [1]
    content {
      flow_logs_enabled   = true
      flow_logs_s3_bucket = var.flow_logs_bucket
      flow_logs_s3_prefix = "globalaccelerator/${local.name}/"
    }
  }

  lifecycle {
    # Deleting the accelerator releases the static IPs FOREVER. Every client
    # firewall allow-list, every cached JVM address, every hard-coded IoT
    # config breaks and cannot be recovered. This is the single most
    # destructive plan in the estate.
    prevent_destroy = true
  }
}

resource "aws_globalaccelerator_listener" "this" {
  accelerator_arn = aws_globalaccelerator_accelerator.this.arn
  client_affinity = var.client_affinity
  protocol        = var.protocol

  dynamic "port_range" {
    for_each = var.listener_ports
    content {
      from_port = port_range.value
      to_port   = port_range.value
    }
  }
}

# --- primary half ------------------------------------------------------------

resource "aws_globalaccelerator_endpoint_group" "primary" {
  listener_arn = aws_globalaccelerator_listener.this.arn

  # EXPLICIT. ForceNew. Never let this be inferred.
  endpoint_group_region = var.primary_region

  traffic_dial_percentage = var.primary_traffic_dial

  # These settings are IGNORED for ALB/NLB endpoints — GA uses the load
  # balancer's own target-group health checks. They are declared anyway so the
  # module also works for EC2/EIP endpoints, and so nobody later "fixes" a
  # failover by tuning numbers that were never in the path.
  health_check_interval_seconds = 10
  threshold_count               = 3

  endpoint_configuration {
    endpoint_id = var.primary_endpoint_arn
    weight      = var.primary_weight

    # Defaults to FALSE in the provider. Without it the ALB access logs and
    # every IP-based WAF rule see Global Accelerator's address, not the
    # client's. Also REQUIRED for dual-stack accelerators.
    #
    # Turning it on makes GA create a security group named "GlobalAccelerator"
    # in the VPC that blocks `terraform destroy` of that VPC. Ephemeral
    # environments must handle this. See the Gotchas section.
    client_ip_preservation_enabled = true
  }

  lifecycle {
    # THE IMPORTANT ONE.
    #
    # Failover is an UpdateEndpointGroup API call made by the runbook, not a
    # terraform apply. Without this, the next routine `apply` after a failover
    # silently reverts the dial to 100 and fails you BACK into the dead
    # region. This is the GA equivalent of the "don't run Terraform during an
    # incident" rule in [[failover-orchestration]], expressed as code.
    #
    # The cost: `terraform plan` no longer tells you the live posture.
    # Observability of the current dial belongs on a dashboard
    # ([[observability-multi-region]]), not in state.
    ignore_changes = [
      traffic_dial_percentage,
      endpoint_configuration[0].weight,
    ]
  }
}

# --- standby half ------------------------------------------------------------

resource "aws_globalaccelerator_endpoint_group" "standby" {
  count = var.standby_enabled ? 1 : 0

  listener_arn          = aws_globalaccelerator_listener.this.arn
  endpoint_group_region = var.standby_region

  # 0 in normal running. Note this does NOT stop automatic failover into it —
  # standby_weight = 0 is what does that.
  traffic_dial_percentage = var.standby_traffic_dial

  health_check_interval_seconds = 10
  threshold_count               = 3

  endpoint_configuration {
    endpoint_id                    = var.standby_endpoint_arn
    weight                         = var.standby_weight
    client_ip_preservation_enabled = true
  }

  lifecycle {
    ignore_changes = [
      traffic_dial_percentage,
      endpoint_configuration[0].weight,
    ]
  }
}
```

```hcl
# modules/pair-accelerator/outputs.tf

output "accelerator_arn" {
  value = aws_globalaccelerator_accelerator.this.arn
}

output "static_ips" {
  description = "The two anycast addresses. Publish these to customers who keep firewall allow-lists — they never change for the life of the accelerator."
  value       = aws_globalaccelerator_accelerator.this.ip_sets
}

output "dns_name" {
  value = aws_globalaccelerator_accelerator.this.dns_name
}

output "hosted_zone_id" {
  description = "Always Z2BJ6XQ5FK7U4H. Exported so the Route 53 alias doesn't hard-code a magic string."
  value       = aws_globalaccelerator_accelerator.this.hosted_zone_id
}

output "endpoint_group_arns" {
  description = <<-EOT
    HARD-CODE THESE INTO THE RUNBOOK as literal strings, exactly as
    [[route53-application-recovery-controller]] says to hard-code ARC's five
    cluster endpoints. Looking them up at 3am with `describe-accelerator` is a
    control-plane call, and you may be failing over BECAUSE the control plane
    is unreachable.
  EOT
  value = {
    primary = aws_globalaccelerator_endpoint_group.primary.arn
    standby = one(aws_globalaccelerator_endpoint_group.standby[*].arn)
  }
}
```

### Calling it, and the Route 53 record that goes with it

```hcl
# live/prod-eu/pair/main.tf

module "ingress_accelerator" {
  source = "../../../modules/pair-accelerator"

  providers = {
    aws.primary = aws.eu_west_1
    aws.standby = aws.eu_west_2
  }

  pair_name       = "prod-eu"
  primary_region  = "eu-west-1"
  standby_region  = "eu-west-2"
  standby_enabled = var.standby_enabled

  primary_endpoint_arn = data.aws_ssm_parameter.primary_alb_arn.value
  standby_endpoint_arn = try(data.aws_ssm_parameter.standby_alb_arn[0].value, null)

  flow_logs_bucket = module.logging.bucket_name
}

# The name never changes again after this. That is the entire point: the TTL
# stops mattering because the answer stops changing.
resource "aws_route53_record" "api" {
  zone_id = data.aws_route53_zone.public.zone_id
  name    = "api.${var.public_domain}"
  type    = "A"

  alias {
    name    = module.ingress_accelerator.dns_name
    zone_id = module.ingress_accelerator.hosted_zone_id

    # FALSE, deliberately. There is nothing useful for Route 53 to evaluate:
    # the accelerator is anycast and always "up". Health evaluation happens
    # inside GA, on the ALB's target health. Setting this true buys nothing and
    # adds a failure mode.
    evaluate_target_health = false
  }
}

# The second control plane, kept dormant and free. GA's control plane is
# us-west-2; Route 53's is us-east-1. Keeping a pre-created, weight-0,
# NO-HEALTH-CHECK record pointed straight at the standby ALB means a us-west-2
# impairment does not remove your ability to move traffic. Zero cost.
# See [[#The control plane — and why us-west-2 is the interesting answer]].
resource "aws_route53_record" "api_bypass_standby" {
  count = var.standby_enabled ? 1 : 0

  zone_id        = data.aws_route53_zone.public.zone_id
  name           = "api-direct-standby.${var.public_domain}"
  type           = "A"
  set_identifier = "bypass-${var.standby_region}"

  # No health_check_id: weight 0 then genuinely means 0. See [[aws-route53]].
  weighted_routing_policy { weight = 0 }

  alias {
    name                   = var.standby_alb_dns_name
    zone_id                = var.standby_alb_zone_id
    evaluate_target_health = false
  }
}
```

### Cross-account endpoints, if the standby lives elsewhere

If the standby region's ALB is in a different AWS account — a question
[[aws-acm]] and [[aws-route53]] both raise — the endpoint cannot simply be
referenced by ARN. The provider models AWS's cross-account attachment flow:

```hcl
# In the ACCOUNT THAT OWNS THE ALB.
resource "aws_globalaccelerator_cross_account_attachment" "standby" {
  provider = aws.standby_account

  name       = "prod-eu-standby-alb"
  principals = [var.accelerator_account_id]

  resource {
    endpoint_id = var.standby_alb_arn
    region      = var.standby_region
  }
}

# In the ACCELERATOR'S account, the endpoint_configuration then carries
# attachment_arn alongside endpoint_id.
```

Budget for this as a two-account, two-apply ordering problem, the same shape as
the Route 53 cross-account zone association.

### What is deliberately *not* in Terraform

| Thing | Why not |
|---|---|
| The failover itself | `ignore_changes` above. It is an `UpdateEndpointGroup` call from the runbook. |
| Accelerator deletion | `prevent_destroy`. Losing the IPs is unrecoverable. |
| WAF web ACL | Lives on the **ALB**, in the regional stack, because GA cannot carry one ([[#Does AWS WAF work in front of Global Accelerator?]]). Two of them, one per region. |
| BYOIP provisioning | A multi-week RIR/ROA process with no Terraform resource for `ProvisionByoipCidr` in this service. Out-of-band. |

## Migration path from single-region

The good news: **this is one of the least disruptive migrations in the vault,
because the accelerator is additive.** Nothing existing is modified in place and
nothing is destroyed. The bad news is a cutover window where two ingress paths
are live at once, which AWS explicitly warns about.

### Step 0 — answer two questions first

1. **How much of the public surface is API Gateway?** GA cannot front it
   ([[#API Gateway — verified negative, and what to do instead]]). If the answer
   is "most of it", stop here and use Route 53 or CloudFront instead.
2. **Are any Montreal resources in AZ ID `cac1-az3`?** Only matters for EC2/EIP
   endpoints, but check before signing off the CA pair
   ([[#`ca-west-1` and `ca-central-1` — the parity check]]).

### Step 1 — create the accelerator alongside the live path (no traffic)

`standby_enabled = false`. One accelerator, one listener, one endpoint group
pointed at the **existing primary ALB**. Traffic dial 100, weight 128.

- **Zero impact.** The live `api.example.com` still resolves to the ALB
  directly. Nobody is using the anycast IPs yet.
- Cost starts here: $18.25/month per accelerator, plus DT-Premium on whatever
  test traffic you send.
- **`client_ip_preservation_enabled = true` creates the `GlobalAccelerator`
  security group in the VPC at this point.** Expect it; do not delete it.

### Step 2 — verify on a parallel hostname

Point `api-ga.example.com` at the accelerator and run the full test suite,
including TLS, WebSockets if any, and — critically — **check that the ALB access
logs show real client IPs, not GA's.** If they show GA's address, client IP
preservation is off and every WAF rule is about to start behaving differently.

### Step 3 — the cutover, and the one AWS warning that matters

Repoint `api.example.com` from the ALB alias to the accelerator alias. This is a
**Route 53 change**, so it inherits all of [[aws-route53]]'s TTL and
client-cache behaviour — **once, at migration time, and never again.** That is
the trade: you pay the DNS-cache tax a single time to buy permanent immunity
from it.

> [!warning] During the cutover both paths are live, and AWS says not to do that
> Verbatim from the endpoint requirements page:
>
> > When you add resources as endpoints behind Global Accelerator, we recommend
> > that you don't also send traffic directly to the same endpoints over the
> > internet. Sending direct traffic can lead to connection collision issues.
>
> A DNS migration **guarantees** a period where some clients reach the ALB
> directly and some arrive via GA — potentially hours, given resolver and JVM
> caching. There is no way to avoid this with a DNS cutover.
>
> **Mitigations:** lower the TTL to 10 s a week in advance; do the cutover in a
> low-traffic window; and if the risk is judged unacceptable, put the ALB behind
> a **new** internal-facing listener for GA and keep the internet-facing one for
> the legacy path, so the two paths do not share a listener. For **ALB**
> endpoints the collision risk is largely theoretical; AWS's stronger advice
> (*"disable cross-zone traffic"*) is aimed at NLBs.

### Step 4 — make the ALB internal (optional, and worth it)

Once no traffic arrives directly, the ALB can become internal, removing the last
internet-facing regional load balancer. **GA supports internal ALBs** —
*"An Application Load Balancer endpoint can be internet-facing or internal."*

**This is a `ForceNew` change on `aws_lb` (`internal` cannot be modified in
place).** It replaces the load balancer, which changes its ARN, which forces a
`UpdateEndpointGroup`. Sequence it as: create new internal ALB → add as a second
endpoint in the same endpoint group at weight 128 → drain the old one to weight 0
→ remove. Do **not** let a single `terraform apply` do all of that. See
[[aws-alb-nlb]] and [[security-posture-of-the-standby]].

### Step 5 — add the standby endpoint group

`standby_enabled = true`, `standby_traffic_dial = 0`, `standby_weight = 0`. This
is a **one-resource diff** and it is the entire multi-region change.

**Before flipping the weight above 0, confirm every target group in the standby
has at least one healthy target** — GA marks an ALB unhealthy if *any* target
group is empty ([[#Detection time, decomposed]]).

### Nothing here forces replacement of anything that already exists

| Change | Effect on `terraform plan` |
|---|---|
| Add accelerator + listener + endpoint group | **Create only.** No existing resource touched. |
| Add the standby endpoint group | **Create only.** |
| Repoint the Route 53 alias | **In-place update** of one record. |
| Change `endpoint_group_region` | **ForceNew** — never do this; create a new group instead |
| Change `ip_addresses` (BYOIP) | **ForceNew on the accelerator — you lose the IPs.** Never. |
| Change accelerator `name` | In-place. Safe. |
| Make the ALB internal | **ForceNew on `aws_lb`** — sequence it, see Step 4 |

## Failover procedure

The GA-specific steps only. The full ordering — pre-scale, promote the database,
fence the old primary — lives in [[failover-orchestration]] and
[[failover-runbook-template]]; **GA is step 6 of about eleven, and it is the
fastest one.**

### Pre-flight (the day before, or continuously in CI)

- [ ] Both endpoint group ARNs are **literal strings in the runbook**, not looked
      up.
- [ ] Break-glass credentials with `globalaccelerator:UpdateEndpointGroup` exist
      and are reachable without the IdP ([[aws-iam]]).
- [ ] Every target group in the standby has ≥ 1 healthy target. **If any target
      group is empty, GA considers the whole standby ALB unhealthy and will
      refuse to send it traffic.**
- [ ] `client_keep_alive = 300` on both ALBs ([[aws-alb-nlb]]). GA does not
      replace this.
- [ ] The dormant Route 53 bypass record exists (the second control plane).

### The shift

```bash
set -euo pipefail

# ORDER MATTERS. Bring the standby up FIRST. If you take the primary down
# first you briefly have no weighted healthy endpoint anywhere and GA fails
# OPEN — it will route to "a random endpoint in the endpoint group that is
# closest to the client", ignoring your dials.

# 1. Arm the standby: weight 128 (from 0 — the real off switch), dial 100.
aws globalaccelerator update-endpoint-group \
  --region us-west-2 \
  --endpoint-group-arn "$STANDBY_EG_ARN" \
  --traffic-dial-percentage 100 \
  --endpoint-configurations \
      EndpointId="$STANDBY_ALB_ARN",Weight=128,ClientIPPreservationEnabled=true

# 2. Verify GA actually considers the standby healthy BEFORE cutting the
#    primary. HealthState must be HEALTHY, not INITIAL and not UNHEALTHY.
aws globalaccelerator describe-endpoint-group \
  --region us-west-2 \
  --endpoint-group-arn "$STANDBY_EG_ARN" \
  --query 'EndpointGroup.EndpointDescriptions[].[EndpointId,HealthState,HealthReason]' \
  --output table

# 3. Cut the primary: dial 0 AND weight 0. Weight 0 is what stops GA
#    automatically failing back the moment the primary looks healthy again.
aws globalaccelerator update-endpoint-group \
  --region us-west-2 \
  --endpoint-group-arn "$PRIMARY_EG_ARN" \
  --traffic-dial-percentage 0 \
  --endpoint-configurations \
      EndpointId="$PRIMARY_ALB_ARN",Weight=0,ClientIPPreservationEnabled=true
```

**Elapsed: seconds.** New connections land in the standby immediately. Then the
part nobody likes:

### What is still true five minutes later

- **Established, actively-transferring TCP connections are still in the dead
  region.** Not moved, not moveable, no timeout applies while data flows.
- **Idle connections persist up to 340 s.** Not tunable.
- **Removing the endpoint does not hang them up either.**

**So the traffic shift is not the end of the failover — fencing is.** The only
way to force the remaining clients over is to break the old path at the region:
scale the deployment to zero, deregister the targets, or revoke the ALB's
security-group ingress. That is [[split-brain-and-fencing]]'s job and it is
mandatory, not optional, because for an async replica *the old primary still
accepting writes is the actual data-loss event.*

### The human decision gate

Nothing above should be automatic. **Run the standby at weight 0 permanently**
so GA physically cannot move traffic on its own, and accept that you have traded
automatic failover — which you did not want at RPO 2h — for a deliberate one.

### If the GA control plane is unreachable

`us-west-2` impaired means no `UpdateEndpointGroup`. Two fallbacks, in order:

1. **Let GA's data plane do it.** Temporarily set the standby weight above 0 —
   oh wait, that is also a control-plane call. **It is not a real fallback.** The
   honest statement is: *with weight 0 on the standby and no control plane, you
   cannot fail over via GA at all.*
2. **Use the other control plane.** Repoint `api.example.com` at the standby ALB
   via Route 53 (`us-east-1` control plane), accept the TTL and the client-cache
   tail, and carry on. **This is why the dormant bypass record exists and why it
   costs nothing to keep.**

That asymmetry is worth stating plainly: **the weight-0 safety measure that
protects you from an unwanted automatic failover is the same thing that removes
your data-plane fallback.** You cannot have both. The Route 53 bypass record is
the resolution.

## Failback

Failback is where GA is genuinely, unusually good — **and where its automatic
behaviour is genuinely, unusually dangerous.** Both are true and they are the
same feature.

### The danger first

Verbatim, from the failover page:

> When recovery occurs, that is, Regions are healthy again, Global Accelerator
> returns to regular routing behavior. This means that, typically, routing will
> start back to healthy endpoints with traffic dials that aren't set to zero in
> about 30 seconds or so.

Read that as an operator, not a marketer. **If you failed over by setting the
primary's dial to 0 but left its weight above 0, then the moment the old primary's
ALB reports healthy targets again — which happens automatically when the region
recovers and pods restart — GA will start sending traffic back to it.** In about
30 seconds. Into a database that is now the *wrong* one, because you promoted the
standby.

**That is split-brain, delivered by a health check, roughly half a minute after
the region comes back, with nobody in the room.** It is the same failure mode
[[aws-route53]] describes for automatic DNS failover, but faster and without a
TTL to slow it down.

> [!danger] The fence must go on before the region recovers, not after
> Setting the old primary's **weight to 0** during the failover (step 3 above)
> is what prevents this, and it is why that step sets *both* the dial and the
> weight. A dial of 0 alone is **not** sufficient — GA ignores traffic dials
> when selecting a failover target. **If you only turn the dial down, you have
> armed an automatic failback.**

### The ordered failback

Failback is a *planned* operation and should be harder to start than failover,
not easier.

1. **Confirm the old primary is genuinely healthy** — not just "the ALB has
   targets", but the full readiness definition: database reachable, migrations
   at the right version, caches warm. GA's health check will not tell you this;
   it only knows "at least one healthy target per target group".
2. **Reverse the replication direction.** The old primary's database is now
   stale and must be rebuilt as a replica *of the promoted standby*. This is the
   long pole and it is entirely [[aws-rds-postgres]] / [[aws-aurora-global-database]]'s
   problem. **Until this is done, failback is not possible at any speed.**
3. **Canary first.** This is GA's best trick and it has no good equivalent in
   Route 53: set the old primary's weight to 128 and its **dial to 10**. Ten
   percent of connections go back, edge-side, immediately, with no TTL to wait
   out and no gradual rollout of a mistake.
4. **Watch.** Error rate, latency, replication lag in the new direction.
5. **Ratchet: 10 → 25 → 50 → 100**, each step an `UpdateEndpointGroup` call
   taking effect in seconds.
6. **Roll back instantly if needed** — set the dial back to 0. There is no TTL
   to wait out, which is the property Route 53 cannot match.
7. **Only once at 100: set the other region's weight to 0** and re-arm it as
   the standby.

**Connections established during the canary stay where they landed** (340 s idle
timeout, indefinite if active), so a 10% canary is 10% of *new connections*, not
10% of users. For long-lived connections the ramp is slower than the dial
suggests. Budget for it.

### Why GA's failback is better than the alternatives

| | Route 53 | CloudFront KVS | **Global Accelerator** |
|---|---|---|---|
| Canary granularity | Weighted records, coarse, TTL-limited | A per-key flag: all or nothing unless the function hashes | **Dial 0–100, edge-side, immediate** |
| Time to roll back a bad failback | TTL + client caches | Seconds | **Seconds, no cache tail** |
| Risk of automatic, unwanted failback | Yes, if health checks attached | No | **Yes — and fast. Weight 0 is the fence.** |

## Gotchas

The list that makes the note worth reading. Roughly in order of how badly each
one bites.

1. **Deleting the accelerator destroys the static IPs permanently.** They are
   not recoverable and not transferable. Every customer firewall allow-list,
   every hard-coded IoT address, every cached JVM entry breaks at once — and the
   thing that made GA valuable (permanent addresses) is exactly what makes this
   unrecoverable. `prevent_destroy = true`, always.
2. **Traffic dial 0 does not stop an automatic failover into the standby.** GA
   *"ignores the traffic dial setting"* when failing over away from an unhealthy
   group. Only **weight 0** removes an endpoint from consideration. For a
   2-hour-stale async replica this is the difference between a controlled
   failover and an unattended data-loss event.
3. **Automatic failback in "about 30 seconds or so".** See [[#Failback]]. The
   same rule: weight 0, not dial 0.
4. **GA marks an ALB unhealthy if *any* target group is empty.** *"Global
   Accelerator considers an Application Load Balancer healthy if every target
   group has at least one healthy target."* A forgotten canary target group, a
   rarely-used service scaled to zero, a blue/green slot with nothing in it —
   any of these makes the entire region unhealthy to GA. This is **stricter than
   Route 53's `evaluate_target_health`**, which reports an ALB with no targets as
   *healthy*, and it directly constrains how far the warm standby can scale down
   ([[aws-eks]]).
5. **The 340-second TCP idle timeout is not tunable, and "idle" is strict.** AWS:
   *"you cannot use TCP keep-alive packets to maintain an open connection"* —
   a connection carrying real data never idles out and stays pinned to the old
   region **indefinitely**. Removing the endpoint does not help.
6. **`client_ip_preservation_enabled` defaults to `false` in Terraform.** Silent,
   and it breaks every IP-based WAF rule, rate limit, geo rule and access log at
   once. Nothing errors; the data is just wrong.
7. **Turning it on creates a `GlobalAccelerator` security group that blocks
   `terraform destroy` of the VPC**, with a `DependencyViolation` that *"cannot
   be resolved by re-running Terraform"*. Every ephemeral/sandbox environment
   needs a teardown hook that deletes it. Pair this with the accelerated-recovery
   zone-deletion trap from [[aws-route53]] in the cookiecutter template.
8. **AWS WAF cannot be attached to an accelerator.** Verified. Web ACLs go on the
   ALB, one per region, and **rate-limit counters reset on failover**.
9. **Fail-open crosses regions.** Without a healthy weighted endpoint in the
   three closest endpoint groups, GA routes to *"a random endpoint in the
   endpoint group that is closest to the client"*. One accelerator for all six
   regions is a data-residency breach waiting for a bad day. **Three
   accelerators, two endpoint groups each.**
10. **`endpoint_group_region` is `ForceNew` and `Computed`.** Omitting it lets
    the provider infer it, and a later refactor can produce a surprise
    destroy/recreate of a live endpoint group.
11. **`ip_addresses` (BYOIP) is `ForceNew`.** You cannot move an existing
    accelerator onto your own addresses without recreating it — i.e. without
    losing the current addresses. Decide BYOIP on day one or never.
12. **The control plane is `us-west-2`, which is the US pair's standby.** A
    `us-west-2` impairment blocks deliberate traffic shifts **for all three
    pairs**, including EU and CA, which have nothing to do with Oregon. Keep the
    dormant Route 53 record as a second control plane.
13. **`ca-central-1` is supported "except AZ `cac1-az3`"** — the *primary*
    carries the exception, not Calgary. AZ **ID**, not letter; check with
    `describe-availability-zones`.
14. **API Gateway cannot be a GA endpoint.** If the public surface is mostly API
    Gateway, GA is not available to you as a failover mechanism.
15. **Dual-stack EIPs cannot be added**, and **dual-stack accelerators require
    client IP preservation** on every endpoint.
16. **Two API calls, not one.** No atomic multi-endpoint-group transaction, so
    there is a window between arming the standby and cutting the primary. ARC's
    `UpdateRoutingControlStates` does this atomically; GA does not. Order the
    calls as in [[#The shift]].
17. **A disabled accelerator still bills.** `enabled = false` saves nothing;
    only deletion stops the $0.025/hour, and deletion is the thing you must never
    do.
18. **Older EC2 instance families cannot be endpoints** (C1, CC1, CC2, CG1, CG2,
    CR1, CS1, G1, G2, HI1, HS1, M1, M2, M3, T1). Only relevant if you use EC2
    endpoints directly.
19. **42 endpoint groups per accelerator is a hard, non-adjustable quota**, and
    10 endpoints per endpoint group. Not a constraint at three regions, but it
    caps any future "one accelerator, many regions" idea — which you should not
    want anyway.
20. **Flow logs are the only per-connection visibility you get.** There is no
    equivalent of a CloudFront real-time log or an ALB access log at the
    accelerator layer. Enable them, or you will be unable to answer "which region
    actually served that customer during the incident" ([[observability-multi-region]]).

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
- [Well-Architecting online applications with CloudFront and AWS Global Accelerator (AWS Networking & Content Delivery blog)](https://aws.amazon.com/blogs/networking-and-content-delivery/well-architecting-online-applications-with-cloudfront-and-aws-global-accelerator/) — AWS's own positioning of the two services: *"Customers use Amazon CloudFront for most HTTP(S) based Web applications"*, GA *"operates at layer 4 of the OSI model, it can be used with any TCP/UDP application"*, and *"AWS WAF rules are enforced in CloudFront PoPs"*. The basis for [[#Global Accelerator vs CloudFront]].
- [AWS WAF — protected resource types (developer guide)](https://docs.aws.amazon.com/waf/latest/developerguide/waf-chapter.html) — the definitive enumeration of what a web ACL can be associated with (CloudFront, API Gateway REST API, ALB, AppSync, Cognito user pool, App Runner, Bedrock AgentCore Gateway, Verified Access, Amplify) and therefore the **verified absence of Global Accelerator**.
- [Use AWS WAF with Global Accelerator to block Layer 7 HTTP method and headers (AWS re:Post knowledge center)](https://repost.aws/knowledge-center/globalaccelerator-aws-waf-filter-layer7-traffic) — AWS's documented workaround: attach the web ACL to the ALB endpoint behind the accelerator, not to the accelerator. (Returned HTTP 403 to automated fetch; content confirmed via AWS search indexing of the same article — treat the architectural claim, which is corroborated by the WAF resource-type list above, as verified and the exact wording as not independently quoted here.)
- [List of AWS resources that AWS Shield Advanced protects](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary-protected-resources.html) — verbatim list including **"AWS Global Accelerator standard accelerators"**; note the word *standard*, which is why custom routing accelerators are not claimed as protectable in this note.
- [AWS Shield Advanced overview](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary.html) — *"Your Shield Advanced subscription covers the costs of using standard AWS WAF capabilities for resources that you protect with Shield Advanced"*, which is what softens the cost of running two regional web ACLs behind GA.
- [Bring your own IP addresses (BYOIP) in Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/using-byoip.html) — BYOIP is IPv4-only; *"You can't use the IP addresses that you bring to AWS for one AWS service with another service"*; the multihoming warning; and the **one-address-per-range / second-address-from-Amazon's-pool network-zone rule**.
- [BYOIP requirements for Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/using-byoip.requirements.html) — **two ranges per account maximum**, **`/24` only**, ARIN/RIPE/APNIC registration to a business entity, clean-reputation requirement, and the per-RIR allocation statuses.
- [Working with custom routing accelerators in AWS Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/work-with-custom-routing-accelerators.html) — verbatim definition, the gaming/VoIP use cases, **VPC-subnet-only endpoints**, and the deterministic port-mapping model that makes them the opposite of a failover mechanism.
- [Support for DNS addressing in AWS Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/dns-addressing-custom-domains.dns-addressing.html) — the accelerator's default `*.awsglobalaccelerator.com` DNS name, used to reason about (and reject) the "GA as a CloudFront custom origin" configuration.

## Related notes

[[aws-route53]] · [[aws-alb-nlb]] · [[aws-cloudfront]] · [[aws-api-gateway]] · [[aws-acm]] · [[aws-waf-shield]] · [[aws-eks]] · [[route53-application-recovery-controller]] · [[failover-orchestration]] · [[split-brain-and-fencing]] · [[observability-multi-region]] · [[region-pair-selection]] · [[provider-aliases-vs-separate-stacks]] · [[module-patterns]] · [[cost-model]] · [[aws-regional-outages]] · [[lessons-and-antipatterns]] · [[data-residency]]
