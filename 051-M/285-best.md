Fluffy Pewter Pike

medium

# The cycle payment due may span over approx. 2 cycles and block the borrower from paying

## Summary

If a borrower makes the expected cycle payment early in the first payment cycle and then tries to make the cycle payment in the second cycle, the payment reverts, due to insufficient amount.

## Vulnerability Detail

Consider the following bid parameters:

 - Principal amount: `30000`
 - acceptedTimestamp: `100`
 - lastRepaidTimestamp: `100 + 1 * 24 * 3600` (1 day after the bid was accepted)
 - loanDuration: `3 * 30 * 24 * 3600` ( 3 months)
 - totalRepaid: `Payment({ principal: 10000, interest: 200 })`
 - Terms:
    paymentCycleAmount: `10000` (pay `10000` each month)
    paymentCycle: `30 * 24 * 3600` (1 month)
    APR: `1200` (12%)
 - State: `BidState.PENDING`
 - paymentType: `PaymentType.EMI`

In our example, the borrower has already paid the minimum amount for the first month, 1 day after the bid was accepted, by calling `repayLoan(bidId, 10000)`. Now, the borrower calls `TellerV2::repayLoanMinimum` to make their regular payment on the day 58, that is, `block.timestamp == 100 + 58 * 24 * 3600`. Since the borrower has paid for the first month already, they would expect the minimum amount to be `10000` plus interest, that is, the payment for the second cycle. However, the call to `TellerV2::repayLoanMinimum` reverts, if the borrower approves only `10000` plus interest (below `300`).

The call reverts, as the minimum expected payment is now `19000`, which is close to the total owed amount (`20000` + interest). This can be clearly seen, when the borrower calls `repayLoan(bidId, 10000 + 300)`:

```
  [Revert] PaymentNotMinimum(1, 10300 [1.03e4], 19000 [1.9e4])
```

The root cause of this issue is the computation in `V2Calculations::calculateAmountOwed`:

```js
>> uint256 owedTime = _timestamp - uint256(_lastRepaidTimestamp);
   ...
   if (_bid.paymentType == PaymentType.Bullet) {
     ...
   } else {
        // Default to PaymentType.EMI
        // Max payable amount in a cycle
        // NOTE: the last cycle could have less than the calculated payment amount
        uint256 owedAmount = isLastPaymentCycle
            ? owedPrincipal_ + interest_
>>          : (_bid.terms.paymentCycleAmount * owedTime) /
                _paymentCycleDuration;

        duePrincipal_ = Math.min(owedAmount - interest_, owedPrincipal_);
   }
```

Basically, when the difference between two payments in `owedTime` is closer to two cycles, the borrower has to make two payment, notwithstanding the actual payment schedule.

A detailed unit test using `TellerV2_Override`:

<details>
  <summary>See the complete unit test</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import { StdStorage, stdStorage } from "forge-std/StdStorage.sol";
import { Testable } from "../Testable.sol";
import { TellerV2_Override } from "./TellerV2_Override.sol";

import { Bid, BidState, Collateral, Payment, LoanDetails, Terms, ActionNotAllowed } from "../../contracts/TellerV2.sol";
import { PaymentType, PaymentCycleType } from "../../contracts/libraries/V2Calculations.sol";

import { ReputationManagerMock } from "../../contracts/mock/ReputationManagerMock.sol";
import { CollateralManagerMock } from "../../contracts/mock/CollateralManagerMock.sol";
import { LenderManagerMock } from "../../contracts/mock/LenderManagerMock.sol";
import { MarketRegistryMock } from "../../contracts/mock/MarketRegistryMock.sol";

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

import "../tokens/TestERC20Token.sol";

import "lib/forge-std/src/console.sol";

contract audit_TellerV2_bids_test is Testable {
    using stdStorage for StdStorage;

    TellerV2_Override tellerV2;

    TestERC20Token lendingToken;

    TestERC20Token lendingTokenZeroDecimals;

    User borrower;
    User lender;
    User receiver;

    User marketOwner;

    User feeRecipient;

    MarketRegistryMock marketRegistryMock;

    ReputationManagerMock reputationManagerMock;
    CollateralManagerMock collateralManagerMock;
    LenderManagerMock lenderManagerMock;

    uint256 marketplaceId = 100;

    //have to copy and paste events in here to expectEmit
    event SubmittedBid(
        uint256 indexed bidId,
        address indexed borrower,
        address receiver,
        bytes32 indexed metadataURI
    );

    function setUp() public {
        tellerV2 = new TellerV2_Override();

        marketRegistryMock = new MarketRegistryMock();
        reputationManagerMock = new ReputationManagerMock();
        collateralManagerMock = new CollateralManagerMock();
        lenderManagerMock = new LenderManagerMock();

        borrower = new User();
        lender = new User();
        receiver = new User();

        marketOwner = new User();
        feeRecipient = new User();

        lendingToken = new TestERC20Token("Wrapped Ether", "WETH", 1e30, 18);
        lendingTokenZeroDecimals = new TestERC20Token(
            "Wrapped Ether",
            "WETH",
            1e16,
            0
        );
    }

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
                    principal: 30000,
                    timestamp: 100,
                    acceptedTimestamp: 100,
                    lastRepaidTimestamp: 100 + 1 * 24 * 3600, // 1 day after
                    loanDuration: 3 * 30 * 24 * 3600, // 3 months
                    totalRepaid: Payment({ principal: 10000, interest: 200 })
                }),
                terms: Terms({
                    paymentCycleAmount: 10000,      // pay 10000
                    paymentCycle: 30 * 24 * 3600,   // each month
                    APR: 1200                       // 12%
                }),
                state: BidState.PENDING,
                paymentType: PaymentType.EMI
            })
        );
    }

    function test_repay_loan_close_to_two_months() public {
        uint256 bidId = 1;
        setMockBid(bidId);

        tellerV2.mock_setBidState(bidId, BidState.ACCEPTED);
        // warp by 58 days since the bid was accepted
        vm.warp(100 + 58 * 24 * 3600);

        assertEq(block.timestamp, 100 + 58 * 24 * 3600);

        //set the account that will be paying the loan off
        tellerV2.setMockMsgSenderForMarket(address(this));

        // approve the next payment + interest

        lendingToken.approve(address(tellerV2), 10000 + 300);

        // this call reverts
        tellerV2.repayLoan(bidId, 10000 + 300);
        assertTrue(tellerV2.repayLoanWasCalled(), "repay loan was not called");
    }

}

contract User {}
```  
</details>

## Impact

The borrower is unable to make the payment for the current payment cycle, even though they have sufficient funds to do so. Unless the user makes the payment of approximately two payment cycles (instead of one), their loan would be liquidated, and they would lose the collateral.

## Code Snippet

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L639-L644

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L600-L620

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/libraries/V2Calculations.sol#L70-L131

## Tool used

Manual Review

## Recommendation

Use the payment schedule as computed by `V2Calculations::calculateNextDueDate` in `V2Calculations::calculateAmountOwed`. I would expect that if the borrower owes a payment for the current cycle, this should be their minimum due amount (plus interest).
