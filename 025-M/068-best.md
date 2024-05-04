Raspy Opaque Gibbon

high

# Borrowers can surpass `liquidityThresholdPercent` and borrow to near 100% of the principal

## Summary
Borrowers can borrow up to 100% of the principal due to [getPrincipalAmountAvailableToBorrow](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L779) relying on deposited principal. This will in tern lock LPs and prevent them from withdrawing.

## Vulnerability Detail
[acceptFundsForAcceptBid](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L336) includes a check that prevents borrowers from borrowing more than the `liquidityThresholdPercent` of the principal. This measure is intended to ensure safe withdrawals for LPs even when demand for borrowing exceeds the available principal. If this threshold is reached, no more assets can be borrowed, though LPs are still free to deposit and withdraw.

However, this check relies on [getPoolTotalEstimatedValue](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L288), which includes `totalPrincipalTokensCommitted` - the total tokens deposited.

```solidity
    function getPrincipalAmountAvailableToBorrow() public view returns (uint256) {     
            return (uint256(getPoolTotalEstimatedValue())).percent(liquidityThresholdPercent) -
            getTotalPrincipalTokensOutstandingInActiveLoans();
    }
```
With this in mind, if `liquidityThresholdPercent` is reached, borrowers can simply deposit some collateral to increase the [getPrincipalAmountAvailableToBorrow](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L779) ratio, borrow the newly available collateral, and withdraw their original stake.

Example:
- `liquidityThresholdPercent` is 80%, expected to be sufficient to ensure normal LP withdrawals.
- Pool value ([getPoolTotalEstimatedValue](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L288)) is $100k.
- Assets borrowed are $80k.
- [getPrincipalAmountAvailableToBorrow](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L779) returns `(100k * 80%) - 80k = 0` -> no borrowing allowed.

Steps:
1. Alice deposits $25k principal (flash loans can be used, but are not required).
2. [getPrincipalAmountAvailableToBorrow](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L779) calculates $20k available (`125k * 80% - $80k = 20k`).
3. Alice borrows $20k.
4. Alice withdraws her $25k principal.

In the example above, our borrower surpassed the `liquidityThresholdPercent`, causing the market to be at 100% utilization, essentially locking LPs until a borrow is repaid or another LP deposits.

## Impact
Pools will be in a locked state, where every new deposit will be instantly borrowed or used as exit liquidity by another LP. This also break core contract functionality - borrowers should not be able to borrow past `liquidityThresholdPercent`.

## Code Snippet
```solidity
        require(
            getPrincipalAmountAvailableToBorrow() >= _principalAmount,
            "Invalid loan max principal"
        );
```

## Tool used
Manual Review

## Recommendation
This issue relates to the overall structure of the protocol and how it implements its math, and not some specific one-liner issue. I am unable to give an exact solution. I can only mention that deposit/withdraw windows will not work here, as the borrower can just withdraw after the time delay.