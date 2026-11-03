# Fluxel Ecosystem

Fluxel is a Rust-first rendering ecosystem. Rust rendering and host contracts
are authoritative.

The four monorepositories are `fluxel-bases`, `fluxel-rendering`,
`fluxel-host`, and `fluxel-jsbridge`; the portable RHI lives in the independent
[`fluxel-rhi`](https://github.com/fluxel-project/fluxel-rhi) repository. RHI is
a stable maintenance baseline with explicit open TODOs and target-evidence
limits. Rendering implements the Bases asset integration, private RenderGraph,
independent Shader core, minimal Material core, and a bounded dense CPU
closed bounded CPU RenderScene/typed-feature data boundary and a provisional
public CPU-feature facade. Its independently reviewed typed storage, issued
handles, and controlled CPU phase do not establish a FramePipeline or GPU scene
API; those remain pending, along with the roadmap's evidence and support limits.

The host repository contains both reusable platform crates and runtime
composition. Platform crates do not depend on rendering; runtime composition
may combine them with `fluxel-rendering` to produce native applications.

Fluxel grows through demonstrated Rust vertical slices and named real-target
evidence. Capture/replay, Canvas 2D/basic text, declarative UI, Blender tooling,
and JavaScript encapsulation are deferred and undecided.

The target renderer is data-oriented and ECS-friendly without requiring an ECS.
Applications assemble ordinary Rust-crate Features and frame policies from typed
batches; local geometry, visibility, ordering, batching, and deformation choices
can coexist. Renderer centralizes frame composition, Graph authority, and
completion-safe GPU lifetime rather than imposing a fixed mesh or one culling
algorithm.
Rendering extensions use Rust generics, traits and monomorphization exclusively,
with no trait-object dispatch or runtime type erasure. Runtime enablement uses
state; required implementation switching uses explicit enum branches among
known compiled types.

For the authoritative plan sequence, ownership model, and evidence rules,
see the organization documentation:

- [Roadmap](https://github.com/fluxel-project/.github/blob/main/ROADMAP.md)
- [Ecosystem architecture](https://github.com/fluxel-project/.github/blob/main/ECOSYSTEM_ARCHITECTURE.md)
- [Development principles](https://github.com/fluxel-project/.github/blob/main/DEVELOPMENT_PRINCIPLES.md)
- [Evidence policy](https://github.com/fluxel-project/.github/blob/main/EVIDENCE_POLICY.md)
