---
title: AWS WAF & Shield — Multi-Region
service: waf-shield
tags: [service, multi-region, waf, wafv2, shield, ddos, security, edge]
status: partial
replication: none — a REGIONAL web ACL is a per-region resource with no native cross-region anything; you deploy a second copy and keep it identical by discipline
rpo_achievable: "N/A — configuration, not data. But rate-based counters and Shield Advanced traffic baselines are per-region runtime state that is lost entirely at failover."
rto_achievable: "0 s if the standby web ACL exists and is associated today; 5–20 min and a security hole if it is created at failover time"
meets_targets: conditional — yes only if the standby ACL is pre-created, pre-associated and drift-checked
updated: 2026-09-22
---

# AWS WAF & Shield — Multi-Region

## TL;DR

- **A regional web ACL is a per-region resource. If you do not duplicate it into the standby with identical rules, the standby is unprotected the moment it takes traffic.** AWS states it flatly: *"The protection pack (web ACL) and any AWS WAF resources that it uses must be located in the Region where the associated resource is located."* There is no replication, no global rule set, no cross-region ARN reference. This is a security hole that opens **exactly at failover**, when every human is looking at database promotion and DNS and nobody is looking at the WAF. Make this the spine of the design.
- **The two scopes are separate, non-interchangeable resources.** `CLOUDFRONT` scope must be created in **`us-east-1`** and can only attach to CloudFront distributions (and Amplify apps). `REGIONAL` scope attaches to ALB, API Gateway REST API, AppSync, Cognito user pools, App Runner, Verified Access, Bedrock AgentCore Gateway and Amplify — and must live in the protected resource's own Region. A `REGIONAL` ACL created in `us-east-1` is a **different object** from a `CLOUDFRONT` ACL created in `us-east-1`. See [[#The two scopes]].
- **WAF cannot be attached to Global Accelerator.** Global Accelerator is not on AWS's list of resources a web ACL can protect. If [[aws-global-accelerator]] is chosen as the failover mechanism — and [[aws-alb-nlb]] recommends exactly that — the WAF still lives on the ALB *behind* the accelerator, per region, and you are back to needing two identical regional ACLs. GA does not simplify the WAF story at all. See [[#Does WAF work in front of Global Accelerator?]].
- **Rate-based rule counters do not cross regions, and Shield Advanced's traffic baseline does not either.** At failover every rate limit restarts from zero: an attacker mid-block is released, and AWS needs *"between 24 hours and 30 days"* to rebuild an application-layer DDoS baseline for a resource it has never seen traffic on. **A standby that has never served production traffic has no DDoS baseline**, which is the least-appreciated finding in this note. See [[#Rate-based rules do not carry counters across regions]].
- **Cost of the duplicate is small and the commitment behind it is not.** A duplicated standby web ACL with 10 rules costs **$5 + $10 = $15/month at zero traffic**, plus managed rule group subscriptions. Shield Advanced is **$3,000/month on a 1-year commitment, billed per payer account for the whole organisation** — so the standby resources cost nothing extra in subscription, but each one needs its **own `aws_shield_protection`**, and a standby resource with no protection object is **not protected**.

## Does this service cross regions at all?

No. In every sense that matters, and more completely than most services in this vault.

| Object | Scope | Crosses regions? |
|---|---|---|
| `aws_wafv2_web_acl` (`REGIONAL`) | The Region it was created in | **No.** Cannot be associated with a resource in another Region. |
| `aws_wafv2_web_acl` (`CLOUDFRONT`) | Global, **created in `us-east-1` only** | Global by nature, but pinned to one Region's control plane. |
| `aws_wafv2_rule_group` | Same scope + Region as the ACL that references it | **No.** Two copies required. |
| `aws_wafv2_ip_set` | Same scope + Region | **No.** Two copies required, and they drift. |
| `aws_wafv2_regex_pattern_set` | Same scope + Region | **No.** Two copies required. |
| Managed rule groups (AWS Managed Rules) | Available in every WAF Region, **versioned per-ACL-reference** | The *catalogue* is global; **your version pin is per ACL**, so it drifts. |
| Rate-based rule counters | Per web ACL, per Region | **No.** Reset to zero in the standby. |
| WAF logging destination | Per Region (CloudWatch Logs / S3 / Firehose) | **No.** Two pipelines required. |
| `aws_shield_protection` | Per resource, **`us-east-1` control plane** | **No.** One protection object per protected resource. |
| Shield Advanced **subscription** | **Account-level, billed per payer account** | **Yes** — this is the one thing that genuinely does not need duplicating. |

The AWS statement that settles it, verbatim from
[Resources that you can protect with AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works-resources.html):

> The protection pack (web ACL) and any AWS WAF resources that it uses must be located in the Region where the associated resource is located. For Amazon CloudFront distributions, this is set to US East (N. Virginia).

And, on data locality — relevant to [[data-residency]] and the CA pair:

> The protection pack (web ACL) and any other AWS WAF resources that it uses must be located in the same Region as the protected resources. When monitoring and managing web requests for a protected regional resource, AWS WAF keeps all data in the same Region as the protected resource.

> [!danger] The finding that matters most in this note
> **A standby ALB with no associated web ACL serves traffic with no WAF at all.**
> It does not fail closed. It does not warn. There is no CloudWatch metric called
> `NoWebAclAttached`. The ALB comes up, the targets pass health checks, DNS or the
> traffic dial moves, and your SQL-injection, XSS, bad-bot, known-bad-input and
> IP-reputation rules are simply **not running** — for the entire duration of the
> incident, which is precisely the window in which an attacker who caused or
> noticed the outage would look.
>
> This is not a hypothetical drift problem. It is the **default state** of a
> standby region that was built by copying the compute and forgetting the edge.

## The two scopes

This is the first thing to get straight because the mistake is silent and common.

### `CLOUDFRONT` scope

- Created **only in `us-east-1`**. Verbatim: *"AWS WAF is available globally for CloudFront distributions, but you must use the Region US East (N. Virginia) to create your protection pack (web ACL) and any resources used in the protection pack (web ACL), such as rule groups, IP sets, and regex pattern sets. Some interfaces offer a region choice of 'Global (CloudFront)'. Choosing this is identical to choosing Region US East (N. Virginia) or 'us-east-1'."*
- This is the **same `us-east-1` constraint** that [[aws-acm]] documents for CloudFront viewer certificates and that [[aws-cloudfront]] documents for the distribution control plane. Three separate resources, one shared pin. They belong in the same Terraform stack — see [[#Terraform implementation]].
- **One-to-many, but exclusively CloudFront.** Verbatim: *"You can associate a protection pack (web ACL) with one or more CloudFront distributions. You cannot associate a protection pack (web ACL) that you have associated with a CloudFront distribution with any other AWS resource type."*
- Amplify is the odd one out: *"You must create any protection pack (web ACL) that you want to associate with an Amplify app in the Global CloudFront Region. You might already have a Regional protection pack (web ACL) in your AWS account, but they are not compatible with Amplify."*

### `REGIONAL` scope

Verbatim, the current list of regional resource types:

> + Amazon API Gateway REST API
> + Application Load Balancer
> + AWS AppSync GraphQL API
> + Amazon Cognito user pool
> + AWS App Runner service
> + Amazon Bedrock AgentCore Gateway
> + AWS Verified Access instance
> + AWS Amplify

Two things worth noting against the estate:

- **API Gateway: REST APIs only.** HTTP APIs (v2) are **not** on this list. If the
  estate uses HTTP APIs, WAF is not available on them directly and the protection
  has to move to CloudFront in front. Check this before designing — see
  [[aws-api-gateway]] and [[#Open questions]].
- **`aws_wafv2_web_acl_association` cannot be used for CloudFront.** The provider
  and the API both require you to set `web_acl_id` on the distribution instead.
  This is the most common Terraform confusion in this area.
- ALBs on Outposts cannot be associated: *"You can only associate a protection pack (web ACL) to an Application Load Balancer that's within AWS Regions."*

### What this estate needs, concretely

| Estate resource | Scope needed | Where the ACL lives | Duplicated for the standby? |
|---|---|---|---|
| CloudFront distribution (one per hostname, global — [[aws-cloudfront]]) | `CLOUDFRONT` | `us-east-1` | **No.** One distribution, one ACL, already global. |
| Public ALB, EU primary | `REGIONAL` | `eu-west-1` | — |
| Public ALB, EU standby | `REGIONAL` | **`eu-west-2`** | **Yes, mandatory** |
| Public ALB, US primary | `REGIONAL` | `us-east-1` | — |
| Public ALB, US standby | `REGIONAL` | **`us-west-2`** | **Yes, mandatory** |
| Public ALB, CA primary | `REGIONAL` | `ca-central-1` | — |
| Public ALB, CA standby | `REGIONAL` | **`ca-west-1`** | **Yes, mandatory** |
| API Gateway REST API (if used regionally) | `REGIONAL` | Each region | **Yes** |
| Cognito user pool | `REGIONAL` | Each region | **Yes** — and note [[aws-cognito]]'s own MRR gap |
| Global Accelerator | **none — not supported** | — | N/A, see below |

**Six regional web ACLs where there are three today.** That is the shape of the
work.

> [!warning] The `us-east-1` collision on the US pair
> For the US pair, the `CLOUDFRONT`-scope ACL and the `REGIONAL`-scope ACL for the
> primary ALB **both live in `us-east-1`**. They are different objects with
> different ARNs and are not interchangeable, but they will appear side by side in
> the same console view and the same Terraform state. **Name them so the scope is
> unmissable** — `prod-edge-cloudfront` vs `prod-us-east-1-alb-regional` — because
> passing the wrong ARN to `aws_wafv2_web_acl_association` fails at apply time
> (good) while passing the wrong one to a CloudFront distribution's `web_acl_id`
> also fails (also good), but reading a console page and believing the wrong thing
> is protected fails silently (bad).

## `ca-west-1` parity — a rare pass

**WAFv2 is available in `ca-west-1`.** The
[AWS WAF endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/waf.html)
page lists Canada West (Calgary) with both a standard and a FIPS endpoint:

> | Canada West (Calgary) | ca-west-1 | wafv2.ca-west-1.amazonaws.com <br /> wafv2-fips.ca-west-1.amazonaws.com | HTTPS<br />HTTPS |

This **closes open question #6 in [[aws-alb-nlb]]**, which flagged "Is WAFv2
`REGIONAL` scope available in `ca-west-1`? Blocks the CA pair's design if not."
The answer is yes, and the CA pair is not forced onto a CloudFront-fronted design
to get a WAF.

Given the region's record in this vault — Calgary has already failed for Cognito
managed-replica, OpenSearch cross-cluster replication, Managed Grafana, Backup
Audit Manager and CloudFront Origin Shield ([[region-pair-selection]]) — this is a
genuinely useful negative result. **Record it as a pass.**

Caveats that are *not* yet verified for `ca-west-1` and should be before sign-off:

- Whether every **AWS Managed Rules** rule group the primary uses is offered in
  `ca-west-1`. AWS publishes the AMR catalogue as generally available but does not
  publish a per-region availability matrix; see [[#Still to research]].
- Whether **Shield Advanced** resource protection is available for `ca-west-1`
  ALBs. Shield Advanced's control plane is `us-east-1` and its protected-resource
  list is by *type*, not by Region, but this needs an explicit check.
- Whether **Bot Control / Fraud Control** (the paid intelligent-threat features)
  are offered there. These are the features most likely to lag in a young region.

## Rule drift between regions

This is the core operational risk and it deserves more space than the mechanics.

### What drifts, and how

| Thing | Why it drifts | How you find out |
|---|---|---|
| **Rule ordering / priority** | Someone reorders rules in the live region to fix a false positive during an incident. | Never, until failover changes which rule terminates first. |
| **Managed rule group version pin** | You pin `Version_2.0` in one region and let the other default to the latest. Or you bump one and forget the other. | Never. Both ACLs look "configured". |
| **IP sets** | Abuse-response automation writes a block to the live region only. | Never. The standby's block list is six months old. |
| **Regex pattern sets** | Same as IP sets, plus hand-edits in the console. | Never. |
| **Excluded / overridden rules within a managed group** | A rule gets set to `count` in the live region to stop a false positive. | **At failover, the standby blocks traffic the primary was allowing.** This is the drift direction that causes an outage rather than a hole. |
| **Default action** (`allow` vs `block`) | Copy-paste during initial build. | A standby with `default_action = block` serves 403 to everything. |
| **WCU budget** | One region accumulates rules until it hits the 5,000 WCU ceiling; the other doesn't. | At apply time — loud, at least. |
| **Logging configuration** | Standby logging never got set up. | At failover, when there are no logs to look at. |
| **Association itself** | The standby ALB was rebuilt and the association was not recreated. | **Never.** This is the total-hole case. |

Note the asymmetry: **drift that removes protection is silent, and drift that adds
protection causes a visible outage.** Teams therefore learn about the second kind
and never learn about the first. Your drift check has to test for both.

### The Terraform answer: one module, two provider aliases, shared rule definitions

The shape that actually works in a cookiecutter-templated monorepo is:

1. **One module** — `modules/regional-waf` — that takes the *rule content* as data
   and knows nothing about which Region it is in.
2. **Rule definitions held once**, in a `locals` block or a shared `.tf` file, and
   passed to both instantiations. Not copied. **The single most valuable property
   is that there is exactly one place in the repo where a rule is written down.**
3. **Two instantiations**, differing only in the provider alias.
4. A **CI check** that diffs the two rendered plans, because Terraform alone does
   not guarantee sameness — see [[#What still drifts despite all of this]].

See [[#Terraform implementation]] for the HCL and [[module-patterns]] for the
conventions it follows.

### What still drifts despite all of this

This is the honest part. One module and two aliases fixes the *declared*
configuration. It does not fix:

1. **Anything written outside Terraform.** IP sets updated by an abuse-response
   Lambda, by a SOC analyst in the console, or by a support runbook. These are the
   highest-frequency writes to a WAF in a live estate and they are almost never
   Terraform-managed, because Terraform is too slow for an active attack.
2. **Managed rule group *content*.** You pin a version; AWS ships a new version;
   you bump one region. Even with the same pin, AWS Managed Rules groups whose
   version is set to "default" track the latest and change under you independently
   per ACL. **Pin explicitly in both regions or neither.**
3. **Shield Advanced's automatically-managed rule group**, which Shield writes into
   the ACL on your behalf and which is derived from *that resource's* traffic
   baseline. It is by construction different in the two regions. You cannot make
   this identical and you should not try.
4. **Runtime state** — rate-based counters, challenge/CAPTCHA tokens, the
   intelligent-threat mitigation's learned models. Covered below.
5. **`lifecycle { ignore_changes }` asymmetry.** If someone adds an
   `ignore_changes` on the live ACL's rules to stop Terraform reverting an
   emergency hand-edit, Terraform stops being the source of truth in one region
   only. This is the most insidious one because it is *in* the code and still
   produces drift.

**The mitigation for (1) is architectural, not procedural:** any automation that
writes to a WAF resource must be written to write to **both** regions, and the
function that does it must take a list of regions, not a region. Make it
impossible to write to one. Cross-ref [[aws-lambda]].

## Rate-based rules do not carry counters across regions

### The counter behaviour

Rate-based rule state is per web ACL. There are two regional web ACLs. Therefore
there are two independent sets of counters, and the standby's are at zero until it
starts seeing traffic.

AWS does not publish a sentence saying "counters do not replicate across Regions"
— it does not need to, because the web ACL itself is a regional resource and there
is no mechanism by which they could. **This is inference from the resource model,
not a quote, and this note says so rather than fabricating one.** What AWS *does*
document, verbatim, from
[Rate-based rule caveats](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-rate-based-caveats.html):

> AWS WAF rate limiting is designed to control high request rates and protect your application's availability in the most efficient and effective way possible. It's not intended for precise request-rate limiting.
>
> + AWS WAF estimates the current request rate using an algorithm that gives more importance to more recent requests. Because of this, AWS WAF will apply rate limiting near the limit that you set, but does not guarantee an exact limit match.
> + Each time that AWS WAF estimates the rate of requests, AWS WAF looks back at the number of requests that came in during the configured evaluation window. Due to this and other factors such as propagation delays, it's possible for requests to be coming in at too high a rate for up to several minutes before AWS WAF detects and rate limits them. Similarly. the request rate can be below the limit for a period of time before AWS WAF detects the decrease and discontinues the rate limiting action. Usually, this delay is below 30 seconds.
> + If you change any of the rate limit settings in a rule that's in use, the change resets the rule's rate limiting counts. This can pause the rule's rate limiting activities for up to a minute. The rate limit settings are the evaluation window, rate limit, request aggregation settings, forwarded IP configuration, and scope of inspection.

Three things follow directly, and the third is the one people miss.

### The evaluation window

The `EvaluationWindowSec` parameter on `RateBasedStatement` accepts **60, 120, 300
or 600 seconds**, with **300 (five minutes) as the default**. Configurable windows
were added in [March 2024](https://aws.amazon.com/about-aws/whats-new/2024/03/aws-waf-rate-based-rules-configurable-time-windows).

So at failover:

- The standby has **zero history** in its evaluation window.
- **Every source IP gets a full fresh budget.** An attacker who was at 4,900 of a
  5,000-per-5-minutes limit in the primary starts again at 0 in the standby.
- An attacker who was **actively blocked** in the primary is **released**. The
  block is not a durable decision; it is a continuously re-evaluated function of a
  counter that no longer exists.
- AWS's own documented detection delay applies again from scratch: *"it's possible
  for requests to be coming in at too high a rate for up to several minutes before
  AWS WAF detects and rate limits them."* **Several minutes of a 900-second RTO
  budget, during which the standby is being rate-limit-free.**

### Sizing: the standby's thresholds are wrong the moment it becomes primary

A subtler problem. Rate-based thresholds are usually tuned against observed
production traffic *in the primary*. If the standby's ACL is a faithful copy, it
inherits thresholds sized for full production load — which is correct, and is the
argument for copying rather than tuning.

But if anyone has "helpfully" lowered the standby's thresholds because it normally
sees no traffic, **the standby will rate-limit legitimate production traffic
within minutes of taking over.** Combine that with the cold-cache retry storm
described in [[aws-cloudfront]] — clients retrying failed requests, CDN cache
misses bunching — and you have a legitimate traffic pattern that looks exactly
like an attack to a rule tuned for an idle region.

> [!danger] The retry storm looks like a DDoS to a fresh rate-based rule
> At failover, real users retry. Mobile apps retry with backoff. The CDN misses
> and re-fetches. Health checkers ramp. The standby's rate-based rules have **no
> historical context** to tell this apart from an attack, because their counters
> just started. If your thresholds are anything other than "sized for full
> production peak", **your own WAF will block your own recovering traffic.**
>
> Concrete mitigation: **size the standby's rate-based thresholds from the
> primary's peak, plus headroom for a retry storm — not from the standby's own
> (zero) traffic.** And deploy them in `count` mode in the standby if you cannot
> be confident, accepting that you lose the protection to keep the availability.
> That is a real trade-off and [[#Decisions to make]] records it as one.

### Shield Advanced's baseline is worse: 24 hours to 30 days

The rate-based counter problem resets in minutes. The Shield Advanced
application-layer baseline does not. Verbatim from
[Automating application layer DDoS mitigation with Shield Advanced](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-automatic-app-layer-response.html):

> Shield Advanced requires time to establish a baseline of your application's normal, historic traffic, which it leverages to detect and isolate attack traffic from normal traffic, to mitigate attack traffic. **The time to establish a baseline is between 24 hours and 30 days from the time you associate a web ACL with the protected application resource.**

> [!danger] A warm standby has no DDoS baseline, by definition
> "Warm standby" means the standby serves ~zero production traffic. Shield
> Advanced's automatic application-layer mitigation builds its model from **that
> resource's** historic traffic. A resource with no historic traffic has no model.
>
> So at failover you get: rate-based counters at zero, a Shield Advanced baseline
> that is either absent or built entirely from health checks and synthetic
> monitors, and a sudden step-change to 100% of production load — which is exactly
> the signal Shield is looking for when it decides an attack is underway.
>
> **The two plausible failure modes are opposite and both bad:** Shield has no
> baseline and does nothing useful, or Shield has a baseline built from an idle
> region and treats the failover itself as the attack.
>
> **There is no public AWS guidance on Shield Advanced baselines in a DR standby**
> — no documentation page, no blog, no re:Invent talk found addressing it. That is
> a finding, not a gap in the research. See [[#Open questions]]; this is worth an
> actual question to AWS Support or the account team, and it is the kind of
> question a TAM can answer definitively in one email.
>
> The partial mitigation available today: **run a non-trivial, continuous synthetic
> traffic load against the standby** so that it has *some* baseline and so that the
> failover step-change is a smaller multiple. This also solves the ALB pre-warming
> problem in [[aws-alb-nlb]] and the "GA won't fail over to an empty target group"
> problem. One mitigation, three benefits — it is probably the single highest-value
> change in the warm-standby shape.

Note also the **Anti-DDoS Managed Rule Group supersession**, verbatim from the same
page:

> Starting March 26, 2026, the Anti-DDoS Managed Rule Group (Anti-DDOS AMR) for AWS WAF becomes the default solution for protection against HTTP request flood attacks... It supersedes the Layer 7 Auto Mitigation (L7AM) feature. If you're an existing Shield Advanced customer, you can continue to use the legacy solution with existing or new AWS accounts. However, we encourage you to adopt the Anti-DDoS Managed Rule Group. The Anti-DDoS Managed Rule Group detects and mitigates attacks within seconds rather than minutes. If you're a new Shield Advanced customer and require access to the legacy solution, contact AWS Support.

**"Within seconds rather than minutes"** is AWS's own comparison and it matters for
a 900-second RTO. If Shield Advanced is in scope, **use the Anti-DDoS AMR, not
L7AM** — and note that being an AMR, it is a rule group referenced from the web
ACL, which means it is **one more thing to pin identically in both regions.**

## Shield Standard vs Shield Advanced

### Shield Standard

Automatic, free, no configuration, applies to all AWS customers. Nothing to
duplicate, nothing to fail over, nothing to check. The standby gets it by existing.
**No action required and no note needed beyond this paragraph.**

### Shield Advanced — the subscription is account-level, not regional

Verbatim from
[AWS Shield Advanced overview](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary.html):

> + For accounts that are members of an AWS Organizations organization, AWS bills the Shield Advanced subscriptions against the payer account for the organization, regardless of whether the payer account itself is subscribed.
> + When you subscribe multiple accounts that are in the same AWS Organizations consolidated billing account family, **one subscription price covers all subscribed accounts in the family.** The organization must own all of the AWS accounts and all of their resources.
> + When you subscribe multiple accounts for multiple organizations, you can still pay one subscription fee across all of the organizations, accounts, and resources providing you own all of them. Contact your account manager or AWS support and request a fee waiver on the AWS Shield Advanced subscription charges for all but one of the organizations.

**So: the subscription is per-organisation (strictly, per payer account), not
per-region and not per-account.** Standing up a standby region adds **zero
subscription cost**. This is the one genuinely good piece of news in this note.

### But protections are per-resource, and the standby is not protected by default

A Shield Advanced *subscription* does not protect anything. A `Protection` object
protects one resource. The resource types you can create protections for, per
[List of AWS resources that AWS Shield Advanced protects](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary-protected-resources.html):

- Amazon CloudFront distributions
- Amazon Route 53 hosted zones
- **AWS Global Accelerator standard accelerators**
- Amazon EC2 Elastic IP addresses
- Application Load Balancers and Classic Load Balancers
- EC2 instances (via an associated Elastic IP)
- Network Load Balancers (via an associated Elastic IP)

> [!danger] The standby ALB is NOT protected by Shield Advanced by default
> Subscribing the account does not enrol resources. **Each standby ALB needs its
> own `aws_shield_protection`.** If you only create it at failover time you are
> making a **`us-east-1` control-plane call during an incident** — the exact
> circular dependency [[aws-route53]] documents for `ChangeResourceRecordSets` and
> [[aws-cloudfront]] documents for `UpdateDistribution`. For the US pair, whose
> primary *is* `us-east-1`, that call may simply not succeed.
>
> **Pre-provision the protection on every standby resource.** It costs nothing
> beyond the subscription you are already paying.

Shield Advanced's control plane home region is `us-east-1`, per the
[AWS Fault Isolation Boundaries whitepaper, Appendix B](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/appendix-b---edge-network-global-service-guidance.html)
— the same source [[aws-acm]] and [[aws-alb-nlb]] cite.

### Cost

From [AWS Shield pricing](https://aws.amazon.com/shield/pricing/):

| Item | Price |
|---|---|
| Shield Advanced subscription | **$3,000 / month**, **1-year commitment**, **billed per payer account** (one fee per organisation) |
| Shield Advanced DTO — CloudFront | **$0.025 / GB** |
| Shield Advanced DTO — ELB / EC2 / Global Accelerator | **$0.050 / GB** |

Plus the WAF-cost offset, verbatim from the Shield Advanced overview:

> Your Shield Advanced subscription covers the costs of using standard AWS WAF capabilities for resources that you protect with Shield Advanced. The standard AWS WAF fees that are covered by your Shield Advanced protections are the cost per protection pack (web ACL), the cost per rule, and the base price per million requests for web request inspection, **up to 1,500 WCUs and up to the default body size.**

And what it does **not** cover:

> Your subscription to Shield Advanced does not cover the use of AWS WAF for resources that you do not protect using Shield Advanced. It also does not cover any additional non-standard AWS WAF costs for protected resources. Examples of non-standard AWS WAF costs are those for Bot Control, for the CAPTCHA rule action, for web ACLs that use more than 1,500 WCUs, and for inspecting the request body beyond the default body size.

> [!tip] A useful cost interaction for the standby
> If Shield Advanced is subscribed **and you create a protection on the standby
> ALB**, then the standby's web ACL fee, rule fees and base request fees are
> **covered by the subscription**. The duplicated standby WAF becomes free. If you
> do *not* protect the standby resource, you pay for its WAF separately.
>
> **That is a direct financial argument for pre-provisioning the standby's Shield
> protection**, on top of the availability argument. It is rare for the secure
> option to also be the cheap one; take the win.

Note the **$0.050/GB DTO on Global Accelerator**. If [[aws-global-accelerator]] is
chosen as the failover mechanism *and* Shield Advanced is subscribed, that stacks
on top of GA's own DT-Premium charge. Feed both into [[cost-model]].

### Shield Response Team and proactive engagement

*(Deepened below in the next pass — see [[#Still to research]].)*

## Does WAF work in front of Global Accelerator?

**No.** Global Accelerator does not appear on AWS's list of resources a web ACL can
protect — the list is CloudFront, API Gateway REST API, ALB, AppSync, Cognito user
pool, App Runner, Bedrock AgentCore Gateway, Verified Access and Amplify. There is
no `GLOBALACCELERATOR` scope and no association API for it.

AWS's own guidance is to put the WAF on the ALB **behind** the accelerator. From
[Use AWS WAF with Global Accelerator to block Layer 7 HTTP method and headers](https://repost.aws/knowledge-center/globalaccelerator-aws-waf-filter-layer7-traffic):
the accelerator routes traffic to an ALB, and the web ACL associated with that ALB
evaluates the request.

**Why this matters for the failover-mechanism decision in
[[aws-global-accelerator]]:**

- Choosing GA over Route 53 does **not** consolidate the WAF. You still have two
  regional web ACLs to keep identical. GA solves the DNS-TTL problem; it solves
  nothing about WAF.
- Choosing **CloudFront** as the entry point *does* consolidate it — one
  `CLOUDFRONT`-scope ACL in `us-east-1` covers both regions, because there is only
  one distribution. That is a real, underrated argument for the CloudFront-fronted
  design in [[aws-cloudfront]].
- **Shield Advanced, by contrast, *can* protect a standard accelerator**, and
  protecting the accelerator is a stronger position than protecting the two ALBs
  because the mitigation happens at the edge. So the correct combination is
  **Shield Advanced on the accelerator, WAF on each regional ALB** — two different
  services protecting two different layers in two different places.

> [!important] The one-line version for the design review
> "Global Accelerator gives you edge DDoS protection via Shield and a failover
> switch with no DNS. It gives you **nothing** on WAF — the web ACL stays on each
> regional ALB and still has to be duplicated. If you want one WAF instead of two,
> the answer is CloudFront, not Global Accelerator."

## Cost of WAF itself

From [AWS WAF pricing](https://aws.amazon.com/waf/pricing/):

| Dimension | Price |
|---|---|
| Web ACL | **$5.00 per month** (per web ACL, prorated hourly) |
| Rule | **$1.00 per month** per rule |
| Requests | **$0.60 per million** requests (standard inspection) |
| Bot Control (Common) | **$10.00/month** subscription per web ACL + **$1.00/million** beyond 10M free requests/month |
| Bot Control (Targeted) | **$10.00/million** after 1M free requests/month |
| Fraud Control — Account Takeover Prevention | **$10.00/month** per web ACL + tiered request pricing from **$1,000/million** |
| Fraud Control — Account Creation Fraud Prevention | **$10.00/month** per web ACL + tiered request pricing from **$1,000/million** |
| CAPTCHA | **$0.40 per thousand** attempts |
| AWS Marketplace managed rule groups | Seller-determined subscription + seller-determined per-million request fees |

### What a duplicated idle standby ACL actually costs

Per standby region, at **zero traffic**:

| Item | Monthly |
|---|---|
| 1 web ACL | $5.00 |
| 10 rules (a realistic count: 4–6 AMR groups + a few custom + 1–2 rate-based) | $10.00 |
| Requests at zero traffic | ~$0.00 |
| **Subtotal, idle standby ACL** | **≈ $15.00** |
| Bot Control, if used, per standby ACL | **+$10.00** |
| Fraud Control (ATP), if used, per standby ACL | **+$10.00** |
| **Realistic worst case per standby region** | **≈ $35.00** |

**Three standby regions: $45–$105/month.** For comparison, [[aws-alb-nlb]] puts
six idle load balancers at ~$99/month.

> [!note] A managed rule group counts as ONE rule
> A common budgeting error is to assume a managed rule group with 30 rules inside
> it costs $30. It does not — the rule group reference is one rule at $1.00/month.
> The free AWS Managed Rules groups (`AWSManagedRulesCommonRuleSet`,
> `AWSManagedRulesKnownBadInputsRuleSet`, `AWSManagedRulesAmazonIpReputationList`,
> `AWSManagedRulesSQLiRuleSet` and the rest of the non-intelligent-threat set) carry
> **no subscription fee** — only the paid ones (Bot Control, Fraud Control,
> Marketplace) do. This is what makes the duplicated standby ACL genuinely cheap.

> [!tip] Do not try to save the $15
> This is the same argument [[aws-alb-nlb]] makes about idle ALBs and it applies
> here with more force, because the thing you lose is not cost, it is **the
> security posture of the standby**. Deleting the standby ACL to save $15/month
> and recreating it at failover means a WAF-less window during your worst hour, a
> `terraform apply` in the recovery path, and no rule history. Cross-ref
> [[security-posture-of-the-standby]].

## Terraform implementation

*(Expanded in the next pass.)*

## Migration path from single-region

*(Expanded in the next pass.)*

## Failover procedure

*(Expanded in the next pass.)*

## Still to research

- Shield Response Team (SRT), proactive engagement, and whether the SRT can help
  during a *regional* failover at all.
- Shield Advanced health-check-based detection (`AssociateHealthCheck`) and what it
  means for a standby.
- WAF logging: destinations, per-region behaviour, redaction, the Firehose naming
  constraint, and where standby logs go.
- Full Terraform implementation — module signature, provider aliases, shared rule
  locals, drift-check CI.
- Migration path: adding a second regional ACL to a live estate; what forces
  replacement in the `aws_wafv2_*` resources.
- Failover procedure and readiness check.
- Failback.
- Gotchas list, decisions table, open questions.
- Managed rule group version pinning mechanics (`version_name`, `AWSManagedRulesX`
  versioning, the `UpdateManagedRuleSetVersionExpiryDate` deprecation path).

## Sources

- [Resources that you can protect with AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works-resources.html) — the verbatim same-Region requirement, the `us-east-1`/"Global (CloudFront)" equivalence, the full regional resource-type list, the CloudFront-exclusivity restriction, the Amplify exception, and the in-Region data-locality statement.
- [AWS WAF (chapter overview)](https://docs.aws.amazon.com/waf/latest/developerguide/waf-chapter.html) — the canonical list of protectable resource types and the ECS-via-ALB note.
- [How AWS WAF works](https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works.html) — web ACL / rule / rule group / WCU component model; confirms a rule is not a standalone AWS resource while a rule group is.
- [AWS WAF endpoints and quotas (General Reference)](https://docs.aws.amazon.com/general/latest/gr/waf.html) — **the `ca-west-1` parity check**: Calgary has `wafv2.ca-west-1.amazonaws.com`. Also the full quota table: 100 web ACLs/account/scope/Region, 100 IP sets, 10 regex pattern sets, 10,000 IPs per IP set, 5,000 WCU per web ACL, 10 rate-based statements per web ACL, 10,000 blockable IPs per rate-based rule, 100,000 req/s per web ACL, 8 KB ALB body inspection vs 64 KB for CloudFront/API Gateway.
- [Rate-based rule caveats](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-rate-based-caveats.html) — verbatim: not precise rate limiting; the look-back evaluation window; "up to several minutes before AWS WAF detects and rate limits them"; changing rate limit settings resets the counts and pauses limiting for up to a minute.
- [AWS WAF enhances rate-based rules to support configurable time windows](https://aws.amazon.com/about-aws/whats-new/2024/03/aws-waf-rate-based-rules-configurable-time-windows) — March 2024; establishes the 60/120/300/600-second `EvaluationWindowSec` options with 300 s default.
- [RateBasedStatement (WAFV2 API Reference)](https://docs.aws.amazon.com/waf/latest/APIReference/API_RateBasedStatement.html) — the `EvaluationWindowSec` parameter definition.
- [AWS Shield Advanced overview](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary.html) — verbatim: subscription billed per payer account, one fee per consolidated-billing family, the multi-organisation fee-waiver path; which standard WAF costs the subscription covers (web ACL, rules, base request price, up to 1,500 WCUs) and which it does not (Bot Control, CAPTCHA, >1,500 WCU, extended body inspection); the 150-WCU cost of the automatic mitigation rule group; 50 billion requests/month included.
- [Automating application layer DDoS mitigation with Shield Advanced](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-automatic-app-layer-response.html) — **the 24-hours-to-30-days baseline statement**, the Anti-DDoS AMR supersession of L7AM from 26 March 2026 ("within seconds rather than minutes"), the v2-web-ACL requirement, the reduced effectiveness on ALBs fronted by a CDN, and the protection-group caveat.
- [List of AWS resources that AWS Shield Advanced protects](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary-protected-resources.html) — CloudFront distributions, Route 53 hosted zones, Global Accelerator standard accelerators, Elastic IPs, ALB/CLB, and NLB/EC2 via Elastic IP.
- [AWS Shield pricing](https://aws.amazon.com/shield/pricing/) — **$3,000/month, 1-year commitment, billed per payer account**; DTO $0.025/GB for CloudFront and $0.050/GB for ELB/EC2/Global Accelerator.
- [AWS WAF pricing](https://aws.amazon.com/waf/pricing/) — $5.00/web ACL/month, $1.00/rule/month, $0.60/million requests, Bot Control $10/month + $1.00/million beyond 10M, Targeted Bot Control $10.00/million beyond 1M, Fraud Control ATP/ACFP $10/month + tiered from $1,000/million, CAPTCHA $0.40/thousand, Marketplace rule groups at seller pricing.
- [Use AWS WAF with Global Accelerator to block Layer 7 HTTP method and headers from access to application (AWS re:Post)](https://repost.aws/knowledge-center/globalaccelerator-aws-waf-filter-layer7-traffic) — AWS's own pattern for "WAF with Global Accelerator", which is WAF on the ALB endpoint behind the accelerator. Corroborates that there is no direct association.
- [AWS Fault Isolation Boundaries — Appendix B](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/appendix-b---edge-network-global-service-guidance.html) — Shield Advanced's `us-east-1` control-plane home region.

## Related notes

[[aws-cloudfront]] · [[aws-alb-nlb]] · [[aws-api-gateway]] · [[aws-route53]] · [[aws-global-accelerator]] · [[aws-acm]] · [[aws-eks]] · [[aws-iam]] · [[aws-s3]] · [[observability-multi-region]] · [[security-posture-of-the-standby]] · [[region-pair-selection]] · [[provider-aliases-vs-separate-stacks]] · [[module-patterns]] · [[failover-orchestration]] · [[cost-model]] · [[aws-regional-outages]] · [[lessons-and-antipatterns]] · [[data-residency]]
