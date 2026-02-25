# Infini-fi — Bug Reports
https://cantina.xyz/code/2ac7f906-1661-47eb-bfd6-519f5db0d36b/overview
---

# [H-1] Wrong Reward Slashing in `UnwindingModule::startUnwinding` causing loss of protocol funds.

## Metadata

- **Number:** #308
- **Severity:** High
- **Status:** Confirmed
- **Likelihood:** High
- **Impact:** Medium
- **Created by:** Bizarro
- **Created at:** April 24, 2025 at 10:36 PM
- **Last updated:** May 13, 2025 at 8:52 PM
- **Reward:** 1782.31

## Description

## Summary
In `UnwindingModule::startUnwinding` when calculating rewardWeight the `slashIndex` is divided rather than multiplying causing the user to get more asset than what they should get.

## Finding Description
In the [`UnwindingModule::startUnwinding`](https://cantina.xyz/code/2ac7f906-1661-47eb-bfd6-519f5db0d36b/src/locking/UnwindingModule.sol?lines=160,160) function, a user starts the process of withdrawing their position, and at that point, the contract calculates something called rewardWeight. This basically determines how many rewards the user will get when they fully withdraw.

Now, to make sure rewards are fair—especially when the farm has taken a loss—the system uses a `slashIndex`. This [`slashIndex`](https://cantina.xyz/code/2ac7f906-1661-47eb-bfd6-519f5db0d36b/src/locking/UnwindingModule.sol?lines=62,62) starts at `1e18` (which represents 100%) and goes down every time the [`applyLosses`](https://cantina.xyz/code/2ac7f906-1661-47eb-bfd6-519f5db0d36b/src/locking/UnwindingModule.sol?lines=326,326) function is called. That way, losses are spread fairly across all users.

But here’s the problem: when calculating `rewardWeight` and `targetRewardWeight`, the code divides by the `slashIndex` instead of multiplying by it. Since the `slashIndex` always stays between 0 and 1e18, dividing by it actually increases the reward instead of reducing it. That means users could end up claiming more rewards than they’re supposed to, especially after the farm has suffered losses.

So, instead of slashing rewards like it’s meant to, the current setup accidentally inflates them—potentially leading to unfair payouts and draining of the reward pool.


```solidity
function startUnwinding(address _user, uint256 _receiptTokens, uint32 _unwindingEpochs, uint256 _rewardWeight)
        external
        onlyCoreRole(CoreRoles.LOCKED_TOKEN_MANAGER)
    {

      ....

@>        uint256 rewardWeight = _rewardWeight.divWadDown(slashIndex);

@>        uint256 targetRewardWeight = _receiptTokens.divWadDown(slashIndex);
          uint256 totalDecrease = rewardWeight - targetRewardWeight;


      ....
    }
```

```solidity
    function applyLosses(uint256 _amount) external onlyCoreRole(CoreRoles.LOCKED_TOKEN_MANAGER) {
        if (_amount == 0) return;
        uint256 _totalReceiptTokens = totalReceiptTokens;
        ERC20Burnable(receiptToken).burn(_amount);
@>      slashIndex = slashIndex.mulDivDown(_totalReceiptTokens - _amount, _totalReceiptTokens);
        totalReceiptTokens = _totalReceiptTokens - _amount;
    }
```

## Impact Explanation
Impact is High as there is direct loss of funds due to this vulnerability.

## Likelihood Explanation
Likelihood of this happening is also High because every time a user tries to unwind their position and the `slashIndex` is low then 

## Proof of Concept
Add this line `assert(rewardWeight <= _rewardWeight);` to the `startUnwinding` function.

```solidity
function startUnwinding(address _user, uint256 _receiptTokens, uint32 _unwindingEpochs, uint256 _rewardWeight)
        external
        onlyCoreRole(CoreRoles.LOCKED_TOKEN_MANAGER)
    {
        bytes32 id = _unwindingId(_user, block.timestamp);
        require(positions[id].fromEpoch == 0, UserUnwindingInprogress());

        console.log("_rewardWeight: ", _rewardWeight);
        uint256 rewardWeight = _rewardWeight.divWadDown(slashIndex);
        console.log("rewardWeight after slashing: ", rewardWeight);
@>      assert(rewardWeight <= _rewardWeight);

        uint256 targetRewardWeight = _receiptTokens.divWadDown(slashIndex);

        ....
    }
```

Paste this test in `UnwindingModule.t.sol` and run `forge test --match-test testCheckWrongSlashCalculation -vvvv`, this will revert with `panic: assertion failed (0x01)`

```solidity
    function testCheckWrongSlashCalculation() public {
        // create position for alice
        _createPosition(alice, 1000, 10);
        _createPosition(bob, 2000, 5);


        // 2. Start Unwinding for alice
        vm.startPrank(alice);
        {
            MockERC20(lockingController.shareToken(10)).approve(address(gateway), 1000);
            gateway.startUnwinding(1000, 10);
        }
        vm.stopPrank();
        uint256 startUnwindingTimestamp = block.timestamp;

        advanceEpoch(6); 
        _depositRewards(330);

        // 3. apply losses which decreases in slashIndex.
        _applyLosses(3330 / 2);
        uint256 slashingIdx = unwindingModule.slashIndex();
        console.log(slashingIdx);

        advanceEpoch(1);

        // 4. create another position for alice
        _createPosition(alice, 1000, 10);
        
        // 5. again start unwinding for alice which will revert because the calculation is wrong.
        vm.startPrank(alice);
        {
            MockERC20(lockingController.shareToken(10)).approve(address(gateway), 1000);
            gateway.startUnwinding(1000, 10);
        }
        vm.stopPrank();
    }
```


## Recommendation
Update the calculation of `rewardWeight` and `targetRewardWeight` to multiply the slashIndex rather than dividing it.

```diff
function startUnwinding(address _user, uint256 _receiptTokens, uint32 _unwindingEpochs, uint256 _rewardWeight)
        external
        onlyCoreRole(CoreRoles.LOCKED_TOKEN_MANAGER)
    {
        bytes32 id = _unwindingId(_user, block.timestamp);
        require(positions[id].fromEpoch == 0, UserUnwindingInprogress());

-       uint256 rewardWeight = _rewardWeight.divWadDown(slashIndex);
+       uint256 rewardWeight = _rewardWeight.mulWadDown(slashIndex);
-       uint256 targetRewardWeight = _receiptTokens.divWadDown(slashIndex);
+       uint256 targetRewardWeight = _receiptTokens.mulWadDown(slashIndex);

        ....
    }
```

## Comments

---

**Nikola** - May 6, 2025 at 2:01 AM

needs further investigation on our end

**Replies:**

  ---

  **kamensec** - May 11, 2025 at 2:24 PM

  @project theres sometimes a reason for this which is why I asked in the duplicate issue. If we want to cancel out the slashing that occured prior to some action we would typically divide to cancel it out. If thats not intentional I believe this is valid, lmk otherwise!

  ---

  **Erwan Beauvois** - May 13, 2025 at 8:52 PM

  This might be counter intuitive but the division is the correct operation to do here. For instance if Alice & Bob both have 1000 tokens, and the following events occur :
  - Alice start to unwind
  - A slashing of 50% occurs (slashIndex = 1.0 -> 0.5)
  - Bob starts to unwind
  
  Bob's reward weight should be 2x that of Alice, even though they stake the same amount, because Alice has been slashed by half
  
  We can't loop through all slashed positions to divide their weight so we keep it the same in storage, and when Bob joins, we set his weight to 1000 / 0.5 = 2000

  ---

  **Bizarro** - May 13, 2025 at 9:53 PM

  The assumption that Bob should receive twice as much as Alice is incorrect. Since Alice started unwinding before the losses occurred, she should receive her full totalReward amount. On the other hand, Bob began his unwinding period after the losses had occurred, causing him to bear the loss.
  
  In fact, Alice should receive 2x the amount that Bob gets, because she had already initiated her unwinding and should not be penalized after her locked tokens were burned — as shown [here](https://cantina.xyz/code/2ac7f906-1661-47eb-bfd6-519f5db0d36b/src/locking/LockingController.sol?lines=231,231).
  

