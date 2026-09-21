---
title: Amazon ECR — Multi-Region
service: ecr
tags: [service, multi-region, ecr, containers, registry, images]
status: partial
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

**AWS Signer / Notation (notary v2)** stores signatures using the OCI
**referrers** API rather than derived tags. Note the PTC documentation explicitly
mentions referrer artefacts and a 6-hour refresh window, and ECR emits an
`ECR Referrer Action` event type — so ECR does treat referrers as
first-class objects. **I could not find an authoritative AWS statement that
registry replication propagates referrer artefacts.** Treat this as unverified
and **test it** before depending on it: sign an image, let it replicate, and run
`aws ecr list-image-referrers` (or `notation verify`) against the destination.

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

## Cost

### What is actually published

| Line | Figure | Source confidence |
|---|---|---|
| ECR private storage | **$0.10 per GB-month** | **Verified**, [ECR pricing](https://aws.amazon.com/ecr/pricing/) |
| ECR → compute in the **same region** | **$0.00/GB** | **Verified** — same page |
| ECR private, cross-region data transfer | **Not published on the ECR pricing page.** The page says data transferred from a private repository is *"billed to the AWS account that owns the private repository"* at tiered rates that aggregate across AWS services — i.e. it falls under standard EC2 inter-region data transfer pricing. | **Not verified for the specific EU/US/CA pairs.** Look it up on the EC2 data transfer pricing page for the exact region pair before putting a number in a budget. |
| ECR **public**, cross-region | **$0.09/GB**, from a worked example on the ECR pricing page | Verified, but it is the *public* registry figure and should not be applied to private replication without checking |
| Is replication itself charged as a separate line? | **The ECR pricing page does not say.** | **Not verified.** Model it as: destination storage at $0.10/GB-month, plus inter-region data transfer on each replicated byte. |
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
- Replication transfer: 20 GB × (inter-region rate). **Look this up.** At a
  plausible order of magnitude it is single-digit dollars per month.
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

## Still to research

This note is `status: partial`. The following are written but not yet complete:

- **Terraform implementation section** — provider aliases, the registry-settings
  root module, the full module signature for the cookiecutter monorepo, and the
  `ForceNew` analysis. **This is the most important gap.**
- **Migration path from single-region** — ordered, with the ordering hazards
  (templates before replication) called out.
- **Failover and failback procedures** for the registry specifically.
- **Warm standby shape** section.
- **The consolidated Gotchas list.**
- **Decisions to make** table.
- **Open questions** and the full **Sources** list with per-source annotations.
- **Verify**: the exact inter-region data transfer rate for `eu-west-1` →
  `eu-west-2`, `us-east-1` → `us-west-2`, `ca-central-1` → `ca-west-1` against
  the EC2 data transfer pricing page.
- **Verify**: whether registry replication propagates OCI **referrer** artefacts
  (AWS Signer / Notation signatures). Currently unverified.
- **Verify**: `ForceNew` behaviour of `aws_ecr_repository.encryption_configuration`
  and `image_tag_mutability` in `hashicorp/aws` v5.x/v6.x against the provider
  source, not just the docs.
- **Check**: whether any AWS regional outage postmortem names ECR as an
  aggravating factor — see [[aws-regional-outages]].
