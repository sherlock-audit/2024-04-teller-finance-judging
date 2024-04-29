Atomic Eggshell Puppy

medium

# In case the user's interest is more than their principal, they can wait to `liquidateDefaultedLoanWithIncentive` for profit

## Summary
If the user has taken a loan from `LenderCommitmentGroup_Smart.sol` and the interest is more than their principal, they can wait to `liquidateDefaultedLoanWithIncentive` for profit 

## Vulnerability Detail
When calling `liquidateDefaultedLoanWithIncentive` within `LenderCommitmentGroup_Smart`, the accrued fees are ignored and user has to only pay based on their principal amount. 

```solidity
    function liquidateDefaultedLoanWithIncentive(
        uint256 _bidId,
        int256 _tokenAmountDifference
    ) public bidIsActiveForGroup(_bidId) {
        uint256 amountDue = getAmountOwedForBid(_bidId, false);

        uint256 loanDefaultedTimeStamp = ITellerV2(TELLER_V2)
            .getLoanDefaultTimestamp(_bidId);

        int256 minAmountDifference = getMinimumAmountDifferenceToCloseDefaultedLoan(
                amountDue,
                loanDefaultedTimeStamp
            );

        require(
            _tokenAmountDifference >= minAmountDifference,
            "Insufficient tokenAmountDifference"
        );
``` 
The problem occurs in the case where the user's interest is more than their principal, meaning that as soon as `liquidateDefaultedLoanWithIncentive` becomes callable, they can call it and pay 2x their principal (which would still be less than what they would pay) and claim back their collateral.

## Impact
Users can liquidate themselves for profit. 

## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L422C1-L439C11

## Tool used

Manual Review

## Recommendation
add proper incentivization for users to repay their loans instead of liquidating themselves.