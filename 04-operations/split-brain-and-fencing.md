---
title: Split-Brain and Fencing — and why failback is harder than failover
tags: [operations, split-brain, fencing, failback, rds, aurora, dynamodb, iam, arc]
status: researched
replication: N/A — this is a process/design note, not a service note
rpo_achievable: N/A
rto_achievable: "Fencing costs 60–120 s of the 15-minute budget if pre-armed. If it requires reaching the primary region's control plane, it costs either 0 s or ∞."
meets_targets: conditional — a pre-armed fence fits the budget, an improvised one does not
updated: 2026-09-17
---

# Split-Brain and Fencing

> This note is step 1 of the sequence in [[failover-orchestration]] — *fence the
> primary* — expanded to the size the problem deserves, plus the thing that
> happens after the incident and that nobody writes down: **failback**.
> Read [[failover-orchestration]] first for the ordered sequence, the time
> budget and the orchestrator placement. This note does not repeat any of it.

## TL;DR

- **You cannot fence a region you cannot reach.** Every fencing technique that
  works by *doing something to the primary* — revoking IAM, locking down
  security groups, scaling to zero, deleting DNS records — is an API call
  against the primary region's control plane, and the case where you need the
  fence most is exactly the case where that control plane is unavailable. This
  is not a detail. It invalidates four of the five techniques people reach for.
- **The only fence that survives an unreachable primary is one that was already
  armed before the incident and that fires on the *absence* of a signal.** That
  is an **application-level write lease**: the primary's write path holds a
  short lease, renews it against a store outside its own region, and stops
  writing when it cannot renew. It self-fences during a partition, with no
  call to anything. **This is the recommendation.** Everything else is a
  best-effort second layer for the much more common "region is sick but
  reachable" case.
- **A promoted RDS read replica cannot be un-promoted.** There is no
  `demote-db-instance`. That turns the failover decision into a one-way door
  and means the correct bias is *wait longer*, because five more minutes of
  outage is cheaper than a wrong promote. See
  [[aws-rds-postgres#Promotion is irreversible — and the consequences are larger than they look]].
- **DynamoDB Global Tables have the opposite problem.** They are multi-active
  by design — the standby is always writable, last-writer-wins is item-level
  and silent, and a stray write in the standby replicates *back* and destroys
  the real record with no error and no metric. There is nothing to promote and
  nothing to fence unless you build the fence yourself. See [[aws-dynamodb]].
- **Failback is the larger project and it is nearly always undocumented.** For
  RDS Postgres a round trip costs **two full cross-region reseeds and a second
  scheduled outage**. Recommendation: **permanent swap, not ping-pong** — after
  an unplanned failover, keep serving from the new region and build the fresh
  standby back in the old one. For Aurora, and only for Aurora, ping-pong via
  `switchover-global-cluster` is cheap enough to be the default.

## The scenario that actually matters

The easy case is not interesting. If `eu-west-1` is genuinely, comprehensively
gone — packets black-holed, control plane returning errors, AWS Health Dashboard
lit up — then promoting `eu-west-2` is uncontroversial. There is no second
writer because there is no first writer.

**The case that matters is the grey one.** Pick any of these:

- A network partition between your monitoring and `eu-west-1`. Your probes fail.
  `eu-west-1` is fine and is still serving customers who reach it by a different
  path.
- The ALB in `eu-west-1` is unhealthy but the EKS pods behind it are up and
  still draining a queue, writing to the database.
- Route 53 health checkers see failures from six of eight checker regions.
  Your database is completely healthy.
- The primary region's *control plane* is impaired but its *data plane* is not —
  which is the single most common shape of an AWS regional event, and is exactly
  what [static stability](https://aws.amazon.com/builders-library/static-stability-using-availability-zones/)
  is designed to survive. Verbatim:

  > An impairment of the Amazon EC2 control plane means that during the event the
  > physical server might not see updates like a new EC2 instance added to a VPC,
  > **or a new Security Group rule**.

  Read that last clause again, because it is the whole of this note in one
  sentence: **during exactly the event where you want to fence the primary with
  a security group change, the security group change may not take effect.**

In all four, the primary is *not dead*. It is *unreachable, or believed
unhealthy*. Promote the standby and you have two writable databases. The
divergence starts immediately and grows at production write rate.

The canonical public example is **GitHub, 21–22 October 2018**
([post-incident analysis](https://github.blog/news-insights/company-news/oct21-post-incident-analysis/)),
already quoted in full in [[failover-orchestration#Branch A — automated]]. The
one line that belongs here rather than there:

> The database servers in the US East Coast data center contained a brief period
> of writes that had not been replicated to the US West Coast facility. Because
> the database clusters in both data centers now contained writes that were not
> present in the other data center, **we were unable to fail the primary back
> over to the US East Coast data center safely.**

**43 seconds of partition. 24 hours and 11 minutes of degradation.** Not because
the failover failed — because it *succeeded* against a primary that was not
actually dead, and the divergence made failback impossible.

## What "fencing" means, precisely

Borrowed from cluster HA, where it is sometimes STONITH — "shoot the other node
in the head". The distributed-systems literature is more careful about it than
most cloud runbooks are, and the careful version is the one you need.

Martin Kleppmann's [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
is the reference, and the key passages are:

> You need to include a fencing token with every write request to the storage
> service. In this context, a fencing token is simply a number that increases
> (e.g. incremented by the lock service) every time a client acquires the lock.

> The storage server remembers that it has already processed a write with a
> higher token number (34), and so it rejects the request with token 33.

> **Note this requires the storage server to take an active role in checking
> tokens, and rejecting any writes on which the token has gone backwards.**

That last sentence is the bar, and it is the bar almost nothing in an AWS DR
runbook clears. **Real fencing is enforced at the protected resource, by the
resource, against a monotonically increasing epoch.** Anything else — revoking
credentials, changing firewall rules, telling the old primary to stop — is
*advisory*, because it requires the thing you are fencing to cooperate, or
requires a control plane you may not be able to reach.

Two corollaries that shape everything below:

1. **"Host unreachable" is not evidence of successful fencing.** It is evidence
   that you cannot tell. An API call that times out has told you nothing about
   whether the old primary is writing.
2. **A fence you execute during the incident is strictly worse than a fence you
   armed before it.** The pre-armed fence has no runtime dependency on anything.

Neither RDS for PostgreSQL nor DynamoDB has any concept of a fencing token.
Aurora has one thing that is close to real fencing — its storage-layer write
fencing — and AWS is explicit that it is best-effort (see below). So for this
estate, **provable fencing is not available from the platform and must be built
in the application, or not at all.**

## The seven techniques, assessed against the case that matters

The assessment column that matters is the last one: **does it work when you
cannot reach the primary region's control plane?**

### 1. Revoke IAM permissions (deny policy / SCP)

The idea: attach a `Deny` on `rds:*`, `dynamodb:Put*`, or the application role
itself, scoped with `aws:RequestedRegion` to the primary region. The app can no
longer authenticate or authorise, so it stops writing.

**How it really behaves.** You have to separate two things that get conflated:

- **IAM authorisation (the data plane) is regionalised and highly available.**
  From the [AWS Fault Isolation Boundaries whitepaper](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/global-services.html),
  verbatim: *"In the `aws` partition, the IAM service's control plane is in the
  `us-east-1` Region, **with isolated data planes in each Region of the
  partition.**"* So a policy that is **already in place** is enforced locally,
  in the impaired region, with no cross-region dependency. That is good news and
  it is the basis of recommendation 3 below.
- **Changing an IAM policy is a `us-east-1` control-plane operation, and it is
  eventually consistent.** From the [IAM troubleshooting docs](https://docs.aws.amazon.com/IAM/latest/UserGuide/troubleshoot.html),
  verbatim:

  > IAM uses a distributed computing model called eventual consistency. Any
  > changes that you make in IAM (or other AWS services) ... take time to become
  > visible from all possible endpoints. Some delay results from the time it
  > takes to send data from server to server, replication zone to replication
  > zone, and Region to Region. **IAM also uses caching to improve performance,
  > but in some cases this can add time.**

  > **We recommend that you do not include such IAM changes in the critical,
  > high availability code paths of your application.**

And the same whitepaper lists, in its own summary of anti-patterns, verbatim:

> **Creating or updating IAM resources, including IAM roles and policies, during
> a failover. This typically isn't intentional, but might be a result of an
> untested failover plan.**

**Verdict: as a fence you *apply at failover time*, this is an AWS-documented
anti-pattern and it fails the test.** It depends on `us-east-1` (which for the
US pair is the region you are failing away from), it is eventually consistent
with no published bound, and IAM caching means "the API returned 200" does not
mean "the deny is live". The IAM session-revocation trick
(`AWSRevokeOlderSessions` with `aws:TokenIssueTime`) has exactly the same
problem — it is an inline-policy write to `us-east-1`.

**But as a fence you *pre-provision*, it is excellent**, because enforcement is
local to the region. That inversion is the single most useful idea in this
section, and [[aws-dynamodb#Blocking writes to the standby with IAM]] already
derives the pattern independently: a standing SCP that denies writes in the
standby region, with a pre-created, heavily-audited `FailoverBreakGlassRole`
excluded from the deny. Failing over does not edit a policy; it starts pods that
already assume the excluded role. Zero control-plane calls, zero propagation
wait.

> [!warning] Do not point this at the primary
> The SCP pattern fences the **standby** (stops stray writes while the primary
> is healthy). It cannot fence the **primary**, because you would have to edit
> the SCP at failover time to add the primary region — which is the anti-pattern
> above, and which also requires the AWS Organizations control plane, also in
> `us-east-1`.

### 2. Security group lockdown

The idea: strip the ingress rules from the database's security group, or from
the application nodes', so writes cannot reach the database.

Attractive because it is fast, scriptable, and does not touch the database.

**Verdict: works well in the "reachable but sick" case, fails in the case that
matters, and fails in a particularly nasty way.** Three reasons:

1. `ModifyDBInstance` / `AuthorizeSecurityGroupIngress` / `RevokeSecurityGroupIngress`
   are **EC2 control-plane calls in the primary region**. If the primary
   region's control plane is what is impaired, they error or hang.
2. Even when the API succeeds, the rule has to propagate to every host. The
   static-stability quote above says plainly that during an EC2 control-plane
   impairment a physical server *"might not see updates like ... a new Security
   Group rule."* So a 200 response is not a fence.
3. **Security groups are stateful.** Revoking an ingress rule does not tear down
   established connections in all cases — and your application's connection pool
   is holding dozens of long-lived, already-established TCP sessions to the
   database. New connections are blocked; the in-flight ones that are actively
   writing are the ones you care about. You have fenced the wrong thing.

Worth keeping as a **second layer** — it is cheap and it does help in the
common grey case — but never as the fence you rely on, and never as the one
that lets you skip the wait in the lease protocol below.

### 3. Scale the primary to zero

The idea: set the EKS workload deployments to 0 replicas, or the node group's
desired capacity to 0, or the Lambda reserved concurrency to 0. No compute, no
writes. This is the most *thorough* fence — it removes the writer rather than
blocking it.

**Verdict: the best fence when the primary is reachable, useless when it is
not, and slower than you think.**

- Scaling a Kubernetes `Deployment` is a call to the **EKS Kubernetes API
  server in the primary region**. Scaling a node group is a call to the **EKS
  and EC2 Auto Scaling control planes in the primary region**. Both are exactly
  the thing that is impaired.
- The Reliability Pillar is explicit that this class of action is the wrong
  shape for a recovery path — [REL11-BP04](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_withstand_component_failures_avoid_control_plane.html),
  verbatim: *"When implementing recovery or mitigation responses to potentially
  resiliency-impacting events, focus on using a minimal number of control plane
  operations."* And from the DR whitepaper, quoted in
  [[failover-orchestration#Gotchas]]: *"Because Auto Scaling is a control plane
  activity, taking a dependency on it will lower the resiliency of your overall
  recovery strategy."*
- Graceful pod termination honours `terminationGracePeriodSeconds` (default 30 s,
  and often raised to 60–120 s precisely so in-flight work finishes). During
  that window the pods are **still writing**. A fence with a deliberate
  keep-writing grace period is a slow fence. If you use this, the runbook needs
  a `--grace-period=0` variant and an explicit acceptance that in-flight
  requests are killed.
- Lambda reserved concurrency = 0 is the one bright spot: it is a fast,
  effective, well-bounded kill switch for Lambda writers specifically, and
  [[aws-lambda]] should carry it. It is still a control-plane call in the
  impaired region.

**Nuance worth keeping:** in the *reachable-but-sick* grey case — which is the
majority of real incidents — scale-to-zero is the most complete fence available
and should be attempt #1. Pair it with a hard grace period. Just never let the
runbook treat its success as a precondition for promotion, because it will fail
open in the case that matters.

### 4. DNS-level fencing

The idea: delete or repoint the internal DNS record the application uses to
find its database (or the record clients use to find the primary region), so
the writer can no longer resolve its target.

**Verdict: not a fence at all. Do not count it as one.** Four independent
reasons, any one of which is fatal:

1. **DNS changes are a Route 53 *control-plane* operation in `us-east-1`.**
   The Fault Isolation Boundaries whitepaper lists *"Making changes to Route 53
   records, like updating an A record's value or changing a weighted record
   set's weights, to perform failover"* as the **first item** in its
   anti-pattern list. REL11-BP04 lists *"Dependence on changing DNS records to
   re-route traffic"* as a common anti-pattern.
2. **Caching.** Record TTL, plus CoreDNS's cache, plus `systemd-resolved`, plus
   the JVM's historical cache-DNS-forever behaviour. [[aws-rds-postgres#Endpoint management at failover]]
   catalogues this in detail. Every layer is a window in which the old writer
   still resolves the old target.
3. **Established connections never re-resolve.** A connection pool holding 50
   open sockets to an IP address will keep using them until they break. This is
   the same point as SG statefulness and it is the decisive one: the writes you
   are trying to stop are flowing over connections that will never do a DNS
   lookup again.
4. It only affects *lookup*, not *authority*. Nothing stops a process that has
   the IP.

DNS is how you **shift traffic** (step 5 of the sequence). It is not how you
**stop writes** (step 1). Conflating the two is one of the most common runbook
errors and it produces a runbook that looks complete and fences nothing.

### 5. Aurora's storage-layer write fencing (the only native one)

If — and only if — the database is [[aws-aurora-global-database]], AWS performs
a genuine fence for you during a managed failover. Verbatim from the
[Aurora global database DR docs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html):

> When you initiate a managed failover, Aurora also attempts to halt write
> traffic through the highly-available Aurora storage layer. We refer to this
> mechanism as "write fencing". If the process succeeds, Aurora emits an RDS
> Event letting you know that writes were stopped. In the unlikely event of
> multiple AZ failures in a Region, it's possible that the write fencing process
> doesn't succeed in a timely manner ... **Because fencing writes is a
> best-effort attempt, it's possible that writes might be momentarily accepted
> in the old primary Region, causing split-brain issues.**

This is the closest thing on the list to Kleppmann's bar — the protected
resource (the Aurora storage layer) is the thing doing the rejecting, not a
firewall in front of it. **But AWS says best-effort, in writing, and also says
it can time out.** Treat it as a very good first layer, not a guarantee, and
note AWS's own first recommendation before failing over is to *take
applications offline* — i.e. AWS also thinks the fence you control is better
than the fence it provides.

**RDS for PostgreSQL has no equivalent.** `promote-read-replica` does nothing
whatsoever to the source instance. The source does not learn that it has been
superseded, does not stop accepting writes, and does not change behaviour in any
way. This asymmetry is a real input to [[rds-vs-aurora-decision]] and it is
under-weighted there relative to the switchover argument.

### 6. Disabling the KMS key (do not do this)

The idea, and people do propose it: the primary's database/table is encrypted
with a regional CMK; disable or schedule deletion of the key and the data
becomes inaccessible, so writes stop.

**Verdict: don't.** It is a control-plane call in the impaired region (same
problem as everything else), the effect is not immediate or well-specified for
a running RDS instance, and on DynamoDB it is actively destructive — an
`INACCESSIBLE_ENCRYPTION_CREDENTIALS` replica state is one of the conditions
that, per [[aws-dynamodb#Failback]], **irreversibly converts a global table
replica to a standalone single-region table after 20 hours**. You would be
using a data-destroying operation as a firewall. Listed here only so that it is
explicitly rejected in writing when someone proposes it at 3am. See [[aws-kms]].

### 7. Application-level write lease (the recommendation)

The idea, and the only one on this list that is *not* an action taken against
the primary during the incident: **the primary fences itself, continuously, by
default, unless it is told otherwise.**

Mechanically:

- A small **lease record** lives in a store outside the primary region — the
  standby region, or ideally a third region. A DynamoDB Global Table row, an S3
  object, a Postgres row in the standby: the storage choice barely matters,
  because the protocol does not require the store to be strongly consistent.
- The primary's write path holds a lease with a **short TTL** — say `L = 30 s` —
  and a **monotonically increasing epoch number**.
- Every `L/3` seconds the primary renews. The renewal is a conditional write
  ("set expiry to now+L **if** epoch == mine").
- **If the primary cannot renew — for any reason: partition, store down, its own
  network gone — it stops accepting writes when its own local clock passes the
  last known expiry.** Fail closed. No call to anything. No control plane. It
  works during a total partition *because* it works on the absence of a signal.
- The standby, before promoting, **increments the epoch and then waits
  `L + clock-skew-margin`** from the lease's last observed expiry. After that
  wait, the old primary has either self-fenced or is so broken that its clock
  has stopped, which is a different conversation.

This is the standard lease protocol from leader election (etcd, Kubernetes
`Lease` objects, Chubby). Two honest caveats, because this note's job is not to
oversell it:

- **It is safe-in-practice, not safe-in-theory.** Kleppmann's whole point is
  that a process can be paused (stop-the-world GC, hypervisor pause, page-fault
  storm) past its lease expiry and resume believing it still holds the lease.
  The complete fix is the fencing token checked *at the resource*, and RDS
  Postgres cannot check one. What you get is: a bounded, well-understood
  residual risk of a handful of writes from a paused process, instead of an
  unbounded stream of writes from a healthy unreachable primary. That is an
  enormous improvement and it is the best available.
- **`L + margin` is real time spent in your 15-minute budget.** With `L = 30 s`
  and a 30 s margin, you have added ~60 s to step 1 of
  [[failover-orchestration#Budget A — RTO(exec): decision → serving, 900 seconds]].
  That is affordable and it is already inside the 60–120 s the budget allocates
  to fencing. Do not be tempted to shrink `L` below ~15 s — a short lease makes
  a transient blip in the lease store into a production write outage.

**The fail-closed property is the point and it is also the risk.** If the lease
store is unreachable, the primary stops writing even though nothing is actually
wrong. Mitigations: put the lease store in a third region so it does not share a
fate with either side of the pair; renew at `L/3` so you tolerate two
consecutive failures; alarm on renewal failures long before expiry. And size `L`
against your RPO, not your RTO — with RPO 2h, a 30-second write pause caused by
an over-eager fence is noise.

#### Sketch, so this is not hand-waving

```python
# In the application's write path. Pseudocode, but this is the whole idea.
# ponytail: single global lease; per-shard leases if one region ever needs
# partial write availability.

LEASE_TTL = 30.0          # seconds
RENEW_EVERY = LEASE_TTL / 3
SKEW_MARGIN = 30.0        # the standby waits TTL + this before promoting

class WriteLease:
    def __init__(self):
        self._expires_at = 0.0      # monotonic clock, NOT wall clock

    def held(self) -> bool:
        # Local check only. No network. This is what makes it work in a partition.
        return time.monotonic() < self._expires_at

    def renew_loop(self):
        while True:
            try:
                # Conditional write against the lease store in a THIRD region.
                # Succeeds only if nobody has bumped the epoch past ours.
                store.renew(region=MY_REGION, epoch=MY_EPOCH, ttl=LEASE_TTL)
                self._expires_at = time.monotonic() + LEASE_TTL
            except Exception:
                metrics.increment("write_lease.renew_failed")   # alarm on this
            time.sleep(RENEW_EVERY)

def handle_write(req):
    if not lease.held():
        # Fail closed. 503, not a silent drop, and not a retry-forever.
        raise ServiceUnavailable("write lease not held in this region")
    return db.write(req)
```

The Terraform surface is small — it is a table, an alarm and a variable:

```hcl
# modules/write-lease/main.tf
# The lease store deliberately lives in a THIRD region so it does not share
# fate with either side of the pair. See failover-orchestration for the
# CA-pair residency caveat on "third region".

resource "aws_dynamodb_table" "write_lease" {
  provider     = aws.lease_store        # e.g. eu-central-1 for the EU pair
  name         = "helios-${var.pair_name}-write-lease"
  billing_mode = "PAY_PER_REQUEST"      # never throttle the thing that gates writes
  hash_key     = "pair"

  attribute {
    name = "pair"
    type = "S"
  }

  point_in_time_recovery { enabled = true }
}

variable "lease_ttl_seconds" {
  type        = number
  default     = 30
  description = <<-EOT
    Write-lease TTL. The standby must wait lease_ttl_seconds +
    lease_skew_margin_seconds after the last observed expiry before promoting.
    Do not drop below 15: a short lease turns a lease-store blip into a
    production write outage. RPO is 2h, so a generous lease costs nothing.
  EOT
  validation {
    condition     = var.lease_ttl_seconds >= 15
    error_message = "Leases shorter than 15s make transient blips into outages."
  }
}

variable "lease_skew_margin_seconds" {
  type    = number
  default = 30
}

# The alarm that matters: renewal failures are the early warning, expiry is
# the outage. Alarm on the former, page on the latter.
resource "aws_cloudwatch_metric_alarm" "lease_renew_failing" {
  provider            = aws.lease_store
  alarm_name          = "helios-${var.pair_name}-write-lease-renew-failing"
  namespace           = "Helios/WriteLease"
  metric_name         = "renew_failed"
  statistic           = "Sum"
  period              = 60
  evaluation_periods  = 2
  threshold           = 3
  comparison_operator = "GreaterThanThreshold"
  alarm_description   = "Primary cannot renew its write lease. It will self-fence at expiry."
}
```

> [!note] Why not just use the lease store as the source of truth for "who is
> primary"?
> Because then the lease store is in the critical path of every write and you
> have built a consensus system. The protocol above deliberately keeps the
> *check* local (`time.monotonic()`) and only the *renewal* remote. The store
> can be down for two renewal intervals and nothing happens.

### Summary table

The last column is the only one that matters.

| # | Technique | Stops in-flight writes? | Speed | Works when the primary's control plane is **unreachable**? |
|---|---|---|---|---|
| 1 | **IAM/SCP deny applied at failover time** | Eventually, unbounded | Unbounded (eventual consistency + caching) | **No.** `us-east-1` control plane; AWS-documented failover anti-pattern |
| 1b | **IAM/SCP deny pre-provisioned, break-glass role excluded** | Yes, for the standby | Instant (already in force) | **Yes — but it fences the *standby*, not the primary** |
| 2 | Security group lockdown | **No** — stateful SGs, established pools survive | 10–60 s when it works | **No.** EC2 control plane in the impaired region; rules may not propagate |
| 3 | Scale primary to zero (EKS/ASG/Lambda) | Yes, after the grace period | 30–120 s | **No.** EKS/ASG control plane in the impaired region |
| 4 | DNS-level fencing | **No** — caching, established connections | Minutes at best | **No.** Route 53 control plane in `us-east-1`; anti-pattern #1 |
| 5 | Aurora storage-layer write fencing | Yes, at the storage layer | Seconds | **Partly — AWS says "best-effort", and it can time out** |
| 6 | Disable the KMS key | Unclear | Unclear | **No**, and it is destructive. Rejected. |
| 7 | **Application write lease (fail-closed)** | **Yes** — the writer stops itself | `L` + skew (~60 s), deterministic | **Yes. This is the only row where the answer is an unqualified yes.** |

### Recommendation: a three-layer fence

Do not pick one. Layer them so that the layer that works in the case that
matters is the one the runbook *waits on*, and the others are opportunistic.

1. **Layer 0 — pre-armed, always on: the write lease.** The only fence that
   survives an unreachable primary. The runbook's fence step is *"confirm the
   lease has expired and wait `L + margin`"*, which is a **wait**, not an API
   call, and therefore cannot fail.
2. **Layer 1 — opportunistic, best-effort: scale the primary's writers to zero**
   (`--grace-period=0`), and set Lambda reserved concurrency to 0. Run it with
   `onFailure: Continue` — see the `FencePrimary` step in
   [[failover-orchestration#Option 2 — SSM Automation documents]], which already
   has exactly this semantic. If it works, you have fenced in 30 seconds and
   can shorten the wait. If it fails, you lose nothing.
3. **Layer 2 — pre-armed, always on: the standby-side SCP deny with an excluded
   `FailoverBreakGlassRole`**, per [[aws-dynamodb#Blocking writes to the standby with IAM]].
   This one runs backwards: it prevents the *standby* from being written to by
   accident while the primary is healthy, which is the DynamoDB flavour of
   split-brain and is far more likely to happen on a random Tuesday than a real
   regional failure is.

**And one thing that is not a fence but does more good than any of them:
[ARC gating rules](https://docs.aws.amazon.com/r53recovery/latest/dg/route53-arc-best-practices.regional.html).**
Model "database promoted" as its own routing control and gate the standby's
traffic control behind it, as [[failover-orchestration#The switch: Route 53 ARC routing controls]]
describes. That does not stop the old primary writing, but it makes the most
expensive ordering error structurally impossible, and it costs a policy
document.

## The one-way door: what RDS promotion irreversibility means for the decision

[[aws-rds-postgres#Promotion is irreversible — and the consequences are larger than they look]]
establishes the fact. This note is about what the fact implies for the
*decision*, which is a different question and gets much less attention.

There is no `demote-db-instance`. AWS, verbatim: *"After you promote the read
replica, it ceases to function as a read replica and becomes a standalone DB
instance ... You can't use the DB instance as a replication target because it is
no longer a read replica."*

### Consequence 1 — the decision is not symmetric, so your bias should not be

Two errors are available at the decision point, and they are not the same size:

| Error | Cost |
|---|---|
| **Fail over too late.** You waited 5 extra minutes and the region really was gone. | 5 minutes of additional outage. Linear, bounded, recoverable, forgettable. |
| **Fail over too early.** The primary was alive. | Two divergent writable databases; a reconciliation that is data forensics, not a runbook step; and — per GitHub — possibly *no safe path back at all*. Non-linear, unbounded. |

**Therefore: bias towards waiting.** This is not cowardice, it is the correct
expected-value calculation on an asymmetric payoff, and it should be written
into the go/no-go criteria explicitly rather than left to a tired engineer's
temperament. Concretely, it means the criteria in
[[failover-orchestration#Recommendation]] should be **conjunctive** (all four
conditions, AND) and should include a **minimum observation window** — "failing
continuously for ≥ 5 minutes", not "currently failing".

### Consequence 2 — you cannot rehearse it on the real standby

Promoting the standby to test it destroys the standby: it stops being a replica,
and rebuilding it is a full cross-region reseed measured in hours. So **every
rehearsal either uses a throwaway replica or burns the real one.** This is a
budget line, not a footnote, and it is the reason
[[dr-testing-and-gamedays]] has to distinguish "test the mechanism" from "test
the whole failover" — with RDS they are not the same exercise.

Aurora does not have this problem: `switchover-global-cluster` is lossless and
reversible, so the rehearsal *is* the real thing. That is a genuine operational
argument for Aurora which [[rds-vs-aurora-decision]] should weigh alongside the
cost.

### Consequence 3 — "promote" should be the *last* thing in the sequence that is irreversible, and everything before it should be free

This is why [[failover-orchestration]] puts the approval gate between step 1
(fence) and step 2 (promote), and why the freeze and the fence are both cheap
and reversible. The design principle generalises: **automate what has an undo;
gate what does not.** The write lease fits this perfectly — letting the lease
expire is reversible (renew it and the primary resumes), so a robot can be
trusted with it. Promotion is not, so a human owns it.

## The other split-brain: DynamoDB Global Tables

Everything above assumes a single writable primary and a read-only standby.
**DynamoDB does not work that way and the difference is dangerous precisely
because it is invisible.**

[[aws-dynamodb#Multi-active by design — and you asked for active/passive]] has
the detail; the split-brain-specific summary is:

- **There is no configuration flag that makes a replica read-only.** The standby
  table in `eu-west-2` will accept a `PutItem` from anything with credentials
  and network access, today, while the primary is perfectly healthy.
- **A stray standby write replicates *back*.** Conflict resolution is
  *"last-writer-wins ... based on item-level timestamps"*, so the stray write —
  being newer — wins, and the real record is gone.
- **The loss is silent.** No error. No DLQ. No metric. `ReplicationLatency` is
  fine, because replication worked perfectly; it faithfully replicated the wrong
  thing. There is no "conflicts detected" counter to alarm on. You find out from
  a customer.
- **LWW is item-level, not attribute-level.** A concurrent write to a
  *different attribute* of the same item still discards the other attributes'
  changes. People expect a merge. There is no merge.

So the DynamoDB flavour of split-brain is not a failover-time risk, it is a
**Tuesday-afternoon risk**: a data-fix script run with a stale `AWS_REGION`, a
CI job with a hard-coded region, a canary left running after a game day. The
failover sequence does nothing to cause it and nothing to prevent it.

**The fence is therefore permanent, not incident-time**, and it is Layer 2
above: a standing SCP denying writes in the standby region, with the DynamoDB
replication service-linked role and the Application Auto Scaling SLR explicitly
excluded (omitting them is a 20-hour fuse on an irreversible conversion to
single-region tables — see [[aws-dynamodb#Blocking writes to the standby with IAM]]),
plus a pre-created `FailoverBreakGlassRole` that the standby's compute assumes.

And the failover-time instruction is a **negative** one, which is why it needs
to be in bold in the runbook: **do not "promote" the DynamoDB table. Do not
remove the replica.** The instinct to promote is wrong; there is nothing to
promote; removing the replica is a control-plane operation that permanently
severs replication and converts failback into a full re-backfill.

There is one genuinely nice consequence: because the standby is always writable
and always current, **DynamoDB's failback is nearly free** — re-apply the fence
to the standby, release it on the primary, shift traffic. It is the only
component in the estate where failback is a change ticket rather than a project.

## Failback

> Everything above is the first half of the incident. This is the half that
> gets skipped in the design review, and it is bigger.

The observation that drives this entire section: **failover is a 15-minute
event under duress with executive attention; failback is a multi-week project
with no attention at all, and it is where the second outage lives.**

### Why it is harder than failover

Five reasons, roughly in order of how much they hurt:

1. **Replication runs the wrong way and cannot simply be turned around.** For
   RDS Postgres the old primary cannot be attached as a replica of the new one.
   There is no demote. You are rebuilding, not reversing.
2. **The two sides have diverged**, by exactly the amount of data your RPO
   permitted to be lost, plus anything the old primary wrote after the replica's
   last received WAL. Those writes exist, they are real customer data, and
   nothing in AWS will reconcile them for you.
3. **The reseed is slow and expensive.** A cross-region replica build is the
   same four-phase snapshot-copy-and-seed as the original, and the AWS docs say
   *"this process can take hours to complete."* The whole database crosses the
   region boundary again, at cross-region transfer rates.
4. **Failback requires a *second* outage** — a planned one, but a real one. The
   failback promotion is still a `promote-read-replica`: still irreversible,
   still a reboot, still ~5–15 minutes of write unavailability. **There is no
   zero-downtime failback for RDS Postgres.** Which means you have to go back to
   the business and schedule downtime, days after the incident, to undo the
   incident. That conversation is much harder than it sounds.
5. **Nobody is watching any more.** The incident channel is closed, the
   executives have moved on, the on-call has slept. The failback is executed by
   whoever picks up the ticket, against a runbook that — if this note has done
   its job — actually exists.

### Reconciling divergent writes: the part with no tooling

This is the honest section. **AWS provides nothing for this.** The Step
Functions DR reference implementation
([AWS networking blog](https://aws.amazon.com/blogs/networking-and-content-delivery/orchestrate-disaster-recovery-automation-using-amazon-route-53-arc-and-aws-step-functions/))
is explicit that it does not handle data reconciliation. Aurora's
best-effort write fencing exists precisely because AWS knows divergence can
happen and has no answer for it either.

What you actually have:

- **The last-received WAL LSN, if you recorded it.** Step 1 of
  [[aws-rds-postgres#Failover procedure]] is `SELECT pg_last_wal_receive_lsn()`
  *before* promoting. That single number is the boundary between "replicated"
  and "lost", and it is the only mechanical artefact you will have. **If the
  runbook does not capture it, reconciliation becomes guesswork.** This is the
  highest-value 10 seconds in the entire failover.
- **The old primary's data, if it comes back.** Take a final snapshot before
  you delete it. Do not delete it in a hurry; do delete it eventually, because
  a writable zombie Postgres still answering to an old DNS name is how you get
  split-brain a second time.
- **Aurora's unplanned-failover snapshot.** Aurora *"attempts to take a snapshot
  of the old storage volume at the point of failure"*, named
  `rds:unplanned-global-failover-<cluster>-<timestamp>`. **It is subject to the
  old cluster's backup retention period** — copy it to a manual snapshot
  immediately or it ages out while you are still deciding what to do with it.
- **Application-level idempotency keys and event logs**, if the application has
  them. This is where reconciliation actually succeeds or fails, and it is an
  application design decision made years before the incident. Orders with
  idempotency keys can be replayed; a counter that was incremented cannot be.

And what you do **not** have:

- **`pg_rewind` is not available.** On self-managed Postgres, `pg_rewind` is the
  standard tool for exactly this — *"synchronizing a PostgreSQL cluster with
  another copy of the same cluster, after the clusters' timelines have
  diverged"*, per the [PostgreSQL docs](https://www.postgresql.org/docs/current/app-pgrewind.html).
  It needs filesystem access to the data directory, `wal_log_hints` or data
  checksums, and enough retained WAL to cover the divergence. **RDS gives you
  none of these**, and no public AWS documentation was found describing
  `pg_rewind` against an RDS instance. **No supported path exists.** Treat this
  as a real, named cost of the managed service — it is not a gap in this
  research, it is a gap in the platform.

**Practical recommendation:** do not plan to reconcile automatically. Plan to
(a) capture the LSN, (b) snapshot the old primary, (c) produce a list of
affected primary keys by diffing the snapshot against the new primary above the
LSN boundary, and (d) hand that list to the business to decide, per record,
what "correct" means. Write step (c) as a script *now*, while nobody is
shouting. It is a day of work and it is the difference between a two-day
reconciliation and a two-week one.

### The decision: ping-pong vs permanent swap

This is the genuine fork, and per the vault's rules both branches get a fair
hearing.

#### Branch A — ping-pong (fail back to the original primary)

After the incident, rebuild replication in reverse, wait for it to catch up,
then schedule a planned failback and return to the original topology.

**For:**
- **The original primary region is the one you actually designed for.** Latency
  to your users, to your partners, to whatever is not multi-region (an
  on-premises system, a third-party VPN, a payment provider's allowlist) is
  measured against it.
- **Instance types, quotas and service availability are known-good there.** The
  standby has never carried production load for a month. Its quotas were sized
  for a warm standby. Its EC2 capacity was never tested at peak — and ARC's own
  best-practices page warns, verbatim: *"Region switch does not guarantee that
  the desired compute capacity with be attained."*
- **Data residency may not be negotiable.** For the CA pair especially, the
  primary region may be the one your legal position was written against. See
  [[data-residency]].
- **You return to a known state.** Only one direction of the runbook has ever
  been rehearsed; staying swapped means the *next* failover runs a direction
  you have never practised.
- It keeps the asymmetry of primary and standby, which may be baked into cost
  allocation, reserved instances and Savings Plans — all of which are
  region-scoped and now cover the wrong region.

**Against:**
- **Two full reseeds per round trip**, hours each, at cross-region transfer
  rates, for RDS.
- **A second planned outage**, which you must sell to the business after they
  have just absorbed an unplanned one.
- **A second irreversible promotion**, i.e. a second opportunity for all of the
  above to happen again, this time entirely self-inflicted.
- Twice the exposure to the whole class of failure, for a benefit that is
  usually "it feels tidier".

#### Branch B — permanent swap (don't fail back; make the new region primary)

After the incident, the standby *is* the primary. You build a fresh standby in
the old primary region and carry on.

**For:**
- **One reseed instead of two. One outage instead of two. One irreversible
  promotion instead of two.** Halves the risk and halves the work, immediately,
  with no cleverness required.
- **The reseed happens on your schedule**, unattended, while you serve traffic
  normally. Nothing is urgent.
- **It forces the Terraform to be symmetric**, which is independently the right
  thing. If "which region is primary" is a variable rather than a structural
  property of the code, you get a free property: *both directions of the
  failover are the same code path*, so both are equally rehearsed. If it is
  not, the permanent swap is impossible and you have just learned something
  important about the repo. See [[terraform-repo-structure]] and
  [[provider-aliases-vs-separate-stacks]].
- **It is honest about what a warm standby is for.** A standby you are unwilling
  to live in is not a standby; it is a very expensive backup.

**Against:**
- **You may not be able to.** Data residency, latency to a non-multi-region
  dependency, a third party's IP allowlist, or a service that simply is not
  available in the standby region. **`ca-west-1` is the live example** —
  [[region-pair-selection]] flags Calgary's service parity as an open risk, and
  "permanently primary" is a much higher bar than "briefly primary".
- **Capacity and quotas.** Service quotas are per-region and the standby's were
  never raised. Reserved Instances and Savings Plans are region-scoped and now
  subsidise an idle region. This is real money and a real lead time.
- **Half the runbook has never been run in that direction.** Mitigated by the
  symmetry argument above, but only if the symmetry is genuine.
- It can become an excuse never to think about the old region again, which is
  how you end up with a standby that has silently rotted — see
  [[dr-testing-and-gamedays]].

#### Recommendation

**Default to the permanent swap (Branch B) for unplanned failovers on RDS
Postgres, for the EU and US pairs.** One reseed, one outage, one one-way door.
The "tidiness" argument for ping-pong does not survive contact with the cost of
a second irreversible promotion, and the AWS DR whitepaper's framing supports
it: every failover carries loss, so minimise the number of failovers, not the
asymmetry of the topology.

Three qualifications, all of which matter:

1. **The CA pair is exempt until `ca-west-1` parity is confirmed.** Living
   permanently in Calgary is a stronger claim than briefly failing into it. If
   parity is not established, the CA pair defaults to ping-pong and the failback
   plan must be written out in full. Take this to [[region-pair-selection]].
2. **If the database is Aurora, invert the recommendation: ping-pong.**
   `switchover-global-cluster` is RPO 0, roughly a minute of unavailability,
   preserves the topology, and the old primary region rejoins automatically as a
   secondary — verbatim: *"As soon as that old primary Region is healthy and
   available again, Aurora automatically adds it back to the global cluster as a
   secondary Region."* At that price the failback is a change ticket, and AWS
   explicitly lists regional rotation as a switchover use case. **On Aurora you
   should be ping-ponging deliberately, on a schedule, as a rehearsal** — see
   [[dr-testing-and-gamedays]]. This makes the ping-pong-vs-swap answer
   *depend on* [[rds-vs-aurora-decision]], and that dependency should be stated
   in that note.
3. **DynamoDB does not get a vote and does not need one.** Its failback is
   symmetric and cheap either way (re-apply the fence to one side, release it on
   the other). Whichever branch the relational database takes, DynamoDB follows
   for free.

**Whichever you choose, choose it in advance and write it in the runbook
header.** "Do we fail back?" is not a question to start answering at 04:00, and
it is not a question to start answering three days later either, because by then
somebody will already have deleted something.

### The failback sequence, at the level of detail that is actually missing

Assume RDS, Branch B (permanent swap). The steps nobody writes down:

| # | Step | Owner | Notes |
|---|---|---|---|
| 1 | **Declare the swap.** New region is primary, permanently, until further notice. | Incident commander | Must be an explicit, communicated decision, not a drift |
| 2 | **Snapshot the old primary, then stop it.** Do not delete yet. | DBA | Forensics + reconciliation input. Stopping it is the second fence |
| 3 | **Reconcile.** Diff above the recorded LSN, produce the affected-record list, get business decisions | Eng + business | The only genuinely hard step. Days, not hours |
| 4 | **Flip the `primary_region` variable in Terraform and plan.** | Platform | **If this produces a destroy/recreate, stop — the repo is not symmetric and that is now the top-priority finding.** See [[terraform-repo-structure]] |
| 5 | **Build the new replica in the old primary region.** Hours. Unattended. | Platform | ARC Region switch models this as a first-class *post-recovery workflow* with an RDS Create Cross-Region Replica execution block — see [[failover-orchestration#Option 1 — ARC Region switch (recommended)]] |
| 6 | **Invert every fence.** Lease epoch, SCP region condition, break-glass role placement, consumer enable/disable flags, EventBridge rule states. | Platform | **The most commonly missed step.** Every asymmetric setting in this note now points the wrong way |
| 7 | **Re-point monitoring, alarms, dashboards, canaries and paging.** | SRE | Third-region canaries must now probe the new pair orientation |
| 8 | **Delete the old primary.** Only now. | DBA | A writable zombie with an old DNS name is split-brain round two |
| 9 | **Re-baseline cost, RIs, Savings Plans and service quotas.** | Finance + Platform | Region-scoped, and now all wrong |
| 10 | **Run a game day in the new direction within 30 days.** | SRE | Otherwise you are single-region in practice. [[dr-testing-and-gamedays]] |

Step 6 is the one to pin on the wall. **Every fence in this note is
directional**, and after a permanent swap every one of them is pointing at the
wrong region. An SCP that denies writes in `eu-west-2` is, the day after a
permanent swap, an SCP that denies writes in **production**.

## Gotchas

1. **"The API call timed out" is not "the fence succeeded."** It is "I have no
   information". Any runbook step that treats a failed fence API as proof of
   anything is broken. This is why the lease wait exists.
2. **Security groups are stateful; revoking ingress does not kill established
   connections.** The writes you are trying to stop are on connections that
   already exist.
3. **`terminationGracePeriodSeconds` is a deliberate keep-writing window.** If
   you fence by scaling to zero, use `--grace-period=0` and accept the
   in-flight failures, or your fence is 60–120 seconds slower than you think.
4. **The JVM's DNS cache and every connection pool in the estate mean DNS is not
   a fence.** See [[aws-rds-postgres#Endpoint management at failover]].
5. **An SCP deny that catches `AWSServiceRoleForDynamoDBReplication` has a
   20-hour fuse on an irreversible conversion to single-region tables.** Always
   exclude the replication SLR and the Application Auto Scaling SLR.
6. **Editing IAM during a failover is an AWS-documented anti-pattern** and
   depends on `us-east-1`. Pre-provision every policy and role the failover
   path needs, months in advance.
7. **Aurora's write fencing is best-effort and can time out.** AWS says so in
   writing. Do not design as though it is a guarantee.
8. **The write lease fails closed, which means the lease store is now a
   production dependency.** Put it in a third region, renew at `L/3`, size `L`
   generously, and alarm on renewal failures rather than on expiry.
9. **A paused process can outlive its lease.** GC, hypervisor pause, page-fault
   storm. Complete safety needs a fencing token checked at the resource, and
   RDS Postgres cannot check one. Know the residual risk; do not pretend it is
   zero.
10. **`pg_rewind` does not exist for you.** The standard Postgres tool for
    resynchronising a diverged old primary requires filesystem access RDS does
    not give you. Failback is a reseed.
11. **Capture `pg_last_wal_receive_lsn()` before promoting.** Ten seconds. It is
    the only mechanical record of the divergence boundary you will ever have.
12. **Copy Aurora's `rds:unplanned-global-failover-*` snapshot to a manual
    snapshot immediately.** It ages out with the old cluster's retention period.
13. **Every fence is directional and must be inverted after a permanent swap.**
    Missing this turns your standby fence into a production outage.
14. **A stopped-but-not-deleted old primary that still answers to an internal
    DNS name is split-brain waiting to happen twice.** Delete the record before
    you forget about the instance.
15. **Fencing is redundant exactly when it is unnecessary and essential exactly
    when it is not**, which is why it is the step most likely to be skipped
    under pressure. Make it a wait, not a judgement call.

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| **Primary fencing mechanism** | Incident-time actions (SG lockdown / scale to zero / IAM deny) | Pre-armed application write lease, fail-closed, with incident-time actions as an opportunistic second layer | **B.** A is an AWS-documented anti-pattern and fails in the exact scenario it exists for. B is the only technique that works when the primary is unreachable. |
| **Lease TTL** | 10 s (fast failover) | 30 s + 30 s skew margin | **B.** 60 s fits inside the 60–120 s the RTO budget already allocates to fencing, and a short lease turns a lease-store blip into a production write outage. RPO is 2h; there is no reason to be greedy here. |
| **Lease store location** | Standby region | A third region | **B**, for the same reason the orchestrator lives in a third region. Note the CA pair's residency problem, identical to the one in [[failover-orchestration#Recommended placement]]. |
| **Standby write fence (DynamoDB)** | Rely on process and convention | Standing SCP + excluded `FailoverBreakGlassRole` | **B.** The stray-write failure mode is a Tuesday risk, not a failover risk, and process does not survive a stale `AWS_REGION` in someone's shell. |
| **Failback strategy — RDS, EU and US pairs** | Ping-pong back to the original primary | **Permanent swap**; build the fresh standby in the old primary region | **B.** One reseed, one outage, one one-way door instead of two of each. Conditional on the Terraform being symmetric in `primary_region`. |
| **Failback strategy — CA pair** | Permanent swap | Ping-pong | **B, until `ca-west-1` parity is confirmed.** Living permanently in Calgary is a stronger claim than briefly failing into it. [[region-pair-selection]] |
| **Failback strategy — if Aurora** | Permanent swap | Ping-pong via `switchover-global-cluster` | **B, and do it deliberately on a schedule.** RPO 0, ~1 minute, topology preserved. Turns failback into a rehearsal instead of a risk. |
| **Reconciliation tooling** | Figure it out during the incident | Write the LSN-boundary diff script now | **B.** One day of work, and it is the difference between a two-day and a two-week reconciliation. |

## Open questions

1. **Can the application actually hold a write lease?** This is an application
   change, not an infrastructure one, and it needs a real answer from whoever
   owns the services. If the answer is no, the honest position is that **this
   estate cannot fence its primary**, and the go/no-go criteria have to get
   correspondingly more conservative to compensate.
2. **Where does the EU pair's lease store live?** `eu-central-1` is the obvious
   answer; confirm it is acceptable under [[data-residency]]. The CA pair has
   the same no-third-region problem as the orchestrator.
3. **What is the estate's actual write mix?** The value of a fence scales with
   write rate. A service writing 10 rows/second diverges slowly enough that a
   manual reconciliation is plausible; one writing 10,000/second does not.
4. **Do the applications have idempotency keys on their write paths?** This
   determines whether reconciliation is "replay the list" or "phone the
   customers". It is the single biggest input to failback cost and nobody has
   asked.
5. **Is the Terraform symmetric in `primary_region` today?** The permanent-swap
   recommendation depends entirely on it. A 20-minute `terraform plan` with the
   variable flipped answers it. Take the result to
   [[provider-aliases-vs-separate-stacks]].
6. **Are the standby regions' service quotas sized for sustained production, or
   for a warm standby?** Relevant to the permanent swap, and quota increases
   have lead times measured in days.
7. **Has anyone confirmed there is no supported `pg_rewind` path on RDS?** This
   note found none and states so; it is worth a support ticket, because if one
   exists it materially cheapens failback.

## Sources

- [Amazon Builders' Library — Static stability using Availability Zones](https://aws.amazon.com/builders-library/static-stability-using-availability-zones/) — the control-plane/data-plane definitions, the definition of static stability, and the decisive verbatim line that during an EC2 control-plane impairment a physical server *"might not see updates like a new EC2 instance added to a VPC, or a new Security Group rule."* **The primary citation for why SG-based fencing fails in the case that matters.**
- [AWS Fault Isolation Boundaries — Global services](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/global-services.html) — IAM's control plane in `us-east-1` *"with isolated data planes in each Region of the partition"* (the basis for pre-provisioned-policy fencing); the partitional service list; the anti-pattern list including *"Creating or updating IAM resources, including IAM roles and policies, during a failover"* and *"Making changes to Route 53 records ... to perform failover"*; the STS global-endpoint default and break-glass user guidance.
- [REL11-BP04 Rely on the data plane and not the control plane during recovery](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_withstand_component_failures_avoid_control_plane.html) — *"focus on using a minimal number of control plane operations"*; the anti-pattern *"Dependence on changing DNS records to re-route traffic"*; the static-stability prescription and the explicit note that *"Adding pods is a data plane action in Kubernetes. Actions should be limited to adding pods and not adding nodes."*
- [Troubleshoot IAM — Changes that I make are not always immediately visible](https://docs.aws.amazon.com/IAM/latest/UserGuide/troubleshoot.html) — IAM's eventual consistency, the caching caveat, and verbatim *"We recommend that you do not include such IAM changes in the critical, high availability code paths of your application."*
- [Martin Kleppmann — How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) — the definition of a fencing token; *"The storage server remembers that it has already processed a write with a higher token number"*; and the requirement that *"this requires the storage server to take an active role in checking tokens, and rejecting any writes on which the token has gone backwards."* **The bar against which every technique in this note is measured, and which only Aurora comes close to meeting.**
- [GitHub — October 21 post-incident analysis](https://github.blog/news-insights/company-news/oct21-post-incident-analysis/) — 43 seconds of partition, 24h11m of degradation; divergent writes in both data centres; *"we were unable to fail the primary back over to the US East Coast data center safely"*; and the remediation preventing cross-region primary promotion. **The canonical public split-brain-caused-by-successful-failover case.**
- [Using switchover or failover in Amazon Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html) — the write-fencing mechanism, the RDS Event on success, the timeout behaviour under multiple AZ failures, and the verbatim admission that *"Because fencing writes is a best-effort attempt, it's possible that writes might be momentarily accepted in the old primary Region, causing split-brain issues."* Also the automatic re-add of the recovered old primary as a secondary, which is the basis of the Aurora ping-pong recommendation.
- [PostgreSQL documentation — `pg_rewind`](https://www.postgresql.org/docs/current/app-pgrewind.html) — what the tool does (*"synchronizing a PostgreSQL cluster with another copy of the same cluster, after the clusters' timelines have diverged"*) and its requirements: filesystem access, `wal_log_hints` or data checksums, and sufficient retained WAL. **Establishes what is not available on RDS.**
- [Guidance for Cross Region Failover & Graceful Failback on AWS](https://docs.aws.amazon.com/solutions/cross-region-failover-and-graceful-failback-on-aws/) — AWS's own reference pattern: an operator-triggered "Failover" button invoking an SSM document that (1) fails over the Aurora global database, (2) updates the database endpoint in Secrets Manager, (3) flips the ARC routing controls. Note what it does **not** cover: any form of write fencing on the old primary, or any data reconciliation. Useful as much for its omissions as its content.
- [Orchestrate disaster recovery automation using Amazon Route 53 ARC and AWS Step Functions](https://aws.amazon.com/blogs/networking-and-content-delivery/orchestrate-disaster-recovery-automation-using-amazon-route-53-arc-and-aws-step-functions/) — the deactivate-primary-first ordering, and the explicit acknowledgement that the reference implementation does not handle data reconciliation.
- [Best practices for Region switch in ARC](https://docs.aws.amazon.com/r53recovery/latest/dg/best-practices.region-switch.html) — verbatim *"Region switch does not guarantee that the desired compute capacity with be attained"*, which is the sharpest argument against assuming the standby can carry production indefinitely under a permanent swap.

## Related notes

[[failover-orchestration]] · [[failover-runbook-template]] · [[dr-testing-and-gamedays]] · [[aws-rds-postgres]] · [[aws-aurora-global-database]] · [[rds-vs-aurora-decision]] · [[aws-dynamodb]] · [[aws-iam]] · [[aws-kms]] · [[aws-route53]] · [[aws-eks]] · [[aws-lambda]] · [[messaging-in-flight-data-loss]] · [[terraform-repo-structure]] · [[provider-aliases-vs-separate-stacks]] · [[region-pair-selection]] · [[data-residency]] · [[security-posture-of-the-standby]] · [[failback]] · [[open-decisions]]
