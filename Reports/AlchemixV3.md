# AlchemixV3 — Bug Reports
https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa/leaderboard/
---

# h-1 _mytSharesDeposited Not Reduced on Liquidation, leading to Deposit Cap Bypass and potential insovency

Details
Report ID
58491
Target
https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistV3.sol
Smart Contract
Impact(s)
Smart contract unable to operate due to lack of token funds
Protocol insolvency
Description
Brief/Intro
In the _doLiquidation function, when a user is liquidated, their collateral balance is reduced but the global varibale _mytSharesDeposited is not decreased accordingly. This results in an inconsistency between the actual MYT collateral held by the contract and the reported global deposits which can lead to insolvency.

Vulnerability Details
The _doLiquidation function fails to decrease _mytSharesDeposited by the amount of collateral removed from the user during liquidation. This leads to an inflated view of total deposited MYT tokens in the protocol In _forceRepay also there is no decrease in _mytSharesDeposited after reducing collateralBalance

function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
        internal
        returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
    {
        ---

        amountLiquidated = convertDebtTokensToYield(liquidationAmount);
        feeInYield = convertDebtTokensToYield(baseFee);

        // update user balance and debt
@>1        account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
        _subDebt(accountId, debtToBurn);

        ---
    }

function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {

        ---

        creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
@2>        account.collateralBalance -= creditToYield;

        if (account.collateralBalance > protocolFeeTotal) {
@3>            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        }

        ---
    }

@1 -> In _doLiquidation the collateralBalance of the user is decreased but the _mytSharesDeposited is not decreased
@2, @3 -> In _forceRepay, user's collateralBalance is decreased but the _mytSharesDeposited is not decreased.
Adding to this, the transmuter also uses the _mytSharesDeposited amount to calculate the badDebtRatio which helps to determine if the user will get 1:1 for their alAsset deposited or not. If the badDebtRatio is greater than 1e18, then user will not get 1:1 for their tokens and the badDebtRatio is calculated as

uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;
Here the denominator is inflated because of the inflated _mytSharesDeposited amount leading to deflated badDebtRatio and causing protocol to pay 1:1 ratio of tokens to the redeemer even though the protocol should be transferring less amount of mytTokens.

        if (badDebtRatio > 1e18) {
            scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
        }
This leads to protocol paying more that it should leading to insolvency as the 1:1 cannot be maintained because of alAssets being more than their collateral backing but still protocol has to pay 1:1 to the redeemer.

Impact Details
Insolvency Risk: _getTotalUnderlyingValue will be incorrect as the _mytSharesDeposited will be inflated leading to insovency as the calculation will lead to a situation where it is unable to meet all user claims(alUSDC can be redeemed for 1:1 even when there isn’t enough backing)
Deposit Cap Bypass: As the mytSharesDeposited is not updated, the protocol's deposit cap logic will be reached way before the actual myt shares in the contract.
References
[https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L852](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L852)

[https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L219](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L219)

Proof of Concept
paste this test in test/AlchemistV3.t.sol and forge test --mt test_mytSharesDeposited_not_Reduced -vvv

    function test_mytSharesDeposited_not_Reduced() external {
        // ------ set the deposit cap to depositAmount * 2 + 1
        vm.startPrank(alchemist.admin());
        alchemist.setDepositCap(depositAmount*2 + 1);
        vm.stopPrank();

        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // ------- User 1 Deposits depositAmount 
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // ------- User 2 Deposits depositAmount and borrow
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization(), address(0xbeef));
        vm.stopPrank();

        uint256 transmuterPreviousBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 900 bps or 9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 900 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // ------- External User liquidates user2 decreasing the collateral and myt balance of the contract.
        vm.startPrank(externalUser);
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
        vm.stopPrank();

        // -------- Another user tries to deposit 2 wei amount of tokens leading to exceeded depositCap.
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(2, yetAnotherExternalUser, 0);
        vm.stopPrank();
    }
The function will revert with IllegalState() error.

[3688] MockMYTVault::approve(TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], 400000000000000000000000 [4e23])
    │   ├─ emit Approval(owner: 0x520aB24368e5Ba8B727E9b8aB967073Ff9316961, spender: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], value: 400000000000000000000000 [4e23])
    │   └─ ← [Return] true
    ├─ [3569] TransparentUpgradeableProxy::fallback(2, 0x520aB24368e5Ba8B727E9b8aB967073Ff9316961, 0)
    │   ├─ [2844] AlchemistV3::deposit(2, 0x520aB24368e5Ba8B727E9b8aB967073Ff9316961, 0) [delegatecall]
    │   │   └─ ← [Revert] IllegalState()
    │   └─ ← [Revert] IllegalState()
    └─ ← [Revert] IllegalState()

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 24.78ms (4.47ms CPU time)

Ran 1 test suite in 332.19ms (24.78ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in src/test/AlchemistV3.t.sol:AlchemistV3Test
[FAIL: IllegalState()] test_mytSharesDeposited_not_Reduced() (gas: 1421377)



# h-2 Incorrect Handling of cumulativeEarmarked in _forceRepay leads to inflated survival accumulator.

Details
Report ID
58337
Target
https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistV3.sol
Smart Contract
Impact(s)
Protocol insolvency
Description
Brief/Intro
In the _forceRepay function, when a liquidator liquidates a user, the user's individial earmarked amount is reduced but the global cumulativeEarmarked variable is not reduced. This leads to an inflated cumulativeEarmarked value, when redemptions occur, which in turn causes the _survivalAccumulator to be incorrectly calculated.

Vulnerability Details
When a liquidator liquidates a user's position, the liquidate function is called which internally calls the _liquidate fucntion, In the liquidate function if the user's collateraleralizaionRation is less than the collateralizationLowerBound, the function calls the _forceRepay to repay the earmarked position of the user. Inside the _forceRepay function user's collateral is reduced by the earmarked amount + fee amount and the user's earmarked amount is set to 0, but here the cumulativeEarmaked which globally tracks the earmarked amount is not updated which leads to inflated cumulativeEarmaked amount. The main problem lies in the redeem function because at the time of redemption, the cumulativeEarmarked is used to calculate the new _survivalAccumulator which signifies the fraction of earmarked debt that has survived redemption. and as the cumulativeEarmarked is wrong as well as the _survivalAccumulator.

function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
        if (amount == 0) {
            return 0;
        }
        _checkForValidAccountId(accountId);
        Account storage account = _accounts[accountId];

        // Query transmuter and earmark global debt
        _earmark();

        // Sync current user debt before deciding how much is available to be repaid
        _sync(accountId);

        uint256 debt;

        // Burning yieldTokens will pay off all types of debt
        _checkState((debt = account.debt) > 0);

        uint256 credit = amount > debt ? debt : amount;
        uint256 creditToYield = convertDebtTokensToYield(credit);
        _subDebt(accountId, credit);

        // Repay debt from earmarked amount of debt first
        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
@1>        account.earmarked -= earmarkToRemove;

        creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
        account.collateralBalance -= creditToYield;

        uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;

        emit ForceRepay(accountId, amount, creditToYield, protocolFeeTotal);

        if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        }

        if (creditToYield > 0) {
            // Transfer the repaid tokens from the account to the transmuter.
            TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
        }
        return creditToYield;
    }

function redeem(uint256 amount) external onlyTransmuter {
        _earmark();

@2>        uint256 liveEarmarked = cumulativeEarmarked;
        if (amount > liveEarmarked) amount = liveEarmarked;

        ---

        uint256 redeemedDebtTotal = amount + coverToApplyDebt;

       // Apply redemption weights/decay to the full amount that left the earmarked bucket
        if (liveEarmarked != 0 && redeemedDebtTotal != 0) {
@3>            uint256 survival = ((liveEarmarked - redeemedDebtTotal) << 128) / liveEarmarked;
@4>            _survivalAccumulator = _mulQ128(_survivalAccumulator, survival);
            _redemptionWeight += PositionDecay.WeightIncrement(redeemedDebtTotal, cumulativeEarmarked);
        }

        // earmarks are reduced by the full redeemed amount (net + cover)
        cumulativeEarmarked -= redeemedDebtTotal;

        // global borrower debt falls by the full redeemed amount
        totalDebt -= redeemedDebtTotal;

        ---

        emit Redemption(redeemedDebtTotal);
    }

@1 -> here user's earmarked amount is redeuced but the cumulativeEarmark amount is not.
@2 -> here liveEarmark is cumulativeEarmark
@3 -> here survival is calculated as (liveEarmarked - redeemedDebtTotal) / liveEarmarked
@4 -> wrong _survivalAccumulator is set because of inflated liveEarmarked.
Impact Details
since the cumulativeEarmarked is too high, the survival ratio decreases more than it should. All remaining user share are reduced unfairly.
Protocol ends up transferring more mytTokens to the transmuter than it should because of increased cumulativeEarmarked causing protocol to transfer unearmarked amount to the transmuter.
        // if the cumulativeEarmarked is inflated(100e18) but the actual earmarkedAmount is 50e18 and user redeems amount(80e18), the protocol ends up paying 30e18 tokens from unearmarked amount from the users.
        uint256 liveEarmarked = cumulativeEarmarked;
        if (amount > liveEarmarked) amount = liveEarmarked;
Attack Path
System starts with 1e6 tokens and 9e5 earmarked.
user liqudiator for 1e5 eamarked, but cumulativeEarmarked stays at 9e5(should be 8e5).
On redemption, the system uses 9e5 in the denominator, so everyone's claim shrinks by more than 1/9th instead of 1/8th.
After several rounds, the sum of all user claims + protocol liabilities exceeds the vault's real assets.
Users cannot withdraw all the valut they are owed, even though no explicit hack or loss happened.
References
[https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L738](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L738)

[https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L592](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L592)

Proof of Concept
paste this test in test/AlchemistV3.t.sol and run forge test --mt test_cumulativeEarmarked_notReduced -vvv

function test_cumulativeEarmarked_notReduced() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // ------- User 1 Deposits depositAmount 
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        uint256 anotherUserToken = AlchemistNFTHelper.getFirstTokenId(address(yetAnotherExternalUser), address(alchemistNFT));
        vm.stopPrank();

        // ------- User 2 Deposits depositAmount and borrow
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization(), address(0xbeef));
        address debtToken = alchemist.debtToken();

        Transmuter muter = Transmuter(alchemist.transmuter());
        IERC20(debtToken).approve(address(muter), 1000e18);
        muter.createRedemption(1000e18);
        vm.stopPrank();


        vm.roll(block.number + 100);

        alchemist.poke(anotherUserToken);

        uint256 earmarkedBefore = alchemist.cumulativeEarmarked();
        console.log(earmarkedBefore);

        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 900 bps or 9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 750 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        uint256 liquidatorBalanceBefore =  IERC20(address(vault)).balanceOf(address(externalUser));
        vm.startPrank(externalUser);
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);

        uint256 earmarkedAfter = alchemist.cumulativeEarmarked();
        console.log(earmarkedAfter);
        vm.stopPrank();

        // -------- After liquidation the cumulativeEarmark does not change even though the user's earmark amount is reduced.
        assert(earmarkedAfter == earmarkedBefore);
    }
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] test_cumulativeEarmarked_notReduced() (gas: 3527162)
Logs:
  19025875190258752
  19025875190258752

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 29.25ms (8.66ms CPU time)

Ran 1 test suite in 264.36ms (29.25ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)

# h-3 Incorrect collateral and fee Check in _doLiquidation Allows Liquidator to loose fee.

Details
Report ID
58061
Target
https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistV3.sol
Smart Contract
Impact(s)
Permanent freezing of funds
Description
Brief/Intro
In _doLiquidation function where the contract checks that collateralBalance >= fee before transferring the fee to the liquidator(msg.sender). This is incorrect because, in calculateLiquidation, the fee is already included in the liquidationAmount returned and is removed from the user's collateral.

Vulnerability Details
The function double-counts the fee in the collateral balance check. The fee is included in the liquidationAmount(already removed from the user's balance), but the contract still checks if the remaining balance is sufficient to pay the fee again.

Path:

User position: collateralBalance = 109, liquidationAmount = 100, fee = 10
_doLiquidation removes 100 from collateralBalance, leaving 9.
The fee check collateralBalance >= fee fails (9<10).
Fee transfer to the liquidator does not occur, even though tokens are already removed from the user.
    function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
        internal
        returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
    {
        Account storage account = _accounts[accountId];

        (uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee, uint256 outsourcedFee) = calculateLiquidation(
            collateralInUnderlying,
            account.debt,
            minimumCollateralization,
            normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt,
            globalMinimumCollateralization,
            liquidatorFee
        );

        amountLiquidated = convertDebtTokensToYield(liquidationAmount);
        feeInYield = convertDebtTokensToYield(baseFee);

        // update user balance and debt
@1>         account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
        _subDebt(accountId, debtToBurn);

        // send liquidation amount - fee to transmuter
        TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);

        // send base fee to liquidator if available
@2>        if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
        }

        // Handle outsourced fee from vault
        if (outsourcedFee > 0) {
            uint256 vaultBalance = IFeeVault(alchemistFeeVault).totalDeposits();
            if (vaultBalance > 0) {
                uint256 feeBonus = normalizeDebtTokensToUnderlying(outsourcedFee);
                feeInUnderlying = vaultBalance > feeBonus ? feeBonus : vaultBalance;
                IFeeVault(alchemistFeeVault).withdraw(msg.sender, feeInUnderlying);
            }
        }

        emit Liquidated(accountId, msg.sender, amountLiquidated + repaidAmountInYield, feeInYield, feeInUnderlying);
        return (amountLiquidated + repaidAmountInYield, feeInYield, feeInUnderlying);
    }

    function calculateLiquidation(
        uint256 collateral,
        uint256 debt,
        uint256 targetCollateralization,
        uint256 alchemistCurrentCollateralization,
        uint256 alchemistMinimumCollateralization,
        uint256 feeBps
    ) public pure returns (uint256 grossCollateralToSeize, uint256 debtToBurn, uint256 fee, uint256 outsourcedFee) {


        ---


        // gross collateral seize = net + fee
@>3        grossCollateralToSeize = debtToBurn + fee;
    }
@3 = calculateLiquidation is returning (debtToBurn + fee, , fee,);
@1 = collateral is reduced by amountLiquidated (debtToBurn + fee converted to yield token)
@2 = after removing the fee from the collateral, collateralBalance >= feeInYield is required to transfer fee to the liquidator
Impact Details
If users' collateralBalance is just above the liquidationAmount then removing liquidationAmount leaves only 9 tokens, the subsequent check (collateralBalance >= fee) fails (9 < 10), so the fee transfer to the liquidator does not occur and the funds are stuck in the protocol, even though the fee was already deducted. This means:

The fee is reduced from the user's collateral but is never transferred to the liquidator.
Liquidator funds are stuck in the contract.
References
[https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1244](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1244)

[https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L852](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L852)

Proof of Concept
Proof of Concept
Paste this test in test/AlchemistV3.t.sol and run forge test --mt testLiquidate_no_fee -vvv

function testLiquidate_no_fee() external {
        vm.startPrank(alchemist.admin());
        alchemist.setLiquidatorFee(9500);
        vm.stopPrank();

        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization(), address(0xbeef));
        vm.stopPrank();

        uint256 transmuterPreviousBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 900 bps or 9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 900 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        uint256 liquidatorBalanceBefore =  IERC20(address(vault)).balanceOf(address(externalUser));
        vm.startPrank(externalUser);
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
        vm.stopPrank(); 
        uint256 liquidatorBalanceAfter = IERC20(address(vault)).balanceOf(address(externalUser));

        assertEq(liquidatorBalanceAfter - liquidatorBalanceBefore, 0);
    }
The test will pass showing that the liquidator will not receive any fee.

