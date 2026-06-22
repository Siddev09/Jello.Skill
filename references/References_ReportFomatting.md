# Report Formatting Reference

Used by: Agent 1 (mind-map + candidate output) and Agent 2 (destruction report). Applies on top of the content rules in SKILL.md — this file governs *how it looks*, not what's allowed to be said. Nothing here changes a gate, a discard condition, or what counts as a survivor. It only fixes structure, scanning, and visual separation so a human reading the output can navigate it without re-reading every line.

---

## Why this file exists

The content rules force short field-by-field output (TYPE/LOC/OBS/WHY WEIRD/REACHABLE/MATTERS, or `[LOC] — gate: reason`). That's correct for keeping reasoning tight. But tight fields with no visual separation between entries, and no structure when a reason genuinely needs more than one sentence, produces a wall of text that's hard to scan even when every individual judgment is sound. This file fixes the presentation layer only.

---

## 1. Mind-Map Formatting — Per-Contract Structure

**Processing is strictly per-contract** (see SKILL.md Step 2's mandatory order): one contract's complete cycle — header, full mind-map, that contract's candidates — finishes before the next contract starts. This section's formatting reinforces that separation rather than working against it.

**Numbering resets per contract.** `[N]` restarts at `[1]` for every new contract's mind-map. This is a deliberate change from treating the whole audit as one flat sequence — each contract is visually its own self-contained unit, which is the whole point of processing them separately. If GeneralVault.sol has 7 functions and LidoVault.sol has 9, LidoVault's first entry is `[1]`, not `[8]`.

```
─────────────────────────────────────────
GeneralVault.sol  ·  7 functions
─────────────────────────────────────────
Role    · [one line]
Holds   · [one line]
Relates · [one line]
Flag    · [one line, or "none noted"]
─────────────────────────────────────────

1  functionName    lineN    [one-line plain description, flag note if flagged]
2  functionName    lineN    [one-line plain description, flag note if flagged]
3  functionName    lineN    clean
4  functionName    lineN    clean
...
7  functionName    lineN    [one-line plain description, flag note if flagged]

─────────────────────────────────────────
CANDIDATES  ·  GeneralVault.sol
─────────────────────────────────────────
[candidates go here — see §2]
[if zero: "no candidates surfaced from this contract."]
─────────────────────────────────────────
SUB-AGENT  ·  GeneralVault.sol
─────────────────────────────────────────
[sub-agent candidates go here — see §2.5]
[if zero: "no math/numerical/semantic-drift candidates surfaced."]
─────────────────────────────────────────

─────────────────────────────────────────
LidoVault.sol  ·  9 functions
─────────────────────────────────────────
...
```

Rules:
- The mind-map table is **one line per function** — function name, line number, and either a short flag note or "clean". No multi-line PLAIN blocks. Functions with nothing suspicious get "clean" and move on.
- A flagged function's note is the flag compressed to one clause — if it needs more, it should be a candidate, not a flag note.
- Contract header meta (Role / Holds / Relates / Flag) uses `·` alignment — four fields, one line each, no colon, no all-caps labels.
- CANDIDATES and SUB-AGENT sections are immediately after the mind-map table under the same contract header, separated by `─────` dividers. Never batch all contracts' candidates together at the end.
- **Never group all contracts' mind-maps together followed by all contracts' candidates together.** Each contract is fully resolved (mind-map + candidates + sub-agent) before the next contract header appears.
- A multi-contract run reads as N consecutive self-contained sections, not a long mind-map followed by a long candidate list.

---

## 2. Candidate Formatting

Candidates use a compact label-value layout. One candidate per block, separated by a blank line. No divider lines between candidates — whitespace is enough. TYPE is on the header line so it's scannable without reading the rest.

```
C-1  FLOW  ·  repay()  ·  line 142
  Obs        [one sentence — what the code does wrong]
  Doc        [what spec says, or "not addressed"]
  Why weird  [why this is suspicious — one clause]
  Reachable  [who reaches it and how]
  Matters    [what the attacker gains or what breaks]

C-2  INVARIANT  ·  deposit()  ·  line 44
  Obs        [one sentence]
  Doc        [not addressed]
  Why weird  [one clause]
  Reachable  [one clause]
  Matters    [one clause]
```

Rules:
- Candidate numbers are prefixed `C-` and **stay sequential across the whole audit, not reset per contract** — unlike mind-map `[N]` numbering, which does reset per contract (see §1). A global `C-N` makes every candidate referenceable by one stable number regardless of which contract it came from.
- TYPE keeps its plain-text value (NUANCE, INVARIANT, TRUST, FLOW, EIP) — on the same header line as `C-N` and the function name.
- Labels are lowercase, left-aligned, value follows on the same line. No all-caps headers, no blank lines between fields inside a candidate block.
- Blank line between candidate blocks is the only separator needed — no `─────` dividers between individual candidates.
- Each field stays one line wherever possible. If a field genuinely needs two lines, wrap the second under the value column — never let it run into the next field's label.

---

## 2.5. Sub-Agent Candidate Formatting

Sub-agent candidates mirror the Option C label-value style used for C-N candidates — same compact layout, same blank-line separation between blocks. Distinguished from Agent 1 candidates by the `SA-` prefix. No TYPE field — every sub-agent candidate is implicitly math/numerical/semantic-drift.

```
SA-1  ·  functionName()  ·  line 89
  Operation  [what the math/numeric op does — concrete values and types]
  Error      [what goes wrong — rounding direction, overflow, drift — as outcome not pattern name]
  Trigger    [input or state condition that causes it]
  Loses      [who loses what — funds / accounting / state]
  Ref        [reference filename that flagged this]

SA-2  ·  functionName()  ·  line 201
  Operation  [...]
  Error      [...]
  Trigger    [...]
  Loses      [...]
  Ref        [...]
```

Rules:
- `SA-N` numbering is **sequential across the whole audit, not reset per contract** — same logic as `C-N`. Starts at `SA-1` and increments through all contracts.
- Labels are lowercase, left-aligned, value on the same line — identical visual grammar to §2 candidate blocks.
- No OBS, DOC fields — the sub-agent does not build a coverage map and does not check docs.
- REF is mandatory — always name which reference file (`References_math-precision-agent_pashov.md`, `References_numerical-gap-agent_pashov.md`, `References_semantic-drift.md`, `References_periphery-agent_pashov.md`, `References_rounding-entitlement.md`) flagged this. If more than one applies, list both on the same line.
- A sub-agent block with zero candidates still renders — write `no math/numerical/semantic-drift candidates surfaced.` and move on. Never skip the block silently.

---

## 3. Discard Log Formatting

This is the section that was hardest to read in practice. Long gate-kill reasoning (especially Gate 4 reachability-not-proven cases, which often require explaining *why* a sequence couldn't be constructed) was getting crammed into a single run-on line. Fix: keep the one-line summary as a scannable headline, then let the full reasoning breathe underneath it as bullets — no length cap, but mandatory structure.

```
─────────────────────────────────────────
[D-N] ✕ Contract.sol::functionName()::lineN  —  GATE: [gate name]

SUMMARY
  [the one-sentence version — this is what you read first, and it
  must stand alone as a complete reason even if no one reads further]

REASONING
  • [first point — e.g. what the code shows]
  • [second point — e.g. what's missing/unproven]
  • [third point — e.g. why the alternative interpretation isn't
    ruled out, if relevant]
  • [as many bullets as the reasoning actually needs — no cap]
─────────────────────────────────────────
```

Rules:
- SUMMARY is mandatory and must be genuinely one sentence — this is the part someone skimming 20 discards reads first to decide whether to open REASONING.
- REASONING is optional structurally (a gate1/gate3 kill is often fully explained by SUMMARY alone — don't manufacture bullets to fill space) but when used, it's bullets, never a paragraph. Break at natural reasoning seams: what the code does, what's missing, why ambiguity remains, what would resolve it.
- `D-N` numbering is sequential across the whole discard log, separate numbering track from mind-map `[N]` and candidate `C-N`.
- ✕ marks every discard log entry as a visual differentiator from the ◆ candidate marker and from emitted findings (which use a different marker, see §4).
- Pre-send discards and gate discards share this exact format — don't format them differently from each other. The only difference is what goes in the GATE field (`pre-send` vs `gate1` through `gate8`).

**Example — applying this to a real gate4 case:**

```
─────────────────────────────────────────
[D-1] ✕ ConvexCurveLPVault.sol::withdrawOnLiquidation()::~165-189  —  GATE: gate4

SUMMARY
  Reachability not proven — the vault's balance of the burned token is
  ~0 in steady state, and no in-scope code shows it being replenished
  before the burn.

REASONING
  • _withdraw() burns internalAssetToken from address(this), but the
    deposit flow transferFroms 100% of that token into LendingPool's
    reserve on mint — the vault doesn't hold a steady-state balance.
  • Whether the vault holds the balance at burn time depends entirely
    on out-of-scope LendingPool's liquidation sequence pushing it back
    first.
  • This would mirror the explicit withdrawFrom-then-burn pattern
    GeneralVault uses for normal withdrawals — but this function
    conspicuously doesn't call that pattern itself.
  • Architecturally, a push-back step is the more plausible design
    (vanilla Aave-style liquidation hands the underlying straight to
    the liquidator, not round-trip through the vault) — but nothing
    in-scope settles it either way.
  • No call sequence can be constructed using only the given code that
    proves insufficiency.
─────────────────────────────────────────
```

Compare to the original wall-of-text version: same content, same rigor, same conclusion — but now scannable. A reader can stop after SUMMARY if they trust the gate, or read REASONING if they want to manually re-check this exact discard (which is the entire point of keeping a discard log in the first place).

---

## 4. Emitted Finding Formatting

Emitted findings already have a detailed format in SKILL.md (TITLE/SEVERITY/CONFIDENCE/LOC/ROOT CAUSE/CALL SEQ/IMPACT/CODE PROOF/COUNTERARGUMENTS FAILED/DOCS/TRUST/SCOPE/JUDGE). This file does not restructure those fields — they're already well-separated. It adds only:

- A `[F-N]` sequential number and a `✓` marker (distinguishing emitted findings from `✕` discards and `◆` candidates) at the top of each finding block.
- A divider before and after each finding, same visual weight as mind-map and discard entries, so the whole report has one consistent visual grammar.

```
─────────────────────────────────────────
[F-1] ✓ HIGH  ·  Contract.sol::functionName()::lineN

TITLE: ...
CONFIDENCE: ...

ROOT CAUSE
  ...

CALL SEQUENCE
  1. ...
  2. ...

IMPACT
  ...

CODE PROOF
  ...

COUNTERARGUMENTS FAILED
  (a) ...
  (b) ...
  (c) ...

DOCS: ...
TRUST: ...
SCOPE: ...
JUDGE: ...
─────────────────────────────────────────
```

---

## 5. Section Headers — top-level report structure

Top-level boundaries use a single heavy `━━━` line with a plain label. Contract boundaries use `─────` lines. Everything else is whitespace and indentation — no `█████` blocks, no `═══` double borders.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
AGENT 1 OUTPUT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

─────────────────────────────────────────
GeneralVault.sol  ·  7 functions
─────────────────────────────────────────
Role    · [one line]
Holds   · [one line]
Relates · [one line]
Flag    · [one line or "none noted"]
─────────────────────────────────────────

1  functionName    lineN    [flag note or "clean"]
2  functionName    lineN    [flag note or "clean"]
...
7  functionName    lineN    [flag note or "clean"]

─────────────────────────────────────────
CANDIDATES  ·  GeneralVault.sol
─────────────────────────────────────────

C-1  FLOW  ·  functionName()  ·  line N
  Obs        [...]
  Doc        [...]
  Why weird  [...]
  Reachable  [...]
  Matters    [...]

C-2  INVARIANT  ·  functionName()  ·  line N
  [...]

─────────────────────────────────────────
SUB-AGENT  ·  GeneralVault.sol
─────────────────────────────────────────

SA-1  ·  functionName()  ·  line N
  Operation  [...]
  Error      [...]
  Trigger    [...]
  Loses      [...]
  Ref        [...]

─────────────────────────────────────────
LidoVault.sol  ·  9 functions
─────────────────────────────────────────
[... repeat pattern ...]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PRE-SEND DISCARD LOG
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[C-N discards only — SA-N bypass pre-send]
[entries per §3 format]
```

For Step 5's final report:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EMITTED FINDINGS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[findings per §4, or "none — no candidates survived destruction this run."]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DISCARD LOG
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[all discards per §3]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
APPENDIX — MIND-MAP
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[full pipeline only — carried forward from Agent 1 output unchanged]
```

Visual hierarchy, top to bottom: `━━━` marks top-level report sections → `─────` marks per-contract blocks → blank lines separate individual candidates and sub-agent entries within a block. Three levels, no more.

---

## 6. What This File Does Not Change

```
Does not loosen the 3-line PLAIN cap or any other content length rule
Does not add a length cap to discard reasoning that didn't exist before
Does not change what counts as a candidate, a discard, or a survivor
Does not change gate order, gate logic, or emission criteria
Does not add new fields beyond numbering and section markers
Does not apply to the halt-and-confirm short summary in Step 3 (Agent 3
  path) — that summary stays as counts-only by design, not full entries
```

Two additions in this file that touch content adjacently (not purely presentation):
- **CONTRACT DESCRIPTION block (§1, §5)** — four-line description per contract, sits between the contract header and the mind-map. Content rules for this block live in SKILL.md Step 2. This file only governs its visual placement and the four-field structure (ROLE/HOLDS/RELATES/FLAG).
- **Sub-agent SA-N format (§2.5, §5)** — four-field candidate format for math/numerical/semantic-drift candidates. Content rules live in SKILL.md Step 2.5. This file only governs its visual structure, marker (`◈`), and placement relative to Agent 1 candidates in the output.

One exception to "presentation only": §1's per-contract interleaving mirrors a processing-order rule that lives in SKILL.md Step 2 (process one contract fully before starting the next). That rule itself is content/process, not formatting — it belongs to SKILL.md and is enforced there. This file's role is narrower: make sure the *output* doesn't silently undo that separation by re-batching mind-maps and candidates back together across contracts after the fact.
