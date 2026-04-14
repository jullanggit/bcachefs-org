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

1. Add helper accessors for normalized target / pending / shrinking / growing.
2. Convert all current raw `target_nbuckets` users to the new helper semantics.
3. Add per-device resize worker state and request sequencing.
4. Refactor shrink into restartable stages with cancellation checkpoints.
5. Unify grow and shrink under the same request/worker path.
6. Update mount/recovery resume logic to use the unified target semantics.
7. Add ktests covering retargeting and recovery.

Notes:

- compatibility with older out-of-tree `target_nbuckets` semantics does not need to be preserved because this has not landed upstream yet
- userspace visibility is intentionally coarse-grained; precise per-request completion history is not required
