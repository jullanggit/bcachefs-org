# Bcachefs Shrink Specification

This document describes how online device shrink currently works in this tree: what state represents an in-flight shrink, which subsystems react to it, how the resize worker drains the tail, and which invariants keep a stale shrink from committing.

It is intentionally implementation-oriented. The goal is to let someone change shrink without re-deriving the control flow from `bcachefs/fs/bcachefs/init/dev.c` and the surrounding allocator/reconcile/journal hooks.

## Scope

This spec covers the current online `device resize` implementation in the kernel tree under `bcachefs/fs/bcachefs/`.

It focuses on shrinking, but it includes the shared resize machinery where shrink depends on it:

- persistent requested-size state in the superblock
- the per-device resize worker and superseding-request semantics
- allocator, discard, journal, btree, and reconcile behavior while shrink is in progress
- mount/recovery resume behavior

This is a description of the code as it exists now, not a proposal for a different interface.

## Terms

- `nbuckets`: the committed current size of a member device.
- `target_nbuckets`: the requested size persisted in the member superblock.
- `cutoff`: the first bucket that must disappear after shrink completes. Buckets `>= target_nbuckets` are in the shrink tail.
- `tail`: the region `[target_nbuckets, nbuckets)`.
- `normalized target`: `target_nbuckets ?: nbuckets`.

Most of the code uses the normalized target, not the raw on-disk field. `target_nbuckets == 0` is only an idle sentinel on disk, that exists for kernel-upgrade compatibility.

## State Model

### Persistent State

Each member stores both the committed size and the requested size in the superblock members area:

- `nbuckets`
- `target_nbuckets`

Current semantics are:

- `target_nbuckets == 0` and  `target_nbuckets == nbuckets`: idle
- `target_nbuckets < nbuckets`: shrink requested or in progress
- `target_nbuckets > nbuckets`: grow requested or in progress

The helper layer in `bcachefs/fs/bcachefs/sb/members.h` normalizes `0` to `nbuckets`:

- `bch2_dev_resize_target()`
- `bch2_dev_resize_pending()`
- `bch2_dev_is_shrinking()`
- `bch2_dev_is_growing()`

That normalization matters because many subsystems need shrink-specific behavior to stop immediately when userspace retargets a device back to its current size or to a grow target.

### In-Memory Coordination

Each `struct bch_dev` also carries serialized resize-worker state in `bcachefs/fs/bcachefs/init/dev.c`:

- `resize_thread`
- `resize_lock`
- `resize_seq`
- `resize_status`
- `resize_wait`

The intent is:

- exactly one worker context owns resize execution for a device
- every new request increments `resize_seq`
- synchronous callers wait for either completion of their request or supersession by a newer request
- older waiters may observe `-ECANCELED` if a newer target arrives first

The implementation does not try to preserve a detailed per-request completion history. It converges to the latest target.

## External Entry Points

Userspace enters through the disk resize ioctls in `bcachefs/fs/bcachefs/init/chardev.c`:

- `bch2_ioctl_disk_resize()`
- `bch2_ioctl_disk_resize_v2()`

Both route to:

- `bch2_dev_resize()`

Mount/recovery resume uses:

- `bch2_fs_resize_on_mount()` in `bcachefs/fs/bcachefs/init/fs.c`
- `bch2_dev_resize_resume()` in `bcachefs/fs/bcachefs/init/dev.c`

## Request Semantics

`bch2_dev_resize()` is a "set the requested final size" interface, not a "run this exact shrink operation to completion" interface.

The request path is:

1. Take `state_lock`.
2. Start the per-device resize thread if needed.
3. Convert a request equal to the current size into `target_nbuckets = 0`.
4. Validate and persist `target_nbuckets` with `bch2_dev_resize_update_target()`.
5. Drop `state_lock`.
6. Bump `resize_seq`, set `resize_status = -EINPROGRESS`, wake the worker, and wait.

Validation in `bch2_dev_resize_update_target()` checks:

- `BCH_MEMBER_NBUCKETS_MAX`
- minimum post-shrink size: `first_bucket + BCH_MIN_NR_NBUCKETS`
- underlying block-device capacity for grows

Persistence uses `bch2_write_super()`, so an interrupted shrink is restartable after remount as long as the request made it to disk.

## Immediate Effects Of Persisting A Shrink Target

Once `target_nbuckets < nbuckets` is visible, helper-based shrink checks start affecting allocation and metadata placement before the worker has finished evacuating the tail.

Current hooks are:

- `bcachefs/fs/bcachefs/alloc/foreground.c`
  - `__try_alloc_bucket()` rejects open-bucket allocations at or beyond the cutoff.
- `bcachefs/fs/bcachefs/data/extents.c`
  - cached pointers past the cutoff are treated as stale and dropped.
- `bcachefs/fs/bcachefs/alloc/discard.c`
  - discard skips buckets in the truncating tail. TODO: document why
- `bcachefs/fs/bcachefs/journal/write.c`
  - journal allocation falls back to the full filesystem if the preferred metadata target consists entirely of shrinking devices.
    - TODO: see if this can be tightened to only fall back if no journal allocations can be made on the device (not just blindly check if its shrinking. (This might already be the case, if so, document it.))
- `bcachefs/fs/bcachefs/btree/interior.c`
  - btree metadata allocation has the same spill-to-anywhere fallback. TODO: same as above
- `bcachefs/fs/bcachefs/data/reconcile/work.c`
  - device reconcile scans start at the shrink cutoff in backpointer key space instead of rescanning the whole device.
    - TODO: see if this is too general. It should only affect scans triggered by shrink, not all scans. This might or might not be an issue.

Those checks are what make the shrink cutoff effective as soon as the request is persisted. They also mean retargeting away from shrink must flip behavior immediately via the normalized helpers.

## Worker Model

`bch2_dev_resize_thread()` is a per-device kthread. On each wakeup it snapshots:

- the current `resize_seq`
- the normalized target from `bch2_dev_resize_target()`

It then does one of three things:

- `target == nbuckets`: nothing to do
  - TODO: reverse any effects that might still linger from shrink (for example discard delays? maybe these can also be handled more cleanly)
- `target > nbuckets`: call `__bch2_dev_grow()`
- `target < nbuckets`: call `__bch2_dev_shrink()`

The worker is restartable rather than strictly cancelable. A newer request does not necessarily undo work already performed, but it prevents stale final commit by making later checkpoints return `-EAGAIN` and restart from the latest target.

The common restart check is `bch2_dev_resize_restart_check()`, which returns:

- `-EINTR` if the worker is being stopped
- `-EAGAIN` if `resize_seq` changed

## Shrink Algorithm

### Phase 1: Enter Shrink Mode

`__bch2_dev_shrink()` first takes `state_lock` and confirms that the current pass still matches the live request:

- `new_nbuckets < old_nbuckets`
- `bch2_dev_resize_target(ca) == new_nbuckets`
- `bch2_dev_is_shrinking(ca)`

It then prepares the allocator state for the cutoff:

- `bch2_open_buckets_stop(c, ca, false, new_nbuckets)` closes open buckets in the tail.
- `bch2_reset_alloc_cursors(c)` avoids allocator churn on stale cursors.

After dropping `state_lock`, it flushes already-running discard work:

- `flush_work(&c->discards.work)`
- `flush_work(&ca->discard_fast_work)`

This matters because shrink later defers discards in the tail to avoid deadlocks with reconcile and metadata rewrite work. An older discard pass still running against the same buckets can deadlock the evacuation path.

### Phase 2: Move Journal State Out Of The Tail

`move_journal_past_cutoff()` runs before the main tail-drain loop.

The journal is handled explicitly because journal buckets are not something shrink should wait for reconcile to discover and evacuate indirectly. (TODO: maybe this should actually be handled by reconcile?) The helper:

- counts journal buckets at or beyond the cutoff
- temporarily grows the journal if more buckets are needed to relocate
- forces the current journal bucket to advance when it still sits in the tail
- flushes the journal to persist that relocation
- deletes now-obsolete journal buckets from the tail

Without this step, journal activity can keep reintroducing references into the very region shrink is trying to empty.

### Phase 3: Drain The Tail

The core shrink loop tracks tail liveness with backpointer scans.

#### Tail snapshots

Two helpers inspect the tail in backpointer key space:

- `tail_head_snapshot()`
  - records the first tail bucket, its first backpointer, and the number of backpointers in that bucket.
- `tail_progress_snapshot()`
  - records the same head data plus the total tail backpointer count.

This is done in backpointer key space because `BTREE_ID_backpointers` is keyed by device and sector, not by bucket. The shrink code converts the bucket cutoff to a backpointer start position before scanning.

#### Empty check

The loop first does a cheap head snapshot. If the tail looks empty, it flushes the btree write buffer and checks again before accepting emptiness. This avoids committing shrink while buffered btree updates still hide references that have not hit the backpointer tree yet.

#### Cached-only tail invalidation

`bch2_dev_shrink_invalidate_tail_cached()` handles a specific false-blocker case:

- live data may already have been rewritten elsewhere
- the old tail buckets may remain only as cached copies
- those cached copies still keep backpointers alive
- reconcile does not need to move them again

The helper scans alloc keys in the tail for cached buckets that are not currently open, then invalidates the matching backpointers directly. This lets shrink wait only on durable data and metadata that still genuinely pins the tail.

TODO: maybe the reconcile helper that actually moves the data can be changed to not leave behind cached copies on move, if they are in the tail.

#### Reconcile kick

After each snapshot, shrink queues reconcile work with `bch2_dev_shrink_queue_reconcile()`:

- optionally `RECONCILE_SCAN_device` for the shrinking device
- always `RECONCILE_SCAN_pending`

The device scan now starts at `target_nbuckets`, not at bucket zero. That avoids needlessly requeuing metadata below the retained region.

Unrelated device-label changes should also stay targeted while shrink is active. They only need a `RECONCILE_SCAN_device` for the relabeled member plus `RECONCILE_SCAN_pending`; escalating them to filesystem-wide or metadata-wide scans is broader than necessary and can contend badly with shrink's own reconcile/allocator work.

#### Waiting strategy

`bch2_dev_shrink_wait_reconcile()` does not simply wait for the entire reconcile kick to drain. Instead it:

- polls once per second
- rechecks whether the tail is now empty
- treats head movement as useful progress
- also returns when the requested reconcile kick has fully completed

This matters because a reconcile kick can spend a long time on unrelated global work after the shrink tail is already empty. Shrink only cares about the tail.

#### Progress heuristic and ENOSPC

Shrink does not fail after a small fixed number of passes. It tracks:

- the current head bucket and first backpointer
- total backpointer count in the tail
- whether the latest no-progress pass actually completed a reconcile kick
- whether the journal advanced during that pass
- whether the pass included a fresh device scan

The current heuristic is:

- progress resets the stall counter
- if the journal moved, assume the tail's blockers are still changing and rescan from the current cutoff instead of treating the tail as impossible
- if a no-progress pass did not include a device scan, force one
- only after repeated no-progress full scans on a journal-quiescent state does shrink declare `-ENOSPC`

The current limit is `32` stalled full rescans.

This journal-based quiescence check is conservative but not shrink-local. `journal_cur_seq()` is filesystem-global, so unrelated metadata-writing IO elsewhere in the filesystem can keep suppressing the stall counter even when the shrink tail itself is not changing. In the worst case, an impossible shrink can keep retrying indefinitely under enough unrelated write churn.

The intended follow-up is to replace the filesystem-global journal signal with a shrink-local churn signal that only tracks changes relevant to the tail blockers being evacuated.

On `-ENOSPC`, shrink calls `bch2_dev_shrink_clear_target()` before returning. That clears `target_nbuckets` back to idle so allocations are no longer blocked past the old cutoff.

### Phase 4: Finalize The Shrink

`bch2_dev_shrink_finalize()` runs under `state_lock` and revalidates that the live request still matches this pass before committing anything irreversible.

The finalization order is important:

1. Recheck `resize_seq`, target, and `bch2_dev_is_shrinking()`.
2. Flush btree interior updates.
3. Flush outstanding journal pins and then device-specific pins.
4. Flush the journal.
5. Recheck that the tail is really empty.
6. Clear `need_discard` entries for truncated tail buckets. TODO: maybe this should bedone before the emptiness check?
7. Drop superblock copies whose offsets are beyond the cutoff.
8. Truncate accounting with `bch2_dev_truncate_accounting()`.
9. Remove alloc metadata for the tail with `bch2_dev_remove_alloc()`.
10. Commit `nbuckets = new_nbuckets` in the superblock and clear `target_nbuckets` if it still matches.
11. Call `bch2_dev_buckets_resize()` so in-memory device sizing matches the newly committed superblock.
12. Recalculate filesystem capacity.

Several points here are easy to get wrong:

- Shrink waits only for pins that were already outstanding when finalization begins. Waiting for all future pins can turn shrink into an unbounded global journal drain. TODO: it does also flush device pins, not only after tail. Maybe implement a more specific flush tail pins function? Generally, document more why flushing pins is even necessary (and what pins are, but thats just for me, not to be documented here if its clear).
- `need_discard` cleanup must scan and filter by decoded bucket because `BTREE_ID_need_discard` is keyed by `(journal seq, encoded bucket)`, not by raw `(dev, bucket)`.
- Superblock copies are not discovered via the backpointer walk; `drop_sbs_after_cutoff()` handles them explicitly before the committed size changes.
- Alloc metadata is removed before the new smaller `nbuckets` becomes visible, so later transactions cannot see stale tail buckets after commit.

There is still a code comment in finalization about whether any of the grow-side `__bch2_dev_resize_alloc()` work should eventually have a shrink-side equivalent. The current shrink implementation does not call it. TODO: that code can probably be removed

## Final Commit Invariant

The key safety rule is:

- shrink may only commit the smaller `nbuckets` while holding `state_lock`, after rechecking that the current normalized target still equals the target used by this pass and is still a shrink target

That invariant prevents an obsolete shrink pass from truncating the device after userspace has already requested cancel or grow.

The worker may do wasted preparatory work on an obsolete target. It must not do a stale final truncate.

## Recovery And Remount Behavior

Shrink persistence and resume rely on `target_nbuckets`.

`bch2_fs_resize_on_mount()` has two distinct roles:

- early grow-on-mount for image expansion before the btree roots are running
- later resume of pending resize requests once the btree/reconcile infrastructure exists

Pending shrink is resumed only when:

- `BCH_FS_btree_running` is set
- the mount is not read-only
- the superblock features do not block normal rw startup (`small_image` / `no_default_sb`)

At that point the mount path calls `bch2_dev_resize_resume()` for each device whose normalized target still differs from `nbuckets`.

This means an interrupted shrink is supposed to continue automatically on the next rw mount, not wait for userspace to reissue the ioctl.

## Shutdown Behavior

Resize workers are stopped in `bch2_fs_read_only()` via `bch2_dev_resize_threads_stop()`.

That ordering matters because a persisted resize may still be performing transactional alloc/accounting work. If the filesystem enters clean shutdown first and the worker keeps running, resize can trip write-path assertions after the fs has already started going read-only.

## Interactions With Other Subsystems

### Allocator

Foreground allocation refuses buckets in the tail as soon as shrink is pending. This is the main mechanism that stops new writes from repopulating the region we are trying to remove.

### Journal

Journal allocation and relocation need special handling:

- the worker explicitly relocates existing journal buckets out of the tail
- new journal bucket allocation can spill outside the preferred metadata target while shrink is active

Without both pieces, shrink can deadlock on its own metadata needs.

### Btree Metadata

Btree node rewrites use the same spill-to-anywhere fallback as journal allocation when all preferred metadata-target devices are shrinking.

### Reconcile

Reconcile is the mechanism that rediscovers and moves tail references, but shrink does not require reconcile to become globally idle. It only needs enough completed work to make the tail empty.

### Discard

Discard work is restricted so it does not race the evacuation path on the truncating tail. Once resize finishes, `bch2_dev_resize_finish()` restarts async discards. TODO: see how much discard blocking is necessary and why these two subsystems even collide.

## Known Performance/Behavioral Properties

These are current implementation properties, not guarantees of an abstract interface:

- newer resize requests supersede older ones
- older synchronous waiters may return `-ECANCELED`
- shrink progress is judged by tail backpointer movement, not by just the first bucket
- one long reconcile kick is intentionally sliced into shorter waits so shrink can observe its own progress
- impossible shrinks clear the pending target on `-ENOSPC` so the old cutoff stops blocking allocations

## Code Map

Main code locations:

- `bcachefs/fs/bcachefs/init/chardev.c`
  - resize ioctl entry points
- `bcachefs/fs/bcachefs/init/dev.c`
  - request path
  - worker thread
  - shrink loop
  - final commit ordering
- `bcachefs/fs/bcachefs/init/fs.c`
  - mount/resume behavior
- `bcachefs/fs/bcachefs/sb/members.h`
  - normalized resize-target helpers
- `bcachefs/fs/bcachefs/alloc/foreground.c`
  - allocator tail exclusion
- `bcachefs/fs/bcachefs/alloc/discard.c`
  - discard deferral and `need_discard` cleanup
- `bcachefs/fs/bcachefs/journal/write.c`
  - journal spill fallback
- `bcachefs/fs/bcachefs/btree/interior.c`
  - btree metadata spill fallback
- `bcachefs/fs/bcachefs/data/reconcile/work.c`
  - shrink-aware reconcile scan start
- `bcachefs/fs/bcachefs/data/extents.c`
  - stale cached-pointer dropping past the cutoff

## Tests That Exercise This

Current targeted ktests live in `ktest/tests/fs/bcachefs/shrink.ktest`, including:

- `online_two_device_data_shrink`
- `online_three_device_variable_buckets_shrink`
- `online_two_device_restart_resume_shrink`
- `online_two_device_shrink_retarget_larger`
- `online_two_device_shrink_retarget_cancel`
- `online_two_device_shrink_retarget_grow`

These tests are useful because they cover both the basic evacuation/finalization path and the newer "latest target wins" worker semantics.
