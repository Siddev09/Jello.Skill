# Coverage Gaps — External Assumptions, EIP Differentials, Non-Theft Bugs, Deeper Socratic, Correlation

**Used by:** Agent 1 (Step 2 mind-map / candidate pass) + Agent 2 (Gate 5 impact framing, Gate 7 Socratic destruction). Not a sub-agent file — Agent 1 reads this directly, same tier as the SOP reference, not the reference-free-by-design sub-agent pool.

**Why this file exists.** Six gaps were identified in live runs, all sharing one shape: something true and surfaceable was being skipped, not because a gate killed it correctly, but because nothing in the pipeline ever asked the question that would have raised it. This file is new question-asking and fact-recording machinery — it does not loosen any gate, does not change the default-discard posture, and does not authorize chaining, amplification, or speculation anywhere they were previously prohibited. Every mechanism below is contained to the same boundaries the rest of the skill already enforces; each section says so explicitly where it matters.

This file does not replace anything in `References_senior-auditor-sop_pashov_updated.md` — it extends it. Read both.

---

## 1. External / Third-Party Integration Assumption Declaration

**The gap:** Agent 1 routinely assumes facts about out-of-scope dependencies — oracles, arbitrary ERC20s, routers, bridges, other protocols' contracts — without ever writing the assumption down. The assumption disappears into "this looks like a normal integration" the same way intendedness judgments used to disappear into "this looks intentional." No surface, no candidate, nothing for a human to manually re-check against an adversarial condition.

**The fix:** every time a function calls something outside the audited scope, state the assumption as a fact, separately from any verdict on whether that fact is safe to assume. The point of writing it down is not for Agent 1 to judge it — it's so the human reviewer can look at one line and immediately think "what if that's false?" without having to re-derive it from the code.

```
EXTERNAL ASSUMPTION SUB-CHECK (runs as part of FLAG, every function that
calls an out-of-scope contract, oracle, arbitrary token, router, bridge,
or any address not under audit — including calls reached only through an
interface type, e.g. `IERC20(token).transferFrom(...)` where `token` is
attacker- or user-supplied):

CALLS:      [name the out-of-scope dependency — e.g. "Chainlink
             AggregatorV3Interface", "arbitrary ERC20 via transferFrom",
             "external DEX router", "user-supplied receiver contract"]
ASSUMES:    [the fact this code needs to be true for the function to
             behave as written — stated as a fact, not a risk rating.
             e.g. "assumes the returned price is fresh and non-zero",
             "assumes token.transfer reverts on failure rather than
             returning false", "assumes the router executes the full
             swap atomically and the reported amountOut is what was
             actually received", "assumes the receiver's callback does
             not re-enter before this function's state write"]
VALIDATED:  [does the code itself check this fact, or is it trusted on
             the call result alone? State yes/no and, if yes, name the
             check. Do NOT assess whether the check is *sufficient* —
             that judgment belongs to Agent 2 (Gate 2/5.5). This field
             is a fact, not a verdict.]
```

This produces no verdict by itself — list the fact, mark whether it's checked, stop. An unstated assumption about an external dependency clears **W** by construction, the same way an unexplained hardcoded constant does — the mere existence of an unverified fact is the weirdness, regardless of how standard the integration looks. If R and M also clear, surface the candidate as **TYPE: TRUST** — the trusted party here is a contract instead of a role, but the shape of the finding (relying on something you don't control without verifying it) is identical.

**Boundary:** this sub-check does not ask Agent 1 to evaluate the third party's actual behavior, look up its source code, or speculate about what it might do. It only asks Agent 1 to notice and name what is being assumed about something outside scope. Confirming whether the assumption is realistic is Gate 2/3/5.5 territory in Agent 2, and is explicitly the part the human reviewer is invited to keep manually thinking about even after a candidate is discarded — an `ASSUMES` line that survives only because Agent 2 accepted a "standard practice" defense is exactly the kind of discard worth a second human look.

---

## 2. EIP Compliance Differential Check

**The gap:** `EIP` is already an allowed Agent 1 candidate type ("material deviation from EIP spec external callers depend on"), but nothing tells Agent 1 what to actually diff against. In practice this collapses into "does it look roughly like an ERC20" instead of a real spec differential, so deviations that matter to integrators get missed.

**The trigger:** run this whenever docs, natspec, an interface import, or a function/variable name (`IERC4626`, `_safeMint`, `onERC1155Received`) claims or implies compliance with a named standard.

```
EIP DIFFERENTIAL SUB-CHECK:

1. Name the exact standard claimed (e.g. "ERC-4626", "ERC-1155").
2. Pull only the MUST / MUST NOT / SHOULD requirements relevant to the
   functions actually present in scope — not the whole spec text, just
   the surface this contract implements or calls into.
3. For each relevant requirement, mark PASS or DEVIATES mechanically,
   off the code as written. No plausibility judgment at this step.
4. Any DEVIATES is a candidate, TYPE: EIP — regardless of how minor it
   looks, and regardless of whether the deviation appears deliberate.
   "It's just a rounding direction" or "that's probably intentional" is
   Agent 2 vocabulary (see senior-auditor-sop §AGENT 1 PROHIBITION) —
   surface every deviation found, full stop.
5. A deviation that exists because the contract *deliberately* restricts
   or extends the standard (e.g. a non-transferable token overriding
   `transfer`) is still surfaced as a candidate. Whether the deliberate
   restriction is itself safe, documented, and non-breaking to external
   callers is Gate 2/5.5's job, not Agent 1's.
```

**Differential checkpoints by standard** (not exhaustive — extend per-protocol as new standards come up; the procedure above is general-purpose):

**ERC-1155**
- `ids`/`values` array length mismatch in batch functions MUST revert, not silently truncate or zip-shortest.
- Effects-before-interaction ordering: state (balances) MUST be updated *before* the `onERC1155Received`/`onERC1155BatchReceived` hook call, not after — reverse ordering is a reentrancy window (cf. Revest Finance, Siren Markets, Hypercerts).
- If a custom receiver-hook caller-authentication path exists, the hook MUST verify the actual caller is the token contract itself, not trust the hook's own `operator`/`from` parameters as proof of origin (cf. Tsuru).
- `balanceOf` for a non-existent token id MUST return `0`, not revert.
- Zero-value transfers MUST still emit the `TransferSingle`/`TransferBatch` event — silently skipping zero-value legs breaks off-chain accounting that listens for every leg.
- Transfer to the zero address MUST revert (this is the standard's burn-prevention rule) — distinct from an intentional `burn()` function, which uses its own explicit path, not `safeTransferFrom` to `address(0)`.

**ERC-4626**
- `previewDeposit`/`previewMint`/`previewWithdraw`/`previewRedeem` MUST NOT revert except due to integer overflow on the inputs, and MUST fully account for any fees the real call would charge — a preview that ignores a fee the real function applies is a spec violation, not just a UX nit.
- `totalAssets()` MUST NOT revert under any normal condition external integrators would rely on.
- `convertToShares`/`convertToAssets` MUST be idealized (no slippage, no fees) — if these functions silently include fee or slippage logic that the preview functions don't, that's an internal inconsistency, not just a documentation gap.
- Inflation/donation attack surface: check whether a first-depositor or low-total-supply state can be sandwiched into giving away a disproportionate share — absence of virtual shares/offset (OZ-style) or a minimum-deposit floor is the deviation to name, not yet the verdict on exploitability.
- Rounding direction MUST consistently favor the vault, not the user, on every conversion (round down shares owed on deposit/mint, round up shares burned on withdraw/redeem) — a reversed direction on even one path is a systemic drain vector, not a rounding nit (cf. Resupply, ~$10M, June 2025 — a downstream LTV/exchange-rate mutation, not a direct rounding bug, but the same family of "wrong direction compounds" deviation).
- `maxDeposit`/`maxMint`/`maxWithdraw`/`maxRedeem` MUST report real, currently-enforced bounds — returning `type(uint256).max` when an actual cap exists elsewhere in the contract is a deviation that breaks every integrator who trusts the `max*` functions to avoid a revert.
- Accounting that infers a deposited/withdrawn amount from a balance delta instead of using the explicit `assets`/`shares` parameter is a deviation surface in itself — fee-on-transfer tokens or any external transfer into the vault between read and write desyncs the inferred amount from the actual parameter.

**ERC-20 (staples worth a standing check even without an explicit EIP claim, since "ERC20" is assumed almost everywhere)**
- Token decimals other than 18.
- Fee-on-transfer / rebasing balance behavior.
- `transfer`/`transferFrom` that return no boolean, or return `false` instead of reverting on failure (older / non-standard tokens) — code that checks `success` via `require(token.transfer(...))` silently passes on a `false`-returning token if the return value isn't actually decoded and checked.
- Non-zero-to-non-zero `approve` revert (USDT-style) breaking any code path that re-approves without resetting to zero first.
- Blacklist-capable tokens (USDC/USDT-style) where a third party can brick a specific address mid-flow.

**ERC-721**
- `safeTransferFrom`'s `onERC721Received` hook fires before any caller-side state write that assumes the transfer is "done" — same effects-before-interaction class as ERC-1155 above.
- Approval state not cleared on transfer, allowing a stale approval to be reused against the new owner if the contract doesn't explicitly clear it.

---

## 3. Non-Theft Bug Classes — DoS, Grief, Logic-Timing, Edge-Logic

**The gap:** Agent 1's `MATTERS` framing (funds / state / permissions / state_transition) and the instinct that drives candidate surfacing both skew toward "can value be extracted." Bugs whose entire effect is *making the mechanism not work*, with no attacker profit at all, get under-surfaced — not because a gate rejects them (Gate 5 already accepts "exact state corruption or permission escalation" for non-fund impact, and the judging reference already has a DoS/freeze severity ladder), but because nothing actively prompts Agent 1 to go look for this shape of bug in the first place.

```
NON-THEFT BUG TAXONOMY — surface these the same as any fund-touching
candidate. No dollar number is required or expected for any of these;
state the exact condition and the exact state/availability impact
instead, same as Gate 5's existing non-fund path.

DOS          — a legitimate actor permanently or temporarily cannot
               complete an action they are entitled to.
GRIEF        — an attacker spends their own resources (gas, a small
               deposit, one transaction) to degrade the experience or
               cost of OTHER users, with no profit motive of their own.
LOGIC-TIMING — the code is correct under the "normal" call order or
               block timing, but a specific sequencing, block-boundary,
               or elapsed-time condition flips the outcome.
EDGE-LOGIC   — the code is correct across the typical input range but
               breaks at a boundary value: zero, exactly-equal,
               first-caller, last-caller, single-element array, empty
               array, max-uint, exact-threshold-met-not-exceeded.
```

**The specific sub-check that catches the highest-value member of this class** — a protective check that becomes the attack surface precisely because it's protective:

```
MECHANISM-BLOCKING SUB-CHECK (runs as part of FLAG, every function
containing a guard whose stated purpose is protective — pause, circuit
breaker, slippage cap, deadline, rate limit, cooldown, max-loss check,
health-factor floor, oracle-staleness revert):

Ask: "Does this check ever block the mechanism from doing its job when
it's needed most?"

State concretely: under what system condition does the exact event this
check exists to protect against (a crash, a stale price, a volume
spike, insolvency) ALSO disable the check's own gate — so the protective
action becomes unreachable at the one moment the system actually needs
it? If that condition can be named, W and R both clear by construction.
Do not judge how likely that crash/spike/condition is — likelihood
belongs to Agent 2 (Gate 5's likelihood field, Gate 5.5), not here.
```

Three short shapes this takes in practice, to calibrate what counts (not an exhaustive library — naming the pattern is not the point, see Section 4):
- A pause switch that also blocks liquidations, exactly when a crash both triggers mass liquidations and is the reason an admin paused.
- An oracle-staleness revert that blocks every price-dependent path, including the emergency-exit path, during the exact window the oracle is down.
- A max-slippage check on exit that blocks withdrawal during the high-volatility event a user most needs to exit in — the check that protects against bad exits also forecloses the only exit available.

**Integration note:** if `MATTERS` in the existing candidate format feels too narrow for a pure-availability finding, write `state_transition` (it already covers "the system can no longer transition correctly") rather than forcing a funds framing — do not invent a fund number to satisfy Gate 5 where none exists; Gate 5's non-fund path already exists for exactly this.

---

## 4. Deeper Socratic Questioning — Two New Drills

**The gap:** Socratic questioning was stopping at the first plausible-sounding assumption, and reaching for a *named* pattern ("is this reentrancy? front-running? a sandwich?") as the test for whether to dig further. Naming the pattern is not the point and was never required to surface a candidate — it's a classification step that comes after the bug is found, not a search strategy for finding it. Stopping at "I can't immediately match this to a known pattern" was closing investigations that a purely mechanical question would have kept open.

These two drills extend `References_senior-auditor-sop_pashov_updated.md` §2 (Socratic Questioning). Same mode boundaries apply: Agent 1 uses them to surface-and-stop, Agent 2 uses them to exhaust-and-decide. Neither drill licenses Agent 1 to dig deeper than one candidate, and neither licenses Agent 2 to chain into a new attack path — both inherit the existing boundaries unchanged.

```
DRILL 4 — Mechanism Availability
"Does this check ever block the mechanism from doing its job when it's
needed most?"

AGENT 1: run this on every protective guard found (see Section 3's
MECHANISM-BLOCKING SUB-CHECK). Name one concrete blocking condition,
surface as a candidate, stop — do not enumerate every guard in the
contract searching for more in the same pass; one per FLAG is enough.

AGENT 2 (Gate 7): try to actually construct the named blocking condition
from code + docs facts alone. If it can be constructed and the guard has
no override or escape path (no admin pause-lift, no fallback oracle, no
emergency function that bypasses the guard), that failed invalidation —
CONFIRM. If an override genuinely exists and is itself reachable in time,
that's a real invalidation — discard, citing the override.
```

```
DRILL 5 — Transaction Insertion
"Can an unrelated transaction be inserted between any two of these
steps, and does that insertion change what the next step sees?"

Apply to any function with two or more sequential steps — two state
reads, two external calls, a check followed by an effect, a snapshot
followed by a later use of that snapshot. This question is asked BEFORE
reaching for a pattern name, not after. You do not need to decide
whether the gap "is" reentrancy, front-running, a sandwich, or none of
the above for the question to be worth asking — the mechanical fact of
an exploitable insertion point is sufficient on its own, with or without
a label.

AGENT 1: find ONE pair of steps where some other transaction — anyone's,
not specifically an attacker's — can land between them and change what
the second step sees without the second step re-validating. Name what
changes and what reads it stale. Surface as a candidate (TYPE: FLOW or
INVARIANT, whichever fits the shape), stop. Do not enumerate every step
pair in the function once one gap is found.

AGENT 2 (Gate 7): exhaust the realistic inserted-transaction space for
THIS candidate's specific step pair only — a deposit, a withdrawal, a
price update, a different user's call to the same function, the same
user calling again. Stay confined to transactions that land at the
named insertion point; do not widen into unrelated functions or
downstream effects elsewhere in the contract — that is amplification,
prohibited exactly as it already is everywhere else in Gate 7.
```

---

## 5. End-of-Contract Correlation Step (intra-contract, mechanical)

**The gap:** a weak signal in isolation is often a strong signal once it's seen to repeat within the same contract — but nothing currently forces a look-back at the end of a contract's mind-map pass to check whether the assumption behind one candidate also sits underneath another function nobody flagged as strongly.

**Where this slots in:** SKILL.md Step 2's per-contract loop currently runs (a) header → (b) mind-map → (c) candidates → (d) move to next contract. This step inserts between (c) and (d), as part of finishing the current contract's cycle — it does not start until that contract's candidate pass is fully done, and it must finish before the next contract's header is written.

```
END-OF-CONTRACT CORRELATION STEP:

Ask, mechanically, looking only at the FLAGged facts already on the
table for THIS contract: "Of the facts I used to raise a candidate in
this contract, does any other function in this same contract share the
same assumption?"

This is contained on purpose:
  - It looks backward only, at this contract's own already-completed
    mind-map. It never looks forward into a contract not yet processed
    (that's Section 6, a separate and explicitly cross-contract step).
  - It never re-opens a candidate already decided by the Output Gate.
  - It asks whether the same input ASSUMPTION recurs (the same numeric
    constant, the same external-trust fact, the same ordering
    assumption) — it does NOT ask whether one candidate's EFFECT
    propagates into another function. Tracing effects across functions
    is chaining, and stays prohibited exactly as before.

If two or more functions share the assumption: do not emit N separate
weak candidates. Emit ONE candidate, TYPE: INVARIANT, citing every LOC
site that shares it. WHY WEIRD becomes "this assumption recurs at N
sites, not one"; MATTERS reflects the combined surface across all
cited sites. A shared assumption recurring across sites is itself the
weirdness — it usually means the contract has an implicit invariant
nobody wrote down, and a candidate with multiple confirming sites
survives Gate 5.5 far better than N isolated single-site NUANCEs would.

If no shared assumption is found: write "Correlation check: none —
[Contract.sol]" and move on. This is not optional to skip even when the
answer is no — same rule as every other "none noted" path elsewhere in
this skill: a bare negative still needs a one-clause confirmation that
the check actually ran, so a human reviewer can tell "checked, found
nothing" apart from "never checked."
```

---

## 6. Shared Mind-Map Register (cross-contract, persistent across the run)

**The gap:** moving to the next contract was, in practice, behaving like moving to a new audit. The instruction to process contracts one at a time (correctly, to prevent skimming under combined weight) had a side effect nobody intended: once Contract A's cycle finished, its facts stopped existing as far as Contract B's mind-map pass was concerned. Shared external dependencies, shared trust roles, and shared numeric constants across sibling contracts were going unnoticed purely because nothing carried them forward.

**The fix:** a short running register, not a re-read of every prior contract's full output — cheap enough to maintain that it doesn't reopen the per-contract isolation the existing loop relies on, but enough to stop genuinely shared facts from going unseen.

```
SHARED MIND-MAP REGISTER:

Starts empty at the beginning of the Agent 1 run. Gains exactly one
short entry per contract, appended at the very end of that contract's
full cycle (after Section 5's correlation step, immediately before the
next contract's header is written). Never reset, never rewritten,
never re-summarized — strictly additive.

REGISTER ENTRY — Contract.sol
  EXTERNAL DEPS:     [out-of-scope contracts/oracles/tokens this
                      contract calls — names only, one line]
  TRUST BOUNDARIES:  [privileged roles and what they can do — one line]
  SHARED CONSTANTS:  [any numeric constant, ratio, or threshold that
                      could plausibly recur in a sibling contract — one
                      line, or "none"]
  CALLS / CALLED BY: [in-scope contracts this one calls or is called by
                      — this formalizes the existing CONTRACT
                      DESCRIPTION RELATES field for reuse by later
                      contracts, it does not replace that field]

Before writing the NEXT contract's header, read every prior REGISTER
ENTRY (not the prior contracts' full mind-maps — just these short
entries) and ask, mechanically: "Does this new contract share an
external dependency, trust boundary, or numeric constant with any
already-registered contract?"

If YES: this does not chain a bug across contracts. It surfaces a
candidate — TYPE: INVARIANT or TRUST, whichever fits — stating that the
SAME assumption holds in both places, citing both LOC sites, one per
contract. This is Section 5's correlation step widened in scope, not in
kind: same contained, mechanical shape, just checked against the whole
audit's registered contracts instead of one contract's own functions.
It still does not ask "if Contract A's bug fires, what happens in
Contract B" — that remains chaining and remains prohibited. It only
asks "is the same unstated fact relied on in more than one place,"
which is a fact about the codebase, not a speculative attack path.

If NO shared dependency/boundary/constant is found for a given new
contract: no extra line is required here — unlike Section 5's
correlation step, this cross-contract check doesn't need its own
explicit "none" confirmation per contract, because the register entry
itself (written regardless of outcome) already serves as the audit
trail proving the check had something to compare against.
```

---

## Integration Notes

This file does not modify `SKILL.md`. If and when it gets wired in, here is where each section actually slots in:

```
§1 External Assumption Sub-Check  → Step 2, part of FLAG, alongside the
                                     existing hardcoded-constant sub-check
§2 EIP Differential Sub-Check     → Step 2, feeds the existing EIP
                                     candidate TYPE; also relevant to
                                     Agent 2's pattern-match step for
                                     cross-checking a survived EIP
                                     candidate against the same tables
§3 Non-Theft Taxonomy             → Step 2 (surfacing instinct) + Gate 5
                                     (non-fund impact framing, already
                                     exists — this just makes Agent 1
                                     actively look)
§4 Drill 4 / Drill 5              → Step 2 Socratic use (surfacing) +
                                     Gate 7 Socratic use (destruction) —
                                     extends senior-auditor-sop §2
§5 End-of-Contract Correlation    → Step 2, inserted between sub-step
                                     (c) candidates and (d) next-contract
                                     move, in the existing per-contract
                                     loop
§6 Shared Mind-Map Register       → Step 2, appended once per contract
                                     at the very end of that contract's
                                     cycle; read once before each new
                                     contract's header
```

If added to the Reference Files table in `SKILL.md`, this file's role is closest to the SOP file's: read by Agent 1 directly (not sub-agent-gated), and by Agent 2 at Gate 5/Gate 7. It is not an attack-pattern-pool file (Section 2's EIP tables are spec differentials, not exploit narratives) and not a sub-agent file (Agent 1 reads this one directly, same tier as `References_senior-auditor-sop_pashov_updated.md`).
