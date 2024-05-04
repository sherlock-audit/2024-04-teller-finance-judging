Raspy Opaque Gibbon

high

# Borrowers can claim part of their interest

## Summary
Before repaying, a borrower can stake into the LenderCommitmentGroup (LCG) to receive a portion of the interest paid back to them.

## Vulnerability Detail
When borrowers repay loans taken from LCG, the function [_sendOrEscrowFunds](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L901) triggers [repayLoanCallback](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L700) to LCG, which in turn increases `totalInterestCollected`. This variable is used to calculate pool profits inside [getPoolTotalEstimatedValue](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L288). Higher profits equate to a higher share value.

This can be abused by a borrower by simply staking ([addPrincipalToCommitmentGroup](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L307)), then repaying, and finally unstaking ([burnSharesToWithdrawEarnings](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L396)). When the actual repayment happens, the borrower will own a percentage of the pool, entitling them to a portion of the interest (that they paid to the LPs). Their leverage can be increased with a flash loan (for free if using Morpho).

Example:
1. Alice is about to repay a 100k loan with 10k interest.
2. The pool has a 1m balance.
3. Alice flash loans 1m and calls [addPrincipalToCommitmentGroup](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L307), increasing her shares.
4. Alice repays the loan.
5. She withdraws, claiming 50% of her interest as increased share value (shares to principal ratio).

## Impact
All borrowers can steal some of the paid-back interest, essentially paying less and reducing LP profits.

## Code Snippet
```solidity
function addPrincipalToCommitmentGroup(
    uint256 _amount,
    address _sharesRecipient
) external returns (uint256 sharesAmount_) {
    principalToken.transferFrom(msg.sender, address(this), _amount);
    sharesAmount_ = _valueOfUnderlying(_amount, sharesExchangeRate());

    totalPrincipalTokensCommitted += _amount;
    //principalTokensCommittedByLender[msg.sender] += _amount;

    poolSharesToken.mint(_sharesRecipient, sharesAmount_);
}
```

## Tool used
Manual Review

## Recommendation
Set a withdrawal time window to prevent borrowers from exploiting this feature.