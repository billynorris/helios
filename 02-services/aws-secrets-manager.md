---
title: AWS Secrets Manager — Multi-Region
service: secrets-manager
tags: [service, multi-region, secrets-manager, encryption]
status: researched
replication: native (replica secrets)
rpo_achievable: "seconds — async replication, well inside 2h"
rto_achievable: "< 1 min for reads; promotion is optional and takes seconds"
meets_targets: yes
updated: 2026-09-16
---

# AWS Secrets Manager — Multi-Region

> **The team has already done this.** Replication is configured and working.
> This note therefore validates the existing choice, then goes after the
> questions that only show up in an actual failover: *can you still rotate a
> secret when the primary region is gone? What exactly does promotion do? What
> happens to the rotation Lambda? Do resource policies come along?*

## TL;DR

- **Replication is native, correct, and the right choice.** `ReplicateSecretToRegion`
  gives you a replica in the standby with the same name, the same value, and an
  ARN that differs only in the region element. RPO is seconds; RTO for reads is
  zero. Nothing here threatens the 2h/15m targets.
- **Replicas are read-only.** AWS's wording: *"A replica secret can't be updated
  independently from its primary secret, **except for its encryption key**."*
  All writes — `PutSecretValue`, `UpdateSecret`, rotation — go to the primary.
- **So: if the primary region is gone, you cannot rotate or update the secret
  until you promote the replica.** `StopReplicationToReplica`, called *from the
  replica region*, severs the link and makes the replica a standalone,
  fully-writable secret. It takes seconds. **It is one-way** — there is no
  "re-attach" API — so promotion is a failback problem you inherit, not a free
  action.
- **The key insight most people miss: you almost never need to promote during a
  failover.** Reads work on a replica. If your standby workload only needs to
  *read* credentials, you fail over, serve traffic, and leave promotion for
  business hours. Promote only when you need to *write* — which in practice
  means only when a rotation is due or a credential must be changed during the
  outage. Put this decision in the runbook explicitly so nobody promotes 400
  secrets reflexively at 3am and creates a failback nightmare.
- **The thing that will bite: rotation and replication are not transactional.**
  Rotation changes the primary, then replication propagates. For a brief window
  the replica serves the *old* value. A standby app that reads the replica and
  connects to a database whose password has just rotated gets an auth failure.
  Mitigation below.

## Does this service cross regions at all?

Yes, natively, and it is one of the few services in this vault where the answer
is a clean yes.

- A secret is regional, but Secrets Manager will maintain **replica secrets** in
  other regions for you. You declare the replica regions on the primary secret.
- The replica's ARN is *"the same as the primary secret except for the Region"* —
  including the random 6-character suffix. So
  `arn:aws:secretsmanager:eu-west-1:1234:secret:prod/db-a1b2c3` becomes
  `arn:aws:secretsmanager:eu-west-2:1234:secret:prod/db-a1b2c3`. **Same suffix.**
  That is genuinely useful: code that resolves the secret *by name* works
  unchanged in either region, and even ARN-templating only needs the region
  substituted.
- Secrets Manager replicates *"the encrypted secret data and metadata such as
  tags and resource policies"*. Resource policies come along — see the caveat
  below.
- **Partition boundaries are hard.** You cannot replicate between commercial
  regions and GovCloud/China. Irrelevant for all three of your pairs (EU, US, CA
  are all in the `aws` partition and all in-partition pairs), but worth knowing.
- **You must enable the destination region on the account first.** `eu-west-2`,
  `us-west-2` and `ca-west-1` are all enabled-by-default regions except
  `ca-west-1`, which — like other post-2019 regions — **is opt-in and must be
  explicitly enabled**. If the CA pair hasn't been stood up yet, that is the
  first step and it is an account-level action, not a Terraform one. See
  [[region-pair-selection]].
- `ca-west-1` has full Secrets Manager endpoints including FIPS
  (`secretsmanager.ca-west-1.amazonaws.com`,
  `secretsmanager-fips.ca-west-1.amazonaws.com`). Secrets Manager is not one of
  `ca-west-1`'s service gaps.

### How the replica is encrypted — validating the KMS story

This is where [[aws-kms]] and this note meet, and the behaviour is exactly right.

The replica is **re-encrypted under a key in the replica region**. From the
console flow: *"(Optional) For Encryption key, choose a KMS key to encrypt the
secret with. **The key must be in the replica Region.**"*

If you don't specify one, the CLI docs are explicit: *"The replica is encrypted
with the AWS managed key `aws/secretsmanager`"* — meaning the **replica
region's** `aws/secretsmanager` key, which is a different key from the primary
region's, because AWS managed keys are always single-Region keys.

Consequences worth being precise about:

1. **You do not need a multi-Region KMS key here, and never did.** See
   [[kms-when-to-use-multi-region-keys]]. The plaintext is decrypted inside
   Secrets Manager in the primary and re-encrypted under the replica's key. No
   ciphertext crosses the boundary.
2. **Defaulting to `aws/secretsmanager` "just works" with zero configuration** —
   which is why replication is so easy to turn on and why nobody notices the KMS
   question. That is fine for many secrets.
3. **But if you use a CMK in the primary and let the replica default to
   `aws/secretsmanager`, you have silently created two different security
   postures for the same secret.** The primary's key policy restricts who can
   decrypt; the AWS managed key in the replica does not, beyond IAM. If the CMK
   exists because of a compliance requirement, the replica breaches it.
   **Audit this.** It is the single most likely defect in an
   already-done-and-working replication setup:

   ```bash
   # For every secret, print the KMS key used in each region.
   for r in "$PRIMARY" "$STANDBY"; do
     aws secretsmanager list-secrets --region "$r" \
       --query 'SecretList[].[Name,KmsKeyId]' --output text | sed "s|^|$r\t|"
   done
   ```

   Any row where the standby shows empty/`aws/secretsmanager` while the primary
   shows a CMK ARN is a finding.
4. **The standby CMK's key policy must grant decrypt to the standby workload's
   roles.** A replica that exists but cannot be decrypted by the standby node
   role is the classic silent failure — it looks perfect in
   `describe-secret` and fails at `GetSecretValue`. See [[aws-kms]] gotcha #5.

## Replicas are read-only — what that actually means

AWS's statement, from the promotion docs:

> "A replica secret can't be updated independently from its primary secret,
> except for its encryption key."

Concretely:

| Operation against a replica | Works? |
|---|---|
| `GetSecretValue` | **Yes** |
| `DescribeSecret` | Yes |
| `BatchGetSecretValue` | Yes |
| `PutSecretValue` | **No** |
| `UpdateSecret` (value) | **No** |
| `UpdateSecret` (`KmsKeyId`) | **Yes** — the documented exception |
| `RotateSecret` | **No** |
| `TagResource` | Tags replicate from the primary |
| `PutResourcePolicy` | Policies replicate from the primary |
| `StopReplicationToReplica` | **Yes** — this is the escape hatch |

### The question that matters: primary region is gone. Now what?

**Reads still work.** The replica is a fully independent copy in a healthy
region with its own KMS key. A standby workload that reads
`GetSecretValue` on boot and caches gets everything it needs, immediately, with
no action from you. This is why Secrets Manager is comfortably inside RTO 15m.

**Writes do not.** You cannot rotate, you cannot change a password, you cannot
add a new secret version. If your outage lasts long enough for a scheduled
rotation to fire, that rotation simply doesn't happen.

Whether this matters depends entirely on how long the outage lasts and what
your rotation cadence is:

| Outage duration | Rotation cadence | Do you need to promote? |
|---|---|---|
| Hours | 30 or 90 days | **No.** Ride it out. Rotation resumes when the primary returns. |
| Days | 30 days | Probably not, but watch for a rotation window inside the outage. |
| Days–weeks | 7 days or shorter | **Yes** — promote, or you will miss rotations and may breach a control. |
| Any | You need to change a credential *because of* the incident (suspected compromise, emergency password change) | **Yes, immediately.** |

**Recommendation for the runbook: do not promote by default.** Promotion is
one-way and creates the failback problem described below. Make it a conscious,
named decision with a trigger ("we have promoted to standalone because X") and
a default of "no".

### `StopReplicationToReplica` — what promotion actually does

> "Promoting a replica secret disconnects the replica secret from the primary
> secret and makes the replica secret a standalone secret. Changes to the
> primary secret won't replicate to the standalone secret."

> "You might want to promote a replica secret to a standalone secret as a
> disaster recovery solution if the primary secret becomes unavailable. Or you
> might want to promote a replica to a standalone secret if you want to turn on
> rotation for the replica."

> "If you promote a replica, be sure to update the corresponding applications to
> use the standalone secret."

Operational facts:

- **You must call it from the replica region.** `aws secretsmanager
  stop-replication-to-replica --secret-id MySecret --region eu-west-2`.
- It is fast — a control-plane call, not a data operation.
- It is subject to a **50 TPS combined quota** shared with `PutSecretValue`,
  `UpdateSecret`, `ReplicateSecretToRegion`, `RemoveRegionsFromReplication` and
  `UpdateSecretVersionStage`, **and that quota is not adjustable.** Promoting
  1,000 secrets takes ≥20 seconds of sustained API calls; 10,000 takes ≥200
  seconds. Still inside a 15-minute RTO, but not instant — script it with
  concurrency and backoff rather than a serial `for` loop, and know your secret
  count. See the promotion script below.
- **There is no inverse API.** You cannot re-link a promoted standalone secret
  to its old primary. Failback means re-establishing replication from whichever
  side you decide is now authoritative, which may mean deleting one of them.
- Promotion is CloudTrail-logged, which is how you'll reconstruct what happened.
- The promoted secret **keeps its own KMS key** (the replica's), so no
  re-encryption is involved.

**Important subtlety:** the primary still thinks it has a replica. When the
primary region comes back, the primary secret's replication status for that
region will be stale/failed. You clean that up with
`RemoveRegionsFromReplication` on the primary. In Terraform terms, the `replica`
block in your config no longer matches reality and the next apply will try to
re-create the replication — **against a region that now contains a standalone
secret with the same name.** That collision is what
`force_overwrite_replica_secret = true` exists for, and it does exactly what it
sounds like: overwrites the standalone secret. Think hard before setting it
permanently in the module; it is a footgun that silently destroys the very
secret you promoted to save yourself.

## Rotation and replication

### Do rotation Lambdas need to exist in both regions?

**No — and you should not create them in both.**

- Rotation is configured on the **primary secret only**. The rotation Lambda is
  invoked by Secrets Manager in the primary region, against the primary secret.
- *"If you turn on rotation for your primary secret, Secrets Manager rotates the
  secret in the primary Region, and the new secret value propagates to all of
  the associated replica secrets. You don't have to manage rotation individually
  for all of the replica secrets."*
- A rotation Lambda in the standby would have nothing to rotate — the replica
  rejects writes.

**But** you must still *deploy* the rotation Lambda (code, IAM role,
VPC config, security group) into the standby region as **cold, unused
infrastructure**, because the moment you promote, you need it. A promoted
standalone secret can have rotation turned on — that's one of the two reasons
AWS lists for promoting — and turning it on requires a Lambda in that region.

This is a genuinely awkward asymmetry and it's worth calling out in the design:
**the standby rotation Lambda is the one piece of the Secrets Manager stack that
is pre-provisioned but never exercised.** Untested code paths in a DR region are
exactly how failovers go wrong. Mitigation: exercise it in the non-prod standby
on a schedule, so the code, the IAM role, the VPC route to the database and the
security group rules are all known-good.

**A further wrinkle for database credentials specifically:** a rotation Lambda
changes the password *in the database*. In the standby region, the database it
would rotate against is the RDS cross-region read replica — which is **read-only
until promoted**. So a promoted secret + a rotation Lambda + an unpromoted
database replica = a rotation that fails. The correct ordering during a failover
is: promote the database first ([[amazon-rds-postgres]]), then promote the
secret, then re-enable rotation. Get this order wrong and the rotation Lambda
will fail in a way that leaves the secret in `AWSPENDING` limbo.

### Managed rotation

For RDS, Aurora, Redshift and DocumentDB master user credentials, AWS offers
**managed rotation** — no Lambda at all. *"Rotation for managed secrets
typically completes within one minute."* If any of your secrets are RDS master
credentials, managed rotation removes the standby-Lambda problem entirely for
those, because there is no Lambda to mirror. Worth checking how many of your
rotating secrets qualify. See [[amazon-rds-postgres]].

### The rotation/replication race — the real gotcha

Rotation and replication are two asynchronous processes and **nothing
coordinates them.** The sequence is:

1. Rotation Lambda runs `createSecret` → new value stored as `AWSPENDING`.
2. `setSecret` → the new password is set in the database.
3. `testSecret` → verified.
4. `finishSecret` → `AWSCURRENT` moves to the new version **in the primary**.
5. *…some time later…* replication propagates the new version to the replica.

Between 4 and 5, **the replica serves the old password while the database
already has the new one.** Any standby component reading the replica in that
window gets a credential that no longer works.

AWS does not publish a replication-lag SLA or a typical figure, and I found **no
authoritative number** — the question has been asked on
[re:Post](https://repost.aws/questions/QUJEj9AP8YTUyd9ppHF8o2WQ/aws-secrets-manager-replication-lag)
without a committed answer from AWS. Treat the lag as "usually seconds,
unbounded in principle".

For an **active/passive** posture this is mostly benign: the standby isn't
serving traffic, so nobody is using the stale value. It becomes a real problem
only if:

- The standby runs warm pods that hold live database connections (they will,
  if you're meeting a 15-minute RTO with pre-provisioned capacity) — those pods
  will get auth failures on reconnect during the rotation window.
- You fail over *during* the propagation window.

**Mitigations, in order of preference:**

1. **Use the alternating-users rotation strategy.** Two database users, `_clone`
   suffix; rotation alternates between them, so the *previous* credential stays
   valid for a full rotation cycle. This makes the propagation window harmless
   and is AWS's own recommendation for high availability. This is the right
   answer and it costs you one extra database user per credential.
2. **Retry on auth failure with a forced cache refresh.** The AWS secrets
   caching libraries and the [Secrets Manager Agent](https://docs.aws.amazon.com/secretsmanager/latest/userguide/secrets-manager-agent.html)
   cache by default; make sure the application invalidates and re-reads on an
   authentication error rather than crash-looping.
3. **Schedule rotation windows away from anything else.** Cosmetic, but cheap.

Do **not** build a "wait for replication before finishing rotation" step into
the rotation Lambda. It couples rotation availability to the standby region's
health — so a standby outage would now break rotation in the healthy primary.
That is the wrong trade for an active/passive design.

## Resource policies

*"Secrets Manager replicates the encrypted secret data and metadata such as tags
and resource policies across the specified Regions."*

So policies do come along. Two caveats:

1. **A resource policy that names region-specific principals or conditions
   replicates verbatim and becomes wrong.** A policy granting
   `arn:aws:iam::1234:role/eks-node-eu-west-1` is replicated to `eu-west-2`
   where that role is not the one reading the secret. IAM role ARNs are
   *account-scoped*, not region-scoped, so a single role ARN works in both — but
   if you have per-region roles (and a cookiecutter monorepo very often does,
   because roles get region suffixes), the replicated policy denies the standby.
   **Use region-neutral role names, or avoid resource policies and grant via
   identity policies instead.** Identity policies are the simpler answer and
   should be the default; a resource policy on a secret is mainly for
   cross-account sharing.
2. Resource policy length is capped at **20,480 characters** per region, not
   adjustable.

## RPO / RTO analysis

| | Verdict |
|---|---|
| **RPO 2h** | **Comfortably met.** Replication is async but propagates in seconds. Even a pathological lag of minutes is two orders of magnitude inside target. |
| **RTO 15m** | **Comfortably met for reads — zero action required.** The replica is already there, already decryptable, already resolvable by the same name. Promotion, *if* you decide you need it, is a 50 TPS control-plane call: seconds for hundreds of secrets, a few minutes for tens of thousands. |

**Read quotas are generous and this is not a stampede risk** (unlike
[[aws-ssm-parameter-store]], where it very much is):

| API | Quota (per region) | Adjustable |
|---|---|---|
| `GetSecretValue` | **10,000/s** | **No** |
| `DescribeSecret` | 40,000/s | No |
| `BatchGetSecretValue` | 100/s | No |
| `PutSecretValue` / `UpdateSecret` / `ReplicateSecretToRegion` / `StopReplicationToReplica` / `RemoveRegionsFromReplication` / `UpdateSecretVersionStage` **combined** | **50/s** | **No** |
| `CreateSecret` | 50/s | No |
| Resource policy APIs combined | 50/s | No |

Note that **none of these are adjustable** — you cannot buy your way out. The
10,000/s read quota is plenty for a cold-starting fleet. The **50/s write quota
is the one that constrains bulk operations**: bootstrapping 5,000 secrets into a
new region is ≥100 seconds of `CreateSecret`, and the promotion script is bound
by the same number.

**The hidden constraint: `GetSecretValue` on a CMK-encrypted secret also calls
KMS `Decrypt`.** So the effective read ceiling in the standby is the *lower* of
10,000/s and the region's KMS quota. In `eu-west-2` that is 20,000/s symmetric
crypto ops shared with everything else in the region — so Secrets Manager is not
the binding constraint, but it contributes to the KMS stampede described in
[[aws-kms]].

Other resource limits worth knowing: **500,000 secrets per region**, **64 KiB
max secret value**, **100 versions per secret**, **20 staging labels across all
versions**.

## Warm standby shape

| Thing | State while idle | Cost |
|---|---|---|
| Replica secrets | Present, current, readable | **$0.40/month each** (see cost section) |
| Standby CMK for secrets | Created, enabled | $1/month (see [[aws-kms]]) |
| Rotation configuration | **Primary only** | — |
| Rotation Lambda in standby | Deployed, never invoked | ~$0 (no invocations); VPC ENIs if VPC-attached |
| Secrets Manager VPC endpoint in standby | Created | **Interface endpoint hourly charge per AZ** — this is a real, non-trivial idle cost. See [[vpc-and-networking]]. |

That last row is worth flagging: if the primary reaches Secrets Manager over an
interface VPC endpoint (it should, for a private EKS cluster), the standby needs
one too, pre-created, and interface endpoints bill hourly per AZ regardless of
traffic. At three AZs that is a standing cost in every standby region, for every
service endpoint you mirror. It is small per endpoint and adds up across the
vault. See [[cost-modelling]].

## Terraform implementation

### The module

```hcl
# modules/secret/variables.tf

variable "name"           { type = string }
variable "env"            { type = string }
variable "description"    { type = string, default = null }
variable "primary_region" { type = string }
variable "standby_region" { type = string }
variable "standby_enabled" { type = bool, default = true }

variable "kms_key_arn_by_region" {
  description = "From module.secrets_key.key_arn_by_region in [[aws-kms]]. The replica MUST use the standby region's key."
  type        = map(string)
}

variable "recovery_window_in_days" {
  description = "0 deletes immediately (do not use in prod). 7-30 otherwise."
  type        = number
  default     = 30
}

variable "rotation_lambda_arn" {
  description = "Rotation Lambda in the PRIMARY region. Rotation is configured on the primary only."
  type        = string
  default     = null
}

variable "rotation_days" {
  type    = number
  default = 30
}
```

```hcl
# modules/secret/main.tf

resource "aws_secretsmanager_secret" "this" {
  name        = "${var.env}/${var.name}"
  description = var.description
  kms_key_id  = var.kms_key_arn_by_region[var.primary_region]

  recovery_window_in_days = var.recovery_window_in_days

  # Deliberately NOT set. If a standalone secret already exists in the standby
  # (e.g. because we promoted during an incident and haven't finished failback),
  # we want the apply to FAIL loudly rather than silently overwrite it.
  # force_overwrite_replica_secret = false

  dynamic "replica" {
    for_each = var.standby_enabled ? [var.standby_region] : []
    content {
      region = replica.value
      # The replica's OWN regional key. Passing the primary's ARN here is
      # rejected: the key must be in the replica region.
      kms_key_id = var.kms_key_arn_by_region[replica.value]
    }
  }

  tags = { Env = var.env, ManagedBy = "terraform" }

  lifecycle {
    prevent_destroy = true
  }
}

# Value is set out of band (pipeline, or a human, or a rotation Lambda's first run).
# Terraform creates the shell so IAM, KMS and replication are all in place.
resource "aws_secretsmanager_secret_version" "bootstrap" {
  secret_id     = aws_secretsmanager_secret.this.id
  secret_string = jsonencode({ bootstrap = "replace-me" })

  lifecycle {
    ignore_changes = [secret_string, version_stages]
  }
}

# Rotation: PRIMARY ONLY. There is no replica equivalent and there should not be.
resource "aws_secretsmanager_secret_rotation" "this" {
  count               = var.rotation_lambda_arn == null ? 0 : 1
  secret_id           = aws_secretsmanager_secret.this.id
  rotation_lambda_arn = var.rotation_lambda_arn

  rotation_rules {
    automatically_after_days = var.rotation_days
  }
}
```

```hcl
# modules/secret/outputs.tf
output "arn_by_region" {
  description = "Consumers index by their own region so a standby resource can never be handed the primary ARN."
  value = merge(
    { (var.primary_region) = aws_secretsmanager_secret.this.arn },
    var.standby_enabled ? {
      (var.standby_region) = replace(
        aws_secretsmanager_secret.this.arn,
        ":${var.primary_region}:",
        ":${var.standby_region}:"
      )
    } : {}
  )
}

output "name" { value = aws_secretsmanager_secret.this.name }
```

The `arn_by_region` output relies on the documented property that *"The ARN for
a replicated secret is the same as the primary secret except for the Region"* —
including the random suffix. That is the *only* reason this string substitution
is safe, and it's worth the comment in the code because it looks like a hack
otherwise.

**Better still: have consumers resolve by name.** `GetSecretValue` accepts the
friendly name, the regional SDK client picks the right region, and there is no
ARN plumbing at all. Reserve ARNs for IAM policies, which do need them.

### The standby rotation Lambda (cold)

```hcl
# Deployed but not wired to any secret. Exists so that a promoted standalone
# secret can have rotation enabled without a deploy during an incident.
module "rotation_lambda_standby" {
  count     = var.standby_enabled ? 1 : 0
  source    = "../../modules/rotation-lambda"
  providers = { aws = aws.standby }

  name       = "${var.env}-${var.name}-rotation"
  vpc_config = var.standby_vpc_config
  kms_key_arn = var.kms_key_arn_by_region[var.standby_region]
  # NOTE: no aws_secretsmanager_secret_rotation resource points at this.
  # It is deliberately unattached until a promotion happens.
}
```

Do **not** attach it with an `aws_secretsmanager_secret_rotation` resource in the
standby. The replica can't rotate, so Terraform would either error or (worse)
appear to succeed and do nothing.

### IAM

Grant on the **regional** ARN. A policy naming only the primary ARN silently
denies in the standby:

```hcl
data "aws_iam_policy_document" "read_secrets" {
  statement {
    actions = ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"]
    # Wildcard the suffix; pin the region to the region this role runs in.
    resources = ["arn:aws:secretsmanager:${var.region}:${var.account_id}:secret:${var.env}/*"]
  }
  statement {
    actions   = ["kms:Decrypt"]
    resources = [var.kms_key_arn_by_region[var.region]]
    condition {
      test     = "StringEquals"
      variable = "kms:ViaService"
      values   = ["secretsmanager.${var.region}.amazonaws.com"]
    }
  }
}
```

The `kms:ViaService` value carries the region. This is [[aws-kms]] gotcha #5 in
its most common concrete form and it is worth writing out once, here, so nobody
copy-pastes the primary's version into the standby.

## Migration path from single-region

Largely **already done** by the team. For any remaining single-region secrets,
and as a verification checklist for what exists:

1. **Adding a `replica` block to an existing `aws_secretsmanager_secret` does
   NOT force replacement.** It is an in-place update that calls
   `ReplicateSecretToRegion`. No downtime, no ForceNew. This is the good case.
2. **Changing `kms_key_id` on the primary does not force replacement either** —
   but old versions stay encrypted under the old key. See the re-encryption table
   in [[aws-kms]].
3. **Changing `name` DOES force replacement**, and with a 30-day recovery window
   the old secret lingers. Combined with `prevent_destroy`, a rename is a
   deliberate, multi-step operation.
4. Check for a stale replica from a previous experiment before applying — a
   name collision in the destination is what `force_overwrite_replica_secret`
   exists for, and you want to hit the error rather than the overwrite.

**Verification checklist for the already-completed work** (this is the actually
useful part of this section, given the work is done):

```bash
# 1. Every secret has a replica, and it's in the right state.
aws secretsmanager list-secrets --region "$PRIMARY" \
  --query 'SecretList[].Name' --output text | tr '\t' '\n' | while read -r s; do
  aws secretsmanager describe-secret --secret-id "$s" --region "$PRIMARY" \
    --query "[Name, ReplicationStatus[].[Region,Status,StatusMessage]]" --output text
done
# Status must be InSync. Anything Failed or InProgress for more than a moment
# is a finding. StatusMessage usually names a KMS permissions problem.

# 2. The replica actually decrypts with the standby's roles.
#    Run this ASSUMING THE STANDBY WORKLOAD ROLE, not as an admin —
#    an admin can decrypt via the key policy's root statement and prove nothing.
aws secretsmanager get-secret-value --secret-id "$s" --region "$STANDBY" >/dev/null

# 3. Name sets match.
diff \
  <(aws secretsmanager list-secrets --region "$PRIMARY" --query 'sort(SecretList[].Name)' --output text | tr '\t' '\n') \
  <(aws secretsmanager list-secrets --region "$STANDBY" --query 'sort(SecretList[].Name)' --output text | tr '\t' '\n')

# 4. KMS key audit — CMK in primary, AWS managed key in standby is a finding.
```

Put checks 1, 3 and 4 in CI on a schedule. `ReplicationStatus` going `Failed`
silently is the main way an already-done replication setup rots, and the usual
cause is a KMS key policy change in the standby.

## Failover procedure

**Default path: do nothing.** Reads work. The standby workload reads the replica
by name, decrypts with the standby key, and serves traffic. This should be the
documented default in [[failover-runbook]].

Pre-flight check:

```bash
# All replicas InSync, and readable as the workload role.
aws secretsmanager list-secrets --region "$STANDBY" --query 'length(SecretList)'
# Compare to the primary's count. Mismatch = investigate before failing over.
```

**Escalation path: promote.** Only when you need writes. Trigger conditions,
written down in advance:

- The outage will outlast a rotation window, **and** the credential class is one
  where a missed rotation is a compliance breach.
- A credential must be changed *because of* the incident.
- The outage is declared permanent / the region is being abandoned.

Promotion script (parallel, respects the 50 TPS combined write quota):

```bash
#!/usr/bin/env bash
# Promote every replica in $STANDBY to standalone. ONE-WAY. Requires an
# explicit confirmation because there is no un-promote API.
set -euo pipefail
: "${STANDBY:?}" "${ENV:?}"

read -rp "Promote ALL $ENV replicas in $STANDBY to standalone? This is IRREVERSIBLE. Type PROMOTE: " c
[[ "$c" == "PROMOTE" ]] || { echo "aborted"; exit 1; }

aws secretsmanager list-secrets --region "$STANDBY" \
  --filters "Key=name,Values=$ENV/" \
  --query 'SecretList[].Name' --output text | tr '\t' '\n' \
| xargs -P 8 -I{} sh -c '
    for i in 1 2 3 4 5; do
      aws secretsmanager stop-replication-to-replica \
        --secret-id "$1" --region "'"$STANDBY"'" >/dev/null 2>&1 && \
        { echo "promoted $1"; exit 0; }
      sleep $((i * 2))
    done
    echo "FAILED $1" >&2
  ' _ {}
```

`-P 8` with retries keeps you under 50 TPS with headroom. Test this script in
the non-prod pair — it is irreversible, so "we'll write it on the night" is not
acceptable.

**Ordering matters.** If the secret holds database credentials and you intend to
re-enable rotation, promote in this order:

1. Promote the **RDS** replica to a writable primary ([[amazon-rds-postgres]]).
2. Promote the **secret** to standalone.
3. Attach the standby rotation Lambda and enable rotation.

Doing 2 and 3 before 1 gives you a rotation Lambda trying to `ALTER USER` on a
read-only replica.

## Failback

**This is the hard part, and it is entirely created by promotion.** If you never
promoted, failback is free: the primary comes back, replication resumes,
nothing to do.

If you promoted, you now have **two independent secrets with the same name in
two regions**, and one of them has newer values. There is no merge and no
re-link API. The steps:

1. **Decide which side is authoritative.** Almost always the promoted standby,
   because it's the one that was live. But check: did anyone change the primary
   during the outage? (`GetSecretValue --version-stage AWSPREVIOUS` and
   `ListSecretVersionIds` on both sides will tell you.)
2. **Copy the authoritative values into the losing side.** A script over
   `GetSecretValue` in the winner → `PutSecretValue` in the loser. Bounded by
   the 50 TPS write quota.
3. **Re-establish replication in the correct direction.** Two sub-cases:
   - *Primary stays primary (normal failback):* delete the promoted standalone
     secret in the standby (30-day recovery window — or `--force-delete-without-recovery`
     if you are certain, which you should not be), then
     `ReplicateSecretToRegion` from the primary again. **There is a gap between
     delete and re-replicate during which the standby has no secret.** Do this
     when the standby is not serving.
   - *Standby becomes the new primary (permanent switch):* the promoted secret
     is now the primary; add the old primary region as a replica of it. This
     requires deleting the old primary secret first, which is the scarier
     version of the same gap.
4. **Fix Terraform.** The config says "primary in `eu-west-1` with a replica in
   `eu-west-2`". Reality may disagree. Expect to reconcile state by hand —
   `terraform import`, `removed` blocks, and careful plan reading. **Do not run
   a blind `terraform apply` during failback.** With
   `force_overwrite_replica_secret` unset it will error (good); with it set it
   will silently overwrite the promoted secret with the stale primary value
   (catastrophic). This is the strongest argument for leaving that flag off.
5. **Re-point rotation** at the primary-region Lambda and detach the standby one.

**Write this sequence down before you need it.** It is fiddly, it is
irreversible at several points, and it is the part of the Secrets Manager story
that the "we've already done Secrets Manager" narrative doesn't cover. See
[[failback-strategy]].

## Gotchas

1. **Replicas are read-only.** No `PutSecretValue`, no `UpdateSecret`, no
   rotation — except `UpdateSecret` on the KMS key, which is the one documented
   exception.
2. **`StopReplicationToReplica` is one-way.** No re-link API. Promotion is a
   decision, not a reflex.
3. **You must call promotion from the replica region**, not the primary.
4. **Rotation and replication are not transactional.** A window exists where the
   replica serves the old value after the database has the new one. Use the
   alternating-users strategy.
5. **Rotation is configured on the primary only**, but you still need the
   rotation Lambda deployed in the standby for post-promotion use — cold,
   unattached, and therefore untested unless you deliberately exercise it.
6. **A replica defaulting to `aws/secretsmanager` while the primary uses a CMK**
   is a silent compliance gap in an otherwise-working setup. Audit for it.
7. **The replica's KMS key must be in the replica region.** Passing the
   primary's key ARN is rejected.
8. **A standby KMS key policy that doesn't grant the standby roles `kms:Decrypt`
   produces a replica that exists and cannot be read.** `describe-secret` looks
   healthy. Test with `get-secret-value` **as the workload role**, not as an
   admin.
9. **Resource policies replicate verbatim**, so region-suffixed IAM role ARNs in
   a policy break the standby. Prefer identity policies.
10. **`force_overwrite_replica_secret = true` will silently destroy a promoted
    standalone secret** on the next apply. Leave it off; take the error.
11. **Write-side APIs share a hard 50 TPS quota that is not adjustable**, and
    that quota covers promotion, bulk creation and bulk updates.
12. **`ca-west-1` is an opt-in region** and must be enabled on the account before
    any replica can be created there. Account-level action, not Terraform.
13. **Replicas are billed as separate secrets** (see cost). Replication doubles
    your Secrets Manager bill. Three pairs → double across all three.
14. **`ReplicationStatus` can go `Failed` long after a successful apply** — e.g.
    when someone edits the standby key policy. Terraform will not notice unless
    it runs. Monitor it independently.
15. **30-day recovery window on delete** means a deleted secret's name is
    blocked for 30 days. This bites during failback when you try to delete and
    immediately re-create.
16. **Renaming a secret is ForceNew.**

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Promote on failover? | Promote everything immediately as part of the runbook | Do not promote; reads work; promote only on a named trigger | **B.** Promotion is irreversible and creates the whole failback problem. Reads are what a standby needs. Make promotion an escalation with written triggers. |
| Replica KMS key | Default `aws/secretsmanager` | CMK per region | **CMK wherever the primary uses a CMK**; consistency of key policy is the point. Defaulting is fine only where the primary also defaults. Audit the current state — this is the likeliest existing defect. |
| Rotation Lambda in standby | Don't deploy; deploy at promotion time | Deploy cold, pre-provisioned | **B.** Deploying a VPC-attached Lambda at 3am is exactly what RTO 15m forbids. Accept the cost of an untested path and buy it down by exercising it in non-prod. |
| Rotation strategy | Single user | Alternating users (`_clone`) | **Alternating users.** It makes the replication lag window harmless and is AWS's own HA recommendation. One extra DB user per credential. |
| `force_overwrite_replica_secret` | Set true so applies never fail | Leave unset | **Leave unset.** A failing apply is a signal; a silent overwrite of a promoted secret is data loss. |
| Secret reference style | By ARN | By name | **By name** in application code (regional SDK client resolves it), by regional ARN in IAM policies. |
| Parameter Store secrets | Leave in [[aws-ssm-parameter-store]] | Move to Secrets Manager | **Move the genuinely-rotatable credentials; leave config behind.** At $0.40/secret/month × 2 regions, moving config is expensive and pointless. |

## Cost

| Item | Price |
|---|---|
| Secret storage | **$0.40 per secret per month** |
| API calls | **$0.05 per 10,000 API calls** |
| New versions from rotation | **Not charged** — *"rotating a secret creates a new version of the secret. You are not charged for creating new versions."* |

**Replica billing:** multiple third-party pricing analyses state that each
cross-region replica is billed as a separate secret at the full $0.40/month.
**AWS's own pricing page does not state this explicitly** — it defers with *"For
pricing information for replica secrets, see AWS Secrets Manager Pricing"* from
the replication docs, which is circular. Treat "replication doubles the storage
line" as the planning assumption and **verify against an actual bill or Cost
Explorer** (filter on the `AWS Secrets Manager` service, group by region) before
committing a number to [[cost-modelling]]. Do not cite a third-party blog as
authoritative on AWS pricing.

Worked example on the planning assumption: **500 secrets** across three primary
regions = $200/month today. Mirroring all three pairs = **$400/month**. Plus API
calls (negligible at these volumes) plus the standby CMK ($1/month) plus the
standby interface VPC endpoints (which will likely exceed the secrets cost).

**Cost levers:**
- **Don't replicate everything.** Replicate the secrets the standby actually
  needs to boot and serve. A secret used only by a batch job that won't run
  during a failover does not need a replica. This is the biggest lever and
  requires a per-secret classification.
- **Consolidate.** One JSON secret with ten keys is $0.40; ten secrets is $4.00.
  64 KiB is a lot of room. The trade-off is blast radius and per-key IAM
  granularity — consolidate within a trust boundary, not across one.
- **Keep non-secret config in [[aws-ssm-parameter-store]]**, where standard-tier
  parameters are free.

## Open questions

- **How many secrets, and how many are replicated?** Drives the cost number and
  the promotion script's runtime.
- **Are replicas using CMKs or the AWS managed key?** Run the audit. If the
  primary uses a CMK for a compliance reason, a defaulted replica is a finding.
- **What is the rotation cadence, and how many secrets rotate at all?** Decides
  whether "don't promote" is viable for a multi-day outage.
- **Which rotating secrets are RDS/Aurora master credentials eligible for
  managed rotation?** Those need no standby Lambda at all.
- **Are any secrets using the alternating-users rotation strategy today?** If
  not, that's a concrete improvement independent of multi-region.
- **Do any resource policies on secrets name region-suffixed IAM roles?**
- **Has `ca-west-1` been enabled on the account?** Blocking for the CA pair.
- **Has a promotion ever been tested end-to-end in non-prod?** Including the
  failback. If not, this is the highest-value game day in this note — it is the
  only irreversible operation in the Secrets Manager story.
- **Who is allowed to call `StopReplicationToReplica`?** It is irreversible and
  should be a break-glass permission, not something in the standard operator
  role.

## Sources

- [Replicate AWS Secrets Manager secrets across Regions](https://docs.aws.amazon.com/secretsmanager/latest/userguide/create-manage-multi-region-secrets.html) — the core doc. Replica ARN differs only by region; *"Secrets Manager replicates the encrypted secret data and metadata such as tags and resource policies"*; rotation on the primary propagates to replicas; *"The key must be in the replica Region"*; the CLI note that an unspecified key means *"The replica is encrypted with the AWS managed key `aws/secretsmanager`"*; the partition restriction; the requirement to enable the destination region first.
- [Promote a replica secret to a standalone secret](https://docs.aws.amazon.com/secretsmanager/latest/userguide/standalone-secret.html) — **the key note for failover**: *"A replica secret can't be updated independently from its primary secret, except for its encryption key"*; what promotion does; the two reasons AWS gives for promoting; *"You must call `stop-replication-to-replica` from within the replica region"*; and the warning to update applications afterwards.
- [AWS Secrets Manager endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/asm.html) — **the quota table**: `GetSecretValue` 10,000/s, `DescribeSecret` 40,000/s, the 50/s combined write quota covering `PutSecretValue`/`UpdateSecret`/`ReplicateSecretToRegion`/`StopReplicationToReplica`/`RemoveRegionsFromReplication`/`UpdateSecretVersionStage`, 500,000 secrets/region, 64 KiB value, 100 versions, 20,480-char resource policy — **and that none of them are adjustable**. Also confirms `ca-west-1` endpoints including FIPS.
- [How to replicate secrets in AWS Secrets Manager to multiple Regions](https://aws.amazon.com/blogs/security/how-to-replicate-secrets-aws-secrets-manager-multiple-regions/) — AWS Security Blog; the reference architecture and *"When you enable rotation in the primary Region, any changes to the secret from the rotation process are also replicated to the replica Region."* Note it does **not** cover promotion or failback, which is why this note does.
- [Managed rotation for AWS Secrets Manager secrets](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotate-secrets_managed.html) — which services offer Lambda-free managed rotation (RDS, Aurora, Redshift, DocumentDB, ECS Service Connect); *"Rotation for managed secrets typically completes within one minute"*; the recommendation to use the alternating-users strategy for highest availability.
- [Rotation strategies](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotation-strategy.html) — single-user vs alternating-users; the basis for the replication-lag mitigation.
- [AWS Secrets Manager replication lag (re:Post)](https://repost.aws/questions/QUJEj9AP8YTUyd9ppHF8o2WQ/aws-secrets-manager-replication-lag) — the question is asked and **no committed AWS figure is given**. Recorded here as a genuine "no public data found": there is no published replication-lag SLA.
- [`aws_secretsmanager_secret` resource docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/secretsmanager_secret) — the `replica` block (`region`, `kms_key_id`, and the computed `status`/`status_message`/`last_accessed_date`), `force_overwrite_replica_secret`, `recovery_window_in_days`, and the interaction between inline `policy` and `aws_secretsmanager_secret_policy`.
- [`aws_secretsmanager_secret_rotation` resource docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/secretsmanager_secret_rotation) — rotation configured against the primary secret only.
- [AWS Secrets Manager pricing](https://aws.amazon.com/secrets-manager/pricing/) — $0.40/secret/month, $0.05 per 10,000 API calls, and that rotation's new versions are not charged. **Does not state replica billing**, which is why this note flags it as needing verification against a real bill.
- [Secrets Manager Agent](https://docs.aws.amazon.com/secretsmanager/latest/userguide/secrets-manager-agent.html) and [AWS Secrets and Configuration Provider (ASCP)](https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html) — client-side caching, which is both the defence against the KMS decrypt stampede and the thing that must invalidate correctly during a rotation.
