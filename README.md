# Fluxel Project Documentation

This repository is Fluxel's organization-level documentation home. It records
ecosystem ownership, delivery order, stable engineering rules, and evidence
requirements. Individual repositories own their public APIs, implementation
details, and release procedures.

Fluxel currently has four monorepositories: `fluxel-bases`,
`fluxel-rendering`, `fluxel-host`, and `fluxel-jsbridge`. The completed `0.16`
baseline establishes the portable RHI. The authorized next release is `0.17`,
shader assembly and material semantics.

`ROADMAP.md` is the single source of truth for version order and authorized
work. This README intentionally summarizes the current state and does not keep
a second stage history.

- [Roadmap](ROADMAP.md)
- [Ecosystem architecture](ECOSYSTEM_ARCHITECTURE.md)
- [Development principles](DEVELOPMENT_PRINCIPLES.md)
- [Evidence policy](EVIDENCE_POLICY.md)
- [Organization profile](profile/README.md)
- [Stage 1 Windows execution guide](stages/stage-01-windows.md)
- [Stage 2.1 WebGL2 execution guide](stages/stage-02-web.md)
- [Stage 2.2 WebGPU execution guide](stages/stage-02-webgpu.md)
