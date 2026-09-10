# Fluxel Ecosystem

Fluxel is a native-first, Three.js-informed, AI-friendly lightweight rendering
runtime. Rust and native runtime behavior are authoritative. Development follows
visible demo closure rather than completing architectural layers in advance.

> **Current target:** close the reproducible headless baseline, then present the
> first DX12 image in a Windows window.

## What Fluxel is

- A focused Rust rendering runtime built from libraries with explicit ownership.
- A portable design proven by real DX12, Vulkan, WebGL2, WebGPU, and platform
  evidence as those targets enter the roadmap.
- A Three-like usability experiment where concepts are adopted only when they
  remain natural, typed Rust APIs.
- A demo-driven project: the next visible closure decides what must exist next.

## What Fluxel is not

- Not a Three.js backend, a 100% Three.js compatibility layer, or
  `threejs-native`.
- Not a Vue, DOM, or CSS compatibility layer, and not `vue-native`.
- Not a plan to create every named library before a running demo needs it.
- Not a place where compile, mock, or `TestRhi` results are presented as
  real-device correctness.

## Layer map

Layer ownership determines where code belongs. It does not determine development
order. Names that do not yet have a demonstrated implementation need are
candidate boundaries, not instructions to create repositories.

- **Foundation and host:** base, time, filesystem, storage, networking, input,
  image, audio, and platform lifecycle are candidate boundaries extracted only
  from a staged demo with an end-to-end consumer.
- **Resources:** `fluxel-loader` obtains and decodes CPU data;
  `fluxel-assets` owns durable identity, typed handles, generations, residency,
  reuse, and safe release.
- **GPU:** `fluxel-rhi` owns native backend realization;
  `fluxel-rendergraph` owns portable single-frame declarations, plans, and
  validation; `fluxel-renderer` owns 3D submission, ordering, and lowering.
  Canvas and shader tooling remain candidate boundaries.
- **Runtime and higher layers:** runtime composition, ABI packaging, JavaScript
  VM and bridge layers, a JS API, and purpose-built declarative UI appear only
  after a demonstrated application or embedding need.

## Development stages

Every stage defines a boundary, a TODO list, and a completion gate in the
[full roadmap](https://github.com/fluxel-project/.github/blob/main/ROADMAP.md).
Descriptions become intentionally coarser as distance from the current target
increases; that coarseness is not permission to exceed the stated boundary.

### Stage 0 — Reproducible baseline

**Boundary:** close the current headless slice. Do not add windows, scene APIs,
new fixed recipes, assets, loaders, or runtime composition.

- [ ] Unify workspace release and Git revision rules.
- [ ] Preserve structured renderer errors.
- [ ] Make CPU, CI, and real-GPU conformance evidence reproducible.
- [ ] Audit RenderGraph public exports.
- [ ] Freeze new `draw_*` recipes and upload-state types.

### Stage 1 — Windows visible renderer

**Boundary:** one persistent Windows scene on DX12 and Vulkan. Keep native types
inside RHI; do not introduce loaders, durable assets, or a public triple-buffer
API.

- [ ] Present the first DX12 triangle.
- [ ] Close resize, minimize, restore, and surface recreation.
- [ ] Prove private multi-frame acquisition, submission, completion, and
  retirement.
- [ ] Align the same demo on Windows Vulkan.
- [ ] Render the minimal deterministic multi-object scene.

See the [Stage 1 execution guide](https://github.com/fluxel-project/.github/blob/main/stages/stage-01-windows.md).

### Stage 2 — Web and one mini-game host

**Boundary:** port the Stage 1 scene, not a new demo. Support claims remain
specific to named browsers, backends, hosts, and devices.

- [ ] Close WebGL2.
- [ ] Close WebGPU.
- [ ] Close one explicitly selected mini-game host on a real device.

### Stage 3 — Three-like API validation

**Boundary:** validate a small Rust-native `Scene`, `Camera`, `Mesh`,
`Geometry`, `Material`, and `Transform` surface. Do not add Three.js
inheritance, plugins, material breadth, or compatibility types.

- [ ] Stage 3A: probe ergonomics on DX12 and WebGL2.
- [ ] Stage 3B: before public API freeze, validate all affected supported
  targets.

### Stage 4 — Windows playable runtime

**Boundary:** extend the existing scene into one playable Windows application.
Introduce platform, time, input, loader, assets, and runtime boundaries only
when the demo proves their independent ownership. Networking, audio, scripting,
ECS, and editor work remain out of scope.

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
text, assets, and input. Do not add Vue compatibility, a virtual DOM, CSS
cascade, or a general reactive framework.

- [ ] Close interaction, state updates, destruction, and resource release for
  the chosen screen.

## Working documents

- [Full roadmap](https://github.com/fluxel-project/.github/blob/main/ROADMAP.md)
- [Development principles](https://github.com/fluxel-project/.github/blob/main/DEVELOPMENT_PRINCIPLES.md)
- [Evidence policy](https://github.com/fluxel-project/.github/blob/main/EVIDENCE_POLICY.md)
- [Stage 1 Windows guide](https://github.com/fluxel-project/.github/blob/main/stages/stage-01-windows.md)

Project-local `AGENTS.md` files define the current repository's ownership,
modification locations, focused verification commands, and real-target gates.
