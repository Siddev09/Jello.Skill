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

Math Pass concepts are not numbered — they're named (`### Concept: [Math Name]`), per its own template below.

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

## Final Output Envelope

Every run opens with this header, regardless of scope:

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

---

## Formatting Rules That Apply Everywhere

- No severity, confidence, or "confirmed" labels in any entry, in any section.
- No section is ever collapsed into another — each keeps its own header and its own numbering series.
- Sections with nothing to report still print their header line with an explicit empty-state message (e.g. `KNOWN ISSUES — none stated in docs.`) rather than being omitted silently.
- No mind-map, call-graph sketch, or per-function count is ever printed — those stay internal regardless of scope.
