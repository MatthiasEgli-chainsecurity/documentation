# Participation

## participation.sol

Description

...

Inherits from Spoke, DSMath (links)

On Construction
The contract becomes a spoke of the hub.


Structs

`Request`

    Member variables:

    `address investmentAsset`
    `uint investmentAmount`
    `uint requestedShares`
    `uint timestamp`
    `uint atUpdateId`


Enums
None.

Public State Variables

`mapping (address => Request) public requests`

`uint public SHARES_DECIMALS`



Public Functions

`function requestInvestment(uint requestedShares, uint investmentAmount, address investmentAsset) external`


// TODO: implement and use below modifiers
// pre_cond(compliance.isInvestmentPermitted(msg.sender, giveQuantity, shareQuantity))    
// Compliance Module: Investment permitted
{
    require(!hub.isShutDown());
    requests[msg.sender] = Request({
        investmentAsset: investmentAsset,
        investmentAmount: investmentAmount,
        requestedShares: requestedShares,
        timestamp: block.timestamp,
        atUpdateId: CanonicalPriceFeed(routes.priceSource).updateId() // TODO: can this be abstracted away?
    });
    SHARES_DECIMALS = 18;
}

`function cancelRequest() external`  

{
    delete requests[msg.sender];
}

`function executeRequest() public`  

{
        executeRequestFor(msg.sender);
    }

`function executeRequestFor(address requestOwner) public`
        // TODO: implement and use below modifiers
        // pre_cond(
        //     Shares(routes.shares).totalSupply() == 0 ||
        //     (
        //         now >= add(requests[id].timestamp, priceSource.getInterval()) &&
        //         priceSource.updateId() >= add(requests[id].atUpdateId, 2)
        //     )
        // )
    {
        require(!hub.isShutDown());
        PolicyManager(routes.policyManager).preValidate(bytes4(sha3("executeRequestFor(address)")), [requestOwner, address(0), address(0), address(0), address(0)], [uint(0), uint(0), uint(0)], "0x0");
        Request memory request = requests[requestOwner];
        require(request.requestedShares > 0);
        bool isRecent;
        (isRecent, , ) = CanonicalPriceFeed(routes.priceSource).getPriceInfo(address(request.investmentAsset));
        require(isRecent);

        // sharePrice quoted in QUOTE_ASSET and multiplied by 10 ** fundDecimals
        uint costQuantity; // TODO: better naming after refactor (this variable is how much the shares wanted cost in total, in the desired payment token)
        costQuantity = mul(request.requestedShares, Accounting(routes.accounting).calcSharePriceAndAllocateFees()) / 10 ** SHARES_DECIMALS; // By definition quoteDecimals == fundDecimals
        // TODO: maybe allocate fees in a separate step (to come later)
        if(request.investmentAsset != address(Accounting(routes.accounting).QUOTE_ASSET())) {
            bool isPriceRecent;
            uint invertedInvestmentAssetPrice;
            uint investmentAssetDecimal;
            (isPriceRecent, invertedInvestmentAssetPrice, investmentAssetDecimal) = CanonicalPriceFeed(routes.priceSource).getInvertedPriceInfo(request.investmentAsset);
            // TODO: is below check needed, given the recency check a few lines above?
            require(isPriceRecent);
            costQuantity = mul(costQuantity, invertedInvestmentAssetPrice) / 10 ** investmentAssetDecimal;
        }

        if (
            // isInvestAllowed[request.investmentAsset] &&
            costQuantity <= request.investmentAmount
        ) {
            delete requests[requestOwner];
            require(ERC20(request.investmentAsset).transferFrom(requestOwner, address(routes.vault), costQuantity));
            // Allocate Value
            Shares(routes.shares).createFor(requestOwner, request.requestedShares);
            // // TODO: this should be done somewhere else
            if (!Accounting(routes.accounting).isInAssetList(request.investmentAsset)) {
                Accounting(routes.accounting).addAssetToOwnedAssets(request.investmentAsset);
            }
        } else {
            revert(); // Invalid Request or invalid giveQuantity / receiveQuantity
        }
    }

    /// @dev "Happy path" (no asset throws & quantity available)
    /// @notice Redeem all shares and across all assets
    `function redeem() public`

    {
        uint ownedShares = Shares(routes.shares).balanceOf(msg.sender);
        redeemQuantity(ownedShares);
    }

    /// @notice Redeem shareQuantity across all assets
    `function redeemQuantity(uint shareQuantity) public`


    {
        address[] memory assetList;
        (, assetList) = Accounting(routes.accounting).getFundHoldings();
        require(redeemWithConstraints(shareQuantity, assetList)); //TODO: assetList from another module
    }

    // NB1: reconsider the scenario where the user has enough funds to force shutdown on a large trade (any way around this?)
    // TODO: readjust with calls and changed variable names where needed
    // @dev Redeem only selected assets (used only when an asset throws)
    `function redeemWithConstraints(uint shareQuantity, address[] requestedAssets) public returns (bool)`


    {
        Accounting accounting = Accounting(routes.accounting);
        Shares shares = Shares(routes.shares);
        Vault vault = Vault(routes.vault);
        require(shares.balanceOf(msg.sender) >= shareQuantity);
        address ofAsset;
        uint[] memory ownershipQuantities = new uint[](requestedAssets.length);
        address[] memory redeemedAssets = new address[](requestedAssets.length);

        // Check whether enough assets held by fund
        for (uint i = 0; i < requestedAssets.length; ++i) {
            ofAsset = requestedAssets[i];
            require(accounting.isInAssetList(ofAsset));
            for (uint j = 0; j < redeemedAssets.length; j++) {
                if (ofAsset == redeemedAssets[j]) {
                    revert();
                }
            }
            redeemedAssets[i] = ofAsset;
            uint quantityHeld = accounting.assetHoldings(ofAsset);
            if (quantityHeld == 0) continue;

            // participant's ownership percentage of asset holdings
            ownershipQuantities[i] = mul(quantityHeld, shareQuantity) / shares.totalSupply();

            // TODO: do we want to represent this as an error and shutdown, or do something else? See NB1 scenario above
            // CRITICAL ERR: Not enough fund asset balance for owed ownershipQuantitiy, eg in case of unreturned asset quantity at address(exchanges[i].exchange) address
            // if (uint(ERC20(ofAsset).balanceOf(address(vault))) < ownershipQuantities[i]) {
            //     isShutDown = true; // TODO: external call most likely
            //     // emit ErrorMessage("CRITICAL ERR: Not enough quantityHeld for owed ownershipQuantitiy");
            //     return false;
            // }
        }

        shares.destroyFor(msg.sender, shareQuantity);

        // Transfer owned assets
        for (uint k = 0; k < requestedAssets.length; ++k) {
            ofAsset = requestedAssets[k];
            if (ownershipQuantities[k] == 0) {
                continue;
            } else {
                vault.withdraw(ofAsset, ownershipQuantities[k]);
                require(ERC20(ofAsset).transfer(msg.sender, ownershipQuantities[k]));
            }
        }
        return true;
    }
}


Document factory?
