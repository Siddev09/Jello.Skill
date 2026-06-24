**Name:** CURIOUS JELLO

---

**Why use it**

Every audit tool either judges too early or pattern-matches too narrowly. CURIOUS JELLO separates the two problems. The suspicion pass has no pattern library so it can't be tilted — it only surfaces what's genuinely weird in this specific code. The sub-agent has no judgment power so it can't close candidates — it only adds what the reference pool knows the suspicion pass isn't looking for. You get both passes without either one polluting the other.

---

**What it does differently**

Three things no standard audit flow does:

First, the suspicion pass is philosophically reference-free. It doesn't know what a reentrancy looks like, what a rounding bug looks like, what solodit says. It reads the contract as a first-principles observer and asks what's weird, reachable, and material — nothing else.

Second, the sub-agent runs inversion silently per candidate. Before writing a single output field it asks "what would have to be true for this to be harmless" and checks whether the code enforces that. The researcher gets sharper TRIGGER and LOSES fields without seeing the reasoning.

Third, the sub-agent builds a promise map before touching the reference pool — every natspec, comment, function name, event name is read as a behavioral contract. Where actual execution breaks that promise, it surfaces as a candidate even if no reference file named the pattern. That's the one gap every reference-only tool has.

---

**How the flow looks / what to expect**

```
You invoke → docs check → MODE declared

Per contract:
  Suspicion pass → CONTRACT HEADER + MIND-MAP (every function)
                 → C-N candidates (weird / reachable / material only)
  Sub-agent     → Promise Map built internally
                 → Reference pool scan → Inversion → State Dependency
                   → Friction check → SA-N candidates written
  
After all contracts:
  Dedupe → C-N and SA-N merged where same LOC
  Materiality filter → C-N only, SA-N bypass
  
Final output:
  Run stats → per-contract breakdown → all candidates → filter log
```

You get a mind-map of every function, two candidate streams (C-N from suspicion, SA-N from references), and a log of everything filtered out. Nothing is a confirmed bug. Everything is a pointer.

---

**How to use it**

Invoke with any natural phrasing — "run curious jello on this", "jello this contract", "curious jello relaxed".

Add `strict` if you have a README/spec to provide — forces docs-first ROLE/HOLDS derivation and natspec cross-checking. Add `relaxed` if you have no docs and want it to run immediately off code alone.

Paste or upload the `.sol` files. If you have a README or natspec, paste that first. That's it — the rest is automatic.
