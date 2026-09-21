# References_MathPass

**Purpose:** Extract and consolidate core mathematical concepts and invariants from a smart contract codebase *before* suspicion/reference passes. Generates one unified mathematical model per codebase to anchor targeted code tracing.

**Output:** One consolidated report of unique mathematical concepts used across the codebase, with concrete code examples and critical boundary constraints.

---

## MATH PASS PROTOCOL

### Overview

A Math Pass is a four-phase process that runs *independently* of suspicion or reference passes:

1. **Per-Contract Math Extraction** — Extract mathematical concepts and invariants from each in-scope contract individually
2. **Deduplication** — Merge identical concepts across contracts into unique entries
3. **Precedent Finder** — For each deduplicated concept, find historical bug bounty/contest precedent of that math going wrong, or, if none exists, its specific high-weight pitfalls
4. **Consolidated Report** — Present unified mathematical model with examples, critical questions, and precedent/pitfall findings

The Math Pass is **reference-free** in the sense that it derives concepts directly from code and documentation, not from pre-built bug patterns. Its output is purely conceptual—no candidates, no findings, no judgment.

---

## DEPTH MANDATE — NO SHALLOW PASSES

**This overrides speed or brevity in every phase below. A Math Pass that moves fast at the cost of missing a concept, a precedent, or a boundary has failed, regardless of how polished the output looks.**

- **No phase is allowed to be shallow.** Skimming a contract, accepting the first search result as sufficient precedent, or writing a boundary entry without actually tracing the code path is a failure of the pass, not an acceptable shortcut.
- **Take the time the codebase demands.** A five-contract swap module and a fifty-contract lending protocol do not get the same effort. Scale extraction, search, and tracing depth to the actual size and complexity in front of you — never compress effort to fit a time budget.
- **Never gloss over a precedent.** If a search turns up a borderline or partial match, dig into it rather than discarding it on the first read. A precedent is only excluded after its triggering condition has actually been checked against this codebase (per Phase 3) — not because checking it seemed like extra work.
- **Never gloss over a pitfall.** When no precedent transfers, the fallback pitfalls must come from genuinely reasoning through how *this* mechanism fails — not from a generic mental checklist skimmed for something plausible-sounding.
- **Never gloss over a boundary.** Every boundary/cap/edge state must be traced through the actual code to confirm it is enforced, unenforced, or partially enforced — not asserted from a first impression of the contract's design.
- **Ask the right questions before concluding anything.** Before extraction is considered complete for a contract, before a concept is deduplicated, and before a precedent is accepted or rejected, stop and ask: What am I assuming here that I haven't verified in the code? What would make this wrong? Is there a related concept nearby I haven't captured yet? If those questions haven't been asked and answered, the step is not done.
- **Depth applies to every phase**, not just extraction: deduplication must confirm concepts are truly identical (not merged on a surface-level name match), the Precedent Finder must complete its full reachability check (see Phase 3) for every candidate, and the final report must not summarize away detail that a manual auditor would need.
- **When in doubt, go deeper, not shorter.** If a concept, precedent, or boundary could plausibly be split into two, or could plausibly hide a second issue underneath it, investigate further before finalizing the entry.

---

## PHASE 1: PER-CONTRACT MATH EXTRACTION

For each in-scope contract, identify and document:

### A. Core Mathematical Operations

**What to extract:**
- **Invariants:** State relationships that must hold (e.g., `x * y = L²` in AMMs, `sum(shares) = totalSupply` in vaults)
- **Directional encoding:** How sign, polarity, or type choice encodes semantic meaning (e.g., positive=input, negative=output)
- **State transitions:** Mathematical relationship between state before and after a function (e.g., `reserve_new = reserve_old + amount_in`)
- **Constraints:** Boundaries, limits, overflow/underflow assumptions (e.g., `amount ≤ int128.max`)

**How to extract:**
- Read contract's natspec and README first (understand stated intent)
- Read — do not skim — all state variables and their types (what's being tracked?); a skimmed variable is how concepts get missed
- Identify each function's mathematical role—is it computing, storing, converting, or validating?
- For critical paths (swap, deposit, burn), trace the math step-by-step: inputs → state changes → outputs
- Do not stop at the first pass over a contract if anything is unclear — re-read the function, trace callers/callees, and confirm the invariant actually holds in code before writing it down. Extraction is not done until every mathematically active function has been accounted for.

**Format per contract:**
```
## ContractName: [One-line role]

### Concept 1: [Math Name]
**Invariant:** [State relationship that must hold]
**Example from code:** [Code snippet showing this concept]
**Boundary/Constraint:** [Limits, overflow risks, type limitations]

### Concept 2: [Math Name]
...
```

**Example (from MetricOmmSwapInputs):**
```
## MetricOmmSwapInputs: Signed Integer Encoding

### Concept 1: Sign Convention for Swap Direction
**Invariant:** Positive int128 = exact-in (user specifies input), Negative int128 = exact-out (user specifies output)
**Example from code:**
asAmountSpecifiedIn(uint128 amountIn)  → int128(+amountIn)
asAmountSpecifiedOut(uint128 amountOut) → int128(-amountOut)
**Boundary/Constraint:** Amounts must fit in int128.max (2^127 - 1). Boundary check: `if (amount > MAX_INT128_AS_UINT128) revert`
```

### B. Cross-Contract Flows

If a contract calls another, note:
- **Data passed between contracts:** Are deltas encoded the same way? Are prices normalized?
- **State dependencies:** Does contract B assume contract A's state property (e.g., "pool invariant holds")?
- **Callback/reentrancy math:** Are transient states isolated or shared?

---

## PHASE 2: DEDUPLICATION

After extracting math from all contracts, consolidate:

1. **Scan all per-contract reports**
2. **Group by concept name** — if "Weight-Based Accounting" appears in 5 contracts, it's one unique concept. Confirm this by checking the actual invariant in each occurrence, not the name alone — a shared name with a different underlying invariant is not the same concept and must stay separate.
3. **Keep strongest example** — select the clearest code snippet that illustrates the concept
4. **Note all locations** — list which contracts use this concept
5. **Merge boundary constraints** — combine all edge cases into one comprehensive boundary description

**Output:** One master list of unique concepts, each with:
- Concept name
- Invariant statement (1–2 sentences)
- Concrete code example (smallest snippet that illustrates it)
- All locations where it's used
- Combined boundary/constraint checklist

---

## PHASE 3: PRECEDENT FINDER

*(Runs after Phase 2 Deduplication is fully complete, on the deduplicated concept list — before the Consolidated Report is written.)*

### Purpose

For each unique mathematical concept surviving deduplication, find real-world cases where *this exact mechanism* — the same math, not the same protocol lineage or the same overall design — was broken somewhere, under some specific triggering condition. Then check, concretely, whether that same triggering condition is reachable in *this* codebase. If it is reachable, that is the precedent to report. If no such mechanism-level break exists anywhere, fall back to the concept's own specific, weighty pitfalls.

### What this phase is NOT

This phase does not do lineage/resemblance reasoning. It is never correct to conclude:
- "This contract's math resembles Protocol X's design, and Protocol X handled it correctly, so this is fine."
- "This looks like a fork of Protocol X" as a stand-in for actually checking the mechanism.
- Citing a protocol merely because it uses a similarly-named concept, without identifying the specific condition that broke it and testing whether that condition exists here.

Resemblance between codebases is not precedent. A precedent is only valid when it identifies the *exact triggering condition* that broke the math (e.g. "totalSupply forced to exactly 1 wei via a rounding-down burn, then a large second deposit exploits the resulting price") and that condition is then checked against this specific contract's actual state machine — not assumed to transfer because the contracts look similar.

### Process (per concept)

1. **Isolate the mechanism, not the protocol.** State the concept's math in mechanism-only terms, stripped of any protocol name — e.g. "share price is derived from `totalSupply` and `reserve` with no floor enforced at zero" rather than "Uniswap-style AMM."
2. **Search for that mechanism breaking anywhere**, regardless of protocol identity or how different the surrounding design is — a lending vault, an AMM, a staking contract, a bridge, anything — as long as the same mathematical mechanism was the root cause. Search using the mechanism's specific trigger condition as the anchor (e.g. "totalSupply zero rounding first depositor exploit"), not the protocol's name or category. Check across:
   - Bug bounty disclosures (Immunefi, HackenProof, etc.)
   - Contest/audit findings (Code4rena, Sherlock, Cantina, Spearbit, Trail of Bits, etc.)
   - Post-mortems of exploited protocols, regardless of whether they're in the same product category as this codebase
3. **Extract the exact triggering condition** from that precedent — the specific state or sequence of calls that broke the invariant (not "rounding was bad" but "an attacker forced `totalSupply` to 1 via burn-to-near-zero, then front-ran a large deposit").
4. **Test that triggering condition against this codebase.** Trace through this contract's actual code and ask: can that same condition be reached here? Is the guard that (possibly) prevented it elsewhere present or absent here? Only report the precedent if the triggering condition is genuinely reachable in this contract — do not report a precedent whose triggering condition this codebase already forecloses. This tracing step must be done in full for every candidate precedent, not stopped at the first plausible-looking match — a shallow reachability check that assumes rather than confirms is not acceptable.
5. **If no mechanism-level precedent transfers** (either none exists, or every one found is blocked by a guard already present in this codebase): fall back to identifying the concept's own specific, weighty pitfalls — not generic risks (overflow, underflow, zero-amount checks, reentrancy-in-general) unless the concept's actual mechanism makes that specific instance unusually severe or non-obvious.
6. **Filter for weight:** a pitfall or precedent only qualifies if it would plausibly cause fund loss, invariant violation, or protocol insolvency — not code-quality or gas concerns. If nothing clears that bar, state that plainly rather than padding the entry.

### Output per concept (feeds into Phase 4 report)

```
**Precedent:** [Mechanism-level description of where this exact math broke + the specific triggering condition + confirmation that this codebase reaches that same condition, OR "No mechanism-level precedent transfers to this codebase"]
**Pitfall (if no precedent):** [1–2 specific, weighty pitfalls tied to this exact mechanism — omit if none clear the bar]
```

Keep this section short by design — one to three lines per concept, no elaboration, no restating the invariant already given in Phase 1/2, and never framed as "this looks like protocol X."

---

## PHASE 4: CONSOLIDATED REPORT FORMAT

The final Math Pass report has this structure:

```markdown
# [Protocol Name] — Math Pass

## Summary Table
| Concept | Used In | Risk Zone |
|---------|---------|-----------|
| Concept 1 | ContractA, ContractB, ContractC | Boundary X, Overflow in step Y |
| Concept 2 | ContractD, ContractE | Edge case: when Z |
| ... | ... | ... |

## Detailed Concepts

### 1. [Concept Name]
**Invariant:** [Concise state relationship]
**Code Example:**
\`\`\`solidity
[Minimal, real snippet from codebase]
\`\`\`
**Used In:** ContractA (line X), ContractB (line Y), ...
**Critical Constraints:**
- Constraint 1
- Constraint 2
- Constraint 3

**Precedent:** [Mechanism-level break + specific triggering condition + confirmation it's reachable in this codebase, OR "No mechanism-level precedent transfers to this codebase"]
**Pitfall (if no precedent):** [1–2 specific, weighty pitfalls — omit if none clear the bar]

---

### 2. [Concept Name]
...

## Cross-Contract Correlations
[Note any data flow or state dependency issues between concepts]

## Boundary Risk Checklist
[For each entry: name the actual enforced or economically meaningful boundary — not a generic type limit — and explain what happens if a value reaches, crosses, or sits exactly at that boundary. A boundary only belongs here if it is a cap, threshold, or state enforced by the protocol's governance/admin/config, or an economically real edge state (e.g. pool at first-LP/last-LP, totalSupply at 0 or at its practical max, a well-funded attacker deliberately pushing a value to that edge). Do not list bare type limits like int128.max or uint256 overflow as boundaries by themselves — only include them if a specific enforced cap, governance parameter, or economic edge case makes hitting that limit realistic and consequential.

Format per entry:
- **Boundary:** [The specific enforced/economic edge — e.g. "totalSupply drops to 0 after last LP exits", "pool reserve hits MIN_LIQUIDITY floor set by admin", "position size hits the governance-set max leverage cap"]
- **What happens at/beyond it:** [Concrete consequence — e.g. "next depositor becomes de facto first LP and can set an arbitrary initial share price", "swap silently reverts leaving funds stuck mid-route", "a well-funded attacker can single-block-donate to force this state deliberately"]
- **Reachable by:** [Normal usage / governance action / a well-funded attacker deliberately engineering it — state which]
]
```

---

## DEDUPLICATION RULES

**Identical Concepts (merge):**
- Same invariant, different function names → one entry, list all locations
- Example: "Weight-based accounting" used in bin tracking (5 contracts) → one entry with 5 locations

**Related but Distinct Concepts (separate):**
- Same operation (e.g., rounding), but different direction or context → keep separate
- Example: "Rounding down on withdrawal" vs. "Rounding up on deposit" → two entries

**Too Generic (discard or note as cross-cutting):**
- "Storage management," "Access control," "Type casting" → only include if it's mathematically significant
- Example: Keep "Transient storage bitpacking" (math-relevant), discard "onlyOwner checks" (not math)

---

## WHAT NOT TO EXTRACT

- **Implementation details:** Loop structure, intermediate variable names, gas optimization tricks
- **Access control:** Permission checks, role-based gates (unless they mathematically affect invariants)
- **Code quality:** Variable naming, natspec, comment quality
- **Gas optimizations:** Bitpacking for storage efficiency (unless the bitpacking itself is mathematically risky)

**Exception:** If an optimization *affects the math* (e.g., bitpacking causes overflow in bit-shift), include it with the boundary constraint.

---

## EXTRACTION WORKFLOW

For each contract:

```
1. Read README/spec first
2. List all state variables and their types
3. For each critical function (swap, deposit, transfer, etc.):
   a. Input: what types, what constraints?
   b. State change: what equation or transformation happens?
   c. Output: what's returned, what's assumed about it?
   d. Are there sign flips, type casts, or encoding schemes?
4. Identify invariants (relationships that must hold)
5. Identify boundaries (where math breaks)
6. Write concept entry
```

---

## DEDUPLICATION WORKFLOW

```
1. Collect all per-contract concept entries
2. Group by concept name and invariant
3. For each group:
   a. Select one strongest code example
   b. List all contracts using this concept
   c. Merge all boundary constraints into one checklist
4. Build summary table (concept → locations → risk zones)
5. Write final consolidated report
```

---

## CRITICAL PRINCIPLES

1. **Concepts are mathematical, not operational.** "Swap routes tokens" is operational. "Zeroforone is encoded as a bitmap bit" is mathematical.

2. **Every boundary constraint is a tracing anchor.** When you manually audit, you know exactly where to look first—the boundaries are where bugs hide.

3. **Deduplication happens last, not during extraction.** Extract all per-contract concepts first; only then dedupe. This prevents missing related concepts that appear in only one contract.

4. **Code examples are real, not synthetic.** Every snippet comes directly from the codebase, with line references.

5. **Boundaries are economic and enforced, not just type limits.** A boundary earns a place in the report only if it's a cap, threshold, or state that governance/admin enforces, or a real economic edge (first LP, last LP, totalSupply at 0 or max, a well-funded attacker forcing the edge) — not a bare int128/uint256 ceiling.

6. **Depth beats speed, every time.** Skipping the reachability trace on a precedent, merging concepts on name-similarity alone, or writing a boundary without confirming it in code are all shallow-pass failures — see DEPTH MANDATE. No phase is exempt.

---

## EXAMPLE: FULL MATH PASS FOR 5-CONTRACT SWAP MODULE

### Input Contracts
- MetricOmmSwapInputs
- MetricOmmSwapPath
- MetricOmmSwapQuoteDecode
- MetricOmmSwapResults
- TransientCallbackPool

### Per-Contract Extraction (abbreviated)
```
MetricOmmSwapInputs:
  - Concept: Signed direction encoding
  - Concept: int128 boundary enforcement

MetricOmmSwapPath:
  - Concept: Bitmap routing and token continuity
  - Concept: Price limit normalization
  - Concept: Max-hops enforcement (uint8.max + 1)

MetricOmmSwapQuoteDecode:
  - Concept: Revert-encoded delta extraction
  - Concept: Memory offset arithmetic
  - Concept: Selector validation

MetricOmmSwapResults:
  - Concept: Bidirectional delta semantics
  - Concept: Signed-to-unsigned conversion
  - Concept: Exact-in delta validation

TransientCallbackPool:
  - Concept: Sub-word bitpacking in transient storage
  - Concept: tradesLeft counter and recursion tracking
  - Concept: Multi-slot transient layout
```

### Deduplication
```
After dedup, unique concepts:
1. Signed direction encoding (appears in SwapInputs, SwapResults)
   → merged into one, list both locations

2. Bitmap routing and token continuity (SwapPath only)
   → kept as-is, single location

3. Integer boundary enforcement (SwapInputs, SwapPath, SwapResults)
   → merged concept with combined constraints

4. Revert-encoded result extraction (SwapQuoteDecode, SwapResults)
   → merged with notes on revert vs. delta shape differences

5. Transient storage bitpacking (TransientCallbackPool)
   → single location, detailed constraints on tradesLeft overflow

[Result: ~7–10 unique concepts, not 15]
```

### Consolidated Report
[See example output in conversation above]

---

## INTEGRATION WITH AUDITING SKILLS

This reference file is **independent** of suspicion/reference passes. It can be:

- **Attached to curious-jello or similar skills** as a pre-pass context layer
- **Run standalone** to build protocol understanding before any auditing
- **Reused across audit runs** — once a protocol's math is mapped, you don't re-extract it

### Workflow with curious-jello

```
Math Pass (independent, runs first — includes Precedent Finder after deduplication)
    ↓
Suspicion Pass (Agent 1, reference-free, first-principles)
    ↓
Reference Pass (Sub-agent, pattern-armed, blind to Agent 1)
    ↓
Manual Audit (you, armed with math understanding)
```

---

## AUTOMATION NOTES

When building an AI skill that executes this protocol:

1. **Phase 1 (per-contract extraction):** Can be parallelized across contracts, each call reads one contract + README + identifies concepts
2. **Phase 2 (deduplication):** Must run after all Phase 1 outputs are available; requires comparison logic (invariant matching, example selection)
3. **Phase 3 (Precedent Finder):** Must run after Phase 2 output is finalized, one lookup per unique concept; requires web/search access. Precedent search should target the concept's mechanism and its specific triggering condition, never the protocol's name, category, or apparent lineage — matching on "this contract resembles a protocol that handled things correctly" is an invalid basis and must be rejected even if found. Every candidate precedent must pass a reachability check: extract the exact condition that broke the mechanism elsewhere, then verify that condition is actually reachable in this codebase before citing it. Fallback-to-pitfalls logic triggers when no precedent's triggering condition transfers, and pitfall output must be filtered for severity — no generic overflow/underflow/zero-check entries unless the mechanism makes that specific instance unusually severe.
4. **Phase 4 (reporting):** Builds summary table and final format, now including the Precedent/Pitfall line per concept; deterministic

**Output deduplication heuristic:**
- If two concepts share ≥70% invariant text overlap → likely duplicate, merge
- If they share operation name but different invariants → likely distinct, keep separate
- Always flag borderline cases for human review

---

## REFERENCE CHECKLIST FOR EXTRACTION

Use this as a per-contract checklist to ensure comprehensive extraction:

- [ ] README/spec read first
- [ ] All state variables identified with types
- [ ] Core invariants listed (equations, relationships)
- [ ] Directional encoding schemes identified (sign, enum, bitmap, etc.)
- [ ] Type boundaries noted (int128.max, uint256 overflow, etc.)
- [ ] Cross-contract data flows noted (what leaves this contract, what enters)
- [ ] Critical function math traced (deposit, swap, burn, etc.)
- [ ] Boundary edge cases listed (zero amount, max amount, overflow scenarios)
- [ ] Code examples selected (real, minimal, line-referenced)
- [ ] Enforced/economic boundaries identified (governance caps, first/last LP, totalSupply 0/max, well-funded-attacker-reachable edges — not bare type limits)
