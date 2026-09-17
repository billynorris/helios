---
title: Provider Aliases vs Separate Stacks — how to express the region pair in code
service: terraform
tags: [terraform, multi-region, repo-structure, decision, blast-radius]
status: researched
replication: N/A — this is a code-structure decision, not an AWS service
rpo_achievable: N/A
rto_achievable: "Directly constrains RTO: option A can make the standby un-appliable during a primary outage"
meets_targets: conditional
updated: 2026-09-17
---

# Provider Aliases vs Separate Stacks

> This is the year-defining decision. Everything in [[module-patterns]],
> [[state-management]] and [[terraform-gotchas]] is downstream of it, and it is
> very expensive to reverse once three environments × three pairs of regions have
> been laid down on top of it.

## TL;DR

1. **A shared Terraform apply is a shared failure domain.** Putting a DR standby
   behind the same apply as its primary partly defeats the purpose of having a
   standby at all. This is the single most important sentence in these notes.
2. **Recommendation: separate root modules per region (Option B), instantiated
   from one parameterised stack template (Option C's mechanism).** In practice
   the recommendation is "C implemented as B" — one shared stack definition,
   rendered by cookiecutter into `eu-west-1/` and `eu-west-2/` root modules with
   independent state and independent applies.
3. The world changed underneath this question in **June 2025**: AWS provider
   v6.0 added a per-resource `region` argument, so "provider aliases" is no
   longer the only way to touch two regions from one configuration. It makes
   Option A *easier to write* — and no safer. Convenience is not the axis that
   matters here.
4. Provider configuration still **cannot be `for_each`'d in HashiCorp
   Terraform** (it can in OpenTofu 1.9+, and inside Terraform Stacks). That
   constraint is what pushes teams toward code generation — and this team
   already has a code generator: cookiecutter. See [[module-patterns]].
5. The thing that will bite: **a regional outage can block your apply**. If the
   primary's API endpoints are unreachable, a root module that spans both
   regions cannot refresh, cannot plan, and therefore cannot apply — including
   the parts of the plan that only touch the healthy standby. That is a 15-minute
   RTO evaporating while you argue with `-target`.

---

## 1. The blast-radius argument (read this first)

The reason to build a standby region is to have a failure domain that does not
share fate with the primary. Every dependency you leave shared is a hole in that
promise. The obvious ones get attention — shared VPC, shared database, shared
DNS. The one that gets missed is the **control plane you use to change things**.

If `eu-west-1` and `eu-west-2` are managed by a single root module with a single
state file and a single `terraform apply`:

- **One bad apply breaks both regions simultaneously.** A typo in a security
  group rule, a module upgrade with a `ForceNew` attribute change, a bad
  `for_each` key that destroys and recreates — all of it lands in both regions in
  the same apply, in the same five minutes, with no gap in which anyone notices.
  The standby's entire job is to be unaffected by whatever just happened to the
  primary. A shared apply is a mechanism that guarantees the opposite.
- **A half-failed apply leaves both regions half-changed.** Terraform is not
  transactional. An apply that dies partway through has mutated some resources
  and not others. With a shared root module, "some" spans two regions, and the
  state file recording which is which is one object.
- **The standby cannot be changed while the primary is broken.** This is the
  operationally fatal one, covered in §5.2.
- **It doubles plan time and halves how often people look at the plan.** Plans
  scale with resource count because refresh is one API call per resource
  ([why Terraform is slow](https://stategraph.com/blog/why-is-terraform-so-slow)
  is blunt about refresh being the bottleneck). Double the resources, double the
  wall clock, and — the real cost — double the diff a human has to read before
  approving. Long plans are skim-read plans.

This is not a novel argument, it is just usually applied at the wrong altitude.
The AWS DevOps blog on multi-region Terraform deployments states the principle
directly, describing the structure as
["Each Region as one cell, with the goal of decreasing cross-regional dependencies"](https://aws.amazon.com/blogs/devops/multi-region-terraform-deployments-with-aws-codepipeline-using-terraform-built-ci-cd/),
and notes that with per-region state *"If there is an issue with one Terraform
state file, then the rest of the state files aren't impacted"*. Gruntwork's
framing of Terragrunt is
[the same idea](https://www.gruntwork.io/blog/terragrunt-iac-collaboration-at-scale):
segment state to minimise blast radius, gain parallelism and reliability.
[Stategraph's blast-radius write-up](https://stategraph.com/blog/terraform-blast-radius)
names monolithic state as cause number one: *"When one `terraform.tfstate`
tracks networking, databases, compute resources, IAM, and application services
for an entire environment, almost every `terraform plan` has the potential to
touch more than the author expects."*

None of those sources are talking about DR specifically. Applied to DR the
argument gets sharper, because the resource you are protecting is *the
independence itself*.

**Counter-argument, stated fairly:** a shared apply makes drift structurally
impossible. Both regions are described by one configuration and converged by one
run, so they cannot silently diverge. That is a real benefit and it is the main
thing Option B gives up. The rest of this note is largely about how to buy that
benefit back without buying a shared failure domain with it (§6.3, and
[[module-patterns]] on drift detection).

---

## 2. What the language actually lets you do (2026 state of the art)

Research this rather than assuming — three things changed between 2024 and 2026.

### 2.1 Provider aliases: the classic mechanics

A `provider` block with `alias` set is an *additional* provider configuration.
Resources select one with the `provider` meta-argument; modules receive them
through the `providers` map.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.primary_region        # default configuration
}

provider "aws" {
  alias  = "standby"
  region = var.standby_region
}

resource "aws_sqs_queue" "orders_standby" {
  provider = aws.standby
  name     = "orders"
}
```

Rules that matter, from
[Providers Within Modules](https://developer.hashicorp.com/terraform/language/modules/develop/providers):

- Child modules **inherit the default (unaliased) configuration automatically**.
  Aliased configurations are *"never inherited automatically"* and must be passed
  explicitly. This is the source of the single most common multi-region bug —
  see [[terraform-gotchas]].
- A reusable module declares what it expects with `configuration_aliases`:

  ```hcl
  terraform {
    required_providers {
      aws = {
        source                = "hashicorp/aws"
        version               = ">= 6.0"
        configuration_aliases = [aws.primary, aws.standby]
      }
    }
  }
  ```

- **`configuration_aliases` is a declaration, not a configuration.** A module
  that only declares aliases and receives them from its caller *is* compatible
  with `count`, `for_each` and `depends_on`. The incompatibility applies only to
  modules that contain their own `provider` blocks, because
  *"a provider configuration must always stay present in the overall Terraform
  configuration for longer than all of the resources it manages"*. Teams
  routinely believe "provider aliases break `for_each` in modules" — that is only
  true for the legacy pattern of embedding `provider` blocks inside a module, a
  pattern deprecated since Terraform v0.13.

### 2.2 Provider configuration still cannot be dynamic — in Terraform

`count` and `for_each` are not usable in a `provider` block in HashiCorp
Terraform. The reason is evaluation order: provider configurations must be
resolved before the resource dependency graph is walked, and `for_each` is
evaluated during the graph walk. Attempting it produces an error that the
argument name is *reserved for use by Terraform in a future version*
([hashicorp/terraform#27448](https://github.com/hashicorp/terraform/issues/27448),
opened January 2021; see also
[#30461](https://github.com/hashicorp/terraform/issues/30461) on dynamic/optional
aliases). HashiCorp's own support guidance is
["Using count or for_each in Provider Configuration"](https://support.hashicorp.com/hc/en-us/articles/6304194229267-Using-count-or-for-each-in-Provider-Configuration)
— the recommendation is to declare provider configurations statically in the
root module and pass them down.

The practical consequence: **the number of regions in a root module is a
compile-time property of the source code.** You cannot add a region by changing a
variable. You add a region by adding text. That is exactly the shape a code
generator is good at, and exactly the shape a `terraform.tfvars` is bad at.

### 2.3 What *did* change

| Change | Shipped | What it gives you | Catch |
|---|---|---|---|
| **OpenTofu 1.9 provider `for_each`** | 9 Jan 2025 ([release post](https://opentofu.org/blog/opentofu-1-9-0/)) | Genuine iteration over provider configurations | Requires migrating the whole estate off Terraform onto OpenTofu |
| **AWS provider v6.0 per-resource `region`** | 18 Jun 2025 ([GA post](https://www.hashicorp.com/en/blog/terraform-aws-provider-6-0-now-generally-available)) | Most resources take a top-level `region`; no alias needed for cross-region resources | Changing `region` **forces replacement**; global services excluded |
| **Terraform Stacks GA** | Sept 2026 ([Stacks overview](https://developer.hashicorp.com/terraform/language/stacks)) | `provider` blocks *do* take `for_each`; per-deployment isolated state | Requires HCP Terraform; new file formats; not a small migration |

**OpenTofu 1.9** syntax, for completeness — this is what the feature request in
§2.2 looks like when granted:

```hcl
provider "aws" {
  alias    = "by_region"
  for_each = var.aws_regions
  region   = each.key
}

resource "aws_vpc" "private" {
  for_each   = var.aws_regions
  provider   = aws.by_region[each.key]
  cidr_block = each.value.vpc_cidr_block
}
```

OpenTofu documents two constraints:
[`for_each` on a provider requires `alias`](https://opentofu.org/docs/language/providers/configuration/)
(*"the default configuration for each provider must always have exactly one
instance"*), and *"The `for_each` expression for a resource must be different
from the `for_each` expression for its associated provider configuration"* — the
resource should iterate a subset, so a region can be removed from the resource
set before the provider that manages it disappears. That second rule is the
same fate-ordering problem as §2.1: providers must outlive their resources.

**Terraform Stacks** offers the same iteration inside a different execution
model — `providers.tfcomponent.hcl`:

```hcl
provider "aws" "configurations" {
  for_each = var.regions

  config {
    region = each.value
    assume_role_with_web_identity {
      role_arn           = var.role_arn
      web_identity_token = var.identity_token
    }
  }
}
```

and `components.tfcomponent.hcl`:

```hcl
component "s3" {
  for_each = var.regions
  source   = "./s3"
  inputs   = { region = each.value }
  providers = {
    aws    = provider.aws.configurations[each.value]
    random = provider.random.this
  }
}
```

Stacks is the only option here that iterates providers *and* keeps state
isolated — each deployment has its own state. It is also the only one that
requires buying HCP Terraform and rewriting the repo's execution model. Costed
in §6.4.

### 2.4 AWS provider v6 `region` — the option that looks like a shortcut

This deserves its own treatment because it will be the first thing someone
proposes, and it is genuinely useful in the right place.

From the
[Enhanced Region Support guide](https://github.com/hashicorp/terraform-provider-aws/blob/main/website/docs/guides/enhanced-region-support.html.markdown):

- `region` is added to most resources, data sources and ephemeral resources. It
  is *Optional and Computed*, defaulting to the provider's configured region,
  and is validated against the configured partition.
- **Changing `region` forces resource replacement.** Removing it does not —
  *"The prior value of `region` stored in Terraform state will be used."*
- Import gained a region suffix: `terraform import aws_vpc.test_vpc vpc-a01106c2@eu-west-1`.
- Global services (IAM, Route 53, CloudFront, Organizations) do not support it,
  nor do specific global resources inside regional services —
  `aws_backup_global_settings`, `aws_dx_gateway`,
  `aws_s3_account_public_access_block` among them. Data sources such as
  `aws_partition`, `aws_regions` and `aws_default_tags` are effectively global.
- Migration guidance: upgrade to 6.0, run `terraform apply -refresh-only`, *then*
  swap `provider = aws.x` for `region = "x"`.

Where this earns its keep: genuinely cross-region *resources* — S3 replication
configuration, a DynamoDB global table replica, a KMS multi-region replica key —
where previously you needed an aliased provider just to place one resource.
[Mattias' write-up](https://mattias.engineer/blog/2025/aws-provider-version-6/)
shows S3 CRR done this way and is honest that for a simple two-region case the
difference is *"not substantial"*.

Where it does not help: **it does nothing about blast radius.** A single root
module with per-resource `region` arguments has exactly the failure domain of
Option A, minus the boilerplate. It arguably makes things worse, because the
region a resource lives in is now a quiet attribute rather than a loud
`provider =` line, and getting it wrong destroys and recreates the resource. Use
it for cross-region *plumbing*; do not use it as the multi-region strategy.

---

## 3. Option A — provider aliases in one root module

One root module, one state, two (or more) aliased providers, every resource
declared twice or passed into modules twice.

```hcl
# environments/prod-eu/main.tf   — ONE root module, BOTH regions

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

module "platform_primary" {
  source    = "../../modules/platform"
  providers = { aws = aws.primary }

  region_role = "primary"
  vpc_cidr    = "10.10.0.0/16"
}

module "platform_standby" {
  source    = "../../modules/platform"
  providers = { aws = aws.standby }

  region_role = "standby"
  vpc_cidr    = "10.20.0.0/16"
}
```

**What you get**

- Drift between the pair is structurally impossible. One configuration, one
  convergence. This is a real and significant benefit and it is the reason the
  option keeps being proposed.
- Cross-region wiring is trivial: `module.platform_primary.kms_key_arn` is just
  an expression, no `terraform_remote_state`, no ordering ceremony.
- One PR, one review, one plan. The reviewer sees the whole pair.

**What you pay**

- **Shared failure domain** (§1). This is not a rounding error; it is the thing
  the project exists to eliminate.
- **Plan and apply time roughly doubles.** With refresh being one API call per
  resource, a 900-resource environment becomes an 1800-resource environment.
- **A regional outage can block the apply entirely.** Covered in §5.2 — this is
  the argument that should end the discussion.
- **`-target` becomes load-bearing during incidents**, which HashiCorp
  explicitly warns against: the flag is
  [*"provided for exceptional circumstances... not recommended for routine use"*](https://developer.hashicorp.com/terraform/cli/commands/plan)
  precisely because it produces undetected drift. Building a DR procedure on top
  of it is building on a documented anti-pattern. See [[terraform-gotchas]].
- **State file size and lock contention.** One lock now serialises all changes to
  both regions.

**When it is nonetheless right:** for the small set of resources that are
*genuinely one logical thing spanning two regions* — a DynamoDB global table, a
KMS multi-region key and its replica, an S3 bucket and its replication rule, a
Route 53 failover record set pointing at both. Those belong together in one
apply because they are one object. See §6.2.

---

## 4. Option B — separate root modules per region

Two root modules, two state files, two applies, one shared module library.

```
terraform/
  modules/                       # shared, versioned, region-agnostic
    platform/
    service-queues/
  live/
    prod-eu/
      eu-west-1/                 # root module, own backend, own state
        main.tf
        backend.tf
        terraform.tfvars
      eu-west-2/                 # root module, own backend, own state
        main.tf
        backend.tf
        terraform.tfvars
```

```hcl
# live/prod-eu/eu-west-2/main.tf     — the STANDBY root module

terraform {
  required_version = ">= 1.11"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.0" }
  }
  backend "s3" {
    bucket       = "acme-tfstate-eu-west-2"
    key          = "prod-eu/eu-west-2/platform.tfstate"
    region       = "eu-west-2"          # state lives WITH the region it manages
    use_lockfile = true
    encrypt      = true
  }
}

provider "aws" {
  region = "eu-west-2"
  default_tags { tags = local.common_tags }
}

module "platform" {
  source = "../../../modules/platform"

  pair_name       = "prod-eu"
  region_role     = "standby"
  peer_region     = "eu-west-1"
  vpc_cidr        = "10.20.0.0/16"
  standby_enabled = true
}
```

**What you get**

- **Proper blast-radius isolation.** A bad apply in `eu-west-2` cannot touch
  `eu-west-1` and vice versa. Two locks, two states, two credentials scopes.
- **You can apply to the standby while the primary is on fire.** The standby root
  module refers to no resource in the primary and reads no state from it, so its
  plan succeeds when the primary's plan does not. During a real incident this is
  the difference between "scale the standby ASG and change the Route 53 weight"
  and "Terraform won't plan, do it by hand in the console and reconcile later".
  At RTO 15m there is no time for the second one.
- Plans are half the size and therefore actually read.
- Per-region credentials become possible: the pipeline role that applies to
  `eu-west-2` needs no permissions in `eu-west-1`.

**What you pay**

- **Drift becomes possible and must be actively policed.** This is the whole
  cost, and it is manageable but it is not free. Mitigations, in increasing
  order of strength:
  1. Both roots call the *same pinned module version*. Most drift risk is
     structural and this removes it.
  2. Generate both roots from one template so the `.tf` is byte-identical apart
     from substituted values — this is where cookiecutter earns its place
     ([[module-patterns]]).
  3. A CI job that renders both roots and fails if the *non-region* portions of
     the tfvars diverge.
  4. A scheduled `terraform plan -detailed-exitcode` against both, alerting on
     exit code 2.
- **Cross-region values need an explicit channel.** `terraform_remote_state`,
  SSM Parameter Store, or plain hardcoded convention. This creates an ordering
  dependency (primary applies first for anything the standby consumes) and — if
  you choose `terraform_remote_state` pointing at the primary's bucket — quietly
  reintroduces the fate-sharing you just removed. See §5.2 and
  [[state-management]].
- **Two PRs, or one PR and two pipeline stages.** More ceremony per change.
- The number of root modules is regions × environments × pairs. Without
  generation that is a lot of directories to keep in step — which is precisely
  Option C.

---

## 5. Option C — one parameterised stack, instantiated per region

The middle ground, and the one that matches how this repo already works.

There is **one stack definition**. It is region-agnostic and takes the region
pair as parameters. It is *instantiated* once per region, producing separate
root modules with separate state — so operationally it is Option B — but there is
only one place to edit, so the drift surface of Option B mostly disappears.

Three mechanisms can do the instantiation:

| Mechanism | How | Fit here |
|---|---|---|
| **cookiecutter generation** | Template renders `live/<env>/<region>/` root modules from one source | **Native.** The repo already does this. Zero new tooling. |
| **Terragrunt `generate` + units** | `terragrunt.hcl` per directory generates `provider.tf`/`backend.tf`; `terragrunt.stack.hcl` generates the units themselves | Powerful, but a new tool, new runner, new mental model |
| **Terraform Stacks deployments** | One component set, a `deployment` block per region, isolated state per deployment | Cleanest model; requires HCP Terraform |

The cookiecutter shape, concretely:

```
terraform/
  templates/
    region-stack/                       # ONE definition
      cookiecutter.json
      {{cookiecutter.pair_name}}-{{cookiecutter.region}}/
        main.tf
        backend.tf
        providers.tf
        terraform.tfvars
```

```json
{
  "pair_name":       "prod-eu",
  "region":          "eu-west-1",
  "region_role":     "primary",
  "primary_region":  "eu-west-1",
  "standby_region":  "eu-west-2",
  "peer_region":     "eu-west-2",
  "standby_enabled": "false",
  "state_bucket":    "acme-tfstate-{{cookiecutter.region}}",
  "vpc_cidr":        "10.10.0.0/16"
}
```

Rendering it twice — once with `region_role=primary`, once with
`region_role=standby` — yields two root modules that are identical except for
region, CIDR, role and backend. Adding the CA pair is two more renders. Adding a
fourth region later is one more render, not a refactor.

The variable surface is spelled out in full in [[module-patterns]]; the four
that matter are `primary_region`, `standby_region`, `pair_name`,
`standby_enabled`.

**What you get:** Option B's isolation with most of Option A's single-source-of-
truth property. Structural changes are made once, in the template, and
re-rendered.

**What you pay:** generated code must be committed and reviewed (do it — a
generated `.tf` that nobody can read in a PR is worse than duplication), and
"re-render everything" becomes a maintenance operation with its own blast radius.
Drift is now possible between *template and rendered output*, which is a much
easier thing to check in CI than drift between two live regions —
`cookiecutter` in a job, `git diff --exit-code`, done.

---

## 6. Recommendation

### 6.1 The call

**Option C's mechanism, Option B's runtime. Separate root modules per region,
separate state, separate applies — generated from one cookiecutter template.**

The reasoning, in priority order:

1. **The standby must be appliable when the primary is not.** At RTO 15 minutes
   this is not a nice-to-have. Option A cannot guarantee it; Options B and C can.
   Nothing else on this list outranks that.
2. **A shared apply is a shared failure domain**, and the project's entire
   purpose is to remove shared failure domains between the pair. Option A
   contradicts the project.
3. **Drift — Option B's only real weakness — is the cheaper problem.** It is
   detectable in CI, it degrades gracefully (a slightly stale standby is still a
   standby), and generation removes most of its causes. A shared blast radius is
   not detectable, does not degrade gracefully, and has no mitigation short of
   not doing it.
4. **It fits the existing repo.** This is an evolution of a cookiecutter-templated
   monorepo, not a replacement of it. No new tool enters the build.

### 6.2 The carve-out — where Option A is correct

Do not be dogmatic. Some resources are *one logical object that happens to span
two regions*, and splitting them across two applies creates ordering problems
worse than the blast radius it avoids. Put these in a small, explicitly
dual-region root module — call it the **pair stack** — with both aliases:

- DynamoDB global tables (one table, N replicas) — see [[aws-dynamodb]]
- KMS multi-region keys and their replicas — see [[kms-when-to-use-multi-region-keys]]
- S3 bucket replication configuration (source rule references destination ARN) — [[aws-s3]]
- Route 53 failover/latency record sets and health checks referencing both — [[aws-route53]]
- ACM certificates in both regions where one cert is validated by one hosted zone — [[aws-acm]]
- Secrets Manager `replica` blocks (already done in prod)

That gives three root modules per pair: `primary`, `standby`, `pair`. The `pair`
stack should be **small and boring** — a few dozen resources, rarely changed. Its
blast radius is real but bounded, and crucially it is not on the failover
critical path: at 3am you are changing the *standby* stack, not the pair stack.

Within the pair stack, prefer AWS provider v6's `region` argument over aliases
for single resources that simply live elsewhere, and keep aliases for anything
that needs a distinct credential or `default_tags` set.

### 6.3 Buying back the drift guarantee

Option A's one honest advantage was that drift is impossible. Replace it with:

1. **One module library, pinned by version.** Both roots call
   `modules/platform` at the same ref. Enforce with a CI check that the
   `source`/`ref` in the two roots match.
2. **Generated roots + `git diff --exit-code` in CI.** If the committed root
   modules don't match a fresh render, the build fails.
3. **Scheduled cross-region plan.** Nightly `terraform plan -detailed-exitcode`
   against both roots; exit 2 raises an alert. This catches console drift too,
   which Option A never did.
4. **Config equivalence assertion.** Compare resource-type/count/attribute
   fingerprints between the two plans. Options and a recommendation are in
   [[module-patterns]].

Note that the AWS/athenahealth multi-region TFE case study reached the same
conclusion from the other direction: they recommend
["drift detection via CI/CD pipelines"](https://aws.amazon.com/blogs/architecture/validating-multi-region-dr-for-terraform-enterprise-with-aws-fis/)
alongside region-independent configuration sources.

### 6.4 "Adopt Terragrunt and restructure everything" — costed, and rejected

The brief says this is admissible only if argued hard. Here is the argument, and
it does not win.

**What Terragrunt would give:** `generate` blocks that emit `provider.tf` and
`backend.tf` per unit — removing exactly the boilerplate that per-region root
modules create; `terragrunt.stack.hcl`
([Stacks](https://www.gruntwork.io/blog/the-road-to-terragrunt-1-0-stacks))
which generates units from a catalog, so a new region is a few lines of `unit`
blocks; dependency graphs between units with `run --all`; per-unit state by
construction. On paper it is the tool built for this problem.

**What it costs here:**

- **A second configuration language on top of HCL.** Every engineer now needs to
  know both, and every error message arrives through a wrapper.
- **It replaces cookiecutter's job without replacing cookiecutter.** The repo
  already has a working, mature, understood generator. Terragrunt's `generate`
  and cookiecutter overlap almost exactly. Running both is worse than either.
- **Every root module's state address changes.** Migrating a live, mature,
  multi-environment estate means moving every state file and re-initialising
  every backend, with a `terraform state mv`-shaped risk per resource. The
  brief's own bar — no destroy/recreate in `terraform plan` — makes this a
  multi-quarter project with a nonzero chance of an outage caused by the
  migration to the tool that was supposed to prevent outages.
- **It is the wrong sequencing.** The team is mid-flight on DynamoDB Global
  Tables ([[dynamodb-table-naming-migration]]) with everything else not started.
  Spending the next two quarters on a tooling migration delays the actual DR
  work by two quarters, during which the estate has no standby at all.
- **The benefit is marginal against the recommendation.** Terragrunt's headline
  win is DRY per-region roots. Cookiecutter already delivers that here.

**Verdict: no.** Revisit only if (a) the number of root modules passes the point
where generation-plus-CI stops being manageable — realistically 100+ units — or
(b) cross-unit dependency orchestration (`run --all` with a dependency graph)
becomes the bottleneck. Neither is true today.

The same reasoning applies to **Terraform Stacks**, with one difference: Stacks
is the technically cleanest fit (provider `for_each`, per-deployment state,
built-in rollout ordering, and a `deployment` block that maps almost exactly onto
"pair"). It fails on commercials and timing, not design — it requires HCP
Terraform, caps at 500 deployments per Stack, and means adopting `.tfcomponent.hcl`
/`.tfdeploy.hcl` across a repo that currently has neither. **Flag it as the
thing to reassess in 12–18 months**, once the standby regions exist and if the
company is already buying HCP Terraform for other reasons.

---

## 7. Comparison table

| | A: aliases, one root | B: root module per region | C: parameterised stack per region (**recommended**) |
|---|---|---|---|
| State files per pair | 1 | 2 | 2 (+1 small pair stack) |
| Blast radius of a bad apply | **Both regions** | One region | One region |
| Can apply to standby during primary outage | **No** | Yes | Yes |
| Drift between pair possible | No | Yes — must be policed | Yes, but mostly designed out |
| Plan time | 2× | 1× per region, parallelisable | 1× per region, parallelisable |
| Cross-region references | Direct expressions | `terraform_remote_state` / SSM / convention | Same as B |
| Adding the CA pair | Edit a big file | Copy a directory tree | One `cookiecutter` render |
| Per-region least-privilege CI creds | Hard | Natural | Natural |
| New tooling required | None | None | None (already have cookiecutter) |
| Fit with existing repo | Good | Fine | **Native** |

---

## 8. Migration path from today

The estate is live and single-region. Nothing below should produce a
destroy/recreate.

1. **Do not move the primary.** The existing root module *becomes* the primary
   root module, in place. Rename the directory at most; do not restructure state.
   If the current layout is `live/prod-eu/`, add `live/prod-eu/eu-west-2/` beside
   it and leave the original alone until the standby is real.
2. **Extract the reusable module first.** Anything currently inline in the
   primary root module that the standby also needs moves into `modules/`. Use
   `moved` blocks so this is a state-address change with no resource churn —
   `moved` works fine for a resource moving into a module in the same
   configuration; it does **not** work across module packages/source URLs, so
   keep the module local. See [[terraform-gotchas]].
3. **Build the template from the (now-extracted) primary.** Cookiecutter-ise the
   primary root module, re-render it, and assert `git diff --exit-code` produces
   nothing. If re-rendering the primary changes a byte, the template is wrong —
   find out now, not after the standby exists.
4. **Bootstrap the standby's prerequisites out of band** — state bucket, KMS,
   IAM roles. Chicken-and-egg, covered in [[state-management]].
5. **Render the standby root module with `standby_enabled = false`.** It applies
   to almost nothing. Merge it. Now the shape exists in the repo and CI runs a
   plan against it.
6. **Flip `standby_enabled = true` service by service**, in the
   prerequisites-first order the team is already following. Each flip is one
   small PR against one root module with a plan that only touches the standby.
   This is the pattern that makes the whole thing incremental — full code in
   [[module-patterns]].
7. **Create the `pair` stack last**, only when there is something genuinely
   dual-region to put in it (the DynamoDB global table is the likely first
   tenant). Adopt already-existing standby resources with `import` blocks rather
   than recreating them.
8. **Then the CA pair, then the US pair.** EU first — `eu-west-2` has no known
   parity gaps, whereas `ca-west-1` does ([[region-pair-selection]]). Do not
   learn the structure and the region gaps at the same time.

---

## 9. Gotchas specific to this decision

Full list in [[terraform-gotchas]]; these are the ones that come from the
structural choice itself.

- **A module that omits `providers = {}` silently inherits the default provider**
  and builds in the wrong region. There is no error — the apply succeeds and
  creates the resource in the primary. Mitigation: never declare a default
  (unaliased) `aws` provider in a root module that has aliases. If every
  configuration is aliased, an omitted `providers` map is a hard error rather
  than a silent wrong-region build. Under Option B/C there is only one region per
  root module anyway, which is most of why this option is safer.
- **`default_tags` has a documented history of being ignored when an alias is
  set** ([#34545](https://github.com/hashicorp/terraform-provider-aws/issues/34545),
  filed Nov 2023 against provider 4.67.0, now closed;
  [#33172](https://github.com/hashicorp/terraform-provider-aws/issues/33172) is
  the same complaint). It is reported fixed in the 5.x line, but the class of bug
  recurs; assert `tags_all` in a test rather than trusting it.
- **Changing a resource's `region` argument forces replacement.** Under Option A
  with v6-style per-resource regions, a refactor that moves a resource between
  regions destroys it. Under Option B it is a `removed`/`import` pair across two
  states, which is more work but is at least visible.
- **`terraform_remote_state` pointing at the primary's bucket re-creates the
  dependency you removed.** If the standby root module reads the primary's state,
  the standby cannot plan during a primary-region S3 outage. Use SSM Parameter
  Store replicated into the standby, or plain convention-derived names. The
  athenahealth/TFE case study hit exactly this and had to remove it — see
  [[state-management]] §on bootstrap.
- **`count`/`for_each` on a module is fine with `configuration_aliases`**, and
  fails only if the module contains `provider` blocks. If an existing shared
  module has embedded `provider` blocks, fixing that is a prerequisite for
  everything here.

## 10. Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Shape of the region pair in code | Aliases, one root module, one state | Root module per region, state per region | **B, generated from one template (C)** — the standby must be appliable while the primary is down |
| Where genuinely dual-region resources live | In each regional root | In a third small `pair` stack with both aliases | **Third `pair` stack**, kept small and off the failover path |
| Generator | Terragrunt `generate`/stacks | Existing cookiecutter | **Cookiecutter** — already mature, no new tool, no state migration |
| Cross-region value passing | `terraform_remote_state` on primary bucket | SSM Parameter Store replicated to standby / naming convention | **SSM + convention.** `terraform_remote_state` on the primary re-shares fate |
| Provider aliases vs v6 `region` argument | Aliases everywhere | `region` argument everywhere | **Aliases for credential/tag boundaries; `region` only for cross-region plumbing inside the pair stack** |
| Terraform vs OpenTofu | Stay on Terraform | Move for provider `for_each` | **Stay.** Cookiecutter solves the same problem without a migration |
| Terraform Stacks | Adopt now | Defer | **Defer, reassess in 12–18 months** — best technical fit, wrong time, needs HCP Terraform |

## 11. Open questions

- Does the existing cookiecutter template render *root modules*, or only
  modules/services within a root module? The recommendation assumes it can emit a
  root module including `backend.tf`. If it cannot today, that is the first
  piece of work.
- Do any existing shared modules contain their own `provider` blocks? If so they
  cannot be `count`/`for_each`'d and must be refactored to
  `configuration_aliases` first.
- Is generated Terraform currently committed to the repo, or rendered in CI? The
  drift-detection design in §6.3 assumes committed.
- Is HCP Terraform already licensed anywhere in the company? Changes the Stacks
  answer.
- How many root modules exist today? The Terragrunt rejection in §6.4 is
  conditional on that number being well under three figures.
- Which AWS provider major version is the estate pinned to? Everything involving
  the `region` argument requires v6+.

## Sources

- [Providers Within Modules — Terraform docs](https://developer.hashicorp.com/terraform/language/modules/develop/providers) — `providers` map, `configuration_aliases`, the `count`/`for_each` incompatibility and its exact scope.
- [Provider Configuration — Terraform docs](https://developer.hashicorp.com/terraform/language/providers/configuration) — alias semantics and default-configuration inheritance.
- [Using count or for_each in Provider Configuration — HashiCorp support](https://support.hashicorp.com/hc/en-us/articles/6304194229267-Using-count-or-for-each-in-Provider-Configuration) — official statement that dynamic provider configuration is unsupported and why.
- [hashicorp/terraform#27448](https://github.com/hashicorp/terraform/issues/27448) — the long-running provider `for_each` request; `for_each` is a reserved argument name in `provider` blocks.
- [hashicorp/terraform#30461](https://github.com/hashicorp/terraform/issues/30461) — dynamic/optional provider aliases in modules.
- [OpenTofu 1.9.0 release](https://opentofu.org/blog/opentofu-1-9-0/) — provider `for_each` shipped 9 Jan 2025.
- [OpenTofu provider configuration docs](https://opentofu.org/docs/language/providers/configuration/) — exact `for_each` provider syntax and the subset rule for resource `for_each`.
- [Terraform AWS provider 6.0 GA](https://www.hashicorp.com/en/blog/terraform-aws-provider-6-0-now-generally-available) — 18 Jun 2025, enhanced region support, memory rationale, breaking-change warning.
- [Enhanced Region Support guide (provider repo)](https://github.com/hashicorp/terraform-provider-aws/blob/main/website/docs/guides/enhanced-region-support.html.markdown) — `region` is Optional+Computed, **changes force replacement**, `@region` import suffix, list of non-region-aware resources, refresh-only migration steps.
- [Terraform Stacks overview](https://developer.hashicorp.com/terraform/language/stacks) — GA status, `.tfcomponent.hcl`/`.tfdeploy.hcl`, per-deployment state, 500-deployment cap, HCP Terraform requirement.
- [Terraform Stacks use cases](https://developer.hashicorp.com/terraform/language/stacks/use-cases) — the multi-region `provider "aws" "configurations" { for_each = var.regions }` example quoted in §2.3.
- [Multi-Region Terraform Deployments with AWS CodePipeline — AWS DevOps blog](https://aws.amazon.com/blogs/devops/multi-region-terraform-deployments-with-aws-codepipeline-using-terraform-built-ci-cd/) — "each Region as one cell", per-region state keys, isolation claim.
- [Validating multi-Region DR for Terraform Enterprise with AWS FIS — AWS Architecture blog](https://aws.amazon.com/blogs/architecture/validating-multi-region-dr-for-terraform-enterprise-with-aws-fis/) — real active/passive TFE case study; the circular dependency on primary-region state files.
- [Blast Radius in Terraform — Stategraph](https://stategraph.com/blog/terraform-blast-radius) — monolithic state as the primary blast-radius multiplier.
- [Why Is Terraform So Slow? — Stategraph](https://stategraph.com/blog/why-is-terraform-so-slow) — refresh as the dominant cost, which is why doubling resources doubles plan time.
- [Terragrunt: IaC Collaboration at Scale — Gruntwork](https://www.gruntwork.io/blog/terragrunt-iac-collaboration-at-scale) — state segmentation for blast radius, performance and parallelism.
- [The Road to Terragrunt 1.0: Stacks — Gruntwork](https://www.gruntwork.io/blog/the-road-to-terragrunt-1-0-stacks) — `terragrunt.stack.hcl`, `unit` blocks, `values`, generation model.
- [terraform plan command reference](https://developer.hashicorp.com/terraform/cli/commands/plan) — the `-target` "exceptional circumstances" warning.
- [Terraform Provider for AWS Version 6: S3 Cross-Region Replication — mattias.engineer](https://mattias.engineer/blog/2025/aws-provider-version-6/) — practical `region`-argument CRR example and the honest "not substantial" caveat.
- [hashicorp/terraform-provider-aws#34545](https://github.com/hashicorp/terraform-provider-aws/issues/34545) and [#33172](https://github.com/hashicorp/terraform-provider-aws/issues/33172) — `default_tags` ignored when an alias is set.
