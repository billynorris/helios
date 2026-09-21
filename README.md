---
title: Helios — Index
tags: [moc, index]
updated: 2026-09-20
---

# Helios

Research vault for taking a mature, single-region-per-deployment AWS estate to
**active/passive multi-region**, one AWS service at a time.

Open this folder as an Obsidian vault. Start at [[research-brief]], then follow
the status tables below. Agents continuing this work: read [[CLAUDE]] first.

## The shape of the problem

Three product deployments today, each self-contained, no data shared between
them. Each gets a paired standby region it can be promoted into.

| Primary | Standby | Pair |
|---|---|---|
| `eu-west-1` Ireland | `eu-west-2` London | EU |
| `us-east-1` N. Virginia | `us-west-2` Oregon | US |
| `ca-central-1` Montreal | `ca-west-1` Calgary | CA |

**RPO 2 hours. RTO 15 minutes.** The asymmetry is the whole design: RPO 2h is
loose enough that async replication — and in places even scheduled cross-region
backup copy — will do, while RTO 15m is tight enough that nothing may be
provisioned at failover time. Warm standby, not pilot light, not active/active.

Strategy is **prerequisites first** — mirror the building blocks ahead of the
compute that consumes them, so that when EKS and the microservices land in the
standby, everything they depend on is already there and populated.

## Read these first

Of what's written so far, these four carry the most leverage:

- [[aws-kms]] and [[kms-when-to-use-multi-region-keys]] — nearly every other
  service's cross-region story bottlenecks on key placement, and there's a
  ForceNew migration trap on existing keys.
- [[provider-aliases-vs-separate-stacks]] — the decision that determines how
  painful the next year is.
- [[aws-route53]] — where the 15-minute RTO is actually won or lost.
- [[region-pair-selection]] — whether the CA pair survives contact with
  `ca-west-1`'s service parity.
- [[aws-cognito]] — if Cognito is in the estate, this is plausibly the hardest
  blocker in the programme, and it is not fixable with Terraform.

## Status

⚠️ **Nothing here has been reviewed.** `status` in each note's frontmatter is the
researcher's own claim, not a verification. Spot-check before acting, starting
with [[aws-kms]] since so much leans on it.

### Services — `02-services/`

**Edge, DNS, delivery**

| Note | Status |
|---|---|
| [[aws-acm]] | ✅ |
| [[aws-route53]] | ✅ |
| [[aws-alb-nlb]] | ✅ |
| [[aws-cloudfront]] | 🟡 **partial** — TL;DR only; see its "Still to research" |
| [[aws-api-gateway]] | ✅ |
| [[aws-global-accelerator]] | ⬜ — should pick the winner vs Route 53 / origin groups |
| [[aws-waf-shield]] | ⬜ — referenced by several notes, never written |

**Data stores**

| Note | Status |
|---|---|
| [[aws-rds-postgres]] | ✅ |
| [[aws-aurora-global-database]] | ✅ |
| [[rds-vs-aurora-decision]] | ✅ |
| [[aws-dynamodb]] | ✅ |
| [[dynamodb-table-naming-migration]] | ✅ — unblocks work in progress |
| [[aws-elasticache-redis]] | ✅ |
| [[redis-self-managed-vs-elasticache]] | ⬜ — **container vs managed, explicitly wanted** |
| [[aws-memorydb]] | ⬜ |
| [[aws-opensearch]] | ✅ |

**Compute and delivery**

| Note | Status |
|---|---|
| [[aws-eks]] | ✅ |
| [[eks-workload-delivery]] | ✅ |
| [[eks-stateful-workloads]] | ⬜ |
| [[aws-ecr]] | 🟡 partial |
| [[aws-lambda]] | ✅ |
| [[aws-step-functions]] | ⬜ |

**Messaging and streaming**

| Note | Status |
|---|---|
| [[aws-sqs]] | ✅ |
| [[messaging-in-flight-data-loss]] | ✅ |
| [[aws-eventbridge]] | ✅ |
| [[aws-sns]] | ⬜ |
| [[aws-msk-kafka]] | ⬜ |
| [[aws-kinesis]] | ⬜ |

**Security, identity, config**

| Note | Status |
|---|---|
| [[aws-kms]] | ✅ |
| [[kms-when-to-use-multi-region-keys]] | ✅ |
| [[aws-secrets-manager]] | ✅ |
| [[aws-ssm-parameter-store]] | ✅ |
| [[aws-iam]] | ✅ |
| [[aws-cognito]] | ✅ — **likely the programme's hardest blocker, read it** |

**Storage, networking, platform**

| Note | Status |
|---|---|
| [[aws-s3]] | 🟡 partial — missing Batch Replication, the live-migration path |
| [[aws-efs-ebs]] | ⬜ |
| [[aws-backup]] | ⬜ |
| [[aws-vpc-networking]] | ✅ |
| [[cross-region-connectivity]] | ⬜ |
| [[observability-multi-region]] | ⬜ — **lost in a cut-off wave, re-research** |

**AI, email, and non-AWS**

| Note | Status |
|---|---|
| [[aws-bedrock]] | ⬜ |
| [[aws-ses]] | ⬜ |
| [[aws-managed-grafana]] | 🟡 partial — **AMG does not exist in `ca-west-1`** |
| [[observability-vendors-multi-region]] | ⬜ — Datadog / Grafana / Prometheus |
| [[third-party-saas-dependencies]] | ⬜ — the shared-fate audit |

### Strategy — `01-strategy/`

| Note | Status |
|---|---|
| [[region-pair-selection]] | ✅ |
| [[rpo-rto-analysis]] | ⬜ |
| [[sequencing-roadmap]] | ⬜ |
| [[open-decisions]] | ⬜ |

### Terraform — `03-terraform/`

| Note | Status |
|---|---|
| [[provider-aliases-vs-separate-stacks]] | ✅ |
| [[state-management]] | ✅ |
| [[module-patterns]] | ✅ |
| [[terraform-delivery-and-state-platform]] | ⬜ — can you even `apply` during a failover? |
| [[terraform-gotchas]] | ⬜ |
| [[repo-structure]] | ⬜ |

### Operations — `04-operations/`

| Note | Status |
|---|---|
| [[failover-orchestration]] | ✅ |
| [[split-brain-and-fencing]] | ✅ — includes failback |
| [[route53-application-recovery-controller]] | 🟡 partial — **verdict: don't buy the £1.8k/mo cluster** |
| [[dr-testing-and-gamedays]] | ⬜ |
| [[failover-runbook-template]] | ⬜ |

### Cost, compliance, case studies

| Note | Status |
|---|---|
| [[cost-model]] | ⬜ |
| [[cost-levers]] | ⬜ |
| [[data-residency]] | ⬜ — Ireland→London adequacy is live legal risk |
| [[regulatory-drivers]] | ⬜ |
| [[security-posture-of-the-standby]] | ⬜ |
| [[aws-regional-outages]] | ✅ |
| [[lessons-and-antipatterns]] | ✅ |

`05-cost/` and `06-compliance/` are still **empty directories**. Nothing in this
vault currently costs the programme anything or tells it what it is legally
allowed to do.
| [[07-case-studies/index]] | ⬜ |

## Open threads

Two questions the vault can't answer for you:

1. **`ca-west-1` parity — the evidence is now stacking up against it.** Calgary
   is young. [[region-pair-selection]] collects the per-service findings, and as
   of 2026-09-20 two more services have failed the check: [[aws-cognito]]
   (not a multi-region-replication Region, so the CA pair has *no* Cognito DR
   path at all) and [[aws-opensearch]] (cross-cluster replication blocked by
   opt-in-Region rules). Every wave of research makes the CA pair look worse.
   If it fails, the pair changes — and leaving Canada has data-residency
   consequences that may be a hard blocker rather than a cost question.
2. **Which RTO are you held to?** 15 minutes from *incident start* and 15
   minutes from *decision to fail over* are wildly different targets.
   [[failover-orchestration]] decomposes the budget. Settle this with
   stakeholders before designing against it.

## Progress in the real estate (outside this vault)

- **Secrets Manager** — done. Native `replica` blocks.
- **DynamoDB** — in progress, moving to Global Tables. Was blocked on per-region
  table naming; see [[dynamodb-table-naming-migration]].
- **Everything else** — not started.
