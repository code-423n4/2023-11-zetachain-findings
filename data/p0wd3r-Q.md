# Repeatedly called code

This line duplicates the call to `RemoveInTxTrackerIfExists` in `defer`.
https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/x/crosschain/keeper/keeper_cross_chain_tx_vote_inbound_tx.go#L241
https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/x/crosschain/keeper/keeper_cross_chain_tx_vote_inbound_tx.go#L123-L129

The `chain == nil` decision is duplicated here.
https://github.com/code-423n4/2023-11-zetachain/blob/main/repos/node/x/crosschain/keeper/keeper_gas_price.go#L129-L137

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