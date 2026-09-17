---
title: Amazon Route 53 — Multi-Region
service: route53
tags: [service, multi-region, route53, dns, failover, edge, arc]
status: researched
replication: global — nothing to replicate, the hosted zone is already everywhere
rpo_achievable: N/A — configuration, not data
rto_achievable: "~2 min best case, 3–6 min realistic for the DNS layer alone; client-side caching can extend it to hours if unmanaged"
meets_targets: yes — with pre-created records, TTL 60, and client DNS caching under control
updated: 2026-09-16
---

# Amazon Route 53 — Multi-Region

## TL;DR

- **Hosted zones are global, not regional.** There is no "Route 53 in `eu-west-1`". One zone, one set of records, answered from a worldwide anycast fleet. **There is nothing to mirror.** This is the single biggest simplification in the whole programme and it should be said out loud before anyone builds a second hosted zone "for the standby".
- **This is where the 15-minute RTO is won or lost.** Every other note in this vault is about making the standby *ready*. This note is about the one action that makes it *serve*: pointing the name somewhere else. The mechanism is cheap; the latency is mostly other people's caches.
- **The DNS layer itself costs ~2 minutes and can be made to cost ~2 minutes.** Fast health check (10 s) × failure threshold 3 ≈ 30 s detection, plus a few seconds of internal propagation, plus a 60 s record TTL. Call it **90–120 s**. See [[#The arithmetic against the 15-minute budget]]. That leaves ~13 minutes for everything else, which is the number the rest of the vault has to fit inside.
- **The control-plane trap: Route 53's control plane lives in `us-east-1`, and `us-east-1` is the US pair's primary.** If your failover plan is "call `ChangeResourceRecordSets`", the US pair's plan has a dependency on the region it is failing away from. This is not theoretical — it is the exact failure mode that ARC routing controls and Route 53 accelerated recovery were built for. See [[#The control-plane trap]].
- **The thing that will bite:** not DNS. **Clients.** A JVM with `networkaddress.cache.ttl` set to a negative value caches a resolved IP **for the life of the process**, and an ALB's default HTTP client keepalive is **3600 seconds**. You can execute a textbook 90-second DNS failover and still have a meaningful share of traffic hammering the dead region an hour later. See [[#The client-side caching problem]].

## Does this service cross regions at all?

It does not need to. Route 53 is a **global service**, in AWS's own fault-isolation taxonomy — the same category as IAM, CloudFront and Global Accelerator.

| Thing | Scope |
|---|---|
| Public hosted zone | **Global.** Not created in a region, has no region attribute, is served from a global anycast name server fleet. |
| Records in that zone | **Global.** |
| Health checks | **Global**, though a health check *targets* a regional endpoint and is *performed* from checkers in eight AWS regions. |
| Private hosted zone | Global object, but **associated with specific VPCs**, which are regional. See [[#Private hosted zones across regions]]. |
| **Route 53 control plane** (the API you call to change anything) | **`us-east-1` only.** This is the problem. |
| **Route 53 data plane** (answering DNS queries, running health checks) | **Global**, and carries a **100% availability SLA**. |

Two practical consequences for the Terraform estate:

1. **`aws_route53_zone` and `aws_route53_record` do not need a provider alias.** They are region-agnostic; whichever provider you use, the SDK talks to the global `route53.amazonaws.com` endpoint, which is physically served from `us-east-1`. Do not create a `dns` stack "per region". Create **one** DNS stack per account that owns the public zone, and have both regional stacks feed it outputs.
2. **`aws_route53_health_check` also has no region**, but the *thing it checks* does. A health check against the standby ALB in `eu-west-2` is declared in the same global zone stack as one against `eu-west-1`. Don't try to co-locate health checks with the resources they check — you'll create a circular dependency between two regional stacks for no benefit.

> [!important] Say this to the team once, clearly
> "Making Route 53 multi-region" is not a workstream. Route 53 is already multi-region. The workstream is *record design* and *failover triggering*. If a ticket exists called "replicate hosted zone to eu-west-2", close it.

## The failover toolkit

Route 53 gives you five routing policies plus health checks. Only three are relevant to an active/passive pair, and the vault should be blunt about which.

### Failover routing (primary/secondary)

The purpose-built policy. Two records, same name, same type, `set_identifier` distinct, one `failover_routing_policy { type = "PRIMARY" }` and one `"SECONDARY"`. From the [AWS docs](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-types.html):

> When responding to queries, Route 53 includes only the healthy primary resources. If all the primary resources are unhealthy, Route 53 begins to include only the healthy secondary resources in response to DNS queries.

**Properties:**

- Semantics match the architecture exactly: one live, one standby, automatic.
- The secondary **can also carry a health check**, which means Route 53 will not fail over into a standby it believes is broken. That is usually what you want — but note [[aws-acm]] gotcha #7: HTTPS health checks do **not** validate certificates, so "green" does not mean "will serve TLS".
- The primary record can be an **alias** to a regional ALB with `evaluate_target_health = true`, which gives you health checking **for free** (no `aws_route53_health_check` resource, no $0.50/month, no health-checker IP allow-listing) by delegating to the ALB's own target health. Cheap and tempting — but see gotcha #4, `evaluate_target_health` has a subtlety on an *empty* target group.

**Downside, and it is the important one:** failover routing is **automatic**. Route 53 will fail your production traffic to the standby region the moment it decides the primary is unhealthy, at 3am, with no human in the loop, based on the opinion of health checkers on the public internet. For a warm standby whose database is an **asynchronous replica with a 2-hour RPO**, an automatic, unattended failover is a *data-loss event triggered by a network blip*. Route 53's health checkers can be wrong; they can be right about the network path from eight AWS regions and wrong about whether your customers can reach you.

### Weighted routing with 100/0 as a manual switch

Two records, same name, `weighted_routing_policy { weight = 100 }` on the primary and `{ weight = 0 }` on the standby. Failover is a weight change: 100/0 → 0/100.

AWS documents this pattern explicitly and also documents its trap, verbatim:

> If you specify nonzero weights for some records and zero weights for other records, Route 53 responds to DNS queries using only healthy records that have nonzero weights. If all the records that have a weight greater than 0 are unhealthy, then Route 53 responds to queries using the zero-weighted records.

Read that carefully, because it is counter-intuitive: a weight of 0 is **not** "never". It is "only if everything else is dead". So a weighted 100/0 pair with health checks attached behaves like a failover pair anyway. If you want a *purely manual* switch with no automatic behaviour, **do not attach health checks** to the weighted records — then 0 genuinely means 0 and the only way traffic moves is a human or pipeline changing the weight.

**Failover policy vs weighted 100/0 — the argument:**

| | Failover routing | Weighted 100/0 |
|---|---|---|
| Trigger | Automatic, on health check | Deliberate: a human or pipeline changes a number |
| Fits RPO 2h async replication? | **No.** Unattended failover to a lagging replica is uncontrolled data loss. | **Yes.** Data loss is a decision someone makes. |
| Partial / canary failover | No — it is binary | **Yes.** 90/10, 50/50, shift gradually, watch error rates, roll back by changing a number |
| Failback | Automatic when primary goes healthy — **including flapping back and forth** | Manual and deliberate, which is correct |
| Requires the Route 53 control plane at failover time? | **No** — the flip happens in the data plane | **Yes** — a weight change is a `ChangeResourceRecordSets` call. **This is the trap.** |
| Cost | ~$0.50–$1.50/mo per health check | $0 (no health checks) |

Neither is free of a flaw, and the flaws are opposite: failover routing is control-plane-free but hands the decision to a robot; weighted 100/0 keeps the decision human but needs the `us-east-1` control plane to execute it.

**The resolution is to use both ideas at once**, which is precisely what ARC routing controls are: a failover-policy record whose health check state is set by a human, through a data plane that isn't in `us-east-1`. See [[#Route 53 Application Recovery Controller]].

> [!note] Flapping
> Failover routing has no damping. If the primary oscillates healthy/unhealthy, DNS oscillates with it, and every oscillation sends a fraction of traffic into a region whose database is a *stale replica* — or, after promotion, back into a region whose database is now the *wrong* one. Split-brain by DNS flap. If you use automatic failover at all, the runbook must include "disable the health check" as the first action after a fail-over, so it cannot fail back on its own. See [[failover-runbooks]].

### Latency-based routing — wrong for active/passive, and worth explaining why

Latency-based routing answers with whichever region gives the *querying resolver* the lowest measured network latency. It is a genuinely good policy — for **active/active**. It is wrong here for three independent reasons:

1. **It spreads traffic across both regions all the time.** That is the definition of active/active. The standby in this architecture has a **2-hour-stale asynchronous database replica** ([[research-brief]]) and, in the warm-standby shape, reduced compute. Sending it 30% of real traffic because it happens to be closer to some resolver in Manchester will serve stale reads and overload scaled-down capacity. The architecture is active/**passive** — the routing policy must be too.
2. **It is not deterministic or steerable.** You cannot say "everyone goes to `eu-west-2` now". You can only remove the `eu-west-1` record and hope. Latency data changes on AWS's schedule, not yours, which means your *failover* and your *failback* both become "wait and see".
3. **It costs more per query** — $0.60 per million vs $0.40 for standard, and geolocation is $0.70 ([Route 53 pricing](https://aws.amazon.com/route53/pricing/)) — for a behaviour you actively do not want.

Latency-based routing is what this architecture would use *if* the target posture were active/active. It is not. Do not let it in through the back door because "it's what we use elsewhere".

### Geolocation / geoproximity — and the one place they genuinely matter here

Normally these would be out of scope. **They are not**, because of the shape of this specific estate, and this deserves more attention than it usually gets:

The company runs **three independent deployments with no data sharing** — a Canadian customer lives entirely in the Canadian deployment. If there is a single customer-facing hostname (`app.example.com`) shared across EU, US and CA, then something is already steering each customer to *their* region, and that something is almost certainly geolocation routing (or per-region hostnames like `eu.app.example.com`, in which case this section is N/A — confirm, see [[#Open questions]]).

That matters for failover because **failover must happen inside a pair, never across pairs**. A Canadian user whose region has failed must go to `ca-west-1` — **not** to `eu-west-1`, which would be a data-residency breach ([[data-residency-compliance]]) as well as a "your data isn't there" bug.

You therefore need **nested routing**: geolocation at the top level, failover underneath. Route 53 supports this by pointing a geolocation record at another record set (via alias-to-record-in-same-zone), or via **Traffic Flow** policy records. Note the pricing: **Traffic Flow is $50.00 per policy record per month** — for three pairs across a few hostnames that adds up fast, and hand-rolled nesting with plain records costs nothing. Also note the [Route 53 best-practices](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/best-practices-dns.html) warning:

> When using geolocation, geoproximity, or latency-based routing, always set a default, unless you want some clients to receive *no answer* responses.

A missing `Default` geolocation record is a silent outage for any client whose location Route 53 can't determine. Every geolocation record set in the estate needs `geolocation_routing_policy { country = "*" }`.

### Multivalue answer — not a failover mechanism

Returns up to eight healthy records at random, health-checked. It is client-side load spreading, not failover, and the [best-practices page](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/best-practices-dns.html) recommends it mainly to keep responses under the 512-byte UDP boundary. Not applicable to an active/passive pair. Mentioned so nobody proposes it.

## Health checks

Three types, and you will probably use all three.

| Type | What it watches | Use here |
|---|---|---|
| **Endpoint** | HTTP/HTTPS/TCP to an IP or FQDN, from checkers worldwide | The default. Points at the regional ALB or a dedicated `/health/deep` endpoint. |
| **Calculated** | Up to **255 child health checks**, healthy if *N* of *M* children are healthy | Composite regional health: "the region is up if the API **and** the database **and** the queue consumer are up". Also the inversion trick for STOP (below). |
| **CloudWatch alarm** | The *data stream* behind a CloudWatch alarm | The escape hatch for anything not reachable from the public internet — replication lag, DLQ depth, EKS node count. |
| **Recovery control** (a fourth, easy to miss) | An ARC routing control's on/off state | The manual switch. See [[#Route 53 Application Recovery Controller]]. |

### How the endpoint check actually decides

From [How Amazon Route 53 determines whether a health check is healthy](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-determining-health-of-endpoints.html), verbatim:

> If more than 18% of health checkers report that an endpoint is healthy, Route 53 considers it healthy. If 18% of health checkers or fewer report that an endpoint is healthy, Route 53 considers it unhealthy.

> The 18% value was chosen to make sure that health checkers in multiple regions consider the endpoint healthy. This prevents an endpoint from being considered unhealthy only because network conditions have isolated the endpoint from some health-checking locations.

The [AWS DR-mechanisms blog](https://aws.amazon.com/blogs/networking-and-content-delivery/creating-disaster-recovery-mechanisms-using-amazon-route-53/) puts numbers on it: checks come from **eight AWS regions**, and with the default checker set that works out to at least **3 of 16 checkers (18.75%)** having to agree the endpoint is up.

**This is deliberately biased toward "healthy".** Route 53 would rather keep you on a partly-degraded primary than fail you over because of a network partition it can see and you can't. For an active/passive pair with a lossy failover, that bias is *correct* — but it means **endpoint health checks are slow to declare a brownout**. An endpoint returning 200 for `/health` while 40% of real requests 500 will stay green forever. **Health-check a deep endpoint, not a liveness ping**, or use a CloudWatch-alarm health check on your actual 5xx rate.

Also from the same page — three sentences that each contain a trap:

> **HTTP and HTTPS health checks** – Route 53 must be able to establish a TCP connection with the endpoint within four seconds. In addition, the endpoint must respond with an HTTP status code of 2xx or 3xx within two seconds after connecting.

A `/health/deep` endpoint that queries the database must respond in **under two seconds** or it is unhealthy. Deep health checks that are too deep cause the outage they were meant to detect.

> HTTPS health checks don't validate SSL/TLS certificates, so checks don't fail if a certificate is invalid or expired.

Cross-referenced in [[aws-acm]]. Your standby can be green and un-servable simultaneously.

> Route 53 considers a new health check to be healthy until there's enough data to determine the actual status... If you chose the option to invert the health check status, Route 53 considers a new health check to be *unhealthy* until there's enough data.

**A freshly-created health check starts healthy.** If your failover plan involves `terraform apply` creating a health check, the record will point at the new target *immediately*, before any check has run. Another reason not to touch Terraform during a failover.

### Health checker source IPs

Health checkers come from **fixed, published, rarely-changing IP ranges**. If the ALB's security group or a WAF rule blocks them, every health check is red and failover routing sends 100% of traffic to the secondary on day one.

From the [Route 53 IP ranges doc](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/route-53-ip-addresses.html): download [ip-ranges.json](https://ip-ranges.amazonaws.com/ip-ranges.json) and filter `"service": "ROUTE53_HEALTHCHECKS"`. "We rarely change the IP address ranges of health checkers."

**Do not hard-code the CIDRs.** AWS publishes a managed prefix list. From [Configuring router and firewall rules](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-router-firewall-rules.html), verbatim:

> To use the AWS-managed prefix list, modify your security group to allow inbound traffic from `com.amazonaws.<region>.route53-healthchecks`, where the `<region>` is the AWS Region of your Amazon EC2 instance or resource. If you are using Route 53 health checks to check IPv6 endpoints, you should also allow inbound traffic from `com.amazonaws.<region>.ipv6.route53-healthchecks`.

```hcl
data "aws_ec2_managed_prefix_list" "route53_healthchecks" {
  provider = aws.standby
  name     = "com.amazonaws.${var.standby_region}.route53-healthchecks"
}

resource "aws_vpc_security_group_ingress_rule" "healthchecks" {
  provider          = aws.standby
  security_group_id = aws_security_group.alb.id
  prefix_list_id    = data.aws_ec2_managed_prefix_list.route53_healthchecks.id
  from_port         = 443
  to_port           = 443
  ip_protocol       = "tcp"
  description       = "Route 53 health checkers"
}
```

> [!warning] Check this exists in `ca-west-1` before relying on it
> Managed prefix lists are per-region objects. Confirm `com.amazonaws.ca-west-1.route53-healthchecks` resolves before writing it into the CA pair's module — newer regions occasionally lag on managed prefix lists. `aws ec2 describe-managed-prefix-lists --region ca-west-1 --filters Name=prefix-list-name,Values=com.amazonaws.ca-west-1.route53-healthchecks`. See [[region-pair-selection]]. Raised in [[#Open questions]].

### The STOP pattern (Standby Takes Over Primary)

Worth knowing, from the [AWS DR-mechanisms blog](https://aws.amazon.com/blogs/networking-and-content-delivery/creating-disaster-recovery-mechanisms-using-amazon-route-53/): instead of health-checking the *primary*, you put a health check on a resource in the **standby** region — typically a file in an S3 bucket or a CloudWatch metric — and invert it. Deleting the file (or tripping the metric) flips the health check and traffic moves. Per the blog, this means "you can fail over without any dependency on your primary Region."

This is the poor-man's ARC routing control: a manual, data-plane-only on/off switch for **$0.75/month** (non-AWS-endpoint health check) instead of $1,825/month. Its weakness versus ARC is that it has no safety rules, no quorum, and its own control dependency (you need S3 or CloudWatch in the standby region to be reachable). For a team that cannot justify an ARC cluster, this is the fallback, and it is genuinely good. Cover it in [[failover-runbooks]].

## The arithmetic against the 15-minute budget

This is the calculation the whole vault hangs off. Two scenarios.

### Scenario A — automatic, health-check-driven failover, fast interval

| Step | Time | Source |
|---|---|---|
| Endpoint actually breaks | t=0 | |
| Checker observes a failed request (10 s interval, request timeout ≤ 4 s connect + 2 s response) | 0–10 s | Health check docs |
| Failure threshold 3 consecutive failures | **30 s** | 10 s × 3 |
| Aggregation across checkers past the 18% line + propagation of health state to the DNS data plane | ~5–10 s (not documented as a hard number) | |
| **Route 53 starts answering with the secondary** | **≈ 35–40 s** | |
| Record TTL 60 s — worst-case resolver still serving the cached primary answer | **+ 60 s** | |
| **All compliant resolvers now answering with the standby** | **≈ 100 s ≈ 1 min 40 s** | |

**Best realistic case: under 2 minutes. Budget 3 minutes.** With the default 30 s interval and threshold 3 it is 90 s + 60 s ≈ **2 min 30 s**, still fine. Fast interval buys you 60 s for $1.00/month per health check — take it, but notice it is not the expensive part of the budget.

### Scenario B — human-decided failover (the realistic one)

| Step | Time | Notes |
|---|---|---|
| Alarm fires, human paged | 1–5 min | Not a Route 53 number. Own it in [[cloudwatch-observability]]. |
| Human confirms it's a regional event, not a bad deploy, and **decides to accept up to 2 hours of data loss** | 2–5 min | **This is the real RTO risk.** A 15-minute RTO with a human decision gate has maybe 8 minutes of machine time in it. |
| Flip the switch (ARC `UpdateRoutingControlState`, or weight change, or pipeline run) | 5–30 s | ARC data plane is fast |
| Route 53 answers change | ~10 s | Recovery-control health checks flip on state change, no polling |
| TTL 60 s expires | 60 s | |
| Standby promoted and serving (DB promotion, EKS scale-out, cache warm) | **the rest** | [[aws-rds-postgres]], [[aws-eks]] — this is where the budget actually goes |
| **DNS contribution** | **≈ 90 s** | |

**Verdict: the DNS layer comfortably fits inside 15 minutes and is not the constraint.** State this loudly, because teams routinely over-engineer the DNS layer and under-engineer database promotion. **Roughly 90 seconds of a 900-second budget is DNS.** The other 810 seconds belong to other notes.

### Where the arithmetic goes wrong

The numbers above assume clients respect DNS. They do not. See next section. **The honest statement is: Route 53 will be serving the new answer in ~90 seconds, and some fraction of your traffic will not notice for much longer than that.** "RTO met" should be defined as *"the standby is serving and the primary's failures are no longer the customer's problem"*, which requires the client-side work below, not just the record change.

## The client-side caching problem

> [!danger] The JVM DNS cache is the legendary failover-breaker
> From the [AWS SDK for Java developer guide](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/jvm-ttl-dns.html), verbatim:
>
> > On some Java configurations, the JVM default TTL is set so that it will *never* refresh DNS entries until the JVM is restarted. Thus, if the IP address for an AWS resource changes while your application is still running, it won't be able to use that resource until you *manually restart* the JVM and the cached IP information is refreshed.
>
> And from the `java.security` file itself, quoted in the same doc:
>
> > `# any negative value: caching forever`
> > `# any positive value: the number of seconds to cache an address for`
> > `# zero: do not cache`
>
> **AWS's recommendation is a TTL of 5 seconds.** `networkaddress.cache.ttl` is a **security property, not a system property — it cannot be set with `-D`.** Teams "fix" this with `-Dnetworkaddress.cache.ttl=5` on the JVM command line, it silently does nothing, and they believe they are fixed. The working options are `java.security.Security.setProperty("networkaddress.cache.ttl", "5")` before any client is constructed, editing `$JAVA_HOME/conf/security/java.security`, or the legacy `-Dsun.net.inetaddr.ttl=5` fallback (documented by Oracle as private and possibly unsupported in future releases).

If any service in the estate is JVM-based, **verifying this setting is a prerequisite for claiming a 15-minute RTO**, and it is invisible to every test that doesn't actually move an IP. Put it in the container base image, not in each application. Add it to [[failover-runbooks]] as a pre-flight check. Also set `networkaddress.cache.negative.ttl` low — a *negative* cache entry taken during the outage will pin a failure just as effectively.

Non-JVM offenders, in rough order of how often they bite:

- **Connection pools.** A DNS change reroutes nothing that is already connected. Established TCP/TLS, HTTP/2 and WebSocket connections, and every database and Redis pool, keep using the IP they resolved at connect time. Idle-timeout and max-lifetime settings on the pool are, in effect, your real failover TTL.
- **ALB HTTP client keepalive.** ARC's own best-practice page is explicit:
  > By default, Application Load Balancers set the HTTP client keepalive duration value to 3600 seconds, or 1 hour. We suggest that you lower the value to be inline with your recovery time goal for your application, for example, 300 seconds.

  **An hour.** With the default, browsers and API clients hold connections to the old ALB for up to an hour after DNS has moved. This is a one-line ALB attribute change ([[aws-alb-nlb]]) that is worth more to your RTO than any DNS tuning.
- **Resolvers that overstay the TTL.** Academic measurement backs this up rather than anecdote: Moura, Heidemann, Schmidt and Hardaker, ["Cache Me If You Can: Effects of DNS Time-to-Live"](https://ant.isi.edu/~johnh/PAPERS/Moura19b.html) (ACM IMC 2019) measures how actual cache lifetimes diverge from configured TTLs across the real resolver population. Treat the configured TTL as a *lower bound on* the *majority*, not a guarantee for *all*.
- **CDNs and reverse proxies with their own origin-resolution cache**, entirely independent of your record TTL. See [[aws-cloudfront]].
- **Go's `net.Resolver`** does not cache by default (good), but `http.Transport` connection reuse does the same job as a cache.

### TTL strategy

| Record | TTL | Why |
|---|---|---|
| Failover-participating A/AAAA/CNAME | **60 s** | AWS's own recommendation: "Setting a TTL of 60 or 120 seconds is a common choice for this scenario." |
| Alias records to ALB/CloudFront | **Not settable** — Route 53 uses the target's TTL (60 s for ELB) | One reason to prefer aliases |
| NS / delegation, MX, SPF/DKIM | **3600–86400 s** | "A value between an hour (3600s) and a day (86,400s) is a common choice." Low TTLs here buy nothing and cost queries. |
| ACM validation CNAMEs | 300 s | Never change; see [[aws-acm]] |

Cost of low TTLs: queries are **$0.40 per million** (standard) and TTL 60 vs TTL 3600 is up to a 60× multiplier on query volume for those specific names. For a name answering, say, 50 million queries/month at TTL 60, that is $20/month. **Low TTLs on failover records are not a meaningful cost.** Low TTLs on *every* record in the zone might be. Set them per-record, not per-zone.

One more from [best practices](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/best-practices-dns.html):

> When you want to make changes to critical DNS entries, we recommend that you temporarily shorten the TTLs. Then you can make the changes, observe, and rollback quickly if you need to.

For a *planned* failover test, drop the TTL to 10 s a day ahead. For an *unplanned* one you get whatever you already had — which is the argument for keeping failover records permanently at 60.

## The control-plane trap

**Route 53's control plane runs in `us-east-1`.** Every `ChangeResourceRecordSets`, every health-check create/update, every hosted-zone operation is a call to an API served from N. Virginia. The data plane — answering queries, running health checks — is global and carries a 100% availability SLA. From the [Route 53 best-practices page](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/best-practices-dns.html), verbatim:

> The data planes for Route 53, including health checks, and Amazon Application Recovery Controller (ARC) routing control are globally distributed, and are designed for 100% availability and functionality, even during severe events. They integrate with each other and don't depend on control plane functionality. While the control planes for these services, including their consoles, are generally very reliable, they're designed in a more centralized way and prioritize durability and consistency rather than high availability.

> [!danger] The US pair is directly exposed
> The US pair is **`us-east-1` → `us-west-2`**. `us-east-1` is the primary. It is also where the Route 53 control plane lives, where the CloudFront control plane lives, where the ACM control plane for CloudFront certs lives ([[aws-acm]]), and where the `CLOUDFRONT`-scope WAF control plane lives.
>
> **A `us-east-1` regional impairment is simultaneously the event you are failing away from and the event that removes your ability to change DNS.** If the US pair's failover runbook says "run the pipeline that changes the weighted record", that runbook has a circular dependency on the thing that's broken, and it will be discovered at exactly the wrong moment.
>
> The EU and CA pairs do not have this problem in the same way — but they still can't change DNS during a `us-east-1` event, so a *global* `us-east-1` impairment freezes DNS for all three pairs at once even though only one pair is affected by the outage itself.

**Three answers, in ascending order of cost:**

1. **Static stability — pre-create everything and fail over in the data plane.** The record pair already exists; only the *health state* changes; health state is a data-plane concept. Failover routing with an endpoint health check needs **zero** control-plane calls. Cost: $0.50–$1.50/month. This is the baseline and every design should have it.
2. **ARC routing controls.** Same as above, but the health state is set *by you*, on demand, through a five-region data plane. The purpose-built answer. $1,825/month per cluster. See below.
3. **Route 53 accelerated recovery** (new, announced 26 November 2025). Opt-in per public hosted zone; Route 53 keeps a copy of the zone in `us-west-2` and fails the *control plane* over within about 60 minutes of AWS detecting a `us-east-1` impairment. **Free.**

### Route 53 accelerated recovery — turn it on, but it is not your failover mechanism

From the [docs](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/accelerated-recovery.html), verbatim:

> Route 53 accelerated recovery for managing public DNS records helps you achieve a 60-minute Recovery Time Objective (RTO) if the US East (N. Virginia) Region becomes unavailable. When you turn on this feature for a Route 53 public hosted zone, you can resume making DNS changes within about 60 minutes after AWS detects that the US East (N. Virginia) Region is impaired.

**60 minutes is four times your RTO.** This feature does not let you fail over inside 15 minutes; it lets you *resume ordinary DNS operations* an hour into a regional event. It is a safety net for everything else (emergency record changes, a second failover, pointing at a status page), not the mechanism.

Turn it on anyway — it is free, per the [announcement blog](https://aws.amazon.com/blogs/networking-and-content-delivery/announcing-amazon-route-53-accelerated-recovery-for-managing-public-dns-records/) ("There is no additional charge for using this feature") — but note the constraints:

- **Public hosted zones only.** "Private hosted zones are not supported."
- **Must be enabled in advance.** "You cannot turn on accelerated recovery after a failover starts."
- Enabling "can take up to several hours" and includes "a brief period of up to several minutes where DNS changes are not accepted." **Do not enable this during a change freeze or near a release.**
- During failover, **only a subset of APIs work**: `ChangeResourceRecordSets`, `GetChange`, `ListResourceRecordSets`, `ListHostedZones` and a handful of getters. You **cannot create or delete hosted zones, or toggle DNSSEC**.
- **"Hosted zones with accelerated recovery can't be deleted."** You must disable it first. **This will surprise a `terraform destroy` in a sandbox environment** — flag it in the cookiecutter template.
- "Stranded changes": in-flight changes accepted by `us-east-1` but not yet copied to `us-west-2` come back as `NoSuchChange` from `GetChange` after failover and must be resubmitted. A CloudFormation or Terraform apply in flight at the moment of impairment will hang and must be retried.
- Limit is around **10 zones per account** enabled simultaneously (per the announcement blog) — fine here, but check before enabling estate-wide.

Enable it with `aws route53 update-hosted-zone-features --enable-accelerated-recovery --hosted-zone-id Z123456789`. As of writing there is no dedicated Terraform resource for it; treat it as a one-time out-of-band setting with an `aws_route53_zone` lifecycle note, or wrap it in a small `null_resource`/`aws_cloudcontrolapi_resource`. Verify current provider support before writing the HCL — see [[#Open questions]].

## Route 53 Application Recovery Controller

ARC is AWS's purpose-built answer to exactly this problem, and it should be evaluated honestly rather than assumed.

### What it is

From the [routing control docs](https://docs.aws.amazon.com/r53recovery/latest/dg/routing-control.html), verbatim:

> *Routing controls* are simple on-off switches that enable you to switch your client traffic from one Regional replica to another. The traffic rerouting is accomplished by *routing control health checks* that are set up with Amazon Route 53 DNS records.

> The routing control components in ARC are: clusters, control panels, routing controls, and routing control health checks... **Each cluster in ARC is a data plane of endpoints in five AWS Regions.**

Concretely:

- A **cluster** is a five-region, quorum-based data plane. The [DR-mechanisms blog](https://aws.amazon.com/blogs/networking-and-content-delivery/creating-disaster-recovery-mechanisms-using-amazon-route-53/) describes it as operating "on a quorum model, which means that the cluster and routing controls continue to function... even if up to two Regional endpoints are unavailable."
- A **routing control** is an on/off boolean hosted on that cluster.
- A **routing control health check** (`aws_route53_health_check` with `type = "RECOVERY_CONTROL"`) exposes that boolean to Route 53 as a health check, which you attach to a failover or weighted record.
- **Safety rules** prevent stupid outcomes. **Assertion rules**: "at least one region must be ON at all times" — blocks you from turning everything off. **Gating rules**: "you cannot turn on `us-west-2` unless a separate `enable-failover` control is on" — a two-key switch. Both can be overridden explicitly during a real emergency ("Overriding safety rules to reroute traffic").

### What it buys you that nothing else does

1. **A failover switch whose data plane is not in `us-east-1`** and is not even in one region. Five endpoints, quorum of three.
2. **Human-decided, not robot-decided.** Fits the 2-hour-RPO async-replica reality. No flapping.
3. **Safety rules.** The thing that stops a tired engineer at 3am from turning both regions off, or turning the standby on before the database is promoted. Genuinely valuable and not reproducible with plain records.
4. **Atomic multi-control updates.** `UpdateRoutingControlStates` (plural) flips several controls in one transaction, so you never sit in a half-switched state.

### What it costs

From the [ARC pricing page](https://aws.amazon.com/application-recovery-controller/pricing/):

| Item | Price | Monthly (730 h) |
|---|---|---|
| **Routing control cluster** | **$2.50 per hour per cluster** | **≈ $1,825 / month** |
| Readiness check | $0.045 per hour per check | ≈ $32.85 / month each |
| **Region switch plan** | **$70 per plan per month** | $70 |
| Zonal shift / zonal autoshift | **"There is no additional charge"** | $0 |

**The cluster is the headline number and it is charged per cluster, not per routing control.** A cluster hosts many routing controls, and **there is a maximum of 2 clusters per account**. So **one cluster can serve all three pairs** — EU, US and CA — for the same $1,825/month. Per-pair that's ~$608/month; against the whole standby estate it may be a rounding error or it may be 30% of the DR budget. Take it to [[cost-modelling]] with the real standby figures.

> [!warning] Readiness checks are effectively gone
> The ARC pricing page carries this notice: **"Readiness check will no longer be available to new customers starting April 30, 2026."** That date has passed. If the company is not already an ARC customer, **readiness checks are not available to you** and the "is my standby actually ready?" function must be built another way — CloudWatch composite alarms, a synthetic canary against the standby, or [[aws-config]] conformance packs. Do not write a design doc that depends on ARC readiness checks. Verify current availability against your account before planning either way.

### ARC Region switch — the cheap option nobody knows about

Announced August 2025, priced at **$70 per plan per month**. Where routing controls give you a *switch*, **Region switch gives you the whole orchestrated runbook as a managed, versioned, testable plan** made of "execution blocks":

- an **ARC routing control** block (flip routing controls),
- a **Route 53 health check** block (create health checks in ARC and associate them with DNS records) — which means **you can drive DNS failover from a Region switch plan without paying for a routing control cluster at all**,
- EKS resource scaling (scale pods in the target region to a percentage of source-region capacity),
- ASG/ECS scaling, Lambda, RDS and Aurora blocks, custom action blocks.

Per AWS's December 2025 update, Region switch now has **post-recovery workflows, native RDS execution blocks, and AWS provider for Terraform support** — so it is declarable in the existing monorepo rather than click-ops.

This is potentially a much better fit than raw routing controls for this project, because the 15-minute RTO problem here is **not** "flip DNS" (that's 90 seconds) — it is **"do the eleven other things in the right order, fast, at 3am, correctly"**. That is exactly what Region switch is for, at 4% of the cluster price. **Investigate it properly before committing to a $1,825/month cluster.** Flagged in [[#Decisions to make]] and [[failover-orchestration]].

### ARC operational discipline (from AWS's own best practices)

The [routing control best-practices page](https://docs.aws.amazon.com/r53recovery/latest/dg/route53-arc-best-practices.regional.html) is unusually specific and all of it belongs in [[failover-runbooks]]:

> **Bookmark or hard code your five Regional cluster endpoints and routing control ARNs.** ... During a failure event, you might not be able to access some API operations, including ARC API operations that are not hosted on the extremely reliable data plane cluster.

> **Choose one of your endpoints at random to update your routing control states.** Routing controls provide five Regional endpoints to ensure high availability... To achieve their full resilience, it's important to have retry logic that can use all five endpoints as necessary.

> **Use the extremely reliable data plane API to list and update routing control states, not the console.**

> **Keep purpose-built, long-lived AWS credentials secure and always accessible.** ... Create IAM long-lived credentials specifically for DR tasks, and keep the credentials securely in an on-premises physical safe or a virtual vault.

That last one deserves a pause. **AWS is telling you to keep break-glass long-lived IAM access keys in a vault, because your SSO/identity provider is itself a dependency that may be down.** It contradicts normal security guidance and it is correct for this specific purpose. Take it to [[aws-iam]] and to whoever owns security policy — it is a decision, not a default. If federated access is the only path to AWS and the IdP is unavailable, your $1,825/month cluster is unreachable.

**The five cluster endpoints must be baked into the failover tool as literal strings.** Discovering them via `DescribeCluster` is a *control-plane* call and defeats the entire point.

## Private hosted zones across regions

A private hosted zone is a global object associated with one or more **VPCs**, which are regional. So a PHZ can serve `internal.example.com` to VPCs in `eu-west-1` **and** `eu-west-2` simultaneously — you do not need two zones, and you should not have two.

**Recommendation: one private hosted zone per environment, associated with the VPCs in both regions of the pair, with region-specific records distinguished by name** (`api.eu-west-1.internal.example.com`) rather than by having two zones with conflicting answers for the same name. Two zones with the same name and different contents is a debugging nightmare and breaks the moment a VPC is peered to both.

Terraform has a well-known sharp edge here. `aws_route53_zone` accepts an inline `vpc { }` block, and `aws_route53_zone_association` manages associations as separate resources. **Using both causes a permanent diff** — the zone resource wants to remove associations it doesn't know about. Pick one. For multi-region, pick `aws_route53_zone_association`, because the inline block is awkward with aliased providers.

```hcl
# Global object. Created with the default provider; region is irrelevant.
# Associate ONLY the primary VPC inline, then add others via the
# association resource. The lifecycle ignore is mandatory.
resource "aws_route53_zone" "internal" {
  name = "internal.${var.env}.example.com"

  vpc {
    vpc_id     = module.vpc_primary.vpc_id
    vpc_region = var.primary_region
  }

  lifecycle {
    ignore_changes = [vpc]
  }
}

# The standby VPC, in the paired region. vpc_region is what makes this
# cross-region; the provider used here just needs to be able to see the VPC.
resource "aws_route53_zone_association" "standby" {
  provider = aws.standby

  zone_id    = aws_route53_zone.internal.zone_id
  vpc_id     = module.vpc_standby.vpc_id
  vpc_region = var.standby_region
}
```

Notes that will save an afternoon:

- **`vpc_region` is required when the VPC is not in the provider's region.** Omit it and you get a confusing "VPC not found" error against the wrong region.
- **Cross-*account* association is a two-step handshake**, not a single resource: `CreateVPCAssociationAuthorization` in the zone's account, then `AssociateVPCWithHostedZone` in the VPC's account. Terraform models this as `aws_route53_vpc_association_authorization` + `aws_route53_zone_association` with two providers. If the standby lives in a different account ([[aws-acm]] raises the same question), budget for this.
- **`enableDnsHostnames` and `enableDnsSupport` must be true on both VPCs** or resolution silently fails in one region only.
- **Accelerated recovery does not cover private hosted zones.** During a `us-east-1` impairment you cannot change internal DNS at all. Design internal names to be static — which is another argument for `api.<region>.internal.example.com` over a single name that has to be repointed.

## Terraform implementation

### Module shape for a templated monorepo

The DNS layer should be **one stack per (account, public zone)**, not one per region. Regional stacks publish outputs (ALB DNS name, zone ID); the DNS stack consumes them via remote state or SSM. That keeps the `ChangeResourceRecordSets` blast radius in one place and makes "who can change production DNS" an answerable question.

```hcl
terraform {
  required_version = ">= 1.9"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.0" }
  }
}

provider "aws" {
  region = var.primary_region
  default_tags { tags = local.common_tags }
}

provider "aws" {
  alias  = "standby"
  region = var.standby_region
  default_tags { tags = local.common_tags }
}
```

### Option 1 — failover records with alias + `evaluate_target_health` (cheapest, automatic)

```hcl
data "aws_route53_zone" "public" {
  name         = var.public_domain
  private_zone = false
}

resource "aws_route53_record" "api_primary" {
  zone_id        = data.aws_route53_zone.public.zone_id
  name           = "api.${var.public_domain}"
  type           = "A"
  set_identifier = "primary-${var.primary_region}"

  failover_routing_policy { type = "PRIMARY" }

  alias {
    name    = var.primary_alb_dns_name
    zone_id = var.primary_alb_zone_id
    # Free health checking, delegated to the ALB's own target health.
    # Caveat: an ALB with ZERO healthy targets in ZERO target groups is
    # reported HEALTHY. See Gotchas.
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "api_secondary" {
  zone_id        = data.aws_route53_zone.public.zone_id
  name           = "api.${var.public_domain}"
  type           = "A"
  set_identifier = "secondary-${var.standby_region}"

  failover_routing_policy { type = "SECONDARY" }

  alias {
    name                   = var.standby_alb_dns_name
    zone_id                = var.standby_alb_zone_id
    evaluate_target_health = true
  }
}
```

Nine lines per region, $0/month, no control-plane dependency at failover, ~100 s failover. **This is the floor, and it is a good floor.** Its only real flaw is that it is automatic.

### Option 2 — explicit health check, deep endpoint, fast interval

```hcl
resource "aws_route53_health_check" "primary_api" {
  fqdn              = "api-direct.${var.primary_region}.${var.public_domain}"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health/deep"   # must answer in < 2 s, see Gotchas
  failure_threshold = 3
  request_interval  = 10               # $1.00/mo optional feature; buys 60 s

  # Without this you get the default 8-region checker set. Narrowing the
  # regions reduces noise but also weakens the 18% quorum — leave default
  # unless you have a reason.
  # regions = ["eu-west-1", "us-east-1", "ap-southeast-1"]

  tags = { Name = "primary-api-${var.env}" }
}

# Region-level health: API AND async workers AND replication lag all OK.
resource "aws_route53_health_check" "primary_region" {
  type                   = "CALCULATED"
  child_health_threshold = 3
  child_healthchecks = [
    aws_route53_health_check.primary_api.id,
    aws_route53_health_check.primary_workers.id,
    aws_route53_health_check.primary_replication_lag.id, # CLOUDWATCH_METRIC
  ]
  tags = { Name = "primary-region-${var.env}" }
}

# The CloudWatch-backed child: replication lag inside the 2 h RPO.
# NOTE: cloudwatch_alarm_region is the region the ALARM lives in, and
# Route 53 does NOT support cross-account CloudWatch alarms.
resource "aws_route53_health_check" "primary_replication_lag" {
  type                            = "CLOUDWATCH_METRIC"
  cloudwatch_alarm_name           = aws_cloudwatch_metric_alarm.replica_lag.alarm_name
  cloudwatch_alarm_region         = var.primary_region
  insufficient_data_health_status = "LastKnownStatus" # NOT "Unhealthy" — see Gotchas
  tags = { Name = "primary-replication-lag-${var.env}" }
}
```

### Option 3 — ARC routing controls (the manual, data-plane switch)

```hcl
# ARC routing-control CONFIG APIs are hosted in us-west-2, not us-east-1.
# This is a real and easily-missed detail: the ARC control plane's home
# region differs from Route 53's.
provider "aws" {
  alias  = "arc"
  region = "us-west-2"
}

resource "aws_route53recoverycontrolconfig_cluster" "main" {
  provider = aws.arc
  name     = "${var.org}-failover"
  # $2.50/hour == ~$1,825/month. Max 2 clusters per ACCOUNT.
  # Share ONE cluster across all three region pairs.
}

resource "aws_route53recoverycontrolconfig_control_panel" "eu" {
  provider    = aws.arc
  name        = "eu-pair"
  cluster_arn = aws_route53recoverycontrolconfig_cluster.main.arn
}

resource "aws_route53recoverycontrolconfig_routing_control" "eu_primary" {
  provider          = aws.arc
  name              = "eu-west-1"
  cluster_arn       = aws_route53recoverycontrolconfig_cluster.main.arn
  control_panel_arn = aws_route53recoverycontrolconfig_control_panel.eu.arn
}

resource "aws_route53recoverycontrolconfig_routing_control" "eu_standby" {
  provider          = aws.arc
  name              = "eu-west-2"
  cluster_arn       = aws_route53recoverycontrolconfig_cluster.main.arn
  control_panel_arn = aws_route53recoverycontrolconfig_control_panel.eu.arn
}

# Assertion rule: never allow BOTH controls off. The 3am guardrail.
resource "aws_route53recoverycontrolconfig_safety_rule" "eu_min_one_on" {
  provider          = aws.arc
  name              = "eu-at-least-one-region-on"
  control_panel_arn = aws_route53recoverycontrolconfig_control_panel.eu.arn
  wait_period_ms    = 5000

  asserted_controls = [
    aws_route53recoverycontrolconfig_routing_control.eu_primary.arn,
    aws_route53recoverycontrolconfig_routing_control.eu_standby.arn,
  ]

  rule_config {
    type      = "ATLEAST"
    threshold = 1
    inverted  = false
  }
}

# Expose each routing control to Route 53 as a health check.
resource "aws_route53_health_check" "eu_primary_rc" {
  type                = "RECOVERY_CONTROL"
  routing_control_arn = aws_route53recoverycontrolconfig_routing_control.eu_primary.arn
}

resource "aws_route53_health_check" "eu_standby_rc" {
  type                = "RECOVERY_CONTROL"
  routing_control_arn = aws_route53recoverycontrolconfig_routing_control.eu_standby.arn
}

# Records are non-alias here so the health check can be attached directly.
# (With an alias you'd point at an intermediate record set instead.)
resource "aws_route53_record" "api_primary_arc" {
  zone_id         = data.aws_route53_zone.public.zone_id
  name            = "api.${var.public_domain}"
  type            = "CNAME"
  ttl             = 60
  set_identifier  = "primary"
  records         = [var.primary_alb_dns_name]
  health_check_id = aws_route53_health_check.eu_primary_rc.id

  failover_routing_policy { type = "PRIMARY" }
}

resource "aws_route53_record" "api_secondary_arc" {
  zone_id         = data.aws_route53_zone.public.zone_id
  name            = "api.${var.public_domain}"
  type            = "CNAME"
  ttl             = 60
  set_identifier  = "secondary"
  records         = [var.standby_alb_dns_name]
  health_check_id = aws_route53_health_check.eu_standby_rc.id

  failover_routing_policy { type = "SECONDARY" }
}
```

The failover action is then a **data-plane call that touches no Terraform**:

```bash
# Hard-code these five. Do NOT call DescribeCluster at 3am — control plane.
ENDPOINTS=(
  https://xxxx.route53-recovery-cluster.eu-west-1.amazonaws.com/v1
  https://xxxx.route53-recovery-cluster.us-east-1.amazonaws.com/v1
  https://xxxx.route53-recovery-cluster.us-west-2.amazonaws.com/v1
  https://xxxx.route53-recovery-cluster.ap-northeast-1.amazonaws.com/v1
  https://xxxx.route53-recovery-cluster.ap-southeast-2.amazonaws.com/v1
)
# Pick at random and retry the others on failure, per AWS best practice.
for ep in $(printf '%s\n' "${ENDPOINTS[@]}" | shuf); do
  aws route53-recovery-cluster update-routing-control-states \
    --region "$(sed -E 's#.*cluster\.([a-z0-9-]+)\.amazonaws.*#\1#' <<<"$ep")" \
    --endpoint-url "$ep" \
    --update-routing-control-state-entries \
      "[{\"RoutingControlArn\":\"$PRIMARY_ARN\",\"RoutingControlState\":\"Off\"},
        {\"RoutingControlArn\":\"$STANDBY_ARN\",\"RoutingControlState\":\"On\"}]" \
    && break
done
```

> [!tip] Note the ARC control-plane anomaly
> `aws_route53recoverycontrolconfig_*` resources are **`us-west-2`**-hosted, not `us-east-1` — consistent with the [Fault Isolation Boundaries whitepaper](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/appendix-b---edge-network-global-service-guidance.html) noted in [[aws-acm]] for Global Accelerator. A useful accident: ARC *configuration* survives a `us-east-1` event too.

### The variable surface for the cookiecutter

```hcl
variable "failover_mechanism" {
  type    = string
  default = "health_check"   # health_check | arc_routing_control | weighted_manual
  validation {
    condition     = contains(["health_check", "arc_routing_control", "weighted_manual"], var.failover_mechanism)
    error_message = "Unsupported failover_mechanism."
  }
}

variable "failover_record_ttl" {
  type        = number
  default     = 60
  description = "Only for non-alias records. AWS recommends 60-120s for health-checked failover records."
}

variable "health_check_interval" {
  type        = number
  default     = 10  # 10 = fast interval, billed as an optional feature
  validation {
    condition     = contains([10, 30], var.health_check_interval)
    error_message = "Route 53 supports only 10 or 30 second intervals."
  }
}

variable "health_check_failure_threshold" {
  type    = number
  default = 3
}
```

Keeping `failover_mechanism` as a variable lets non-production environments run the cheap `health_check` path while production runs ARC — **and lets you dry-run the ARC path in staging**, which is the only way anyone will ever trust it.

## Migration path from single-region

Every step here is additive. **Nothing in this migration forces a resource replacement**, with one exception noted below.

1. **Inventory and import.** Get every existing record into Terraform state. `aws route53 list-resource-record-sets --hosted-zone-id Z...`. Import format is `ZONEID_NAME_TYPE`, and for records with a `set_identifier` it is `ZONEID_NAME_TYPE_SETID`. Confirm an **empty plan** before doing anything else.
2. **Lower the TTL on the names that will participate in failover — and only those.** Change `ttl` from whatever it is to `60`. Apply. **This is an in-place update, not a replacement.** Then **wait for the old TTL to drain** before assuming the new one is in effect: if the record was at 86400, resolvers may cache the old value for a day. Do this at least 48 hours before any failover test.
3. **Enable accelerated recovery on the public hosted zone.** Free, out-of-band, takes up to several hours, and includes a few minutes where changes are rejected. Do it in a quiet window, well before step 4.
4. **Add the standby ALB** ([[aws-alb-nlb]]) and its certificate ([[aws-acm]]). No DNS change yet.
5. **Allow the health checker prefix list** through the standby ALB's security group. Do this *before* creating health checks, or you'll spend an hour debugging a red check that is actually a firewall.
6. **Create health checks against both regions, attached to nothing.** Watch them for a week. This is free reconnaissance: you learn your primary's real health-check flap rate before it can affect traffic, and you learn whether the standby is actually serving.
7. **Convert the simple record into a failover pair.** This is the one step to be careful about.
   > [!warning] Adding `set_identifier` forces replacement
   > `set_identifier` is a `ForceNew` attribute on `aws_route53_record`. Converting `api.example.com` from a plain A record into a PRIMARY failover record **destroys and recreates the record**. Terraform's default ordering is destroy-then-create, so there is a window — typically sub-second, but real — where **the name does not resolve at all**, and any resolver that queries during it will cache an NXDOMAIN for the negative-caching TTL (your SOA minimum).
   >
   > **Mitigation:** set the zone's SOA minimum TTL low beforehand, and do the conversion in a low-traffic window. Alternatively, migrate via a new name (`api-v2`), prove it, then CNAME over — slower but zero-risk. `create_before_destroy` does **not** help, because the old and new records have the same name and type and cannot coexist.
8. **Attach the health checks.** Now failover is live. Traffic behaviour is unchanged while the primary is healthy.
9. **Run a real failover test.** Force the primary health check unhealthy (invert it, or point it at a deliberately-broken path), watch traffic move, watch it move back. **Measure the actual time**, with a client outside your network, and compare it against the arithmetic above. If your measured number is far worse than 100 seconds, you have found your client-caching problem — which is the point of the exercise.
10. **Only then** decide whether to add ARC. The health-check path is a prerequisite for the ARC path anyway; ARC just changes what drives the health check.

## Failover procedure

Automated (no human): none, by design. The recommendation below is that the Route 53 layer is *manually triggered*, because promoting an async replica is a data-loss decision.

**Human decision (the expensive minutes):**
1. Confirm this is a regional impairment, not a bad deploy. Check the AWS Health Dashboard from a machine outside the affected region.
2. Confirm the standby is actually ready — database replica lag inside RPO, EKS nodes present, queues drained. This gate is what ARC readiness checks used to do; build it as a dashboard ([[cloudwatch-observability]]).
3. **Accept the data loss.** Explicitly, by a named person.

**Mechanical (the cheap seconds):**
4. Promote the database ([[aws-rds-postgres]]) — this, not DNS, is the long pole.
5. Scale the standby to full capacity ([[aws-eks]]).
6. Flip the switch: `update-routing-control-states`, or set the weights, or invert the primary health check.
7. **Disable the primary's health check** so nothing fails back automatically while you're working.
8. Watch the standby ALB's request count rise and the primary's fall. If the primary's doesn't fall, you have a client-caching problem — the ALB keepalive and the JVM TTL are the first two things to check.

## Failback

Harder than failover and routinely forgotten.

- **Failback is a planned change, never automatic.** With failover routing and a health check, the moment the primary goes healthy Route 53 will *silently move production back* — into a region whose database is now a stale former-primary. **This is the single most dangerous default in this note.** Either use ARC/manual switching, or make step 7 above ("disable the primary health check") non-optional.
- The data has to go back first. Re-establishing replication in the opposite direction is the real work and belongs to [[aws-rds-postgres]]; DNS is the last five minutes of a multi-day process.
- **Failback should be gradual**, which is the one argument for weighted records over failover records: 10/90, watch, 50/50, watch, 100/0. Failover routing cannot do this. Consider keeping a weighted record pair alongside the failover pair for *planned* moves and using failover/ARC only for emergencies.
- Re-enable health checks, re-verify the TTLs are still 60 (someone will have "optimised" them), and re-run the ARC safety rules.

## Gotchas

1. **`set_identifier` is `ForceNew`.** Converting a plain record into a failover/weighted record destroys and recreates it, with a brief non-resolution window and possible NXDOMAIN negative caching. See migration step 7. This is the loud one the brief asks for.
2. **A weight of 0 does not mean "never".** If every non-zero-weighted record is unhealthy, Route 53 serves the zero-weighted one. Quoted verbatim above. Weighted 100/0 is a manual switch **only if no health checks are attached**.
3. **`insufficient_data_health_status` defaults matter enormously.** A CloudWatch-alarm health check whose metric stops being published — which is exactly what happens in a regional impairment — hits `INSUFFICIENT_DATA`. Setting this to `Unhealthy` gives you automatic failover on *monitoring* failure; `Healthy` gives you a failover that never fires. `LastKnownStatus` is usually least-worst. Decide deliberately; the default will not match your intent.
4. **`evaluate_target_health = true` on an ALB alias does not mean what you think.** It reflects the ALB's target health — but an ALB with **no target groups at all**, or target groups with **no registered targets**, is reported as healthy. A standby whose EKS cluster has scaled to zero can therefore present a green alias. Belt-and-braces: a real health check against a real application path.
5. **Route 53 does not support cross-account CloudWatch alarms.** Stated verbatim in the health-check docs. If monitoring lives in a central observability account, CloudWatch-metric health checks cannot see it. Discover this now, not during implementation.
6. **A new health check starts *healthy*** (or *unhealthy* if inverted), before any data exists. Never create health checks as part of a failover.
7. **Deep health checks must answer within 2 seconds** after a 4-second connect. A `/health/deep` that does three database round-trips will time out under exactly the load conditions you built it for.
8. **HTTPS health checks don't validate certificates.** An expired standby cert is green. Cross-ref [[aws-acm]].
9. **Health checker IPs must be allowed.** Use the managed prefix list, not hard-coded CIDRs. Verify the prefix list exists in `ca-west-1`.
10. **The JVM caches DNS forever in some configurations, and `-Dnetworkaddress.cache.ttl` does not work** because it is a security property, not a system property. The #1 cause of "we failed over and nothing happened".
11. **The ALB's default HTTP client keepalive is 3600 seconds.** Lower it (AWS suggests 300) or clients stay pinned to the dead region for an hour.
12. **Accelerated recovery blocks zone deletion.** "Hosted zones with accelerated recovery can't be deleted." Will break ephemeral/sandbox environments that `terraform destroy`.
13. **Accelerated recovery gives a 60-minute RTO, not 15.** It is not a failover mechanism. Enable it anyway; it's free.
14. **ARC readiness checks are unavailable to new customers as of 30 April 2026.** Don't design around them.
15. **ARC's config plane is `us-west-2`, Route 53's is `us-east-1`.** Two different "the control plane is in..." facts in one architecture.
16. **`DescribeCluster` is a control-plane call.** Hard-code the five ARC endpoints in the runbook.
17. **`aws_route53_zone`'s inline `vpc` block fights `aws_route53_zone_association`.** Use `lifecycle { ignore_changes = [vpc] }` or a permanent diff is yours forever.
18. **Missing geolocation `Default` record = no answer for unlocatable clients.** A silent, partial, very confusing outage.
19. **`ChangeResourceRecordSets` has an account-wide API rate limit (5 requests/second).** A `terraform apply` touching hundreds of records in a monorepo will throttle. Another reason to isolate the DNS stack.
20. **Alias records ignore your `ttl`.** They inherit the target's — 60 s for an ELB. Setting `ttl` alongside `alias` is a Terraform error, and people are surprised their "TTL strategy" has no effect on alias records. (Here that's fine; 60 is what you wanted.)

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| **Failover trigger** | **A: Automatic** — failover routing + endpoint health checks. $0.50–$1.50/mo. No control-plane dependency. Fires at 3am with nobody watching. | **B: Manual** — ARC routing control (or STOP, or weighted flip). A human accepts the data loss. Costs money (ARC) or adds a dependency (weighted flip needs `us-east-1`). | **B, implemented as ARC routing controls or the STOP pattern.** RPO is 2 hours of *asynchronous* replication: an unattended failover is an unattended data-loss event, and failover routing will also silently fail *back* into a stale ex-primary. Keep the health checks — but use them to *alarm*, not to *route*. |
| **ARC routing control cluster vs plain records flipped by a pipeline** | **A: ARC cluster.** Five-region quorum data plane, works when `us-east-1` doesn't, safety rules prevent double-off, atomic multi-control updates. **$2.50/hr = ~$1,825/month**, one cluster covers all three pairs (max 2/account). | **B: Weighted/failover records flipped by a Lambda or pipeline.** ~$0/month. But the flip is `ChangeResourceRecordSets` — a **`us-east-1` control-plane call**. For the US pair, whose primary *is* `us-east-1`, the failover mechanism depends on the region it's escaping. Accelerated recovery restores this ability, but **after 60 minutes**, i.e. 4× the RTO. | **A for production, B for non-production** — *if* the DR budget can carry $1,825/month (take it to [[cost-modelling]]). **If it cannot, do not fall back to B: use the STOP pattern instead** — an inverted health check on an S3 object in the standby region, ~$0.75/month, manual, and **also control-plane-free**. B is the only option here that is genuinely broken for the US pair, and it is the one teams reach for first. |
| **ARC routing controls vs ARC Region switch** | **A: Routing controls.** $1,825/mo. Just a switch — you still hand-write the surrounding orchestration. | **B: Region switch plans.** **$70/plan/month.** Orchestrates the whole sequence (routing controls, Route 53 health checks, EKS/ASG/ECS scaling, RDS), is versioned, testable, has Terraform provider support and post-recovery workflows. | **Evaluate B first.** The 15-minute problem is orchestration, not DNS, and Region switch is 4% of the cluster price. Its Route 53 health check execution block may remove the need for a routing control cluster entirely. **This is the single highest-value open investigation in this note** — see [[failover-orchestration]]. |
| **Record TTL on failover names** | A: 30 s — faster, ~2× the query cost on those names | B: 60 s — AWS's own recommendation | **B (60 s).** The 30 s you'd save is ~3% of a 900 s budget and is dwarfed by client-side caching. Spend the effort on the ALB keepalive and the JVM setting instead. |
| **Alias + `evaluate_target_health` vs explicit health check** | A: Alias, free, zero resources | B: Explicit health check on a deep path, $0.50–$1.50/mo | **B for the customer-facing name, A everywhere else.** A can't see application-level failure and reports an empty ALB as healthy. |
| **Accelerated recovery** | A: Enable | B: Don't | **A.** Free, and the alternative is being unable to touch DNS at all during the most likely failure scenario. Just don't enable it on zones that sandbox pipelines destroy. |
| **Nested geo → failover routing** | A: Hand-rolled nested records | B: Traffic Flow policy records, **$50/policy record/month** | **A**, unless the policy graph gets genuinely complex. $50/record/month across three pairs and several hostnames is real money for something plain records do. |

## Cost

Per month, for the DNS layer of **one** region pair, at a modest query volume.

| Item | Cost |
|---|---|
| Public hosted zone | $0.50 (first 25 zones) |
| Standard queries, 50M/month | $20.00 |
| Endpoint health check (AWS endpoint), 2 regions | $1.00 |
| Fast-interval optional feature ×2 | $2.00 |
| Calculated health check ×2 | $1.00 |
| CloudWatch-metric health check ×2 | $1.00 |
| **Subtotal — health-check failover** | **≈ $25.50/month** |
| Route 53 accelerated recovery | **$0.00** |
| **ARC routing control cluster (shared across all three pairs)** | **$1,825.00/month** |
| ARC Region switch plan, per plan | $70.00/month |
| Traffic Flow policy record, if used | $50.00 each |

**The levers, in order:**
1. **The ARC cluster is the only large number.** $1,825/month = ~$21,900/year. Against three pairs it is $608/pair/month. Whether that is reasonable depends entirely on the standby's total cost — if the three warm standbys cost $40k/month, ARC is 4.5% for the one component that works when `us-east-1` doesn't. If they cost $4k/month, ARC costs more than a standby region and the STOP pattern is the answer.
2. **Free tier: 50 AWS-endpoint health checks** are included. This estate will not exceed it. Health checks are effectively free; the $0.50 figures above are conservative.
3. **Query cost scales with traffic, not with TTL alone.** Only lower TTLs on records that participate in failover.
4. **Traffic Flow at $50/policy record/month** is the sneaky one. Easy to enable in the console, easy to forget, expensive at scale. Hand-roll nested records.

## Open questions

1. **Is there one global hostname with geolocation routing, or per-region hostnames?** This decides whether the failover design is a simple record pair or a nested geo→failover policy, and whether Traffic Flow is in play. Blocks the record design.
2. **Can the DR budget carry $1,825/month for an ARC cluster?** Needs [[cost-modelling]] with real standby figures before the decision above can be closed.
3. **Is the company already an ARC customer?** Determines whether readiness checks are available at all (cut off for new customers 30 April 2026).
4. **Has anyone evaluated ARC Region switch?** At $70/plan/month with Terraform support, it may dominate both alternatives. Highest-value item here.
5. **Which services are JVM-based, and what is `networkaddress.cache.ttl` in the container base image today?** A single answer that could invalidate the entire RTO. Check before designing anything else.
6. **What is the ALB `client_keep_alive` setting today?** If it is the 3600 s default, that alone exceeds the RTO by 4×.
7. **Does `com.amazonaws.ca-west-1.route53-healthchecks` exist?** Blocks the CA pair's security group module.
8. **Is the public hosted zone in the same account as the workloads, and who can change it?** Determines whether the "one DNS stack" recommendation is even possible.
9. **Does the standby live in a different AWS account?** If so, private hosted zone association becomes a two-resource cross-account handshake.
10. **Is break-glass long-lived IAM credential storage acceptable to security?** AWS explicitly recommends it for ARC failover. If the answer is no, the ARC failover path has an IdP dependency and the cluster's resilience is partly wasted.
11. **Does the AWS provider yet support Route 53 accelerated recovery declaratively?** If not, document it as an out-of-band setting with a drift check.

## Sources

- [How Amazon Route 53 determines whether a health check is healthy](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-determining-health-of-endpoints.html) — the 18% quorum rule verbatim, 10s/30s intervals, the 4s connect + 2s response budget, calculated and CloudWatch-alarm health check semantics, "a new health check is healthy until there's enough data", no cross-account alarms, HTTPS checks don't validate certs.
- [Active-active and active-passive failover](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-types.html) — failover routing semantics and the verbatim caveat that zero-weighted records are served when all non-zero-weighted records are unhealthy.
- [Best practices for Amazon Route 53 DNS](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/best-practices-dns.html) — TTL guidance (60–172,800 range; 60–120 s for failover records; 3600–86400 for delegations), the data-plane-vs-control-plane recommendation verbatim, the geolocation default-record warning, alias-over-CNAME, the 512-byte response guidance.
- [IP address ranges of Amazon Route 53 servers](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/route-53-ip-addresses.html) — `ROUTE53_HEALTHCHECKS` in ip-ranges.json, "we rarely change" these ranges.
- [Configuring router and firewall rules for Route 53 health checks](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-router-firewall-rules.html) — the exact managed prefix list names `com.amazonaws.<region>.route53-healthchecks` and the IPv6 variant, verbatim.
- [Enabling accelerated recovery for managing public DNS records](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/accelerated-recovery.html) — the 60-minute RTO statement, the `us-west-2` zone copy, the allowed-API list during failover, stranded changes, the can't-delete-the-zone constraint, the CLI commands.
- [Announcing Amazon Route 53 accelerated recovery for managing public DNS records](https://aws.amazon.com/blogs/networking-and-content-delivery/announcing-amazon-route-53-accelerated-recovery-for-managing-public-dns-records/) — 26 November 2025 announcement; "There is no additional charge for using this feature"; ~10 zones per account.
- [Amazon Route 53 pricing](https://aws.amazon.com/route53/pricing/) — $0.50/hosted zone/month, $0.40 per million standard queries, $0.60 latency-based, $0.70 geolocation, health checks $0.50 (AWS) / $0.75 (non-AWS) plus $1.00/$2.00 per optional feature, 50 free AWS-endpoint health checks, Traffic Flow $50/policy record/month.
- [Application Recovery Controller pricing](https://aws.amazon.com/application-recovery-controller/pricing/) — **$2.50/hour/cluster**, $0.045/hour/readiness check, **$70/month per Region switch plan**, zonal shift free, and the notice that readiness checks are unavailable to new customers from 30 April 2026.
- [Routing control in ARC](https://docs.aws.amazon.com/r53recovery/latest/dg/routing-control.html) — "simple on-off switches", the components list, and "Each cluster in ARC is a data plane of endpoints in five AWS Regions".
- [Best practices for routing control in ARC](https://docs.aws.amazon.com/r53recovery/latest/dg/route53-arc-best-practices.regional.html) — the five-endpoint rotation rule, use the data plane not the console, hard-code endpoints, break-glass long-lived credentials, TTL 60–120 s, and the ALB 3600 s keepalive default with the 300 s suggestion.
- [Creating disaster recovery mechanisms using Amazon Route 53](https://aws.amazon.com/blogs/networking-and-content-delivery/creating-disaster-recovery-mechanisms-using-amazon-route-53/) — health checks from eight regions / 3-of-16 checkers, the ARC quorum model tolerating two unavailable endpoints, and the **STOP** (Standby Takes Over Primary) pattern with no dependency on the primary region.
- [Set the JVM TTL for DNS name lookups (AWS SDK for Java)](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/jvm-ttl-dns.html) — "the JVM default TTL is set so that it will *never* refresh DNS entries until the JVM is restarted", the security-property-not-system-property warning, the `java.security` comment block ("any negative value: caching forever"), and the 5-second recommendation.
- [Moura, Heidemann, Schmidt, Hardaker — "Cache Me If You Can: Effects of DNS Time-to-Live", ACM IMC 2019](https://ant.isi.edu/~johnh/PAPERS/Moura19b.html) — peer-reviewed measurement of how real resolver cache lifetimes diverge from configured TTLs. Use this instead of anecdote when arguing that TTLs are advisory.
- [AWS Fault Isolation Boundaries — Appendix B](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/appendix-b---edge-network-global-service-guidance.html) — control-plane home regions: Route 53 and CloudFront in `us-east-1`, Global Accelerator in `us-west-2`.
- [`aws_route53recoverycontrolconfig_routing_control` (Terraform Registry)](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route53recoverycontrolconfig_routing_control) — resource surface for the ARC config plane.
- [`aws_route53_health_check` (Terraform Registry)](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route53_health_check) — `routing_control_arn` for `RECOVERY_CONTROL` type, `insufficient_data_health_status`, `request_interval`.
- [Amazon Application Recovery Controller Region switch now supports three new capabilities](https://aws.amazon.com/about-aws/whats-new/2025/12/amazon-application-recovery-controller-region-switch-new-capabilities) — post-recovery workflows, native RDS execution blocks, and **AWS provider for Terraform support** for Region switch.

## Related notes

[[aws-acm]] · [[aws-alb-nlb]] · [[aws-cloudfront]] · [[aws-rds-postgres]] · [[aws-eks]] · [[failover-runbooks]] · [[failover-orchestration]] · [[cost-modelling]] · [[region-pair-selection]] · [[data-residency-compliance]] · [[cloudwatch-observability]] · [[terraform-repo-structure]] · [[aws-iam]] · [[aws-waf-shield]]
