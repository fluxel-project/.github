# Fluxel Ecosystem

Fluxel is intended to grow as focused libraries with explicit ownership
boundaries, not as one renderer-shaped engine. This document is an ecosystem
map and development direction, not a release schedule, compatibility promise,
or commitment to create every named library.

## Layers

### L0 — Foundation capabilities

`fluxel-base` provides small, stateless, cross-platform utilities such as
Base64, hashing such as MD5 where required, encoding, and byte/string helpers.

`fluxel-time` provides clocks, timers, timeouts, intervals, and frame timing.

`fluxel-fs` provides files, directories, paths, streams, and random-access
read/write operations. `fluxel-storage` separately provides application data
persistence such as key-value data, JSON, blobs, and local storage.

`fluxel-net` provides HTTP, WebSocket, download, and upload capabilities.
`fluxel-input` provides mouse, keyboard, touch, wheel, and gamepad input;
input events belong to this library rather than to a separate events layer.

`fluxel-image` provides image decoding, encoding, and pixel data.
`fluxel-audio` provides audio decoding, playback, volume, and output-device
integration.

`fluxel-platform` provides application startup, windows, screens, DPI,
resizing, the main loop, surfaces, and frame-rate control. Foreground and
background lifecycle policy is added incrementally when a staged demo requires
it.

### L1 — Resources

`fluxel-loader` obtains CPU resources from memory, files, URLs, and supported
formats. It loads data such as textures, meshes, shaders, JSON, and eventually
formats such as GLTF.

`fluxel-assets` owns resource identity, typed handles, generations,
cross-frame references and lifetime, cache/reuse, CPU/GPU residency, and safe
release.

The distinction is intentional: the loader decides how resource data is
obtained; assets decide how an obtained resource remains available and when it
can be released.

### L2 — GPU and rendering

`fluxel-rhi` owns GPU hardware abstraction and backend implementations,
including DX12, Vulkan, and future supported backends.

`fluxel-rendergraph` owns single-frame GPU passes, resource dependencies,
logical synchronization requirements, and portable execution plans. Native
barriers, fences, queues, and synchronization realization remain in
`fluxel-rhi`.

`fluxel-renderer` owns 3D render submission, render packets, ordering,
grouping, batching, instancing, and lowering into the RenderGraph. It consumes
prepared resources; it does not own durable asset identity or host policy.

`fluxel-canvas` will own Canvas-style 2D drawing, including images, sprites,
text, paths, transforms, clipping, blending, and offscreen work.

`fluxel-shader` may later own shader compilation, reflection, variants, and
caching once those concerns have real independent demand. Renderers consume
its outputs rather than owning its toolchain policy.

### L3 — Runtime

`fluxel-runtime` composes foundation capabilities, assets, 2D and 3D
rendering, and platform services into a running application runtime.

`fluxel-runtime-abi` is the stable outer ABI and packaging boundary. It can
package the complete runtime as a DLL, SO, dylib, or WASM module. Its
exports represent the assembled runtime—2D, 3D, resources, networking, files,
and audio—not merely a collection of miscellaneous native functions.

Internal libraries continue to use direct Rust APIs. The ABI boundary exists
for embedders and packaging, not as an internal replacement for Rust types.

### L4 — JavaScript

`fluxel-vm-js` abstracts JavaScript VM integration, including V8, QuickJS,
JavaScriptCore, and web or Edge environments where appropriate.

`fluxel-jsbridge` owns Rust/JavaScript objects, handles, functions, promises,
asynchronous conversion, and data conversion.

`fluxel-js` is the developer-facing JavaScript API. It may eventually take
inspiration from mini-game or Three.js-style APIs, but internal runtime and
renderer design must not be shaped by that public API.

### L5 — Higher-level UI

`fluxel-ui` is a possible future home for purpose-built declarative UI. It may
study mini-program and Vue ergonomics, but it is not a Vue compatibility layer
or `vue-native`; its design is intentionally not specified yet.

## Development stages

Fluxel is a native-first, Three-like, AI-friendly lightweight rendering
runtime. Development follows visible demo closure rather than layer order.
Platform, asset, loader, and runtime boundaries are extracted only when the
current demo needs them. Three.js is an API reference, demo source, and
performance comparison point, not a compatibility target.

### Development principles

#### Native-first semantics

- Rust APIs and native runtime behavior are authoritative. JavaScript, WASM,
  foreign-function, and declarative UI layers adapt those semantics; they do
  not define the renderer or runtime core.
- Native-first does not mean desktop-only. Portable behavior is designed from
  evidence on each target, while backend-specific objects remain behind their
  native boundaries.
- Prefer explicit ownership, borrowing, typed handles, lifecycle states, and
  structured errors. Do not replace a constraint with documentation when the
  Rust API can make the invalid state unrepresentable.

#### Three-like, not Three.js-compatible

- Three.js is a source of API comparisons, small demo scenarios, usability
  questions, and performance workloads.
- Fluxel does not target 100% Three.js API or behavioral compatibility, does
  not act as a backend for Three.js, and will not create a `threejs-native`
  runtime.
- Concepts such as `Scene`, `Camera`, `Mesh`, `Geometry`, `Material`, and
  `Transform` may be adopted only when they remain natural Rust concepts.
  Three.js inheritance, plugin contracts, implicit global state, material
  breadth, and `Three*` compatibility types must not enter the native core.
- A future JS or TS facade must remain removable and replaceable. Compatibility
  work, if ever reconsidered, requires a separate explicit decision and cannot
  be inferred from the phrase “Three-like.”

#### Purpose-built UI, not Vue-native

- Fluxel UI grows from demonstrated game HUD, settings, and application-screen
  needs after Canvas, text, input, and runtime lifecycle are proven.
- Fluxel does not target 100% Vue, DOM, or CSS compatibility and will not build
  a `vue-native` clone. Vue templates, reactivity, component lifecycle, and
  plugin behavior are not contracts for the native core.
- Declarative ideas may be studied later, but the retained-mode or declarative
  surface must follow Fluxel ownership and resource-lifetime semantics.

#### Demo-driven scope

- Development order is determined by the next visible, testable demo closure,
  not by completing every library in layer order.
- Reuse and extend the current demo. Add assets, loaders, platform services,
  runtime composition, and higher layers only when the demo exposes the need.
- Do not broaden a public API for a hypothetical later platform. Record the
  missing case and wait for a stage that can test it on the real target.
- Optimize only after a representative baseline and profile identify the cost.
  Backend count, feature count, and abstraction count are not progress by
  themselves.

#### AI-friendly engineering

- Provide one recommended path for each capability. Avoid parallel “simple,”
  “advanced,” legacy, backend-specific, and convenience paths that encode the
  same operation differently.
- Keep names, module layout, lifecycle terms, and backend structure stable and
  symmetric. A future agent should be able to predict where an implementation,
  test, example, and document belong.
- Prefer small explicit APIs, typed errors, visible state transitions, and
  deterministic behavior. Avoid macro-generated public surfaces, magic
  registration, hidden global state, stringly typed contracts, and implicit
  resource cloning.
- Use the same nouns and operation order in code, demos, tests, diagnostics,
  and documentation. When a term changes, update all of them in the same
  release rather than leaving aliases for speculative compatibility.
- Repositories participating in a staged change must state in `AGENTS.md` where
  the capability is owned, which files normally change, which focused commands
  verify it, and which real platform evidence is required. A green mock or
  compile check must never be described as hardware correctness.

This section is an ordered execution map. Later work must follow these rules:

- Advance one small release at a time. Finish its plan, API review,
  implementation, focused tests, independent implementation review, and
  evidence before selecting the next release.
- Do not skip a stage because a later API is easier to design in isolation.
  Reduce the current stage instead if its full gate cannot yet be proved.
- Extend the same scene and application unless a platform requires a minimal
  host-specific shell. A new demo must not hide a broken earlier path.
- Layer ownership determines where code lives, not when a crate must be
  created. Extract a crate only after the current vertical slice demonstrates
  an independently changing responsibility.
- Keep native resources, barriers, commands, submission, and readback inside
  `fluxel-rhi`. Keep RenderGraph limited to declarations, dependency plans,
  portable semantics, and validation. Keep durable identity and host lifecycle
  out of `fluxel-renderer`.
- Treat compile, mock, and `TestRhi` results as logic evidence only. A gate that
  names a GPU, browser, mini-game host, or mobile device requires evidence from
  that real target.
- Before implementing a stage, freeze its target matrix and measurable oracle:
  OS and version, browser or host where relevant, backend, device class,
  expected visual comparison method, sustained-run duration and workload,
  sampling method, and pass/fail thresholds. A review may change the matrix
  explicitly; implementation must not silently narrow it to make the gate pass.
- Record the highest stage and capability set proved for each target. A later
  platform-specific stage does not automatically claim its new capabilities on
  earlier targets, but any change to shared code must rerun the affected earlier
  oracle or mark that target unsupported by an explicit reviewed decision.

### Stage 0 — Freeze the current baseline

Turn the current headless vertical slice into a reproducible baseline before
adding more fixed rendering recipes.

Complete these work packages in order:

1. **Release identity.** Use one version for the workspace crates that ship
   together. Define the tag format and make every Git dependency example pin a
   tag or revision rather than a moving branch. The documented checkout, test,
   and GPU-evidence commands must all identify the same commit.
2. **Error preservation.** Keep renderer failures structured through the
   public renderer boundary. Context may be added, but backend, validation,
   resource, and submission failures must not be flattened into `String` or
   become recoverable only by parsing text.
3. **GPU conformance entry point.** Provide one repeatable script or command
   that records commit SHA, OS, target triple, backend, adapter, device, driver,
   invoked tests, pass/fail results, and diagnostics. It must run the relevant
   ignored real-GPU tests in both RHI and renderer and retain partial evidence
   when a test fails.
4. **RenderGraph public-surface audit.** Enumerate the exports used by current
   callers. Remove accidental backend and implementation exports. Do not move
   native resource, barrier, command, submission, or readback behavior into
   RenderGraph to make the public API look convenient.
5. **Fixed-recipe freeze.** Repair existing headless paths as needed, but add
   no new `draw_*` family member and no new public upload-state type. A request
   for another recipe is evidence that Stage 1 or the later object model must
   replace recipe growth.

Stage 0 does not add a window, surface API, multi-frame API, scene object model,
loader, asset manager, or runtime crate. It may make the minimum compatibility
fixes needed to preserve the existing verified slice.

This stage is complete only when one fixed commit reproduces the documented
CPU and CI results and the retained real-GPU report. The report must distinguish
unsupported hardware or unavailable validation layers from an implementation
failure; missing hardware evidence leaves the stage open.

### Stage 1 — Windows visible renderer

Build one durable Windows scene before broadening the runtime.

Stage 1 is split into five ordered closures. Do not start the next closure while
the previous one has an unresolved lifetime, validation, or public-contract
problem.

#### 1.1 — DX12 first image

- Create one Windows window, one DX12 device path, and the surface or swapchain
  state needed by the demo.
- Clear the acquired image, draw a fixed triangle, present it, handle the close
  event, wait for required GPU work, and release resources in a valid order.
- Keep the window integration narrow. It may live in an example or temporary
  host boundary; do not create a general `fluxel-platform` API from a single
  window path.
- Keep HWND, DXGI, command queues, fences, descriptors, and other native types
  behind the RHI native boundary. Renderer and RenderGraph public APIs must not
  acquire DX12-specific variants.

The closure requires a retained image, backend and adapter diagnostics, clean
shutdown evidence, and focused tests for any new platform-independent state.
A clear-only window, an offscreen image, or a triangle that is never presented
does not close 1.1.

#### 1.2 — Surface lifecycle

- Treat resize, zero width or height, minimization, restoration, and close as
  distinct host events rather than one boolean state.
- Do not acquire or submit a drawable frame while the surface size is invalid.
  Resume only after a valid size is observed and required surface resources are
  recreated.
- Define which resources survive resize and which are retired before
  replacement. Never destroy or overwrite swapchain-dependent state still in
  use by the GPU.
- Return structured, diagnosable device, surface, acquisition, and presentation
  failures. Known validation noise must be documented explicitly; it must not
  be silently ignored.

The closure requires scripted or repeatable resize, minimize, restore, and exit
sequences plus a real DX12 run that completes them without an unexplained
validation error.

#### 1.3 — Multi-frame lifetime

- Validate three frames in flight by default, but expose no `TripleBuffer`,
  numeric ring-slot, fence value, or backend synchronization object as the
  public abstraction.
- Public semantics are limited to frame acquisition, frame submission,
  completion observation, and retirement. Ring indexing and backend fence or
  timeline mechanics remain private.
- Associate transient allocations, uploads, descriptors, and other reused
  frame resources with the completion that makes reuse safe. The CPU must not
  overwrite data referenced by unfinished GPU work.
- When the GPU falls behind, wait or throttle at a documented boundary. Do not
  allocate without limit, silently reduce correctness to one frame in flight,
  or rely on driver serialization.
- Exercise normal progress, delayed completion, back pressure, shutdown with
  outstanding work, and the relevant failure paths. Logic tests establish the
  state machine; a real-GPU run establishes backend behavior.

The closure requires a continuous multi-frame DX12 run and evidence that at
least three reusable frame contexts cycle safely under induced or observed GPU
lag. The public contract must remain valid if the private in-flight count later
changes.

#### 1.4 — Vulkan alignment on Windows

- Run the same executable-level demo scene through Vulkan. Use equivalent
  shader inputs and rendering semantics and compare against the same expected
  image; compiled shader binaries and backend setup may differ.
- Apply the same resize, minimization, restoration, acquisition, submission,
  completion, retirement, and shutdown contract used by DX12.
- Fix portable contract errors above the backend. Keep Vulkan layouts,
  synchronization primitives, queues, and surface objects within RHI.
- Do not weaken the DX12 path or expose a lowest-common-denominator multi-queue
  API merely to make the implementations textually symmetric.

The closure requires retained DX12 and Vulkan results from the same commit,
with backend-specific validation enabled and any expected diagnostic difference
recorded.

#### 1.5 — Minimal multi-object scene

- Replace the fixed-triangle-only call path with the minimum composition needed
  for one camera, multiple meshes, multiple materials, and an independent
  transform per object.
- Define deterministic draw order for identical input. Reuse compatible
  pipelines and bindings without promising a general batching, instancing, or
  visibility system.
- Use embedded or generated resources so loader and asset lifetime do not enter
  this stage. Cross-frame durable identity remains deferred to Stage 4.
- Keep ordinary scene construction above RHI and RenderGraph. Backend-specific
  handles must not be required to create a mesh, material, transform, or
  camera.

Stage 1 is complete when the same multi-object scene runs continuously on DX12
and Vulkan, survives the lifecycle sequence, shuts down cleanly, and has no
unexplained validation error. Retain expected and actual images or recordings,
compared by the exact or tolerance-based oracle frozen in the stage plan, plus
frame-lifetime diagnostics, device and driver metadata, and the first
representative CPU-submission and frame-time baseline.

### Stage 2 — Web and one mini-game host

Keep the Stage 1 scene and expected image instead of starting a new demo.
Before implementation, the stage plan names the browser, browser version, OS,
GPU, and selected mini-game host used as real-target oracles; “works on Web” is
not an acceptable unnamed environment.

#### 2.1 — WebGL2 closure

- Create the canvas and WebGL2 context, map logical size and device-pixel size
  explicitly, and resize without stretching or rendering into stale targets.
- Render the Stage 1 camera, objects, materials, transforms, and deterministic
  draw order with the same expected visual result.
- Handle page visibility changes and context loss or restoration. Recreate or
  rebind GPU resources from retained CPU-side information instead of assuming
  WebGL objects survive.
- Record release bundle size, cold startup time, CPU submission cost, and
  memory for the fixed scene. Define the measurement procedure before using
  the numbers as regression guards.

Do not redesign the native APIs around WebGL global-state conventions. WebGL2
backend limitations must either have a portable structured fallback or be
declared unsupported.

#### 2.2 — WebGPU closure

- Run the same scene, camera, transforms, material semantics, expected image,
  canvas sizing behavior, and long-running loop through WebGPU.
- Exercise surface reconfiguration and device loss, including failure to
  restore. The public error must retain whether recovery, recreation, or
  application restart is required.
- Validate that RenderGraph declarations and portable semantics are sufficient
  without exposing WebGPU objects or adding a public multi-queue model.
- Compare WebGPU and WebGL2 using the same measurement definitions. The gate is
  per-backend stability and regression detection, not identical performance.

#### 2.3 — One real mini-game host

- Select and name exactly one host before implementation. Do not claim generic
  mini-game support from a browser wrapper or an untested abstraction.
- Prove its startup and shutdown lifecycle, foreground/background transitions,
  canvas or surface ownership, resize rules, minimal input, packaged resource
  paths, and the backend actually available on a device.
- Add only the host adapter required by the shared demo. Do not define a broad
  platform or JavaScript SDK from one host.

Stage 2 is complete only when WebGL2, WebGPU, and the selected mini-game host
each retain expected and actual output, lifecycle diagnostics, environment
metadata, and their scoped performance baseline from one fixed revision.
Browser evidence cannot substitute for real mini-game-device evidence, and one
web backend cannot close the other.

### Stage 3 — Three-like API probe

Express one basic Three.js example with Fluxel-native `Scene`, `Camera`,
`Mesh`, `Geometry`, `Material`, and `Transform` concepts. Ordinary users must
not touch RHI or RenderGraph. Rust defines the semantics, while a JS or TS
adapter remains replaceable. Do not import Three.js inheritance, plugin
contracts, material breadth, or `Three*` special cases into the native core.

The probe must answer these questions before it becomes public API:

- Can a normal user construct the scene without importing RHI or RenderGraph
  types, supplying raw GPU handles, or manually sequencing barriers?
- Can the recommended application path acquire a frame, render and present the
  scene, respond to resize or device loss, and recover from structured errors
  without exposing RHI or RenderGraph to ordinary scene code?
- Are object ownership, shared geometry or material use, transform mutation,
  camera selection, and removal from the scene explicit in Rust?
- Is draw order deterministic, and can an error identify the object or
  resource that caused it without relying on a global registry?
- Is the example short because the model is coherent, rather than because a
  macro, hidden singleton, implicit clone, or magic registration performs the
  work?
- Can a thin JS or TS experiment map to the Rust concepts without becoming the
  source of truth or adding `Three*` types to native crates?

Use one small Three.js demo as the comparison case and document deliberate API
and behavior differences. Do not add a general scene hierarchy, complete
material catalog, loader ecosystem, animation system, or plugin API during the
probe.

Stage 3 is complete when the same concise Fluxel example runs on the already
proven DX12, Vulkan, WebGL2, WebGPU, and selected mini-game-host paths, with the
Rust API documented as the authoritative contract. An attractive sketch that
has not survived those targets is not sufficient.

### Stage 4 — Windows playable runtime

Extend the existing multi-object scene into a small playable application.
This is the first stage where long-lived application ownership is the subject,
so `fluxel-platform`, `fluxel-time`, `fluxel-input`, `fluxel-loader`,
`fluxel-assets`, and `fluxel-runtime` may be extracted as their responsibilities
become real. Their existence is not a prerequisite for starting the stage.

Complete the work in this dependency order:

1. **Host loop and time.** Turn the Stage 1 window harness into an application
   loop with explicit startup, frame, pause, resume, close, and shutdown
   behavior. Separate measured delta time from fixed simulation updates and
   define clamping or catch-up behavior so a pause cannot create an unbounded
   update burst.
2. **Input and camera control.** Add only the keyboard, mouse, or controller
   inputs needed to operate the demo. Platform events are translated into
   Fluxel input values before game or camera logic consumes them; renderer APIs
   do not receive window messages.
3. **File loading and decoding.** Replace selected embedded scene data with
   files and the minimum image and mesh formats needed by the demo. The loader
   owns obtaining and decoding CPU data. It does not assign durable identity or
   decide GPU retirement. A decoder may remain a focused dependency until a
   reusable `fluxel-image` boundary is demonstrated; do not create a format
   framework or add GLTF without a chosen asset that requires it.
4. **Durable asset identity.** Introduce typed handles, generations, explicit
   invalid-handle behavior, cache keys, and cross-frame references. Assets own
   CPU/GPU residency, reuse, replacement, and safe release; renderer packets
   only consume prepared resources for a frame.
5. **Cross-frame reuse and retirement.** Load shared geometry and textures once,
   reuse them across objects and frames, and release GPU state only after both
   asset ownership and relevant frame completions allow it. Test stale
   generations, duplicate requests, failed uploads, dropped owners, and device
   or surface recreation.
6. **Asynchronous loading.** Add asynchronous work only after the synchronous
   ownership path is correct. Define cancellation, failure, duplicate in-flight
   request handling, readiness observation, and shutdown. Do not select or
   expose an async runtime merely because loading is asynchronous.
7. **Runtime composition.** Extract `fluxel-runtime` when the demo has a stable
   composition boundary across platform, time, input, assets, and rendering.
   The runtime coordinates these services; it must not absorb their internal
   policy or replace direct Rust APIs with an ABI.

Stage 4 does not add networking, storage, audio, a general scripting layer, a
complete file-format catalog, an ECS, or an editor. It does not redesign the
Stage 3 object model unless the playable demo exposes a concrete ownership or
usability failure. Its new playable-runtime capability is Windows-only until a
later stage proves otherwise; it must not be advertised for Web or the selected
mini-game host.

Stage 4 is complete when the Windows demo is operable, pauses and resumes
correctly, shuts down with outstanding work safely resolved, and survives a
documented long run. Evidence must show that duplicate requests reuse one
asset, stale handles fail structurally, GPU resources retire after their last
safe use, and memory does not grow without explanation. Changes to shared
renderer, scene, RenderGraph, or RHI code must also rerun the affected Stage 3
scene oracle on DX12, Vulkan, WebGL2, WebGPU, and the selected mini-game host;
this regression check does not claim that Stage 4 loading or runtime features
have been ported to those targets.

### Stage 5 — Android and iOS runtime

Port the Stage 4 application rather than creating mobile-only renderer demos.
Keep scene, asset identity, fixed-update behavior, and expected visual result
shared; isolate only host lifecycle, input mapping, surface integration,
resource packaging, and backend-specific RHI code.

Before implementation, the stage plan names the Android and iOS versions,
toolchains, real devices, GPU backends, visual oracle, sustained-run workload,
and minimum duration. These targets may be revised by review but not selected
after seeing which configuration passes.

#### 5A — Android gate

- Run the playable demo on a named Android target and real device using the
  selected supported backend and NDK/toolchain combination.
- Map touch input, pause updates while backgrounded, and recover after
  foregrounding without retaining an invalid surface.
- Handle surface destruction and recreation independently of process lifetime,
  including orientation and size changes.
- Retain device, OS, GPU, driver, memory, and thermal observations from a
  sustained run. Emulator compile or launch evidence is useful but cannot
  replace the device gate.

#### 5B — iOS gate

- Run the same playable demo on a named iOS version and real device through
  Metal.
- Map touch input and handle inactive, background, and foreground transitions
  without submitting to an unavailable surface.
- Recreate surface-dependent state for size, orientation, scale, and safe-area
  changes while preserving only resources whose lifetime remains valid.
- Retain device, OS, GPU, memory, and sustained-run observations. Simulator
  evidence cannot replace the device gate.

Android and iOS are accepted independently. Stage 5 closes only after both
gates pass; success on one platform must not be generalized to “mobile
support.” Platform fixes must preserve the Windows and Web contracts already
proved, plus the selected mini-game-host contract, unless a documented portable
contract change is reviewed explicitly.

### Stage 6 — Canvas and minimal text

Build one shared Canvas demo on the runtime and resource lifecycle already
proved. `fluxel-canvas` owns 2D drawing semantics and lowers work into the GPU
layers; it does not become a second platform loop, asset manager, or UI system.
The stage plan freezes the platform/backend matrix inherited from earlier
stages and records any deliberate exclusion before implementation.

The first Canvas slice contains images or sprites, 2D transforms, rectangular
clipping, alpha blending, explicit layer or submission order, and an offscreen
target used by the demo. Define coordinate origin, pixel scaling, color and
alpha convention, clipping behavior, and ordering deterministically across
backends.

The first text slice contains one selected font path, LTR text, a declared
basic character set, basic line breaking, fixed alignment choices, and a glyph
cache integrated with asset and GPU retirement. Missing glyphs, cache growth,
cache invalidation, device loss, and release must have defined behavior.

Paths, arbitrary brushes, complex shaping, bidirectional text, fallback font
stacks, rich text, broad multilingual layout, and CSS layout are excluded.
Stage 6 closes when the same fixed Canvas scene passes its documented visual
comparison on every target frozen in the stage plan and sustained runs meet the
predeclared glyph-cache, memory, and resource-lifetime thresholds.

### Stage 7 — UI closure

Use the existing runtime, Canvas, text, and input to build one durable game HUD
or settings page. Start from a concrete screen and list its required layout
rules before designing the API. The stage plan also freezes the target platform
matrix, interaction script, sustained-run duration, and memory or cache limits.

The slice may include basic parent/child layout, text and images, click and
touch interaction, platform-relevant hover or focus, state-driven updates,
show/hide behavior, page creation and destruction, and resource release. Event
delivery order, hit testing, focus ownership, removal during interaction, and
the boundary between UI state and application state must be explicit wherever
the chosen screen exercises them.

Do not add a virtual DOM, template compiler, CSS cascade, general reactive
runtime, component plugin protocol, or Vue compatibility surface. Reuse Canvas,
text, input, assets, and runtime lifecycles rather than creating UI-specific
copies of those systems.

Stage 7 closes when the chosen HUD or settings page remains interactive during
the predeclared sustained run on every target in that matrix, updates
deterministically, and can be hidden, destroyed, and recreated without stale
input targets or exceeding the declared resource and cache thresholds.

### Evidence required at every stage

A stage review must point to all applicable artifacts below rather than merely
state that the stage works:

- one runnable demo and the exact command or host procedure used to launch it;
- fixed inputs, expected output, and actual screenshots, recordings, or
  headless output captures suitable for comparison;
- for visual output, an exact hash where stable, or a documented tolerance,
  mask, and review procedure where backend or text-rasterization differences
  are expected;
- focused automated tests for new logic, important failure paths, lifecycle
  transitions, and regressions introduced by the stage;
- commit SHA, OS and version, target triple, backend, device or adapter, driver,
  validation configuration, and relevant diagnostics;
- a named representative workload, measurement method, raw result, and
  regression threshold for startup, CPU submission, frame time, memory, bundle
  size, or other metrics that the stage changes;
- one minimal example using the recommended public path, with no private or
  backend-specific escape hatch;
- updated API and ownership documentation using the same terminology as code,
  tests, examples, and diagnostics;
- an explicit list of unsupported behavior, untested targets, accepted known
  diagnostics, and deferred work.

Stage 0 retains conformance output and diagnostics in place of a window
capture. Compilation, linking, mocks, `TestRhi`, simulators, and emulators may
supplement evidence but do not replace a real target named by the stage. A run
in the browser and version selected by the stage plan is real-target evidence
for WebGL2 or WebGPU; browser evidence cannot substitute for the selected
mini-game host or a mobile device. Performance comparisons are meaningful only
within the documented workload and environment; they do not promise parity
across devices.

### Unscheduled candidate stages

The following directions preserve the broader ecosystem map but are not yet
ordered or committed. A candidate enters the main sequence only after a demo
or embedding requirement establishes its scope and ownership boundary.

#### Broader application services

Add `fluxel-base`, `fluxel-fs`, `fluxel-storage`, `fluxel-net`, `fluxel-image`,
and `fluxel-audio` as real applications require them. Each service needs one
end-to-end consumer; creating every foundation crate is not itself a stage.

#### Runtime packaging and embedding

Define `fluxel-runtime-abi` only after a concrete embedder needs a stable outer
boundary. Native library and WASM packaging then prove that same assembled
runtime contract rather than exposing unrelated native functions.

#### JavaScript runtime surface

Add the VM abstraction, Rust/JavaScript bridge, and minimal JavaScript API for
a selected host. Rust remains the semantic authority, and the JS surface does
not reshape the renderer or runtime core around Three.js conventions.

#### Richer rendering and content workflows

Candidates include shader tooling, animation and skinning, physics,
navigation, Blender workflows, richer scene capabilities, and broader text or
UI. Each needs a representative demo before its API or crate boundary is
chosen.

#### Scale and optimization

Additional GPU backends, multi-queue scheduling, parallel command recording,
resource aliasing, and similar work require profiling on representative
workloads. Backend count and architectural complexity are not goals by
themselves.

ECS, editor tooling, a broader JavaScript SDK, and purpose-built declarative UI
remain open possibilities. Three.js- or Vue-compatible facades,
`threejs-native`, and `vue-native` are outside this roadmap. Reconsidering that
boundary would require replacing these principles explicitly, not merely
adding an unscheduled candidate stage.
