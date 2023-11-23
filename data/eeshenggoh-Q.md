## Unused contracts should be removed

These unused contracts should be removed, it can be misleading to developers and gas if accidentally deployed
```
repos/protocol-contracts/contracts/zevm/Uniswap.sol
repos/protocol-contracts/contracts/zevm/UniswapPeriphery.sol
```

## Zero Address Check

Apparently ** ALL THE CONTRACT IN SCOPE ** has no zero address check which means that you can accidentally transfer to zero address or deploy contracts with null address. Do include zero address checks in ALL contracts in scope. I did not add code lines here as there is too much.