# [L-01]: Failed event firing due to omission of `emit` keyword. 

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
Use the `emit` keyword to fire the events. 

```Solidity
function deposit() public payable {
    balanceOf[msg.sender] += msg.value;
    emit Deposit(msg.sender, msg.value);
}
```