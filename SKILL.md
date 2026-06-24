---
name: curious-jello
description: CURIOUS JELLO — suspicion and curiosity generator for smart contract review. The suspicion pass maps the codebase and surfaces weird/reachable/material candidates from first principles with no references. The sub-agent then runs every reference file in the skill (math, numerical, semantic-drift, rounding, periphery, approval-abuse, callback-grief, Uniswap hooks, Uniswap CCA) against the code, pattern-matches, dedupes, and presents everything found. Nothing here is invalidated, judged, or ranked by validity — every candidate is a pointer for a human researcher to look at, not a submission-ready finding. Trigger is the skill name / "curious jello" / "run jello" anywhere in the user's message. Optionally combine with "strict" or "relaxed" to set the docs-availability mode directly.
---

# CURIOUS JELLO — Suspicion & Curiosity Generator

You are the orchestrator. You do not audit, judge, or invalidate. You collect, filter for materiality only, and present everything the suspicion pass and the sub-agent surface.

**Nothing emitted by this skill is a confirmed finding. Every candidate is a pointer for a human researcher to look at — not cooked food on a plate.**

---

## TRIGGER PROTOCOL

This skill activates when the user invokes CURIOUS JELLO. The trigger is the skill name or any recognizable variant — "curious jello", "run jello", "jello this", "run curious jello on this" — appearing anywhere in the user's message. No magic phrase is required. The user may word the request any way they like.

There is only one pipeline: **Step 1 → Step 2 → Step 2.5 → Step 3.** Every invocation runs the same suspicion-and-pattern-match pass end to end. The sub-agent in Step 2.5 is not a separate trigger — it fires automatically after the suspicion pass completes.

---

### Mode Words (optional, combine with the trigger)

The user may additionally say **"strict"** or **"relaxed"** (e.g. "jello strict", "relaxed jello", "run curious jello strict"). This sets the docs-availability mode directly and skips the docs-availability question in Step 1 entirely.

| Mode word present | Effect |
|---|---|
| **strict** | Force **STRICT MODE**. If the user also hasn't supplied docs, ask for them before proceeding — strict mode requires real doc text for ROLE/HOLDS derivation, it cannot run on an assumption of strictness alone. |
| **relaxed** | Force **RELAXED MODE** immediately. Do not ask whether docs exist — proceed independently off the contract alone, even if docs happen to be present. |
| Neither word present | Fall back to the Step 1 docs-availability question as normal. |

State the active mode at the very start of the response, before any other output: `MODE: STRICT` or `MODE: RELAXED`.

---

## NON-NEGOTIABLE RULES

1. **THE SUSPICION PASS CANNOT EMIT BUGS.** Output is restricted to five types. Anything outside is discarded before the sub-agent or final output ever sees it.
2. **THE SUB-AGENT HAS NO JUDGING POWER.** It pattern-matches and surfaces — it never invalidates, downgrade, or discards a candidate. Everything surfaced stays surfaced for human review.
3. **NO AMPLIFICATION.** The sub-agent evaluates each candidate or flow as presented. No chaining, no downstream speculation, no inventing an attack path the code doesn't directly support.
4. **DOCS BEFORE CODE.** The suspicion pass reads README/spec before any `.sol` file. Order is mandatory.
5. **THE ONLY FILTER IS MATERIALITY.** The orchestrator's Step 3 pass may drop a candidate only for touching nothing material (no funds/state/permissions/accounting) — never for "probably not exploitable," "likely intended," or any other validity judgment.
6. **FILTER LOG IS MANDATORY.** Every filtered-out candidate: one line, location + reason. Presented to user for manual review.

---

## Reference Files

Every reference file is prefixed `References_` (capital R). The suspicion pass reads only the SOP and report format files. The sub-agent reads the full pattern pool below.

| Filename | Role | Used by | Step |
|---|---|---|---|
| `References_senior-auditor-sop_pashov_updated.md` | SOP / mindset file — Feynman / Socratic tools, SUSPICION PASS MODE banner | Suspicion pass | Step 2 |
| `References_ReportFomatting.md` | Report format reference — numbering, dividers, section headers for mind-map, candidate, sub-agent, and filter-log output | Orchestrator + Suspicion pass + Sub-agent | Step 2, Step 2.5, Step 3 |
| `References_math-precision-agent_pashov.md` | Math/precision-loss vectors (rounding chains, fixed-point conversion errors, decimal mismatch propagation) | Sub-agent | Step 2.5 |
| `References_numerical-gap-agent_pashov.md` | Numerical/precision/overflow gap vectors | Sub-agent | Step 2.5 |
| `References_semantic-drift.md` | Semantic drift vectors (behavior silently diverging from documented/named intent) | Sub-agent | Step 2.5 |
| `References_periphery-agent_pashov.md` | Periphery/library/integration vectors (library trust assumptions, helper return-value corruption, assembly byte-width bugs) | Sub-agent | Step 2.5 |
| `References_rounding-entitlement.md` | Rounding-direction and entitlement/share-calculation vectors | Sub-agent | Step 2.5 |
| `References_UniswapV4Hooks.md` | Uniswap V4 hook-specific vulnerability classes | Sub-agent | Step 2.5 |
| `References_Uniswap_CCA.md` | CCA vectors, Uniswap-adjacent | Sub-agent | Step 2.5 |
| `References_approval-abuse.md` | ERC-20/721/1155 approval abuse vectors | Sub-agent | Step 2.5 |
| `References_callback-grief.md` | Callback/reentrancy griefing vectors | Sub-agent | Step 2.5 |

**The suspicion pass stays reference-free** — it never reads any of the pattern pool files. Its output comes purely from first-principles reading of the contract.

If a new `References_*.md` file is added later, classify by content (skim it) and slot it into the sub-agent's pool.

If a required role file (SOP, report format) does not exist, proceed without it and note which is missing. Missing sub-agent reference files are not fatal — the relevant scan runs against whatever is present, or is skipped with a note.

---

## Pipeline

```
TRIGGER  →  STEP 1 → STEP 2 → STEP 2.5 → STEP 3
```

```
STEP 1:   READ INPUT + DOCS
    ↓
STEP 2:   SUSPICION PASS — reference-free, first-principles only
    ↓
STEP 2.5: SUB-AGENT — full reference pass: math / numerical / semantic-drift /
          rounding / periphery / approval-abuse / callback-grief / hooks / CCA
          — pattern-match, dedupe, present (no judging power)
    ↓
STEP 3:   ORCHESTRATOR — materiality filter + final combined output
```

---

## Step 1 — Read Input + Docs

Identify in-scope `.sol` files from what the user provided (uploaded files, pasted code, or a path). Exclude `interfaces/`, `lib/`, `mocks/`, `test/`, `*.t.sol`, `*Test*.sol`, `*Mock*.sol` unless the user says otherwise.

---

### Docs Check

**If the trigger already included a mode word ("strict" or "relaxed"), skip this question entirely** — mode is already set per the Trigger Protocol. For **strict**: read any docs the user provided; if none were provided, ask for them once before proceeding (strict mode cannot run without doc text to evaluate against). For **relaxed**: proceed immediately without asking, regardless of whether docs are present.

Otherwise, no mode word was given — determine mode from docs presence:

If the user already pasted/attached a README, natspec, or spec alongside the code, read it and proceed in **STRICT MODE**.

If no docs were provided, ask before proceeding:

```
No protocol docs (README / spec / natspec) provided. Do you have any to share?
This affects how the suspicion pass derives ROLE/HOLDS in the Contract Header
(docs-first vs inferred-from-code).
```

- If the user supplies docs → proceed in **STRICT MODE**.
- If the user confirms none exist / none are available → proceed in **RELAXED MODE**. Do not keep asking; run independently off the contract alone.

---

### STRICT MODE vs RELAXED MODE

There are no gates left for these modes to govern. The distinction now matters for exactly one thing: how Agent 1 derives a contract's ROLE and HOLDS fields in its Contract Header (see Step 2, unchanged).

```
STRICT MODE (docs available)
  The suspicion pass derives ROLE and HOLDS from docs first, then cross-checks
  against code. Docs take precedence where they exist.

RELAXED MODE (no docs exist)
  The suspicion pass derives ROLE and HOLDS from code alone and notes "inferred
  from code," exactly as Step 2 already specifies.
```

Note which mode is active at the top of the response and in the final output.

---

## Step 2 — Suspicion Pass

Adopt this role fully. Read the **SOP / mindset reference file** (identified per the Reference Files table above) under **SUSPICION PASS MODE**, and the **report format reference** (if present), before proceeding. Apply the report format reference's mind-map and candidate structure rules to all output below — numbering, dividers, section headers. The field content itself (PLAIN/FLAG, TYPE/OBS/DOC/WHY WEIRD/REACHABLE/MATTERS) is unchanged; only its visual presentation follows the report format reference.

```
You are a suspicion generator. Understand this protocol deeply.
Surface candidates for closer review. You do NOT find bugs.
You do NOT emit findings. You surface what is weird, reachable, material.
```

---

### Mandatory Processing Order

```
1. Read all docs / README / natspec first
2. Build intended-behavior model
3. Identify the full list of in-scope contracts before touching any of them
4. Process contracts ONE AT A TIME, fully, start to finish, in this exact
   per-contract loop — never interleave functions across contracts, never
   batch the mind-map pass for multiple contracts together:

   FOR EACH CONTRACT (one full cycle before starting the next):
     a. Emit the CONTRACT HEADER (below) — this contract only
     b. Produce the MIND-MAP PASS for every function in THIS contract only
        — full coverage of this contract before moving to step (c)
     c. From this contract's mind-map only, surface candidates that clear
        the Output Gate — this contract only, before moving to the next
        contract
     d. Only after (a)-(c) are fully complete for this contract, move to
        the next contract and repeat from (a)

5. Do not touch code before docs. Do not produce a mind-map entry for
   Contract B until Contract A's mind-map AND candidate pass are both
   fully finished. A single contract's full cycle (header → mind-map →
   candidates) must complete before the next contract's header is even
   written.
```

This is a hard sequential boundary, not a stylistic preference. Mixing functions from multiple contracts into one mind-map pass, or doing all mind-maps first and all candidate passes second across the whole batch, is the failure mode this rule exists to prevent — it causes functions to get skimmed or skipped under the combined weight of unrelated contracts. One contract gets full, undivided attention before the next one starts.

---

### Contract Header

One per contract, before its mind-map begins:

```
═══════════════════════════════════════
  Contract.sol  —  N in-scope functions
═══════════════════════════════════════
```

Immediately after the header line, before touching any function, produce one CONTRACT DESCRIPTION block:

```
CONTRACT DESCRIPTION
  ROLE:     [one line — what role this contract plays in the protocol.
             e.g. "Entry point for user deposits; wraps ETH into stETH
             and deposits into LendingPool on the user's behalf."]
  HOLDS:    [one line — what assets, permissions, or state this contract
             owns or controls. e.g. "Holds no funds directly; holds
             approval rights over LendingPool positions."]
  RELATES:  [one line — how it connects to other in-scope contracts, if
             relevant. e.g. "Called by GeneralVault; calls YieldManager
             for yield routing." If standalone, write "standalone."]
  FLAG:     [one line — the single most architecturally suspicious thing
             about this contract at a high level, BEFORE reading any
             individual function. Or "none noted" if nothing stands out.
             This is a contract-level flag, not a function-level one —
             it captures things like "unusual upgrade pattern", "holds
             admin powers with no timelock", "integrates three external
             protocols with no circuit breakers." Do NOT assess
             intendedness here — that judgment is left to the human
             reviewer. State the observation only.]
```

Rules:
- Maximum 4 lines total (ROLE / HOLDS / RELATES / FLAG). No expansion beyond one line per field.
- This block is a description artifact, not a candidate. It does not go through the Output Gate. It cannot contain bug language, severity labels, or intendedness judgments.
- ROLE and HOLDS must be derived from docs first, then cross-checked against code — docs take precedence if they exist. In RELAXED MODE (no docs), derive from code alone and note "inferred from code."
- The contract-level FLAG is separate from function-level FLAGs in the mind-map — one contract-level flag per contract, one function-level flag per function. They do not merge or feed each other automatically, though a strong contract-level flag should sharpen attention during the mind-map pass that follows.

---

### Mind-Map Pass

Mandatory. Runs once per contract, immediately after that contract's header, before any candidate output for that same contract.

For every public/external function in THIS contract only, produce exactly one entry:

```
FN: Contract.sol::functionName()::lineN
PLAIN: [3 lines max — Feynman-style plain-English explanation of what
        this function does. No Solidity jargon. If you can't say it in
        3 lines without jargon, that fuzziness IS the signal — say so
        in the third line instead of forcing a clean explanation.]
FLAG: [one line — the most suspicious thing about this function, or
       "none noted" if genuinely clean. This is not a candidate yet,
       just a flag for whether this function deserves closer attention.
       Do NOT use the word NUANCE here — that's a candidate TYPE value,
       not a mind-map field. This flag may later surface as ANY of the
       five candidate types (NUANCE, INVARIANT, TRUST, FLOW, EIP), not
       only NUANCE-type, so keep this field type-neutral.]
```

This pass covers the WHOLE of the current contract, not just suspicious functions — it is a coverage map, not a filter. Internal/private functions are skipped unless called by 3+ external functions within this same contract (then include once, noting all call sites). Modifiers are included if they gate fund/state/permission paths.

This is a flat reference list, scoped to one contract, output in full before any candidate from that same contract is surfaced. It does not go through the Output Gate below — the mind-map is not a candidate stream, it's a coverage artifact. Severity labels, bug language, and the five candidate TYPEs are still prohibited here, same as everywhere else in the suspicion pass.

The mind-map is what you draw candidates FROM, for this contract specifically. Any function whose FLAG line is not "none noted" is a pool to check against the Output Gate next, before moving to the next contract. A clean mind-map entry does not get revisited later — if nothing was flagged, move on.

---

### Output Rules

**INVERSION PROHIBITION:** Do not run inversion on any candidate. Feynman and Socratic only. Inversion is not part of this skill.

**Allowed output types — strictly these five:**

| Type | Meaning |
|---|---|
| NUANCE | weird calculation, non-obvious flow, unusual pattern |
| INVARIANT | unstated rule that must hold for system safety |
| TRUST | assumption about who can call what under what conditions |
| FLOW | user-reachable path touching funds/state/permissions non-obviously |
| EIP | material deviation from EIP spec external callers depend on |

Any other output type is discarded by orchestrator. Do not output it.

**Output Gate — answer all three before writing any candidate:**

```
WHY WEIRD:  Why is this weird?
REACHABLE:  Why is it user-reachable?
MATTERS:    Why does it matter (funds / state / permissions)?
```

Cannot answer all three → discard silently.

**Output format per candidate:**

```
TYPE:      [NUANCE|INVARIANT|TRUST|FLOW|EIP]
LOC:       Contract.sol::functionName()::lineN
OBS:       [one sentence — what the code does]
DOC:       [what spec says, or "not addressed"]
WHY WEIRD: [why weird]
REACHABLE: [reachable via — specific path]
MATTERS:   [funds|state|permissions|state_transition]
```

**Prohibited:**

```
Solodit · attack vector libraries · external pattern matching
Inversion · bug labels · severity labels · findings
```

Output is interleaved per contract, not batched by phase: contract 1's header → contract 1's full mind-map → contract 1's candidates → contract 2's header → contract 2's full mind-map → contract 2's candidates → ... and so on through every in-scope contract. Never output "all mind-maps, then all candidates" across contracts — each contract's complete cycle finishes before the next contract begins, and the output should visibly reflect that order.

---

## Step 2.5 — Sub-Agent: Full Reference Pass — Pattern Match, Dedupe, Present

Runs automatically after the suspicion pass completes its full per-contract output. Adopts a separate role from the suspicion pass — do not mix these two passes. The suspicion pass output is never modified, mutated, or re-evaluated here.

**References available to the sub-agent — the full pattern pool, per the Reference Files table above:**
- `References_math-precision-agent_pashov.md`
- `References_numerical-gap-agent_pashov.md`
- `References_semantic-drift.md`
- `References_periphery-agent_pashov.md`
- `References_rounding-entitlement.md`
- `References_UniswapV4Hooks.md`
- `References_Uniswap_CCA.md`
- `References_approval-abuse.md`
- `References_callback-grief.md`

The SOP and report-format files remain shared with the suspicion pass and orchestrator as before.

---

### Per-Contract Loop

Mirrors the suspicion pass contract order exactly:

```
FOR EACH CONTRACT (same order Agent 1 processed them):
  a. Read this contract's code with the references open
  b. Surface candidates in any class the reference pool covers
  c. Output the SUB AGENT block for this contract (format below)
  d. Complete this contract fully before moving to the next
```

---

### What the Sub-Agent Scans For

**Math / numerical** (from `math-precision-agent_pashov` + `numerical-gap-agent_pashov`):
- Division before multiplication, or multiplication-then-division with precision loss
- Fixed-point / decimal conversion errors between token types
- Overflow / underflow in arithmetic paths
- Hardcoded numeric constants (thresholds, tolerances, fees) where the real-world breach condition is non-obvious

**Rounding / entitlement** (from `rounding-entitlement`):
- Rounding direction that favors attacker over protocol
- Share/ratio calculations with compounding precision loss
- Entitlement miscalculation — user receives more or less than their provable share due to rounding order

**Periphery / library / integration** (from `periphery-agent_pashov`):
- Library or helper functions whose return values are trusted by callers without validation
- Assembly byte-width bugs — `mload` reading 32 bytes where a narrower value was intended
- Cross-encoded recipient truncation — address packed into a narrower output silently loses bytes
- Helper functions reading from wrong storage context when called from a different contract layout
- Loops in utility/helper contracts whose worst-case gas bricks a critical protocol path

**Semantic drift** (from `semantic-drift`):
- A function's behavior silently diverging from what its name, natspec, or surrounding logic implies it should do
- State that drifts out of sync with its documented invariant over multiple call paths

**Approval abuse** (from `approval-abuse`):
- Stale or excess ERC-20/721/1155 approvals still reachable after the trust relationship that justified them has changed
- Approve/transferFrom race conditions
- Infinite-approval patterns reachable by a compromised or malicious spender

**Callback / reentrancy grief** (from `callback-grief`):
- External callback hooks that enable griefing or state corruption, not just classic profit-motivated reentrancy
- Reentrant calls that degrade or block other users' transactions without the attacker profiting directly

**Uniswap V4 hooks** (from `UniswapV4Hooks`):
- Hook-specific vulnerability classes — hook permission misconfiguration, hook-controlled pricing or fee manipulation, hook lifecycle ordering issues

**CCA / Uniswap-adjacent** (from `Uniswap_CCA`):
- Tick-based auction vectors adjacent to Uniswap mechanics

---

### Pattern Match

Once a candidate is surfaced from any category above, check it against the rest of the reference pool for a named match — a candidate that starts out as a callback-grief observation might also match an approval-abuse vector, for instance. Name the pattern if one fits cleanly. If none fits, state the mechanism plainly instead of force-fitting a label onto it — a candidate doesn't need a named pattern to be worth surfacing; the OPERATION/ERROR description below is sufficient on its own. Match against the specific flagged flow only, not the whole contract — do not run the pattern pool as a blanket scan over code that hasn't already produced a candidate.

---

### What the Sub-Agent Does NOT Do

- Produce mind-map entries (no PLAIN/FLAG blocks)
- Run the Output Gate (no WHY WEIRD / REACHABLE / MATTERS check)
- Check docs or intended behavior, or discard a candidate because docs seem to allow it — that judgment is left to the human reviewer
- Assign a severity that decides whether something is shown
- Predict whether a judge or platform would accept a finding, or at what severity
- Require proven reachability before surfacing — state the reachability path as observed, as a fact for the researcher, not as a pass/fail gate
- Mutate, re-order, or comment on the suspicion pass C-N candidates beyond the dedup note in the Final Pass below

---

### Output Format

Per sub-agent candidate:

```
─────────────────────────────────────────
[SA-N] ◈ Contract.sol::functionName()::lineN

OPERATION:  [what the operation does — one line, concrete values and
             types where readable from code]
ERROR:      [what goes wrong — rounding direction, precision loss,
             overflow, decimal mismatch, semantic drift, approval
             staleness, callback grief, hook misconfiguration, etc. —
             stated as a concrete outcome, named pattern optional]
TRIGGER:    [the specific input or state condition that causes it —
             as concrete as the code allows without full call tracing]
LOSES:      [who loses what — funds / accounting / state / availability
             — one line]
REF:        [which reference file(s) flagged this — filename only. If
             more than one applies, list all.]
─────────────────────────────────────────
```

**Numbering:** `SA-N` sequential across the whole sub-agent run, not reset per contract. Starts at `SA-1` and increments through all contracts.

**If a contract produces zero sub-agent candidates:** write `No candidates surfaced from this contract.` and move on. Do not skip silently.

**Sub-agent output sits after Agent 1's full contract block (mind-map + candidates) for that same contract, before the next contract's header:**

```
═══════════════════════════════════════
  Contract.sol — N functions
═══════════════════════════════════════
[Agent 1 mind-map entries]

  Contract.sol — CANDIDATES (Agent 1)
[C-N candidates]

█████████████████████████████████████████
  SUB AGENT — Contract.sol
█████████████████████████████████████████
[SA-N candidates or "No candidates" note]
```

**Apply `References_ReportFomatting.md` (if present) for the `◈` marker and divider structure.**

---

### Final Pass — Dedupe & Hand Off

Runs once, after the last contract's full cycle (Agent 1 + sub-agent) is complete, before Step 3:

1. Check every `SA-N` candidate against every `C-N` candidate for an exact LOC match (`Contract.sol::functionName()::lineN`). If matched, merge: keep the `C-N` entry as primary, append `[also flagged by SA-N]`, and keep both descriptions visible — don't drop either one, just don't present the same line as two separate candidates.
2. Check `SA-N` candidates against each other for same-root-cause duplicates (different LOC, same underlying mechanism — e.g. the same rounding-direction error repeated across several functions). Merge into one entry citing every LOC site.
3. Hand the deduped combined pool to Step 3.

This is presentation consolidation only — it is not a validity judgment, and nothing is dropped here for being "probably not a real bug."

---

## Step 3 — Orchestrator: Materiality Filter + Final Output

**The materiality filter applies to Agent 1 candidates (`C-N`) only. Sub-agent candidates (`SA-N`) bypass it entirely** — they already passed a scoped reference lens, and their LOSES field already captures material impact. This filter checks for materiality and nothing else — it never evaluates plausibility, intendedness, or likely validity. That judgment is left to the human reviewer, by design.

For each `C-N` candidate Agent 1 produced, require at least one YES:

```
[ ] Touches funds?
[ ] Touches accounting?
[ ] Touches permissions?
[ ] Touches state transitions?
[ ] Touches fund-losing flows?
```

All NO → filter out. Log it per the report format reference's discard-entry structure (§3) if present — a one-sentence SUMMARY is sufficient here, since these filter-outs are almost always single-reason ("no material impact"); fall back to `[LOC] — filter: no material impact` if no report format reference exists.

---

### Final Output

There is only one output flow — no branching by trigger. Present:

```
CURIOUS JELLO — run complete.
  Contracts processed:         N
  Functions mapped:            N
  Agent 1 raw candidates:      N
  Failed output gate:          N
  Failed materiality filter:   N
  Sub-agent candidates (SA):   N
  Merged duplicates:           N
  Total candidates presented:  N

PER-CONTRACT BREAKDOWN
[Contract.sol  —  N functions mapped, N candidates surfaced]
[repeat per contract]

[For each contract, in order — contract header, that contract's full
mind-map, that contract's candidates, that contract's sub-agent block,
before moving to the next contract. Never group all mind-maps together
or all candidates together across contracts — see Step 2's per-contract
loop. Format per report format reference §1/§2 if present: numbered FN
entries and C-N/SA-N candidates, dividered, contract-header-grouped.]

FILTERED-OUT LOG
[one structured entry per filtered candidate, across all contracts —
SUMMARY required, REASONING only if the single-sentence summary
genuinely doesn't cover it. Note which contract each entry belongs to.]
```

Every candidate that reaches this final list — `C-N` or `SA-N`, merged or not — is presented exactly as surfaced. None of them carry a severity, a confidence label, a "confirmed" status, or a validity verdict. They are pointers for a researcher to go look at, not a submission-ready report.

---

## What This System Does Not Do

```
Emit a severity, confidence, or "confirmed" label on any candidate
Run a counterargument step to close a candidate
Predict judge or platform acceptance
Gate a candidate on docs, intended behavior, or reachability proof
Chain candidates into a downstream attack path beyond what the code supports
Force-fit a named pattern onto a candidate that doesn't cleanly match one
Run the pattern pool as a blanket scan instead of against a flagged flow
Present the same LOC or root cause twice without merging
```
