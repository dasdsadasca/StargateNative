## User Flow 1: ERC20 Cross-Chain Send (Pool to Pool - Taxi Mode)

### Description

This flow describes the sequence of operations when a user sends ERC20 tokens from a `StargatePool` on a source chain to a recipient address on a destination chain, specifically utilizing the "Taxi" mode for the cross-chain message delivery. The Taxi mode generally implies a more direct and immediate attempt at token delivery.

**Initiation (Source Chain):**

1.  **User Interaction:** The user initiates the transfer by calling the `send(SendParam calldata _sendParam, MessagingFee calldata _fee, address _refundAddress)` function on the source `StargatePool` contract.
    *   `_sendParam`: A struct containing details like `dstEid` (destination chain's LayerZero Endpoint ID), `to` (recipient address on the destination chain, encoded as `bytes32`), `amountLD` (amount of tokens in local decimals to send), and `minAmountLD` (minimum amount expected on the destination after fees).
    *   `_fee`: A struct detailing the messaging fees, including `nativeFee` (for LayerZero gas on the source chain) and `lzTokenFee` (if LayerZero requires a separate token fee).
    *   `_refundAddress`: Address to refund any excess LayerZero native fees.
    *   Pre-requisite: The user must have previously approved the source `StargatePool` contract to spend at least `_sendParam.amountLD` of their ERC20 tokens.

2.  **`StargatePool.send()` -> `StargateBase.sendToken()`:** The call is typically handled by `sendToken()` in `StargateBase`, which `StargatePool` inherits.

3.  **`_inflowAndCharge()` (Internal `StargateBase` logic):**
    *   **`_inflow(amountLD, msg.sender)`:** This function, overridden in `StargatePool`, is called. It executes an `IERC20(token).transferFrom(msg.sender, address(this), amountLD)`, pulling the user's ERC20 tokens into the source `StargatePool`. The amount is converted to shared decimals (`amountInSD`).
    *   **`_buildFeeParams(...)`:** Gathers parameters required for the fee calculation, such as the current path deficit, `amountInSD`, etc.
    *   **`_chargeFee(feeParams)`:**
        *   Calls `IStargateFeeLib(feeLib).applyFee(feeParams)` on the designated fee library contract.
        *   This returns `amountOutSD`, which is the amount in shared decimals that will be credited/sent to the destination, after protocol fees or rewards are applied.
        *   The `treasuryFee` portion is updated within `StargateBase`.
        *   A check is performed: `amountOutSD` (converted to local decimals) must be `>= _sendParam.minAmountLD`. If not, the transaction reverts.
    *   **`paths[dstEid].decreaseCredit(amountOutSD)`:** The available credit on the path from the source chain to the `dstEid` is reduced by `amountOutSD`. This is an optimistic debit, assuming the Taxi will succeed.
    *   **`_postInflow(amountInSD, msg.sender)`:** This hook, overridden in `StargatePool`, is called. It increases the `paths[localEid()].credit` (credit of the local pool itself, reflecting more liquidity available for export from this pool) and also increases the `poolBalanceSD` (the actual ERC20 balance of the pool).

4.  **Mode Determination:** For a standard `send()` call without special `oftCmd` parameters, the system defaults to or determines that "Taxi" mode should be used for this transfer.

5.  **`_assertMessagingFee(fee)`:** This internal check in `StargateBase` ensures that `msg.value` provided by the user matches `_fee.nativeFee`. For ERC20 sends, `msg.value` only covers the native gas fee for the LayerZero message.

6.  **`_taxi(sendParam, amountOutSD, fee, refundAddress)` (Internal `StargateBase` logic):**
    *   **LZ Token Fee Payment:** If `_fee.lzTokenFee > 0`, an internal `_payLzToken()` function is called, which transfers the specified `lzTokenFee` amount of the LZ fee token (if applicable on the source chain) from the user to the LayerZero `endpoint` contract.
    *   **Call `ITokenMessaging.taxi()`:** The core cross-chain call is made:
        `ITokenMessaging(tokenMessaging).taxi{value: _fee.nativeFee}(taxiParams, _fee, _refundAddress)`.
        *   `taxiParams`: A struct containing all necessary information for the destination, including `sender` (source pool), `dstEid`, `receiver` (the `_sendParam.to` address), `amountSD` (which is `amountOutSD`), and any other relevant payload data like `composeMsg`.
        *   The `msg.value` forwards the `_fee.nativeFee` to pay for the LayerZero messaging.

7.  **Emit Event:** An `OFTSent` event is emitted by `StargateBase` logging the details of the outbound transfer.

**LayerZero Relaying:**

1.  **Message Relay:** The `ITokenMessaging` contract on the source chain interacts with the LayerZero `endpoint`. The LayerZero network of off-chain relayers and oracles observes this transaction and securely relays the message payload (containing `taxiParams`) from the source chain to the LayerZero `endpoint` on the destination chain. The destination `endpoint` then passes it to the destination `ITokenMessaging` contract.

**Reception (Destination Chain):**

1.  **`ITokenMessaging` -> `StargatePool.receiveTokenTaxi()`:** The `ITokenMessaging` contract on the destination chain, upon receiving the relayed message, calls `receiveTokenTaxi(Origin calldata _origin, bytes32 _guid, address _receiver, uint256 _amountSD, bytes calldata _composeMsg)` on the destination `StargatePool` contract. The `StargatePool` implements `ITokenMessagingHandler`.
    *   `_origin`: Contains `srcEid` (source chain's LayerZero Endpoint ID) and `srcAddress` (source `StargatePool` address).
    *   `_guid`: A globally unique identifier for the LayerZero message.
    *   `_receiver`: The recipient address on the destination chain (originally `_sendParam.to`).
    *   `_amountSD`: The amount of tokens in shared decimals to be delivered (this is `amountOutSD` from the source chain).
    *   `_composeMsg`: Additional message data if this transfer is part of a compose (multi-action) sequence.

2.  **`StargatePool.receiveTokenTaxi()` (Inherited from `StargateBase`):** The logic within `StargateBase` (and its overrides in `StargatePool`) processes the incoming tokens.

3.  **Amount Conversion:** `_amountSD` is converted to `amountLD` (local decimals for the destination pool's token).

4.  **`_outflow(amountLD, _receiver)` (Internal `StargateBase` logic, overridden in `StargatePool`):**
    *   This function is called to transfer `amountLD` of the corresponding ERC20 token from the destination `StargatePool`'s holdings to the `_receiver` address. It executes an `IERC20(token).transfer(_receiver, amountLD)`.

5.  **Successful Outflow:**
    *   If the `_outflow()` transfer is successful:
        *   **`_postOutflow(amountSD, _receiver)`:** This hook, overridden in `StargatePool`, is called. It decreases the `poolBalanceSD` of the destination pool.
        *   **Compose Handling:** If `_composeMsg` is not empty and valid, `endpoint.sendCompose(_composeMsg)` is called to continue a composed transaction sequence.
        *   **Emit Event:** An `OFTReceived` event is emitted by `StargateBase` logging the details of the inbound transfer.

6.  **Failed Outflow:**
    *   If `_outflow()` fails (e.g., the destination pool has insufficient balance of the specific token, or the token transfer itself reverts for some reason):
        *   The details of the undelivered transfer (including `_origin.srcEid`, `_receiver`, `_amountSD`, and a hash of the payload) are cached by storing them in the `unreceivedTokens` mapping in `StargateBase`.
        *   An `UnreceivedTokenCached` event is emitted.
        *   The end-user or a keeper service can later attempt to complete the transfer by calling `retryReceiveToken()` on the destination `StargatePool`, typically once liquidity conditions improve or the issue is resolved.

### Diagram

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant SrcPool as Source StargatePool
    participant FeeLib as Source IStargateFeeLib
    participant SrcTokenMsg as Source ITokenMessaging
    participant LZ as LayerZero Network
    participant DstTokenMsg as Destination ITokenMessaging
    participant DstPool as Destination StargatePool
    participant Recipient

    User->>SrcPool: send(sendParam, fee, refundAddress) [Approve ERC20 beforehand]
    activate SrcPool
    SrcPool->>SrcPool: _inflowAndCharge()
    activate SrcPool # self-call visual
    SrcPool->>SrcPool: _inflow() (ERC20.transferFrom user to SrcPool)
    SrcPool->>FeeLib: applyFee(feeParams)
    activate FeeLib
    FeeLib-->>SrcPool: amountOutSD (amount after fees)
    deactivate FeeLib
    SrcPool->>SrcPool: paths[dstEid].decreaseCredit(amountOutSD)
    SrcPool->>SrcPool: _postInflow() (update local credit & poolBalanceSD)
    deactivate SrcPool # self-call visual

    SrcPool->>SrcPool: _assertMessagingFee() (check msg.value == fee.nativeFee)

    SrcPool->>SrcPool: _taxi(sendParam, amountOutSD, ...)
    activate SrcPool # self-call visual
    opt if fee.lzTokenFee > 0
        SrcPool->>SrcPool: _payLzToken() (transfer LZ token fee from user)
    end
    SrcPool->>SrcTokenMsg: taxi{value: fee.nativeFee}(taxiParams, fee, refundAddress)
    deactivate SrcPool # self-call visual
    deactivate SrcPool

    activate SrcTokenMsg
    SrcTokenMsg->>LZ: Relay Message (taxiParams includes amountOutSD, recipient)
    deactivate SrcTokenMsg

    activate LZ
    LZ->>DstTokenMsg: Deliver Relayed Message
    deactivate LZ

    activate DstTokenMsg
    DstTokenMsg->>DstPool: receiveTokenTaxi(origin, guid, receiver, amountSD, composeMsg)
    deactivate DstTokenMsg

    activate DstPool
    DstPool->>DstPool: Convert amountSD to amountLD
    DstPool->>DstPool: _outflow(amountLD, receiver)
    activate DstPool # self-call visual

    alt Outflow Successful
        DstPool->>Recipient: ERC20.transfer(receiver, amountLD)
        DstPool->>DstPool: _postOutflow() (decrease poolBalanceSD)
        opt if composeMsg exists
            DstPool->>LZ: endpoint.sendCompose(composeMsg)
        end
        DstPool-->>User: (Event: OFTReceived)
    else Outflow Failed (e.g., insufficient pool balance)
        DstPool->>DstPool: cache in unreceivedTokens (srcEid, receiver, amountSD, payloadHash)
        DstPool-->>User: (Event: UnreceivedTokenCached)
    end
    deactivate DstPool # self-call visual
    deactivate DstPool

```
