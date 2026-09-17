---
title: Note Template
tags: [meta, template]
---

# Note Template

Copy this shape for every service note in `02-services/`. Sections may be
reordered or merged if the service demands it, but nothing should be silently
dropped — if a section doesn't apply, say "N/A" and why in one line.

```markdown
---
title: <AWS Service> — Multi-Region
service: <service>
tags: [service, multi-region, <service-slug>]
status: researched
replication: native | manual | none
rpo_achievable: <e.g. "seconds (async replication)" | "N/A — stateless">
rto_achievable: <e.g. "< 1 min if pre-provisioned" | "30-60 min — fails RTO">
meets_targets: yes | no | conditional
updated: 2026-09-16
---

# <AWS Service> — Multi-Region

## TL;DR
Five bullets. What the answer is, whether it meets RPO 2h / RTO 15m, and the
one thing that will bite.

## Does this service cross regions at all?
Regional vs global. Whether AWS gives you anything native. Whether the resource
is even addressable from another region.

## Replication / mirroring options
Every mechanism available, with what each actually guarantees. Include the
"do nothing, just deploy a second copy" option explicitly — for stateless
services that is usually the right answer and the note should say so.

## RPO / RTO analysis
Against the 2h / 15m targets specifically. Where does the time actually go?
What is pre-provisioned vs provisioned-at-failover?

## Warm standby shape
What exists in the standby region while the primary is healthy. What is scaled
to zero. What costs money while idle.

## Terraform implementation
Provider aliases, module structure, real HCL. How this fits a cookiecutter-
templated multi-env monorepo. Include the variable surface you'd expose.

## Migration path from single-region
Step by step, from live single-region to mirrored, without downtime. Flag
anything that forces resource replacement.

## Failover procedure
The actual steps at 3am. What is automated, what is a human decision.

## Failback
Usually harder than failover and usually forgotten. Cover it.

## Gotchas
The list that makes this note worth reading.

## Decisions to make
| Decision | Option A | Option B | Recommendation |
|---|---|---|---|

## Cost
What the standby costs while idle, roughly, and the levers.

## Open questions
Things needing an answer from inside the company.

## Sources
Real URLs only, with a line on what each contributes.
```
