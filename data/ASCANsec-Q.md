## Wrong naming of Interfaces in ZetaNonEthInterface
***It is clear that the ZetaNonEthInterface.sol file in the zeta chain repositories is intended to be an interface. However, it was mistakenly given the name ZetaNonEthInterface instead of IZetaNonEth. By following the convention of naming interfaces with an "I" prefix in Solidity, renaming it to IZetaNonEth would make it easy to identify as an interface and avoid any confusion.***

## Wrong naming of interfaces in CommonErrors contract
***It is essential to have a clear contract naming convention to ensure readability and prevent confusion. The CommonError contract serves as a notification on the blockchain and is utilized by every function. However, the contract's current name does not reflect its nature as an interface, which could make it difficult for users to understand its purpose. Therefore, it is crucial to rename the contract appropriately to improve its clarity and readability.***

##### Recommendation 
***Change CommonError.sol to ICommonError.sol since it an interface.***

## Wrong naming of interfaces in ZetaInteractorErrors
***There are numerous issues with the naming conventions of the Zeta Chain Protocol. However, it's important to note that when using a contract as an interface, it should be clearly specified as such. Unfortunately, the EVM interface in the Zeta Chain contract has naming flaws that can lead to confusion. In order to avoid such confusion, it's essential to follow best practices when naming a contract.***

##### Recommendation
***Change [ZetaInteractorErros.sol](https://github.com/code-423n4/2023-11-zetachain/tree/main/repos/protocol-contracts/contracts/evm/interfaces) to IZetaInteractorErros.sol***

## Avoid the wrong naming of interfaces
***This issue affects multiple contracts in the Zeta EVM chain, such as zContract, ZetaErrors, TridalPoolRouter, ZetaInterfaces, and Interfaces.***

##### Recommendation
***Change all interface contract I mentioned by adding ```I``` as a suffix.***

## Unused contract
***It seems that the [contracts/zevm/UniswapPeriphery.sol](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/UniswapPeriphery.sol) and [Uniswap.sol](https://github.com/code-423n4/2023-11-zetachain/blob/2834e3f85b2c7774e97413936018a0814c57d860/repos/protocol-contracts/contracts/zevm/Uniswap.sol#L4), [Zeta.eth.sol](sol) [repos/protocol-contracts/contracts/evm/interfaces/ZetaNonEthInterface.sol](https://github.com/code-423n4/2023-11-zetachain/blob/2834e3f85b2c7774e97413936018a0814c57d860/repos/protocol-contracts/contracts/evm/interfaces/ZetaNonEthInterface.sol#L4), [contracts/evm/Zeta.eth.sol](https://github.com/search?q=repo%3Acode-423n4%2F2023-11-zetachain%20contracts%2Fevm%2FZeta.eth.sol&type=code),  was not imported into any other contract, indicating that it might be unused. it might be unused.***
#### Recommendation 
***Either it should be used or deleted***

## No documentation about the interface functions
***It appears that there is a lack of documentation regarding the usage and origin of the interface contract in the Zeta EVM chain. It would be helpful to have more detailed information about which contracts are using this interface and where it was forked from.***

#### Recommendation
***Giving concise details about the interface will help users understand the protocol better when imported into a contract.***

## Avoid using floating pragma
[repos/protocol-contracts/contracts/zevm/WZETA.sol](https://github.com/code-423n4/2023-11-zetachain/blob/2834e3f85b2c7774e97413936018a0814c57d860/repos/protocol-contracts/contracts/zevm/WZETA.sol#L5)


##### Recommendation
***Use a fixed pragma***

## Avoid using the Old solidity version
***[repos/protocol-contracts/contracts/zevm/UniswapPeriphery.sol](https://github.com/code-423n4/2023-11-zetachain/blob/2834e3f85b2c7774e97413936018a0814c57d860/repos/protocol-contracts/contracts/zevm/UniswapPeriphery.sol#L4)***





























repos/protocol-contracts/contracts/zevm/ZetaConnectorZEVM.sol
repos/protocol-contracts/contracts/zevm/SystemContract.sol
repos/protocol-contracts/contracts/evm/tools/ZetaTokenConsumerUniV2.strategy.sol
repos/protocol-contracts/contracts/evm/ERC20Custody.sol
repos/protocol-contracts/contracts/evm/tools/ZetaTokenConsumerUniV3.strategy.sol
repos/protocol-contracts/contracts/evm/tools/ZetaTokenConsumerTrident.strategy.sol
repos/protocol-contracts/contracts/evm/tools/ZetaTokenConsumerPancakeV3.strategy.sol
contracts/zevm/ZRC20.sol

# [L1]No address zero checks in constructors
***


# [L2] Use of old solidity version


# [L3] Missing Interface Usage in WZETA Contract
***The contract [WZETA](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/WZETA.sol) does not utilize or import its associated interface, [IWZETA](https://github.com/code-423n4/2023-11-zetachain/tree/main/repos/protocol-contracts/contracts/zevm/interfaces/IWZETA.sol). This finding could potentially lead to a lack of clarity in the contract's intended structure and interactions. Interfaces serve as a crucial element in ensuring standardized communication between contracts and providing a clear definition of expected functionalities.***

#### Impact
***Lack of interface usage might make it challenging for developers, auditors, and other stakeholders to quickly understand the intended functions and external interactions of the WZETA contract.***
***Interfaces play a key role in standardizing contract interactions. Without importing and implementing the associated interface, there might be a risk of non-standard or unexpected behavior when interacting with other contracts or systems.***

### Recommendation
***It is recommended to import the IWZETA interface in the WZETA contract and ensure that all the required functions are appropriately implemented. This step will enhance the contract's readability and provide a clear definition of its expected behavior.***

