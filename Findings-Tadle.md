# Tadle 

# Table of contents

- ## Medium Risk Findings
    - ### [M-01. The protocol is incompatible with fee on transfer tokens](#M-01)
- ## Low Risk Findings
    - ### [L-01. The `DeliveryPlace::closeBidOffer` function checks if the offer status is `Virgin` instead of `Settling`](#L-01)
    - ### [L-02. The `PreMarkets::createOffer` function increments the `offerId` before updating the mappings](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Tadle

### Dates: Aug 5th, 2024 - Aug 12th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-08-tadle)
    
# Medium Risk Findings

## <a id='M-01'></a>M-01. The protocol is incompatible with fee on transfer tokens            



## Links

<https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/TokenManager.sol#L259-L261>

## Summary

The purpose of the `TokenManager::_transfer` function is to transfer ERC20 tokens between addresses. However, it is incompatible with fee-on-transfer tokens due to strict balance checks before and after the transfer.

## Vulnerability Details

The `TokenManager::_transfer` function is used to transfer a given token from `_from` address to `_to` address:

```javascript

    function _transfer(
        address _token,
        address _from,
        address _to,
        uint256 _amount,
        address _capitalPoolAddr
    ) internal {
        uint256 fromBalanceBef = IERC20(_token).balanceOf(_from);
        uint256 toBalanceBef = IERC20(_token).balanceOf(_to);

        if (
            _from == _capitalPoolAddr &&
            IERC20(_token).allowance(_from, address(this)) == 0x0
        ) {
            ICapitalPool(_capitalPoolAddr).approve(address(this));
        }

        _safe_transfer_from(_token, _from, _to, _amount);

        uint256 fromBalanceAft = IERC20(_token).balanceOf(_from);
        uint256 toBalanceAft = IERC20(_token).balanceOf(_to);

        if (fromBalanceAft != fromBalanceBef - _amount) {
            revert TransferFailed();
        }

@>      if (toBalanceAft != toBalanceBef + _amount) {
            revert TransferFailed();
        }
    }

```

The function checks the balance before and after the transfer of the sender and the receiver. Also, according to the docs of the contest the protocol should be compatible with every ERC-20 token:

```javascript
    Tokens:
      - ETH
      - WETH
      - ERC20 (any token that follows the ERC20 standard)

```

But some ERC-20 tokens are fee on transfer tokens (STA, PAXG, USDC can be in the future) and the function will not work with them because of the check `toBalanceAft != toBalanceBef + _amount` that will always revert.

## Impact

The balance checks before and after the transfer assume that the exact `_amount` of tokens will be transferred. This assumption fails for fee-on-transfer tokens, which deduct a fee during the transfer. As a result the receiver's balance will increase by less than `_amount` and the `_transfer` function will always revert. That means every function that relies on the `_transfer` function will also revert.  The `_transfer` function is called in `TokenManager::tillIn` function that is critical for the protocol functionality because it transfers a token from the user to the capital pool and the `tillIn` function is used in many important for the protocol functions.
If fee on transfer token is used the functionality of the protocol will be broken.

## Tools Used

Manual Review

## Recommendations

Implement logic to account for the fee deducted during the transfer. This involves estimating the fee and adjusting the balance checks accordingly.


# Low Risk Findings

## <a id='L-01'></a>L-01. The `DeliveryPlace::closeBidOffer` function checks if the offer status is `Virgin` instead of `Settling`            



## Links

<https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L58-L60>

## Summary

There is a logic error in the `DeliveryPlace::closeBidOffer` function. The function incorrectly checks if the `offerStatus` is `Virgin` instead of `Settling`.

## Vulnerability Details

The purpose of the `DeliveryPlace::closeBidOffer` function is to close a bid offer under specific conditions. It ensures that the offer is in the correct state and that the caller has the appropriate authority to close the bid.

```javascript

function closeBidOffer(address _offer) external {
        (
            OfferInfo memory offerInfo,
            MakerInfo memory makerInfo,
            ,
            MarketPlaceStatus status
        ) = getOfferInfo(_offer);

        if (_msgSender() != offerInfo.authority) {
            revert Errors.Unauthorized();
        }

        if (offerInfo.offerType == OfferType.Ask) {
            revert InvalidOfferType(OfferType.Bid, OfferType.Ask);
        }

        if (
            status != MarketPlaceStatus.AskSettling &&
            status != MarketPlaceStatus.BidSettling
        ) {
            revert InvaildMarketPlaceStatus();
        }

@>      if (offerInfo.offerStatus != OfferStatus.Virgin) {
            revert InvalidOfferStatus();
        }

        uint256 refundAmount = OfferLibraries.getRefundAmount(
            offerInfo.offerType,
            offerInfo.amount,
            offerInfo.points,
            offerInfo.usedPoints,
            offerInfo.collateralRate
        );

        ITokenManager tokenManager = tadleFactory.getTokenManager();
        tokenManager.addTokenBalance(
            TokenBalanceType.MakerRefund,
            _msgSender(),
            makerInfo.tokenAddress,
            refundAmount
        );

        IPerMarkets perMarkets = tadleFactory.getPerMarkets();
        perMarkets.updateOfferStatus(_offer, OfferStatus.Settled);

        emit CloseBidOffer(
            makerInfo.marketPlace,
            offerInfo.maker,
            _offer,
            _msgSender()
        );
    }
```

The offer status should be `Settling` according to the comments above the function:

```javascript

    /**
     * @notice Close bid offer
     * @dev caller must be offer authority
     * @dev offer type must Bid
     * @dev offer status must be Settling
     * @dev refund amount = offer amount - used amount
     */

```

The problem is that the function checks if the `offerStatus` is `Virgin`, which is incorrect for the intended functionality.
The correct status to check should be `Settling` to ensure the offer is in the appropriate state for closing.

## Impact

This incorrect check can prevent valid bid offers from being closed. It allows also offers in the wrong state to be closed.

## Tools Used

Manual Review

## Recommendations

The `offerStatus` should be checked against `OfferStatus.Settling` to ensure that the offer is in the correct state for closing:

```diff

function closeBidOffer(address _offer) external {
        (
            OfferInfo memory offerInfo,
            MakerInfo memory makerInfo,
            ,
            MarketPlaceStatus status
        ) = getOfferInfo(_offer);

        if (_msgSender() != offerInfo.authority) {
            revert Errors.Unauthorized();
        }

        if (offerInfo.offerType == OfferType.Ask) {
            revert InvalidOfferType(OfferType.Bid, OfferType.Ask);
        }

        if (
            status != MarketPlaceStatus.AskSettling &&
            status != MarketPlaceStatus.BidSettling
        ) {
            revert InvaildMarketPlaceStatus();
        }

-       if (offerInfo.offerStatus != OfferStatus.Virgin) {
-            revert InvalidOfferStatus();
-        }

+        if (offerInfo.offerStatus != OfferStatus.Settling) {
+            revert InvalidOfferStatus();
+        }
        .
        .
        .
```

## <a id='L-02'></a>L-02. The `PreMarkets::createOffer` function increments the `offerId` before updating the mappings            



## Links

<https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/PreMarkets.sol#L83>

## Summary

There is a logical issue in `PreMarkets::createOffer` function. The function updated the `offerId` variable before updating the `makerInfoMap`, `offerInfoMap` and `stockInfoMap` mappings.

## Vulnerability Details

The `PreMarkets::createOffer` is responsible for creating new offers. It involves generating unique addresses for the `maker`, `offer`, and `stock` based on the `offerId`:

```javascript

function createOffer(CreateOfferParams calldata params) external payable {
        /**
         * @dev points and amount must be greater than 0
         * @dev eachTradeTax must be less than 100%, decimal scaler is 10000
         * @dev collateralRate must be more than 100%, decimal scaler is 10000
         */
        if (params.points == 0x0 || params.amount == 0x0) {
            revert Errors.AmountIsZero();
        }

        if (params.eachTradeTax > Constants.EACH_TRADE_TAX_DECIMAL_SCALER) {
            revert InvalidEachTradeTaxRate();
        }

        if (params.collateralRate < Constants.COLLATERAL_RATE_DECIMAL_SCALER) {
            revert InvalidCollateralRate();
        }

        /// @dev market place must be online
        ISystemConfig systemConfig = tadleFactory.getSystemConfig();
        MarketPlaceInfo memory marketPlaceInfo = systemConfig
            .getMarketPlaceInfo(params.marketPlace);
        marketPlaceInfo.checkMarketPlaceStatus(
            block.timestamp,
            MarketPlaceStatus.Online
        );

        /// @dev generate address for maker, offer, stock.
        address makerAddr = GenerateAddress.generateMakerAddress(offerId);
        address offerAddr = GenerateAddress.generateOfferAddress(offerId);
        address stockAddr = GenerateAddress.generateStockAddress(offerId);

        if (makerInfoMap[makerAddr].authority != address(0x0)) {
            revert MakerAlreadyExist();
        }

        if (offerInfoMap[offerAddr].authority != address(0x0)) {
            revert OfferAlreadyExist();
        }

        if (stockInfoMap[stockAddr].authority != address(0x0)) {
            revert StockAlreadyExist();
        }

@>      offerId = offerId + 1;

        {
            /// @dev transfer collateral from _msgSender() to capital pool
            uint256 transferAmount = OfferLibraries.getDepositAmount(
                params.offerType,
                params.collateralRate,
                params.amount,
                true,
                Math.Rounding.Ceil
            );

            ITokenManager tokenManager = tadleFactory.getTokenManager();
            tokenManager.tillIn{value: msg.value}(
                _msgSender(),
                params.tokenAddress,
                transferAmount,
                false
            );
        }

        /// @dev update maker info
        makerInfoMap[makerAddr] = MakerInfo({
            offerSettleType: params.offerSettleType,
            authority: _msgSender(),
            marketPlace: params.marketPlace,
            tokenAddress: params.tokenAddress,
            originOffer: offerAddr,
            platformFee: 0,
            eachTradeTax: params.eachTradeTax
        });

        /// @dev update offer info
        offerInfoMap[offerAddr] = OfferInfo({
            id: offerId,
            authority: _msgSender(),
            maker: makerAddr,
            offerStatus: OfferStatus.Virgin,
            offerType: params.offerType,
            points: params.points,
            amount: params.amount,
            collateralRate: params.collateralRate,
            abortOfferStatus: AbortOfferStatus.Initialized,
            usedPoints: 0,
            tradeTax: 0,
            settledPoints: 0,
            settledPointTokenAmount: 0,
            settledCollateralAmount: 0
        });

        /// @dev update stock info
        stockInfoMap[stockAddr] = StockInfo({
            id: offerId,
            stockStatus: StockStatus.Initialized,
            stockType: params.offerType == OfferType.Ask
                ? StockType.Bid
                : StockType.Ask,
            authority: _msgSender(),
            maker: makerAddr,
            preOffer: address(0x0),
            offer: offerAddr,
            points: params.points,
            amount: params.amount
        });

        emit CreateOffer(
            offerAddr,
            makerAddr,
            stockAddr,
            params.marketPlace,
            _msgSender(),
            params.points,
            params.amount
        );
    }
```

The problem is that the function increments the `offerId` variable before updating the mappings (`makerInfoMap`, `offerInfoMap` and `stockInfoMap`). That means the `id` in the mappings will be different than the one used for generating the `makerAddr`, `offerAddr` and `stockAddr`.

## Impact

The use of the incorrect `id` in the mappings leads to inconsistencies where the generated addresses (`makerAddr`, `offerAddr`, `stockAddr`) do not correspond to the correct `offerId`.
Also, if the `createTaker` function is called immediately after the `createOffer` function, the generated address `stockAddr` and the `stockInfoMap` in `createTaker` will use the same `offerId` that corresponds to the updating mappings in the `PreMarkets::createOffer` function and that is incorrect.
The `createTaker` function increments correctly the `offerId` after updating the mapping, but the `createOffer` function doesn't update it in the correct place.

## Tools Used

Manual Review

## Recommendations

Update the `offerId` after updating the mappings to ensure that the `offerId` for the generated addresses is the same as the one used in the updating the mappings:

```diff

function createOffer(CreateOfferParams calldata params) external payable {
       .
       .
       .

        if (stockInfoMap[stockAddr].authority != address(0x0)) {
            revert StockAlreadyExist();
        }

-       offerId = offerId + 1;

        {
            /// @dev transfer collateral from _msgSender() to capital pool
            uint256 transferAmount = OfferLibraries.getDepositAmount(
                params.offerType,
                params.collateralRate,
                params.amount,
                true,
                Math.Rounding.Ceil
            );

            ITokenManager tokenManager = tadleFactory.getTokenManager();
            tokenManager.tillIn{value: msg.value}(
                _msgSender(),
                params.tokenAddress,
                transferAmount,
                false
            );
        }

        /// @dev update maker info
        makerInfoMap[makerAddr] = MakerInfo({
            offerSettleType: params.offerSettleType,
            authority: _msgSender(),
            marketPlace: params.marketPlace,
            tokenAddress: params.tokenAddress,
            originOffer: offerAddr,
            platformFee: 0,
            eachTradeTax: params.eachTradeTax
        });

        /// @dev update offer info
        offerInfoMap[offerAddr] = OfferInfo({
            id: offerId,
            authority: _msgSender(),
            maker: makerAddr,
            offerStatus: OfferStatus.Virgin,
            offerType: params.offerType,
            points: params.points,
            amount: params.amount,
            collateralRate: params.collateralRate,
            abortOfferStatus: AbortOfferStatus.Initialized,
            usedPoints: 0,
            tradeTax: 0,
            settledPoints: 0,
            settledPointTokenAmount: 0,
            settledCollateralAmount: 0
        });

        /// @dev update stock info
        stockInfoMap[stockAddr] = StockInfo({
            id: offerId,
            stockStatus: StockStatus.Initialized,
            stockType: params.offerType == OfferType.Ask
                ? StockType.Bid
                : StockType.Ask,
            authority: _msgSender(),
            maker: makerAddr,
            preOffer: address(0x0),
            offer: offerAddr,
            points: params.points,
            amount: params.amount
        });

+       offerId = offerId + 1;

        emit CreateOffer(
            offerAddr,
            makerAddr,
            stockAddr,
            params.marketPlace,
            _msgSender(),
            params.points,
            params.amount
        );
    }

```



