MalfurionWhitehat

medium

# Possible division by zero depending on `TradingModule.getOraclePrice` return values

## Summary

Some functions depending on `TradingModule.getOraclePrice` accept non-negative `(int256 answer, int256 decimals)` return values. In case any of those are equal to zero, division depending on `answer` or `decimals` will revert. In the worst case scenario, this will prevent the protocol from continuing operating.

## Vulnerability Detail

The function `TradingModule.getOraclePrice` properly validates that return values from Chainlink price feeds [are](https://github.com/notional-finance/leveraged-vaults/blob/dd797156651580737ba54506db075b9fdb3d35e9/contracts/trading/TradingModule.sol#L244) [positive](https://github.com/notional-finance/leveraged-vaults/blob/dd797156651580737ba54506db075b9fdb3d35e9/contracts/trading/TradingModule.sol#L248). 

Nevertheless, `answer` may _currently_ return zero, as [it is calculated as](https://github.com/notional-finance/leveraged-vaults/blob/dd797156651580737ba54506db075b9fdb3d35e9/contracts/trading/TradingModule.sol#L250-L252) `(basePrice * quoteDecimals * RATE_DECIMALS) / (quotePrice * baseDecimals);`, which can be truncated down to zero, depending on base/quote prices [1]. Additionally, `decimals` may _in the future_ return zero, depending on changes to the protocol code, as [the NatSpec states that](https://github.com/notional-finance/leveraged-vaults/blob/dd797156651580737ba54506db075b9fdb3d35e9/contracts/trading/TradingModule.sol#L229) this is a `number of decimals in the rate, currently hardcoded to 1e18` [2].

If any of these return values are zero, calculations that use division depending on `TradingModule.getOraclePrice` will revert.

More specifically:

**[1]**

**1.1** [`TradingModule.getLimitAmount`](https://github.com/notional-finance/leveraged-vaults/blob/dd797156651580737ba54506db075b9fdb3d35e9/contracts/trading/TradingModule.sol#L282)

```solidity
        require(oraclePrice >= 0); /// @dev Chainlink rate error
```

that calls `TradingUtils._getLimitAmount`, which reverts if [`oraclePrice` is `0`](https://github.com/notional-finance/leveraged-vaults/blob/dd797156651580737ba54506db075b9fdb3d35e9/contracts/trading/TradingUtils.sol#L207)

```solidity
            oraclePrice = (oracleDecimals * oracleDecimals) / oraclePrice;
```

**[2]**
**2.1** [`TwoTokenPoolUtils._getOraclePairPrice`](https://github.com/notional-finance/leveraged-vaults/blob/dd797156651580737ba54506db075b9fdb3d35e9/contracts/vaults/balancer/internal/pool/TwoTokenPoolUtils.sol#L101-L105)

```solidity
            require(decimals >= 0);

            if (uint256(decimals) != BalancerConstants.BALANCER_PRECISION) {
                rate = (rate * int256(BalancerConstants.BALANCER_PRECISION)) / decimals;
            }
```

**2.2** [`TradingModule.getLimitAmount`](https://github.com/notional-finance/leveraged-vaults/blob/dd797156651580737ba54506db075b9fdb3d35e9/contracts/trading/TradingModule.sol#L283)

```solidity
        require(oracleDecimals >= 0); /// @dev Chainlink decimals error
```

that calls `TradingUtils._getLimitAmount`, which reverts if [`oracleDecimals` is `0`](https://github.com/notional-finance/leveraged-vaults/blob/dd797156651580737ba54506db075b9fdb3d35e9/contracts/trading/TradingUtils.sol#L210-L214)

```solidity
            limitAmount =
                ((oraclePrice + 
                    ((oraclePrice * uint256(slippageLimit)) /
                        Constants.SLIPPAGE_LIMIT_PRECISION)) * amount) / 
                oracleDecimals;
```

**2.3** [`CrossCurrencyfCashVault.convertStrategyToUnderlying`](https://github.com/notional-finance/leveraged-vaults/blob/b7f75fd6946dad9661d2476b7b9c7fe3eb78bcd2/contracts/vaults/CrossCurrencyfCashVault.sol#L174-L182)

```solidity
        return (pvInternal * borrowTokenDecimals * rate) /
            (rateDecimals * int256(Constants.INTERNAL_TOKEN_PRECISION));
```

## Impact

In the worst case, the protocol might stop operating. 

Albeit unlikely that `decimals` is ever zero, since currently this is a hardcoded value, it is possible that `answer` might be zero due to round-down performed by the division in `TradingModule.getOraclePrice`. This can happen if the quote token is much more expensive than the base token. In this case, `TradingModule.getLimitAmount` and depending calls, such as `TradingModule.executeTradeWithDynamicSlippage` might revert.

## Code Snippet

```solidity
        answer =
            (basePrice * quoteDecimals * RATE_DECIMALS) /
            (quotePrice * baseDecimals);
        decimals = RATE_DECIMALS;
```

## Tool used

Manual Review

## Recommendation

Validate that the return values are strictly positive (instead of non-negative) in case depending function calculations may result in division by zero. This can be either done on `TradingModule.getOraclePrice` directly or on the depending functions.

```diff
diff --git a/contracts/trading/TradingModule.sol b/contracts/trading/TradingModule.sol
index bfc8505..70b40f2 100644
--- a/contracts/trading/TradingModule.sol
+++ b/contracts/trading/TradingModule.sol
@@ -251,6 +251,9 @@ contract TradingModule is Initializable, UUPSUpgradeable, ITradingModule {
             (basePrice * quoteDecimals * RATE_DECIMALS) /
             (quotePrice * baseDecimals);
         decimals = RATE_DECIMALS;
+
+        require(answer > 0); /// @dev Chainlink rate error
+        require(decimals > 0); /// @dev Chainlink decimals error
     }
 
     function _hasPermission(uint32 flags, uint32 flagID) private pure returns (bool) {
@@ -279,9 +282,6 @@ contract TradingModule is Initializable, UUPSUpgradeable, ITradingModule {
         // prettier-ignore
         (int256 oraclePrice, int256 oracleDecimals) = getOraclePrice(sellToken, buyToken);
 
-        require(oraclePrice >= 0); /// @dev Chainlink rate error
-        require(oracleDecimals >= 0); /// @dev Chainlink decimals error
-
         limitAmount = TradingUtils._getLimitAmount({
             tradeType: tradeType,
             sellToken: sellToken,
diff --git a/contracts/vaults/balancer/internal/pool/TwoTokenPoolUtils.sol b/contracts/vaults/balancer/internal/pool/TwoTokenPoolUtils.sol
index 4954c59..6315c0a 100644
--- a/contracts/vaults/balancer/internal/pool/TwoTokenPoolUtils.sol
+++ b/contracts/vaults/balancer/internal/pool/TwoTokenPoolUtils.sol
@@ -76,10 +76,7 @@ library TwoTokenPoolUtils {
         (int256 rate, int256 decimals) = tradingModule.getOraclePrice(
             poolContext.primaryToken, poolContext.secondaryToken
         );
-        require(rate > 0);
-        require(decimals >= 0);
 
         if (uint256(decimals) != BalancerConstants.BALANCER_PRECISION) {
             rate = (rate * int256(BalancerConstants.BALANCER_PRECISION)) / decimals;
         }

```



