# 🧾 Covenant Audit Findings

This document contains my findings from the **Covenant** audit contest on CodeArena.  
Each issue includes context, impact, and proposed mitigations.

**Platform:** Code4rena  
**Contest:** [Covenant](https://code4rena.com/audits/2025-10-covenant)  
**Dates:** Oct 22, 2025 - Nov 03, 2025   
**Role:** Warden  
**Findings:** 5 Lows

## Summary Table

| ID | Title | Severity |
|----|-------|----------|
| Low-01 | Missing Validation for Self-Transfer in `redeem()` | Low |
| Low-02 | Unnecessary State Write and Event Emission in `setDefaultFee` | Low |
| Low-03 | Zero Address Allowed for Pause Operator Disables Emergency Controls | Low |
| Low-04 | Legacy Pause Operator Retains Control After Ownership Transfer | Low |
| Low-05 | Protocol Fees Become Inaccessible While Market Is Paused | Low |

---

---

# [01] Missing Validation for Contract Address in `redeem()`

**Contract Name:** `Covenant.sol`  
**Function Name:** `redeem`

---

### 🧠 Description
The `redeem()` function allows users to specify any address as the redemption recipient (`redeemParams.to`).  
However, the function does not prevent sending redeemed base tokens to the Covenant contract itself (`address(this)`).

If a user sets `to = address(this)`, the function transfers tokens to the contract while also decreasing `marketState.baseSupply`.  
This creates an accounting mismatch — the internal supply is reduced, but the actual ERC20 balance inside the contract does not decrease.  
Over time, this can desynchronize internal accounting, lead to locked funds, or disrupt later operations that depend on accurate balances.

---

### 🧰 Mitigation
Add a validation to prevent self-transfer before the base token transfer occurs:

```solidity
if (redeemParams.to == address(this)) revert Errors.E_Unauthorized();
IERC20(mp.baseToken).safeTransfer(redeemParams.to, amountOut);
```


# [02] Unnecessary State Write and Event Emission in `setDefaultFee`

**Contract Name:** `Covenant.sol`  
**Function Name:** `setDefaultFee`

---

### 🧠 Description
The `setDefaultFee()` function updates `_defaultProtocolFee` and emits an event even when the new fee value is identical to the current one.

```solidity
_defaultProtocolFee = newFee;
emit Events.UpdateDefaultProtocolFee(oldFee, newFee);

When newFee == _defaultProtocolFee, this results in:
```
A redundant storage write (SSTORE) that consumes unnecessary gas.

A redundant event emission, generating noise in off-chain logs.

This has no functional impact but introduces avoidable inefficiency.

🧰 Mitigation

Add a conditional check to skip the write and event emission if the fee remains unchanged:

```solidity
if (newFee == _defaultProtocolFee) return;

uint32 oldFee = _defaultProtocolFee;
_defaultProtocolFee = newFee;
emit Events.UpdateDefaultProtocolFee(oldFee, newFee);
```

💥 Impact

Minor gas overhead and unnecessary log entries without affecting functionality.

# [03] Missing Validation for Zero Pause Address Disables Emergency Controls

**Contract Name:** `Covenant.sol`  
**Functions:** `setDefaultPauseAddress`, `setMarketPauseAddress`

---

### 🧠 Description
The protocol’s pause control addresses can be updated through `setDefaultPauseAddress()` and `setMarketPauseAddress()`.  
However, both functions allow assigning the zero address (`address(0)`) without any validation.

If a zero address is ever set — whether accidentally or through misconfiguration — the pause mechanism becomes **irreversibly disabled**, since `msg.sender` can never be `address(0)`.  
As a result, future attempts to pause markets will always revert, leaving the system unable to freeze operations in emergencies.

---

### ⚙️ Vulnerable Code
```solidity
function setDefaultPauseAddress(address newPauseAddress) external onlyOwner {
    _defaultPauseAddress = newPauseAddress; // ❌ Missing zero address check
    emit Events.UpdateDefaultPauseAddress(oldPauseAddress, newPauseAddress);
}

function setMarketPauseAddress(MarketId marketId, address newPauseAddress) external {
    if (_msgSender() != owner() && _msgSender() != marketState[marketId].authorizedPauseAddress)
        revert Errors.E_Unauthorized();
    marketState[marketId].authorizedPauseAddress = newPauseAddress; // ❌ Missing zero address check
    emit Events.UpdateMarketPauseAddress(marketId, oldPauseAddress, newPauseAddress);
}
```
🧰 Mitigation

Add an explicit validation to reject zero addresses before assignment:
```solidity
if (newPauseAddress == address(0)) revert Errors.E_InvalidAddress();
```

This ensures that the pause functionality remains operational and cannot be permanently disabled due to an invalid configuration.

💥 Impact

Loss of emergency control capabilities if address(0) is assigned.
While unlikely in normal operation, this mistake would prevent pausing during a system failure or attack.


# [04] Legacy Pause Operator Retains Control After Ownership Transfer

**Contract Name:** `Covenant.sol`  
**Functions:** `setDefaultPauseAddress`, `createMarket`

---

### 🧠 Description
Each market stores a snapshot of the global pause operator (`_defaultPauseAddress`) at the time of its creation.  
However, when protocol ownership is transferred via `Ownable2Step`, this mapping does not automatically update — meaning that **existing markets continue to reference the previous pause operator**.

As a result, the former owner (or a compromised key) can still pause or unpause active markets, even after governance has been transferred to a new owner.  
This leaves operational control in outdated hands and weakens the integrity of the protocol’s access model.

---

### ⚙️ Vulnerable Code
```solidity
// setDefaultPauseAddress
_defaultPauseAddress = newPauseAddress;

// createMarket
marketState[marketId].authorizedPauseAddress = _defaultPauseAddress;
```

Each market only snapshots _defaultPauseAddress once during creation, and there is no automatic sync afterward.

🧰 Mitigation

Synchronize the authorizedPauseAddress of existing markets when the default pause operator is updated:
```solidity
function setDefaultPauseAddress(address newPauseAddress) external onlyOwner noDelegateCall {
    if (newPauseAddress == address(0)) revert Errors.E_ZeroAddress();

    address oldPauseAddress = _defaultPauseAddress;
    _defaultPauseAddress = newPauseAddress;
    emit Events.UpdateDefaultPauseAddress(oldPauseAddress, newPauseAddress);

    for (uint256 i; i < marketIdList.length; ++i) {
        MarketId id = marketIdList[i];
        if (marketState[id].authorizedPauseAddress == oldPauseAddress) {
            marketState[id].authorizedPauseAddress = newPauseAddress;
            emit Events.UpdateMarketPauseAddress(id, oldPauseAddress, newPauseAddress);
        }
    }
}
```
This ensures that all existing markets inherit the updated operator and prevents legacy addresses from retaining control.

💥 Impact
After an ownership transfer, previous pause operators may still control pausing for active markets.
Although not an immediate exploit, it creates a long-term governance risk if old keys are lost, compromised, or maliciously reused.

# [05] Protocol Fees Become Inaccessible When Market Is Paused

**Contract Name:** `Covenant.sol`  
**Function:** `collectProtocolFee`

---

### 🧠 Description
The `collectProtocolFee()` function is protected by the `lock` modifier, which prevents execution when a market is paused.  
While this behavior protects against state changes during emergencies, it unintentionally **blocks governance from withdrawing accumulated protocol fees** while the market remains paused.

In critical situations where liquidity issues occur, the inability to access treasury funds may delay recovery actions — precisely when fast access to protocol fees could be most valuable.

---

### ⚙️ Vulnerable Code
```solidity
function collectProtocolFee(...) external onlyOwner noDelegateCall lock(marketId) { ... }

modifier lock(MarketId marketId) {
    uint8 statusFlag = marketState[marketId].statusFlag;
    if (statusFlag == STATE_PAUSED) revert Errors.E_MarketPaused();
    ...
}
```

Because lock reverts when the market is paused, collectProtocolFee() cannot be called until the market is unpaused.

🧰 Mitigation

Allow fee collection even when the market is paused by using a non-mutating modifier or a custom check:
```solidity
function collectProtocolFee(
    MarketId marketId,
    address recipient,
    uint128 amountRequested
) external onlyOwner noDelegateCall lockView(marketId) {
    if (recipient == address(0)) revert Errors.E_ZeroAddress();
    if (amountRequested == 0) revert Errors.E_ZeroAmount();
    _collectProtocolFee(marketId, recipient, amountRequested);
}
```

This maintains security while ensuring the protocol can still recover or move treasury funds during paused states.

💥 Impact

Protocol fees cannot be withdrawn during paused periods, limiting treasury flexibility and delaying response to incidents that may require rapid access to funds.
