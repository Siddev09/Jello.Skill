# References_formatting.md — Final Output Formatting Spec

**Used by:** Sub-agent, Step 4c (Concise Report Assembly) and Final Output assembly.

This file owns the *presentation layer* only — how the surviving `P-N` candidates get written down for the user to read. It has no say over what survives to this point; dedupe (4a) and the trust filter (4b) are decided before this file is ever consulted. Nothing in here can add, remove, or reweight a candidate — it only formats what Step 4b already handed over.

If the output template ever needs to change — a new field, a different grouping, a different summary line — change it here. SKILL.md should not need to be touched for a formatting-only change.

---

## 1. Per-Candidate Line (Concise Report Assembly)

For every candidate that survives dedupe (4a) and the trust filter (4b), write **one to two lines, no more**:

```
[P-N] Contract.sol::function()::lineN — <what goes wrong, plain language, one sentence> — costs: <what it costs, plain language, open-ended — funds, availability, gas, permissions, information, whatever actually applies>
```

Add a second line only if the trigger condition genuinely isn't inferable from the first line:

```
     trigger: <the one concrete condition that causes it>
```

**Prohibited in the per-candidate line:**
```
OPERATION / ERROR / TRIGGER / LOSES / REF blocks (sub-agent's internal working format)
OBS / DOC / WHY WEIRD / REACHABLE / COSTS blocks (Agent 1's internal working format)
Dividers between candidates
Severity, confidence, or "confirmed" labels
Function counts or per-function detail
```

This is the deliverable — it has to be scannable in one pass down the page. No internal working fields survive into it; those exist purely to get the sub-agent to a correct one-line call.

---

## 2. Full Report Template

```
CURIOUS JELLO — run complete.
  Contracts processed:           N
  Trust-filtered out:            N
  Merged duplicates:             N
  Total candidates presented:    N

[P-N] Contract.sol::function()::lineN — one-line description — costs: X
[P-N] Contract.sol::function()::lineN — one-line description — costs: X
[repeat, grouped by contract, in the order contracts were processed]

FILTERED-OUT LOG
[one line per trust-filtered candidate: LOC — role — role assumed trusted, no doc override]
```

**Grouping order:** candidates are grouped by contract, contracts in the order they were originally processed (Step 2's order) — not by severity, not by candidate type, not by ID number.

**Summary counts:** four counts only — contracts processed, trust-filtered out, merged duplicates, total candidates presented. No other summary statistic is added (no function counts, no per-type breakdown, no severity tally).

---

## 3. Formatting Constraints That Apply Throughout

```
No function counts anywhere in the output
No per-candidate multi-field blocks
No severity, confidence, or "confirmed" label on any P-N
No mind-map, per-function list, or function-count statistic shown to the user
Every P-N is a pointer for a researcher to go look at — nothing more
```

These constraints are absolute regardless of how many candidates survive, how complex the protocol is, or how confident the sub-agent is in a given candidate — the formatting never expands to carry a judgment call the pipeline itself doesn't make (see Non-Negotiable Rules in SKILL.md for why those judgment calls don't exist).
