Little Sapphire Lobster

high

# Missing `__Ownable_init()` call in `LenderCommitmentGroup_Smart::initialize()`

## Summary

`__Ownable_init()` is not called in `LenderCommitmentGroup_Smart::initialize()`, which will make the contract not have any owner.

## Vulnerability Detail

`LenderCommitmentGroup_Smart::initialize()` does not call `__Ownable_init()` and will be left without owner.  

## Impact

Inability to [pause](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L793) and [unpause](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L800) borrowing in `LenderCommitmentGroup_Smart` due to having no owner, as these functions are `onlyOwner`.

## Code Snippet

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L158

## Tool used

Manual Review

Vscode

## Recommendation

Modify `LenderCommitmentGroup_Smart::initialize()` to call  `__Ownable_init()`:
```solidity
function initialize(
    ...
) external initializer returns (address poolSharesToken_) {
    __Ownable_init();
}
```