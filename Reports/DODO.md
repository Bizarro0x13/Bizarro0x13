# DODO — Bug Reports
https://audits.sherlock.xyz/contests/991
---

# [H-1] Attacker can claim other user's `refundAmount`

## Description

When a transaction fails while calling `gatewayZEVM`, the contract invokes the `onRevert` function. If the revert target address is a Solana address, a `refundInfo` is created for the corresponding `externalId`, which can later be claimed by the user.

```solidity
function onRevert(RevertContext calldata context) external onlyGateway {
    // 52 bytes = 32 bytes externalId + 20 bytes evmWalletAddress
    bytes32 externalId = bytes32(context.revertMessage[0:32]);
    bytes memory walletAddress = context.revertMessage[32:];

    if(context.revertMessage.length == 52) {
        address receiver = address(uint160(bytes20(walletAddress)));
        TransferHelper.safeTransfer(context.asset, receiver, context.amount);

        emit EddyCrossChainRevert(
            externalId,
            context.asset,
            context.amount,
            walletAddress
        );
    } else {
        RefundInfo memory refundInfo = RefundInfo({
            externalId: externalId,
            token: context.asset,
            amount: context.amount,
            walletAddress: walletAddress
        });

        refundInfos[externalId] = refundInfo;

        emit EddyCrossChainRefund(
            externalId,
            context.asset,
            context.amount,
            walletAddress
        );
    }
}
```

The problem lies in the `claimRefund` function, which only checks that `msg.sender` matches the stored `walletAddress` if the address is an EVM address, and skips any validation if the address is a Solana address.

As a result, an attacker could exploit this logic to claim refunds intended for other users.

```solidity
function claimRefund(bytes32 externalId) external {
    RefundInfo storage refundInfo = refundInfos[externalId];

    address receiver = msg.sender;

    if(refundInfo.walletAddress.length == 20) {
        receiver = address(uint160(bytes20(refundInfo.walletAddress)));
    }

    require(bots[msg.sender] || msg.sender == receiver, "INVALID_CALLER");
    require(refundInfo.externalId != "", "REFUND_NOT_EXIST");

    TransferHelper.safeTransfer(refundInfo.token, receiver, refundInfo.amount);

    delete refundInfos[externalId];

    emit EddyCrossChainRefundClaimed(
        externalId,
        refundInfo.token,
        refundInfo.amount,
        abi.encodePacked(msg.sender)
    );
}
```

---

## Impact

If the `claimRefund` function does not properly authenticate the claimant for non-EVM (e.g., Solana) addresses, any attacker can steal refunds intended for Solana users.

---

## Proof of Concept (POC)

1. A cross-chain transaction to a Solana address fails.
2. The `onRevert` function stores a `refundInfo` with:
   - `externalId`
   - `token`
   - `amount`
   - `walletAddress` (in this case, a raw Solana address as bytes)

3. Later, when `claimRefund()` is called:
   - If the refund was for an EVM address, it checks `msg.sender == walletAddress`.
   - But if the address is a Solana address, this check is skipped or bypassed.

4. A malicious user can:
   - Call `claimRefund()` with the correct `externalId` for any Solana recipient.
   - Steal their funds, even though they are not the intended recipient.

---

## Recommendation

Verify the caller's address even when the stored address is a Solana address and enforce proper authentication before allowing refunds to be claimed.

---

# [M-1] On-chain calculation of `amountInMax` allows sandwich attacks

## Description

The `_swapAndSendERC20Tokens` function in `GatewayCrossChain.sol` calculates the `amountInMax` parameter on-chain by querying the Uniswap pool for a quote and then applying a slippage percentage.

This approach is insecure because the on-chain price can be manipulated by attackers (e.g., via flash loans or sandwich attacks), leading to incorrect or exploitable swap amounts. As a result, the contract may approve and spend more tokens than necessary, or the transaction may revert unexpectedly.

```solidity
function _swapAndSendERC20Tokens(
    address targetZRC20,
    address gasZRC20,
    uint256 gasFee,
    uint256 targetAmount
) internal returns(uint256 amountsOut) {

    // Get amountOut for Input gasToken
    uint[] memory amountsQuote = UniswapV2Library.getAmountsIn(
        UniswapFactory,
        gasFee,
        getPathForTokens(targetZRC20, gasZRC20) // [targetZRC, gasZRC] or [targetZRC, WZETA, gasZRC]
    );

    uint amountInMax = (amountsQuote[0]) + (slippage * amountsQuote[0]) / 1000;

    IZRC20(targetZRC20).approve(UniswapRouter, amountInMax);

    // Swap TargetZRC20 to gasZRC20
    uint[] memory amounts = IUniswapV2Router01(UniswapRouter)
        .swapTokensForExactTokens(
            gasFee, // Amount of gas token required
            amountInMax,
            getPathForTokens(targetZRC20, gasZRC20),
            address(this),
            block.timestamp + MAX_DEADLINE
    );

    require(
        IZRC20(gasZRC20).balanceOf(address(this)) >= gasFee,
        "INSUFFICIENT_GAS_FOR_WITHDRAW"
    );

    require(
        targetAmount - amountInMax > 0,
        "INSUFFICIENT_AMOUNT_FOR_WITHDRAW"
    );

    IZRC20(gasZRC20).approve(address(gateway), gasFee);
    IZRC20(targetZRC20).approve(address(gateway), targetAmount - amounts[0]);

    amountsOut = targetAmount - amounts[0];
}
```

---

## Proof of Concept (POC)

1. User calls the `GatewaySend` contract to swap **USDC (mainnet)** for **USDT (Polygon)**.
2. The `GatewaySend` function calls the `GatewayCrossChain` contract's `onCall` function.
3. The `onCall` function swaps the `zUSDT` token for `ZETA` token to pay the `gasFee`.
4. The `_swapAndSendERC20Tokens` function is called to perform the swap.
5. A frontrunner observes the transaction in the mempool and manipulates the liquidity pool price before the swap executes.
6. This manipulation causes the user to spend more `zUSDT` to obtain the required `gasFee` in `ZETA`.
7. After the swap completes, the frontrunner restores the price to normal.

---

## Impact

Attackers can observe the transaction in the mempool and manipulate the pool state just before the swap is executed, causing the contract to overpay for swaps or lose funds.

---

## Recommendation

In the `depositAndCall` function in the `GatewaySend` contract, add a `minAmountIn` parameter in the encoded payload so that the user can enforce acceptable slippage limits and prevent manipulation.