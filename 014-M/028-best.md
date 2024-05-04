Atomic Eggshell Puppy

medium

# Possible donation attack within `LenderCommitmentGroup_Smart`

## Summary
There's currently a possible donation attack within `LenderCommitmentGroup_Smart` 

## Vulnerability Detail
Any attacker can do a donation attack within `LenderCommitmentGroup_Smart` in the following way
1. Attacker deposits 100 USDC within `LenderCommitmentGroup_Smart` 
2. Attacker takes a loan against them with an insanely high interest (e.g. 100 USDC per second)
3. Attacker sends 100 USDC to the `LenderCommitmentGroup_Smart` contract 
4. Attacker withdraws all of their shares, except for 1 wei. (the rate is still 1:1) 
5. Attacker then waits for innocent user to attempt a deposit. Attacker then front-runs it and repays their loan. 
6. The accrued interest inflates the share price (e.g. to 1000 USDC per 1 wei share). 
7. The innocent user's deposit rounds down to 0 shares.



## Impact
Loss of funds

## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L294C1-L296C50

## Tool used

Manual Review

## Recommendation
add shares offset, the same way it is in OZ ERC4626 vaults