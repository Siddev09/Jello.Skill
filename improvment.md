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
