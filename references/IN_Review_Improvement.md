# Coverage Gaps — External Assumptions, EIP Differentials, Non-Theft Bugs, Deeper Socratic, Correlation

**Used by:** Agent 1 (Step 2 mind-map / candidate pass) + Agent 2 (Gate 5 impact framing, Gate 7 Socratic destruction).
**Tier:** Same as `References_senior-auditor-sop_pashov_updated.md` — Agent 1 reads this directly. Not sub-agent territory.
**Purpose:** Six surfacing gaps were found in live runs. Nothing was being skipped because a gate correctly killed it — it was being skipped because nothing asked the right question first. This file adds those questions. No gate is loosened. Default-discard posture is unchanged.

Read both this file and the SOP. They do not overlap — SOP gives HOW to reason, this file gives WHAT to ask.

---

## Agent 1 — what changes

Four new sub-checks run as part of the FLAG pass. Two new correlation steps run at the end of each contract's cycle. None of them produce verdicts — they surface facts and candidates only, same as everything else Agent 1 does.

```
Per function (during FLAG pass):
  §1  External Assumption Sub-Check  — every function calling out-of-scope
  §2  EIP Differential Sub-Check     — every function claiming a named standard
  §3  Mechanism-Blocking Sub-Check   — every function containing a protective guard

End of contract cycle (after candidates, before next contract header):
  §4  Intra-Contract Correlation     — shared assumptions across this contract's functions
  §5  Cross-Contract Register        — shared facts across all processed contracts
```

Agent 2 reads §3's Drill 4 and §6's Drill 5 at Gate 7 only.

---

## §1 — External Assumption Sub-Check

**When:** any function calling an out-of-scope contract, oracle, token, router, bridge, or user-supplied address — including interface calls like `IERC20(token).transferFrom(...)` where `token` is caller-supplied.

**What to output:**

```
CALLS:      [name the dependency — e.g. "Chainlink AggregatorV3", "arbitrary ERC20", "user-supplied receiver"]
ASSUMES:    [the fact the code needs to be true — stated as a fact, not a risk.
             e.g. "price is fresh and non-zero", "transfer reverts on failure",
             "receiver callback does not re-enter before state write"]
VALIDATED:  [yes/no — does the code check this fact? if yes, name the check.
             do NOT assess sufficiency — that is Gate 2/5.5, not Agent 1.]
```

**What happens next:** produces no verdict. If VALIDATED is no and R + M clear the Output Gate → surface as TYPE: TRUST. The trusted party is a contract, not a role — same shape, same type.

**Boundary:** Agent 1 does not evaluate the third party's actual behavior or speculate about what it might do. Name the assumption, mark whether it's checked, stop.

---

## §2 — EIP Differential Sub-Check

**When:** docs explicitly state EIP compliance is required — README, natspec, or scope definition names a standard (e.g. "ERC-4626 compliant", "implements EIP-1155"). Interface imports or function names alone do NOT trigger this. Docs must say it.

**What to do:** read the code against the named standard spontaneously — no checklist. If the implementation diverges from how the standard says it should behave, flag it in 2 lines and move on.

```
EIP DIFF: [standard] — [what the code does] vs [what the spec requires]
```

Surface as TYPE: EIP. Human reviews and decides validity. Do not assess severity, do not judge intent, do not go deeper.

---

## §3 — Non-Theft Bug Classes + Mechanism-Blocking Sub-Check

**The gap:** Agent 1's surfacing instinct skews toward value extraction. Bugs whose only effect is making the mechanism not work — no attacker profit — get missed. Gate 5's non-fund path already handles them; nothing was prompting Agent 1 to look.

**Taxonomy — surface these the same as any fund-touching candidate:**

```
DOS          — legitimate actor permanently or temporarily blocked from an entitled action.
GRIEF        — attacker degrades other users at own cost, no profit.
LOGIC-TIMING — correct under normal call order; a specific sequence or timing flips it.
EDGE-LOGIC   — correct across typical range; breaks at zero, max, first, last, empty, exact-threshold.
```

No dollar number required. State exact condition + exact state/availability impact. Use `state_transition` for `MATTERS` field if funds framing doesn't fit — do not invent a fund number.

**Mechanism-Blocking Sub-Check** — highest-value member of this class:

```
When: any function containing a protective guard (pause, circuit breaker, slippage cap,
deadline, rate limit, cooldown, health-factor floor, oracle-staleness revert).

Ask: "Does this check ever block the mechanism from doing its job when it's needed most?"

State: under what system condition does the event this check protects against ALSO
disable the check's gate — making the protective action unreachable at the exact
moment it's needed?

If that condition can be named → W and R clear by construction. Surface as candidate.
Do not judge likelihood — that is Gate 5's likelihood field and Gate 5.5, not Agent 1.
```

Three shapes to calibrate (not a library — do not pattern-match, see §5 Drill 4):
- Pause that blocks liquidations during the crash that triggered the pause.
- Oracle-staleness revert that blocks the emergency-exit path during the outage.
- Slippage cap that blocks withdrawal during the volatility event the user needs to exit in.

---

## §4 — Intra-Contract Correlation (end of contract cycle)

**When:** after candidates pass is complete for this contract, before moving to the next.

**What to ask:** "Of the assumptions I used to raise a candidate here, does any other function in this same contract share the same assumption?"

```
Rules:
- Looks backward only — this contract's completed mind-map. Never forward.
- Never re-opens a candidate already decided by the Output Gate.
- Asks whether the same INPUT ASSUMPTION recurs — not whether one candidate's
  EFFECT propagates. Effect tracing is chaining. Prohibited exactly as before.

If shared assumption found:
  Emit ONE candidate, TYPE: INVARIANT, citing every LOC that shares it.
  WHY WEIRD: "this assumption recurs at N sites."
  MATTERS: combined surface across all cited sites.
  Do not emit N separate weak candidates.

If no shared assumption:
  Write: "Correlation: none — [Contract.sol]"
  This line is mandatory even when the answer is no — proves the check ran.
```

---

## §5 — Cross-Contract Register (persistent across run)

**Purpose:** stops shared facts from going unseen when moving between contracts. The per-contract isolation that prevents skimming had a side effect — facts from Contract A stopped existing for Contract B.

**Mechanics:** one register entry appended at the end of each contract's full cycle (after §4, before next contract header). Never reset. Never rewritten. Strictly additive.

```
REGISTER — [Contract.sol]
  DEPS:      [out-of-scope contracts/oracles/tokens called — names only]
  ROLES:     [privileged roles and what they control — one line]
  CONSTANTS: [numeric constants/thresholds that could recur in siblings — or "none"]
  WIRING:    [in-scope contracts this one calls or is called by]
```

Before each new contract header, read all prior register entries and ask: "Does this new contract share a dependency, trust boundary, or constant with any registered contract?"

```
YES → surface candidate, TYPE: INVARIANT or TRUST.
      State: same assumption held in both places, cite both LOC sites.
      Do NOT ask "if A's bug fires, what happens in B" — that is chaining.
      Ask only "is the same unstated fact relied on in two places."

NO  → no confirmation line needed — the register entry itself proves the check ran.
```

---

## §6 — Deeper Socratic Drills (extends SOP §2)

Two drills that extend `References_senior-auditor-sop_pashov_updated.md` §2. Same mode boundaries: Agent 1 surfaces-and-stops, Agent 2 exhausts-and-decides. Neither drill licenses chaining or amplification.

**Drill 4 — Mechanism Availability**
```
Question: "Does this check ever block the mechanism from doing its job when it's needed most?"

Agent 1 (FLAG pass): name one concrete blocking condition per guard, surface as candidate, stop.
                     Do not enumerate every guard searching for more.

Agent 2 (Gate 7):   construct the blocking condition from code + docs facts.
                    If constructable and no override/escape path exists → CONFIRM.
                    If override exists and is reachable in time → discard, cite override.
```

**Drill 5 — Transaction Insertion**
```
Question: "Can an unrelated transaction land between any two steps here and change
           what the next step sees?"

Apply to: any function with two or more sequential steps — two reads, two external
          calls, check-then-effect, snapshot-then-use.

Ask BEFORE reaching for a pattern name. You do not need to label it reentrancy,
front-running, or sandwich — the insertion point is sufficient with or without a label.

Agent 1 (FLAG pass): find ONE step pair where any transaction can land between them
                     and change what the second step sees without re-validation.
                     Name what changes and what reads it stale.
                     TYPE: FLOW or INVARIANT, whichever fits. Stop after one.

Agent 2 (Gate 7):   exhaust the realistic inserted-transaction space for THIS step
                    pair only — deposit, withdrawal, price update, same-function
                    re-entry, different user. Stay confined to this insertion point.
                    Widening to unrelated functions = amplification = prohibited.
```

---

## Integration — where each section slots into SKILL.md

```
§1 External Assumption   → Step 2, FLAG pass, alongside hardcoded-constant check
§2 EIP Differential      → Step 2, FLAG pass, feeds existing EIP candidate type
§3 Non-Theft + Blocking  → Step 2, FLAG pass, surfacing instinct + MATTERS framing
§4 Intra-Contract Corr.  → Step 2, between (c) candidates and (d) next-contract move
§5 Cross-Contract Reg.   → Step 2, end of contract cycle + before next contract header
§6 Drill 4 / Drill 5     → Step 2 Agent 1 surfacing + Gate 7 Agent 2 destruction
```

This file belongs in the Reference Files table in SKILL.md alongside the SOP file — same tier, read by Agent 1 directly and by Agent 2 at Gate 7. Not an attack-pattern file. Not a sub-agent file.
