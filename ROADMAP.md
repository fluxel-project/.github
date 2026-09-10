# Fluxel Roadmap

Fluxel is a native-first, Three-like, AI-friendly lightweight rendering
runtime. “Three-like” describes approachable scene concepts, not Three.js API
compatibility. This is an execution map, not a release schedule or a promise
to create every named crate.

**Current target:** reproducible baseline, then the first visible DX12 window.

Development follows the next visible demo closure, not the layer map. Reuse
and extend one scene and application wherever possible. Extract a crate only
when that demo proves an independently changing ownership boundary.

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

**Status:** In progress.

**Latest retained evidence:** [`fluxel-renderer` v0.7.0](https://github.com/fluxel-project/fluxel-renderer/releases/tag/v0.7.0),
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
- [ ] Re-run the complete baseline on the final fixed Stage 0 commit and retain
      the verified Release artifacts.

**Close when:** the final fixed Stage 0 commit reproduces documented CPU, CI,
and retained real-GPU evidence. The v0.7.0 evidence is the latest baseline, not
automatic closure for a later commit.

## Stage 1 — Windows visible renderer

Build the durable Windows rendering path. The detailed work sequence, API
boundaries, and evidence for this stage are in
[Stage 1: Windows](stages/stage-01-windows.md).

**Boundary:** Windows only. Native DX12 and Vulkan objects stay in RHI. This
stage proves a visible renderer, not assets, loading, a general platform API,
or a playable runtime.

**TODO**

- [ ] 1.1 DX12 first image: create a window and surface, clear, draw a fixed
      triangle, present, close cleanly, and release in a valid order.
- [ ] 1.2 Surface lifecycle: handle resize, minimize, restore, invalid sizes,
      recreation, and structured surface or device failures.
- [ ] 1.3 Multi-frame lifetime: prove safe reuse with three frames in flight
      by default without exposing a triple-buffer API.
- [ ] 1.4 Vulkan alignment: run the same scene and lifecycle contract on
      Windows Vulkan.
- [ ] 1.5 Minimal multi-object scene: add a camera, meshes, materials,
      independent transforms, deterministic order, and minimal reuse of
      compatible pipelines and bindings.

**Close when:** the same scene runs continuously on DX12 and Vulkan, survives the
defined lifecycle sequence, shuts down cleanly, and retains visual,
diagnostic, and baseline evidence without unexplained validation errors.

## Stage 2 — Web and one mini-game host

Keep the Stage 1 scene and expected image. Name the browser and exactly one
mini-game host in the stage plan before implementation.

**Boundary:** support WebGL2, WebGPU, and one named mini-game host separately.
Do not claim generic web or mini-game support, redesign native APIs around web
global state, or use browser evidence for the host device.

**TODO**

- [ ] 2.1 WebGL2: canvas creation and resize, multi-object rendering,
      visibility and context recovery, resource rebinding, and scoped package,
      startup, CPU-submission, and memory measurements.
- [ ] 2.2 WebGPU: the same scene and expected output, surface reconfiguration,
      device-loss behavior, and portable RenderGraph semantics without a public
      multi-queue API.
- [ ] 2.3 One real mini-game host: prove device startup, lifecycle, canvas or
      surface, minimal input, packaged resource paths, and foreground or
      background behavior.

**Close when:** WebGL2, WebGPU, and the selected mini-game host each have their own
real-target visual, lifecycle, diagnostics, and scoped performance evidence.
Browser evidence does not close the mini-game host gate.

## Stage 3 — Three-like API validation

Use one small Three.js example only as a comparison case. Express it with
Fluxel-native `Scene`, `Camera`, `Mesh`, `Geometry`, `Material`, and
`Transform` concepts. Ordinary scene code must not use RHI or RenderGraph.

**Boundary:** Rust owns the semantics. This validates a small object model; it
does not create a complete scene framework, a Three.js backend, or a JavaScript
source of truth.

### Stage 3A — Low-cost API probe

**TODO**

- [ ] Design the smallest coherent Rust object model for the example.
- [ ] Make ownership, sharing, mutation, removal, deterministic draw order,
      and structured errors explicit.
- [ ] Keep JS or TS experiments thin and replaceable; Rust remains the
      authority.
- [ ] Run the example on DX12 and WebGL2, chosen as distinct representative
      environments for finding API mistakes early.
- [ ] Record deliberate differences from the source Three.js example.

**Close when:** the example exposes one coherent Rust API on DX12 and WebGL2,
and the review records the API mistakes found and the deliberate differences
from Three.js. Stage 4 may proceed with this API marked experimental.

### Stage 3B — API freeze for supported targets

**TODO**

- [ ] Before publicly committing the API, run the frozen example on every
      affected supported target.
- [ ] Resolve or document target-specific semantic differences without leaking
      backend objects into scene APIs.
- [ ] Publish one recommended scene construction and rendering path.

**Close when:** the resulting public contract passes every affected supported
target before it is declared stable. Stage 3B is a public-API freeze gate, not
a reason to run the full platform matrix during Stage 3A.

Do not add Three.js inheritance, plugin contracts, `Three*` compatibility
types, a complete material catalog, or a general scene framework. Fluxel does
not target full Three.js compatibility, does not provide a Three.js backend,
and will not create `threejs-native`.

## Stage 4 — Windows playable runtime

Extend the existing multi-object scene into a small long-running Windows
application. Create `fluxel-platform`, `fluxel-time`, `fluxel-input`,
`fluxel-loader`, `fluxel-assets`, and `fluxel-runtime` only when the following
work demonstrates their separate ownership.

**Boundary:** Windows playable runtime only. Create boundaries on demonstrated
need, not because they appear in the layer map. Do not add networking, storage,
audio, ECS, editors, general scripting, or a broad file-format framework.

**TODO**

- [ ] Add a host loop with startup, frame, pause, resume, close, shutdown,
      delta time, and fixed-update behavior.
- [ ] Add only the required input, file or memory loading, mesh and image
      decoding, and camera or game control.
- [ ] Establish typed handles, generations, reuse, CPU/GPU residency, and
      retirement; prove shared resources load once and release safely.
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
Do not create a general UI framework, virtual DOM, CSS system, or Vue
compatibility layer.

**TODO**

- [ ] Add only required basic layout, text, images, interaction, state updates,
      show or hide behavior, page destruction, and resource release.
- [ ] Define relevant hit-testing, event order, focus, and removal semantics.
- [ ] Prove the screen remains interactive and can be recreated without stale
      input targets or resource growth.

**Close when:** the chosen screen is interactive for its declared sustained run
and can be recreated without stale input targets or resource growth.

Do not add a virtual DOM, template compiler, CSS cascade, general reactive
runtime, plugin protocol, Vue compatibility surface, or `vue-native`.

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

No candidate implies a Three.js-compatible facade, `threejs-native`,
Vue-compatible UI, or `vue-native`.
