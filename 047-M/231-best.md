Unique Chartreuse Badger

medium

# Deviation in oracle price could lead to arbitrage in high LTV markets

## Summary

Deviation in oracle price could lead to arbitrage in high LTV markets.

## Vulnerability Detail

In `LenderCommitmentGroup_Smart`, the maximum amount a user can borrow is calculated with the conversion rate between two assets in a uniV3 pool:

    function getCollateralRequiredForPrincipalAmount(uint256 _principalAmount)
        public
        view
        returns (uint256)
    {
        uint256 baseAmount = _calculateCollateralTokensAmountEquivalentToPrincipalTokens(
                _principalAmount
            );

        //this is an amount of collateral
        return baseAmount.percent(collateralRatio);
    }

`_calculateCollateralTokensAmountEquivalentToPrincipalTokens` is calculated by calling uniV3 oracle's observe()function, which returns a TWAP value.

    function _calculateCollateralTokensAmountEquivalentToPrincipalTokens(
        uint256 principalTokenAmountValue
    ) internal view returns (uint256 collateralTokensAmountToMatchValue) {
        ...
        uint256 pairPriceWithTwap = _getUniswapV3TokenPairPrice(twapInterval);
        uint256 pairPriceImmediate = _getUniswapV3TokenPairPrice(0);
        ...
    }

However, Uniwap V3 TWAP update is susceptible to front-running as their prices tend to lag behind an asset's real-time price. (More specifically: Uniwap V3 TWAP returns the average price over the past X number of blocks, which means it will
always lag behind the real-time price.)

For Teller, this becomes profitable when the price deviation is sufficiently large for an attacker to open positions that become bad debt. 

## Impact

All profit gained from arbitrage causes a loss of funds for lenders as the remaining bad debt is socialized by them.

## Code Snippet

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L634-L663

## Tool used

Manual Review

## Recommendation

Consider implementing a borrowing fee to mitigate against arbitrage opportunities. Ideally, this fee would be larger than the oracle's maximum price deviation so that it is not possible to profit from arbitrage.

Multiple solutions may be studied:

Further possible mitigations have also been explored by other protocols:
• [Angle Protocol: Oracles and Front-Running](https://medium.com/angle-protocol/angle-research-series-part-1-oracles-and-front-running-d75184abc67)
• [Liquity: The oracle conundrum](https://www.liquity.org/blog/the-oracle-conundrum)
