Generous Carmine Cyborg

medium

# Substracting repaymentAmount instead of _flashAmount when computing the funds remaining in executeOperation will make the call fail for payments where the flash borrowed amount was higher than the loan repayment amount

## Summary

Substracting repaymentAmount instead of _flashAmount when computing the fundsRemaining in the executeOperation function from FlashRolloverLoan_G5 will make the function always fail because more assets than the intended will be transferred to the borrower.

## Vulnerability Detail

The **`FlashRolloverLoan_G5`** contract allows users to rollover their existing loans using a flash loan from AAVE. In order to do so, the `rolloverLoanWithFlash` function is called. This will trigger a flasloan that will execute the `executeOperation` callback and transfer the flash loan funds to **`FlashRolloverLoan_G5`:**

```solidity
// FlashRolloverLoan_G5.sol
function executeOperation(
        address _flashToken,
        uint256 _flashAmount,
        uint256 _flashFees,
        address _initiator,
        bytes calldata _data
    ) external virtual onlyFlashLoanPool returns (bool) {
        ...

        uint256 repaymentAmount = _repayLoanFull(
            _rolloverArgs.loanId,
            _flashToken,
            _flashAmount
        );

        AcceptCommitmentArgs memory acceptCommitmentArgs = abi.decode(
            _rolloverArgs.acceptCommitmentArgs,
            (AcceptCommitmentArgs)
        );

        // Accept commitment and receive funds to this contract

        (uint256 newLoanId, uint256 acceptCommitmentAmount) = _acceptCommitment( 
            _rolloverArgs.lenderCommitmentForwarder,
            _rolloverArgs.borrower,
            _flashToken,
            acceptCommitmentArgs
        );

        //approve the repayment for the flash loan
        IERC20Upgradeable(_flashToken).approve(
            address(POOL()),
            _flashAmount + _flashFees
        );

        uint256 fundsRemaining = acceptCommitmentAmount +
            _rolloverArgs.borrowerAmount -
            repaymentAmount - 
            _flashFees;

        if (fundsRemaining > 0) {
            IERC20Upgradeable(_flashToken).transfer(
                _rolloverArgs.borrower,
                fundsRemaining
            );
        }

        emit RolloverLoanComplete(
            _rolloverArgs.borrower,
            _rolloverArgs.loanId,
            newLoanId,
            fundsRemaining
        );

        return true;
    }

```

The `executeOperation` will perform the following steps:

1. The `_repayLoanFull` function will be called. This will repay the specified loan in full using the `_flashAmount` assets, which are the assets requested to AAVE’s flash loan. It is important to note that the requested `_flashAmount` can differ from the actual amount repaid in the loan, which is returned and stored in the `repaymentAmount` variable, meaning that the `repaymentAmount` could be smaller than `_flashAmount`.
2. After this, the `_acceptCommitment` function will be called. This function will create a new loan, and `acceptCommitmentAmount` of collateral will be transferred to the**`FlashRolloverLoan_G5`** contract.
3. Finally, the flash loan will be repaid. Prior to repaying it, a computation will be performed to check the `fundsRemaining` in the contract. `fundsRemaining` is the amount that can be transferred to the borrower after repaying the flash loan.

The problem is that the `fundsRemaining` are computed by substracting the `repaymentAmount`, instead of the `_flashAmount`.

As mentioned in step 1, the `_flashAmount` requested by the user when performing the flash loan can be greater than the actual amount repaid for the loan. However, the `fundsRemaining` computation does not consider this situation, and wrongly substracts `repaymentAmount` instead of `_flashAmount`, assuming that the difference between `_flashAmount` and `repaymentAmount` are assets that belong to the borrower, when in reality belong to the repayment of the flash loan (note how the prior approval to the pool approves `_flashAmount + _flashFees` instead of `repaymentAmount + _flashFees`).

This will make the `executeOperation` to always fail when the difference between `_flashAmount` and `repaymentAmount` is greater than zero, because AAVE’s pool will try to pull `_flashAmount + _flashFees` from `FlashRolloverLoan_G5`, but only `_flashAmount - repaymentAmount + _flashFees` will remain in the contract.

## Impact

Medium. This is a likely scenario where the `executeOperation` will always fail if the amount requested by the user in the flash loan is greater than the amount needed to fully repay the loan.

## Code Snippet

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/FlashRolloverLoan_G5.sol#L201

## Tool used

Manual Review

## Recommendation

Substract `_flashAmount` instead of `repaymentAmount` when computing the `fundsRemaining`:

```diff
// FlashRolloverLoan_G5
function executeOperation(
        address _flashToken,
        uint256 _flashAmount,
        uint256 _flashFees,
        address _initiator,
        bytes calldata _data
    ) external virtual onlyFlashLoanPool returns (bool) {
        ...
        
          uint256 fundsRemaining = acceptCommitmentAmount +
            _rolloverArgs.borrowerAmount -
-            repaymentAmount - 
+            _flashAmount - 
            _flashFees;

```
