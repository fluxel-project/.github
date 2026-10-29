# Fluxel Development Principles

These principles govern the Fluxel ecosystem: the `fluxel-bases`,
`fluxel-rendering`, `fluxel-host`, and `fluxel-jsbridge` monorepositories, plus
the independent [`fluxel-rhi`](https://github.com/fluxel-project/fluxel-rhi)
repository. They are stable decision rules, not a release schedule or a promise
to create every named crate.

Fluxel is a native-first, AI-friendly lightweight rendering runtime with an
approachable scene API. Familiar concepts may inform usability studies, but no
third-party API or behavioral compatibility is implied.

## Native-first semantics

- Rust rendering and host contracts are authoritative. Platform integration does
  not define renderer or host core semantics.
- Native-first does not mean desktop-only. Portable behavior is proved on each
  target, while backend-specific objects remain behind native boundaries.
- Prefer explicit ownership, borrowing, typed handles, lifecycle states, and
  structured errors. Make invalid states unrepresentable when the Rust API can
  do so.

## Approachable native scene API, not a compatibility layer

- Fluxel does not target another engine's API or behavioral compatibility and
  is not a backend for another scene runtime.
- `Scene`, `Camera`, `Mesh`, `Geometry`, `Material`, and `Transform` may be
  adopted only when they are natural Rust concepts.
- Foreign inheritance, plugin contracts, implicit global state, material
  breadth, and compatibility wrapper types must not enter the native core.

## Demo-driven scope and ownership

- The next visible, testable demo closure determines implementation order.
  Repository ownership determines where code belongs, not when a crate must
  exist.
- [ECOSYSTEM_ARCHITECTURE.md](ECOSYSTEM_ARCHITECTURE.md) assigns each capability
  to an owning repository. It reserves candidate crate boundaries
  but does not authorize extraction or change roadmap order.
- Reuse and extend the current demo. Extract a crate only after the vertical
  slice demonstrates an independently changing responsibility.
- The planned `fluxel-material` and `fluxel-shader` crates are the explicit
  exception: their separate semantic responsibilities are part of the approved
  rendering target. Their APIs still grow only through vertical-slice evidence.
- When a real vertical slice discovers a missing capability owned by another
  repository, implement that owner's smallest complete behavior and make the
  current slice actually consume and verify it. Do not duplicate a temporary
  substitute in the caller, and do not use the discovery to start the owner's
  unneeded future framework.
- Do not broaden a public API for a hypothetical target. Record the missing
  case and wait for a stage that can prove it on that target.
- Optimize only after a representative baseline and profile identify a concrete
  cost. Feature count, backend count, and abstraction count are not progress.

## Ownership boundaries

- `fluxel-bases` is the dependency leaf. It holds shared mechanisms and
  contracts, including asset identity and lifecycle plus diagnostic schema and
  routing, but never platform implementations, GPU objects, or application
  policy.
- `fluxel-rhi` owns the portable RHI contract and its private backend
  implementations: physical resource realization, command recording,
  submission, completion, presentation, and GPU synchronization. It has no
  renderer, scene, main loop, or platform-service policy.
- `fluxel-rendering` owns renderer submission, rendering-side asset residency,
  the renderer-internal RenderGraph, `FramePipeline` SPI, material semantics,
  and shader infrastructure. Its
  target crate layout is `fluxel-material`, `fluxel-shader`, and
  `fluxel-renderer`; RenderGraph and FramePipeline are renderer internals. It
  may depend on `fluxel-rhi`; it does not own or embed RHI backends.
- `fluxel-host` owns native platform lifecycle and window-handle lifetimes,
  input and time acquisition, filesystem, storage,
  networking, audio, video, native diagnostic sinks, and executable packaging.
  Its reusable platform crates may depend on bases but not rendering. The host
  repository's runtime/executable-composition layer may depend on those crates
  and rendering; neither bases nor rendering may depend on host.
- A host window supplies standard window/display handles. A host-to-RHI
  integration adapter borrows them while establishing an opaque presentation
  target; `fluxel-rhi` never imports Host. RHI owns the native surface,
  swapchain, presentation, and GPU synchronization derived from the handle.
- Existing browser and mini-game integrations own their proved platform
  lifecycle adaptation and diagnostics; they do not become semantic authority.
- Shared assets and logs remain split by responsibility: bases owns asset
  identity and diagnostic records; rendering owns GPU residency; host and JS
  adapters own platform reads and diagnostic sinks.
- RHI is maintained, not extended by a feature checklist. A change starts with
  a real renderer or graph consumer, an exact dependency revision, and a
  bounded integration proof. Existing RHI TODOs and unproved target evidence
  remain visible rather than being silently treated as closed.
- Capture/replay, Canvas 2D/basic text, declarative UI, Blender tooling, and
  JavaScript encapsulation are deferred and undecided. No current principle
  assigns them an architecture, API, owner, dependency, or delivery condition.

## Resource lifetime domains

- Logical assets own identity, typed handles, content generation, loading
  state, logical references, reuse candidates, and CPU-side cache budgets.
  Logical reference loss means “future use is no longer requested”; it never
  proves that a GPU object is safe to destroy.
- Rendering residency maps a logical asset generation and identity facts exposed
  by the selected RHI baseline to a persistent GPU realization. Frame preparation
  resolves it once into a prepared/imported resource; passes do not repeatedly
  query the asset system. A pending realization may be usable by one named Graph
  consumer when an enforceable RHI producer dependency exists; it is not globally
  complete or reusable. Old realizations retire only after their recorded last
  GPU use has terminal completion evidence.
- Per-frame transform, uniform, and bone-palette input has a Frame Data Upload
  lifecycle, separate from persistent residency. Both paths may reuse one lower
  upload mechanism and RHI state/completion authority. GPU skinning, simulation,
  culling, and indirect generation remain Graph passes.
- RenderGraph owns per-frame logical resource versions and usage, dependencies,
  definedness, culling, scheduling, and lifetime analysis. It uses RHI portable
  resource, command, and capability vocabulary directly; graph code never
  accesses backend-private native API. Graph-transient resources are not assets.
  Cross-graph pooling, reuse between distinct logical resources, and memory
  aliasing require their own measured workload and explicit alias-safety model.
- RHI reports physical allocation requirements, realizes and destroys native
  objects when instructed, and executes backend-specific aliasing and
  synchronization instructions. RenderGraph and the renderer decide reuse and
  retirement from their lifetime plan and observed completion evidence. A graph
  resource reaching logical end-of-life does not imply a physical create/drop
  cycle each frame.
- The first renderer/graph integration keeps a cached plan live across two or
  three submitted frames and retires a recycled resource only after the
  completion point that covers its final use. Partial acceptance and indeterminate
  native effects retain all affected resources until RHI provides a terminal
  outcome; old accepted-unknown naming does not remove this obligation.
- Capability floors are explicit. Common raster/resource behavior must not
  acquire storage or compute semantics merely because modern backends provide
  them; unsupported requirements fail with a structured capability result.
- Asset single-flight coordinates one producer for one logical identity but
  does not own source mechanics. File/network/memory reads, decoding,
  cancellation, retry, progress, and source errors belong to caller-owned loading
  orchestration and platform services. A reusable Loader crate remains a
  candidate until a real consumer proves that extraction.

## Documentation and cross-repository truth

- Organization `ROADMAP.md` is the single authority for plan order and
  milestone authorization. An owning repository is authoritative for its public
  contract and recommended usage. The integration artifact owner is
  authoritative for its scripts and locks; a Release manifest is authoritative
  for exact commits, targets, commands, and retained evidence. Until S0 creates
  it, the Rendering plan records the intended artifact and manifest locations;
  a plan never claims those paths already exist.
- Repository READMEs summarize current capability and link to those sources;
  they do not copy the full stage history or become a competing roadmap.
- `ECOSYSTEM_ARCHITECTURE.md` assigns ownership and dependency direction only.
  It does not restate plan sequencing or define RHI, RenderGraph, or renderer API
  semantics. The authoritative RHI contract lives in the independent
  `fluxel-rhi` repository.
- A producer contract change must build and test every affected pinned
  consumer before merge. A cross-repository Release manifest records all
  participating commits/tags, lockfiles, capability facts, and evidence.
- Fast CI catches compile and contract drift. Named real-target evidence remains
  the support oracle and cannot be replaced by a hosted smoke test.

## AI-friendly engineering

- Give each capability one recommended path. Do not retain parallel simple,
  advanced, legacy, backend-specific, and convenience paths for the same work.
- Keep names, module layout, lifecycle terms, and backend structure stable and
  symmetric. An agent should be able to predict where code, tests, examples,
  and documentation belong.
- Prefer small explicit APIs, typed errors, visible state transitions, and
  deterministic behavior. Avoid magic registration, hidden global state,
  stringly typed contracts, implicit resource cloning, and macro-generated
  public surfaces.
- Use the same terms and operation order in code, demos, tests, diagnostics,
  and documentation. Rename consistently instead of retaining speculative
  compatibility aliases.
- Local execution guidance is maintained outside published repositories; public
  repository documents contain durable ownership, usage, and evidence rules.

## Delivery discipline

- Advance one coherent plan milestone at a time: plan, continuous implementation
  of its internal work packages, concentrated verification, one independent
  review, and one retained evidence closure.
- Roadmap milestones state a boundary, the work now authorized, and the
  condition that closes the milestone. Later milestones intentionally become
  coarser; they guide direction but do not authorize implementation or API
  expansion before their own boundary and close conditions are refined.
- Do not skip an earlier closure because a later API is easier to design. Reduce
  the current scope when its full gate cannot yet be proved.
- An explicitly independent target gate may remain open while work with no
  dependency on that target proceeds. This does not close or broaden the open
  target's support claim.
- A change that alters a public contract, ownership boundary, or supported
  target updates the applicable plan and review before implementation.
- Follow [EVIDENCE_POLICY.md](EVIDENCE_POLICY.md) for the strength, frequency,
  and retention of verification evidence.
