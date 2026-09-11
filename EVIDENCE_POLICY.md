# Fluxel Evidence Policy

This policy defines what a development claim proves. It does not prescribe a
roadmap stage or replace focused repository test instructions.

## Evidence levels

Logic evidence includes compilation, linking, unit tests, mocks, and `TestRhi`.
It establishes portable logic and state-machine behavior, not hardware
correctness.

Real-target evidence is a run on the actual target named by the active work:
a GPU/backend, named browser and version, selected mini-game host device, or
physical mobile device. It establishes behavior only for that stated target.

Stage-close or release evidence is the retained matrix for every affected
supported target. It establishes the support claim made by that closure or
release; it does not prove untested future targets.

## Verification frequency

### Every commit

Run focused CPU tests and the relevant compile, formatting, Clippy, and
documentation checks for changed crates or modules. Run focused failure-path
and regression checks for the changed contract. These checks are deliberately
fast; they do not substitute for a real target.

### Active-stage development

Run the active closure's real-target oracle whenever a change can affect its
visible behavior, lifecycle, backend realization, or performance measurement.
Use focused checks while the closure is evolving; a full device matrix is not
required for every shared-code commit.

Escalate the active check when risk warrants it, such as a change to portable
resource lifetime, synchronization semantics, shared rendering behavior, or a
known platform-sensitive path. Record an explicit unsupported decision rather
than silently narrowing a target to avoid a failing oracle.

### Stage close or release

Run the complete affected real-target evidence matrix. Re-run prior target
oracles affected by shared-code changes and retain their results with the
release evidence. A platform is not re-certified merely because another
platform passed.

## Required retained artifacts

Each stage-close or release review points to the applicable artifacts:

- exact commit SHA and invocation command or host procedure;
- OS and version, target triple where relevant, backend, adapter or device,
  driver, browser or host version, and validation configuration;
- fixed input or interaction script, expected output, and actual screenshot,
  recording, or headless capture;
- focused automated-test results, relevant diagnostics, and validation output;
- representative workload, measurement procedure, raw metric, and declared
  regression threshold for every metric the work changes;
- a minimal example on the recommended public path and updated API or ownership
  documentation; and
- unsupported behavior, untested targets, accepted known diagnostics, and
  deferred work.

Stage 0 may retain conformance output and diagnostics in place of a visual
capture. Preserve partial output and diagnostics when a run fails.

## Visual and performance oracles

Any example, visual test, or graphical program that opens a window or produces
continuous frames requires an inspected real-run screenshot; process survival,
successful present calls, logs, and clean validation do not prove the image.
For a static result, retain at least one stable frame. For animation,
continuous rendering, resize, or another cross-frame path, retain multiple
separated frames and pair them with a dense readback/frame-marker sample or a
recording. Review clear color, geometry, dimensions, color, stale contents,
tearing, alternating frames, and flicker. Identical sparse screenshots do not
by themselves rule out intermittent clears. Every visual artifact records its
exact commit, backend, GPU and driver, window size, frame number or timestamp,
and validation diagnostics.

Use an exact image hash when output is stable. Otherwise freeze the tolerance,
mask, input, and review procedure before treating a visual difference as a
pass or failure. Text rasterization and backend differences do not justify an
unstated visual oracle.

Performance evidence is meaningful only for its documented workload and
environment. It records regression protection, not parity across devices or
backends. Define the workload, sampling method, duration, raw result, and
threshold before using a measurement as a gate.

## Target semantics and capability ledger

Browser evidence proves only the named browser, version, OS, and backend.
It cannot replace evidence for a selected mini-game host. Emulators and
simulators can supplement development evidence but cannot replace a required
physical mobile-device run.

Maintain a capability ledger that records the highest proved capability set for
each target. Do not generalize a result from one backend or platform to another.
A later platform-specific closure does not change earlier target claims unless
its oracle is rerun or an explicit reviewed support decision changes them.

Distinguish unavailable hardware, driver, validation layers, or host services
from an implementation failure. Missing required real-target evidence leaves
the relevant support claim open; it cannot be replaced by compilation, mocks,
or documentation.
