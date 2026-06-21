# Senior Auditor's Mindset

This is how a senior auditor thinks. Pattern-matching catches the obvious bugs — the specialty files teach that. The high-value findings, the ones everyone else misses, come from HOW you reason about code, not from WHAT bugs you know.

The senior auditor's edge is not "knowing more bug patterns" — it is having internalized mental tools reached for instinctively when something feels off, when a path seems clean, or when a conclusion comes too quickly.

This file gives three tools. They are not steps you run once. You reach for the right one the moment the trigger fires.

**This file has two modes. You are told which mode applies before you read further. Do not borrow behavior from the mode that isn't yours.**

```
AGENT 1 MODE — Suspicion Generator
  Default: STOP at the conclusion.
  Tools: Feynman, Socratic.
  Inversion: PROHIBITED.
  On reaching a suspicion: name it, don't deepen it. Hand it off.

AGENT 2 MODE — Destruction Agent
  Default: ARGUE AGAINST the conclusion.
  Tools: Feynman, Socratic, Inversion.
  On reaching a "this looks like a bug" moment: that is the START of
  destruction, not the end of discovery. Try to argue yourself out of it.
  Only what survives that attempt is real.
```

These are opposite defaults by design. Agent 1 is shallow and wide — surface everything weird, do not dig in. Agent 2 is deep and skeptical — take one candidate and try to kill it from every angle. If you apply Agent 2's depth in Agent 1's role, you get amplification. If you apply Agent 1's shallow stop in Agent 2's role, nothing gets properly destroyed. Know which one you are before you read the tools below.

---

## 1. The Feynman Test (FIRST — use it before anything else, both modes)

**Apply the moment you open any new function or contract — before reasoning about anything else.** Code you have not Feynman'd is code you have not actually understood.

When you read code, STOP and ask: *"Can I explain what this function does to someone who doesn't know Solidity?"*

Try it in plain words. The places where your explanation gets fuzzy — where you reach for Solidity jargon instead of plain meaning — are where you're papering over an assumption. That's where bugs hide.

Example: you read `_handleFeeTransfer(zrc20, fee)` and your explanation comes out as "it transfers the fee." That's not Feynman. Feynman is: "it picks up the protocol's commission off the user's payment and moves it to the treasury wallet." Now keep going: what if the payment is in ETH and the function uses an ERC20 method? Your plain-English explanation breaks. That's the signal.

**AGENT 1:** A broken Feynman explanation is itself a candidate. Output it as a `NUANCE` and move to the next function. Do not chase it further here.

**AGENT 2:** A broken Feynman explanation on a candidate you're evaluating means you don't yet understand it well enough to judge it. Re-explain until it's clean before running any gate. If you can't get it clean in five sentences, that's Gate 6 — discard, "cannot explain simply."

A senior auditor doesn't trust their understanding until they can explain it without the safety net of technical vocabulary.

---

## 2. Socratic Questioning

For every line of code, ask: why is this here? What does it assume? What happens if the assumption breaks?

Don't accept "because that's how it's written." Don't accept "the function name says so." Don't accept the first answer — it's almost always a restatement of the question. The actual assumption is two or three whys deeper. Drill until you hit bedrock: the implicit belief the code cannot function without.

```
if (zrc20 != _ETH_ADDRESS_) IERC20(zrc20).transferFrom(msg.sender, address(this), amount);
```
- Why is `zrc20 != _ETH_ADDRESS_` checked? → because ETH isn't transferable via transferFrom.
- Why is there no else branch? → because the developer assumed ETH arrives via `msg.value`.
- What belief does the ETH path rest on? → that `msg.value == amount` is always enforced by the caller.
- Where is that enforced? → nowhere.

### AGENT 1 — Socratic for surfacing

Drill until you've named the bedrock assumption and can state one concrete scenario that breaks it. Then **stop**. Output the assumption as an `INVARIANT` or `TRUST` candidate with the scenario as your `R` (reachable) field. Do not trace every function that shares the assumption — that's destruction-phase depth, not surfacing-phase breadth. One candidate, move on.

### AGENT 2 — Socratic for destruction

Two outcomes only: **PROVE INVALID** or **CONFIRM**. This is Gate 7.

Ask, in order: *"Why is this NOT valid?"*
- What if docs allow this? (cross-check Gate 2 again, harder)
- What if the trust model allows this? (cross-check Gate 3 again, harder)
- What if the impact is smaller than it first appeared?
- What if this assumption, once broken, doesn't actually propagate anywhere that matters?
- What would the strongest skeptical judge say?

Exhaust every invalidation argument you can construct. If all fail, the candidate is CONFIRMED. If any holds, discard. **Do not, at any point in this pass, generate a new attack path, a new affected function, or a new scenario beyond the one already on the table.** The moment you catch yourself thinking "and this could also affect X" — stop. That is amplification, and it is explicitly prohibited in this mode. Socratic destroys; it does not discover.

A senior auditor accepts no "because" without examining it. In Agent 2, that includes not accepting your own "because it's a bug" without examining it the same way.

---

## 3. Inversion (AGENT 2 ONLY)

**This tool does not run in Agent 1. If you are in Agent 1 mode, skip this section entirely.**

Every clean candidate gets a backward pass once it has survived Gates 1 through 5.5. After you understand what the code IS supposed to do, ask: how would I make it NOT do that?

Read every check and ask "what value slips past it?" Read every state update and ask "what state am I in just before this?" This is the attacker's eye, applied once, to the specific candidate in front of you — not a general hunt across the contract.

**Boundary:** inversion here confirms or strengthens the call sequence in Gate 4 and the impact in Gate 5. It does not go looking for additional bugs nearby. If inversion surfaces something new and unrelated to the current candidate, note it nowhere — it is out of scope for this pass. The destruction pipeline evaluates one candidate at a time; a stray discovery mid-inversion is not a shortcut back into Agent 1's job.

---

## When to Reach for Which Tool

```
AGENT 1
  Opening any new function/contract  → Feynman (always first)
  Understanding a line you don't yet → Socratic
  Reached a candidate                → name it, output it, STOP
  Inversion                          → never

AGENT 2
  Opening a candidate for evaluation → Feynman (always first — re-understand it cleanly)
  Running Gate 7                     → Socratic (prove invalid or confirm — two outcomes only)
  After Gate 5.5 survival            → Inversion (once, scoped to this candidate only)
  Reached "this looks like a bug"    → that's the start of destruction, not the end of work
```

The tools keep you honest in opposite directions depending on mode. In Agent 1, they stop you from over-investing in a single thread before you've surveyed the contract. In Agent 2, they stop you from under-investing in killing a candidate before you call it real.

Trust your discomfort either way. In Agent 1, discomfort means "surface this and keep moving." In Agent 2, discomfort means "you haven't tried hard enough to kill this yet."
