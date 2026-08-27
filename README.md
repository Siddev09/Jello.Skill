# Curious Jello 

## What It Is
- A single-agent Claude skill for smart contract review
- Converts raw contract code (plus docs, if available) into a structured **understanding map**
- Runs as a **pre-audit comprehension tool**, not an audit tool
- Explicitly does **not**: hunt for bugs, flag findings, assign severity, or make exploit claims

## Core Idea
- Audit time is often lost re-deriving context on the fly
- This skill front-loads that work into a clean, repeatable artifact
- Goal: start bug-hunting from a solid mental model instead of building one mid-review
- Output is a map of what the code/docs *say* — never a pointer to a problem

## The Four Passes

### 1. Docs Deep Dive
- Per-contract role summaries (what it holds, how it relates to other contracts)
- Protocol purpose and core mechanism, restated in plain language
- User-facing flows (deposit, withdraw, swap, mint, redeem, etc.), step by step
- Known Issues — relayed only from the docs themselves, never invented
- Consolidated invariants:
  - Internally gathered, then deduplicated down from raw candidates
  - Only the final small set is shown to the user
  - Each one tagged as `doc reference` or `inferred from code`

### 2. Periphery & Uniswap Crawl
- Scoped to contracts touching routers, hooks, pool managers, or fund-moving library calls
- For each: what it calls, what it expects back, what lifecycle point it fires at
- Uses attacker-voice reference material only as **vocabulary**, not as a checklist

### 3. Integrator Crawl
- Maps how external contracts plug into the core/base contracts
- Covers: entry points, call order, approval relationships, callback relationships
- States plainly what the base contract assumes about its caller

### 4. Math Pass
- 3-phase protocol: per-contract extraction → dedupe → consolidated report
- Extracts: invariants, directional encoding, state transitions, constraints
- Cross-contract math flows also noted (data passed between contracts, encoding consistency)
- Each concept ends with "tracing questions" for later investigative use

## Modes
- **Strict mode** — invariants checked against supplied docs; asks for docs if none given (DOCS/FULL scope only)
- **Relaxed mode** — everything inferred from code alone, no docs question asked
- MATH and CRAWL scopes read docs opportunistically but never block on them

## How It Helps
- Gives a **fast, consistent onboarding pass** on unfamiliar codebases
- Well-suited to DeFi categories in your audit portfolio: AMMs, lending protocols, stablecoins, CDPs, cross-chain bridges
- Strict/relaxed split keeps invariant claims honest about their source — useful when citing invariants in a report
- Refusing to speculate on exploits keeps it a **safe first artifact** that won't bias later bug-hunting
- Precise vocabulary (approval abuse, callback grief, math precision) makes write-ups audit-standard without making accusatory claims
- Used as the first stage of a two-skill pipeline with `solidity-auditor` in past audits (e.g. Arcadia)

## In Short
A **context-building skill**, not a bug-finding one — hands you a deterministic, well-formatted map of the territory so review time goes into judgment, not orientation.
