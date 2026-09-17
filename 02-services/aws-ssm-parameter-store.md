---
title: AWS SSM Parameter Store — Multi-Region
service: ssm-parameter-store
tags: [service, multi-region, ssm, parameter-store, configuration]
status: researched
replication: none — no native cross-region replication exists
rpo_achievable: "seconds (event-driven Lambda) / deploy cadence (Terraform or pipeline)"
rto_achievable: "< 1 min if parameters are pre-populated in the standby; fails RTO badly if not"
meets_targets: conditional
updated: 2026-09-16
---

# AWS SSM Parameter Store — Multi-Region

## TL;DR

- **Parameter Store has no cross-region replication. None. At all.** Unlike
  [[aws-secrets-manager]], which the team has already done natively, there is no
  `replica` block, no API, no console button. This is a real gap and it is the
  single most important fact in this note.
- **Four viable options**, covered in detail below: (a) Terraform writes to both
  regions, (b) an EventBridge-triggered replication Lambda, (c) migrate the
  secrets to Secrets Manager and keep only non-secret config here, (d)
  config-as-code where the deploy pipeline writes both regions.
  **Recommendation: (a) for everything Terraform already owns, plus (d) for
  anything the pipeline owns. Do not build (b) unless you find parameters that
  are genuinely written at runtime by the application.**
- **The landmine nobody plans for: parameters whose *values* are region-specific.**
  A queue URL, a bucket name, an RDS endpoint, a KMS key ARN, an ECR URI — all
  of these are commonly stored as parameters, and every one of them is wrong in
  the standby if you replicate the value verbatim. **Blind replication is worse
  than no replication**, because it produces a standby that comes up and quietly
  talks to the dead region. Classify every parameter before you replicate
  anything.
- **`GetParameter`/`GetParameters`/`GetParametersByPath` share a default quota of
  40 TPS per region.** Forty. That is trivially exceeded by a fleet cold-starting
  in the standby at failover. `PutParameter` is **3 TPS** by default, which
  breaks bulk bootstrap and any replication Lambda under load. Both are
  raisable, at a cost. Raise them before failover day.
- **RPO/RTO verdict: conditional.** Trivially meets both *if* parameters are
  pre-populated. If you plan to copy them at failover time, `PutParameter` at
  3 TPS means 10,000 parameters takes about an hour — you fail RTO by a factor
  of four.

## Does this service cross regions at all?

No. Parameter Store is regional in every sense:

- A parameter lives in one region. `arn:aws:ssm:eu-west-1:111122223333:parameter/app/db/host`.
- There is no global namespace, no `aws:ssm` global endpoint, no replica
  concept. `ReplicateSecretToRegions` has no Parameter Store equivalent.
- `/aws/reference/secretsmanager/<secret-id>` lets Parameter Store *read* a
  Secrets Manager secret — but only one in the same region, so it doesn't help.
- Parameters **can** be shared across *accounts* (advanced tier only) but not
  across regions.
- `ca-west-1` has full SSM endpoints (`ssm.ca-west-1.amazonaws.com`, plus FIPS),
  so the CA pair is not blocked on service availability here. Parameter Store is
  not one of `ca-west-1`'s gaps — see [[region-pair-selection]].

There is one genuine cross-region read: the AWS-managed public parameters under
`/aws/service/...` (e.g. the EKS optimised AMI IDs, the ECS AMI IDs). Those are
published by AWS in every region and resolve locally, so they work in the
standby automatically. Note that **the AMI ID under the same parameter path is
different in each region** — which is correct behaviour and a good illustration
of the "region-specific value" theme below.

## Replication / mirroring options

### The classification step — do this first

Before choosing a mechanism, split your parameters into three buckets. This is
not optional busywork; it determines what each mechanism has to do.

| Class | Example | What the standby needs |
|---|---|---|
| **Region-neutral config** | `/prod/app/log-level` = `info`, `/prod/app/feature-flags/x` = `true`, `/prod/app/timeout-ms` = `5000` | **Identical value.** Straight copy. |
| **Region-specific config** | `/prod/app/sqs-url`, `/prod/app/rds-endpoint`, `/prod/app/kms-key-arn`, `/prod/app/ecr-uri`, `/prod/app/bucket-name` | **A different value, computed for the standby.** A copy is actively harmful. |
| **Secrets (SecureString)** | `/prod/app/db-password`, `/prod/app/third-party-api-key` | Same value, but re-encrypted under the **standby region's** KMS key. Also: strongly consider whether these should be in [[aws-secrets-manager]] instead. |

**A blanket "replicate everything under `/prod/`" rule will silently produce a
standby that is pointed at the primary region.** At 3am, with traffic already
shifted, this manifests as a fleet that is healthy, serving, and writing to a
queue in the region that just died. That is worse than an outright failure,
because it looks fine on the dashboard.

The practical test: **grep your parameter values for region strings and account
ARNs.**

```bash
aws ssm get-parameters-by-path --path /prod --recursive --with-decryption \
  --region eu-west-1 --query 'Parameters[].[Name,Value]' --output text \
  | grep -E 'eu-west-1|arn:aws|\.amazonaws\.com|vpc-|subnet-|sg-'
```

Every line that comes back is a parameter that needs a per-region value, not a
copy. Run this before you write any replication code.

### Option (a) — Terraform writes to both regions from one module

```hcl
resource "aws_ssm_parameter" "primary" { ... }
resource "aws_ssm_parameter" "standby" { provider = aws.standby, ... }
```

**Pros.** Zero new infrastructure. One source of truth. Drift detection for
free — `terraform plan` tells you the two regions diverged. Handles the
region-specific case naturally, because the value is an expression and can
reference standby-region resources. Fits the existing cookiecutter monorepo,
which is explicitly the thing you're supposed to evolve rather than replace.

**Cons.** The well-known one: **`SecureString` values land in Terraform state
in plaintext.** HashiCorp is explicit about this — state is not an encryption
boundary and any value passed to a resource is stored in it. If your state
backend is an S3 bucket with SSE-KMS, bucket-policy-restricted access and
versioning, that is a defensible position and most organisations accept it. If
your state is readable by anyone who can run `terraform plan`, it is not.

**Mitigation that keeps the benefits:** let Terraform manage the parameter
*resource* (name, type, tier, KMS key, tags, IAM) but not the *value*.

```hcl
resource "aws_ssm_parameter" "app_secret" {
  name   = "/${var.env}/app/db-password"
  type   = "SecureString"
  key_id = var.kms_key_arn        # region-appropriate key
  value  = "PLACEHOLDER_SET_OUT_OF_BAND"

  lifecycle {
    ignore_changes = [value]      # the classic pattern
  }
}
```

`ignore_changes = [value]` means Terraform creates the parameter once with a
dummy value, then never touches it again. A human or a pipeline sets the real
value out of band. **The trap:** `ignore_changes` is *not* `prevent_destroy`.
If anything forces replacement — the parameter name changing, the module being
moved without a `moved` block — Terraform destroys and recreates it, and the
recreated parameter contains `PLACEHOLDER_SET_OUT_OF_BAND`. Pair the two:

```hcl
  lifecycle {
    ignore_changes  = [value]
    prevent_destroy = true
  }
```

Also note `ignore_changes` means the standby's value will *not* track the
primary's. You have created two parameters that Terraform believes are in sync
and are not. For secrets, that's the right trade; just be honest that you have
moved the sync problem to whatever sets the value.

### Option (b) — EventBridge-triggered replication Lambda

Parameter Store emits **native EventBridge events**, no CloudTrail required:

```json
{
  "source": ["aws.ssm"],
  "detail-type": ["Parameter Store Change"],
  "detail": {
    "name": [{ "prefix": "/prod/" }],
    "operation": ["Create", "Update", "Delete", "LabelParameterVersion"]
  }
}
```

A Lambda on that rule calls `GetParameter --with-decryption` in the primary and
`PutParameter --overwrite` in the standby.

**Pros.** Near-real-time (RPO measured in seconds, far inside the 2h target).
Catches parameters written by the *application* at runtime, which Terraform by
definition cannot.

**Cons — and there are more than people expect:**

1. **AWS states the events are best-effort.** From the docs: *"Events are
   emitted on a best effort basis."* That is not a durability guarantee. A
   dropped event means a silently stale parameter in the standby, which you
   discover during a failover. Any serious implementation therefore needs a
   **periodic full reconciliation sweep** as well as the event path — at which
   point you have built two systems.
2. **The event does not contain the value.** It carries the name, operation,
   type and description. The Lambda must call back to `GetParameter`, which
   means the Lambda holds plaintext secrets in memory and needs `kms:Decrypt`
   on the primary key **and** `kms:Encrypt`/`GenerateDataKey` on the standby
   key. That is a cross-region, high-privilege blast radius in one function. Do
   not point it at `/` — scope the EventBridge rule and the IAM policy to
   specific path prefixes.
3. **`PutParameter` is 3 TPS by default.** A bulk change (a pipeline writing
   200 parameters) produces a burst the Lambda cannot drain without throttling
   and retries. You will need the higher-throughput setting (10 TPS) and a
   DLQ, and even 10 TPS is not much.
4. **It blindly copies region-specific values**, unless you build value-rewriting
   logic — which is a transformation engine, in a Lambda, in the failover path.
5. **It runs in the primary region.** If the primary is degraded, so is the
   replicator. It cannot help you *during* an incident, only before one.
6. Deletes need handling, or the standby accumulates parameters that no longer
   exist in the primary.

There is prior art — [CustomInk's write-up](https://technology.customink.com/blog/2022/01/10/aws-systems-manager-ssm-cross-region-replication/)
and [alessandrobologna/parameter-store-replicator](https://github.com/alessandrobologna/parameter-store-replicator)
are both real, public implementations — so this is a well-trodden path, not an
exotic one. But it is a piece of bespoke infrastructure in the critical path of
your DR story, and it is infrastructure that only fails in ways you discover
during a disaster.

### Option (c) — Migrate to Secrets Manager

Secrets Manager replicates natively, the team has already done the work, and
the pattern is proven in this estate. See [[aws-secrets-manager]].

**Pros.** Deletes the problem rather than solving it. Native replication with a
supported SLA. Rotation support you don't get in Parameter Store. You are
already operating it.

**Cons.** **Cost.** Secrets Manager is **$0.40 per secret per month** plus
**$0.05 per 10,000 API calls**, and *each replica counts as a separate secret
for billing*. Parameter Store standard parameters are **free**, with API calls
at **$0.05 per 10,000 interactions**. If you have 2,000 parameters, moving all
of them is roughly $800/month at the primary plus another $800 for the
replicas. For 2,000 *secrets* that might be justified; for 2,000 *config
values* it is absurd.

**The nuance that makes this the right answer for part of the estate:** you
almost certainly do not have 2,000 secrets. You have a few dozen secrets and a
lot of config. Split them.

### Option (d) — Config-as-code; the deploy pipeline writes both regions

Parameter values live in version control (SOPS/`git-crypt`-encrypted, or
sourced from Secrets Manager at deploy time), and the deploy pipeline calls
`PutParameter` against every region in the pair.

**Pros.** Git is the source of truth and the audit log. Trivially handles
region-specific values — the pipeline templates them per region. No plaintext
in Terraform state. Works for parameters Terraform doesn't own.

**Cons.** A parameter changed by hand in the console drifts until the next
deploy. Ordering: the pipeline must write the standby even when it isn't
deploying application code there (your standby is warm, not cold, so this is
fine but must be explicit). Needs a bootstrap path for a brand-new region.

### Recommendation

**(a) + (d), with (c) applied to the genuinely-secret subset. Do not build (b)
unless you can name specific parameters the application writes at runtime.**

Rationale:

- Terraform already owns most of this and already has the standby provider
  aliased for [[aws-kms]] and [[aws-secrets-manager]]. Adding a second
  `aws_ssm_parameter` costs one resource block and gives you drift detection,
  which none of the other options do.
- Region-specific values are the actual hard problem, and (a) and (d) solve
  them by construction — the value is an expression or a template, evaluated
  per region — whereas (b) has to be taught about them.
- Your RPO is **2 hours**, not 2 seconds. Config does not change hourly. The
  near-real-time property that (b) buys is worth almost nothing here, and it is
  bought with bespoke, disaster-path-only infrastructure.
- (c) is right where the data is a real secret, wrong where it is a log level.

**If** the inventory turns up parameters written at runtime by the application
(feature flags toggled by an admin UI, values written by a Lambda), those
specific paths need (b) or need to move somewhere that replicates. Find out
before deciding. That is the top open question below.

## Standard vs Advanced tier

Relevant because the tier is per parameter per region, and **it is a one-way
door**.

| | Standard | Advanced |
|---|---|---|
| Max parameters (per account per region) | **10,000** | **100,000** |
| Max value size | **4 KB** | **8 KB** |
| Parameter policies (expiry, expiry notification, no-change notification) | Not supported | Supported, max 10 per parameter |
| Cross-**account** sharing | Not supported | Supported |
| Cost | **No additional charge** | **$0.05 per advanced parameter per month**, prorated hourly |
| Tier change | Upgradeable to Advanced | **Not downgradeable** |

AWS is explicit about the one-way part: *"You can change a standard parameter
to an advanced parameter at any time. You can't change an advanced parameter to
a standard parameter"* — because it would truncate 8 KB to 4 KB, drop policies,
and *"Advanced and standard parameters use a different form of encryption."* To
go back you must delete and recreate.

**Implications for the standby:**

- The **default tier is a per-account, per-region service setting**
  (`/ssm/parameter-store/default-parameter-tier`, values `Standard` /
  `Advanced` / `Intelligent-Tiering`). A brand-new standby region defaults to
  `Standard`. **If your primary is set to `Advanced` or `Intelligent-Tiering`
  and the standby isn't, replicating a >4 KB parameter into the standby fails.**
  Check and set this as part of standing the region up — it is a two-line
  service setting that nobody thinks about.
- Always set `tier` explicitly on `aws_ssm_parameter` rather than relying on the
  regional default. Relying on an account-and-region-scoped service setting is
  exactly the kind of implicit dependency that behaves differently in a region
  you rarely deploy to.
- Advanced parameters cost **$0.05/month each, in each region**. Mirroring 1,000
  advanced parameters is **$50/month extra**. Mirroring 1,000 standard
  parameters is free. Another reason to keep things standard-tier.
- **Parameter policies do not help you here.** Expiration/notification policies
  fire in the region they're defined in, and there is no cross-region
  interaction.

## Throughput limits — the failover stampede

This is the operational risk and it is much tighter than people assume.

| API | Default TPS | With higher throughput enabled |
|---|---|---|
| `GetParameter` + `GetParameters` + `GetParametersByPath` | **40, shared across all three** | `GetParameter` **10,000**, `GetParameters` **1,000**, `GetParametersByPath` **100** |
| `PutParameter` (shared with `DescribeParameters`, `GetParameterHistory`, `LabelParameterVersion`, `UnlabelParameterVersion`) | **3** | **10** |
| `DeleteParameter`, `DeleteParameters` | **3** | **5** |

Three things follow:

1. **40 TPS shared is nothing.** If every pod calls `GetParametersByPath` on
   boot, 40 pods starting in the same second exhausts it. In a normal rolling
   deploy that's fine because starts are spread out. **At failover, when the
   standby scales from 2 replicas to 200 in one go, it is not fine.** You get
   `ThrottlingException`, the SDK retries with backoff, pod startup stretches,
   and your 15-minute RTO evaporates in exponential backoff.
2. **`PutParameter` at 3 TPS makes "copy the parameters at failover time" a
   non-strategy.** 10,000 parameters ÷ 3/s ≈ 55 minutes, before retries. Even
   at the raised 10 TPS it's ~17 minutes. **Parameters must be pre-populated.**
   This is the concrete reason the "prerequisites first" strategy is correct for
   Parameter Store specifically.
3. **SecureString throughput is additionally bounded by KMS.** The AWS quota
   table carries this footnote verbatim: *"Throughput for SecureString
   parameters might be further limited by AWS Key Management Service (AWS KMS)
   throughput limits depending on the Region."* Given `eu-west-2` has a 20,000/s
   symmetric KMS quota against `eu-west-1`'s 100,000/s (see [[aws-kms]]), the EU
   standby is the constrained one on both axes at once.

**Mitigations:**

- **Enable higher throughput in the standby regions now.** Note it is billed
  the same $0.05/10,000 interactions either way, per the pricing page — the
  setting raises the ceiling, and AWS documents that *"Increasing the TPS quota
  incurs a charge on your AWS account"*, so confirm the current billing detail
  before assuming it is free.
- **Fetch parameters once per pod, not per request.** A sidecar or init
  container that materialises the whole path into a file or env vars turns N
  requests into one `GetParametersByPath` page set. If you're on EKS, the
  [AWS Secrets and Configuration Provider (ASCP) for the Kubernetes Secrets
  Store CSI Driver](https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html)
  does this for both Parameter Store and Secrets Manager and has a caching TTL.
  See [[amazon-eks]].
- **Batch.** `GetParameters` takes up to 10 names per call; `GetParametersByPath`
  pages. Ten parameters fetched individually is ten interactions against the
  quota *and* ten billed interactions. Batching cuts both.
- Note the billing definition: *"a Parameter Store API interaction is defined as
  an interaction between an API request and an individual parameter"* — so
  batching reduces round trips but **not** the billed interaction count.

## RPO / RTO analysis

| | Verdict |
|---|---|
| **RPO 2h** | **Easily met by any option.** Config changes at deploy cadence, not continuously. Terraform (a) and pipeline (d) give RPO = "time since last deploy", typically hours to days, which sounds bad but is fine because nothing *changed* in between. The event-driven Lambda (b) gives seconds. All comfortably inside 2h. |
| **RTO 15m** | **Met only if parameters are pre-populated.** Reading is fast; writing is not. The whole risk is the 40 TPS read quota during the cold-start stampede, plus the 3 TPS write quota if anyone imagines copying at failover time. |

**Time budget at failover, assuming pre-populated:**

| Step | Time |
|---|---|
| Parameters already exist in standby | 0s |
| Standby KMS key already decrypts SecureStrings | 0s (see [[aws-kms]]) |
| Pods read config on boot | seconds — **unless throttled**, then minutes |
| Any parameter that needs its value changed at failover | **this is the part to eliminate entirely** |

That last row is the design goal: **no parameter should need to change value at
failover.** If a parameter says "which region is active", you have put a
failover decision inside a config store with a 3 TPS write limit. Put that
decision in Route 53 health checks or a feature flag service instead — see
[[failover-orchestration]] and [[route-53]].

## Warm standby shape

| Thing | State while idle | Cost |
|---|---|---|
| Standard parameters, fully populated | Present, current | **Free** |
| Advanced parameters | Present, current | **$0.05/each/month** per region |
| SecureStrings | Present, encrypted under the standby's CMK | Free (parameter) + KMS key cost |
| Higher-throughput setting | Enabled | See pricing page; API interactions billed at $0.05/10,000 |
| Replication Lambda (option b only) | Idle, ~0 invocations | Pennies |

**Standard-tier Parameter Store in a standby region is essentially free.** There
is no cost argument against pre-populating it, which removes the only reason
anyone would defer it. Do it early.

## Terraform implementation

### The module shape for a cookiecutter monorepo

The key design decision: **take a map of parameters and a per-region override
map**, so region-specific values are expressed rather than copied.

```hcl
# modules/ssm-parameters/variables.tf

variable "env" { type = string }

variable "primary_region" { type = string }
variable "standby_region" { type = string }

variable "parameters" {
  description = <<-EOT
    Region-neutral parameters. Written identically to both regions.
    Do NOT put anything region-specific in here.
  EOT
  type = map(object({
    value       = string
    type        = optional(string, "String")   # String | StringList | SecureString
    tier        = optional(string, "Standard")
    description = optional(string)
  }))
  default = {}
}

variable "regional_parameters" {
  description = <<-EOT
    Parameters whose value differs per region.
    Key = parameter name suffix, value = map of region -> value.
    Every parameter here MUST have an entry for every region in the pair;
    the validation below enforces it, which is the whole point of this module.
  EOT
  type = map(object({
    values      = map(string)   # region => value
    type        = optional(string, "String")
    tier        = optional(string, "Standard")
    description = optional(string)
  }))
  default = {}
}

variable "kms_key_arn_by_region" {
  description = "From module.kms_pair.key_arn_by_region. SecureStrings use the key for their own region."
  type        = map(string)
}

variable "standby_enabled" {
  type    = bool
  default = true
}

variable "path_prefix" {
  description = "e.g. /prod/app. Parameter names are <path_prefix>/<map key>."
  type        = string
}
```

```hcl
# modules/ssm-parameters/main.tf

locals {
  regions = compact([var.primary_region, var.standby_enabled ? var.standby_region : ""])

  # Fail the plan, loudly, if a regional parameter is missing a value for a
  # region in the pair. This is the guardrail that stops a half-configured
  # standby reaching production.
  missing = flatten([
    for k, p in var.regional_parameters : [
      for r in local.regions : "${k}@${r}" if !contains(keys(p.values), r)
    ]
  ])
}

resource "terraform_data" "validate_regional_parameters" {
  lifecycle {
    precondition {
      condition     = length(local.missing) == 0
      error_message = "regional_parameters missing values for: ${join(", ", local.missing)}"
    }
  }
}

# ---------- primary region ----------

resource "aws_ssm_parameter" "primary" {
  for_each = var.parameters

  name        = "${var.path_prefix}/${each.key}"
  type        = each.value.type
  tier        = each.value.tier
  description = each.value.description
  value       = each.value.value
  key_id      = each.value.type == "SecureString" ? var.kms_key_arn_by_region[var.primary_region] : null

  tags = { Env = var.env, ManagedBy = "terraform", Scope = "region-neutral" }

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_ssm_parameter" "primary_regional" {
  for_each = var.regional_parameters

  name        = "${var.path_prefix}/${each.key}"
  type        = each.value.type
  tier        = each.value.tier
  description = each.value.description
  value       = each.value.values[var.primary_region]
  key_id      = each.value.type == "SecureString" ? var.kms_key_arn_by_region[var.primary_region] : null

  tags = { Env = var.env, ManagedBy = "terraform", Scope = "region-specific" }

  lifecycle {
    prevent_destroy = true
  }
}

# ---------- standby region ----------

resource "aws_ssm_parameter" "standby" {
  for_each = var.standby_enabled ? var.parameters : {}
  provider = aws.standby

  name        = "${var.path_prefix}/${each.key}"
  type        = each.value.type
  tier        = each.value.tier
  description = each.value.description
  value       = each.value.value
  # NOTE: the STANDBY's key. Passing the primary's ARN here fails at apply time
  # with an unhelpful error, because a SecureString can only use a local key.
  key_id      = each.value.type == "SecureString" ? var.kms_key_arn_by_region[var.standby_region] : null

  tags = { Env = var.env, ManagedBy = "terraform", Scope = "region-neutral" }

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_ssm_parameter" "standby_regional" {
  for_each = var.standby_enabled ? var.regional_parameters : {}
  provider = aws.standby

  name        = "${var.path_prefix}/${each.key}"
  type        = each.value.type
  tier        = each.value.tier
  description = each.value.description
  value       = each.value.values[var.standby_region]   # <-- the standby's own value
  key_id      = each.value.type == "SecureString" ? var.kms_key_arn_by_region[var.standby_region] : null

  tags = { Env = var.env, ManagedBy = "terraform", Scope = "region-specific" }

  lifecycle {
    prevent_destroy = true
  }
}
```

Calling it:

```hcl
module "app_parameters" {
  source = "../../modules/ssm-parameters"
  providers = { aws = aws, aws.standby = aws.standby }

  env            = var.env
  path_prefix    = "/${var.env}/app"
  primary_region = var.primary_region
  standby_region = var.standby_region

  kms_key_arn_by_region = module.app_key.key_arn_by_region

  parameters = {
    "log-level"       = { value = "info" }
    "request-timeout" = { value = "5000" }
    "feature/new-ui"  = { value = "true" }
  }

  regional_parameters = {
    "sqs-url" = {
      values = {
        (var.primary_region) = module.queue.url_by_region[var.primary_region]
        (var.standby_region) = module.queue.url_by_region[var.standby_region]
      }
    }
    "rds-endpoint" = {
      values = {
        (var.primary_region) = module.db.primary_endpoint
        (var.standby_region) = module.db.standby_endpoint
      }
    }
  }
}
```

The value of this shape is that **a region-specific parameter cannot be added
without supplying a standby value** — the precondition fails the plan. That
turns the note's biggest gotcha into a compile-time error instead of a 3am
discovery.

### The `ignore_changes` variant for out-of-band secrets

Where you accept Terraform creating the parameter but not owning the value:

```hcl
resource "aws_ssm_parameter" "managed_shell" {
  for_each = var.externally_set_parameters   # map(name => {type, tier})
  provider = aws.standby

  name   = "${var.path_prefix}/${each.key}"
  type   = "SecureString"
  tier   = each.value.tier
  key_id = var.kms_key_arn_by_region[var.standby_region]
  value  = "PLACEHOLDER_SET_BY_PIPELINE"

  lifecycle {
    ignore_changes  = [value, version]
    prevent_destroy = true
  }
}
```

Include `version` in `ignore_changes` — `aws_ssm_parameter` exposes `version`
as a computed attribute that increments on every out-of-band write, and leaving
it out has historically produced noisy plans.

### Adopting existing parameters

Parameters already exist in the primary. Use `import` blocks (TF ≥ 1.5) rather
than recreating:

```hcl
import {
  to = module.app_parameters.aws_ssm_parameter.primary["log-level"]
  id = "/prod/app/log-level"
}
```

Generate these mechanically from `describe-parameters` output; there will be
hundreds. Then `terraform plan` and confirm **0 to add, 0 to destroy** for the
primary — only the standby resources should be creates.

## Migration path from single-region

1. **Inventory.** Dump every parameter, every region, every environment:

   ```bash
   for r in eu-west-1 us-east-1 ca-central-1; do
     aws ssm describe-parameters --region "$r" \
       --query 'Parameters[].[Name,Type,Tier,LastModifiedDate,LastModifiedUser]' \
       --output text | sed "s|^|$r\t|"
   done
   ```

   `LastModifiedUser` is the important column: anything not modified by a
   Terraform or CI role is a parameter someone edits by hand, and is therefore
   drift waiting to happen.
2. **Classify** into region-neutral / region-specific / secret, using the grep
   from earlier. Expect surprises. This is the step that takes the time and the
   step that determines whether the migration works.
3. **Set the standby's default parameter tier** to match the primary
   (`aws ssm update-service-setting --setting-id .../default-parameter-tier`),
   and request the higher-throughput setting if the primary has it.
4. **Import existing primary parameters** into the new module. Confirm a
   no-change plan.
5. **Apply the standby side.** Purely additive — no resource in the primary is
   touched. **No downtime, no replacement.**
6. **Verify by comparison, not by faith:**

   ```bash
   diff \
     <(aws ssm get-parameters-by-path --path /prod --recursive --region eu-west-1 \
        --query 'sort_by(Parameters,&Name)[].Name' --output text | tr '\t' '\n') \
     <(aws ssm get-parameters-by-path --path /prod --recursive --region eu-west-2 \
        --query 'sort_by(Parameters,&Name)[].Name' --output text | tr '\t' '\n')
   ```

   Names should match exactly; values should differ **only** for the parameters
   you classified as region-specific. Put this in CI as a scheduled check — it
   is about ten lines and it catches the entire class of "someone added a
   parameter to prod and forgot the standby".
7. **Add a guardrail** so new parameters cannot be added single-region: a CI
   check that every `aws_ssm_parameter` in the repo either goes through the
   module or is explicitly annotated as primary-only.

**Nothing in this migration forces replacement.** `aws_ssm_parameter` changes
that do force replacement are `name` and `type`
(`String` ↔ `SecureString` ↔ `StringList`) — so renaming a parameter or
changing its type destroys and recreates it. With `prevent_destroy` set, that
becomes a plan error you have to consciously work around, which is what you
want.

## Failover procedure

If parameters are pre-populated: **nothing to do.** That is the goal.

Pre-flight check for [[failover-runbook]]:

```bash
# Parameter name sets must match between the pair.
PRI=$(aws ssm get-parameters-by-path --path "/$ENV" --recursive --region "$PRIMARY" \
      --query 'Parameters[].Name' --output text | tr '\t' '\n' | sort)
STB=$(aws ssm get-parameters-by-path --path "/$ENV" --recursive --region "$STANDBY" \
      --query 'Parameters[].Name' --output text | tr '\t' '\n' | sort)
diff <(echo "$PRI") <(echo "$STB") && echo "parameter sets match"

# SecureStrings actually decrypt in the standby (catches a broken KMS key policy).
aws ssm get-parameters-by-path --path "/$ENV" --recursive --with-decryption \
  --region "$STANDBY" >/dev/null && echo "securestrings decrypt OK"
```

That second check is worth more than it looks: a standby key policy that
doesn't grant the standby's node role `kms:Decrypt` produces parameters that
exist, look fine in `describe-parameters`, and fail at read time. It is exactly
the failure mode described in [[aws-kms]] gotcha #5.

## Failback

Config written in the standby during an outage (a feature flag flipped, a
timeout tuned under load) does **not** flow back. With option (a) the next
`terraform apply` after the primary returns will revert the standby to the
committed value, which may or may not be what you want — it is correct if the
change was an emergency hack, and a silent regression if it was a real fix.

**Process, not technology:** any config change made during an incident must be
committed to the Terraform repo before the incident is closed. Put it in the
incident template as a checklist item. There is no technical mechanism that
will do this for you, and pretending otherwise is how the same emergency change
gets made twice. See [[failback-strategy]].

## Gotchas

1. **No native replication exists.** The single most important fact. Do not
   assume Parameter Store behaves like Secrets Manager just because both are
   "AWS config stores".
2. **Parameters containing region-specific values are landmines.** A replicated
   SQS URL, RDS endpoint, bucket name or KMS ARN points the standby at the dead
   region. Classify before you replicate.
3. **`GetParameter*` share a 40 TPS default.** A cold-starting fleet exhausts it.
4. **`PutParameter` is 3 TPS default, 10 TPS raised.** Bulk writes and
   replication Lambdas throttle. Copying parameters at failover time cannot meet
   a 15-minute RTO.
5. **SecureString throughput is additionally limited by KMS**, per AWS's own
   footnote, and the EU standby has 1/5th the KMS quota of the EU primary.
6. **`SecureString` values appear in Terraform state in plaintext.** Either
   accept it with an encrypted, access-controlled backend, or use
   `ignore_changes = [value]` and set values out of band.
7. **`ignore_changes` does not prevent destroy.** A forced replacement recreates
   the parameter with the placeholder value. Always pair with `prevent_destroy`.
8. **A SecureString's `key_id` must be a key in its own region.** Passing the
   primary key's ARN to the standby parameter fails. Use the
   `key_arn_by_region` map from [[aws-kms]].
9. **The default parameter tier is a per-region service setting** that a new
   region won't inherit. A >4 KB parameter will fail to replicate into a
   Standard-default standby. Set it explicitly per parameter.
10. **Advanced tier is one-way.** You cannot downgrade; you must delete and
    recreate. And it costs $0.05/parameter/month in *each* region.
11. **EventBridge parameter events are best-effort** — AWS's words. Any
    event-driven replicator needs a reconciliation sweep as well.
12. **The EventBridge event does not carry the value**, so a replicator must
    re-read it, which means a cross-region function with decrypt rights on
    production secrets.
13. **Only 100 parameter versions are retained.** A chatty replicator that
    rewrites unchanged values burns version history and can lose the ability to
    roll back.
14. **Changing a parameter's `type` or `name` in Terraform forces replacement.**
15. **Billing counts per-parameter interactions, not per-request** — batching
    with `GetParameters` saves latency and quota but not money.
16. **Parameters deleted in the primary are not deleted in the standby** by any
    mechanism except Terraform. Stale parameters accumulate and can shadow real
    config.

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Replication mechanism | Terraform writes both regions (+ pipeline for pipeline-owned params) | EventBridge + Lambda replicator | **A.** RPO 2h does not need seconds. A gives drift detection, handles region-specific values natively, adds no bespoke infrastructure, and cannot fail silently during an incident. Build B only for parameters the application writes at runtime — and first check whether any exist. |
| SecureStrings in Terraform state | Accept, with encrypted S3 backend + strict access | `ignore_changes = [value]`, set out of band | **Depends on who can read state.** If state access ≈ production secret access already, accept it and keep the drift detection. If not, `ignore_changes` + `prevent_destroy`. Decide once, repo-wide, and write it down. |
| Secrets in Parameter Store vs Secrets Manager | Keep in Parameter Store (free, no replication) | Move to Secrets Manager ($0.40/secret/month × 2 regions, native replication) | **Move the genuine secrets; keep the config.** Rotatable credentials → Secrets Manager. Log levels, timeouts, feature flags → Parameter Store standard tier. The split is by nature of the data, not by convenience. |
| Region-specific values | Replicate verbatim and fix at failover | Compute per region in Terraform | **B, emphatically.** "Fix at failover" is 3 TPS of `PutParameter` on your worst night. |
| Tier | Standard everywhere | Advanced where needed | **Standard by default**, explicit per parameter, Advanced only where >4 KB or policies are genuinely required. It's one-way and it's billed per region. |
| Higher throughput setting | Default 40 TPS | Enable in standby | **Enable in the standby before it's needed**, and load-test a simulated cold start. |

## Cost

| Item | Price |
|---|---|
| Standard parameters | **No additional charge** |
| Advanced parameters | **$0.05 per advanced parameter per month**, prorated hourly, **per region** |
| API interactions (standard throughput) | **$0.05 per 10,000 interactions** |
| API interactions (higher throughput) | **$0.05 per 10,000 interactions**; AWS documents that enabling the higher TPS quota *"incurs a charge on your AWS account"* — confirm the current detail on the pricing page before budgeting |

An interaction is *"an interaction between an API request and an individual
parameter"* — a `GetParameters` call for 10 names is 10 interactions.

**Practical standby cost: near zero** if you stay on the standard tier. This is
one of the cheapest prerequisites in the whole programme, which is a strong
argument for doing it early rather than deferring it behind [[amazon-eks]].

## Open questions

- **Does anything write parameters at runtime?** Check `LastModifiedUser` across
  the estate for non-Terraform, non-CI principals. This single answer decides
  whether option (b) is needed at all.
- How many parameters are there, per environment, per region? (Drives tier,
  throughput and the size of the classification exercise.)
- Which parameters contain region-specific values? Run the grep. Expect the
  number to be higher than anyone guesses.
- Is the higher-throughput setting currently enabled in the primaries? If yes,
  what drove it — and is the standby sized for the same load?
- Which "parameters" are actually secrets that belong in
  [[aws-secrets-manager]]? Count them; the cost calculus depends on it.
- Is Terraform state currently readable by anyone who shouldn't see production
  secrets? Determines the `ignore_changes` decision.
- Are any parameters consumed by things outside the application — CodeBuild
  buildspecs, CloudFormation `{{resolve:ssm:...}}`, EC2 user data? Those
  consumers also need to resolve in the standby region.

## Sources

- [Choosing parameter tiers in Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-advanced-parameters.html) — the Standard/Advanced comparison table (10,000 vs 100,000 parameters, 4 KB vs 8 KB), the default-tier service setting, and the explicit "can't change an advanced parameter to a standard parameter" with AWS's three reasons.
- [AWS Systems Manager endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/ssm.html) — **the throughput numbers**: 40 TPS shared across `GetParameter`/`GetParameters`/`GetParametersByPath`; `PutParameter` 3 → 10 TPS; `DeleteParameter` 3 → 5; 100 retained versions; 10 policies per advanced parameter; and the footnote that SecureString throughput is further limited by KMS. Also confirms `ca-west-1` SSM endpoints exist.
- [Increasing or resetting Parameter Store throughput](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-throughput.html) — how to turn on the higher-throughput setting, referenced from the quota table.
- [Setting up notifications or triggering actions based on Parameter Store events](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-paramstore-cwe.html) — the native EventBridge integration: `source: aws.ssm`, `detail-type: "Parameter Store Change"`, operations `Create`/`Update`/`Delete`/`LabelParameterVersion`, plus `"Parameter Store Policy Action"`. **And the load-bearing caveat: "Events are emitted on a best effort basis."**
- [AWS Systems Manager pricing](https://aws.amazon.com/systems-manager/pricing/) — standard parameters free, advanced $0.05/parameter/month prorated hourly, $0.05 per 10,000 API interactions, and the per-parameter definition of an "interaction".
- [AWS Systems Manager (SSM) Cross Region Replication — CustomInk Technology Blog](https://technology.customink.com/blog/2022/01/10/aws-systems-manager-ssm-cross-region-replication/) — a real engineering-team write-up of building option (b). Useful for the shape of the problem and as evidence the gap is widely felt.
- [alessandrobologna/parameter-store-replicator](https://github.com/alessandrobologna/parameter-store-replicator) — an open-source implementation of the EventBridge + Lambda replicator, if option (b) turns out to be required.
- [`aws_ssm_parameter` resource docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ssm_parameter) — `tier`, `key_id`, `overwrite` semantics, the computed `version` attribute, and the ForceNew behaviour on `name`/`type`.
- [AWS Secrets and Configuration Provider (ASCP) for the Secrets Store CSI Driver](https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html) — the supported way to mount both Parameter Store parameters and Secrets Manager secrets into EKS pods with caching, which is the main defence against the 40 TPS read quota.
