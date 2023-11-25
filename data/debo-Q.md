## [L-01] SWC-115 Use of `"tx.origin"` as a part of `authorization` control.
## IMPACT
Using `"tx.origin"` as a security control can lead to `authorization` bypass vulnerabilities. 
A call could be made to the vulnerable contract that passes the authorization check since tx.origin returns the original sender of the transaction which in this case is the authorized account.
Consider using `"msg.sender"` unless you really know what you are doing.
## POC
**Vulnerable Code**
```sol
// Line 99
            tx.origin,
```
## References
```https://github.com/code-423n4/2023-11-zetachain/blob/2834e3f85b2c7774e97413936018a0814c57d860/repos/protocol-contracts/contracts/zevm/ZetaConnectorZEVM.sol#L99```
## Remediation
`tx.origin` should not be used for `authorization`. Use `msg.sender` instead.