Raspy Opaque Gibbon

medium

# Some NFTs will not work with TellerV2

## Summary
TellerV2 will not function correctly when lender NFTs are utilized as collateral for another borrow.

## Vulnerability Detail
Lenders can mint NFTs on their lent loans, representing either the repayment + interest (if the borrower repays) or the collateral (if the borrower gets liquidated). These NFTs inherently hold value, making them ideal collateral for another borrow. Although this mechanic is not explicitly described in the docs, it's common in NFT lending platforms, resembling AAVE's leverage staking.

However, when the original borrower repays their due amount, the tokens are sent to the bid's `CollateralEscrowV1` contract. These funds are not recorded in any balances and thus are lost. The same applies to airdrops, which also end up in `CollateralEscrowV1`.

Example:
1. Alice lends 10 WETH and receives her NFT.
2. Alice needs money, so she lends her 10 WETH NFT for some USDC on another market.
3. When the original borrower makes a repayment, the funds go to the bid's `CollateralEscrowV1`.

## Impact
Funds from repayments will be lost.

## Code Snippet
[_sendOrEscrowFunds](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L909-L916)
```solidity
bid.loanDetails.lendingToken.transferFrom{ gas: 100000 }(
    _msgSenderForMarket(bid.marketplaceId),
    lender, //@audit can be CollateralEscrowV1 if I leverage my NFT
    _paymentAmount
)
```

## Tool used
Manual Review

## Recommendation
Implement a function within `CollateralEscrowV1` that allows the owner of the bid or the new lender to withdraw any assets not directly used as collateral.