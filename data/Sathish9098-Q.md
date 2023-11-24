# QA Report

##

## [L-3] Contradiction between documentation and implementation in ``deposit()`` Function

### Impact

The comment suggests exclusive access for the Fungible module, but the implementation allows both the Fungible module and the System contract to call the function. This broader access control could be intentional (for added flexibility) or an oversight.

The comment describes the function as handling deposits from an external chain, but the implementation does not show any direct mechanism for interacting with external chains. Instead, it mints tokens on the current chain.

### POC
```solidity
FILE: Breadcrumbs2023-11-zetachain/repos/protocol-contracts/contracts/zevm/ZRC20.sol

 /**
     * @dev Deposits corresponding tokens from external chain, only callable by Fungible module.
     * @param to, recipient address.
     * @param amount, amount to deposit.
     * @return true/false if succeeded/failed.
     */
    function deposit(address to, uint256 amount) external override returns (bool) {
        if (msg.sender != FUNGIBLE_MODULE_ADDRESS && msg.sender != SYSTEM_CONTRACT_ADDRESS) revert InvalidSender();
        _mint(to, amount);
        emit Deposit(abi.encodePacked(FUNGIBLE_MODULE_ADDRESS), to, amount);
        return true;
    }

```
https://github.com/code-423n4/2023-11-zetachain/blob/2834e3f85b2c7774e97413936018a0814c57d860/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L219-L230

### Recommended Mitigation

```solidity
/**
 * @dev Mints tokens to a specific address, callable by either Fungible module or System contract.
 * This function does not handle external chain deposits directly but mints new tokens on the current chain.
 * Access is restricted to ensure controlled minting operations.
 * @param to The recipient address to receive the minted tokens.
 * @param amount The amount of tokens to be minted and deposited to the 'to' address.
 * @return true Indicates successful execution of the function.
 */

```


## [L-1] Hardcoded ChainId

### Impact
In systems where signatures are used for authentication or transaction verification, a hardcoded CHAIN_ID can cause issues if the network’s chain ID changes. This could invalidate existing signatures or compromise the integrity of new ones.

For protocols designed to operate across multiple chains, a hardcoded CHAIN_ID can limit functionality. It may prevent the protocol from correctly identifying and interacting with different networks, thereby reducing its cross-chain operability.

Hardcoding CHAIN_ID can create hurdles when moving a contract from a testnet to the mainnet or vice versa, as these environments typically have different chain IDs.

### POC

```solidity

FILE: 2023-11-zetachain/repos/protocol-contracts/contracts/zevm
/ZRC20.sol

24:  uint256 public immutable CHAIN_ID;

```
https://github.com/code-423n4/2023-11-zetachain/blob/2834e3f85b2c7774e97413936018a0814c57d860/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L24

### Recommended Mitigation
Make chain id configurable 

```diff
- 24:  uint256 public immutable CHAIN_ID;
+ 24:  uint256 public CHAIN_ID;
```

##

## [L-2] Lack of sanity checks for ``address`` and ``uint256``

### Impact
The absence of sanity checks for address and uint256 types in the provided smart contract constructor can lead to several potential issues and vulnerabilities.

Add checks to ensure that no critical addresses are set to the zero address. Additionally, verify the type of address if necessary (e.g., smart contract vs. EOA).

Implement range checks and logical consistency validations for uint256 parameters. Ensure that values like gasLimit_ are within sensible limits.

### POC
```solidity
FILE: 2023-11-zetachain/repos/protocol-contracts/contracts/zevm/ZRC20.sol

constructor(
        string memory name_,
        string memory symbol_,
        uint8 decimals_,
        uint256 chainid_,
        CoinType coinType_,
        uint256 gasLimit_,
        address systemContractAddress_
    ) {
        if (msg.sender != FUNGIBLE_MODULE_ADDRESS) revert CallerIsNotFungibleModule();
        _name = name_;
        _symbol = symbol_;
        _decimals = decimals_;
        CHAIN_ID = chainid_;
        COIN_TYPE = coinType_;
        GAS_LIMIT = gasLimit_;
        SYSTEM_CONTRACT_ADDRESS = systemContractAddress_;
    }

```
https://github.com/code-423n4/2023-11-zetachain/blob/2834e3f85b2c7774e97413936018a0814c57d860/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L60-L77

### Recommended Mitigations

```solidity

require(systemContractAddress_ != address(0), "System contract address cannot be zero");

    // uint256 sanity checks
    require(chainid_ > 0, "Invalid Chain ID");
    require(gasLimit_ >= minGasLimit && gasLimit_ <= maxGasLimit, "Gas limit out of range");

```





