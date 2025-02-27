# Sablier

# Table of contents
- ## Medium Risk Findings
    - ### [M-01. Potential reorg attack in `SablierV2MerkleLockupFactory`](#M-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Sablier

### Dates: May 10th, 2024 - May 31st, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-05-Sablier)
    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Potential reorg attack in `SablierV2MerkleLockupFactory`            

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-Sablier/blob/43d7e752a68bba2a1d73d3d6466c3059079ed0c6/v2-periphery/src/SablierV2MerkleLockupFactory.sol#L36

https://github.com/Cyfrin/2024-05-Sablier/blob/43d7e752a68bba2a1d73d3d6466c3059079ed0c6/v2-periphery/src/SablierV2MerkleLockupFactory.sol#L71

## Summary

The `SablierV2MerkleLockupFactory` contract uses the `create` method in the `createMerkleLL` and `createMerkleLT` functions. While the deployment of a new `SablierV2MerkleLL` and `SablierV2MerkleLT` contracts using `create` is generally safe, there is a potential risks associated with blockchain reorgs.

## Vulnerability Details

The `SablierV2MekleLockupFactory::createMerkleLL` and `SablierV2MekleLockupFactory::createMerkleLT` functions deploy a new `SablierV2MerkleLL` and `SablierV2MerkleLT` contracts using the `create` method, where the address derivation depends only on the arguments passed.

```javascript

    function createMerkleLL(
        MerkleLockup.ConstructorParams memory baseParams,
        ISablierV2LockupLinear lockupLinear,
        LockupLinear.Durations memory streamDurations,
        uint256 aggregateAmount,
        uint256 recipientCount
    )
        external
        returns (ISablierV2MerkleLL merkleLL)
    {
        // Deploy the MerkleLockup contract with CREATE.
@>      merkleLL = new SablierV2MerkleLL(baseParams, lockupLinear, streamDurations);

        // Log the creation of the MerkleLockup contract, including some metadata that is not stored on-chain.
        emit CreateMerkleLL(merkleLL, baseParams, lockupLinear, streamDurations, aggregateAmount, recipientCount);
    }

```

```javascript

    function createMerkleLT(
        MerkleLockup.ConstructorParams memory baseParams,
        ISablierV2LockupTranched lockupTranched,
        MerkleLT.TrancheWithPercentage[] memory tranchesWithPercentages,
        uint256 aggregateAmount,
        uint256 recipientCount
    )
        external
        returns (ISablierV2MerkleLT merkleLT)
    {
        // Calculate the sum of percentages and durations across all tranches.
        uint64 totalPercentage;
        uint256 totalDuration;
        for (uint256 i = 0; i < tranchesWithPercentages.length; ++i) {
            uint64 percentage = tranchesWithPercentages[i].unlockPercentage.unwrap();
            totalPercentage = totalPercentage + percentage;
            unchecked {
                // Safe to use `unchecked` because its only used in the event.
                totalDuration += tranchesWithPercentages[i].duration;
            }
        }

        // Check: the sum of percentages equals 100%.
        if (totalPercentage != uUNIT) {
            revert Errors.SablierV2MerkleLockupFactory_TotalPercentageNotOneHundred(totalPercentage);
        }

        // Deploy the MerkleLockup contract with CREATE.
@>      merkleLT = new SablierV2MerkleLT(baseParams, lockupTranched, tranchesWithPercentages);

        // Log the creation of the MerkleLockup contract, including some metadata that is not stored on-chain.
        emit CreateMerkleLT(
            merkleLT,
            baseParams,
            lockupTranched,
            tranchesWithPercentages,
            totalDuration,
            aggregateAmount,
            recipientCount
        );
    }

```

The problem is that some of the chains like Arbitrum and Polygon are suspicious of the reorg attack. Polygon reorg reference: 
https://protos.com/polygon-hit-by-157-block-reorg-despite-hard-fork-to-reduce-reorgs/ 
This one happened in February, 2023.

A blockchain reorg occurs when nodes in the network receive blocks that create a longer chain than the current one, causing the network to discard the shorter chain. This can lead to a temporary state where transactions that were considered confirmed are reverted.

## Impact

If a reorg occurs after the deployment transaction is included in a block but before it is deeply confirmed, the transaction might be reverted. This could lead to a scenario where the contract is considered deployed and events are emitted, but the contract does not exist after the reorg. This leads to inconsistencies for the transactions that assume its existence.
Also, claims validated and processed before a reorg might be invalidated if the transactions are rolled back. And `streams` created via `LOCKUP_LINEAR.createWithDurations` and/or `LOCKUP_TRANCHED.createWithDurations` will be rolled back. This means the recipient will not receive the funds, and the `streamId` would not exist anymore.

## Tools Used
Manual Review

## Recommendations

Use `create2` method with `salt` that includes `msg.sender` to deploy the `SablierV2MerkleLL` and `SablierV2MerkleLT` contracts.




