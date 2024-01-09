## What is Morpho Blue and MetaMorpho

Morpho Blue and MetaMorpho form part of the vision to rebuild decentralized lending in layers, with MetaMorpho enabling any lending experience to be rebuilt on a shared and immutable base layer: Morpho Blue.

Morpho Blue is a trustless lending primitive that offers unparalleled efficiency and flexibility. It enables the creation of isolated lending markets by specifying any loan asset, any collateral asset, a liquidation LTV (LLTV), an oracle, and an interest rate model.



### [M-01]- Partial liquidation can leave hanging borrow shares

In the `liquidate` function, when no bad debt is realized(the position is unhealthy but doesn't result in losses for the protocol) all borrower's borrow shares should be liquidated at the end of the liquidation, and some collateral must be left. This is what happens when the whole position is liquidated in one single transaction.

However, it's not always the case for partial liquidations. Since the `liquidate` function allows for the liquidator to choose either how much collateral wants to seize or how much borrow shares he's going to repay, a liquidator could partially liquidate a unhealthy position. Well, in cases where a partial liquidation is done by providing the shares to repay, the position could become healty again, and not all the borrow shares would be liquidated as expected and happened in the full liquidation. This should not occur in any case. In theory, the liquidator repay's borrower debt and withdraws (debt value + liquidator incentive) value from the collateral, so the position would actually become slightly more unhealthy, and not the opposite.

This is due to a rounding error. The conversion from repaidShares to repaidAssets is done by rounding up while the conversion from repaidAssets to seizedAssets is done by rounding down, potentially making the position healthy again, and therefore preventing the remaining liquidations from happening.

Besides of the severity of function leading to an unexptected healthy position, this results in the borrower paying less liquidation incentives than it should.

Note that the lower the difference between the current LLTV and the target LLTV, the more borrow shares can be left hanging, because the closer the borrower is from being in a healthy position again, so less borrow shares need to be liquidated in order for this to happen. This is a very likley scenario, because liquidator act very fast usually.

#### Proof of Concept:
    
    function testHangingBorrowSharesNoBadDebtRealized() public {
        // settings: default test environment settings => 80% lltv

        // some collateral amount for this loan
        uint256 amountCollateral = 20000 ether; // 20000000000000000000000
        console.log("Initial collateral : ", amountCollateral);
        // number of loan tokens you get with 1 collateral token(scaled to 1e36)
        uint256 collateralPrice = 2 * ORACLE_PRICE_SCALE;
        // the amount borrowed is 80% of colateral value(the limit => lltv is 80%)
        uint256 amountBorrowed = amountCollateral * (collateralPrice) / (ORACLE_PRICE_SCALE) * 80 / 100; // = 32000000000000000000000
        //supply enough amount for the market to lend the loan token
        uint256 amountSupplied = amountBorrowed * 20;
        _supply(amountSupplied);
        // we set the price in the mock oracle
        oracle.setPrice(collateralPrice);
        collateralToken.setBalance(BORROWER, amountCollateral);

        // we deposit collateral and borrow as the borrower
        vm.startPrank(BORROWER);
        morpho.supplyCollateral(marketParams, amountCollateral, BORROWER, hex"");
        (uint256 assets, uint256 shares) = morpho.borrow(marketParams, amountBorrowed, 0, BORROWER, RECEIVER);

        // the amount of borrow shares NOT TO LIQUIDATE in the error path
        uint256 borrowSharesDelta = shares / 2;
        vm.stopPrank();

        // someone else borrows
        // we make this to make sure this is not valid for markets with 1 borrower only
        vm.startPrank(LIQUIDATOR);
        collateralToken.setBalance(LIQUIDATOR, amountCollateral);
        morpho.supplyCollateral(marketParams, amountCollateral, LIQUIDATOR, hex"");
        morpho.borrow(marketParams, amountBorrowed, 0, LIQUIDATOR, LIQUIDATOR);
        vm.stopPrank();

        // collateral value drops so the position is unhealthy and can be liquidated
        oracle.setPrice(collateralPrice * 99 / 100);

        console.log("===============BORROW=================");
        console.log("BORROWER BORROW SHARES : ", shares);
        loanToken.setBalance(LIQUIDATOR, amountBorrowed * 3);

        // save current state to come back later
        uint256 snapshotId = vm.snapshot();

        // in the full liquidation(happy path) the liquidator  will pay all the borrow shares
        // and therefore borrowShares == 0 in the end 
        uint256 sharesRepaid = shares; // = 32000000000000000000000000000
        console.log("===============HAPPY PATH=================");
        console.log("===============LIQUIDATE FULLY=================");
        vm.prank(LIQUIDATOR);
        // liquidate the position all at once
        (uint256 seized, uint256 repaid) = morpho.liquidate(marketParams, BORROWER, 0, sharesRepaid, hex"");
        console.log("SEIZED COLLATERAL 1: ", seized); // 17193208682570384694949
        console.log("REPAID SHARES 1: ", repaid); // 32000000000000000000000
        Position memory pos = morpho.position(id, BORROWER);
        console.log("===============HAPPY PATH RESULTS====================== ");
        console.log("BORROWER BORROW SHARES : ", pos.borrowShares); //  0
        console.log("BORROWER COLLATERAL ASSETS : ", pos.collateral); // 2806791317429615305051

        // borrow shares must be 0 at the end
        assertEq(pos.borrowShares, 0);
        assertEq(pos.collateral, 2806791317429615305051);

        // go back to the initial setting
        vm.revertTo(snapshotId);
        
        // now for the error path the first liquidator will do a partial liquidation
        // amount = sharesRepaid - borrowSharesDelta
        console.log("===============ERROR PATH=================");
        sharesRepaid -= borrowSharesDelta;

        // liquidate as specified
        vm.prank(LIQUIDATOR);
        console.log("===============LIQUIDATE 1=================");
        (seized, repaid) = morpho.liquidate(marketParams, BORROWER, 0, sharesRepaid, hex"");
        console.log("SEIZED COLLATERAL 1: ", seized); // 8596604341285192347474
        console.log("REPAID SHARES 1: ", repaid); // 16000000000000000000000

        // now someone tries to liquidate the remaining collateral with seized assets as input
        // SURPRISE! THE POSITION IS HEALTHY!
        vm.prank(LIQUIDATOR);
        console.log("===============TRY LIQUIDATE 2 WITH ASSETS=================");
        pos = morpho.position(id, BORROWER);
        // it will revert since the contract will identify the position as healthy
        vm.expectRevert(bytes(ErrorsLib.HEALTHY_POSITION));
        (seized, repaid) = morpho.liquidate(marketParams, BORROWER, uint256(pos.collateral/2), 0, hex"");
        console.log("SEIZED COLLATERAL 2: ", seized);
        console.log("REPAID SHARES 2: ", repaid);

        // now someone tries to liquidate it but using the shares instead of assets
        // STILL HEALTHY!
        vm.prank(LIQUIDATOR);
        console.log("===============TRY LIQUIDATE 2 WITH SHARES=================");
        // the same will happen
        vm.expectRevert(bytes(ErrorsLib.HEALTHY_POSITION));
        // there is one share remaining so liquidate it
        (seized, repaid) = morpho.liquidate(marketParams, BORROWER, 0, pos.borrowShares, hex"");
        console.log("SEIZED COLLATERAL 2: ", seized);
        console.log("REPAID SHARES 2: ", repaid);
        pos = morpho.position(id, BORROWER);
        // result : not all borrow shares have been liquidated, instead the position has become healthy
        // after the first partial liquidation
        console.log("===============ERROR PATH RESULTS====================== ");
        console.log("BORROWER BORROW SHARES : ", pos.borrowShares);  //  16000000000000000000000000000
        console.log("BORROWER COLLATERAL ASSETS : ", pos.collateral); // 11403395658714807652526

        // the numbers are different if not liquidated all at once
        assertEq(pos.borrowShares, borrowSharesDelta); 
        assertEq(pos.collateral, 11403395658714807652526); 
    }

#### Recommendation: 
The current rounding is done in favor of the borrower. It's recommended to do in favor of the liquidator, in order to prevent this error from happening.

### [L-01]- Missing address(this) check results in locked funds.

This applies for two of the functions that allow calling them on behalf of an address without the need of their explicit auhorization(Morpho::supply and Morpho::addCollateral). If done on behalf of the contract address caller's funds will get permanently stuck in the contract.

#### Recommendation: 
Add an extra security check in those functions, not allowing to operate on behalf of address(this).

### [L-02]- Limited marked supply. 

At present, the lending protocol sets the maximum market supply at the maximum value of uint128. While this limit appears generous for most tokens, it poses potential constraints for tokens with either high supplies or an extensive number of decimals. Notably, tokens with 36 decimals are capped at 340(2128 / 10 36), which could be insufficient. While unlikely for the high decimals case, this limitation becomes more pronounced for tokens with exceptionally high supplies(this has same effect as a high number of decimals), potentially impeding the protocol's ability to effectively handle tokens with significant diluted values(e.g. SHIB token).

#### Recommendation: 
Manage market supplies with a more generous unsigned integer type such as uint256.

### [L-03]- Persistens stuck tokens without function invocation. 

Tokens sent directly to the contract without interacting with any of its functions become permanently stuck, creating a scenario where these assets are inaccessible within the lending platform, and cannot be absorbed into the protocol, nor be withdrawn.

#### Recommendation: 
Add a skim function similar to Uniswap V2 so the locked tokens are added to a market supply, so the tokens aren't wasted and they contribute to the stability of the supply.
// reinvest stuck funds into a market.

    function skim(Id id, address token) external onlyOwner {
        require (idToMarketParams[id].loanToken == token, "Invalid market");
        uint256 totalBalance = IERC20(token).balanceOf(address(this));
        // add mapping(address => uint256) _totalBalance
        // and update it on every interaction where the totalBalance changes
        uint256 cacheBalance = _totalBalance[token];
        uint256 newSupply = totalBalance - cacheBalance;
        market[id].totalSupplyAssets += newSupply;
        _totalBalance = totalBalance;
    }

### [I-01]- Add names return variables. 

Consider adding named return variables in the funcions for a better readability and slight gas optimization.

### [I-02]- 2 transaction signature authorization.

One of the main purposes of doing approvals with off-chain generated signatures, is reducing the number of steps required for an action that requires an approval , such as transferFrom function in ERC20 , that requires the allower to call approve first. in ERC20 this 2 steps are reduced to 1 thanks to the ERC20Permit standard, using the permit function that uses EIP712 signatures. In the case of Morpho, includes the setAuthorizationWithSig that allows the authorized address to aquire the authorization himself, using allower's off-chain generated signature. This process would still involve 2 transactions.

#### Recommendation: 
In order to get the most out of signatures, protocol could have functions that allow to do all the process in one single transaction :

    function borrowWithSig(
        MarketParams memory marketParams,
        uint256 assets,
        uint256 shares,
        address onBehalf,
        address receiver,
        Signature calldata signature
    ) external returns (uint256, uint256) {
    //...
    }

    function withdrawWithSignature(
        MarketParams memory marketParams,
        uint256 assets,
        uint256 shares,
        address onBehalf,
        address receiver,
        Signature calldata signature
    ) external returns (uint256, uint256) {
    //...
    }
    function withdrawCollateral(
        MarketParams memory marketParams, 
        uint256 assets, 
        address onBehalf, 
        address receiver,
        Signature calldata signature
    ) external{
    //...
    }

#### Note: 
The inteded behaviour is that once calling one of this functions the authorized address can get can is approved for later interactions, so it doesnt need the allower being active and providing a signature every time.
