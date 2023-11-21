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

// 
```