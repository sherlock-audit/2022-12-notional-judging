hansfriese

medium

# DOS: `LinearMath._toNominal()` might revert for uint underflow when it should work



## Summary
`LinearMath._toNominal()` calculates the `nominal` amount from `real` and it might revert for uint underflow for some cases.

## Vulnerability Detail
`LinearMath._toNominal()` calculates the `nominal` by substracting  `fees` from `real` like below.

```solidity
    function _toNominal(uint256 real, Params memory params) private pure returns (uint256) {
        // Fees are always rounded down: either direction would work but we need to be consistent, and round down
        // uses less gas.

        if (real < params.lowerTarget) {
            uint256 fees = (params.lowerTarget - real).mulDown(params.fee);
            return real.sub(fees); //@audit possible revert
        } else if (real <= params.upperTarget) {
            return real;
        } else {
            uint256 fees = (real - params.upperTarget).mulDown(params.fee);
            return real.sub(fees);
        }
    }
```

But it might revert for some edge cases when `real < params.lowerTarget`.

1. Let's assume `params.fee = 10%, params.lowerTarget = 1000, real = 90`.
2. Then `fees = (1000 - 90) * 10% = 91`.
3. So the final result will be `90 - 91 = -1` and it will revert for uint underflow.

`_toNominal()` is used in several functions and it should work for any possible cases without reverts.

## Impact
`LinearMath._toNominal()` might revert for some cases and several functions using this function wouldn't work properly.

## Code Snippet
https://github.com/sherlock-audit/2022-12-notional/blob/main/contracts/vaults/balancer/internal/math/LinearMath.sol#L61-L63

## Tool used
Manual Review

## Recommendation
I think we can modify like the below to return 0 for the revert case.

```solidity
    function _toNominal(uint256 real, Params memory params) private pure returns (uint256) {
        // Fees are always rounded down: either direction would work but we need to be consistent, and rounding down
        // uses less gas.

        if (real < params.lowerTarget) {
            uint256 fees = (params.lowerTarget - real).mulDown(params.fee);
            if (real < fees) //++++++++++++++++++
                return 0;
            return real.sub(fees);
        } else if (real <= params.upperTarget) {
            return real;
        } else {
            uint256 fees = (real - params.upperTarget).mulDown(params.fee);
            return real.sub(fees);
        }
    }
```
