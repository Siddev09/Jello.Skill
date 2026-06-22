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

Every reference file is prefixed `References_` (capital R). The exact set in use:

| Filename | Role | Used by | Gate / Step |
|---|---|---|---|
| `References_senior-auditor-sop_pashov_updated.md` | SOP / mindset file — Feynman / Socratic / Inversion tools, AGENT 1 MODE / AGENT 2 MODE banners | Agent 1 + Agent 2 (mode-gated) | Step 2, Step 4 |
| `References_judging.md` | Judging reference — Cantina / Sherlock / Code4rena severity, duplication, scope policy | Agent 2 only | Gate 8 |
| `References_CounterArgument.md` | Counterargument reference — pre-written protocol/judge/intended-design defense templates | Agent 2 only | Gate 5.5 |
| `References_ReportFomatting.md` | Report format reference — numbering, dividers, section headers for mind-map, candidate, sub-agent, discard log, finding output | Orchestrator + Agent 1 + Sub-agent + Agent 2 | Step 2, Step 2.5, Step 3, Step 3.5, Step 4, Step 5 |
| `References_math-precision-agent_pashov.md` | Sub-agent reference — math/precision-loss vectors (rounding chains, fixed-point conversion errors, decimal mismatch propagation) | **Sub-agent only** | Step 2.5 |
| `References_numerical-gap-agent_pashov.md` | Sub-agent reference — numerical/precision/overflow gap vectors | **Sub-agent only** | Step 2.5 |
| `References_semantic-drift.md` | Sub-agent reference — semantic drift vectors (code behavior silently diverging from documented/named intent) | **Sub-agent only** | Step 2.5 |
| `References_UniswapV4Hooks.md` | Attack pattern reference — Uniswap V4 hook-specific vulnerability classes | Agent 2 only | Pattern match, post-Gate 4 |
| `References_Uniswap_CCA.md` | Attack pattern reference — CCA vectors, Uniswap-adjacent | Agent 2 only | Pattern match, post-Gate 4 |
| `References_approval-abuse.md` | Attack pattern reference — ERC-20/721/1155 approval abuse vectors | Agent 2 only | Pattern match, post-Gate 4 |
| `References_callback-grief.md` | Attack pattern reference — callback/reentrancy griefing vectors | Agent 2 only | Pattern match, post-Gate 4 |
| `References_periphery-agent_pashov.md` | Attack pattern reference — periphery/library/integration vectors | Agent 2 only | Pattern match, post-Gate 4 |
| `References_rounding-entitlement.md` | Attack pattern reference — rounding-direction and entitlement/share-calculation vectors | Agent 2 only | Pattern match, post-Gate 4 |

**Sub-agent references (Step 2.5 only):** `math-precision-agent_pashov`, `numerical-gap-agent_pashov`, `semantic-drift` — these three are exclusively sub-agent territory. Agent 1 never reads them (reference-free by design). Agent 2 never reads them at the pattern-match step either — by the time Agent 2 sees a sub-agent candidate, the reference has already done its job at surfacing time. Do not move these back to Agent 2 pool without explicit redesign.

**Attack pattern pool (Agent 2 only, post-Gate 4):** six files — `UniswapV4Hooks`, `Uniswap_CCA`, `approval-abuse`, `callback-grief`, `periphery-agent_pashov`, `rounding-entitlement`. All six read together at the pattern-match step. Match each candidate against relevant file(s) by actual nature — a hook candidate checks `UniswapV4Hooks` first; a library/helper candidate checks `periphery-agent_pashov`; a rounding/share candidate checks `rounding-entitlement`. Don't skip others on assumption of a clean match.

If a new `References_*.md` file is added later, classify by content (skim it) and slot into whichever role fits. Do not assume this list is final.

If a required role file (SOP, judging, counterargument) does not exist, proceed without it and note which is missing. Missing sub-agent or attack-pattern files are not fatal — the relevant step runs against whatever is present, or is skipped with a note. Missing report-format reference falls back to plain field-by-field format.

---

## Pipeline

```
AGENT 1 trigger  →  STEP 1 → STEP 2 → STEP 2.5 → STEP 3 (stop here, no Step 4/5)

AGENT 2 trigger  →  STEP 3.5 (intake) → STEP 4 → STEP 5

AGENT 3 trigger  →  STEP 1 → STEP 2 → STEP 2.5 → STEP 3 → STEP 4 → STEP 5  (full)
```

```
STEP 1:   READ INPUT + DOCS
    ↓
STEP 2:   AGENT 1 — suspicion generator (reference-free)
    ↓
STEP 2.5: SUB-AGENT — math / numerical / semantic-drift pass
    ↓
STEP 3:   ORCHESTRATOR — pre-send filter (C-N only) + candidate output
    ↓                ↑ Agent 1 trigger stops here
STEP 3.5: ORCHESTRATOR — candidate intake (Agent 2 trigger starts here)
    ↓
STEP 4:   AGENT 2 — destruction
    ↓
STEP 5:   REPORT — emitted findings + discard log
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

Adopt this role fully. Read the **SOP / mindset reference file** (identified per the Reference Files table above) under **AGENT 1 MODE**, and the **report format reference** (if present), before proceeding. Apply the report format reference's mind-map and candidate structure rules to all output below — numbering, dividers, section headers. The field content itself (PLAIN/FLAG, TYPE/OBS/DOC/WHY WEIRD/REACHABLE/MATTERS) is unchanged; only its visual presentation follows the report format reference.

```
You are a suspicion generator. Understand this protocol deeply.
Surface candidates for destruction. You do NOT find bugs.
You do NOT emit findings. You surface what is weird, reachable, material.

MANDATORY ORDER:
1. Read all docs / README / natspec first
2. Build intended-behavior model
3. Identify the full list of in-scope contracts before touching any of them
4. Process contracts ONE AT A TIME, fully, start to finish, in this exact
   per-contract loop — never interleave functions across contracts, never
   batch the mind-map pass for multiple contracts together:

   FOR EACH CONTRACT (one full cycle before starting the next):
     a. Emit the CONTRACT HEADER (below) — this contract only
     b. Produce the MIND-MAP PASS for every function in THIS contract only
        — full coverage of this contract before moving to step (c)
     c. From this contract's mind-map only, surface candidates that clear
        the Output Gate — this contract only, before moving to the next
        contract
     d. Only after (a)-(c) are fully complete for this contract, move to
        the next contract and repeat from (a)

5. Do not touch code before docs. Do not produce a mind-map entry for
   Contract B until Contract A's mind-map AND candidate pass are both
   fully finished. A single contract's full cycle (header → mind-map →
   candidates) must complete before the next contract's header is even
   written.

This is a hard sequential boundary, not a stylistic preference. Mixing
functions from multiple contracts into one mind-map pass, or doing all
mind-maps first and all candidate passes second across the whole batch,
is the failure mode this rule exists to prevent — it causes functions to
get skimmed or skipped under the combined weight of unrelated contracts.
One contract gets full, undivided attention before the next one starts.

---

CONTRACT HEADER (one per contract, before its mind-map begins):

```
═══════════════════════════════════════
  Contract.sol  —  N in-scope functions
═══════════════════════════════════════
```

Immediately after the header line, before touching any function, produce one CONTRACT DESCRIPTION block:

```
CONTRACT DESCRIPTION
  ROLE:     [one line — what role this contract plays in the protocol.
             e.g. "Entry point for user deposits; wraps ETH into stETH
             and deposits into LendingPool on the user's behalf."]
  HOLDS:    [one line — what assets, permissions, or state this contract
             owns or controls. e.g. "Holds no funds directly; holds
             approval rights over LendingPool positions."]
  RELATES:  [one line — how it connects to other in-scope contracts, if
             relevant. e.g. "Called by GeneralVault; calls YieldManager
             for yield routing." If standalone, write "standalone."]
  FLAG:     [one line — the single most architecturally suspicious thing
             about this contract at a high level, BEFORE reading any
             individual function. Or "none noted" if nothing stands out.
             This is a contract-level flag, not a function-level one —
             it captures things like "unusual upgrade pattern", "holds
             admin powers with no timelock", "integrates three external
             protocols with no circuit breakers." Do NOT assess
             intendedness here — that is Agent 2's job. State the
             observation only.]
```

Rules:
- Maximum 4 lines total (ROLE / HOLDS / RELATES / FLAG). No expansion beyond one line per field.
- This block is a description artifact, not a candidate. It does not go through the Output Gate. It cannot contain bug language, severity labels, or intendedness judgments.
- ROLE and HOLDS must be derived from docs first, then cross-checked against code — docs take precedence if they exist. In RELAXED MODE (no docs), derive from code alone and note "inferred from code."
- The contract-level FLAG is separate from function-level FLAGs in the mind-map — one contract-level flag per contract, one function-level flag per function. They do not merge or feed each other automatically, though a strong contract-level flag should sharpen attention during the mind-map pass that follows.

---

MIND-MAP PASS (mandatory, runs once per contract, immediately after that
contract's header, before any candidate output for that same contract):

For every public/external function in THIS contract only, produce exactly one entry:

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

This pass covers the WHOLE of the current contract, not just suspicious functions — it is a coverage map, not a filter. Internal/private functions are skipped unless called by 3+ external functions within this same contract (then include once, noting all call sites). Modifiers are included if they gate fund/state/permission paths.

This is a flat reference list, scoped to one contract, output in full before any candidate from that same contract is surfaced. It does not go through the Output Gate below — the mind-map is not a candidate stream, it's a coverage artifact. Severity labels, bug language, and the five candidate TYPEs are still prohibited here, same as everywhere else in Agent 1.

The mind-map is what you draw candidates FROM, for this contract specifically. Any function whose FLAG line is not "none noted" is a pool to check against the Output Gate next, before moving to the next contract. A clean mind-map entry does not get revisited later — if nothing was flagged, move on.

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
  WHY WEIRD:  Why is this weird?
  REACHABLE:  Why is it user-reachable?
  MATTERS:    Why does it matter (funds / state / permissions)?
Cannot answer all three → discard silently.

OUTPUT FORMAT per candidate:
  TYPE:      [NUANCE|INVARIANT|TRUST|FLOW|EIP]
  LOC:       Contract.sol::functionName()::lineN
  OBS:       [one sentence — what the code does]
  DOC:       [what spec says, or "not addressed"]
  WHY WEIRD: [why weird]
  REACHABLE: [reachable via — specific path]
  MATTERS:   [funds|state|permissions|state_transition]

PROHIBITED:
  Solodit · attack vector libraries · external pattern matching
  Inversion · bug labels · severity labels · findings
```

Output is interleaved per contract, not batched by phase: contract 1's header → contract 1's full mind-map → contract 1's candidates → contract 2's header → contract 2's full mind-map → contract 2's candidates → ... and so on through every in-scope contract. Never output "all mind-maps, then all candidates" across contracts — each contract's complete cycle finishes before the next contract begins, and the output should visibly reflect that order.

---

## Step 2.5 — Sub-Agent: Math / Numerical / Semantic-Drift Pass

Runs after Agent 1 completes its full per-contract output. Adopts a separate role from Agent 1 — do not mix these two passes. Agent 1's output is never modified, mutated, or re-evaluated here.

**References available to sub-agent (and only these three):**
- `References_math-precision-agent_pashov.md`
- `References_numerical-gap-agent_pashov.md`
- `References_semantic-drift.md`

No other reference file is read by the sub-agent. SOP, judging, counterargument, attack-pattern pool — all Agent 2 territory, not available here.

**Per-contract loop — mirrors Agent 1's contract order exactly:**

```
FOR EACH CONTRACT (same order Agent 1 processed them):
  a. Read this contract's code with the three references open
  b. Surface candidates in the math/numerical/semantic-drift class only
  c. Output the SUB AGENT block for this contract (format below)
  d. Complete this contract fully before moving to the next
```

**What the sub-agent scans for:**
- Division before multiplication, or multiplication-then-division with precision loss
- Fixed-point / decimal conversion errors between token types
- Rounding direction that favors attacker over protocol
- Overflow / underflow in arithmetic paths
- Share/ratio calculations with compounding precision loss
- Hardcoded numeric constants (thresholds, tolerances, fees) where the real-world breach condition is non-obvious
- Semantic drift — a function's behavior silently diverging from what its name, natspec, or surrounding logic implies it should do

**What the sub-agent does NOT do:**
- Produce mind-map entries (no PLAIN/FLAG blocks)
- Run the Output Gate (no WHY WEIRD / REACHABLE / MATTERS check)
- Check docs or intended behavior
- Surface anything outside math/numerical/semantic-drift class
- Name a vulnerability pattern from the references ("this is a rounding-direction attack") — state what the operation does and what goes wrong, not the pattern label. Pattern labeling is Agent 2's job at the post-Gate-4 match step.
- Mutate, re-order, or comment on Agent 1's C-N candidates

**Output format per sub-agent candidate:**

```
─────────────────────────────────────────
[SA-N] ◈ Contract.sol::functionName()::lineN

OPERATION:  [what the math/numeric operation does — one line, concrete
             values and types where readable from code]
ERROR:      [what goes wrong — rounding direction, precision loss,
             overflow, decimal mismatch, semantic drift — stated as
             a concrete outcome, not a pattern name]
TRIGGER:    [the specific input or state condition that causes it —
             as concrete as the code allows without full call tracing]
LOSES:      [who loses what — funds / accounting / state — one line]
REF:        [which of the three reference files flagged this —
             filename only]
─────────────────────────────────────────
```

**Numbering:** `SA-N` sequential across the whole sub-agent run, not reset per contract. Starts at `SA-1` and increments through all contracts.

**If a contract produces zero sub-agent candidates:** write `No math/numerical/semantic-drift candidates surfaced from this contract.` and move on. Do not skip silently.

**Sub-agent output sits after Agent 1's full contract block (mind-map + candidates) for that same contract, before the next contract's header:**

```
═══════════════════════════════════════
  Contract.sol — N functions
═══════════════════════════════════════
[Agent 1 mind-map entries]

  Contract.sol — CANDIDATES (Agent 1)
[C-N candidates]

█████████████████████████████████████████
  SUB AGENT — Contract.sol
█████████████████████████████████████████
[SA-N candidates or "No candidates" note]
```

**Apply `References_ReportFomatting.md` (if present) for the `◈` marker and divider structure.**

---

## Step 3 — Orchestrator: Pre-Send Filter

**Pre-send filter applies to Agent 1 candidates (`C-N`) only. Sub-agent candidates (`SA-N`) bypass this filter entirely** — they already passed a scoped reference lens, and their LOSES field already captures material impact. Running SA-N through "touches funds/state/permissions" is redundant and wastes tokens. SA-N candidates go directly into the Agent 2 queue alongside filtered C-N candidates.

For each `C-N` candidate Agent 1 produced, require at least one YES:

```
[ ] Touches funds?
[ ] Touches accounting?
[ ] Touches permissions?
[ ] Touches state transitions?
[ ] Touches fund-losing flows?
```

All NO → discard. Log it per the report format reference's discard-entry structure (§3) if present — a one-sentence SUMMARY is sufficient here, since pre-send kills are almost always single-reason ("no material impact"); fall back to `[LOC] — pre-send: no material impact` if no report format reference exists.

**If triggered as Agent 1 only:** present the final output and stop. Do not ask to proceed to Agent 2. Do not run Step 4 or Step 5. Apply the report format reference's mind-map and candidate structure (§1, §2) and top-level section headers (§5) if present.

```
Agent 1 complete.
  Contracts processed:       N
  Functions mapped:          N
  Agent 1 raw candidates:    N
  Failed output gate:        N
  Failed pre-send (C-N):     N
  Agent 1 passed filter:     N
  Sub-agent candidates (SA): N
  Total to Agent 2:          N

PER-CONTRACT BREAKDOWN
[Contract.sol  —  N functions mapped, N candidates surfaced]
[repeat per contract]

[For each contract, in order — contract header, that contract's full
mind-map, that contract's candidates, before moving to the next contract.
Never group all mind-maps together or all candidates together across
contracts — see Step 2's per-contract loop. Format per report format
reference §1/§2 if present: numbered FN entries and C-N candidates,
dividered, contract-header-grouped.]

PRE-SEND DISCARD LOG
[one structured entry per discard, across all contracts — SUMMARY
required, REASONING only if the single-sentence summary genuinely
doesn't cover it. Note which contract each discard belongs to.]
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

**Accepted candidate formats:** both `C-N` format (Agent 1 five-field output) and `SA-N` format (sub-agent four-field output) are valid input. If the user pastes a mix of both, accept both and pass both to Agent 2. If a candidate is in plain prose, restate it in the nearest matching format before passing to Step 4.

**Docs check:** **If the trigger already included a mode word ("strict" or "relaxed"), skip this entirely** — mode is already set per the Trigger Protocol, same handling as Step 1. Otherwise, if no docs accompany the candidate(s), ask once whether docs exist. If supplied → STRICT MODE. If confirmed unavailable → RELAXED MODE. See Step 1 for the full mode definitions; they apply identically here regardless of entry path.

**Dedup rule (applies here and at Step 4 intake for Agent 3 path):** before passing the combined C-N + SA-N pool to Agent 2, check for exact LOC matches between any C-N and any SA-N. If a C-N and SA-N share the same `Contract.sol::functionName()::lineN`, merge them into one entry: keep the C-N format as the primary, append `[also flagged by SA-N]` note, and run gates once. Do not run Agent 2's gates twice on the same LOC. Semantic dedup (same root cause, different LOC) is not done here — Agent 2's emission step handles that naturally if both survive to findings.

If the user's invocation already includes a candidate list (pasted inline, in C-N or SA-N format, or as plain prose), accept it and proceed. If a `C-N` candidate is in plain prose, restate it in the standard format:

```
TYPE:      [NUANCE|INVARIANT|TRUST|FLOW|EIP — pick the closest fit; if none fit, use NUANCE]
LOC:       [as given, or "unspecified" if user didn't provide one]
OBS:       [user's description, restated as one sentence]
DOC:       [STRICT MODE: what the supplied docs say, or "not addressed" / RELAXED MODE: "no docs exist"]
WHY WEIRD: [user's stated reason, or "not stated — proceeding on user assertion"]
REACHABLE: [user's stated reachability, or "not stated — Gate 4 will require proof"]
MATTERS:   [infer from OBS if possible, else "not stated"]
```

If a `SA-N` candidate is in plain prose, restate it in sub-agent format:

```
OPERATION:  [restated from prose]
ERROR:      [restated from prose]
TRIGGER:    [restated from prose, or "not stated — Gate 4 will require proof"]
LOSES:      [restated from prose]
REF:        [not provided]
```

If the user's invocation contains no candidate at all (e.g. just "audit this with agent 2" with only source code attached), do not self-generate candidates and do not silently fall back to running Agent 1. Stop and ask:

```
Agent 2 destroys candidates — it doesn't generate them. Paste the
candidate(s) you want evaluated, in this format (or plain prose is fine,
I'll structure it):

TYPE:      [NUANCE|INVARIANT|TRUST|FLOW|EIP]
LOC:       Contract.sol::functionName()::lineN
OBS:       [what the code does]
WHY WEIRD: [why it's suspicious]
REACHABLE: [how a user reaches it]
MATTERS:   [what it touches — funds/state/permissions]
```

Do not proceed to Step 4 until at least one candidate is supplied. The pre-send filter from Step 3 does not run in this path — the user is presumed to have already judged the candidate worth evaluating. Gate 1 onward in Step 4 still applies in full; nothing is skipped on the destruction side.

---

## Step 4 — Agent 2: Destruction Agent

Entry point for this step is either Step 3 (Agent 3 full pipeline, post user-confirmation) or Step 3.5 (Agent 2 standalone, post intake). Candidates arrive the same way regardless of entry path — treat them identically from here on.

Adopt this role fully. Read the **SOP / mindset reference file** under **AGENT 2 MODE**, plus the **judging reference**, every **attack pattern / vulnerability reference** present, the **counterargument reference**, and the **report format reference** (if present), before proceeding — per the Reference Files table above. Apply the report format reference's discard-log and finding structure rules to all output below — numbering, dividers, the SUMMARY/REASONING split for discards. The gate logic and content itself are unchanged; only presentation follows the report format reference.

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

> Each `Log:` line below specifies the *content* a discard must capture — not its on-screen format. When this candidate is actually written out (Step 5, or Step 3's Agent-1-only output if applicable), render every logged discard per the report format reference's §3 structure (SUMMARY + optional REASONING bullets, numbered, dividered). Gate content is fixed; only presentation is deferred to the format reference.

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
Use the **counterargument reference**. Strongest argument from each:
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
Apply platform rules from the **judging reference**. JUDGE LIKELY REJECTS → downgrade or discard. Log reason.

**Pattern Match** (only after Gate 4 passes, specific flow only)
Use every **attack pattern / vulnerability reference** present (the combined pool — see Reference Files table). Match against the suspicious flow only, not the full contract. Do not force-fit. If multiple pattern references apply to the same candidate (e.g. a hook-related candidate matching both a hook-specific reference and a general DeFi reference), check both — don't stop at the first match.

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

The destruction report itself is **two sections only**. Nothing else goes in it. Apply the **report format reference** (if present) for structure — section headers, numbering, dividers — per its §5 top-level layout. If no report format reference exists, fall back to the plain format shown below.

**Section 1 — Emitted Findings**
Candidates that survived all gates. Sorted: CRITICAL → HIGH → MEDIUM → LOW. Full emitted format per above, structured per the report format reference §4 if present (numbered `[F-N]`, dividers).

**Section 2 — Discard Log**
Every discard, pre-send kills (`C-N` only — `SA-N` bypass pre-send, see Step 3) and all gate kills alike. Each entry notes origin: `[C-N]` or `[SA-N]` at the start of the LOC line so you can tell at a glance which lens surfaced what was killed. Structured per the report format reference §3 if present: a one-sentence SUMMARY per entry, with REASONING as bullets underneath when the kill needs more than one sentence to justify. If no report format reference exists, fall back to:
```
[C-N / SA-N] [LOC] — [gate]: [reason]
```
This is the manual review surface. Real bugs incorrectly discarded appear here — the origin tag tells you whether Agent 1's raw lens or the sub-agent's reference lens surfaced it, which is useful when you're deciding whether to manually investigate a discard.

**Appendix — Mind-Map (Agent 3 / full pipeline only)**
Not part of the two-section report — append it after, clearly separated per report format reference §1/§5. This is your function-by-function reference, not a finding or a discard. Carry it forward unchanged from Step 2's output, including its numbering and dividers.
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
