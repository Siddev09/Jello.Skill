# References_MathPass

**Purpose:** Extract and consolidate core mathematical concepts and invariants from a smart contract codebase *before* suspicion/reference passes. Generates one unified mathematical model per codebase to anchor targeted code tracing.

**Output:** One consolidated report of unique mathematical concepts used across the codebase, with concrete code examples and critical boundary constraints.

---

## MATH PASS PROTOCOL

### Overview

A Math Pass is a three-phase process that runs *independently* of suspicion or reference passes:

1. **Per-Contract Math Extraction** — Extract mathematical concepts and invariants from each in-scope contract individually
2. **Deduplication** — Merge identical concepts across contracts into unique entries
3. **Consolidated Report** — Present unified mathematical model with examples and critical questions

The Math Pass is **reference-free** in the sense that it derives concepts directly from code and documentation, not from pre-built bug patterns. Its output is purely conceptual—no candidates, no findings, no judgment.

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
- Skim all state variables and their types (what's being tracked?)
- Identify each function's mathematical role—is it computing, storing, converting, or validating?
- For critical paths (swap, deposit, burn), trace the math step-by-step: inputs → state changes → outputs

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
2. **Group by concept name** — if "Weight-Based Accounting" appears in 5 contracts, it's one unique concept
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

## PHASE 3: CONSOLIDATED REPORT FORMAT

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

**Audit Tracing Questions:**
- Q1: [Where would this invariant break?]
- Q2: [What edge case is risky?]
- Q3: [Can this be exploited?]

---

### 2. [Concept Name]
...

## Cross-Contract Correlations
[Note any data flow or state dependency issues between concepts]

## Boundary Risk Checklist
[Quick reference of all identified overflow/underflow/rounding/encoding risks across all concepts]
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

5. **Questions are investigative, not conclusive.** "Can this invariant be violated?" not "This invariant is always maintained."

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
Math Pass (independent, runs first)
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
3. **Phase 3 (reporting):** Builds summary table and final format; deterministic

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
- [ ] Questions generated (where would this break?)
