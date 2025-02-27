# The Standard

# Table of contents

- ## Medium Risk Findings
    - ### [M-01. Insufficient Slippage Protection in `SmartVaultV3::swap` function](#M-01)
- ## Low Risk Findings
    - ### [L-01. Potential zero swap fee in `SmartVaultV3::swap` function ](#L-01)
    - ### [L-02. Stakers receive reduced rewards due to rounding issue in `LiquidationPool::distributeAssets` function](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: The Standard

### Dates: Dec 27th, 2023 - Jan 10th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2023-12-the-standard)
    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Insufficient Slippage Protection in `SmartVaultV3::swap` function            

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L223

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L226

## Summary

The `SmartVaultV3::swap` function is designed to facilitate token exchanges using an external swap router. The function parameters include a `deadline` set to `block.timestamp` and a `sqrtPriceLimitX96` set to `0`. These parameters raise concerns regarding the potential for failed transactions and unbounded slippage, which could negatively impact the reliability of the contract and the user's funds.

## Vulnerability Details

The `SmartVaultV3::swap` function has parameters `deadline` and `sqrtPriceLimitX96` respectively set to `block.timestamp` and `0`:

```javascript

    function swap(bytes32 _inToken, bytes32 _outToken, uint256 _amount) external onlyOwner {
        uint256 swapFee = _amount * ISmartVaultManagerV3(manager).swapFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
        address inToken = getSwapAddressFor(_inToken);
        uint256 minimumAmountOut = calculateMinimumAmountOut(_inToken, _outToken, _amount);
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
                tokenIn: inToken,
                tokenOut: getSwapAddressFor(_outToken),
                fee: 3000,
                recipient: address(this),
@>              deadline: block.timestamp,
                amountIn: _amount - swapFee,
                amountOutMinimum: minimumAmountOut,
@>              sqrtPriceLimitX96: 0
            });
        inToken == ISmartVaultManagerV3(manager).weth() ?
            executeNativeSwapAndFee(params, swapFee) :
            executeERC20SwapAndFee(params, swapFee);
    }
```

The `deadline` parameter is set to the current block timestamp. This means the transaction must be executed immediately, leaving no time buffer for the transaction to be mined. In a congested network, this could lead to transaction failures, as miners may not include the transaction in the current block.

The `sqrtPriceLimitX96` parameter is set to 0, indicating no limit on the execution price for the swap. This exposes the user to high slippage in volatile market conditions, potentially resulting in a swap execution at an unfavorable rate.

## Impact

Setting the `sqrtPriceLimitX96` parameter to `0` in `SmartVaultV3::swap` function means that there is no limit on the price for the swap. The swap will execute at the best available price between the specified input and output tokens, regardless of how the price may have moved. This could be risky in volatile markets, as the swap could execute at an unfavorable rate if the market price moves significantly between the time the transaction is sent and when it is mined.

Setting the `sqrtPriceLimitX96` parameter to `0`, makes the parameter useless. In production this parameter should not be `0`. In the `Uniswap V3` documentation for the example in the docs is said: 

"sqrtPriceLimitX96: We set this to zero - which makes this parameter inactive. In production, this value can be used to set the limit for the price the swap will push the pool to, which can help protect against price impact or for setting up logic in a variety of price-relevant mechanisms."

Link to the documentation: https://docs.uniswap.org/contracts/v3/guides/swaps/single-swaps

Also, the impact of the setting `deadline` parameter to `block.timestamp` is that in some cases, transactions might be delayed due to network congestion or other reasons, causing them to miss their deadline. This could lead to the transaction failing unnecessarily.

## Tools Used

Manual Review

## Recommendations

Calculate an appropriate value for the `sqrtPriceLimitX96` parameter based on the acceptable slippage percentage and set it accordingly to provide slippage protection.

Also, it is recommended to set the `deadline` parameter to a future timestamp, providing a time window during which the transaction can be mined. This window should be long enough to account for network congestion and potential delays but not too long as to expose the transaction to unnecessary market risk. A common practice is to add a buffer of several minutes (e.g., 15 minutes) to the current block timestamp:

```javascript

deadline: block.timestamp + 15 minutes

```

# Low Risk Findings

## <a id='L-01'></a>L-01. Potential zero swap fee in `SmartVaultV3::swap` function             

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L215

## Summary

The `SmartVaultV3::swap` function calculates the `swapFee` based on the input `_amount` parameter, the `swapFeeRate` and `HUNDRED_PC` obtained from the `ISmartVaultManagerV3(manager)`. If the `_amount` is a small, the `swapFee` can be `0` due to the rounding down by division.

## Vulnerability Details

In the `deploy.js` script the `swapFeeRate` is set to 500. The `HUNDRED_PC` variable is a constant and it is set to `1e5`. Therefore, if the `_amount` parameter is a small (lower than 200), the value of `swapFee` will be `0`:

```javascript

    function swap(bytes32 _inToken, bytes32 _outToken, uint256 _amount) external onlyOwner {
@>      uint256 swapFee = _amount * ISmartVaultManagerV3(manager).swapFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
        address inToken = getSwapAddressFor(_inToken);
        uint256 minimumAmountOut = calculateMinimumAmountOut(_inToken, _outToken, _amount);
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
                tokenIn: inToken,
                tokenOut: getSwapAddressFor(_outToken),
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: _amount - swapFee,
                amountOutMinimum: minimumAmountOut,
                sqrtPriceLimitX96: 0
            });
        inToken == ISmartVaultManagerV3(manager).weth() ?
            executeNativeSwapAndFee(params, swapFee) :
            executeERC20SwapAndFee(params, swapFee);
    }

```
Let's consider the value of `swapFee` if the `_amount` is `20`:

```javascript
uint256 swapFee = 20 * 500 / 1e5
```
In that case the value of `swapFee` will be `0` due to the rounding down by division. 

## Impact

If the `_amount` parameter in `SmartVaultV3::swap` function is lower than `200`, the calculated amount for `swapFee` variable will be `0`. That allows the users to swap small amounts without fees. That can be issue for the protocol, because the protocol will not receive fees for small values of `_amount`.

Additionally, if the `SmartVaultV3::swap` function calls the `SmartVaultV3::executeERC20SwapAndFee` function with `swapFee` parameter equals to `0` and the protocol will use in the future weird `ERC20` tokens which revert on zero value transfer (e.g. `LEND`), the entire transaction will fail, including the `swap` operation.

```javascript

   function executeERC20SwapAndFee(ISwapRouter.ExactInputSingleParams memory _params, uint256 _swapFee) private {
@>      IERC20(_params.tokenIn).safeTransfer(ISmartVaultManagerV3(manager).protocol(), _swapFee);
        IERC20(_params.tokenIn).safeApprove(ISmartVaultManagerV3(manager).swapRouter2(), _params.amountIn);
        ISwapRouter(ISmartVaultManagerV3(manager).swapRouter2()).exactInputSingle(_params);
        IWETH weth = IWETH(ISmartVaultManagerV3(manager).weth());
        // convert potentially received weth to eth
        uint256 wethBalance = weth.balanceOf(address(this));
        if (wethBalance > 0) weth.withdraw(wethBalance);

```

## Tools Used

Manual Review

## Recommendations

Implement a minimum fee threshold to prevent the fee from being zero.
Add validation checks to ensure that the calculated `swapFee` is greater than zero before proceeding with the swap.
While the risk of financial loss due to a zero `swapFee` is low, it is important to address this issue to ensure that the protocol's fee mechanisms are enforced as intended.

## <a id='L-02'></a>L-02. Stakers receive reduced rewards due to rounding issue in `LiquidationPool::distributeAssets` function            

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L219

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L223

## Summary

The `LiquidationPool` contract is responsible for managing staking positions and distributing rewards based on the staked assets. The `LiquidationPool::distributeAssets` function is a key component that allocates rewards to stakers proportionally to their stake in the pool. But stakers may receive less than their fair share of rewards, potentially due to rounding errors inherent in the Solidity programming language's handling of integer division.

## Vulnerability Details

Solidity performs integer division, which truncates the decimal part of the quotient, potentially leading to a loss of precision in calculations. In the `LiquidationPool::distributeAssets` the `_portion` is calculated as follows:

```javascript
  uint256 _portion = asset.amount * _positionStake / stakeTotal;
```

But this calculation can result in truncated values, causing some stakers to receive slightly fewer rewards.

## Impact

Let's consider the following scenario:
The amount of a given asset is 100. Thera are 3 holders and their positions are respectively: 1, 2 and 3.
The `_portion` amount for the first holder will be: `100 * 1 / 6` which is `16,666666667` but in Solidity is rounding down to `16`.
The `_portion` amount for the second holder will be: `100 * 2 / 6` which is `33,333333333` but in Solidity is rounding down to `33`.
The `_portion` amount for the third holder will be: `100 * 3 / 6` which is `50`. 

The first two holders will receive less than their fair share of rewards. 

Also, the `LiquidationPool::distributeAssets` function does not appear to handle any remainder that might be left after all rewards have been distributed.

Additionally, if `costInEuros` is greater than `_position.EUROs` the `_portion` will be calculated as the following:

```javascript
    if (costInEuros > _position.EUROs) {
@>          _portion = _portion * _position.EUROs / costInEuros;
            costInEuros = _position.EUROs;
        }

```

In that case multiplication of the result of division will be executed. Also, if `costInEuros` is greater than `_portion * position.EUROs`, `_portion` will be `0`.

## Tools Used

Manual Review

## Recommendations

You can track the amount of the distributed assets and the last holder can receive the remaining undistributed assets.
Or you can track the remainder of rewards that are not distributed due to rounding and this remainder can be added to the reward pool for the next distribution cycle.



