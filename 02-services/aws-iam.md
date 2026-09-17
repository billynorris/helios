---
title: IAM — Multi-Region
service: iam
tags: [service, multi-region, iam, sts, scp, foundation]
status: researched
replication: none — IAM is global, there is nothing to replicate
rpo_achievable: N/A — no regional state
rto_achievable: "0 if everything is pre-created; unbounded if you try to create IAM resources at failover"
meets_targets: yes — with the caveats below
updated: 2026-09-17
---

# IAM — Multi-Region

## TL;DR

- **IAM is global.** Roles, policies, users, groups, instance profiles, OIDC providers and SAML providers are account-wide, not regional. `iam.amazonaws.com` is a single endpoint served for every Region. **"Mirroring IAM" is mostly a no-op** — the role your pods assume in `eu-west-1` is the same role object in `eu-west-2`. This is the single largest simplification in the entire programme and it deserves to be said before anything else.
- **The first blocker you will hit is not IAM at all — it is Region enablement.** `ca-west-1` is an **opt-in Region**. IAM data and credentials are *not* present there until someone with `account:EnableRegion` in the management account turns it on, and AWS says that propagation "takes a few minutes for most accounts, but can sometimes take several hours". Nothing — not a VPC, not a Terraform plan, not an `sts:GetCallerIdentity` — works in `ca-west-1` before that. `eu-west-2` and `us-west-2` are default-enabled and need nothing.
- **The second blocker is your own SCPs.** If the organisation runs a Region-deny SCP (Control Tower's Region deny control, or a hand-rolled `aws:RequestedRegion` deny), the three standby Regions must be **added to the allowlist before the first `terraform apply`**. Teams routinely lose a day to this, because the failure mode is a generic `AccessDenied` on an API that the engineer *knows* they have permission for.
- **The exceptions to "IAM is global" that actually bite:** region-specific ARNs inside policy documents; `aws:RequestedRegion` conditions on your own roles; role trust policies bound to one EKS cluster's OIDC provider; and permissions boundaries that were themselves written with a single region in mind.
- **The thing that will bite:** IAM is eventually consistent and AWS explicitly tells you not to put IAM changes in a high-availability code path. **Every IAM object the standby needs must exist before the incident starts.** There is no "create the role at failover" plan that survives contact with a 15-minute RTO.

---

## Does this service cross regions at all?

It does not need to. IAM is a **global service in the `aws` partition**. Every Region's IAM endpoint is literally the same hostname — the [IAM endpoints and quotas reference](https://docs.aws.amazon.com/general/latest/gr/iam-service.html) lists `iam.amazonaws.com` for `eu-west-1`, `eu-west-2`, `us-east-1`, `us-west-2`, `ca-central-1` and `ca-west-1` alike. There is no `iam.eu-west-2.amazonaws.com`.

The boundary that *is* hard is the **partition**, not the Region. Per the [AWS Account Management reference](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-regions.html): "Partitions have independent instances of AWS Identity and Access Management (IAM) and provide a hard boundary between Regions in different partitions." All six regions in scope are in the `aws` partition, so this is a non-issue here — but it is the reason `arn:aws:` vs `arn:aws-cn:` vs `arn:aws-us-gov:` matters, and a good argument for using `data.aws_partition.current.partition` rather than hardcoding `aws` in policy ARNs.

Practical consequence for the Terraform estate: **IAM resources should be declared exactly once, in one provider, and not duplicated behind a `aws.standby` alias.** If you instantiate an IAM role module twice with two aliases you will get two roles with different names (or a name collision and a failed apply), not a mirror. Declare IAM under the primary provider; it is visible everywhere.

### The one genuinely regional thing: Region enablement

This is the exception that matters most for this project, and it is not usually filed under "IAM":

> "When you create an AWS account, your IAM data and credentials are automatically configured to work across all default Regions… AWS opt-in Regions are disabled by default, **and IAM data and credentials are not initially available in these Regions**, which prevents access to AWS services in that Region. When you choose to enable an opt-in Region, AWS propagates your IAM data and credentials to that Region."
> — [Enable or disable AWS Regions in your account](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-regions.html)

Verified status for the six Regions in scope:

| Region | Opt-in or default? | Action needed |
|---|---|---|
| `eu-west-1` | Default | None |
| `eu-west-2` | Default | None |
| `us-east-1` | Default | None |
| `us-west-2` | Default | None |
| `ca-central-1` | Default | None |
| **`ca-west-1`** | **Opt-in** | **`account:EnableRegion` per account, asynchronously, before anything else** |

Operational facts worth writing into the runbook:

- The call is `aws account enable-region --region-name ca-west-1`, optionally with `--account-id` from the management account or the Account Management delegated admin, and requires trusted access for AWS Account Management to be enabled in the Organization.
- It is **asynchronous** with statuses `ENABLING` / `ENABLED` / `DISABLING` / `DISABLED`, polled with `get-region-opt-status`. You cannot cancel mid-flight.
- **6 concurrent region-opt requests per account; 50 open across an Organization.** If the estate has dozens of accounts, enabling `ca-west-1` everywhere is a batched, rate-limited campaign, not a single command. Plan it.
- "A completed (Enabled/Disabled) region-opt request is dependent on the provisioning of key underlying AWS services. There might be some AWS services that will not be immediately usable despite the status being `ENABLED`." Translation: `ENABLED` is necessary, not sufficient. Budget a soak day.
- There is **no first-class Terraform resource** to enable a Region — the provider has a `aws_account_regions` *data source* but no `aws_account_region` resource. This step is a one-off CLI/console action outside Terraform, which means it must live in a runbook and a checklist rather than in the repo. Subscribe to the EventBridge region-opt status notification if you want it automated.
- Disabling a Region later **does not delete resources and does not stop the charges** — it just removes your IAM access to them. Do not "save money" by disabling `ca-west-1`.

**This is a dependency of [[aws-vpc-networking]], not a consequence of it.** Put it at the top of the CA pair's project plan.

---

## Exception 1: region-specific ARNs in policy documents

The policies themselves are global; the ARNs inside them are not. `arn:aws:sqs:eu-west-1:123456789012:orders` does not grant anything in `eu-west-2`. When the standby's workload assumes the same role and talks to the standby's queue, the policy silently denies it.

There are exactly two shapes, and it is a real security trade-off:

### Option A — wildcard the region

```json
{
  "Effect": "Allow",
  "Action": ["sqs:SendMessage", "sqs:ReceiveMessage"],
  "Resource": "arn:aws:sqs:*:123456789012:orders"
}
```

**Pro:** one statement, no growth, works in every current and future region without a policy change.
**Con:** it grants the permission in all 30-odd enabled Regions, including ones you do not use. Combined with a *missing* Region-deny SCP, an attacker with this role can create and use resources anywhere. Wildcard-region policies are only as safe as your Region-deny SCP — which makes the SCP load-bearing for security, not just for tidiness.

### Option B — explicit dual-region statements

```json
{
  "Effect": "Allow",
  "Action": ["sqs:SendMessage", "sqs:ReceiveMessage"],
  "Resource": [
    "arn:aws:sqs:eu-west-1:123456789012:orders",
    "arn:aws:sqs:eu-west-2:123456789012:orders"
  ]
}
```

**Pro:** least privilege is actually least privilege. An audit of the policy tells you exactly which regions it reaches.
**Con:** **every policy roughly doubles in size**, and IAM's [managed policy length limit is 6,144 characters and is *not adjustable*](https://docs.aws.amazon.com/general/latest/gr/iam-service.html). This is the quota that will actually stop you. A policy that sits comfortably at 3,500 characters today does not survive doubling. The `Resource` list is also the place where a typo silently produces a deny.

### Recommendation

**Option B (explicit dual-region ARNs), generated from a single list of regions in Terraform, with `aws:RequestedRegion` on the role as a backstop — and Option A permitted *only* where an explicit list would blow the 6,144 character limit, documented case by case.**

The reason is that the whole point of this programme is that the standby is a mirror of *one* primary. `eu-west-1` + `eu-west-2` is the entire allowed surface for the EU workload. A `*` says "any of 30 regions" for a workload whose correct answer is "two". And because the list is generated, the cost of Option B in a templated monorepo is near zero.

Where you do hit the character limit, the fix is usually to split into more, smaller managed policies (20 managed policies per role by default, adjustable) rather than to reach for wildcards.

---

## Exception 2: `aws:RequestedRegion` conditions that silently block the standby

A condition like this is extremely common in mature estates and it is exactly the kind of thing that was written when there was one region:

```json
"Condition": {
  "StringEquals": { "aws:RequestedRegion": "eu-west-1" }
}
```

At failover this does not produce a helpful error. It produces `AccessDenied` on an API the engineer knows the role is allowed to call, at 3am, under time pressure.

Two places it hides:

1. **On your own roles** — as a defensive condition someone added to a broad policy.
2. **On permissions boundaries** — a boundary with a region condition caps the *effective* permissions regardless of what the role's own policy says, and boundaries are much less frequently read than policies.

**Action: grep the entire Terraform repo for `RequestedRegion` before starting.** Every hit is a decision. The correct form for a pair is a list:

```json
"Condition": {
  "StringEquals": { "aws:RequestedRegion": ["eu-west-1", "eu-west-2"] }
}
```

Note the STS wrinkle: [requests to the STS *global* endpoint always report `aws:RequestedRegion` as `us-east-1`](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_region-endpoints.html), regardless of which Region actually served the request. A region condition that omits `us-east-1` will therefore block global-endpoint STS calls from any region. This is a real and confusing failure mode; it is one more reason to move everything to regional STS endpoints (below) and to `NotAction` `sts:*` in Region-deny SCPs.

---

## Exception 3: SCPs denying non-approved Regions — the first blocker

**Make this the first ticket in the whole programme, for every pair, not just CA.**

If the organisation uses AWS Control Tower's [Region deny control](https://docs.aws.amazon.com/controltower/latest/userguide/region-deny.html) or an equivalent hand-rolled SCP, then `eu-west-2`, `us-west-2` and `ca-west-1` are *denied for every principal in every account* until someone edits the policy at the Organization level. The symptom is an `AccessDenied` with no obvious cause; the engineer's own IAM policy is fine, and `iam:SimulatePrincipalPolicy` will not show it because SCPs are evaluated separately.

The canonical shape, with the global-service exemption that everyone forgets:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonApprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "iam:*", "sts:*", "organizations:*", "account:*",
        "route53:*", "route53domains:*", "cloudfront:*",
        "waf:*", "wafv2:*", "shield:*", "globalaccelerator:*",
        "support:*", "health:*", "trustedadvisor:*",
        "budgets:*", "ce:*", "cur:*", "tag:*",
        "a4b:*", "artifact:*", "chime:*",
        "kms:*", "acm:RequestCertificate", "acm:DescribeCertificate"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "eu-west-1", "eu-west-2",
            "us-east-1", "us-west-2",
            "ca-central-1", "ca-west-1"
          ]
        }
      }
    }
  ]
}
```

Why the `NotAction` list exists at all: global services have control-plane endpoints physically hosted in `us-east-1`, so a naive Region deny that does not exempt them breaks IAM, Route 53, CloudFront and Organizations across the entire organisation. `us-east-1` also has to stay in the allowed list for CloudFront/ACM reasons — see [[aws-acm]] and [[cloudfront]].

Four things to check when you edit it:

1. **`ca-west-1` must be enabled on the account before the SCP change is meaningful.** Two independent gates; both must open.
2. **Control Tower's managed Region deny is configured through Control Tower, not by editing the SCP directly.** Editing the generated SCP by hand drifts and gets reverted on the next landing zone update. Use `controltower:EnableControl` / the Control Tower console.
3. **`NotAction` with `Deny` is a blunt instrument** — it exempts those actions in *all* regions. That is the accepted trade-off; the alternative (enumerating allowed actions) is unmaintainable.
4. **Roll it out to a sandbox OU first.** A wrong Region-deny SCP applied at the root is one of the few changes that can take an entire organisation offline.

There is a related trap: **`aws:RequestedRegion` does not constrain global services**, because their requests are attributed to `us-east-1`. So a Region-deny SCP is a *data residency* control for regional services only, not a hard residency guarantee. For the CA pair specifically, where residency is likely the reason `ca-central-1` exists at all, that distinction needs to be in [[data-residency-compliance]] rather than assumed.

---

## Exception 4: role trust policies bound to an EKS OIDC provider

This is the one that actually forces work.

Every EKS cluster has its **own OIDC issuer URL**, fixed for the life of the cluster and unique per cluster. IRSA trust policies name it explicitly:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:sub": "system:serviceaccount:orders:orders-sa",
      "oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:aud": "sts.amazonaws.com"
    }
  }
}
```

The standby cluster has a different issuer URL. The IAM role is global and does not need duplicating — **the trust policy needs a second statement.** This is the one place where "mirroring IAM" means real edits, across potentially hundreds of roles.

```hcl
data "aws_iam_policy_document" "irsa_trust" {
  dynamic "statement" {
    for_each = var.oidc_providers # map of region_key => { arn, url }
    content {
      effect  = "Allow"
      actions = ["sts:AssumeRoleWithWebIdentity"]

      principals {
        type        = "Federated"
        identifiers = [statement.value.arn]
      }

      condition {
        test     = "StringEquals"
        variable = "${statement.value.url}:sub"
        values   = ["system:serviceaccount:${var.namespace}:${var.service_account}"]
      }

      condition {
        test     = "StringEquals"
        variable = "${statement.value.url}:aud"
        values   = ["sts.amazonaws.com"]
      }
    }
  }
}
```

⚠️ **Watch the trust policy length quota: 2,048 characters, default.** Two OIDC statements with full issuer URLs and two conditions each is roughly 900–1,100 characters. Three clusters (primary, standby, and a spare) will exceed it. The quota *is* adjustable (`L-C07B4B0D`) — raise it proactively in all accounts before you start, because hitting it mid-rollout is a Service Quotas ticket on the critical path.

The namespace and service-account name **must be identical in both clusters**, otherwise the `:sub` condition needs a per-cluster value and the policy grows again. That is a constraint on the EKS work, and it is easy to satisfy if stated up front — see [[aws-eks]].

**EKS Pod Identity is the escape hatch worth evaluating.** It replaces the OIDC federation model with a per-cluster association resource, so the IAM role's trust policy is a static `pods.eks.amazonaws.com` principal with no cluster-specific identifiers at all — which means **the trust policy becomes genuinely region-agnostic and the standby needs only a `aws_eks_pod_identity_association` in its own region.** For a two-region estate that is a materially simpler model. Cost of switching: it is an EKS-side migration per service account. Flag it for [[aws-eks]] to decide; it is the single biggest lever on IAM's multi-region workload.

---

## Exception 5: permissions boundaries

Boundaries are global objects like any other policy, so they replicate for free. The failure modes are the same as for ordinary policies but harder to spot, because a boundary denies by *omission*:

- A boundary listing `arn:aws:*:eu-west-1:...` resources caps everything the role can do in `eu-west-2` to nothing, even though the role's own policy is correct.
- A boundary with `aws:RequestedRegion` pinned to one region does the same.
- A boundary is frequently owned by a different team (platform/security) than the role, so the fix requires a cross-team change with its own lead time.

**Action:** enumerate every distinct permissions boundary in use (`aws iam list-policies --scope Local` and cross-reference `PermissionsBoundary` on roles), and review each one for region-bound ARNs and region conditions. This is a small, finite piece of work and it is much cheaper to do now than to discover during a failover test.

---

## IAM eventual consistency — everything must be pre-created

AWS's own words:

> "Any changes that you make in IAM (or other AWS services), including attribute-based access control (ABAC) tags, take time to become visible from all possible endpoints… **We recommend that you do not include such IAM changes in the critical, high availability code paths of your application.** Instead, make IAM changes in a separate initialization or setup routine that you run less frequently. Also, be sure to verify that the changes have been propagated before production workflows depend on them."
> — [Troubleshoot IAM: Changes that I make are not always immediately visible](https://docs.aws.amazon.com/IAM/latest/UserGuide/troubleshoot_general.html)

There is no published SLA on IAM propagation and there is no API that tells you a change is globally visible. Anecdotally it is seconds; occasionally it is much longer; the distribution has a tail and you will meet the tail during an incident, because that is when you are making changes.

**Stated plainly: the failover runbook must not create, modify or delete any IAM resource.** Not a role, not a policy, not a trust policy edit, not an OIDC provider, not an instance profile. Every one of them is pre-created and sitting idle. This costs nothing — IAM objects are free — so there is no cost argument against it, only an inventory-hygiene one.

The corollary is that **IAM should be the very first thing deployed for a standby, not the last.** Roles that nothing assumes yet are harmless. Roles that do not exist when you need them are an outage.

A short list of things teams typically leave to failover and must not:

| Commonly deferred | Why it fails at failover |
|---|---|
| Creating the standby cluster's OIDC provider | `aws_iam_openid_connect_provider` + trust policy edits, then propagation. Minutes at best. |
| Adding the standby region's ARNs to policies | A policy edit is an IAM change. Same problem. |
| Creating the replication role for S3/DynamoDB/Backup | Those services also need the role to be *assumable* at the moment they retry; a fresh role can fail the first attempts. |
| Creating instance profiles for the standby's node groups | Instance profile propagation to EC2 is its own, separately slow, consistency path. |
| Raising the trust-policy-length quota | Service Quotas request, hours to days. |

---

## STS: regional endpoints vs the global endpoint

### What actually changed

AWS has updated the behaviour of the global endpoint, and the current position is more nuanced than "it is all `us-east-1`":

> "Previously, all requests to the AWS STS global endpoint were served by a single AWS Region, US East (N. Virginia). Now in Regions enabled by default, requests to the AWS STS global endpoint are automatically served in the same Region where the request originates… **These changes will not be deployed to opt-in Regions.**"
> — [AWS STS Regions and endpoints › AWS STS global endpoint changes](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_region-endpoints.html)

And the routing table from that page:

| DNS resolver | Global-endpoint request served locally? |
|---|---|
| Amazon DNS resolver in a VPC in a **default** Region | **Yes** |
| Amazon DNS resolver in a VPC in an **opt-in** Region | **No — routed to `us-east-1`** |
| Any non-Amazon DNS resolver (ISP, public DNS, corporate) | **No — routed to `us-east-1`** |

### What this means for each pair

| Pair | Global-endpoint risk | Notes |
|---|---|---|
| EU (`eu-west-1` → `eu-west-2`) | Low | Both default Regions; global-endpoint calls from inside a VPC are served locally |
| **US (`us-east-1` → `us-west-2`)** | **The interesting one** | Both are default Regions, so `us-west-2` global-endpoint calls are served in `us-west-2`. **But the primary is `us-east-1` itself** — for the US pair, a `us-east-1` event is simultaneously a loss of the primary *and* of the historical global STS home. Anything in the estate still resolving `sts.amazonaws.com` through a non-Amazon resolver — CI runners, on-prem agents, containers with a custom `resolv.conf`, anything behind a corporate DNS forwarder — is still hitting `us-east-1`. Those are exactly the systems you need working in order to *run* the failover. |
| **CA (`ca-central-1` → `ca-west-1`)** | **The broken one** | `ca-west-1` is opt-in, so the local-serving change is explicitly **not** deployed there. Every global-endpoint STS call from `ca-west-1` goes to `us-east-1`. The Canadian standby has a hard `us-east-1` dependency unless you configure regional STS. For a region pair that probably exists for data-residency reasons, that is also a compliance conversation — see [[data-residency-compliance]]. |

### The second, sharper CA problem: session token version

> "Session tokens from the *global* AWS STS endpoint are valid only in AWS Regions that you enable, or that are enabled by default."
> — [Enable or disable AWS Regions in your account](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-regions.html)

Version 1 tokens (the default from the global endpoint) **do not work in opt-in Regions**. A credential minted from `sts.amazonaws.com` before `ca-west-1` was enabled, or minted by a system still using v1 tokens, will simply be rejected in `ca-west-1`. Two fixes, and you should do the first:

1. **Use regional STS endpoints everywhere.** Regional-endpoint session tokens are valid in all Regions, full stop. This is AWS's recommendation and it also removes the latency and blast-radius arguments.
2. Or run `aws iam set-security-token-service-preferences --global-endpoint-token-version v2Token` (needs `iam:SetSecurityTokenServicePreferences`). Note the warning: v2 tokens are **longer**, and "might affect systems where you temporarily store tokens" — anything putting a session token in a cookie, a header with a size cap, or a fixed-width database column.

### How to configure regional STS

Estate-wide, in this order of preference:

```bash
# 1. Environment, for every container, EC2 userdata, CI runner, Lambda
AWS_STS_REGIONAL_ENDPOINTS=regional
```

```ini
# 2. Shared config file, for humans and build agents
[default]
sts_regional_endpoints = regional
```

```hcl
# 3. Terraform provider — pin the endpoint explicitly where it matters
provider "aws" {
  alias  = "standby"
  region = var.standby_region
  # SDK v2-based provider honours AWS_STS_REGIONAL_ENDPOINTS; set it in CI.
  # For belt-and-braces on an opt-in region:
  endpoints {
    sts = "https://sts.${var.standby_region}.amazonaws.com"
  }
}
```

Most modern AWS SDKs already default to `regional`. The ones that do not are old SDK versions, `boto3` in some configurations, and anything using a hand-rolled signer. **Audit rather than assume**, and use the CloudTrail `endpointType` / `awsServingRegion` fields (added specifically for this) to find the stragglers:

```
# CloudTrail Lake / Athena
SELECT useridentity.arn, count(*)
FROM cloudtrail
WHERE eventsource = 'sts.amazonaws.com'
  AND json_extract_scalar(additionaleventdata, '$.endpointType') = 'global'
GROUP BY 1 ORDER BY 2 DESC
```

Finally: if you put an **STS interface VPC endpoint** in the standby VPC (recommended — see [[aws-vpc-networking]]), it only serves the *regional* endpoint name. Traffic to `sts.amazonaws.com` bypasses it entirely and goes out via NAT. That is another practical reason to move to regional STS: your endpoint is otherwise decorative.

---

## Service-linked roles: global, created on first use

**Precise answer: a service-linked role is an ordinary IAM role and is therefore a global, account-wide object. There is one per account per service, not one per region.** `AWSServiceRoleForAutoScaling` exists once in the account; using Auto Scaling in `eu-west-2` does not create a second one.

The "created on first use in a region" phrasing that circulates is about the **trigger**, not the scope. The sequence is:

1. You perform an action in a new region (create an ASG, create an ElastiCache cluster, enable GuardDuty).
2. The service checks whether its SLR exists **in the account**.
3. If it does not, the service calls `iam:CreateServiceLinkedRole` on your behalf. If it does, nothing happens.

So for a standby region, **if the primary already uses the service, the SLR already exists and there is nothing to do.** That is the common case and it is genuinely a no-op.

The three cases where it is not a no-op:

1. **A service used only in the standby.** Rare, but e.g. if only the standby uses a replication-specific feature, its SLR has never been created. Pre-create it: `aws iam create-service-linked-role --aws-service-name <service>.amazonaws.com`, or in Terraform.
2. **Your Terraform role lacks `iam:CreateServiceLinkedRole`.** The implicit creation is performed with *your* credentials' permissions. A tightly-scoped CI role that has never needed it in the primary (because the SLR was created years ago by a human in the console) will fail the first time it creates that resource type in a new region. This is a genuinely common and confusing first-apply failure.
3. **An SCP or permissions boundary denies `iam:CreateServiceLinkedRole`.** Same symptom, harder to find.

```hcl
# Pre-create SLRs the standby will need. Global resources: declare once, no alias.
resource "aws_iam_service_linked_role" "this" {
  for_each         = toset(var.service_linked_roles)
  aws_service_name = each.value
  # e.g. ["elasticache.amazonaws.com", "autoscaling.amazonaws.com",
  #       "elasticloadbalancing.amazonaws.com", "eks.amazonaws.com",
  #       "eks-nodegroup.amazonaws.com", "rds.amazonaws.com"]
}
```

⚠️ Importing pre-existing SLRs is fiddly (`terraform import aws_iam_service_linked_role.this["eks.amazonaws.com"] arn:aws:iam::123456789012:role/aws-service-role/eks.amazonaws.com/AWSServiceRoleForAmazonEKS`) and deleting them requires deleting the service's resources first. In an estate this old, most SLRs already exist; the lazy and correct move is to **grant `iam:CreateServiceLinkedRole` to the CI role and let AWS handle it**, rather than to bring dozens of pre-existing global roles under Terraform management for no benefit.

Note also: service-linked roles count toward the 1,000-roles-per-account quota, **but can exceed it** — only SLRs get that exemption.

---

## Terraform implementation

### The shape: IAM is declared once, regions come from a list

The clean pattern for "this policy needs ARNs for both regions of the pair" is a **`regions` variable and a `for` expression**, not a duplicated module.

```hcl
# ── variables.tf ─────────────────────────────────────────────────────────────
variable "regions" {
  description = "Both regions of this pair, primary first. Drives every ARN in every policy."
  type = object({
    primary = string
    standby = string
  })
}

variable "oidc_providers" {
  description = "EKS OIDC providers, keyed by region. Empty until the standby cluster exists."
  type = map(object({
    arn = string
    url = string # e.g. oidc.eks.eu-west-2.amazonaws.com/id/ABCD...
  }))
  default = {}
}
```

```hcl
# ── locals.tf ────────────────────────────────────────────────────────────────
data "aws_caller_identity" "current" {}
data "aws_partition" "current" {}

locals {
  account   = data.aws_caller_identity.current.account_id
  partition = data.aws_partition.current.partition
  # The single source of truth. Every ARN builder iterates this.
  region_list = [var.regions.primary, var.regions.standby]

  # Helper: build the same ARN in every region of the pair.
  arns = {
    for name, spec in var.resources :
    name => [
      for r in local.region_list :
      "arn:${local.partition}:${spec.service}:${r}:${local.account}:${spec.suffix}"
    ]
  }
}
```

```hcl
# ── policy.tf ────────────────────────────────────────────────────────────────
data "aws_iam_policy_document" "app" {
  statement {
    sid     = "Queues"
    effect  = "Allow"
    actions = ["sqs:SendMessage", "sqs:ReceiveMessage", "sqs:DeleteMessage",
               "sqs:GetQueueAttributes", "sqs:GetQueueUrl"]
    # -> both regions, generated, never hand-listed
    resources = flatten([for q in var.queue_names : [
      for r in local.region_list :
      "arn:${local.partition}:sqs:${r}:${local.account}:${q}"
    ]])
  }

  statement {
    sid       = "Secrets"
    effect    = "Allow"
    actions   = ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"]
    resources = flatten([for s in var.secret_names : [
      for r in local.region_list :
      "arn:${local.partition}:secretsmanager:${r}:${local.account}:secret:${s}-*"
    ]])
  }

  # Backstop: this role may only ever act in its own pair of regions.
  # Keeps least privilege even where a wildcard-region ARN was unavoidable,
  # and includes us-east-1 so STS global-endpoint calls are not blocked.
  statement {
    sid       = "DenyOutsidePair"
    effect    = "Deny"
    actions   = ["*"]
    resources = ["*"]
    condition {
      test     = "StringNotEquals"
      variable = "aws:RequestedRegion"
      values   = concat(local.region_list, ["us-east-1"])
    }
    # Global services are attributed to us-east-1; without that entry this
    # statement denies IAM, Route 53 and CloudFront for the role.
  }
}

resource "aws_iam_policy" "app" {
  name   = "${var.env}-${var.app}-app"
  policy = data.aws_iam_policy_document.app.json

  lifecycle {
    # 6,144 char limit is NOT adjustable. Fail the plan, not the apply.
    precondition {
      condition     = length(data.aws_iam_policy_document.app.json) < 6000
      error_message = "Policy is ${length(data.aws_iam_policy_document.app.json)} chars; IAM managed policy limit is 6144 and is not adjustable. Split it."
    }
  }
}
```

The `precondition` on policy length is the piece worth stealing. Dual-region ARNs double policy size, the limit is hard, and discovering it at apply time in a CI pipeline is annoying; discovering it at plan time is free.

### How this fits the cookiecutter monorepo

- **IAM gets its own root module per account, deployed under the primary provider only.** No `aws.standby` alias in it. If the repo currently instantiates IAM inside a per-region stack, that is the one structural change worth making — otherwise the second region's stack will fight the first over the same global role names. See [[terraform-repo-structure]].
- **`regions` becomes a cookiecutter variable at the pair level**, rendered into every environment. Adding a standby is then a one-line change to the pair definition, not an edit to every policy.
- **Keep the OIDC provider map optional and default-empty**, so the IAM stack can be deployed *before* the standby EKS cluster exists (which it must be — see the ordering above) and updated in place once the cluster ID is known.
- **Use `aws_iam_policy_document` everywhere, never heredoc JSON.** Heredocs cannot iterate a region list, which is the entire point.

---

## Migration path from single-region

Nothing here changes behaviour in the primary, which makes this one of the safest workstreams in the vault. Order matters though:

1. **Enable `ca-west-1`** on every account that needs it (CA pair only). Asynchronous; start it first, it has the longest lead time.
2. **Amend the Region-deny SCP** to allowlist `eu-west-2`, `us-west-2`, `ca-west-1`. Sandbox OU first, then production OUs.
3. **Raise the role-trust-policy-length quota** (`L-C07B4B0D`) in every account, to say 4,096. Free, and takes it off the critical path.
4. **Audit and fix**, in this order — each is a grep and a review, no downtime:
   - `grep -rn "RequestedRegion" .` — every region condition.
   - `grep -rnE "arn:aws[a-z-]*:[a-z0-9-]+:(eu|us|ca)-[a-z]+-[0-9]" .` — every hardcoded regional ARN in a policy.
   - Every permissions boundary in use.
   - Every `sts.amazonaws.com` reference and every SDK/agent defaulting to the global endpoint.
5. **Refactor policies to generate ARNs from `local.region_list`.** Applying this to the primary is a **policy-content change, not a replacement** — `aws_iam_policy.policy` is an in-place update and creates a new policy *version*, not a new policy. Zero downtime. ⚠️ The exception is `name`/`name_prefix`/`path`, which are `ForceNew`: do not rename policies during this refactor.
6. **Set `AWS_STS_REGIONAL_ENDPOINTS=regional`** in CI, container base images and EC2 userdata. Verify via CloudTrail `endpointType`.
7. **Pre-create anything the standby will need** — SLRs, replication roles, the standby's OIDC provider once the cluster exists.

Replacement risks, flagged:
- `aws_iam_role.name`, `aws_iam_policy.name`, `path` on either — **ForceNew.** Renaming a role that things assume is an outage.
- `aws_iam_openid_connect_provider.url` — **ForceNew.** Fine; the standby's is new anyway.
- `aws_iam_role.permissions_boundary` is in-place, but changing it can instantly cap live permissions. Treat as a production change.
- Adding a statement to a trust policy is in-place and safe. Removing one is not — it revokes access immediately.

---

## Failover procedure

**The IAM layer does nothing at failover. That is the design and it should be defended.**

The only IAM-adjacent steps in a failover runbook should be *verifications*, run as part of the standing readiness check rather than during the incident:

1. `aws sts get-caller-identity --region <standby>` from the standby's compute — proves credentials resolve and the Region is enabled.
2. A canary pod in the standby cluster performing one real API call per service via IRSA/Pod Identity — proves the trust policy, the OIDC provider, the policy ARNs and the SCP are all correct *together*. This is the only test that catches all five exception classes at once. **Run it on a schedule, not at failover.**
3. `aws iam get-role --role-name <each role>` reachable — trivially true if IAM is up, but it is the check that catches an SCP change someone made last Tuesday.

**Human decision:** none. If anyone is editing an IAM policy during a failover, the pre-work was incomplete and the RTO is already blown.

## Failback

Symmetrical and boring, which is the correct outcome. Two things to watch:

- **Do not "clean up" the standby's IAM after failback.** Removing the standby OIDC provider or trimming the standby's ARNs out of policies re-creates the exact gap you just spent a quarter closing. Both regions' entries are permanent.
- **Anything created *during* the incident** — an emergency break-glass role, a widened policy, a temporarily attached `AdministratorAccess` — must be reverted deliberately and logged. This is the most common real-world source of post-incident privilege creep. Put it in the failback checklist explicitly and diff the IAM state against Terraform afterwards.

---

## Gotchas

1. **`ca-west-1` is opt-in.** IAM credentials do not exist there until the Region is enabled; enabling takes minutes to hours, is asynchronous, is rate-limited to 6 in-flight per account and 50 per Organization, and has no Terraform resource.
2. **`ENABLED` does not mean usable.** AWS explicitly warns some services may not be immediately usable after the status flips.
3. **Region-deny SCPs block the standby before you write a line of Terraform**, and the error is a bare `AccessDenied` that policy simulation will not explain.
4. **Managed policy length is 6,144 characters and is NOT adjustable.** Dual-region ARNs roughly double policy size. This is the quota that stops the "explicit ARNs" approach.
5. **Role trust policy length is 2,048 characters by default.** Two OIDC statements nearly fill it. Raise it before you need it.
6. **STS global-endpoint requests always report `aws:RequestedRegion` as `us-east-1`** — a region condition omitting `us-east-1` blocks them from everywhere.
7. **The STS global-endpoint local-serving improvement is not deployed to opt-in Regions** — `ca-west-1` global STS calls still go to `us-east-1`.
8. **v1 session tokens from the global endpoint do not work in opt-in Regions.** Silent, confusing, and fixed either by regional STS or `set-security-token-service-preferences`.
9. **An STS interface VPC endpoint does not intercept `sts.amazonaws.com`** — only the regional name. Global-endpoint traffic leaves via NAT.
10. **IAM is eventually consistent with no propagation SLA.** Nothing IAM-shaped belongs in a failover runbook.
11. **`iam:CreateServiceLinkedRole` is called with *your* credentials.** A CI role that never needed it in the primary will fail on first use of a service type in a new region.
12. **Permissions boundaries deny by omission**, are often owned by another team, and are the least-read policy in any estate.
13. **Duplicating IAM behind a provider alias produces two roles, not a mirror.** Declare IAM once.
14. **Hardcoding `arn:aws:`** breaks in other partitions. Use `data.aws_partition.current.partition` — costs nothing, and it is the kind of thing that only gets fixed once.
15. **Access Analyzer is regional** even though IAM is not — an analyzer in `eu-west-1` does not analyse anything about `eu-west-2` resource policies. Create one per region. Easy to miss precisely because "IAM is global".

---

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Regional ARNs in policies | Wildcard region (`arn:aws:sqs:*:...`) | Explicit list of both regions, generated | **B**, generated from `local.region_list`, with a `precondition` on policy length. Fall back to A only where the 6,144 limit forces it, and document each case |
| `aws:RequestedRegion` on roles | Omit it entirely | Deny outside the pair (+ `us-east-1`) | **B.** It is the backstop that makes any residual wildcard safe |
| Region-deny SCP | Hand-rolled SCP | Control Tower Region deny control | **Whichever you already run** — but do not hand-edit a Control Tower-managed SCP, it drifts and reverts |
| IRSA trust policies | Multi-statement trust policy per role, one per cluster | Migrate to EKS Pod Identity | **Evaluate B seriously in [[aws-eks]].** Pod Identity removes cluster-specific identifiers from IAM entirely, which is the single biggest simplification available. If that migration is too large, A works and is well understood |
| STS endpoints | Leave as-is | `AWS_STS_REGIONAL_ENDPOINTS=regional` estate-wide | **B, unconditionally.** It is a free configuration change that removes a `us-east-1` dependency and is required for `ca-west-1` to work reliably |
| Global-endpoint token version | Leave at v1 | Set v2 | **Do both**: set v2 as a safety net, but fix the root cause by moving to regional endpoints. Test v2 token length against anything that stores tokens first |
| Service-linked roles | Bring all under Terraform | Grant `iam:CreateServiceLinkedRole` to CI and let AWS create them | **B.** Importing dozens of pre-existing global roles buys nothing |
| Where IAM lives in the repo | Per-region stack | One global stack per account | **B.** IAM has no region; a per-region stack will collide on names |

---

## Cost

**Zero.** IAM roles, policies, users, groups, instance profiles, OIDC providers and service-linked roles are free. Enabling an opt-in Region is free ("There is no charge to enable or disable a Region"). There is no idle cost, no per-request cost, and no cost difference between one region and six.

This is worth stating loudly in the cost model, because it changes the sequencing argument: **there is no financial reason to defer any IAM work.** Pre-creating every role the standby will ever need, years before the standby is used, costs nothing. Contrast with [[aws-vpc-networking]], where pre-provisioning is the main cost driver. IAM should therefore be the *most* eagerly pre-provisioned layer in the programme.

The only indirect cost is quota-shaped: 1,000 roles and 1,500 customer managed policies per account by default, both adjustable. A dual-region estate does not increase role count (roles are global) but the ARN-doubling refactor can increase *policy* count if you split policies to stay under 6,144 characters. Watch the managed-policies-per-role limit (20, adjustable) if you do.

---

## Open questions

1. **Does the organisation run a Region-deny SCP, and is it Control Tower-managed or hand-rolled?** Determines who makes the change and how long it takes.
2. **Is `ca-west-1` already enabled in any account?** If not, who holds `account:EnableRegion` in the management account, and how many accounts need it?
3. **How many AWS accounts are there, and what is the per-region/per-environment topology?** Drives the region-enablement campaign and whether IAM lives in a shared account.
4. **Are there permissions boundaries in use, and who owns them?**
5. **How many IRSA roles exist today?** This is the sizing input for the biggest piece of IAM work, and the input to the Pod Identity vs IRSA decision.
6. **Is anything still using the STS global endpoint?** Answerable from CloudTrail today using the `endpointType` field — worth running before the design is finalised.
7. **Does anything store STS session tokens in a size-constrained place** (cookie, header, fixed-width column)? Blocks the v2 token change.
8. **Are there policies already near 6,144 characters?** `aws iam list-policies --scope Local` plus a length check gives the list in minutes and tells you how big the ARN-doubling problem really is.

---

## Sources

- [AWS Identity and Access Management endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/iam-service.html) — proves IAM is served by a single `iam.amazonaws.com` endpoint in every Region; source of the 6,144-character managed policy limit (not adjustable), the 2,048-character trust policy limit (adjustable), 1,000 roles and 20 managed policies per role.
- [Enable or disable AWS Regions in your account](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-regions.html) — the authoritative opt-in vs default Region list confirming `ca-west-1` is opt-in and the other five are default; IAM data propagation on enablement; "a few minutes to several hours"; the 6-per-account / 50-per-Organization request limits; partitions as independent IAM instances; the `account:EnableRegion` API and permissions.
- [Manage AWS STS in an AWS Region](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_enable-regions.html) — why AWS recommends regional STS; session token validity; `set-security-token-service-preferences` and token versions.
- [AWS STS Regions and endpoints › AWS STS global endpoint changes](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_region-endpoints.html) — the per-Region STS activation table, the DNS-resolver routing table, the statement that the local-serving change is not deployed to opt-in Regions, and that global-endpoint requests report `aws:RequestedRegion` as `us-east-1`. The core source for the US and CA pair analysis.
- [Troubleshoot IAM — Changes that I make are not always immediately visible](https://docs.aws.amazon.com/IAM/latest/UserGuide/troubleshoot_general.html) — AWS's own eventual-consistency guidance and the explicit "do not include IAM changes in critical, high availability code paths".
- [Create a service-linked role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create-service-linked-role.html) — SLRs are ordinary account-wide IAM roles created by the service on first use; the `iam:CreateServiceLinkedRole` permission shapes; the fact that SLRs may exceed the role quota.
- [Configure the Region deny control — AWS Control Tower](https://docs.aws.amazon.com/controltower/latest/userguide/region-deny.html) — the managed Region-deny control, and why it must be changed through Control Tower rather than by editing the generated SCP.
- [Deny access to AWS based on the requested AWS Region — AWS Control Tower](https://docs.aws.amazon.com/controltower/latest/controlreference/primary-region-deny-policy.html) — the reference policy including the global-service `NotAction` exemption list.
- [Restrict data transfers across AWS Regions — AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/privacy-reference-architecture/restrict-data-transfers-across-regions.html) — the residency framing of Region restriction and the limits of `aws:RequestedRegion` against global services.
- [AWS STS Regionalized endpoints — AWS SDKs and Tools Reference](https://docs.aws.amazon.com/sdkref/latest/guide/feature-sts-regionalized-endpoints.html) — the `AWS_STS_REGIONAL_ENDPOINTS` environment variable and `sts_regional_endpoints` config setting.
- [Troubleshoot an OIDC provider and IRSA in Amazon EKS](https://repost.aws/knowledge-center/eks-troubleshoot-oidc-and-irsa) — per-cluster OIDC issuer URLs and the trust policy conditions, the basis for the multi-statement trust policy pattern.
- [IRSA vs Pod Identity: Rethinking Workload Identity in Amazon EKS](https://builder.aws.com/content/3B76CLg0IEmLTN9gdnThnVmykm5/irsa-vs-pod-identity-rethinking-workload-identity-in-amazon-eks) — the comparison behind the Pod Identity recommendation. Community content on an AWS property, not official documentation; treat the conclusions as a prompt to test rather than as a guarantee.

## Related

[[aws-vpc-networking]] · [[cross-region-connectivity]] · [[aws-eks]] · [[aws-kms]] · [[aws-acm]] · [[cloudfront]] · [[data-residency-compliance]] · [[terraform-repo-structure]] · [[failover-runbook]] · [[quotas-and-limits]] · [[cost-modelling]] · [[region-pair-selection]]
