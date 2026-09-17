---
title: Terraform State Management for a Region Pair
service: terraform
tags: [terraform, multi-region, state, s3-backend, locking, bootstrap]
status: researched
replication: manual (S3 CRR) — the backend has no native failover
rpo_achievable: "seconds–15 min for state objects with S3 RTC; state is not the RPO-critical data"
rto_achievable: "0 if state lives in the region it manages; 30-60+ min if you must recover a state bucket first"
meets_targets: conditional
updated: 2026-09-17
---

# Terraform State Management for a Region Pair

## TL;DR

1. **The state bucket is itself a single-region dependency.** If your state lives
   in the region that just died, you cannot run Terraform to fix anything —
   including the standby you built precisely so you would not need Terraform in a
   hurry. Most teams never notice because they never test it.
2. **The fix is boring and cheap: put each root module's state in the region that
   root module manages.** The standby's state lives in the standby region. No
   replication, no failover procedure, no DNS games. This falls straight out of
   the recommendation in [[provider-aliases-vs-separate-stacks]] and is most of
   the reason to prefer it.
3. **Replicating the state bucket does not give you a working backend in the
   other region.** With DynamoDB locking, the LockID embeds the source bucket
   name, so a failover to the replica *cannot take the lock*
   ([hashicorp/terraform#32190](https://github.com/hashicorp/terraform/issues/32190),
   still open). CRR is for *recovery*, not for *failover*. Say that out loud
   before someone designs around it.
4. **DynamoDB is no longer required for locking.** `use_lockfile = true` uses an
   S3 conditional write on a `.tflock` object; introduced experimentally in
   Terraform **1.10**, de-experimented in **1.11**, where `dynamodb_table` was
   marked deprecated — *"DynamoDB-based locking is deprecated and will be removed
   in a future minor version."* One fewer regional resource to bootstrap and
   mirror.
5. **The bootstrap chicken-and-egg is real and it is on the critical path.** The
   state bucket, the KMS key that encrypts it, and the IAM role the pipeline
   assumes must all exist in the standby *before* any Terraform can run there.
   That is a hand-rolled or CloudFormation step, done once per region, and it
   must be done before the first `standby_enabled = true`.

---

## 1. The finding most teams miss

Terraform is not a thing you run once. It is the control plane you reach for when
something is wrong. Every dependency in the path between "engineer types
`terraform apply`" and "AWS API accepts the call" is a dependency of your
recovery, and it deserves the same scrutiny as the application's dependencies.

That path includes:

| Dependency | Where it lives | Fails with the primary region? |
|---|---|---|
| State object in S3 | wherever you put the bucket | **yes, if the bucket is in the primary** |
| Lock (DynamoDB table or `.tflock` object) | same place | **yes** |
| KMS key encrypting the state | a region | **yes** |
| IAM role the pipeline assumes | global, but STS regional endpoints matter | partially |
| The CI runner itself | a region, or a SaaS region | **yes, if it runs in the primary** |
| Provider's API endpoints for every resource in the config | every region the config touches | **yes, for anything in the primary** |

A single-region estate has no reason to care. A DR estate has to care about all
six, and the last one is the subtlest — covered in
[[provider-aliases-vs-separate-stacks]] §5 and [[terraform-gotchas]].

**Real evidence this is not theoretical.** The AWS Architecture blog's write-up
of athenahealth making Terraform Enterprise itself multi-region DR-capable found
exactly this class of bug during AWS FIS testing: their failover scripts
*"initially attempted to retrieve infrastructure identifiers from primary-Region
S3 state files"*, and during an actual regional outage *"this made recovery
impossible"*. Their remedy was to hardcode identifiers or use a region-independent
configuration source, plus drift detection in CI. That is a DR-hardened Terraform
deployment, built by people who do this for a living, and the circular dependency
still got in.

Treat it as a rule: **nothing on the failover path may read from the primary
region.** Not state, not SSM, not a data source, not a health-check lookup.

---

## 2. One state per region, or one shared state?

### 2.1 The options

| | Shared state for the pair | State per region (**recommended**) | State per region + a small pair state |
|---|---|---|---|
| Object count | 1 | 2 | 3 |
| Bucket location | one region — pick one and lose | each region hosts its own | pair state needs a home (§2.3) |
| Standby appliable during primary outage | **no** | yes | yes for the standby; pair state may be blocked |
| Lock contention | one lock for both regions | independent | independent |
| Cross-region references | free (same state) | needs an explicit channel (§4) | free *within* the pair state |
| Blast radius | both regions | one region | one region + a small bounded pair |

Shared state is the natural partner of Option A in
[[provider-aliases-vs-separate-stacks]], and it inherits all of Option A's
problems plus one more: **the bucket has to be somewhere.** Whichever region you
choose, you have created a state backend that is unavailable exactly when you
need it, for one of the two regions in the pair. There is no third option that
doesn't introduce a third region.

### 2.2 The recommendation

```
acme-tfstate-eu-west-1   (eu-west-1)   ← state for the eu-west-1 root module only
acme-tfstate-eu-west-2   (eu-west-2)   ← state for the eu-west-2 root module only
acme-tfstate-us-east-1   (us-east-1)   ← …and so on, one per region in use
acme-tfstate-us-west-2   (us-west-2)
acme-tfstate-ca-central-1 (ca-central-1)
acme-tfstate-ca-west-1   (ca-west-1)
```

One bucket per region, in that region. Every root module's backend points at the
bucket in the region it manages. Keys namespace by pair, environment and stack:

```hcl
terraform {
  backend "s3" {
    bucket       = "acme-tfstate-eu-west-2"
    key          = "prod-eu/eu-west-2/platform.tfstate"
    region       = "eu-west-2"
    use_lockfile = true
    encrypt      = true
    kms_key_id   = "arn:aws:kms:eu-west-2:111122223333:key/<standby-state-key>"
  }
}
```

This is not a clever design. That is the point: **there is no failover procedure
for the state backend, because there is nothing to fail over.** The standby's
backend was never in the primary region, so a primary outage does not touch it.

Note the region is also in the bucket *name*. Under the cookiecutter scheme in
[[module-patterns]] the bucket name is derived from the region variable, so the
backend block is generated, never hand-written, and it is impossible to point a
standby root module at a primary bucket by copy-paste error.

### 2.3 Where does the `pair` stack's state live?

[[provider-aliases-vs-separate-stacks]] §6.2 carves out a small third root module
for resources that genuinely span both regions — DynamoDB global tables, KMS
multi-region keys, S3 replication configuration, Route 53 failover records. Its
state has to live in one of the two regions, so it has the shared-state problem
in miniature.

Three answers, in order of preference:

1. **Put it in the standby region.** Counter-intuitive and correct. The pair
   stack's resources are mostly *created* during normal operations and *read*
   during an incident. If the primary is down, the standby-hosted state is
   reachable, and if the standby is down you are not failing over anyway. This
   also means the one stack whose plan spans both regions cannot be planned
   during a primary outage — accept that, and make sure nothing on the 15-minute
   failover path lives in it.
2. **Put it in a third, unrelated region.** Cleanest isolation, worst
   operational ergonomics, and for the CA pair it may be a data-residency
   violation ([[data-residency]]). Probably not.
3. **Put it in the primary and accept it.** Only acceptable if you have genuinely
   verified nothing in the pair stack is needed during a failover.

Critically: **Route 53 failover records must not require the pair stack to
apply.** Route 53 is global with a control plane whose write path is hosted in
`us-east-1`; failover DNS changes at 3am should be made by an automated
mechanism (health checks, ARC routing control) rather than by a Terraform apply
at all. See [[aws-route53]] and [[failover-orchestration]].

---

## 3. Locking

### 3.1 S3 native locking (`use_lockfile`) vs DynamoDB

`use_lockfile = true` makes Terraform write a `.tflock` object next to the state
object using an S3 conditional write — a `PutObject` with an `If-None-Match`
header, which
[S3 made generally available on 20 August 2024](https://aws.amazon.com/about-aws/whats-new/2024/08/amazon-s3-conditional-writes).
The put succeeds only if no lock object exists; if one does, S3 returns
`412 PreconditionFailed` and Terraform reports who holds the lock. Releasing the
lock deletes the object. The implementation is
[hashicorp/terraform#35661](https://github.com/hashicorp/terraform/pull/35661),
merged 11 October 2024.

Version timeline:

| Version | State of `use_lockfile` |
|---|---|
| 1.10 | Introduced, documented as experimental, opt-in |
| 1.11 | "Experimental" removed; **`dynamodb_table`, `dynamodb_endpoint` and `endpoints.dynamodb` marked deprecated** |
| current (1.16.x) | `use_lockfile` documented as *"Whether to use a lockfile for locking the state file. Defaults to `false`."* DynamoDB path still present but deprecated: *"DynamoDB-based locking is deprecated and will be removed in a future minor version."* |

**Recommendation: `use_lockfile = true`, no DynamoDB lock table.**

Reasons that matter for this project specifically:

- **One fewer regional resource to bootstrap in each standby region.** The
  bootstrap list drops from {bucket, KMS key, DynamoDB table, IAM role} to
  {bucket, KMS key, IAM role}. Every item on that list is a hand-rolled step
  (§5), so removing one is real.
- **One fewer thing to get wrong per region.** A DynamoDB lock table in the
  wrong region is a silent cross-region dependency.
- **It removes the DynamoDB-in-us-east-1 correlation.** The 19–20 October 2025
  event was a DynamoDB DNS failure in `us-east-1` that took out, among much else,
  the ability to reach the DynamoDB regional endpoint
  ([AWS post-event summary](https://aws.amazon.com/message/101925)). A lock table
  is a hard dependency on DynamoDB being reachable. Locking in S3 instead does not
  make you outage-proof, but it removes one independent service from the path.
- The team is mid-migration to DynamoDB Global Tables
  ([[dynamodb-table-naming-migration]]); not having lock tables is one less
  DynamoDB naming problem.

**Migration**, per-backend, is a two-step that must be coordinated so no one is
mid-apply:

```hcl
# Step 1 — both mechanisms, for one release. Terraform takes both locks.
terraform {
  backend "s3" {
    bucket         = "acme-tfstate-eu-west-1"
    key            = "prod-eu/eu-west-1/platform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "acme-tfstate-lock"   # deprecated, still honoured
    use_lockfile   = true
    encrypt        = true
  }
}

# Step 2 — drop dynamodb_table once every runner is on the new config.
```

Running both for one release is the safe path: a runner on the old config takes
only the DynamoDB lock, a runner on the new config takes only the S3 lock, and
during the overlap nothing protects you from the two colliding. With both set,
both are taken.

### 3.2 The lock and cross-region replication do not mix

If you replicate the state bucket (§4), the `.tflock` objects replicate too. A
lock taken in the primary appears in the replica seconds later, and a lock
*released* in the primary is a delete that only replicates if delete-marker
replication is enabled — and delete-marker replication
[does not carry the S3 RTC 15-minute SLA](https://aws.amazon.com/blogs/storage/managing-delete-marker-replication-in-amazon-s3).
The failure mode is a phantom lock in the replica bucket that no one holds and
`terraform force-unlock` is the only way out of, at 3am, under time pressure.

Filter `.tflock` out of the replication rule, or accept that using the replica
means `force-unlock` first. This is another argument for §2.2: if you never plan
to *run* from the replica, none of this matters.

---

## 4. Cross-region replication of the state bucket

### 4.1 What CRR is for, and what it is not for

**CRR protects against losing the state, not against being unable to reach it.**
The distinction is the whole section.

It genuinely protects against:
- Region-level destruction of the bucket (the true disaster case)
- Bucket-level accidents, when combined with versioning and MFA delete
- Account-level accidents, if the replica is in a different account

It does **not** give you a working backend in the other region, because:

- **With DynamoDB locking, the LockID embeds the source bucket name.** From
  [hashicorp/terraform#32190](https://github.com/hashicorp/terraform/issues/32190)
  (open): *"bucket name of initial apply is in the LockID value which makes it
  impossible to run failover if AWS is down in the inital apply region"*, and the
  reporter's summary of the operational reality — *"in a failover scenario I have
  to modify lockIDs manually.. not something I want to do to swing an application
  to another region"*. `use_lockfile` sidesteps this (the lock is an object key
  relative to the state key, not a composite ID), which is another point in its
  favour, but see §3.2.
- **Every backend config pins `bucket` and `region`.** Failing over means editing
  and re-initialising the backend on every root module, or wiring
  `-backend-config` overrides through CI. Doable, but it is a procedure, and
  procedures that only run during disasters do not work.
- **S3 Multi-Region Access Points are not supported by the S3 backend**
  ([terraform-provider-aws#28490](https://github.com/hashicorp/terraform-provider-aws/issues/28490)
  is the standing request). There is no transparent endpoint.
- **CRR is asynchronous.** Even with
  [S3 Replication Time Control](https://aws.amazon.com/s3/features/replication/),
  the guarantee is *"99.99 percent of new objects... within 15 minutes"*, backed
  by an [SLA on 99.9% within 15 minutes per billing month](https://www.aws.amazon.com/s3/sla-rtc/).
  A state object that was written during the last apply before the outage may not
  be in the replica. Applying against a stale state is how you get duplicate
  resources.

### 4.2 Recommendation

**Enable CRR on every state bucket, treat it as backup, and never build a
failover procedure on it.** Configuration that matters:

```hcl
resource "aws_s3_bucket" "tfstate" {
  bucket = "acme-tfstate-${var.region}"
}

resource "aws_s3_bucket_versioning" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  versioning_configuration { status = "Enabled" }   # required for replication
}

resource "aws_s3_bucket_replication_configuration" "tfstate" {
  depends_on = [aws_s3_bucket_versioning.tfstate]

  bucket = aws_s3_bucket.tfstate.id
  role   = aws_iam_role.state_replication.arn

  rule {
    id       = "state-to-peer"
    status   = "Enabled"
    priority = 1

    filter {
      prefix = ""
    }

    # Do NOT replicate lock objects — see §3.2
    # (S3 filters are prefix/tag based, not suffix based, so if lock hygiene
    #  matters, key lock objects under a distinct prefix or accept force-unlock.)

    delete_marker_replication { status = "Disabled" }

    destination {
      bucket        = var.peer_state_bucket_arn
      storage_class = "STANDARD_IA"

      encryption_configuration {
        replica_kms_key_id = var.peer_state_kms_key_arn   # a key in the PEER region
      }

      # RTC is optional. State is small; the 15-minute SLA is not the binding
      # constraint here, and RTC is billed on top of transfer.
      # replication_time { status = "Enabled" time { minutes = 15 } }
      # metrics         { status = "Enabled" event_threshold { minutes = 15 } }
    }

    source_selection_criteria {
      sse_kms_encrypted_objects { status = "Enabled" }
    }
  }
}
```

Three details people get wrong:

- `delete_marker_replication` is **disabled by default** and that is what you
  want for a backup copy — you do not want a `terraform state rm` gone wrong, or
  a lifecycle expiry, to propagate. Note the corollary: the replica is not a
  mirror, it is an append-only-ish archive. Fine for backup, wrong for failover.
- **KMS: the replica must be encrypted with a key in the destination region.**
  A KMS key is regional; the destination bucket cannot use the source's key. This
  means the standby's state KMS key is a bootstrap prerequisite (§5) and the
  replication IAM role needs `kms:Decrypt` on the source key and
  `kms:Encrypt`/`GenerateDataKey` on the destination key. See [[aws-kms]].
- **Bidirectional replication is possible** (two rules, plus
  [replica modification sync](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-for-metadata-changes.html))
  and the athenahealth/TFE design used it — *"bidirectional... with less than 30
  seconds replication lag"* — because it *"supports failback without data
  resynchronization"*. For a TFE deployment whose whole job is holding state,
  that is right. For our case, where each root module's state already lives in
  the right region, bidirectional replication buys little and adds a loop that
  can fight itself. Recommend one-way, primary → standby and standby → primary as
  two independent one-way rules only if failback analysis demands it
  ([[failback]]).

### 4.3 The things that actually protect the state

In rough order of how often they save you:

1. **Versioning.** Every state write is a new version. Recovering from a bad
   apply or a truncated state is `aws s3api list-object-versions` and a copy.
   Non-negotiable, and a prerequisite for CRR anyway.
2. **`prevent_destroy` on the bucket** plus a bucket policy denying
   `s3:DeleteBucket` outside a break-glass role.
3. **CRR to the peer region** (§4.2).
4. **AWS Backup on the bucket**, if a longer retention or a logically-air-gapped
   vault is wanted. See [[aws-backup]].
5. **Object Lock** in governance mode if compliance asks. Note this interacts
   badly with routine state writes if applied to the state key prefix — apply it
   to a separate archival copy, not the live state.

---

## 5. The bootstrap chicken-and-egg

Before a single `terraform apply` can run against `eu-west-2`, these must already
exist **in `eu-west-2`**:

1. **The state S3 bucket**, versioned, encrypted, public access blocked.
2. **A KMS key** for state encryption (regional — the primary's key is not usable
   from the standby for new writes, and cross-region `kms:Decrypt` is not a
   thing for symmetric regional keys). See
   [[kms-when-to-use-multi-region-keys]] for whether this should be an MRK.
3. **The IAM role the CI pipeline assumes**, with a trust policy the CI OIDC
   provider satisfies, and permissions on the bucket and key. IAM itself is
   global, so the role *exists* everywhere — but the `sts:AssumeRoleWithWebIdentity`
   call must be made against a **regional STS endpoint**, not the global one, or
   you have reintroduced a `us-east-1` dependency. See §6.
4. **(Not needed if `use_lockfile = true`)** the DynamoDB lock table.
5. Optionally, **SSM parameters** carrying the bucket/key names so other
   automation can discover them without reading state.

None of these can be created by the Terraform that will later use them. Options:

| Approach | Verdict |
|---|---|
| Manual console clicks | Works once, undocumented forever. No. |
| **A `bootstrap/` root module with local state, committed to the repo, state file committed or archived** | Simple, auditable, reproducible. **Recommended.** |
| A `bootstrap/` root module that stores its own state in the bucket it creates (create → `init -migrate-state`) | Elegant, and the standard trick. Slight risk: the bootstrap state is then subject to the thing it bootstraps. Acceptable. |
| CloudFormation stack | Genuinely good — CFN has no state backend of its own, so there is no egg. Costs a second tool. |
| `aws cloudformation` / CLI script in the repo | Same as above without the stack management |

**Recommendation: a `bootstrap/` cookiecutter-rendered root module per region,
run once by a human with elevated credentials, that then migrates its own state
into the bucket it just created.** Commit the rendered module. The procedure for
adding a region is then: render `bootstrap/<region>`, apply it, render the
regular root modules, apply those.

```hcl
# bootstrap/eu-west-2/main.tf  — run with LOCAL state the first time,
# then `terraform init -migrate-state` after the bucket exists.

provider "aws" { region = "eu-west-2" }

resource "aws_kms_key" "tfstate" {
  description             = "Terraform state encryption — eu-west-2"
  enable_key_rotation     = true
  deletion_window_in_days = 30
}

resource "aws_kms_alias" "tfstate" {
  name          = "alias/tfstate"
  target_key_id = aws_kms_key.tfstate.key_id
}

resource "aws_s3_bucket" "tfstate" {
  bucket = "acme-tfstate-eu-west-2"

  lifecycle { prevent_destroy = true }
}

resource "aws_s3_bucket_versioning" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.tfstate.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "tfstate" {
  bucket                  = aws_s3_bucket.tfstate.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_iam_role" "terraform_ci" {
  name = "terraform-ci-eu-west-2"
  assume_role_policy = data.aws_iam_policy_document.ci_trust.json
}
```

**Sequencing note for the prerequisites-first strategy:** bootstrap is the
*zeroth* prerequisite. It comes before Secrets Manager, before DynamoDB, before
anything in [[sequencing-roadmap]]. If it has not been done for `eu-west-2`,
`us-west-2` and `ca-west-1`, nothing else in this vault can start. Confirm its
status early.

---

## 6. Does HCP Terraform remove the concern?

Partly, and it moves the rest.

**What it removes:** you no longer operate a state backend. HashiCorp does.
There is no bucket to place, no bucket to replicate, no lock table to bootstrap,
and no `-backend-config` to swap during a failover. Locking, versioning and
state history are managed. For an estate whose main state risk is operational
sloppiness, that is a genuine improvement.

**What it does not remove:**

- **You have swapped an AWS regional dependency for a SaaS dependency.** HCP
  Terraform's own availability is now on your failover path. HashiCorp operates
  its DR; you do not control it and you cannot test it.
  [HCP Europe](https://developer.hashicorp.com/hcp/docs/hcp/europe) exists as a
  separately hosted, separately billed instance for European data residency —
  which is a consideration for the EU and CA pairs ([[data-residency]]) and a
  complication, since the CA pair's residency requirements are Canadian, not
  European.
- **You still need your own state backup.** HashiCorp publishes
  [a procedure for downloading state via the API](https://support.hashicorp.com/hc/en-us/articles/4411620536979-How-to-backup-your-state-file-from-HCP-Terraform-for-disaster-recovery)
  precisely because accidental workspace deletion is your problem, not theirs.
- **It does nothing about the bigger problem**, which is not where the state
  lives but **whether the plan can be produced at all** when one region's API
  endpoints are unreachable. That is a property of the configuration's structure
  ([[provider-aliases-vs-separate-stacks]]), not of the backend.
- **Agents still run somewhere.** If you use self-hosted agents in the primary
  region, you have re-added the dependency you paid to remove.

**Recommendation: no, not for this.** Adopting HCP Terraform is a reasonable
independent decision on collaboration, policy-as-code and run history grounds. It
is not a solution to the state-region problem, because the state-region problem
is already solved by putting each state in its own region for free. Do not let
"it fixes our DR state problem" be the business case, because it mostly doesn't.

The exception worth revisiting: **Terraform Stacks** requires HCP Terraform and
gives per-deployment state isolation *by construction* — see
[[provider-aliases-vs-separate-stacks]] §6.4. If Stacks is ever adopted, this
section's answer changes.

---

## 7. Running Terraform *during* an incident

The scenario: `eu-west-1` is degraded. The decision to fail over has been made.
The clock says 15 minutes.

What must work:

1. **The CI runner must not be in `eu-west-1`.** If the pipeline executes in the
   dead region, nothing below matters. See [[terraform-gotchas]] §CI/CD.
2. **`terraform init` against `acme-tfstate-eu-west-2`** — unaffected, the bucket
   is in the healthy region.
3. **Lock acquisition** — an S3 conditional write in `eu-west-2`. Unaffected.
4. **Refresh and plan** — every resource in the `eu-west-2` root module is in
   `eu-west-2`. Unaffected. This is the payoff for §2.2 and for separate root
   modules.
5. **Apply** — scale the standby up, flip `standby_enabled` flags that were
   deliberately left off, adjust DNS weights if DNS is Terraform-managed (it
   should not be, see §2.3).

What will not work, and must therefore not be on the path:

- Any `terraform_remote_state` data source pointing at the primary's bucket.
- Any `data "aws_..."` resolved through a provider configured for the primary.
- Any apply against the `pair` stack, if it is hosted in the primary.
- Any `-target`-based surgery on a shared root module. It is documented as being
  for *"exceptional circumstances"* and produces undetected drift; an incident is
  the worst possible time to acquire undetected drift.

**Test this.** The athenahealth/TFE work used AWS FIS —
`aws:network:disrupt-connectivity` on compute subnets to simulate regional S3
access loss — and that is what surfaced the circular dependency. A game day that
blackholes the primary region's endpoints from the CI runner and then asks
"can we still apply to the standby?" is a one-day exercise that will find
something. See [[dr-testing-and-gamedays]].

---

## 8. Migration path from today

The estate is live and single-region per deployment, presumably with one state
bucket per environment or one per account.

1. **Inventory.** For each root module: which bucket, which region, which lock
   mechanism, which KMS key. Expect surprises — a `us-east-1` bucket holding
   `eu-west-1` state is the classic.
2. **Do not move existing primary state.** Moving state is risk with no reward
   while there is only one region. If a primary's state is already in the same
   region as its resources, it is already correct.
3. **Fix any state that is in the wrong region.** If `eu-west-1`'s state is in
   `us-east-1`, that is a live single-region correlation and a cheap fix:
   `aws s3 cp` the object to a new bucket in `eu-west-1`, update `backend.tf`,
   `terraform init -migrate-state`, verify with `terraform plan` showing no
   changes. Do this before the DR work, not during it.
4. **Migrate locking to `use_lockfile`** (§3.1), one backend at a time, with the
   both-mechanisms overlap release.
5. **Bootstrap the three standby regions** (§5). `eu-west-2` first.
6. **Add CRR to every state bucket**, pairing each with its peer (§4.2).
7. **Then**, and only then, render the standby root modules with
   `standby_enabled = false` and begin the service-by-service rollout.

Nothing here forces a resource replacement. `terraform init -migrate-state`
rewrites where state lives, not what it describes; verify with a no-change plan
after each move.

---

## 9. Gotchas

- **`terraform_remote_state` is a fate-sharing device.** The standby reading the
  primary's outputs is the single most common way this design gets quietly
  broken. Prefer values replicated *into* the standby: SSM Parameter Store is the
  obvious channel, but note SSM has no native cross-region replication
  ([[aws-ssm-parameter-store]]) — this is a real gap and one of the reasons that
  note matters. Second choice: naming convention, so the standby *derives* the
  name rather than looking it up.
- **The backend block cannot use variables.** `bucket`, `key` and `region` must
  be literals or supplied via `-backend-config`. This is why the backend must be
  *generated* (cookiecutter) rather than parameterised. See [[module-patterns]].
- **`encrypt = true` plus `kms_key_id` needs KMS permissions on the role**:
  `kms:Encrypt`, `kms:Decrypt`, `kms:GenerateDataKey`. Missing `Decrypt` produces
  an `init` that succeeds and a `plan` that fails confusingly.
- **A replicated `.tflock` is a phantom lock** (§3.2).
- **`workspace_key_prefix` defaults to `env:`.** If anyone is using CLI
  workspaces to separate regions today, that is a different (and worse) model
  than per-region root modules; note that workspaces share a backend and
  therefore share the bucket's region. Migrating off workspaces onto directories
  is a `state pull`/`state push` exercise. Flag it if found.
- **STS global endpoint.** `sts.amazonaws.com` resolves to `us-east-1`. Provider
  configuration should set `sts_region` / use regional STS endpoints so that
  assuming the pipeline role does not depend on `us-east-1`. For the US pair this
  is doubly important, because `us-east-1` *is* the primary.
- **State bucket names are globally unique.** `acme-tfstate-eu-west-2` must be
  claimed before someone else does. Claim all six now, including regions not yet
  in use.
- **Replica buckets are ordinary writable buckets.** Nothing stops a confused
  engineer running Terraform against the replica and creating a divergent state.
  Deny `s3:PutObject` on the replica for everyone except the replication role.

---

## 10. Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| State layout | One shared state for the pair | One state per region, in that region | **B** — the standby's backend must survive the primary's outage, and B achieves that for free |
| Locking | DynamoDB table | `use_lockfile = true` | **`use_lockfile`** — one fewer regional resource to bootstrap, removes a DynamoDB dependency, and DynamoDB locking is deprecated |
| CRR on state buckets | Skip it, state per region is enough | Enable it as backup | **Enable, as backup only.** Never build a failover procedure on the replica |
| Replication direction | Bidirectional with replica modification sync | One-way per pair | **One-way**, revisit under [[failback]] |
| Where the `pair` stack's state lives | Primary region | Standby region | **Standby region**, and keep the pair stack off the 15-minute failover path |
| Bootstrap mechanism | Manual / console | `bootstrap/` root module per region, rendered by cookiecutter | **Bootstrap module**, state migrated into the bucket it creates |
| HCP Terraform | Adopt to solve state DR | Stay on S3 backend | **Stay.** HCP solves a different problem and adds a SaaS dependency to the failover path |
| STS endpoints | Global | Regional | **Regional** — otherwise the US pair's Terraform depends on its own failed region |

## 11. Cost

Trivial, and worth stating so it is not an objection:

- Six state buckets, each holding megabytes. Storage cost is rounding error.
- CRR: per-GB replication transfer plus `PUT` requests. State objects are small
  and written a few times a day. Still rounding error. RTC would add a per-GB
  charge for no benefit here — skip it.
- No DynamoDB lock tables under the recommendation — a small saving, but the
  point is fewer moving parts, not the money.
- KMS: one customer-managed key per region, $1/month each plus request charges.

The only non-trivial cost in this note is engineering time on the bootstrap, and
that is one-off per region.

## 12. Open questions

- Where does each existing state bucket actually live today, and does any of it
  sit in a different region from the resources it manages?
- Is anyone using CLI workspaces (`workspace_key_prefix`) to separate
  environments or regions? That changes the migration significantly.
- Which Terraform version is the estate on? `use_lockfile` needs 1.10+; the
  recommendation assumes 1.11+.
- Are the standby regions bootstrapped at all yet? `eu-west-2` may be, given
  Secrets Manager replication is already live — but replica secrets are created
  by the *primary's* provider, so it is entirely possible no Terraform has ever
  run *in* `eu-west-2`.
- Is `ca-west-1` even enabled on the account? Newer regions are opt-in, and
  enabling a region is itself an account-level action
  ([[region-pair-selection]]).
- Where does CI execute today, and is it pinned to a region?

## Sources

- [Backend Type: s3 — Terraform docs](https://developer.hashicorp.com/terraform/language/backend/s3) — `use_lockfile` (*"Defaults to `false`"*), `dynamodb_table` marked Deprecated, *"DynamoDB-based locking is deprecated and will be removed in a future minor version."*, `kms_key_id` permission requirements, `workspace_key_prefix` default of `env:`.
- [hashicorp/terraform#35661 — Introduce S3-native state locking](https://github.com/hashicorp/terraform/pull/35661) — `.tflock` object, conditional write, 412 on conflict, merged 11 Oct 2024.
- [Amazon S3 now supports conditional writes](https://aws.amazon.com/about-aws/whats-new/2024/08/amazon-s3-conditional-writes) — `If-None-Match` GA, 20 Aug 2024; the primitive the lockfile depends on.
- [hashicorp/terraform#32190 — S3 Backend: Support multi-region bucket replication](https://github.com/hashicorp/terraform/issues/32190) — open; LockID embeds the bucket name, so failover to a replicated bucket requires manual LockID editing.
- [terraform-provider-aws#28490 — Allow use of S3 Multi-Region Access Point for state storage](https://github.com/hashicorp/terraform-provider-aws/issues/28490) — MRAPs are not supported by the S3 backend.
- [Validating multi-Region DR for Terraform Enterprise with AWS FIS — AWS Architecture blog](https://aws.amazon.com/blogs/architecture/validating-multi-region-dr-for-terraform-enterprise-with-aws-fis/) — active/passive TFE, bidirectional S3 CRR with <30s lag, 12–14 minute RTO, <1 minute RPO, and the circular dependency on primary-region state discovered by FIS.
- [Amazon S3 Replication (features)](https://aws.amazon.com/s3/features/replication/) and [S3 RTC SLA](https://www.aws.amazon.com/s3/sla-rtc/) — RTC replicates 99.99% of objects within 15 minutes, SLA on 99.9%.
- [Managing delete marker replication in Amazon S3 — AWS Storage blog](https://aws.amazon.com/blogs/storage/managing-delete-marker-replication-in-amazon-s3) — delete-marker replication is off by default and outside the RTC SLA.
- [Replicating metadata changes with replica modification sync — S3 docs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-for-metadata-changes.html) — what bidirectional replication actually requires.
- [AWS post-event summary, 19–20 Oct 2025](https://aws.amazon.com/message/101925) — DynamoDB regional endpoint DNS failure in `us-east-1`; why a DynamoDB lock table is an extra dependency worth removing.
- [HCP Europe — HashiCorp docs](https://developer.hashicorp.com/hcp/docs/hcp/europe) — separately hosted/billed for EU data residency.
- [How to backup your state file from HCP Terraform for disaster recovery — HashiCorp support](https://support.hashicorp.com/hc/en-us/articles/4411620536979-How-to-backup-your-state-file-from-HCP-Terraform-for-disaster-recovery) — HashiCorp's own position that you still need your own state backups on HCP.
- [Multi-Region Terraform Deployments with AWS CodePipeline — AWS DevOps blog](https://aws.amazon.com/blogs/devops/multi-region-terraform-deployments-with-aws-codepipeline-using-terraform-built-ci-cd/) — per-env bucket with per-region state keys, `-backend-config` driven `init`, and the isolation claim.
- [terraform plan command reference](https://developer.hashicorp.com/terraform/cli/commands/plan) — the `-target` "exceptional circumstances" warning quoted in §7.
