# Zaros Part 1

# Table of contents
- ## High Risk Findings
    - ### [H-01. Incorrect order execution in the `SettlementBranch::fillOffchainOrders` function](#H-01)
- ## Medium Risk Findings
    - ### [M-01. The checks that confirm that the sequencer is up in `ChainlinkUtil::getPrice` function are inefficient](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: Zaros

### Dates: Jul 17th, 2024 - Jul 31st, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-zaros)

# High Risk Findings

## <a id='H-01'></a>H-01. Incorrect order execution in the `SettlementBranch::fillOffchainOrders` function            


## Summary

There is an inconsistency between the comment and the implementation of the order execution logic in the `SettlementBranch::fillOffchainOrders` function. The comment describes the desired behavior for filling orders, but the code implements the opposite logic.
Link: <https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/perpetuals/branches/SettlementBranch.sol#L283-L287>

## Vulnerability Details

The `SettlementBranch::fillOffchainOrders` function fills pending, eligible offchain orders targeting the given market id. The problem is that the function should ensure that if the order increases the trading account's position (buy order), the fill price must be less than or equal to the target price, if it decreases the trading account's position (sell order), the fill price must be greater than or equal to the target price. But the code actually does the opposite:

```javascript

    function fillOffchainOrders(
        uint128 marketId,
        OffchainOrder.Data[] calldata offchainOrders,
        bytes calldata priceData
    )
        external
        onlyOffchainOrdersKeeper(marketId)
    {
       .
       .
       .

        // if the order increases the trading account's position (buy order), the fill price must be less than or
        // equal to the target price, if it decreases the trading account's position (sell order), the fill price
        // must be greater than or equal to the target price.
@>      ctx.isFillPriceValid = (ctx.isBuyOrder && ctx.offchainOrder.targetPrice <= ctx.fillPriceX18.intoUint256())
            || (!ctx.isBuyOrder && ctx.offchainOrder.targetPrice >= ctx.fillPriceX18.intoUint256());

        .
        .
        .
    }


```

In the contest's README it is written:

```javascript
**NOTE**
_**The repo code is the final word on functionality. Protocol documentation may not be the most up to date.**_
```

But in the `SettlementBranch::fillOffchainOrders` function there is a comment:

```javascript
// if the order increases the trading account's position (buy order), the fill price must be less than or
// equal to the target price, if it decreases the trading account's position (sell order), the fill price
// must be greater than or equal to the target price.

```

But the current code logic is that: for buy order, the fill price must be greater than or equal to the target price. And for the sell order, the fill price must be less than or equal to the target price.
That is the opposite of the comment and because the repo code and comments in it are the final word on functionality, there is an issue in the code. The current version of the code doesn't implement the required functionality.

## Impact

The `isFillPriceValid` will be `true`, for example, in the case when the order increases the trading account's position and the fill price is greater than or equal to the target price. Also, it will be `true` in the case when the order decreases the trading account's position and the fill price is less than or equal to the target price.

But according to the the comment above the code the `isFillPriceValid` must be `true` in the cases: when the order increases the trading account's position and the fill price is less than or equal to the target price or when the order decreases the trading account's position and the fill price is greater than or equal to the target price.
The incorrect order execution leads to orders that might be executed at prices higher than the target price, leading to potential overpayment (for buy orders) and orders that might be executed at prices lower than the target price, leading to potential underselling (for sell orders).

## Tools Used

Manual Review

## Recommendations

Change the code to align the comment above:

```diff
     function fillOffchainOrders(
        uint128 marketId,
        OffchainOrder.Data[] calldata offchainOrders,
        bytes calldata priceData
    )
        external
        onlyOffchainOrdersKeeper(marketId)
    {
       .
       .
       .

        // if the order increases the trading account's position (buy order), the fill price must be less than or
        // equal to the target price, if it decreases the trading account's position (sell order), the fill price
        // must be greater than or equal to the target price.
-       ctx.isFillPriceValid = (ctx.isBuyOrder && ctx.offchainOrder.targetPrice <= ctx.fillPriceX18.intoUint256())
-            || (!ctx.isBuyOrder && ctx.offchainOrder.targetPrice >= ctx.fillPriceX18.intoUint256());

+       ctx.isFillPriceValid = (ctx.isBuyOrder && ctx.offchainOrder.targetPrice >= ctx.fillPriceX18.intoUint256())
+            || (!ctx.isBuyOrder && ctx.offchainOrder.targetPrice <= ctx.fillPriceX18.intoUint256());

        .
        .
        .
    }


```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. The checks that confirm that the sequencer is up in `ChainlinkUtil::getPrice` function are inefficient            



## Summary

In the previous audit there is an issue related to missing check if the `L2 Sequencer` is down and in the current version of the protocol there are checks in the `ChainlinkUtil::getPrice` function to confirm if the `Sequencer` has the correct status, but this checks are not efficient.
Link: <https://github.com/Cyfrin/2024-07-zaros/blob/d687fe96bb7ace8652778797052a38763fbcbb1b/src/external/chainlink/ChainlinkUtil.sol#L42-L56>

## Vulnerability Details

In the chainlink docs is written that `sequencerUptimeFeed` can return `0` value for `startedAt` if it is called during an `invalid round`:

Link: <https://docs.chain.link/data-feeds/l2-sequencer-feeds>

```Solidity

The sequencerUptimeFeed object returns the following values:

answer: A variable with a value of either 1 or 0
    - 0: The sequencer is up
    - 1: The sequencer is down

startedAt: This timestamp indicates when the sequencer changed status. This timestamp returns 0 if a round is invalid. An invalid round does not indicate sequencer status, so you must decide for your application how best to handle an invalid round. When the sequencer comes back up after an outage, wait for the GRACE_PERIOD_TIME to pass before accepting answers from the data feed. Subtract startedAt from block.timestamp and revert the request if the result is less than the GRACE_PERIOD_TIME.

If the sequencer is up and the GRACE_PERIOD_TIME has passed, the function retrieves the latest answer from the data feed using the dataFeed object.

```

It is important to note that `invalid round` means that there was a problem updating the Sequencer's status, possibly due to network issues or problems with data from oracles, and is shown by a `startedAt` time of `0` and `answer` is `0`. One of the chainlink engineers gave an additional clarification about that and it can be read in the public [chainlink discord channel](https://discord.com/channels/592041321326182401/605768708266131456/1213847312141525002):

> An "invalid round" means there was a problem updating the sequencer's status, possibly due to network issues or problems with data from oracles, and is shown by a startedAt time of 0. Normally, when a round starts, startedAt is recorded, and the initial status (answer) is set to 0. Later, both the answer and the time it was updated (updatedAt) are set at the same time after getting enough data from oracles, making sure that answer only changes from 0 when there's a confirmed update different from the start time. This process helps avoid mistakes in judging if the sequencer is available, which could cause security issues. Making sure startedAt isn't 0 is crucial for keeping the system secure and properly informed about the sequencer's status.

The checks in the `ChainlinkUtil::getPrice` function to ensure that the Sequencer's status is up are the following:

```javascript

    if (address(sequencerUptimeFeed) != address(0)) {
        try sequencerUptimeFeed.latestRoundData() returns (
            uint80, int256 answer, uint256 startedAt, uint256, uint80
        ) {
@>          bool isSequencerUp = answer == 0;
@>          if (!isSequencerUp) {
                revert Errors.OracleSequencerUptimeFeedIsDown(address(sequencerUptimeFeed));
            }

@>           uint256 timeSinceUp = block.timestamp - startedAt;
@>           if (timeSinceUp <= Constants.SEQUENCER_GRACE_PERIOD_TIME) {
                revert Errors.GracePeriodNotOver();
            }
        } catch {
            revert Errors.InvalidSequencerUptimeFeedReturn();
        }
    }

```

But the check for the `timeSinceUp` is inefficient if its called in an `invalid round`. In that case the `startedAt` will be `0` and `block.timestamp - startedAt` (1722184616 - 0 = 1722184616) will result in a value greater than `SEQUENCER_GRACE_PERIOD_TIME` that is `3600` and the code will not revert.

Also, there is already created PR about that `L2 Sequencer Feed` code sample does not match the recommendation from the chainlink docs:
<https://github.com/smartcontractkit/documentation/pull/1995>

## Impact

We can have a scenario where a round begins with `startedAt` set to `0` and `answer`, the initial status, also set to `0`. According to the documentation, if `answer` equals `0`, it indicates that the `sequencer` is `up`, while a value of `1` indicates that the `sequencer` is `down`. However, in this situation, both `answer` and `startedAt` can initially be `0`. These values will only be updated to reflect the correct status of the `sequencer` after all data has been retrieved from oracles and the update is confirmed.

The inefficient checks to confirm the correct status of the `sequencer` leads the `getPrice` function to not revert even when the sequencer uptime feed is not updated or it is called in an invalid round.

## Tools Used

Manual Review

## Recommendations

Add a check to ensure that `startedAt` is not `0`:

```diff

    function getPrice(
        IAggregatorV3 priceFeed,
        uint32 priceFeedHeartbeatSeconds,
        IAggregatorV3 sequencerUptimeFeed
    )
        internal
        view
        returns (UD60x18 price)
    {
        uint8 priceDecimals = priceFeed.decimals();
        // should revert if priceDecimals > 18
        if (priceDecimals > Constants.SYSTEM_DECIMALS) {
            revert Errors.InvalidOracleReturn();
        }

        if (address(sequencerUptimeFeed) != address(0)) {
            try sequencerUptimeFeed.latestRoundData() returns (
                uint80, int256 answer, uint256 startedAt, uint256, uint80
            ) {
                
+               bool validRound = startedAt > 0;
+                if (!validRound){
+                    revert InvalidUptimeFeedRound();
+                }
         
                bool isSequencerUp = answer == 0;
                if (!isSequencerUp) {
                    revert Errors.OracleSequencerUptimeFeedIsDown(address(sequencerUptimeFeed));
                }

                uint256 timeSinceUp = block.timestamp - startedAt;
                if (timeSinceUp <= Constants.SEQUENCER_GRACE_PERIOD_TIME) {
                    revert Errors.GracePeriodNotOver();
                }
            } catch {
                revert Errors.InvalidSequencerUptimeFeedReturn();
            }
        }
    .
    .
    .
    }


```





