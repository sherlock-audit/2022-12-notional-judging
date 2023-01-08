xiaoming90

high

# `totalBPTSupply` will be excessively inflated

## Summary

The `totalBPTSupply` will be excessively inflated as `totalSupply` was used instead of `virtualSupply`. This might cause a boosted balancer leverage vault not to be emergency settled in a timely manner and holds too large of a share of the liquidity within the pool, thus having problems exiting its position.

## Vulnerability Detail

Balancer's Boosted Pool uses Phantom BPT where all pool tokens are minted at the time of pool creation and are held by the pool itself. Therefore, `virtualSupply` should be used instead of `totalSupply` to determine the amount of BPT supply in circulation.

However, within the `Boosted3TokenAuraVault.getEmergencySettlementBPTAmount` function, the `totalBPTSupply` at Line 169 is derived from the `totalSupply` instead of the `virtualSupply`.  As a result, `totalBPTSupply` will be excessively inflated (`2**(111)`).

https://github.com/sherlock-audit/2022-12-notional/blob/main/contracts/vaults/Boosted3TokenAuraVault.sol#L169

```solidity
File: Boosted3TokenAuraVault.sol
165:     function getEmergencySettlementBPTAmount(uint256 maturity) external view returns (uint256 bptToSettle) {
166:         Boosted3TokenAuraStrategyContext memory context = _strategyContext();
167:         bptToSettle = context.baseStrategy._getEmergencySettlementParams({
168:             maturity: maturity, 
169:             totalBPTSupply: IERC20(context.poolContext.basePool.basePool.pool).totalSupply()
170:         });
171:     }
```

As a result, the `emergencyBPTWithdrawThreshold` threshold will be extremely high. As such, the condition at Line 97 will always be evaluated as true and result in a revert.

https://github.com/sherlock-audit/2022-12-notional/blob/main/contracts/vaults/balancer/internal/settlement/SettlementUtils.sol#L86

```solidity
File: SettlementUtils.sol
86:     function _getEmergencySettlementParams(
87:         StrategyContext memory strategyContext,
88:         uint256 maturity,
89:         uint256 totalBPTSupply
90:     )  internal view returns(uint256 bptToSettle) {
91:         StrategyVaultSettings memory settings = strategyContext.vaultSettings;
92:         StrategyVaultState memory state = strategyContext.vaultState;
93: 
94:         // Not in settlement window, check if BPT held is greater than maxBalancerPoolShare * total BPT supply
95:         uint256 emergencyBPTWithdrawThreshold = settings._bptThreshold(totalBPTSupply);
96: 
97:         if (strategyContext.vaultState.totalBPTHeld <= emergencyBPTWithdrawThreshold)
98:             revert Errors.InvalidEmergencySettlement();
```

https://github.com/sherlock-audit/2022-12-notional/blob/main/contracts/vaults/balancer/internal/BalancerVaultStorage.sol#L52

```solidity
File: BalancerVaultStorage.sol
52:     function _bptThreshold(StrategyVaultSettings memory strategyVaultSettings, uint256 totalBPTSupply)
53:         internal pure returns (uint256) {
54:         return (totalBPTSupply * strategyVaultSettings.maxBalancerPoolShare) / BalancerConstants.VAULT_PERCENT_BASIS;
55:     }
```

## Impact

Anyone (e.g. off-chain keeper or bot) that relies on the `SettlementUtils.getEmergencySettlementBPTAmount` to determine if an emergency settlement is needed would be affected. The caller will presume that since the function reverts, emergency settlement is not required and the BPT threshold is still within the healthy level. The caller will wrongly decided not to perform an emergency settlement on a vault that has already exceeded the BPT threshold.

If a boosted balancer leverage vault is not emergency settled in a timely manner and holds too large of a share of the liquidity within the pool, it will have problems exiting its position.

## Code Snippet

https://github.com/sherlock-audit/2022-12-notional/blob/main/contracts/vaults/Boosted3TokenAuraVault.sol#L169

## Tool used

Manual Review

## Recommendation

Update the function to compute the `totalBPTSupply` from the virtual supply.

```diff
    function getEmergencySettlementBPTAmount(uint256 maturity) external view returns (uint256 bptToSettle) {
        Boosted3TokenAuraStrategyContext memory context = _strategyContext();
        bptToSettle = context.baseStrategy._getEmergencySettlementParams({
            maturity: maturity, 
-           totalBPTSupply: IERC20(context.poolContext.basePool.basePool.pool).totalSupply()
+			totalBPTSupply: context.poolContext._getVirtualSupply(context.oracleContext)
        });
    }
```