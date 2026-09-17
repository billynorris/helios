# Helios — agent instructions

Obsidian research vault for taking a mature AWS estate to active/passive
multi-region. **Research and writing only — this repo provisions nothing.**

## Read first

`00-meta/research-brief.md` is the source of truth for the problem: region pairs,
RPO 2h / RTO 15m, the prerequisites-first strategy, and the bar every note must
clear. Do not restate it here and do not re-derive it. `00-meta/note-template.md`
is the required section shape for service notes.

`README.md` is the index and the live status table. **Update it whenever you add
a note** — it is how the next agent knows what is left.

## How to continue the work

Remaining notes are the `⬜ not started` rows in `README.md`. Work is done by
fanning out background `general-purpose` agents, roughly one per service family,
3–4 notes each.

Every agent brief must contain, near the top:

> **CRITICAL PROCESS INSTRUCTION.** Write each note to disk as soon as you have
> researched it. Research note 1 → write note 1 → research note 2 → write note 2.
> If cut off, everything already written survives. If budget runs short,
> immediately write what you have with `status: partial` in frontmatter and a
> "## Still to research" section.

This is not boilerplate. The first research wave did ~400 web searches, batched
all its writes to the end, hit a token limit and **lost twelve notes' worth of
completed research**. One note survived. Never batch writes.

Also give each agent: the two `00-meta/` paths, the exact output paths in order,
the specific sub-topics to dig into, a target of 20+ distinct searches, and the
no-invented-sources rule.

## Standards

- **Never invent a URL, quote, price, benchmark, AZ count, incident or case
  study.** "No public data found" is a legitimate finding and must be written as
  such. This vault's only value is that it is true.
- Verify anything lookup-able (prices, region service parity, AZ counts) against
  a real AWS page rather than estimating.
- Where there's a genuine fork, document **both** branches with a
  recommendation — never collapse a real trade-off, never leave it hanging.
- Every service note answers: can this meet RPO 2h / RTO 15m, what must be
  pre-provisioned, and how do we migrate a *live* single-region resource without
  downtime or a destroy/recreate in `terraform plan`.
- Obsidian `[[wikilinks]]` liberally. A link to a note that doesn't exist yet is
  fine — it marks the gap.
- `status` in frontmatter is the researcher's own claim. Nothing here has been
  reviewed by the user yet.

## Open threads worth surfacing

- **`ca-west-1` parity** — Calgary is young and may lack services the CA pair
  needs. If it fails the parity check the CA pair changes, and leaving Canada has
  data-residency consequences. Several agents were asked to verify this; collect
  their findings into `01-strategy/region-pair-selection.md`.
- **RTO definition** — 15 minutes from *incident start* and 15 minutes from
  *decision to fail over* are wildly different targets. The user should confirm
  which one they're held to. Flag it rather than assuming.

## Git

Local: `/home/ubuntu/workspace/helios`. Remote: `billynorris/helios` (private).
Commit after each batch of notes lands. Push may be blocked by the safety
classifier — if so, tell the user to run it themselves rather than retrying.
