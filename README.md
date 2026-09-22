# Jello


## What It Solves

Most AI-assisted review tools jump straight to flagging issues — and in doing so, flood the auditor with false positives that waste review time and quietly narrow their thinking to whatever the tool already flagged, instead of the full range of attack vectors a codebase actually has.

Jello does the opposite by default: it builds understanding first. Its modes walk you through what a protocol is and how it actually works — flows, invariants, math, integration surfaces — so you form your own mental model and construct your own attack scenarios, rather than anchoring on someone else's signal.

RAGE mode is the exception, by choice: a separate, opt-in trigger that generates an initial signal by checking the codebase against a library of historically-seen gap patterns — useful as a starting point, never a substitute for your own review.



---

A single-agent Claude skill for smart contract review. It reads code (and docs, if available) and produces a structured **understanding map** — contract roles, user flows, invariants, math, and integration surfaces — as a pre-audit comprehension pass, not a bug-finding tool. No findings, no severity, no exploit claims.

## The Four Passes

| Pass | Trigger | Produces |
|---|---|---|
| **Docs Deep Dive** | `docs pass` | Per-contract roles, protocol summary, user flows, docs-stated known issues, consolidated invariants |
| **Periphery & Uniswap Crawl** | `periphery check` | What router/hook/pool-manager surfaces call, expect back, and fire on |
| **Integrator Crawl** | `integrator check` | How external contracts connect into the base contract — entry points, approvals, callbacks |
| **Math Pass** | `math pass` | 3-phase extraction → dedupe → consolidated model of invariants, encoding, constraints |
| **All four, in order** | `curious jello` / `run jello` | Full run |

**Modes:** `strict` checks invariants against supplied docs (asks for them if missing). `relaxed` infers everything from code alone. Applies to Docs Deep Dive and full runs; Math Pass and Crawl read docs opportunistically without asking.

## RAGE MODE

A separate, opt-in trigger — `jello rage` — that inverts the skill's default posture. Instead of understanding only, it runs a full attacker-mindset pattern match against a 14-category audit checklist (access control, reentrancy, oracle manipulation, math, DoS, governance, front-running/MEV, plus ERC-3643/4626/7518 and Uniswap V4 Hooks).

Every reported lead is proof-gated: a match is only surfaced once the exact code path, function, and trigger condition are traced and confirmed. Unproven suspicions get one summary line; anything short of that is left out — no hedging, no speculation. This is the one place the skill's "no findings" rule is deliberately suspended, and that exception never carries into a normal run.

## Usage

Attach your contracts (and docs, if any) and say a trigger phrase — automatic matching also works if your request clearly describes the task. Example: `"run the math pass on these"` → Math Pass only, no docs question. Every response opens with `SCOPE:`/`MODE:` so you know which run you got.

## In Short

A context-building skill, not a bug-finding one — a deterministic map of the territory so review time goes into judgment, not orientation. RAGE MODE is the opt-in switch for proof-gated leads instead.
