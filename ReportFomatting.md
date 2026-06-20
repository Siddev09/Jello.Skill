# Report Formatting Reference

Used by: Agent 1 (mind-map + candidate output) and Agent 2 (destruction report). Applies on top of the content rules in SKILL.md — this file governs *how it looks*, not what's allowed to be said. Nothing here changes a gate, a discard condition, or what counts as a survivor. It only fixes structure, scanning, and visual separation so a human reading the output can navigate it without re-reading every line.

---

## Why this file exists

The content rules force short field-by-field output (TYPE/LOC/OBS/W/R/M, or `[LOC] — gate: reason`). That's correct for keeping reasoning tight. But tight fields with no visual separation between entries, and no structure when a reason genuinely needs more than one sentence, produces a wall of text that's hard to scan even when every individual judgment is sound. This file fixes the presentation layer only.

---

## 1. Mind-Map Formatting

Every mind-map entry is numbered and separated by a divider. Never run entries together with only a blank line.

```
─────────────────────────────────────────
[N] FN: Contract.sol::functionName()::lineN

PLAIN
  [line 1]
  [line 2]
  [line 3, if needed]

FLAG
  [the flag, or "none noted"]
─────────────────────────────────────────
```

Rules:
- `[N]` is sequential across the whole mind-map, not reset per contract. If GeneralVault.sol has 7 functions and LidoVault.sol has 9, LidoVault's first entry is `[8]`, not `[1]`.
- PLAIN stays max 3 lines as the content rules already require — this file does not loosen that cap. What changes is each line gets its own visual line in the output, not run together in one paragraph.
- FLAG is one line. If the flag needs more explanation than one line provides, that's a signal that this function should already be heading to candidate output — don't expand FLAG into a paragraph, expand it into a candidate instead.
- Group entries by contract with a contract header before the first entry of each file:

```
═══════════════════════════════════════
  GeneralVault.sol  —  7 functions
═══════════════════════════════════════
[1] FN: ...
─────────────────────────────────────────
[2] FN: ...
─────────────────────────────────────────
...

═══════════════════════════════════════
  LidoVault.sol  —  9 functions
═══════════════════════════════════════
[8] FN: ...
```

---

## 2. Candidate Formatting

Every candidate is numbered and separated by a divider, same pattern as the mind-map, with the candidate TYPE visually tagged at the top so the type is scannable without reading the LOC line.

```
─────────────────────────────────────────
[C-N] ◆ TYPE  ·  Contract.sol::functionName()::lineN

OBS
  [one sentence]

DOC
  [what spec says, or "not addressed"]

WHY WEIRD
  [the W field]

REACHABLE
  [the R field]

MATTERS
  [the M field]
─────────────────────────────────────────
```

Rules:
- Candidate numbers are prefixed `C-` and sequential (`C-1`, `C-2`...) to visually distinguish them from mind-map `[N]` numbers when both appear in the same output (Step 3's full-pipeline summary, for instance).
- TYPE keeps its plain-text value (NUANCE, INVARIANT, TRUST, FLOW, EIP) — the `◆` is a scan marker, not a replacement for the type label.
- Each field (OBS/DOC/WHY WEIRD/REACHABLE/MATTERS) gets its own line and its own short header. No field is allowed to run into the next field's text — if W and R blur together in a single paragraph, split them, even if that means repeating a clause.

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

When a full report combines mind-map + candidates + discard log (or emitted findings + discard log), separate the major sections with a heavier header than the per-entry dividers, so the eye can jump straight to "did anything survive" without scrolling past the mind-map first.

```
█████████████████████████████████████████
  MIND-MAP
█████████████████████████████████████████

[mind-map entries]

█████████████████████████████████████████
  CANDIDATES
█████████████████████████████████████████

[candidate entries]

█████████████████████████████████████████
  PRE-SEND DISCARD LOG
█████████████████████████████████████████

[pre-send discards]
```

For Step 5's final report:

```
█████████████████████████████████████████
  EMITTED FINDINGS
█████████████████████████████████████████

[findings, or "None — no candidates survived destruction this run." if empty]

█████████████████████████████████████████
  DISCARD LOG
█████████████████████████████████████████

[all discards]

█████████████████████████████████████████
  APPENDIX — MIND-MAP
█████████████████████████████████████████

[mind-map, full pipeline only]
```

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
