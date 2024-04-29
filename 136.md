Raspy Opaque Gibbon

medium

# Block stuffing can be used to profit from auctions

## Summary
Block stuffing on L2 is quite cheap and can be used to prevent anyone from closing the auction. This will generate bad debt for the LPs and possibly profit for the operator doing it if they then liquidate the position.

## Vulnerability Detail
The current auction mechanism gives 100% collateral and only changes the amount of principal it asks. There are two parts: one from `T=0 to T=24h` where the auction goes from asking 800% to 100%, and the second part from `T=24h to T=27h`, where the auction goes from asking 100% principal to 0 principal.

Big auctions on L2 chains have the possibility of being block stuffed to reach as low of a point as possible. We only need the stuffing from about `T=24h to T=27h`, or as much as we can get.

We can use this app to calculate our needed gas - [cryptoneur](https://www.cryptoneur.xyz/en/gas-fees-calculator?gas-input=15000000&gas-price-option=on&usedGas=32000000&txnType=Custom&gasPrice=standard), where ARB has a capacity of 32m and it will cost ~$10.1 to fill a block. This means it will take us roughly `5000 * $10.1 = $50,500` to get 100% of the collateral for free. Even if we are not able to obtain 100% of the loan, it will be profitable to keep the auction alive for as long as possible, because on every dollar we spend we get a few back from the decrease in price.

While this amount may seem massive, note that Teller already has loans that are worth way more, for example, [1643 - $115k](https://app.teller.org/ethereum/loan/1643), [1571 - $208k](https://app.teller.org/ethereum/loan/1571), and so on.

## Impact
Loss of funds + breaking of core contract mechanic.

## Code Snippet
principal [math](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L509)
```solidity
int256 incentiveMultiplier = int256(86400) -
    int256(secondsSinceDefaulted);

if (incentiveMultiplier < -10000) {
    incentiveMultiplier = -10000;
}
```

## Tool used
Manual Review

## Recommendation
Extend the second part of the auction. This will significantly increase the complexity and cost of such operations.