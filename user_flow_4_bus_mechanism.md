## User Flow 4 (Conceptual): Bus Mechanism for Token Transfer

### Description

The "Bus" mechanism in Stargate provides an alternative way to transfer tokens cross-chain, contrasting with the more direct "Taxi" mode. It's conceptually a two-step process on the source chain (riding and driving the bus), followed by reception on the destination chain. This mechanism is primarily orchestrated through interactions with a contract implementing the `ITokenMessaging` interface, which `StargateBase` (and thus its derivatives like `StargatePool`) calls.

**Key Characteristics of Bus Mode:**

*   **Batching:** A core feature. Multiple individual token transfer requests ("passengers") destined for the same chain can be bundled together by a "Planner" and sent in a single LayerZero message. This can lead to amortization of base LayerZero fees across many transfers, potentially making it more cost-effective for users.
*   **Planner Dependent:** The actual cross-chain dispatch of the batched transfers (`driveBus` step) relies on an external, authorized entity known as a "Planner" (or Keeper/Bot). This entity monitors pending passengers and decides when and how to dispatch a bus.
*   **Native Gas Payment for Dispatch:** The `driveBus` transaction, which sends the batched message, is typically paid for in the native currency of the source chain by the Planner.
*   **Asynchronous Nature:** Unlike Taxi mode which attempts immediate delivery, Bus mode is inherently more asynchronous due to the two-step process and reliance on a Planner.

**Conceptual Sequence of Operations:**

**Step 1: Riding the Bus (Source Chain - Initiated by User/Sender via Stargate contract)**

1.  **User Initiation:** A user performs an action on a Stargate contract (e.g., `StargatePool.send()` with specific parameters, or potentially a dedicated function) that signals their intent to use the "Bus" mode for their cross-chain token transfer. This includes providing the destination details (`dstEid`, recipient `to`), the amount of tokens (`amountLD`), and any fees associated with "riding" the bus.
2.  **Stargate Contract Logic:** The source `StargateBase`-derived contract processes the initial part of the transfer:
    *   Standard `_inflowAndCharge()` logic is executed: tokens are transferred from the user to the Stargate contract, fees are calculated and applied (protocol fees, etc.), and the `amountOutSD` (amount in shared decimals to be transferred) is determined. Crucially, path credits are typically decreased at this stage (`paths[dstEid].decreaseCredit(amountOutSD)`), earmarking these funds for the cross-chain transfer.
3.  **Call `_rideBus()` (Internal `StargateBase` logic):**
    *   This internal function is invoked to interact with the `ITokenMessaging` system.
    *   **LZ Token Fee Check:** It often reverts if `fee.lzTokenFee > 0`, as Bus mode is typically designed for native gas payments by the Planner for the `driveBus` step, not per-passenger LZ token fees.
    *   **Call `ITokenMessaging.rideBus()`:** `StargateBase` calls `ITokenMessaging(tokenMessaging).rideBus(rideBusParams)`.
        *   `rideBusParams`: Contains details like the `sender` (Stargate contract address), `dstEid`, `receiver` (final recipient on destination), `amountSD` (the `amountOutSD`), and a `nativeDrop` flag (indicating if native currency is being dropped on the destination).
    *   **`ITokenMessaging.rideBus()` Behavior:**
        *   The `ITokenMessaging` contract is expected to store these `rideBusParams` (the "passenger") in some form of temporary storage (e.g., a mapping or array associated with the `dstEid`).
        *   It returns a `MessagingReceipt` (which includes the `fare` charged by the Token Messaging system for this specific passenger to board the bus) and a `Ticket`.
    *   **Fare Handling:** `_rideBus()` in `StargateBase` compares the `providedFare` (from the user's initial `MessagingFee` input) with the `busFare` from the `MessagingReceipt`. It may refund any excess `providedFare` to the user or revert if the `providedFare` is insufficient to cover the `busFare`.
4.  **Ticket Generation:** A `Ticket` (often containing a unique `ticketId` and the `passengerBytes` – the encoded `rideBusParams`) is conceptually generated and available. The `passengerBytes` are what the Planner will later collect. The Stargate contract might store or emit this ticket information.

**Step 2: Driving the Bus (Source Chain - Initiated by Planner/Keeper)**

1.  **Planner Monitoring:** An off-chain "Planner" (an authorized account or automated bot) monitors the `ITokenMessaging` contract for pending passengers (stored `passengerBytes`) for various destinations.
2.  **Batch Assembly:** The Planner decides to dispatch a "bus" to a specific `dstEid`. It gathers multiple `passengerBytes` (from the tickets of passengers going to that `dstEid`).
3.  **Call `ITokenMessaging.driveBus()`:** The Planner calls `ITokenMessaging(tokenMessaging).driveBus{value: totalNativeFee}(dstEid, allPassengersBytes, ...otherParams)`.
    *   `value`: The Planner sends native currency along with this call to cover the LayerZero messaging fees for the entire batch of `allPassengersBytes`.
    *   `dstEid`: The destination chain.
    *   `allPassengersBytes`: A concatenation or array of `passengerBytes` for all passengers on this bus.
4.  **`ITokenMessaging.driveBus()` Behavior (Source Chain):**
    *   The `ITokenMessaging` contract processes `allPassengersBytes`, potentially validating tickets or passenger details.
    *   It then constructs a single LayerZero message containing the data for all passengers.
    *   It sends this batched message to the destination `dstEid` via the LayerZero `endpoint`.
    *   **Important:** The actual token value (`amountSD` for each passenger) remains held by the Stargate contract (e.g., `StargatePool`) on the source chain. The bus message itself doesn't move the tokens directly; it carries the *instructions* for the destination Stargate contract to release equivalent tokens, relying on the credit/path system.

**Step 3: Reception (Destination Chain - Handled by Stargate contract)**

1.  **LayerZero Relaying:** The LayerZero network relays the batched "driven bus" message from the source chain `ITokenMessaging` contract to its counterpart on the destination chain.
2.  **Destination `ITokenMessaging` Processing:** The destination `ITokenMessaging` contract receives the batched message. It then iterates through each passenger's data in the batch.
3.  **Call `StargateBase.receiveTokenBus()` (For each passenger):** For each passenger, the destination `ITokenMessaging` contract calls `receiveTokenBus(Origin calldata _origin, bytes32 _guid, uint256 _seatNumber, address _receiver, uint256 _amountSD)` on the destination `StargateBase`-derived contract (which implements `ITokenMessagingHandler`).
    *   `_origin`: Source chain and contract details.
    *   `_guid`: LayerZero global unique identifier for the *entire bus message*.
    *   `_seatNumber`: The index or identifier for this specific passenger within the bus batch. This, combined with `_guid`, uniquely identifies the individual transfer.
    *   `_receiver`: The final recipient address.
    *   `_amountSD`: The amount in shared decimals to be delivered to this passenger.
4.  **Destination `StargateBase.receiveTokenBus()` Execution:**
    *   Converts `_amountSD` to `amountLD` (local decimals).
    *   Calls `_outflow(_receiver, amountLD)` to attempt the transfer of `amountLD` of the local token from the Stargate pool to the `_receiver`.
    *   **Successful Outflow:** If `_outflow` succeeds, `_postOutflow()` is called (updating pool balance), and an `OFTReceived` event is emitted for this passenger.
    *   **Failed Outflow:** If `_outflow` fails (e.g., insufficient liquidity in the destination pool *despite* the credit system, or other token transfer issues), the transfer details are cached in `unreceivedTokens` using a key derived from `_guid` and `_seatNumber`. An `UnreceivedTokenCached` event is emitted. The user or a keeper can later attempt `retryReceiveToken()`.

This multi-step, batched approach allows for potentially lower average transaction costs for users willing to tolerate the delay introduced by waiting for a Planner to dispatch a bus.

### Diagram

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant SrcStargate as Source StargateContract
    participant SrcTokenMsg as Source ITokenMessaging
    participant Planner
    participant LZ as LayerZero Network
    participant DstTokenMsg as Destination ITokenMessaging
    participant DstStargate as Destination StargateContract
    participant FinalReceiver as Token Recipient

    box LightBlue Source Chain Operations
        participant User
        participant SrcStargate
        participant SrcTokenMsg
        participant Planner
    end
    box GhostWhite LayerZero Relay
        participant LZ
    end
    box LightGreen Destination Chain Operations
        participant DstTokenMsg
        participant DstStargate
        participant FinalReceiver
    end

    == Phase 1: Ride the Bus (User Initiates on Source Chain) ==
    User->>SrcStargate: send(paramsForBusMode, feeForRide, ...)
    activate SrcStargate
    SrcStargate->>SrcStargate: _inflowAndCharge() (tokens from user, protocol fees, decrease path credit)
    note right of SrcStargate: Tokens now held by SrcStargate, path credit reduced.
    SrcStargate->>SrcStargate: _rideBus()
    activate SrcStargate # self-call visual
    SrcStargate->>SrcTokenMsg: rideBus(rideBusParams: amountSD, dstEid, receiver, etc.)
    activate SrcTokenMsg
    SrcTokenMsg->>SrcTokenMsg: Store passenger details (e.g., by dstEid)
    SrcTokenMsg-->>SrcStargate: MessagingReceipt (busFare), Ticket (ticketId, passengerBytes)
    deactivate SrcTokenMsg
    SrcStargate->>SrcStargate: Assert providedFare vs busFare (refund/revert)
    deactivate SrcStargate # self-call visual
    SrcStargate-->>User: (Send Initiated / Ticket Info)
    deactivate SrcStargate

    == Phase 2: Drive the Bus (Planner Initiates on Source Chain) ==
    Planner->>SrcTokenMsg: driveBus(dstEid, collectedPassengerBytesArray) {value: totalNativeFeeForLZ}
    activate SrcTokenMsg
    SrcTokenMsg->>SrcTokenMsg: Process passengerBytes, prepare batched LZ payload
    SrcTokenMsg->>LZ: Relay Batched Message (containing data for all passengers on this bus)
    deactivate SrcTokenMsg

    == Phase 3: Reception (Destination Chain) ==
    activate LZ
    LZ->>DstTokenMsg: Deliver Batched Message from Source TokenMessaging
    deactivate LZ

    activate DstTokenMsg
    DstTokenMsg->>DstTokenMsg: Parse batched message, iterate through passengers
    loop For Each Passenger Data in Bus Message
        DstTokenMsg->>DstStargate: receiveTokenBus(origin, guid, seatNumber, receiver, amountSD)
        activate DstStargate
        DstStargate->>DstStargate: Convert amountSD to amountLD
        DstStargate->>DstStargate: _outflow(receiver, amountLD)
        activate DstStargate # self-call visual for outflow
        alt Outflow Successful
            DstStargate->>FinalReceiver: ERC20/Native Token transfer(receiver, amountLD)
            DstStargate->>DstStargate: _postOutflow() (decrease poolBalanceSD)
            note left of DstStargate: OFTReceived event emitted
        else Outflow Failed (e.g., temporary lack of liquidity)
            DstStargate->>DstStargate: cache in unreceivedTokens (using guid, seatNumber)
            note left of DstStargate: UnreceivedTokenCached event emitted
        end
        deactivate DstStargate # self-call visual for outflow
        deactivate DstStargate
    end
    deactivate DstTokenMsg

```
