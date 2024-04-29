Atomic Eggshell Puppy

medium

# `_repayLoan` now allows for overpaying of loan and could cause DoS within `LenderCommitmentGroup_Smart`

## Summary
`_repayLoan` now allows for overpaying of loan


## Vulnerability Detail
Let's look at the code of  `_repayLoan` 
```solidity
    function _repayLoan(
        uint256 _bidId,
        Payment memory _payment,
        uint256 _owedAmount,
        bool _shouldWithdrawCollateral
    ) internal virtual {
        Bid storage bid = bids[_bidId];
        uint256 paymentAmount = _payment.principal + _payment.interest;

        RepMark mark = reputationManager.updateAccountReputation(
            bid.borrower,
            _bidId
        );

        // Check if we are sending a payment or amount remaining
        if (paymentAmount >= _owedAmount) {
            paymentAmount = _owedAmount;

            if (bid.state != BidState.LIQUIDATED) {
                bid.state = BidState.PAID;
            }

            // Remove borrower's active bid
            _borrowerBidsActive[bid.borrower].remove(_bidId);

            // If loan is is being liquidated and backed by collateral, withdraw and send to borrower
            if (_shouldWithdrawCollateral) {
                //   _getCollateralManagerForBid(_bidId).withdraw(_bidId);
                collateralManager.withdraw(_bidId);
            }

            emit LoanRepaid(_bidId);
        } else {
            emit LoanRepayment(_bidId);
        }

        _sendOrEscrowFunds(_bidId, _payment); //send or escrow the funds
```
In the case where the user attempts to overpay, `paymentAmount` is adjusted to `owedAmount`. However, the `_payment` memory struct remains the same. Meaning that if the user has inputted more than supposed to, it will go through, causing an overpay. 

To furthermore make it worse, this would allow for overpaying loans within `LenderCommitmentGroup_Smart`. This is problematic as it would make the following function revert.
```solidity
    function getTotalPrincipalTokensOutstandingInActiveLoans()
        public
        view
        returns (uint256)
    {
        return totalPrincipalTokensLended - totalPrincipalTokensRepaid;
    }
```

Since the function is called when a user attempts to borrow, this would cause permanent DoS for borrows within the `LenderCommitmentGroup_Smart` 

## Impact
Users accidentally overpaying
Permanent DoS within `LenderCommitmentGroup_Smart` 

## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L867

## Tool used

Manual Review

## Recommendation
adjust the memory struct 