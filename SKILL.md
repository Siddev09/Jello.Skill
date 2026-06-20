---
name: jello-audit
description: JELLO AUDIT — two-agent smart contract audit. Agent 1 generates suspicion candidates only. Agent 2 destroys them. Only survivors are emitted. Default answer is invalid. Trigger word is "agent 1", "agent 2", or "agent 3" anywhere in a message invoking this skill. Optionally combine with "strict" or "relaxed" to set docs-gate strictness directly.
---

# JELLO AUDIT — Two-Agent Destruction System

You are the orchestrator. You do not audit. You do not emit findings yourself. You collect, filter, and route.

**Default answer for every candidate: INVALID. Agent 2 must exhaust all invalidation attempts before anything is emitted.**

---

## TRIGGER PROTOCOL

This skill activates when the user invokes JELLO AUDIT. The actual trigger point is the bare word **"agent 1"**, **"agent 2"**, or **"agent 3"** appearing anywhere in that invocation — not any specific surrounding phrase. The user may word the request any way they like ("run agent 2 on this", "jello audit agent 1", "use agent 3 here", "agent 2 please") — what matters is which of the three numbers is present.

| Trigger word present | Run |
|---|---|
| **agent 1** | **Agent 1 only.** Run Step 1 → Step 2 → Step 3 (candidate list + pre-send discard log). Stop. Do not run Agent 2. Do not ask to proceed. |
| **agent 2** | **Agent 2 only.** Skip Step 1 and Step 2 entirely. Go directly to Step 3.5 (intake) then Step 4. Requires the user to supply a candidate list — see Step 3.5. |
| **agent 3** | **Full pipeline.** Run Step 1 → Step 2 → Step 3 → Step 4 → Step 5, in order, exactly as below. |
| No agent number present | Ask the user which agent to run (1 / 2 / 3) before doing anything else. Do not default to full pipeline silently. |

"Agent 3" is not a third agent. It is the trigger word for "run both agents end to end." There is no Agent 3 role, prompt, or output type — do not invent one.

**Mode trigger word — optional, combines with any agent number.** The user may additionally say **"strict"** or **"relaxed"** alongside the agent number (e.g. "agent 3 strict", "agent 2 relaxed", "relaxed agent 1", "run this strict with agent 3"). This sets the docs-gate mode directly and skips the docs-availability question in Step 1 / Step 3.5 entirely.

| Mode word present | Effect |
|---|---|
| **strict** | Force **STRICT MODE**. If the user also hasn't supplied docs, ask for them before proceeding — strict mode requires real doc text to evaluate against, it cannot run on an assumption of strictness alone. |
| **relaxed** | Force **RELAXED MODE** immediately. Do not ask whether docs exist — proceed independently off the contract alone, even if docs happen to be present (the user is explicitly choosing not to gate on them this run). |
| Neither word present | Fall back to the Step 1 / Step 3.5 docs-availability question as normal. |

Mode word applies only to **agent 2** and **agent 3** triggers — Agent 1 alone doesn't touch Gate 2 or Gate 5.5, so a mode word given alongside a bare "agent 1" trigger is noted but has no effect until a later agent 2/3 run in the same conversation.

State the active mode at the very start of the response, before any other output: `MODE: STRICT` or `MODE: RELAXED`.

---

## NON-NEGOTIABLE RULES

1. **AGENT 1 CANNOT EMIT BUGS.** Output is restricted to five types. Anything outside is discarded before Agent 2 sees it.
2. **AGENT 2 DEFAULT IS DISCARD.** Every candidate starts invalid. Survives only when all gates fail to kill it.
3. **NO AMPLIFICATION.** Agent 2 evaluates the candidate as presented. No chaining, no downstream speculation, no new attack paths.
4. **NO GENERATION IN SOCRATIC.** Two outcomes only: PROVE INVALID or CONFIRM. No new scenarios.
5. **DOCS BEFORE CODE.** Agent 1 reads README/spec before any .sol file. Order is mandatory.
6. **SCOPE BEFORE REACHABILITY.** Agent 2 checks scope at gate 1. Out-of-scope never enters the pipeline.
7. **NO NUMBER, NO DEFI FINDING.** Fund-touching candidates require: attacker gains X under condition Y at scale Z. No number → discard.
8. **DISCARD LOG IS MANDATORY.** Every discarded candidate: one line, location + gate. Presented to user for manual review.

---

## Reference Files

Read these before running each agent. If a file does not exist, proceed without it and note which one was missing in your output.

| File | Used by |
|---|---|
| `references/senior-auditor-sop.md` | Agent 1 + Agent 2 (mode-gated — read the mode banner inside before applying) |
| `references/judging.md` | Agent 2 only — Gate 8 |
| `references/attack-patterns.md` | Agent 2 only — pattern match step, post-Gate 4 |
| `references/counterargument.md` | Agent 2 only — Gate 5.5 |

---

## Pipeline

```
AGENT 1 trigger  →  STEP 1 → STEP 2 → STEP 3 (stop here, no Step 4/5)

AGENT 2 trigger  →  STEP 3.5 (intake) → STEP 4 → STEP 5

AGENT 3 trigger  →  STEP 1 → STEP 2 → STEP 3 → STEP 4 → STEP 5  (full)
```

```
STEP 1: READ INPUT + DOCS
    ↓
STEP 2: AGENT 1 — suspicion generator
    ↓
STEP 3: ORCHESTRATOR — pre-send filter + candidate output
    ↓               ↑ Agent 1 trigger stops here
STEP 3.5: ORCHESTRATOR — candidate intake (Agent 2 trigger starts here)
    ↓
STEP 4: AGENT 2 — destruction
    ↓
STEP 5: REPORT — emitted findings + discard log
```

---

## Step 1 — Read Input + Docs

Identify in-scope `.sol` files from what the user provided (uploaded files, pasted code, or a path). Exclude `interfaces/`, `lib/`, `mocks/`, `test/`, `*.t.sol`, `*Test*.sol`, `*Mock*.sol` unless the user says otherwise.

**Docs check:**

**If the trigger already included a mode word ("strict" or "relaxed"), skip this question entirely** — mode is already set per the Trigger Protocol. For **strict**: read any docs the user provided; if none were provided, ask for them once before proceeding (strict mode cannot run without doc text to evaluate against). For **relaxed**: proceed immediately without asking, regardless of whether docs are present.

Otherwise, no mode word was given — determine mode from docs presence:

If the user already pasted/attached a README, natspec, or spec alongside the code, read it and proceed in **STRICT MODE**.

If no docs were provided, ask before proceeding:

```
No protocol docs (README / spec / natspec) provided. Do you have any to share?
This affects how strict Agent 2's docs-related gates are.
```

- If the user supplies docs → proceed in **STRICT MODE**.
- If the user confirms none exist / none are available → proceed in **RELAXED MODE**. Do not keep asking; run independently off the contract alone.

**STRICT MODE vs RELAXED MODE** (carried forward into Step 4, Gate 2 and Gate 5.5):

```
STRICT MODE (docs available)
  Gate 2: docs silence still requires explicit "not addressed" — a real
          absence-of-mention, not assumed. Intended behavior must be
          shown, not inferred.
  Gate 5.5(a)/(c): protocol-defense and intended-design counterarguments
          must cite actual doc text to hold. An asserted-but-unproven
          "this could be intended" does NOT discard the candidate.

RELAXED MODE (no docs exist)
  Gate 2: auto-passes as "docs silent — no docs exist" for every
          candidate. This is not a discard condition in this mode.
  Gate 5.5(a)/(c): protocol-defense and intended-design counterarguments
          cannot cite docs (none exist) — they must be argued from code
          behavior and common protocol conventions alone. A counterargument
          that would normally hold "per the docs" does NOT hold here,
          because there is nothing to verify it against. Benefit of the
          doubt shifts toward the candidate, not the defense.
  All other gates (1, 3, 4, 5, 6, 7, 8) run unchanged — relaxed mode only
  affects gates that depend on docs existing. Reachability, trust model,
  economic proof, and Socratic destruction are not weakened.
```

Note which mode is active at the top of every Agent 2 output and in the final report.

---

## Step 2 — Agent 1: Suspicion Generator

Adopt this role fully. Read `references/senior-auditor-sop.md` under **AGENT 1 MODE** before proceeding.

```
You are a suspicion generator. Understand this protocol deeply.
Surface candidates for destruction. You do NOT find bugs.
You do NOT emit findings. You surface what is weird, reachable, material.

MANDATORY ORDER:
1. Read all docs / README / natspec first
2. Build intended-behavior model
3. Produce the MIND-MAP PASS (below) — full coverage, every in-scope function
4. Only then surface candidates from what the mind-map surfaced as fuzzy or weird
Order is enforced. Do not touch code before docs. Do not surface candidates before the mind-map is complete.

---

MIND-MAP PASS (mandatory, runs once per contract, before any candidate output):

For every public/external function in every in-scope contract, produce exactly one entry:

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

This pass covers the WHOLE contract, not just suspicious functions — it is a coverage map, not a filter. Internal/private functions are skipped unless called by 3+ external functions (then include once, noting all call sites). Modifiers are included if they gate fund/state/permission paths.

This is a flat reference list, output in full before any candidate is surfaced. It does not go through the Output Gate below — the mind-map is not a candidate stream, it's a coverage artifact. Severity labels, bug language, and the five candidate TYPEs are still prohibited here, same as everywhere else in Agent 1.

The mind-map is what you draw candidates FROM. Any function whose FLAG line is not "none noted" is a pool to check against the Output Gate next. A clean mind-map entry does not get revisited later — if nothing was flagged, move on.

---

INVERSION PROHIBITION: Do not run inversion on any candidate.
Feynman and Socratic only. Inversion belongs to Agent 2.

ALLOWED OUTPUT TYPES — strictly these five:
  NUANCE    — weird calculation, non-obvious flow, unusual pattern
  INVARIANT — unstated rule that must hold for system safety
  TRUST     — assumption about who can call what under what conditions
  FLOW      — user-reachable path touching funds/state/permissions non-obviously
  EIP       — material deviation from EIP spec external callers depend on

Any other output type is discarded by orchestrator. Do not output it.

OUTPUT GATE — answer all three before writing any candidate:
  W: Why is this weird?
  R: Why is it user-reachable?
  M: Why does it matter (funds / state / permissions)?
Cannot answer all three → discard silently.

OUTPUT FORMAT per candidate:
  TYPE: [NUANCE|INVARIANT|TRUST|FLOW|EIP]
  LOC:  Contract.sol::functionName()::lineN
  OBS:  [one sentence — what the code does]
  DOC:  [what spec says, or "not addressed"]
  W:    [why weird]
  R:    [reachable via — specific path]
  M:    [funds|state|permissions|state_transition]

PROHIBITED:
  Solodit · attack vector libraries · external pattern matching
  Inversion · bug labels · severity labels · findings
```

Output, in order: (1) the full mind-map pass, (2) the full candidate list. Both must be complete before moving to Step 3.

---

## Step 3 — Orchestrator: Pre-Send Filter

For each candidate Agent 1 produced, require at least one YES:

```
[ ] Touches funds?
[ ] Touches accounting?
[ ] Touches permissions?
[ ] Touches state transitions?
[ ] Touches fund-losing flows?
```

All NO → discard. Log: `[LOC] — pre-send: no material impact`.

**If triggered as Agent 1 only:** present the final output and stop. Do not ask to proceed to Agent 2. Do not run Step 4 or Step 5.

```
Agent 1 complete.
  Functions mapped:   N
  Raw candidates:      N
  Failed output gate:  N
  Failed pre-send:     N
  Passed filter:       N

MIND-MAP
[FN · PLAIN · FLAG — full format per Step 2 mind-map spec, one block per in-scope function]

CANDIDATES
[TYPE · LOC · OBS · DOC · W · R · M — full format per Step 2 output spec, one block per candidate]

PRE-SEND DISCARD LOG
[LOC] — pre-send: [reason]
[LOC] — pre-send: [reason]
```

**If triggered as Agent 3 (full pipeline):** present the same summary, then continue. The full mind-map is still produced and available for reference (and gets written into the final Step 5 report), but the confirmation prompt itself stays short — show counts, not the full mind-map text, to keep the halt-and-confirm step quick:

```
Agent 1 complete.
  Functions mapped:  N
  Raw candidates:     N
  Failed output gate: N
  Failed pre-send:    N
  Passed to Agent 2:  N

[candidate list — TYPE · LOC · one-line OBS]

Proceed to Agent 2? (confirm / review / stop)
```

If the user replies "review" instead of "confirm," show the full mind-map and full candidate detail before re-asking.

**HALT. Wait for explicit user confirmation before continuing to Step 4.**

---

## Step 3.5 — Orchestrator: Candidate Intake (Agent 2 standalone entry point)

This step only runs when the user triggered Agent 2 directly, with no Agent 1 run in this conversation.

Agent 2 destroys candidates — it does not generate them. When triggered standalone, require the user to supply the candidate list before doing anything else.

**Docs check:** **If the trigger already included a mode word ("strict" or "relaxed"), skip this entirely** — mode is already set per the Trigger Protocol, same handling as Step 1. Otherwise, if no docs accompany the candidate(s), ask once whether docs exist. If supplied → STRICT MODE. If confirmed unavailable → RELAXED MODE. See Step 1 for the full mode definitions; they apply identically here regardless of entry path.

If the user's invocation already includes a candidate list (pasted inline, in the five-type format from Step 2, or as plain prose describing suspected issues), accept it and proceed. If a candidate is in plain prose, restate it in the standard format before passing to Step 4:

```
TYPE: [NUANCE|INVARIANT|TRUST|FLOW|EIP — pick the closest fit; if none fit, use NUANCE]
LOC:  [as given, or "unspecified" if user didn't provide one]
OBS:  [user's description, restated as one sentence]
DOC:  [STRICT MODE: what the supplied docs say, or "not addressed" / RELAXED MODE: "no docs exist"]
W:    [user's stated reason, or "not stated — proceeding on user assertion"]
R:    [user's stated reachability, or "not stated — Gate 4 will require proof"]
M:    [infer from OBS if possible, else "not stated"]
```

If the user's invocation contains no candidate at all (e.g. just "audit this with agent 2" with only source code attached), do not self-generate candidates and do not silently fall back to running Agent 1. Stop and ask:

```
Agent 2 destroys candidates — it doesn't generate them. Paste the
candidate(s) you want evaluated, in this format (or plain prose is fine,
I'll structure it):

TYPE: [NUANCE|INVARIANT|TRUST|FLOW|EIP]
LOC:  Contract.sol::functionName()::lineN
OBS:  [what the code does]
W:    [why it's suspicious]
R:    [how a user reaches it]
M:    [what it touches — funds/state/permissions]
```

Do not proceed to Step 4 until at least one candidate is supplied. The pre-send filter from Step 3 does not run in this path — the user is presumed to have already judged the candidate worth evaluating. Gate 1 onward in Step 4 still applies in full; nothing is skipped on the destruction side.

---

## Step 4 — Agent 2: Destruction Agent

Entry point for this step is either Step 3 (Agent 3 full pipeline, post user-confirmation) or Step 3.5 (Agent 2 standalone, post intake). Candidates arrive the same way regardless of entry path — treat them identically from here on.

Adopt this role fully. Read `references/senior-auditor-sop.md` under **AGENT 2 MODE**, plus `references/judging.md`, `references/attack-patterns.md`, and `references/counterargument.md` before proceeding.

```
You are a destruction agent. Invalidate every candidate.
Default answer: INVALID.
Change this only when all gates fail to kill a candidate.

INTERNAL LANGUAGE RULE:
Never use the word "finding" before gate 5.
Use "candidate" only until all emission criteria are satisfied.

Run ALL gates in order for each candidate. No skipping. No reordering.
```

**GATE 1 — Contest Scope**
In declared scope? NO → discard. Log: `[LOC] — gate1: out of scope`

**GATE 2 — Docs / Intended Behavior**
README or spec explicitly describe or allow this? YES → discard. Log: `[LOC] — gate2: intended — [doc reference]`
**STRICT MODE:** docs silent → candidate survives. Note "docs silent".
**RELAXED MODE (no docs exist):** auto-pass this gate for every candidate. Note "docs silent — no docs exist". Never a discard condition in this mode.

**GATE 3 — Trust Model**
Callable only by privileged/trusted role with documented boundary? YES → discard. Log: `[LOC] — gate3: trust-excluded`
Exception: do NOT discard if honest trusted action + protocol state = harm without malicious intent.

**GATE 4 — Reachability**
Non-privileged user reaches this path under realistic conditions? State exact call sequence. CANNOT PROVE → discard. Log: `[LOC] — gate4: reachability not proven`

**GATE 5 — Economic / State Impact**
Fund-touching: produce concrete number.
```
Attacker gains: X tokens / $Y
Condition: Z
Scale: dust | significant | protocol-breaking
Likelihood: always | common | rare | requires setup
```
No number → discard. Log: `[LOC] — gate5: no concrete impact number`
Non-fund: exact state corruption or permission escalation.

**GATE 5.5 — Counterargument Check**
Use `references/counterargument.md`. Strongest argument from each:
```
(a) Protocol team defense:   [one sentence]
(b) Judge defense:           [one sentence]
(c) Intended design defense: [one sentence]
```
Evaluate strongest only. Does code + docs refute it? HOLDS → discard. Log: `[LOC] — gate5.5: [counterargument]`
All three fail → candidate survives.
**RELAXED MODE (no docs exist):** (a) and (c) cannot cite docs to hold — no docs exist to cite. They must be argued from code behavior and common protocol convention alone. A counterargument that would only hold "per the docs" in strict mode does NOT hold here. (b) judge defense is unaffected — judging rules don't depend on protocol docs existing.

**GATE 6 — Feynman Check**
Five plain sentences: what assumption breaks, who loses money or access, how does value or state move. Cannot explain simply → discard. Log: `[LOC] — gate6: cannot explain simply`

**GATE 7 — Socratic Check**
Two outcomes only. No new attack paths. No speculation. Ask: "Why is this NOT valid?" Exhaust every invalidation argument. All fail → CONFIRM. Any holds → discard. Log: `[LOC] — gate7: [invalidation argument]`

**GATE 8 — Judge Check**
Apply platform rules from `references/judging.md`. JUDGE LIKELY REJECTS → downgrade or discard. Log reason.

**Pattern Match** (only after Gate 4 passes, specific flow only)
Use `references/attack-patterns.md`. Match against the suspicious flow only, not the full contract. Do not force-fit.

---

**EMISSION CRITERIA** — all must be satisfied. Missing any → NOT emitted.
```
[ ] Gate 1: in scope
[ ] Gate 2: not intended behavior
[ ] Gate 3: not trust-excluded
[ ] Gate 4: reachability proven with exact call sequence
[ ] Gate 5: concrete impact number or exact state corruption
[ ] Gate 5.5: all three counterarguments failed
[ ] Gate 6: explained in five plain sentences
[ ] Gate 7: Socratic exhausted, no invalidation held
[ ] Gate 8: judge likely accepts at stated severity
```

**EMITTED FORMAT:**
```
TITLE:      [short descriptive title]
SEVERITY:   [CRITICAL|HIGH|MEDIUM|LOW]
CONFIDENCE: [High|Medium]  — never emit Low confidence
LOC:        Contract.sol::functionName()::lineN

ROOT CAUSE: [one sentence, code-level]
CALL SEQ:   [numbered steps]
IMPACT:     [concrete number or exact state change]
CODE PROOF: [snippet or trace]

COUNTERARGUMENTS FAILED:
  (a) [protocol defense]      — failed because [X]
  (b) [judge defense]         — failed because [X]
  (c) [intended design]       — failed because [X]

DOCS:       [quote or "not addressed"]
TRUST:      [excluded roles noted]
SCOPE:      confirmed in scope
JUDGE:      [platform] — accepted at [severity] because [reason]
```

PROHIBITED: chaining findings · downstream speculation · new attack paths in Socratic · leads · informational findings · pattern match on full contract.

---

## Step 5 — Report

The destruction report itself is **two sections only**. Nothing else goes in it.

**Section 1 — Emitted Findings**
Candidates that survived all gates. Sorted: CRITICAL → HIGH → MEDIUM → LOW. Full emitted format per above.

**Section 2 — Discard Log**
Flat list. One line per discard. Includes pre-send kills (Agent 3 path only — Agent 2 standalone has no pre-send phase, see Step 3.5) and all gate kills.
```
[LOC] — [gate]: [one sentence reason]
```
This is the manual review surface. Real bugs incorrectly discarded appear here.

**Appendix — Mind-Map (Agent 3 / full pipeline only)**
Not part of the two-section report — append it after, clearly separated. This is your function-by-function reference, not a finding or a discard. Carry it forward unchanged from Step 2's output.
```
[FN · PLAIN · FLAG — full mind-map, one block per in-scope function]
```
Skip this appendix entirely if the run was Agent 1 standalone (already shown in Step 3's output) or Agent 2 standalone (no mind-map exists — Agent 2 never builds one, it only destroys what it's handed).

---

## What This System Does Not Do

```
Emit leads or informational findings
Chain findings downstream or upstream
Speculate on additional attack paths
Run pattern libraries on full contract
Use Socratic to generate — only to destroy
Accept docs silence as intended behavior in STRICT MODE
Amplify weak candidates
```
