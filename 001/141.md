Delightful Flint Armadillo

medium

# Lack of slippage protection in `addPrincipalToCommitmentGroup` and `burnSharesToWithdrawEarnings` functions

## Summary
`LenderCommitmentGroup_Smart` is a contract that acts as it's own loan committment, which has liquidity pools with `principal token` and `collateral token`. Users can deposit `principal tokens` in exchange for `share tokens` and burn `share tokens` in exchange for `principal tokens`. When depositing/burning, the amount sent to the user is relative to the exchange rate.

The problem is that when these functions are called, the exchange rate can change while it is still in the mempool. In this case, users may experience loss of funds.

## Vulnerability Detail
When users want to add principal tokens to the pool, they call the following function:

`LenderCommitmentGroup_Smart::addPrincipalToCommitmentGroup` 
```javascript
    function addPrincipalToCommitmentGroup(
        uint256 _amount,
        address _sharesRecipient
    ) external returns (uint256 sharesAmount_) {

        // @audit PT transferred from user
        principalToken.transferFrom(msg.sender, address(this), _amount);

        // @audit this returns shares for user
        sharesAmount_ = _valueOfUnderlying(_amount, sharesExchangeRate());

        // @audit this is used for pool estimated value
        totalPrincipalTokensCommitted += _amount;

        // @audit shares minted to user here
        poolSharesToken.mint(_sharesRecipient, sharesAmount_);
    }
```

The shares minted to the user is calculated by the following exchange rate calculation:

```javascript
    function sharesExchangeRate() public view virtual returns (uint256 rate_) {
@>      uint256 poolTotalEstimatedValue = getPoolTotalEstimatedValue();

        if (poolSharesToken.totalSupply() == 0) {
            return EXCHANGE_RATE_EXPANSION_FACTOR; // 1 to 1 for first swap
        }

@>      rate_ =
            (poolTotalEstimatedValue * EXCHANGE_RATE_EXPANSION_FACTOR) /
            poolSharesToken.totalSupply();
    }
```

```javascript
    function getPoolTotalEstimatedValue()
        public
        view
        returns (uint256 poolTotalEstimatedValue_)
    {
        // @audit Here we keep internal accounting of PT
@>      int256 poolTotalEstimatedValueSigned = int256(
            totalPrincipalTokensCommitted
        ) +
            int256(totalInterestCollected) +
            int256(tokenDifferenceFromLiquidations) -
            int256(totalPrincipalTokensWithdrawn);

        //if the poolTotalEstimatedValue_ is less than 0, we treat it as 0.
        poolTotalEstimatedValue_ = poolTotalEstimatedValueSigned > int256(0)
            ? uint256(poolTotalEstimatedValueSigned)
            : 0;
    }
```

```javascript
    function _valueOfUnderlying(
        uint256 amount,
        uint256 rate
    ) internal pure returns (uint256 value_) {
        if (rate == 0) {
            return 0;
        }

@>      value_ = (amount * EXCHANGE_RATE_EXPANSION_FACTOR) / rate;
    }
```

As you can see, the amount of shares minted is relative to the exchange rate at that time, which can change prior to the execution of the call (while it's in the mempool). The same applies to burning shares:

```javascript
    function burnSharesToWithdrawEarnings(
        uint256 _amountPoolSharesTokens,
        address _recipient
    ) external returns (uint256) {
        poolSharesToken.burn(msg.sender, _amountPoolSharesTokens);

 @>     uint256 principalTokenValueToWithdraw = _valueOfUnderlying(
            _amountPoolSharesTokens,
            sharesExchangeRateInverse()
        );

        totalPrincipalTokensWithdrawn += principalTokenValueToWithdraw;

@>      principalToken.transfer(_recipient, principalTokenValueToWithdraw);

        return principalTokenValueToWithdraw;
    }
```


## Impact
Lack of slippage protection, allowing possibility of front-run/sandwich attacks. Loss of funds for users.

## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L307

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L262-L275

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L288-L302

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L324-L334

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L396

## Tool used
Manual Review

## Recommendation
Incorporate slippage protection for `addPrincipalToCommitmentGroup` and `burnSharesToWithdrawEarnings` functions