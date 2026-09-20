---
title: Amazon Cognito — Multi-Region
service: cognito
tags: [service, multi-region, cognito, identity, auth]
status: researched
replication: native (multi-Region replication, GA 2026-06-04) — with material gaps
rpo_achievable: "near-real-time / eventually consistent for user directory; AWS publishes no lag SLA. Comfortably inside 2h."
rto_achievable: "< 5 min for the EU and US pairs if pre-provisioned. Not achievable at all for the CA pair — ca-west-1 is not an MRR Region."
meets_targets: conditional
updated: 2026-09-20
---

# Amazon Cognito — Multi-Region

> **Read the first section before anything else.** The premise this note was
> commissioned under — *"Cognito user pools have no native cross-region
> replication"* — was true for the entire history of the service up to
> **4 June 2026**. It is no longer true. AWS shipped multi-Region replication
> (MRR) for user pools on that date. Everything written about Cognito DR before
> mid-2026 (including most of the blog posts you will find on the first page of
> a search) describes a world that no longer exists.
>
> **But do not close the note there.** MRR is gated behind an infrastructure
> eligibility check, a feature-plan upgrade, a customer-managed KMS key, an
> issuer-URL change that breaks two other AWS integrations you almost certainly
> use, and a Region list that **does not include `ca-west-1`**. For the CA pair
> this note's answer is still "no".

## TL;DR

- **Cognito user pools now replicate natively.** `CreateUserPoolReplica` creates
  a read-mostly replica in a second Region that shares the *same user pool ID*,
  holds a synchronised copy of the user directory including credentials, and can
  serve `InitiateAuth` / token issuance during a failover. Announced
  [4 June 2026](https://aws.amazon.com/about-aws/whats-new/2026/06/amazon-cognito-multi-region/).
  Users are **not** forced to reset passwords at failover. That was the single
  biggest historical objection and it is gone.
- **`ca-west-1` (Calgary) is not on the MRR Region list.** Supported Regions are
  US East (Ohio, N. Virginia), US West (N. California, Oregon), Asia Pacific
  (Mumbai, Seoul, Singapore, Sydney, Tokyo), **Canada (Central)**, Europe
  (Frankfurt, Ireland, London, Paris, Stockholm), South America (São Paulo).
  Calgary is absent. `ca-central-1` can be a *primary* but has no eligible
  in-country partner. **The CA pair cannot use Cognito MRR at all** — see
  [[region-pair-selection]]. This is a headline finding.
- **The EU pair and US pair are both fully covered.** `eu-west-1` → `eu-west-2`
  and `us-east-1` → `us-west-2` are all four on the list. For those two pairs
  Cognito goes from "programme blocker" to "a two-week workstream with an
  awkward client-side change in the middle".
- **The awkward client-side change is real and it is the thing that will bite.**
  AWS recommends switching the user pool to the **updated issuer**
  (`https://issuer-cognito-idp.<region>.amazonaws.com/<poolId>`) so that JWTs
  validate in both Regions. That issuer type is **explicitly not compatible with
  ALB authentication-with-Cognito or API Gateway Cognito authorizers**. If the
  estate uses either — and with [[aws-alb-nlb]] and [[aws-api-gateway]] both in
  scope, it probably does — you must migrate those to a custom authorizer or a
  JWT authorizer before you can adopt the recommended issuer.
- **Terraform does not support it yet.** There is no `aws_cognito_user_pool_replica`
  resource. [hashicorp/terraform-provider-aws#48227](https://github.com/hashicorp/terraform-provider-aws/issues/48227)
  is open (filed 5 June 2026, no milestone, a linked PR in flight as of this
  research). CloudFormation shipped `AWS::Cognito::UserPoolReplica`. Until the
  provider catches up, the replica is a `null_resource`/`awscc` shim or a
  click-op — and that is a governance problem in a cookiecutter monorepo, not a
  technical one. See [[module-patterns]].

---

## Does this service cross regions at all?

Three separate answers, because Cognito is three separate things.

### 1. User pools — yes, since June 2026, conditionally

A user pool is a Regional resource with a Regional ID (`eu-west-1_AbCdEfGhI`).
Historically that ID was the end of the conversation: the pool existed in one
Region, its JWKS lived at a Regional URL, its app client IDs were Regional, and
there was no AWS-sanctioned way to get a second copy.

MRR changes the model in an unusual way. From the docs:

> "When you configure MRR, Amazon Cognito creates separate user pools with a
> shared user pool ID. Each replica user pool hosts authentication services for
> a shared user directory."

So the *pool ID does not change* in the secondary Region. The replica's ARN is
`arn:aws:cognito-idp:us-west-2:111122223333:userpool/us-east-1_EXAMPLE` — note
the Region element of the ARN is `us-west-2` while the pool ID embedded in the
resource path still says `us-east-1_`. That is deliberate and it is what makes
client-side failover tractable: **you change the endpoint Region, not the pool
ID or the app client ID.**

**Eligibility gate.** MRR requires what AWS calls the *next-generation Cognito
infrastructure* — a rebuilt storage layer that AWS has been migrating pools onto
transparently. The docs are blunt:

> "Multi-Region replication is not available for all user pools at this time.
> Multi-Region replication requires the modern Amazon Cognito infrastructure
> with enhanced capabilities and scalability. Some user pools are still on a
> previous infrastructure and will be upgraded by AWS to the new infrastructure,
> which will unlock this feature."

There is no published API to query eligibility and no published timeline. The
only stated detection method is the console: eligible pools show MRR
configuration options, ineligible pools show an exception message. **This is an
open question for the company** (see [Open questions](#open-questions)) and it
is the first thing to check, before any design work — if the production pools
are on the legacy infrastructure, none of the rest of this note is actionable
yet.

### 2. Identity pools — no

Identity pools (`cognito-identity`, the thing that exchanges a user pool token
for temporary IAM credentials) got nothing in the June 2026 launch. There is no
identity-pool replication. An identity pool ID is Regional
(`eu-west-1:a1b2c3d4-...`) and there is no mechanism to produce the same ID in
another Region.

The good news is that identity pools are almost entirely *configuration*, not
data. What lives in one is: the list of trusted providers, the authenticated and
unauthenticated IAM role ARNs, role-mapping rules, and the developer-provider
name. All of that is Terraform-managed and mirrors trivially — `aws_cognito_identity_pool`
and `aws_cognito_identity_pool_roles_attachment` against a second provider
alias, done.

The bad news is the **identity ID**. Each end user gets a stable identity ID
minted by the identity pool, and if you have used that as a partition key or a
foreign key anywhere — in [[aws-dynamodb]], in [[aws-s3]] prefixes, in an RDS
column — the standby pool will mint *different* identity IDs for the same users.
Cognito Sync / identity-ID datasets are not replicated. Search this estate for
`cognito-identity` ID usage as a durable key before assuming identity pools are
a non-issue. The `DeveloperProviderName` / `GetOpenIdTokenForDeveloperIdentity`
flow lets you pin identity IDs to your own user identifiers, which is the escape
hatch if you find you need one — but it is an application change, not a
Terraform change.

Cross-ref [[aws-iam]] for the role side: the authenticated role's trust policy
contains `"cognito-identity.amazonaws.com:aud": "<identity-pool-id>"`, which is
Regional, so the standby needs its own role or a trust policy with both audience
values. IAM roles are global, so a single role with a two-value `aud` condition
is the cleaner pattern.

### 3. Managed login / hosted UI domains — partially

A user pool domain (prefix or custom) is fronted by a service-owned CloudFront
distribution. Cognito now supports **domain-level failover**: you attach a
Route 53 health check ID to the domain and Cognito itself flips which Region
serves managed login when that check goes unhealthy.

```json
{
 "CustomDomainConfig": { "CertificateArn": "arn:aws:acm:us-east-1:111122223333:certificate/..." },
 "Domain": "auth.example.com",
 "ManagedLoginVersion": 2,
 "Routing": {
    "Failover": {
       "SecondaryRegion": "us-west-2",
       "PrimaryRoute53HealthCheckId": "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111"
    }
 },
 "UserPoolId": "us-east-1_EXAMPLE"
}
```

This is genuinely good: it means the browser-based sign-in path fails over
without any client change, driven by a health check you define. Cross-ref
[[aws-route53]] and [[failover-orchestration]] — this health check should be the
*same* health check that drives the rest of the failover, or you get a
split-brain where auth has moved Region and the API has not. See
[[split-brain-and-fencing]].

Constraint: **"You can't configure a different custom domain with replica user
pools."** One domain, both Regions. That is what you want, but it means you
cannot pre-test the standby on a separate hostname through the managed-login
path.

---

## What the old answer was, and why it still matters

Even with MRR shipped, you need to understand the pre-2026 position, for three
reasons: the CA pair still lives in it, pools not yet on the next-gen
infrastructure still live in it, and it is the fallback if MRR adoption stalls.

### What you lose in a Region failure with no replication

| Asset | Recoverable? | Notes |
|---|---|---|
| Usernames and standard attributes | Yes, via `ListUsers` export | Rate-limited; `UserList` category quota |
| Custom attributes | Yes | Must exist in the target pool's schema first |
| Groups and group membership | Yes, via `ListGroups` / `ListUsersInGroup` | |
| **Password hashes** | **No — cannot be exported from Cognito** | The core problem. See below. |
| TOTP MFA registrations | **No** | Secret is not exportable; users re-enrol |
| SMS/email MFA preference | Partially — the preference flag, not the verified state | |
| WebAuthn / passkey credentials | **No** | Bound to the RP ID *and* the pool |
| Device tracking / remembered devices | **No** | Every device becomes "new", which re-triggers MFA |
| Refresh tokens | **No** | All sessions die; every user re-authenticates |
| `sub` (the user's UUID) | **No** — a new pool mints new `sub` values | Catastrophic if `sub` is a foreign key |
| App client IDs | **No** — new pool, new client IDs | Baked into SPAs and mobile builds |
| App client secrets | **No** | Server-side, so at least redeployable |

Two of those rows are the ones that kill you.

**Password hashes cannot be exported from Cognito.** This deserves precision,
because the documentation has recently become confusing on the point. As of 2026
Cognito *can* **import** password hashes — `create-user-import-job
--password-hashing-algorithm BCRYPT|SCRYPT|ARGON2ID|PBKDF2_SHA256` with a
`password_hash` column in the CSV. That is for migrating *into* Cognito from a
system whose hashes you already hold. There is still **no API that emits a
Cognito user's password hash**. So for a Cognito-to-Cognito rebuild, the import
feature does not help you — you have nothing to put in the column.

Without hashes, CSV import lands every user in `RESET_REQUIRED`:

> "By default, the import process sets values for all user attributes except
> **password**. This means that your users must change their passwords the first
> time they sign in. Your users are in a `RESET_REQUIRED` state when imported
> using this method."

**Quantify what that means against RTO 15m.** It is not a 15-minute event; it is
not an event with a duration at all. Failing over into a password-reset pool
means:

- Every user's next sign-in fails, then requires an email or SMS code round-trip.
- That code goes out over SES or SNS **in the standby Region**, which has its own
  cold reputation and possibly its own sandbox — see [[aws-ses]]. A mass password
  reset is exactly the traffic shape that gets a cold SES identity throttled or
  spam-foldered. The two failure modes compound.
- `AccountRecovery` and `UserAuthentication` category quotas in the standby are
  separate from the primary's (below) and a synchronised reset of the whole user
  base is a thundering herd against both.
- Support volume: assume a double-digit percentage of users cannot complete the
  reset (dead email on file, changed phone number, corporate mail filtering the
  code) and become manual tickets.
- **`sub` changes.** If anything downstream keys on `sub`, a rebuilt pool is a
  data-integrity incident, not an availability incident.

So the honest pre-MRR verdict: **RTO for authentication was measured in days,
not minutes, and the recovery was partial and lossy.** That is why this was the
worst service in the estate.

### The user-migration Lambda trigger: the only sanctioned live path

The `UserMigration` trigger was — and for the CA pair remains — the only way to
move a live user base between pools without a forced reset. Mechanics:

1. Target pool has a `UserMigration` Lambda attached.
2. A user signs in to the *new* pool with a username Cognito doesn't recognise.
3. Cognito invokes the Lambda with `triggerSource: UserMigration_Authentication`,
   passing `userName` and **the plaintext password** in `request.password`.
4. The Lambda authenticates that credential against the *old* pool (an
   `AdminInitiateAuth` with `ADMIN_USER_PASSWORD_AUTH` cross-Region, or
   cross-account) and, on success, returns the user's attributes.
5. Cognito silently creates the user in the new pool with a password verifier
   derived from the plaintext it was handed. The user never sees a reset.

There is also `UserMigration_ForgotPassword`, which migrates a user who arrives
via the forgot-password flow instead.

**What it can carry:** username, standard and custom attributes, verified
email/phone flags, group membership (if the Lambda calls `AdminAddUserToGroup`
itself), `finalUserStatus` and `messageAction`, and — critically — the working
password.

**What it cannot carry:**

- **Federated users.** A user who only ever signed in via SAML/OIDC/social has no
  password for the trigger to validate. They are created fresh in the new pool on
  their next federated sign-in, with a new `sub`.
- **MFA state.** TOTP secrets and WebAuthn credentials do not transit. Users
  re-enrol.
- **Device tracking.**
- **`sub`.** The new pool mints a new one. There is no API to set it.
- **Anyone who does not sign in.** Migration is lazy by construction. Dormant
  users are never migrated. If the old pool is deleted, they are gone.

**The flow-type constraint that catches everyone:** the trigger only fires for
flows where Cognito sees the plaintext password. You must enable
`USER_PASSWORD_AUTH` or `ADMIN_USER_PASSWORD_AUTH` on the app client. The default
and recommended `USER_SRP_AUTH` never hands Cognito a plaintext password, so the
trigger cannot fire. **Turning on `USER_PASSWORD_AUTH` to enable migration is a
real, if temporary, security posture downgrade** and must be time-boxed and
signed off. Revert to SRP-only once the migration window closes.

**Why this is a migration tool and not a DR tool.** The trigger requires the
*source* pool to be reachable to validate the password. In the failover scenario
the source pool's Region is the one that is down. A migration Lambda pointed at a
dead Region returns errors for every user. It is an excellent tool for a planned
pool-to-pool move and useless as a disaster response. Anyone proposing it as the
DR answer has not thought it through.

### The AWS-published DIY pattern

AWS maintains [Guidance for User Profiles Export with Amazon Cognito](https://docs.aws.amazon.com/solutions/latest/cognito-user-profiles-export-reference-architecture/overview.html)
(formerly the *Cognito User Profiles Export Reference Architecture*, now moved to
[aws-samples](https://github.com/aws-samples/sample-cognito-user-profiles-export-reference-architecture)).
A Step Functions `ExportWorkflow` runs on a schedule — daily, weekly, or every 30
days — and writes user profiles, groups and memberships to a **DynamoDB global
table** replicated to the backup Region. An `ImportWorkflow` can rehydrate an
empty pool from that table in either Region.

Be clear-eyed about it: AWS's own documentation states the solution does not
export federated users, and that "some data loss will result" because it runs
periodically. And it carries no passwords — this is a profile export, not a
credential export. **A daily schedule is an RPO of 24 hours against a target of
2 hours.** You could run it hourly and blow through `UserList` quota; you still
would not have passwords. It is a *data-preservation* tool (protects you from
"someone deleted the pool"), not a failover tool. Worth running regardless of
which branch you choose, for exactly that reason. Cross-ref [[aws-dynamodb]]
since it lands on a global table you already have the pattern for.

---

## Replication / mirroring options

### Option A — Native MRR (recommended where available)

Architecture: one primary pool, one secondary replica, shared pool ID, shared
app client IDs, shared custom domain, Route 53 health check driving failover.

**What replicates, per AWS:** "user profiles, credentials, and pool
configurations", plus "machine secrets" (M2M app client credentials). App client
IDs and secrets replicate, with **eventual consistency**.

**What does not, and must be built separately in the standby:**

| Thing | Why | Cross-ref |
|---|---|---|
| Lambda triggers | Lambda is Regional. Replica has its own trigger config. | [[aws-lambda]] |
| Email configuration (SES) | SES identity is Regional. | [[aws-ses]] |
| SMS configuration (SNS) | SNS + spending limit are Regional. | |
| WAF web ACL | Regional. | |
| Log export configuration | Regional log group. | |
| Tags | Set independently per replica. | |
| Threat-protection notification email config | Regional. | |

Those seven are precisely the settings the docs list as independently
configurable per replica. Everything else is set on the primary and synced.

**Hard limitations you must design around:**

1. **"You can't generate new users in secondary user pools, either by sign-up or
   by administrator creation."** During failover, sign-up is dead. Your app must
   detect the failover state and disable the registration flow, or users hit raw
   API errors.
2. **"Users can't reset their passwords or modify their profiles in secondary
   user pools."** Same — disable forgot-password and profile-edit UI during
   failover. AWS says so explicitly: "In a failover state, disable these
   operations in the user interface."
3. **"TOTP MFA is not supported in secondary replicas. Users with TOTP MFA
   configured must authenticate when the user pool in the primary Region is
   servicing requests."** Read that twice. **Any user with TOTP MFA cannot sign
   in at all during a failover.** If TOTP is mandatory for admin or privileged
   users — a very common configuration — then your admins are locked out during
   the exact incident where you need them. Plan an SMS or email OTP fallback
   factor for those users, or accept that privileged access runs through a
   break-glass IAM path instead. This is the most operationally dangerous line in
   the MRR documentation.
4. **Federated users must have signed in to the primary at least once.** "Federated
   users can only sign in to a secondary user pool in the failover state if they
   have previously signed in to the primary user pool." A brand-new SSO user
   during a failover cannot get in.
5. **One replica only.** "You can have at most one secondary replica in an
   additional Region per user directory." Fine for an active/passive pair; rules
   out a three-Region fan-out.
6. **Lockout counters are not synchronised.** "The count of password-based
   authentication attempts before lockout isn't synchronized across Regions."
   Minor, but it is a genuine security-control weakening: an attacker who can
   reach both Regional endpoints gets two independent lockout budgets.

**Prerequisites, in order:**

1. Pool is on the next-gen infrastructure (unverifiable except via console).
2. Pool is on the **Essentials** or **Plus** feature plan. Lite pools cannot
   enable MRR. If the estate is on Lite, this is a price increase from $0.0055 to
   $0.015 per MAU *before* the MRR add-on.
3. Pool is encrypted with a **multi-Region customer-managed KMS key**, with the
   MRK replica present in the secondary Region. Cognito only accepts symmetric
   keys in the same Region as the pool, by ARN not alias — so the primary points
   at the MRK primary and the replica points at the MRK replica. This is exactly
   the use case [[kms-when-to-use-multi-region-keys]] exists for; note it is one
   of the rare cases where an MRK is *mandated*, not merely convenient. Key policy
   must trust `cognito-idp.amazonaws.com` and `identitystore.amazonaws.com` with
   `Encrypt`, `Decrypt`, `ReEncrypt*`, `GenerateDataKeyWithoutPlainText` and
   `DescribeKey`, scoped by the `aws:cognito-idp:userpool-arn` encryption context.
   Cross-ref [[aws-kms]].
4. **Issuer type switched to "updated"** (see below) — technically optional, but
   without it your standby-issued tokens carry a different issuer claim than your
   primary-issued ones and every validator needs to accept both.

### Option B — Dual-write to two independent pools (active/active auth)

Two real pools, two pool IDs, two sets of app client IDs. Every sign-up, profile
change and password change is written to both by application code (or by a
`PostConfirmation` / `PreTokenGeneration` Lambda fan-out).

Why anyone would do this: it is the only option for the **CA pair**, and it is
the only option for pools stuck on legacy infrastructure.

Why it is unpleasant:

- **Password changes are the killer.** A `ForgotPassword`/`ConfirmForgotPassword`
  in pool A gives your code no plaintext to replay into pool B. You can hook
  `CustomMessage` or use `AdminSetUserPassword` from a Lambda that *does* see the
  plaintext — which means intercepting the password-change flow in your own
  backend rather than using Cognito's native one. You have now written half an
  IdP.
- Two `sub` values per user. You must maintain your own mapping, or key
  everything on `email`/`username` instead, which has its own uniqueness and
  mutability problems.
- Two JWKS, two issuers, two audiences. Every validator accepts both.
- Divergence is silent and permanent. There is no reconciliation primitive. You
  will build one, and it will be the thing that pages you.
- No reduction in MAU billing — you pay MAU on both pools, at full rate, which is
  *more* expensive than the MRR add-on ($0.015 + $0.015 = $0.030/MAU on
  Essentials, versus $0.015 + $0.0045 = $0.0195/MAU with MRR).

**Genuinely not recommended** unless forced. Documented here because for
`ca-central-1` → `ca-west-1` you *are* forced, and because it is the pattern the
pre-2026 blog posts advocate.

### Option C — Third-party IdP

Move authentication to Auth0/Okta, Microsoft Entra ID, or self-hosted Keycloak,
and keep Cognito (if at all) as a thin federation shim.

- **Auth0/Okta:** you inherit *their* multi-region story, which is largely
  opaque, tenant-region-pinned, and not something you can fail over on your own
  timetable. You are trading a problem you can see for a problem you cannot. For
  a Canadian data-residency requirement in particular, verify the vendor's
  Canadian region posture directly — do not assume.
- **Keycloak on [[aws-eks]] with [[aws-aurora-global-database]]:** this *does*
  give you a real, self-controlled active/passive auth tier with an Aurora
  Global Database RPO typically ~1s and a managed-failover RTO in the low
  minutes. It is also a whole new production system to operate, patch and
  security-review, and it puts your IdP's availability underneath your EKS
  cluster's availability — which is a dependency inversion worth staring at.
- Migration cost in either direction is the same user-migration-Lambda /
  password-reset problem described above, plus every app client integration.

Honest assessment: **for the EU and US pairs this is a wildly disproportionate
response to a problem AWS just solved.** For the CA pair it is a serious
contender, because the alternative is dual-write or a long auth RTO.

### Option D — Accept a long RTO for auth specifically

Deliberately scope authentication out of the 15-minute target. Failover brings up
the standby *application*, but sign-in remains unavailable (or read-only for
already-authenticated sessions) until the primary recovers or until a rebuild
completes.

This is more defensible than it sounds if — and only if — access tokens are
long-lived enough that already-signed-in users keep working. Cognito access and
ID tokens default to 60 minutes and can be configured from 5 minutes to 24
hours; refresh tokens from 60 minutes to 10 years. A 24-hour access token means a
Regional auth outage is invisible to anyone already signed in, at the cost of
being unable to revoke that token for 24 hours. That is a security trade-off the
security team owns, not the DR team. Note also that under MRR, refresh tokens
*are* interoperable between Regions, so this lever is less necessary when MRR is
available.

---

## The issuer problem — read this before designing anything

Cognito user pools offer two issuer types.

**Original issuer** (today's default for existing pools):

```
https://cognito-idp.eu-west-1.amazonaws.com/eu-west-1_AbCdEfGhI
```

Region-pinned. Both the discovery document
(`/.well-known/openid-configuration`) and the JWKS (`/.well-known/jwks.json`)
are served from that Regional host. If `eu-west-1` is unreachable, **your token
validators cannot fetch the JWKS** — so even tokens minted in London fail
validation if your validator has a cold JWKS cache. This is a subtle and
vicious failure mode: auth *issuance* has failed over, auth *verification* has
not.

**Updated issuer** (recommended by AWS for MRR):

```
https://issuer-cognito-idp.eu-west-1.amazonaws.com/eu-west-1_AbCdEfGhI
```

Note it still contains the *primary* Region string — it is not region-agnostic in
appearance. What changes is the backing: per AWS, "Updated issuers host the same
JWKS content in multiple Regions, resulting in improved resilience and
efficiency." The hostname is served multi-Region, so JWKS retrieval survives a
Regional failure. Configured via the `IssuerConfiguration` field on
`UpdateUserPool` (`IssuerConfigurationType` in the API reference).

**The trap.** Straight from the docs:

> "The updated issuer type is not currently compatible with Application Load
> Balancer authentication with Amazon Cognito or Amazon API Gateway Amazon
> Cognito authorizers."

That is two first-class AWS integrations, both of which are in scope for this
programme:

- **ALB `authenticate-cognito` listener rules** — see [[aws-alb-nlb]]. If any ALB
  offloads OIDC to Cognito, switching the issuer breaks it. Replacement: move
  authentication into the application, or front with a Lambda@Edge/CloudFront
  Function, or keep the original issuer and accept the JWKS-availability risk.
- **API Gateway `COGNITO_USER_POOLS` authorizers** — see [[aws-api-gateway]].
  Replacement: an HTTP API **JWT authorizer** configured with the updated issuer
  URL (JWT authorizers take an arbitrary issuer and audience, so they work), or a
  Lambda `REQUEST`/`TOKEN` authorizer that validates the JWT itself. For REST
  APIs there is no JWT authorizer type, so it is a Lambda authorizer or nothing.

**This must be discovered before the Cognito workstream starts, not during it.**
Grep the Terraform estate for `authenticate-cognito`, `aws_lb_listener_rule` with
`authenticate_cognito`, and `aws_api_gateway_authorizer` with
`type = "COGNITO_USER_POOLS"`. The size of that result set determines whether the
Cognito workstream is two weeks or two quarters.

**If you keep the original issuer with MRR:** MRR still works. Both Regions still
issue tokens. But you carry the JWKS-availability risk, and validators must
tolerate tokens whose `iss` claim points at a host that may be down. Mitigate by
pre-warming and long-caching the JWKS in every validator (keys rotate rarely;
a 24-hour cache with stale-if-error is defensible), and by baking the JWKS into
the standby deployment as a static fallback. That is a real, workable middle
path and it should be on the table.

---

## RPO / RTO analysis

### RPO

AWS describes MRR replication as **"near real-time"** in the launch post and
**"eventually consistent"** in the developer guide, and publishes **no lag
metric and no SLA**. That is a gap you should note as a risk rather than paper
over.

Against a 2-hour RPO this is almost certainly fine with enormous margin — the
observed behaviour of a replication pipeline described as near-real-time is
seconds. But "almost certainly" is not "measured". **Measure it in a game day**:
create a user in the primary, poll `AdminGetUser` against the secondary
endpoint, record the delta. Do the same for an attribute update and a password
change. Put the numbers in this note.

What is actually at risk within the lag window: users who signed up, changed a
password, or changed an attribute in the seconds before the Region failed. Since
the secondary cannot accept sign-ups anyway, a lost in-flight sign-up is
indistinguishable to the user from "sign-up was unavailable". Low impact.

For **Option B (dual-write)** RPO is whatever your fan-out code guarantees —
which, if the fan-out is synchronous, is zero, and if it is queued via
[[aws-sqs]], is queue depth. For **Option D**, RPO is not the binding constraint;
RTO is.

### RTO

| Step | Pre-provisioned? | Time |
|---|---|---|
| Detect | Route 53 health check, 3 failed checks @ 30s | ~90s |
| Cognito flips managed-login serving Region | Automatic, health-check driven | seconds |
| Replica already `ACTIVE`? | **Must be, in advance** | 0 if yes |
| Client SDK repoints to standby Region endpoint | **Application logic, must be pre-built** | 0–∞ (see below) |
| Lambda triggers present in standby | **Must be pre-deployed** | 0 if yes |
| SES/SNS configured in standby | **Must be pre-verified** | 0 if yes |
| Standby quota sufficient | **Check in advance; provisioned limits are instant if not** | 0–2 min |
| **Total (MRR, well-prepared)** | | **~2–5 min** |

**MRR meets RTO 15m for the EU and US pairs — comfortably — provided four
things are true in advance:**

1. The replica is in `ACTIVE` status, not `INACTIVE`. New replicas start
   `INACTIVE` and an `INACTIVE` replica serves *no* authentication operations.
   Flipping to `ACTIVE` is an `UpdateUserPoolReplica` call and is fast, but if you
   leave it for failover time you have added an API call and a human decision to
   the critical path. **Set it `ACTIVE` and leave it `ACTIVE`.** An `ACTIVE`
   replica does not take traffic on its own; traffic direction is the health
   check's job.
2. Client-side routing exists. This is the one that actually determines your RTO
   and it is discussed in its own section below.
3. Lambda triggers are deployed and attached in the standby. A missing
   `PreTokenGeneration` trigger means tokens come out with the wrong claims and
   your authorization silently breaks — worse than an outage.
4. Standby quotas are adequate. See below.

**The CA pair fails RTO 15m for authentication outright.** With no MRR and no
dual-write, recovery means rebuilding a pool and mass-resetting passwords: days.
With dual-write, it can meet RTO, at the engineering cost described in Option B.

### The client-side routing problem — the honest bit

For the managed-login/hosted-UI path, Cognito's own domain failover handles it.
Nothing to do.

For anything calling the Cognito API directly — `InitiateAuth` from a mobile app,
Amplify in an SPA, a backend doing `AdminInitiateAuth` — **the client chooses the
Regional endpoint and AWS does not choose it for you.** From the docs:

> "If you use the Amazon Cognito APIs or SDKs, there's no usage of a custom
> domain and your application is responsible for routing traffic to the Amazon
> Cognito service regional endpoint."

The brief's framing — *"a client-side change is not a 15-minute operation"* — is
exactly right and it is the crux. If your failover plan is "ship a new mobile
build with `region: 'eu-west-2'`", your auth RTO is **7–14 days** (App Store
review) plus however long it takes users to update, and a meaningful tail never
updates at all. That is not an RTO, it is a wish.

MRR removes the *pool ID* and *client ID* from this problem — those are shared —
but it does not remove the *endpoint Region*. So the mitigation is architectural
and must be built before the incident:

**Pattern 1 — remote config (recommended for SPA and mobile).** The client fetches
its Cognito Region from a config endpoint at launch, served from
[[aws-cloudfront]] over an [[aws-s3]] origin or a tiny [[aws-api-gateway]] +
[[aws-lambda]]. Failover = change one JSON value. Cache TTL becomes your RTO
contribution: a 60-second TTL is the right order. Cost: one extra network
round-trip on cold start, and a hard dependency on that config endpoint being the
most available thing you own. Make it a CloudFront distribution with two origins.

**Pattern 2 — backend proxy.** The client never talks to Cognito directly; it
talks to your API, which does `AdminInitiateAuth` server-side. Now the routing
decision lives in server code you can redeploy in minutes. AWS's own security
blog suggests exactly this: "Consider a serverless application backend to help
determine which Region authentication with Amazon Cognito should begin against."
Cost: your API is now on the auth critical path and must itself be multi-Region,
and you lose SRP (the proxy needs `ADMIN_USER_PASSWORD_AUTH`, i.e. it handles
plaintext passwords) unless you proxy the SRP challenge/response round-trips too,
which is fiddly but doable.

**Pattern 3 — client-side try/fallback.** Client attempts primary, and on
connection error or 5xx retries against the secondary Region with the same pool
ID and client ID. Works because MRR shares those IDs. Cost: doubles latency on
every failure, and needs care not to hammer a degraded-but-alive primary. Best as
a *belt-and-braces* addition to pattern 1, not as the only mechanism.

**Recommendation: pattern 1 for public clients, pattern 2 where a backend already
sits on the path, pattern 3 as a safety net on top of pattern 1.** Whichever you
pick, the work is application work, it is on the critical path for the auth
RTO, and it should start before the Terraform work does. Flag it in
[[failover-orchestration]].

### Quotas are per-Region and do not follow you

From the quotas page, verbatim:

> "AWS can only grant a quota increase request in one Region at a time. A
> successful quota increase in US East (N. Virginia) has no effect on your
> maximum request rate in Asia Pacific (Tokyo)."

Defaults: `UserAuthentication` **120 RPS**, `UserCreation` **50 RPS**.
`RespondToAuthChallenge`/`AdminRespondToAuthChallenge` get 3× the
`UserAuthentication` category limit. If the primary has been raised to, say, 500
RPS over the years, the standby is sitting at 120 and **will throttle the moment
it takes production load** — and a failover is precisely when every client
retries at once, so the standby sees *more* than steady-state primary load.

Good news, and it is recent: **provisioned limits**, launched
[6 July 2026](https://aws.amazon.com/blogs/security/from-2-weeks-to-2-minutes-amazon-cognito-launches-provisioned-limits-for-self-service-rate-limit-management/),
let you set the enforced rate yourself via `UpdateProvisionedLimit`, per Region
per account, **effective immediately**, anywhere between the default and your
Service Quotas account-level max. You are billed for provisioned capacity above
the default regardless of use.

That gives you a genuine cost lever that fits the warm-standby posture:

- Raise the **Service Quotas account-level max** in the standby Region to match
  the primary **now** (this is the slow part — mostly auto-approved in minutes,
  but can require a support ticket).
- Leave the **provisioned limit** at the default 120 RPS while idle, so you pay
  nothing extra.
- At failover, `UpdateProvisionedLimit` to production levels — immediate, and one
  more step in the runbook.

Or, if you would rather not have an API call on the critical path, provision it
permanently and pay. **Recommendation: raise the account-level max in the standby
now, script the `UpdateProvisionedLimit` into the failover runbook, and test that
it takes effect during a game day.** The account-level max increase is the bit
that fails RTO if you leave it to the incident; the provisioned-limit bump is
instant and safe to automate.

Also flagged by AWS: raising Cognito quotas may require raising SNS and SES
quotas too, or MFA codes and password-reset emails fail. Cross-ref [[aws-ses]] —
this is the same compounding failure described earlier.

---

## Warm standby shape

With MRR, in the standby Region while the primary is healthy:

| Component | State | Idle cost |
|---|---|---|
| Replica user pool | `ACTIVE`, receiving replication, serving no traffic | MRR add-on per MAU |
| KMS MRK replica | Live | ~$1/mo + requests ([[aws-kms]]) |
| Lambda triggers | Deployed, zero invocations | ~$0 ([[aws-lambda]]) |
| SES identity + DKIM | Verified, out of sandbox, warm | ~$0 ([[aws-ses]]) |
| SNS SMS spending limit | Raised | $0 |
| WAF web ACL | Attached to replica | ~$5/mo + rules |
| Route 53 health check | Monitoring primary | ~$0.50–$1/mo ([[aws-route53]]) |
| Service Quotas account max | Raised to primary levels | $0 |
| Provisioned limit | Left at default | $0 |
| Identity pool (if used) | Deployed via Terraform, idle | $0 |

Nothing here is scaled-to-zero in the EC2 sense — Cognito has no capacity to
scale. The idle cost is almost entirely the MRR MAU add-on, which is
proportional to your *whole* user base, not to standby traffic. That is the
uncomfortable part of the cost model: **you pay for the replica on every monthly
active user, every month, forever, to insure against an event that may never
happen.** Size it before you commit — the arithmetic is in [Cost](#cost).

---

## Terraform implementation

### The provider gap, stated plainly

As of this research there is **no `aws_cognito_user_pool_replica` resource** and
no MRR-related arguments on `aws_cognito_user_pool`.
[Issue #48227](https://github.com/hashicorp/terraform-provider-aws/issues/48227)
was opened 5 June 2026 and is open with no milestone; a PR (#48670) is linked and
in flight. CloudFormation has `AWS::Cognito::UserPoolReplica`.

Note also that `kms_key_id` on `aws_cognito_user_pool` is **not** the at-rest
encryption key — it is the key used to encrypt codes sent to `CustomEmailSender`
/ `CustomSMSSender` triggers. The at-rest CMK is set via the `KeyConfiguration`
field on `CreateUserPool`/`UpdateUserPool`, and provider coverage for it should
be verified against the version you are pinned to before you plan the work. The
API allows `KeyConfiguration` on `UpdateUserPool`, so **switching a live pool
from an AWS-owned key to a CMK is an in-place update at the API level, not a
replacement** — which is the good news for migration (below).

Three ways to bridge the gap, in order of preference:

**1. `awscc` provider for the replica only.** The AWS Cloud Control provider
(`hashicorp/awscc`) exposes CloudFormation resource types generically, so
`awscc_cognito_user_pool_replica` should be available shortly after the
CloudFormation type ships if not already. This keeps the replica in Terraform
state and in the plan, which is what matters for a cookiecutter monorepo.
**Verify availability against your pinned `awscc` version** — this note has not
confirmed the resource exists today.

**2. A CloudFormation stack wrapping the CFN resource.** `aws_cloudformation_stack`
with a three-line template containing `AWS::Cognito::UserPoolReplica`. Ugly, but
it is declarative, idempotent, shows up in plan, and deletes cleanly. This is the
pragmatic choice if `awscc` does not yet cover it.

**3. `null_resource` + `local-exec` with the AWS CLI.** Last resort. No drift
detection, no clean destroy, breaks on any runner without credentials. If you do
this, at least add a `data` source or a `check` block that asserts the replica
exists and is `ACTIVE`, so drift is visible.

**Recommendation: option 2 now, migrate to the native resource when it ships.**
Keep it isolated in one small module so the swap is a one-file change. And add a
tracking item to revisit [#48227](https://github.com/hashicorp/terraform-provider-aws/issues/48227).

### Module shape for the monorepo

The natural fit for a cookiecutter-templated multi-env monorepo is a single
`cognito` module that takes both provider aliases and owns the whole pair,
mirroring the pattern in [[provider-aliases-vs-separate-stacks]]. Cognito is a
good candidate for the aliased-single-stack branch specifically because the
replica genuinely *cannot* exist without the primary — there is no meaningful
"standby-only" apply, so the usual argument for separate stacks (blast radius,
independent apply during a primary outage) is weaker here.

```hcl
# modules/cognito-user-pool/versions.tf
terraform {
  required_version = ">= 1.9"
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = ">= 5.70, < 7.0"
      configuration_aliases = [aws.primary, aws.standby, aws.us_east_1]
    }
  }
}
```

Three aliases, because the custom-domain certificate must live in `us-east-1`
regardless of where the pool is. Same constraint as [[aws-cloudfront]]; see
[[aws-acm]].

```hcl
# modules/cognito-user-pool/variables.tf

variable "name" {
  description = "Base name, e.g. \"helios\". Env suffix is applied by the caller."
  type        = string
}

variable "env" {
  description = "Environment slug from the cookiecutter context (dev/stg/prd)."
  type        = string
}

variable "primary_region" { type = string }
variable "standby_region" { type = string }

variable "enable_replication" {
  description = <<-EOT
    Create and activate a secondary replica. Must be false where the standby
    Region is not on the Cognito MRR Region list -- notably ca-west-1.
    See 02-services/aws-cognito.md.
  EOT
  type        = bool
  default     = false
}

variable "feature_plan" {
  description = "LITE | ESSENTIALS | PLUS. MRR requires ESSENTIALS or PLUS."
  type        = string
  default     = "ESSENTIALS"
  validation {
    condition     = contains(["LITE", "ESSENTIALS", "PLUS"], var.feature_plan)
    error_message = "feature_plan must be LITE, ESSENTIALS or PLUS."
  }
}

variable "kms_multi_region_key_arn" {
  description = <<-EOT
    ARN of the MRK *primary* in var.primary_region. The module derives the
    replica ARN. Required when enable_replication = true.
  EOT
  type        = string
  default     = null
}

variable "issuer_type" {
  description = <<-EOT
    ORIGINAL | UPDATED. UPDATED is required for cross-Region JWKS resilience but
    is incompatible with ALB authenticate-cognito and API Gateway
    COGNITO_USER_POOLS authorizers. Audit those before flipping.
  EOT
  type        = string
  default     = "ORIGINAL"
}

variable "custom_domain" {
  description = "e.g. auth.example.com. null for prefix domain only."
  type        = string
  default     = null
}

variable "primary_health_check_id" {
  description = "Route 53 health check driving managed-login failover."
  type        = string
  default     = null
}

variable "lambda_triggers" {
  description = <<-EOT
    Map of trigger name -> { primary_arn, standby_arn }. Both ARNs are required
    when enable_replication = true: Lambda triggers do NOT replicate and are
    configured per replica.
  EOT
  type = map(object({
    primary_arn = string
    standby_arn = string
  }))
  default = {}
}

variable "ses_from_arn" {
  description = "Map of region -> verified SES identity ARN. SES config is per-replica."
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/cognito-user-pool/main.tf

locals {
  pool_name = "${var.name}-${var.env}"

  # MRR Region list as published 2026-06. Verify before each use --
  # AWS adds Regions to this list over time.
  mrr_regions = [
    "us-east-1", "us-east-2", "us-west-1", "us-west-2",
    "ap-south-1", "ap-northeast-2", "ap-southeast-1", "ap-southeast-2", "ap-northeast-1",
    "ca-central-1",
    "eu-central-1", "eu-west-1", "eu-west-2", "eu-west-3", "eu-north-1",
    "sa-east-1",
  ]
}

# Fail the plan loudly rather than the apply obscurely.
resource "terraform_data" "mrr_region_guard" {
  count = var.enable_replication ? 1 : 0

  lifecycle {
    precondition {
      condition     = contains(local.mrr_regions, var.standby_region)
      error_message = <<-EOT
        ${var.standby_region} is not on the Cognito multi-Region replication
        Region list. ca-west-1 (Calgary) in particular is NOT supported.
        Set enable_replication = false and use the dual-write or long-RTO
        branch documented in 02-services/aws-cognito.md.
      EOT
    }
    precondition {
      condition     = var.feature_plan != "LITE"
      error_message = "Cognito MRR requires the ESSENTIALS or PLUS feature plan."
    }
    precondition {
      condition     = var.kms_multi_region_key_arn != null
      error_message = "Cognito MRR requires a multi-Region customer-managed KMS key."
    }
  }
}

resource "aws_cognito_user_pool" "this" {
  provider = aws.primary
  name     = local.pool_name

  user_pool_tier = var.feature_plan

  # NOTE: at-rest CMK (KeyConfiguration) coverage varies by provider version.
  # If your pinned version lacks it, set it once out-of-band -- UpdateUserPool
  # accepts KeyConfiguration in place, so this is NOT a destroy/recreate.

  dynamic "lambda_config" {
    for_each = length(var.lambda_triggers) > 0 ? [1] : []
    content {
      pre_token_generation = try(var.lambda_triggers["pre_token_generation"].primary_arn, null)
      post_confirmation    = try(var.lambda_triggers["post_confirmation"].primary_arn, null)
      pre_sign_up          = try(var.lambda_triggers["pre_sign_up"].primary_arn, null)
      # user_migration deliberately omitted: see the migration section.
    }
  }

  lifecycle {
    # A user pool is an irreplaceable data store. Never let a plan destroy it.
    prevent_destroy = true

    # Replication and out-of-band console settings will drift. Ignore the
    # fields you have deliberately chosen not to manage here.
    ignore_changes = [
      schema, # adding a custom attribute is one-way; see Gotchas
    ]
  }
}

resource "aws_cognito_user_pool_client" "app" {
  provider     = aws.primary
  name         = "${local.pool_name}-app"
  user_pool_id = aws_cognito_user_pool.this.id

  # Under MRR this client ID is valid in BOTH Regions. Do not create a
  # second client in the standby.
  explicit_auth_flows = [
    "ALLOW_USER_SRP_AUTH",
    "ALLOW_REFRESH_TOKEN_AUTH",
    # ALLOW_USER_PASSWORD_AUTH: enable ONLY during a user-migration window.
  ]

  access_token_validity  = 60
  id_token_validity      = 60
  refresh_token_validity = 30
  token_validity_units {
    access_token  = "minutes"
    id_token      = "minutes"
    refresh_token = "days"
  }
}

# ---------------------------------------------------------------------------
# The replica. No native resource yet (#48227). CloudFormation shim.
# Swap for aws_cognito_user_pool_replica when it lands.
# ---------------------------------------------------------------------------
resource "aws_cloudformation_stack" "replica" {
  count    = var.enable_replication ? 1 : 0
  provider = aws.primary
  name     = "${local.pool_name}-cognito-replica"

  template_body = jsonencode({
    AWSTemplateFormatVersion = "2010-09-09"
    Resources = {
      Replica = {
        Type = "AWS::Cognito::UserPoolReplica"
        Properties = {
          UserPoolId = aws_cognito_user_pool.this.id
          RegionName = var.standby_region
          # Verify property names against the CFN resource reference before use.
        }
      }
    }
  })

  depends_on = [terraform_data.mrr_region_guard]
}
```

```hcl
# modules/cognito-user-pool/domain.tf

resource "aws_acm_certificate" "auth" {
  count             = var.custom_domain != null ? 1 : 0
  provider          = aws.us_east_1 # non-negotiable: Cognito custom domains are CloudFront-fronted
  domain_name       = var.custom_domain
  validation_method = "DNS"

  lifecycle { create_before_destroy = true }
}

resource "aws_cognito_user_pool_domain" "custom" {
  count        = var.custom_domain != null ? 1 : 0
  provider     = aws.primary
  domain       = var.custom_domain
  user_pool_id = aws_cognito_user_pool.this.id
  certificate_arn = aws_acm_certificate.auth[0].arn

  # Routing { Failover { SecondaryRegion, PrimaryRoute53HealthCheckId } } has
  # no provider argument yet. Apply out-of-band with UpdateUserPoolDomain, or
  # via the same CFN shim, until the provider catches up.
}
```

Caller, from a cookiecutter-rendered env stack:

```hcl
module "cognito" {
  source = "../../modules/cognito-user-pool"

  providers = {
    aws.primary   = aws.eu_west_1
    aws.standby   = aws.eu_west_2
    aws.us_east_1 = aws.us_east_1
  }

  name           = "helios"
  env            = "prd"
  primary_region = "eu-west-1"
  standby_region = "eu-west-2"

  enable_replication       = true
  feature_plan             = "ESSENTIALS"
  kms_multi_region_key_arn = module.kms.cognito_mrk_arn
  issuer_type              = "ORIGINAL" # flip after the ALB/API-GW authorizer audit

  custom_domain           = "auth.helios.example.com"
  primary_health_check_id = module.route53.primary_health_check_id

  lambda_triggers = {
    pre_token_generation = {
      primary_arn = module.auth_lambdas_primary.pre_token_arn
      standby_arn = module.auth_lambdas_standby.pre_token_arn
    }
  }
}
```

For the CA stack the same module is rendered with `enable_replication = false`,
the precondition never fires, and the difference is one boolean in the
cookiecutter context. That is the right shape: **the Region gap is expressed as
data, not as a forked module.**

---

## Migration path from single-region

The live pool must survive. Nothing here may destroy it.

### Phase 0 — audit (do this first, it may change the plan)

1. **Is the pool on next-gen infrastructure?** Open the console, Settings →
   Multi-Region replication. Eligible pools show configuration; ineligible ones
   show an exception. If ineligible, raise a support case asking for a timeline
   and **stop** — the rest of this is blocked.
2. **Find every ALB `authenticate-cognito` rule and every API Gateway
   `COGNITO_USER_POOLS` authorizer.** This determines whether the updated issuer
   is available to you.
3. **Find every place a Cognito Region, pool ID, app client ID or issuer URL is
   hard-coded** — mobile builds, SPA bundles, backend config, CI variables,
   third-party integrations, partner IdP relying-party configs.
4. **Find every place `sub` or a Cognito identity ID is used as a durable key.**
5. **Record current Service Quotas values** for `UserAuthentication`,
   `UserCreation`, `AccountRecovery` in the primary.

### Phase 1 — feature plan

`UpdateUserPool` with `UserPoolTier: ESSENTIALS`. **In-place, no downtime, no
replacement.** Cost impact is immediate and material if you are on Lite (roughly
2.7× the per-MAU rate before the MRR add-on). Get that signed off before you
click it.

Terraform: setting `user_pool_tier` on `aws_cognito_user_pool` is an update, not
a replacement. Verify with a `terraform plan` and read the output — if you see
`# forces replacement` anywhere near this resource, **stop and do it out of
band**, then `ignore_changes` the attribute. A user pool replacement is
unrecoverable.

### Phase 2 — KMS MRK

1. Create a multi-Region KMS key in the primary Region, replicate it to the
   standby ([[aws-kms]], [[kms-when-to-use-multi-region-keys]]).
2. Apply the Cognito key policy (both service principals, both encryption-context
   conditions, both `ViaService` statements).
3. `UpdateUserPool` with `KeyConfiguration: { KeyType: CUSTOMER_MANAGED_KEY,
   KmsKeyArn: <mrk-primary-arn> }`.

**This is an in-place update at the API level** — `UpdateUserPool` accepts
`KeyConfiguration`, so switching from AWS-owned to customer-managed does not
require a new pool. That is the single most important `ForceNew` question in this
note and the answer is favourable.

> ⚠️ **`ForceNew` watch.** There is a known history of Terraform destroying
> Cognito-adjacent resources on an encryption change —
> [hashicorp/terraform-provider-aws#28321](https://github.com/hashicorp/terraform-provider-aws/issues/28321)
> reports `encrypt_at_rest` destroying and recreating a resource with the same
> name, causing data loss. **Read every plan involving encryption settings on a
> user pool character by character. If in doubt, make the change with the AWS
> CLI and then `terraform import`/`ignore_changes` to reconcile.** A pool you
> cannot recreate deserves that paranoia.

### Phase 3 — standby dependencies (before the replica)

- **SES:** verify the domain identity in the standby, publish per-Region DKIM
  CNAMEs, and **get out of the sandbox there** — see [[aws-ses]], this is that
  note's headline failure mode.
- **SNS:** raise the SMS spending limit in the standby.
- **Lambda:** deploy every trigger into the standby ([[aws-lambda]]). Resource
  policies must allow `cognito-idp.amazonaws.com` with the *replica's* ARN as
  `SourceArn` — which, remember, has the standby Region in the ARN but the
  primary Region string inside the pool ID. Get that string right.
- **WAF:** create the web ACL in the standby.
- **Service Quotas:** raise the account-level max in the standby to match the
  primary. Start this early — it is the only step with a multi-day tail.
- **Route 53:** create the health check ([[aws-route53]]).

### Phase 4 — create the replica

`CreateUserPoolReplica`. It arrives `PENDING_CREATE`, then `INACTIVE`. Initial
sync time is undocumented and "depends on the amount of data in the user pool" —
for a large directory, allow real time and do not schedule anything behind it.

While `INACTIVE`, configure the per-replica settings: Lambda triggers, SES, SNS,
WAF, log export, tags. Verify with `DescribeUserPool` against the standby
endpoint.

### Phase 5 — activate and verify

`UpdateUserPoolReplica` → `ACTIVE`. Then, against the **standby Regional
endpoint**:

- `AdminGetUser` for a known user — present? attributes correct?
- `InitiateAuth` with a test user's real password — succeeds? tokens returned?
- Decode the token: is `iss` what your validators expect?
- Present a standby-issued token to the primary's API. Accepted?
- Present a primary-issued token to the standby's API. Accepted?
- Create a user in the primary; time how long until `AdminGetUser` in the standby
  returns it. **Record the number.** That is your measured RPO.
- Confirm `ListUserPoolClients` in the standby returns the same client IDs.

### Phase 6 — client-side routing

Build pattern 1 and/or 2 from the RTO section. **This is the long pole** and it
is application work, not infrastructure work. Ship it to production, behind the
same config the failover will flip, and exercise it in a low-traffic window.

### Phase 7 — issuer (only if phase 0 cleared it)

`UpdateUserPool` with the updated `IssuerConfiguration`. Before flipping:

- Every validator accepts **both** issuer strings for a transition period. Tokens
  minted before the flip carry the old `iss` and remain valid until expiry.
- No ALB `authenticate-cognito` rule and no API Gateway `COGNITO_USER_POOLS`
  authorizer remains.
- Replacements for those are deployed and tested.

Flip in a maintenance window; the blast radius is "every token validator in the
estate".

### Phase 8 — domain failover

`UpdateUserPoolDomain` with `Routing.Failover`. Test by inverting the health
check and confirming managed login serves from the standby. Then invert it back.

---

## Failover procedure

Assume an [[failover-orchestration]] runbook triggers this; these are the
Cognito-specific steps.

**Automated (no human):**

1. Route 53 health check goes unhealthy.
2. Cognito flips managed-login / federation / M2M-token serving to the standby.
3. The config endpoint (pattern 1) — driven by the same health check — starts
   returning the standby Region. Clients pick it up within one cache TTL.

**Human decision:**

4. Confirm this is a failover, not a flap. Cognito's domain failover is automatic
   and will flap with the health check; your client-side routing should be
   deliberately stickier. Think about hysteresis here or you get auth
   oscillating between Regions mid-session.
5. `UpdateProvisionedLimit` for `UserAuthentication` (and `AccountRecovery`) in
   the standby, if you chose not to provision permanently.
6. **Disable sign-up, forgot-password and profile-edit in the UI.** These will
   fail in the secondary and AWS explicitly tells you to hide them. If your app
   does not have a feature flag for this, add one *before* the incident — it is a
   two-line change now and an unshippable emergency later.
7. **Communicate the TOTP situation.** Users with TOTP MFA cannot sign in. If that
   is your admin population, your incident responders may be locked out of your
   own product. Have the break-glass path written down and tested.
8. Watch standby throttling metrics. Cognito emits throttle counts to CloudWatch;
   alarm on them in *both* Regions, permanently.

**Do not do during failover:**

- Do not promote the replica to primary. There is no such operation exposed in
  the documented API surface, and the replica is designed to be a replica.
- Do not create a new pool "just in case". You will end up with two directories
  and no way to merge them.

---

## Failback

Easier than most services here, and that is a genuine advantage of MRR over the
dual-write branch.

Because **the secondary never accepted writes**, there is nothing to reconcile.
No sign-ups happened there, no password changes, no profile edits. The primary's
directory is still authoritative and still complete (modulo the lag window at
the moment of failure, which was lost, not diverged). When the primary Region
recovers:

1. Verify the primary pool is healthy — `DescribeUserPool` and a test
   `InitiateAuth` against the primary endpoint, before touching the health check.
2. Let the Route 53 health check recover. Cognito routes managed login back
   automatically: "When the health check enters a healthy state, Amazon Cognito
   begins routing traffic back to the primary replica."
3. Flip the client-routing config back.
4. Drop the standby provisioned limit back to default (this is money).
5. Re-enable sign-up / forgot-password / profile-edit in the UI.
6. Confirm replication has caught up in the new direction — the primary is once
   again the write target and the secondary resyncs from it.

**The one thing to watch:** users who signed in during the failover hold tokens
with the standby's session state. Because refresh tokens are interoperable
between Regions and "automatically reflect the current issuer configuration when
refreshed", these should transition cleanly. **Verify it in a game day rather
than trusting it** — a failback that silently signs out your entire user base is
a self-inflicted second incident.

Contrast with **Option B (dual-write)**, where failback is the hard part: both
pools took writes, you have divergence in both directions, and there is no merge
primitive. That asymmetry is a strong argument for MRR wherever it is available.

---

## Gotchas

1. **`ca-west-1` is not an MRR Region.** The whole CA pair strategy for Cognito
   is different from EU and US. Feed this into [[region-pair-selection]]. Note
   that Cognito *itself* has been available in Calgary since
   [July 2024](https://aws.amazon.com/about-aws/whats-new/2024/07/amazon-cognito-canada-west-calgary-region/) —
   the service is there, the *replication feature* is not. "Service available in
   Region" and "feature available in Region" are different questions and this is
   the cleanest example of it in the vault.
2. **Eligibility is invisible from the API.** You cannot programmatically
   determine whether a pool can use MRR. In a multi-env monorepo this means dev
   and prod pools may differ in capability with no way to assert it in code.
3. **`prevent_destroy` on every user pool, today, before anything else.** A user
   pool is the least recreatable resource in the estate. Add the lifecycle block
   as a standalone PR now.
4. **Adding a custom attribute to a user pool is one-way.** You cannot delete or
   rename a custom attribute. Any schema mistake is permanent for the life of the
   pool. Hence `ignore_changes = [schema]` above — you want schema changes to be
   deliberate, reviewed, out-of-band operations, not something a template
   re-render can do to you.
5. **TOTP users cannot authenticate during failover.** Restating it because it is
   the sharpest edge in the whole feature and it is one bullet in the AWS docs.
6. **The updated issuer breaks ALB Cognito auth and API Gateway Cognito
   authorizers.** Restating for the same reason.
7. **JWKS availability is a separate failure from token issuance.** With the
   original issuer, your validators fetch keys from a Region-pinned host. Cache
   aggressively with stale-if-error, and consider bundling a JWKS snapshot into
   the standby deployment.
8. **Quota increases do not cross Regions.** Verbatim from AWS. Your standby is
   at defaults unless you did something about it.
9. **Raising Cognito quotas may require raising SNS/SES quotas.** AWS flags this
   explicitly; compound failure with [[aws-ses]].
10. **The custom-domain ACM cert must be in `us-east-1`**, regardless of the
    pool's Region, because Cognito fronts it with CloudFront. Same as
    [[aws-cloudfront]]. See [[aws-acm]].
11. **The parent domain needs a real A record.** Cognito refuses to attach
    `auth.example.com` unless `example.com` resolves to an IP — an anti-hijacking
    check. "A Start of Authority (SOA) record isn't a sufficient DNS record for
    the purposes of parent-domain verification." Catches people using a
    subdomain-only hosted zone. See [[aws-route53]].
12. **Only the custom domain serves `/.well-known/openid-configuration`.** If you
    have both a prefix domain and a custom domain, discovery only works on the
    custom one.
13. **Changing the custom-domain certificate takes up to an hour to propagate.**
    Routine ACM renewals keep the same ARN and need no action; *replacing* a cert
    changes the ARN and needs an `UpdateUserPoolDomain` plus an hour. Do not do
    this during a failover.
14. **App client secrets replicate with eventual consistency.** A client secret
    rotated seconds before a failover may not be present in the standby.
15. **`aws_cognito_user_pool.kms_key_id` is not the at-rest key.** It is the
    custom-sender-trigger key. Two different things with confusingly similar
    names.
16. **Enabling `USER_PASSWORD_AUTH` for a migration is a security downgrade.**
    Time-box it, and make reverting it a tracked task rather than a good
    intention.
17. **The user-migration Lambda needs the source pool to be alive.** Useless as a
    DR mechanism. Excellent as a migration mechanism.
18. **Identity pools are not replicated and identity IDs are not portable.** If
    an identity ID is a durable key anywhere, that is a data-migration problem,
    not a config problem.
19. **MAU billing for MRR is on your whole user base, not on failover traffic.**
    The standby costs real money every month whether or not you ever fail over.
20. **Managed-login domain failover is automatic and will flap.** Your
    client-side routing needs different (stickier) hysteresis than the health
    check, or sessions oscillate.

---

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| **EU + US pairs: replication approach** | Native MRR | Dual-write to two pools | **A.** MRR is cheaper per MAU, has no divergence problem, makes failback trivial, and preserves passwords, `sub`, app client IDs and refresh tokens. Dual-write's only advantage is working everywhere. |
| **CA pair: what to do** | Accept long auth RTO; keep the profile-export backup running; wait for AWS to add `ca-west-1` | Build dual-write to a second `ca-central-1` pool, or to `ca-west-1` | **A, with a caveat.** Dual-write is a lot of bespoke identity engineering for one Region pair. Document the auth RTO gap as an accepted risk, raise it with your AWS account team as a Region-expansion ask, and re-check the MRR Region list quarterly. **If Canadian auth availability is a contractual obligation, escalate now** — the honest answer is that AWS does not currently support it. |
| **CA pair: change the pair?** | Keep `ca-central-1` → `ca-west-1` and accept no Cognito replication | Pair `ca-central-1` → `us-east-1` for Cognito only | **A.** Option B breaks Canadian data residency for the user directory, which is almost certainly the reason the CA deployment exists. Losing residency to gain auth DR is the wrong trade. Note it in [[region-pair-selection]] and let the business decide. |
| **Issuer type** | Keep ORIGINAL, mitigate JWKS risk with aggressive caching | Switch to UPDATED, migrate off ALB/API-GW Cognito auth | **Depends on the phase-0 audit.** If the audit finds zero ALB `authenticate-cognito` rules and zero `COGNITO_USER_POOLS` authorizers → **B**, it is strictly better. If it finds many → **A** initially, with B as a follow-on workstream. Do not let the issuer question block the replica. |
| **Client routing** | Remote config endpoint | Backend proxy | **A for public clients** (works for mobile without a resubmit), **B where a backend is already on the path.** Add try/fallback as a safety net either way. |
| **Standby quota** | Raise account max now, bump provisioned limit at failover | Provision permanently | **A.** Provisioned limits take effect immediately, so the bump is safe on the critical path, and you avoid paying for idle RPS. But raise the *account-level max* now — that is the part with a multi-day tail. |
| **Replica activation state** | Leave `ACTIVE` permanently | Activate at failover | **A.** An `ACTIVE` replica serves no traffic until routed to. Activating at failover adds an API call and a decision to the critical path for no benefit. |
| **Terraform for the replica** | `awscc` / CFN shim now | Wait for the native resource | **A.** Waiting means the replica is a click-op, which in a templated monorepo is how environments drift. Isolate the shim in one module and swap it later. |
| **Feature plan** | Essentials | Plus | **Essentials**, unless you already use threat protection / advanced security, in which case you are on Plus anyway. Plus costs $0.020 vs $0.015 per MAU and $0.006 vs $0.0045 for the MRR add-on. |

---

## Cost

All figures from the [Amazon Cognito pricing page](https://aws.amazon.com/cognito/pricing/),
verified at the time of writing. **Re-verify before quoting to anyone.**

| Item | Rate |
|---|---|
| Lite, direct/social sign-in | $0.0055/MAU (first 90k over free tier), then $0.0046/MAU |
| Essentials, direct/social sign-in | $0.015/MAU |
| Plus, direct/social sign-in | $0.020/MAU |
| SAML/OIDC federation (all tiers) | $0.015/MAU above a 50-MAU free tier |
| Free tier (Lite and Essentials) | 10,000 MAU/month/account |
| **MRR add-on, Essentials** | **$0.0045/MAU per replica Region** |
| **MRR add-on, Plus** | **$0.006/MAU per replica Region** |
| M2M tokens, standard | $0.00225 per successful token request |
| M2M tokens with MRR | $0.002925 per token request (~30% uplift) |
| Provisioned limits | Billed for capacity provisioned above default, used or not |

Worked example, 500,000 MAU on Essentials in one Region pair:

- Today, Essentials, single Region: 490,000 billable × $0.015 = **$7,350/mo**
- Add MRR: + 500,000 × $0.0045 ≈ **+$2,250/mo** → **$9,600/mo**
- Dual-write alternative: 2 × $7,350 ≈ **$14,700/mo** *and* all the engineering

So MRR is roughly a **30% uplift on the Cognito line** and roughly **65% of the
cost of dual-write**, before counting the engineering time dual-write consumes.
If you are currently on **Lite**, the jump is larger, because you pay the
tier increase *and* the add-on: 490,000 × $0.0055 = $2,695/mo today →
$9,600/mo. **That is the number that will surprise the finance conversation**,
and it is a tier change, not a DR feature, that causes most of it.

Other line items in the standby are rounding errors: KMS MRK replica ~$1/mo,
Route 53 health check ~$0.50–$1/mo, WAF web ACL ~$5/mo plus rules, Lambda at
effectively zero while idle.

**Levers:**
- Stay on Essentials rather than Plus ($0.0045 vs $0.006 on the add-on, plus
  $0.005 on the base).
- Apply MRR per-environment: production only. Dev and staging do not need a
  replica, and the cookiecutter boolean makes that free to express. This is the
  single biggest lever.
- Keep provisioned limits at default while idle.
- Reduce MAU by tightening what counts as "active" — a user is an MAU if they
  perform an identity operation in the month, so chatty background token
  refreshes on inactive clients cost money. Worth an audit.

---

## Open questions

1. **Is Cognito actually in use in this estate, and where?** The brief says "if
   Cognito is in use". Confirm before any of this is scheduled. If the product
   uses a third-party IdP already, this note becomes background reading.
2. **Are the production user pools on the next-generation infrastructure?** Check
   the console for MRR options. **This gates everything** and there is no API for
   it. If not eligible, open a support case for a timeline.
3. **Which feature plan are the pools on today?** Lite → the cost conversation is
   much bigger than the DR conversation.
4. **How many ALB `authenticate-cognito` rules and API Gateway
   `COGNITO_USER_POOLS` authorizers exist?** Determines whether the updated
   issuer is reachable this year.
5. **Is `sub` (or a Cognito identity ID) used as a durable foreign key anywhere?**
   If yes, that is a data-integrity dependency on never rebuilding the pool, and
   it strengthens the case for MRR considerably.
6. **Is TOTP MFA mandatory for any user population — especially admins?** If yes,
   those users cannot sign in during a failover, and you need an alternative
   factor or a break-glass path.
7. **Are identity pools in use, or only user pools?** Identity pools got nothing
   in the June 2026 launch.
8. **Is Canadian data residency for the user directory a contractual
   obligation?** If yes, the CA pair has no compliant Cognito DR option today and
   that must be escalated rather than engineered around.
9. **What are the current Service Quotas values in each primary Region**, and how
   far above default? That is the size of the standby quota gap.
10. **What is the measured replication lag?** AWS publishes none. Measure it in a
    game day and record it here.
11. **What are the actual token lifetimes configured today?** Determines how much
    of the user base an auth outage is invisible to, and therefore how much
    Option D is worth.
12. **Is mobile in scope?** A mobile client with a hard-coded Region is the single
    worst constraint in this note, because the fix has an App Store review in the
    middle of it.

---

## Sources

- [Multi-Region replication for user pools — Amazon Cognito Developer Guide](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-multi-region.html)
  — the authoritative reference. Region list, limitations (TOTP, no writes in
  secondary, one replica, lockout counters), per-replica configurable settings,
  API operations allowed in `INACTIVE` vs `ACTIVE`, failover configuration.
- [Amazon Cognito now supports multi-Region replication — What's New, 4 June 2026](https://aws.amazon.com/about-aws/whats-new/2026/06/amazon-cognito-multi-region/)
  — launch date and Region list.
- [Improve your application resilience with Amazon Cognito multi-Region replication — AWS News Blog](https://aws.amazon.com/blogs/aws/improve-your-application-resilience-with-amazon-cognito-multi-region-replication/)
  — narrative walkthrough; the "redeploying server-side applications and
  resubmitting mobile apps to app stores" point comes from here.
- [Architecting resilient authentication with Amazon Cognito multi-Region replication — AWS Security Blog](https://aws.amazon.com/blogs/security/architecting-resilient-authentication-with-amazon-cognito-multi-region-replication/)
  — the three architecture patterns, app-client-secret eventual consistency,
  refresh-token interoperability, CloudWatch Synthetics and FIS testing guidance.
- [Amazon Cognito unlocks advanced capabilities with next-generation infrastructure — AWS Security Blog](https://aws.amazon.com/blogs/security/amazon-cognito-unlocks-advanced-capabilities-with-next-generation-infrastructure/)
  — the eligibility gate. Confirms MRR, CMK and high throughput all depend on the
  rebuilt storage layer; gives no eligibility API and no timeline.
- [Identity provider and relying party endpoints — Amazon Cognito](https://docs.aws.amazon.com/cognito/latest/developerguide/federation-endpoints.html)
  — original vs updated issuer formats, and the ALB / API Gateway
  incompatibility. The single most consequential paragraph for this programme.
- [IssuerConfigurationType — Cognito User Pools API Reference](https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_IssuerConfigurationType.html)
  — the API surface for changing issuer type.
- [Data protection in Amazon Cognito](https://docs.aws.amazon.com/cognito/latest/developerguide/data-protection.html)
  — CMK requirements (symmetric, same Region, ARN not alias), the full key
  policy, and confirmation that `KeyConfiguration` is settable on
  `UpdateUserPool` (i.e. not a replacement).
- [Importing users into user pools from a CSV file](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-using-import-tool.html)
  — the `RESET_REQUIRED` quote, the 500,000-row / 100 MB / 16,000-char limits,
  one active import job per account, and the 2026 password-hash import feature
  with its four supported algorithms. Confirms hashes can be imported *in* but
  says nothing about exporting them *out*.
- [Migrate user Lambda trigger — Amazon Cognito](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-migrate-user.html)
  — trigger sources `UserMigration_Authentication` and
  `UserMigration_ForgotPassword`.
- [Importing users with a user migration Lambda trigger](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-import-using-lambda.html)
  — the `USER_PASSWORD_AUTH` / `ADMIN_USER_PASSWORD_AUTH` requirement.
- [Approaches for migrating users to Amazon Cognito user pools — AWS Security Blog](https://aws.amazon.com/blogs/security/approaches-for-migrating-users-to-amazon-cognito-user-pools/)
  — AWS's own comparison of bulk import vs just-in-time migration.
- [Using your own domain for managed login](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-add-custom-domain.html)
  — the `us-east-1` ACM requirement ("regardless of the AWS Region of your user
  pool"), parent-domain A-record check, one-hour cert propagation, TLS policy
  options.
- [Quotas in Amazon Cognito](https://docs.aws.amazon.com/cognito/latest/developerguide/quotas.html)
  — "AWS can only grant a quota increase request in one Region at a time";
  `UserAuthentication` default 120 RPS, `UserCreation` 50 RPS, the 3×
  `RespondToAuthChallenge` rule, and the warning about SNS/SES quotas.
- [From 2 weeks to 2 minutes: Amazon Cognito launches Provisioned limits — AWS Security Blog, 6 July 2026](https://aws.amazon.com/blogs/security/from-2-weeks-to-2-minutes-amazon-cognito-launches-provisioned-limits-for-self-service-rate-limit-management/)
  — self-service rate limits, effective immediately, billed on provisioned
  capacity. Changes the standby-quota story materially.
- [Amazon Cognito pricing](https://aws.amazon.com/cognito/pricing/)
  — all per-MAU figures and the MRR add-on rates.
- [Guidance for User Profiles Export with Amazon Cognito](https://docs.aws.amazon.com/solutions/latest/cognito-user-profiles-export-reference-architecture/overview.html)
  — the DIY export/import pattern onto a DynamoDB global table, and AWS's own
  admission that federated users are not exported and "some data loss will
  result".
- [aws-samples/sample-cognito-user-profiles-export-reference-architecture](https://github.com/aws-samples/sample-cognito-user-profiles-export-reference-architecture)
  — the code for the above, now moved out of aws-solutions.
- [hashicorp/terraform-provider-aws#48227 — `aws_cognito_user_pool` multi-region replication](https://github.com/hashicorp/terraform-provider-aws/issues/48227)
  — open, filed 5 June 2026, no milestone. The provider gap.
- [hashicorp/terraform-provider-aws#28321 — enabling `encrypt_at_rest` is destroying the existing domain](https://github.com/hashicorp/terraform-provider-aws/issues/28321)
  — precedent for an encryption-setting change causing a destroy/recreate with
  data loss. The reason to read encryption plans character by character.
- [AWS::Cognito::UserPoolReplica — CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-cognito-userpoolreplica.html)
  — the CloudFormation resource that makes the shim viable.
- [CreateUserPoolReplica — Cognito User Pools API Reference](https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_CreateUserPoolReplica.html)
  — the API call and its response shape.
- [Amazon Cognito is now available in Canada West (Calgary) Region — July 2024](https://aws.amazon.com/about-aws/whats-new/2024/07/amazon-cognito-canada-west-calgary-region/)
  — Cognito exists in Calgary. MRR does not. The distinction that matters for the
  CA pair.

**Not found, stated as such:** no public post-mortem or case study of a real
production Cognito Regional failover was located — MRR is three months old at the
time of writing and nobody has published a war story yet. No AWS-published
replication-lag figure or SLA for MRR. No API for querying next-generation
infrastructure eligibility. No documented operation to promote a replica to
primary.
