# ADR-Light

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) for maintaining a **single append-only decision ledger** — a typed, causally-linked record of every event that changes a project's trajectory.

## Why not just ADRs?

Classic [Architecture Decision Records](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions) record *decisions*, one file each. Real projects produce more kinds of trajectory-changing events than decisions:

| Type | What it records |
|------|-----------------|
| **DEC** | A decision took effect |
| **DEF** | Work consciously deferred, with an explicit `revisit_when` condition |
| **HYP** | A hypothesis with candidate interventions, a test matrix, and results — promoted to a DEC when a winner emerges |
| **DIS** | A discovery that corrects how an existing signal should be *interpreted* |
| **INC** | An incident: defect found, root cause, blast radius (*the cause record*) |
| **OUT** | An outage: severity, duration, damage (*the impact record*) |

Entries link into a causal DAG via `triggered_by`, `supersedes`, and `resolves` edges, so "why is the system shaped this way?" is answerable by following edges — by humans or by coding agents, months later.

Everything lives in **one grep-able file**. One file means trivial diffs, no index to maintain, a single ID allocator (which prevents the ID-collision failure mode that one-file-per-record systems hit), and a single thing to point an agent at.

## Design principles

- **The ledger is the durable layer.** RFCs and plans are ephemeral deliberation — they get absorbed and deleted. Ledger entries never do.
- **INC explains, OUT quantifies.** When a defect causes service impact, both records are filed and cross-referenced, because forensics and damage accounting are separate workstreams that close at different times.
- **Work items are edges, not entries.** The ledger records events; mutable task state belongs in a tracker.
- **The format evolves in-band** via dated format-change notes — old entries are never retrofitted.

## Install

```bash
# Per-project
mkdir -p .claude/skills/adr-light && cp SKILL.md .claude/skills/adr-light/

# Or globally
mkdir -p ~/.claude/skills/adr-light && cp SKILL.md ~/.claude/skills/adr-light/
```

Then tell your agent to start a ledger, or just make a decision — the skill suggests entries at decision moments and never appends without confirmation.

## Provenance

Extracted from a production ML pipeline project where the ledger accumulated **169 decisions, 27 deferrals, hypotheses with multi-run test matrices, and paired incident/outage records over ~4 months** of human+agent development. The rules in the skill — ledger-allocated IDs, the OUT⇄INC pairing, append-only with status-line-only edits — each exist because their absence caused a real failure.
