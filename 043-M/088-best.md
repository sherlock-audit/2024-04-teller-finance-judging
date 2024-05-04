Straight Crepe Blackbird

high

# Flash loans will not work due to incorrect address verifications

## Summary
Flash loans in `FlashRolloverLoan_G5.sol` will not work due to incorrect address verifications in `onlyFlashLoanPool` modifier and `_initiator` check.

## Vulnerability Detail
`FlashRolloverLoan_G5.sol` contract works with AAVE pools to execute flash loans. 
[https://docs.aave.com/developers/guides/flash-loans#id-2.-calling-flashloan-or-flashloansimple](https://docs.aave.com/developers/guides/flash-loans#id-2.-calling-flashloan-or-flashloansimple)
According to the AAVE documentation, this contract implements a `rolloverLoanWithFlash()` function to call the `IPool(POOL()).flashLoanSimple()` pool's function and pass it the necessary parameters. An `executeOperation()` callback function is also implemented. 
The problem is that this function assumes that it will be called by the AAVE pool. If we look at the `AavePoolMock.sol` test contract, we can see that the `executeOperation()` function is called internally in `flashLoanSimple()`. 
But there is no callback function in the original AAVE pool, instead it is called in the `executeFlashLoanSimple()` function from the `FlashLoanLogic()` contract:
```solidity
// Pool.sol

 /// @inheritdoc IPool
  function flashLoanSimple(
    address receiverAddress,
    address asset,
    uint256 amount,
    bytes calldata params,
    uint16 referralCode
  ) public virtual override {
    DataTypes.FlashloanSimpleParams memory flashParams = DataTypes.FlashloanSimpleParams({
      receiverAddress: receiverAddress,
      asset: asset,
      amount: amount,
      params: params,
      referralCode: referralCode,
      flashLoanPremiumToProtocol: _flashLoanPremiumToProtocol,
      flashLoanPremiumTotal: _flashLoanPremiumTotal
    });
    FlashLoanLogic.executeFlashLoanSimple(_reserves[asset], flashParams);  <<<-------------------
  }
```
[https://github.com/aave/aave-v3-core/blob/master/contracts/protocol/pool/Pool.sol](https://github.com/aave/aave-v3-core/blob/master/contracts/protocol/pool/Pool.sol)
```solidity
// FlashLoanLogic .sol

function executeFlashLoanSimple(
    DataTypes.ReserveData storage reserve,
    DataTypes.FlashloanSimpleParams memory params
  ) external {
     // . . . ;

    require(
      receiver.executeOperation(   <<<------------------
        params.asset,
        params.amount,
        totalPremium,
        msg.sender,      // - AAVE pool
        params.params
      ),
      Errors.INVALID_FLASHLOAN_EXECUTOR_RETURN
    );

    // . . . 
  }
```
[https://github.com/aave/aave-v3-core/blob/master/contracts/protocol/libraries/logic/FlashLoanLogic.sol](https://github.com/aave/aave-v3-core/blob/master/contracts/protocol/libraries/logic/FlashLoanLogic.sol)

The difference in implementations creates problems, because for `FlashLoanLogic.sol` contract `msg.sender` will be the AAVE pool, but not `address(this)`. As a result, this check in `executeOperation()` will revert:
```solidity
require(
            _initiator == address(this),      // _initiator == AAVE pool
            "This contract must be the initiator"
        );
```
This check also will revert, because for `FlashRolloverLoan_G5.executeOperation()` function `msg.sender` will be `FlashLoanLogic.sol` contract, not AAVE pool:
```solidity
modifier onlyFlashLoanPool() {
        require(
            msg.sender == address(POOL()),
            "FlashRolloverLoan: Must be called by FlashLoanPool"
        );

        _;
    }
```
## Impact
Borrower cannot rollover their existing loan using a flash loan mechanism. Considering the potential loss of funds for the borrower and disruption of the entire contract, high severity is relevant.
## Code Snippet
[https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/FlashRolloverLoan_G5.sol#L72-L79](https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/FlashRolloverLoan_G5.sol#L72-L79)
[https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/FlashRolloverLoan_G5.sol#L163-L166](https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/FlashRolloverLoan_G5.sol#L163-L166)
## Tool used

Manual Review

## Recommendation
Сhange the address checks, considering that the `_initiator ` of flash loans will be the AAVE pool, and the callback function will be called by `FlashLoanLogic.sol`.