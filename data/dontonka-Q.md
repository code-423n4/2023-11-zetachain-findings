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