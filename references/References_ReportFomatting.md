# References_ReportFormatting

**Purpose:** The single source of truth for how curious-jello formats its output. The skill decides *what* to extract and in what order; this file decides *how it's printed*. Every pass points here instead of carrying its own template.

**Principle:** deterministic and scannable. Same section always gets the same shape, same numbering scheme, same header style, run to run. No severity, no confidence, no "confirmed" status ever appears anywhere in any section below — that's a skill-level rule (see SKILL.md Rule 1), not a formatting choice, but it applies to every template here.

---

## Numbering Series

Each series is independent, sequential across the whole run (not reset per contract), and never reused across sections:

| Series | Used by |
|---|---|
| `UF-N` | User Flow entries |
| `KI-N` | Known Issues entries |
| `INV-N` | Invariant entries (consolidated set only — see Invariants template) |
| `PER-N` | Periphery & Uniswap Crawl entries |
| `INT-N` | Integrator Crawl entries |
| `LEAD-N` | RAGE MODE proven leads (RAGE MODE only — see below) |

Math Pass concepts are not numbered — they're named (`### Concept: [Math Name]`), per its own template below.

`LEAD-N` is sequential across the whole RAGE run, never reset per contract or per checklist category — same rule as every other series above. It is never reused by, or combined with, any other series; a RAGE run does not touch `UF-N`/`KI-N`/`INV-N`/`PER-N`/`INT-N` at all, since RAGE MODE produces no Docs Deep Dive, Crawl, or normal Math Pass content.

---

## Contract Header

Used at the start of the Docs Deep Dive's per-contract pass, and (in lighter form) at the start of a standalone Crawl run.

**Full form (Docs Deep Dive):**
```
═══════════════════════════════════════
  Contract.sol
═══════════════════════════════════════
  ROLE:     [one line — this contract's role in the protocol]
  HOLDS:    [one line — assets/permissions/state it owns]
  RELATES:  [one line — connection to other in-scope contracts, or "standalone"]
```

**Light form (standalone Crawl, no Docs Deep Dive preceding it):**
```
Contract.sol — [one-line role, inferred from code if no docs given]
```

---

## Docs Deep Dive Sections

### Docs Summary
```
DOCS SUMMARY
  - [bullet]
  - [bullet]
```
Relaxed mode / no docs: `DOCS SUMMARY — not available, no docs provided`

### User Flows
```
USER FLOWS
  UF-N  [action name] — [who calls what, in what order, what they receive]
```

### Known Issues
```
KNOWN ISSUES (per docs)
  KI-N  [issue as stated in docs] — source: [doc section/heading]
```
No such section in docs: `KNOWN ISSUES — none stated in docs.`
Relaxed mode / no docs: `KNOWN ISSUES — not available, no docs provided.`

### Invariants (consolidated)

This section shows the *final, deduplicated* set only — never the raw per-function candidate list that fed it (that list is internal, see SKILL.md Pass: Docs Deep Dive, Step E). A healthy consolidated list for a real codebase is usually single digits to low teens, not one entry per guard clause.

```
INVARIANTS
  INV-N  [the invariant, stated as a system-wide property that must hold]
         basis: [doc reference]  |  basis: inferred from code
         enforced: [where it's enforced — one contract/function, or a short
                    list if the same property is protected at multiple
                    sites. Omit this line if there's only one site and it's
                    already obvious from the invariant statement.]
```

---

## Periphery & Uniswap Crawl

```
IF the contract has no periphery/Uniswap surface:
    Contract.sol — not applicable

IF it does:
    PER-N  Contract.sol::function() — [what the periphery/Uniswap
           interaction does: what it calls, what it expects back, what
           hook/lifecycle point it fires at, how permissions/deltas are
           encoded]
```

---

## Integrator Crawl

```
INT-N  Caller.sol::callerFn() → Base.sol::baseFn() — [what the call does,
       what approval/callback relationship it involves (if any), what the
       base contract assumes about the caller or prior state]
```

---

## Math Pass

### Phase 1 — Per-Contract Extraction
```
## ContractName: [one-line role]

### Concept: [Math Name]
INVARIANT:   [state relationship]
EXAMPLE:     [real code snippet, line-referenced]
BOUNDARY:    [limits/overflow risk described as mechanism, e.g. "amounts
              above int128.max are rejected by this check"]
```

### Phase 3 — Consolidated Report
```markdown
## Math Pass — Consolidated Model

### Summary Table
| Concept | Used In | Boundary |
|---|---|---|
| [name] | ContractA, ContractB | [edge condition, plain language] |

### Detailed Concepts

#### [Concept Name]
INVARIANT:   [concise state relationship]
EXAMPLE:
\`\`\`solidity
[minimal real snippet]
\`\`\`
USED IN:     ContractA (line X), ContractB (line Y)
CONSTRAINTS:
  - [constraint]

TRACING QUESTIONS (investigative, not conclusive):
  - [where would this invariant need to hold across a call sequence?]
```

(Phase 2, deduplication, produces no output of its own — it's the merge step between Phase 1 and Phase 3.)

---

## RAGE MODE

**This template is the one exception to the "no severity, no confidence, no confirmed status" principle stated at the top of this file.** RAGE MODE's own rules in `SKILL.md` explain why (Rule 1 is deliberately suspended for RAGE MODE only); this file only supplies the shape. Every other template above and below this one still follows the no-severity principle without exception.

RAGE MODE does not use the Contract Header, Docs Deep Dive, Periphery/Integrator Crawl, or Math Pass templates — it has its own header line, its own body sections, and its own numbering series (`LEAD-N`, above). Nothing from those other templates is reused here, and nothing from this template leaks into them.

### Run Header
```
CURIOUS JELLO — RAGE MODE pass complete.
TRIGGER: jello rage
MODE: [STRICT|RELAXED]
  Contracts processed: N
  Checklist sections walked: [core 1–14, plus any of ERC-3643 / ERC-4626 /
                              ERC-7518 / Uniswap V4 Hooks that applied]
```

### LEADS (proven)
```
── LEADS (proven) ─────────────────────────

LEAD-N  Contract.sol::function() — [checklist item, in a few words]
        PROOF:   [the exact code path/lines that make this true]
        TRIGGER: [the exact condition that exercises it]
```
Nothing beyond `PROOF` and `TRIGGER` per entry — no restated checklist prose, no framing sentence, no hedging language (`might`, `could potentially`, `appears to`). If a lead is written here, it is because the proof is closed; see `SKILL.md`'s Proof Bar for what "closed" requires before an entry ever reaches this template.

Multiple call sites of the same proven pattern are one `LEAD-N` entry listing every location, not one entry per site.

If there are no proven leads in the run: print the header and one line — `LEADS (proven) — none.` — rather than omitting the section.

### SUSPECTED, NOT PROVEN
```
── SUSPECTED, NOT PROVEN ──────────────────
[Contract.sol — checklist item, in a few words] — [why it couldn't be
 proven, in a few words, e.g. "depends on an out-of-scope library call"]
```
One line per item. No elaboration beyond the reason it couldn't be closed out — this is a summary list, not a second tier of leads. If nothing falls here: print the header and `SUSPECTED, NOT PROVEN — none.` rather than omitting the section.

### SECTIONS WITH NO MATCH
```
── SECTIONS WITH NO MATCH ─────────────────
[Category name] — walked, nothing found in [contract list]
```
One line per checklist category (core 1–14, plus any applicable token-standard/hooks section) that was walked in full and produced neither a `LEAD-N` nor a `SUSPECTED, NOT PROVEN` entry. Every applicable category appears here or in one of the two sections above — never silently absent, per the Depth Mandate in `SKILL.md`.

### NOT APPLICABLE
```
── NOT APPLICABLE ─────────────────────────
[ERC-4626 / ERC-3643 / ERC-7518 / Uniswap V4 Hooks section] — [why it
 doesn't apply to this codebase, in a few words]
```
Only the token-standard/hooks sections of `References_PatternMatch.md` can land here — the core 14 categories are always applicable in some form and always resolve into one of the three sections above, never this one.

---

## Final Output Envelope

**This envelope covers the four normal scopes (DOCS/MATH/CRAWL/FULL) only.** RAGE MODE has its own run header and its own set of sections — see the RAGE MODE template above — and does not use the envelope below. If the RAGE trigger fired, skip this section entirely and assemble the response from the RAGE MODE template instead.

Every normal-scope run opens with this header:

```
CURIOUS JELLO — [scope] pass complete.
SCOPE: [DOCS|MATH|CRAWL|FULL]
MODE: [STRICT|RELAXED|n/a]
  Contracts processed: N
```

Then, only the section(s) belonging to the active scope:

**SCOPE: DOCS**
```
── DOCS DEEP DIVE ───────────────────────
[Contract headers]
[DOCS SUMMARY]
[USER FLOWS]
[KNOWN ISSUES]
[INVARIANTS — consolidated set only]
```

**SCOPE: MATH**
```
── MATH PASS ─────────────────────────────
[Consolidated Model: Summary Table + Detailed Concepts]
```

**SCOPE: CRAWL**
```
── PERIPHERY & UNISWAP CRAWL ────────────
[PER-N entries or "not applicable"]

── INTEGRATOR CRAWL ─────────────────────
[INT-N entries]
```

**SCOPE: FULL** — all four sections, in this order, never merged into each other:
1. Docs Deep Dive
2. Periphery & Uniswap Crawl
3. Integrator Crawl
4. Math Pass

**RAGE MODE** is not a row in this envelope and never appears alongside DOCS/MATH/CRAWL/FULL content in the same response — see the RAGE MODE template above for its self-contained header and section order (Run Header → LEADS (proven) → SUSPECTED, NOT PROVEN → SECTIONS WITH NO MATCH → NOT APPLICABLE, always in that order).

---

## Formatting Rules That Apply Everywhere

- No severity, confidence, or "confirmed" labels in any entry, in any section — **except the RAGE MODE template above, which is the one deliberate exception to this rule, scoped exactly as `SKILL.md`'s RAGE MODE section describes.** Every other template in this file follows this rule without exception.
- No section is ever collapsed into another — each keeps its own header and its own numbering series.
- Sections with nothing to report still print their header line with an explicit empty-state message (e.g. `KNOWN ISSUES — none stated in docs.`, `LEADS (proven) — none.`) rather than being omitted silently.
- No mind-map, call-graph sketch, or per-function count is ever printed — those stay internal regardless of scope, RAGE MODE included.
- RAGE MODE output never merges into or shares a response with a normal-scope section, and a normal-scope response never borrows RAGE MODE's sections (`LEADS`, `SUSPECTED, NOT PROVEN`, etc.) — the two are always presented as fully separate runs, per `SKILL.md`'s RAGE MODE boundary rules.
