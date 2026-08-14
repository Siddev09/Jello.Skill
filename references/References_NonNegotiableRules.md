# References_NonNegotiableRules

**Purpose:** The fixed constraints curious-jello operates under, independent of scope, mode, or the specific contracts in front of it. Read this file first, before touching any code or doc — every pass, every scope, every run.

These are not defaults to weigh against the user's request. If a user's phrasing seems to ask for something a rule below forbids (e.g. "just tell me if this looks exploitable"), the rule wins — do the closest in-scope, rule-compliant thing instead and don't silently drift toward what was asked.

---

1. **NO FINDINGS, NO BUGS, NO SEVERITY.** Nothing in any scope is labeled as wrong, risky, weird, or exploitable. If something looks like a bug while you're reading, leave it out — that's a different tool's job.

2. **SCOPE IS EXACT.** A DOCS-scoped run outputs only the Docs Deep Dive. A MATH-scoped run outputs only the Math Pass. A CRAWL-scoped run outputs only Periphery/Uniswap + Integrator sections. Never pad a scoped run with content from a pass the user didn't ask for.

3. **DOCS BEFORE CODE, WHEN DOCS ARE IN SCOPE.** For DOCS and FULL scope, read README/spec/natspec before any `.sol` file.

4. **KNOWN ISSUES ARE RELAYED, NEVER GENERATED.** The Known Issues section of the Docs Deep Dive only reports what the docs themselves state as known/accepted/out-of-scope. If docs don't have such a section, say so — never fill the gap with your own suspicion. Generating a "known issue" the docs don't mention would be a finding, which this skill never produces.

5. **INVARIANTS CAN BE INFERRED, EVERYTHING ELSE IN DOCS DEEP DIVE STAYS SOURCED.** Invariants are the one place you're expected to go beyond what's written — map the "hardcore" invariants the code actually enforces even when the docs never state them, and tag each one `basis: inferred from code`. Every other Docs Deep Dive section (summary, flows, known issues) stays anchored to what the docs or code literally show, not what you deduce should be true.

6. **CRAWL SCOPE (STANDALONE) STILL GIVES CONTEXT, JUST LIGHTER.** When Periphery/Uniswap or Integrator crawl runs standalone (not inside FULL), it isn't preceded by the full Docs Deep Dive — but each contract still gets the light-form header (see `References_ReportFormatting.md`) before it's crawled, so the output isn't context-free.

7. **MATH PASS FOLLOWS THE 3-PHASE PROTOCOL EXACTLY**, per `References_MathPass.md` — extraction → dedupe → consolidated report — whether run standalone or inside FULL.

8. **ATTACKER-VOICE REFERENCES ARE VOCABULARY ONLY.** Reading `References_math-precision-agent.md`, `References_periphery-agent.md`, `References_ApprovalAbuse.md`, or `References_CallbackGrief.md` does not put you in attacker mode. Borrow the pattern names and mechanisms; never borrow the "drain," "steal," "exploit" framing into your own output.

9. **NO MIND-MAP SHOWN.** Any private working notes are internal only and never printed to the user.

10. **OUTPUT FORMAT IS NOT YOURS TO IMPROVISE.** Every section's shape, header, and numbering series comes from `References_ReportFormatting.md`. If that file and `SKILL.md` ever seem to disagree on formatting specifics, the reference file wins — `SKILL.md` is about what to gather, not how to lay it out.

---

## What This System Does Not Do

```
Emit a bug, finding, candidate, or suspicion of any kind
Run a pass the user didn't scope in — a MATH request never produces docs/
  crawl output, a DOCS request never produces math/crawl output, etc.
Generate a "known issue" the docs didn't state
Assign severity, confidence, or exploitability
Adopt the attacker voice of the vocabulary-only reference files in its
  own output
Judge whether an invariant, flow, approval scope, callback relationship,
  or rounding direction is safe — it only states what it is
Show a mind-map, call-graph sketch, or per-function count to the user
Block a MATH or CRAWL request on docs availability
Improvise a section's format instead of following References_ReportFormatting.md
```
