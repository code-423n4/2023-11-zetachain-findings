## L-01 : `transferFrom` not  following CEI 
the  `transferFrom` function in ZRC20.sol doesnot follow CEI pattern . It is standard to follow CEI when there is a fund transfer to prevent unwanted scenarios like re-entrancy , erc777 re-entrancy etc.  


```solidity 
 function transferFrom(address sender, address recipient, uint256 amount) public virtual override returns (bool) {
        _transfer(sender, recipient, amount);//@ci low not following CEI ..! 

        uint256 currentAllowance = _allowances[sender][_msgSender()];
        if (currentAllowance < amount) revert LowAllowance();

        _approve(sender, _msgSender(), currentAllowance - amount);

        return true;
    }

```

code link : https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L157

## Recommendation : 
rewrite the function as follows to follow CEI : 
```diff
  function transferFrom(address sender, address recipient, uint256 amount) public virtual override returns (bool) {
-       _transfer(sender, recipient, amount);//@ci low not following CEI ..! 

        uint256 currentAllowance = _allowances[sender][_msgSender()];
        if (currentAllowance < amount) revert LowAllowance();

        _approve(sender, _msgSender(), currentAllowance - amount);
+       _transfer(sender, recipient, amount);
        return true;
    }
```

## L-0:  Zero withdraw amounts should be reverted cause withdraw charges gas amount to user . 
`withdraw` function  Withraws ZRC20 tokens to external chains,  causes cctx module to send out outbound tx to the outbound chain. 
`withdraw ` functionality in ZRC20.sol charges calculated gas Fee to the user and burns the token amount . If an user mistakenly inputs zero as amount it still charges gas fee from the user . Which should not be the case . 
 

```solidity 

 function withdraw(bytes memory to, uint256 amount) external override returns (bool) {
        (address gasZRC20, uint256 gasFee) = withdrawGasFee();
        if (!IZRC20(gasZRC20).transferFrom(msg.sender, FUNGIBLE_MODULE_ADDRESS, gasFee)) {
            revert GasFeeTransferFailed();
        }
        _burn(msg.sender, amount);//@ci low zero amounts needs to be restricted here . 
        emit Withdrawal(msg.sender, to, amount, gasFee, PROTOCOL_FLAT_FEE);
        return true;
    }
```

code link : https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L256

## Recommendation : 
Zero withdraw amounts should be reverted . 
```diff
 function withdraw(bytes memory to, uint256 amount) external override returns (bool) {
+       require(amount > 0 , "Amount should be larger than zero ");
        (address gasZRC20, uint256 gasFee) = withdrawGasFee();
        if (!IZRC20(gasZRC20).transferFrom(msg.sender, FUNGIBLE_MODULE_ADDRESS, gasFee)) {
            revert GasFeeTransferFailed();
        }
        _burn(msg.sender, amount);//@ci low zero amounts needs to be restricted here . 
        emit Withdrawal(msg.sender, to, amount, gasFee, PROTOCOL_FLAT_FEE);
        return true;
    }
```

## L-03: No error msg in `WZETA::withdraw` 
Withdraw function in WZETA.sol does not throw any error message while reverting . The below require statement should throw an error message explaining the  reason of error . 
```solidity 
 function withdraw(uint wad) public {
        require(balanceOf[msg.sender] >= wad);//@ci low doesnot throw any error message !
        balanceOf[msg.sender] -= wad;
        msg.sender.transfer(wad);
        Withdrawal(msg.sender, wad);
    }

```

code link : https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/WZETA.sol#L25

## Recommendation : 
Add error msg to the code as follows : 
```diff
 function withdraw(uint wad) public {
-       require(balanceOf[msg.sender] >= wad);
+       require(balanceOf[msg.sender] >= wad , " Withdraw Amount can't be larger than balance" );
        balanceOf[msg.sender] -= wad;
        msg.sender.transfer(wad);
        Withdrawal(msg.sender, wad);
    }
```

## L-04:  title 


```solidity 


```

code link : 

## Recommendation : 

```diff
```