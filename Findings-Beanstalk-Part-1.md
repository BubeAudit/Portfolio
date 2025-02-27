# Beanstalk Part 1

# Table of contents

- ## Medium Risk Findings
    - ### [M-01. `Chainlink` oracle returns stale price due to `CHAINLINK_TIMEOUT` variable in `LibChainlinkOracle` being set to 4 hours](#M-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Beanstalk

### Dates: Feb 26th, 2024 - Mar 25th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-02-Beanstalk-1)

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. `Chainlink` oracle returns stale price due to `CHAINLINK_TIMEOUT` variable in `LibChainlinkOracle` being set to 4 hours            

### Relevant GitHub Links

https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/libraries/Oracle/LibChainlinkOracle.sol#L22

https://github.com/Cyfrin/2024-02-Beanstalk-1/blob/a3658861af8f5126224718af494d02352fbb3ea5/protocol/contracts/libraries/Oracle/LibChainlinkOracle.sol#L138-L149

## Summary

The `LibChainlinkOracle` library utilizes a `CHAINLINK_TIMEOUT` constant set to `14400` seconds (4 hours). This duration is four times longer than the `Chainlink` heartbeat that is `3600` seconds (1 hour), potentially introducing a significant delay in recognizing stale or outdated price data.

## Vulnerability Details

The `LibChainlinkOracle::checkForInvalidTimestampOrAnswer` function accepts three input arguments: `timestamp`, `answer` and `currentTimestamp` and check if the return answer from `Chainlinlink Oracle` or the timestamp is invalid:

```javascript

    function checkForInvalidTimestampOrAnswer(
        uint256 timestamp,
        int256 answer,
        uint256 currentTimestamp
    ) private pure returns (bool) {
        // Check for an invalid timeStamp that is 0, or in the future
        if (timestamp == 0 || timestamp > currentTimestamp) return true;
        // Check if Chainlink's price feed has timed out
        if (currentTimestamp.sub(timestamp) > CHAINLINK_TIMEOUT) return true;
        // Check for non-positive price
        if (answer <= 0) return true;
    }

```

The function also checks if the difference between the `currentTimestamp` and the `timestamp` is greater then `CHAINLINK_TIMEOUT`. The `CHAINLINK_TIMEOUT` is defined to be `4 hours`:

```javascript

 uint256 public constant CHAINLINK_TIMEOUT = 14400; // 4 hours: 60 * 60 * 4

```

## Impact

The `Chainlink` heartbeat indicates the expected frequency of updates from the oracle. The `Chainlink` heartbeat on Ethereum for `Eth/Usd` is `3600` seconds (1 hour).

https://docs.chain.link/data-feeds/price-feeds/addresses?network=ethereum&page=1&search=0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419

But the defined `CHAINLINK_TIMEOUT` in the `LibChainlinkOracle` is `14400` seconds (4 hours).

A `CHAINLINK_TIMEOUT` that is significantly longer than the heartbeat can lead to scenarios where the `LibChainlinkOracle` library accepts data that may no longer reflect current market conditions. Also, in volatile markets, a 4-hour window leads to accepting outdated prices, increasing the risk of price slippage.

## Tools Used

Manual Review

## Recommendations

Consider reducing the `CHAINLINK_TIMEOUT` to align more closely with the `Chainlink` heartbeat on Ethereum, enhancing the relevance of the price data.




