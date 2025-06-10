## User Flow 3: LP Token Redemption from StargatePool for ERC20 Tokens

### Description

This flow describes the sequence of operations when a user redeems their LP (Liquidity Provider) tokens from a `StargatePool` to receive the underlying ERC20 tokens back. This process effectively withdraws the user's liquidity from the pool.

**Process:**

1.  **User Call `redeem()`:** The user initiates the redemption by calling the `redeem(uint256 _amountLPT, address _to)` function on the `StargatePool` contract.
    *   `_amountLPT`: The amount of LP tokens the user wishes to redeem.
    *   `_to`: The address that will receive the underlying ERC20 tokens. This is typically the user's own address (`msg.sender`).
    *   Pre-requisite: The `msg.sender` must own at least `_amountLPT` of the pool's LP tokens. For `burnFrom(account, amount)`, if `account` is `msg.sender`, direct ownership is checked. If `account` were different from `msg.sender`, then `msg.sender` would need an allowance from `account` to burn its LP tokens.

2.  **Modifier Check:** The `nonReentrantAndNotPaused` modifier (inherited) is checked to prevent reentrancy attacks and ensure the contract is not paused.

3.  **Amount Conversion (LP to Shared Decimals):**
    *   The input `_amountLPT` (which represents the amount of LP tokens, often having the same decimals as the underlying local token) is converted to shared decimals (`amountSD`) using `_ld2sd()`. This `amountSD` will represent the proportional claim on the underlying asset in shared decimal precision. *Self-correction: The variable name `amountLD` in the prompt for LP tokens might be confusing. Typically, `StargatePool.redeem` takes the amount of LP tokens directly. The conversion to `amountSD` is to calculate the equivalent value of underlying tokens. The amount of LP tokens to burn is `_amountLPT`.* We will assume `_amountLPT` is the quantity of LP tokens. The corresponding value of underlying tokens is what gets converted via `ld2sd` and `sd2ld`.

4.  **Decrease Local Path Credit:**
    *   `paths[localEid()].decreaseCredit(amountSD)`: The credit for the local pool's own path (its Endpoint ID) is decreased by `amountSD`. This reflects that the pool's capacity to source outgoing transfers (or its own available liquidity) is being reduced by this redemption.

5.  **Amount Re-Conversion (Shared to Local Decimals for Underlying Token):**
    *   `amountUnderlyingLD = _sd2ld(amountSD)`: The `amountSD` (representing the value of underlying tokens to be redeemed) is converted back to local decimals for the underlying ERC20 token. This step "de-dusts" the amount, ensuring it aligns with the actual decimal precision of the token being transferred out.

6.  **Burn LP Tokens:**
    *   `lp.burnFrom(msg.sender, _amountLPT)`: The `StargatePool` contract calls the `burnFrom` function on its associated `LPToken` contract (address stored in `lp`).
    *   This action burns `_amountLPT` of LP tokens directly from the `msg.sender`'s balance. The `LPToken` contract's `burnFrom` will internally call `_burn(msg.sender, _amountLPT)`, which typically checks if `msg.sender` has sufficient LP balance.

7.  **Update Total Value Locked (TVL):**
    *   `tvlSD = tvlSD - amountSD`: The pool's `tvlSD` (Total Value Locked in shared decimals) is decreased by `amountSD`, reflecting the withdrawal of assets from the pool.

8.  **Safe Outflow of Underlying ERC20 Tokens (`_safeOutflow`):**
    *   The internal function `_safeOutflow(_to, amountUnderlyingLD)` is called.
    *   **`_outflow(_to, amountUnderlyingLD)` (Overridden in `StargatePool`):** This hook is executed. It calls `Transfer.transferToken(token, _to, amountUnderlyingLD)`, which performs the actual transfer of `amountUnderlyingLD` of the underlying ERC20 `token` from the `StargatePool` contract to the specified `_to` address.
    *   **Error Check:** If the `_outflow` call (specifically, the token transfer) fails (e.g., the token contract reverts, though `safeTransferToken` usually handles non-reverting failures too), `_safeOutflow` will cause the transaction to revert, often with an error like `Stargate_OutflowFailed`.

9.  **Post-Outflow Updates (`_postOutflow`):**
    *   If the outflow was successful, `_postOutflow(amountSD, _to)` (overridden in `StargatePool`) is called.
    *   `poolBalanceSD = poolBalanceSD - amountSD`: The pool's internal accounting variable `poolBalanceSD` (tracking the balance of the underlying ERC20 token in shared decimals) is decreased.

10. **Emit Event:**
    *   An `Redeemed(msg.sender, _to, _amountLPT, amountUnderlyingLD)` event is emitted (parameters might vary slightly based on exact implementation). The prompt example shows `Redeemed(msg.sender, receiver, amountLD)`. This event logs the details of the redemption: the user who initiated it, the recipient of the underlying tokens, the amount of LP tokens burned, and the amount of underlying ERC20 tokens received.

The `_to` address now has the redeemed ERC20 tokens, and the user's LP token balance has decreased.

### Diagram

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant Pool as StargatePool
    participant LP_Token as LPToken Contract
    participant Underlying_ERC20 as Underlying ERC20 Token Contract

    User->>Pool: redeem(amountLPT, receiverAddress)
    activate Pool

    Pool->>Pool: Check nonReentrantAndNotPaused
    Pool->>Pool: amountSD = _ld2sd(amountLPT) (Calculate value of LPT in SD)
    Pool->>Pool: paths[localEid].decreaseCredit(amountSD)
    Pool->>Pool: amountUnderlyingLD = _sd2ld(amountSD) (Convert back to token's local decimals)

    Pool->>LP_Token: burnFrom(msg.sender, amountLPT)
    activate LP_Token
    LP_Token->>LP_Token: _burn(msg.sender, amountLPT) (Internal burn logic)
    LP_Token-->>Pool: (LP Tokens Burned from User)
    deactivate LP_Token

    Pool->>Pool: tvlSD -= amountSD

    Pool->>Pool: _safeOutflow(receiverAddress, amountUnderlyingLD)
    activate Pool # self-call visual for _safeOutflow
    Pool->>Pool: _outflow(receiverAddress, amountUnderlyingLD)
    activate Pool # self-call visual for _outflow
    Pool->>Underlying_ERC20: transfer(receiverAddress, amountUnderlyingLD) [Pool transfers to Receiver]
    activate Underlying_ERC20
    Underlying_ERC20-->>Pool: (Transfer Success/Failure)
    deactivate Underlying_ERC20
    deactivate Pool # self-call visual for _outflow
    %% Check if outflow failed, revert if so (handled by _safeOutflow)
    deactivate Pool # self-call visual for _safeOutflow

    Pool->>Pool: _postOutflow(amountSD, receiverAddress)
    activate Pool # self-call visual for _postOutflow
    Pool->>Pool: poolBalanceSD -= amountSD
    deactivate Pool # self-call visual for _postOutflow

    Pool-->>User: (Event: Redeemed)
    deactivate Pool

```
