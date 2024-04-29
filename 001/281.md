Virtual Peanut Seagull

medium

# LenderCommitmentGroup_Smart.sol#__valueOfUnderlying() - If rate = 0, then users will receive no shares if they attempt to mint, and burn won't work

## Summary
LenderCommitmentGroup_Smart.sol#__valueOfUnderlying() - If rate = 0, then users will receive no shares if they attempt to mint, and burn won't work.

## Vulnerability Detail
The protocol uses `_valueOfUnderlying` to value the shares it has to mint and the principal it has to transfer when burning share tokens.

```solidity
function _valueOfUnderlying(
        uint256 amount,
        uint256 rate
    ) internal pure returns (uint256 value_) {
        if (rate == 0) {
            return 0;
        }

        value_ = (amount * EXCHANGE_RATE_EXPANSION_FACTOR) / rate;
    }
```

It has a special case `if (rate == 0)`, this can happen quite rarely, but it's still possible.

Example:
1. Users commit 100e18 tokens, so `totalPrincipalTokensCommitted = 100e18`. We assume pool share tokens are minted 1:1 to simplify the example.
2. A loan is accepted by the contract for exactly 100e18 tokens.
3. Time passes and the loan defaults.
4. Liquidators wait to liquidate the entire loan for free, when this happens they liquidate it, thus `tokenDifferenceFromLiquidations = 100e18`.
5. Now, a user wants to add some principal, so he calls `addPrincipalToCommitmentGroup` with 100e18.
6. First we `getPoolTotalEstimatedValue`
```jsx
int256 poolTotalEstimatedValueSigned = int256(
            totalPrincipalTokensCommitted
        ) +
            int256(totalInterestCollected) +
            int256(tokenDifferenceFromLiquidations) -
            int256(totalPrincipalTokensWithdrawn) = 100e18 + 0 + (-100e18) - 0 = 0
```
7. Next we get the `sharesExchangeRate`
```jsx
rate_ =
            (poolTotalEstimatedValue * EXCHANGE_RATE_EXPANSION_FACTOR) /
            poolSharesToken.totalSupply() = 0 * 1e36 / 100e18 = 0
```
8. `_valueOfUnderlying` is then called and because `rate = 0` we return 0.
9. The user receives no share tokens, but his 100e18 are deposited, effectively he lost all of his tokens.

Note that `burnSharesToWithdrawEarnings` will always revert, as we use `sharesExchangeRateInverse`.

```jsx
function sharesExchangeRateInverse()
        public
        view
        virtual
        returns (uint256 rate_)
    {
        return
            (EXCHANGE_RATE_EXPANSION_FACTOR * EXCHANGE_RATE_EXPANSION_FACTOR) /
            sharesExchangeRate();
    }
```
Since `sharesExchangeRate = 0`, the tx will revert as we are attempting to divide by 0.

## Impact
Complete loss of funds for the user that adds principal and the burn is completely unusable.

## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L324-L334

## Tool used
Manual Review

## Recommendation
We recommend to force the initializer of the contract, to transfer some principal tokens when initializing the contract and add those to `totalPrincipalTokensCommitted`. This way `totalPrincipalTokensCommitted` should always be larger than `tokenDifferenceFromLiquidations`. 