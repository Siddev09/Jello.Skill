# Counterargument Templates

Used only at Gate 5.5. Default stance: the candidate is invalid until the strongest counterargument fails.

For every candidate, produce exactly one instance of each of the three perspectives below. Pick the template that matches the candidate's nature. If no template fits cleanly, construct the strongest version of that perspective from scratch — do not skip a perspective because no template applies.

**Rule:** evaluate the single strongest counterargument across all three. Weak counterarguments are not evidence of validity. A candidate survives only when the strongest version of each defense, not a weak version, fails against code + docs.

---

## (a) Protocol Team Defense

The protocol team's job is to explain why this is fine, in the most credible way available to them. Construct their best version, not a strawman.

**Templates by candidate type:**

```
ACCESS CONTROL / TRUST
"This function is only callable by [role]. We trust [role] by design —
see docs section on trusted roles. This is not a bug, it's the threat model."
Refute only if: role is NOT documented as trusted, OR role is acquirable
permissionlessly (fee payment, governance token purchase, etc.)

ROUNDING / PRECISION LOSS
"This rounds in the protocol's favor, which is standard and documented
practice to prevent value leakage. Dust-level loss is expected behavior."
Refute only if: rounding direction favors the attacker, not the protocol,
OR the loss is not dust-level (apply Gate 5 thresholds), OR it's exploitable
repeatedly to extract non-dust value.

EXTERNAL DEPENDENCY / ORACLE
"We rely on [oracle/provider] which is an industry-standard integration.
Any flaw is in their system, not ours, and out of scope."
Refute only if: the integration code itself fails to validate staleness/
deviation/return values that ARE within the auditor's control to check.

UNUSUAL BUT INTENTIONAL DESIGN
"This is intentional. See [docs reference] — it's a deliberate tradeoff,
not an oversight."
Refute only if: docs do NOT actually describe this specific behavior,
or describe something materially different.

FEE-ON-TRANSFER / NON-STANDARD TOKEN
"We don't support non-standard tokens. This is documented as
out-of-scope token behavior."
Refute only if: docs do not explicitly exclude this token class, OR
the protocol's own token allowlist permits it without restriction.
```

---

## (b) Judge Defense

The judge's job is to apply contest rules mechanically, often more conservatively than a security argument alone would suggest. Construct the rejection a judge would issue, citing the framework in judging.md.

**Templates by candidate type:**

```
PRECONDITION-HEAVY
"Requires [N] external conditions to align (specific block, specific
state, specific actor sequence). Per judging.md, extensive precondition
complexity pulls severity down — likely capped at Medium or rejected as Low."
Refute only if: preconditions are trivial, commonly occurring, or
attacker-controllable without cost.

SUB-THRESHOLD IMPACT
"Loss is under the $10 / 0.01% floor — per Sherlock-style thresholds
this is Invalid unless it breaks a stated invariant."
Refute only if: a concrete number from Gate 5 clears the threshold,
OR an explicit README invariant is broken regardless of dollar size.

ADMIN/TRUSTED-ROLE PATH
"This requires admin or owner action. Per the exclusion checklist,
admin error and malicious-admin scenarios are out of scope by default."
Refute only if: the bug is in the admin function's IMPLEMENTATION
(a flaw a trusted admin couldn't avoid even acting honestly), not
the call itself, OR the role is documented as untrusted/permissionless.

NO PoC / THEORETICAL
"No concrete proof of exploitability — reentrancy, gas/DoS, and
precision-loss claims require a PoC per judging standards. Without
one, this is high risk of rejection or downgrade."
Refute only if: Gate 4's call sequence is concrete and traceable
directly from source, not speculative.

DUPLICATE / ROOT-CAUSE OVERLAP
"This shares root cause with [pattern class] — same logic mistake,
different surface. Likely grouped and judged at a single severity."
Refute only if: the mechanism is genuinely distinct (different fix,
different code-level cause, different attack path) — not just a
different function with the same root cause, which IS groupable.
```

---

## (c) Intended Design Defense

The hardest defense to refute, because intended behavior cannot be a bug. Construct the version where this is exactly how the system is supposed to work, even if it looks wrong from the outside.

**Templates by candidate type:**

```
DOCS SILENT, BEHAVIOR CONSISTENT
"Docs don't address this explicitly, but the behavior is internally
consistent across the whole contract — this reads as an intentional
design pattern, not an oversight."
Refute only if: the behavior is inconsistent elsewhere in the same
contract (similar functions handle the case differently), which is
evidence of oversight rather than design.

ECONOMIC ASSUMPTION
"The system assumes [market condition / actor behavior] holds. This
is a standard assumption across DeFi (e.g., liquidity exists, oracles
aren't simultaneously stale and manipulated). Not a bug, a known limit."
Refute only if: the assumption breaks under conditions the protocol
ITSELF claims to defend against, or under conditions cheap enough
to be realistic (not a black-swan event requiring coordinated failure
across multiple independent systems).

MIGRATION / VERSIONING ARTIFACT
"This is leftover from a prior version / migration path and is
intentionally inert in current deployment."
Refute only if: the code path IS reachable in the current deployment
configuration — verify via constructor params or active deployment,
not just code presence.

GRIEFING WITHOUT PROFIT
"This costs the attacker more than it costs the victim — pure
griefing with no profit motive is typically informational, not a
funded attack anyone would execute."
Refute only if: the attacker profits (directly or via short position,
arbitrage, or competitive advantage), OR the cost asymmetry still
makes it cheap enough to be a viable nuisance attack with real harm.
```

---

## Evaluation Output Format

```
CANDIDATE: [LOC]

(a) Protocol defense: [selected/constructed template]
    Refuted by code/docs: [Y/N] — [one sentence why]

(b) Judge defense: [selected/constructed template]
    Refuted by code/docs: [Y/N] — [one sentence why]

(c) Intended design defense: [selected/constructed template]
    Refuted by code/docs: [Y/N] — [one sentence why]

STRONGEST DEFENSE: [a/b/c]
STRONGEST HOLDS: [Y/N]

VERDICT: [DISCARD — strongest defense holds | SURVIVES — all three refuted]
```

If STRONGEST HOLDS = Y → discard at Gate 5.5. Log: `[LOC] — gate5.5: [strongest defense, one sentence]`.

If all three are refuted → candidate proceeds to Gate 6.

---

## What This File Does Not Do

```
Does not generate weak counterarguments to wave through findings
Does not skip a perspective because none seems to apply — construct one
Does not accept "could be intended" as ambiguous — force a specific defense
Does not let a refuted (a) excuse skipping (b) and (c) — all three required
```
