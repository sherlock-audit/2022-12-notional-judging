xiaoming90

high

# Scaling factor of the wrapped token is incorrect

## Summary

The scaling factor of the wrapped token within the Boosted3Token leverage vault is incorrect. Thus, all the computations within the leverage vault will be incorrect. This leads to an array of issues such as users being liquidated prematurely or users being able to borrow more than they are allowed to.

## Vulnerability Detail

In Line 120, it calls the `getScalingFactors` function of the LinearPool to fetch the scaling factors of the LinearPool.

In Line 123, it computes the final scaling factor of the wrapped token by multiplying the main token's decimal scaling factor with the wrapped token rate, which is incorrect.

https://github.com/sherlock-audit/2022-12-notional/blob/main/contracts/vaults/balancer/mixins/Boosted3TokenPoolMixin.sol#L109

```solidity
File: Boosted3TokenPoolMixin.sol
109:     function _underlyingPoolContext(ILinearPool underlyingPool) private view returns (UnderlyingPoolContext memory) {
110:         (uint256 lowerTarget, uint256 upperTarget) = underlyingPool.getTargets();
111:         uint256 mainIndex = underlyingPool.getMainIndex();
112:         uint256 wrappedIndex = underlyingPool.getWrappedIndex();
113: 
114:         (
115:             /* address[] memory tokens */,
116:             uint256[] memory underlyingBalances,
117:             /* uint256 lastChangeBlock */
118:         ) = Deployments.BALANCER_VAULT.getPoolTokens(underlyingPool.getPoolId());
119: 
120:         uint256[] memory underlyingScalingFactors = underlyingPool.getScalingFactors();
121:         // The wrapped token's scaling factor is not constant, but increases over time as the wrapped token increases in
122:         // value.
123:         uint256 wrappedScaleFactor = underlyingScalingFactors[mainIndex] * underlyingPool.getWrappedTokenRate() /
124:             BalancerConstants.BALANCER_PRECISION;
125: 
126:         return UnderlyingPoolContext({
127:             mainScaleFactor: underlyingScalingFactors[mainIndex],
128:             mainBalance: underlyingBalances[mainIndex],
129:             wrappedScaleFactor: wrappedScaleFactor,
130:             wrappedBalance: underlyingBalances[wrappedIndex],
131:             virtualSupply: underlyingPool.getVirtualSupply(),
132:             fee: underlyingPool.getSwapFeePercentage(),
133:             lowerTarget: lowerTarget,
134:             upperTarget: upperTarget    
135:         });
136:     }
```

Following is the source code of LinearPool taken from Balancer - https://github.com/balancer-labs/balancer-v2-monorepo/blob/master/pkg/pool-linear/contracts/LinearPool.sol

The correct way of calculating the final scaling factor of the wrapped token is to multiply the wrapped token's decimal scaling factor by the wrapped token rate as shown below:

```solidity
scalingFactors[_wrappedIndex] = _scalingFactorWrappedToken.mulDown(_getWrappedTokenRate());
```

The `_scalingFactorWrappedToken` is the scaling factor that, when multiplied to a token amount, normalizes its balance as if it had 18 decimals. The `_getWrappedTokenRate` function returns the wrapped token rate.

It is important to note that the decimal scaling factor of the main and wrapped tokens are not always the same. Thus, they cannot be used interchangeably.

https://github.com/balancer-labs/balancer-v2-monorepo/blob/c3479e35f01e690fb71b2bf6b38a15cb92128586/pkg/pool-linear/contracts/LinearPool.sol#L504

https://github.com/balancer-labs/balancer-v2-monorepo/blob/c3479e35f01e690fb71b2bf6b38a15cb92128586/pkg/pool-linear/contracts/LinearPool.sol#L521

```solidity
    // Scaling factors

    function _scalingFactor(IERC20 token) internal view virtual returns (uint256) {
        if (token == _mainToken) {
            return _scalingFactorMainToken;
        } else if (token == _wrappedToken) {
            // The wrapped token's scaling factor is not constant, but increases over time as the wrapped token
            // increases in value.
            return _scalingFactorWrappedToken.mulDown(_getWrappedTokenRate());
        } else if (token == this) {
            return FixedPoint.ONE;
        } else {
            _revert(Errors.INVALID_TOKEN);
        }
    }

    /**
     * @notice Return the scaling factors for all tokens, including the BPT.
     */
    function getScalingFactors() public view virtual override returns (uint256[] memory) {
        uint256[] memory scalingFactors = new uint256[](_TOTAL_TOKENS);

        // The wrapped token's scaling factor is not constant, but increases over time as the wrapped token increases in
        // value.
        scalingFactors[_mainIndex] = _scalingFactorMainToken;
        scalingFactors[_wrappedIndex] = _scalingFactorWrappedToken.mulDown(_getWrappedTokenRate());
        scalingFactors[_BPT_INDEX] = FixedPoint.ONE;

        return scalingFactors;
    }
```

## Impact

Within the Boosted 3 leverage vault, the balances are scaled before passing them to the stable math function for computation since the stable math function only works with balances that have been normalized to 18 decimals. If the scaling factor is incorrect, all the computations within the leverage vault will be incorrect, which affects almost all the vault functions.

For instance, the `Boosted3TokenAuraVault.convertStrategyToUnderlying` function relies on the wrapped scaling factor for its computation under the hood. This function is utilized by Notional's `VaultConfiguration.calculateCollateralRatio` function to determine the value of the vault share when computing the collateral ratio. If the underlying result is wrong, the collateral ratio will be wrong too, and this leads to an array of issues such as users being liquidated prematurely or users being able to borrow more than they are allowed to.

## Code Snippet

https://github.com/sherlock-audit/2022-12-notional/blob/main/contracts/vaults/balancer/mixins/Boosted3TokenPoolMixin.sol#L109

## Tool used

Manual Review

## Recommendation

There is no need to manually calculate the final scaling factor of the wrapped token again within the code. This is because the wrapped token scaling factor returned by the `LinearPool.getScalingFactors()` function already includes the token rate. Refer to the Balancer's source code above for referen

```diff
function _underlyingPoolContext(ILinearPool underlyingPool) private view returns (UnderlyingPoolContext memory) {
    (uint256 lowerTarget, uint256 upperTarget) = underlyingPool.getTargets();
    uint256 mainIndex = underlyingPool.getMainIndex();
    uint256 wrappedIndex = underlyingPool.getWrappedIndex();

    (
        /* address[] memory tokens */,
        uint256[] memory underlyingBalances,
        /* uint256 lastChangeBlock */
    ) = Deployments.BALANCER_VAULT.getPoolTokens(underlyingPool.getPoolId());

    uint256[] memory underlyingScalingFactors = underlyingPool.getScalingFactors();
-   // The wrapped token's scaling factor is not constant, but increases over time as the wrapped token increases in
-   // value.
-   uint256 wrappedScaleFactor = underlyingScalingFactors[mainIndex] * underlyingPool.getWrappedTokenRate() /
-       BalancerConstants.BALANCER_PRECISION;

    return UnderlyingPoolContext({
        mainScaleFactor: underlyingScalingFactors[mainIndex],
        mainBalance: underlyingBalances[mainIndex],
-       wrappedScaleFactor: wrappedScaleFactor,
+       wrappedScaleFactor: underlyingScalingFactors[wrappedIndex],        
        wrappedBalance: underlyingBalances[wrappedIndex],
        virtualSupply: underlyingPool.getVirtualSupply(),
        fee: underlyingPool.getSwapFeePercentage(),
        lowerTarget: lowerTarget,
        upperTarget: upperTarget    
    });
}
```