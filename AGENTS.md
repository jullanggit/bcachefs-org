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

## Notes
### bcachefs
- `bch2_write_super()` copies the filesystem-global superblock (`c->disk_sb.sb`) onto per-device superblocks before writing them, except for single-device fields like journal buckets. If you want a non-journal superblock change to persist, update `c->disk_sb.sb`, not just `ca->disk_sb.sb`.
- `BTREE_ID_backpointers` keys are in device+sector space, not device+bucket space. When a shrink cutoff starts in bucket coordinates, translate it before touching backpointer ranges.
- Shrink can need fresh metadata allocations before reconcile has moved any data. If `metadata_target`/`foreground_target` only points at the device being shrunk, internal metadata writes such as reconcile scan cookies, btree rewrites, and journal buckets need a spill-to-anywhere fallback or the shrink can deadlock on `no_buckets_found` before evacuation starts.
