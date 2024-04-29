Original Flint Koala

high

# Shares Minted Can Round In Favor Of User

## Summary

In share minting, the denominator _rounds down_, which can lead to the result _rounding up_.

## Vulnerability Detail

Here's a general principal. Let's in this equation, all the divisions are integer divisions:

```solidity
result = a / (b / c)
```

Does `result` always return a lower value than the float division of the same equation? No.

For example let `a = 10`, `b = 3`, `and c = 2.`

**In float division:**
b / c in float would be `3 / 2 = 1.5`.
`a / (b / c)` in float would be 10 / 1.5 ≈ **6.666....**

In integer division as Solidity does:
`b / c` would be `3 / 2 = 1` (since the fractional part is discarded).
`result = a / (b / c) `would be 10 / 1 = **10.**

-----------------------------------------------

This exact scenario happens during share calculation. The share calculation formula is:

`_sharesAmount = _valueOfUnderlying(_amount, sharesExchangeRate());

`sharesExchangeRate()` returns:
`value_ = (amount * EXCHANGE_RATE_EXPANSION_FACTOR) / rate;`

We can see that `sharesExchangeRate()` rounds DOWN
Since it is the divisor, it means that the `_sharesAmount` can round UP


## Impact

By simply deposting and withdrawing specific amounts, users can get more tokens that they put in. Asset-to-share ratio can inflate either through a first deposit attack, or simply through the accumulation of liquidations and multiple withdrawals and deposits which can cause this gain/loss to exceed the gas cost of executing the transactions.

## Code Snippet

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L315

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L333

## Tool used

Manual Review

## Recommendation

`sharesExchangeRate()` could have an extra `bool roundUp` condition, and set it to true in the `addPrincipalToCommitmentGroup()` 
