Jolly Bamboo Goose

high

# liquidateDefaultedLoanWithIncentive can be gamed to avoid paying loans interest

## Summary
The amountDue calculation at the liquidateDefaultLoanWithIncentive function at the LenderCommitmentGroup contract calls getAmountOwedForBid with a false argument, which means it considers the principal lent as owed but not the interest that was accrued during the borrow period.
This can be gamed by malicious users to avoid paying loans interest.
## Vulnerability Detail
Take a look at the getAmountOwedForBid function:
```solidity
function getAmountOwedForBid(uint256 _bidId, bool _includeInterest)
        public
        view
        virtual
        returns (uint256 amountOwed_)
    {
        Payment memory amountOwedPayment = ITellerV2(TELLER_V2)
            .calculateAmountOwed(_bidId, block.timestamp);

        amountOwed_ = _includeInterest
            ? amountOwedPayment.principal + amountOwedPayment.interest
            : amountOwedPayment.principal;
    }
```

Notice it will only return the principal amount if a false boolean is passed as the second argument of a call to it.

At the liquidateDefaultedLoanWithIncentive function, this is exactly what happens:
```solidity
function liquidateDefaultedLoanWithIncentive(
        uint256 _bidId,
        int256 _tokenAmountDifference
    ) public bidIsActiveForGroup(_bidId) {
        uint256 amountDue = getAmountOwedForBid(_bidId, false);
        ...
}
```

This means the amountDue does not include interest.

However, this is not on par with TellerV2 contract, as its liquidation function does repay both the owed principal and the interest. This can be seen at the following code snippet:
```solidity
function _liquidateLoanFull(uint256 _bidId, address _recipient)
        internal
        acceptedLoan(_bidId, "liquidateLoan")
    {
...    
_repayLoan(
            _bidId,
            Payment({ principal: owedPrincipal, interest: interest }),
            owedPrincipal + interest,
            false
        );
...
}
```
## Impact
Users can arbitrarily decide to liquidate repaying interest or not back to Teller lenders by liquidating via the TellerV2 contract or via LenderCommitmentGroups contracts.
LenderCommitmentGroups's liquidation function spreads bad debt to the whole lending pool while benefitting the liquidator.
## Code Snippet

[2024-04-teller-finance/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol at main · sherlock-audit/2024-04-teller-finance (github.com)](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L426)

[2024-04-teller-finance/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol at main · sherlock-audit/2024-04-teller-finance (github.com)](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L474)

[2024-04-teller-finance/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol at main · sherlock-audit/2024-04-teller-finance (github.com)](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L803)
## Tool used

Manual Review

## Recommendation
Ensure the calculation of amountDue accounts for interest when repaying a liquidated bid. The following change could be done at the code:
```solidity
uint256 amountDue = getAmountOwedForBid(_bidId, true);
```
