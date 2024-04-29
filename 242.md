Thankful Corduroy Stallion

medium

# Some functions lack `whenNotPaused` modifier.

## Summary
There are some function which do not include the `whenNotPaused` modifier.
## Vulnerability Detail
There are several functions, including `TellerV2::cancelBid` ,  `TellerV2::marketOwnerCancelBid`, `TellerV2::repayLoanMinimum`, `TellerV2::repayLoanFull` that lack the `whenNotPaused` modifier.
## Impact
This can lead to a borrower being able to cancel their bid even when the protocol is paused and all operations must be temporarily paused.
## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L428-L459
```solidity
function cancelBid(uint256 _bidId) external {
        if (
            _msgSenderForMarket(bids[_bidId].marketplaceId) !=
            bids[_bidId].borrower
        ) {
            revert ActionNotAllowed({
                bidId: _bidId,
                action: "cancelBid",
                message: "Only the bid owner can cancel!"
            });
        }
        _cancelBid(_bidId);
    }
```
```solidity
function marketOwnerCancelBid(uint256 _bidId) external {
        if (
            _msgSender() !=
            marketRegistry.getMarketOwner(bids[_bidId].marketplaceId)
        ) {
            revert ActionNotAllowed({
                bidId: _bidId,
                action: "marketOwnerCancelBid",
                message: "Only the market owner can cancel!"
            });
        }
        _cancelBid(_bidId);
        emit MarketOwnerCancelledBid(_bidId);
    }
```
etc.
## Tool used

Manual Review

## Recommendation
Implement the following changes to all the functions that lack the modifier:
```solidity
function cancelBid(uint256 _bidId) external whenNotPaused {
        if (
            _msgSenderForMarket(bids[_bidId].marketplaceId) !=
            bids[_bidId].borrower
        ) {
            revert ActionNotAllowed({
                bidId: _bidId,
                action: "cancelBid",
                message: "Only the bid owner can cancel!"
            });
        }
        _cancelBid(_bidId);
    }
```
```solidity
function marketOwnerCancelBid(uint256 _bidId) external whenNotPaused {
        if (
            _msgSender() !=
            marketRegistry.getMarketOwner(bids[_bidId].marketplaceId)
        ) {
            revert ActionNotAllowed({
                bidId: _bidId,
                action: "marketOwnerCancelBid",
                message: "Only the market owner can cancel!"
            });
        }
        _cancelBid(_bidId);
        emit MarketOwnerCancelledBid(_bidId);
    }
``` 