Little Sapphire Lobster

medium

# `FlashRolloverLoan_G5` will fail for `LenderCommitmentGroup_Smart` due to `CollateralManager` pulling collateral from `FlashRolloverLoan_G5`

## Summary

`FlashRolloverLoan_G5` calls `SmartCommitmentForwarder::acceptCommitmentWithRecipient()`, which will have `CollateralManager` commiting tokens from `FlashRolloverLoan_G5`, which will revert as it does not approve it nor have the funds.

## Vulnerability Detail

The issue lies in the fact that `FlashRolloverLoan_G5` assumes `SmartCommitmentForwarder` gets the borrower from the [last 20 bytes](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/FlashRolloverLoan_G5.sol#L303), but it sets the `borrower` to [msg.sender](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/SmartCommitmentForwarder.sol#L106) instead.

Thus, in `SmartCommitmentForwarder::acceptCommitmentWithRecipient()`, `TellerV2::submitBid()` is called with the borrower being `FlashRolloverLoan_G5`, which will end up having the `CollateralManager` [pulling](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L334-L336) collateral from `FlashRolloverLoan_G5`, which will fail, as it does not deal with this.

## Impact

`FlashRolloverLoan_G5` will never work for `LenderCommitmentGroup_Smart` loans.

## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/FlashRolloverLoan_G5.sol#L303
https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/SmartCommitmentForwarder.sol#L106

## Tool used

Manual Review

Vscode

## Recommendation

In `FlashRolloverLoan_G5::_acceptCommitment()` pull the collateral from the borrower and approve the `CollateralManager`.