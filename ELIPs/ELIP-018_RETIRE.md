# ELIP-018: RETIRE - Retirement Enabling Terminal, Irreversible Restaking Exit

| Author(s) | Created | Status | References | Discussions |
| :---- | :---- | :---- | :---- | :---- |
| [Matt Curtis](mailto:matt.curtis@eigenlabs.org) | 2026-07-10 | `draft` | [PR #1758](https://github.com/Layr-Labs/eigenlayer-contracts/pull/1758) | Forum: TODO |

---

# Executive Summary

`RETIRE` is an `EigenPod` upgrade that lets a Pod Owner **permanently and irreversibly disable restaking** on their pod via a new `disablePod` method on the `EigenPodManager`. Once disabled, the pod is retired from EigenLayer's share accounting: no new shares can be minted, all ETH the pod holds is treated as non-restaked and is fully sweepable, and [validator consolidation](#consolidation-retired-pods-only) becomes available to the owner — to **any** target, including validators in other pods or outside EigenLayer entirely. Consolidation is now **exclusively** a retired-pod capability: the in-pod consolidation path introduced in [ELIP-009 (MOOCOW)](./ELIP-009.md) is withdrawn for pods that are still restaking, as a security hardening adopted from audit (see [Security Considerations](#security-considerations)).

This unlocks two capabilities that EigenPods have historically lacked: a way to **rotate the keys** controlling a pod's validators (by consolidating their balance into freshly-keyed validators) without exiting and re-entering the (presently multi-week) beacon-chain entry queue, and a **low-friction exit** from EigenLayer that does not require completing a final checkpoint.

Retirement itself is **opt-in** and **irreversible**. Pods that never call `disablePod` keep restaking exactly as before, with one deliberate exception: `requestConsolidation` is no longer callable on a pod that has not been retired. No existing method signature changes. The upgrade adds one new `EigenPodManager` method (`disablePod`), two new `EigenPod` methods (`disableRestaking`, `withdrawDisabledPodETH`), one new `DelegationManager` method (`clearQueuedWithdrawalsForDisabledPod`), a new `PAUSED_DISABLE_POD` pause flag, a `TRUSTED_CHECKPOINT_TIMESTAMP` checkpoint-freshness gate, and a set of preconditions that guarantee **no value is lost, double-paid, or used to evade slashing** across the irreversible transition.

# Motivation

EigenPods are keyed to the staker address that creates them, deterministically via `CREATE2`. Unless a staker had the foresight to place a rekeyable contract in front of EigenLayer staking, **there is currently no mechanism to rotate the keys** controlling a pod's validators. For large staking providers this is a material operational gap: a provider with thousands of validators pointed at EOA-created pods — representing hundreds of millions of dollars of staked ETH — has no way to rekey short of exiting every validator and re-entering the beacon-chain entry queue. With the entry queue exceeding 50 days, that exit-and-re-enter path can cost millions of dollars in lost staking yield, purely to change a key.

More broadly, native ETH in EigenPods is the most illiquid and lowest-value-to-protocol capital locked in EigenLayer, and it is also the hardest to remove. Redistribution of native ETH without creating perverse incentives to abandon it to slashing on the BeaconChain is difficult due to the unbounded nature of the withdrawal queue, and the same illiquidity is the chief barrier to native ETH leaving the protocol when restakers wish to exit. Fully exiting a pod today is onerous — requiring the restaker to exit the beaconchain entirely and choose between forgoing validation rewards for the escrow period by fully exiting the validators before queuing EigenLayer withdrawals or completing additional checkpoints and withdrawals to handle rewards accrued during escrow.

[ELIP-009 (MOOCOW)](./ELIP-009.md) introduced execution-layer-triggerable validator consolidation, by which the balance of one validator (`source`) is transferred in its entirety to another (`target`). This is a promising mechanism to transfer the ownership of a validator; however, the accounting mechanisms of the pod require the `target` to have verified withdrawal credentials pointed at the **same** pod — the sole barrier preventing consolidation to validators in other pods or outside EigenLayer. That check exists to keep **restaked** assets in the pod's custody.

The root cause of all of these pain points is the pod's share accounting. So long as any assets entering the EigenPod must be considered for possible beaconchain or EigenLayer slashing, stringent accounting checks must remain in place to ensure that penalties can be properly applied. However, assets that have completed the escrow period and are ready to withdraw are no longer considered restaked, and no longer subject to modification by slashing on the beaconchain or EigenLayer. As a result, it is already possible for an EigenPod to enter a state in which it is not actively participating in EigenLayer, yet the stringent accounting checks remain in place.

`RETIRE` resolves these pain points by introducing a **terminal state**. A Pod Owner who has fully withdrawn may set a permanent `restakingDisabled` flag that turns off share minting forever; once set, the pod no longer needs to track the provenance of ETH entering it and merely acts as a passthrough giving the owner custodial access to their funds. With share minting permanently off, consolidation to arbitrary targets becomes safe, enabling both key rotation (consolidate into freshly-keyed validators) and a clean exit (sweep the pod directly). Audit review of this release found that consolidation while a pod is still restaking can interfere with the accuracy of the checkpoint-based balance accounting that the retirement preconditions depend on, so `RETIRE` confines consolidation to retired pods entirely (see [Security Considerations](#security-considerations)). The transition is gated by preconditions ensuring the pod carries no in-flight share-mutating operations and no restaked value in excess of what has already been queued for withdrawal at the moment it freezes.

# Features & Specification

## Overview

The feature centers on a single irreversible state flag, `restakingDisabled`, on the `EigenPod`. A Pod Owner triggers the transition by calling `disablePod` on the `EigenPodManager`, which validates the staker-level preconditions and then, atomically:

1. Sets the pod's `restakingDisabled` flag via `EigenPod.disableRestaking`, freezing all share-minting entry points (`verifyWithdrawalCredentials`, `startCheckpoint`, `verifyStaleBalance`, `stake`) and the standard beacon-chain-ETH withdrawal path (`withdrawRestakedBeaconChainETH`).
2. Clears the Pod Owner's now-uncompletable queued beacon-chain-ETH withdrawals from the `DelegationManager` via `clearQueuedWithdrawalsForDisabledPod`.
3. Unlocks `requestConsolidation` — owner-only, any target — which is callable **only** on a retired pod.

Recovery of the pod's ETH thereafter flows through `withdrawDisabledPodETH`, a direct sweep of the pod's entire ETH balance that bypasses the `DelegationManager` withdrawal queue.

## Preconditions

The safety of this feature is ensured by a set of preconditions verified in `EigenPodManager.disablePod` (and, for the checkpoint check, in `EigenPod.disableRestaking`) to ensure the EigenPod has no in-flight share-mutating operations and holds no restaked value beyond what has already been queued for withdrawal at the time it freezes.

Firstly, the caller MUST have a deployed pod (`EigenPodDoesNotExist`), and the EigenPod cannot have a checkpoint in progress (`currentCheckpointTimestamp == 0`, `CheckpointAlreadyActive`), as the completion of a checkpoint adds restaked shares to the EigenPod.

The pod's most recent completed checkpoint MUST also be recent enough to trust: `pod.lastCheckpointTimestamp()` MUST be at or after `TRUSTED_CHECKPOINT_TIMESTAMP` (else `StaleCheckpointSnapshot`). `TRUSTED_CHECKPOINT_TIMESTAMP` is an immutable on the `EigenPodManager`, set on mainnet to `1_753_132_583` (2025-07-21, the v1.6.0 `EigenPod` upgrade) and to `1` on other networks. The value-conservation check below reads the pod's beacon-chain balance from its retained checkpoint snapshot; a pod that last checkpointed before the current checkpoint layout was introduced — or that has never checkpointed — would present a zero or misinterpreted balance and could pass that check while still controlling restaked value on the beacon chain. Such pods must complete one fresh checkpoint before retiring.

Secondly, the Pod Owner MUST have exactly zero deposit shares in the `EigenPodManager` (`DepositSharesNotZero`). This is achieved by queuing withdrawals of all previously checkpointed shares. In this state, the EigenPod has no active stake, and future slashing cannot mutate withdrawable amounts.

Thirdly, every queued withdrawal that includes the beacon-chain-ETH strategy MUST be pure beacon-chain ETH (`MixedWithdrawalPending`) and MUST have remained slashable for the full withdrawal delay (`block.number > startBlock + minWithdrawalDelayBlocks`, else `WithdrawalStillSlashable`). Waiting out the delay locks the withdrawal's slashing factor before the freeze, so any beacon-chain or EigenLayer slashing that applies to the pod's value has already been priced into the queued amount.

Finally, the pod's total restaked balance MUST NOT exceed the value already queued for withdrawal. `disablePod` computes the pod's restaked balance (`withdrawableRestakedExecutionLayerGwei` plus the most recent checkpoint's `prevBeaconBalanceGwei` and `balanceDeltasGwei`) and requires it to be less than or equal to the summed completion value of the staker's queued beacon-chain-ETH withdrawals, with **1 gwei of slack** to absorb rounding in the `DelegationManager` (`PodValueExceedsQueuedWithdrawals`). Note that the beacon-chain component is read from the pod's *last completed* checkpoint: finalization clears only `currentCheckpointTimestamp`, leaving the checkpoint struct itself in storage, so `currentCheckpoint()` returns that retained snapshot when — as `disablePod` requires — no checkpoint is active; the `TRUSTED_CHECKPOINT_TIMESTAMP` gate above ensures that snapshot was produced under the current checkpoint layout. This value-conservation check is the core safety invariant: it guarantees the owner cannot disable while the pod still custodies restaked value that has not been priced through the slashing-aware withdrawal queue.

### Limitation: AVS-Slashed Pods Cannot Currently Disable

The value-conservation check compares two figures that account for AVS slashing differently. A queued withdrawal's completion value, as returned by `getQueuedWithdrawals`, is scaled down by the operator's slashing factor at `slashableUntil` — so an AVS slash **reduces** the queued figure. The pod's tracked restaked balance, however, is **not** reduced by AVS slashing: there is presently no mechanism to burn slashed shares from an `EigenPod`. When beacon-chain-ETH shares are AVS-slashed, the burned amount is accumulated in `EigenPodManager.burnableETHShares` but is never debited from the pod's `withdrawableRestakedExecutionLayerGwei` or checkpoint accounting.

The consequence is that an AVS-slashed pod's tracked balance permanently exceeds the slashing-adjusted value it can queue for withdrawal. Such a pod can never satisfy `podBalanceWei <= queuedWithdrawableWei + 1 gwei`, so `disablePod` reverts with `PodValueExceedsQueuedWithdrawals`. **A pod that has been AVS-slashed on the beacon-chain-ETH strategy therefore cannot use the disable feature** until a mechanism exists to remove slashed shares from the pod's balance. Pods that have only been beacon-chain slashed are unaffected, because beacon-chain slashing is reflected in the pod's balance via checkpoints. This limitation is a candidate for a future follow-up (see [Forwards Compatibility](#forwards-compatibility)).

### Limitation: Legacy Queued Withdrawals Must Be Completed First

`disablePod` and `clearQueuedWithdrawalsForDisabledPod` operate on the withdrawals enumerated in `_stakerQueuedWithdrawalRoots`. Beacon-chain-ETH withdrawals queued before the v1.0 slashing release are tracked only in the legacy `pendingWithdrawals` mapping, so they are neither counted toward the queued value in the conservation check nor cleared at retirement. In practice an outstanding legacy withdrawal will usually cause `disablePod` to revert with `PodValueExceedsQueuedWithdrawals`, because the legacy amount still inflates the pod's tracked balance without a matching queued entry. Should a retirement nonetheless succeed with a legacy withdrawal outstanding, that entry can never be completed afterward — both completion paths revert with `RestakingDisabled` — though the underlying ETH remains recoverable through the pod sweep. Stakers should complete any legacy withdrawals before calling `disablePod`.

## Core Functionality

### Permanent Disablement (`disablePod`)

```
/// @notice Permanently disables restaking for the caller's pod once all native-ETH shares have
/// been queued and remained slashable for the full withdrawal delay. Callable by the Pod Owner.
function disablePod() external; // EigenPodManager
```

Validates the preconditions above, then calls `EigenPod.disableRestaking` (setting `restakingDisabled = true`) and `DelegationManager.clearQueuedWithdrawalsForDisabledPod`, and emits `PodRestakingDisabled(podOwner, pod)`. The flag can never be cleared. `disablePod` is guarded by the new `PAUSED_DISABLE_POD` pause flag and is `nonReentrant`.

```
/// @notice Permanently disables restaking for this pod. Callable only by the EigenPodManager.
function disableRestaking() external; // EigenPod
```

Sets `restakingDisabled = true` and emits `RestakingPermanentlyDisabled()`. Reverts if the pod is already disabled (`RestakingDisabled`) or if a checkpoint is active (`CheckpointAlreadyActive`) — finalizing a checkpoint after the freeze would attempt to credit shares and revert. This method is `onlyEigenPodManager`; the Pod Owner reaches it only transitively through `disablePod`.

### Queue Cleanup (`clearQueuedWithdrawalsForDisabledPod`)

```
/// @notice Clears queued beacon-chain withdrawals for a staker whose pod has been disabled.
/// @dev Callable only by the EigenPodManager.
function clearQueuedWithdrawalsForDisabledPod(address staker) external; // DelegationManager
```

Invoked by `disablePod` directly on the `DelegationManager` (`onlyEigenPodManager`). For each of the staker's queued withdrawals containing the beacon-chain-ETH strategy, it removes the queue entry (`_stakerQueuedWithdrawalRoots`, `_queuedWithdrawals`, `pendingWithdrawals`) and emits `QueuedWithdrawalClearedForDisabledPod(withdrawalRoot)`. Deposit shares and operator delegation were already decremented at queue time, so **no shares are re-credited and no delegation is changed** — only the now-uncompletable entries are removed. Pure non-beacon-chain-ETH withdrawals (which route through the `StrategyManager` and are unaffected by the pod disabling) are left untouched. As defense-in-depth, the method reverts (`MixedWithdrawalNotClearable`) on any mixed beacon-chain-ETH \+ other-strategy withdrawal, even though the disable preconditions already reject those upstream.

### Disabled-Pod Sweep (`withdrawDisabledPodETH`)

```
/// @notice Withdraws all ETH held by a disabled pod to `recipient`.
/// @dev Callable only by the Pod Owner once `restakingDisabled` is true.
function withdrawDisabledPodETH(address recipient) external; // EigenPod
```

Transfers the pod's **entire** ETH balance (`address(this).balance`) to `recipient` and emits `DisabledPodETHWithdrawn(recipient, amountWei)`. Requires `restakingDisabled` (`RestakingNotDisabled`) and a non-zero recipient (`InputAddressZero`), is guarded by the `PAUSED_NON_PROOF_WITHDRAWALS` pause flag, and is `nonReentrant`. This is the primary exit path for ETH arriving at a disabled pod (validator exits, consolidation targets pointed back at the pod, dust, direct sends), since checkpoints are locked and shares cannot be minted. The standard `withdrawRestakedBeaconChainETH` path reverts (`RestakingDisabled`) once disabled, so `restakedExecutionLayerGwei` becomes inert and the full balance is recoverable only through this sweep.

### Consolidation (Retired Pods Only)

`requestConsolidation` is callable **only** once `restakingDisabled == true` (`RestakingNotDisabled` otherwise) and **only by the Pod Owner** (`OnlyEigenPodOwner`). For a retired pod it accepts any target validator — including one outside this pod or outside EigenLayer entirely — because a disabled pod mints no shares, so there is no accounting invariant for a target check to protect. Both forms of EIP-7251 request are covered: standard consolidations (`srcPubkey != targetPubkey`) and switch-to-compounding requests (`srcPubkey == targetPubkey`). EIP-7251 itself still restricts the *source* to validators whose withdrawal credentials are this pod.

Pods that are still restaking can no longer request consolidations at all. This replaces the ELIP-009 behavior (consolidation permitted while restaking, provided the target was `ACTIVE` in the same pod) and is a security hardening adopted from the audit of this release: consolidation on an actively restaking pod can interfere with the accuracy of the checkpoint-based accounting that the retirement preconditions depend on — see [Rationale](#consolidation-only-after-retirement) and [Security Considerations](#security-considerations). Because retirement already requires all shares to be queued and un-slashable, and a retired pod cannot checkpoint, confining consolidation to retired pods removes the concern without affecting the retirement workflows. The Proof Submitter cannot submit consolidations — see [Owner-Only Consolidation](#owner-only-consolidation).

`requestWithdrawal` is unchanged in access (Pod Owner or Proof Submitter); on a disabled pod it drops the "validator MUST be `ACTIVE`" check, allowing the owner to exit validators that were never verified to the pod.

## Process Flow: Key Rotation

The motivating workflow — rotating the keys controlling a pod's validators without exiting the beacon chain — proceeds as follows:

1. Create new beacon-chain target validators with the desired withdrawal credentials.
2. Queue EigenLayer withdrawals of all native ETH in the pod, as set by the most recently completed checkpoint.
3. Wait for the EigenLayer escrow period (`minWithdrawalDelayBlocks`) to elapse, so the queued withdrawals are past `slashableUntil`.
4. Call `disablePod` on the `EigenPodManager`.
5. Submit beacon-chain consolidation requests (via `requestConsolidation` — owner-only, and available only now that the pod is retired) to transfer balance from the pod's validators into the newly-keyed validators.
6. Sweep any ETH dust that arrives in the pod to the owner via `withdrawDisabledPodETH`.

This lets a restaker signal intent to fully exit the current pod **without completing any further checkpoints**, avoiding both the lost yield of the exit-and-re-enter path and the "restaked dust" of the conventional exit ordering.

## Process Flow: Exit Without a Final Checkpoint

The second motivating workflow — fully exiting a pod and recovering its ETH without completing a final checkpoint — proceeds as follows:

1. Queue EigenLayer withdrawals of all native ETH in the pod, as set by the most recently completed checkpoint, bringing the Pod Owner's deposit shares to zero.
2. Wait for the EigenLayer escrow period (`minWithdrawalDelayBlocks`) to elapse, so the queued withdrawals are past `slashableUntil`.
3. Call `disablePod` on the `EigenPodManager`.
4. Broadcast full-exit requests for the pod's validators, so their balances flow back to the pod as they exit the beacon chain.
5. As exited balances (and any partial-withdrawal or consolidation ETH) arrive, sweep the pod's entire balance to the owner via `withdrawDisabledPodETH`. This may be called repeatedly as more ETH lands.

### Why This Is Better Than the Conventional Exit

Under the conventional exit ordering, a restaker exiting a pod must reconcile beacon-chain withdrawals with EigenLayer's checkpoint-based share accounting, and faces an awkward choice:

* **Exit validators first, then queue EigenLayer withdrawals.** The restaker forgoes validation rewards for the full escrow period, since the validators are already off the beacon chain before the EigenLayer withdrawal can even be queued.
* **Queue EigenLayer withdrawals against the last checkpoint, then exit validators.** Rewards accrued between the last checkpoint and the validator exit land in the pod as **"restaked dust"** — ETH that is credited as new shares on the next checkpoint. Recovering it requires completing *additional* checkpoints and queuing *further* withdrawals, each carrying its own escrow period and additional gas costs.

`RETIRE` collapses this into a single terminal transition. Once the pod is disabled, checkpoints are frozen and no further shares can ever be minted, so any ETH arriving after disablement — late validator exits, skimmed rewards, consolidation targets pointed back at the pod — is **not** restaked dust; it is plain, immediately-sweepable balance. The restaker:

* completes **no further checkpoints** after the single queued withdrawal clears escrow;
* incurs the escrow period **once**, rather than repeatedly chasing residual rewards;
* keeps validators earning on the beacon chain right up until they choose to exit them, rather than idling them to avoid dust; and
* recovers *all* pod ETH — including amounts that would otherwise have been trapped as restaked dust — through the direct `withdrawDisabledPodETH` sweep.

# Rationale

## Irreversibility

Disablement is permanent by design. A reversible flag would allow a Pod Owner to toggle around the frozen-accounting guarantees — for example, disabling to escape the consolidation restriction, then re-enabling with balance redirected. Making the flag write-once (enforced by the `RestakingDisabled` guard in `disableRestaking`) removes that entire class of state-toggling attack and lets every downstream guarantee assume the pod never re-enters an active state.

## Value Conservation

Disablement is gated on a single **value-conservation invariant**: the pod's tracked restaked balance must not exceed the completion value of its queued beacon-chain-ETH withdrawals (with 1 gwei of slack for rounding). Combined with the requirement that every such withdrawal has remained slashable for the full delay — locking its slashing factor before the freeze — this guarantees the owner cannot walk away with more than the slashing-adjusted value already committed to the queue.

The check exists to stop a pod owner from recollecting funds an AVS has slashed from them. When beacon-chain-ETH shares are AVS-slashed, the burn is recorded in `burnableETHShares` but never removed from the pod, so an AVS-slashed pod holds more ETH than it is owed — which a disabled owner could recollect by sweeping the pod or consolidating its validators to external targets. Rather than build machinery to burn slashed shares out of a pod, RETIRE keeps disable strictly safe by refusing to freeze any pod whose tracked balance exceeds its queued value. (Beacon-chain slashing needs no such handling — it settles on the beacon chain itself, so a slashed validator simply delivers less ETH to the pod.) The trade is deliberate: AVS-slashed pods cannot yet retire, but no disable can ever release an un-burned slash — see [Limitation: AVS-Slashed Pods Cannot Currently Disable](#limitation-avs-slashed-pods-cannot-currently-disable).

## Inert REL and Full-Balance Sweep

In the standard `DelegationManager` withdrawal path, beacon-chain-ETH withdrawals must be completed against the pod's `restakedExecutionLayerGwei` (REL) via `withdrawRestakedBeaconChainETH`. Once the pod is disabled, that method reverts, so REL can neither be increased (checkpoints are frozen) nor drawn down through the standard path — it becomes inert. Recovery is routed entirely through `withdrawDisabledPodETH`, which sweeps the pod's **entire** ETH balance in one call. The value-conservation precondition ensures the pod holds no restaked value beyond what was already queued and cleared, so sweeping the full balance cannot extract share-reserved funds.

## Queue Cleanup vs. Inert Withdrawals

Once `addShares` is blocked and the standard token path reverts for a disabled pod, any queued beacon-chain-ETH withdrawal becomes completable by neither path (tokens nor shares). Clearing is therefore **mandatory** and performed atomically within `disablePod`, because a "pending but permanently uncompletable" entry is a standing invariant violation that pollutes queue views and off-chain indexers. Clearing re-credits nothing (shares and delegation were decremented at queue time); the value is recovered via validator consolidations to external pods or the `withdrawDisabledPodETH` sweep. While this expands the scope of changes to involve the `DelegationManager`, the modification is justified.

Note that both `disablePod` and `clearQueuedWithdrawalsForDisabledPod` iterate over the staker's full list of queued withdrawals, which is in principle unbounded and could in the extreme make the transaction too gas-heavy to execute. In practice this does not prevent disablement: the list only grows through explicit `queueWithdrawals` calls by the staker, each of which costs gas, so reaching a length that would exhaust the block gas limit is neither cheap nor accidental. A staker who has fanned their beacon-chain-ETH shares across an impractically large number of queued withdrawals can simply complete or consolidate some of them before disabling to bring the list back within a workable range. The common case — a pod with a small number of queued withdrawals — is unaffected.

## Rejecting Mixed-Strategy Withdrawals

A withdrawal bundling the beacon-chain-ETH strategy with another strategy in a single `Withdrawal` cannot be safely handled at disable: completing it would revert on the frozen beacon-chain-ETH leg, trapping the other leg, whose value cannot be recovered from the pod. Such withdrawals are only producible by calling `queueWithdrawals` directly with multiple strategies in one parameter — `undelegate`/`redelegate` always queue one withdrawal per strategy. Disable therefore rejects them (`MixedWithdrawalPending`), requiring the owner to complete them first, and `clearQueuedWithdrawalsForDisabledPod` reverts on them (`MixedWithdrawalNotClearable`) as defense-in-depth.

## Consolidation Only After Retirement

Consolidation is the one capability `RETIRE` removes from actively restaking pods. Under ELIP-009 an active pod could consolidate provided the target validator was `ACTIVE` in the same pod; that check kept restaked balance inside the pod, but the audit of this release found that consolidation on an active pod can still interfere with the accuracy of the pod's checkpoint-based balance accounting — the accounting on which the value-conservation precondition depends. Rather than add further accounting machinery to track consolidations, `RETIRE` removes the interaction: because retirement already requires all shares to be queued and un-slashable, and a retired pod can never checkpoint again, consolidation on a retired pod cannot influence any accounting the protocol relies on — which makes "retired pods only" both the safe and the sufficient rule. The cost is that an owner who wants to consolidate must first retire the pod; since consolidation's motivating uses (key rotation and exit) already run through retirement, this is judged acceptable.

## Owner-Only Consolidation

`requestConsolidation` is restricted to the Pod Owner (`onlyEigenPodOwner`). The Proof Submitter — a secondary hot-wallet address able to submit proofs and withdrawal requests on the owner's behalf — cannot submit consolidations. Because a retired pod's consolidations can direct validator balance to arbitrary external targets, moving value out of the pod entirely, extending that power to a delegated Proof Submitter would meaningfully expand the trust placed in it. Keeping the highest-consequence action under the owner's direct control is the conservative choice; the Proof Submitter retains `verifyWithdrawalCredentials`, `startCheckpoint`, and `requestWithdrawal`.

## Updates to Existing Functionality

### Enhanced Event System

* `PodRestakingDisabled(podOwner, pod)` — emitted by the `EigenPodManager` when a pod is disabled.
* `RestakingPermanentlyDisabled()` — emitted by the `EigenPod` when its flag is set.
* `DisabledPodETHWithdrawn(recipient, amountWei)` — emitted on a disabled-pod sweep.
* `QueuedWithdrawalClearedForDisabledPod(withdrawalRoot)` — emitted by the `DelegationManager` per cleared queue entry.

### Storage & Method Changes

* New `EigenPod` storage flag `restakingDisabled` (write-once). It packs into the existing `proofSubmitter` slot and consumes no new storage slot; `EigenPodStorage.__gap` is reduced from 36 to 35 accordingly.
* New `EigenPodManager` immutable `TRUSTED_CHECKPOINT_TIMESTAMP` (mainnet: `1_753_132_583`; other networks: `1`). `disablePod` reverts with `StaleCheckpointSnapshot` for pods whose last completed checkpoint predates it.
* New `EigenPodManager` pause flag `PAUSED_DISABLE_POD` (index 11).
* `requestConsolidation` now requires `restakingDisabled` (`RestakingNotDisabled` otherwise) and is `onlyEigenPodOwner`; its former in-pod `ACTIVE`-target check is removed. This is a behavioral change for pods that have not retired.
* `requestWithdrawal` gains a conditional branch keyed on `restakingDisabled` (the `ACTIVE`-validator check is skipped for a disabled pod); access is unchanged.
* `withdrawDisabledPodETH` is `nonReentrant`.
* `verifyWithdrawalCredentials`, `startCheckpoint`, `verifyStaleBalance`, `stake`, and `withdrawRestakedBeaconChainETH` revert (`RestakingDisabled`) for a disabled pod.
* `EigenPodManager.addShares` reverts (`RestakingDisabled`) for a disabled pod.

## Compatibility & Integration

### Backwards Compatibility

Retirement is opt-in via an explicit owner call, no existing method signature changes, and the new errors and events are additive. There is **one intentional behavioral change for pods that never retire**: `requestConsolidation` reverts (`RestakingNotDisabled`) unless the pod is disabled, so in-pod consolidation of an actively restaking pod — permitted since ELIP-009 — is no longer available. Owners who wish to consolidate must first retire the pod; all other behavior of non-retired pods is unchanged.

### Forwards Compatibility

The terminal state is intentionally minimal and self-contained, leaving room for future capabilities to be layered on without revisiting the core invariants. In particular, a mechanism to remove AVS-slashed shares from an `EigenPod`'s tracked balance would let AVS-slashed pods satisfy the value-conservation invariant and use the disable feature, lifting the limitation described above.

# Security Considerations

## Protocol Security

* **Risk:** A disabled pod could be used to evade slashing — e.g. disable, consolidate the validator out to an external target, and abandon a slashed-rate queued claim.

  * **Mitigation:** Disable requires that every queued beacon-chain-ETH withdrawal has remained slashable for the full withdrawal delay (locking its slashing factor), and that the pod's tracked restaked balance does not exceed the queued withdrawals' completion value (`PodValueExceedsQueuedWithdrawals`). Together with the zero-deposit-shares requirement, this guarantees no restaked value escapes the slashing-aware queue at the freeze.


* **Risk:** Consolidation requests made while a pod is actively restaking could interfere with the accuracy of the pod's checkpoint-based balance accounting, on which the value-conservation check depends.

  * **Mitigation:** `requestConsolidation` is callable only on a retired pod. Retirement requires every native-ETH share to have been queued and to have remained slashable for the full withdrawal delay, and a retired pod cannot start or complete checkpoints, so no consolidation can influence the balance from which the value-conservation check or the slashing factor is computed. Ordinary validator exits and EIP-7002 withdrawal requests do not raise the same concern — the withdrawn ETH lands in the pod and is accounted for within the same checkpoint — so `requestWithdrawal` remains available to active pods.


* **Risk:** The value-conservation check reads the pod's beacon-chain balance from the retained `_currentCheckpoint` snapshot. For a pod whose last checkpoint predates the current checkpoint layout (the v1.0 slashing release added `prevBeaconBalanceGwei` to the struct), or that has never checkpointed, that snapshot is uninitialized or misinterpreted and can read as zero while the pod still controls validator balance on the beacon chain — allowing a slashed owner to retire, delete the slashed queued withdrawal, and later exit the validators and sweep unslashed ETH.

  * **Mitigation:** `disablePod` requires `pod.lastCheckpointTimestamp() >= TRUSTED_CHECKPOINT_TIMESTAMP` (`StaleCheckpointSnapshot`). The mainnet constant corresponds to the v1.6.0 `EigenPod` upgrade, so any checkpoint started after it was produced under the current layout. Affected pods complete one fresh checkpoint before retiring.


* **Risk:** Clearing a queued withdrawal could re-credit shares or destroy recoverable value.

  * **Mitigation:** `clearQueuedWithdrawalsForDisabledPod` only deletes queue entries; it never calls `addShares`, never touches `operatorShares` or deposit shares, and never changes delegation (all decremented at queue time). It is access-controlled (`onlyEigenPodManager`, reached only via `disablePod`), so a pod can clear only its own owner's withdrawals. Pure non-beacon withdrawals are skipped, and mixed withdrawals revert (`MixedWithdrawalNotClearable`) as defense-in-depth.


* **Risk:** Reentrancy across the `EigenPodManager` → `EigenPod` → `DelegationManager` call chain during disable.

  * **Mitigation:** No step in the disable flow hands control to attacker-controlled code — every call is to a trusted protocol contract (`EigenPodManager` → `EigenPod`, `EigenPodManager` → `DelegationManager`). `disablePod` and `clearQueuedWithdrawalsForDisabledPod` are additionally `nonReentrant`. `disableRestaking` is `onlyEigenPodManager` and reachable only through `disablePod`, so it inherits that guard and cannot be re-entered; the flag is set before the queue-clearing call, so the pod is already frozen for the remainder of the flow.


* **Risk:** The disabled-pod sweep could withdraw value reserved for the share-withdrawal flow.

  * **Mitigation:** The sweep is gated on `restakingDisabled`, and the value-conservation precondition guarantees the pod holds no restaked value beyond what was already queued and cleared before the freeze. The standard token path (`withdrawRestakedBeaconChainETH`) reverts once disabled, so no double-payment against REL is possible.


* **Risk:** Consolidation requests are transfers of validator balance, and ETH arriving at a disabled pod is instantly withdrawable by the owner. If the keys controlling a pod are compromised, the attacker can consolidate the pod's validators to targets they control.

  * **Mitigation:** This is inherent to the key-rotation use case rather than introduced by it — a compromised key already controls the pod. The protocol guarantees only that disable cannot be used to evade slashing or extract more than the pod's own value. Disabled-pod consolidation is restricted to the Pod Owner (the Proof Submitter is barred), keeping the highest-consequence action under direct owner control. Operationally, a Pod Owner rotating compromised keys is in a race to consolidate to their new validators before the attacker does; this should be understood before relying on the feature for incident response.

# Impact Summary

* **Native ETH Restakers / Pod Owners:** Gain the ability to **rotate the keys** controlling a pod's validators without exiting the beacon-chain entry queue, and a clean, low-friction **exit** from EigenLayer that requires no further checkpoints. The action is irreversible and gated, so it must be used deliberately. Validator consolidation is available only after retirement; owners of actively restaking pods who wish to consolidate must retire the pod first.
* **Large Staking Providers:** Providers operating many validators against EOA-created pods gain a viable rekeying path, avoiding the multi-week, multi-million-dollar cost of exiting and re-entering the entry queue solely to change keys.
* **Proof Submitters:** The Proof Submitter cannot submit consolidation requests — `requestConsolidation` is owner-only and available only on retired pods. `verifyWithdrawalCredentials`, `startCheckpoint`, and `requestWithdrawal` are unchanged. Only the Pod Owner can call `disablePod`.
* **Operators / AVSs:** No change to slashing guarantees — the disable preconditions ensure a pod cannot disable while holding restaked value that has not been priced through the slashing-aware withdrawal queue. As a corollary, a pod that has been AVS-slashed on the beacon-chain-ETH strategy cannot currently disable at all (see [Limitation: AVS-Slashed Pods Cannot Currently Disable](#limitation-avs-slashed-pods-cannot-currently-disable)).
* **Integrators / Indexers:** Queue views remain clean (no permanently-uncompletable entries). New events (`PodRestakingDisabled`, `RestakingPermanentlyDisabled`, `QueuedWithdrawalClearedForDisabledPod`, `DisabledPodETHWithdrawn`) should be indexed, and the new `PAUSED_DISABLE_POD` pause flag tracked. Integrations that call `requestConsolidation` on actively restaking pods will revert (`RestakingNotDisabled`) after the upgrade and should be updated. New errors `StaleCheckpointSnapshot` and `RestakingNotDisabled` may surface from `disablePod` and `requestConsolidation` respectively.

# Action Plan

## Implementation Timeline

### Phase 1 — Implementation & Testing

* **Status:** Complete
* **Scope:** `EigenPod`, `EigenPodManager`, `DelegationManager` changes; unit \+ integration tests; documentation.
* **Deliverables:** `disablePod`, `disableRestaking`, `withdrawDisabledPodETH`, `clearQueuedWithdrawalsForDisabledPod`, retired-pod-only `requestConsolidation`, `TRUSTED_CHECKPOINT_TIMESTAMP` gate, `PAUSED_DISABLE_POD` pause flag, full test coverage (including `Disable_EigenPod.t.sol` integration flows such as deposit → queue → disable → consolidate → exit → sweep), updated `EigenPod.md` and `EigenPodManager.md`.

### Phase 2 — Audit & Review

* **Status:** Complete
* **Dependencies:** Phase 1
* **Scope:** External security assessment by Certora (August 2026) of the retirement and queue-clearing logic, plus internal review; governance review.
* **Findings and resolutions:**
  * *Low* — `withdrawDisabledPodETH` performed an external value transfer with no reentrancy guard. **Fixed:** the method is now `nonReentrant`.
  * *Low* — the value-conservation check could read a stale or uninitialized checkpoint snapshot for pods that had not checkpointed under the current checkpoint layout, letting such a pod pass the check while still controlling restaked value. **Fixed:** `disablePod` requires `lastCheckpointTimestamp >= TRUSTED_CHECKPOINT_TIMESTAMP` (`StaleCheckpointSnapshot`).
  * *Low* — legacy (pre-slashing-release) queued beacon-chain withdrawals are neither counted nor cleared and cannot be completed after retirement. **Documented:** see [Limitation: Legacy Queued Withdrawals Must Be Completed First](#limitation-legacy-queued-withdrawals-must-be-completed-first).
  * *Informational* — the `EigenPodStorage` gap was over-reserved for `restakingDisabled`, which packs into an existing slot. **Fixed:** `__gap` reduced from 36 to 35.
  * *Internal review* — consolidation on actively restaking pods could interfere with checkpoint-based balance accounting (see [Security Considerations](#security-considerations)). **Fixed:** `requestConsolidation` is restricted to retired pods and to the Pod Owner.
* The final audit report will be linked here on publication.

### Phase 3 — Deployment

* **Status:** Pending
* **Dependencies:** Phase 2, Protocol Council approval
* **Scope:** Mainnet upgrade of `EigenPod`, `EigenPodManager`, and `DelegationManager`, shipping as the v1.14.0 core upgrade ([PR #1758](https://github.com/Layr-Labs/eigenlayer-contracts/pull/1758)). The same release carries unrelated `EmissionsController` changes that are covered separately.

***Note:*** The `DelegationManager` gains a new external method (`clearQueuedWithdrawalsForDisabledPod`); deployment planning should confirm the contract remains within the EIP-170 code size limit after the change.

# References & Relevant Discussions

* [ELIP-009: MOOCOW — Massively Optimized Operations for Consolidation and Withdrawals](./ELIP-009.md)
* [EIP-7251: Increase the MAX\_EFFECTIVE\_BALANCE](https://eips.ethereum.org/EIPS/eip-7251)
* [EIP-170: Contract code size limit](https://eips.ethereum.org/EIPS/eip-170)
* [EigenLayer Contracts PR #1758 — v1.14.0: permanent EigenPod retirement](https://github.com/Layr-Labs/eigenlayer-contracts/pull/1758)
* Forum discussion — TODO
