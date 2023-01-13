hansfriese

medium

# `call()` should be used instead of `transfer()` on an address payable

## Summary
`call()` should be used instead of `transfer()` on an address payable

## Vulnerability Detail
The transfer() and send() functions forward a fixed amount of 2300 gas. Historically, it has often been recommended to use these functions for value transfers to guard against reentrancy attacks. However, the gas cost of EVM instructions may change significantly during hard forks which may break already deployed contract systems that make fixed assumptions about gas costs.

## Impact
The use of the deprecated transfer() function for an address will inevitably make the transaction fail when:

The claimer smart contract does not implement a payable function.
The claimer smart contract does implement a payable fallback which uses more than 2300 gas unit.
The claimer smart contract implements a payable fallback function that needs less than 2300 gas units but is called through proxy, raising the call's gas usage above 2300.
Additionally, using higher than 2300 gas might be mandatory for some multisig wallets.

## Code Snippet
https://github.com/sherlock-audit/2022-12-notional/blob/main/contracts/vaults/BaseStrategyVault.sol#L188-L189

## Tool used
Manual Review

## Recommendation
Use call() instead of transfer()