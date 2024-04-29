Dapper Plum Ferret

medium

# M-2:  Reentrancy Risk in addPrincipalToCommitmentGroup Function

## Summary
The `addPrincipalToCommitmentGroup` function allows users to add principal tokens to a commitment group and receive corresponding shares. However, there are potential reentrancy vulnerabilities present in the function due to the interaction with external contracts and the order of operations within the function.
## Vulnerability Detail
The vulnerability arises from the use of the `transferFrom` function to transfer tokens from the caller to the contract before updating the state variables and minting shares. This order of operations creates an opportunity for a reentrancy attack where an attacker can recursively call back into the function before the initial token transfer is completed and manipulate the transferred amount to receive more shares than intended.
## Impact
If exploited, the reentrancy vulnerability could allow an attacker to manipulate the amount of tokens transferred and receive a disproportionate number of shares, potentially gaining undue control or benefits within the commitment group.
## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L307
```solidity
 function addPrincipalToCommitmentGroup(
        uint256 _amount,
        address _sharesRecipient
    ) external returns (uint256 sharesAmount_) {
        //transfers the primary principal token from msg.sender into this contract escrow

        principalToken.transferFrom(msg.sender, address(this), _amount);

        sharesAmount_ = _valueOfUnderlying(_amount, sharesExchangeRate());

        totalPrincipalTokensCommitted += _amount;
        //principalTokensCommittedByLender[msg.sender] += _amount;

        //mint shares equal to _amount and give them to the shares recipient !!!
        poolSharesToken.mint(_sharesRecipient, sharesAmount_);
    }
```
## Tool used

Manual Review

## Recommendation
Implement the checks-effects-interactions pattern to ensure that state changes occur before any external calls.
Or : Use the ReentrancyGuard pattern from OpenZeppelin to automatically prevent reentrancy in vulnerable functions.