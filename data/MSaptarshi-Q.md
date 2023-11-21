# L-01. 
### Wrong event emit for 1.mint
https://github.com/code-423n4/2023-11-zetachain/blob/44c8dd426e829536850b5d42b3f0ade1ce29a23c/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L196
### 2.burn
https://github.com/code-423n4/2023-11-zetachain/blob/44c8dd426e829536850b5d42b3f0ade1ce29a23c/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L208
## Recommended: use different events for both the functions for improving code readability