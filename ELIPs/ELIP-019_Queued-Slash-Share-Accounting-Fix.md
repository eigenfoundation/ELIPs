# EigenLayer Improvement Proposal-019: Queued Slash Share Accounting Fix

| Author(s) | Created | Status | References | Discussions |
| :---- | :---- | :---- | :---- | :---- |
| ELHAJIN (Amin), Nadir Akhtar | 2026-07-13 | `merged` | [DelegationManager PR (private #84)](https://github.com/Layr-Labs/eigenlayer-contracts-private/pull/84) | [Forum Announcement](https://forum.eigenlayer.xyz/t/merged-elip-019-delegation-manager-upgrade-pre-execution/14852), [Forum Retro](https://forum.eigenlayer.xyz/t/merged-elip-019-delegation-manager-upgrade-retro/14858) |

## Executive Summary

This minor proposal corrects a wei-scale over-accounting bug in the `DelegationManager`: when a slash lands while stakers have withdrawals in the slashable delay window, the protocol could burn or redistribute marginally more shares than the withdrawal queue actually backs. The over-count is dust-sized per slash but grows with the number of queued withdrawals against the operator in a given strategy.

## Motivation

When a delegated staker queues a withdrawal, the shares stay slashable for `MIN_WITHDRAWAL_DELAY_BLOCKS`. To account for this, the `DelegationManager` records the *scaled* shares entering the operator's slashable queue in `_cumulativeScaledSharesHistory`, and at slash time `_getSlashableSharesInQueue` re-multiplies the cumulative total by the operator's magnitude to reconstruct how much of the queue is still slashable. That figure is added to `totalDepositSharesToSlash` and burned or redistributed.

Previously the queue recorded the raw `scaleForQueueWithdrawal` value — `floor(depositShares · scalingFactor)`. The backing actually removed from the operator (via `_decreaseDelegation`) is `withdrawableShares` = `calcWithdrawable(depositShares, slashingFactor)`, which re-multiplies that scaled value by the operator's magnitude and floors a second time. The over-count comes from a single rounding mismatch in how those per-staker amounts are aggregated:

**Floor-of-sum vs. sum-of-floors.** At slash time the queue reconstruction re-multiplies the ***cumulative* recorded total** by magnitude and floors **once**:

>`floor(Σ scaledSharesᵢ · magnitude)`

The backing actually removed is instead a **sum of per-staker floors** :
>`Σ floor(scaledSharesᵢ · magnitude)` 

Each staker's withdrawable amount was floored individually when its withdrawal was queued. Since `floor(Σ xᵢ) ≥ Σ floor(xᵢ)`, the reconstructed queue figure sits **at or above** the true backing, and the gap grows by up to *one wei* per a queued withdrawal (the sum is over queue entries against the operator in this strategy, note that one staker queuing multiple withdrawals contributes multiple terms as well). For a single queued withdrawal the two are identical; the divergence exists only in aggregate, as an artifact of summing many independently-floored values.

The resulting flow is:
> `totalDepositSharesToSlash` → `increaseBurnOrRedistributableShares` → `clearBurnOrRedistributableSharesByStrategy` → `strategy.withdraw`

So the excess is *over-burned or over-redistributed* — that excess is **unbacked**: the burn/redistribution path pays underlying out of the strategy's shared pool (`strategy.withdraw`), so burning more shares than the operator's account backs spends tokens that belong to the strategy's other stakers.

> Example: a strategy holds 100 shares — 40 delegated to operatorX by staker1 and 60 undelegated belonging to staker2; a slash against operatorX that over-counts by 1 burns 41, when staker2 later tries to withdraw their 60, the withdrawal reverts because only 59 are left.

The over-count is wei-scale and there is no path to draining user funds directly, but it accrues against the stakers whose funds sit in the strategy, which can leave the last withdrawer(s) facing a shortfall, and breaks a share-accounting invariant that the burn/redistribution path relies on. The invariant should hold exactly.

## Features and Specs

The fix records the queued slashable shares by **inverting the exact quantity removed from the operator**, rather than a raw scaled value that skips the magnitude round-trip. In `DelegationManager._removeSharesAndQueueWithdrawal`:

```solidity
// Before: raw scaled shares — floor(depositShares · scalingFactor), skips
// the magnitude round-trip the removed backing goes through
_addQueuedSlashableShares(operator, strategies[i], scaledShares[i]);

// After: derived by inverting withdrawableShares (the exact amount removed
// from the operator via _decreaseDelegation), so the slash-time re-multiply
// reconstructs no more than what was actually removed.
uint256 slashableScaledShares =
    slashingFactors[i] == 0 ? 0 : withdrawableShares[i].divWad(slashingFactors[i]);
_addQueuedSlashableShares(operator, strategies[i], slashableScaledShares);
```

The `slashingFactors[i] == 0` guard returns `0` when the operator has been fully slashed (magnitude `0`): there is nothing left in the queue to slash, and it avoids a division by zero.

This is a single-behavior change to one contract. It ships as a proxy-implementation upgrade of `DelegationManager` in release `v1.13.1` (three-phase upgrade: EOA deploy → multisig queue → multisig complete after timelock). No interface, storage layout, or event changes.

## Rationale

The recorded scaled shares exist so that, at slash time, `_getSlashableSharesInQueue` can reconstruct the still-slashable backing by multiplying the cumulative record by the operator's magnitude. For that reconstruction to never exceed reality, the recorded value must be the magnitude-*inverse* of the backing that was actually removed from the operator — not a separately floored quantity that skips the round-trip.

Recording `withdrawableShares.divWad(slashingFactor)` makes the record the inverse of the exact amount removed via `_decreaseDelegation`. Re-multiplying it by magnitude at slash time then reconstructs no more than that backing, by construction — the queue's contribution to a slash is bounded by what the queue actually holds. This is the minimal change that restores the invariant without touching the slashing or redistribution paths themselves.

## Security Considerations

The bug is a wei-scale over-burn/over-redistribution — dust per slash, growing with the number of queued withdrawals against the operator in the strategy with each queue contributing 1 wei at max, with no path to draining user funds — but it is a correctness defect in a share-accounting invariant, and the fix removes it at the source for ERC-20/LST strategies. This change has been reviewed and approved by the external auditors; their findings and the accepted residuals are recorded below.

**Validation.** The invariant under test is `queueSlashable ≤ removedFromOperator` — the queue may never claim more slashable shares than were actually removed from `operatorShares`. It is exercised with exact-wei bounds:

- **LST double-count** — `DelegationUnit.t.sol::test_slashOperatorShares_DoesNotDoubleCountRoundedQueueDust`: 100 stakers, slash → queue → slash. Under the old accounting the second slash re-counted the same rounded-down dust; with the fix, `getSlashableSharesInQueue` returns only backed shares and total burned never exceeds delegated.
- **LST fuzz** — `testFuzz_slashOperatorShares_QueuedSharesAreBackedByRemovedShares` and `QueueSlashAccounting.t.sol` (`Integration_QueueSlashAccounting`) assert `queueSlashable ≤ removedFromOperator` across randomized magnitudes and staker counts.

`_cumulativeScaledSharesHistory` has exactly one writer (`_addQueuedSlashableShares`, single call site), so no sibling path records with the old pattern — the fix is complete on the LST path, not local.

**Accepted residual — beacon chain.** The auditors' PoC shows the invariant can *still* be violated for `beaconChainETHStrategy` when `beaconChainSlashingFactor < 1`: the queued value is recorded by dividing `withdrawableShares` by the *full* `slashingFactor = maxMagnitude · beaconChainSlashingFactor`, but `_getSlashableSharesInQueue` re-derives the slashable amount by multiplying by `maxMagnitude` alone. The `beaconChainSlashingFactor` therefore does not cancel, and the queued figure can exceed the shares removed from `operatorShares`. This is a pre-existing known asymmetry, not introduced by this fix, and it is **currently inert**: beacon-chain slashed shares accumulate only in `EigenPodManager.burnableETHShares`, which is written but never consumed (no burn/withdraw path reads it). Beacon-share burning remains disabled, so the over-count has no realizable effect. If beacon burning is ever enabled, this asymmetry must be closed first — a code comment at `_getSlashableSharesInQueue` documenting the dependency is recommended.

**Transition window.** The fix is not retroactive. After the upgrade, `_cumulativeScaledSharesHistory` will briefly hold a mix of pre-upgrade entries (recorded as raw `scaledShares`) and post-upgrade entries (recorded as `withdrawableShares / slashingFactor`). Until `MIN_WITHDRAWAL_DELAY_BLOCKS` have elapsed since the last pre-upgrade queue event, a slash may still observe the old-formula over-accounting. This introduces no new risk and self-heals once the window passes; the auditors did not recommend pausing queuing/slashing to force a clean cut, judging it too strict for a wei-scale effect.

**No under-slashing.** Because the pre-fix behavior over-counted, the auditors checked whether the corrected calculation could now let a staker *under*-pay. It cannot: if `withdrawableShares` rounds down at queue time, completion computes `sharesToWithdraw` from the same rounded `scaledShares · slashingFactor` path, so the staker's realized withdrawal rounds down in lockstep. The new `slashableScaledShares` calculation never lets a staker withdraw value that was skipped at queue time.

**Upgrade scripts.** The auditors reviewed the upgrade scripts as they stood at the audited commit (`b2eebba`) and found them structurally consistent with prior releases, with no issues. Note that the deploy-at-execution and CREATE2 address-pinning mechanics used for the actual `v1.13.1` upgrade were introduced *after* the audit window and were therefore not part of that review.
