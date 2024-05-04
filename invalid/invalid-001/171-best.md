Virtual Peanut Seagull

medium

# TellerV2.sol#_liquidateLoanFull() - Doesn't set the reputation mark correctly

## Summary
TellerV2.sol#_liquidateLoanFull() - Doesn't set the reputation mark correctly

## Vulnerability Detail
The protocol implements borrower reputation, so that lenders can check if a borrower has ever been late/defaulted on one or more of his loans.

The only place where reputation is updated in inside `_repayLoan`. The function is called when a loan is being repaid (partially or fully).

The problem is when a loan gets liquidated, the `bid.state` is first set to `LIQUIDATED` and then inside `_repayLoan` we call `updateAccountReputation`.

`updateAccountReputation` uses `_applyReputation` to set add marks to a borrower who has been defaulted or was late for a payment, you can see that if `isLoanDefaulted = true` then a mark will be added. 

```solidity
function _applyReputation(address _account, uint256 _bidId)
        internal
        returns (RepMark mark_)
    {
        mark_ = RepMark.Good;

        if (tellerV2.isLoanDefaulted(_bidId)) {
            mark_ = RepMark.Default;

            // Remove delinquent status
            _removeMark(_account, _bidId, RepMark.Delinquent);
        } else if (tellerV2.isPaymentLate(_bidId)) {
            mark_ = RepMark.Delinquent;
        }

        // Mark status if not "Good"
        if (mark_ != RepMark.Good) {
            _addMark(_account, _bidId, mark_);
        }
    }
```

In the case of liquidations, a loan becomes liquidatable after it has become defaulted, so in theory, when a liquidation of a loan occurs, the borrower should incur a `RepMark.Default`, but because `isLoanDefaulted` is written in the following way, he won't.

```jsx
function _isLoanDefaulted(uint256 _bidId, uint32 _additionalDelay)
        internal
        view
        returns (bool)
    {
        Bid storage bid = bids[_bidId];

        // Make sure loan cannot be liquidated if it is not active
        if (bid.state != BidState.ACCEPTED) return false;

        uint32 defaultDuration = bidDefaultDuration[_bidId];

        uint32 dueDate = calculateNextDueDate(_bidId);

        return
            uint32(block.timestamp) >
            dueDate + defaultDuration + _additionalDelay;
    }
```

As you can see if `bid.state != ACCEPTED` then then `isLoanDefaulted == false`, so no mark will be added, but that shouldn't be the case.

## Impact
Because no mark is added to the borrower, his reputation won't be affected, so future potential lenders won't know that the borrower has defaulted on his loans before.

## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L812

## Tool used
Manual Review

## Recommendation
Add the following to `_liquidateLoanFull`
```solidity
function _liquidateLoanFull(uint256 _bidId, address _recipient)
        internal
        acceptedLoan(_bidId, "liquidateLoan")
    {
        require(isLoanLiquidateable(_bidId), "Loan must be liquidateable.");

        Bid storage bid = bids[_bidId];
// Loan is being liquidated, update borrower's mark
RepMark mark = reputationManager.updateAccountReputation(
            bid.borrower,
            _bidId
        );
        // change state here to prevent re-entrancy
        bid.state = BidState.LIQUIDATED;

        (uint256 owedPrincipal, , uint256 interest) = V2Calculations
            .calculateAmountOwed(
                bid,
                block.timestamp,
                _getBidPaymentCycleType(_bidId),
                _getBidPaymentCycleDuration(_bidId)
            );

        //this sets the state to 'repaid'
        _repayLoan(
            _bidId,
            Payment({ principal: owedPrincipal, interest: interest }),
            owedPrincipal + interest,
            false
        );

        /*
         _getCollateralManagerForBid(_bidId).liquidateCollateral(
            _bidId,
            _recipient
        ); 
      */

        collateralManager.liquidateCollateral(_bidId, _recipient);

        address liquidator = _msgSenderForMarket(bid.marketplaceId);

        emit LoanLiquidated(_bidId, liquidator);
    }
```