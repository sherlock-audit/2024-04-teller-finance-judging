Jolly Bamboo Goose

medium

# Malicious LenderCommitmentGroup creator can set twapInterval to zero and induce price-manipulation attacks

## Summary
When initializing the LenderCommitmentGroup contract, there are not sanity checks for the twapInterval variable. This allows the contract creator to set it to zero, exposing lenders to a price-manipulation prone contract. 
## Vulnerability Detail
At the initialize function of the LenderCommitmentGroup, users can pass any value as the desired twapInterval. This means one could put very small or even zero seconds twap intervals.
This can make both the getSqrtTwapX96 and the getUniswapV3TokenPairPrice view functions return instant prices when it should not. It also enables the getCollateralRequiredForPrincipalAmount function to be based on instant prices, as it calls the calculateCollateralTokensAmountEquivalentToPrincipalTokens internal view function that relies on the twap price and on the instant price:
```solidity
uint256 pairPriceWithTwap = _getUniswapV3TokenPairPrice(twapInterval);
        uint256 pairPriceImmediate = _getUniswapV3TokenPairPrice(0);
```

If the twap interval is zero, there is no safety mechanism to avoid very common pool-price-manipulation attack vectors. 
## Impact
Since LenderCommitmentGroup contract creation is permissionless, the owner of this contract cannot be considered a TRUSTED role. 
This means a malicious owner can create contracts with zero twap intervals, have lenders deposit their principal, then acquire loans with little to none required collateral by manipulating the pool's instant price. 
As this is much cheaper and faster than manipulating a TWAP price in this edge case the existence of the extra safety mechanisms are not enough.
Since the impact is high, but the likelihood is mid-to-low, this issue is of medium severity.

Another impact would be if the malicious owner of the contract set the twapInterval to one. This would yield int56 tickCumulatives to be dangerously truncated to int24 and return unpredictable sqrtPrices: 
```solidity
sqrtPriceX96 = TickMath.getSqrtRatioAtTick(
                int24(
                    (tickCumulatives[1] - tickCumulatives[0]) /
                        int32(twapInterval)
                )
            );
```
## Code Snippet
[2024-04-teller-finance/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol at main · sherlock-audit/2024-04-teller-finance (github.com)](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L539)

[2024-04-teller-finance/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol at main · sherlock-audit/2024-04-teller-finance (github.com)](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L570)

[2024-04-teller-finance/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol at main · sherlock-audit/2024-04-teller-finance (github.com)](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L539)

## Tool used

Manual Review

## Recommendation
Make sure to only allow a minimal twap interval during the contract's initialization, as the TWAP price safety mechanics currently does not handle cases where both the TWAP and the instant price are the same.