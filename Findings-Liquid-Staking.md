# Liquid Staking 

- ## Medium Risk Findings
    - ### [M-01. Denial of Service in `WithdrawalPool::getBatchIds` and `WithdrawalPool::getFinalizedWithdrawalIdsByOwner` due to `withdrawalBatches` array](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: Stakelink

### Dates: Sep 30th, 2024 - Oct 17th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-09-stakelink)
    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Denial of Service in `WithdrawalPool::getBatchIds` and `WithdrawalPool::getFinalizedWithdrawalIdsByOwner` due to `withdrawalBatches` array            

## Summary

The `withdrawalBatches` array increases its size in `WithdrawalPool::_finalizeWithdrawals` function and this leads to denial of service.

## Vulnerability Details

When a withdraw is finalized in [`WithdrawalPool::_finalizeWithdrawals`](https://github.com/Cyfrin/2024-09-stakelink/blob/f5824f9ad67058b24a2c08494e51ddd7efdbb90b/contracts/core/priorityPool/WithdrawalPool.sol#L453-L455) a new `batchId` is appended to the `withdrawalBatches` array. In that way the `withdrawalBatches` array can become very large. Also, there is no way to remove elements from this array, it increases only. This problem leads to denial of service in functions [`WithdrawalPool::getBatchIds`](https://github.com/Cyfrin/2024-09-stakelink/blob/f5824f9ad67058b24a2c08494e51ddd7efdbb90b/contracts/core/priorityPool/WithdrawalPool.sol#L155-L169) and [`WithdrawalPool::getFinalizedWithdrawalIdsByOwner`](https://github.com/Cyfrin/2024-09-stakelink/blob/f5824f9ad67058b24a2c08494e51ddd7efdbb90b/contracts/core/priorityPool/WithdrawalPool.sol#L213-L253). 
This issue is reported in [Cyfrin - LINK Staking Withdrawals audit (7.3.1)](https://github.com/Cyfrin/2024-09-stakelink/blob/main/audits/%5B2024-09-17%5D%20Cyfrin%20-%20LINK%20Staking%20Withdrawals.pdf) and the proposed solution is to find a cut-off batch id and all batches up to and including this cutoff batch id can be safely ignored. 
The protocol team has implemented  [`WithdrawalPool::updateWithdrawalBatchIdCutoff`](https://github.com/Cyfrin/2024-09-stakelink/blob/f5824f9ad67058b24a2c08494e51ddd7efdbb90b/contracts/core/priorityPool/WithdrawalPool.sol#L370-L398) function that updates the variable `withdrawalIdCutoff` and this variable is correctly used then in `getBatchIds` function. 
The problem is that the function `WithdrawalPool::updateWithdrawalBatchIdCutoff` is not explicitly called in some of the functions in the contract. Therefore, if this function is never called the value of the `withdrawalIdCutoff` will be zero and the function will be useless.

## Impact

The implemented fix is not sufficient to solve the problem and the possibility of denial of service in the `WithdrawalPool::getBatchIds` and `WithdrawalPool::getFinalizedWithdrawalIdsByOwner` functions still persist.

## Tools Used

Manual Review

## Recommendations

Call the function `WithdrawalPool::updateWithdrawalBatchIdCutoff` after the withdraw is finalized to update the withdraw batch id.





