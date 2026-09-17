---
title: EKS Workload Delivery Across Regions — GitOps, Images, Secrets, Ingress
service: eks
tags: [service, multi-region, eks, kubernetes, gitops, argocd, ecr, delivery]
status: researched
replication: manual (Git is the source of truth; every mechanism below is "deploy a second copy")
rpo_achievable: N/A — no business data; the RPO here is "how stale is the standby's manifest set", target zero
rto_achievable: "< 2 min if manifests and images are already in the standby; 30+ min if the standby must sync from a hub that is in the failed region (FAILS RTO)"
meets_targets: conditional
updated: 2026-09-16
---

# EKS Workload Delivery Across Regions

> Companion to [[aws-eks]], which covers the cluster itself. This note covers
> **getting your workloads into the standby cluster and keeping them there**:
> GitOps, container images, secrets, and ingress. [[eks-stateful-workloads]]
> covers storage.

## TL;DR

- **The standby cluster's manifests must already be applied, at `replicas: 0`, before the incident.** "Sync at failover time" is a bet on the GitOps controller, the Git host, and the image registry all being healthy during a regional outage. Two of those three might be in the region that just died.
- **A hub-and-spoke ArgoCD in the primary region is a single point of failure, and it is the most common way this architecture is built.** If the hub lives in `eu-west-1` and `eu-west-1` is gone, you cannot sync, cannot scale via Git, and cannot roll back. Mitigations in [[#Is the GitOps controller a SPOF]] — the short answer is per-region controllers, or a hub outside both regions.
- **ECR cross-region replication is asynchronous and AWS's own documented expectation is "the majority of images replicate in less than 30 minutes"**. Against a 15-minute RTO that is a real hazard for a just-pushed image. It also **does not backfill existing images**, does not replicate deletes, and does not replicate repository or lifecycle policies. See [[aws-ecr]].
- **Secrets are already solved at the AWS layer** — Secrets Manager replication is done per [[research-brief]]. The remaining work is the Kubernetes side: an External Secrets Operator `ClusterSecretStore` in the standby pointing at the **standby region's** endpoint, using a **standby-cluster-scoped** IAM identity. Getting the region wrong here produces a standby that reads secrets cross-region from the dead region.
- **The thing that will bite:** two ExternalDNS instances writing one global Route 53 zone. Without distinct `--txt-owner-id` values they will delete each other's records in a loop — this is a real, filed, reproduced bug ([external-dns#1588](https://github.com/kubernetes-sigs/external-dns/issues/1588)), and it takes down the *primary* while the standby is perfectly healthy.

## Does this cross regions at all?

Nothing here replicates natively except ECR. Everything else is "run a second copy, pointed at the second region's endpoints". The interesting question is not *whether* you can run a second copy — you obviously can — but **what the second copy depends on, and whether those dependencies survive the outage you are failing over from.**

That dependency question is the whole note. Build the standby so that promoting it requires:
1. No Git pull.
2. No image pull from a remote region.
3. No control-plane call into the failed region.
4. No human editing YAML.

Anything that violates one of those four is a latent RTO failure.

## GitOps across two clusters

### The three topologies

| Topology | What it is | Survives losing the primary region? |
|---|---|---|
| **Hub and spoke** | One ArgoCD in a management cluster; both prod clusters registered as remote clusters. AWS's own reference pattern ([EKS Blueprints: multi-cluster hub-spoke ArgoCD](https://aws-ia.github.io/terraform-aws-eks-blueprints/patterns/gitops/gitops-multi-cluster-hub-spoke-argocd/)) uses an `awsAuthConfig.roleARN` per spoke. | **Only if the hub is not in the primary region.** If the hub is in `eu-west-1`, no. |
| **Per-region controllers** | An ArgoCD (or Flux) in each prod cluster, each syncing the same Git repo, each scoped to its own cluster. | **Yes.** Each cluster self-heals independently. |
| **Pull-model / agent** | Hub computes desired state, an agent in each spoke pulls it. [stolostron/argocd-pull-integration](https://github.com/stolostron/argocd-pull-integration) is the reference implementation. | Partially — the agent keeps running, but new desired state still originates at the hub. |

### Is the GitOps controller a SPOF?

**Yes, in the hub-and-spoke topology, and this is the default shape most teams land on** because it is the one AWS documents and the one that gives a single pane of glass.

Walk the failure through concretely. Hub ArgoCD in `eu-west-1`. `eu-west-1` has a regional event. You decide to fail over.

- You cannot change `replicas: 0` → `N` via Git, because the thing that reads Git is down.
- You cannot see sync status for the standby, because the UI is down.
- You cannot roll back a bad deploy in the standby, for the same reason.
- If anything in the standby needs a *new* manifest — a config change, a feature flag, a reduced resource request to fit smaller instances — you are editing YAML by hand with `kubectl` at 3am, which is precisely the situation GitOps exists to prevent.
- Worse: if the standby's workloads were already deployed and the hub is down, ArgoCD is not reconciling. That is survivable (Kubernetes keeps running what it has) but it means the standby is now unmanaged for the duration.

**The mitigations, ranked:**

1. **Per-region ArgoCD, one per prod cluster.** Each cluster runs its own controller, syncing the same repository, with an ApplicationSet scoped by region. Nothing about failing over one region touches the other's control loop. You lose the single pane of glass — mitigate with an ArgoCD UI federation or just two bookmarks. **This is the recommendation.** The operational cost is real but small; the SPOF removal is total.
2. **Hub in a third region.** Keeps single-pane-of-glass, moves the SPOF somewhere uncorrelated. Reasonable, but you have just introduced a *third* region to the design for the sake of a control plane, and a management cluster is now a production dependency with its own upgrade and DR story. Also: the hub still has to reach the standby's Kubernetes API, so the standby's API endpoint and the IAM path to it must work from that third region.
3. **Do not need GitOps at failover at all.** This is the strongest mitigation and it composes with the others: if the standby's manifests are **already applied** and the only failover action is `kubectl scale` / raising the node group, then ArgoCD being down is an inconvenience, not a blocker. Design for this regardless of topology.

> [!important] The scale-up mechanism must not require the hub
> Whatever else you choose, the *specific* action "take the standby from 0 replicas to N"
> must work with ArgoCD down. Two ways:
> - **KEDA / HPA with a `minReplicas` driven by something outside Git** — an SQS queue depth,
>   a CloudWatch metric, a ConfigMap the failover Lambda writes. The scale-up becomes a
>   consequence of traffic arriving rather than a deploy.
> - **A plain `kubectl scale` in the runbook**, accepted as temporary drift, with ArgoCD
>   configured not to fight it (`ignoreDifferences` on `/spec/replicas` for the standby's
>   Applications — which you want anyway if you use HPA).
>
> **Recommendation: `ignoreDifferences` on `/spec/replicas` plus a scripted `kubectl scale`,
> with a pre-authored PR merged afterwards to make Git match reality.** Fast at 3am,
> auditable by morning, and it does not depend on the hub.

### ApplicationSet with a cluster generator

The clean shape: one ApplicationSet, a cluster generator that discovers both registered clusters, and a per-cluster label carrying the region and the role.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: helios-workloads
  namespace: argocd
spec:
  goTemplate: true
  generators:
    - matrix:
        generators:
          # Every registered prod cluster, primary and standby alike.
          - clusters:
              selector:
                matchLabels:
                  helios.io/env: prod
          # Every service directory in the repo.
          - git:
              repoURL: https://github.com/helios/deploy.git
              revision: main
              directories:
                - path: services/*
  template:
    metadata:
      name: '{{.path.basename}}-{{.name}}'
    spec:
      project: prod
      source:
        repoURL: https://github.com/helios/deploy.git
        targetRevision: main
        path: '{{.path.path}}'
        helm:
          valueFiles:
            - values.yaml
            # Region-specific overlay: replica floors, region-scoped ARNs.
            - 'values-{{index .metadata.labels "helios.io/region"}}.yaml'
            # Role overlay: the ONLY place primary/standby differ.
            - 'values-{{index .metadata.labels "helios.io/role"}}.yaml'
      destination:
        server: '{{.server}}'
        namespace: '{{.path.basename}}'
      syncPolicy:
        automated: { prune: true, selfHeal: true }
      ignoreDifferences:
        - group: apps
          kind: Deployment
          jsonPointers: ['/spec/replicas']   # HPA and failover scale-up own this
```

The structural property that matters: **adding a directory under `services/` deploys it to both clusters automatically.** There is no separate list of "things the standby has". That is what makes the drift failure mode from [[aws-eks#Cluster version and config drift]] structurally hard rather than merely discouraged.

`values-standby.yaml` should contain **only** `replicaCount: 0` and, if genuinely necessary, smaller resource requests. The moment it starts carrying feature differences you have two products.

### Flux, briefly

Flux is naturally per-cluster — the controllers run in the cluster they manage, and multi-cluster is done by pointing each cluster's `GitRepository` at the same repo with a different `Kustomization` path. That means **Flux does not have the hub SPOF by default**, which is a genuine advantage for this specific design. If the estate has no incumbent, Flux's per-cluster model is the better fit for active/passive. If ArgoCD is already in place — which it usually is — deploy it per-region rather than migrating.

### The deploy pipeline: does every deploy go to both regions?

**Yes, it must.** The alternative — deploying only to the primary and "syncing the standby before failover" — is the drift failure mode with extra steps, and "before failover" is not a time that exists.

The hard question is the partial failure: **a deploy succeeds in `eu-west-1` and fails in `eu-west-2`.**

| Failure | Consequence | Response |
|---|---|---|
| Primary succeeds, standby fails | Standby is one version behind. Usually benign. Occasionally not — if the release included a DB migration, the standby's old code may be incompatible with the new schema. | **Alert loudly, do not block traffic.** The primary is serving; the standby is degraded. Fix forward. But treat "standby sync failed" as a **page-worthy** event, not a Slack message, because it is silently eroding your DR posture. |
| Standby succeeds, primary fails | Standby is one version *ahead*. | Same class of problem. Rolling *back* the standby is usually the right move so the pair is consistent. |
| Both fail | Normal deploy failure. | Nothing multi-region about it. |

**Key design rule: the standby must never be ahead of the primary.** If a failover lands on a version that has never served production traffic, you have converted a regional incident into a bad-release incident. Two ways to enforce it:

- **Sequence the sync waves.** ArgoCD sync waves or a pipeline that syncs primary → soak → standby. Adds latency to the standby's convergence, which is the cost.
- **Sync both simultaneously but gate on the primary's health.** If the primary's rollout fails, auto-rollback both.

**Recommendation: sync primary first, soak for the duration of your canary/bake period, then sync the standby.** The standby trails by minutes-to-an-hour, which is irrelevant for DR purposes (your RPO is 2 hours; a standby running the previous release is not a data-loss event) and it guarantees the standby never runs untested code. Make the trailing window visible on the `dr_readiness` dashboard from [[aws-eks]].

> [!warning] Database migrations are the real coupling
> In active/passive with async replication, the standby's database is a replica of the
> primary's, so a migration applied in the primary **arrives at the standby via replication**,
> not via deploy. That means the standby's *schema* moves with the primary while the standby's
> *code* moves with the deploy pipeline. If those desync, failover lands old code on a new
> schema. This is the strongest argument for keeping the trailing window short and for
> expand/contract migration discipline. See [[aws-rds-postgres]].

## Image availability in the standby

Covered in depth in [[aws-ecr]]; here is what matters for the cluster.

### ECR cross-region replication — the facts, from the docs

From the [ECR private image replication documentation](https://docs.aws.amazon.com/AmazonECR/latest/userguide/replication.html), verbatim or near-verbatim:

- **"The majority of images replicate in less than 30 minutes, but in rare cases the replication might take longer."** This is the one that interacts with your RTO. Push an image and fail over 10 minutes later and the standby may not have it.
- **"Only repository content pushed or restored to a repository after replication is configured is replicated. Any preexisting content in a repository isn't replicated."** No backfill. Every image currently in your registry stays put unless you re-push it. This is a **migration task**, not a config change.
- **"Registry replication doesn't perform any delete actions or archive actions."** Deletes do not propagate. Over time the standby registry accumulates everything ever pushed and costs more than the primary.
- **"Repository policies, including IAM policies, and lifecycle policies aren't replicated."** Your lifecycle rule that deletes untagged images after 14 days exists only in the primary. Use **repository creation templates** to apply settings to auto-created destination repositories, or the standby registry grows without bound.
- **Tag immutability trap:** "If tag immutability is enabled on a repository and an image is replicated that uses the same tag as an existing image, the image is replicated but won't contain the duplicated tag. This might result in the image being untagged." An untagged image is an image your Deployment cannot pull. If you use immutable tags *and* ever re-push a tag, the standby silently ends up with an untagged blob.
- **"A replication action only occurs once per image push"** — replication does not chain. A → B and B → C does not give you A → C.
- **Cross-Region replication is not supported between AWS partitions** — irrelevant for this estate (all commercial), but worth knowing.
- Limits: 25 rules, 25 unique destinations, 100 filters per rule. Comfortable for three pairs.
- Cross-account needs a **registry permissions policy on the destination** granting `ecr:ReplicateImage` and `ecr:CreateRepository`. The source account needs nothing. If `ecr:CreateRepository` is withheld, destination repositories must pre-exist or replication silently fails.

### ECR-to-ECR pull-through cache — the newer option

AWS announced [ECR-to-ECR pull through cache in March 2025](https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-ecr-pull-through-cache/): the standby registry is configured with a pull-through rule pointing at the primary registry, and on first pull ECR fetches and caches the image locally.

| | Replication | Pull-through cache |
|---|---|---|
| When images arrive | On push, async, "usually < 30 min" | On first pull |
| Storage cost | Everything you ever pushed | Only what you actually pulled |
| Works at failover from cold? | Yes, if replication completed | **No — the first pull is a cross-region fetch from the registry in the dead region.** |
| Backfill of existing images | No | N/A, it is pull-driven |

**For an active/passive DR standby, replication is correct and pull-through cache is wrong**, for exactly one reason: the pull-through cache's upstream is the primary registry, which is in the region you are failing away from. The cache helps a *warm* standby that has already pulled everything; it does nothing for a cold one, and it makes your failover depend on a dead region. Pull-through cache is a cost optimisation for multi-region *active/active*, not a DR mechanism.

**Recommendation: cross-region replication with a repository prefix filter** (replicate `prod/*`, not your entire registry including CI scratch images), plus a repository creation template carrying the lifecycle policy, plus a backfill pass at migration time.

### The strongest option: pre-pull onto the standby's nodes

Replication puts the image in the standby's *registry*. It does not put it on the standby's *nodes*. At failover, a node coming up from zero still pulls a multi-hundred-megabyte image over the network before the first container starts.

Three mechanisms, increasing in effort:

1. **A pre-pull DaemonSet on the system node group.** A DaemonSet whose containers are the real application images with a no-op command (`sleep infinity` or an init container that exits). Images land in the node's containerd store; the first real pod starts instantly. This is the [single-use DaemonSet pattern](https://codefresh.io/blog/single-use-daemonset-pattern-pre-pulling-images-kubernetes/). Cheap, works, and it doubles as a **continuous test that the image is actually present and pullable in the standby region** — which is exactly the sort of latent failure that Shape A in [[aws-eks]] otherwise hides. **Do this.**
   - Caveat: only helps nodes that exist. Nodes Karpenter creates during the failover surge pull fresh.
2. **Bake images into a custom AMI / Bottlerocket data volume.** AWS documents [reducing container startup time with a Bottlerocket data volume](https://aws.amazon.com/blogs/containers/reduce-container-startup-time-on-amazon-eks-with-bottlerocket-data-volume/): pre-pull images onto an EBS volume, snapshot it, and attach the snapshot to new nodes. **Every** node, including Karpenter-launched ones, starts with the images present. Costs you an AMI/snapshot pipeline that must track image releases — real operational weight, but it removes image pull from the critical path entirely.
3. **SOCI (Seekable OCI) lazy loading.** [awslabs/soci-snapshotter](https://github.com/awslabs/soci-snapshotter) starts containers before the image is fully downloaded, by indexing byte ranges. Natively supported by Bottlerocket. Helps most for large images. More moving parts; consider only if image size is measurably the bottleneck.

**Recommendation: (1) always — it is a few lines of YAML and it is also a monitor. Add (2) if measured pull time is a material part of your failover budget.** Measure during the first drill before deciding.

## Secrets reaching pods in the standby

Secrets Manager replication is already done per [[research-brief]], so the AWS-side data is present in the standby region. The Kubernetes side is what remains.

### External Secrets Operator (recommended)

ESO watches `ExternalSecret` custom resources and materialises real Kubernetes `Secret` objects. Two things must be right in the standby:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets
spec:
  provider:
    aws:
      service: SecretsManager
      # CRITICAL: the STANDBY's own region. If this says eu-west-1 in the
      # eu-west-2 cluster, every secret read during a failover goes to the
      # region that just died. This is the single most common mistake here.
      region: eu-west-2
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

Two properties of this that matter:

- **The `region` field must be templated per-cluster**, from the same region label the ApplicationSet uses. Hardcoding it is how you get a standby that reads from the dead region. Because the *secret names* are identical in both regions (that is what Secrets Manager replication gives you), the `ExternalSecret` resources themselves are region-agnostic — only the `ClusterSecretStore` differs. Put the store in the platform layer, not the app layer.
- **The service account's IAM identity must work in the standby cluster.** This is [[aws-eks#IRSA across regions]] again. With IRSA, the ESO role needs a trust statement for the standby cluster's OIDC issuer. With Pod Identity, one association per cluster and nothing in IAM changes. ESO failing to authenticate is a *quiet* failure — the `ExternalSecret` sits in `SecretSyncedError` and nobody is looking at the standby's CR statuses.

**Known limitation, from upstream:** an `ExternalSecret` references exactly one `SecretStore`/`ClusterSecretStore`, and **there is no native fallback to a second store if the primary provider is unavailable** ([external-secrets#5882](https://github.com/external-secrets/external-secrets/issues/5882)). For this design that is fine — each cluster points at its own region and never needs to fail over the store itself — but it rules out the "one store with a regional fallback" pattern people reach for first.

**The property that saves you:** ESO writes a real Kubernetes `Secret`. Once written, it persists in etcd. If Secrets Manager in the standby region is slow or unreachable at failover time, the previously-synced Secret is still there and pods still start. This is genuine resilience and it is the main reason to prefer ESO over the CSI driver for a DR standby.

### Secrets Store CSI driver + AWS provider (ASCP)

Mounts secrets as a tmpfs volume at pod start; can optionally sync to a Kubernetes Secret.

- **Pro:** no long-lived Kubernetes `Secret` object, which some security postures require.
- **Con for DR:** the fetch happens **synchronously at pod start**. If the standby region's Secrets Manager endpoint is degraded during the same event, pods fail to start rather than starting with slightly stale values. In a DR scenario that is exactly the wrong failure mode.
- **Con:** ASCP is a DaemonSet, so it does not work on Fargate — relevant if you chose the Fargate-hosted-Karpenter bootstrap from [[aws-eks]].

**Recommendation: External Secrets Operator.** The materialised-Secret behaviour that a security review dislikes is precisely the behaviour that makes failover robust. If policy forbids Kubernetes Secrets, use the CSI driver *with* `secretObjects` sync enabled so you get both, and accept the same trade-off.

### SSM Parameter Store

Same story — ESO supports it as a provider. But note the asymmetry flagged in [[aws-ssm-parameter-store]]: Parameter Store has **no native cross-region replication**, unlike Secrets Manager. If any config lives in Parameter Store, replicating it is your problem, not AWS's, and it is a prerequisite for the standby's pods to start. Audit for this early — it is the sort of thing that is discovered when a pod crashloops on a missing `/helios/prod/feature-flags` parameter.

## Ingress: ALBs, target groups, and Route 53

### AWS Load Balancer Controller in the standby

The controller is regional by nature: it runs in a cluster, watches `Ingress`/`Service` objects, and calls the ELB API **in its own region**. Two clusters, two controllers, two sets of ALBs. Nothing crosses.

The decision that matters: **does the standby's ALB exist while the primary is healthy?**

| Option | Idle cost | Failover time | Notes |
|---|---|---|---|
| **ALB pre-created, zero healthy targets** | ~$16–20/month per ALB + LCU | Targets register as pods become ready: **~30–90 s** | The ALB DNS name is stable and known, so Route 53 records can point at it in advance. **Recommended.** |
| **ALB created at failover** by the controller reacting to an Ingress | $0 | ALB provisioning is minutes, then DNS propagation for a brand-new name, then target registration | Also means Route 53 cannot be pre-configured, because the target doesn't exist yet. Avoid. |

To pre-create it, the `Ingress` must exist in the standby even with zero backing pods. It will, if you deploy manifests at `replicas: 0` as recommended — the Ingress object is independent of the Deployment's replica count, and the controller creates the ALB and an empty target group. The ALB will report unhealthy, which your monitoring must be taught to expect for standby ALBs, or you will alarm-fatigue yourself into ignoring it.

> [!tip] Pre-warm awareness
> An idle ALB scales its capacity down. A failover sends full production traffic at a
> load balancer that has been serving nothing for months. ALBs do scale automatically but
> not instantaneously. If your traffic profile is spiky enough for this to matter, this is
> an argument for the warm Shape B from [[aws-eks]], or for sending a trickle of synthetic
> traffic at the standby ALB continuously. See [[aws-alb-nlb]].

**Target type:** use `ip` mode rather than `instance` mode. With `ip` targets the ALB registers pod IPs directly, so target registration tracks pod readiness with no NodePort hop and no dependency on node group membership. It also avoids a whole class of "node is in the target group but the pod moved" problems during a scale-up surge.

### Reaching Route 53

Route 53 is **global**, which means both clusters' ExternalDNS instances write into the same hosted zone. This is the single most dangerous piece of configuration in this note.

**The failure:** ExternalDNS with `policy=sync` treats records it believes it owns as its to delete. Two instances sharing a `--txt-owner-id` (or both using the default) each conclude they own everything, and enter a delete/create loop. This is filed, reproduced, and well documented — [external-dns#1588: "I have two external dns pods pointing to same route53 hosted zone and each of it is deleting the other entry"](https://github.com/kubernetes-sigs/external-dns/issues/1588) and [helm/charts#22196](https://github.com/helm/charts/issues/22196). **The blast radius is the primary**, not the standby: your healthy production region loses its DNS records because a cluster that serves no traffic decided to tidy up.

**Configuration that prevents it:**

```yaml
# Primary cluster
--txt-owner-id=helios-prod-eu-west-1
--policy=sync

# Standby cluster
--txt-owner-id=helios-prod-eu-west-2
--policy=upsert-only        # belt and braces: cannot delete anything, ever
```

Distinct owner IDs are the actual fix. `upsert-only` on the standby is a second, independent guard: even if the owner ID is misconfigured, an `upsert-only` instance is incapable of deleting a record. For a standby that should not be authoritative for anything, there is no downside — the cost is that stale records in the standby's own namespace are never cleaned up, which for a standby is not a problem worth having.

**Better still: do not let ExternalDNS own the failover records at all.** The record that actually moves traffic — the Route 53 failover/weighted record or health-checked alias pointing at one ALB or the other — should be managed by **Terraform**, not by a controller reacting to Kubernetes objects. Reasons:

- It is the most important record in the system and it should change only by a deliberate, reviewed act.
- At failover you want to flip it via an AWS API call from outside both clusters, which Terraform-managed records support and ExternalDNS-managed records fight.
- ExternalDNS can still own per-service records in a `*.internal` or region-qualified zone, where its dynamism is useful and its blast radius is small.

Full treatment in [[aws-route53]]; the split is: **Terraform owns the failover apex, ExternalDNS owns region-scoped service records only.**

## Warm standby shape (delivery layer)

While the primary is healthy, the standby has:

- ArgoCD/Flux running and syncing (its own controller, per the recommendation above).
- Every workload's `Deployment`, `Service`, `Ingress`, `ServiceAccount`, `ConfigMap`, `NetworkPolicy`, `PDB` and `PriorityClass` applied — with `replicas: 0`.
- Every `ExternalSecret` synced, meaning real Kubernetes `Secret` objects present in etcd.
- Every ALB created, with empty target groups.
- Images replicated into the standby's ECR, and pre-pulled onto the system nodes by the DaemonSet.
- A **DR canary** — one real pod of a real service at `replicas: 1`, exercising the image, IAM, secrets and database path continuously ([[aws-eks#Warm standby shape]]).

Failover then reduces to: raise the node group, scale replicas, flip DNS. Everything else already happened, days ago, under no time pressure.

## Terraform implementation

Most of this layer is Kubernetes objects, not AWS resources — so the Terraform surface is the *bootstrap* of the platform layer plus the AWS-side registry and DNS config.

### ECR replication (registry-level, configured in the source region)

```hcl
# Registry replication is a per-registry (per account, per region) setting.
# Configure it in each PRIMARY region, pointing at that pair's standby.
resource "aws_ecr_replication_configuration" "eu" {
  provider = aws.primary   # eu-west-1

  replication_configuration {
    rule {
      destination {
        region      = var.standby_region      # eu-west-2
        registry_id = data.aws_caller_identity.current.account_id
      }
      # Do NOT replicate the whole registry. CI scratch images are the bulk
      # of most registries and none of them belong in a DR standby.
      repository_filter {
        filter      = "prod/"
        filter_type = "PREFIX_MATCH"
      }
    }
  }
}

# Lifecycle policies are NOT replicated. Without this, the standby registry
# grows forever and costs more than the primary.
resource "aws_ecr_repository_creation_template" "standby" {
  provider = aws.standby

  prefix               = "prod/"
  applied_for          = ["REPLICATION"]
  image_tag_mutability = "IMMUTABLE"

  lifecycle_policy = jsonencode({
    rules = [{
      rulePriority = 1
      description  = "Expire untagged images after 14 days"
      selection    = { tagStatus = "untagged", countType = "sinceImagePushed", countUnit = "days", countNumber = 14 }
      action       = { type = "expire" }
    }]
  })
}
```

### Platform layer via Helm, per cluster

The platform components differ only by region and cluster name. Same module, two providers, mirroring the [[aws-eks]] layout.

```hcl
# modules/eks-platform/main.tf
variable "cluster_name" { type = string }
variable "region"       { type = string }
variable "role"         { type = string }   # primary | standby
variable "vpc_id"       { type = string }
variable "hosted_zone_id" { type = string }

resource "helm_release" "aws_load_balancer_controller" {
  name       = "aws-load-balancer-controller"
  repository = "https://aws.github.io/eks-charts"
  chart      = "aws-load-balancer-controller"
  version    = var.chart_versions["aws_load_balancer_controller"]   # pinned, shared
  namespace  = "kube-system"

  set = [
    { name = "clusterName", value = var.cluster_name },
    { name = "region",      value = var.region },
    { name = "vpcId",       value = var.vpc_id },
    # System node group only — see the CriticalAddonsOnly taint in [[aws-eks]]
    { name = "nodeSelector.helios\\.io/pool", value = "system" },
  ]
}

resource "helm_release" "external_dns" {
  name       = "external-dns"
  repository = "https://kubernetes-sigs.github.io/external-dns"
  chart      = "external-dns"
  version    = var.chart_versions["external_dns"]
  namespace  = "external-dns"

  set = [
    # Unique per cluster. This single value is what stops the two clusters
    # deleting each other's Route 53 records. See external-dns#1588.
    { name = "txtOwnerId", value = "${var.cluster_name}-${var.region}" },
    # The standby can create and update records but can never delete one.
    { name = "policy",     value = var.role == "standby" ? "upsert-only" : "sync" },
    { name = "domainFilters[0]", value = var.external_dns_domain },
    { name = "aws.region",       value = var.region },
  ]
}

resource "helm_release" "external_secrets" {
  name       = "external-secrets"
  repository = "https://charts.external-secrets.io"
  chart      = "external-secrets"
  version    = var.chart_versions["external_secrets"]
  namespace  = "external-secrets"
  create_namespace = true
}

# The ClusterSecretStore's region MUST be this cluster's own region.
resource "kubernetes_manifest" "cluster_secret_store" {
  manifest = {
    apiVersion = "external-secrets.io/v1beta1"
    kind       = "ClusterSecretStore"
    metadata   = { name = "aws-secrets" }
    spec = {
      provider = {
        aws = {
          service = "SecretsManager"
          region  = var.region          # <- not a hardcoded string, anywhere
          auth    = { jwt = { serviceAccountRef = {
            name = "external-secrets", namespace = "external-secrets"
          }}}
        }
      }
    }
  }
  depends_on = [helm_release.external_secrets]
}

# Pre-pull DaemonSet: keeps application images resident on the system nodes
# and doubles as a continuous "is the image actually here" monitor.
resource "kubernetes_manifest" "image_prepull" {
  count = var.role == "standby" ? 1 : 0
  manifest = yamldecode(templatefile("${path.module}/prepull-daemonset.yaml.tftpl", {
    images = var.prepull_images        # the same image list the pipeline publishes
    region = var.region
  }))
}
```

> `chart_versions` is a single shared map, exactly like `addon_versions` in [[aws-eks]].
> One variable, both clusters, no per-region overrides. This is the mechanism that keeps
> the platform layers identical.

### Provider plumbing note

The `helm` and `kubernetes` providers cannot be aliased as cleanly as `aws` when the cluster they target is created in the same apply — you get the classic "provider configuration depends on a resource that doesn't exist yet" problem. In a templated monorepo the standard resolution is to **split cluster creation and platform installation into separate root modules / separate state**, with the platform root reading the cluster via `data "aws_eks_cluster"`. That is one more root per region, which the cookiecutter structure should absorb comfortably. See [[terraform-repo-structure]] — this is a structural decision, not a detail.

An alternative many mature shops prefer: **Terraform creates the cluster and installs exactly one Helm release — ArgoCD — and everything else is GitOps.** That keeps the provider problem to a single bootstrap, and it means the platform layer is managed the same way as workloads. **Recommended** if ArgoCD is already the deployment mechanism.

## Migration path from single-region

1. **Audit what the workloads actually need.** Every image, every secret ARN, every parameter path, every IAM role, every ConfigMap that contains a region or an ARN. The last category is the one that surprises people — grep the whole repo for `eu-west-1` and for `arn:aws:` and expect to find more than you thought.
2. **Turn on ECR replication and backfill.** Replication is not retroactive. Re-push or copy (`crane copy`, `skopeo sync`) every image tag the standby might need. Do this before anything tries to deploy there.
3. **Add a repository creation template** so replicated repositories inherit the lifecycle policy.
4. **Register the standby cluster with the GitOps controller** (or install a per-region controller — decide this first, it changes step 5).
5. **Introduce the `role` and `region` cluster labels and the `values-<role>.yaml` overlay.** At this point the primary is unchanged; the overlay for `primary` is empty.
6. **Deploy the platform layer to the standby.** ESO, ALB controller, ExternalDNS (**with the distinct owner ID and `upsert-only` set from the very first deploy** — get this wrong once and you take out the primary's DNS).
7. **Deploy workloads at `replicas: 0`.** Watch every `ExternalSecret` reach `SecretSynced` and every `Ingress` produce an ALB. These two checks catch the majority of cross-region IAM mistakes.
8. **Add the pre-pull DaemonSet** and the DR canary pod.
9. **Drill.** Scale up, verify, scale down, record the elapsed time.

Nothing here forces replacement of a primary-region resource. The one genuinely dangerous step is 6 — ExternalDNS misconfiguration is the only action in this list that can cause a production outage.

## Failover procedure (delivery layer)

Assumes [[aws-eks]]'s Shape A and everything above pre-staged.

1. Node group raised via the EKS API (step 2 of [[aws-eks#Failover procedure]]) — in flight already.
2. **Scale workloads.** `kubectl scale --replicas=N` against the standby, or merge the pre-authored PR. Does not require the GitOps hub.
3. **Verify secrets.** `kubectl get externalsecrets -A` — anything not `SecretSynced` is a pod that will not start. Check before you flip DNS, not after.
4. **Watch ALB target health.** Targets register as readiness probes pass.
5. **Flip Route 53.** The Terraform-managed failover record, via AWS API. See [[aws-route53]].
6. **Confirm ExternalDNS in the standby is not now deleting things.** If it is `upsert-only` it cannot. Verify anyway, once, on the first real failover.

## Failback

- **Images:** replication is one-way, primary → standby. Any image built and pushed *during* the failover went to the region that is now primary-in-practice but configured as the standby. The replication rule points the wrong way. Either reverse it temporarily, or — cleaner — pin the failover-period deploys and re-push from CI after failback. Decide in advance; this is the sort of thing that is not obvious at 4am.
- **GitOps:** if you scaled with `kubectl` rather than Git, Git now disagrees with both clusters. Reconcile deliberately: one PR that sets the standby back to 0 and the primary back to N, reviewed, merged, synced.
- **ExternalDNS:** if you flipped the standby to `policy=sync` during the failover (don't), flip it back before the primary comes up, or it will delete the recovering primary's records.
- **ALBs:** the old primary's ALBs have been idle for the duration and are now cold. Same pre-warm consideration, in reverse.

## Gotchas

1. **ExternalDNS mutual deletion.** Distinct `--txt-owner-id` per cluster, `upsert-only` in the standby. Blast radius is the *healthy* region. [external-dns#1588](https://github.com/kubernetes-sigs/external-dns/issues/1588).
2. **ECR replication does not backfill.** Every image that exists today stays put. Requires an explicit migration pass.
3. **ECR replication lag of up to ~30 minutes**, per AWS's own documentation. A deploy-then-fail-over sequence inside that window finds a missing image.
4. **ECR lifecycle and repository policies are not replicated.** Use repository creation templates or pay for unbounded growth.
5. **Tag immutability + a re-pushed tag = untagged image in the destination.** Silently unpullable.
6. **`ClusterSecretStore` with a hardcoded region.** The standby reads secrets from the dead region. Template it from the cluster's own region label.
7. **ESO auth failures are silent.** `SecretSyncedError` on a CR nobody watches. Alert on `externalsecret_status_condition` in the standby specifically.
8. **The GitOps hub in the primary region.** The whole [[#Is the GitOps controller a SPOF]] section.
9. **ArgoCD `selfHeal` fighting your failover scale-up.** Without `ignoreDifferences` on `/spec/replicas`, ArgoCD reverts your `kubectl scale` back to 0 within seconds of you running it. This *will* happen on the first drill if not configured.
10. **Standby ALBs alarm as unhealthy forever.** Expected state; configure monitoring to know the difference, or you will train the team to ignore ALB alarms.
11. **`instance`-mode target groups** couple target registration to node group membership and add a NodePort hop. Use `ip` mode.
12. **Parameter Store has no native replication.** Unlike Secrets Manager. If workloads read config from SSM, that replication is your job. See [[aws-ssm-parameter-store]].
13. **Hardcoded region strings and ARNs in ConfigMaps and Helm values.** The long tail. Grep for them; there are always more than expected.
14. **The standby running a newer release than the primary.** Sequence the sync waves so the standby trails.

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| GitOps topology | Hub-and-spoke, one ArgoCD | One controller per prod cluster | **B.** Removes the SPOF entirely. The lost single-pane-of-glass is worth less than the removed dependency. |
| Standby workload state | Manifests applied at `replicas: 0` | Nothing applied until failover | **A**, without qualification. B makes failover depend on Git, the registry and the controller all being healthy during an outage. |
| Failover scale-up mechanism | Git commit + sync | Scripted `kubectl scale` with `ignoreDifferences` | **B for the action, A for the record.** Script it at 3am; merge the PR in the morning. |
| Deploy ordering | Both regions simultaneously | Primary, soak, then standby | **B.** Guarantees the standby never runs untested code. The lag is irrelevant to a 2-hour RPO. |
| Image distribution | ECR cross-region replication | ECR-to-ECR pull-through cache | **A.** Pull-through's upstream is the region you are failing away from. |
| Image warm-up | Pre-pull DaemonSet | Baked AMI / Bottlerocket data volume | **Start with the DaemonSet** (also a monitor). Add the baked volume only if measured pull time is a material part of the failover budget. |
| Secrets delivery | External Secrets Operator | Secrets Store CSI driver | **A.** The materialised Secret persists in etcd and survives a degraded Secrets Manager endpoint during the same event. |
| Failover DNS record ownership | ExternalDNS | Terraform | **Terraform for the failover apex**, ExternalDNS for region-scoped service records only. |
| Platform layer management | Terraform Helm releases | Terraform bootstraps ArgoCD, ArgoCD does the rest | **B** if ArgoCD is already in use. Avoids the Kubernetes-provider bootstrap problem and keeps platform and workloads managed identically. |

## Cost

Delivery-layer additions per standby region, per month, rough:

| Line | Cost |
|---|---|
| ECR storage for replicated images | $0.10/GB-month. A prefix-filtered prod repo set is usually tens of GB → **$5–20**. Unbounded without a lifecycle policy. |
| ECR cross-region data transfer (replication) | Inter-region transfer rates, charged on each push. Proportional to release frequency × image size. See [[aws-ecr]]. |
| Idle ALB(s) | ~$16–20 each + minimal LCU. |
| ArgoCD/ESO/ALB controller/ExternalDNS pods | Fit on the existing system node group — **$0 incremental**. |
| Pre-pull DaemonSet | EBS for the images on system nodes, a few GB. Negligible. |
| DR canary pod | One pod. Negligible. |

**~$25–45/month per standby region** on top of [[aws-eks]]'s ~$220. The levers are the ECR prefix filter (do not replicate CI images), the lifecycle policy on the destination (or storage grows forever), and image size itself — which also buys you failover speed.

## Open questions

1. **ArgoCD or Flux today, and where does the controller run?** Determines whether the SPOF recommendation is a migration or a config change.
2. **Is there a management/tooling cluster, and which region is it in?** If it is in a primary region, it is a dependency nobody has counted.
3. **Does CI push images to one registry or per-region registries today?** Changes whether replication is additive or a pipeline change.
4. **How much config lives in SSM Parameter Store vs Secrets Manager?** Parameter Store has no native replication and is a hidden prerequisite.
5. **Are there ConfigMaps or Helm values containing hardcoded region strings or ARNs?** Needs an actual grep of the deploy repo.
6. **What is the current image size and cold pull time?** Determines whether the baked-AMI option is worth the operational weight.
7. **Is ExternalDNS already running, and with what `txt-owner-id` and policy?** If it is running with defaults, adding a second cluster is an outage waiting to happen. Verify **before** step 6 of the migration.
8. **Does the release process include database migrations?** Determines how tight the primary→standby sync window must be.

## Sources

- [Private image replication in Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/replication.html) — the authoritative list of replication behaviours: the <30-minute expectation, no backfill, no delete propagation, policies not replicated, the tag-immutability trap, no chaining, cross-account destination policy requirements, and the 25-rule limits.
- [Amazon ECR announces ECR to ECR pull through cache (Mar 2025)](https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-ecr-pull-through-cache/) — the newer sync mechanism and why its upstream dependency makes it wrong for DR.
- [EKS Blueprints: ArgoCD multi-cluster hub and spoke](https://aws-ia.github.io/terraform-aws-eks-blueprints/patterns/gitops/gitops-multi-cluster-hub-spoke-argocd/) — AWS's reference hub-and-spoke pattern, including the per-spoke `awsAuthConfig.roleARN`. Useful as the thing to deviate from, and worth reading for the cluster-registration mechanics.
- [stolostron/argocd-pull-integration](https://github.com/stolostron/argocd-pull-integration) — the pull-model alternative to push-based hub-and-spoke.
- [external-dns#1588 — two ExternalDNS pods deleting each other's Route 53 entries](https://github.com/kubernetes-sigs/external-dns/issues/1588) and [helm/charts#22196 — ownership conflict in multi-cluster setup](https://github.com/helm/charts/issues/22196) — the mutual-deletion failure, filed and reproduced. Read these before deploying a second ExternalDNS.
- [external-dns TXT registry documentation](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/registry/txt.md) — how ownership is recorded and why `--txt-owner-id` is the control.
- [External Secrets Operator — AWS Secrets Manager provider](https://external-secrets.io/latest/provider/aws-secrets-manager/) — the `region` and `auth` fields on the store, and the `replicationLocations` support on `PushSecret`.
- [external-secrets#5882 — native fallback/failover between multiple SecretStores](https://github.com/external-secrets/external-secrets/issues/5882) — confirms there is no automatic store-level failover; each cluster must point at its own region.
- [ClusterSecretStore API reference](https://external-secrets.io/latest/api/clustersecretstore/) — cluster-scoped store semantics.
- [Reduce container startup time on Amazon EKS with a Bottlerocket data volume (AWS Containers Blog)](https://aws.amazon.com/blogs/containers/reduce-container-startup-time-on-amazon-eks-with-bottlerocket-data-volume/) — the pre-baked image volume approach that also covers Karpenter-launched nodes.
- [Start pods faster by prefetching images (AWS Containers Blog)](https://aws.amazon.com/blogs/containers/start-pods-faster-by-prefetching-images/) — AWS's own treatment of image prefetching.
- [The single-use DaemonSet pattern for pre-pulling images (Codefresh)](https://codefresh.io/blog/single-use-daemonset-pattern-pre-pulling-images-kubernetes/) — the pre-pull DaemonSet pattern in detail.
- [awslabs/soci-snapshotter](https://github.com/awslabs/soci-snapshotter) and the [SOCI on EKS guide](https://github.com/awslabs/soci-snapshotter/blob/main/docs/kubernetes.md) — lazy image loading, for when image size is genuinely the bottleneck.

### Explicitly not found

- **No public postmortem of a multi-region GitOps failover where the hub was in the failed region.** The failure mode is obvious on inspection and widely warned about; nobody has published the incident.
- **No AWS-published percentile distribution for ECR replication latency** — only the "majority under 30 minutes" statement. If your RTO depends on a tighter bound, you must measure it yourself or pre-pull.
- **No measured comparison of standby ALB cold-start behaviour under a sudden production-volume cutover.** The pre-warm concern is inferred from ALB scaling behaviour, not from a published benchmark.
