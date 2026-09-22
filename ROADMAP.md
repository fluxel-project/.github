# Fluxel Roadmap

This is the single source of truth for Fluxel plan order, authorized work, and
delivery sequencing. Repository documents define the public contract and
implementation detail for work authorized here; READMEs summarize current
status and must not create a competing roadmap.

The portable RHI baseline is complete. The remaining plan is deliberately
consumer-first: a public SPI is not frozen before its primary input model has
been demonstrated.

```text
Shader and material semantics
  -> Minimal scene inputs and FramePipeline SPI
  -> Retained scene completion and reference pipelines
  -> Blender tooling
  -> Preview/runtime equivalence and export
  -> Render-scene recording and replay
  -> JavaScript API
  -> Canvas 2D and minimal text
  -> Declarative UI
```

## Completed baseline

### Portable RHI baseline

Completed. The RHI provides portable resource, recording, submission,
completion, presentation, loss, and capability contracts. Backends are
evidence for this one model, not separate public architectures.

## Product plan

### Shader assembly and material system

Owner: `fluxel-rendering`.

Entry gate: the RHI must reject incomplete finite-domain capability snapshots
during device construction so every query on a published device is total.

Deliver `MaterialGraph`, typed values, validation, Material IR, parameter and
variant semantics, shader assembly, reflection, artifacts, and cache identity.
Prove the Rust graph-to-variant route that later Blender tooling consumes.
Blender nodes are an input format, never the public semantic model.

### Minimal scene inputs and FramePipeline SPI

Owner: `fluxel-rendering`.

First freeze the smallest `RenderScene`, `RenderObject`, and `RenderView`
input model. `GeometryHandle` and `MaterialHandle` are conceptual typed logical
references; when backed by Fluxel assets, their durable identity is
`AssetId<K>` plus content generation, not a second asset-identity domain.
Renderer-local identities, such as a material-instance identity, remain
distinct from material-asset identity.

Then define `FramePipeline` over those inputs, prove one Forward reference
pipeline, and have it construct RenderGraph work using the RHI portable
contract. This prevents a pipeline SPI from being frozen before its primary
consumer inputs exist.

### Retained scene completion and reference-pipeline completion

Owner: `fluxel-rendering`.

Complete retained scene updates, culling, deterministic ordering, visibility,
frame preparation, and material/shader variant selection. Finish the Forward
reference pipeline, then add Deferred as an independent second implementation
of the SPI. Neither reference pipeline owns separate material semantics.

### Blender-native editor and runtime loop

Owner: `fluxel-rendering`, with scoped Blender tooling.

Deliver the native Blender add-on/tooling path, Blender node-tree to
`MaterialGraph` translation, material and scene import, structured diagnostics,
and Fluxel viewport preview. Unsupported nodes and shader/material failures
must be explicit diagnostics.

### Preview/runtime equivalence and export

Owner: `fluxel-rendering`, with scoped Blender tooling.

Deliver asset export, standalone Rust runtime loading, equivalence fixtures,
and evidence that the preview and exported runtime consume the same established
material, scene, renderer, RenderGraph, and RHI contracts.

### Render-scene recording and replay

Owner: `fluxel-rendering`.

Record and replay the complete RenderScene-driven path: scene inputs,
view/frame configuration, material/shader decisions, pipeline selection,
RenderGraph inputs, portable execution evidence, and observations. Replay uses
the normal renderer, RenderGraph, and RHI contracts; it is not a second
renderer.

### JavaScript API

Owner: `fluxel-jsbridge`.

Expose a narrow JavaScript API over established Rust contracts. JavaScript does
not own GPU resources, shader semantics, material identity, or RenderScene. An
SDK core may be extracted when an established Rust contract and at least one
real adapter prove a stable language-level boundary; later adapters must not
force platform-specific behavior into that core.

### Canvas 2D and minimal text

Owner: `fluxel-rendering`.

Deliver Canvas-style 2D drawing and deliberately minimal text on prepared
resources and the established RenderGraph/RHI architecture. This is the input
layer for UI, not a DOM/CSS compatibility surface.

### Declarative UI

Owner: `fluxel-rendering`.

Deliver a purpose-built declarative UI on the proved Canvas, text, input, and
host lifecycle foundations. It is not a Vue, DOM, CSS, or third-party UI
compatibility layer.

## Cross-cutting completion rule

Every plan milestone requires implementation, structured refusal behavior, unit
and contract tests, relevant backend evidence, and an end-to-end ecosystem
proof. One green library test cannot close a cross-repository milestone. See
[EVIDENCE_POLICY.md](EVIDENCE_POLICY.md).
