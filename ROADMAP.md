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

The S0/S1 foundation has been accepted for the named Windows DX12 and WGL
matrix. Its [Rendering closure record](https://github.com/fluxel-project/fluxel-rendering/blob/1085c9e7e8fd50b1cfa58201b1838687263c482e/documents/s0-s1-closure.md)
identifies exact dependencies, evidence and deferred targets. Rendering commit
[`8ddf2da`](https://github.com/fluxel-project/fluxel-rendering/commit/8ddf2da072e7d48331e423a6921f4deaffadf279)
implements S2 Shader and minimal S3 Material, with focused tests and named
Windows DX12/WGL integration runs. The subsequent review follow-up closes the
named S2/S3 fixture matrix with refreshed native evidence using Bases
`ee640d57a8d7c32d251252d0e8b32950636a770c`; the
[S2/S3 closure record](https://github.com/fluxel-project/fluxel-rendering/blob/main/documents/s2-s3-closure.md)
retains the exact source archives, locks, diagnostics and inspected output.
This does not extend support to other targets or certify full regression.
Rendering also has a closed bounded S4 CPU
RenderScene/typed-feature data boundary, with independently reviewed external
evidence at product `e4ec252`. The facade is not FramePipeline or GPU scene work
and creates no new support claim. The subsequent [S5 Forward closure](https://github.com/fluxel-project/fluxel-rendering/blob/6e160488d0e3e4d6663f7a866e2b445235b1cbd8/documents/s5-forward-closure.md)
adds the bounded public FramePipeline, multi-layout Scene/Material/Shader
preparation, persistent residency and completion-owned frame data. Its six
DX12/WGL Forward, device-isolation and Surface attempts have independently
reviewed raw evidence, and both routes retain five-sample release baselines.
This closes the S5 candidate. S6's bounded product MVP is now accepted: its
external Rust crate composes a typed Feature, custom material and pipeline with
the built-in mesh in one frame through the normal Graph path. Four audited DX12
and WGL offscreen/Surface attempts and archived five-sample baselines support
only that bounded matrix; [the S6 closure record](https://github.com/fluxel-project/fluxel-rendering/blob/c163db2362154fbb75289a6a5263312b69350d1e/documents/s6-typed-forward-closure.md)
retains the exact source and evidence identity. S7 is closed only for its named
Deferred DX12/WGL consumer, and S8 only for its named material-heavy Forward
CPU/API workload and lifecycle matrix. The final closure record identifies the
exact revisions and commands. Other backends, browser hosts, and unselected
pipeline capabilities remain open or deferred; no stage creates a general
engine or programmable-render-pipeline claim.

The subsequent [S0–S8 maintenance review](https://github.com/fluxel-project/fluxel-rendering/blob/main/documents/review-follow-up.md#current-maintenance-review)
addresses texture-subresource admission and commit-level CPU/tooling gates.
Historical GPU archives keep their original input identities. Future cross-pass
extension work must first select a real external producer/consumer; this review
does not add new pipeline features or expand backend certification.

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

Its first consumer is a standalone WGSL vertex/fragment or compute program that is validated and reflected, then creates a real RHI shader and pipeline and executes through S1. Its logical input interface supports consumer-provided geometry semantics without freezing a renderer vertex struct; Preparation later resolves stream packing and executable layouts. `fluxel-rhi` remains the authority for an opened device's shader route, target ABI key, backend Naga/native lowering, native compilation, and executable pipeline validation. Logical source artifacts cache from logical composition inputs; device-resident shader and pipeline keys additionally include the selected device, route, and ABI facts.

**Exit:** a complete non-Material program produces retained real-target output, with reflected logical bindings, selected RHI route, exact revision, and structured unsupported or ABI-mismatch diagnostics.

## S3 — Material core

Owner: `fluxel-rendering`.

Create `fluxel-material` for material functions, typed parameter and resource schemas, static choices, material assets/instances, normalized Material IR, and generation of evaluation WGSL. The first implementation is Rust-defined functions and composition; a complete authoring graph or broad Code Node user experience is not a prerequisite.

Material lowers its private IR to generic WGSL source units and interface metadata in Shader-owned vocabulary, optionally named `MaterialShaderContribution`. The FramePipeline/renderer combines that contribution with a pass template, entry points, geometry interface, target facts, and static inputs into `ShaderCompositionRequest`. Shader never imports Material IR; Material never chooses Forward or Deferred.

**Exit:** several functions and parameters compose into evaluation WGSL, then into a complete Shader program that the S2 route compiles, reflects, and executes through the S1 Graph path with retained real-target output.

## S4 — RenderScene and prepared extension boundary

Owner: `fluxel-rendering`.

The implemented CPU subset provides render-only `RenderScene<R>` storage with
generational `RenderId`/view IDs, consumer-chosen cloneable record resources,
views, transforms, bounds, transactional ChangeSets, dirty ranges, and stable
frame snapshots. `RenderResources<G, M>` is a convenience geometry/material
reference bundle, not a per-scene type limit. Ordinary and ECS-shaped adapters
map external identity into that batch boundary; Renderer does not import an ECS
or prescribe a universal SoA/AoS layout. This is not heterogeneous Feature
storage. Mutable records are dense and compact on removal while stable
generational identities and dirty slot ranges remain observable.

The independent external CPU reference compiles against the public scene and
CPU-feature types, compares ordinary and ECS-shaped adapters, retains an old
snapshot across an update, and associates two differently shaped application-
owned typed payloads with scene versions. It uses `FeatureIssuer`-issued opaque
handles and contexts, issuer-bound `TypedFeatureStorage`, the static
`CpuFeature` trait, and controlled CPU phase contributions. Scene, issuer,
handle, version, storage, collector, and replay mismatches are structured
errors. It does not define `RenderFeature` or `FramePipeline`, and cannot reach
Graph/RHI execution state. Graph compiler, allocator, scheduler, RHI submission,
and backend objects remain private.

This is a bounded provisional public CPU facade, not a Forward/GPU extension
surface. The compile-fail negative confirms that the external dependency cannot
name Renderer-private Graph state. Its original data/privacy and independent-
consumer acceptance is closed by the reviewed
[S4 CPU closure record](https://github.com/fluxel-project/fluxel-rendering/blob/main/documents/s4-cpu-closure.md).
No unfinished S4 requirement moves to S5. S5 consumes this closed CPU boundary
for Forward/GPU integration and does not retroactively expand it.

Algorithm-specific traits normally remain Feature- or consumer-owned. Core adds
only a cross-Feature protocol with demonstrated shared snapshot, identity,
phase/Graph, resource-access, and lifecycle semantics; repeated consumers alone
do not justify promotion. Rendering extensibility uses Rust generics, traits
and monomorphization exclusively, including setup. Concrete pipelines and typed
Feature sets are composed at compile time without `dyn Trait`, trait-object
registries or runtime type erasure. Runtime enablement uses state; required
selection uses explicit enum branches over known compiled implementations.
Start with ordinary traits, structs, tuples and necessary associated types,
not a generic framework. Static dispatch still needs measurement of code size,
compile time and workload behavior rather than a zero-cost claim.

Focused CPU evidence covers incremental multi-object/view updates, stale-ID and
conflicting-batch refusal, capacity and value refusal, immutable concurrent
snapshot reads, dirty ranges, adapter equivalence, and the public CPU facade's
typed-storage/issuer/phase error boundaries. This is not GPU culling, indirect
work, a FramePipeline contract, or a GPU/full-regression closure.

## S5 — Forward integration candidate

Owner: `fluxel-rendering`.

Implement the public FramePipeline facade, phase policy, batch preparation, Material/Shader resolution, persistent residency, and a minimal Forward pipeline. Local visibility, ordering, batching, geometry, and deformation algorithms remain replaceable Feature/Pipeline choices; no global culling/sorting singleton is required. Render a textured multi-mesh scene through `RenderScene -> FramePipeline -> RenderGraph -> RHI`.

Persistent residency uploads serve immutable or streaming device-local content. Per-frame transform, uniform, and bone-palette data use a Frame Data Upload path with a separate lifetime and update policy, although both may reuse the same low-level upload mechanism and RHI completion/state authority. GPU skinning, simulation, culling, and indirect generation are RenderGraph passes.

An accepted upload with an enforceable RHI dependency may serve its named graph consumer before terminal completion; it is not globally complete or reusable. Features request preparation and never submit independently.

**Exit:** prepared binding manifests cover every concrete buffer/texture access, including ranges and dynamic changes; the in-tree Forward path renders real output; compatible interleaved, separate, and mixed geometry streams resolve against Shader requirements; and the external prototype replaces one local algorithm through its public facade. Missing attributes or incompatible phase/product combinations fail structurally.

## S6 — Extensibility MVP closure

Owner: `fluxel-rendering`, with an independent external Rust crate.

The external crate supplies one RenderFeature and one FramePipeline through only the public Renderer facade shaped in S4/S5. In one real frame it contributes batched typed data and a phase, declares Graph resource access, uses a custom material function, and renders with the same Material, Shader, Renderer, Graph, and RHI path as the built-in mesh. The same frame demonstrates composition with a distinct local provider; phase policy performs any required final merge/order or rejects an incompatible combination. It cannot introduce a core object-type switch, direct submission, or a second frame scheduler.

**Exit:** that integrated artifact and its failure cases prove the exercised extension subset, using statically composed concrete Feature and Pipeline types without trait-object dispatch or runtime type erasure. This is the first renderer product closure. Runtime loading of unknown Feature types and a stable plugin binary ABI are outside the static extension architecture.

The bounded exit is accepted for the audited DX12/WGL matrix. Rendering retains
the integration artifact and external crate; the consumed RHI (`b5d569f`),
Bases (`ee640d5`), and Host (`33a6bbb`) revisions remain unchanged. Native
Surface resize is Host lifecycle evidence, while logical asset replacement is
the Bases contract consumed by Rendering. This does not authorize browser,
Metal, broad plugin, or production-scene support claims.

## S7 — Selected rendering expansion

Owner: `fluxel-rendering`.

Expand only through selected consumers: Code Nodes and broader MaterialGraph authoring, PBR, Forward+, Deferred, shadows, PostFX, and chosen Features such as skinning, terrain, water, or particles. Deferred is a second FramePipeline over shared scene/material contracts. A materially different second Feature proves the selected deformation, GPU-product, cross-pass, or shader-extension contracts before they stabilize. Each selected capability has a bounded acceptance artifact; this stage does not require every listed algorithm.

## S8 — Measured performance and resource strategy

Owner: `fluxel-rendering`, with RHI changes only for proven consumer needs.

Measurements begin with every earlier real workload. Add batching, streaming, LOD, GPU culling, indirect work, async compute, allocation reuse, aliasing, and memory budgeting only when representative baselines identify a need. Each selected optimization retains workloads, raw metrics, thresholds, completion-safe lifetime evidence, and the relevant target matrix; this stage does not require every listed technique.

The selected S8 result is an index-based Graph-validation optimization for the
10,000-draw/1,000-texture material-heavy Forward workload. It retains raw
samples, thresholds, allocation/upload counters, and completion-safe lifecycle
evidence. CPU/API median submit-time improvements were DX12 89.976% cold,
91.598% warm, and 92.166% after ten-percent replacement; WGL was 91.714%,
91.779%, and 92.429%. No GPU timestamps were observed, so these values do not
claim GPU-duration improvement. Native lifecycle workloads cover bounded
in-flight replacement and device separation; Renderer fault tests separately
cover cancellation plus upload/recording failure ownership. The result does not
certify browser support, Unity/SRP parity, general lighting, Forward+, shadows,
post-processing, or unselected techniques.

## Deferred and future scope

Capture/replay, Canvas 2D/basic text, declarative UI, Blender tooling, and JavaScript encapsulation are deferred and undecided. A future game engine needs separate planning for Game Scene, gameplay/script runtime, game loop, bindings, and hot reload. It may adapt batches into RenderScene, but it is not a Rendering Roadmap stage.

## Cross-cutting requirements

- Use pinned revisions and retain the integration manifest, locks, target matrix, commands, and evidence for every cross-repository closure.
- Resources are valid only for the selected RHI device identity and its current lifecycle. A foreign or stale resource returns a structured error before a backend call; renderer-private cache keys use only identity facts exposed by the selected RHI baseline.
- Browser and mini-game adapters own lifecycle adaptation only. GPU objects, contexts, leases, and browser promises stay backend-private.
- Unsupported requirements fail explicitly. Material preparation resolves reflected binding usage to concrete resource identities, ranges, and access modes; Graph validates it before recording.
