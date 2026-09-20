---
title: Amazon API Gateway — Multi-Region
service: api-gateway
tags: [service, multi-region, api-gateway, rest-api, http-api, websocket, quotas, edge]
status: researched
replication: manual — deploy a second copy per region; nothing replicates natively
rpo_achievable: "N/A — stateless control/data plane. The only stateful thing is usage-plan quota counters, which do NOT replicate."
rto_achievable: "~0 s for the gateway itself if pre-provisioned. The RTO risk is not API Gateway — it is the standby region's account-level throttle quota."
meets_targets: conditional
updated: 2026-09-17
---

# Amazon API Gateway — Multi-Region

## TL;DR

- **API Gateway is regional. There is no multi-region API Gateway.** The
  mirroring answer is "deploy a second, identical API in the standby region and
  put your own routing in front of it" — and that is genuinely the right answer,
  not a cop-out. The gateway contributes ~0 seconds to the RTO if it is
  pre-provisioned.
- **Edge-optimized endpoints are the trap.** They are fronted by an AWS-managed
  CloudFront distribution, which makes them *sound* global. **The API still lives
  in exactly one region**, and an edge-optimized custom domain name is globally
  unique — **you cannot create the same edge-optimized custom domain in two
  regions**. That single fact rules edge-optimized out of this architecture. See
  [[#Edge-optimized is not multi-region and this confuses everyone]].
- **The core pattern is one `REGIONAL` custom domain name per region, each with
  its own in-region ACM certificate, with Route 53 failover records over the two
  `regional_domain_name` values.** Full Terraform in
  [[#Terraform implementation]]. Cross-reference [[aws-acm]] — the ACM validation
  CNAME is shared across regions, so this costs zero extra DNS records.
- **Two customer-visible failure modes that only appear at failover:**
  **(1) API keys are regional and do not replicate** — a partner's key silently
  returns `403 Forbidden` in the standby unless the *same key value* was
  pre-created there; **(2) the standby's account-level throttle quota is almost
  certainly still at the default** while the primary's has been raised over
  years. Both are invisible in every test that doesn't actually shift production
  traffic.
- **The thing that will bite, stated once and loudly:** *account-level service
  quotas are per-region and do not follow you.* For the **CA pair this is already
  a confirmed 4× gap** — `ca-west-1` (Calgary) is on AWS's reduced-default list
  at **2,500 RPS / 1,250 burst**, while `ca-central-1` gets the standard
  **10,000 RPS / 5,000 burst**. Failing Montreal over to Calgary today means
  landing production traffic on a region whose gateway throttle ceiling is a
  quarter of what you left. See [[#Account-level throttling is a per-region quota and this is a silent RTO killer]].

## Does this service cross regions at all?

No. Every object in API Gateway is regional:

| Object | Scope | Notes |
|---|---|---|
| `RestApi` / `HttpApi` / `WebSocketApi` | **Regional** | The `{api-id}` differs per region. Any hard-coded `execute-api` URL is region-pinned. |
| Stage, deployment, stage variables | Regional | |
| **Custom domain name (`REGIONAL`)** | **Regional** | The *same name* can exist in many regions. This is the whole pattern. |
| **Custom domain name (`EDGE`)** | **Globally unique** | Cannot be created twice. See below. |
| **API key** | **Regional** | **Does not replicate.** Same name ≠ same key value. |
| Usage plan + its quota counters | **Regional** | Counters are per-region; a 1M/month quota becomes 2M/month across a pair. |
| Authorizer (Lambda / Cognito / JWT) | Regional | Points at a regional Lambda ARN or a regional user pool. |
| VPC link | **Regional**, and the NLB must be in the same region **and account** | |
| Resource policy | Regional, but its contents embed region-specific ARNs | |
| WAF web ACL association (`REGIONAL` scope) | Regional | |
| mTLS truststore | The S3 URI is a normal `s3://` URI | See [[#Mutual TLS]] |
| **Account-level throttle quota** | **Per account, per Region** | **The one that kills you.** |

There is no `CreateGlobalApi`, no replication flag, no equivalent of Secrets
Manager's replica regions or DynamoDB's Global Tables. The design work is
entirely in **what you put in front of the two copies**.

## Edge-optimized is not multi-region, and this confuses everyone

This section exists because the confusion is near-universal and it has cost
teams months.

An **edge-optimized** REST API endpoint is served through an **AWS-managed
CloudFront distribution** that AWS creates and owns on your behalf. Requests hit
the nearest CloudFront point of presence and are carried over the AWS backbone to
your API. Because CloudFront is a global service and the console shows you a
`*.cloudfront.net` `distributionDomainName`, it looks like the API is
"everywhere".

**It is not.** The distribution is a *front door*. The API — the thing that
executes your integrations, applies your authorizers and holds your stages —
lives in exactly one region. If that region goes away, the edge-optimized
distribution has nothing to forward to and every PoP on earth returns errors in
unison. Edge-optimized buys you **latency**, not **availability**.

From the [API endpoint types doc](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-endpoint-types.html), verbatim:

> Any custom domain name that you use for an edge-optimized API applies across
> all regions.

and, for regional:

> For a Regional API, any custom domain name that you use is specific to the
> Region where the API is deployed. If you deploy a Regional API in multiple
> Regions, it can have the same custom domain name in all Regions. You can use
> custom domains together with Amazon Route 53 to perform tasks such as
> latency-based routing.

Those two sentences are the entire decision. **"Applies across all regions"
means the name is claimed globally and can only be attached to one
CloudFront distribution — so you physically cannot create `api.example.com` as an
edge-optimized custom domain in both `eu-west-1` and `eu-west-2`.** The second
`CreateDomainName` fails. There is no routing layer you can build on top of a
name you can only create once.

Three further reasons edge-optimized is wrong here, beyond the blocking one:

1. **You cannot attach your own WAF web ACL to the AWS-managed distribution.**
   You attach a `REGIONAL`-scope web ACL to the *stage* instead, which works, but
   you get none of the CloudFront-layer controls.
2. **You cannot see, tune or invalidate the distribution.** No origin timeouts,
   no origin groups, no cache policies, no CloudFront Functions.
3. **The edge-optimized certificate must be in `us-east-1`** regardless of where
   the API lives ([[aws-acm]]). For the EU and CA pairs this drags a `us-east-1`
   control-plane dependency into a deployment that otherwise has none — and
   `us-east-1` is where the Route 53 and CloudFront control planes live too
   ([[aws-route53]]).

> [!important] The correct statement to put in the design doc
> "Edge-optimized gives you a CloudFront distribution you do not control, in
> front of an API that is still single-region. Regional gives you an endpoint you
> *do* control the routing to. Active/passive requires controlling the routing.
> Therefore: **Regional, everywhere, no exceptions.**"

If you want CloudFront's latency benefit *and* multi-region, you bring **your
own** CloudFront distribution in front of two regional APIs. AWS says so directly
in the same doc:

> In cases where API clients are geographically dispersed, it may still make
> sense to use a Regional API endpoint, together with your own Amazon CloudFront
> distribution to ensure that API Gateway does not associate the API with
> service-controlled CloudFront distributions.

See [[#The alternative: skip API Gateway custom domains entirely]].

## The three API types, compared for this architecture

| | **REST API** (`apigateway` v1) | **HTTP API** (`apigatewayv2`) | **WebSocket API** (`apigatewayv2`) |
|---|---|---|---|
| Endpoint types | `EDGE`, `REGIONAL`, `PRIVATE` | **`REGIONAL` only** | **`REGIONAL` only** |
| Edge-optimized custom domain | Supported (and unusable here) | **Not supported** | **Not supported** |
| Multi-region custom domain | Yes, via `REGIONAL` | **Yes — the only option, which is a feature** | Yes |
| Terraform domain resource | `aws_api_gateway_domain_name` | `aws_apigatewayv2_domain_name` | `aws_apigatewayv2_domain_name` |
| Route 53 alias target attrs | `regional_domain_name` / `regional_zone_id` | `domain_name_configuration[0].target_domain_name` / `.hosted_zone_id` | same as HTTP API |
| Mapping resource | `aws_api_gateway_base_path_mapping` | `aws_apigatewayv2_api_mapping` | `aws_apigatewayv2_api_mapping` |
| **API keys / usage plans** | **Yes — and they are the regional trap** | No native API keys | No |
| Authorizers | Lambda (TOKEN/REQUEST), Cognito, IAM | Lambda (simple + IAM-policy), **native JWT**, IAM | Lambda (REQUEST), IAM |
| WAF | Yes, `REGIONAL` ACL on the stage | **No native WAF association** | **No native WAF association** |
| Resource policies | Yes | No | No |
| mTLS | Yes (regional custom domain only) | Yes | No |
| Stage variables | Yes | Yes | Yes |
| VPC link | v1 VPC link → **NLB only** | v2 VPC link → **ALB, NLB, Cloud Map** | v2 VPC link |
| Price (first 300M req/mo, US East / Ireland) | **$3.50 / million** | **$1.00 / million** | **$1.00 / million messages** + **$0.25 / million connection-minutes** |
| Idle cost | $0 (plus caching, if enabled) | $0 | $0 |

Sources: [endpoint types](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-endpoint-types.html),
[HTTP API custom domains](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-custom-domain-names.html),
[pricing](https://aws.amazon.com/api-gateway/pricing/).

### The multi-region story differs per type — the short version

- **HTTP APIs are the easiest to make multi-region**, precisely because AWS
  removed the footgun: there is no edge-optimized option, so every HTTP API
  custom domain is already regional and already duplicable. If the estate is on
  HTTP APIs, the custom-domain work is nearly free. Note the constraint: *"You
  can only map HTTP APIs to a regional custom domain name with the TLS 1.2
  security policy."*
- **REST APIs are the hard case**, because the default endpoint type for a REST
  API is `EDGE`. Anything created without thinking is edge-optimized, which means
  **the migration in [[#Migration path from single-region]] is a real workstream,
  not a no-op**. REST APIs are also the only type with API keys and usage plans,
  which is the other regional trap.
- **WebSocket APIs are the awkward case, and not because of the custom domain.**
  The domain mirrors fine. The problem is that a WebSocket API's value is a
  **long-lived connection whose state — the `connectionId` and whatever mapping
  you hold from `connectionId` to user — lives in the region that accepted the
  connection.** At failover:
  - Every open connection dies. There is no connection migration and no
    equivalent of Global Accelerator's "established connections are preserved"
    (they aren't preserved there either — see [[aws-alb-nlb]]).
  - The standby's `@connections` management API is a **different endpoint**
    (`https://{new-api-id}.execute-api.{standby}.amazonaws.com/{stage}`). Any
    backend that posts to connections must resolve that endpoint **at runtime**,
    not from a baked-in environment variable. This is the single most common
    WebSocket failover bug.
  - `connectionId`s from the dead region are meaningless in the new one. Your
    connection table (usually DynamoDB — [[aws-dynamodb]]) will be full of stale
    IDs that all return `410 Gone`. Store the region alongside the
    `connectionId` and have the fan-out path skip or purge foreign-region rows.
  - **Clients must reconnect.** That is a client-side requirement — exponential
    backoff with jitter, and a re-subscribe on reconnect. If the mobile/web
    client does not reconnect automatically, your WebSocket RTO is "whenever the
    user reloads the page", which is unbounded. Put this in
    [[failover-runbooks]] as a client-readiness pre-flight, in the same family as
    the JVM DNS TTL check in [[aws-route53]].
  - The **connection-minutes** billing line means a warm standby WebSocket API
    with zero connections costs **$0**. There is no pre-warming to pay for.

## The core pattern: one regional custom domain per region

This is the part to get exactly right, because everything else hangs off it.

`aws_api_gateway_domain_name` (and `aws_apigatewayv2_domain_name`) is a
**regional resource**. To serve `api.example.com` from two regions you:

1. Create an **ACM certificate for `api.example.com` in each region**
   (`eu-west-1` and `eu-west-2`). Per [[aws-acm]], both validate against the
   **same, single** Route 53 CNAME — mirroring the certificate adds **zero** new
   DNS records.
2. Create a **custom domain name with the same name in each region**, each
   referencing its own regional certificate, with
   `endpoint_configuration { types = ["REGIONAL"] }`.
3. Each one exposes a distinct AWS-owned target hostname —
   `regional_domain_name`, of the form `d-xxxxxxxxxx.execute-api.<region>.amazonaws.com`
   — plus `regional_zone_id`, the Route 53 hosted-zone ID you need for an alias
   record.
4. Create **two Route 53 records with the same name**, `set_identifier`
   distinct, one `PRIMARY` and one `SECONDARY`, each an **alias** to that
   region's `regional_domain_name` / `regional_zone_id`.
5. Map stages onto the domain with `aws_api_gateway_base_path_mapping` (v1) or
   `aws_apigatewayv2_api_mapping` (v2), **in each region**.

The client sees one hostname. Route 53 decides which region's custom domain it
resolves to. Nothing about the API Gateway configuration changes at failover —
which is exactly the static stability the brief demands.

> [!note] `regional_zone_id` is not your hosted zone ID
> `regional_zone_id` is the zone ID of the **AWS-managed zone that owns the
> `d-xxxx.execute-api.<region>.amazonaws.com` name**, and it is a per-region
> constant. It goes in `alias { zone_id = ... }`. Your own zone ID goes in the
> record's top-level `zone_id`. Getting these the wrong way round produces a
> confusing `InvalidChangeBatch` and is the most common first-attempt error.

## API keys are regional and do not replicate

> [!danger] This is a customer-visible failure at the worst possible moment
> `aws_api_gateway_api_key` creates a key **in one region**. Creating a key with
> the same *name* in the standby produces a **different key value**. A partner
> who has hard-coded `x-api-key: abc123...` gets a **`403 Forbidden` with
> `{"message":"Forbidden"}`** the instant DNS moves — not a 5xx, not a retryable
> error, a flat authentication failure that looks to them like *you revoked their
> access*. Support tickets, at 3am, during an outage, from your paying
> customers.

Three things compound it:

1. **The value is immutable.** From the
   [API keys doc](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-setup-api-key-with-console.html):
   *"After you create an API key value, it cannot be changed."* You cannot
   retrofit a matching value onto an auto-generated key in the standby; you must
   delete and recreate.
2. **The primary's existing keys were almost certainly auto-generated.** So you
   cannot simply "create the same ones" — you have to read the existing values
   out of the primary and import them into the standby.
3. **Reading them out requires an explicit flag.** `aws apigateway get-api-keys`
   omits values by default; you need `--include-values`. That is a deliberately
   privileged operation and the output is a secret.

### The fix

AWS supports **customer-specified key values** — the console's "Custom" option,
and the `value` argument on `CreateApiKey` / `ImportApiKeys`. So:

**Make every API key value an explicitly managed secret, generated once, stored
in Secrets Manager, and applied identically to both regions.** Secrets Manager
already replicates natively ([[aws-secrets-manager]], already done in prod), so
the secret is available in the standby for free.

```hcl
# The key value is generated ONCE and lives in Secrets Manager, which already
# replicates to the standby region. Both regional api_key resources read the
# same value, so a customer's key works in either region.
data "aws_secretsmanager_secret_version" "partner_key" {
  for_each  = var.api_key_consumers          # e.g. { acme = "acme-corp", globex = "globex" }
  secret_id = "apigw/${var.env}/api-keys/${each.key}"
}

resource "aws_api_gateway_api_key" "primary" {
  for_each = var.api_key_consumers
  provider = aws

  name  = each.value
  value = jsondecode(data.aws_secretsmanager_secret_version.partner_key[each.key].secret_string)["key"]

  lifecycle {
    # The value is immutable in AWS. Without this, a secret rotation produces a
    # destroy/create that revokes the customer's access mid-apply.
    prevent_destroy = true
  }
}

resource "aws_api_gateway_api_key" "standby" {
  for_each = var.api_key_consumers
  provider = aws.standby

  name  = each.value
  value = jsondecode(data.aws_secretsmanager_secret_version.partner_key[each.key].secret_string)["key"]

  lifecycle {
    prevent_destroy = true
  }
}
```

Notes:

- **API Gateway enforces a minimum length on custom key values.** Generate them
  with a generous length (32+ characters of `[A-Za-z0-9]`) and you will never
  meet the boundary. Verify the exact minimum against the
  [API key file format doc](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-key-file-format.html)
  before writing a generator.
- **The value lands in Terraform state in plaintext.** It is a credential in
  state either way; that is an argument for the state-encryption posture in
  [[terraform-state-management]], not an argument against this pattern.
- **HTTP APIs have no API keys at all.** If the estate is on HTTP APIs this whole
  section is N/A and you are using a Lambda authorizer or JWT authorizer
  instead — which has its own, different, regional problem (below).

### Usage plans: the quota counter does not replicate either

A usage plan holds two things: **throttle** settings (rate/burst) and a **quota**
(e.g. 1,000,000 requests per month). The configuration mirrors easily — it is
just Terraform. **The counters do not.**

Consequences worth writing down:

- A customer on a 1M-requests-per-month quota has, in practice, **1M in the
  primary and a fresh 1M in the standby**. Across a failover month they can
  legitimately consume up to 2M. If quota enforcement is contractual or is how
  you bill, **this is a revenue leak that appears only in failover months**.
- Conversely, if you fail over on the 28th of the month, a customer who had
  burned 95% of their primary quota arrives in the standby with a **full**
  quota. That direction is customer-friendly and nobody will complain.
- **Usage plan *ids* differ per region**, so any external system that stores a
  usage plan ID (a self-service portal, a billing integration) must resolve it
  per region, not from a config file.
- Quotas are documented as best-effort anyway: *"Both throttles and quotas are
  applied on a best-effort basis and should be thought of as targets rather than
  guaranteed request ceilings."* Do not build billing on them in either region.

**Recommendation: mirror the usage plans, accept the counter reset, and treat
quota as a protection mechanism rather than a billing mechanism.** There is no
mechanism to synchronise the counters and building one is not worth it for a
service that is passive 99.9% of the time. If quota is genuinely contractual,
meter it in your own application (or in the authorizer) against a replicated
store — [[aws-dynamodb]] Global Tables — rather than in API Gateway.

```hcl
# Usage plans, mirrored. Note the stage reference is per-region: a usage plan
# can only reference API stages in its own region.
resource "aws_api_gateway_usage_plan" "standard" {
  for_each = local.regions   # { primary = ..., standby = ... } via a small module, see below
  name     = "${var.env}-standard"

  api_stages {
    api_id = each.value.rest_api_id
    stage  = each.value.stage_name

    throttle {
      path        = "/orders/POST"
      rate_limit  = 50
      burst_limit = 100
    }
  }

  throttle_settings {
    rate_limit  = 200
    burst_limit = 400
  }

  quota_settings {
    limit  = 1000000
    period = "MONTH"
    # NOTE: this counter is per-REGION. A failover resets it. See note above.
  }
}
```

## Account-level throttling is a per-region quota, and this is a silent RTO killer

**This is the most important section in this note and the point generalises
across the whole project.**

### The mechanism

From the [throttling doc](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html), verbatim:

> API Gateway throttles requests to your API using the token bucket algorithm,
> where a token counts for a request. Specifically, API Gateway examines the rate
> and a burst of request submissions against all APIs in your account, **per
> Region**.

and:

> Per-account limits are applied to all APIs in an account in a specified Region.
> The account-level rate limit can be increased upon request — higher limits are
> possible with APIs that have shorter timeouts and smaller payloads. To request
> an increase of account-level throttling limits per Region, contact the AWS
> Support Center.

The order of application, verbatim from the same page:

> 1. Per-client or per-method throttling limits that you set for an API stage in a usage plan
> 2. Per-method throttling limits that you set for an API stage
> 3. Account-level throttling per Region
> 4. AWS Regional throttling

### The numbers

From [Amazon API Gateway quotas](https://docs.aws.amazon.com/apigateway/latest/developerguide/limits.html):

| Quota | Default | Adjustable |
|---|---|---|
| Throttle quota per account, per Region, **across HTTP APIs, REST APIs, WebSocket APIs and WebSocket callback APIs** | **10,000 RPS**, burst bucket capacity **5,000** | **Yes** |

and the footnote that matters enormously here, verbatim:

> For the following Regions, the default throttle quota is 2500 RPS and the
> default burst quota is 1250 RPS: Africa (Cape Town), Europe (Milan), Asia
> Pacific (Jakarta), Middle East (UAE), Asia Pacific (Hyderabad), Asia Pacific
> (Melbourne), Europe (Spain), Europe (Zurich), Israel (Tel Aviv), **Canada West
> (Calgary)**, Asia Pacific (Malaysia), Asia Pacific (Thailand), and Mexico
> (Central).

Note also: **the burst quota is not customer-controllable.** *"The burst quota is
determined by the API Gateway service team based on the overall RPS quota for the
account in the Region. It is not a quota that a customer can control or request
changes to."* You raise RPS; burst follows at AWS's discretion.

### What this means for each pair

| Pair | Primary default | Standby default | Gap before any increases |
|---|---|---|---|
| EU: `eu-west-1` → `eu-west-2` | 10,000 | 10,000 | None *at default* — but see below |
| US: `us-east-1` → `us-west-2` | 10,000 | 10,000 | None *at default* — but see below |
| **CA: `ca-central-1` → `ca-west-1`** | **10,000** | **2,500 / burst 1,250** | **4× shortfall, built in, today** |

> [!danger] The CA pair has a confirmed, documented 4× throttle gap
> `ca-west-1` is on AWS's reduced-default list. `ca-central-1` is not. Even with
> **zero** history of quota increases, failing Montreal over to Calgary lands
> production traffic on a gateway whose account-level ceiling is **2,500 RPS with
> a 1,250 burst bucket**. If Montreal peaks above 2,500 RPS, the failover
> produces a wall of `429 Too Many Requests` — a total outage that looks exactly
> like a successful failover on every dashboard except the customer's.
>
> This is not speculative and it is not "check whether". It is documented. It is
> the single most concrete `ca-west-1` finding in this vault so far. Feed it to
> [[region-pair-selection]].

And for EU and US, the *defaults* match — but **the defaults are not what you
have.** A production account that has been running `eu-west-1` for years has
almost certainly had its throttle quota raised, possibly several times, possibly
by an engineer who has since left. **Quota increases are per-account,
per-region, and they are invisible in Terraform, invisible in the console's
default view, and invisible in every DR test that does not send real traffic
volume.** The standby is sitting at 10,000 while the primary is at 40,000, and
nobody knows.

### Why this is *silent*

Every other failover failure mode announces itself. A missing certificate fails
TLS. A missing DNS record fails to resolve. A missing Lambda fails with a 502.

A quota shortfall produces **a working system that serves a fraction of its
traffic**. Health checks pass — Route 53 health checkers send a handful of
requests per minute and will never see a 429 ([[aws-route53]]). Synthetic
canaries pass. The standby looks healthy. And the load test you ran against the
standby last quarter used 1% of production volume, so it passed too.

**The only things that detect it are (a) an explicit quota audit, and (b) a real
load test at production volume.** Do both.

### Auditing quota parity between a region pair

Service Quotas exposes applied values through an API, so this is a cron job, not
a project.

```bash
#!/usr/bin/env bash
# quota-parity.sh — diff APPLIED service quotas between two regions.
# Run weekly per pair; alert on any difference. Exit 1 on drift.
set -euo pipefail

PRIMARY="${1:?primary region}"
STANDBY="${2:?standby region}"
# apigateway = API Gateway, lambda, dynamodb, ec2, vpc, elasticloadbalancing, sqs, kms ...
SERVICES="${3:-apigateway lambda ec2 vpc elasticloadbalancing dynamodb sqs kms}"

dump() {  # region service -> "QuotaCode<TAB>Value<TAB>Name"
  aws service-quotas list-service-quotas \
    --region "$1" --service-code "$2" \
    --query 'Quotas[].[QuotaCode,Value,QuotaName]' \
    --output text 2>/dev/null | sort
}

drift=0
for svc in $SERVICES; do
  diff <(dump "$PRIMARY" "$svc") <(dump "$STANDBY" "$svc") > "/tmp/${svc}.diff" || {
    echo "=== $svc: quota drift $PRIMARY vs $STANDBY ==="
    cat "/tmp/${svc}.diff"
    drift=1
  }
done
exit "$drift"
```

Two caveats that will confuse you the first time:

- **`list-service-quotas` returns *applied* quotas — i.e. quotas that have been
  explicitly changed.** Quotas still at their default value may not appear at
  all. Use `list-aws-default-service-quotas` to get the baseline, and
  `get-service-quota --service-code apigateway --quota-code L-8A5B8E43` for a
  specific one. The API Gateway account throttle quota code is **`L-8A5B8E43`**
  (visible in the Service Quotas console URL AWS links from its own docs). A
  quota being *absent* from `list-service-quotas` in the standby while *present*
  in the primary **is itself the drift signal** — it means the primary was raised
  and the standby was not.
- **Service Quotas does not cover every quota.** Some limits are still
  support-ticket-only and not represented. The script is a floor, not a
  guarantee.

### Requesting increases ahead of time

- Raise them **now**, in a change window, months before you need them — not in
  the runbook. Quota increase requests are a **control-plane operation processed
  by humans or by automated approval**, and during a large regional event AWS
  Support is the busiest it ever gets. "Request a quota increase" is not a
  15-minute RTO step and must never appear in a failover runbook.
- Request **parity plus headroom**: whatever the primary is at, request the same
  in the standby. Do not request "what we think we need" — at failover the
  standby takes **100%** of the primary's traffic, not a share of it.
- API Gateway's own guidance is that *"higher limits are possible with APIs that
  have shorter timeouts and smaller payloads"* — so a request for a very high RPS
  may come back with questions about your integration timeouts. Have the numbers
  ready.
- Terraform can *request* increases declaratively via
  `aws_servicequotas_service_quota`, which is worth doing so parity is enforced
  by code rather than by memory:

  ```hcl
  # Declare the account throttle quota in BOTH regions so drift is a plan diff.
  # L-8A5B8E43 = API Gateway "Throttle rate" (account, per Region).
  resource "aws_servicequotas_service_quota" "apigw_throttle_primary" {
    provider     = aws
    service_code = "apigateway"
    quota_code   = "L-8A5B8E43"
    value        = var.apigw_account_throttle_rps
  }

  resource "aws_servicequotas_service_quota" "apigw_throttle_standby" {
    provider     = aws.standby
    service_code = "apigateway"
    quota_code   = "L-8A5B8E43"
    value        = var.apigw_account_throttle_rps   # SAME variable. That is the point.
  }
  ```

  Be aware this resource models a *request*: applying it opens a quota increase
  case and the apply can sit waiting, and lowering a quota is not always
  possible. Treat it as a one-way ratchet, and consider running it from a
  separate, rarely-applied `quotas/` stack rather than the main service stack.

> [!important] Generalise this beyond API Gateway
> **Every account-level quota in the standby region is at its default unless
> someone raised it.** The same argument applies verbatim to
> [[aws-lambda]] concurrent executions (the worst one), EC2 vCPU limits for EKS
> nodes ([[aws-eks]]), VPC elastic IPs and NAT gateways ([[aws-vpc-networking]]),
> ELB count ([[aws-alb-nlb]]), SQS ([[aws-sqs]]), KMS request rates
> ([[aws-kms]]) and RDS instance counts ([[aws-rds-postgres]]). **Quota parity
> should be its own workstream item and its own recurring check**, not a
> paragraph inside twenty service notes. Raise it in [[sequencing-roadmap]].

## Authorizers

### Lambda authorizers

A Lambda authorizer is a regional Lambda function referenced by its regional ARN.
Mirroring it means deploying the function in the standby and pointing the standby
API's authorizer at the standby ARN — straightforward, and covered in
[[aws-lambda]].

The failover-specific problem is **cold start, at exactly the moment you can
least afford it**:

- At failover, every request arriving at the standby is a **cache miss** on the
  authorizer result cache, because the cache is per-API-Gateway-deployment and
  the standby's is empty.
- So the first wave of traffic invokes the authorizer Lambda **once per unique
  token**, concurrently, into a function with **zero warm containers**. That is a
  cold-start stampede on the authentication path — the one path where a failure
  means a 401/403 rather than a retry.
- The authorizer is usually the *slowest-to-initialise* Lambda in the estate,
  because it loads JWKS, fetches signing keys over the network, and often
  initialises a crypto library.

Mitigations, in order of cost-effectiveness:

1. **Turn the authorizer result cache on and set a sensible TTL.** Caching is
   configurable **0–3600 seconds**. `authorizer_result_ttl_in_seconds = 300` is a
   reasonable default: long enough to collapse the stampede within seconds of
   failover, short enough that a revoked token dies quickly. **A TTL of 0
   disables caching entirely** and is a common accidental setting.
   - **Cache key matters.** For a `TOKEN` authorizer the key is the token itself,
     so a thousand distinct users produce a thousand distinct invocations. For a
     `REQUEST` authorizer, `identity_source` defines the key — and if it includes
     something high-cardinality (a request path, a correlation header) **the
     cache hit rate is approximately zero** and you have paid for a cache that
     does nothing. Check your `identity_source` before trusting the cache.
2. **Provisioned concurrency on the authorizer only.** Authorizers are small,
   uniform and predictable, which makes them the *best* candidate for provisioned
   concurrency in the whole estate — and the cheapest, because you need far fewer
   units than for the business functions. See the quantified argument in
   [[aws-lambda]].
3. **Prefer a native JWT authorizer where possible.** HTTP APIs have a built-in
   JWT authorizer that API Gateway executes itself — **no Lambda, no cold start,
   no concurrency consumption, no extra cost**. If the estate authenticates with
   OIDC/Cognito JWTs and is on (or can move to) HTTP APIs, this deletes the
   problem rather than mitigating it. That is the lazy answer and it is the right
   one where it applies.
4. **Cognito user pool authorizers** are also executed by API Gateway, but note
   that **a Cognito user pool is itself a regional resource with no native
   cross-region replication** — mirroring user pools is a much larger problem
   than mirroring an authorizer, and it deserves its own note. Flagged in
   [[#Open questions]]; link target [[aws-cognito]].

Quota note: **10 authorizers per API** (Lambda and Cognito), increasable by
support request.

### IAM (SigV4) authorization

The cleanest option for service-to-service and internal callers, because IAM is
global ([[aws-iam]]) — the same principal, the same policy, works in both
regions. **Watch the resource ARNs in the policy**: an IAM policy granting
`execute-api:Invoke` on
`arn:aws:execute-api:eu-west-1:123456789012:abc123/prod/GET/*` will **not**
authorise the standby, which has a different region *and* a different API ID.
Either wildcard the region and API ID, or emit both ARNs. This is exactly the
"ARNs embedded in policies" gotcha the brief calls out, and it is a silent 403 at
failover.

## Resource policies, WAF, VPC links, mTLS, stage variables

### Resource policies

REST-API-only. A resource policy is regional, but its `Resource` elements embed
`arn:aws:execute-api:<region>:<account>:<api-id>/...`. **Never hand-write these
per region** — build them in Terraform from the region's own API ID:

```hcl
data "aws_iam_policy_document" "api_resource_policy" {
  statement {
    effect    = "Allow"
    actions   = ["execute-api:Invoke"]
    resources = ["${aws_api_gateway_rest_api.this.execution_arn}/*"]   # region-correct by construction

    principals {
      type        = "AWS"
      identifiers = ["*"]
    }

    condition {
      test     = "IpAddress"
      variable = "aws:SourceIp"
      values   = var.partner_allowlist_cidrs   # the SAME list in both regions
    }
  }
}
```

The gotcha is the *allow-list content*, not the syntax. If the policy allows a
list of partner egress IPs, that list must be identical in both regions — and it
is exactly the kind of thing that gets patched by hand in the primary during an
incident and never back-ported. **Make the CIDR list a single variable consumed
by both regional module calls.** Resource policy length quota: **8,192
characters**, increasable — which is also AWS's stated reason to prefer WAF for
IP rules.

### WAF

`REGIONAL`-scope web ACLs are regional and associate with a **stage ARN**
(`arn:aws:apigateway:<region>::/restapis/<api-id>/stages/<stage>`). Everything in
[[aws-alb-nlb]]'s WAF section applies: **mirroring the ACL is easy, keeping the
rules in sync is the hard part**, and IP sets are the usual drift source
(`aws_wafv2_ip_set` is regional too).

Two API-Gateway-specific points:

- **HTTP APIs and WebSocket APIs cannot have a WAF web ACL attached.** If WAF is
  a compliance requirement and the estate is on HTTP APIs, you must front them
  with **CloudFront + a `CLOUDFRONT`-scope web ACL in `us-east-1`** — which drags
  the whole `us-east-1` dependency back in ([[aws-acm]]). This is a real
  constraint that changes the architecture; discover it before, not after.
- Rate-based WAF rules **count per region**, like usage plan quotas. A 2,000
  req/5min rate rule becomes effectively 4,000 across a pair. Same reasoning,
  same verdict: it is a protection mechanism, not a contract.

### VPC links pointing at a regional NLB

A v1 VPC link targets an **internal NLB**, and per AWS's guidance and the console
behaviour the **NLB must be in the same Region and the same AWS account as the
API**. VPC peering does not help; there is no cross-region VPC link.

So the standby needs:

- Its own internal NLB ([[aws-alb-nlb]]),
- Its own `aws_api_gateway_vpc_link` (v1) or `aws_apigatewayv2_vpc_link` (v2),
- Its own integration `uri` and `connection_id` in the standby API.

Timing notes that matter for the RTO:

- **VPC link creation is slow** — minutes, and the control-plane API is rate
  limited to **1 `CreateVpcLink` every 15 seconds per account**. It must be
  pre-provisioned. It is not something you create at failover.
- **VPC links per account per Region: 20**, increasable. A pair doubles your
  consumption across the estate — check headroom in the standby as part of the
  quota audit above.
- The standby NLB must have **registered, healthy targets** for the integration
  to work at all. If the warm-standby posture is "pods at zero replicas", the
  VPC link resolves to an NLB with an empty target group and every request
  returns 5xx. This is the same trap as Global Accelerator's health requirement
  in [[aws-alb-nlb]] and is another argument for the "run the pods, disable the
  subscriptions" shape recommended in [[messaging-in-flight-data-loss]].

### Mutual TLS

From the [mTLS doc](https://docs.aws.amazon.com/apigateway/latest/developerguide/rest-api-mutual-tls.html), the prerequisites are verbatim:

> + A Regional custom domain name
> + At least one certificate configured in AWS Certificate Manager for your custom domain name
> + A truststore configured and uploaded to Amazon S3

and:

> To configure mutual TLS for a REST API, you must use a Regional custom domain
> name for your API, with a `TLS_1_2` security policy.

Three multi-region consequences:

1. **mTLS *requires* a regional custom domain.** So if mTLS is in use today, the
   estate is already regional and the edge-optimized migration problem does not
   exist. That is worth checking first — it may make
   [[#Migration path from single-region]] much shorter.
2. **The truststore is an `s3://bucket/key` URI.** The AWS docs do not state a
   same-region requirement and the CLI example passes a plain bucket name, but
   **do not build a failover on an unverified cross-region read**: put a copy of
   the truststore in a bucket in the standby region, keep it in sync with S3
   Cross-Region Replication ([[aws-s3]]), and point the standby domain at the
   local copy. It removes a cross-region data-plane dependency from the TLS
   handshake path for the cost of one replicated object. Verify the actual
   requirement before assuming either way — noted in [[#Open questions]].
3. **Truststore versions are per-domain-name configuration.** Updating the
   truststore is a two-step (upload new S3 version, then patch
   `truststoreVersion` on the domain) and **must be done in both regions**. AWS
   warns: *"API Gateway produces certificate warnings only when you update your
   domain name. API Gateway doesn't notify you if a previously uploaded
   certificate expires."* A standby whose truststore was last updated two years
   ago will reject clients whose CA has since rotated — and, like the ACM
   expiry problem in [[aws-acm]], **nothing will tell you**.

Also note, for imported / Private-CA certificates: the
`ownershipVerificationCertificate` must stay valid *forever* —
*"If a certificate expires and auto-renew fails, all updates to the domain name
will be locked."* A locked domain name in the standby is a domain name you cannot
reconfigure during an incident.

### Stage variables

Stage variables are the clean way to keep **one API definition, two regions**.
Quotas: **100 per stage**, key ≤ 64 chars, value ≤ 512 chars.

Use them for the things that genuinely differ per region — the Lambda alias or
function name, the VPC link ID, the internal NLB hostname:

```hcl
resource "aws_api_gateway_stage" "this" {
  rest_api_id   = aws_api_gateway_rest_api.this.id
  deployment_id = aws_api_gateway_deployment.this.id
  stage_name    = var.stage_name

  variables = {
    region        = data.aws_region.current.name
    backendHost   = var.internal_nlb_dns_name     # regional
    lambdaAlias   = var.lambda_alias              # e.g. "live"
  }
}
```

Two traps:

- **Stage variables in an integration URI are resolved at request time, and a
  missing variable is a 500**, not a config-time error. A stage variable present
  in the primary and absent in the standby is a runtime outage discovered at
  failover. Enforce the *set of keys* in the module, not per-environment.
- **Do not put secrets in stage variables.** They are visible to anyone with
  `apigateway:GET` on the stage. Use [[aws-secrets-manager]].

## The alternative: skip API Gateway custom domains entirely

A genuinely different architecture, and worth considering rather than defaulting.

### Option A — your own CloudFront distribution in front of two regional APIs

One CloudFront distribution, one `us-east-1` viewer certificate, two origins (the
two regions' `execute-api` or regional custom-domain hostnames), and an **origin
group** for automatic failover.

**The blocker, verbatim from the
[origin failover doc](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/high_availability_origin_failover.html):**

> CloudFront fails over to the secondary origin only when the HTTP method of the
> viewer request is `GET`, `HEAD`, or `OPTIONS`. CloudFront does not fail over
> when the viewer sends a different HTTP method (for example `POST`, `PUT`, and
> so on).

> [!danger] Origin groups do not fail over writes
> For a **read-heavy public API** this is fine. For **any API that takes POSTs**
> — which is to say, almost any product API — origin-group failover covers your
> reads and silently does nothing for your writes. You would fail over the GET
> traffic automatically and still need a manual mechanism for everything else,
> which is the worst of both worlds: two failover mechanisms with different
> trigger conditions.

Also relevant: the default is *"3 connection attempts of 10 seconds each"* before
failing over — **30 seconds of latency added to every failing request** unless
you tune `connection_attempts` (1–3) and `connection_timeout` (1–10 s). And note
`origin_read_timeout` is 1–120 s, above API Gateway's own **29-second integration
timeout**, so a slow backend produces a 504 from API Gateway before CloudFront
ever considers failing over.

**Where CloudFront in front genuinely wins:** you need edge caching, you need a
`CLOUDFRONT`-scope WAF on an HTTP API that can't have a regional one, you need
CloudFront Functions for request normalisation, or you want the origin *hostname*
to be an implementation detail the client never sees. Use it for those reasons —
and drive failover with **Route 53 in front of it or by updating the origin**,
not with an origin group.

### Option B — Global Accelerator in front of two regional APIs

[[aws-alb-nlb]] recommends Global Accelerator for the ALB-fronted entry point and
the argument mostly transfers: two static anycast IPs, a traffic dial as a
manual/gradual switch, a `us-west-2` control plane that survives a `us-east-1`
event, and **no DNS TTL problem at all**.

**But check endpoint support before designing on it.** Global Accelerator's
standard endpoint types are ALB, NLB, EC2 instances and Elastic IPs — **API
Gateway is not in that list.** Fronting API Gateway with GA therefore means
either a custom-routing/ALB indirection or putting the API behind an ALB, which
defeats most of the point. **Verify current GA support for API Gateway endpoints
against the [AWS docs](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoints.html)
before proposing it** — do not assume from the ALB note. Flagged in
[[#Open questions]].

### Recommendation

**Route 53 failover records over two `REGIONAL` custom domain names.** It is the
pattern AWS documents, the pattern its own samples use, it needs no extra service,
it costs nothing while idle, and it works identically for REST, HTTP and
WebSocket APIs.

**Add your own CloudFront distribution only if you need caching, edge compute or
a `CLOUDFRONT`-scope WAF** — and if you do, keep Route 53 as the failover
mechanism and treat the origin group as a bonus for idempotent reads, not as the
plan.

## RPO / RTO analysis

**RPO: N/A.** API Gateway holds no customer data. The only stateful object is the
usage-plan quota counter, which does not replicate and does not need to (see
above). If API caching is enabled, the standby's cache is cold — that is a
latency event, not a data-loss event.

**RTO: the gateway itself contributes ~0 seconds, if pre-provisioned.**

| Phase | Time | Pre-provisioned? |
|---|---|---|
| Standby API, stages, deployment exist | 0 s | **Yes — must be** |
| Standby regional custom domain + in-region ACM cert exist | 0 s | **Yes — must be** ([[aws-acm]]) |
| Base path / API mappings exist | 0 s | **Yes — must be** |
| VPC link exists and its NLB has healthy targets | 0 s | **Yes — must be** (creation takes minutes) |
| API keys exist **with matching values** | 0 s | **Yes — must be** |
| WAF ACL associated to the standby stage | 0 s | **Yes** |
| Route 53 record flip + TTL | **~90 s** | See [[aws-route53]] |
| Authorizer cold-start storm | **~1–5 s of elevated p99, or minutes if the authorizer is heavy** | Mitigate with cache TTL + provisioned concurrency |
| **Account throttle quota sufficient for 100% of traffic** | **0 s — or total failure** | **Yes — must be, and is the thing that isn't** |

If instead you create things at failover time:

| Phase | Time |
|---|---|
| `CreateDomainName` | rate-limited to **1 request every 30 seconds per account**, then minutes to provision |
| `CreateVpcLink` | rate-limited to **1 every 15 seconds**, then minutes |
| `CreateDeployment` | **1 request every 5 seconds per account** |
| Request an ACM cert | up to 30 min, hard timeout 72 h ([[aws-acm]]) |
| Request a quota increase | **hours to days, via humans** |

**Every one of those blows the 15-minute RTO, and the rate limits mean a
"terraform apply the standby at failover" plan does not merely take a while — it
gets throttled by the control plane while you watch.** Pre-provision everything.

## Warm standby shape

While the primary is healthy, the standby region holds, and costs:

| Object | State | Idle cost |
|---|---|---|
| REST/HTTP/WebSocket API + stages | Deployed, current | **$0** |
| Regional custom domain name | Created, certificate attached | **$0** |
| ACM certificate | `ISSUED`, attached (which is what keeps it renewing — [[aws-acm]]) | **$0** |
| API keys + usage plans | Created, values matching the primary | **$0** |
| WAF web ACL + rules | Created and associated | **Not $0** — WAF bills per web ACL, per rule and per request; see [[aws-waf-shield]] |
| VPC link + internal NLB | Provisioned, targets healthy | **NLB hourly + LCU** ([[aws-alb-nlb]]) |
| Lambda authorizer | Deployed; provisioned concurrency optional | **$0**, or PC cost — [[aws-lambda]] |
| **API caching** | **Off** | Caching bills **hourly whether used or not** (e.g. ~$0.038/hr for the 1.6 GB tier, US East) — **do not enable it in the standby.** |
| Route 53 records + health checks | Pre-created | ~$0.50–1.50/mo per health check |

**API Gateway itself is one of the cheapest things to keep warm in the entire
estate: the request-priced tiers mean an idle API costs nothing.** The money is
in the NLB, the WAF and — if you enable it — the cache.

> [!warning] Do not enable API caching in the standby "to be ready"
> Cache is billed per hour by size, not per request, so a warm cache in a passive
> region is pure waste. And a *cold* cache at failover is only a latency blip.
> Enable it at failover if you must — it is a stage setting, not a rebuild — or
> just accept the cold-cache latency for the first few minutes.

## Terraform implementation

### Module shape for a templated monorepo

The cookiecutter-friendly shape is **one `api` module, invoked once per region
with a different provider**, plus **one `dns` invocation that consumes both
regions' outputs**. That keeps the module free of any notion of "primary" or
"standby" — which matters, because after a failback-as-permanent-swap
([[split-brain-and-fencing]]) those labels move.

```hcl
terraform {
  required_version = ">= 1.9"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.0" }
  }
}

provider "aws" {
  region = var.primary_region
  default_tags { tags = local.common_tags }
}

provider "aws" {
  alias  = "standby"
  region = var.standby_region
  default_tags { tags = local.common_tags }
}
```

### `modules/apigw-regional/main.tf` — the region-agnostic half

```hcl
variable "domain_name"      { type = string }                       # api.example.com — identical in both regions
variable "certificate_arn"  { type = string }                       # MUST be an ACM cert in THIS region
variable "stage_name"       { type = string }
variable "openapi_body"     { type = string }                       # rendered OpenAPI, region-neutral
variable "vpc_link_target_arns" {
  type        = list(string)
  default     = []
  description = "Internal NLB ARN(s) in THIS region. Empty = no private integration."
}

data "aws_region" "current" {}

resource "aws_api_gateway_rest_api" "this" {
  name = "${var.stage_name}-api"
  body = var.openapi_body

  endpoint_configuration {
    types = ["REGIONAL"]      # NEVER "EDGE". See the note above.
  }
}

# Redeploy whenever the body changes. The sha1 trigger is the standard idiom;
# without it Terraform updates the RestApi and never deploys it, so the change
# is invisible at runtime — a classic "it applied but nothing happened".
resource "aws_api_gateway_deployment" "this" {
  rest_api_id = aws_api_gateway_rest_api.this.id
  triggers = {
    redeployment = sha1(var.openapi_body)
  }
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_api_gateway_stage" "this" {
  rest_api_id   = aws_api_gateway_rest_api.this.id
  deployment_id = aws_api_gateway_deployment.this.id
  stage_name    = var.stage_name

  variables = {
    region = data.aws_region.current.name
  }
}

# Per-stage throttling. This is YOUR ceiling, below the account ceiling.
# Keep it IDENTICAL in both regions, or the standby quietly serves less.
resource "aws_api_gateway_method_settings" "all" {
  rest_api_id = aws_api_gateway_rest_api.this.id
  stage_name  = aws_api_gateway_stage.this.stage_name
  method_path = "*/*"

  settings {
    throttling_rate_limit  = var.stage_throttle_rate
    throttling_burst_limit = var.stage_throttle_burst
    metrics_enabled        = true
    logging_level          = "INFO"
  }
}

# ---- the custom domain: REGIONAL, in-region certificate ----
resource "aws_api_gateway_domain_name" "this" {
  domain_name              = var.domain_name
  regional_certificate_arn = var.certificate_arn   # NOT certificate_arn — that field is the EDGE one
  security_policy          = "TLS_1_2"

  endpoint_configuration {
    types = ["REGIONAL"]
  }
}

resource "aws_api_gateway_base_path_mapping" "this" {
  api_id      = aws_api_gateway_rest_api.this.id
  stage_name  = aws_api_gateway_stage.this.stage_name
  domain_name = aws_api_gateway_domain_name.this.domain_name
  # base_path omitted == the root mapping
}

# ---- private integration, if used ----
resource "aws_api_gateway_vpc_link" "this" {
  count       = length(var.vpc_link_target_arns) > 0 ? 1 : 0
  name        = "${var.stage_name}-vpclink-${data.aws_region.current.name}"
  target_arns = var.vpc_link_target_arns    # internal NLB, SAME region, SAME account
}

# ---- what the DNS layer needs ----
output "regional_domain_name" {
  description = "d-xxxxxxxxxx.execute-api.<region>.amazonaws.com — the Route 53 alias TARGET."
  value       = aws_api_gateway_domain_name.this.regional_domain_name
}

output "regional_zone_id" {
  description = "AWS-owned hosted zone ID for the alias target. NOT your zone."
  value       = aws_api_gateway_domain_name.this.regional_zone_id
}

output "execution_arn" {
  value = aws_api_gateway_rest_api.this.execution_arn
}

output "stage_arn" {
  description = "For WAF association."
  value       = aws_api_gateway_stage.this.arn
}
```

### Calling it for a pair, and wiring Route 53

```hcl
module "api_primary" {
  source = "../../modules/apigw-regional"

  domain_name          = "api.${var.public_domain}"
  certificate_arn      = module.cert_primary.certificate_arn   # see aws-acm
  stage_name           = var.env
  openapi_body         = local.openapi_body
  vpc_link_target_arns = [module.nlb_primary.arn]

  providers = { aws = aws }
}

module "api_standby" {
  source = "../../modules/apigw-regional"

  # IDENTICAL domain name. This is legal, and it is the whole pattern.
  domain_name          = "api.${var.public_domain}"
  certificate_arn      = module.cert_standby.certificate_arn   # cert in the STANDBY region
  stage_name           = var.env
  openapi_body         = local.openapi_body                    # same definition, byte for byte
  vpc_link_target_arns = [module.nlb_standby.arn]

  providers = { aws = aws.standby }
}

data "aws_route53_zone" "public" {
  name         = var.public_domain
  private_zone = false
}

# ---------- Route 53 failover over the two regional domain names ----------

resource "aws_route53_health_check" "primary_api" {
  fqdn              = module.api_primary.regional_domain_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/${var.env}/health"
  failure_threshold = 3
  request_interval  = 10
  tags              = { Name = "apigw-primary-${var.env}" }
}

resource "aws_route53_record" "api_primary" {
  zone_id        = data.aws_route53_zone.public.zone_id
  name           = "api.${var.public_domain}"
  type           = "A"
  set_identifier = "primary-${var.primary_region}"

  failover_routing_policy { type = "PRIMARY" }
  health_check_id = aws_route53_health_check.primary_api.id

  alias {
    name    = module.api_primary.regional_domain_name
    zone_id = module.api_primary.regional_zone_id
    # API Gateway regional domains do NOT support target health evaluation.
    # Leave this false and use the explicit health check above.
    evaluate_target_health = false
  }
}

resource "aws_route53_record" "api_secondary" {
  zone_id        = data.aws_route53_zone.public.zone_id
  name           = "api.${var.public_domain}"
  type           = "A"
  set_identifier = "secondary-${var.standby_region}"

  failover_routing_policy { type = "SECONDARY" }

  alias {
    name                   = module.api_standby.regional_domain_name
    zone_id                = module.api_standby.regional_zone_id
    evaluate_target_health = false
  }
}
```

> [!tip] Prefer the manual switch over automatic failover
> The records above are **automatic** — Route 53 will move production traffic on
> health-check opinion alone. [[aws-route53]] argues at length that this is wrong
> for a pair whose database is a 2-hour-lagging async replica. Swap
> `health_check_id` for an **ARC routing control health check**
> (`type = "RECOVERY_CONTROL"`) and the same records become a human-operated,
> data-plane-only switch. AWS's own sample,
> [`aws-samples/apigw-multi-region-failover`](https://github.com/aws-samples/apigw-multi-region-failover),
> is built exactly this way — API Gateway regional endpoints, Route 53 records,
> ARC routing controls per service — and is worth reading before writing your
> own. Note its stated cost is **approximately $1,900/month across both
> regions**, essentially all of it the ARC cluster.

### HTTP API (v2) equivalent

The v2 resources differ enough to be worth writing out:

```hcl
resource "aws_apigatewayv2_domain_name" "this" {
  domain_name = var.domain_name

  domain_name_configuration {
    certificate_arn = var.certificate_arn          # in-region
    endpoint_type   = "REGIONAL"                   # the ONLY valid value for HTTP APIs
    security_policy = "TLS_1_2"                    # required for HTTP API mappings
  }
}

resource "aws_apigatewayv2_api_mapping" "this" {
  api_id      = aws_apigatewayv2_api.this.id
  domain_name = aws_apigatewayv2_domain_name.this.id
  stage       = aws_apigatewayv2_stage.this.id
}

output "target_domain_name" {
  value = aws_apigatewayv2_domain_name.this.domain_name_configuration[0].target_domain_name
}

output "target_zone_id" {
  value = aws_apigatewayv2_domain_name.this.domain_name_configuration[0].hosted_zone_id
}
```

The Route 53 records are identical in shape; only the alias `name`/`zone_id`
source changes. The provider has a long history of confusion here — see
[hashicorp/terraform-provider-aws#12892](https://github.com/terraform-providers/terraform-provider-aws/issues/12892)
for the v2/WebSocket variant of the question.

## Migration path from single-region

Assume the live region has an **edge-optimized** REST API with
`api.example.com` — the default and therefore the most likely state.

**The good news: AWS supports adding a `REGIONAL` endpoint configuration to an
existing custom domain name alongside the `EDGE` one, so the migration is
additive and does not require deleting the name.** From the
[migration doc](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-regional-api-custom-domain-migrate.html):

> You first add the new endpoint configuration type to the existing
> `endpointConfiguration.types` list for the custom domain name. Next, you set up
> a DNS record to point the custom domain name to the newly provisioned endpoint.
> Finally, you remove the obsolete custom domain name endpoint.

and the timing:

> It might take up to 60 seconds to complete a migration between an
> edge-optimized custom domain name and a Regional custom domain name. The
> migration time also depends on when you update your DNS records.

### Sequence

1. **Inventory.** For each custom domain: endpoint type, certificate ARN and
   region, security policy, mTLS on/off, base path mappings. `aws apigateway
   get-domain-names --region <primary>`. Also `get-api-keys --include-values`
   (privileged — do this deliberately, not casually) and `get-usage-plans`.
2. **Certificates.** Request an ACM certificate for the domain **in the primary
   region** (you currently only have the `us-east-1` one) and **in the standby
   region**. Both validate against the existing CNAME — no DNS change
   ([[aws-acm]]).
3. **Add the regional endpoint to the existing domain in the primary.** In
   Terraform this is adding `regional_certificate_arn` and `REGIONAL` to
   `endpoint_configuration.types` on the existing `aws_api_gateway_domain_name`.
   **Check the plan says `~ update in-place`, not `-/+ replacement`.** If it says
   replacement, stop — a replacement deletes a live custom domain name and is a
   hard outage. Fall back to the CLI `update-domain-name --patch-operations`
   shown in the AWS doc and then `terraform import` / refresh to reconcile.
   During this phase the domain has **both** endpoints and both work.
4. **Repoint DNS from the `distributionDomainName` alias to the
   `regional_domain_name` alias**, at a low TTL, during business hours. Watch
   `Count`/`4XXError`/`5XXError` in CloudWatch for the API. **This is the only
   customer-affecting moment and it is a TTL-bounded cutover, not an outage.**
5. **Soak for at least a day.** Latency will rise for geographically distant
   clients — that is the CloudFront edge you just gave up. If that is
   unacceptable, this is the point where you decide to bring your **own**
   CloudFront distribution back in front (above), which you can do without
   undoing anything.
6. **Remove the `EDGE` endpoint configuration.** AWS: *"To complete the
   migration, make sure that you remove the obsolete endpoint from your custom
   domain name."* The `us-east-1` certificate can then be retired — but see
   [[aws-acm]] gotcha #2 before deleting anything.
7. **Now, and only now, deploy the standby.** `module "api_standby"` with the
   same domain name and the standby certificate. This step touches nothing in the
   primary and is safe any time.
8. **Mirror API keys** (matching values), usage plans, WAF ACL + IP sets,
   resource policy, VPC link + NLB, authorizer Lambda.
9. **Add the Route 53 secondary record** with weight 0 / SECONDARY. No traffic
   moves.
10. **Audit quota parity** (script above) and **raise the standby's account
    throttle quota to match the primary**. For the CA pair, budget for this
    explicitly — `ca-west-1` starts at 2,500 RPS.
11. **Load-test the standby at production peak volume.** Not a smoke test. This
    is the only thing that proves step 10 worked.

### What forces replacement

| Change | Effect |
|---|---|
| `aws_api_gateway_rest_api.endpoint_configuration.types` EDGE→REGIONAL **on the API** | Update in place; but the `execute-api` hostname changes. Anything hard-coding it breaks. |
| `aws_api_gateway_domain_name.domain_name` | **ForceNew.** Never change it. |
| `aws_api_gateway_domain_name` security policy / endpoint type | In-place, but see the AWS note that once two endpoint configurations exist you **can't change the endpoint access mode**. |
| `aws_api_gateway_api_key.value` | **ForceNew, and it revokes the customer's key.** Guard with `prevent_destroy`. |
| `aws_api_gateway_vpc_link.target_arns` | **ForceNew** — replacing a VPC link takes minutes and breaks private integrations while it happens. |
| `aws_acm_certificate` SAN changes | ForceNew — see [[aws-acm]]'s `create_before_destroy` warning. |

## Failover procedure

API Gateway's step is **nothing**, and that is the goal. The full sequence:

1. **Decide** (human — this is the real RTO risk, see [[aws-route53]]).
2. **Promote the data layer** ([[aws-rds-postgres]] / [[aws-dynamodb]]) — this is
   irreversible, do it last among the irreversible things
   ([[split-brain-and-fencing]]).
3. **Enable the standby's consumers** (Lambda ESMs / pollers — [[aws-lambda]],
   [[messaging-in-flight-data-loss]]).
4. **Flip the switch**: ARC routing control, or the Route 53 weight/failover
   record. ~90 s including TTL.
5. **Watch `4XXError` and `5XXError` on the standby API, and specifically watch
   for `429`.** A spike in 429s means you hit the account throttle quota and the
   failover is failing in the one way that looks like success. **This alarm
   should already exist and should be on the standby's dashboard before you
   start.**
6. **Watch the authorizer's `Duration` p99 and Lambda `Throttles`.** Cold-start
   storm is expected and should decay within seconds once the authorizer cache
   fills.
7. **Do not `terraform apply` during a failover.** `CreateDeployment` is rate
   limited to one per five seconds per account and `CreateDomainName` to one per
   thirty; a pipeline that touches API Gateway during an incident will be
   throttled by the control plane.

**Pre-flight checks that belong in [[failover-runbooks]]**, all runnable weekly:

```bash
# 1. Both regional custom domains exist and are AVAILABLE
for r in eu-west-1 eu-west-2; do
  aws apigateway get-domain-names --region "$r" \
    --query "items[?domainName=='api.example.com'].[domainName,domainNameStatus,regionalDomainName,endpointConfiguration.types]" \
    --output text
done

# 2. API key VALUES match across the pair (hash only — never log the value)
for r in eu-west-1 eu-west-2; do
  echo -n "$r "
  aws apigateway get-api-keys --region "$r" --include-values \
    --query 'items[].[name,value]' --output text | sort | sha256sum
done   # the two hashes MUST be identical

# 3. Account throttle quota parity
for r in eu-west-1 eu-west-2; do
  echo -n "$r "
  aws service-quotas get-service-quota --region "$r" \
    --service-code apigateway --quota-code L-8A5B8E43 \
    --query 'Quota.Value' --output text 2>/dev/null \
    || aws service-quotas get-aws-default-service-quota --region "$r" \
         --service-code apigateway --quota-code L-8A5B8E43 \
         --query 'Quota.Value' --output text
done
```

## Failback

Easier than most services, because nothing in API Gateway was promoted or
mutated — both regions were always fully configured and one of them simply had no
traffic.

Failback is: flip the routing control / Route 53 record back. ~90 seconds.

The things that are *not* trivial:

- **The old primary's stages may be behind.** If deployments continued during the
  incident (hotfixes shipped into the live standby), the recovered region is
  running older code. **Re-run the deployment pipeline against the recovered
  region and verify the deployment ID before shifting traffic back.** A
  `terraform plan` showing an empty diff is not enough — `aws_api_gateway_deployment`
  can be up to date in state while the *stage* points at an older deployment.
- **The usage plan counters diverged.** Nothing to do; see above.
- **Consider not failing back at all.** [[split-brain-and-fencing]] argues for
  "permanent swap" over "ping-pong". API Gateway is one of the services that
  makes permanent swap cheap, because both regions are genuinely symmetric —
  there is no "primary" attribute anywhere in the configuration. Keep the module
  symmetric (as written above) so the swap is a variable change, not a refactor.
- **If you used the ARC/weighted manual switch, disable automatic failback.**
  Otherwise a recovering-then-flapping primary drags traffic back and forth
  across a pair whose databases have now diverged.

## Gotchas

1. **Edge-optimized custom domain names are globally unique.** You cannot create
   the same one in two regions. This is the blocking fact and it is the reason
   this note exists.
2. **`certificate_arn` vs `regional_certificate_arn` on
   `aws_api_gateway_domain_name`.** The first is the edge-optimized field and
   expects a `us-east-1` ARN; the second is the regional one. Setting the wrong
   one produces a confusing `BadRequestException` about the certificate region.
3. **`regional_zone_id` is AWS's zone, not yours.** Swapping it with your hosted
   zone ID is the classic first-attempt failure.
4. **`evaluate_target_health` doesn't work for API Gateway regional domains.**
   Unlike an ALB alias ([[aws-route53]] Option 1), you need an explicit
   `aws_route53_health_check`. Budget $0.50–$1.50/month per check and remember a
   **newly created health check starts healthy**.
5. **API keys do not replicate and their values are immutable.** The failure is a
   customer-facing 403 at the worst moment. Import matching values from a
   replicated secret.
6. **Usage-plan quota counters reset on failover.** A monthly quota is
   effectively doubled in a failover month.
7. **Account-level throttle quotas are per-region, invisible to Terraform, and
   `ca-west-1` starts at one quarter of `ca-central-1`.** The biggest item in
   this note.
8. **The account throttle quota is shared across REST, HTTP, WebSocket *and*
   WebSocket callback APIs.** A chatty WebSocket fan-out consumes the same bucket
   as your REST traffic. If both land in the standby at once, they compete.
9. **`aws_api_gateway_deployment` without a `triggers` block never redeploys.**
   The API definition updates, the stage keeps serving the old deployment, and
   the two regions drift apart invisibly. Always set
   `triggers = { redeployment = sha1(...) }` and `create_before_destroy`.
10. **Control-plane API rate limits are brutal and per-account (not per-region).**
    `CreateDeployment` 1 per 5 s; `CreateDomainName` / `UpdateDomainName` /
    `DeleteDomainName` 1 per 30 s; `CreateVpcLink` 1 per 15 s; total operations
    **10/s with a burst of 40**. A parallel `terraform apply` across two regions
    with many APIs **will** hit these and produce flaky `TooManyRequestsException`
    failures. Consider `-parallelism` tuning in the pipeline.
11. **A VPC link's NLB must be in the same region *and account*.** No peering, no
    cross-region. The standby needs its own NLB with healthy targets.
12. **HTTP APIs and WebSocket APIs cannot have a WAF web ACL attached.** If WAF
    is mandated, you need CloudFront in front — which means a `us-east-1` ACM
    cert and a `CLOUDFRONT`-scope ACL.
13. **CloudFront origin groups do not fail over POST/PUT/DELETE.** Reads fail
    over, writes don't. Never present an origin group as "the failover plan" for
    a write API.
14. **mTLS truststore drift.** API Gateway warns about bad certificates *only*
    when you update the domain name and *never* when one expires. Two regions
    means two truststores to keep current.
15. **IAM policies and resource policies embed region + API ID.** A
    `execute-api:Invoke` grant scoped to the primary's ARN silently 403s in the
    standby.
16. **A missing stage variable is a runtime 500, not a plan error.** Enforce the
    key set in the module.
17. **`x-amazon-apigateway-integration` URIs in an imported OpenAPI body are full
    ARNs.** If the OpenAPI document is checked in with hard-coded
    `arn:aws:apigateway:eu-west-1:lambda:path/...`, importing it into `eu-west-2`
    creates an API that invokes the **primary region's Lambda**. It will *work*,
    which is worse than failing — you will have a standby that depends on the
    dead region. **Template the region into the OpenAPI body.** This is the
    nastiest silent failure in this note after the throttle quota.
18. **API caching is billed hourly by size.** Never leave it on in a passive
    region.

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Endpoint type | **A: `REGIONAL`** in both regions, Route 53 in front. | B: `EDGE`. | **A, and it is not a choice** — B physically cannot produce two custom domains with one name. |
| Failover trigger | A: Route 53 failover records + endpoint health checks. Automatic, ~$1/mo, no control plane needed. | B: ARC routing controls as `RECOVERY_CONTROL` health checks. Human-decided, safety rules, data plane outside `us-east-1`. ~$1,825/mo per cluster. | **B if the ARC cluster is being bought anyway for the rest of the estate** ([[aws-route53]]) — one cluster covers all three pairs. **A as the floor**, because an automatic failover into a 2-hour-lagging replica is a data-loss event triggered by a network blip. |
| Front with your own CloudFront? | A: No — Route 53 straight to the two regional domains. Simplest, $0, no `us-east-1` dependency. | B: Yes — caching, edge compute, `CLOUDFRONT`-scope WAF, one hostname the client never sees change. Adds a `us-east-1` cert and control-plane dependency. | **A**, unless you need WAF on an HTTP API, edge caching, or CloudFront Functions. If you take B, **still fail over with Route 53**, not with an origin group. |
| API key strategy | A: Import matching values from a replicated Secrets Manager secret into both regions. | B: Drop API keys entirely; move to a JWT/Lambda authorizer with a replicated identity store. | **A now** (it is ~20 lines and fixes a customer-visible bug), **B eventually** — API keys are a weak authentication mechanism and their regional behaviour is a permanent tax. |
| Authorizer | A: Lambda authorizer with `authorizer_result_ttl_in_seconds = 300` + provisioned concurrency in both regions. | B: Native JWT authorizer (HTTP APIs) — executed by API Gateway, no cold start, no concurrency, no cost. | **B where the API type and auth model allow it.** It deletes the failover cold-start problem instead of paying to mitigate it. A otherwise. |
| Quota parity enforcement | A: A weekly audit script + an alarm. | B: `aws_servicequotas_service_quota` in Terraform for both regions from one variable. | **Both.** B makes drift a plan diff; A catches the quotas Service Quotas doesn't model. **Do not rely on memory.** |
| API caching in the standby | A: Off. | B: On, sized as primary. | **A.** Billed hourly whether used or not; a cold cache costs you minutes of latency, not availability. |

## Cost

| Item | Standby idle cost |
|---|---|
| REST / HTTP / WebSocket API, stages, deployments | **$0** — request-priced |
| Regional custom domain name | **$0** |
| ACM certificate | **$0** ([[aws-acm]]) |
| API keys, usage plans | **$0** |
| Route 53 health check (fast interval) | ~$1.00–1.50/mo each |
| WAF web ACL + rules | **Real** — per ACL, per rule, per request. See [[aws-waf-shield]] |
| VPC link's internal NLB | **Real** — hourly + LCU. See [[aws-alb-nlb]] |
| API caching | **Real if enabled** (e.g. ~$0.038/hr for the 1.6 GB tier, US East) — **leave it off** |
| Lambda authorizer provisioned concurrency | **Real** — see [[aws-lambda]] |
| ARC routing control cluster, if chosen | **~$1,825/mo**, shareable across all three pairs |

**API Gateway is close to free to keep warm.** The cost levers are the NLB, the
WAF and — if chosen — ARC. At failover the standby's request charges simply
replace the primary's; there is no doubling. Feed to [[cost-modelling]].

## Open questions

1. **Are the existing custom domains edge-optimized or regional?** This single
   answer determines whether the migration is a two-week cutover or a no-op. Run
   `aws apigateway get-domain-names --query 'items[].[domainName,endpointConfiguration.types]'`
   in all three primaries. **Do this first.**
2. **Are API keys in use, and by whom?** If external partners hold
   auto-generated keys, this is a customer-communication problem as well as a
   Terraform problem. How many, and can the values be rotated to
   Secrets-Manager-managed values during a maintenance window?
3. **What is each primary's *current* account-level throttle quota?** Not the
   default — the applied value. Until someone runs the audit, the RTO is unknown.
4. **Does `ca-west-1`'s 2,500 RPS default clear `ca-central-1`'s observed peak
   RPS?** If not, the quota increase is a blocker for the CA pair, not a
   nice-to-have. Feed the answer to [[region-pair-selection]].
5. **Is WAF mandated by compliance, and are any APIs HTTP APIs?** If both, the
   architecture must include CloudFront and the `us-east-1` dependency comes
   back. ([[data-residency-compliance]] should also confirm a `us-east-1`
   CloudFront cert is acceptable for the CA deployment.)
6. **Does the mTLS truststore bucket need to be in-region?** AWS docs are silent.
   Plan for a replicated in-region copy regardless.
7. **Are there WebSocket APIs?** If so, does the client reconnect automatically,
   and does the backend resolve the `@connections` endpoint at runtime rather
   than from a baked-in env var?
8. **Is the OpenAPI body checked in with hard-coded integration ARNs?** If yes,
   the standby will silently invoke the primary's Lambdas. Grep for
   `arn:aws:apigateway:` in the repo.
9. **Does Global Accelerator support API Gateway endpoints?** [[aws-alb-nlb]]
   recommends GA for ALB-fronted traffic; whether that recommendation extends
   here needs verifying against the GA endpoint-types doc before anyone assumes
   one uniform entry-point design.
10. **Is a Cognito user pool in the authorization path?** User pools are regional
    with no native replication and would be a separate, larger workstream
    ([[aws-cognito]]).

## Sources

- [API endpoint types for REST APIs in API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-endpoint-types.html) — the load-bearing source. Verbatim: edge-optimized custom domains "apply across all regions"; regional custom domains are per-region and "can have the same custom domain name in all Regions"; and AWS's own recommendation to use a regional endpoint with *your own* CloudFront distribution.
- [Amazon API Gateway quotas](https://docs.aws.amazon.com/apigateway/latest/developerguide/limits.html) — **the `ca-west-1` 2,500 RPS / 1,250 burst finding**, the 10,000 RPS / 5,000 burst default, the statement that burst is not customer-adjustable, and the control-plane API rate limits (`CreateDeployment` 1/5 s, `CreateDomainName` 1/30 s, `CreateVpcLink` 1/15 s, total 10/s burst 40).
- [Throttle requests to your REST APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html) — verbatim "against all APIs in your account, per Region", the four-layer ordering of throttle settings, and the "best-effort, targets not ceilings" caveat.
- [Quotas for configuring and running a REST API](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-execution-service-limits-table.html) — 120 public custom domain names/region, 10,000 API keys/region, 300 usage plans/region, 20 VPC links/region, 10 authorizers/API, 100 stage variables/stage, 8,192-char resource policy, 29 s integration timeout.
- [Migrate a custom domain name to a different API endpoint type](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-regional-api-custom-domain-migrate.html) — the additive EDGE→REGIONAL migration, the "up to 60 seconds" figure, the `update-domain-name --patch-operations` commands, and the endpoint-access-mode constraint.
- [Set up API keys for REST APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-setup-api-key-with-console.html) — verbatim "After you create an API key value, it cannot be changed", and the Custom/import path that makes matching values across regions possible.
- [Require client certificates with mutual TLS](https://docs.aws.amazon.com/apigateway/latest/developerguide/rest-api-mutual-tls.html) — regional custom domain + `TLS_1_2` prerequisite, the truststore S3 URI, the "no notification when a certificate expires" warning, and the `ownershipVerificationCertificate` lock-out failure mode.
- [Custom domain names for HTTP APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-custom-domain-names.html) — HTTP APIs are regional-custom-domain-only and require the TLS 1.2 security policy.
- [Implementing multi-Region failover for Amazon API Gateway (AWS Compute Blog)](https://aws.amazon.com/blogs/compute/implementing-multi-region-failover-for-amazon-api-gateway/) — the per-service-subdomain pattern with independent ARC control panels, so services fail over individually rather than all-or-nothing.
- [`aws-samples/apigw-multi-region-failover`](https://github.com/aws-samples/apigw-multi-region-failover) — working reference implementation of the above; states an approximate cost of **$1,900/month across both regions**, essentially all ARC cluster.
- [Building a multi-Region serverless application with Amazon API Gateway and AWS Lambda (AWS Compute Blog, 13 Nov 2017)](https://aws.amazon.com/blogs/compute/building-a-multi-region-serverless-application-with-amazon-api-gateway-and-aws-lambda) — the original regional-custom-domain + Route 53 health check pattern. Old, and it explicitly does **not** cover data replication.
- [`aws-samples/serverless-samples` — multiregional-private-api](https://github.com/aws-samples/serverless-samples/blob/main/multiregional-private-api) — the private-API variant: VPC endpoints, Transit Gateway inter-region peering, private hosted zones, latency routing with health checks. Relevant if any API is `PRIVATE`.
- [Optimize high availability with CloudFront origin failover](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/high_availability_origin_failover.html) — verbatim the GET/HEAD/OPTIONS-only limitation, the failover status codes, and the default "3 connection attempts of 10 seconds each".
- [Amazon API Gateway pricing](https://aws.amazon.com/api-gateway/pricing/) — REST $3.50/M (first 300M), HTTP $1.00/M, WebSocket $1.00/M messages + $0.25/M connection-minutes, caching billed hourly by size.
- [`aws_api_gateway_domain_name` (Terraform registry)](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/api_gateway_domain_name) — `regional_domain_name` / `regional_zone_id` attribute semantics.
- [`aws-samples/sample-service-quotas-replicator-for-aws`](https://github.com/aws-samples/sample-service-quotas-replicator-for-aws) — an AWS sample whose stated use cases include "Disaster Recovery Planning: Verify that your DR region has the same quota limits as your primary region to support failover scenarios". Evidence that AWS considers quota drift a known DR problem, not a niche one.

### Searched for and did not find

- **No public postmortem or engineering write-up of an API Gateway account-level
  throttle quota shortfall during a real regional failover.** The failure mode is
  well-supported by documentation but I found no first-hand account of it
  happening. Written here as a reasoned risk, not as a cited incident.
- **No published case study from a named company** describing a production
  active/passive API Gateway failover with numbers (observed RTO, error rates).
  The available material is AWS's own blogs and samples. If the team wants
  external validation before committing, it does not exist publicly — the
  `aws-samples` repo is the closest thing to a reference implementation.

## Related notes

[[aws-route53]] · [[aws-acm]] · [[aws-alb-nlb]] · [[aws-lambda]] ·
[[aws-cloudfront]] · [[aws-waf-shield]] · [[aws-secrets-manager]] ·
[[aws-vpc-networking]] · [[aws-iam]] · [[region-pair-selection]] ·
[[failover-runbooks]] · [[split-brain-and-fencing]] · [[cost-modelling]] ·
[[sequencing-roadmap]]
