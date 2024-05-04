Silly Linen Turtle

high

# Incorrect calculation of `LenderCommitmentGroup_Smart.getCollateralRequiredForPrincipalAmount()`.

## Summary

In `LenderCommitmentGroup_Smart.getCollateralRequiredForPrincipalAmount()`, there is a miscalculation by `collateralRatio`.

## Vulnerability Detail

At `L530` of `LenderCommitmentGroup_Smart.getCollateralRequiredForPrincipalAmount()`, `baseAmount` is the amount of collateral token that is worth as principal tokens. However, the collateral token has got the collateral ratio that falls down the value of collateral. So, the required amount of collateral token should be greater than `baseAmount`. But, as you can see at `L535`, it rather returns a smaller value, `baseAmount` multipled by `collateralRatio`.

```solidity
    function getCollateralRequiredForPrincipalAmount(uint256 _principalAmount)
        public
        view
        returns (uint256)
    {
530     uint256 baseAmount = _calculateCollateralTokensAmountEquivalentToPrincipalTokens(
                _principalAmount
            );

        //this is an amount of collateral
535     return baseAmount.percent(collateralRatio);
    }
```

As a result, the check for `_collateralAmount` at `L368` of `LenderCommitmentGroup_Smart.acceptFundsForAcceptBid()` can be bypassed with a samller `_collateralAmount` than it should be, leading to loss of funds to the lenders.

```solidity
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
        
        [...]

        //this is expanded by 10**18
363     uint256 requiredCollateral = getCollateralRequiredForPrincipalAmount(
            _principalAmount
        );

        require(
368         (_collateralAmount * STANDARD_EXPANSION_FACTOR) >=
                requiredCollateral,
            "Insufficient Borrower Collateral"
        );
 
        [...]

    }
```

## Impact

Borrowers can borrow money with a small amount of collateral than needed, resulting in loss of funds for the lenders.

## Code Snippet

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L336-L382

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L525-L536

## Tool used

Manual Review

## Recommendation

`LenderCommitmentGroup_Smart.getCollateralRequiredForPrincipalAmount()` should be fixed as follows.

```diff
    function getCollateralRequiredForPrincipalAmount(uint256 _principalAmount)
        public
        view
        returns (uint256)
    {
        uint256 baseAmount = _calculateCollateralTokensAmountEquivalentToPrincipalTokens(
                _principalAmount
            );

        //this is an amount of collateral
-       return baseAmount.percent(collateralRatio);
+       return baseAmount * 10000 / collateralRatio;
    }
```