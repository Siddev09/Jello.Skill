Real root cause, more fundamental than "check market history harder": I broke the skill's own separation of concerns. While wearing the Agent 1 hat, I reasoned "seems like an intentional design choice" and used that to *not* surface the 99% constant as a candidate at all. That's Agent 2's job (Gate 2 / Gate 5.5 / counterargument.md), not Agent 1's. Agent 1's mandate is W/R/M only — weird, reachable, material — never "is this probably fine." I silently pre-judged intendedness and killed a candidate before it ever reached the Output Gate. No gate caught this because the candidate never existed.

Three concrete fixes, in priority order:

**1. Hard rule in SKILL.md / senior-auditor-sop.md (AGENT 1 MODE) — close the actual hole:**

```
AGENT 1 PROHIBITION — INTENDEDNESS JUDGMENT
Agent 1 may never use "this looks intentional / reasonable design / standard
practice" as a reason to skip the Output Gate or withhold a candidate.
Intendedness is exclusively Gate 2 / Gate 5.5 territory (Agent 2). If W/R/M
are all answerable, the candidate is surfaced — full stop, no matter how
plausible the design looks. Self-judging design intent in Agent 1 is itself
a discard, and it must never happen silently.
```

This alone would have caught H-01 — the 99% constant trivially clears W ("hardcoded tolerance, no adjustment mechanism"), R ("every withdrawCollateral call"), M ("funds — can permanently block withdrawal"). It only got killed by me skipping straight to "intentional," which the rule above now forbids.

**2. Mind-Map constant check — catch the *class* of bug, not just this instance:**

```
HARDCODED CONSTANT SUB-CHECK (runs as part of FLAG, every function touching
a numeric threshold/ratio/slippage/price constant):
State the real-world condition that breaches this constant. Do not assess
plausibility — assessing plausibility is Agent 2's job. The mere existence
of a breach condition is enough to FLAG, regardless of how unlikely Agent 1
thinks it is.
```

**3. Cross-link before Output Gate — I also split one root cause into two weak nuances** (the `pricePerShare()=1e18` hardcode and the `99_00` slippage check are the same peg assumption surfacing twice):

```
CORRELATION CHECK (runs once, after full mind-map, before candidate output):
Scan all FLAGged entries in this contract + parent contracts for shared
numeric/market assumptions. If two or more flags rest on the same bedrock
assumption, emit ONE candidate (INVARIANT type) citing all LOC sites, not
N separate weak NUANCEs. A merged candidate with two confirming sites is
harder to wave away at Gate 5.5 than two isolated ones.
```

**4. Tighten counterargument.md's "Economic Assumption" template** so Agent 2 doesn't just inherit the same blind spot on the other side of the pipeline:

```
ECONOMIC ASSUMPTION — refutation now requires evidence, not plausibility:
Before accepting this defense, check whether the assumed condition (peg
holding, price stability, liquidity depth) has a documented historical
breach for this specific asset or asset class. If uncertain, search for it
before ruling — do not accept "this is a standard DeFi assumption" on priors
alone. A known precedent (stETH ~0.98 May 2022, USDC ~0.88 Mar 2023, UST
collapse 2022, etc.) refutes the defense outright.
```

Want me to apply these four edits to the actual files (`SKILL.md`, `References_senior-auditor-sop_pashov_updated.md`, `References_CounterArgument.md`) now?


---

Gaps — things in your original idea not yet built

Fuzzing module integration / invariant testing. Your original notes mention this twice ("FUZZING MODULES INTEGRATIONS — INVARIANTS BASICALLY"). The SKILL.md lists mcp__sc-auditor__run-echidna, run-medusa, run-halmos as allowed tools for Agent 2, but there's no instruction telling Agent 2 when or how to invoke them. Right now they're available but orphaned — no gate references them. This is the single largest functional gap.
"Explanation of contract code each function in depth within 3 lines" as a structured Agent 1 output. Your idea wanted a per-function 3-line explainer pass across the whole contract — closer to a systematic mind-map than candidate-by-candidate surfacing. What got built is candidate-only output (Agent 1 only writes when something is weird). The deep mind-map / full-coverage explanation step doesn't exist as a separate artifact — it happens implicitly during Feynman but is never written down. You lose the side benefit of having a full function-by-function reference map for your own manual review.
Math/numerical module as a distinct pass. You asked for this explicitly ("Feynman/Socratic/math/numerical/economic/periphery module from pashov skill"). Gate 5 covers economic impact as a single check, but there's no dedicated numerical-precision pass (rounding direction, decimals mismatch, overflow bounds) the way CCA's VC5/VC6 vectors do it for that specific protocol. You have it for CCA-shaped protocols only, not as a general-purpose module Agent 2 always runs.
QuillShield-specific references. Mentioned in your "what I need to add" list. Never supplied, never built. attack-patterns.md currently has: Hook vulnerabilities, CCA vectors, Approval Abuse, Callback Grief, Periphery Agent. No QuillShield content exists yet — open placeholder.
Shared-rules.md is still an empty placeholder. Referenced throughout SKILL.md (references/shared-rules.md) for both agents but you've never actually supplied content for it. Right now Agent 1 and Agent 2 both point at a file that doesn't exist yet — the output-format binding currently lives duplicated inline in SKILL.md itself instead of being centralized there.
Token-saving compression pass not applied to the actual SKILL.md. You wrote a detailed plan for Haiku-token compression (numeric verdicts instead of prose, collapsed reference tables, y/n gates). The current SKILL.md is still in fairly verbose directive-prose form, not the compressed [ ] Reachable [ ] Not in docs... checklist style you specified as the target format. This was discussed but never executed as an edit to the actual file.


Net assessment
Structurally and philosophically you're close to fully aligned — the destruction-first asymmetry, the gate ordering, the counterargument strength rule, and the no-amplification boundary are all faithfully built.
The gaps are concentrated in three areas: (1) fuzzing/invariant tooling is present but not wired into any gate, (2) the full-contract function-by-function mind-map output was dropped in favor of candidate-only output, and (3) the token-compression pass you designed was never actually applied to the live SKILL.md — it's still prose-weight, not checklist-weight.
If you want to close these before testing, the fuzzing integration and the compression pass are the two with the highest practical impact on your stated goals (false positive reduction and Haiku cost).



---

Stripped of the contest specifics, there's one defect: **Agent 1's burden of proof is inverted.** It should default to surfacing and discard only on mechanical proof; in practice I let it default to discarding on vibes ("looks intentional") and surface only on suspicion. That single inversion is what ate H-01, and it'll eat the next unrelated bug class too, because it's not a slippage-constant problem — it's a judgment-leak problem. Four structural fixes, all generalized:

**1. Ban plausibility-language at Agent 1, full stop.**
```
AGENT 1 — BURDEN OF PROOF
Default action is SURFACE. Discard only by mechanical failure of W, R, or M
using code-level facts — never by judgment of intent or design quality.
Any internal reasoning containing "intentional," "reasonable," "standard
practice," "probably fine," or "by design" is Agent 2 vocabulary. If that
vocabulary appears anywhere in Agent 1's reasoning, the candidate must be
surfaced regardless, and the vocabulary discarded instead.
```
This isn't about slippage constants. It's the rule that would have caught the miss regardless of what shape the bug took.

**2. Separate inventory from judgment — two literal sub-passes, not one.**
Right now Mind-Map does PLAIN-explain-then-FLAG in one motion, per function. That's where judgment leaks into the inventory step. Split it:
```
PASS A — MECHANICAL INVENTORY (zero commentary, zero "is this a problem")
List every: hardcoded numeric constant, every external call, every
unchecked return value, every return/require/revert ordering, every
access-control boundary. Pure enumeration.

PASS B — JUDGMENT (scoped to R and M only)
Apply reachability/materiality to Pass A's list. Weirdness (W) is never
judged here — anything mechanically enumerated in Pass A already clears W
by construction (it's an assumption baked into code about something that
can vary). Judgment is restricted to "does anything reach it" and "does it
matter," never "is it likely to be wrong."
```
This generalizes past constants — it covers the same leak in any category (unchecked calls, control-flow ordering, trust boundaries).

**3. Kill silent "none noted." Force a visible reason.**
The actual moment the bug disappeared was invisible — I wrote "seems intentional" in my own head, and the mind-map just said nothing. A human reviewer (you) had zero surface to catch that, because there was no surface.
```
"None noted" is never a bare verdict for any item produced by Pass A.
It must carry a one-clause reason, printed in the mind-map output:
"FLAG: none noted — [reason]." If the reason is a judgment call rather
than a code fact, that's now visible and reviewable by a human, instead
of disappearing inside the model's reasoning.
```
This doesn't rely on me not slipping again — it assumes I will, and makes the slip auditable instead of silent. Cheaper and more reliable than "try harder."

**4. Correlate flags across the whole codebase, not per-function.**
A weak signal in isolation is often a strong signal once you see it repeat. This generalizes past "two constants encode the same peg assumption" — it's "the same shape recurring anywhere is evidence of a systemic gap, not two unrelated dust-level notes."
```
After Pass A+B for all in-scope contracts: cluster flagged items sharing a
root assumption (same constant, same unchecked pattern, same trust
boundary logic) across functions AND contracts. Merge into one candidate
per cluster, citing every site. A clustered candidate is the unit Agent 2
evaluates — never N isolated weak ones.
```

The umbrella point underneath all four: neither agent should ever settle a question by what looks plausible. Agent 2 already runs on "default invalid, prove it" against code+docs facts. Agent 1 needs the mirror-image discipline — "default surface, disprove it" against code facts — and right now it doesn't have that discipline encoded anywhere, it only has it as something I'm supposed to remember to do.

Want these four written into `SKILL.md` and `senior-auditor-sop.md` now?
