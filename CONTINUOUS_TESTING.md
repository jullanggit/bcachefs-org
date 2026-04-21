## Relevant properties
- clearly possible shrinks should always succeed and not be unnecessarily stalled
- clearly impossible shrinks should fail reasonably fast
- previous/in-progress resizes should not influence new resizes beyond a short additional delay
- operation history should converge to the latest requested resize state, not to stale intermediate requests
- the filesystem should remain mountable and fsck-clean across long mixed-operation runs
- failures must be replayable from a saved seed plus an operation log

## Overall Goal
This harness is meant to approximate realistic, and at times deliberately extreme, filesystem usage over extended runs:
- mixed API usage, not isolated shrink microcases
- enough surface coverage to exercise real-world interactions and edge cases
- long enough runtime to catch stateful bugs that do not show up in short directed tests

The intent is for it to become a "last proof of correctness" style test: if this passes, the current shrink behavior is plausibly ready for real-world usage.

That does not mean it can be sloppy about oracles. The harness still needs to be selective about what it fails on, so it does not waste time and compute chasing nondeterministic noise. In practice:
- keep ambiguous cases for stress coverage
- only hard-fail on outcomes that are clearly wrong from the observed state
- prefer conservative assertions that remain stable under long runs and future background workloads
- record enough context that rare failures are replayable instead of one-off ghosts

## Agreed Design
The continuous harness should not replay the existing explicit shrink tests as scripts. Instead, it should absorb their dimensions into one long-running randomized system that explores combinations over many runs.

The basic structure is:
- one manager thread
- asynchronous operations by default
- long-lived background stress workers that can be toggled on and off
- observation-driven outcome checking
- randomized format-time choices at startup
- randomized runtime choices over the lifetime of the filesystem

The aim is to cover at least the same space as the current explicit tests, but through combinations and overlap rather than one hand-written scenario per test.

## Manager Loop
The harness should use one manager thread, not a sequential property-test driver.

The manager's job is:
- snapshot the relevant live filesystem state
- decide which operation to launch next
- launch operations asynchronously
- keep multiple operations in flight at once
- inspect completions against expectations derived from:
  - the state before launch
  - the state after completion
  - which other operations were already in flight
  - which newer operations superseded older ones

This is a better fit for resize semantics such as:
- newer resize requests canceling or superseding older ones
- long-running operations overlapping realistic filesystem activity
- expectations that depend on more than a single pure transition function

The oracle should stay observation-driven:
- snapshot the relevant live filesystem state before launch
- derive the expected result from that observation plus the requested operation
- let the operation run concurrently with other in-flight work
- when it completes, snapshot again and compare against the expected outcome

An explicit model can still exist, but only as a helper for operation legality or bookkeeping. It should not be the primary oracle once free-space, snapshots, replication, or concurrent supersession semantics become too rich to mirror faithfully.

## Rust Harness Sketch
The test harness should be a Rust controller with a small amount of internal bookkeeping and a set of long-lived background workers.

### Format-Time Randomization
Each case starts by randomly choosing a valid format-time profile.

Current agreed axes:
- replication level in `[1, 3]`
- erasure coding on/off, but only when valid for the chosen redundancy level
- encryption on/off
- compression off or `zstd:1`
- per-device bucket size randomly chosen from a sensible fixed set

Deliberately not a format-time axis:
- initial member count
- device labels
- foreground/background/promote targets

Instead, the filesystem should start with the minimum number of devices required by the chosen redundancy mode. Topology growth and shrink after that belong to the runtime manager.

Labels and targets should be shifted into the runtime manager. This keeps startup profiles simpler and lets the harness exercise delayed-effect configurations over time instead of baking them in at format.

Targets should still be set independently of whether they currently match any device labels. A target that points at no current devices is still useful state, because later device additions or relabeling can make it become active.

Current implementation note:
- the current bcachefs target parser does not accept arbitrary future labels; runtime target changes only accept an existing device or an existing disk-group label at the time of the change
- if the harness eventually wants to exercise unresolved future targets, that requires a bcachefs behavior change rather than just more userspace harness logic
- until then, the continuous harness should only generate `SetTarget` operations for labels that currently exist on at least one active device

### Controller Responsibilities
- manage one event loop that launches and polls asynchronous operations
- maintain enough bookkeeping to know which requests are current or superseded
- start and stop background workers
- classify outcomes as expected success, expected failure, or unknown from live observations
- run cheap live assertions at operation completion
- run heavier checkpoints at quiescent points
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

Current implementation note:
- the first worker toggle should be a simple file-churn loop under the mounted filesystem
- the next worker toggle should be a bounded-size `fio` randrw job so resize/topology operations start overlapping heavier writeback pressure
- snapshot churn should keep only a small rolling window of snapshots, but should leave some snapshots behind when stopped so later operations can hit snapshot-pinned state generated by the manager itself
- toggles can be manager-immediate operations even if the worker they control runs asynchronously in the background

### Managed Operations
Likely operation set:
- `Resize { dev, target, mode }`
- `AddDevice { dev }`
- `RemoveDevice { dev }`
- `SetTarget { kind, label }`
- `SetDeviceLabel { dev, label }`
- `ToggleRandrw`
- `ToggleFileChurn`
- `ToggleSnapshotChurn`

The important point is that controller operations stay coarse-grained and semantically meaningful. The manager should not devolve into emitting raw shell commands as its primary abstraction.

Managed operations should be actual filesystem mutations, not assertions. Checks such as remounts, offline `fsck`, and usage sampling belong in the assertion/checkpoint layer that runs around selected operations and at case boundaries.

For the initial design:
- labels only need to cover `foreground` and `background`
- only one worker instance per stress type should exist at a time
- worker operations are best modeled as toggles rather than separate start/stop enums
- when labels or targets are currently unset, the manager should be more likely to issue the operations that set them

### Weighted Selection
Runtime operations should be selected randomly, but not necessarily uniformly.

Weighted selection is useful because some important states are under-sampled by naive uniform random choices. In particular, the harness should be willing to bias toward:
- overlapping resizes on the same device
- target changes and label changes
- operations under active background stress
- harder or rarer topology states

Bias should be introduced only where ordinary randomized running would likely not hit important cases often enough.

### Bookkeeping
Internal bookkeeping should track only what is needed to launch legal operations and reason about supersession:
- current member devices and nominal sizes
- persisted/latest requested resize targets
- which operations are currently in flight
- which requests have been superseded by newer ones
- which background workers are running
- a conservative free-space estimate
- snapshot count / pinned-space estimate
- current replication policy

For the first implementation, the bookkeeping does not need to be a deep reference model. It mainly needs to know:
- which devices currently exist and are active
- the most recent requested resize per device
- which resizes have been superseded
- which stress workers are currently enabled
- the current target and label configuration

For resize-oracle purposes, requested shrinks should still be classified into:
- `ClearlyPossible`
- `ClearlyImpossible`
- `Unknown`

Only the first two classes get hard outcome assertions. The `Unknown` class is still useful for stress, but should not produce false failures.

The same applies to non-resize topology changes. `bcachefs device remove` is not the operation that evacuates a live member; it removes a member after evacuation has completed. The continuous harness should therefore follow the real workflow for hard-success remove cases: `bcachefs device evacuate <dev>`, wait for it to complete, then `bcachefs device remove ...`. Remove should only be expected to fail once evacuation itself cannot move the remaining state elsewhere.

For the first resize operation set, generate a wider spread of target sizes per device rather than a single shrink point, so the controller exercises more of the online resize surface. Outcome checking can still stay conservative: compare the requested target against the live `Used:` value from `bcachefs fs usage --all`, treat anything within `+-50MB` as ambiguous, and only hard-fail the run when a resize outside that ambiguity band behaves contrary to expectation. Ambiguous resizes should remain in the generated histories even though they do not currently produce pass/fail signals.

### Invariants And Checkpoints
After selected operations, and always at the end of a run, check:
- filesystem is still mountable with the full device set when it should be
- offline `fsck` is clean
- latest requested resize target converges eventually
- superseded resize requests do not stall later requests for long
- running workers can be stopped cleanly

Useful checkpoint policy:
- cheap live checks at operation completion
- heavier checks only when the manager has drained in-flight work and stopped background workers
- full `fsck` and remount before declaring success
- start each case from a freshly prepared filesystem state so later cases do not inherit stale topology or usage state

Runtime state generation should mostly happen inside the manager loop, not as a separate startup fixture phase. The same long-running case that resizes and retargets devices should also be able to:
- write and overwrite data
- build metadata-heavy trees
- create and delete snapshots
- flip labels and targets
- run with stress workers enabled

## Logging And Replay
The framework should produce:
- newline-delimited JSON or similarly structured logs
- a human-greppable error summary
- a deterministic scheduler seed
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
- controller randomly chooses a valid format-time profile at startup
- manager launches resize, topology, label, target, and worker-toggle operations over time
- overlap is allowed where the oracle can still reason about the result, especially for superseding resizes
- end-of-run always does worker shutdown, unmount, offline `fsck`, and remount

Later extensions:
- multiple predefined stress profiles
- longer overnight runs with persisted regression seeds
- CI mode with bounded case count
- dedicated reproducer mode that replays one saved operation transcript exactly
