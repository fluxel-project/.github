# Fluxel Ecosystem

Fluxel is a Rust-first rendering ecosystem with Blender as its primary
authoring environment. Rust rendering and host contracts are authoritative;
browser, JavaScript, WASM, and tooling layers adapt them.

The four monorepositories are `fluxel-bases`, `fluxel-rendering`,
`fluxel-host`, and `fluxel-jsbridge`. The portable RHI baseline is complete at
`0.16`; the current authorized work is `0.17` shader assembly and material
semantics.

The host repository contains both reusable platform crates and runtime
composition. Platform crates do not depend on rendering; runtime composition
may combine them with `fluxel-rendering` to produce native applications.

Fluxel is not a compatibility layer for another engine, UI framework, DOM, or
CSS implementation. It grows through demonstrated vertical slices and named
real-target evidence.

For the authoritative version sequence, ownership model, and evidence rules,
see the organization documentation:

- [Roadmap](https://github.com/fluxel-project/.github/blob/main/ROADMAP.md)
- [Ecosystem architecture](https://github.com/fluxel-project/.github/blob/main/ECOSYSTEM_ARCHITECTURE.md)
- [Development principles](https://github.com/fluxel-project/.github/blob/main/DEVELOPMENT_PRINCIPLES.md)
- [Evidence policy](https://github.com/fluxel-project/.github/blob/main/EVIDENCE_POLICY.md)
