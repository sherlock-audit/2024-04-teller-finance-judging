Fancy Raspberry Puppy

medium

# SmartCommitmentForwarder.acceptCommitmentWithRecipient does not work when the loan requires no collateral

## Summary
The [`SmartCommitmentForwarder.acceptCommitmentWithRecipient`](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/SmartCommitmentForwarder.sol#L38C14-L38C43) will always revert when the smart commitment contract doesn't require collateral. 

## Vulnerability Detail
`SmartCommitmentForwarder` forwards bid creation to `TellerV2` contract, and later communicate with `SmartCommitment` contract to accept a loan. In `_acceptCommitment`, which is called by  `acceptCommitmentWithRecipient` as an internal function:

```solidity
    function _acceptCommitment(
        address _smartCommitmentAddress,
        uint256 _principalAmount,
        uint256 _collateralAmount,
        uint256 _collateralTokenId,
        address _collateralTokenAddress,
        address _recipient,
        uint16 _interestRate,
        uint32 _loanDuration
    ) internal returns (uint256 bidId) {
        ISmartCommitment _commitment = ISmartCommitment(
            _smartCommitmentAddress
        );

        CreateLoanArgs memory createLoanArgs;

        createLoanArgs.marketId = _commitment.getMarketId();
        createLoanArgs.lendingToken = _commitment.getPrincipalTokenAddress();
        createLoanArgs.principal = _principalAmount;
        createLoanArgs.duration = _loanDuration;
        createLoanArgs.interestRate = _interestRate;
        createLoanArgs.recipient = _recipient;

        CommitmentCollateralType commitmentCollateralTokenType = _commitment
            .getCollateralTokenType();

        if (commitmentCollateralTokenType != CommitmentCollateralType.NONE) {
            createLoanArgs.collateral = new Collateral[](1);
            createLoanArgs.collateral[0] = Collateral({
                _collateralType: _getEscrowCollateralType(
                    commitmentCollateralTokenType
                ),
                _tokenId: _collateralTokenId,
                _amount: _collateralAmount,
                _collateralAddress: _collateralTokenAddress // commitment.collateralTokenAddress
            });
        }

        bidId = _submitBidWithCollateral(createLoanArgs, _msgSender());

        _commitment.acceptFundsForAcceptBid(
            _msgSender(), //borrower
            bidId,
            _principalAmount,
            _collateralAmount,
            _collateralTokenAddress,
            _collateralTokenId,
            _loanDuration,
            _interestRate
        );

        emit ExercisedSmartCommitment(
            _smartCommitmentAddress,
            _msgSender(),
            _principalAmount,
            bidId
        );
    }
```

The function checks collateral type, then adds the collateral info as parameter before eventually forwards to the `TellerV2` contract. But in the case of no collateral, which `commitmentCollateralTokenType ` is `NONE`, it would forward an empty struct of `Collateral[]`, which would fail. The `TellerV2` contract supports the submission of a bid with no collateral, as we can see in the comments:

```solidity
    /**
     * @notice Function for a borrower to create a bid for a loan without Collateral.
     * @param _lendingToken The lending token asset requested to be borrowed.
     * @param _marketplaceId The unique id of the marketplace for the bid.
     * @param _principal The principal amount of the loan bid.
     * @param _duration The recurrent length of time before which a payment is due.
     * @param _APR The proposed interest rate for the loan bid.
     * @param _metadataURI The URI for additional borrower loan information as part of loan bid.
     * @param _receiver The address where the loan amount will be sent to.
     */
    function submitBid(
        address _lendingToken,
        uint256 _marketplaceId,
        uint256 _principal,
        uint32 _duration,
        uint16 _APR,
        string calldata _metadataURI,
        address _receiver
    ) public override whenNotPaused returns (uint256 bidId_) {
        bidId_ = _submitBid(
            _lendingToken,
            _marketplaceId,
            _principal,
            _duration,
            _APR,
            _metadataURI,
            _receiver
        );
    }
```

But `TellerV2MarketForwarder_G2._submitBidWithCollateral` only supports the other function which has collateral parameters in it:

```solidity
        responseData = _forwardCall(
            abi.encodeWithSignature(
                "submitBid(address,uint256,uint256,uint32,uint16,string,address,(uint8,uint256,uint256,address)[])",
                _createLoanArgs.lendingToken,
                _createLoanArgs.marketId,
                _createLoanArgs.principal,
                _createLoanArgs.duration,
                _createLoanArgs.interestRate,
                _createLoanArgs.metadataURI,
                _createLoanArgs.recipient,
                _createLoanArgs.collateral
            ),
            _borrower
        );
```

## Impact
For now, the only `SmartCommitment` contract is `LenderCommitmentGroup_Smart` and has the collateral type hardcoded as ERC20, but for future updates which may support no collateral loans, this may make the feature broken.

## Code Snippet
```solidity
    function _acceptCommitment(
        address _smartCommitmentAddress,
        uint256 _principalAmount,
        uint256 _collateralAmount,
        uint256 _collateralTokenId,
        address _collateralTokenAddress,
        address _recipient,
        uint16 _interestRate,
        uint32 _loanDuration
    ) internal returns (uint256 bidId) {
        ISmartCommitment _commitment = ISmartCommitment(
            _smartCommitmentAddress
        );

        CreateLoanArgs memory createLoanArgs;

        createLoanArgs.marketId = _commitment.getMarketId();
        createLoanArgs.lendingToken = _commitment.getPrincipalTokenAddress();
        createLoanArgs.principal = _principalAmount;
        createLoanArgs.duration = _loanDuration;
        createLoanArgs.interestRate = _interestRate;
        createLoanArgs.recipient = _recipient;

        CommitmentCollateralType commitmentCollateralTokenType = _commitment
            .getCollateralTokenType();

        if (commitmentCollateralTokenType != CommitmentCollateralType.NONE) {
            createLoanArgs.collateral = new Collateral[](1);
            createLoanArgs.collateral[0] = Collateral({
                _collateralType: _getEscrowCollateralType(
                    commitmentCollateralTokenType
                ),
                _tokenId: _collateralTokenId,
                _amount: _collateralAmount,
                _collateralAddress: _collateralTokenAddress // commitment.collateralTokenAddress
            });
        }

        bidId = _submitBidWithCollateral(createLoanArgs, _msgSender());

        _commitment.acceptFundsForAcceptBid(
            _msgSender(), //borrower
            bidId,
            _principalAmount,
            _collateralAmount,
            _collateralTokenAddress,
            _collateralTokenId,
            _loanDuration,
            _interestRate
        );

        emit ExercisedSmartCommitment(
            _smartCommitmentAddress,
            _msgSender(),
            _principalAmount,
            bidId
        );
    }
```

## Tool used

Manual Review

## Recommendation
Add an if check on `NONE` collaterals, and in this case, forwards the function to `submitBid(address,uint256,uint256,uint32,uint16,string,address)`
