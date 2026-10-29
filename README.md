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
starts with the Bases asset core and renderer-internal RenderGraph integration,
then establishes separate Material and Shader crates.

`ROADMAP.md` is the single source of truth for plan order and authorized work.
This README intentionally summarizes the current state and does not keep a
second plan history.

- [Roadmap](ROADMAP.md)
- [Ecosystem architecture](ECOSYSTEM_ARCHITECTURE.md)
- [Development principles](DEVELOPMENT_PRINCIPLES.md)
- [Evidence policy](EVIDENCE_POLICY.md)
- [Organization profile](profile/README.md)
- [Historical Windows execution guide](stages/stage-01-windows.md)
- [Historical WebGL2 execution guide](stages/stage-02-web.md)
- [Historical WebGPU execution guide](stages/stage-02-webgpu.md)
