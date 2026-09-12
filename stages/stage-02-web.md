# Stage 2.1 / 0.9 — WebGL2 retained-scene closure

0.9 ports the retained Stage 1 deterministic red/green/blue multi-object scene
to one real WebGL2 target. It is a single WebGL2 closure inside Stage 2, not a
claim of generic browser, WebGPU, or mini-game support.

**Named supported target:** Windows 11 x64 with Google Chrome Stable
`153.0.8010.36`, WebGL2. Microsoft Edge 153 may be run as auxiliary
compatibility evidence, but does not extend the support claim. WebGPU and one
real mini-game host remain later, separately named Stage 2 gates.

**Owning repository:** `fluxel-rendering` owns the WASM rendering package,
portable RenderGraph semantics, rendering lifetime, and the retained scene.

**Participating repository:** `fluxel-jsbridge` owns the browser adapter, DOM
canvas selection/events, and the minimal JavaScript-facing bootstrap. Its
dependency direction is one-way: `fluxel-jsbridge` starts and drives
`fluxel-rendering` WASM; rendering does not import JS bridge, browser, or host
types. WASM only borrows the explicitly supplied canvas; its private
`fluxel-rhi` WASM browser implementation creates and owns the WebGL execution
objects derived from that canvas.

**Artifact under test:** a `fluxel-rendering` WASM package and a
`fluxel-jsbridge` browser integration running the retained scene in the named
Chrome target.

## Boundary

- Preserve the Stage 1 expected scene, deterministic order, imported
  presentable-resource terminology, and renderer-owned resource lifetime.
- Implement only the bridge needed to select/bind the DOM canvas, start,
  resize, submit the retained scene, observe visibility/context transitions,
  recover the required bindings, and report structured diagnostics.
- Keep DOM canvas selection/events and browser platform adaptation in the JS
  bridge adapter. The explicitly supplied canvas is borrowed by rendering;
  private `fluxel-rhi` WASM code creates and owns WebGL execution objects. JS
  does not define scene, RenderGraph, RHI, GPU synchronization, or lifecycle
  semantics.
- Keep the existing completion-driven retirement and private bounded
  frames-in-flight contract. A browser callback, context event, or resize is
  not evidence that GPU work has completed.
- Treat context loss and restoration as explicit normal lifecycle outcomes
  with resource-generation rebinding; do not silently recreate resources or
  accept stale output. Actual RHI operation failures remain structured
  diagnostics; expected loss/backpressure/blocked states are not errors.

## 0.9 TODO

- [x] Build the narrow `fluxel-jsbridge -> fluxel-rendering` WASM startup,
  canvas binding, resize, scene submission, and structured-diagnostic path.
- [x] Render the retained deterministic multi-object scene through WebGL2.
- [x] Handle visibility and WebGL context loss/restoration with correct
  resource rebinding and no stale frame/resource reuse.
- [x] Measure and retain scoped package size, startup, CPU submission, and
  memory baselines for the fixed workload and named Chrome environment.
- [x] Produce the named-target visual, lifecycle, diagnostic, and performance
  evidence; inspect it before declaring the target supported.

## Required checks and evidence

- Run focused Rust/WASM and JS bridge tests, formatting, Clippy/lint, and
  documentation checks for changed modules. Compile, mock, and WASM-only checks
  prove logic only, never WebGL correctness.
- Run the fixed interaction script on the named Chrome target: startup, stable
  rendering, resize, hide/show or equivalent visibility sequence, forced or
  controlled context loss/restoration where the target permits it, and clean
  teardown.
- Retain exact commit SHA, commands/host procedure, Windows and Chrome version,
  WebGL renderer/driver and validation configuration, fixed input, expected and
  actual output, diagnostics, and all focused-test results.
- Inspect multiple time-separated screenshots for continuous or lifecycle
  paths, together with at least fifteen dense frame-marker/readback samples per
  visible state or a recording.
  Review clear colour, red/green/blue geometry, dimensions, stale contents,
  tearing, alternating frames, and flicker. Stable frames use an exact image
  hash; otherwise freeze tolerance, mask, and review procedure before a pass.
- Retain the workload, sampling method, duration, raw package/startup/
  CPU-submission/memory measurements, and declared regression thresholds.
- If Edge 153 is run, retain it as explicitly auxiliary evidence and report it
  separately. It cannot change the Chrome-only 0.9 support claim.

## Non-goals

- WebGPU, generic browser support, and any mini-game host or device.
- A generic web global-state API, a general Host runtime, input framework,
  assets/loader, or public multi-queue API.
- Third-party scene compatibility, an external-engine backend, a JavaScript scene authority,
  or widening the Rust/native public contract for a hypothetical web target.
- Extracting a crate or adapter framework before this vertical slice proves an
  independently changing responsibility.

## Stop / close condition

Close 0.9 only when the named Chrome target runs the retained scene through
WebGL2, survives the declared resize/visibility/context-recovery sequence,
reports clean or explicitly accepted diagnostics, and has retained visual,
lifecycle, and scoped performance evidence. Missing real-target evidence keeps
the Chrome support claim open. Completion of 0.9 does not close Stage 2, does
not authorize WebGPU, and does not choose or close the mini-game host gate.

## 0.9 closure record

`fluxel-rendering` candidate `098ee1bd5d87ef17ba2cc8ec1031636b3e4e57d3` and
`fluxel-jsbridge` candidate `db6361fbc015522bf8abf37919da45a48a7f0daa` were
jointly exercised on the named Chrome target.  The retained evidence manifest
binds both repository SHAs, Chrome `153.0.8010.36`, Windows, the AMD Radeon
780M WebGL2 renderer, exact wasm-bindgen/browser commands, all lifecycle input,
and diagnostics.

The run passed eight states: stable, resized, zero-size, restored, hidden,
visible, context-lost, and context-restored.  It retained three screenshots for
each state (24 total), and each visible state additionally passed fifteen dense
frame-marker/readback samples.  All visible readbacks matched black clear
`[0,0,0,255]`, red `[255,0,0,255]`, green `[0,255,0,255]`, and blue
`[0,89,255,255]`; frame markers strictly increased, diagnostics were clean,
WASM memory remained 1,179,648 bytes, and CPU submission was at most 1 ms for
the fixed workload.  Review opened representative stable, resized, and
context-restored frames to confirm the visual oracle rather than relying on
logs alone.

The rendering candidate also passed its Windows AMD Radeon 780M real-GPU DX12
and Vulkan conformance gate, 83 passed / 0 failed across the renderer and RHI
test binaries.  That gate supports retained
renderer correctness but is not substituted for the Chrome evidence above.
This record closes only Stage 2.1 / 0.9; it leaves WebGPU and the explicitly
selected real mini-game-host closure unchecked.

Durable publication:

- [`fluxel-rendering` v0.9.0](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.9.0)
  contains the evidence archive and checksums.  The archive is 110,953 bytes
  with SHA-256
  `de4ad88e2cfb7e5641ea6cc124295c0acbd4a9dd2f76e98b575cdb89b686c9d0`.
- [`fluxel-jsbridge` v0.1.0](https://github.com/fluxel-project/fluxel-jsbridge/releases/tag/v0.1.0)
  publishes the browser lifecycle adapter proven by that evidence.
