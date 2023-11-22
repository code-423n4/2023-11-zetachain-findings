[L-1] Validating initializationCode Length and Eliminating Redundant Code
Validating the length of initializationCode is essential, and the computation of targetDeploymentAddress is unnecessary at this stage; it can be checked after a successful deployment or just remove it. The following is the final code with all optimization:
```solidity
File: repos/protocol-contracts/evm/tools/ImmutableCreate2Factory.sol
 function safeCreate2Internal(
        bytes32 salt,
        bytes memory initializationCode
    ) internal returns (address deploymentAddress) {
        require(initializationCode.length != 0, "Bytecode length is zero");
        // using inline assembly: load data and length of data, then call CREATE2.
        assembly {
            deploymentAddress := create2(
                callvalue, // wei sent with current call
                // Actual code starts after skipping the first 32 bytes
                add(initializationCode, 0x20),
                mload(initializationCode), // Load the size of code contained in the first 32 bytes
                salt // Salt from function arguments
            )
        }
        require(deploymentAddress != address(0), "Failed on deploy");
        // ensure that a contract hasn't been previously deployed to target address.
        // this require also unnecessary
        require(!_deployed[deploymentAddress], "Contract deployed");
        // record the deployment of the contract to prevent redeploys.
        _deployed[deploymentAddress] = true;
    }

```
For the above code require: 
```require(!_deployed[deploymentAddress], "Contract deployed"); ```
is also unnecessary, as it will revert when deploying to the same address. I suggest removing it too.

[L-2] WZETA transferFrom does not follow spec
There are two conditions that do not adhere to the specifications.
1. If the message sender is the source of a transferFrom call, the sender’s allowance will not
be considered, and the transfer will initiate immediately. This breaks invariants expected of
transferFrom.
Traditionally, the transferFrom method moves tokens from one account to another,
provided the source account has approved the sender to send such an amount using the
ERC20 method approve. However, the transferFrom function in WETH9’s ERC20 token
does not require approval if the sender is the source of the account.

2. If the message sender is not the source of a transferFrom call, and the allowance is set to the maximum value of uint256, the sender's allowance will not be subtracted. Consequently, the transfer will be initiated immediately. This violation of the expected invariants of transferFrom can lead to issues.

```solidity
File: repos/protocol-contracts/contracts/zevm/WZETA.sol
 function transferFrom(address src, address dst, uint wad) public returns (bool) {
        require(balanceOf[src] >= wad);

        if (src != msg.sender && allowance[src][msg.sender] != uint(-1)) {
            require(allowance[src][msg.sender] >= wad);
            allowance[src][msg.sender] -= wad;
        }

        balanceOf[src] -= wad;
        balanceOf[dst] += wad;

        Transfer(src, dst, wad);

        return true;
    }

```
Exploit Scenario
Alice sends a transaction that invokes transferFrom with her own address as the source
address, assuming it will fail if no approval was set beforehand. Instead, the transfer
succeeds, and Alice’s funds are lost.

Recommendation
Short term, document this contract’s non-standard behavior and verify that all code
interfacing with it does not break due to this behavior.

Long term, use Echidna to review the ERC20 specification and verify your contracts meet
the standard. When interfacing with external ERC20 tokens, be wary of popular tokens that
do not properly implement the standard (e.g., many tokens do not include return values for
approve, transfer, transferFrom, etc.).

