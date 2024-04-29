Atomic Eggshell Puppy

medium

# Borrowers can create an unliquidateable loan

## Summary
Borrowers can create an unliquidateable loan

## Vulnerability Detail
In the case where a user wants to create an indefinite loan - borrowing an amount of money at a said APY, without having a foreseeable end date, simply an APY they have to repay, it can be assumed that the user will simply input a high enough loan duration. 

The problem is that a borrower can utilize this by simply putting a loan duration of `uint32.max - block.timestamp + 1`. By doing this, any time someone tries to liquidate them, `calculateNextDueDate` will revert due to overflow.

```solidity
        uint32 endOfLoan = _acceptedTimestamp + _loanDuration;
``` 

Such calculations are not made upon regular repayments, which makes this a profitable scenario for the borrower.


## Impact
Borrowers can create unliquidateable loans.

## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L379

## Tool used

Manual Review

## Recommendation
set a max loan length (e.g. 5 years) 