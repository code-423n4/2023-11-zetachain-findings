## Summary 
The .call function was introduced in Solidity version 0.4.0, which was released in March 2016. This function was used to call arbitrary contracts and functions, and it's a powerful tool for interacting with other contracts in the Ethereum network.

Ever since its introduction, transfer() has typically been recommended by the security community because it helps guard against reentrancy attacks. This guidance made sense under the assumption that gas costs wouldn’t change, but that assumption turned out to be incorrect. We now recommend that transfer() be avoided as gas costs can and will change.

And if gas costs are subject to changes, then smart contracts can not depend on any particular gas costs. A smart contract making use of transfer() is taking a hard dependency on gas costs by forwarding a fixed amount of gas i.e., 2300.

## Impact
There is a gas limit of 2300 gas, which is enough to complete the transfer operation leading to out of gas error.
```
    function withdraw(uint wad) public {
        require(balanceOf[msg.sender] >= wad);
        balanceOf[msg.sender] -= wad;
        msg.sender.transfer(wad); @audit-issue 
        Withdrawal(msg.sender, wad);
    }

https://github.com/code-423n4/2023-11-zetachain/blob/2834e3f85b2c7774e97413936018a0814c57d860/repos/protocol-contracts/contracts/zevm/WZETA.sol#L28
```
## Recommendation
Allow users to set their gas usage which allows interoperability too.
https://docs.soliditylang.org/en/v0.4.18/control-structures.html#function-calls