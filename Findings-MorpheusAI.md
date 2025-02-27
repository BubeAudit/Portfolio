# MorpheusAI

# Table of contents

- ## Low Risk Findings
    - ### [L-01. Lack of access control](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: MorpheusAI

### Dates: Jan 30th, 2024 - Feb 3rd, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-01-Morpheus)
 


# Low Risk Findings

## <a id='L-01'></a>L-01. Lack of access control            

### Relevant GitHub Links

https://github.com/Cyfrin/2024-01-Morpheus/blob/07c900d22073911afa23b7fa69a4249ab5b713c8/contracts/mock/tokens/WStETHMock.sol#L15

https://github.com/Cyfrin/2024-01-Morpheus/blob/07c900d22073911afa23b7fa69a4249ab5b713c8/contracts/mock/tokens/StETHMock.sol#L19

https://github.com/Cyfrin/2024-01-Morpheus/blob/07c900d22073911afa23b7fa69a4249ab5b713c8/contracts/mock/tokens/StETHMock.sol#L59

## Summary

There are some functions which have no access control.

## Vulnerability Details

Anyone can call the functions `WStETHMock::mint`, `StETHMOCK::mint` and `StETHMOCK::transferSharesFrom`.

## Impact

Anyone can call the `mint` functions in `WStETHMOCK` and `StETHMOCK` contracts. The `mint` function allows the caller to mint new shares to any account. Although there is a limit on the amount per mint, there is no cap on the total number of mints, which could lead to an excessive increase in supply and potential devaluation of the token.

Also, anyone can call the `StETHMOCK::transferSharesFrom` with arbitrary address for `sender` and `recipient`. 

The mock contracts are in scope and will be used in production, therefore it is crucial to be secured.

## Tools Used

Manual Review

## Recommendations

Add access control to the functions `WStETHMock::mint`, `StETHMOCK::mint` and `StETHMOCK::transferSharesFrom`.


