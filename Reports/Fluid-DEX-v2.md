# Fluid-DEX-v2 — Bug Reports
https://audits.sherlock.xyz/contests/1225
---

# [H-1] Liquidity Drain via Uncapped Withdraw Amount in _processNormalSupplyAction

## Summary
In processNormalSupplyAction(), a user can request to withdraw more tokens than they have deposited.

The function correctly caps withdrawAmountRaw_ to the user’s actual balance (tokenRawSupply_). However, it does not update supplyAmount_ after this cap.

As a result, the internal accounting burns only the capped amount, but LIQUIDITY.operate() is called with the original uncapped supplyAmount_.

This allows an attacker to withdraw more tokens from the liquidity layer than they actually deposited.

## details
When a user withdraws:

withdrawAmountRaw_ is caculated from supplyAmount_
if withdrawAmountRaw > tokenRawSupply_, it is capped to tokenRawSupply_.
However, supplyAmount_ is not recalculated after this cap.
Later:

updateStorageForWithdraw() uses the capped withdrawAmountRaw (correct).
LIQUIDITY.operate() uses the original supplyAmount_ (uncapped).
This creates a mismatch between internal accounting and the external liquidity transfer.

```solidity
// Lines 705-742 (_processNormalSupplyAction)  
  
function _processNormalSupplyAction(  
    uint256 nftId_,  
    uint256 nftConfig_,  
    uint256 positionIndex_,  
    uint256 positionData_,  
    uint256 emode_,  
    bytes calldata actionData_  
) internal {  
    // ... token setup ...  
      
    // Decode action data  
    (int256 supplyAmount_, address to_) = abi.decode(actionData_, (int256, address));  
      
    if (supplyAmount_ > 0) {  
        // Supply logic (safe)  
    } else {  
          
        // Get exchange price  
        (uint256 supplyExchangePrice_, ) = _getExchangePrices(token_);  
          
        uint256 withdrawAmountRaw_;  
          
        if (supplyAmount_ == type(int256).min) {  
            // Full withdrawal (safe path)  
            withdrawAmountRaw_ = tokenRawSupply_;  
            uint256 withdrawAmount_ = ((tokenRawSupply_ * supplyExchangePrice_) - 1) / LC.EXCHANGE_PRICES_PRECISION;  
            if (withdrawAmount_ > 0) withdrawAmount_ -= 1;  
            supplyAmount_ = -int256(withdrawAmount_);  
        } else {  
              
            // Calculate raw amount from requested withdraw  
            withdrawAmountRaw_ = (((uint256(-supplyAmount_) * LC.EXCHANGE_PRICES_PRECISION) + 1) / supplyExchangePrice_) + 1;  
              
            // @note: Cap withdrawAmountRaw_ but DON'T update supplyAmount_  
            if (withdrawAmountRaw_ > tokenRawSupply_) {  
                withdrawAmountRaw_ = tokenRawSupply_;  // Cap raw amount  
                // @note MISSING: supplyAmount_ should be recalculated here!  
            }  
        }  
          
        _verifyAmountLimits(supplyAmount_);  
          
        // Update storage: Burns only withdrawAmountRaw_ (capped amount)  
        _updateStorageForWithdraw(  
            nftId_,   
            nftConfig_,   
            positionIndex_,   
            positionData_,   
            tokenIndex_,   
            tokenRawSupply_,   
            withdrawAmountRaw_  // Uses CAPPED amount (e.g., 1000 tokens)  
        );  
          
        _checkHf(nftId_, IS_OPERATE);  
          
        // @audit BUG: Passes UNCAPPED supplyAmount_ to Liquidity  
        LIQUIDITY.operate(  
            token_,   
            supplyAmount_,  // << Still contains INFLATED value (e.g., -1,000,000 tokens)  
            0,   
            to_,   
            address(0),   
            abi.encode(MONEY_MARKET_IDENTIFIER, NORMAL_WITHDRAW_ACTION_IDENTIFIER)  
        );  
        // Result:   
        // - Internal accounting burns 1000 tokens  
        // - Liquidity contract sends 1,000,000 tokens to attacker  
        // - Attacker profits: 999,000 tokens stolen  
    }  
}  
```
## Attack path
User requests withdrawal of 1,000,000 tokens (via supplyAmount_ = -1,000,000)
They only have 1,000 tokens deposited (tokenRawSupply_ = 1000)
Function calculates withdrawAmountRaw_ ≈ 1,000,000 (huge)
Function caps withdrawAmountRaw_ = 1000 (correct)
Function FAILS to update supplyAmount_ to match the capped amount
_updateStorageForWithdraw() receives 1000 (burns correct amount from user)
LIQUIDITY.operate() receives -1,000,000 (sends inflated amount to attacker)
Result: Attacker drains 999,000 tokens that don't belong to them.

## Impact
This allows a user to withdraw more tokens than they supplied.

- Internal accounting remains correct.
- External liquidity transfer is inflated.
- The difference is taken from protocol liquidity.
- An attacker can repeatedly exploit this to drain available liquidity.

## Recommendation
Recalculate the supply amount after capping
```solidity
function _processNormalSupplyAction(...) internal {  
    // ... existing code ...  
    if (supplyAmount_ > 0) {  
      
    } else {  
        // User is withdrawing  
        (uint256 supplyExchangePrice_, ) = _getExchangePrices(token_);  
          
        uint256 withdrawAmountRaw_;  
          
        if (supplyAmount_ == type(int256).min) {  
            // Full withdrawal (already safe)  
            withdrawAmountRaw_ = tokenRawSupply_;  
            uint256 withdrawAmount_ = ((tokenRawSupply_ * supplyExchangePrice_) - 1) / LC.EXCHANGE_PRICES_PRECISION;  
            if (withdrawAmount_ > 0) withdrawAmount_ -= 1;  
            supplyAmount_ = -int256(withdrawAmount_);  
        } else {  
            // Partial withdrawal  
            withdrawAmountRaw_ = (((uint256(-supplyAmount_) * LC.EXCHANGE_PRICES_PRECISION) + 1) / supplyExchangePrice_) + 1;  
              
            // Cap and recalculate supplyAmount_ if needed  
            if (withdrawAmountRaw_ > tokenRawSupply_) {  
                withdrawAmountRaw_ = tokenRawSupply_;  
                  
                // Recalculate supplyAmount_ to match capped withdrawAmountRaw_  
                uint256 actualWithdrawAmount = ((tokenRawSupply_ * supplyExchangePrice_) - 1) / LC.EXCHANGE_PRICES_PRECISION;  
                if (actualWithdrawAmount > 0) actualWithdrawAmount -= 1;  
                supplyAmount_ = -int256(actualWithdrawAmount);  
            }  
        }  
          
        _verifyAmountLimits(supplyAmount_);  
          
        _updateStorageForWithdraw(  
            nftId_,   
            nftConfig_,   
            positionIndex_,   
            positionData_,   
            tokenIndex_,   
            tokenRawSupply_,   
            withdrawAmountRaw_  
        );  
          
        _checkHf(nftId_, IS_OPERATE);  
          
        // Now supplyAmount_ is guaranteed to match withdrawAmountRaw_  
        LIQUIDITY.operate(token_, supplyAmount_, 0, to_, address(0), abi.encode(MONEY_MARKET_IDENTIFIER, NORMAL_WITHDRAW_ACTION_IDENTIFIER));  
    }  
}  
```

# [M-1] Pool Price Manipulation in D3 Liquidation Leading to Incorrect averageLiquidationPenalty_

## summary
In liquidate() function, when processing a D3 (smart collateral/concentrated liquidity) position withdrawal, the token supply amounts (token0SupplyAmount, token1SupplyAmount) are calculated using the manipulable pool price (AT_POOL_PRICE), while their USD values are computed using oracle prices. This inconsistency allows an attacker to manipulate the pool price within a single transaction to skew the averageLiquidationPenalty_ in their favor, extracting excess collateral from the borrower.

## details
```solidity
// STEP 1: Supply amounts calculated at MANIPULABLE pool price  
(  
    vl_.token0SupplyAmount,   
    vl_.token1SupplyAmount,   
    feeAccruedToken0_,   
    feeAccruedToken1_  
) = _getD3SupplyAmounts(  
    _getDexId(dexKey_),   
    GetD3D4AmountsParams({  
        ...  
        sqrtPriceX96: AT_POOL_PRICE  // ← pool price, manipulable via flash loan  
    })  
);  
  
// STEP 2: USD values computed using ORACLE prices (safe)  
uint256 supplyValue0_ = (((vl_.token0SupplyAmount * vl_.token0Price) + 1) / EIGHTEEN_DECIMALS) + 1;  
uint256 supplyValue1_ = (((vl_.token1SupplyAmount * vl_.token1Price) + 1) / EIGHTEEN_DECIMALS) + 1;  
  
// STEP 3: averageLiquidationPenalty weighted by MANIPULATED supply values  
averageLiquidationPenalty_ = (  
    (vl_.token0LiquidationPenalty * supplyValue0_) +   
    (vl_.token1LiquidationPenalty * supplyValue1_)  
) / supplyValue_;  
  
// STEP 4: withdrawValue scaled by MANIPULATED penalty  
v_.withdrawValue = ((v_.withdrawValue * (THREE_DECIMALS + averageLiquidationPenalty_)) - 1) / THREE_DECIMALS;  
```

In a Uniswap V3-style concentrated liquidity pool, the ratio of token0 to token1 in any liquidity position depends directly on the current sqrtPriceX96.

This means:
If price is near tickLower → position holds mostly token1
If price is near tickUpper → position holds mostly token0
Because AT_POOL_PRICE (the live pool price) is used to derive token amounts, an attacker can manipulate this price within the same transaction to control the token0/token1 ratio, which in turn controls the averageLiquidationPenalty_.

The pool price is NOT protected against within-block manipulation. Any actor with sufficient capital (e.g., via a flash loan) can shift the pool price before calling liquidate().

## Impact
Liquidators can artificially inflate averageLiquidationPenalty causing direct loss of user funds.

## Recommendation
Use the oracle-driven price to compute sqrtPriceX96 when fetching D3 supply amounts, ensuring both the token amount computation and the USD valuation use prices from the same trusted source

# [M-2] Permanent Fund Lock in _userStoredTokenAmount for EOA Recipients

## summary
When the Liquidity layer fails during settle(), the protocol falls back to storing _userStoredTokenAmount[to_][token_]. But, there is no mechanism for EOA addresses to claim these stored tokens, resulting in permanent fund loss for end users.

## description
The settle function contains a fallback mechanism that executed when LIQUIDITY.operate() reverts:

```solidity
// Lines 242-256  
  
else if (netAmount_ < 0) {  
    if (borrowAmount_ != 0) _unaccountedBorrowAmount[BASE_SLOT][token_] += borrowAmount_;  
  
    _userStoredTokenAmount[BASE_SLOT][to_][token_] += uint256(-netAmount_);  
  
    emit LogSettle(address(0), token_, 0, 0, -netAmount_, to_);  
}  
```

Here the _userStoredTokenAmount[BASE_SLOT][to_][token_] is increased, but there is no dedicated withdrawal function to claim stored tokens

However:

- there is no dedicated external withdrawal function
- the only way to consume stored balance is through settle()
- setle() is protected by _onlyAfterOperationStarted
- startOperation() required a callback execution pattern
- EOAs cannot execute callback-based flows.

## Failure Scenario
1. User(EOA) interacts with the moneyMarket contract
2. During settle(), LIQUIDITY.operate() reverts(eg. insufficient liquidity)
3. instead of reverting the entire transaction, the protocol credits: _userStoredTokenAmount[to_][token_]
4. Transaction completes successfully.
5. User believes funds are settled.
6. User later attempts to withdraw or recover.
7. No external claim mechanism exists.
8. EOA cannot call settle() directly.
9. Funds remain permanently locked.

## Impact
This causes permanent fund lock for users.

## Recommendation
Implement a function that allows userss to claim their stored token amounts without calling the startOperation function.