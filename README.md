# Fluxel Project Documentation

This repository is the authoritative home for Fluxel's ecosystem-wide
architecture, roadmap, development principles, and evidence rules. Individual
repositories document their own implementation and release procedures.

- [Organization profile](profile/README.md)
- [Ecosystem architecture](ECOSYSTEM_ARCHITECTURE.md)
- [Roadmap](ROADMAP.md)
- [Development principles](DEVELOPMENT_PRINCIPLES.md)
- [Evidence policy](EVIDENCE_POLICY.md)
- [Stage 1 Windows execution guide](stages/stage-01-windows.md)
- [Stage 2.1 WebGL2 execution guide](stages/stage-02-web.md)
- [Stage 2.2 WebGPU execution guide](stages/stage-02-webgpu.md)

Fluxel is organized as four monorepos: `fluxel-bases`, `fluxel-rendering`,
`fluxel-host`, and `fluxel-jsbridge`. Repository boundaries are established;
internal crates and packages are still introduced only when a roadmap stage
demonstrates their need.
