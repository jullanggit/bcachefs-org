# Changeable Resize Design

## Goal

Make `device resize` behave like a reconcile-style "set the target" interface:

- userspace requests the desired final device size
- the filesystem converges toward the latest request
- a newer request supersedes older in-flight requests
- older synchronous callers may be told `-ECANCELED`
- grow and shrink use the same target field and the same request path

This document records the semantics agreed on before implementation.

## Core Semantics

`target_nbuckets` is the single persistent requested-size field.

Its meaning is:

- `target_nbuckets == 0`: idle, equivalent to `target_nbuckets == nbuckets`
- `target_nbuckets == nbuckets`: idle
- `target_nbuckets < nbuckets`: shrink requested/in progress
- `target_nbuckets > nbuckets`: grow requested/in progress

The intended interface is not "start one shrink operation and wait for that exact operation to finish". It is "change the target size to this value".

Consequences:

- a new request always replaces the previous requested target
- the latest request wins
- if an earlier caller is still waiting when a later request arrives, the earlier caller may return `-ECANCELED`
- it is acceptable if a request that completed only moments earlier is still reported to userspace as `-ECANCELED` because a newer request was queued before the waiter observed completion
- it is acceptable if a failure from an older request is masked by a newer request

The interface is intentionally coarse-grained. It does not attempt to preserve a precise per-request completion history for userspace.

## Normalized Target Helpers

Raw `target_nbuckets == 0` is kept as an on-disk idle sentinel. Callers should not interpret the raw field directly.

Implementation should add helpers with semantics equivalent to:

```c
target = ca->mi.target_nbuckets ?: ca->mi.nbuckets;
resize_pending = target != ca->mi.nbuckets;
is_shrinking = target < ca->mi.nbuckets;
is_growing = target > ca->mi.nbuckets;
```

Shrink-only behavior must use these helpers, not a raw `target_nbuckets != 0` check.

In particular, all of the following need to switch to helper-based checks:

- allocator tail exclusion
- stale cached-pointer dropping past the shrink cutoff
- journal metadata spill fallback
- btree metadata spill fallback
- mount/recovery resize resume logic

This is required so that:

- `target > nbuckets` behaves as a grow request instead of as "shrinking"
- changing from shrink to grow or cancel immediately relaxes the shrink cutoff

## Single Worker Model

Resize execution becomes background-owned, with exactly one serialized resize worker per device.

The resize worker is the only context allowed to perform the final grow or shrink commit.

This avoids stale local state from the current synchronous shrink path, where a caller snapshots a single `new_nbuckets` and can otherwise carry that stale value all the way to final truncate.

The worker may be implemented as per-device `work_struct` or equivalent serialized background execution.

## Minimal In-Memory Coordination State

Per device, the design only requires:

- `u64 resize_seq`
- `int resize_status`
- `wait_queue_head_t resize_wait`
- serialized background execution state, e.g. `struct work_struct resize_work`

`resize_seq` is incremented for every new resize request.

`resize_status` stores the current request state:

- `-EINPROGRESS` while the latest request is still being processed
- `0` on success
- another error on failure

No separate `resize_done_seq` is required. If a newer request races with completion, earlier waiters may observe `-ECANCELED`.

No separate `resize_running` flag is required for correctness.

## Request Path

Every resize request follows the same path, regardless of whether it currently implies grow, shrink, or idle:

1. Validate the requested size against device bounds and minimum shrink limits.
2. Persist `target_nbuckets = requested`.
3. Make the new target visible in memory.
4. Increment `resize_seq`.
5. Set `resize_status = -EINPROGRESS`.
6. Wake/queue the per-device resize worker.
7. If the ioctl is synchronous, wait for either:
   - `resize_seq != my_seq`, then return `-ECANCELED`
   - `resize_status != -EINPROGRESS`, then return that status

This preserves primarily synchronous userspace semantics while still allowing new requests to supersede old ones.

## Worker Loop

The worker repeatedly converges toward the latest target:

1. Read the normalized target and snapshot `seq = resize_seq`.
2. If `target == nbuckets`, there is nothing to do; report success and sleep.
3. If `target > nbuckets`, run the grow path toward `target`.
4. If `target < nbuckets`, run the shrink path toward `target`.
5. If, at any cancellation checkpoint, `resize_seq != seq`, abandon the current pass and restart from step 1 with the latest target.

The important semantic is that the worker does not "finish what it started" once superseded. It switches over to the newest request at explicit checkpoints.

## Grow Semantics

`target > nbuckets` means "grow to target".

If the previous request was shrinking, updating `target_nbuckets` to a grow target must immediately stop shrink-only behavior:

- allocator tail exclusion must stop applying
- shrink-specific stale-pointer rejection past the old cutoff must stop applying
- shrink-specific metadata spill fallback logic must stop applying

The grow path then commits the new larger `nbuckets` in the normal way.

If the target changes again during grow, the worker restarts from the latest target after a cancellation checkpoint.

## Shrink Semantics

`target < nbuckets` means "shrink to target".

The shrink path still needs the existing mechanics:

- persist the requested target before excluding new allocations
- close open buckets beyond the cutoff
- queue reconcile scans so the tail is rediscovered
- move journal buckets out of the tail
- wait for the tail to drain
- fence again before final truncate
- truncate alloc/accounting state and then commit the smaller `nbuckets`

However, the current monolithic shrink flow must be made restartable.

The worker must check for superseding requests at least:

- after observing/persisting the target
- after the initial journal relocation
- inside the tail-drain wait loop
- before the final journal fence
- under `state_lock` immediately before the final truncate/accounting commit

If superseded at any checkpoint, the current shrink pass is abandoned and restarted from the latest target.

## Final Commit Invariant

The final grow or shrink commit must re-check the current normalized target under `state_lock` before modifying committed size state.

For shrink, commit is allowed only if:

- the current normalized target still equals the target this pass is using
- that target is still `< nbuckets`

Otherwise the worker must not truncate and must restart from the latest target.

This is the key invariant that prevents a stale shrink pass from completing after userspace has requested cancel or grow.

## Cancellation Semantics

"Cancellation" in this design means:

- earlier waiting callers may return `-ECANCELED`
- stale worker passes do not commit obsolete work
- newly persisted targets change filesystem behavior immediately at the helper/check level

Some already-started work may still complete:

- an extent move already in flight may finish
- journal relocation already started may finish
- reconcile may touch some stale scan state before it notices the wakeup/restart

This is acceptable. The important requirement is not zero wasted work. The requirement is:

- no stale final shrink commit
- no continued enforcement of an obsolete shrink cutoff once the new target is visible
- rapid restart toward the newest target

## Mount And Recovery

Mount/recovery logic should use the same normalized target semantics:

- normalized target `== nbuckets`: nothing to do
- normalized target `> nbuckets`: resume or complete grow
- normalized target `< nbuckets`: resume shrink once the required btrees/reconcile infrastructure is available

This replaces the current shrink-only interpretation of `target_nbuckets`.

## Testing Requirements

Implementation should add tests covering at least:

- shrink retargeted to a smaller shrink target
- shrink retargeted to a less aggressive shrink target
- shrink retargeted to current size, cancelling the shrink
- shrink retargeted to a grow target
- grow retargeted back to shrink
- remount/recovery with a persisted target in each of the idle, grow, and shrink states
- repeated retargeting while the worker is in the tail-drain loop

The critical property to verify is that the final committed size always converges to the latest request, not to an obsolete intermediate one.
