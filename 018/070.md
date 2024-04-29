Raspy Opaque Gibbon

medium

# Utilization math should include `liquidityThresholdPercent`

## Summary
Utilization math should include `liquidityThresholdPercent`, as this represents the maximum value that can be borrowed. This means that if borrowing reaches `principal * liquidityThresholdPercent`, then 100% of the available assets are considered borrowed.

## Vulnerability Detail
[getPoolUtilizationRatio](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L757) is used within [getMinInterestRate](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L769) to calculate the interest rate at which borrowers borrow. Higher utilization equates to a higher APR.

However, in the current case, [getMinInterestRate](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L769) fails to include `liquidityThresholdPercent` in its calculation. This variable is crucial as it caps the maximum borrowing allowed (e.g., if it's 80%, then a maximum of 80% of the assets can be borrowed).

Not including this variable means that the utilization (and thus the APR) will appear lower than it actually is. An example of this is a more risky market with a lower `liquidityThresholdPercent`, such as 40%. In these markets, even though there is some risk that the LPs take, the maximum utilization will be 40%, as borrowers cannot borrow above that, even if there is demand for this token. This in tern will decease the profits LPs make from staking in risky markets.

## Impact
This results in a breakdown of core contract functionality. Utilization above `liquidityThresholdPercent` is unreachable, leading to lower LP profits.

## Code Snippet
```solidity
    function getPoolUtilizationRatio() public view returns (uint16) {
        if (getPoolTotalEstimatedValue() == 0) { return 0; }

        return uint16( Math.min(
           getTotalPrincipalTokensOutstandingInActiveLoans() * 10000 / 
           getPoolTotalEstimatedValue(), 10000 ));
    }   
```
## Tool used
Manual Review

## Recommendation
Include `liquidityThresholdPercent` in [getPoolUtilizationRatio](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L757). For example:

```diff
-   return uint16(Math.min(getTotalPrincipalTokensOutstandingInActiveLoans() * 10000 / getPoolTotalEstimatedValue(), 10000));
+   return uint16(Math.min(getTotalPrincipalTokensOutstandingInActiveLoans() * 10000 / (getPoolTotalEstimatedValue().percent(liquidityThresholdPercent)), 10000));
```