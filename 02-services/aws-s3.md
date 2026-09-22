---
title: Amazon S3 — Multi-Region
service: s3
tags: [service, multi-region, s3, replication, storage]
status: researched
replication: native
rpo_achievable: "seconds to ~15 min for new writes (CRR/RTC); hours-to-days for the initial backfill of existing objects (Batch Replication)"
rto_achievable: "< 5 min if the standby bucket is pre-created and every consumer resolves the bucket name from config"
meets_targets: conditional
updated: 2026-09-21
---

# Amazon S3 — Multi-Region

## TL;DR

- **S3 buckets are regional resources with a global-ish name.** There is no
  "promote the bucket" operation. Multi-region S3 means *two buckets*, one in
  each region, plus a **Cross-Region Replication (CRR)** rule pushing objects
  from primary to standby. The standby bucket must exist and be populated long
  before failover.
- **CRR only replicates objects written *after* the rule exists.** Everything
  already in the live bucket needs an explicitly invoked **S3 Batch Replication**
  job. This is the single most important fact in this note and the step teams
  forget. `terraform apply` will not do it for you — the provider explicitly
  refuses (`existing_object_replication` returns `MalformedXML`).
- **RPO 2h is easy; the 15-minute RTC SLA is not an RTO.** Plain CRR is
  best-effort and comfortably inside 2h for normal traffic. **S3 RTC** buys a
  contractual *replication-time* commitment — but the SLA is *"99.9% of objects
  within 15 minutes, measured monthly, with a service-credit remedy"*, not a
  guarantee that the last object written before an outage is in the standby. See
  [RTC vs the RTO](#rtc-vs-the-rto-they-are-not-the-same-fifteen-minutes).
- **Your replica is not a backup.** By default delete markers are **not**
  replicated on a V2 rule, and a delete of a *specific version ID* is **never**
  replicated. Teams routinely believe the opposite in both directions. See
  [Delete markers and the replica-deletion trap](#delete-markers-and-the-replica-deletion-trap).
- **The bucket name is the failover problem, not the data.** As of **12 March
  2026** S3 supports **account regional namespaces**
  (`prefix-<accountid>-<region>-an`), which finally makes standby bucket names
  deterministic and templatable — but existing buckets cannot be converted, and
  `bucket_namespace` is `ForceNew` in Terraform. Every consumer with a hardcoded
  bucket name still breaks at failover.

---

## Does this service cross regions at all?

Partly, and the parts that do are not the parts you'd guess.

| Aspect | Regional or global? | Consequence for failover |
|---|---|---|
| Bucket *data* | **Regional.** Objects live in one region. | Needs replication. No "promote". |
| Bucket *name* | **Partition-global by default**, or account-regional since Mar 2026 | Standby bucket cannot reuse the primary's global name. |
| `CreateBucket` / `DeleteBucket` control plane | **Has a `us-east-1` dependency** for name uniqueness even when the call targets another region | Do not create buckets at failover time. See [[aws-regional-outages]]. |
| ~24 `PutBucket*` / `DeleteBucket*` config APIs | **`us-east-1` dependency** — includes `PutBucketReplication`, `PutBucketVersioning`, `PutBucketPolicy`, `PutBucketEncryption`, `PutBucketNotification` | You cannot reliably *reconfigure* a bucket during a `us-east-1` event. Configure ahead of time. |
| Data plane (`GetObject`/`PutObject`) | Regional endpoint, no cross-region dependency | Good. A healthy standby bucket serves reads even if the primary region is gone. |
| Multi-Region Access Points control plane | **`us-west-2` only** | A cross-region dependency you are adding, not removing. |
| MRAP *failover* control plane | Only `us-east-1`, `us-west-2`, `ap-southeast-2`, `ap-northeast-1`, `eu-west-1` | No Canadian region. The CA pair cannot fail an MRAP over from inside Canada. |

The `us-east-1` control-plane dependency list is captured in
[[aws-regional-outages]] and is the reason this note is adamant that **nothing
about the standby bucket may be created or modified during a failover**.

### Is the resource addressable from the other region?

Yes, and that is a trap rather than a feature. `s3://bucket-in-eu-west-1` is
reachable from `eu-west-2` using a regional endpoint with the correct region in
the SigV4 signature, and the SDKs will follow the `PermanentRedirect` /
`x-amz-bucket-region` hint. So an application failed over to `eu-west-2` with a
hardcoded primary bucket name **will keep working right up until the primary
region is actually unreachable** — which is precisely the scenario you failed
over for. The failure mode is invisible in a DR test that only checks "does the
app come up in the standby region". Your DR test must block network egress to
the primary region's S3 endpoints, or it proves nothing.

---

## Replication / mirroring options

| Option | What it guarantees | RPO | Handles existing objects | Cost shape | Verdict here |
|---|---|---|---|---|---|
| **Do nothing — deploy an empty standby bucket** | Nothing | ∞ | No | Free | Correct *only* for genuinely reconstructible buckets (build artefacts you can re-publish, caches). Say so explicitly per bucket. |
| **CRR, plain** | Async, best-effort, no time commitment | Typically seconds–minutes; no ceiling | **No** — Batch Replication required | $0.02/GB inter-region DT + destination PUTs + destination storage | **Default recommendation** for most buckets. Meets RPO 2h with large margin. |
| **CRR + S3 RTC** | 99.9%/month within 15 min, SLA-backed, plus replication metrics and `OperationMissedThreshold` events | ≤15 min for 99.9% of objects | **No** — Batch Replication required | plain CRR **+ $0.015/GB** surcharge + CloudWatch custom-metric charges | Reserve for the small set of buckets where a 2h RPO is genuinely not enough, or where you want the metrics/events and are happy to pay for them. |
| **S3 Batch Replication** | On-demand replication of *existing* objects, per-object success/failure report | N/A (one-shot) | **Yes — this is its entire purpose** | $0.25/job + $1 per million objects + manifest generation + DT + PUTs | **Mandatory** as the one-off migration step for every live bucket. |
| **S3 Batch Copy** | Copies objects in place or cross-bucket, creating *new versions* | N/A | Yes | Same Batch Operations pricing + COPY request cost | The escape hatch when Batch Replication refuses (objects deleted by version ID from the destination). |
| **AWS Backup for S3** | Point-in-time backups, cross-region copy of recovery points | Backup-plan frequency | Yes (continuous/periodic backup) | Backup storage + restore | Complementary, not a substitute. Covers *deletion*, which replication does not. See [[aws-backup]]. |
| **DataSync / `aws s3 sync`** | Whatever you script | Whatever you script | Yes | EC2/DataSync task cost + DT | No. Reinvents Batch Replication badly and has no per-object status. |
| **Multi-Region Access Point** | A single global alias in front of both buckets, with a failover control | N/A (routing, not replication) | N/A | $0.0033/GB routed + per-GB DT | See [Multi-Region Access Points](#multi-region-access-points-worth-it-here). Recommendation: **no**. |

### What replication actually copies

Per [What does Amazon S3 replicate?](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-what-is-isnot-replicated.html):

**Replicated:** objects created after the rule exists; unencrypted, SSE-S3,
SSE-KMS, DSSE-KMS and SSE-C objects; object metadata; object tags; object ACL
updates; S3 Object Lock retention information; object annotations.

**Not replicated (the list that bites):**

- Objects that existed before the configuration. (Batch Replication.)
- Objects that are themselves replicas created by another rule — replication is
  **not transitive**. A → B and B → C does not give you A's objects in C.
- Objects already replicated to a *different* destination. Changing the
  destination bucket in an existing rule does **not** re-replicate.
- **Bucket-level subresources.** Lifecycle configuration, notification
  configuration, CORS, bucket policy — *none* of these replicate. Your standby
  bucket's configuration is entirely Terraform's job. This is fine for us (we
  want it in Terraform anyway) but it means "the replica is identical to the
  source" is false at the configuration level.
- **Actions performed by lifecycle configuration.** If only the source has an
  expiry rule, S3 creates delete markers for expired objects and does *not*
  replicate them — the replica silently grows relative to the source. Put the
  same lifecycle config on both, or accept divergence and budget for it.
- Glacier Flexible Retrieval / Deep Archive objects, and Intelligent-Tiering
  Archive Access / Deep Archive Access tiers. Restore first.
- Objects the bucket owner lacks permission to read.
- With tag-based rules: objects tagged *after* `PutObject`. Live replication
  only sees tags supplied in the `PutObject` call. Retro-tagging does nothing.

---

## RTC vs the RTO: they are not the same fifteen minutes

This is the distinction the note exists to make sharply, because "S3 RTC gives
you 15 minutes and our RTO is 15 minutes" is a sentence that sounds like an
answer and is not one.

### What the documentation claims

> "S3 RTC replicates most objects that you upload to Amazon S3 in seconds, and
> 99.9 percent of those objects within 15 minutes."
> — [Meeting compliance requirements with S3 Replication Time Control](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-time-control.html)

### What the SLA actually commits to

From the [Amazon S3 Replication Time Control Feature Service Level
Agreement](https://aws.amazon.com/s3/sla-rtc/) (Last Updated: **May 4, 2022**):

> "AWS will use commercially reasonable efforts to make the RTC Feature meet a
> Monthly 15-minute Replication Percentage for each region pair per account"

and the metric is defined as:

> "calculated by subtracting from 100% the percentage of the objects replicated
> by the RTC Feature that did not successfully complete replication within 15
> minutes in each region pair per account during the monthly billing cycle in
> which the replication was initiated"

with this remedy table:

| Monthly 15-minute Replication Percentage | Service credit |
|---|---|
| < 99.9% but ≥ 98.0% | 10% |
| < 98.0% but ≥ 95.0% | 25% |
| < 95.0% | 100% |

### Why that is not an RTO

Read the four properties of that commitment carefully:

1. **It is a monthly aggregate, not a per-object guarantee.** 99.9% of objects
   within 15 minutes over a billing month means that, at a million objects a
   month, up to a thousand objects per month may be *arbitrarily* late and AWS
   is still fully inside its SLA. There is no stated ceiling on how late the
   0.1% are.
2. **The remedy is a service credit, not data.** The SLA pays you back a
   percentage of the RTC fee, the replication requests, the inter-region data
   transfer and the destination storage for affected objects. It does not
   produce the missing object in the standby region at 03:00.
3. **The exclusions swallow the disaster case.** The SLA does not apply to
   issues *"related to the performance of Amazon S3 … in either the source or
   destination AWS region"*, nor to anything *"caused by factors outside of our
   reasonable control"*. A regional S3 impairment — the exact event you are
   failing over for — is excluded. The SLA covers steady-state replication
   quality, not disasters.
4. **Two documented self-inflicted exclusions.** The SLA "doesn't apply to time
   periods when Amazon S3 performance guidelines on requests per second are
   exceeded", and it "doesn't apply during time periods where your replication
   data transfer rate exceeds the default 1 gigabit per second (Gbps) quota".
   A bulk ingest or a Batch Replication backfill running concurrently can push
   you over both and void the SLA for that period. **1 Gbps is roughly 450
   GB/hour** — a number worth checking against your actual write rate before
   assuming RTC applies to you at all.

### The framing that is correct

- **RTC is an RPO instrument.** It bounds (statistically, with a credit remedy)
  how far behind the standby copy is. Our RPO is 2 hours. Plain CRR already
  clears that by two orders of magnitude in normal operation.
- **RTO is a different clock entirely.** S3's contribution to RTO is
  approximately zero *provided the standby bucket already exists and is already
  populated*. There is no promotion step, no endpoint to swing, no capacity to
  warm. The S3 part of failover is "point the application at the other bucket
  name", which is a config change measured in seconds.
- **Therefore: RTC does not help the RTO, and the RPO does not need it.** The
  honest recommendation is that RTC is *not* required for the 2h RPO. Buy it
  where you want the SLA as a compliance artefact, or where a specific bucket has
  a much tighter real RPO than the blanket 2h.

**However** — and this is the one real argument for turning it on — **enabling
RTC automatically enables S3 Replication metrics**, which is how you observe lag
at all. You can enable replication metrics *independently* of RTC, at the same
CloudWatch custom-metric cost and without the $0.015/GB surcharge. So the
correct move is: **enable replication metrics everywhere, enable RTC only where
justified.** Most teams that "turned on RTC for observability" were paying a
per-GB surcharge for a feature they could have had for the CloudWatch cost
alone.

---

## RPO / RTO analysis

### Against RPO 2 hours

| Mechanism | Steady-state lag | Meets RPO 2h? |
|---|---|---|
| Plain CRR, small objects, normal write rate | seconds to low minutes (no published ceiling) | **Yes**, with enormous margin |
| CRR + RTC | 99.9% ≤ 15 min/month | Yes |
| Plain CRR, multi-GB objects | "For large objects, it can take several hours" (AWS docs) | **Conditional — this is the failure case.** |
| Delete marker replication | Explicitly outside the RTC SLA | Best-effort only |
| Batch Replication backfill | Hours to days depending on object count | N/A (one-off) |

The one place RPO 2h is genuinely at risk on plain CRR is **large objects**. The
docs are unambiguous:

> "The time it takes for Amazon S3 to replicate an object depends on the size of
> the object. For large objects, it can take several hours."
> — [Requirements and considerations for replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-requirements.html)

If any bucket in scope holds multi-GB objects (media masters, database dumps,
ML model artefacts), the 2h RPO for *that bucket* is not automatically met and
needs a measurement, not an assumption. RTC does not fix this either — an object
that takes three hours to transfer will miss the 15-minute threshold and simply
count against the 0.1%.

**Action:** for each in-scope bucket, record p99 object size. Anything with a p99
over ~1 GB gets an explicit RPO measurement rather than an inherited one.

### Against RTO 15 minutes

S3's contribution to the RTO budget, assuming the standby bucket is
pre-provisioned and pre-populated:

| Step | Time | Automated? |
|---|---|---|
| Decide to fail over | Human | No — see [[failover-orchestration]] |
| Application resolves standby bucket name | 0 s if from SSM Parameter Store / env at startup | Yes ([[aws-ssm-parameter-store]]) |
| Standby bucket serves reads | Immediate | Yes |
| Standby bucket accepts writes | Immediate *if* the bucket policy and IAM roles in the standby region already permit it | Yes, if pre-provisioned ([[aws-iam]]) |
| KMS decrypt of replicas | Immediate *if* the destination key and grants exist | Yes, if pre-provisioned ([[aws-kms]]) |
| Reverse the replication rule | **Not required at failover.** Defer to failback. | N/A |

**Verdict: S3 meets RTO 15 minutes comfortably, and contributes near-zero to the
budget, on three pre-provisioning conditions.**

1. The standby bucket exists, with versioning, encryption, policy, lifecycle,
   CORS and notification configuration already applied by Terraform.
2. The standby-region KMS key exists and the standby-region compute roles have
   `kms:Decrypt` on it.
3. No consumer anywhere holds a hardcoded primary bucket name.

Condition 3 is the one that fails in practice, and it fails in places nobody
greps: CloudFormation/Terraform outputs consumed by other stacks, CI/CD
pipelines, Athena table `LOCATION` clauses, Glue catalogue entries, `presigned
URL` generators, mobile clients with a baked-in CDN/bucket host, partner
integrations, and SQL in a data warehouse. See [[lessons-and-antipatterns]].

---

## Warm standby shape

What exists in the standby region while the primary is healthy:

| Thing | State while idle | Cost while idle |
|---|---|---|
| Standby bucket | Exists, versioning on, encryption on, policy applied | $0 for the bucket itself |
| Replicated objects | Present and current | **Full duplicate storage** — the dominant cost |
| Destination KMS key | Exists, enabled | $1/month per key + per-request |
| Replication IAM role | Exists in the primary account, assumed by `s3.amazonaws.com` | $0 |
| Replication rule on the source bucket | Enabled | $0 (the data transfer is the cost) |
| CloudWatch alarms on `ReplicationLatency` / `OperationsPendingReplication` | Armed **in the destination region** | ~$0.10/alarm/month + metric cost |
| Reverse replication rule (standby → primary) | **Not created.** Created at failback time, or created-but-disabled. | $0 |
| S3 Storage Lens | Optional | Free tier is free; advanced metrics $0.20 per million objects/month |

There is **nothing to scale to zero** — S3 has no idle capacity. The standby
costs exactly what the duplicated bytes cost, which makes storage class the only
meaningful cost lever. See [Cost](#cost).

---

## Versioning is a hard prerequisite, and turning it on is not free

> "Both source and destination buckets must have versioning enabled."
> — [Requirements and considerations for replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-requirements.html)

There is no replication without versioning. If the live buckets are unversioned
today — and in a mature estate some will be — enabling versioning is the first
migration step, and it changes behaviour on a live bucket in four ways.

### 1. Lifecycle rules stop meaning what they meant

This is the change that costs money silently. AWS states it plainly:

> "If you have an object Expiration lifecycle configuration, after you enable
> versioning, add a `NonCurrentVersionExpiration` policy to maintain the same
> permanent delete behavior as before you enabled versioning."
>
> "If you have a Transition lifecycle configuration, after you enable
> versioning, consider adding a `NonCurrentVersionTransition` policy."

Before versioning, `Expiration` deleted the object. After versioning,
`Expiration` on a current version just adds a **delete marker** and the old
version becomes noncurrent and **keeps being billed indefinitely**. A bucket
with a 30-day expiry rule and a high overwrite rate will grow without bound
after versioning is switched on, and nobody will notice until the bill.

**Every lifecycle configuration on every bucket being versioned must be audited
and extended with `noncurrent_version_expiration` in the same change.** Treat
this as part of the versioning step, not a follow-up.

### 2. `DeleteObject` semantics change for consumers

Any application that deletes objects and then re-`PutObject`s the same key, or
that lists a bucket and expects deleted keys to be gone, now sees delete markers
and version IDs. `ListObjectsV2` hides delete-marked keys so most code is fine;
code using `ListObjectVersions`, or code that relies on a `404` after delete, may
not be. Audit for `list-object-versions` usage before enabling.

### 3. The 15-minute write-quiesce recommendation

The Terraform provider documentation for `aws_s3_bucket_versioning` carries the
AWS guidance verbatim:

> "If you are enabling versioning on the bucket for the first time, AWS
> recommends that you wait for 15 minutes after enabling versioning before
> issuing write operations (PUT or DELETE) on objects in the bucket."

This is a *recommendation*, not an enforced gate, and it is the only part of the
whole S3 migration that touches "downtime". In practice it means: enable
versioning in a low-traffic window, then wait 15 minutes before enabling the
replication rule. It does not require you to stop writes — but writes in that
window may not get well-formed version IDs, and those objects are the ones that
will show up as replication failures later. Budget a maintenance window even
though nothing is technically down.

### 4. Versioning can no longer be turned off

> "If you attempt to disable versioning on the source bucket, Amazon S3 returns
> an error. You must remove the replication configuration before you can disable
> versioning on the source bucket."
>
> "If you disable versioning on the destination bucket, replication fails. The
> source object has the replication status `FAILED`."

Note also that S3 versioning can only ever be **suspended**, never truly
disabled once used — existing versions remain and remain billed. Enabling
versioning is a one-way door with a permanent cost floor. That is acceptable
here (we need it), but it should be a conscious decision recorded per bucket,
not a side effect of "we turned on replication".

### Terraform ordering

`aws_s3_bucket_versioning` must be applied and settled on **both** buckets before
`aws_s3_bucket_replication_configuration` is applied. Express this with an
explicit `depends_on` rather than relying on implicit graph ordering through the
bucket ID — the implicit edge exists but does not encode the 15-minute
recommendation or the cross-provider ordering.

---

## Delete markers and the replica-deletion trap

This section is where most "our replica is our backup" beliefs die. Get the
matrix exactly right.

| Operation on the source | Default replication behaviour | Can you change it? |
|---|---|---|
| `PutObject` (new object or overwrite) | Replicated | — |
| `DeleteObject` **without** version ID (adds a delete marker), **V2 rule** (i.e. `filter` present) | **Delete marker is NOT replicated** | Yes — `delete_marker_replication { status = "Enabled" }` |
| `DeleteObject` **without** version ID, **V1 rule** (no `filter`, `prefix` only) | **Delete marker IS replicated** (user-initiated deletes) | Only by migrating to a V2 rule |
| `DeleteObject` **with** a version ID | **NEVER replicated.** The version is removed from the source only. | **No. There is no setting.** |
| Delete marker created by a **lifecycle expiration rule** | **NEVER replicated**, in either V1 or V2 | No |
| Delete marker on a **tag-based** rule | **Not supported at all** | No |
| Delete marker replication latency | **Explicitly excluded from the RTC 15-minute SLA** | No |

AWS's own words on the two load-bearing rows:

> "If you specify an object version ID to delete in a `DELETE` request, Amazon S3
> deletes that object version in the source bucket. But it doesn't replicate the
> deletion in the destination buckets. In other words, it doesn't delete the same
> object version from the destination buckets. This protects data from malicious
> deletions."
> — [What does Amazon S3 replicate?](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-what-is-isnot-replicated.html)

> "Delete marker replication isn't supported for tag-based replication rules.
> Delete marker replication also doesn't adhere to the 15-minute service-level
> agreement (SLA) that's granted when you're using S3 Replication Time Control
> (S3 RTC)."
> — [Replicating delete markers between buckets](https://docs.aws.amazon.com/AmazonS3/latest/userguide/delete-marker-replication.html)

> "If you enable delete marker replication and your source bucket has an S3
> Lifecycle expiration rule, the delete markers added by the S3 Lifecycle
> expiration rule won't be replicated to the destination bucket."

### The two opposite wrong beliefs

**Wrong belief A: "the replica is a backup, so a malicious mass-delete can't
hurt us."**
False if you enabled delete marker replication — which many teams do, because
otherwise the buckets diverge and the divergence looks like a bug. With delete
marker replication on, `aws s3 rm --recursive` propagates. The objects are
recoverable (the versions still exist under the delete markers on both sides)
but a *permanent* delete loop that removes version IDs will empty the source and
leave the replica intact-but-diverged.

**Wrong belief B: "our buckets are identical, so we can compare object counts to
prove replication health."**
False in the default configuration. With V2 rules and delete markers not
replicated, the destination accumulates objects the source has delete-marked.
Add lifecycle expiry on the source only, and the destination grows monotonically.
Object-count parity is **not** a valid replication health check. Use
`OperationsPendingReplication` and `OperationsFailedReplication` instead (see
[Monitoring](#monitoring-replication-lag)).

### The decision, and the recommendation

There are exactly two coherent postures. Pick one **per bucket class** and write
it down; the incoherent middle is where incidents come from.

| Posture | Config | You get | You lose |
|---|---|---|---|
| **Mirror** | `delete_marker_replication = Enabled`, same lifecycle config on both buckets, `noncurrent_version_expiration` on both | A genuine mirror. Failover to the standby gives users the same view. Storage costs stay bounded. | Deletes propagate. The replica is not a ransomware/fat-finger backstop. |
| **Archive** | `delete_marker_replication = Disabled` (the default), no expiry on the destination | The destination retains everything ever written. Genuine protection against source-side deletion. | The destination diverges from the source and grows without bound. After failover, users see objects they deleted months ago reappear. **This is a correctness bug, not just a cost bug.** |

**Recommendation: Mirror.** The target posture in
[[00-meta/research-brief|the brief]] is *active/passive warm standby* — the
standby exists to become the primary. A standby that resurrects deleted objects
on promotion is not a warm standby, it is a landmine. Enable delete marker
replication, mirror the lifecycle configuration onto the destination, and get
deletion protection from the mechanism that is actually designed for it:
**[[aws-backup]] with cross-region copy**, plus S3 Versioning + MFA Delete or an
Object Lock retention policy where the data warrants it.

Record the exception list explicitly: audit-log buckets (CloudTrail, access
logs) should be **Archive** posture with Object Lock, because for those buckets
"deletes must not propagate" is the entire point. See
[Special-case buckets](#special-case-buckets).

---

## KMS-encrypted objects across regions

KMS keys are regional and **AWS KMS keys aren't shared outside the AWS Region in
which they were created** — the replica is decrypted in the source region and
re-encrypted in the destination region with a *different* key. Three things must
line up. See [[aws-kms]] for the key-policy detail.

### 1. Opt in on the source side

By default **S3 does not replicate SSE-KMS or DSSE-KMS objects at all.** You must
explicitly opt in:

```hcl
source_selection_criteria {
  sse_kms_encrypted_objects { status = "Enabled" }
}
```

Omit this and KMS-encrypted objects are silently skipped. `PutBucketReplication`
returns `200`. Nothing is replicated. Nothing alarms unless you have replication
metrics on. **This is the quietest failure mode in the whole service.**

### 2. Name a destination key that lives in the destination region

```hcl
destination {
  bucket = aws_s3_bucket.standby.arn
  encryption_configuration {
    replica_kms_key_id = aws_kms_key.standby.arn   # MUST be a key in the destination region
  }
}
```

AWS is explicit and the failure mode is nasty:

> "The KMS key *must* be valid. The `PutBucketReplication` API operation doesn't
> check the validity of KMS keys. If you use a KMS key that isn't valid, you will
> receive the HTTP `200 OK` status code in response, but replication fails."
> — [Replicating encrypted objects](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-config-for-kms-objects.html)

A `terraform apply` that passes an `eu-west-1` key ARN into an `eu-west-2`
destination **succeeds**. Replication then fails per-object, forever, and you
find out from `OperationsFailedReplication` — if you enabled it. Put a
`validation` block or at minimum a `precondition` on the module asserting that
the destination key ARN's region matches the destination bucket's region.

### 3. Grant the replication role on both keys, with `kms:ViaService` scoped to the right region

The replication role is a single IAM role in the source account, assumed by
`s3.amazonaws.com`. It needs `kms:Decrypt` on the **source-region** key and
`kms:Encrypt` on the **destination-region** key, and AWS's own example scopes
each with a region-specific `kms:ViaService`:

```json
{
  "Action": ["kms:Decrypt"],
  "Effect": "Allow",
  "Condition": { "StringLike": {
      "kms:ViaService": "s3.eu-west-1.amazonaws.com",
      "kms:EncryptionContext:aws:s3:arn": ["arn:aws:s3:::SOURCE-BUCKET/*"] }},
  "Resource": ["arn:aws:kms:eu-west-1:111122223333:key/SOURCE-KEY-ID"]
},
{
  "Action": ["kms:Encrypt"],
  "Effect": "Allow",
  "Condition": { "StringLike": {
      "kms:ViaService": "s3.eu-west-2.amazonaws.com",
      "kms:EncryptionContext:aws:s3:arn": ["arn:aws:s3:::DEST-BUCKET/*"] }},
  "Resource": ["arn:aws:kms:eu-west-2:111122223333:key/DEST-KEY-ID"]
}
```

Copy-pasting the source statement and forgetting to change `s3.eu-west-1` to
`s3.eu-west-2` in the destination statement is the single most common S3
replication IAM bug. It produces a `200 OK` on the config and per-object
failures at runtime.

Additional permission requirements worth calling out:

- Use **`s3:GetObjectVersionForReplication`**, not `s3:GetObjectVersion`. AWS is
  explicit that `s3:GetObjectVersion` "allows replication of unencrypted and
  SSE-S3-encrypted objects, but not of objects that are encrypted by using KMS
  keys (SSE-KMS or DSSE-KMS)". A role built from an old blog post will replicate
  your plaintext objects and silently skip the encrypted ones.
- If the destination bucket has default SSE-KMS encryption and the source object
  is plaintext, the role additionally needs **`kms:GenerateDataKey`** for the
  destination context and key.
- If **Object Lock** is enabled on the source, the role needs
  `s3:GetObjectRetention` and `s3:GetObjectLegalHold`.
- If **S3 Bucket Keys** are enabled on either side, the KMS encryption context
  changes from the *object* ARN to the *bucket* ARN. Your `kms:EncryptionContext:aws:s3:arn`
  condition must be `arn:aws:s3:::bucket-name`, **without** the `/*`. Enabling
  Bucket Keys on a bucket that already has a working replication role will break
  replication until the policy is changed. Because Bucket Keys cut KMS request
  costs by up to 99%, teams enable them for cost reasons and break replication as
  a side effect.

### 4. Multi-Region KMS keys do not help here

This is the finding that matters for [[kms-when-to-use-multi-region-keys]]:

> "You can use multi-Region AWS KMS keys in Amazon S3. However, Amazon S3
> currently treats multi-Region keys as though they were single-Region keys, and
> does not use the multi-Region features of the key."
> — [Replicating encrypted objects](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-config-for-kms-objects.html)

So the classic reason to reach for a multi-Region key — "the replica is
encrypted under the same key material so I don't need a second key" — **does not
apply to S3**. S3 will decrypt with the source key and re-encrypt with the
destination key regardless. Using an MRK here costs you the MRK replica-key
pricing and buys nothing for S3. **Recommendation: use ordinary single-region
KMS keys for S3, one per region.** Reserve MRKs for the cases
[[kms-when-to-use-multi-region-keys]] identifies (client-side encryption
envelopes, DynamoDB Global Tables, Secrets Manager replicas) where the key
material genuinely has to be the same on both sides.

### 5. KMS throttling during the backfill

> "When you add many new objects with AWS KMS encryption after enabling
> Cross-Region Replication (CRR), you might experience throttling (HTTP `503
> Service Unavailable` errors)."

Replication makes KMS calls on your behalf and they count against your account's
KMS requests-per-second quota. AWS's rule of thumb: replicating 1,000 objects
per second consumes roughly **2,000** KMS requests per second. A Batch
Replication backfill of a large KMS-encrypted bucket will throttle your
*production* KMS traffic unless the quota is raised first. **Raise the KMS RPS
quota in both regions before running the backfill**, and run the backfill with
job priority set low. See [[aws-kms]].

("The backfill" is the S3 Batch Replication job described in the next section.
The two sections are a pair: the backfill is the single largest burst of KMS
traffic your account will ever generate, and it is the one you can schedule.)

---

## S3 Batch Replication for existing objects

**This is the section that makes the rest of the note actionable.** Everything
above describes steady-state replication of new writes. None of it moves the
petabyte already sitting in the live production bucket. That is a separate,
explicitly invoked, one-shot job, and it is the step that gets forgotten because
`terraform apply` returns green without it.

### Why Terraform cannot do this for you

Two independent gaps, both worth understanding because people keep looking for
the Terraform answer and there isn't one.

1. **`ExistingObjectReplication` is not a public API.** The field exists in the
   S3 replication XML schema and in the SDK models (it is visible in, for
   example, the [Kotlin SDK's `ReplicationRule`
   model](https://docs.aws.amazon.com/sdk-for-kotlin/api/latest/s3control/aws.sdk.kotlin.services.s3control.model/-replication-rule/existing-object-replication.html)),
   but AWS does not accept it from customers. The AWS CLI documentation for
   `put-bucket-replication` states plainly that the parameter "is not supported
   by Amazon S3 at this time" and that specifying it returns `MalformedXML`.
   The provider issue tracking this is
   [hashicorp/terraform-provider-aws#43746](https://github.com/hashicorp/terraform-provider-aws/issues/43746).
   Internally AWS *does* set this field — it is how the console's "replicate
   existing objects?" checkbox works — but the customer-facing path is the Batch
   Operations job.
2. **There is no `aws_s3control_job` resource.** S3 Batch Operations jobs have
   never been modelled in the `hashicorp/aws` provider. The original feature
   request is
   [#18538](https://github.com/hashicorp/terraform-provider-aws/issues/18538)
   (April 2021, closed) and the current open request for a resource that mirrors
   `aws s3control create-job` is
   [#46231](https://github.com/hashicorp/terraform-provider-aws/issues/46231)
   (opened 30 January 2026, not implemented as of this writing). Even if it
   existed, a Batch Operations job is a *one-shot imperative action with a
   terminal state*, not a declarative resource — modelling it in Terraform would
   be a category error. You would end up with a resource that Terraform wants to
   recreate every time the bucket contents change.

**Consequence for the monorepo:** the backfill is a runbook step, not a
Terraform step. Terraform's job is to pre-create the two IAM roles, the manifest
bucket and the completion-report bucket so that the runbook step is a single
`aws s3control create-job` invocation with no console clicking. That split is
written up under [Terraform implementation](#terraform-implementation).

### Preconditions

From [Replicating existing objects with Batch
Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-batch-replication-batch.html):

> "Your source bucket must have an existing replication configuration."

So the order is fixed and non-negotiable:

1. Versioning on both buckets (and the 15-minute settle — see above).
2. Destination bucket, destination KMS key, destination bucket policy.
3. Replication role with source *and* destination KMS grants.
4. `aws_s3_bucket_replication_configuration` applied and **propagated**. AWS:
   *"If you recently added or updated the replication configuration on the
   source bucket, expect a delay of a few minutes before the change is fully
   propagated. We recommend waiting before creating a Batch Replication job with
   an S3-generated manifest."* In practice: wait 15 minutes, then verify by
   writing a canary object and confirming it lands in the destination.
5. **Then** the Batch Replication job.

Skipping step 4's wait produces a job whose S3-generated manifest is built from
a stale replication configuration — it will silently under-scope.

### Manifests: generated vs supplied

This is the first real decision in the section.

| | **S3-generated manifest** | **User-supplied manifest** |
|---|---|---|
| What it is | S3 walks the source bucket at job-creation time and writes an inventory-format manifest | An existing S3 Inventory report, or a CSV you produce |
| Scope | *"the objects listed use the same source bucket, prefix, and tags as your replication configurations on the source bucket"* — i.e. it inherits the replication rule's filter automatically | Exactly what is in your file, and **nothing else** |
| Versions | *"With a generated manifest, Amazon S3 replicates all eligible versions of your objects."* | *"If the objects in your manifest are in a versioned bucket, you must specify the version IDs for the objects. Only the object with the version ID that's specified in the manifest will be replicated."* |
| Where it must live | *"the manifest must be stored in the same AWS Region as the source bucket"* | Same-region bucket you own |
| Extra IAM | `s3:PutInventoryConfiguration` and `s3:GetReplicationConfiguration` on the source bucket, plus `s3:PutObject` on the manifest bucket | Only `s3:GetObject`/`s3:GetObjectVersion` on the manifest bucket |
| Extra cost | `$0.015 per 1 million objects in source bucket` (usage type `*-BatchOperations-Manifest`) | Free if you already run S3 Inventory; S3 Inventory itself is `$0.0025 per 1 million objects listed` in eu-west-1, `$0.0028` in ca-central-1 |
| Latency to start | S3 must scan the bucket before the job starts. For a very large bucket this is itself hours. | Instant if the inventory report already exists |

**Recommendation: S3-generated manifest for the first backfill of every bucket.**
The "all eligible versions" behaviour is what you want for a DR mirror, the
filter inheritance removes an entire class of scoping mistake, and $0.015 per
million objects is noise. Switch to a user-supplied manifest only when you are
doing something the generated manifest cannot express — most commonly chunking
one enormous bucket into several parallel jobs by key range, which is the
technique in AWS's [Accelerate Amazon S3 Replication with automated S3 Batch
Operations
parallelization](https://aws.amazon.com/blogs/storage/accelerate-amazon-s3-replication-with-automated-s3-batch-operations-parallelization/)
solution.

Note the hard ceiling: **"A single Batch Replication job can support a manifest
with up to 20 billion objects."** Above that you must split the job, and the
parallelization blog above notes that buckets exceeding 20 billion objects
trigger automatic S3 Inventory report configuration in its Step Functions
orchestration. The screenshot in that post is from *"a migration of an S3 bucket
with over 50 billion objects"* — that is the largest publicly documented Batch
Replication migration I could find, and AWS does not publish how long it took.

### Scoping and filters

Two filters, and getting them right is the difference between a $200 job and a
$20,000 one.

**1. `ObjectReplicationStatuses`** — one or more of `NONE`, `FAILED`,
`COMPLETED`, `REPLICA`. The AWS mapping, verbatim in intent:

| Goal | Filter |
|---|---|
| First backfill: replicate everything never attempted | `["NONE"]` |
| Retry only what failed (the re-run after fixing a KMS grant) | `["FAILED"]` |
| Backfill + retry in one pass | `["NONE","FAILED"]` |
| Backfill a *second* destination that already has a first destination | include `"COMPLETED"` |
| Re-replicate objects that are themselves replicas | include `"REPLICA"` |

**If you supply no filter, Batch Operations attempts to replicate every object
in the manifest regardless of status** — including the ones already successfully
replicated. On a re-run that is a full-price second copy of the entire bucket in
data transfer and destination PUTs, for zero benefit. **Always set
`ObjectReplicationStatuses` explicitly.** Treat an unfiltered Batch Replication
job as a production incident waiting to be billed.

**2. Object creation date** — `ObjectCreationTime` lets you bound the job to
objects older than the moment live replication was switched on. Combined with
`["NONE"]` this is belt and braces: live replication handles everything from
time T onwards, the batch job handles everything before T, and there is no
overlap to pay for twice.

### The IAM role (there are two of them, and people conflate them)

| Role | Trusted principal | What it does |
|---|---|---|
| **Replication role** | `s3.amazonaws.com` | Reads objects from source, writes to destination, KMS decrypt/encrypt. Already exists — it is the `role` on `aws_s3_bucket_replication_configuration`. |
| **Batch Operations role** | `batchoperations.s3.amazonaws.com` | Runs the *job*: reads/generates the manifest, calls `s3:InitiateReplication`, writes the completion report. **Does not touch object data.** |

Batch Operations trust policy, verbatim from [Configuring an IAM role for S3
Batch
Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-batch-replication-policies.html):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "batchoperations.s3.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Permissions policy **when S3 generates the manifest** (the variant you want):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow",
      "Action": ["s3:InitiateReplication"],
      "Resource": ["arn:aws:s3:::SOURCE-BUCKET/*"] },
    { "Effect": "Allow",
      "Action": ["s3:GetReplicationConfiguration", "s3:PutInventoryConfiguration"],
      "Resource": ["arn:aws:s3:::SOURCE-BUCKET"] },
    { "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:GetObjectVersion"],
      "Resource": ["arn:aws:s3:::MANIFEST-BUCKET/*"] },
    { "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": ["arn:aws:s3:::COMPLETION-REPORT-BUCKET/*",
                   "arn:aws:s3:::MANIFEST-BUCKET/*"] }
  ]
}
```

If you supply your own manifest, drop the `s3:GetReplicationConfiguration` /
`s3:PutInventoryConfiguration` statement and drop the manifest bucket from the
`s3:PutObject` statement. AWS flags this explicitly: *"Your IAM role for Batch
Replication needs different permissions, depending on whether you are generating
a manifest or supplying one."* Using the generated-manifest policy with a
supplied manifest grants `s3:PutInventoryConfiguration` you don't need; using
the supplied-manifest policy with a generated manifest makes the job fail at
creation.

Three permission traps beyond the documented policy:

- **`s3:InitiateReplication` is on the object ARN** (`bucket/*`), not the bucket
  ARN. Putting it on the bucket ARN produces an access-denied on every task.
- **The *replication* role, not the Batch role, still needs to be correct.** AWS:
  *"make sure to update your replication configuration by granting the IAM role
  that's attached to the replication rule the proper permissions."* A backfill
  will happily start and then fail every single task because the replication role
  is missing `kms:Encrypt` on the destination key. The completion report will
  tell you — which is why you always generate one.
- **SSE-S3 → SSE-KMS history.** AWS calls this out as its own bullet: *"If you
  use S3 Batch Replication to replicate datasets cross region and your objects
  previously had their server-side encryption type updated from SSE-S3 to
  SSE-KMS, you may need additional permissions. On the source region bucket, you
  must have `kms:decrypt` permissions. Then, you will need the `kms:decrypt` and
  `kms:encrypt` permissions for the bucket in the destination region."* A mature
  estate that switched a bucket from SSE-S3 to SSE-KMS at some point in its life
  has a mixed-encryption object population, and only the backfill will surface it.

### Completion reports — always on, always "all tasks"

> "When you create a Batch Replication job, you can request a CSV completion
> report. This report shows the objects, replication success or failure codes,
> outputs, and descriptions."

You get to choose "failed tasks only" or "all tasks". **Choose all tasks for the
migration backfill.** The failed-only report tells you what broke; the all-tasks
report is the artefact that proves the migration was complete, object by object,
and it is the only cheap way to answer "did key X get copied" six months later
during an audit. It is a CSV in S3; it costs storage and nothing else.

Failure codes are documented under [Amazon S3 replication failure
reasons](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-metrics-events.html)
and Batch-specific troubleshooting under [Batch Replication
errors](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-troubleshoot.html).
Budget for a second job filtered to `["FAILED"]` after the first one — on a real
bucket with mixed encryption history and a few million objects, a non-zero
failure count on the first pass is the normal outcome, not the exception.

### What Batch Replication will not do

- **Glacier Flexible Retrieval and Glacier Deep Archive objects are not
  supported.** Not "slow" — unsupported. If a bucket has lifecycle transitions
  into Glacier FR/DA, the archived portion of that bucket cannot be backfilled at
  all by this mechanism. Your options are (a) accept that the standby has only
  the non-archived tier, (b) restore then replicate then re-archive, which is
  expensive and slow, or (c) use [[aws-backup]] cross-region copy for the archive
  tier instead. **Decide this per bucket before you start, not after the job
  reports failures.**
- **S3 Intelligent-Tiering Archive Access / Deep Archive Access tiers** need a
  restore request first, and you must wait for the objects to move back to
  Frequent Access.
- **Objects deleted from the destination by version ID cannot be re-replicated.**
  AWS: *"Batch Replication doesn't support re-replicating objects that were
  deleted by specifying the version ID of the object from the destination
  bucket."* The escape hatch is a **Batch Copy** job that copies the source
  objects in place, creating new versions in the source, which then replicate
  normally. Note the side effect: this doubles the storage of every copied object
  in the *source* bucket until noncurrent-version expiry catches up. Also:
  *"Deleting and recreating the destination bucket doesn't initiate
  replication."*

### Turn off lifecycle rules while the job runs

An easy one to miss, and AWS spells out the exact race:

> "If you have S3 Lifecycle configured for your bucket, we recommend disabling
> your lifecycle rules while the Batch Replication job is active. Doing so helps
> ensure parity between the source and destination buckets."

The documented scenario: Batch Replication may replicate a delete marker to the
destination *before* it replicates the object versions underneath it; if both
buckets have an "remove expired delete markers" lifecycle rule, the destination
expires the delete marker before the versions arrive, and the two buckets end up
structurally different. On a multi-day backfill this is not a theoretical race.

Practical consequence for the runbook: disabling a lifecycle rule on a
production bucket for the duration of a multi-day job means transitions and
expirations stop happening for those days, and the bucket's storage bill
temporarily rises. Say so in the change ticket rather than discovering it on the
invoice.

### Concurrency and job priority

- *"If you submit multiple Batch Replication jobs for the same bucket within a
  short time frame, Amazon S3 runs those jobs concurrently."*
- *"If you submit multiple Batch Replication jobs for two different buckets, be
  aware that Amazon S3 might not run all jobs concurrently. If you exceed the
  number of Batch Replication jobs that can run at one time on your account,
  Amazon S3 pauses the lower priority jobs to work on the higher priority ones.
  After the higher priority jobs are completed, any paused jobs become active
  again."*

Two things follow. First, **set `--priority` deliberately** — it is the only
throttle you have. Give the migration backfill a *low* priority so that any
`["FAILED"]` retry job or any genuinely urgent job pre-empts it, and so that a
runaway backfill cannot starve other Batch Operations work. Second, **do not fire
all N bucket backfills at once** expecting them to run in parallel; they will
queue, and you will have no visibility into the ordering. Run them in a
deliberate sequence, smallest bucket first, so that the first one you run is the
one that teaches you what the failure modes are.

### How long does a large backfill actually take?

**Honest finding first: AWS publishes no throughput figure for S3 Batch
Replication, and I could not find a credible public benchmark.** The AWS launch
post says only:

> "existing objects can take longer to replicate than new objects, and the
> replication speed largely depends on the AWS Regions, size of data, object
> count, and encryption type"
> — [NEW – Replicate Existing Objects with Amazon S3 Batch
> Replication](https://aws.amazon.com/blogs/aws/new-replicate-existing-objects-with-amazon-s3-batch-replication/) (8 February 2022)

The parallelization blog post describes a real migration of *"over 50 billion
objects"* but publishes no duration. Anyone quoting you an objects-per-second
number for Batch Replication is guessing. **Plan the backfill as a measured
exercise: run it on your smallest bucket first and extrapolate from your own
numbers.**

What you *can* reason about are the two documented ceilings, and they bound the
answer usefully.

**Ceiling 1 — the 1 Gbps replication bandwidth quota.** S3 replication has a
default account-level data transfer rate quota of **1 Gbps**, raisable via
Service Quotas or a support case. This is the same quota named in the RTC SLA
exclusions. At 1 Gbps:

| Bucket size | Wall-clock at 1 Gbps (125 MB/s) | At a raised 10 Gbps |
|---|---|---|
| 1 TB | ~2.3 hours | ~14 minutes |
| 10 TB | ~23 hours | ~2.3 hours |
| 100 TB | ~9.6 days | ~23 hours |
| 1 PB | ~96 days | ~9.6 days |

Arithmetic, not a benchmark: 1 Gbps = 125 MB/s = 450 GB/hour = ~10.8 TB/day.
That number is the single most useful planning input in this section. **If your
largest bucket is above ~10 TB, raise the replication bandwidth quota before you
start or the backfill will run for days.**

**Ceiling 2 — the request rate.** AWS documents the per-object request
amplification: *"For each object replicated, Amazon S3 replication makes up to
five GET/HEAD requests and one PUT request to the source bucket, and one PUT
request to each destination bucket."* Those requests count against the standard
S3 request-rate guidelines (3,500 PUT/COPY/POST/DELETE and 5,500 GET/HEAD per
second per partitioned prefix) for **both** buckets, alongside your production
traffic. For a bucket of many small objects, this — not bandwidth — is the
binding constraint, and it is why a 100-million-small-object bucket can take
longer than a 10 TB bucket of large objects.

**The planning rule this note recommends:** compute both bounds (bytes ÷ 450
GB/hour, and objects ÷ a conservative few-hundred-per-second) and take the
larger. Then double it, because the first pass will have failures and you will
run a second `["FAILED"]` job.

### Cost of the backfill

Batch Operations pricing, from the AWS Price List API (`AmazonS3` offer,
eu-west-1 and ca-central-1 both identical on these three line items):

| Line item | Usage type | Price |
|---|---|---|
| Job fee | `EU-BatchOperations-Jobs` | **$0.25 per job** |
| Object operations | `EU-BatchOperations-Objects` | **$1.00 per 1 million object operations** |
| Generated manifest | `EU-BatchOperations-Manifest` | **$0.015 per 1 million objects in source bucket** |

Those are the *Batch Operations* charges only. The backfill also incurs the
ordinary replication charges, which dominate:

| Line item | Price (EU pair) | Price (CA pair) |
|---|---|---|
| Inter-region data transfer out | **$0.02/GB** (`EU-EUW2-AWS-Out-Bytes`, `InterRegion Outbound`) | **$0.02/GB** (`CAN1-CAN2-AWS-Out-Bytes`) |
| Destination PUT requests (Tier 1) | **$0.005 per 1,000** | **$0.0055 per 1,000** |
| Source GET requests (Tier 2) | **$0.004 per 10,000** | **$0.0044 per 10,000** |
| Destination storage (S3 Standard, first 50 TB) | **$0.024/GB-month** in eu-west-2 | **$0.025/GB-month** in ca-west-1 |
| RTC surcharge, if enabled | **$0.015/GB** (`EU-EUW2-S3RTC-Out-Bytes`) | **$0.015/GB** (`CAN1-CAN2-S3RTC-Out-Bytes`) |

**Worked example — a 10 TB bucket with 20 million objects, eu-west-1 →
eu-west-2, one backfill job, generated manifest, no RTC:**

| Component | Calculation | Cost |
|---|---|---|
| Batch job fee | 1 × $0.25 | $0.25 |
| Object operations | 20M × $1.00/M | $20.00 |
| Manifest generation | 20M × $0.015/M | $0.30 |
| Inter-region DT out | 10,240 GB × $0.02 | $204.80 |
| Destination PUTs | 20M × $0.005/1,000 | $100.00 |
| Source GETs (assume 1 GET/object) | 20M × $0.004/10,000 | $8.00 |
| **One-off backfill total** | | **≈ $333** |
| *Then, ongoing:* destination storage | 10,240 GB × $0.024 | **$245.76/month** |

The shape to internalise: **the one-off backfill is cheap; the duplicated
storage is forever.** $333 once versus $246 every month. Any conversation about
"can we afford multi-region S3" is a conversation about the monthly storage line,
not the migration. That is why [replica storage class](#replica-storage-class-as-a-cost-lever)
is the lever that matters.

Add RTC and the backfill's DT line goes from $204.80 to $358.40 (`$0.035/GB`
effective) — and remember the RTC SLA does not apply while the backfill pushes
you past 1 Gbps anyway. **Do not enable RTC before the backfill. Enable it, if at
all, after the backfill has drained.**

### The runbook command

```bash
# 1. Verify live replication is working before you backfill anything.
aws s3api put-object --bucket "$SRC" --key "_canary/$(date +%s)" --body /dev/null
sleep 120
aws s3api head-object --bucket "$DST" --key "_canary/..."   # must return 200

# 2. Create the backfill job. Note: --priority low, --no-confirmation-required
#    omitted deliberately so the job lands in Suspended and a human must resume it.
aws s3control create-job \
  --account-id "$ACCOUNT_ID" \
  --region eu-west-1 \
  --operation '{"S3ReplicateObject":{}}' \
  --priority 1 \
  --report '{
      "Bucket":"arn:aws:s3:::helios-s3-batchreports-111122223333-eu-west-1",
      "Prefix":"backfill/'"$SRC"'",
      "Format":"Report_CSV_20180820",
      "Enabled":true,
      "ReportScope":"AllTasks"
    }' \
  --manifest-generator '{
      "S3JobManifestGenerator": {
        "ExpectedBucketOwner":"111122223333",
        "SourceBucket":"arn:aws:s3:::'"$SRC"'",
        "EnableManifestOutput": true,
        "ManifestOutputLocation": {
          "ExpectedManifestBucketOwner":"111122223333",
          "Bucket":"arn:aws:s3:::helios-s3-batchmanifests-111122223333-eu-west-1",
          "ManifestFormat":"S3InventoryReport_CSV_20211130"
        },
        "Filter": {
          "EligibleForReplication": true,
          "ObjectReplicationStatuses": ["NONE"],
          "CreatedBefore": "2026-10-01T00:00:00Z"
        }
      }
    }' \
  --role-arn "arn:aws:iam::111122223333:role/helios-s3-batchreplication" \
  --description "Helios backfill $SRC -> $DST"

# 3. The job is created in 'Suspended' state. Inspect the generated manifest's
#    object count and total size BEFORE resuming — this is your last chance to
#    catch a mis-scoped job.
aws s3control describe-job --account-id "$ACCOUNT_ID" --region eu-west-1 --job-id "$JOB_ID"
aws s3control update-job-status --account-id "$ACCOUNT_ID" --region eu-west-1 \
  --job-id "$JOB_ID" --requested-job-status Ready
```

Leaving the job in `Suspended` and resuming it by hand is a deliberate safety
gate: `describe-job` after manifest generation tells you exactly how many objects
the job will touch, which is the number you multiply by $1/million and by your
KMS RPS quota before committing. **Never create a Batch Replication job with
`--no-confirmation-required` on a production bucket.**

---

## Bucket names are globally unique

The data is the easy half. The name is the hard half, and it is the half that
actually causes the outage.

### The rule, verbatim

> "General purpose buckets exist in a global namespace, which means that each
> bucket name must be unique across all AWS accounts in all the AWS Regions
> within a partition. A partition is a grouping of Regions. AWS currently has
> four partitions: `aws` (Standard Regions), `aws-cn` (China Regions),
> `aws-us-gov` (AWS GovCloud (US)), and `aws-eusc` (European Sovereign Cloud)."
>
> "When you create a general purpose bucket, you choose its name and the AWS
> Region to create it in. **After you create a general purpose bucket, you can't
> change its name or Region.**"
> — [General purpose bucket naming
> rules](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucketnamingrules.html)

Three consequences, in increasing order of how much they hurt:

1. The standby bucket **cannot** have the same name as the primary. There is no
   equivalent of DynamoDB Global Tables' shared table name, no equivalent of
   Secrets Manager's replica-with-the-same-name. Two buckets, two names, always.
2. Therefore **every consumer must be told which bucket to use**, at runtime,
   from configuration. A hardcoded bucket name is a failover bug with a long
   fuse.
3. Bucket names cannot be renamed. Whatever convention you adopt is permanent
   for every bucket created under it. Get it right once.

### Account regional namespaces (March 2026) — what actually changed

On **12 March 2026** AWS shipped [account regional namespaces for general purpose
buckets](https://aws.amazon.com/about-aws/whats-new/2026/03/amazon-s3-account-regional-namespaces),
*"at no additional cost"*. The mechanism:

> "The account regional namespace is a reserved subdivision of the global bucket
> namespace. Only your account can create general purpose buckets in this
> namespace. New general purpose buckets created in your account regional
> namespace are unique to your account. These buckets can never be re-created by
> another account."
> — [Namespaces for general purpose
> buckets](https://docs.aws.amazon.com/AmazonS3/latest/userguide/gpbucketnamespaces.html)

Name shape:

```
{bucket-name-prefix}-{accountId}-{region}-an
amzn-s3-demo-bucket-111122223333-us-west-2-an
```

You opt in per bucket with the `x-amz-bucket-namespace: account-regional` header
on `CreateBucket`, and you can *enforce* it estate-wide with the
`s3:x-amz-bucket-namespace` condition key in an SCP or RCP — AWS publishes ready
IAM/RCP/SCP examples on the same page.

**Why this matters enormously for Helios specifically.** The whole problem with
standby bucket naming used to be that you could not guarantee
`myapp-prod-eu-west-2` was free, so every convention degenerated into either a
central registry or a random suffix, and a random suffix means the standby name
is not derivable from the primary name. Account regional namespaces make the
standby name **a pure function of (prefix, account id, region)**. In a
cookiecutter monorepo that is exactly the shape you want: the module computes
both names from the same three inputs and nobody ever hand-writes a bucket name
again.

**The restrictions, all of which bite:**

- **Character budget.** *"Your account regional suffix counts towards the maximum
  number of 63-characters allowed in general purpose bucket names. So if your
  account regional suffix is `-012345678910-us-east-1-an`, then you have
  37-characters available for your bucket name prefix."* For `eu-west-1`
  (9 chars) the suffix is `-111122223333-eu-west-1-an` = 26 chars, leaving 37.
  For `ca-central-1` (12 chars) the suffix is 29 chars, leaving **34**. Your
  naming convention must fit the *longest* region code in your estate, which for
  Helios is `ca-central-1`. **Budget 34 characters for the prefix, not 37.**
- **Not every region.** *"You can create buckets in your account regional
  namespace in all AWS Regions except Middle East (Bahrain) and Middle East
  (UAE)."* Note this contradicts the launch announcement's "37 AWS Regions"; the
  user-guide page is the more current statement. Either way — **`ca-west-1`,
  `eu-west-2` and `us-west-2` are all supported.** This is one of the few
  `ca-west-1` parity checks in this vault that comes back clean. See
  [[region-pair-selection]].
- **Use regional endpoints.** *"You should use the S3 regional endpoints to
  create buckets in your account regional namespace."* The legacy global endpoint
  works only for `us-east-1`, for backwards compatibility.
- **Existing buckets cannot be converted.** This is the big one — see below.

### The migration trap: you cannot convert an existing bucket

AWS's own migration guide is blunt:

> "Because existing buckets can't be renamed, migration requires creating a new
> bucket and transitioning workloads to it."
> — [Migrate to Amazon S3 account regional
> namespaces](https://aws.amazon.com/blogs/storage/migrate-to-amazon-s3-account-regional-namespaces/)

And its recommended migration is, recursively, *this note*: create the new
bucket, set up live replication old → new, run a **Batch Replication job** for
the existing objects, update every reference, watch the old bucket until it sees
zero requests, then retire it. Plus a warning that maps exactly onto the naming
rules above: deleted global-namespace bucket names can be claimed by other
accounts, so the safest end state is to **keep the old bucket forever, emptied**,
rather than delete it.

**The decision this forces on Helios.** The live primary buckets are already
global-namespace buckets. You have two coherent options:

| | **Option A: standby buckets in the account regional namespace, primaries left alone** | **Option B: migrate everything, primaries and standbys, to the account regional namespace** |
|---|---|---|
| Work | Zero extra — you are creating the standby buckets anyway, so create them `-an` | A second full copy-and-cutover per bucket, on the *primary*, on top of the DR work |
| Naming | **Asymmetric.** Primary is `myapp-prod-data`, standby is `myapp-prod-data-111122223333-eu-west-2-an`. The two names are not related by a formula. | **Symmetric and derivable.** Both names are `format("%s-%s-%s-an", prefix, account, region)`. |
| Squatting risk | Primary name still squattable if ever deleted | Eliminated |
| Failback | Reverse replication targets the old global-namespace primary. Fine. | Clean |
| Risk | Low | A cutover on a live production bucket, for a benefit that is cosmetic *today* |

**Recommendation: Option A now, Option B opportunistically.** Create every *new*
bucket — which includes every standby bucket — in the account regional namespace,
and enforce it with the SCP so nobody creates a global-namespace bucket again.
Do **not** bundle a primary-bucket rename into the multi-region programme; it is
a second cutover with its own Batch Replication job and its own consumer-update
sweep, and it competes for the same risk budget as the DR work that actually
moves the RPO/RTO needle. Migrate primaries to `-an` later, per bucket, when
something else already requires touching them.

The asymmetry in Option A is genuinely annoying and worth naming: **your module
must accept the primary bucket name as an input rather than deriving it.** That
is one variable, and it is honest about reality.

### The naming convention to adopt

For new buckets (all standbys, and any greenfield primary):

```
{product}-{env}-{purpose}-{accountid}-{region}-an
helios-prod-uploads-111122223333-eu-west-2-an
```

Rules for the prefix portion, which must fit in **34 characters**:

- `{product}` and `{env}` come from the cookiecutter context, so they are already
  consistent across the monorepo.
- Never encode the region in the prefix — the suffix already has it, and encoding
  it twice guarantees they will disagree one day.
- Never encode "primary"/"standby"/"dr" in the name. After a failback-less
  failover the standby *is* the primary and the name becomes a lie. This is the
  same mistake as naming a database replica `readonly-db`.
- Never encode the account id in the prefix — same reason, the suffix has it.

### Parameterisation: how consumers find the right bucket

This is the part that determines whether failover takes 30 seconds or three
hours. The rule is short: **no application, pipeline, policy or query may contain
a literal bucket name.** Where the name comes from instead:

| Consumer | Where the bucket name comes from | Failover behaviour |
|---|---|---|
| Application code (EKS pods, Lambda) | Env var populated from **SSM Parameter Store** at pod/function start, e.g. `/helios/prod/s3/uploads/bucket` | Restart picks up the new value. Pre-create the parameter in *both* regions ([[aws-ssm-parameter-store]]). |
| Terraform in other stacks | `terraform_remote_state` output or an SSM data source — never a literal | See [[state-management]] |
| IAM policies | Built from the module's output ARN, both buckets granted | No change needed at failover if both are already granted |
| CI/CD pipelines | Pipeline variable from the same SSM parameter | Redeploy or re-read |
| Athena / Glue `LOCATION` | **The hard one.** `LOCATION` is baked into table DDL. | Requires re-registering tables, or a second Glue database pointed at the standby. Budget for this explicitly. |
| Presigned URL generators | Same SSM parameter as the app | Old presigned URLs remain pointed at the dead region and will fail — they are short-lived, accept it |
| Mobile / browser clients | **Must go via CloudFront, never direct to a bucket host.** | [[aws-cloudfront]] origin failover handles it; a baked-in bucket hostname in a shipped mobile app cannot be fixed at 3am |
| Partner integrations | Documented endpoint, ideally a CloudFront or API Gateway URL | If a partner has your bucket name, your RTO now includes a phone call |

**The parameter to standardise on.** One SSM parameter per logical bucket per
region, holding the *bucket name for the region the reader is running in*:

```
/helios/{env}/s3/{purpose}/bucket-name
```

Written by Terraform in both regions with the region-appropriate value. An
application in `eu-west-2` reads the `eu-west-2` copy and gets the standby bucket
name; the same code in `eu-west-1` gets the primary. **The application never
knows which region is primary and never needs a failover switch.** This is the
single highest-leverage pattern in this note and it costs one extra SSM parameter
per bucket.

The corollary: while the primary is healthy, compute in the standby region (if
any) reads and writes the *standby* bucket, which has one-way replication
incoming. Anything that writes there will be overwritten or will diverge. Warm
standby compute must be read-only, or scaled to zero, until promotion. See
[[failover-orchestration]].

### Finding the hardcoded names you already have

Before any of this is worth anything, find the existing literals. A starting
sweep, in rough order of yield:

```bash
# 1. Source and IaC — the easy 80%
rg -n --hidden -g '!.git' '\bs3://|\.s3\.[a-z0-9-]+\.amazonaws\.com|\.s3\.amazonaws\.com'
rg -n 'bucket\s*=\s*"'   # Terraform literals that should be var/module outputs

# 2. Deployed IAM — the policies nobody greps
aws iam list-policies --scope Local --query 'Policies[].Arn' --output text \
  | xargs -n1 -I{} aws iam get-policy-version --policy-arn {} \
      --version-id "$(aws iam get-policy --policy-arn {} --query 'Policy.DefaultVersionId' --output text)"

# 3. Glue / Athena — the ones that will still be wrong a week after failover
aws glue get-tables --database-name "$DB" \
  --query 'TableList[].[Name,StorageDescriptor.Location]' --output text
```

CloudTrail `GetObject`/`PutObject` data events on the primary bucket, aggregated
by `userIdentity`, is the authoritative list of who actually touches it — and it
will be longer than the list anybody can give you from memory. Run that query
*before* you plan the cutover, not after. See [[lessons-and-antipatterns]].

---

## Multi-Region Access Points: worth it here?

### What they solve

A Multi-Region Access Point (MRAP) is a single global hostname —
`{alias}.accesspoint.s3-global.amazonaws.com` — in front of N buckets in N
regions. Requests are routed over the AWS global network to the lowest-latency
healthy bucket, and you can flip a **failover control** to change which regions
are "active".

That is genuinely the shape of the problem: *"one name, two buckets, switch at
failover"* is exactly the naming problem the previous section spent 200 lines on.
So MRAP deserves a serious look rather than a reflexive no. It gets a no anyway,
for reasons that are specific and checkable.

### Failover behaviour

From AWS's launch announcement of [failover controls for S3 Multi-Region Access
Points](https://aws.amazon.com/about-aws/whats-new/2022/11/amazon-s3-multi-region-access-points-failover-active-passive-configurations-failovers)
(November 2022):

> "you can typically shift S3 data access request traffic from an active AWS
> Region to a passive AWS Region within 2 minutes when initiating failover"

And on the routing model, from [Amazon S3 Multi-Region Access Points routing
states](https://docs.aws.amazon.com/AmazonS3/latest/userguide/FailoverConfiguration.html):
in an active-passive configuration the active regions receive traffic and the
passive ones do not. Two minutes sits comfortably inside a 15-minute RTO. On the
face of it, MRAP is the answer.

### The restrictions that kill it

All of the following are from [Multi-Region Access Point restrictions and
limitations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/MultiRegionAccessPointRestrictions.html).

| Restriction | Verbatim | Why it matters here |
|---|---|---|
| Control plane is one region | *"All control plane requests to create or maintain Multi-Region Access Points must be routed to the `US West (Oregon)` Region."* | You are **adding** a hard dependency on `us-west-2` to the EU and CA deployments. For a data-residency-sensitive Canadian deployment that is a conversation with legal. See [[data-residency]]. |
| Failover control plane is five regions | *"requests must be routed to one of these five supported Regions: `US East (N. Virginia)`, `US West (Oregon)`, `Asia Pacific (Sydney)`, `Asia Pacific (Tokyo)`, `Europe (Ireland)`"* | **No Canadian region.** The CA pair cannot initiate an MRAP failover from inside Canada at all. And for the EU pair, the only in-region failover control plane is `eu-west-1` — *the region you are failing away from*. That is a circular dependency at exactly the wrong moment. |
| Batch Operations unsupported | *"The S3 Batch Operations feature isn't supported."* | The migration backfill cannot go through the MRAP. Not fatal — you target the buckets directly — but it means the MRAP is not a complete abstraction and you still need both bucket names in your runbooks. |
| SigV4A required | *"Support for Signature Version 4 (SigV4A) — This version of SigV4 allows requests to be signed for multiple AWS Regions."* Plus: *"Certain AWS SDKs aren't supported."* | See below. |
| No gateway VPC endpoints | *"You can't access data through a Multi-Region Access Point by using gateway endpoints. However, you can access data through a Multi-Region Access Point by using interface endpoints."* | Gateway endpoints for S3 are **free**. Interface endpoints are not. Every VPC that currently reaches S3 over a free gateway endpoint would need a billed interface endpoint. |
| No IPv6 | *"be aware that IPv6 isn't supported"* | — |
| CloudFront needs custom origin | *"To use Multi-Region Access Points with Amazon CloudFront, you must configure the Multi-Region Access Point as a `Custom Origin` distribution type."* | This is the one that directly conflicts with the [Special-case buckets](#special-case-buckets) recommendation below: a `Custom Origin` is not an S3 origin, so the clean **Origin Access Control** path does not apply. See [[aws-cloudfront]]. |
| Buckets frozen after creation | *"After you create a Multi-Region Access Point, you can't add, modify, or remove buckets from the Multi-Region Access Point configuration. To change the buckets, you must delete the entire Multi-Region Access Point and create a new one."* | In Terraform this means any change to the bucket set is a destroy/recreate of the MRAP — and the alias changes, so every consumer breaks. |
| Buckets pinned by the MRAP | *"Underlying buckets (in the same account) that are used in a Multi-Region Access Point can be deleted only after a Multi-Region Access Point is deleted."* | Ordering constraint on teardown. |
| No anonymous requests | *"Multi-Region Access Points don't support anonymous requests."* | Rules out public-read buckets entirely. |
| Quotas | *"a maximum of 100 Multi-Region Access Points per account"*; *"a limit of 17 Regions for a single Multi-Region Access Point"* | 100/account is a real ceiling if you were tempted to put one in front of every bucket in a large estate. |

**`ca-west-1` parity check:** Calgary *is* supported as a **bucket** region — it
appears in the opt-in region list, and MRAP data-transfer usage types
(`CAN2-MRAP-In-Bytes` / `CAN2-MRAP-Out-Bytes`, $0.0033/GB) exist in the ca-west-1
price list. So the CA pair could technically have an MRAP. But the **failover
control plane has no Canadian region**, which means the one operation you would
actually need at 3am must be issued from the US, Europe, Japan or Australia.
Unlike the Cognito/OpenSearch/Grafana/Backup-Audit-Manager gaps catalogued in
[[region-pair-selection]], this one is not "Calgary is young" — it is a design
limitation of MRAP that applies to every Canadian region.

### SigV4A is a bigger ask than it looks

SigV4A is a different signing algorithm — an ECDSA-based extension of SigV4 that
produces a signature valid across regions. Modern AWS SDKs do it transparently.
The problems are at the edges:

- **Non-SDK clients.** Anything that hand-rolls SigV4 — an in-house signing
  helper, a third-party tool, a partner's integration, older `boto3`/`botocore`,
  certain CLI wrappers — will not sign correctly for an MRAP. AWS maintains an
  explicit [SDK compatibility
  list](https://docs.aws.amazon.com/sdkref/latest/guide/feature-s3-mrap.html)
  precisely because coverage is not universal.
- **Global STS tokens.** *"If you request temporary credentials from the global
  AWS STS endpoint (`sts.amazonaws.com`), then you must first set the Region
  compatibility of session tokens for the global endpoint to be valid in all AWS
  Regions."* An estate that still uses the global STS endpoint anywhere has a
  latent, silent failure here.

In a mature estate you do not know, today, which of your consumers sign requests
by hand. Finding out is a discovery project.

### Cost

From the AWS Price List API (`AmazonS3`, eu-west-1 and ca-west-1):

| Line item | Usage type | Price |
|---|---|---|
| MRAP data routing in | `EU-MRAP-In-Bytes`, `CAN2-MRAP-In-Bytes` | **$0.0033/GB** |
| MRAP data routing out | `EU-MRAP-Out-Bytes`, `CAN2-MRAP-Out-Bytes` | **$0.0033/GB** |

This is *on top of* normal request, storage and data-transfer charges. For a
request routed through an MRAP you pay both legs, so budget **$0.0066/GB of
traffic through the access point**. There is no per-hour or per-MRAP charge.

At 10 TB/month of application traffic through the access point that is
~$68/month per MRAP, per environment. Not enormous, but it is a permanent tax on
every byte, paid in exchange for a failover mechanism you were going to have to
build anyway because MRAP does not cover the consumers that don't use it.

### Recommendation: **no**

Do not use Multi-Region Access Points for Helios. The reasoning, in order of
weight:

1. **MRAP does not eliminate the work; it adds to it.** Even with an MRAP you
   still need the SSM-parameter pattern for Batch Operations, for Athena/Glue,
   for CloudFront origins and for every non-SDK consumer. You end up maintaining
   *two* indirection mechanisms instead of one.
2. **It replaces a config lookup with a control-plane dependency in another
   region.** The whole point of the pre-provisioning discipline in this vault is
   that failover touches as few control planes as possible. MRAP adds `us-west-2`
   for management and constrains failover to five regions that do not include
   Canada.
3. **The 2-minute failover is solving a problem we don't have.** S3's
   contribution to RTO is already near zero once the standby bucket is populated
   and the bucket name comes from config. We are not latency-routing; we are
   active/passive. MRAP's headline feature — proximity-based routing for an
   active/active application — is a feature for a posture the brief explicitly
   rules out.
4. **The CloudFront custom-origin requirement conflicts with OAC**, which is the
   recommended pattern for the origin buckets below.

**When would the answer change?** If the posture moved to active/active with
users served from their nearest region, MRAP plus two-way replication becomes
genuinely attractive and this recommendation should be revisited. It is also the
right answer for a *global* namespace problem — one dataset served worldwide —
which is not what three independent regional deployments are. Record it as a
rejected option with a stated trigger rather than a closed door.

---

## Monitoring replication lag

Replication is silent when it breaks. `PutBucketReplication` returns `200` with a
bad KMS key; objects fail one by one; the bucket keeps serving traffic perfectly;
and you discover it during a failover. **Replication monitoring is not optional
for a DR mechanism you only exercise once a year.** Cross-reference
[[observability-multi-region]] for how these alarms fit the wider picture.

### The four metrics, verified

Namespace **`AWS/S3`**. Dimensions: **`SourceBucket`**, **`DestinationBucket`**,
**`RuleId`** — note that means metrics are *per replication rule*, so your rule
IDs must be stable and meaningful, not Terraform-generated noise.

| Metric | Units | Valid statistics | Published in which Region? | Still published if replication isn't happening? |
|---|---|---|---|---|
| `ReplicationLatency` | Seconds | `Max` | **Destination bucket's Region** | Yes |
| `BytesPendingReplication` | Bytes | `Max` | **Destination bucket's Region** | Yes |
| `OperationsPendingReplication` | Count | `Max` | **Destination bucket's Region** | Yes |
| `OperationsFailedReplication` | Count | `Sum` (total failures), `Average` (failure rate), `SampleCount` (total replication operations) | **Source bucket's Region** | **No** |

Every cell above is from [Using S3 Replication
metrics](https://docs.aws.amazon.com/AmazonS3/latest/userguide/repl-metrics.html)
and [Metrics and
dimensions](https://docs.aws.amazon.com/AmazonS3/latest/userguide/metrics-dimensions.html).

### The gotchas hiding in that table

**1. The metrics are split across two regions.** Three of the four are published
in the *destination* region; `OperationsFailedReplication` is published in the
*source* region. A single-region CloudWatch dashboard cannot show you all four.
For the EU pair that means `ReplicationLatency` lives in `eu-west-2` and
`OperationsFailedReplication` lives in `eu-west-1`. Either build a cross-region
CloudWatch dashboard (CloudWatch dashboards can render metrics from multiple
regions in one widget) or centralise into whatever the estate uses — see
[[observability-multi-region]].

This also means **your standby-region alarms are load-bearing**: the lag metric
you care most about is only observable from the region that is idle. If nobody
has console access or alarm routing configured in `eu-west-2`, replication lag is
unmonitored.

**2. `OperationsFailedReplication` is the only one that covers Batch
Replication.** From the docs: the three pending/latency metrics apply *"only to
new objects that are replicated with S3 CRR or S3 SRR"*, whereas Operations
Failed Replication *"applies both to new objects that are replicated with S3 CRR
or S3 SRR and also to existing objects that are replicated with S3 Batch
Replication"*. So during the migration backfill, `ReplicationLatency` and
`BytesPendingReplication` tell you **nothing** about the backfill's progress. The
Batch Operations job status and the completion report are the only progress
signal. Do not build a dashboard that implies otherwise.

And the sharp edge on top of it: *"If an S3 Batch Replication job fails to run at
all, metrics aren't sent to Amazon CloudWatch. For example, your job won't run if
you don't have the necessary permissions to run an S3 Batch Replication job, or
if the tags or prefix in your replication configuration don't match."* **A
totally broken backfill is indistinguishable from no backfill, in metrics.** Alarm
on the Batch Operations job reaching a terminal state instead.

**3. Missing data is the normal case, and it will flap your alarms.** AWS says it
explicitly:

> "You can enable alarms for your replication metrics in Amazon CloudWatch. When
> you set up alarms for your replication metrics, set the **Missing data
> treatment** field to **Treat missing data as ignore (maintain the alarm
> state)**."
> — [Metrics and dimensions](https://docs.aws.amazon.com/AmazonS3/latest/userguide/metrics-dimensions.html)

`OperationsFailedReplication` is *not* published when replication isn't failing —
so a "failures > 0" alarm sits in `INSUFFICIENT_DATA` almost all the time. In
Terraform that is `treat_missing_data = "ignore"`. Getting this wrong produces
either constant pages or an alarm that never fires, and teams usually fix the
noise by deleting the alarm.

**4. Deleting the destination bucket silences three of the four metrics.** *"Is
this metric still published if the destination bucket is deleted?"* — **No** for
latency and the two pending metrics, **Yes** for failed operations. So "our
replication dashboard went flat" can mean "healthy" or "someone deleted the
standby bucket". Pair the lag alarms with a simple existence check.

### Enabling metrics — and doing it without paying for RTC

Replication metrics are **not on by default**. They come on automatically with
RTC, but you can enable them independently:

> "S3 Replication metrics are turned on automatically when you enable S3
> Replication Time Control (S3 RTC) … You can also enable S3 Replication metrics
> independently of S3 RTC while creating or editing a rule."

In the replication configuration that is `Destination.Metrics.Status = Enabled`,
and in Terraform it is a `metrics { status = "Enabled" }` block inside
`destination`. **Do this on every rule.** It is the cheap half of RTC —
the section [RTC vs the RTO](#rtc-vs-the-rto-they-are-not-the-same-fifteen-minutes)
already makes the argument; this is where you act on it.

**Cost:** *"S3 Replication metrics are billed at the same rate as Amazon
CloudWatch custom metrics."* At the standard CloudWatch custom-metric rate that
is four metrics per replication rule. For an estate with, say, 30 replicated
buckets and one rule each, that is 120 custom metrics — a few dollars a month.
Compare that with the RTC surcharge of **$0.015/GB** and the arithmetic is not
close. Verify the current custom-metric rate on the [CloudWatch pricing
page](https://aws.amazon.com/cloudwatch/pricing/) before quoting a total; this
note deliberately does not state a CloudWatch price it did not fetch.

One timing note: *"If you're using S3 Replication Time Control, Amazon CloudWatch
begins reporting replication metrics 15 minutes after you enable S3 RTC on the
respective replication rule."* Don't panic-debug an empty dashboard in the first
quarter of an hour.

### The alarms to actually create

```hcl
# In the DESTINATION region provider. ReplicationLatency is published there.
resource "aws_cloudwatch_metric_alarm" "replication_latency" {
  provider = aws.standby

  alarm_name          = "${var.name}-s3-replication-latency"
  namespace           = "AWS/S3"
  metric_name         = "ReplicationLatency"
  statistic           = "Maximum"
  period              = 300
  evaluation_periods  = 2
  comparison_operator = "GreaterThanThreshold"

  # RPO is 2h = 7200s. Alarm at 25% of budget so there is time to react.
  threshold          = 1800
  treat_missing_data = "ignore" # AWS's documented guidance for these metrics

  dimensions = {
    SourceBucket      = var.source_bucket
    DestinationBucket = var.destination_bucket
    RuleId            = var.replication_rule_id
  }

  alarm_actions = [var.alarm_topic_standby_arn]
  ok_actions    = [var.alarm_topic_standby_arn]
}

# In the SOURCE region provider. OperationsFailedReplication is published there.
resource "aws_cloudwatch_metric_alarm" "replication_failures" {
  provider = aws.primary

  alarm_name          = "${var.name}-s3-replication-failures"
  namespace           = "AWS/S3"
  metric_name         = "OperationsFailedReplication"
  statistic           = "Sum"
  period              = 300
  evaluation_periods  = 1
  comparison_operator = "GreaterThanThreshold"
  threshold           = 0
  treat_missing_data  = "ignore" # not published when there are no failures

  dimensions = {
    SourceBucket      = var.source_bucket
    DestinationBucket = var.destination_bucket
    RuleId            = var.replication_rule_id
  }

  alarm_actions = [var.alarm_topic_primary_arn]
}
```

Threshold rationale, because arbitrary thresholds get muted: the RPO is 7,200
seconds. Alarming at 1,800 seconds of latency gives 90 minutes of head-room to
investigate before the RPO is actually breached. Alarming at 7,200 would mean the
first page coincides with the objective already being missed. If you enable RTC
on a bucket, tighten that bucket's threshold to 900 seconds to match the RTC
threshold.

`OperationsFailedReplication > 0` should page during the migration and during
business hours, and ticket otherwise — a handful of permanent failures on
pathological objects is common and does not warrant a 3am page, but a *rising*
count does. Consider a second alarm on the `Average` statistic (failure rate) for
that.

### Event notifications: the per-object detail

Metrics tell you *that* replication failed; they never tell you *which object*.
For that, S3 Event Notifications publishes four replication event types to SNS,
SQS or Lambda:

| Event | Meaning |
|---|---|
| `s3:Replication:OperationFailedReplication` | This specific object failed. The one to route to a queue. |
| `s3:Replication:OperationMissedThreshold` | RTC only — object missed the 15-minute threshold |
| `s3:Replication:OperationReplicatedAfterThreshold` | RTC only — the late object finally arrived |
| `s3:Replication:OperationNotTracked` | The object is outside RTC tracking |

AWS is explicit that this is the intended pairing: *"To identify the specific
objects that have failed replication and their failure reasons, subscribe to the
`OperationFailedReplication` event in Amazon S3 Event Notifications."*

**Recommendation:** route `OperationFailedReplication` to an SQS queue per
environment. The queue depth *is* your remediation backlog, and its contents are
the input to the next `["FAILED"]` Batch Replication job. Note the two RTC-only
events are dead weight unless you bought RTC — which is another small argument
against buying it.

### S3 Storage Lens as the estate-wide view

The per-rule alarms are the operational layer. Storage Lens is the "have we
missed a bucket entirely" layer, and that is a different and equally important
question in a mature estate.

- **Free tier:** *"you can see metrics such as the total number of bytes that are
  replicated from the source bucket or the count of replicated objects from the
  source bucket."* Free, account-wide, daily.
- **Advanced metrics:** *"you can see how many replication rules you have of
  various types, including the count of replication rules with a replication
  destination that's not valid."* That last one is the direct detector for the
  invalid-KMS-key and deleted-destination failure modes described above, at the
  estate level rather than per bucket.

Advanced-metrics pricing from the Price List API (eu-west-1
`EU-StorageLens-ObjCount`, and the same tiers in ca-central-1):

| Tier | Price |
|---|---|
| First 25 billion objects/month | **$0.20 per million objects/month** |
| 25–100 billion objects/month | **$0.16 per million objects/month** |
| Over 100 billion objects/month | **$0.12 per million objects/month** |

Free-tier Storage Lens is `*-StorageLensFreeTier-ObjCount` at **$0.00**.

**Recommendation:** free-tier Storage Lens on by default account-wide (it costs
nothing and gives you the "which buckets have no replication at all" answer);
advanced metrics on for the duration of the multi-region programme, where
"replication rules with a destination that's not valid" is worth real money in
avoided incidents, then reassess. At $0.20/million objects, a 100-million-object
estate is $20/month — cheap for the migration window, worth re-justifying as a
permanent line item.

### What not to monitor

**Object-count parity between the buckets.** Covered in [Delete markers and the
replica-deletion trap](#delete-markers-and-the-replica-deletion-trap) and repeated
here because it is the check everyone reaches for first: with V2 rules, lifecycle
differences and non-replicated delete markers, the two buckets are *expected* to
have different object counts. A parity check will be permanently red, will be
muted within a week, and will have taught the team that replication alarms are
noise. Don't build it.

---

## Special-case buckets

The generic answer — "make a standby bucket, turn on CRR, backfill with Batch
Replication" — is wrong for roughly a third of the buckets in a mature estate.
Classify every bucket before you start. The classification is the deliverable;
the Terraform follows from it.

| Class | Replicate? | Why |
|---|---|---|
| Application data (uploads, documents, exports) | **Yes, CRR + backfill** | The default case this note is written for |
| CloudTrail log target | **No — point the trail at one bucket** | See below |
| S3 server access log target | **Separate bucket per region, no replication** | Same-region constraint, see below |
| Terraform state | **Careful yes, one-way, never two-way** | See below — this is the dangerous one |
| CloudFront origin | **Yes, plus an origin group** | See below |
| Lambda artifacts | **Yes, or rebuild — either works** | Same-region constraint on `CreateFunction` |
| Build caches, thumbnails, derived data | **No — reconstruct** | Cheaper to regenerate than to duplicate |
| Athena query results | **No** | Ephemeral by definition; create an empty standby bucket and point the workgroup at it |

### CloudTrail buckets — do not replicate, just don't move them

A trail's destination bucket does not have to be in the trail's region:

> "As long as CloudTrail has permissions to write to an S3 bucket, the bucket for
> a multi-Region trail does not have to be in the trail's home Region."
> — [Receiving CloudTrail log files from multiple
> Regions](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/receive-cloudtrail-log-files-from-multiple-regions.html)

So the multi-region problem for CloudTrail is not a replication problem; it is an
*audit* problem. Two coherent postures:

| | **A: one central log-archive bucket, third region or security account** | **B: per-region CloudTrail bucket, replicated to the standby** |
|---|---|---|
| Shape | Every trail in every region writes to one bucket in a dedicated log-archive account | Trail writes locally, CRR mirrors to the standby |
| Failover | Nothing to do. CloudTrail keeps writing. | Nothing to do either — but you now have two copies with divergent lifecycle |
| Availability | If the log-archive region is impaired, CloudTrail buffers and retries; you still have the CloudTrail *Event history* (90 days) in each region | Logs from the failed region stop; logs from the standby start |
| Cost | One copy | Two copies of everything, forever |
| Residency | **The blocker for the CA pair.** A Canadian trail writing to an Irish bucket is a data-export event. See [[data-residency]]. | Stays in Canada |

**Recommendation: Option A per *regulatory jurisdiction*, not globally.** One
central log-archive bucket for the EU pair, one for the US pair, and one *inside
Canada* for the CA pair. That gives you one copy per jurisdiction instead of six
copies globally, and it keeps each jurisdiction's audit trail inside its own
borders. Do not replicate CloudTrail buckets.

**If you do keep a per-region CloudTrail bucket**, use the **Archive** posture
from the delete-markers section — `delete_marker_replication` *disabled* — plus
**Object Lock** in `COMPLIANCE` mode. For an audit-log bucket, "deletes must not
propagate" is the whole point, and the divergence that makes Archive posture
wrong for application data makes it right here.

**Object Lock + replication, two documented catches:**

1. *"if the source bucket has Object Lock enabled, the destination buckets must
   also have Object Lock enabled"* and the replication role needs
   **`s3:GetObjectRetention`** and **`s3:GetObjectLegalHold`** — *"If the role has
   an `s3:Get*` permission statement, that statement satisfies the requirement."*
2. **After you enable Object Lock on a bucket, you can't disable Object Lock or
   suspend versioning for that bucket.** A one-way door, deliberately.

**The "contact AWS Support" question, resolved as far as public sources allow.**
AWS re:Post knowledge-center material states that enabling Object Lock on an
existing replication pair requires you to contact AWS Support for an *Object Lock
token*. The current [Object Lock
considerations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock-managing.html)
user-guide page carries no such statement — it says plainly *"To set up
replication on a bucket with Object Lock enabled, you can use the S3 console, AWS
CLI, Amazon S3 REST API, or AWS SDKs."* But the Terraform provider still
documents the mechanism explicitly, on
`aws_s3_bucket_replication_configuration`:

> "`token` - (Optional) Token to allow replication to be enabled on an Object
> Lock-enabled bucket. You must contact AWS support for the bucket's 'Object Lock
> token'."

So the token path demonstrably still exists in the API. The honest reading: the
straightforward order (**enable Object Lock at bucket creation, then configure
replication**) needs no token, which is why the user guide no longer mentions it;
the retrofit order (**bucket already replicating, now add Object Lock**) is the
one that needs the support ticket. **Create audit buckets with Object Lock
enabled from the start** and the question never arises. If you must retrofit,
open the support case before you plan the change window — it is not same-day.

### Server access log buckets — you have no choice

> "The target bucket must be in the same AWS Region and AWS account as the source
> bucket."
> — [Enabling Amazon S3 server access
> logging](https://docs.aws.amazon.com/AmazonS3/latest/userguide/enable-server-access-logging.html)

Attempting otherwise returns `Cross S3 location logging not allowed`. So:

- **The standby bucket needs its own, same-region, access-log target bucket.**
  Terraform must create `helios-prod-uploads-logs-...-eu-west-2-an` alongside the
  standby data bucket, or `PutBucketLogging` on the standby fails and your
  standby has no access logging — which you discover during the post-failover
  audit.
- **The access-log buckets themselves do not need replicating.** Each region logs
  its own bucket's access locally. Treat them as regional artefacts.
- **Object Lock is incompatible with being a log target:** *"S3 buckets with
  Object Lock can't be used as destination buckets for server access logs."* So
  the Object Lock recommendation above applies to CloudTrail buckets, **not** to
  server-access-log buckets.

This is a small thing that fails an entire `terraform apply` late in a migration.
Put the log-target bucket in the same module as the data bucket so they are
always created as a pair.

### Terraform state buckets — the circular dependency

This is the one to think hardest about. See [[state-management]] for the wider
repo-structure discussion; here is the S3-specific part.

**The problem.** Your Terraform state for the `eu-west-1` deployment lives in an
S3 bucket in `eu-west-1`. `eu-west-1` has a regional S3 impairment. You now want
to run `terraform apply` to promote the standby — and Terraform cannot read its
own state. **The tool you would use to fix the outage is inside the outage.**

Worse, the state bucket is not the only dependency: if you use S3 native state
locking (`use_lockfile = true`, GA in Terraform 1.11 — the DynamoDB path is now
deprecated per the [S3 backend
docs](https://developer.hashicorp.com/terraform/language/backend/s3)), the lock
object is also in that bucket, in that region. And `PutBucketVersioning`,
`PutBucketPolicy` and friends have a `us-east-1` control-plane dependency, so you
cannot even reconfigure your way out during a `us-east-1` event.

**The three options.**

| | **A: replicate state one-way to the standby region** | **B: state bucket in a third region** | **C: don't need Terraform at failover** |
|---|---|---|---|
| Shape | CRR from `eu-west-1` state bucket to an `eu-west-2` state bucket; at failover, reconfigure the backend to the replica | State for the EU pair lives in, say, `eu-central-1` — a region unrelated to either side of the pair | Everything needed to promote is pre-provisioned; failover is DNS + config, not `terraform apply` |
| Failover step | `terraform init -reconfigure -backend-config=...` pointing at the replica, then apply | Nothing. State was never in the failed region. | Nothing |
| Risk | **Split-brain.** Two state files for one set of resources. If anyone applies against the original after you have applied against the replica, you have divergent state and no merge story. | Adds a third region's availability to the dependency graph of both regions | None from S3 |
| RPO on state | Replication lag — state written seconds before the outage may not be there | Zero | N/A |
| Blast radius | Whole-estate if mishandled | One more region to keep an eye on | — |

**Recommendation: C as the primary design, B as the safety net, and A only as a
read-only disaster copy.**

- **C is the real answer, and it is the whole thesis of this vault.** The
  prerequisites-first strategy exists precisely so that at 3am nobody runs
  `terraform apply`. If promoting the standby requires Terraform, the 15-minute
  RTO is already lost to `terraform init` and plan review, never mind a state
  bucket. Any note in this vault whose failover procedure contains
  `terraform apply` should be treated as failing the RTO. See
  [[failover-orchestration]].
- **B costs nothing and removes the circularity.** Put each pair's state in a
  region that is neither the primary nor the standby, chosen for residency
  compatibility (`eu-central-1` for the EU pair, `us-east-2` for the US pair,
  and — importantly — **`ca-west-1` or `ca-central-1` for the CA pair, which means
  the CA pair genuinely cannot have a third region inside Canada**; there are only
  two Canadian regions, so the CA pair's state must live in whichever Canadian
  region is *not* the one currently primary, or accept the circularity). Flag
  this as an open question for the CA pair rather than pretending it resolves.
- **A is fine as a read-only copy** — replicate state to the standby with
  versioning and *no* backend ever configured against it, purely so a human can
  read the last-known resource IDs during an incident. Make the replica bucket's
  policy deny `s3:PutObject` from everything except the replication role, so
  nobody can accidentally apply against it. **Never configure a live backend
  against a replica without a deliberate, documented, one-way cutover.**

Whatever you choose: **versioning on, `force_destroy = false`, MFA delete or a
deny-delete bucket policy, and a `prevent_destroy` lifecycle block.** The state
bucket is the single most dangerous bucket in the estate.

### CloudFront origin buckets

Covered properly in [[aws-cloudfront]]; the S3-side facts:

**The good news.** CloudFront origin groups give you automatic, no-human-required
failover between two S3 origins, and it composes cleanly with OAC:

- Put an **Origin Access Control** on *each* bucket, with each bucket policy
  granting `s3:GetObject` to `cloudfront.amazonaws.com` conditioned on the
  distribution ARN. Both buckets stay private.
- Create an **origin group** with the primary bucket's origin first and the
  standby bucket's origin second.
- Failover criteria: from [Optimize high availability with CloudFront origin
  failover](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/high_availability_origin_failover.html)
  you may choose any combination of **400, 403, 404, 416, 429, 500, 502, 503,
  504**. For a private S3 origin behind OAC, include **403 and 404** alongside the
  5xx codes — an S3 origin that cannot serve an object returns 403/404, not 503.

**The limits, quoted, because they determine whether this is a real solution:**

> "CloudFront fails over to the secondary origin only when the HTTP method of the
> viewer request is `GET`, `HEAD`, or `OPTIONS`. CloudFront does not fail over
> when the viewer sends a different HTTP method (for example `POST`, `PUT`, and
> so on)."

> "CloudFront will not failover if `OPTIONS` are not set as a Cached HTTP methods
> in your cache behavior."

> "By default, CloudFront tries to connect to the primary origin in an origin
> group for as long as 30 seconds (3 connection attempts of 10 seconds each)
> before failing over to the secondary origin."

> "CloudFront routes all incoming requests to the primary origin, even when a
> previous request failed over to the secondary origin. CloudFront only sends
> requests to the secondary origin after a request to the primary origin fails."

Three consequences:

1. **Reads fail over automatically; writes do not.** Origin groups are a read
   availability mechanism. Any upload path through CloudFront still needs the
   config-driven bucket switch.
2. **30 seconds of latency per uncached request during an outage**, because
   CloudFront retries the primary every time rather than remembering. Tune
   `connection_attempts = 1` and `connection_timeout = 2` on the primary origin to
   cut this to ~2 seconds. That is 1 of the 1–3 attempts and 2 of the 1–10 seconds
   the docs permit.
3. **OPTIONS must be in the cached methods list** or failover silently doesn't
   happen for preflighted requests — a CORS-shaped outage that looks like an
   application bug.

**Recommendation:** origin group + OAC on both buckets, tuned timeouts, and 403 +
404 + 500 + 502 + 503 + 504 as failover criteria. This is the one place in the
whole S3 story where failover is genuinely automatic and sub-RTO with no human
involved, and it costs nothing extra. Do it.

### Lambda artifact buckets

> "An Amazon S3 bucket must be in the same Amazon Web Services Region as your
> Lambda function."
> — [`CreateFunction`](https://docs.aws.amazon.com/lambda/latest/APIReference/API_CreateFunction.html)

So a Lambda in `eu-west-2` cannot be created from a zip in an `eu-west-1` bucket,
full stop. Two ways to satisfy it, and unusually both are fine:

| | **A: replicate the artifact bucket** | **B: publish to both buckets from CI** |
|---|---|---|
| Mechanism | CRR from the `eu-west-1` artifact bucket to `eu-west-2` | The pipeline uploads the same zip to both buckets before deploying |
| Failure mode | Artifact published minutes before the outage may not have replicated; the standby Lambda deploys an older version | Pipeline must succeed in both regions; a partial upload is visible immediately |
| Cost | Inter-region DT on every artifact | Data transfer out from CI, same order of magnitude |
| Determinism | Eventual | **Immediate and verifiable in the pipeline** |

**Recommendation: B — publish from CI to both regions.** Artifact buckets are the
textbook case where "just deploy a second copy" beats replication: the content is
reproducible, the publisher is a pipeline you control, and you want a hard
build-time failure rather than a soft replication lag. It also means the artifact
bucket needs **no Batch Replication backfill** — re-publish the current release to
the standby bucket and you are done.

Keep it lean: a lifecycle rule expiring artifacts after N releases, applied
identically in both regions, so the standby doesn't accumulate every zip ever
built. See [[aws-lambda]], and [[aws-ecr]] for the container-image equivalent,
which has its own native cross-region replication and is a different story.

### Buckets to deliberately *not* replicate

State it positively in the design doc, because "we forgot" and "we decided not
to" look identical in a post-incident review:

- **Derived/reconstructible data** — thumbnails, transcodes, search indexes,
  build caches. Create the standby bucket empty, and make the regeneration path
  part of the failover runbook (it is usually a backfill job you already have).
- **Athena/query-result buckets** — create empty, point the workgroup at the
  regional one via the same SSM parameter.
- **Scratch and temp buckets** with aggressive lifecycle expiry.

For each of these, the standby bucket still gets **created by Terraform** with
full configuration. Only the *data* is skipped. The distinction matters: an empty
pre-created bucket costs nothing and takes zero time at failover; a bucket that
must be created at failover time hits the `us-east-1` `CreateBucket` dependency
described at the top of this note.

---

## Replica storage class as a cost lever

The [Warm standby shape](#warm-standby-shape) section established that the
duplicated bytes are essentially the entire idle cost of multi-region S3. This is
the one lever that moves that number, and it is a big one.

You can set the replica's storage class independently of the source's. The
`Destination.StorageClass` field accepts **`DEEP_ARCHIVE`, `GLACIER`,
`GLACIER_IR`, `INTELLIGENT_TIERING`, `ONEZONE_IA`, `REDUCED_REDUNDANCY`,
`STANDARD`, `STANDARD_IA`** ([`Destination` API
reference](https://docs.aws.amazon.com/AmazonS3/latest/API/API_Destination.html)).
In Terraform it is `destination { storage_class = "..." }`.

**The key insight: the RPO/RTO asymmetry in the brief points straight at this.**
RPO 2h is loose — replication speed is not the constraint. RTO 15m is tight —
*read latency after promotion* is the constraint. So the question for each class
is only: **can the standby serve reads immediately on promotion?**

### The classes, priced and judged

eu-west-2 prices from the AWS Price List API, first-50-TB tier where tiered:

| Class | Storage $/GB-mo | vs Standard | Readable immediately after failover? | Verdict for a DR replica |
|---|---|---|---|---|
| `STANDARD` | **$0.024** | — | Yes | Correct default. No surprises. |
| `STANDARD_IA` | **$0.0131** | **−45%** | **Yes** | **The recommendation.** See below. |
| `ONEZONE_IA` | **$0.01048** | −56% | Yes, but single-AZ | **No.** A DR copy that itself has a single-AZ failure mode is not a DR copy. |
| `GLACIER_IR` | **$0.005** | **−79%** | **Yes** (millisecond retrieval) | Strong candidate for cold, rarely-read buckets. Watch the retrieval fee. |
| `GLACIER` (Flexible Retrieval) | **$0.00405** | −83% | **No — requires a restore** | **Fails RTO 15m.** Restore is minutes to hours. |
| `DEEP_ARCHIVE` | (cheaper still) | — | **No — hours** | **Fails RTO 15m** by an order of magnitude. |
| `INTELLIGENT_TIERING` | $0.024 FA tier | — | Depends on tier | **No.** A replica that is written once and never read will be tiered down automatically — possibly into an archive tier that fails your RTO — and you have no control over when. |
| `REDUCED_REDUNDANCY` | $0.0252 | +5% | Yes | Legacy, *more* expensive than Standard. Never. |

### The hidden costs that eat the saving

A −45% headline is not a −45% bill. Three charges move in the wrong direction:

**1. Minimum billable object size.** `STANDARD_IA`, `ONEZONE_IA` and
`GLACIER_IR` have a **128 KB minimum billable object size**. A bucket of 20 KB
objects billed at 128 KB each is being charged 6.4× its actual size — at
$0.0131/GB that is an *effective* $0.084/GB, three and a half times Standard.
**For small-object buckets, IA classes are more expensive than Standard.** Check
your mean object size before switching; anything with a mean under ~256 KB should
stay on Standard.

**2. Minimum storage duration.** 30 days for `STANDARD_IA` / `ONEZONE_IA`,
90 days for `GLACIER_IR` and `GLACIER`. Objects deleted or overwritten before
that are billed for the full minimum anyway. **A bucket with a high overwrite
rate pays the minimum-duration charge on every superseded version.** For a
write-heavy bucket this can wipe out the entire saving. The interaction with
versioning is nasty: every noncurrent version is a separate object with its own
minimum-duration clock.

**3. Replication PUT requests cost more.** From the eu-west-2 price list:

| Destination class | Tier-1 (PUT) request price | vs Standard |
|---|---|---|
| `STANDARD` (`EUW2-Requests-Tier1`) | $0.0053 per 1,000 | — |
| `STANDARD_IA` (`EUW2-Requests-SIA-Tier1`) | $0.01 per 1,000 | **1.9×** |
| `GLACIER_IR` (`EUW2-Requests-GIR-Tier1`) | $0.02 per 1,000 | **3.8×** |

Since replication does one PUT per object per destination, this scales with
object *count*, not size. Another reason small-object buckets are the wrong
candidates.

**4. Retrieval fees, paid only when you actually fail over.**

| Class | Retrieval fee (eu-west-2) |
|---|---|
| `STANDARD` | $0 |
| `STANDARD_IA` (`EUW2-Retrieval-SIA`) | **$0.01/GB** |
| `ONEZONE_IA` (`EUW2-Retrieval-ZIA`) | $0.01/GB |
| `GLACIER_IR` (`EUW2-Retrieval-GIR`) | **$0.03/GB** |

This one is worth thinking about carefully, because it is a **contingent** cost
that only lands during an incident. You are trading a certain monthly saving for
an uncertain one-off charge at the worst possible moment.

### Quantified: the 10 TB, 20-million-object bucket

Same bucket as the backfill worked example. Mean object size 512 KB, so the
128 KB minimum does not bite.

| | `STANDARD` | `STANDARD_IA` | `GLACIER_IR` |
|---|---|---|---|
| Standby storage, 10,240 GB | $245.76/mo | **$134.14/mo** | **$51.20/mo** |
| Monthly saving vs Standard | — | **$111.62 (−45%)** | **$194.56 (−79%)** |
| Annual saving | — | **$1,339** | **$2,335** |
| Backfill PUTs (20M, one-off) | $106.00 | $200.00 | $400.00 |
| Retrieval fee *if you fail over and read all 10 TB* | $0 | $102.40 | $307.20 |
| Break-even on the extra PUT cost | — | **< 1 month** | **< 2 months** |

**The retrieval fee is less than one month of the saving in both cases.** Even if
you failed over and re-read the entire dataset *every single month*, `STANDARD_IA`
would still be cheaper than `STANDARD`. That is the number that settles the
argument.

Scale it to a realistic estate: at 100 TB of replicated data, `STANDARD_IA` saves
roughly **$13,400/year** and `GLACIER_IR` roughly **$23,300/year**, per region
pair. Across three pairs that is real money for a one-line Terraform change.

### Recommendation

**Default the replica to `STANDARD_IA`, with three documented exceptions.**

1. **Mean object size under ~256 KB → stay on `STANDARD`.** The 128 KB minimum
   inverts the saving.
2. **High overwrite rate (objects superseded inside 30 days) → stay on
   `STANDARD`.** The minimum-duration charge eats it.
3. **Buckets you expect to read heavily immediately after promotion, where the
   retrieval fee lands at the worst moment and you would rather not explain it on
   an incident call → `STANDARD`.** This is a judgement call, and the numbers
   above say it is usually the wrong call, but it is a legitimate one.

**`GLACIER_IR` for genuinely cold replicas** — document archives, historical
exports, compliance data, anything with a low read rate — where −79% is worth the
$0.03/GB retrieval and the 90-day minimum duration.

**Never `GLACIER`, `DEEP_ARCHIVE` or `INTELLIGENT_TIERING` for a DR replica.**
The first two fail RTO 15m outright because the replica is not readable without a
restore; the third fails it *unpredictably*, which is worse, because a replica
that is read-once-a-year will be tiered into Archive Access exactly when you stop
watching.

**The failback consequence, which is easy to miss:** replicating *back* from a
`STANDARD_IA` standby to the primary after a failover means reading every object
out of IA, and you pay the retrieval fee on the whole dataset. Budget for it in
[Failback](#failback) — it is a real line item, not a rounding error.

### The one-line change

```hcl
destination {
  bucket        = aws_s3_bucket.standby.arn
  storage_class = var.replica_storage_class   # default "STANDARD_IA"
}
```

This does **not** force replacement, and it does **not** retroactively change the
class of objects already replicated. Existing replicas stay where they are. To
move them you either run a lifecycle transition on the destination bucket or
re-run Batch Replication — and a lifecycle transition on the destination is by
far the cheaper option. Setting the right class *before* the backfill saves you
a transition charge on every object; **decide the storage class before you run
the Batch Replication job, not after.**

Note the provider default, which is not "Standard": *"By default, Amazon S3 uses
the storage class of the source object to create the object replica."* If you
omit `storage_class` you inherit whatever the source object had, per object.

---

## Terraform implementation

### The shape: one module, two providers, one call site

The cookiecutter monorepo already has a per-environment stack. What this note
adds is a **`s3-replicated-bucket` module** that owns *both* buckets and the rule
between them, configured through provider aliases. Do not split it into two
modules called twice — the replication rule needs ARNs and KMS key ARNs from both
sides, and passing those between two module invocations turns an internal
dependency into a fragile external contract.

```hcl
# modules/s3-replicated-bucket/versions.tf
terraform {
  required_version = ">= 1.11"
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = ">= 5.70, < 7.0"
      configuration_aliases = [aws.primary, aws.standby]
    }
  }
}
```

`configuration_aliases` is the important line: it makes the module *declare* that
it needs two configured providers, so the call site must pass both explicitly.
That is what stops someone wiring the standby bucket into the primary region by
accident. See [[provider-aliases-vs-separate-stacks]] for when this pattern stops
scaling and you should split stacks instead — for S3 it does not, because the two
buckets genuinely have to be created together.

### Variable surface

```hcl
# modules/s3-replicated-bucket/variables.tf

variable "name_prefix" {
  description = <<-EOT
    Customer-chosen portion of the bucket name, WITHOUT the account regional
    suffix. Max 34 characters: the longest region code in this estate is
    ca-central-1, whose suffix "-111122223333-ca-central-1-an" is 29 chars of
    the 63-char bucket name limit.
  EOT
  type        = string
  validation {
    condition     = length(var.name_prefix) <= 34 && can(regex("^[a-z0-9][a-z0-9-]*[a-z0-9]$", var.name_prefix))
    error_message = "name_prefix must be <= 34 lowercase alphanumeric/hyphen chars, not starting or ending with a hyphen."
  }
}

variable "primary_region" { type = string }
variable "standby_region" { type = string }

variable "primary_bucket_name_override" {
  description = <<-EOT
    Set ONLY when migrating an existing global-namespace bucket. Existing buckets
    cannot be renamed or moved into the account regional namespace, so the
    primary name has to be supplied rather than derived. Leave null for new
    buckets. See the "Bucket names are globally unique" section.
  EOT
  type        = string
  default     = null
}

variable "replicate_data" {
  description = <<-EOT
    false = create the standby bucket fully configured but DO NOT replicate data.
    Correct for reconstructible buckets (caches, derived data, Athena results)
    and for Lambda artifacts published to both regions by CI.
  EOT
  type        = bool
  default     = true
}

variable "replica_storage_class" {
  description = "STANDARD | STANDARD_IA | GLACIER_IR. Never GLACIER/DEEP_ARCHIVE/INTELLIGENT_TIERING - they fail RTO 15m."
  type        = string
  default     = "STANDARD_IA"
  validation {
    condition     = contains(["STANDARD", "STANDARD_IA", "GLACIER_IR"], var.replica_storage_class)
    error_message = "GLACIER, DEEP_ARCHIVE and INTELLIGENT_TIERING replicas are not readable within the 15-minute RTO."
  }
}

variable "delete_marker_replication" {
  description = "Mirror posture (true) vs Archive posture (false). Audit-log buckets use false. See the delete-marker section."
  type        = bool
  default     = true
}

variable "enable_rtc" {
  description = "Replication Time Control. Costs $0.015/GB. Metrics are enabled regardless - RTC is not needed for RPO 2h."
  type        = bool
  default     = false
}

variable "lifecycle_rules" {
  description = "Applied IDENTICALLY to both buckets. Divergent lifecycle is how replicas silently grow."
  type        = any
  default     = []
}

variable "replication_latency_alarm_seconds" {
  description = "25% of the 2h RPO budget, leaving 90 minutes to react."
  type        = number
  default     = 1800
}
```

### Names and KMS keys

```hcl
# modules/s3-replicated-bucket/main.tf

data "aws_caller_identity" "current" {}

locals {
  account_id = data.aws_caller_identity.current.account_id

  # Account regional namespace: {prefix}-{accountId}-{region}-an
  primary_bucket_name = coalesce(
    var.primary_bucket_name_override,
    format("%s-%s-%s-an", var.name_prefix, local.account_id, var.primary_region),
  )
  standby_bucket_name = format("%s-%s-%s-an", var.name_prefix, local.account_id, var.standby_region)

  # New buckets go in the account regional namespace; a migrated primary can't.
  primary_namespace = var.primary_bucket_name_override == null ? "account-regional" : "global"
}

resource "aws_kms_key" "primary" {
  provider                = aws.primary
  description             = "S3 ${var.name_prefix} (${var.primary_region})"
  enable_key_rotation     = true
  deletion_window_in_days = 30
}

resource "aws_kms_key" "standby" {
  provider                = aws.standby
  description             = "S3 ${var.name_prefix} (${var.standby_region})"
  enable_key_rotation     = true
  deletion_window_in_days = 30
}
```

Two ordinary single-region keys, deliberately — see [KMS-encrypted objects across
regions](#kms-encrypted-objects-across-regions) and
[[kms-when-to-use-multi-region-keys]]. A multi-Region key buys nothing here
because S3 treats MRKs as single-Region keys.

### Buckets, both sides

```hcl
resource "aws_s3_bucket" "primary" {
  provider         = aws.primary
  bucket           = local.primary_bucket_name
  bucket_namespace = local.primary_namespace
  force_destroy    = false

  lifecycle { prevent_destroy = true }
}

resource "aws_s3_bucket" "standby" {
  provider         = aws.standby
  bucket           = local.standby_bucket_name
  bucket_namespace = "account-regional"
  force_destroy    = false

  lifecycle { prevent_destroy = true }
}

# Access-log target buckets MUST be in the same region as the bucket they log.
resource "aws_s3_bucket" "primary_logs" {
  provider         = aws.primary
  bucket           = format("%s-logs-%s-%s-an", var.name_prefix, local.account_id, var.primary_region)
  bucket_namespace = "account-regional"
}

resource "aws_s3_bucket" "standby_logs" {
  provider         = aws.standby
  bucket           = format("%s-logs-%s-%s-an", var.name_prefix, local.account_id, var.standby_region)
  bucket_namespace = "account-regional"
}

# Versioning. Hard prerequisite for replication, on BOTH sides.
resource "aws_s3_bucket_versioning" "primary" {
  provider = aws.primary
  bucket   = aws_s3_bucket.primary.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_versioning" "standby" {
  provider = aws.standby
  bucket   = aws_s3_bucket.standby.id
  versioning_configuration { status = "Enabled" }
}

# Encryption, with Bucket Keys on for the KMS cost reduction. NOTE: enabling
# bucket keys changes the KMS encryption context from the object ARN to the
# bucket ARN - the replication role policy below already accounts for this.
resource "aws_s3_bucket_server_side_encryption_configuration" "primary" {
  provider = aws.primary
  bucket   = aws_s3_bucket.primary.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.primary.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "standby" {
  provider = aws.standby
  bucket   = aws_s3_bucket.standby.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.standby.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "primary" {
  provider                = aws.primary
  bucket                  = aws_s3_bucket.primary.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_public_access_block" "standby" {
  provider                = aws.standby
  bucket                  = aws_s3_bucket.standby.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Lifecycle applied IDENTICALLY both sides. Divergence is how replicas grow
# without bound. Note noncurrent_version_expiration is mandatory once versioning
# is on - see the versioning section.
resource "aws_s3_bucket_lifecycle_configuration" "primary" {
  count    = length(var.lifecycle_rules) > 0 ? 1 : 0
  provider = aws.primary
  bucket   = aws_s3_bucket.primary.id
  dynamic "rule" {
    for_each = var.lifecycle_rules
    content {
      id     = rule.value.id
      status = "Enabled"
      filter { prefix = try(rule.value.prefix, "") }
      dynamic "expiration" {
        for_each = try([rule.value.expiration_days], [])
        content { days = expiration.value }
      }
      noncurrent_version_expiration { noncurrent_days = try(rule.value.noncurrent_days, 30) }
    }
  }
}
# ... identical resource for aws.standby, omitted for length.
```

### The replication role and its policy

```hcl
resource "aws_iam_role" "replication" {
  provider = aws.primary   # the role lives in the SOURCE account/region partition
  name     = "${var.name_prefix}-s3-replication"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "s3.amazonaws.com" }
      Action    = "sts:AssumeRole"
      Condition = {
        StringEquals = { "aws:SourceAccount" = local.account_id }
        ArnLike      = { "aws:SourceArn" = aws_s3_bucket.primary.arn }
      }
    }]
  })
}

data "aws_iam_policy_document" "replication" {
  statement {
    sid     = "ReadSource"
    actions = [
      "s3:GetReplicationConfiguration",
      "s3:ListBucket",
      "s3:GetObjectVersionForReplication",  # NOT GetObjectVersion - see KMS section
      "s3:GetObjectVersionAcl",
      "s3:GetObjectVersionTagging",
      "s3:GetObjectRetention",              # required if Object Lock is on
      "s3:GetObjectLegalHold",
    ]
    resources = [aws_s3_bucket.primary.arn, "${aws_s3_bucket.primary.arn}/*"]
  }

  statement {
    sid       = "WriteDestination"
    actions   = ["s3:ReplicateObject", "s3:ReplicateDelete", "s3:ReplicateTags", "s3:ObjectOwnerOverrideToBucketOwner"]
    resources = ["${aws_s3_bucket.standby.arn}/*"]
  }

  statement {
    sid       = "DecryptSourceKey"
    actions   = ["kms:Decrypt"]
    resources = [aws_kms_key.primary.arn]
    condition {
      test     = "StringLike"
      variable = "kms:ViaService"
      values   = ["s3.${var.primary_region}.amazonaws.com"]
    }
    condition {
      test     = "StringLike"
      variable = "kms:EncryptionContext:aws:s3:arn"
      # Bucket Keys are enabled, so the encryption context is the BUCKET arn,
      # not the object arn. With bucket keys off this must be "<bucket>/*".
      values = [aws_s3_bucket.primary.arn]
    }
  }

  statement {
    sid       = "EncryptDestinationKey"
    actions   = ["kms:Encrypt", "kms:GenerateDataKey"]
    resources = [aws_kms_key.standby.arn]
    condition {
      test     = "StringLike"
      variable = "kms:ViaService"
      # THE most-copy-pasted bug in S3 replication: this must be the STANDBY
      # region, not the primary. A wrong value here returns 200 on the config
      # and fails every object at runtime.
      values = ["s3.${var.standby_region}.amazonaws.com"]
    }
    condition {
      test     = "StringLike"
      variable = "kms:EncryptionContext:aws:s3:arn"
      values   = [aws_s3_bucket.standby.arn]
    }
  }
}
```

### The replication rule, with a guard rail

```hcl
resource "aws_s3_bucket_replication_configuration" "this" {
  count    = var.replicate_data ? 1 : 0
  provider = aws.primary

  # AWS's own example carries this depends_on. The implicit edge through
  # bucket.id exists but does not encode "versioning must be settled first".
  depends_on = [
    aws_s3_bucket_versioning.primary,
    aws_s3_bucket_versioning.standby,
  ]

  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.primary.id

  rule {
    # Stable, meaningful ID: it becomes the RuleId CloudWatch dimension.
    id       = "${var.name_prefix}-to-${var.standby_region}"
    status   = "Enabled"
    priority = 1

    # An EMPTY filter block is what makes this a V2 rule. Without it the
    # provider falls back to the deprecated `prefix` (V1) behaviour, and
    # delete_marker_replication becomes invalid.
    filter {}

    delete_marker_replication {
      status = var.delete_marker_replication ? "Enabled" : "Disabled"
    }

    source_selection_criteria {
      # WITHOUT THIS, KMS-ENCRYPTED OBJECTS ARE SILENTLY NOT REPLICATED.
      sse_kms_encrypted_objects { status = "Enabled" }
    }

    destination {
      bucket        = aws_s3_bucket.standby.arn
      storage_class = var.replica_storage_class

      encryption_configuration {
        replica_kms_key_id = aws_kms_key.standby.arn
      }

      # Always on. Four CloudWatch custom metrics per rule, a few dollars a
      # month, and the only way to observe replication at all.
      metrics {
        status = "Enabled"
        event_threshold { minutes = 15 }
      }

      # RTC requires metrics. Off by default - RPO is 2h, RTC is $0.015/GB.
      dynamic "replication_time" {
        for_each = var.enable_rtc ? [1] : []
        content {
          status = "Enabled"
          time { minutes = 15 }
        }
      }
    }
  }

  lifecycle {
    precondition {
      # The single highest-value assertion in this module. PutBucketReplication
      # does NOT validate the KMS key, returns 200, then fails every object.
      condition     = split(":", aws_kms_key.standby.arn)[3] == var.standby_region
      error_message = "replica_kms_key_id must be a key in the standby region. PutBucketReplication does not validate this and will return 200 while failing every object."
    }
  }
}
```

**Do not add an `existing_object_replication` block.** The provider documents it
and AWS rejects it:

> "The `existing_object_replication` parameter is not supported by Amazon S3 at
> this time and should not be included in your `rule` configurations. Specifying
> this parameter will result in `MalformedXML` errors."
> — [`aws_s3_bucket_replication_configuration`](https://github.com/hashicorp/terraform-provider-aws/blob/main/website/docs/r/s3_bucket_replication_configuration.html.markdown)

### Outputs — the contract with everything else

```hcl
output "primary_bucket_name"  { value = aws_s3_bucket.primary.bucket }
output "standby_bucket_name"  { value = aws_s3_bucket.standby.bucket }
output "primary_bucket_arn"   { value = aws_s3_bucket.primary.arn }
output "standby_bucket_arn"   { value = aws_s3_bucket.standby.arn }
output "primary_kms_key_arn"  { value = aws_kms_key.primary.arn }
output "standby_kms_key_arn"  { value = aws_kms_key.standby.arn }
output "replication_rule_id"  { value = "${var.name_prefix}-to-${var.standby_region}" }

# The Batch Replication runbook needs these. Emitting them as outputs is what
# keeps the backfill a one-line CLI call instead of a console session.
output "batch_replication_role_arn" { value = aws_iam_role.batch_replication.arn }
output "batch_manifest_bucket"      { value = aws_s3_bucket.batch_manifests.bucket }
output "batch_report_bucket"        { value = aws_s3_bucket.batch_reports.bucket }
```

### Region-local SSM parameters — the failover mechanism

```hcl
# The SAME parameter path in both regions, holding the region-local bucket name.
# An app reads its own region's copy and never knows which region is primary.
resource "aws_ssm_parameter" "bucket_name_primary" {
  provider = aws.primary
  name     = "/${var.env}/s3/${var.name_prefix}/bucket-name"
  type     = "String"
  value    = aws_s3_bucket.primary.bucket
}

resource "aws_ssm_parameter" "bucket_name_standby" {
  provider = aws.standby
  name     = "/${var.env}/s3/${var.name_prefix}/bucket-name"
  type     = "String"
  value    = aws_s3_bucket.standby.bucket
}
```

This is four lines of Terraform and it is the whole failover story for
application consumers. See [[aws-ssm-parameter-store]].

### Call site in the cookiecutter stack

```hcl
provider "aws" {
  alias  = "primary"
  region = var.primary_region   # from the cookiecutter env context
}

provider "aws" {
  alias  = "standby"
  region = var.standby_region
}

module "uploads_bucket" {
  source = "../../modules/s3-replicated-bucket"

  providers = {
    aws.primary = aws.primary
    aws.standby = aws.standby
  }

  name_prefix    = "${var.product}-${var.env}-uploads"
  primary_region = var.primary_region
  standby_region = var.standby_region

  # Live bucket - existing global-namespace name, cannot be renamed.
  primary_bucket_name_override = "helios-prod-uploads"

  replica_storage_class = "STANDARD_IA"
  lifecycle_rules = [
    { id = "expire-tmp", prefix = "tmp/", expiration_days = 7, noncurrent_days = 7 },
  ]
}

module "lambda_artifacts" {
  source = "../../modules/s3-replicated-bucket"
  providers = {
    aws.primary = aws.primary
    aws.standby = aws.standby
  }

  name_prefix    = "${var.product}-${var.env}-artifacts"
  primary_region = var.primary_region
  standby_region = var.standby_region

  # CI publishes to both regions. Standby bucket is created and configured,
  # but no replication rule and no Batch Replication backfill.
  replicate_data = false
}
```

Two module calls, one with replication and one without, both producing a fully
configured standby bucket. That is the shape that fits a templated monorepo: the
per-environment `terraform.tfvars` names the buckets and their posture, and
nothing about regions or replication mechanics leaks into the environment layer.
See [[module-patterns]].

### What Terraform does not do

Say it once more, in the section where someone will look for it: **Terraform does
not run the Batch Replication backfill.** There is no `aws_s3control_job`
resource. The module creates the Batch Operations role and the manifest/report
buckets; the runbook runs `aws s3control create-job`. Resist the temptation to
bolt it on with a `null_resource` + `local-exec` — a multi-hour, multi-thousand-
dollar, non-idempotent job invoked as a Terraform side effect is a way to
accidentally run it twice.

---

## Migration path from single-region

Live bucket in `eu-west-1`, zero downtime, no destroy/recreate. Nine steps.
Steps 1–3 can run days apart; steps 4–7 should be one change window per bucket.

### Step 0 — Classify and measure (before any change)

For every bucket in scope, record:

| Field | Why | How |
|---|---|---|
| Total size, object count | Backfill duration and cost | `BucketSizeBytes` / `NumberOfObjects` CloudWatch metrics, or Storage Lens |
| Mean and p99 object size | IA storage-class viability; large-object RPO risk | S3 Inventory + Athena |
| Versioning on/off | Whether step 2 applies | `aws s3api get-bucket-versioning` |
| Encryption (none / SSE-S3 / SSE-KMS) and *history* | Whether the SSE-S3→SSE-KMS permission trap applies | `get-bucket-encryption` + an Inventory report including encryption status |
| Lifecycle rules | Must be mirrored; must gain `noncurrent_version_expiration` | `get-bucket-lifecycle-configuration` |
| Objects in Glacier FR / Deep Archive | **Cannot be batch-replicated at all** | Inventory report by storage class |
| Consumers holding the name | The actual RTO risk | CloudTrail data events, grouped by `userIdentity` |
| Class (app data / logs / state / origin / artifacts / derived) | Determines which of the sections above applies | Human judgement |

**This step is not optional and it is not quick.** Every later decision — storage
class, backfill duration, whether the bucket is replicated at all — comes from
this table. A team that skips it discovers the Glacier objects during the
backfill, which is the expensive time to discover them.

### Step 1 — Create the standby bucket and KMS key (no replication yet)

`terraform apply` with `replicate_data = false`. This creates the standby bucket,
its KMS key, its access-log bucket, versioning, encryption, public access block,
lifecycle and the SSM parameters. **Nothing touches the primary bucket.** Safe,
reversible, zero risk.

Verify: write a test object to the standby bucket and read it back. If the KMS
grants are wrong you find out now, on a scratch object, not during the backfill.

### Step 2 — Enable versioning on the primary (if not already on)

**This is the only step with any operational risk, and it is a one-way door.**
Re-read [Versioning is a hard
prerequisite](#versioning-is-a-hard-prerequisite-and-turning-it-on-is-not-free).

1. Audit every lifecycle rule and add `noncurrent_version_expiration` **in the
   same change**. Without it, a bucket with an expiry rule and a high overwrite
   rate grows without bound and you find out on the bill.
2. Grep for `list-object-versions` and for code that depends on a 404 after
   delete.
3. Apply in a low-traffic window.
4. **Wait 15 minutes** before step 4 — AWS's documented recommendation.

If the bucket is already versioned, skip straight to step 3.

### Step 3 — Raise quotas

Before any replication traffic exists:

- **S3 replication data transfer rate.** Default 1 Gbps. If the bucket is over
  ~10 TB, raise it or the backfill runs for days. Service Quotas or a support
  case; not instant, so do it early.
- **KMS requests per second, in BOTH regions.** Replication makes roughly 2,000
  KMS requests per second per 1,000 objects per second replicated. The backfill
  is the largest KMS burst your account will ever produce and it will throttle
  *production* traffic. See [[aws-kms]].

### Step 4 — Turn on live replication

`terraform apply` with `replicate_data = true`. This adds the replication role,
its policies, the rule, the metrics and the alarms.

**Verify before proceeding — do not trust the green apply:**

```bash
# Canary: a new object must land in the standby within seconds.
aws s3api put-object --bucket "$SRC" --key "_canary/$(date +%s)" --body /dev/null
sleep 120
aws s3api head-object --bucket "$DST" --key "_canary/<same key>"   # expect 200

# And check the replication status the object itself reports.
aws s3api head-object --bucket "$SRC" --key "_canary/<same key>" \
  --query 'ReplicationStatus'   # expect "COMPLETED", not "PENDING" or "FAILED"
```

A `FAILED` here is almost always the KMS grant or the missing
`sse_kms_encrypted_objects` opt-in. Fix it now. **Every object written between
here and step 6 replicates automatically** — that is the point of doing live
replication before the backfill.

Wait 15 minutes for the configuration to propagate before step 5. AWS: *"If you
recently added or updated the replication configuration on the source bucket,
expect a delay of a few minutes before the change is fully propagated."*

### Step 5 — Run the Batch Replication backfill

Per [S3 Batch Replication for existing
objects](#s3-batch-replication-for-existing-objects). Summarised:

```bash
aws s3control create-job ... \
  --operation '{"S3ReplicateObject":{}}' \
  --priority 1 \
  --report '{... "ReportScope":"AllTasks"}' \
  --manifest-generator '{"S3JobManifestGenerator":{...
      "Filter":{"EligibleForReplication":true,
                "ObjectReplicationStatuses":["NONE"],
                "CreatedBefore":"<timestamp of step 4>"}}}'
```

Then: inspect the generated manifest's object count in `describe-job`, sanity
check the cost, resume the job, and **watch `OperationsFailedReplication` in the
source region** — it is the only one of the four metrics that covers Batch
Replication.

Disable lifecycle rules on both buckets for the duration if the job will run more
than a few hours. Re-enable them afterwards. Put both in the change ticket.

### Step 6 — Reconcile

1. Read the completion report. Non-zero failures on the first pass are normal.
2. Group failures by reason code. Fix the systemic ones (permissions, KMS).
3. Run a second job with `ObjectReplicationStatuses: ["FAILED"]`.
4. For anything still failing: the version-ID-delete case needs a **Batch Copy**
   job instead; the Glacier FR/DA case cannot be batch-replicated at all and
   needs the decision from step 0.
5. Reconcile totals — but **not by object count** (see [What not to
   monitor](#what-not-to-monitor)). Use the completion report's task count
   against the manifest's object count.

### Step 7 — De-hardcode the consumers

Using the CloudTrail list from step 0, move every consumer to the SSM parameter.
Order matters: do this **after** the standby has data, so that a consumer
accidentally pointed at the standby reads real objects rather than 404s.

This is the longest step in elapsed time and the one that actually determines
your RTO. Track it per consumer, not as one ticket.

### Step 8 — Test the failover properly

The test that proves nothing: deploy to the standby region and check the app
starts. It will pass even with every bucket name hardcoded to the primary,
because `eu-west-1` buckets are reachable from `eu-west-2`.

**The test that proves something: block network egress from the standby region's
VPC to the primary region's S3 endpoints, then run the application.** Anything
that breaks was holding a primary bucket name. Run this at least once before
declaring the bucket migrated, and put it in the regular DR exercise. See
[[lessons-and-antipatterns]].

### What does *not* force replacement

Good news, because the brief asks about it explicitly. None of these cause a
destroy/recreate on the live bucket:

| Change | `ForceNew`? |
|---|---|
| Enabling versioning (`aws_s3_bucket_versioning`) | No — separate resource |
| Adding `aws_s3_bucket_replication_configuration` | No — separate resource |
| Changing `storage_class` on the destination | No |
| Adding/changing `metrics`, `replication_time` | No |
| Adding `aws_s3_bucket_server_side_encryption_configuration` | No |
| Changing lifecycle configuration | No |
| **Changing `bucket`** | **YES — `(Optional, Forces new resource)`** |
| **Changing `bucket_namespace`** | **YES — `(Optional, Forces new resource)`** |

**The two `ForceNew` rows are the same fact in two costumes: you cannot rename or
re-namespace a bucket.** If a `terraform plan` on a live bucket ever shows
`# aws_s3_bucket.primary must be replaced`, stop. Something changed
`local.primary_bucket_name` — most likely someone removed
`primary_bucket_name_override`, or the account ID / region interpolation changed.
This is exactly why `prevent_destroy = true` is on both buckets in the module: it
turns a catastrophic apply into a failed plan.

---

## Failover procedure

At 3am, for S3, assuming everything above is in place.

| # | Step | Automated? | Time |
|---|---|---|---|
| 1 | Decide to fail over | **Human.** See [[failover-orchestration]]. | The whole RTO risk |
| 2 | Route 53 / traffic swing to the standby region | Automated | seconds |
| 3 | Standby compute starts, reads `/{env}/s3/{purpose}/bucket-name` from *its own* region's SSM | Automated | seconds |
| 4 | Standby bucket serves reads and accepts writes | Already true | 0 |
| 5 | CloudFront origin group has already failed over for cached/read paths | Automatic, ~2–30s per uncached object | seconds |
| 6 | **Do not** reverse the replication rule | — | — |
| 7 | **Do not** create or reconfigure any bucket | — | — |
| 8 | Record the failover timestamp | Human | seconds |

**S3's contribution to the RTO is effectively zero.** There is no promotion, no
endpoint swing, no capacity to warm. If the three pre-provisioning conditions
from the [RPO/RTO analysis](#against-rto-15-minutes) hold, S3 is done before the
human has finished reading the alert.

### Steps 6 and 7 deserve emphasis

**Do not reverse the replication rule during the failover.** It is tempting —
"now `eu-west-2` is primary, so replication should go the other way" — and it is
wrong for three reasons:

1. `PutBucketReplication` has a **`us-east-1` control-plane dependency**. If the
   failover is during a `us-east-1` event, the call may simply fail, and you will
   burn RTO discovering that.
2. The old primary may come back mid-incident. A reversed rule pointed at a
   bucket that is intermittently available produces replication failures and, if
   delete-marker replication is on, can propagate deletes in a direction nobody
   intended.
3. You do not need it. Data written to the standby after promotion is safe in the
   standby. Getting it back to the old primary is a **failback** problem, handled
   deliberately, during business hours.

**Do not create or modify any bucket during a failover.** `CreateBucket`,
`PutBucketVersioning`, `PutBucketPolicy`, `PutBucketEncryption`,
`PutBucketNotification` and ~20 other configuration APIs have a `us-east-1`
dependency. Everything must be pre-provisioned. If your failover runbook contains
a `terraform apply`, the runbook is wrong. See [[aws-regional-outages]].

### The one thing to check manually

```bash
# How far behind was the standby at the moment of failover? This is your actual
# data loss, and someone will ask. ReplicationLatency is in the STANDBY region.
aws cloudwatch get-metric-statistics \
  --region "$STANDBY_REGION" \
  --namespace AWS/S3 --metric-name ReplicationLatency \
  --dimensions Name=SourceBucket,Value="$SRC" \
               Name=DestinationBucket,Value="$DST" \
               Name=RuleId,Value="$RULE_ID" \
  --start-time "$(date -u -d '2 hours ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time   "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period 300 --statistics Maximum
```

Capture this *early* — CloudWatch retains it, but having the number in the
incident channel within the first few minutes changes the conversation about what
was lost.

---

## Failback

Harder than failover, always forgotten, and for S3 it has a specific shape: **the
two buckets have diverged and there is no merge operation.**

### What diverged

While `eu-west-2` was primary:

- New objects were written **only** to `eu-west-2`.
- Objects deleted in `eu-west-2` still exist in `eu-west-1`.
- `eu-west-1` may hold objects written in the last minutes before the outage that
  never replicated — **these are real data and they are only in the old primary.**

S3 gives you nothing to reconcile this. There is no "sync and resolve conflicts".

### The procedure

**1. Decide whether to fail back at all.** Genuinely ask. If `eu-west-2` is
serving fine, the cheapest correct answer is often **promote the standby
permanently and build a new standby in `eu-west-1`**. This is why the naming
convention forbids `primary`/`standby` in bucket names — the roles are meant to
be swappable. Reasons to actually fail back: data residency, latency to users,
reserved capacity, or other services in the estate that cannot swap as cleanly.

**2. Salvage the unreplicated tail from the old primary.** Before anything
overwrites it, identify objects that exist in `eu-west-1` and not in `eu-west-2`:
run S3 Inventory on both buckets, load both reports into Athena, and diff on
`key` + `etag`. Objects only in `eu-west-1`, written near the outage window, are
candidates for manual recovery into `eu-west-2`. **Do this before step 3, because
step 3 will eventually overwrite them.**

**3. Create the reverse replication rule, `eu-west-2` → `eu-west-1`.** A second
replication role and a second rule. **This is when you create it — not at
failover time.**

Two options for how it lives in Terraform:

| | **A: create it now, at failback time** | **B: keep it permanently, disabled** |
|---|---|---|
| Shape | A `reverse_replication` module flag flipped on during failback | Both rules always exist; the reverse one has `status = "Disabled"` |
| Risk | A `terraform apply` during a stressful period | Enabling a rule is a smaller, better-understood change |
| Gotcha | `PutBucketReplication` has a `us-east-1` dependency — but failback is planned, so you can wait | Someone enables both directions at once and creates a replication loop |
| Cost | $0 either way | $0 either way |

**Recommendation: B, with the reverse rule permanently present and disabled**, and
a loud comment plus a CI check asserting that exactly one of the two rules is
`Enabled` at any time. The cost is zero and it removes a control-plane call from
a recovery path. Replication is not transitive, so A→B→A will not loop
*objects*, but two enabled rules with delete-marker replication on both sides is
a genuinely bad configuration and the check is worth having.

**4. Backfill in reverse.** Everything written to `eu-west-2` during the outage
is "existing" from the reverse rule's point of view — so it needs **its own S3
Batch Replication job**, `eu-west-2` → `eu-west-1`, filtered to
`ObjectReplicationStatuses: ["NONE"]` and `CreatedAfter` the failover timestamp.
This is why step 8 of the failover procedure is "record the failover timestamp":
that timestamp is the filter bound, and without it you re-replicate the entire
bucket at full cost.

**Cost warning:** if the standby is `STANDARD_IA` (the recommendation), this
backfill reads every object out of IA and you pay **$0.01/GB retrieval** on the
whole reverse-replicated set, on top of inter-region data transfer. For a 10 TB
bucket that is ~$100 of retrieval plus ~$205 of transfer. Not large, but put it
in the plan rather than on the invoice.

**5. Cut traffic back, then reverse the rules.** Traffic first, replication
direction second, and leave a settling period between them. Verify
`ReplicationLatency` in `eu-west-1` has caught up and
`OperationsPendingReplication` is at zero before swinging traffic.

**6. Fix the storage classes.** After failback, the objects in `eu-west-1` that
came back via reverse replication have whatever class the reverse rule specified.
If you set `STANDARD_IA` on the reverse rule, your *primary* is now partly IA —
which is wrong, because the primary serves live traffic and pays retrieval fees
on every read. **Set the reverse rule's `storage_class` to `STANDARD`.** It is a
different value from the forward rule and that asymmetry is deliberate.

### The failback rehearsal

Failback is the step nobody has ever run. Rehearse it on a non-production bucket
at least once, with a real reverse Batch Replication job, and time it. Until you
have, your failback plan is a document rather than a procedure.

---

## Gotchas

The list that makes the note worth reading. Everything here is sourced above.

1. **`PutBucketReplication` does not validate the destination KMS key.** Wrong
   region, wrong key, deleted key — all return `200 OK`, then fail every object
   forever. The module `precondition` in the HCL above exists solely for this.
2. **KMS-encrypted objects are not replicated unless you explicitly opt in.**
   Omit `sse_kms_encrypted_objects` and nothing replicates, silently, with no
   error anywhere.
3. **`s3:GetObjectVersion` is not `s3:GetObjectVersionForReplication`.** The
   former replicates plaintext and SSE-S3 objects and silently skips KMS ones. A
   role built from an old blog post has exactly this bug.
4. **Enabling S3 Bucket Keys breaks a working replication role.** The KMS
   encryption context changes from the object ARN to the bucket ARN, so the
   `kms:EncryptionContext:aws:s3:arn` condition must lose its `/*`. Teams enable
   Bucket Keys for the KMS cost saving and break replication as a side effect.
5. **Multi-Region KMS keys do nothing for S3.** *"Amazon S3 currently treats
   multi-Region keys as though they were single-Region keys."* You pay MRK
   pricing for no benefit.
6. **CRR does not replicate existing objects, and Terraform cannot make it.**
   `existing_object_replication` → `MalformedXML`. There is no `aws_s3control_job`
   resource. The backfill is a runbook step.
7. **An unfiltered Batch Replication re-run re-copies the whole bucket.** Always
   set `ObjectReplicationStatuses`.
8. **Glacier Flexible Retrieval and Deep Archive objects cannot be
   batch-replicated at all.** Not slow — unsupported. Discover this in step 0, not
   during the job.
9. **Lifecycle rules must be disabled during a long backfill** or the two buckets
   structurally diverge via the delete-marker-before-versions race.
10. **Enabling versioning changes what `Expiration` means.** Without
    `noncurrent_version_expiration`, a versioned bucket with an expiry rule grows
    forever. Audit every lifecycle rule in the same change.
11. **Versioning cannot be turned off, only suspended.** One-way door with a
    permanent cost floor.
12. **Delete markers are not replicated by default on V2 rules, but *are* on V1
    rules.** Same config, opposite behaviour, distinguished only by whether a
    `filter` block is present.
13. **Deletes by version ID are never replicated. There is no setting.** And
    Batch Replication cannot repair the resulting gap — only a Batch *Copy* can.
14. **Object-count parity is not a replication health check.** It will be
    permanently red and will train the team to ignore replication alarms.
15. **Replication metrics are split across two regions.** Three in the
    destination, `OperationsFailedReplication` in the source. One-region dashboards
    lie.
16. **Only `OperationsFailedReplication` covers Batch Replication.** The lag
    metrics say nothing about backfill progress. And a job that fails to start
    emits no metrics at all.
17. **`INSUFFICIENT_DATA` is the normal alarm state.** Set
    `treat_missing_data = "ignore"`, per AWS's own guidance, or the alarm gets
    muted within a week.
18. **A hardcoded primary bucket name works perfectly from the standby region**
    until the primary region is actually unreachable. Your DR test must block
    egress to the primary S3 endpoints or it proves nothing.
19. **Bucket names cannot be changed. `bucket` and `bucket_namespace` are both
    `ForceNew`.** If a plan on a live bucket says "must be replaced", stop.
20. **Existing buckets cannot be moved into the account regional namespace.**
    *"Because existing buckets can't be renamed, migration requires creating a new
    bucket and transitioning workloads to it."*
21. **The account regional suffix eats your name budget.** 63 − 29 = **34
    characters** for the prefix, sized for `ca-central-1`, not 37.
22. **`CreateBucket` and ~24 `PutBucket*` APIs have a `us-east-1` dependency.**
    Never create or reconfigure a bucket during a failover.
23. **Server access log targets must be in the same region as the source
    bucket.** The standby needs its own log bucket or `PutBucketLogging` fails.
24. **Object Lock buckets cannot be server-access-log targets.**
25. **Object Lock is a one-way door** and retrofitting it onto a replicating
    bucket needs an Object Lock token from AWS Support.
26. **Lambda deployment packages must be in the same region as the function.**
27. **CloudFront origin groups only fail over `GET`/`HEAD`/`OPTIONS`**, only if
    `OPTIONS` is in the cached methods list, and retry the primary for up to 30s
    on every request by default.
28. **MRAP's control plane is `us-west-2` only and its failover control plane
    excludes Canada entirely.**
29. **IA storage classes have a 128 KB minimum billable object size.** For
    small-object buckets, `STANDARD_IA` is *more* expensive than `STANDARD`.
30. **The RTC SLA excludes regional S3 impairment** — the exact event you are
    failing over for — and is voided while you exceed 1 Gbps of replication
    traffic, which a backfill will.
31. **The Terraform state bucket can be inside the outage you are recovering
    from.** Design so that failover needs no `terraform apply`.

---

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| **Replication mechanism** | Plain CRR | CRR + RTC ($0.015/GB) | **Plain CRR.** RPO 2h is met with orders of magnitude to spare; RTC is an RPO instrument, not an RTO one, and its SLA excludes the disaster case. |
| **Replication metrics** | Off (free) | On (4 CloudWatch custom metrics per rule) | **On, everywhere.** It is the cheap half of RTC and the only way to see replication at all. |
| **Delete-marker posture** | Mirror (`Enabled`) | Archive (`Disabled`) | **Mirror for application data** — a standby that resurrects deleted objects on promotion is a correctness bug. **Archive for audit-log buckets**, where non-propagation is the point. Get deletion protection from [[aws-backup]], not from replication. |
| **Replica storage class** | `STANDARD` | `STANDARD_IA` (−45%) / `GLACIER_IR` (−79%) | **`STANDARD_IA` by default.** Exceptions: mean object size < 256 KB, or high overwrite rate. `GLACIER_IR` for cold archives. Never `GLACIER`/`DEEP_ARCHIVE`/`INTELLIGENT_TIERING`. |
| **KMS key type** | Two single-region keys | One multi-Region key | **Two single-region keys.** S3 treats MRKs as single-Region keys; the MRK buys nothing and costs replica-key pricing. |
| **Bucket naming** | Keep global-namespace names | Account regional namespace (`-an`) | **`-an` for every new bucket** (including all standbys), enforced by SCP. Leave live primaries on their global names; a primary rename is a second cutover competing for the same risk budget. |
| **Consumer name resolution** | Terraform outputs / env vars baked at deploy | Region-local SSM parameter, same path both regions | **SSM, same path, region-local value.** The application never learns which region is primary. Four lines of Terraform; the entire failover story for app consumers. |
| **Multi-Region Access Points** | Use them | Don't | **Don't.** Adds a `us-west-2` control-plane dependency, excludes Canada from failover control, requires SigV4A, breaks gateway VPC endpoints and OAC, and solves an active/active problem we don't have. Revisit only if the posture changes to active/active. |
| **Batch Replication manifest** | S3-generated | User-supplied (Inventory/CSV) | **S3-generated for the first backfill of every bucket.** It inherits the rule's filter and covers all eligible versions. Supplied manifests only for key-range chunking of very large buckets. |
| **Terraform state location** | In the primary region, replicated | In a third region | **Neither is the real fix — design so failover needs no `terraform apply`.** Then a third region for state, with a read-only replica in the standby. **Open question for the CA pair**, which has only two Canadian regions. |
| **CloudTrail buckets** | Replicate per-region buckets | One central log-archive bucket per jurisdiction | **One central bucket per jurisdiction** (EU, US, Canada). One copy instead of six, and each jurisdiction's audit trail stays inside its borders. |
| **Lambda artifacts** | Replicate the bucket | Publish from CI to both regions | **Publish from CI.** Reproducible content, verifiable at build time, no backfill needed. |
| **Reverse replication rule** | Create at failback time | Exists permanently, `Disabled` | **Exists permanently, disabled**, with a CI check that exactly one direction is enabled. Removes a `us-east-1`-dependent control-plane call from the recovery path. |

---

## Cost

### What the standby costs while idle

S3 has no idle capacity, so the standby's cost is the duplicated bytes plus a
rounding error. All prices from the AWS Price List API, retrieved 21 September
2026, eu-west-2 (EU pair standby) unless noted.

| Component | Rate | Notes |
|---|---|---|
| Standby bucket itself | **$0** | Buckets are free |
| **Replicated storage, `STANDARD`** | **$0.024/GB-mo** (first 50 TB) | The dominant line |
| **Replicated storage, `STANDARD_IA`** | **$0.0131/GB-mo** | **The lever.** −45% |
| **Replicated storage, `GLACIER_IR`** | **$0.005/GB-mo** | −79%, for cold data |
| Inter-region data transfer out | **$0.02/GB** | Ongoing, proportional to write volume — not to stored volume |
| Destination PUT requests | **$0.0053/1,000** (Standard) / $0.01 (SIA) / $0.02 (GIR) | One PUT per replicated object |
| Destination KMS key | ~$1/key/month + per-request | Bucket Keys cut the per-request cost substantially |
| Replication metrics | 4 CloudWatch custom metrics per rule | A few dollars a month across the estate |
| CloudWatch alarms | ~2 per bucket | Cents |
| RTC surcharge (**if enabled**) | **$0.015/GB** | Recommendation: don't |
| Storage Lens free tier | **$0.00** | Leave it on |
| Storage Lens advanced metrics | **$0.20 per million objects/mo** (first 25 B) | Migration-window cost, reassess after |

Regional storage price check across all three pairs (S3 Standard, first 50 TB):

| Region | $/GB-mo |
|---|---|
| `eu-west-1` (EU primary) | **$0.023** |
| `eu-west-2` (EU standby) | **$0.024** |
| `us-east-1` (US primary) | **$0.023** |
| `us-west-2` (US standby) | **$0.023** |
| `ca-central-1` (CA primary) | **$0.025** |
| `ca-west-1` (CA standby) | **$0.025** |

Useful detail: **`ca-west-1` is priced identically to `ca-central-1`**, and
`us-west-2` identically to `us-east-1`. Only the EU standby carries a premium,
and it is 4%. **Region choice is not a meaningful cost lever here. Storage class
is.**

### One-off migration cost

Per the [worked example](#cost-of-the-backfill) — a 10 TB, 20-million-object
bucket, eu-west-1 → eu-west-2:

| | Cost |
|---|---|
| Batch job fee + object ops + manifest | $20.55 |
| Inter-region data transfer (10,240 GB) | $204.80 |
| Destination PUTs (20M, to `STANDARD_IA`) | $200.00 |
| Source GETs | $8.00 |
| **One-off total** | **≈ $433** |

### Ongoing, same bucket

| Storage class | Monthly | Annual |
|---|---|---|
| `STANDARD` | $245.76 | $2,949 |
| **`STANDARD_IA`** | **$134.14** | **$1,610** |
| `GLACIER_IR` | $51.20 | $614 |

Plus ongoing inter-region transfer proportional to the *write* rate: at 500 GB of
new writes per month that is $10/month, not a factor.

### The shape of the answer

**The migration is cheap and the standby is a permanent line item.** $433 once
versus $134–$246 every month, forever, per bucket. Two consequences for how you
talk about this internally:

1. **Do not let cost block the migration.** The one-off number is small enough to
   approve without a business case.
2. **Do put real effort into the storage class and into which buckets get
   replicated at all.** Those two decisions, made per bucket in step 0, are worth
   more than every other cost optimisation in this note combined. Choosing
   `STANDARD_IA` over `STANDARD` on 100 TB of replicated data saves roughly
   **$13,400/year per region pair**. Deciding that a 20 TB derived-data bucket
   doesn't need replicating at all saves **$5,900/year** and one Batch
   Replication job.

See [[cost-model]] for how this rolls up with the rest of the estate.

---

## Open questions

Things that need an answer from inside the company before this plan is final.

1. **The bucket inventory.** How many buckets, in which classes, at what size and
   object count? Step 0 of the migration path is a prerequisite for every
   estimate in this note. Nothing here can be costed without it.
2. **Mean and p99 object size per bucket.** Determines whether `STANDARD_IA` saves
   money or costs money (the 128 KB minimum), and whether plain CRR meets RPO 2h
   (the multi-GB-object problem). **Anything with a p99 over ~1 GB needs its own
   measured RPO, not an inherited one.**
3. **Are there Glacier Flexible Retrieval or Deep Archive objects in scope?** They
   cannot be batch-replicated. If yes, the decision is: accept a partial standby,
   restore-and-re-archive, or route the archive tier through [[aws-backup]].
4. **Which buckets have had their encryption changed from SSE-S3 to SSE-KMS?**
   AWS calls out extra permissions specifically for this case, and only the
   backfill will surface it.
5. **Which buckets are unversioned today?** Each one needs the versioning step,
   with its lifecycle audit, and it is a one-way door with a permanent cost floor.
6. **Where does Terraform state live, and does any failover path require
   `terraform apply`?** If yes, the RTO is not 15 minutes regardless of what S3
   does. **And specifically for the CA pair: with only two Canadian regions, where
   does CA state live without either leaving Canada or being circular?**
7. **RTO definition.** 15 minutes from *incident start* or from *decision to fail
   over*? The whole vault turns on this and S3 is one of the few services where
   the answer doesn't change the design — but the number in the runbook should be
   honest. Flagged in `CLAUDE.md` as an estate-wide open thread.
8. **Is anyone outside the company holding a bucket name?** Partner integrations
   and shipped mobile clients cannot be fixed at 3am. If the answer is yes, the
   RTO for those paths includes a phone call.
9. **Does the Canadian deployment's compliance posture permit CloudTrail logs, S3
   Inventory reports or Batch Replication manifests leaving Canada?** Everything
   in this note assumes it does not, and routes accordingly. Confirm with whoever
   owns [[data-residency]].
10. **Who owns the Batch Replication backfill runbook, and who is authorised to
    approve a job that will cost several hundred dollars and run for days?** The
    `Suspended`-then-resume gate in the runbook needs a named human.
11. **Should the primaries eventually migrate to the account regional namespace?**
    This note recommends deferring it. Someone should own the decision to revisit,
    with a trigger, rather than letting it become permanent by default.

---


## Sources

All URLs below were fetched and read during the research for this note. Nothing
here is quoted from memory. Where a claim could not be sourced, the note says so
in the body rather than citing anything.

### AWS documentation — replication core

| Source | What it contributes |
|---|---|
| [What does Amazon S3 replicate?](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-what-is-isnot-replicated.html) | The definitive replicated / not-replicated list; the "deletes by version ID are never replicated" quote. |
| [Requirements and considerations for replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-requirements.html) | Versioning prerequisite; "for large objects, it can take several hours"; the per-object request amplification (up to five GET/HEAD + one PUT to source, one PUT per destination). |
| [Replicating delete markers between buckets](https://docs.aws.amazon.com/AmazonS3/latest/userguide/delete-marker-replication.html) | V1 vs V2 delete-marker defaults; tag-based rules unsupported; delete-marker replication excluded from the RTC SLA. |
| [Replicating objects created with server-side encryption (SSE-C, SSE-S3, SSE-KMS, DSSE-KMS)](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-config-for-kms-objects.html) | `sse_kms_encrypted_objects` opt-in; "`PutBucketReplication` doesn't check the validity of KMS keys"; multi-Region keys treated as single-Region; the `kms:ViaService` region-scoped policy example; `s3:GetObjectVersionForReplication` vs `s3:GetObjectVersion`. |
| [Meeting compliance requirements with S3 Replication Time Control](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-time-control.html) | The "99.9% within 15 minutes" claim; the 1 Gbps default replication data-transfer-rate quota and that it is raisable via Service Quotas. |
| [Amazon S3 Replication Time Control Feature SLA](https://aws.amazon.com/s3/sla-rtc/) | Verbatim SLA text, the service-credit table, and the exclusions (regional S3 performance, request-rate guideline breaches, >1 Gbps replication rate). Last updated 4 May 2022. |

### AWS documentation — Batch Replication

| Source | What it contributes |
|---|---|
| [Replicating existing objects with Batch Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-batch-replication-batch.html) | The whole Batch Replication section: preconditions, generated vs supplied manifests, `ObjectReplicationStatuses` filters, the 20-billion-object manifest ceiling, the lifecycle-rule race, job concurrency and priority pre-emption, Glacier FR/DA being unsupported, the version-ID-delete escape hatch via Batch Copy, and the SSE-S3→SSE-KMS permission note. |
| [Configuring an IAM role for S3 Batch Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-batch-replication-policies.html) | The `batchoperations.s3.amazonaws.com` trust policy and both permissions-policy variants, verbatim. |
| [Tracking job status and completion reports](https://docs.aws.amazon.com/AmazonS3/latest/userguide/batch-ops-job-status.html) | Completion-report scope options. |
| [Amazon S3 replication failure reasons](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-metrics-events.html) | Failure-code catalogue referenced by the completion report. |
| [Troubleshooting replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-troubleshoot.html) | Batch Replication error triage. |
| [`aws s3control create-job`](https://docs.aws.amazon.com/cli/latest/reference/s3control/create-job.html) | The CLI surface the runbook command is built from, including `S3JobManifestGenerator` and the `Suspended` initial state when confirmation is required. |

### AWS blogs and announcements

| Source | What it contributes |
|---|---|
| [NEW – Replicate Existing Objects with Amazon S3 Batch Replication](https://aws.amazon.com/blogs/aws/new-replicate-existing-objects-with-amazon-s3-batch-replication/) (8 Feb 2022) | Launch announcement; the only AWS statement on backfill duration — "existing objects can take longer to replicate than new objects, and the replication speed largely depends on the AWS Regions, size of data, object count, and encryption type". |
| [Accelerate Amazon S3 Replication with automated S3 Batch Operations parallelization](https://aws.amazon.com/blogs/storage/accelerate-amazon-s3-replication-with-automated-s3-batch-operations-parallelization/) | The key-range parallelization technique for buckets past the 20-billion-object ceiling; references a real migration of "over 50 billion objects" but publishes no duration. |
| [Amazon S3 introduces account regional namespaces for general purpose buckets](https://aws.amazon.com/about-aws/whats-new/2026/03/amazon-s3-account-regional-namespaces) (12 Mar 2026) | Launch date, "at no additional cost", the "37 AWS Regions" availability claim. |
| [Migrate to Amazon S3 account regional namespaces](https://aws.amazon.com/blogs/storage/migrate-to-amazon-s3-account-regional-namespaces/) | "Because existing buckets can't be renamed, migration requires creating a new bucket and transitioning workloads to it"; the five-step replicate-and-cutover migration; the advice to keep the old bucket forever rather than delete it. |

### AWS documentation — special-case buckets

| Source | What it contributes |
|---|---|
| [Receiving CloudTrail log files from multiple Regions](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/receive-cloudtrail-log-files-from-multiple-regions.html) | "the bucket for a multi-Region trail does not have to be in the trail's home Region" — the basis for not replicating CloudTrail buckets. |
| [Enabling Amazon S3 server access logging](https://docs.aws.amazon.com/AmazonS3/latest/userguide/enable-server-access-logging.html) | "The target bucket must be in the same AWS Region and AWS account as the source bucket", and the `Cross S3 location logging not allowed` error. |
| [Object Lock considerations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock-managing.html) | Destination buckets must also have Object Lock; the `s3:GetObjectRetention` / `s3:GetObjectLegalHold` requirement; "you can't disable Object Lock or suspend versioning"; "S3 buckets with Object Lock can't be used as destination buckets for server access logs"; and the *absence* of any current "contact AWS Support" requirement, contradicting older re:Post material. |
| [Optimize high availability with CloudFront origin failover](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/high_availability_origin_failover.html) | The allowed failover status codes (400, 403, 404, 416, 429, 500, 502, 503, 504); "CloudFront fails over to the secondary origin only when the HTTP method … is `GET`, `HEAD`, or `OPTIONS`"; the `OPTIONS`-must-be-cached caveat; the 30-second / 3-attempt default and the 1–3 attempts and 1–10 second connection-timeout tuning range; and that CloudFront re-tries the primary on every request. |
| [`CreateFunction` API reference](https://docs.aws.amazon.com/lambda/latest/APIReference/API_CreateFunction.html) | "An Amazon S3 bucket must be in the same Amazon Web Services Region as your Lambda function." |
| [Terraform S3 backend documentation](https://developer.hashicorp.com/terraform/language/backend/s3) | `use_lockfile` and S3-native state locking; "DynamoDB-based locking is deprecated and will be removed in a future minor version"; the recommendation to enable bucket versioning on the state bucket. |

### AWS documentation — monitoring

| Source | What it contributes |
|---|---|
| [Using S3 Replication metrics](https://docs.aws.amazon.com/AmazonS3/latest/userguide/repl-metrics.html) | The per-metric table stating which Region each metric is published in (three in the destination, `OperationsFailedReplication` in the source), which metrics cover Batch Replication, whether they survive destination-bucket deletion, the "billed at the same rate as Amazon CloudWatch custom metrics" statement, that metrics can be enabled independently of RTC, and the 15-minute reporting delay after enabling RTC. |
| [Metrics and dimensions](https://docs.aws.amazon.com/AmazonS3/latest/userguide/metrics-dimensions.html) | Exact metric names, units and valid statistics for `ReplicationLatency`, `BytesPendingReplication`, `OperationsPendingReplication`, `OperationsFailedReplication`; the `SourceBucket` / `DestinationBucket` / `RuleId` dimensions; and the "Treat missing data as ignore" alarm guidance. |
| [Monitoring replication with metrics, event notifications, and statuses](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-metrics.html) | The four `s3:Replication:*` event types; the `PENDING`/`COMPLETED`/`FAILED`/`REPLICA` object replication statuses; the Storage Lens free vs advanced replication metrics, including "the count of replication rules with a replication destination that's not valid". |
| [Receiving replication failure events with Amazon S3 Event Notifications](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-metrics-events.html) | Per-object failure events and failure reason codes. |
| [Amazon CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/) | Referenced for the custom-metric rate; deliberately not quoted, as it was not fetched for this note. |

### AWS documentation — Multi-Region Access Points

| Source | What it contributes |
|---|---|
| [Multi-Region Access Point restrictions and limitations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/MultiRegionAccessPointRestrictions.html) | The whole MRAP restrictions table: `us-west-2`-only control plane, the five failover-control regions, SigV4A and the global-STS caveat, Batch Operations unsupported, no gateway VPC endpoints, no IPv6, CloudFront `Custom Origin` requirement, frozen bucket set, no anonymous requests, the 100-MRAP and 17-Region quotas, and the default/opt-in region lists that confirm `ca-west-1` is a supported bucket region. |
| [Amazon S3 Multi-Region Access Points routing states](https://docs.aws.amazon.com/AmazonS3/latest/userguide/FailoverConfiguration.html) | Active-active vs active-passive routing semantics. |
| [S3 Multi-Region Access Points now support failover controls](https://aws.amazon.com/about-aws/whats-new/2022/11/amazon-s3-multi-region-access-points-failover-active-passive-configurations-failovers) (Nov 2022) | The "within 2 minutes" failover figure. |
| [Amazon S3 Multi-Region Access Points — SDK compatibility](https://docs.aws.amazon.com/sdkref/latest/guide/feature-s3-mrap.html) | Which SDKs implement SigV4A. |

### AWS documentation — naming and namespaces

| Source | What it contributes |
|---|---|
| [General purpose bucket naming rules](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucketnamingrules.html) | Partition-global uniqueness, the four partitions, "you can't change its name or Region", the reserved-suffix list including `-an`, and the 3–63 character limit. |
| [Namespaces for general purpose buckets](https://docs.aws.amazon.com/AmazonS3/latest/userguide/gpbucketnamespaces.html) | Account regional namespace mechanics: the `{prefix}-{accountId}-{region}-an` shape, the `x-amz-bucket-namespace: account-regional` header, the 37-characters-of-prefix budget, the "all Regions except Middle East (Bahrain) and Middle East (UAE)" availability statement, and the IAM / RCP / SCP examples using the `s3:x-amz-bucket-namespace` condition key. |

### Terraform provider

| Source | What it contributes |
|---|---|
| [hashicorp/terraform-provider-aws#43746 — S3 ExistingObjectReplication is no longer supported via s3api](https://github.com/hashicorp/terraform-provider-aws/issues/43746) | Confirms `existing_object_replication` returns `MalformedXML` and is not a customer-facing API. |
| [hashicorp/terraform-provider-aws#18538 — Feature Request: Support for S3 Batch Operations](https://github.com/hashicorp/terraform-provider-aws/issues/18538) | Original (closed) request for Batch Operations support, April 2021. |
| [hashicorp/terraform-provider-aws#46231 — Create aws_s3control_job resource for Batch Operations Jobs](https://github.com/hashicorp/terraform-provider-aws/issues/46231) | Current open request, opened 30 January 2026; establishes that there is still no way to create a Batch Replication job from Terraform. |
| [`aws_s3_bucket_versioning` registry docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_versioning) | The verbatim 15-minute post-enable write-quiesce recommendation. |
| [`aws_s3_bucket_replication_configuration` registry docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_replication_configuration) | Argument surface used in the HCL in this note. |
| [`aws_s3_bucket` resource docs, provider `main`](https://github.com/hashicorp/terraform-provider-aws/blob/main/website/docs/r/s3_bucket.html.markdown) | The verbatim `bucket_namespace` argument text — "(Optional, Forces new resource) … Valid values: `account-regional`, `global`. Defaults to `global`" — the `bucket_prefix` 37-character cap, and the official account-regional example, which builds the full name with `format()` rather than having the provider append the suffix. |
| [hashicorp/terraform-provider-aws#46902 — Account regional namespaces for general purpose buckets](https://github.com/hashicorp/terraform-provider-aws/issues/46902) | The provider issue that tracked adding `bucket_namespace`. |

### Pricing

All prices in this note come from the **AWS Price List Bulk API**, not from the
marketing pricing pages, because the pricing pages do not expose per-region
per-usage-type detail. Prices retrieved 21 September 2026.

| Source | What it contributes |
|---|---|
| `https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonS3/current/<region>/index.json` | Storage-class prices per region; `BatchOperations-Jobs` / `-Objects` / `-Manifest`; `S3RTC-Out-Bytes`; request tiers; Storage Lens and S3 Inventory object charges. Retrieved for `eu-west-1`, `eu-west-2`, `us-east-1`, `us-west-2`, `ca-central-1`, `ca-west-1`. |
| `https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSDataTransfer/current/<region>/index.json` | `InterRegion Outbound` prices for the three pairs. Retrieved for `eu-west-1`, `us-east-1`, `ca-central-1`. |
| [AWS Price List Bulk API documentation](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/price-changes.html) | How to reproduce the above. |
