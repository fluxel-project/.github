# Fluxel Roadmap

This is the single source of truth for Fluxel plan order and milestone authorization. Repository documents define public contracts and implementation details; READMEs summarize current status and do not create a competing plan.

The independent [`fluxel-rhi`](https://github.com/fluxel-project/fluxel-rhi) is a stable maintenance baseline. Its authority is its [RHI architecture](https://github.com/fluxel-project/fluxel-rhi/blob/main/crates/rhi/DESIGN.md), [core API and SPI design](https://github.com/fluxel-project/fluxel-rhi/blob/main/crates/core/DESIGN.md), [verification record](https://github.com/fluxel-project/fluxel-rhi/blob/main/documents/verification.md), and [open TODOs](https://github.com/fluxel-project/fluxel-rhi/blob/main/TODO.md). It owns portable resources, command recording, submission, completion, presentation, device loss, and private backend realization. It changes only when an implemented consumer supplies a bounded requirement and an integration proof. Its open TODOs and target-evidence limits, including the current Metal evidence boundary, remain open facts; they do not block work that does not require them.

## Delivery order

```text
Foundation: Fluxel Bases asset-core reuse
  + Renderer-internal RenderGraph foundation and stable RHI integration
  -> Shader + Material core, starting with real WGSL/Naga-to-RHI compilation
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

Distinguish a stable Graph definition, its device-specific compiled plan, and
per-frame resource bindings. Reuse compatible plans and revalidate dynamic
bindings; topology, dependency, or incompatible resource-use changes require
recompilation. Reuse is a tested case, not a requirement that every frame has
identical topology.

## 3. Shader and Material core

Owner: `fluxel-rendering`.

Establish `fluxel-material` and `fluxel-shader` as separate crates. `fluxel-material` owns `MaterialGraph`, typed material values, validation, Material IR, parameter/resource schema, material assets and instances, and graph-to-IR lowering. `fluxel-shader` owns reusable shader composition, reflection, variants, `ShaderArtifact`, `ShaderInterface`, pipeline requirements, target profiles, and cache identity.

Material owns material function definitions/references, parameters/uniform
schemas, resource requirements, call composition, and material evaluation WGSL
generation. It converts Material IR into Shader-owned `ShaderCompositionRequest`
containing WGSL and generic interface metadata. Shader owns complete
program/module composition, entry points, compilation, metadata, and reflection;
it never imports MaterialIr or depends on Material. Renderer provides the pass
template and pipeline requirements. No new crate or executable IR is needed.

First prove a small WGSL -> Naga -> ShaderArtifact -> real RHI shader/pipeline
creation and executable output. Then compile one Rust-defined material through
the same route into an artifact, interface, and compiled material variant for
renderer preparation. Full MaterialGraph, Code Nodes, and PBR are not gates for
the first textured output. Shader remains reusable by compute and PostFX.

## 4. Material/data-driven Renderer MVP

Owner: `fluxel-rendering`.

Define the smallest `RenderScene`, `RenderObject`, `RenderView`, camera, mesh,
draw-list, material-selection, and `FramePipeline` vocabulary. Renderer owns the
FramePipeline module and exposes a public Rust extension trait and controlled
context for external implementations; Graph compiler internals remain private.
One built-in Forward path proves the normal scene-to-graph-to-RHI route.

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
   All frame-work resource accesses and passes remain under RenderGraph authority;
   a feature never submits independently.
4. A minimal Forward pipeline renders the in-tree mesh path with a prepared
   textured material. An independently compiled Rust feature then proves the
   same scene, material, shader, pipeline, graph, and RHI path before these
   exercised extension contracts are stabilized. Also prove an externally
   implemented FramePipeline through that public Rust facade.

The MVP includes an in-tree mesh path and one separate Rust crate implementing a minimal external `RenderFeature`. The external feature must accept batch incremental data, participate in a pipeline phase, and declare its GPU resource accesses through Graph without a core object-type switch or separate submissions. It need not implement a full terrain, water, or particle system. The gate proves that this extension uses the same material, shader, renderer, internal graph, and RHI contracts as the built-in mesh path. Include basic visibility, deterministic phase sorting/batching, stable generational render identities, and a frame snapshot boundary; ECS-style and ordinary adapters use the same batch boundary without introducing an ECS.

The integrated artifact must render a custom material function, replace the
FramePipeline from an external Rust crate, and consume the external Feature.
Prepared binding manifests cover all frame accesses; residency upload requests
use the shared coordinator described below rather than Feature submissions.

## 5. Broader renderer and RenderFeatures

Owner: `fluxel-rendering`.

The MVP validates only its exercised extension subset. A second, materially
different Feature must validate cross-pass and shader-extension interfaces
before those contracts stabilize.

Expand the MVP's culling, ordering, batching, material/shader selection, diagnostics, and retained scene updates against real workloads. Broaden MaterialGraph and material variants only where an implemented feature needs them. Forward+ and Deferred are independent internal `FramePipeline` implementations sharing the same material, shader, graph, and RHI contracts. Terrain, water, particles, sky, vegetation, and similar algorithms enter as RenderFeatures when a bounded consumer justifies them. This work consumes the extension seams verified in milestone 4; each pipeline explicitly rejects an unavailable phase or incompatible shader, binding, or capability requirement. Rust crate plugins remain the supported extension form. Dynamic DLL loading and a stable plugin binary ABI are unscheduled.

## 6. Performance and resource strategy

Owner: `fluxel-rendering`, with RHI changes only for proven consumer needs.

Develop GPU residency, upload policy, streaming coordination, LOD, GPU culling, indirect draws, batching, async compute, and memory budgeting from representative workloads and profiles. Every reuse, aliasing, or allocation claim needs a measured workload and completion-safe lifetime evidence.

## Deferred and undecided scope

Capture/replay, Canvas 2D and basic text, declarative UI, Blender tooling, and
JavaScript encapsulation are deferred and undecided. They have no approved
order, owner, architecture, API, dependency, or acceptance plan.

### Future engine runtime direction (not an approved stage)

This is a rendering ecosystem roadmap. A future Rust + TypeScript game engine
needs separate planning and ownership for Game Scene, gameplay/script runtime,
the game loop, Rust/TS bindings, and script hot reload. Preserve this possible
integration boundary without implementing those systems in the rendering MVP:

```text
TypeScript Gameplay / Scene -> Scene Adapter -> Batch ChangeSets
  -> Rust RenderScene -> FramePipeline -> RenderGraph -> RHI
```

Adapters map external scene identities to generational RenderIds and submit
batches; RenderScene provides render-only snapshots. `fluxel-jsbridge` owns
browser/mini-game platform lifecycle adaptation, separate from any future
TypeScript gameplay runtime. This direction does not authorize a runtime
milestone or settle the undecided JavaScript wrapper scope.

## Cross-cutting requirements

- Use precise pinned revisions for cross-repository integration, and retain the manifest that names every participating revision and lockfile.
- Resources are owned by device/context identity plus generation. A resource from another device or generation fails with a structured library error before a backend or driver is asked to use it.
- Browser and mini-game adapters own platform lifecycle adaptation only: canvas or surface setup, resize, foreground/background, and loss/restore. Backend-private GPU objects never enter the public rendering object model.
- Unsupported requirements fail explicitly. Do not silently approximate them.
- Material preparation resolves reflected binding usage to concrete resource
  identities, ranges, and access modes. Graph validates the contents of bindings
  against pass declarations and includes them in dependency/lifetime analysis.
- Renderer residency uploads may submit transfers outside Graph through one
  upload coordinator. Producer dependencies, resource states, and completion
  enter Graph/RHI synchronization before consumption; Features never submit
  independently. GPU skinning, simulation, culling, and other frame work remain
  Graph passes. Upload coordination does not become another frame scheduler.
- RHI maintenance work follows real renderer/graph consumers; it is not a parallel feature backlog.
