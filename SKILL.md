---
name: adr-light
description: Maintain a single append-only project decision ledger with typed entries (decisions, deferrals, hypotheses, discoveries, incidents, outages, human corrections) linked into a causal DAG. Use when the user makes an architectural or policy decision, defers work with a revisit condition, starts a systematic experiment, discovers a misinterpreted signal, investigates a defect or outage, corrects an AI over-claim, resolves an RFC, says "log this decision" / "ledger this", or asks "why did we choose X" / "what happened with Y".
---

# ADR-Light: The Decision Ledger

One append-only markdown file that records everything that *happened* on a project: decisions, deferrals, hypotheses, discoveries, incidents, outages, and human corrections — each typed, numbered, and linked into a causal DAG.

This is **not** classic ADRs. ADRs record decisions, one file each. A ledger records *events of every kind that change the project's trajectory*, in one grep-able, diff-able file, with provenance edges (`triggered_by`, `supersedes`, `resolves`) that let you trace *why* the project is shaped the way it is — months later, across hundreds of entries, by humans and agents alike.

**Core principle: the ledger is the durable layer.** Plans, RFCs, and working docs are ephemeral deliberation — they get rewritten, absorbed, and deleted. Ledger entries are never deleted. When an absorbed RFC is removed, its DEC entries remain; `docs_updated` lines pointing at deleted files are acceptable historical audit trail.

## Ledger Location

Default: `docs/decision_log.md`. If the project keeps agent-facing docs elsewhere (e.g. `claudedocs/`), put it there and index it from the project's CLAUDE.md / README with: *"Log every architectural or policy decision in the decision ledger."*

## Initialization

If no ledger exists and a decision moment occurs, ask the user before creating one. Seed it with:

```markdown
# <Project> Decision Ledger (ADR-light)

Append-only ledger for decisions, deferrals, hypotheses, discoveries, incidents, outages, and human corrections.

---

## Format

Entry types: DEC (decision), DEF (deferral), HYP (hypothesis), DIS (discovery),
INC (incident), OUT (outage), BOT (human correction). Each type has its own number space (DEC-001, DEF-001, ...).
IDs are allocated by appending an entry (or stub) to this file — nowhere else.

Format changes are recorded as dated blockquote notes in this section, never by
rewriting old entries.

---

## Outages

## Hypotheses

## Discoveries

## Incidents

## Decisions

## Human Corrections
```

## Entry Types

| Type | Records | Trigger | Status lifecycle |
|------|---------|---------|------------------|
| **DEC** | A decision took effect (architecture, policy, process, schema) | A choice is made; an RFC is resolved; an incident is remediated | `accepted` → `superseded by DEC-###` |
| **DEF** | A *conscious non-decision* — work judged worthwhile but deferred, with an explicit re-entry condition | "Not now, but revisit when..." | `active` → `resolved by DEC-###` |
| **HYP** | An open question + candidate interventions + test matrix + results | Systematic experimentation begins (tuning, A/B, performance work) | `open` → promoted to a DEC when a winner is chosen |
| **DIS** | An **interpretation correction** — an observation that changes how an existing signal/metric/behavior should be *read*, with no action attached yet | Live observation contradicts assumed semantics | `observed` (terminal; often later cited as `triggered_by` of a DEC) |
| **INC** | The **cause record** — a defect or design flaw found during live investigation: symptom, root cause, blast radius | Investigation finds the *why* behind observed behavior | `open` → remediated via DEC |
| **OUT** | The **impact record** — a service-affecting event: severity, duration, damage quantification | Service impact observed | `open` (may be a *live document*) → closed at recovery |
| **BOT** | A **human correction** — human intuition caught and overrode an AI over-claim or mistake ("caught the bots"); the human-in-the-loop catch recorded as a first-class event | A person pushes back on / corrects something the AI(s) asserted | `logged` (terminal; often later cited as `triggered_by` of a HYP or DEC) |

**RFC** is part of the system but is *not* a ledger entry type — see [RFCs](#rfcs-the-deliberation-layer).

## Choosing the Entry Type

```
Something happened. What kind of thing?

├─ We chose a direction / changed policy / locked a design     → DEC
├─ We chose NOT to do something, with a condition to revisit   → DEF
│    (no revisit condition? it's just scope — don't log it)
├─ We have a question + candidate fixes to test systematically → HYP
├─ We learned a signal doesn't mean what we thought it meant,
│  but nothing needs to change yet                             → DIS
├─ A human caught/overrode an AI over-claim or mistake         → BOT
├─ We found a defect / design flaw explaining behavior         → INC
├─ Service was impacted (downtime, data loss, degradation)     → OUT
│    └─ Was it caused by a discoverable defect? File the
│       OUT⇄INC pair (see pairing rule below)
├─ We discovered additional blocking WORK                      → not a ledger
│    entry. Work items live in the project's work tracker;
│    the spawning entry cites them in `impact:` (see Rule 6)
└─ The change is too large to decide inline                    → open an RFC
     doc; the eventual resolution lands as DEC(s)
```

Don't log trivia: variable naming, formatting, routine refactors. The bar is *"would someone six months from now need to know why?"*

## Shared Fields

Every entry carries:

- `id`: stable identifier (`DEC-###`, `DEF-###`, ...)
- `date`: YYYY-MM-DD
- `status`: per the type's lifecycle
- `triggered_by` *(optional but strongly encouraged)*: what caused this entry — another entry ID, `"research: <doc link>"`, or `"live observation: <what>"`
- `docs_updated`: which files were edited alongside this entry
- `related` *(optional)*: non-causal cross-references (entry IDs, file paths, plan sections)

`supersedes` (DEC only): which prior DEC this replaces. Mark the old entry's status `superseded by DEC-###` — this status edit is the **only** permitted modification to a past entry.

## Per-Type Templates

### DEC

```markdown
### DEC-042: <imperative title — what changed>
- `id`: DEC-042
- `date`: 2026-06-05
- `status`: accepted
- `supersedes`: DEC-017 (optional)
- `triggered_by`: INC-003 investigation / "research: docs/rfc_caching.md" / live observation
- `decision`: What changed. Specific enough to act on; multiple sub-decisions as a list.
- `rationale`: Why. Include rejected alternatives when they were seriously considered.
- `impact`: Code/docs affected, work items spawned, blast radius.
- `docs_updated`: paths edited alongside this entry
- `related`: DEC-040 (parent), DEF-002 (resolves), src/cache/manager.py
```

### DEF

```markdown
### DEF-003: Defer <the work>
- `id`: DEF-003
- `date`: 2026-06-05
- `status`: active
- `triggered_by`: what surfaced the work and why it's justified-but-not-now
- `decision`: What is deferred, and what proceeds in the meantime.
- `rationale`: Why deferring is safe; what the interim costs are.
- `impact`: What stays manual/slow/scan-based until resolved.
- `revisit_when`: The explicit re-entry condition. This field defines a DEF —
  no condition, no DEF.
```

### HYP

```markdown
### HYP-002
- `id`: HYP-002
- `date`: 2026-06-05
- `status`: open
- `triggered_by`: the observation that raised the question
- `question`: The open question, stated falsifiably.
- `observation`: Current evidence and constraints.

#### Candidate interventions
| ID | Intervention | Mechanism | Predicted effect | Risk |
|----|-------------|-----------|------------------|------|
| H2a | ... | ... | ... | ... |

#### Test matrix
| Run | Change from previous | Cumulative config |
|-----|---------------------|-------------------|
| 1 | H2a | baseline + H2a |

#### Results
(updated as runs complete — appending results to an open HYP is allowed;
that's the entry doing its job, not editing history)
```

Promote to a DEC when a winner is chosen: the DEC's `triggered_by` cites the HYP; the HYP's status becomes `promoted to DEC-###`. Rejected interventions stay in the HYP with one-line reasons — they're the "alternatives considered" record.

### DIS

```markdown
### DIS-002: <what the signal actually means>
- `id`: DIS-002
- `date`: 2026-06-05
- `status`: observed
- `triggered_by`: the live observation
- `finding`: What was observed and what it reveals about how the
  signal/metric/behavior actually works.
- `implication`: How to read this signal from now on; what NOT to conclude
  from it; what to correlate it with before acting.
- `docs_updated`: ...
```

### INC

```markdown
### INC-005: <defect summary — symptom and ceiling/effect>
- `id`: INC-005
- `date`: 2026-06-05
- `status`: open
- `triggered_by`: the investigation that found it
- `symptom`: Observable behavior, with numbers.
- `root_cause`: The defect, precisely located (file:line where applicable).
- `blast_radius`: What this constrains or breaks downstream.
- `why_not_caught_earlier`: Honest answer. This field earns its keep —
  it's where process improvements come from.
- `planned_remediation`: Options under consideration, or the DEC that fixes it.
- `docs_updated`: ...
```

### OUT

```markdown
### OUT-003
- `id`: OUT-003
- `date`: 2026-06-05
- `severity`: Low | Medium | High
- `summary`: One line — what was impacted, for how long, what was lost.
- `remediation`: What restored service (or link to the remediation DEC).
- `detail`: link to the full incident report (timeline, forensics) in the
  project's outage/incident directory — details live there, not here.
```

### BOT

```markdown
### BOT-001: <the over-claim or mistake the human caught>
- `id`: BOT-001
- `date`: 2026-06-05
- `status`: logged
- `triggered_by`: the claim/analysis the correction lands on (entry ID, or
  "live observation: <what the AI asserted>")
- `claim`: What the AI(s) asserted — the over-claim or error, stated plainly.
- `correction`: The human's intervention — what they pushed back on, and why.
- `verified`: What the data/analysis showed afterward (did the correction hold?).
- `lesson`: The failure mode to watch for next time.
```

BOT is terminal (`logged`) like DIS: it records the catch, it carries no action
itself. A BOT commonly reappears as the `triggered_by` of the HYP or DEC the
correction sets in motion (e.g. a pushback that forces a re-test or a reversal).

## The Relationship DAG

```
RFC (working doc, OPEN)
   │ resolved by
   ▼
DEC ──supersedes──► DEC (older)
 │ ▲ ▲ ▲                              ┌──────────────┐
 │ │ │ └──promotes──── HYP            │  OUT (impact)│
 │ │ └────resolves──── DEF            │      ⇄       │  paired when a defect
 │ └────triggered_by── DIS ───────────┤  INC (cause) │  causes service impact;
 │                     INC ───────────┤              │  a DEC files the pair
 │ spawns (impact:)                   └──────────────┘
 ▼
work items (external tracker — cited as edges, never ledger entries)
```

## Rules

1. **Append-only.** Never delete or rewrite a past entry. Corrections are new entries; a wrong DEC is superseded, not edited. The only permitted touch to an old entry is updating its `status` line (superseded / resolved / promoted) and appending results to an open HYP/OUT.

2. **The ledger allocates all IDs.** Every ID — including INC/OUT whose full report lives in an external file — is allocated by appending an entry or stub to the ledger. Filenames are not an ID registry. *(Field lesson: the originating project let INC numbers drift to external filenames after INC-001; two different incidents both became "INC-004." A stub in the ledger costs four lines and makes collisions impossible.)*

3. **OUT⇄INC pairing — INC explains, OUT quantifies.** When a defect causes service impact, file both, cross-referenced, ideally with matching slugs (`OUT-003_redis_data_loss` ⇄ `INC-004_redis_data_loss`). They are separate workstreams that close at different times: forensics (INC) often closes before damage accounting (OUT). An OUT may be a *live document*, updated in place until close-out. INC stands alone for latent defects with no service impact; OUT stands alone when the cause is trivial and fixed in the remediation line. For major events, the act of filing the pair is itself a DEC ("File INC-### / OUT-### — <classification and recovery constraints>"), which anchors the pair in the DAG.

4. **RFC lifecycle.** Changes too large to decide inline get an RFC: a separate working doc (`docs/rfc_<topic>.md` or the project's plans directory), status OPEN. Review happens in the doc. Resolution lands as one or more DECs whose `triggered_by` cites the RFC; the doc's status flips to DECIDED. Once the work ships and the doc is fully absorbed, **deleting the RFC is healthy** — the DECs and the commits are the record. Number RFCs in their own space (RFC-001...) and allocate those IDs in the ledger via the resolving DEC.

5. **Format evolves in-band.** Changing the entry format (new field, new type) is recorded as a dated blockquote note in the ledger's Format section — e.g. *"Format change 2026-02-12: added `supersedes` and `triggered_by`. Entries DEC-001–016 predate this and are not backfilled."* Old entries are never retrofitted.

6. **Work items are edges, not entries.** When an entry's consequence is *new work* (follow-ups, blocking discoveries during implementation), the entry cites the work item in `impact:` (`impact: code removal = TICKET-41`) and the item lives in the project's work tracker — issues, a followups doc, anything with mutable state. The ledger records events; it must never hold open/closed task state. Boundary test: *"we chose not to do X, revisit when Y"* → DEF (ledger). *"We must do X before Z ships"* → work item (tracker).

7. **Doc-sync.** Every entry lists `docs_updated`. If the decision changes paths, interfaces, schemas, or policy described in other project docs, those docs are edited in the same change set — the ledger entry is the receipt. Inverted: any substantive edit to architecture/policy docs should be traceable to a ledger entry.

8. **The ledger outlives its references.** `docs_updated` and `related` lines pointing at since-deleted files are correct historical record, not rot. Never "clean up" old entries to fix dead links.

## Writing Entries: Detection Signals

Suggest an entry (never auto-append without confirmation) when:

- A direction is chosen after weighing alternatives — DEC
- The user says "let's not do that now" with a condition attached — DEF
- A tuning/experiment loop begins ("let's try X, then Y") — HYP
- Debugging reveals a metric/signal means something different than assumed — DIS
- A root cause is found — INC
- Service impact is observed or reported — OUT
- A human corrects/overrides an AI over-claim or mistake ("caught the bots") — BOT
- The user says "log this decision", "ledger this", "record why"

When the user asks **"why did we choose X?"** or **"what happened with Y?"** — grep the ledger first (`grep -n "X" decision_log.md`), follow `triggered_by`/`supersedes`/`related` edges, and answer with entry IDs cited.

## Anti-Patterns

- **Essay entries.** `decision` and `rationale` should be tight. Long-form deliberation belongs in an RFC; the ledger gets the verdict.
- **Decision laundering.** Making a significant change in code and never filing the DEC. If a reviewer would ask "why?", it needs an entry.
- **Status-less drift.** A DEF with no `revisit_when`, a HYP never promoted or closed, an OUT left "live" after recovery. Sweep statuses periodically.
- **Backfilling silently.** Recording a past decision is fine — date it with the original date and note it's retroactive (`OUT-000a (retroactive)`).
- **Treating the ledger as documentation.** It records *what changed and why*, not *how things work*. Architecture docs describe the present; the ledger explains how the present came to be.
