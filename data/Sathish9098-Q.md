# QA Report

##

## [L-1] Hardcoded ChainId

### Impact
In systems where signatures are used for authentication or transaction verification, a hardcoded CHAIN_ID can cause issues if the network’s chain ID changes. This could invalidate existing signatures or compromise the integrity of new ones.

For protocols designed to operate across multiple chains, a hardcoded CHAIN_ID can limit functionality. It may prevent the protocol from correctly identifying and interacting with different networks, thereby reducing its cross-chain operability.

Hardcoding CHAIN_ID can create hurdles when moving a contract from a testnet to the mainnet or vice versa, as these environments typically have different chain IDs.

### POC

```solidity

FILE: 2023-11-zetachain/repos/protocol-contracts/contracts/zevm
/ZRC20.sol

24:  uint256 public immutable CHAIN_ID;

```
https://github.com/code-423n4/2023-11-zetachain/blob/2834e3f85b2c7774e97413936018a0814c57d860/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L24

##

## [L-2] Lack of sanity checks for address 


