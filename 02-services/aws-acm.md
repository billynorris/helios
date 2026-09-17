---
title: AWS Certificate Manager (ACM) — Multi-Region
service: acm
tags: [service, multi-region, acm, tls, certificates, edge]
status: researched
replication: manual (re-request per region; validation record is shared)
rpo_achievable: N/A — stateless config, no data
rto_achievable: "< 1 min if pre-provisioned; 30 min – 72 h if created at failover time (FAILS RTO)"
meets_targets: conditional
updated: 2026-09-16
---

# AWS Certificate Manager (ACM) — Multi-Region

## TL;DR

- **ACM certificates are regional and cannot be copied, exported, or shared between regions.** An ACM-issued public certificate's private key is never released to you, so there is no "copy the cert to the standby" option. You must **request a separate certificate in every region** that needs one.
- **This is painless, because of one specific behaviour:** the DNS validation `CNAME` that ACM asks you to publish is *not* per-region and *not* per-certificate. One record in Route 53 validates certificates for that FQDN in **every** region, forever. AWS states this explicitly — see [[#The one insight that makes this painless]].
- **CloudFront certificates must live in `us-east-1`**, no exceptions, regardless of where the distribution's origins are. Same for WAF `CLOUDFRONT`-scope web ACLs. This forces a permanently-aliased `us-east-1` provider into every stack that touches the edge.
- **Certificates must be pre-provisioned in the standby.** Issuance is usually minutes but the console warns it can show `Pending validation` for up to 30 minutes, and ACM will wait up to **72 hours** before timing out. A 15-minute RTO cannot contain "request a cert and wait". This is called out directly in [[research-brief]] as an RTO-killer.
- **The thing that will bite:** deleting the validation `CNAME` after the cert is issued. Everything keeps working for up to 13 months, then **every** certificate for that domain in **every** region silently fails to auto-renew at once. The record must persist forever.

## Does this service cross regions at all?

No. ACM is a **regional** service with a regional endpoint in every region you'd care about, including `ca-west-1` (`acm.ca-west-1.amazonaws.com`, with FIPS endpoints available) — so there is no `ca-west-1` gap here, unlike some other services in this vault. See [[region-pair-selection]].

Three separate facts get conflated and are worth separating:

| Claim | True? | Detail |
|---|---|---|
| A certificate ARN in `eu-west-1` can be attached to an ALB in `eu-west-2` | **No** | The ARN is rejected. ELB only accepts certificate ARNs from its own region. |
| You can export/copy an ACM-issued public certificate to another region | **No** | ACM never releases the private key for ACM-issued public certificates. `ExportCertificate` applies to private CA certs and to the newer exportable public certs, not to the classic ACM-managed public cert flow. |
| You must re-validate domain ownership in each region | **No** | This is the important one. See below. |

The middle row is why `replication: manual` in the frontmatter. But the third row is why "manual" is cheap.

### The one insight that makes this painless

From the [ACM DNS validation documentation](https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html), verbatim:

> Without the need to repeat validation, you can request additional ACM certificates for your fully qualified domain name (FQDN) for as long as the CNAME record remains in place. That is, you can create replacement certificates that have the same domain name, or certificates that cover different subdomains. **Since the CNAME validation token works for any AWS Region, you can re-create the same certificate in multiple Regions.** You can also replace a deleted certificate.

Read that carefully, because it determines the whole Terraform shape:

- The validation record name/value pair is derived from **the domain name and your AWS account**, not from the certificate ID and not from the region.
- So `_a79865eb....example.com. CNAME _424c7224....acm-validations.aws.` published **once** in the (global) Route 53 hosted zone will validate:
  - the `eu-west-1` cert for `api.example.com`,
  - the `eu-west-2` cert for `api.example.com`,
  - the `us-east-1` CloudFront cert for `api.example.com`,
  - every future renewal of all three,
  - and any replacement cert you create after deleting one.
- A wildcard and its base domain produce **the same** record. `*.example.com` and `example.com` share one identical name/value pair. So a single wildcard cert strategy needs exactly one record per zone.

Practical consequence: **mirroring ACM into the standby region adds zero new DNS records.** The `aws_route53_record` resources you already have are sufficient. You only add a second `aws_acm_certificate` + `aws_acm_certificate_validation` pair under a new provider alias. That is about 15 lines of HCL per region. This is the cheapest win in the entire prerequisites programme.

> [!warning] Same account only
> The shared-token behaviour is scoped to **your AWS account**. A certificate for the same FQDN requested from a *different* account gets a *different* validation token and needs its own record. If the standby region lives in a separate AWS account (common in a mature multi-account org — check against [[terraform-repo-structure]]), you need a second CNAME, and the two records have different names so they can coexist in the one zone. Confirm the account model before assuming one record is enough — this is in [[#Open questions]].

## Replication / mirroring options

### Option 1 — Request a second certificate per region (recommended)

The only real option for ACM-managed public certs. Cost: **$0** (ACM public certificates are free). Effort: one provider alias and one module call. Meets RTO trivially because the cert sits there, issued, doing nothing, indefinitely.

### Option 2 — Import the same third-party certificate into both regions

If the company uses an external CA (DigiCert etc.), you *can* import the identical cert+key into every region, and then the ALB in both regions presents a byte-identical certificate. This is the only way to have literally the same certificate in two regions.

Trade-off: **imported certificates are NOT eligible for ACM managed renewal.** From the [managed renewal docs](https://docs.aws.amazon.com/acm/latest/userguide/managed-renewal.html): "NOT ELIGIBLE if imported." You own the renewal calendar, in every region, forever, and a missed renewal is a total outage. Do not do this unless there is a compliance reason to pin to a specific CA.

### Option 3 — ACM Private CA

Relevant only for **internal** mTLS / service-to-service traffic, not for the public edge. Notes for the standby:

- A Private CA is itself a regional resource. Cross-region issuance is not a thing; you either create a subordinate CA in the standby region, or you share one CA across regions via **AWS RAM** (RAM supports sharing private CAs) and issue region-local end-entity certs from it.
- Private CA has a real standing cost — it is one of the more expensive AWS control-plane resources, billed per CA per month plus per certificate issued. If the standby needs its own CA, that cost doubles. Check current figures on the [AWS Private CA pricing page](https://aws.amazon.com/private-ca/pricing/) before committing; do not quote from memory.
- Private certs issued via the AWS Private CA `IssueCertificate` API are **not** eligible for ACM managed renewal either. Only private certs requested through ACM's `RequestCertificate` *and* then associated with a service or exported are renewed automatically.
- **Trust store mirroring is the real work**, not the CA. Every workload in the standby must trust the standby's CA chain. If you create a *separate* root in the standby, you have doubled the trust distribution problem. Prefer one shared root (via RAM) with regional subordinates.

If the product is a public HTTPS API with no internal mTLS, Private CA is **N/A** — skip it.

### Option 4 — Do nothing, use the default `*.amazonaws.com` cert

Only viable for internal, non-customer-facing endpoints where the client tolerates the ELB's own hostname. Explicitly not viable for the customer-facing product. Mentioned only for completeness.

### The ACME wrinkle (new, worth knowing)

ACM now exposes **ACME endpoints** (the Let's Encrypt protocol) as a certificate automation path, with its own quotas (50 ACME endpoints per region, 100 domain names per ACME-issued cert vs the default 10 for classic ACM certs). Two things matter for this project:

- ACME-issued certificates **do not count** toward the 2,500-certificate quota.
- ACME-issued certificates are **not eligible for ACM managed renewal** — "Renewal of ACME certificates is managed by the ACME client."

That second point makes ACME a *worse* fit for a warm standby than the classic flow, because a standby region's cert renewal would depend on an ACME client that is only running in... where, exactly? Stick with classic `RequestCertificate` + DNS validation. Flagged here so nobody "modernises" into a renewal outage.

## The `us-east-1` rule

**CloudFront viewer certificates must be requested or imported in `us-east-1`.** From the [CloudFront SSL/TLS requirements](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cnames-and-https-requirements.html):

> To use a certificate in AWS Certificate Manager (ACM) to require HTTPS between viewers and CloudFront, make sure you request (or import) the certificate in the US East (N. Virginia) Region (`us-east-1`).

The same page notes the useful exception: for **CloudFront-to-origin** HTTPS where the origin is an ELB, the origin's certificate "can be requested or imported in any AWS Region." So the split is:

| Certificate | Region |
|---|---|
| Viewer ↔ CloudFront (the distribution's `acm_certificate_arn`) | **`us-east-1` only** |
| CloudFront ↔ ALB origin (the cert on the ALB listener) | The ALB's own region (`eu-west-1`, `eu-west-2`, …) |
| Viewer ↔ ALB directly (no CloudFront) | The ALB's own region |

Other global services with the same `us-east-1` pin, all confirmed in the [AWS Fault Isolation Boundaries whitepaper, Appendix B](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/appendix-b---edge-network-global-service-guidance.html):

- **AWS WAF with `CLOUDFRONT` scope** — the web ACL must be created in `us-east-1`. A `REGIONAL`-scope ACL created in `us-east-1` is a *different object* and cannot be attached to a distribution. See [[aws-cloudfront]] and [[aws-waf-shield]].
- **CloudFront control plane** — hosted in `us-east-1`.
- **Route 53 control plane** — hosted in `us-east-1`. See [[aws-route53]], this is the big one.
- **Shield Advanced control plane** — `us-east-1`.
- **Global Accelerator control plane** — `us-west-2`, not `us-east-1`. A useful anomaly; see [[aws-alb-nlb]].

The whitepaper also makes the static-stability point for ACM specifically:

> Using custom certificates with your CloudFront distribution relies on the ACM control plane in the us-east-1 Region. During a control plane impairment, your existing certificates configured in your distribution will continue to work as well as automatic certificate renewals. **Do not rely on changing the distribution's configuration or creating new certificates as part of your recovery path.**

That sentence is the whole ACM DR strategy in one line. Pre-provision; never create at failover.

## RPO / RTO analysis

**RPO: N/A.** A certificate holds no customer data. Nothing to lose.

**RTO: passes, but only if pre-provisioned.** Where the time goes:

| Phase | Duration | Pre-provisioned? |
|---|---|---|
| Cert exists and is `ISSUED` in standby region | 0 s at failover | **Yes — must be** |
| Cert attached to standby ALB listener | 0 s at failover | **Yes — must be** |
| Cert attached to CloudFront distribution | 0 s at failover | Yes (one dist, one cert, no change) |

If instead you create the certificate at failover time:

| Phase | Duration |
|---|---|
| `RequestCertificate` API call | seconds |
| Write validation CNAME (if not already present) | seconds, **but needs the Route 53 control plane in `us-east-1`** |
| ACM observes the record and issues | typically minutes; console warns **"up to 30 minutes"**; hard timeout at **72 hours** |
| Attach to ALB listener | seconds |

Best case ~2 minutes, realistic case 30 minutes, worst case 72 hours. The 30-minute figure alone blows a 15-minute RTO, and it depends on a control plane that may be the very thing that failed. **Creating certificates at failover time fails the RTO. Full stop.**

The `aws_acm_certificate_validation` resource has a **default create timeout of 75 minutes**, which is Terraform's own admission of how long this can take. If your failover runbook contains a `terraform apply` that includes this resource, your runbook has a 75-minute step in it.

## Warm standby shape

While the primary is healthy, the standby region holds:

- One `aws_acm_certificate` per domain set, status `ISSUED`, **attached to the standby ALB's HTTPS listener**.
- Nothing else. No compute needed for the cert to exist or renew.

**Cost while idle: $0.** ACM public certificates are free. There is no per-certificate charge, no charge for renewal, no charge for the validation record beyond the standard Route 53 record (which you already have and which is shared with the primary anyway).

This makes ACM the single best value item in the prerequisites programme: full RTO compliance for zero standing cost.

> [!important] Attachment is what keeps renewal alive
> A certificate is eligible for managed renewal only if it is "associated with another AWS service, such as Elastic Load Balancing or CloudFront" or has been exported since issuance. A certificate sitting **unattached** in the standby region is **not eligible for automatic renewal** and will expire after ~13 months.
>
> This is a genuine warm-standby trap: if your cost-saving posture is "the standby ALB is deleted until we need it", the standby certificate quietly stops renewing, and the day you fail over you attach an expired certificate. **Keep the standby ALB provisioned** (which [[aws-alb-nlb]] argues for on pre-warming grounds anyway) so the certificate stays attached and stays renewing.

## Quotas

Per region, per account, from the [ACM quotas page](https://docs.aws.amazon.com/acm/latest/userguide/acm-limits.html) and the [General Reference](https://docs.aws.amazon.com/general/latest/gr/acm.html):

| Quota | Default | Adjustable |
|---|---|---|
| ACM certificates (PENDING or ISSUED) | **2,500 per region** | Yes |
| ACM certificates requested in last 365 days | 5,000 per region | Yes |
| Imported certificates | 2,500 per region | Yes |
| **Domain names per ACM certificate** | **10** | Yes, up to 100 |
| Private CAs | 200 | — |

Two notes for this project:

1. **The quota is per region, so mirroring does not consume primary-region quota.** Doubling certificate count across two regions does not bring you closer to either region's limit. At the scale implied by one product in three regions, 2,500 is not a constraint.
2. **The 10-domain-names-per-cert default is the one you'll actually hit.** Expired and revoked certificates *do* count toward the 2,500 total, which matters if something in CI is churning certificates — worth a `ListCertificates` audit before assuming headroom.

## Terraform implementation

### The mandatory provider block

Every stack that touches the edge needs three aliases. Put this in the cookiecutter root template so it is impossible to forget:

```hcl
terraform {
  required_version = ">= 1.9"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

# Primary — the region this environment currently serves from.
provider "aws" {
  region = var.primary_region
  default_tags { tags = local.common_tags }
}

# Standby — the paired mirror region.
provider "aws" {
  alias  = "standby"
  region = var.standby_region
  default_tags { tags = local.common_tags }
}

# us-east-1 — NOT a region choice. A hard requirement for CloudFront
# certificates and CLOUDFRONT-scope WAF web ACLs. Present in every stack
# even when primary_region is already us-east-1, so that module code can
# unconditionally reference provider = aws.use1 without branching.
provider "aws" {
  alias  = "use1"
  region = "us-east-1"
  default_tags { tags = local.common_tags }
}
```

> [!note] Why alias `us-east-1` even when it *is* the primary
> For the US pair, `primary_region = "us-east-1"`. It is tempting to drop the alias and use the default provider. Don't. If you do, the module's `providers = { aws.use1 = aws }` mapping becomes conditional on which pair you're deploying, and the cookiecutter template stops being uniform. Two provider blocks pointing at the same region is free and keeps one template for all three pairs.

### The certificate module

`modules/acm-certificate/main.tf` — designed so that the *same* module is called once per region and the validation records are created only once, by the primary call.

```hcl
variable "domain_name" {
  type        = string
  description = "Primary FQDN, e.g. api.example.com. Also used as the cert's CN."
}

variable "subject_alternative_names" {
  type        = list(string)
  default     = []
  description = "Additional SANs. Remember the default quota is 10 names total."
}

variable "hosted_zone_id" {
  type        = string
  description = "Route 53 zone that will hold the validation CNAMEs. Global; the same zone serves every region."
}

variable "create_validation_records" {
  type        = bool
  default     = true
  description = <<-EOT
    Set true for exactly ONE invocation per (account, domain). The ACM DNS
    validation token is identical across regions for the same FQDN in the same
    account, so additional regional certificates validate against the records
    the first invocation created. Set false for standby/us-east-1 copies.
  EOT
}

variable "validation_record_fqdns" {
  type        = list(string)
  default     = []
  description = "When create_validation_records = false, pass the FQDNs emitted by the invocation that did create them, so the validation waiter has a dependency edge."
}

resource "aws_acm_certificate" "this" {
  domain_name               = var.domain_name
  subject_alternative_names = var.subject_alternative_names
  validation_method         = "DNS"

  # Without this, any change to SANs destroys the cert while it is still
  # attached to the listener -> hard outage. This is the single most
  # important lifecycle block in the estate.
  lifecycle {
    create_before_destroy = true
  }
}

# Only the designated owner invocation writes the records.
resource "aws_route53_record" "validation" {
  for_each = var.create_validation_records ? {
    for dvo in aws_acm_certificate.this.domain_validation_options :
    dvo.domain_name => {
      name  = dvo.resource_record_name
      type  = dvo.resource_record_type
      value = dvo.resource_record_value
    }
  } : {}

  zone_id = var.hosted_zone_id
  name    = each.value.name
  type    = each.value.type
  records = [each.value.value]
  ttl     = 300

  # A wildcard and its apex produce an IDENTICAL record. Without this,
  # a cert for both example.com and *.example.com fails with a duplicate
  # record error on the second one.
  allow_overwrite = true
}

resource "aws_acm_certificate_validation" "this" {
  certificate_arn = aws_acm_certificate.this.arn

  validation_record_fqdns = var.create_validation_records ? [
    for r in aws_route53_record.validation : r.fqdn
  ] : var.validation_record_fqdns

  timeouts {
    create = "20m" # default is 75m; fail fast in CI rather than hang a pipeline
  }
}

output "certificate_arn" {
  # Depend on the validation resource, not the certificate, so that consumers
  # never attach a PENDING_VALIDATION cert to a listener.
  value = aws_acm_certificate_validation.this.certificate_arn
}

output "validation_record_fqdns" {
  value = [for r in aws_route53_record.validation : r.fqdn]
}
```

### Calling it — one call per region, one shared set of records

```hcl
# The primary region owns the validation records.
module "cert_primary" {
  source = "../../modules/acm-certificate"

  domain_name               = "api.${var.public_domain}"
  subject_alternative_names = ["*.api.${var.public_domain}"]
  hosted_zone_id            = data.aws_route53_zone.public.zone_id
  create_validation_records = true

  providers = { aws = aws }
}

# The standby reuses them. Zero new DNS records.
module "cert_standby" {
  source = "../../modules/acm-certificate"

  domain_name               = "api.${var.public_domain}"
  subject_alternative_names = ["*.api.${var.public_domain}"]
  hosted_zone_id            = data.aws_route53_zone.public.zone_id
  create_validation_records = false
  validation_record_fqdns   = module.cert_primary.validation_record_fqdns

  providers = { aws = aws.standby }
}

# The CloudFront cert. us-east-1 whether you like it or not.
module "cert_cloudfront" {
  source = "../../modules/acm-certificate"

  domain_name               = "www.${var.public_domain}"
  subject_alternative_names = [var.public_domain]
  hosted_zone_id            = data.aws_route53_zone.public.zone_id
  create_validation_records = false
  validation_record_fqdns   = module.cert_primary.validation_record_fqdns

  providers = { aws = aws.use1 }
}
```

> [!tip] aws provider v6 `region` argument
> Provider v6 added a `region` argument on individual resources, letting you target a region without declaring an alias. It is tempting to collapse the aliases. **Resist it for `us-east-1`.** An explicit `aws.use1` alias is self-documenting — a reader sees immediately that the CloudFront cert is regionally pinned by requirement, not by accident. A bare `region = "us-east-1"` attribute buried in a resource reads like a mistake and eventually someone "fixes" it.

### The `create_before_destroy` trap

Without `lifecycle { create_before_destroy = true }` on `aws_acm_certificate`, any change to `domain_name` or `subject_alternative_names` is a **ForceNew** replacement, and Terraform's default ordering is destroy-then-create. The certificate is destroyed *while still attached* to the ALB listener and CloudFront distribution. This is a real outage, not a theoretical one.

With `create_before_destroy`, Terraform issues the new cert, you re-point the listener, then the old cert is removed. Add it now, before anyone edits a SAN list. This is the `ForceNew` the brief asks to be flagged loudly.

## Migration path from single-region

No downtime, no replacement, genuinely easy. This is the one service where the migration is a pure addition.

1. **Audit.** In the live region, list certificates and confirm which are ACM-issued (renewable) vs imported (not). `aws acm list-certificates --includes keyTypes=RSA_2048` then `describe-certificate` for `Type` and `RenewalEligibility`.
2. **Confirm the validation records already exist in Route 53 and are correct.** If the certificates were originally created by hand in the console with "Create records in Route 53", the records exist but are **not in Terraform state**. Import them:
   ```
   terraform import 'module.cert_primary.aws_route53_record.validation["api.example.com"]' \
     Z1234567890ABC__a79865eb4cd1a6ab990a45779b4e0b96.api.example.com._CNAME
   ```
   (Zone ID, underscore-underscore, record name, underscore, type.) Getting these into state is the prerequisite for everything else — an unmanaged validation record is one careless `terraform destroy` away from breaking renewals in all regions.
3. **Bring the existing primary certificate under the module** with `terraform import` on `aws_acm_certificate` (import by ARN). Run `terraform plan` and confirm **empty diff** before proceeding. If the plan wants to replace the cert, stop — you have a SAN ordering or `domain_name` mismatch, fix the config, not the cloud.
4. **Add the standby module call** with `create_validation_records = false`. `terraform apply`. The cert issues in minutes against the existing records, with no DNS change whatsoever. Nothing in the primary region is touched. This step is safe to run during business hours.
5. **Attach the standby cert to the standby ALB listener** (see [[aws-alb-nlb]]) so managed renewal stays eligible.
6. **Verify renewal eligibility** in the standby: `aws acm describe-certificate --region eu-west-2 --certificate-arn ... --query 'Certificate.RenewalEligibility'` should return `ELIGIBLE`. If it returns `INELIGIBLE`, the cert is not attached to anything — go back to step 5. **Add this as a recurring check**, see [[failover-runbooks]].

Nothing in this sequence forces replacement of any existing resource, provided step 3's empty diff is verified.

## Failover procedure

**Nothing to do.** This is the correct and desired answer, and it is worth stating loudly: a well-configured ACM setup contributes **zero seconds** to the RTO.

At failover, the standby ALB already has an `ISSUED`, in-region, auto-renewing certificate on its HTTPS listener. DNS moves (see [[aws-route53]]), traffic arrives, TLS terminates. No ACM API call is made, which is exactly right given that the ACM control plane for the CloudFront cert lives in `us-east-1` and may be the thing that's broken.

The only ACM-related failover check worth putting in the runbook is a pre-flight, not a failover step:

```bash
# Run weekly, not at 3am. Alert if anything is not ISSUED/ELIGIBLE.
for region in eu-west-1 eu-west-2 us-east-1; do
  aws acm list-certificates --region "$region" \
    --certificate-statuses ISSUED \
    --query 'CertificateSummaryList[].[DomainName,CertificateArn]' --output text
done
```

## Failback

Also nothing to do. Both certificates continue to exist and renew independently. Failback is a DNS operation only.

The one asymmetry worth noting: `ACM certificates are regional resources. If you have certificates for the same domain name in multiple AWS Regions, each of these certificates must be renewed independently.` Renewals are therefore not synchronised — the primary and standby certs will have different `NotAfter` dates and different serial numbers. That is fine and expected. It does mean **certificate-pinning clients will break on failover**, because the standby presents a different certificate. If any mobile client or B2B partner pins a leaf certificate, failover breaks them regardless of DNS. Pin to the CA or nothing. Raised in [[#Open questions]].

## Gotchas

1. **Deleting the validation CNAME kills renewal everywhere at once.** The blast radius is every region and every certificate for that FQDN, and the failure is silent for up to 13 months. Protect the record: `prevent_destroy`, or at minimum a policy check in CI. This is the #1 ACM incident pattern.
2. **An unattached standby certificate does not auto-renew.** Covered above. The interaction between "save money by deleting the standby ALB" and "certificate renewal eligibility" is non-obvious and will not be caught by any plan diff.
3. **`create_before_destroy` is not the default.** Omitting it turns a SAN edit into an outage.
4. **Wildcard and apex share a validation record.** `allow_overwrite = true` on `aws_route53_record`, or the second one errors. Everyone hits this once.
5. **`aws_acm_certificate_validation` is a fiction.** It creates nothing in AWS; it is a waiter. It exists solely to give you a dependency edge so listeners don't attach a pending cert. Always output `aws_acm_certificate_validation.this.certificate_arn`, never `aws_acm_certificate.this.arn`, or you will race.
6. **75-minute default timeout.** If validation is misconfigured, your pipeline hangs for 75 minutes before failing. Set `timeouts { create = "20m" }`.
7. **HTTPS health checks don't validate certificates.** From the [Route 53 health check docs](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-determining-health-of-endpoints.html): "HTTPS health checks don't validate SSL/TLS certificates, so checks don't fail if a certificate is invalid or expired." **An expired certificate in the standby region will not be detected by Route 53 health checks.** Your failover target can be broken and green at the same time. Monitor `DaysToExpiry` from the ACM CloudWatch metric instead — see [[cloudwatch-observability]].
8. **72-hour validation timeout.** If the record is wrong, ACM sits in `PENDING_VALIDATION` for 72 hours, then moves to `VALIDATION_TIMED_OUT` and the certificate is unusable — you cannot "retry" it, you must request a new one.
9. **CNAME chain limit of five.** "CNAME resolution will fail if more than five CNAMEs are chained together." Only relevant if the DNS estate has delegation layers, but it silently breaks validation if so.
10. **Email validation cannot be converted to DNS validation.** "After you create a certificate with email validation, you cannot switch to validating it with DNS. To use DNS validation, delete the certificate and then create a new one." If any live certificate uses email validation, replacing it is a migration task, not a config change.
11. **ACME-issued and imported certs are not managed-renewal eligible.** Different renewal owners for different certs in the same estate is a recipe for a missed expiry.

## Why DNS validation beats email validation here

Not a close call:

| | DNS validation | Email validation |
|---|---|---|
| Automatable in Terraform | Yes, fully | No — a human must click a link in an inbox |
| Works for a standby region with no new action | **Yes — same record serves all regions** | No — new approval email per certificate |
| Auto-renewal | Automatic, indefinitely, as long as the record persists | ACM emails you; **a human must act** every ~11 months |
| Renewal at 3am during an incident | Not needed; already done | Impossible |
| Requires control of `admin@`/`hostmaster@` mailboxes | No | Yes — often owned by a different team or a dead distribution list |
| Convertible later | — | **No.** One-way door. |

Email validation is incompatible with an automated multi-region estate in three independent ways: it can't be Terraformed, it doesn't share across regions, and its renewal is a manual step that will eventually be missed. If any existing certificate uses it, replacing it is a prerequisite for this programme, not an optional tidy-up.

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Per-region certs vs imported third-party | A: ACM-issued per region. Free, auto-renewing, one shared validation record. Different serial per region. | B: Import one third-party cert everywhere. Byte-identical cert in all regions. **No managed renewal — you own expiry, per region, forever.** | **A.** B trades an automated, free, silent-success path for a manual, paid, silent-failure path. Only choose B if a client pins a specific CA. |
| Wildcard vs explicit SAN list | A: `*.example.com` — one cert, one validation record, new subdomains need no Terraform change. | B: Explicit SANs — tighter blast radius if the key is compromised, but every new subdomain is a cert replacement (`ForceNew`). | **A (wildcard)** for the standby mirroring effort specifically: it makes the number of validation records constant, which is what makes mirroring free. Use B only where a security review demands it. |
| Who owns the validation record | A: Primary-region module writes it, standby consumes (`create_validation_records` flag). | B: A separate `dns/` stack owns all validation records; both regional stacks consume. | **B if the estate already has a dedicated DNS stack**, A otherwise. B is cleaner (the record's lifetime is genuinely independent of any one region) but A is fewer moving parts. Do not let both regions try to write it. |
| Private CA for internal mTLS | A: Shared root via AWS RAM + regional subordinates. | B: Independent root per region. | **A**, if internal mTLS exists at all. B doubles the trust-distribution problem for no benefit. Confirm whether mTLS is in scope — see Open questions. |

## Cost

| Item | Cost |
|---|---|
| ACM public certificate (issued, renewed, any region) | **$0** |
| Validation CNAME in Route 53 | Included in the hosted zone's record allowance (charges start above 10,000 records per zone at $0.0015/record/month) |
| Standby region certificate | **$0** |
| ACM Private CA, if used | Per-CA monthly + per-certificate. Verify on the [AWS Private CA pricing page](https://aws.amazon.com/private-ca/pricing/) — not quoted here to avoid a stale figure. |

**ACM adds $0 to the standby's idle cost.** There is no lever to pull because there is nothing to save. The only cost-adjacent decision is Private CA, which is a real line item if internal mTLS is in scope.

## Open questions

1. **Is the standby region in the same AWS account as the primary?** The shared-validation-token behaviour is account-scoped. A separate standby account means a second CNAME per domain. This changes the module's `create_validation_records` design and needs answering before the module is written. Cross-reference [[terraform-repo-structure]].
2. **Do any current certificates use email validation?** If so, they need replacing (a one-way door) before mirroring, and that replacement needs a maintenance window.
3. **Are any existing certificates imported rather than ACM-issued?** Those have a manual renewal calendar that must be extended to the standby region. Who owns it today?
4. **Is internal mTLS in scope?** Determines whether ACM Private CA is a real workstream or N/A. Currently written as "probably N/A" — confirm.
5. **Does any client pin certificates?** Leaf-pinning breaks on failover by construction, because the standby's certificate is a different certificate. Mobile apps and B2B partners are the usual suspects.
6. **Which team owns the public hosted zone?** If DNS is managed outside this Terraform repo, the validation-record step needs a cross-team process and the `create_validation_records = true` path may not be usable at all.

## Sources

- [AWS Certificate Manager DNS validation](https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html) — the load-bearing source for this note. Contains the explicit statement that the CNAME validation token works for any AWS Region, the wildcard/apex identical-record table, the 72-hour timeout, and the 5-CNAME chain limit.
- [Managed certificate renewal in AWS Certificate Manager](https://docs.aws.amazon.com/acm/latest/userguide/managed-renewal.html) — renewal eligibility rules (attached-to-a-service requirement, imported and ACME certs ineligible), and the statement that certificates in multiple regions renew independently.
- [Requirements for using SSL/TLS certificates with CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cnames-and-https-requirements.html) — the `us-east-1` rule verbatim, plus the exception for CloudFront-to-origin certificates.
- [ACM Quotas](https://docs.aws.amazon.com/acm/latest/userguide/acm-limits.html) — 2,500 certs/region, 10 domains/cert, API rate limits, ACME quotas.
- [ACM endpoints and quotas (General Reference)](https://docs.aws.amazon.com/general/latest/gr/acm.html) — confirms `acm.ca-west-1.amazonaws.com` exists, so no `ca-west-1` gap; adjustable-quota table.
- [AWS Fault Isolation Boundaries — Appendix B, Edge network global service guidance](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/appendix-b---edge-network-global-service-guidance.html) — control-plane locations for ACM, CloudFront, WAF, Route 53, Shield, Global Accelerator, and the "do not create certificates in your recovery path" guidance.
- [How Amazon Route 53 determines whether a health check is healthy](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-determining-health-of-endpoints.html) — source for gotcha #7, that HTTPS health checks do not validate certificates.
- [`aws_acm_certificate_validation` resource docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/acm_certificate_validation) — confirms the resource "does not represent a real-world entity in AWS" and the 75-minute default create timeout.

## Related notes

[[aws-route53]] · [[aws-cloudfront]] · [[aws-alb-nlb]] · [[aws-waf-shield]] · [[terraform-repo-structure]] · [[failover-runbooks]] · [[region-pair-selection]] · [[cost-modelling]]
