# Fluxel Roadmap

Fluxel is a Rust-first rendering framework with Blender as its primary
authoring environment. This roadmap is the ecosystem delivery authority;
repository documents define detailed contracts.

## Current direction

```text
RHI -> RenderGraph -> shader assembly/materials -> renderer framework
     -> custom pipeline SPI + Forward + Deferred
     -> native Blender editor/tooling -> preview/export/runtime equivalence
```

The immediate feature priority after the completed `0.16` RHI baseline is
shader assembly and the material system. Rust owns the semantics; Blender provides the editing
experience. The target is Blender-authored material == Fluxel runtime material,
not compatibility with EEVEE, Cycles, or a JavaScript rendering model.

## Completed 0.16 baseline

The current `0.16` release is the completed RHI baseline:

```text
0.16  completed portable RHI protocol and backend baseline
```

These versions establish device/context identity, generation, resource,
submission, completion, and graph contracts. WebGPU, WebGL2, and other
platforms are backend evidence for the same RHI, not separate architectures.

## Product train

### 0.17 — Shader assembly and material system

Owner: `fluxel-rendering`.

Deliver:

- `MaterialGraph`, typed values, validation, and `Material IR`;
- dynamic/static parameters and `MaterialAsset`/`MaterialInstance`;
- shader modules, pass templates, interface composition, reflection, variant
  keys, artifacts, and cache identity;
- a Rust graph-to-variant proof;
- a Rust-only graph-to-variant proof producing the same semantic contract that
  Blender will consume later.

Blender nodes are an input format, not the public semantic model.

### 0.18 — RenderGraph and renderer framework

Owner: `fluxel-rendering`.

Deliver:

- a custom `FramePipeline` SPI;
- built-in Forward and Deferred pipelines;
- RenderGraph lowering and evidence that all routes consume the same material
  and shader contracts.

The full retained scene model is defined in `0.19`.

The built-in pipelines are reference pipelines, not competing material
implementations.

### 0.19 — RenderScene

Owner: `fluxel-rendering`.

Deliver:

- `RenderScene`, `RenderObject`, and `RenderView`;
- culling, deterministic ordering, visibility, and frame preparation;
- material/shader variant selection;
- scene-to-`FramePipeline`-to-RenderGraph construction.

### 0.20 — Blender-native editor and runtime loop

Deliver the native Blender add-on/tooling path, material and scene import,
structured diagnostics, and Fluxel viewport preview.

### 0.21 — Preview/runtime equivalence and export

Owner: `fluxel-rendering`, with a scoped Blender tooling package.

Deliver:

- native Blender add-on/tooling for material and scene translation;
- Blender diagnostics for unsupported nodes and shader/material failures;
- Fluxel viewport preview through the renderer;
- asset export, standalone Rust runtime loading, and equivalence fixtures.

### 0.22 — RenderScene recording and replay

Record and replay the complete RenderScene-driven path, including scene inputs,
view/frame configuration, material/shader decisions, pipeline selection,
RenderGraph inputs, portable execution evidence, and output observations.
Replay uses the normal renderer, RenderGraph, and RHI contracts.

### 0.23 — JavaScript API interface

Expose a narrow JavaScript API over the established Rust contracts. JavaScript
does not own GPU resources, shader semantics, material identity, or RenderScene.

### 0.24 — Declarative UI and Canvas 2D

Deliver a declarative Vue-like UI framework and a Canvas 2D API on top of the
Fluxel renderer and prepared resources. Both consume the established
RenderGraph/RHI architecture.

## Ecosystem structure

```text
fluxel-bases       shared platform-neutral mechanisms when proven reusable
fluxel-rendering   RHI, RenderGraph, shader/material, renderer, Blender tools
fluxel-host        native host lifecycle and application integration
fluxel-jsbridge    JavaScript/browser/mini-game adapters when concretely needed
```

The dependency direction and public resource rules are defined in
[ECOSYSTEM_ARCHITECTURE.md](ECOSYSTEM_ARCHITECTURE.md). No layer may expose a
browser session/token model or move backend-private GPU objects into the
public architecture.

## Completion rule

Every milestone requires implementation, structured refusal behavior, unit and
contract tests, relevant backend evidence, and an end-to-end ecosystem proof.
One green library test cannot close a cross-repository milestone. Detailed
evidence requirements remain in [EVIDENCE_POLICY.md](EVIDENCE_POLICY.md).
