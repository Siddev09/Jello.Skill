# Audit Findings Checklist

---

## 1. Access Control & Privilege
- [ ] Anyone can call a "logging" or "hook" function meant to be internal-only (e.g. `DefiSaverLogger.Log`, Umbra hook receivers) — check callback/log functions for missing `onlyX` modifiers.
- [ ] Privileged role removal is broken — `setOwners()`-style functions that add new owners but never clear `isOwner` for removed ones. **Check: can an old admin/owner still act after "removal"?**
- [ ] Permission system too coarse — any authorized address can call *any* restricted function instead of scoped per-function permissions; addresses can be added but never removed.
- [ ] Single admin/owner holds excessive power (upgrade logic, mint unlimited rewards, change interest/oracle models) — single point of failure, no timelock/multisig.
- [ ] KYC/whitelist-admin or similar freezing power granted without a delay — check for `TimelockController` usage before granting sensitive roles.
- [ ] Non-privileged function allows listing/whitelisting new assets when docs say only governance should (e.g. `addToken`) — **doc vs. code mismatch on auth**.
- [ ] Zero-address checks missing in `transferOwnership`, constructors, or `Router`-style constructors — could brick or effectively burn ownership unintentionally.
- [ ] Missing/incorrect `_owner` validation in constructor **and** setter can permanently lock the owner role (bad zero address, wrong address, or no way to recover) — check all paths that set an owner, not just one.
- [ ] `liquidateFrom`/"on behalf of" style functions are public when they should be internal — lets anyone act on behalf of an arbitrary account and hijack another user's position.
- [ ] Same privileged account handles both frequently-updated and rarely-updated critical parameters — no separation of duties; compromise of one key endangers everything.
- [ ] Critical one-shot admin operations (role transfer, address updates) done as a single-step `set` instead of a two-step propose/accept (approve → claim) pattern — a bad input is irreversible.
- [ ] Governance/voting **quorum can be trivially bypassed with sybil accounts** — check whether quorum is measured by *number of voters* vs. *token-weighted % of supply*.
- [ ] Voting/quorum parameters (`votingPeriod`, `votingQuorum`) validated at initialization but **not in their setter functions** — can be reset to 0 later, breaking or trivializing governance.
- [ ] Governance/registry addresses (e.g. `registryAddress`, `guardianAddress`) can be updated with **no event emitted** — silent trust-boundary change invisible to monitoring.
- [ ] A `Governor`/`Timelock` pair: verify `cancelTransaction`/`executeTransaction` can actually be called through the intended path — check for (a) missing function to ever call cancel, and (b) missing access control letting `executeTransaction` be called directly, bypassing the higher-level `execute()` gate.
- [ ] Regular proposals can include a transaction that changes `Timelock.admin` itself — a normal proposal could take over the timelock; this power should be restricted to a separate guarded path.
- [ ] `initialize()` functions on proxy/delegatecall-pattern contracts (no constructor) can be **front-run** — attacker calls `initialize()` first with malicious parameters. Check factory-deployment pattern or atomic deploy+init.

## 2. Token / ERC20 Assumptions
- [ ] Contract assumes "normal" ERC20 behavior — check for fee-on-transfer, deflationary, inflationary, or rebasing tokens breaking internal accounting.
- [ ] ERC777/callback-token compatibility — could enable reentrancy via transfer hooks.
- [ ] Token decimals assumption — `TOKEN_DECIMALS <= 18` hardcoded assumption causes underflow if violated.
- [ ] Missing `symbol()/name()/decimals()` on a custom ERC20-like token.
- [ ] `safeApprove`/`approve` return values not checked, or custom safe-transfer reimplementations that skip returndata checks (vs. OZ `SafeERC20`).
- [ ] Non-whitelisted or unvalidated collateral/asset can be used in a product, pool, or option — check whitelist enforcement at every entry point, not just the "main" one.
- [ ] `type(uint256).max` / return value of a withdraw/transfer function silently discarded, causing accounting drift.
- [ ] `transfer`/`transferFrom` return value not checked — some ERC20s return `false` on failure instead of reverting; wrap in `require()` or use `SafeERC20`.
- [ ] Tokens with **more than 18 decimals** (e.g. YAMv2 = 24) — check every place decimals are assumed ≤ 18, not just the obvious oracle/price spots.
- [ ] External protocol calls (e.g. Compound `enterMarket`/`exitMarket`) return an **error code instead of reverting** — verify every such call's return code is checked and reverted on failure.
- [ ] Parameter order mismatch between an `allowance()` check and the subsequent `safeTransferFrom()` call — args silently swapped, checking/moving allowance for the wrong owner/spender pair.
- [ ] No validation of the "transfer from" address in liquidity/deposit functions — allows stealing another user's **token approvals** by passing their address as the source while attacker is set as beneficiary/pool owner. **Always transfer from `msg.sender`, never from an arbitrary passed-in address.**
- [ ] Deposit/liquidity functions accept a **zero-amount deposit**, permanently locking a single-deposit-only pool for everyone else — require nonzero minimums.
- [ ] Value-received checks computed via balance-difference are compared to the **wrong reference value** (e.g. comparing router balance instead of the actual recipient's balance change) — always verify the diffed identity matches the identity the check is meant to protect.
- [ ] Non-compliant ERC20s that **return no value at all** on `transfer`/`transferFrom` (not just `false`) will cause a revert under strict Solidity ABI decoding — use `SafeERC20`, don't assume standard-return-type compliance.
- [ ] Fee-charging or deflationary tokens transferred into a contract as collateral/premium — received amount may be less than sent amount, causing insolvency; check for a post-transfer balance-delta verification instead of trusting the nominal amount.
- [ ] Contract always assumes a specific stablecoin is 1:1 with its peg (e.g. hardcoded "1 USDC = 1 USD") instead of querying a live oracle — a depeg event breaks every dependent calculation.
- [ ] `safeApprove`-style helpers that reset allowance to zero before setting a new value, reintroducing the very front-running race that `safeApprove` restrictions were meant to prevent — use `safeIncreaseAllowance`/`safeDecreaseAllowance` instead.
- [ ] Blacklist/denylist modifier applied to `msg.sender` and `to` in `transferFrom` but **not to `from`** — lets a blacklisted account keep moving funds via someone else's allowance.

## 3. Reentrancy & Ordering
- [ ] Modifier order wrong — reentrancy lock (`nonReentrant`) should be first/last, not sandwiched between logging/other modifiers.
- [ ] Checks-Effects-Interactions violated — state-changing external call made before events are emitted, causing **out-of-order event emission** on reentrant calls (subtle: doesn't corrupt state, but breaks off-chain monitoring).
- [ ] Hook/callback contracts invoked with attacker-controlled address and insufficient caller/param validation.
- [ ] "Recipe"/batched-action executors that grant a helper contract (e.g. flash-loan wrapper) **execution permission over a user's proxy for the whole batch duration** — a malicious call anywhere in the batch can call back in and inject arbitrary actions before permission is revoked. Use a mutex, and/or scope permission per-call rather than per-batch.
- [ ] Reentrancy in swap/trade functions lets an attacker **re-enter and execute their own trade using the in-flight tokens** before the original caller's trade settles — needs a simple reentrancy guard even when the "obvious" fund-draining path looks closed.
- [ ] `mint`/`mintMultiple`/`redeem`-style entrypoints call out to untrusted/arbitrary contracts without a reentrancy guard **and** without validating the asset/amount first — check both the guard and the input validation independently.
- [ ] Sensitive state (e.g. `grant.complete`, loan-closed flags) set **after** an external token transfer instead of before — classic CEI violation enabling double-spend-style reentry on the same function.
- [ ] Reentrancy risk from **ERC777-compliant tokens** even when the ERC20 interface itself looks safe — hooks fire on transfer regardless of the interface used to call it.
- [ ] New/pluggable adapter or module architecture where **any single approved module can access all user funds** — a newly added (or compromised) adapter inherits blanket approval; consider architecture where a central contract holds approvals and moves funds to adapters, not the reverse.
- [ ] Upgradeable/pluggable adapters callable via `DELEGATECALL` can **overwrite shared storage/logic of other adapters** — one malicious module can corrupt the whole system's behavior, not just its own slot.
- [ ] Owner/admin can **swap out a module or adapter implementation and front-run users** who are mid-transaction expecting the old logic — consider disallowing modification of existing adapters (add-new/disable-old only).

## 4. Oracle & Price Manipulation
- [ ] Flash-loan-driven manipulation of interest rates / prices by momentarily adding/removing large liquidity.
- [ ] Reliance on deprecated Chainlink API functions (`latestAnswer()`, `getTimestamp()`).
- [ ] No historical/volatility-based sanity check on oracle price deltas.
- [ ] `block.timestamp` used for critical time logic — miner manipulation risk.
- [ ] Oracle update **sandwiched atomically** within a single block: rate/weight only updates once per block, so an attacker can (1) lock in the stale rate with a tiny tx priced above the oracle update, then (2) execute a large trade at the stale rate and reverse it after the update lands — all risk-free, often flash-loan funded. Check: can a user trade at a stale rate *and* trigger the rate update in the same transaction/block?
- [ ] Event emitted **before** the state variables it's supposed to describe are actually updated — logged data becomes stale/incorrect (e.g. a "window update" event firing pre-mutation).
- [ ] Commit-reveal oracle/voting schemes where the **commitment isn't cryptographically bound to the voter** — allows a copycat to blindly replicate a target's (large-balance) commitment and later "reveal" the same values as their own, undermining the anti-herding purpose of commit-reveal.
- [ ] Chainlink-style request/callback systems where the **callback address is attacker-specifiable** — a malicious actor can front-run or replicate a legitimate request pointed at a target contract, poisoning or DOSing that contract's expected callback.

## 5. Math / Arithmetic
- [ ] Insufficient SafeMath / unchecked arithmetic in critical trade/curve calculations.
- [ ] Rounding to zero when `duration > reward` (integer division) — silently zeroes out reward distribution.
- [ ] Unsafe cast between `uint` and `int` without range check.
- [ ] Off-by-semantics: `_min*`/`_max*` implemented as **exclusive** bounds when the codebase/docs imply inclusive — check every `<`/`<=` boundary in financial math.
- [ ] Borrow-rate / rate-per-block formulas hardcode "blocks per year" — breaks if block time changes or contract is redeployed on another chain.
- [ ] Curve/AMM math not fuzz/property tested — pathological parameter combos untested.
- [ ] Overflow-prone casts feeding a mint/burn pre-transfer hook — can let a caller mint tokens to themselves before a transfer or burn tokens from the recipient. Check every cast inside incentive/penalty calculators.
- [ ] Codebase deliberately skips SafeMath/overflow protection on the theory that "values are derived from ETH and can't realistically overflow" — verify that assumption explicitly; prefer defensive math regardless.
- [ ] Token-supply overflow (via a malicious/inflated token) cascades into **system-wide halt** because SafeMath reverts on core functions like `processProposal`/`cancelProposal` — consider explicitly allowing overflow for untrusted tokens rather than let it brick unrelated functionality.
- [ ] **Incorrect comparison operator** in a payment/settlement check (e.g. `>=` used where `<=` is required, or vice versa) — can let a swap/settlement succeed while paying **less** than required, draining the pool at no cost. Read every comparison operator in a payment-verification `require` character-by-character; don't assume it's correct because "a check exists."
- [ ] Rounding/accounting bug lets a user **transfer or withdraw more than their tracked balance** (e.g. a rebasing-token credits/balance conversion that rounds in the user's favor) — verify balance is checked *before* the arithmetic that derives the transferable amount, and that rounding always favors the protocol, not the user.
- [ ] A global invariant (e.g. "total supply ≥ sum of user balances") can be **broken by an opt-out/opt-in mechanism** (like exempting an address from rebasing) — enumerate every documented invariant and check whether any user-facing toggle can violate it.
- [ ] Use of **undefined evaluation order** — e.g. a variable assigned inside a comparison and used on both sides of that same comparison in one expression. Evaluation order of subexpressions is unspecified in Solidity; rewrite to avoid assign-and-compare-same-variable idioms.
- [ ] Division functions (`rdivide`, `wdivide`, etc.) that accept a user/caller-supplied divisor without checking for **zero** — will simply revert, but confirm whether that revert is exploitable as a DoS vector on a critical path.
- [ ] "Balance" or "rate" helper functions that are supposed to convert between two unit systems (raw vs. numeraire, base vs. quote) but **return the wrong unit in some code paths** — cross-check every helper's return unit against how callers use it; mixed units silently produce wrong amounts rather than reverting.

## 6. Function Correctness / Dead Logic
- [ ] Function declares a return type but has **no return statement on all paths** (defaults to 0/false) — check every `bool`/`uint` returning function for complete path coverage.
- [ ] Function meant to validate/assert something always returns `false` or reverts, never `true` — effectively useless as a boolean check.
- [ ] Redundant code / dead branches because an invariant elsewhere makes them unreachable (verify assumptions, don't just trust comments).
- [ ] Contracts implement an interface's logic but don't formally inherit from it — later interface changes silently diverge from implementation.
- [ ] Mismatches between interface definitions and actual contract implementations.
- [ ] `newCurve()`/factory-style functions silently return an **existing** instance instead of reverting when parameters differ from what was requested.
- [ ] A state-mutating value (e.g. committed amount, staked balance) is **overwritten** on repeat calls instead of accumulated/added — check every "commit"/"stake"/"deposit" tracker for `=` vs `+=`. Bonus: does calling with amount `0` erase a prior commitment entirely (unauthorized griefing)?
- [ ] Functions meant to be gated to a specific lifecycle phase (pre-launch, post-launch, active, paused) remain callable in the "wrong" phase because the guard was never added — enumerate every lifecycle-sensitive function and confirm phase-gating exists on **each**, not just the obvious entrypoint.
- [ ] Two functions are documented/intended to be **mutually exclusive** (e.g. normal exit vs. emergency exit) but nothing enforces that calling one disables the other — check for missing cross-guards between "either/or" code paths.
- [ ] A boolean/status helper (`isTimeEnded`, `isInitialized`, `isPaused`) returns a **misleading value before proper initialization** (e.g. computing "ended" from a zeroed `startTime`, so an un-started timer looks expired) — trace what every status-check function returns on default/uninitialized storage.
- [ ] Cancel/pause functionality that updates a tracking mapping (`cancelled[orderId] = true`) but **nothing downstream ever reads it** — the cancel has no actual effect; verify every "cancel" or "revoke" actually gates the corresponding execution path.
- [ ] A capped/limited resource (loan supply, mint cap) is checked **before** the action but not **after** — an action that individually looks fine can push the total over the cap; check both pre- and post-condition around every cap enforcement.
- [ ] Specification says one thing (e.g. "2 week timelock") but the code makes that value **configurable with no range check** — always diff the written spec against the actual enforced bounds, not just presence of a variable with the right name.

## 7. Storage / Variable Shadowing
- [ ] Child contract redefines a storage variable already declared in parent (e.g. `_allowances`, `_totalSupply`) — shadowing causes inconsistent reads depending on which contract's function is called.
- [ ] Stale comments describing storage packing that no longer applies (copy-pasted from another codebase, e.g. Uniswap-style slot packing).
- [ ] Proxy pattern (`DELEGATECALL`) where the **implementation contract's constructor sets state that the proxy can never see** — proxy only initializes its own admin/implementation slots, so `owner`/`name`/`symbol`/etc. from the implementation's constructor read as default/zero through the proxy. Use an explicit one-time `initialize()` function instead of a constructor for anything meant to be proxy-visible.
- [ ] Multi-level inheritance in an **upgradeable** contract where a new storage variable added to a parent contract shifts the storage layout of all child contracts — check for storage-gap (`uint256[50] __gap;`) reservations in every upgradeable parent.
- [ ] Non-"upgrade-safe" base contracts (plain OpenZeppelin instead of `@openzeppelin/contracts-upgradeable`) mixed into an upgradeable inheritance chain — breaks the `Initializable` linearization the compiler expects.
- [ ] Initializer function doesn't set **all** state variables the contract depends on — some fields silently stay at default/zero after "initialization"; diff the constructor-equivalent list of fields against what `initialize()` actually assigns.
- [ ] Initialization-check helper (e.g. `_requireIsInitialized`) exists but is **applied inconsistently** across functions — some getters/setters skip it with no documented reason, allowing calls against an uninitialized contract.

## 8. DoS / Gas / Unbounded Loops
- [ ] Unbounded loop over user-growable data structure (linked list, array) used in critical read/write path — becomes a DoS vector past a certain size (calculate the practical element-count ceiling given block gas limit).
- [ ] External calls inside a potentially unbounded loop (e.g. flash-loan aggregation across multiple sources).
- [ ] No fee/cost on spam-able actions (e.g. account creation) allowing resource exhaustion of a fixed-capacity structure (e.g. Merkle tree depth).
- [ ] Hardcoded gas limits for calls/forwarding — may break if gas costs change (post-EIP repricing).
- [ ] A user-controlled whitelist/array (whitelisted tokens, loan accounts) is iterated over in a "must succeed" function (ragequit, batch process) with **no cap on its size** — growth over time can push the loop past the block gas limit and **permanently lock funds** for everyone, not just the user who caused the growth.
- [ ] An account/loan tracker array pushes a new entry **every time**, even for repeat/duplicate actions by the same account — bloats the array unnecessarily and worsens the unbounded-loop DoS above; only push on genuinely new entries.
- [ ] A single reverting recipient (e.g. a smart-contract address with a broken/absent payable fallback) inside a **batch payout loop** can block the entire batch, freezing payouts for every other unrelated recipient — prefer a pull-over-push withdrawal pattern for batched payouts.
- [ ] `.transfer()` used to send ETH to a possibly-contract recipient — the 2300 gas stipend can be insufficient for the recipient's fallback (or when called through a proxy that adds overhead), permanently trapping funds. Use `Address.sendValue` (with reentrancy guard) instead of `.transfer()`/`.send()`.
- [ ] Excess ETH sent beyond what a batched/looped action actually consumes is **never refunded** — no `withdrawEth`-style escape hatch, so overpayment gets permanently trapped in the contract.
- [ ] "NoThrow"/try-catch batch variants exist for some entrypoints (e.g. order filling) but **not for others** (e.g. transaction execution, order matching) that share the same underlying failure mode — one bad item in the batch reverts the entire batch for those entrypoints; check for consistency across all batch-processing surfaces.

## 9. Event / Logging Integrity
- [ ] Critical state-changing functions emit **no event at all** — breaks monitoring/indexing.
- [ ] Events emitted **without checking preconditions** — e.g. `cancelTransaction` emits even if nothing was queued, or `withdraw(0)` emits `Withdrawn` — enables **event log poisoning / spam** and can confuse off-chain monitors.
- [ ] Idempotent state changes (add/remove delegate) emit events even when no actual state change occurred — event log bloat.
- [ ] Event parameters not indexed — hinders off-chain filtering (minor, but check systematically).
- [ ] Sensitive state changes (funding rate updates, reward transfers, address/role changes) happen with **zero event emission** — specifically check any function whose name suggests a pure "getter" but which actually mutates critical state as a side effect (see section 14's "unexpected side effects" item — these two issues travel together).

## 10. Governance / Timing
- [ ] Governance proposals cancellable by the proposer even after being accepted/queued — check state-machine restrictions on `cancel()`.
- [ ] Admin can change system parameters/upgrade contracts with **no timelock**, enabling front-running of pending user transactions or malicious surprise changes.
- [ ] `setFrozen()`/pause-style admin functions with no delay — can be used to front-run and block specific deposits/withdrawals/swaps.
- [ ] Sensitive parameter setters (`setParams`, fee schedules, margin rates) take effect **instantly** with no time-lock — users have no chance to react (close positions, withdraw) before a change that could liquidate or disadvantage them lands.
- [ ] Setters like `setParams`/`setGovernanceParameter` accept **any value with no sanity/threshold/range checks** — a valid-looking call can silently misconfigure the system (e.g. margin rate that breaks liquidation logic).
- [ ] A pause/shutdown function exists with **no corresponding unpause**, and no way to redeploy/relink the paused instance — a "temporary" admin action becomes permanent for that pool/pair.
- [ ] Protocol fee or economic parameter is a function of `tx.gasprice` or otherwise **user-controllable** — lets users set gas price to zero (or a favorable value) to dodge fees, or gives certain actors (e.g. block producers/market makers) a structural discount on front-running costs.
- [ ] Bidding/voting system with no incentive to act early — anyone with sufficient capital can wait until the last possible moment (when opponents can no longer react) to decide the outcome; consider decay-weighted bids or commit-reveal with time-weighting.

## 11. Initialization & Deployment
- [ ] Contract usable (deposits/withdrawals allowed) before fully initialized / in a semi-configured state.
- [ ] Constructor parameters (e.g. maturity timestamp, addresses, interest rate) lack range/sanity validation — can be deployed with nonsensical values (e.g. maturity in the past).
- [ ] Hardcoded contract addresses instead of constructor/setter-injected — increases redeploy/testing risk and cross-chain reuse errors.
- [ ] Low-level calls (`call`, `delegatecall`, raw `safeTransfer`-style helpers) invoked **without checking that the target contract actually exists** — EVM returns "success" for calls to non-existent or self-destructed addresses. An attacker can pre-register/pre-deposit against a not-yet-deployed address (deterministic via CREATE/CREATE2) and later drain it once deployed, or exploit a token that has since self-destructed.

## 12. Compiler / Tooling Hygiene
- [ ] Floating pragma (`^0.6.0`) on top-level deployed contracts instead of pinned version.
- [ ] Outdated compiler version in use relative to current stable release.
- [ ] Solidity compiler optimizations enabled without weighing historical optimizer bugs.
- [ ] `ABIEncoderV2` (or other experimental features) used in production.
- [ ] Duplicate contract names across the repo — breaks tooling (Slither, Buidler/Hardhat artifact generation).
- [ ] Unnecessarily small integer types (`uint8`, `uint128`, etc.) used without genuine storage-packing benefit — extra gas from EVM zero-padding.
- [ ] `uint` used instead of explicit `uint256` throughout — explicitness/readability.

## 13. Signatures, Replay & Off-Chain Approvals
- [ ] Permit/meta-transaction signature schemes (`ERC20Permit`/EIP-2612 style) that don't bind `chainId` **into the signed message itself** (only into a fixed domain separator) — a post-deployment chain fork can make the same signature valid on both forks, enabling cross-chain replay.
- [ ] `ecrecover`/signature-recovery libraries return `address(0)` on an **invalid** signature rather than reverting — if the caller doesn't explicitly check for the zero address, an invalid signature can silently be treated as "signed by address 0," potentially matching an uninitialized or default state elsewhere in the code.
- [ ] Delegated signature-validator patterns (`setSignatureValidatorApproval`) — if a validator contract is ever compromised, check for a race condition that lets an attacker validate arbitrary malicious transactions during the compromise window.
- [ ] Oracle-message/signature-based "instant withdraw" flows — check whether an attacker can **replay someone else's valid oracle message/signature** with a deliberately-limited gas budget to burn the associated one-time-use nonce (`userInteractionNumber`) while making their own call fail, permanently blocking the legitimate user's real withdrawal.

## 14. Front-Running / MEV Patterns (cross-cutting — check every state-changing entrypoint against this list)
- [ ] Can this function's parameters be **copied from the mempool and resubmitted with higher gas** by someone else to their own benefit or to grief the original sender? (classic front-running)
- [ ] Can an admin/privileged actor **front-run their own users** by pushing a parameter/logic change moments before a queued user transaction executes against the old assumption?
- [ ] Two-transaction submission flows (submit proposal → sponsor/confirm separately) — can the second step be front-run by an unrelated third party to seize control of, block, or grief the proposal?
- [ ] Any function that assigns a "claim" to whichever address calls first (e.g. `delegateKey` assignment) — can a third party front-run a user's own assignment transaction and claim it first?
- [ ] Pool/market **initialization** with no access control — first caller sets the initial price/state; can be front-run to set an unfair price and drain the real first depositor.

---

# Token Standard Audit Checklist (ERC-3643 / ERC-4626 / ERC-7518)
Each check line names the control to verify; the sub-line is what breaks if it's missing.

---

# ERC-3643 (T-REX)

**Transfer gate**
- [ ] `transfer` and `transferFrom` both run pause/frozen/identity/compliance checks
  → one path skips a check the other has, bypassing compliance entirely
- [ ] Balance check uses free balance (balance − frozen), not total
  → holder moves tokens that should be locked
- [ ] `transferred` hook fires after every successful transfer
  → compliance module counters (holder count, volume) drift out of sync

**Mint / Burn**
- [ ] Mint requires verified receiver identity
  → tokens land on an ineligible/unverified address
- [ ] Burn correctly reduces frozen count, never below actual balance
  → frozen count exceeds real balance, breaking future transfer checks
- [ ] Mint/burn restricted to Agent/Owner; batch versions validate array lengths
  → unauthorized supply changes, or mismatched batch arrays corrupt state

**Forced transfer**
- [ ] Destination still must be verified even though compliance is overridden
  → admin override becomes a way to move funds to an illegitimate address
- [ ] Source's frozen accounting updated when frozen tokens are force-moved
  → stale frozen count blocks or wrongly permits future transfers

**Freeze / Pause**
- [ ] Full freeze blocks both send and receive; partial freeze can't exceed balance
  → frozen holder still moves funds, or freeze count goes negative/overflows
- [ ] Pause halts transfer/mint/forced-transfer; only Agent/Owner can toggle
  → anyone can halt the token, or paused state doesn't actually stop transfers

**Recovery**
- [ ] Recovery moves balance + frozen count + identity link; old wallet fully retired
  → partial migration leaves frozen/locked assets stuck, or old wallet still usable
- [ ] New wallet is verified under the same identity; caller proves ownership of lost wallet
  → recovery becomes an unauthorized fund-redirection path

**Identity Registry**
- [ ] `isVerified` checks valid, unexpired, unrevoked claims from a trusted issuer per required topic
  → expired/revoked/wrong-issuer claims still pass verification
- [ ] Registration restricted to Agent; registry/storage/topics swaps guarded
  → unauthorized identity edits, or a swap silently changes the entire compliance basis

**Trusted Issuers / Claim Topics**
- [ ] Issuers scoped to specific topics, not trusted for everything
  → an issuer trusted for KYC can also forge accreditation claims
- [ ] Issuer/topic lists bounded
  → verification loop runs out of gas, freezing all transfers
- [ ] Removing an issuer/topic immediately affects live verification
  → a revoked issuer's claims still count, or removal silently locks out holders unexpectedly

**Modular Compliance**
- [ ] `canTransfer` is read-only; state changes only in hooks callable by the token itself
  → an unrestricted hook lets anyone desync module counters at will
- [ ] Modules can't be gamed with dust/self-transfers/rounding; no unbounded loops
  → caps and limits bypassed, or module iteration bricks all transfers
- [ ] A single reverting module can't brick the whole token
  → one bad module permanently freezes every transfer system-wide

**Roles & Upgrades**
- [ ] Owner/Agent separated; ownership transfer is two-step or multisig
  → single-tx ownership transfer to a wrong/malicious address is irreversible
- [ ] Initializers run once; storage layout preserved across upgrades
  → re-initialization hijack, or an upgrade corrupts existing storage
- [ ] Upgrade authority behind multisig/timelock
  → single compromised key can rewrite all token logic instantly

**Deployment**
- [ ] Deploy order correct (identity/compliance before token); all registries wired correctly; no leftover test data
  → token launches pointing at wrong/uninitialized registry, or test addresses retain live privileges

---

# ERC-4626 (Vaults)

**Share price / totalAssets**
- [ ] `totalAssets` counts only real backing assets, values strategy positions live (not stale)
  → share price is inflated/deflated by phantom or stale value
- [ ] Direct token donations can't move share price unfairly
  → attacker donates tokens to manipulate the price other users convert at
- [ ] `convertToShares`/`convertToAssets` are exact inverses within rounding
  → arbitrageable gap lets users round-trip for free value

**Inflation / donation attack**
- [ ] Virtual shares/dead shares/decimal offset (or seeded deposit) actually implemented and tested
  → first depositor donates to spike price, next depositor's shares round down to ~0
- [ ] Zero-supply cold start handled safely
  → division by zero or exploitable conversion on an empty vault

**Rounding**
- [ ] Deposit rounds shares down, mint rounds assets up, withdraw rounds shares up, redeem rounds assets down
  → rounding the wrong way lets users extract value one wei at a time
- [ ] No tiny amount produces a "free" operation (zero shares burned, assets still move)
  → dust transactions bypass economic logic entirely

**Interface compliance**
- [ ] `preview*` exactly matches real execution outcome, incl. fees
  → integrators relying on preview get misled and lose funds
- [ ] `max*` reflects true limits, returns 0 when paused/disabled
  → users attempt operations that will revert, or a paused vault appears open
- [ ] `withdraw` delivers exactly the requested amount or reverts
  → silent partial fulfillment breaks integrator assumptions

**Reentrancy / CEI**
- [ ] Shares burned / accounting updated before assets leave the vault
  → reentrant call drains vault before state reflects the withdrawal
- [ ] Read-only reentrancy considered (no callback reads a mid-update price)
  → external protocol reads a manipulated intermediate share price
- [ ] Deposit credits actual received balance, not requested amount
  → fee-on-transfer token causes vault to over-credit shares

**Non-standard tokens**
- [ ] Fee-on-transfer / rebasing tokens handled or explicitly excluded
  → vault accounting silently diverges from actual token balance
- [ ] Transfer-hook tokens can't re-enter via callback
  → ERC777-style hook becomes a reentrancy vector
- [ ] Decimals not hardcoded to 18
  → high/low-decimal tokens cause rounding blowups or truncation to zero

**Oracle risk**
- [ ] No manipulable spot price/reserve read drives valuation
  → flash loan distorts share price within a single transaction
- [ ] Feeds checked for staleness and sane bounds; signed updates carry nonce/timestamp
  → stale or replayed price feed misprices the vault

**Yield / rewards**
- [ ] Reward state updates before balance changes (mint/burn/transfer)
  → user is paid on a balance they no longer hold, or misses rewards they earned
- [ ] Small deposits can't grief reward accrual for others
  → attacker spams tiny deposits to deny/delay rewards to real stakers
- [ ] Negative yield/loss handled without corrupting accounting
  → a bad harvest silently breaks share-to-asset conversion

**Strategy integration**
- [ ] Shares vs. assets never confused in strategy return values
  → misinterpreted return value causes massive accounting error
- [ ] Deployed balances tracked by actual amount moved
  → tracked vs. real balance drifts, eventually causing insolvency
- [ ] Paused/failing strategy doesn't brick the whole vault; real slippage protection on strategy moves
  → one broken integration locks all user funds, or a swap gets sandwiched

**Access control**
- [ ] Strategy/fee/cap/pause functions restricted; caps enforced against actual depositor not just caller
  → unauthorized fund movement, or caps bypassed via a router/proxy
- [ ] Harvest can't be sandwiched; rescue function can't pull user-owed assets
  → MEV extraction on harvest, or "rescue" becomes a rug mechanism
- [ ] Powerful actions (fee/strategy/upgrade changes) behind multisig/timelock
  → single key can instantly redirect vault funds or logic

**Upgrades / deployment**
- [ ] Initializer runs once; storage layout preserved; underlying asset pinned at deployment
  → re-init hijack, storage collision on upgrade, or asset swapped out from under holders

---

# ERC-7518 (DyCIST)

**Transfer / compliance gate**
- [ ] `safeTransferFrom` and batch transfer both run `canTransfer`; no inherited ERC-1155 bypass
  → raw ERC-1155 path moves tokens without any compliance check
- [ ] Transferable balance = `balanceOf − lockedBalanceOf`, enforced not just displayed
  → locked tokens move anyway if a caller passes a larger amount
- [ ] Receiver eligibility checked, not just sender
  → tokens land on a frozen/ineligible receiver
- [ ] Batch transfers apply the same checks per item with matching array lengths
  → one disallowed leg smuggled through inside a batch

**Partitions & locks**
- [ ] Can't lock more than balance; unlock can't go negative
  → lock accounting overflows/underflows, corrupting transferability
- [ ] Lock state isolated per account and per partition
  → locking one holder's partition wrongly affects another's
- [ ] Total locked never exceeds total supply across mint/burn/transfer/force-transfer/merge
  → invariant break lets more tokens be "locked" than exist, freezing legitimate transfers
- [ ] Partition merges preserve total balance and carry over remaining locks
  → merge silently mints or destroys value

**Vouchers / dynamic compliance**
- [ ] Signature recovers to the expected trusted attestor only
  → forged voucher from an untrusted signer passes eligibility
- [ ] Voucher has nonce + expiry, enforced at validation
  → same voucher replayed to bypass compliance repeatedly
- [ ] Voucher bound to specific holder/partition/amount/intent
  → a voucher issued for one transfer is reused to authorize a different one
- [ ] EIP-712 domain binds to this chain and this contract; encoding exact
  → voucher replayed across chains/contracts, or malformed payload still validates

**Freezing**
- [ ] Frozen address blocked from send, receive, and payout collection
  → a side-door path (e.g. bulk payout) ignores the freeze
- [ ] Freeze/unfreeze access-controlled and logged with reason
  → unauthorized freeze, or no audit trail for compliance action

**Forced transfer / admin powers**
- [ ] Force transfer gated to correct role, lands on eligible destination, keeps lock accounting consistent
  → unauthorized or misdirected forced transfer, or leftover inconsistent lock state
- [ ] Roles for mint/lock/freeze/force-transfer are distinct and least-privileged
  → one compromised key controls every admin function at once

**Payouts**
- [ ] Frozen/ineligible recipients excluded from payouts; balance sufficiency checked before batch starts
  → payout to a non-compliant holder, or batch fails halfway leaving inconsistent state
- [ ] Pro-rata math rounds sensibly; one bad recipient can't grief the whole batch
  → dust exploited for outsized share, or single failure blocks all distributions

**Cross-chain**
- [ ] Bridge messages/vouchers carry a nonce, consumed once
  → replayed message mints or unlocks tokens twice
- [ ] Destination mint backed by a verified source-chain burn
  → unchecked message allows minting without a corresponding burn


---

# Uniswap V4 Hooks Security Checklist
Each check names the control to verify; the sub-line is what breaks if it's missing.

**Hook Permission / Address Config**
- [ ] Declared `getHookPermissions()` matches the permission bits encoded in the deployed hook address
  → mismatch means PoolManager silently never calls a function the hook claims to support
- [ ] Address doesn't encode extra permission bits beyond what's implemented
  → PoolManager tries to call a non-existent function, causing DoS
- [ ] Return types match exactly what PoolManager expects (e.g. tuple types)
  → wrong return type/size causes the transaction to revert
- [ ] Future upgrades that add new callbacks are matched by a redeploy at an address with the right permission bits
  → newly added function (e.g. `afterSwap`) is silently never invoked since address permissions are immutable

**Custom Accounting (BeforeSwapDelta / BalanceDelta)**
- [ ] Fee deltas are returned as negative, rebates as positive, from the hook's perspective
  → sign error causes the hook to pay out instead of collect, or vice versa
- [ ] Upper/lower 128-bit fields map to the correct token/amount per the type's spec (BeforeSwapDelta vs BalanceDelta have different orderings)
  → swapped fields misattribute deltas between specified/unspecified or token0/token1
- [ ] Delta parameter order matches `zeroForOne` direction
  → wrong-direction delta causes incorrect swap execution or settlement failure
- [ ] All deltas returned are fully settled/accounted for
  → unsettled balance causes transaction failure or stuck funds

**Async Hooks (full custody of swap)**
- [ ] Hook enforces where swapped tokens actually go; no arbitrary-address transfer path
  → malicious hook directly steals user's swapped tokens
- [ ] Hook always returns/settles the swapped assets back
  → faulty hook fails to return funds, permanently locking them
- [ ] No hook-controlled delay or front-run window around execution
  → hook exploits timing to extract value from price movement at user's expense

**Front-Running / MEV via Hook Logic**
- [ ] Price- or fee-adjustment logic doesn't rely on predictable, front-runnable on-chain state
  → MEV bots anticipate fee/price changes and extract value before the user's tx lands
- [ ] No reliance on `block.timestamp` or other manipulable timing signals for execution logic
  → time-sensitive logic gets gamed by miners/sequencers
- [ ] On-chain price reads use manipulation-resistant sources (TWAP, not spot)
  → hook's pricing logic is manipulated via flash loan or spot-price attack

**Gas / DoS**
- [ ] No unbounded loops over growable arrays (e.g. authorized-user lists) inside swap-path callbacks
  → array grows large enough to exceed block gas limit, blocking all swaps
- [ ] Use mappings instead of arrays for membership checks where possible
  → avoids O(n) gas cost scaling with list size
- [ ] `require`/`revert` conditions are correct and can't wrongly fire on valid input
  → legitimate swaps revert due to a faulty guard condition

**Tick Traversal / Order Execution**
- [ ] Tick search direction logic matches Uniswap's own directional convention (`tickNext - 1` when searching down, `tickNext` when searching up)
  → incorrect direction (e.g. `+1` regardless of direction) skips over valid initialized ticks, especially at tick spacing of 1
- [ ] Custom tick-traversal code is tested against the exact reference implementation, not just "close enough" logic
  → subtle off-by-one silently breaks order/position discovery without reverting

**Share Rounding**
- [ ] Share-to-asset conversion is protected against first-depositor/donation manipulation
  → attacker deposits minimally then donates tokens directly to skew the exchange rate and drain later depositors
- [ ] Rounding always favors the protocol/pool, never the caller
  → repeated small-amount rounding errors are exploited to siphon value over many transactions

**Range Order / Timing Arbitrage**
- [ ] Range order creation isn't purely dependent on the pool's current tick at time of inclusion
  → attacker sandwiches a zero-input swap with an extreme `sqrtPriceLimit` right before the user's tx to move the pool to a manipulated price, then profits once the user's order is created there
- [ ] Users can set acceptable price bounds (min/max tick) for order creation, similar to slippage tolerance
  → without bounds, users have no protection against price manipulation between submission and inclusion

**Liquidity Management / Fee Distribution (hook as position owner)**
- [ ] Hook correctly separates protocol fees from caller/principal deltas
  → fee misallocation between the hook, LPs, and swappers
- [ ] Slippage protection applies only to principal deltas, not fee accruals
  → fee accrual gets blocked or manipulated by slippage checks meant for principal only
- [ ] Position salts are guaranteed unique
  → salt collision lets an attacker overwrite or hijack another position
- [ ] Concurrent/just-in-time liquidity modifications don't corrupt fee tracking
  → JIT liquidity added right before fee accrual skews fee distribution unfairly

**Swap Symmetry / Custom Calculation**
- [ ] Exact-input and exact-output swaps are handled symmetrically across `beforeSwap`/`afterSwap`
  → asymmetric handling creates an arbitrage path between the two swap directions
- [ ] Custom swap math doesn't depend on manipulable balances
  → attacker moves balances mid-transaction to bias the calculation
- [ ] Custom calculations are checked for rounding errors under adversarial inputs
  → accumulated rounding error becomes an exploitable value leak

**Pool Management / State Isolation**
- [ ] Multi-pool hooks maintain fully isolated storage per pool
  → cross-pool state contamination causes accounting errors or fund misallocation between unrelated pools
- [ ] Single-pool hooks restrict registration in `afterInitialize` (or equivalent) to the intended pool only
  → unauthorized pools register against the hook and exploit or drain shared logic/resources

**Native Token (ETH) Handling**
- [ ] `msg.value` handling and settlement follow checks-effects-interactions
  → reentrancy during native-token settlement manipulates pool/hook state
- [ ] Excess ETH is correctly refunded and accounted for
  → improper excess handling causes fund loss or incorrect settlement

**Callback Skipping / Execution Flow**
- [ ] Hook doesn't assume validations/state updates happen identically for self-initiated vs. external-caller operations
  → permissioned callbacks are skipped for self-initiated PoolManager calls, creating a logic gap the hook doesn't account for

**Hook State Consistency**
- [ ] State written in `beforeSwap` is not assumed to still be valid/unchanged by the time `afterSwap` reads it
  → concurrent swaps or reentrant calls cause `afterSwap` to compare against stale or mismatched state, reverting or misbehaving
