---
title: Amazon CloudFront — Multi-Region
service: cloudfront
tags: [service, multi-region, cloudfront, cdn, edge, failover, origin-groups]
status: partial
replication: global — nothing to replicate; the distribution is already everywhere
rpo_achievable: N/A — cache and configuration, not data
rto_achievable: "seconds (origin group failover, no DNS change) for GET/HEAD/OPTIONS; seconds (CloudFront Function + KeyValueStore origin selection) for all methods; 5–15 min if the plan is to edit the distribution"
meets_targets: conditional — yes as a failover *mechanism*, no if your runbook edits the distribution
updated: 2026-09-17
---

# Amazon CloudFront — Multi-Region

## TL;DR

- **CloudFront is global by nature. It is not something you mirror.** There is no "CloudFront in `eu-west-1`". One distribution is served from every edge location on earth. **Do not create a second distribution for the standby region** — that is the single most common wrong instinct on this note, and it costs you a second certificate, a second WAF ACL, a second cache, and a DNS layer you did not need. The question this note answers is not *"how do I make CloudFront multi-region"* but ***"how do I use CloudFront as the failover mechanism"***.
- **Origin groups are a genuinely strong RTO option and the best one in this vault for the read path.** Failover happens **at the edge, per request, in seconds**, with **no `ChangeResourceRecordSets` call, no Route 53 control plane, no TTL, and no client DNS cache problem at all**. Compare [[aws-route53]], where ~90 seconds of the 900-second budget is DNS and the JVM cache can silently extend that to an hour. Origin failover has none of that. See [[#Origin groups and origin failover]].
- **But origin failover only applies to `GET`, `HEAD` and `OPTIONS`.** AWS states this verbatim and there is no setting to change it. For a write-heavy API, **`POST`/`PUT`/`PATCH`/`DELETE` are returned to the client as errors and are never retried against the standby.** This one sentence decides whether origin groups are your failover mechanism or just a nice safety net on the read path. Read [[#The GET/HEAD/OPTIONS limitation — the decision point]] before anything else.
- **There is a way around it, and it is new: a CloudFront Function at *viewer request* that calls `cf.selectRequestOriginById()` or `cf.createRequestOriginGroup()`, reading an active-region flag out of CloudFront KeyValueStore.** That is *origin selection*, not *origin failover*, so it applies to **every** HTTP method, and a KVS key update propagates to all edges "in a few seconds" **without a distribution deployment**. This is the highest-value finding in this note. See [[#Edge functions as the real failover switch]].
- **The thing that will bite:** two things, and they are both about the seconds after the switch. (1) **CloudFront does not remember that it failed over** — "CloudFront routes all incoming requests to the primary origin, even when a previous request failed over" — so *every* cache-miss request pays the full primary timeout (**up to 30 s by default**) before reaching the standby. (2) **Cold cache thundering herd**: the standby takes 100% of origin traffic with an empty edge cache, while it is scaling out and its database was just promoted. See [[#The cold-cache thundering herd]].

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
3. **The distribution is a shared fate you cannot mirror away.** On **16 July 2026** a CloudFront **VPC Origins** failure served global 5xx for ~3h33m to every customer using that feature, and *no regional failover helped*, because the fault was in a global configuration-distribution system. See [[#Case study — the 16 July 2026 CloudFront VPC Origins event]].

> [!important] Say this to the team once, clearly
> "Replicate the CloudFront distribution to eu-west-2" is not a ticket. There is nothing to replicate. If such a ticket exists, close it and replace it with "add the standby ALB as a second origin and decide how we choose between them."

## Still to research

This note was cut off mid-write. The TL;DR above is the researcher's own map of
what was found but never written up — treat it as a brief, not as findings. Every
section below is referenced by a link in the TL;DR and does not yet exist:

- `## Origin groups and origin failover` — the failover criteria, timeouts and
  attempt counts; what the 30 s default connection timeout does to the budget.
- `## The GET/HEAD/OPTIONS limitation — the decision point` — verify the verbatim
  AWS wording and decide whether origin groups are the mechanism or a read-path
  safety net only.
- `## Edge functions as the real failover switch` — `cf.selectRequestOriginById()`
  / `cf.createRequestOriginGroup()` plus KeyValueStore; confirm propagation time
  and the KVS consistency model against a real AWS page.
- `## The cold-cache thundering herd` — Origin Shield placement as the mitigation.
- `## Terraform implementation` — the single global edge stack consuming both
  regional stacks' outputs.
- `## Case study — the 16 July 2026 CloudFront VPC Origins event` — **must be
  verified against a first-party AWS post-event summary before it is trusted.**
  It is currently an unsourced claim in the TL;DR.

Also missing and required by [[note-template]]: Migration path, Failover
procedure, Failback, Decisions to make, Cost, Open questions, Sources.
