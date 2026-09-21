---
title: Amazon S3 — Multi-Region
service: s3
tags: [service, multi-region, s3, replication, storage]
status: partial
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

## Still to research

Cut off by a session limit. What is written above is sound; what is missing is
roughly half the note, including the section that matters most for migration.

- **`## S3 Batch Replication for existing objects` — the biggest gap.** CRR only
  replicates objects written *after* the rule exists. Every object already in the
  live bucket needs an explicitly invoked Batch Replication job. Cover job
  scoping, manifests vs auto-generated inventories, completion reporting, IAM and
  cost. Without this section the note does not answer the brief's
  migrate-a-live-resource requirement. (The KMS throttling section above already
  assumes this backfill exists — it is referenced but never explained.)
- `## Bucket names are globally unique` — the standby bucket cannot share a name,
  so every consumer holding a hardcoded name breaks at failover. Naming
  convention and parameterisation for a cookiecutter monorepo.
- `## Multi-Region Access Points` — what they solve, SigV4A signing, cost, and a
  recommendation on whether they are worth it here.
- `## Monitoring replication lag` — `ReplicationLatency`,
  `OperationsPendingReplication`, Storage Lens. Cross-ref
  [[observability-multi-region]].
- `## Special-case buckets` — CloudTrail/access-log targets, Terraform state
  (cross-ref [[state-management]]; note the circular dependency if state lives in
  the region that just failed), CloudFront origin buckets with OAC (cross-ref
  [[aws-cloudfront]]), Lambda artifact buckets (cross-ref [[aws-lambda]]).
- Replica storage class as a cost lever — replicating to a cheaper class still
  meets RPO 2h. Quantify it.
- Template sections not yet written: Terraform implementation, Migration path,
  Failover procedure, Failback, Gotchas, Decisions to make, Cost, Open questions,
  and **Sources** (inline links exist but were never collected into a section).


