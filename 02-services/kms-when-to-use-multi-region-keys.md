---
title: KMS — When do you actually want a multi-Region key?
service: kms
tags: [service, multi-region, kms, decision-guide]
status: researched
replication: n/a — decision guide
rpo_achievable: "N/A"
rto_achievable: "N/A"
meets_targets: n/a
updated: 2026-09-16
---

# KMS — When do you actually want a multi-Region key?

> Companion to [[aws-kms]], which covers the mechanics. This note answers one
> question only: **for each thing we encrypt, do we need a multi-Region key
> (MRK), or will two ordinary single-region keys do?**

## TL;DR

- **For server-side encryption of AWS-managed resources, the answer is
  essentially always no.** AWS services re-encrypt at the region boundary as a
  matter of design. AWS says so in its own words: most services *"currently
  treat multi-Region keys as though they were single-Region keys."*
- **The right mental model:** an MRK is not "a key that works in two regions" —
  every region needs a key regardless. An MRK is **portable ciphertext**. Ask
  "do raw encrypted bytes cross the region boundary without an AWS service
  re-encrypting them?" If no, you don't need an MRK.
- **There is exactly one question that decides this for your estate:** do you
  do any *client-side* encryption (AWS Encryption SDK, DynamoDB/Database
  Encryption SDK, S3 client-side encryption), or do you sign anything with a
  KMS asymmetric key? If the answer is no to both, **you need zero MRKs**, and
  the whole feature is a distraction.
- Of the thirteen services below, **twelve are a clear "no"**. AWS Backup is the
  only AWS-managed one where an MRK is defensible, and even there it's a
  convenience, not a requirement.
- **The cost of a wrong "yes" is high and one-way.** `multi_region` is
  `ForceNew` and the AWS API has no conversion; a key you make multi-Region by
  mistake cannot be un-made without re-encrypting everything under it. Default
  to no. See the migration section of [[aws-kms]].

## The decision rule

Ask these in order. Stop at the first "yes".

1. **Do encrypted bytes cross the region boundary *without* an AWS service
   decrypting and re-encrypting them?** If yes → MRK.
   In practice this means: your application code produced the ciphertext, and
   your application code (or a copy job you wrote) moves it.
2. **Does key *identity* have to be stable across regions — i.e. does something
   outside AWS verify a signature or pin a public key?** If yes → MRK.
3. Otherwise → **two independent single-region keys, same alias name.**

Everything below is an application of those three questions.

## The verdict table

| # | Use case | MRK needed? | Why — precisely |
|---|---|---|---|
| 1 | **EBS volume encryption** | **No** | An EBS volume can only use a key in its own region. A snapshot copied cross-region is re-encrypted: `CopySnapshot` takes a `KmsKeyId` for the destination region and AWS re-wraps the volume's data key under it. The snapshot data key never needs to be readable by the source key in the destination. Give the standby its own key and name it in the destination launch template / `CopySnapshot` call. |
| 2 | **RDS / Aurora encryption, cross-region read replica** | **No — and this is the one people get most wrong** | `CreateDBInstanceReadReplica` into another region takes a `--kms-key-id` which AWS documents as *"The AWS KMS key identifier of the KMS key to use to encrypt the read replica in the destination AWS Region."* RDS creates a snapshot in the source region, performs a **cross-region snapshot copy** (which re-encrypts under the destination key), and loads the replica from it. The source key is used only in the source region; the destination key only in the destination. There is no point at which one key must read the other's ciphertext. An MRK would work, but would be doing nothing an ordinary destination-region CMK doesn't. See [[amazon-rds-postgres]]. |
| 3 | **DynamoDB Global Tables** | **No — each replica can and should use its own regional CMK** | Encryption is explicitly a **per-replica** setting. `CreateReplicationGroupMemberAction` / `UpdateReplicationGroupMemberAction` carry a `KMSMasterKeyId`, and the AWS docs describe multi-account global tables as having *"Distinct IAM, KMS, billing, CloudTrail, and governance per account"*, with same-account global tables sharing *"A single IAM and KMS boundary"* — but that is about the *account* boundary, not the key. Concretely: replication is an internal DynamoDB path; DynamoDB decrypts in the source region and encrypts under the destination replica's key. The service-linked role `AWSServiceRoleForDynamoDBReplication` needs KMS permissions **on the key in each region**. Terraform: `kms_key_arn` inside the `replica` block of `aws_dynamodb_table`, or on `aws_dynamodb_table_replica`. **Caveat:** `kms_key_arn` on `aws_dynamodb_table_replica` is `ForceNew` — changing it replaces the replica. See [[amazon-dynamodb]]. |
| 4 | **S3 SSE-KMS with Cross-Region Replication** | **No — and AWS explicitly says an MRK gains you nothing here** | The destination key is named in the replication rule's `Destination.EncryptionConfiguration.ReplicaKmsKeyID`, and the docs are blunt: *"The KMS key must have been created in the same AWS Region as the destination bucket."* Also: *"You must create two separate KMS keys… AWS KMS keys aren't shared outside the AWS Region in which they were created."* And the decisive quote: *"You can use multi-Region AWS KMS keys in Amazon S3. However, Amazon S3 currently treats multi-Region keys as though they were single-Region keys, and does not use the multi-Region features of the key."* The replication IAM role needs `kms:Decrypt` on the source key (with `kms:ViaService: s3.<source>.amazonaws.com`) and `kms:Encrypt` on the destination key (with `kms:ViaService: s3.<dest>.amazonaws.com`). See [[amazon-s3]]. |
| 5 | **Secrets Manager cross-region replicas** | **No** | Secrets Manager re-encrypts the replica under a key **in the replica region**. The console flow says *"(Optional) For Encryption key, choose a KMS key to encrypt the secret with. **The key must be in the replica Region.**"* If you don't specify one, the CLI docs state plainly: *"The replica is encrypted with the AWS managed key `aws/secretsmanager`"* — i.e. the replica region's own AWS managed key, which is a *different* key from the primary region's `aws/secretsmanager` (AWS managed keys are always single-Region). So the default behaviour already does the right thing with zero configuration. Specify a CMK per region only if you need to control the key policy. See [[aws-secrets-manager]]. |
| 6 | **SQS / SNS server-side encryption** | **No** | SSE protects messages at rest in one region's queue/topic. Neither service replicates messages cross-region natively — you stand up an independent queue/topic in the standby and point producers at it. There is no ciphertext crossing the boundary, so there is nothing an MRK could help with. See [[amazon-sqs]], [[amazon-sns]]. |
| 7 | **Lambda environment variable encryption** | **No** | Env vars are encrypted by Lambda against a key in the function's own region. Deploying the same function to the standby means a separate function resource with a separate `kms_key_arn` pointing at the standby key. The plaintext comes from your Terraform/pipeline, not from the primary region's ciphertext. See [[aws-lambda]]. |
| 8 | **SSM Parameter Store SecureString** | **No — but for a bad reason** | Parameter Store has **no native cross-region replication at all** (see [[aws-ssm-parameter-store]]). Whatever mechanism you build writes the *plaintext* value into the standby region and lets Parameter Store encrypt it there under the standby's key. Since you never move the ciphertext, an MRK is irrelevant. The one exception: if you build a replication Lambda that reads the raw encrypted blob rather than calling `GetParameter --with-decryption`… don't. Decrypt in the source, re-encrypt in the destination, and keep the plaintext in memory only. |
| 9 | **ECR** | **No** | ECR's encryption key is set at repository creation and **must exist in the repository's region** — the key is specified *"using the alias, key ID, or full ARN, and the key must exist in the same Region as the repository."* ECR cross-region replication creates an independent repository in the destination with its own encryption configuration. **Gotcha that has nothing to do with MRKs but will bite you:** for KMS-encrypted ECR replication the repository name must match exactly across regions. Also, encryption config is immutable — you cannot change a repo's key later. See [[amazon-ecr]]. |
| 10 | **CloudWatch Logs** | **No** | `AssociateKmsKey` on a log group requires a key in the same region. Log groups do not replicate cross-region natively; if you ship logs to the standby you are re-ingesting them there, under that region's key. See [[cloudwatch-observability]]. |
| 11 | **EFS** | **No** | Encryption key is fixed at file system creation and must be regional. EFS Replication creates a destination file system; the destination has its own KMS key (you choose it when creating the replication configuration). No portable ciphertext. See [[amazon-efs]]. |
| 12 | **AWS Backup cross-region copy** | **Defensible yes — the only AWS-managed case on this list** | See the detailed treatment below. Strictly, **not required**: *"A copy of a backup to another AWS Region is encrypted using the key of the destination vault."* But AWS Backup is where an MRK earns its keep operationally, for reasons that are real but are about consistency rather than capability. |
| 13 | **EKS secrets envelope encryption** | **No** | Set per cluster at creation, regional key. The standby cluster is a separate cluster with a separate key. Note it is effectively one-way: once `AssociateEncryptionConfig` is set you cannot remove it. See [[amazon-eks]]. |

### Where an MRK **is** the right answer

| Use case | Why an MRK is genuinely required |
|---|---|
| **Client-side encrypted data readable in both regions** | The AWS Encryption SDK, the AWS Database Encryption SDK (formerly DynamoDB Encryption Client) and S3 client-side encryption produce ciphertext that *your* code decrypts. If a `eu-west-2` pod must read a record your `eu-west-1` pod encrypted, and neither DynamoDB nor S3 knows the data is encrypted (because it's opaque bytes to them), then **nothing re-encrypts it at the boundary** and the `eu-west-2` key must be able to decrypt the `eu-west-1` ciphertext. That is exactly and only what an MRK does. AWS's own overview scopes MRKs to this case: *"when you need to encrypt or sign data in client-side applications across multiple Regions, multi-Region keys might be the solution."* **Note the nasty interaction: if you client-side encrypt items in a DynamoDB Global Table, the Global Table happily replicates the ciphertext — and the standby cannot read it.** That is the single most likely place this bites you, given DynamoDB Global Tables are already in progress. |
| **Asymmetric signing keys where key identity matters** | If you sign JWTs, artifacts, webhooks or licence files with a KMS `SIGN_VERIFY` key, the verifier holds a public key or a key ID. Failing over to a standby with a *different* keypair breaks every verifier that pinned the old one — and you cannot roll a pinned public key out to third parties in 15 minutes. An MRK produces *"identical digital signatures consistently and repeatedly in different AWS Regions."* AWS's docs add a genuinely useful caveat: if you have a proper CA chain with a single root and regional intermediates, you **don't** need an MRK — the chain already solves it. MRKs are for the case where the consumer can't handle intermediates, *"such as application signing."* |
| **HMAC keys used for tokens/signatures verified in either region** | Same argument. An HMAC tag generated in the primary must verify in the standby; nothing re-encrypts it. |
| **Data you copy between regions yourself** | Any home-grown replication — a Lambda that copies blobs, a batch job that ships files, a queue consumer that forwards payloads — where the payload is KMS ciphertext. If you wrote the copier, you own the re-encryption problem, and an MRK is the cheap way out. (The other way out: decrypt and re-encrypt in the copier. That's more code but keeps key isolation.) |
| **Encrypted data in a DynamoDB Global Table or S3 CRR that you encrypted before handing it to AWS** | The generalisation of the two rows above. The rule is: **AWS re-encrypts what it knows is encrypted. It cannot re-encrypt what looks like an opaque blob.** |

## AWS Backup — the one interesting case

This deserves more than a table row because the answer is genuinely "it depends
what you're backing up".

AWS Backup splits resource types in two:

- **Fully managed by AWS Backup** (S3, EFS, DynamoDB with Advanced Backup,
  Timestream, CloudFormation, SAP HANA, VMware): the backup is encrypted with
  **the backup vault's KMS key**, independently of how the source resource was
  encrypted. AWS calls this *"independent encryption"*.
- **Not fully managed** (EBS, EC2/AMI, RDS, Aurora, DocumentDB, Neptune,
  Redshift, FSx, Storage Gateway, plain DynamoDB): the backup *"inherits the
  encryption settings from their source resource"* — i.e. it is encrypted with
  the key that encrypted the source volume/database.

On cross-region copy, AWS Backup states: *"A copy of a backup to another AWS
Region is encrypted using the key of the destination vault."* So a copy **is**
re-encrypted, and an MRK is **not required**.

But then comes the constraint that makes MRKs attractive:

> "For a copy of a recovery point of a resource that is **not fully managed** by
> AWS Backup, the key associated to the destination vault must be a CMK or the
> managed key of the service that owns the underlying resource. For example, if
> you are copying an EC2 instance, a Backup managed key cannot be used. Instead,
> a CMK or Amazon EBS KMS key (`aws/ebs`) must be used to avoid copy job
> failure."

So for exactly the resource types you most want in a DR vault — EBS and RDS —
you must supply a real CMK in the destination region, and its key policy must
grant AWS Backup `kms:CreateGrant`, `kms:GenerateDataKey` and `kms:Decrypt`.
Now you have a backup chain (source vault key → destination vault key →
restored resource key) with three key policies across two regions that must all
be right, and a copy job that fails silently-ish if any of them isn't.

**The MRK argument here is consistency, not capability:** one key ID, one alias
resolving correctly in both regions, one mental model for the key policy, one
identity in CloudTrail across the whole backup chain, and a restore that works
identically whichever region you restore into. When you are restoring at 3am
with a 15-minute RTO, "the key is the same key" is worth something.

**The counter-argument:** your backup vault key is the key that protects
*everything*, including the backups you'd use to recover from a compromise.
It is the single worst key in your estate to widen the blast radius on. An
attacker with the replica key policy can decrypt every backup of both regions.

**Recommendation: single-region keys per vault, unless and until the operational
pain of managing the copy chain proves real.** Revisit in [[aws-backup]] once
the vault design exists. Whatever you choose, decide it *before* creating the
vaults — **a backup vault's encryption key cannot be changed after creation.**
That irreversibility is the reason this is the one row worth genuinely thinking
about rather than defaulting.

## Common false positives

Reasons people reach for MRKs that do not survive scrutiny:

| "We need an MRK because…" | Reality |
|---|---|
| "…the standby region needs a key." | Every region needs a key. That's an argument for *creating a key in the standby*, which is Option A. It is not an argument for the two keys being related. |
| "…we want the same alias in both regions." | Aliases are regional and independent. You can use `alias/prod-app-data` in both regions with two totally unrelated keys. Aliases have nothing to do with MRKs. (Several third-party blog posts cite "same alias in both regions" as an MRK benefit. It isn't one.) |
| "…otherwise data would be transmitted unencrypted between regions." | Flatly untrue for every AWS-managed replication path. AWS re-encrypts *inside* the service; nothing traverses the internet or the AWS backbone in plaintext. The [yobyot.com DLM post](https://yobyot.com/aws/aws-multi-region-keys-and-ec2-data-lifecycle-manager/2021/08/18/) makes this claim ("you had to decrypt it in the sending region, transmit it unencrypted and re-encrypt the data in the new region") and it is wrong — a useful example of a plausible-sounding blog that will mislead a reviewer. Cite the AWS docs, not this. |
| "…it's simpler in Terraform." | It isn't. `aws_kms_replica_key` + aliased provider + a second `aws_kms_alias` is the same line count as a second `aws_kms_key` + aliased provider + a second `aws_kms_alias`. And you still write the key policy twice, because policies are not a shared property. |
| "…rotation is easier — one key to rotate." | True but marginal, and it cuts both ways: rotation control lives only on the primary, so when the primary region is impaired, rotation is stuck. Two independent keys with rotation enabled on each are *more* resilient, not less. |
| "…cross-region S3 replication needs it." | AWS documents the exact opposite. See row 4. |
| "…we're going multi-region so everything should be multi-region." | The whole point of an active/passive pair is that the two sides are as independent as possible. Sharing key material is a *coupling*, and coupling is what you are trying to reduce. |

## What this means for the Helios estate

Concrete next actions, in order:

1. **Answer the deciding question.** Grep the application source for
   `aws-encryption-sdk`, `aws-database-encryption-sdk`,
   `aws-dynamodb-encryption`, `CryptoMaterialsManager`,
   `AmazonS3EncryptionClient`, and for direct `kms.encrypt` / `kms.sign` calls.
   If there are no hits, **no MRKs are required anywhere** and this note
   resolves to "single-region keys, done".
2. **Check the DynamoDB Global Tables work in flight.** If any table's items are
   client-side encrypted, that project needs an MRK *before* it goes
   multi-region, or the standby will replicate unreadable items. This is the
   highest-urgency item in the note because that migration is already underway.
   See [[amazon-dynamodb]].
3. **Inventory asymmetric keys.** `aws kms list-keys` + `describe-key`, filter
   `KeyUsage == SIGN_VERIFY`. Any of those that have external verifiers need an
   MRK, and need it planned as a key migration with a signature-verification
   compatibility window. See the migration section of [[aws-kms]].
4. **Decide the AWS Backup vault key model before creating vaults.** It is
   immutable after creation.
5. **Put the guardrail in.** An SCP denying `kms:CreateKey` with
   `kms:MultiRegion = true` except for an approved list, and pinning
   `kms:ReplicaRegion` to the designated DR partner, turns this note from
   guidance into policy.

## Open questions

- Is any data client-side encrypted today? (The whole note hinges on this.)
- Are there KMS asymmetric signing or HMAC keys with verifiers outside the
  region or outside AWS?
- Does the Canadian deployment's residency posture permit key material to exist
  in two regions at all? `ca-central-1` → `ca-west-1` is both-in-Canada so this
  is probably fine, but it needs a yes from whoever owns
  [[data-residency-and-compliance]], because AWS frames the no-conversion rule
  as a residency guarantee.
- For AWS Backup: are we backing up EBS and RDS (not-fully-managed, inherit
  source key, destination vault must be a CMK) or only S3/EFS/DynamoDB
  (fully-managed, independent encryption)? Changes the difficulty materially.

## Sources

- [Multi-Region keys in AWS KMS](https://docs.aws.amazon.com/kms/latest/developerguide/multi-region-keys-overview.html) — "Most AWS services… currently treat multi-Region keys as though they were single-Region keys", the S3 CRR example, the four scenarios AWS thinks MRKs are for (DR, global data management, distributed signing, active-active), and the CA-chain caveat on signing.
- [Replicating encrypted objects (S3)](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-config-for-kms-objects.html) — `ReplicaKmsKeyID` must be a key in the destination bucket's region; "You must create two separate KMS keys"; the explicit note that S3 does not use the multi-Region features of an MRK; the `kms:Decrypt`/`kms:Encrypt` split with per-region `kms:ViaService`; and the KMS throttling warning when enabling CRR on a large bucket.
- [Creating a read replica in a different AWS Region (RDS)](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.XRgn.html) — `--kms-key-id` is "the KMS key to use to encrypt the read replica in the destination AWS Region"; the snapshot-copy mechanism that makes an MRK unnecessary.
- [Global tables — multi-active, multi-Region replication (DynamoDB)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html) and [Global tables core concepts](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/globaltables-CoreConcepts.html) — per-replica configuration model; the same-account vs multi-account KMS boundary table.
- [`aws_dynamodb_table` / `aws_dynamodb_table_replica` provider docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/dynamodb_table_replica) — `kms_key_arn` per replica, and the `ForceNew` behaviour on the replica resource.
- [terraform-provider-aws issue #16358](https://github.com/hashicorp/terraform-provider-aws/issues/16358) and [#21726](https://github.com/hashicorp/terraform-provider-aws/issues/21726) — historical friction setting/changing a per-replica CMK on DynamoDB global tables. Worth reading before the DynamoDB migration lands.
- [Replicate AWS Secrets Manager secrets across Regions](https://docs.aws.amazon.com/secretsmanager/latest/userguide/create-manage-multi-region-secrets.html) — "The key must be in the replica Region", and the CLI note that without an explicit key "the replica is encrypted with the AWS managed key `aws/secretsmanager`".
- [Encryption for backups in AWS Backup](https://docs.aws.amazon.com/aws-backup/latest/devguide/encryption.html) — the fully-managed vs not-fully-managed table, "independent encryption", "A copy of a backup to another AWS Region is encrypted using the key of the destination vault", the CMK requirement for not-fully-managed cross-region copies, and the minimum key policy (`kms:CreateGrant`, `kms:GenerateDataKey`, `kms:Decrypt`).
- [Configuring KMS encryption at rest on ECR repositories with ECR replication](https://aws.amazon.com/blogs/containers/configuring-kms-encryption-at-rest-on-ecr-repositories-with-ecr-replication/) — AWS Containers blog; the key must be in the repository's region, and the repository name must match exactly across regions for KMS-encrypted replication to work.
- [aws-samples/amazon-ecr-kms-replication](https://github.com/aws-samples/amazon-ecr-kms-replication) — AWS sample CloudFormation for the ECR + KMS + replication combination, if you want a reference implementation.
- [AWS multi-region KMS keys and Data Lifecycle Manager: better together (yobyot.com)](https://yobyot.com/aws/aws-multi-region-keys-and-ec2-data-lifecycle-manager/2021/08/18/) — cited **as a counter-example**: it claims cross-region copies previously required transmitting data unencrypted, which is not correct, and it lists "same alias in both regions" as an MRK benefit when aliases are independent regional resources either way.
- [KMS Multi-Region Keys (What's New, June 2021)](https://aws.amazon.com/about-aws/whats-new/2021/06/kms-multi-region-keys) — the original launch announcement; useful for the framing AWS used at launch.
