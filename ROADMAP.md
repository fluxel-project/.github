# Fluxel Roadmap

This is the single source of truth for Fluxel version order, authorized work,
and delivery sequencing. Repository documents define the public contract and
implementation detail for work authorized here; READMEs summarize current
status and must not create a competing roadmap.

The completed `0.16` release establishes the portable RHI baseline. The
remaining sequence is deliberately consumer-first where a public SPI needs its
input model before it can be frozen.

```text
0.17 shader/material semantics
  -> 0.18 minimal RenderScene/RenderView + FramePipeline SPI
  -> 0.19 retained scene completion + reference pipelines
  -> 0.20 Blender tooling
  -> 0.21 preview/runtime equivalence and export
  -> 0.22 capture/replay
  -> 0.23 JavaScript API
  -> 0.24 Canvas 2D and minimal text
  -> 0.25 declarative UI
```

## Completed baseline

### 0.16 — Portable RHI baseline

Completed. The RHI provides portable resource, recording, submission,
completion, presentation, loss, and capability contracts. Backends are
evidence for this one model, not separate public architectures.

## Product train

### 0.17 — Shader assembly and material system

Owner: `fluxel-rendering`.

Entry gate: the RHI must reject incomplete finite-domain capability snapshots
during device construction so every query on a published device is total. The
renderer-owned, workspace-private RenderGraph/RHI bridge ownership is frozen;
neither public crate may acquire a dependency on the other.

Deliver `MaterialGraph`, typed values, validation, Material IR, parameter and
variant semantics, shader assembly, reflection, artifacts, and cache identity.
Prove the Rust graph-to-variant route that later Blender tooling consumes.
Blender nodes are an input format, never the public semantic model.

### 0.18 — Minimal scene inputs and FramePipeline SPI

Owner: `fluxel-rendering`.

First freeze the smallest `RenderScene`, `RenderObject`, and `RenderView`
input model. Then define `FramePipeline` over those inputs, prove one Forward
reference pipeline, and lower it through the RenderGraph/RHI bridge. This
prevents a pipeline SPI from being frozen before its primary consumer inputs
exist.

### 0.19 — Retained RenderScene and reference-pipeline completion

Owner: `fluxel-rendering`.

Complete retained scene updates, culling, deterministic ordering, visibility,
frame preparation, and material/shader variant selection. Finish the Forward
reference pipeline, then add Deferred as an independent second implementation
of the SPI. Neither reference pipeline owns separate material semantics.

### 0.20 — Blender-native editor and runtime loop

Owner: `fluxel-rendering`, with scoped Blender tooling.

Deliver the native Blender add-on/tooling path, Blender node-tree to
`MaterialGraph` translation, material and scene import, structured diagnostics,
and Fluxel viewport preview. Unsupported nodes and shader/material failures
must be explicit diagnostics.

### 0.21 — Preview/runtime equivalence and export

Owner: `fluxel-rendering`, with scoped Blender tooling.

Deliver asset export, standalone Rust runtime loading, equivalence fixtures,
and evidence that the preview and exported runtime consume the same established
material, scene, renderer, RenderGraph, and RHI contracts.

### 0.22 — RenderScene recording and replay

Owner: `fluxel-rendering`.

Record and replay the complete RenderScene-driven path: scene inputs,
view/frame configuration, material/shader decisions, pipeline selection,
RenderGraph inputs, portable execution evidence, and observations. Replay uses
the normal renderer, bridge, RenderGraph, and RHI contracts; it is not a second
renderer.

### 0.23 — JavaScript API interface

Owner: `fluxel-jsbridge`.

Expose a narrow JavaScript API over established Rust contracts. JavaScript does
not own GPU resources, shader semantics, material identity, or RenderScene.

### 0.24 — Canvas 2D and minimal text

Owner: `fluxel-rendering`.

Deliver Canvas-style 2D drawing and deliberately minimal text on prepared
resources and the established RenderGraph/RHI architecture. This is the input
layer for UI, not a DOM/CSS compatibility surface.

### 0.25 — Declarative UI

Owner: `fluxel-rendering`.

Deliver a purpose-built declarative UI on the proved Canvas, text, input, and
host lifecycle foundations. It is not a Vue, DOM, CSS, or third-party UI
compatibility layer.

## Cross-cutting completion rule

Every milestone requires implementation, structured refusal behavior, unit and
contract tests, relevant backend evidence, and an end-to-end ecosystem proof.
One green library test cannot close a cross-repository milestone. See
[EVIDENCE_POLICY.md](EVIDENCE_POLICY.md).
