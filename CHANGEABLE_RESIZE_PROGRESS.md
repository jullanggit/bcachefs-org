# Changeable Resize Progress

## 2026-04-14

Current status:

- traced the existing shrink/resize flow in `bcachefs/fs/bcachefs/init/dev.c`
- traced the current recovery resume path in `bcachefs/fs/bcachefs/init/fs.c`
- identified current `target_nbuckets` users that assume "nonzero means shrinking"
- wrote `CHANGEABLE_RESIZE_DESIGN.md` to capture the agreed semantics before implementation

Agreed design at this point:

- `target_nbuckets` is the single persistent requested-size field
- `0` and `== nbuckets` are idle
- `< nbuckets` is shrink
- `> nbuckets` is grow
- newer requests supersede older ones
- older synchronous callers may return `-ECANCELED`
- a per-device background worker owns actual resize execution
- the worker abandons stale passes at explicit checkpoints instead of finishing obsolete work

Next implementation steps:

1. Add retargeting-focused ktests that exercise shrink-to-shrink, shrink-to-cancel, and shrink-to-grow transitions.
2. Run the new tests and existing shrink coverage until the new request/worker path is stable.
3. Split and record the resulting kernel and ktest changes into reviewable commits.

Notes:

- compatibility with older out-of-tree `target_nbuckets` semantics does not need to be preserved because this has not landed upstream yet
- userspace visibility is intentionally coarse-grained; precise per-request completion history is not required

Implementation progress since the initial design write-up:

- added normalized resize-target helpers and switched existing shrink-only checks to use them
- updated member validation to accept grow targets in `target_nbuckets`
- converted device resize requests to a target-setting path with per-device request sequencing
- added a per-device background resize kthread plus waitqueue/status handoff for synchronous ioctls
- refactored shrink execution so stale passes bail out on explicit restart checkpoints instead of finishing obsolete targets
- converted recovery resume to the unified resize-resume path
- fixed read-only/shutdown interaction by stopping resize workers before clean shutdown; otherwise a persisted resize could keep committing alloc/accounting work during unmount and trip write-path assertions
- kernel build check passes with `direnv exec . make W=1 O=out -j$(nproc) fs/bcachefs/`
- targeted shrink coverage now passes:
  - single-run confirmation of `online_two_device_restart_resume_shrink`
  - single-run confirmation of `online_two_device_shrink_retarget_larger`, `online_two_device_shrink_retarget_cancel`, and `online_two_device_shrink_retarget_grow`
  - four consecutive confirmation passes of `online_two_device_restart_resume_shrink`, `online_two_device_shrink_retarget_larger`, `online_two_device_shrink_retarget_cancel`, and `online_two_device_shrink_retarget_grow`
