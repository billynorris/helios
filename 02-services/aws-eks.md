---
title: Amazon EKS — Multi-Region
service: eks
tags: [service, multi-region, eks, kubernetes, compute]
status: researched
replication: none (no multi-region cluster exists — you run a second cluster)
rpo_achievable: N/A — the cluster holds no business data; etcd holds config, which is re-derivable from Git
rto_achievable: "3–8 min if control plane + system node group are pre-provisioned; 15–25 min+ if the control plane is created at failover time (FAILS RTO outright)"
meets_targets: conditional
updated: 2026-09-16
---

# Amazon EKS — Multi-Region

## TL;DR

- **There is no such thing as a multi-region EKS cluster.** The control plane is a regional AWS resource with a regional API endpoint (`eks.<region>.amazonaws.com`). The standby region needs its own cluster, its own OIDC issuer, its own add-ons, its own node groups. Nothing about the cluster "replicates".
- **The EKS control plane must be pre-provisioned, full stop.** Creating one takes roughly 9–20 minutes depending on region — see [[#Real numbers for control plane creation]]. That alone consumes or exceeds the entire 15-minute RTO before a single pod has been scheduled. This is exactly the case [[research-brief]] calls out as an RTO-killer, and it is the single most important sentence in this note.
- **"Warm" here means: control plane always on + a small always-on system node group + workload capacity at or near zero.** The control plane bills `$0.10/cluster/hour` (~$73/month) whether it has zero nodes or five hundred. That $73 is the cheapest insurance in this entire programme. The system node group is where you choose your cost/speed point.
- **Karpenter cannot bootstrap itself from zero nodes.** The controller is a Deployment; it needs a node to run on. So a pilot-light cluster at literally zero EC2 instances needs either a 1–2 node managed node group or a Fargate profile for the Karpenter namespace — or you use **EKS Auto Mode**, where AWS runs the provisioning controller outside your data plane and a genuinely zero-node cluster can scale up on demand. See [[#The Karpenter bootstrap problem]].
- **The thing that will bite:** not the mechanics — the **drift**. A standby cluster that nobody upgrades, nobody deploys to, and nobody tests is not a standby, it is a 4-month-old cluster with expired add-on versions, a Kubernetes version drifting toward extended support at 6× the price, and IRSA roles that were never given a trust statement for the second cluster's OIDC issuer. See [[#Cluster version and config drift]].

## Does this service cross regions at all?

No, and not in any partial sense either:

| Thing | Regional or global? | Consequence for the standby |
|---|---|---|
| EKS control plane (`aws_eks_cluster`) | **Regional** | Standby needs its own. New name or same name, different region — both work, see [[#Naming]]. |
| Cluster API endpoint | **Regional**, per-cluster DNS name | `kubectl` contexts are per-cluster. Your CI needs two kubeconfigs. |
| OIDC issuer URL (`oidc.eks.<region>.amazonaws.com/id/<hash>`) | **Per-cluster**, unique hash | Every IRSA trust policy is cluster-specific. This is the big one — [[#IRSA across regions]]. |
| EKS managed node groups | **Regional**, tied to one cluster | Standby needs its own. |
| EKS add-ons (`aws_eks_addon`) | **Per-cluster** | Versions pin per-cluster and drift independently. |
| etcd / cluster state | **Regional**, AWS-managed, not exportable | You cannot replicate etcd. You re-create state from Git — see [[eks-workload-delivery]]. |
| ECR images | Regional registry, **replicable** | See [[aws-ecr]]. |
| IAM roles | **Global** | The role object crosses; the *trust policy* does not, because it names the cluster's OIDC provider. |

The only thing that is global is IAM, and IAM is precisely where the cross-region pain lives.

### Naming

You can give both clusters the same name (`helios-prod`) because cluster names are scoped per-region-per-account. This is attractive: manifests, kubeconfig context templates, ArgoCD cluster names and dashboards all stay symmetric. The cookiecutter templates in the central Terraform repo almost certainly already interpolate the region into names — resist the urge to add `-eu-west-2` here. Same name, different region, is cleaner, and it matches what [[aws-dynamodb]] is being forced into by Global Tables anyway.

The one place same-naming bites: an operator with two kubeconfigs where both contexts are called `helios-prod` and they run `kubectl delete` against the wrong one. Mitigate with context names that *do* carry the region (`helios-prod@eu-west-2`) even when the cluster name does not, and with `kubectl` prompt colouring. This is a runbook concern — see [[failover-runbook]].

## Real numbers for control plane creation

This is the number the whole design hinges on, so it is worth being precise and honest about provenance.

**AWS's own claim (2021):** AWS announced a 40% reduction in cluster creation time, stating you can create a new EKS control plane "in 9 minutes or less, on average" ([AWS What's New, March 2021](https://aws.amazon.com/about-aws/whats-new/2021/03/amazon-eks-reduces-cluster-creation-time-40-percent/)). AWS has not, as far as I can find, published an updated figure since. The long-running community request for faster creation is [aws/containers-roadmap#1227](https://github.com/aws/containers-roadmap/issues/1227).

**A measured third-party benchmark (April 2025):** [Eason Tech Talk](https://easontechtalk.com/unofficial-eks-cluster-creation-performance-across-aws-regions/) ran `eksctl create cluster --fargate` concurrently across 21 regions on 16 Apr 2025. Relevant rows:

| Region | Measured | Relevance |
|---|---|---|
| `eu-west-2` (London) | 13 m 13 s | EU standby |
| `eu-west-1` (Ireland) | 15 m 13 s | EU primary |
| `us-west-2` (Oregon) | 15 m 00 s | US standby |
| `us-east-1` (N. Virginia) | 20 m 32 s | US primary |
| `ca-central-1` (Montreal) | 15 m 14 s | CA primary |
| `ca-west-1` (Calgary) | **not measured** | CA standby — no public figure found |

> [!warning] Read the methodology before quoting these
> `eksctl create cluster` does *more* than create a control plane: it creates CloudFormation stacks for VPC and IAM, then a Fargate profile. So these are an **upper bound on the whole bootstrap**, not a clean control-plane-only measurement. The true control-plane-only figure sits somewhere between AWS's 9 minutes and these 13–20 minutes. It does not matter which: **every number in that range is ≥ 60% of the 15-minute RTO budget, and several exceed it entirely.**

**Conclusion, stated plainly: creating the EKS control plane at failover time is not on the table.** It must exist, warm and idle, before the incident starts. Everything else in this note is downstream of that one fact.

### Where the rest of the failover time goes

Assuming the control plane is pre-provisioned, the remaining clock:

| Step | Time | Notes |
|---|---|---|
| Human decision to fail over | 0–5 min | Not technical. Usually the real bottleneck. See [[failover-runbook]]. |
| Scale node capacity 0 → N (Cluster Autoscaler) | **3–5 min** | CA polls (default 10 s scan interval), simulates scheduling, then edits the ASG desired count and waits on the ASG's own provisioning path. |
| Scale node capacity 0 → N (Karpenter) | **~45–60 s** | Karpenter watches unschedulable pods via informers and calls `RunInstances` directly, skipping the ASG. Figures from vendor comparisons ([ScaleOps](https://scaleops.com/blog/karpenter-vs-cluster-autoscaler/), [CAST AI](https://cast.ai/blog/karpenter-vs-cluster-autoscaler/)) — treat as indicative, not AWS-published. |
| EC2 boot + kubelet register + node `Ready` | 60–120 s | Bottlerocket / AL2023 minimal AMIs are at the fast end. Included in the Karpenter figure above. |
| Image pull, first pod | **0 s – several min** | Entirely dependent on whether the image is in the local region's ECR and whether it is already on the node's disk. This is the most controllable variable and the most commonly ignored. See [[eks-workload-delivery]] and [[aws-ecr]]. |
| Readiness probes + app warm-up (JIT, connection pools, caches) | 30 s – several min | Application-specific. Measure it. |
| ALB target registration + healthy | 30–90 s | See [[aws-alb-nlb]]. |
| DNS/Route 53 failover propagation | TTL-bound | See [[aws-route53]]. |

**Realistic totals from a pre-provisioned cluster:**

- Warm standby with pods already running at 1 replica: **~2–4 minutes**, dominated by DNS and ALB health checks. Comfortably inside 15 min.
- Pilot light with a 2-node system group and Karpenter, images pre-cached: **~5–8 minutes**. Inside 15 min with margin.
- Pilot light at literally zero nodes, Cluster Autoscaler, cold image pulls: **~10–15 minutes**. On the edge — one slow image pull or one `InsufficientInstanceCapacity` retry and you have missed the target.
- Anything that includes creating the control plane: **fails.**

## The Karpenter bootstrap problem

Worth stating explicitly because it is the thing people design around too late.

Karpenter runs as a Deployment **inside the cluster it manages**. If the cluster has zero nodes, there is nowhere for the Karpenter pod to be scheduled, and therefore nothing to observe pending pods and call `RunInstances`. A truly zero-node cluster with self-managed Karpenter is inert — it will sit there with pending pods forever.

The [EKS Best Practices Guide](https://docs.aws.amazon.com/eks/latest/best-practices/karpenter.html) is direct about it: Karpenter should not run on nodes that Karpenter manages, and the controller needs a stable home. Three options:

| Option | What it costs idle | Failover speed | Notes |
|---|---|---|---|
| **Small managed node group** (2 × `t3.medium` or `m7g.medium`, across 2 AZs) | ~$25–60/month | Fastest — Karpenter already resident and watching | Also gives CoreDNS, the EBS CSI controller, the AWS Load Balancer Controller and your GitOps agent somewhere to live. **Recommended.** |
| **Fargate profile for the `karpenter` namespace** | Pay only while the Fargate pod runs — but it runs continuously if Karpenter is to be ready, so it is not free | Similar | AWS documents this pattern in [manage scale-to-zero scenarios with Karpenter and serverless](https://aws.amazon.com/blogs/containers/manage-scale-to-zero-scenarios-with-karpenter-and-serverless/). Note: CoreDNS defaults to EC2 and needs extra configuration to run on Fargate, so "everything on Fargate" is more work than it sounds. |
| **EKS Auto Mode** | $0.10/hr cluster fee only, with zero nodes | Fast, and no bootstrap problem at all | AWS runs the Karpenter-derived provisioning controller **outside** your data plane, and folds CNI/DNS/CSI/load-balancing into the AMI rather than DaemonSets ([Under the hood: EKS Auto Mode](https://aws.amazon.com/blogs/containers/under-the-hood-amazon-eks-auto-mode/)). A genuinely zero-node Auto Mode cluster can provision on the first pod. The catch: Auto Mode adds ~10–12% on top of the EC2 price while nodes run, and it takes opinionated control of the node lifecycle, which is a big change for a mature estate to adopt *only in the standby*. |

> [!tip] Auto Mode is genuinely interesting here, but only if you adopt it in *both* regions
> The cost profile — pay nothing for nodes while idle, no bootstrap node group, AWS keeps the add-ons current — is close to ideal for a pilot light. But running Auto Mode in the standby and self-managed node groups in the primary means you are failing over to a *differently shaped* cluster, which is exactly the drift problem in [[#Cluster version and config drift]] wearing a nicer hat. Either both or neither. Flagged in [[#Decisions to make]].

Cluster Autoscaler does not have the bootstrap problem in the same form — it also runs in-cluster, but the ASG desired-count can be raised from **outside** the cluster entirely, by an AWS API call (`aws eks update-nodegroup-config --scaling-config minSize=3,desiredSize=3`). That is a real advantage for a pilot light: your failover automation can raise capacity via the AWS API without needing anything alive in the cluster first. Karpenter cannot be driven that way; the only lever is Kubernetes objects.

**Practical consequence:** whichever autoscaler you use, the failover automation should scale the **managed node group** via the EKS API as step one, because that path works with an empty cluster and no kubeconfig. Karpenter then picks up the rest once it is resident.

## Warm standby shape

Three candidate shapes. Prices are rough, on-demand, `eu-west-2`-ish, per month, for one cluster. Confirm against the real bill — see [[cost-model]].

### Shape A — Pilot light (recommended)

```
Control plane            always on        $73/mo
System node group        2 × m7g.large    ~$110/mo   (Karpenter, CoreDNS, ALB controller,
                                                      ArgoCD agent, External Secrets)
Workload node capacity   0 nodes          $0
Workload Deployments     replicas: 0      $0
EBS for system nodes     2 × 20 GiB gp3   ~$4/mo
NAT gateway              pre-provisioned  ~$35/mo + data (see [[aws-vpc-networking]])
```

**~$220/month per standby cluster**, before NAT data charges. Failover = scale node group + scale Deployments. Measured earlier at **~5–8 minutes**. This is the shape that fits 15 minutes with margin without paying for idle workload capacity.

### Shape B — True warm standby

Same as A, but every Deployment runs at 1 replica and workload nodes are at a floor of 2–3. Costs whatever one replica of your stack costs — for a microservice estate this is not negligible, easily $500–2000/month per region.

**Buys you:** failover in ~2–4 minutes, and — far more valuable — **continuous proof that the standby works**. Images are pulled. Secrets resolve. IRSA tokens mint. Startup probes pass. Database connections open. Every one of those is a failure mode that Shape A only discovers during an incident.

### Shape C — Zero nodes

Control plane only, $73/month, Auto Mode or Fargate-hosted Karpenter. Cheapest that can still meet RTO, but every single failure mode is discovered at failover time. Only defensible with **scheduled, automated, weekly failover drills** that scale it up and run a smoke test. If you will not commit to the drills, do not choose this shape.

**Recommendation: Shape A, with one exception — run a single replica of a "canary" workload continuously.** One pod that pulls a real application image, assumes a real IRSA/Pod Identity role, reads a real replicated secret from [[aws-secrets-manager]], opens a connection to the standby [[aws-rds-postgres]] replica, and reports to [[cloudwatch-observability]]. That single pod converts Shape A's biggest weakness — untested plumbing — into a continuously-monitored signal, for the price of one pod. It is the highest-leverage thing in this note.

## IRSA across regions

This deserves its own section because it is the part that silently doesn't work.

### Why it breaks

IRSA works by EKS projecting a signed service-account token into the pod, and the pod calling `sts:AssumeRoleWithWebIdentity`. IAM validates the token against an **IAM OIDC identity provider** that you registered, whose URL is the cluster's issuer:

```
https://oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E
```

That hash is **unique per cluster**. It is generated at cluster creation. It is not derivable, not transferable, and not stable across a cluster rebuild. The trust policy on every role therefore reads:

```json
{
  "Effect": "Allow",
  "Principal": { "Federated": "arn:aws:iam::111122223333:oidc-provider/oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLED..." },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": { "StringEquals": {
    "oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLED...:sub": "system:serviceaccount:payments:payments-api",
    "oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLED...:aud": "sts.amazonaws.com"
  }}
}
```

IAM roles are global, so the *role* is visible from `eu-west-2`. But a pod in the `eu-west-2` cluster presents a token signed by the `eu-west-2` issuer, which this policy does not trust. **`AccessDenied`. Every IRSA-using pod in the standby fails to get credentials.**

This failure mode is particularly nasty because it is invisible until failover: the roles exist, `terraform plan` is clean, the IAM console looks healthy, and nothing in the standby exercises the path.

### The two fixes

**Fix 1 — one role, two trust statements.** Register an IAM OIDC provider for the standby cluster too, and add a second `Statement` (or a second `Federated` principal) to each role's trust policy.

Pros: one role ARN, so your Helm values / ServiceAccount annotations are **identical in both regions**. This matters a lot for GitOps — see [[eks-workload-delivery]] — because it means the same manifest deploys to both clusters unmodified.

Cons: **trust policy size.** The default quota for role trust policy length is **2,048 characters**, adjustable to a maximum of **4,096** ([IAM quotas](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_iam-quotas.html)). Two OIDC statements with full `sub` and `aud` conditions run roughly 700–900 characters. Two clusters fits. Two clusters plus staging plus a rebuilt cluster whose old issuer you never cleaned up starts to press. Community reports of hitting this wall at ~8 trust relationships are consistent with the 4,096 ceiling.

**Fix 2 — separate roles per cluster.** `payments-api-eu-west-1` and `payments-api-eu-west-2`, each with one trust statement.

Pros: no size limit worry, clean blast radius, each role's permissions can diverge if the standby genuinely needs less.

Cons: the ServiceAccount annotation `eks.amazonaws.com/role-arn` now differs per region, so your manifests are no longer identical and you need a Kustomize overlay, Helm value, or ApplicationSet parameter per region. Manageable, but it is a per-region difference and per-region differences are where drift lives.

### Fix 3 — EKS Pod Identity, which removes the problem entirely

[EKS Pod Identity](https://aws.amazon.com/blogs/containers/amazon-eks-pod-identity-a-new-way-for-applications-on-eks-to-obtain-iam-credentials/) (GA Nov 2023) moves the cluster→role mapping **out of the IAM trust policy and into the EKS API**. The role's trust policy becomes cluster-agnostic:

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "pods.eks.amazonaws.com" },
  "Action": ["sts:AssumeRole", "sts:TagSession"]
}
```

You then create an `aws_eks_pod_identity_association` per cluster binding namespace + service account → role. **The same role serves both clusters with zero trust-policy edits**, because the binding lives in EKS, not IAM. No OIDC provider to register. No per-cluster hash anywhere.

For a two-cluster-per-pair, three-pair estate — six clusters — this is a materially better fit than IRSA, and it is the answer to "what happens when we rebuild the standby cluster and every issuer hash changes". `terraform-aws-modules/eks` v21 has already moved this way: native IRSA support for Karpenter was **removed** and Pod Identity is enabled by default ([v21 upgrade notes](https://github.com/terraform-aws-modules/terraform-aws-eks)).

Caveats, honestly stated:

- Requires the **EKS Pod Identity Agent** add-on (a DaemonSet) on your nodes. One more add-on to version-pin.
- Does **not** work for Fargate pods. If your Karpenter-on-Fargate bootstrap (above) needs AWS credentials, that specific pod still needs IRSA.
- Some older AWS SDK versions do not support the container credentials provider path Pod Identity uses. Check your oldest service.
- Not every AWS service integration supported Pod Identity at launch; verify for anything unusual in your estate.
- The `eks-auth` API endpoint is what the agent calls. It **exists in `ca-west-1`** (`eks-auth.ca-west-1.api.aws`) — confirmed in the [EKS endpoints and quotas doc](https://docs.aws.amazon.com/general/latest/gr/eks.html). So Pod Identity is not a `ca-west-1` gap.

**Recommendation: migrate to Pod Identity as part of the multi-region work, not after it.** You are about to touch every IAM role that a pod assumes. Touching them once to add a second OIDC trust statement, then again later to convert to Pod Identity, is doing the work twice. If the migration is too large to land in this programme, use **Fix 1 (two trust statements)** so that manifests stay identical, and accept the 4,096-character ceiling as a tracked risk.

## Add-ons and version pinning

Every EKS cluster carries a set of add-ons whose versions are pinned **per cluster**. Left to `most_recent = true`, the two clusters will diverge the first time you `apply` them at different moments.

| Add-on | Managed by | Cross-region concern |
|---|---|---|
| **VPC CNI** (`vpc-cni`) | EKS add-on | Version drift changes IP allocation behaviour and prefix delegation. A CNI version that works with your subnet sizing in the primary may exhaust IPs in a standby with smaller subnets. See [[aws-vpc-networking]]. |
| **CoreDNS** | EKS add-on | Needs a node to run on — with zero nodes, DNS is down and *everything* fails, including image pulls that resolve ECR endpoints. Another argument for the system node group. |
| **kube-proxy** | EKS add-on | Version must be compatible with the control plane version. Skew rules bite during unsynchronised upgrades. |
| **EBS CSI driver** | EKS add-on | Needs IRSA/Pod Identity. Volumes it creates **do not cross regions** — see [[eks-stateful-workloads]]. |
| **EFS CSI driver** | EKS add-on | Relevant only if you use EFS — see [[eks-stateful-workloads]]. |
| **Pod Identity Agent** | EKS add-on | Prerequisite for Fix 3 above. |
| **AWS Load Balancer Controller** | Helm, *not* an EKS add-on | Creates ALBs/NLBs in its own region. See [[eks-workload-delivery]] and [[aws-alb-nlb]]. |
| **ExternalDNS** | Helm | Writes to a **global** Route 53 zone from both regions. Two ExternalDNS instances writing the same records is a genuine hazard — cover the `txt-owner-id` configuration. See [[aws-route53]]. |
| **cert-manager** | Helm | Usually for internal/mesh certs; public certs come from [[aws-acm]]. If it does DNS-01 against Route 53 it has the same dual-writer concern as ExternalDNS. |

**Pin everything explicitly, from one variable, applied to both clusters.** The cookiecutter template should carry a single `addon_versions` map that is *not* per-environment-overridable without a code review:

```hcl
variable "addon_versions" {
  description = "Pinned EKS add-on versions. Changed in lockstep across all regions in a pair."
  type        = map(string)
  default = {
    vpc-cni                = "v1.19.2-eksbuild.1"
    coredns                = "v1.11.4-eksbuild.2"
    kube-proxy             = "v1.31.3-eksbuild.2"
    aws-ebs-csi-driver     = "v1.38.1-eksbuild.1"
    eks-pod-identity-agent = "v1.3.4-eksbuild.1"
  }
}
```

> Those specific version strings are illustrative — resolve real ones with
> `aws eks describe-addon-versions --kubernetes-version 1.31 --addon-name vpc-cni`.
> Never commit a version string you have not resolved against the API.

## Cluster version and config drift

This is the section to read if you only read one.

### The failure mode

A standby cluster is, by construction, the cluster nobody looks at. Four months after it is built:

- The primary was upgraded 1.31 → 1.32 during a planned window. The standby was not, because the runbook said "upgrade the cluster" and the engineer upgraded *the* cluster.
- Helm charts in the primary have moved three minor versions via CI. The standby's charts were installed once by hand during the build-out.
- Three new microservices exist. They were added to the primary's ArgoCD app-of-apps. Nobody added them to the standby's.
- Two new IAM roles were created for those services, with one trust statement each, naming the primary's OIDC issuer.
- A `PodDisruptionBudget` and a `PriorityClass` were added in the primary to fix a production incident. The standby has neither.
- The standby's Kubernetes version has quietly crossed the 14-month standard-support boundary and is in **extended support at $0.60/cluster/hour** — 6× the normal rate, and if the cluster's upgrade policy is `STANDARD` rather than `EXTENDED`, AWS will **auto-upgrade it for you**, possibly at an inconvenient moment ([EKS version lifecycle](https://docs.aws.amazon.com/eks/latest/userguide/view-upgrade-policy.html), [extended support pricing](https://aws.amazon.com/blogs/containers/amazon-eks-extended-support-for-kubernetes-versions-pricing/)).

Then the region fails, you fail over, and you discover all six of those at once, at 3am, under pressure, with customers down. **A drifted standby is worse than no standby, because you planned around it.**

I looked hard for a published postmortem describing exactly this. **No public war story found** — organisations do not publish "our DR failed because we forgot about it". The absence of stories is not evidence it doesn't happen; it is evidence of what people are willing to write down.

### What actually prevents it

Ranked by how much they help per unit of effort:

1. **One Terraform module, two provider aliases, one `apply`.** The standby is not a separate stack that someone remembers to run. It is the same module invoked twice in the same root. If you cannot apply the primary without also applying the standby, they cannot diverge. This is the single highest-value structural decision and it belongs in [[terraform-repo-structure]].
2. **Kubernetes version as a single variable feeding both.** `var.kubernetes_version` used by both module calls. A version bump is one line and it necessarily hits both. Do not let the standby carry its own version variable "for flexibility" — flexibility here means drift.
3. **GitOps deploying to both clusters from one ApplicationSet.** Adding a service to the primary should be structurally impossible without adding it to the standby. See [[eks-workload-delivery]].
4. **Scheduled failover drills.** Quarterly at minimum, monthly if you can. The drill is the only thing that tests the parts Terraform and GitOps do not cover: images, secrets, IAM, database connectivity, DNS. See [[failover-runbook]].
5. **Drift detection as a CI job.** `terraform plan -detailed-exitcode` on a schedule against both regions, alerting on non-zero. Plus a simple diff of `kubectl get deploy,sts,ds -A -o jsonpath=...` between clusters. Crude, catches a lot.
6. **A `dr_readiness` dashboard.** Both clusters' Kubernetes versions, all add-on versions, node counts, and the canary pod's status, side by side, on one screen. If the two columns are not identical, that is an alert. Cheap to build, and it makes drift *visible* rather than discoverable.

### Upgrade ordering

Which cluster do you upgrade first?

**Upgrade the standby first.** It is the low-stakes environment that is nonetheless identically shaped to production — a better canary than staging, because it has production's exact networking, IAM, add-ons, and instance types. Upgrade standby, run the drill against the upgraded standby (which doubles as your upgrade validation), then upgrade the primary. This turns the drill and the upgrade into one activity, which is the only realistic way either of them happens consistently.

The counter-argument: for the window between the two upgrades, the pair is version-skewed, and if you fail over during that window you land on a version your application has not been validated against. Keep the window short — same day, ideally same change window — and note that a one-minor-version skew is very unlikely to break a workload that Kubernetes itself still supports.

## EC2 capacity in the standby during a real regional disaster

A real and under-discussed risk. The scenario: `eu-west-1` has a genuine regional event. **Everyone** with a warm standby in `eu-west-2` scales up simultaneously. `eu-west-2` is a smaller region than `eu-west-1`. Your `aws eks update-nodegroup-config` returns success and then the ASG or Karpenter gets `InsufficientInstanceCapacity` on `RunInstances`, retries, and your 15-minute RTO becomes "when AWS has spare m7g.2xlarge in eu-west-2b".

AWS's own DR guidance acknowledges this directly: the pilot light strategy "brings a risk that you might not be able to provision the compute capacity you need when you want to fail over to the secondary Region, especially if you require a specific instance type" ([Pilot light with reserved capacity](https://aws.amazon.com/blogs/architecture/pilot-light-with-reserved-capacity-how-to-optimize-dr-cost-using-on-demand-capacity-reservations/)).

### Mitigations, cheapest first

| Mitigation | Cost | Effectiveness |
|---|---|---|
| **Instance type flexibility** — Karpenter NodePool with a broad `instance-category`/`instance-generation` requirement rather than a pinned type, across all AZs | **Free** | High. The overwhelming majority of `InsufficientInstanceCapacity` events are type+AZ specific, not region-wide. Do this regardless of what else you do. |
| **Spread across all AZs** in the standby | Free | High, same reason. Note `ca-west-1` gives you 3 AZs, same as the others. |
| **Do not use Spot for the failover surge** | — | Spot is the *first* capacity to disappear in a regional capacity crunch. Spot is fine for the primary's batch work; it is actively wrong for the standby's failover capacity. |
| **A node floor in the standby** (Shape B) | Cost of those nodes | Capacity you already hold cannot be taken away. This is a real, underrated argument for the warm shape over the pilot light. |
| **On-Demand Capacity Reservations (ODCR)** | **Full on-demand rate, 24/7, whether or not an instance runs in it** | Total, for the exact type+AZ+count reserved. This is insurance priced as compute. |

### ODCR economics

You pay the on-demand rate for the reserved capacity continuously. For a standby that needs, say, 10 × `m7g.2xlarge`, that is essentially the cost of running those ten instances all year — which defeats most of the point of a pilot light.

Two things make it tolerable, both from the AWS blog above:

1. **Savings Plans apply to ODCR charges.** The blog states: *"By using Savings Plans, you can achieve up to a 72% discount, a very significant cost reduction for DR instances that have to stay available all year long."* Compute Savings Plans give ~27% (1-yr no-upfront) up to ~66–72% (3-yr all-upfront, instance SP). A DR reservation is the ideal Savings Plan candidate precisely because it never varies.
2. **Share the ODCR with non-production.** Put the reservation in a shared capacity pool and let dev/test/CI workloads run inside it. You are paying for the capacity anyway; running something useful in it costs nothing extra. At failover you evict the non-production workloads. AWS has since launched **interruptible ODCRs**, which let the capacity owner reclaim shared capacity with a 2-minute notice — purpose-built for this.

**Recommendation: do not buy ODCRs on day one.** Do instance-type flexibility and full-AZ spread first, because they are free and address most of the risk. Revisit ODCRs only if (a) the workload genuinely requires a narrow instance type (GPU, high-memory, specific local NVMe), or (b) a risk review decides that region-wide capacity exhaustion is a scenario you must survive rather than merely survive-usually. If you do buy them, buy them **with a Savings Plan and a non-production tenant**, or the finance conversation will kill the whole DR programme.

## `ca-west-1` — what's actually there

Checked directly rather than assumed, because [[research-brief]] flags Calgary as the region with real gaps.

| Fact | Finding | Source |
|---|---|---|
| EKS available? | **Yes.** `eks.ca-west-1.amazonaws.com` is a published service endpoint. | [EKS endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/eks.html) |
| EKS Pod Identity available? | **Yes.** `eks-auth.ca-west-1.api.aws` is published. | Same |
| FIPS endpoint? | **No** — ca-west-1 has only the standard endpoint, unlike us-east-1/us-west-2 which list `fips.eks.*`. Irrelevant unless you have a FIPS requirement. | Same |
| Availability Zones | **3** (`ca-west-1a/b/c`) — same as the other five regions in scope. The design does not need to change for AZ count. | [AWS regions doc](https://docs.aws.amazon.com/global-infrastructure/latest/regions/aws-regions.html) |
| Region age | Launched **20 Dec 2023** with 70 services — the youngest region in the estate by a wide margin. | [AWS News Blog](https://aws.amazon.com/blogs/aws/the-aws-canada-west-calgary-region-is-now-available/) |
| Supported Kubernetes versions | **No evidence of a version gap found.** AWS does not publish per-region Kubernetes version availability, and I found no report of a region lagging. Treat as "assume parity, verify before relying on it". | — |
| Instance types | **Narrower than ca-central-1.** Launch families were roughly C5, M5/M5d, R5, C6g/C6gn/C6i/C6id, M6g/M6gd/M6i/M6id, R6i/R6id, I3en/I4i, T3/T4g. Newer 7th-generation families (m7i, c7i, r7g and friends) require verification — I found no authoritative current list. | [AWS News Blog](https://aws.amazon.com/blogs/aws/the-aws-canada-west-calgary-region-is-now-available/) |

> [!important] Verify instance types with the API, not with a blog post
> ```bash
> aws ec2 describe-instance-type-offerings \
>   --region ca-west-1 --location-type availability-zone \
>   --filters Name=instance-type,Values='m7g.*','m6g.*','m7i.*','c7g.*' \
>   --query 'InstanceTypeOfferings[].[InstanceType,Location]' --output table
> ```
> Run this before the CA pair design is finalised. If the primary `ca-central-1` node groups use an instance family that Calgary does not have, that is a **hard blocker discovered at the worst possible moment**, and the fix — re-benchmarking the workload on a different family — is not a 15-minute fix. This is the single highest-priority verification item in this note. Carry it into [[region-pair-selection]].

**Broader `ca-west-1` caution:** a young region with 70 services at launch is a region where *something else* in the stack is more likely to be missing. EKS itself is fine. Check every other service in [[research-brief]]'s scope list against Calgary individually — notably [[aws-elasticache]], [[aws-eventbridge]] and [[aws-backup]]. EFS, at least, is **fine**: `elasticfilesystem.ca-west-1.amazonaws.com` exists and EFS replication is available in every region where EFS is available — see [[eks-stateful-workloads]]. Do not take third-party "supported regions" lists at face value here; several still quote the 2023 launch list.

## Terraform implementation

Shape for a cookiecutter-templated monorepo: **one root module per region-pair per environment**, two `module` calls, two provider aliases, one shared variable set. Using `terraform-aws-modules/eks/aws` v21 (which requires AWS provider ≥ v6.0 and Terraform ≥ 1.5.7, and renamed `cluster_name` → `name`, `cluster_version` → `kubernetes_version`).

### Providers

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

### The pair module — the variable surface you actually want

```hcl
# modules/eks-region/variables.tf
variable "name"               { type = string }
variable "kubernetes_version" { type = string }   # ONE value, used by BOTH calls
variable "addon_versions"     { type = map(string) }
variable "vpc_id"             { type = string }
variable "subnet_ids"         { type = list(string) }

variable "role" {
  description = "primary | standby. Drives capacity floors only — never behaviour."
  type        = string
  validation {
    condition     = contains(["primary", "standby"], var.role)
    error_message = "role must be primary or standby."
  }
}

variable "system_node_group" {
  description = "Always-on capacity that hosts Karpenter, CoreDNS, controllers."
  type = object({
    instance_types = list(string)
    min_size       = number
    max_size       = number
    desired_size   = number
  })
  default = {
    instance_types = ["m7g.large", "m6g.large"]   # two families = capacity flexibility
    min_size       = 2
    max_size       = 4
    desired_size   = 2
  }
}

variable "workload_node_group" {
  description = "Scaled to zero in the standby; scaled at failover. Karpenter handles the rest."
  type = object({
    instance_types = list(string)
    min_size       = number
    max_size       = number
    desired_size   = number
  })
}
```

> The `role` variable is deliberately narrow. It must **only** influence capacity numbers.
> The moment `role` starts gating *features* — "standby doesn't need the mesh", "standby
> skips this add-on" — you have built two different clusters and the drift problem is
> structural rather than accidental.

### The cluster

```hcl
# modules/eks-region/main.tf
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 21.0"

  name               = var.name              # same name in both regions, by design
  kubernetes_version = var.kubernetes_version

  vpc_id     = var.vpc_id
  subnet_ids = var.subnet_ids

  endpoint_public_access  = false
  endpoint_private_access = true

  # Pod Identity over IRSA: cluster-agnostic IAM roles, no per-cluster OIDC hash.
  # See the IRSA section of this note for why this matters across regions.
  enable_irsa = true   # keep on during migration; the addon below is the real mechanism

  addons = {
    coredns = {
      addon_version = var.addon_versions["coredns"]
      # CoreDNS must land on the always-on system nodes, or a zero-workload-node
      # cluster has no DNS and cannot even pull images.
      configuration_values = jsonencode({
        nodeSelector = { "helios.io/pool" = "system" }
        tolerations  = [{ key = "CriticalAddonsOnly", operator = "Exists" }]
      })
    }
    kube-proxy             = { addon_version = var.addon_versions["kube-proxy"] }
    vpc-cni                = { addon_version = var.addon_versions["vpc-cni"], before_compute = true }
    aws-ebs-csi-driver     = { addon_version = var.addon_versions["aws-ebs-csi-driver"] }
    eks-pod-identity-agent = { addon_version = var.addon_versions["eks-pod-identity-agent"] }
  }

  eks_managed_node_groups = {
    # Always on, both regions. Hosts Karpenter, CoreDNS, ALB controller,
    # External Secrets, the GitOps agent, and the DR canary pod.
    system = {
      instance_types = var.system_node_group.instance_types
      min_size       = var.system_node_group.min_size
      max_size       = var.system_node_group.max_size
      desired_size   = var.system_node_group.desired_size
      ami_type       = "BOTTLEROCKET_ARM_64"

      labels = { "helios.io/pool" = "system" }
      taints = {
        critical = { key = "CriticalAddonsOnly", value = "true", effect = "NO_SCHEDULE" }
      }
    }

    # Zero in the standby. Scaled by the failover automation via the EKS API —
    # note this path works with an empty cluster and no kubeconfig, which is
    # exactly what you want as failover step one.
    workload = {
      instance_types = var.workload_node_group.instance_types
      min_size       = var.workload_node_group.min_size
      max_size       = var.workload_node_group.max_size
      desired_size   = var.workload_node_group.desired_size
      ami_type       = "BOTTLEROCKET_ARM_64"
      labels         = { "helios.io/pool" = "workload" }
    }
  }

  # Terraform must not fight the failover automation or Karpenter over node counts.
  # Without this, the next `terraform apply` after a failover scales you back to zero.
  node_security_group_tags = { "karpenter.sh/discovery" = var.name }
}
```

> [!warning] `desired_size` and `terraform apply` after a failover
> This is a classic own-goal. You fail over, automation sets `desiredSize=20`, service is
> restored — and then someone merges an unrelated PR and the pipeline applies
> `desired_size = 0` from the standby's tfvars, deleting every node under the running
> workload. The v21 module still surfaces `desired_size`; guard it with a
> `lifecycle { ignore_changes = [...] }` on the node group, or — better — make the
> failover automation write the new desired size back into the tfvars/SSM parameter
> that Terraform reads. Covered in [[failover-runbook]]; whichever you choose, choose
> one *before* the first drill, not after the first outage.

### Wiring the pair

```hcl
# environments/prod-eu/main.tf
locals {
  kubernetes_version = "1.31"   # ONE line. Bumping it necessarily hits both regions.
}

module "eks_primary" {
  source    = "../../modules/eks-region"
  providers = { aws = aws.primary }

  name                = "helios-prod"
  kubernetes_version  = local.kubernetes_version
  addon_versions      = var.addon_versions
  role                = "primary"
  vpc_id              = module.vpc_primary.vpc_id
  subnet_ids          = module.vpc_primary.private_subnets
  workload_node_group = { instance_types = ["m7g.2xlarge", "m6g.2xlarge"], min_size = 6, max_size = 40, desired_size = 6 }
}

module "eks_standby" {
  source    = "../../modules/eks-region"
  providers = { aws = aws.standby }

  name                = "helios-prod"          # same name, different region
  kubernetes_version  = local.kubernetes_version
  addon_versions      = var.addon_versions     # same pins
  role                = "standby"
  vpc_id              = module.vpc_standby.vpc_id
  subnet_ids          = module.vpc_standby.private_subnets
  workload_node_group = { instance_types = ["m7g.2xlarge", "m6g.2xlarge"], min_size = 0, max_size = 40, desired_size = 0 }
}
```

The only differences between the two calls are the provider, the VPC, and three numbers. That is the target state — and it is what makes drift structurally hard.

### IRSA with two trust statements (if you are not moving to Pod Identity yet)

```hcl
data "aws_iam_policy_document" "payments_api_trust" {
  dynamic "statement" {
    for_each = {
      primary = module.eks_primary.oidc_provider_arn
      standby = module.eks_standby.oidc_provider_arn
    }
    content {
      effect  = "Allow"
      actions = ["sts:AssumeRoleWithWebIdentity"]
      principals {
        type        = "Federated"
        identifiers = [statement.value]
      }
      condition {
        test     = "StringEquals"
        variable = "${replace(
          statement.key == "primary" ? module.eks_primary.oidc_provider : module.eks_standby.oidc_provider,
          "https://", ""
        )}:sub"
        values = ["system:serviceaccount:payments:payments-api"]
      }
      condition {
        test     = "StringEquals"
        variable = "${replace(
          statement.key == "primary" ? module.eks_primary.oidc_provider : module.eks_standby.oidc_provider,
          "https://", ""
        )}:aud"
        values = ["sts.amazonaws.com"]
      }
    }
  }
}

resource "aws_iam_role" "payments_api" {
  name               = "helios-prod-payments-api"   # ONE role, both clusters
  assume_role_policy = data.aws_iam_policy_document.payments_api_trust.json
}
```

### The same thing with Pod Identity — note how much smaller it is

```hcl
resource "aws_iam_role" "payments_api" {
  name = "helios-prod-payments-api"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "pods.eks.amazonaws.com" }
      Action    = ["sts:AssumeRole", "sts:TagSession"]
    }]
  })
}

resource "aws_eks_pod_identity_association" "payments_api_primary" {
  provider        = aws.primary
  cluster_name    = module.eks_primary.cluster_name
  namespace       = "payments"
  service_account = "payments-api"
  role_arn        = aws_iam_role.payments_api.arn
}

resource "aws_eks_pod_identity_association" "payments_api_standby" {
  provider        = aws.standby
  cluster_name    = module.eks_standby.cluster_name
  namespace       = "payments"
  service_account = "payments-api"
  role_arn        = aws_iam_role.payments_api.arn
}
```

No OIDC hashes anywhere. Rebuild either cluster and nothing in IAM changes. Add a third cluster and it is six lines. **This is the argument for Pod Identity in one screen.**

## Migration path from single-region

The good news: **this is almost entirely additive.** You are creating new resources in a new region, not mutating live ones. Nothing here forces replacement of the running cluster — with two exceptions, flagged below.

1. **Verify `ca-west-1` instance type availability** (the API call above). Do this first; it can invalidate the CA pair design. Everything else waits on nothing.
2. **Stand up the standby VPC.** Prerequisite, and the long pole if you need peering/TGW/Direct Connect. See [[aws-vpc-networking]].
3. **Refactor the existing cluster into the pair module.** This is the one step with replacement risk. Moving `aws_eks_cluster` between module paths changes its Terraform address — use `moved` blocks (or `terraform state mv`) and **read the plan character by character**. A destroy/create on a live EKS control plane is a full outage plus a new OIDC issuer hash, which invalidates every IRSA trust policy you have. `name` and `vpc_config.subnet_ids` changes also force replacement; keep them byte-identical during the refactor.
4. **Apply the standby cluster.** New control plane, ~15 minutes, zero production impact.
5. **Add the second OIDC provider trust statement (or Pod Identity associations) to every role.** Purely additive to IAM. Do this before any workload lands.
6. **Install the platform layer in the standby** — Karpenter, ALB controller, External Secrets, ExternalDNS (see [[eks-workload-delivery]] for the ExternalDNS dual-writer hazard), observability agents. Via GitOps, not by hand.
7. **Deploy the DR canary pod.** One pod, real image, real IAM, real secret, real DB connection. Alert on it. From this point forward the standby is continuously proven rather than hoped-about.
8. **Deploy all workloads at `replicas: 0`** so the manifests, images, secrets and RBAC are all present and correct.
9. **Run the first drill.** Scale up, smoke test, scale down. Time it. The number you get is your real RTO; everything before this step was an estimate.
10. **Add the drill to the calendar.** Quarterly minimum.

## Failover procedure

Assumes Shape A. Target: node capacity ready in ~5 minutes, traffic in ~8.

1. **Human decision.** Not automatable, and should not be. See [[failover-runbook]].
2. **Scale the workload node group via the AWS API** — first, because it is the longest pole and it works without a kubeconfig:
   ```bash
   aws eks update-nodegroup-config --region eu-west-2 \
     --cluster-name helios-prod --nodegroup-name workload \
     --scaling-config minSize=6,maxSize=40,desiredSize=6
   ```
3. **Promote the data layer.** [[aws-rds-postgres]] replica promotion, [[aws-dynamodb]] Global Tables (already active), [[aws-elasticache]]. Runs in parallel with step 2 — do not serialise these.
4. **Scale workloads up.** Either a GitOps change (`replicas: 0` → `N` in the standby overlay, committed and synced) or a scripted `kubectl scale`. GitOps is auditable and slower; the script is faster and drifts from Git. **Recommendation: GitOps with a pre-merged, pre-reviewed PR** that the runbook merges — you get the audit trail without the 3am authoring.
5. **Wait for readiness.** Pods ready, ALB targets healthy. Karpenter adds capacity beyond the node group floor automatically as pods go pending.
6. **Flip DNS.** [[aws-route53]] — the actual moment of failover, and the only irreversible-feeling one.
7. **Verify.** Smoke tests, error rates, the canary. Do not declare success on "pods are Running".

## Failback

Harder than failover and consistently under-planned.

The asymmetry: failover is *decided* under duress but *executed* against a standby that was built for it. Failback is executed against a primary that has been down, may have partially-stale state, and is now the cold side. Specifically:

- **The old primary's EKS cluster is fine** — it is AWS-managed and it comes back healthy. This is the easy part.
- **The data is not fine.** Whatever wrote to the standby during the outage must get back. This is a database problem, not a Kubernetes problem — see [[aws-rds-postgres]] and [[aws-dynamodb]]. It usually means rebuilding replication in the reverse direction and waiting for it to catch up.
- **Roles swap.** The standby's node group now holds production capacity; the old primary's is at whatever it was. Failback means scaling one up and one down — and your Terraform's `desired_size` values now describe the wrong region. Whatever mechanism you chose in the `desired_size` warning above has to work in both directions.
- **Do not fail back under pressure.** Once the standby is serving successfully, the incident is over. Failback is a planned change for a weekday morning. Write that into the runbook explicitly, because the instinct at 4am is to put things back.
- **Realistically, consider not failing back at all.** If both regions are truly symmetric — which is the whole point of this design — "the standby is now the primary" is a legitimate end state, and the cheapest one. It also means you have just proven the new primary works. Revisit at the next maintenance window.

## Gotchas

1. **The OIDC issuer hash changes if you ever recreate a cluster.** Every IRSA trust policy silently breaks. Pod Identity removes this entire class of failure.
2. **Zero nodes means zero CoreDNS means zero DNS.** Image pulls resolve ECR hostnames. A truly empty cluster cannot pull the image that would give it DNS. Always keep the system node group, or use Auto Mode / Fargate deliberately.
3. **Karpenter cannot bootstrap itself from zero nodes.** Covered above. Also: never let Karpenter manage the node its own controller runs on — consolidation will evict it.
4. **`desired_size` in Terraform vs. the failover automation.** The next unrelated `apply` scales your failed-over region back to zero. Decide the reconciliation strategy before the first drill.
5. **Extended support at $0.60/hr on a forgotten standby.** 6× the control plane cost on the cluster nobody looks at. And with a `STANDARD` upgrade policy, AWS eventually upgrades it *for* you, unattended.
6. **Two ExternalDNS instances writing one global Route 53 zone.** Without correct `txt-owner-id` and a policy of `upsert-only` (or `sync` scoped per-owner), the standby's ExternalDNS will happily delete the primary's records because it doesn't see those services. This is a genuine outage waiting in a "harmless" add-on. See [[aws-route53]].
7. **Security group and NACL drift.** The standby VPC was built once, by hand-ish, and every firewall change since landed only in the primary. Symptom at failover: pods run, health checks fail, nothing in the Kubernetes layer looks wrong. See [[aws-vpc-networking]].
8. **Subnet IP exhaustion in the standby.** The standby's subnets were sized for a pilot light. At failover you need full production pod density, and the VPC CNI allocates ENIs/IPs per node. Size the standby's subnets for **full production**, not for the idle state. Cheap to get right now, impossible to fix during an incident.
9. **Service quotas are per-region and default in the standby.** Nobody has ever raised a limit in `eu-west-2` because nothing runs there. EC2 vCPU quotas in particular default low and are the classic failover blocker. Audit and raise **every** relevant quota in the standby to match the primary — this is a quiet, boring, essential task.
10. **`ca-west-1` instance families.** Verify before finalising. Repeated because it is the one that can invalidate the design.
11. **Cross-region `terraform apply` ordering.** Both clusters in one root means one `apply` touching two regions. A failure partway leaves a half-applied pair. Not fatal, but ensure the pipeline re-runs to convergence rather than alerting and stopping.
12. **kubeconfig context confusion at 3am.** Same cluster name in two regions is right for the config and wrong for the human. Region-qualified context names, always.

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Standby shape | Pilot light: control plane + 2 system nodes, workloads at 0 (~$220/mo) | Warm standby: 1 replica of everything, node floor ($500–2000/mo) | **A, plus a DR canary pod.** Gets pilot-light cost with warm-standby confidence in the parts that actually break. |
| Workload identity | IRSA with two trust statements per role | EKS Pod Identity | **B.** Cluster-agnostic roles, no OIDC hashes, survives cluster rebuilds, no 4,096-char ceiling. Do it as part of this programme, not after. |
| Autoscaler | Cluster Autoscaler (scalable from outside the cluster via the EKS API) | Karpenter (~45–60 s provisioning, better bin-packing) | **Both, layered.** Managed node group floor raised via the EKS API as failover step one; Karpenter handles everything above the floor. Belt and braces, and it removes the bootstrap problem. |
| Node management | Self-managed node groups + Karpenter, both regions | EKS Auto Mode, both regions | **A**, unless the estate is already moving to Auto Mode. Auto Mode's zero-node economics are ideal for a standby, but adopting it *only* in the standby creates exactly the asymmetry this design exists to prevent. |
| Cluster naming | Same name both regions | Region-suffixed names | **A.** Symmetric manifests and GitOps. Mitigate the human risk in kubeconfig context names. |
| Upgrade order | Standby first, then primary | Primary first | **A.** The standby becomes a production-shaped canary, and the upgrade and the DR drill become one activity — the only way both happen reliably. |
| Capacity guarantee | Instance-type + AZ flexibility (free) | On-Demand Capacity Reservations (full on-demand rate 24/7) | **A first.** Revisit B only for narrow instance requirements or an explicit risk decision, and then only with a Savings Plan and a non-production tenant sharing the reservation. |
| Terraform layout | One root, two module calls, two provider aliases | Separate roots per region | **A.** Structurally prevents version and config drift; you cannot apply one without the other. Confirm against [[terraform-repo-structure]]. |

## Cost

Per standby cluster, per month, rough, on-demand:

| Line | Shape A (pilot light) | Shape B (warm) | Shape C (zero nodes) |
|---|---|---|---|
| EKS control plane | $73 | $73 | $73 |
| System node group (2 × m7g.large) | ~$110 | ~$110 | $0 (Auto Mode / Fargate) |
| Workload capacity | $0 | $500–2000 | $0 |
| EBS (system nodes) | ~$4 | ~$10 | $0 |
| NAT gateway | ~$35 + data | ~$35 + data | ~$35 + data |
| ECR replication storage | see [[aws-ecr]] | see [[aws-ecr]] | see [[aws-ecr]] |
| **Total** | **~$220** | **~$730–2,230** | **~$110** |

Across three pairs: **~$660/month** for Shape A. Against the cost of a regional outage, this is not a number worth optimising. Control plane fee is fixed and non-negotiable ([EKS pricing](https://aws.amazon.com/eks/pricing/)).

**The levers, in order of size:**
1. **Workload replica count in the standby** — the difference between A and B, and by far the biggest line.
2. **Graviton for the system node group** — ~20% off for pods that do nothing but wait.
3. **NAT gateway** — one per AZ is the default and it is three times the cost of one. A standby that idles does not need per-AZ NAT redundancy; one NAT is a defensible standby-only saving. See [[aws-vpc-networking]].
4. **Do not let the standby drift into extended support.** $0.60/hr is $438/month — twice the entire Shape A cost, on a cluster doing nothing.
5. **Savings Plans cover the system node groups** across all three standbys. Predictable, never-varying load; ideal SP candidate.

## Open questions

1. **Does `ca-west-1` offer the instance families `ca-central-1` currently runs?** The blocking verification. Run the `describe-instance-type-offerings` call.
2. **Is the standby in the same AWS account as the primary?** Changes the IAM story, the ECR replication story, and the Terraform provider configuration. [[aws-acm]] flags the same question — worth answering once, centrally, in [[terraform-repo-structure]].
3. **What is the current workload identity mechanism — IRSA everywhere, or already some Pod Identity?** Determines whether the recommendation above is a migration or a greenfield choice.
4. **Karpenter or Cluster Autoscaler today?** And is Karpenter's controller already isolated onto a dedicated node group?
5. **What are the actual EC2 vCPU service quotas in the three standby regions right now?** Almost certainly default. Almost certainly insufficient. Needs a ticket per region, and those tickets take days.
6. **Are workloads truly stateless?** If yes, [[eks-stateful-workloads]] is largely moot and this programme is much smaller than it looks. Answer this early — it is the single biggest scope determinant.
7. **What is the tolerance for a quarterly failover drill?** Every recommendation about drift assumes drills happen. If they will not happen, Shape B becomes much more attractive, because continuous running is the only substitute for testing.
8. **Which Kubernetes version are the clusters on, and how far into the 14-month standard support window?** Determines whether an upgrade lands inside this programme or just after it.

## Sources

- [Amazon EKS endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/eks.html) — authoritative list of EKS and `eks-auth` regional endpoints; confirms both exist in `ca-west-1`, and gives the per-region quotas (100 clusters, 30 node groups/cluster, 450 nodes/node group).
- [Amazon EKS reduces control plane creation time by 40% (AWS, Mar 2021)](https://aws.amazon.com/about-aws/whats-new/2021/03/amazon-eks-reduces-cluster-creation-time-40-percent/) — AWS's own "9 minutes or less, on average" claim. The most favourable number available, and still too slow for a 15-minute RTO.
- [aws/containers-roadmap#1227 — Reduction in EKS cluster creation time](https://github.com/aws/containers-roadmap/issues/1227) — the long-running community ask; useful for tracking whether this ever gets fast enough to matter.
- [Unofficial EKS cluster creation performance across AWS regions (Eason Tech Talk, Apr 2025)](https://easontechtalk.com/unofficial-eks-cluster-creation-performance-across-aws-regions/) — the only per-region measured dataset I could find. `eksctl`-based, so an upper bound. Provides the eu-west-1/2, us-east-1, us-west-2 and ca-central-1 figures used above. `ca-west-1` not covered.
- [EKS Best Practices Guide — Karpenter](https://docs.aws.amazon.com/eks/latest/best-practices/karpenter.html) — the authoritative statement that Karpenter needs a stable home outside the nodes it manages.
- [Manage scale-to-zero scenarios with Karpenter and serverless (AWS Containers Blog)](https://aws.amazon.com/blogs/containers/manage-scale-to-zero-scenarios-with-karpenter-and-serverless/) — the Fargate-profile-for-Karpenter pattern, plus the CoreDNS-on-Fargate caveat.
- [Under the hood: Amazon EKS Auto Mode (AWS Containers Blog)](https://aws.amazon.com/blogs/containers/under-the-hood-amazon-eks-auto-mode/) — confirms AWS runs the Karpenter-derived provisioner outside the customer data plane and folds CNI/DNS/CSI into the AMI, which is what makes a genuinely zero-node cluster viable.
- [Amazon EKS Pod Identity announcement (AWS Containers Blog)](https://aws.amazon.com/blogs/containers/amazon-eks-pod-identity-a-new-way-for-applications-on-eks-to-obtain-iam-credentials/) — the mechanism, and the explicit statement that roles are no longer tied to a single cluster.
- [IAM and AWS STS quotas](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_iam-quotas.html) — role trust policy length: 2,048 characters default, 4,096 maximum. The hard ceiling on the "N trust statements" approach.
- [Pilot light with reserved capacity: optimise DR cost using ODCRs (AWS Architecture Blog)](https://aws.amazon.com/blogs/architecture/pilot-light-with-reserved-capacity-how-to-optimize-dr-cost-using-on-demand-capacity-reservations/) — AWS acknowledging the standby-capacity risk directly, the Savings Plans discount figures (up to 72%), and the share-with-non-production pattern.
- [Disaster recovery options in the cloud (AWS whitepaper)](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html) — the canonical pilot light vs warm standby definitions this note's shapes map onto.
- [Amazon EKS extended support pricing (AWS Containers Blog)](https://aws.amazon.com/blogs/containers/amazon-eks-extended-support-for-kubernetes-versions-pricing/) and [cluster upgrade policy](https://docs.aws.amazon.com/eks/latest/userguide/view-upgrade-policy.html) — the $0.60/hr penalty and the `STANDARD`/`EXTENDED` auto-upgrade behaviour that makes a forgotten standby expensive *and* unpredictable.
- [Amazon EKS pricing](https://aws.amazon.com/eks/pricing/) — the $0.10/cluster/hour control plane fee.
- [terraform-aws-modules/terraform-aws-eks](https://github.com/terraform-aws-modules/terraform-aws-eks) — v21 requires AWS provider ≥ 6.0 / Terraform ≥ 1.5.7, renames `cluster_name`→`name` and `cluster_version`→`kubernetes_version`, drops the `aws-auth` submodule, and defaults Karpenter to Pod Identity.
- [The AWS Canada West (Calgary) Region is now available (AWS News Blog)](https://aws.amazon.com/blogs/aws/the-aws-canada-west-calgary-region-is-now-available/) — launch date, 70 services at launch, and the launch instance families.
- [AWS Regions and Availability Zones](https://docs.aws.amazon.com/global-infrastructure/latest/regions/aws-regions.html) — AZ counts; `ca-west-1` has 3.
- [ScaleOps: Karpenter vs Cluster Autoscaler](https://scaleops.com/blog/karpenter-vs-cluster-autoscaler/) and [CAST AI: Karpenter vs Cluster Autoscaler](https://cast.ai/blog/karpenter-vs-cluster-autoscaler/) — the 45–60 s vs 3–5 min provisioning comparison. **Vendor blogs, not AWS-published benchmarks** — directionally reliable, but measure your own before you put a number in a runbook.

### Explicitly not found

- **No AWS-published benchmark of managed node group 0→N scale time.** The 3–5 minute figure is from vendor comparisons and community reports only.
- **No public postmortem of a multi-region Kubernetes failover that failed due to standby drift.** Searched specifically for this. The failure mode is widely described in guidance; nobody has published the incident.
- **No per-region Kubernetes version availability matrix from AWS**, so the `ca-west-1` version-parity assumption is unverified.
- **No authoritative current instance-type list for `ca-west-1`** beyond the launch announcement. Must be checked with the API.
