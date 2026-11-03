# Fluxel Project Documentation

This repository is Fluxel's organization-level documentation home. It records
ecosystem ownership, delivery order, stable engineering rules, and evidence
requirements. Individual repositories own their public APIs, implementation
details, and release procedures.

Fluxel currently has four monorepositories: `fluxel-bases`,
`fluxel-rendering`, `fluxel-host`, and `fluxel-jsbridge`, plus the independent
[`fluxel-rhi`](https://github.com/fluxel-project/fluxel-rhi) repository. RHI is
a stable maintenance baseline: consumer-proven additions remain possible, while
its open TODOs and target-evidence limits remain explicit. The current roadmap
starts with the Bases asset core and renderer-internal RenderGraph integration.
Rendering implements the independent Shader and minimal Material cores plus a
bounded dense CPU RenderScene snapshot reference and a provisional public
CPU-feature facade. The facade supplies typed storage, issued handles, and a
controlled CPU phase only; FramePipeline and GPU scene integration remain open.
See the roadmap for implementation and evidence limits.

`ROADMAP.md` is the single source of truth for plan order and authorized work.
This README intentionally summarizes the current state and does not keep a
second plan history.

Fluxel Rendering is a data-oriented, trait-extensible framework: external Rust
crates assemble Feature-local geometry, visibility, ordering, batching, and
deformation algorithms over typed batches. Renderer remains ECS-friendly without
depending on an ECS; FramePipeline coordinates phases and Graph remains the only
normal GPU-work route. The roadmap defines when these contracts become public.
Rendering extension contracts use generics, traits and monomorphization
exclusively, with no trait-object dispatch or runtime type erasure. Runtime
enablement uses state; implementation switching uses explicit enum branches
among compiled concrete types.

- [Roadmap](ROADMAP.md)
- [Ecosystem architecture](ECOSYSTEM_ARCHITECTURE.md)
- [Development principles](DEVELOPMENT_PRINCIPLES.md)
- [Evidence policy](EVIDENCE_POLICY.md)
- [Organization profile](profile/README.md)
- [Historical Windows execution guide](stages/stage-01-windows.md)
- [Historical WebGL2 execution guide](stages/stage-02-web.md)
- [Historical WebGPU execution guide](stages/stage-02-webgpu.md)
