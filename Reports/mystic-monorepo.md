# Mystic-monorepo — Bug Reports
https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/overview
---

# [H-1] Wrong calculation of `currentWithheldETH` in restake and withdraw function can lead to unstaking the whole position.


## Summary
In stPlume's [unstake]() and [withdraw]() wrongly calculates the `currentWithheldETH` which can lead to the whole position getting unstaked from `plumeStaking` contract.

## Finding Description
If a user unstakes their position and the currentWithheldETH is less than the unstake amount, then currentWithheldETH is set to 0, and the withdrawal amount from plumeStaking is reduced by the previous value of currentWithheldETH. When the user withdraws their unstaked amount and the amount is greater than currentWithheldETH, the withdrawn amount is incorrectly reduced again. This results in the withdrawn amount being calculated as withdrawn amount - unstaked amount, even though the unstaked amount was already reduced earlier based on currentWithheldETH, which had been set to 0. As a result, the amount is effectively reduced twice, leading to incorrect currentWithheldETH calculations.

## Impact Explanation
The calculation of currentWithheldETH would always be wrong.

## Likelihood Explanation
Likelihood of this happening is high

## Proof of Concept
paste this test in `stPlumeMinter.t.sol` and run `forge test --mc StPlumeMinterForkTest --fork-url https://rpc.plume.org --mt test_wrong_calculation_in_withdraw_function -vvv`

```solidity
function test_wrong_calculation_in_withdraw_function() public {
        // deposit ether
        vm.prank(user1);
        minter.submit{value: 5 ether}();

        vm.prank(user2);
        minter.submit{value: 5 ether}();

        vm.deal(user2, 25 ether);
        vm.prank(user2);
        for (uint256 i = 0; i < 10; i++) {
            minter.submit{value: 0.09 ether}();
        }
        // rebalance increasing the currentWithheldETH.
        vm.prank(owner);
        minter.rebalance();

        uint256 withHeldEthBeforeUnstake = minter.currentWithheldETH();
        
        console.log("currentWithheldETH", minter.currentWithheldETH());

        // unstake for user 1 and 2.
        vm.prank(user1);
        frxETHToken.approve(address(minter), 1 ether);
        vm.prank(user1);
        minter.unstake(1 ether);

        vm.prank(user2);
        frxETHToken.approve(address(minter), 1 ether);
        vm.prank(user2);
        minter.unstake(1 ether);

        // now the withdrawable amount from plumeStaking would be 4 ether.

        // e deposit ether for user 3.
        address user3 = address(0xabcd);
        vm.deal(user3, 25 ether);
        vm.prank(user3);
        for (uint256 i = 0; i < 20; i++) {
            minter.submit{value: 0.09 ether}();
        }
        // rebalance increasing the currentWithheldETH to 1.8 ether.
        vm.prank(owner);
        minter.rebalance();

        vm.warp(block.timestamp + 3 days);

        assertEq(1.8 ether, minter.currentWithheldETH());
        
        // get the totalWithdrawableAmount and this should be total withdraw request by user 1 and user 2 - withHeldEthBeforeUnstake
        vm.prank(address(minter));
        uint256 amountWithdrawable = mockPlumeStaking.amountWithdrawable();
        assertEq(amountWithdrawable, 2 ether - withHeldEthBeforeUnstake);
        // user1 withdraws.
        // as withdraw for user1
        vm.prank(user1);
        uint256 amountWithdrawn = minter.withdraw(user1);
        // e this will revert.
        // current currentWithheldETH should be 1.9 ether
        console.log("currentWithheldETH after rebalance call", minter.currentWithheldETH());
        assertEq(minter.currentWithheldETH(), amountWithdrawable + withHeldEthBeforeUnstake - 1 ether);
    }
```

The function will revert
```
stPlumeMinter::currentWithheldETH() [staticcall]
    │   └─ ← [Return] 800055872686956224 [8e17]
    ├─ [0] VM::assertEq(800055872686956224 [8e17], 1000000000000000000 [1e18]) [staticcall]
    │   └─ ← [Revert] assertion failed: 800055872686956224 != 1000000000000000000
    └─ ← [Revert] assertion failed: 800055872686956224 != 1000000000000000000
```

## Recommendation
Reimplementing the whole unstake and withdraw function would be the best option.

# [H-2] The depositEther function will revert when _amount is equal to 0 causing all the `_rebalance` calls to revert.

## Metadata

- **Number:** #316
- **Severity:** High
- **Status:** Duplicate
- **Likelihood:** Medium
- **Impact:** High
- **Created by:** Bizarro
- **Created at:** May 17, 2025 at 1:22 PM
- **Last updated:** June 3, 2025 at 3:31 PM
- **Reward:** 16.44

## Description

## Summary
`stPlumeMinter::depositEther` function is called by the `stPlumeMinter::_rebalance` function which checks if _amount is greater than 0 or not, this will cause function to revert when there is no reward in plumeStaking contract.

## Finding Description
The depositEther() function is called during rebalancing and is responsible for depositing ETH into validators. However, it contains the following [check](https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/Liquid-Staking/src/stPlumeMinter.sol?lines=260,260)

```solidity
require(_amount > 0, "Amount must be greater than 0");
```
This results in reversion if _amount == 0 — even when this is a perfectly valid case during rebalance cycles. Since the _amount passed to depositEther() is derived from the contract’s ETH balance, the function will always revert when:

1. All user ETH is already staked.
2. There are no rewards to claim from validators.
3. A user tries to unstake() (which internally calls rebalance() ➝ depositEther(0)).

This causes permanent reverts for user unstake and withdraw operations — blocking protocol usability.

## Impact Explanation
Prevents users from unstaking or withdrawing under certain common conditions.

## Likelihood Explanation
The bug is critical to protocol availability: any time there's no ETH in the contract but all ETH is in staking, users are permanently locked out of withdrawals and unstaking.

## Proof of Concept
paste this test in `stPlumeMinter.t.sol` and run `forge test --mc StPlumeMinterForkTest --fork-url https://rpc.plume.org --mt test_depositEther_fails_when_amount_is_zero -vvvv`, this will revert with `[FAIL: Amount must be greater than 0]`.

```solidity
    function test_depositEther_fails_when_amount_is_zero() public {
        // mutiple users deposit ether to the minter contract.
        vm.prank(user1);
        minter.submit{value: 5 ether}();

        vm.prank(user2);
        minter.submit{value: 5 ether}();

        address user3 = address(0xabcd);
        vm.deal(user3, 15 ether);
        vm.prank(user3);
        for (uint256 i = 0; i < 20; i++) {
            minter.submit{value: 0.09 ether}();
        }

        // User2 unstake their amount causing the protocol to rebalance.
        vm.prank(user2);
        frxETHToken.approve(address(minter), 1 ether);
        vm.prank(user2);
        minter.unstake(1 ether);

        // User1 tries to unstake, this call will revert with [FAIL: Amount must be greater than 0]
        vm.prank(user1);
        frxETHToken.approve(address(minter), 1 ether);
        vm.prank(user1);
        minter.unstake(1 ether);
    }
```

```
stPlumeMinter::unstake(1000000000000000000 [1e18])
    │   ├─ [7488] 0xA20bfe49969D4a0E9abfdb6a46FeD777304ba07f::claim(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE)
    │   │   ├─ [6607] 0x431E7b32634dbefF77111c36A720945a3791aC85::claim(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE) [delegatecall]
    │   │   │   └─ ← [Return] 0
    │   │   └─ ← [Return] 0
    │   ├─ emit RewardClaimed(user: stPlumeMinter: [0xFa0F05A76eBcbf400856cCcCF29FF66d8acC843f], token: 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE, amount: 0)
    │   ├─ [4900] frxETH::minter_mint(stPlumeMinter: [0xFa0F05A76eBcbf400856cCcCF29FF66d8acC843f], 0)
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: stPlumeMinter: [0xFa0F05A76eBcbf400856cCcCF29FF66d8acC843f], value: 0)
    │   │   ├─ emit TokenMinterMinted(from: stPlumeMinter: [0xFa0F05A76eBcbf400856cCcCF29FF66d8acC843f], to: stPlumeMinter: [0xFa0F05A76eBcbf400856cCcCF29FF66d8acC843f], amount: 0)
    │   │   └─ ← [Stop]
    │   ├─ [3089] frxETH::transfer(sfrxETH: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 0)
    │   │   ├─ emit Transfer(from: stPlumeMinter: [0xFa0F05A76eBcbf400856cCcCF29FF66d8acC843f], to: sfrxETH: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], value: 0)
    │   │   └─ ← [Return] true
    │   └─ ← [Revert] Amount must be greater than 0
    └─ ← [Revert] Amount must be greater than 0
```

## Recommendation
in [_rebalance](https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/Liquid-Staking/src/stPlumeMinter.sol?lines=288,288) function add a zero amount check.
```diff
    function _rebalance() internal {
        uint256 amount = _claim();
+       if(amount == 0) return;
        frxETHToken.minter_mint(address(this), amount);
        frxETHToken.transfer(address(sfrxETHToken), amount);
        depositEther(address(this).balance);
    }
```

# [H-3] User withdrawal will revert because of wrong calculation of currentWithheldETH.

## Metadata

- **Number:** #307
- **Severity:** High
- **Status:** Duplicate
- **Created by:** Bizarro
- **Created at:** May 17, 2025 at 11:51 AM
- **Last updated:** May 26, 2025 at 8:21 PM
- **Assigned to:** Nyksx
- **Reward:** 6.24

## Description

## Summary
In `stPlumeMinter::unstake()` currentWithheldETH is not updated when currentWithheldETH is greater than the unstake amount which can lead of  withdraw failure for the user.

## Finding Description
In the unstake() function of the minter contract, when a user attempts to unstake an amount less than or equal to currentWithheldETH, the entire requested amount is considered unstaked. However, the value of currentWithheldETH is not updated (i.e., reduced by the amount used), leading to incorrect assumptions of available withheld ETH for subsequent users.

This results in:
1. Successful withdrawals for early users (e.g., user1)

2. Reverts for subsequent users (e.g., user2) due to [InvalidAmount(amount)](https://github.com/plumenetwork/contracts/blob/18738402d7e00dbdf0c831800c2d484ef3edd2e6/plume/src/facets/StakingFacet.sol#L434) (error selector 0x3728b83d)

## Impact Explanation
Funds may appear available to users during unstake but become inaccessible during withdrawal.

## Likelihood Explanation
High as it Prevents valid withdrawals and causes inconsistent and unfair behavior

## Proof of Concept
paste this test in `stPlumeMinter.t.sol` and run `forge test --mc StPlumeMinterForkTest --fork-url https://rpc.plume.org --mt test_reverts_when_withdrawing -vvv`.

```solidity
 function test_reverts_when_withdrawing() public {
        // Stake eth on behalf of user1, user2 and user3.
         vm.prank(user1);
        minter.submit{value: 5 ether}();

        vm.prank(user2);
        minter.submit{value: 5 ether}();

        address user3 = address(0xabcd);
        vm.deal(user3, 15 ether);
        vm.prank(user3);
        for (uint256 i = 0; i < 20; i++) {
            minter.submit{value: 0.09 ether}();
        }

        // Rebalance to increase the currentWithHeldETH
        vm.prank(owner);
        minter.rebalance();
        // currentWithheldETH = 1802160976259203458 (1.802 ether)
        console.log("currentWithheldETH before rebalance call", minter.currentWithheldETH());

        // Add some rewards to the contract
        vm.deal(address(mockPlumeStaking), address(mockPlumeStaking).balance + 1 ether);
        vm.deal(address(minter), 1 ether);

        // Unstake for user 1 
        // currentWithheldETH is greater than 1, so no amount will be unstaked from plumeStaking contract.
        vm.prank(user1);
        frxETHToken.approve(address(minter), 1 ether);
        vm.prank(user1);
        minter.unstake(1 ether);

        vm.deal(address(minter), 1 ether); 
        // Unstake for user 2
        // currentWithheldETH is greater than 1, so no amount will be unstaked from plumeStaking contract.
        vm.prank(user2);
        frxETHToken.approve(address(minter), 1 ether);
        vm.prank(user2);
        minter.unstake(1 ether);

        // Fast-forward past cooldown period
        vm.warp(block.timestamp + 3 days);

        vm.deal(address(minter), 1 ether);
        console.log("user 1 balance before withdraw", user1.balance); 

        vm.deal(address(minter), 1 ether);
        // Withdraw for user 
        // this will succeed since the currentWithheldETH is greater than 1.
        // currentWithheldETH = 802252589924368298 (0.8022 ether)
        vm.prank(user1);
        uint256 amountWithdrawn = minter.withdraw(user1);
        console.log("currentWithheldETH after rebalance call", minter.currentWithheldETH());
        console.log("user 1 balance after withdraw", user1.balance);
        vm.deal(address(minter), 1 ether);

        // this will fail since the currentWithheldETH is less than 1 and no amount was unstaked from plumeStaking.
        // this will revert with the error custom error 0x3728b83d
        vm.prank(user2);
        uint256 amountWithdrawn2 = minter.withdraw(user2);
    }
```

This will revert with `custom error 0x3728b83d` which is   ![Screenshot 2025-05-17 at 11.32.17 AM.png](https://imagedelivery.net/wtv4_V7VzVsxpAFaxzmpbw/5c79c540-498d-4665-e3f0-53feffebec00/public)  

and is present [here](https://github.com/plumenetwork/contracts/blob/18738402d7e00dbdf0c831800c2d484ef3edd2e6/plume/src/facets/StakingFacet.sol#L434)

```
 0xA20bfe49969D4a0E9abfdb6a46FeD777304ba07f::withdraw()
    │   │   ├─ [2698] 0x76eF355e6DdB834640a6924957D5B1d87b639375::withdraw() [delegatecall]
    │   │   │   └─ ← [Revert] custom error 0x3728b83d: 
    │   │   └─ ← [Revert] custom error 0x3728b83d: 
    │   └─ ← [Revert] custom error 0x3728b83d: 
    └─ ← [Revert] custom error 0x3728b83d:
```

## Recommendation
My suggestion would be creating a new storage variable to store `totalUnstakedAmount` and increase it everytime someone unstakes and and decrease the `currentWithheldETH` and when user withdraws decrease the `totalUnstakedAmount` recover the remaining by withdrawing from plumeStaking contract.

# [H-4]`stPlumeMinter::_rebalance` function reStakes the token balance rather than reward amount.

## Metadata

- **Number:** #235
- **Severity:** High
- **Status:** Duplicate
- **Likelihood:** High
- **Impact:** High
- **Created by:** Bizarro
- **Created at:** May 16, 2025 at 5:35 PM
- **Last updated:** June 3, 2025 at 4:33 PM
- **Reward:** 0.34

## Description

## Summary
`stPlumeMinter::_rebalance` after claiming rewards from plume staking contract stakes the token balance of the contract.

## Finding Description
The `stPlumeMinter::_rebalance` function claims rewards from the Plume staking contract and stakes them back into the same contract. However, instead of staking only the reward amount, the function stakes the entire native token balance of the contract. This causes all available funds, including `currentWithheldETH` and `withHoldETH`, to be staked. Moreover, these global state variables are not even updated, as seen [here](https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/Liquid-Staking/src/stPlumeMinter.sol?lines=257,257)

```solidity
    function _rebalance() internal {
        uint256 amount = _claim();
        frxETHToken.minter_mint(address(this), amount);
        frxETHToken.transfer(address(sfrxETHToken), amount);
@>      depositEther(address(this).balance);
    }
```

## Impact Explanation
This can lead to unnecessary reverts everytime user tries to withdraw their unStaked tokens leading to tokens being stuck. 

## Likelihood Explanation
Likelihood of this happening is high because everytime _rebalance function is called, the function makes the wrong deposit.

## Proof of Concept
Paste this test in `stPlumeMinter.t.sol` and run `forge test --mc StPlumeMinterForkTest --fork-url https://rpc.plume.org --mt test_rebalance_fail -vvv`

```solidity
 function test_rebalance_fail() public {
        address[2] memory users = [user1, user2];

        // Step 1: All users submit 5 ETH
        for (uint256 i = 0; i < users.length; i++) {
            vm.prank(users[i]);
            minter.submit{value: 5 ether}();
        }

        // Step 2: All users unstake 2 ETH
        for (uint256 i = 0; i < users.length; i++) {
            vm.startPrank(users[i]);
            frxETHToken.approve(address(minter), 2 ether);
            minter.unstake(2 ether);
            vm.stopPrank();
        }

        // Step 3: Only user1 withdraws after cooldown
        (uint256 requestAmount, uint256 requestTimestamp) = minter.withdrawalRequests(user1);
        assertEq(requestAmount, 2 ether);

        uint256 balanceBefore = user1.balance;

        vm.warp(requestTimestamp + 1 days);
        vm.startPrank(user1);
        minter.withdraw(user1);
        uint256 balanceBeforeRebalance = address(minter).balance;
        uint256 withheldETHBefore = (minter).currentWithheldETH();
        console.log("contract balance before rebalance call", balanceBeforeRebalance);
        console.log("currentWithheldETH before rebalance call", withheldETHBefore);
        vm.stopPrank();

        vm.prank(owner);
        minter.rebalance();

        uint256 balanceAfterRebalance = address(minter).balance;
        uint256 withheldETHAfter = (minter).currentWithheldETH();
        console.log("contract balance after rebalance call", balanceAfterRebalance);
        console.log("currentWithheldETH after rebalance call", withheldETHAfter);

        assertGt(user1.balance, balanceBefore + 2 ether - 0.1 ether); // Consider fee
        assertEq(frxETHToken.balanceOf(user1), 3 ether); // 5 - 2 = 3
    }
```
 this will return 

```sh
Logs:
  contract balance before rebalance call 2015702584502601532
  currentWithheldETH before rebalance call 1993543738535645960
  contract balance after rebalance call 0
  currentWithheldETH after rebalance call 1993543738535645960
```

Above we can see that the after rebalance the minter contract's eth balance becomes 0 but `currentWithheldEth` stays the same.

## Recommendation
```diff
    function _rebalance() internal {
        uint256 amount = _claim();
        frxETHToken.minter_mint(address(this), amount);
        frxETHToken.transfer(address(sfrxETHToken), amount);
-        depositEther(address(this).balance);
+        depositEther(amount);
    }
```

# [H-5]spentCollateral calculation is wrong in `MysticLeverageBundler::_createCloseLeverageBundleWithLoops`

## Metadata

- **Number:** #168
- **Severity:** High
- **Status:** Confirmed
- **Likelihood:** High
- **Impact:** Low
- **Created by:** Bizarro
- **Created at:** May 15, 2025 at 10:11 PM
- **Last updated:** May 31, 2025 at 4:30 PM
- **Reward:** 291.14

## Description

## Summary
In `MysticLeverageBundler::_createCloseLeverageBundleWithLoops` the calculation for spentCollateral is wrong which can lead to wrong collateral amount of user.

## Finding Description
In `MysticLeverageBundler::_createCloseLeverageBundleWithLoops` the function's aim is to close position of the user and transfer their collateral amount when the mysticPool's liquidity is less than the debtToClose.

The function first stores the calls in the mainBundle and then calls the multicall function from bundler. the `spentCollateral` amount calculated is wrong here because it converts the debtToClose - remainingDebt to collateral asset and the correct calculation should be collateralToWithdraw converted to collateralToken.


## Impact Explanation
Impact is high because the borrow amount of user would decrease but the collateral amount of user won't decrease the same amount leading to wrong leverage calculation

## Likelihood Explanation
Likelihood is high because this will happen everytime _createCloseLeverageBundleWithLoops is called.

## Proof of Concept
1. assume user has totalCollateralsPerUser as 200e18 and totalBorrowsPerUser as 100e18. and debtToClose/remainingDebt is 100e18.
2. collateralToWithdraw will be 200e18.
3. loop starts
4. mystic withdraws maxWithdrawable amount from the mysticPool, lets assume the amount(borrowable) is 50e18 collateralTokens.
5. maverick swaps the collateralTokens for assetTokens.
6. mystic repays the borrowed amount with the asset tokens.
7. remainingDebt = 100e18 - 50e18 = 50e18.
8. borrowable = 50e18 / 0.75 = 66.66e18.
9. loop runs again
10. at the end remainingDebt < borrowable, so. the remainingDebt will be 0 so the loop breaks.
11. in the last call we withdraw all the amount from the mysticPool and transfer it to user.
12. position is updated making the `totalBorrowsPerUser` as 100e18 - 100e18 =0 and `totalCollateralPerUser` as 200e18 - 100e18 = 100e18 which is wrong as it should be 0.

## Coded POC

change this [here](https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/bundler3/src/calls/MysticLeverageBundler.sol?lines=184,184) for the `_createCloseLeverageBundleWithLoops` call.

```diff
function createCloseLeverageBundle(address asset, address collateralAsset, uint256 debtToClose) external returns (Call[] memory bundle) {
      bytes32 pairKey = getPairKey(asset, collateralAsset);
      if(debtToClose == type(uint256).max || totalBorrowsPerUser[pairKey][msg.sender] <= debtToClose) {
        debtToClose = totalBorrowsPerUser[pairKey][msg.sender];
      }

      require(debtToClose > 0, "no debt found");
      
      // Check if there's enough liquidity for flashloan or if we're closing a small position
-      if (mysticAdapter.getAvailableLiquidity(asset) > debtToClose) {
+      if (mysticAdapter.getAvailableLiquidity(asset) < debtToClose) {
        return _createCloseLeverageBundleWithFlashloan(asset, collateralAsset, debtToClose);
      } else {
        return _createCloseLeverageBundleWithLoops(asset, collateralAsset, debtToClose);
      }
    }
    
```

Paste this test in `poc.t.sol` and run `forge test --fork-url https://phoenix-rpc.plumenetwork.xyz --mc POC_Test --mt test_closeLeverageBundle_is_wrong -vvv`

```solidity
function test_closeLeverageBundle_is_wrong() public {
         IBundler3 bundler2 = new Bundler3();
        MaverickSwapAdapter maverickAdapterMock2 = new MaverickSwapAdapter(
            maverickFactoryMock, maverickQuoterMock
        );
        MysticAdapter mysticAdapter = new MysticAdapter(
            address(bundler2), address(0xCE192A6E105cD8dd97b8Dedc5B5b263B52bb6AE0), address(0xEa237441c92CAe6FC17Caaf9a7acB3f953be4bd1)
        );
        MysticLeverageBundler leverageBundler2 = new MysticLeverageBundler(
            address(bundler2), address(mysticAdapter), address(maverickAdapterMock2)
        );
        vm.startPrank(USER);
        collateralToken.approve(address(mysticAdapter), type(uint256).max);
        ERC20Mock(0x9fbC367B9Bb966a2A537989817A088AFCaFFDC4c).approve(address(mysticAdapter), type(uint256).max);
        borrowToken.approve(address(mysticAdapter), type(uint256).max);
        ICreditDelegationToken(0xA9b705D4719002030386fa83087c905Ca4c25eB2).approveDelegation(
            address(mysticAdapter), type(uint128).max
        );
        IERC20(0xAf5aEAb2248415716569Be5d24FbE10b16590D6c).approve(address(mysticAdapter), type(uint128).max);
        IERC20(0xDb224c353CFB74e220b7B6cB12f8D8Bc7c8B2863).approve(address(mysticAdapter), type(uint128).max);
        IERC20(0x593cCcA4c4bf58b7526a4C164cEEf4003C6388db).approve(
            address(0xCE192A6E105cD8dd97b8Dedc5B5b263B52bb6AE0), type(uint128).max
        );
        ICreditDelegationToken(0xA9b705D4719002030386fa83087c905Ca4c25eB2).approveDelegation(
            address(mysticAdapter), type(uint128).max
        );
        IERC20(0xAf5aEAb2248415716569Be5d24FbE10b16590D6c).approve(address(mysticAdapter), type(uint128).max);
        IERC20(0xd1a7183708EF9706F3dD2d51B27a7e02a70F30fa).approve(address(mysticAdapter), type(uint128).max);
        IERC20(0x593cCcA4c4bf58b7526a4C164cEEf4003C6388db).approve(
            address(0xCE192A6E105cD8dd97b8Dedc5B5b263B52bb6AE0), type(uint128).max
        );
        vm.stopPrank();

        // create Open leverage
        vm.prank(USER);
        leverageBundler2.createOpenLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            address(collateralToken),
            INITIAL_COLLATERAL,
            LEVERAGE_2X,
            DEFAULT_SLIPPAGE
        );
        // close the whole position of the user.
        vm.prank(USER);
        leverageBundler2.createCloseLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            type(uint256).max
        );
        bytes32 pairKey = leverageBundler2.getPairKey(address(borrowToken), address(collateralToken));
        uint256 userCollateral = leverageBundler2.totalCollateralsPerUser(
            pairKey, USER
        );
        // this will revert.
        assertEq(userCollateral, 0);
    }
```

This will revert with 
```
MysticLeverageBundler::totalCollateralsPerUser(0x8b64e17695f3b5cc5de143055200076583a10ecb39665aa564c39ceb766a85b4, 0x18E1EEC9Fa5D77E472945FE0d48755386f28443c) [staticcall]
    │   └─ ← [Return] 99902 [9.99e4]
    ├─ [0] VM::assertEq(99902 [9.99e4], 0) [staticcall]
    │   └─ ← [Revert] assertion failed: 99902 != 0
    └─ ← [Revert] assertion failed: 99902 != 0
```

## Recommendation
```diff
function _createCloseLeverageBundleWithLoops(address asset, address collateralAsset, uint256 debtToClose) internal returns (Call[] memory bundle) {
      bytes32 pairKey = getPairKey(asset, collateralAsset);
      uint256 collateralToWithdraw = (totalCollateralsPerUser[pairKey][msg.sender] * debtToClose) / totalBorrowsPerUser[pairKey][msg.sender];
      uint256 leverage = (totalCollateralsPerUser[pairKey][msg.sender]) / (totalCollateralsPerUser[pairKey][msg.sender] - totalBorrowsPerUser[pairKey][msg.sender]);
      uint256 ltv = mysticAdapter.getAssetLtv(collateralAsset);
      uint8 numLoops = 20;
      uint256 remainingDebt = debtToClose;
      uint256 borrowable = mysticAdapter.getWithdrawableLiquidity(msg.sender, collateralAsset);
      Call[] memory mainBundle = new Call[](numLoops * 4+1); 
      
      // For each loop iteration
      for (uint8 i = 0; i < numLoops; i++) {
          uint256 baseIndex = i * 4;
          mainBundle[baseIndex + 0] = _createMysticWithdrawCall(collateralAsset, type(uint256).max, msg.sender, address(maverickAdapter));
          mainBundle[baseIndex + 1] = _createMaverickSwapCall(collateralAsset, asset, type(uint256).max, 0, 0, false);
          mainBundle[baseIndex + 2] = _createERC20TransferFromCall(asset, address(this), address(mysticAdapter), type(uint256).max);
          mainBundle[baseIndex + 3] = _createMysticRepayCall(asset, type(uint256).max, VARIABLE_RATE_MODE, msg.sender);
          
          remainingDebt = remainingDebt > borrowable? remainingDebt - borrowable:0;
          borrowable = borrowable * SLIPPAGE_SCALE/ ltv;
          if(remainingDebt == 0) break;
      }
      
      mainBundle[numLoops * 4] = _createMysticWithdrawCall(collateralAsset, type(uint256).max, msg.sender, msg.sender);
-      uint spentCollateral = getQuote(asset, collateralAsset, debtToClose - remainingDebt);
+      uint spentCollateral = getQuote(asset, collateralAsset, collateralToWithdraw);
      bundler.multicall(mainBundle);
      updatePositionTracking(pairKey, debtToClose - remainingDebt, spentCollateral, msg.sender, false);
      
      emit BundleCreated(msg.sender, keccak256("CLOSE_LEVERAGE_LOOPS"), mainBundle.length);
      emit LeverageClosed(msg.sender, collateralAsset, asset, collateralToWithdraw, totalCollaterals[pairKey], totalBorrows[pairKey]);
      
      return mainBundle;
  }
```

# [H-6]`MysticLeverageBundler::updateLeverageBundle` does not decrease user's leverage.

## Metadata

- **Number:** #140
- **Severity:** High
- **Status:** Duplicate
- **Likelihood:** High
- **Impact:** High
- **Created by:** Bizarro
- **Created at:** May 15, 2025 at 5:44 PM
- **Last updated:** May 26, 2025 at 5:18 AM

## Description

## Summary
When a user wants to decrease their newTargetLeverage by calling `MysticLeverageBundler::updateLeverageBundle` the user get's their collateral back making the leverage stay same and getting their tokens back.

## Finding Description
The aim of `MysticLeverageBundler::updateLeverageBundle` function is to increase or decrease their newTarget leverage. 
If the user wants to decrease their leverage the function first calculates the repayAmount, then takes a flashloan from mysticPool and repays the pool with the amount and then withdraws the `collateralForRepayment` amount to pay the pool for flash loan after swapping for assetToken and after paying the pool for flashLoan asset tokens are converted to the collateral token and sent back to the user, which is wrong because the amount transferred to the user should be supplied to the pool rather than user to maintain the new leverage.

1. Assume `totalCollateralsPerUser` is 200e18 ( user has 200e18 aTokens ) and `totalBorrowsPerUser`  is 100e18 maintaining a leverage of 2X
2. user wants to decrease the leverage from 2X to 1.5X, this makes the repayAmount to be 50e18 and totalCollateralForRepayment 100e18
3. function loans 50e18 and repays the pool the 50e18.
4. function then withdraws 100e18 collateralTokens (making the user balance as 100e18 aToken )which are then swapped for 100e18 asset tokens.
5. from the 100e18 assetTokens we pay the loan amount which is 50e18 + fee (assuming 50e18 for simplicity).
6. remaining 50e18 assetTokens will be converted to 50e18 collateralTokens and then transferred to the user.

Initially user had 200e18 atokens and borrowAmount of 100e18 making the  leverage 2X but after the function execution user has 100e18 aToken and 50e18 Collateral tokens and the borrowAmount is 50e18 making the leverage again 2X.

```solidity
function updateLeverageBundle(
      address asset,
      address collateralAsset,
      uint256 newTargetLeverage,
      uint256 slippageTolerance
  ) external returns (Call[] memory bundle) {
      require(newTargetLeverage > SLIPPAGE_SCALE, "Leverage must be > 1");
      require(newTargetLeverage <= 1000000, "Leverage too high"); // Max 100x
      bytes32 pairKey = getPairKey(asset, collateralAsset);
      uint256 slippage = slippageTolerance == 0 ? DEFAULT_SLIPPAGE : slippageTolerance;
      uint256 currentCollateral = totalCollateralsPerUser[pairKey][msg.sender];
      uint256 currentBorrow = totalBorrowsPerUser[pairKey][msg.sender];

      require(currentCollateral > 0 && currentBorrow > 0, "No existing position");
      uint256 currentLeverage = (currentCollateral * SLIPPAGE_SCALE) / (currentCollateral - currentBorrow);
      uint256 newBorow =  currentBorrow * (newTargetLeverage - SLIPPAGE_SCALE) * currentLeverage / (newTargetLeverage * (currentLeverage - SLIPPAGE_SCALE)); //getQuote(collateralAsset, asset, currentCollateral * (newTargetLeverage - SLIPPAGE_SCALE) / newTargetLeverage);
      int256 borrowDelta = int256(newBorow) - int256(currentBorrow);
      
      // Create appropriate bundles based on the operation type
      Call[] memory mainBundle;
      Call[] memory flashloanCallbackBundle;
 
      if (borrowDelta > 0) {
          // INCREASE LEVERAGE CASE
        uint256 additionalBorrowAmount = uint256(borrowDelta);
        uint totalBorrowAmount = additionalBorrowAmount + mysticAdapter.flashLoanFee(additionalBorrowAmount) + 1;

        mainBundle = new Call[](1);
        flashloanCallbackBundle = new Call[](5);
        flashloanCallbackBundle[0] = _createERC20TransferCall(asset, address(maverickAdapter), type(uint256).max);
        flashloanCallbackBundle[1] = _createMaverickSwapCall(asset, collateralAsset, type(uint256).max, 0, slippage, false);
        flashloanCallbackBundle[2] = _createERC20TransferFromCall(collateralAsset, address(this), address(mysticAdapter), type(uint256).max);
        flashloanCallbackBundle[3] = _createMysticSupplyCall( collateralAsset, type(uint256).max, msg.sender);
        flashloanCallbackBundle[4] = _createMysticBorrowCall(asset, totalBorrowAmount, VARIABLE_RATE_MODE, msg.sender, address(mysticAdapter));

        mainBundle[0] = _createMysticFlashloanCall(asset,additionalBorrowAmount,false,abi.encode(flashloanCallbackBundle));
        updatePositionTracking(pairKey, additionalBorrowAmount, 0, msg.sender, true);
      } else if (borrowDelta < 0) {
        uint256 repayAmount = uint256(-borrowDelta);
        uint256 collateralForRepayment = (totalCollateralsPerUser[pairKey][msg.sender] * repayAmount) / totalBorrowsPerUser[pairKey][msg.sender];
          
        mainBundle = new Call[](5);
        flashloanCallbackBundle = new Call[](4);

        flashloanCallbackBundle[0] = _createMysticRepayCall(asset, repayAmount, VARIABLE_RATE_MODE, msg.sender);
        flashloanCallbackBundle[1] = _createMysticWithdrawCall(collateralAsset, collateralForRepayment, msg.sender, address(maverickAdapter));
        flashloanCallbackBundle[2] = _createMaverickSwapCall(collateralAsset, asset, collateralForRepayment, repayAmount , 0, false);
        flashloanCallbackBundle[3] = _createERC20TransferFromCall(asset, address(this), address(mysticAdapter), type(uint256).max);
        // Compressed main bundle creation
        mainBundle[0] = _createMysticFlashloanCall(asset,repayAmount,false, abi.encode(flashloanCallbackBundle));
        mainBundle[1] = _createERC20TransferCall(asset,address(maverickAdapter), type(uint256).max);
        mainBundle[2] = _createMaverickSwapCall(asset, collateralAsset, type(uint256).max, 0 , 0, false);
        mainBundle[3] = _createERC20TransferCall(collateralAsset,address(this), type(uint256).max);
@>        mainBundle[4] = _createERC20TransferFromCall(collateralAsset, address(this), msg.sender, type(uint256).max);
        updatePositionTracking(pairKey, repayAmount, 0, msg.sender, false);
      } else {
          revert("No changes to position");
      }
      
      bundler.multicall(mainBundle);
      
      emit BundleCreated(msg.sender, keccak256("UPDATE_LEVERAGE"), mainBundle.length);
      emit LeverageUpdated(msg.sender, collateralAsset, asset, currentCollateral, currentLeverage, newTargetLeverage, totalCollaterals[pairKey], totalBorrows[pairKey]);
      
      return mainBundle;
  }
```

## Impact Explanation
The impact is high as user can call the decrease the leverage without actually decreasing leverage and risking token.

## Likelihood Explanation
Likelihood is high as this will happen everytime updateLeverageBundle function is called

## Proof of Concept
copy this test in `MysticLeverageBundlerForkTest.sol` and run `forge test --fork-url https://phoenix-rpc.plumenetwork.xyz --mc POC_Test --mt test_sending_amount_to_user -vvv`

```solidity
 function test_sending_amount_to_user() external {
       uint256 collateralAmount = 100e6;
        uint256 balanceBeforeDeposit = IERC20(collateralToken).balanceOf(USER);
        vm.prank(USER);
        leverageBundler.createOpenLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            address(collateralToken),
            collateralAmount,
            20000,
            DEFAULT_SLIPPAGE
        );
        uint256 AmountDeposited = balanceBeforeDeposit - balanceBeforeUpdate;
        console.log("Amount deposited: ",AmountDeposited);
        
        uint256 balanceBeforeUpdate = IERC20(collateralToken).balanceOf(USER);
        PositionData memory userDataBefore = getPositionData(USER, address(borrowToken), address(collateralToken));
        console.log("totalBorrowsUser before updating leverage: ",userDataBefore.totalBorrowsUser);


        // Update leverage: increase from current to higher
        vm.prank(USER);
        Call[] memory bundleCalls = leverageBundler.updateLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            15000, // decrease leverage
            DEFAULT_SLIPPAGE
        );
        uint256 balanceAfter = IERC20(collateralToken).balanceOf(USER);

        uint256 amountReturned = balanceAfter - balanceBeforeUpdate;
        console.log("Amount returned: ",amountReturned);

        PositionData memory userDataAfter = getPositionData(USER, address(borrowToken), address(collateralToken));
        console.log("totalBorrowsUser after updating leverage: ",userDataAfter.totalBorrowsUser);

    }
```

This test will return 

```
  Amount deposited:  100000000
  totalBorrowsUser before updating leverage:  102310571
  Amount returned:  53448714
  totalBorrowsUser after updating leverage:  47560806

```
here we can see that before updating the leverage the leverage is nearly 2X and after updating the leverage, the leverage is nearly the same and user has received tokens from the protocol.

## Recommendation
Rather than transferring the collateral token to the user, the function should supply the token to mysticPool, that will make the leverage user wanted.

```diff
function updateLeverageBundle(
      address asset,
      address collateralAsset,
      uint256 newTargetLeverage,
      uint256 slippageTolerance
  ) external returns (Call[] memory bundle) {
      require(newTargetLeverage > SLIPPAGE_SCALE, "Leverage must be > 1");
      require(newTargetLeverage <= 1000000, "Leverage too high"); // Max 100x
      bytes32 pairKey = getPairKey(asset, collateralAsset);
      uint256 slippage = slippageTolerance == 0 ? DEFAULT_SLIPPAGE : slippageTolerance;
      uint256 currentCollateral = totalCollateralsPerUser[pairKey][msg.sender];
      uint256 currentBorrow = totalBorrowsPerUser[pairKey][msg.sender];

      require(currentCollateral > 0 && currentBorrow > 0, "No existing position");
      uint256 currentLeverage = (currentCollateral * SLIPPAGE_SCALE) / (currentCollateral - currentBorrow);
      uint256 newBorow =  currentBorrow * (newTargetLeverage - SLIPPAGE_SCALE) * currentLeverage / (newTargetLeverage * (currentLeverage - SLIPPAGE_SCALE)); //getQuote(collateralAsset, asset, currentCollateral * (newTargetLeverage - SLIPPAGE_SCALE) / newTargetLeverage);
      int256 borrowDelta = int256(newBorow) - int256(currentBorrow);
      
      // Create appropriate bundles based on the operation type
      Call[] memory mainBundle;
      Call[] memory flashloanCallbackBundle;
 
      if (borrowDelta > 0) {
          // INCREASE LEVERAGE CASE
        uint256 additionalBorrowAmount = uint256(borrowDelta);
        uint totalBorrowAmount = additionalBorrowAmount + mysticAdapter.flashLoanFee(additionalBorrowAmount) + 1;

        mainBundle = new Call[](1);
        flashloanCallbackBundle = new Call[](5);
        flashloanCallbackBundle[0] = _createERC20TransferCall(asset, address(maverickAdapter), type(uint256).max);
        flashloanCallbackBundle[1] = _createMaverickSwapCall(asset, collateralAsset, type(uint256).max, 0, slippage, false);
        flashloanCallbackBundle[2] = _createERC20TransferFromCall(collateralAsset, address(this), address(mysticAdapter), type(uint256).max);
        flashloanCallbackBundle[3] = _createMysticSupplyCall( collateralAsset, type(uint256).max, msg.sender);
        flashloanCallbackBundle[4] = _createMysticBorrowCall(asset, totalBorrowAmount, VARIABLE_RATE_MODE, msg.sender, address(mysticAdapter));

        mainBundle[0] = _createMysticFlashloanCall(asset,additionalBorrowAmount,false,abi.encode(flashloanCallbackBundle));
        updatePositionTracking(pairKey, additionalBorrowAmount, 0, msg.sender, true);
      } else if (borrowDelta < 0) {
        uint256 repayAmount = uint256(-borrowDelta);
        uint256 collateralForRepayment = (totalCollateralsPerUser[pairKey][msg.sender] * repayAmount) / totalBorrowsPerUser[pairKey][msg.sender];
          
        mainBundle = new Call[](5);
        flashloanCallbackBundle = new Call[](4);

        flashloanCallbackBundle[0] = _createMysticRepayCall(asset, repayAmount, VARIABLE_RATE_MODE, msg.sender);
        flashloanCallbackBundle[1] = _createMysticWithdrawCall(collateralAsset, collateralForRepayment, msg.sender, address(maverickAdapter));
        flashloanCallbackBundle[2] = _createMaverickSwapCall(collateralAsset, asset, collateralForRepayment, repayAmount , 0, false);
        flashloanCallbackBundle[3] = _createERC20TransferFromCall(asset, address(this), address(mysticAdapter), type(uint256).max);
        // Compressed main bundle creation
        mainBundle[0] = _createMysticFlashloanCall(asset,repayAmount,false, abi.encode(flashloanCallbackBundle));
        mainBundle[1] = _createERC20TransferCall(asset,address(maverickAdapter), type(uint256).max);
        mainBundle[2] = _createMaverickSwapCall(asset, collateralAsset, type(uint256).max, 0 , 0, false);
        mainBundle[3] = _createERC20TransferCall(collateralAsset,address(this), type(uint256).max);
-        mainBundle[4] = _createERC20TransferFromCall(collateralAsset, address(this), msg.sender, type(uint256).max);
+        mainBundle[4] = _createMysticSupplyCall( collateralAsset, type(uint256).max, msg.sender);
        updatePositionTracking(pairKey, repayAmount, 0, msg.sender, false);
      } else {
          revert("No changes to position");
      }
      
      bundler.multicall(mainBundle);
      
      emit BundleCreated(msg.sender, keccak256("UPDATE_LEVERAGE"), mainBundle.length);
      emit LeverageUpdated(msg.sender, collateralAsset, asset, currentCollateral, currentLeverage, newTargetLeverage, totalCollaterals[pairKey], totalBorrows[pairKey]);
      
      return mainBundle;
  }
```

# [H-7]totalCollateralsPerUser is not updated when user calls `MysticLeverageBundler::updateLeverageBundle`.

## Metadata

- **Number:** #133
- **Severity:** High
- **Status:** Duplicate
- **Likelihood:** High
- **Impact:** High
- **Created by:** Bizarro
- **Created at:** May 15, 2025 at 4:00 PM
- **Last updated:** May 26, 2025 at 5:18 AM
- **Reward:** 41.77

## Description

## Summary
When user calls [MysticLeverageBundler::updateLeverageBundle](https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/bundler3/src/calls/MysticLeverageBundler.sol?lines=256,256) user's `totalCollateralPerUser` is not updated leading to wrong calculation of collateral amount and borrow amount.

## Finding Description
In [MysticLeverageBundler::updateLeverageBundle](https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/bundler3/src/calls/MysticLeverageBundler.sol?lines=256,256) user can increase and decrease their previous leverage to newTargetLeverage. If the newtargetLeverage is greater than previous leverage then the protocol flashloans assetToken from the mysticPool and supplies the collateralAsset to the mysticPool after swapping the assetToken for collateralAsset from maverick and at last borrows some amount from mystic pool to pay the flashLoan back to the pool.
the new supplied amount is the new collateral added on behalf of user after borrowing which should increase the `totalBorrowsPerUser` and `totalCollateralPerUser` but only `totalBorrowsPerUser` is updated, this could lead to wrong leverage calculation and withdraw amount when user decides to close their position.

The same can be seen when the user decides to decrease the newTargetLeverage.

```solidity
function updateLeverageBundle(
      address asset,
      address collateralAsset,
      uint256 newTargetLeverage,
      uint256 slippageTolerance
  ) external returns (Call[] memory bundle) {
      require(newTargetLeverage > SLIPPAGE_SCALE, "Leverage must be > 1");
      require(newTargetLeverage <= 1000000, "Leverage too high"); // Max 100x
      bytes32 pairKey = getPairKey(asset, collateralAsset);
      uint256 slippage = slippageTolerance == 0 ? DEFAULT_SLIPPAGE : slippageTolerance;
      uint256 currentCollateral = totalCollateralsPerUser[pairKey][msg.sender];
      uint256 currentBorrow = totalBorrowsPerUser[pairKey][msg.sender];

      require(currentCollateral > 0 && currentBorrow > 0, "No existing position");
      uint256 currentLeverage = (currentCollateral * SLIPPAGE_SCALE) / (currentCollateral - currentBorrow);
      uint256 newBorow =  currentBorrow * (newTargetLeverage - SLIPPAGE_SCALE) * currentLeverage / (newTargetLeverage * (currentLeverage - SLIPPAGE_SCALE)); //getQuote(collateralAsset, asset, currentCollateral * (newTargetLeverage - SLIPPAGE_SCALE) / newTargetLeverage);
      int256 borrowDelta = int256(newBorow) - int256(currentBorrow);
      
      // Create appropriate bundles based on the operation type
      Call[] memory mainBundle;
      Call[] memory flashloanCallbackBundle;
 
      if (borrowDelta > 0) {
          // INCREASE LEVERAGE CASE
        uint256 additionalBorrowAmount = uint256(borrowDelta);
        uint totalBorrowAmount = additionalBorrowAmount + mysticAdapter.flashLoanFee(additionalBorrowAmount) + 1;

        mainBundle = new Call[](1);
        flashloanCallbackBundle = new Call[](5);
        flashloanCallbackBundle[0] = _createERC20TransferCall(asset, address(maverickAdapter), type(uint256).max);
        flashloanCallbackBundle[1] = _createMaverickSwapCall(asset, collateralAsset, type(uint256).max, 0, slippage, false);
        flashloanCallbackBundle[2] = _createERC20TransferFromCall(collateralAsset, address(this), address(mysticAdapter), type(uint256).max);
        flashloanCallbackBundle[3] = _createMysticSupplyCall( collateralAsset, type(uint256).max, msg.sender);
        flashloanCallbackBundle[4] = _createMysticBorrowCall(asset, totalBorrowAmount, VARIABLE_RATE_MODE, msg.sender, address(mysticAdapter));

        mainBundle[0] = _createMysticFlashloanCall(asset,additionalBorrowAmount,false,abi.encode(flashloanCallbackBundle));
@>   updatePositionTracking(pairKey, additionalBorrowAmount, 0, msg.sender, true);
      } else if (borrowDelta < 0) {
        uint256 repayAmount = uint256(-borrowDelta);
        uint256 collateralForRepayment = (totalCollateralsPerUser[pairKey][msg.sender] * repayAmount) / totalBorrowsPerUser[pairKey][msg.sender];
          
        mainBundle = new Call[](5);
        flashloanCallbackBundle = new Call[](4);

        flashloanCallbackBundle[0] = _createMysticRepayCall(asset, repayAmount, VARIABLE_RATE_MODE, msg.sender);
        flashloanCallbackBundle[1] = _createMysticWithdrawCall(collateralAsset, collateralForRepayment, msg.sender, address(maverickAdapter));
        flashloanCallbackBundle[2] = _createMaverickSwapCall(collateralAsset, asset, collateralForRepayment, repayAmount , 0, false);
        flashloanCallbackBundle[3] = _createERC20TransferFromCall(asset, address(this), address(mysticAdapter), type(uint256).max);
        // Compressed main bundle creation
        mainBundle[0] = _createMysticFlashloanCall(asset,repayAmount,false, abi.encode(flashloanCallbackBundle));
        mainBundle[1] = _createERC20TransferCall(asset,address(maverickAdapter), type(uint256).max);
        mainBundle[2] = _createMaverickSwapCall(asset, collateralAsset, type(uint256).max, 0 , 0, false);
        mainBundle[3] = _createERC20TransferCall(collateralAsset,address(this), type(uint256).max);
        mainBundle[4] = _createERC20TransferFromCall(collateralAsset, address(this), msg.sender, type(uint256).max);
@>  updatePositionTracking(pairKey, repayAmount, 0, msg.sender, false);
      } else {
          revert("No changes to position");
      }
      
      bundler.multicall(mainBundle);
      
      emit BundleCreated(msg.sender, keccak256("UPDATE_LEVERAGE"), mainBundle.length);
      emit LeverageUpdated(msg.sender, collateralAsset, asset, currentCollateral, currentLeverage, newTargetLeverage, totalCollaterals[pairKey], totalBorrows[pairKey]);
      
      return mainBundle;
  }
```

## Impact Explanation
Impact is high because the main storage variables are storing wrong info which can lead to wrong leverage calculation and withdraw amount.

## Likelihood Explanation
Likelihood is high because the attacker can easily manipulate the values by calling updateLeverageBundle function

## Proof of Concept
copy this test in `MysticLeverageBundlerForkTest.sol` and run `forge test --fork-url https://phoenix-rpc.plumenetwork.xyz --mc MysticLeverageBundlerRWATest --mt test_update_leverage_isNot_updating_collateral -vvvv`

```solidity
    function test_update_leverage_isNot_updating_collateral() external {
        PositionData memory before = _createInitialPosition();

        PositionData memory userDataBefore = getPositionData(USER, address(borrowToken), address(collateralToken));

        // Update leverage: increase from current to higher
        vm.prank(USER);
        Call[] memory bundleCalls = leverageBundler.updateLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            LEVERAGE_3X, // Increase leverage
            DEFAULT_SLIPPAGE
        );

        PositionData memory userDataAfter = getPositionData(USER, address(borrowToken), address(collateralToken));

        assertEq(userDataAfter.totalCollateralUser-userDataBefore.totalCollateralUser, INITIAL_COLLATERAL);
    }
```
this will fail with error

```
 0x593cCcA4c4bf58b7526a4C164cEEf4003C6388db::balanceOf(0x18E1EEC9Fa5D77E472945FE0d48755386f28443c) [staticcall]
    │   └─ ← [Return] 99999900000 [9.999e10]
    ├─ [621] 0x593cCcA4c4bf58b7526a4C164cEEf4003C6388db::balanceOf(0x598Fc8cD4335D5916Fa81Ec0Efa25b462aA721F1) [staticcall]
    │   └─ ← [Return] 0
    ├─ [621] 0x593cCcA4c4bf58b7526a4C164cEEf4003C6388db::balanceOf(0xE2314ECb6Ae07a987018a71e412897ED2F54E075) [staticcall]
    │   └─ ← [Return] 0
    ├─ [0] VM::assertEq(0, 100000 [1e5]) [staticcall]
    │   └─ ← [Revert] assertion failed: 0 != 100000
    └─ ← [Revert] assertion failed: 0 != 100000
```

## Recommendation

```diff
function updateLeverageBundle(
      address asset,
      address collateralAsset,
      uint256 newTargetLeverage,
      uint256 slippageTolerance
  ) external returns (Call[] memory bundle) {
      require(newTargetLeverage > SLIPPAGE_SCALE, "Leverage must be > 1");
      require(newTargetLeverage <= 1000000, "Leverage too high"); // Max 100x
      bytes32 pairKey = getPairKey(asset, collateralAsset);
      uint256 slippage = slippageTolerance == 0 ? DEFAULT_SLIPPAGE : slippageTolerance;
      uint256 currentCollateral = totalCollateralsPerUser[pairKey][msg.sender];
      uint256 currentBorrow = totalBorrowsPerUser[pairKey][msg.sender];

      require(currentCollateral > 0 && currentBorrow > 0, "No existing position");
      uint256 currentLeverage = (currentCollateral * SLIPPAGE_SCALE) / (currentCollateral - currentBorrow);
      uint256 newBorow =  currentBorrow * (newTargetLeverage - SLIPPAGE_SCALE) * currentLeverage / (newTargetLeverage * (currentLeverage - SLIPPAGE_SCALE)); //getQuote(collateralAsset, asset, currentCollateral * (newTargetLeverage - SLIPPAGE_SCALE) / newTargetLeverage);
      int256 borrowDelta = int256(newBorow) - int256(currentBorrow);
      
      // Create appropriate bundles based on the operation type
      Call[] memory mainBundle;
      Call[] memory flashloanCallbackBundle;
 
      if (borrowDelta > 0) {
          // INCREASE LEVERAGE CASE
        uint256 additionalBorrowAmount = uint256(borrowDelta);
        uint totalBorrowAmount = additionalBorrowAmount + mysticAdapter.flashLoanFee(additionalBorrowAmount) + 1;

        mainBundle = new Call[](1);
        flashloanCallbackBundle = new Call[](5);
        flashloanCallbackBundle[0] = _createERC20TransferCall(asset, address(maverickAdapter), type(uint256).max);
        flashloanCallbackBundle[1] = _createMaverickSwapCall(asset, collateralAsset, type(uint256).max, 0, slippage, false);
        flashloanCallbackBundle[2] = _createERC20TransferFromCall(collateralAsset, address(this), address(mysticAdapter), type(uint256).max);
        flashloanCallbackBundle[3] = _createMysticSupplyCall( collateralAsset, type(uint256).max, msg.sender);
        flashloanCallbackBundle[4] = _createMysticBorrowCall(asset, totalBorrowAmount, VARIABLE_RATE_MODE, msg.sender, address(mysticAdapter));

        mainBundle[0] = _createMysticFlashloanCall(asset,additionalBorrowAmount,false,abi.encode(flashloanCallbackBundle));
-        updatePositionTracking(pairKey, additionalBorrowAmount, 0, msg.sender, true);
+       updatePositionTracking(pairKey, additionalBorrowAmount, additionalBorrowAmount, msg.sender, true);
      } else if (borrowDelta < 0) {
        uint256 repayAmount = uint256(-borrowDelta);
        uint256 collateralForRepayment = (totalCollateralsPerUser[pairKey][msg.sender] * repayAmount) / totalBorrowsPerUser[pairKey][msg.sender];
          
        mainBundle = new Call[](5);
        flashloanCallbackBundle = new Call[](4);

        flashloanCallbackBundle[0] = _createMysticRepayCall(asset, repayAmount, VARIABLE_RATE_MODE, msg.sender);
        flashloanCallbackBundle[1] = _createMysticWithdrawCall(collateralAsset, collateralForRepayment, msg.sender, address(maverickAdapter));
        flashloanCallbackBundle[2] = _createMaverickSwapCall(collateralAsset, asset, collateralForRepayment, repayAmount , 0, false);
        flashloanCallbackBundle[3] = _createERC20TransferFromCall(asset, address(this), address(mysticAdapter), type(uint256).max);
        // Compressed main bundle creation
        mainBundle[0] = _createMysticFlashloanCall(asset,repayAmount,false, abi.encode(flashloanCallbackBundle));
        mainBundle[1] = _createERC20TransferCall(asset,address(maverickAdapter), type(uint256).max);
        mainBundle[2] = _createMaverickSwapCall(asset, collateralAsset, type(uint256).max, 0 , 0, false);
        mainBundle[3] = _createERC20TransferCall(collateralAsset,address(this), type(uint256).max);
        mainBundle[4] = _createERC20TransferFromCall(collateralAsset, address(this), msg.sender, type(uint256).max);
-        updatePositionTracking(pairKey, repayAmount, 0, msg.sender, false);
+        updatePositionTracking(pairKey, repayAmount, collateralForRepayment, msg.sender, false);
      } else {
          revert("No changes to position");
      }
      
      bundler.multicall(mainBundle);
      
      emit BundleCreated(msg.sender, keccak256("UPDATE_LEVERAGE"), mainBundle.length);
      emit LeverageUpdated(msg.sender, collateralAsset, asset, currentCollateral, currentLeverage, newTargetLeverage, totalCollaterals[pairKey], totalBorrows[pairKey]);
      
      return mainBundle;
  }
```

# [M-1]totalCollateralsPerUser is not updated when user calls `MysticLeverageBundler::updateLeverageBundle`.

## Metadata

- **Number:** #133
- **Severity:** High
- **Status:** Duplicate
- **Likelihood:** High
- **Impact:** High
- **Created by:** Bizarro
- **Created at:** May 15, 2025 at 4:00 PM
- **Last updated:** May 26, 2025 at 5:18 AM
- **Reward:** 41.77

## Description

## Summary
When user calls [MysticLeverageBundler::updateLeverageBundle](https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/bundler3/src/calls/MysticLeverageBundler.sol?lines=256,256) user's `totalCollateralPerUser` is not updated leading to wrong calculation of collateral amount and borrow amount.

## Finding Description
In [MysticLeverageBundler::updateLeverageBundle](https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/bundler3/src/calls/MysticLeverageBundler.sol?lines=256,256) user can increase and decrease their previous leverage to newTargetLeverage. If the newtargetLeverage is greater than previous leverage then the protocol flashloans assetToken from the mysticPool and supplies the collateralAsset to the mysticPool after swapping the assetToken for collateralAsset from maverick and at last borrows some amount from mystic pool to pay the flashLoan back to the pool.
the new supplied amount is the new collateral added on behalf of user after borrowing which should increase the `totalBorrowsPerUser` and `totalCollateralPerUser` but only `totalBorrowsPerUser` is updated, this could lead to wrong leverage calculation and withdraw amount when user decides to close their position.

The same can be seen when the user decides to decrease the newTargetLeverage.

```solidity
function updateLeverageBundle(
      address asset,
      address collateralAsset,
      uint256 newTargetLeverage,
      uint256 slippageTolerance
  ) external returns (Call[] memory bundle) {
      require(newTargetLeverage > SLIPPAGE_SCALE, "Leverage must be > 1");
      require(newTargetLeverage <= 1000000, "Leverage too high"); // Max 100x
      bytes32 pairKey = getPairKey(asset, collateralAsset);
      uint256 slippage = slippageTolerance == 0 ? DEFAULT_SLIPPAGE : slippageTolerance;
      uint256 currentCollateral = totalCollateralsPerUser[pairKey][msg.sender];
      uint256 currentBorrow = totalBorrowsPerUser[pairKey][msg.sender];

      require(currentCollateral > 0 && currentBorrow > 0, "No existing position");
      uint256 currentLeverage = (currentCollateral * SLIPPAGE_SCALE) / (currentCollateral - currentBorrow);
      uint256 newBorow =  currentBorrow * (newTargetLeverage - SLIPPAGE_SCALE) * currentLeverage / (newTargetLeverage * (currentLeverage - SLIPPAGE_SCALE)); //getQuote(collateralAsset, asset, currentCollateral * (newTargetLeverage - SLIPPAGE_SCALE) / newTargetLeverage);
      int256 borrowDelta = int256(newBorow) - int256(currentBorrow);
      
      // Create appropriate bundles based on the operation type
      Call[] memory mainBundle;
      Call[] memory flashloanCallbackBundle;
 
      if (borrowDelta > 0) {
          // INCREASE LEVERAGE CASE
        uint256 additionalBorrowAmount = uint256(borrowDelta);
        uint totalBorrowAmount = additionalBorrowAmount + mysticAdapter.flashLoanFee(additionalBorrowAmount) + 1;

        mainBundle = new Call[](1);
        flashloanCallbackBundle = new Call[](5);
        flashloanCallbackBundle[0] = _createERC20TransferCall(asset, address(maverickAdapter), type(uint256).max);
        flashloanCallbackBundle[1] = _createMaverickSwapCall(asset, collateralAsset, type(uint256).max, 0, slippage, false);
        flashloanCallbackBundle[2] = _createERC20TransferFromCall(collateralAsset, address(this), address(mysticAdapter), type(uint256).max);
        flashloanCallbackBundle[3] = _createMysticSupplyCall( collateralAsset, type(uint256).max, msg.sender);
        flashloanCallbackBundle[4] = _createMysticBorrowCall(asset, totalBorrowAmount, VARIABLE_RATE_MODE, msg.sender, address(mysticAdapter));

        mainBundle[0] = _createMysticFlashloanCall(asset,additionalBorrowAmount,false,abi.encode(flashloanCallbackBundle));
@>   updatePositionTracking(pairKey, additionalBorrowAmount, 0, msg.sender, true);
      } else if (borrowDelta < 0) {
        uint256 repayAmount = uint256(-borrowDelta);
        uint256 collateralForRepayment = (totalCollateralsPerUser[pairKey][msg.sender] * repayAmount) / totalBorrowsPerUser[pairKey][msg.sender];
          
        mainBundle = new Call[](5);
        flashloanCallbackBundle = new Call[](4);

        flashloanCallbackBundle[0] = _createMysticRepayCall(asset, repayAmount, VARIABLE_RATE_MODE, msg.sender);
        flashloanCallbackBundle[1] = _createMysticWithdrawCall(collateralAsset, collateralForRepayment, msg.sender, address(maverickAdapter));
        flashloanCallbackBundle[2] = _createMaverickSwapCall(collateralAsset, asset, collateralForRepayment, repayAmount , 0, false);
        flashloanCallbackBundle[3] = _createERC20TransferFromCall(asset, address(this), address(mysticAdapter), type(uint256).max);
        // Compressed main bundle creation
        mainBundle[0] = _createMysticFlashloanCall(asset,repayAmount,false, abi.encode(flashloanCallbackBundle));
        mainBundle[1] = _createERC20TransferCall(asset,address(maverickAdapter), type(uint256).max);
        mainBundle[2] = _createMaverickSwapCall(asset, collateralAsset, type(uint256).max, 0 , 0, false);
        mainBundle[3] = _createERC20TransferCall(collateralAsset,address(this), type(uint256).max);
        mainBundle[4] = _createERC20TransferFromCall(collateralAsset, address(this), msg.sender, type(uint256).max);
@>  updatePositionTracking(pairKey, repayAmount, 0, msg.sender, false);
      } else {
          revert("No changes to position");
      }
      
      bundler.multicall(mainBundle);
      
      emit BundleCreated(msg.sender, keccak256("UPDATE_LEVERAGE"), mainBundle.length);
      emit LeverageUpdated(msg.sender, collateralAsset, asset, currentCollateral, currentLeverage, newTargetLeverage, totalCollaterals[pairKey], totalBorrows[pairKey]);
      
      return mainBundle;
  }
```

## Impact Explanation
Impact is high because the main storage variables are storing wrong info which can lead to wrong leverage calculation and withdraw amount.

## Likelihood Explanation
Likelihood is high because the attacker can easily manipulate the values by calling updateLeverageBundle function

## Proof of Concept
copy this test in `MysticLeverageBundlerForkTest.sol` and run `forge test --fork-url https://phoenix-rpc.plumenetwork.xyz --mc MysticLeverageBundlerRWATest --mt test_update_leverage_isNot_updating_collateral -vvvv`

```solidity
    function test_update_leverage_isNot_updating_collateral() external {
        PositionData memory before = _createInitialPosition();

        PositionData memory userDataBefore = getPositionData(USER, address(borrowToken), address(collateralToken));

        // Update leverage: increase from current to higher
        vm.prank(USER);
        Call[] memory bundleCalls = leverageBundler.updateLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            LEVERAGE_3X, // Increase leverage
            DEFAULT_SLIPPAGE
        );

        PositionData memory userDataAfter = getPositionData(USER, address(borrowToken), address(collateralToken));

        assertEq(userDataAfter.totalCollateralUser-userDataBefore.totalCollateralUser, INITIAL_COLLATERAL);
    }
```
this will fail with error

```
 0x593cCcA4c4bf58b7526a4C164cEEf4003C6388db::balanceOf(0x18E1EEC9Fa5D77E472945FE0d48755386f28443c) [staticcall]
    │   └─ ← [Return] 99999900000 [9.999e10]
    ├─ [621] 0x593cCcA4c4bf58b7526a4C164cEEf4003C6388db::balanceOf(0x598Fc8cD4335D5916Fa81Ec0Efa25b462aA721F1) [staticcall]
    │   └─ ← [Return] 0
    ├─ [621] 0x593cCcA4c4bf58b7526a4C164cEEf4003C6388db::balanceOf(0xE2314ECb6Ae07a987018a71e412897ED2F54E075) [staticcall]
    │   └─ ← [Return] 0
    ├─ [0] VM::assertEq(0, 100000 [1e5]) [staticcall]
    │   └─ ← [Revert] assertion failed: 0 != 100000
    └─ ← [Revert] assertion failed: 0 != 100000
```

## Recommendation

```diff
function updateLeverageBundle(
      address asset,
      address collateralAsset,
      uint256 newTargetLeverage,
      uint256 slippageTolerance
  ) external returns (Call[] memory bundle) {
      require(newTargetLeverage > SLIPPAGE_SCALE, "Leverage must be > 1");
      require(newTargetLeverage <= 1000000, "Leverage too high"); // Max 100x
      bytes32 pairKey = getPairKey(asset, collateralAsset);
      uint256 slippage = slippageTolerance == 0 ? DEFAULT_SLIPPAGE : slippageTolerance;
      uint256 currentCollateral = totalCollateralsPerUser[pairKey][msg.sender];
      uint256 currentBorrow = totalBorrowsPerUser[pairKey][msg.sender];

      require(currentCollateral > 0 && currentBorrow > 0, "No existing position");
      uint256 currentLeverage = (currentCollateral * SLIPPAGE_SCALE) / (currentCollateral - currentBorrow);
      uint256 newBorow =  currentBorrow * (newTargetLeverage - SLIPPAGE_SCALE) * currentLeverage / (newTargetLeverage * (currentLeverage - SLIPPAGE_SCALE)); //getQuote(collateralAsset, asset, currentCollateral * (newTargetLeverage - SLIPPAGE_SCALE) / newTargetLeverage);
      int256 borrowDelta = int256(newBorow) - int256(currentBorrow);
      
      // Create appropriate bundles based on the operation type
      Call[] memory mainBundle;
      Call[] memory flashloanCallbackBundle;
 
      if (borrowDelta > 0) {
          // INCREASE LEVERAGE CASE
        uint256 additionalBorrowAmount = uint256(borrowDelta);
        uint totalBorrowAmount = additionalBorrowAmount + mysticAdapter.flashLoanFee(additionalBorrowAmount) + 1;

        mainBundle = new Call[](1);
        flashloanCallbackBundle = new Call[](5);
        flashloanCallbackBundle[0] = _createERC20TransferCall(asset, address(maverickAdapter), type(uint256).max);
        flashloanCallbackBundle[1] = _createMaverickSwapCall(asset, collateralAsset, type(uint256).max, 0, slippage, false);
        flashloanCallbackBundle[2] = _createERC20TransferFromCall(collateralAsset, address(this), address(mysticAdapter), type(uint256).max);
        flashloanCallbackBundle[3] = _createMysticSupplyCall( collateralAsset, type(uint256).max, msg.sender);
        flashloanCallbackBundle[4] = _createMysticBorrowCall(asset, totalBorrowAmount, VARIABLE_RATE_MODE, msg.sender, address(mysticAdapter));

        mainBundle[0] = _createMysticFlashloanCall(asset,additionalBorrowAmount,false,abi.encode(flashloanCallbackBundle));
-        updatePositionTracking(pairKey, additionalBorrowAmount, 0, msg.sender, true);
+       updatePositionTracking(pairKey, additionalBorrowAmount, additionalBorrowAmount, msg.sender, true);
      } else if (borrowDelta < 0) {
        uint256 repayAmount = uint256(-borrowDelta);
        uint256 collateralForRepayment = (totalCollateralsPerUser[pairKey][msg.sender] * repayAmount) / totalBorrowsPerUser[pairKey][msg.sender];
          
        mainBundle = new Call[](5);
        flashloanCallbackBundle = new Call[](4);

        flashloanCallbackBundle[0] = _createMysticRepayCall(asset, repayAmount, VARIABLE_RATE_MODE, msg.sender);
        flashloanCallbackBundle[1] = _createMysticWithdrawCall(collateralAsset, collateralForRepayment, msg.sender, address(maverickAdapter));
        flashloanCallbackBundle[2] = _createMaverickSwapCall(collateralAsset, asset, collateralForRepayment, repayAmount , 0, false);
        flashloanCallbackBundle[3] = _createERC20TransferFromCall(asset, address(this), address(mysticAdapter), type(uint256).max);
        // Compressed main bundle creation
        mainBundle[0] = _createMysticFlashloanCall(asset,repayAmount,false, abi.encode(flashloanCallbackBundle));
        mainBundle[1] = _createERC20TransferCall(asset,address(maverickAdapter), type(uint256).max);
        mainBundle[2] = _createMaverickSwapCall(asset, collateralAsset, type(uint256).max, 0 , 0, false);
        mainBundle[3] = _createERC20TransferCall(collateralAsset,address(this), type(uint256).max);
        mainBundle[4] = _createERC20TransferFromCall(collateralAsset, address(this), msg.sender, type(uint256).max);
-        updatePositionTracking(pairKey, repayAmount, 0, msg.sender, false);
+        updatePositionTracking(pairKey, repayAmount, collateralForRepayment, msg.sender, false);
      } else {
          revert("No changes to position");
      }
      
      bundler.multicall(mainBundle);
      
      emit BundleCreated(msg.sender, keccak256("UPDATE_LEVERAGE"), mainBundle.length);
      emit LeverageUpdated(msg.sender, collateralAsset, asset, currentCollateral, currentLeverage, newTargetLeverage, totalCollaterals[pairKey], totalBorrows[pairKey]);
      
      return mainBundle;
  }
```


# [M-2] User won't be able to deposit when validator's limit has almost reached.

## Metadata

- **Number:** #266
- **Severity:** Medium
- **Status:** Duplicate
- **Created by:** Bizarro
- **Created at:** May 17, 2025 at 12:09 AM
- **Last updated:** May 24, 2025 at 11:50 AM
- **Reward:** 2.18

## Description

## Summary
When a User call the submit function to stake their eth, the deposit will revert everytime the current validator's limit is almost reached.

## Finding Description
When a user calls the `frxETHMinter::submit` function to stake their ETH, the function first mints `frxETHToken` and then calls `stPlumeMinter::depositEther`.
In depositEther, the function attempts to stake ETH into the plumeStaking contract. If the current validator has reached its maximum capacity, the function looks for another validator to stake the remaining ETH.

The problem lies in the while loop, which continues as long as `remainingAmount > 0`. If the previous validator is nearly maxed out and the user tries to deposit exactly `plumeStaking.getMinStakeAmount()`, the first validator might accept only a portion of it. The leftover amount is then passed to the next validator. However, if this remaining amount is now less than minStakeAmount, the function will revert.


## Impact Explanation
Impact is High because the depositEther function will not work and is called in every function.

## Likelihood Explanation
Likelihood of this happening is medium because the current validator has to be almost maxed out for this to happen.

## Proof of Concept

first change the start index of the loop from 0 to 1 so that we can start staking on the 2nd validator.

https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/Liquid-Staking/src/stPlumeMinter.sol?lines=60,60

```diff
function getNextValidator(uint depositAmount) public returns (uint256 validatorId, uint256 capacity) {
        // Make sure there are free validators available
        uint numVals = numValidators();
        require(numVals != 0, "Validator stack is empty");

-      for (uint256 i = 0; i < numVals; i++) {
+      for (uint256 i = 1; i < numVals; i++) {
            ---
        }

        revert("No validator with sufficient capacity");
    }
```

add validator id 1 and 2.

https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/Liquid-Staking/test/fork/stPlumeMinter.t.sol?lines=69,69

```diff
        validators[1] = OperatorRegistry.Validator(2);
        validators[2] = OperatorRegistry.Validator(3);
```

paste this test in `stPlumeMinter.t.sol` and run `forge test --mc StPlumeMinterForkTest --fork-url https://rpc.plume.org --mt test_depositEther_fail -vvv` 

```solidity
function test_depositEther_fail() public {
        // add 2 new validators in mockPlumeStaking contract.
        uint16 newValidatorId = 2;
        uint256 commission = 5e16;
        address newAdminForVal2 = makeAddr("newAdminForVal2");
        address l2Withdraw = newAdminForVal2; // Often the same, but can be different
        string memory l1ValAddr = "0xval3";
        string memory l1AccAddr = "0xacc3";
        address l1AccEvmAddr = address(0x1234);
        uint256 maxCapacity = 1_000_000e18;
        vm.prank(address(0xC0A7a3AD0e5A53cEF42AB622381D0b27969c4ab5));
         mockPlumeStaking.addValidator(
            newValidatorId,
            commission,
            newAdminForVal2, // Use the new admin address
            l2Withdraw,
            l1ValAddr,
            l1AccAddr,
            l1AccEvmAddr,
            maxCapacity
        );
        vm.prank(address(0xC0A7a3AD0e5A53cEF42AB622381D0b27969c4ab5));
        mockPlumeStaking.addValidator(
            3,
            commission,
            newAdminForVal2, // Use the new admin address
            l2Withdraw,
            l1ValAddr,
            l1AccAddr,
            l1AccEvmAddr,
            maxCapacity
        );

        uint256 depositAmount = maxCapacity - 100;
        vm.deal(address(this), depositAmount);
        // stake maxCapacity - 10 directly to PlumeStaking contract.
        mockPlumeStaking.stake{value: depositAmount}(uint16(2));

        // Submit ETH
        vm.deal(user1, 0.1 ether);
        vm.prank(user1);
        
        minter.submit{value: 0.1 ether}();
    }
```

This test will revert because of this error [here](https://github.com/plumenetwork/contracts/blob/18738402d7e00dbdf0c831800c2d484ef3edd2e6/plume/src/facets/StakingFacet.sol#L86)

```
1335] 0xA20bfe49969D4a0E9abfdb6a46FeD777304ba07f::stake{value: 100}(2)
    │   │   ├─ [447] 0x76eF355e6DdB834640a6924957D5B1d87b639375::stake{value: 100}(2) [delegatecall]
    │   │   │   └─ ← [Revert] custom error 0x7f5b0618: 0000000000000000000000000000000000000000000000000000000000000064000000000000000000000000000000000000000000000000016345785d8a0000
    │   │   └─ ← [Revert] custom error 0x7f5b0618: 0000000000000000000000000000000000000000000000000000000000000064000000000000000000000000000000000000000000000000016345785d8a0000
    │   └─ ← [Revert] custom error 0x7f5b0618: 0000000000000000000000000000000000000000000000000000000000000064000000000000000000000000000000000000000000000000016345785d8a0000
    └─ ← [Revert] custom error 0x7f5b0618: 0000000000000000000000000000000000000000000000000000000000000064000000000000000000000000000000000000000000000000016345785d8a0000
```

## Recommendation
Add the `getMinStakeAmount` check inside the loop before adding staking eth.

# [M-3] Front runner can attack the swapping between collateral token and asset token

## Metadata

- **Number:** #179
- **Severity:** Medium
- **Status:** Duplicate
- **Created by:** Bizarro
- **Created at:** May 15, 2025 at 11:13 PM
- **Last updated:** May 31, 2025 at 3:58 PM
- **Reward:** 12.52

## Description

## Summary
A front runner can easily attack the maverickSwapCall present in the MysticLeverageBundler leading to protocol loosing funds.

## Finding Description
Whenever _createMaverickSwapCall is called by the contract with amountIn as `type(uint256).max` and `slippage` as 0, a front runner can easily attack the transaction by first increasing the price of the token to be swapped and then after execution of the swap reset the price to normal.

In [`MaverickAdapter::swapExactTokensForTokens`](https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/bundler3/src/adapters/MaverickAdapter.sol?lines=29,29) function implements a case where if amountIn is `type(uint256).max` then amountOutMin will be `amountIn * slippage / SLIPPAGE_SCALE`

```solidity
function swapExactTokensForTokens(
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 amountOutMin,
        uint256 slippage,
        address to,
        int32 tickRange
    ) external returns (uint256 amountOut) {
        // Get the best pool for this pair
        IMaverickV2Pool pool = getBestPool(tokenIn, tokenOut);
        uint256 balance = IERC20(tokenIn).balanceOf(address(this));

        if (amountIn == type(uint256).max || amountIn > balance) {
            amountIn = balance;
@>            amountOutMin = amountIn * slippage / SLIPPAGE_SCALE;
        }
        
        IERC20(tokenIn).safeTransfer(address(pool), amountIn);
        bool tokenAIn = address(pool.tokenA()) == tokenIn;
        int32 tickLimit = tokenAIn ? type(int32).max : type(int32).min;
        
        IMaverickV2Pool.SwapParams memory swapParams = IMaverickV2Pool.SwapParams({
            amount: amountIn,
            tokenAIn: tokenAIn,
            exactOutput: false,
            tickLimit: tickLimit
        });
        
        (, uint256 amountOutReceived) = pool.swap{gas: 800_000}(to, swapParams, "");
        require(amountOutReceived >= amountOutMin, "Insufficient output amount");
        return amountOutReceived;
    }

```
So if `slippage` is 0 then `amountOutMin` will be zero and the attacker can easily frontrun the swap.

there are multiple places where we can see that one of the two are 0.


1. https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/bundler3/src/calls/MysticLeverageBundler.sol?lines=208,208
2. https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/bundler3/src/calls/MysticLeverageBundler.sol?lines=235,235
3. https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/bundler3/src/calls/MysticLeverageBundler.sol?lines=307,307

## Impact Explanation
Impact is High because the contract will loose a significant amount of portion from the swap

## Likelihood Explanation
Likelihood is medium because attacker has to take a flashLoan for such a big price manipulation.

## Proof of Concept
1. protocol calls the _createMaverickSwapCall with amountIn as `type(uint256).max`, and slippage as 0.
2. the function calls the swapExactTokensForTokens function
3. hence amountIn is `type(uint256).max` and slippage is 0, the amountOutMin would be 0.
4. function calls the pool to swap.
5. attacker sees the transaction in the mempool
6. with more gas the attacker increase the price of the input token.
7. user gets worst price, as amountOutReceived is always greater than 0, the tx gets execeuted.
8. attacker decreases the price to the normal price.

Add this line here [MaverickAdapter::swapExactTokensForTokens](https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/bundler3/src/adapters/MaverickAdapter.sol?lines=44,44)

```diff
function swapExactTokensForTokens(
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 amountOutMin,
        uint256 slippage,
        address to,
        int32 tickRange
    ) external returns (uint256 amountOut) {
        // Get the best pool for this pair
        IMaverickV2Pool pool = getBestPool(tokenIn, tokenOut);
        uint256 balance = IERC20(tokenIn).balanceOf(address(this));

        if (amountIn == type(uint256).max || amountIn > balance) {
            amountIn = balance;
            amountOutMin = amountIn * slippage / SLIPPAGE_SCALE;
+           require(amountOutMin > 0, "AmountOutMin is 0"); 
        }
        
        IERC20(tokenIn).safeTransfer(address(pool), amountIn);
        bool tokenAIn = address(pool.tokenA()) == tokenIn;
        int32 tickLimit = tokenAIn ? type(int32).max : type(int32).min;
        
        IMaverickV2Pool.SwapParams memory swapParams = IMaverickV2Pool.SwapParams({
            amount: amountIn,
            tokenAIn: tokenAIn,
            exactOutput: false,
            tickLimit: tickLimit
        });
        
        (, uint256 amountOutReceived) = pool.swap{gas: 800_000}(to, swapParams, "");
        require(amountOutReceived >= amountOutMin, "Insufficient output amount");
        return amountOutReceived;
    }

```

Paste this test to `AaveLeverageBundlerTest.t.sol` and run `forge test --fork-url https://phoenix-rpc.plumenetwork.xyz --mc MysticLeverageBundlerRWATest --mt test_revert_when_amountOutMin_is_zero -vvv`

```solidity
function test_revert_when_amountOutMin_is_zero() external {
        IBundler3 bundler2 = new Bundler3();
        MaverickSwapAdapter maverickAdapterMock2 = new MaverickSwapAdapter(
            maverickFactoryMock, maverickQuoterMock
        );
        MysticAdapter mysticAdapter = new MysticAdapter(
            address(bundler2), address(0xCE192A6E105cD8dd97b8Dedc5B5b263B52bb6AE0), address(0xEa237441c92CAe6FC17Caaf9a7acB3f953be4bd1)
        );
        MysticLeverageBundler leverageBundler2 = new MysticLeverageBundler(
            address(bundler2), address(mysticAdapter), address(maverickAdapterMock2)
        );
        vm.startPrank(USER);
        collateralToken.approve(address(mysticAdapter), type(uint256).max);
        ERC20Mock(0x9fbC367B9Bb966a2A537989817A088AFCaFFDC4c).approve(address(mysticAdapter), type(uint256).max);
        borrowToken.approve(address(mysticAdapter), type(uint256).max);
        ICreditDelegationToken(0xA9b705D4719002030386fa83087c905Ca4c25eB2).approveDelegation(
            address(mysticAdapter), type(uint128).max
        );
        IERC20(0xAf5aEAb2248415716569Be5d24FbE10b16590D6c).approve(address(mysticAdapter), type(uint128).max);
        IERC20(0xDb224c353CFB74e220b7B6cB12f8D8Bc7c8B2863).approve(address(mysticAdapter), type(uint128).max);
        IERC20(0x593cCcA4c4bf58b7526a4C164cEEf4003C6388db).approve(
            address(0xCE192A6E105cD8dd97b8Dedc5B5b263B52bb6AE0), type(uint128).max
        );
        ICreditDelegationToken(0xA9b705D4719002030386fa83087c905Ca4c25eB2).approveDelegation(
            address(mysticAdapter), type(uint128).max
        );
        IERC20(0xAf5aEAb2248415716569Be5d24FbE10b16590D6c).approve(address(mysticAdapter), type(uint128).max);
        IERC20(0xd1a7183708EF9706F3dD2d51B27a7e02a70F30fa).approve(address(mysticAdapter), type(uint128).max);
        IERC20(0x593cCcA4c4bf58b7526a4C164cEEf4003C6388db).approve(
            address(0xCE192A6E105cD8dd97b8Dedc5B5b263B52bb6AE0), type(uint128).max
        );
        vm.stopPrank();



        // First open a position
        PositionData memory before = getPositionData(USER, address(borrowToken), address(collateralToken));

        vm.prank(USER);

        leverageBundler2.createOpenLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            address(collateralToken),
            INITIAL_COLLATERAL,
            LEVERAGE_2X,
            DEFAULT_SLIPPAGE
        );

        vm.prank(USER);
        leverageBundler2.createCloseLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            type(uint256).max
        );
    }
```
This test will revert with
```
0xdddD73F5Df1F0DC31373357beAC77545dC5A6f3F::balanceOf(MaverickSwapAdapter: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   └─ ← [Return] 101899 [1.018e5]
    │   │   │   ├─ [0] console::log("amountOutMin", 0) [staticcall]
    │   │   │   │   └─ ← [Stop]
    │   │   │   └─ ← [Revert] AmountOutMin is 0
    │   │   └─ ← [Revert] AmountOutMin is 0
    │   └─ ← [Revert] AmountOutMin is 0
    └─ ← [Revert] AmountOutMin is 0
```

## Recommendation
pass correct slippage to the swap params before calling the swap call.


# [M-4] `MysticLeverageBundler::_createOpenLeverageBundleWithLoops`  has an incorrect calculation when computing newBorrow.

## Metadata

- **Number:** #165
- **Severity:** Medium
- **Status:** Duplicate
- **Likelihood:** High
- **Impact:** Low
- **Created by:** Bizarro
- **Created at:** May 15, 2025 at 9:23 PM
- **Last updated:** May 24, 2025 at 4:35 PM
- **Reward:** 58.96

## Description

## Summary
In `MysticLeverageBundler::_createOpenLeverageBundleWithLoops` the calculation for newCollateral is wrong which will later decrease the `totalBorrowsPerUser` and will wrong leverage.

## Finding Description
The [MysticLeverageBundler::_createOpenLeverageBundleWithLoops](https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/bundler3/src/calls/MysticLeverageBundler.sol?lines=135,135) function is called when there is not enough liquidity in the mysticPool, this function first transfers collateralToken from user to mysticAdapter and supplies the collateralToken to the mysticPool.
Then a loop is executed where first the function borrows from mysticPool and then swap collateral token for assetToken which will be supplied to the mysticPool and this goes on until 90% of targetLeverage is achieved.

The problem lies in the calculation of newBorrow amount because we are decreasing the newBorrow depending on ltv and newCollateral and newCollateral has already been decreased based on the ltv.

This can cause wrong calculation of leverage and user can get a position with high leverage and contract will show low leverage.

## Impact Explanation
Impact is high because the leverage will be wrongly calculated and `totalBorrowsPerUser` will be wrong.

## Likelihood Explanation
Likelihood of this happening is high because this will happen everytime `_createOpenLeverageBundleWithLoops` is called.

## Proof of Concept
1. Assume user wants to openLeverage with `InitialCollateralAmount` of 100e18 and `targetLeverage` of `1.75X` and the ltv for the `collateralToken` is 7500.
2. user transfers 100e18 collateral token to the mystic contract which will be supplied to the mystic contract (here, `totalCollateralsPerUser` = 100e18)
3. loop starts
4. mystic borrows asset token from mysticPool which will be 75e18 (75% of the deposited amount if the token is eth)
5. maverick swaps the assetToken for collateralToken.
6. mystic supplies 75e18 asset token to the mysticPool.
7. newCollateral = 100e18 * 7500 / 10000 = 75e18.
8. newBorrow(wrong) = 75e18 * 7500 / 10000 = 56.25e18.
9. updatePositionTracking function is called which will increase the `totalCollateralsPerUser` by newCollateral and `totalBorrowsPerUser` by newBorrow.
10. now, `totalCollateralsPerUser` is 175e18 and `totalBorrowsPerUser` is 56.25e18.
11. so the leverage will be 175e18 * 10000 / (175e18 - 56.25e18) = 14736 (1.4736X)
12. as the calculated leverage is not greater than 90% of targetLeverage(15750) the loop will run again.

The actual should calculation should be 

8. newBorrow = newCollateral.
9. updatePositionTracking function is called which will increase the `totalCollateralsPerUser` by newCollateral and `totalBorrowsPerUser` by newBorrow.
10. now, `totalCollateralsPerUser` is 175e18 and `totalBorrowsPerUser` is 75e18.
11. so the leverage will be 175e18 * 10000 / (175e18 - 75e18) = 17500 (1.75X)
12. as the calculated leverage is greater than 90% of targetLeverage(15750) the loop will break.

## Coded POC

Update this line [here](https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/bundler3/src/calls/MysticLeverageBundler.sol?lines=89,89)

```diff
function createOpenLeverageBundle(address asset, address collateralAsset, address inputAsset, uint256 initialCollateralAmount, uint256 targetLeverage, uint256 slippageTolerance) external returns (Call[] memory bundle) {
      require(initialCollateralAmount > 0, "Zero collateral amount");
      require(targetLeverage > SLIPPAGE_SCALE, "Leverage must be > 1");
      require(targetLeverage <= 1000000, "Leverage too high");
      
      uint256 positionSize = initialCollateralAmount * targetLeverage / SLIPPAGE_SCALE;

      IERC20(collateralAsset).approve(address(mysticAdapter), type(uint256).max);
      IERC20(asset).approve(address(mysticAdapter), type(uint256).max);
      
      // Check if there's enough liquidity for flashloan
-      if (mysticAdapter.getAvailableLiquidity(asset) > positionSize) {
+      if (mysticAdapter.getAvailableLiquidity(asset) < positionSize) {
          return _createOpenLeverageBundleWithFlashloan(asset, collateralAsset, inputAsset, initialCollateralAmount, targetLeverage, slippageTolerance);
      } else { // loop can still accomodate smaller leverages even with insufficient liqudiity in a pool, there will be a warning in the frontend though
          return _createOpenLeverageBundleWithLoops(asset, collateralAsset, inputAsset, initialCollateralAmount, targetLeverage, slippageTolerance);
      }
    }
``` 

Paste this test in `AaveLeverageBundlerTest.sol` and run `forge test --fork-url https://phoenix-rpc.plumenetwork.xyz --mc MysticLeverageBundlerRWATest --mt test_open_leverage_is_wrong -vvv`

```solidity
function test_open_leverage_is_wrong() public {
        IBundler3 bundler2 = new Bundler3();
        MaverickSwapAdapter maverickAdapterMock2 = new MaverickSwapAdapter(
            maverickFactoryMock, maverickQuoterMock
        );
        MysticAdapter mysticAdapter = new MysticAdapter(
            address(bundler2), address(0xCE192A6E105cD8dd97b8Dedc5B5b263B52bb6AE0), address(0xEa237441c92CAe6FC17Caaf9a7acB3f953be4bd1)
        );
        MysticLeverageBundler leverageBundler2 = new MysticLeverageBundler(
            address(bundler2), address(mysticAdapter), address(maverickAdapterMock2)
        );
        vm.startPrank(USER);
        collateralToken.approve(address(mysticAdapter), type(uint256).max);
        ERC20Mock(0x9fbC367B9Bb966a2A537989817A088AFCaFFDC4c).approve(address(mysticAdapter), type(uint256).max);
        borrowToken.approve(address(mysticAdapter), type(uint256).max);
        ICreditDelegationToken(0xA9b705D4719002030386fa83087c905Ca4c25eB2).approveDelegation(
            address(mysticAdapter), type(uint128).max
        );
        IERC20(0xAf5aEAb2248415716569Be5d24FbE10b16590D6c).approve(address(mysticAdapter), type(uint128).max);
        IERC20(0xDb224c353CFB74e220b7B6cB12f8D8Bc7c8B2863).approve(address(mysticAdapter), type(uint128).max);
        IERC20(0x593cCcA4c4bf58b7526a4C164cEEf4003C6388db).approve(
            address(0xCE192A6E105cD8dd97b8Dedc5B5b263B52bb6AE0), type(uint128).max
        );
        ICreditDelegationToken(0xA9b705D4719002030386fa83087c905Ca4c25eB2).approveDelegation(
            address(mysticAdapter), type(uint128).max
        );
        IERC20(0xAf5aEAb2248415716569Be5d24FbE10b16590D6c).approve(address(mysticAdapter), type(uint128).max);
        IERC20(0xd1a7183708EF9706F3dD2d51B27a7e02a70F30fa).approve(address(mysticAdapter), type(uint128).max);
        IERC20(0x593cCcA4c4bf58b7526a4C164cEEf4003C6388db).approve(
            address(0xCE192A6E105cD8dd97b8Dedc5B5b263B52bb6AE0), type(uint128).max
        );
        vm.stopPrank();

        // create Open leverage
        vm.prank(USER);
        leverageBundler2.createOpenLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            address(collateralToken),
            INITIAL_COLLATERAL,
            20000,
            DEFAULT_SLIPPAGE
        );

        bytes32 pairKey = leverageBundler2.getPairKey(address(borrowToken), address(collateralToken));
        uint256 useBorrows = leverageBundler2.totalBorrowsPerUser(
            pairKey, USER
        );
        // this will revert.
        assertEq(INITIAL_COLLATERAL, useBorrows);
    }
```

The function will revert.

## Recommendation
```diff
function _createOpenLeverageBundleWithLoops(address asset, address collateralAsset, address inputAsset,  uint256 initialCollateralAmount, uint256 targetLeverage, uint256 slippageTolerance) internal returns (Call[] memory bundle) {
        // very expensive gas wise, with a limit of 25 loops(4x leverage), and extremely ineffective, only to be used if pool cannot fulfill flashloan
        // we understand that iterations != leverage but fo the sake of limiting gas, we assume iteration == loop instead of 1-ltv**(n+1)/1-ltv, where n is iteration
        require(inputAsset == collateralAsset, "Input asset must be the same as collateral asset");
        uint256 slippage = slippageTolerance == 0 ? DEFAULT_SLIPPAGE : slippageTolerance;
        uint256 ltv = mysticAdapter.getAssetLtv(collateralAsset);
        uint8 loop = 20;
        bytes32 pairKey = getPairKey(asset, collateralAsset);
        require(ltv > 0, "Collateral asset has no LTV");
        
        Call[] memory mainBundle = new Call[](2+ loop*4);
        mainBundle[0] = _createERC20TransferFromCall(inputAsset,msg.sender,address(mysticAdapter),initialCollateralAmount);
        mainBundle[1] = _createMysticSupplyCall( collateralAsset, type(uint256).max, msg.sender);
        uint256 newCollateral = initialCollateralAmount;
        totalCollaterals[pairKey] += newCollateral;
        totalCollateralsPerUser[pairKey][msg.sender] += newCollateral;

        for (uint8 i=0; i< loop; i++){
          uint256 idx = 2 + i * 4;
          mainBundle[idx] = _createMysticBorrowCall(asset, type(uint256).max, VARIABLE_RATE_MODE, msg.sender, address(maverickAdapter));
          mainBundle[idx+1] = _createMaverickSwapCall(asset, collateralAsset, type(uint256).max, 0, slippage, false);
          mainBundle[idx+2] = _createERC20TransferFromCall(collateralAsset, address(this), address(mysticAdapter), type(uint256).max);
          mainBundle[idx+3] = _createMysticSupplyCall( collateralAsset, type(uint256).max, msg.sender);

          newCollateral = newCollateral * ltv / SLIPPAGE_SCALE;
-          uint256 newBorrow = getQuote(collateralAsset, asset, newCollateral) * ltv / SLIPPAGE_SCALE;
-          updatePositionTracking(pairKey, newBorrow, newCollateral, msg.sender, true);
+         updatePositionTracking(pairKey, newCollateral, newCollateral, msg.sender, true);
          
          uint leverage = (totalCollateralsPerUser[pairKey][msg.sender]) * SLIPPAGE_SCALE / (totalCollateralsPerUser[pairKey][msg.sender] - totalBorrowsPerUser[pairKey][msg.sender]);
          if (leverage >= (targetLeverage * 9000) / SLIPPAGE_SCALE) break; // break if leverage gotten is in similar range as expected 10% error margin 4 -> 3.6 is fine
        }
        require(totalBorrowsPerUser[pairKey][msg.sender] < totalCollateralsPerUser[pairKey][msg.sender], "Leverage too high");

        bundler.multicall(mainBundle);

        emit BundleCreated(msg.sender, keccak256("OPEN_LEVERAGE"), mainBundle.length);
        emit LeverageOpened(msg.sender, collateralAsset, asset, initialCollateralAmount, targetLeverage, totalCollaterals[pairKey], totalBorrows[pairKey]);
        return mainBundle;
    }
```
# [M-5] Wrong calculation of newBorrow amount in `MysticLeverageBundler::updateLeverageBundle`

## Metadata

- **Number:** #131
- **Severity:** Medium
- **Status:** Duplicate
- **Likelihood:** High
- **Impact:** Low
- **Created by:** Bizarro
- **Created at:** May 15, 2025 at 3:35 PM
- **Last updated:** May 31, 2025 at 4:03 PM
- **Reward:** 58.96

## Description

## Summary
The calculation in `MysticLeverageBundler::updateLeverageBundle` can lead to wrong allocation of user borrows and collateral amount.

## Finding Description
When a user wants to update the leverage of their position through [updateLeverageBundle](https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/bundler3/src/calls/MysticLeverageBundler.sol?lines=256,256) they have to pass newTargetLeverage if the old leverage is 30,000( 2x leverage) and user wants to set newLeverage as 40,000( 3x leverage ). 

 let us assume, totalCollateralPerUser is 200e18 and totalBorrowsPerUser is 100e18 representing 2x leverage. 

now the user wants to update the leverage from 2x to 3x then the newBorrow amount should be 200e18, making the totalCollateralPerUser 300e18 and totalBorrowsPerUser 200e18.

But according to the calculation of [newBorrow](https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/bundler3/src/calls/MysticLeverageBundler.sol?lines=271,271)

newBorrow = currentBorrow * (newTargetLeverage - SLIPPAGE_SCALE) * currentLeverage / (newTargetLeverage * (currentLeverage - SLIPPAGE_SCALE));

100e18 * (40000 - 10000) * 30000 / (40000 * (30000 - 10000)) == 100e18 * 9 / 8

newBorrow = 112.5e18.

```solidity
function updateLeverageBundle(
      address asset,
      address collateralAsset,
      uint256 newTargetLeverage,
      uint256 slippageTolerance
  ) external returns (Call[] memory bundle) {
      require(newTargetLeverage > SLIPPAGE_SCALE, "Leverage must be > 1");
      require(newTargetLeverage <= 1000000, "Leverage too high"); // Max 100x
      bytes32 pairKey = getPairKey(asset, collateralAsset);
      uint256 slippage = slippageTolerance == 0 ? DEFAULT_SLIPPAGE : slippageTolerance;
      uint256 currentCollateral = totalCollateralsPerUser[pairKey][msg.sender];
      uint256 currentBorrow = totalBorrowsPerUser[pairKey][msg.sender];

      require(currentCollateral > 0 && currentBorrow > 0, "No existing position");
      uint256 currentLeverage = (currentCollateral * SLIPPAGE_SCALE) / (currentCollateral - currentBorrow);
@>      uint256 newBorow =  currentBorrow * (newTargetLeverage - SLIPPAGE_SCALE) * currentLeverage / (newTargetLeverage * (currentLeverage - SLIPPAGE_SCALE)); //getQuote(collateralAsset, asset, currentCollateral * (newTargetLeverage - SLIPPAGE_SCALE) / newTargetLeverage);
      int256 borrowDelta = int256(newBorow) - int256(currentBorrow);
      
      // Create appropriate bundles based on the operation type
      Call[] memory mainBundle;
      Call[] memory flashloanCallbackBundle;
 
      if (borrowDelta > 0) {
            ---
           }
  }
```
## Impact Explanation
Impact is high because everytime user updates the leverage, the allocated user borrows and collateral amount will be wrong.

## Likelihood Explanation
Likelihood is High.

## Proof of Concept
To prove the second function copy this test in `MysticLeverageBundlerForkTest.sol` and run `forge test --fork-url https://phoenix-rpc.plumenetwork.xyz --mc MysticLeverageBundlerRWATest --mt test_leverage_amount_inUpdate_isWrong -vvvv`

```solidity
function test_leverage_amount_inUpdate_isWrong() external {
        // let's increase the leverage from 2x to 3x,
        vm.prank(USER);
        leverageBundler.createOpenLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            address(collateralToken),
            INITIAL_COLLATERAL,
            20000,
            DEFAULT_SLIPPAGE
        );

        PositionData memory userDataBefore = getPositionData(USER, address(borrowToken), address(collateralToken));

        // Update leverage: increase from current to higher
        vm.prank(USER);
        Call[] memory bundleCalls = leverageBundler.updateLeverageBundle(
            address(borrowToken),
            address(collateralToken),
            30000, // Increase leverage
            DEFAULT_SLIPPAGE
        );

        PositionData memory userDataAfter = getPositionData(USER, address(borrowToken), address(collateralToken));
        // if we are increasing the leverage from 2x to 3x then the totalBorrowsuser must be updated from initialCollateral to 2 * initialCollateral.
        assertNotEq(userDataAfter.totalBorrowsUser, 2 * INITIAL_COLLATERAL);
    }
```
## Recommendation
Update the newBorrow calculation to calculate the right amount.
