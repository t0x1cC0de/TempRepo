# Leaderboard
[Concrete Results]()<br>

`Rank ? / ?`

# Audited Code Repo
### [Code4rena: Concrete](https://code4rena.com/audits/2024-11-concrete)
### [Github: Concrete](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/tree/main)

<br>

# <a id="summaryTable"></a>Bugs Filed & Their Status

| #      | Bug ID          | Name | URL    | Adjudged Status  |
|--------|-----------------|------|:------:|-----------------:|
| 1      | [H-01](#h-01)   | `mint()` reverts for numerous combinations of `shares_` amount and strategy allocation percentages due to incorrect rounding direction | [97](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-97) |  |
| 2      | [H-02](#h-02)   | `withdraw()` reverts while attempting to pull funds from strategies due to use of `amount_` instead of `diff` inside `_withdrawStrategyFunds()` | [193](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-193) |  |
| 3      | [H-03](#h-03)   | Incorrect `amount` transferred inside `requestToken()` if `tokenCascade[]` is involved | [296](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-296) |  |
| 4      | [H-04](#h-04)   | A replaced protect-strategy can steal all the funds from the vault | [317](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-317) | Monitor -- 2 dupes |
| 5      | [H-05](#h-05)   | Reward tokens can be stolen due to unvalidated user supplied `ctAssetToken` address in `swapTokensForReward()` | [349](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-349) |  |
| 6      | [H-06](#h-06)   | `_withdrawStrategyFunds()` reverts prematurely even if withdrawable amount in vault is sufficient | [498](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-498) |  |
| 7      | [M-01](#m-01)   | Some reward tokens once removed, can not be added again due to absence of allowance reset | [92](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-92) |  |
| 8      | [M-02](#m-02)   | `depositLimit` not respected during deposit due to incorrect logic | [94](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-94) |  |
| 9      | [M-03](#m-03)   | Funds can get stuck and can't be withdrawn entirely via `redeem()` or `withdraw()` | [152](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-152) |  |
| 10     | [M-04](#m-04)   | Call to `withdraw()` reverts due to attempt to burn more shares than issued | [187](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-187) | Monitor -- 3 dupes |
| 11     | [M-05](#m-05)   | User can escape deposit fee by making multiple small deposits due to incorrect rounding direction | [188](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-188) |  |
| 12     | [M-06](#m-06)   | ProtocolFee is charged even for the `paused` duration | [203](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-203) | Monitor -- 2 dupes |
| 13     | [M-07](#m-07)   | Tiered fee not charged for upper boundary values | [214](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-214) | Monitor -- incorrect dispute |
| 14     | [M-08](#m-08)   | Incorrect reward accounting inside `harvestRewards()` if `TokenHelper.attemptSafeTransfer()` fails | [223](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-223) | Monitor -- 9 dupes |
| 15     | [M-09](#m-09)   | No way to remove rewardToken if added twice in `rewardTokens` array during initialization | [306](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-306) |  |
| 16     | [M-10](#m-10)   | Call to `pauseAllVaults()` fails if any vault is already in paused state | [355](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-355) | Monitor -- 2 dupes |
| 17     | [M-11](#m-11)   | Token once unregistered from registry can't be ever registered again | [385](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-385) |  |
| 18     | [M-12](#m-12)   | A token can be a reward token and yet unregistered | [399](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-399) |  |
| 19     | [M-13](#m-13)   | Use of strict equality disallows adding new strategy | [420](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-420) |  |
| 20     | [M-14](#m-14)   | Claim status not updated if `_availableAssets` exactly equals requested `amount` | [566](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-566) |  |
| 21     | [M-15](#m-15)   | Lack of slippage protection & deadline can lead to loss | [810](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-810) |  |

<br>
<br>

## **HIGH-SEVERITY BUGS**
---
### <a id="h-01"></a>[H-01]
## **`mint()` reverts for numerous combinations of `shares_` amount and strategy allocation percentages due to incorrect rounding direction**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L322
<br>

## Description
Whenever the `shares_` converted to `assets` amount is not fully divisible by the strategy allocation percentages (which occurs for numerous combinations), there is the possibility for `mint()` function to revert with error `ERC20InsufficientBalance` due to use of rounding direction `Ceil` instead of `Floor`:
```js
  File: src/vault/ConcreteMultiStrategyVault.sol

   290:              function mint(
   291:                  uint256 shares_,
   292:                  address receiver_
   293:              ) public override nonReentrant whenNotPaused returns (uint256 assets) {
   294:                  _validateAndUpdateDepositTimestamps(receiver_);
   295:
   296:                  if (shares_ == 0) revert ZeroAmount();
   297:
   298:                  // Calculate the deposit fee in shares
   299:                  uint256 depositFee = uint256(fees.depositFee);
   300:                  uint256 feeShares = shares_.mulDiv(MAX_BASIS_POINTS, MAX_BASIS_POINTS - depositFee, Math.Rounding.Floor) -
   301:                      shares_;
   302:
   303:                  // Calculate the total assets required for the minted shares, including fees
   304:                  assets = _convertToAssets(shares_ + feeShares, Math.Rounding.Ceil);
   305:
   306:                  if (assets > maxMint(receiver_)) revert MaxError();
   307:
   308:                  // Mint shares to fee recipient and receiver
   309:                  if (feeShares > 0) _mint(feeRecipient, feeShares);
   310:                  _mint(receiver_, shares_);
   311:
   312:                  // Transfer the assets from the sender to the vault
   313:                  IERC20(asset()).safeTransferFrom(msg.sender, address(this), assets);
   314:
   315:                  // If the vault is not idle, allocate the assets into strategies
   316:                  if (!vaultIdle) {
   317:                      uint256 len = strategies.length;
   318:                      for (uint256 i; i < len; ) {
   319:                          //We control both the length of the array and the external call
   320:                          // slither-disable-next-line unused-return,calls-loop
   321:                          strategies[i].strategy.deposit(
   322:@--->                         assets.mulDiv(strategies[i].allocation.amount, MAX_BASIS_POINTS, Math.Rounding.Ceil),
   323:                              address(this)
   324:                          );
   325:                          unchecked {
   326:                              i++;
   327:                          }
   328:                      }
   329:                  }
   330:              }
```
<br>

Example:<br>
If `10` has to be allocated among 3 strategies with `33.33%` allocation each (the default value used in provided tests), the protocol calculates this as:
- Strategy1 allocation = `Ceil((10 * 3333) / 10000)` = 4. We've remaining `assets` as `10 - 4 = 6`.
- Strategy2 allocation = `Ceil((10 * 3333) / 10000)` = 4. We've remaining `assets` as `6 - 4 =  2`.
- Strategy3 allocation = `Ceil((10 * 3333) / 10000)` = 4. We've underflow due to `2 - 4 < 0`. **_Reverts_**.

## Impact
`mint()` can not be used (reverts) for numerous combinations of `shares_` amount and strategy allocation percentages.

## Tools Used
Foundry

## Proof of Concept
We will modify the existing test [test_mintRedeem()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/test/ConcreteMultiStrategyVault.t.sol#L370) to witness it fail when run with `forge test --mt test_mintRedeem -vv`:
```diff
  File: test/ConcreteMultiStrategyVault.t.sol

   370:              function test_mintRedeem() public {
   371:                  (ConcreteMultiStrategyVault newVault, ) = _createNewVault(false, false, false);
-  372:                  uint256 amount_ = 1e18;
+  372:                  uint256 amount_ = 1e10;
   373:                  uint256 jimmyShareAmount = amount_;
   374:                  asset.mint(jimmy, jimmyShareAmount);
   375:
   376:                  vm.prank(jimmy);
   377:                  asset.approve(address(newVault), jimmyShareAmount);
   378:
   379:                  vm.prank(jimmy);
   380:                  uint256 jimmyAssetAmount = newVault.mint(jimmyShareAmount, jimmy);
   381:
   382:                  assertApproxEqAbs(newVault.previewDeposit(jimmyAssetAmount), jimmyShareAmount, 1e9, "Preview Deposit");
   383:
   384:                  assertEq(newVault.balanceOf(jimmy), jimmyShareAmount, "Jimmy balance in vault");
   385:                  assertEq(newVault.totalAssets(), jimmyAssetAmount, "Total assets");
   386:              }
```
<br>

Output:
```text
Ran 1 test for test/ConcreteMultiStrategyVault.t.sol:ConcreteMultiStrategyVaultTest
[FAIL. Reason: ERC20InsufficientBalance(0xD6BbDE9174b1CdAa358d2Cf4D57D1a9F7178FBfF, 2, 4)] test_mintRedeem() (gas: 5080659)
```

The error `ERC20InsufficientBalance` shows that address `0xD6BbDE9174b1CdAa358d2Cf4D57D1a9F7178FBfF` has a balance of `2` but an attempt is being made to transfer `4`, thus causing the failure.

## Recommendation
Two changes need to be done:
1. Use `Floor` rounding direction instead of `Ceil`.
2. For the last strategy, use all of the remaining amount instead of calculating it directly.
```diff
  File: src/vault/ConcreteMultiStrategyVault.sol

   290:              function mint(
   291:                  uint256 shares_,
   292:                  address receiver_
   293:              ) public override nonReentrant whenNotPaused returns (uint256 assets) {
   294:                  _validateAndUpdateDepositTimestamps(receiver_);
   295:
   296:                  if (shares_ == 0) revert ZeroAmount();
   297:
   298:                  // Calculate the deposit fee in shares
   299:                  uint256 depositFee = uint256(fees.depositFee);
   300:                  uint256 feeShares = shares_.mulDiv(MAX_BASIS_POINTS, MAX_BASIS_POINTS - depositFee, Math.Rounding.Floor) -
   301:                      shares_;
   302:
   303:                  // Calculate the total assets required for the minted shares, including fees
   304:                  assets = _convertToAssets(shares_ + feeShares, Math.Rounding.Ceil);
   305:
   306:                  if (assets > maxMint(receiver_)) revert MaxError();
   307:
   308:                  // Mint shares to fee recipient and receiver
   309:                  if (feeShares > 0) _mint(feeRecipient, feeShares);
   310:                  _mint(receiver_, shares_);
   311:
   312:                  // Transfer the assets from the sender to the vault
   313:                  IERC20(asset()).safeTransferFrom(msg.sender, address(this), assets);
   314:
   315:                  // If the vault is not idle, allocate the assets into strategies
   316:                  if (!vaultIdle) {
+  316:                  uint256 assetsAllocated = 0;
   317:                      uint256 len = strategies.length;
-  318:                      for (uint256 i; i < len; ) {
+  318:                      for (uint256 i; i < len - 1; ) {
   319:                          //We control both the length of the array and the external call
   320:                          // slither-disable-next-line unused-return,calls-loop
   321:                          strategies[i].strategy.deposit(
-  322:                              assets.mulDiv(strategies[i].allocation.amount, MAX_BASIS_POINTS, Math.Rounding.Ceil),
+  322:                              assets.mulDiv(strategies[i].allocation.amount, MAX_BASIS_POINTS, Math.Rounding.Floor),
   323:                              address(this)
   324:                          );
+  324:                          assetsAllocated += assets.mulDiv(strategies[i].allocation.amount, MAX_BASIS_POINTS, Math.Rounding.Floor);
   325:                          unchecked {
   326:                              i++;
   327:                          }
   328:                      }
+  328:                      strategies[len - 1].strategy.deposit(assets - assetsAllocated, address(this)); // remaining assets go to the last strategy
   329:                  }
   330:              }
```

[Back to Top](#summaryTable)
---

### <a id="h-02"></a>[H-02]
## **`withdraw()` reverts while attempting to pull funds from strategies due to use of `amount_` instead of `diff` inside `_withdrawStrategyFunds()`**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L464
<br>

## Description & Impact
Consider the user flow:
- Vault exists with a single strategy with 100% allocation
- Vault set to idle initially
- Hazel Deposits 1 ether (stays in vault due to idle; is not allocated to strategy)
- Vault set to non-idle mode
- Hazel Deposits 2 ether (100% goes to strategy)
- Hazel attempts to withdraw 2.5 ether
<br>
Expected protocol behaviour:
- `float` i.e. the vault balance outside of strategies is calculated as `1 ether`
- Hazel receives `1 ether` from `float` balance, making it `0`.
- Hazel receives remaining `1.5 ether` from strategy balance, making the remaining fund balance there `0.5 ether`.
<br>
Current (incorrect) behaviour:
- `float` i.e. the vault balance outside of strategies is calculated as `1 ether`
- Protocol attempts to withdraw entire `2.5 ether` from the strategy as per [this line of code](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L464), thus reverting on [L471](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L471).
<br>
Note that although the current example makes use of a toggle between vault's idle & non-idle state to achieve the configuration where some funds are in float while others in strategy, it can happen under other circumstances too such as addition/removal of strategies at some point of time during the course of vault's operation.

```js
  File: src/vault/ConcreteMultiStrategyVault.sol

   444:              function _withdrawStrategyFunds(uint256 amount_, address receiver_) private {
   445:                  IERC20 asset_ = IERC20(asset());
   446:                  //Not in a loop
   447:                  //slither-disable-next-line calls-loop
   448:                  uint256 float = asset_.balanceOf(address(this));
   449:                  // If there is enough in the vault, send the full amount
   450:                  if (amount_ <= float) {
   451:                      asset_.safeTransfer(receiver_, amount_);
   452:                  } else {
   453:                      uint256 diff = amount_ - float;
   454:                      uint256 totalWithdrawn = 0;
   455:                      uint256 len = strategies.length;
   456:                      for (uint256 i; i < len; ) {
   457:                          Strategy memory strategy = strategies[i];
   458:                          //We control both the length of the array and the external call
   459:                          //slither-disable-next-line calls-loop
   460:                          uint256 withdrawable = strategy.strategy.previewRedeem(strategy.strategy.balanceOf(address(this)));
   461:                          if (diff.mulDiv(strategy.allocation.amount, MAX_BASIS_POINTS, Math.Rounding.Ceil) > withdrawable) {
   462:                              revert InsufficientFunds(strategy.strategy, diff * strategy.allocation.amount, withdrawable);
   463:                          }
   464:@--->                     uint256 amountToWithdraw = amount_.mulDiv(   // @audit-issue : should be `diff.mulDiv` instead of `amount_.mulDiv`
   465:                              strategy.allocation.amount,
   466:                              MAX_BASIS_POINTS,
   467:                              Math.Rounding.Ceil
   468:                          );
   469:                          //We control both the length of the array and the external call
   470:                          //slither-disable-next-line unused-return,calls-loop
   471:@--->                     strategy.strategy.withdraw(amountToWithdraw, receiver_, address(this));
   472:                          totalWithdrawn += amountToWithdraw;
   473:                          unchecked {
   474:                              i++;
   475:                          }
   476:                      }
   477:                      // If enough wasn't drawn out, use the float (this is due to dust)
   478:                      if (totalWithdrawn < amount_ && amount_ - totalWithdrawn <= float) {
   479:                          asset_.safeTransfer(receiver_, amount_ - totalWithdrawn);
   480:                      }
   481:                  }
   482:              }
```

## Proof of Concept
Add this test inside `test/ConcreteMultiStrategyVault.t.sol` to see it fail when run with `forge test --mt test_t0x1c_withdraw_from_strategy -vv`:
```js
    function test_t0x1c_withdraw_from_strategy() public {
        (ConcreteMultiStrategyVault newVault, Strategy[] memory strats) = _createNewVault(false, false, false);
        
        // Set allocation to 100% for first strategy, remove others
        vm.startPrank(admin);
        newVault.removeStrategy(2);  // Remove last strategy first
        newVault.removeStrategy(1);  // Remove second strategy
        Allocation[] memory allocations = new Allocation[](1);
        allocations[0] = Allocation({index: 0, amount: 10000});  // 100%
        newVault.changeAllocations(allocations, false);
        
        // Set vault to idle
        newVault.toggleVaultIdle();
        vm.stopPrank();

        // Mint test assets
        asset.mint(hazel, 5 ether);
        vm.startPrank(hazel);
        asset.approve(address(newVault), 5 ether);

        // 1. First deposit (1 ether) while vault is idle
        console.log("\nDepositing 1 ether while vault idle...");
        newVault.deposit(1 ether, hazel);
        // Verify deposit
        assertEq(asset.balanceOf(address(newVault)), 1 ether, "Vault balance incorrect after idle deposit");
        assertEq(asset.balanceOf(address(strats[0].strategy)), 0, "Strategy balance non-zero after idle deposit");
        
        // Take vault out of idle
        vm.stopPrank();
        vm.prank(admin);
        newVault.toggleVaultIdle();
        
        // 2. Second deposit (2 ether) while vault is not idle
        vm.startPrank(hazel);
        console.log("\nDepositing 2 ether while vault not idle...");
        newVault.deposit(2 ether, hazel);
        
        // Verify deposits went to strategy
        assertEq(asset.balanceOf(address(newVault)), 1 ether, "Vault balance incorrect after non-idle deposit");
        assertEq(asset.balanceOf(address(strats[0].strategy)), 2 ether, "Strategy balance incorrect after non-idle deposit");
        assertEq(newVault.totalAssets(), 3 ether, "Total assets incorrect");

        // 3. Withdraw 2.5 ether
        console.log("\nWithdrawing 2.5 ether...");
        uint256 preWithdrawBalance = asset.balanceOf(hazel);
        uint256 sharesToBurn = newVault.withdraw(2.5 ether, hazel, hazel);
        console.log("Shares burned:", sharesToBurn);
        
        // Verify withdrawal
        assertEq(
            asset.balanceOf(hazel) - preWithdrawBalance, 
            2.5 ether, 
            "Hazel received incorrect withdrawal amount"
        );
        assertEq(asset.balanceOf(address(strats[0].strategy)), 0.5 ether, "Strategy balance incorrect after withdraw");
        assertEq(newVault.totalAssets(), 0.5 ether, "Total assets after withdrawal incorrect");
        vm.stopPrank();
    }
```

## Mitigation
```diff
  File: src/vault/ConcreteMultiStrategyVault.sol

   444:              function _withdrawStrategyFunds(uint256 amount_, address receiver_) private {
   445:                  IERC20 asset_ = IERC20(asset());
   446:                  //Not in a loop
   447:                  //slither-disable-next-line calls-loop
   448:                  uint256 float = asset_.balanceOf(address(this));
   449:                  // If there is enough in the vault, send the full amount
   450:                  if (amount_ <= float) {
   451:                      asset_.safeTransfer(receiver_, amount_);
   452:                  } else {
   453:                      uint256 diff = amount_ - float;
   454:                      uint256 totalWithdrawn = 0;
   455:                      uint256 len = strategies.length;
   456:                      for (uint256 i; i < len; ) {
   457:                          Strategy memory strategy = strategies[i];
   458:                          //We control both the length of the array and the external call
   459:                          //slither-disable-next-line calls-loop
   460:                          uint256 withdrawable = strategy.strategy.previewRedeem(strategy.strategy.balanceOf(address(this)));
   461:                          if (diff.mulDiv(strategy.allocation.amount, MAX_BASIS_POINTS, Math.Rounding.Ceil) > withdrawable) {
   462:                              revert InsufficientFunds(strategy.strategy, diff * strategy.allocation.amount, withdrawable);
   463:                          }
-  464:                          uint256 amountToWithdraw = amount_.mulDiv(
+  464:                          uint256 amountToWithdraw = diff.mulDiv(
   465:                              strategy.allocation.amount,
   466:                              MAX_BASIS_POINTS,
   467:                              Math.Rounding.Ceil
   468:                          );
   469:                          //We control both the length of the array and the external call
   470:                          //slither-disable-next-line unused-return,calls-loop
   471:                          strategy.strategy.withdraw(amountToWithdraw, receiver_, address(this));
   472:                          totalWithdrawn += amountToWithdraw;
   473:                          unchecked {
   474:                              i++;
   475:                          }
   476:                      }
-  477:                      // If enough wasn't drawn out, use the float (this is due to dust)
+  477:                      // Use the float for any remaining amount
   478:                      if (totalWithdrawn < amount_ && amount_ - totalWithdrawn <= float) {
   479:                          asset_.safeTransfer(receiver_, amount_ - totalWithdrawn);
   480:                      }
   481:                  }
   482:              }
```

[Back to Top](#summaryTable)
---

### <a id="h-03"></a>[H-03]
## **Incorrect `amount` transferred inside `requestToken()` if `tokenCascade[]` is involved**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/claimRouter/ClaimRouter.sol#L207
<br>

## Description & Impact
When [requestToken()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/claimRouter/ClaimRouter.sol#L194) is called with a `tokenAddress` and none of the vaults with this token have a protection strategy configured, alternate token is used for executing the borrow claim. This alternate token address is picked up from the `tokenCascade` array. For example, `tokenAddress` could be WBTC while the chosen `tokenCascade[i]` could be DAI. The `amount_` initially supplied as a parameter in the function call assumes WBTC. Hence, this needs to be converted to DAI amount before calling `_getStrategy()` on [L211](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/claimRouter/ClaimRouter.sol#L211) and eventually `executeBorrowClaim()` on [L225](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/claimRouter/ClaimRouter.sol#L225).
The current logic however only converts it into an USD amount instead of a DAI amount on [L207](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/claimRouter/ClaimRouter.sol#L207). Thus the user may get a significantly higher or lower amount. 

```js
  File: src/claimRouter/ClaimRouter.sol

   186:              function requestToken(
   187:                  VaultFlags,
   188:                  address tokenAddress,
   189:                  uint256 amount_,
   190:                  address payable userBlueprint
   191:              ) external onlyRole(BLUEPRINT_ROLE) {
   192:                  uint256 amount = amount_;
   193:                  (address protectionStrat, bool requiresFunds) = _getStrategy(tokenAddress, amount);
   194:                  if (protectionStrat == address(0x0)) {
   195:                      //iterates over array
   196:                      uint256 len = tokenCascade.length;
   197:                      for (uint256 i; i < len; ) {
   198:                          //We avoid using the same token as the one that failed
   199:                          if (tokenAddress == tokenCascade[i]) {
   200:                              unchecked {
   201:                                  i++;
   202:                              }
   203:                              continue;
   204:                          }
   205:                          //TODO change amount
   206:
   207:@--->                     amount = _convertFromTokenToStable(tokenAddress, amount_);
   208:
   209:                          //We control both the length of the array and the external call
   210:                          // slither-disable-next-line calls-loop
   211:@--->                     (protectionStrat, requiresFunds) = _getStrategy(tokenCascade[i], amount);
   212:                          if (protectionStrat != address(0x0)) {
   213:                              break;
   214:                          }
   215:                          unchecked {
   216:                              i++;
   217:                          }
   218:                      }
   219:                  }
   220:
   221:                  if (protectionStrat == address(0x0)) {
   222:                      revert NoProtectionStrategiesFound();
   223:                  }
   224:                  emit ClaimRequested(protectionStrat, amount, IProtectStrategy(protectionStrat).asset(), userBlueprint);
   225:@--->             IProtectStrategy(protectionStrat).executeBorrowClaim(amount, userBlueprint);
   226:              }
```

## Mitigation
Do the correct rate conversion by adding the following line:
```diff
  File: src/claimRouter/ClaimRouter.sol

   186:              function requestToken(
   187:                  VaultFlags,
   188:                  address tokenAddress,
   189:                  uint256 amount_,
   190:                  address payable userBlueprint
   191:              ) external onlyRole(BLUEPRINT_ROLE) {
   192:                  uint256 amount = amount_;
   193:                  (address protectionStrat, bool requiresFunds) = _getStrategy(tokenAddress, amount);
   194:                  if (protectionStrat == address(0x0)) {
   195:                      //iterates over array
   196:                      uint256 len = tokenCascade.length;
   197:                      for (uint256 i; i < len; ) {
   198:                          //We avoid using the same token as the one that failed
   199:                          if (tokenAddress == tokenCascade[i]) {
   200:                              unchecked {
   201:                                  i++;
   202:                              }
   203:                              continue;
   204:                          }
   205:                          //TODO change amount
   206:
   207:                          amount = _convertFromTokenToStable(tokenAddress, amount_);
+  207:                          amount = _convertFromStableToToken(tokenCascade[i], amount);
   208:
   209:                          //We control both the length of the array and the external call
   210:                          // slither-disable-next-line calls-loop
   211:                          (protectionStrat, requiresFunds) = _getStrategy(tokenCascade[i], amount);
   212:                          if (protectionStrat != address(0x0)) {
   213:                              break;
   214:                          }
   215:                          unchecked {
   216:                              i++;
   217:                          }
   218:                      }
   219:                  }
   220:
   221:                  if (protectionStrat == address(0x0)) {
   222:                      revert NoProtectionStrategiesFound();
   223:                  }
   224:                  emit ClaimRequested(protectionStrat, amount, IProtectStrategy(protectionStrat).asset(), userBlueprint);
   225:                  IProtectStrategy(protectionStrat).executeBorrowClaim(amount, userBlueprint);
   226:              }
```

[Back to Top](#summaryTable)
---

### <a id="h-04"></a>[H-04]
## **A replaced protect-strategy can steal all the funds from the vault**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/libraries/MultiStrategyVaultHelper.sol#L237-L239
<br>

## Description
Calling the [addStrategy()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L810) function with `replace_ = true` as a param lets the owner replace a vault's strategy. This internally calls [MultiStrategyVaultHelper::addOrReplaceStrategy()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/libraries/MultiStrategyVaultHelper.sol#L204) which has a logic flaw. 
If a vault's protect-strategy is **replaced with a non-protect strategy**, then the `protectStrategy` [variable which is meant to point to the address of the protect strategy contract](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L42) is not reset to `address(0)` and continues to erroneously point to the old address.

## Impact
The impact seems to be high since vaults with a protect-strategy are checked across the protocol. The above vault will now be incorrectly included in the list while executing:
- `_getStrategy()` : supposed to be used to find the best vault with protection-strategy to withdraw from. `_getStrategy()` is internally called by:
  - `requestToken()` : Used to request assets from the vault and execute a borrow claim through the protection strategy.
- `_addTokensToStrategy()` : only supposed to consider vaults with a protection-strategy. This messes up the following functions which internally call `_addTokensToStrategy()`:
  - `addRewards()`
  - `repay()`
- `_getTokenTotalBorrowDebt` : `DebtVaults.vaultsWithProtect` count is incorrectly calculated. `_getTokenTotalBorrowDebt()` is internally called by `_addTokensToStrategy()` mentioned above.
- `getTokenTotalAvaliableForProtection` : incorrectly calculates the total avalaible tokens for protection across all vaults.

**Importantly, it violates the security assumption of trusted roles**. A function having the modifier `onlyProtect` like [requestFunds()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L1098) can still be called by this protect-strategy contract in spite of the fact that it has been removed from the vault. This can be used to pull funds from every other strategy in the vault, including the float amount.

## Proof of Concept
Add the following inside `test/ConcreteMultiStrategyVault.t.sol` to see it pass when run with `forge test --mt test_t0x1c_replaceProtectStrategyWithNonProtect -vv`:
```js
    function test_t0x1c_replaceProtectStrategyWithNonProtect() public {
        // Setup: create a vault containing a protect strategy
        (ConcreteMultiStrategyVault newVault, ) = _createNewVault(false, false, false);
        MockERC4626Protect protectStrat = new MockERC4626Protect(
            IERC20(address(asset)), 
            "Mock Protect", 
            "MP"
        );
        Strategy memory protectStrategy = Strategy({
            strategy: IStrategy(address(protectStrat)),
            allocation: Allocation({index: 0, amount: 3333})
        });
        vm.startPrank(admin);
        newVault.addStrategy(0, true, protectStrategy);
        assertEq(newVault.protectStrategy(), address(protectStrategy.strategy), "incorrect protectStrategy address");
        // Setup ends


        // bug flow: have a non-protect strategy replace the protectStrategy. 
        // We see that the `newVault.protectStrategy()` is not reset to 0x0 and keeps pointing to the same old address.
        Strategy memory newNonProtectStrategy = _createMockStrategy(IERC20(address(asset)), false);
        newVault.addStrategy(0, true, newNonProtectStrategy); // replace strategy
        assertEq(newVault.protectStrategy(), address(protectStrategy.strategy), "no bug"); // @audit-issue : should have been 0x0
    }
```

## Mitigation 
Add this single line of code:
```diff
  File: src/libraries/MultiStrategyVaultHelper.sol

   204:              function addOrReplaceStrategy(
   205:                  Strategy[] storage strategies,
   206:                  Strategy memory newStrategy_,
   207:                  bool replace_,
   208:                  uint256 index_,
   209:                  address protectStrategy_,
   210:                  IERC20 asset
   211:              ) public returns (address protectStrategy, IStrategy newStrategyIfc, IStrategy stratToBeReplacedIfc) {
   212:                  // Calculate total allotments of current strategies
   213:                  protectStrategy = protectStrategy_;
   214:                  uint256 allotmentTotals = 0;
   215:                  uint256 len = strategies.length;
   216:                  for (uint256 i = 0; i < len; ) {
   217:                      allotmentTotals += strategies[i].allocation.amount;
   218:                      unchecked {
   219:                          i++;
   220:                      }
   221:                  }
   222:
   223:                  // Adding or replacing strategy based on `replace_` flag
   224:                  if (replace_) {
   225:                      if (index_ >= len) revert InvalidIndex(index_);
   226:
   227:                      // Ensure replacing doesn't exceed total allotment limit
   228:                      if (
   229:                          allotmentTotals - strategies[index_].allocation.amount + newStrategy_.allocation.amount >
   230:                          MAX_BASIS_POINTS
   231:                      ) {
   232:                          revert AllotmentTotalTooHigh();
   233:                      }
   234:
   235:                      // Replace the strategy at `index_`
   236:                      stratToBeReplacedIfc = strategies[index_].strategy;
   237:                      protectStrategy_ = removeStrategy(stratToBeReplacedIfc, protectStrategy_, asset);
+  238:                      protectStrategy = protectStrategy_;  // if vault's protect-strategy was removed, this becomes 0x0 here
   239:                      strategies[index_] = newStrategy_;
   240:                  } else {
   241:                      // Ensure adding new strategy doesn't exceed total allotment limit
   242:                      if (allotmentTotals + newStrategy_.allocation.amount > MAX_BASIS_POINTS) {
   243:                          revert AllotmentTotalTooHigh();
   244:                      }
   245:
   246:                      // Add the new strategy to the array
   247:                      strategies.push(newStrategy_);
   248:                  }
   249:
   250:                  // Handle protect strategy assignment if applicable
   251:                  if (newStrategy_.strategy.isProtectStrategy()) {
   252:                      if (protectStrategy_ != address(0)) revert MultipleProtectStrat();
   253:                      protectStrategy = address(newStrategy_.strategy);
   254:                  }
   255:
   256:                  // Approve the asset for the new strategy
   257:                  asset.forceApprove(address(newStrategy_.strategy), type(uint256).max);
   258:
   259:                  // Return the address of the new strategy
   260:                  newStrategyIfc = newStrategy_.strategy;
   261:              }
```

[Back to Top](#summaryTable)
---

### <a id="h-05"></a>[H-05]
## **Reward tokens can be stolen due to unvalidated user supplied `ctAssetToken` address in `swapTokensForReward()`**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/swapper/Swapper.sol#L79
<br>

## Description
Any external user can call `swapTokensForReward()` with a [malicious ctAssetToken_ address](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/swapper/Swapper.sol#L79) and steal all the reward tokens. This is possible because `ctAssetToken_` address is never validated by the protocol. 
```js
  File: src/swapper/Swapper.sol

    73:              /// @notice Swaps ctAsset tokens for reward tokens
    74:              /// @param ctAssetToken_ The address of the ctAsset token
    75:              /// @param rewardToken_ The address of the reward token
    76:              /// @param ctAssetAmount_ The amount of ctAsset tokens to swap
    77:              /// @dev The function checks if the reward token is a valid reward token
    78:              function swapTokensForReward(
    79:@--->             address ctAssetToken_,
    80:                  address rewardToken_,
    81:                  uint256 ctAssetAmount_
    82:              ) external nonReentrant {
```

## Impact
All available reward tokens in the treasury can be stolen.

## Proof of Concept
Use the following patch to update `test/Swapper.t.sol` and run the test with `forge test --mt test_t0x1c_swapTokensForReward -vv` to see it pass:

```diff
diff --git a/Swapper.t.sol b/Swapper.t.sol
index 18268f5..f611cd0 100644
--- a/Swapper.t.sol
+++ b/Swapper.t.sol
@@ -349,8 +349,83 @@ contract StrategyBaseTest is Test, SwapperEvents {
             ORACLE_QUOTE_DECIMALS, // oracleDecimals_,
             rewardUSDPair
         ); // oraclePair_
+    } 
+    
+    function test_t0x1c_swapTokensForReward() public {
+        uint256 rewardAmountInTreasury = 100_000_000 * 10 ** ASSET_DECIMALS;
+        int256 rewardPrice = int256(500 * 10 ** ORACLE_QUOTE_DECIMALS);
+        int256 assetPrice = int256(22 * 10 ** ORACLE_QUOTE_DECIMALS);
+        // mint some reward tokens to the treasury
+        rewardToken.mint(treasury, rewardAmountInTreasury);
+
+        // Malicious ctAsset token
+        MaliciousCTtoken maliciousctToken = new MaliciousCTtoken(address(asset));
+
+        // set prices for the oracle
+        _setOraclePrices(assetPrice, rewardPrice);
+
+        // Hazel wants to swap half of his ctAsset belongings
+        uint256 hazelCtAssetAmount = maliciousctToken.balanceOf(hazel);
+        uint256 ctAssetAmountToBeSwapped = hazelCtAssetAmount;
+        // treasury to approve the spending of reward tokens
+        vm.prank(treasury);
+        rewardToken.approve(address(swapper), rewardAmountInTreasury);
+        vm.prank(hazel);
+        maliciousctToken.approve(address(swapper), ctAssetAmountToBeSwapped);
+
+        // calculate the expected answer
+        uint256 assetAmountFromCtAssetAmount = maliciousctToken.convertToAssets(ctAssetAmountToBeSwapped); // @audit-info : returns a made up number
+        console.log("assetAmountFromCtAssetAmount =", assetAmountFromCtAssetAmount);
+        uint256 totalRewardTokenAmount = _getExpectedRewardTokenAmount(
+            assetAmountFromCtAssetAmount,
+            assetPrice,
+            rewardPrice
+        );
+
+        vm.startPrank(hazel);
+        vm.expectEmit(false, true, true, true, address(swapper));
+        emit SwapperEvents.Swapped(
+            hazel,
+            address(maliciousctToken),
+            address(rewardToken),
+            ctAssetAmountToBeSwapped,
+            totalRewardTokenAmount
+        );
+        uint256 beforeAttack = rewardToken.balanceOf(hazel);
+        console.log("Hazel's reward token balance before attack=", beforeAttack);
+        // create the swap
+        swapper.swapTokensForReward(address(maliciousctToken), address(rewardToken), ctAssetAmountToBeSwapped);
+
+        vm.stopPrank();
+        console.log("Hazel's reward token balance after attack =", rewardToken.balanceOf(hazel));
+        assertGt(rewardToken.balanceOf(hazel), beforeAttack, "Attack failed");
     }
 }
 
+contract MaliciousCTtoken is ERC20 {
+    address public _asset;
+    
+    constructor(address asset_) ERC20("Malicious ct Token", "MTKN") {
+        _asset = asset_;
+        _mint(msg.sender, 1000000 * 10**18); // Mint some tokens to attacker
+    }
+    
+    // Fake convertToAssets function that returns inflated value
+    function convertToAssets(uint256) public pure returns (uint256) {
+        // Return massively inflated value
+        // return shares * 1000000;
+        return 900 ether;
+    }
+
+    function asset() public view returns (address) {
+        return _asset;
+    }
+
+    // Override transfer to always return true but not actually transfer
+    function transferFrom(address, address, uint256) public pure override returns (bool) {
+        // Do nothing but return success
+        return true;
+    }
+}
 // mint some reward tokens to the treasury
 // reward.mint(treasury, 1_000_000 * 10**ASSET_DECIMALS);
```

## Mitigation 
Maintain a whitelist of allowed `ctAssetToken_` addresses and ensure that the user-supplied address belongs to it.

[Back to Top](#summaryTable)
---

### <a id="h-06"></a>[H-06]
## **`_withdrawStrategyFunds()` reverts prematurely even if withdrawable amount in vault is sufficient**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L461-L463
<br>

## Description
The following logic in [_withdrawStrategyFunds()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L461-L463) causes premature revert without checking if there are enough funds across all strategies to cover the requested `amount_`:
```js
  File: src/vault/ConcreteMultiStrategyVault.sol

   444:              function _withdrawStrategyFunds(uint256 amount_, address receiver_) private {
   445:                  IERC20 asset_ = IERC20(asset());
   446:                  //Not in a loop
   447:                  //slither-disable-next-line calls-loop
   448:                  uint256 float = asset_.balanceOf(address(this));
   449:                  // If there is enough in the vault, send the full amount
   450:                  if (amount_ <= float) {
   451:                      asset_.safeTransfer(receiver_, amount_);
   452:                  } else {
   453:                      uint256 diff = amount_ - float;
   454:                      uint256 totalWithdrawn = 0;
   455:                      uint256 len = strategies.length;
   456:                      for (uint256 i; i < len; ) {
   457:                          Strategy memory strategy = strategies[i];
   458:                          //We control both the length of the array and the external call
   459:                          //slither-disable-next-line calls-loop
   460:                          uint256 withdrawable = strategy.strategy.previewRedeem(strategy.strategy.balanceOf(address(this)));
   461:@--->                     if (diff.mulDiv(strategy.allocation.amount, MAX_BASIS_POINTS, Math.Rounding.Ceil) > withdrawable) {
   462:@--->                         revert InsufficientFunds(strategy.strategy, diff * strategy.allocation.amount, withdrawable);
   463:@--->                     }
   464:                          uint256 amountToWithdraw = amount_.mulDiv(
   465:                              strategy.allocation.amount,
   466:                              MAX_BASIS_POINTS,
   467:                              Math.Rounding.Ceil
   468:                          );
   469:                          //We control both the length of the array and the external call
   470:                          //slither-disable-next-line unused-return,calls-loop
   471:                          strategy.strategy.withdraw(amountToWithdraw, receiver_, address(this));
   472:                          totalWithdrawn += amountToWithdraw;
   473:                          unchecked {
   474:                              i++;
   475:                          }
   476:                      }
   477:                      // If enough wasn't drawn out, use the float (this is due to dust)
   478:                      if (totalWithdrawn < amount_ && amount_ - totalWithdrawn <= float) {
   479:                          asset_.safeTransfer(receiver_, amount_ - totalWithdrawn);
   480:                      }
   481:                  }
   482:              }
```

It seems the code assumes that all deposited funds will be distributed & present across strategies as per the allocation ratios. This is not necessarily the case. It's quite possible that a vault (float + strategy funds) may have have enough funds to satisfy a withdrawal request even when an individual strategy lacks the proportional amount. Example (multiple other scenarios are possible):
- Vault is created.
- Vault has strategies S1 and S2, each with 40% allocation.
- Amount deposited = 100. S1 & S2 get 40 each. Float amount = 20.
- Strategy S3 added. Allocations set to 33.33% for each strategy.
- Amount deposited = 90. Each strategy gets 30 (rounded value).
- Current allocations: float amount = 20 and S1 = 70, S2 = 70, S3 = 30.
- Withdrawal request raised for amount = 140.  Clearly, there are enough funds to process the request.
- However, function logic checks:
    - S1 funds >= (33.33% of (140 - 20)) ? Yes.
    - S2 funds >= (33.33% of (140 - 20)) ? Yes.
    - S3 funds >= (33.33% of (140 - 20)) ? No. Code reverts.

## Impact
The function reverts even if there are enough funds to fulfill the withdrawal request.

## Proof of Concept
Add the following inside `test/ConcreteMultiStrategyVault.t.sol` to see it revert when run with `forge test --mt test_t0x1c_withdraw_enough_funds_but_reverts -vv`:
```js
    function test_t0x1c_withdraw_enough_funds_but_reverts() public {
        (ConcreteMultiStrategyVault newVault, ) = _createNewVault(false, false, false);
        
        // Set allocation to 40% for first & second strategy
        vm.startPrank(admin);
        newVault.removeStrategy(2); 
        newVault.removeStrategy(1); 
        newVault.removeStrategy(0); 
        newVault.addStrategy(0, false, 
            Strategy({
                strategy: IStrategy(address(new MockERC4626(asset, "Mock Shares", "MS"))),
                allocation: Allocation({index: 0, amount: 4000})
            }));
        newVault.addStrategy(1, false, 
            Strategy({
                strategy: IStrategy(address(new MockERC4626(asset, "Mock Shares", "MS"))),
                allocation: Allocation({index: 1, amount: 4000})
            }));
        Strategy[] memory strats = newVault.getStrategies();
        vm.stopPrank();

        // Mint test assets
        asset.mint(hazel, 190 ether);
        vm.startPrank(hazel);
        asset.approve(address(newVault), 190 ether);

        // First deposit (100 ether) 
        newVault.deposit(100 ether, hazel);
        // Verify deposit
        assertEq(asset.balanceOf(address(newVault)), 20 ether);
        assertEq(asset.balanceOf(address(strats[0].strategy)), 40 ether);
        assertEq(asset.balanceOf(address(strats[1].strategy)), 40 ether);
        vm.stopPrank();

        // Add third strategy and reallocate
        vm.startPrank(admin);
        newVault.addStrategy(2, false, 
            Strategy({
                strategy: IStrategy(address(new MockERC4626(asset, "Mock Shares", "MS"))),
                allocation: Allocation({index: 2, amount: 1000})
            }));
        Allocation[] memory allocations = new Allocation[](3);
        allocations[0] = Allocation({index: 0, amount: 3333});
        allocations[1] = Allocation({index: 1, amount: 3333});
        allocations[2] = Allocation({index: 2, amount: 3334});
        newVault.changeAllocations(allocations, false);
        strats = newVault.getStrategies();
        vm.stopPrank();
        
        // Second deposit (90 ether)
        vm.startPrank(hazel);
        newVault.deposit(90 ether, hazel);
        // Verify deposits 
        assertEq(asset.balanceOf(address(newVault)), 20 ether, "Float balance incorrect after second deposit");
        assertApproxEqAbs(asset.balanceOf(address(strats[0].strategy)), 70 ether, 1e17, "Strategy1 balance incorrect after second deposit");
        assertApproxEqAbs(asset.balanceOf(address(strats[1].strategy)), 70 ether, 1e17, "Strategy2 balance incorrect after second deposit");
        assertApproxEqAbs(asset.balanceOf(address(strats[2].strategy)), 30 ether, 1e17, "Strategy3 balance incorrect after second deposit");

        // Withdraw 140 ether
        console.log("Withdrawing 140 ether...");
        newVault.withdraw(140 ether, hazel, hazel); // @audit-issue : reverts

        // unreachable code
        console.log("Withdrawal successful?");
    }
```

## Mitigation 
**_Note that there is an additional issue on L464 where `diff` should be used instead of `amount_` and has been raised seperately under the title_** "`withdraw()` reverts while attempting to pull funds from strategies due to use of `amount_` instead of `diff` inside `_withdrawStrategyFunds()`". **_The following fix should ideally be implemented in conjunction with that._**

To fix the current issue, considerable rework of logic is required:
- If entire `amount_` is available as the float amount, transfer and exit (Same as current implementation).
- Else try to withdraw the `diff` as per allocation ratio from each strategy (Same as current implementation).
- If a strategy does not have enough `withdrawable`, withdraw whatever it has. **Do not** revert. Increment `totalWithdrawn` accordingly.
- Introduce a variable `updatedWithdrawable[i]` which makes note of the amount for this strategy once above withdrawal took place.
- At the end of the loop, check if `totalWithdrawn >= diff`. If not, then loop through the strategies again and pull whatever funds are available in each strategy (denoted by `updatedWithdrawable[i]`). Do so until `totalWithdrawn >= diff` or the loop ends.
- Keep the final check on L478 as-is but add an `else if` condition to it:
```diff
   477:                      // If enough wasn't drawn out, use the float (this is due to dust)
   478:                      if (totalWithdrawn < amount_ && amount_ - totalWithdrawn <= float) {
   479:                          asset_.safeTransfer(receiver_, amount_ - totalWithdrawn);
   480:                      }
+  480:                      else if (totalWithdrawn < amount_ && amount_ - totalWithdrawn > float) revert InsufficientFundsInVaultToWithdraw(xx, yy);
```

[Back to Top](#summaryTable)


---

<br>

## **MEDIUM-SEVERITY BUGS**
---
 
### <a id="m-01"></a>[M-01]
## **Some reward tokens once removed, can not be added again due to absence of allowance reset**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/strategies/StrategyBase.sol#L217-L228
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/strategies/StrategyBase.sol#L206
<br>

## Summary
Observed behaviour:
- Step1. Add Bancor (`BNT`) as a reward token by calling [addRewardToken()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/strategies/StrategyBase.sol#L191).
- Step2. Remove BNT by calling [removeRewardToken()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/strategies/StrategyBase.sol#L217).
- Step3. At some time in the future, again add BNT by calling `addRewardToken()`. **_It reverts._**

When [addRewardToken()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/strategies/StrategyBase.sol#L191) is called, allowance is set to `type(uint256).max` [here](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/strategies/StrategyBase.sol#L206). If a reward token is to be removed, [removeRewardToken()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/strategies/StrategyBase.sol#L217) is then called but its allowance is never reset to zero.<br>
Some tokens like `Bancor (BNT)` (address: [0x1F573D6Fb3F13d689FF844B4cE37794d79a7FF1C](https://etherscan.io/address/0x1F573D6Fb3F13d689FF844B4cE37794d79a7FF1C)) require that the allowance be set to zero before being set to a non-zero value. Hence, if the protocol decides to later on add Bancor once again to the list of reward tokens by calling `addRewardToken()` now, it will revert and there's no way to do so.

<br>

_**Note**_: It's important to note that although there are other tokens like `USDT` which too require allowance to be first set to zero before any change, the `approve()` function of USDT **_does not_** return a boolean and hence [such tokens are out of scope as per the contest details](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/tree/main?tab=readme-ov-file#erc20-token-behaviors-in-scope) (`Missing return values` are `Out of scope`). In fact USDT can't even be added as a reward token since it will fail the `if` condition on [L206 inside `addRewardToken()`](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/strategies/StrategyBase.sol#L206).<br>
BNT's `approve()` on the other hand **_does_** return a `bool` as per ERC20 standards and hence is in-scope (`Approval race protections` are `In scope`). See BNT's code [here](https://etherscan.io/address/0x1F573D6Fb3F13d689FF844B4cE37794d79a7FF1C#code) L268-L286.

## Description
[Relevant lines of code:](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/strategies/StrategyBase.sol#L191-L228)
```js
  File: src/strategies/StrategyBase.sol

   191:              function addRewardToken(RewardToken calldata rewardToken_) external onlyOwner nonReentrant {
   192:                  // Ensure the reward token address is not zero, not already approved, and its parameters are correctly initialized.
   193:                  if (address(rewardToken_.token) == address(0)) {
   194:                      revert InvalidRewardTokenAddress();
   195:                  }
   196:                  if (rewardTokenApproved[address(rewardToken_.token)]) {
   197:                      revert RewardTokenAlreadyApproved();
   198:                  }
   199:                  if (rewardToken_.accumulatedFeeAccounted != 0) {
   200:                      revert AccumulatedFeeAccountedMustBeZero();
   201:                  }
   202:
   203:                  // Add the reward token to the list and approve it for unlimited spending by the strategy.
   204:                  rewardTokens.push(rewardToken_);
   205:                  rewardTokenApproved[address(rewardToken_.token)] = true;
   206:@--->             if (!rewardToken_.token.approve(address(this), type(uint256).max)) {
   207:                      revert ERC20ApproveFail();
   208:                  }
   209:              }
   210:
   211:              /**
   212:               * @notice Removes a reward token from the strategy.
   213:               * @dev This function allows the owner to remove a reward token from the strategy's list of reward tokens.
   214:               * It shifts the elements in the array to maintain a compact array after removal.
   215:               * @param rewardToken_ The reward token to be removed, encapsulated in a RewardToken struct.
   216:               */
   217:              function removeRewardToken(RewardToken calldata rewardToken_) external onlyOwner {
   218:                  // Ensure the reward token is approved before attempting removal.
   219:                  if (!rewardTokenApproved[address(rewardToken_.token)]) {
   220:                      revert RewardTokenNotApproved();
   221:                  }
   222:
   223:                  rewardTokens[_getIndex(address(rewardToken_.token))] = rewardTokens[rewardTokens.length - 1];
   224:                  rewardTokens.pop();
   225:
   226:                  // Mark the reward token as not approved.
   227:@--->             rewardTokenApproved[address(rewardToken_.token)] = false;
   228:              }
```

## Impact
Some tokens can't be added again as reward tokens once removed.

## Tools Used
Foundry

## Proof of Concept
Add the following test file inside the `test/` folder to witness `testCorrectAllowanceChange()` pass and `testWrongAllowanceChange()` fail when run with `forge test --mc BancorApproveTest -vvvv`:
```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Test} from "forge-std/Test.sol";
import {console2 as console} from "forge-std/console2.sol";

interface IERC20 {
    function approve(address spender, uint256 amount) external returns (bool);
    function transfer(address recipient, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function balanceOf(address owner) external view returns (uint256);
}

contract BancorApproveTest is Test {
    address constant BNT_ADDRESS = 0x1F573D6Fb3F13d689FF844B4cE37794d79a7FF1C; // Mainnet BNT contract
    IERC20 bnt = IERC20(BNT_ADDRESS);
    address user = makeAddr("user");
    address spender = makeAddr("spender");

    function setUp() public {
        vm.createSelectFork("https://rpc.ankr.com/eth"); 
        vm.deal(user, 10 ether); // Give `user` 10 ether to cover transaction costs
        uint256 bntAmount = 10 * 10 ** 18; // 10 BNT (BNT has 18 decimals)
        deal(BNT_ADDRESS, user, bntAmount);
    }

    function testCorrectAllowanceChange() public { // PASSES
        vm.startPrank(user); 
        
        uint256 approveAmount = 1 * 10 ** 18; 

        require(bnt.approve(spender, approveAmount)); // Approving 1 BNT
        uint256 allowance1 = bnt.allowance(user, spender);
        console.log("allowance =", allowance1);

        require(bnt.approve(spender, 0)); // Allowance changed to 0
        uint256 allowance2 = bnt.allowance(user, spender);
        console.log("allowance2 =", allowance2);

        require(bnt.approve(spender, approveAmount)); // Approving 1 BNT
        uint256 allowance3 = bnt.allowance(user, spender);
        console.log("allowance3 =", allowance3);
    }

    function testWrongAllowanceChange() public { // FAILS
        vm.startPrank(user); 
        
        uint256 approveAmount = 1 * 10 ** 18; 

        require(bnt.approve(spender, approveAmount)); // Approving 1 BNT
        uint256 allowance1 = bnt.allowance(user, spender);
        console.log("allowance =", allowance1);

        // @audit-info : give the same allowance once more (without first resetting to 0) and see it revert
        bnt.approve(spender, approveAmount); // Approving 1 BNT

        // unreachable code
        uint256 allowance2 = bnt.allowance(user, spender);
        console.log("allowance2 =", allowance2);
    }
}
```

## Recommendation
Reset the allowance to zero inside `removeRewardToken()`. Note that tokens which `Revert on zero value approvals` are also in-scope as per the [contest guidelines](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/tree/main?tab=readme-ov-file#erc20-token-behaviors-in-scope) and hence the protocol may want to maintain a list of tokens where this is not allowed so that those cases can be handled individually. Consider using OZ's `forceApprove()` instead of raw `approve()`.
```diff
   217:              function removeRewardToken(RewardToken calldata rewardToken_) external onlyOwner {
   218:                  // Ensure the reward token is approved before attempting removal.
   219:                  if (!rewardTokenApproved[address(rewardToken_.token)]) {
   220:                      revert RewardTokenNotApproved();
   221:                  }
   222:
   223:                  rewardTokens[_getIndex(address(rewardToken_.token))] = rewardTokens[rewardTokens.length - 1];
   224:                  rewardTokens.pop();
   225:
   226:                  // Mark the reward token as not approved.
   227:                  rewardTokenApproved[address(rewardToken_.token)] = false;
+  228:                  require(rewardToken_.token.approve(address(this), 0)); // reset allowance to zero
   228:              }
```

[Back to Top](#summaryTable)
---
 
### <a id="m-02"></a>[M-02]
## **`depositLimit` not respected during deposit due to incorrect logic**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L233
<br>

## Description
`depositLimit` [is defined as](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L53-L55):
```js
    /// @notice The maximum amount of assets that can be deposited into the vault.
    /// @dev Public variable to store the deposit limit of the vault.
    uint256 public depositLimit;
``` 
However the [check inside `deposit()`](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L233) only verifies the **_current_** deposit amount of assets instead of the total amount in the vault:
```js
  File: src/vault/ConcreteMultiStrategyVault.sol

   227:              function deposit(
   228:                  uint256 assets_,
   229:                  address receiver_
   230:              ) public override nonReentrant whenNotPaused returns (uint256 shares) {
   231:                  _validateAndUpdateDepositTimestamps(receiver_);
   232:
   233:@--->             if (assets_ > maxDeposit(receiver_) || assets_ > depositLimit) revert MaxError();
   234:
   235:                  // Calculate the fee in shares
   236:                  uint256 feeShares = _convertToShares(
   237:                      assets_.mulDiv(uint256(fees.depositFee), MAX_BASIS_POINTS, Math.Rounding.Floor),
   238:                      Math.Rounding.Floor
   239:                  );
   240:
```

Hence if `depositLimit` is say, `100`, two calls to `deposit()` with `assets_` as `70` in one and `90` in the other will pass the check and cause the deposits to reach `160`, way past the `100` limit.

## Impact
Vault's `depositLimit` can be breached.

## PoC
We will modify the existing test [test_shouldNotLetUserViolateDepositLimit()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/test/ConcreteMultiStrategyVault.t.sol#L1131) to see it pass when run with `forge test --mt test_shouldNotLetUserViolateDepositLimit -vv`:
```diff
    function test_shouldNotLetUserViolateDepositLimit() public {
-       uint256 amount_ = 4 ether;
+       uint256 amount_ = 0.9 ether;
        (ConcreteMultiStrategyVault newVault, ) = _createNewVault(false, false, false);
        vm.prank(admin);
        newVault.setDepositLimit(1e18);

-       asset.mint(hazel, amount_);
+       asset.mint(hazel, 2 * amount_);
        vm.prank(hazel);
-       asset.approve(address(newVault), amount_);
+       asset.approve(address(newVault), 2 * amount_);

-       vm.expectRevert(Errors.MaxError.selector);
        vm.prank(hazel);
        newVault.deposit(amount_, hazel);

+       vm.prank(hazel);
+       newVault.deposit(amount_, hazel);

+       assertGt(newVault.totalAssets(), 1e18, "within limits"); // @audit : limit breached
    }
```

## Recommendation
- Option-1:
Change the condition to `if (assets_ > maxMint(receiver_)) revert MaxError();` like has been done [inside `mint()`](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L306). Note that this seems to be incomplete because `maxMint()` implementation [ignores the `address` param passed to it](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L657) and only concerns itself with `totalAssets()`. But if it is acceptable in `mint()`, the same can be applied here too.

- Option-2:
Add a new function which returns the amount already deposited by an address say, `alreadyDepositedByAddress()` and change the condition to:
```js
if (alreadyDepositedByAddress(receiver_) + assets_ > maxDeposit(receiver_) || totalAssets() + assets_ > depositLimit) revert MaxError();
```

[Back to Top](#summaryTable)

---

### <a id="m-03"></a>[M-03]
## **Funds can get stuck and can't be withdrawn entirely via `redeem()` or `withdraw()`**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L304
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L362
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L395
<br>

## Description
When user calls `mint(shares)` user's funds are pulled into the vault as assets and `shares` are issued to them. However it's possible that the user can't `withdraw()` or `redeem()` their entire assets and funds remain stuck in the vault.
<br>

Example:<br>
- **Step1:** Hazel calls [mint(15e8)](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L304) and the protocol uses `CEIL` rounding direction to pull funds from her and stores `2` assets inside the vault.
  - $assets = ceiling(\frac{15e8 * 1}{1e9}) = 2$
- **Step2:** If Hazel wants to call [redeem(15e8)](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L362), the asset amount is calculated using `FLOOR` rounding direction and hence she will receive just `1` assset back.
  - $assets = floor(\frac{15e8 * (2 + 1)}{15e8 + 1e9}) = 1$
- **Step3:** Instead of `redeem()`, she can choose to call [withdraw(2)](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L395) to pull out all her funds. But `withdraw()` uses `CEIL` to calculate how many shares to burn and the value exceeds what she was actually issued. So this reverts with `MaxError()`.
  - $shares = ceiling(\frac{2 * (15e8 + 1e9)}{(2 + 1)}) = 1666666667$ which is greater than $15e8$.

## Impact
Funds remain stuck and can not be withdrawn.

## Proof of Concept
Add the following test inside `test/ConcreteMultiStrategyVault.t.sol` to see it revert when run with `forge test --mt test_t0x1c_mint_redeem_failure -vv`:
```js
    function test_t0x1c_mint_redeem_failure() public {
        (ConcreteMultiStrategyVault newVault, ) = _createNewVault(false, false, false);
        asset.mint(hazel, 1 ether);
        vm.label(address(newVault), "newVault");
        uint256 hazelShares = 15e8 ;

        vm.prank(admin);
        newVault.toggleVaultIdle();

        vm.startPrank(hazel);
        asset.approve(address(newVault), type(uint256).max);

        // mint
        uint256 hazelAssets = newVault.mint(hazelShares, hazel);
        console.log("\nhazelAssets =", hazelAssets);

        // redeem preview
        uint256 hazelAssetsRedeemExpected = newVault.previewRedeem(hazelShares); 
        console.log("\nassets returned if all shares are redeemed =", hazelAssetsRedeemExpected); // @audit-info : all assets are not redeemable
        require(hazelAssetsRedeemExpected < hazelAssets, "entire asset amount correctly redeemed");
     
        // withdraw 
        console.log("\nAttempting to withdraw all assets.."); 
        newVault.withdraw(hazelAssets); // @audit-info : reverts with `MaxError()`
        console.log("Withdraw worked."); // @audit-info : unreachable code
    }
```
<br>

Output:
```text
Logs:
hazelAssets = 2                                   <-------------- @audit-info : 2 assets pulled from the user into the vault.

assets returned if all shares are redeemed = 1    <-------------- @audit-info : redeem() does not return all the assets.

Attempting to withdraw all assets..               <-------------- @audit-info : reverts with `MaxError()`.

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 31.49ms (8.12ms CPU time)

Ran 1 test suite in 102.03ms (31.49ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/ConcreteMultiStrategyVault.t.sol:ConcreteMultiStrategyVaultTest
[FAIL. Reason: MaxError()] test_t0x1c_mint_redeem_failure() (gas: 5144010)
```

## Recommendation
Note that all the rounding directions in these functions are correct and as expected, in the favour of the protocol. However during the calculations inside `mint()`, it is recommended to follow one of these approaches:
- **Option1:**(Round-down shares) Instead of pulling `2` assets against `15e8` shares, issue `1` and reduce the shares to be issued to `1e9`. That's because $\frac{1 * (0 + 1e9)}{0 + 1} = 1e9$. Ignore the user's demand for the remaining $15e8 - 1e9 = 5e8$ shares.
- **Option2:**(Round-up shares) Pull `2` assets and issue `2e9` shares. This is less preferable as the user may not prefer a higher fund amount.

[Back to Top](#summaryTable)

---

### <a id="m-04"></a>[M-04]
## **Call to `withdraw()` reverts due to attempt to burn more shares than issued**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L403
<br>

## Summary
Upon calling `withdraw()`, the protocol attempts to burn `shares + feeShares` of the user when they would really have only `shares` issued to them. The protocol incorrectly increases quantity of `shares` to burn instead of decreasing the amount of `assets` to return. This causes a revert.

## Description
The withdrawal fee ought to be deducted from the existing shares and the assets corresponding to the remaining shares need to be returned to the withdrawer when [withdraw()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L403) is called. That's how it's implemented inside `redeem()` too.<br>
However, the protocol **_adds_** the fee shares on top of the shares corresponding to the asset amount requested. This results in a situation where the user would not have enough shares thus reverting the transaction.
<br>

```js
  File: src/vault/ConcreteMultiStrategyVault.sol

   388:              function withdraw(
   389:                  uint256 assets_,
   390:                  address receiver_,
   391:                  address owner_
   392:              ) public override nonReentrant whenNotPaused returns (uint256 shares) {
   393:                  if (receiver_ == address(0)) revert InvalidRecipient();
   394:                  if (assets_ > maxWithdraw(owner_)) revert MaxError();
   395:                  shares = _convertToShares(assets_, Math.Rounding.Ceil);
   396:                  if (shares <= DUST) revert ZeroAmount();
   397:
   398:                  // If msg.sender is the withdrawal queue, go straght to the actual withdrawal
   399:                  uint256 withdrawalFee = uint256(fees.withdrawalFee);
   400:                  uint256 feeShares = msg.sender != feeRecipient
   401:                      ? shares.mulDiv(MAX_BASIS_POINTS, MAX_BASIS_POINTS - withdrawalFee, Math.Rounding.Floor) - shares
   402:                      : 0;
   403:@--->             shares += feeShares; // @audit : this surpasses the number of shares issued to the user
   404:
   405:                  _withdraw(assets_, receiver_, owner_, shares, feeShares);
   406:              }
```

Example:<br>
- **Step1:** Hazel calls `deposit(1e18)`. No deposit fee levied by the protocol. She is issued `1e27` shares.
  - $shares = \frac{1e18 * (0 + 1e9)}{0 + 1} = 1e27$
- **Step2:** Hazel calls `withdraw(1e18)`. `1%` withdrawal fee (`100 bips`) levied by the protocol. Now, `shares` is incremented on [L403](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L403) followed by `_withdraw(..., shares, ..)` on the next line.
  - `shares += feeShares;`
  - ` _withdraw(assets_, receiver_, owner_, shares, feeShares);`
- `_withdraw()` tries to burn `shares` on [L423](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L423). Since the user has only `1e27` shares, this reverts with error `ERC20InsufficientBalance`.

## Proof of Concept
Add the following test inside `test/ConcreteMultiStrategyVault.t.sol` to see it revert when run with `forge test --mt test_t0x1c_withdraw_with_fees -vv`:
```js
    function test_t0x1c_withdraw_with_fees() public {
        (ConcreteMultiStrategyVault newVault, ) = _createNewVault(false, false, false);
        asset.mint(hazel, 100 ether);
        vm.label(address(newVault), "newVault");

        vm.startPrank(admin);
        newVault.setVaultFees(VaultFees({depositFee: 0, withdrawalFee: 100, protocolFee: 0, performanceFee: zeroFees}));
        newVault.toggleVaultIdle();
        vm.stopPrank();

        vm.startPrank(hazel);
        asset.approve(address(newVault), type(uint256).max);

        // deposit
        uint256 hazelShares = newVault.deposit(1 ether);
        console.log("\nhazelShares =", hazelShares);

        // withdraw 
        console.log("\nAttempting to withdraw all assets.."); 
        // @audit-issue : throws error --> ERC20InsufficientBalance(address(hazel), 1e27, 1.01e27)
        newVault.withdraw(1 ether); 
    }
```
<br>

Output:
```text
Failing tests:
Encountered 1 failing test in test/ConcreteMultiStrategyVault.t.sol:ConcreteMultiStrategyVaultTest
[FAIL. Reason: ERC20InsufficientBalance(0x0000000000000000000000000000000000003333, 1000000000000000000000000000 [1e27], 1010101010101010101010101010 [1.01e27])] test_t0x1c_withdraw_with_fees() (gas: 5262722)
```

## Mitigation
With the following changes, user would be able to call `withdraw(1 ether)` (_i.e. withdraw ALL assets_) and witness the exact same behaviour as `redeem(1e27)` (_i.e. redeem ALL shares_):
```diff
  File: src/vault/ConcreteMultiStrategyVault.sol

   388:              function withdraw(
   389:                  uint256 assets_,
   390:                  address receiver_,
   391:                  address owner_
   392:              ) public override nonReentrant whenNotPaused returns (uint256 shares) {
   393:                  if (receiver_ == address(0)) revert InvalidRecipient();
   394:                  if (assets_ > maxWithdraw(owner_)) revert MaxError();
   395:                  shares = _convertToShares(assets_, Math.Rounding.Ceil);
   396:                  if (shares <= DUST) revert ZeroAmount();
   397:
   398:                  // If msg.sender is the withdrawal queue, go straght to the actual withdrawal
   399:                  uint256 withdrawalFee = uint256(fees.withdrawalFee);
   400:                  uint256 feeShares = msg.sender != feeRecipient
-  401:                      ? shares.mulDiv(MAX_BASIS_POINTS, MAX_BASIS_POINTS - withdrawalFee, Math.Rounding.Floor) - shares
+  401:                      ? shares - shares.mulDiv(MAX_BASIS_POINTS, MAX_BASIS_POINTS + withdrawalFee, Math.Rounding.Floor)
   402:                      : 0;
-  403:                  shares += feeShares;
   404:
-  405:                  _withdraw(assets_, receiver_, owner_, shares, feeShares);
+  405:                  _withdraw(_convertToAssets(shares - feeShares, Math.Rounding.Floor), receiver_, owner_, shares, feeShares);
   406:              }
```

## Severity
I am torn between `Low` and `Med` because the user _can_ do the calculation themself off-chain and in order to correctly withdraw without a revert, they can call `withdraw(0.99 ether)` and it would work exactly like `redeem(1e27)`. I believe however that the general expectation by end user would be to see it work as suggested in the `Mitigation` section above ( _withdraw(ALL) should behave like redeem(ALL)_ ) and hence raising it as a `Med`.

[Back to Top](#summaryTable)

---

### <a id="m-05"></a>[M-05]
## **User can escape deposit fee by making multiple small deposits due to incorrect rounding direction**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L236-L239
<br>

## Description
The [deposit()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L236-L239) function calculates the `feeShares` by rounding in favour of the user instead of the protocol. Supposing a `1%` deposit fee (`100 bips`), a user can make multiple small deposits of `99` to escape the fee since `Floor((99 * 100) / 10000) = 0`.
<br>

```js
  File: src/vault/ConcreteMultiStrategyVault.sol

   235:                  // Calculate the fee in shares
   236:                  uint256 feeShares = _convertToShares(
   237:@--->                 assets_.mulDiv(uint256(fees.depositFee), MAX_BASIS_POINTS, Math.Rounding.Floor),
   238:@--->                 Math.Rounding.Floor
   239:                  );
```

This exploit can be implemented on chains with low gas fees.

## Proof of Concept
Add the following test inside `test/ConcreteMultiStrategyVault.t.sol` to see it pass when run with `forge test --mt test_t0x1c_deposit_with_fees -vv`:
```js
    function test_t0x1c_deposit_with_fees() public {
        (ConcreteMultiStrategyVault newVault, ) = _createNewVault(false, false, false);
        asset.mint(hazel, 100 ether);
        vm.label(address(newVault), "newVault");

        vm.startPrank(admin);
        newVault.setVaultFees(VaultFees({depositFee: 100, withdrawalFee: 0, protocolFee: 0, performanceFee: zeroFees})); // @audit-info : 1% deposit fee
        newVault.toggleVaultIdle();
        vm.stopPrank();

        vm.startPrank(hazel);
        asset.approve(address(newVault), type(uint256).max);

        // deposit
        uint256 hazelShares = newVault.deposit(99); // @audit-info : small deposit to escape fee
        console.log("\nhazelShares =", hazelShares);
        assertEq(hazelShares, 99e9, "fees charged");
    }
```

## Mitigation
```diff
  File: src/vault/ConcreteMultiStrategyVault.sol

   235:                  // Calculate the fee in shares
   236:                  uint256 feeShares = _convertToShares(
-  237:                      assets_.mulDiv(uint256(fees.depositFee), MAX_BASIS_POINTS, Math.Rounding.Floor),
-  238:                      Math.Rounding.Floor
+  237:                      assets_.mulDiv(uint256(fees.depositFee), MAX_BASIS_POINTS, Math.Rounding.Ceil),
+  238:                      Math.Rounding.Ceil
   239:                  );
```

[Back to Top](#summaryTable)
---

### <a id="m-06"></a>[M-06]
## **ProtocolFee is charged even for the `paused` duration**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L695
<br>

## Description
The [accruedProtocolFee()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L695) function forgets to exclude any duration of time for which the protocol may have been `paused`. The users couldn't have withdrawn during this period but are still forced to pay the fee.

- Suppose `feesUpdatedAt = 5 April 2024 10:10 PM`
- At time `6 April 2024 10:10 PM` any user/admin calls the external function [takePortfolioAndProtocolFees()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L735) which has the modifier `takeFees` which [internally calls accruedProtocolFee() on L105](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L105).
- Suppose the protocol was in paused state between `5 April 2024 11:09 PM` and `6 April 2024 10:09 PM`, for a duration of 23 hours. Actions like `withdraw()` or `redeem()` would have been not possible for any user due to the `whenNotPaused` modifier. 
- Yet, they end up paying the protocol fee for the paused duration of 23 hours.

```js
    /**
     * @notice Calculates the accrued protocol fee based on the current protocol fee rate and time elapsed.
     * @dev The protocol fee is calculated as a percentage of the total assets, prorated over time since the last fee update.
     * @return The accrued protocol fee in asset units.
     */
    function accruedProtocolFee() public view returns (uint256) {
        // Only calculate if a protocol fee is set
        if (fees.protocolFee > 0) {
            // Calculate the fee based on time elapsed and total assets, using floor rounding for precision
            return
                uint256(fees.protocolFee).mulDiv(
@-->                totalAssets() * (block.timestamp - feesUpdatedAt),   <-------- @audit : Any duration for which protocol was paused within this time should be excluded
                    SECONDS_PER_YEAR,
                    Math.Rounding.Floor
                ) / 10000; // Normalize the fee percentage
        } else {
            return 0;
        }
    }
```

## Recommendation
Store the start and end timestamp of the `pause` and exclude that duration while calculating time elapsed inside `accruedProtocolFee()`.

[Back to Top](#summaryTable)
---

### <a id="m-07"></a>[M-07]
## **Tiered fee not charged for upper boundary values**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/libraries/MultiStrategyVaultHelper.sol#L157
<br>

## Description
[IConcreteMultiStrategyVault.sol](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/interfaces/IConcreteMultiStrategyVault.sol#L6-L12) gives example of tiered fee:
```js
    // Example performanceFee: [{0000, 500, 300}, {501, 2000, 1000}, {2001, 5000, 2000}, {5001, 10000, 5000}]
    // == 0-5% increase 3%, 5.01-20% increase 10%, 20.01-50% increase 20%, 50.01-100% increase 50%
```

**Even if** we modify the tier boundaries to overlap like shown below, the code on [L157 inside MultiStrategyVaultHelper.sol](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/libraries/MultiStrategyVaultHelper.sol#L157) results in a situation where values at upper boundaries are charged no fee:
```js
    // performanceFee: [{0000, 500, 300}, {500, 2000, 1000}, {2000, 5000, 2000}, {5000, 10000, 5000}]
```

```js
  File: src/libraries/MultiStrategyVaultHelper.sol

   142:              function calculateTieredFee(
   143:                  uint256 shareValue,
   144:                  uint256 highWaterMark,
   145:                  uint256 totalSupply,
   146:                  VaultFees storage fees
   147:              ) public view returns (uint256 fee) {
   148:                  if (shareValue <= highWaterMark) return 0;
   149:                  // Calculate the percentage difference (diff) between share value and high water mark
   150:                  uint256 diff = uint256(shareValue.mulDiv(MAX_BASIS_POINTS, highWaterMark, Math.Rounding.Floor)) -
   151:                      uint256(MAX_BASIS_POINTS);
   152:
   153:                  // Loop through performance fee tiers
   154:                  uint256 len = fees.performanceFee.length;
   155:                  if (len == 0) return 0;
   156:                  for (uint256 i = 0; i < len; ) {
   157:@--->                 if (diff < fees.performanceFee[i].upperBound && diff > fees.performanceFee[i].lowerBound) {
   158:                          fee = ((shareValue - highWaterMark) * totalSupply).mulDiv(
   159:                              fees.performanceFee[i].fee,
   160:                              MAX_BASIS_POINTS * 1e18,
   161:                              Math.Rounding.Floor
   162:                          );
   163:                          break; // Exit loop once the correct tier is found
   164:                      }
   165:                      unchecked {
   166:                          i++;
   167:                      }
   168:                  }
   169:              }
```

## Impact
For a value like `100% or 10000` or `5% or 500` or any other upper boundary, no fee is charged (This is when configuration has overlapping boundaries. For non-overlapping ones, both boundaries are skipped while calculating the fees).

## Proof of Concept
Add this test inside `test/ConcreteMultiStrategyVault.t.sol` to see it pass when run with `forge test --mt test_t0x1c_tiered_fee_boundary_values -vv`:
```js
    function test_t0x1c_tiered_fee_boundary_values() public {
        // Setup vault with specific fee tiers and deposit a known amount
        (ConcreteMultiStrategyVault newVault, Strategy[] memory strats) = _createNewVault(true, false, false);
        
        // Define graduated fees with clear boundaries for testing
        GraduatedFee[] memory fees = new GraduatedFee[](2);
        fees[0] = GraduatedFee({lowerBound: 0, upperBound: 500, fee: 1000}); // 10% fee for 0-5% gain
        fees[1] = GraduatedFee({lowerBound: 500, upperBound: 1000, fee: 2000}); // 20% fee for 5-10% gain 
        // @audit-info : the example performance fee shown inside `src/interfaces/IConcreteMultiStrategyVault.sol` is even worse which has no overlap between lower & upper bounds:
        // Example performanceFee: [{0000, 500, 300}, {501, 2000, 1000}, {2001, 5000, 2000}, {5001, 10000, 5000}]
        // == 0-5% increase 3%, 5.01-20% increase 10%, 20.01-50% increase 20%, 50.01-100% increase 50%
        
        vm.prank(admin);
        newVault.setVaultFees(VaultFees({
            depositFee: 0,
            withdrawalFee: 0,
            protocolFee: 0,
            performanceFee: fees
        }));

        // Deposit initial amount
        uint256 depositAmount = 100 ether;
        asset.mint(hazel, depositAmount);
        vm.startPrank(hazel);
        asset.approve(address(newVault), depositAmount);
        newVault.deposit(depositAmount, hazel);
        vm.stopPrank();

        // Fast forward time to allow fees to accrue
        vm.warp(block.timestamp + 365.25 days);

        // Simulate exactly 5% gain (500 basis points)
        uint256 gain = (depositAmount * 501) / 10000; // @audit : causes share value to be 105.0099999 ether and hence `diff` to be 500
        asset.mint(address(strats[0].strategy), gain);

        // Take fees
        vm.prank(admin);
        newVault.takePortfolioAndProtocolFees();

        // Verify that fee recipient got nothing despite 5% gain
        // This is the bug
        uint256 feeRecipientBalance = newVault.balanceOf(feeRecipient);
        console.log("Fee recipient balance:", feeRecipientBalance);
        assertEq(feeRecipientBalance, 0, "Fee recipient should have received nothing due to boundary bug");
    }
```
## Recommendation
Two steps need to be taken:
1. Use overlapping boundaries. For example: `performanceFee: [{0000, 500, 300}, {500, 2000, 1000}, {2000, 5000, 2000}, {5000, 10000, 5000}]`
2. Use `<=` instead of `<` for upper bound check:
```diff
  File: src/libraries/MultiStrategyVaultHelper.sol

   142:              function calculateTieredFee(
   143:                  uint256 shareValue,
   144:                  uint256 highWaterMark,
   145:                  uint256 totalSupply,
   146:                  VaultFees storage fees
   147:              ) public view returns (uint256 fee) {
   148:                  if (shareValue <= highWaterMark) return 0;
   149:                  // Calculate the percentage difference (diff) between share value and high water mark
   150:                  uint256 diff = uint256(shareValue.mulDiv(MAX_BASIS_POINTS, highWaterMark, Math.Rounding.Floor)) -
   151:                      uint256(MAX_BASIS_POINTS);
   152:
   153:                  // Loop through performance fee tiers
   154:                  uint256 len = fees.performanceFee.length;
   155:                  if (len == 0) return 0;
   156:                  for (uint256 i = 0; i < len; ) {
-  157:                      if (diff < fees.performanceFee[i].upperBound && diff > fees.performanceFee[i].lowerBound) {
+  157:                      if (diff <= fees.performanceFee[i].upperBound && diff > fees.performanceFee[i].lowerBound) {
   158:                          fee = ((shareValue - highWaterMark) * totalSupply).mulDiv(
   159:                              fees.performanceFee[i].fee,
   160:                              MAX_BASIS_POINTS * 1e18,
   161:                              Math.Rounding.Floor
   162:                          );
   163:                          break; // Exit loop once the correct tier is found
   164:                      }
   165:                      unchecked {
   166:                          i++;
   167:                      }
   168:                  }
   169:              }
```

[Back to Top](#summaryTable)
---

### <a id="m-08"></a>[M-08]
## **Incorrect reward accounting inside `harvestRewards()` if `TokenHelper.attemptSafeTransfer()` fails**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/strategies/StrategyBase.sol#L351-L358
<br>

## Description
[StrategyBase::harvestRewards()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/strategies/StrategyBase.sol#L351-L358) makes use of `TokenHelper.attemptSafeTransfer()` twice with last param as `false` in order to avoid any reverts until the end of the loop. However it never checks the return value of the transfer which gives rise to issues:
```js
  File: src/strategies/StrategyBase.sol

   337:              function harvestRewards(
   338:                  bytes memory data
   339:              ) public virtual override(IStrategy) onlyVault returns (ReturnedRewards[] memory) {
   340:                  _getRewardsToStrategy(data);
   341:                  uint256 len = rewardTokens.length;
   342:                  ReturnedRewards[] memory returnedRewards = new ReturnedRewards[](len);
   343:                  for (uint256 i = 0; i < len; ) {
   344:                      IERC20 rewardAddress = rewardTokens[i].token;
   345:
   346:                      uint256 netReward = 0;
   347:                      uint256 claimedBalance = rewardAddress.balanceOf(address(this));
   348:                      if (claimedBalance != 0) {
   349:                          uint256 collectedFee = claimedBalance.mulDiv(rewardTokens[i].fee, 10000, Math.Rounding.Ceil);
   350:
   351:@--->                     if (TokenHelper.attemptSafeTransfer(address(rewardAddress), feeRecipient, collectedFee, false)) {
   352:                              rewardTokens[i].accumulatedFeeAccounted += collectedFee;
   353:                              netReward = claimedBalance - collectedFee;
   354:                              emit Harvested(_vault, netReward);
   355:                          }
   356:
   357:                          // rewardAddress.safeTransfer(_vault, netReward);
   358:@--->                     TokenHelper.attemptSafeTransfer(address(rewardAddress), _vault, netReward, false);
   359:                      }
   360:
   361:                      returnedRewards[i] = ReturnedRewards(address(rewardAddress), netReward);
   362:                      unchecked {
   363:                          ++i;
   364:                      }
   365:                  }
   366:                  return returnedRewards;
   367:              }
```

## Impact
Imagine the following flow of events:
- `harvestRewards()` is called for a list of tokens. For simplicity, we can imagine a single token i.e. `rewardTokens.length = 1`.
- Protocol deducts fee on L351. Assume this `Call1` to `TokenHelper.attemptSafeTransfer()` works just fine.
- On L358 `Call2` happens to `TokenHelper.attemptSafeTransfer()` to transfer `netReward` to the `_vault`. This fails. Since the last parameter is specified as `false`, it won't revert but **_will_** [return a boolean](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/node_modules/%40blueprint-finance/hub-and-spokes-libraries/src/libraries/TokenHelper.sol#L19) `false`. 
- Since this boolean is never checked, the execution continues and the protocol stores on L361 that `returnedRewards[i] = ReturnedRewards(address(rewardAddress), netReward);` even though the transfer never happened. As a result of this inside [ConcreteMultiStrategyVault::harvestRewards()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L1014) the `rewardIndex[]` array is updated incorrectly.

What if the admin notices it (somehow) and retries the call?<br>
Let's suppose both `Call1` and `Call2` succeed this time. This causes :
1. The fee to be deducted again on L351 of `StrategyBase.sol`.
2. The reward is incremented once again on L1014 of `ConcreteMultiStrategyVault.sol`.

## Recommendation
Capture the return vaules of `TokenHelper.attemptSafeTransfer()` on L351 & L358 and do the following: 
| `Call1` returned | `Call2` returned | Action                                                       |
|------------------|------------------|--------------------------------------------------------------|
| True             | True             | Proceed as usual                                             |
| True             | False            | Transfer the fee back and do not increment the rewardIndex   |
| False            | -                | Do not attempt `Call2` if `Call1` returned `false`           |

In this manner, the admin can later retry calling `harvestRewards()`.

[Back to Top](#summaryTable)

---

### <a id="m-09"></a>[M-09]
## **No way to remove rewardToken if added twice in `rewardTokens` array during initialization**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/strategies/StrategyBase.sol#L101
<br>

## Description
Inside [__StrategyBase_init()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/strategies/StrategyBase.sol#L101) `rewardTokens` array is populated but never checked for duplicate entries:
```js
  File: src/strategies/StrategyBase.sol

    70:              // slither-disable-next-line reentrancy-no-eth,reentrancy-benign,naming-convention
    71:              function __StrategyBase_init(
    72:                  IERC20 baseAsset_,
    73:                  string memory shareName_,
    74:                  string memory shareSymbol_,
    75:                  address feeRecipient_,
    76:                  uint256 depositLimit_,
    77:                  address owner_,
    78:                  RewardToken[] memory rewardTokens_,
    79:                  address vault_
    80:              ) internal initializer nonReentrant {
    81:                  // Initialize inherited contracts
    82:                  __ERC4626_init(IERC20Metadata(address(baseAsset_)));
    83:                  __ERC20_init(shareName_, shareSymbol_);
    84:                  __Ownable_init(owner_);
    85:
    86:                  // Iterate through the provided reward tokens to set them up
    87:                  if (rewardTokens_.length != 0) {
    88:                      for (uint256 i; i < rewardTokens_.length; ) {
    89:                          // Validate reward token address, current fee accounted, and high watermark
    90:                          if (address(rewardTokens_[i].token) == address(0)) {
    91:                              revert InvalidRewardTokenAddress();
    92:                          }
    93:                          if (rewardTokens_[i].accumulatedFeeAccounted != 0) {
    94:                              revert AccumulatedFeeAccountedMustBeZero();
    95:                          }
    96:
    97:                          // Approve the strategy to spend the reward token without limit
    98:                          if (!rewardTokens_[i].token.approve(address(this), type(uint256).max)) revert ERC20ApproveFail();
    99:                          // Add the reward token to the strategy's list and mark it as approved
   100:@--->                     rewardTokens.push(rewardTokens_[i]);
   101:                          rewardTokenApproved[address(rewardTokens_[i].token)] = true;
   102:                          unchecked {
   103:                              i++;
   104:                          }
   105:                      }
   106:                  }
```

## Impact
**_Ques_**: Why is this important to avoid, even if it is done by mistake and not maliciously?<br>
**_Ans_** : Because it's not possible to correct the mistake by use of [removeRewardToken()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/strategies/StrategyBase.sol#L219) or any other provided function. Calling `removeRewardToken()` will remove the first instance of the token from the array but also mark its `rewardTokenApproved[]` as `false` on L227:
```js
  File: src/strategies/StrategyBase.sol

   216:               */
   217:              function removeRewardToken(RewardToken calldata rewardToken_) external onlyOwner {
   218:                  // Ensure the reward token is approved before attempting removal.
   219:@--->             if (!rewardTokenApproved[address(rewardToken_.token)]) {
   220:@--->                 revert RewardTokenNotApproved();
   221:                  }
   222:
   223:                  rewardTokens[_getIndex(address(rewardToken_.token))] = rewardTokens[rewardTokens.length - 1];
   224:                  rewardTokens.pop();
   225:
   226:                  // Mark the reward token as not approved.
   227:@--->             rewardTokenApproved[address(rewardToken_.token)] = false;
   228:              }
```
If we attempt to call `removeRewardToken()` once again to remove the other remaining entry, it will revert due to the condition on L219. 
**We now have a token address in the `rewardTokens` array which has `rewardTokenApproved` set to `false` and impossible to remove.**

Note that additionally, [modifyRewardFeeForRewardToken()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/strategies/StrategyBase.sol#L238-L239) can't be called too due to a revert because of a similar check:
```js
  File: src/strategies/StrategyBase.sol

   236:              function modifyRewardFeeForRewardToken(uint256 newFee_, RewardToken calldata rewardToken_) external onlyOwner {
   237:                  // Ensure the reward token is approved before attempting to modify its fee.
   238:@--->             if (!rewardTokenApproved[address(rewardToken_.token)]) {
   239:@--->                 revert RewardTokenNotApproved();
   240:                  }
   241:
   242:                  // Find the index of the reward token to modify.
   243:                  uint256 index = _getIndex(address(rewardToken_.token));
   244:
   245:                  // Update the fee for the specified reward token.
   246:                  rewardTokens[index].fee = newFee_;
   247:              }
```

## Mitigation 
Add the following lines to skip adding duplicates:
```diff
  File: src/strategies/StrategyBase.sol

    70:              // slither-disable-next-line reentrancy-no-eth,reentrancy-benign,naming-convention
    71:              function __StrategyBase_init(
    72:                  IERC20 baseAsset_,
    73:                  string memory shareName_,
    74:                  string memory shareSymbol_,
    75:                  address feeRecipient_,
    76:                  uint256 depositLimit_,
    77:                  address owner_,
    78:                  RewardToken[] memory rewardTokens_,
    79:                  address vault_
    80:              ) internal initializer nonReentrant {
    81:                  // Initialize inherited contracts
    82:                  __ERC4626_init(IERC20Metadata(address(baseAsset_)));
    83:                  __ERC20_init(shareName_, shareSymbol_);
    84:                  __Ownable_init(owner_);
    85:
    86:                  // Iterate through the provided reward tokens to set them up
    87:                  if (rewardTokens_.length != 0) {
    88:                      for (uint256 i; i < rewardTokens_.length; ) {
    89:                          // Validate reward token address, current fee accounted, and high watermark
    90:                          if (address(rewardTokens_[i].token) == address(0)) {
    91:                              revert InvalidRewardTokenAddress();
    92:                          }
    93:                          if (rewardTokens_[i].accumulatedFeeAccounted != 0) {
    94:                              revert AccumulatedFeeAccountedMustBeZero();
    95:                          }
+   96:                          if (rewardTokenApproved[address(rewardTokens_[i].token)]) {   // no need to add if already approved
+   97:                             unchecked {
+   98:                               i++;
+   99:                             }
+  100:                             continue;
+  101:                          }    
    96:
    97:                          // Approve the strategy to spend the reward token without limit
    98:                          if (!rewardTokens_[i].token.approve(address(this), type(uint256).max)) revert ERC20ApproveFail();
    99:                          // Add the reward token to the strategy's list and mark it as approved
   100:                          rewardTokens.push(rewardTokens_[i]);
   101:                          rewardTokenApproved[address(rewardTokens_[i].token)] = true;
   102:                          unchecked {
   103:                              i++;
   104:                          }
   105:                      }
   106:                  }
```

[Back to Top](#summaryTable)
---

### <a id="m-10"></a>[M-10]
## **Call to `pauseAllVaults()` fails if any vault is already in paused state**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/managers/VaultManager.sol#L47-L73
<br>

## Description
The functions [pauseAllVaults() and unpauseAllVaults()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/managers/VaultManager.sol#L47-L73) are meant to be typically called in emergency situations, specially `pauseAllVaults()`. However, OZ's `_pause()`, which is internally called by `pauseAllVaults()` enforces the condition that the vault must not already be paused, and is checked via the modifier `whenNotPaused` as can be seen [here](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/utils/PausableUpgradeable.sol#L122). Similar condition of `whenPaused` exists inside [_unpause()](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/utils/PausableUpgradeable.sol#L135).

Hence if a vault had been individually paused previously, `pauseAllVaults()` will fail. **Note that there's no function inside the protocol currently to retrieve a list of all vaults which are currently in paused state.** As a result, custom logic will have to be written & deployed first to retrieve list of all vaults, check if they are in paused state and exclude them from the list OR unpause them before calling `pauseAllVaults()`. 

In an emergency situation, this is valuable time lost.  

## Impact
Not possible to pause or unpause all vaults in emergency situations if one of them is already in paused or unpaused state.

## Proof of Concept
We will modify [this pre-existing test](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/test/VaultManager.t.sol#L136) inside `test/VaultManager.t.sol` to see it fail when run with `forge test --mt test_pauseAllVaults -vv`:
```diff
    function test_pauseAllVaults() public {
        (address vault1, ) = _deployVault(false);
        (address vault2, ) = _deployVault(true);
        (address vault3, ) = _deployVault(true);

-       vm.prank(admin);
+       vm.startPrank(admin);
+       vaultManager.pauseVault(vault2); // pause a single vault first
        vaultManager.pauseAllVaults();   // @audit-issue : This will revert with error `EnforcedPause()`

        assertEq(ConcreteMultiStrategyVault(vault1).paused(), true, "Vault 1 should be paused");
        assertEq(ConcreteMultiStrategyVault(vault2).paused(), true, "Vault 2 should be paused");
        assertEq(ConcreteMultiStrategyVault(vault3).paused(), true, "Vault 3 should be paused");
    }
```

Output:
```text
Ran 1 test for test/VaultManager.t.sol:VaultManagerTest
[FAIL. Reason: EnforcedPause()] test_pauseAllVaults() (gas: 18508720)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 10.13ms (5.88ms CPU time)
```

## Mitigation 
Add the following conditions:
```diff
File: src/managers/VaultManager.sol

    function pauseAllVaults() external onlyRole(VAULT_MANAGER_ROLE) {
        address[] memory vaults = vaultRegistry.getAllVaults();
        emit AllVaultsPaused();
        uint256 vaultsLength = vaults.length;
        for (uint256 i = 0; i < vaultsLength; ) {
            //We control both the length of the array and the external call
            // slither-disable-next-line calls-loop
-           IConcreteMultiStrategyVault(vaults[i]).pause();
+           if (!IConcreteMultiStrategyVault(vaults[i]).paused()) IConcreteMultiStrategyVault(vaults[i]).pause();
            unchecked {
                i++;
            }
        }
    }

    function unpauseAllVaults() external onlyRole(VAULT_MANAGER_ROLE) {
        emit AllVaultsUnpaused();
        address[] memory vaults = vaultRegistry.getAllVaults();
        uint256 vaultsLength = vaults.length;
        for (uint256 i = 0; i < vaultsLength; ) {
            //We control both the length of the array and the external call
            // slither-disable-next-line calls-loop
-           IConcreteMultiStrategyVault(vaults[i]).unpause();
+           if (IConcreteMultiStrategyVault(vaults[i]).paused()) IConcreteMultiStrategyVault(vaults[i]).unpause();
            unchecked {
                i++;
            }
        }
    }
```

[Back to Top](#summaryTable)
---

### <a id="m-11"></a>[M-11]
## **Token once unregistered from registry can't be ever registered again**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/registries/TokenRegistry.sol#L62
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/registries/TokenRegistry.sol#L81-L86
<br>

## Description & Impact
The [unregisterToken()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/registries/TokenRegistry.sol#L81-L86) function can be used to unregister a registered token. This is different from the [removeToken()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/registries/TokenRegistry.sol#L70) function in the sense that we only set `_token[tokenAddress_].isRegistered = false` and it still remains in the `_listedTokens` **Enumerable.AddressSet**. A fair assumption would be that the owner has not used `removeToken()` and rather used `unregisterToken()` because they want to later have the option of calling [registerToken()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/registries/TokenRegistry.sol#L45) on the same token address. This is however not possible due to the nature of an **Enumerable.AddressSet**.

OZ's [documentation](https://docs.openzeppelin.com/contracts/3.x/api/utils#EnumerableSet-add-struct-EnumerableSet-Bytes32Set-bytes32-) outlines the following:
> **add**(struct EnumerableSet.Bytes32Set set, bytes32 value) → bool
> Add a value to a set. O(1).
> Returns true if the value was added to the set, that is if it was not already present.

Now imagine the following flow:
- Owner calls `registerToken()` for a token.
- Later on, owner calls `unregisterToken()` for that token. Note that it is **not** removed from the **Enumerable.AddressSet** `_listedTokens`:
```js
  File: src/registries/TokenRegistry.sol

    81:              function unregisterToken(
    82:                  address tokenAddress_
    83:              ) external override(ITokenRegistry) onlyOwner onlyRegisteredToken(tokenAddress_) {
    84:                  _token[tokenAddress_].isRegistered = false;
    85:                  emit TokenUnregistered(tokenAddress_);
    86:              }
```
- After some time, owner tries to call `registerToken()` again for the same token (_Why? Because maybe they want to add it as a reward token by calling `updateIsReward()` and the protocol requires a reward token to be registered first_). It reverts because of this line of code on [L62](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/registries/TokenRegistry.sol#L62):
```js
    62:              if (!_listedTokens.add(tokenAddress_)) revert Errors.AdditionFail();
```
Since token address was never removed from `_listedTokens`, trying to add it again will return `false` and hence revert.

**Q:** Is there a way to circumvent this? What if the owner tries to call `removeToken()` first to solve the issue?
**A:** `removeToken()` can't be called now because it has the `onlyRegisteredToken` modifier and the unregistered token can't be passed as a param to it.

**Impact:** There's neither a way to `removeToken()` or `registerToken()` now for this token address.

## Proof of Concept
We will modify [this pre-existing test](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/test/TokenRegistry.t.sol#L116) and run it with `forge test --mt test_unregisterToken -vv` to see it revert:
```diff
    function test_unregisterToken() public {
        bool isReward = true;
        address oracleAddr = address(0x6666);
        uint8 oracleDecimals = 8;
        string memory pair = "ASSET/STABLE";
        vm.startPrank(admin);
        tokenRegistry.registerToken(address(token), isReward, oracleAddr, oracleDecimals, pair);
        vm.expectEmit(false, true, true, true, address(tokenRegistry));
        emit TokenRegistryEvents.TokenUnregistered(address(token));
        tokenRegistry.unregisterToken(address(token));
-       vm.stopPrank();
        // check whether token is registered. Assert False that it is not registered
        assertEq(tokenRegistry.isRegistered(address(token)), false, "Token is registered");
        // check whether token is reward. Assert True that it is reward
        assertEq(tokenRegistry.isRewardToken(address(token)), true, "Token is not reward");
        address[] memory tokens = new address[](1);
        tokens[0] = address(token);
        assertEq(tokenRegistry.getTokens(), tokens, "Tokens array is not equal to the expected array");
+       tokenRegistry.registerToken(address(token), isReward, oracleAddr, oracleDecimals, pair);  // @audit-issue : reverts if owner tries to register again
    }
```

Output:
```text
Ran 1 test for test/TokenRegistry.t.sol:TokenRegistryTest
[FAIL. Reason: AdditionFail()] test_unregisterToken() (gas: 170847)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 2.40ms (431.23µs CPU time)
```

## Mitigation 
While there may be multiple ways to handle this, a simple solution could be to add a new function named `removeUnregisteredToken()`. Note that the function signature needs to be added in `ITokenRegistry` too:
```js
    function removeUnregisteredToken(address tokenAddress_) external override(ITokenRegistry) onlyOwner {
        require(!_token[tokenAddress_].isRegistered);
        if (!_listedTokens.remove(tokenAddress_)) revert Errors.RemoveFail();
        emit TokenRemoved(tokenAddress_);
    }
```

[Back to Top](#summaryTable)
---

### <a id="m-12"></a>[M-12]
## **A token can be a reward token and yet unregistered**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/registries/TokenRegistry.sol#L113
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/registries/TokenRegistry.sol#L81
<br>

## Description & Impact
The [updateIsReward()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/registries/TokenRegistry.sol#L113) function clearly specifies that "unregistered tokens cannot be rewards". If one had to guess, most probably unregistered tokens cannot be reward tokens because the system requires a formal registration process to ensure that only valid and approved tokens are used as rewards. This registration process likely involves verifying certain properties of the token to ensure it meets specific standards or criteria set by the platform:
```js
  File: src/registries/TokenRegistry.sol

   111:              function updateIsReward(address tokenAddress_, bool isReward_) external override(ITokenRegistry) onlyOwner {
   112:                  if (!isRegistered(tokenAddress_) && isReward_) {
   113:@--->                 revert Errors.UnregisteredTokensCannotBeRewards(tokenAddress_); // check if token is registered
   114:                  }
   115:                  _token[tokenAddress_].isReward = isReward_;
   116:                  emit IsRewardUpdated(tokenAddress_, isReward_);
   117:              }
```

This condition can however be bypassed by the owner, either maliciously or by an unfortunate ordering of function calls:
- Step1: Register token as a reward token via `registerToken()`. OR register it as a non-reward token and later call `updateIsReward()` to update it as one.
- Step2: Call `unregisterToken()`.

**Impact:** The token is now an unregistered reward token. It was unregistered as it was not seen fit any more, but it continues to operate as a reward token.

This mistake can be remedied however **if the owner notices the error**. They can call `updateIsReward(tokenAddress, false)`.

## Proof of Concept
We will modify [this pre-existing test](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/test/TokenRegistry.t.sol#L150) and run it with `forge test --mt test_updateIsReward -vv` to see it pass:
```diff
    function test_updateIsReward() public {
        bool isReward = true;
        address oracleAddr = address(0x6666);
        uint8 oracleDecimals = 8;
        string memory pair = "ASSET/STABLE";
        vm.startPrank(admin);
        tokenRegistry.registerToken(address(token), isReward, oracleAddr, oracleDecimals, pair);
-       vm.expectEmit(false, true, true, true, address(tokenRegistry));
-       emit TokenRegistryEvents.IsRewardUpdated(address(token), false);
-       tokenRegistry.updateIsReward(address(token), false);
-       vm.stopPrank();
-       // check whether token is reward. Assert False that it is not reward
-       assertEq(tokenRegistry.isRewardToken(address(token)), false, "Token is reward");
        assertEq(tokenRegistry.isRewardToken(address(token)), true, "Token is not reward");
+       tokenRegistry.unregisterToken(address(token));
+       // check whether token is registered. 
+       assertEq(tokenRegistry.isRegistered(address(token)), false, "Token is registered");
+       // check whether token is still a reward token.
+       assertEq(tokenRegistry.isRewardToken(address(token)), true, "Token is not a reward token");
    }
```

## Mitigation 
Do not allow to unregister a reward token. This will ensure that the owner always sets `isReward` to `false` inside `updateIsReward()` first and then calls `unregisterToken()`:
```diff
  File: src/registries/TokenRegistry.sol

    81:              function unregisterToken(
    82:                  address tokenAddress_
    83:              ) external override(ITokenRegistry) onlyOwner onlyRegisteredToken(tokenAddress_) {
+   84:                  require(!isRewardToken(tokenAddress_), "cannot unregister reward token");
    84:                  _token[tokenAddress_].isRegistered = false;
    85:                  emit TokenUnregistered(tokenAddress_);
    86:              }
```

[Back to Top](#summaryTable)
---

### <a id="m-13"></a>[M-13]
## **Use of strict equality disallows adding new strategy**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L881
<br>

## Description
Inside [changeAllocations()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L881) function the code reverts if the sum of all strategy's allocation is not equal to `100%`:
```js
  File: src/vault/ConcreteMultiStrategyVault.sol

   865:              function changeAllocations(
   866:                  Allocation[] calldata allocations_,
   867:                  bool redistribute
   868:              ) external nonReentrant onlyOwner takeFees {
   869:                  uint256 len = allocations_.length;
   870:          
   871:                  if (len != strategies.length) revert InvalidLength(len, strategies.length);
   872:          
   873:                  uint256 allotmentTotals = 0;
   874:                  for (uint256 i; i < len; ) {
   875:                      allotmentTotals += allocations_[i].amount;
   876:                      strategies[i].allocation = allocations_[i];
   877:                      unchecked {
   878:                          i++;
   879:                      }
   880:                  }
   881:@--->             if (allotmentTotals != 10000) revert AllotmentTotalTooHigh();
   882:          
   883:                  if (redistribute) {
   884:                      pullFundsFromStrategies();
   885:                      pushFundsToStrategies();
   886:                  }
   887:          
   888:                  emit StrategyAllocationsChanged(allocations_);
   889:              }
```

This is clearly inconsistent with the approach taken across the protocol under other functions and hence seems like a mistake. Here are the other instances which easily allow allocations to be less than `100%`:
- `addOrReplaceStrategy()` only checks that the allocation sum isn't over `100%` [here](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/libraries/MultiStrategyVaultHelper.sol#L242) and [here](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/libraries/MultiStrategyVaultHelper.sol#L229-L230).
- [removeStrategy()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/libraries/MultiStrategyVaultHelper.sol#L268) allows directly removing a strategy which can make allocation sum less than `100%`.
- No constraint when strategies are being initiated inside [initialize()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L141).

## Impact
This blocks the capability to add a new strategy in many situations:
- Suppose vault has 2 strategies S1 and S2 with 50% allocation each. This is a pre-requisite for this bug: the sum of current allocations ought to be 100%.
- Admin wants to add a third strategy S3 and change allocations to 25%, 25% & 50% (any ratio that adds up to 100%). **_There's no way to do this!_**
    - Option1: Change allocation of S1 and S2 first and then add S3.
        - This won't be allowed because when we call `changeAllocations()`, the allocations should add up to exactly 100%. Hence reverts.
    - Option2: Add S3 first and then change allocation of S1, S2 and S3.
        - This won't be allowed because when we call `addStrategy()`, the allocations add up to be greater than 100%. Hence reverts.
- The only (ugly) way is to replace/remove an existing strategy and then add a new one. This works because allocation does not exceed 100% upon removing/replacing a strategy. **_Big downside_** though: This triggers a redeem of all existing strategy funds at the time of removal/replacement. This might not be desirable in most cases.

## Mitigation 
Change `!=` to `>`:
```diff
  File: src/vault/ConcreteMultiStrategyVault.sol

   865:              function changeAllocations(
   866:                  Allocation[] calldata allocations_,
   867:                  bool redistribute
   868:              ) external nonReentrant onlyOwner takeFees {
   869:                  uint256 len = allocations_.length;
   870:          
   871:                  if (len != strategies.length) revert InvalidLength(len, strategies.length);
   872:          
   873:                  uint256 allotmentTotals = 0;
   874:                  for (uint256 i; i < len; ) {
   875:                      allotmentTotals += allocations_[i].amount;
   876:                      strategies[i].allocation = allocations_[i];
   877:                      unchecked {
   878:                          i++;
   879:                      }
   880:                  }
-  881:                  if (allotmentTotals != 10000) revert AllotmentTotalTooHigh();
+  881:                  if (allotmentTotals > 10000) revert AllotmentTotalTooHigh();
   882:          
   883:                  if (redistribute) {
   884:                      pullFundsFromStrategies();
   885:                      pushFundsToStrategies();
   886:                  }
   887:          
   888:                  emit StrategyAllocationsChanged(allocations_);
   889:              }
```

[Back to Top](#summaryTable)
---

### <a id="m-14"></a>[M-14]
## **Claim status not updated if `_availableAssets` exactly equals requested `amount`**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/queue/WithdrawalQueue.sol#L170
<br>

## Description
If the `amount` in the withdrawal request queue for a particular requestId is exactly equal to the `_avaliableAssets`, the function [prepareWithdrawal()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/queue/WithdrawalQueue.sol#L170) does not update the `request.claimed` to `true` after transferring requested withdrawal `amount`. It also does not remove the `_requestId` from `_requestsByOwner[recipient]`. This is due to use of `>` instead of `>=` [here](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/queue/WithdrawalQueue.sol#L170):
```js
  File: src/queue/WithdrawalQueue.sol

   153:              function prepareWithdrawal(
   154:                  uint256 _requestId,
   155:                  uint256 _avaliableAssets
   156:              ) external onlyOwner returns (address recipient, uint256 amount, uint256 avaliableAssets) {
   157:                  if (_requestId == 0) revert InvalidRequestId(_requestId);
   158:                  if (_requestId < lastFinalizedRequestId) revert RequestNotFoundOrNotFinalized(_requestId);
   159:          
   160:                  WithdrawalRequest storage request = _requests[_requestId];
   161:          
   162:                  if (request.claimed) revert RequestAlreadyClaimed(_requestId);
   163:          
   164:                  recipient = request.recipient;
   165:          
   166:                  WithdrawalRequest storage prevRequest = _requests[_requestId - 1];
   167:          
   168:                  amount = request.cumulativeAmount - prevRequest.cumulativeAmount;
   169:          
   170:@--->             if (_avaliableAssets > amount) {
   171:                      assert(_requestsByOwner[recipient].remove(_requestId));
   172:                      avaliableAssets = _avaliableAssets - amount;
   173:                      request.claimed = true;
   174:                      //This is commented to fit the requirements of the vault
   175:                      //instead of this we will call _withdrawStrategyFunds
   176:                      //IERC20(TOKEN).safeTransfer(recipient, realAmount);
   177:          
   178:                      emit WithdrawalClaimed(_requestId, recipient, amount);
   179:                  }
   180:              }
```

Even though `avaliableAssets` is never explicitly set to zero, it still remains as zero as it's never touched after initialization thus returning the correct value as it ought to.

## Impact
Although the overall accounting is not affected by it, the fact that `request.claimed` is still `false` and that the number of requests by the recipient still shows up as 1 instead of zero, may later on lead to disputes or discrepancies. 
**It's also important to note that the `WithdrawalClaimed()` event is never emitted in this case** and hence any contract or monitoring service relying on it for accurate records would suffer, specially if they reside on the other layer and depend on it.

## Proof of Concept
Add the following patch to include the test inside `test/ConcreteMultiStrategyVault.t.sol` and run it to see it fail via `forge test --mt test_t0x1c_prepareWithdrawal_bug -vv`:
```diff
diff --git a/ConcreteMultiStrategyVault.t.sol b/ConcreteMultiStrategyVault.t.sol
index bbd2af9..e9d0951 100644
--- a/ConcreteMultiStrategyVault.t.sol
+++ b/ConcreteMultiStrategyVault.t.sol
@@ -16,7 +16,7 @@ import {IStrategy, ReturnedRewards} from "../src/interfaces/IStrategy.sol";
 import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
 import {Errors} from "../src/interfaces/Errors.sol";
 import {WithdrawalQueue} from "../src/queue/WithdrawalQueue.sol";
-
+import {console2 as console} from "forge-std/console2.sol";
 contract ConcreteMultiStrategyVaultTest is Test {
     using Math for uint256;
 
@@ -2079,6 +2079,36 @@ contract ConcreteMultiStrategyVaultTest is Test {
         assertEq(claimedRewards[1].rewardAmount, 20000000, "Hazel reward 1b address");
         assertEq(claimedRewards[2].rewardAmount, 10000000, "Hazel reward 2 address");
     }
+
+    function test_t0x1c_prepareWithdrawal_bug() public {
+        (ConcreteMultiStrategyVault newVault, WithdrawalQueue queue) = createQueue_t0x1c(5 ether);
+        
+        console.log("\nInitial state:");
+        console.log("Total withdrawable assets:", newVault.getAvailableAssetsForWithdrawal()); // 5 ether
+        console.log("Hazel balance:", asset.balanceOf(hazel)); // 0
+
+        console.log("Unfinalized amount:", newVault.getUnfinalizedAmount()); // 5 ether
+        console.log("Withdrawal Requests Count:", queue.getWithdrawalRequests(hazel).length); // 1
+        uint256[] memory reqIds = new uint256[](1);
+        reqIds[0] = 1;
+        WithdrawalQueue.WithdrawalRequestStatus[] memory reqStatuses = queue.getWithdrawalStatus(reqIds);
+        bool claimStatus = reqStatuses[0].isClaimed;
+        console.log("Withdrawal Request claimed?", claimStatus); // false
+        
+        console.log("\nBatchClaimWithdrawal call:");
+        vm.prank(admin);
+        newVault.batchClaimWithdrawal(1);
+
+        console.log("\nAfter batchClaimWithdrawal:");
+        console.log("Unfinalized amount:", newVault.getUnfinalizedAmount()); // 0 ether
+        reqStatuses = queue.getWithdrawalStatus(reqIds);
+        claimStatus = reqStatuses[0].isClaimed;
+        console.log("Total withdrawable assets:", newVault.getAvailableAssetsForWithdrawal()); // 0 ether  <--- correct
+        console.log("Hazel balance:", asset.balanceOf(hazel)); // 5 ether  <--- correct
+        console.log("Withdrawal Requests Count:", queue.getWithdrawalRequests(hazel).length); // 1  <--- @audit-issue : should have been 0
+        console.log("Withdrawal Request claimed?", claimStatus); // false  <--- @audit-issue : should have been true
+        assertTrue(claimStatus, "claimStatus not updated to TRUE");
+    }
     // ============= UTILITIES =========================
 
     function _createMockStrategy(IERC20 asset_, bool decreased_) internal returns (Strategy memory) {
@@ -2198,4 +2228,36 @@ contract ConcreteMultiStrategyVaultTest is Test {
             }
         }
     }
-}
+
+    function createQueue_t0x1c(
+        uint256 amount_
+    )
+        public
+        returns (
+            ConcreteMultiStrategyVault _newVault,
+            WithdrawalQueue queue
+        )
+    {
+        (ConcreteMultiStrategyVault newVault, Strategy[] memory strats) = _createNewVault(false, false, true);
+        _newVault = newVault;
+
+        queue = new WithdrawalQueue(address(newVault));
+        vm.prank(admin);
+        newVault.setWithdrawalQueue(address(queue));
+
+        uint256 hazelsDespositedAmount = amount_;
+        asset.mint(hazel, hazelsDespositedAmount);
+        vm.startPrank(hazel);
+        asset.approve(address(newVault), hazelsDespositedAmount);
+        newVault.deposit(hazelsDespositedAmount, hazel);
+        newVault.withdraw(amount_, hazel, hazel);
+        vm.stopPrank();
+
+        for (uint256 i = 0; i < strats.length; ) {
+            IMockStrategy(address(strats[i].strategy)).setAvailableAssetsZero(false);
+            unchecked {
+                i++;
+            }
+        }
+    }
+}
```

## Mitigation 
Change `>` to `>=`:
```diff
  File: src/queue/WithdrawalQueue.sol

   153:              function prepareWithdrawal(
   154:                  uint256 _requestId,
   155:                  uint256 _avaliableAssets
   156:              ) external onlyOwner returns (address recipient, uint256 amount, uint256 avaliableAssets) {
   157:                  if (_requestId == 0) revert InvalidRequestId(_requestId);
   158:                  if (_requestId < lastFinalizedRequestId) revert RequestNotFoundOrNotFinalized(_requestId);
   159:          
   160:                  WithdrawalRequest storage request = _requests[_requestId];
   161:          
   162:                  if (request.claimed) revert RequestAlreadyClaimed(_requestId);
   163:          
   164:                  recipient = request.recipient;
   165:          
   166:                  WithdrawalRequest storage prevRequest = _requests[_requestId - 1];
   167:          
   168:                  amount = request.cumulativeAmount - prevRequest.cumulativeAmount;
   169:          
-  170:                  if (_avaliableAssets > amount) {
+  170:                  if (_avaliableAssets >= amount) {
   171:                      assert(_requestsByOwner[recipient].remove(_requestId));
   172:                      avaliableAssets = _avaliableAssets - amount;
   173:                      request.claimed = true;
   174:                      //This is commented to fit the requirements of the vault
   175:                      //instead of this we will call _withdrawStrategyFunds
   176:                      //IERC20(TOKEN).safeTransfer(recipient, realAmount);
   177:          
   178:                      emit WithdrawalClaimed(_requestId, recipient, amount);
   179:                  }
   180:              }
```

[Back to Top](#summaryTable)
---

### <a id="m-15"></a>[M-15]
## **Lack of slippage protection & deadline can lead to loss**
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L351
#### https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/swapper/Swapper.sol#L78-L82
<br>

## Description
Throughout the protocol, functions like [swapTokensForReward()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/swapper/Swapper.sol#L78-L82), [redeem()](https://code4rena.com/evaluate/2024-11-concrete/submissions/S-2024-11-concrete/blob/main/src/vault/ConcreteMultiStrategyVault.sol#L351), `deposit()`, `mint()`, etc. are implemented without any slippage protection or deadline support. In general, as [this informative article](https://dacian.me/defi-slippage-attacks#heading-minting-exposes-users-to-unlimited-slippage) quite nicely outlines, situations or functions where transfer of tokens may be taking place may be masked under different names, and hence should always incorporate slippage protection & deadline params.

## Impact
Absence of a deadline means that if the transaction does not get executed immediately, which is quite common, or other large transactions front-run the current transaction, then a rate fluctuation can result in the caller either paying more in term of assets or receiving lesser shares than anticipated. 

## Recommendation
Allow the caller to specify `minExpectedAmount` or `minShares` alongwith a `deadline`.

[Back to Top](#summaryTable)
