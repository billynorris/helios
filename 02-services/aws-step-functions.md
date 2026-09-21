---
title: AWS Step Functions — Multi-Region
service: step-functions
tags: [service, multi-region, step-functions, sfn, orchestration, failover, idempotency]
status: partial
replication: none — state machine *definitions* are Terraform config and mirror trivially; *executions* do not replicate and cannot be migrated
rpo_achievable: "N/A for config. For in-flight executions the RPO is effectively *the age of the oldest open execution* — up to 1 year for Standard. This is an application-design problem, not an infrastructure one."
rto_achievable: "< 1 min — a mirrored state machine is ready the moment its ARNs resolve. The risk is not time, it is correctness."
meets_targets: conditional — yes for the control plane, no for in-flight work unless every task is idempotent
updated: 2026-09-21
---

# AWS Step Functions — Multi-Region

> **This note has two halves and they are about different problems.**
>
> **Part 1** treats Step Functions as *a workload to be failed over*. The
> definition mirrors in one Terraform module call. The running executions do
> not mirror at all, and that is where the whole note lives.
>
> **Part 2** treats Step Functions as *the tool that implements the failover
> runbook* — the state machine that promotes the database, scales the standby
> and flips DNS. [[failover-orchestration]] lists it as Option 3 and moves on;
> this note interrogates it properly and reaches a recommendation that agrees
> with [[route53-application-recovery-controller]] for a reason that note
> does not give.
>
> Read [[failover-orchestration]] first for the 15-minute budget decomposition.
> Nothing here re-derives it.

## TL;DR

- **The definition is config; the executions are the problem.** `aws_sfn_state_machine` in a module called twice with two provider aliases gives you an identical state machine in the standby in about ten lines. That part is genuinely trivial and this note spends one section on it. **What does not cross the region boundary is a running execution.** An in-flight Standard workflow in the failed region is not paused, not queued, not replicated — it is simply somewhere else, holding the middle of a business transaction.
- **A half-completed workflow is strictly worse than a lost message.** [[messaging-in-flight-data-loss]] is about work that had not started yet. A Step Functions execution that died between `ChargeCard` and `RecordOrder` is work that *has already had side effects in the outside world* and has no local record of them. The charge is durable at the payment processor; the order row was never written; and the only evidence that the charge happened is a `GetExecutionHistory` call against a regional API in a region that is down. **This is an application-design problem. No amount of Terraform fixes it.**
- **`RedriveExecution` is a failback tool, not a failover tool.** It is Standard-only, same-region, same-execution-ARN, same-definition, and bounded by a 14-day redrivable window whose clock runs *during* your outage. You cannot redrive an `eu-west-1` execution from `eu-west-2`. "Re-driving" in the standby is not redrive at all — it is a fresh `StartExecution` from reconstructed input, and whether that is safe is entirely down to whether your tasks are idempotent. Step Functions' own `StartExecution` name-based idempotency is **per account, per Region, per state machine**, so it gives you exactly nothing across a failover. See [[#Idempotency, redrive and the re-run decision]].
- **Mirroring a state machine verbatim into the standby produces a state machine that still points at the primary region.** `terraform plan` is clean, the definition validates, the console renders the graph — and at execution time it invokes the *primary's* Lambda and writes to the *primary's* table. If the primary is fully down you get a loud failure; if it is only degraded you get a **silent no-op failover**, with traffic in the standby writing into the region you just evacuated. This is the single most likely real-world bug in this note and it gets its own section with the Terraform to prevent it: [[#Service-integration ARNs are region-specific].
- **Verified parity and quota findings.** Step Functions is present in all six of this estate's regions, **including `ca-west-1`, with FIPS endpoints** — Calgary *passes* this check, unlike Cognito, OpenSearch CCR and Managed Grafana. But the `StateTransition` quota is **5,000/s in `us-east-1`, `us-west-2` and `eu-west-1` and 800/s in every other Region**, so **the EU pair falls off a 6.25× cliff at failover** into London, while the US pair is symmetric and the CA pair is symmetrically low. Quota increases are per-account-per-Region and do not replicate. See [[#Quotas — the EU pair's 6.25× cliff]].
- **Recommendation for Part 2: do not make Step Functions the top-level failover orchestrator.** Its one decisive advantage over SSM Automation — native `Parallel` — is also native in ARC Region switch, which additionally has a data plane in *every* Region so you never have to solve the "where does the orchestrator live" problem yourself. Step Functions' place is one level down: as a **worker invoked by a Region switch or SSM step** where real fan-out is needed. Full argument and the honest counter-case in [[#Part 2 — Step Functions as the failover orchestrator]].

---

# Part 1 — Step Functions as a workload to be failed over

## Does this service cross regions at all?

No. Step Functions is a wholly regional service and nothing about it is global.

| Thing | Scope | Crosses regions? |
|---|---|---|
| State machine (`arn:aws:states:<region>:<acct>:stateMachine:<name>`) | Regional | No — you create one per Region |
| Execution (`...:execution:<sm>:<name>`) | Regional | **No. Not replicated, not migratable, not visible from elsewhere.** |
| Execution history | Regional, retained 90 days | No — `GetExecutionHistory` is a regional API call |
| Activity (`...:activity:<name>`) | Regional | No — workers poll a regional ARN |
| Task token | Bound to an execution, therefore regional, and same-account-only | **No** |
| State machine name uniqueness | Per account **per Region** | No — the same name is free in the standby |
| Execution name uniqueness (Standard) | Per account, per Region, per state machine, 90 days | **No — and this is why its built-in idempotency is useless to you** |
| Quotas | Per account **per Region**, and they differ by Region | No — increases must be requested per Region |

There is no cross-Region replication feature, no global state machine, no "execution follows the data". Contrast with [[aws-dynamodb]] Global Tables or [[aws-secrets-manager]] replicas, where AWS gives you a native mirroring primitive. Here the mirroring primitive is **your Terraform module**, and the thing that cannot be mirrored — execution state — has no primitive at all.

### `ca-west-1` parity check — **PASS**

Calgary has failed the parity check in this vault three times already (Cognito multi-region replication, OpenSearch cross-cluster replication, Amazon Managed Grafana — see [[region-pair-selection]]). **Step Functions is not a fourth failure.**

The [AWS General Reference endpoints table](https://docs.aws.amazon.com/general/latest/gr/step-functions.html) lists, for `ca-west-1` (Canada West, Calgary), verbatim:

> `states.ca-west-1.amazonaws.com`, `sync-states.ca-west-1.api.aws`, `sync-states-fips.ca-west-1.api.aws`, `sync-states-fips.ca-west-1.amazonaws.com`, `states-fips.ca-west-1.api.aws`, `sync-states.ca-west-1.amazonaws.com`, `states-fips.ca-west-1.amazonaws.com`, `states.ca-west-1.api.aws`

That is the **full** endpoint set — standard, dual-stack (`.api.aws`), sync-express (`sync-states.*`) and **FIPS**, in both standard and dual-stack and sync flavours. `ca-west-1` has a *richer* published endpoint set than `eu-west-1` or `eu-west-2`, which have no FIPS endpoints at all. Both Canadian regions and both European regions and both US regions appear.

All six regions in scope:

| Region | Role | Step Functions endpoint | FIPS | Sync-Express |
|---|---|---|---|---|
| `eu-west-1` | EU primary | ✅ | ❌ | ✅ |
| `eu-west-2` | EU standby | ✅ | ❌ | ✅ |
| `us-east-1` | US primary | ✅ | ✅ | ✅ |
| `us-west-2` | US standby | ✅ | ✅ | ✅ |
| `ca-central-1` | CA primary | ✅ | ✅ | ✅ |
| `ca-west-1` | CA standby | ✅ | ✅ | ✅ |

> [!note] The parity that matters is not presence, it is quota
> Calgary *has* Step Functions. What Calgary does not have is `us-east-1`-class
> throughput defaults, and neither does `ca-central-1` — so for the CA pair the
> gap is symmetric and therefore invisible in a parity check. The EU pair's gap
> is **asymmetric** and that is the one that bites.
> See [[#Quotas — the EU pair's 6.25× cliff]].

## Mirroring the definition — the easy half, stated once

A state machine is a JSON document (ASL), an IAM role ARN, a workflow type, a logging config and some tags. All of it is Terraform. Instantiate the module twice with two provider aliases and the standby has a byte-identical state machine. [[provider-aliases-vs-separate-stacks]] covers the aliasing pattern; [[module-patterns]] covers the module shape.

Two mechanical facts you need before you write that module:

1. **`name` and `type` are `ForceNew` on `aws_sfn_state_machine`.** Changing either destroys and recreates the state machine. `definition`, `role_arn`, `logging_configuration`, `tracing_configuration`, `publish` and `encryption_configuration` all update in place. See [[#Migration path from single-region]] — the `ForceNew` on `name` is the one that will bite an estate with per-region resource naming, exactly as it did for [[dynamodb-table-naming-migration]].
2. **Workflow type is immutable in the service, not just in Terraform.** AWS, verbatim: *"The workflow type can **not** be updated after you create a state machine."* So a Standard→Express decision made three years ago is a rebuild, not a toggle.

That is the whole of the easy half. **Everything else in Part 1 is about the fact that a perfect copy of the definition gives you none of the state.**

## Running executions do not migrate

This is the RPO question for Step Functions and it deserves more space than everything else combined.

### What "does not migrate" actually means

There is no ambiguity and no partial credit. When `eu-west-1` becomes unavailable:

- Open executions in `eu-west-1` are **not** copied to `eu-west-2`.
- They are **not** queued for later delivery.
- They are **not** visible from `eu-west-2` by any API.
- `RedriveExecution` cannot be called cross-Region.
- The standby state machine starts its life having never run anything.

And critically — they are **not destroyed either**. AWS's own description of Standard workflows, verbatim from [Choosing workflow type](https://docs.aws.amazon.com/step-functions/latest/dg/choosing-workflow-type.html):

> Execution state internally persists between state transitions.

So a Standard execution that was mid-flight when the Region went dark is *durable state sitting in a Region you have evacuated*. It has not failed. It has not succeeded. It is open, and it can remain open for **up to one year** (the hard maximum execution time). When the Region recovers, it is still there, and its tasks will resume, retry, or time out according to the retriers and timeouts you wrote — **against a database you have since demoted, and a standby that has since done the same work.**

> [!danger] Open executions in the recovered Region are a split-brain generator
> This is the point that is almost never written down. [[split-brain-and-fencing]]
> is about two writable databases. **Step Functions gives you a second vector:
> tens of thousands of durable, resumable, side-effect-producing workflows that
> wake up in the old primary when it comes back.**
>
> Concretely: an execution sitting on `"Type": "Wait", "Seconds": 3600` in
> `eu-west-1` at the moment of failover will, when the Region returns,
> transition to its next state and call `ChargeCard` — an hour after you failed
> over, for an order the standby has already fulfilled.
>
> **Therefore: enumerating and stopping open executions in the failed Region is
> a required step in the fencing and failback procedure**, not an optional
> tidy-up. It is missing from [[failover-orchestration]]'s six-step sequence and
> from [[failover-runbook-template]]. See [[#Fencing Step Functions]] for why
> you probably cannot do it at the moment you most want to.

### What a half-completed workflow leaves behind

Force yourself through a concrete one. A four-state Standard workflow, a real shape:

```
ValidateOrder  →  ChargeCard  →  RecordOrder  →  NotifyCustomer
  (Lambda)        (Lambda →      (DynamoDB      (SNS publish)
                   Stripe)        PutItem)
```

The Region fails between `ChargeCard` returning and `RecordOrder` being scheduled. What exists in the world:

| Artefact | Where it is | Survives the failover? |
|---|---|---|
| The Stripe charge | Stripe's systems | **Yes — durable, and irreversible without a refund** |
| The `ChargeCard` result (the charge ID) | `eu-west-1` execution history | **No — unreachable while the Region is down** |
| The order row | Nowhere. Never written. | — |
| The customer notification | Never sent | — |
| Any record that the charge happened | **Only the Step Functions execution history, in the dead Region** | **No** |

**You have taken a customer's money and you have no idea that you did.** The reconciliation is not "replay the failed executions" — you cannot see the failed executions. It is "pull a settlement report from Stripe, diff it against the orders table in the promoted standby, and find the charges with no order." That is a *finance* process invented under pressure at 4am, and it only works because Stripe happens to be an external system of record that survived the Region.

Now vary it. Replace `ChargeCard` with something whose system of record was *also* in the dead Region:

| First task did… | What you are left with |
|---|---|
| `ecs:runTask.sync` provisioning a tenant's resources | A provisioned resource with no tracking row. Nobody will ever bill for it, and nobody will ever delete it. |
| `sqs:sendMessage` to a downstream service | A message that the downstream service consumed and acted on, for a workflow that no longer exists. [[aws-sqs]], [[messaging-in-flight-data-loss]]. |
| `dynamodb:updateItem` decrementing inventory | Stock reserved for an order that was never placed. Replicated to the standby by Global Tables, so the reservation survives and the order does not. [[aws-dynamodb]]. |
| `events:putEvents` fanning out an `OrderPlaced` event | Six downstream consumers acted on an order the system has no record of. [[aws-eventbridge]]. |
| `lambda:invoke` sending an email | Customer told their thing happened. It did not. |

Note the pattern in the last column. **In every case the damage is the side effect that escaped the Region while the coordinating state stayed inside it.** That asymmetry is the whole failure mode, and it exists because Step Functions' durability guarantee is scoped to a Region while your side effects are scoped to the internet.

### The honest reframe

> [!important] This is an application-design problem wearing infrastructure clothes
> There is no Step Functions feature, no Terraform module, no replication
> setting and no AWS service that makes an in-flight execution survive a
> Regional failure. AWS does not offer one and is unlikely to, because the
> semantics are undefinable: a task that was half-done in one Region cannot be
> meaningfully resumed in another without knowing whether its side effect
> landed — and only your application knows that.
>
> The only thing that makes a lost execution recoverable is that **every task
> is idempotent and the workflow can be re-run from the beginning with the same
> business key and produce the same end state.** That is a property of your
> Lambda code, your DynamoDB conditional writes and your payment-processor
> idempotency keys. It is not a property of your infrastructure.
>
> **The correct output of this note for the application teams is a single
> question per state machine: "if we re-ran this execution from the top with
> the same input, what breaks?"** If the answer is "nothing", the state machine
> is DR-ready. If the answer is "we'd charge them twice", it is not, and no
> amount of standby provisioning will change that.

### Sizing the exposure

The exposure is not "2 hours" like the rest of this estate's RPO. It is **the age of the oldest open execution**, which for Standard workflows can be a year.

Get the number before you design anything:

```bash
# Per state machine, per Region. Run against the PRIMARY today.
aws stepfunctions list-executions \
  --state-machine-arn arn:aws:states:eu-west-1:111122223333:stateMachine:order-fulfilment \
  --status-filter RUNNING \
  --region eu-west-1 \
  --query 'length(executions)'

# And the oldest one — this is your true Step Functions RPO.
aws stepfunctions list-executions \
  --state-machine-arn arn:aws:states:eu-west-1:111122223333:stateMachine:order-fulfilment \
  --status-filter RUNNING --region eu-west-1 \
  --query 'sort_by(executions,&startDate)[0].startDate'
```

Also publish it continuously. The `ExecutionsStarted` minus `ExecutionsSucceeded`/`ExecutionsFailed`/`ExecutionsAborted`/`ExecutionsTimedOut` CloudWatch metrics give you open-execution count per state machine; put it on the same dashboard as replica lag, because **it is the same kind of number** — it is how much in-flight work you lose if you fail over right now. [[observability-multi-region]].

> [!tip] A design rule that costs nothing and shrinks the exposure by orders of magnitude
> **Long `Wait` states are RPO.** A workflow that does its work in 4 seconds and
> then sits on `"Wait": 7 days` before sending a follow-up email has a 7-day
> window in which a Regional failure loses it. Move the wait out of the state
> machine: finish the execution, and schedule the follow-up with
> [[aws-eventbridge]] Scheduler or a DynamoDB TTL, both of which have real
> cross-Region stories. **Executions should be short not because Step Functions
> is expensive but because open executions are unreplicated state.**

## Service-integration ARNs are region-specific

**This is the bug you will actually ship.** Everything above is about data you knew you might lose. This is about a deployment that looks completely correct and is completely wrong.

### The mechanic

A `Task` state has two places a Region can hide, and only one of them is obvious.

**Place 1 — the `Resource` field.** For optimized and SDK integrations this is a *pseudo-ARN* with empty region and account fields:

```
arn:aws:states:::lambda:invoke
arn:aws:states:::aws-sdk:dynamodb:putItem
arn:aws:states:::ecs:runTask.sync
```

AWS explains the `:::` deliberately, verbatim from [Discover service integration patterns](https://docs.aws.amazon.com/step-functions/latest/dg/connect-to-resource.html):

> The `:::` portion of the value denotes empty `region` and `account-id` fields which are unnecessary because both are inferred from the region and account in which the workflow runs.

**So the `Resource` field is region-portable by construction.** Copy it to the standby and it will call the standby's service. This is the half people look at, and it is fine.

**Place 2 — the `Parameters` / `Arguments` block.** This is where the Region actually lives, and it is *invisible in the graph view*. AWS's own canonical Lambda example, verbatim from [Task workflow state](https://docs.aws.amazon.com/step-functions/latest/dg/state-task.html):

```json
"Lambda Invoke": {
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "Parameters": {
    "Payload.$": "$",
    "FunctionName": "arn:aws:lambda:{{region}}:{{account-id}}:function:{{HelloFunction}}:$LATEST"
  },
  "End": true
}
```

The `Resource` is region-less. The `FunctionName` is a **fully-qualified, region-bearing ARN**. Copy that document into `eu-west-2` unchanged and every execution in London invokes a Lambda in Ireland.

### The taxonomy you actually need

Not every parameter carries a Region. Sort your integrations into three buckets — **this table is the review checklist**:

| Integration | Region-bearing parameter | Behaviour if copied verbatim | Risk |
|---|---|---|---|
| `lambda:invoke` | `FunctionName` (full ARN, as AWS's own example) | Invokes the **primary's** function | 🔴 **High** |
| `lambda:invoke` | `FunctionName` (bare name, e.g. `order-processor`) | Resolves in the **local** Region | 🟢 Safe |
| `sns:publish` | `TopicArn` — ARN is the only accepted form | Publishes to the **primary's** topic | 🔴 **High** — [[aws-sns]] |
| `sqs:sendMessage` | `QueueUrl` — `https://sqs.<region>.amazonaws.com/...` | Sends to the **primary's** queue | 🔴 **High** — [[aws-sqs]] |
| `states:startExecution[.sync]` | `StateMachineArn` | Starts a child in the **primary** | 🔴 **High** |
| `ecs:runTask[.sync]` | `Cluster`, `TaskDefinition` (both ARNs) | Runs the task in the **primary's** cluster | 🔴 **High** |
| `events:putEvents` | `EventBusName` — accepts a name **or** an ARN | Depends which you wrote | 🟠 **Audit it** — [[aws-eventbridge]] |
| `aws-sdk:dynamodb:putItem` | `TableName` — a bare name | Resolves in the **local** Region | 🟢 Safe *(and the reason [[dynamodb-table-naming-migration]]'s same-name-everywhere requirement helps here)* |
| `aws-sdk:s3:getObject` | `Bucket` — a global-ish name | Resolves to the **actual bucket**, wherever it is | 🟠 Bucket names are global; a primary-region bucket name still points at the primary. [[aws-s3]] |
| `aws-sdk:secretsmanager:getSecretValue` | `SecretId` — name **or** ARN | Name → local replica ✅. ARN → primary ❌ | 🟠 **Audit it** — [[aws-secrets-manager]] |
| `aws-sdk:ssm:getParameter` | `Name` — a bare name | Local — but [[aws-ssm-parameter-store]] has no native replication, so the parameter may not *exist* | 🟠 Different failure |
| Activity task | `Resource` is a **real, region-bearing ARN** (`arn:aws:states:<region>:<acct>:activity:<name>`) | Polls the **primary's** activity | 🔴 **High** — and it is in the `Resource` field, so it breaks the "`Resource` is safe" heuristic |
| HTTP Task | An EventBridge Connection ARN + a URL | Connection ARN is regional | 🟠 — and the Connection must exist in the standby |

> [!warning] Two exceptions to "the `Resource` field is region-less"
> 1. **Activities.** `"Resource": "arn:aws:states:eu-west-1:111122223333:activity:CreditCheck"` is a genuine regional ARN in the `Resource` field. It is the one task type where the obvious place *is* the dangerous place.
> 2. **Legacy Lambda integrations.** AWS, verbatim: *"Legacy integrations to AWS Lambda are the one exception where the Resource value specifies an actual Lambda function resource."* An old state machine with `"Resource": "arn:aws:lambda:eu-west-1:...:function:foo"` is region-pinned in the `Resource` field and the modern console will not even let you edit it graphically. **Grep for this first; it is exactly the shape an estate that started with Step Functions in 2019 will have.**

### Why you cannot just point the standby at the primary and forget it

Because **Step Functions will not let you, and you would not want it to.** Verbatim from [Task workflow state](https://docs.aws.amazon.com/step-functions/latest/dg/state-task.html):

> Step Functions doesn't support referencing ARNs across partitions or regions. For example, `aws-cn` can't invoke tasks in the `aws` partition, and the other way around.

And the AWS CDK ships a construct that exists solely to work around the gap — [`CallAwsServiceCrossRegion`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_stepfunctions_tasks.CallAwsServiceCrossRegion.html), described verbatim as:

> A Step Functions task to call an AWS service API across regions.

> This task creates a Lambda function to call cross-region AWS API and invokes it.

**A whole Lambda, per cross-Region call, because the service integration cannot do it.** Note what that implies for Part 2: a failover orchestrator in a *third* Region cannot use SDK integrations to act on the standby at all. It must proxy every call through a Lambda with an explicit region override. That is a hard architectural constraint on third-region placement and it is dealt with in [[#Where does the orchestrator run?]].

> [!note] What "unsupported" means in practice
> Some of these *do* physically work — a full ARN in a `FunctionName` parameter
> is passed through to the Lambda SDK and a cross-Region invoke may succeed.
> The docs say it is unsupported; treat "works today, unsupported, and
> region-pinning your failover path" as three independent reasons not to do it.
> **No public AWS statement was found promising cross-Region service
> integrations as a roadmap item.** Do not design assuming one arrives.

### The failure modes, ranked by how much they hurt

1. **Primary fully dark.** The standby state machine calls the primary's Lambda, gets a connection error, retries through its `Retry` block, and fails. **Loud. Recoverable. The best case.**
2. **Primary partially impaired — the real nightmare.** The standby's executions invoke the primary's Lambda, which *works*, and write to the primary's DynamoDB table, which *works*. Traffic has moved. The data has not. **Your failover silently did nothing** and you are now writing production data into the Region you just evacuated, with a promoted standby database sitting empty beside it. You will discover this by reconciliation, days later. This is the worst outcome in this note and the only defence against it is the parameterisation below.
3. **Cross-Region latency and egress.** Even when everything "works", each hop is a cross-Region round trip plus inter-Region data transfer charges. Slow and billed. [[cost-model]].
4. **IAM denies it.** If the standby's execution role is correctly scoped to standby-Region ARNs (it should be — see below), a mirrored primary ARN produces `AccessDenied`. **This is a feature.** See the guard rail.

### Terraform: parameterise every ARN, then prove it

The module renders the ASL from a template and injects a region-scoped set of ARNs. The point is that **the definition cannot contain a literal Region**, because there is nowhere for one to come from.

```hcl
# modules/state-machine/variables.tf
variable "name" {
  type        = string
  description = "Unqualified state machine name. MUST be identical in both regions — see the ForceNew note in the migration section."
}

variable "region" {
  type        = string
  description = "The region this instance is deployed into. Injected, never inferred inside the ASL."
}

variable "definition_template" {
  type        = string
  description = "Path to the .asl.json.tftpl template. The template MUST NOT contain a literal region string."
}

# Everything the ASL is allowed to reference, resolved per-region by the CALLER.
variable "integration_arns" {
  type        = map(string)
  description = <<-EOT
    Map of logical name -> fully-qualified ARN/URL, resolved in THIS region.
    e.g. { validate_order = "arn:aws:lambda:eu-west-2:...:function:validate-order" }
  EOT

  validation {
    condition     = alltrue([for a in values(var.integration_arns) : can(regex("^(arn:aws:|https://)", a))])
    error_message = "integration_arns values must be ARNs or URLs."
  }
}

variable "workflow_type" {
  type    = string
  default = "STANDARD"
  validation {
    condition     = contains(["STANDARD", "EXPRESS"], var.workflow_type)
    error_message = "STANDARD or EXPRESS. Note: this is ForceNew and immutable in the service."
  }
}
```

```hcl
# modules/state-machine/main.tf

locals {
  definition = templatefile(var.definition_template, {
    region  = var.region
    account = data.aws_caller_identity.this.account_id
    arns    = var.integration_arns
  })
}

data "aws_caller_identity" "this" {}
data "aws_region" "this" {}

# ---------------------------------------------------------------------------
# GUARD RAIL 1 — the region we were told matches the provider we were given.
# Catches the classic copy-paste where a module block keeps provider = aws.eu_w1
# but the caller passes region = "eu-west-2".
# ---------------------------------------------------------------------------
resource "terraform_data" "assert_region" {
  lifecycle {
    precondition {
      condition     = data.aws_region.this.name == var.region
      error_message = "var.region (${var.region}) != provider region (${data.aws_region.this.name}). The provider alias and the region variable have diverged."
    }
  }
}

# ---------------------------------------------------------------------------
# GUARD RAIL 2 — THE IMPORTANT ONE.
# No ARN in the rendered definition may name a region other than this one.
# This is the check that would have caught the silent-no-op failover.
# ---------------------------------------------------------------------------
resource "terraform_data" "assert_no_foreign_region_arns" {
  lifecycle {
    precondition {
      condition = length([
        for m in regexall("arn:aws:[a-z0-9-]+:([a-z]{2}-[a-z]+-[0-9]):", local.definition)
        : m[0] if m[0] != var.region
      ]) == 0
      error_message = <<-EOT
        The rendered ASL references a region other than ${var.region}.
        A mirrored state machine that points at the primary is the single most
        common multi-region Step Functions bug. Parameterise the ARN.
      EOT
    }
  }
}

resource "aws_sfn_state_machine" "this" {
  name       = var.name              # ForceNew
  type       = var.workflow_type     # ForceNew, and immutable in the service
  role_arn   = aws_iam_role.execution.arn
  definition = local.definition
  publish    = true

  logging_configuration {
    log_destination        = "${aws_cloudwatch_log_group.this.arn}:*"
    include_execution_data = false   # see the PII note in Gotchas
    level                  = "ERROR" # ALL in the standby during a game day
  }

  tracing_configuration { enabled = true }

  depends_on = [
    terraform_data.assert_region,
    terraform_data.assert_no_foreign_region_arns,
  ]
}
```

The template itself never names a Region:

```json
// order-fulfilment.asl.json.tftpl
{
  "Comment": "Order fulfilment. No literal region appears in this file.",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "${arns.validate_order}",
        "Payload.$": "$"
      },
      "Next": "ChargeCard"
    },
    "RecordOrder": {
      "Type": "Task",
      "Resource": "arn:aws:states:::aws-sdk:dynamodb:putItem",
      "Parameters": {
        "TableName": "${table_name}",
        "Item": { "...": "..." },
        "ConditionExpression": "attribute_not_exists(pk)"
      },
      "End": true
    }
  }
}
```

And the caller in a cookiecutter monorepo — one `locals` block per Region, one module call per Region, **same module, same template**:

```hcl
# live/<env>/eu/state-machines.tf

locals {
  sm_arns = {
    eu_west_1 = {
      validate_order = module.lambda_eu_west_1.function_arns["validate-order"]
      charge_card    = module.lambda_eu_west_1.function_arns["charge-card"]
      notify_topic   = module.sns_eu_west_1.topic_arns["order-events"]
    }
    eu_west_2 = {
      validate_order = module.lambda_eu_west_2.function_arns["validate-order"]
      charge_card    = module.lambda_eu_west_2.function_arns["charge-card"]
      notify_topic   = module.sns_eu_west_2.topic_arns["order-events"]
    }
  }
}

module "sm_order_fulfilment_primary" {
  source    = "../../../modules/state-machine"
  providers = { aws = aws.eu_west_1 }

  name                = "order-fulfilment"        # SAME NAME both sides
  region              = "eu-west-1"
  definition_template = "${path.root}/asl/order-fulfilment.asl.json.tftpl"
  integration_arns    = local.sm_arns.eu_west_1
}

module "sm_order_fulfilment_standby" {
  source    = "../../../modules/state-machine"
  providers = { aws = aws.eu_west_2 }

  name                = "order-fulfilment"        # SAME NAME both sides
  region              = "eu-west-2"
  definition_template = "${path.root}/asl/order-fulfilment.asl.json.tftpl"
  integration_arns    = local.sm_arns.eu_west_2   # the ONLY difference
}
```

> [!tip] The execution role is your second line of defence — scope it narrowly on purpose
> It is tempting to write `"Resource": "arn:aws:lambda:*:*:function:*"` in the
> standby's execution role so that "it just works". **Do the opposite.** Scope
> the standby's role to `arn:aws:lambda:eu-west-2:<acct>:function:*` and to
> standby-Region table, queue and topic ARNs only. Then a leaked primary ARN
> fails **immediately, loudly and safely** with `AccessDenied` instead of
> silently succeeding. A wildcard role converts failure mode 1 (loud) into
> failure mode 2 (silent, catastrophic). See [[aws-iam]].

> [!tip] And a third: make the standby run
> Both guard rails above are static. Neither proves the state machine works.
> **Run one synthetic execution per state machine per hour in the standby, end
> to end, writing to a throwaway partition key.** It costs a handful of state
> transitions, and it is the only thing that actually verifies that the
> mirrored definition resolves to live standby resources. This is the same
> argument [[route53-application-recovery-controller]] makes against readiness
> checks — config parity is not function. [[dr-testing-and-gamedays]].

## Idempotency, redrive and the re-run decision

The question the reader is really asking: *after a failover, can I just re-run the lost work?*

### `RedriveExecution` — what it is, and why it does not help you

Redrive was released in November 2023 and it is genuinely good — for the problem it solves, which is *not* this one. Verbatim from [Restarting state machine executions with redrive](https://docs.aws.amazon.com/step-functions/latest/dg/redrive-executions.html):

> You can use redrive to restart executions of Standard Workflows that didn't complete successfully in the last 14 days. These include failed, aborted, or timed out executions.

> When you redrive an execution, Step Functions continues the failed execution from the unsuccessful step and uses the same input. Step Functions preserves the results and execution history of the successful steps, which are not rerun when you redrive an execution.

**"Continues from the unsuccessful step" and "does not rerun the successful steps" is exactly the semantics you want after a partial failure.** It is also exactly the semantics that is impossible across a Region boundary, because the record of which steps succeeded lives in the dead Region.

The full eligibility list, verbatim:

> + You started the execution on or after November 15, 2023.
> + The execution status isn't `SUCCEEDED`.
> + The workflow execution hasn't exceeded the redrivable period of 14 days.
> + The workflow execution hasn't exceeded the maximum open time of one year.
> + The execution event history count is less than 24,999.

And three properties that decide the matter:

> Redriven executions use the same state machine definition and execution ARN that was used for the original execution attempt.

> Because redriven executions use the same state machine definition, you must start a new execution if you update your state machine definition.

> Redrive is not supported for Express workflows.

| Property | Consequence for DR |
|---|---|
| Same **execution ARN** | The ARN contains a Region. **Redrive is structurally same-Region.** There is no cross-Region form of this API and there cannot be. |
| Standard only | Express workflows have no redrive at all. |
| 14-day redrivable window | **The clock runs during your outage.** A three-day Regional impairment eats 21% of it. A long recovery plus a slow reconciliation and the window closes on you. |
| Only after the execution *closes* | *"This period starts from the day a state machine completes its execution."* An execution still `RUNNING` in the dead Region is not redrivable — it is not eligible until it fails or times out, and its `TimeoutSeconds` may be the one-year default. |
| Same definition | If you shipped a fix during the incident, redrive runs the **old** definition. |
| < 24,999 events | Long-running workflows near the 25,000 hard history cap cannot be redriven at all. |
| Retry counts reset to 0 | Generous, and means a redrive gets a full fresh set of attempts. |
| State-machine `TimeoutSeconds` reset to 0 | *"When you redrive an execution, the state machine level timeout, if defined, is reset to 0."* A 30-minute workflow timeout starts again — so a redrive three days later gets a full fresh 30 minutes, which is almost certainly not what a business-time-sensitive workflow wants. |

> [!important] Redrive is a **failback** tool
> Frame it correctly in the runbook. The place redrive belongs is *after* the
> primary Region returns: enumerate the executions that failed during the
> outage, decide which ones the standby already completed, `StopExecution` those,
> and redrive the remainder. That is a real and valuable use — but it happens on
> day 3, not at minute 8. **Nothing in the 15-minute failover budget involves
> redrive.** [[failover-runbook-template]] should say so explicitly so nobody
> reaches for it under pressure.

### "Re-running in the standby" is `StartExecution`, and the built-in dedup does not cross Regions

If you want the lost work done *now*, in the standby, you are not redriving. You are calling `StartExecution` on the standby's state machine with a reconstructed input. Which raises the dedup question, and Step Functions has an answer that is very nearly useful:

Verbatim from [`StartExecution`](https://docs.aws.amazon.com/step-functions/latest/apireference/API_StartExecution.html):

> `StartExecution` is idempotent for `STANDARD` workflows. For a `STANDARD` workflow, if you call `StartExecution` with the same name and input as a running execution, the call succeeds and return the same response as the original request. If the execution is closed or if the input is different, it returns a `400 ExecutionAlreadyExists` error. You can reuse the name 90 days after it closes.

> `StartExecution` isn't idempotent for `EXPRESS` workflows.

and on the name's scope:

> For STANDARD workflows, this name must be unique for your AWS account, region, and state machine.

> [!danger] Read that scope again: **account, Region, and state machine**
> The execution name `order-8f3a91` having already run to completion in
> `eu-west-1` places **no constraint whatsoever** on starting an execution
> called `order-8f3a91` in `eu-west-2`. Different Region, different namespace,
> clean slate.
>
> **Step Functions' built-in idempotency is Regional, and therefore contributes
> exactly nothing to failover safety.** Every team that has relied on
> `ExecutionAlreadyExists` as its duplicate-suppression mechanism — and it is a
> common and otherwise sensible pattern — loses that protection at the moment
> it matters most, silently, with no error and no log line.

That said, **deterministic execution naming is still the right practice**, for two reasons that do survive:

1. **Within a Region it prevents the retry storm.** If your API layer retries `StartExecution` during the chaos of a failover, a deterministic name collapses the duplicates.
2. **On failback it is a tripwire.** If the recovered primary is asked to start `order-8f3a91` again, it returns `ExecutionAlreadyExists` and you have caught a double-process.

So: **name executions after the business key, never `uuid4()`.**

```
BAD:   executionName = uuid4()                    # every retry is a new workflow
OK:    executionName = f"order-{order_id}"        # regional dedup, failback tripwire
BEST:  executionName = f"order-{order_id}"  AND  a cross-region idempotency key (below)
```

Note the 80-character name limit and the character restrictions (no `/`, `:`, `#`, whitespace, brackets). A raw UUID fits; a URL or an email address does not. Hash if needed — but hash *deterministically*, and keep the business key readable in the input so a human at 4am can find it.

### The idempotency key, which is the thing that actually works

The only mechanism that makes a lost execution safe to re-run in another Region is an **idempotency key in a store that spans both Regions**. In this estate that is a [[aws-dynamodb]] Global Table, which is already in flight per [[research-brief]].

The pattern, per side-effecting task:

```json
"ChargeCard": {
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "Parameters": {
    "FunctionName": "${arns.charge_card}",
    "Payload": {
      "orderId.$": "$.orderId",
      "amount.$": "$.amount",
      "idempotencyKey.$": "States.Format('charge-{}', $.orderId)"
    }
  },
  "Retry": [
    { "ErrorEquals": ["Lambda.TooManyRequestsException", "Lambda.ServiceException"],
      "IntervalSeconds": 2, "MaxAttempts": 4, "BackoffRate": 2.0 }
  ],
  "Next": "RecordOrder"
}
```

The key is **derived from the input, not generated**, so a re-run in the standby computes the *same* key. The Lambda passes it to Stripe as an `Idempotency-Key` header, or does a DynamoDB conditional `PutItem` with `attribute_not_exists(pk)` before acting. `States.Format` is an intrinsic and needs no code.

For pure-AWS side effects the conditional write *is* the idempotency, and it is free:

```json
"RecordOrder": {
  "Type": "Task",
  "Resource": "arn:aws:states:::aws-sdk:dynamodb:putItem",
  "Parameters": {
    "TableName": "${table_name}",
    "Item": { "pk": { "S.$": "States.Format('ORDER#{}', $.orderId)" } },
    "ConditionExpression": "attribute_not_exists(pk)"
  },
  "Catch": [
    { "ErrorEquals": ["DynamoDB.ConditionalCheckFailedException"],
      "Next": "AlreadyRecorded", "ResultPath": "$.err" }
  ],
  "Next": "NotifyCustomer"
}
```

> [!warning] Global Tables are last-writer-wins, so a conditional write is not a global lock
> A conditional `PutItem` is atomic **within one Region's replica**. Two
> simultaneous conditional writes to the same key in `eu-west-1` and `eu-west-2`
> will *both* succeed locally and then reconcile by last-writer-wins.
> [[aws-dynamodb]] covers this in full. For active/passive it is adequate —
> only one Region is serving at a time — but it is adequacy by operational
> discipline, not by the data store. It is a *further* reason the fencing step
> in [[split-brain-and-fencing]] is not optional.

### The decision tree for a lost execution

Put this in [[failover-runbook-template]]:

```
An execution was open in the failed region when we failed over.
  │
  ├─ Is every task in this state machine idempotent on a business key?
  │     YES ──► Re-run it in the standby via StartExecution with the same
  │             deterministic name. Safe. Automatable.
  │             (And on failback, the primary will reject the duplicate.)
  │
  │     NO  ──► DO NOT re-run it blind. Two options:
  │             (a) Reconcile from the external system of record (Stripe
  │                 settlement report, downstream service's ledger) and
  │                 hand-repair. Slow, correct.
  │             (b) Wait for the primary to return, StopExecution anything
  │                 the standby has since completed, redrive the rest
  │                 (14-day clock permitting). Slower, more correct.
  │
  └─ Do we even KNOW which executions were open?
        Only if the open-execution list was exported BEFORE the outage, or
        the primary's control plane is reachable. ──► see the pre-flight
        export in the Failover procedure below. THIS IS THE STEP PEOPLE MISS.
```

## Standard vs Express

The choice is usually made on cost and duration. **For DR it is a choice about how much in-flight work you can lose and whether you were forced to make it safe.**

| | **Standard** | **Express (async)** | **Express (sync)** |
|---|---|---|---|
| Max duration | **1 year** | 5 minutes | 5 minutes |
| Execution semantics | *"Exactly-once"* | *"At-least-once"* | *"At-most-once"* |
| State persistence | *"Execution state internally persists between state transitions."* | *"Execution state doesn't persist between state transitions."* | Same |
| `StartExecution` idempotency | Automatic on name | *"Idempotency is not automatically managed."* | Not managed |
| Execution history | In Step Functions, **90 days** | **Not captured** — CloudWatch Logs only | Not captured |
| Redrive | Yes, 14 days | **No** | No |
| `.sync` | Yes | **No** | **No** |
| `.waitForTaskToken` | Yes | **No** | **No** |
| Distributed Map | Yes | **No** | **No** |
| Activities | Yes | **No** | **No** |
| Billing | Per state transition | Per execution + duration + memory | Same |
| `StateTransition` quota | Regional — see below | *"Unlimited"* | Unlimited |

All rows verbatim from [Choosing workflow type](https://docs.aws.amazon.com/step-functions/latest/dg/choosing-workflow-type.html) and the quotas page.

### The DR reading of that table

**Express has a bounded RPO. Standard does not.**

An Express workflow cannot be in flight for more than five minutes, so the worst case a Regional failure can destroy is five minutes of Express work. A Standard workflow can be in flight for a year. **Express's exposure is bounded by a hard service limit; Standard's is bounded only by your own discipline.**

Better still, Express *forces you to build the thing that makes failover safe*. AWS is explicit, verbatim:

> Express Workflows use an *at-least-once* model, so an execution could potentially run more than once. The at-least-once model makes Express Workflows better suited for orchestrating **idempotent** actions, such as transforming input data to store in Amazon DynamoDB using a PUT action.

and for Standard:

> The exactly-once model makes Standard Workflows suited to orchestrating **non-idempotent** actions, such as starting an Amazon EMR cluster or processing payments.

> [!important] The sharp version
> **Express workflows make you build idempotency on day one. Standard workflows
> let you skip it — and the bill arrives at failover.**
>
> AWS's guidance is written for the single-Region case, where "exactly-once"
> genuinely holds. Across a Region boundary, *Standard's exactly-once guarantee
> evaporates*: the moment you re-run a lost execution in the standby, the task
> that already ran in the primary runs a second time. **A Standard workflow in a
> multi-Region estate has at-least-once semantics whether you designed for them
> or not.** Teams that chose Standard *specifically because* the actions were
> non-idempotent are the ones most exposed, and they are the ones least likely
> to have noticed.

### Recommendation

**Do not migrate anything to Express for DR reasons.** The type is `ForceNew` in Terraform *and* immutable in the service, Express loses `.sync`, callbacks, Distributed Map, Activities and the retained history, and the history loss alone is disqualifying for a payments workflow. The conversion cost is a rewrite.

Instead:

1. **Keep Standard for anything non-idempotent or auditable** — and treat the exactly-once guarantee as **Region-local only**, which is the honest reading.
2. **Choose Express for new, short, idempotent, high-volume workflows**, and note in the ADR that the DR exposure is capped at five minutes. That is a real benefit worth writing down.
3. **Shrink Standard executions regardless.** Every hour you remove from the p99 execution duration is an hour off your Step Functions RPO. See the `Wait`-state rule above.
4. **Where a Standard workflow is long only because it waits**, split it: a short Standard workflow, then [[aws-eventbridge]] Scheduler, then a second short workflow. Two bounded exposures instead of one unbounded one.

## Execution history is your incident evidence, and it is in the wrong Region

After the incident you will be asked three questions: *what was in flight, what side effects escaped, and who did we take money from without delivering?* All three are answered from execution history.

Verbatim from the [quotas page](https://docs.aws.amazon.com/step-functions/latest/dg/service-quotas.html):

> **Execution history retention time** — 90 days after an execution is closed. After this time, you can no longer retrieve or view the execution history. There is no further quota for the number of closed executions that Step Functions retains.

> To meet compliance, organizational, or regulatory requirements, you can reduce the execution history retention period to 30 days by sending a quota request.

> The change to reduce the retention period to 30 days applies **per account per Region**. To reduce retention across multiple Regions or accounts, submit a separate request for each account-Region combination.

Three DR consequences:

1. **`GetExecutionHistory` is a regional API call against the Region that just failed.** Your evidence and your incident are co-located. If the Region is hard-down, the forensics wait for the Region to come back — and your reconciliation with the payment processor, which is the thing actually costing money, waits with it.
2. **90 days is generous but it is not an archive**, and it is not queryable. You cannot ask "show me every execution that got past `ChargeCard` but not past `RecordOrder` between 02:14 and 02:31". You can page through `ListExecutions` and call `GetExecutionHistory` per execution, at a refill rate of **20/s** for `GetExecutionHistory` and **5/s** (2/s outside the big three Regions) for `ListExecutions`. Enumerating 50,000 executions at 5/s is nearly three hours before you have even started reading them.
3. **A retention reduction to 30 days does not replicate.** If compliance forced 30 days in `eu-west-1`, `eu-west-2` silently sits at 90 until someone raises a second ticket. That is a compliance *failure* in the standby, in the permissive direction — data retained longer than policy allows. Nobody will notice, because nobody looks at the standby.

### What to do instead

**Do not rely on Step Functions' own history as your post-incident evidence. Stream it out.**

```hcl
resource "aws_cloudwatch_log_group" "sfn" {
  name              = "/aws/vendedlogs/states/${var.name}"
  retention_in_days = 30
  kms_key_id        = var.log_kms_key_arn
}

resource "aws_sfn_state_machine" "this" {
  # ...
  logging_configuration {
    log_destination        = "${aws_cloudwatch_log_group.sfn.arn}:*"
    # ERROR is enough to see which state a failed execution died in.
    # ALL doubles as an audit log but is expensive and captures payloads.
    level                  = "ERROR"
    include_execution_data = false
  }
}
```

and then ship those log groups cross-Region — CloudWatch Logs subscription filter → Kinesis/Firehose → an S3 bucket with cross-Region replication, or whatever [[observability-multi-region]] settles on. **The requirement this note adds to that note: the destination must not be in either Region of the pair.**

> [!warning] `include_execution_data = true` puts your payloads in CloudWatch Logs
> Step Functions' logging can include the full state input and output. For an
> order-fulfilment workflow that is names, addresses, partial card data and
> anything else in the payload — now duplicated into a log group, then into an
> S3 bucket, then replicated to a third Region. **That is a data-residency and
> minimisation decision, not a logging decision**, and for the CA pair it is
> the decision: replicating Canadian execution payloads to a third Region
> outside Canada is precisely the thing [[region-pair-selection]] flags. Set it
> to `false` by default and turn it on per-state-machine with a documented
> reason. [[aws-kms]] for the log-group CMK.

> [!tip] The cheap pre-flight that makes the whole reconciliation tractable
> **Export the open-execution list on a schedule, to a bucket outside the
> Region.** A 15-line Lambda on a 5-minute EventBridge schedule, per state
> machine: `list-executions --status-filter RUNNING`, write JSON to S3.
>
> That file is the difference between "we know there were 812 open executions
> and here are their names and inputs" and "we will find out when the Region
> comes back." It costs approximately nothing and it is the single
> highest-value operational change in this note. It also directly feeds the
> decision tree above, and it is what makes a standby re-run *possible at all*
> when the primary's control plane is unreachable.

## `.sync`, Activities and task tokens

Three ways a state machine holds a handle on something outside itself. All three break at a Region boundary, in different and instructive ways.

### `.sync` — long-running jobs you may not be able to cancel

`.sync` is supported on ECS/Fargate, EKS, Batch, Glue, Glue DataBrew, EMR, EMR on EKS, EMR Serverless, Athena, CodeBuild, SageMaker, MediaConvert, Bedrock and nested Step Functions (per the [integration-patterns table](https://docs.aws.amazon.com/step-functions/latest/dg/connect-to-resource.html)). The state machine submits the job and blocks until it finishes.

The DR-relevant paragraph, verbatim:

> If a task using this (`.sync`) service integration pattern is aborted, and Step Functions is unable to cancel the task, **you might incur additional charges from the integrated service.** A task can be aborted if:
> + The state machine execution is stopped.
> + A different branch of a Parallel state fails with an uncaught error.
> + An iteration of a Map state fails with an uncaught error.

> Step Functions will make a **best-effort attempt** to cancel the task... However, it is possible that Step Functions will be unable to cancel the task. Reasons for this include, but are not limited to:
> + Your IAM execution role lacks permission to make the corresponding API call.
> + **A temporary service outage occurred.**

> [!danger] "A temporary service outage occurred" is your exact scenario
> Read the failure mode in order. You `StopExecution` the open executions in the
> failed Region (assuming you can reach the control plane at all). Step Functions
> makes a *best-effort* cancel of each `.sync` task. The integrated service is in
> the same impaired Region. Best-effort fails.
>
> You are now left with **orphaned ECS tasks, Glue jobs and EMR clusters running
> in the Region you just evacuated, with no state machine watching them.**
>
> AWS frames this as a billing problem — *"you might incur additional charges"*.
> **It is a fencing problem.** An orphaned `ecs:runTask.sync` container is a live
> compute process in the dead Region, holding database connections and writing
> rows, at exactly the moment [[split-brain-and-fencing]] needs everything in that
> Region to stop. It will not appear in any inventory of "things to fence",
> because it was created by a workflow, not by Terraform.
>
> **Action: the fence must be enforced at a layer the orphan cannot escape** —
> a security-group lockdown, a revoked database credential, or the pre-armed SCP
> deny in [[split-brain-and-fencing]]. Do not try to fence by cancelling jobs.

One more `.sync` cost, easy to miss:

> When you use the `.sync` service integration pattern, Step Functions uses polling that consumes your assigned quota and events to monitor a job's status.

So a fleet of `.sync` tasks is silently consuming the standby's already-lower API quotas after failover. Relevant to the section below.

### Task tokens — unredeemable in a dead Region

`.waitForTaskToken` pauses an execution until someone calls `SendTaskSuccess` / `SendTaskFailure` with the token. The token is an opaque string bound to a specific task in a specific execution — therefore a specific Region and, verbatim:

> You must pass task tokens from principals within the same AWS account. The tokens won't work if you send them from principals in a different AWS account.

**A token issued by an execution in `eu-west-1` can only be redeemed by an API call to `states.eu-west-1.amazonaws.com`.** There is no cross-Region redemption, and no re-issue: the token *is* the execution's identity.

Where this stings:

| Token holder | What happens at failover |
|---|---|
| An approval email to a human ("click to approve") | The link calls `SendTaskSuccess` in the dead Region. It fails. The human retries, gives up, and pages someone. |
| A partner webhook | The partner's callback 5xx's for the duration of the outage, then succeeds when the Region returns — resuming a workflow you abandoned days ago. **A latent side-effect bomb.** |
| A message on an SQS queue carrying `$$.Task.Token` | The token outlives the queue message's usefulness. Cross-Region queue replication (if any — see [[aws-sqs]] and [[messaging-in-flight-data-loss]]) replicates a token that is **meaningless in the destination Region.** Worse than not replicating it. |
| A `Credentials`-based cross-account call | Same problem plus the same-account restriction above. |

> [!warning] The default timeout on a callback task is effectively forever
> From the docs: a task waiting for a token *"will wait until the execution
> reaches the one year service quota"*, and `HeartbeatSeconds` /
> `TimeoutSeconds` both default to **99,999,999** seconds (≈ 3.17 years,
> clamped by the one-year execution maximum).
>
> **Every `.waitForTaskToken` task in this estate must carry an explicit
> `HeartbeatSeconds` or `TimeoutSeconds`**, with a `Catch` on `States.Timeout`
> routing to a compensating state. Otherwise a Regional outage converts every
> pending callback into an execution that stays open for a year and then wakes
> up. This is a one-line ASL change and it is the cheapest item in this note:
>
> ```json
> "AwaitPartnerConfirmation": {
>   "Type": "Task",
>   "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
>   "HeartbeatSeconds": 3600,
>   "TimeoutSeconds": 86400,
>   "Parameters": {
>     "QueueUrl": "${arns.partner_queue}",
>     "MessageBody": { "orderId.$": "$.orderId", "TaskToken.$": "$$.Task.Token" }
>   },
>   "Catch": [
>     { "ErrorEquals": ["States.Timeout"], "Next": "CompensateUnconfirmedOrder" }
>   ],
>   "Next": "Fulfil"
> }
> ```
>
> Add a `terraform` or CI lint that fails any ASL containing
> `waitForTaskToken` without one of the two timeout fields. It is a
> ten-line `jq` check and it closes a whole class of DR bug.

### Activities — a worker fleet pointed at the wrong Region

An Activity is a regional resource (`arn:aws:states:<region>:<acct>:activity:<name>`) that your own workers long-poll with `GetActivityTask`. Standard workflows only; Express does not support them.

The multi-region shape is unambiguous:

- Create the activity in **both** Regions (it is an `aws_sfn_activity`, trivially mirrored).
- **The `Resource` field in the ASL is a real regional ARN** — so this is the one place the ARN parameterisation above must reach into the `Resource` field itself, not just `Parameters`. The `assert_no_foreign_region_arns` guard rail covers it.
- Your workers must poll the **local** activity ARN. Workers in the standby polling `eu-west-1`'s activity will, in normal operation, silently steal tasks from the primary — and in a failover will poll a dead endpoint.
- Quota: **1,000 activity pollers per ARN**, per Region, non-adjustable. If your worker fleet scales out at failover, check you are not about to exceed it.
- Warm standby: workers must be running in the standby *before* failover, polling an activity that has no tasks. `GetActivityTask` long-polls for 60 seconds and returns empty — **cheap, but not free**, and it consumes the standby's `GetActivityTask` quota, which is **1,500/300 outside the big three Regions versus 3,000/500 inside**.

> [!tip] Activities are the one integration where "just deploy a second copy" is the whole answer
> Unlike `.sync` and task tokens, there is no in-flight state to reason about
> beyond the task itself. Mirror the activity, run the workers warm in the
> standby, point them at the local ARN. If you are choosing between Activities
> and `.waitForTaskToken` for new work and DR is a concern, **Activities are the
> safer pattern** — the worker fleet is mirrorable, a task token is not.

## Distributed Map mid-flight

Distributed Map is where the numbers get large enough that "just re-run it" stops being a sentence anyone can say casually. From the [quotas page](https://docs.aws.amazon.com/step-functions/latest/dg/service-quotas.html):

| Quota | Value | Adjustable |
|---|---|---|
| Maximum number of open Map Runs | **1,000** | No |
| Maximum number of parallel child executions within a single Map Run | **10,000** | No |
| Maximum Number of Items Read per Map Run | **100,000,000** | No |
| Input file data size for a Map Run | **10 GB** | No |
| Maximum redrives of a Map Run | **1,000** | No |
| `StartMapRun` throttle | 25 bucket / 25 refill | **No** |
| Child dispatch rate | *"Step Functions dispatches express executions at up to 1,000 TPS and standard executions at up to 100 TPS"* | — |

A Map Run over 10 million S3 objects, 5,000-way concurrent, when the Region goes dark:

1. **Every child execution is lost**, along with the knowledge of which items they had completed.
2. **The `ItemReader` source is in the primary.** If the manifest or dataset is an S3 bucket in `eu-west-1`, the standby cannot even *start* unless [[aws-s3]] cross-Region replication has caught up. And CRR is asynchronous with no SLA short of S3 RTC — which the 2-hour RPO makes affordable to skip, but then a large recent dataset may be only partly replicated. **Check whether your Map Run inputs are on a replicated prefix.**
3. **The `ResultWriter` output in the primary is partial and, crucially, indistinguishable from complete** unless you wrote a completion marker. A downstream job that reads the results prefix will happily process a half-finished result set.
4. **Redriving is the right tool and you cannot use it.** Redrive of a Map Run *"redrives the unsuccessful child workflow executions in a Map Run"* — precisely the semantics you want, available only in the original Region, for 14 days.
5. **Restarting in the standby re-processes everything.** 10 million items, all of them, including the 6 million already done. At a 100 TPS standard dispatch rate, 10 million children is over **27 hours** of dispatch alone. That is not a failover activity; that is a project.
6. **The `States.DataLimitExceeded` edge case makes it worse.** Even an in-Region redrive reruns *successful* children if the state failed that way: *"If the state failed because of a `States.DataLimitExceeded` error, the Distributed Map state is rerun. This includes the child workflows that were successful in the original execution attempt."*

> [!important] The design rule for Distributed Map in a multi-Region estate
> **Make the per-item work idempotent and checkpoint completion per item, in a
> store that spans Regions.** Then a restart in the standby is a *resume*: each
> child does a conditional write, finds the item already done, and exits in one
> state transition.
>
> Concretely: a [[aws-dynamodb]] Global Table keyed on `(mapRunJobId, itemKey)`,
> written by the child before it acts. The cost is one conditional write per
> item — at 10 million items, real money, and far less than 27 hours of
> re-processing plus an unknown quantity of double side effects.
>
> Alternatively, **make the output the checkpoint**: write results to a
> deterministic S3 key per item and have each child `HeadObject` first. Free-ish,
> and uses infrastructure you already have. Prefer this when the items are pure
> transforms; prefer the DynamoDB ledger when the items have external side
> effects.

> [!note] The honest scoping question
> Large Distributed Map jobs are usually **batch**, and batch usually does not
> need a 15-minute RTO. **Ask whether the Map Run is in scope for failover at
> all.** "We abandon in-flight batch, re-run it from the top tomorrow, and the
> RTO for batch is 24 hours" is a legitimate and much cheaper answer than
> engineering cross-Region resumability — *provided* the per-item work is
> idempotent, which loops back to the rule above. Decide it explicitly and
> write it in the runbook rather than discovering it during an incident.

## Quotas — the EU pair's 6.25× cliff

This is the most concrete, most verifiable and most estate-specific finding in the note, and it is invisible unless you read the quota table two columns at a time.

Verbatim from [Step Functions service quotas](https://docs.aws.amazon.com/step-functions/latest/dg/service-quotas.html), *Quotas related to state throttling*:

| Service metric | Standard bucket | Standard refill/s |
|---|---|---|
| `StateTransition` — **US East (N. Virginia), US West (Oregon), and Europe (Ireland)** | **5,000** | **5,000** |
| `StateTransition` — **All other regions** | **800** | **800** |

and *Quotas related to API action throttling*:

| API | Big-three bucket / refill | All-other-regions bucket / refill |
|---|---|---|
| `StartExecution` (Standard) | 1,300 / **300** | 800 / **150** |
| `StopExecution` | 1,000 / **200** | 500 / **25** |
| `GetActivityTask` | 3,000 / 500 | 1,500 / 300 |
| `SendTaskSuccess` / `SendTaskFailure` / `SendTaskHeartbeat` | 3,000 / 500 | 1,500 / 300 |
| `ListExecutions` | 200 / 5 | 100 / 2 |
| `DescribeExecution` | 300 / 15 | 250 / 10 |
| `RedriveExecution` | 1,300 / 300 | 800 / 150 |

The "big three" are exactly `us-east-1`, `us-west-2` and `eu-west-1`. Now map that onto this estate's pairs:

| Pair | Primary | Standby | `StateTransition` | `StartExecution` refill | `StopExecution` refill | Verdict |
|---|---|---|---|---|---|---|
| **EU** | `eu-west-1` **(big three)** | `eu-west-2` **(other)** | **5,000 → 800** | **300 → 150** | **200 → 25** | 🔴 **Asymmetric. Falls off a cliff.** |
| **US** | `us-east-1` (big three) | `us-west-2` **(big three)** | 5,000 → 5,000 | 300 → 300 | 200 → 200 | 🟢 Symmetric. No cliff. |
| **CA** | `ca-central-1` (other) | `ca-west-1` (other) | 800 → 800 | 150 → 150 | 25 → 25 | 🟠 Symmetric, but **symmetrically low.** |

> [!danger] The EU pair loses 84% of its state-transition throughput at the moment of failover
> `eu-west-1` is one of only three Regions AWS gives 5,000 state transitions per
> second by default. `eu-west-2` is not. **If the EU deployment's steady-state
> Step Functions load is anywhere above 800 transitions/second, the standby will
> throttle from the first minute** — and it will throttle in the most confusing
> possible way, as `ExecutionThrottled` in CloudWatch rather than as an error on
> any individual call.
>
> AWS names the metric explicitly: *"Throttling on the `StateTransition` service
> metric is reported as `ExecutionThrottled` in Amazon CloudWatch."* **Alarm on
> `ExecutionThrottled` in both Regions today**, so you learn your real headroom
> before you need it. It is a free measurement.
>
> The fix is a Service Quotas increase request against `eu-west-2`, raised
> **months in advance**, because a quota increase is not a failover-time action.
> It is `L-137B3F65` (bucket) and `L-AD9B4E93` (refill). Both are marked
> adjustable.

Two compounding traps:

> [!warning] Quota increases are per account, per Region, and they do not replicate
> AWS states it plainly: *"Throttling quotas are per account, per AWS Region."*
> Every increase this company has ever been granted in `eu-west-1`,
> `us-east-1` or `ca-central-1` over the life of the product exists **only
> there**. The standby starts at defaults.
>
> This is the same class of problem [[route53-application-recovery-controller]]
> describes readiness checks solving — *"when ARC detects a mismatch with a
> readiness check, it can take steps to align the quotas for the replicas by
> increasing the lower quota to match the higher quota"* — and that product is
> **closed to new customers.** So the quota-parity job is yours. A scheduled
> Lambda diffing `service-quotas list-service-quotas --service-code states`
> between each pair, alarming on any mismatch, is ~50 lines and is the
> replacement that note recommends. Build it once, point it at every service,
> not just Step Functions.

> [!danger] The auto-raise mechanism cannot help a warm standby — by construction
> Verbatim, from the top of the quotas page:
>
> > **New AWS accounts have reduced state transition quotas. AWS raises these
> > quotas automatically based on your usage.**
>
> **A warm standby has no usage.** The mechanism AWS relies on to grow your
> limits is driven by exactly the signal a passive standby is designed never to
> produce. The standby's quotas will sit at their floor indefinitely, and the
> first traffic they ever see will be 100% of production at 3am.
>
> This generalises well beyond Step Functions and belongs in
> [[lessons-and-antipatterns]]: **any AWS quota that is auto-tuned on observed
> usage is a warm-standby trap.** It is also a concrete argument for the
> synthetic hourly execution recommended above — a trickle of real usage in the
> standby is worth more than its cost.

> [!note] A documented inconsistency between two AWS pages — verify in your own account
> The Step Functions Developer Guide gives `DescribeExecution` as **300 bucket /
> 15 refill** (big three) and **250 / 10** elsewhere. The
> [AWS General Reference](https://docs.aws.amazon.com/general/latest/gr/step-functions.html)
> gives *"DescribeExecution throttle token refill rate per second — Each
> supported Region: 50"*. **These contradict each other.** The two pages agree
> on every other differentiated quota (`StateTransition`, `StartExecution`,
> `StopExecution`, `GetActivityTask`, `SendTask*`, `ListExecutions`,
> `ListActivities`), so the pattern is not in doubt — but the specific number is.
>
> **Do not plan against either. Read your own account's values from the Service
> Quotas console for both Regions of each pair and record them in the runbook.**
> That is a 20-minute task and it replaces every number in this section with a
> fact.

### Non-throughput quotas worth pre-checking in the standby

| Quota | Default | Why it matters at failover |
|---|---|---|
| Open executions | 1,000,000 per Region (adjustable) | If the primary routinely holds hundreds of thousands open, confirm the standby's value matches. |
| Registered state machines | 100,000 per Region | Only a problem for very large estates; check anyway, it is one API call. |
| Execution history size | **25,000 events, hard** | A long workflow that fits in the primary fits in the standby. Not a Region problem, but a redrive blocker at >24,999. |
| Execution history retention | 90 days (adjustable **down** to 30) | Compliance parity, see above. |
| Activity pollers per ARN | 1,000, hard | The standby's worker fleet must fit. |
| Open Map Runs | 1,000, hard | Batch estates only. |

## Fencing Step Functions — you probably cannot

[[failover-orchestration]] step 1 is *fence the primary*. For Step Functions the honest answer is that the fence is unavailable exactly when you need it.

To stop work in the failed Region you would:

1. `ListExecutions --status-filter RUNNING` for every state machine — a **control-plane** call against the impaired Region, refill 5/s (2/s outside the big three).
2. `StopExecution` on each — refill **200/s** in `eu-west-1`, **25/s** in `eu-west-2` and both Canadian Regions.
3. Accept that each `.sync` task gets only a best-effort cancel, which fails during a Regional outage (quoted above).

Two problems:

- **Both APIs are control-plane calls against the Region that is impaired.** If it is hard-down, none of this runs. If it is soft-down — which is the more common and more dangerous case — it half-runs.
- **Even when it works, the arithmetic is not free.** 50,000 open executions in `eu-west-1` at 200 stops/second is **250 seconds** — four minutes out of a 900-second budget, spent on a step that is not on the critical path to serving traffic. And in the reverse direction (failing *back* from `eu-west-2`) the same operation takes **33 minutes** at 25/s.

> [!important] Therefore: do not fence at the Step Functions layer
> Fence at a layer the executions cannot escape — the pre-armed SCP deny, the
> revoked database credential, or the security-group lockdown described in
> [[split-brain-and-fencing]]. An execution that cannot reach the database
> cannot do damage, whether or not you managed to stop it.
>
> Then treat **`StopExecution` sweeps as a failback activity**, run against a
> recovered Region before it is allowed to resume anything. That is the correct
> place for it, it is not time-critical there, and it is the step that prevents
> the resurrection scenario described at the top of this note.
>
> **This is a change to [[failover-orchestration]]'s sequence**: step 1 (fence)
> does not include Step Functions; a new failback step does.

## RPO / RTO analysis

Against the 2h / 15m targets.

### RTO — **passes comfortably, and time is not the risk**

| Element | Pre-provisioned? | Time at failover |
|---|---|---|
| State machine definition | **Yes** — Terraform, always present | 0 |
| IAM execution role | **Yes** — must be pre-created (creating IAM roles at failover is an AWS-documented anti-pattern, see [[failover-orchestration]] gotcha 6) | 0 |
| CloudWatch log group | Yes | 0 |
| Activities registered | Yes | 0 |
| Activity workers running | **Must be** — they are your compute | 0 if warm |
| First `StartExecution` | — | milliseconds |
| Throughput ramp | — | **0 if quotas are right, unbounded if they are not** |

**There is no cold-start, no provisioning step and no propagation delay.** A mirrored state machine is ready the instant something calls `StartExecution`. Step Functions contributes **zero seconds** to the 15-minute budget.

The RTO risk is not duration, it is **correctness**: a state machine that starts instantly and invokes the wrong Region's Lambda has met the RTO and failed the failover. See the whole of [[#Service-integration ARNs are region-specific]].

### RPO — **the 2-hour target does not apply, and pretending it does is the error**

There is no replication to lag. The RPO for Step Functions is the **age of the oldest open execution**, which is:

| Workflow shape | Effective RPO |
|---|---|
| Express | **≤ 5 minutes** (hard service limit) |
| Standard, short-running (p99 < 60 s) | **≤ ~1 minute** — genuinely excellent |
| Standard, with a human-approval callback | **Days to a year** — bounded only by `TimeoutSeconds`, which defaults to ~3 years |
| Standard, with long `Wait` states | **As long as the longest wait** |
| Distributed Map over a large dataset | **The duration of the Map Run** |

> [!important] The correct statement for [[rpo-rto-analysis]]
> **Step Functions does not have an RPO in the replication sense. It has an
> exposure window equal to the p100 open-execution age, and that number is set
> by your workflow design, not by AWS.**
>
> A shop whose Standard workflows all complete in under a minute meets a
> 2-hour RPO with three orders of magnitude to spare and needs none of the
> mitigations in this note. A shop with 30-day approval workflows has a 30-day
> RPO on that workflow and no infrastructure change will move it.
>
> **Go and measure the p100 open-execution age per state machine.** It is one
> CLI call (above) and it tells you which of these two shops you are, which
> determines how much of this note applies.

### Verdict: `meets_targets: conditional`

- **RTO 15m — yes, trivially.** Nothing to provision, nothing to wait for.
- **RPO 2h — yes for the *control plane*** (definitions never diverge), **and for any workflow whose executions complete in under two hours.**
- **No, for long-running Standard executions with non-idempotent tasks.** And this is not fixable by infrastructure. It is fixed by making tasks idempotent or by shortening executions.

## Warm standby shape

What exists in `eu-west-2` while `eu-west-1` is healthy:

| Component | State while idle | Costs money? |
|---|---|---|
| State machine definitions | Deployed, identical, zero executions | **No** — Standard bills per state transition |
| IAM execution roles | Deployed, scoped to standby-Region ARNs | No |
| CloudWatch log groups | Created, empty | Storage only (pennies) |
| Activities | Registered | No |
| Activity workers | **Running and long-polling** | **Yes** — this is compute, count it under [[aws-eks]] |
| Lambda functions the state machines invoke | Deployed | No ([[aws-lambda]] bills on invocation) |
| Hourly synthetic execution | Running | **Yes, and negligible** — see the cost section |
| Quota increases | **Requested and granted in advance** | No |
| Open-execution export Lambda | Running in **both** Regions on a 5-min schedule | Pennies |

> [!tip] Step Functions is very close to free to keep warm
> This is genuinely one of the cheapest services in the estate to run
> active/passive. Standard workflows bill **per state transition** — a state
> machine with zero executions costs **zero**. There is no per-hour charge, no
> provisioned capacity, no minimum. Contrast [[aws-rds-postgres]] (a running
> replica), [[aws-eks]] (a control plane plus nodes), or ARC routing controls
> at $1,825/month.
>
> **Deploy every state machine into the standby on day one.** There is no
> financial argument for waiting, and having them there is a prerequisite for
> the synthetic execution that proves the ARN parameterisation is right.

## The KMS trap — a multi-Region key is not enough

If any state machine uses a customer-managed key (`encryption_configuration`), the mirroring is **not** solved by reaching for a multi-Region KMS key. [[kms-when-to-use-multi-region-keys]] should record this case because it is a good example of why MRKs are only half an answer.

Two region-bearing conditions appear in AWS's own recommended policies:

**1. The encryption context is the state machine ARN.** Verbatim:

> Step Functions provides an encryption context in AWS KMS cryptographic operations, where the key is `aws:states:stateMachineArn` for State Machines or `aws:states:activityArn` for Activities, and the value is the resource Amazon Resource Name (ARN).

AWS's recommended execution-role policy scopes on it:

```json
"Condition": {
  "StringEquals": {
    "kms:EncryptionContext:aws:states:stateMachineArn":
      "arn:aws:states:us-east-1:123456789012:stateMachine:stateMachineName"
  }
}
```

**That ARN contains a Region.** Copy the policy to the standby and every execution fails on `kms:GenerateDataKey`. The MRK replicated fine; the *condition* did not.

**2. `kms:ViaService` is region-qualified.** AWS's example, verbatim: `"kms:ViaService": "states.us-east-1.amazonaws.com"`. A key policy locked to the primary's service endpoint denies the standby outright.

And a warning AWS gives that is easy to read past:

> Adding `kms:ViaService` condition to an **execution role** can prevent a new execution from starting or cause a running execution to fail.

### The fix

Render both conditions per-Region from the same Terraform, listing **both** Regions' ARNs:

```hcl
data "aws_iam_policy_document" "sfn_kms" {
  statement {
    sid     = "AllowStepFunctionsDataKeys"
    effect  = "Allow"
    actions = ["kms:Decrypt", "kms:GenerateDataKey"]
    resources = [var.kms_key_arn]

    condition {
      test     = "StringEquals"
      variable = "kms:EncryptionContext:aws:states:stateMachineArn"
      # BOTH regions. The standby's ARN must be here before the failover,
      # and creating/updating IAM at failover time is an AWS anti-pattern.
      values = [
        "arn:aws:states:${var.primary_region}:${var.account_id}:stateMachine:${var.name}",
        "arn:aws:states:${var.standby_region}:${var.account_id}:stateMachine:${var.name}",
      ]
    }
  }
}
```

Two further KMS facts worth knowing before you adopt CMKs at all:

- **`DescribeKey` is used "to verify if the AWS KMS customer managed key... exists in the account and region."** A single-Region CMK simply is not visible from the standby; you need either an MRK replica or a separate per-Region key. Either works — the per-Region key is simpler and avoids MRK's own complexity; the MRK is better if the same key must decrypt data moved between Regions, which here it does not. **Recommendation: a separate CMK per Region.** See [[kms-when-to-use-multi-region-keys]].
- **Key disabled or pending deletion = total outage.** Verbatim: *"If a AWS KMS key is disabled in AWS KMS, any related running executions will fail. New executions cannot be started."* The standby's key is a single point of failure that nobody exercises. The hourly synthetic execution catches this; nothing else will.
- **`Activity` encryption config is immutable.** *"You cannot update the `encryptionConfiguration` for an activity ARN of an existing activity; you must create a new Activity resource."* A `ForceNew` by another name, and a rename, which means the ASL `Resource` ARN changes too.
- **Encrypted executions lose payload data in EventBridge events.** *"Execution Input, Output, Error, and Cause will not be included for execution status change events for workflows that are encrypted using your customer managed AWS KMS key."* If any failover automation keys off those fields, CMK encryption silently breaks it. [[aws-eventbridge]].

## Migration path from single-region

> [!tip] The good news first: **there is no forced rename and no `ForceNew` in the migration path**
> Unlike [[aws-dynamodb]] Global Tables — where every replica must share one
> table name, which is the entire subject of [[dynamodb-table-naming-migration]]
> — **Step Functions imposes no cross-Region naming requirement whatsoever.**
> Each Region's state machine is an independent resource. If today's names are
> `order-fulfilment-euw1-prod` and you want `order-fulfilment-euw2-prod` in the
> standby, that is completely legal and nothing breaks.
>
> **So the migration is purely additive and there is no downtime.** This is the
> easiest migration in the vault. Do not go looking for the pain that
> [[dynamodb-table-naming-migration]] describes; it is not here.

Step by step:

1. **Inventory.** For every state machine in every primary Region, record: workflow type, p50/p99/**p100 execution duration**, current open-execution count, whether it uses `.sync`, `.waitForTaskToken`, Activities, Distributed Map, or a CMK, and whether every task is idempotent. This is a spreadsheet and it determines everything else. **The idempotency column is the deliverable.**
2. **Grep for region-pinned ARNs.** In the ASL and in whatever generates it:
   ```bash
   grep -rnoE 'arn:aws:[a-z0-9-]+:[a-z]{2}-[a-z]+-[0-9]:' --include='*.json' --include='*.tftpl' --include='*.asl' . | sort -u
   grep -rn 'sqs\.[a-z]\{2\}-[a-z]*-[0-9]\.amazonaws\.com' .   # QueueUrls
   grep -rn '"Resource": *"arn:aws:lambda:'                     # legacy Lambda integrations
   ```
   Every hit is a parameterisation task. **Do this before anything else** — it sizes the real work.
3. **Refactor to a templated definition.** Move the ASL into `.tftpl`, inject `arns` from the caller. Apply to the **primary only**. `definition` is an in-place update, so this is a no-op change to a live state machine: same rendered JSON, zero disruption, and `terraform plan` should show a definition diff only if you accidentally changed something. **Verify the rendered output is byte-identical before applying.**
4. **Add the guard rails** (`assert_no_foreign_region_arns`). Still primary-only. They should pass trivially.
5. **Fix `.waitForTaskToken` timeouts.** Add `HeartbeatSeconds`/`TimeoutSeconds` and a `Catch` to every callback task. In-place. Do this in the primary now — it is a correctness fix regardless of multi-Region.
6. **Add the open-execution export Lambda** to the primary. Five minutes' schedule, writes to a bucket outside the Region. Now you have the artefact the runbook depends on.
7. **Request quota increases in the standby Regions.** `StateTransition` and `StartExecution` for `eu-west-2` especially. Lead time is days-to-weeks, so raise them here, not later.
8. **Deploy the standby.** New module call, second provider alias. **Nothing in the primary changes**; `terraform plan` shows only additions. Costs nothing while idle.
9. **Turn on the hourly synthetic execution in the standby.** This is the acceptance test for step 2 — if any ARN is still pinned to the primary, it fails here, in daylight, three months before the incident.
10. **Diff the quotas** and alarm on `ExecutionThrottled` in both Regions.
11. **Game day.** Start real executions in the standby against the promoted database. [[dr-testing-and-gamedays]].

### If you *do* want to align names (and you should, eventually)

Identical names across Regions are not required, but they make the runbook, the dashboards, the ARN templates and the module signature dramatically simpler — one `var.name`, not two. If today's names are region-suffixed, aligning them **is** a replacement, because `name` is `ForceNew`.

Do not let Terraform do it in one apply. The zero-downtime path:

1. Create the new-named state machine **alongside** the old one, in the same Region. Two state machines, same definition.
2. Move callers to the new ARN (feature flag, or a `StartExecution` wrapper that reads an SSM parameter).
3. **Drain**: wait until the old one has zero `RUNNING` executions. This is the step people skip. From [`DeleteStateMachine`](https://docs.aws.amazon.com/step-functions/latest/apireference/API_DeleteStateMachine.html), verbatim:
   > This is an asynchronous operation. It sets the state machine's status to `DELETING` and begins the deletion process. **A state machine is deleted only when all its executions are completed. On the next state transition, the state machine's executions are terminated.**

   Read the last sentence carefully: deleting a state machine with live executions **kills them at their next state transition**. On a payments workflow that is data loss you inflicted on yourself during a routine rename. **Drain first, always.**
4. Delete the old one, and remove it from Terraform.

Note also that deletion *"deletes all versions and aliases associated with a state machine"*, and history for those executions goes with it.

## Failover procedure

Step Functions is not on the critical path of [[failover-orchestration]]'s six-step sequence. It is a **pre-flight** item and a **post-failover** item.

### Continuously (not at failover time)

- Export open executions every 5 minutes to a bucket outside the Region.
- Run the hourly synthetic in the standby.
- Alarm on `ExecutionThrottled` in both Regions.

### During the failover

| Step | Action | Owner |
|---|---|---|
| Before step 1 (fence) | **Grab the latest open-execution export.** It is already in S3; you are just noting the object key and the timestamp. 10 seconds. | Orchestrator |
| Step 1 (fence) | **Do not attempt `StopExecution` sweeps.** See [[#Fencing Step Functions]]. Fence at the SCP / credential / SG layer instead. | Orchestrator |
| Step 4 (enable consumers) | Step Functions has no "enable" switch — it is driven by whatever calls `StartExecution`. **The thing to enable is the caller**: the [[aws-eventbridge]] rule, the [[aws-lambda]] event source mapping, the API route. [[failover-orchestration]] step 4 already covers this; just make sure state-machine triggers are on the list. | Orchestrator |
| Step 6 (validate) | **The synthetic execution is your validation probe.** It already exists and it exercises a real write. Run it once, synchronously, and assert `SUCCEEDED`. | Orchestrator |

### After traffic is stable (hours, not minutes)

1. Read the open-execution export. This is your worklist.
2. Split it by the decision tree above: idempotent → re-run in the standby; non-idempotent → reconcile from the external system of record.
3. **Start the clock on the 14-day redrive window** and put the expiry date in the incident channel. It is the only hard deadline in the recovery.

## Failback

Failback is where Step Functions gets genuinely dangerous, and it is the reason this section exists at all.

> [!danger] The recovered Region will resume its executions
> When `eu-west-1` comes back, its open Standard executions are **still open**,
> their state **still persisted**, and they will continue from wherever they
> were — invoking Lambdas, writing to a database that is now a replica again,
> charging cards for orders the standby already fulfilled.
>
> **The very first action on the recovered Region — before restoring
> replication, before any traffic, before anything — is to stop them.**

The ordered failback procedure for Step Functions specifically:

1. **Keep the recovered Region fenced.** It comes back fenced and stays fenced until step 3 is done.
2. **Enumerate.** `list-executions --status-filter RUNNING` for every state machine in the recovered Region. Diff against the pre-incident export to see what started *during* the impairment.
3. **Stop everything.** `StopExecution` on all of them, with a `--cause` string naming the incident so the history is self-documenting:
   ```bash
   aws stepfunctions list-executions --region eu-west-1 \
     --state-machine-arn "$SM_ARN" --status-filter RUNNING \
     --query 'executions[].executionArn' --output text |
   tr '\t' '\n' |
   xargs -P 8 -I{} aws stepfunctions stop-execution --region eu-west-1 \
     --execution-arn {} \
     --error "HeliosFailover" \
     --cause "Stopped during INC-1234 failback. Work completed in eu-west-2."
   ```
   Budget this: 200 stops/second in `eu-west-1`, so 50,000 executions is ~4 minutes of API calls. **Rate-limit your parallelism** — `-P 8` above is deliberate; a naive `-P 100` will throttle and you will not notice which ones failed.
4. **Accept the orphans.** Any `.sync` ECS/Glue/EMR job that Step Functions could not cancel is still out there. Sweep them separately at the service level. Budget for the bill.
5. **Now decide about redrive.** Executions you just stopped are `ABORTED`, therefore *eligible* for redrive for 14 days from the stop. **Only redrive the ones the standby did not already complete** — the decision tree again. Redrive resumes from the failed step, which is exactly right for the ones that genuinely need finishing.
6. **Only now restore replication and consider moving traffic back.** [[failover-orchestration]] and [[split-brain-and-fencing]] own the rest.

> [!note] Failback is cheap for Step Functions in one respect
> There is no reseed, no replica rebuild, no data copy. The state machines in
> the recovered Region are already correct — they never diverged, because they
> are Terraform. **The entire failback cost is the execution sweep and the
> reconciliation.** Compared with [[aws-rds-postgres]] failback this is a
> pleasant afternoon.

---

# Part 2 — Step Functions as the failover orchestrator

[[failover-orchestration]] lists Step Functions as Option 3 in a field of five and gives it four sentences. It deserves more, because on paper it is the best fit of the five and the reasons not to use it are non-obvious.

## Why it looks right

[[failover-orchestration]] establishes the decisive constraint itself, verbatim from that note:

> **The honest weakness of SSM Automation: `mainSteps` run sequentially.** There is no native parallel construct, and the whole 15-minute budget depends on running promote and scale-out concurrently.

and, from its time budget:

> | | **Serialised total** | **~16–22 min — FAILS** |
> | | **Parallelised total** (2 ∥ 3) | **≈ 9.5–14 min — PASSES, with no slack** |

**The difference between meeting and missing the RTO is a `Parallel` state.** Step Functions has one natively. That is a real, specific, budget-relevant advantage and it is the reason this option cannot be dismissed casually.

It brings four more things a shell script does not:

| Capability | Why it matters at 3am |
|---|---|
| `Parallel` state | Promote ∥ scale. **Six minutes of the budget.** |
| `Retry` with `BackoffRate`/`MaxAttempts` per state | Declarative, per-step, auditable. A transient `ThrottlingException` on `promote-read-replica` does not end the failover. |
| `Catch` per state routing to a compensating state | Every step has a written rollback target — which is exactly the table [[failover-orchestration#The rollback points, stated plainly]] asks for, expressed as code rather than prose. |
| `.waitForTaskToken` | Human approval as a first-class state, with `HeartbeatSeconds` to bound it. |
| Execution history + graph view | An auditor artefact and a live incident display. Somebody can *watch the failover happen* on a screen. |

### Real ASL — the EU failover plan

This is the shape, written honestly, with the cross-Region constraints from Part 1 respected. **Note how much of it has to go through Lambda**; that is the finding, not an accident of style.

```json
{
  "Comment": "Helios EU-pair failover. Deployed in eu-west-2 (standby) and eu-central-1 (third).",
  "StartAt": "AcquireFailoverLock",
  "TimeoutSeconds": 2700,
  "States": {

    "AcquireFailoverLock": {
      "Comment": "Three orchestrators, one outcome. StartExecution name-idempotency is REGIONAL and does not help. See the mutual-exclusion gotcha.",
      "Type": "Task",
      "Resource": "arn:aws:states:::aws-sdk:dynamodb:putItem",
      "Parameters": {
        "TableName": "helios-failover-lock",
        "Item": {
          "pk":        { "S": "eu" },
          "incident":  { "S.$": "$.incidentRef" },
          "claimedBy": { "S.$": "$$.Execution.Id" }
        },
        "ConditionExpression": "attribute_not_exists(pk)"
      },
      "Catch": [
        { "ErrorEquals": ["DynamoDB.ConditionalCheckFailedException"],
          "Next": "AnotherFailoverInProgress" }
      ],
      "Next": "FreezePipeline"
    },

    "FreezePipeline": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": { "FunctionName": "${arns.freeze_pipeline}" },
      "TimeoutSeconds": 60,
      "Retry": [{ "ErrorEquals": ["States.TaskFailed"], "IntervalSeconds": 2, "MaxAttempts": 2 }],
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "FencePrimary", "ResultPath": "$.freezeError" }],
      "Next": "FencePrimary"
    },

    "FencePrimary": {
      "Comment": "Acts on eu-west-1 from eu-west-2. MUST be a Lambda: SDK integrations cannot target another region.",
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "${arns.fence_primary}",
        "Payload": { "targetRegion": "eu-west-1", "incidentRef.$": "$.incidentRef" }
      },
      "TimeoutSeconds": 180,
      "Comment2": "onFailure equivalent of Continue: the primary may already be unreachable, which is fine.",
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "AwaitApproval", "ResultPath": "$.fenceError" }],
      "Next": "AwaitApproval"
    },

    "AwaitApproval": {
      "Comment": "THE ONE-WAY DOOR. Fencing is done and reversible; promotion is not.",
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish.waitForTaskToken",
      "TimeoutSeconds": 600,
      "Parameters": {
        "TopicArn": "${arns.approval_topic}",
        "Subject": "FAILOVER APPROVAL REQUIRED — EU pair",
        "Message": {
          "incidentRef.$": "$.incidentRef",
          "approveUrl.$": "States.Format('${approval_base_url}/approve?token={}', $$.Task.Token)",
          "rejectUrl.$":  "States.Format('${approval_base_url}/reject?token={}',  $$.Task.Token)",
          "warning": "Approving PROMOTES the eu-west-2 replica. For RDS Postgres this is IRREVERSIBLE. You are accepting up to 2h of data loss.",
          "checklist": [
            "External probe from a THIRD region failing >= 5 min",
            "AWS Health Dashboard shows an event in eu-west-1",
            "Replica lag < 2h",
            "FencePrimary succeeded, or eu-west-1 confirmed unreachable"
          ]
        }
      },
      "Catch": [
        { "ErrorEquals": ["States.Timeout"], "Next": "AbortNoApproval" },
        { "ErrorEquals": ["States.ALL"],     "Next": "AbortNoApproval" }
      ],
      "Next": "PromoteAndScale"
    },

    "PromoteAndScale": {
      "Comment": "THE REASON THIS OPTION EXISTS. Serialised these are ~10 min; parallel they are ~5.",
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "PromoteDatabase",
          "States": {
            "PromoteDatabase": {
              "Type": "Task",
              "Resource": "arn:aws:states:::aws-sdk:rds:promoteReadReplica",
              "Parameters": { "DbInstanceIdentifier": "${db_standby_id}" },
              "Retry": [
                { "ErrorEquals": ["Rds.RdsException"], "IntervalSeconds": 5,
                  "MaxAttempts": 3, "BackoffRate": 2.0 }
              ],
              "Next": "WaitForWritable"
            },
            "WaitForWritable": {
              "Comment": "promoteReadReplica returns immediately. Poll until pg_is_in_recovery() is false.",
              "Type": "Task",
              "Resource": "arn:aws:states:::lambda:invoke",
              "Parameters": { "FunctionName": "${arns.await_db_writable}" },
              "TimeoutSeconds": 600,
              "Retry": [
                { "ErrorEquals": ["NotYetWritable"], "IntervalSeconds": 10,
                  "MaxAttempts": 60, "BackoffRate": 1.0 }
              ],
              "End": true
            }
          }
        },
        {
          "StartAt": "ScaleNodeGroup",
          "States": {
            "ScaleNodeGroup": {
              "Type": "Task",
              "Resource": "arn:aws:states:::aws-sdk:eks:updateNodegroupConfig",
              "Parameters": {
                "ClusterName": "${eks_standby_cluster}",
                "NodegroupName": "${eks_nodegroup}",
                "ScalingConfig": { "MinSize": ${ng_target}, "DesiredSize": ${ng_target} }
              },
              "Next": "AwaitNodesReady"
            },
            "AwaitNodesReady": {
              "Type": "Task",
              "Resource": "arn:aws:states:::lambda:invoke",
              "Parameters": { "FunctionName": "${arns.await_nodes_ready}" },
              "TimeoutSeconds": 600,
              "End": true
            }
          }
        }
      ],
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "HaltForHuman", "ResultPath": "$.promoteError" }],
      "ResultPath": "$.promoteAndScale",
      "Next": "GateOnPromotion"
    },

    "GateOnPromotion": {
      "Comment": "CLIENT-SIDE gate. Weaker than an ARC gating rule — see the safety section.",
      "Type": "Choice",
      "Choices": [
        { "And": [
            { "Variable": "$.promoteAndScale[0].writable", "BooleanEquals": true },
            { "Variable": "$.promoteAndScale[1].nodesReady", "BooleanEquals": true }
          ],
          "Next": "EnableConsumers" }
      ],
      "Default": "HaltForHuman"
    },

    "EnableConsumers": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": { "FunctionName": "${arns.enable_consumers}" },
      "TimeoutSeconds": 180,
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "HaltForHuman", "ResultPath": "$.consumerError" }],
      "Next": "ShiftTraffic"
    },

    "ShiftTraffic": {
      "Comment": "MUST be a Lambda. ARC requires addressing a specific cluster --endpoint-url, and AWS SDK service integrations provide no endpoint override. See the ARC note below.",
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": { "FunctionName": "${arns.arc_update_routing_controls}" },
      "TimeoutSeconds": 120,
      "Retry": [{ "ErrorEquals": ["States.ALL"], "IntervalSeconds": 3, "MaxAttempts": 5, "BackoffRate": 1.5 }],
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "HaltForHuman", "ResultPath": "$.trafficError" }],
      "Next": "ValidateSyntheticWrite"
    },

    "ValidateSyntheticWrite": {
      "Comment": "A REAL transaction. Not a liveness probe. See failover-orchestration step 6.",
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": { "FunctionName": "${arns.synthetic_write}" },
      "TimeoutSeconds": 300,
      "Retry": [{ "ErrorEquals": ["States.ALL"], "IntervalSeconds": 15, "MaxAttempts": 6, "BackoffRate": 1.0 }],
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "HaltForHuman", "ResultPath": "$.validateError" }],
      "Next": "Succeeded"
    },

    "HaltForHuman": {
      "Comment": "NEVER auto-rollback past the one-way door. Page and stop.",
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "${arns.approval_topic}",
        "Subject": "FAILOVER HALTED — MANUAL INTERVENTION REQUIRED",
        "Message.$": "States.JsonToString($)"
      },
      "Next": "Failed"
    },

    "AnotherFailoverInProgress": { "Type": "Fail", "Error": "LockHeld",
      "Cause": "Another orchestrator holds the EU failover lock. Check before overriding." },
    "AbortNoApproval": { "Type": "Fail", "Error": "NoApproval",
      "Cause": "Approval timed out or was rejected. Primary remains fenced — UNFENCE IT if standing down." },
    "Failed":    { "Type": "Fail", "Error": "FailoverIncomplete" },
    "Succeeded": { "Type": "Succeed" }
  }
}
```

> [!warning] Four things that ASL taught us while writing it
> 1. **`AbortNoApproval` leaves the primary fenced.** Step 1 is reversible, but *someone must actually reverse it*. A `Fail` state does not un-fence. The `Cause` string carries the instruction because there is nowhere better to put it. **This is a real hazard of encoding a runbook as a state machine: the failure paths get less attention than the happy path, and the failure paths are the whole point.**
> 2. **Four of the nine working states are `lambda:invoke`.** Fencing (cross-Region), waiting for the DB (polling), waiting for nodes (polling) and the ARC flip (endpoint override) all need code. Step Functions is orchestrating Lambdas, not AWS APIs. **That undercuts the "declarative runbook" pitch substantially** — the logic that matters is in Python, same as SSM's `executeScript`.
> 3. **`Retry` on `NotYetWritable` with `BackoffRate: 1.0` is a polling loop** pretending to be error handling. It works, and it is billed as state transitions, and it is less readable than a `Wait`/`Choice` loop. Both are ugly. `.sync` would solve it but there is no `.sync` for RDS promotion.
> 4. **Verify `DbInstanceIdentifier`.** AWS states the convention — *"Parameters in Step Functions are expressed in PascalCase, even if the native service API is in camelCase"* — but the RDS API member is `DBInstanceIdentifier` with three capitals, and how Step Functions normalises that specific member was **not confirmed against a documented example**. Test it in a sandbox before it is on your critical path. This is precisely the kind of detail that is fine in a blog post and unacceptable in a runbook.

## Where does the orchestrator run?

The obvious flaw, and it is fatal in the form people first reach for.

### The three placements

**In the primary — useless, and this is not a strawman.** It is where the Lambdas already are, where the ARNs already resolve, and where a developer adding "a failover state machine" to the existing stack will naturally put it. A state machine in `eu-west-1` cannot orchestrate its own Region's evacuation.

**In the standby — workable, and the default answer.** By hypothesis the standby is healthy. `eu-west-2` can promote `eu-west-2`'s database and scale `eu-west-2`'s node group using native SDK integrations, which is the cheap path. But:

- **Fencing the primary must go through a Lambda.** *"Step Functions doesn't support referencing ARNs across partitions or regions."* Every cross-Region action needs a function with an explicit region override, which is the CDK `CallAwsServiceCrossRegion` pattern by hand.
- **The orchestrator inherits every Part 1 hazard.** Its own ARNs must be parameterised; its own KMS conditions must name the right Region; its own `StartExecution` name-idempotency does not span Regions.
- **It does not cover the bilateral case** — an event affecting both Regions of the pair, or an `eu-west-2` impairment that makes you want to *not* fail over.

**In a third Region — covers the bilateral case, at a real cost.** `eu-central-1` for EU, `us-east-2` for US. And then:

> [!danger] A third-Region Step Functions orchestrator cannot use SDK integrations for anything
> This follows directly from the cross-Region restriction. A state machine in
> `eu-central-1` acting on `eu-west-2` resources must proxy **every single call**
> through a Lambda with an explicit region override.
>
> At that point the state machine is a sequencer for a bag of Lambdas and you
> have lost the declarative-AWS-API property that was the main aesthetic
> argument for choosing it over SSM. **The third-Region placement — the one that
> covers the failure mode you cannot otherwise cover — is the placement where
> Step Functions' advantages are smallest.**
>
> And for the **CA pair there is no third Canadian Region at all**.
> [[failover-orchestration]] already flags this as a data-residency question
> needing a legal answer; Step Functions does not change it, and Step Functions'
> answer to it is worse than ARC's, because ARC requires no placement decision.

### The bootstrapping problem

Even with the state machine in the right Region, something has to call `StartExecution`, and that something has its own dependencies:

| Dependency | Failure | Mitigation |
|---|---|---|
| **Credentials** | The IdP federates through an impaired Region; nobody can authenticate to start the failover | Break-glass credentials in a vault. [[aws-iam]], and [[failover-orchestration]] gotcha 8. |
| **STS endpoint** | The SDK default global STS endpoint is `us-east-1` | Regional STS endpoints. [[failover-orchestration]] gotcha 7. |
| **Console access** | The Step Functions console is regional; reaching `eu-west-2`'s console still needs sign-in, which needs IAM | CLI + break-glass, rehearsed |
| **The inputs** | `incidentRef`, ARNs, cluster names — if read from Terraform state in the primary's S3, you have the athenahealth circular dependency verbatim | **Bake every input into the definition at plan time.** The `${...}` template variables above are literals by the time the state machine exists. |
| **The Lambda code** | Packaged as a container image in the primary's ECR | [[aws-ecr]] cross-Region replication, or zip-packaged Lambdas only in the recovery path |
| **Secrets the Lambdas read** | SSM Parameter Store has no native cross-Region replication ([[aws-ssm-parameter-store]]) | [[aws-secrets-manager]] replicas — already solved in this estate |

**None of these are Step Functions problems specifically.** They are *self-hosted orchestrator* problems, and they are identical for SSM Automation. That is the point: **the placement and bootstrapping work is the same whichever self-hosted option you pick, and it is work that ARC Region switch does not require at all.**

### Mutual exclusion: three orchestrators, one outcome

If you deploy to the standby *and* a third Region — which you must, to cover the bilateral case — two people can start two failovers. [[failover-orchestration]] gotcha 12 raises this and suggests a DynamoDB Global Table lock row. The `AcquireFailoverLock` state above implements it.

**Be honest about how strong that lock is: not very.**

- A conditional `PutItem` is atomic within one replica. Two orchestrators writing the same key in two Regions simultaneously **both succeed**, and Global Tables reconciles by last-writer-wins. [[aws-dynamodb]].
- **Step Functions' own `StartExecution` name-idempotency does not help**, because it is per-Region (Part 1).
- The window is small — minutes apart in practice, because a human starts each one — but it is not zero, and "small window" is exactly what the GitHub 43-second incident was.

So: the lock is a **tripwire, not a mutex**. It will catch the realistic case (two engineers, two minutes apart) and will not catch the adversarial one. Write it that way in the runbook, and keep the human coordination step — a named incident commander — because that is what actually enforces exclusivity.

## Against the alternatives

[[failover-orchestration]] has a five-way table. This section only argues the three that are live, and only on the axes where they actually differ.

| | **Step Functions** | **ARC Region switch** | **SSM Automation** | **Human runbook** |
|---|---|---|---|---|
| **Data plane in every Region** | ❌ You deploy and maintain N copies | ✅ **AWS's problem** | ❌ You deploy N copies | ✅ Trivially |
| **Parallel steps** | ✅ Native `Parallel` | ✅ Native (sequence or parallel) | ❌ **Sequential only** | ❌ Humans serialise |
| **Fits the 900 s budget** | ✅ ~9.5–14 min | ✅ ~9.5–14 min | ⚠️ Only via a Python fan-out blob | ❌ ~16–22 min |
| **Human approval** | ✅ `.waitForTaskToken` | ✅ Manual Approval block | ✅ `aws:approve` (7-day default — override) | ✅ It *is* the human |
| **Ordering enforced server-side** | ❌ `Choice` is client-side | ✅ **ARC gating rules** | ❌ | ❌ |
| **Cross-Region API calls** | ❌ **Not supported — Lambda proxy required** | ✅ By design | ⚠️ Via `executeScript` | ✅ |
| **Readable as a runbook** | ❌ ASL JSON | ⚠️ Console/plan view | ✅ **Best — it *is* the runbook** | ✅ |
| **Rehearsal mode** | ⚠️ Run it in non-prod | ✅ **Practice mode; graceful/ungraceful** | ⚠️ Non-prod | ✅ Game day |
| **Measures your RTO** | ❌ (execution duration ≈ it) | ✅ **Automatically** | ❌ | ❌ |
| **Terraform** | ✅ `aws_sfn_state_machine`, mature | ⚠️ Provider support announced Dec 2025 — **verify your pin** | ✅ `aws_ssm_document`, mature | N/A |
| **Maturity** | ✅ Since 2016 | ⚠️ GA Aug 2025 | ✅ Very | ✅ |
| **Cost** | ~$0 | $70/plan/month ($210 for three pairs) | ~$0 | $0 |

### The argument that settles it

Read the first three rows together.

**Step Functions' entire case over SSM is the `Parallel` state**, because that is what turns a 16–22-minute serialised sequence into a 9.5–14-minute one. Everything else — retries, catches, approval gates, Terraform support, observability — SSM has in some adequate form, and SSM is *more readable*, which for a runbook is worth real money.

**ARC Region switch has native parallel steps too.** From the [Region switch docs](https://docs.aws.amazon.com/r53recovery/latest/dg/region-switch.html), a plan contains workflows made of steps containing execution blocks, *"run in sequence or in parallel"*.

> [!important] So Step Functions' one decisive advantage evaporates the moment you choose Region switch
> And Region switch additionally brings the thing **no self-hosted option can
> offer at any price** — verbatim:
>
> > A data plane in each AWS Region, so that you can execute your Region switch
> > plan without taking a dependency on the Region that you're deactivating.
>
> That single sentence is worth more than every feature in the table above,
> because it deletes the *entire* "Where does the orchestrator run?" section —
> the placement decision, the third-Region deployment, the CA pair's
> no-third-Canadian-Region problem, the mutual-exclusion lock, the
> bootstrapping checklist, and the ongoing cost of keeping three copies of a
> state machine in sync forever.
>
> **[[route53-application-recovery-controller]] recommends Region switch. This
> note independently reaches the same conclusion by a different route** — not
> "AWS-managed is better" but "Step Functions' only real advantage over the
> cheaper, more readable alternative is also present in Region switch, so
> Step Functions is dominated."

### Where Step Functions still wins, honestly

Three cases, and they are not nothing:

1. **You reject Region switch on maturity grounds.** GA August 2025, capabilities added December 2025, Terraform coverage unverified. A conservative estate may reasonably refuse to put a service that young on the critical path of its DR plan. **If Region switch is off the table, the fork is genuinely Step Functions vs SSM**, and then the `Parallel` state matters again — see the hybrid recommendation below.
2. **Genuine fan-out over a variable-length list.** Forty microservices to scale, sixty event source mappings to enable, a list that changes every sprint. An `inline Map` over a dynamic array is the right tool and neither SSM nor a Region switch execution block expresses it as cleanly.
3. **As a sub-step, invoked by something else.** See the recommendation.

### And a genuinely useful integration nobody mentions

Step Functions has AWS SDK integrations for **both** ARC APIs:

```
arn:aws:states:::aws-sdk:arcregionswitch:{apiAction}
arn:aws:states:::aws-sdk:route53recoverycluster:{apiAction}
```

(from the [supported SDK integrations list](https://docs.aws.amazon.com/step-functions/latest/dg/supported-services-awssdk.html)).

So a state machine **can** invoke a Region switch plan directly — which makes the hybrid below cheap to build.

> [!warning] But do not use `route53recoverycluster` via the SDK integration for the traffic flip
> ARC routing controls require addressing **a specific cluster endpoint URL**,
> and the `--region` must match it. [[route53-application-recovery-controller]]
> quotes AWS verbatim: *"You must specify one of these Regional endpoints (the
> AWS Region and the endpoint URL) when you make calls to the cluster."* It also
> records the failure mode: *"Omit `--endpoint-url` and the CLI silently talks
> to the default regional service endpoint, which is not your cluster. You get
> an error at 3am that reads like a permissions problem."*
>
> **No documented mechanism was found for supplying an endpoint-URL override to
> a Step Functions AWS SDK integration.** The integration surface is the API's
> parameters, and the endpoint is not one of them. That means the SDK
> integration would call the default regional endpoint — i.e. **the exact
> documented failure mode**, silently, in the middle of your failover.
>
> Corroborating evidence: AWS's own reference implementation, the
> [networking blog on orchestrating DR with ARC and Step Functions](https://aws.amazon.com/blogs/networking-and-content-delivery/orchestrate-disaster-recovery-automation-using-amazon-route-53-arc-and-aws-step-functions/),
> stores *"the Route 53 ARC regional cluster endpoints, control panel ARN, and
> the order of Routing Controls in the global DynamoDB tables"* — i.e. it reads
> endpoints from a store and calls them from code, because it has to.
>
> **Use a Lambda for the ARC flip. Every time. It must rotate all five endpoints
> anyway**, which an SDK integration could not express either.

## Safety gating, and where the human belongs

### `Choice` is a client-side check and must be labelled as one

The `GateOnPromotion` state above encodes "do not shift traffic until the database is promoted". It is real and it is worth having. It is also **strictly weaker than the ARC gating rule** that [[route53-application-recovery-controller]] describes, and the difference is not cosmetic.

| | **ASL `Choice` gate** | **ARC gating rule** |
|---|---|---|
| Enforced where | Inside this state machine's execution | **ARC data plane, server-side, on the transaction** |
| Blocks a human running the CLI by hand? | ❌ **No — completely bypassed** | ✅ Yes — the API rejects the call |
| Blocks a *second* orchestrator? | ❌ No | ✅ Yes |
| Blocks a panicking engineer at 3am? | ❌ No | ✅ Yes, and tells them which rule blocked it |
| Overridable deliberately | N/A | ✅ `--safety-rules-to-override` |
| Cost | $0 | Requires a **$1,825/month** routing-control cluster |

That ARC note's verdict is *"build the write lease, and do not buy the cluster for the safety rules alone"*, and it explicitly names this exact fallback:

> The gating-rule guarantee can be reproduced — imperfectly but adequately — as **a precondition check inside the orchestrator**... That is weaker (it is a client-side check, not a server-side one, and a human running the steps by hand can skip it) but it is not nothing, and it is free.

**The `Choice` state above *is* that fallback.** This note's contribution is to say plainly what it does not cover: **the 3am manual failover.** And to note the sharpest version of the counter-argument — the scenario in which the orchestrator is bypassed is *precisely* the scenario in which you most want the gate. Do not let a green `Choice` state in a diagram create a feeling of safety that only exists on the happy path.

### Where the human belongs

Unchanged from [[failover-orchestration]], and worth restating because the ASL makes it concrete: **between fence and promote.**

- Fencing is reversible. Do it automatically, on alarm, before the human is awake. It shrinks the split-brain divergence window that ruined GitHub's day.
- Promotion is a one-way door for [[aws-rds-postgres]]. Gate it on a named human.
- Everything after the approval is machine-driven and parallelised.

Three implementation details Step Functions gets right and one it gets wrong:

**Right.** `.waitForTaskToken` puts the approval *inside* the execution, so the approval and the actions are one auditable artefact — better than SSM's `aws:approve`, which cannot be used in multi-account/multi-Region automations at all.

**Right.** `TimeoutSeconds: 600` on the approval, with a `Catch` on `States.Timeout` → `AbortNoApproval`. **Timing out must mean stop, never proceed.** A `Choice` with a `Default` that continued would be the bug.

**Right.** The approval message is a *checklist*, not a yes/no. The four boolean conditions from [[failover-orchestration]] travel with the page.

> [!danger] Wrong: the approval task token is regional, and the approval path may be too
> The token in `AwaitApproval` can only be redeemed against the Region running
> the orchestrator. Everything in the approval chain must therefore avoid the
> failing Region:
>
> - The **SNS topic** must be in the orchestrator's Region, not the primary's.
> - The **approval endpoint** behind `approveUrl` — API Gateway + Lambda calling
>   `SendTaskSuccess` — must be in the orchestrator's Region.
> - Its **custom domain and ACM certificate** must exist there ([[aws-acm]]) and
>   its Route 53 record must already resolve. **Do not create DNS or
>   certificates during a failover.**
> - The **paging system** (PagerDuty, Slack) must not route through the failing
>   Region.
>
> This is the [[failover-orchestration]] dependency checklist applied to the
> approval path specifically, and it is the part people forget because the
> approval feels like a human process rather than infrastructure. **An approval
> link that 404s is an unbounded RTO.**
>
> Rehearse it: click the real link in a game day. [[dr-testing-and-gamedays]].

### Automation vs false positives

Nothing in this note changes [[failover-orchestration]]'s verdict, which rests on the GitHub October 2018 postmortem and AWS's own *"Automatically initiated failover based on health checks or alarms should be used with caution."* **Manual trigger, automated execution.**

The one thing Step Functions adds: it makes the **stage-1-only** pattern easy and cheap. An EventBridge rule on a composite alarm starts the state machine; the state machine freezes, fences, and then blocks on `AwaitApproval`. Reversible work happens automatically while the human wakes up; the one-way door stays shut. That is the highest-value hybrid available and in ASL it is free — it is just where you put the `.waitForTaskToken` state.

## Recommendation

**Do not make Step Functions the top-level failover orchestrator. Use it one level down.**

1. **Top level: ARC Region switch**, one plan per pair, $70/plan/month. Agreeing with [[route53-application-recovery-controller]] and [[failover-orchestration]]. The reason, stated in this note's terms: **Step Functions' only decisive advantage over the cheaper and more readable SSM option is native parallelism, and Region switch has that too — plus a data plane in every Region, practice mode, graceful/ungraceful modes, and automatic RTO measurement.** Step Functions is dominated, not merely beaten.
2. **Use Step Functions as a *worker*, invoked by a Region switch or SSM step**, wherever a single step needs genuine fan-out — "scale these 40 deployments", "enable these 60 event source mappings". An inline `Map` over a dynamic list is the right tool and nothing else in the stack expresses it as well. The `arn:aws:states:::aws-sdk:arcregionswitch:` integration also exists in the other direction if you want a state machine to kick off a plan.
3. **If Region switch is rejected on maturity grounds — the hybrid.** Write the sequence as an **SSM Automation document**, because it doubles as the human-readable runbook and the auditor's artefact, and have its one time-critical step invoke a **Step Functions state machine** that runs promote ∥ scale. You get SSM's readability and Step Functions' parallelism, and the Python-fan-out blob that [[failover-orchestration]] calls *"precisely the thing runbooks-as-code was supposed to avoid"* disappears. **This is the answer if you must self-host.**
4. **Build the SSM document first regardless.** It forces you to write the sequence down, it is what you evaluate Region switch *against*, and it is the fallback when the new service surprises you.
5. **Whatever you choose, deploy it to the standby *and* a third Region**, carry every input as a literal, and accept that the CA pair's third Region is a legal question rather than an engineering one — which is itself another argument for Region switch on the CA pair specifically.

> [!note] The one-line version
> **Step Functions is the right tool for orchestrating your *application*. It is
> the second-best tool for orchestrating your *recovery*, and the gap is
> entirely about where its data plane lives.**

---

## Gotchas

The list that makes this note worth reading.

1. **A mirrored state machine points at the primary.** The `Resource` field is region-less (`:::`) but the `Parameters` block carries full ARNs. `terraform plan` is clean, the graph renders, and the failover silently writes into the Region you just evacuated. **The single most likely real-world bug in this note.**
2. **Activities and legacy Lambda integrations break the "`Resource` is safe" heuristic.** Both put a real regional ARN in the `Resource` field. Grep for `"Resource": "arn:aws:lambda:` and `"Resource": "arn:aws:states:<region>` specifically.
3. **`StartExecution` name-idempotency is per account, per Region, per state machine.** Teams relying on `ExecutionAlreadyExists` for duplicate suppression lose it at failover, silently, with no error.
4. **`RedriveExecution` cannot cross Regions**, is Standard-only, and its 14-day clock runs *during your outage*. It is a failback tool.
5. **The 14-day redrive window only starts when the execution *closes*.** An execution still `RUNNING` in a dead Region is not yet redrivable, and its default `TimeoutSeconds` is ~3 years.
6. **`eu-west-2` has 800 state transitions/second where `eu-west-1` has 5,000.** A 6.25× cliff at exactly the wrong moment. `us-east-1`/`us-west-2` are symmetric; `ca-central-1`/`ca-west-1` are symmetric but symmetrically low.
7. **`StopExecution` refills at 25/s outside the big three Regions versus 200/s inside.** Sweeping 50,000 executions takes 4 minutes in Ireland and 33 in London.
8. **Quota increases are per account, per Region, and never replicate.** Every increase ever granted in the primary exists only there.
9. **"AWS raises these quotas automatically based on your usage" cannot help a warm standby**, which by design has no usage. Generalise this to every auto-tuned quota in the estate — it belongs in [[lessons-and-antipatterns]].
10. **Open executions in the recovered Region resume and produce side effects.** Standard workflow state *"internally persists between state transitions"*. A `Wait` state that expires after failback will charge a card for an order the standby already fulfilled. **Stopping them is a mandatory failback step.**
11. **You probably cannot fence Step Functions during a real disaster.** `ListExecutions` and `StopExecution` are control-plane calls against the impaired Region. Fence at the SCP/credential/security-group layer instead.
12. **`.sync` cancellation is best-effort and explicitly fails during a service outage.** AWS frames the orphaned ECS/Glue/EMR job as a billing problem; **it is a fencing problem** — live compute in the "dead" Region, invisible to any Terraform-derived inventory.
13. **Task tokens are unredeemable from another Region and cannot be reissued.** A token replicated on an SQS message to the standby is *worse* than one that was not replicated — it looks valid and is not.
14. **`HeartbeatSeconds` and `TimeoutSeconds` default to 99,999,999 seconds.** Every `.waitForTaskToken` task without an explicit timeout becomes a year-long open execution after an outage. **Lint for this in CI.**
15. **Deleting a state machine terminates its running executions at their next state transition.** A routine rename can destroy in-flight payments. Drain first.
16. **`name` and `type` are `ForceNew`, and `type` is immutable in the service too.** Standard→Express is a rebuild, not a toggle.
17. **`Activity` encryption configuration is immutable** — changing it requires a new activity, hence a new ARN, hence an ASL change.
18. **KMS: the encryption context is the state machine ARN and `kms:ViaService` is `states.<region>.amazonaws.com`.** Both are region-bearing, so a *multi-Region KMS key does not solve this* — the key policy conditions do. Pre-authorise both Regions' ARNs.
19. **CMK-encrypted executions omit Input/Output/Error/Cause from EventBridge status-change events.** Any automation keying off those fields breaks silently.
20. **A disabled or pending-deletion CMK in the standby is a total outage of that state machine**, and nothing exercises it unless you run a synthetic.
21. **Execution history lives in the Region that had the incident**, is not queryable, and `ListExecutions` refills at 2/s outside the big three. Stream logs out to a third Region.
22. **A 30-day retention reduction granted for compliance does not replicate.** The standby silently retains for 90 — a compliance failure in the permissive direction that nobody will notice.
23. **`include_execution_data = true` copies payloads into CloudWatch Logs** and then into wherever you replicate them. A data-residency decision disguised as a logging setting — acute for the CA pair.
24. **Distributed Map restarts from zero in the standby.** 10M items at a 100 TPS standard child-dispatch rate is 27+ hours of dispatch alone. Checkpoint per item or declare batch out of scope for failover.
25. **Distributed Map inputs may not have replicated.** If the manifest is on an unreplicated S3 prefix the standby cannot start at all.
26. **Cross-Region service integrations do not exist.** *"Step Functions doesn't support referencing ARNs across partitions or regions."* The CDK ships a construct that spawns a Lambda per call to work around it. A third-Region orchestrator must proxy everything.
27. **ARC routing controls cannot be flipped via the `route53recoverycluster` SDK integration**, because there is no endpoint-URL override and ARC requires one. Use a Lambda that rotates all five endpoints.
28. **A DynamoDB Global Table lock is a tripwire, not a mutex.** Conditional writes are atomic per-replica; Global Tables reconcile last-writer-wins.
29. **A `Fail` state does not un-fence the primary.** Encoding a runbook as a state machine biases attention to the happy path; the failure paths are the point.
30. **The Developer Guide and the General Reference disagree on `DescribeExecution`'s refill rate** (15/10 vs 50). Read your own account's values from the Service Quotas console rather than trusting either page.

## Decisions to make

| Decision | Option A | Option B | Recommendation |
|---|---|---|---|
| **How do we mirror definitions?** | Copy the ASL per Region, edit ARNs by hand | One `.tftpl` template + injected `integration_arns` map + a `no foreign region ARNs` precondition | **B.** A hand-edited copy will drift, and the drift is invisible until failover. The precondition is ~10 lines and catches the worst bug in the note. |
| **Standard or Express for new workflows?** | Standard everywhere (familiar, exactly-once, full history) | Express where the work is short and idempotent | **B for new short/idempotent work; keep Standard elsewhere.** Express caps DR exposure at 5 minutes by service limit. But **do not migrate existing Standard workflows** — `type` is immutable and you lose `.sync`, callbacks, Distributed Map, Activities and 90-day history. |
| **How do we make lost executions recoverable?** | Reconcile manually from external systems of record after each incident | Idempotency keys derived from the business key, in a Global Table / passed to external APIs | **B, and treat it as the real deliverable of this note.** A is unbounded manual work at 4am and only works when an external system happens to be the record. B is an application change, so start the conversation now — it is the long-pole item. |
| **Execution naming** | `uuid4()` | Deterministic from the business key (`order-{id}`) | **B.** Regional dedup plus a failback tripwire, for free. It does **not** protect across Regions — pair it with a real idempotency key. |
| **`ca-west-1` for the CA pair?** | Keep Calgary | Move to a non-Canadian standby | **Keep Calgary — Step Functions is not a reason to move.** Full endpoint set including FIPS. Feed into [[region-pair-selection]] as a pass. |
| **Standby quotas** | Leave at defaults, raise if we throttle | Request increases now, diff them on a schedule | **B.** The EU pair is 6.25× short on `StateTransition` and the auto-raise mechanism cannot help a Region with no traffic. Lead time is days-to-weeks; a failover is minutes. |
| **Do we keep state machines warm in the standby?** | Deploy at failover time | Deploy always | **B, without hesitation.** Standard bills per state transition, so an idle state machine costs **$0**. There is no financial argument for A and deploying-at-failover-time is a cold-provisioning step the 15-minute RTO forbids. |
| **Do we run synthetic executions in the standby?** | No — it is idle by definition | Hourly, end to end, per state machine | **B.** It is the only thing that proves the mirrored ARNs resolve locally, and it also exercises the standby CMK. Costs pennies. |
| **Fencing Step Functions at failover** | Sweep `StopExecution` across the primary | Do not; fence at the SCP/credential layer, sweep on failback | **B.** The sweep needs the impaired Region's control plane and costs 4 minutes of a 900-second budget for something off the critical path. |
| **Orchestrator: who runs the failover?** | Step Functions state machine, self-hosted in standby + third Region | ARC Region switch | **B**, with an SSM document built first as the readable fallback. Step Functions' one advantage (`Parallel`) is also in Region switch, which additionally has a data plane in every Region. |
| **If Region switch is rejected** | Pure SSM with a Python fan-out blob | SSM document that invokes a Step Functions state machine for the parallel step | **B — the hybrid.** SSM stays the human-readable runbook; Step Functions does the one thing SSM cannot. |
| **Ordering enforcement in the orchestrator** | ASL `Choice` gate (free, client-side) | ARC gating rule (server-side, needs a $1,825/mo cluster) | **A, knowingly.** Agrees with [[route53-application-recovery-controller]]. Record in the runbook that the `Choice` gate does **not** bind a human running the CLI by hand — which is exactly the 3am scenario. |

## Cost

Step Functions is close to free to run active/passive, which is unusual in this vault and worth saying plainly.

From the [pricing page](https://aws.amazon.com/step-functions/pricing/) (figures quoted are **US East (N. Virginia)**):

| Item | Price |
|---|---|
| Standard — state transitions | **$0.000025 per state transition** |
| Standard — free tier | **4,000 free state transitions per month** |
| Express — requests | **$1.00 per million requests** |
| Express — duration | Tiered per GB-second, billed in **64 MB** chunks |
| Idle state machine | **$0** — no per-hour or provisioned charge |

> [!note] Regional price variation was not confirmed
> The pricing page presents US East (N. Virginia) figures and **does not publish
> a differentiated table for `eu-west-1`, `eu-west-2`, `us-west-2`,
> `ca-central-1` or `ca-west-1`.** AWS commonly varies per-Region pricing, so
> **do not assume parity** — pull the real numbers from the Pricing Calculator
> or Cost Explorer per Region before putting figures in [[cost-model]]. This is
> a "no public data found at the granularity we need" finding, not a claim of
> equality.

### What the warm standby actually costs

| Item | Monthly, per pair | Note |
|---|---|---|
| Idle state machine definitions | **$0** | The important line. Nothing to pay for. |
| Hourly synthetic execution, 10 state machines × ~8 states | ~72,000 transitions/month ⇒ **≈ $1.70** (at the us-east-1 rate, less the 4,000 free) | **The best value in this note.** |
| Open-execution export Lambda, 5-min schedule, both Regions | pennies | [[aws-lambda]] |
| CloudWatch log groups (standby, near-empty) | pennies | |
| Cross-Region log shipping to a third Region | see [[observability-multi-region]] | Data transfer + storage, not a Step Functions charge |
| Activity workers running warm | **the real cost** | This is compute; count it under [[aws-eks]], not here |
| Failover orchestrator state machine, ~20 transitions per rehearsal | **≈ $0.0005 per run** | Effectively free even rehearsed weekly |

**The levers**, in order of size:

1. **Activity workers are the only meaningful standby cost**, and they are an [[aws-eks]] line item. If a state machine uses Activities purely for legacy reasons, replacing them with `lambda:invoke` removes a warm-fleet requirement entirely.
2. **State-transition count is a design choice.** Polling loops implemented as `Retry` with `BackoffRate: 1.0` bill per attempt. A 60-attempt DB-readiness poll is 60 transitions every failover rehearsal — trivial here, but the same pattern in a high-volume application workflow is not.
3. **Express is cheaper at volume and caps DR exposure** — but the workflow-type decision should be made on semantics, not on this.
4. **Distributed Map child executions bill individually.** A per-item DynamoDB checkpoint (recommended above) adds a write per item. At 10M items that is real money — weigh it against 27 hours of re-processing.

## Open questions

Things that need an answer from inside the company, not from AWS.

1. **What is the p100 open-execution age, per state machine, in each primary Region?** *The single most important number in this note.* It is one CLI call, and it determines whether Step Functions' RPO exposure is one minute or one year — i.e. whether most of Part 1 applies to you at all.
2. **Which state machines have non-idempotent tasks?** The inventory's decisive column. It determines whether "re-run it in the standby" is a button or a six-week engineering programme.
3. **What is the steady-state `StateTransition` rate in `eu-west-1`?** If it is anywhere near 800/s, the EU standby throttles from minute one. Alarm on `ExecutionThrottled` in both Regions today; it is free and it answers this in a week.
4. **Do any state machines use legacy Lambda `Resource` ARNs?** One grep. They are region-pinned in the field everyone assumes is safe, and the modern console cannot edit them graphically.
5. **Do we use Activities anywhere, and if so where do the workers run?** A worker fleet polling the primary's activity ARN is both a failover blocker and, potentially, a task-stealing bug in normal operation.
6. **Do any `.waitForTaskToken` tasks lack `HeartbeatSeconds`/`TimeoutSeconds`?** One `jq` check. Each one is a latent year-long execution.
7. **Is there any Distributed Map in production, and is batch in scope for the 15-minute RTO at all?** "Batch has a 24-hour RTO" is a legitimate and much cheaper answer — but it must be decided, not discovered.
8. **Has a 30-day execution-history retention reduction ever been requested for compliance?** If so it exists in exactly one Region and the standby is out of policy.
9. **Are CMKs used on any state machine or activity?** If yes, the KMS condition work is on the critical path and the standby key is an unexercised single point of failure.
10. **What are our actual Step Functions quota values in all six Regions?** Two AWS pages disagree on at least one. Read the Service Quotas console and record them in the runbook.
11. **Does ARC Region switch have adequate Terraform provider coverage on the version we pin?** Shared with [[failover-orchestration]] — it decides whether the Part 2 recommendation is buildable or whether the SSM+SFN hybrid is the answer.
12. **For the CA pair, is a non-Canadian third Region acceptable for an orchestrator holding identifiers and credentials but no customer data?** A legal question, shared with [[failover-orchestration]] and [[region-pair-selection]]. Note that choosing ARC Region switch makes it moot.
13. **What is the real per-Region price of a state transition?** Not published at the granularity [[cost-model]] needs.

## Sources

- [Choosing workflow type in Step Functions](https://docs.aws.amazon.com/step-functions/latest/dg/choosing-workflow-type.html) — the Standard/Express comparison table verbatim: one year vs five minutes, exactly-once vs at-least-once vs at-most-once, *"Execution state internally persists between state transitions"*, the *"Idempotency is not automatically managed"* line for Express, 90-day history and the 30-day reduction, and the rows showing Express does **not** support `.sync`, `.waitForTaskToken`, Distributed Map or Activities. **The primary source for the Standard-vs-Express DR analysis.**
- [Restarting state machine executions with redrive](https://docs.aws.amazon.com/step-functions/latest/dg/redrive-executions.html) — the full eligibility list (14 days, post-15-Nov-2023, <24,999 events, one-year open time), *"continues the failed execution from the unsuccessful step"*, *"Redriven executions use the same state machine definition and execution ARN"*, the reset of retry counts and state-machine timeout, and the per-state redrive behaviour table including Distributed Map and the `States.DataLimitExceeded` re-run case. **The source for "redrive is a failback tool".**
- [Step Functions service quotas](https://docs.aws.amazon.com/step-functions/latest/dg/service-quotas.html) — **the `StateTransition` 5,000-vs-800 split between `us-east-1`/`us-west-2`/`eu-west-1` and "all other regions"**, the `StartExecution`, `StopExecution`, `GetActivityTask`, `SendTask*`, `ListExecutions` and `RedriveExecution` differentials, *"Throttling quotas are per account, per AWS Region"*, *"New AWS accounts have reduced state transition quotas. AWS raises these quotas automatically based on your usage"*, the 25,000-event history cap, the 90-day retention and its per-account-per-Region reduction, and the Distributed Map quotas. **The source for the EU pair's 6.25× cliff.**
- [AWS General Reference — Step Functions endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/step-functions.html) — the full endpoint table confirming **`ca-west-1` has Step Functions including FIPS and sync-Express endpoints**, plus all six of this estate's Regions; the Service Quotas quota codes (`L-137B3F65`, `L-AD9B4E93`, etc.) and their adjustability; and the `DescribeExecution` refill figure that contradicts the Developer Guide.
- [Task workflow state](https://docs.aws.amazon.com/step-functions/latest/dg/state-task.html) — **the decisive line: *"Step Functions doesn't support referencing ARNs across partitions or regions."*** Plus the canonical `lambda:invoke` example with a fully-qualified region-bearing `FunctionName`, the activity `Resource` ARN syntax, the `Credentials` field, and the `TimeoutSeconds`/`HeartbeatSeconds` defaults of 99,999,999.
- [Discover service integration patterns in Step Functions](https://docs.aws.amazon.com/step-functions/latest/dg/connect-to-resource.html) — the `:::` explanation (*"inferred from the region and account in which the workflow runs"*), the legacy-Lambda-integration exception, the optimized-integration support matrix, the `.sync` abort paragraph (*"best-effort attempt to cancel"*, *"A temporary service outage occurred"*, *"you might incur additional charges"*), `.sync` polling consuming your quota, and **_"You must pass task tokens from principals within the same AWS account."_**
- [`StartExecution` API reference](https://docs.aws.amazon.com/step-functions/latest/apireference/API_StartExecution.html) — **_"this name must be unique for your AWS account, region, and state machine"_** and the idempotency paragraph (same name + same input → same response; closed or different input → `400 ExecutionAlreadyExists`; 90-day name reuse; not idempotent for Express). **The source for "the built-in dedup does not cross Regions".**
- [`DeleteStateMachine` API reference](https://docs.aws.amazon.com/step-functions/latest/apireference/API_DeleteStateMachine.html) — *"A state machine is deleted only when all its executions are completed. **On the next state transition, the state machine's executions are terminated.**"* The source for the drain-before-rename warning.
- [Learning to use AWS service SDK integrations](https://docs.aws.amazon.com/step-functions/latest/dg/supported-services-awssdk.html) — the `arn:aws:states:::aws-sdk:{serviceName}:{apiAction}` syntax, the PascalCase parameter convention, and the full service list confirming that **`arcregionswitch` and `route53recoverycluster` both have SDK integrations** (relevant to the Part 2 hybrid, and to the endpoint-override caveat).
- [Data at rest encryption in Step Functions](https://docs.aws.amazon.com/step-functions/latest/dg/encryption-at-rest.html) — the encryption context `aws:states:stateMachineArn` / `aws:states:activityArn` (both region-bearing), the `kms:ViaService: states.us-east-1.amazonaws.com` example, `DescribeKey` verifying the key *"exists in the account and region"*, the disabled/deleted-key FAQs, activity encryption immutability, and the note that CMK-encrypted executions omit Input/Output/Error/Cause from EventBridge events.
- [`CallAwsServiceCrossRegion` (AWS CDK)](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_stepfunctions_tasks.CallAwsServiceCrossRegion.html) — *"A Step Functions task to call an AWS service API across regions"* and *"This task creates a Lambda function to call cross-region AWS API and invokes it."* **First-party corroboration that cross-Region SDK integration requires a Lambda proxy.**
- [Terraform `aws_sfn_state_machine`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/sfn_state_machine) — the argument surface and the confirmation that **`name` and `type` force replacement** while `definition`, `role_arn`, logging, tracing, `publish` and `encryption_configuration` update in place.
- [AWS Step Functions pricing](https://aws.amazon.com/step-functions/pricing/) — $0.000025 per state transition and 4,000 free per month (US East, N. Virginia); $1.00 per million Express requests; duration billed per GB-second in 64 MB chunks. **Note: no per-Region table is published, so regional parity is unconfirmed.**
- [Orchestrate disaster recovery automation using Amazon Route 53 ARC and AWS Step Functions](https://aws.amazon.com/blogs/networking-and-content-delivery/orchestrate-disaster-recovery-automation-using-amazon-route-53-arc-and-aws-step-functions/) — AWS's own reference implementation, already cited in [[failover-orchestration]]. Used here for one specific point: it stores ARC cluster endpoints and control ARNs in DynamoDB global tables and calls them **from code**, corroborating that the ARC flip cannot be done with a bare SDK integration.
- [Region switch in ARC](https://docs.aws.amazon.com/r53recovery/latest/dg/region-switch.html) — *"A data plane in each AWS Region, so that you can execute your Region switch plan without taking a dependency on the Region that you're deactivating"*, and that plan steps run *"in sequence or in parallel"*. **The two lines that make Step Functions the dominated option.**

> [!note] What is not public
> **No public postmortem, case study or engineering blog was found describing a
> real company losing in-flight Step Functions executions in a Regional
> failover, or describing how they reconciled afterwards.** This is not a search
> failure — full-Region evacuations are rare, and the companies that do them
> publish about traffic steering rather than workflow reconciliation (the same
> finding [[failover-orchestration]] records for database promotion).
>
> **No AWS statement was found promising cross-Region service integrations, or
> cross-Region redrive, as a roadmap item.** Design as though neither is coming.
>
> Every timing and arithmetic figure in this note that is not a quoted AWS quota
> is derived from those quotas (e.g. "50,000 stops at 200/s ≈ 4 minutes"), not
> measured. **Measure your own.**

## Related notes

[[failover-orchestration]] · [[route53-application-recovery-controller]] · [[split-brain-and-fencing]] · [[failover-runbook-template]] · [[dr-testing-and-gamedays]] · [[observability-multi-region]] · [[messaging-in-flight-data-loss]] · [[aws-lambda]] · [[aws-sqs]] · [[aws-sns]] · [[aws-eventbridge]] · [[aws-dynamodb]] · [[dynamodb-table-naming-migration]] · [[aws-s3]] · [[aws-iam]] · [[aws-kms]] · [[kms-when-to-use-multi-region-keys]] · [[aws-eks]] · [[aws-rds-postgres]] · [[aws-ecr]] · [[aws-acm]] · [[aws-secrets-manager]] · [[aws-ssm-parameter-store]] · [[region-pair-selection]] · [[provider-aliases-vs-separate-stacks]] · [[module-patterns]] · [[aws-regional-outages]] · [[lessons-and-antipatterns]] · [[cost-model]] · [[rpo-rto-analysis]]

## Still to research

The note is substantially complete and everything above is sourced. Marked `partial` because four items were identified and not closed:

1. **The `DbInstanceIdentifier` vs `DBInstanceIdentifier` casing** in the `aws-sdk:rds:promoteReadReplica` integration. AWS documents the PascalCase convention but not this specific member's normalisation. **Test in a sandbox before it goes in a runbook** — it appears in the Part 2 ASL.
2. **Whether any mechanism exists to override the endpoint URL in an AWS SDK service integration.** None was found, and AWS's own ARC+Step Functions reference implementation uses code instead, which is strong corroboration — but "not found" is weaker than "documented as impossible". The recommendation (use a Lambda) is safe either way.
3. **Per-Region Step Functions pricing.** The pricing page publishes only US East (N. Virginia). [[cost-model]] needs real per-Region figures from the Pricing Calculator.
4. **The `DescribeExecution` quota discrepancy** between the Developer Guide (300/15, 250/10) and the General Reference (refill 50). Resolve from the Service Quotas console in the actual account — which is the answer for every quota in this note.

Not researched because out of scope for a service note, and belonging elsewhere:

- **EventBridge Scheduler and EventBridge Pipes as cross-Region execution triggers** — who calls `StartExecution` in the standby and how that is enabled at step 4 of the failover. Belongs in [[aws-eventbridge]].
- **Step Functions versions and aliases as a deployment-safety mechanism across Regions** — whether alias-pointer parity between Regions is worth enforcing, and whether a redrive pinned to *"the version associated with the original execution attempt"* interacts badly with a mid-incident deploy.

