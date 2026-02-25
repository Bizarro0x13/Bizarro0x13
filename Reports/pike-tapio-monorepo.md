# Pike-tapio-monorepo — Bug Reports
https://cantina.xyz/code/a0806644-7d91-457a-a08d-aee2db73f352/overview
---

## [M-1] Incorrect Dynamic Fee Calculation Due to Redundant Exchange Rate Multiplication

### Summary
The mint function incorrectly calculates the xs value during dynamic fee computation by multiplying the token balances with the exchange rate again, dispite the fact that the token balances are already stored in underlyingToken-equivalent terms post exchange rate conversion. This causes the dynamicFee to be higher than expected.

### Finding Description
uint256 xs = ((balances[i] + _balances[i]) * exchangeRateProviders[i].exchangeRate()) / (10 ** exchangeRateDecimals[i]);
This line intends to compute the weighted liquidity value(xs) of a token in underlyingToken value. However balances[i] and _balances[i] values are already incorporate the exchange rate, as shown here:

```js
function _updateBalancesForDeposit(
        uint256[] memory _balances,
        uint256[] calldata _amounts
    )
        internal
        view
        returns (uint256[] memory)
    {
        for (uint256 i = 0; i < _balances.length; i++) {
            if (_amounts[i] == 0) continue;
            uint256 bal = (_amounts[i] * exchangeRateProviders[i].exchangeRate()) / (10 ** exchangeRateDecimals[i]);
            _balances[i] += bal * precisions[i];
        }
        return _balances;
    }
```
Within _updateBalancesForDeposit, each _amounts[i] is multiplied by the exchange rate and added to the balances[i].

Applying the exchange rate again results in:

```
effective_balance = (rate × amount) × rate = amount × rate²
```
- If rate is 1.1e18, then effective xs = 1.21x intended value.
- Fee is calculated as function of (xs,ys) -> hence the fee is incorrectly inflated.
```
fees[i] = (difference * (_dynamicFee(xs, ys, mintFee) + _volatilityFee(i, mintFee))) / FEE_DENOMINATOR;
```
Since xs is higher than it should be, the _dynamicFee(xs, ys, mintFee) value increases due to:

```
xps2 = (xpi + xpj)^2  → inflated by xs²
```

Thus, liquidity providers receive fewer LP tokens for the same deposit.

### Impact Explanation
Impact is High because when the exchange rate is over 1e18 then the user looses tokens and when the rate is under 1e18 then the protocol looses tokens.

### Likelihood Explanation
Likelihood if this happening is High because this affects all minting operations when tokens have exchange rates significantly above 1 or below 1.

### Proof of Concept
Paste this line of code in the SelfPeggingAsset.sol

```js
function mint(
        uint256[] calldata _amounts,
        uint256 _minMintAmount
    )
        external
        nonReentrant
        syncRamping
        returns (uint256)
    {
       .....
       
       
        if (mintFee > 0 && oldD != 0) {
               ...
            for (uint256 i = 0; i < _balances.length; i++) {
                uint256 idealBalance = newD * balances[i] / oldD;
                uint256 difference =
                    idealBalance > _balances[i] ? idealBalance - _balances[i] : _balances[i] - idealBalance;
                uint256 xs = ((balances[i] + _balances[i]) * exchangeRateProviders[i].exchangeRate())
                    / (10 ** exchangeRateDecimals[i]);
                fees[i] = (difference * (_dynamicFee(xs, ys, mintFee) + _volatilityFee(i, mintFee))) / FEE_DENOMINATOR;
                _balances[i] -= fees[i];
+                uint256 actualDynamicFee = _dynamicFee(xs, ys, mintFee);
+                uint256 correctDynamicFee = _dynamicFee(balances[i] + _balances[i], ys, mintFee);
+                console2.log("Actual Fee: ", actualDynamicFee);
+                console2.log("Correct Fee: ", correctDynamicFee);
                
            }
            newD = _getD(_balances, A);
            mintAmount = newD - oldD;
        }
            .....
    }
```
Paste the below test in the SelfPeggingAsset.t.sol and run forge test --mt test_wrong_dynamicFee_calc -vvv

```js
function test_wrong_dynamicFee_calc() public {
        MockToken rETH = new MockToken("rETH", "rETH", 18);
        MockToken wstETH = new MockToken("wstETH", "wstETH", 18);

        MockExchangeRateProvider provider0 = new MockExchangeRateProvider(1.1e18, 18); // <--- The exchange rate is set to be greater than 1e18.
        MockExchangeRateProvider provider1 = new MockExchangeRateProvider(1e18, 18);

        ERC1967Proxy proxy = new ERC1967Proxy(address(new SPAToken()), new bytes(0));
        SPAToken spaToken1 = SPAToken(address(proxy));

        address[] memory tokens = new address[](2);
        tokens[0] = address(rETH);
        tokens[1] = address(wstETH);

        IExchangeRateProvider[] memory providers = new IExchangeRateProvider[](2);
        providers[0] = provider0;
        providers[1] = provider1;

        uint256[] memory fees = new uint256[](3);
        fees[0] = 0.01e10; // 1% mint fee
        fees[1] = 0.01e10; // 1% swap fee
        fees[2] = 0.01e10; // 1% redeem fee

        uint256[] memory precisionsArray = new uint256[](2);
        precisionsArray[0] = 1;
        precisionsArray[1] = 1;

        bytes memory data = abi.encodeCall(
            SelfPeggingAsset.initialize,
            (tokens, precisionsArray, fees, 1e11, spaToken1, 100, providers, address(0), 1e10, owner)
        );

        proxy = new ERC1967Proxy(address(new SelfPeggingAsset()), data);
        SelfPeggingAsset spa = SelfPeggingAsset(address(proxy));
        spaToken1.initialize("SPA Token", "TSPA", 5e8, owner, address(spa));

        uint256[] memory amounts1 = new uint256[](2);
        amounts1[0] = 10_000e18;
        amounts1[1] = 5_000e18;

        rETH.mint(user, amounts1[0]);
        wstETH.mint(user, amounts1[1]);

        vm.startPrank(user);
        rETH.approve(address(spa), amounts1[0]);
        wstETH.approve(address(spa), amounts1[1]);
        spa.mint(amounts1, 0);
        vm.stopPrank();

        rETH.mint(user2, amounts1[0]);
        wstETH.mint(user2, amounts1[1]);

        vm.startPrank(user2);
        rETH.approve(address(spa), amounts1[0]);
        wstETH.approve(address(spa), amounts1[1]);
        spa.mint(amounts1, 0);
        vm.stopPrank();
    }
```
The test will log
```sh
[PASS] test_wrong_dynamicFee_calc() (gas: 8406780)
Logs:
  Actual Fee:  103898119       # here the fee is high because of double multiplication
  Correct Fee:  102301167     # here the fee is correct
  Actual Fee:  105025365
  Correct Fee:  105025365

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 10.85ms (3.58ms CPU time)
```

### Recommendation

```js
function mint(
        uint256[] calldata _amounts,
        uint256 _minMintAmount
    )
        external
        nonReentrant
        syncRamping
        returns (uint256)
    {
       .....
       
       
        if (mintFee > 0 && oldD != 0) {
               ...
            for (uint256 i = 0; i < _balances.length; i++) {
                uint256 idealBalance = newD * balances[i] / oldD;
                uint256 difference =
                    idealBalance > _balances[i] ? idealBalance - _balances[i] : _balances[i] - idealBalance;
-                uint256 xs = ((balances[i] + _balances[i]) * exchangeRateProviders[i].exchangeRate()) / (10 ** exchangeRateDecimals[i]);
+                uint256 xs = (balances[i] + _balances[i]);
                fees[i] = (difference * (_dynamicFee(xs, ys, mintFee) + _volatilityFee(i, mintFee))) / FEE_DENOMINATOR;
                _balances[i] -= fees[i];
                
            }
            newD = _getD(_balances, A);
            mintAmount = newD - oldD;
        }
            .....
    }
```