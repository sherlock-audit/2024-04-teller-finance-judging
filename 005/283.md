Straight Crepe Blackbird

medium

# Function getSqrtTwapX96 doesn't round to negative infinity for negative ticks

## Summary
Function `getSqrtTwapX96()` doesn't round to negative infinity for negative ticks and may return wrong sqrt price.
## Vulnerability Detail
Function `getSqrtTwapX96()` is used for calculating sqrt price based on `tickCumulatives[]` array from `IUniswapV3Pool.observe()`. The problem arises in these lines:
```solidity
sqrtPriceX96 = TickMath.getSqrtRatioAtTick(  
                int24(
                    (tickCumulatives[1] - tickCumulatives[0]) / 
                        int32(twapInterval)
                )
            );
```
They differ from Uniswap V3 implementation. When the UniV3Oracle calls `OracleLibrary.consult()`, it returns the `arithmeticMeanTick`. This value is rounded DOWN to the nearest tick. This is because, in most use cases, the price being returned by an oracle is used to determine the value of an asset to be used for valuing collateral, where the caller is the one whose collateral is on the line, and it is crucial to ensure that user assets are not overvalued so as to give them an edge.
The above-mentioned lines of code are exactly necessary for the calculation of required collateral amount in `acceptFundsForAcceptBid()` function.
The problem is that in case if `int24(tickCumulatives[1] - tickCumulatives[0])` is negative, then the tick should be rounded down as it's done in the [uniswap library](https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/OracleLibrary.sol#L36). In this case returned tick will be bigger then it should be.
## Impact
Incorrect calculation of `requiredCollateral` may force `acceptFundsForAcceptBid()` function revert.
## Code Snippet
[https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L363](https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L363)
[https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L580-L594](https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L580-L594)
## Tool used

Manual Review

## Recommendation
```diff
uint32[] memory secondsAgos = new uint32[](2);
            secondsAgos[0] = twapInterval; // from (before)
            secondsAgos[1] = 0; // to (now)

            (int56[] memory tickCumulatives, ) = IUniswapV3Pool(UNISWAP_V3_POOL)
                .observe(secondsAgos);

+          int56 tickCumulativesDelta = tickCumulatives[1] - tickCumulatives[0];
+          int24 arithmeticMeanTick = int24(tickCumulativesDelta / twapInterval);
             //// Always round to negative infinity
+          if (tickCumulativesDelta < 0 && (tickCumulativesDelta % twapInterval != 0)) arithmeticMeanTick--;
            // tick(imprecise as it's an integer) to price
-            sqrtPriceX96 = TickMath.getSqrtRatioAtTick(  
-                int24(
-                    (tickCumulatives[1] - tickCumulatives[0]) / 
-                        int32(twapInterval)
-                )
+          sqrtPriceX96 = TickMath.getSqrtRatioAtTick(arithmeticMeanTick);
```
