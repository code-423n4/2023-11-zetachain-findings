# Repeatedly called code

This line duplicates the call to `RemoveInTxTrackerIfExists` in `defer`.
https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/x/crosschain/keeper/keeper_cross_chain_tx_vote_inbound_tx.go#L241
https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/x/crosschain/keeper/keeper_cross_chain_tx_vote_inbound_tx.go#L123-L129

The `chain == nil` decision is duplicated here.
https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/x/crosschain/keeper/keeper_gas_price.go#L129-L137

Here, the range of `confirmedBlockNum` is checked twice for duplication.
https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/zetaclient/bitcoin_client.go#L332-L335
https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/zetaclient/bitcoin_client.go#L210-L218

# Prints logs without determining the `to` of the donation transaction

This loop iterates through all the transactions in the block, determining whether they are donations or not is not interpreting whether the `to` address is the project address.

https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/zetaclient/evm_client.go#L943-L952
```go
			for _, tx := range block.Transactions() {
				if tx.To() == nil {
					continue
				}
				if bytes.Compare(tx.Data(), []byte(DonationMessage)) == 0 {
					ob.logger.ExternalChainWatcher.Info().Msgf("thank you rich folk for your donation!: %s", tx.Hash().Hex())
					continue
				}

				if *tx.To() == tssAddress {
```

# Events are never triggered

The transferFrom of ZETA, WZETA and ZRC20 will always return true, so the events in the following lines will never be triggered.

https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/evm/ZetaConnector.eth.sol#L60-L61
```solidity
        bool success = IERC20(zetaToken).transfer(destinationAddress, zetaValue);
        if (!success) revert ZetaTransferError();
```

https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/ZetaConnectorZEVM.sol#L94
```solidity
        if (!WZETA(wzeta).transferFrom(msg.sender, address(this), input.zetaValueAndGas)) revert WZETATransferFailed();
```

https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/protocol-contracts/contracts/zevm/ZRC20.sol#L258-L260
```solidity
        if (!IZRC20(gasZRC20).transferFrom(msg.sender, FUNGIBLE_MODULE_ADDRESS, gasFee)) {
            revert GasFeeTransferFailed();
        }
```
