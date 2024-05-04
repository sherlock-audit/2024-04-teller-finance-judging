Little Sapphire Lobster

medium

# `LenderCommitmentGroup_Smart` will not work for some `Uniswap` pools due to `sqrtPriceX96` overflow

## Summary

`LenderCommitmentGroup_Smart::_getPriceFromSqrtX96()` overflows when calculating `priceX96` for some pools.

## Vulnerability Detail

Uniswap `sqrtPriceX96` is the square root of the ratio between the tokens, `sqrt(token1 / token0)`, multiplied by `2**96`. Thus, when [squared](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L559), as in  `LenderCommitmentGroup_Smart::_getPriceFromSqrtX96()`, it may overflow for certain token pairs, as the math becomes `token1 / token0 ** 2^192`, which means that `token1 / token0` may only have `256 - 192 = 64` bits. If there is a mismatch between the decimal of the tokens, in which `token1` has more decimals than `token0`, the number of available bits goes down further. For example, `WBTC` and `DAI` have `8` and `18` decimals, respectively, and `WBTC` is `token0` in the pool, so the price adds `10**10 = 34` bits. The number of available bits is now `256 - 192 - 34 = 30`, so the max price between 1 unit of each token is `2^30 = 1073741824`, which may not ever happen for `WBTC` and `DAI` but some other coin with lower value could reach this `token1 / token0` ratio. Alternatively, the decimals between `token1` and `token0` may have a bigger difference, for example, if `DAI` had `24` bits, there would be `10**16 = 54` bits less due to decimals offset, having only `256 - 192 - 54 = 10` bits available for the price, or `2**10 = 1024` `token1 / token0` price.

Uniswap addresses this issue in their [periphery](https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/OracleLibrary.sol#L49) contracts, so the issue is real and should be addressed.

## Impact

Some `LenderCommitmentGroup_Smart` will not be able to lend funds due to the Uniswap price overflowing when calculating the required collateral amount.

## Code Snippet

[LenderCommitmentSmart::_getPriceFromSqrtX96()](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L559)
```solidity
function _getPriceFromSqrtX96(uint160 _sqrtPriceX96)
    internal
    pure
    returns (uint256 price_)
{
    
    uint256 priceX96 = (uint256(_sqrtPriceX96) * uint256(_sqrtPriceX96)) /
        (2**96);
    ...
}
```

## Tool used

Manual Review

Vscode

## Recommendation

Use the [code](https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/OracleLibrary.sol#L49-L69) from `OracleLibrary::getQuoteAtTick()`.