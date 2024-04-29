Delightful Flint Armadillo

high

# `LenderCommitGroup_Smart::acceptFundsForAcceptBid` will return incorrect amount of required collateral if pool token decimals don't match

## Summary
`LenderCommitmentGroup_Smart` is a contract that acts as it's own loan committment, which has liquidity pools with `principal token` and `collateral token`. Users can deposit `principal tokens` in exchange for `share tokens`.

Due to an issue regarding decimal precision, the calculation of `required collateral` to ensure `borrower` has specified sufficient amount of collateral, will be incorrect.

## Vulnerability Detail
`LenderCommitmentGroup_Smart::acceptFundsForAcceptBid` [#L336](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L336)
```javascript
   function acceptFundsForAcceptBid(
        address _borrower,
        uint256 _bidId,
        uint256 _principalAmount,
        uint256 _collateralAmount,
        address _collateralTokenAddress,
        uint256 _collateralTokenId,
        uint32 _loanDuration,
        uint16 _interestRate
    ) external onlySmartCommitmentForwarder whenNotPaused {
        require(
            _collateralTokenAddress == address(collateralToken),
            "Mismatching collateral token"
        );
        
        .
        .
        .

@>      uint256 requiredCollateral = getCollateralRequiredForPrincipalAmount(
            _principalAmount
        );

@>      require(
            (_collateralAmount * STANDARD_EXPANSION_FACTOR) >=
                requiredCollateral,
            "Insufficient Borrower Collateral"
        );
    }
```

If we follow the flow of `getCollateralRequiredForPrincipalAmount`, we execute the following:

```javascript
    function _getUniswapV3TokenPairPrice(
        uint32 _twapInterval
    ) internal view returns (uint256) {
@>      uint160 sqrtPriceX96 = getSqrtTwapX96(_twapInterval);
@>      return _getPriceFromSqrtX96(sqrtPriceX96);
    }

    function _getPriceFromSqrtX96(
        uint160 _sqrtPriceX96
    ) internal pure returns (uint256 price_) {
@>       uint256 priceX96 = (uint256(_sqrtPriceX96) * uint256(_sqrtPriceX96)) /
            (2 ** 96);
        price_ = priceX96;
    }

    function getSqrtTwapX96(
        uint32 twapInterval
    ) public view returns (uint160 sqrtPriceX96) {
        if (twapInterval == 0) {
@>          (sqrtPriceX96, , , , , , ) = IUniswapV3Pool(UNISWAP_V3_POOL)
                .slot0();
        } else {
            uint32[] memory secondsAgos = new uint32[](2);
            secondsAgos[0] = twapInterval; 
            secondsAgos[1] = 0;

            (int56[] memory tickCumulatives, ) = IUniswapV3Pool(UNISWAP_V3_POOL)
                .observe(secondsAgos);

@>          sqrtPriceX96 = TickMath.getSqrtRatioAtTick(
                int24(
                    (tickCumulatives[1] - tickCumulatives[0]) /
                        int32(twapInterval)
                )
            );
        }
    }
```

We can see from above that there is no scaling involved to account for the decimal precision of the respective tokens.

The `sqrtPriceX96` value (ratio of pool) returned by `_getUniswapV3TokenPairPrice()` is used for the `priceToken1PerToken0` parameter here: 

```javascript
    function token0ToToken1(
        uint256 amountToken0,
        uint256 priceToken1PerToken0
    ) internal pure returns (uint256) {
        return
            MathUpgradeable.mulDiv(
                amountToken0, //@audit this is in decimals of principal token
                UNISWAP_EXPANSION_FACTOR,
                priceToken1PerToken0, //@audit ratio of pool, either using uniswap slot() or tick 
                MathUpgradeable.Rounding.Up
            );
    }

    function token1ToToken0(
        uint256 amountToken1,
        uint256 priceToken1PerToken0
    ) internal pure returns (uint256) {
        return
            MathUpgradeable.mulDiv(
                amountToken1, //@audit this is in decimals of principal token
                priceToken1PerToken0, //@audit ratio of pool, either using uniswap slot() or tick
                UNISWAP_EXPANSION_FACTOR,  
                MathUpgradeable.Rounding.Up
            );
    }
```

Depending on whether `principal token` is `token0` or `token1` of the pool, we execute the above functions.

We can see from above that the price returned will always have the same decimals as the `principal token`. Now, looking back at `acceptFundsForAcceptBid()`:

Example 1:

`principal token` = WETH (18 decimals) as `token0` of pool
`collateral token` = USDC (6 decimals) as `token1` of pool

Since `requiredCollateral` returns decimals in `principal token`, 18 decimal precision of `USDC` is used, which is highly expensive and most likely causing DoS due to insufficient collateral.

Example 2:

`collateral token` = USDC (6 decimals) as `token0` of pool
`principal token` = WETH (18 decimals) as `token1` of pool

`requiredCollateral` returns `WETH` amount with 6 decimal precison, effectively allowing users to borrow `USDC` for very little collateral.


## Impact
Depending on the decimals of token0 and token1, borrower users can borrow far more than they should be able to for the amount of collateral provided, or may need to put up much more collateral.

## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L336

## Tool used
Manual Review

## Recommendation
Ensure that the precision of the tokens are correctly accounted for when calculating the required collateral.
