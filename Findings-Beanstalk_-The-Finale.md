# Beanstalk: The Finale 
# Table of contents
- ## Medium Risk Findings
    - ### [M-01. The used `CHAIN_ID` and hardcoded addresses for `chainlinkRegistry` and `WETH` are incorrect on chains different from Ethereum mainnet](#M-01)
- ## Low Risk Findings
    - ### [L-01. `WETH` token doesn't have `permit` function](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Beanstalk

### Dates: May 30th, 2024 - Jul 8th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-05-beanstalk-the-finale)
    
# Medium Risk Findings

## <a id='M-01'></a>M-01. The used `CHAIN_ID` and hardcoded addresses for `chainlinkRegistry` and `WETH` are incorrect on chains different from Ethereum mainnet            



## Summary

The library [`LibUsdOracle`](https://github.com/Cyfrin/2024-05-beanstalk-the-finale/blob/4e0ad0b964f74a1b4880114f4dd5b339bc69cd3e/protocol/contracts/libraries/Oracle/LibUsdOracle.sol#L36) relies on the Chainlink price feed registry with hardcoded address, but this address is specific to the Ethereum mainnet. However, the protocol is intended to be deployed on other EVM-equivalent Layer 2 chains, which don't support this registry address. Also, the protocol uses a `WETH` address that is for Ethereum mainnet. The functionality of the protocol that relies on this address will be broken on the other chains.

## Vulnerability Details

The `chainlinkRegistry` is used in the `LibUsdOracle::getTokenPriceFromExternal` function to obtain the address of the Chainlink price feed aggregator for a specific token pair (e.g., token/USD), if the target address in the `oracleImpl` is not set. This is achieved through the `getFeed` function, which takes the base token address and the quote token address (USD) as parameters.

```javascript

 function getTokenPriceFromExternal(
        address token,
        uint256 lookback
    ) internal view returns (uint256 tokenPrice) {
        AppStorage storage s = LibAppStorage.diamondStorage();
        Implementation memory oracleImpl = s.sys.oracleImplementation[token];

        // If the encode type is type 1, use the default chainlink implementation instead.
        // `target` refers to the address of the price aggergator implmenation
        if (oracleImpl.encodeType == bytes1(0x01)) {
            // if the address in the oracle implementation is 0, use the chainlink registry to lookup address
            address chainlinkOraclePriceAddress = oracleImpl.target;
            if (chainlinkOraclePriceAddress == address(0)) {
                // use the chainlink registry
@>              chainlinkOraclePriceAddress = ChainlinkPriceFeedRegistry(chainlinkRegistry).getFeed(
                        token,
                        0x0000000000000000000000000000000000000348
                    ); // 0x0348 is the address for USD
            }

            return
                uint256(1e24).div(
                    LibChainlinkOracle.getTokenPrice(
                        chainlinkOraclePriceAddress,
                        LibChainlinkOracle.FOUR_HOUR_TIMEOUT,
                        lookback
                    )
                );
        } else if (oracleImpl.encodeType == bytes1(0x02)) {
            // assumes a dollar stablecoin is passed in
            // if the encodeType is type 2, use a uniswap oracle implementation.
            address chainlinkToken = IUniswapV3PoolImmutables(oracleImpl.target).token0();
            chainlinkToken = chainlinkToken == token
                ? IUniswapV3PoolImmutables(oracleImpl.target).token1()
                : token;
            tokenPrice = LibUniswapOracle.getTwap(
                lookback == 0 ? LibUniswapOracle.FIFTEEN_MINUTES : uint32(lookback),
                oracleImpl.target,
                chainlinkToken,
                token,
                uint128(10) ** uint128(IERC20Decimals(token).decimals())
            );
            // USDC/USD
            // call chainlink oracle from the OracleImplmentation contract
            Implementation memory chainlinkOracleImpl = s.sys.oracleImplementation[chainlinkToken];
            address chainlinkOraclePriceAddress = chainlinkOracleImpl.target;

            if (chainlinkOraclePriceAddress == address(0)) {
                // use the chainlink registry
@>              chainlinkOraclePriceAddress = ChainlinkPriceFeedRegistry(chainlinkRegistry).getFeed(
                        chainlinkToken,
                        0x0000000000000000000000000000000000000348
                    ); // 0x0348 is the address for USD
            }

            uint256 chainlinkTokenPrice = LibChainlinkOracle.getTokenPrice(
                chainlinkOraclePriceAddress,
                LibChainlinkOracle.FOUR_HOUR_TIMEOUT,
                lookback
            );
            return tokenPrice.mul(chainlinkTokenPrice).div(1e6);
        }

       ...
    }

```

The problem is that the Chainlink registry address is harcoded in the contract:

```javascript
address constant chainlinkRegistry = 0x47Fb2585D2C56Fe188D0E6ec628a38b74fCeeeDf;
```

This hardcoded address is the address of the `chainlinkRegistry` on the Ethereum mainnet. In the `README` is written that the protocol will be deployed also on EVM-equivalent L2 chains. But deploying the protocol on other chains without modifying this address will result in failed oracle lookups and incorrect price feeds. Additionally, according to the chainlink [documentation](https://docs.chain.link/data-feeds/feed-registry#contract-addresses) the `chainlinkRegistry` for now is available only on Ethereum mainnet.

Also, the protocol uses the `WETH` address, that is hardcoded in some contracts (`LibWeth.sol` and `C.sol` (out of scope, but the impact is in the scope))
Link: <https://github.com/Cyfrin/2024-05-beanstalk-the-finale/blob/4e0ad0b964f74a1b4880114f4dd5b339bc69cd3e/protocol/contracts/libraries/Token/LibWeth.sol#L16>

In `LibWeth.sol` library:

```javascript
address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
```

The problem is that the `WETH` token has different addresses on the different chains. The hardcoded address is the `WETH` address on the Ethereum mainnet. But if the protocol is deployed on the other EVM-equivalent L2 chain the address will be different. For example, in [Arbitrum](https://arbiscan.io/token/0x82af49447d8a07e3bd95bd0d56f35241523fbab1) the address of the `WETH` token is: `0x82af49447d8a07e3bd95bd0d56f35241523fbab1`.

There is also one more important hardcoded variable that wouldn't be correct when the protocol is deployed on L2 chain that is the `CHAIN_ID` in the `C.sol` contract. The `CHAIN_ID` is retrived by the function `C.hetChainId` in `LibTractor`, `LibSiloPermit` and `LibTokenPermit` contracts in order to build the domain separator. The hardcoded `CHAIN_ID` constant specified in `C.sol` is not recommended also due to possibility of replaying of all signed permits from Ethereum mainnet on the forked chain, but this is a known issue from the Cyfrin audit.

## Impact

The `LibUsdOracle::getTokenPriceFromExternal` is called in the `OracleFacet::getTokenPriceFromExternal` to retrive the token price. If the Chainlink registry is not available on the deployed chain the protocol will be unable to retrive the price feed.

If the `LibWeth` is deployed on the EVM-equivalent L2 chains the library will be useless, because its functionality relies on the hardcoded `WETH` address that is not correct for the other chain different from Ethereum mainnet.

The `CHAIN_ID` in the `C.sol` is defined as `1` and this is correct for the Ethereum mainnet, but for example the Arbitrium's chainId is different (42161). The transaction will be considered invalid by the network, and nodes will reject it. This is because the `chainId` is used to prevent replay attacks, ensuring that the transaction is meant for the specific blockchain.

## Tools Used

Manual Review

## Recommendations

Don't rely on the Chainlink registry on chain different from Ethereum mainnet and check all hardcoded addresses and varibales if they are correct for the choosen L2 chain before deploying on it. Also, you can add functionality the owner to modify these addresses.


# Low Risk Findings

## <a id='L-01'></a>L-01. `WETH` token doesn't have `permit` function            



## Summary

The [`TokenSupportFacet::permitERC20`](https://github.com/Cyfrin/2024-05-beanstalk-the-finale/blob/4e0ad0b964f74a1b4880114f4dd5b339bc69cd3e/protocol/contracts/beanstalk/farm/TokenSupportFacet.sol#L30-L41) function is designed to support `ERC-20` tokens that implement the `IERC20Permit` interface. However, the `Wrapped Ether (WETH)` token does not implement the `permit` function. This could allow a malicious user to call the function without a valid signature, leading to potential security vulnerabilities.

## Vulnerability Details

The `TokenSupportFacet::permitERC20` function calls the `ERC20::permit` function and the goal of the function is to support the contracts that inherit `ERC20Permit` contract.

```javascript

    function permitERC20(
        IERC20Permit token,
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) public payable fundsSafu noNetFlow noSupplyChange {
@>      token.permit(owner, spender, value, deadline, v, r, s);
    }

```

The problem is that not all ERC20 tokens have `permit` function in their token contracts. The [`WETH`](https://etherscan.io/token/0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2#code) token is one of them and this token is used from the protocol. If a token like `WETH`, which does not implement `permit`, is passed, the function will not revert, potentially allowing unauthorized approvals:

```Solidity
Most ERC20 have the permit function to approve a spender if a valid signature is provided.

However WETH does not. Surprisingly, when permit is called on WETH, the function call will execute without any errors.

This is because the fallback inside WETH is execute when permit is called.
```

Link: <https://solidity-by-example.org/hacks/weth-permit/>

## Impact

Malicious users could call the `permitERC20` function with an empry signature and `WETH` token that does not implement permit function, bypassing signature verification and gaining approval to transfer tokens from the victim's account without their consent. This leads to loss of tokens for the victim.

## Tools Used

Manual Review

## Recommendations

Implement a check to ensure the token contract supports the permit function before calling it.



