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