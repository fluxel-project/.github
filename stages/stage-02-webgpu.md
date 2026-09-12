# Stage 2.2 / 0.10 — WebGPU retained-scene closure

0.10 closes one named WebGPU target for the retained Stage 1 black/red/green/
blue scene. It follows, and does not replace, the completed Stage 2.1 WebGL2
closure in [stage-02-web.md](stage-02-web.md). It is not a generic-browser,
other-browser, or mini-game claim.

**Named supported target:** Windows 11 x64, Google Chrome Stable
`153.0.8010.36` with default WebGPU on AMD Radeon 780M driver
`32.0.21028.2002`.

**Owning repository:** `fluxel-rendering` owns the renderer preparation,
portable RenderGraph declaration, wasm-private WebGPU RHI objects,
completion/recovery state, and rendering WASM capsule.

**Participating repository:** `fluxel-jsbridge` owns the supplied DOM canvas,
CSS/DPR resize observation, visibility, one RAF producer, and adapter teardown.
It starts the WASM capsule but does not request adapters/devices or hold WebGPU
objects. Dependency direction remains `fluxel-jsbridge -> fluxel-rendering`.

**Artifact under test:** the `fluxel-rendering-wasm` capsule and
`@fluxel/browser` WebGPU adapter run together on the named Chrome target.

## Boundary

- Preserve renderer-owned P*V*M, clip validation, material color, insertion
  order, and the one black-clear/store raster pass. WebGPU consumes the same
  compiled graph semantics as native and WebGL2; it does not create a web-only
  scene or graph compiler.
- Renderer exposes only the closed two-format presentation profile
  (`Rgba8Unorm` or `Bgra8Unorm`) with fixed opaque-alpha/render-attachment
  facts. RHI/browser format mapping is exhaustive and fails closed.
- The wasm-private RHI owns the canvas context, adapter, device, queue,
  pipelines, buffers, current texture/view, tickets, observers, diagnostics,
  and recovery. No GPU object or Promise becomes a public RenderGraph,
  renderer, or JS-bridge API.
- At most three submissions are live. A ticket retires only after its matching
  `queue.onSubmittedWorkDone()` settlement; RAF, resize, visibility, Promise
  creation, and callback scheduling are not completion.
- Device loss stops submission. Recovery requests a new adapter/device,
  rebuilds all device-local objects, configures only the latest non-zero canvas
  extent, and advances device generation. Resize changes canvas epoch but does
  not claim ticket completion.
- Async callbacks are generation/token scoped. Disposing cancels new work,
  prevents stale recovery from reviving RAF, awaits the terminal queue outcome,
  and reports loss as loss rather than clean completion.

## 0.10 TODO

- [x] Compile the shared retained graph against the closed WebGPU presentation
  profile without changing native or WebGL2 graph meaning.
- [x] Keep WebGPU device/canvas objects and async observer/ticket lifetime
  private to wasm RHI; bound frames in flight and report normal backpressure.
- [x] Provide same-canvas resize/zero-size/visibility behavior, controlled
  destroyed-device loss/recovery, and async idempotent disposal.
- [x] Keep the JS adapter as the only RAF producer; recovery and public
  lifecycle calls cannot create a second producer.
- [x] Run the fixed Chrome interaction/evidence procedure and inspect its
  visual, pixel, diagnostic, lifecycle, and scoped-measurement outputs.

## Required evidence and publication

The final release record must bind both repository commits, Windows, exact
Chrome version, GPU/driver, adapter facts, preferred/configured format, commands
and interaction inputs to the evidence. It must retain representative stable,
resized, and recovered screenshots; dense marker/pixel samples for each visible
state; console/page/WebGPU diagnostics; and the scoped package/startup/
CPU-submission/WASM-memory measurements. The controlled `device.destroy()`
result with `reason=destroyed` is an expected lifecycle fact; validation, OOM,
internal, and unaccounted Promise errors fail the target.

The final commits are `8d18080efb9c14cc14ef05861660f8a7ed856309` for
`fluxel-rendering` and `65631997dbdc34c3ad2b44c91a099b507c72ead9` for
`fluxel-jsbridge`. Their durable releases are
[`fluxel-rendering` v0.10.0](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.10.0)
and [`fluxel-jsbridge` v0.2.0](https://github.com/fluxel-project/fluxel-jsbridge/releases/tag/v0.2.0).
The downloaded `fluxel-rendering-v0.10.0-evidence.zip` is 60,347 bytes with
SHA-256 `94280a7ad6cccf518a1c8975326fe847f448956681a1c97de93955d72691b8fd`.
The WebGPU adapter reported AMD/RDNA3 and configured `bgra8unorm`; generation
advanced from 1 to 2 with expected loss reason `destroyed`, followed by a
terminal `Disposed` state and clean structured diagnostics.

## Non-goals and closure boundary

This stage does not add generic WebGPU/browser support, a public WebGPU API,
multiple queues, compute/storage/material breadth, JS scene authority, Host
runtime/input, or a mini-game host. Its success does not alter the completed
WebGL2 claim. Stage 2 remains open until a separately selected real mini-game
host has its own target-specific evidence.
