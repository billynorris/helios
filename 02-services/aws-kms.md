---
title: AWS KMS — Multi-Region
service: kms
tags: [service, multi-region, kms, encryption, keystone]
status: researched
replication: native (multi-Region keys) — but usually the wrong tool
rpo_achievable: "N/A — keys hold no customer data; key material is synchronous within KMS"
rto_achievable: "< 1 min if the standby-region key exists and consumers reference it by the right ARN"
meets_targets: conditional
updated: 2026-09-16
---

# AWS KMS — Multi-Region

> **This is the keystone note of the vault.** Nearly every other service's
> cross-region story bottlenecks on "…but the key is in the wrong region".
> Read this before [[aws-secrets-manager]], [[aws-ssm-parameter-store]],
> [[amazon-s3]], [[amazon-rds-postgres]], [[amazon-dynamodb]], [[aws-backup]].
> The decision guide lives in a sibling note:
> **[[kms-when-to-use-multi-region-keys]]**.

## TL;DR

- **KMS keys are regional. Always.** A key ARN contains a region and a
  `kms:Decrypt` call goes to the regional endpoint. A resource in `eu-west-2`
  cannot be encrypted with an `eu-west-1` key. Full stop. There is no
  cross-region KMS call in the data path of any AWS-managed encryption.
- **Multi-Region keys (MRKs) do not make a key global.** They make *two
  independent regional keys that happen to share key material and key ID*.
  Different ARNs, separate key policies, separate grants, separate aliases,
  separate quotas, separate CloudTrail. They buy you exactly one thing:
  **ciphertext portability** — bytes encrypted under the primary decrypt under
  the replica.
- **For this project, the default answer is: do NOT use MRKs.** Create an
  ordinary single-region CMK in each standby region and let each AWS service's
  own replication mechanism re-encrypt on the way across. That is what almost
  every AWS service does anyway, whether you hand it an MRK or not. See
  [[kms-when-to-use-multi-region-keys]] for the per-service verdict table.
- **The thing that will bite: `multi_region` is `ForceNew` in Terraform, and
  the AWS API has no conversion operation either.** You have live single-region
  keys in all three primaries today. You cannot upgrade them. Anything you
  decide needs an MRK requires a new key plus a data re-encryption migration,
  per consuming service, with the old key kept alive to read old ciphertext.
- **Second thing that will bite: KMS request quotas differ 5× between your EU
  primary and your EU standby.** `eu-west-1` is 100,000 symmetric crypto ops/s;
  `eu-west-2` is 20,000/s. On failover every pod restarts and calls
  `Decrypt`/`GenerateDataKey` at once into a region with a fifth of the
  headroom. Raise the standby quota *now*, not at 3am.

## Does this service cross regions at all?

KMS is a **regional service** with no global endpoint. Concretely:

- A key ARN is `arn:aws:kms:<region>:<account>:key/<key-id>`. The region is
  structural, not cosmetic.
- Every cryptographic operation is an API call to `kms.<region>.amazonaws.com`.
  Cross-region calls are possible from *your* code (nothing stops an
  `eu-west-2` Lambda calling `kms.eu-west-1.amazonaws.com`) but they are a
  hard dependency on the failed region, so they are useless for DR and you
  should treat them as an anti-pattern in this project.
- **No AWS service will make a cross-region KMS call for you.** An `eu-west-2`
  EBS volume, RDS instance, SQS queue or S3 bucket must be given a key in
  `eu-west-2`.
- **AWS managed keys (`aws/s3`, `aws/secretsmanager`, `aws/ebs`, …) are always
  single-Region keys.** AWS docs state this explicitly: *"AWS managed keys, the
  KMS keys that AWS services create in your account for you, are always
  single-Region keys."* You cannot replicate them and you cannot control their
  policy. If a service is currently using an AWS managed key in the primary,
  the standby will silently get a *different* AWS managed key of the same name.
  For most services that is fine (see the per-service table), but it means you
  have zero control over the key policy during a failover investigation.

### How multi-Region keys actually work

A set of related MRKs is: **one primary key** plus **zero or more replica
keys**, at most one per region, all within the same AWS partition (`aws`,
`aws-cn`, `aws-us-gov` — you cannot replicate across partitions).

What is **shared** (AWS calls these *shared properties*, synchronised from the
primary by KMS itself):

| Shared property | Note |
|---|---|
| Key ID | **Same across all replicas.** The `Region` element of the ARN differs. |
| Key material | Transported across the region boundary inside KMS, never in plaintext. |
| Key material origin (`AWS_KMS` / `EXTERNAL`) | |
| Key spec and encryption algorithms | |
| Key usage (ENCRYPT_DECRYPT / SIGN_VERIFY / GENERATE_VERIFY_MAC) | |
| Automatic key rotation setting | Settable **only on the primary**. |
| On-demand rotation | Initiated **only on the primary**. |
| Primary/replica designation | Changed by `UpdatePrimaryRegion`. |

What is **independent** (AWS synchronises *none* of this — this is the list
that generates production incidents):

- **Key policy.** *"Key policies are not shared properties of multi-Region
  keys. AWS KMS does not copy or synchronize key policies among related
  multi-Region keys."* You write it twice.
- **Grants.** *"AWS KMS grants are Regional. Each grant allows permissions to
  one KMS key… you cannot use a single grant to allow permissions to multiple
  KMS keys, even if they are related multi-Region keys."* Every service that
  creates grants on your behalf (EBS/EC2, RDS, Lambda, DynamoDB, Redshift…)
  creates them **per region, per key**. Grants do not cross regions. This is
  the single most common MRK misconception.
- **Aliases.** An alias is a regional resource. `alias/helios-app` in
  `eu-west-1` and `alias/helios-app` in `eu-west-2` are two unrelated
  resources that you must create separately. In Terraform that means **two
  `aws_kms_alias` resources with different providers**, not one.
- **Tags, description, enabled/disabled state.** You can disable the replica
  while the primary stays enabled, and vice versa.
- **CloudTrail, CloudWatch metrics, quotas, pricing.** Each key in the set is
  billed as its own $1/month key and counts against its own region's quotas.

Critically: *"A replica key is a fully functional KMS key with its own key
policy, grants, alias, tags, and other properties. It is not a copy of or
pointer to the primary key… You can use a replica key even if its primary key
and all related replica keys are disabled."* That last clause is the actual DR
property you are buying: **the replica keeps working when the primary region
is gone.** Once created, a replica depends on its primary only for key
rotation and for `UpdatePrimaryRegion`.

### The ARN trap

This is the thing that trips people constantly, so state it plainly:

```
primary  arn:aws:kms:eu-west-1:111122223333:key/mrk-a1b2c3d4...
replica  arn:aws:kms:eu-west-2:111122223333:key/mrk-a1b2c3d4...
                     ^^^^^^^^^                  ^^^^^^^^^^^^^ same
```

Same key ID (prefixed `mrk-`), **different ARN**. Consequences:

- Any IAM policy, key policy, S3 bucket policy, SQS policy or resource policy
  that names the key by ARN is region-pinned and will not match the replica.
  You need both ARNs, or a wildcard on the region element, or you templatise
  the region.
- A KMS ciphertext blob embeds the key ARN of the key that produced it. The
  AWS Encryption SDK and the AWS SDKs have specific MRK-aware logic that lets a
  ciphertext created under the `eu-west-1` ARN be decrypted by calling the
  `eu-west-2` key — this is the "MRK-aware" discovery behaviour and it is what
  makes the feature useful. Plain `kms:Decrypt` against the replica works too:
  you are calling the regional endpoint with the regional key, and KMS
  recognises the shared key material.
- **`kms:ViaService` condition keys are region-specific** (`s3.eu-west-1.amazonaws.com`).
  Copy-pasting a key policy into the standby without editing the region string
  produces a key that silently denies everything.

## AWS's own guidance: MRKs are an exception, not a default

This matters because the temptation in a project like ours is to say "we're
going multi-region, so make all the keys multi-region". AWS's documentation
argues the opposite, and it's worth having the exact wording to hand when the
question comes up in review.

From the [Multi-Region keys overview](https://docs.aws.amazon.com/kms/latest/developerguide/multi-region-keys-overview.html):

> "For most data security needs, the Regional isolation and fault tolerance of
> Regional resources make standard AWS KMS single-Region keys a best-fit
> solution. However, when you need to encrypt or sign data in client-side
> applications across multiple Regions, multi-Region keys might be the
> solution."

Note what that sentence scopes MRKs to: **client-side applications**. Not
server-side encryption of AWS resources.

> "You are not required to replicate a primary key… However, because
> multi-Region keys have different security properties than single-Region keys,
> we recommend that you create a multi-Region key only when you plan to
> replicate it."

And the crucial one for the per-service table in
[[kms-when-to-use-multi-region-keys]]:

> "Most AWS services that integrate with AWS KMS for encryption at rest or
> digital signatures currently treat multi-Region keys as though they were
> single-Region keys. For example, Amazon S3 cross-Region replication decrypts
> and re-encrypts the data keys used to encrypt object data under the KMS key
> in the destination Region, **even when the KMS key in both Regions is a
> related multi-Region key.**"

Read that twice. For most AWS-service-managed encryption, an MRK gives you
**nothing** over two independent keys — the service re-encrypts regardless. You
pay the isolation cost and get no benefit.

From [Control access to multi-Region keys](https://docs.aws.amazon.com/kms/latest/developerguide/multi-region-keys-auth.html):

> "the security properties of multi-Region keys are significantly different
> from those of single-Region keys"

…and AWS recommends *"using caution when authorizing the creation, management,
and use of multi-Region keys"*, using authorization tools to *"prevent creation
and use of multi-Region keys in any scenario where a single-Region will
suffice."*

### Why AWS says this — the blast radius argument

1. **Key isolation is a security control you are deleting.** With single-region
   keys, an attacker who compromises credentials in `eu-west-2` can decrypt
   `eu-west-2` data only. With an MRK, the same credentials plus the replica's
   key policy decrypt `eu-west-1` ciphertext too. The key material now exists in
   two regions' HSM fleets.
2. **Two key policies, two chances to get it wrong.** Policies are not
   synchronised, so drift is the default, not the exception. A replica created
   by a different team with a looser policy is a real and easy failure mode.
3. **Data residency.** AWS explicitly frames the no-conversion rule as a
   residency guarantee: *"You cannot convert an existing single-Region key to a
   multi-Region key. This design ensures that all data protected with existing
   single-Region keys maintain the same data residency and data sovereignty
   properties."* For the **CA pair**, where the whole reason Canadian data lives
   in `ca-central-1`/`ca-west-1` may be residency, turning a key multi-Region is
   a compliance-relevant act, not a technical one. Loop in whoever owns
   [[data-residency-and-compliance]] before doing it.
4. **Rotation couples the regions.** Automatic rotation can only be enabled on
   the primary, and *"AWS KMS does not encrypt any data with the new key
   material until that key material is available in the primary key and every
   one of its replica keys."* If the replica region is degraded, rotation stalls.

### Enforcement

If you decide "single-region by default, MRK by exception", enforce it rather
than documenting it. The relevant condition keys are:

- `kms:MultiRegion` — `true`/`false`, deny `CreateKey` with `true` unless tagged/approved.
- `kms:MultiRegionKeyType` — `PRIMARY` / `REPLICA`.
- `kms:ReplicaRegion` — restrict which regions `ReplicateKey` may target (pin
  each primary to exactly its DR partner: `eu-west-1 → eu-west-2` only).
- `kms:PrimaryRegion` — restrict where a primary may live.

An SCP using `kms:ReplicaRegion` that only permits the designated partner
region is the cheapest guardrail against "someone replicated the Canadian key
into us-east-1".

## Replication / mirroring options

### Option A — Two independent single-region CMKs (**recommended default**)

Create `aws_kms_key` in the primary and a *separate, unrelated* `aws_kms_key`
in the standby, with the same alias name in each region. Every consuming
service picks up "the key in my region" via the alias.

- **What it guarantees:** full regional isolation, independent blast radius,
  independent rotation, no cross-region control-plane coupling. The standby key
  works even if KMS in the primary is completely unavailable, because there is
  no relationship at all.
- **What it costs you:** ciphertext is *not* portable. Bytes encrypted under
  the `eu-west-1` key cannot be read in `eu-west-2`. This is only a problem if
  *you* are moving raw ciphertext across the boundary yourself. For every
  AWS-managed replication path (S3 CRR, RDS cross-region replicas, DynamoDB
  Global Tables, Secrets Manager replication, AWS Backup copy), AWS
  re-encrypts under the destination key for you, so it is a non-problem.
- **RTO impact:** zero. The key already exists, warm, costing $1/month.

### Option B — Multi-Region key (primary + replica)

- **What it guarantees:** ciphertext portability and one key ID to reason about.
- **What it costs you:** the isolation properties above, plus operational
  coupling on rotation, plus you still write the key policy, alias, grants and
  IAM twice. **Note carefully: option B is not less Terraform than option A.**
  The only line that differs is `aws_kms_replica_key` instead of a second
  `aws_kms_key`. People adopt MRKs expecting a config saving; there isn't one.
- **RTO impact:** zero, same as A.

### Option C — Imported key material (`EXTERNAL` origin) into both regions

Bring your own key material and import the identical bytes into a
single-region key in each region. Gives you ciphertext portability *without*
the MRK relationship, at the price of owning the key material lifecycle, the
expiry, and the re-import on rotation. For MRKs with `EXTERNAL` origin you must
*also* import into each replica individually — so you get the worst of both.

**Only relevant if an external HSM / BYOK mandate already exists.** No evidence
one does here. See [[data-residency-and-compliance]].

### Option D — Do nothing (the standby uses AWS managed keys)

For services where you have not configured a CMK today (a queue using
`aws/sqs`, a bucket using SSE-S3), the standby just gets its own AWS managed
key automatically. Zero work, zero cost, zero control. Perfectly acceptable for
non-sensitive resources; unacceptable anywhere you need to audit or restrict
key usage. Be deliberate about which bucket this is.

**Recommendation: A by default, B by exception with a written justification,
C never unless mandated, D for genuinely low-sensitivity resources.**

## RPO / RTO analysis

KMS stores no customer data, so **RPO is not really a KMS concept**. The
questions that matter are RTO ones:

| Failover step | Time | Pre-provisioned? |
|---|---|---|
| Standby-region key exists and is enabled | 0s | **Yes — must be.** Creating a key at failover time is fine latency-wise (`CreateKey` is seconds) but the *data* encrypted under it doesn't exist yet, so this is meaningless. |
| Key policy in standby grants the standby roles | 0s | **Yes.** A key policy that only names `eu-west-1` role ARNs is a silent, total failure at failover. |
| Grants exist for services that need them | varies | **Mostly automatic**, created when the standby resource is created. If the standby resource is pre-provisioned (it should be, for RTO 15m), its grants already exist. |
| Consumers resolve the right key | 0s if by alias | **Yes.** Reference keys by `alias/…` or by a region-templated ARN, never a hard-coded primary ARN. |
| KMS request quota absorbs the stampede | **This is the risk** | **Yes — raise it in advance.** See below. |
| `UpdatePrimaryRegion` (MRK only) | not on the critical path | **No — do not do this during failover.** |

**Verdict: KMS meets RTO 15m trivially, provided the standby key and its policy
are created ahead of time.** Which is exactly the "prerequisites first" strategy
the team is already following. KMS should be one of the first things done,
because [[aws-secrets-manager]], [[amazon-s3]], [[amazon-rds-postgres]],
[[amazon-dynamodb]], [[aws-backup]] and [[amazon-sqs]] all block on it.

### The failover stampede and request quotas

This is the operationally interesting part and it is specific to your region
pairs. From the [AWS General Reference KMS quotas table](https://docs.aws.amazon.com/general/latest/gr/kms.html),
default **Cryptographic operations (symmetric) request rate**:

| Region | Role in this project | Default symmetric crypto ops/s |
|---|---|---|
| `us-east-1` | US primary | **100,000** |
| `us-west-2` | US standby | **100,000** |
| `eu-west-1` | EU primary | **100,000** |
| `eu-west-2` | **EU standby** | **20,000** |
| `ca-central-1` | CA primary | **10,000** (falls in "each of the other supported Regions") |
| `ca-west-1` | **CA standby** | **10,000** |

**The EU pair has a 5× quota cliff.** You fail over from a region with 100k/s
to one with 20k/s, at the exact moment every pod in the standby cold-starts,
every Secrets Manager cache misses, every Lambda re-decrypts its environment
variables, every EBS volume attaches and every RDS instance opens its
encryption context. That is the peak KMS request rate your system will ever
generate, landing in your lowest-quota region.

The CA pair is symmetric at 10k/s, which is a tenth of the EU/US primaries —
fine if `ca-central-1` traffic is genuinely a tenth, but *verify against
CloudWatch*, don't assume.

Mitigations, in order of effort:

1. **Raise the quota in the standby regions now.** All KMS quotas are adjustable
   except on-demand rotation count and CloudHSM key store rate. Service Quotas
   code for symmetric crypto ops is `L-6E3AF000`. Requesting a quota increase in
   a region you have no traffic in is easy to forget and slow to get approved —
   do it as part of standing the region up, not during the incident. Ask for at
   least parity with the primary.
2. **Measure first.** The `AWS/Usage` namespace publishes
   `CallCount` for KMS by `Resource` (API name); the AWS Security Blog piece
   [Manage your AWS KMS API request rates using Service Quotas and Amazon
   CloudWatch](https://aws.amazon.com/blogs/security/manage-your-aws-kms-api-request-rates-using-service-quotas-and-amazon-cloudwatch/)
   walks through wiring the Service Quotas usage metric to an alarm. Do this in
   the primary to learn your real steady-state, then multiply for cold start.
3. **Data key caching.** If you do client-side envelope encryption, the AWS
   Encryption SDK's caching CMM reuses data keys under explicit security
   thresholds (max age, max messages encrypted, max bytes encrypted), which
   collapses `GenerateDataKey` volume. AWS is pointed about this being a
   deliberate trade-off: *"Data key caching is an optional feature of the AWS
   Encryption SDK that you should use cautiously… In general, use data key
   caching only when it is required to meet your performance goals."* If you
   are hitting quota, it is required. Docs:
   [Data key caching](https://docs.aws.amazon.com/encryption-sdk/latest/developer-guide/data-key-caching.html).
4. **Cache decrypted secrets in-process.** Most of the stampede in a
   microservice fleet is N pods × M secrets all calling `Decrypt` on boot. The
   [AWS Secrets Manager agent / caching libraries](https://docs.aws.amazon.com/secretsmanager/latest/userguide/retrieving-secrets_cache.html)
   turn that into one call per pod per TTL. See [[aws-secrets-manager]].
5. **Stagger pod startup.** A `maxSurge`-limited rollout in the standby is
   cheaper than a quota increase, but conflicts with a 15-minute RTO. Prefer
   the quota increase.

Throttling manifests as `ThrottlingException` from KMS, which most AWS SDKs
retry with backoff — but consuming *services* (EBS attach, RDS start) may
surface it as an outright failure to start, which is much worse at 3am. There is
**no public postmortem I could find** describing a KMS throttle during a
regional failover; treat this as reasoned risk, not documented incident.

### `ca-west-1` specifics

- **KMS is available in `ca-west-1`**, with standard and FIPS endpoints
  (`kms.ca-west-1.amazonaws.com`, `kms-fips.ca-west-1.amazonaws.com`). AWS docs
  state MRKs are *"supported in all AWS Regions that AWS KMS supports"*, so
  `ca-central-1 → ca-west-1` replication is available.
- **Quota is the "other Regions" default of 10,000/s symmetric**, same as
  `ca-central-1`. No cliff, but a low ceiling in absolute terms.
- The real `ca-west-1` risk is not KMS itself but the **services that consume
  KMS being absent from `ca-west-1`**. `ca-west-1` launched later and has a
  thinner service catalogue than `ca-central-1`. Before designing the CA pair,
  check each dependent service against the
  [AWS Regional Services List](https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/)
  — see [[region-pair-selection]]. A KMS key in a region where the service that
  would use it doesn't exist is a key nobody needs.
- Both `ca-central-1` and `ca-west-1` are in the `aws` partition, so MRK
  replication between them is legal (unlike, say, `aws` → `aws-cn`).

## Warm standby shape

What exists in the standby while the primary is healthy:

| Thing | State while idle | Cost |
|---|---|---|
| CMK per purpose (app data, RDS, S3, secrets, logs, EBS) | Created, **enabled** | **$1/key/month** each |
| Aliases | Created, one per region | free |
| Key policy | Full, naming the standby roles | free |
| Grants | Created automatically as standby resources are created | free (50,000 grants/key quota) |
| Rotation | Enabled (or on the primary only, for an MRK) | **+$1/month for the 1st and 2nd rotation only**, capped |
| Request volume | Near zero (standby is idle) | Effectively free; 20,000 requests/month free tier across all regions, then **$0.03 per 10,000 requests** |

Do **not** leave the standby key disabled to "save money" — it saves nothing
($1/month is charged either way) and it is the single most likely reason a
3am failover fails. A disabled key rejects every cryptographic operation.

Rough standing cost: if you have 8 CMKs per region and you mirror all three
pairs, that's 24 new keys = **$24/month**, plus rotation surcharge. This is
noise. KMS is not where your DR budget goes — see [[cost-modelling]].

## Terraform implementation

### Provider aliases (v5 and v6)

On **aws provider v5.x** you must use provider aliases. On **v6.x** most
resources also accept a top-level `region` argument, which is a meaningful
simplification for a cookiecutter monorepo because you can pass region as a
variable rather than plumbing an aliased provider through every module. Both
forms are shown below; pick one and be consistent.

```hcl
# providers.tf — v5/v6 compatible
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.primary_region   # e.g. eu-west-1
}

provider "aws" {
  alias  = "standby"
  region = var.standby_region   # e.g. eu-west-2
}
```

### Option A (recommended): two independent keys, one alias name

This is the shape to put in the cookiecutter template.

```hcl
# modules/kms-pair/variables.tf
variable "name" {
  description = "Logical key name, e.g. \"app-data\". Alias becomes alias/<env>-<name>."
  type        = string
}

variable "env" {
  type = string
}

variable "description" {
  type    = string
  default = null
}

variable "key_admin_role_arns" {
  description = "Roles allowed to administer the key. Must be resolvable in BOTH regions (use the account-level role ARN, which is region-free)."
  type        = list(string)
}

variable "key_user_role_arns" {
  description = "Roles allowed to use the key for cryptographic operations."
  type        = list(string)
}

variable "service_principals" {
  description = "AWS service principals granted use of the key via kms:ViaService, e.g. [\"s3\",\"sqs\"]. Region is templated in per-region."
  type        = list(string)
  default     = []
}

variable "enable_rotation" {
  type    = bool
  default = true
}

variable "rotation_period_in_days" {
  description = "90-2560. Null uses the AWS default of 365."
  type        = number
  default     = null
}

variable "deletion_window_in_days" {
  type    = number
  default = 30
}

variable "standby_enabled" {
  description = "Create the standby-region key. Lets you roll regions out per-env."
  type        = bool
  default     = true
}
```

```hcl
# modules/kms-pair/main.tf
locals {
  alias_name = "alias/${var.env}-${var.name}"
}

data "aws_caller_identity" "current" {}

# Key policy is generated per region so that kms:ViaService carries the right
# region string. This is the #1 copy-paste bug with multi-region KMS.
data "aws_iam_policy_document" "key" {
  for_each = toset(compact([var.primary_region, var.standby_enabled ? var.standby_region : ""]))

  statement {
    sid     = "EnableIAMUserPermissions"
    actions = ["kms:*"]
    resources = ["*"]
    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"]
    }
  }

  statement {
    sid = "KeyAdministration"
    actions = [
      "kms:Create*", "kms:Describe*", "kms:Enable*", "kms:List*", "kms:Put*",
      "kms:Update*", "kms:Revoke*", "kms:Disable*", "kms:Get*", "kms:Delete*",
      "kms:TagResource", "kms:UntagResource", "kms:ScheduleKeyDeletion",
      "kms:CancelKeyDeletion", "kms:RotateKeyOnDemand",
    ]
    resources = ["*"]
    principals {
      type        = "AWS"
      identifiers = var.key_admin_role_arns
    }
  }

  statement {
    sid = "KeyUsage"
    actions = [
      "kms:Encrypt", "kms:Decrypt", "kms:ReEncrypt*",
      "kms:GenerateDataKey*", "kms:DescribeKey",
    ]
    resources = ["*"]
    principals {
      type        = "AWS"
      identifiers = var.key_user_role_arns
    }
  }

  # Grants for AWS services. CreateGrant is what EBS/RDS/Lambda actually need.
  dynamic "statement" {
    for_each = length(var.service_principals) > 0 ? [1] : []
    content {
      sid       = "AllowServiceGrants"
      actions   = ["kms:CreateGrant", "kms:ListGrants", "kms:RevokeGrant"]
      resources = ["*"]
      principals {
        type        = "AWS"
        identifiers = var.key_user_role_arns
      }
      condition {
        test     = "Bool"
        variable = "kms:GrantIsForAWSResource"
        values   = ["true"]
      }
      condition {
        test     = "StringEquals"
        variable = "kms:ViaService"
        # NOTE the region is each.key, not a hard-coded primary.
        values   = [for s in var.service_principals : "${s}.${each.key}.amazonaws.com"]
      }
    }
  }
}

resource "aws_kms_key" "primary" {
  description             = coalesce(var.description, "${var.env} ${var.name}")
  enable_key_rotation     = var.enable_rotation
  rotation_period_in_days = var.rotation_period_in_days
  deletion_window_in_days = var.deletion_window_in_days
  multi_region            = false # explicit. See migration path — this is ForceNew.
  policy                  = data.aws_iam_policy_document.key[var.primary_region].json
  tags                    = { Name = local.alias_name, Env = var.env }
}

resource "aws_kms_alias" "primary" {
  name          = local.alias_name
  target_key_id = aws_kms_key.primary.key_id
}

# --- standby region: a SEPARATE, UNRELATED key with the SAME alias name ---

resource "aws_kms_key" "standby" {
  count                   = var.standby_enabled ? 1 : 0
  provider                = aws.standby
  description             = coalesce(var.description, "${var.env} ${var.name}")
  enable_key_rotation     = var.enable_rotation
  rotation_period_in_days = var.rotation_period_in_days
  deletion_window_in_days = var.deletion_window_in_days
  multi_region            = false
  policy                  = data.aws_iam_policy_document.key[var.standby_region].json
  tags                    = { Name = local.alias_name, Env = var.env }
}

# ALIASES ARE REGIONAL. This second resource is mandatory, not optional.
resource "aws_kms_alias" "standby" {
  count         = var.standby_enabled ? 1 : 0
  provider      = aws.standby
  name          = local.alias_name
  target_key_id = aws_kms_key.standby[0].key_id
}
```

```hcl
# modules/kms-pair/outputs.tf
output "primary_key_arn"   { value = aws_kms_key.primary.arn }
output "primary_key_id"    { value = aws_kms_key.primary.key_id }
output "standby_key_arn"   { value = try(aws_kms_key.standby[0].arn, null) }
output "standby_key_id"    { value = try(aws_kms_key.standby[0].key_id, null) }
output "alias_name"        { value = local.alias_name }

# Consumers should take a map keyed by region so callers can never accidentally
# hand a primary ARN to a standby resource.
output "key_arn_by_region" {
  value = merge(
    { (var.primary_region) = aws_kms_key.primary.arn },
    var.standby_enabled ? { (var.standby_region) = aws_kms_key.standby[0].arn } : {}
  )
}
```

The `key_arn_by_region` output is the important bit for a templated monorepo:
**make it structurally impossible to pass the wrong region's ARN.** Downstream
modules take `kms_key_arn = module.app_key.key_arn_by_region[var.region]`.

### Option B: multi-Region key, when you have justified one

```hcl
resource "aws_kms_key" "mrk_primary" {
  description             = "${var.env} ${var.name} (MRK primary)"
  multi_region            = true          # ForceNew — see migration path
  enable_key_rotation     = true          # rotation is a SHARED property: set it here only
  rotation_period_in_days = 365
  deletion_window_in_days = 30
  policy                  = data.aws_iam_policy_document.key[var.primary_region].json
}

resource "aws_kms_alias" "mrk_primary" {
  name          = local.alias_name
  target_key_id = aws_kms_key.mrk_primary.key_id
}

resource "aws_kms_replica_key" "mrk_replica" {
  provider                = aws.standby
  description             = "${var.env} ${var.name} (MRK replica)"
  primary_key_arn         = aws_kms_key.mrk_primary.arn
  deletion_window_in_days = 30

  # Key policy is NOT inherited. You must set it, with the standby's region
  # baked into any kms:ViaService conditions.
  policy = data.aws_iam_policy_document.key[var.standby_region].json
}

# Still need a second alias. Still regional. Still a separate resource.
resource "aws_kms_alias" "mrk_replica" {
  provider      = aws.standby
  name          = local.alias_name
  target_key_id = aws_kms_replica_key.mrk_replica.key_id
}
```

Notes on `aws_kms_replica_key`:

- `primary_key_arn` is **required** and must point at a multi-Region *primary*
  in a **different** region. You cannot replicate into the primary's own region,
  nor create two replicas of the same primary in one region.
- There is **no `enable_key_rotation` argument** — rotation is inherited from
  the primary. `key_rotation_enabled` is exported as a read-only attribute.
  Trying to manage rotation on the replica is a category error.
- `deletion_window_in_days` (7–30) is per-key. AWS will not delete the primary
  until all replicas are deleted; the primary's waiting period only *starts*
  when the last replica is gone. Budget for that in any teardown plan: worst
  case 30 + 30 = 60 days.
- `bypass_policy_lockout_safety_check` exists and the provider docs warn it
  *"increases the risk that the KMS key becomes unmanageable"*. Don't.
- The provider added `multi_region` on `aws_kms_key` and the
  `aws_kms_replica_key` resource in **v3.64.0**
  ([PR #20533](https://github.com/hashicorp/terraform-provider-aws/pull/20533),
  [issue #19896](https://github.com/hashicorp/terraform-provider-aws/issues/19896)).
  Anything on v5/v6 has it.
- There is a standing request for a single `aws_kms_multi_region_key` resource
  that manages primary and replicas together
  ([issue #22243](https://github.com/hashicorp/terraform-provider-aws/issues/22243));
  it has not been implemented. You will write two resources with two providers.

### Fitting a cookiecutter monorepo

- Put `kms-pair` in the shared modules directory and instantiate it **once per
  logical key purpose**, not once per consuming resource. A key per broad data
  class (app-data, database, secrets, logs, backups) is the right granularity —
  fine enough to revoke independently, coarse enough that the ARN plumbing
  stays sane.
- Because the standby key is gated on `standby_enabled`, you can roll the
  second region out environment by environment (dev → staging → prod) from the
  same template. That matters: KMS is the prerequisite for nearly everything
  else, so it wants to land first and land quietly.
- **Never hard-code a key ARN in a downstream module.** Take
  `key_arn_by_region` or resolve `alias/${env}-${name}` with a
  `data "aws_kms_alias"` in the consuming region.

## Migration path from single-region

**This is the most important section of the note for this company, because you
are live in three regions today with single-region keys everywhere.**

### The blocker, stated precisely

Two independent facts, both hard:

1. **The AWS API has no conversion operation.** *"You cannot convert a
   single-Region key to multi-Region key or a convert a multi-Region key to a
   single-Region key. To move existing workloads into multi-Region scenarios,
   you must re-encrypt your data or create new signatures with new multi-Region
   keys."* There is no support ticket that fixes this; it is a deliberate data
   residency guarantee.
2. **`multi_region` is `ForceNew` in the Terraform provider.** From
   `internal/service/kms/key.go`:

   ```go
   "multi_region": {
       Type:     schema.TypeBool,
       Optional: true,
       Computed: true,
       ForceNew: true,
   },
   ```

   So flipping `multi_region = false` → `true` on an existing `aws_kms_key`
   produces `# aws_kms_key.this must be replaced`. Terraform will happily do it:
   create a new key, schedule the old one for deletion, and update the alias.
   **Your plan will look clean and your data will become unreadable in 30
   days.** This is the single most dangerous plan output in this whole
   migration. Anyone reviewing KMS plans should treat
   `# forces replacement` on a KMS key as a stop-the-line event.

**Guardrail to add today**, before anyone touches KMS in the repo:

```hcl
resource "aws_kms_key" "primary" {
  # ...
  lifecycle {
    prevent_destroy = true
  }
}
```

`prevent_destroy` turns the silent catastrophe into a hard plan error. It is
three lines and it is the highest-value change in this note. Put it in the
cookiecutter template so every new key gets it by default.

### The good news

**For almost everything, you do not need to migrate at all.** Because most AWS
services re-encrypt at the region boundary (see
[[kms-when-to-use-multi-region-keys]]), the correct target state for existing
single-region keys is: *leave them alone, and create a brand-new, unrelated
single-region key in the standby region.* No re-encryption, no downtime, no
ForceNew. The migration is purely additive.

The only things that need a genuine key migration are cases where you have
ciphertext **you** move across the region boundary yourself, outside an AWS
replication mechanism.

### If you do need to migrate to a new key: the general shape

The invariant: **the old key must stay alive, enabled, and readable for as long
as any ciphertext under it exists.** Deleting a KMS key destroys the data it
protects, irreversibly, and the only warning you get is a 7–30 day window.

1. Create the new key (MRK primary + replica) alongside the old one. New alias,
   e.g. `alias/prod-app-data-mrk`. Do **not** repoint the existing alias yet.
2. Grant both keys to the workload. Decrypt permission on old, encrypt+decrypt
   on new.
3. **Flip writes to the new key.** New data is encrypted under the new key.
4. **Backfill**: re-encrypt existing data (mechanism below, per service).
5. Verify nothing references the old key: CloudTrail `Decrypt` events with the
   old key ARN over a full business cycle (including monthly/quarterly batch
   jobs — this is where people get caught). `kms:LastUsedDate` via
   `DescribeKey`/IAM Access Advisor style tooling helps, but CloudTrail is
   authoritative.
6. Repoint the alias, then disable the old key (reversible) and wait. Only
   after a long soak — a quarter, not a week — `ScheduleKeyDeletion` with a
   30-day window.

### Re-encryption, per consuming service

| Service | How you re-encrypt under a new key | Downtime | Notes |
|---|---|---|---|
| **S3** (SSE-KMS) | `aws s3 cp s3://b/ s3://b/ --recursive --sse aws:kms --sse-kms-key-id <new>` copies objects in place, or use **S3 Batch Operations "Copy"** with the new key for scale. | None | In-place copy rewrites the object; versioning means you now store both versions — budget the storage and set a lifecycle rule to expire noncurrent versions. Object metadata/ACLs need `--metadata-directive COPY`. Batch Operations is the only sane option above ~millions of objects. See [[amazon-s3]]. |
| **EBS** | Snapshot → `CopySnapshot` with `KmsKeyId = <new>` → create volume from the copy → stop instance, detach, attach, start. | **Yes** — instance stop/start per volume. | There is no in-place EBS key change. For EKS nodes this is irrelevant: cycle the node group with a new launch template whose block device mapping names the new key. That's a rolling replacement, no downtime. See [[amazon-eks]]. |
| **RDS / Aurora** | Snapshot → `CopyDBSnapshot`/`CopyDBClusterSnapshot` with `KmsKeyId = <new>` → **restore to a new instance/cluster** → cut over. | **Yes** — a cutover. | You cannot change the KMS key of a live RDS instance or Aurora cluster. This is the expensive one. Plan it as a DB migration, not a key rotation. See [[amazon-rds-postgres]]. |
| **DynamoDB** | `UpdateTable` with a new `SSESpecification.KMSMasterKeyId`. | **None** — online. | DynamoDB genuinely supports changing the CMK in place. Table stays available; re-encryption happens in the background. One of the few easy ones. See [[amazon-dynamodb]]. |
| **Secrets Manager** | `UpdateSecret` with the new `KmsKeyId`, then `PutSecretValue` to create a new version. **Old versions stay encrypted under the old key.** | None | You must rotate the value, not just the key reference, or the current version stays on the old key. See [[aws-secrets-manager]]. |
| **SSM Parameter Store (SecureString)** | `PutParameter --overwrite --type SecureString --key-id <new>`. | None | Trivially scriptable. The parameter history keeps old versions under the old key. See [[aws-ssm-parameter-store]]. |
| **SQS / SNS** | Set `KmsMasterKeyId` on the queue/topic. | None | Only affects messages encrypted *after* the change. In-flight messages under the old key are unreadable if you delete the old key — but with a 14-day max retention, a two-week soak fully drains them. Easy. See [[amazon-sqs]], [[amazon-sns]]. |
| **Lambda env vars** | Update `KMSKeyArn` on the function config and re-deploy. | None (brief config update) | Env vars are re-encrypted by Lambda on update. See [[aws-lambda]]. |
| **ECR** | **Not changeable.** Repository encryption config is set at creation. | — | You must create a new repository with the new key and re-push images. For the standby region you are pushing to a new repo anyway. See [[amazon-ecr]]. |
| **CloudWatch Logs** | `AssociateKmsKey` on the log group. | None | Applies to **newly ingested** data only. Existing log events stay under the old key; they become unreadable if you delete it. Either keep the old key until retention expires, or accept the loss. Retention is usually short enough that this self-resolves. See [[cloudwatch-observability]]. |
| **EFS** | **Not changeable.** Set at file system creation. | — | Requires a new file system + DataSync/rsync copy. See [[amazon-efs]]. |
| **AWS Backup** | Vault encryption key is set at vault creation and **cannot be changed**. Copy jobs re-encrypt into the destination vault's key. | — | New vault, new copies. See [[aws-backup]]. |
| **EKS secrets envelope encryption** | Key is set at cluster creation; changing it requires `AssociateEncryptionConfig` (one-way, cannot be removed) or a new cluster. | — | See [[amazon-eks]]. |

Read the "Not changeable" rows carefully — **ECR, EFS, AWS Backup vaults and
RDS are the services where "we'll change the key later" is not available.**
Decide the key for those at creation time in the standby region, because
you only get one shot per resource.

### Adopting an existing key into the new module

If you refactor keys into a shared `kms-pair` module, use `moved` blocks (or
`import` blocks on TF ≥ 1.5) rather than `terraform state mv` by hand:

```hcl
moved {
  from = aws_kms_key.app_data
  to   = module.app_data_key.aws_kms_key.primary
}

moved {
  from = aws_kms_alias.app_data
  to   = module.app_data_key.aws_kms_alias.primary
}
```

Then run `terraform plan` and confirm it reports **0 to add, 0 to destroy**. If
it reports a destroy on the key, stop. See [[terraform-repo-structure]].

## Failover procedure

For Option A (independent keys), **there is no KMS failover step at all.** That
is the point. The standby key already exists, already has its policy, and
already holds grants for the standby resources. Promotion of the application
does not touch KMS.

For Option B (MRK), also nothing on the critical path:

- **Do not run `UpdatePrimaryRegion` during a failover.** It is not required —
  a replica key is fully functional for all cryptographic operations regardless
  of where the primary is. It exists so you can move the *management* plane
  (rotation control) after the dust settles. It is rate-limited to 5 req/s and
  is a control-plane operation subject to eventual consistency. Running it
  under pressure adds risk and buys nothing. Put it in the **failback/steady-state**
  runbook, not the failover one.
- The one real check: **is the replica key enabled?** Add it to the failover
  pre-flight script (`aws kms describe-key --key-id alias/prod-app-data --region eu-west-2`
  → `KeyState == "Enabled"`).

Pre-flight checklist for [[failover-runbook]]:

```bash
# Every key the standby needs, enabled and resolvable by alias.
for a in app-data database secrets logs backups; do
  aws kms describe-key --region "$STANDBY" --key-id "alias/${ENV}-${a}" \
    --query 'KeyMetadata.[KeyState,MultiRegion,Arn]' --output text
done
# Expect: Enabled <bool> arn:aws:kms:<STANDBY>:...
```

## Failback

Also nothing, for Option A. The primary key was never touched.

For Option B there is one genuine failback consideration: if you *did* run
`UpdatePrimaryRegion` to move the primary to the standby, you must run it again
to move it back, and while the primary lives in the DR region, rotation is
controlled from there. If you never run it, rotation control stays in a region
that may still be down — which stalls automatic rotation (KMS will not encrypt
under new material until it is present in every replica) but does **not** stop
cryptographic operations. Since RPO/RTO are unaffected either way, the lazy
answer is right: **leave the primary where it is, fix it in business hours.**

The harder failback question is not KMS's: it is that any *new* data written in
the standby during the outage is encrypted under the standby key, and needs to
get back to the primary. That is a per-service replication question — see
[[failback-strategy]].

## Gotchas

1. **`multi_region = true` on an existing key is `ForceNew`.** Terraform will
   replace the key without complaint. Add `prevent_destroy = true` to every KMS
   key resource in the repo today.
2. **Aliases are regional and are a separate resource per region.** People
   create the replica key and forget the replica alias, then every consumer that
   resolves `alias/prod-app-data` in the standby gets `NotFoundException` at
   failover. There is no cross-region alias.
3. **Grants do not cross regions.** *"AWS KMS grants are Regional… you cannot
   use a single grant to allow permissions to multiple KMS keys, even if they
   are related multi-Region keys."* Every service-created grant exists only in
   the region where the resource lives. This also means: a service that needs
   `kms:CreateGrant` needs it granted in **both** key policies.
4. **Key policies are not synchronised between MRK primary and replica.** They
   drift. Generate both from one Terraform data source, and put a Config rule or
   a drift check on them.
5. **`kms:ViaService` condition values embed the region.** `s3.eu-west-1.amazonaws.com`
   in a key policy attached to a `eu-west-2` key denies everything, silently,
   with an unhelpful `AccessDeniedException`.
6. **AWS managed keys are always single-region and cannot be replicated.**
   Anything relying on `aws/<service>` gets a different key in the standby with
   a policy you do not control.
7. **Most AWS services re-encrypt at the region boundary anyway**, so an MRK
   usually buys nothing — AWS calls this out for S3 CRR by name. Check
   [[kms-when-to-use-multi-region-keys]] before assuming an MRK helps.
8. **The EU standby has 1/5th the KMS crypto-op quota of the EU primary**
   (20,000/s vs 100,000/s). Raise it before you need it.
9. **`CreateKey` and `ReplicateKey` are 5 req/s.** Bootstrapping a large
   templated estate into a new region can throttle the *apply*. Not fatal
   (Terraform retries), but it makes the first standby apply slow and noisy.
10. **Deleting a multi-Region primary is blocked until every replica is
    deleted**, and the primary's deletion waiting period only starts after the
    last replica is gone. A full teardown is a 60-day operation, not a 30-day
    one.
11. **`EXTERNAL` origin MRKs do not replicate key material.** KMS copies the key
    material *identifier* to the replica; you import the bytes into each replica
    yourself. If anyone proposes BYOK plus MRK, they are signing up for a manual
    per-region import on every rotation.
12. **MRKs cannot be created in a custom key store** (CloudHSM or external key
    store). If a compliance requirement ever pushes you to a custom key store,
    MRKs are off the table entirely.
13. **`GetParametersForImport` is 1 req/s.** Irrelevant unless you go BYOK, in
    which case it is a genuine bootstrap constraint.
14. **Rotation is billed.** The first and second rotations each add $1/month per
    key, capped after the second. With 24+ keys across three pairs that's real
    but still small money; just don't be surprised by it.

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Default key topology | Two independent single-region CMKs per pair, same alias name | Multi-Region key (primary + replica) | **A.** AWS's own guidance, full regional isolation, and identical Terraform effort. MRKs by exception only, justified in writing, per [[kms-when-to-use-multi-region-keys]]. |
| Key granularity | One CMK per logical data class (app-data, db, secrets, logs, backups) | One CMK per resource | **A (per data class).** Per-resource keys multiply the ARN plumbing and the $1/month with no practical isolation gain at this scale. |
| Key reference style | Resolve `alias/<env>-<name>` in-region | Pass ARN from a module output | **Pass a `key_arn_by_region` map.** Aliases are great for humans and for out-of-band scripts; explicit ARNs are better in Terraform because a wrong-region ARN becomes a plan error rather than a runtime one. Create the aliases regardless. |
| Enforce single-region-by-default | Document it | SCP on `kms:MultiRegion` / `kms:ReplicaRegion` | **SCP.** Documentation does not survive contact with a hurried engineer. Pin `kms:ReplicaRegion` to the designated DR partner so a Canadian key can never land in `us-east-1`. |
| Standby quota | Request increase at failover | Request increase now | **Now.** Quota increases are a support-ticket turnaround; a failover is 15 minutes. `eu-west-2` is the urgent one. |
| Existing single-region keys | Migrate to MRKs | Leave them, add new standby keys | **Leave them.** The migration is expensive (RDS/ECR/EFS need resource recreation) and buys nothing for AWS-managed replication. |
| `prevent_destroy` on keys | Rely on review | Set it in the module | **Set it in the module.** Non-negotiable. |

## Cost

| Item | Price | Source |
|---|---|---|
| Customer managed key | **$1/month**, prorated hourly, per key — **including each MRK replica** | [KMS pricing](https://aws.amazon.com/kms/pricing/) |
| 1st and 2nd key rotation | **+$1/month each**, capped at the second rotation | same |
| Requests | **$0.03 per 10,000 requests** | same |
| Free tier | **20,000 requests/month across all Regions** | same |
| AWS managed / AWS owned keys | no storage charge | same |

Standing standby cost is essentially `number_of_keys × $1/month`. For 8 keys
across 3 standby regions: **~$24/month plus rotation surcharge**. Request
charges in an idle standby are near zero.

**Cost levers (all small):** fewer, coarser keys; use AWS managed keys for
low-sensitivity resources; data key caching if request volume ever becomes
material (it won't at these rates unless you're doing per-item client-side
encryption at high throughput). **KMS is not a place to optimise.** Spend the
effort on [[amazon-eks]] and [[amazon-rds-postgres]], which is where the standby
money actually goes — see [[cost-modelling]].

## Open questions

- Which CMKs exist today, per region, per environment? A full inventory
  (`aws kms list-keys` + `list-aliases` + tags across all three regions and all
  envs) is the actual first task. Nothing in this note can be planned without it.
- Is any data currently encrypted client-side (AWS Encryption SDK, DynamoDB
  Encryption Client, S3 client-side encryption)? **This is the question that
  decides whether you need any MRKs at all.** If the answer is "no, everything
  is server-side AWS-managed encryption", the answer to
  [[kms-when-to-use-multi-region-keys]] is "none of them".
- Are there signing keys (asymmetric SIGN_VERIFY) in use — JWT signing, artifact
  signing, webhook signatures? If a verifier outside AWS pins a public key, an
  MRK is the right call.
- Does the Canadian deployment have a contractual or regulatory data residency
  obligation that a multi-Region key would breach? Owner: whoever signs off
  [[data-residency-and-compliance]].
- What is the current steady-state KMS request rate per region (`AWS/Usage`
  `CallCount`)? Needed to size the standby quota request.
- Are any keys currently `EXTERNAL` origin / BYOK, or in a custom key store?
  Either forecloses MRKs.
- Who owns KMS key policies today — a platform team, or each service team? This
  determines whether "two policies, not synchronised" is a manageable risk or a
  guaranteed drift source.

## Sources

- [Multi-Region keys in AWS KMS](https://docs.aws.amazon.com/kms/latest/developerguide/multi-region-keys-overview.html) — the primary source. Shared vs independent properties, the "single-Region keys are a best-fit solution for most needs" guidance, the no-conversion rule, and the explicit statement that most AWS services re-encrypt at the boundary even with related MRKs.
- [Control access to multi-Region keys](https://docs.aws.amazon.com/kms/latest/developerguide/multi-region-keys-auth.html) — "security properties… significantly different", grants are regional, key policies are not synchronised, and the `kms:MultiRegion` / `kms:ReplicaRegion` / `kms:PrimaryRegion` / `kms:MultiRegionKeyType` condition keys.
- [Rotate AWS KMS keys](https://docs.aws.amazon.com/kms/latest/developerguide/rotate-keys.html) — rotation is a shared property, enabled only on the primary; new material is not used until present in every replica; `EXTERNAL` origin requires per-replica import.
- [AWS KMS quotas](https://docs.aws.amazon.com/kms/latest/developerguide/limits.html) — all quotas adjustable except on-demand rotation count and CloudHSM key store rate; calculated per region per account.
- [AWS KMS endpoints and quotas (General Reference)](https://docs.aws.amazon.com/general/latest/gr/kms.html) — **the per-region symmetric crypto-op quota table** (us-east-1/us-west-2/eu-west-1 = 100,000/s; eu-west-2 = 20,000/s; others incl. ca-central-1 and ca-west-1 = 10,000/s), the management API rates (`CreateKey` 5/s, `ReplicateKey` 5/s, `CreateGrant` 50/s, `UpdatePrimaryRegion` 5/s), and confirmation that `ca-west-1` has KMS endpoints.
- [AWS KMS increases default service quotas for cryptographic operations (Jul 2024)](https://aws.amazon.com/about-aws/whats-new/2024/07/aws-kms-increases-default-service-quotas-cryptographic-operations) — the 50,000 → 100,000 change in us-east-1/us-west-2/eu-west-1 and 500 → 1,000 for RSA/ECC everywhere. Explains why older blog posts quote different numbers.
- [AWS KMS pricing](https://aws.amazon.com/kms/pricing/) — $1/key/month including replicas, $0.03/10,000 requests, 20,000 free requests/month, rotation surcharge capped at the second rotation.
- [Manage your AWS KMS API request rates using Service Quotas and Amazon CloudWatch](https://aws.amazon.com/blogs/security/manage-your-aws-kms-api-request-rates-using-service-quotas-and-amazon-cloudwatch/) — AWS Security Blog; how to measure current request rate and alarm before throttling. The prerequisite for a sensible quota-increase request.
- [Data key caching (AWS Encryption SDK)](https://docs.aws.amazon.com/encryption-sdk/latest/developer-guide/data-key-caching.html) — the caching CMM and security thresholds; AWS's "use cautiously" framing.
- [AWS Encryption SDK: How to Decide if Data Key Caching is Right for Your Application](https://aws.amazon.com/blogs/security/aws-encryption-sdk-how-to-decide-if-data-key-caching-is-right-for-your-application/) — the trade-off discussion AWS points to from the docs above.
- [`aws_kms_key` resource docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/kms_key) — `multi_region` defaults to `false`; deletion waiting period for an MRK primary starts only when the last replica is deleted; warning against mixing inline `policy` with `aws_kms_key_policy`.
- [`aws_kms_replica_key` resource docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/kms_replica_key) — `primary_key_arn` required and must be in another region; one replica per region; no rotation argument; `bypass_policy_lockout_safety_check` warning.
- [terraform-provider-aws `internal/service/kms/key.go`](https://github.com/hashicorp/terraform-provider-aws/blob/main/internal/service/kms/key.go) — the `ForceNew: true` on `multi_region`, read directly from provider source. This is the migration blocker in code.
- [terraform-provider-aws issue #19896 — Support for KMS Multi-Region Keys](https://github.com/hashicorp/terraform-provider-aws/issues/19896) and [PR #20533](https://github.com/hashicorp/terraform-provider-aws/pull/20533) — when and how MRK support landed (v3.64.0).
- [terraform-provider-aws issue #22243 — New resource: `aws_kms_multi_region_key`](https://github.com/hashicorp/terraform-provider-aws/issues/22243) — the still-open request for a combined resource; confirms you must write primary and replica separately.
- [The AWS Canada West (Calgary) Region is now available](https://aws.amazon.com/blogs/aws/the-aws-canada-west-calgary-region-is-now-available/) — launch post for `ca-west-1`; useful for the service-availability caveat in [[region-pair-selection]].
