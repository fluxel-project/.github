# Fluxel Roadmap

This is the single source of truth for Fluxel plan order and milestone authorization. Repository documents define public contracts and implementation details; READMEs summarize current status and do not create a competing plan.

The independent [`fluxel-rhi`](https://github.com/fluxel-project/fluxel-rhi) is a stable maintenance baseline. It owns portable resources, command recording, submission, completion, presentation, device loss, and private backend realization. It changes only when an implemented consumer supplies a bounded requirement and an integration proof. Its open TODOs and target-evidence limits, including the Metal evidence boundary, remain open facts; they do not block work that does not require them.

## Delivery model

S0 through S8 are implementation work packages, not release promises. Each closes only with implementation, structured refusal behavior, focused tests, relevant backend evidence, and an end-to-end ecosystem artifact. A green library test cannot close a cross-repository milestone.

```text
S0 Foundation -> S1 RenderGraph -> S2 Shader
                                      |-> S3 Material --+
                                      |-> S4 RenderScene +-> S5 Forward -> S6 Extension MVP
                                                                    -> S7 selected expansion -> S8 optimization
```

S0 and S1 are one foundation closure. After S0 pins the inputs, their bounded work may overlap, but neither closes independently. S4 may proceed alongside S3 after S2 establishes the executable shader route. S5 shapes the public extension surface and supplies a small external compile/prototype proof; S6 is the integrated closure that proves it under real rendering work.

## S0 — Foundation

Owners: `fluxel-bases`, `fluxel-rendering`, and `fluxel-rhi` for their respective contracts.

Reuse the Bases asset core: typed `AssetId`, immutable content generation, snapshots, producer coordination, and CPU cache policy. Assets do not own GPU residency or source I/O policy. Select the exact stable RHI revision and features; do not treat the renderer's historical RHI pin as compatibility evidence.

Before changing the dependency edge, name one runnable integration artifact in `fluxel-rendering`, its owning smoke script, every participating repository revision, each relevant lockfile, and the named target/backend matrix. The evidence manifest schema is required for this closure.

**Exit:** the artifact builds the selected Bases/RHI/Rendering revisions together, observes asset identity and content replacement, creates/uploads/submits a small resource fixture, and retains resources from RHI completion facts.

## S1 — Renderer-internal RenderGraph

Owner: `fluxel-rendering`, consuming the selected `fluxel-rhi` contract.

Move the standalone RenderGraph into `fluxel-renderer`; do not maintain two graph engines. Preserve logical resource versions, definedness, pass-local access, dependencies, deterministic scheduling, transient lifetime analysis, diagnostics, and direct RHI execution. A fixed shader or copy/clear fixture is sufficient; Material and RenderScene are not prerequisites.

A graph definition, device-specific compiled plan, and per-frame binding are separate products. Resource instances may change without recompiling when their validated concrete manifests preserve declared accesses and dependency contracts. Recompile only when topology, hazard/dependency edges, binding layout, or incompatible access contracts change. Revalidation includes newly aliased dynamic resources.

**Exit:** Raster and Copy execute on the named matrix; Compute executes only on
selected targets that advertise its required route, while unsupported targets
such as WebGL reject it structurally before backend work. A compiled plan
survives two or three live submissions before waiting; completion points govern
reuse and retirement. Wrong-device, invalid-import, stale-resource, and
unavailable-capability failures have no native effect. Partial acceptance and
indeterminate native-effect outcomes retain affected resources until RHI supplies
a terminal outcome. Loss invalidates device-specific plans while preserving
logical asset snapshots.

## S2 — Shader core

Owner: `fluxel-rendering`.

Create `fluxel-shader` as reusable program composition infrastructure. It owns complete logical program assembly, source-level validation and reflection, logical interfaces, source artifacts, variants, and their cache identity. It is usable by Material, compute, particles, and post-processing.

Its first consumer is a standalone WGSL vertex/fragment or compute program that is validated and reflected, then creates a real RHI shader and pipeline and executes through S1. `fluxel-rhi` remains the authority for an opened device's shader route, target ABI key, backend Naga/native lowering, native compilation, and executable pipeline validation. Logical source artifacts cache from logical composition inputs; device-resident shader and pipeline keys additionally include the selected device, route, and ABI facts.

**Exit:** a complete non-Material program produces retained real-target output, with reflected logical bindings, selected RHI route, exact revision, and structured unsupported or ABI-mismatch diagnostics.

## S3 — Material core

Owner: `fluxel-rendering`.

Create `fluxel-material` for material functions, typed parameter and resource schemas, static choices, material assets/instances, normalized Material IR, and generation of evaluation WGSL. The first implementation is Rust-defined functions and composition; a complete authoring graph or broad Code Node user experience is not a prerequisite.

Material lowers its private IR to generic WGSL source units and interface metadata in Shader-owned vocabulary, optionally named `MaterialShaderContribution`. The FramePipeline/renderer combines that contribution with a pass template, entry points, geometry interface, target facts, and static inputs into `ShaderCompositionRequest`. Shader never imports Material IR; Material never chooses Forward or Deferred.

**Exit:** several functions and parameters compose into evaluation WGSL, then into a complete Shader program that the S2 route compiles, reflects, and executes through the S1 Graph path with retained real-target output.

## S4 — RenderScene and prepared extension boundary

Owner: `fluxel-rendering`.

Build render-only, data-oriented scene storage: generational `RenderId`, mesh, camera/view, transform, bounds, batched ChangeSets, dirty ranges, and a stable frame snapshot. ECS and ordinary adapters both map external identity into that batch boundary; Renderer does not import an ECS.

Define only the public Rust types needed by an external FramePipeline and RenderFeature: the extension trait, controlled context, opaque handles, phase contribution, error surface, and visibility restrictions. Keep Graph compiler, allocator, scheduler, RHI submission, and backend objects private. An external crate must compile against this provisional surface before it is stabilized.

**Exit:** multiple objects and views update incrementally with stale-ID and conflicting-batch refusal; snapshots admit concurrent reads; the external prototype cannot reach private Graph/RHI execution state.

## S5 — Forward integration candidate

Owner: `fluxel-rendering`.

Implement the public FramePipeline facade, phase policy, culling, sorting, draw preparation, Material/Shader resolution, persistent residency, and a minimal Forward pipeline. Render a textured multi-mesh scene through `RenderScene -> FramePipeline -> RenderGraph -> RHI`.

Persistent residency uploads serve immutable or streaming device-local content. Per-frame transform, uniform, and bone-palette data use a Frame Data Upload path with a separate lifetime and update policy, although both may reuse the same low-level upload mechanism and RHI completion/state authority. GPU skinning, simulation, culling, and indirect generation are RenderGraph passes.

An accepted upload with an enforceable RHI dependency may serve its named graph consumer before terminal completion; it is not globally complete or reusable. Features request preparation and never submit independently.

**Exit:** prepared binding manifests cover every concrete buffer/texture access, including ranges and dynamic changes; the in-tree Forward path renders real output and the external prototype exercises its public facade.

## S6 — Extensibility MVP closure

Owner: `fluxel-rendering`, with an independent external Rust crate.

The external crate supplies one RenderFeature and one FramePipeline through only the S4 public facade. In one real frame it contributes batched typed data and a phase, declares Graph resource access, uses a custom material function, and renders with the same Material, Shader, Renderer, Graph, and RHI path as the built-in mesh. It cannot introduce a core object-type switch, direct submission, or a second frame scheduler.

**Exit:** that integrated artifact and its failure cases prove the exercised extension subset. This is the first renderer product closure; dynamic DLL loading and a stable plugin binary ABI remain unscheduled.

## S7 — Selected rendering expansion

Owner: `fluxel-rendering`.

Expand only through selected consumers: Code Nodes and broader MaterialGraph authoring, PBR, Forward+, Deferred, shadows, PostFX, and chosen Features such as skinning, terrain, water, or particles. Deferred is a second FramePipeline over shared scene/material contracts. A materially different second Feature must prove cross-pass and shader-extension contracts before they stabilize. Each selected capability has a bounded acceptance artifact; this stage does not require every listed algorithm.

## S8 — Measured performance and resource strategy

Owner: `fluxel-rendering`, with RHI changes only for proven consumer needs.

Measurements begin with every earlier real workload. Add batching, streaming, LOD, GPU culling, indirect work, async compute, allocation reuse, aliasing, and memory budgeting only when representative baselines identify a need. Each selected optimization retains workloads, raw metrics, thresholds, completion-safe lifetime evidence, and the relevant target matrix; this stage does not require every listed technique.

## Deferred and future scope

Capture/replay, Canvas 2D/basic text, declarative UI, Blender tooling, and JavaScript encapsulation are deferred and undecided. A future game engine needs separate planning for Game Scene, gameplay/script runtime, game loop, bindings, and hot reload. It may adapt batches into RenderScene, but it is not a Rendering Roadmap stage.

## Cross-cutting requirements

- Use pinned revisions and retain the integration manifest, locks, target matrix, commands, and evidence for every cross-repository closure.
- Resources are valid only for the selected RHI device identity and its current lifecycle. A foreign or stale resource returns a structured error before a backend call; renderer-private cache keys use only identity facts exposed by the selected RHI baseline.
- Browser and mini-game adapters own lifecycle adaptation only. GPU objects, contexts, leases, and browser promises stay backend-private.
- Unsupported requirements fail explicitly. Material preparation resolves reflected binding usage to concrete resource identities, ranges, and access modes; Graph validates it before recording.
