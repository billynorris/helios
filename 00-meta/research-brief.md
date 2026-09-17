---
title: Research Brief — read this before writing any note
tags: [meta, brief]
---

# Research Brief

This is the shared context for every note in this vault. Researchers: read this
file in full before you start. Do not re-derive any of it.

## The situation

A company runs a single product, deployed independently into **three regions
today**. Each deployment is a self-contained copy of the product. There is **no
data sharing between the three existing regions** — a customer in Canada lives
entirely in the Canadian deployment.

| Today (primary) | Mirror (target standby) | Pair name used in this vault |
|---|---|---|
| `eu-west-1` (Ireland) | `eu-west-2` (London) | EU pair |
| `us-east-1` (N. Virginia) | `us-west-2` (Oregon) | US pair |
| `ca-central-1` (Montreal) | `ca-west-1` (Calgary) | CA pair |

> These pairs are a **working assumption**, not a final decision. See
> [[region-pair-selection]]. Whenever a service behaves differently for a
> specific pair — and `ca-west-1` in particular has real service gaps — call it
> out explicitly in the note rather than writing to the general case.

**"Multi-region" in this project means mirroring an existing single-region
deployment into a paired standby region.** It does not mean making the three
existing regions talk to each other. Three independent active/passive pairs, not
one global mesh.

## The target posture

- **Active / passive with a warm standby.** Not active/active. Traffic serves
  from the primary; the standby exists to be promoted.
- **RPO: 2 hours.** Up to two hours of data loss is tolerable.
- **RTO: 15 minutes.** From decision-to-fail-over to serving traffic.
- The RTO is the hard constraint and it is aggressive. 15 minutes rules out
  anything that requires provisioning from cold at failover time (EKS control
  planes, RDS restores from snapshot, ACM DNS validation waits, new NAT
  gateways). Every note must state plainly whether its service can meet 15
  minutes and what has to be pre-provisioned for it to do so.
- Note the asymmetry: RPO 2h is *loose* (async replication is fine, continuous
  sync is not required) while RTO 15m is *tight* (warm, pre-provisioned capacity
  is required). Where a service offers a cheaper/slower option that still meets
  RPO 2h, say so — that is where the cost savings are.

## The strategy being followed

**Prerequisites first.** The team is working service-by-service, making each AWS
building block multi-region *ahead of* the compute that consumes it. When EKS
and the microservices are eventually deployed into the standby region, the
secrets, queues, topics, certificates, keys, parameters and databases they
depend on are already sitting there, warm and populated.

Progress so far:
- **Secrets Manager** — done. Native cross-region replication; you declare a
  replica region on the secret and AWS mirrors it.
- **DynamoDB** — in progress, moving to Global Tables. Tables are currently named
  per-region/per-environment, which has to be reconciled with Global Tables'
  requirement that all replicas share one table name.
- **Everything else** — not started. That is what this vault is for.

## The Terraform estate

- One central repository holds all Terraform for the company.
- Heavily and maturely templated with **cookiecutter**, multiple environments.
- Considered a good, mature setup — the goal is to **evolve** it, not replace it.
- Any recommendation that amounts to "rewrite your Terraform" is a bad
  recommendation unless the note argues hard for why. Prefer changes that fit a
  templated, multi-environment monorepo.

## Services in scope

EKS, RDS Postgres, DynamoDB, ElastiCache/Redis, SQS, SNS, EventBridge, Lambda,
API Gateway, ACM, Route 53, ALB/NLB, CloudFront, S3, ECR, KMS, Secrets Manager,
SSM Parameter Store, IAM, VPC & networking, CloudWatch/observability, AWS Backup.

Plus the cross-cutting concerns: failover orchestration and runbooks, Terraform
repo structure, cost modelling, data residency/compliance.

## What a good note looks like

Every service note follows [[note-template]]. Beyond the template, the bar is:

1. **Depth over breadth.** The reader is a competent Terraform engineer. Skip
   "what is SQS". Go straight to what breaks across regions.
2. **Real sources.** Cite actual URLs — AWS docs, AWS architecture blog, re:Invent
   talks, HashiCorp docs, provider issues, engineering blogs, conference talks,
   postmortems. **Never invent a citation, a URL, a quote, or a case study.** If
   you could not find a real example, write "no public example found" — that is
   a useful finding, not a failure.
3. **Real Terraform.** Working HCL against the `hashicorp/aws` provider (v5.x/v6.x
   era), using provider aliases for the second region. Show the module signature
   you'd actually want in a templated monorepo, not a toy snippet.
4. **Named gotchas.** The things that only show up in production: replication
   lag, IAM/KMS grants that don't cross regions, ARNs embedded in policies,
   quota differences between regions, services absent from a region, resources
   that must live in `us-east-1` regardless (CloudFront certs, some global
   services), eventual consistency on control-plane operations.
5. **Decisions, not verdicts.** Where there is a genuine fork, document *both*
   branches: "Choose A and you get X, at cost Y. Choose B and you get P, at cost
   Q. Recommendation: ___ because ___." Do not collapse a real trade-off into a
   single answer, and do not stall on it either — always give a recommendation.
6. **Migration path.** The company is live in one region today. Every note must
   answer: how do you get from the current single-region resource to the
   mirrored one *without downtime and without a destroy/recreate in Terraform
   plan*. Call out anything that forces replacement (`ForceNew`) loudly.

## Style

- Obsidian-flavoured markdown. Use `[[wikilinks]]` liberally to other notes —
  a link to a note that doesn't exist yet is fine and desirable, it marks the
  gap.
- Frontmatter on every note (see template).
- Tables where comparing, prose where explaining, code fenced as ```hcl.
- Write for a reader who will read this in Obsidian six months from now and has
  forgotten the context. Be self-contained per note.
- Length: these should be substantial. A thin note is a failed note.
