Raspy Opaque Gibbon

high

# Borrowers can brick the commitment group pool

## Summary
Borrowers can [setRepaymentListenerForBid](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L1248) after directly borrowing from `TellerV2`. Upon repayment, `totalPrincipalTokensRepaid` will increase without a corresponding increase in `totalPrincipalTokensLended`. This discrepancy causes `getTotalPrincipalTokensOutstandingInActiveLoans` to underflow, which will brick commitment group contract.

## Vulnerability Detail
When borrowing from LenderCommitmentGroup (LCG), the `acceptFundsForAcceptBid` function increases `totalPrincipalTokensLended` and calls [_acceptBidWithRepaymentListener](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L384) to log the bid as borrowed from LCG. Later, the [_repayLoan](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L851) function, having made `repaymentListenerForBid[_bidId] = _listener`, triggers [_sendOrEscrowFunds](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L901) which will call the [repayLoanCallback](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L700) inside LCG. This increases both `totalPrincipalTokensRepaid` and `totalInterestCollected`. 

The flow balances `totalPrincipalTokensLended` on borrowing with `totalPrincipalTokensRepaid` on repayment, equalizing them.

However, borrowers can directly borrow from TellerV2 and call [setRepaymentListenerForBid](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L1248). Upon repayment, `totalPrincipalTokensRepaid` and `totalInterestCollected` will increase without a corresponding increase in `totalPrincipalTokensLended` (because the borrowing was direct from TellerV2).

- The inflation of `totalInterestCollected` causes share values to increase, generating unbacked profits.
- The inflation of `totalPrincipalTokensRepaid` leads to `getTotalPrincipalTokensOutstandingInActiveLoans` underflowing, potentially bricking the contract.

Note that user can even make custom tokens with infinite liquidity and use them to borrow / repay to maximize the damage.

## Impact
Loss of funds.

## Code Snippet
```solidity
function setRepaymentListenerForBid(uint256 _bidId, address _listener) external {
    address sender = _msgSenderForMarket(bids[_bidId].marketplaceId);
    require(sender == bids[_bidId].lender, "Only the bid lender may set the repayment listener");
    repaymentListenerForBid[_bidId] = _listener;
}
```
## Tool used
Manual Review

## Recommendation
Make sure [setRepaymentListenerForBid]() is callable only by the LCG.
```diff
-  function setRepaymentListenerForBid(uint256 _bidId, address _listener) external
+  function setRepaymentListenerForBid(uint256 _bidId, address _listener) onlyCommitmentGroup external
    {
        address sender = _msgSenderForMarket(bids[_bidId].marketplaceId);
        require( sender == bids[_bidId].lender,"Only bid lender may set repayment listener");
        repaymentListenerForBid[_bidId] = _listener;
    }
```