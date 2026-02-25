# succinct-network — Bug Reports
https://cantina.xyz/competitions/bd882748-077e-4e55-853f-f8df70109dbb
---

## [M-1] Unfair Distribution of Rewards and Slashes During Unstake Period

### Summary
In the current implementation of the SuccinctStaking contract, when a user requests to unstake, there is a period (the "unstake period") before they can finalize and withdraw their tokens. During this period, slashes (penalties) are correctly applied to the unstaking user, but rewards (from dispensing) are distributed to all stakers, including those who are not unstaking. This creates an unfair economic incentive structure that can discourage participation and is biased against users who choose to unstake.

```js
 function _unstake(address _staker, address _prover, uint256 _stPROVE, uint256 _iPROVESnapshot)
        internal
        returns (uint256 PROVE)
    {
        // Ensure unstaking a non-zero amount.
        if (_stPROVE == 0) revert ZeroAmount();

        // Burn the $stPROVE from the staker
        _burn(_staker, _stPROVE);

        // Withdraw $PROVER-N from this contract to have this contract receive $iPROVE.
        uint256 iPROVEReceived = IERC4626(_prover).redeem(_stPROVE, address(this), address(this));

        // Determine how much $iPROVE to redeem for the staker based on rewards or slashing.
        uint256 iPROVE;
        if (iPROVEReceived > _iPROVESnapshot) {
            // Rewards were earned during unstaking. Return the excess to the prover.
            uint256 excess = iPROVEReceived - _iPROVESnapshot;
            IERC20(iProve).safeTransfer(_prover, excess);
            iPROVE = _iPROVESnapshot;
        } else {
            // Either no change or slashing occurred. Staker gets what's available.
            iPROVE = iPROVEReceived;
        }

        // Withdraw $iPROVE from this contract to have the staker receive $PROVE.
        PROVE = IERC4626(iProve).redeem(iPROVE, _staker, address(this));

        emit Unstake(_staker, _prover, PROVE, iPROVE, _stPROVE);
    }

```

## Impact Explanation
Users who are unstaking are penalized by slashes but do not benefit from rewards during the unstake period.

### Likelihood Explanation
Likelihood is High

### Proof of Concept
Paste this test in the `SuccinctStaking.slash.t.sol` and run `forge test --mt test_user_gets_even_while_unstaking -vvv`

```js
function test_user_gets_even_while_unstaking() public {
        uint256 stakeAmount = STAKER_PROVE_AMOUNT;

        uint256 balanceBefore = IERC20(PROVE).balanceOf(STAKER_1);
        _permitAndStake(STAKER_1, STAKER_1_PK, ALICE_PROVER, stakeAmount);
        _permitAndStake(STAKER_2, STAKER_2_PK, ALICE_PROVER, stakeAmount);

        _requestUnstake(STAKER_1, stakeAmount);

        uint256 slashAmount = SuccinctStaking(STAKING).proverStaked(ALICE_PROVER);
        _completeSlash(ALICE_PROVER, slashAmount);

        skip(UNSTAKE_PERIOD);

        _finishUnstake(STAKER_1);

        uint256 balanceAfter = IERC20(PROVE).balanceOf(STAKER_1);
        console.log("balanceAfter", balanceAfter); 

        assert(balanceAfter == balanceBefore); // This will revert as all the user balance gets slashed.


    }
```

### Recommendation
The contract should be modified so that both rewards and slashes during the unstake period are handled consistently. Either both should be applied to the unstaking user, or both should be distributed among all stakers.