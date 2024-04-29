Raspy Opaque Gibbon

medium

# Big LP providers can just exit to avoid negative yield

## Summary
Liquidations take approximately 1 day and 3 hours to reduce principal needed from 100% to 0%. This duration provides ample time for large LPs to exit the pool.

## Vulnerability Detail
[liquidateDefaultedLoanWithIncentive](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L422) employs an auction mechanism where at T = 0, the liquidator needs to provide 8x the principal amount. At T = 1 day, the liquidator can provide just the principal for the collateral, and by T = 1 day + ~3 hours, 100% of the collateral is given for 0% of the principal.

These prolonged auctions give LP providers sufficient time to notice and exit the pool just before the auctions commence, and then re-enter after they conclude. Exiting the pool saves them from the bad debt (when `tokenDifferenceFromLiquidations` is decreased). It also  distributes the debt to other less fortunate LPs.

Example:
1. Alice owns 10% of the pool shares.
2. Bob's loan defaults.
3. Alice realizes that it would be profitable to liquidate Bob’s loan on T = 1 day.
4. A few minutes before T = 1 day, Alice exits the pool.
5. Liquidators proceed with the liquidation of Bob’s loan.
6. Alice re-enters the pool with the same amount, now owning a larger fraction of it (as the share price has decreased).

## Impact
LPs can save some of their assets by avoiding bad debt. This bad debt is then distributed among other LPs who did not leave the pool.

## Code Snippet
```solidity
        } else {
            uint256 tokensToGiveToSender = abs(_tokenAmountDifference);

            IERC20(principalToken).transferFrom(
                msg.sender,
                address(this),
                amountDue - tokensToGiveToSender
            );

            tokenDifferenceFromLiquidations -= int256(tokensToGiveToSender);
            totalPrincipalTokensRepaid += amountDue;
        }
```
## Tool used
Manual Review

## Recommendation
Implement a two-step withdrawal procedure, where LPs need to schedule a withdrawal and then wait the duration it takes to liquidate a loan. This will prevent them from avoiding bad debt.