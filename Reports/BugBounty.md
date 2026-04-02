# Division by Zero in Funding Socialization Causes Debt Transfer to Future Traders

**Severity:** High\
**Category:** Logic Error / Economic Exploit / DoS

------------------------------------------------------------------------

## Summary

A division-by-zero condition in the funding socialization logic prevents
the finalization of bankrupt accounts when total open interest becomes
zero.

This results in a **two-phase vulnerability**:

-   **Phase 1 --- Liquidation DoS:** Bankrupt accounts cannot be
    finalized, leaving bad debt permanently unresolved.
-   **Phase 2 --- Debt Transfer:** Once new traders enter and open
    interest becomes non-zero again, the previously stuck bad debt is
    **fully socialized onto these new participants**, who had no
    exposure to the original loss.

This leads to **silent fund loss for innocent users** and violates
expected economic fairness.

------------------------------------------------------------------------

## Root Cause

The protocol attempts to distribute bad debt across active positions
using a per-share funding adjustment:

``` solidity
int128 fundingPerShare = -balance.vQuoteBalance.div(state.openInterest);
```

However:

-   `state.openInterest` can be **zero**
-   The division function **reverts on zero divisor**

There is **no guard condition** preventing this call when open interest
is zero.

------------------------------------------------------------------------

## Key State Mismatch

Two independent variables create the issue:

-   `openInterest` → tracks active position size
-   `vQuoteBalance` → tracks accumulated PnL and funding

When positions are closed:

-   `openInterest → 0`
-   `vQuoteBalance` **can remain negative**

------------------------------------------------------------------------

## Trigger Conditions

The issue occurs when all of the following are true:

1.  `openInterest == 0`
2.  A bankrupt account has `vQuoteBalance < 0`
3.  Insurance cannot fully cover the loss

------------------------------------------------------------------------

## Exploit Scenario

### Phase 1 --- Liquidation Failure

1.  Traders exit → `openInterest = 0`
2.  Bankrupt account still has negative balance
3.  Liquidation attempt reverts due to division by zero

### Phase 2 --- Debt Socialization

1.  New traders enter → `openInterest > 0`
2.  Liquidation succeeds
3.  Bad debt is distributed to new traders via funding

------------------------------------------------------------------------

## Impact

-   Direct loss to new traders\
-   Permanent liquidation DoS\
-   Hidden loss mechanism via funding\
-   Economic unfairness

------------------------------------------------------------------------

## Recommended Fix

``` solidity
if (balance.vQuoteBalance < 0) {
    if (state.openInterest > 0) {
        int128 fundingPerShare = -balance.vQuoteBalance.div(state.openInterest);
        state.cumulativeFundingLong += fundingPerShare;
        state.cumulativeFundingShort -= fundingPerShare;
    } else {
        balance.vQuoteBalance = 0;
    }
}
```

------------------------------------------------------------------------

## Key Insight

Bad debt should never be transferred to future participants. If no
traders exist, the protocol must absorb the loss.


# Multi-Execution Fee Miscalculation Due to Using Current Price for Historical Volume Reconstruction

**Severity:** Medium
**Component:** `OrderProcessor.sol`
**Function:** `processOrderExecutions()`, `calculateAndApplyFee()`

---

## Summary

When processing multi-execution orders, the order processor calculates cumulative historical trading volume (`cumulativeVolumeExecuted`) by multiplying the **current execution price** by the total previously filled token amount. Since each execution occurs against a different counterparty at a different price, this reconstruction is incorrect. The error causes the fee calculation to over- or under-charge the order initiator depending on price direction between executions.

---

## Root Cause

**OrderProcessor.sol** (execution handler)

```solidity
calculateAndApplyFee(
    orderId,
    orderDetails.buyer,
    marketInfo,
    -currentPrice.mul(previouslyFilledAmount[orderDetails.buyer.id]),  // ← BUG
    orderDetails.metadata,
    true
);
```

The protocol only stores cumulative **base amount** filled per order via `previouslyFilledAmount[id]`. It does not store cumulative **quote volume** (settlement token). To reconstruct the historical quote volume, it multiplies:

```
cumulativeVolumeExecuted = -(current execution price) × (total base amount filled so far)
```

But the actual historical quote volume is:

```
cumulativeVolumeExecuted = -(sum of each execution's price × that execution's base amount)
```

These are only equal if every execution occurred at the same price — which is not the case for multi-execution orders by design.

---

## How `cumulativeVolumeExecuted` Is Used in Fee Calculation

Inside `calculateAndApplyFee()`, `cumulativeVolumeExecuted` determines how much of the current trade falls into the fee-exempt zone (`minTradeSize`) vs. the percentage-fee zone:

```solidity
// First execution gets flat fee on minTradeSize
if (cumulativeVolumeExecuted == 0) {
    feeableQuote += marketInfo.minTradeSize;
    ...
}

// Calculate fee-eligible portion
int128 feeableAmount = MathHelper.abs(cumulativeVolumeExecuted + currentExecutionQuote) - marketInfo.minTradeSize;

// Cap to current execution size
feeableAmount = MathHelper.min(feeableAmount, currentExecutionQuote.abs());
```

The expression `abs(cumulativeVolumeExecuted + currentExecutionQuote) - minTradeSize` computes how far the **cumulative volume** exceeds the `minTradeSize` threshold. If `cumulativeVolumeExecuted` is wrong, the boundary between "fee-exempt" and "fee-charged" shifts incorrectly.

---

## Attack Scenario: Order Initiator Overpays Fees (Price Rises Between Executions)

### Setup
```
Order: BUY +100 tokens (multi-execution)
minTradeSize = 25,000 settlement units
feeRate = 0.02% (2 bps)
```

### Execution 1 — Fill 10 tokens @ price level 2,000

```
previouslyFilledAmount[id] = 0 (first execution)
cumulativeVolumeExecuted = -2,000 × 0 = 0

executionQuote = -20,000 settlement units
```

Fee calculation:
```
cumulativeVolumeExecuted == 0 → flat fee on minTradeSize
feeableQuote = -25,000

feeableAmount = abs(0 + (-20,000)) - 25,000 = -5,000 → negative → skipped
feeableQuote capped to executionQuote → feeableQuote = -20,000
Fee = 20,000 × 0.02% = $4.00
```

After execution 1: `previouslyFilledAmount[id] = +10`

### Execution 2 — Fill 2 tokens @ price level 3,000 (price rose)

**What the code calculates:**
```
cumulativeVolumeExecuted = -3,000 × 10 = -30,000  ← WRONG (overstated by 10,000)
executionQuote = -6,000

feeableAmount = abs(-30,000 + (-6,000)) - 25,000 = 36,000 - 25,000 = 11,000
cap: min(11,000, 6,000) = 6,000
feeableQuote = 0 + (-6,000) = -6,000

Fee = 6,000 × 0.02% = $1.20
```

**What should be calculated:**
```
cumulativeVolumeExecuted = -2,000 × 10 = -20,000  ← CORRECT
executionQuote = -6,000

feeableAmount = abs(-20,000 + (-6,000)) - 25,000 = 26,000 - 25,000 = 1,000
cap: min(1,000, 6,000) = 1,000
feeableQuote = 0 + (-1,000) = -1,000

Fee = 1,000 × 0.02% = $0.20
```

**Result: Order initiator pays $1.20 instead of $0.20 — a 6x overpayment on this execution.**

The overstated `cumulativeVolumeExecuted` makes the code think the initiator already passed the `minTradeSize` threshold by 5,000 more than reality, so more of execution 2's volume falls into the percentage-fee zone.

---

## Attack Scenario: Order Initiator Underpays Fees (Price Falls Between Executions)

### Setup
```
Order: BUY +100 tokens (multi-execution)
minTradeSize = 25,000 settlement units
feeRate = 0.02% (2 bps)
```

### Execution 1 — Fill 10 tokens @ price level 3,000

```
Actual volume spent: 30,000 settlement units
previouslyFilledAmount[id] = +10
```

### Execution 2 — Fill 5 tokens @ price level 1,000 (price crashed)

**What the code calculates:**
```
cumulativeVolumeExecuted = -1,000 × 10 = -10,000  ← WRONG (understated by 20,000)
executionQuote = -5,000

feeableAmount = abs(-10,000 + (-5,000)) - 25,000 = 15,000 - 25,000 = -10,000 → negative → skipped
feeableQuote = 0

Fee = $0 (percentage portion is zero — code thinks cumulative is still below minTradeSize)
```

**What should be calculated:**
```
cumulativeVolumeExecuted = -3,000 × 10 = -30,000  ← CORRECT
executionQuote = -5,000

feeableAmount = abs(-30,000 + (-5,000)) - 25,000 = 35,000 - 25,000 = 10,000
cap: min(10,000, 5,000) = 5,000
feeableQuote = 0 + (-5,000) = -5,000

Fee = 5,000 × 0.02% = $1.00
```

**Result: Order initiator pays $0 instead of $1.00 — complete fee evasion on this execution.**

The understated `cumulativeVolumeExecuted` makes the code think the initiator hasn't passed the `minTradeSize` threshold yet (cumulative = 15,000 < 25,000), when in reality they passed it in execution 1 (30,000 > 25,000).

---

## Impact

| Condition | Effect | Who Benefits |
|---|---|---|
| Price **rises** between executions | `cumulativeVolumeExecuted` overstated → more volume in fee zone → **initiator overpays** | Protocol gains |
| Price **falls** between executions | `cumulativeVolumeExecuted` understated → more volume in exempt zone → **initiator underpays** | Initiator gains |
| Large price swings + many executions | Error compounds across executions | Unpredictable |
| Price stable between executions | Error is negligible | Neither |

### Who is affected
- **All multi-execution order initiators** — every multi-execution order with >1 segment executed at different prices has incorrect fee calculation.
- **Protocol revenue** — fees are non-deterministically wrong in both directions (over-collection and under-collection).
- **Single-execution orders** — not affected, since `previouslyFilledAmount` is 0 on first execution and single orders complete in one call.

### Exploitation potential
While the order initiator does not control which counterparty they are matched with (order processor decides), a **malicious or buggy order processor** could deliberately sequence executions to manipulate fees:
- Execute early segments at high price → then execute later segments at low price → `cumulativeVolumeExecuted` is understated → initiator pays less fees → protocol loses revenue.

Even without a malicious processor, this is a **systemic accounting error** on every multi-execution order in volatile market conditions.

---

## Proof of Concept (Trace)

```
1. User submits multi-execution order for 100 tokens
2. Processor executes: segment 1 = 10 tokens, matched at price 2,000
   → previouslyFilledAmount[id] = +10
   → Fee calculated correctly (cumulativeVolumeExecuted = 0, first execution)

3. Price rises to 3,000

4. Processor executes: segment 2 = 10 tokens, matched at price 3,000
   → Code: cumulativeVolumeExecuted = -3,000 * 10 = -30,000  (should be -20,000)
   → Fee overcharged: code sees 30k past volume, reality is 20k
   → Excess fee extracted from user's balance

5. Price drops to 1,500

6. Processor executes: segment 3 = 10 tokens, matched at price 1,500
   → previouslyFilledAmount[id] = +20
   → Code: cumulativeVolumeExecuted = -1,500 * 20 = -30,000  (should be -50,000)
   → Fee undercharged: code sees 30k, reality is 50k past volume
   → User pays less than they should
```

---

## Recommended Fix

Track cumulative quote volume per order alongside base amount:

```solidity
// Add new storage mapping
mapping(bytes32 => int128) internal cumulativeQuoteVolume;

// In processOrderExecutions, after computing quoteDelta:
if (order.initiator != SYSTEM_ACCOUNT) {
    previouslyFilledAmount[order.id] += order.baseAmountDelta;
    cumulativeQuoteVolume[order.id] += order.quoteDelta;  // NEW
}

// In the calculateAndApplyFee call, use actual historical quote volume:
calculateAndApplyFee(
    orderId,
    orderDetails.buyer,
    marketInfo,
    cumulativeQuoteVolume[order.id],  // FIX: actual cumulative quote volume
    orderDetails.metadata,
    true
);
```

This ensures `cumulativeVolumeExecuted` reflects the true settlement token volume across all previous executions regardless of price variation.

---

## References

- OrderProcessor.sol — incorrect `cumulativeVolumeExecuted` calculation in execution handler
- OrderProcessor.sol — `calculateAndApplyFee` logic using `cumulativeVolumeExecuted`
- OrderProcessor.sol — `previouslyFilledAmount` update (base only, no quote tracking)

# Empty `validateAccountHealth()` Stub in OrderProcessor Bypasses Post-Trade Margin Enforcement

**Severity:** High
**Component:** `OrderProcessor.sol`
**Functions:** `OrderProcessor.validateAccountHealth()`, `OrderProcessor.executeOrders()`

---

## Summary

The `validateAccountHealth()` function in `OrderProcessor.sol` is a virtual stub that unconditionally returns `true`. It is never overridden. The `executeOrders` function calls `validateAccountHealth()` on both parties after executing a trade, but because the stub always passes, **no post-trade health validation occurs on-chain**. A user with minimal collateral can open an arbitrarily large derivative position — far beyond the protocol's risk limits — and if the position moves against them, the resulting bad debt is socialized to other traders.

The transaction processor may perform off-chain health simulation before submitting, but:
1. There is no on-chain function exposed to simulate post-trade health (no health preview function exists)
2. The on-chain enforcement was clearly **designed** to be the safety net — it exists, it's called, but its implementation is empty
3. A compromised, buggy, or misconfigured processor would submit overleveraged trades with zero on-chain resistance

---

## Root Cause

**OrderProcessor.sol** (validation stub)

```solidity
function validateAccountHealth(
    bytes32 /* account */
) internal view virtual returns (bool) {
    return true;
}
```

The function accepts an account parameter (which it ignores) and returns `true` unconditionally. It is marked `virtual`, suggesting it was intended to be overridden by a child contract — but no override exists anywhere in the codebase.

**OrderProcessor.sol** (enforcement point)

```solidity
require(validateAccountHealth(buyer.account), ERR_INVALID_BUYER);
require(validateAccountHealth(seller.account), ERR_INVALID_SELLER);
```

These lines execute after `_updateAccountBalances` has already modified both sides' positions. They are the **only** post-trade health gate in `executeOrders`. Because `validateAccountHealth()` always returns `true`, these `require` statements are no-ops.

---

## Comparison: Every Other State-Changing Path Enforces Health

| Function | Component | Health Check |
|---|---|---|
| `withdrawCollateral` | AccountManager | `computeHealth(account, INITIAL) >= 0` |
| `transferSettlement` | AccountManager | `isAboveInitialMargin(account)` |
| `openPosition` | AccountManager | `computeHealth(account, INITIAL) >= 0` |
| `closePosition` | AccountManager | `computeHealth(account, MAINTENANCE) >= 0` |
| **`executeOrders`** | **OrderProcessor** | **`validateAccountHealth() → true` (stub)** |

`executeOrders` is the only state-mutating function that modifies account balances without a real health check. Every other path routes through `AccountManager.computeHealth()` which queries multiple engines to compute weighted health.

---

## Vulnerable Code Path

```
User deposits 1000 settlement units
  └─► EntryPoint.depositCollateral()
        └─► settledTxQueue → (delay period) → processSettledTransaction()
              └─► AccountManager.depositCollateral()
                    └─► updateBalance(settlement asset, account, +1000)

Processor submits executeOrders (100 derivative contracts, ~6.8M notional)
  └─► EntryPoint.submitOrdersChecked()
        └─► processOrderExecution(OrderExecution)
              └─► validateAccount(buyer)  ← only checks account exists
              └─► validateAccount(seller)  ← only checks account exists
              └─► OrderProcessor.executeOrders()
                    ├─► _validateOrderSignatures()     ← signature, expiry, price crossing
                    ├─► _updateAccountBalances(buyer)   ← +100 contracts, -6.8M settlement
                    ├─► _updateAccountBalances(seller)  ← -100 contracts, +6.8M settlement
                    ├─► require(validateAccountHealth(buyer))   ← returns true (STUB)
                    └─► require(validateAccountHealth(seller))  ← returns true (STUB)

Post-trade state:
  Buyer has 1000 collateral backing ~6.8M notional
  AccountManager.computeHealth(buyer, INITIAL) = -171,538 units
  Effective leverage: 6,806x (max allowed: ~40x)
```

---

## Attack Scenario

### Setup

```
Derivative oracle price:   68,062 units
Initial margin ratio:      0.975
Max leverage:              1 / (1 - 0.975) ≈ 40x
Buyer deposit:             1000 settlement units
```

### Step 1 — User Deposits Minimal Collateral

The attacker deposits 1000 settlement units through the normal `EntryPoint.depositCollateral()` path. After the settlement delay period, the deposit is processed and the account has 1000 settlement units on the collateral ledger.

### Step 2 — Processor Matches Overleveraged Order

The attacker signs a buy order for 100 derivative contracts. The processor (or a compromised/buggy processor) includes it in order execution. The `executeOrders` function:

1. Validates signatures — pass (validly signed)
2. Validates price crossing — pass (buyer price >= seller price)
3. Updates balances — buyer gets +100 contracts, settlementBalance = -6.8M units
4. Calls `validateAccountHealth(buyer)` — **returns `true`** (stub)
5. Trade succeeds

### Step 3 — Post-Trade Insolvency

After the trade, the buyer's on-chain health (queried via `AccountManager.computeHealth`) is:

```
health = collateral + settlement_balance + (position × margin_ratio × price)
       = 1000 + (-6,808,603) + (100 × 0.975 × 68,062)
       = 1000 - 6,808,603 + 6,636,045
       = -171,558 units

INITIAL health:      -171,558 units  (insolvent)
MAINTENANCE health:  -86,460 units   (immediately liquidatable)
```

The account is **6,806x leveraged** against a maximum allowed leverage of ~40x.

### Step 4 — Bad Debt Socialization

If derivative price drops by just 1%:

```
Loss = 100 contracts × 68,062 units × 1% = 68,062 units
Collateral = 1000 units
Shortfall = 67,062 units
```

The 67,062 unit shortfall cannot be covered by the buyer's 1000 unit collateral. During liquidation finalization, the risk engine distributes the unrecoverable loss across all remaining position holders via funding rate adjustments — traders who had no involvement in the overleveraged position absorb the loss.

---

## Impact

| Dimension | Detail |
|---|---|
| **Unlimited leverage** | No on-chain bound on position size relative to collateral. The POC demonstrates 6,806x leverage where ~40x is the maximum. |
| **Bad debt creation** | Any adverse price move larger than the thin collateral creates unrecoverable bad debt. A 1% price move on the POC position creates ~67K units of bad debt from 1K collateral. |
| **Socialized losses** | Bad debt from liquidation is distributed to innocent position holders through funding rate adjustments. |
| **No on-chain simulation** | There is no public/external function to simulate post-trade health. The processor cannot call an on-chain view function to verify margin before submitting. |
| **Single point of failure** | The processor is the sole gatekeeper. A bug, misconfiguration, or compromise in the off-chain health simulation results in unchecked trades landing on-chain with no fallback enforcement. |

### Who is affected

- **All derivative traders** — bad debt from overleveraged positions is socialized across all holders of the affected product
- **Protocol solvency** — the insurance fund is depleted first, then remaining debt hits traders
- **The processor** — carries the full burden of health enforcement with no on-chain safety net

---

## Proof of Concept

A test demonstrating the vulnerable code path follows the real exploit path:

1. Buyer and seller deposit 1000 settlement units each via `EntryPoint.depositCollateral()`
2. Settlement delay is satisfied
3. `executeOrders` is called with 100 derivative contracts (as the processor would)
4. Trade succeeds — no revert
5. `AccountManager.computeHealth()` confirms deeply negative health on both INITIAL and MAINTENANCE types

**Expected result:**

```
  Buyer deposited 1000 units via EntryPoint.depositCollateral()
  Seller deposited 1000 units via EntryPoint.depositCollateral()

  >>> executeOrders SUCCEEDED -- no revert! <<<

  ---- On-Chain Health from AccountManager.computeHealth() ----
    INITIAL health (type=0):      -171538532816896252074514
    MAINTENANCE health (type=1):  -86460772085672289964177

  >>> INITIAL HEALTH IS NEGATIVE -- ACCOUNT IS INSOLVENT <<<
  >>> MAINTENANCE HEALTH IS NEGATIVE -- LIQUIDATABLE <<<

  ---- Leverage Analysis ----
    Effective leverage (X): 6806
    Max allowed leverage: ~40x (initial margin ratio = 0.975)
    Min collateral required for health >= 0: 171538532816896252074514
```

---

## Fix

Replace the `validateAccountHealth()` stub with an actual health check that queries `AccountManager.computeHealth()`:

```solidity
function validateAccountHealth(bytes32 account) internal view virtual returns (bool) {
    return IAccountManager(accountManager).computeHealth(
        account,
        IMarginEngine.HealthType.INITIAL
    ) >= 0;
}
```

This aligns `executeOrders` with every other state-changing function in the protocol, all of which enforce health through `AccountManager.computeHealth()`. The INITIAL health type should be used (consistent with deposit/withdrawal and position opening) to ensure positions remain within risk limits after trade execution.
