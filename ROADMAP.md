# Fluxel Roadmap

This is the single source of truth for Fluxel plan order and milestone authorization. Repository documents define public contracts and implementation details; READMEs summarize current status and do not create a competing plan.

The independent [`fluxel-rhi`](https://github.com/fluxel-project/fluxel-rhi) is a stable maintenance baseline. Its authority is its [RHI architecture](https://github.com/fluxel-project/fluxel-rhi/blob/main/crates/rhi/DESIGN.md), [core API and SPI design](https://github.com/fluxel-project/fluxel-rhi/blob/main/crates/core/DESIGN.md), [verification record](https://github.com/fluxel-project/fluxel-rhi/blob/main/documents/verification.md), and [open TODOs](https://github.com/fluxel-project/fluxel-rhi/blob/main/TODO.md). It owns portable resources, command recording, submission, completion, presentation, device loss, and private backend realization. It changes only when an implemented consumer supplies a bounded requirement and an integration proof. Its open TODOs and target-evidence limits, including the current Metal evidence boundary, remain open facts; they do not block work that does not require them.

## Delivery order

```text
Foundation: Fluxel Bases asset-core reuse
  + Renderer-internal RenderGraph foundation and stable RHI integration
  -> Material semantics and independent Shader crate
  -> Material/data-driven Renderer MVP with one external Rust RenderFeature
  -> Broader material system, Forward+/Deferred, and RenderFeatures
  -> Performance and resource strategy
```

Every milestone closes only with implementation, structured refusal behavior, focused tests, relevant backend evidence, and an end-to-end ecosystem proof. A green library test cannot close a cross-repository milestone.

Milestones 1 and 2 form one foundation closure for the first renderer vertical
slice. After the integration owner has selected the exact Bases and RHI
revisions, recorded the lock inputs, and assigned the consumer artifact, their
independent work packages may proceed in parallel: Bases reuse or its bounded
extension on one side, and the renderer-internal Graph migration and direct RHI
execution on the other. This does not permit either milestone's gate to be
skipped, nor does it authorize Material, Shader, or Renderer MVP interfaces
before their later milestones.

## 1. Fluxel Bases asset core

Owner: `fluxel-bases`.

Reuse the existing logical-asset core and extend it only where the rendering
vertical slice proves a bounded need: typed identity, content generation,
handles, CPU-side state, reuse, and budgets. Assets remain independent from GPU
residency and source I/O policy. Loading orchestration, source reads, decoding,
retry, and cancellation retain their existing owner boundaries.

The gate is a renderer consumer that observes stable logical identity and generation change without treating an asset handle as permission to retire a GPU realization.

## 2. Renderer-internal RenderGraph and stable RHI integration

Owner: `fluxel-rendering`, consuming the stable `fluxel-rhi` contract.

Migrate the existing standalone RenderGraph into `fluxel-renderer` as an internal module. Its public role is not preserved as an independent crate. The module owns pass and resource declarations, logical versions, dependencies, definedness, validation, scheduling, transient lifetime analysis, and lowering to RHI recording/submission.

The foundation may use direct RHI fixtures and a small fixed shader or other
minimal executable work to prove graph execution. It does not wait for the
MaterialGraph, material variants, or the independent Shader crate. Conversely,
that minimal graph proof does not establish a material-driven frame pipeline.

The first integration gate uses an exact RHI revision, caches a compiled plan, submits two or three frames before waiting, retains distinct completion points, and retires each recycled resource only after the completion covering its last use. The current renderer still pins RHI revision `97a003fb11e30bd97ec62efe82e031f7bcf26ab5` as version `0.16.0`; the stable local RHI baseline is revision `e39c0e50406d764e6f4fb53950f7c8bc00469d04` (workspace `0.1.0`, WebGPU crate `0.1.1`). A migration plan must name its selected revision and real integration artifact before changing that pin. Existing standalone `fluxel-rendergraph`, `fluxel-rendering-wasm`, and old RHI dependency edges are migration state, not completed architecture.

## 3. Material semantics and independent Shader crate

Owner: `fluxel-rendering`.

Establish `fluxel-material` and `fluxel-shader` as separate crates. `fluxel-material` owns `MaterialGraph`, typed material values, validation, Material IR, parameter/resource schema, material assets and instances, and graph-to-IR lowering. `fluxel-shader` owns reusable shader composition, reflection, variants, `ShaderArtifact`, `ShaderInterface`, pipeline requirements, target profiles, and cache identity.

Material describes semantic requirements and does not own renderer behavior. Shader compilation is reusable by material, compute, particles, and post processing; it does not own MaterialGraph. The proof compiles one Rust-defined material through Material IR and shader composition into an artifact, interface, and compiled material variant that renderer preparation consumes.

## 4. Material/data-driven Renderer MVP

Owner: `fluxel-rendering`.

Define the smallest `RenderScene`, `RenderObject`, `RenderView`, camera, mesh, draw-list, material-selection, and `FramePipeline` vocabulary. `FramePipeline` is an internal renderer SPI, not a crate. One built-in Forward path proves the normal scene-to-graph-to-RHI route.

The MVP is developed and reviewed through these internal checkpoints:

1. `RenderScene` accepts typed batch updates, uses stable generational render
   identities, and creates a frame snapshot before frame construction.
2. The internal `FramePipeline` SPI defines phases and receives the renderer
   services needed for visibility, sorting, material-variant resolution, and
   shader/pipeline resolution. A pipeline owns frame policy: it selects phases,
   draw preparation, and graph topology for a scene and view.
3. Rust extension seams allow an external `RenderFeature` to own typed batch
   data and contribute to declared phases. Features that affect shading use
   controlled material/shader extension seams; a Rust crate may also provide a
   pipeline through the established internal SPI boundary. A feature contributes
   content and declared work, while the selected pipeline retains phase policy.
   All feature resource accesses and passes remain under RenderGraph authority;
   a feature never submits independently.
4. A minimal Forward pipeline renders the in-tree mesh path with a prepared
   textured material. An independently compiled Rust feature then proves the
   same scene, material, shader, pipeline, graph, and RHI path before these
   extension contracts are stabilized.

The MVP includes an in-tree mesh path and one separate Rust crate implementing a minimal external `RenderFeature`. The external feature must accept batch incremental data, participate in a pipeline phase, and declare its GPU resource accesses through Graph without a core object-type switch or separate submissions. It need not implement a full terrain, water, or particle system. The gate proves that this extension uses the same material, shader, renderer, internal graph, and RHI contracts as the built-in mesh path. Include basic visibility, deterministic phase sorting/batching, stable generational render identities, and a frame snapshot boundary; ECS-style and ordinary adapters use the same batch boundary without introducing an ECS.

## 5. Broader renderer and RenderFeatures

Owner: `fluxel-rendering`.

Expand the MVP's culling, ordering, batching, material/shader selection, diagnostics, and retained scene updates against real workloads. Broaden MaterialGraph and material variants only where an implemented feature needs them. Forward+ and Deferred are independent internal `FramePipeline` implementations sharing the same material, shader, graph, and RHI contracts. Terrain, water, particles, sky, vegetation, and similar algorithms enter as RenderFeatures when a bounded consumer justifies them. This work consumes the extension seams verified in milestone 4; each pipeline explicitly rejects an unavailable phase or incompatible shader, binding, or capability requirement. Rust crate plugins remain the supported extension form. Dynamic DLL loading and a stable plugin binary ABI are unscheduled.

## 6. Performance and resource strategy

Owner: `fluxel-rendering`, with RHI changes only for proven consumer needs.

Develop GPU residency, upload policy, streaming coordination, LOD, GPU culling, indirect draws, batching, async compute, and memory budgeting from representative workloads and profiles. Every reuse, aliasing, or allocation claim needs a measured workload and completion-safe lifetime evidence.

## Deferred and undecided scope

Capture/replay, Canvas 2D and basic text, declarative UI, Blender tooling, and
JavaScript encapsulation are deferred and undecided. They have no approved
order, owner, architecture, API, dependency, or acceptance plan.

## Cross-cutting requirements

- Use precise pinned revisions for cross-repository integration, and retain the manifest that names every participating revision and lockfile.
- Resources are owned by device/context identity plus generation. A resource from another device or generation fails with a structured library error before a backend or driver is asked to use it.
- Browser and mini-game adapters own platform lifecycle adaptation only: canvas or surface setup, resize, foreground/background, and loss/restore. Backend-private GPU objects never enter the public rendering object model.
- Unsupported requirements fail explicitly. Do not silently approximate them.
- RHI maintenance work follows real renderer/graph consumers; it is not a parallel feature backlog.
