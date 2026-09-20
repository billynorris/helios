---
title: AWS Lambda — Multi-Region
service: lambda
tags: [service, multi-region, lambda, compute, serverless]
status: researched
replication: none — deploy a second copy
rpo_achievable: "N/A — the function itself is stateless. RPO belongs to whatever it reads and writes."
rto_achievable: "< 1 min for the control plane; 2-10 min for real traffic unless concurrency is pre-warmed and the regional quota is pre-raised"
meets_targets: conditional — yes on the function, no by default on the concurrency quota
updated: 2026-09-20
---

# AWS Lambda — Multi-Region

## TL;DR

- **Lambda is stateless, so "deploy a second copy into the standby region" is the
  correct and complete answer to the function itself.** There is no replication
  to configure, no data to sync, no promotion step. The Terraform is a provider
  alias and a second module instantiation. If the note stopped here it would be
  four lines long. It does not stop here, because *everything around the
  function* is regional and most of it fails silently.
- **The deployment artifact is the first thing that breaks.** A `.zip` in S3 must
  live in a bucket **in the same Region as the function** — AWS returns
  `PermanentRedirect` otherwise. A container image must live in an **ECR
  repository in the same Region as the function**. Both S3 Cross-Region
  Replication and ECR cross-region replication are **not retroactive**: turn them
  on today and yesterday's artifacts are still only in the primary. See
  [[aws-s3]] and [[aws-ecr]] — this is the same trap in two services.
- **The account concurrency quota is the silent RTO killer.** It is **1,000 by
  default and it applies per AWS Region** ([AWS Lambda
  quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)).
  If someone raised `eu-west-1` to 10,000 three years ago via a support ticket,
  `eu-west-2` is still at 1,000 and nobody will find out until 100% of
  production traffic arrives there at 03:00 and 90% of it gets a `429`. Raise the
  standby quota *now*, as a deliverable, and put a CloudWatch alarm on the
  standby's `ConcurrentExecutions` against the quota.
- **The real cost lever is provisioned concurrency in the standby, and the honest
  answer is: mostly don't buy it.** Lambda's on-demand scaling rate is **1,000
  new execution environments every 10 seconds, per function, per Region** — a
  fleet that needs 3,000 concurrency gets there in ~30 seconds of degraded
  latency, not 15 minutes. Buy provisioned concurrency only for the handful of
  synchronous, user-facing, slow-initialising functions on the critical path, and
  buy it *scheduled to zero* until failover only if you accept that allocation
  takes "a minute or two of preparation" plus ramp. Quantified below.
- **The thing that will bite:** event source mappings. An ESM must point at a
  source **in its own Region** (SQS explicitly: "The Lambda function and the
  Amazon SQS queue must be in the same AWS Region"). So the standby ESM points at
  the *standby* queue, which is a different queue with different messages — and
  if you leave it `Enabled` while the primary is healthy you have two live
  consumers. Deploy it `Enabled = false` and flip it at failover. AWS now ships a
  purpose-built primitive for exactly this flip (ARC Region Switch's Lambda ESM
  execution block). See [[messaging-in-flight-data-loss]] and
  [[split-brain-and-fencing]].

---

## Does this service cross regions at all?

No, and that is a feature.

A Lambda function is a **regional resource** with a regional ARN:

```
arn:aws:lambda:eu-west-1:111122223333:function:helios-orders-api:prod
```

There is no "global function", no `replica` block (contrast
[[aws-secrets-manager]]), no global table analogue (contrast [[aws-dynamodb]]),
no cross-region endpoint. `lambda:InvokeFunction` *can* be called cross-region by
an SDK client that constructs a regional endpoint — the API is reachable from
anywhere — but nothing in Lambda's own machinery reaches across a Region
boundary.

The two partial exceptions are worth naming precisely because they confuse
people:

1. **Lambda@Edge.** The function is *authored* in `us-east-1` and AWS replicates
   it to edge locations worldwide on your behalf via the
   `AWSServiceRoleForLambdaReplicator` service-linked role. This is a genuine
   AWS-managed cross-region replication — and it is the **only** one Lambda has.
   It applies only to CloudFront-triggered functions and it is not available to
   your ordinary application functions. Covered in full below and in
   [[aws-cloudfront]].
2. **Cross-account, same-region.** Both S3 artifact buckets and ECR repositories
   may be in a *different account* as long as they are in the *same Region*. This
   trips people who read "cross-account is supported" and assume cross-region is
   too. It is not.

Everything else — versions, aliases, ESMs, concurrency settings, function URLs,
provisioned concurrency allocations, the CloudWatch log group — is per-Region and
must exist twice.

### What this means for the Helios shape

Because [[research-brief|the brief]] says each of the three deployments is a
self-contained copy of the product with no data sharing, the Lambda story is
mechanically the easiest in the entire vault. There is no "promote the replica"
step because there is no replica — there is a second, independently-created
function that happens to run identical code. The entire multi-region cost of
Lambda is:

| Cost | Where it lands |
|---|---|
| Artifact availability in the standby Region | Build/CI pipeline, [[aws-s3]], [[aws-ecr]] |
| Regional concurrency quota | A support ticket, done once, per Region, per account |
| Cold-start burst at failover | Provisioned concurrency spend, or accepted latency |
| Region-specific ARNs in configuration | [[module-patterns]] — parameterise, don't hardcode |
| Event source mapping state | [[failover-orchestration]] runbook step |
| VPC attachment | Pre-provisioned ENIs, [[aws-vpc-networking]] |

---

## Replication / mirroring options

### Option 0 — "Do nothing, deploy a second copy." This is the answer.

State it plainly so nobody spends a sprint looking for something cleverer. For a
stateless compute primitive, the mirroring mechanism is your existing deployment
pipeline pointed at a second Region. The function in `eu-west-2` is not a copy of
the function in `eu-west-1` in any AWS-visible sense; it is a separate function
built from the same source. That is exactly what you want, because it means:

- No replication lag to reason about.
- No RPO contribution whatsoever.
- No promotion, no failback data reconciliation.
- Drift is detectable by `terraform plan`, which is the only drift detector you
  actually trust.

The rest of this note is about the four things that make Option 0 non-trivial in
practice: **artifacts, concurrency, ARNs, and event sources.**

### The deployment-artifact problem

This is the single most common cause of a standby Lambda deployment failing at
the worst possible moment.

#### `.zip` from S3

From [Troubleshoot deployment issues in
Lambda](https://docs.aws.amazon.com/lambda/latest/dg/troubleshooting-deployment.html):
when you upload a function's deployment package from an S3 bucket, **the bucket
must be in the same Region as the function**. If it is not, `CreateFunction` /
`UpdateFunctionCode` fails with `PermanentRedirect`. The bucket may be in a
different *account*.

So the estate needs an artifact bucket per Region:

```
helios-artifacts-eu-west-1
helios-artifacts-eu-west-2
helios-artifacts-us-east-1
helios-artifacts-us-west-2
helios-artifacts-ca-central-1
helios-artifacts-ca-west-1
```

Two ways to populate the standby bucket:

| Approach | Mechanism | Failure mode |
|---|---|---|
| **CI pushes to both** | The build job does two `aws s3 cp` calls | A CI change, a new pipeline, or a hotfix deployed by hand skips one Region and the standby silently goes stale |
| **S3 Cross-Region Replication** | `aws_s3_bucket_replication_configuration` on the primary artifact bucket | **CRR is not retroactive.** Objects that existed before the rule was created are never replicated unless you run S3 Batch Replication. Also: replication is asynchronous with no SLA-bounded lag on the standard tier, so a deploy-then-immediately-fail-over sequence can miss the newest artifact |

**Recommendation: CI pushes to both, and make it a single build-system
primitive** (one `publish_artifact` step that loops over the region list from the
cookiecutter config), not two copy-pasted steps that can diverge. Then add CRR
*as well*, purely as a backstop, with the explicit understanding that it does not
backfill. If you turn CRR on for the artifact bucket, run a one-off S3 Batch
Replication job to seed history, or accept that only artifacts published after
switch-on exist in the standby. See [[aws-s3]].

**A subtlety that bites in Terraform:** if `aws_lambda_function` references
`s3_key = "orders/${var.release_sha}.zip"`, then `terraform apply` in the standby
Region fails hard if the object is not there yet. This is *good* — a loud failure
at deploy time is infinitely better than a quiet one at failover time. Do not
"fix" it by adding `lifecycle { ignore_changes = [s3_key] }` unless you have
thought hard about the consequence, which is that Terraform will stop noticing
that the standby is running last quarter's code.

#### Container images from ECR

Same rule, different service: **the ECR repository must be in the same Region as
the Lambda function** ([Create a Lambda function using a container
image](https://docs.aws.amazon.com/lambda/latest/dg/images-create.html)).
Cross-account is supported (and has been since 2021, with GovCloud and China
following later); cross-region is not.

ECR has a native `aws_ecr_replication_configuration` that mirrors pushes to other
Regions — and it carries **exactly the same not-retroactive trap as S3 CRR**. The
[ECR replication
docs](https://docs.aws.amazon.com/AmazonECR/latest/userguide/replication.html)
and the long-running feature request [containers-roadmap
#1289](https://github.com/aws/containers-roadmap/issues/1289) confirm that only
images pushed *after* the rule exists are replicated. Enabling replication on a
five-year-old repository gives you an empty destination repository.

There is a second, sharper trap specific to Lambda + ECR: **Lambda resolves
`image_uri` to an image digest at function-create/update time and pins it.** If
your standby function is created with a tag that has not yet replicated, it fails
to create. And if you deploy by mutable tag (`:latest`, `:prod`), the digest the
standby pinned can differ from the primary's. Deploy by immutable digest or by
immutable SHA tag, and set `image_tag_mutability = "IMMUTABLE"` on the
repository. See [[aws-ecr]].

#### Which packaging should the standby use?

| | `.zip` in S3 | Container image in ECR |
|---|---|---|
| Same-Region constraint | Yes (bucket) | Yes (repository) |
| Native cross-region replication | S3 CRR | ECR replication config |
| Retroactive? | **No** (Batch Replication needed) | **No** (re-push needed) |
| Size ceiling | 50 MB zipped via API, 250 MB unzipped incl. layers | 10 GB uncompressed |
| Code-storage quota | Counts against the 300 GB per-Region function+layer storage quota | Counts against [ECR quotas](https://docs.aws.amazon.com/AmazonECR/latest/userguide/service-quotas.html) instead |
| SnapStart | Supported | Supported (since July 2026) |
| Lambda@Edge | Supported | **Not supported** |

Figures from [Lambda
quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html).

**Recommendation: don't change packaging as part of the multi-region work.**
Whatever the estate uses today, the multi-region delta is the same shape — a
per-Region artifact store plus a CI step that writes to both. Migrating `.zip` →
`Image` is a `package_type` change, and `package_type` is a replacement-forcing
attribute in the AWS provider (see the Terraform section). Doing that *while*
standing up a standby Region is two risky changes at once.

### The one genuine AWS-native replication: Lambda@Edge

From [Restrictions on
Lambda@Edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-at-edge-function-restrictions.html):

> **The Lambda function must be in the US East (N. Virginia) Region.**

and

> You must use a numbered version of the Lambda function, not `$LATEST` or
> aliases.

CloudFront then replicates that `us-east-1`-authored version to edge locations
via the `AWSServiceRoleForLambdaReplicator` service-linked role
([permissions docs](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-edge-permissions.html)).

This matters to Helios in three ways:

1. **It puts `us-east-1` on the critical path for every Region's edge logic,
   including the EU pair and the CA pair.** [[aws-regional-outages]] is the note
   on why that is a problem: six of the six AWS post-event summaries it catalogues
   were `us-east-1` events, including [the June 2023 Lambda
   event](https://aws.amazon.com/message/061323/). If `us-east-1` Lambda is
   impaired you cannot *deploy or update* a Lambda@Edge function — already-replicated
   versions keep running at the edge, but you have lost the ability to change them.
2. **Lambda@Edge cannot do most of what an application function does.** No
   environment variables (except the reserved ones), no VPC attachment, no
   layers, no container images, no provisioned concurrency, no X-Ray, no dead
   letter queues, no arm64, no ephemeral storage above 512 MB, no
   customer-managed-key-encrypted `.zip`. Only the latest Node.js and Python
   runtimes. That list is from the AWS restrictions page above. **Anything with
   region-specific config in it cannot be a Lambda@Edge function**, which
   conveniently means Lambda@Edge is rarely the thing that breaks a failover.
3. **The Lambda@Edge service-linked roles are supported in a fixed Region list
   that does not include Canada.** The AWS permissions page lists `us-east-1`,
   `us-east-2`, `us-west-1`, `us-west-2`, `ap-south-1`, `ap-northeast-2`,
   `ap-southeast-1`, `ap-southeast-2`, `ap-northeast-1`, `eu-central-1`,
   `eu-west-1`, `eu-west-2`, `sa-east-1`. **Neither `ca-central-1` nor
   `ca-west-1` appears.** Since Lambda@Edge functions live in `us-east-1`
   regardless, this is mostly about where the SLR can be used from, but it is a
   real CA-pair data point — record it in [[region-pair-selection]]. If the
   Canadian deployment has edge logic and a data-residency constraint,
   Lambda@Edge is the wrong tool and **CloudFront Functions** (which run at the
   edge with no `us-east-1` function resource) or origin-side logic is the
   answer. See [[aws-cloudfront]].

**Recommendation for Helios: prefer CloudFront Functions over Lambda@Edge for
anything that can be expressed as header/URI manipulation**, and keep Lambda@Edge
for the cases that genuinely need a full runtime. Fewer `us-east-1`
dependencies is the whole game.

---

## RPO / RTO analysis

### RPO: N/A, and say so

Lambda holds no durable state. `/tmp` is ephemeral (512 MB–10,240 MB, scoped to
an execution environment). The execution environment itself is destroyed at will.
**Lambda contributes exactly zero to the 2-hour RPO.**

What *does* contribute is the work in flight at the moment the region dies:

- A synchronous invocation in progress — the caller gets an error and retries or
  fails. Not RPO, that's availability.
- An asynchronous invocation sitting in Lambda's internal queue — Lambda's
  internal async queue is regional and opaque. Events queued for a function in
  the dead Region are not visible to you and are not recoverable. Their fate is
  the same argument [[messaging-in-flight-data-loss]] makes for SQS: they are
  *pending work*, not data, unless something outside the system was told they
  were accepted. Configure **on-failure destinations** on async invocations and
  point them at a durable store, and the question mostly goes away for the
  non-catastrophic cases.
- A batch pulled from an ESM but not yet deleted from the source — the messages
  become visible again after the visibility timeout. They are still in the
  *primary* queue, which is in the dead Region. That is an SQS problem, not a
  Lambda problem: see [[aws-sqs]].

### RTO: the function is instant, the capacity is not

Decompose the 15 minutes.

| Step | Time | Pre-provisioned? |
|---|---|---|
| Standby function exists and is `Active` | 0 s | **Yes — must be** |
| DNS/traffic shifts to the standby | See [[aws-route53]] | n/a |
| First invocations arrive, cold starts begin | 0 s | n/a |
| Lambda scales to N concurrency on demand | `ceil(N / 1000) × 10 s` | No |
| VPC-attached function acquires a Hyperplane ENI | 0 s if warm, **"several minutes"** if cold | **Yes — must be** |
| Provisioned concurrency allocated at failover time | "a minute or two of preparation" + `N / 6000` min | Only if you choose to |
| Regional concurrency quota raised | **Hours to days — a support ticket** | **Yes — must be, absolutely** |

Three of those rows are the whole story.

#### 1. The concurrency quota is the RTO killer

AWS is explicit ([Lambda
quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)):

> Quotas for concurrent executions and storage apply per AWS Region.

Default is **1,000 concurrent executions**, increasable "to tens of thousands".
The Service Quotas console is per-Region; a quota increase granted in `eu-west-1`
does **not** propagate to `eu-west-2`. A mature estate that has been running in
`us-east-1` for years has almost certainly had this raised at some point, quite
possibly by someone who has left.

There is a second, coupled limit that is easy to miss: **Lambda enforces a
requests-per-second ceiling equal to 10× your concurrency quota.** At the default
1,000 concurrency, the account cannot exceed 10,000 requests/second in that
Region regardless of how short the functions are. A fleet of 30 ms functions
doing 20,000 rps needs a concurrency quota of at least 2,000 *even though its
measured concurrency is only 600*. From [Understanding Lambda function
scaling](https://docs.aws.amazon.com/lambda/latest/dg/lambda-concurrency.html):

> Across all functions in your account, Lambda enforces a requests per second
> limit that's equal to 10 times your account concurrency.

**Action, and it is a cheap one:** for each pair, run
`aws lambda get-account-settings --region <primary>` and
`--region <standby>` and compare `ConcurrentExecutions`. If they differ, raise a
Service Quotas request for the standby to match *today*. There is no charge for a
higher quota — you pay for what you use — so there is no reason to under-ask.
Then wire an alarm.

Also note the warning at the top of the quotas page: *"New AWS accounts have
reduced concurrency and memory quotas for Lambda Functions"* and AWS raises them
automatically based on usage. A standby Region with near-zero usage will not get
the automatic raises the primary got.

#### 2. On-demand scaling is fast enough for most workloads — do the arithmetic before buying provisioned concurrency

From the same page:

> In each AWS Region, and for each function, your concurrency scaling rate is
> 1,000 execution environment instances every 10 seconds (or 10,000 requests per
> second every 10 seconds).

Critically, **this is a per-function limit**, so 40 functions scale in parallel,
each at 1,000/10 s. For a typical microservice fleet the aggregate ramp is not
the constraint; the per-function ramp is.

Worked example against the 15-minute RTO:

| Peak concurrency needed for the busiest function | Time to reach it from zero | Verdict vs RTO 15m |
|---|---|---|
| 200 | < 10 s | Trivially fine |
| 1,000 | ~10 s | Fine |
| 3,000 | ~30 s | Fine |
| 10,000 | ~100 s | Fine on RTO; needs a quota increase to 10,000+ |

**The scaling rate is not what fails the RTO.** What fails is (a) the quota, and
(b) the *latency* of those cold starts — a Java function with a 4-second init
that cold-starts 3,000 times in 30 seconds produces 30 seconds of very ugly p99
and a wave of client-side timeouts that looks, from the outside, like the
failover didn't work.

So the real question is not "can we scale in time" but **"is a 30–90 second
latency spike at the moment of failover acceptable?"** For an internal batch
consumer: obviously yes. For the checkout API: probably not.

#### 3. Provisioned concurrency: what it actually costs to pre-warm the standby

Prices below are **US East (N. Virginia), x86**, taken from the [AWS Lambda
pricing page](https://aws.amazon.com/lambda/pricing/):

| Item | Price |
|---|---|
| On-demand duration | $0.0000166667 per GB-second |
| Requests | $0.20 per 1M |
| **Provisioned concurrency (the reservation)** | **$0.0000041667 per GB-second** |
| Duration while provisioned concurrency is enabled | $0.0000097222 per GB-second |
| SnapStart cache | $0.0000015046 per GB-second |
| SnapStart restore | $0.0001397998 per GB |

> **Verification caveat, stated honestly:** the pricing page's regional selector
> is client-side and the fetched page only exposed the `us-east-1` figures. I did
> **not** verify the `eu-west-1`, `eu-west-2`, `ca-central-1`, `ca-west-1` or
> `us-west-2` rates and I am not going to guess them. AWS Lambda pricing does
> vary by Region. Before committing a budget, pull the real per-Region numbers
> from the AWS Price List API
> (`aws pricing get-products --service-code AWSLambda --region us-east-1
> --filters ...`) or the pricing page's Region selector. Do not extrapolate from
> the numbers above.

The idle cost of a warm standby, at `us-east-1` rates, for an *unused*
provisioned concurrency reservation:

```
cost/month = PC_units × memory_GB × 0.0000041667 × 730 × 3600
           = PC_units × memory_GB × $10.95
```

| Reservation | Memory | Idle cost/month (us-east-1 rate) |
|---|---|---|
| 10 units | 512 MB | ~$55 |
| 100 units | 512 MB | ~$548 |
| 100 units | 1,024 MB | ~$1,095 |
| 500 units | 1,024 MB | ~$5,475 |

That is per function. Multiply by the number of critical-path functions and it
becomes the largest single line in the Lambda standby bill by a wide margin.

**The allocation-speed question — can you buy it only at failover?** From the AWS
docs:

> Provisioned concurrency doesn't come online immediately after you configure it.
> Lambda starts allocating provisioned concurrency after a minute or two of
> preparation. For each function, Lambda can provision up to 6,000 execution
> environments every minute […] When you submit a request to allocate provisioned
> concurrency, you can't access any of those environments until Lambda completely
> finishes allocating them.

So "allocate PC as step 3 of the runbook" costs roughly **1–2 minutes of
preparation plus `N/6000` minutes of allocation, and gives you nothing until it
completes**. For N = 500 that is ~2 minutes total. That *fits* inside a
15-minute RTO — but it consumes 15% of the budget, it is a control-plane
operation in a Region that may itself be having a bad day, and it is
all-or-nothing. Note also that 6,000/minute is exactly the same as the on-demand
scaling rate of 1,000 per 10 seconds; **allocating provisioned concurrency at
failover time is not faster than just letting Lambda scale on demand.** Its only
advantage is that the environments are initialised *before* the first request
rather than during it.

That gives three genuine branches:

| Branch | What you do | Idle cost | Failover latency | Verdict |
|---|---|---|---|---|
| **A — Nothing** | Standby function exists, zero PC | $0 | 10–90 s of cold starts, p99 spike | Right for async, batch, ESM consumers, anything not user-facing |
| **B — Standing PC in the standby** | PC allocated permanently in the standby at (say) 20% of primary peak | Full price, 24×7, for capacity that serves nothing | ~0 for the first 20% of traffic, on-demand ramp for the rest | Right for a small set of critical synchronous functions |
| **C — Scheduled/triggered PC** | PC set to 0 normally; an Application Auto Scaling scheduled action or the failover runbook raises it | ~$0 | +1–2 min prep, then ~N/6000 min | Attractive on paper, buys little over Branch A, adds a control-plane dependency at the worst moment |

**Recommendation: Branch A by default; Branch B for a named, short list of
user-facing synchronous functions; do not bother with Branch C.** Branch C's
pitch is "pay nothing, get warm capacity at failover", but since allocation is no
faster than on-demand scaling, all you actually buy is *pre-initialisation* —
and you buy it with a mandatory 1–2 minute wait and an extra API call that has to
succeed during an incident. If the init cost is what hurts, the correct fix is
**SnapStart**, not scheduled PC.

The short list for Branch B should be derived, not guessed: take the functions
that sit behind [[aws-api-gateway]] or the [[aws-alb-nlb]] target groups, sort by
p99 init duration (the `InitDuration` CloudWatch metric), and take the ones over
~500 ms. Everything else can cold-start.

#### SnapStart as the cheaper answer to init cost

[SnapStart](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html) takes a
Firecracker microVM snapshot of the initialised execution environment when you
publish a version, and resumes from it instead of re-initialising. Verified facts:

- **Runtimes:** Java 11+, Python 3.12+, .NET 8+, across both `.zip` and container
  image packaging. Node.js and Ruby are **not** supported. OS-only (`provided`)
  runtimes are not supported.
- **Regions:** "available in all commercial Regions except Asia Pacific (New
  Zealand) and Asia Pacific (Taipei)". **That includes `ca-west-1`** — a rare
  clean parity result for Calgary, and worth recording in
  [[region-pair-selection]].
- **Mutually exclusive with provisioned concurrency.** You cannot have both on
  the same function version.
- **Versions/aliases only.** Not available on `$LATEST`.
- **Not compatible with** EFS, or ephemeral storage above 512 MB.
- **Pricing:** no additional charge for Java managed runtimes. For others you pay
  a cache charge per published version (minimum 3 hours) plus a per-restore
  charge.
- **The uniqueness trap:** the snapshot is reused across execution environments,
  so anything unique generated during init — UUIDs, seeded PRNGs, cached
  credentials, open connections — is *shared*. AWS calls this out explicitly and
  has a [dedicated
  page](https://docs.aws.amazon.com/lambda/latest/dg/snapstart-uniqueness.html)
  on it. This is an application-code review, not a Terraform change.

**Multi-region relevance:** SnapStart snapshots are per-Region and per-version.
Publishing a version in the standby Region creates and caches a snapshot there,
which means **the standby accrues a per-version SnapStart cache charge even
though it serves no traffic** — small, but it is one of the few Lambda line items
that is non-zero at idle. It also means the standby's snapshots are warm and
ready, which is exactly what you want. A published version with SnapStart that
sits unused still gets patched by AWS, and "charges apply each time that Lambda
re-runs your initialization code to apply software updates."

**Recommendation: if the estate is Java or Python 3.12+, enable SnapStart on the
critical-path functions in both Regions and skip provisioned concurrency
entirely.** It is dramatically cheaper and it solves the failover cold-start
problem at the point where it actually hurts (init), for zero incremental
failover-time work. If the estate is Node.js, SnapStart is not available and you
are back to Branch A / Branch B above.

---

## Warm standby shape

What exists in `eu-west-2` while `eu-west-1` is serving:

| Resource | State in standby | Costs at idle? |
|---|---|---|
| `aws_lambda_function` (all of them) | Created, `Active` | No — Lambda has no per-function idle charge |
| Published versions + `prod` alias | Created | No (except SnapStart cache, see above) |
| Function code in S3/ECR | Present, current | S3/ECR storage only, pennies |
| CloudWatch log groups | Created, empty, retention set | No |
| IAM execution roles | Created (global — see [[aws-iam]]) | No |
| VPC attachment + Hyperplane ENI | **Must be warm** — see below | ENI itself is free; the NAT/subnet infra behind it is not ([[aws-vpc-networking]]) |
| Event source mappings | Created with `enabled = false` | No |
| Provisioned concurrency | Zero, or a small Branch-B reservation | **Yes, and this is the only material line** |
| Account concurrency quota | Raised to match primary | No |
| Function URLs | Created if used | No |

**Lambda's warm standby is essentially free.** That is unusual in this vault and
worth saying out loud: compared to [[aws-rds-postgres]] (a running replica
instance), [[aws-eks]] (a control plane plus nodes), or [[aws-elasticache-redis]],
Lambda's idle cost is rounding-error unless you choose provisioned concurrency.
This makes Lambda the *easiest* service in the estate to make multi-region and a
good early win for the [[sequencing-roadmap]].

### The one thing that genuinely must be pre-warmed: VPC attachment

From [Giving Lambda functions access to resources in an Amazon
VPC](https://docs.aws.amazon.com/lambda/latest/dg/foundation-networking.html):

> The first time you attach a function to a VPC using a particular subnet and
> security group combination, Lambda creates a Hyperplane ENI. […] For new
> functions, while Lambda is creating a Hyperplane ENI, your function remains in
> the Pending state and you can't invoke it. Your function transitions to the
> Active state only when the Hyperplane ENI is ready, **which can take several
> minutes.**

and, the one that actually bites a warm standby:

> if a Lambda function remains idle for **14 days**, Lambda reclaims any unused
> Hyperplane ENIs and sets the function state to `Inactive`. The next invocation
> attempt will fail, and the function re-enters the Pending state until Lambda
> completes the creation or allocation of a Hyperplane ENI.

**Read that again in the context of a warm standby.** A VPC-attached function in
the standby Region that receives no traffic for 14 days goes `Inactive`. At
failover, the *first invocation fails outright* and the function sits in
`Pending` for "several minutes" while a Hyperplane ENI is provisioned. That is a
multi-minute hole punched in a 15-minute RTO, and it fires only after two weeks
of the standby being healthy and ignored — i.e. exactly when you have stopped
thinking about it.

Mitigations, in order of preference:

1. **Keep at least one VPC-attached function per (subnet, security group)
   combination invoked on a schedule.** A single EventBridge Scheduler rule
   firing a trivial health-check function every hour keeps the ENI for that
   subnet+SG combination alive, and — per the AWS docs — *other* functions using
   the same subnet+SG combination share that ENI. One canary per network
   configuration, not one per function. This is the cheapest possible insurance
   and it doubles as a standby smoke test. See [[failover-orchestration]].
2. **Standardise the subnet+SG combination across functions** so there are two or
   three combinations in the Region, not thirty. Fewer ENIs to keep warm, fewer
   to hit the 500-per-VPC ENI quota with (shared with EFS and others — see [VPC
   quotas](https://docs.aws.amazon.com/vpc/latest/userguide/amazon-vpc-limits.html)).
3. **Don't attach to a VPC at all if you don't need to.** A function that only
   talks to DynamoDB, S3, SQS and Secrets Manager does not need VPC attachment;
   it needs an IAM role. Detaching removes a whole class of failover risk. (It
   also takes "up to 20 minutes" for Lambda to delete the ENI afterwards, so this
   is a change to make calmly, not during a migration.)

Note also the interaction with the 14-day rule and **canary invocations of the
standby generally**: a standby that has never been invoked has never had its
execution role exercised, its Secrets Manager replica read, its DynamoDB Global
Table replica written, or its VPC route to RDS tested. The hourly canary is worth
far more than its cost.

---

## Terraform implementation

### Shape: one region-agnostic module, instantiated per Region

This fits a cookiecutter-templated monorepo cleanly. The module knows nothing
about "primary" or "standby"; it takes a provider and a set of region-scoped
inputs. See [[provider-aliases-vs-separate-stacks]] for the broader decision —
the short version for Lambda is that **either approach works**, because there are
no cross-region resource references to resolve. Lambda is one of the few services
where separate stacks cost you nothing.

```hcl
# ---- modules/lambda-function/versions.tf ----
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.60, < 7.0"
    }
  }
}
```

```hcl
# ---- modules/lambda-function/variables.tf ----

variable "name" {
  description = "Function name, unqualified. Region is NOT part of the name."
  type        = string
}

variable "role_arn" {
  description = "Execution role ARN. IAM is global, so this is the same string in both regions."
  type        = string
}

# --- artifact -------------------------------------------------------------
variable "artifact" {
  description = <<-EOT
    Where the code comes from. Exactly one of the two shapes:
      { type = "zip",   bucket = "...", key = "...", version_id = optional }
      { type = "image", image_uri = "<acct>.dkr.ecr.<region>.amazonaws.com/repo@sha256:..." }
    The caller is responsible for passing a bucket / repo IN THIS MODULE'S REGION.
  EOT
  type        = any
}

# --- runtime --------------------------------------------------------------
variable "runtime"       { type = string, default = null }   # null for image
variable "handler"       { type = string, default = null }
variable "memory_mb"     { type = number, default = 512 }
variable "timeout_s"     { type = number, default = 30 }
variable "architectures" { type = list(string), default = ["arm64"] }

variable "snap_start" {
  description = "Enable SnapStart. Java 11+/Python 3.12+/.NET 8+ only. Mutually exclusive with provisioned_concurrency."
  type        = bool
  default     = false
}

# --- region-scoped config -------------------------------------------------
variable "environment" {
  description = <<-EOT
    Environment variables. MUST NOT contain a hardcoded ARN or region string.
    Region-specific values are injected by the caller from that region's locals.
    Total size across all vars is capped at 4 KB by Lambda.
  EOT
  type        = map(string)
  default     = {}
}

variable "kms_key_arn" {
  description = "CMK for environment-variable encryption. Regional key or the regional alias of a multi-region key — see kms-when-to-use-multi-region-keys."
  type        = string
  default     = null
}

variable "vpc_config" {
  description = "null to run outside a VPC. Subnets/SGs are region-specific IDs."
  type = object({
    subnet_ids         = list(string)
    security_group_ids = list(string)
  })
  default = null
}

# --- the multi-region switches -------------------------------------------
variable "is_active_region" {
  description = <<-EOT
    True in the region currently serving traffic. Controls the DEFAULT state of
    things that must not run in two places at once (event source mappings,
    scheduled invocations). Deliberately NOT called "is_primary" — at failover
    this flips, and calling it "primary" invites people to treat it as static.
  EOT
  type        = bool
}

variable "provisioned_concurrency" {
  description = "Units of provisioned concurrency on the live alias. 0 = none. Costs money 24x7."
  type        = number
  default     = 0
}

variable "reserved_concurrency" {
  description = "Reserved (max AND min) concurrency. -1 = unset. Counts against the regional account quota."
  type        = number
  default     = -1
}

variable "event_source_mappings" {
  description = <<-EOT
    Map of ESM name => config. event_source_arn MUST be an ARN in this module's
    region — Lambda does not support cross-region event source mappings.
  EOT
  type = map(object({
    event_source_arn                   = string
    batch_size                         = optional(number, 10)
    maximum_batching_window_in_seconds = optional(number, 0)
    function_response_types            = optional(list(string), ["ReportBatchItemFailures"])
    scaling_config_max_concurrency     = optional(number, null)
  }))
  default = {}
}

variable "log_retention_days" { type = number, default = 30 }
variable "tags"               { type = map(string), default = {} }
```

```hcl
# ---- modules/lambda-function/main.tf ----

locals {
  is_zip = try(var.artifact.type, "zip") == "zip"
}

resource "aws_lambda_function" "this" {
  function_name = var.name
  role          = var.role_arn
  memory_size   = var.memory_mb
  timeout       = var.timeout_s
  architectures = var.architectures

  package_type = local.is_zip ? "Zip" : "Image"

  # --- zip path: bucket MUST be in this region -------------------------
  s3_bucket         = local.is_zip ? var.artifact.bucket : null
  s3_key            = local.is_zip ? var.artifact.key : null
  s3_object_version = local.is_zip ? try(var.artifact.version_id, null) : null
  runtime           = local.is_zip ? var.runtime : null
  handler           = local.is_zip ? var.handler : null

  # --- image path: repository MUST be in this region -------------------
  image_uri = local.is_zip ? null : var.artifact.image_uri

  # Publish a version on every change. Required for aliases, provisioned
  # concurrency and SnapStart.
  publish = true

  kms_key_arn = var.kms_key_arn

  dynamic "environment" {
    for_each = length(var.environment) > 0 ? [1] : []
    content {
      variables = var.environment
    }
  }

  dynamic "vpc_config" {
    for_each = var.vpc_config == null ? [] : [var.vpc_config]
    content {
      subnet_ids         = vpc_config.value.subnet_ids
      security_group_ids = vpc_config.value.security_group_ids
    }
  }

  dynamic "snap_start" {
    for_each = var.snap_start ? [1] : []
    content {
      apply_on = "PublishedVersions"
    }
  }

  # Lambda's own CW log group is created implicitly with no retention; we
  # create it explicitly below, so make ordering deterministic.
  depends_on = [aws_cloudwatch_log_group.this]

  tags = var.tags
}

resource "aws_cloudwatch_log_group" "this" {
  name              = "/aws/lambda/${var.name}"
  retention_in_days = var.log_retention_days
  tags              = var.tags
}

resource "aws_lambda_alias" "live" {
  name             = "live"
  function_name    = aws_lambda_function.this.function_name
  function_version = aws_lambda_function.this.version
}

resource "aws_lambda_function_event_invoke_config" "this" {
  function_name          = aws_lambda_alias.live.function_name
  qualifier              = aws_lambda_alias.live.name
  maximum_retry_attempts = 2
  # on-failure destination lives in THIS region; see the SNS/SQS notes
}

resource "aws_lambda_provisioned_concurrency_config" "this" {
  count = var.provisioned_concurrency > 0 ? 1 : 0

  function_name                     = aws_lambda_alias.live.function_name
  qualifier                         = aws_lambda_alias.live.name
  provisioned_concurrent_executions = var.provisioned_concurrency
}

# --- event source mappings -------------------------------------------------
# The single most important line in this module is `enabled`.
resource "aws_lambda_event_source_mapping" "this" {
  for_each = var.event_source_mappings

  function_name    = aws_lambda_alias.live.arn
  event_source_arn = each.value.event_source_arn

  enabled = var.is_active_region

  batch_size                         = each.value.batch_size
  maximum_batching_window_in_seconds = each.value.maximum_batching_window_in_seconds
  function_response_types            = each.value.function_response_types

  dynamic "scaling_config" {
    for_each = each.value.scaling_config_max_concurrency == null ? [] : [1]
    content {
      maximum_concurrency = each.value.scaling_config_max_concurrency
    }
  }

  lifecycle {
    # The failover runbook (or ARC Region Switch) flips `enabled` out of band.
    # Without this, the next `terraform apply` silently re-disables the standby
    # consumers you just failed over to. See failover-orchestration.
    ignore_changes = [enabled]
  }
}

resource "aws_lambda_function_concurrency" "this" {
  count                          = var.reserved_concurrency >= 0 ? 1 : 0
  function_name                  = aws_lambda_function.this.function_name
  reserved_concurrent_executions = var.reserved_concurrency
}
```

> `aws_lambda_function_concurrency` is shown for clarity; older provider versions
> express reserved concurrency as the `reserved_concurrent_executions` argument
> on `aws_lambda_function` itself. Check the version pinned in the monorepo
> before copying — this is the sort of thing that changed across provider major
> versions.

### Instantiating it for a pair

```hcl
# ---- environments/prod-eu/main.tf ----

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

locals {
  # Everything region-specific in ONE place per region. This is the discipline
  # that makes the whole thing work.
  regional = {
    primary = {
      region            = "eu-west-1"
      artifact_bucket   = "helios-artifacts-eu-west-1"
      orders_queue_arn  = "arn:aws:sqs:eu-west-1:111122223333:orders"
      events_bus_arn    = "arn:aws:events:eu-west-1:111122223333:event-bus/helios"
      kms_key_arn       = "arn:aws:kms:eu-west-1:111122223333:alias/helios-app"
      subnet_ids        = ["subnet-0aaa", "subnet-0bbb", "subnet-0ccc"]
      security_group_ids = ["sg-0aaa"]
    }
    standby = {
      region            = "eu-west-2"
      artifact_bucket   = "helios-artifacts-eu-west-2"
      orders_queue_arn  = "arn:aws:sqs:eu-west-2:111122223333:orders"
      events_bus_arn    = "arn:aws:events:eu-west-2:111122223333:event-bus/helios"
      kms_key_arn       = "arn:aws:kms:eu-west-2:111122223333:alias/helios-app"
      subnet_ids        = ["subnet-0ddd", "subnet-0eee", "subnet-0fff"]
      security_group_ids = ["sg-0ddd"]
    }
  }

  # Which region is live. Flipping this is a Terraform-mediated failover; the
  # runbook flips ESMs out of band FIRST and this follows later. See
  # failover-orchestration for why you must not make the 15-minute RTO depend
  # on a terraform apply.
  active = "primary"
}

module "orders_api_primary" {
  source    = "../../modules/lambda-function"
  providers = { aws = aws.primary }

  name     = "helios-orders-api"
  role_arn = aws_iam_role.orders_api.arn   # IAM is global: same ARN both sides

  artifact = {
    type       = "zip"
    bucket     = local.regional.primary.artifact_bucket
    key        = "orders-api/${var.release_sha}.zip"
  }
  runtime = "java21"
  handler = "com.helios.orders.Handler::handleRequest"

  snap_start = true          # Java: free, and removes the failover cold-start problem

  environment = {
    ORDERS_QUEUE_URL = local.regional.primary.orders_queue_arn
    EVENT_BUS_ARN    = local.regional.primary.events_bus_arn
    DEPLOY_REGION    = local.regional.primary.region
  }

  kms_key_arn = local.regional.primary.kms_key_arn
  vpc_config = {
    subnet_ids         = local.regional.primary.subnet_ids
    security_group_ids = local.regional.primary.security_group_ids
  }

  event_source_mappings = {
    orders = { event_source_arn = local.regional.primary.orders_queue_arn }
  }

  is_active_region        = local.active == "primary"
  provisioned_concurrency = 0     # SnapStart instead
  tags                    = local.common_tags
}

module "orders_api_standby" {
  source    = "../../modules/lambda-function"
  providers = { aws = aws.standby }

  name     = "helios-orders-api"                     # SAME NAME. Not -standby.
  role_arn = aws_iam_role.orders_api.arn

  artifact = {
    type   = "zip"
    bucket = local.regional.standby.artifact_bucket   # DIFFERENT BUCKET
    key    = "orders-api/${var.release_sha}.zip"      # same key
  }
  runtime = "java21"
  handler = "com.helios.orders.Handler::handleRequest"

  snap_start = true

  environment = {
    ORDERS_QUEUE_URL = local.regional.standby.orders_queue_arn
    EVENT_BUS_ARN    = local.regional.standby.events_bus_arn
    DEPLOY_REGION    = local.regional.standby.region
  }

  kms_key_arn = local.regional.standby.kms_key_arn
  vpc_config = {
    subnet_ids         = local.regional.standby.subnet_ids
    security_group_ids = local.regional.standby.security_group_ids
  }

  event_source_mappings = {
    orders = { event_source_arn = local.regional.standby.orders_queue_arn }
  }

  is_active_region        = local.active == "standby"   # => ESM disabled today
  provisioned_concurrency = 0
  tags                    = local.common_tags
}
```

**Keep the function name identical across Regions.** The ARN differs because the
Region does; adding `-standby` to the name means every alarm, dashboard, log
query, IAM policy and runbook has to special-case the standby. This is the same
argument [[dynamodb-table-naming-migration]] makes for DynamoDB and it applies
here for softer reasons — Lambda does not *force* it the way Global Tables do,
but you will regret the asymmetry. Note in passing that Lambda names are unique
per-Region per-account, so there is no collision.

### The cookiecutter angle

In a templated monorepo the above collapses to a `for_each` over a region map
supplied by the template, which is the shape [[module-patterns]] argues for:

```hcl
module "orders_api" {
  source   = "../../modules/lambda-function"
  for_each = local.regional
  providers = {
    aws = each.key == "primary" ? aws.primary : aws.standby   # NOT VALID — see note
  }
  # ...
}
```

**That does not work**: `providers` cannot be computed, and `for_each` over
providers is not supported in Terraform. This is the single biggest ergonomic
tax of multi-region Terraform and it is why [[provider-aliases-vs-separate-stacks]]
exists. The two real options are (a) explicit paired module blocks as shown
above, generated by cookiecutter so the duplication is in the template not in the
repo, or (b) separate root modules per Region with a shared varfile. For Lambda
specifically, (b) is unusually attractive because there is genuinely nothing to
correlate across the two Regions at plan time.

### Fields to be careful with

| Field | Behaviour | Why it matters here |
|---|---|---|
| `function_name` | Changing it replaces the function | Do not "rename to add -standby" on a live function |
| `package_type` | `Zip` ↔ `Image` forces replacement (see [hashicorp/terraform-provider-aws#29653](https://github.com/hashicorp/terraform-provider-aws/issues/29653)) | Don't re-platform packaging during the multi-region migration |
| `s3_bucket` / `image_uri` / `filename` | Mutually exclusive; switching between them is the `package_type` change above | — |
| `publish` | `false` means no versions, which means no aliases, no provisioned concurrency and no SnapStart | Turn it on before you need any of those |
| `snap_start.apply_on` | `PublishedVersions` or `None` | Toggling is an update, not a replacement |
| `enabled` on the ESM | An in-place update | The runbook flips this; `ignore_changes` or Terraform will flip it back |
| `vpc_config` | Adding/removing is an update, but triggers the ENI lifecycle (minutes to attach, up to 20 to detach) | Not a `terraform apply` to run under time pressure |

> **Verification note:** I attempted to confirm the exact "Forces new resource"
> annotations from the `aws_lambda_function` registry docs and the fetch returned
> no argument-reference content (the registry page is client-rendered). The
> `package_type` and `function_name` replacement behaviour above comes from
> provider issue threads, not from the argument reference. **Before relying on
> this, run `terraform plan` against a scratch function and read the plan.** That
> is a five-minute experiment and it is authoritative in a way a GitHub issue is
> not.

---

## Migration path from single-region

The good news: **there is no migration.** The primary function is untouched. You
are creating new resources in a new Region. Nothing in the standby's Terraform
can produce a destroy/recreate in the primary's plan, provided you follow one
rule: **do not restructure the existing module while adding the second Region.**

The temptation is enormous — the existing per-Region Lambda code will not have a
`providers` argument, will hardcode `eu-west-1` ARNs in `environment`, and will
look ripe for a tidy-up. Resist. Do it in two passes.

### Pass 1 — make the existing module region-agnostic, with zero diff

Refactor the module so region-specific values arrive as variables rather than
literals, and wire the primary's call site to pass exactly the values that are
hardcoded today. The acceptance criterion is literally **`terraform plan` shows
no changes**. If it shows a change to `environment.variables`, you have got a
value wrong — fix it before proceeding, because an environment-variable change on
a published-version function is an update to `$LATEST` and a new version, and
while that is harmless it muddies the signal.

Watch for:
- The implicit CloudWatch log group. If the existing setup lets Lambda create
  `/aws/lambda/<name>` implicitly and you add an explicit
  `aws_cloudwatch_log_group`, Terraform will try to *create* a group that already
  exists and fail with `ResourceAlreadyExistsException`. Import it:
  `terraform import module.orders_api.aws_cloudwatch_log_group.this /aws/lambda/helios-orders-api`.
- Adding `publish = true` to a function that did not have it. This creates a
  version. Harmless, but it starts the version-number sequence and, if SnapStart
  is also enabled, starts the snapshot cache charge.
- Adding an `aws_lambda_alias` where callers currently invoke the unqualified
  function. Invoking `arn:...:function:name` (no qualifier) always routes to
  `$LATEST`. Moving callers to `:live` is a change of behaviour — do it
  deliberately, and note that **Lambda@Edge cannot use an alias at all**.

### Pass 2 — add the standby

1. **Create the regional artifact store.** Bucket in [[aws-s3]], repository in
   [[aws-ecr]]. Do this first and independently; it is the long pole because of
   the not-retroactive problem.
2. **Teach CI to publish to both.** Verify by listing the standby bucket/repo
   after a deploy. Add a CI assertion, not a human check.
3. **Raise the standby's concurrency quota.** Support ticket. Days of lead time.
   Do it on day one of the workstream, not the week before the game day.
4. **Stand up the prerequisites the functions consume.** Per the brief's
   prerequisites-first strategy: [[aws-secrets-manager]] replicas (done),
   [[aws-dynamodb]] Global Tables (in progress), [[aws-sqs]] standby queues,
   [[aws-sns]] standby topics, [[aws-kms]] keys, [[aws-ssm-parameter-store]]
   values. A Lambda deployed before its dependencies exist will deploy fine and
   fail on first invocation, which is the worst of both worlds.
5. **Deploy the functions with `is_active_region = false`.** ESMs disabled,
   provisioned concurrency zero.
6. **Invoke them.** Manually, then on a schedule. This is the step everyone
   skips. A standby function that has never been invoked is a hypothesis, not a
   standby. The canary must exercise the real dependency graph: read the
   replicated secret, write the Global Table, publish to the standby topic.
7. **Add the hourly VPC canary** per the 14-day ENI rule above.
8. **Alarm on the deltas.** Standby function count vs primary. Standby code SHA
   vs primary. Standby concurrency quota vs primary. These three alarms catch
   ~90% of the ways a warm standby rots.

### What cannot be migrated without a replacement

| Thing | Why | Workaround |
|---|---|---|
| Function URL | Deleting a function URL and creating a new one yields **a different URL** (AWS: "When you delete a function URL, you can't recover it. Creating a new function URL results in a different URL address.") | Never delete; front it with [[aws-cloudfront]] or [[aws-api-gateway]] so the URL the world sees is yours, not `*.lambda-url.*.on.aws` |
| Lambda@Edge association | Requires a numbered version, no alias, `us-east-1` only | Plan the version bump; CloudFront distribution updates take time to propagate |
| `package_type` change | Forces replacement | Do it as a separate change, in the primary, before the standby exists |

---

## Failover procedure

Lambda's part of the runbook is short, which is the point. Ordering matters more
than content.

**Precondition (verified continuously, not at 3am):** standby functions `Active`,
current SHA, quota matched, ENIs warm.

1. **Fence the primary.** Before anything is enabled in the standby, stop the
   primary from processing. For Lambda this has a neat primitive: **set reserved
   concurrency to 0**. AWS documents this explicitly for function URLs — "To
   deactivate your function URL, set the reserved concurrency to zero. This
   throttles all requests to your function, resulting in HTTP 429 status
   responses" — and it works for any invocation path. It is instant, it is a
   single API call, and it is trivially reversible. This is the cleanest fencing
   mechanism in the estate; [[split-brain-and-fencing]] should reference it.
   Caveat: if the primary Region is the thing that is broken, this call may fail.
   Plan for the ungraceful path.
2. **Disable primary event source mappings** (if the Region is reachable).
   `aws lambda update-event-source-mapping --uuid ... --no-enabled`. This is what
   ARC Region Switch's *disable* block automates.
3. **Enable standby event source mappings.** ARC Region Switch's *enable* block.
   AWS's own framing of the problem matches this note's: "For active-passive
   workloads, customers may maintain Lambda functions in each Region but process
   events in only one Region at a time. These event source mappings must be
   toggled during failover to avoid duplicate processing — a manual, error-prone
   step."
4. **(Optional, Branch C only)** Allocate provisioned concurrency. Budget 1–2
   minutes of preparation plus allocation time; nothing is usable until it
   completes.
5. **Shift traffic.** [[aws-route53]] / [[aws-cloudfront]] / [[aws-api-gateway]].
   This is where the RTO is actually won or lost.
6. **Watch `Throttles`, `ConcurrentExecutions`, `InitDuration`, `Errors`.** The
   failure signature of an under-provisioned standby is a `Throttles` spike with
   flat `Invocations` — that is the concurrency quota, and at that point the only
   fix is a Service Quotas request, which does not complete in 15 minutes.

**What is automated vs. human:** steps 2 and 3 should be automated —
either as an ARC Region Switch plan ([Lambda event source mapping execution
block](https://docs.aws.amazon.com/r53recovery/latest/dg/lambda-event-source-mapping-block.html),
announced [May
2026](https://aws.amazon.com/about-aws/whats-new/2026/05/region-switch-lambda-esm-execution-block/))
or as a scripted step in whatever [[failover-orchestration]] settles on. Step 1
is a human decision. Step 5 is a human decision with an automated execution.

**Do not make the RTO depend on `terraform apply`.** Flipping
`local.active = "standby"` and applying is the *reconciliation* step, run after
the incident, not during it. That is why the ESM resource carries
`ignore_changes = [enabled]`.

---

## Failback

Easier than most services in this vault, because there is no data to move back.
The hard parts are ordering and the things that accumulated while you were
failed over.

1. **Confirm the original Region is genuinely healthy** — not just that the AWS
   Health Dashboard is green. Invoke the canary. Check ENI state: if the primary's
   functions sat idle for more than 14 days during a long failover, **the primary
   is now the one with `Inactive` VPC-attached functions** and a multi-minute
   ENI provisioning wait on first invocation. Failback inherits the exact problem
   failover was designed around, in the opposite direction. Warm it before you
   cut over.
2. **Verify code parity.** Any hotfix deployed to the standby during the incident
   must also be in the primary. This is the most common failback bug in any
   active/passive design and Lambda makes it easy to get wrong because deploying
   to one Region "works".
3. **Fence the standby** (reserved concurrency 0, then disable its ESMs).
4. **Re-enable the primary's ESMs.** Order matters: disable before enable, or you
   double-process.
5. **Drain the standby's queues first** if they hold anything — messages written
   to the standby SQS/SNS during the incident do not migrate back. See
   [[messaging-in-flight-data-loss]].
6. **Reconcile Terraform.** Set `local.active` back to `primary`, apply, confirm
   the plan is only the expected diffs.
7. **Remove provisioned concurrency** from the standby if you allocated it. This
   is the line item people forget and then find on next month's bill.

---

## Gotchas

1. **The concurrency quota is per-Region and does not follow your quota
   increases.** Verified: "Quotas for concurrent executions and storage apply per
   AWS Region." Default 1,000. Check both sides with `get-account-settings`.
2. **The 10× requests-per-second ceiling.** Concurrency quota 1,000 means 10,000
   rps *account-wide in that Region*, no matter how short the functions are.
   Short-duration functions can be rps-bound while showing low concurrency.
3. **The 14-day Hyperplane ENI reclaim.** A VPC-attached standby function that is
   never invoked goes `Inactive`; the next invocation *fails* and the function
   re-enters `Pending` for several minutes. Hourly canary per subnet+SG
   combination.
4. **The artifact bucket / ECR repo must be in the function's Region.**
   `PermanentRedirect` for S3. Cross-*account* is fine, cross-*region* is not.
5. **Neither S3 CRR nor ECR replication is retroactive.** Enabling either gives
   you a destination containing only things published after the rule existed.
6. **`image_uri` pins a digest at create time.** Mutable tags let the two Regions
   pin different images. Use immutable tags or digests.
7. **Environment variables are version-locked.** They are part of the
   version-specific configuration snapshot taken at publish. A standby whose env
   vars point at the primary's queue ARNs will work perfectly in testing (the
   cross-region SDK call succeeds!) and be catastrophically wrong at failover —
   the standby will drain the *dead* Region's queue, or try to. This is the
   quietest failure in the note: **cross-region SDK calls from Lambda are not
   blocked, so a misconfigured standby looks healthy.**
8. **4 KB total for all environment variables.** Region-suffixing a lot of ARNs
   can push you over. If you're close, move config to
   [[aws-ssm-parameter-store]] or [[aws-secrets-manager]] and read at init — but
   then note that the parameter/secret must also exist in the standby Region, and
   if you use SnapStart, values read during init are baked into the snapshot.
9. **KMS key for environment-variable encryption is regional.** `KMSKeyArn` must
   resolve in the function's Region. A single-region CMK in `eu-west-1` cannot
   encrypt `eu-west-2`'s env vars. See [[aws-kms]] and
   [[kms-when-to-use-multi-region-keys]] — this is one of the cases where a
   multi-region key with its regional alias genuinely simplifies the Terraform.
10. **Event source mappings cannot cross Regions.** SQS is explicit; the same
    holds for the other ESM sources. A standby ESM consumes a *different* queue.
11. **An enabled standby ESM is a live second consumer.** `enabled = false` by
    default, and `ignore_changes = [enabled]` so the runbook's flip survives.
12. **`.sync`-style long polls and `ReportBatchItemFailures`.** ESMs process each
    event at least once — AWS says so in bold. Duplicate processing across a
    failover is not hypothetical; make handlers idempotent.
13. **SnapStart and provisioned concurrency are mutually exclusive** on the same
    function version. Choose one per function.
14. **SnapStart's uniqueness problem is an application-code problem.** UUIDs,
    entropy, cached credentials and open connections created during init are
    shared across every environment restored from the snapshot.
15. **SnapStart charges accrue in the standby.** Per published version, minimum 3
    hours, plus re-init charges when AWS patches the snapshot. Small, but
    non-zero, and it is one of the only idle Lambda costs.
16. **Lambda@Edge is `us-east-1`-only and its service-linked-role Region list
    excludes both Canadian Regions.** Record in [[region-pair-selection]].
17. **Lambda@Edge requires a numbered version, not an alias.** Every
    Lambda@Edge deployment is a distribution update, which propagates on
    CloudFront's schedule, not yours.
18. **Function URLs are not available in every Region.** AWS says so and points at
    the Regional Services page rather than publishing a list. **I could not verify
    `ca-west-1` function-URL availability from a static page** — check the console
    or the regional services explorer before designing around it.
19. **Function URLs have no PrivateLink support** and cannot take a custom domain
    directly. Front with CloudFront if you need either.
20. **Deleting and recreating a function URL gives you a new URL.** There is also
    a documented race: deleting a function and immediately recreating one with
    the same name can map the old URL to the new function.
21. **The implicit CloudWatch log group.** Lambda creates `/aws/lambda/<name>`
    with no retention if you don't. Standing up a standby with an explicit log
    group resource after the fact requires an import.
22. **The 300 GB per-Region storage quota for `.zip` functions and layers.**
    Every published version consumes it. A mature estate with `publish = true`
    and no version cleanup will hit this eventually — and it will hit it in the
    standby too, because versions are published there as well.
23. **Control-plane API throttling: 15 requests/second across all Lambda
    control-plane APIs, not per API, and not increasable.** A `terraform apply`
    that creates hundreds of functions in a fresh standby Region will be
    throttled. Expect a slow first apply and use `-parallelism` sensibly.
24. **`us-east-1` is on the critical path for creating things that need DNS.**
    [[aws-regional-outages]] documents that Lambda **function URLs** are on AWS's
    own list of resources whose creation depends on the `us-east-1` Route 53
    control plane. Creating a function URL during a failover takes a
    `us-east-1` dependency. Pre-create them.
25. **The June 2023 Lambda event is the argument for all of this.** Per
    [[aws-regional-outages]] and [AWS's
    summary](https://aws.amazon.com/message/061323/): a latent defect in the
    Lambda Frontend fleet surfaced when organic traffic growth crossed a capacity
    threshold. Nobody deployed anything. STS, the Console, EventBridge (delivery
    latencies "of up to 801 seconds"), EKS cluster provisioning and AWS Support
    were all collateral. **A Lambda regional failure is a real, documented,
    first-party event, and during it you could not reach the Console or the
    global STS endpoint to do anything about it.** Break-glass credentials and
    regional STS endpoints are prerequisites for the Lambda failover runbook, not
    polish. See [[aws-iam]].

---

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Artifact distribution | CI publishes to every Region's bucket/repo | S3 CRR / ECR replication does it | **A, with B as a backstop.** Replication is not retroactive and is async with no useful lag guarantee. CI is deterministic and fails loudly. |
| Packaging | `.zip` + S3 | Container image + ECR | **Whichever you use today.** `package_type` forces replacement; don't stack that risk onto the multi-region work. |
| Cold-start mitigation in the standby | Provisioned concurrency | SnapStart | **SnapStart if the runtime supports it** (Java/Python 3.12+/.NET 8+) — free for Java, no idle reservation, available in `ca-west-1`. PC only for Node.js/Ruby critical-path functions. |
| Provisioned concurrency posture | Standing reservation in the standby | Allocate at failover | **Neither, for most functions.** On-demand scaling of 1,000/10 s per function clears the RTO. Standing PC only for a short, evidence-based list of user-facing sync functions. Failover-time allocation buys nothing over on-demand (same 6,000/min rate) and adds a control-plane dependency. |
| Standby ESM state | Enabled (hot consumers) | Disabled until failover | **Disabled.** Consistent with the recommendation in [[messaging-in-flight-data-loss]]: warm process, cold subscription. |
| ESM flip mechanism | ARC Region Switch ESM execution block | Custom script / Step Functions | **ARC Region Switch**, if [[failover-orchestration]] adopts ARC generally. It is purpose-built for exactly this and handles the ungraceful path. Otherwise a script — but see ARC's own control-plane caveat in [[aws-regional-outages]]. |
| Function naming | Identical name in both Regions | `-standby` suffix | **Identical.** Asymmetric names infect every alarm, dashboard, query and policy. |
| VPC attachment | Keep it | Drop it where unnecessary | **Drop it where unnecessary.** It removes the 14-day ENI reclaim risk, the several-minute Pending state, and a dependency on [[aws-vpc-networking]] being correct in the standby. |
| Edge logic | Lambda@Edge | CloudFront Functions | **CloudFront Functions where the logic fits.** Fewer `us-east-1` dependencies, and Lambda@Edge's SLR Region list excludes Canada. |
| Terraform layout | Paired module blocks with provider aliases | Separate root stacks per Region | **Either — Lambda is the one service where this is genuinely a free choice**, because there are no cross-region references. Defer to whatever [[provider-aliases-vs-separate-stacks]] concludes for the estate as a whole. |

---

## Cost

**At idle, a Lambda warm standby costs approximately nothing.** Lambda has no
per-function standing charge. The standby's bill while the primary is healthy:

| Line | Cost at idle |
|---|---|
| Functions, versions, aliases | $0 |
| Invocations, duration | $0 (canaries: pennies) |
| Artifact storage (S3 or ECR) | Storage rate × artifact size × number of retained versions |
| CloudWatch log groups | $0 empty; canary logs are pennies |
| Event source mappings (disabled) | $0 |
| Hyperplane ENIs | $0 for the ENI; the subnets/NAT behind it are [[aws-vpc-networking]]'s cost, not Lambda's |
| SnapStart cache | Per published version, min 3 hours, at $0.0000015046/GB-s (us-east-1) — dollars per month for a typical estate |
| **Provisioned concurrency** | **The only material line. ~$10.95 per GB per month per unit at the us-east-1 rate.** |

Levers, in order of impact:

1. **Don't buy provisioned concurrency you don't need.** This is 95% of the
   decision. The default should be zero.
2. **Use SnapStart instead where the runtime allows.** Free for Java managed
   runtimes.
3. **Right-size memory.** Every Lambda cost — on-demand, provisioned, SnapStart
   cache, SnapStart restore — is denominated per GB. Memory is also CPU (1 vCPU
   at 1,769 MB), so over-provisioning memory to fix a latency problem is a
   legitimate move; over-provisioning it by accident doubles the standby's PC
   bill.
4. **Prune published versions.** They consume the 300 GB per-Region storage
   quota and, with SnapStart, each one carries a cache charge. In both Regions.
5. **arm64 (Graviton)** is cheaper per GB-second than x86 on AWS's published
   rates and is supported everywhere except Lambda@Edge. If the estate is not on
   arm64 yet, the multi-region rebuild is a natural moment — but see the caveat
   about stacking changes.

For the full picture, this feeds [[cost-model]] and [[cost-levers]]. The headline
for those notes: **Lambda is the cheapest service in the estate to make
multi-region, and it should be sequenced early precisely because it produces a
visible win at near-zero standing cost.**

---

## Open questions

1. **What is the current account concurrency quota in each of the six Regions?**
   A two-minute `get-account-settings` check per Region that nobody has run. This
   is the highest-value unanswered question in this note.
2. **What is the p99 `InitDuration` per function, and which functions are
   actually on the synchronous user-facing path?** Without this, the provisioned
   concurrency list is guesswork.
3. **What runtimes does the estate use?** SnapStart eligibility (Java 11+,
   Python 3.12+, .NET 8+) changes the recommendation completely. If it is all
   Node.js, provisioned concurrency is the only lever and the cost conversation
   is real.
4. **Are any functions VPC-attached, and how many distinct (subnet, security
   group) combinations are there?** Determines how many warming canaries are
   needed and whether the 14-day reclaim is a live risk.
5. **Does anything use Lambda@Edge today?** If yes, the CA deployment has a
   `us-east-1` dependency that data residency may not tolerate. See
   [[data-residency]].
6. **Does anything use function URLs?** If yes, are they fronted by CloudFront?
   Unfronted function URLs cannot be failed over — the hostname is AWS's and
   contains the Region.
7. **Do handlers currently assume regional ARNs from environment variables, or do
   they construct ARNs from `AWS_REGION` at runtime?** The latter is much safer
   and may already be in place. Worth a grep before writing any Terraform.
8. **Is there any function that is not idempotent?** ESM flips guarantee at least
   one duplicate-processing window at failover. This is the one Lambda question
   that is really an application-architecture question.
9. **Is `us-east-1` the primary for the US pair by choice?** [[aws-regional-outages]]
   makes the case that it is the highest-risk Region in the partition. Out of
   scope for this note but it keeps coming up.

---

## Sources

**AWS first-party documentation**

- [Lambda quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)
  — the per-Region scoping of the concurrency quota, the 1,000 default, the 300 GB
  storage quota, the 500-ENI-per-VPC quota, the 15 rps control-plane API limit.
  The load-bearing citation for this note's RTO argument.
- [Understanding Lambda function scaling](https://docs.aws.amazon.com/lambda/latest/dg/lambda-concurrency.html)
  — the 1,000-environments-per-10-seconds scaling rate, the 10× rps ceiling,
  reserved vs provisioned concurrency semantics, and the "6,000 per minute"
  provisioned-concurrency allocation rate with the "minute or two of preparation"
  caveat.
- [Improving startup performance with Lambda SnapStart](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html)
  — supported runtimes, the "all commercial Regions except Asia Pacific (New
  Zealand) and Asia Pacific (Taipei)" availability statement (the `ca-west-1`
  parity finding), mutual exclusivity with provisioned concurrency, and the
  pricing model.
- [Giving Lambda functions access to resources in an Amazon VPC](https://docs.aws.amazon.com/lambda/latest/dg/foundation-networking.html)
  — Hyperplane ENI lifecycle, the "several minutes" Pending state, the 14-day
  idle reclaim, the 20-minute detach, the subnet+SG sharing rule. The source of
  the single most under-appreciated warm-standby risk in this note.
- [Troubleshoot deployment issues in Lambda](https://docs.aws.amazon.com/lambda/latest/dg/troubleshooting-deployment.html)
  — the S3 artifact bucket same-Region requirement and the `PermanentRedirect`
  error.
- [Create a Lambda function using a container image](https://docs.aws.amazon.com/lambda/latest/dg/images-create.html)
  — the ECR same-Region requirement with cross-account permitted.
- [How Lambda processes records from stream and queue-based event sources](https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventsourcemapping.html)
  — which sources use ESMs, at-least-once delivery, provisioned mode.
- [Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)
  — the explicit sentence: "The Lambda function and the Amazon SQS queue must be
  in the same AWS Region, although they can be in different AWS accounts."
- [Working with Lambda environment variables](https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html)
  — 4 KB aggregate limit, version-locking at publish, `KMSKeyARN`, and the
  reserved `AWS_REGION` variable that handlers should be using instead of
  hardcoded region strings.
- [Creating and managing Lambda function URLs](https://docs.aws.amazon.com/lambda/latest/dg/urls-configuration.html)
  — the `https://<url-id>.lambda-url.<region>.on.aws` format, "not supported in
  all AWS regions", no PrivateLink, the delete-is-irreversible behaviour, and the
  reserved-concurrency-zero deactivation trick that doubles as a fencing
  primitive.
- [Restrictions on Lambda@Edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-at-edge-function-restrictions.html)
  — "The Lambda function must be in the US East (N. Virginia) Region", numbered
  versions only, and the full unsupported-feature list.
- [Set up IAM permissions and roles for Lambda@Edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-edge-permissions.html)
  — the replicator service-linked role, and the supported-Region list that omits
  both Canadian Regions.
- [AWS Lambda pricing](https://aws.amazon.com/lambda/pricing/) — the us-east-1
  x86 rates quoted above. **Regional rates were not obtainable from the static
  page; treat the non-us-east-1 numbers as unverified.**
- [Private image replication in Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/replication.html)
  — the replication configuration that is not retroactive.
- [Lambda event source mapping execution block](https://docs.aws.amazon.com/r53recovery/latest/dg/lambda-event-source-mapping-block.html)
  and the [May 2026 announcement](https://aws.amazon.com/about-aws/whats-new/2026/05/region-switch-lambda-esm-execution-block/)
  — ARC Region Switch's purpose-built ESM enable/disable blocks, including the
  "ungraceful" mode for when the deactivating Region is unreachable. AWS's own
  problem statement matches this note's almost word for word.
- [Summary of the AWS Lambda Service Event in the Northern Virginia (US-EAST-1)
  Region, June 13 2023](https://aws.amazon.com/message/061323/) — the documented
  Lambda regional failure. Analysed in [[aws-regional-outages]].

**Provider / community**

- [hashicorp/terraform-provider-aws#29653](https://github.com/hashicorp/terraform-provider-aws/issues/29653)
  — Lambda replacement behaviour and the cascade into dependent resources.
- [hashicorp/terraform-provider-aws#10298](https://github.com/hashicorp/terraform-provider-aws/issues/10298)
  — `aws_lambda_alias` failing to update in place when the function name changes;
  relevant to the "don't rename the function" rule.
- [aws/containers-roadmap#1289](https://github.com/aws/containers-roadmap/issues/1289)
  — the open request for ECR to replicate *existing* images. Still open, which is
  itself the finding.

### Searched for and did not find

- **A per-Region Lambda pricing table in machine-readable form on the pricing
  page.** The Region selector is client-side. The Price List API is the way, and
  the numbers in this note are `us-east-1` only and labelled as such.
- **A published AWS figure for Hyperplane ENI provisioning time.** AWS says
  "several minutes"; third-party blogs quote "up to 90 seconds" but none of them
  cite a first-party source, so that number is deliberately not used here.
- **Any public post-mortem of a customer whose Lambda failover failed on the
  standby's concurrency quota.** This is the most-predicted failure mode in this
  note and I found no public write-up of it happening. The mechanism is
  documented and the arithmetic is unambiguous, but treat the *scenario* as
  reasoned, not observed.
- **An AWS statement that Lambda function URLs are or are not available in
  `ca-west-1`.** AWS points at the Regional Services explorer rather than
  publishing a list. Verify in-console.
