# Fluxel Development Principles

These principles govern the Fluxel ecosystem and its four monorepositories:
`fluxel-bases`, `fluxel-rendering`, `fluxel-host`, and `fluxel-jsbridge`. They
are stable decision rules, not a release schedule or a promise to create every
named crate.

Fluxel is a native-first, AI-friendly lightweight rendering runtime with an
approachable scene API. Familiar concepts may inform usability studies, but no
third-party API or behavioral compatibility is implied.

## Native-first semantics

- Rust rendering and host contracts are authoritative. JavaScript, WASM, FFI,
  and declarative UI layers adapt those contracts; they do not define renderer
  or host core semantics.
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
- The JavaScript SDK is replaceable: Rust, C#, Lua, or another language may
  provide an SDK over the same lower contracts. Compatibility work requires an
  explicit new decision; it cannot be inferred from familiar scene names.

## Purpose-built UI

Fluxel UI grows from demonstrated HUD, settings, and application-screen needs
after Canvas, text, input, and host lifecycles are proven.

- Fluxel does not target third-party UI, DOM, or CSS compatibility.
- Templates, reactivity, component lifecycles, and plugin behavior are not
  native-core contracts.
- Declarative ideas may be studied later, but any retained surface follows
  Fluxel ownership and resource-lifetime semantics.

## Demo-driven scope and ownership

- The next visible, testable demo closure determines implementation order.
  Repository ownership determines where code belongs, not when a crate must
  exist.
- [ECOSYSTEM_ARCHITECTURE.md](ECOSYSTEM_ARCHITECTURE.md) assigns each capability
  to one of the four monorepositories. It reserves candidate crate boundaries
  but does not authorize extraction or change roadmap order.
- Reuse and extend the current demo. Extract a crate only after the vertical
  slice demonstrates an independently changing responsibility.
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
- `fluxel-rendering` owns RHI, RenderGraph, renderer submission, rendering-side
  asset residency, Canvas, UI, shaders, and native/WASM rendering packaging. It
  has no main loop, I/O, input collection, storage, networking, audio, or
  video.
- `fluxel-host` owns native platform lifecycle and implementations for window
  and surface handling, input and time acquisition, filesystem, storage,
  networking, audio, video, native diagnostic sinks, and executable packaging.
  It may depend on rendering and bases; neither may depend on host.
- A host window supplies standard window/display handles; rendering may accept
  those traits from Host or another standards-compatible provider but never
  imports Host. RHI owns the native surface, swapchain, presentation, and GPU
  synchronization derived from the handle.
- `fluxel-jsbridge` owns the default JavaScript SDK plus browser, mini-game, and
  native adapters. It composes lower contracts for developers without becoming
  their semantic authority.
- Shared assets and logs remain split by responsibility: bases owns asset
  identity and diagnostic records; rendering owns GPU residency; host and JS
  adapters own platform reads and diagnostic sinks.

## Resource lifetime domains

- Logical assets own identity, typed handles, content generation, loading
  state, logical references, reuse candidates, and CPU-side cache budgets.
  Logical reference loss means “future use is no longer requested”; it never
  proves that a GPU object is safe to destroy.
- Rendering residency maps a logical asset generation and device generation to
  a persistent GPU realization. Frame preparation resolves it once into a
  prepared/imported resource; passes do not repeatedly query the asset system.
  Pending realizations become committed only after their upload completion is
  known, and old realizations retire only after their recorded last GPU use.
- RenderGraph owns per-frame virtual resource usage, dependencies, states,
  barriers, and last-use facts. Graph-transient resources are not assets.
  Repeated execution of one stable compiled graph should reuse its compatible
  physical realization after GPU-safe completion. Cross-graph pooling, reuse
  between distinct logical resources, and memory aliasing require their own
  measured workload and explicit alias-safety model.
- RHI owns physical creation, reuse, and destruction. A graph resource reaching
  logical end-of-life does not imply a physical create/drop cycle each frame.
- Capability floors are explicit. Common raster/resource behavior must not
  acquire storage or compute semantics merely because modern backends provide
  them; unsupported requirements fail with a structured capability result.
- Asset single-flight coordinates one producer for one logical identity but
  does not own source mechanics. File/network/memory reads, decoding,
  cancellation, retry, progress, and source errors belong to Loader.

## Documentation and cross-repository truth

- Organization `ROADMAP.md` is authoritative for stage authorization and the
  supported-target ledger. An owning repository is authoritative for its
  public contract and recommended usage. The integration artifact owner is
  authoritative for its scripts and locks; a Release manifest is authoritative
  for exact commits, targets, commands, and retained evidence.
- Repository READMEs summarize current capability and link to those sources;
  they do not copy the full stage history or become a competing roadmap.
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

- Advance one coherent release series at a time: plan, continuous implementation
  of its internal work packages, concentrated verification, one independent
  review, and one retained release-evidence closure.
- Roadmap stages state a boundary, the work now authorized, and the condition
  that closes the stage. Later stages intentionally become coarser; they guide
  direction but do not authorize implementation or API expansion before their
  own boundary and close conditions are refined.
- Do not skip an earlier closure because a later API is easier to design. Reduce
  the current scope when its full gate cannot yet be proved.
- An explicitly independent target gate may remain open while work with no
  dependency on that target proceeds. This does not close or broaden the open
  target's support claim.
- A change that alters a public contract, ownership boundary, or supported
  target updates the applicable plan and review before implementation.
- Follow [EVIDENCE_POLICY.md](EVIDENCE_POLICY.md) for the strength, frequency,
  and retention of verification evidence.
