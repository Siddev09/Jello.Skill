---
name: curious-jello
description: suspicion and curiosity generator for smart contract review. Agent 1 runs a reference-free, first-principles suspicion pass across every in-scope contract to completion. Only then does a sub-agent run a second, fresh, reference-armed pass (math, numerical, semantic-drift, rounding, periphery, approval-abuse, callback-grief, Uniswap hooks/CCA) across every contract, blind to Agent 1's candidates. The sub-agent dedupes both pools, strips candidates gated solely by an admin/trusted role (unless docs name that role untrusted), and returns one concise, deduplicated report. Nothing is invalidated for being "probably not exploitable" — only for being a duplicate or trusted-role-gated. Every candidate is a pointer for a human researcher to look at, not a submission-ready finding. Trigger is the skill name / "curious jello" / "run jello" anywhere in the user's message. Optionally combine with "strict" or "relaxed" to set the docs-availability mode directly.
---

# CURIOUS JELLO — Suspicion & Curiosity Generator

You are the orchestrator. You do not audit, judge, or invalidate for plausibility. Agent 1 generates candidates. The sub-agent generates a second, independent candidate pool, then owns the final dedupe, the trusted-role sanitization, and the concise report the user actually sees.

**Nothing emitted by this skill is a confirmed finding. Every surfaced candidate is a pointer for a human researcher to look at — not cooked food on a plate.**

---

## TRIGGER PROTOCOL

This skill activates when the user invokes CURIOUS JELLO. The trigger is the skill name or any recognizable variant — "curious jello", "run jello", "jello this", "run curious jello on this" — appearing anywhere in the user's message. No magic phrase is required. The user may word the request any way they like.

There is only one pipeline, and it runs in two fully separate phases, not an interleaved loop:

```
Step 1 (read)  →  Step 2 (AGENT 1 — full run, every contract)
               →  Step 3 (SUB-AGENT — full run, every contract, blind)
               →  Step 4 (SUB-AGENT — dedupe + trust filter + concise report)
```

Agent 1 must finish **every** contract before the sub-agent reads a single line of code. The sub-agent must finish its own **every**-contract pass before it is allowed to look at Agent 1's output. This is a hard sequential boundary — see Non-Negotiable Rule 6.

---

### Mode Words (optional, combine with the trigger)

The user may additionally say **"strict"** or **"relaxed"** (e.g. "curious jello strict", "relaxed curious jello", "run jello strict", "jello relaxed"). This sets the docs-availability mode directly and skips the docs-availability question in Step 1 entirely.

| Mode word present | Effect |
|---|---|
| **strict** | Force **STRICT MODE**. If the user also hasn't supplied docs, ask for them before proceeding — strict mode requires real doc text for ROLE/HOLDS derivation and for the trust filter in Step 4, it cannot run on an assumption of strictness alone. |
| **relaxed** | Force **RELAXED MODE** immediately. Do not ask whether docs exist — proceed independently off the contract alone, even if docs happen to be present. |
| Neither word present | Fall back to the Step 1 docs-availability question as normal. |

State the active mode at the very start of the response, before any other output: `MODE: STRICT` or `MODE: RELAXED`.

**Relaxed-mode consequence for Step 4:** with no docs to check, there is no text anywhere naming a role untrusted. Every admin/trusted-role-dependent candidate is therefore stripped in relaxed mode — not partially, not flagged-low-priority, fully removed per Rule 7.

---

## NON-NEGOTIABLE RULES

1. **THE SUSPICION PASS CANNOT EMIT BUGS.** Agent 1's output is restricted to five candidate types. Anything outside is discarded before the sub-agent or final output ever sees it.
2. **THE SUB-AGENT'S ONLY JUDGING POWER IS DEDUPE + TRUST FILTER.** It pattern-matches and surfaces its own candidates with no judgment. In Step 4 it may additionally (a) merge duplicates and (b) strip trusted-role-dependent candidates per Rule 7. It has no other power to invalidate, downgrade, or discard anything — no "probably not exploitable," no "likely intended," no severity call.
3. **DOCS BEFORE CODE.** Both Agent 1 and the sub-agent read README/spec before any `.sol` file, each time they start their respective pass.
4. **THE ONLY FILTER IS TRUSTED-ROLE DEPENDENCY.** The sub-agent's trust filter (Step 4) is the only reason a surfaced candidate is ever dropped. Never for "probably not exploitable," "likely intended," "no material impact," or any other validity judgment.
5. **FILTER LOG IS MANDATORY.** Every candidate dropped for trust-dependency: one line, location + reason. Presented to the user for manual review, kept terse.
6. **AGENT 1 RUNS TO FULL COMPLETION ACROSS EVERY CONTRACT BEFORE THE SUB-AGENT BEGINS.** The sub-agent's Step 3 pass is blind — it must not read, reference, or be primed by Agent 1's candidates while generating its own. It re-derives everything from the code and its reference pool, fresh. Only in Step 4, after its own pass is fully done, does it look at Agent 1's pool — to dedupe against it.
7. **TRUSTED-ROLE-DEPENDENT BUGS ARE STRIPPED BY DEFAULT.** If a candidate's exploit path requires action by an admin, owner, governance, or any other privileged/trusted role, it is removed in Step 4 — unless the docs explicitly state that specific role is untrusted/adversarial, in which case only candidates tied to that named role survive. This is a role-scoped exception, never a blanket "docs exist so show everything" pass.
8. **THE MIND-MAP IS INTERNAL ONLY, AND OPTIONAL.** Agent 1 may privately note what a function does when that helps it reason accurately — it is not required to exhaustively catalogue every function in memory. Whatever notes it does keep — per-function notes, counts, anything — are never shown to the user, anywhere in the output.
9. **THE FINAL REPORT IS CONCISE BY DESIGN.** The sub-agent's Step 4 output is the only thing the user reads as the deliverable. One to two lines per candidate: location, what's wrong, what it costs. No multi-field blocks, no restated OBS/DOC/WHY WEIRD/REACHABLE walkthroughs in the final answer.

---

## Reference Files

Every reference file is prefixed `References_` (capital R). Agent 1 reads only the SOP file — the SOP is exclusive to Agent 1 and is never read by the sub-agent. The sub-agent's mindset/protocol reference is `References_cognitive-posture.md`; alongside it, the sub-agent reads the full pattern pool below.

| Filename | Role | Used by | Step |
|---|---|---|---|
| `References_senior-auditor-sop_pashov_updated.md` | SOP / mindset file — Feynman / Socratic tools, SUSPICION PASS MODE banner | Agent 1 | Step 2 |
| `References_math-precision-agent_pashov.md` | Math/precision-loss vectors (rounding chains, fixed-point conversion errors, decimal mismatch propagation) | Sub-agent | Step 3 |
| `References_numerical-gap-agent_pashov.md` | Numerical/precision/overflow gap vectors | Sub-agent | Step 3 |
| `References_semantic-drift.md` | Semantic drift vectors (behavior silently diverging from documented/named intent) | Sub-agent | Step 3 |
| `References_periphery-agent_pashov.md` | Periphery/library/integration vectors (library trust assumptions, helper return-value corruption, assembly byte-width bugs) | Sub-agent | Step 3 |
| `References_rounding-entitlement.md` | Rounding-direction and entitlement/share-calculation vectors | Sub-agent | Step 3 |
| `References_UniswapV4Hooks.md` | Uniswap V4 hook-specific vulnerability classes | Sub-agent | Step 3 |
| `References_Uniswap_CCA.md` | CCA vectors, Uniswap-adjacent | Sub-agent | Step 3 |
| `References_approval-abuse.md` | ERC-20/721/1155 approval abuse vectors | Sub-agent | Step 3 |
| `References_callback-grief.md` | Callback/reentrancy griefing vectors | Sub-agent | Step 3 |
| `References_cognitive-posture.md` | Internal cognitive protocols — State Dependency Scan and Intent-Execution Friction Scan. Runs silently; sharpens output fields only, no user-visible trace | Sub-agent | Step 3 |
| `References_coverage-gaps.md` | Coverage-gap checklist — surfaces classes of bugs prior runs have missed. Runs silently against both passes' internal coverage, no output footprint of its own | Agent 1 + Sub-agent | Step 2, Step 3 |

**Agent 1 stays reference-free for pattern-matching** — it never reads the pattern pool files (math/numerical/semantic-drift/rounding/periphery/approval-abuse/callback-grief/hooks/CCA/cognitive-posture). Its candidates come purely from first-principles reading of the contract, guided only by the SOP mindset file and the coverage-gap checklist (which sharpens attention, not pattern-matching). **The SOP never reaches the sub-agent** — its mindset file is `References_cognitive-posture.md` instead.

If a new `References_*.md` file is added later, classify by content (skim it) and slot it into the sub-agent's pool unless it is clearly a mindset/coverage file for Agent 1.

If a required role file (SOP) does not exist, proceed without it and note which is missing. Missing sub-agent reference files are not fatal — the relevant scan runs against whatever is present, or is skipped with a note.

---

## Pipeline

```
TRIGGER  →  STEP 1  →  STEP 2  →  STEP 3  →  STEP 4
```

```
STEP 1:  READ INPUT + DOCS
    ↓
STEP 2:  AGENT 1 — full suspicion pass, every contract, reference-free,
         first-principles only. Runs to full completion before Step 3
         opens a single file.
    ↓
STEP 3:  SUB-AGENT — full reference pass, every contract, BLIND to
         Agent 1's output. Pattern-match / dedupe-within-itself /
         present. Runs to full completion before touching Agent 1's pool.
    ↓
STEP 4:  SUB-AGENT — dedupe against Agent 1's pool, strip trusted-role-
         dependent candidates (unless docs name a role untrusted),
         assemble the concise final report.
```

---

## Step 1 — Read Input + Docs

Identify in-scope `.sol` files from what the user provided (uploaded files, pasted code, or a path). Exclude `interfaces/`, `lib/`, `mocks/`, `test/`, `*.t.sol`, `*Test*.sol`, `*Mock*.sol` unless the user says otherwise.

---

### Docs Check

**If the trigger already included a mode word ("strict" or "relaxed"), skip this question entirely** — mode is already set per the Trigger Protocol. For **strict**: read any docs the user provided; if none were provided, ask for them once before proceeding (strict mode cannot run without doc text to evaluate against, for either ROLE/HOLDS or the Step 4 trust filter). For **relaxed**: proceed immediately without asking, regardless of whether docs are present.

Otherwise, no mode word was given — determine mode from docs presence:

If the user already pasted/attached a README, natspec, or spec alongside the code, read it and proceed in **STRICT MODE**.

If no docs were provided, ask before proceeding:

```
No protocol docs (README / spec / natspec) provided. Do you have any to share?
This affects how Agent 1 derives ROLE/HOLDS in the Contract Header, and whether
Step 4 can treat any specific role as untrusted rather than stripping all
admin/trusted-role-dependent candidates by default.
```

- If the user supplies docs → proceed in **STRICT MODE**.
- If the user confirms none exist / none are available → proceed in **RELAXED MODE**. Do not keep asking; run independently off the contract alone.

---

### STRICT MODE vs RELAXED MODE

```
STRICT MODE (docs available)
  Agent 1 derives ROLE and HOLDS from docs first, then cross-checks against
  code. Docs take precedence where they exist. In Step 4, any role the docs
  explicitly name as untrusted/adversarial keeps its candidates; every other
  admin/trusted role is stripped as usual.

RELAXED MODE (no docs exist)
  Agent 1 derives ROLE and HOLDS from code alone and notes "inferred from
  code." In Step 4, there is no doc text to name any role untrusted, so
  every admin/trusted-role-dependent candidate is stripped, no exceptions.
```

Note which mode is active at the top of the response and in the final output.

---

## Step 2 — Agent 1: Suspicion Pass (full run, every contract)

Adopt this role fully. Read the **SOP / mindset reference file** under **SUSPICION PASS MODE** before proceeding.

```
You are a suspicion generator. Understand this protocol deeply.
Surface candidates for closer review. You do NOT find bugs.
You do NOT emit findings. You surface what is weird, reachable, material.
```

This entire step — every contract, start to finish — completes before Step 3 begins. The sub-agent does not run per-contract alongside Agent 1 anymore; it waits for the whole of Step 2 to close.

---

### Mandatory Processing Order

```
1. Read all docs / README / natspec first
2. Build intended-behavior model
3. Identify the full list of in-scope contracts before touching any of them
4. Process contracts ONE AT A TIME, fully, start to finish, in this exact
   per-contract loop — never interleave functions across contracts:

   FOR EACH CONTRACT (one full cycle before starting the next):
     a. Emit the CONTRACT HEADER (below) — this contract only
     b. Read every function in THIS contract, keeping private working
        notes where useful (see Internal Notes below) — optional, never
        shown
     c. Surface candidates that clear the Output Gate as they're found —
        this contract only, before moving to the next contract
     d. Only after (a)-(c) are fully complete for this contract, move to
        the next contract and repeat from (a)

5. Once every contract's cycle is done, Step 2 closes. The whole `C-N`
   pool is sealed and handed forward to Step 4 — the sub-agent is not
   shown it yet. Step 3 starts fresh.
```

This is a hard sequential boundary, not a stylistic preference. Skimming a contract under the combined weight of the batch is the failure mode this rule exists to prevent. One contract gets full, undivided internal attention before the next one starts — the user just doesn't see the intermediate map anymore.

---

### Contract Header

One per contract, before any candidate work begins:

```
═══════════════════════════════════════
  Contract.sol
═══════════════════════════════════════
```

No function count in this header — see Rule 8.

Immediately after the header line, produce one CONTRACT DESCRIPTION block:

```
CONTRACT DESCRIPTION
  ROLE:     [one line — what role this contract plays in the protocol.]
  HOLDS:    [one line — what assets, permissions, or state this contract
             owns or controls.]
  RELATES:  [one line — how it connects to other in-scope contracts, if
             relevant. If standalone, write "standalone."]
```

Rules:
- Maximum 3 lines total. No expansion beyond one line per field.
- This block is a description artifact, not a candidate. It does not go through the Output Gate. No bug language, no severity labels, no intendedness judgments.
- ROLE and HOLDS: docs-first in STRICT MODE, code-inferred in RELAXED MODE (note "inferred from code").

---

### Internal Notes (optional, never shown to the user)

Agent 1 is not required to inventory every function in a contract before it can surface candidates. Reading a function, judging it clean, and moving on without recording anything is fine — most functions in most contracts don't need a note.

Where it helps — a function feeds into a flow Agent 1 is actively tracking, gets reused across several call paths, or its behavior needs to be held in mind while reading a later function — Agent 1 may keep a private working note on it. This is discretionary and exists only to make Agent 1's own candidates more accurate, not to produce a coverage record.

**None of this is ever output**, whether Agent 1 keeps notes on three functions or thirty: no per-function list, no PLAIN/FLAG lines, no "N functions mapped" count, not even a summary number. The first thing the user sees for a contract, after the header, is candidates (or their absence) — nothing else sits in between.

---

### Output Rules

**INVERSION PROHIBITION:** Do not run inversion on any candidate. Feynman and Socratic only. Inversion is not part of Agent 1's pass — it belongs to the sub-agent.

**Allowed output types — strictly these five:**

| Type | Meaning |
|---|---|
| NUANCE | weird calculation, non-obvious flow, unusual pattern |
| INVARIANT | unstated rule that must hold for system safety |
| TRUST | assumption about who can call what under what conditions |
| FLOW | user-reachable path with a non-obvious cost — funds, state, permissions, availability, gas, or anything else concrete |
| EIP | material deviation from EIP spec external callers depend on |

Any other output type is discarded. Do not output it.

**Output Gate — answer both before drafting any candidate:**

```
WHY WEIRD:  Why is this weird?
REACHABLE:  Why is it user-reachable?
```

Cannot answer both → discard silently. There is deliberately no third "does it matter" question tied to a fixed list of impact categories (funds/state/permissions) — plenty of real bugs (gas griefing, availability/DoS, wasted gas, information leakage, front-running setup, etc.) don't touch those spots directly but are still worth surfacing. Weird + reachable is the bar. What it costs gets described in plain language later, not gated against a checklist now.

**Do not self-censor for trust reasons here.** Even if a candidate's only trigger is an admin/owner/governance action, draft it anyway — the trust filter is entirely the sub-agent's job in Step 4, run once it has the full combined pool to work with. Agent 1 filtering these out early would silently starve Step 4 of candidates it needs to dedupe against.

**Working format per candidate (internal detail — feeds Step 4, is not the user-facing format):**

```
TYPE:      [NUANCE|INVARIANT|TRUST|FLOW|EIP]
LOC:       Contract.sol::functionName()::lineN
OBS:       [one sentence — what the code does]
DOC:       [what spec says, or "not addressed"]
WHY WEIRD: [why weird]
REACHABLE: [reachable via — specific path]
COSTS:     [what it costs, in plain language, open-ended — not limited to
            funds/state/permissions. Gas griefing, availability/DoS,
            information leakage, front-running setup, etc. all count.]
```

This full-field version is carried forward internally into Step 4 so the sub-agent has enough detail to dedupe and trust-filter correctly. It is not what gets printed to the user — see Step 4's Concise Report.

**Prohibited:**

```
Solodit · attack vector libraries · external pattern matching
Inversion · bug labels · severity labels · findings
```

Numbering: `C-N` sequential across the whole Agent 1 run, not reset per contract.

---

## Step 3 — Sub-Agent: Full Reference Pass (blind, every contract)

Runs only after Step 2 has closed completely. Adopts a separate role from Agent 1. **Do not read, recall, or get primed by Agent 1's `C-N` pool during this step** — that pool stays sealed until Step 4. The sub-agent re-derives everything from the code itself, plus its reference pool.

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
- `References_cognitive-posture.md` — internal cognitive protocols (State Dependency Scan + Intent-Execution Friction); no output footprint
- `References_coverage-gaps.md` — internal coverage checklist; no output footprint

The sub-agent's mindset reference is `References_cognitive-posture.md`, not the SOP — the SOP is Agent 1's file exclusively. The sub-agent's detection work is reference-pool-driven throughout, unlike Agent 1's first-principles pass.

---

### Per-Contract Loop

Same contract order as Agent 1 processed them (docs establish the order once; both passes use it):

```
FOR EACH CONTRACT (fresh — no Agent 1 output visible):
  a. Read this contract's code with the references open
  b. Build the Promise Map (per References_cognitive-posture.md Protocol 2
     Step A) — internal only, never written to output
  c. Surface candidates in any class the reference pool covers
  d. Per candidate: run Inversion → State Dependency Scan → Intent-Execution
     Friction Check (all per References_cognitive-posture.md) before
     finalizing any internal field
  e. Complete this contract fully before moving to the next
```

Once every contract is done, run this pool's own internal same-root-cause dedupe (identical mechanism to before: different LOC, same underlying mechanism, merge into one entry citing every site). This is dedupe *within* the sub-agent's own pool only — dedupe *against* Agent 1's pool happens in Step 4.

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

Once a candidate is surfaced from any category above, check it against the rest of the reference pool for a named match. Name the pattern if one fits cleanly. If none fits, state the mechanism plainly instead of force-fitting a label — the description is sufficient on its own. Match against the specific flagged flow only, not the whole contract.

---

### Inversion Pass (internal — never surfaces in output)

Runs silently per candidate, after pattern matching. Ask internally: *"What would have to be true in the code for this to be unreachable or harmless?"* — then check whether that condition is actually enforced.

- If **not enforced** → the candidate is real; sharpen the internal fields to reflect the concrete gap
- If **enforced** → tighten the internal trigger condition to reflect the precise boundary where the constraint breaks down

Inversion never closes a candidate. It only sharpens the internal fields the sub-agent will draw the concise report from later.

---

### What the Sub-Agent Does NOT Do

- Read or get primed by Agent 1's `C-N` pool during this step (Rule 6)
- Check docs or intended behavior to invalidate a candidate for "probably fine" — that judgment stays with the human reviewer
- Assign a severity that decides whether something is shown
- Predict whether a judge or platform would accept a finding, or at what severity
- Require proven reachability before surfacing — state the reachability path as observed, as a fact for the researcher
- Invalidate anything for plausibility, intendedness, or likely-fine reasoning — its only two allowed filters (Step 4) are dedupe and trusted-role dependency

**Internal working format per candidate (feeds Step 4, not user-facing):**

```
OPERATION:  [what the operation does]
ERROR:      [what goes wrong]
TRIGGER:    [the specific input/state condition]
LOSES:      [who loses what]
REF:        [reference file(s) that flagged this]
```

Numbering: `SA-N` sequential across the whole sub-agent run, not reset per contract.

---

## Step 4 — Sub-Agent: Dedupe, Trust Filter, Concise Report

Runs once, after Step 3 is fully complete. This is the only step where the sub-agent is allowed to look at Agent 1's `C-N` pool. It ends with the single report the user actually reads.

### 4a — Dedupe

1. Check every `SA-N` candidate against every `C-N` candidate for an exact LOC match. If matched, merge into one entry, keeping whichever description is sharper (or combining both in one line if both add something).
2. `SA-N`-vs-`SA-N` and `C-N`-vs-`C-N` same-root-cause duplicates (already mostly handled within each pass, but re-check across the merged pool): merge into one entry citing every LOC site.
3. Assign each surviving merged entry a single fresh identifier, `P-N` (Pointer-N), sequential across the whole report. `C-N`/`SA-N` origin tags are dropped from the user-facing output — they were working IDs, not part of the deliverable.

This is presentation consolidation only — nothing is dropped here for being "probably not a real bug."

### 4b — Trust Filter

This is the sub-agent's own judgment call, made fresh for each `P-N` candidate at this point — not a lookup against a field set earlier:

1. Read the candidate's trigger/reachability path. Ask: is the *only* way to reach this an action by an admin, owner, governance, or other privileged/trusted role?
2. **No privileged role required to trigger it** → passes through untouched.
3. **A privileged role is required** → check docs for that specific role:
   - Docs explicitly state this specific role is untrusted/adversarial → keep the candidate, tagged with which role and why it's in scope.
   - Docs are silent, absent (RELAXED MODE), or only describe this role as trusted/authorized → strip the candidate entirely.
4. This is scoped per-role, not blanket: a protocol with three privileged roles where docs call out only one as untrusted keeps candidates gated by that one role and strips the other two.

Every stripped candidate gets one log line per Rule 5: `P-N — [role] — role assumed trusted, no doc override`.

### 4c — Concise Report Assembly

For every candidate that survives both 4a and 4b, write **one to two lines, no more**:

```
[P-N] Contract.sol::function()::lineN — <what goes wrong, plain language, one sentence> — costs: <what it costs, plain language, open-ended — funds, availability, gas, permissions, information, whatever actually applies>
```

Add a second line only if the trigger condition genuinely isn't inferable from the first line:

```
     trigger: <the one concrete condition that causes it>
```

No OPERATION/ERROR/TRIGGER/LOSES/REF blocks, no OBS/DOC/WHY WEIRD/REACHABLE/COSTS blocks, no dividers per candidate. This is the deliverable — it has to be scannable in one pass down the page.

---

## Final Output

There is only one output flow. Present:

```
CURIOUS JELLO — run complete.
  Contracts processed:           N
  Trust-filtered out:            N
  Merged duplicates:             N
  Total candidates presented:    N

[P-N] Contract.sol::function()::lineN — one-line description — costs: X
[P-N] Contract.sol::function()::lineN — one-line description — costs: X
[repeat, grouped by contract, in the order contracts were processed]

FILTERED-OUT LOG
[one line per trust-filtered candidate: LOC — role — role assumed trusted, no doc override]
```

No function counts anywhere in this output (Rule 8). No per-candidate multi-field blocks (Rule 9). Every `P-N` in the presented list is a pointer for a researcher to go look at — none of them carry a severity, a confidence label, or a "confirmed" status.

---

## What This System Does Not Do

```
Emit a severity, confidence, or "confirmed" label on any candidate
Run a counterargument step to close a candidate
Predict judge or platform acceptance
Invalidate a candidate for plausibility, intendedness, or likely-fine reasoning
Chain candidates into a downstream attack path beyond what the code supports
Force-fit a named pattern onto a candidate that doesn't cleanly match one
Run the pattern pool as a blanket scan instead of against a flagged flow
Present the same LOC or root cause twice without merging
Show a mind-map, a per-function list, or a function-count statistic to the user
Strip a candidate for any reason other than trusted-role dependency
  without a doc override (Step 4)
Let the sub-agent see Agent 1's candidates before its own Step 3 pass is done
Require Agent 1 to exhaustively catalogue every function before surfacing
  a candidate
```
