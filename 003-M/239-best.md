Interesting Pine Cheetah

medium

# The `LenderCommitmentGroup_Smart` contract cannot use USDT as its principal token, because `USDT.transfer()` does not return a boolean value.

## Summary

The `transferFrom()`, `transfer()`, and `approve()` functions from the `IERC20` standard return a boolean value, but the ones of `USDT` does not. This makes it impossible for `USDT` to be used as the principal token of the `LenderCommitmentGroup_Smart` contract.

## Vulnerability Detail

The document of this protocol said, "We are allowing any standard token that would be compatible with Uniswap V3 to work with our codebase, just as was the case for the original audit of TellerV2.sol . The tokens are assumed to be able to work with Uniswap V3."

USDT is one of the most widely-used tokens on the Uniswap V3 decentralized exchange. 

And, the `USDT.transfer()` function does not return a boolean value, which is different from other tokens that inherit the IERC20 standard.

https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol

```solidity

    interface IERC20 {
        // ...
        function transfer(address to, uint256 value) external returns (bool);

        function approve(address spender, uint256 value) external returns (bool);

        function transferFrom(address from, address to, uint256 value) external returns (bool);
        // ...
    }
   
```

In the [LenderCommitmentGroup_Smart.sol](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol), however, the use of `transferFrom()`, `transfer()`, and `approve()` functions from the `IERC20` standard makes it impossible for `USDT` to be used as the principal token, because any call to these functions would revert.

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol

```solidity

contract LenderCommitmentGroup_Smart is
    // ...
85      IERC20 public principalToken;
    // ...
313     principalToken.transferFrom(msg.sender, address(this), _amount);
    // ...
373     principalToken.transfer(_recipient, principalTokenValueToWithdraw);
    // ...
412     principalToken.approve(address(TELLER_V2), _principalAmount);
    // ...

```

## Impact

The `LenderCommitmentGroup_Smart` contract cannot use USDT as its principal token.

## Code Snippet

https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L412

## Tool used

Manual Review

## Recommendation

SafeERC20 library should used for IERC20.