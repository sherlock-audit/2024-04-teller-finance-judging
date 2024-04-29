Raspy Opaque Gibbon

medium

# `_borrowerAmount` is not used, preventing users to use some of their balances to lower the loan

## Summary
`_borrowerAmount` is not used, and users are forced to pay 100% of the loan using flash loans.

## Vulnerability Detail
[rolloverLoanWithFlash](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/FlashRolloverLoan_G5.sol#L97) allows the use of a flash loan to pay off the debt and create a new loan. It includes `_flashLoanAmount`—how much to flash loan, and `_borrowerAmount`—any additional amount that the borrower may want to add:
```solidity
* @param _borrowerAmount Additional amount that the borrower may want to add during rollover.
```
However, when executing the FL inside [executeOperation](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/FlashRolloverLoan_G5.sol#L156), `_rolloverArgs.borrowerAmount` is not used to offset the repayment amount:
```solidity
        uint256 repaymentAmount = _repayLoanFull(
            _rolloverArgs.loanId,
            _flashToken,
            _flashAmount  // @audit no _rolloverArgs.borrowerAmount
        );
```
This means that if the user inputs 90% of the loan to be paid with FL and 10% with `borrowerAmount`, the transaction will revert. Anything less than a 100% FL amount will revert as the loan will not be paid in full.

`_rolloverArgs.borrowerAmount` is later returned to the user—the only place it's used.

```solidity
        uint256 fundsRemaining = acceptCommitmentAmount +
            _rolloverArgs.borrowerAmount -
            repaymentAmount -
            _flashFees;
```
## Impact
Users cannot add their amounts to offset the loan, even if they have the funds available.

## Code Snippet
```solidity
        uint256 repaymentAmount = _repayLoanFull(
            _rolloverArgs.loanId,
            _flashToken,
            _flashAmount  // @audit no _rolloverArgs.borrowerAmount
        );
```
## Tool used
Manual Review

## Recommendation
Offset the loan by this amount.

```diff
        uint256 repaymentAmount = _repayLoanFull(
            _rolloverArgs.loanId,
            _flashToken,
-           _flashAmount
+           _flashAmount + _rolloverArgs.borrowerAmount
        );
```