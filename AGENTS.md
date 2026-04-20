The goal of these repositories is to implement filesystem shrinking for bcachefs.
They each use their own nix flakes to get the required dependencies, so make sure to use the correct ones. The kernels devshell is located elsewhere, but direnv still works.
When editing the bcachefs source, you do not have to update the userspace mirror in the bcachefs-tools repository.

Whenever you discover something that would be good to have in this AGENTS.md file, add it.

Always describe your changes and make them understandable to me after you're done. Explain in detail with reasoning for why you made these changes.
Document anything tricky or noteworthy directly in the code. The end goal of these changes is to get merged into mainline, any aid in understandability is good.

After you're done with your changes, a (or multiple) commit(s) with messanges (with further body if neccessary) describing your changes. Unlike your description in the chat, these should not be targeted at me, but the upstream maintainer. Please do split out logically separate changes into multiple commits, make them as easy to review as possible.

## Commands
### general
Use jj for version control, commit whenever you find it appropriate, you can also rewrite the history of your _own_ commits if you find it appropriate.
Keep the commit title on the first line; put any extended explanation in the commit body after a blank line.
The kernel source is a separate repo in `bcachefs/`; run `jj` there when checking status or committing kernel changes.
### bcachefs
check if it compiles - `make W=1 O=out -j (nproc) fs/bcachefs/`
### ketst
- run shrink tests
- after both of the commands below the test name(s) to run can be specified, if only some of them should be run. For this, strip the test_ prefix from the test name. (./build-test-kernel run ... online_two_device_data_shrink)
- never blindly wait for one of these commands to exit, as they can hang indefinitely on error.
#### single-run
`./build-test-kernel run -K -k ../bcachefs tests/fs/bcachefs/shrink.ktest`
#### until failure
This command runs the test(s) in a loop until a failure is encountered. Wait for a number of iterations that seems appropriate. For final confirmations, at least four times.
`./build-test-kernel run -L -K -k ../bcachefs tests/fs/bcachefs/shrink.ktest`
- remember to _always_ stop the process(es) after you've got the information you wanted, as -R keeps them alive until explicitly stopped, and they eat up a lot of memory, potentially crashing the system.
#### multi-run
Until now more useful as some of the tests are quite flaky, and might even have multiple different errors that might occur.
This command runs the test(s) in a loop until interrupted, continuing after failures. This is mainly useful, if there are multiple errors, if you want to confirm you fixed one of them.
`./build-test-kernel run -R -K -k ../bcachefs tests/fs/bcachefs/shrink.ktest`
- remember to _always_ stop the process(es) after you've got the information you wanted, as -R keeps them alive until explicitly stopped, and they eat up a lot of memory, potentially crashing the system.
#### general
- when adding or changing shrink ktests, prefer an explicit remount after `fsck` with the full member-device list. Resize/shrink changes both allocator state and per-device superblocks, and the remount catches reopen regressions that a clean `fsck` alone can miss.
- `build-test-kernel -B <dir>` can reuse another kernel `O=` tree, but only when both trees were built under a compatible userspace/toolchain. Mixing build dirs across different devshell/glibc revisions can break host tools such as `objtool` or `extract-cert`; if that happens, fall back to a fresh per-ktest build dir.
- For larger Rust-based filesystem stress/property tests, keep `ktest` as the outer VM/device harness and run the Rust controller inside the guest from a `.ktest` wrapper. Replacing `ktest` would duplicate scratch-device provisioning, guest lifecycle handling, and log collection.
#### ssh
You can run `./build-test-kernel ssh` to ssh into a running test instance and inspect it.

## Notes
### bcachefs
- `bch2_write_super()` copies the filesystem-global superblock (`c->disk_sb.sb`) onto per-device superblocks before writing them, except for single-device fields like journal buckets. If you want a non-journal superblock change to persist, update `c->disk_sb.sb`, not just `ca->disk_sb.sb`.
- Pending shrink state is persisted in each member's `target_nbuckets`. If a shrink is interrupted after that field is written, remounting read-write should resume the shrink from mount/recovery rather than requiring the ioctl to be reissued.
- Filesystem-owned resize workers must be stopped before `bch2_fs_read_only()` reaches clean shutdown. Otherwise a persisted resize can keep issuing transactional alloc/accounting updates during unmount and trip write-path assertions after the filesystem has started going read-only.
- `BTREE_ID_backpointers` keys are in device+sector space, not device+bucket space. When a shrink cutoff starts in bucket coordinates, translate it before touching backpointer ranges.
- `BTREE_ID_need_discard` is keyed by journal sequence plus encoded bucket position, not by device bucket space. Tail cleanup for shrink/truncate must scan and filter decoded bucket positions instead of trying to range-delete by `(dev, bucket)`.
- Shrink can need fresh metadata allocations before reconcile has moved any data. If `metadata_target`/`foreground_target` only points at the device being shrunk, internal metadata writes such as reconcile scan cookies, btree rewrites, and journal buckets need a spill-to-anywhere fallback or the shrink can deadlock on `no_buckets_found` before evacuation starts.
- If shrink has to defer discard work to avoid resize/reconcile deadlocks, only defer buckets in the shrinking tail at or beyond the current `target_nbuckets`. Deferring the whole device leaves retained buckets stuck in `need_discard`, which turns `online_three_device_variable_buckets_shrink` from a seconds-scale test into a minutes-scale regression.
- In shrink's final commit path, flushing `journal_cur_seq()` is enough; `bch2_journal_flush_all_pins()` can wait behind unrelated reconcile/key-cache pins that are created after the tail is already empty and turn the ioctl into a multi-minute stall.
- A shrink-triggered reconcile kick can keep draining unrelated global work long after the truncating tail is already empty. Poll tail emptiness while waiting instead of requiring the whole kick to drain before shrink can commit the new cutoff.
- Shrink-triggered `RECONCILE_SCAN_device` work should start at the current tail (`target_nbuckets`) in backpointer key space. Re-scanning the whole device pointlessly requeues metadata below the retained region and can turn `online_three_device_variable_buckets_shrink` into a minute-scale run.
- Live foreground IO can keep a shrinking tail non-empty for much longer than a couple of reconcile kicks even when enough space exists. When deciding that shrink is truly stuck, key the ENOSPC decision off stalled tail progress rather than a small fixed retry count or a wall-clock timeout.
- The first remaining tail bucket can stay unchanged while shrink is still making space deeper in the tail. Liveness heuristics should watch aggregate tail-backpointer progress as well as the head bucket, slice shrink waits so one long reconcile kick cannot hide that progress behind minute-scale stalls, and count only completed no-progress reconcile passes that actually scanned or processed shrink work.
