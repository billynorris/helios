---
title: Amazon ECR — Multi-Region
service: ecr
tags: [service, multi-region, ecr, containers, registry, images]
status: researched
replication: native (registry-level cross-region replication) — but not retroactive and not synchronous
rpo_achievable: "N/A for business data. For images: 'the last image pushed more than ~30 minutes ago' — AWS publishes no SLA, only a 'majority under 30 minutes' expectation"
rto_achievable: "< 1 min to first pull IF the image is already in the standby region's registry AND the manifest references the standby region's registry hostname. Unbounded / total failure if either is false."
meets_targets: conditional
updated: 2026-09-21
---

# Amazon ECR — Multi-Region

> Companion to [[aws-eks]] (the cluster) and [[eks-workload-delivery]] (GitOps,
> secrets, ingress). Those two notes established *that* ECR replication exists
> and named the headline gotchas. **This note is the full treatment**: the
> registry-level mechanics, the backfill problem, the race between CI and
> replication, the image-reference problem that replication does not solve, and
> what the CI pipeline has to become.

## TL;DR

- **The question that matters: when `eu-west-1` is gone, can the standby EKS cluster in `eu-west-2` pull the images it needs to start?** The answer is **yes, but only if two independent things are both true**, and most estates get exactly one of them right.
  1. The image **bits** are physically present in the `eu-west-2` registry. That is what ECR cross-region replication gives you — subject to the not-retroactive trap and the async lag below.
  2. The image **reference in the Kubernetes manifest** names the `eu-west-2` registry. `123456789012.dkr.ecr.eu-west-1.amazonaws.com/prod/payments@sha256:…` in a standby Deployment is a **hard dependency on the dead region**, and it is the default outcome of copying the primary's manifests. Replication does not help you here at all. **This is the single most important sentence in the note.**
- **ECR replication configuration is registry-wide, not per-repository.** One `PutReplicationConfiguration` per account per region. There is no `replication` block on `aws_ecr_repository`. In a multi-account estate that means one replication configuration per account per primary region — it is not something you can express once in a shared module and forget.
- **Replication is not retroactive.** AWS: *"Only repository content pushed or restored to a repository after replication is configured is replicated. Any preexisting content in a repository isn't replicated."* Every image you have today stays in `eu-west-1` forever unless you explicitly copy it. This is the exact same shape as the [[aws-s3]] CRR problem (CRR replicates new objects; existing objects need S3 Batch Replication) — except **ECR has no Batch Replication equivalent**. You must bring your own copy job. Recommendation: `crane`/`regctl` bulk copy, once, at migration time. See [[#The not-retroactive trap]].
- **Replication is asynchronous and AWS publishes no SLA** — only *"The majority of images replicate in less than 30 minutes, but in rare cases the replication might take longer."* A CI pipeline that pushes and immediately deploys to both regions **races the replication**, and the standby loses. There is a real completion signal — `DescribeImageReplicationStatus` and the EventBridge `ECR Replication Action` event — and the pipeline should gate on it. See [[#Detecting replication completion]].
- **The thing that will bite (other than the image reference):** `:latest`, or any mutable tag. Replication resolves a tag to a digest at push time in the source and pushes that digest to the destination. If the two registries ever disagree about what `:latest` means — and with mutable tags plus a partial replication they *will* — then failover starts an **untested build**. Combined with the documented tag-immutability interaction (*"the image is replicated but won't contain the duplicated tag. This might result in the image being untagged"*), mutable tags in a two-region estate produce an image that is either wrong or unpullable. **Deploy by digest.** See [[#Immutable tags and why `:latest` is fatal]].

## Does this service cross regions at all?

ECR is a **regional** service with a per-account, per-region registry. The
registry hostname is region-qualified and there is no global endpoint:

```
<account-id>.dkr.ecr.<region>.amazonaws.com        # OCI/Docker client (pulls/pushes)
api.ecr.<region>.amazonaws.com                     # control plane API
<account-id>.dkr-ecr.<region>.on.aws               # dual-stack, added April 2025
ecr.<region>.api.aws                               # dual-stack API
```

| Thing | Regional or global? | Consequence for the standby |
|---|---|---|
| The registry itself | **Regional**, one per account per region | The standby has its own, empty until you fill it. |
| Repository (`aws_ecr_repository`) | **Regional** | Must exist in the standby. Replication can auto-create it. |
| Replication configuration (`aws_ecr_replication_configuration`) | **Registry-scoped** — one per account per region | Configured in the **source** region. Not per-repository. |
| Registry permissions policy (`aws_ecr_registry_policy`) | **Registry-scoped**, regional | Needed in the **destination** for cross-*account* replication only. |
| Repository policy (`aws_ecr_repository_policy`) | **Per-repository**, regional | **Not replicated.** Must be recreated in the standby. |
| Lifecycle policy (`aws_ecr_lifecycle_policy`) | **Per-repository**, regional | **Not replicated.** Runs independently per region — see [[#Lifecycle policies apply per region]]. |
| Repository creation template (`aws_ecr_repository_creation_template`) | **Registry-scoped**, regional | Lives in the **destination**. This is how replicated repos inherit settings. |
| Pull-through cache rule | **Registry-scoped**, regional | Per region. See [[#Pull-through cache rules]]. |
| KMS key encrypting a repository | **Regional** — *"the key must exist in the same Region as your repository"* | Two keys, or one multi-Region key. See [[#KMS encryption per region]]. |
| Image tag → digest mapping | **Per-registry** | The two registries can disagree. This is the `:latest` failure. |
| ECR Public | **Global-ish**, `public.ecr.aws`, registry lives in `us-east-1` | Not a DR mechanism. See [[#ECR Public vs private]]. |

**Native cross-region support: yes**, and it is better than most services in
this vault. But it is *push-triggered fan-out*, not a mirror. Nothing
reconciles. Nothing backfills. Nothing deletes. It is closer to "ECR forwards
each push" than to "the two registries are kept in sync", and every gotcha in
this note is downstream of that distinction.

## The spine: can the standby actually pull?

Walk it concretely. `eu-west-1` is gone. The `eu-west-2` cluster from
[[aws-eks]] scales its node group from 0 to 6. Six nodes boot, kubelet starts,
pods are scheduled, and kubelet must now pull `prod/payments`.

There are **four** independent things that must be true. Miss any one and the
pod is `ImagePullBackOff` and the 15-minute RTO is gone.

### 1. The bits are in `eu-west-2`

Replication must have been configured *before* the image was pushed, the
repository prefix filter must match, and replication must have completed.
Covered in [[#Cross-region replication]] and [[#The not-retroactive trap]].

### 2. The manifest names the `eu-west-2` registry

This is the one that is almost always wrong, because it is invisible while the
primary is healthy.

An ECR image reference **encodes the region in the hostname**. There is no
region-neutral ECR name. So a Deployment that says:

```yaml
image: 123456789012.dkr.ecr.eu-west-1.amazonaws.com/prod/payments@sha256:9f86d0…
```

will, when scheduled in `eu-west-2`, attempt a pull **from `eu-west-1`**. You
have replicated every byte, paid for the storage twice, and the standby still
has a hard dependency on the dead region. The failure is worse than slow:

- If the standby VPC is private-only with ECR interface endpoints (the normal
  shape for this estate — see [[aws-vpc-networking]]), the `eu-west-2` interface
  endpoint **does not resolve or serve `eu-west-1`'s registry**. The pull does
  not merely go over the internet; it fails at DNS or TLS. There is no slow path.
- If the standby has a NAT route out, the pull attempts a genuine cross-region
  fetch from a region that is having a regional event. Best case it is slow
  enough to blow the RTO; worst case it hangs on connect timeouts and
  `ImagePullBackOff` backs off exponentially (kubelet's backoff caps at 5
  minutes), so even after the primary recovers your pods take minutes longer.
- **kubelet's ECR credential provider will happily try.** The AWS-provided
  `ecr-credential-provider` parses the registry host from the image reference
  and calls `GetAuthorizationToken` against *that* region. So authentication is
  not what breaks — which is exactly why this is silent. It fails at the network
  layer, deep in the pull, not at a policy check you might have tested.

**The fix: the image reference must be templated per cluster.** Three ways, in
order of preference for a GitOps estate:

| Mechanism | How | Notes |
|---|---|---|
| **Kustomize `images:` transformer** in a per-region overlay | `images: [{name: app, newName: "123456789012.dkr.ecr.eu-west-2.amazonaws.com/prod/payments"}]` | Cleanest. The digest stays in the base; only the host changes. Composes with the ApplicationSet from [[eks-workload-delivery#ApplicationSet with a cluster generator]]. |
| **Helm value** `image.registry`, fed from the cluster's region label | `registry: "{{ .Values.awsAccountId }}.dkr.ecr.{{ .Values.region }}.amazonaws.com"` | Fine if you already use Helm. The region must come from the *cluster* label, never a hardcoded string. |
| **containerd registry mirror / host rewrite on the node** | Bottlerocket and AL2023 both support containerd `config_path` host directories; you point `<acct>.dkr.ecr.eu-west-1.amazonaws.com` at the local region's endpoint | Keeps manifests byte-identical across regions, which is genuinely attractive. But it is node-level config that is invisible from Kubernetes, it is a per-AMI concern, and the auth flow gets subtle. **Not recommended as the primary mechanism** — good as a belt-and-braces net underneath one of the above. |

> [!warning] Grep for this before anything else
> `grep -rE '\.dkr\.ecr\.[a-z0-9-]+\.amazonaws\.com' <deploy-repo>` across the
> deploy repo, every Helm `values.yaml`, every ConfigMap, every Dockerfile
> `FROM`, every CI workflow, and every ArgoCD Application. Then do it again for
> the Terraform repo (Lambda container images — see [[aws-lambda]] — and ECS
> task definitions if any survive). Count the hits that hardcode a region.
> That number is your real exposure, and it is reliably larger than anyone
> expects. This is the same class of finding as
> [[eks-workload-delivery#Gotchas]] item 13, and it is the highest-value single
> command in this note.

### 3. The pull path out of the standby VPC works

The `eu-west-2` nodes need to reach `eu-west-2`'s ECR. In a private-subnet
cluster that means either a NAT path or, preferably, VPC endpoints. ECR pulls
need **three** endpoints, not two, because layer blobs are served from S3:

| Endpoint | Type | Why |
|---|---|---|
| `com.amazonaws.eu-west-2.ecr.api` | Interface | `GetAuthorizationToken`, `BatchGetImage`, `GetDownloadUrlForLayer` |
| `com.amazonaws.eu-west-2.ecr.dkr` | Interface | The Docker Registry API the container runtime speaks |
| `com.amazonaws.eu-west-2.s3` | **Gateway** (or interface) | The actual layer bytes. Forgetting this is the classic "auth works, pull hangs". |

If the standby VPC was built as a cheap pilot light with one of these missing,
you discover it during the drill — or during the incident. This belongs in
[[aws-vpc-networking]] but the consequence lands here.

### 4. ECR in `eu-west-2` can serve six nodes' worth of pulls at once

See [[#ECR throttling during a mass pull]]. The short answer is that the default
quotas are generous enough for this estate, and the constraint is bandwidth and
NAT, not the ECR API — but the numbers are worth knowing rather than assuming.

### Verdict on the spine

**Yes, the standby can genuinely start without the primary region — but only
after a specific piece of work that ECR replication alone does not do for you.**
Replication solves problem (1). Problems (2), (3) and (4) are yours. Of those,
(2) is the one that silently converts a warm standby into an illusion, and it is
not mentioned in AWS's replication documentation at all, because from ECR's
point of view nothing is wrong.

## Cross-region replication

### The mechanism

You set a **replication configuration on the registry** in the source region.
It contains up to 25 rules; each rule has up to 100 repository filters and
points at one or more destinations (region + account). On each successful
`PutImage`, ECR asynchronously copies the image to every matching destination.

Key structural facts, all from the
[ECR replication docs](https://docs.aws.amazon.com/AmazonECR/latest/userguide/replication.html):

- **It is registry-wide.** There is no per-repository replication setting. You
  select repositories with **prefix filters** inside the registry-level rules,
  not by tagging repositories.
- **Repository names are preserved and cannot be changed.** *"The repository name
  will remain the same across Regions and accounts when replication has
  occurred. Amazon ECR doesn't support changing the repository name during
  replication."* So if your repositories are named per-region today
  (`prod-eu-west-1/payments`), replication will faithfully create
  `prod-eu-west-1/payments` in London. Fix the naming before you turn this on —
  this is the same reconciliation problem [[research-brief]] describes for
  DynamoDB Global Tables.
- **A service-linked role is created on first configuration**
  (`AWSServiceRoleForECRReplication`). You do not manage it; you do need
  `iam:CreateServiceLinkedRole` on the principal that first applies the config.
- **Replication does not chain.** *"A replication action only occurs once per
  image push or image restore."* A → B and B → C does **not** give A → C. For
  three independent pairs this is irrelevant; it matters the moment someone
  proposes a hub registry.
- **Not across partitions.** Irrelevant here (all commercial `aws`), noted for
  completeness.
- **Blob mounting**: when replicating, ECR checks whether layers already exist in
  the destination and mounts them rather than re-transferring. Cross-*account*
  replication requires blob mounting enabled on **both** registries for this to
  apply. For same-account cross-region this is a free efficiency win — shared
  base layers replicate once.

### Registry-wide vs a multi-account estate

This is the part that bites templated monorepos.

The replication configuration is **one object per account per region**. It is
not composable: a second `aws_ecr_replication_configuration` in the same
account+region does not add rules, it **replaces** the whole configuration. In
Terraform, two module instances each declaring one is a fight, and the last
`apply` wins silently.

Consequences for the estate:

| Shape | What it means for ECR |
|---|---|
| **Single shared account**, three region-pairs | **Three** replication configurations: one in `eu-west-1` → `eu-west-2`, one in `us-east-1` → `us-west-2`, one in `ca-central-1` → `ca-west-1`. Each lives in its own region so they do not collide. Clean. |
| **Account per environment** (prod / staging / dev), shared across regions | One replication configuration per account per primary region. Still clean, but it must be owned by exactly one root module per account+region. |
| **Account per region** (prod-eu, prod-us, prod-ca) | Same as above, plus the cross-account policy work below. |
| **Central "shared services" registry account**, workloads elsewhere | The registry account owns all replication configurations. Workload accounts need `ecr:GetAuthorizationToken` + repository policies granting cross-account pull. And you must answer [[aws-eks]]'s open question 2 ("is the standby in the same account?") before designing this. |

**Terraform rule that follows from this:** the replication configuration must be
declared in exactly **one** root module per account+region, at the registry
level — *not* inside a per-service or per-repository module that gets
instantiated fifty times. If your cookiecutter template has a
`modules/ecr-repository` that teams instantiate per microservice, the
replication configuration does **not** belong in it. Put it in the
account-and-region-level "registry settings" root alongside the registry
scanning configuration and the repository creation templates. See
[[#Terraform implementation]] and [[terraform-delivery-and-state-platform]].

### Cross-account replication

Only the **destination** needs a policy. AWS is unusually explicit about this
because people get it backwards:

> Cross-account ECR replication requires policy configuration on the
> **destination account only**. The source account does not require any special
> repository or registry policies.

The destination registry policy must grant the source account:

- `ecr:ReplicateImage`
- `ecr:CreateRepository` — **and if you withhold this, replication silently
  fails unless you pre-create every destination repository by hand.** AWS:
  *"If you do not grant the `ecr:CreateRepository` permission, you must manually
  create repositories with the same names in the destination account before
  replication can succeed."*

If you use repository creation templates with **resource tags** in a
cross-account setup, the destination registry policy also needs the tagging
permission (`ecr:TagResource`) — the docs call this out specifically for
cross-region replication with templates.

Also noted in the docs, and worth knowing for an incident: *"If the permission
policy for a private registry are changed to remove a permission, any
in-progress replications previously granted may complete."* Revoking access is
not an immediate stop button.

### Limits

| Limit | Value | Adjustable | Headroom for this estate |
|---|---|---|---|
| Rules per replication configuration | 25 | No | Three pairs need 1–3 rules each. Vast headroom. |
| Unique destinations across all rules | 25 | No | Need 1 per configuration. Vast headroom. |
| Filters per rule | 100 | No | Prefix filters, so a handful. Fine. |
| Registered repositories per region | 100,000 | Yes | Fine. |
| Images per repository | 100,000 | Yes | Fine — but see [[#Lifecycle policies apply per region]], because without a destination lifecycle policy this is the number that eventually matters. |
| Pull-through cache rules per registry | 50 | **No** | Four upstreams per region is comfortable. Worth knowing it is a hard ceiling if anyone proposes a PTC rule per vendor. |
| Rules per lifecycle policy | 50 | **No** | Fine. |
| Lifecycle policy length | 30,720 characters | **No** | Fine, but a generated policy with a long `tagPatternList` can approach it. |
| Tags per image | 1,000 | No | Fine. |

All verified against the [ECR service quotas table](https://docs.aws.amazon.com/AmazonECR/latest/userguide/service-quotas.html), which states *"Each supported Region"* for every entry — there is no published per-Region variation, including for `ca-west-1`.

## The not-retroactive trap

### What AWS actually says

> Only repository content pushed or restored to a repository after replication
> is configured is replicated. Any preexisting content in a repository isn't
> replicated. If an image is restored after replication is turned on, it will be
> replicated. If it is restored before replication is turned on, it won't be
> replicated.

Turn replication on this afternoon and your standby registry is **empty**. It
stays empty until the next push. For a service that deploys weekly, that is a
week of a standby cluster that cannot start. For a service that has not been
deployed in six months — and every estate has three of those — it is empty
**forever**.

### This is the S3 CRR problem, and ECR is worse

[[aws-s3]] covers the identical shape: S3 Cross-Region Replication only
replicates objects written *after* the rule exists, and existing objects need a
separate **S3 Batch Replication** job. Cross-reference it, because the mental
model transfers exactly — and so does the failure: a team turns on replication,
sees "Enabled" in the console, and believes the standby is populated.

The difference, and it is not in ECR's favour:

| | S3 | ECR |
|---|---|---|
| New data replicates automatically | Yes | Yes |
| Existing data replicates automatically | No | No |
| **AWS-provided backfill** | **Yes — S3 Batch Replication**, a first-class managed job with a completion report | **No. Nothing.** |
| Replication metrics / SLA option | Yes — S3 RTC, 15-minute SLA for 99.99% of objects | **No. No SLA at all.** |
| Per-object replication status | `x-amz-replication-status` header | `DescribeImageReplicationStatus` (good, see below) |

**ECR has no Batch Replication and no RTC equivalent.** The backfill is entirely
your problem, and the tightest bound you get on latency is a prose sentence in
the documentation.

### What you actually do about it

Four options. All of them work; they differ in blast radius and in whether they
preserve digests.

#### Option A — Re-push everything from CI

Re-run the build/push stage for every service, which triggers replication
naturally.

- **Pro:** no new tooling, uses the path you already trust.
- **Con, and it is fatal:** a rebuild produces a **different digest** unless your
  builds are bit-for-bit reproducible, which they are not. You have now
  replicated an image that is *not* the one running in production. Failover
  lands on a rebuild. This is the same untested-build failure as `:latest`,
  arrived at from a different direction.
- **Con:** you cannot re-push at all for images whose source no longer builds —
  the old service nobody touches, the vendored third-party image, the base image
  from a version of the Dockerfile that has been deleted.

**Do not do this.** It looks like the cheapest option and it quietly defeats the
purpose.

#### Option B — Pull and re-push the exact bytes (`docker pull` + `docker tag` + `docker push`)

Preserves the digest for single-architecture images. But `docker pull` of a
multi-arch manifest list pulls **one** platform, and `docker push` then publishes
a single-platform manifest. If any of your images are multi-arch — and with
Graviton node groups in [[aws-eks]] they very likely are — you silently lose the
`arm64` variant, or the `amd64` one, and the standby's nodes cannot run what
they pull. Also slow: every byte goes through the copying machine.

**Only viable if you have verified every image is single-arch. Assume you
haven't.**

#### Option C — Registry-to-registry copy with `crane` / `skopeo` / `regctl` (recommended)

These tools copy **manifests and blobs directly between registries** without
materialising the image locally, and they preserve the digest and the full
multi-arch manifest list.

```bash
# google/go-containerregistry — crane
# --all-tags copies every tag; the digest is preserved byte-for-byte.
crane copy \
  123456789012.dkr.ecr.eu-west-1.amazonaws.com/prod/payments:v1.42.0 \
  123456789012.dkr.ecr.eu-west-2.amazonaws.com/prod/payments:v1.42.0

# regclient — regctl. Same idea, and `--digest-tags` also carries
# digest-tagged referrers (cosign signatures live as digest-derived tags).
regctl image copy --digest-tags \
  123456789012.dkr.ecr.eu-west-1.amazonaws.com/prod/payments:v1.42.0 \
  123456789012.dkr.ecr.eu-west-2.amazonaws.com/prod/payments:v1.42.0

# containers/skopeo — sync mode copies a whole repository in one invocation.
skopeo sync --all --src docker --dest docker \
  123456789012.dkr.ecr.eu-west-1.amazonaws.com/prod/payments \
  123456789012.dkr.ecr.eu-west-2.amazonaws.com/prod
```

A driver over the whole registry:

```bash
#!/usr/bin/env bash
# One-off ECR backfill. Run from somewhere with good bandwidth to both regions
# (a small EC2 box in the SOURCE region is usually fastest and cheapest).
set -euo pipefail
ACCOUNT=123456789012
SRC=eu-west-1
DST=eu-west-2
PREFIX=prod/          # match the replication rule's filter exactly

aws ecr get-login-password --region "$SRC" \
  | crane auth login --username AWS --password-stdin "$ACCOUNT.dkr.ecr.$SRC.amazonaws.com"
aws ecr get-login-password --region "$DST" \
  | crane auth login --username AWS --password-stdin "$ACCOUNT.dkr.ecr.$DST.amazonaws.com"

aws ecr describe-repositories --region "$SRC" \
    --query "repositories[?starts_with(repositoryName, '$PREFIX')].repositoryName" \
    --output text | tr '\t' '\n' | while read -r REPO; do

  # Destination repo must exist. Replication auto-creates; a manual copy does not.
  aws ecr describe-repositories --region "$DST" --repository-names "$REPO" >/dev/null 2>&1 \
    || aws ecr create-repository --region "$DST" --repository-name "$REPO" \
         --image-tag-mutability IMMUTABLE >/dev/null

  # Only the tags that matter. Copying five years of CI tags is a waste of
  # money and of the destination lifecycle policy's time.
  aws ecr describe-images --region "$SRC" --repository-name "$REPO" \
      --query 'sort_by(imageDetails,&imagePushedAt)[-20:].imageTags[]' \
      --output text | tr '\t' '\n' | grep -v '^None$' | while read -r TAG; do
    echo "copy $REPO:$TAG"
    crane copy "$ACCOUNT.dkr.ecr.$SRC.amazonaws.com/$REPO:$TAG" \
               "$ACCOUNT.dkr.ecr.$DST.amazonaws.com/$REPO:$TAG"
  done
done
```

> [!note] `crane copy` is not free and not instant
> Every layer crosses a region boundary. Budget the inter-region data transfer
> (see [[#Cost]]) and run it from an instance in the **source** region — egress
> is charged from the source, and pulling to a laptop and pushing back doubles
> the bytes and adds your office internet as the bottleneck.

#### Option D — Accept the gap

Turn replication on, do not backfill, and accept that the standby can only start
services that have been deployed since the cutover.

Defensible **only** if you commit to a forced full redeploy of every service
within a known window, and you can prove it happened. In practice this is Option
A wearing a disguise, with the same rebuilt-digest problem. The one context where
it is genuinely right: if you are *also* moving to deploy-by-digest and a
pipeline that pushes to both regions (see
[[#What the CI pipeline must change to]]), the backfill is subsumed by the first
full redeploy anyway.

### Recommendation

**Option C, once, at migration time, restricted to the tags the standby could
plausibly need** — the currently-deployed digest per service plus the last
N releases for rollback. Then **verify** by diffing the two registries rather
than trusting the copy:

```bash
# Repositories present in the primary but missing in the standby.
comm -23 \
  <(aws ecr describe-repositories --region eu-west-1 \
      --query 'repositories[].repositoryName' --output text | tr '\t' '\n' | sort) \
  <(aws ecr describe-repositories --region eu-west-2 \
      --query 'repositories[].repositoryName' --output text | tr '\t' '\n' | sort)
```

And the check that actually matters — **every digest a running pod references
must exist in the standby registry**:

```bash
# Run against the PRIMARY cluster. Any output is a failover you would not survive.
kubectl get pods -A -o jsonpath='{range .items[*].status.containerStatuses[*]}{.imageID}{"\n"}{end}' \
  | sed 's|.*/||' | sort -u | while read -r REF; do
    REPO="${REF%@*}"; DIGEST="${REF#*@}"
    aws ecr describe-images --region eu-west-2 \
        --repository-name "$REPO" --image-ids imageDigest="$DIGEST" >/dev/null 2>&1 \
      || echo "MISSING IN STANDBY: $REPO@$DIGEST"
  done
```

**Make that second check a scheduled job and put its result on the
`dr_readiness` dashboard from [[aws-eks#What actually prevents it]].** It is the
only continuous, honest answer to "can the standby start", and it costs one
Lambda on a cron. It catches the backfill gap, the prefix-filter mismatch, the
lifecycle-policy deletion in [[#Lifecycle policies apply per region]], and a
half-finished replication, all with one signal.

## Replication latency and the CI race

### What is published

The only latency statement AWS makes is:

> The majority of images replicate in less than 30 minutes, but in rare cases the
> replication might take longer.

**Verified: there is no ECR replication SLA, no published percentile
distribution, and no ECR equivalent of S3 Replication Time Control.** I searched
specifically for one. "Majority" and "in rare cases" is the entire published
guarantee. Anything tighter than that in your runbook is something you measured
yourself, and you should say so in the runbook.

### The race

```
t+0s    CI builds, pushes 123456789012.dkr.ecr.eu-west-1…/prod/payments:v1.43.0
t+2s    push completes, ECR emits "ECR Image Action" (PUSH, SUCCESS)
t+3s    CI's next step syncs ArgoCD in BOTH regions to v1.43.0
t+10s   eu-west-1 pods roll. Fine.
t+10s   eu-west-2 pods roll → ImagePullBackOff. The image is not there yet.
t+8m    replication completes. eu-west-2 recovers on kubelet's next retry.
```

Between t+10s and t+8m the standby is **broken**, and if `eu-west-1` fails in
that window you fail over into a region whose pods cannot start. This is not
hypothetical — it is the default behaviour of any pipeline that pushes once and
deploys twice.

Note that the standby usually *self-heals* because kubelet retries. That is what
makes it dangerous: the pipeline goes green, the pods eventually run, and nobody
learns that there is a window. The only symptom is a burst of
`ImagePullBackOff` in a region nobody looks at.

Note also that [[eks-workload-delivery]] already recommends **sync primary
first, soak, then standby** — which, as a side effect, is a decent mitigation
here, because the soak window and the replication window overlap. That is a
happy accident, not a design. Make it deliberate.

### Detecting replication completion

There are two real signals, and you should use both.

**1. `DescribeImageReplicationStatus` — a synchronous poll, called in the source
region.**

```bash
aws ecr describe-image-replication-status \
  --region eu-west-1 \
  --repository-name prod/payments \
  --image-id imageDigest=sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
```

```json
{
  "repositoryName": "prod/payments",
  "imageId": { "imageDigest": "sha256:9f86d0…", "imageTag": "v1.43.0" },
  "replicationStatuses": [
    { "region": "eu-west-2", "registryId": "123456789012", "status": "COMPLETE" }
  ]
}
```

`status` is one of `IN_PROGRESS`, `COMPLETE`, `FAILED`; on failure there is a
`failureCode`. This is the API to gate a pipeline on. A pipeline step:

```bash
# Block until the image is in the standby, or fail the deploy.
# ponytail: fixed 15-min ceiling; if you routinely hit it, the problem is the
# image size or the pipeline shape, not the timeout.
DEADLINE=$(( $(date +%s) + 900 ))
until [ "$(aws ecr describe-image-replication-status --region "$SRC" \
            --repository-name "$REPO" --image-id imageDigest="$DIGEST" \
            --query "replicationStatuses[?region=='$DST'].status | [0]" \
            --output text)" = "COMPLETE" ]; do
  [ "$(date +%s)" -lt "$DEADLINE" ] || { echo "replication to $DST timed out"; exit 1; }
  sleep 10
done
```

**2. The EventBridge `ECR Replication Action` event** — announced
[July 2024](https://aws.amazon.com/about-aws/whats-new/2024/07/amazon-ecr-eventbridge-ecrs-replication-feature/),
emitted **in the destination region** when a replication completes:

```json
{
  "version": "0",
  "detail-type": "ECR Replication Action",
  "source": "aws.ecr",
  "account": "123456789012",
  "time": "2024-05-08T20:44:54Z",
  "region": "us-east-1",
  "resources": ["arn:aws:ecr:us-east-1:123456789012:repository/docker-hub/alpine"],
  "detail": {
    "result": "SUCCESS",
    "repository-name": "docker-hub/alpine",
    "image-digest": "sha256:7f5b2640…",
    "source-account": "123456789012",
    "action-type": "REPLICATE",
    "source-region": "us-west-2",
    "image-tag": "3.17.2"
  }
}
```

Note the shape carefully: the event's top-level `region` is the **destination**,
and `detail.source-region` is where it came from. That is what makes it usable
as a "the standby now has it" trigger — it fires in the region that matters, so
a rule in `eu-west-2` can drive the standby's deploy without any dependency on
`eu-west-1`. Two good uses:

- **Event-driven standby deploy.** Rule in `eu-west-2` on
  `detail-type: ECR Replication Action` + `result: SUCCESS` + your repo prefix →
  Lambda → sync the standby's ArgoCD Application. The standby converges *because*
  the image arrived, rather than hoping it has. This is strictly better than a
  timer and it removes the race entirely.
- **Latency measurement.** Correlate the `ECR Image Action` (PUSH) event in the
  source with the `ECR Replication Action` event in the destination on
  `image-digest`, and emit the delta as a CloudWatch metric. **Now you have your
  own replication-latency distribution** instead of "the majority under 30
  minutes", and you can put a real p99 in the runbook. This is a few dozen lines
  and it is the only way anyone in this estate will ever know the true number.
  See [[observability-multi-region]].

> [!caution] "Events are emitted on a best effort basis"
> AWS states this directly on the
> [ECR EventBridge page](https://docs.aws.amazon.com/AmazonECR/latest/userguide/ecr-eventbridge.html).
> So the event is fine as a *trigger* and fine for *measurement*, but do not
> build a correctness gate that assumes the event always arrives. Gate on
> `DescribeImageReplicationStatus` (authoritative poll); trigger on the event
> (fast path); and have the poller as the backstop.

### Does replication failure alarm?

Not by default. A `FAILED` replication produces a `FAILED` status on
`DescribeImageReplicationStatus` and, per the event schema, a `result` that is
not `SUCCESS`. Nothing creates a CloudWatch alarm for you. **Add one.** An
EventBridge rule matching `detail.result != "SUCCESS"` → SNS → the on-call
channel is about eight lines of Terraform and it is the difference between
"the standby is a release behind" and "the standby has been empty since March".

## Immutable tags and why `:latest` is fatal

### The failure, stated plainly

The standby exists so you can fail over to **the thing you were running**. If the
standby resolves a tag to a different digest than the primary was running, you
have not failed over — you have performed an emergency, untested deploy, during
an incident, with no rollback, under time pressure. That is a strictly worse
outcome than the outage you were mitigating.

Mutable tags make this the *expected* behaviour, not an edge case, because the
two registries are independent mutable key-value stores that only ever receive
*forward* updates.

### How the two registries diverge

1. **Partial replication.** You push `:latest` = digest `A`. It replicates. You
   push `:latest` = digest `B`. Replication is in flight, or the prefix filter
   changed, or it failed. Primary says `latest → B`, standby says `latest → A`.
   Nothing reconciles this, ever. There is no re-sync.
2. **The documented tag-immutability interaction.** From the ECR docs:

   > If tag immutability is enabled on a repository and an image is replicated
   > that uses the same tag as an existing image, the image is replicated but
   > won't contain the duplicated tag. This might result in the image being
   > untagged.

   So `IMMUTABLE` on the destination does not *protect* you from a re-pushed tag;
   it produces an **untagged image** in the standby. An untagged image is not
   pullable by tag. Your Deployment references `:v1.43.0`, the bytes are sitting
   right there in `eu-west-2`, and the pull 404s. Worse, an untagged image is
   exactly what a `tagStatus: untagged` lifecycle rule deletes — see
   [[#Lifecycle policies apply per region]]. This is the sharpest single
   interaction in the note: **immutable tags + a re-pushed tag + an untagged-image
   lifecycle rule in the standby = the image is deleted from the standby while
   the primary happily keeps running it.**
3. **Independent lifecycle policies.** Covered below, but it is a third,
   entirely separate path to divergence.
4. **Deletes do not replicate.** *"Registry replication doesn't perform any delete
   actions or archive actions."* Delete a bad image from the primary and it stays
   in the standby. Over years the two registries' contents drift apart in both
   directions.

### What to do

Two decisions, and they are separable.

**Decision 1: tag mutability on the repository.**

| | `MUTABLE` | `IMMUTABLE` |
|---|---|---|
| Re-push same tag in primary | Overwrites. Registries diverge. | Rejected at push. Pipeline fails loudly. |
| Replication of a duplicate tag | Overwrites in destination | **Image arrives untagged** (per docs) |
| Pull-through cache repos | Required — ECR must be able to update the cached tag. AWS explicitly recommends `MUTABLE` for PTC repos. | Breaks cache refresh: *"Turning on image tag immutability for repositories using a pull through cache rule will prevent Amazon ECR from updating images using the same tag."* |
| Recommendation | Only for pull-through-cache repositories | **Everything you build**, in both regions |

ECR has since added **tag mutability exclusion filters** — you can select
`MUTABLE` with an immutable-tag exclusion list, or `IMMUTABLE` with a mutable-tag
exclusion list. That is the clean way to have an immutable registry that still
lets a `:latest` or `:main` convenience tag float for humans. Configure it in the
repository creation template so replicated repositories inherit it.

**Decision 2: what the Deployment references.**

**Deploy by digest.** This is the recommendation and it is close to
non-negotiable for a two-region estate:

```yaml
# Not this:
image: 123456789012.dkr.ecr.eu-west-2.amazonaws.com/prod/payments:latest
# Not even this:
image: 123456789012.dkr.ecr.eu-west-2.amazonaws.com/prod/payments:v1.43.0
# This:
image: 123456789012.dkr.ecr.eu-west-2.amazonaws.com/prod/payments@sha256:9f86d081884c…
```

A digest is **content-addressed and globally identical across every registry in
the world**. It cannot resolve to a different image in `eu-west-2` than it did in
`eu-west-1`, because if the bytes differ the digest differs. Every divergence
path above is closed at once:

- Partial replication → the pull fails **loudly** (`manifest unknown`) instead of
  succeeding with the wrong image. A loud failure during a drill is a gift.
- The untagged-image trap → **irrelevant**, because you never referenced the tag.
  The image being untagged does not stop a digest pull. (It *does* still expose
  you to an untagged-image lifecycle rule; see below.)
- Independent lifecycle policies → still a risk, but now detectable by the digest
  existence check in [[#Recommendation]].
- `imagePullPolicy` subtleties → gone. A digest reference is immutable, so
  `IfNotPresent` is safe and caching is correct.

The cost is that the pipeline must resolve tag → digest at build time and write
the digest into the manifest. Every sensible CI does this already: `docker
buildx` prints the digest, `crane digest` fetches it, and ArgoCD Image Updater
supports a digest update strategy. It is a small change and it is the single
highest-leverage change in this note after fixing the registry hostname.

> [!tip] Keep a human-readable tag too
> Push **both** the digest-immutable tag (`v1.43.0`, or `sha-<gitsha>`) and
> deploy by digest. The tag is for humans reading `describe-images` at 3am; the
> digest is for the machine. They are not in conflict.

### The `:latest` summary for the runbook

> In a single-region estate, `:latest` is sloppy. In a two-region estate,
> `:latest` is a mechanism for failing over into a build that has never served
> production traffic. The primary and the standby are two independent registries
> and nothing reconciles their tag→digest maps. Deploy by digest.

## Lifecycle policies apply per region

### The mechanic

Lifecycle policies are **per repository, per region**, and they are **not
replicated**:

> Repository policies, including IAM policies, and lifecycle policies aren't
> replicated and don't have any effect other than on the repository they are
> defined for.

So the standby's repositories start with **no lifecycle policy at all** unless a
repository creation template supplies one. Two symmetric failures, and they pull
in opposite directions:

**Failure 1 — no policy in the standby. The registry grows forever.** Combined
with "replication doesn't perform any delete actions", every image ever pushed
accumulates in `eu-west-2` while `eu-west-1` is being pruned. Over a year, the
standby registry becomes **larger and more expensive than the primary**. This is
the one people notice, eventually, on the bill.

**Failure 2 — the policy is copied verbatim and it deletes what the primary is
still running.** This is the one that hurts, and the brief asks about it
directly. Verified: the mechanism is real. Consider the common rule:

```json
{
  "rules": [{
    "rulePriority": 1,
    "description": "Expire untagged images after 14 days",
    "selection": {
      "tagStatus": "untagged",
      "countType": "sinceImagePushed",
      "countUnit": "days",
      "countNumber": 14
    },
    "action": { "type": "expire" }
  }]
}
```

Now recall the documented tag-immutability behaviour: a replicated duplicate tag
arrives in the destination **untagged**. Fourteen days later, this rule deletes
it. The primary is still running that digest. Your standby has silently lost the
image for the service that is currently in production, and nothing alarms.

The `imageCountMoreThan` variant is just as dangerous by a different route:

```json
{ "selection": { "tagStatus": "tagged", "tagPrefixList": ["v"],
                 "countType": "imageCountMoreThan", "countNumber": 10 },
  "action": { "type": "expire" } }
```

"Keep the 10 most recent" is evaluated **independently in each region against
that region's contents**. If the standby received a different subset of pushes —
because replication was enabled later, or a prefix filter changed, or some
pushes failed — the two regions' "10 most recent" sets are different, and the
standby can expire a digest the primary still considers recent. The policies are
identical; the inputs are not.

**Failure 3 — `sinceImagePulled` is catastrophic in a standby, and it is the
rule most likely to be recommended to you.** This is the sharpest finding in the
whole lifecycle story and it is specific to active/passive.

From the [lifecycle policy evaluation rules](https://docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html):

> With `countType = sinceImagePulled`, all images whose `last_recorded_pulltime`
> is older than the specified number of days based on `countNumber` are
> archived. If an image was never pulled, the image's `pushed_at_time` is used
> instead of the `last_recorded_pulltime`.

Now consider what "last pulled" means in a standby region. **Nothing pulls from
the standby's registry.** The standby cluster runs zero workload pods; that is
the definition of a pilot light. So every image in `eu-west-2` has either never
been pulled, or was last pulled at the migration backfill. A
`sinceImagePulled: 30 days` rule — a completely reasonable, widely-recommended
"clean up images nobody uses" policy — will, in a standby region, eventually
select **every image in the registry**, including the one the primary is running
right now.

The rule is not wrong. Its input signal is. "Nobody has pulled this in 30 days"
means "this is dead" in an active region and "this is a standby doing its job"
in a passive one. Copying the primary's lifecycle policy to the standby imports
a heuristic whose premise is false there.

> [!danger] Never use `sinceImagePulled` in a standby region
> It is the one lifecycle rule whose failure mode is *deleting the entire DR
> registry*, and it will look correct in review because it looks correct in the
> primary. If a platform-wide policy template exists, it needs an explicit
> standby carve-out. This is also a second, independent reason to run the
> pre-pull DaemonSet from
> [[eks-workload-delivery#The strongest option: pre-pull onto the standby's nodes]]:
> a DaemonSet that pulls the production images in the standby keeps
> `last_recorded_pulltime` fresh, which defuses the rule even if someone
> re-introduces it. Defence in depth, for free.

### Rule evaluation semantics worth knowing

Four statements from the same page that change how you write a standby policy:

- **"All rules are evaluated at the same time, regardless of rule priority. After
  all rules are evaluated, they are then applied based on rule priority."** Lower
  priority number wins. So a protective high-priority rule genuinely shields
  images from a lower-priority sweeping rule.
- **"An image is expired or archived by exactly one or zero rules."** There is no
  compounding.
- **"Only one rule selecting a specific storage class is allowed to select
  untagged images."** You cannot write two untagged rules to hedge.
- **"When reference artifacts are present in a repository, Amazon ECR lifecycle
  policies automatically expire or archive those artifacts within 24 hours of the
  deletion or archival of the subject image."** Signatures and SBOMs stored as
  OCI 1.1 referrers are tied to their subject image's fate rather than being
  swept independently — which is a real argument for referrer-based signing over
  cosign's derived-tag layout. See [[#Image signing and attestations]].
- **"A lifecycle policy rule may specify either `tagPatternList` or
  `tagPrefixList`, but not both"**, and either may only be used when `tagStatus`
  is `tagged`. `tagPatternList` supports wildcards, max four `*` per string.
- **"If an image is referenced by a manifest list, it cannot be expired or
  archived without the manifest list being deleted or archived first."** Useful:
  multi-arch images are protected from having their per-platform children swept
  out from under them.

### The fix

1. **Give the standby a lifecycle policy, via a repository creation template**,
   so replicated repositories are born with it. (Templates apply at repository
   *creation* only — see the caveat below.)
2. **Make the standby's policy strictly more conservative than the primary's.**
   Longer retention, higher counts. The standby's storage is cheap
   ($0.10/GB-month); a missing image at failover is not. Concretely: if the
   primary keeps 10 tagged releases, the standby keeps 30; if the primary expires
   untagged after 14 days, the standby expires after 90 — or, better, **not at
   all** for the `prod/` prefix.
3. **Never write an `untagged` expiry rule in the standby for production
   repositories.** The one thing replication reliably produces in the destination
   is untagged images. An untagged-expiry rule in the standby is a rule designed
   to delete exactly the artefacts replication creates. If you deploy by digest,
   untagged images in the standby are *load-bearing*.
4. **Verify with `--dry-run` before applying anything.** `aws ecr
   start-lifecycle-policy-preview` shows exactly which images a policy would
   expire, without expiring them:
   ```bash
   aws ecr start-lifecycle-policy-preview --region eu-west-2 \
     --repository-name prod/payments \
     --lifecycle-policy-text file://standby-lifecycle.json
   aws ecr get-lifecycle-policy-preview --region eu-west-2 \
     --repository-name prod/payments \
     --query 'previewResults[].{digest:imageDigest,tags:imageTags,rule:appliedRulePriority}'
   ```
   Run this against the standby before every lifecycle policy change, and
   cross-check the output against the digests the primary is running. This is the
   cheapest possible guard on the sharpest edge in this note.
5. **The digest-existence check** from [[#Recommendation]] catches it after the
   fact, on a cron. Belt and braces.

> [!warning] Repository creation templates only apply at creation
> *"The settings in a repository creation template are only applied during
> repository creation and don't have any effect on existing repositories or
> repositories created using any other method."* So a template added **after**
> replication has already created the destination repositories does nothing to
> them. If you turn replication on first and add the template second — which is
> the natural order — you get a set of destination repositories with default
> settings (`MUTABLE`, `AES256`, no lifecycle policy, no repository policy) and a
> template that will never touch them. **Create the templates in the destination
> region before you enable replication in the source.** Ordering matters and it
> is not enforced by anything.

## KMS encryption per region

### The constraints

- A repository's **encryption configuration is set at creation and cannot be
  changed**: *"Repository Encryption Configuration can't be changed after a
  repository is created."* This is a `ForceNew` in Terraform — see
  [[#ForceNew and replacement risk]].
- **KMS keys are regional**: *"When you use KMS encryption with your own KMS key,
  the key must exist in the same Region as your repository."* There is no
  cross-region key use for ECR. The standby needs its **own** key, or a
  **multi-Region key** replica in `eu-west-2`.
- ECR does not use the key directly per operation; it creates **two KMS grants**
  on the key at repository creation, with the ECR repository as grantee, and
  retires them at repository deletion. Do not revoke those grants — AWS: *"To
  revoke access rights, you should delete the repository rather than revoking the
  grant."*
- The encryption context is `{"aws:s3:arn": …, "aws:ecr:arn": "arn:aws:ecr:<region>:<acct>:repository/<name>"}`.
  The ECR ARN is **region-qualified**, which matters if you write
  `kms:EncryptionContext:` conditions into a key policy — a condition written
  against the primary's ARN will not match in the standby.
- **`kms:ViaService` is region-qualified too.** AWS's guidance for locking an ECR
  key down is the `kms:ViaService` condition key with the value
  `ecr.<region>.amazonaws.com`. Copy the primary's key policy to the standby
  verbatim and the condition names `ecr.eu-west-1.amazonaws.com` on a key in
  `eu-west-2` — so **every** ECR operation against it is denied, and the symptom
  is repository creation failing, which is replication failing, silently. This is
  the third member of the region-qualified-ARN family alongside the image
  reference and the IAM policy; see [[aws-kms]] and [[aws-iam]].
- **Who needs which KMS permission is split, and the split is non-obvious.**
  `kms:RetireGrant` **must** be on the IAM policy of the principal creating the
  repository; `kms:CreateGrant` and `kms:DescribeKey` may live on either the key
  policy or that IAM policy. In a templated monorepo where the CI role creates
  repositories, that means the CI role — and the creation template's
  `custom_role_arn` — both need this, in the standby region, against the standby
  key.

### What this means for replication

The destination repository is created by ECR on your behalf. Unless a
**repository creation template** in the destination specifies `KMS` and a key,
ECR creates it with the default `AES256`. So:

> **A KMS-encrypted primary silently becomes an AES256-encrypted standby.**

Whether that is a problem is a compliance question, not a technical one — the
images still work. But if your control framework says "customer-managed keys for
all image storage", your DR region quietly fails the control, and the only way to
fix it afterwards is to **delete and recreate every destination repository**,
because encryption configuration is immutable. Get this right before the first
replication, not after.

The destination template must carry:

```hcl
encryption_configuration {
  encryption_type = "KMS"
  kms_key         = aws_kms_key.ecr_standby.arn   # a key in eu-west-2
}
```

…and, because the template uses KMS, it **must** specify a
`custom_role_arn`. AWS: *"This role must be provided when using repository tags
and/or KMS in the template, otherwise the repository creation will fail."* That
role is assumed by ECR when creating the repository and needs
`kms:CreateGrant`, `kms:DescribeKey`, `kms:RetireGrant` on the destination key
plus `ecr:CreateRepository`, `ecr:PutLifecyclePolicy`, `ecr:SetRepositoryPolicy`,
`ecr:TagResource`.

**This is the quietest failure mode in the whole ECR story**: a template that
specifies KMS without a valid `custom_role_arn` causes **repository creation to
fail**, which causes **replication to fail**, and the only evidence is a `FAILED`
status on `DescribeImageReplicationStatus` for images nobody is checking. It is
precisely why the EventBridge failure alarm above is worth the eight lines.

### Single-region keys vs a multi-Region KMS key

| | Two independent regional keys | One multi-Region key (primary + replica) |
|---|---|---|
| Key material | Different | Same material, same key ID across regions |
| Key policy management | Two policies to keep in sync | One logical key, replicas managed per region |
| Blast radius of a key policy mistake | One region | Both |
| Terraform | `aws_kms_key` × 2, two aliases | `aws_kms_key` with `multi_region = true` + `aws_kms_replica_key` |
| Relevance to ECR specifically | **Fully sufficient.** ECR never needs to decrypt a foreign region's ciphertext — replication re-encrypts under the destination's key. | Works, but buys nothing ECR needs. |

**Recommendation: two independent regional keys.** Multi-Region KMS keys exist
to let one region decrypt ciphertext produced by another, and ECR replication
never requires that — each region encrypts its own copy under its own key. Using
an MRK here adds coupling for no benefit. (This is a different answer from some
other services in the estate; see [[aws-kms]] for where MRKs genuinely earn
their place, and note the contrast deliberately.)

### `ca-west-1` note

Calgary supports KMS and ECR (endpoints confirmed below), so nothing here is a CA
gap. The CA gap is elsewhere — see [[#`ca-west-1` parity check]].

## ECR throttling during a mass pull

The scenario the brief asks about: `eu-west-1` dies, the `eu-west-2` node group
goes 0 → 6, Karpenter adds another 20 nodes, and several hundred pods all pull
at once. Does ECR throttle you out of your RTO?

### The published quotas

All from
[ECR service quotas](https://docs.aws.amazon.com/AmazonECR/latest/userguide/service-quotas.html),
all **per region, per account**:

| Quota | Default | Adjustable | Called when |
|---|---|---|---|
| `GetAuthorizationToken` | **500/sec** | Yes | Once per node per ~12h (token TTL) by the ECR credential provider |
| `BatchGetImage` | **2,000/sec** | Yes | Once per image pull — fetches the manifest |
| `GetDownloadUrlForLayer` | **3,000/sec** | Yes | **Once per layer not already cached on the node** |
| `BatchCheckLayerAvailability` | 1,000/sec | Yes | Push path |
| `PutImage` | **10/sec** | Yes | Push path — the tightest quota in the table |
| `InitiateLayerUpload` | 100/sec | Yes | Push path |
| `UploadLayerPart` | 500/sec | Yes | Push path |
| `CompleteLayerUpload` | 100/sec | Yes | Push path |

### Doing the arithmetic for a real failover

Take a deliberately unfavourable case: **500 pods** starting simultaneously,
each a distinct image, each **15 layers**, on **30 cold nodes**.

- `GetAuthorizationToken`: ~30 calls (one per node), spread over the boot window.
  Against 500/sec, **irrelevant**.
- `BatchGetImage`: 500 calls. If they all landed in one second that is 25% of the
  2,000/sec quota. They will not — pods start over 60–120 seconds. **Irrelevant.**
- `GetDownloadUrlForLayer`: 500 × 15 = **7,500 calls**. Against 3,000/sec, this
  needs ≥ 2.5 seconds of wall clock to not throttle. Real pulls spread over
  tens of seconds. **Irrelevant.**

**Verdict: the ECR API quotas are not the RTO risk.** They are an order of
magnitude above what a 500-pod cold start generates. Anyone who tells you ECR
throttling is the failover bottleneck has not done the arithmetic.

What *is* the bottleneck, in order:

1. **Bandwidth.** 500 pods × 400 MB of uncached layers is 200 GB pulled in a few
   minutes. This is the real constraint and it is why the **pre-pull DaemonSet**
   and **Bottlerocket data volume** recommendations in
   [[eks-workload-delivery#The strongest option: pre-pull onto the standby's nodes]]
   matter far more than any quota.
2. **NAT gateway.** If the standby's pulls go through NAT rather than a VPC
   endpoint you are paying NAT data-processing charges on every byte *and*
   funnelling a burst through a NAT that has been idle for months. NAT gateways
   scale, but this is pure waste — see [[aws-vpc-networking]].
3. **`PutImage` at 10/sec** — the genuinely tight one, and it is on the **push**
   path, not the pull path. Irrelevant during failover; relevant during a
   **backfill** (`crane copy` calls `PutImage` once per manifest, and a multi-arch
   image is several). A parallel backfill script with 50 workers **will** throttle
   here. Keep backfill concurrency modest (8–16) and expect
   `ThrottlingException` retries.

### What to do anyway

- **Raise the quotas in the standby regions as part of the standby quota audit**
  ([[aws-eks#Gotchas]] item 9). They are adjustable and the request is free. Even
  though the arithmetic says you are fine, a quota ticket takes days and cannot
  be filed during an incident.
- **Alarm on the ECR usage metrics.** ECR publishes CloudWatch usage metrics
  matching each of these quotas, and Service Quotas can create the alarm for you.
  Set them at 70% in the primary; in the standby they will read zero until the
  drill, which is itself a useful signal.
- **Do not let the standby's pulls be the first time anything has pulled there.**
  The pre-pull DaemonSet covers this, and it is the recommendation in the
  companion note.

## Pull-through cache rules

### What they are for

A pull-through cache (PTC) rule maps a prefix in *your* registry to an **upstream
registry**, and populates on first pull. Supported upstreams, per the
[PTC docs](https://docs.aws.amazon.com/AmazonECR/latest/userguide/pull-through-cache.html):

| Upstream | Auth |
|---|---|
| ECR Public, Kubernetes registry (`registry.k8s.io`), Quay | None |
| Docker Hub, Azure CR, GitHub CR, GitLab CR (SaaS only), Chainguard | Secrets Manager secret, name must start `ecr-pullthroughcache/`, same account + region |
| **Another ECR private registry** | IAM role (only needed cross-account) |

Refresh semantics: on a pull of a cached tag, *"Amazon ECR checks whether it has
validated the image against the upstream registry within the last 24 hours"* and
only then goes upstream. Referrer artefacts (signatures, SBOMs) use a **6-hour**
window.

### Do they help or hurt at failover?

**Both, depending on which upstream.** This is the distinction the companion note
compresses into one row and it deserves separating properly.

**PTC with a third-party upstream (Docker Hub, `registry.k8s.io`, Quay, ECR
Public) — HELPS, substantially.** A PTC rule in the **standby region** means
your third-party base images, sidecars, controllers and operators are served from
`eu-west-2`'s own ECR rather than from the public internet. During a failover you
are not depending on Docker Hub being up, or on your egress to it, or on a rate
limit. **Set these up in both regions.** Two independent PTC rules, two
independent caches — they do not share state and they do not need to.

**PTC with the primary ECR registry as upstream — HURTS.** ECR-to-ECR PTC
([announced March 2025](https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-ecr-pull-through-cache/))
makes `eu-west-2` fetch from `eu-west-1` on a cache miss. At failover, `eu-west-1`
is the dead region. Every uncached image is unfetchable. This is a cost
optimisation for active/active, not a DR mechanism, and it is **the wrong tool**
for this estate's own images. Use replication for those.

### The Docker Hub rate-limit amplifier

This is the cheap-to-miss outage amplifier the brief names, and it is real.

Docker Hub's [published limits](https://docs.docker.com/docker-hub/usage/):

| Tier | Pulls | Window |
|---|---|---|
| **Unauthenticated** | **100 per IPv4 address or IPv6 /64 subnet** | 6 hours |
| Authenticated, Personal | 200 | 6 hours |
| Pro / Team / Business | Unlimited | — |

Now the failover scenario. 30 cold nodes come up in `eu-west-2`. Every node pulls
the CNI, kube-proxy, CoreDNS, the Karpenter controller, the ALB controller, ESO,
the ArgoCD agent, the observability agents, plus every application sidecar.
Several of those are Docker Hub images in most estates. **All 30 nodes egress
through the same NAT gateway, and therefore present Docker Hub with the same
source IP.** Thirty nodes × four Docker Hub images = 120 pulls from one IP —
**over the 100-pull unauthenticated limit**, in the first two minutes of your
failover.

Docker Hub then returns `429 Too Many Requests`. kubelet reports
`ImagePullBackOff` with a toomanyrequests error, backs off exponentially, and
your RTO is now bounded by a **six-hour** rate-limit window on a third party you
have no relationship with. The primary region never hit this because its nodes
warmed up gradually over months and its images were long since cached on disk.

This is exactly the class of dependency [[third-party-saas-dependencies]] exists
to catalogue. Three mitigations, and you should do the first one regardless:

1. **PTC rule for Docker Hub in every region, authenticated with a paid Docker
   account, plus rewrite every Docker Hub reference to the ECR PTC prefix.** The
   pull then comes from your own ECR; Docker Hub sees one authenticated pull per
   image per 24 hours instead of 30 anonymous ones per node.
   ```
   docker.io/library/redis:7        →  123456789012.dkr.ecr.eu-west-2.amazonaws.com/docker-hub/library/redis:7
   ```
   Note the cost: PTC-cached images are ordinary ECR images and you pay
   $0.10/GB-month for them, in both regions. Trivial against the risk.
2. **Vendor third-party images into your own repositories** at build time and
   replicate them like your own. Heavier (you own the update treadmill) but it
   removes the third party from the runtime path entirely. Best for the small set
   of images that are genuinely critical-path.
3. **Pre-pull them onto the standby's system nodes** via the DaemonSet from the
   companion note, so at least the always-on nodes never need Docker Hub.
   Karpenter-launched surge nodes still will — which is why this is a supplement,
   not a fix.

> [!warning] PTC's first pull needs a route to the internet
> From the docs: *"When an image is pulled using the pull through cache rule for
> the first time a route to the internet may be required… if you've configured
> Amazon ECR to use an interface VPC endpoint using AWS PrivateLink then you need
> to ensure the first pull has a route to the internet."* A fully private standby
> VPC with no egress cannot populate a cold PTC cache. **Warm the standby's PTC
> cache while the primary is healthy** — pull every third-party image into the
> standby's cache once, from a box with egress, and it is then served locally
> forever. Add it to the migration checklist; it is a one-time action that is
> impossible to perform during the incident.

Two more PTC facts worth knowing here:

- **Lambda cannot pull via a PTC rule** (*"AWS Lambda doesn't support pulling
  container images from Amazon ECR using a pull through cache rule"*). If any
  container-image Lambdas exist, they need real repositories. See [[aws-lambda]].
- **Do not push into a PTC repository.** *"Pushing images directly to a pull
  through cache repository is not a supported pattern"* — it causes stale reads
  until the cache refreshes or you delete the image. Keep PTC prefixes
  (`docker-hub/`, `quay/`, `k8s/`) strictly separate from your own (`prod/`), and
  make the replication rule's prefix filter enforce the split.

## Registry and repository policies

Three distinct policy surfaces, regularly confused:

| Policy | Scope | Terraform | Replicated? | Used for |
|---|---|---|---|---|
| **Registry permissions policy** | Registry (account + region) | `aws_ecr_registry_policy` | N/A (per-registry) | Cross-*account* replication (`ecr:ReplicateImage`, `ecr:CreateRepository`), cross-account PTC |
| **Repository policy** | One repository | `aws_ecr_repository_policy` | **No** | Cross-account pull/push of that repository |
| **IAM identity policy** | Principal | `aws_iam_role_policy` | N/A (IAM is global) | What your nodes/CI are allowed to do |

### Cross-account pull from the standby

If the standby cluster's nodes live in a different account from the registry —
common with a shared-services registry account — the node role needs **both** an
identity policy and a repository policy in the **standby region's** registry.
Both halves are required; either alone yields `AccessDenied`.

Node role identity policy (attach to the EKS node role in the standby account):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchCheckLayerAvailability"
      ],
      "Resource": "arn:aws:ecr:eu-west-2:111122223333:repository/prod/*"
    }
  ]
}
```

Note `ecr:GetAuthorizationToken` **must** be `Resource: "*"` — it is a
registry-level action with no resource to scope to. And note the ARN is
**region-qualified**: an identity policy that was written once for `eu-west-1`
and copied does not grant anything in `eu-west-2`. **Grep every IAM policy in the
Terraform repo for `arn:aws:ecr:` and check whether the region segment is
hardcoded or interpolated.** This is the [[aws-iam]] version of the
image-reference problem and it has the same shape: the object is global, the ARN
inside it is not.

Repository policy in the standby registry (via the repository creation template,
so replicated repositories are born with it):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowStandbyClusterNodesToPull",
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::444455556666:root" },
    "Action": [
      "ecr:BatchGetImage",
      "ecr:GetDownloadUrlForLayer",
      "ecr:BatchCheckLayerAvailability"
    ]
  }]
}
```

**The failure to plan for:** repository policies are not replicated, so a
repository created by replication has **no** repository policy. In a
cross-account estate that means the standby's nodes cannot pull the replicated
image, even though it is right there. The repository creation template is the
only sane fix, and it only works if it existed **before** the repository was
created.

## `ca-west-1` parity check

The brief asks for this explicitly, and Calgary has already failed the check for
Cognito MRR and OpenSearch CCR. **For ECR it passes.**

| Check | Finding | Source |
|---|---|---|
| ECR API endpoint in `ca-west-1`? | **Yes** — `api.ecr.ca-west-1.amazonaws.com`, `ecr.ca-west-1.amazonaws.com` | [ECR endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/ecr.html) |
| Docker/OCI endpoint? | **Yes** — `dkr.ecr.ca-west-1.amazonaws.com` | Same |
| Dual-stack endpoints? | **Yes** — `ecr.ca-west-1.api.aws`, `dkr-ecr.ca-west-1.on.aws` | Same |
| FIPS endpoint? | **No.** Only `us-east-*`, `us-west-*` and GovCloud list `ecr-fips.*`. `ca-central-1` doesn't have one either, so this is **not** a regression for the CA pair. | Same |
| Replication supported? | **No region-specific exclusion found.** Replication is a registry-level feature of ECR, available where ECR is. AWS publishes no per-region replication support matrix. Treat as "assume yes, verify with a real `put-replication-configuration` before relying on it." | — |
| Service quotas | Quota table says *"Each supported Region"* for every entry — no per-region variation published. | Same |

### The real `ca-west-1` finding: opt-in

`ca-west-1` launched 20 Dec 2023, after the 20 March 2019 cutoff, so it is an
**opt-in region** — confirmed in the
[opt-in region table](https://docs.aws.amazon.com/general/latest/gr/rande-manage.html).
`ca-central-1`, `eu-west-1`, `eu-west-2`, `us-east-1` and `us-west-2` are all
**default** regions.

This matters for ECR specifically, because the replication documentation says:

> For cross-Region replication to occur, **both the source and destination
> accounts must be opted-in to the Region** prior to any replication actions
> occurring within or to that Region.

So for the CA pair, and **only** the CA pair:

1. **Every account** that participates in replication into Calgary must have
   `ca-west-1` enabled — not just the account running the workload. In a
   multi-account organisation that is an explicit `account:EnableRegion` per
   member account, or an org-wide enablement from the management account.
2. Enabling is asynchronous and *"takes a few minutes for most accounts, but can
   sometimes take several hours"*. Not something to discover during a cutover.
3. An account can have **6 region-opt requests in flight**; an organisation **50**
   (the docs also state 20 in the CLI section — the two numbers disagree on that
   page; treat 20 as the safe assumption). For a large org this is a **batched,
   multi-day task**, not a checkbox.
4. A disabled region **does not delete resources and does not stop charges** — so
   a half-completed Calgary rollout can leave a replicated registry accruing
   storage cost in an account that cannot see it.

**Recommendation: verify `ca-west-1` opt-in status across every account in the
organisation before any CA-pair design work, with
`aws account get-region-opt-status --region-name ca-west-1` per account.** It is
a prerequisite with a multi-day lead time and it is invisible in
`terraform plan`. Carry it into [[region-pair-selection]] alongside the EKS
instance-family check.

**Net: ECR is not a reason to abandon the CA pair.** The instance-family question
from [[aws-eks]] and the Cognito/OpenSearch gaps remain the live risks.

## ECR Public vs private

Briefly, because it is not a DR mechanism and should not be mistaken for one.

| | ECR Public | ECR Private |
|---|---|---|
| Registry | `public.ecr.aws`, a **single global registry** | Per account, per region |
| Where the API lives | `us-east-1` only (`api.ecr-public.us-east-1.amazonaws.com`) | Every region |
| Terraform | `aws_ecrpublic_repository` — **provider must be aliased to `us-east-1`** | `aws_ecr_repository` |
| Replication | N/A — it is one registry | Registry-level rules |
| Cost | 50 GB/month storage free to all; 500 GB/month anonymous egress to internet free; 5 TB/month authenticated; **unlimited free transfer to AWS compute** | $0.10/GB-month + data transfer |

**Why it is not a DR answer:** you would be publishing your production images
publicly, the control plane is `us-east-1`-only (so a `us-east-1` event affects
your ability to manage it from any region), and pulls are rate-limited for
anonymous clients. It is a distribution mechanism for open-source artefacts.

**Where it does touch this design:** ECR Public is a *supported PTC upstream*, and
several images you depend on (the EKS add-on images, for instance) are served
from ECR Public or from `registry.k8s.io`. A PTC rule for `public.ecr.aws` in each
standby region is cheap and removes another cross-region/internet dependency at
failover. Do it alongside the Docker Hub rule.

One `us-east-1` note for the estate: the `aws_ecrpublic_repository` resource
requires a `us-east-1`-aliased provider. That puts it in the same bucket as
CloudFront certificates from [[aws-acm]] — a resource that must live in
`us-east-1` regardless of which pair you are building. Worth a line in
[[provider-aliases-vs-separate-stacks]].

## Image signing and attestations

### Does a signature survive replication?

**Yes, if your signing scheme stores signatures as OCI artefacts in the same
repository — which both major schemes do — and if the replication rule's prefix
filter covers them.** The mechanism, and the caveat, are worth being precise
about.

**Sigstore / cosign** stores a signature as a separate image whose tag is derived
from the signed image's digest: `sha256-<digest>.sig`. It is an ordinary image in
the same repository. A registry-level replication rule that matches the
repository therefore replicates the `.sig` artefact on its own push, as a
separate replication action, on its own timeline. Two consequences:

- The signature and the image **replicate independently**. There is a window in
  which the image is in the standby and its signature is not. If the standby runs
  an admission controller with `enforce` mode, pods are **rejected** — the image
  is present and the deploy still fails. This is a real, non-obvious failover
  hazard.
- If you copy images with `crane copy` during the backfill, the `.sig` tag is a
  *different tag* and is **not** copied by a digest or single-tag copy. Use
  `regctl image copy --digest-tags` (which explicitly carries digest-derived
  tags), `cosign copy` (which knows about the signature layout), or `crane
  copy --all-tags` on the repository. **A backfill that copies images but not
  signatures produces a standby that is unpullable-by-policy.**

**AWS Signer / Notation (notary v2)**, and cosign v3, store signatures using the
OCI 1.1 **referrers** API rather than derived tags — the registry tracks the
subject relationship server-side instead of requiring a derived tag.

**Verified: ECR replication does propagate referrers.** The AWS Open Source Blog
post on [OCI 1.1 support in ECR](https://aws.amazon.com/blogs/opensource/diving-into-oci-image-and-distribution-1-1-support-in-amazon-ecr/)
states that ECR's replication feature replicates referrers to configured
destinations on push, so image signatures, SBOMs and other referrers are present
in any repository you replicate images to, across accounts or regions. OCI 1.1
support is available in all commercial Regions, so this holds for all three
pairs including `ca-west-1`.

Two follow-on facts from the same post, both of which matter here:

- **Lifecycle policies understand referrers.** Reference artefacts pointing at a
  live image are protected from expiry by lifecycle rules until the subject image
  is deleted, and are cleaned up within 24 hours of the subject's deletion. This
  meaningfully softens [[#Lifecycle policies apply per region]] *for referrer-based
  signing* — though not for cosign's older derived-tag layout, where the `.sig`
  artefact is an ordinary image with no protected relationship.
- **Referrers still replicate as their own push**, so the ordering window above
  remains real: the image can land in the standby moments before its signature
  does.

> [!note] This resolves an open item, and it changes the recommendation
> An earlier draft of this note recorded the referrer question as unverified.
> It is now verified in AWS's favour. **If you are choosing a signing scheme for
> a two-region estate, prefer a referrers-based one (Notation/AWS Signer, or
> cosign v3 in OCI 1.1 mode) over cosign's legacy derived-tag layout** — the
> referrer travels with replication and is protected by lifecycle policy, and
> the `.sig` tag is neither. That is a real multi-region argument for a choice
> usually made on other grounds. Still test it in the first drill:
> `aws ecr list-image-referrers` against the destination, or `notation verify`.

### Recommendation

- If you sign, **verify in the standby continuously** — the DR canary pod from
  [[aws-eks#Warm standby shape]] should be subject to the same admission policy
  as production, so signature-replication failures surface while the primary is
  healthy rather than at 3am.
- **Set admission policy failure mode deliberately.** An `enforce`-mode policy in
  the standby that cannot verify because a signature did not replicate converts a
  regional outage into a total outage. There is a genuine fork here and it is a
  security decision, not an availability one: see [[#Decisions to make]].
- Whatever you do, **do not discover the signature question during the
  failover.** Test it in the first drill.

## Warm standby shape

ECR is the cheapest warm standby in this vault, because the "warm" part is just
bytes at rest. There is no capacity to pre-provision, no control plane to keep
alive, nothing to scale to zero. What has to exist in `eu-west-2` while
`eu-west-1` is healthy:

| Thing | State while primary is healthy | Cost while idle | Provisioned at failover? |
|---|---|---|---|
| Registry | Exists implicitly — every account has one per Region | $0 | No, it is already there |
| Repositories (`prod/*`) | Created by replication (or by the backfill), one per service | $0 for the repository itself | **No — must pre-exist** |
| Images | Every digest the primary is currently running, plus the last N releases for rollback | Storage at $0.10/GB-month | **No — this is the whole point** |
| Repository creation template | Present in the standby **before** replication is enabled | $0 | No |
| Lifecycle policy (via the template) | Present, and more conservative than the primary's | $0 | No |
| Repository policy (via the template) | Present, if cross-account | $0 | No |
| KMS key + alias | Present, with the grants ECR created at repository creation | ~$1/key/month + requests | No |
| Pull-through cache rules (Docker Hub, ECR Public, `registry.k8s.io`, Quay) | Present **and warmed** — at least one pull of every third-party image has already happened | Storage of the cached layers | **No — a cold PTC cache needs internet egress the standby may not have** |
| Secrets Manager secret for the Docker Hub PTC credential | Present in `eu-west-2`, named `ecr-pullthroughcache/...` | ~$0.40/month | No |
| VPC endpoints: `ecr.api`, `ecr.dkr`, S3 gateway | Present | ~$7/month/interface endpoint/AZ | No |
| Raised service quotas | Requested and granted | $0 | **Cannot be done at failover — tickets take days** |
| The digest-existence check (cron Lambda) | Running, reporting to the `dr_readiness` dashboard | Pennies | N/A |
| Replication-failure EventBridge rule → SNS | Armed | $0 | N/A |

**Nothing here is scaled to zero, because nothing here scales.** That is a
genuine advantage: unlike [[aws-eks]]'s pilot light, there is no "will it scale
in time" question for ECR. The entire risk is *correctness* — is the right
digest present, under the right name, pullable by the right principal — and
correctness does not improve under time pressure. Everything in the table above
must be true continuously, and the only honest way to know it is true is the
digest-existence check plus the pre-pull DaemonSet from
[[eks-workload-delivery#The strongest option: pre-pull onto the standby's nodes]],
which is simultaneously a warm cache and a continuous pull test.

> [!tip] The standby's ECR is "warm" in a way the standby's nodes are not
> Replication warms the **registry**. It does nothing for the **nodes**. At
> failover a node coming up from zero still pulls hundreds of megabytes before
> the first container starts. Registry-warm is necessary and not sufficient;
> node-warm is what buys you the RTO. Both notes need reading together.

> [!note] Stateful workloads pull images too
> If [[eks-stateful-workloads]] concludes that operators (a Postgres operator, a
> Redis operator, a CSI driver) run in the standby, their images and their
> sidecars are part of the replication scope too — and they are the images most
> likely to sit outside the `prod/` prefix filter, because they come from
> Quay, `registry.k8s.io` or a vendor registry. Check the prefix filter against
> the *actual* list of images the standby needs to start, not against the list
> of services the company builds.

## Terraform implementation

### The shape, and why it is not the same shape as EKS

[[aws-eks]] uses *one root module per region-pair*, two module calls, two
provider aliases. ECR **cannot** use exactly that shape for everything, because
its resources fall into two structurally different classes:

| Class | Resources | Cardinality | Who owns it |
|---|---|---|---|
| **Registry-scoped singletons** | `aws_ecr_replication_configuration`, `aws_ecr_registry_policy`, `aws_ecr_registry_scanning_configuration`, `aws_ecr_repository_creation_template`, `aws_ecr_pull_through_cache_rule` | **Exactly one per account per Region** (templates and PTC rules are one per prefix, but they share one namespace) | One, and only one, root module per account+region |
| **Per-repository** | `aws_ecr_repository`, `aws_ecr_repository_policy`, `aws_ecr_lifecycle_policy` | One per service, times two regions | The per-service module, or replication |

Collapsing these into one module is the mistake. A `modules/ecr-repository`
that teams instantiate fifty times **must not** contain a replication
configuration: the second instance does not add a rule, it **replaces the whole
registry configuration**, and the last `apply` silently wins. In a cookiecutter
monorepo where each service owns a directory, this is a race condition between
teams that `terraform plan` will not warn you about.

So: **two modules, and a hard rule about which one owns what.** This is the
"account-and-region-scoped singleton vs per-workload resource" split that
[[module-patterns]] covers generally; ECR is the sharpest example of it in the
estate, because the singleton silently overwrites rather than erroring.

```
terraform/
  modules/
    ecr-registry-settings/     # registry-scoped singletons. ONE instantiation per account+region.
    ecr-repository/            # per-service repositories. Many instantiations.
  live/
    prod-eu/
      registry/                # <- the ONLY place ecr-registry-settings is called for prod-eu
        main.tf
      services/
        payments/main.tf       # <- calls ecr-repository
        ledger/main.tf
```

### Providers

Same two aliases as [[aws-eks]], so the cookiecutter template is unchanged. See
[[provider-aliases-vs-separate-stacks]] for the general argument; ECR is a
straightforward case for aliases because both halves are small and must move
together.

```hcl
terraform {
  required_version = ">= 1.5.7"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.0" }
  }
}

provider "aws" {
  alias  = "primary"
  region = var.primary_region        # eu-west-1 | us-east-1 | ca-central-1
  default_tags { tags = local.common_tags }
}

provider "aws" {
  alias  = "standby"
  region = var.standby_region        # eu-west-2 | us-west-2 | ca-west-1
  default_tags { tags = local.common_tags }
}
```

> **`ca-west-1` note:** a provider block pointed at an un-opted-in Region fails
> at plan time with an endpoint/credential error that does not say "opt in".
> Enable the Region in every participating account *before* the first `plan`.
> See [[#The real `ca-west-1` finding: opt-in]].

### `modules/ecr-registry-settings` — the variable surface you actually want

```hcl
# modules/ecr-registry-settings/variables.tf

variable "replicate_to" {
  description = <<-EOT
    Destination regions for registry replication, configured in THIS region.
    Empty list = this region is a standby and replicates nowhere.
    A standby must never replicate back to the primary: replication does not
    chain, but a reciprocal pair would double-store every image for no benefit.
  EOT
  type    = list(string)
  default = []
}

variable "replicate_prefixes" {
  description = <<-EOT
    Repository prefixes to replicate. NEVER default this to "" (the whole
    registry): most registries are majority CI scratch images and PTC caches,
    none of which belong in a DR standby. This is the single biggest cost lever.
  EOT
  type    = list(string)
  default = ["prod/"]
  validation {
    condition     = !contains(var.replicate_prefixes, "")
    error_message = "Refusing to replicate the entire registry. Name the prefixes."
  }
}

variable "destination_registry_id" {
  description = "Account ID of the destination registry. Same account unless you run a shared-services registry."
  type        = string
}

variable "creation_template_prefixes" {
  description = <<-EOT
    Prefixes for repository creation templates. Set in the STANDBY region.
    These are what give replication-created repositories a lifecycle policy,
    a repository policy, KMS encryption and tag-immutability settings.
    Templates only apply at repository CREATION, so these must exist before
    replication is enabled in the primary.
  EOT
  type    = list(string)
  default = ["prod/"]
}

variable "creation_template_role_arn" {
  description = <<-EOT
    Required by AWS whenever a creation template sets resource_tags or KMS
    encryption. If null and either of those is set, repository creation fails
    -- which makes REPLICATION fail, silently.
  EOT
  type    = string
  default = null
}

variable "kms_key_arn" {
  description = "CMK in THIS region for repository encryption. null = AES256 (AWS-owned key)."
  type        = string
  default     = null
}

variable "standby_lifecycle" {
  description = <<-EOT
    Retention for the standby. Deliberately separate from the primary's, and
    deliberately more generous. Storage is $0.10/GB-month; a missing image at
    failover is an outage. Do NOT expire untagged images in a standby holding
    production repositories -- untagged images are exactly what replication
    produces, and under deploy-by-digest they are load-bearing.
  EOT
  type = object({
    keep_tagged_count      = number
    expire_untagged_days   = optional(number)   # leave null for prod prefixes
  })
  default = {
    keep_tagged_count    = 30
    expire_untagged_days = null
  }
}

variable "pull_through_cache" {
  description = "Upstream registries to cache locally. Configure identically in BOTH regions."
  type = map(object({
    upstream_registry_url = string
    credential_arn        = optional(string)   # Secrets Manager secret, SAME region, name must start ecr-pullthroughcache/
  }))
  default = {}
}

variable "cross_account_puller_arns" {
  description = <<-EOT
    Account/role ARNs allowed to pull from repositories created by the template.
    Only needed if the standby cluster lives in a different account from the
    registry. Repository policies are NOT replicated, so without this a
    cross-account standby cannot pull a perfectly-present image.
  EOT
  type    = list(string)
  default = null
}

variable "role" {
  description = "primary | standby. Drives replication direction and retention generosity only."
  type        = string
  validation {
    condition     = contains(["primary", "standby"], var.role)
    error_message = "role must be primary or standby."
  }
}
```

### `modules/ecr-registry-settings/main.tf`

```hcl
# ---------------------------------------------------------------------------
# REPLICATION -- configured in the SOURCE region only.
# Registry-scoped singleton: exactly one of these may exist per account+region.
# A second declaration does not merge, it replaces. Guard it with the
# count below so a standby instantiation is a no-op rather than a fight.
# ---------------------------------------------------------------------------
resource "aws_ecr_replication_configuration" "this" {
  count = length(var.replicate_to) > 0 ? 1 : 0

  replication_configuration {
    rule {
      dynamic "destination" {
        for_each = var.replicate_to
        content {
          region      = destination.value
          registry_id = var.destination_registry_id
        }
      }

      dynamic "repository_filter" {
        for_each = var.replicate_prefixes
        content {
          filter      = repository_filter.value
          filter_type = "PREFIX_MATCH"   # the only value the API accepts
        }
      }
    }
  }
}

# ---------------------------------------------------------------------------
# REPOSITORY CREATION TEMPLATES -- configured in the DESTINATION region.
# This is the ONLY mechanism that gives a replication-created repository a
# lifecycle policy, a repository policy, KMS encryption, or immutable tags.
# It applies at repository CREATION only: adding it later does nothing to
# repositories replication has already created.
# ---------------------------------------------------------------------------
resource "aws_ecr_repository_creation_template" "this" {
  for_each = var.role == "standby" ? toset(var.creation_template_prefixes) : toset([])

  prefix          = each.value          # ForceNew. "ROOT" matches anything unmatched.
  description     = "DR standby settings for ${each.value}* (managed by Terraform)"
  applied_for     = ["REPLICATION"]     # add CREATE_ON_PUSH if CI also pushes here
  custom_role_arn = var.creation_template_role_arn

  # IMMUTABLE_WITH_EXCLUSION lets a convenience tag float for humans while
  # everything else is immutable. Requires provider >= 6.x.
  image_tag_mutability = "IMMUTABLE_WITH_EXCLUSION"
  image_tag_mutability_exclusion_filter {
    filter      = "latest"
    filter_type = "WILDCARD"
  }

  encryption_configuration {
    encryption_type = var.kms_key_arn == null ? "AES256" : "KMS"
    kms_key         = var.kms_key_arn
  }

  repository_policy = var.cross_account_puller_arns == null ? null : data.aws_iam_policy_document.standby_pull[0].json

  lifecycle_policy = jsonencode({
    rules = concat(
      [{
        rulePriority = 1
        description  = "Keep ${var.standby_lifecycle.keep_tagged_count} most recent releases (more generous than the primary)"
        selection = {
          tagStatus     = "tagged"
          tagPatternList = ["v*"]
          countType     = "imageCountMoreThan"
          countNumber   = var.standby_lifecycle.keep_tagged_count
        }
        action = { type = "expire" }
      }],
      # Deliberately conditional and deliberately off by default. See the
      # lifecycle section: an untagged-expiry rule in the standby deletes
      # exactly the artefacts replication creates.
      var.standby_lifecycle.expire_untagged_days == null ? [] : [{
        rulePriority = 2
        description  = "Expire untagged after ${var.standby_lifecycle.expire_untagged_days} days"
        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = var.standby_lifecycle.expire_untagged_days
        }
        action = { type = "expire" }
      }]
    )
  })
}

# ---------------------------------------------------------------------------
# PULL-THROUGH CACHE -- identical in BOTH regions. This is what stops a
# failover depending on Docker Hub's rate limiter.
# ---------------------------------------------------------------------------
resource "aws_ecr_pull_through_cache_rule" "this" {
  for_each = var.pull_through_cache

  ecr_repository_prefix = each.key                              # ForceNew
  upstream_registry_url = each.value.upstream_registry_url      # ForceNew
  credential_arn        = try(each.value.credential_arn, null)  # Secrets Manager, same region
}

# ---------------------------------------------------------------------------
# SCANNING -- registry-scoped. Set it in BOTH regions or the standby's images
# are unscanned, which is a compliance finding waiting to be written up.
# Note: destroying this resource does not remove it, it reverts to BASIC.
# ---------------------------------------------------------------------------
resource "aws_ecr_registry_scanning_configuration" "this" {
  scan_type = "ENHANCED"
  rule {
    scan_frequency = var.role == "primary" ? "CONTINUOUS_SCAN" : "SCAN_ON_PUSH"
    repository_filter {
      filter      = "*"
      filter_type = "WILDCARD"
    }
  }
}
```

> [!note] Enhanced scanning in the standby is a real cost decision, not a rounding error
> `CONTINUOUS_SCAN` re-scans on new CVE intelligence and is billed per image
> per month. Applying it to a standby registry that holds a duplicate of every
> production image doubles that line. `SCAN_ON_PUSH` in the standby is the
> defensible middle: replicated images are scanned once on arrival, and the
> continuous intelligence comes from the primary's copy of the same digest.
> Verify current enhanced-scanning pricing on the
> [ECR pricing page](https://aws.amazon.com/ecr/pricing/) before choosing —
> **I have not verified the per-image figure** and it moves.

### `modules/ecr-repository` — the per-service module

The important design decision: **does this module create the standby's
repository, or does replication?** Both work and they trade differently.

| | Replication auto-creates | Terraform creates in both regions |
|---|---|---|
| Destination repository settings | Whatever the creation template says, or defaults | Exactly what you wrote |
| If the template was added late | Destination repos keep default settings forever | N/A — Terraform converges |
| Drift detection | None. Terraform does not know the repository exists | `terraform plan` shows it |
| Repository exists before the first push | **No** — a backfill with `crane` must create it | **Yes** — backfill just works |
| Extra Terraform surface | None | Doubles the repository resource count |
| Failure mode | Silent (a `FAILED` replication nobody reads) | Loud (`plan` fails) |

**Recommendation: Terraform creates both, and the creation template exists
anyway as a backstop.** The template alone relies on ordering that nothing
enforces and on a `FAILED` status nobody polls; declaring both repositories
makes the standby's registry visible to `terraform plan`, which is the only
drift detector this layer has. The cost is a doubled resource count in a module
that is three resources long. Take it.

```hcl
# modules/ecr-repository/main.tf
#
# Called once per service. Creates the repository in BOTH regions.
# Does NOT contain a replication configuration -- that is registry-scoped and
# belongs to ecr-registry-settings. If you put one here, the fiftieth
# instantiation silently overwrites the other forty-nine.

terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = "~> 6.0"
      configuration_aliases = [aws.primary, aws.standby]
    }
  }
}

variable "name"               { type = string }                  # e.g. "prod/payments"
variable "primary_kms_key_arn" { type = string, default = null }
variable "standby_kms_key_arn" { type = string, default = null }
variable "keep_tagged_primary" { type = number, default = 10 }
variable "keep_tagged_standby" { type = number, default = 30 }   # deliberately larger

locals {
  regions = {
    primary = { provider_key = "primary", kms = var.primary_kms_key_arn, keep = var.keep_tagged_primary }
    standby = { provider_key = "standby", kms = var.standby_kms_key_arn, keep = var.keep_tagged_standby }
  }
}

resource "aws_ecr_repository" "primary" {
  provider             = aws.primary
  name                 = var.name                    # ForceNew
  image_tag_mutability = "IMMUTABLE_WITH_EXCLUSION"  # NOT ForceNew -- safe to change on a live repo
  image_tag_mutability_exclusion_filter {
    filter      = "latest"
    filter_type = "WILDCARD"
  }
  image_scanning_configuration { scan_on_push = true }

  # ForceNew, and immutable in the API too. Getting this wrong means deleting
  # and recreating the repository -- i.e. deleting every image in it.
  encryption_configuration {
    encryption_type = var.primary_kms_key_arn == null ? "AES256" : "KMS"
    kms_key         = var.primary_kms_key_arn
  }
}

resource "aws_ecr_repository" "standby" {
  provider             = aws.standby
  name                 = var.name                    # SAME NAME. Replication cannot rename.
  image_tag_mutability = "IMMUTABLE_WITH_EXCLUSION"
  image_tag_mutability_exclusion_filter {
    filter      = "latest"
    filter_type = "WILDCARD"
  }
  image_scanning_configuration { scan_on_push = true }

  encryption_configuration {
    encryption_type = var.standby_kms_key_arn == null ? "AES256" : "KMS"
    kms_key         = var.standby_kms_key_arn        # a key in eu-west-2, NOT the primary's
  }
}

resource "aws_ecr_lifecycle_policy" "primary" {
  provider   = aws.primary
  repository = aws_ecr_repository.primary.name
  policy = jsonencode({ rules = [{
    rulePriority = 1
    description  = "Keep ${var.keep_tagged_primary} releases"
    selection    = { tagStatus = "tagged", tagPatternList = ["v*"], countType = "imageCountMoreThan", countNumber = var.keep_tagged_primary }
    action       = { type = "expire" }
  }] })
}

resource "aws_ecr_lifecycle_policy" "standby" {
  provider   = aws.standby
  repository = aws_ecr_repository.standby.name
  policy = jsonencode({ rules = [{
    rulePriority = 1
    description  = "Keep ${var.keep_tagged_standby} releases -- MORE than the primary, on purpose"
    selection    = { tagStatus = "tagged", tagPatternList = ["v*"], countType = "imageCountMoreThan", countNumber = var.keep_tagged_standby }
    action       = { type = "expire" }
  }] })
  # Note the absence of an untagged rule. That absence is the design.
}

output "primary_url" { value = aws_ecr_repository.primary.repository_url }
output "standby_url" { value = aws_ecr_repository.standby.repository_url }
```

The two `repository_url` outputs are what the deploy pipeline and the Kustomize
overlays consume — **never a hand-written hostname**. If the only place a
registry hostname is typed is a Terraform output, the grep in
[[#2. The manifest names the `eu-west-2` registry]] comes back clean by
construction.

### The root module for a pair

```hcl
# live/prod-eu/registry/main.tf

module "registry_primary" {
  source    = "../../../modules/ecr-registry-settings"
  providers = { aws = aws.primary }

  # Ordering is not optional. The standby's repository creation templates must
  # exist BEFORE the primary starts replicating, or the first wave of
  # destination repositories is born with default settings (MUTABLE, AES256,
  # no lifecycle policy, no repository policy) that the template will never
  # revisit -- templates apply at creation only.
  depends_on = [module.registry_standby]

  role                    = "primary"
  replicate_to            = [var.standby_region]
  replicate_prefixes      = ["prod/"]
  destination_registry_id = data.aws_caller_identity.current.account_id
  kms_key_arn             = aws_kms_key.ecr_primary.arn

  pull_through_cache = {
    "docker-hub" = { upstream_registry_url = "registry-1.docker.io", credential_arn = aws_secretsmanager_secret.dockerhub_primary.arn }
    "ecr-public" = { upstream_registry_url = "public.ecr.aws" }
    "k8s"        = { upstream_registry_url = "registry.k8s.io" }
    "quay"       = { upstream_registry_url = "quay.io" }
  }
}

module "registry_standby" {
  source    = "../../../modules/ecr-registry-settings"
  providers = { aws = aws.standby }

  role                       = "standby"
  replicate_to               = []                 # standby replicates nowhere
  creation_template_prefixes = ["prod/"]
  creation_template_role_arn = aws_iam_role.ecr_template_standby.arn
  destination_registry_id    = data.aws_caller_identity.current.account_id
  kms_key_arn                = aws_kms_key.ecr_standby.arn

  # Identical PTC rules, with a SEPARATE secret in eu-west-2 -- the credential
  # secret must live in the same account AND region as the rule.
  pull_through_cache = {
    "docker-hub" = { upstream_registry_url = "registry-1.docker.io", credential_arn = aws_secretsmanager_secret.dockerhub_standby.arn }
    "ecr-public" = { upstream_registry_url = "public.ecr.aws" }
    "k8s"        = { upstream_registry_url = "registry.k8s.io" }
    "quay"       = { upstream_registry_url = "quay.io" }
  }
}
```

> [!warning] `depends_on` between module calls is the only thing enforcing the ordering
> Nothing in the ECR API enforces "templates before replication". Nothing in
> `terraform plan` warns. If the two modules land in separate `apply`s — which
> they will if someone splits the root — the ordering is a human
> responsibility. Put it in the migration checklist as well as the code.

### The custom role for the creation template

Required as soon as the template sets KMS encryption or resource tags, and the
failure mode if it is missing or under-permissioned is *replication failing
silently*.

```hcl
resource "aws_iam_role" "ecr_template_standby" {
  provider = aws.standby
  name     = "ecr-repository-creation-template"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "replication.ecr.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "ecr_template_standby" {
  provider = aws.standby
  role     = aws_iam_role.ecr_template_standby.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ecr:CreateRepository",
          "ecr:PutLifecyclePolicy",
          "ecr:SetRepositoryPolicy",
          "ecr:TagResource",
          "ecr:PutImageTagMutability",
        ]
        Resource = "arn:aws:ecr:${var.standby_region}:${data.aws_caller_identity.current.account_id}:repository/*"
      },
      {
        Effect   = "Allow"
        Action   = ["kms:CreateGrant", "kms:DescribeKey", "kms:RetireGrant"]
        Resource = aws_kms_key.ecr_standby.arn
      },
    ]
  })
}
```

> **Verify the trust principal before you rely on this.** The service principal
> ECR uses to assume a repository-creation-template role is not something I
> could confirm from a primary AWS source with certainty, and it differs between
> the replication, create-on-push and pull-through-cache paths. Create the role,
> trigger one replication, and read CloudTrail's `AssumeRole` event to get the
> exact principal — then pin it. Do not ship this block unverified.

### ForceNew and replacement risk

The question every migration note in this vault has to answer: what in here can
destroy a live resource? Checked against the provider **source**, not only the
docs:

| Resource / argument | ForceNew? | What replacement costs you |
|---|---|---|
| `aws_ecr_repository.name` | **Yes** | Deleting the repository deletes **every image in it**. Renaming a repository is a data-loss operation dressed as a rename. |
| `aws_ecr_repository.encryption_configuration` (and `encryption_type`, `kms_key`) | **Yes** | Same — total image loss. And the API agrees: encryption configuration cannot be changed after creation. **Switching an existing repository from AES256 to a CMK is not possible in place.** |
| `aws_ecr_repository.image_tag_mutability` | **No** | Updated in place via `PutImageTagMutability`. Safe to flip `MUTABLE` → `IMMUTABLE_WITH_EXCLUSION` on a live repository. |
| `aws_ecr_repository.image_tag_mutability_exclusion_filter` | **No** | Same call, same safety. |
| `aws_ecr_repository.image_scanning_configuration` | **No** | `PutImageScanningConfiguration`, in place. |
| `aws_ecr_repository_creation_template.prefix` | **Yes** | Cheap — the template is metadata. Replacing it does not touch repositories. But note that repositories created under the old template keep their settings. |
| `aws_ecr_pull_through_cache_rule.ecr_repository_prefix`, `.upstream_registry_url`, `.upstream_repository_prefix` | **Yes** | Cheap-ish. Replacing the rule does not delete the cached repositories, but it does reset the cache relationship. |
| `aws_ecr_replication_configuration` | N/A (singleton) | Destroying it stops replication. It does **not** delete anything already replicated. |
| `aws_ecr_registry_scanning_configuration` | N/A (singleton) | Destroying it reverts the registry to `BASIC`, it does not remove the resource. |

**The two that matter, loudly:**

1. **`encryption_configuration` is ForceNew and API-immutable.** If the estate
   currently runs AES256 repositories and a control requires CMKs, the only
   path is: create a new repository, copy the images across with `crane`,
   repoint the deploy pipeline, delete the old one. There is no in-place
   migration. **Decide the encryption posture before you create the standby's
   repositories, because fixing it later is the same amount of work as the
   original migration.** See [[aws-kms]].
2. **`name` is ForceNew, and replication cannot rename.** If the current
   repositories are named per-region (`prod-eu-west-1/payments`), replication
   will faithfully create `prod-eu-west-1/payments` in London — a repository
   whose name is a lie. Fixing that is a rename, which is a
   delete-and-recreate, which is image loss. The safe sequence is *create the
   new name alongside, copy, cut the pipeline over, delete the old* — exactly
   the shape of [[dynamodb-table-naming-migration]], and worth reading that
   note for the pattern even though the service is different.

**Everything else is additive.** The standby's repositories, the templates, the
replication configuration, the PTC rules and the KMS keys are all new
resources in a new Region. `terraform plan` on the primary should show
**zero changes** to existing repositories apart from the in-place
`image_tag_mutability` flip. If it shows a replacement, stop.

## Migration path from single-region

Ordering is the whole difficulty. Several steps are irreversible-ish and two of
them must happen before replication is enabled or they never take effect.

1. **Answer the naming question first.** Are repositories named per-region or
   per-environment today? If per-region, the rename is the long pole and
   everything else waits on it, because replication preserves names and a
   renamed repository is a recreated repository. Do this as its own project.
2. **Answer the account question.** Is the standby in the same account? It
   changes whether you need a destination registry policy and cross-account
   repository policies at all. This is [[aws-eks]]'s open question 2 and it
   needs one answer for the whole estate.
3. **`ca-west-1` only: opt in, in every account**, and confirm with
   `aws account get-region-opt-status --region-name ca-west-1`. Multi-day lead
   time, invisible in `terraform plan`, and a hard prerequisite for replication
   into Calgary.
4. **Decide the encryption posture** (AES256 vs CMK) and create the standby KMS
   key if CMK. ForceNew, so this cannot be revisited cheaply.
5. **Create the repository creation template(s) in the standby**, with the
   custom role, and verify the role's trust principal from a CloudTrail
   `AssumeRole` event. **Before step 7.**
6. **Create the standby repositories in Terraform** with the same names as the
   primary. Additive, zero production impact. Doing this explicitly rather than
   leaving it to replication means the backfill in step 8 has somewhere to
   land and `terraform plan` can see drift.
7. **Enable replication in the primary**, with a prefix filter. From this
   moment every new push fans out. Watch `DescribeImageReplicationStatus` on
   the next deploy to confirm it works before trusting it.
8. **Backfill with `crane`/`regctl`** — the currently-deployed digest per
   service plus the last N releases. Run it from an instance in the **source**
   region, with concurrency 8–16 to stay under the 10/sec `PutImage` quota.
   See [[#Option C — Registry-to-registry copy with `crane` / `skopeo` / `regctl` (recommended)]].
9. **Verify the backfill by digest**, not by eyeball — the
   `kubectl`-to-`describe-images` check. Any output at all is a blocker.
10. **Flip `image_tag_mutability` to `IMMUTABLE_WITH_EXCLUSION`** in both
    regions. In-place, not ForceNew. Expect the first few pipeline failures
    from someone re-pushing a tag; that is the control working.
11. **Change the deploy pipeline to resolve and deploy by digest**, and to
    template the registry hostname per cluster. See
    [[#What the CI pipeline must change to]]. **This is the step that actually
    fixes the spine problem**; everything before it only moved bytes.
12. **Create PTC rules in both regions and warm the standby's cache** by
    pulling every third-party image through it once, from somewhere with
    internet egress. One-time, impossible during an incident.
13. **Arm the alarms**: EventBridge rule on `ECR Replication Action` with
    `result != SUCCESS` → SNS; Service Quotas alarms at 70% on the four pull-path
    quotas; the digest-existence cron on the `dr_readiness` dashboard.
14. **Raise the standby's ECR quotas** as part of the standby quota audit. Free,
    slow, and not filable during an incident.
15. **Deploy the pre-pull DaemonSet** in the standby so images land on nodes,
    not just in the registry — and so you have a continuous pull test.
16. **Drill.** Scale the standby, confirm every pod pulls, time it.

> [!warning] Steps 5 and 7 are ordered and nothing enforces it
> Template first, replication second. Reverse them and you get a set of
> destination repositories with `MUTABLE` tags, `AES256` encryption, no
> lifecycle policy and no repository policy — and a template that will never
> touch them, because templates only apply at creation. The only remedy is to
> delete those repositories and let replication recreate them, which means
> re-running the backfill.

## What the CI pipeline must change to

This is the section that turns the note into work. Four changes, in dependency
order.

### Change 1 — Resolve the digest at build time and carry it forward

```bash
# One push. Capture the digest the registry actually assigned.
docker buildx build --push \
  -t "$ACCOUNT.dkr.ecr.$PRIMARY.amazonaws.com/prod/payments:$VERSION" \
  --platform linux/amd64,linux/arm64 .

DIGEST=$(aws ecr describe-images --region "$PRIMARY" \
  --repository-name prod/payments --image-ids imageTag="$VERSION" \
  --query 'imageDetails[0].imageDigest' --output text)
```

Everything downstream references `$DIGEST`, never `$VERSION`. The tag survives
for humans reading `describe-images` at 3am.

### Change 2 — Gate the standby deploy on replication completion

The race in [[#The race]] is closed by polling
`DescribeImageReplicationStatus` before syncing the standby, and/or by driving
the standby's sync from the destination-region `ECR Replication Action` event.
Both, ideally: event as the fast path, poll as the authoritative backstop.

### Change 3 — Template the registry hostname per cluster

The manifest carries a digest and a *placeholder* registry. The per-region
overlay supplies the hostname, sourced from the Terraform output, never typed.

```yaml
# overlays/eu-west-2/kustomization.yaml
images:
  - name: payments
    newName: 123456789012.dkr.ecr.eu-west-2.amazonaws.com/prod/payments
    digest: sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
```

### Change 4 — Rewrite third-party image references to the PTC prefix

`docker.io/library/redis:7` becomes
`<acct>.dkr.ecr.<region>.amazonaws.com/docker-hub/library/redis:7`, per region.
This is what removes Docker Hub from the failover path.

### The fork: push to both registries, or push once and let replication fan out?

A genuine decision, and the brief asks for both branches.

| | **A — Push once, replication fans out** | **B — CI pushes to both registries** |
|---|---|---|
| Who copies the bytes | ECR, asynchronously | Your pipeline, synchronously |
| Timing guarantee | None. "Majority under 30 minutes." | Deterministic — the push either succeeded or the build failed |
| Digest preserved | Yes | Yes, **if** you `crane copy` rather than rebuild. A second `docker build` produces a different digest and is wrong. |
| Cost | ECR data transfer out from the source region | Same bytes, same direction, same charge — plus CI minutes |
| Pipeline complexity | Low. Plus a polling gate. | Higher. Two registry logins, two failure paths, retry logic. |
| Failure mode | Silent (`FAILED` status nobody reads) unless you alarm on it | Loud — the build goes red |
| Covers pre-existing images | No — the backfill is separate | No — the backfill is still separate |
| Covers images pushed by *anything other than CI* | **Yes** — a human `docker push`, a vendored image, a restore | **No** — anything bypassing the pipeline is not mirrored |
| Works when the standby region is degraded | Yes — replication retries | **No** — a `eu-west-2` blip fails your `eu-west-1` deploy. You have coupled the primary's release train to the standby's health. |
| Cross-account story | Destination registry policy, done once | Cross-account push credentials in CI, rotated |

**Recommendation: A — push once, let replication fan out, and gate the standby's
deploy on `DescribeImageReplicationStatus`.** Three reasons, in order:

1. **B couples the primary's ability to ship to the standby's health.** A
   region you are not serving from should never be able to block a release.
   That is a worse property than the replication lag it fixes.
2. **A catches everything, not just CI.** Registry replication is a property of
   the registry; it mirrors a human's emergency `docker push` at 2am just as
   faithfully as a pipeline's. B mirrors only what the pipeline does, and the
   images that end up missing at failover are precisely the ones that did not
   come from the pipeline.
3. **The thing B is supposed to fix — the race — is already fixable in A** for
   about ten lines of polling. You do not need to change where the bytes come
   from to fix *when* you deploy them.

**Take B only if** you have measured your own replication latency and found it
unacceptable (measure it first — see
[[#Detecting replication completion]]), or if a compliance control requires a
synchronous, auditable copy with a recorded outcome per release. If you do take
B, **copy with `crane`, never rebuild**, and keep replication enabled anyway as
the net for out-of-band pushes. A and B are not mutually exclusive; B plus A is
belt and braces, B instead of A is a gap.

## Failover procedure

ECR is the rare service with **no failover steps**, and that is the point.

It contributes **no steps** to the runbook in [[failover-orchestration]], which
is the correct outcome and worth stating explicitly there so nobody adds one.

1. **Nothing.** If the migration was done, the images are in `eu-west-2`, the
   standby's manifests already name `eu-west-2`'s registry, the node role's IAM
   policy already covers the `eu-west-2` repository ARNs, and the PTC cache is
   warm. The registry does not need promoting, flipping, or scaling.
2. **Confirm, do not assume.** The one action worth taking in the first five
   minutes, because it is read-only and fast:
   ```bash
   # Does the standby have every digest the primary was running?
   # Run against the last known-good pod inventory, captured by the cron job.
   aws ecr describe-images --region eu-west-2 \
     --repository-name prod/payments --image-ids imageDigest="$DIGEST"
   ```
   If the cron check has been green, skip even this.
3. **Stop replication into the dead region — later, not now.** Once
   `eu-west-2` becomes the region being deployed to, the replication
   configuration in `eu-west-1` is pointing the wrong way. It is harmless while
   `eu-west-1` is down (nothing is being pushed there) and it becomes a
   *failback* concern. Do not touch it during the incident.
4. **Watch for `ImagePullBackOff` specifically.** It is the symptom of every
   failure this note describes, and it is distinguishable from an application
   failure in one `kubectl get events`. If you see it, the diagnosis tree is:
   wrong registry hostname → missing digest → missing IAM/repository policy →
   missing VPC endpoint → Docker Hub 429. In that order, because that is the
   order of likelihood.

**Time budget: zero minutes.** ECR consumes none of the 15-minute RTO if the
work was done, and **unbounded** time if it was not — there is no fast remedy
for a missing image at 3am. That asymmetry is why this note is long.

## Failback

Failback is where the registry stops being free, because the direction of
replication is now wrong and the two registries have diverged.

**What happened during the outage:** you deployed to `eu-west-2` — hotfixes,
rollbacks, whatever the incident needed. Those images were pushed to
`eu-west-2`'s registry. `eu-west-1`'s registry **does not have them**, because
replication is one-directional and does not chain, and because `eu-west-1` was
down anyway.

Steps, in order:

1. **Do not fail back under pressure.** Same rule as [[aws-eks#Failback]]. Once
   the standby is serving, the incident is over. Failback is a weekday-morning
   planned change.
2. **Inventory the divergence.** Which digests does `eu-west-2` have that
   `eu-west-1` does not? The same `comm`-based diff from
   [[#Recommendation]], run in the opposite direction. This is a mechanical,
   scriptable answer and it should be the first thing you produce.
3. **Copy the gap back with `crane`**, from `eu-west-2` to `eu-west-1`. This is
   a second backfill and it has all the same properties: preserves digests,
   preserves multi-arch, respects the `PutImage` 10/sec quota, costs
   source-region data transfer out (now billed against `eu-west-2`).
4. **Reverse — or do not reverse — the replication configuration.** Two
   branches:
   - **Actually fail back.** Delete the `eu-west-2` → `eu-west-1` replication
     config once `eu-west-1` is primary again, and re-enable
     `eu-west-1` → `eu-west-2`. Note the not-retroactive trap applies *again*
     in the new direction: anything pushed to `eu-west-2` during the outage is
     pre-existing content from the new rule's point of view and will never
     replicate. That is what step 3 is for, and it is why step 3 comes first.
   - **Promote the standby permanently.** `eu-west-2` becomes the primary,
     `eu-west-1` becomes the standby, and you flip the `role` variable in the
     root module. Given the modules above are symmetric, this is a variable
     change, not a rewrite — which is a deliberate design property and the
     strongest argument for the `role`-driven module shape. **For ECR
     specifically, this is usually the better answer**, because it avoids a
     second backfill in the other direction and you have just proved the new
     primary works.
5. **Re-warm the old primary's PTC cache** if it expired. The 24-hour
   revalidation window means a long outage leaves stale cache state; the first
   pull after failback may need internet egress.
6. **Re-check lifecycle policies.** While `eu-west-1` was down its lifecycle
   rules kept running — ECR does not pause them for a Region you are not using.
   If `eu-west-1` had the aggressive "keep 10" policy and you were away for
   three weeks, it may have expired the digests you are about to fail back onto.
   **Run `start-lifecycle-policy-preview` against the old primary before you
   send traffic to it.**

> [!caution] The failback trap, in one sentence
> The old primary's lifecycle policy keeps deleting images while you are not
> looking, and replication will not put them back, because replication is not
> retroactive and does not run backwards.

## Gotchas

The consolidated list. Several are restatements — they are here because this is
the section people actually read. Items 1, 2, 4 and 6a are the ones worth
promoting into [[lessons-and-antipatterns]]: each is a case where the control
plane reports success, the console shows green, and the standby is broken.

1. **The image reference in the standby's manifest names the primary's
   registry.** Silent while the primary is healthy, total at failover. The
   single highest-value finding in this note. Grep for
   `\.dkr\.ecr\.[a-z0-9-]+\.amazonaws\.com` everywhere, including Terraform,
   Dockerfile `FROM` lines and Lambda image URIs.
2. **Replication is not retroactive.** Enabling it populates nothing. There is
   no ECR equivalent of S3 Batch Replication. Backfill with `crane`.
3. **Replication is async with no SLA.** "Majority under 30 minutes" is the
   entire published guarantee, and the ECR SLA (99.9% monthly uptime, per
   Region) covers API availability, not replication latency. A pipeline that
   pushes then immediately deploys to both regions races it and the standby
   loses.
4. **`:latest`, or any mutable tag, means the two registries can disagree about
   what you are running.** Failover then lands on an untested build. Deploy by
   digest.
5. **Immutable tags plus a re-pushed tag produce an *untagged* image in the
   destination**, not a rejected push. An untagged image is not pullable by tag,
   and it is exactly what a `tagStatus: untagged` lifecycle rule deletes.
6. **Lifecycle policies run independently per Region and are not replicated.**
   Either the standby has none and grows forever, or it has the primary's and
   can delete a digest the primary still runs. Make the standby's strictly more
   conservative, and never expire untagged images there.
6a. **`countType: sinceImagePulled` in a standby will eventually select every
    image in the registry**, because nothing ever pulls from a passive region and
    AWS falls back to `pushed_at_time` for images never pulled. The most
    reasonable-looking cleanup rule in the catalogue is the most destructive one
    here. Never use it in a standby.
7. **Repository creation templates only apply at repository creation.** Add one
   after replication has already created the destination repositories and it
   does nothing to them, forever. Templates before replication.
8. **A creation template that specifies KMS or resource tags without a valid
   `custom_role_arn` fails repository creation, which fails replication,
   silently.** The only evidence is a `FAILED` status nobody is polling.
9. **`encryption_configuration` is ForceNew and API-immutable.** AES256 → CMK on
   an existing repository is not a change, it is a migration.
10. **`name` is ForceNew and replication cannot rename.** Per-region repository
    names replicate as per-region repository names. Fix the naming first.
11. **Repository policies are not replicated.** In a cross-account estate the
    standby's nodes cannot pull a perfectly-present image. The creation template
    is the only sane fix, subject to gotcha 7.
12. **IAM policy ARNs are region-qualified.** `arn:aws:ecr:eu-west-1:...` in a
    node role grants nothing in `eu-west-2`. IAM is global; the ARNs inside it
    are not. Same shape as gotcha 1, different file.
13. **The S3 gateway endpoint is the third ECR endpoint people forget.** Layer
    blobs are served from S3. Symptom: auth succeeds, pull hangs.
14. **Docker Hub's unauthenticated limit is 100 pulls per 6 hours per IP, and
    every node behind one NAT gateway is one IP.** Thirty cold nodes at failover
    will blow it. PTC rules in both regions, authenticated, with references
    rewritten.
15. **A cold PTC cache needs internet egress.** A fully private standby VPC
    cannot populate one. Warm it while the primary is healthy.
16. **ECR-to-ECR pull-through cache pointed at the primary is an anti-pattern
    for DR.** Its upstream is the region you are failing away from.
17. **Deletes do not replicate.** A bad image deleted from the primary lives on
    in the standby. Over years the registries drift in both directions.
18. **Archive behaviour is asymmetric.** Per the replication docs, an image
    archived in the source is *not* archived in the destination, and an image
    replicated to a destination where it is archived gets *restored* there.
    Worth knowing if ECR's archival tiering is ever enabled — the standby's
    storage profile will not match the primary's.
19. **`PutImage` is 10/sec and it is the tightest quota in the table.** It does
    not matter at failover (pull path) and it very much matters during a
    parallel backfill. Concurrency 8–16.
20. **The replication configuration is a registry-scoped singleton.** Two
    Terraform declarations in one account+region do not merge; the last `apply`
    silently wins. Keep it out of the per-service module.
21. **The Terraform provider documents a maximum of 10 rules per replication
    configuration while AWS documents 25.** Not a problem at this estate's scale
    (one rule per pair), but do not design around 25 without testing it.
22. **Enhanced scanning with `CONTINUOUS_SCAN` in the standby doubles the
    scanning bill** on a registry holding a duplicate of every production image.
23. **Lambda cannot pull through a PTC rule.** Container-image Lambdas need real
    repositories in both regions. See [[aws-lambda]].
24. **Signatures replicate, but on their own timeline.** There is a window where
    the image is in the standby and its signature is not. An `enforce`-mode
    admission policy rejects the pod. Test this in the first drill.
25. **A `crane copy` of a single tag does not carry cosign's `.sig` tag.** Use
    `regctl image copy --digest-tags`, `cosign copy`, or `crane copy --all-tags`.
26. **`ca-west-1` is opt-in, and replication requires *both* accounts opted in.**
    Multi-day lead time across an organisation, invisible in `terraform plan`.

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| How images reach the standby | Registry replication, push once | CI pushes to both registries | **A**, with a `DescribeImageReplicationStatus` gate before the standby deploy. B couples the primary's release train to the standby's health, and misses anything not pushed by CI. Add B on top only if you measure replication latency and dislike it. |
| Backfilling existing images | Re-push from CI | `crane`/`regctl` registry-to-registry copy | **B.** Re-pushing rebuilds, a rebuild changes the digest, and you have replicated an image that is not the one in production. Restrict the copy to the live digest plus the last N releases. |
| What the Deployment references | Immutable tag (`v1.43.0`) | Digest (`@sha256:…`) | **B.** A digest cannot resolve differently in two registries. It converts every silent divergence into a loud `manifest unknown`. Keep the tag as well, for humans. |
| Tag mutability | `MUTABLE` | `IMMUTABLE_WITH_EXCLUSION` with `latest` excluded | **B** for everything you build; `MUTABLE` only for PTC repositories, where ECR needs to update cached tags. |
| Who creates the standby's repositories | Replication auto-creates them | Terraform declares them in both regions | **B**, *and* keep the creation template as a backstop. Terraform-declared repositories are the only drift detection this layer has, and the backfill needs somewhere to land. |
| Standby lifecycle policy | Mirror the primary's | Strictly more generous; no untagged rule for `prod/` | **B.** Storage is $0.10/GB-month. A missing image at failover is an outage. The asymmetry is the design, not an oversight. |
| Encryption | AES256 (AWS-owned) in both regions | CMK per region | Depends entirely on the control framework, and it is **ForceNew either way** — decide before creating repositories. If CMKs are required, **two independent regional keys, not a multi-Region key**: ECR never decrypts a foreign region's ciphertext. See [[aws-kms]]. |
| Third-party images | Pull from Docker Hub / `registry.k8s.io` directly | PTC rules in both regions, references rewritten | **B**, unconditionally. It is cheap, it removes a rate-limited third party from the failover path, and it is the one mitigation that works for Karpenter-launched surge nodes too. See [[third-party-saas-dependencies]]. |
| Standby deploy trigger | Timer / same pipeline step as primary | EventBridge `ECR Replication Action` in the destination region | **B as the fast path, polling as the backstop.** The event fires in the standby region, so the standby converges without any dependency on the primary — but AWS emits it best-effort, so it cannot be the only gate. |
| Replication scope | Whole registry | `prod/` prefix filter | **B.** Most registries are majority CI scratch images. This is the single largest cost lever and it costs nothing. |
| Enhanced scanning in the standby | `CONTINUOUS_SCAN` | `SCAN_ON_PUSH` | **B.** The continuous CVE intelligence applies to the same digest in the primary; paying twice for it buys nothing. Revisit if a control demands per-registry continuous scanning. |
| Admission policy in the standby when a signature has not replicated | `enforce` — fail closed | `audit`/`warn` — fail open, alarm loudly | **A for the primary, and a deliberate, documented, time-boxed break-glass for the standby.** Failing closed on a signature-replication lag converts a regional outage into a total outage; failing open silently is how unsigned images reach production. Neither is comfortable, which is why it must be a written decision rather than a default. |
| Failback | Reverse replication back to the old primary | Promote the standby permanently, flip `role` | **B, usually.** It avoids a second backfill and you have just proved the new primary works. The symmetric module shape makes it a variable change. |

## Cost

### What is actually published

| Line | Figure | Source confidence |
|---|---|---|
| ECR private storage | **$0.10 per GB-month** | **Verified**, [ECR pricing](https://aws.amazon.com/ecr/pricing/) |
| ECR → compute in the **same region** | **$0.00/GB** | **Verified** — same page |
| ECR private, data transferred **out** of a private repository | **$0.09 per GB** | **Verified** — stated under the private-registry Data Transfer heading on the [ECR pricing page](https://aws.amazon.com/ecr/pricing/). Note this is the same first-tier figure AWS uses for egress generally; it is *not* the cheaper inter-Region EC2 rate that third-party blogs quote. |
| Is replication itself charged as a separate line? | **No — it is billed as source-region data transfer out.** The ECR pricing page states: *"Data transferred when copying images across regions using Cross Region Replication incur ECR data transfer out charges based on the source repository's region."* | **Verified.** So the replication bill lands on the **primary's** account and region, not the standby's. Budget it there. |
| A cheaper inter-Region rate for the specific pairs (`eu-west-1`→`eu-west-2` etc.) | **Not found on an AWS page.** Third-party sources widely quote $0.02/GB for EU-to-EU inter-Region EC2 traffic, but the EC2 on-demand pricing page's data-transfer table did not render for retrieval, and the ECR page points at ECR data transfer out rates rather than the EC2 rate card. | **Not verified.** Budget at $0.09/GB, reconcile against a real bill, and correct this row afterwards. Do not put the $0.02 figure in a plan on the strength of a blog. |
| ECR **public**, egress | 500 GB/month free anonymous, 5 TB/month authenticated, **unlimited free to AWS compute in any region** | Verified — public-registry section of the same page. Listed separately because the two tables are easy to conflate. |
| ECR SLA | **99.9% monthly uptime per Region**, scoped to API availability | **Verified** — [ECR SLA](https://aws.amazon.com/ecr/sla/). It says nothing about replication latency, which is why there is no SLA to point at in [[#What is published]]. |
| ECR free tier | 500 MB/month private for 12 months (new accounts); 50 GB/month public storage for all | Verified |
| KMS | Per-key monthly charge plus request charges; ECR calls `GenerateDataKey`/`Decrypt` per layer operation | See [[aws-kms]] |

### Modelling the standby

The standby's ECR bill has three parts:

```
Storage (duplicate)   = replicated GB × $0.10/GB-month
Replication transfer  = GB pushed per month × inter-region $/GB   [rate not verified]
KMS                   = key/month + requests, if CMK-encrypted
```

Concretely, if the `prod/` prefix is **50 GB** and you push **20 GB/month** of new
layers:

- Standby storage: 50 × $0.10 = **$5.00/month**, rising as new images land.
- Replication transfer: 20 GB × $0.09 = **$1.80/month**, billed to
  `eu-west-1`'s account. If the real inter-Region rate turns out lower, this is
  an over-estimate — which is the safe direction for a budget.
- **Without a destination lifecycle policy this grows monotonically forever**,
  because deletes do not replicate. 20 GB/month of accumulation is $2/month of
  *additional* run-rate each month — $24/month after a year, $48 after two, and
  the standby overtakes the primary. This is the line item that surprises people,
  and it is entirely self-inflicted.

**The levers, largest first:**

1. **The replication rule's prefix filter.** Do not replicate the whole registry.
   Most registries are majority CI scratch images, PR builds and PTC caches. A
   `prod/` filter is often a 5–10× reduction and costs nothing to apply. This is
   the single biggest lever.
2. **A lifecycle policy in the destination** (via a repository creation template).
   Bounds the growth. Must be more conservative than the primary's — see
   [[#Lifecycle policies apply per region]].
3. **Image size.** Smaller images cost less to store, less to replicate, and —
   far more importantly — pull faster at failover. This is the rare optimisation
   that improves cost *and* RTO. Distroless/Chainguard bases and multi-stage
   builds. See [[cost-model]].
4. **Blob mounting / shared base layers.** Layers shared between images replicate
   once and are mounted, not re-transferred. Standardising on one base image
   across the estate meaningfully reduces both the replicated bytes and the
   destination storage.
5. **Do not replicate PTC-cached third-party images.** Set up an independent PTC
   rule in the standby instead — it populates on demand, and only with what is
   actually used.

**Against the ~$220/month per standby cluster in [[aws-eks]], ECR is noise
— tens of dollars.** The reason to care about the cost section is not the
money; it is that the *shape* of the cost (unbounded growth, no deletes) is a
symptom of the same design fact that causes the correctness problems.

## Open questions

Things this note cannot answer from outside the company.

1. **How are repositories named today — per-region, per-environment, or flat?**
   This is the blocking question. Replication preserves names and cannot rename,
   and `aws_ecr_repository.name` is `ForceNew`, so a per-region naming scheme
   turns the whole migration into a rename-and-copy project. Answer this before
   anything else. Same shape as [[dynamodb-table-naming-migration]].
2. **Is the standby cluster in the same AWS account as the registry?** Decides
   whether you need a destination registry permissions policy, cross-account
   repository policies, and blob mounting on both registries. [[aws-eks]] asks
   the same question — it needs one answer for the estate.
3. **Are repositories encrypted with AES256 or a CMK today?** ForceNew and
   API-immutable, so if the answer needs to change, it changes via a
   copy-and-cut-over, not an `apply`. See [[aws-kms]].
4. **How many hardcoded `*.dkr.ecr.<region>.amazonaws.com` strings exist across
   the deploy repo, the Terraform repo, Helm values, Dockerfiles and CI
   workflows?** One `grep` produces the number, and the number is the real size
   of the spine problem. Nobody's estimate has ever been high enough.
5. **Does the pipeline deploy by tag or by digest today?** Determines whether
   [[#What the CI pipeline must change to]] is a small change or a project.
6. **Is anything signed, and with which scheme?** cosign's legacy derived-tag
   layout and the OCI 1.1 referrers layout behave differently under both
   replication and lifecycle policy. The recommendation differs.
7. **What is the total size of the `prod/` prefix, and the monthly push
   volume?** Two numbers, both one CLI call, and they turn the cost model from a
   shape into a figure.
8. **What is the real replication latency for this estate?** AWS publishes only
   "majority under 30 minutes". The EventBridge correlation in
   [[#Detecting replication completion]] is a few dozen lines and produces a
   real p99. Nobody will know the true number until somebody measures it.
9. **Is `ca-west-1` opted in across every account in the organisation?** A
   multi-day, batched prerequisite for the CA pair that is invisible in
   `terraform plan`. Carry the answer into [[region-pair-selection]].
10. **Which service principal does ECR assume for a repository creation
    template's `custom_role_arn`?** Not confirmable from public documentation
    with enough certainty to ship. One CloudTrail `AssumeRole` event after the
    first replication settles it.
11. **Are there container-image Lambdas?** They cannot use a pull-through cache
    rule and need real repositories in both regions. See [[aws-lambda]].
12. **Does the control framework require continuous enhanced scanning in a DR
    standby?** If yes, the scanning line roughly doubles and it should be in
    [[cost-model]] rather than discovered on a bill.

### On regional outages

The brief asked whether any AWS regional outage postmortem names ECR as an
aggravating factor. **Checked, and the honest answer is: not in AWS's own
words.** The official
[summary of the October 2025 US-EAST-1 event](https://aws.amazon.com/message/101925/)
names container launch failures and cluster scaling delays across ECS, EKS and
Fargate, but does **not** name ECR or image pulls as a contributing mechanism.
Third-party write-ups of the same event — for example
[Jonathon Belotti's analysis](https://thundergolfer.com/blog/aws-us-east-1-outage-oct20) —
list ECR among the ~140 affected services, but as a *casualty* of the DynamoDB
and EC2 failures rather than as an amplifier.

The ECR-shaped risk that *is* real and *is* documented is structural rather than
incident-derived: **ECR Public's registry and control plane live in `us-east-1`**
(`api.ecr-public.us-east-1.amazonaws.com`), and EKS add-on images and a great
many open-source images are served from it. That makes ECR Public a shared
`us-east-1` dependency for clusters in every Region — including, uncomfortably,
the US pair's own standby, `us-west-2`, whose failover scenario is precisely
"`us-east-1` is gone". **A pull-through cache rule for `public.ecr.aws` in every
Region converts that shared dependency into a local one**, and for the US pair it
is not a nice-to-have. This is the strongest concrete argument in the note for
doing the PTC work, and it belongs in [[aws-regional-outages]] and
[[third-party-saas-dependencies]] as well as here.

No postmortem specifically attributing a failed multi-region failover to ECR
replication gaps was found. The failure mode is well described in guidance;
nobody appears to have published the incident. Same finding as [[aws-eks]]'s.

## Sources

### AWS documentation — ECR

- [Private image replication in Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/replication.html) — the load-bearing source for this note. Every "Considerations" bullet quoted above comes from here: not-retroactive, name preservation, no chaining, no deletes, the "majority under 30 minutes" latency statement, the 25-rule/25-destination/100-filter limits, the tag-immutability-produces-untagged behaviour, blob mounting, the opt-in-Region requirement, and the explicit statement that repository and lifecycle policies are not replicated.
- [Private registry permissions in Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry-permissions.html) — the destination-only registry policy for cross-account replication, and the `ecr:ReplicateImage` + `ecr:CreateRepository` pair.
- [Repository creation templates](https://docs.aws.amazon.com/AmazonECR/latest/userguide/repository-creation-templates.html) — the only mechanism that gives a replication-created repository a lifecycle policy, a repository policy, KMS encryption or tag-immutability settings. Also the source of the "only applied during repository creation" caveat and the `custom_role_arn` requirement when using KMS or resource tags.
- [Amazon ECR service quotas](https://docs.aws.amazon.com/AmazonECR/latest/userguide/service-quotas.html) — the throttling arithmetic: `GetAuthorizationToken` 500/s, `BatchGetImage` 2,000/s, `GetDownloadUrlForLayer` 3,000/s, `PutImage` 10/s. All per-Region, per-account, all adjustable.
- [Amazon ECR endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/ecr.html) — the `ca-west-1` parity check. Confirms `api.ecr`, `dkr.ecr` and dual-stack endpoints exist in Calgary, and that no `ecr-fips` endpoint does (nor in `ca-central-1`, so not a regression).
- [Using pull through cache rules](https://docs.aws.amazon.com/AmazonECR/latest/userguide/pull-through-cache.html) — supported upstreams, the Secrets Manager `ecr-pullthroughcache/` naming rule, the 24-hour image / 6-hour referrer revalidation windows, the "first pull may require a route to the internet" warning, the Lambda exclusion, and the "don't push into a PTC repository" statement.
- [Amazon ECR events and EventBridge](https://docs.aws.amazon.com/AmazonECR/latest/userguide/ecr-eventbridge.html) — the `ECR Replication Action` and `ECR Image Action` event schemas, and the "emitted on a best effort basis" caveat that stops you using the event as a correctness gate.
- [Encryption at rest for Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/encryption-at-rest.html) — the key must be in the same Region as the repository, encryption configuration is immutable after creation, the two KMS grants ECR creates, and the `aws:ecr:arn` encryption context.
- [`DescribeImageReplicationStatus` API reference](https://docs.aws.amazon.com/AmazonECR/latest/APIReference/API_DescribeImageReplicationStatus.html) — the authoritative per-image, per-destination replication status (`IN_PROGRESS` / `COMPLETE` / `FAILED`) that a CI gate should poll.
- [Amazon ECR lifecycle policies](https://docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html) — rule evaluation semantics and the `start-lifecycle-policy-preview` dry-run that is the cheapest guard against a standby policy deleting a live digest.

### AWS announcements and blogs

- [Amazon ECR now supports EventBridge notifications for replication (AWS, July 2024)](https://aws.amazon.com/about-aws/whats-new/2024/07/amazon-ecr-eventbridge-ecrs-replication-feature/) — when the completion signal became available, and the destination-region emission that makes event-driven standby deploys possible.
- [Amazon ECR announces pull through cache support for ECR private registries (AWS, March 2025)](https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-ecr-pull-through-cache/) — ECR-to-ECR PTC. Named here mainly to rule it out: its upstream is the region you are failing away from.
- [Diving into OCI Image and Distribution 1.1 support in Amazon ECR (AWS Open Source Blog)](https://aws.amazon.com/blogs/opensource/diving-into-oci-image-and-distribution-1-1-support-in-amazon-ecr/) — **resolves the signature question.** States that ECR's replication feature replicates referrers to configured destinations on push, so signatures and SBOMs land alongside replicated images; and that lifecycle policies protect reference artefacts whose subject image is still present, cleaning them up within 24 hours of the subject's deletion.
- [Amazon ECR supports OCI Image and Distribution specification v1.1 (AWS, June 2024)](https://aws.amazon.com/about-aws/whats-new/2024/06/amazon-ecr-oci-image-distribution-version-1-1) — confirms referrer support is available in all commercial Regions, which matters for the `ca-west-1` check.

### Pricing

- [Amazon ECR pricing](https://aws.amazon.com/ecr/pricing/) — $0.10/GB-month private storage; $0.00/GB to AWS compute in the same Region; **$0.09/GB for data transferred out of a private repository**; and the footnote that *"Data transferred when copying images across regions using Cross Region Replication incur ECR data transfer out charges based on the source repository's region"*. That footnote is the answer to "is replication billed separately" — it is billed as source-region data transfer out, not as a replication line item.
- [Amazon ECR Service Level Agreement](https://aws.amazon.com/ecr/sla/) — 99.9% Monthly Uptime Percentage per Region, defined against API request success. Cited for what it *doesn't* cover: replication latency is outside the SLA entirely.
- [AWS Regions and opt-in management](https://docs.aws.amazon.com/general/latest/gr/rande-manage.html) — `ca-west-1` is an opt-in Region; enabling is asynchronous ("a few minutes … sometimes several hours"); disabling does not delete resources or stop charges. Combined with the replication doc's opt-in requirement, this is the CA pair's real prerequisite.

### Terraform

- [`aws_ecr_repository` resource docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ecr_repository) and [the provider source for that resource](https://github.com/hashicorp/terraform-provider-aws/blob/main/internal/service/ecr/repository.go) — **checked against the source, not just the docs.** `name` and the whole `encryption_configuration` block (including `encryption_type` and `kms_key`) are `ForceNew: true`. `image_tag_mutability`, `image_tag_mutability_exclusion_filter` and `image_scanning_configuration` are **not** — the update path calls `PutImageTagMutability` and `PutImageScanningConfiguration` in place.
- [`aws_ecr_repository_creation_template` resource docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ecr_repository_creation_template) — `prefix` (with the special `ROOT` value) forces replacement; `applied_for` takes `CREATE_ON_PUSH`, `PULL_THROUGH_CACHE`, `REPLICATION`; `custom_role_arn` is required when the template sets resource tags or KMS encryption.
- [`aws_ecr_replication_configuration` resource docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ecr_replication_configuration) — the registry-scoped singleton. Confirms there is one per account per Region and that a second declaration replaces rather than merges.

### Outages

- [Summary of the Amazon DynamoDB Service Disruption in the Northern Virginia (US-EAST-1) Region, October 2025 (AWS)](https://aws.amazon.com/message/101925/) — AWS's own post-event summary. Names container launch failures and cluster scaling delays across ECS, EKS and Fargate; **does not name ECR or image pulls** as a mechanism. The basis for the negative finding in [[#On regional outages]].
- [More Than DNS: The 14 hour AWS us-east-1 outage — Jonathon Belotti](https://thundergolfer.com/blog/aws-us-east-1-outage-oct20) — third-party analysis listing ECR among the affected services. Useful for scope; explicitly **not** an AWS source and it treats ECR as a casualty, not a cause.

### Third party

- [Docker Hub usage and rate limits](https://docs.docker.com/docker-hub/usage/) — 100 pulls per 6 hours per IPv4 address / IPv6 /64 for unauthenticated users; 200 for authenticated personal accounts; unlimited on paid business tiers. The NAT-gateway-shared-IP arithmetic in [[#The Docker Hub rate-limit amplifier]] rests entirely on this page.
- [google/go-containerregistry — `crane`](https://github.com/google/go-containerregistry/blob/main/cmd/crane/doc/crane_copy.md), [regclient — `regctl image copy`](https://github.com/regclient/regclient/blob/main/docs/regctl.md) and [containers/skopeo — `skopeo sync`](https://github.com/containers/skopeo/blob/main/docs/skopeo-sync.1.md) — the three registry-to-registry copy tools that preserve digests and multi-arch manifest lists. `regctl --digest-tags` is the one that also carries cosign's digest-derived signature tags.

### Explicitly not found

- **No ECR replication SLA, and no published percentile distribution.** Searched specifically. The entire published guarantee is *"The majority of images replicate in less than 30 minutes, but in rare cases the replication might take longer."* There is no ECR equivalent of S3 Replication Time Control. If you want a p99, you have to measure it yourself — see [[#Detecting replication completion]].
- **No AWS-provided backfill for pre-existing images.** There is no ECR analogue of S3 Batch Replication. Confirmed by absence across the replication docs, the API reference and the ECR console.
- **No per-Region ECR feature-support matrix.** AWS publishes endpoints per Region and a quota table that says "Each supported Region" for every entry, but nothing that states which Regions support replication, PTC or referrers individually. The `ca-west-1` replication answer is therefore "no exclusion found, verify with a real `put-replication-configuration`" rather than a documented yes.
- **No published AWS inter-Region data transfer rate specific to ECR replication beyond the $0.09/GB private-repository figure.** Third-party blogs widely quote $0.02/GB for EU-to-EU inter-Region EC2 traffic; that figure is **not** on an AWS page I could retrieve, and the ECR pricing page's own footnote points at ECR data transfer out rates rather than the EC2 rate card. Budget with $0.09/GB and reconcile against a real bill.
