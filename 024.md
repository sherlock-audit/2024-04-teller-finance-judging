Atomic Eggshell Puppy

high

# Borrower can send the `repayment` with less gas and it will `repayLoanCallback` will not be executed.

## Summary
Borrower can send the `repayment` with less gas and it will `repayLoanCallback` will not be executed

## Vulnerability Detail
When repaying a loan, if there's loan repayment listener set, it attempts to call the `repayLoanCallback`  in a try-catch, passing 80,000 gas for the call 
```solidity
        if (loanRepaymentListener != address(0)) {
            try
                ILoanRepaymentListener(loanRepaymentListener).repayLoanCallback{
                    gas: 80000
                }( //limit gas costs to prevent lender griefing repayments
                    _bidId,
                    _msgSenderForMarket(bid.marketplaceId),
                    _payment.principal,
                    _payment.interest
                )
            {} catch {}
```

If at the time of entering the try-catch statement, the gas left is less than 80,000, it will simply assign 63/64th of it for the `repayLoanCallback`. If that gas is not enough, it will simply enter the catch statement. 

This allows for users both purposefully and accidentally sending the tx with just enough gas so that it is not enough to execute the `repayLoanCallback` and it enters the catch statement.

Since `LenderCommitmentGroup_Smart.sol` depends on the execution of the callback for the sake of proper accounting, the described attack path will break all accounting and result as stuck funds (as the contract will assume the funds are still not repaid)

## Impact
Broken accounting, permanently stuck funds

## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L951C1-L961C24

## Tool used

Manual Review

## Recommendation
before calling `repayLoanCallback`, make sure there's at least 80,000 `gasleft()` 