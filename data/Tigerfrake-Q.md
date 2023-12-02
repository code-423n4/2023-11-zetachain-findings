# [L-01]: Redundant State changes.

##### Instances:
1. <https://github.com/code-423n4/2023-11-zetachain/blob/6a9fdc29ce7e142facfcd6f15a16bf00b659d53b/repos/protocol-contracts/contracts/evm/ERC20Custody.sol#L144-L147>

```Solidity
function whitelist(IERC20 asset) external onlyTSS {
    whitelisted[asset] = true;
    emit Whitelisted(asset);
}
```
`whitelist()` function does not check that the asset in question isn’t already whitelisted.

2. <https://github.com/code-423n4/2023-11-zetachain/blob/6a9fdc29ce7e142facfcd6f15a16bf00b659d53b/repos/protocol-contracts/contracts/evm/ERC20Custody.sol#L153-L156>

```Solidity
function unwhitelist(IERC20 asset) external onlyTSS {
    whitelisted[asset] = false;
    emit Unwhitelisted(asset);
}
```
`unwhitelist()` function does not check whether the asset in question isn't already unwhitelisted.

## Impact
This would result to unnecessary Redundant State changes that might lead to  confusion as the state of the contract does not accurately represent the intended logic.

## Mitigation
> Any function that is involved in the changing of state in a contract should first ensure that the change for which it is designed to effect isn't already satisfied to to avoid redundant state changes


# [NC-01]: Failed event firing due to omission of `emit` keyword. 

Example: [deposit() function](https://github.com/code-423n4/2023-11-zetachain/blob/6a9fdc29ce7e142facfcd6f15a16bf00b659d53b/repos/protocol-contracts/contracts/zevm/WZETA.sol#L20-L23)
```Solidity
function deposit() public payable {
    balanceOf[msg.sender] += msg.value;
    Deposit(msg.sender, msg.value);
}
```

## Impact
The events will not actually be emitted and this means that the listeners will not receive it and this will break integrations that rely on events.

##### More instances.
- <https://github.com/code-423n4/2023-11-zetachain/blob/6a9fdc29ce7e142facfcd6f15a16bf00b659d53b/repos/protocol-contracts/contracts/zevm/WZETA.sol#L25-L30>
- <https://github.com/code-423n4/2023-11-zetachain/blob/6a9fdc29ce7e142facfcd6f15a16bf00b659d53b/repos/protocol-contracts/contracts/zevm/WZETA.sol#L36-L39>

## Mitigation
> Use the `emit` keyword to fire the events. 

```Solidity
function deposit() public payable {
    balanceOf[msg.sender] += msg.value;
    emit Deposit(msg.sender, msg.value);
}
```

# [NC-02]: Inaccurate use of `virtual` and `override` specifiers. 
## Impact
Using `virtual` and `override` specifiers   in the same function  is invalid syntax and will result in compiler error. 
Example:
```Solidity
function name() public view virtual override returns (string memory) {
    return _name;
}
```

##### More instances.
- <https://github.com/code-423n4/2023-11-zetachain/blob/6a9fdc29ce7e142facfcd6f15a16bf00b659d53b/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L91-L93>
- <https://github.com/code-423n4/2023-11-zetachain/blob/6a9fdc29ce7e142facfcd6f15a16bf00b659d53b/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L99-L101>
- <https://github.com/code-423n4/2023-11-zetachain/blob/6a9fdc29ce7e142facfcd6f15a16bf00b659d53b/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L107-L109>

## Mitigation
> Use `virtual` keyword when declaring a function in a base contract to indicate that it may be overridden by derived contracts.
>Use the `override` keyword when implementing a function in a derived contract that is intended to override a virtual function in the base contract.

# [NC-03]: Floating pragma effects

## Impact
Using floating pragmas would mean that future compiler changes break functionalities in the contract.

Example: 
```Solidity
pragma solidity ^0.4.18;
```
##### Instance:
- <https://github.com/code-423n4/2023-11-zetachain/blob/6a9fdc29ce7e142facfcd6f15a16bf00b659d53b/repos/protocol-contracts/contracts/zevm/WZETA.sol#L1>

## Mitigation
Lock `pragmas` to specify the compiler version. This ensures that future compiler changes do not break functionality.

```Solidity
pragma solidity 0.4.18;
```

