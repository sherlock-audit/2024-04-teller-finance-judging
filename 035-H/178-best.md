Virtual Peanut Seagull

high

# If `repayLoanCallback` address doesn't implement `repayLoanCallback`  try/catch won't go into the catch and will revert the tx

## Summary
If `repayLoanCallback` address doesn't implement `repayLoanCallback`  try/catch won't go into the catch and will revert the tx

## Vulnerability Detail
If a contract, which is set as `loanRepaymentListener` from a lender doesn't implement `repayLoanCallback` transaction will revert and `catch` block won't help.
This is serious and even crucial problem, because a malicous lender could prevent borrowers from repaying their loans, as `repayLoanCallback` is called inside [the only function used to repay loans](https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L887). This way he guarantees himself their collateral tokens.

[Converastion explaining why `try/catch` helps only if transaction is reverted in the target, contrac, which is not the case here](https://ethereum.stackexchange.com/questions/129150/solidity-try-catch-call-to-external-non-existent-address-method)

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
        }
```
## Impact
Lenders can stop borrowers from repaying their loans, forcing their loans to default.

## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L953

## Tool used
Manual Review

## Recommendation
Maybe use a wrapper contract, which is trusted to you and is internally calling the `repayLoanCallback` on the untrusted target.