## [L-01] SWC-123 Requirement violation
## Impact
A requirement was violated in a nested call and the call was reverted as a result. Make sure valid inputs are provided to the nested call (for instance, via passed arguments).
## Remediation
If the required logical condition is too strong, it should be weakened to allow all valid external inputs.
Otherwise, the bug must be in the contract that provided the external input and one should consider fixing its code by making sure no invalid inputs are provided.
## Location
```sol
// https://github.com/code-423n4/2023-11-zetachain/blob/2834e3f85b2c7774e97413936018a0814c57d860/repos/protocol-contracts/contracts/evm/ERC20Custody.sol#L11
contract ERC20Custody is ReentrancyGuard {
```