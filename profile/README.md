# Fluxel Ecosystem

Fluxel is a Rust-first rendering ecosystem. Rust rendering and host contracts
are authoritative.

The four monorepositories are `fluxel-bases`, `fluxel-rendering`,
`fluxel-host`, and `fluxel-jsbridge`; the portable RHI lives in the independent
[`fluxel-rhi`](https://github.com/fluxel-project/fluxel-rhi) repository. RHI is
a stable maintenance baseline with explicit open TODOs and target-evidence
limits. Rendering implements the Bases asset integration, private RenderGraph,
independent Shader core, minimal Material core, and the closed bounded S4 CPU
scene/feature data facade. The [S5 Forward candidate](https://github.com/fluxel-project/fluxel-rendering/blob/6e160488d0e3e4d6663f7a866e2b445235b1cbd8/documents/s5-forward-closure.md) now adds the public
bounded FramePipeline, scene preparation, residency and frame data. Six reviewed
DX12/WGL Forward/device-isolation/Surface attempts and separate release baselines
support that scope. The bounded S6 product MVP is accepted for the audited
DX12/WGL matrix: an external typed Feature and custom material/pipeline compose
with the built-in mesh in one Graph-backed frame. The [S6 closure record](https://github.com/fluxel-project/fluxel-rendering/blob/c163db2362154fbb75289a6a5263312b69350d1e/documents/s6-typed-forward-closure.md)
and archived baselines define that limited result. S7 is closed only for its
bounded Deferred/PBR/CodeNode DX12/WGL consumer, while S8 is closed only for
the selected 10,000-draw/1,000-texture Forward CPU/API validation workload and
lifecycle matrix; the [S8 measured closure](https://github.com/fluxel-project/fluxel-rendering/blob/main/documents/s8-measured-closure.md)
retains its source-bound evidence. Neither result claims GPU timing,
browser/Metal expansion, Unity/SRP parity, general lighting, Forward+, shadows,
or post-processing. Historical closure records retain their own exact input
identities.

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
