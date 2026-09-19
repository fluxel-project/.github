# Fluxel Roadmap

Fluxel is a native-first, AI-friendly lightweight rendering runtime with an
approachable scene API. This is an execution map, not a release schedule or a
promise to create every named crate or compatibility layer.

**Latest closure:** 0.15 / Stage 3E puts desktop GL 4.x, GLES 3.x, and WebGL2
behind one private `api -> state -> compat` GL-family boundary. The released
rendering tag is `v0.15.0` at
`e3134619f3ebe3130248042351acee8144d99354`. Its retained evidence proves the
named GL4 and browser WebGL2 routes and auxiliary GLES implementation evidence;
it explicitly does not claim a physical GLES 3.1 device result. Earlier 0.14
renderer-residency evidence remains the logical-asset baseline.

Development normally follows the next visible demo closure, not the layer map.
The `0.16`-`0.20` foundation train is the explicit exception: reviewed RHI,
RenderGraph, and capture/replay contracts must replace the accumulated proof
interfaces before another high-level API is frozen.

**Next execution sequence:** `0.16` native RHI -> `0.17` all RHI platforms ->
`0.18` RenderGraph semantic core -> `0.19` RenderGraph execution/trace closure
-> `0.20` portable capture/replay. Each version must close and retain its full
ecosystem evidence before the next opens. The approachable Rust scene API is
postponed until after `0.20`.

Read [development principles](DEVELOPMENT_PRINCIPLES.md) before changing the
roadmap or public APIs. Read the [evidence policy](EVIDENCE_POLICY.md) before
claiming a stage is complete. Use the
[ecosystem architecture](ECOSYSTEM_ARCHITECTURE.md) to place responsibilities;
it does not authorize a stage or repository.

## Stage 0 — Reproducible baseline

Turn the current headless vertical slice into a trustworthy baseline. Do not
extend fixed rendering recipes in this stage.

**Boundary:** preserve and prove the existing headless slice. No window,
surface API, multi-frame API, scene model, loader, asset manager, or runtime
crate belongs here.

**Owning repository:** `fluxel-rendering`.

**Participating repositories:** none.

**Artifact under test:** the `fluxel-rendering` headless conformance and
release artifacts for the existing vertical slice.

**Cross-repository contract changes:** none. This stage freezes and proves the
existing rendering boundary; it does not create a `fluxel-bases` dependency or
any Host or JS bridge contract.

**Status:** Complete.

**Latest retained evidence:** [`fluxel-rendering` v0.7.0](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.7.0),
tag commit `6bd3a25`. Its retained `manifest.json` and `cargo.log` record Windows
`x86_64-pc-windows-msvc` on AMD Radeon 780M Graphics, with 83/83 ignored
real-GPU cases passing across DX12 and Vulkan and no validation diagnostics
observed.

**TODO**

- [x] Define one workspace version and release-tag rule for crates shipped
      together.
- [x] Pin the documented workspace Git dependency examples to tag `v0.7.0`.
- [x] Keep renderer failures structured through its public boundary; do not
      erase backend, validation, resource, or submission failures into text.
- [x] Add one GPU conformance entry point that records commit SHA, OS, target,
      backend, adapter, device, driver, commands, results, and diagnostics.
- [x] Run and retain results for the relevant ignored real-GPU RHI and renderer
      tests.
- [x] Audit RenderGraph exports and remove accidental implementation or backend
      surface.
- [x] Keep native resources, barriers, commands, submission, and readback in
      RHI; keep RenderGraph portable and declarative.
- [x] Freeze the `draw_*` recipe family and public upload-state types for Stage
      0; additions require reopening the stage plan.
- [x] Retain the complete baseline on the final fixed Stage 0 source commit.

**Closure:** `v0.7.0` / `6bd3a25` is the final fixed Stage 0 source and evidence
commit. The following repository-alignment commits changed documentation only;
they did not change Cargo manifests, Rust sources, examples, CI, the conformance
gate, or the released GPU behavior, and therefore do not define a second GPU
certification target.

## Stage 1 — Windows visible renderer

Build the durable Windows rendering path. The detailed work sequence, API
boundaries, and evidence for this stage are in
[Stage 1: Windows](stages/stage-01-windows.md).

**Boundary:** Windows only. Native DX12 and Vulkan objects stay in RHI. This
stage proves a visible renderer, not assets, loading, a general platform API,
or a playable runtime.

**Owning repository:** `fluxel-rendering`.

**Participating repositories:** `fluxel-host` supplies the deliberately small
Win32 `Window` primitive used by the proof path. The proof executable and all
rendering work remain in `fluxel-rendering`; this does not start a general Host
runtime.

**Artifact under test:** the temporary Windows executable that exercises the
`fluxel-rendering` DX12 and Vulkan visible-scene path.

**Cross-repository contract changes:** `fluxel-host` owns fixed-size window
creation, HWND lifetime, message pumping, close observation, and standard
window/display-handle traits. Rendering consumes only those standard traits:
it must not depend on Host. RHI owns surface creation, swapchain configuration,
presentation, and GPU synchronization; HWND, DXGI, and COM remain private.
Input, clock, DPI policy, resize policy, fullscreen, and a Host runtime remain
out of scope.

**Status:** Complete.

**TODO**

- [x] 1.1 DX12 first image: create a window and surface, clear, draw a fixed
      triangle, present, close cleanly, and release in a valid order.
- [x] 1.2 Surface lifecycle: handle resize, minimize, restore, invalid sizes,
      recreation, and structured surface or device failures. A new surface
      generation stops acquisition from the old generation but does not release
      its presentable resources until every GPU reference reaches known completion.
- [x] 1.3 Multi-frame lifetime: prove bounded frames-in-flight resource reuse,
      completion-driven retirement, and back pressure. The Windows demo uses
      `N=3` by default without exposing that policy as a triple-buffer API.
- [x] 1.4 Vulkan alignment: the shared Windows presentation path runs on Vulkan
      under the same lifecycle contract as DX12.
- [x] 1.5 Minimal multi-object scene: deterministic
      multi-object preparation and draw submission, with private `slot-graph`
      use only for a naturally occurring renderer CPU-preparation DAG. RenderGraph retains GPU-resource
      semantics; `slot-graph` neither owns GPU synchronization nor escapes the
      renderer boundary.

Stage 1.1 closed in `fluxel-rendering` `v0.8.0` (`800b390`) with the reusable
Window primitive supplied by `fluxel-host` `v0.1.0` (`e02b736`). The retained
release evidence records 899 presented DX12 frames, 120 dense client captures
with a stable visible triangle, clean required validation, and clean shutdown.

Stage 1.2 closed in `fluxel-rendering` `v0.8.1` (`97558a8`) with ordered window
lifecycle events supplied by `fluxel-host` `v0.2.0` (`ae4bda3`). RHI-owned
surface generations stop acquisition before completion-driven retirement and
reconfiguration; invalid extents suspend drawable work. The retained DX12
evidence records generations 1–4 across resize, minimize, restore, and close,
5161 presented frames, 191 suspended iterations, 120 timestamped samples / 240
screenshots, clean validation, and clean shutdown. Reviewers opened samples at
each boundary and found the expected stable black clear and blue triangle.

Stage 1 closed in [`fluxel-rendering` `v0.8.3`](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.8.3)
at `1778226fb74d3fc0f2fdb12b6dd8da9d9d149960`. The Release's durable
`fluxel-rendering-v0.8.3-evidence.zip` binds the final DX12/Vulkan conformance,
visual, lifecycle, and diagnostic evidence to that source commit.

**Close when:** the same scene runs continuously on DX12 and Vulkan, survives the
defined lifecycle sequence, shuts down cleanly, and retains visual,
diagnostic, and baseline evidence without unexplained validation errors.

## Stage 2 — Web and one mini-game host

Keep the Stage 1 scene and expected image. Each substage names its real target
before implementation; it does not authorize the later WebGPU or mini-game
gate. Its completed browser substages have separate execution contracts:
[Stage 2.1: WebGL2](stages/stage-02-web.md) and
[Stage 2.2: WebGPU](stages/stage-02-webgpu.md).

**Boundary:** support WebGL2, WebGPU, and one named mini-game host separately.
Do not claim generic web or mini-game support, redesign native APIs around web
global state, or use browser evidence for the host device.

**Owning repository:** `fluxel-rendering`.

**Participating repositories:** `fluxel-jsbridge`.

**Artifact under test:** the rendering WASM package plus each separately named
browser or mini-game integration package, run on its real target.

**Cross-repository contract changes:** define only the narrow rendering-to-JS
bridge needed to start, resize, submit the preserved scene, and report
structured diagnostics. Platform APIs remain owned by their adapters; neither
repository gains a generic host contract.

**TODO**

- [x] 2.1 WebGL2 / 0.9: Windows 11 x64 Google Chrome Stable `153.0.8010.36`:
      canvas creation and resize, retained multi-object rendering, visibility
      and context recovery, resource rebinding, and scoped package, startup,
      CPU-submission, and memory measurements. Edge 153 is auxiliary evidence
      only, not another supported browser claim.
- [x] 2.2 WebGPU / 0.10: on its named Chrome/Windows/AMD target, preserve the
      same scene and expected output through canvas reconfiguration,
      completion-bounded submission, device-loss/recovery, and portable
      RenderGraph semantics without a public multi-queue API.
- [ ] 2.3 One later, explicitly selected real mini-game host: prove device
      startup, lifecycle, canvas or surface, minimal input, packaged resource
      paths, and foreground or background behavior. This target is not chosen
      or authorized by 0.10.

**Close when:** WebGL2, WebGPU, and the selected mini-game host each have their own
real-target visual, lifecycle, diagnostics, and scoped performance evidence.
Browser evidence does not close the mini-game host gate.

The unselected 2.3 target is an independent support gate. It may remain open
while later native/resource work that does not consume it proceeds; doing so
neither closes Stage 2 as a whole nor creates a generic mini-game claim.

**2.1 result:** `fluxel-rendering` candidate `098ee1bd5d87ef17ba2cc8ec1031636b3e4e57d3`
and `fluxel-jsbridge` candidate `db6361fbc015522bf8abf37919da45a48a7f0daa`
close the Chrome/WebGL2 substage.  The named-target run covered stable, resize,
zero-size, restore, hidden, visible, context-lost, and context-restored states;
it retained three representative screenshots per state plus fifteen dense
frame-marker/readback samples for every visible state.  It observed the expected
black clear and red/green/blue scene, clean structured/browser/WebGL diagnostics,
stable 1,179,648-byte WASM memory, and CPU submission at or below 1 ms in the
fixed workload.  The same rendering candidate's Windows AMD Radeon 780M DX12 and
Vulkan conformance gate passed 83/83 real-GPU cases across the renderer and RHI
test binaries.  This is evidence for 2.1
only: WebGPU and a named mini-game host remain required to close Stage 2.
The durable releases are
[`fluxel-rendering` v0.9.0](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.9.0)
and [`fluxel-jsbridge` v0.1.0](https://github.com/fluxel-project/fluxel-jsbridge/releases/tag/v0.1.0).

**2.2 release result:** the same retained black/red/green/blue scene
has completed the named Chrome WebGPU lifecycle: stable rendering, resize,
zero-size/restore, hidden/visible, controlled `device.destroy()` loss,
terminal loss followed by a newly requested device identity/generation domain,
and async dispose. Old handles never revive. The WebGPU path keeps
canvas context, device, queue, pipeline/buffer objects, completion tickets,
and recovery private to rendering WASM/RHI; the JS bridge owns only DOM
lifecycle and one RAF. The release evidence records bounded `N=3`
completion-driven admission, diagnostics, scoped measurements, and visual plus
pixel/frame-marker samples. It is evidence for 2.2 only: WebGL2 remains its
own 2.1 closure and the mini-game gate remains open.

The durable releases are
[`fluxel-rendering` v0.10.0](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.10.0)
and [`fluxel-jsbridge` v0.2.0](https://github.com/fluxel-project/fluxel-jsbridge/releases/tag/v0.2.0).
The downloaded `fluxel-rendering-v0.10.0-evidence.zip` is 60,347 bytes with
SHA-256 `94280a7ad6cccf518a1c8975326fe847f448956681a1c97de93955d72691b8fd`.
The compatible corrective release
[`fluxel-rendering` v0.10.1](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.10.1)
at `b7b6504a8cbf9568335e9f4bbe3f7335b7a764f6` joins in-flight recovery during
terminal disposal and publishes replacement-device facts only at their commit
point. Its exact-SHA native and named-browser regressions pass; it supplements,
rather than replaces, the original closure evidence.

## Stage 3 — Resource foundation and scene API

The retained scene has proved four backends, but its proof interfaces and
resource breadth are still intentionally narrow. Close the real resource and
lifetime model before a public scene API can accidentally encode raw GPU
objects, backend-specific completion, or per-pass asset lookup.

**Boundary:** Rust owns the semantics. Keep logical assets, persistent GPU
residency, and graph-transient resources as three separate lifetime domains.
Do not create a general asset pipeline, filesystem loader, material catalog,
compatibility facade, or speculative transient allocator.

**Owning repositories:** `fluxel-rendering` for RHI, RenderGraph, renderer,
residency, and internal contracts; `fluxel-bases` for logical asset mechanisms
only after a real consumer proves them.

**Participating repository:** `fluxel-jsbridge` for browser contract and real
WASM integration gates.

### Stage 3A / 0.11 — Architecture and contract consolidation

No new rendering feature belongs in this closure.

**TODO**

- [x] Add wasm32 compile/Clippy/bindgen smoke gates to rendering, Node CI to
      the JS bridge, Windows CI to Host, and one pinned real-WASM/browser
      cross-repository smoke artifact.
- [x] Record every integration repository SHA/tag and lock input; a producer
      contract change must build and test its affected pinned consumer.
- [x] Split browser manual frame submission from last-frame observation. Loop
      ownership must not change a command into a query. `requestFrame()` asks
      for one submission opportunity and returns an explicit submitted/blocked/
      terminal outcome; it neither promises immediate submission nor reads the
      last report. `lastFrameReport()` is a query and never submits.
- [x] Make JS compute CSS/DPR policy while rendering WASM/RHI alone mutates the
      canvas drawing-buffer extent.
- [x] Move proof-only cross-crate APIs out of documentation-hidden public
      semver surfaces into an explicitly supported or non-published internal
      boundary.
- [x] Use `Active`, `Suspended`, `Lost`, `Recovering`, `Disposing`, `Disposed`,
      and `Poisoned` as the canonical upper lifecycle vocabulary where each
      term truly applies. Identically named JS producer and RHI device states
      need not transition together; backend completion/loss remains private.
- [x] Make replacement device, queue, format, adapter metadata, generation, and
      device-affine objects one candidate transaction. Failed or stale attempts
      publish none of those facts.
- [x] Define terminal disposal by absence of live work, not by an enum alone:
      after the Promise settles there is no active submission, detached
      completion, recovery attempt, stale callback producer, RAF/listener, or
      future GPU-object commit.
- [x] Audit callback-capable FFI so no Rust dynamic borrow or lock guard crosses
      a browser call. Use production predicates plus deterministic interleaving
      models for recovery, disposal, completion, resize, visibility, and
      candidate-install failure.
- [x] Reduce repository READMEs to current capability and recommended usage;
      this roadmap remains the stage/status authority.

**Close when:** native and browser public surfaces contain no accidental proof
API, command/query and resize ownership are unambiguous, and fast CI compiles
the real cross-repository composition. Existing real-target support must remain
unchanged and affected release oracles must pass.

This is an ecosystem milestone, not a shared package version. Each repository
releases only for its actual public changes and keeps its own version sequence.

Closure releases are
[`fluxel-rendering` v0.11.0](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.11.0)
and [`fluxel-jsbridge` v0.3.0](https://github.com/fluxel-project/fluxel-jsbridge/releases/tag/v0.3.0).
Rendering CI passed every stable/MSRV/native/WASM/cross-repository job; the
exact-SHA DX12+Vulkan gate passed 83/83 and WebGL2/WebGPU browser lifecycle
evidence passed against the pinned JS-bridge revision. The downloaded evidence
archive is 71,379 bytes with SHA-256
`a8564b6431a41c1e37e0f52f1c4a577815f90682c71d5d80872c75e625387f1d`.

The compatible review-gap patches are
[`fluxel-rendering` v0.11.1](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.11.1)
at `d2fc277d5b6f4faa18477fba6142027ca7d8dabf` and
[`fluxel-jsbridge` v0.3.1](https://github.com/fluxel-project/fluxel-jsbridge/releases/tag/v0.3.1)
at `c010016701f566123040d2735de1c2b9549bdca2`. They add a pinned real-WebGPU
Chrome integration job beside WebGL2, report a coalesced manual frame request
as `already-scheduled`, and correct next-series documentation without changing
the renderer feature or supported-target claims. Rendering run `34701624449`
passed all 17 jobs; its exact-SHA DX12+Vulkan gate passed 83/83 and both named
Windows Chrome browser gates passed. The downloaded v0.11.1 evidence archive
is 65,515 bytes with SHA-256
`fe2c5b9ae4a206fd32a3157e5bfad40012689a6b3983321d2e1576b9193b02bf`.

### Stage 3B / 0.12 — Minimum GPU resource closure

**TODO**

- [x] Before changing the resource contract, add an RHI-private injectable
      browser-request provider and wasm browser tests which defer/resolve/reject
      the actual production `requestAdapter`/`requestDevice` recovery path.
      A duplicate reducer or JavaScript-only fake does not close this gate.
- [x] Prove a **common resource floor** across the selected backend matrix:
      vertex/index/uniform buffers, sampled textures and samplers, offscreen
      color targets, depth, upload, copy/readback, and multipass execution.
- [x] Prove **modern capabilities** separately on each named supporting backend:
      storage buffers, storage textures, and compute. RenderGraph requirements
      are explicit; a backend below that capability returns a structured
      `UnsupportedCapability` result and never emulates or silently lowers it.
- [x] Keep persistent resources as explicit imports with provider-owned
      physical identity and initial state; keep graph-created resources
      transient and reject unsupported physical aliasing.
- [x] Prove that repeated execution of one stable compiled graph reuses its
      compatible physical transient resources across frames instead of creating
      and destroying them per frame. Graph invalidation, extent changes, and
      device loss retire the old realization only after known completion.
- [x] Prove state, last use, accepted work awaiting terminal receipt/completion,
      device identity/generation-domain recreation, and completion-driven
      retirement on the named backend matrix.
- [x] Freeze only capability facts demonstrated by those tests. Do not mirror
      a platform API or expose multi-queue policy.

**Close when:** the common floor, persistent imports, transient virtual
resources, and stable-graph physical reuse pass every selected backend;
modern-capability cases pass only their declared supporting backends and fail
closed elsewhere, with deterministic readback and clean lifecycle diagnostics.

**0.12 release result:** [`fluxel-rendering` v0.12.0](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.12.0)
at `69d75cbc6008acbf6bfb547733410ae762a3b863` closes Stage 3B. Its
[CI run](https://github.com/fluxel-project/fluxel-rendering/actions/runs/34740447079)
and retained `manifest.json` (18,340 bytes, SHA-256
`ee92eb7c6e73294788ab01a6f49b45dad10a6a8d9df1e9e6d7da9926aaf42c67`) and
`cargo.log` (556,429 bytes, SHA-256
`490b17c0da8c6790c9df6aeab9caa11913eeca5852041bbdacf54712457ae6d5`) bind
the exact source to clean-validation resource evidence. DX12, Vulkan, and
WebGPU close the resource and compute floor; Vulkan storage-texture read/write
has exact readback; DX12 supports storage write, while its AMD Radeon 780M
storage-texture read returned zero and is disabled in production with a
structured rejection. WebGL2 compute and storage return `UnsupportedCapability`
without emulation or side effects.

### Stage 3C / 0.13 — Logical asset core

Use embedded/in-memory content first so platform I/O cannot define the asset
contract.

**TODO**

- [x] In `fluxel-bases`, prove typed logical identity, content generation,
      explicit `Ready`/`Missing` state, strong/weak ownership, stale-handle
      rejection, and structured production failure.
- [x] Let the asset core coordinate one producer per logical identity so
      concurrent consumers await the same result. The core does not know
      whether production used embedded bytes, generated data, file/network I/O,
      or decoding; those source operations remain Loader responsibility.
- [x] Treat zero logical users as a reusable CPU-cache candidate, not immediate
      destruction; add size accounting and a small explicit budget/eviction
      policy without raw manager back-pointers or GPU knowledge.
- [x] Keep logical identity separate from loaded bytes and from every RHI
      resource. Do not make handle dereference hide generation/state changes.

**Close when:** two real consumers share one logical content generation and one
identity-level producer, stale identity fails closed, and cache collection is
deterministic under the fixed budget workload without a Loader dependency.

**0.13 release result:** [`fluxel-bases` v0.13.4](https://github.com/fluxel-project/fluxel-bases/releases/tag/v0.13.4)
at `22c4eb0e199575aa71b59f3abc6ec3f72d934b9a` closes Stage 3C. The series
records its architecture in v0.13.0, then makes the API declarations, six
numbered examples, and public-only contract tests executable in v0.13.1. v0.13.2
implements the thread-safe logical-asset core: typed identities and immutable
content generations; explicit missing, producing, ready, and failed observations;
strong/weak ownership; deterministic stale-identity rejection and lowest-free-slot
recycling; single-flight producer permits and attempt-bound waiters; replacement,
failure, cancellation, retry, and structured errors; and explicit CPU resident-byte
budgets with deterministic collection. It imports neither a Loader nor RHI/GPU
objects.

v0.13.3 adds the reproducible representative asset-store benchmark and accepted
baseline: 1,024 acquire/observe/release uses of 4,096 prepared identities take
48.322--48.563 us across three rounds with zero allocations; correctness,
examples, Clippy, docs, and benchmark-smoke gates remain green. The final review
patch, v0.13.4, fixes resident-byte-overflow completion so a failed commit no
longer leaves its producer permit permanently in `Producing`; its regression
proves that the identity recovers through the structured failure/retry path.

The review also retained reproducible rejected-candidate evidence. `slotmap`
cannot preserve the contract's deterministic lowest-free-slot reuse and checked
generation exhaustion without a second source of truth. Interleaved testing of
`parking_lot::Mutex` was inconclusive and would remove the public poisoning
diagnostic while adding dependencies; `parking_lot::RwLock` was about 22% slower
on the representative workload and adds atomic/locking complexity. Production
therefore retains the hand-rolled deterministic slot store guarded by
`std::sync::Mutex`. The exact-source [CI run](https://github.com/fluxel-project/fluxel-bases/actions/runs/34745220511)
passed the release gates; benchmark evidence describes the CPU asset contract
only, not GPU residency.

### Stage 3D / 0.14 — Rendering residency

**TODO**

- [x] Map `(asset identity, content generation, device generation)` to a
      persistent rendering realization with explicit pending, committed, and
      retire-candidate states.
- [x] Resolve residency once during frame preparation, import the result into
      RenderGraph, and remove asset management from pass execution.
- [x] Commit uploads only after known completion; logical release merely
      requests retirement, while graph-recorded last use plus submission
      completion authorizes RHI destruction or reuse.
- [x] Prove device-loss recreation retains logical assets while replacing only
      the dead GPU generation.

**Close when:** one shared mesh/image survives reuse, content replacement,
device recreation, and completion-safe retirement without duplicate GPU
realization or stale-generation access.

**0.14 release result:** [`fluxel-rendering` v0.14.0](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.14.0)
at `ad37a14da5415b880c4746ce3238e83aa8ea933a` closes Stage 3D. Its
[exact-source CI run](https://github.com/fluxel-project/fluxel-rendering/actions/runs/34748806169)
passed, and the retained native conformance manifest (18,564 bytes, SHA-256
`c739af27309695874c66da828deb207999cbe3d4970569dca00322f6b0ed3396`) and
Cargo log (556,804 bytes, SHA-256
`ed79c7f54a55375e18934b5297e60f036d78157cbd8634c02540b17c90f8a570`)
bind the release SHA to clean-validation DX12/Vulkan evidence. Chrome 153
WebGPU and WebGL2 production-path archives are also attached at 39,088 bytes
(`b864ddcf707dd03ff15768a1d68e7acc52ca757f5b35a65c7ea4d0040473cf57`)
and 38,045 bytes
(`b9d0eef9a198962f3afb3f81d54d71da7efe4804a9f519aa44429532cb3c5c91`).
The closed scope is renderer-private residency for fixed mesh and RGBA8 image
assets; loader/I/O, generic material and scene APIs, and a generic residency
manager remain future work.

### Stage 3E / 0.15 — GL-family boundary

**Status:** Complete.

One GL-family backend now covers desktop GL 4.x, GLES 3.x, and browser WebGL2
profiles behind private `api`, `state`, and `compat` layers. It is a peer of the
other RHI backends and does not expose a public GL-flavored graphics API. The
closure retains named route/refusal facts and the desired/applied/unknown state
model required by a stateful backend.

**Closure:** [`fluxel-rendering` v0.15.0](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.15.0)
at `e3134619f3ebe3130248042351acee8144d99354`. The evidence covers the named
desktop GL and WebGL2 paths plus auxiliary GLES evidence; a portable real-device
GLES claim remains deliberately open.

### Foundation train contract

Stages 3F-3J are sequential. A later stage does not begin until the prior stage
is released with retained ecosystem evidence. During the train, higher-level
renderer, WASM entry, example, or fixed-recipe code may be explicitly dormant
while its dependency is replaced, but it must not be deleted or replaced by a
second public resource model. No advertised release may ship an advertised crate
as commented-out code.

The canonical detailed documents are maintained in `fluxel-rendering`:

- [normative RHI API v1 specification](https://github.com/fluxel-project/fluxel-rendering/blob/main/documents/design-rhi.md);
- [framework/interface contract](https://github.com/fluxel-project/fluxel-rendering/blob/main/documents/design-foundation-interfaces.md);
- [RHI architecture](https://github.com/fluxel-project/fluxel-rendering/blob/main/documents/design-rhi.md);
- [RenderGraph architecture](https://github.com/fluxel-project/fluxel-rendering/blob/main/documents/design-rendergraph.md);
- [capture/replay architecture](https://github.com/fluxel-project/fluxel-rendering/blob/main/documents/design-capture-replay.md); and
- [five-version execution plan](https://github.com/fluxel-project/fluxel-rendering/blob/main/documents/version-plan.md).

The API v1 specification is the sole normative RHI API source. Architecture
documents preserve layer ownership and the version plan preserves delivery and
evidence gates. This roadmap is the ecosystem status authority only. A conflict
in a public type, error, capability, command, submission, presentation, graph
bridge, or tooling contract is resolved in favor of API v1.

### Stage 3F / 0.16 — Portable RHI protocol and native backends

**Status:** Planned/in progress. No completion claim exists until every gate
below, including real Metal, is retained on one released source revision.

**Boundary:** implement the frozen portable semantic API on DX12, Vulkan, and
Metal. Capability is adapter/device/format/surface/route instance data, not
trait presence. Every device-affine public value is validated against opaque
device/context identity plus generation; device loss terminates that identity
and recreation creates a new one. Browser sessions/tokens, native handles,
barriers, fences, semaphores, native queues, descriptor heaps, and memory
offsets do not enter the public model.

**TODO**

- [ ] Implement and test the complete frozen API surface: identity/error model;
      provider, adapter, request, and device lifecycle; capability/format/route/
      lane facts; resources, transfer, and readback; shader, binding, and
      pipeline objects; recording and actual uses; submission, completion,
      retirement, and presentation; logical statistics; `graph_bridge`; and
      the reconstructability/tooling hooks required by portable capture.
- [ ] Provider/adapter/device discovery, async request seam, adopted-context
      seam, identity/generation, and available/required/enabled capabilities.
- [ ] Per-format/route/artifact facts and pairwise lane dependency routes.
      Delete real-overlap and global GPU-dependency booleans.
- [ ] Canonical resources/views/samplers, upload/copy/readback, binding and
      pipeline interfaces, recording, submission receipts, per-point
      completion, presentation outcomes, loss, and retirement.
- [ ] Provide the opaque transient allocation-requirements query needed by the
      future graph; do not expose native heap/memory types or enable aliasing.
- [ ] Put present mode in presentation configuration and present frame/after
      relation in `SubmissionPlan`.
- [ ] Define reconstructable shader artifacts and canonical object/command/
      submission observation from the first RHI version; do not freeze a
      capture file format here.
- [ ] Prove tooling subscribe/drop linearization and callback restrictions;
      lazy description of pre-existing objects/work and presentation fixtures;
      external dependencies on GPU, ordered, and collapsed routes; builder
      failure/Drop frame recovery; and conditional direct-MSAA fallback.
- [ ] Replace the borrowed DX12/Vulkan implementation and add Metal without
      weakening the frozen baseline refusals.
- [ ] Run the shared contract suite, real Windows DX12/Vulkan evidence, and a
      named real macOS/Metal run. Compile-only Metal cannot close the stage.
- [ ] Prove dependency removal in every published feature combination.

**Close when:** `v0.16.0` proves every P0 invariant and structured refusal in
API v1, including multi-device misuse, loss/terminal states, declared-versus-
actual graph coverage, presentation ownership and no-submit recovery,
statistics, tooling linearization, complete object describability, external
dependency routes, and conditional MSAA fallback, on all three real native
backends. It does not claim all
RHI platforms and does not open Graph implementation.

`stages/stage-03f-common-rhi-layer.md` is a historical implementation journal,
not an API authority. Its trait-based capability, closed-recipe, compressed-
format, or old-facade entries are superseded where they conflict with API v1;
no work item may be implemented from it without reconciliation.

### Stage 3G / 0.17 — All RHI platforms

**Boundary:** put browser WebGPU and the GL family (desktop GL, GLES, WebGL2
profiles) behind the same RHI semantic/ownership model, migrate every affected
consumer, and rerun native evidence on the integrated source.

**TODO**

- [ ] Implement WebGPU resources, bindings, commands, completion, canvas
      identity/generation domains, terminal device loss, old-identity rejection, and
      terminal disposal through the common RHI contract.
- [ ] Make GL `compat` implement the same contract while `api` and `state`
      remain backend-private; default framebuffer is `FrameAttachment`, never a
      fake texture.
- [ ] Preserve separate GL4/GLES/WebGL2 facts and structured refusals; never
      emulate compute/storage on a profile that lacks them.
- [ ] Reject deferred compressed formats through the common structured API;
      do not predeclare unsupported P1/P2 vocabulary in the P0 surface.
- [ ] Re-enable or minimally migrate renderer, WASM, examples, JS bridge, and
      required Host consumers only for RHI integration proof; remove obsolete
      parallel browser resource models but add no high-level feature.
- [ ] Rerun final-source DX12, Vulkan, and Metal evidence plus named Chrome
      WebGPU/WebGL2, desktop GL, and real-device GLES evidence. Emulator GLES is
      auxiliary only.
- [ ] Prove unsupported cells fail before context/resource/submission side
      effects and that no public/internal resource model uses browser sessions
      or asset tokens.

**Close when:** `v0.17.0` retains one exact capability/route/lifecycle matrix
for every declared backend/profile and every cross-repository consumer passes at
pinned revisions. Only this closure opens RenderGraph implementation.
The same API v1 contract, rather than a browser- or GL-specific parallel
contract, must pass on every declared backend/profile.

### Stage 3H / 0.18 — RenderGraph semantic core

**Boundary:** implement graph authoring, validation, logical compilation, and
per-frame instantiation over the closed RHI. Do not hide core correctness behind
parallelism or alias optimization.
Graph consumes the frozen `graph_bridge` and must neither replace nor extend
the public RHI API.

**TODO**

- [ ] Typed pass-local buffer/texture/attachment handles and resolver
      enforcement for stable Raster/Compute/Copy passes. Host remains unfrozen
      without a real consumer.
- [ ] Logical versions, full/partial/discard definedness, subresource/range
      inheritance, RAW/WAR/WAW and memory dependencies.
- [ ] Portable `initial_use`/`final_use` imports/exports, owner lease/history
      validation, observable roots, reverse culling, and present/readback roots.
- [ ] Target-aware immutable `CompiledGraph`, fingerprints, deterministic
      report/visualization, per-frame `GraphInstantiation`, and explicit
      `GraphExecutionPlan -> SubmissionPlan` lowering.
- [ ] Query opaque allocation requirements but emit a conservative no-alias
      plan, and use only the guaranteed serial lane in this release.
- [ ] Run exhaustive CPU/model/TestRhi tests and common real-backend graph
      workloads; unsupported modern paths fail before side effects.

**Close when:** `v0.18.0` proves every scoped version, dependency, root,
declaration, import/export, instantiation, and error rule deterministically.

### Stage 3I / 0.19 — RenderGraph execution and trace closure

**Boundary:** complete target-aware allocation/reuse/alias, lane-route
scheduling, optional equivalent lowering optimizations, completion-aware
readback, and a non-replay diagnostic artifact.

**TODO**

- [ ] Activate reuse/alias planning from the already-frozen opaque requirements;
      implement `AliasBoundary`, realization, and mandatory no-alias fallback.
- [ ] Prove completion-safe cross-frame reuse and same-frame aliasing through
      success, preflight rejection, accepted pending/failed terminal state, and loss.
- [ ] Implement pairwise lane routes and correct serial collapse without any
      claim of hardware overlap.
- [ ] Gate parallel recording, merge, batching, and coalescing behind
      serial-oracle equivalence tests.
- [ ] Publish versioned `GraphTraceArtifact`, typed diagnostics/checkpoints, and
      native-tool correlation. It must not be accepted as replay input.
- [ ] Implement and verify the already-frozen Graph/RHI observation and
      reconstructability hooks; do not redefine their public semantics or
      freeze persistent capture storage.

**Close when:** `v0.19.0` passes alias/no-alias, lane/serial, trace, readback,
and full real-backend regression gates. The scoped RenderGraph is then complete.

### Stage 3J / 0.20 — Portable capture and replay

**Boundary:** record and rebuild Fluxel semantics, never native commands or Rust
layout. `PortableCommandIR` plus objects/snapshots/submission is normal replay
truth. `FrozenGraphIR` is provenance/declared-use validation; recompilation is a
separate comparison mode.
This stage freezes artifact storage and ReplayRuntime behavior, not a second
RHI API.

**TODO**

- [ ] Capture scopes and dependency/snapshot closure, typed IDs, canonical
      object/command tables, mutations, resource snapshots, checkpoints, and
      external-input fixtures/providers.
- [ ] Shader portability classes and pipeline reconstruction without requiring
      a backend binary.
- [ ] Canonical submission/present/completion/retirement and negotiated
      `Direct`, `Adapted`, or `Unsupported` replay.
- [ ] Headless/windowed replay, command legality/hash/tolerant observations,
      query policy, debugger inspection mode, diff, first-failure provenance,
      and report.
- [ ] Complete/partial/failed finalization, privacy/redaction, integrity,
      version handling, optional signatures, and bounded untrusted parsing.
- [ ] Prove same-backend replay on every declared profile and cross-backend
      replay wherever shader/capability negotiation permits; prove normal replay
      is independent of current Graph compiler output.

**Close when:** `v0.20.0` ships the first proved artifact schema and retained
capture/replay evidence. The promise is verifiable portable semantics, not
native barrier order, physical placement, driver commands, or exact timing.

### Stage 3K — Approachable scene API validation (post-0.20)

**Entry gate:** Stages 3F-3J are all released and the final ecosystem
composition passes at exact revisions.

**TODO**

- [ ] Design the smallest coherent Rust `Scene`, `Camera`, `Mesh`, `Geometry`,
      `Material`, and `Transform` model over the proved asset/residency path.
- [ ] Make ownership, sharing, mutation, removal, deterministic draw order,
      and structured errors explicit; ordinary scene code cannot access RHI or
      RenderGraph.
- [ ] Keep language bindings thin and replaceable; Rust remains authoritative.
- [ ] Probe the experimental API on two distinct representative backends, then
      run every affected supported target before declaring it stable.
- [ ] Publish one recommended scene construction and rendering path.

**Close when:** one coherent scene API consumes logical assets and prepared
residency across every affected supported target without exposing backend or
resource-manager internals.

Deferred RHI/Graph families remain consumer- and evidence-gated. The word
“complete” in Stages 3F-3J means their reviewed scoped contracts, not ray
tracing, bindless, work graphs, sparse resources, multi-device, a speculative
Host pass, or every future optimization.

## Stage 4 — Windows playable runtime

Extend the Stage 3 scene into a small long-running Windows application. Consume
the proved asset/residency contracts; introduce platform, time, input, loader,
and application-composition boundaries in their owning repositories only when
the running application demonstrates separate ownership.

**Boundary:** Windows playable runtime only. Create boundaries on demonstrated
need, not because they appear in the layer map. Do not add networking, storage,
audio, ECS, editors, general scripting, or a broad file-format framework.

**Owning repository:** `fluxel-host`.

**Participating repositories:** `fluxel-bases`, `fluxel-rendering`, and
`fluxel-jsbridge`.

**Artifact under test:** the Windows EXE produced by `fluxel-host`, linked to
the selected rendering artifact and exercised as one sustained application.

**Cross-repository contract changes:** introduce only contracts demonstrated by
the application: Base ownership and diagnostic types where shared, Host-driven
surface and frame calls into Rendering, and a narrow native JS bridge if the
application uses one. Rendering does not acquire host lifecycle, I/O, input,
or process ownership.

**TODO**

- [ ] Add a host loop with startup, frame, pause, resume, close, shutdown,
      delta time, and fixed-update behavior.
- [ ] Adopt `async-runtime` as the host-owned native scheduler: the host owns
      worker shutdown and drives owner-thread local domains within an explicit
      frame budget; rendering and RenderGraph do not acquire runtime policy.
- [ ] Add only the required input, file or memory loading, mesh and image
      decoding, and camera or game control.
- [ ] Load the Stage 3 logical assets through the smallest synchronous platform
      reader and preserve the proved GPU residency/retirement contract.
- [ ] Add asynchronous loading only after the synchronous lifecycle is sound.
- [ ] Extract platform, loader, assets, time, input, and runtime boundaries
      only where the running application makes them real.

**Close when:** the Windows application is operable and stable in a sustained run,
with duplicate reuse, stale-handle failure, device or surface recovery, and
safe shutdown evidenced. Networking, storage, audio, ECS, editors, and general
scripting are out of scope.

## Stage 5 — Mobile runtime

Port the Stage 4 application rather than creating mobile-only renderer demos.
Android and iOS are independent closures. Completion of one must be reported
as that platform only; neither closure blocks a separately scoped Canvas
experiment.

**Boundary:** port the existing application and preserve its semantics. Do not
make a mobile-only renderer demo, infer support on one platform from the other,
or block a separately planned Canvas slice on unavailable mobile hardware.

**Owning repository:** `fluxel-host`.

**Participating repositories:** `fluxel-bases`, `fluxel-rendering`, and
`fluxel-jsbridge`.

**Artifact under test:** the Android APK/AAB and iOS IPA produced by
`fluxel-host`; each is an independent real-device application artifact.

**Cross-repository contract changes:** preserve the Stage 4 contracts while
adding only platform-specific Host lifecycle and surface mappings. Any shared
identity, diagnostic, capability, or bridge representation belongs in its
owning repository; mobile support must not leak OS objects into Rendering.

### Stage 5A — Android

**TODO**

- [ ] Run the application on a named real Android device and supported backend.
- [ ] Handle touch, foreground and background transitions, surface destruction
      and recreation, orientation, and resize.
- [ ] Retain device, OS, GPU, driver, memory, thermal, visual, and sustained
      run evidence.

### Stage 5B — iOS

**TODO**

- [ ] Run the same application on a named real iOS device through Metal.
- [ ] Handle touch, inactive, background, foreground, surface recreation,
      size, orientation, scale, and safe-area changes.
- [ ] Retain device, OS, GPU, memory, visual, and sustained run evidence.

**Close when:** 5A and 5B each close independently when their named real-device
gate passes. Work scoped to an already proved platform may continue after
either closure; a cross-mobile support claim requires both.

## Stage 6 — Canvas and minimal text

Build one Canvas demo on the proven runtime and resource lifecycle. Freeze its
actual target matrix before implementation.

**Boundary:** keep 2D drawing semantics separate from the platform loop, asset
manager, and UI system. Extract `fluxel-canvas` only if the demo proves that
independent ownership boundary. Text is intentionally minimal.

**Owning repository:** `fluxel-rendering`.

**Participating repositories:** `fluxel-bases` and `fluxel-jsbridge`.

**Artifact under test:** the selected-target Canvas demo built from the
`fluxel-rendering` artifact, with its JS bridge integration where that target
uses one.

**Cross-repository contract changes:** add only demonstrated Canvas/text data,
render-resource identity, and structured diagnostics contracts. Base retains
shared mechanisms, while JS bridge bindings are adapters rather than a second
Canvas semantic authority; no platform-loop or UI-system API is introduced.

**TODO**

- [ ] Add images or sprites, 2D transforms, rectangular clipping, alpha
      blending, explicit ordering, and an offscreen target.
- [ ] Define coordinates, pixel scaling, color and alpha conventions, clipping,
      and ordering deterministically.
- [ ] Add one font path, LTR text, a basic character set, basic line breaking,
      fixed alignment, and a glyph cache tied to asset and GPU retirement.
- [ ] Define missing-glyph, cache growth, invalidation, device-loss, and
      release behavior.

**Close when:** the fixed Canvas demo passes its declared visual and lifetime
oracle on every target selected for this slice.

Complex shaping, bidirectional text, fallback font stacks, rich text, broad
multilingual layout, arbitrary paths, and CSS layout are not part of this
stage.

## Stage 7 — UI closure

Build one durable HUD or settings screen using the existing runtime, Canvas,
text, and input. Start from the screen's real requirements.

**Boundary:** ship one concrete screen and reuse existing lifecycle systems.
Do not create a general UI framework, virtual DOM, CSS system, or third-party
compatibility layer.

**Owning repository:** `fluxel-rendering`. The UI closure extends the existing
Canvas and text path with one purpose-built screen; it does not make a language
adapter the authority for UI or rendering semantics.

**Participating repositories:** `fluxel-bases`, `fluxel-jsbridge`, and
`fluxel-host`.

**Artifact under test:** the selected HUD or settings-screen demo from
`fluxel-rendering`, running through its declared JS bridge adapter and host
where those repositories participate in the selected target.

**Cross-repository contract changes:** define only the screen's needed bridge
for input delivery, state updates, Canvas/text commands, lifecycle teardown,
and structured diagnostics. Host continues to own platform events, Rendering
continues to own drawing and resource lifetime, and Base holds only genuinely
shared mechanism types.

**TODO**

- [ ] Add only required basic layout, text, images, interaction, state updates,
      show or hide behavior, page destruction, and resource release.
- [ ] Define relevant hit-testing, event order, focus, and removal semantics.
- [ ] Prove the screen remains interactive and can be recreated without stale
      input targets or resource growth.

**Close when:** the chosen screen is interactive for its declared sustained run
and can be recreated without stale input targets or resource growth.

Do not add a virtual DOM, template compiler, CSS cascade, general reactive
runtime, plugin protocol, or third-party compatibility surface.

## Unscheduled candidates

These directions have no current delivery, library, version, or compatibility
commitment. A candidate enters the roadmap only after a representative demo or
embedder proves its scope and ownership.

- [ ] Foundation services: base utilities, filesystem, storage, networking,
      image codecs, and audio.
- [ ] Packaging and embedding: native and WASM runtime ABI after a concrete
      embedder requires it.
- [ ] JavaScript: a VM abstraction, bridge, and minimal API for a selected
      host, without changing Rust-native semantics.
- [ ] Content and rendering: shader tooling, animation, skinning, physics,
      navigation, Blender workflows, and richer scene capabilities.
- [ ] Scale work: additional backends, multi-queue scheduling, parallel command
      recording, and resource aliasing only after representative profiling and
      benchmarks demonstrate the need.
- [ ] ECS, editor tooling, a broader JavaScript SDK, and purpose-built
      declarative UI.

No candidate implies a third-party engine, scene-API, or UI compatibility
facade.
