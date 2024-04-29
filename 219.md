Generous Carmine Cyborg

high

# Not transferring collateral when submitting bids allows malicious users to create honeypot-style attacks

## Summary
Currently, Teller does not require collateral to be transferred when a bid request with collateral is submitted. Instead, the collateral is pulled from the borrower when the bid is accepted by the lender in a different transaction. This pattern allows attackers to leverage certain collaterals to perform honeypot-style attacks.

## Vulnerability Detail

The current flow to create a bid in Teller putting some collateral consists in the following steps:

1. Call `submitBid` with an array of `Collateral`. This creates the bid request, but **does not transfer the collateral to Teller**. Instead, Teller only check that Teller performs is checking the collateral balance of the borrower to guarantee that he actually owns the collateral assets.
2. After submitting the bid request, an interested lender can lend his assets to the borrower by calling `lenderAcceptBid`. This is the step where collateral will actually be transferred from the borrower, as shown in the following code snippet:
    
    ```solidity
    // TellerV2.sol
    
    function lenderAcceptBid(uint256 _bidId)
            external
            override
            pendingBid(_bidId, "lenderAcceptBid")
            whenNotPaused
            returns (
                uint256 amountToProtocol,
                uint256 amountToMarketplace,
                uint256 amountToBorrower
            )
        {
            ...
    
            // Tell the collateral manager to deploy the escrow and pull funds from the borrower if applicable
            collateralManager.deployAndDeposit(_bidId);
            
            ...
            
        }
    ```
    

Although this pattern (checking the borrower’s collateral balance in step 1 to ensure he actually owns the assets) might look like a correct approach, it’s actually incorrect and can lead to honeypot-style attacks when specific NFTs are used as collateral.

Consider the following scenario: a malicious borrower holds a Uniswap V3 liquidity position NFT with liquidity worth 10 ETH. He decides to trick borrowers in Teller, and submits a loan request asking for only 1 ETH (a very attractive offer, given that on default, the lender will gain access to 10 ETH worth of collateral):

1. Borrower creates the borrow request by calling `submitBid`. `submitBid` then checks and guarantees that the borrower indeed holds the Uniswap liquidity position.
2. After some time, a lender sees the borrow request and decides that they want to lend their assets to the borrower by calling `lenderAcceptBid`, as the Uniswap liquidity position (which is worth 10 ETH) is attractive enough to cover the possibility of borrowed assets never being repaid. 
3. The malicious borrower then frontruns the lender transaction and decreases the Uniswap liquidity position to nearly 0 (because collateral has not been transferred when submitting the bid request, the NFT owner is still the borrower, hence why liquidity can be decreased by him). 
4. After decreasing the Uniswap position liquidity, the lender’s `lenderAcceptBid` transaction actually gets executed. Uniswap’s liquidity position NFT gets transferred to Teller from the borrower, and the borrowed funds are transferred to the borrower.

As we can see, not transferring the collateral when the bid request is submitted can lead to these type of situations. Borrowers can easily trick lenders, making them believe that their loaned assets are backed by an NFT worth an X amount, when in reality the NFT will be worth nearly 0 when the transactions actually get executed.

## Impact

High. Attackers can easily steal all the borrowed assets from lenders with no collateral cost at all, given that the collateral NFT will be worth 0 when the borrow is actually executed.

## Code Snippet

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L334

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/TellerV2.sol#L521

## Tool used

Manual Review

## Recommendation

One way to mitigate this type of attack is to force users to transfer their collateral to Teller when a borrow request is submitted. This approach would easily mitigate this issue, as users won’t be able to perform any action over the collateral NFTs as they won’t be the owners. In the situation where the loan is never accepted and the bidExpirationTime is reached, borrowers should be able to withdraw their collateral assets.
