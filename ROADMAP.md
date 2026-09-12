# Fluxel Roadmap

Fluxel is a native-first, Three-like, AI-friendly lightweight rendering
runtime. “Three-like” describes approachable scene concepts, not Three.js API
compatibility. This is an execution map, not a release schedule or a promise
to create every named crate.

**Latest closure:** 0.10 / Stage 2.2: retain the Stage 1 scene in WebGPU on
Windows 11 x64, Google Chrome Stable `153.0.8010.36` on the AMD Radeon 780M
driver `32.0.21028.2002`. The final rendering and JS-bridge commits are
`8d18080efb9c14cc14ef05861660f8a7ed856309` and
`65631997dbdc34c3ad2b44c91a099b507c72ead9`.
This is a named-target WebGPU claim only; it neither widens WebGL2 support nor
closes the separately unselected mini-game gate. See
[Stage 2.2: WebGPU](stages/stage-02-webgpu.md).

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
      or authorized by 0.9.

**Close when:** WebGL2, WebGPU, and the selected mini-game host each have their own
real-target visual, lifecycle, diagnostics, and scoped performance evidence.
Browser evidence does not close the mini-game host gate.

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
recovery into a new device generation, and async dispose. The WebGPU path keeps
canvas context, device, queue, pipeline/buffer objects, completion tickets,
and recovery private to rendering WASM/RHI; the JS bridge owns only DOM
lifecycle and one RAF. The candidate evidence records bounded `N=3`
completion-driven admission, diagnostics, scoped measurements, and visual plus
pixel/frame-marker samples. It is evidence for 2.2 only: WebGL2 remains its
own 2.1 closure and the mini-game gate remains open.

The durable releases are
[`fluxel-rendering` v0.10.0](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.10.0)
and [`fluxel-jsbridge` v0.2.0](https://github.com/fluxel-project/fluxel-jsbridge/releases/tag/v0.2.0).
The downloaded `fluxel-rendering-v0.10.0-evidence.zip` is 60,347 bytes with
SHA-256 `94280a7ad6cccf518a1c8975326fe847f448956681a1c97de93955d72691b8fd`.

## Stage 3 — Three-like API validation

Use one small Three.js example only as a comparison case. Express it with
Fluxel-native `Scene`, `Camera`, `Mesh`, `Geometry`, `Material`, and
`Transform` concepts. Ordinary scene code must not use RHI or RenderGraph.

**Boundary:** Rust owns the semantics. This validates a small object model; it
does not create a complete scene framework, a Three.js backend, or a JavaScript
source of truth.

**Owning repository:** `fluxel-rendering`.

**Participating repositories:** `fluxel-jsbridge`.

**Artifact under test:** the same experimental scene rendered by the
`fluxel-rendering` DX12 executable and the JS-bridge WebGL2 integration.

**Cross-repository contract changes:** expose the experimental Rust scene
concepts through a thin, replaceable JS bridge only as needed by the example.
Rust remains the semantic authority; no Three.js compatibility or independent
JS scene contract is created.

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
application. Introduce internal platform, time, input, loader, asset, and
application-composition boundaries in their owning repositories only when the
following work demonstrates their separate ownership.

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
Do not create a general UI framework, virtual DOM, CSS system, or Vue
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
