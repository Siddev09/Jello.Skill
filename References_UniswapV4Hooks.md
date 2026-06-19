### Hook Config

- A misconfigured Hook may cause transaction to revert due to invalid function implementation or return values of unexpected type or size.

```solidity

contract VulnerableConfigurationHook is Ownable, UUPSUpgradeable {
    function getHookPermissions() public pure returns (HookPermissions memory) {
        return HookPermissions({
            beforeSwap: true, 
            afterSwap: false,
            beforeModifyLiquidity: false,
            afterModifyLiquidity: false
        });
    }

    function beforeSwap(...) external returns (uint256 lpFeeOverride) {
        return 5000;
    }

    function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}

    // Future plan: Implement afterSwap() in next upgrade...
}

```
- UniV4 it derives permission directly from the Hooks address using bitwise operations not from the contract itself
- Vulnerable example contract claims to support `beforeSwap` but if it is deployed at an address that does not encode `BEFORE_SWAP_FLAG` PoolManager will not recognize it , and `beforeSwap` will never be called.

#### ISSUE

- Mismatch between the hooks declared permissions and its encoded address permissions.
- If hooks claims to support a specific Function but its address does not encode its permission bits poolManager never calls it. 
- If Hooks address includes extra permission bits , poolManager may attempt to execute non-existing function resulting in Dos
- Check the return type in beforeSwap because uniswap processes only `tuple` Type.
- Hook may implement new functions in future upgrades, its deployment address does not encode permissions for these functions. If afterSwap() is added in an upgrade, PoolManager will not recognize it because the contract's address lacks the required permission bits.

---


### Custom Accounting 

##### Explainer 

- custom accounting mechanism that allows Hooks to return deltas, which impact swap execution and liquidity modification. It allows Hooks to take fees from swaps and liquidity modifications, providing fine-grained control over balance adjustments and even overriding default swap behavior.

```solidity
type BeforeSwapDelta is int256;
…
function getSpecifiedDelta(BeforeSwapDelta delta) internal pure returns (int128 deltaSpecified) {
        assembly ("memory-safe") {
            deltaSpecified := sar(128, delta)
        }
    }

    /// extracts int128 from the lower 128 bits of the BeforeSwapDelta
    /// returned by beforeSwap and afterSwap
    function getUnspecifiedDelta(BeforeSwapDelta delta) internal pure returns (int128 deltaUnspecified) {
        assembly ("memory-safe") {
            deltaUnspecified := signextend(15, delta)
        }
    }
```

- One of the Parameters returned by beforeSwap() Hook is a BeforeSwapDelta, which is an alias type for int256, where upper 128 bits are for delta in specified tokens (e.g., token0 or token1) and lower 128 bits are for delta in unspecified tokens (fee adjustments or additional charges).

---

```solidity

type BalanceDelta is int256;
function amount0(BalanceDelta balanceDelta) internal pure returns (int128 _amount0) {
        assembly ("memory-safe") {
            _amount0 := sar(128, balanceDelta)
        }
    }

    function amount1(BalanceDelta balanceDelta) internal pure returns (int128 _amount1) {
        assembly ("memory-safe") {
            _amount1 := signextend(15, balanceDelta)
        }
    }


```

- Other Hooks use BalanceDelta, which is also an alias for int256 type. It shares the same design as BeforeSwapDelta, but the order is different. Upper 128 bits are for amount0, representing the delta in token0, and lower 128 bits are for amount1, representing the delta in token1.

---

BeforeSwapDelta is from the perspective of the Hook : 

- If the Hook takes a fee, it must pass the value as a negative delta (-a).
- If the Hook grants a rebate, it must pass the value as a positive delta (+a).

Uniswap processes the delta : The swap amount (amountToSwap) is initially set to params.amountSpecified. The specified token delta (HookDeltaSpecified) is added to amountToSwap, modifying the swap execution.


```solidity

contract VulnerableDeltaAdjustmentHook is BaseHook {

    function beforeSwap(
        address sender,
        PoolKey calldata key,
        SwapParams calldata params,
        bytes calldata HookData
    ) external override returns (bytes4 selector, int256 HookDelta, uint24 lpFeeOverride) {
        int128 token0Delta = 500 * 10**18; // fee
        int128 token1Delta = 0;

      HookDelta = BeforeSwapDelta.toBeforeSwapDelta(token0Delta, token1Delta);

        selector = bytes4(keccak256("beforeSwap()"));
        return (selector, HookDelta, lpFeeOverride);
    }
}


```


### ISSUES 

- token0Delta is intended to represent the Hook fee. since BeforeSwapDelta is considered from the perspective of Hook the fee shoul dbe negative when the Hook deducts an amount

- Order of the parameters should be correct as determined by the swapParam.zeroForOne parameter.
- If deltas are not properly handled, it can lead to tx failures due to unsettled token balance.

---

## Async Hooks

Async Hooks introduce a unique security concern in Uniswap V4 overriding Uniswap’s native swap logic. Unlike standard Hooks that modify swap parameters, Async Hooks replace the core swapping mechanism by taking full control over the user’s sent tokens and executing their own logic.

Async Hooks replace Uniswap’s swap logic by reversing amountToSwap in the delta calculations, meaning that instead of Uniswap executing the swap, the Hook takes full custody of the swapped amount.



### ISSUES

The ability of Async Hooks to assume full custody over swapped funds creates significant risks. A malicious or misconfigured Hook can:

- Steal Funds – Since Uniswap does not enforce execution rules, an attacker can create a Hook that directly transfers swapped tokens to an unauthorized address.
- Block Execution – A faulty Hook could fail to return swapped assets, resulting in locked user funds.
- Price Manipulation – If a Hook front-runs or delays swap execution, it can take advantage of price fluctuations at the user’s expense.


---

### FrontRun 

custom execution logic at key points in swaps and liquidity management. If improperly designed, Hooks can become vulnerable to front-running and MEV (Maximal Extractable Value) attacks, where malicious actors manipulate transactions order for profit.


```solidity

contract VulnerablePriceOracleHook is BaseHook {
    PriceOracle public oracle;
    uint24 public dynamicSwapFee = 3000;
    mapping(address => uint256) public lastRecordedPrice;

    constructor(address _oracle) {
        oracle = PriceOracle(_oracle);
    }

    function beforeSwap(
        address sender,
        PoolKey calldata key,
        SwapParams calldata params,
        bytes calldata HookData
    ) external override returns (bytes4 selector, int256 HookDelta, uint24 lpFeeOverride) {
       
        uint256 currentPrice = oracle.getPrice(key.token0);
        uint256 previousPrice = lastRecordedPrice[key.token0];

        if (previousPrice == 0) {
            lastRecordedPrice[key.token0] = currentPrice;
            return (bytes4(keccak256("beforeSwap()")), 0, dynamicSwapFee);
        }

        // Adjust fees based on actual price movement over time
        if (currentPrice > 1.1 * previousPrice) { 
            dynamicSwapFee = 500; // Lower fee
        }
        else if (currentPrice < 0.9 * previousPrice) {
            dynamicSwapFee = 10000; // High penalty fee
        }

        lastRecordedPrice[key.token0] = currentPrice; // Update the price

        HookDelta = BeforeSwapDelta.toBeforeSwapDelta(-int128(dynamicSwapFee), 0);
        selector = bytes4(keccak256("beforeSwap()"));
        lpFeeOverride = dynamicSwapFee;

        return (selector, HookDelta, lpFeeOverride);
    }
}

```


### ISSUES 

- Price-dependent logic – If a Hook adjusts swap fees or liquidity based on external market data, MEV bots can anticipate changes and exploit them.
- Time-sensitive execution – Hooks that enable delayed swaps or rely on block timestamps can be manipulated.
- Oracle-based pricing – Hooks that fetch on-chain prices may be vulnerable to oracle manipulation attacks.



---


### DOS


If improperly implemented, Hooks can increase gas costs, introduce infinite loops, or cause unnecessary reverts, ultimately leading to denial-of-service (DoS) attacks


```solidity
contract VulnerableDoSHook is BaseHook {
    address[] public authorizedUsers;
    ExternalContract public externalContract;

    constructor(address _externalContract) {
        externalContract = ExternalContract(_externalContract);
    }

    function beforeSwap(
        address sender,
        PoolKey calldata key,
        SwapParams calldata params,
        bytes calldata HookData
    ) external override returns (bytes4 selector, int256 HookDelta, uint24 lpFeeOverride) {
       
        // Loop through all authorized users
        for (uint256 i = 0; i < authorizedUsers.length; i++) { 
            require(externalContract.checkAccess(authorizedUsers[i]), "Access check failed");
        }

        selector = bytes4(keccak256("beforeSwap()"));

        return (selector, HookDelta, lpFeeOverride);
    }
}


```

### ISSUES

- If authorizedUsers grows too large, the beforeSwap() function will consume excessive gas, potentially exceeding block gas limits. This prevents swaps from executing, leading to a denial-of-service condition.
- Hook should limit the size of arrays and use more efficient data structures like mappings to ensure that swaps remain executable under all conditions.
- Incorrectly implemented require() or revert() statements can block transactions even when they should succeed.


---

### Tick spacing order execution

- it stems from the inconsistent implemetaion of tick traversal logic between standard Uniswap mechanisms and custom order management system.
- In Uniswaps `nextInitializedTickWithinOneWord` function, the protocol correctly handles directional tick searching by adjusting the starting position based on the search direction .
- when searching downward (ZeroForOne = true) it starts from [ tickNext - 1 ], and when searching upward (zeroForOne = false) , it starts from tickNext.

```solidity
// Correct Uniswap implementation
tick = zeroForOne ? 
    nextInitializedTick - 1 : // When going down, start before
    nextInitializedTick;      // When going up, start at position

// Flawed protocol implementation  
tick = zeroForOne ?
    nextInitializedTick - 1 : 
    nextInitializedTick + 1;  // Incorrectly starts after position

```

- in affected protocol’s [_findOverlappingPositions] function , the implemetation incorrectly sets the next tick as `nextInitializedTick + 1` regarless of direction when `zeroForOne` is false.
- This creates a critical flaw , when tick `spacing is 1` the algo attempts to search from `nextInitializedTick + 1` but since the next initialized tick is actually at `nextInitializedTick` , the search effectively skips over all valid positions.

---

### Share Rounding Exploit

- share based uniswaps hooks protocol face critical rounding vulnerabilities where attackers exploit maths to drain funds  . By making minimal inital deposits and manipulating the share to asset exchange rate through direct token transfers.

---

### Range order Arbitrage

- Range order creation mechanism that allows malicious actors to exploit the timing difference between transaction submission and block inclusion.
- When users create range orders the system calcualtes the starting position based on the current pools rounded tick price.
- Uniswapv4 pool the potentially minimal liquidity below market price , attackers can manipulate the pool by executing a zero-input swap with an exteremely low `sqrtPriceLimit` just before the users transaction  is processed
- This artificially depresses the pool price , causing the users range order to be created at the manipulated lower price point.
- Once the users liquidity is deployed at this disadvantageous level , the attacker can immediately purchase the tokens at the artifically low price and profit from the difference.

MITIGATION 

- Imelementing user-defined minimum and maximum tick parameters similar to slippage tolerance controls. allowing users to specify acceptable price ranges for the their orders and preventing execution if the pool price has moved beyond their specified bounds during the transaction processing window.

---

### Liquidity Management and Fee Distribution

- Hooks that calls modifyLiquidity becomes position owners, inheriting complex responsibilities for fee management and user fund protection.
- Since anyone can trigger fee accurals at any time , hooks must account for just-in-time liquidity modification that could conflict with custom fee logic.

MITIGATIONS

- Implement robust tracking mechanisms that can handle concurrent fee accrual events without corruption or loss.
- Fee Delta Seperation - Properly distinguish between protocol fees and caller deltas to prevent fee misallocation.
- Slippage Protection - Apply slippage controls only to principal deltas, not fee accurals to prevent manipulation
- Salt Uniqueness - Ensure position salt generation creates truly unique identifiers to prevent position collision attacks

---

### Swap Logic Symmetry and Custom Calculation

Hooks must handle both exact - input and exact-output swaps symmetrically across beforeSwap and afterSwap callbacks. Asymmetric implemetation create arbitrage opportunities where attackers can exploit differences in swap direction handling to extract value from the protocol

Custom Swap Logic

- Hooks implemeting custom swap calcualtion face price manipulation vulnerabilities especially when calculation depend on manipulable balances or exhibit round errors.

---

### Pool Management and State Isolation

- MultiPool - when designing hooks that support multiple pools developers must implement strict state isolation to prevent cross-pool contamination without proper separation, malicious pools can overwrite or currupt state variables belonging to legitimate pools, leading to accounting errors , fund misallocation .
- Each pool must maintain its own dedicated storage space within the hook contract.
- Single Pool Restriction - For hook desined for exclusive single pool operation implement initlaization controls in the `afterInitialize` callback to prevent unauthoirized pool registraction .
- Failing to restrict pool access can allow maliciois actors to create competing pools that exploit the hooks logic or drain resources intended for the legitimate pool.

---

### Native Token Handling Vulnerabilities

- Native token support introduces reentrancy risks during `msg.value`  handling and settlement operation. Attackers can exploit reentrancy to manupulate pool or hook state, especially in system with custom accounting logic that depeds on token balances.
- Improper `msg.value`  management can lead to fund loss through failed excess returns or incorrect settlement calcualtion.

---

### Callback Skipping and execution flow

- When hooks initiate PoolManager calls, permissioned callbacks are skipped for self-initiated operations but trigger for external callers. This asymmetric behavior can create logic gaps where hooks assume certain validations or state updates occur during self-initiated operations,


---

### INCONSISTENT HOOK

- Improper hook execution order can lead to inconsistent state updates.

EX : 

```solidity



contract InconsistentHook {
    uint256 public lastSwapAmount;

    function beforeSwap(
        address sender,
        PoolKey calldata poolKey,
        SwapParams calldata params,
        bytes calldata data
    ) external override returns (bytes4, int128) {
        lastSwapAmount = uint256(params.amountSpecified); // Updates before swap
        return (this.beforeSwap.selector, 0);
    }

    function afterSwap(
        address sender,
        PoolKey calldata poolKey,
        int128 amount0Delta,
        int128 amount1Delta,
        bytes calldata data
    ) external override returns (bytes4) {
        require(uint256(amount0Delta) == lastSwapAmount, "State mismatch"); // Mismatch due to beforeSwap update
        return this.afterSwap.selector;
    }
}


```

