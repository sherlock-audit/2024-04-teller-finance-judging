Raspy Opaque Gibbon

medium

# LPs earn less yeild even in full markets

## Summary
The current system utilizes a fixed APR, which does not incentivize borrowers to repay their debt promptly. This behavior is undesirable as it encourages full markets and slower loan repayments, while lower LP profits.

## Vulnerability Detail
The market currently employs a fixed APR, determined at the time the loan is made. With longer loan durations, this fixed APR can lead to issues as the market approaches full utilization. Lower APRs generally incentivize borrowers to repay loans more slowly, as repaying faster and needing another borrow would result in a much higher APR for subsequent loans.

This situation can result in significantly lower APRs, even when the market is fully utilized. There is even the possibility of borrowers simply staking into the market to earn yield, particularly if their APR is smaller than the rate at which the share value grows.

Example:
1. Loan duration is 1 year.
2. APR ranges from 2% to 20% depending on utilization.
3. 100% of the collateral is borrowed in the first year.
4. The average APR that LPs earn will be: `2% + 20% / 2 = 11%`.

Even with the market fully utilized, LPs still only earn 11% APR, despite the maximum APR being 20%. Currently, the first few borrowers with `APR < 11%` can deposit their funds and earn interest.

## Impact
Slower loan repayments and significantly lower LP profits, even though they are in a full market, pose considerable risks, as there could be a point where there are no funds to withdraw.

## Code Snippet
APR is calculated within [getMinInterestRate](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L769-L771):
```solidity
    function getMinInterestRate() public view returns (uint16) {
        return interestRateLowerBound + uint16(uint256(interestRateUpperBound - interestRateLowerBound).percent(getPoolUtilizationRatio()));
    }
```

## Tool used
Manual Review

## Recommendation
Include a function that can update the APR on bids, or modify the approach slightly to incorporate floating APRs and calculate them on repayments or liquidations.