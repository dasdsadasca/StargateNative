# StargatePoolNative.sol - Advanced Calculation and Logic Audit Notes (Finalized)

This document provides a finalized deep dive into `msg.value` handling, calculations, conversions, and logic flows specific to the overrides within a conceptual `StargatePoolNative.sol`.

## `_inflow(address /*_from*/, uint256 _amountLD)` Override

*   **Logic:** `return _ld2sd(_amountLD);`
*   **Context:** This function is called by the `deposit()` function (inherited from `StargatePool`). Crucially, before `_inflow` is called, `deposit()` first invokes `_assertMsgValue(_amountLD)`.
*   **`_assertMsgValue` in `StargatePoolNative` (Overridden):** This function checks two conditions:
    1.  `_amountLD == msg.value`: Ensures the amount of native currency sent with the transaction (`msg.value`) exactly matches the `_amountLD` parameter specified by the user for the deposit.
    2.  `_amountLD == _sd2ld(_ld2sd(_amountLD))`: This is a "de-dusting" or "canonical amount" check. It ensures that `_amountLD` is an amount that can be perfectly converted to shared decimals (SD) and then back to local decimals (LD) without any loss of value due to integer truncation in the `_ld2sd` conversion.
*   **Conclusion on `_inflow`:** The `_inflow` override itself is simple because the critical validation of the native asset transfer (i.e., `msg.value`) is handled by `_assertMsgValue`. `_inflow` correctly converts this validated and de-dusted `_amountLD` into its shared decimal representation (`amountSD`) for further use in LP minting and accounting. This separation of concerns is logical and secure. No direct value manipulation occurs within `_inflow` for native assets.

## `_outflow(address payable _to, uint256 _amountLD)` Override

*   **Logic:** `bool success = Transfer.transferNative(_to, _amountLD, true); // gasLimited = true`
    `return success;`
*   **Context:** Called by `StargateBase` functions like `_receiveTokenTaxi` and `_receiveTokenBus` (within `_safeReceive`) when native assets arrive from another chain and need to be paid out to the recipient.
*   **Gas Limitation:** `gasLimited = true` typically means the native transfer (`_to.call{value: _amountLD}("")`) is given a fixed, small gas stipend (e.g., 2300 gas via `transferGasLimit` in `Transfer.sol`).
*   **Implications for Contract Recipients:** If `_to` is a smart contract with a `receive()` or `fallback()` payable function that consumes more than this limited gas stipend, the native asset transfer will fail (the `call` will return `false`).
    *   This leads to `success = false`, and the calling function in `StargateBase` (e.g., `_safeReceive`) would typically revert (e.g., with `Stargate_OutflowFailed`), causing the incoming tokens to be cached in `unreceivedTokens`.
    *   This is not a Stargate vulnerability per se but a known behavior of EVM/Solidity that can affect usability for recipients with complex fallback logic. The funds are not lost but require a `retryReceiveToken` call.
*   **Return Value:** `_outflow` correctly returns the success status of the transfer.

## `_safeOutflow(address payable _to, uint256 _amountLD)` Override

*   **Logic:** `Transfer.transferNative(_to, _amountLD, false); // gasLimited = false`
*   **Context:** Called by `StargatePool.redeem()` and `StargateBase.retryReceiveToken()`. These are user-initiated actions for withdrawing their own funds or retrying a failed receipt.
*   **Gas Forwarding:** `gasLimited = false` means more (potentially all available) gas is forwarded with the native asset transfer.
*   **Reentrancy Risk:** This allows the recipient contract `_to` (if it's a contract) to execute with more gas, increasing its capability to perform a reentrancy attack. The `nonReentrantAndNotPaused` modifier on the public entry points (`redeem`, `retryReceiveToken`) is the primary defense against this. Its robustness is critical.
*   **Failure Handling:** `Transfer.transferNative` with `gasLimited = false` will typically revert the entire transaction if the underlying `call` to `_to` reverts (e.g., if `_to` explicitly reverts). This is appropriate.

## `_assertMessagingFee(MessagingFee memory _fee, uint256 _amountInLD)` Override

*   **Logic (Critical Calculation):**
    1.  `uint256 expectedMsgValue = _fee.nativeFee + _amountInLD;` (For native sends, `_amountInLD` is part of `msg.value`).
    2.  `if (msg.value < expectedMsgValue) revert Stargate_InvalidAmount();`
    3.  `if (msg.value > expectedMsgValue) { _fee.nativeFee = msg.value - _amountInLD; }`
*   **Impact of Excess `msg.value`:** If `msg.value` is greater than `expectedMsgValue`, the excess amount (`msg.value - expectedMsgValue`) is effectively added to `_fee.nativeFee`. This means the entire `msg.value` beyond `_amountInLD` is treated as the `nativeFee` component.
*   **Fund Loss Vector:** This modified (potentially inflated) `_fee.nativeFee` is then used as the `value` in the subsequent call to the `ITokenMessaging` contract (e.g., `ITokenMessaging(tokenMessaging).taxi{value: _messagingFee.nativeFee}(...)`).
    *   If the `ITokenMessaging` contract and the underlying LayerZero system do not have a mechanism to refund any overpayment of native fees for message relaying, the user's excess ETH (the amount by which `msg.value` exceeded `expectedMsgValue`) is effectively lost to that external layer.
    *   This is not a case where Stargate contracts directly appropriate the funds, but the design facilitates this potential loss for the user if they overpay `msg.value`.
*   **Severity:** Medium. While not a direct theft by Stargate, it can lead to user fund loss.
*   **Recommendation:**
    *   The most secure approach would be to require `msg.value == expectedMsgValue` and revert otherwise.
    *   Alternatively, if flexibility for overpayment is desired (e.g., for potential future gas auctioning on the messaging layer), a refund mechanism for the calculated excess (`msg.value - expectedMsgValue`) should be implemented *within `_assertMessagingFee`* to send the excess back to `msg.sender` or the specified `_refundAddress`. The current logic of silently adding it to `_fee.nativeFee` is not ideal.

## `_assertMsgValue(uint256 _amountLD)` Override

*   **Logic:** `if (_amountLD != msg.value || _amountLD != _sd2ld(_ld2sd(_amountLD))) revert Stargate_InvalidAmount();`
*   **Strictness and Purpose:**
    1.  `_amountLD == msg.value`: This is a crucial check ensuring that the amount of native currency sent with the transaction (`msg.value`) precisely matches the `_amountLD` parameter that the user intends to deposit. This prevents discrepancies and potential over/under payments for the deposit itself.
    2.  `_amountLD == _sd2ld(_ld2sd(_amountLD))`: This "de-dusting" or "canonical amount" check ensures that `_amountLD` is an amount that can be converted to Shared Decimals (SD) via `_ld2sd` (which involves flooring) and then converted back to Local Decimals (LD) via `_sd2ld` without any change in value. Effectively, it means `_amountLD` must be a perfect multiple of `convertRate` (when considering the smallest atomic units of the native asset).
*   **Impact on Users:** Users attempting to deposit very small native asset amounts that are not perfect multiples for SD conversion (i.e., would result in dust if not for this check) will have their transactions reverted. For example, if `sharedDecimals=6` and `localDecimals=18` (ETH), then `convertRate = 10^12`. A user must deposit an amount in wei that is a multiple of `10^12` for it to be valid.
*   **Security Implication:** This is a strong integrity check that prevents several issues:
    *   It stops "dust deposits" where a user sends native assets, but so little that it rounds down to 0 SD, leading to 0 LP tokens being minted (as seen in `StargatePool` analysis where dust can become a pool gain). Here, such deposits are outright rejected.
    *   It ensures consistency in value accounting between LD and SD for all processed native asset deposits.
*   **Conclusion:** This is a robust check that enhances financial integrity, albeit with a strict constraint on deposit amounts.

## `_thisBalance()` and `_plannerFee()` Overrides

*   **`_thisBalance()`:** Returns `address(this).balance;`. Correctly provides the total native currency balance of the contract.
*   **`_plannerFee()`:** Calculates surplus native currency: `return _thisBalance() - _sd2ld(poolBalanceSD + accTreasuryFee);`
    *   `poolBalanceSD` (pool's accounted liquidity in SD) and `accTreasuryFee` (protocol's fees in SD, from `StargateBase`) are summed.
    *   This sum (total accounted SD value) is converted to LD using `_sd2ld`.
    *   This LD value is subtracted from the contract's actual total native balance (`_thisBalance()`).
    *   **Logic:** This correctly identifies any native currency in the contract that is not already accounted for as part of the pool's active liquidity or collected treasury fees. This surplus is considered the planner fee.
    *   **Precision:** `_sd2ld(poolBalanceSD + accTreasuryFee)` can result in minor differences if `poolBalanceSD + accTreasuryFee` is not a perfect multiple for `convertRate`, due to the scaling. This might leave tiny, unclaimable dust amounts or make tiny booked amounts part of the planner fee. This is minor.
    *   **Integrity:** Relies on `poolBalanceSD` and `accTreasuryFee` being accurately maintained.

## `fallback() external payable onlyOwner {}` and `receive() external payable onlyOwner {}`

*   **Access Control:** `onlyOwner`.
*   **Functionality:** Allows only the `owner` to send ETH directly to the contract address without calling a specific function.
*   **Impact:**
    *   Prevents accidental locking of ETH sent by arbitrary users.
    *   Any ETH sent by the owner via these functions increases `address(this).balance`. This ETH is not automatically part of `poolBalanceSD` or `accTreasuryFee`. Consequently, it will be included in the amount calculated by `_plannerFee()` and can be withdrawn by the planner.
    *   This is an intentional design choice allowing the owner to fund planner operations or subsidize the pool in a way that becomes planner-claimable.

## Interaction with Inherited Logic (`StargatePool` -> `StargateBase`)

*   **`deposit()`:** The inherited `deposit` flow from `StargatePool` correctly utilizes the overridden `_assertMsgValue` (for strict `msg.value` checking and de-dusting) and `_inflow` (which just converts the validated `_amountLD` to SD). LP token minting (using `_sd2ld` of the `amountDepositedSD`) remains consistent with the (de-dusted) native asset value deposited.
*   **`redeem()`:** The inherited `redeem` flow correctly uses the overridden `_safeOutflow` (which calls `Transfer.transferNative` with full gas) to return native assets. The amount of native assets returned corresponds to the LP tokens burned, consistent with the deposit value.
*   **`sendToken()` (from `StargateBase`, called by `send()`):** Correctly uses the overridden `_assertMessagingFee` (which handles `msg.value` for native amount + fee) and `_taxi` or `_rideBus`.

## Summary of Key Calculation/Logic Findings for `StargatePoolNative.sol`

1.  **`_assertMessagingFee` - Potential User Fund Loss (Medium Risk):** The most significant concern. If a user overpays `msg.value` when sending native tokens cross-chain, the excess is added to `_fee.nativeFee` and passed to `ITokenMessaging`. If this layer doesn't refund, the user loses the excess. Stargate should ideally revert on exact `msg.value` mismatch or implement a refund for the excess.
2.  **`_assertMsgValue` - Strict De-Dusting (Design Choice):** Enforces deposits to be exact multiples of `convertRate` (effectively). This is good for accounting integrity but strict for users with very small, non-convertible amounts.
3.  **Gas in `_outflow` (Low Risk/Usability):** Limited gas (2300) for `receiveTokenBus/Taxi` payouts can cause issues for complex contract recipients, leading to funds being cached.
4.  **Owner ETH Contributions & Planner Fee:** ETH sent by owner via `fallback/receive` becomes part of the balance claimable by the planner. This is likely intentional.
5.  **Overall Consistency:** The overrides effectively adapt the ERC20-based logic from parent contracts to handle native assets, with `msg.value` being the primary mechanism for value transfer into the contract. LP token calculations remain consistent with the de-dusted value of native assets.

The logic for native asset handling appears robust, with the main point of concern being the handling of excess `msg.value` in `_assertMessagingFee`.The `stargate_pool_native_advanced_audit_notes.md` file has been successfully created with the finalized deep-dive analysis for `StargatePoolNative.sol`.

This iteration focused on:
1.  **`msg.value` Handling:**
    *   Confirmed `_inflow()` correctly relies on `_assertMsgValue` for validation.
    *   Detailed the critical concern with `_assertMessagingFee()` where excess `msg.value` is added to `_fee.nativeFee`, potentially leading to user fund loss if the external `ITokenMessaging` layer doesn't refund. Recommended reverting on mismatch or implementing a direct refund.
    *   Analyzed the strictness of `_assertMsgValue()` (exact value and de-dusting checks) as a strong integrity measure but a usability constraint for non-convertible small amounts.
2.  **Native Asset Transfers (`_outflow`, `_safeOutflow`):**
    *   Noted implications of limited gas in `_outflow` for contract recipients.
    *   Confirmed `_safeOutflow` forwards more gas, appropriate for user-initiated redemptions/retries, relying on the `nonReentrantAndNotPaused` guard.
3.  **Fee Calculations (`_plannerFee`):**
    *   Verified the `_plannerFee` calculation correctly accounts for `poolBalanceSD` and `treasuryFee` against the contract's total native balance.
    *   Noted how owner's direct ETH sends via `fallback/receive` contribute to planner-claimable fees.
4.  **Interaction with Inherited Logic:**
    *   Confirmed that inherited functions like `deposit` and `redeem` correctly use the native-specific overrides, maintaining consistency for LP token operations relative to the (de-dusted) native asset value.

This completes the deep dive for `StargatePoolNative.sol`. All analysis and documentation tasks as per the original plan have now been completed and refined.

The next step is to submit the report for this final task, concluding the entire engagement.
