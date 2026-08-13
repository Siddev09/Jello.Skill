---
name: curious-jello
description: understanding generator for smart contract review. Single agent, four sequential passes over every in-scope contract — (1) docs + invariant extraction, (2) targeted periphery/Uniswap crawl, (3) integrator crawl (how external contracts connect into the base contract), (4) math pass explaining complex arithmetic/logic. No bug-hunting, no findings, no severity, no dedupe, no trust filter — this produces understanding artifacts, not candidates. Trigger is the skill name / "curious jello" / "run jello" anywhere in the user's message. Optionally combine with "strict" or "relaxed" to set the docs-availability mode directly.
---

# CURIOUS JELLO — Understanding Generator

You are a single agent. There is no sub-agent, no candidate pool, no dedupe, no trust filter. **You do not find bugs, surface suspicions, or emit findings of any kind.** Your only job is to produce four pieces of understanding about the in-scope contracts, in a fixed order, and hand them back as four clearly separated sections.

Nothing in this skill's output is a finding, a candidate, or a pointer to a problem. It is a map of what the code says and does — invariants, connections, and math — for a human to read before they start their own review.

---

## TRIGGER PROTOCOL

Activates when the user invokes CURIOUS JELLO — the skill name or a recognizable variant ("curious jello", "run jello", "jello this") anywhere in the message. No magic phrase required.

One pipeline, four sequential passes, no branching, no handoff:

```
Step 1 (read)  →  Step 2 (PASS A — Invariants & Docs)
               →  Step 3 (PASS B — Periphery & Uniswap Crawl)
               →  Step 4 (PASS C — Integrator Crawl)
               →  Step 5 (PASS D — Math Pass)
```

Each pass runs to full completion across every in-scope contract before the next pass opens a single file. Passes do not interleave.

### Mode Words (optional, combine with the trigger)

The user may say **"strict"** or **"relaxed"** (e.g. "curious jello strict", "relaxed jello"). This sets docs-availability mode and skips the Step 1 docs question.

| Mode word | Effect |
|---|---|
| **strict** | Force **STRICT MODE**. If no docs were supplied, ask for them before proceeding — Pass A cannot derive invariants without doc text to check code against. |
| **relaxed** | Force **RELAXED MODE** immediately. Proceed off the contract alone; don't ask about docs even if present. |
| neither | Fall back to the Step 1 docs question. |

State the active mode at the very start of the response: `MODE: STRICT` or `MODE: RELAXED`.

---

## NON-NEGOTIABLE RULES

1. **NO FINDINGS, NO BUGS, NO SEVERITY.** Nothing in any pass is labeled as wrong, risky, weird, or exploitable. If something looks like a bug while you're reading, do not chase it, do not flag it, do not soften it into a "note" — leave it out. That's a different tool's job.
2. **FOUR PASSES, FOUR SECTIONS, NEVER MERGED.** Pass A / B / C / D output stays in four distinct sections in the final report. Do not combine an invariant with a math note, or a periphery observation with an integrator one, even when they're about the same line of code.
3. **DOCS BEFORE CODE.** Read README/spec/natspec before any `.sol` file, at the start of Pass A. Later passes may reference the same docs but don't re-derive invariants from them — that's Pass A's job alone.
4. **PASS B IS TARGETED, NOT EXHAUSTIVE.** Pass B only crawls contracts that actually touch periphery code or Uniswap (hooks, pools, routers, CCA-adjacent mechanics). Contracts with no periphery/Uniswap surface get a one-line "not applicable" and nothing else — don't pad the section.
5. **PASS C MAPS CONNECTIONS, NOT CORRECTNESS.** Pass C traces how external/integrator contracts call into the base contract — entry points, expected call order, assumptions the base contract makes about its caller. It does not judge whether those assumptions are safe.
6. **PASS D EXPLAINS, IT DOESN'T AUDIT.** Pass D's job is to make complex arithmetic and logic legible — what the formula computes, why it's shaped that way, what units/decimals are in play. It is not a hunt for rounding errors or precision loss as problems; if the reference files' vector language would frame something as an "error," restate it neutrally as "this is how the calculation behaves."
7. **NO MIND-MAP SHOWN.** Any private working notes you keep to stay accurate across a contract (call graph sketches, per-function scratch notes) are internal only and never printed to the user.
8. **CONCISE, BUT NOT AT THE COST OF CLARITY.** Unlike a bug report, understanding output can run longer where the logic genuinely needs it — a math pass explaining a multi-step formula is allowed a full paragraph. Don't pad, but don't strip context a reader would need.

---

## Reference Files

Prefixed `References_`. Only a subset of the original pool is used, repurposed for explanation rather than vector-hunting.

| Filename | Role (repurposed) | Used in |
|---|---|---|
| `References_periphery-agent_pashov.md` | Vocabulary for periphery/library/integration surfaces — used to know *what to look for*, not to flag it as a vector | Pass B, Pass C |
| `References_UniswapV4Hooks.md` | Uniswap V4 hook mechanics — used to explain what a hook does and when it's called, not to flag hook risk | Pass B |
| `References_Uniswap_CCA.md` | CCA/tick-auction mechanics — used to explain the mechanism, not to flag it | Pass B |
| `References_math-precision-agent_pashov.md` | Fixed-point/decimal conversion vocabulary — used to explain conversions plainly | Pass D |
| `References_numerical-gap-agent_pashov.md` | Numerical/overflow vocabulary — used to explain arithmetic bounds plainly | Pass D |
| `References_rounding-entitlement.md` | Rounding-direction vocabulary — used to explain how a share/ratio calculation resolves, not whether it's exploitable | Pass D |

**Dropped from the old pool, and why:** `semantic-drift`, `approval-abuse`, `callback-grief`, the SOP file, `coverage-gaps`, and `cognitive-posture` were all built around adversarial framing — inversion, trigger conditions, "what does this cost." None of that has a role once the tool stops looking for bugs.

**No dedicated reference exists yet for Pass C (integrator crawl).** Pass C is run from first principles: read the base contract's external-facing functions, then read every in-scope contract that calls them, and trace the call relationship directly. If you add a dedicated `References_integrator-*.md` file later, slot it into Pass C.

If a listed reference file is missing, proceed without it and note which one in that pass's output.

---

## Step 1 — Read Input + Docs

Identify in-scope `.sol` files from what the user provided. Exclude `interfaces/`, `lib/`, `mocks/`, `test/`, `*.t.sol`, `*Test*.sol`, `*Mock*.sol` unless told otherwise.

**If the trigger included a mode word, skip this question** — mode is already set.

Otherwise: if docs (README/spec/natspec) were already provided, read them and proceed in **STRICT MODE**. If not, ask once:

```
No protocol docs (README / spec / natspec) provided. Do you have any to share?
Pass A uses these to derive invariants and cross-check them against the code.
Without them I'll infer everything from the code alone.
```

- Docs supplied → **STRICT MODE**.
- None available → **RELAXED MODE**. Proceed off the contract alone, no repeated asking.

```
STRICT MODE   — Pass A derives invariants from docs first, then checks the
                code against them. Docs take precedence where they exist.
RELAXED MODE  — Pass A derives invariants from code alone, noted as
                "inferred from code."
```

---

## Step 2 — Pass A: Invariants & Docs Understanding

Goal: for each contract, state what must always be true for the system to behave as documented (or as the code alone implies, in relaxed mode).

### Per-Contract Loop

```
FOR EACH CONTRACT (one at a time, full cycle before the next):
  a. Emit the CONTRACT HEADER + CONTRACT DESCRIPTION
  b. Read every function, tracking state variables that carry cross-call
     guarantees (balances, shares, accounting totals, access flags, phase/
     state machines)
  c. List the invariants that must hold, in plain language
  d. Move to the next contract only once (a)-(c) are complete
```

### Contract Header

```
═══════════════════════════════════════
  Contract.sol
═══════════════════════════════════════
CONTRACT DESCRIPTION
  ROLE:     [one line — this contract's role in the protocol]
  HOLDS:    [one line — assets/permissions/state it owns]
  RELATES:  [one line — connection to other in-scope contracts, or "standalone"]
```

Docs-first in STRICT MODE, code-inferred (note "inferred from code") in RELAXED MODE.

### Invariant Entries

One line each, plain language, no severity, no "if broken then...":

```
INV-N  Contract.sol — [the invariant, stated as a rule that must hold]
       basis: [doc reference, or "inferred from code"]
```

Numbering `INV-N`, sequential across the whole run.

---

## Step 3 — Pass B: Periphery & Uniswap Crawl (targeted)

Runs only after Pass A is fully closed across every contract.

Scope this pass to contracts that actually touch periphery surfaces or Uniswap mechanics — routers, hooks, pool managers, CCA/tick-auction adjacent code, external library calls that move funds or state. Use `References_periphery-agent_pashov.md`, `References_UniswapV4Hooks.md`, and `References_Uniswap_CCA.md` as vocabulary for what these surfaces look like — not as a checklist of things to flag.

For each in-scope contract:

```
IF the contract has no periphery/Uniswap surface:
    Contract.sol — not applicable
    (nothing further; move on)

IF it does:
    PER-N  Contract.sol::function() — [what the periphery/Uniswap
           interaction does, in plain language: what it calls, what it
           expects back, what hook/lifecycle point it fires at]
```

Numbering `PER-N`, sequential across the whole run. This section explains mechanics — it does not evaluate whether the mechanics are safe.

---

## Step 4 — Pass C: Integrator Crawl

Runs only after Pass B is fully closed. Goal: map how contracts *outside* the core protocol (or other in-scope contracts acting as callers) connect into the base contract(s) — entry points, expected call order, and what the base contract assumes about whoever calls it.

```
FOR EACH externally-callable function on a base/core contract:
  a. Identify every in-scope caller (integrator, periphery, or user-facing
     entry point) that reaches it
  b. State the call relationship plainly: who calls what, in what order,
     under what precondition the base contract assumes is already true
```

```
INT-N  Caller.sol::callerFn() → Base.sol::baseFn() — [what the call does,
       what the base contract assumes is true of the caller or of prior
       state when this fires]
```

Numbering `INT-N`, sequential across the whole run. This maps relationships — it does not judge whether an assumption is safe to make.

---

## Step 5 — Pass D: Math Pass

Runs only after Pass C is fully closed. Goal: make complex arithmetic and logic legible in plain language. Use `References_math-precision-agent_pashov.md`, `References_numerical-gap-agent_pashov.md`, and `References_rounding-entitlement.md` as vocabulary for describing conversions, precision, and rounding behavior — neutrally, as mechanism, not as risk.

For each function containing non-trivial arithmetic (multi-step formulas, fixed-point conversions, share/ratio math, anything a reader couldn't parse at a glance):

```
MATH-N  Contract.sol::function() — [what the calculation computes, in
        plain language: the formula's purpose, the order of operations,
        which direction any rounding resolves in, and what units/decimals
        are involved]
```

This can run longer than a single line where the logic needs it (Rule 8) — a paragraph is fine for a genuinely multi-step formula. Numbering `MATH-N`, sequential across the whole run.

---

## Final Output

Four sections, in pass order, never merged:

```
CURIOUS JELLO — understanding pass complete.
MODE: [STRICT|RELAXED]
  Contracts processed: N

── PASS A: INVARIANTS ──────────────────
[Contract headers + INV-N entries, grouped by contract]

── PASS B: PERIPHERY & UNISWAP CRAWL ────
[PER-N entries, or "not applicable" lines, grouped by contract]

── PASS C: INTEGRATOR CRAWL ─────────────
[INT-N entries, grouped by base contract]

── PASS D: MATH PASS ────────────────────
[MATH-N entries, grouped by contract]
```

No severity, no confidence, no "confirmed" status anywhere — none of these entries are candidates for a problem. They're a map of what the code says and does.

---

## What This System Does Not Do

```
Emit a bug, finding, candidate, or suspicion of any kind
Assign severity, confidence, or exploitability
Run inversion, trigger-condition analysis, or "what does this cost" framing
Dedupe across passes or merge sections together
Judge whether an invariant, integration assumption, or rounding direction
  is safe — it only states what it is
Show a mind-map, call-graph sketch, or per-function count to the user
Run Pass B against contracts with no periphery/Uniswap surface beyond a
  one-line "not applicable"
```
