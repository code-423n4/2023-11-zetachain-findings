## Summary 
The .call function was introduced in Solidity version 0.4.0, which was released in March 2016. This function was used to call arbitrary contracts and functions, and it's a powerful tool for interacting with other contracts in the Ethereum network.

## Impact
There is a gas limit of 2300 gas, which is enough to complete the transfer operation leading to out of gas error
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