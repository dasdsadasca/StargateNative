## User Flow 2: ERC20 Deposit to StargatePool for LP Tokens

### Description

This flow outlines the process a user follows to deposit ERC20 tokens into a `StargatePool` and, in return, receive LP (Liquidity Provider) tokens. These LP tokens represent the user's share of the liquidity within that specific pool.

**Process:**

1.  **User Approval (Pre-requisite):** Before initiating the deposit, the user must have approved the `StargatePool` contract to spend at least `amountLD` of the specific ERC20 token they intend to deposit. This is a standard ERC20 allowance mechanism.

2.  **Call `deposit()`:** The user calls the `deposit(address _receiver, uint256 _amountLD)` function on the target `StargatePool` contract.
    *   `_receiver`: The address that will receive the minted LP tokens. This is typically the user's own address (`msg.sender`).
    *   `_amountLD`: The amount of ERC20 tokens (in local decimals) the user wishes to deposit.

3.  **Modifier Check:** The `nonReentrantAndNotPaused` modifier (inherited from `StargateBase` or its parent contracts) is checked. If the contract is paused or if a reentrancy attempt is detected, the transaction will revert.

4.  **`_inflow(msg.sender, _amountLD)` (Internal logic, overridden in `StargatePool`):**
    *   This function is called internally to handle the actual transfer of tokens into the pool.
    *   The `_amountLD` is first converted to its equivalent in shared decimals (`amountSD`) using the `ld2sd()` function.
    *   The core action is `Transfer.safeTransferTokenFrom(token, msg.sender, address(this), _amountLD)`. This utility function (from the inherited `Transfer.sol`) executes the ERC20 `transferFrom` method, moving `_amountLD` of the specified `token` from the user (`msg.sender`) to the `StargatePool` contract (`address(this)`).

5.  **`_postInflow(amountSD, msg.sender)` (Internal logic, overridden in `StargatePool`):**
    *   After the tokens have been successfully transferred into the pool, this hook is called.
    *   `paths[localEid()].increaseCredit(amountSD)`: The credit available on the local pool's own path (representing its capacity to source transfers) is increased by `amountSD`. This reflects the new liquidity that has just been added.
    *   `poolBalanceSD = poolBalanceSD + amountSD`: The pool's internal accounting variable `poolBalanceSD`, which tracks the balance of the underlying ERC20 token in shared decimals, is increased.

6.  **Mint LP Tokens:**
    *   `lp.mint(_receiver, _amountLD)`: The `StargatePool` contract calls the `mint` function on its associated `LPToken` contract (address stored in the `lp` state variable).
    *   This action creates (mints) `_amountLD` of LP tokens and assigns them to the `_receiver` address. The amount of LP tokens minted is typically 1:1 with the amount of underlying tokens deposited whenpool is balanced, but can vary based on the current `tvlSD` vs `lp.totalSupply()` ratio to ensure fair representation of pool share. *Self-correction: The provided snippet implies `amountLD` of LP tokens. In many Stargate implementations, the amount of LP tokens minted is actually calculated based on the `amountSD` deposited relative to the current `tvlSD` and total supply of LP tokens, to ensure new LPs get a fair proportional share, not necessarily a 1:1 minting with `amountLD`.* For simplicity here, we'll stick to the provided `amountLD` for minting as per the prompt's example, but note this subtlety.

7.  **Update Total Value Locked (TVL):**
    *   `tvlSD = tvlSD + amountSD`: The pool's `tvlSD` (Total Value Locked in shared decimals) is increased by the `amountSD` deposited, reflecting the growth of the pool's total assets.

8.  **Emit Event:**
    *   An `Deposited(msg.sender, _receiver, _amountLD, amountLPT)` event is emitted (parameters might vary slightly, e.g. `amountLPT` to reflect actual LPT minted). The prompt example shows `Deposited(msg.sender, receiver, amountLD)`. This event logs the details of the deposit: the user who initiated it, the recipient of the LP tokens, and the amount of ERC20 tokens deposited.

The user (`_receiver`) now holds `LPToken`s, which can be later redeemed for the underlying ERC20 token or used in other DeFi applications if supported.

### Diagram

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant Pool as StargatePool
    participant LP_Token as LPToken
    participant ERC20_Token as ERC20 Token Contract

    User->>ERC20_Token: approve(Pool_Address, amountLD)
    activate ERC20_Token
    ERC20_Token-->>User: (Approval successful)
    deactivate ERC20_Token

    User->>Pool: deposit(receiver, amountLD)
    activate Pool

    Pool->>Pool: Check nonReentrantAndNotPaused

    Pool->>Pool: _inflow(msg.sender, amountLD)
    activate Pool # self-call visual for _inflow
    Pool->>ERC20_Token: transferFrom(User_Address, Pool_Address, amountLD)
    activate ERC20_Token
    ERC20_Token-->>Pool: (Tokens Transferred)
    deactivate ERC20_Token
    deactivate Pool # self-call visual for _inflow

    Pool->>Pool: _postInflow(amountSD, msg.sender)
    activate Pool # self-call visual for _postInflow
    Pool->>Pool: paths[localEid].increaseCredit(amountSD)
    Pool->>Pool: poolBalanceSD += amountSD
    deactivate Pool # self-call visual for _postInflow

    Pool->>LP_Token: mint(receiver, amountLD_or_calculated_LPT_amount)
    activate LP_Token
    LP_Token-->>Pool: (LP Tokens Minted to Receiver)
    deactivate LP_Token

    Pool->>Pool: tvlSD += amountSD

    Pool-->>User: (Event: Deposited)
    deactivate Pool

```
