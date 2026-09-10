# Stage 1 — Windows visible renderer

Build one durable Windows scene before broadening the runtime. This stage
extends the Stage 0 baseline; it does not establish `fluxel-assets`,
`fluxel-loader`, a general `fluxel-platform` API, or `fluxel-runtime`.

Use the same demo throughout the stage. Complete a closure before starting the
next one. A lifetime, validation, or public-contract defect blocks closure.
Global artifact, diagnostic, and performance requirements are defined in
[the evidence policy](../EVIDENCE_POLICY.md).

## 1.1 — DX12 first visible image

### Boundary

- [ ] Create one Windows window and the minimum DX12 surface or swapchain path
  needed by the demo.
- [ ] Acquire an image, clear it, draw a fixed triangle, present it, and handle
  the close event.
- [ ] Wait for required GPU work and release resources in a valid shutdown
  order.
- [ ] Keep window integration in the demo or a narrow host boundary. Do not
  create a general platform API from this one path.
- [ ] Keep HWND, DXGI, queues, fences, descriptors, and all other native types
  inside the RHI native boundary. Renderer and RenderGraph public APIs must not
  gain DX12 variants.

### Focused checks

- [ ] Add focused tests for newly introduced platform-independent state and
  error paths.
- [ ] Check normal creation, present, close, GPU completion, and repeated
  startup/shutdown.
- [ ] Confirm that a presentable triangle, rather than an offscreen or
  clear-only result, is the demonstrated path.

### Real-target proof

- [ ] Retain a Windows DX12 capture of the presented triangle.
- [ ] Retain adapter, driver, backend, validation, and clean-shutdown
  diagnostics according to the evidence policy.

### Stop / close condition

- [ ] Close only when the triangle is visibly presented and the window exits
  without unexplained validation errors. A clear-only window, offscreen image,
  or unpresented triangle does not close this substage.

## 1.2 — Surface lifecycle

### Boundary

- [ ] Model resize, zero width or height, minimization, restoration, and close
  as distinct host events.
- [ ] Do not acquire or submit a drawable frame while size is invalid.
- [ ] Recreate required surface resources after a valid size returns.
- [ ] Define which resources survive resize and which retire before
  replacement; never overwrite or destroy GPU-in-use swapchain state.
- [ ] Preserve structured, diagnosable device, surface, acquisition, and
  presentation failures. Do not silence known validation noise.

### Focused checks

- [ ] Exercise resize, zero-size, minimize, restore, and exit as separate
  cases.
- [ ] Exercise recreation after prior work is still completing.
- [ ] Assert that invalid dimensions cause no drawable submission.

### Real-target proof

- [ ] Run a repeatable DX12 resize, minimize, restore, and exit sequence on a
  real Windows target.
- [ ] Retain lifecycle diagnostics and record any accepted validation
  diagnostic under the evidence policy.

### Stop / close condition

- [ ] Close only when the sequence completes without unexplained validation
  errors or submission at invalid size.

## 1.3 — Private multi-frame lifetime

### Boundary

- [ ] Validate three frames in flight by default, while keeping this count and
  every ring slot, fence value, and native synchronization object private.
- [ ] Limit public semantics to frame acquisition, frame submission,
  completion observation, and retirement.
- [ ] Associate transient allocations, uploads, descriptors, and reused frame
  resources with the completion that makes reuse safe.
- [ ] Prevent the CPU from overwriting data referenced by unfinished GPU work.
- [ ] When the GPU falls behind, wait or throttle at a documented boundary.
  Do not grow allocations without limit, silently collapse to one frame, or
  depend on driver serialization.

### Focused checks

- [ ] Test normal progress, delayed completion, back pressure, shutdown with
  outstanding work, and relevant failure paths.
- [ ] Verify the portable lifecycle state machine without exposing DX12
  synchronization mechanics.
- [ ] Verify reuse is deferred until the completion associated with that reuse
  is observed.

### Real-target proof

- [ ] Run DX12 continuously with at least three reusable frame contexts cycling
  under induced or observed GPU lag.
- [ ] Retain frame-lifetime diagnostics showing completion, retirement, and
  back-pressure behavior.

### Stop / close condition

- [ ] Close only when no in-use frame resource is overwritten and the public
  contract remains valid if the private in-flight count changes later.

## 1.4 — Vulkan alignment on Windows

### Boundary

- [ ] Run the same executable-level demo scene through Vulkan on Windows.
- [ ] Keep equivalent shader inputs, rendering semantics, and expected visual
  output; backend setup and compiled binaries may differ.
- [ ] Apply the DX12 lifecycle contract for resize, minimization, restoration,
  acquisition, submission, completion, retirement, and shutdown.
- [ ] Fix portable contract defects above a backend. Keep Vulkan layouts,
  synchronization, queues, and surface objects within RHI.
- [ ] Do not weaken DX12 or publish a lowest-common-denominator multi-queue API
  merely for textual symmetry.

### Focused checks

- [ ] Run the Stage 1.2 lifecycle cases and Stage 1.3 lifetime cases through
  the shared portable contract.
- [ ] Check structured backend-specific failures without leaking Vulkan native
  objects across the RHI boundary.

### Real-target proof

- [ ] Retain DX12 and Vulkan output from the same commit and scene.
- [ ] Enable backend validation and record expected backend-specific diagnostic
  differences under the evidence policy.

### Stop / close condition

- [ ] Close only when both Windows backends meet the same visible-scene and
  lifecycle contract without unexplained validation errors.

## 1.5 — Minimal multi-object scene

### Boundary

- [ ] Replace the fixed-triangle-only path with one camera, multiple meshes,
  multiple materials, and an independent transform for each object.
- [ ] Define deterministic draw order for identical input.
- [ ] Reuse compatible pipelines and bindings, without promising general
  batching, instancing, visibility, or optimization APIs.
- [ ] Use embedded or generated resources. Loader behavior and durable asset
  identity remain out of scope until the runtime stage.
- [ ] Keep ordinary scene construction above RHI and RenderGraph. A mesh,
  material, transform, or camera must not require backend handles.

### Focused checks

- [ ] Check deterministic ordering and independent transform updates.
- [ ] Check compatible pipeline and binding reuse at the intended boundary.
- [ ] Check invalid scene or submission inputs return structured errors.

### Real-target proof

- [ ] Run the same multi-object scene continuously on DX12 and Vulkan.
- [ ] Retain expected and actual images or recordings using the comparison
  oracle named by the stage plan.
- [ ] Retain frame-lifetime diagnostics and the first representative CPU
  submission and frame-time baseline.

### Stop / close condition

- [ ] Close Stage 1 only when both backends run the same scene, survive the
  lifecycle sequence, and shut down cleanly without unexplained validation
  errors.
- [ ] Do not add assets, loaders, a general platform abstraction, batching,
  instancing, or additional scene systems to make this stage appear complete.
