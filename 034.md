Gigantic Bubblegum Wolf

high

# The calculation in _isLoanDefaulted is wrong and lender can take the collateral of user

## Summary
The calculation in _isLoanDefaulted is wrong and lender can take the collateral of user
## Vulnerability Detail

When Lender uses the lenderCloseLoan it checks if the user has paid the the payment cycle if not, the loan is set to default and the lender can take the collateral. The issue comes calculateNextDueDate because even if the user has paid e.g. this month (the paymentCycle) the loan is still going to be defaulted.

e.g. if the  bidPaymentCycleType is seconds it will summ the acceptedTimestap with paymentCycle => dueDate = 100 + 2000 = 2100
lets say that the borrower payed the minimum loan at 2000 bts
=> delta = 2000 - 100 = 1900
repaymentCycle = 1900 / 200 = 0

dueDate += ( 0 * 2000) += 0

this leads to dueDate  = 2100
when the function return the value the lender can see that after 101s the loan is going to be defaulted and he could get the collateral of the user

function test_lender_canCloseLoanEvenAfterBorrowerPayed() public {
        uint256 bidId = 1;
        setMockBid(bidId);

        tellerV2.setCollateralManagerSuper(address(collateralManagerMock));

        tellerV2.mock_setBidState(bidId, BidState.ACCEPTED);

        vm.warp(2000);

        tellerV2.setMockMsgSenderForMarket(address(this));

        lendingToken.approve(address(tellerV2), 1e20);

        tellerV2.repayLoanMinimum(bidId);

        vm.warp(2101);

        tellerV2.setMockMsgSenderForMarket(address(lender));

        lendingToken.approve(address(tellerV2), 1e20);

        tellerV2.lenderCloseLoan(bidId);

        BidState state = tellerV2.getBidState(bidId);
        // make sure the state is now CLOSED

        require(state == BidState.CLOSED, "bid was not closed");
    }

## Impact
Lenders can take the collateral of the users
## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L1100

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/libraries/V2Calculations.sol#L197-L206
## Tool used

Manual Review

## Recommendation
make a variable to count the times when the users has paid the payment cycle and multiply by the time of paymentCycle so the calculation is like this: uint32(block.timestamp) > `theVariableThatWillCound` * paymentCycle + dueDate + defaultDuration + _additionalDelay

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L1102-L1104