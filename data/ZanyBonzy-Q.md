- [`ZRC20.sol`](https://github.com/code-423n4/2023-11-zetachain/blob/b237708ed5e86f12c4bddabddfd42f001e81941a/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L160C1-L163C67) - Spender's allowance decreases even when allowance is set to `type(uint256).max`.
In most erc20 token implementations, `type(uint256).max` is taken as infinite, and spender's allowance is not reduced when it has been set to this value.The current implementation the `ZRC20` token does not follow this logic and decrease it by the amount transferred from the sender to the recipient. This might create issues in contracts that are used to a more common behavior like the one used in OpenZeppelin/Solmate and have approved `ZRC20` token only once (without a way to update such value) because at one point, the the spender would not have enough allowance anymore.

```
    if (currentAllowance != type(uint256).max) {
        uint256 currentAllowance = _allowances[sender][_msgSender()];
        if (currentAllowance < amount) revert LowAllowance();

        _approve(sender, _msgSender(), currentAllowance - amount);
    }

```

- [ERC20Custody.sol](https://github.com/code-423n4/2023-11-zetachain/blob/b237708ed5e86f12c4bddabddfd42f001e81941a/repos/protocol-contracts/contracts/evm/ERC20Custody.sol#L153) - Funds can be lost if assets are unwhitelisted indefintely before withdrawing their balance. 
The TSS can remove supported assets, by calling the `unwhitelist` function,
```
    function unwhitelist(IERC20 asset) external onlyTSS {
        whitelisted[asset] = false;
        emit Unwhitelisted(asset);
    }
```
but this function doesn't checks if the contract holds tokens of the asset to be unwhitelisted. The deposited tokens may be "lost" if the removed asset are somehow forgotten or to be removed indefinetly. Consider withdrawing all asset balances before removing from whitelist.

- Hardcoded swap path might not be the most optimal/liquid one
In the `ZetaTokenConsumerPancakeV3.strategy.sol`, `ZetaTokenConsumerTrident.strategy.sol`, `ZetaTokenConsumerUniV3.strategy.sol` contracts, a path through `WETH` is defined and hardcoded (token - weth - zetatoken) and vice versa. Hardcoding this path might not result on the optimal route and this wouldn't be able to be modified after deployment.
For example, there might be more liquidity in another path, or the `WETH`/token pool might not might not exist. Using a hardcoded swap may force swaps on a improper pool with little liquidity which can cause loss of funds, users might not be able to make swaps when `amountOut` is less than user specified `amountOutMinimum` or if the pool doesn't exist.
Consider adding setter functions for both the token paths and make them configurable.