## What is Superform

Superform is the first non-custodial yield marketplace.

### [M-01]- Vaults write function return values could not reflect the real output

When the superform calls either deposit or redeem methods of the underlying vault, it uses the return value of those functions to fetch the amount of shares minted on deposit and the amount of assets withdrawn on redeem. Some vaults could not reflect the reality in their return values(maybe they decided to return an empty value, or returns the shares value when redeeming, instead of the real assets might be less due to slippage of swaps in the withdraw process) and this would make the superform have an incorrect accounting of it's shares and superpositions, potentially accounting users' positions unfairly.

#### Recommendation: 
Fetch the real output directly from the shares or assets balance, using balanceOf function for extra security. The contract could even require the return value to equal the real output for more sanity.

    if (singleVaultData.retain4626) {
            uint256 balanceBefore = v.balanceOf(address(singleVaultData.receiverAddress));
            v.deposit(vars.assetDifference, singleVaultData.receiverAddress);
            dstAmount = v.balanceOf(address(singleVaultData.receiverAddress)) - balanceBefore;
        } else {
            //...
    }
### [M-02]- No vault shares price change protection

The users could get an undesired output from either a deposit(not enough shares) or a redeem(less assets than expected), potentially incurring economic losses for the user. Some reasons for this to happen:

The vault needs to perform swaps in the withdrawal process incurring slippage for the user.
Impermanent loss: the vault holds some position in a DEX and the lp price has changed.
Any kind of attacks that could indirectly or directly affect the vaults: oracle manipulations, vault inflation attacks...
This is specially likely to happen in crosschain operations, where some delay may exist from the moment the order is placed to the completion of the operation.

Additionally, a router is meant to work as an intermediary contract that performs strict security checks to control the process and allow the user to interact with the protocol safely.

#### Recommendation: 
The same way there is a slippage protection for liquidity transactions, add an extra parameter to the SingleVaultSFData that specifies the minimum expected shares for deposits and minimum assets for redeems for vault protection, and require the needs are satisfied when doing the final deposit/redeem:

    struct SingleVaultSFData {
        // superformids must have same destination. Can have different underlyings
        uint256 superformId;
        uint256 amount;
        uint256 maxSlippage;
        uint256 minOutput; // min shares/assets
        LiqRequest liqRequest; // if length = 1; amount = sum(amounts)| else  amounts must match the 
        amounts being sent
        bytes permit2data;
        bool hasDstSwap;
        bool retain4626; // if true, we don't mint SuperPositions, and send the 4626 back to the user instead
        address receiverAddress;
        bytes extraFormData; // extraFormData
    }
    
    function directDepositIntoVault(
        InitSingleVaultData memory singleVaultData_,
        address srcSender_
    )
        external
        payable
        override
        onlySuperRouter
        notPaused(singleVaultData_)
        returns (uint256 dstAmount)
    {
        dstAmount = _directDepositIntoVault(singleVaultData_, srcSender_);
        if (dstAmount < singleVaultData_.minOutput) revert NOT_ENOUGH_SHARES();
    }
Note that for an extra security you can even add a deadline parameter to make sure orders are not executed later.

### [L-01]- Users who accidentally specify an invalid superformId in cross-chain deposits will result in loss of funds

This is an intended behaviour but still a bad practice, the user could do this accidentally, plus the protocol won't benefit from this stuck funds neither.

The SuperformFactory of each chain only tracks the superformId of the forms created in that specificchain. Therefore, the contract can only check if the superform exists before performing the whole operation in the direct deposits. In the case of the crosschain deposits, there's no way if the operation is gonna fail due to a non-existant superformId at first glance:

    // only knows if it exists if it's a direct deposit
     if (dstChainId_ == CHAIN_ID && !factory_.isSuperform(superformId_)) {
            return false;
     }
    // it will pass this one too, since the default value for that mapping is NON_PAUSED
    if (isDeposit_ && factory_.isFormImplementationPaused(formImplementationId)) return false;
    //...


This means that if a user specifies a superformId that doesn't exist in the desintation chain the first crosschain transaction could go through. The user would pay for the gas required for the AMBs to dispatch the payload (could be very expensive if the source chain is mainnet) and finally get his funds stuck in the destination CoreStateRegistry . This would happen in the updateDepositPayload function:


            if (
                !(
                    ISuperformFactory(_getAddress(keccak256("SUPERFORM_FACTORY"))).isSuperform(superformId_)
                        && finalState_ == PayloadState.UPDATED
                )
            ) {
                failedDeposits[payloadId_].superformIds.push(superformId_);

                address asset;
                try IBaseForm(_getSuperform(superformId_)).getVaultAsset() returns (address asset_) {
                    asset = asset_;
                } catch {
                    /// @dev if its error, we just consider asset as zero address
                }
                /// @dev if superform is invalid, try catch will fail and asset pushed is address (0)
                /// @notice this means that if a user tries to game the protocol with an invalid superformId, the funds
                /// bridged over that failed will be stuck here
                failedDeposits[payloadId_].settlementToken.push(asset);
                failedDeposits[payloadId_].settleFromDstSwapper.push(false);

                /// @dev sets amount to zero and will mark the payload as PROCESSED (overriding the previous memory
                /// settings)
                amount_ = 0;
                finalState_ = PayloadState.PROCESSED;
                
Even considering that users will interact with the contracts from the frontend and params will be set from the Superform API a frontend attack could modify the params or smart contracts interacting with the contracts directly on-chain could make a mistake.

#### Recommendation: 
Broadcast the transaction when a superform is created,so all the factories are up to date with the existing superforms. This is not a problem since the superformId cannot collide: their chainId makes their Id unique. This would add an extra robustness layer and paramters sanity, reverting at the beginning of the operation so the user cannot lead his tokens to being stuck forever. Another option is to add a admin role skim function in the CoreStateRegistryso lost funds can be withdrawn.

### [L-02]- The router allows to mint 0 shares on deposit

A user can deposit in a vault that mints 0 shares and the transaction would still go through. ERC4626 is meant to revert when shares are 0, but a different implementation could not revert. This could result in the user losing his funds since he will not be able to withdraw anything with 0 shares.

#### Recommendation:
Revert if no shares are minted in any deposit.

### [I-01]- AMBs can dublicate proofs and increase the quorum any number of times

The BaseStateRegistry::dispatchPayload function requires the AMB ids to be different so no proof is duplicated. But a malicious AMB could dispatch a proof more than once and reach quorum bypassing other AMB's authority:

    function receivePayload(uint64 srcChainId, bytes memory message) external override 
    // it checks that the sender is a AMB implementation
    onlyValidAmbImplementation {


        AMBMessage memory data = abi.decode(message, (AMBMessage));

        if (data.params.length == 32) {
            bytes32 proofHash = abi.decode(data.params, (bytes32));
            // the AMB can increase the quorum any number of times
            // it doesnt check that the AMB has submitted the proof hash already 
            ++messageQuorum[proofHash];

            emit ProofReceived(data.params);
        } 
#### Recommendation: 
Create a mapping that tracks if a address has already submitted a proof to prevent duplicates, and revert if the same AMB submits the same proof more than once.

    mapping(bytes32 => mapping(address => bool)) dispatched;
    //...
    if (data.params.length == 32) {
            bytes32 proofHash = abi.decode(data.params, (bytes32));
           if(dispatched[proofHash][msg.sender])  revert DUPLICATE_PROOF();
            ++messageQuorum[proofHash];

            emit ProofReceived(data.params);
    }
    
### [I-02]- Inconsistent minting process

In the same chain deposits the one that mints and burns the supershares is the router while in the crosschain ones is the state registry the one doing it, resulting in a less cleaner design.

#### Recommendation: 
Perform all the actions from a same type of contract

