Great Gunmetal Bat

medium

# LenderCommitmentGroup_Smart :: burnSharesToWithdrawEarnings() If a malicious lender burns their shares sequentially in small amounts can extract extra rewards at the expense of other lenders.

## Summary
**`burnSharesToWithdrawEarnings()`** allows lenders to burn their shares to retrieve both the principal tokens and the earnings they've accrued. However, a potential issue arises if a malicious lender repeatedly burns their shares in small amounts, as they could unfairly claim a disproportionate amount of rewards at the expense of other lenders.
## Vulnerability Detail
**`burnSharesToWithdrawEarnings()`** enables lenders to retrieve both their principal tokens and the earnings they've accrued.
```Solidity
 function burnSharesToWithdrawEarnings(
        uint256 _amountPoolSharesTokens,
        address _recipient
    ) external returns (uint256) {
       

        
        poolSharesToken.burn(msg.sender, _amountPoolSharesTokens);

        uint256 principalTokenValueToWithdraw = _valueOfUnderlying(
            _amountPoolSharesTokens,
            sharesExchangeRateInverse()
        );

        totalPrincipalTokensWithdrawn += principalTokenValueToWithdraw;

        principalToken.transfer(_recipient, principalTokenValueToWithdraw);

        return principalTokenValueToWithdraw;
    }
```
As depicted, the process initially involves burning the lender's shares, followed by the valuation of the burned shares. Then, the exchange rate of the shares is calculated, along with the determination of the underlying asset's value based on the shares' quantity. Finally, the tokens along with the earnings are disbursed to the lender.

However, an issue arises if the lender burns their shares in multiple small amounts transactions. In such cases, they may receive more earnings than if the burn were executed in a single transaction. To comprehend the matter fully, we must delve into the formulas employed.

$poolTotalEstimatedValue = totalPrincipalTokensCommitted + totalInterestCollected + tokenDifferenceFromLiquidations - totalPrincipalTokensWithdrawn$

$Rate = poolTotalEstimatedValue / sharesTotalSupply$

$ValueOfUnderlying = ShareAmount / Rate$

These three formulas serve to determine the number of tokens the lender will receive. To illustrate their function, let's delve into an example for better comprehension.

**`totalPrincipalTokensCommitted = 100,000`**
**`totalInterestCollected = 1,000`**
**`tokenDifferenceFromLiquidations  = 0`**
**`totalPrincipalTokensWithdrawn = 0`**

-------------------------------------------------- Example1: Withdraw all the shares at once -------------------------------------------

**`AmountToWithdraw = 100,000`**

**`poolTotalEstimatedValue = 100,000 + 1000 + 0 - 0  = 101000`**
**`Rate = 101,000 / 100,000 = 1.01 = 1 / 1.01 = 0.99`**
(We require the inverse because the rate calculates the number of shares you'll obtain for a specific amount of principal tokens, but now we need the opposite.)
**`ValueOfUnderlying = 100,000 / 0.99 = 101000`**

The lender will receive 101,000 tokens: 100,000 for their initial deposit and 1,000 for the interest. This is correct.

-------------------------------------------------- Example2: Withdraw all the sahres in two times -------------------------------------

**`AmountToWithdraw = 50,000`**

**`poolTotalEstimatedValue1 = 100,000 + 1000 + 0 - 0 = 101,000`**
**`poolTotalEstimatedValue2 = 100,000 + 1000 + 0 - 50,000 = 51,000`**

**`Rate1= 101000 / 100000 = 1.01 = 1 / 1.01 = 0.99 `**
**`Rate2 = 51000 / 50000 = 1.01 = 1 / 1.01 = 0.98`**

**`ValueOfUnderlying1 = 50,000/ 0.99 = 50505.5`**
**`ValueOfUnderlying2 = 100,000 / 0.99 = 51020.4`**

**`TotalValueOfUnderlying = 50,505.5 + 51,020.4 = 101525.9`**

The lender will receive 101,525.9, 525.9 more than expected.

In this scenario, it's not feasible because the maximum funds are capped at 101,000. However, in a real situation where there are more funds, a malicious lender could obtain funds from other lenders. I've demonstrated this using just one user for simplicity.
## POC
To demonstrate the impact of this issue, I'll provide an example with 20 withdrawals to illustrate the problem's magnitude.

  |     |  |
  | :--------: | :-------: |
  | totalPrincipalTokensCommitted | 100,000    |   
  | Interest| 1,000     |
  | Withdrawals    | 20    |

|   Withdrawal | totalPrincipalTokensCommitted |   totalSharesSupply |Rates|  ValueOfUnderlying |
  | :--------: | :-------: |  :-------: |   :-------: |   :-------: |                
  |1 |101,000 |100,000 |0.9900 | 5,050|
  | 2| 96,000 |95,000| 0.9896|5,052|
  |3 | 91,000|90,000|0.9890 |5,055|
| 4| 86,000|85,000|0.9883 |5,058|
  |5 | 81,000   |80,000|0.9876 |5,062|
  | 6|76,000 |75,000|0.9868 |5,066|
 | 7|71,000  |70,000| 0.9859|5,071|
  |8 |66,000 |65,000|0.9848 |5,076|
  | 9|61,000 |60,000| 0.9836|5,083|
|10 |56,000 |55,000| .98210|5,090|
  | 11|  51,000    |50,000| 0.9803|5,100|
  |12 |46,000 |45,000 |0.9782|5,111|
|13 |  41,000    |40,000|0.9756 |5,125|
  |14 |36,000|35,000|0.9722 |5,125|
 | 15| 31,000|30,000 | 0.9677|5,142|
  |16 | 26,000|25,000|0.9615 |5,166|
  | 17| 21,000|20,000|0.9524 |5,200|
|18 |16,000 |15,000|0.9375 |5,250|
  | 19|   11,000   |10,000|0.9090 |5,500|
  |20 | 6,000|5,000| 0.8333|6,000|

The sum of all the **`ValueOfUnderlying`** amounts to 103,597. The profit made by the malicious lender is calculated as 103,597 - 101,000 = 2,597. If the token is USDC, the total profit from this exploit would be 2,597 USDC. Given this profit, the transaction fees incurred in executing this exploit are justified.
## Impact
Malicious lenders can exploit the system by sequentially withdrawing small amounts of shares, thereby extracting more earnings at the expense of other lenders.
## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L277-L286
https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L288-L302
https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L324-L334
https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L396-L415
## Tool used
Manual Review.
## Recommendation
There isn't a trivial solution for this problem, but one approach could be to require lenders to burn all their shares if they wish to recover their principal tokens along with the earnings.