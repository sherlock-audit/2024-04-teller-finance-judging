Fun Blonde Chicken

high

# Logic error in LenderCommitmentForwarder_G2::_acceptCommitment() causing Dos

## Summary
Logic error in `LenderCommitmentForwarder_G2::_acceptCommitment()` causing Dos
## Vulnerability Detail
```javascript
function _acceptCommitment(
        uint256 _commitmentId,
        uint256 _principalAmount,
        uint256 _collateralAmount,
        uint256 _collateralTokenId,
        address _collateralTokenAddress,
        address _recipient,
        uint16 _interestRate,
        uint32 _loanDuration
    ) internal returns (uint256 bidId) {
        Commitment storage commitment = commitments[_commitmentId];

        //make sure the commitment data adheres to required specifications and limits
        validateCommitment(commitment);

        //the collateral token of the commitment should be the same as the acceptor expects
        require(
            _collateralTokenAddress == commitment.collateralTokenAddress,
            "Mismatching collateral token"
        );
        //the interest rate must be at least as high has the commitment demands. The borrower can use a higher interest rate although that would not be beneficial to the borrower.
        require(
            _interestRate >= commitment.minInterestRate,
            "Invalid interest rate"
        );
        //the loan duration must be less than the commitment max loan duration. The lender who made the commitment expects the money to be returned before this window.
        require(
            _loanDuration <= commitment.maxDuration,
            "Invalid loan max duration"
        );

        require(
@>            commitmentPrincipalAccepted[bidId] <= commitment.maxPrincipal,
            "Invalid loan max principal"
        );

        require(
            commitmentBorrowersList[_commitmentId].length() == 0 ||
                commitmentBorrowersList[_commitmentId].contains(_msgSender()),
            "unauthorized commitment borrower"
        );
        //require that the borrower accepting the commitment cannot borrow more than the commitments max principal
        if (_principalAmount > commitment.maxPrincipal) {
            revert InsufficientCommitmentAllocation({
                allocated: commitment.maxPrincipal,
                requested: _principalAmount
            });
        }

        uint256 requiredCollateral = getRequiredCollateral(
            _principalAmount,
            commitment.maxPrincipalPerCollateralAmount,
            commitment.collateralTokenType,
            commitment.collateralTokenAddress,
            commitment.principalTokenAddress
        );

        if (_collateralAmount < requiredCollateral) {
            revert InsufficientBorrowerCollateral({
                required: requiredCollateral,
                actual: _collateralAmount
            });
        }

        //ERC721 assets must have a quantity of 1
        if (
            commitment.collateralTokenType == CommitmentCollateralType.ERC721 ||
            commitment.collateralTokenType ==
            CommitmentCollateralType.ERC721_ANY_ID ||
            commitment.collateralTokenType ==
            CommitmentCollateralType.ERC721_MERKLE_PROOF
        ) {
            require(
                _collateralAmount == 1,
                "invalid commitment collateral amount for ERC721"
            );
        }

        //ERC721 and ERC1155 types strictly enforce a specific token Id.  ERC721_ANY and ERC1155_ANY do not.
        if (
            commitment.collateralTokenType == CommitmentCollateralType.ERC721 ||
            commitment.collateralTokenType == CommitmentCollateralType.ERC1155
        ) {
            require(
                commitment.collateralTokenId == _collateralTokenId,
                "invalid commitment collateral tokenId"
            );
        }

        commitmentPrincipalAccepted[_commitmentId] += _principalAmount;

        require(
            commitmentPrincipalAccepted[_commitmentId] <=
                commitment.maxPrincipal,
            "Exceeds max principal of commitment"
        );

        CreateLoanArgs memory createLoanArgs;
        createLoanArgs.marketId = commitment.marketId;
        createLoanArgs.lendingToken = commitment.principalTokenAddress;
        createLoanArgs.principal = _principalAmount;
        createLoanArgs.duration = _loanDuration;
        createLoanArgs.interestRate = _interestRate;
        createLoanArgs.recipient = _recipient;
        if (commitment.collateralTokenType != CommitmentCollateralType.NONE) {
            createLoanArgs.collateral = new Collateral[](1);
            createLoanArgs.collateral[0] = Collateral({
                _collateralType: _getEscrowCollateralType(
                    commitment.collateralTokenType
                ),
                _tokenId: _collateralTokenId,
                _amount: _collateralAmount,
                _collateralAddress: commitment.collateralTokenAddress
            });
        }

        bidId = _submitBidWithCollateral(createLoanArgs, _msgSender());

        _acceptBid(bidId, commitment.lender);

        emit ExercisedCommitment(
            _commitmentId,
            _msgSender(),
            _principalAmount,
            bidId
        );
    }
```
bidId is a local variable, its initial value is always 0.Comparing commitmentPrincipalAccepted[bidId] namely is Comparing commitmentPrincipalAccepted[0].
require commitmentPrincipalAccepted[0] is alwayse less than commitment.maxPrincipal 
cause the problem of DoS.
#### POC
Add this function `test_acceptCommitment_logic_error_check()` in LenderCommitmentForwarder_Unit_Test.sol.
```javascript
function test_acceptCommitment_logic_error_check() public {
        ILenderCommitmentForwarder.Commitment
            memory c = ILenderCommitmentForwarder.Commitment({
                maxPrincipal: maxPrincipal,
                expiration: expiration,
                maxDuration: maxDuration,
                minInterestRate: minInterestRate,
                collateralTokenAddress: address(collateralToken),
                collateralTokenId: collateralTokenId,
                maxPrincipalPerCollateralAmount: maxPrincipalPerCollateralAmount,
                collateralTokenType: collateralTokenType,
                lender: address(lender),
                marketId: marketId,
                principalTokenAddress: address(principalToken)
            });

         //uint256 commitmentId = 0;

        // lenderCommitmentForwarder.setCommitment(commitmentId, c);
        uint256 commitmentId = lender._createCommitment(c, emptyArray);

        uint256 principalAmount = maxPrincipal;
        uint256 collateralAmount = 1000;
        uint16 interestRate = minInterestRate;
        uint32 loanDuration = maxDuration;

        // vm.expectRevert("collateral token mismatch");
        lenderCommitmentForwarder.acceptCommitment(
            commitmentId,
            principalAmount,
            collateralAmount,
            collateralTokenId,
            address(collateralToken),
            interestRate,
            loanDuration
        );

        assertEq(
            lenderCommitmentForwarder.getCommitmentMaxPrincipal(commitmentId),
            maxPrincipal,
            "Max principal changed"
        );

        ILenderCommitmentForwarder.Commitment
            memory c2 = ILenderCommitmentForwarder.Commitment({
                maxPrincipal: maxPrincipal - 100,
                expiration: expiration,
                maxDuration: maxDuration,
                minInterestRate: minInterestRate,
                collateralTokenAddress: address(collateralToken),
                collateralTokenId: collateralTokenId,
                maxPrincipalPerCollateralAmount: maxPrincipalPerCollateralAmount,
                collateralTokenType: collateralTokenType,
                lender: address(lender),
                marketId: marketId,
                principalTokenAddress: address(principalToken)
            });

            principalAmount = maxPrincipal - 100;
        //commitmentId = 1
         commitmentId = lender._createCommitment(c2, emptyArray);
        lenderCommitmentForwarder.acceptCommitment(
            commitmentId,
            principalAmount,
            collateralAmount,
            collateralTokenId,
            address(collateralToken),
            interestRate,
            loanDuration
        );
    }

```
Then run `forge test --mt test_acceptCommitment_logic_error_check -vv` in terminal, we will get:
```bash
Ran 1 test for tests/LenderCommitmentForwarder/LenderCommitmentForwarder_Unit_Test.sol:LenderCommitmentForwarder_Test
[FAIL. Reason: revert: Invalid loan max principal] test_acceptCommitment_logic_error_check() (gas: 403345)
Traces:
  [403345] LenderCommitmentForwarder_Test::test_acceptCommitment_logic_error_check()
    ├─ [169126] LenderCommitmentUser::_createCommitment(Commitment({ maxPrincipal: 100000000000000000000 [1e20], expiration: 64001 [6.4e4], maxDuration: 2480000 [2.48e6], minInterestRate: 3000, collateralTokenAddress: 0xA4AD4f68d0b91CFD19687c881e50f3A00242828c, collateralTokenId: 0, maxPrincipalPerCollateralAmount: 100, collateralTokenType: 0, lender: 0xa0Cb889707d426A7A386870A03bc70d1b0697598, marketId: 2, principalTokenAddress: 0x1d1499e622D69689cdf9004d05Ec547d650Ff211 }), [])
    │   ├─ [164569] LenderCommitmentForwarder_Override::createCommitment(Commitment({ maxPrincipal: 100000000000000000000 [1e20], expiration: 64001 [6.4e4], maxDuration: 2480000 [2.48e6], minInterestRate: 3000, collateralTokenAddress: 0xA4AD4f68d0b91CFD19687c881e50f3A00242828c, collateralTokenId: 0, maxPrincipalPerCollateralAmount: 100, collateralTokenType: 0, lender: 0xa0Cb889707d426A7A386870A03bc70d1b0697598, marketId: 2, principalTokenAddress: 0x1d1499e622D69689cdf9004d05Ec547d650Ff211 }), [])
    │   │   ├─ emit UpdatedCommitmentBorrowers(commitmentId: 0)
    │   │   ├─ emit CreatedCommitment(commitmentId: 0, lender: LenderCommitmentUser: [0xa0Cb889707d426A7A386870A03bc70d1b0697598], marketId: 2, lendingToken: TestERC20Token: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211], tokenAmount: 100000000000000000000 [1e20])
    │   │   └─ ← 0
    │   └─ ← 0
    ├─ [53706] LenderCommitmentForwarder_Override::acceptCommitment(0, 100000000000000000000 [1e20], 1000, 0, TestERC20Token: [0xA4AD4f68d0b91CFD19687c881e50f3A00242828c], 3000, 2480000 [2.48e6])
    │   ├─ emit ExercisedCommitment(commitmentId: 0, borrower: LenderCommitmentForwarder_Test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], tokenAmount: 100000000000000000000 [1e20], bidId: 1)
    │   └─ ← 1
    ├─ [526] LenderCommitmentForwarder_Override::getCommitmentMaxPrincipal(0) [staticcall]
    │   └─ ← 100000000000000000000 [1e20]
    ├─ [144726] LenderCommitmentUser::_createCommitment(Commitment({ maxPrincipal: 99999999999999999900 [9.999e19], expiration: 64001 [6.4e4], maxDuration: 2480000 [2.48e6], minInterestRate: 3000, collateralTokenAddress: 0xA4AD4f68d0b91CFD19687c881e50f3A00242828c, collateralTokenId: 0, maxPrincipalPerCollateralAmount: 100, collateralTokenType: 0, lender: 0xa0Cb889707d426A7A386870A03bc70d1b0697598, marketId: 2, principalTokenAddress: 0x1d1499e622D69689cdf9004d05Ec547d650Ff211 }), [])
    │   ├─ [142669] LenderCommitmentForwarder_Override::createCommitment(Commitment({ maxPrincipal: 99999999999999999900 [9.999e19], expiration: 64001 [6.4e4], maxDuration: 2480000 [2.48e6], minInterestRate: 3000, collateralTokenAddress: 0xA4AD4f68d0b91CFD19687c881e50f3A00242828c, collateralTokenId: 0, maxPrincipalPerCollateralAmount: 100, collateralTokenType: 0, lender: 0xa0Cb889707d426A7A386870A03bc70d1b0697598, marketId: 2, principalTokenAddress: 0x1d1499e622D69689cdf9004d05Ec547d650Ff211 }), [])
    │   │   ├─ emit UpdatedCommitmentBorrowers(commitmentId: 1)
    │   │   ├─ emit CreatedCommitment(commitmentId: 1, lender: LenderCommitmentUser: [0xa0Cb889707d426A7A386870A03bc70d1b0697598], marketId: 2, lendingToken: TestERC20Token: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211], tokenAmount: 99999999999999999900 [9.999e19])
    │   │   └─ ← 1
    │   └─ ← 1
    ├─ [2380] LenderCommitmentForwarder_Override::acceptCommitment(1, 99999999999999999900 [9.999e19], 1000, 0, TestERC20Token: [0xA4AD4f68d0b91CFD19687c881e50f3A00242828c], 3000, 2480000 [2.48e6])
    │   └─ ← revert: Invalid loan max principal
    └─ ← revert: Invalid loan max principal

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 4.53ms (393.79µs CPU time)

```
## Impact
When the value commitmentPrincipalAccepted of commitmentId=0 is bigger than  commitment.maxPrincipal of other commitmentId. `_acceptCommitment` will revert.

## Code Snippet
https://github.com/sherlock-audit/2024-04-teller-finance/blob/defe55469a2576735af67483acf31d623e13592d/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/LenderCommitmentForwarder_G2.sol#L448C5-L574C6
## Tool used

Manual Review

## Recommendation
```diff
require(
-            commitmentPrincipalAccepted[bidId] <= commitment.maxPrincipal,
+            commitmentPrincipalAccepted[_commitmentId] <= commitment.maxPrincipal,
            "Invalid loan max principal"
        );
```

