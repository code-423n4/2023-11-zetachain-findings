### **[[ 1 ]]** 
`evm_signer::TryProcessOutTx` `err` variable doesn't need to be declared [here](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/zetaclient/evm_signer.go#L330), as it will be created [here](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/zetaclient/evm_signer.go#L357) and is not used before that. So the [following code](https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/zetaclient/evm_signer.go#L351-L354) is also useless. So I suggest to remove those.
```diff
	var to ethcommon.Address
-	var err error
	var toChain *common.Chain
	if send.CctxStatus.Status == types.CctxStatus_PendingRevert {
		to = ethcommon.HexToAddress(send.InboundTxParams.Sender)
		toChain = common.GetChainFromChainID(send.InboundTxParams.SenderChainId)
		if toChain == nil {
			logger.Error().Msgf("Unknown chain: %d", send.InboundTxParams.SenderChainId)
			return
		}
		logger.Info().Msgf("Abort: reverting inbound")
	} else if send.CctxStatus.Status == types.CctxStatus_PendingOutbound {
		to = ethcommon.HexToAddress(send.GetCurrentOutTxParam().Receiver)
		toChain = common.GetChainFromChainID(send.GetCurrentOutTxParam().ReceiverChainId)
		if toChain == nil {
			logger.Error().Msgf("Unknown chain: %d", send.GetCurrentOutTxParam().ReceiverChainId)
			return
		}
	} else {
		logger.Info().Msgf("Transaction doesn't need to be processed status: %d", send.CctxStatus.Status)
		return
	}
-	if err != nil {
-		logger.Error().Err(err).Msg("ParseChain fail; skip")
-		return
-	}

	// Early return if the cctx is already processed
	included, confirmed, err := evmClient.IsSendOutTxProcessed(send.Index, send.GetCurrentOutTxParam().OutboundTxTssNonce, send.GetCurrentOutTxParam().CoinType, logger)
	if err != nil {
		logger.Error().Err(err).Msg("IsSendOutTxProcessed failed")
	}
```
https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/zetaclient/evm_signer.go#L351-L354


### **[[ 2 ]]** 
`evm_signer::TryProcessOutTx` does allow to send out CCTX tx to address(0), which would effectivelly cause a funds loss. This is low as self-reck. I would recommend to add the following conditions.

```diff
	if send.CctxStatus.Status == types.CctxStatus_PendingRevert {
		to = ethcommon.HexToAddress(send.InboundTxParams.Sender)
		toChain = common.GetChainFromChainID(send.InboundTxParams.SenderChainId)
		if toChain == nil {
			logger.Error().Msgf("Unknown chain: %d", send.InboundTxParams.SenderChainId)
			return
		}
+		if to == (ethcommon.Address{}) {
+			logger.Error().Msgf("Invalid receiver %s", send.InboundTxParams.Sender)
+			return
+		}
		logger.Info().Msgf("Abort: reverting inbound")
	} else if send.CctxStatus.Status == types.CctxStatus_PendingOutbound {
		to = ethcommon.HexToAddress(send.GetCurrentOutTxParam().Receiver)
		toChain = common.GetChainFromChainID(send.GetCurrentOutTxParam().ReceiverChainId)
		if toChain == nil {
			logger.Error().Msgf("Unknown chain: %d", send.GetCurrentOutTxParam().ReceiverChainId)
			return
		}
+		if to == (ethcommon.Address{}) {
+			logger.Error().Msgf("Invalid receiver %s", send.GetCurrentOutTxParam().Receiver)
+			return
+		}
	} else {
		logger.Info().Msgf("Transaction doesn't need to be processed status: %d", send.CctxStatus.Status)
		return
	}
```

https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/zetaclient/evm_signer.go#L332-L349


### **[[ 3 ]]** 
`SystemContract.sol` could have some minor refactor introducing a `modifier` as follow to aliviate and make the code cleaner, and use it on all the `external` functions and the contructor as they are all gated with the `same condition` below.
```diff
+    /**
+     * @dev Only Fungible address allowed modifier.
+     */
+     modifier onlyFungibble() {
+        if (msg.sender != FUNGIBLE_MODULE_ADDRESS) revert CallerIsNotFungibleModule();
+        _;
+    }
```

https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/SystemContract.sol#L51


### **[[ 4 ]]** 
`ERC20Custody.sol` has the following useless code in `pause` function which I suggest to remove. Since the function is guarded by `onlyTSS` the modifier, which ensure `msg.sender == TSSAddress` before going even further into the function, this means `TSSAddress` can never be `address(0)`, which make this code totally useless. 
```diff
    function pause() external onlyTSS {
        if (paused) {
            revert IsPaused();
        }
-       if (TSSAddress == address(0)) {
-           revert ZeroAddress();
-       }
        paused = true;
        emit Paused(msg.sender);
    }
```
https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/evm/ERC20Custody.sol#L122-L124


### **[[ 5 ]]** 
`ERC20Custody.sol` has a `divergence` in the implementation regarding fees, consider the following:
- `zeta` which represent the Zeta token contract on the chain is `immutable`, so can't be changed after contract contruction.
- `zetaFee` is not immutable and can be updated at any point in time by `updateZetaFee`.

The `deposit` function along with the `constructor` are causing this divergence, as constructor allow zeta to be address(0) at construction, which deposit account for in the following code, but then if `zetaFee` are modified later on (Zeta wants to start charging fees on ERC20 deposits), those fees will never be able to be charged in case zeta was passed as address(0) at constrution.

So I would recommend the following changes to mitigate this. While we are here, I would also suggest to add a validation for `zetaMaxFee` in the constructor since it's immutable too.

```diff
    constructor(address TSSAddress_, address TSSAddressUpdater_, uint256 zetaFee_, uint256 zetaMaxFee_, IERC20 zeta_) {
        TSSAddress = TSSAddress_;
        TSSAddressUpdater = TSSAddressUpdater_;
        zetaFee = zetaFee_;
+       if (zeta == address(0)) revert ZeroAddress();
        zeta = zeta_;
        zetaMaxFee = zetaMaxFee_;
    }

    function deposit(
        bytes calldata recipient,
        IERC20 asset,
        uint256 amount,
        bytes calldata message
    ) external nonReentrant {
        if (paused) {
            revert IsPaused();
        }
        if (!whitelisted[asset]) {
            revert NotWhitelisted();
        }
-       if (zetaFee != 0 && address(zeta) != address(0)) {
+       if (zetaFee != 0) {
            zeta.safeTransferFrom(msg.sender, TSSAddress, zetaFee);
        }
        uint256 oldBalance = asset.balanceOf(address(this));
        asset.safeTransferFrom(msg.sender, address(this), amount);
        // In case if there is a fee on a token transfer, we might not receive a full exepected amount
        // and we need to correctly process that, o we subtract an old balance from a new balance, which should be an actual received amount.
        emit Deposited(recipient, asset, asset.balanceOf(address(this)) - oldBalance, message);
    }
```
https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/evm/ERC20Custody.sol#L177


### **[[ 6 ]]** 
`evm_signer` is hardcoding the `gaslimit` at `21000` in multiples places (see links below). We have seen in previous EVM forks changing the gas cost of some key opcode, for example the SSTORE opcode. This can happen again in the future and if it does happen that this hardcoded gas limit is not enought anymore for the target chain, it will lead to CCTX not working as expected and failures on the network. Finally, since this value need to be working for all the EVM-supported chain, the gas costs can be different to a level that this hardcoded value will not satisfy all the chains.

I would recommend to fetch this value at least from a configuration file, and have it per EVM chain, such that it is flexible enought to adapt to the future not requiring to recompile a new binary.

https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/zetaclient/evm_signer.go#L210
https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/zetaclient/evm_signer.go#L237
https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/zetaclient/evm_signer.go#L286
