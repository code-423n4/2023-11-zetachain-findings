# 01. Wrong event emit for 1.mint, 2.burn
https://github.com/code-423n4/2023-11-zetachain/blob/44c8dd426e829536850b5d42b3f0ade1ce29a23c/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L196
https://github.com/code-423n4/2023-11-zetachain/blob/44c8dd426e829536850b5d42b3f0ade1ce29a23c/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L208
## Recommended: 
### use different events for both the functions for improving code readability

# 02 use the particular name of the import that we are importing like this : 
+ import {IERC20} fom "@openzeppelin/contracts/token/ERC20/IERC20.sol";
## instead of this :
- import {IERC20} fom "@openzeppelin/contracts/token/ERC20/IERC20.sol";