# octant-v2-core — Bug Reports
https://cantina.xyz/competitions/917d796b-48d0-41d0-bb40-be137b7d3db5
--- 

## [M-1] Loss Not Tracked in lossAmount When Dragon Router Shares Are Insufficient

### Summary
YieldDonatingTokenizedStrategy::report() function socializes the losses rather than keeping a lossAmount variable to track losses leading to stakers paying for the loss

### Finding Description
The report() function in YieldDonatingTokenizedStrategy attempts to protect user principal by burning shares from the dragonRouter to cover realized losses. However, if the dragonRouter does not have enough shares to cover the loss, the remaining loss is immediately socialized to all shareholders, reducing the overall principal rather than tracking it in a dedicated lossAmount storage variable. This behavior is contradictory to the intended loss protection mechanism detailed in the documentation, where uncovered losses should be tracked and offset against future profits before any new shares are minted to the dragonRouter.

Storage Variables: - lossAmount (uint256): Accumulated losses to offset against future profits - enableBurning (bool): Whether to burn shares from dragon router during loss protection

Loss Handling Process: 1. When losses occur, the system first attempts to burn dragon router shares to cover the loss 2. Any remaining loss that cannot be covered by burning is tracked in S.lossAmount 3. Future profits first offset tracked losses before minting new shares to dragon router
```js
function _handleDragonLossProtection(StrategyData storage S, uint256 loss) internal {
        if (S.enableBurning) {
            // Convert loss to shares that should be burned
            uint256 sharesToBurn = _convertToShares(S, loss, Math.Rounding.Ceil);

            // Can only burn up to available shares from dragon router
@>            uint256 sharesBurned = Math.min(sharesToBurn, S.balances[S.dragonRouter]);

            if (sharesBurned > 0) {
                // Convert shares to assets BEFORE burning to get correct value
                uint256 assetValueBurned = _convertToAssets(S, sharesBurned, Math.Rounding.Floor);

                // Burn shares from dragon router
@>                _burn(S, S.dragonRouter, sharesBurned);
                emit DonationBurned(S.dragonRouter, assetValueBurned);
            }
        }
    }
}
```
### Impact Explanation
Impact is High because user's principal will suffer due to the loss leading to their deposited asset amount going down to 0 as the profits are claimed by the dragonRouter and losses are socialized.

### Likelihood Explanation
Likelihood is Medium as it requires low dragon router balance and protocol losses in the same tx.

### Proof of Concept
paste this test in the test/unit/strategies/yieldDonating/Accounting.t.sol and run forge test --mt test_loss_socialized -vvv

```js
function test_loss_socialized() public {
        uint256 depositAmount = 100e18;
        address user1 = address(0x1);
        mintAndDepositIntoStrategy(strategy, user1, depositAmount); // user 1 deposits 100e18 tokens
        address user2 = address(0x2);
        mintAndDepositIntoStrategy(strategy, user2, depositAmount); // user 2 deposits 100e18 tokens

        createAndCheckProfit(strategy, 5e18); // strategy makes 5e18 profit increasing the balance of dragonRouter


        createAndCheckLoss(strategy, 10e18); // strategy makes 10e18 loss which is more than the profit made so far
        // so the loss will be socialized between user1 and user2 and dragon router

        uint256 user1Shares = strategy.balanceOf(user1);
        uint256 amountBeforeRedeem = asset.balanceOf(user1);
        vm.prank(user1);
        strategy.redeem(user1Shares, user1, user1); // user1 redeems all his shares
        uint256 amountAfterRedeem = asset.balanceOf(user1);
        uint256 assetRedeemed = amountAfterRedeem - amountBeforeRedeem; // 95121951219512195121
        console.log("user1 redeemed amount", assetRedeemed);
        assertNotEq(depositAmount, assetRedeemed); // 100e18 != 95.121951219512195121e18

    }
```

### Recommendation
keep a record of lossAmount when the dragonRouter balance is less than the loss.

## [M-2] Proposal Cancellation Does Not Reset recipientUsed Flag

### Summary
When the proposal is canceled for a recipient, the recipient cannot be used again because the recipientUsed flag is not reset during the cancellation process.

### Finding Description
When a proposer cancels a proposal in TokenizedAllocationMechanism, the contract does not reset the recipientUsed flag for the proposal’s recipient. This flag is set to true during proposal creation to prevent the same recipient from being assigned to multiple proposals. However, if a proposal is canceled (e.g., due to incorrect description or other reasons), the recipient remains blocked from being used in future proposals, even though the proposal is no longer active or valid.

```js
// In propose()
pid = ++s.proposalIdCounter;
s.proposals[pid] = Proposal(0, proposer, recipient, description, false);
s.recipientUsed[recipient] = true;
```
```js
// In cancelProposal()
p.canceled = true; // Does not reset s.recipientUsed[recipient]
```

### Impact Explanation
Impact is medium because if the proposer makes an error and cancels the proposal, there is no way to reuse the intended recipient address, resulting in permanent denial-of-service for that recipient.

### Likelihood Explanation
Likelihood is Medium.

### Proof of Concept
Alice proposes to send shares to Bob (recipient: Bob), but makes a Mistake in the description.
Alice cancels her proposal before the tally is finalized.
Alice tries to create a new proposal for Bob with the correct description.
The contract reverts with RecipientUsed(Bob), blocking the proposal.
Paste this test in QuadraticVotingE2E.sol and run forge test --mt test_cancelProposal

```js
function test_cancelProposal() public {
        // Mistake
        uint256 pid1 = _createProposal(alice, bob, "Edcation");
        vm.prank(alice);
        _tokenized(address(mechanism)).cancelProposal(pid1);
        _createProposal(alice, bob, "Education");
    }
│   │   └─ ← [Revert] RecipientUsed(0x0000000000000000000000000000000000000102)
    │   └─ ← [Revert] RecipientUsed(0x0000000000000000000000000000000000000102)
    └─ ← [Revert] RecipientUsed(0x0000000000000000000000000000000000000102)
```
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 14.17ms (5.68ms CPU time)

### Recommendation
Reset the recipientUsed flag to false when a proposal is canceled, allowing the recipient to be used in future proposals.