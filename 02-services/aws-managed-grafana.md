---
title: Amazon Managed Grafana — Multi-Region
service: managed-grafana
tags: [service, multi-region, observability, grafana, amg, amp, prometheus, tooling]
status: partial
replication: none (no AWS-native cross-region capability for a workspace) | manual (dashboards-as-code is the only real mirror)
rpo_achievable: "workspace configuration: whatever is in Git — effectively 0 if dashboards-as-code, total loss if not. Metric/log data RPO is set by the data source, not by Grafana."
rto_achievable: "< 5 min to a pre-provisioned standby workspace, **if and only if** the sign-in path does not depend on the dead Region. Hours-to-never if the only workspace lives in the failed Region."
meets_targets: conditional — yes with a pre-provisioned workspace outside the primary Region plus SAML auth; **no for the CA pair, where AMG does not exist at all**
updated: 2026-09-21
---

# Amazon Managed Grafana — Multi-Region

> **The framing for this whole note: your monitoring must not live in the Region
> you are monitoring.**
>
> Every other note in this vault asks "how do I mirror this service into the
> standby Region". This note asks a different question, because observability is
> not part of the workload — it is the instrument you read *to decide whether to
> fail the workload over*. If `eu-west-1` is impaired, the dashboards, the alert
> rules, the alert evaluation engine and the login page you need in order to
> answer "is this bad enough to fail over?" are exactly the things you cannot
> reach, because you put them in `eu-west-1`.
>
> Amazon Managed Grafana (AMG) is a **regional** AWS service. It inherits that
> problem completely. There is no "multi-region workspace", no cross-region
> replication, no global endpoint. So the note is really about breaking shared
> fate between the observability plane and the thing it observes.
>
> See [[failover-orchestration]] for the decision procedure this feeds, and
> [[observability-multi-region]] for the wider picture (CloudWatch, logs, traces).

## TL;DR

- **AMG does not exist in Canada.** It is available in 12 Regions and neither
  `ca-central-1` nor `ca-west-1` is one of them. Verified twice, independently:
  the AWS regional services table and the AWS Price List API for offer code
  `AmazonGrafana` both list the same 12 Regions (`us-east-1`, `us-east-2`,
  `us-west-2`, `eu-west-1`, `eu-west-2`, `eu-central-1`, `ap-northeast-1`,
  `ap-northeast-2`, `ap-southeast-1`, `ap-southeast-2`, plus the two GovCloud
  Regions). **The CA deployment cannot be monitored from an in-country AMG
  workspace at all** — the nearest options are `us-east-1`/`us-east-2`/`us-west-2`,
  which is a [[data-residency]] decision, not an availability one. Amazon Managed
  Service for Prometheus (AMP), by contrast, *is* in both Canadian Regions. Feed
  this into [[region-pair-selection]].
- **A workspace is a black box of state with no export, no backup and no
  replication.** Dashboards, folders, alert rules, contact points, notification
  policies, silences, teams, data source definitions, service accounts and API
  keys all live inside the workspace. AWS gives you no `BackupWorkspace`,
  no `ExportWorkspace`, no snapshot, and no cross-region copy. If the workspace's
  Region is gone, so is all of it — at precisely the moment you need it.
  **Dashboards-as-code is therefore not a nice-to-have, it is the entire
  mitigation.** It is what makes a second workspace a 10-minute `terraform apply`
  instead of a week of rebuilding from memory.
- **One pane of glass across Regions is real — for data.** A single CloudWatch
  data source can query metrics in any Region (the Region is a per-query field,
  and `defaultRegion` is just a default), and with CloudWatch cross-account
  observability it can query other accounts too. So a workspace in a third Region
  can genuinely see both primary and standby. The limit is that cross-account
  observability via OAM is **single-Region** per link ("monitor and troubleshoot
  applications across multiple regional accounts"), so the cross-account and
  cross-Region dimensions do not compose for free — see below.
- **The sharp finding is authentication, and it is worse than the usual
  "Identity Center is single-region" claim.** IAM Identity Center *can* now be
  replicated to additional Regions (GA, 2026). But AWS's own application table
  says Amazon Managed Grafana **does not support deployment in additional Regions
  of IAM Identity Center**, and AWS states that "Each AWS managed application
  connects to a specific IAM Identity Center Region during deployment. The
  application then depends on that Region for user sign-in, even if your IAM
  Identity Center is enabled in multiple Regions." So if your Identity Center
  primary Region is the Region that just died, **your engineers cannot log in to
  Grafana at all — including the standby workspace in the other Region.**
  The fix is to authenticate the standby workspace with **SAML direct to the
  corporate IdP**, which AMG supports as a first-class alternative and which has
  no Identity Center dependency. Cross-ref [[security-posture-of-the-standby]]
  and [[aws-iam]].
- **Cost is not the constraint.** AMG is billed per *active* user per workspace
  per month: $9 editor/administrator, $5 viewer, and a floor of one editor
  licence per workspace "even if no users log in". A second, idle,
  fully-provisioned standby workspace therefore costs **$9/month**, plus $9/month
  for the Terraform service account that keeps its dashboards in sync. Roughly
  **$18/month to remove your single largest 3am blind spot.** There is no
  hourly/instance charge and no charge for dashboards, alert rules or data
  sources.
- **Recommendation: one AMG workspace in a third Region, driven entirely from
  Git, authenticated by SAML, alerting on CloudWatch alarms rather than
  Grafana-evaluated rules for the failover-decision signals.** Second choice if
  a third Region is politically impossible: a workspace in each of the pair, same
  Git source, SAML on both. And seriously consider option (d) — put the
  failover-decision dashboard on something that is not AWS at all — for the
  handful of signals that drive the decision. Details and honest costs below.

## Does this service cross Regions at all?

No. Not in any sense.

| Dimension | Behaviour |
|---|---|
| Workspace | Regional resource. ID like `g-abc12345`, endpoint `https://g-abc12345.grafana-workspace.<region>.amazonaws.com`. |
| Cross-region replication | **None.** No AWS-native mechanism of any kind. |
| Backup / export / snapshot | **None in the AMG API.** The only export path is the Grafana HTTP API (or Terraform state), driven by you. |
| Global endpoint | None. |
| Addressable from another Region? | The *control plane* (`grafana.<region>.amazonaws.com`) is regional. The *workspace URL* is reachable from anywhere on the internet — but it is served out of its home Region, so a regional impairment takes it with it. |
| Data plane reach | This is the one thing that *does* cross: a workspace can query data sources in other Regions and other accounts. |

The distinction that matters: **AMG's reach crosses Regions, AMG's state does
not.** A workspace in `eu-west-2` can happily graph `eu-west-1` metrics. What it
cannot do is *become* the `eu-west-1` workspace, or inherit its dashboards.

### Region availability — verified

Checked against the AWS regional services table (`source:version 20251113063700`)
and corroborated by the AWS Price List API.

| Region | Amazon Managed Grafana | Amazon Managed Service for Prometheus |
|---|---|---|
| `eu-west-1` (Ireland) | ✅ | ✅ |
| `eu-west-2` (London) | ✅ | ✅ |
| `us-east-1` (N. Virginia) | ✅ | ✅ |
| `us-west-2` (Oregon) | ✅ | ✅ |
| `us-east-2` (Ohio) | ✅ | ✅ |
| `eu-central-1` (Frankfurt) | ✅ | ✅ |
| `ca-central-1` (Montreal) | ❌ **not available** | ✅ |
| `ca-west-1` (Calgary) | ❌ **not available** | ✅ |

Full AMG list: `ap-northeast-1`, `ap-northeast-2`, `ap-southeast-1`,
`ap-southeast-2`, `eu-central-1`, `eu-west-1`, `eu-west-2`, `us-east-1`,
`us-east-2`, `us-gov-east-1`, `us-gov-west-1`, `us-west-2`. Twelve Regions.

**Consequences for the CA pair:**

1. There is no in-country managed Grafana. Monitoring the Canadian deployment
   with AMG means a workspace in the US reading Canadian operational metrics.
   Metric names, dimension values, log samples and trace attributes routinely
   carry customer-identifying strings. That is a [[data-residency]] question for
   legal, not an engineering one. Raise it before building anything.
2. The alternatives for Canada are: self-hosted Grafana on the Canadian EKS
   clusters (dies with the cluster — see [[eks-workload-delivery]]), Grafana
   Cloud or another vendor with a Canadian region (see
   [[observability-vendors-multi-region]]), or CloudWatch dashboards only
   (in-Region, and equally dead when the Region is).
3. This is now the **third** Calgary parity failure recorded in this vault,
   after Cognito multi-region user pools and OpenSearch cross-cluster
   replication. Unlike those two, this one hits `ca-central-1` as well —
   it is not a Calgary problem, it is a Canada problem.

### What survives a Region failure, and what does not

Assume the workspace lives in `eu-west-1` and `eu-west-1` is impaired.

| Thing | Survives? | Notes |
|---|---|---|
| Dashboards | ❌ | Stored in the workspace. Unreachable, not deleted. |
| Alert rules (Grafana-managed) | ❌ | Stored in the workspace; evaluation happens in-Region. |
| Alert evaluation / notifications | ❌ | Silently stops. No notification that notifications stopped. |
| Data source definitions | ❌ | Workspace state. |
| Service accounts / API keys | ❌ | Workspace state. |
| Users and role assignments | ❌ | Workspace state (mapping), plus the IdP side. |
| The metric data itself | depends | CloudWatch metrics for `eu-west-1` resources live in `eu-west-1` — also unreachable. Metrics remote-written to an AMP workspace elsewhere survive. |
| Your ability to log in | ❌ if Identity Center primary Region is `eu-west-1` | See the authentication section. This is the one that surprises people. |
| A second workspace in `eu-west-2`/`eu-central-1` | ✅ | Provided its auth path is independent. |

## What actually lives inside a workspace

This is the inventory you have to reproduce elsewhere, and it is longer than
"dashboards":

- **Dashboards** and their JSON models, including library panels.
- **Folders**, and folder-level permissions.
- **Data source definitions** — type, URL/ARN, `defaultRegion`, assume-role ARN,
  auth provider, and any secure JSON (secrets are write-only via the API).
- **Alert rules**, **contact points**, **notification policies**, **mute
  timings**, **message templates**, **silences**.
- **Teams** and team memberships, **users**, **role assignments**.
- **Service accounts** and their tokens; legacy **API keys**.
- **Plugins** installed via plugin management (if enabled).
- **Annotations** (including alert-state annotations) and **snapshots** — these
  are genuinely unreproducible state and are lost. Accept that.
- **Playlists**, **preferences**, **organisation settings**.

Everything in that list except annotations, snapshots and alert-state history is
expressible as code. That is the whole strategy: make the workspace disposable
by making its contents declarative, so that "stand up another one" is a
`terraform apply` rather than an archaeology project.

## Replication / mirroring options

There is nothing native to replicate. So the real question is *where you put
workspaces*, and how you keep their contents identical.

### (a) One workspace in a third Region, querying both

A single AMG workspace in a Region that is neither the primary nor the standby —
for the EU pair, `eu-central-1`; for the US pair, `us-east-2`. It holds
CloudWatch data sources pointed at both `eu-west-1` and `eu-west-2` (and both
accounts if the estate is multi-account), plus AMP/X-Ray/OpenSearch/Athena data
sources.

| | |
|---|---|
| **Cost** | One workspace. `$9` floor + `$9` per actual editor + `$5` per viewer per month. No duplication. |
| **Operational load** | One set of dashboards. No drift. Dashboards-as-code still worth doing, but the pressure is lower. |
| **What it buys** | The observability plane has no shared fate with either half of the pair. During a `eu-west-1` event the dashboards are up, the alert rules are still being evaluated, and you can see both sides while you decide. |
| **What it costs you** | Every query is cross-Region: slower panels, and cross-Region API calls. It adds a third Region to the blast radius list, the Terraform estate and the compliance surface. It is also a new Region with its own quotas, VPC endpoints and IAM. |
| **Honest weakness** | It is still one Region. If `eu-central-1` has a bad day you are blind — but you are blind while the *product is fine*, which is a far better failure mode than being blind during an outage. And it is a Region you can choose for correlation properties rather than latency. |
| **CA pair** | Not possible in-country. The "third Region" for Canada is a US Region. |

### (b) A workspace in each Region of the pair, same dashboards

`eu-west-1` and `eu-west-2` each get a workspace, both provisioned from the same
Git repo.

| | |
|---|---|
| **Cost** | Two workspaces. The idle one costs the `$9`/month editor floor, plus `$9`/month if a Terraform service account with editor rights touches it that month (it will — that is how the dashboards get there). Call it **$18/month per pair per environment**. |
| **Operational load** | Real drift risk unless 100% of content is code. Anyone who edits a dashboard in the UI on the primary creates an inconsistency that will only be discovered during an incident. |
| **What it buys** | The standby is warm in exactly the way the rest of the estate is warm. There is no third Region to justify to anyone. It is the shape that matches the rest of this vault. |
| **Honest weakness** | **During a primary-Region event you are logging into the standby workspace for the first time ever**, possibly at 3am, possibly with an auth path you have never exercised. And if Identity Center's primary Region is the dead one, you cannot log in at all (see below). Option (b) *only works* with SAML auth and with periodic drills — see [[dr-testing-and-gamedays]]. |

### (c) AMG in the standby Region only

One workspace, in `eu-west-2`, used all the time, monitoring the `eu-west-1`
primary across the Region boundary.

| | |
|---|---|
| **Cost** | Cheapest of the "correct" options — one workspace. |
| **What it buys** | Zero shared fate with the primary. The dashboards you use every day are the dashboards you use during the outage — no cold path. This is the single biggest advantage and it should not be underestimated: an untested standby is a liability, a daily-driver is not. |
| **Honest weakness** | You have now given the standby Region a production-critical role while the primary is healthy, which cuts against the "standby is dormant" posture in [[security-posture-of-the-standby]] — the standby's IAM, network and availability now matter every day. And if you fail over *into* `eu-west-2`, your observability is suddenly in the same Region as the workload again, with shared fate restored exactly when you are running degraded. |
| **Verdict** | Better than (b) for day-one exercise value, worse than (a) for the post-failover state. |

### (d) Don't use AMG for the failover decision

Keep AMG for everyday dashboards, but put the small set of signals that drive
the go/no-go decision somewhere with no AWS regional dependency: an external
synthetic monitor, a status dashboard on a different cloud, PagerDuty/Slack fed
by an external prober, or a vendor SaaS (see
[[observability-vendors-multi-region]]).

| | |
|---|---|
| **Cost** | A vendor bill, or a small externally-hosted prober. |
| **What it buys** | The decision instrument has genuinely no shared fate with AWS Regions, AWS IAM, AWS Identity Center or the AWS console. During a broad regional event — the kind where the console itself is degraded — this is the only option in this list that is definitely still working. |
| **Honest weakness** | Another vendor, another contract, another set of credentials to manage, another thing to keep in sync. And it does not replace deep debugging dashboards; it only answers "is the primary serving?". |

### Recommendation

**Do (a) *and* a thin slice of (d).**

- One AMG workspace per pair, in a third Region, provisioned entirely from Git,
  authenticated by SAML (not Identity Center), with CloudWatch data sources for
  both Regions and a single AMP workspace fed by remote-write from both.
- Plus a handful of **external** black-box probes (from outside AWS) whose output
  lands in the on-call channel directly. The failover decision needs about five
  signals — is the public endpoint answering, error rate, p99, replication lag,
  and queue depth. Those five do not need Grafana. Everything *after* the
  decision does.
- Fall back to (b) if a third Region is unacceptable to compliance or to the
  Terraform estate's shape — but only with SAML auth and a quarterly drill that
  logs into the standby workspace for real.

For the **CA pair**, (a) is unavailable in-country; the recommendation becomes
either "US workspace, with legal sign-off on what metric metadata leaves Canada"
or "no AMG for Canada — use in-Region CloudWatch dashboards plus external
probes". Do not let this decision be made implicitly by a Terraform module that
silently skips Canada.

## Data sources across Regions

The good news is that Grafana's AWS data sources were designed for this.

### CloudWatch

- **Region is a per-query field.** The data source has a `defaultRegion` in its
  `jsonData`, but every panel query can override it. One data source in a third
  Region can serve panels for `eu-west-1` and `eu-west-2` side by side. There is
  no need for one data source per Region (though naming them per-Region is often
  clearer for dashboard variables).
- **Cross-account** works two ways:
  1. **Assume-role ARN** on the data source. AWS's guidance: "When you use an
     Assume Role ARN, attach query permissions to the assumed role. The primary
     credentials only need permission to perform `sts:AssumeRole`." With
     `SERVICE_MANAGED` permissions and an `ORGANIZATION` account access type,
     AMG uses CloudFormation StackSets to deploy the roles across the org.
  2. **CloudWatch cross-account observability (OAM)**, which AMG supports
     natively: add `oam:ListSinks` and `oam:ListAttachedLinks` to the workspace
     role and the CloudWatch plugin surfaces linked source accounts in the query
     editor. Requires Grafana workspace version 9 or later.
- **The limitation that matters:** AWS describes cross-account observability as
  working "across multiple regional accounts" and the Grafana docs note you
  retrieve metrics and logs across accounts **in a single Region**. OAM links
  are per-Region. So cross-account and cross-Region do not multiply: to see
  account B's `eu-west-2` metrics from a workspace in `eu-central-1`, you either
  need an OAM sink in `eu-west-2` that the workspace can reach, or the classic
  assume-role path. In a multi-account estate, plan the assume-role path — it
  composes across both dimensions and OAM does not.
- **VPC caveat:** if the workspace is in a VPC (`vpc_configuration`), OAM has no
  VPC endpoint, so you need a NAT gateway for the workspace to reach the OAM
  APIs. AWS says this explicitly. That NAT gateway is a standing cost and a
  standing dependency.
- **EC2 instance attributes cannot be queried cross-account** — they come from
  the EC2 API, not the CloudWatch API.

### Amazon Managed Service for Prometheus (AMP)

AMP workspaces are **also regional**, with the same "no cross-region
replication" story. But AMP has something AMG does not: an ingestion path you
control.

- **Remote-write from both Regions into one AMP workspace** is the pattern that
  actually works. Prometheus/ADOT/the AMP managed collector in `eu-west-1` and
  in `eu-west-2` both `remote_write` to a single AMP workspace in the third
  Region. You then get genuinely unified PromQL — `sum by (region) (...)` across
  the pair, in one query, with no federation gymnastics.
- **Failure mode, and it is the important one:** remote-write is a *push* from
  the monitored Region. If the primary Region is impaired in a way that takes out
  the collectors (EKS control plane, node networking, NAT), the push stops and
  the standby AMP workspace shows a **flat line, not an alert**. A gap in a graph
  is indistinguishable from "everything is fine and quiet" to a naive alert rule.
  Every dashboard built this way needs an explicit staleness/heartbeat rule:
  alert when `absent()` or when the newest sample is older than N minutes.
  Prometheus's remote-write WAL will buffer and backfill when connectivity
  returns, which is good for the historical record and bad for alerting — you
  can get a flood of late-arriving samples that re-evaluate rules oddly.
- **Cross-Region data transfer** on remote-write is real money at high
  cardinality. It is inter-Region egress from the monitored Region. Budget it in
  [[cost-model]]; do not let it be a surprise line item.
- **AMP query federation / cross-region query:** AMP exposes a
  Prometheus-compatible query endpoint per workspace. There is no AWS-managed
  "query these two AMP workspaces as one". You can point Grafana at both
  workspaces as two data sources and use a mixed-datasource dashboard, which is
  fine for side-by-side panels and poor for a single aggregate expression. This
  is why single-sink remote-write beats two workspaces.
- **AMP *is* available in `ca-central-1` and `ca-west-1`** — so for Canada, the
  metrics pipeline can stay in-country even though AMG cannot.
- AMP alerting (ruler + alert manager) runs **inside the AMP workspace's
  Region**. Same shared-fate problem, same fix: put the AMP workspace that
  evaluates the critical rules outside the monitored Region.

### Other data sources

| Data source | Cross-Region from one workspace? | Notes |
|---|---|---|
| CloudWatch metrics/logs | Yes, per-query Region | The workhorse. |
| AWS X-Ray | Region-scoped data source | X-Ray traces are per-Region; a trace that crosses Regions is two traces. One data source per Region. |
| Amazon OpenSearch | Yes, by endpoint | The data source points at a domain endpoint, so "cross-Region" just means "another endpoint". See [[aws-opensearch]] for why the standby domain's saved objects are a separate problem. |
| Amazon Athena | Region-scoped | Athena workgroup + Glue catalog are regional; S3 data may be replicated. Good for post-hoc log analysis over S3, useless during an outage if the catalog is in the dead Region. |
| Amazon Timestream | Region-scoped | Same shape. |
| Amazon Redshift | Region-scoped | Same shape. |
| Prometheus (AMP or self-hosted) | By endpoint | See above. |

**Rule of thumb:** a data source that is addressed by a *URL/endpoint* crosses
Regions trivially. A data source that is addressed by *"the current Region"*
needs one definition per Region. CloudWatch is the exception that does both.

## Authentication — the chicken-and-egg problem

This is the most important section in the note.

### What AMG requires

AMG does not accept IAM users or roles for workspace sign-in. AWS is explicit:
"Amazon Managed Grafana does not support the use of IAM users and roles to assign
permissions within an Amazon Managed Grafana workspace." Your only two options
are:

- **SAML 2.0** direct to your IdP (Okta, Entra ID, PingOne, OneLogin, CyberArk
  are the tested ones), or
- **AWS IAM Identity Center**.

A workspace can have one or both (`authentication_providers = ["AWS_SSO",
"SAML"]`).

### The Identity Center dependency, stated precisely

The common claim is "IAM Identity Center is single-region". As of the 2026 GA of
multi-Region support, **that claim is now out of date, and the real situation is
worse for Grafana specifically.** Three facts, all from AWS documentation:

1. **Identity Center can be replicated.** "When you enable an organization
   instance of IAM Identity Center, you choose a single AWS Region (primary
   Region). You can replicate this instance to additional AWS Regions during
   instance creation or after... IAM Identity Center automatically replicates
   workforce identities, permission sets, user and group assignments, sessions,
   and other metadata from the primary Region to the chosen additional Regions."
   The stated benefit is that "Your workforce can access their AWS accounts even
   if the IAM Identity Center instance experiences a service disruption in its
   primary Region."
2. **But each AWS managed application is pinned to one Identity Center Region.**
   "Each AWS managed application connects to a specific IAM Identity Center
   Region during deployment. The application then depends on that Region for
   user sign-in, even if your IAM Identity Center is enabled in multiple Regions.
   If your IAM Identity Center is experiencing a disruption in that Region,
   users might not be able to access AWS managed applications connected to the
   Region."
3. **And Amazon Managed Grafana is not one of the applications that can be
   deployed in an additional Region.** In AWS's table of managed applications,
   the "Supports deployment in additional Regions of IAM Identity Center" column
   for Amazon Managed Grafana reads **No**. (The same row reads "Yes" for
   customer-managed KMS key support — note the page renders a negative icon next
   to that "Yes", which appears to be a documentation bug; the text is what the
   table's own legend describes. Verify this before it becomes load-bearing.)

**Put together:** with Identity Center auth, every AMG workspace you own connects
to the Identity Center **primary** Region for sign-in. Replicating Identity
Center to a second Region improves your *AWS account* access resiliency; it does
**not** give a Grafana workspace a Region-local sign-in path. So:

- Identity Center primary in `eu-west-1`, AMG workspace in `eu-west-1`:
  `eu-west-1` dies → no dashboards, no login. Obvious.
- Identity Center primary in `eu-west-1`, standby AMG workspace in `eu-west-2`:
  `eu-west-1` dies → **the standby workspace is up, and you still cannot log in
  to it.** This is the trap. A perfectly provisioned warm standby workspace,
  unreachable, because the login depends on the Region you are failing away
  from.
- Identity Center primary in `eu-central-1` (a third Region), AMG workspace
  anywhere: `eu-west-1` dies → login works. But you have just made every
  Identity Center administrative operation for the whole organisation depend on
  a Region chosen for Grafana's benefit, and **the primary Region cannot be
  changed after Identity Center is enabled** — "the primary Region cannot be
  changed after IAM Identity Center is enabled". For an existing estate this is
  effectively a rebuild of Identity Center. It is not a realistic remedy.

### The answer: SAML on the standby, always

**Use SAML 2.0 directly on at least the workspace you intend to read during a
failover.** It goes browser → workspace → corporate IdP → workspace. No Identity
Center, no AWS regional identity dependency at all — the only AWS component in
the path is the workspace itself, in a Region you chose.

Caveats to write into the runbook:

- AMG **does not support IdP-initiated login**: "Amazon Managed Grafana does not
  currently support IdP initiated login for workspaces. You should set up your
  SAML applications with a blank Relay State." So the bookmark your on-call needs
  is the *workspace URL*, not a tile in the Okta dashboard. Put the literal URL
  in the runbook. People will try the IdP portal first and it will not work.
- AMG supports SP-initiated requests only, HTTP-POST and HTTP-Redirect SP→IdP,
  HTTP-POST IdP→SP, and "signed and encrypted assertions, but does not support
  signed or encrypted requests."
- Creating a SAML workspace requires the creating principal to have
  `AWSGrafanaAccountAdministrator`.
- Your IdP is now the single point of failure for observability sign-in. That is
  a *better* single point of failure than an AWS Region during an AWS Region
  outage, but it is not zero. Keep one break-glass local admin path documented —
  and note that AMG gives you no local user database, so "break-glass" here means
  "an IdP account in a separate IdP tenant/realm", or the AWS console path to
  re-point the SAML config.
- AWS's own advice, in the Identity Center resiliency page, is to "set up AWS
  break-glass access... to maintain AWS access for a small group of privileged
  users during events such as a service disruption in the external IdP."

### Knock-on for the wider runbook

This is not only a Grafana finding. Any AWS managed application in the table with
"No" in the additional-Regions column has the same pinning. And more generally:
**if your engineers reach the AWS console via Identity Center, your ability to
run a failover at all depends on the Identity Center primary Region.** Identity
Center multi-Region replication does fix *that* narrower case ("your workforce
can access their AWS accounts... using already provisioned permissions"), but
prerequisites apply and one of them matters here:

- Organization instance only (not account instances).
- Identity source must be an external IdP or the Identity Center directory —
  **not** AWS Managed Microsoft AD.
- **"Multi-Region support is available in commercial Regions enabled by default
  in your AWS account. Opt-in Regions are not currently supported."** →
  **`ca-west-1` is an opt-in Region, so Identity Center cannot be replicated to
  Calgary.** Another CA-pair parity gap, consistent with the OpenSearch CCR
  finding already in this vault.
- Requires a multi-Region customer-managed KMS key — see
  [[kms-when-to-use-multi-region-keys]].
- The external IdP must support multiple ACS URLs (Okta, Entra ID, PingFederate,
  PingOne, JumpCloud are named as supporting this; Google Workspace is named as
  not).

Write this up properly in [[security-posture-of-the-standby]] and
[[failover-orchestration]]; the "can the humans log in" question belongs in the
runbook's first five steps, not its appendix.

## Alerting

Three mechanisms, with very different failure characteristics.

| Mechanism | Where it evaluates | Survives its Region dying? | Use for |
|---|---|---|---|
| **Grafana-managed alert rules** (unified alerting, Grafana 9+) | Inside the AMG workspace | ❌ | Rich, multi-datasource, expression-based alerts for everyday work. |
| **CloudWatch alarms** | In the Region of the metric — **always** | ❌ | See the hard limitation below. An alarm cannot watch another Region's metric. |
| **AMP ruler + alert manager** | Inside the AMP workspace's Region | ❌ for that Region | Prometheus-native rules; put the workspace outside the monitored Region and they survive it. |

Key points:

- **Alert evaluation stops silently when the workspace's Region is impaired.**
  There is no "my alerting stopped" alert. You must monitor the monitor: a
  heartbeat alert that fires *from a different Region* if it stops hearing from
  the workspace, or a dead-man's-switch pattern (a rule that always fires into a
  receiver which pages if it *stops* receiving). Grafana-managed alerting has no
  built-in equivalent, so this is plumbing you build.
- **Contact points are workspace state.** The Slack webhook, the PagerDuty
  integration key, the SNS topic ARN — all of it lives inside the workspace, and
  all of it must be re-created in the standby workspace. Secrets in contact
  points are write-only over the API, which means Terraform can set them but not
  read them back for drift detection; keep them in Secrets Manager (already
  cross-region replicated per the research brief) and inject.
- **SNS topics referenced by contact points are regional.** A contact point in a
  `eu-central-1` workspace pointing at an `eu-west-1` SNS topic is a shared-fate
  bug hiding in a config field. See [[aws-sns]] / [[aws-eventbridge]].
- **Unified alerting can be switched off** via the workspace `configuration`
  JSON (`"unifiedAlerting": {"enabled": false}`). Do not; classic alerting is the
  legacy path. If you disable it you lose the provisioning API that the Grafana
  Terraform provider uses for alert resources.

### Alert noise from an idle standby

This is the operational trap that makes teams turn off the thing that would have
saved them.

A warm standby that is scaled to zero — zero-replica deployments, zero-desired
node groups, an RDS read replica with no traffic, empty queues — will trip
every threshold rule written for a busy Region: "pods available < 1",
"request rate == 0", "no healthy targets", "queue consumers == 0". Within a week
someone silences the standby. Within a month the silence is permanent and
undocumented. Then, during the failover you finally perform, the standby is
promoted and **the silence is still in place**, so the alerts that would tell you
the promotion went wrong never fire.

Mitigations, in order of preference:

1. **Make "expected state" a label, not a silence.** Tag standby resources with
   `role=standby` and write rules as `... and on() role_is_active == 1`, driven
   by a single authoritative "which Region is active" metric. The rule set is
   identical in both Regions; only the activation metric differs. When you fail
   over you flip one metric and the whole alert posture flips with it.
2. **Separate rule groups per role.** `rules/active/*.yaml` and
   `rules/standby/*.yaml`, with the standby set containing only the alerts that
   *should* fire for a healthy standby: replication lag, replica health,
   certificate expiry, capacity reservation, drift.
3. **Time-bounded silences only.** If you must silence, use Grafana/Alertmanager
   silences with an explicit end time, never indefinite. Alertmanager silences
   expire by design; use that.
4. **Alert on the standby's readiness, not its activity.** The question for a
   standby is never "is it serving traffic" (it is not) but "could it, within the
   RTO?" — replication lag under budget, node group warm, image present in the
   standby ECR, secret replicated, certificate valid. Those alerts are
   meaningful every day and they are the ones that catch a standby that has
   silently rotted.

Cross-ref [[lessons-and-antipatterns]] — "the standby that had been silently
broken for months" is the single most common multi-region failure story and
alert hygiene is what prevents it.

## RPO / RTO analysis

Against the vault's targets of RPO 2h / RTO 15m. Note that for a monitoring tool
these mean something slightly different: the "data" is configuration, and the
"recovery" is *an engineer seeing a working dashboard*.

**RPO — workspace configuration**

| Approach | RPO |
|---|---|
| Dashboards edited in the UI, no code | **Total loss.** Unbounded. There is no backup API. |
| Dashboards-as-code in Git, applied by CI | **0** — Git is the source of truth and lives outside AWS (or in CodeCommit/GitHub, either way not in the workspace). |
| Hybrid (code + UI edits) | Equal to "time since the last UI edit was reverse-engineered back into code", i.e. unknown. Do not do this; set `disable_provenance = false` so Terraform-provisioned alerting objects are read-only in the UI. |

**RPO — the metric data itself** is not AMG's problem. CloudWatch metrics are
regional and stay with their Region; AMP data has the RPO of your remote-write
lag (seconds) if you push to a workspace outside the monitored Region.
Comfortably inside 2h either way. See [[observability-multi-region]].

**RTO — time to a working dashboard**

| Starting point | Time to first useful dashboard | Meets 15m? |
|---|---|---|
| Pre-provisioned workspace in a third Region, SAML, dashboards already applied | Seconds. You just open the bookmark. | ✅ |
| Pre-provisioned workspace in the standby Region, SAML | Seconds — but first-ever use, so budget for surprises. | ✅ with drills |
| Pre-provisioned workspace, Identity Center auth, IdC primary = dead Region | **Never** — you cannot sign in. | ❌ |
| No standby workspace; create one at failover time from Git | Workspace creation is a control-plane operation measured in minutes, plus SAML config, plus role association, plus the dashboards-as-code apply. Realistically 15–40 minutes with a following wind, and you are doing it under incident pressure with an untested path. | ❌ |
| No standby workspace, no dashboards-as-code | Days. | ❌ |

**Where the time actually goes** if you have not pre-provisioned: workspace
creation, the IAM role and StackSet propagation for `SERVICE_MANAGED`
permissions, SAML metadata exchange, and then discovering that your data source
UIDs are different in the new workspace so every dashboard's panel queries point
at nothing. That last one is the silent killer and it is why data source UIDs
must be pinned in code (see Terraform below).

**Conclusion: AMG meets RTO 15m only as a pre-provisioned, pre-authenticated,
pre-populated workspace outside the monitored Region.** There is no
provision-at-failover story that is credible at 15 minutes.

## Warm standby shape

What exists while the primary is healthy, under the recommended (a) shape:

| Component | State while primary is healthy | Idle cost |
|---|---|---|
| AMG workspace (third Region) | Running, in daily use | `$9`/mo floor + per-active-user |
| AMG workspace (standby Region), if using shape (b) | Running, nobody logs in | `$9`/mo |
| Terraform service account + token | Exists; used by CI each apply | `$9`/mo (counts as an active user in any month it is used) |
| Dashboards / folders / alert rules | Applied from Git, identical both sides | $0 |
| CloudWatch data sources | Defined, pointing at both Regions | $0 to define; per-query API cost |
| AMP workspace (third Region) | Receiving remote-write from both Regions | Ingestion + storage + query, see Cost |
| AMP workspace in the standby Region | Usually unnecessary if remote-writing to a single sink | — |
| Contact points / notification policies | Applied from Git | $0 |
| Alert rules for standby health | Active, evaluating | Query cost only |

Nothing here is "scaled to zero" in the EC2 sense — AMG has no capacity dial.
The only lever is *active users*, and that lever is nearly free.

## Terraform implementation

The estate is a cookiecutter-templated multi-env monorepo (see
[[module-patterns]] and [[provider-aliases-vs-separate-stacks]]). Two providers
are in play and they are *layered*: the `aws` provider creates the workspace, and
the `grafana` provider fills it. That layering has a bootstrapping wrinkle
because the `grafana` provider needs a token that does not exist until the `aws`
provider has run.

### Layer 1 — the workspace (aws provider, aliased per Region)

```hcl
terraform {
  required_version = ">= 1.9"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0" # 5.x also fine; service accounts need >= 5.62-era
    }
    grafana = {
      source  = "grafana/grafana"
      version = "~> 3.0"
    }
  }
}

provider "aws" {
  alias  = "observability"   # the third Region: eu-central-1 / us-east-2
  region = var.observability_region
}

provider "aws" {
  alias  = "primary"
  region = var.primary_region
}

provider "aws" {
  alias  = "standby"
  region = var.standby_region
}
```

Module signature — one module, instantiated once per *pair*, not once per
Region, because the whole point is that the workspace is not in either Region:

```hcl
# modules/observability-grafana/variables.tf

variable "name" {
  description = "Workspace name, e.g. \"helios-eu-prod\"."
  type        = string
}

variable "env" {
  description = "Cookiecutter environment slug (dev/stage/prod)."
  type        = string
}

variable "monitored_regions" {
  description = <<-EOT
    The Regions this workspace is responsible for. Used to generate one
    CloudWatch data source per Region with a pinned UID, and to build the
    dashboard "region" template variable. Deliberately a list, not a
    primary/standby pair, so a third Region can be added without a rewrite.
  EOT
  type        = list(string)
}

variable "monitored_account_ids" {
  description = "Accounts whose CloudWatch data this workspace may read. Empty list = current account only."
  type        = list(string)
  default     = []
}

variable "authentication_providers" {
  description = <<-EOT
    ["SAML"] is the default and the recommendation: it removes the IAM Identity
    Center regional sign-in dependency. Use ["AWS_SSO"] only for workspaces that
    are never needed during a failover.
  EOT
  type        = list(string)
  default     = ["SAML"]

  validation {
    condition     = length(setsubtract(var.authentication_providers, ["SAML", "AWS_SSO"])) == 0
    error_message = "authentication_providers must be a subset of [\"SAML\", \"AWS_SSO\"]."
  }
}

variable "saml_idp_metadata_url" {
  description = "IdP metadata URL. Required when SAML is in authentication_providers."
  type        = string
  default     = null
}

variable "saml_admin_role_values" {
  type    = list(string)
  default = ["grafana-admin"]
}

variable "saml_editor_role_values" {
  type    = list(string)
  default = ["grafana-editor"]
}

variable "grafana_version" {
  description = "Pin it. Workspace upgrades are one-way; see Gotchas."
  type        = string
  default     = "10.4"
}

variable "amp_workspace_arns" {
  description = "AMP workspaces to expose as data sources, in any Region."
  type        = list(string)
  default     = []
}

variable "vpc_configuration" {
  description = "Optional. Setting this requires a NAT gateway for OAM and for any public data source. Leave null unless you have private data sources."
  type = object({
    security_group_ids = list(string)
    subnet_ids         = list(string)
  })
  default = null
}
```

```hcl
# modules/observability-grafana/main.tf

resource "aws_grafana_workspace" "this" {
  name                     = var.name
  description              = "Observability for ${join(", ", var.monitored_regions)} (${var.env})"
  account_access_type      = length(var.monitored_account_ids) > 0 ? "ORGANIZATION" : "CURRENT_ACCOUNT"
  authentication_providers = var.authentication_providers
  permission_type          = "CUSTOMER_MANAGED"
  role_arn                 = aws_iam_role.workspace.arn
  grafana_version          = var.grafana_version

  data_sources = ["CLOUDWATCH", "PROMETHEUS", "XRAY", "ATHENA", "AMAZON_OPENSEARCH_SERVICE"]

  dynamic "vpc_configuration" {
    for_each = var.vpc_configuration == null ? [] : [var.vpc_configuration]
    content {
      security_group_ids = vpc_configuration.value.security_group_ids
      subnet_ids         = vpc_configuration.value.subnet_ids
    }
  }

  lifecycle {
    # Deleting a workspace deletes every dashboard in it. Even with
    # dashboards-as-code, a rebuild loses annotations and alert history.
    prevent_destroy = true
  }
}

resource "aws_grafana_workspace_saml_configuration" "this" {
  count = contains(var.authentication_providers, "SAML") ? 1 : 0

  workspace_id       = aws_grafana_workspace.this.id
  idp_metadata_url   = var.saml_idp_metadata_url
  admin_role_values  = var.saml_admin_role_values
  editor_role_values = var.saml_editor_role_values

  # Map your IdP's claim names.
  role_assertion  = "role"
  email_assertion = "email"
  login_assertion = "login"
  name_assertion  = "displayName"
  groups_assertion = "groups"
}
```

The IAM role is where cross-Region and cross-account reach is granted. Note that
CloudWatch, Logs, X-Ray and Tag APIs are `Resource = "*"` by nature — they are
Region-scoped by *endpoint*, not by ARN, so the role does not need per-Region
statements. What it does need is the OAM pair if you use cross-account
observability:

```hcl
data "aws_iam_policy_document" "workspace" {
  statement {
    sid    = "ReadCloudWatchAnyRegion"
    effect = "Allow"
    actions = [
      "cloudwatch:DescribeAlarmsForMetric",
      "cloudwatch:DescribeAlarmHistory",
      "cloudwatch:DescribeAlarms",
      "cloudwatch:ListMetrics",
      "cloudwatch:GetMetricData",
      "cloudwatch:GetInsightRuleReport",
      "logs:DescribeLogGroups",
      "logs:GetLogGroupFields",
      "logs:StartQuery",
      "logs:StopQuery",
      "logs:GetQueryResults",
      "logs:GetLogEvents",
      "ec2:DescribeTags",
      "ec2:DescribeInstances",
      "ec2:DescribeRegions",
      "tag:GetResources",
    ]
    resources = ["*"]
  }

  statement {
    sid       = "AllowReadingAcrossAccounts"
    effect    = "Allow"
    actions   = ["oam:ListSinks", "oam:ListAttachedLinks"]
    resources = ["*"]
  }

  dynamic "statement" {
    for_each = length(var.amp_workspace_arns) > 0 ? [1] : []
    content {
      sid    = "QueryAMPAnyRegion"
      effect = "Allow"
      actions = [
        "aps:ListWorkspaces",
        "aps:DescribeWorkspace",
        "aps:QueryMetrics",
        "aps:GetLabels",
        "aps:GetSeries",
        "aps:GetMetricMetadata",
      ]
      resources = var.amp_workspace_arns
    }
  }

  dynamic "statement" {
    for_each = length(var.monitored_account_ids) > 0 ? [1] : []
    content {
      sid       = "AssumeQueryRolesInMemberAccounts"
      effect    = "Allow"
      actions   = ["sts:AssumeRole"]
      resources = [for id in var.monitored_account_ids : "arn:aws:iam::${id}:role/${var.name}-grafana-query"]
    }
  }
}
```

### Layer 2 — the token, and the bootstrapping wrinkle

```hcl
resource "aws_grafana_workspace_service_account" "terraform" {
  name         = "terraform"
  grafana_role = "ADMIN"
  workspace_id = aws_grafana_workspace.this.id
}

resource "aws_grafana_workspace_service_account_token" "terraform" {
  name               = "terraform-${formatdate("YYYYMMDD", timestamp())}"
  service_account_id = aws_grafana_workspace_service_account.terraform.service_account_id
  workspace_id       = aws_grafana_workspace.this.id
  seconds_to_live    = 60 * 60 * 24 * 30 # 30 days is the documented maximum

  lifecycle {
    create_before_destroy = true
  }
}
```

Two things to internalise here:

- **`seconds_to_live` maxes out at 30 days** ("You can set the time up to 30
  days in the future"). A long-lived Terraform token is not available. Either
  your CI mints a fresh token per run (best — treat it as ephemeral and never
  store it), or you accept a monthly rotation job. The `timestamp()` in the name
  above forces a new token on every apply, which is the honest version of
  "ephemeral" if your pipeline runs regularly; it also means the resource is
  perpetually diffed, so many teams prefer to mint the token *outside*
  Terraform with `aws grafana create-workspace-service-account-token` in the CI
  step and pass it in as a variable. Pick one and write it down.
- **You cannot update a service account or its token** — the provider docs say
  changing any attribute deletes and recreates. Fine for an ephemeral token,
  surprising if you did not expect it.
- Each service account is billed as a user: `$9`/month for an ADMIN/Editor one.

### Layer 3 — dashboards and alerts as code (grafana provider)

```hcl
provider "grafana" {
  alias = "observability"
  url   = aws_grafana_workspace.this.endpoint
  auth  = aws_grafana_workspace_service_account_token.terraform.key
}
```

> **Provider-configuration caveat.** Configuring a provider from a resource
> attribute means the workspace must exist before this provider can be
> initialised — a classic two-phase apply. In a cookiecutter monorepo the clean
> answer is **two stacks**: `observability-workspace` (aws provider only, owns
> the workspace, outputs the endpoint) and `observability-content` (grafana
> provider only, reads the endpoint from remote state or SSM, owns everything
> inside). This also matches [[provider-aliases-vs-separate-stacks]]: aliases
> for same-provider multi-Region, separate stacks when a provider's own
> configuration depends on another stack's output. Do not try to do both in one
> root module and then wonder why `terraform plan` fails on a clean checkout.

```hcl
# Pin the UID. This is the single most important line in the file: dashboards
# reference data sources by UID, and a UID that differs between the primary and
# standby workspace turns every panel into "Datasource not found".
resource "grafana_data_source" "cloudwatch" {
  provider = grafana.observability
  for_each = toset(var.monitored_regions)

  type = "cloudwatch"
  name = "cloudwatch-${each.key}"
  uid  = "cw-${each.key}" # deterministic, identical in every workspace

  json_data_encoded = jsonencode({
    defaultRegion = each.key
    authType      = "default" # the workspace IAM role
  })
}

resource "grafana_data_source" "amp" {
  provider = grafana.observability
  count    = length(var.amp_workspace_arns) > 0 ? 1 : 0

  type = "prometheus"
  name = "amp-unified"
  uid  = "amp-unified"
  url  = "${var.amp_query_endpoint}api/v1/query"

  json_data_encoded = jsonencode({
    httpMethod    = "POST"
    sigV4Auth     = true
    sigV4AuthType = "ec2_iam_role"
    sigV4Region   = var.observability_region
  })
}

resource "grafana_folder" "helios" {
  provider = grafana.observability
  title    = "helios-${var.env}"
  uid      = "helios-${var.env}"
}

resource "grafana_dashboard" "all" {
  provider = grafana.observability
  for_each = fileset("${path.module}/dashboards", "*.json")

  folder      = grafana_folder.helios.uid
  config_json = templatefile("${path.module}/dashboards/${each.value}", {
    env               = var.env
    monitored_regions = var.monitored_regions
  })
}
```

Alerting as code, so that contact points and rules are reproduced identically
wherever the workspace lives:

```hcl
resource "grafana_contact_point" "oncall" {
  provider = grafana.observability
  name     = "oncall"

  pagerduty {
    integration_key = data.aws_secretsmanager_secret_version.pagerduty.secret_string
    severity        = "critical"
  }

  # Keep Terraform authoritative; block UI edits that would silently drift.
  disable_provenance = false
}

resource "grafana_rule_group" "failover_signals" {
  provider         = grafana.observability
  name             = "failover-signals"
  folder_uid       = grafana_folder.helios.uid
  interval_seconds = 60

  rule {
    name           = "primary-region-metrics-stale"
    for            = "5m"
    condition      = "C"
    no_data_state  = "Alerting"   # a gap IS the signal here — do not use NoData
    exec_err_state = "Alerting"

    data {
      ref_id = "A"
      relative_time_range {
        from = 600
        to   = 0
      }
      datasource_uid = grafana_data_source.amp[0].uid
      model = jsonencode({
        refId = "A"
        expr  = "count(up{region=\"${var.primary_region}\"} == 1)"
      })
    }
    # ... reduce (B) and threshold (C) expressions omitted for brevity
  }
}
```

Note `no_data_state = "Alerting"` on the staleness rule. The default (`NoData`)
produces a distinct state that people route to nowhere. For a cross-Region
liveness check, **no data is the alert**.

### Fitting the monorepo

- One module, `modules/observability-grafana`, instantiated once per pair per
  environment. Its `monitored_regions` variable is where the cookiecutter
  environment config plugs in.
- Dashboards live as `.json` files next to the module and are picked up by
  `fileset()`. Reviewers diff JSON, which is unpleasant but honest; the
  alternative (Jsonnet/Grafonnet) is a bigger commitment than this estate needs
  today.
- **Do not** create a workspace per environment per Region unless dev/stage
  actually need independent dashboards. Each workspace is another `$9` floor and
  another drift surface. One prod workspace plus one non-prod workspace per pair
  is usually right.
- For the CA pair, the module instantiation has to be either omitted or pointed
  at a US Region. Make that explicit in the environment config with a comment,
  not implicit via a `count = 0`.

## Migration path from single-region

Today: presumably one AMG workspace per Region, or none, with dashboards built in
the UI. Target: a Git-driven workspace outside the monitored Regions.

1. **Inventory the existing workspace.** `GET /api/search?type=dash-db` and
   `GET /api/dashboards/uid/<uid>` via a service account token, plus
   `GET /api/datasources`, `GET /api/v1/provisioning/alert-rules`,
   `.../contact-points`, `.../policies`. Commit the raw JSON. This is the backup
   that does not currently exist.
2. **Normalise data source UIDs.** The exported dashboards will reference
   auto-generated UIDs like `PD8C576611E62080A`. Rewrite them to the
   deterministic `cw-<region>` / `amp-unified` scheme *before* you import
   anywhere. This is the step that makes the dashboards portable and it is
   tedious — a `sed`/`jq` pass over the JSON, reviewed once.
3. **Stand up the new workspace in the third Region**, SAML-authenticated,
   empty. Costs `$9`.
4. **Apply the code.** Dashboards, folders, data sources, contact points, rules.
5. **Run both in parallel for a sprint.** Old workspace stays; new workspace is
   the one on the wall. Fix what looks wrong.
6. **Import the old workspace into Terraform** (`aws_grafana_workspace` supports
   import by workspace ID) so nothing is orphaned, then either retire it or
   demote it to "Region-local debugging only".
7. **Turn off UI editing** for the content that matters — `disable_provenance =
   false` on alerting resources makes Terraform authoritative and the UI
   read-only for those objects. Dashboards have no equivalent lock; rely on
   folder permissions and on a CI job that re-applies and reports drift.

**Nothing in this path forces a replacement of a live resource** — you are
creating a new workspace alongside, not mutating the old one. Watch out for:

- `aws_grafana_workspace` attribute changes that force replacement. Changing
  `permission_type` between `SERVICE_MANAGED` and `CUSTOMER_MANAGED`, or moving
  a workspace's `account_access_type`, should be treated as
  potentially-replacing until you have seen the plan. **A replacement deletes
  the workspace and everything in it.** `prevent_destroy = true` is not
  paranoia here.
- `aws_grafana_workspace_service_account` and `..._token`: documented as
  delete-and-recreate on *any* attribute change.
- `grafana_version`: upgrades are applied to a live workspace and are one-way.

## Failover procedure

Under the recommended shape, the observability plane does not fail over. That is
the point. The steps are therefore about *using* it, not repairing it.

**Before the incident (this is the actual work):**

- The workspace URL is a bookmark in the runbook, written out in full, because
  IdP-initiated login does not work.
- Every on-call engineer has logged into it with SAML at least once this quarter.
- A dashboard called something like `helios / failover decision` exists, is
  pinned, and shows only the five signals that drive the decision, for both
  Regions side by side.
- External probes page independently of all of the above.

**During:**

1. Open the failover-decision dashboard. If you cannot, the external probes are
   your instrument and you proceed on those — do not spend RTO budget debugging
   Grafana.
2. Confirm the primary is genuinely impaired and not merely reporting
   impairment (metrics can go stale because the *metrics path* broke). The
   staleness rule tells you which.
3. Make the call. See [[failover-orchestration]] and
   [[split-brain-and-fencing]].
4. **Flip the "active Region" metric** as an explicit step in the runbook, so the
   alert posture follows the traffic. If you forget this step you will spend the
   next hour reading alerts about the wrong Region.
5. Expect a gap in the primary Region's graphs. It is not a dashboard bug.

**If your only workspace was in the dead Region** (i.e. you did not follow this
note): you are creating a workspace under incident pressure. Realistic sequence
— `aws grafana create-workspace` in a live Region, wait for ACTIVE, attach SAML
(needs the IdP metadata URL, which someone has to find), create a service
account and token, run the dashboards-as-code pipeline against the new endpoint.
If dashboards-as-code exists this is maybe 15–40 minutes. If it does not, you
are working from CloudWatch's console for the rest of the incident. This is the
argument for the `$9`.

## Failback

Mostly N/A in the recommended shape — a third-Region workspace never failed over,
so there is nothing to fail back. Under shape (b) or (c) there is real work:

- **Re-point the "active Region" metric** back after failback, or the alert
  posture stays inverted.
- **Remove any silences** created during the incident. Every one. Diff against
  the pre-incident silence list, which you should have captured.
- **Recover the annotations you lost.** If the primary workspace was unreachable
  for six hours, six hours of alert-state history and annotations are missing
  from it. They exist in the standby workspace. There is no merge tool. Accept
  the split history and record in the incident doc which workspace holds which
  window.
- **Re-apply from Git to both workspaces** before declaring done, so that any
  emergency UI edits made during the incident are either promoted to code or
  reverted. Emergency UI edits *will* happen; the failback checklist is where
  they get reconciled.

## Gotchas

1. **AMG is not available in Canada — either Region.** The single biggest
   finding in this note. Plan the CA pair's observability separately.
2. **No backup, no export, no restore.** The AMG API has no workspace-level
   backup or export operation. Your only export is the Grafana HTTP API. If you
   take nothing else from this note, run a nightly job that dumps dashboards and
   alerting config to S3 (see [[aws-s3]]) in a *different* Region.
3. **Deleting a workspace deletes everything in it, immediately.** No recycle
   bin, no 7-day window. `prevent_destroy = true`.
4. **Data source UIDs are generated per workspace.** Two workspaces built by
   clicking will have different UIDs, and dashboards exported from one will show
   "Datasource not found" in the other. Pin UIDs in code. This is the number one
   reason "we have a standby workspace" turns out to be false at 3am.
5. **The Identity Center sign-in pin.** Covered at length above. AMG is "No" for
   deployment in additional Identity Center Regions, so a replicated Identity
   Center does not give the standby workspace a local sign-in path.
6. **No IdP-initiated SAML login.** Blank Relay State; bookmark the workspace URL.
7. **Alert evaluation stops silently** when the workspace's Region is impaired.
   Build a dead-man's switch.
8. **Contact points contain regional ARNs.** An SNS topic in the dead Region is a
   silent shared-fate dependency inside a config field.
9. **OAM has no VPC endpoint.** A VPC-attached workspace needs NAT to use
   cross-account observability.
10. **Cross-account observability is per-Region.** Cross-account and cross-Region
    do not compose via OAM; use assume-role for the both-at-once case.
11. **Service account tokens expire in ≤ 30 days.** No long-lived Terraform
    credential exists. Plan the rotation before the pipeline breaks on a Sunday.
12. **Service accounts and tokens are immutable** — any change is a
    delete-and-recreate.
13. **Every service account and API key is billed as a user.** A `$9`/month
    line item you did not expect, per workspace.
14. **The `$9` floor applies even to a workspace nobody uses** — "Each workspace
    requires a minimum of one Amazon Managed Grafana Editor license in order to
    manage and log into the workspace, even if no users log in."
15. **Grafana version upgrades are one-way** and change alerting behaviour
    (classic → unified). Pin `grafana_version` and upgrade deliberately, in
    non-prod first, and expect the Terraform Grafana provider's supported
    resource set to move with it.
16. **Remote-write gaps look like silence, not failure.** `absent()` /
    staleness rules are mandatory, and `no_data_state` must be `Alerting` for
    liveness rules.
17. **Cross-Region remote-write is inter-Region egress.** At high cardinality it
    is a real cost line.
18. **An idle standby generates false alerts, and the silence you add will still
    be there when you promote it.** Use role-labelled rules, not silences.
19. **`SERVICE_MANAGED` permissions with `ORGANIZATION` access deploys IAM via
    CloudFormation StackSets**, outside your Terraform. In a mature Terraform
    estate that is an unwelcome second source of truth — prefer
    `CUSTOMER_MANAGED` and own the roles yourself, which is what the module above
    does.
20. **Annotations and alert history are unrecoverable.** They are the one part of
    workspace state that is not expressible as code. Plan to lose them.

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| Where the workspace lives | In the pair (primary and/or standby) | A third Region | **B.** Breaks shared fate with both halves of the pair; costs one extra Region in the Terraform estate. Fall back to A-standby-only if a third Region is unacceptable. |
| Authentication | IAM Identity Center | SAML direct to the corporate IdP | **B, without qualification**, for any workspace you might need during a failover. IdC pins sign-in to its primary Region and AMG cannot use a replica. |
| Dashboards | Managed in the UI | Dashboards-as-code in Git | **B.** This is the whole mitigation. Without it there is no standby worth having. |
| Alert evaluation for failover signals | Grafana-managed rules | CloudWatch alarms (+ external probes) | **B for the decision signals**, A for everything else. CloudWatch alarms can live in a Region you choose and do not depend on the workspace being up. |
| Metrics backend | AMP workspace per Region | One AMP workspace fed by remote-write from both | **B.** Real cross-Region PromQL in one expression; accept the egress cost and build staleness alerts. |
| Canada | AMG workspace in a US Region | No AMG for Canada; CloudWatch + external probes | **Escalate, do not decide in engineering.** It is a [[data-residency]] question. Default to B until legal says otherwise. |
| Standby alert noise | Silence the standby | Role-labelled rules driven by an "active Region" metric | **B.** Silences outlive their reason and survive into the promotion. |
| Terraform layering | One root module using both providers | Two stacks: workspace, then content | **B.** The grafana provider's configuration depends on an aws resource attribute; two stacks avoids the chicken-and-egg on a clean checkout. |

## Cost

All figures verified against live AWS sources on 2026-09-21: the Amazon Managed
Grafana pricing page, and the AWS Price List API (offer codes `AmazonGrafana` and
`AmazonPrometheus`).

### Amazon Managed Grafana

Per the pricing page: "You pay only for what you use, based on an active user
licence per workspace... An 'Active user' is any user that has logged in to an
Amazon Managed Grafana workspace or made an API request at least once during a
monthly billing cycle."

| Item | Price | Notes |
|---|---|---|
| Editor / administrator licence | **$9 per active editor or administrator user per workspace** per month | Confirmed identical in `eu-west-1`, `eu-west-2`, `us-east-1`, `us-west-2` via the Price List API. |
| Viewer licence | **$5 per active user per workspace** per month | Same across those Regions. |
| Enterprise plugins | **$45 per active user per workspace** per month, additional | Only if you need third-party enterprise data sources. |
| API key / service account | Billed as a user: **$9** (admin/editor) or **$5** (viewer) per active one | "Each Service account is billed as an Amazon Manager Grafana user" [sic, AWS's typo]. |
| Free trial | 90-day free trial, up to five free users per account | One-off. |
| Minimum | **One editor licence per workspace, "even if no users log in"** | AWS's own Example 3: a workspace with zero logins in January bills `1 × $9.00 = $9.00`. |

**What a duplicated standby workspace actually adds:**

| Scenario | Monthly |
|---|---|
| Standby workspace, nobody logs in, no automation touches it | **$9** |
| Standby workspace kept in sync by a Terraform ADMIN service account | **$18** ($9 floor + $9 service account) |
| Standby workspace with 3 engineers logging in during a quarterly drill (that month only) | $18 + 3 × $9 = **$45** in drill months |
| Third-Region workspace replacing both (recommended shape) | $9 floor + $9 service account + actual users — i.e. *less* than running two |

This is the cheapest insurance in the entire vault. Compare with an RDS standby
or a warm EKS node group in [[cost-model]]. Do not let a `$9`–`$18`/month line
item be the reason the failover decision is made blind.

### Amazon Managed Service for Prometheus

Live per-Region figures from the Price List API (USD):

| Item | `eu-west-1` / `eu-west-2` / `us-east-1` / `us-west-2` | `ca-central-1` | `ca-west-1` |
|---|---|---|---|
| Ingestion, first 2B samples/mo | $0.90 per 10M samples | $0.981 | $0.924 |
| Ingestion, next 250B | $0.35 per 10M | $0.381 | $0.359 |
| Ingestion, over 252B | $0.16 per 10M | $0.174 | $0.164 |
| Storage above 10 GB | $0.03 per GB-month | $0.0327 | $0.030801 |
| Query samples processed | $0.10 per billion | $0.10 | $0.10 |
| Managed collector | $0.04 per collector-hour | $0.0436 | $0.041068 |
| Samples collected (managed collector) | $0.03 per 10M | $0.032 | $0.031 |

Free tier: 40M samples ingested, 200B query samples processed, 10 GB stored.

**Cost of the "remote-write both Regions into one workspace" pattern:** you pay
ingestion once per sample, so sending `eu-west-2`'s samples to a `eu-central-1`
workspace costs the same in AMP terms as sending them to a `eu-west-2` workspace.
The delta is **inter-Region data transfer out** of the monitored Region, which is
charged under EC2 data transfer rather than by AMP — not verified here, and it
scales with cardinality, so measure it on a sample of the real metric set before
committing. The standby Region's idle metric volume is low by construction
(fewer pods, no traffic), so the incremental ingestion from adding the standby to
the same workspace is small.

### What is free

Workspaces themselves, dashboards, folders, alert rules, contact points, data
source definitions, and the AMG control plane. There is no hourly charge. The
entire cost model is "who logged in".

## AMG versus the alternatives

Kept deliberately short; the detail belongs in
[[observability-vendors-multi-region]].

| Option | Shared fate with the monitored Region? | Verdict |
|---|---|---|
| **AMG in a third Region** | No | The recommendation. Cheap, managed, native IAM integration, one pane of glass across Regions. Weaknesses: no Canada, Identity Center sign-in trap, no backup API. |
| **Self-hosted Grafana on EKS** | **Yes, totally** — it dies with the cluster it runs on, and a cluster in the primary Region dies with the primary Region | Only defensible if it runs in a *different* cluster in a *different* Region, at which point you have rebuilt AMG by hand with worse auth and an upgrade treadmill. See [[eks-workload-delivery]]. The one advantage: it is the only option that can run in `ca-central-1` under your own control. |
| **Grafana Cloud** | No — it is outside your AWS account entirely | The strongest answer to the shared-fate question, and the same Grafana UI/dashboards-as-code you already wrote (the Terraform provider targets both). Costs move to a vendor bill and data leaves your account, which is a [[data-residency]] conversation. |
| **Datadog** | No | Same shared-fate advantage; a much larger commercial and migration commitment; not a like-for-like swap for dashboards-as-code you have already built against Grafana. |

### Is observability the thing to take off AWS deliberately?

**Yes, at least in part — and this is the one place in this vault where "use a
non-AWS service" is the straightforwardly correct engineering answer.**

The argument is not about features. It is that the instrument you use to decide
whether AWS is broken should not be built on AWS. Every AWS-hosted option in the
table above shares *some* fate with AWS: the Region, IAM, Identity Center, the
console, the status page. During a broad regional event those correlate, and
they correlate exactly when you need them.

The pragmatic split:

- **Keep AMG** for the 95% of observability that is everyday work: deep
  dashboards, debugging, capacity, cost. It is cheap, it is native, and the
  dashboards-as-code you write for it are portable.
- **Put the failover-decision signals outside AWS.** A handful of black-box
  probes from a third-party prober, paging directly. That is a small, cheap,
  boring dependency and it is the only thing in the whole architecture that is
  guaranteed to be working when everything else is not.

Doing *only* the second is not enough (you cannot debug from five signals).
Doing *only* the first leaves you with a blind spot you cannot see precisely
when it matters. Do both; the second costs very little.

## Open questions

Things that need an answer from inside the company before this becomes a plan:

1. **Where is the IAM Identity Center primary Region today?** This determines
   whether the current AMG setup already has the sign-in trap. Nobody outside
   the org can answer it and it changes the recommendation's urgency.
2. **Is Identity Center replicated to any additional Regions?** And does the
   corporate IdP support multiple ACS URLs (Okta/Entra/Ping/JumpCloud yes,
   Google Workspace no)? Affects the wider console-access story in
   [[failover-orchestration]].
3. **Is the identity source AWS Managed Microsoft AD?** If so, Identity Center
   multi-Region replication is unavailable entirely and the SAML-direct
   recommendation becomes the only option.
4. **Canada: what may leave the country?** Metric names and dimension values,
   log samples, trace attributes — each is a different answer. Needed before any
   US-hosted observability for `ca-central-1`.
5. **Does a third Region (`eu-central-1` / `us-east-2`) create a compliance or
   contractual problem?** GDPR for the EU pair is probably fine intra-EU;
   confirm rather than assume. See [[data-residency]].
6. **Does an AMG workspace already exist, and is anything in it hand-built?**
   The migration path's step 1 is an inventory, and the answer determines whether
   this is a fortnight or a quarter.
7. **Who owns the on-call paging path today** — is PagerDuty/Opsgenie already
   independent of AWS, or is it fed through an SNS topic in the primary Region?
   If the latter, the paging path has the same shared-fate bug as the dashboards
   and it is more urgent than any of this.
8. **Which RTO definition applies** — 15 minutes from incident start, or from
   decision? If from incident start, the observability plane's detection latency
   eats the budget and the external-probe recommendation moves from "nice" to
   "mandatory". Flagged in `CLAUDE.md` as an open thread for the whole vault.

## Still to research

- AMG's published SLA, if any, and its design-availability goal.
- Whether AWS Backup lists Amazon Managed Grafana as a supported resource
  (expected: no, but unverified).
- The exact set of Grafana versions AMG currently supports and the upgrade
  semantics (`grafana_version` values, one-way or not).
- AMG network access control (`network_access_control`) and VPC data source
  reachability specifics.
- Whether AMP offers any cross-Region query or federation capability beyond
  remote-write and mixed-datasource dashboards.
- Whether any AWS blog or re:Invent talk documents a real multi-region AMG
  topology (no public example found yet — to be confirmed or written up as a
  negative finding).
- Inter-Region data transfer pricing applicable to remote-write, quantified.

## Sources

Real URLs only. Each line says what the source contributes.

- <https://docs.aws.amazon.com/grafana/latest/userguide/disaster-recovery-resiliency.html>
  — AMG's resilience page. Contributes mainly by omission: it discusses AZs only
  and offers no cross-Region or backup capability.
- <https://docs.aws.amazon.com/grafana/latest/userguide/authentication-in-AMG.html>
  — the two authentication options; "Each workspace can use one or both".
- <https://docs.aws.amazon.com/grafana/latest/userguide/authentication-in-AMG-SAML.html>
  — SAML support, tested IdPs, SP-initiated only, blank Relay State, assertion
  mapping.
- <https://docs.aws.amazon.com/grafana/latest/userguide/authentication-in-AMG-SSO.html>
  — Identity Center integration; "Amazon Managed Grafana does not support the use
  of IAM users and roles to assign permissions within an Amazon Managed Grafana
  workspace"; the required admin policies.
- <https://docs.aws.amazon.com/grafana/latest/userguide/cloudwatch-cross-account.html>
  — cross-account observability in AMG: required `oam:` actions, Grafana 9+,
  no VPC endpoint for OAM.
- <https://docs.aws.amazon.com/singlesignon/latest/userguide/resiliency-regional-behavior.html>
  — "the primary Region cannot be changed after IAM Identity Center is enabled";
  replication to additional Regions; design goals 99.95% data plane / 99.90%
  control plane; break-glass advice.
- <https://docs.aws.amazon.com/singlesignon/latest/userguide/multi-region-iam-identity-center.html>
  — what replicates, and the prerequisites: organization instance, external IdP
  or Identity Center directory (not AD), **opt-in Regions not supported**,
  multi-Region CMK required, multiple ACS URLs.
- <https://docs.aws.amazon.com/singlesignon/latest/userguide/multi-region-application-use.html>
  — the load-bearing quote: "Each AWS managed application connects to a specific
  IAM Identity Center Region during deployment. The application then depends on
  that Region for user sign-in..."
- <https://docs.aws.amazon.com/singlesignon/latest/userguide/awsapps-that-work-with-identity-center.html>
  — the applications table. Amazon Managed Grafana: "Supports deployment in
  additional Regions of IAM Identity Center" = **No**.
- <https://aws.amazon.com/grafana/pricing/> — $9 editor / $5 viewer / $45
  enterprise plugins per active user per workspace per month; the "Active user"
  definition; the one-editor minimum and worked Example 3.
- <https://aws.amazon.com/prometheus/pricing/> — AMP ingestion/storage/query/
  collector model and free tier.
- <https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonGrafana/current/index.json>
  — AWS Price List API; confirms the $9/$5/$45 rates per Region and, by the set
  of `location` values present, confirms AMG's 12-Region footprint with no
  Canadian Region.
- <https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonPrometheus/current/index.json>
  — AWS Price List API; per-Region AMP rates including `ca-central-1` and
  `ca-west-1`.
- <https://api.regional-table.region-services.aws.a2z.com/index.json> — the JSON
  behind the AWS regional services table; used to verify AMG and AMP Region
  availability (`source:version 20251113063700`).
- <https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/grafana_workspace>
  — `aws_grafana_workspace` arguments: `account_access_type`,
  `authentication_providers` (`AWS_SSO`/`SAML`), `permission_type`,
  `configuration` JSON including `unifiedAlerting`.
- <https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/grafana_workspace_service_account_token>
  — `seconds_to_live` "up to 30 days in the future"; tokens and service accounts
  are delete-and-recreate on change; `key` attribute for the Grafana HTTP API.
- <https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/grafana_workspace_saml_configuration>
  — SAML configuration surface: `idp_metadata_url`/`idp_metadata_xml`, role and
  assertion mappings.
- <https://registry.terraform.io/providers/grafana/grafana/latest/docs> — the
  Grafana provider: `url` + `auth`, and the `grafana_dashboard`, `grafana_folder`,
  `grafana_data_source`, `grafana_rule_group`, `grafana_contact_point` resources
  used for dashboards-as-code.
- <https://aws.amazon.com/blogs/opensource/set-up-cross-region-metrics-collection-for-amazon-managed-service-for-prometheus-workspaces/>
  — AWS Open Source blog on cross-Region metrics collection into AMP; the
  remote-write-across-Regions pattern.
- <https://aws.amazon.com/blogs/opensource/setting-up-amazon-managed-grafana-cross-account-data-source-using-customer-managed-iam-roles/>
  — AWS Open Source blog on AMG cross-account data sources with customer-managed
  IAM roles.
- <https://grafana.com/docs/grafana/latest/datasources/aws-cloudwatch/aws-authentication/>
  — Grafana's own CloudWatch auth docs: assume-role behaviour, and that the
  primary credentials need only `sts:AssumeRole`.
