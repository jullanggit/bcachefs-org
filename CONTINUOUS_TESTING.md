## Relevant properties
- clearly possible shrinks should always succeed and not be unnecessarily stalled
- clearly impossible shrinks should fail reasonably fast
- previous/in-progress resizes should not influence new resizes beyond a short additional delay
- operation history should converge to the latest requested resize state, not to stale intermediate requests
- the filesystem should remain mountable and fsck-clean across long mixed-operation runs
- failures must be replayable from a saved seed plus an operation log

## Proptest Fit
`proptest` is a good fit for this framework, with one important caveat: it should generate the controller's operation sequence, not try to model the kernel's scheduler.

That means:
- the generated test case is a sequential control plan
- the harness itself creates real overlap by leaving background workers running while later operations execute
- shrinking still works on the operation history, even though the exact runtime interleaving remains nondeterministic

This matches the intended design of:
- one synchronous controller thread
- multiple asynchronous/background activities that stay live across controller steps

`proptest-state-machine` is a plausible starting point because the test naturally has:
- an abstract reference model
- generated transitions
- invariants that should hold after every controller step

Current limitation:
- `proptest-state-machine` is sequential-only, so it will not systematically explore thread schedules

That is acceptable here as long as the framework is explicit that:
- `proptest` explores operation histories
- background worker timing is treated as a stress dimension, not as a fully modeled state-space dimension

## Rust Model Sketch
The test harness should be a Rust controller with a small state model and a set of long-lived background workers.

### Controller Responsibilities
- generate operations with `proptest`
- maintain the reference model
- start and stop background workers
- issue synchronous foreground operations
- classify outcomes as expected success, expected failure, or unknown
- run checkpoints and invariant validation
- emit structured logs and a replayable operation transcript

### Background Workers
Prefer a fixed set of worker profiles instead of generating arbitrary thread topologies.

Candidate worker profiles:
- file churn: create/write/rename/delete loops
- readback worker: checksum/verify existing files
- `fio` randrw worker
- `fsstress` worker
- snapshot churn worker
- mount option toggle worker, if safe to model

Each worker should have:
- a stable worker id
- a small configuration chosen by the generator
- explicit start/stop operations
- bounded shutdown logic so failing runs do not leak workload processes

### Generated Operations
Likely operation enum:
- `StartWorker(WorkerSpec)`
- `StopWorker(WorkerId)`
- `Resize { dev, target, mode }`
- `AddDevice { dev }`
- `RemoveDevice { dev }`
- `SetReplication { data, metadata }`
- `CreateSnapshot { src, dst }`
- `DeleteSnapshot { path }`
- `Wait { ms }`

The important point is that controller operations stay coarse-grained and semantically meaningful. The generator should not try to produce raw shell commands.

Generated operations should be actual filesystem mutations, not assertions. Checks such as remounts, offline `fsck`, and usage sampling belong in the assertion/checkpoint layer that runs around selected operations and at case boundaries. Treating those checks as generated transitions muddies the model and weakens shrinking.

### Reference Model
The model should track only what is needed to decide whether strong assertions are valid:
- current member devices and nominal sizes
- persisted/latest requested resize targets
- whether the filesystem is mounted
- which background workers are running
- a conservative free-space estimate
- snapshot count / pinned-space estimate
- current replication policy

The model should classify requested shrinks into:
- `ClearlyPossible`
- `ClearlyImpossible`
- `Unknown`

Only the first two classes get hard outcome assertions. The `Unknown` class is still useful for stress, but should not produce false failures.

### Invariants And Checkpoints
After selected operations, and always at the end of a run, check:
- filesystem is still mountable with the full device set when it should be
- offline `fsck` is clean
- latest requested resize target converges eventually
- superseded resize requests do not stall later requests for long
- running workers can be stopped cleanly

Useful checkpoint policy:
- cheap checks after most controller steps
- heavier checks after topology changes and at end-of-run
- full `fsck` and remount before declaring success
- restore the starting topology between generated cases so later cases and proptest shrink reruns do not inherit stale device membership

## Logging And Replay
The framework should produce:
- newline-delimited JSON or similarly structured logs
- a human-greppable error summary
- saved `proptest` seed / regression case
- a complete operation transcript
- per-command stdout/stderr snippets for failures

Errors should be explicitly tagged, for example:
- `ERROR unexpected_resize_success`
- `ERROR unexpected_resize_failure`
- `ERROR fsck_not_clean`
- `ERROR convergence_timeout`
- `ERROR worker_shutdown_timeout`

## Where It Should Run
The test should run inside a `ktest` guest. `ktest` should remain the outer harness.

Reasoning:
- this is a filesystem/kernel integration test, not just a userspace property test
- `ktest` already solves kernel build, VM launch, scratch device provisioning, dependency installation, and output capture
- replacing `ktest` would mean reimplementing block-device setup, guest lifecycle handling, and log collection
- host-side orchestration over ssh would add another distributed control layer without improving the core test semantics

So the layering should be:
- outer layer: `build-test-kernel`/`ktest`
- inner layer: guest-side Rust continuous-test binary
- wrapper: a `.ktest` file that provisions devices and invokes the Rust binary

## Implementation Location
Prefer a dedicated Rust test crate under the `ktest` tree, launched by a dedicated `.ktest` wrapper.

Good shape:
- `ktest/tests/fs/bcachefs/continuous.ktest`
- `ktest/tests/fs/bcachefs/continuous/`

Why this is preferable:
- the test belongs to the VM test environment, not to the kernel tree itself
- it can reuse existing `.ktest` dependency patterns and scratch-device setup
- it keeps guest-workload code separate from host-side CI utilities

Avoid making it replace `ktest`, and avoid putting it in the kernel repo root as if it were part of the kernel build.

If needed later, a host-only smoke mode could exist for faster local iteration with loop devices, but that should be a secondary convenience mode. The primary/authoritative version should stay in `ktest`.

## Practical Shape
Initial version:
- one `.ktest` test case that formats a filesystem, mounts it, and runs the Rust controller once
- controller starts a small set of background workers
- generator issues resize and topology operations between checkpoints
- end-of-run always does worker shutdown, unmount, offline `fsck`, and remount

Later extensions:
- multiple predefined stress profiles
- longer overnight runs with persisted regression seeds
- CI mode with bounded case count
- dedicated reproducer mode that replays one saved operation transcript exactly
