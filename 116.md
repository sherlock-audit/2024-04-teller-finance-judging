Jolly Bamboo Goose

high

# Malicious borrower can pay each payment and make its own loan default 1 month later

## Summary
There's an edge case in which NextDueDate calculation will yield a due date much longer than what it ought to.
## Vulnerability Detail
If the user repays a portion of its loan exactly one day and one second after the accepted timestamp, the next due date will not be 1 month later, but two.
Take a look at the calculateNextDueDate function at the V2Calculations library, notice the following snippet:
```solidity
if (
                BPBDTL.getDay(_lastRepaidTimestamp) >
                BPBDTL.getDay(_acceptedTimestamp)
            ) {
                lastPaymentCycle += 2;
            } else {
                lastPaymentCycle += 1;
            }
```

Since one lastPaymentCycle unit will be summed to the due date as the equivalent of one month, lastRepaidTimestamp just has to be on the further day after the accepted timestamp to yield not 1 month but a 2 month due date. This can be used by malicious parties to avoid defaults and to repay loans with much smaller rates.

There's also a second case in which it fails:
```solidity
dueDate_ = _acceptedTimestamp + _paymentCycle;
            // Calculate the cycle number the last repayment was made
            uint32 delta = _lastRepaidTimestamp - _acceptedTimestamp;
            if (delta > 0) {
                uint32 repaymentCycle = uint32(
                    Math.ceilDiv(delta, _paymentCycle)
                );
                dueDate_ += (repaymentCycle * _paymentCycle);
            }
```
The due date can be of two cycles if the borrower pays back a little later than 1 payment cycle duration. 

The following POC utilizes the second case to exemplify

**POC**
Paste the following code snippet at the TellerV2_bids.sol contract:
```solidity
function test_repay_loan_minimum2_a() public {
        uint256 bidId = 1;
        setMockBid(bidId);

        tellerV2.mock_setBidState(bidId, BidState.ACCEPTED);
        vm.warp(2 days + block.timestamp);

        //set the account that will be paying the loan off
        tellerV2.setMockMsgSenderForMarket(address(this));

        tellerV2.calculateNextDueDate(bidId);
        //need to get some weth

        lendingToken.approve(address(tellerV2), 1e20);

        tellerV2.calculateAmountOwed(bidId, block.timestamp);
        vm.warp( 28 days + block.timestamp );
        tellerV2.repayLoan(bidId, 100);

        tellerV2.calculateAmountOwed(bidId, block.timestamp);
        assertEq(tellerV2.calculateNextDueDate(bidId), 5184100);
    }

    function test_repay_loan_minimum2_b() public {
        uint256 bidId = 1;
        setMockBid(bidId);

        tellerV2.mock_setBidState(bidId, BidState.ACCEPTED);
        vm.warp(2 days + block.timestamp);

        //set the account that will be paying the loan off
        tellerV2.setMockMsgSenderForMarket(address(this));

        tellerV2.calculateNextDueDate(bidId);
        //need to get some weth

        lendingToken.approve(address(tellerV2), 1e20);

        tellerV2.calculateAmountOwed(bidId, block.timestamp);
        vm.warp( 28 days + block.timestamp + 1 hours);
        tellerV2.repayLoan(bidId, 100);

        tellerV2.calculateAmountOwed(bidId, block.timestamp);
        assertEq(tellerV2.calculateNextDueDate(bidId), 7776100);
    }
```


Before running the tests, make sure to alter the following snippets:
TellerV2_bids.sol setMockBid function:
```solidity
function setMockBid(uint256 bidId) public {
        tellerV2.mock_setBid(
            bidId,
            Bid({
                borrower: address(borrower),
                lender: address(lender),
                receiver: address(receiver),
                marketplaceId: marketplaceId,
                _metadataURI: "0x1234",
                loanDetails: LoanDetails({
                    lendingToken: lendingToken,
                    principal: 100000,
                    timestamp: 100,
                    acceptedTimestamp: 100,
                    lastRepaidTimestamp: 0,
                    loanDuration: 365 days,
                    totalRepaid: Payment({ principal: 0, interest: 0 })
                }),
                terms: Terms({
                    paymentCycleAmount: 10,
                    paymentCycle: 30 days,
                    APR: 10
                }),
                state: BidState.PENDING,
                paymentType: PaymentType.EMI
            })
        );
    }
```

TellerV2_Override.sol repayLoan function:
```solidity
function _repayLoan(
        uint256 _bidId,
        Payment memory _payment,
        uint256 _owedAmount,
        bool _shouldWithdrawCollateral
    ) internal override {
        Bid storage bid = bids[_bidId];

        bid.loanDetails.totalRepaid.principal += _payment.principal;
        bid.loanDetails.totalRepaid.interest += _payment.interest;
        bid.loanDetails.lastRepaidTimestamp = uint32(block.timestamp);
        
    }
```

Run the tests with the following command:
```shell
forge test --match-test test_repay_loan_minimum2 -vvvvv 
```

Take a look at the execution traces, the test b ends up with a calculateNextDueDate resulting in 7776100 and test a resulting in 5184100 while having the same amount owed. This effectively means a borrower can partially delay his/her payment to get much later dates for the next payment.
## Impact
Borrowers can avoid defaults and repayments by arbitrarily paying on certain timestamps. In the worst case a borrower can make multiple monthly payments every two months, essentially halving the borrow APY.
This issue is IN-SCOPE as these calculations are utilized by TellerV2.sol.
As it is a very easy to setup attack vector, the likelihood is high. As it doesn't incur loss of funds, but decreases the earnings for lenders, the impact is medium.
## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L1036
[2024-04-teller-finance/teller-protocol-v2-audit-2024/packages/contracts/contracts/libraries/V2Calculations.sol at main · sherlock-audit/2024-04-teller-finance (github.com)](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/libraries/V2Calculations.sol#L199)
[2024-04-teller-finance/teller-protocol-v2-audit-2024/packages/contracts/contracts/libraries/V2Calculations.sol at main · sherlock-audit/2024-04-teller-finance (github.com)](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/libraries/V2Calculations.sol#L185)

## Tool used

Manual Review

## Recommendation
Instead of the possibility of adding two months delay as the next payment, a better option would be to always enable a 1 month delay:
```solidity
FROM:
if (
                BPBDTL.getDay(_lastRepaidTimestamp) >
                BPBDTL.getDay(_acceptedTimestamp)
            ) {
                lastPaymentCycle += 2;
            } else {
                lastPaymentCycle += 1;
            }
TO:
                lastPaymentCycle += 1;


```

For the second case, the next payment should always be after a single payment cycle, so the following dueDate should be as follows:
```solidity
if (delta > 0) {
                uint32 repaymentCycle = uint32(
                    Math.ceilDiv(delta, _paymentCycle)
                );
                // no repayment cycle multiplication as that will round in favour of the borrower and make the next repayment happen on a much later time period
                dueDate_ += _paymentCycle;
            }
```
