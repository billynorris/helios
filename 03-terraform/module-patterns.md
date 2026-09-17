---
title: Module Patterns for a Region Pair — wrappers, generation, feature flags, drift
service: terraform
tags: [terraform, multi-region, modules, cookiecutter, code-generation, feature-flags, drift]
status: researched
replication: N/A — this is a code-structure note, not an AWS service
rpo_achievable: N/A
rto_achievable: "Indirect: the standby_enabled pattern is what makes a 15-minute failover a flag flip rather than an apply of new resources"
meets_targets: conditional
updated: 2026-09-17
---

# Module Patterns for a Region Pair

> Read [[provider-aliases-vs-separate-stacks]] first. That note makes the
> structural call — **separate root modules per region, generated from one
> cookiecutter template, plus a small dual-region `pair` stack**. This note is
> what you actually write inside that structure: the module signatures, the
> generation mechanics, the rollout flag, and how you prove the standby matches
> the primary. [[state-management]] covers where the state for each of those root
> modules lives.

## TL;DR

1. **Two module shapes exist and you need both.** A *pair wrapper* taking
   `aws.primary` + `aws.standby` via `configuration_aliases`, for the handful of
   resources that are genuinely one object spanning two regions. A *plain
   regional module* called once per root module, for everything else. Default to
   the second; reach for the first only in the `pair` stack.
2. **The pair wrapper creates a dependency ordering you cannot escape.** Feeding
   `module.x.primary_arn` into a standby resource means the standby's plan cannot
   be produced unless the primary's refresh succeeds. That is the failure mode
   [[provider-aliases-vs-separate-stacks]] §5 is about, restated at module scope.
3. **`for_each` over a region map does not work for providers in HashiCorp
   Terraform, and no amount of cleverness changes that.** The number of regions
   in a root module is a property of the *text*. Code generation is therefore not
   a workaround, it is the correct answer — and this team already owns a
   generator. Cookiecutter, not Terragrunt, not OpenTofu.
4. **`standby_enabled` should be a map of per-service flags, not a single
   boolean.** The prerequisites-first strategy is a per-service rollout, so the
   flag surface must be per-service or the flag is useless. Gate with
   `count = ... ? 1 : 0` at the module call site; the index-churn hazard is not
   the flip, it is the *first* introduction of the flag onto an already-existing
   resource, which changes its state address from `foo` to `foo[0]`. That needs a
   `moved` block or it is a destroy/recreate.
5. **The thing that will bite:** nobody checks the standby. Structural drift is
   solved by shared modules; *value* drift (a tfvar someone edited in the primary
   only) is not, and it is invisible until the failover. Nightly
   `terraform plan -detailed-exitcode` against both roots is the cheapest real
   answer and AWS Well-Architected REL13-BP04 names failing to do this as a
   direct anti-pattern.

---

## 1. Shape A — the pair wrapper module

A module that takes two aliased providers and emits both halves of a pair in one
call. This is the shape most teams write first, and it is right for a small,
specific set of resources.

### 1.1 The module contract

```hcl
# modules/pair-queue/versions.tf

terraform {
  required_version = ">= 1.11"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 6.0"

      # A DECLARATION, not a configuration. The caller must supply both.
      # Because this module contains no `provider` block of its own, it remains
      # compatible with count/for_each/depends_on at the call site.
      configuration_aliases = [aws.primary, aws.standby]
    }
  }
}
```

The `configuration_aliases` distinction matters and is widely misunderstood.
[Providers Within Modules](https://developer.hashicorp.com/terraform/language/modules/develop/providers)
is explicit that aliased configurations are *"never inherited automatically"* by
child modules — they must be passed. It is equally explicit that the
`count`/`for_each` prohibition applies to modules containing their own `provider`
blocks, because *"a provider configuration must always stay present in the
overall Terraform configuration for longer than all of the resources it
manages."* A module that only *declares* aliases is fine.

### 1.2 The module body

```hcl
# modules/pair-queue/variables.tf

variable "pair_name" {
  description = "Logical name of the region pair, e.g. prod-eu. Used in resource names."
  type        = string
}

variable "queue_name" {
  description = "Unqualified queue name, identical in both regions."
  type        = string
}

variable "standby_enabled" {
  description = "Build the standby half. False leaves the primary untouched."
  type        = bool
  default     = false
}

variable "message_retention_seconds" {
  description = "Set once. Applied identically to both halves — this is the point of the wrapper."
  type        = number
  default     = 345600
}
```

```hcl
# modules/pair-queue/main.tf

locals {
  # Identical name in both regions. The region is already in the ARN; putting it
  # in the name too is the single most common thing that makes a standby
  # non-substitutable. See [[dynamodb-table-naming-migration]] for the version of
  # this mistake that is currently blocking the DynamoDB work.
  name = "${var.pair_name}-${var.queue_name}"
}

# --- primary half ------------------------------------------------------------

resource "aws_kms_key" "primary" {
  provider = aws.primary

  description             = "Encryption for ${local.name}"
  enable_key_rotation     = true
  deletion_window_in_days = 30
  multi_region            = false # ForceNew. See [[aws-kms]] before changing.
}

resource "aws_sqs_queue" "primary" {
  provider = aws.primary

  name                      = local.name
  message_retention_seconds = var.message_retention_seconds
  kms_master_key_id         = aws_kms_key.primary.arn
}

# --- standby half ------------------------------------------------------------

resource "aws_kms_key" "standby" {
  provider = aws.standby
  count    = var.standby_enabled ? 1 : 0

  description             = "Encryption for ${local.name} (standby)"
  enable_key_rotation     = true
  deletion_window_in_days = 30
  multi_region            = false
}

resource "aws_sqs_queue" "standby" {
  provider = aws.standby
  count    = var.standby_enabled ? 1 : 0

  name                      = local.name # SAME name. Different region.
  message_retention_seconds = var.message_retention_seconds
  kms_master_key_id         = aws_kms_key.standby[0].arn
}
```

```hcl
# modules/pair-queue/outputs.tf

output "primary_queue_arn" {
  value = aws_sqs_queue.primary.arn
}

output "standby_queue_arn" {
  # one() returns null for a zero-element list rather than erroring — exactly the
  # shape a count-gated resource produces. Documented for this use case.
  value = one(aws_sqs_queue.standby[*].arn)
}

output "queue_name" {
  description = "Region-independent. Consumers should derive, not look up."
  value       = local.name
}
```

`one()` is the correct accessor here. The
[function docs](https://developer.hashicorp.com/terraform/language/functions/one)
describe precisely this case: *"a conditional item is represented as either a
zero- or one-element list, where a module author wishes to return a single value
that might be null instead."* `aws_sqs_queue.standby[0].arn` errors when the flag
is off; `one(...)` returns `null`.

### 1.3 Calling it

```hcl
# live/prod-eu/pair/main.tf   — the small dual-region "pair" stack

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

module "orders_queue" {
  source = "../../../modules/pair-queue"

  providers = {
    aws.primary = aws.primary
    aws.standby = aws.standby
  }

  pair_name       = "prod-eu"
  queue_name      = "orders"
  standby_enabled = true
}
```

Note there is deliberately **no default (unaliased) `aws` provider** in this root
module. That is a safety property, not a style choice — see §6.

---

## 2. Shape B — a plain regional module, called once per root module

The same outcome, expressed as a region-agnostic module that knows nothing about
pairs and is instantiated once in each region's root module.

```hcl
# modules/queue/versions.tf   — no configuration_aliases at all

terraform {
  required_version = ">= 1.11"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 6.0"
    }
  }
}
```

```hcl
# modules/queue/variables.tf

variable "pair_name" { type = string }
variable "queue_name" { type = string }

variable "region_role" {
  description = "primary | standby. Drives nothing structural — only tags and alarms."
  type        = string

  validation {
    condition     = contains(["primary", "standby"], var.region_role)
    error_message = "region_role must be primary or standby."
  }
}

variable "peer_region" {
  description = "The other half of the pair. Used to derive peer names, never to look them up."
  type        = string
}

variable "message_retention_seconds" {
  type    = number
  default = 345600
}
```

```hcl
# modules/queue/main.tf

locals {
  name = "${var.pair_name}-${var.queue_name}"
}

resource "aws_kms_key" "this" {
  description             = "Encryption for ${local.name}"
  enable_key_rotation     = true
  deletion_window_in_days = 30
}

resource "aws_sqs_queue" "this" {
  name                      = local.name
  message_retention_seconds = var.message_retention_seconds
  kms_master_key_id         = aws_kms_key.this.arn
}

# The DLQ alarm only pages in the region that is actually serving. The standby's
# queues are empty by design; alarming on them generates noise that trains people
# to ignore the standby's alarms, which is worse than not having them.
resource "aws_cloudwatch_metric_alarm" "dlq_depth" {
  count = var.region_role == "primary" ? 1 : 0

  alarm_name  = "${local.name}-dlq-depth"
  namespace   = "AWS/SQS"
  metric_name = "ApproximateNumberOfMessagesVisible"
  dimensions  = { QueueName = local.name }

  comparison_operator = "GreaterThanThreshold"
  threshold           = 0
  evaluation_periods  = 1
  period              = 300
  statistic           = "Maximum"
}
```

Called identically from both root modules, differing only in the tfvars:

```hcl
# live/prod-eu/eu-west-1/main.tf
provider "aws" { region = "eu-west-1" }

module "orders_queue" {
  source      = "../../../modules/queue"
  pair_name   = "prod-eu"
  queue_name  = "orders"
  region_role = "primary"
  peer_region = "eu-west-2"
}
```

```hcl
# live/prod-eu/eu-west-2/main.tf
provider "aws" { region = "eu-west-2" }

module "orders_queue" {
  source      = "../../../modules/queue"
  pair_name   = "prod-eu"
  queue_name  = "orders"
  region_role = "standby"
  peer_region = "eu-west-1"
}
```

### 2.1 A vs B

| | A: pair wrapper | B: regional module × 2 |
|---|---|---|
| Provider plumbing | `configuration_aliases`, `providers = {}` at every call | None — default provider inherited |
| Number of applies | 1 | 2 (one per root module) |
| Blast radius | both regions | one region each |
| Standby appliable during primary outage | **no** | yes |
| Cross-region references | free expressions | need a channel (§3) |
| Drift between halves | impossible | possible, must be policed (§5) |
| Fits the recommendation in [[provider-aliases-vs-separate-stacks]] | only in the `pair` stack | **yes, everywhere else** |
| Module reusable for a third region later | no — two is baked in | yes |

**Recommendation: Shape B by default; Shape A only inside the `pair` stack.**

The last row of that table is the one people under-weight. A pair wrapper hard-
codes the arity of the relationship into the module's type signature. The day
someone wants `eu-west-1` + `eu-west-2` + `eu-central-1`, a Shape A module is a
rewrite and a Shape B module is one more `module` block. Given the company
already runs three independent deployments and the brief explicitly frames the
pairs as *"a working assumption, not a final decision"*, baking "exactly two" into
every module signature is a bet worth not taking.

The honest counter: Shape A really is better for objects that *are* one thing.
A DynamoDB global table is not two tables, it is one table with replicas; an S3
replication rule references the destination bucket ARN; a KMS multi-region
replica key must reference its primary. Splitting those across two applies
creates ordering ceremony worse than the blast radius it avoids. That is exactly
the carve-out list in [[provider-aliases-vs-separate-stacks]] §6.2.

---

## 3. How primary outputs feed standby inputs, and the ordering it creates

This is where the two shapes genuinely diverge, and it is worth being precise
because the consequence is operational, not aesthetic.

### 3.1 Inside a pair wrapper: implicit, free, and fate-sharing

```hcl
# modules/pair-bucket/main.tf   — S3 CRR, a legitimate Shape A case

resource "aws_s3_bucket" "primary" {
  provider = aws.primary
  bucket   = "${var.pair_name}-${var.bucket_name}-eu-west-1"
}

resource "aws_s3_bucket" "standby" {
  provider = aws.standby
  bucket   = "${var.pair_name}-${var.bucket_name}-eu-west-2"
}

resource "aws_s3_bucket_versioning" "standby" {
  provider = aws.standby
  bucket   = aws_s3_bucket.standby.id
  versioning_configuration { status = "Enabled" }
}

# The replication rule lives on the PRIMARY but references the STANDBY's ARN.
# This edge is what makes the pair a single object.
resource "aws_s3_bucket_replication_configuration" "primary" {
  provider   = aws.primary
  depends_on = [aws_s3_bucket_versioning.primary, aws_s3_bucket_versioning.standby]

  bucket = aws_s3_bucket.primary.id
  role   = aws_iam_role.replication.arn

  rule {
    id     = "to-standby"
    status = "Enabled"
    filter {}

    destination {
      bucket        = aws_s3_bucket.standby.arn
      storage_class = "STANDARD_IA"

      encryption_configuration {
        replica_kms_key_id = aws_kms_key.standby.arn # a key in the DESTINATION region
      }
    }

    delete_marker_replication { status = "Disabled" }
  }
}
```

The dependency graph Terraform derives from this is:

```
aws_kms_key.standby ─┐
aws_s3_bucket.standby ┼─► aws_s3_bucket_replication_configuration.primary
aws_s3_bucket.primary ─┘        ▲
                                └── aws_iam_role.replication (global)
```

Two things follow, and both are load-bearing:

- **The apply is ordered standby-then-primary for this object**, which is the
  opposite of what people assume. You cannot create the replication rule before
  the destination bucket exists.
- **Every resource in the configuration is refreshed before any of it is
  planned.** Terraform refreshes the whole state, not the subgraph you care
  about. If `eu-west-1`'s S3 endpoint is unreachable, the refresh of
  `aws_s3_bucket.primary` fails and you get no plan at all — including for the
  `eu-west-2` resources that are perfectly healthy. `-refresh=false` sidesteps
  the refresh but not the apply, and it is not a thing to discover at 3am. This
  is the concrete mechanism behind the abstract warning in
  [[provider-aliases-vs-separate-stacks]] §5.2.

### 3.2 Across root modules: explicit, and deliberately not fate-sharing

With Shape B the standby root module has no expression that can reach the
primary. Three channels exist and only two are acceptable.

| Channel | Ordering created | Survives a primary outage | Verdict |
|---|---|---|---|
| `terraform_remote_state` on the primary's bucket | primary must apply first | **no** — reads S3 in the dead region | **Banned on the failover path** |
| SSM Parameter Store, written by the primary, read by the standby | primary must apply first | only if the parameter is *replicated into* the standby | Acceptable with care |
| Naming convention — the standby *derives* the name | **none** | yes | **Preferred** |

The convention channel, concretely:

```hcl
# In the standby root module. No lookup, no data source, no cross-region read.
locals {
  # The primary's queue is named by the same rule this module uses. We do not
  # need to ask AWS what it is called; we already know.
  peer_queue_name = "${var.pair_name}-orders"

  peer_queue_arn = format(
    "arn:%s:sqs:%s:%s:%s",
    data.aws_partition.current.partition,
    var.peer_region,                      # NOT hardcoded, NOT data.aws_region
    data.aws_caller_identity.current.account_id,
    local.peer_queue_name,
  )
}
```

This is the same conclusion the AWS Architecture blog reached the hard way in the
athenahealth Terraform Enterprise DR work: their failover scripts *"initially
attempted to retrieve infrastructure identifiers from primary-Region S3 state
files"*, and during a regional outage *"this made recovery impossible"* — the fix
was region-independent configuration sources
([Validating multi-Region DR for Terraform Enterprise with AWS FIS](https://aws.amazon.com/blogs/architecture/validating-multi-region-dr-for-terraform-enterprise-with-aws-fis/)).

If a value genuinely cannot be derived — an auto-generated ID, an ACM certificate
ARN, an EKS OIDC issuer hash — the SSM channel is the fallback, and the parameter
must be written **into the standby region** by the primary's apply, not read
across regions at plan time:

```hcl
# In the PRIMARY root module: publish, into the STANDBY region.
provider "aws" {
  alias  = "publish_to_standby"
  region = var.standby_region
}

resource "aws_ssm_parameter" "primary_cluster_oidc" {
  provider = aws.publish_to_standby

  name  = "/${var.pair_name}/peer/eks/oidc_issuer"
  type  = "String"
  value = module.eks.oidc_issuer_url
}
```

```hcl
# In the STANDBY root module: read locally. eu-west-2 reading eu-west-2.
data "aws_ssm_parameter" "peer_oidc" {
  name = "/${var.pair_name}/peer/eks/oidc_issuer"
}
```

Yes, that is a provider alias inside a "no aliases" root module. It is the one
justified use: a *write* into the peer during normal operations, on the primary
side only, where failure is tolerable. The standby never reads across a region
boundary. See [[aws-ssm-parameter-store]] — SSM has no native cross-region
replication, so this publish-on-write pattern is the workaround, and it is a
real gap worth naming.

---

## 4. `for_each` over a region map, and why generation wins

### 4.1 What you want to write, and why it does not work

```hcl
# THIS DOES NOT WORK IN HASHICORP TERRAFORM.
variable "regions" {
  type = map(object({ vpc_cidr = string, role = string }))
  default = {
    "eu-west-1" = { vpc_cidr = "10.10.0.0/16", role = "primary" }
    "eu-west-2" = { vpc_cidr = "10.20.0.0/16", role = "standby" }
  }
}

provider "aws" {
  alias    = "regional"
  for_each = var.regions   # Error: Reserved argument name in provider block
  region   = each.key
}
```

The error is that `for_each` is *reserved for use by Terraform in a future
version* inside a `provider` block. The cause is evaluation order: provider
configurations are resolved before the resource graph is walked, and `for_each`
is evaluated during the walk.
[hashicorp/terraform#27448](https://github.com/hashicorp/terraform/issues/27448)
has tracked this since January 2021;
[#24476](https://github.com/hashicorp/terraform/issues/24476) is the same request
phrased as "pass providers to modules in `for_each`". HashiCorp's support article
[Using count or for_each in Provider Configuration](https://support.hashicorp.com/hc/en-us/articles/6304194229267-Using-count-or-for-each-in-Provider-Configuration)
states the position: declare provider configurations statically in the root
module and pass them down.

The consequence is the sentence to internalise: **the set of regions a root
module manages is a property of its source text, not of its inputs.** You cannot
add a region by editing a `.tfvars`. You add a region by adding text.

### 4.2 The four ways out, scored for this team

| Workaround | What it is | Cost here |
|---|---|---|
| **Hand-written duplicate root modules** | Copy the directory, change three strings | Free today, unmaintainable at regions × envs × pairs |
| **Code generation (cookiecutter)** | One template, rendered per region, output committed | **Already owned. No new tool, no state migration.** |
| **Terragrunt `generate`** | `terragrunt.hcl` emits `provider.tf`/`backend.tf` per unit | A second config language, overlapping cookiecutter's job. Rejected in [[provider-aliases-vs-separate-stacks]] §6.4 |
| **OpenTofu 1.9 / Terraform Stacks** | Genuine provider `for_each` | Whole-estate migration, or HCP Terraform. Rejected/deferred, same note |

For completeness, Terragrunt's mechanism looks like this — it is genuinely neat,
and the reason to say no is duplication of an existing capability, not quality:

```hcl
# terragrunt.hcl at the region level
locals {
  region_vars = read_terragrunt_config(find_in_parent_folders("region.hcl"))
  aws_region  = local.region_vars.locals.aws_region
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "${local.aws_region}"
}
EOF
}
```

([Using Terragrunt's generate block to make your Terraform DRY](https://medium.com/singapore-gds/using-terragrunts-generate-block-to-make-your-terraform-dry-b0edf835f428),
Singapore GDS.)

### 4.3 What a generated per-region root module actually looks like

This is the concrete proposal. Directory layout is in [[repo-structure]]; this
section is the template contents.

**The template's variable surface** — `cookiecutter.json`:

```json
{
  "pair_name": "prod-eu",
  "environment": "prod",
  "region": "eu-west-1",
  "region_role": "primary",
  "primary_region": "eu-west-1",
  "standby_region": "eu-west-2",
  "peer_region": "{{ cookiecutter.standby_region if cookiecutter.region_role == 'primary' else cookiecutter.primary_region }}",
  "vpc_cidr": "10.10.0.0/16",
  "state_bucket": "acme-tfstate-{{ cookiecutter.region }}",
  "state_key": "{{ cookiecutter.pair_name }}/{{ cookiecutter.region }}/platform.tfstate",
  "aws_account_id": "111122223333",
  "standby_services": {
    "secrets": true,
    "kms": true,
    "ssm": false,
    "queues": false,
    "topics": false,
    "database": false,
    "eks": false
  }
}
```

**`backend.tf`** — this is the file that *must* be generated, because the backend
block accepts no variables at all. Region, bucket and key are literals or they
come from `-backend-config`. Generating them is how you make it impossible to
point the standby root module at the primary's bucket by copy-paste.

```hcl
{%- raw %}# GENERATED FILE — edit templates/region-root/, not this.
{% endraw %}
terraform {
  required_version = ">= 1.11"

  backend "s3" {
    bucket       = "{{ cookiecutter.state_bucket }}"
    key          = "{{ cookiecutter.state_key }}"
    region       = "{{ cookiecutter.region }}"
    encrypt      = true
    use_lockfile = true
  }
}
```

**`providers.tf`** — one region per root module, so exactly one provider. The
`us-east-1` alias is conditional because only the root module that owns edge
resources needs it (§ [[terraform-gotchas]] on `us-east-1`).

```hcl
provider "aws" {
  region = "{{ cookiecutter.region }}"

  default_tags {
    tags = {
      Environment = "{{ cookiecutter.environment }}"
      Pair        = "{{ cookiecutter.pair_name }}"
      RegionRole  = "{{ cookiecutter.region_role }}"
      ManagedBy   = "terraform"
      Repo        = "acme/terraform"
    }
  }

  # Assert we are pointed at the account we think we are. A wrong-account apply
  # is a worse incident than a failed one.
  allowed_account_ids = ["{{ cookiecutter.aws_account_id }}"]
}
{% if cookiecutter.region_role == "primary" %}
# Edge resources (CloudFront certs, CLOUDFRONT-scope WAF) are us-east-1-only.
# Only the primary root module owns them.
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"

  default_tags {
    tags = {
      Environment = "{{ cookiecutter.environment }}"
      Pair        = "{{ cookiecutter.pair_name }}"
      RegionRole  = "{{ cookiecutter.region_role }}"
      ManagedBy   = "terraform"
      Repo        = "acme/terraform"
    }
  }
}
{% endif %}
```

**`main.tf`** — identical text in both renders. Everything that differs is a
variable.

```hcl
locals {
  pair_name   = "{{ cookiecutter.pair_name }}"
  region      = "{{ cookiecutter.region }}"
  region_role = "{{ cookiecutter.region_role }}"
  peer_region = "{{ cookiecutter.peer_region }}"

  # A service is built here if we are the primary (always) or if the standby
  # rollout has reached it. This single local is the whole rollout mechanism.
  build = {
    for svc, on in var.standby_services :
    svc => local.region_role == "primary" ? true : on
  }
}

module "network" {
  source = "../../../modules/network"

  pair_name   = local.pair_name
  region_role = local.region_role
  vpc_cidr    = "{{ cookiecutter.vpc_cidr }}"
}

module "queues" {
  source = "../../../modules/queue"
  count  = local.build["queues"] ? 1 : 0

  pair_name   = local.pair_name
  queue_name  = "orders"
  region_role = local.region_role
  peer_region = local.peer_region
}
```

**`terraform.tfvars`** — the *only* file a human edits routinely, and the only
file whose content legitimately differs between the two renders:

```hcl
standby_services = {
  secrets  = true
  kms      = true
  ssm      = false
  queues   = false
  topics   = false
  database = false
  eks      = false
}
```

The rendering step, and the CI check that keeps it honest:

```bash
# tools/render.sh
set -euo pipefail

for spec in live/*/*/pair.yaml; do
  cookiecutter --no-input --overwrite-if-exists \
    --output-dir "$(dirname "$(dirname "$spec")")" \
    templates/region-root \
    --config-file "$spec"
done
```

```yaml
# .github/workflows/render-check.yml  (fragment)
- name: Re-render all root modules
  run: ./tools/render.sh

- name: Fail if committed output differs from a fresh render
  run: git diff --exit-code -- live/
```

That `git diff --exit-code` is the whole drift-between-template-and-output story.
It is three lines and it removes an entire class of divergence.

### 4.4 Generation vs runtime — the recommendation, stated plainly

**Generate.** Not because generation is elegant — it is not, generated code is
tedious to review — but because the alternative does not exist. Terraform will
not iterate providers, and every runtime-flavoured workaround (a giant
`for_each` over a region map with per-resource `region` arguments, one root
module spanning all regions) buys dynamism by reintroducing the shared blast
radius the whole programme exists to remove.

Three things make generation *cheap specifically here*:

1. The team already runs cookiecutter maturely. The marginal cost is a new
   template, not a new tool, a new runner, or a state migration.
2. Generated output is committed, so PRs still show real Terraform and reviewers
   still read real plans. This is the difference between generation and magic,
   and it is worth the repo noise.
3. The `git diff --exit-code` check makes template drift a build failure rather
   than an archaeology exercise.

The one rule that makes it survivable: **generated files carry a header saying
they are generated and where the source is**, and CI enforces that nobody edits
them. Without that, six months from now someone hand-patches
`live/prod-eu/eu-west-2/main.tf`, the next render silently reverts it, and
confidence in the generator is gone permanently.

---

## 5. The `standby_enabled` feature-flag pattern

This is very likely the most important section of this note for the next twelve
months, because it is the mechanism that turns "build a standby region" from a
single terrifying project into fifteen small boring PRs.

### 5.1 Why a single boolean is the wrong surface

`standby_enabled = true/false` for a whole root module gives you exactly two
states: nothing, and everything. The prerequisites-first strategy the team is
already following needs *fifteen* states — Secrets Manager on, then KMS on, then
SSM on, then SQS on. A single boolean cannot express the rollout it is supposed
to drive.

So: **a boolean for the region, and a map for the services.**

```hcl
# modules/.../variables.tf, and the root module's variable surface

variable "standby_enabled" {
  description = <<-EOT
    Master switch for this root module. When false, the standby root module
    exists, has state, is planned by CI, and builds (almost) nothing. This is the
    state a newly-rendered standby root module is merged in.
  EOT
  type        = bool
  default     = false
}

variable "standby_services" {
  description = <<-EOT
    Per-service rollout flags for the standby. Ignored entirely in the primary,
    where every service is always on. Flip one key per PR, in the order set by
    [[sequencing-roadmap]].
  EOT
  type        = map(bool)
  default     = {}
}
```

```hcl
locals {
  # The primary builds everything, always. The standby builds what has been
  # rolled out to it. Two rules, one line, no other conditionals anywhere.
  build = {
    for svc in local.all_services :
    svc => var.region_role == "primary" ? true : (
      var.standby_enabled && lookup(var.standby_services, svc, false)
    )
  }

  all_services = [
    "network", "kms", "secrets", "ssm", "queues",
    "topics", "database", "cache", "eks", "edge",
  ]
}
```

A rollout PR is then a one-line diff in one file:

```diff
 standby_services = {
   secrets  = true
   kms      = true
-  ssm      = false
+  ssm      = true
   queues   = false
 }
```

…producing a plan that touches exactly one region and creates only adds. That is
a plan a human can actually read, which is the real goal.

### 5.2 Gating with `count` — and the index-churn hazard, precisely located

The standard gate is
[Cloud Posse's convention](https://docs.cloudposse.com/learn/component-development/terraform-in-depth/terraform-count-vs-for-each/):
*"When you have a simple case where you know you want to create zero or one
instance of a resource, particularly as the result of a boolean input variable,
`count` is the best choice."*

```hcl
module "queues" {
  source = "../../../modules/queue"
  count  = local.build["queues"] ? 1 : 0
  # ...
}
```

**The flip itself is safe.** `count = 0 → 1` creates `module.queues[0]`;
`1 → 0` destroys it. Index `0` is stable across flips because there is only ever
one element. The famous `count` index-churn problem is a problem with *lists*:
Cloud Posse illustrate it with an IAM user example where inserting a user at the
front shifts every subsequent index, so unrelated resources get destroyed and
recreated. A 0-or-1 gate has no ordering to shift.

**Three real hazards remain, and they are the ones to design around:**

**Hazard 1 — introducing the flag onto an existing resource changes its
address.** This is the one that will actually happen, because the primary's
resources exist today with no `count` at all. Adding `count` to a resource
changes `aws_sqs_queue.this` to `aws_sqs_queue.this[0]`, and Terraform reads that
as "destroy the old thing, create a new thing". On a live production queue that
is data loss.

```hcl
# Add BOTH of these in the same PR that introduces the count gate.
moved {
  from = module.queues
  to   = module.queues[0]
}

# And for a resource inside a module that gains a gate:
moved {
  from = aws_sqs_queue.this
  to   = aws_sqs_queue.this[0]
}
```

The
[refactoring docs](https://developer.hashicorp.com/terraform/language/modules/develop/refactoring)
confirm the meta-argument transitions are supported —
`moved { from = aws_instance.c[0] to = aws_instance.c["small"] }` and
`moved { from = aws_instance.d[2] to = aws_instance.d }` are both documented
forms. Two constraints matter: *"A module may only make `moved` statements about
its own objects and objects of its child modules"* (so a `moved` block cannot
reach across module packages or across state files), and for shared modules
*"We strongly recommend that you retain all historical `moved` blocks from earlier
versions of your modules to preserve the upgrade path for users."* In a monorepo
with local modules, retaining them costs nothing; do it.

**Hazard 2 — gating with a computed count, not a boolean.** This is the version
that does churn:

```hcl
# DON'T. If var.subnet_cidrs is reordered, every subnet after the change is
# destroyed and recreated — in a live VPC.
resource "aws_subnet" "this" {
  count      = var.standby_enabled ? length(var.subnet_cidrs) : 0
  cidr_block = var.subnet_cidrs[count.index]
}

# DO. Keys are stable strings; reordering the map is a no-op.
resource "aws_subnet" "this" {
  for_each   = var.standby_enabled ? var.subnet_cidrs : {}
  cidr_block = each.value
  availability_zone = each.key
}
```

Rule: **`count` for on/off, `for_each` for everything with more than one
instance.** Cloud Posse's version — *"Use `for_each` when possible, and `count`
when you can't use `for_each`"* — is the right default.

**Hazard 3 — `for_each` gates must be known at plan time.** The
[`for_each` docs](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each)
are explicit: *"All values that the `for_each` argument iterates over must be
known before Terraform performs any remote resource operations"*, and *"You cannot
use sensitive values... as arguments in `for_each`."* A flag read from a variable
or a tfvars file is known. A flag derived from a resource attribute, a data
source that hits the primary region, or a secret, is not — and the plan fails
with "Invalid for_each argument" at the worst possible moment. Keep the flags in
tfvars. Always.

### 5.3 The `for_each` variant of the gate, and when it is worth it

If you expect a gate to become multi-valued later — "the standby", then "the
standby and a second standby" — a set-based gate avoids a future `moved` block:

```hcl
resource "aws_sqs_queue" "standby" {
  for_each = var.standby_enabled ? toset(["standby"]) : toset([])

  name = local.name
}

output "standby_arn" {
  value = try(aws_sqs_queue.standby["standby"].arn, null)
}
```

Address is `aws_sqs_queue.standby["standby"]` — a stable string key, not an
index. Migrating from the `count` form later is
`moved { from = aws_sqs_queue.standby[0] to = aws_sqs_queue.standby["standby"] }`.

**Recommendation: use `count` for the flags.** The `for_each` form is uglier at
every call site and buys insurance against a scenario (a third region in the same
root module) that the recommended structure already rules out — under
[[provider-aliases-vs-separate-stacks]] a new region is a new root module, not a
new key. Take the simpler form and accept a `moved` block in the unlikely event.

### 5.4 What `standby_enabled = false` should still build

Not literally nothing. A standby root module at `false` should still create the
things that are free, slow to create, or prerequisites for everything else:

| Built even at `false` | Why |
|---|---|
| VPC, subnets, route tables, security groups | Free (NAT gateways are not — gate those). CIDRs must be allocated before anything else; see [[aws-vpc-networking]] |
| IAM roles and OIDC provider for the CI pipeline | Needed to apply anything at all |
| ACM certificates | Issuance can take up to 72h. Cannot be created at failover time — see [[aws-acm]] |
| ECR repositories | Empty repos are free; image replication needs a target |
| CloudWatch log groups | Free until written to |
| KMS keys and aliases | $1/month each, and every other service needs them |

| Gated behind a flag | Why |
|---|---|
| NAT gateways | ~$32/month each before data charges |
| RDS/Aurora replicas | The largest standing cost |
| EKS node groups (the control plane should be always-on; see [[aws-eks]]) | Instance hours |
| ElastiCache nodes | Instance hours |

This is the shape that makes `standby_enabled = false` a *useful* state rather
than a placeholder: the expensive things are off, the slow things are already
warm, and the failover-time work is a flag flip, not a build.

---

## 6. Asserting the standby matches the primary

[[provider-aliases-vs-separate-stacks]] gives up one real thing by splitting the
root modules: drift between the pair becomes possible. This section buys it back.

AWS names the failure directly.
[REL13-BP04 Manage configuration drift at the DR site or Region](https://docs.aws.amazon.com/wellarchitected/latest/framework/rel_planning_for_recovery_config_drift.html)
lists as common anti-patterns: *"You fail to update recovery locations when
changes are made to the primary locations"* and *"You fail to detect
configuration drift, which leads to a false sense of DR site readiness prior to
an incident."* Risk level: **High**. Its implementation steps include using IaC
and *"regularly apply[ing] them to your disaster recovery environment"*,
configuring CI/CD to deploy to both, and — notably — *"Stagger deployments
between the primary and DR environments... This approach prevents defects from
being simultaneously pushed to production and the DR site at the same time."*

That last point is worth pausing on, because it is an argument *for* separate
applies that has nothing to do with outages: a shared apply cannot stagger. AWS's
[multi-Region fundamental 4](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-multi-region-fundamentals/fundamental-4.html)
says the same thing as a rule: *"The deployment process should target one Region
at a time instead of involving multiple Regions simultaneously."*

### 6.1 The five options

| # | Mechanism | Catches | Misses | Cost |
|---|---|---|---|---|
| 1 | **Shared module + shared tfvars** | Structural drift: different resources, different arguments | Console changes; tfvars that legitimately differ; anything not in the module | Zero — it is the design |
| 2 | **Nightly `plan -detailed-exitcode` on both roots** | Console drift, failed applies, expired things, anything Terraform owns | Drift in resources Terraform does not manage | One CI job per root module |
| 3 | **Plan-JSON fingerprint comparison** | Value drift between the pair specifically | Attributes that legitimately differ (ARNs, IDs, AZ names) | A script to write and maintain |
| 4 | **AWS Config + conformance packs** | Everything, including resources Terraform never created | Not a pair-comparison tool; it evaluates rules, not equality | Config recorder cost per region |
| 5 | **Third-party drift tooling** | As #2 with a UI | Cost, another vendor | Licence |

### 6.2 Option 2 in full — the one to build first

```yaml
# .github/workflows/drift.yml
name: drift

on:
  schedule:
    - cron: "0 3 * * *"
  workflow_dispatch:

jobs:
  plan:
    strategy:
      fail-fast: false        # a broken primary must not cancel the standby's check
      matrix:
        root:
          - { dir: live/prod-eu/eu-west-1, region: eu-west-1, role: primary }
          - { dir: live/prod-eu/eu-west-2, region: eu-west-2, role: standby }
          - { dir: live/prod-us/us-east-1, region: us-east-1, role: primary }
          - { dir: live/prod-us/us-west-2, region: us-west-2, role: standby }
    runs-on: ubuntu-latest
    permissions: { id-token: write, contents: read }
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111122223333:role/terraform-ci-${{ matrix.root.region }}
          aws-region: ${{ matrix.root.region }}   # regional STS, not the global endpoint

      - run: terraform init -input=false
        working-directory: ${{ matrix.root.dir }}

      - id: plan
        continue-on-error: true
        run: terraform plan -input=false -lock=false -detailed-exitcode -out=tfplan
        working-directory: ${{ matrix.root.dir }}

      - name: Alert on drift
        if: steps.plan.outcome == 'failure' || steps.plan.conclusion == 'failure'
        run: ./tools/notify-drift.sh "${{ matrix.root.dir }}"
```

`-detailed-exitcode` returns `0` for no changes, `1` for an error and `2` for
"changes present". On a scheduled run with no code change in between, exit 2 *is*
drift by definition — nothing in the repo moved, so anything the plan proposes
came from outside Terraform.

Two details that are easy to get wrong:

- **`fail-fast: false`.** Without it, a failing primary cancels the standby's
  job, and you lose visibility into the half you care about at exactly the moment
  you need it. This is the same principle as the CI/CD section in
  [[terraform-gotchas]], applied to the scheduled job.
- **`-lock=false`** on a read-only drift plan, so a nightly job cannot block a
  human's apply. It does mean the plan may race a concurrent apply; on a nightly
  schedule that is an acceptable trade for never blocking.

### 6.3 Option 3 — comparing the pair, not just each half

Option 2 tells you each region matches its own code. It does not tell you the two
regions match *each other*, because they are described by different tfvars. For
that, compare structured plan output:

```bash
#!/usr/bin/env bash
# tools/pair-diff.sh — assert the two halves of a pair are structurally identical.
set -euo pipefail

primary_dir="$1"   # live/prod-eu/eu-west-1
standby_dir="$2"   # live/prod-eu/eu-west-2

fingerprint() {
  terraform -chdir="$1" show -json tfplan \
    | jq -S '
        [ .planned_values.root_module
          | .. | objects | select(has("type") and has("name"))
          | { address: .address, type: .type, values: (
                .values
                # Attributes that legitimately differ between the pair.
                | del(.arn, .id, .region, .availability_zone, .availability_zones,
                      .cidr_block, .tags_all.RegionRole, .kms_key_id,
                      .owner_id, .hosted_zone_id)
              ) }
        ] | sort_by(.address)'
}

diff <(fingerprint "$primary_dir") <(fingerprint "$standby_dir") \
  || { echo "PAIR DIVERGENCE — see diff above"; exit 1; }
```

Be honest about this one: **the `del(...)` list is the whole game, and it will
grow forever.** Every legitimate difference has to be enumerated, every new
resource type adds candidates, and a stale exclusion list silently hides real
drift. It is worth building only once options 1 and 2 are in place and you have
evidence of a specific class of value drift they miss. Start with the resource
*inventory* comparison — same set of `type` + logical name in both, ignoring
values entirely — which is far more stable and catches "somebody added a queue to
the primary and forgot the standby", which is the realistic failure.

### 6.4 `check` blocks — assertions that live with the code

Terraform 1.5+ `check` blocks are underused and fit this problem well. They run
*"as the last step of plan or apply operation"* and, crucially, *"When a `check`
block's assertion fails, Terraform reports a warning and continues executing the
current operation."* A warning, not an error, is exactly right for a standby
parity assertion: you want to know, you do not want the failover apply blocked by
it.

```hcl
# In the STANDBY root module. Runs on every plan, including during an incident.
check "standby_matches_primary_config" {
  assert {
    condition     = var.region_role == "standby"
    error_message = "This root module is rendered for the standby; region_role says otherwise. The template and the tfvars disagree."
  }

  assert {
    condition = alltrue([
      for svc, on in var.standby_services : on
    ]) || !var.failover_ready
    error_message = "failover_ready is set but not every service is rolled out to the standby. This region is not promotable."
  }
}

check "standby_capacity" {
  data "aws_ec2_instance_type_offerings" "available" {
    filter {
      name   = "instance-type"
      values = [var.node_instance_type]
    }
    location_type = "availability-zone"
  }

  assert {
    condition     = length(data.aws_ec2_instance_type_offerings.available.instance_types) > 0
    error_message = "${var.node_instance_type} is not offered in ${var.region}. The standby cannot host the primary's node group shape."
  }
}
```

That second check is the kind of thing that matters for `ca-west-1` specifically
— Calgary is young and instance-type availability is a real parity question. See
[[region-pair-selection]].

### 6.5 AWS Config and conformance packs

AWS Config is the answer to a question the Terraform plans cannot answer: *is
there something in the standby that Terraform does not know about?*

Facts, verified:
[a conformance pack](https://docs.aws.amazon.com/config/latest/developerguide/conformance-packs.html)
is *"a collection of AWS Config rules and remediation actions that can be easily
deployed as a single entity in an account and a Region or across an organization
in AWS Organizations."* Per account **and Region** — so a pack is deployed once
per region, and deploying the same pack to both halves of a pair is the parity
mechanism. All six regions in scope are supported, **including `ca-west-1`**,
for both single-account and organization deployment.

What it is good at: "every EBS volume is encrypted", "every S3 bucket blocks
public access", "every RDS instance has backups on" — invariants that should hold
identically in both regions, checked against reality rather than against state.
What it is not: an equality checker. Config cannot tell you the standby has 3
subnets where the primary has 4.

**Recommendation: deploy the same conformance pack to both halves of every pair,
as a security-posture control (see [[security-posture-of-the-standby]]), and do
not count it as drift detection between the pair.** Different job, worth doing,
wrong tool for this section's question.

### 6.6 Third-party tooling

Worth stating so it does not get proposed as new: **driftctl is in maintenance
mode.** Its README says *"This project is now in maintenance mode. We cannot
promise to review contributions. Please feel free to fork the project to apply
any changes you might want to make."* Do not adopt it.

The commercial options (HCP Terraform health assessments, Spacelift, env0, Scalr)
all do scheduled drift detection well, and all of them are a platform decision
far larger than this note. If the company is already buying one, turn the feature
on. If not, §6.2 is one CI job and gets you most of the value.

### 6.7 Recommendation

**Build in this order, and stop when the marginal one stops earning:**

1. Shared modules pinned to one version, called from both roots — free,
   structural, do it as part of the design.
2. `git diff --exit-code` on a fresh render — three lines, kills template drift.
3. Nightly `plan -detailed-exitcode` on every root module, `fail-fast: false` —
   one CI job, catches everything Terraform owns. **This is the one that matters.**
4. `check` blocks for the invariants you can state in HCL — cheap, lives with
   the code, warns rather than blocks.
5. A resource-*inventory* comparison between the pair — same types and names on
   both sides, values ignored.
6. Full plan-JSON value comparison — only if 1–5 demonstrably miss something.
   The exclusion list is a maintenance burden that grows forever.

AWS Config conformance packs sit alongside all of this doing a different job.

---

## 7. The variable surface

Consolidated, because [[repo-structure]] will reference it and because getting
this list right is most of the design.

| Variable | Type | Scope | Notes |
|---|---|---|---|
| `pair_name` | string | template + module | `prod-eu`, `prod-us`, `prod-ca`. Appears in every resource name. **Region-free by construction** |
| `environment` | string | template + module | `prod`, `staging`, `dev` |
| `region` | string | template only | The region this root module manages. Never a runtime variable — it is baked into the backend |
| `region_role` | string | template + module | `primary` \| `standby`. Drives alarms and tags, never structure |
| `primary_region` | string | template + module | Needed for name derivation and for the SSM publish path |
| `standby_region` | string | template + module | Ditto |
| `peer_region` | string | module | Derived from the two above by the template. Modules take this, not the pair |
| `standby_enabled` | bool | root tfvars | Master switch. `false` for a freshly-rendered standby |
| `standby_services` | map(bool) | root tfvars | **The rollout surface.** One key per service family |
| `vpc_cidr` | string | template | From [[aws-vpc-networking]]'s allocation plan. `ForceNew` — get it right once |
| `state_bucket` / `state_key` | string | template only | Backend literals. Generated, never hand-written |
| `aws_account_id` | string | template | Feeds `allowed_account_ids` |

Two rules about this table:

- **`region` never appears as a Terraform variable inside a module.** If a module
  needs to know its region it uses `data "aws_region" "current" {}`, which
  resolves against whichever provider configuration that module was given. Note
  the attribute is now `region`; `name` and `id` on that data source are
  [deprecated](https://raw.githubusercontent.com/hashicorp/terraform-provider-aws/main/website/docs/d/region.html.markdown).
- **`region_role` must never gate structure.** The moment `region_role ==
  "standby"` starts controlling which resources exist, the two halves stop being
  the same thing and the drift problem becomes unsolvable. Use it for alarms,
  tags and desired capacity. Use `standby_services` for existence.

---

## 8. Gotchas specific to module patterns

Full list in [[terraform-gotchas]]; these come from this note's material.

- **A module with no `providers = {}` inherits the default configuration
  silently.** No error, no warning, resources built in the wrong region. The
  structural defence is to declare *no* default `aws` provider in any root module
  that has aliases — then omission is a hard error.
- **`data "aws_region"` in a child module resolves against that module's
  provider, and there is a long-standing report that it did not always do so.**
  [terraform-provider-aws#2368](https://github.com/terraform-providers/terraform-provider-aws/issues/2368),
  *"It seems that the `aws_region` data source doesn't respect the `providers`
  block that was added in Terraform 0.11"*, is still open. Assert it in a test
  rather than assuming.
- **`moved` cannot cross module packages or state files.** It is fine for
  "resource moves into a local module" and for count↔for_each transitions. It is
  useless for "this resource moves from the primary's state to the standby's" —
  that is `removed` + `import`, covered in [[terraform-gotchas]].
- **Adding a `count` gate to an existing resource is a destroy/recreate without a
  `moved` block.** §5.2 Hazard 1. This will happen during the very first rollout
  PR if nobody is looking for it.
- **`for_each` gates must be plan-time known and non-sensitive.** §5.2 Hazard 3.
- **Generated files get hand-edited.** Header, CI check, or the generator dies.
- **Pair wrapper modules bake arity into the type signature.** Two regions
  forever, by construction. Fine inside the `pair` stack, wrong as a default.
- **`one()` vs `[0]` on gated outputs.** `[0]` errors when the gate is off; a
  consumer module then fails to plan for reasons that look unrelated.

---

## 9. Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Default module shape | Pair wrapper with `aws.primary`/`aws.standby` | Region-agnostic module called once per root module | **B.** A wrapper bakes "exactly two regions" into every signature. Reserve A for the `pair` stack's genuinely-dual-region objects |
| Iterating regions | `for_each` over a region map | Code generation | **Generation.** Terraform will not iterate providers; this is not a preference |
| Generator | Terragrunt `generate` | Existing cookiecutter | **Cookiecutter.** Already owned, already mature, no new runner |
| Generated output | Rendered in CI | Committed to the repo | **Committed.** Reviewers must be able to read the plan's source |
| Rollout flag surface | One `standby_enabled` boolean | Boolean + `standby_services` map | **Both.** The boolean is the region switch; the map is the rollout |
| Gate mechanism | `count = flag ? 1 : 0` | `for_each = flag ? toset(["x"]) : []` | **`count`.** Simpler at every call site; the structure already rules out a third instance |
| Cross-region values | `terraform_remote_state` | Naming convention, SSM published into the standby as fallback | **Convention first, SSM second.** Remote state on the primary re-shares fate |
| Drift detection | Build the full plan-JSON comparator | Shared modules + render check + nightly `-detailed-exitcode` | **The latter three first.** Add comparison only when something demonstrably slips through |
| Standby alarms | Mirror the primary's | Primary only until promoted | **Primary only**, plus standby-side health checks. Noisy standby alarms train people to ignore the standby |

---

## 10. Open questions

- Does the existing cookiecutter template emit **root modules** (including
  `backend.tf`) or only service modules within a hand-written root? Everything in
  §4.3 assumes the former; if it is the latter that is the first piece of work.
- Do any existing shared modules contain their own `provider` blocks? Those
  cannot be `count`-gated and must be refactored to `configuration_aliases`
  before §5 is possible at all.
- What is the current convention for per-environment values — tfvars files,
  cookiecutter context, or a `locals` map keyed by environment? `standby_services`
  should follow it rather than inventing a third mechanism.
- Is generated Terraform currently committed? §4.3's `git diff --exit-code`
  check assumes yes.
- Is there an existing `enabled`-style flag convention in the repo's modules? If
  so, `standby_services` should reuse its spelling rather than adding a parallel
  one.
- Which services are already being alarmed on, and would mirroring those alarms
  into the standby page anyone at 3am for an empty queue?

---

## Sources

- [Providers Within Modules — Terraform docs](https://developer.hashicorp.com/terraform/language/modules/develop/providers) — `configuration_aliases`, the non-inheritance of aliased configurations, and the exact scope of the `count`/`for_each` restriction.
- [Module Composition — Terraform docs](https://developer.hashicorp.com/terraform/language/modules/develop/composition) — HashiCorp's stated preference for explicit composition and swappable implementations over built-in abstraction; the argument against "lowest common denominator" wrapper modules.
- [`for_each` meta-argument — Terraform docs](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each) — *"All values that the `for_each` argument iterates over must be known before Terraform performs any remote resource operations"*, and the prohibition on sensitive values and impure functions.
- [`count` meta-argument — Terraform docs](https://developer.hashicorp.com/terraform/language/meta-arguments/count) — *"Use the `count` argument when you want to create nearly identical instances. Use `for_each` when some instance arguments must have distinct values that can't be directly derived from an integer index."*
- [`one()` function — Terraform docs](https://developer.hashicorp.com/terraform/language/functions/one) — the documented pattern for returning a possibly-null value from a `count = 0 or 1` resource.
- [Refactoring / `moved` blocks — Terraform docs](https://developer.hashicorp.com/terraform/language/modules/develop/refactoring) — count↔for_each transitions, *"A module may only make `moved` statements about its own objects and objects of its child modules"*, and the advice to retain historical `moved` blocks.
- [`check` block reference — Terraform docs](https://developer.hashicorp.com/terraform/language/block/check) — checks run last in plan/apply; *"When a `check` block's assertion fails, Terraform reports a warning and continues executing the current operation."*
- [hashicorp/terraform#27448](https://github.com/hashicorp/terraform/issues/27448) — provider `for_each`; `for_each` is a reserved argument name in `provider` blocks.
- [hashicorp/terraform#24476](https://github.com/hashicorp/terraform/issues/24476) — "Ability to pass providers to modules in for_each", the same constraint framed at module level.
- [Using count or for_each in Provider Configuration — HashiCorp support](https://support.hashicorp.com/hc/en-us/articles/6304194229267-Using-count-or-for-each-in-Provider-Configuration) — official guidance to declare providers statically in the root module and pass them down.
- [terraform-provider-aws#2368](https://github.com/terraform-providers/terraform-provider-aws/issues/2368) — open report that `aws_region` did not respect the `providers` block in child modules.
- [`aws_region` data source docs (provider source)](https://raw.githubusercontent.com/hashicorp/terraform-provider-aws/main/website/docs/d/region.html.markdown) — the data source discovers the provider's configured region; `name` and `id` are deprecated in favour of `region`.
- [Count vs For Each — Cloud Posse reference architecture](https://docs.cloudposse.com/learn/component-development/terraform-in-depth/terraform-count-vs-for-each/) — *"Use `for_each` when possible, and `count` when you can't use `for_each`"*; `count = var.bastion_enabled ? 1 : 0` as the canonical enabled-flag; the index-churn illustration.
- [REL13-BP04 Manage configuration drift at the DR site or Region — AWS Well-Architected](https://docs.aws.amazon.com/wellarchitected/latest/framework/rel_planning_for_recovery_config_drift.html) — failing to detect drift is a named anti-pattern at **High** risk; staggered deployments; CI/CD to both sites; AWS Config for drift.
- [Multi-Region fundamental 4: Operational readiness — AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-multi-region-fundamentals/fundamental-4.html) — *"The deployment process should target one Region at a time instead of involving multiple Regions simultaneously"*; per-Region IAM role isolation; quota parity.
- [Conformance Packs for AWS Config — AWS docs](https://docs.aws.amazon.com/config/latest/developerguide/conformance-packs.html) — packs deploy *"in an account and a Region or across an organization"*; Region support table confirms all six in-scope regions including `ca-west-1`.
- [Validating multi-Region DR for Terraform Enterprise with AWS FIS — AWS Architecture blog](https://aws.amazon.com/blogs/architecture/validating-multi-region-dr-for-terraform-enterprise-with-aws-fis/) — failover scripts reading primary-Region state files made recovery impossible; region-independent configuration sources plus CI drift detection as the fix.
- [Using Terragrunt's generate block to make your Terraform DRY — Singapore GDS](https://medium.com/singapore-gds/using-terragrunts-generate-block-to-make-your-terraform-dry-b0edf835f428) — the `generate "provider"` pattern quoted in §4.2, for comparison against cookiecutter.
- [snyk/driftctl](https://github.com/snyk/driftctl) — *"This project is now in maintenance mode."*
