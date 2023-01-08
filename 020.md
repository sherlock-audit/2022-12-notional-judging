xiaoming90

medium

# Unable to deploy new leverage vault for certain MetaStable Pool

## Summary

Notional might have an issue deploying the new leverage vault for a MetaStable Pool that does not have Balancer Oracle enabled.

## Vulnerability Detail

During deployment, the `MetaStable2TokenVaultMixin` contract checks if the MetaStable Pool's oracle is enabled. However, the Balancer Oracle has been deprecated (https://docs.balancer.fi/products/oracles-deprecated). In addition, Notional is no longer relying on Balancer Oracle in this iteration. Therefore, there is no need to check if the Balancer Oracle is enabled on the MetaStable Pool.

https://github.com/sherlock-audit/2022-12-notional/blob/main/contracts/vaults/balancer/mixins/MetaStable2TokenVaultMixin.sol#L11

```solidity
File: MetaStable2TokenVaultMixin.sol
11: abstract contract MetaStable2TokenVaultMixin is TwoTokenPoolMixin {
12:     constructor(NotionalProxy notional_, AuraVaultDeploymentParams memory params)
13:         TwoTokenPoolMixin(notional_, params)
14:     {
15:         // The oracle is required for the vault to behave properly
16:         (/* */, /* */, /* */, /* */, bool oracleEnabled) = 
17:             IMetaStablePool(address(BALANCER_POOL_TOKEN)).getOracleMiscData();
18:         require(oracleEnabled);
19:     }
```

## Impact

Notional might have an issue deploying the new leverage vault for a MetaStable Pool that does not have Balancer Oracle enabled. Since Balancer Oracle has been deprecated, the Balancer Oracle will likely be disabled on the MetaStable Pool.

## Code Snippet

https://github.com/sherlock-audit/2022-12-notional/blob/main/contracts/vaults/balancer/mixins/MetaStable2TokenVaultMixin.sol#L11

## Tool used

Manual Review

## Recommendation

Remove the Balancer Oracle check from the constructor.

```diff
    constructor(NotionalProxy notional_, AuraVaultDeploymentParams memory params)
        TwoTokenPoolMixin(notional_, params)
    {
-        // The oracle is required for the vault to behave properly
-        (/* */, /* */, /* */, /* */, bool oracleEnabled) = 
-            IMetaStablePool(address(BALANCER_POOL_TOKEN)).getOracleMiscData();
-        require(oracleEnabled);
    }
```