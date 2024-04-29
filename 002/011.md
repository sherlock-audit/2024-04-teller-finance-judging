Cuddly Strawberry Gibbon

high

# Lender may not be able to close loan or get back lending token.

## Summary
The `TellerV2` smart contract offers lenders to claim a NFT by calling `claimLoanNFT`. This will set the `bid.lender` to the constant variable `USING_LENDER_MANAGER`. This is done to ensure that the lender will only be able to claim 1 NFT per bid. Because of that `getLoanLender` function is added to check who the real owner of the NFT is(the real lender). However that whole logic introduces problems that can cause lender to not be able to close a loan or get his lending tokens.

## Vulnerability Detail
Lets split the issues and look at them separately:

#### Not being able to close a loan
A lender can close a loan as long as the bid is accepted. This can be done by calling `lenderCloseLoan` or `lenderCloseLoan`.

```solidity
    function lenderCloseLoan(uint256 _bidId)
        external
        acceptedLoan(_bidId, "lenderClaimCollateral")
    {
        Bid storage bid = bids[_bidId];
        address _collateralRecipient = bid.lender;

        _lenderCloseLoanWithRecipient(_bidId, _collateralRecipient);
    }
```

```solidity
    function lenderCloseLoanWithRecipient(
        uint256 _bidId,
        address _collateralRecipient
    ) external {
        _lenderCloseLoanWithRecipient(_bidId, _collateralRecipient);
    }
```

As can be seen, both functions call internal function `_lenderCloseLoanWithRecipient`.

```solidity
    function _lenderCloseLoanWithRecipient(
        uint256 _bidId,
        address _collateralRecipient
    ) internal acceptedLoan(_bidId, "lenderClaimCollateral") {
        require(isLoanDefaulted(_bidId), "Loan must be defaulted.");

        Bid storage bid = bids[_bidId];
        bid.state = BidState.CLOSED;

        address sender = _msgSenderForMarket(bid.marketplaceId);
        require(sender == bid.lender, "only lender can close loan");

        /*


          address collateralManagerForBid = address(_getCollateralManagerForBid(_bidId)); 

          if( collateralManagerForBid == address(collateralManagerV2) ){
             ICollateralManagerV2(collateralManagerForBid).lenderClaimCollateral(_bidId,_collateralRecipient);
          }else{
             require( _collateralRecipient == address(bid.lender));
             ICollateralManager(collateralManagerForBid).lenderClaimCollateral(_bidId );
          }
          
          */

        collateralManager.lenderClaimCollateral(_bidId);

        emit LoanClosed(_bidId);
    }
```

Notice this validation `require(sender == bid.lender, "only lender can close loan");`. If a lender claims loan NFT, the bid.lender will be `USING_LENDER_MANAGER`. This means that this check will fail and lender will not be able to close the loan. Also the same validation can be seen in `setRepaymentListenerForBid`.

#### Not being able to get lending tokens
Lender got his loan NFT and now his bid is either `PAID` or `LIQUIDATED`. This will call the internal function `_sendOrEscrowFunds`:

```solidity
    function _sendOrEscrowFunds(uint256 _bidId, Payment memory _payment)
        internal
    {
        Bid storage bid = bids[_bidId];
        address lender = getLoanLender(_bidId);

        uint256 _paymentAmount = _payment.principal + _payment.interest;

        // some code
    }
```

This time the lender is determined by calling `getLoanLender` function:

```solidity
    function getLoanLender(uint256 _bidId)
        public
        view
        returns (address lender_)
    {
        lender_ = bids[_bidId].lender;

        if (lender_ == address(USING_LENDER_MANAGER)) {
            return lenderManager.ownerOf(_bidId);
        }

        //this is left in for backwards compatibility only
        if (lender_ == address(lenderManager)) {
            return lenderManager.ownerOf(_bidId);
        }
    }
```

If the bid.lender is `USING_LENDER_MANAGER`, then the owner of the NFT is the lender. However this may not be true as the lender can lose(transfer to another account or get stolen), or sell(without knowing it's purpose) . This means that the new owner of the NFT will get the lending tokens as well.

## Impact
Lenders not being able to close a loan or get back lending tokens.

## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L724-L732

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L738-L743

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L745-L755

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L901-L905

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L1173-L1188

## Tool used
Manual Review

## Recommendation
For the first problem use `getLoanLender()` instead of `bid.lender`. Also in `getLoanLender` do not determine the lender by the owner of the NFT, but instead add a mapping that will keep track of who the real lender is.