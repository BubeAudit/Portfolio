# Sparkn 
# Table of contents

- ## Low Risk Findings
    - ### [L-01. Check that a contract exists before calling it](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: CodeFox Inc.

### Dates: Aug 21st, 2023 - Aug 29th, 2023

[See more contest details here](https://codehawks.cyfrin.io/c/2023-08-sparkn)
  


# Low Risk Findings

## <a id='L-01'></a>L-01. Check that a contract exists before calling it            

### Relevant GitHub Links

https://github.com/Cyfrin/2023-08-sparkn/blob/0f139b2dc53905700dd29a01451b330f829653e9/src/Proxy.sol#L44-L46

## Summary
In `Proxy.sol` contract is missing the check that the implementation address exists or is the zero address (0x0).

## Vulnerability Details
The `Proxy.sol` contract is based on `OpenZeppelin's Proxy` contract, but it does not check for the contract’s existence prior to returning. As a result, the `Proxy` contract may return success to a failed call and result in incorrect behavior.

Low-level calls, including assembly, lack the protections offered by high-level Solidity calls. In particular, low-level calls will not check that the called account has code. The Solidity documentation warns:

"The low-level call, delegatecall and callcode will return success if the called account is non-existent, as part of the design of EVM. Existence must be checked prior to calling if desired."

## Impact
If the destination of delegatecall has no code, then the call will successfully return. If the proxy is set incorrectly, or if the destination was destroyed, any call to the proxy will succeed but will not send back data. A contract calling the proxy may change its own state under the assumption that its interactions are successful, even though they are not.

## Tools Used
Manual review, VS Code

## Recommendations
Check before the low-level call that the address actually exists. Before the low level call in the `fallback` function in `Proxy.sol` you should check that the input constructor parameter `implementation` is a contract by checking its code size. Also, you should check if the `implementation` address is not the zero address.


