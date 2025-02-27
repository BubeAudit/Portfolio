# DittoETH

# Table of contents

- ## Low Risk Findings
    - ### [L-01. Risk of division by zero](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Ditto

### Dates: Sep 8th, 2023 - Oct 9th, 2023

[See more contest details here](https://codehawks.cyfrin.io/c/2023-09-ditto)
 


# Low Risk Findings

## <a id='L-01'></a>L-01. Risk of division by zero            

### Relevant GitHub Links

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/MarketShutdownFacet.sol#L99

## Summary
There is a potential risk of division by zero in `_getAssetCollateralRatio()` function in smart contract
`MarketShutdownFacet.sol`.

## Vulnerability Details
In Solidity, a division by zero error will cause a transaction to fail and revert. This means that all changes made during the transaction will be rolled back and no state changes will be made to the blockchain. The entire transaction will be considered invalid and the gas used by the transaction will not be refunded.

## Impact
In the contract `MarketShutdownFacet.sol` in function `_getAssetCollateralRatio()` there is possibility for division by zero. That can occur if the function `getPrice()` returns zero for some reasons. If a division by zero error occurs, it means that the function was unable to calculate the collateral ratio for the given asset. This could prevent other functions that rely on `_getAssetCollateralRatio()` function from working correctly.

For example, the `shutdownMarket()` and `redeemErc()` functions both use `_getAssetCollateralRatio()` to determine the collateral ratio of an asset. If a division by zero error occurs in `_getAssetCollateralRatio()`, these functions will also fail and revert.

This could potentially lock up user funds or disrupt the normal operation of the contract. Therefore, it's important to handle the division by zero case properly to prevent such issues.

## Tools Used
Manual review, VS Code

## Recommendations
Add a condition to check if the price returned by `getPrice()` is zero before performing the division. If the price is zero, you can return zero or revert the transaction.

```
function _getAssetCollateralRatio(address asset)
    private
    view
    returns (uint256 cRatio)
{
    STypes.Asset storage Asset = s.asset[asset];
    uint256 price = LibOracle.getPrice(asset);
    if (price == 0) {
        revert Errors.ZeroPrice(); // or return 0;
    }
    return Asset.zethCollateral.div(price.mul(Asset.ercDebt));
}
```

In this modified version of the function, if `getPrice()` returns zero, the function reverts with an error message ZeroPrice. You can replace `Errors.ZeroPrice()` with your own error message or return zero instead of reverting the transaction. Please note that you need to define `Errors.ZeroPrice()` if you choose to use it.


