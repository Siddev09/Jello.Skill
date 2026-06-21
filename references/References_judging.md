# Unified Vulnerability Severity Framework
*Merged from: Cantina Severity Classification, Sherlock Criteria for Issue Validity, and Immunefi-style Smart Contract + Blockchain/DLT classification tables.*

This is a decision tool, not a fourth independent rubric. Where the three sources agree, that's the default rule. Where they conflict, both positions are kept visible so you can pick the right one per program instead of the framework silently choosing for you.

No classification rule below has been altered, softened, or merged across platforms — every threshold, exclusion, and doctrine from all three sources is preserved exactly. The only change from a plain merge is *order*: the decision path you actually run during triage is now §1, and the full reference material it points to follows after, so you're never holding five rubrics in your head at once while judging a single finding.

---

## 1. Fast Path — run this first, in order, on every finding

1. **Exclusion pass** — check §5. Matches an exclusion and none of the listed exceptions apply → stop here: Informational/Invalid.
2. **Impact classification** — match the outcome to a row in §2 using the Immunefi/Cantina example columns ("theft," "temp freeze," "permanent freeze," "griefing," "DoS," etc.).
3. **Quantify** — apply the §4 thresholds. No quantifiable loss and no core-functionality break → cap at Low.
4. **Duration check** — if it's a freeze/DoS finding, run the §4 ladder.
5. **Invariant check** — if quantification caps it below Medium, check whether it breaks a README-stated invariant (§6); if so, floor it at Medium.
6. **Precondition check** — "extensive external conditions" or "elevated privilege required" pulls a High down toward Medium; trivial/no-precondition pushes a Medium toward High.
7. **Platform lens** — relabel for the target program: Sherlock has no Critical/Low tier (fold Critical→High, Low→Invalid-unless-invariant); Cantina/Immunefi keep all five tiers as-is.

If a finding resolves cleanly by step 3 or 4, you don't need to read further — §2 onward exists to justify the call and to handle the edge cases steps 1–7 don't fully settle on their own.

---

## 2. Master Tier Table

| Tier | Cantina | Sherlock | Immunefi (Smart Contract) | Immunefi (Chain/Consensus) | **Unified rule** |
|---|---|---|---|---|---|
| **Critical** | Catastrophic damage + high likelihood, minimal interaction needed. Permanent asset loss, network shutdown, hard fork, insolvency, governance takeover. | **No formal Critical tier.** Catastrophic cases are graded **High** — Sherlock only judges High/Medium. | Direct theft of funds/NFTs, permanent freezing, governance manipulation, insolvency, unauthorized mint/burn. | Direct theft via consensus manipulation, permanent protocol-level freeze, total network shutdown, hard fork required, chain split. | Direct + irreversible loss/freeze of funds, OR breaks solvency/consensus, with no significant precondition, blast radius = protocol-wide. **On Sherlock, downgrade the label to "High" — the impact doesn't change, only the tier name does.** |
| **High** | Significant but not catastrophic. Medium-high likelihood, some interaction/conditions required. Temp freeze, unclaimed yield theft, elevated-privilege exploits w/ high impact. | Direct loss **without extensive external-condition limitations**, and loss is **significant**: **>1% AND >$10** of principal, yield, or protocol fees. | Temporary freezing of funds/NFTs, theft of unclaimed funds, permanent freezing of *unclaimed* funds, high-impact oracle manipulation. | App-level DoS that crashes nodes, temporary freezing of network transactions, temporary freezing of funds (liveness failure). | Direct/quantifiable loss clearing the **1%/$10** bar, OR temporary freezing of principal/yield, OR unauthorized minting/oracle manipulation enabling theft — with limited precondition complexity. |
| **Medium** | Moderate damage, limited to specific users/conditions, medium likelihood. Gas theft, griefing w/o profit motive, unintended behavior w/o direct financial risk. | Loss requiring **specific conditions/state**, or **highly constrained** loss, OR breaks core contract functionality. Loss is **relevant**: **>0.01% AND >$10**. | Theft of gas (unbounded loops), gas-limit/OOG bugs, DoS (gas exhaustion, block stuffing), no-profit griefing. | App-level DoS on a node subset, unintended behavior from protocol quirks/edge consensus rules, timestamp-manipulation attacks, minor reorg exploits. | Constrained/conditional loss clearing **0.01%/$10**, OR DoS/griefing with no attacker profit, OR core-function breakage without fund loss, OR funds locked **1 day–1 week** (longer if it also hits a time-sensitive function — see §4). |
| **Low** | Minor damage, non-critical functionality, low likelihood, heavy interaction required. Fee-param drift, non-critical UI/info disclosure. | **Not a formal payable tier.** Anything below the Medium threshold, fully admin-recoverable, or self-inflicted-only is **Invalid** — *unless* it breaks an explicit README invariant, in which case it's Medium regardless of $ size. | Failure to deliver promised yield/APY, uninitialized storage variables. | <25% node shutdown (non-critical), fee miscalculation with no economic impact, P2P gossip propagation delays. | Real issue, no fund-loss path, OR below the $10/0.01% floor, OR fully recoverable by admin/redeploy. **Treat as Low on Cantina/Immunefi. Treat as Invalid on Sherlock unless an explicit README invariant is broken.** |
| **Informational** | Bottom-right cell of the impact×likelihood matrix (Low×Low). | The large "not considered valid" list (§5) plus anything sub-threshold with no invariant breach. | Not a separate tier — CVSS is used for anything outside the listed categories. | Same as SC column. | Best-practice/code-quality note with no demonstrated path to loss or functional break. |

---

## 3. The core conflict you need to resolve per-contest: does likelihood matter?

- **Cantina**: severity is explicitly `f(impact, likelihood)` via a 4×4 matrix — a Critical-impact bug with Low likelihood gets capped at **Medium**.
- **Sherlock**: states outright that *"Likelihood is not considered when identifying the severity and the validity of the report."* Only loss-conditionality and the $/% thresholds matter.
- **Immunefi-style**: implicitly impact-driven — the example tables describe outcomes, not probability of occurrence.

**Default when you don't know the house rule:** classify by impact outcome + the Sherlock $/% thresholds first (they're the only concrete numbers across all three docs, and the most conservative). Only apply a likelihood discount if the specific program states it uses a Cantina-style matrix.

---

## 4. The only hard numbers in any of the three sources (all Sherlock)

- **High**: loss **> 1% AND > $10** of principal, yield, or fees, with no extensive precondition.
- **Medium**: loss **> 0.01% AND > $10** of principal, yield, or fees, OR breaks core functionality outright.
- **Replayable small losses**: a single 0.01% loss that can be replayed indefinitely is treated as a 100% loss → Medium or High depending on remaining constraints.

**DoS / asset-locking duration ladder** (Sherlock's rule, useful as a default everywhere since none of the others give a number):

| Lock duration | Hits a time-sensitive function? | Severity |
|---|---|---|
| < 1 week, single occurrence | No | Low / Invalid |
| < 1 week, single occurrence | Yes | Medium |
| ≥ 1 week | No | Medium |
| ≥ 1 week | Yes | High |
| Long single attack (>2 days) needing only 2–3 reps to reach 7 days | — | Treat as valid on the High track |

---

## 5. Merged "Out of Scope / Invalid by default" checklist

Run every candidate finding through this before assigning *any* severity — both Cantina and Sherlock independently exclude almost the same things:

**Admin/trust-related**
- Admin error (wrong call order, wrong params) — invalid unless it's a *bug in an admin function's implementation*, not the call itself.
- Malicious or compromised admin/owner — out of scope unless explicitly in-scope.
- Internal protocol roles are trusted by default; untrusted only if the README says so, or the role is acquirable without permission (e.g. by paying a fee).
- Admin/contract address getting blacklisted by a third party and breaking protocol function — invalid, *unless* an attacker can weaponize that blacklisting against the protocol.

**Token/asset-related**
- Non-standard ("weird") ERC20 behavior — invalid unless explicitly named in-scope. Decimals 6–18 are never "weird."
- Accidental direct token transfers into in-scope contracts — user error if it only hurts the sender; valid if it harms the protocol or other users.
- Loss of airdrops/rewards outside the original protocol design — invalid.

**Process/general**
- Gas optimizations, incorrect event values, missing zero-address checks, storage-gap gaps in simple inheritance trees (valid only in highly branched inheritance), stale-price/round-completeness recommendations (valid only for pull-based oracles like Pyth where staleness isn't checked at all).
- Theoretical issues without a PoC, social engineering, physical-access attacks, rate-limiting, SSL/TLS/email-config opinions, automated-scanner output, sequencer-downtime assumptions, chain re-orgs at the consensus level, EVM-opcode-availability concerns (compilation flags solve this).
- Design-philosophy disagreements on a permissionless protocol — informational, not a bug.
- Front-running an initializer with no irreversible damage (just redeploy) — invalid.

---

## 6. Doctrines to apply consistently across programs

- **README > code comments.** If they conflict, README wins. A judge may rule a code comment stale and fall back to default rules.
- **Invariant rule**: if the README explicitly states an invariant ("X can only be called once"), breaking it is **at least Medium** regardless of stated impact, as long as it doesn't conflict with common sense.
- **Front-running downgrade on private-mempool chains**: High → Medium, Medium → Invalid. The report must explain how the front-run happens *unintentionally* — "attacker watches the mempool" is not sufficient on a chain with a private mempool.
- **Scope inheritance**: parent contracts of an in-scope contract are in scope. A library bug reachable from an in-scope contract is in scope. A bug in an out-of-scope contract is never valid, even if it's sitting in the same repo.
- **PoC requirement**: expected for reentrancy, gas/DoS claims, precision-loss claims, and any non-obvious/complex path or non-trivial input constraint. No PoC on these = high risk of being judged invalid or downgraded — this maps directly onto your existing Foundry POC workflow, so treat "no working POC" as a blocker before submission, not a nice-to-have.
- **Duplication / root-cause grouping** (Sherlock-specific, but useful for organizing your own report even off-platform): two findings share a root cause if they're the same logic mistake, the same conceptual mistake (e.g. "different untrusted admins can steal funds"), or fall in one of: slippage protection, reentrancy (further split by same-function / cross-function / cross-contract / read-only / cross-chain), access control, front-run/sandwich. Grouped findings get the *highest* severity among the group.

---

## 7. One thing this framework can't resolve for you

None of the three sources defines a numeric bar for "significant blast radius" or "widespread compromise" at the Critical tier — that judgment stays qualitative everywhere. Use §2's Critical row as a pattern-match (theft/freeze/insolvency/consensus-break, no precondition, protocol-wide) rather than expecting a formula.
