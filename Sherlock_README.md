# Leaderboard
[Numa Results]()<br>

`Rank ? / ?`

# Audited Code Repo
### [Sherlock: Numa](https://audits.sherlock.xyz/contests/554)
### [Github: Numa](https://github.com/sherlock-audit/2024-12-numa-audit)

<br>

# <a id="summaryTable"></a>Bugs Filed & Their Status

| #      | Bug ID          | Name | URL    | Adjudged Status  |
|--------|-----------------|------|:------:|-----------------:|
| 1      | [H-01](#h-01)   | Protocol ignores debt rewards when checking if rewards are above threshold | [66](https://audits.sherlock.xyz/contests/554/voting/66) |  |
| 2      | [H-02](#h-02)   | Debts can never be fully liquidated due to closeFactorMantissa constraint | [80](https://audits.sherlock.xyz/contests/554/voting/80) |  |
| 3      | [H-03](#h-03)   | Incorrect liquidation mechanics either causes revert on liquidation due to insufficient seizeTokens or causes transition into bad debt | [101](https://audits.sherlock.xyz/contests/554/voting/101) |  |
| 4      | [H-04](#h-04)   | closeFactorMantissa is not multiplied while handling max liquidations by liquidateLstBorrower() and liquidateNumaBorrower() when type(uint256).max is passed | [135](https://audits.sherlock.xyz/contests/554/voting/135) |  |
| 5      | [M-01](#m-01)   | Multiplication instead of division causes incorrect decimal precision conversion in ethToToken() and ethToTokenRoundUp() | [29](https://audits.sherlock.xyz/contests/554/voting/29) |  |
| 6      | [M-02](#m-02)   | Healthy debts can be liquidated by manipulating nuAsset price | [61](https://audits.sherlock.xyz/contests/554/voting/61) |  |
| 7      | [M-03](#m-03)   | Liquidator may end up receiving no seizeTokens and lose all repayAmount | [62](https://audits.sherlock.xyz/contests/554/voting/62) |  |
| 8      | [M-04](#m-04)   | Removal of closeFactorMantissa & maxClose constraints in deprecated markets allows attack vector to worsen protocol's health | [63](https://audits.sherlock.xyz/contests/554/voting/63) |  |
| 9      | [M-05](#m-05)   | Interest charged to borrower even when protocol is paused | [64](https://audits.sherlock.xyz/contests/554/voting/64) |  |
| 10     | [M-06](#m-06)   | closeLeverageStrategy() is not allowed during vault's paused state but repayBorrow() is | [65](https://audits.sherlock.xyz/contests/554/voting/65) |  |
| 11     | [M-07](#m-07)   | Deprecated markets allow profitable exploitation of bad debt liquidations | [67](https://audits.sherlock.xyz/contests/554/voting/67) |  |
| 12     | [M-08](#m-08)   | Checks not implemented for lower & upper bounds of closeFactorMantissa | [78](https://audits.sherlock.xyz/contests/554/voting/78) |  |
| 13     | [M-09](#m-09)   | Missing validation in setSellFee() can cause sell_fee_withPID to never rebase | [82](https://audits.sherlock.xyz/contests/554/voting/82) |  |
| 14     | [M-10](#m-10)   | Users may end up paying higher than required fees during opening & closing of leveraged strategies | [91](https://audits.sherlock.xyz/contests/554/voting/91) |  |

<br>
<br>

## **HIGH-SEVERITY BUGS**
---

### <a id="h-01"></a>[H-01]
## **Protocol ignores debt rewards when checking if rewards are above threshold**
#### https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L383
<br>

## Summary
The following check inside [extractRewardsNoRequire()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L383) should be modified as follows to include the debt reward while comparing with `rwd_threshold`. Identical logic exists inside [lstToNuma()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L697) and [numaToLst()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L719):
```diff
    function extractRewardsNoRequire() internal {
        if (block.timestamp >= (last_extracttimestamp + 24 hours)) {
            (
                uint256 rwd,
                uint256 currentvalueWei,
                uint256 rwdDebt
            ) = rewardsValue();
-           if (rwd > rwd_threshold) {
+           if (rwd + rwdDebt > rwd_threshold) {
                extractInternal(rwd, currentvalueWei, rwdDebt);
            }
        }
    }
```

## Description
Consider the following flow:
- `rwd_threshold` is set as `1 ether`. Note that currently it's set at `0` but can be readily changed by the owner calling [setRewardsThreshold()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L296)
- Inside vault, total rEth value in ETH is `8 ETH` (for ease of calculation, let's say 1 rEth = 8 ETH)
- A 25% rebase occurs pushing rEth value to `10 ETH`. Vault can be updated via `updateVault()` which internally calls `extractRewardsNoRequire()` if 24 hours have passed since the last update. We'll assume it has been more than 24 hours.
- Case 1:
    - No debts.
    - `updateVault() --> extractRewardsNoRequire() --> rewardsValue()` is called.
    - `rwd` is evaluated as `2 ether` and `debtRwd = 0`.
    - Since `rwd > rwd_threshold` is `true`, `extractInternal()` will be called, rewards distributed and `last_lsttokenvalueWei` will be updated to the new price of `10 ETH`
- Case 2:
    - Debt value = 6 ETH.
    - `updateVault() --> extractRewardsNoRequire() --> rewardsValue()` is called.
    - `rwd` is evaluated as `0.8 ether` and `debtRwd = 1.2 ether`.
    - Since `rwd > rwd_threshold` is `false`, `extractInternal()` will be not be called, [rewardsFromDebt is not incremented on L358](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L358), rewards not distributed,  and `last_lsttokenvalueWei` not updated to the new price of `10 ETH`
        - In case of [lstToNuma()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L697) and [numaToLst()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L719), `refValue` is not updated to `currentvalueWei`

## Impact
Stale values used if vault has debt, leading to incorrect accounting across the protocol.

## Mitigation 
Add `rwdDebt` to `rwd` whenever comparing with `rwd_threshold`:
```diff
-           if (rwd > rwd_threshold) {
+           if (rwd + rwdDebt > rwd_threshold) {
```

[Back to Top](#summaryTable)
---

### <a id="h-02"></a>[H-02]
## **Debts can never be fully liquidated due to closeFactorMantissa constraint**
#### https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L604-L611
<br>

## Summary
A debt can only be partially liquidated due to the `closeFactorMantissa` & `maxClose` constraint. The remaining balance remains stuck in the system as unhealthy debt.

## Root Causes
- `closeFactorMantissa` is mandated by the protocol to [always be less than 1](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L108)
- `maxClose` is [calculated by multiplying `borrowBalance` with `closeFactorMantissa`](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L605-L608)
- [repayAmount > maxClose](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L609-L611) is not allowed by the protocol. 

## Description
- Although the tests have been setup with a `closeFactorMantissa` of 1, this is not as per specs of the protocol as can be seen [here](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L108):
```js
        // closeFactorMantissa must be strictly greater than this value
        uint internal constant closeFactorMinMantissa = 0.05e18; // 0.05

@-->    // closeFactorMantissa must not exceed this value
        uint internal constant closeFactorMaxMantissa = 0.9e18; // 0.9
```

Currently `_setCloseFactor()` has a bug which allows surpassing this limit, but assuming that to not be the case in the live version, we assume a suitable value for `closeFactorMantissa` say, `0.9e18` or `90%`.

- This means whatever `repayAmount` is passed by the liquidator, it should not exceed `90%` of the `borrowBalance`

- The protocol [allows only full liquidations](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L1135-L1138) if the `borrowBalance` goes below `minBorrowAmountAllowPartialLiquidation` which [is currently set to](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L85) `10 ether`. This however is not working either due to the `closeFactorMantissa` & `maxClose` constraint.

As a result, there is no way to fully liquidate a debt with a shortfall once it goes below `minBorrowAmountAllowPartialLiquidation`.

## Impact
This is highly problematic because this would occur for **_all the debts with a shortfall which go below minBorrowAmountAllowPartialLiquidation before attaining a healthy state via partial liquidations_**. Consider the following:

- Setup: Acceptable LTV is 80% and current LTV is 90%. `borrowBalance = 20 ether`. Eligible for liquidation.

- First liquidation call : Partial liquidation improves the LTV to 85% but now `borrowBalance = 9 ether` i.e. below `minBorrowAmountAllowPartialLiquidation`. 

- Second liquidation call: Only full liquidation allowed now. But even if the liquidator passes `type(uint256).max` as the repay amount, it will be reduced due to `closeFactorMantissa < 1` and hence call will revert with error `TOO_MUCH_REPAY`.

## Proof of Concept
The following test shows how a shortfall-debt which was initially above `minBorrowAmountAllowPartialLiquidation` eventually transitioned into `borrowBalance < minBorrowAmountAllowPartialLiquidation` & could not be fully liquidated at any stage:

- Add the necessary imports inside `Vault.t.sol` first:
```js
    import {ComptrollerErrorReporter, TokenErrorReporter} from "../lending/ErrorReporter.sol";
```

- Then add the following test and run to see it pass:
```js
    function test_cannotLiquidateFully() public {
        // Initial setup
        vm.startPrank(deployer);
        vaultManager.setSellFee(1 ether); // no sell fee
        comptroller._setCollateralFactor(cNuma, 0.85 ether); // 85% LTV allowed
        vm.stopPrank();

        uint collateralAmount = 25 ether;
        uint borrowAmount = 20 ether;  // 80% LTV to start

        deal({token: address(rEth), to: userA, give: collateralAmount});
        
        // First approve and enter the NUMA market
        vm.startPrank(userA);
        address[] memory t = new address[](1);
        t[0] = address(cNuma);
        comptroller.enterMarkets(t);

        // Buy NUMA with rETH to use as collateral
        rEth.approve(address(vault), collateralAmount);
        uint numas = vault.buy(collateralAmount, 0, userA);

        // Deposit NUMA as collateral
        uint cNumaBefore = cNuma.balanceOf(userA);
        numa.approve(address(cNuma), numas);
        cNuma.mint(numas);
        uint cNumas = cNuma.balanceOf(userA) - cNumaBefore;
        
        emit log_named_decimal_uint("Numas deposited =", numas, 18);
        emit log_named_decimal_uint("cNumas received =", cNumas, 18);

        // Borrow rETH
        cReth.borrow(borrowAmount);
        uint initialBorrowBalance = cReth.borrowBalanceCurrent(userA);
        emit log_named_decimal_uint("Initial borrow =", initialBorrowBalance, 18);
        (, , uint shortfall, uint badDebt) = comptroller.getAccountLiquidityIsolate(userA, cNuma, cReth);
        assertEq(shortfall, 0, "Unhealthy borrow");
        vm.stopPrank();

        // Make position liquidatable
        vm.startPrank(deployer);
        vaultManager.setSellFee(0.90 ether); 
        
        // Verify position is liquidatable
        (, , shortfall, badDebt) = comptroller.getAccountLiquidityIsolate(userA, cNuma, cReth);
        assertGt(shortfall, 0, "Position should be liquidatable");
        assertEq(badDebt, 0, "Position shouldn't be in badDebt region");
        emit log_named_decimal_uint("Shortfall =", shortfall, 18);

        // Set liquidation incentive
        comptroller._setLiquidationIncentive(1.05e18); // 5% liquidation incentive
        // Set close factor
        comptroller._setCloseFactor(0.9e18); // 90%
        vm.stopPrank();

        // First liquidation attempt
        vm.startPrank(userC); // liquidator
        uint borrowBalance = cReth.borrowBalanceCurrent(userA);
        uint repayAmount = (borrowBalance * 55) / 100; // repaying 55% of the debt
        
        deal({token: address(rEth), to: userC, give: repayAmount});
        rEth.approve(address(vault), repayAmount);
        
        vault.liquidateLstBorrower(userA, repayAmount, false, false);
        emit log_named_decimal_uint("First liquidation repaid =", repayAmount, 18);

        uint remainingBorrow = cReth.borrowBalanceCurrent(userA);
        emit log_named_decimal_uint("Remaining borrow =", remainingBorrow, 18);
        // Only full liquidation allowed now
        assertLt(remainingBorrow, vault.minBorrowAmountAllowPartialLiquidation(), "below minBorrowAmountAllowPartialLiquidation");
        // Verify again the position is liquidatable but is not in the badDebt region
        (, , shortfall, badDebt) = comptroller.getAccountLiquidityIsolate(userA, cNuma, cReth);
        emit log_named_decimal_uint("Shortfall2 =", shortfall, 18);
        emit log_named_decimal_uint("BadDebt2   =", badDebt, 18);
        assertGt(shortfall, 0, "Position2 should be liquidatable");
        assertEq(badDebt, 0, "Position2 shouldn't be in badDebt region");

        deal({token: address(rEth), to: userC, give: remainingBorrow});
        uint repay = remainingBorrow; // full repayment
        rEth.approve(address(vault), repay);

        // @audit-issue : full repay not allowed
        vm.expectRevert(abi.encodeWithSelector(TokenErrorReporter.LiquidateComptrollerRejection.selector, uint(ComptrollerErrorReporter.Error.TOO_MUCH_REPAY)));
        vault.liquidateLstBorrower(userA, repay, false, false); 
        
        // @audit-issue : partial repay not allowed too 
        vm.expectRevert("min liquidation");
        vault.liquidateLstBorrower(userA, repay * 8 / 10, false, false); 
        vm.stopPrank();
        
        // @audit-info : Let's also check that IF the bug didn't exist and full liquidation was
        // allowed, whether or not it would've went through successfully?
        vm.prank(deployer);
        comptroller._setCloseFactor(1e18); // temporary hack to verify intended behaviour

        vm.prank(userC); // liquidator
        vault.liquidateLstBorrower(userA, repay, false, false); // should pass
    }
```

## Mitigation 
One approach would be to ignore the `_setCloseFactor` calculation under following circumstances:
- when `borrowBalance < minBorrowAmountAllowPartialLiquidation`. 
- (optional) when user opts for a full liquidation.

This means that [the protocol should consider `maxClose` to be equal to `borrowBalance`](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L604-L611) in the above scenarios.

[Back to Top](#summaryTable)
---

### <a id="h-03"></a>[H-03]
## **Incorrect liquidation mechanics either causes revert on liquidation due to insufficient seizeTokens or causes transition into bad debt**
#### https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/CToken.sol#L1020-L1024
<br>

## Summary
The protocol's liquidation mechanics are fundamentally flawed in two distinct ways that emerge when performing liquidations on positions. Either:

1. The protocol reverts on liquidation of remaining debt due to insufficient collateral to seize, **OR**

2. The position transitions into bad debt status after a partial liquidation, making the remaining debt unprofitable for liquidators and worsening protocol's health.

## Description
The protocol has two distinct broken liquidation paths that emerge when liquidating positions:

### Prerequisite
The borrower's LTV should have worsened enough such that if the entire debt were to be liquidated, there wouldn't be enough collateral cTokens to seize after adding the liquidation incentive on top. Or in other words the liquidator would encounter a revert with error `LIQUIDATE_SEIZE_TOO_MUCH` if he tried to liquidate the entire debt.

### Path 1: Insufficient `seizeTokens`--> ( _coded as `test_liquidationMechanics_path01`_ )
In this scenario:

1. Imagine that a position becomes liquidatable (has shortfall but not in badDebt). And the `borrowBalance` is above `minBorrowAmountAllowPartialLiquidation`.

2. A liquidator attempts a liquidation. This _will always be a partial liquidation_ due to one of these 3 reasons:
    a. The [closeFactorMantissa](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L104-L108) setting constraints that `repayAmount` is no more than 90% of the `borrowBalance`, even in the best of scenarios.
    b. Liquidator could choose a partial repayment based on their financial capacity.
    c. Liquidator could maliciously choose a partial repayment in order to carry out this attack.

3. The liquidator is awarded a [liquidationIncentive](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L1487-L1511) (for e.g., 12%) and is able to seize those collateral cTokens from the borrower. 

4. Due to the above, every iteration of partial liquidation worsens the ratio of collateral cTokens to the remaining `borrowBalance` ( LTV increases with each iteration ).

5. Eventually a state is arrived where `borrowBalance` is below `minBorrowAmountAllowPartialLiquidation`. Now [only full liquidations are allowed](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L1135-L1138) and hence every liquidation attempt [will revert with error](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/CToken.sol#L1020-L1024) `LIQUIDATE_SEIZE_TOO_MUCH`. Note that the debt is still not in the badDebt territory and hence `liquidateBadDebt()` can't be called yet which wouldn't have cared about awarding any `liquidationIncentive`.

### Path 2: Transition to Bad Debt--> ( _coded as `test_liquidationMechanics_path02`_ )
In the second scenario:

- Steps 1-4 same as above.

- Step 5: The worsening ratio with each partial liquidation iteration eventually pushes the debt into the badDebt territory where `borrowBalance` is greater than the remaining collateral. Although someone can call `liquidateBadDebt()`, it doesn't really offer any incentive to the liquidator. The protocol is already losing money at this point, even if someone cleans up the remaining borrowed balance. 

## Impact
The impact is severe:
1. In Path 1, positions are left with debt that cannot be liquidated due to reverting transactions, leaving the protocol with unclearable bad positions
2. In Path 2, positions transition into bad debt and will now be closed at a loss. The protocol's health is worse than before.

## Proof of Concept
Add the 2 tests inside `Vault.t.sol` and run with `FOUNDRY_PROFILE=lite forge test --mt test_liquidationMechanics -vv` to see them pass:
<details>

<summary>
Click to View
</summary>


```js
    function test_liquidationMechanics_path01() public {
        // Initial setup
        vm.startPrank(deployer);
        vaultManager.setSellFee(1 ether); // no sell fee
        comptroller._setCollateralFactor(cNuma, 0.85 ether); // 85% LTV allowed
        vm.stopPrank();

        uint collateralAmount = 25 ether;
        uint borrowAmount = 20 ether;  // 80% LTV to start

        deal({token: address(rEth), to: userA, give: collateralAmount});
        
        // First approve and enter the NUMA market
        vm.startPrank(userA);
        address[] memory t = new address[](1);
        t[0] = address(cNuma);
        comptroller.enterMarkets(t);

        // Buy NUMA with rETH to use as collateral
        rEth.approve(address(vault), collateralAmount);
        uint numas = vault.buy(collateralAmount, 0, userA);

        // Deposit NUMA as collateral
        uint cNumaBefore = cNuma.balanceOf(userA);
        numa.approve(address(cNuma), numas);
        cNuma.mint(numas);
        uint cNumas = cNuma.balanceOf(userA) - cNumaBefore;
        
        emit log_named_decimal_uint("Numas deposited =", numas, 18);
        emit log_named_decimal_uint("cNumas received =", cNumas, 18);

        // Borrow rETH
        cReth.borrow(borrowAmount);
        uint initialBorrowBalance = cReth.borrowBalanceCurrent(userA);
        emit log_named_decimal_uint("Initial borrow =", initialBorrowBalance, 18);
        (, , uint shortfall, uint badDebt) = comptroller.getAccountLiquidityIsolate(userA, cNuma, cReth);
        assertEq(shortfall, 0, "Unhealthy borrow");
        vm.stopPrank();

        // Make position liquidatable
        vm.startPrank(deployer);
        vaultManager.setSellFee(0.90 ether); 
        
        // Verify position is liquidatable
        (, , shortfall, badDebt) = comptroller.getAccountLiquidityIsolate(userA, cNuma, cReth);
        assertGt(shortfall, 0, "Position should be liquidatable");
        assertEq(badDebt, 0, "Position shouldn't be in badDebt region");
        emit log_named_decimal_uint("Shortfall =", shortfall, 18);

        // Set liquidation incentive
        comptroller._setLiquidationIncentive(1.12e18); // 12% premium
        // Set close factor
        comptroller._setCloseFactor(0.9e18); // 90%
        vm.stopPrank();

        // First liquidation attempt
        vm.startPrank(userC); // liquidator
        uint borrowBalance = cReth.borrowBalanceCurrent(userA);
        uint repayAmount = (borrowBalance * 55) / 100; // repaying 55% of the debt
        
        deal({token: address(rEth), to: userC, give: repayAmount});
        rEth.approve(address(vault), repayAmount);
        
        // This should succeed since there's enough collateral for the first liquidation
        vault.liquidateLstBorrower(userA, repayAmount, false, false);
        emit log_named_decimal_uint("First liquidation repaid =", repayAmount, 18);

        uint remainingBorrow = cReth.borrowBalanceCurrent(userA);
        emit log_named_decimal_uint("Remaining borrow =", remainingBorrow, 18);
        // Only full liquidation allowed now
        assertLt(remainingBorrow, vault.minBorrowAmountAllowPartialLiquidation(), "below minBorrowAmountAllowPartialLiquidation");
        // Verify again the position is liquidatable but is not in the badDebt region
        (, , shortfall, badDebt) = comptroller.getAccountLiquidityIsolate(userA, cNuma, cReth);
        emit log_named_decimal_uint("Shortfall2 =", shortfall, 18);
        emit log_named_decimal_uint("BadDebt2   =", badDebt, 18);
        assertGt(shortfall, 0, "Position2 should be liquidatable");
        assertEq(badDebt, 0, "Position2 shouldn't be in badDebt region");
        vm.stopPrank();

        // temporary hack required to allow full liquidation now since 
        // `borrowBalance < minBorrowAmountAllowPartialLiquidation`. Needs to be done due 
        // to existence of a different bug
        vm.prank(deployer);
        comptroller._setCloseFactor(1e18); 

        // Second liquidation attempt for remaining debt
        vm.startPrank(userC); // liquidator
        deal({token: address(rEth), to: userC, give: remainingBorrow});
        rEth.approve(address(vault), remainingBorrow);
        vm.expectRevert("LIQUIDATE_SEIZE_TOO_MUCH");
        vault.liquidateLstBorrower(userA, remainingBorrow, false, false);  // @audit-issue : no way to liquidate !
        vm.stopPrank();
    }

    function test_liquidationMechanics_path02() public {
        // Initial setup
        vm.prank(deployer);
        vaultManager.setSellFee(1 ether); // no sell fee

        uint collateralAmount = 100 ether;
        uint borrowAmount = 80 ether;  // 80% LTV to start

        deal({token: address(rEth), to: userA, give: collateralAmount});
        
        // First approve and enter the NUMA market
        vm.startPrank(userA);
        address[] memory t = new address[](1);
        t[0] = address(cNuma);
        comptroller.enterMarkets(t);

        // Buy NUMA with rETH to use as collateral
        rEth.approve(address(vault), collateralAmount);
        uint numas = vault.buy(collateralAmount, 0, userA);

        // Deposit NUMA as collateral
        uint cNumaBefore = cNuma.balanceOf(userA);
        numa.approve(address(cNuma), numas);
        cNuma.mint(numas);
        uint cNumas = cNuma.balanceOf(userA) - cNumaBefore;
        
        emit log_named_decimal_uint("Numas deposited =", numas, 18);
        emit log_named_decimal_uint("cNumas received =", cNumas, 18);

        // Borrow rETH
        cReth.borrow(borrowAmount);
        uint initialBorrowBalance = cReth.borrowBalanceCurrent(userA);
        emit log_named_decimal_uint("Initial borrow =", initialBorrowBalance, 18);
        (, , uint shortfall, uint badDebt) = comptroller.getAccountLiquidityIsolate(userA, cNuma, cReth);
        assertEq(shortfall, 0, "Unhealthy borrow");
        vm.stopPrank();

        // Make position liquidatable by manipulating the sell fee
        vm.startPrank(deployer);
        vaultManager.setSellFee(0.87 ether); // price drop making position liquidatable
        
        // Verify position is liquidatable
        (, , shortfall, badDebt) = comptroller.getAccountLiquidityIsolate(userA, cNuma, cReth);
        assertGt(shortfall, 0, "Position should be liquidatable");
        assertEq(badDebt, 0, "Position shouldn't be in badDebt region");
        emit log_named_decimal_uint("Shortfall =", shortfall, 18);

        // Set liquidation incentive 
        comptroller._setLiquidationIncentive(1.12e18); // 12% premium
        // Set close factor 
        comptroller._setCloseFactor(0.85e18); // 85%
        vm.stopPrank();

        // First liquidation attempt
        vm.startPrank(userC); // liquidator
        uint borrowBalance = cReth.borrowBalanceCurrent(userA);
        uint repayAmount = (borrowBalance * 85) / 100; // 85% of the debt
        
        deal({token: address(rEth), to: userC, give: repayAmount});
        rEth.approve(address(vault), repayAmount);
        
        // This should succeed since there's enough collateral for the first liquidation
        vault.liquidateLstBorrower(userA, repayAmount, false, false);
        emit log_named_decimal_uint("First liquidation repaid =", repayAmount, 18);

        // Second liquidation attempt for remaining debt
        uint remainingBorrow = cReth.borrowBalanceCurrent(userA);
        emit log_named_decimal_uint("Remaining borrow =", remainingBorrow, 18);
        // Verify the position again for badDebt
        (, , shortfall, badDebt) = comptroller.getAccountLiquidityIsolate(userA, cNuma, cReth);
        emit log_named_decimal_uint("Shortfall2 =", shortfall, 18);
        emit log_named_decimal_uint("BadDebt2   =", badDebt, 18);
        assertGt(badDebt, 0, "Position2 should be in badDebt region"); // @audit-issue : has become badDebt now; unprofitable for liquidators
        vm.stopPrank();
    }
```

</details>
<br>

Output:
```text
Ran 2 tests for contracts/Test/Vault.t.sol:VaultTest
[PASS] test_liquidationMechanics_path01() (gas: 2288863)
Logs:
  VAULT TEST
  Numas deposited =: 176595.092400931043253778
  cNumas received =: 0.000882975462004655
  Initial borrow =: 20.000000000000000000
  Shortfall =: 1.817974907058021698
  redeem? 0

  First liquidation repaid =: 11.000000000000000000
  Remaining borrow =: 9.000000000000000000
  Shortfall2 =: 1.289974907057879019
  BadDebt2   =: 0.000000000000000000

[PASS] test_liquidationMechanics_path02() (gas: 1828185)
Logs:
  VAULT TEST
  Numas deposited =: 706380.369603724173015115
  cNumas received =: 0.003531901848018620
  Initial borrow =: 80.000000000000000000
  Shortfall =: 1.264378335974950255
  redeem? 0

  First liquidation repaid =: 68.000000000000000000
  Remaining borrow =: 12.000000000000000000
  Shortfall2 =: 5.573919060292848876
  BadDebt2   =: 5.235704273992472501

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 52.60s (5.50s CPU time)
```

## Mitigation 
Add the following 2 checks:

1. If a partial liquidation attempt would result in the debt going into badDebt territory, then it should not be allowed. Full liquidations should be allowed in such cases with a reduced liquidation incentive applicable. The code should allow `repayAmount = borrowBalance` and bypass the `closeFactorMantissa` constraint ( or set it temporarily to `1e18` ).

2. If a full liquidation attempt ( possible when `borrowBalance < minBorrowAmountAllowPartialLiquidation` ) would result in `seizeTokens` to be greater than the cToken collateral balance of the borrower, then the liquidator should still be allowed to go ahead and be awarded all the available cTokens in borrower's balance. 

This possibility of a reduced liquidation incentive should be properly documented so that liquidators know the risk in advance.

[Back to Top](#summaryTable)
---

### <a id="h-04"></a>[H-04]
## **closeFactorMantissa is not multiplied while handling max liquidations by liquidateLstBorrower() and liquidateNumaBorrower() when type(uint256).max is passed**
#### https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L1131-L1133
#### https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L986-L987

<br>

## Summary
The protocol allows a liquidator to pass repay amount as `type(uint256).max` ([here](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L986-L987) and [here](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L1131-L1133)) so that they can liquidate the entire borrow balance but forgets to multiply it with the `closeFactorMantissa`, thus making it impossible to use this option and risking repeated revert of the liquidation attempt (due to front-running).

## Description
Liquidators who wish to avoid situations where their tx gets reverted because someone else (intentionally or unintentionally) front-ran and liquidated some amount before them (quite common) are allowed to pass `type(uint256).max` as the amount to liquidate and the protocol readjusts it to the current borrow balance. However since [closeFactorMantissa](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L104-L108) decides how much of the borrow balance can be closed in one go, their transaction will always revert if `closeFactorMantissa` has not been set to `1e18` (which it should not be, as the max allowed is `0.9e18`). To ensure tx is not reverted, the condition should take care of multiplying with `closeFactorMantissa`:
```diff
  File: contracts/NumaProtocol/NumaVault.sol

   966:              function liquidateNumaBorrower(
   967:                  address _borrower,
   968:                  uint _numaAmount,
   969:                  bool _swapToInput,
   970:                  bool _flashloan
   971:              ) external whenNotPaused notBorrower(_borrower) {
   972:                  // if using flashloan, you have to swap collateral seized to repay flashloan
   973:                  require(
   974:                      ((_flashloan && _swapToInput) || (!_flashloan)),
   975:                      "invalid param"
   976:                  );
   977:          
   978:                  uint criticalScaleForNumaPriceAndSellFee = startLiquidation();
   979:          
   980:                  uint numaAmount = _numaAmount;
   981:          
   982:                  // minimum liquidation amount
   983:                  uint borrowAmount = cNuma.borrowBalanceCurrent(_borrower);
   984:          
   985:                  // AUDITV2FIX: handle max liquidations
   986:                  if (_numaAmount == type(uint256).max) {
-  987:                      numaAmount = borrowAmount;
+  987:                      numaAmount = borrowAmount * closeFactorMantissa;
   988:                  } else {
```

and 

```diff
  File: contracts/NumaProtocol/NumaVault.sol

    1113:              function liquidateLstBorrower(
    1114:                  address _borrower,
    1115:                  uint _lstAmount,
    1116:                  bool _swapToInput,
    1117:                  bool _flashloan
    1118:              ) external whenNotPaused notBorrower(_borrower) {
    1119:                  // if using flashloan, you have to swap colletral seized to repay flashloan
    1120:                  require(
    1121:                      ((_flashloan && _swapToInput) || (!_flashloan)),
    1122:                      "invalid param"
    1123:                  );
    1124:          
    1125:                  uint lstAmount = _lstAmount;
    1126:          
    1127:                  // min liquidation amount
    1128:                  uint borrowAmount = cLstToken.borrowBalanceCurrent(_borrower);
    1129:          
    1130:                  // AUDITV2FIX: handle max liquidations
    1131:                  if (_lstAmount == type(uint256).max) {
-   1132:                      lstAmount = borrowAmount;
+   1132:                      lstAmount = borrowAmount * closeFactorMantissa;
    1133:                  }
```

## Impact
Liquidators would find it extremely difficult to liquidate the max allowed amount and pocket the profits since they run the risk of a reverted tx if someone front runs them (specially on chains like Ethereum). They will have to be satisfied with choosing a lower amount thus limiting their profits.

## Mitigation 
Multiply with `closeFactorMantissa` as shown above.

[Back to Top](#summaryTable)

<br>

## **MEDIUM-SEVERITY BUGS**
---

### <a id="m-01"></a>[M-01]
## **Multiplication instead of division causes incorrect decimal precision conversion in ethToToken() and ethToTokenRoundUp()**
#### https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/libraries/OracleUtils.sol#L98
#### https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/libraries/OracleUtils.sol#L151
<br>

## Description
Functions [ethToToken()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/libraries/OracleUtils.sol#L98) and [ethToTokenRoundUp()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/libraries/OracleUtils.sol#L151) modify the decimal precision between `token` and `ETH` by using the following logic:
```js
        tokenAmount = tokenAmount * 10 ** (18 - _decimals);
```

This is incorrect and division should be used instead of multiplication:
```diff
-       tokenAmount = tokenAmount * 10 ** (18 - _decimals);
+       tokenAmount = tokenAmount / 10 ** (18 - _decimals);
```

**Example:**
1. We want to convert 1 ETH to USDC using a price of $2000 USD/ETH. Starting values:
```js
    _ethAmount = 1 ETH = 1e18 
    price = $2000 * 1e8 (Chainlink price feed uses 8 decimals)
    _decimals = 6 (USDC decimals)
```

2. This gives us:
```js
    tokenAmount = FullMath.mulDiv(
        _ethAmount,
        uint256(price), 
        10 ** AggregatorV3Interface(_pricefeed).decimals()
    )   // because `ethLeftSide(_pricefeed) = TRUE`
    
    => 
    tokenAmount = (1e18 * 2000e8) / 1e8
                = 2000e18
```

3. The returned result ought to be `2000e6` and hence the final step **should be**:
```js
    tokenAmount = tokenAmount / 10 ** (18 - _decimals)

=>  tokenAmount = tokenAmount / 10 ** (18 - 6)

=>  tokenAmount = 2000e18 / 10 ** 12 = 2000e6
```

4. However the current logic returns an astronomically high value (`10 ** 24 times` the correct value):
```js
    tokenAmount = tokenAmount * 10 ** (18 - _decimals)

=>  tokenAmount = tokenAmount * 10 ** (18 - 6)

=>  tokenAmount = 2000e18 * 10 ** 12 = 2000e30
```

## Impact
These functions are [only called from inside `nuAssetManager.sol`](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/nuAssets/nuAssetManager.sol#L195-L213) through functions `ethToNuAsset()` and `ethToNuAssetRoundUp()` which hard-code the `_decimals` to 18. 

The protocol it seems considers that nuAssets will always have a 18-decimal precision. If however this changes in the future and a nuAsset with less than 18 decimals is supported, the impact will be high.

```js
    function ethToNuAsset(
        address _nuAsset,
        uint256 _amount
    ) public view returns (uint256 EthValue) {
        require(contains(_nuAsset), "bad nuAsset");
        nuAssetInfo memory info = getNuAssetInfo(_nuAsset);
        (address priceFeed, uint128 heartbeat) = (info.feed, info.heartbeat);
@-->    return ethToToken(_amount, priceFeed, heartbeat, 18);  // @audit : This should ideally be `return ethToToken(_amount, priceFeed, heartbeat, IERC20Metadata(_nuAsset).decimals());`
    }

    function ethToNuAssetRoundUp(
        address _nuAsset,
        uint256 _amount
    ) public view returns (uint256 EthValue) {
        require(contains(_nuAsset), "bad nuAsset");
        nuAssetInfo memory info = getNuAssetInfo(_nuAsset);
        (address priceFeed, uint128 heartbeat) = (info.feed, info.heartbeat);
@-->    return ethToTokenRoundUp(_amount, priceFeed, heartbeat, 18);  // @audit : This should ideally be `return ethToToken(_amount, priceFeed, heartbeat, IERC20Metadata(_nuAsset).decimals());`
    }
```

## Mitigation 
```diff
-       tokenAmount = tokenAmount * 10 ** (18 - _decimals);
+       tokenAmount = tokenAmount / 10 ** (18 - _decimals);
```

and also
```diff
    function ethToNuAsset(
        address _nuAsset,
        uint256 _amount
    ) public view returns (uint256 EthValue) {
        require(contains(_nuAsset), "bad nuAsset");
        nuAssetInfo memory info = getNuAssetInfo(_nuAsset);
        (address priceFeed, uint128 heartbeat) = (info.feed, info.heartbeat);
-       return ethToToken(_amount, priceFeed, heartbeat, 18);
+       return ethToToken(_amount, priceFeed, heartbeat, IERC20Metadata(_nuAsset).decimals());
    }

    function ethToNuAssetRoundUp(
        address _nuAsset,
        uint256 _amount
    ) public view returns (uint256 EthValue) {
        require(contains(_nuAsset), "bad nuAsset");
        nuAssetInfo memory info = getNuAssetInfo(_nuAsset);
        (address priceFeed, uint128 heartbeat) = (info.feed, info.heartbeat);
-       return ethToToken(_amount, priceFeed, heartbeat, 18);
+       return ethToToken(_amount, priceFeed, heartbeat, IERC20Metadata(_nuAsset).decimals());
    }
```

[Back to Top](#summaryTable)
---

### <a id="m-02"></a>[M-02]
## **Healthy debts can be liquidated by manipulating nuAsset price**
#### https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/VaultManager.sol#L744
<br>

## Description
While calculating the LTV of a borrow, the protocol internally calls conversion functions like `numaToToken()` which goes on to [calculate the backing i.e. the LST value in ETH](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/VaultManager.sol#L744) using `EthBalance - synthValueInEth` where `synthValueInEth` is based on oracle price of the nuAsset (synthetic asset) fetched via [getTotalSynthValueEth()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/VaultManager.sol#L703). This gives rise to the following attack vector:

1. Alice buys a large quantity of NUMA by depositing rEth in a vault.
2. Bob buys some NUMA by depositing rEth in the vault.
3. Bob mints cNuma by depositing NUMA in a lending contract.
4. Against his cNuma, Bob borrows some rEth at LTV 78%. Max allowed is say, 80%. At this time let's say there are no synthetics minted.
5. Alice commences her attack. She mints considerable nuAssets using her NUMA. The NUMA/rEth rate isn't drastically effected yet.
6. Alice manipulates oracle rate of nuAsset by purchasing a huge amount of it on a LP pool. She can use a flash loan for funding. Even if TWAP prices are used, the impact is dampened but not mitigated.
    - It's important to note that `getTotalSynthValueEth()` is the total value of all nuAssets. The attack may even work better if multiple nuAssets exist. Now Alice can manipulate & hike the price of each nuAsset just enough to remain within limits (bypassing [checks](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaOracle.sol#L292-L310) related to [maxSpotOffset](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaOracle.sol#L26-L27)) and achieve an overall increase in `synthValueInEth`.
7. This causes `EthBalance - synthValueInEth` to reduce considerably and hence rEth becomes significantly expensive than before when compared to NUMA.
8. The rEth borrowed by Bob now is deemed to have a much higher value and pushes his LTV beyond the acceptable 80%.
9. Alice liquidates Bob and is free to use the `flashLoan = true` route while doing so. She pockets the profit of liquidation.
10. Alice sells the previously purchased nuAsset back on LP pool and returns any flash loans she may have taken, pushing oracle rates back down to normal.
11. She can now burn her vault nuAssets too and get back NUMA. She may choose to also sell this NUMA and get rEth back from the vault now.

## Impact
Healthy debts can be manipulated to be made unhealthy and liquidated by a malicious liquidator.

## Recommendation 
Couple of approaches the protocol could take:
- Add circuit breakers that prevent liquidations if synthetic prices move beyond certain thresholds in a short time period
- Implement time delays before liquidations are possible if LTV has moved up more than a threshold % within a short time

[Back to Top](#summaryTable)
---

### <a id="m-03"></a>[M-03]
## **Liquidator may end up receiving no seizeTokens and lose all repayAmount**
#### https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/CToken.sol#L1008-L1009
<br>

## Summary
During liquidation the protocol calculates the amount of cTokens to seize or `seizeTokens` like [here](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/CToken.sol#L1008-L1009). There is however 
- no check throughout the liquidation flow to verify if `seizeTokens > 0`. 
    - Note that `seizeTokens = 0` is possible because the calculation rounds down in favour of the borrower, not the liquidator [here](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L1458) and [here](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L1510).
- no slippage control param (for example, minAmountExpected) which the liquidator can specify while calling the liquidate functions.

This can result in the liquidator ending up paying `repayAmount` and reducing the borrower's debt but not receiving anything in return.

## Description
Imagine the following:
1. Numa per cNuma = 2e8 (as per current rates observable in tests)
2. rEth per Numa = 1_000_000e18 (Numa has appreciated considerably in future)
3. Bob has borrowed `0.18e18 rEth`. This is equivalent to `18e10 Numa` which turns out to be `900 cNuma`
4. Alice wants to liquidate Bob's unhealthy debt and calls `liquidateBadDebt()` with `_percentagePosition1000` as `1`. This denotes `0.1%` of debt to be liquidated. Note that this works similarly in a regular non-bad debt (just shortfall) situation too - the calls to the other liquidate functions need a repayAmount param instead of a percentage param, based on which a ratio is calculated and `seizeTokens` are calculated ( _See inside [liquidateCalculateSeizeTokens()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L1510)_ ).
5. rEth amount equal to `0.18e15` is pulled from Alice's account to commence liquidation
6. In return [she gets](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L1458): `seizeTokens = 1 * 900 / 1000 = 0`.
7. No further checks revert due to this and the liquidation flow concludes.

Note that such a situation can happen inadvertently during periods of high volatility too. Alice may have calculated correctly prior to sending her tx so that she receives non-zero seizeTokens but by the time the tx executes, price movement could well rob her of any returns.

## Impact
Liquidator can end up paying `repayAmount` and reducing the borrower's debt but not receiving anything in return.

## Recommendation 
- Add verification for `seizeTokens > 0`. 
- Add a slippage control param (for example, minAmountExpected) which a liquidator can specify while calling the liquidate functions.

[Back to Top](#summaryTable)
---

### <a id="m-04"></a>[M-04]
## **Removal of closeFactorMantissa & maxClose constraints in deprecated markets allows attack vector to worsen protocol's health**
#### https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L579-L583
<br>

## Summary
In a deprecated market, the `closeFactorMantissa` & `maxClose` constraints are not applied and the [only check active is](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L579-L583) whether `borrowBalance >= repayAmount`. This allows the following attack vectors:

1. Malicious user partially liquidates a shortfall debt position just enough to push it into a `badDebt` position.

2. Malicious user partially liquidates a shortfall debt position just enough to leave behind dust amount of unhealthy debt which would be unprofitable for others to liquidate due to gas costs, specially on chains like Ethereum.

## Description
Here are the flows:

### Attack Vector 1 ( _push into badDebt territory_ )
**Setup:**
- Bob has borrowed 40 ETH worth against collateral of 50 ETH worth. 85% LTV is allowed.
- Liquidation incentive is 12%.
- Price movement causes Bob borrow to be worth 45 ETH, making it eligible for liquidation by Alice.

1. **IF** `closeFactorMantissa` of `0.9e18` or `90%` was applicable here, Alice would have at best:
    - Liquidated a max `90% * 45 = 40.5 ETH` of debt.
    - Received `12% * 40.5 = 4.86 ETH` of liquidation incentive.
    - Remaining debt = `45 - 40.5 = 4.5 ETH` and remaining collateral = `50 - 40.5 - 4.86 = 4.64 ETH`. 
    - New LTV = `100 * 4.5 / 4.64 = 96.98%`. 
    - Also, the remaining debt is less than the [minBorrowAmountAllowPartialLiquidation limit](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L85) of `10 ETH` which will force full liquidation for the next liquidation attempt. ( _There is a different bug which won't allow the next liquidation attempt but that is unrelated to the current issue_ ).
    - No attack vector here.

2. However `closeFactorMantissa` constraint is not applicable in a deprecated market. So Alice does this:
    - Liquidates `44 ETH` of debt.
    - Receives `12% * 44 = 5.28 ETH` of liquidation incentive.
    - Remaining debt = `45 - 44 = 1 ETH` and remaining collateral = `50 - 44 - 5.28 = 0.72 ETH`. 
    - New LTV = `100 * 1 / 0.72 = 138.89%`. 
    - We moved into `badDebt` territory. No incentive for liquidators now to liquidate this.

### Attack Vector 2 ( _leave behind unhealthy dust amount of debt_ )
**Setup:**
- Bob has borrowed 9 ETH worth against collateral of 11 ETH worth. 85% LTV is allowed.
- Liquidation incentive is 1%.
- Price movement causes Bob borrow to be worth `10.8890888899 ETH`, and collateral to be worth `10.998 ETH` making it eligible for liquidation by Alice.

1. Alice liquidates `10.8888889 ETH` worth of debt. She receives `10.8888889 * 1.01 = 10.997777789 ETH` worth of collateral.
2. Remaining debt = `10.998 - 10.8888889 = 0.000199989900000475 ETH` and remaining collateral = `10.998 - 10.997777789 = 0.000222211000000527 ETH`. 
3. New LTV = `100 * 0.000199989900000475 / 0.000222211000000527 = 90%`. 

The remaining collateral is just too low to be profitable after gas costs.

## Impact
In the end these unhealthy low value debts & bad debts will never get liquidated, worsening protocol health. The protocol's goal of clearing a deprecated market is also not achieved.

## Mitigation
Add these 2 checks:
1. If upon partial liquidation, leftover debt is in the `badDebt` territory, then it should revert and only allow full liquidation albeit with a reduced liquidation incentive.
2. If upon partial liquidation, leftover debt is less than `minLeftoverDebtAllowed` (some small value decided by the protocol) then it should revert.

[Back to Top](#summaryTable)
---

### <a id="m-05"></a>[M-05]
## **Interest charged to borrower even when protocol is paused**
#### https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/CToken.sol#L450-L462
<br>

## Description
[accrueInterest()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/CToken.sol#L450-L462) only takes into account the `blockDelta` which is `currentBlockNumber - accrualBlockNumberPrior` for calculating the interest payable by a borrower. It unfairly includes any duration of time for which the protocol was paused when the borrower couldn't have repaid even if they wanted to.

- Let's say Alice borrowed on a leveraged strategy on Monday at 2:00 PM
- At 2:05 PM the protocol gets paused
- Alice is now unable to repay until the protocol is unpaused since the call to `closeLeverageStrategy()` [internally calls](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/CNumaToken.sol#L322) `vault.repayLeverage(true)` and [repayLeverage is protected](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L1267) by the `whenNotPaused` modifier.
- According to the code, interest continues accruing during this period
- When unpaused on Tuesday at 2:05 PM, Alice would owe a full day's interest

**_NOTE: It's interesting to observe_** that while `closeLeverageStrategy()` has `whenNotPaused`, the regular [repayBorrow()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/CErc20.sol#L104) is allowed to operate even when the protocol is paused, showing the inconsistency in code implementation.

## Impact
Borrowers unfairly pay interest even for the paused duration when they did not have the power to repay and close their debts.

## Mitigation
Interest accrual should be suspended during paused periods.

[Back to Top](#summaryTable)
---

### <a id="m-06"></a>[M-06]
## **closeLeverageStrategy() is not allowed during vault's paused state but repayBorrow() is**
#### https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L1267
<br>

## Description
It's the protocol's prerogative to take the design call whether repayment of borrows should be allowed or not when the vault is paused. The protocol could either say:
1. It's allowed because we want borrowers to be able to close their risk exposure and threat of liquidation OR
2. It's not allowed because protocols are paused in emergency situations when a high risk vulnerability has been uncovered and hence we do not want any balance changes to happen due to user transactions right now.

It's interesting to observe however that:
1. `closeLeverageStrategy()` is _not allowed_ during a paused state. It [internally calls](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/CNumaToken.sol#L322) `vault.repayLeverage(true)` and [repayLeverage is protected](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L1267) by the `whenNotPaused` modifier.
2. The regular [repayBorrow()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/CErc20.sol#L104) _is allowed_ to operate even when the protocol is paused, showing the inconsistency in code implementation. The PoC section shows a test demonstrating this.

## Impact
Irrespective of whether the protocol's design decision is the first or the second, the inconsistency in the two repay functions causes either the potential loss of funds to the borrowers, or the risk of protocol exploit during the paused state.

## Proof of Concept
Add the test inside `Vault.t.sol` and see `repayBorrow()` pass even when the vault is paused:
```js
    function test_repayDuringPause() public {
        uint borrowAmount = 0.1 ether;
        deal({token: address(rEth), to: userA, give: 10000 ether});
        vm.prank(deployer);
        comptroller._setCollateralFactor(cNuma, 1 ether);
        
        // First approve and enter the NUMA market
        vm.startPrank(userA);
        address[] memory t = new address[](1);
        t[0] = address(cNuma);
        comptroller.enterMarkets(t);

        // Buy some NUMA
        rEth.approve(address(vault), 10 ether);
        uint numas = vault.buy(10 ether, 0, userA);

        // Deposit enough NUMA as collateral
        numa.approve(address(cNuma), numas);
        cNuma.mint(numas);

        // Borrow rEth
        uint balanceBefore = rEth.balanceOf(userA);
        cReth.borrow(borrowAmount);
        assertEq(rEth.balanceOf(userA) - balanceBefore, borrowAmount, "Borrow failed");

        // Get current borrow balance 
        uint borrowBalance = cReth.borrowBalanceCurrent(userA);
        assertGt(borrowBalance, 0, "No borrow balance");
        vm.stopPrank();

        // Pause the vault
        vm.prank(deployer);
        vault.pause();

        // Try to repay while paused
        vm.startPrank(userA);
        rEth.approve(address(cReth), borrowBalance);
        cReth.repayBorrow(borrowBalance);  // <----------------- @audit : does not revert !

        vm.stopPrank();
    }
```

## Mitigation
Either both functions' call shouldn't be allowed, or both should be.

## Note
It might be the case that `closeLeverageStrategy()` should perhaps be allowed even in paused state as per the [following comment](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/ComptrollerStorage.sol#L76) present inside `ComptrollerStorage.sol`:
> Actions which allow users to remove their own assets cannot be paused.

[Back to Top](#summaryTable)
---

### <a id="m-07"></a>[M-07]
## **Deprecated markets allow profitable exploitation of bad debt liquidations**
#### https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L579-L583
<br>

## Summary
When markets are deprecated in the protocol, bad debt positions can be liquidated using regular liquidation functions instead of the dedicated bad debt liquidation path. This bypasses important safeguards and allows liquidators to extract a profit, worsening the protocol's position even further.

**_It's important to note_** that in a deprecated market even a healthy borrow position can be liquidated. That situation _could_ be attributed to user error as it may be reasonable to assume that the protocol would give enough prior warnings of the event so that users can close their positions & withdraw their deposits. 
BadDebt borrowers however would've no such incentive to close their position and hence the current vulnerability exists, exacerbating the harm to the protocol health.

## Description
The protocol provides two distinct liquidation paths:

1. Regular liquidation - Used for positions with shortfall but sufficient collateral value. This provides liquidators with a [liquidation incentive multiplier](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L1489) on the collateral they receive:
```js
    function liquidateCalculateSeizeTokens(
        address cTokenBorrowed,
        address cTokenCollateral,
        uint actualRepayAmount
    ) external view returns (uint, uint) {
        ....
        ....

        /*
         * Get the exchange rate and calculate the number of collateral tokens to seize:
@--->    *  seizeAmount = actualRepayAmount * liquidationIncentive * priceBorrowed / priceCollateral
         *  seizeTokens = seizeAmount / exchangeRate
         *   = actualRepayAmount * (liquidationIncentive * priceBorrowed) / (priceCollateral * exchangeRate)
         */

        ....
        ....
    }
```

2. Bad debt liquidation - Used when collateral value is less than the borrowed amount. This uses a simpler [percentage-based calculation](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L1458) with **no additional** `liquidationIncentive`:
```js
    function liquidateBadDebtCalculateSeizeTokensAfterRepay(
        address cTokenCollateral,
        address borrower,
        uint percentageToTake
    ) external view override returns (uint, uint) {
        /*
         * Get the exchange rate and calculate the number of collateral tokens to seize:
         * for bad debt liquidation, we take % of amount repaid as % of collateral seized
         *  seizeAmount = (repayAmount / borrowBalance) * collateralAmount
         *  seizeTokens = seizeAmount / exchangeRate
         *
         */

        (, uint tokensHeld, , ) = CToken(cTokenCollateral).getAccountSnapshot(
            borrower
        );
@--->   uint seizeTokens = (percentageToTake * tokensHeld) / (1000);
        return (uint(Error.NO_ERROR), seizeTokens);
    }
```


However, when a market is deprecated (collateralFactor = 0, borrowing paused, reserveFactor = 100%), the code only checks this inside [liquidateBorrowAllowed()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L579-L583):
```js
        if (isDeprecated(CToken(cTokenBorrowed))) {
            require(
@--->           borrowBalance >= repayAmount,
                "Can not repay more than the total borrow"
            );
        }
```

The liquidator has no need to go through a path which internally calls [liquidateBadDebtAllowed()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L620).
This allows bad debt positions to be liquidated using the regular liquidation path (via `liquidateNumaBorrower()` or `liquidateLstBorrower()`) that includes the liquidation incentive multiplier. Thus liquidator can provide a lower `repayAmount` than the collateral is worth and end up receiving a profit, worsening the protocol's health even further.

## Proof of Concept
Run the following test with `FOUNDRY_PROFILE=lite forge test --mt test_deprecatedMarketLiquidation -vv` to see the following output:
<details>
<summary>
Click to view
</summary>

1. First, add a console statement inside `liquidateLstBorrower()` for easier monitoring:
```diff
    function liquidateLstBorrower(
        address _borrower,
        uint _lstAmount,
        bool _swapToInput,
        bool _flashloan
    ) external whenNotPaused notBorrower(_borrower) {
        // < existing code... >
        ...
        ...

        if (_swapToInput) {
            // sell numa to lst
            uint lstReceived = NumaVault(address(this)).sell(
                receivedNuma,
                lstAmount,
                address(this)
            );

            uint lstLiquidatorProfit = lstReceived - lstAmount;

            // cap profit
            if (lstLiquidatorProfit > maxLstProfitForLiquidations)
                lstLiquidatorProfit = maxLstProfitForLiquidations;

            uint lstToSend = lstLiquidatorProfit;
            if (!_flashloan) {
                // send profit + input amount
                lstToSend += lstAmount;
            }
            // send profit
            SafeERC20.safeTransfer(IERC20(lstToken), msg.sender, lstToSend);
        } else {
            uint numaProvidedEstimate = vaultManager.tokenToNuma(
                lstAmount,
                last_lsttokenvalueWei,
                decimals,
                criticalScaleForNumaPriceAndSellFee
            );
            uint maxNumaProfitForLiquidations = vaultManager.tokenToNuma(
                maxLstProfitForLiquidations,
                last_lsttokenvalueWei,
                decimals,
                criticalScaleForNumaPriceAndSellFee
            );

            uint numaLiquidatorProfit;
            // we don't revert if liquidation is not profitable because it might be profitable
            // by selling lst to numa using uniswap pool
            if (receivedNuma > numaProvidedEstimate) {
                numaLiquidatorProfit = receivedNuma - numaProvidedEstimate;
            }

            uint vaultProfit;
            if (numaLiquidatorProfit > maxNumaProfitForLiquidations) {
                vaultProfit =
                    numaLiquidatorProfit -
                    maxNumaProfitForLiquidations;
            }
+           console2.log("\n Liquidator's NUMA Profit =", (numaLiquidatorProfit - vaultProfit) / 1e18, "ether");
            uint numaToSend = receivedNuma - vaultProfit;
            // send to liquidator
            SafeERC20.safeTransfer(
                IERC20(address(numa)),
                msg.sender,
                numaToSend
            );

            // AUDITV2FIX: excess vault profit numa is burnt
            if (vaultProfit > 0) numa.burn(vaultProfit);
        }
        endLiquidation();
    }
```

2. Now add this test inside `Vault.t.sol`:
```js
    function test_deprecatedMarketLiquidation() public {
        uint funds = 100 ether;
        uint borrowAmount = 80 ether; 

        deal({token: address(rEth), to: userA, give: funds * 2});
        
        // First approve and enter the NUMA market
        vm.startPrank(userA);
        address[] memory t = new address[](1);
        t[0] = address(cNuma);
        comptroller.enterMarkets(t);

        // Buy some NUMA
        rEth.approve(address(vault), funds);
        uint numas = vault.buy(funds, 0, userA);

        // Deposit enough NUMA as collateral
        uint cNumaBefore = cNuma.balanceOf(userA);
        numa.approve(address(cNuma), numas);
        cNuma.mint(numas);
        uint cNumas = cNuma.balanceOf(userA) - cNumaBefore;
        emit log_named_decimal_uint("Numas deposited =", numas, 18);
        emit log_named_decimal_uint("cNumas minted   =", cNumas, 18);

        // Borrow rEth
        uint balanceBefore = rEth.balanceOf(userA);
        cReth.borrow(borrowAmount);

        // Get current borrow balance 
        uint borrowBalance = cReth.borrowBalanceCurrent(userA);

        emit log_named_decimal_uint("borrowBalance befor =", borrowBalance, 18); 
        vm.stopPrank();

        vm.startPrank(deployer);
        // make the borrow a bad-debt
        vaultManager.setSellFee(0.5 ether); // 50%
        (, , , uint badDebt) = comptroller.getAccountLiquidityIsolate(userA, cNuma, cReth);
        assertGt(badDebt, 0, "no bad-debt");
        emit log_named_decimal_uint("badDebt   =", badDebt, 18); 

        // deprecate the market
        assertFalse(comptroller.isDeprecated(cReth));
        comptroller._setCollateralFactor(cReth, 0);
        comptroller._setBorrowPaused(cReth, true);
        cReth._setReserveFactor(1e18);
        assertTrue(comptroller.isDeprecated(cReth));
        console2.log("market successfully deprecated");
        vm.stopPrank();

        // liquidate via "shortfall" route instead of "badDebt" route
        vm.startPrank(userB); // liquidator
        uint repay = borrowBalance / 2;
        deal({token: address(rEth), to: userB, give: repay});
        rEth.approve(address(vault), repay);
        vault.liquidateLstBorrower(userA, repay, false, false); // @audit-info : smaller `repay` avoids `LIQUIDATE_SEIZE_TOO_MUCH` by ensuring `seizeTokens < cTokenCollateral`
        console2.log("liquidated successfully");
    }
```

</details>
<br>

Output:
```text
[PASS] test_deprecatedMarketLiquidation() (gas: 1630968)
Logs:
  VAULT TEST
  Numas deposited =: 749432.837569203755749171
  cNumas minted   =: 0.003747164187846018
  borrowBalance befor =: 80.000000000000000000
  badDebt   =: 32.360563278288316029
  market successfully deprecated
  redeem? 0

 Liquidator's NUMA Profit = 78656 ether    <------------- liquidator received profit on a badDebt by calling `liquidateLstBorrower()`
  liquidated successfully
```

## Severity
Impact: High. Worsens the protocol's health even further.

Likelihood: Low/Medium. Requires an event where a market has been deprecated. 

Overall Severity: Medium

## Mitigation
Add a check that even for deprecated markets, regular (shortfall) liquidation path is not allowed for badDebt positions.

[Back to Top](#summaryTable)
---

### <a id="m-08"></a>[M-08]
## **Checks not implemented for lower & upper bounds of closeFactorMantissa**
#### https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L104-L108
<br>

## Description
`NumaComptroller.sol` [specifies that](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L104-L108):
```js
@-->    // closeFactorMantissa must be strictly greater than this value
        uint internal constant closeFactorMinMantissa = 0.05e18; // 0.05

@-->    // closeFactorMantissa must not exceed this value
        uint internal constant closeFactorMaxMantissa = 0.9e18; // 0.9
```

However this check is never implemented inside [_setCloseFactor](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L1551):
```js
    function _setCloseFactor(
        uint newCloseFactorMantissa
    ) external returns (uint) {
        // Check caller is admin
        require(msg.sender == admin, "only admin can set close factor");

        uint oldCloseFactorMantissa = closeFactorMantissa;
        closeFactorMantissa = newCloseFactorMantissa;
        emit NewCloseFactor(oldCloseFactorMantissa, closeFactorMantissa);

        return uint(Error.NO_ERROR);
    }
```

## Weaponizing the Admin's Error
Someone could put forward an argument that this is just a cosmetic bound decided somewhat arbitrarily and even if the admin sets `closeFactorMantissa` to say `1e18` or `0.9999e18`, there's no harm. In fact the current tests use `closeFactorMantissa = 1e18`!

However that's not true. The following flow shows how an attack path emerges in such a scenario:
1. Bob has `15 ether` of debt which has gone underwater. Let's assume admin has correctly set `closeFactorMantissa = 0.9e18`.
2. Alice tries to maliciously liquidate Bob partially for amount of `14.99999 ether` so that a dust amount of leftover debt remains. This is allowed by the logic [here](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L993) and [here](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L1135). As long as `current debt value > minBorrowAmountAllowPartialLiquidation`, partial liquidations are allowed. By doing so with many shortfall debts, she can flood the system with dust debts too unprofitable for anyone to liquidate, specially on a chain like Ethereum with high gas costs.
3. Alice's attempt is thwarted by the checks related to `maxClose` and `closeFactorMantissa`. She can't repay [more than 90% of the debt](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L108) in the worst case:
    - `maxClose` is [calculated by multiplying `borrowBalance` with `closeFactorMantissa`](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L605-L608)
    - [repayAmount > maxClose](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaComptroller.sol#L609-L611) is not allowed by the protocol. 
4. Attack not possible because `closeFactorMantissa = 0.9e18`. But if it were `1e18`, she **could** have repaid `14.99999 ether` and the attack works.
5. With `closeFactorMantissa = 0.9e18`, the smallest leftover debt that can remain after an attacker's attempt is around `1 ether`. This is because the protocol implements the [minBorrowAmountAllowPartialLiquidation limit](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L85) of `10 ether` and partial repayment below this limit is not possible. So even if an attacker tries the attack when `borrowBalance = 10 ether + 1`, they will be able to repay only around `9 ether` and not more.

## Impact
Admin can inadvertently set it to values outside bounds and dilute/negate the protection mechanisms in place against avoidance of dust leftover debts. This issue is contingent on an admin error hence setting severity as medium.

## Mitigation 
```diff
    function _setCloseFactor(
        uint newCloseFactorMantissa
    ) external returns (uint) {
        // Check caller is admin
        require(msg.sender == admin, "only admin can set close factor");
+       require(newCloseFactorMantissa > closeFactorMinMantissa && newCloseFactorMantissa <= closeFactorMaxMantissa, "newCloseFactorMantissa outside limits");

        uint oldCloseFactorMantissa = closeFactorMantissa;
        closeFactorMantissa = newCloseFactorMantissa;
        emit NewCloseFactor(oldCloseFactorMantissa, closeFactorMantissa);

        return uint(Error.NO_ERROR);
    }
```

[Back to Top](#summaryTable)
---

### <a id="m-09"></a>[M-09]
## **Missing validation in setSellFee() can cause sell_fee_withPID to never rebase**
#### https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/VaultManager.sol#L257-L268
<br>

## Description
[setSellFee()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/VaultManager.sol#L257-L268) has the following validation missing. Owner **should not** be allowed to set `sell_fee` less than `sell_fee_minimum` and needs to be explicitly checked, else a situation could arise where rebase of `sell_fee_withPID` is stuck:
```diff
    function setSellFee(uint _fee) external onlyOwner {
        require(_fee <= 1 ether, "fee too high");
+       require(_fee > sell_fee_minimum, "fee too low");
        sell_fee = _fee;

        // careful
        // changing sell fee will reset sell_fee scaling
        sell_fee_withPID = sell_fee;
        lastBlockTime_sell_fee = block.timestamp;
        //sell_fee_update_blocknumber = block.number;

        emit SellFeeUpdated(_fee);
    }
```

## Impact
If `sell_fee` is set below `sell_fee_minimum`, the following can happen:
    - Assume `sell_fee_minimum = 0.5 ether` ([default value](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/VaultManager.sol#L53)).
    - Owner sets `sell_fee = 0.3 ether` due to the missing aforementioned check. The function [sets `sell_fee_withPID` too to `0.3 ether`](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/VaultManager.sol#L263).
    - Inside [getSellFeeScaling()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/VaultManager.sol#L401), `currentLiquidCF < cf_liquid_severe` and we need to enter the debase branch.
    - The [code takes care of clipping](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/VaultManager.sol#L420-L422) `lastSellFee` to `sell_fee_minimum`. This results in `sell_fee_withPID` to be updated to `0.5 ether`. The value of `sell_fee` is still at `0.3 ether`.
    - Next time if `currentLiquidCF >= cf_liquid_severe` and we need to rebase, the [conditional check here](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/VaultManager.sol#L426) of `if (sell_fee_withPID < sell_fee)` is never satisfied and rebase never happens.

The only remedy now is for the owner to call `setSellFee()` again with a correct value of `_fee` so that it resets scaling.

From a financial standpoint, this means that users would be stuck getting less LST tokens than they should upon selling Numa.

## Mitigation 
```diff
    function setSellFee(uint _fee) external onlyOwner {
        require(_fee <= 1 ether, "fee too high");
+       require(_fee > sell_fee_minimum, "fee too low");
        sell_fee = _fee;

        // careful
        // changing sell fee will reset sell_fee scaling
        sell_fee_withPID = sell_fee;
        lastBlockTime_sell_fee = block.timestamp;
        //sell_fee_update_blocknumber = block.number;

        emit SellFeeUpdated(_fee);
    }
```

[Back to Top](#summaryTable)
---

### <a id="m-10"></a>[M-10]
## **Users may end up paying higher than required fees during opening & closing of leveraged strategies**
#### https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L488-L493
<br>

## Summary
[NumaLeverageVaultSwap::swap()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaLeverageVaultSwap.sol#L43) calculates buy/sell fees based on the estimated input amount rather than the actually used amount during token swaps. This leads to users paying higher fees than necessary when the actual amount needed for the swap is less than the initial estimate.

## Description
When a user initiates a leverage strategy through [CNumaToken::leverageStrategy()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/CNumaToken.sol#L141), the protocol first estimates the amount of tokens needed for the swap (`borrowAmount`) to ensure getting the desired output amount (`_borrowAmount`) after accounting for slippage. This estimate calculated by `NumaLeverageVaultSwap::getAmountIn()` [is intentionally higher](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaLeverageVaultSwap.sol#L29-L36) than likely needed as a safety margin. This and other factors give rise to the following situation:

1. The protocol [estimates required input](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/CNumaToken.sol#L193) (e.g., 31 rETH) to get desired output (e.g., 30 NUMA worth):
```js
uint borrowAmount = strat.getAmountIn(_borrowAmount, false);
```

2. This full estimated amount is [temporarily borrowed](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/CNumaToken.sol#L198) to repay the vault:
```js
borrowInternalNoTransfer(borrowAmount, msg.sender);
```

3. The [swap is executed](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/CNumaToken.sol#L208-L212):
```js
(uint collateralReceived, uint unUsedInput) = strat.swap(
    borrowAmount,
    _borrowAmount,
    false
);
```

4. `strat.swap` [internally called](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/NumaLeverageVaultSwap.sol#L62-L63) `vault.buy()` which [called `buyNoMax()`](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L445). Fee was [calculated based on the output `numaAmount`](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/NumaProtocol/NumaVault.sol#L470-L493). This value `numaAmount` after deduction of fees was returned by the function, which got stored in the variable `collateralReceived` above.


5. `CNumaToken::leverageStrategy()` then continues and checks if `collateralReceived > _borrowAmount` and [returns excess amount to the borrower](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/CNumaToken.sol#L221-L229) if true:
```js
        //refund if more collateral is received than needed
        if (collateralReceived > _borrowAmount) {
            // send back the surplus
            SafeERC20.safeTransfer(
                IERC20(underlyingCollateral),
                msg.sender,
                collateralReceived - _borrowAmount
            );
        }
```

As can be seen, while the excess `collateralReceived` itself is properly returned, the excess fees charged is not. User had to pay the fees even on the `collateralReceived - _borrowAmount` amount.

## Impact
Users pay higher fees than expected.

## Additional Impacted Area
The same issue also crops up when closing a leveraged strategy:
- **Leverage Closing**:
   - [closeLeverageStrategy()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/CNumaToken.sol#L263) --> [closeLeverageAmount()](https://github.com/NumaMoney/Numa/blob/c6476d828f556967e64410b5c11c1f2cd77220c7/contracts/lending/CNumaToken.sol#L243) --> `NumaLeverageVaultSwap::getAmountIn()` then `strat.swap()`

## Mitigation
Implement a fee refund mechanism that returns the excess fee after the `if (collateralReceived > _borrowAmount)` check.

[Back to Top](#summaryTable)
