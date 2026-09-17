---
title: Helios — Index
tags: [moc, index]
updated: 2026-09-17
---

# Helios

Research vault for taking a mature, single-region-per-deployment AWS estate to
**active/passive multi-region**, one AWS service at a time.

Open this folder as an Obsidian vault. Start at [[research-brief]], then follow
the status table below.

## The shape of the problem

Three product deployments today, each self-contained, no data shared between
them. Each gets a paired standby region it can be promoted into.

| Primary | Standby | Pair |
|---|---|---|
| `eu-west-1` Ireland | `eu-west-2` London | EU |
| `us-east-1` N. Virginia | `us-west-2` Oregon | US |
| `ca-central-1` Montreal | `ca-west-1` Calgary | CA |

**RPO 2 hours. RTO 15 minutes.** The asymmetry is the whole design: RPO 2h is
loose enough that async replication and even scheduled cross-region backup copy
will do, while RTO 15m is tight enough that nothing may be provisioned at
failover time. Warm standby, not pilot light, not active/active.

Strategy is **prerequisites first** — mirror the building blocks ahead of the
compute that consumes them, so that when EKS and the microservices land in the
standby, everything they depend on is already there and populated.

## Ground rules for this vault

Set out in full in [[research-brief]]. The short version:

- Every note answers: *can this meet RPO 2h / RTO 15m, and what must be
  pre-provisioned for it to?*
- Every note answers: *how do we get there from a live single-region resource
  without downtime and without a destroy/recreate in `terraform plan`?*
- Real sources only. No invented URLs, quotes, prices or case studies. "No
  public data found" is a legitimate finding.
- Where there's a genuine fork, **both** branches get documented with a
  recommendation — not collapsed into one answer, not left hanging.

## Status

Nothing here is reviewed yet. `status` in each note's frontmatter is the
researcher's own claim, not a verification.

### Services — [[02-services]]

| Note | Status |
|---|---|
| [[aws-acm]] | ✅ written |
| [[aws-route53]] | ⬜ not started |
| [[aws-cloudfront]] | ⬜ not started |
| [[aws-alb-nlb]] | ⬜ not started |
| [[aws-rds-postgres]] | ⬜ not started |
| [[aws-aurora-global-database]] | ⬜ not started |
| [[rds-vs-aurora-decision]] | ⬜ not started |
| [[aws-dynamodb]] | ⬜ not started |
| [[dynamodb-table-naming-migration]] | ⬜ not started — live blocker |
| [[aws-elasticache-redis]] | ⬜ not started |
| [[aws-eks]] | ⬜ not started |
| [[eks-workload-delivery]] | ⬜ not started |
| [[eks-stateful-workloads]] | ⬜ not started |
| [[aws-sqs]] | ⬜ not started |
| [[aws-sns]] | ⬜ not started |
| [[aws-eventbridge]] | ⬜ not started |
| [[messaging-in-flight-data-loss]] | ⬜ not started |
| [[aws-kms]] | ⬜ not started — keystone note |
| [[kms-when-to-use-multi-region-keys]] | ⬜ not started |
| [[aws-secrets-manager]] | ⬜ not started — already done in prod, validate |
| [[aws-ssm-parameter-store]] | ⬜ not started — no native replication, real gap |
| [[aws-api-gateway]] | ⬜ not started |
| [[aws-lambda]] | ⬜ not started |
| [[aws-s3]] | ⬜ not started |
| [[aws-ecr]] | ⬜ not started |
| [[aws-efs-ebs]] | ⬜ not started |
| [[aws-backup]] | ⬜ not started |
| [[aws-vpc-networking]] | ⬜ not started — CIDR planning gates everything |
| [[aws-iam]] | ⬜ not started |
| [[cross-region-connectivity]] | ⬜ not started |
| [[observability-multi-region]] | ⬜ not started |

### Strategy — [[01-strategy]]

| Note | Status |
|---|---|
| [[region-pair-selection]] | ⬜ not started — `ca-west-1` parity is the open risk |
| [[rpo-rto-analysis]] | ⬜ not started |
| [[sequencing-roadmap]] | ⬜ not started |
| [[open-decisions]] | ⬜ not started |

### Terraform — [[03-terraform]]

| Note | Status |
|---|---|
| [[repo-structure]] | ⬜ not started |
| [[provider-aliases-vs-separate-stacks]] | ⬜ not started — the year-defining decision |
| [[state-management]] | ⬜ not started |
| [[module-patterns]] | ⬜ not started |
| [[terraform-gotchas]] | ⬜ not started |

### Operations — [[04-operations]]

| Note | Status |
|---|---|
| [[failover-orchestration]] | ⬜ not started |
| [[failover-runbook-template]] | ⬜ not started |
| [[dr-testing-and-gamedays]] | ⬜ not started |
| [[failback]] | ⬜ not started |
| [[split-brain-and-fencing]] | ⬜ not started |

### Cost, compliance, case studies

| Note | Status |
|---|---|
| [[cost-model]] | ⬜ not started |
| [[cost-levers]] | ⬜ not started |
| [[data-residency]] | ⬜ not started — Ireland→London adequacy is live legal risk |
| [[regulatory-drivers]] | ⬜ not started |
| [[security-posture-of-the-standby]] | ⬜ not started |
| [[07-case-studies/index]] | ⬜ not started |

## Progress in the real estate (outside this vault)

- **Secrets Manager** — done. Native `replica` blocks.
- **DynamoDB** — in progress, moving to Global Tables. Blocked on per-region
  table naming; see [[dynamodb-table-naming-migration]].
- **Everything else** — not started.
