# Fluxel Ecosystem

Fluxel is a native-first, AI-friendly lightweight rendering ecosystem with an
approachable scene API and no implied third-party compatibility. Rust rendering
and host contracts are authoritative. Development
follows visible demo closure rather than completing architectural layers in
advance.

> **Latest closure:** 0.10 / Stage 2.2 retains the Stage 1 scene in WebGPU on
> its named Windows 11 x64, Google Chrome Stable `153.0.8010.36`, AMD Radeon
> 780M target. This adds only that named WebGPU claim: the 0.9 WebGL2 closure
> remains separate, and the mini-game host still prevents Stage 2 as a whole
> from closing.

## What Fluxel is

- Four focused monorepositories with explicit, one-way ownership:
  `fluxel-bases`, `fluxel-rendering`, `fluxel-host`, and `fluxel-jsbridge`.
- A portable design proven by real DX12, Vulkan, WebGL2, WebGPU, and platform
  evidence as those targets enter the roadmap.
- A scene-API usability experiment where concepts are adopted only when they
  remain natural, typed Rust APIs.
- A demo-driven project: the next visible closure decides what must exist next.
- When that closure exposes a capability belonging to another repository, its
  owner supplies the smallest complete behavior that the closure immediately
  consumes and verifies; it is neither copied locally nor expanded into an
  unproven framework.

## What Fluxel is not

- Not a backend or compatibility layer for another scene engine.
- Not a third-party UI, DOM, or CSS compatibility layer.
- Not a plan to create every candidate crate before a running demo needs it.
- Not a place where compile, mock, or `TestRhi` results are presented as
  real-device correctness.

## Layer map

Repository ownership determines where code belongs. It does not determine
development order; candidate crates are extracted only when a demonstrated
consumer proves their independent boundary. The complete ownership and
dependency map is in [Ecosystem architecture](https://github.com/fluxel-project/.github/blob/main/ECOSYSTEM_ARCHITECTURE.md).

- **`fluxel-bases`:** the dependency leaf for cross-platform mechanisms and
  contracts. It owns small shared types, asset identity and lifecycle, loading
  policy, portable time/input values, image data, and diagnostic schema/routing;
  it does not own GPU resources, platform I/O, or output sinks.
- **`fluxel-rendering`:** embeddable GPU and rendering libraries: RHI,
  RenderGraph, renderer submission, rendering-side asset residency, Canvas,
  UI, shaders, and native/WASM packaging. It has no main loop, platform I/O,
  input collection, audio, video, or storage.
- **`fluxel-host`:** native platform implementation and executable host:
  lifecycle, windows and surfaces, platform I/O, input/time acquisition,
  audio/video, native diagnostic sinks, and EXE, APK/AAB, or IPA packaging. It
  may depend on bases and rendering.
- **`fluxel-jsbridge`:** the default JS SDK with browser, mini-game, and native
  adapters. It composes lower APIs into a uniform developer experience, but JS
  is replaceable by SDKs in other languages and does not define core semantics.

The permitted dependency direction is `fluxel-host -> fluxel-rendering ->
fluxel-bases`, with `fluxel-jsbridge` depending on rendering WASM and/or host
APIs. `fluxel-bases` never depends upward; rendering never depends on host or
the JS SDK.

## Development stages

Every stage defines a boundary, a TODO list, and a completion gate in the
[full roadmap](https://github.com/fluxel-project/.github/blob/main/ROADMAP.md).
Descriptions become intentionally coarser as distance from the current target
increases; that coarseness is not permission to exceed the stated boundary.

### Stage 0 — Reproducible baseline

**Boundary:** close the current headless slice. Do not add windows, scene APIs,
new fixed recipes, assets, loaders, or runtime composition.

**Status:** Complete.

**Latest retained evidence:** [`fluxel-rendering` v0.7.0](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.7.0),
with 83/83 ignored real-GPU cases passing on AMD Radeon 780M Graphics across
DX12 and Vulkan.

- [x] Unify workspace release and Git revision rules.
- [x] Preserve structured renderer errors.
- [x] Make CPU, CI, and real-GPU conformance evidence reproducible.
- [x] Audit RenderGraph public exports.
- [x] Freeze new `draw_*` recipes and upload-state types.
- [x] Retain the complete baseline on the final Stage 0 source commit.

`v0.7.0` / `6bd3a25` is the fixed source and evidence commit. Later
repository-alignment commits changed documentation only and do not define a
second GPU certification target.

### Stage 1 — Windows visible renderer

**Boundary:** one persistent Windows scene on DX12 and Vulkan. Keep native types
inside RHI; do not introduce loaders, durable assets, or a public triple-buffer
API.

`fluxel-host` supplies only the Win32 Window primitive required by this first
proof (window/HWND lifetime, message pumping, close observation, and standard
window/display handles). The executable remains in `fluxel-rendering`; RHI owns
surface, swapchain, presentation, and GPU synchronization, and rendering does
not depend on Host.

- [x] Present the first DX12 triangle (`fluxel-rendering` `v0.8.0`; 120 dense
  visual samples across 899 presented frames, clean validation and shutdown).
- [x] Close resize, minimize, restore, and surface recreation
  (`fluxel-rendering` `v0.8.1`; generations 1–4, 120 timestamped samples / 240
  screenshots, clean validation and shutdown; `fluxel-host` `v0.2.0`).
- [x] Prove private multi-frame acquisition, submission, completion, and
  retirement.
- [x] Align the same demo on Windows Vulkan.
- [x] Render the minimal deterministic multi-object scene, including a private
  real renderer CPU-preparation DAG through `slot-graph`.

Stage 1 closed in [`fluxel-rendering` `v0.8.3`](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.8.3)
at `1778226fb74d3fc0f2fdb12b6dd8da9d9d149960`; its durable
`fluxel-rendering-v0.8.3-evidence.zip` binds the cross-backend evidence.

See the [Stage 1 execution guide](https://github.com/fluxel-project/.github/blob/main/stages/stage-01-windows.md).

### Stage 2 — Web and one mini-game host

**Boundary:** port the Stage 1 scene, not a new demo. Support claims remain
specific to named browsers, backends, hosts, and devices.

- [x] 0.9 / WebGL2: close the retained scene on Windows 11 x64, Google Chrome
  Stable `153.0.8010.36`; Edge 153 is auxiliary evidence only.
- [x] 0.10 / WebGPU: close the same scene on its named Chrome/Windows/AMD
  target with completion-bounded submission, canvas reconfiguration,
  device-loss/recovery, and async disposal evidence.
- [ ] Later: close one explicitly selected mini-game host on a real device.

See the [Stage 2.1 WebGL2 execution guide](https://github.com/fluxel-project/.github/blob/main/stages/stage-02-web.md)
and [Stage 2.2 WebGPU execution guide](https://github.com/fluxel-project/.github/blob/main/stages/stage-02-webgpu.md).

`fluxel-rendering` candidate `098ee1bd5d87ef17ba2cc8ec1031636b3e4e57d3` and
`fluxel-jsbridge` candidate `db6361fbc015522bf8abf37919da45a48a7f0daa` complete
the Chrome-only WebGL2 claim.  The real Chrome run retained the lifecycle
sequence, 24 representative screenshots, 15 dense samples per visible state,
expected black/red/green/blue readbacks, clean diagnostics, and fixed-workload
startup/CPU-submission/WASM-memory measurements.  It does not claim Edge,
WebGPU, generic browser, or mini-game support.
The corresponding releases are
[`fluxel-rendering` v0.9.0](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.9.0)
and [`fluxel-jsbridge` v0.1.0](https://github.com/fluxel-project/fluxel-jsbridge/releases/tag/v0.1.0).

The 0.10 release separately closes the named WebGPU browser target:
the retained RGB scene survived resize, zero-size/restore, visibility,
controlled destroyed-device loss/recovery, and async disposal with bounded
completion tickets. The final commits are
`8d18080efb9c14cc14ef05861660f8a7ed856309` and
`65631997dbdc34c3ad2b44c91a099b507c72ead9`; the releases are
[`fluxel-rendering` v0.10.0](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.10.0)
and [`fluxel-jsbridge` v0.2.0](https://github.com/fluxel-project/fluxel-jsbridge/releases/tag/v0.2.0).
The downloaded evidence archive is 60,347 bytes with SHA-256
`94280a7ad6cccf518a1c8975326fe847f448956681a1c97de93955d72691b8fd`.
The compatible rendering-only correctness release
[`v0.10.1`](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.10.1)
at `b7b6504a8cbf9568335e9f4bbe3f7335b7a764f6` closes recovery/disposal commit
points without adding a rendering feature or changing the JS bridge revision.
Its evidence archive is 43,974 bytes with SHA-256
`7e78ba31b2df6d78d5ac80bda2584238ac78c20ec50f2741053072b8afbc72ae`.
This does not authorize a generic-browser or mini-game claim.

### Stage 3 — Resource foundation and scene API

**Boundary:** close the three distinct lifetime domains before freezing scene
ergonomics: logical assets in bases, persistent GPU residency in rendering,
and per-frame virtual/transient usage in RenderGraph.

- [x] 0.11: consolidate CI, cross-repository contracts, browser command/query
  semantics, canvas resize ownership, and proof-only public surfaces without
  adding renderer features.
- [ ] 0.12: prove a common raster/resource floor on all selected backends,
  separate storage/compute capability gates on modern backends, and one stable
  compiled graph's completion-safe physical reuse across frames.
- [ ] 0.13: prove logical typed asset identity, generation, one source-agnostic
  producer per identity, reuse, size accounting, and deterministic budget
  eviction with no GPU, I/O, or decode ownership.
- [ ] 0.14: resolve persistent residency once during preparation, import it
  into RenderGraph, and prove pending/committed generations, device recreation,
  last-use tracking, and completion-safe retirement.
- [ ] Only then validate and freeze the smallest Rust-native `Scene`, `Camera`,
  `Mesh`, `Geometry`, `Material`, and `Transform` surface across affected
  supported targets.

The 0.11 closure is published as
[`fluxel-rendering` v0.11.0](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.11.0)
and [`fluxel-jsbridge` v0.3.0](https://github.com/fluxel-project/fluxel-jsbridge/releases/tag/v0.3.0);
Host contributed CI coverage without an artificial version bump.

Stable-graph physical reuse belongs to 0.12. Cross-graph pooling, reuse across
distinct logical resources, and memory aliasing remain separate profiling-
driven gates; logical graph lifetime never implies per-frame destruction.

### Stage 4 — Windows playable runtime

**Boundary:** extend the existing scene into one playable Windows application.
Consume the proved asset/residency model and introduce platform, time, input,
loader, and runtime boundaries only when the demo proves their independent
ownership. Networking, audio, scripting, ECS, and editor work remain out of
scope.

- [ ] Close input, updates, loading, reuse, asynchronous work, pause/resume, and
  safe shutdown in the same demo.

### Stage 5 — Mobile runtime

**Boundary:** port the Stage 4 demo without creating a second mobile-only
runtime. Android and iOS are separate support claims.

- [ ] Stage 5A: close Android lifecycle, touch, surface recreation, and
  real-device evidence.
- [ ] Stage 5B: close iOS lifecycle, touch, Metal surface recreation, and
  real-device evidence.

Either substage may close independently. Scoped Canvas experiments do not wait
for both; claiming cross-mobile support requires both.

### Stage 6 — Canvas and minimal text

**Boundary:** one shared Canvas demo with images, 2D transforms, clipping,
alpha, ordering, offscreen work, and deliberately limited text. Complex shaping,
rich text, multilingual layout, and CSS remain excluded.

- [ ] Close the declared platform matrix with stable glyph and resource
  lifetimes.

### Stage 7 — UI closure

**Boundary:** one real HUD or settings page using existing runtime, Canvas,
text, assets, and input. Do not add third-party compatibility, a virtual DOM,
CSS cascade, or a general reactive framework.

- [ ] Close interaction, state updates, destruction, and resource release for
  the chosen screen.

## Working documents

- [Full roadmap](https://github.com/fluxel-project/.github/blob/main/ROADMAP.md)
- [Ecosystem architecture](https://github.com/fluxel-project/.github/blob/main/ECOSYSTEM_ARCHITECTURE.md)
- [Development principles](https://github.com/fluxel-project/.github/blob/main/DEVELOPMENT_PRINCIPLES.md)
- [Evidence policy](https://github.com/fluxel-project/.github/blob/main/EVIDENCE_POLICY.md)
- [Stage 1 Windows guide](https://github.com/fluxel-project/.github/blob/main/stages/stage-01-windows.md)
- [Stage 2 Web guide](https://github.com/fluxel-project/.github/blob/main/stages/stage-02-web.md)
- [Stage 2 WebGPU guide](https://github.com/fluxel-project/.github/blob/main/stages/stage-02-webgpu.md)

Local execution guidance is maintained above the repository group and is not
published as project documentation.
