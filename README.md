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
Rendering implements the independent Shader and minimal Material cores, the
closed bounded S4 CPU RenderScene/typed-feature facade, and the
[closed S5 Forward candidate](https://github.com/fluxel-project/fluxel-rendering/blob/6e160488d0e3e4d6663f7a866e2b445235b1cbd8/documents/s5-forward-closure.md). S5 adds the bounded public FramePipeline,
scene preparation, residency and frame data, with six reviewed DX12/WGL native
attempts and retained release measurements. S6's bounded product MVP is
accepted for the audited DX12/WGL matrix: an external typed Feature, custom
material and pipeline compose with a built-in mesh in one Graph-backed frame.
Its [closure record](https://github.com/fluxel-project/fluxel-rendering/blob/c163db2362154fbb75289a6a5263312b69350d1e/documents/s6-typed-forward-closure.md)
and archived five-sample baselines retain the limited evidence. The RHI
`b5d569f` and Bases `ee640d5` consumer revisions remain unchanged; the native
window policy is supplied by Host `8d10162`. S7 is closed only for its bounded
Deferred/PBR/CodeNode DX12/WGL consumer. S8 is closed only for the selected
10,000-draw/1,000-texture Forward CPU/API validation workload and lifecycle
matrix; its [measured closure](https://github.com/fluxel-project/fluxel-rendering/blob/c24f9adf17cfd12ed2bc3035f45cd1f477879afc/documents/s8-measured-closure.md)
records the source-bound evidence. These results do not claim GPU timing,
browser/Metal support, Unity/SRP parity, general lighting, Forward+, shadows,
or post-processing. Historical S2/S3 and S4 closures retain their original
source/evidence identities.

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
