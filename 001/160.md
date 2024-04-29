Virtual Peanut Seagull

high

# `LenderCommitmentGroup_Smart` if all shares are burned after accrued interest, new depositor loosees part of his funds

## Summary
There could be a situation of a `LenderCommitmentGroup_Smart` where the vault accrues some interest over time (lets say 10 WETH) and slowly all depositors exit the system by calling `burnSharesToWithdrawEarnings` untill `poolSharesToken.totalsupply = 0`
## Vulnerability Detail
After some time if a new depositor enters the system with 5 WETH, we would be minted 5 poolSharesToken, because of the following line inside [sharesExchangeRate](https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L262-L275)
```solidity
        if (poolSharesToken.totalSupply() == 0) {
            return EXCHANGE_RATE_EXPANSION_FACTOR; // 1 to 1 for first swap
        }
```
Now the shares for the next depositor are deflated, because there are still `10e18` in `totalInterestCollected`, and so `getPoolTotalEstimatedValue` will return 5e18(totalPrincipalTokensCommitted - totalPrincipalTokensWithdrawn) + 10e18 (getPoolTotalEstimatedValue)

Now inside `sharesExchangeRate` we get 10e18 * 1e36 / 5e18 = 2 * 1e36
```solidity
        rate_ = 
            (poolTotalEstimatedValue  *
                EXCHANGE_RATE_EXPANSION_FACTOR) /
            poolSharesToken.totalSupply(); 
```
which means that `_valueOfUnderlying` for the new depositor of 5 WETH would be
`5e18 * 1e36/2*36 = 2.5e18 shareTokens`
```solidity
    function _valueOfUnderlying(uint256 amount, uint256 rate)
        internal
        pure
        returns (uint256 value_)
    {
        if (rate == 0) {
            return 0;
        }
      
        value_ = (amount * EXCHANGE_RATE_EXPANSION_FACTOR) / rate; 
    }
```

Now `shareTokens` is 7.5e18 for `totalPrincipalTokensCommitted - totalPrincipalTokensWithdrawn = 10e18`, which means that corresponding WETH to first depositor for  7.5e18 `shareTokens` is 1.5 WETH, while second depositor has only 0,5

__NOTE__
The same issue is present with `tokenDifferenceFromLiquidations`, which won't be counted for second time first depositor
## Impact

User funds loss
## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L267-L270
## Tool used

Manual Review

## Recommendation

- One solution is to reset `totalInterestCollected` while `shareTokens.totalSupply` is zero
- Another is to check if totalInterestCollected is greater than 0 here:
```solidity
        if (poolSharesToken.totalSupply() == 0 && totalInterestCollected == 0) {
            return EXCHANGE_RATE_EXPANSION_FACTOR; // 1 to 1 for first swap
        }
```