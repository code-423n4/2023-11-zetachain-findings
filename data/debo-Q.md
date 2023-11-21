## [L-01] EVENT BASED REENTRANCY in the ZetaConnector.non-eth contract
In a Re-entrancy attack, a malicious contract calls back into the calling contract before the first invocation of the function is finished. 
This may cause the different invocations of the function to interact in undesirable ways, especially in cases where the function is updating state variables after the external calls.
In the case of event-based Re-entrancy attacks, events are emitted after an external call leading to missing event calls.
## Remediation and Mitigation
It is recommended to add a [Re-entrancy Guard] to the functions making external calls. 
The functions should use a Checks-Effects-Interactions pattern. 
The external calls should be executed at the end of the function and all the state-changing and event emits must happen before the call.
## Locations
```sol
// repos/protocol-contracts/contracts/evm/ZetaConnector.non-eth.sol#L40-L53
function send(ZetaInterfaces.SendInput calldata input) external override whenNotPaused {

        ZetaNonEthInterface(zetaToken).burnFrom(msg.sender, input.zetaValueAndGas);


        emit ZetaSent(

            tx.origin,

            msg.sender,

            input.destinationChainId,

            input.destinationAddress,

            input.zetaValueAndGas,

            input.destinationGasLimit,

            input.message,

            input.zetaParams

        );

    }

// repos/protocol-contracts/contracts/evm/ZetaConnector.non-eth.sol#L61-L79
function onReceive(

        bytes calldata zetaTxSenderAddress,

        uint256 sourceChainId,

        address destinationAddress,

        uint256 zetaValue,

        bytes calldata message,

        bytes32 internalSendHash

    ) external override onlyTssAddress {

        if (zetaValue + ZetaNonEthInterface(zetaToken).totalSupply() > maxSupply) revert ExceedsMaxSupply(maxSupply);

        ZetaNonEthInterface(zetaToken).mint(destinationAddress, zetaValue, internalSendHash);


        if (message.length > 0) {

            ZetaReceiver(destinationAddress).onZetaMessage(

                ZetaInterfaces.ZetaMessage(zetaTxSenderAddress, sourceChainId, destinationAddress, zetaValue, message)

            );

        }


        emit ZetaReceived(zetaTxSenderAddress, sourceChainId, destinationAddress, zetaValue, message, internalSendHash);

    }

// repos/protocol-contracts/contracts/evm/ZetaConnector.non-eth.sol#L87-L122
function onRevert(

        address zetaTxSenderAddress,

        uint256 sourceChainId,

        bytes calldata destinationAddress,

        uint256 destinationChainId,

        uint256 remainingZetaValue,

        bytes calldata message,

        bytes32 internalSendHash

    ) external override whenNotPaused onlyTssAddress {

        if (remainingZetaValue + ZetaNonEthInterface(zetaToken).totalSupply() > maxSupply)

            revert ExceedsMaxSupply(maxSupply);

        ZetaNonEthInterface(zetaToken).mint(zetaTxSenderAddress, remainingZetaValue, internalSendHash);


        if (message.length > 0) {

            ZetaReceiver(zetaTxSenderAddress).onZetaRevert(

                ZetaInterfaces.ZetaRevert(

                    zetaTxSenderAddress,

                    sourceChainId,

                    destinationAddress,

                    destinationChainId,

                    remainingZetaValue,

                    message

                )

            );

        }


        emit ZetaReverted(

            zetaTxSenderAddress,

            sourceChainId,

            destinationChainId,

            destinationAddress,

            remainingZetaValue,

            message,

            internalSendHash

        );

    }

// repos/protocol-contracts/contracts/evm/tools/ZetaTokenConsumerUniV3.strategy.sol#L98-L122
function getZetaFromToken(

        address destinationAddress,

        uint256 minAmountOut,

        address inputToken,

        uint256 inputTokenAmount

    ) external override returns (uint256) {

        if (destinationAddress == address(0) || inputToken == address(0)) revert ZetaCommonErrors.InvalidAddress();

        if (inputTokenAmount == 0) revert InputCantBeZero();


        IERC20(inputToken).safeTransferFrom(msg.sender, address(this), inputTokenAmount);

        IERC20(inputToken).safeApprove(address(uniswapV3Router), inputTokenAmount);


        ISwapRouter.ExactInputParams memory params = ISwapRouter.ExactInputParams({

            deadline: block.timestamp + MAX_DEADLINE,

            path: abi.encodePacked(inputToken, tokenPoolFee, WETH9Address, zetaPoolFee, zetaToken),

            recipient: destinationAddress,

            amountIn: inputTokenAmount,

            amountOutMinimum: minAmountOut

        });


        uint256 amountOut = uniswapV3Router.exactInput(params);


        emit TokenExchangedForZeta(inputToken, inputTokenAmount, amountOut);

        return amountOut;

    }

// repos/protocol-contracts/contracts/evm/tools/ZetaTokenConsumerUniV3.strategy.sol#L124-L156
function getEthFromZeta(

        address destinationAddress,

        uint256 minAmountOut,

        uint256 zetaTokenAmount

    ) external override returns (uint256) {

        if (destinationAddress == address(0)) revert ZetaCommonErrors.InvalidAddress();

        if (zetaTokenAmount == 0) revert InputCantBeZero();


        IERC20(zetaToken).safeTransferFrom(msg.sender, address(this), zetaTokenAmount);

        IERC20(zetaToken).safeApprove(address(uniswapV3Router), zetaTokenAmount);


        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({

            deadline: block.timestamp + MAX_DEADLINE,

            tokenIn: zetaToken,

            tokenOut: WETH9Address,

            fee: zetaPoolFee,

            recipient: address(this),

            amountIn: zetaTokenAmount,

            amountOutMinimum: minAmountOut,

            sqrtPriceLimitX96: 0

        });


        uint256 amountOut = uniswapV3Router.exactInputSingle(params);


        WETH9(WETH9Address).withdraw(amountOut);


        emit ZetaExchangedForEth(zetaTokenAmount, amountOut);


        (bool sent, ) = destinationAddress.call{value: amountOut}("");

        if (!sent) revert ErrorSendingETH();


        return amountOut;

    }

// repos/protocol-contracts/contracts/evm/tools/ZetaTokenConsumerPancakeV3.strategy.sol#L126-L149
function getZetaFromToken(

        address destinationAddress,

        uint256 minAmountOut,

        address inputToken,

        uint256 inputTokenAmount

    ) external override returns (uint256) {

        if (destinationAddress == address(0) || inputToken == address(0)) revert ZetaCommonErrors.InvalidAddress();

        if (inputTokenAmount == 0) revert InputCantBeZero();


        IERC20(inputToken).safeTransferFrom(msg.sender, address(this), inputTokenAmount);

        IERC20(inputToken).safeApprove(address(pancakeV3Router), inputTokenAmount);


        ISwapRouterPancake.ExactInputParams memory params = ISwapRouterPancake.ExactInputParams({

            path: abi.encodePacked(inputToken, tokenPoolFee, WETH9Address, zetaPoolFee, zetaToken),

            recipient: destinationAddress,

            amountIn: inputTokenAmount,

            amountOutMinimum: minAmountOut

        });


        uint256 amountOut = pancakeV3Router.exactInput(params);


        emit TokenExchangedForZeta(inputToken, inputTokenAmount, amountOut);

        return amountOut;

    }

// repos/protocol-contracts/contracts/evm/tools/ZetaTokenConsumerPancakeV3.strategy.sol#L151-L182
function getEthFromZeta(

        address destinationAddress,

        uint256 minAmountOut,

        uint256 zetaTokenAmount

    ) external override returns (uint256) {

        if (destinationAddress == address(0)) revert ZetaCommonErrors.InvalidAddress();

        if (zetaTokenAmount == 0) revert InputCantBeZero();


        IERC20(zetaToken).safeTransferFrom(msg.sender, address(this), zetaTokenAmount);

        IERC20(zetaToken).safeApprove(address(pancakeV3Router), zetaTokenAmount);


        ISwapRouterPancake.ExactInputSingleParams memory params = ISwapRouterPancake.ExactInputSingleParams({

            tokenIn: zetaToken,

            tokenOut: WETH9Address,

            fee: zetaPoolFee,

            recipient: address(this),

            amountIn: zetaTokenAmount,

            amountOutMinimum: minAmountOut,

            sqrtPriceLimitX96: 0

        });


        uint256 amountOut = pancakeV3Router.exactInputSingle(params);


        WETH9(WETH9Address).withdraw(amountOut);


        emit ZetaExchangedForEth(zetaTokenAmount, amountOut);


        (bool sent, ) = destinationAddress.call{value: amountOut}("");

        if (!sent) revert ErrorSendingETH();


        return amountOut;

    }

// repos/protocol-contracts/contracts/evm/tools/ZetaTokenConsumerTrident.strategy.sol#L99-L134
function getZetaFromToken(

        address destinationAddress,

        uint256 minAmountOut,

        address inputToken,

        uint256 inputTokenAmount

    ) external override returns (uint256) {

        if (destinationAddress == address(0) || inputToken == address(0)) revert ZetaCommonErrors.InvalidAddress();

        if (inputTokenAmount == 0) revert InputCantBeZero();


        IERC20(inputToken).safeTransferFrom(msg.sender, address(this), inputTokenAmount);

        IERC20(inputToken).safeApprove(address(tridentRouter), inputTokenAmount);


        (address token0, address token1) = getPair(inputToken, WETH9Address);

        address[] memory pairPools1 = poolFactory.getPools(token0, token1, 0, 1);


        (token0, token1) = getPair(WETH9Address, zetaToken);

        address[] memory pairPools2 = poolFactory.getPools(token0, token1, 0, 1);


        address[] memory path = new address[](2);

        path[0] = pairPools1[0];

        path[1] = pairPools2[0];


        IPoolRouter.ExactInputParams memory params = IPoolRouter.ExactInputParams({

            tokenIn: inputToken,

            amountIn: inputTokenAmount,

            amountOutMinimum: minAmountOut,

            path: path,

            to: destinationAddress,

            unwrap: false

        });


        uint256 amountOut = tridentRouter.exactInput(params);


        emit TokenExchangedForZeta(inputToken, inputTokenAmount, amountOut);

        return amountOut;

    }

// repos/protocol-contracts/contracts/evm/tools/ZetaTokenConsumerTrident.strategy.sol#L136-L164
function getEthFromZeta(

        address destinationAddress,

        uint256 minAmountOut,

        uint256 zetaTokenAmount

    ) external override returns (uint256) {

        if (destinationAddress == address(0)) revert ZetaCommonErrors.InvalidAddress();

        if (zetaTokenAmount == 0) revert InputCantBeZero();


        IERC20(zetaToken).safeTransferFrom(msg.sender, address(this), zetaTokenAmount);

        IERC20(zetaToken).safeApprove(address(tridentRouter), zetaTokenAmount);


        (address token0, address token1) = getPair(zetaToken, WETH9Address);

        address[] memory pairPools = poolFactory.getPools(token0, token1, 0, 1);


        IPoolRouter.ExactInputSingleParams memory params = IPoolRouter.ExactInputSingleParams({

            tokenIn: zetaToken,

            amountIn: zetaTokenAmount,

            amountOutMinimum: minAmountOut,

            pool: pairPools[0],

            to: destinationAddress,

            unwrap: true

        });


        uint256 amountOut = tridentRouter.exactInputSingle(params);


        emit ZetaExchangedForEth(zetaTokenAmount, amountOut);


        return amountOut;

    }

// repos/protocol-contracts/contracts/evm/tools/ZetaTokenConsumerUniV2.strategy.sol#L58-L93
function getZetaFromToken(

        address destinationAddress,

        uint256 minAmountOut,

        address inputToken,

        uint256 inputTokenAmount

    ) external override returns (uint256) {

        if (destinationAddress == address(0) || inputToken == address(0)) revert ZetaCommonErrors.InvalidAddress();

        if (inputTokenAmount == 0) revert InputCantBeZero();


        IERC20(inputToken).safeTransferFrom(msg.sender, address(this), inputTokenAmount);

        IERC20(inputToken).safeApprove(address(uniswapV2Router), inputTokenAmount);


        address[] memory path;

        if (inputToken == wETH) {

            path = new address[](2);

            path[0] = wETH;

            path[1] = zetaToken;

        } else {

            path = new address[](3);

            path[0] = inputToken;

            path[1] = wETH;

            path[2] = zetaToken;

        }


        uint256[] memory amounts = uniswapV2Router.swapExactTokensForTokens(

            inputTokenAmount,

            minAmountOut,

            path,

            destinationAddress,

            block.timestamp + MAX_DEADLINE

        );

        uint256 amountOut = amounts[path.length - 1];


        emit TokenExchangedForZeta(inputToken, inputTokenAmount, amountOut);

        return amountOut;

    }

// repos/protocol-contracts/contracts/evm/tools/ZetaTokenConsumerUniV2.strategy.sol#L95-L122
function getEthFromZeta(

        address destinationAddress,

        uint256 minAmountOut,

        uint256 zetaTokenAmount

    ) external override returns (uint256) {

        if (destinationAddress == address(0)) revert ZetaCommonErrors.InvalidAddress();

        if (zetaTokenAmount == 0) revert InputCantBeZero();


        IERC20(zetaToken).safeTransferFrom(msg.sender, address(this), zetaTokenAmount);

        IERC20(zetaToken).safeApprove(address(uniswapV2Router), zetaTokenAmount);


        address[] memory path = new address[](2);

        path[0] = zetaToken;

        path[1] = wETH;


        uint256[] memory amounts = uniswapV2Router.swapExactTokensForETH(

            zetaTokenAmount,

            minAmountOut,

            path,

            destinationAddress,

            block.timestamp + MAX_DEADLINE

        );


        uint256 amountOut = amounts[path.length - 1];


        emit ZetaExchangedForEth(zetaTokenAmount, amountOut);

        return amountOut;

    }

// repos/protocol-contracts/contracts/evm/tools/ZetaTokenConsumerUniV2.strategy.sol#L124-L160
function getTokenFromZeta(

        address destinationAddress,

        uint256 minAmountOut,

        address outputToken,

        uint256 zetaTokenAmount

    ) external override returns (uint256) {

        if (destinationAddress == address(0) || outputToken == address(0)) revert ZetaCommonErrors.InvalidAddress();

        if (zetaTokenAmount == 0) revert InputCantBeZero();


        IERC20(zetaToken).safeTransferFrom(msg.sender, address(this), zetaTokenAmount);

        IERC20(zetaToken).safeApprove(address(uniswapV2Router), zetaTokenAmount);


        address[] memory path;

        if (outputToken == wETH) {

            path = new address[](2);

            path[0] = zetaToken;

            path[1] = wETH;

        } else {

            path = new address[](3);

            path[0] = zetaToken;

            path[1] = wETH;

            path[2] = outputToken;

        }


        uint256[] memory amounts = uniswapV2Router.swapExactTokensForTokens(

            zetaTokenAmount,

            minAmountOut,

            path,

            destinationAddress,

            block.timestamp + MAX_DEADLINE

        );


        uint256 amountOut = amounts[path.length - 1];


        emit ZetaExchangedForToken(outputToken, zetaTokenAmount, amountOut);

        return amountOut;

    }

// repos/protocol-contracts/contracts/evm/ZetaConnector.eth.sol#L31-L45
function send(ZetaInterfaces.SendInput calldata input) external override whenNotPaused {

        bool success = IERC20(zetaToken).transferFrom(msg.sender, address(this), input.zetaValueAndGas);

        if (!success) revert ZetaTransferError();


        emit ZetaSent(

            tx.origin,

            msg.sender,

            input.destinationChainId,

            input.destinationAddress,

            input.zetaValueAndGas,

            input.destinationGasLimit,

            input.message,

            input.zetaParams

        );

    }

// repos/protocol-contracts/contracts/evm/ZetaConnector.eth.sol#L52-L70
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

// repos/protocol-contracts/contracts/evm/ZetaConnector.eth.sol#L77-L111
function onRevert(

        address zetaTxSenderAddress,

        uint256 sourceChainId,

        bytes calldata destinationAddress,

        uint256 destinationChainId,

        uint256 remainingZetaValue,

        bytes calldata message,

        bytes32 internalSendHash

    ) external override whenNotPaused onlyTssAddress {

        bool success = IERC20(zetaToken).transfer(zetaTxSenderAddress, remainingZetaValue);

        if (!success) revert ZetaTransferError();


        if (message.length > 0) {

            ZetaReceiver(zetaTxSenderAddress).onZetaRevert(

                ZetaInterfaces.ZetaRevert(

                    zetaTxSenderAddress,

                    sourceChainId,

                    destinationAddress,

                    destinationChainId,

                    remainingZetaValue,

                    message

                )

            );

        }


        emit ZetaReverted(

            zetaTxSenderAddress,

            sourceChainId,

            destinationChainId,

            destinationAddress,

            remainingZetaValue,

            message,

            internalSendHash

        );

    }

// repos/protocol-contracts/contracts/zevm/ZetaConnectorZEVM.sol#L92-L108
function send(ZetaInterfaces.SendInput calldata input) external {

        // Transfer wzeta to "fungible" module, which will be burnt by the protocol post processing via hooks.

        if (!WZETA(wzeta).transferFrom(msg.sender, address(this), input.zetaValueAndGas)) revert WZETATransferFailed();

        WZETA(wzeta).withdraw(input.zetaValueAndGas);

        (bool sent, ) = FUNGIBLE_MODULE_ADDRESS.call{value: input.zetaValueAndGas}("");

        if (!sent) revert FailedZetaSent();

        emit ZetaSent(

            tx.origin,

            msg.sender,

            input.destinationChainId,

            input.destinationAddress,

            input.zetaValueAndGas,

            input.destinationGasLimit,

            input.message,

            input.zetaParams

        );

    }

// repos/protocol-contracts/contracts/zevm/ZRC20.sol#L256-L264
function withdraw(bytes memory to, uint256 amount) external override returns (bool) {

        (address gasZRC20, uint256 gasFee) = withdrawGasFee();

        if (!IZRC20(gasZRC20).transferFrom(msg.sender, FUNGIBLE_MODULE_ADDRESS, gasFee)) {

            revert GasFeeTransferFailed();

        }

        _burn(msg.sender, amount);

        emit Withdrawal(msg.sender, to, amount, gasFee, PROTOCOL_FLAT_FEE);

        return true;

    }
```
## [L-02] ## Impact
OpenZeppelin's safeApprove() function is deprecated and should not be used since it is affected by similar issues as approve() and can be exploited in frontrunning or sandwich attacks.
Losses to Users: The main linked impact is upon users who may be victim to front running. 
Users might be affected by economic shortfall while users' transactions get front-run by hackers, making them to pay more gas fees or get incorrect prices.
There is no way to safely decrease or increase the amount of the token as the owner can only be the contract.
The recipient can exploit this by pretending to be the sender and sending token as the sender to themselves, the attacker or malicious recipient. 
Also, if the sender updates the amount they are trying to send then the malicious recipient can withdraw both amounts from the sender's account.
Hencefotrh a front-running attack on the ERC20 token.
Hence the function can front-run by manipulating the approve function.
## Proof of Concept
**Manual POC**
**Approve Function**
The approve() function takes control of the current funds even if the customer already utilised it or didn't, hence no chance to elevate or reduce funds by a particular value holistically but only in the case that the token possessor is the smart contract, not the account address.
Which potentially leads to misconduct by a token recipient when the malicious recipient attempt the transfer of particular tokens out of the victim's account who is sending it.
At the same time, it is possible that the sender makes a decision to update the amount and calls an extra approve transaction, and then the recipient is able to see this transaction prior to it's being mined and has the ability to take the tokens from the two transactions, hence, illegally taking tokens from the pending and non pending transactions. 
This is called front-running attack within the ERC20 Approve method.
The method approve is possibly front-run by misusing the safeApprove method.
**Vulnerable safeApprove function**
```sol
// Line 108
        IERC20(inputToken).safeApprove(address(uniswapV3Router), inputTokenAmount);
```
## Tools Used
VS Code.
## Recommended Mitigation Steps
It is recommended to use safeIncreaseAllowance() and safeDecreaseAllowance instead of safeApprove().
Just utilise the approve method of the ERC/BEP standard to update the allowed value to 0 or from 0 (wait until the transaction is committed and approved).
Token possessor must confirm the first transaction has been mined from N to 0, that is, that the sender did not transfer a part of N allowed tokens prior to the first transaction was committed. 
These checks are doable utilising complicated blockchain explorers like `[Etherscan.io](https://etherscan.io/)`
Alternative ways to avoid the vulnerability is to approve token transfers solely to smart contracts that are verified code which do not contain business logic for carrying front running attacks also to accounts owned by users you know well.

## [L-03] 