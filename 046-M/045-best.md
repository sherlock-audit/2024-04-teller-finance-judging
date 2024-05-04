Raspy Opaque Gibbon

high

# FlashRolloverLoan_G5 can repay more than it's needed

## Summary
[executeOperation](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/FlashRolloverLoan_G5.sol#L199-L202) refunds the borrower all the unspent funds. However, if the flash loan (FL) was larger than the needed funds, the excess amount is transferred to the borrower. This can lead to either the transaction reverting if we are unable to repay the FL, or the borrower receiving extra funds and the FL being repaid from the contract's balances.

## Vulnerability Detail
Users can choose their desired `_flashLoanAmount` when using [rolloverLoanWithFlash](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/FlashRolloverLoan_G5.sol#L97). This, in turn, causes the loan provider to send us the funds and execute [executeOperation](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/FlashRolloverLoan_G5.sol#L156). After we have repaid the FL, the amount plus fees are approved for the FL provider to take, and the remaining funds are sent to the borrower:

```solidity
uint256 fundsRemaining = acceptCommitmentAmount +
    _rolloverArgs.borrowerAmount -
    repaymentAmount -
    _flashFees;
```

However, the current calculation does not account for the possibility that **we can borrow far more than needed to repay**. [_repayLoan](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L866) will pass without issue, as it has a check to see if we are sending more funds than necessary:

```solidity
if (paymentAmount >= _owedAmount) {
    paymentAmount = _owedAmount;
```

The extra funds will remain inside the FlashRolloverLoan_G5 contract, where will be accounted for when calculating `fundsRemaining` and transferred to the user. This will either:
- drain the contract - if there are funds inside they will go towards paying the flash loan.
- revert the TX - if the contract has 0 balance, then we need `FL amount == repay amount` or else the TX will always revert.

Achieving an `FL amount == repaymentAmount` is challenging as the interest changes every block. Moreover, we cannot do `FL amount < repaymentAmount` because it will revert, even if we supply enough `borrowerAmount`, which is another issue.

## Impact
Users can extract funds from the `FlashRolloverLoan_G5` or cause any transaction with an FL amount greater than the `repaymentAmount` to revert. 

## Code Snippet
```solidity
uint256 fundsRemaining = acceptCommitmentAmount +
    _rolloverArgs.borrowerAmount -
    repaymentAmount -
    _flashFees;
```

## Tool used
Manual Review

## Recommendation
Ensure the calculations take into account how large the FL is. A simpler option would be to limit the flash loan amount to be equal to the repayment amount.