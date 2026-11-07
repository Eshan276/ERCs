---
title: Protected Accounts for Fungible Tokens
description: Opt-in account flag for ERC-20 tokens that delays large outgoing transfers and lets a backup wallet cancel or rescue
author: Eshan (@Eshan276)
discussions-to: https://ethereum-magicians.org/t/TODO
status: Draft
type: Standards Track
category: ERC
created: 2026-11-07
requires: 20
---

## Abstract

This proposal extends [ERC-20](./eip-20.md) with an opt-in, per-account, publicly readable **protected** flag. While an account is protected, outgoing transfers above a holder-chosen **daily limit** do not settle instantly. They are **scheduled** and can only execute after a holder-chosen **hold window**, during which the holder or a pre-registered **backup** address can cancel them. A **stress** action, callable by the holder or the backup, moves the account's whole spendable balance to the backup and **locks** it there for a preconfigured duration. Changes that increase protection apply instantly; changes that decrease it (turning the flag off, raising the limit, shortening a window, changing the backup) only apply after a holder-chosen **off-delay**. Incoming transfers are never delayed, and accounts that never opt in behave exactly like plain ERC-20 accounts.

## Motivation

Token transfers are instant and final. Once an attacker obtains a private key, or physically coerces a holder (a "wrench attack"), the whole balance can be moved in one transaction. Existing mitigations are either opt-in per transfer, require cooperation from the recipient, depend on a specific smart-contract wallet, or are custodial.

This proposal puts the protection in the token, at the account level:

- It works for any holder, including plain externally owned accounts, with any wallet or application.
- It is public. Anyone (an attacker, a merchant, a frontend) can see that draining an account will not be instant, which deters attacks.
- It does not require the recipient's cooperation and does not break composability: receiving is always instant, unprotected accounts are unaffected, and over-limit pulls by contracts fail cleanly instead of being silently held.
- The holder chooses the trade-off between convenience and safety: a daily limit for everyday use, and waiting periods for everything larger.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Definitions

- **Holder**: an address holding the token.
- **Protected**: a holder that has turned the flag on and whose pending flag-off, if any, has not matured.
- **Backup**: the address a protected holder registers to cancel scheduled transfers and receive funds on stress.
- **Daily limit**: the amount a protected holder MAY send per limit period without scheduling.
- **Hold window**: how long a scheduled transfer waits before it can execute.
- **Off-delay**: how long any protection-decreasing change waits before taking effect.
- **Lock duration**: how long funds moved by stress stay locked in the backup.
- **Reserved**: the part of a holder's balance set aside for pending scheduled transfers.
- **Locked**: the part of a balance locked by stress and not yet expired.
- **Available balance**: balance minus reserved minus locked.

### Interface

Compliant tokens MUST implement [ERC-20](./eip-20.md) and the following interface.

```solidity
interface IProtectedToken /* is IERC20 */ {
    enum Param { DailyLimit, HoldWindow, OffDelay, LockDuration }

    struct ProtectionView {
        bool active;          // protected right now
        address backup;       // effective backup
        uint256 dailyLimit;
        uint64 holdWindow;    // seconds
        uint64 offDelay;      // seconds
        uint64 lockDuration;  // seconds
        uint64 flagOffAt;     // pending flag-off time, 0 if none
        bool fastLane;        // backup may approve scheduled transfers early
        uint64 fastLaneAt;    // pending fast-lane enable time, 0 if none
    }

    event FlagOn(address indexed account, address indexed backup, uint256 dailyLimit, uint64 holdWindow, uint64 offDelay, uint64 lockDuration, bool fastLane);
    event FlagOffRequested(address indexed account, uint64 effectiveAt);
    event FlagOffCancelled(address indexed account);
    event FlagOff(address indexed account);
    event ParamUpdated(address indexed account, Param indexed param, uint256 value);
    event ParamChangeRequested(address indexed account, Param indexed param, uint256 value, uint64 effectiveAt);
    event ParamChangeCancelled(address indexed account, Param indexed param);
    event BackupChangeRequested(address indexed account, address indexed newBackup, uint64 effectiveAt);
    event BackupChangeCancelled(address indexed account);
    event BackupChanged(address indexed account, address indexed oldBackup, address indexed newBackup);
    event FastLaneUpdated(address indexed account, bool enabled);
    event FastLaneChangeRequested(address indexed account, uint64 effectiveAt);
    event TransferQueued(uint256 indexed id, address indexed from, address indexed to, uint256 amount, uint64 executableAt);
    event TransferCancelled(uint256 indexed id, address indexed by);
    event TransferExecuted(uint256 indexed id, address indexed by);
    event TransferApproved(uint256 indexed id, address indexed backup);
    event StressTriggered(address indexed account, address indexed backup, address indexed by, uint256 amount, uint64 lockedUntil);

    error ExceedsDailyLimit(address account, uint256 amount, uint256 remaining);
    error InsufficientUnlockedBalance(address account, uint256 amount, uint256 available);
    error HoldNotExpired(uint256 id, uint64 executableAt);
    error TransferNotPending(uint256 id);

    // Settings (all act on msg.sender)
    function protect(address backup, uint256 dailyLimit, uint64 holdWindow, uint64 offDelay, uint64 lockDuration, bool fastLane) external;
    function requestFlagOff() external;
    function cancelFlagOff() external;
    function setParam(Param param, uint256 value) external;
    function cancelParamChange(Param param) external;
    function requestBackupChange(address newBackup) external;
    function cancelBackupChange() external;
    function setFastLane(bool enabled) external;

    // Scheduled transfers
    function queueTransfer(address to, uint256 amount) external returns (uint256 id);
    function cancelTransfer(uint256 id) external;
    function executeTransfer(uint256 id) external;
    function approveTransfer(uint256 id) external;

    // Emergency
    function stress(address account) external;

    // Views
    function isProtected(address account) external view returns (bool);
    function protectionOf(address account) external view returns (ProtectionView memory);
    function dailyLimitRemaining(address account) external view returns (uint256);
    function availableBalanceOf(address account) external view returns (uint256);
    function reservedOf(address account) external view returns (uint256);
    function lockOf(address account) external view returns (uint256 amount, uint64 until);
    function pendingParamChange(address account, Param param) external view returns (uint256 value, uint64 effectiveAt);
    function pendingBackupChange(address account) external view returns (address newBackup, uint64 effectiveAt);
    function pendingTransferIds(address account) external view returns (uint256[] memory);
}
```

### Rules

**Turning protection on.** `protect` MUST take effect immediately for `msg.sender`, MUST revert if the caller is already protected, and MUST revert if `backup` is the zero address, the caller, or the token itself. Implementations MUST enforce minimum values for `holdWindow`, `offDelay` and `lockDuration` (see Rationale) and SHOULD enforce maximums.

**Increase now, decrease later.** A change that increases protection MUST apply immediately: a lower daily limit; a longer hold window, off-delay or lock duration; turning the fast lane off; cancelling any pending decrease. A change that decreases protection MUST only take effect after the off-delay in force **when it was requested**: a higher daily limit; a shorter hold window, off-delay or lock duration; enabling the fast lane after setup; any backup change; turning protection off. Pending decreases MUST be visible through the view functions. Applying an immediate increase SHOULD clear a pending decrease of the same parameter.

**Daily limit.** For a protected account, every outgoing balance movement (`transfer`, `transferFrom`, spending through [ERC-2612](./eip-2612.md) permits, burns) except `executeTransfer`, `approveTransfer` and `stress` MUST count against the daily limit. A movement that would exceed the remaining limit MUST NOT move tokens. Approvals themselves MUST NOT be restricted.

**Over-limit transfers.** `transferFrom` above the remaining limit MUST revert with `ExceedsDailyLimit`. It MUST NOT return `true` without moving tokens. `transfer` above the remaining limit MUST either revert with `ExceedsDailyLimit` or, if the caller is an externally owned account sending its own transaction, MAY instead schedule the transfer exactly as `queueTransfer` would and return `true`. In that case no ERC-20 `Transfer` event may be emitted until the transfer executes.

**Scheduled transfers.** `queueTransfer` MUST only be callable by a protected account, MUST reserve `amount` from the caller's available balance without moving it, and MUST emit `TransferQueued`. A pending scheduled transfer:
- MUST be cancellable by its sender or the sender's effective backup, which releases the reservation;
- MUST be executable by anyone once `block.timestamp >= executableAt`, which moves the tokens without counting against the daily limit;
- MAY be executed earlier by the sender's backup through `approveTransfer`, but only if the sender has the fast lane enabled.

**Stress.** `stress(account)` MUST be callable only by `account` or its effective backup while `account` is protected. It MUST cancel every pending scheduled transfer of `account` and every pending protection decrease, move the account's available balance to the backup, and lock that amount in the backup until `block.timestamp + lockDuration`. The lock duration MUST come from the account's settings, not from the caller. Locked tokens MUST NOT be movable by any path until the lock expires. A stress while a lock is active SHOULD add to the existing lock without extending its expiry.

**Reserved and locked funds.** No outgoing movement may spend reserved or locked tokens, whether or not the account is protected.

**Incoming transfers** MUST never be delayed or restricted by this proposal.

**Unprotected accounts** MUST behave exactly as in [ERC-20](./eip-20.md), apart from the reserved/locked rule above, which only applies to tokens set aside before the flag was turned off.

## Rationale

**Token-level, not wallet-level.** Putting the rules in the token makes them work for every holder and every wallet, including plain externally owned accounts, without migrating to a smart-contract wallet.

**Revert, don't silently hold, on `transferFrom`.** Contracts that pull tokens (routers, vaults, lending markets) act on the result immediately. If an over-limit pull returned `true` without moving tokens, a holder could "deposit" scheduled funds, be credited, and then cancel the scheduled transfer with their own backup. A clean revert keeps every integration safe.

**Optional auto-scheduling for direct sends.** When an externally owned account calls `transfer` itself, no third-party contract is acting on the result, so scheduling instead of reverting makes ordinary wallet "Send" buttons work. Emitting the ERC-20 `Transfer` event only at execution keeps indexers from crediting recipients early. Contracts and code-bearing accounts get a revert. Accounts with an [EIP-7702](./eip-7702.md) delegation are treated as code-bearing; see Security Considerations.

**Minimums and maximums.** Without minimums, protection could be configured to be meaningless (a zero-second hold or off-delay). Without maximums, a typo or a coerced holder could freeze funds for decades. A deployment-time choice of minimums lets test deployments use minutes while production deployments use days; RECOMMENDED production minimums are a 1-day hold window, a 7-day off-delay and a 1-day lock.

**O(1) stress.** Implementations SHOULD invalidate all pending scheduled transfers at once (for example with a per-account epoch), so an attacker cannot block `stress` by queueing many small transfers.

**Fast lane.** Letting the backup approve a scheduled transfer means big payments never require switching protection off, which removes the window an attacker who silently holds a copy of the key would wait for. It is opt-in and its enabling is delayed, because a holder who keeps both keys together can be forced to approve with both.

## Backwards Compatibility

The proposal is fully backwards compatible with [ERC-20](./eip-20.md). Unprotected accounts are unaffected, incoming transfers are never restricted, and over-limit pulls by contracts revert just like any failed `transferFrom`. Existing tokens can adopt the behavior through a 1:1 wrapper.

## Reference Implementation

A reference implementation (an abstract `ProtectedERC20` on top of OpenZeppelin's ERC-20 and ERC-20 Permit, plus a 1:1 wrapper) with a full test suite, including a mainnet-fork test against an unmodified Uniswap V2 router, will be added to the assets directory of this proposal.

## Security Considerations

- **Not retroactive.** Protection only guards against keys compromised after it is turned on. If the key is already compromised, an attacker can drain the account before `protect`, or front-run `protect` with their own backup. Holders SHOULD enable protection on a fresh account before funding it, and interfaces SHOULD flag a backup the user did not choose.
- **Leaked key, waiting attacker.** Once the flag is off, whoever holds the key has full control. Holders who suspect a compromise SHOULD move funds to a new account via scheduled transfers instead of turning protection off.
- **The backup is the root of trust.** If both the holder's key and the backup's key are compromised, the attacker can stress the funds into the backup and withdraw them after the lock.
- **Watching is required.** Delays only help if someone notices. Interfaces and watchers SHOULD alert the holder and backup on `TransferQueued`, `FlagOffRequested`, `ParamChangeRequested`, `BackupChangeRequested` and `FastLaneChangeRequested`.
- **Value extraction through scheduled swaps.** Contracts that let a protected holder pre-register actions on scheduled funds (for example a swap) MUST NOT let the action's terms send value to anyone but the holder. They SHOULD constrain those terms, such as fixing the venue and enforcing a minimum output, because an attacker could otherwise route the funds through a venue they control.
- **EIP-7702 accounts.** Delegated accounts carry code. Implementations that auto-schedule direct sends only for code-less callers will revert for them instead; interfaces SHOULD offer explicit scheduling via `queueTransfer`.
- **Timestamp dependence.** All windows use `block.timestamp`. The windows are minutes to days long, so validator influence over timestamps is negligible.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
