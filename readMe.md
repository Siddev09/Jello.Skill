## CORE IDEA 

### Theme 

keep things simple and straight forward to reduce false positives

- idea of 2 agents
- 1 to solve the problem of understanding the contract with docs 
- which creates a very deep mind map for the auditor and force on 

```
NUANCES IN CODE WIERD CALCULATION , OR FLOWS WHICH IS USER RECHABLE OR EDGE CASE CAN OCCUR 

- EXPLAINATION OF THE contract code each 1 by 1 WITH DOCS EACH FUNCTION IN depth within 3 lines -- point out most wierd nuances you discovered    
- HEAVY INVESTMENT ON EDGE CASES AROUND NUANCES 
- FUZZING MODULES INTEGRATIONS -- INVARIANTS BASICALLY , 
- EXTRACT UNKNOWN OR UNWRITTEN INVARIANTS WHICH IS NOT TOLD BY THE DOCS BUT IF THEY GO WRONG MAJOR HACK CAN BE DONE - EXTRACT THOSE INVARIANTS BUT NOT OBVIOUS INVARIANT LIKE REMOVEING ANYTIHING MIGHT CAUSE CATASTROPHY WHICH IS OBVIOUS BUT SOME RULES WHICH SEEMS RIGHT AND RECHABLE BUT IF THE USER MESS WITH IT IT MIGHT GO WRONG WHICH ALSO SEEN AS NUANCE 
- CRITICAL CRITISISM 
- HEAVY INVESTMENT ON EDGE CASES AROUND NUANCES 
- FUZZING MODULES INTEGRATIONS -- INVARIANTS BASICALLY 
- CRITICAL CRITISISM -- WITH PROOFS WHY ITS WRONG 
- TURST ROLES - EXCLUSION 
- INTENTED FLOWS -- WHICH MIGHT MITIGATED BY THE DOCS 
- EIP SPECIFICS - IF ANY EIP RULES ARE VIOLATED ANYWHERE each param only major differences which comes under nuances , not low tier mitigatable differences or design choices 
  
  
```

- 2nd agent to actually audit and check what agent 1 hinted or told where the gaps are 
	and without amplification of bug try to criticize the gaps until they are proven bug by using Socratic questioning Feynman skill and check what is the full extent of this bug as per the docs.
- every finding is invalid in the start now duty is to make them completly invalid if the finding still stands the the real bug

```

- judging criteria -- as per the famous audit platforms encode there rules so the judging can be done correctly 

```

---

what i need to add 

- i need Feynman/Socratic/math/numerical/economic/periphery/ module from pashov skill
- critical plus negative judgements not positive : how do i invalidate this finding as per scope or docs and after all attempt if that stands that's a valid bug
- cca-uniswap-attack-vector : useful in defi contracts 
- quill sheild specific imporoved verisons of references of attack vectors 


---

what i don't need 

- amplification of bugs 
- chaining to get more vulnerabilities down/up stream 
- going too off from the scenario or validity of the bug 
- Leads -- of info bugs
- i want to reduce false positives by criticism and intended flow  





---
# Two-Agent Audit System — Design Report

**Purpose:** Reduce false positives through separation of understanding and judgment.  
**Philosophy:** Agents must earn the right to call something a bug, not assume it.

---

## Core Principle

Most audit false positives come from one mistake: jumping from "this looks weird" to "this is a bug" without a destruction phase in between. This system forces a hard boundary between those two things.

---

## Agent 1 — Reconnaissance Agent

### Role

Understand the protocol deeply. Do not find bugs. Produce a structured map of everything suspicious, unusual, or edge-case-prone.

### Inputs

- All in-scope `.sol` files
- Protocol README / docs / spec (ingested **before** code, not alongside)
- Known EIP specs if any EIP is implemented

### Outputs Only (strictly no bug claims)

|Output Type|Description|
|---|---|
|**Nuance**|Weird calculation, unusual flow, non-obvious behavior|
|**Hidden Invariant**|An unstated rule that must hold for the system to be safe|
|**Trust Assumption**|Who can call what, and under what conditions|
|**Suspicious Flow**|A user-reachable path that touches funds, state, or permissions in a non-obvious way|
|**EIP Deviation**|A parameter or behavior that differs from spec in a material way (major deviations only, not design choices)|

### Output Gate — Every item must answer all three:

```
1. Why is this weird?
2. Why is it user-reachable?
3. Why does it matter?
```

Cannot answer all three → **discard immediately**.

### Pre-Send Filter to Agent 2

Before passing anything forward, require at least one YES:

```
[ ] User reachable?
[ ] Touches funds?
[ ] Touches accounting?
[ ] Touches permissions?
[ ] Touches state transitions?
[ ] Touches fund-losing flows?
```

All NO → discard. Does not reach Agent 2.

### What Agent 1 Produces Per Item

```
TYPE: [Nuance | Hidden Invariant | Trust Assumption | Suspicious Flow | EIP Deviation]

Location: Contract.sol :: functionName() :: line N

Observation: [What is happening in code]
Doc context: [What the README/spec says about this area]
Why weird: [One sentence]
User reachable: [Yes — via X path]
Matters because: [Touches funds / accounting / permissions / state]
```

---

## Agent 2 — Destruction Agent

### Role

Take every item from Agent 1 and attempt to kill it. Only items that survive destruction become findings.

### Primary Question (asked first, always)

```
Why is this NOT a bug?
```

Not: "How can this be exploited?"

### Destruction Pipeline (ordered, all steps mandatory)

```
Step 1 — Reachability
  Is the code path actually reachable by a non-privileged user
  under realistic conditions?
  FAIL → Discard

Step 2 — Docs / Spec Check
  Does the README or spec explicitly allow or describe this behavior?
  Intended behavior → Discard

Step 3 — Trust Model Check
  Is this callable only by a trusted role?
  Is the trust boundary documented?
  Trust-excluded → Discard

Step 4 — Economic Sanity Check (mandatory for DeFi)
  Provide a concrete number:
    "Attacker profits X under condition Y at scale Z"
  Cannot produce a number → Discard

Step 5 — Counterargument Check
  Produce the THREE strongest arguments against this finding:
    (a) Protocol team's defense
    (b) Judge's defense
    (c) Why this could be intended design
  All three must fail for the finding to survive.
  Strongest counterargument wins, not weakest.

Step 6 — Contest Scope Check
  Is this within the declared contest scope?
  Is the affected contract in scope?
  Out of scope → Discard

Step 7 — Judge Likelihood Check
  Encode contest platform rules:
    Cantina: severity thresholds, duplication policy
    Sherlock: admin trust exclusions, duplication rules, known judging patterns
    Code4rena: QA vs Medium criteria
  Would the judge accept this at the claimed severity?
  Likely rejected → Downgrade or Discard
```

### Reasoning Engine (inside Agent 2, not separate agents)

**Feynman Module** — Checks whether the finding is understood:

```
Can you explain the bug in 5 sentences of plain language?
What assumption breaks?
Who loses money?
How exactly does value move from victim to attacker?
```

If the agent cannot explain it simply, the finding is not understood. Discard.

**Socratic Module** — Checks whether the finding survives criticism:

```
Why might this be intended?
What assumption am I making that could be wrong?
What if the docs allow this?
What if the trust model allows this?
What if impact is too small to matter?
What would the judge say against this?
```

Hard exit: **two outcomes only** — Prove Invalid or Confirm.  
No generating alternative attack paths. No speculation. Restrictive by design.

**Numerical Module** — Required for any economic finding:

```
Exact profit: $X or X tokens
Condition required: Y
Scale: Z (dust / significant / protocol-breaking)
Likelihood of condition: [always / common / rare / requires adversarial setup]
```

No number → not emitted.

### Attack Vector Libraries (CCA, Uniswap, QuillShield, DeFi patterns)

Used only **after** the nuance is identified and filtered. Pattern-matched against the specific suspicious flow only, not the whole contract. Never used as the starting point.

### Emission Criteria — All five must be satisfied

```
[ ] Reachability proven (not assumed)
[ ] Impact proven with concrete number or state change
[ ] Docs checked — not intended behavior
[ ] Trust model checked — not excluded
[ ] All three counterarguments failed
```

Missing any one → **not emitted**.

### Output Format Per Finding

```
FINDING

Title: [Short descriptive title]
Severity: [Critical / High / Medium / Low]
Location: Contract.sol :: functionName() :: line N
Confidence: [High / Medium] — never emit Low confidence findings

Root Cause: [One sentence, code-level]
Attack Path: [Numbered steps, concrete]
Economic Impact: [Exact number or bounded range]
Proof: [Code snippet or trace]
Fix: [Minimal change only]

Counterarguments Attempted:
  (a) [Protocol defense — failed because X]
  (b) [Judge defense — failed because X]
  (c) [Intended design — failed because X]

Docs reference: [Quote or "not addressed"]
Trust model: [Excluded roles and their scope]
Contest scope: [Confirmed in scope]
```

---

## Full Pipeline

```
README / Docs
    ↓
Agent 1 ingests docs first
    ↓
Agent 1 reads code
    ↓
Nuance / Hidden Invariant / Suspicious Flow identified
    ↓
Agent 1 Output Gate (why weird + reachable + matters)
    ↓
Agent 1 Pre-Send Filter (touches funds/state/permissions?)
    ↓
Agent 2 receives filtered items
    ↓
Feynman: Can I explain this simply?
    ↓
Reachability → Docs Check → Trust Check
    ↓
Economic Sanity (concrete number)
    ↓
Socratic: Prove invalid or confirm (two outcomes only)
    ↓
Counterargument Check (strongest 3, not weakest 3)
    ↓
Contest Scope + Judge Check (platform-specific rules)
    ↓
All five emission criteria satisfied?
    ↓
Emit Finding / Reject
```

---

## What This System Deliberately Avoids

|Avoided Behavior|Reason|
|---|---|
|Bug chaining downstream|Amplification, not signal|
|Upstream root cause speculation|Goes off scenario|
|Information bugs / leads|Noise|
|Positive reinforcement of findings|Creates overconfidence|
|Open-ended Socratic exploration|Generates alternative attack paths, the opposite of what we want|
|Attack vector library on full contract|Carpet-bomb false positives|
|Emitting without a concrete number (DeFi)|Unverifiable impact|
|Generic counterarguments|Weak counterarguments create false confidence|

---

## Key Design Decisions

**Docs before code.** Agent 1 must read the spec before touching the contract. A nuance spotted without knowing the documented behavior is noise.

**Agent 1 cannot emit bugs.** Hard constraint. The moment an agent can call something a bug while also identifying it, the destruction phase is skipped by habit.

**Strongest counterargument wins.** Not weakest. Many findings survive three weak counterarguments and get emitted as false positives. The question is whether the strongest possible defense fails.

**Socratic has a hard exit.** Two outcomes. "Prove invalid" or "confirm." Not "what else could be true." The moment it becomes generative it creates the amplification the system is designed to kill.

**Platform-specific judge encoding.** Cantina, Sherlock, and Code4rena have materially different rules on admin trust, duplication, and severity thresholds. Generic "would a judge accept this" is insufficient.

**No number, no DeFi finding.** If economic impact cannot be bounded with a concrete number or range under stated conditions, the finding is not ready to emit.

---

_This design is optimized for precision over recall. It will miss some bugs. That is the correct tradeoff for contest submission where false positives cost credibility and judging time._


---

### how to save tokens 

Haiku is fine for pattern recognition and rule-following. It struggles with multi-step reasoning chains and subtle economic logic. Keep that boundary in mind.

**Token reduction techniques that don't hurt precision**

Structured output over prose. Instead of asking Claude to explain its reasoning in paragraphs, give it a fixed schema it fills in. Prose costs 3-5x the tokens for the same information density.

```
# bad — costs tokens
Explain why this is suspicious and what it touches.

# good — costs nothing extra, same info
SUSPICIOUS: [y/n] | TOUCHES: [funds/state/perms/none] | REACH: [direct/indirect/no]
```

Numeric verdicts over text verdicts. At every gate, use 1/0 or y/n not "yes this is reachable because..." The reasoning is only needed when the answer is borderline.

**Chunking strategy**

Don't feed the whole contract to one call. Split by contract file, run Agent 1 per file, aggregate. Smaller context = cheaper per call, and Haiku performs better on focused context than sprawling one.

**What to strip from SKILL.md for Haiku**

Remove all explanatory prose from the skill itself. Haiku doesn't need to understand why a rule exists, it needs to follow it. Every sentence in SKILL.md that starts with "because" or "this is important because" is tokens burned at read time on every single run.

```
# strip this
# Agent 1 must read docs before code because otherwise it will 
# find things and then rationalize them which causes false positives

# keep this
RULE: Read docs before code. Order is mandatory.
```

**Prompt compression for the two-agent system**

Agent 1 prompt can be collapsed to a reference table:

```
OUTPUT TYPES ONLY:
N = Nuance | I = Invariant | T = Trust | F = Flow | E = EIP

GATE (all 3 required):
W = Why weird? | R = User reachable? | M = Matters to funds/state/perms?

FORMAT:
[TYPE] [LOC] [W:...] [R:...] [M:...]
```

Agent 2 destruction pipeline as a checklist Claude marks inline:

```
[ ] Reachable
[ ] Not in docs
[ ] Not trust-excluded  
[ ] Number: $X at condition Y
[ ] 3 counterargs failed
EMIT: y/n
```

That entire pipeline takes maybe 200 tokens to run per finding instead of 800.

**Biggest single token save**

Don't ask Haiku to generate counterarguments from scratch. Pre-write the three standard counterarguments as templates in `references/counterargument.md` and tell Haiku to evaluate each as true/false with one sentence. Generation is expensive, evaluation is cheap.

```
# expensive
Generate three counterarguments against this finding.

# cheap
Evaluate each. True/False + one sentence why.
CA1: This is documented intended behavior → [T/F]: 
CA2: Only callable by trusted role → [T/F]: 
CA3: Impact is dust-level under realistic conditions → [T/F]: 
```

**Where not to cut**

Don't compress the emission criteria gate. Those five checks are your entire false positive defense. Running them costs maybe 100 tokens. Skipping or collapsing them is where noise comes back in.

Don't compress the docs check. That one check kills more false positives than anything else in the system. Keep it explicit even if everything else is terse.

---
That reframe is correct and it's actually a stronger design than most AI audit tools.

You're not building a bug finder. You're building a **validity stress tester**. The output isn't "here are bugs" — it's "here are the things that survived destruction." That's a fundamentally different and more defensible artifact.

---

**What this means for the skill design**

Agent 1 isn't really a recon agent. It's a **suspicion generator**. Its only job is to produce candidates for destruction. Lower bar for entry, higher bar for exit.

Agent 2 isn't an audit agent. It's a **falsification machine**. Its default answer is invalid. It only changes that answer when it runs out of ways to invalidate.

The asymmetry matters. Most AI audit tools default to "this looks like a bug, let me verify." Yours defaults to "this is not a bug, prove otherwise." That inversion is where your false positive reduction actually comes from.

---

**Practical implication for writing the skill**

Agent 2 should never use the word "finding" internally until step 7. Before that it only uses "candidate." Language shapes behavior in LLMs. If the prompt says "evaluate this finding" it's already biased toward emission. If it says "attempt to invalidate this candidate" the destruction is genuine.

Every gate in Agent 2 should be written as a rejection criterion, not a validation criterion.

```
# wrong framing — validates
Is this reachable? Yes → continue

# right framing — destroys
Can I prove this is NOT reachable? 
Yes → discard
No → candidate survives this gate only
```

Small difference in wording. Large difference in how Haiku executes it.

---

**The one risk in pure negative criticism mode**

Haiku will sometimes invalidate things correctly for the wrong reason — it finds a weak counterargument, marks the gate as failed, and discards a real bug. You won't see those because they never surface.

Mitigation: keep a **discard log**. Every rejected candidate gets one line: what it was and which gate killed it. You skim that manually in 2 minutes. Real bugs discarded for wrong reasons show up there. This costs almost no tokens and prevents the system from being a black box.

That's the only structural addition I'd make to your current design.
