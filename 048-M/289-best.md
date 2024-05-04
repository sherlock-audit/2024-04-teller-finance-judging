Kind Red Gorilla

medium

# Users can bypass auction mechanism for `LenderCommitmentGroup_Smart` liquidation mechanism for loans that are close to end of loan


## Summary

`LenderCommitmentGroup_Smart` has an auction mechanism for users to help liquidate default loans. The auction begins from 8x of amountOwed gradually decreasing to 0x. However, for loans that are close to end of loan, users can bypass this mechanism and pay only 1x of amountOwed to perform liquidation.

## Vulnerability Detail

For loans that are close to end of loan, the calculated due date and default timestamp will not change upon a repay. See function `getLoanDefaultTimestamp()` which calls `calculateNextDueDate()`. If the calculated `dueDate` is larger than `endOfLoan`, the due date will return `endOfLoan`.

```solidity
    function getLoanDefaultTimestamp(uint256 _bidId)
        public
        view
        returns (uint256)
    {
        Bid storage bid = bids[_bidId];

        uint32 defaultDuration = _getBidDefaultDuration(_bidId);

        uint32 dueDate = calculateNextDueDate(_bidId);

        return dueDate + defaultDuration;
    }

    function calculateNextDueDate(
        uint32 _acceptedTimestamp,
        uint32 _paymentCycle,
        uint32 _loanDuration,
        uint32 _lastRepaidTimestamp,
        PaymentCycleType _bidPaymentCycleType
    ) public view returns (uint32 dueDate_) {
        ...
        uint32 endOfLoan = _acceptedTimestamp + _loanDuration;
        //if we are in the last payment cycle, the next due date is the end of loan duration
>       if (dueDate_ > endOfLoan) {
>           dueDate_ = endOfLoan;
>       }
    }

```

This means for a loan that is close to the end of loan, and is already default. The normal way to liquidate would be participating the auction using `liquidateDefaultedLoanWithIncentive()`.

However, a user can first repay the collateral by `TellerV2#repayLoanFullWithoutCollateralWithdraw` and make the amountOwed equal to zero, then call `liquidateDefaultedLoanWithIncentive()` and pass `_tokenAmountDifference == 0`. Since the due date does not change upon a repay, the loan is still in default, and the user can successfully perform the liquidation.

If the user participated the auction, he would have to pay 8x the amount of tokens. However, by repaying in TellerV2, he only needs to pay 1x and can perform the liquidation.

```solidity
    function liquidateDefaultedLoanWithIncentive(
        uint256 _bidId,
        int256 _tokenAmountDifference
    ) public bidIsActiveForGroup(_bidId) {
>       uint256 amountDue = getAmountOwedForBid(_bidId, false);

        uint256 loanDefaultedTimeStamp = ITellerV2(TELLER_V2)
            .getLoanDefaultTimestamp(_bidId);

>       int256 minAmountDifference = getMinimumAmountDifferenceToCloseDefaultedLoan(
>               amountDue,
>               loanDefaultedTimeStamp
            );

        require(
            _tokenAmountDifference >= minAmountDifference,
            "Insufficient tokenAmountDifference"
        );

        if (_tokenAmountDifference > 0) {
            //this is used when the collateral value is higher than the principal (rare)
            //the loan will be completely made whole and our contract gets extra funds too
            uint256 tokensToTakeFromSender = abs(_tokenAmountDifference);

            IERC20(principalToken).transferFrom(
                msg.sender,
                address(this),
                amountDue + tokensToTakeFromSender
            );

            tokenDifferenceFromLiquidations += int256(tokensToTakeFromSender);

            totalPrincipalTokensRepaid += amountDue;
        } else {
           
            uint256 tokensToGiveToSender = abs(_tokenAmountDifference);

>           IERC20(principalToken).transferFrom(
                msg.sender,
                address(this),
                amountDue - tokensToGiveToSender
            );

            tokenDifferenceFromLiquidations -= int256(tokensToGiveToSender);

            totalPrincipalTokensRepaid += amountDue;
        }

        //this will give collateral to the caller
        ITellerV2(TELLER_V2).lenderCloseLoanWithRecipient(_bidId, msg.sender);
    }
```

## Impact

For loans that are close to end of loan, users can bypass auction mechanism and pay only 1x of amountOwed to perform liquidation.

## Code Snippet

- https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L422
- https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L1234-L1246
- https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/libraries/V2Calculations.sol#L212-L214

## Tool used

Manual review

## Recommendation

If a loan is already close to end of loan date, only allow the lender to repay.
