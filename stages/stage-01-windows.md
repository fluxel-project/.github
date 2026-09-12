# Stage 1 — Windows visible renderer

Build one durable Windows scene before broadening the runtime. This stage
extends the Stage 0 baseline; it does not establish asset or loader contracts,
a general platform API, or an application runtime.

**Owning repository:** `fluxel-rendering`.

**Participating repositories:** `fluxel-host` supplies the minimum Win32
`Window` primitive used by this proof. The temporary executable remains a
`fluxel-rendering` artifact; this is not an early general Host runtime.

**Artifact under test:** a temporary Windows executable owned by
`fluxel-rendering` that presents the Stage 1 scene through DX12 and Vulkan.

**Cross-repository contract changes:** Host owns fixed-size window creation,
HWND lifetime, message pumping, close observation, and standard
window/display-handle traits. Renderer/RHI consumes only those standard traits,
so rendering does not depend on Host. RHI alone owns native surface,
swapchain, presentation, and GPU synchronization. This slice excludes input,
clock, DPI and resize policy, fullscreen, and a general Host runtime.

Use the same demo throughout the stage. Complete a closure before starting the
next one. A lifetime, validation, or public-contract defect blocks closure.
Global artifact, diagnostic, and performance requirements are defined in
[the evidence policy](../EVIDENCE_POLICY.md).

## 1.1 — DX12 first visible image

### Boundary

- [x] In `fluxel-host`, create the minimum fixed-size Win32 `Window`: creation
  and destruction, message pump, close request, and standard window/display
  handles. Do not add Input, Clock, or a general platform API.
- [x] In `fluxel-rendering`, use only those standard handles to create the
  minimum DX12 surface or swapchain path needed by the demo.
- [x] Acquire an image, clear it, draw a fixed triangle, present it, and handle
  the close event.
- [x] Wait for required GPU work and release resources in a valid shutdown
  order.
- [x] Keep the reusable window primitive in Host and the proof executable in
  rendering. Do not create a general platform API from this one path.
- [x] Keep HWND, DXGI, queues, fences, descriptors, and all other native types
  inside the RHI native boundary. Renderer and RenderGraph public APIs must not
  gain DX12 variants.

### Focused checks

- [x] Add focused tests for newly introduced platform-independent state and
  error paths.
- [x] Check normal creation, present, close, GPU completion, and repeated
  startup/shutdown.
- [x] Confirm that a presentable triangle, rather than an offscreen or
  clear-only result, is the demonstrated path.

### Real-target proof

- [x] Retain and inspect a Windows DX12 capture of the presented triangle. For
  continuous frames, retain multiple time-separated captures plus a dense
  frame sample, readback sequence, or recording sufficient to expose an
  alternating clear/flicker defect.
- [x] Retain adapter, driver, backend, validation, and clean-shutdown
  diagnostics according to the evidence policy.

### Stop / close condition

- [x] Close only when the triangle is visibly presented and the window exits
  without unexplained validation errors. A clear-only window, offscreen image,
  or unpresented triangle does not close this substage.

Closed by [`fluxel-rendering` `v0.8.0`](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.8.0)
at `800b390`, using [`fluxel-host` `v0.1.0`](https://github.com/fluxel-project/fluxel-host/releases/tag/v0.1.0)
at `e02b736`. The release artifact records the AMD Radeon 780M / driver
`32.0.21028.2002`, required validation with no diagnostics, 899 presented
frames, and 120 dense client-area captures over about 6.86 seconds. Reviewers
opened the first, middle, and final full-window samples plus a content crop;
all showed the same black background and blue triangle without blank,
alternating-clear, residual, or occlusion artifacts, followed by clean close.

## 1.2 — Surface lifecycle

### Boundary

- [x] Model resize, zero width or height, minimization, restoration, and close
  as distinct host events.
- [x] Do not acquire or submit a drawable frame while size is invalid.
- [x] Recreate required surface resources after a valid size returns.
- [x] Assign each recreated surface state a generation. Recreation stops new
      acquisition from the old generation but does not make its presentable
      resources releasable; retire that generation only after every GPU
      reference has reached known completion.
- [x] Define which resources survive resize and which retire before
  replacement; never overwrite or destroy GPU-in-use swapchain state.
- [x] Preserve structured, diagnosable device, surface, acquisition, and
  presentation failures. Do not silence known validation noise.

### Focused checks

- [x] Exercise resize, zero-size, minimize, restore, and exit as separate
  cases.
- [x] Exercise recreation after prior work is still completing.
- [x] Assert that invalid dimensions cause no drawable submission.

### Real-target proof

- [x] Run a repeatable DX12 resize, minimize, restore, and exit sequence on a
  real Windows target.
- [x] Retain lifecycle diagnostics and record any accepted validation
  diagnostic under the evidence policy.

### Stop / close condition

- [x] Close only when the sequence completes without unexplained validation
  errors or submission at invalid size.

Closed by [`fluxel-rendering` `v0.8.1`](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.8.1)
at `97558a8`, using [`fluxel-host` `v0.2.0`](https://github.com/fluxel-project/fluxel-host/releases/tag/v0.2.0)
at `ae4bda3`. The release artifact records AMD Radeon 780M / driver
`32.0.21028.2002`, surface generations 1–4 over logical extents 960x540,
800x600, suspended zero-size, restored 800x600, and 1100x620, plus 5161
presented frames and 191 suspended iterations. Its 120 timestamped sampling
points retain both desktop and exact-window images (240 PNG); reviewers opened
the first, middle, final, and lifecycle-boundary samples and observed the same
black clear and blue triangle without residual, alternating, or flickering
frames. Validation, close, and shutdown were clean.

## 1.3 — Private bounded frames-in-flight lifetime

### Boundary

- [x] Validate a bounded maximum `N` frames in flight, with `N=3` as the Windows
      demo default, while keeping the count, every ring slot, fence value, and
      native synchronization object private.
- [x] Limit public semantics to frame acquisition, frame submission,
  completion observation, and retirement.
- [x] Associate transient allocations, uploads, descriptors, and reused frame
  resources with the completion that makes reuse safe.
- [x] Prevent the CPU from overwriting data referenced by unfinished GPU work.
- [x] When the GPU falls behind, wait or throttle at a documented boundary.
  Do not grow allocations without limit, silently collapse to one frame, or
  depend on driver serialization.

### Focused checks

- [x] Test normal progress, delayed completion, back pressure, shutdown with
  outstanding work, and relevant failure paths.
- [x] Verify the portable lifecycle state machine without exposing DX12
  synchronization mechanics.
- [x] Verify reuse is deferred until the completion associated with that reuse
      is observed.
- [x] Artificially delay GPU completion, submit until all `N` slots are live,
      assert the next frame is constrained by back pressure, then observe one
      completion and prove only its corresponding slot becomes reusable.

### Real-target proof

- [x] Run DX12 continuously with at least three reusable frame contexts cycling
  under induced or observed GPU lag.
- [x] Retain frame-lifetime diagnostics showing completion, retirement, and
  back-pressure behavior.

### Stop / close condition

- [x] Close only when no in-use frame resource is overwritten and the public
  contract remains valid if the private in-flight count changes later.

## 1.4 — Vulkan alignment on Windows

**Closed by:** [`fluxel-rendering` `v0.8.3`](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.8.3)
at `1778226fb74d3fc0f2fdb12b6dd8da9d9d149960`; the durable
`fluxel-rendering-v0.8.3-evidence.zip` Release attachment binds this closure's
cross-backend evidence and diagnostics.

### Boundary

- [x] Run the same executable-level demo scene through Vulkan on Windows.
- [x] Keep equivalent shader inputs, rendering semantics, and expected visual
  output; backend setup and compiled binaries may differ.
- [x] Apply the DX12 lifecycle contract for resize, minimization, restoration,
  acquisition, submission, completion, retirement, and shutdown.
- [x] Fix portable contract defects above a backend. Keep Vulkan layouts,
  synchronization, queues, and surface objects within RHI.
- [x] Do not weaken DX12 or publish a lowest-common-denominator multi-queue API
  merely for textual symmetry.

### Focused checks

- [x] Run the Stage 1.2 lifecycle cases and Stage 1.3 lifetime cases through
  the shared portable contract.
- [x] Check structured backend-specific failures without leaking Vulkan native
  objects across the RHI boundary.

### Real-target proof

- [x] Retain DX12 and Vulkan output from the same commit and scene.
- [x] Enable backend validation and record expected backend-specific diagnostic
  differences under the evidence policy.

### Stop / close condition

- [x] Close only when both Windows backends meet the same visible-scene and
  lifecycle contract without unexplained validation errors.

## 1.5 — Minimal multi-object scene

**Closed by:** [`fluxel-rendering` `v0.8.3`](https://github.com/fluxel-project/fluxel-rendering/releases/tag/v0.8.3)
at `1778226fb74d3fc0f2fdb12b6dd8da9d9d149960`; the durable
`fluxel-rendering-v0.8.3-evidence.zip` Release attachment binds this closure's
cross-backend scene, visual, diagnostic, and lifetime evidence.

### Boundary

- [x] Replace the fixed-triangle-only path with one camera, multiple meshes,
  multiple materials, and an independent transform for each object.
- [x] Define deterministic draw order for identical input.
- [x] Reuse compatible pipelines and bindings, without promising general
      batching, instancing, visibility, or optimization APIs.
- [x] Adopt `slot-graph` only if the completed scene naturally exposes at least
      one real CPU-preparation dependency DAG. Do not create artificial tasks
      to justify the dependency; keep its types behind renderer-owned prepared
      data and outside GPU synchronization.
- [x] Use embedded or generated resources. Loader behavior and durable asset
  identity remain out of scope until the runtime stage.
- [x] Keep ordinary scene construction above RHI and RenderGraph. A mesh,
  material, transform, or camera must not require backend handles.

### Focused checks

- [x] Check deterministic ordering and independent transform updates.
- [x] Check compatible pipeline and binding reuse at the intended boundary.
- [x] Check invalid scene or submission inputs return structured errors.

### Real-target proof

- [x] Run the same multi-object scene continuously on DX12 and Vulkan.
- [x] Retain expected and actual images or recordings using the comparison
  oracle named by the stage plan.
- [x] Retain frame-lifetime diagnostics and the first representative CPU
  submission and frame-time baseline.

### Stop / close condition

- [x] Close Stage 1 only when both backends run the same scene, survive the
  lifecycle sequence, and shut down cleanly without unexplained validation
  errors.
- [x] Do not add assets, loaders, a general platform abstraction, batching,
  instancing, or additional scene systems to make this stage appear complete.
