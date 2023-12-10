# BlurExchangeV2

#### takeAskSingle()

```solidity 
/**
     * @notice Wrapper of _takeAskSingle that verifies an oracle signature of the calldata before executing
     * @param inputs Inputs for _takeAskSingle
     * @param oracleSignature Oracle signature of inputs
     */
    function takeAskSingle(
        TakeAskSingle memory inputs,
        bytes calldata oracleSignature
    )
        public
        payable
        nonReentrant
        verifyOracleSignature(_hashCalldata(msg.sender), oracleSignature)
    {
        _takeAskSingle(
            inputs.order,
            inputs.exchange,
            inputs.takerFee,
            inputs.signature,
            inputs.tokenRecipient
        );
    }
```

_takeAskSingle() is the function that actually executes the trade. It is called by takeAskSingle().

```solidity
 /**
     * @notice Take a single ask
     * @param order Order of listing to fulfill
     * @param exchange Exchange struct indicating the listing to take and the parameters to match it with
     * @param takerFee Taker fee to be taken
     * @param signature Order signature
     * @param tokenRecipient Address to receive the token transfer
     */
    function _takeAskSingle(
        Order memory order,
        Exchange memory exchange,
        FeeRate memory takerFee,
        bytes memory signature,
        address tokenRecipient
    ) internal {
        Fees memory fees = Fees(protocolFee, takerFee);
        Listing memory listing = exchange.listing;
        uint256 takerAmount = exchange.taker.amount;

        /* Validate the order and listing, revert if not. */
        if (
            !_validateOrderAndListing(
                order,
                OrderType.ASK,
                exchange,
                signature,
                fees
            )
        ) {
            revert InvalidOrder();
        }

        /* Create single execution batch and insert the transfer. */
        bytes memory executionBatch = _initializeSingleExecution(
            order,
            OrderType.ASK,
            listing.tokenId,
            takerAmount,
            tokenRecipient
        );

        /* Set the fulfillment of the order. */
        unchecked {
            amountTaken[order.trader][bytes32(order.salt)][
                listing.index
            ] += takerAmount;
        }

        /* Execute the token transfers, revert if not successful. */
        {
            bool[] memory successfulTransfers = _executeNonfungibleTransfers(
                executionBatch,
                1
            );
            if (!successfulTransfers[0]) {
                revert TokenTransferFailed();
            }
        }

        (
            uint256 totalPrice,
            uint256 protocolFeeAmount,
            uint256 makerFeeAmount,
            uint256 takerFeeAmount
        ) = _computeFees(listing.price, takerAmount, order.makerFee, fees);

        /* If there are insufficient funds to cover the price with the fees, revert. */
        unchecked {
            if (address(this).balance < totalPrice + takerFeeAmount) {
                revert InsufficientFunds();
            }
        }

        /* Execute ETH transfers. */
        _transferETH(fees.protocolFee.recipient, protocolFeeAmount);
        _transferETH(fees.takerFee.recipient, takerFeeAmount);
        _transferETH(order.makerFee.recipient, makerFeeAmount);
        unchecked {
            _transferETH(
                order.trader,
                totalPrice - makerFeeAmount - protocolFeeAmount
            );
        }

        _emitExecutionEvent(
            executionBatch,
            order,
            listing.index,
            totalPrice,
            fees,
            OrderType.ASK
        );

        /* Return dust. */
        _transferETH(msg.sender, address(this).balance);
    }
```