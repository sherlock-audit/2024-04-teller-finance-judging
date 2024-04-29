Kind Red Gorilla

medium

# `LenderCommitmentGroup_Smart.sol` cannot deploy pools with non-string symbol() ERC20s.


## Summary

In `LenderCommitmentGroup_Smart.sol`, when initializing, it needs to fetch the `symbol()` for the principalToken and collateralToken. However, it assumes the `symbol()` returns a string, which is actually OPTIONAL for ERC20 standards. This means it would fail to create pools for these ERC20s.

## Vulnerability Detail

First, let's quote the [EIP20](https://eips.ethereum.org/EIPS/eip-20) to show `symbol()` is optional:

> symbol
>
> Returns the symbol of the token. E.g. “HIX”.
>
> OPTIONAL - This method can be used to improve usability, but interfaces and other contracts MUST NOT expect these values to be present.
>
> function symbol() public view returns (string)

The most famous token that uses bytes32 instead of string as `symbol()` return value is [MKR](https://etherscan.io/address/0x9f8f72aa9304c8b593d555f12ef6589cc3a579a2#code).

The contest README states that any tokens compatible with Uniswap V3 should be supported, which includes MKR: https://info.uniswap.org/#/tokens/0x9f8f72aa9304c8b593d555f12ef6589cc3a579a2

> We are allowing any standard token that would be compatible with Uniswap V3 to work with our codebase, just as was the case for the original audit of TellerV2.sol. The tokens are assumed to be able to work with Uniswap V3.

At last, let's see the code. `_generateTokenNameAndSymbol` assumes that both principalToken and collateralToken has a `symbol()` function that returns `string`, which is inaccurate. This would revert if we try to create a pool with MKR.

```solidity
    function _generateTokenNameAndSymbol(address principalToken, address collateralToken) 
    internal view 
    returns (string memory name, string memory symbol) {
        // Read the symbol of the principal token
>       string memory principalSymbol = ERC20(principalToken).symbol();
        
        // Read the symbol of the collateral token
>       string memory collateralSymbol = ERC20(collateralToken).symbol();
        
        // Combine the symbols to create the name
        name = string(abi.encodePacked("GroupShares-", principalSymbol, "-", collateralSymbol));
        
        // Combine the symbols to create the symbol
        symbol = string(abi.encodePacked("SHR-", principalSymbol, "-", collateralSymbol));
    }
```

## Impact

`LenderCommitmentGroup_Smart` pools cannot be created for ERC20s that does not implement `function symbol() public view returns (string)`.

## Code Snippet

- https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L241-L255

## Tool used

Manual review

## Recommendation

Consider using a try-catch, an example would be:

```solidity
    function computeSymbol(
        address token
    ) external view returns (string memory) {
        try IERC20Metadata(token).symbol() returns (string memory tokenSymbol) {
            return tokenSymbol;
        } catch {
            return "???";
        }
    }
```