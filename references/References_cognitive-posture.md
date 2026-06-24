# Sub-Agent Cognitive Posture — Internal Reasoning Layer

This file defines two internal cognitive protocols the sub-agent runs silently.
Neither produces output fields, labels, or traces visible to the user.
Both run per-contract, per-candidate, as internal reasoning steps only.
Their only effect is on the precision of OPERATION / ERROR / TRIGGER / LOSES fields
before they are written.

---

## Protocol 1 — State Dependency Scan

**When it runs:** Per candidate, after the Inversion Pass, before writing any output field.

**What it does:**

Every candidate depends on one or more state assumptions being true at the moment
the vulnerable flow executes. This protocol asks whether those assumptions can be
invalidated from *outside* the candidate's own function — by a different function
or contract already in scope.

Internal reasoning sequence:

```
1. Identify the state assumptions this candidate requires to be harmful.
   (e.g. "balance has not been drained", "index has not been updated",
   "approval is still live", "pool ratio has not moved")

2. Scan the full set of in-scope contracts already read for any function
   that can mutate those assumptions independently of this candidate's flow.

3. If such a function exists:
   → The cross-contract path is real. Extend TRIGGER to include the
     external entry point that can set up or amplify the vulnerable state.
   → Do not invent a path. Only include what is directly visible in
     in-scope code already read.

4. If no such function exists:
   → TRIGGER stays scoped to the local flow. No change.
```

**What it never does:**

- Chains candidates together into a multi-step attack narrative
- Invents state manipulation not visible in the in-scope code
- Produces a new candidate — it only sharpens an existing one
- Adds any field or label to the output

---

## Protocol 2 — Intent-Execution Friction Scan

**When it runs:** Once per contract, before the reference pool pattern match begins.
Also runs per candidate during ERROR field construction.

**What it does:**

Natspec, inline comments, function names, event names, and error names are
behavioral promises the code makes to its callers. Where actual execution
breaks those promises, that gap is a candidate — regardless of whether any
reference file named the pattern.

**Step A — Promise Map (pre-scan, per contract):**

```
Before running any reference against this contract, read every:
  - natspec comment (@notice, @dev, @param, @return)
  - inline comment
  - function name
  - event name
  - error name / revert string

Build an internal promise map: for each function, what does it
claim to do, return, emit, or guarantee?

This map is internal only — it is never written to output.
```

**Step B — Friction Check (per candidate, during ERROR field construction):**

```
For each candidate being written, check it against the promise map:

If the candidate's observed behavior conflicts with the promise map
for that function:
  → The conflict sharpens the ERROR field. State the divergence
    concretely: "natspec claims X; code does Y."
  → Do not editorialize. State both sides as observations.

If a promise map conflict exists but NO reference file flagged this
function at all:
  → Surface it as a new candidate under the semantic-drift class.
  → This is the ONE exception to the rule that sub-agent candidates
    require a reference file match. The anchor is always the observable
    gap between stated promise and actual execution — never inference
    or assumption about developer intent.
  → TRIGGER for these candidates: the call path that reaches the
    divergent behavior.
  → LOSES for these candidates: whoever relies on the stated promise
    (caller, integrator, user) receiving incorrect state or funds.
```

**What it never does:**

- Assumes the developer made a mistake — it only reads what the code
  claims vs. what it does
- Invents a promise not stated in natspec/comments/names
- Produces a severity label or validity judgment
- Runs as a rebuttal against an existing candidate

---

## How These Two Protocols Interact With Each Other and With Inversion

Per-candidate internal execution order:

```
1. Reference pool pattern match → candidate surfaced
2. Inversion Pass → fields sharpened for precision
3. State Dependency Scan → TRIGGER extended if cross-contract path exists
4. Intent-Execution Friction Check → ERROR sharpened if promise map conflict exists
5. Write output fields
```

For Protocol 2 Step A (Promise Map), the pre-scan runs once before step 1,
at the start of each contract's sub-agent cycle.

These protocols compound — a candidate that survives all three internal passes
(inversion + state dependency + friction check) before being written has the
most precise TRIGGER and ERROR fields the sub-agent can produce.

---

## False Positive Guard

These protocols increase surface area. To prevent noise:

- **State Dependency Scan:** only extends TRIGGER when the cross-contract
  path is directly readable in already-scanned in-scope code. Never speculate
  about external contracts not in scope.

- **Intent-Execution Friction:** only surfaces a friction candidate when the
  promise is *explicitly stated* in natspec/comments/names — not implied,
  not inferred from context. If it isn't written down as a promise, it isn't
  a promise for this protocol's purposes.

Both constraints keep output grounded in what the code says and does,
not in what a researcher imagines it might do.
