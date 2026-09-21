---
title: Amazon Application Recovery Controller (ARC) — evaluated for three active/passive pairs
service: arc
tags: [operations, arc, failover, routing-control, readiness-check, zonal-shift, cost]
status: partial
replication: N/A — ARC is a control/decision layer, it replicates nothing
rpo_achievable: N/A — ARC moves traffic, it does not move data
rto_achievable: "routing-control flip: 5–30 s, plus DNS TTL. Does not shorten the other ~13 minutes of the budget."
meets_targets: conditional — ARC helps the traffic-shift step only, and that step was never the constraint
updated: 2026-09-21
---

# Amazon Application Recovery Controller (ARC)

> **Scope.** [[aws-route53]] covers DNS mechanics and where the RTO is won.
> [[failover-orchestration]] covers the ordered sequence and who presses the
> button. [[split-brain-and-fencing]] covers stopping the old primary writing.
> **This note is the product evaluation of ARC itself** — what each of its four
> components actually does, what it really costs for *this* estate, and a
> committed adopt/don't-adopt verdict. Read those three first; this one does not
> repeat them.

## TL;DR

- **ARC is four largely unrelated products sold under one name.** Readiness
  checks, routing controls, Region switch, and zonal shift/autoshift share a
  console and a pricing page and almost nothing else. Evaluating "ARC" as a unit
  is the first mistake. Evaluate the four separately; the answers differ wildly.
- **Readiness checks — the "is our warm standby actually warm?" product — are
  closed to new customers.** AWS, verbatim: *"The readiness check feature in
  Amazon Application Recovery Controller (ARC) is no longer open to new
  customers."* Unless this company is already an ARC readiness customer, the
  feature that most directly addresses this vault's central anxiety **cannot be
  bought**. AWS's own migration advice is to use Region switch's *plan
  evaluation* instead. And even if you could buy it: **there is no EKS readiness
  rule**, which for an EKS estate guts most of its value.
- **Routing controls are excellent engineering at an indefensible price for this
  estate.** $2.50/hour/cluster = **$1,825/month, $21,900/year**, and that is
  *per AWS account*, not per estate — if the three pairs live in three accounts
  it is **$5,475/month**. What you buy is a five-region quorum data plane for a
  boolean. [[aws-route53]] already documents a $0.75/month alternative (the STOP
  pattern) with the same control-plane-free property and no safety rules.
- **Safety rules are the one genuinely non-reproducible thing ARC sells**, and
  [[split-brain-and-fencing]] independently reached for them. A gating rule makes
  "never shift traffic before the database is promoted" a structural invariant
  rather than a line in a runbook. **But you cannot buy safety rules without
  buying the $1,825/month cluster**, and the write-lease fence in
  [[split-brain-and-fencing]] solves the harder half of the same problem for the
  cost of a DynamoDB table.
- **Verdict: adopt zonal autoshift (free) and ARC Region switch ($70/plan/month,
  $210 for three pairs). Do not buy a routing-control cluster.** The $1,825/month
  buys resilience for a 90-second step in a 900-second budget, while the
  remaining 810 seconds — database promotion, EKS scale-out, consumer enablement
  — are exactly what Region switch orchestrates at 4% of the price. Full
  reasoning and the cost table in [[#The verdict]].

## The four components, honestly separated

AWS markets ARC as one service. It is four, with four different maturity levels,
four different data planes, four different region-availability stories and a
~26,000× price spread between the cheapest and the dearest.

| Component | What it actually is | Data plane | Price | Available to a new customer in 2026? |
|---|---|---|---|---|
| **Readiness check** | A config-diff engine that compares your standby's resources against your primary's, once a minute | Not HA — AWS says so explicitly | $0.045/hr/check ≈ **$32.85/mo** | **No — closed to new customers** |
| **Routing control** | A boolean, hosted on a five-region quorum cluster, surfaced to Route 53 as a health check | Five regions, quorum | **$2.50/hr/cluster ≈ $1,825/mo** | Yes |
| **Safety rules** | Transactional constraints on routing-control state changes | Same cluster | $0 — but requires a cluster | Yes, if you buy a cluster |
| **Region switch** | A managed, versioned, testable failover *runbook* (execution blocks, graceful/ungraceful modes, approval steps) | One in every Region | **$70/plan/mo** | Yes |
| **Zonal shift / autoshift** | Drain one AZ out of an ALB/NLB/ASG/EKS | Regional | **$0** | Yes |

Sources for every number: [ARC pricing](https://aws.amazon.com/application-recovery-controller/pricing/)
and the [readiness availability change notice](https://docs.aws.amazon.com/r53recovery/latest/dg/arc-readiness-availability-change.html).

> [!important] The single most useful sentence on the ARC docs site
> From the availability-change page, verbatim:
> *"ARC and ARC Region switch continue to be fully supported. Only the readiness
> check feature is affected by this change. There are no changes to Region
> switch, routing controls, zonal shift, and zonal autoshift."*
>
> Read the omission: AWS is steering new customers away from the component that
> answers "is my standby ready?" and towards the component that answers "run my
> failover for me". That is a product-strategy signal and it happens to point at
> the right answer for this estate.

## Component 1 — Readiness checks

### What it actually does

Not "health checks for your standby". A **continuous configuration and capacity
differ** between replicas. Verbatim from the
[docs](https://docs.aws.amazon.com/r53recovery/latest/dg/readiness-what-is.html):

> A readiness check in ARC continually (at one-minute intervals) audits for
> mismatches in AWS provisioned capacity, service quotas, throttle limits, and
> configuration and version discrepancies for the resources included in the
> check.

And on capacity specifically:

> To be prepared for recovery, you must maintain sufficient spare capacity in
> replicas at all times, to absorb failover traffic from another Availability
> Zone or Region. ARC continually (once a minute) inspects your application to
> ensure that your provisioned capacity matches across all Availability Zones or
> Regions. The capacity that ARC inspects includes, for example, Amazon EC2
> instance counts, Aurora read and write capacity units, and Amazon EBS volume
> size. If you scale up the capacity in your primary replica for resource values
> but forget to also increase the corresponding values in your standby replica,
> ARC detects the mismatch so that you can increase the values in the standby.

The quota behaviour is the part people miss and it is genuinely clever:

> For quotas, when ARC detects a mismatch with a readiness check, it can take
> steps to align the quotas for the replicas by increasing the lower quota to
> match the higher quota. When the quotas match, the readiness check status shows
> `READY`. (Note that this isn't an immediate update process, and the total time
> depends on the specific resource type and other factors.)

**ARC will raise your standby region's service quotas to match the primary's, by
itself.** That is a real problem solved — silent per-region quota divergence is a
classic warm-standby killer and is called out as a named gotcha across this
vault. Nothing else on the market does it.

### The model: recovery groups, cells, resource sets, readiness scopes

Four nouns, verbatim from the
[components page](https://docs.aws.amazon.com/r53recovery/latest/dg/introduction-components-readiness.html):

- **Cell** — *"defines your application's replicas or independent units of
  failover... cells typically represent an Availability Zone or a Region."*
- **Recovery group** — *"represents an application or group of applications that
  you want to check failover readiness for. It consists of two or more cells."*
- **Resource set** — *"a set of resources, including AWS resources or DNS target
  resources, that span multiple cells."* Same type of resource, one per cell.
- **Readiness scope** — *"identifies the grouping of resources that a specific
  readiness check encompasses... can be a recovery group (that is, global to the
  whole application) or a cell."*
- **Readiness rule** — *"audits that ARC performs against a set of resources in a
  resource set. ARC has a set of readiness rules for each type of resource."*

Mapped onto this estate: one recovery group per pair (EU, US, CA), two cells per
group (primary region, standby region), one resource set per resource *type* per
pair, and a readiness check per resource set. That is **not** one readiness
check per pair — it is one per resource type per pair, which is where the cost
compounds. See [[#Component 1 cost, if you could buy it]].

### What "readiness" really measures — and the honest limits

The rules are published in full at
[Readiness rules descriptions](https://docs.aws.amazon.com/r53recovery/latest/dg/recovery-readiness.rules-resources.html).
They fall into three families:

1. **"Same as"** — `LambdaMemorySize`, `SqsQueueVisibilityTimeout`,
   `VpcCidrBlock`, `ElbV2IdleTimeoutSeconds`. Pure config parity. These are
   ~80% of the rule set.
2. **"Within N%"** — `AsgNormalizedCapacity` and `RdsNormalizedCapacity`
   (*"within 15% of the maximum in the resource set"*), `DynamoCapacity` and
   `DynamoGsiCapacity` (*"within 20% of the maximum capacities"*),
   `ElbV2ProvisionedCapacityLcuCount` (*"within 20% of the highest provisioned
   LCU"*). These are the capacity ones and the reason the product exists.
3. **"Is healthy right now"** — `DynamoTableStatus` (`ACTIVE`), `ElbV2State`
   (`ACTIVE`), `RdsClusterStatus` (`AVAILABLE`/`BACKING-UP`),
   `CloudWatchAlarmState` (not `ALARM` or `INSUFFICIENT_DATA`),
   `ElbV2TargetGroupsCanServeTraffic` (*"at least one healthy Amazon EC2
   instance"*).

Two rules are worth singling out because they encode advice this vault gives
independently:

- **`RdsGlobalReplicaLag`** — *"Inspects each Aurora cluster to ensure that it
  has a `Global Replica Lag` of less than 30 seconds."* A hard-coded 30 seconds.
  Our RPO is **2 hours**. So this rule will mark a perfectly RPO-compliant
  Aurora standby `NOT READY`, permanently, and there is no threshold to tune.
  See [[aws-aurora-global-database]].
- **`DnsTargetResourceRecordSetConfigurationRule`** — *"ensure that they have the
  same resource record cache time to live (TTL) and that the TTLs are less than
  or equal to 300."* AWS enforcing the low-TTL advice from
  [[aws-route53#TTL strategy]] as a machine-checkable rule. Useful.

> [!danger] The three limits that matter, in AWS's own words
> All verbatim from the same page:
>
> > **Readiness checks shouldn't be used to indicate whether your production
> > replica is healthy, nor should you rely on readiness checks as a primary
> > trigger for failover during a disaster event.**
>
> > **ARC readiness checks are not highly available, so you should not depend on
> > the checks being accessible during an outage.** In addition, the resources
> > that are checked might also not be available during a disaster event.
>
> > Although readiness checks ensure that your configured capacities across
> > replicas are consistent, **you should not expect them to decide on your
> > behalf what the capacity of your replica should be.**
>
> Translated: it does not test your application, it is not available when you
> need it, and it tells you the two sides *match*, not that either side *works*.
> A standby that is an exact mirror of a primary running a broken build is
> `READY`.

### What is not in the supported resource list

The full supported set is: API Gateway v1 and v2 stages, Aurora clusters, Auto
Scaling groups, CloudWatch alarms, customer gateways, DNS target resources,
DynamoDB tables, Classic Load Balancers, EBS volumes, Lambda functions,
ALB/NLB, MSK clusters, Route 53 health checks, SNS subscriptions, SNS topics,
SQS queues, VPCs, Site-to-Site VPN connections and gateways.

> [!warning] There is no EKS readiness rule. There is no RDS-for-PostgreSQL rule.
> This estate's two hardest failover steps — [[aws-eks]] node-group scale-out
> and [[aws-rds-postgres]] replica promotion — are **both invisible to readiness
> checks**. `RdsClusterStatus` and friends are *Aurora cluster* rules; a plain
> RDS Postgres cross-region read replica is not a supported resource type. And
> EKS appears in ARC only under *zonal* shift, not readiness.
>
> The Auto Scaling group rules would cover an EKS **managed node group's**
> underlying ASG if you pointed a resource set at the ASG ARN directly — but that
> is an undocumented use of the resource type, it would not see Karpenter at all,
> and `AsgMinSizeAndMaxSize` comparing a scaled-down warm standby against a
> full-size primary would sit permanently `NOT READY`. See
> [[aws-eks]] and [[eks-workload-delivery]].
>
> **So even in the counterfactual where readiness checks were purchasable, they
> would audit the easy half of this estate and ignore the hard half.**

### Component 1 cost, if you could buy it

$0.045/hour/check = **$32.85/month/check**
([pricing page](https://aws.amazon.com/application-recovery-controller/pricing/),
whose own worked example is *"Two readiness checks (one for Auto Scaling Groups,
one for DynamoDB tables) = $0.09 per hour"*).

One check per resource *type* per pair. A realistic set for one pair — ALB,
DynamoDB, Lambda, SQS, SNS topics, VPC, CloudWatch alarms, Route 53 health
checks — is 8 checks:

| Scope | Checks | Monthly |
|---|---|---|
| One pair, 8 resource types | 8 | $262.80 |
| **Three pairs** | **24** | **$788.40** |

**$9,460/year to be told your Lambda memory sizes differ**, for a feature you
cannot buy, that does not cover EKS or RDS Postgres, and that AWS says is not
available during the outage it exists for. This is the clearest "no" in the note.

### What to do instead

AWS's own answer, verbatim from the availability-change page:

> For capabilities similar to readiness check, we recommend onboarding your
> multi-Region application to ARC Region switch. ARC Region switch is a fully
> managed service that provides complete multi-Region recovery orchestration. It
> includes a capability called **plan evaluation**, which regularly monitors the
> state of your Region switch plan to ensure readiness for execution.

Plus the things this estate should be doing regardless, none of which cost $789
a month:

| Readiness concern | Replacement |
|---|---|
| Config parity between regions | **Terraform itself.** The same module, instantiated twice with different provider aliases, *is* a parity guarantee — see [[module-patterns]] and [[provider-aliases-vs-separate-stacks]]. A `terraform plan` that is not empty in the standby is a readiness failure. Run it on a schedule as a drift check. |
| Service quota divergence | Service Quotas + [[aws-config]]-style conformance rules, or a scheduled Lambda that diffs `service-quotas list-service-quotas` between the pair and alarms on mismatch. ~50 lines. This is the one real gap. |
| Capacity headroom in the standby | A synthetic load test during [[dr-testing-and-gamedays]], plus On-Demand Capacity Reservations. Readiness checks compare *configured* capacity, which is not the same as *obtainable* capacity — see the Region switch capacity caveat in [[failover-orchestration]]. |
| "Does the standby actually work?" | A canary running a **real write transaction** against the standby from a third region. This is the thing readiness checks explicitly do not do, and it is the thing you actually want. [[dr-testing-and-gamedays#Continuous verification]]. |

## Component 2 — Routing controls

### What it actually is

A boolean. That is not a criticism — the engineering is in *where the boolean
lives*, not in the boolean.

Verbatim from the
[components page](https://docs.aws.amazon.com/r53recovery/latest/dg/introduction-components-routing.html):

> **Cluster** — A cluster is a set of five redundant Regional endpoints against
> which you initiate API calls to update or get routing control states. A cluster
> includes a default control panel, and you can host multiple control panels and
> routing controls on one cluster.

> **Routing controls** — A routing control is a simple on/off switch, hosted on a
> cluster, that you use to control routing of client traffic in and out of cells.

> **Routing control health check** — Routing controls are integrated with health
> checks in Route 53. The health checks are associated with DNS records that
> front each application replica, for example, failover records. When you change
> routing control states, ARC updates the corresponding health checks, which
> redirect traffic.

> **Endpoint (cluster endpoint)** — Each cluster in ARC has five Regional
> endpoints that you can use for setting and retrieving routing control states.
> **Your process for accessing the endpoints should assume that ARC regularly
> brings the endpoints up and down for maintenance, so you should try each
> endpoint in succession until you connect to one.**

That last sentence is the operationally important one and it is easy to skim
past. **A given cluster endpoint being down is normal, expected, routine
maintenance — not a signal of anything.** Any tool that calls one endpoint and
treats failure as "ARC is broken" is wrong by construction. Retry across all
five, always, even in a drill.

### The five-region claim — verified

The claim from [[aws-route53]] is that a cluster is a data plane spread across
five AWS Regions. **Confirmed, and the five regions are named.** Verbatim from
the [AWS General Reference — ARC endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/arc.html):

> ARC creates endpoints for each cluster in the following five Regions: US East
> (N. Virginia) (`us-east-1`), Europe (Ireland) (`eu-west-1`), US West (Oregon)
> (`us-west-2`), Asia Pacific (Tokyo) (`ap-northeast-1`), and Asia Pacific
> (Sydney) (`ap-southeast-2`). Routing Controls provide five regional endpoints
> to ensure high availability, even in the face of failures. To achieve their
> full resilience, it's important to have retry logic that can use all five
> endpoints as necessary.

The same page gives the endpoint *shape*, which is what you actually hard-code:

| Example endpoint | Region |
|---|---|
| `https://aaaaaaaa.route53-recovery-cluster.eu-west-1.amazonaws.com` | `eu-west-1` |
| `https://bbbbbbb.route53-recovery-cluster.ap-northeast-1.amazonaws.com` | `ap-northeast-1` |
| `https://ccccccc.route53-recovery-cluster.us-west-2.amazonaws.com` | `us-west-2` |
| `https://ddddddd.route53-recovery-cluster.us-east-1.amazonaws.com` | `us-east-1` |
| `https://eeeeeee.route53-recovery-cluster.ap-southeast-2.amazonaws.com` | `ap-southeast-2` |

The hostname prefix is per-cluster and random; the rest is fixed. The quorum
property — *"the cluster and routing controls continue to function even if up to
two Regional endpoints are unavailable"* — is quoted in [[aws-route53]] from the
[DR-mechanisms blog](https://aws.amazon.com/blogs/networking-and-content-delivery/creating-disaster-recovery-mechanisms-using-amazon-route-53/).
Five endpoints, tolerate two down, need three.

> [!danger] Three consequences of the five-region list that nobody notices
> **1. Two of the five are this estate's own regions.** `us-east-1` is the US
> pair's primary; `eu-west-1` is the EU pair's primary; `us-west-2` is the US
> pair's *standby*. So when you fail the US pair away from `us-east-1`, two of
> the five ARC endpoints you might call are in regions directly involved in the
> event. Quorum still holds (three of five remain, in Ireland, Tokyo and Sydney),
> but the retry loop must genuinely rotate — a tool that tries `us-east-1` first
> and gives up has no resilience at all.
>
> **2. There is no Canadian endpoint.** Not `ca-central-1`, not `ca-west-1`. The
> switch that fails over the Canadian deployment is operated from Virginia,
> Ireland, Oregon, Tokyo and Sydney. The *state of a boolean* is not customer
> personal data, but if the data-residency posture is "nothing about the Canadian
> deployment leaves Canada", someone in compliance needs to sign this off.
> Raised in [[#Open questions]] and relevant to [[data-residency]].
>
> **3. There is no `eu-west-2` endpoint either.** The EU pair's standby is
> London; the nearest ARC endpoint is Dublin — the region you are failing away
> from. Same quorum argument applies, same "rotate properly" requirement.

### The operational detail that is easiest to get wrong at 3am

**You must address a specific regional cluster endpoint by URL, *and separately*
set `--region` to that endpoint's region.** This is not the normal SDK pattern
and it is not discoverable. Verbatim from the
[CLI guide](https://docs.aws.amazon.com/r53recovery/latest/dg/getting-started-cli-routing.control-state.html):

> For each cluster that you create, ARC provides you with a set of cluster
> endpoints, one in each of five AWS Regions. **You must specify one of these
> Regional endpoints (the AWS Region and the endpoint URL) when you make calls to
> the cluster to retrieve or set routing control states to `On` or `Off`.** When
> you use the AWS CLI, to get or update routing control states, in addition to
> the Regional endpoint, you must also specify the `--region` of the Regional
> endpoint.

So there are **two** things to get right, not one, and they must agree:

```bash
aws route53-recovery-cluster update-routing-control-states \
    --update-routing-control-state-entries \
    '[{"RoutingControlArn": "arn:aws:route53-recovery-control::111122223333:controlpanel/0123456bbbbbbb0123456bbbbbb0123456/routingcontrol/abcdefg1234567",
       "RoutingControlState": "Off"},
      {"RoutingControlArn": "arn:aws:route53-recovery-control::111122223333:controlpanel/0123456bbbbbbb0123456bbbbbb0123456/routingcontrol/hijklmnop987654321",
       "RoutingControlState": "On"}]' \
    --region us-west-2 \
    --endpoint-url https://host-dddddd.us-west-2.example.com/v1
```

(Shape taken verbatim from the AWS CLI example; the ARNs and the endpoint host
are AWS's placeholders.) A successful response is `{}` — **empty**. An operator
who expects confirmation output will think the call failed and run it again.
Write that in the runbook.

Three separate traps live in that one command:

1. **Omit `--endpoint-url` and the CLI silently talks to the default regional
   service endpoint**, which is not your cluster. You get an error at 3am that
   reads like a permissions problem.
2. **Mismatch `--region` and the endpoint host** and SigV4 signing fails — the
   signature is scoped to the region string, not the hostname.
3. **`DescribeCluster` — the call that tells you the five endpoints — is a
   *control-plane* call**, served only from `us-west-2`. Verbatim from the
   General Reference: *"When you use the AWS CLI or SDKs to submit requests with
   ARC Recovery Readiness API (for readiness checks), Recovery Control
   Configuration API or Recovery Cluster API (for routing control), you must
   specify the AWS Region as `us-west-2`."* If your runbook starts with "run
   `describe-clusters` to get the endpoints", **your data-plane-only failover
   mechanism has a control-plane dependency in its first step**. AWS says this
   outright: *"During a failure event, you might not be able to access some API
   operations, including ARC API operations that are not hosted on the extremely
   reliable data plane cluster."*

> [!important] The rule, stated once
> **Hard-code all five endpoint URLs and all routing-control ARNs as literals in
> the failover tool. Rotate through them in random order. Treat an individual
> endpoint failure as routine. Never call `DescribeCluster` during a failover.**
>
> [[failover-orchestration]] already encodes this as a Terraform variable with a
> `length == 5` validation, which is the right shape. This note adds the *why*:
> the endpoints are only discoverable via a `us-west-2` control-plane API.

### The other data-plane operations

| API | Plane | Endpoint | Use |
|---|---|---|---|
| `ListRoutingControls` | **Data** | one of five cluster endpoints | Read current state before deciding |
| `GetRoutingControlState` | **Data** | one of five cluster endpoints | Read one control |
| `UpdateRoutingControlState` | **Data** | one of five cluster endpoints | Flip one — **do not use for failover** |
| `UpdateRoutingControlStates` | **Data** | one of five cluster endpoints | Flip several **atomically** — use this |
| `DescribeCluster`, `CreateRoutingControl`, `CreateSafetyRule`, everything `route53-recovery-control-config` | **Control** | `us-west-2` only | Build time only |

**Always use the plural `UpdateRoutingControlStates`.** Two singular calls leave
you in a half-switched state between them — both regions on, or both off — and
the second call can fail. The plural form is the transaction boundary that safety
rules are evaluated against. [[failover-orchestration]] makes the same point from
the orchestrator's side.

### Route 53 integration

A routing control is exposed to DNS as `aws_route53_health_check` with
`type = "RECOVERY_CONTROL"` and a `routing_control_arn`. Route 53 then treats
the boolean as health: `On` → healthy → records served; `Off` → unhealthy →
records withdrawn. No polling, no interval, no failure threshold, no health
checker IP allow-listing, no 18% quorum rule — **the state change propagates to
the Route 53 data plane directly**. That is why [[aws-route53]]'s Scenario B
budgets ~10 s for the answer to change versus ~35–40 s for an endpoint check.

**Roughly 25 seconds saved on a 900-second budget, for $1,825/month.** Hold that
number; it comes back in [[#The verdict]].

### Quotas

From [Quotas for routing control](https://docs.aws.amazon.com/r53recovery/latest/dg/route53-ar-quotas.html), verbatim:

| Entity | Quota |
|---|---|
| Number of clusters per account | **2** |
| Number of control panels per cluster | 50 |
| Number of routing controls per control panel | 100 |
| Total number of routing controls (in all control panels) per cluster | 300 |
| Number of safety rules per control panel | 20 |
| Number of routing controls per `UpdateRoutingControlStates` operation call | **10** |
| Number of mutating API calls to a cluster endpoint, per second | **3** |

Three of these matter here:

- **2 clusters per account.** This is what makes "one cluster serves all three
  pairs" possible — and it is also why the price is **per account**. See
  [[#The cost, without flinching]].
- **10 routing controls per batch call.** Comfortable for a pair (you need 2–4:
  primary-traffic, standby-traffic, and optionally db-promoted and a
  master-arm). It would *not* be comfortable if you modelled every microservice
  as its own routing control. **Model at the region level, not the service
  level.**
- **3 mutating calls per second per endpoint.** Fine for a failover. It rules
  out any design where routing controls are flipped programmatically at
  per-request or per-deployment frequency.

## Component 3 — Safety rules

This is the part of ARC that is genuinely hard to build yourself, and the part
[[split-brain-and-fencing]] independently reinvented. It deserves a head-to-head.

### What they are, precisely

Verbatim from the
[CreateSafetyRule API reference](https://docs.aws.amazon.com/recovery-cluster/latest/api/safetyrule.html):

> **Assertion rule**: An assertion rule enforces that, when you change a routing
> control state, that a certain criteria is met. For example, the criteria might
> be that at least one routing control state is `On` after the transaction so
> that traffic continues to flow to at least one cell for the application. **This
> ensures that you avoid a fail-open scenario.**

> **Gating rule**: A gating rule lets you configure a gating routing control as
> an overall "on/off" switch for a group of routing controls. Or, you can
> configure more complex gating scenarios, for example by configuring multiple
> gating routing controls.

And the crucial enforcement semantics, verbatim:

> An assertion rule enforces that, when you change a routing control state, that
> the criteria that you set in the rule configuration is met. **Otherwise, the
> change to the routing control is not accepted.**

**"Not accepted" — the API rejects the call.** This is not an alarm, not a
warning, not a Slack message. It is a hard constraint evaluated server-side, in
the data plane, on the transaction. That distinction is the entire value.

### The rule grammar

`RuleConfig` has exactly three fields — `Type`, `Threshold`, `Inverted` — and
three types. Verbatim:

> `ATLEAST` - At least N routing controls must be set. You specify N as the
> `Threshold` in the rule configuration.
> `AND` - All routing controls must be set. This is a shortcut for "At least N,"
> where N is the total number of controls in the rule.
> `OR` - Any control must be set. This is a shortcut for "At least N," where N
> is 1.

Plus `Inverted`: *"Logical negation of the rule. If the rule would usually
evaluate true, it's evaluated as false, and vice versa."*

So `AND` and `OR` are sugar over `ATLEAST`, and `Inverted` gives you NAND/NOR.
**That is the whole language.** It is deliberately, correctly tiny — there are no
conditionals, no references to external state, no time windows. You cannot
express "only if replica lag < 2h" as a safety rule; you can only express it by
having your orchestrator set a routing control that *represents* that fact. See
[[#Modelling non-traffic facts as routing controls]].

`WaitPeriodMs` is the fourth field and it is not part of the logic:

> An evaluation period, in milliseconds (ms), during which any request against
> the target routing controls will fail. **This helps prevent "flapping" of
> state.** The wait period is 5000 ms by default, but you can choose a custom
> value.

Note what that means operationally: **after any accepted state change, further
changes to the same targets are rejected for 5 seconds by default.** A retry loop
that fires immediately on a transient error will hit this and see a failure that
looks like the safety rule blocking it. Budget for it; do not set it to 0 to
make the errors go away, because flap-damping is the point.

### The two rules this estate should have

**Rule A — assertion: never both off.** The fail-open guard.

```
AssertedControls = [eu-primary-traffic, eu-standby-traffic]
RuleConfig       = { Type = ATLEAST, Threshold = 1, Inverted = false }
```

Any transaction that would leave both `Off` is rejected. This is the 3am
fat-finger guard and it costs nothing once you have a cluster.

> [!warning] `ATLEAST 1` does NOT prevent both being ON
> It prevents the *fail-open* (everything off) case only. For active/passive
> with an async replica, **both-on is the dangerous state** — it is split-brain
> at the traffic layer, sending writes to two databases.
>
> There is no `EXACTLY 1` rule type. The closest expression is a second
> assertion rule with `Inverted = true` over `AND`: "NOT (both on)". Combine the
> two and you have `exactly one`, at the cost of two of your 20 rules per control
> panel. **Write both rules. A single `ATLEAST 1` is a half-guard and reads like
> a whole one.**

**Rule B — gating: you may not turn the standby on until the database is
promoted.** This is the one that earns its keep.

```
GatingControls = [eu-db-promoted]
TargetControls = [eu-standby-traffic]
RuleConfig     = { Type = OR, Threshold = 1, Inverted = false }
```

From the API docs, verbatim: *"Routing controls that can only be set or unset if
the specified `RuleConfig` evaluates to true for the specified
`GatingControls`... your ability to change the routing controls that you have
specified as `TargetControls` is gated by the rule that you set for the routing
controls in `GatingControls`."*

[[failover-orchestration#Why each edge in that graph exists]] spends a page
explaining why "promote before you shift traffic" matters — read-only
transaction errors, half-committed workflows, retry storms, no free rollback. A
gating rule converts that page of prose into a server-side constraint. **The
most expensive ordering error in the runbook becomes structurally impossible.**

### Modelling non-traffic facts as routing controls

The trick that makes gating rules useful is that **a routing control does not
have to control routing**. `eu-db-promoted` above is attached to no Route 53
health check at all. It is a shared, five-region-replicated boolean that the
orchestrator sets after `promote-read-replica` returns, and whose only purpose is
to be read by a gating rule.

Candidates worth modelling this way for this estate:

| Control | Set by | Gates |
|---|---|---|
| `<pair>-db-promoted` | Orchestrator, after promotion succeeds | `<pair>-standby-traffic` |
| `<pair>-standby-scaled` | Orchestrator, after the EKS node group reaches target | `<pair>-standby-traffic` |
| `<pair>-failover-armed` | A **human**, deliberately, as a two-key step | everything |
| `<pair>-consumers-enabled` | Orchestrator, after ESMs/rules enabled | — (assert, don't gate) |

`<pair>-failover-armed` is the two-key switch and it is the cheapest safety
feature in the product: nothing can move traffic until a human has explicitly
armed the pair, and arming is itself an audited, five-region-durable action.

### Breaking the glass

Safety rules must be overridable or they become the outage. They are. Verbatim
from [Overriding safety rules to reroute traffic](https://docs.aws.amazon.com/r53recovery/latest/dg/routing-control.override-safety-rule.html):

> In a "break glass" scenario like this, you can override one or more safety
> rules to change a routing control state and fail over your application.

> You can bypass safety rules when you update a routing control state (or
> multiple routing control states) by using the `update-routing-control-state` or
> `update-routing-control-states` AWS CLI command with the
> `safety-rules-to-override` parameter.

> When a safety rule blocks a routing control state update, **the error message
> includes the ARN of the rule that blocked the update.** So you can make a note
> of the ARN, and then specify it in a routing control state CLI command with the
> safety rule override parameter.

And the trap, verbatim:

> Because more than one safety rule might be in place for the routing controls
> that you're updating, **you could run the CLI command to update your routing
> control state with one safety rule override but get an error that another
> safety rule is blocking the update. Continue to add safety rule ARNs to the
> list of rules to override in the update command, separated by commas, until the
> update command completes successfully.**

> [!danger] Read that last paragraph as an operational spec, because it is one
> **Overriding is iterative.** You discover blocking rules one at a time, by
> being blocked by them. With four safety rules on a control panel you may run
> the command five times, each time pasting one more 100-character ARN, under
> time pressure, at 3am.
>
> **Mitigations, both required:**
> 1. **Put every safety rule ARN in the runbook as a literal**, next to the five
>    cluster endpoints. They are outputs of the Terraform that creates them, so
>    render them into the runbook at plan time.
> 2. **Rehearse the override path in a game day.** Not the happy path — the
>    override path. It is the one people have never typed. [[dr-testing-and-gamedays]].
>
> Also note: this is the mechanism by which safety rules stop being safety. An
> engineer who has been blocked twice will reach for `--safety-rules-to-override`
> with *all* the ARNs by reflex. Design the rule set small enough that overriding
> is rare and deliberate.

### Head to head: ARC safety rules vs the hand-built fence

[[split-brain-and-fencing]] designs a three-layer fence — an application write
lease in a third region, opportunistic scale-to-zero, and a pre-armed
standby-side SCP deny. It then says, of ARC gating rules:

> That does not stop the old primary writing, but it makes the most expensive
> ordering error structurally impossible, and it costs a policy document.

That is exactly right, and it is the key to the comparison: **these two things
solve different problems and are not substitutes.**

| | **ARC safety rules** | **Write lease ([[split-brain-and-fencing]])** |
|---|---|---|
| Question answered | "Is it safe to *move traffic* right now?" | "Is it safe for this process to *write* right now?" |
| Enforced where | ARC data plane, at the `UpdateRoutingControlStates` transaction | Inside the application's write path, on every write |
| Stops the old primary writing? | **No.** Not even slightly. | **Yes.** The only technique in that note that does. |
| Prevents shifting traffic to an unpromoted DB? | **Yes**, structurally | No — it is not in that path |
| Works when the primary region is unreachable? | Yes (five-region data plane) | Yes (lease store in a third region, fail-closed) |
| Works when the *operator* is wrong? | Yes — server-side rejection | Yes — the writer stops itself |
| Cost | **$1,825/month** (requires a cluster) | ~$0 (a DynamoDB item + application code) |
| Build cost | Terraform, ~40 lines | Application change in every writer — **the expensive part** |
| Blast radius if misconfigured | Failover blocked; overridable | Writes blocked; an outage |

**Recommendation: build the write lease, and do not buy the cluster for the
safety rules alone.** Reasoning:

1. The write lease covers the *dangerous* failure (two writable databases,
   divergent data, GitHub-October-2018). Safety rules cover the *expensive*
   failure (traffic on a read-only replica — bad, but detectable in 60 seconds
   and recoverable by finishing the promotion).
2. Safety rules cannot be bought separately. You are paying $21,900/year for a
   constraint engine bundled with a boolean you already have three ways of
   storing.
3. The gating-rule guarantee can be reproduced — imperfectly but adequately — as
   **a precondition check inside the orchestrator**: the Region switch plan or
   SSM document simply does not execute the traffic step until the promote step
   has returned success, and `onFailure: Abort` enforces it. That is weaker
   (it is a client-side check, not a server-side one, and a human running the
   steps by hand can skip it) but it is not nothing, and it is free.
4. The honest counter-argument: **a human running the steps by hand is exactly
   the 3am scenario**, and that is precisely when the client-side check is
   absent. If your organisation genuinely expects manual, out-of-band failovers
   rather than orchestrator-driven ones, the gating rule is worth more than this
   note credits it for. **The right response to that is to fix the manual
   failover, not to buy a $1,825/month guardrail for it.**

## Component 4 — Zonal shift and zonal autoshift

> [!tip] The explicit callout the brief asked for
> **This is the part of ARC that is free, and it is very likely the part that
> delivers the most resilience per pound in the entire programme.** It has
> nothing to do with multi-region. It is not a competitor to the rest of ARC. It
> should be adopted regardless of every other decision in this note, and it
> should probably be adopted *first*, because it is cheap, reversible, and
> exercises the same "can we run degraded?" muscle that the region work needs.

### Zonal shift

Verbatim from the [docs](https://docs.aws.amazon.com/r53recovery/latest/dg/arc-zonal-shift.html):

> Amazon Application Recovery Controller (ARC) *zonal shift* allows you to shift
> traffic for a supported resource away from an impaired Availability Zone (AZ)
> in an AWS Region to healthy AZs in the same Region.

> All zonal shifts are temporary mitigations. **You set an initial expiration
> when you start a zonal shift, from one minute up to three days (72 hours)**,
> which you can extend, if you need to continue the traffic shift.

> Before you start a zonal shift, **you must prescale your application and ensure
> that you have sufficient capacity** to shift traffic away from an Availability
> Zone.

The expiry is a good design: a zonal shift cannot be forgotten. It either gets
extended deliberately or it unwinds itself.

**Supported resources**, verbatim from
[Supported resources](https://docs.aws.amazon.com/r53recovery/latest/dg/arc-zonal-shift.resource-types.html):

> + Amazon EC2 Auto Scaling groups
> + Amazon Elastic Kubernetes Service
> + Application Load Balancers with cross-zone load balancing enabled or disabled
> + Network Load Balancers with cross-zone load balancing enabled or disabled

**ALB, NLB, ASG and EKS.** That is this estate's entire compute and ingress
surface ([[aws-alb-nlb]], [[aws-eks]]). Unlike readiness checks, zonal shift
covers the parts of the estate that matter.

Two conditions that will catch people, verbatim:

> When a Network Load Balancer or Application Load Balancer is **in a fail open
> state, a zonal shift will have no effect**. This is expected behavior because
> zonal shift cannot force an AZ to be unhealthy and then shift traffic to the
> other AZs in a Region when a load balancer is failing open.

> If multiple load balancers are forwarding traffic to the same targets, **a
> zonal shift on a cross-zone enabled load balancer drops target capacity for all
> load balancers**, even if their traffic is not shifted by a zonal shift.

The first is the zonal analogue of [[aws-route53]]'s gotcha #4: when everything
looks unhealthy, the load balancer serves everything anyway, and your mitigation
silently does nothing. The second is a real trap in a shared-target-group EKS
setup — shifting one ALB away from an AZ removes that AZ's pods from *every*
ALB pointing at them.

### Zonal autoshift

The same mechanism, but AWS pulls the trigger. Verbatim:

> With zonal autoshift, you authorize AWS to shift away resource traffic for an
> application from an Availability Zone (AZ) during events, on your behalf, to
> help reduce time to recovery. **AWS starts an autoshift when internal telemetry
> indicates that there is an Availability Zone impairment that could potentially
> impact customers.**

> **Be aware that ARC does not inspect the health of individual resources.** AWS
> starts an autoshift when AWS telemetry detects that there is an Availability
> Zone impairment that could potentially impact customers. **In some cases,
> traffic might be shifted away for resources that are not experiencing impact.**

That is an unusually honest paragraph and it is the whole trade: **you are
buying AWS's internal AZ telemetry, which is better than anything you can
build, at the price of occasional shifts you did not need.** For an AZ-level
mitigation that is a good trade — losing one of three AZs costs you capacity,
not correctness. It is precisely the opposite of the regional trade, where a
false positive costs an irreversible database promotion
([[failover-orchestration#Branch A — automated]]).

**Practice runs are mandatory**, and this is the underrated feature:

> **Practice runs are required for zonal autoshift.** The zonal shifts that ARC
> starts for practice runs help you to ensure that shifting away traffic from an
> Availability Zone during an autoshift is safe for your application. **Practice
> runs take place weekly**, and provide an outcome—such as `SUCCEEDED` or
> `FAILED`—to help you understand if the application operates as expected.

> With a practice run, traffic is shifted away from an Availability Zone for a
> single resource for **about 30 minutes**, and then shifted back to all
> Availability Zones in the Region.

**A free, AWS-operated, weekly, automatically-evaluated chaos experiment against
production, with a pass/fail outcome.** For an organisation that does not yet
run game days, this is the cheapest possible on-ramp — see
[[dr-testing-and-gamedays]]. The weekly `SUCCEEDED`/`FAILED` signal is also an
auditor artefact that costs nothing to produce.

And the prerequisite, stated as strongly as AWS ever states anything:

> Before you configure practice runs or enable zonal autoshift, **we strongly
> recommend that you pre-scale your application resource capacity in all
> Availability Zones**... **You should not rely on scaling on demand when an
> autoshift or practice run starts. Zonal autoshift, including practice runs,
> works independently, and does not wait for auto scaling actions to complete.**

> If you use auto scaling to handle regular cycles of traffic, we strongly
> recommend that you configure the **minimum capacity** of your auto scaling to
> continue operating normally with the loss of an Availability Zone.

**This is a real cost, and it is the only cost.** "Run at N+1 AZ capacity at all
times" is a standing ~50% compute uplift in a 3-AZ region if you were previously
sized with no AZ headroom. Most mature estates already do this; if this one does
not, enabling autoshift is a capacity decision disguised as a free feature.
Take it to [[cost-model]] alongside the standby figures.

### Why this is the best value in the product

| | Zonal autoshift | Routing-control cluster |
|---|---|---|
| Failure class covered | **Single-AZ impairment** | Full-region impairment |
| Relative frequency | Far more common | Rare |
| Cost | **$0** | $1,825/month |
| Data loss on trigger | **None** — same database, same region | Up to 2 hours (RPO) |
| Reversible | **Yes, automatically** | Only by failing back |
| Rehearsed | **Weekly, by AWS, for free** | Only if you build game days |
| Human required | **No** | Yes, and correctly so |

AZ impairments are the failure this estate is statistically most likely to meet,
and they are the one where automation is unambiguously correct because nothing
is irreversible. See [[aws-regional-outages]] for the base rates this vault has
collected.

> [!note] Region availability — zonal shift is everywhere, including Calgary
> The [endpoints page](https://docs.aws.amazon.com/general/latest/gr/arc.html)
> states, verbatim: *"Zonal shift in ARC is available in all AWS Regions,
> including the Beijing and Ningxia Regions and AWS GovCloud (US)."* The regional
> endpoint table explicitly lists `arc-zonal-shift.ca-west-1.amazonaws.com`,
> `arc-zonal-shift.ca-central-1.amazonaws.com`, `arc-zonal-shift.eu-west-1...`,
> `arc-zonal-shift.eu-west-2...`, `arc-zonal-shift.us-east-1...` and
> `arc-zonal-shift.us-west-2...`. **All six of this estate's regions are
> covered.** Note `ca-west-1` has three AZs, so zonal shift is meaningful there.

## Still to research

Cut off by a session limit. The four components are analysed and **the verdict in
the TL;DR is supported by the pricing and availability research above** — that
much is safe to act on. What is missing is the implementation detail behind it.

- **`## Component 5 — ARC Region switch`.** The verdict recommends it at
  $70/plan/month and it is the one component *not* written up. It surfaced during
  research as AWS's own replacement for the closed-to-new-customers readiness
  checks, so it carries the recommendation and needs its own section: plan
  structure, plan evaluation, execution blocks, what it automates vs what stays
  manual, and how it relates to [[failover-orchestration]].
- `## The verdict against the alternatives` — the head-to-head this note was
  commissioned to settle. [[aws-route53]] health checks and the $0.75/month STOP
  pattern, [[aws-global-accelerator]] traffic dials, and [[aws-cloudfront]] origin
  groups. The TL;DR states the conclusion; the comparison table behind it is not
  written.
- `## Terraform implementation` — and specifically **whether the `hashicorp/aws`
  provider covers ARC well**. Provider gaps are a real adoption cost and were
  never checked.
- `## DR testing and game days` — flipping routing controls as a rehearsal
  mechanism. Cross-ref [[dr-testing-and-gamedays]].
- Template sections not written: Migration path, Failback, Gotchas, Decisions to
  make, a consolidated Cost table, Open questions, and **Sources** (22 inline
  links exist but were never collected).

**Verify before acting on the headline claims**, since both are load-bearing and
each rests on a single AWS page: (1) that readiness checks are closed to new
customers, and (2) the $2.50/hour/cluster routing-control price and whether it is
billed per account or per estate — the recommendation turns on that distinction.
