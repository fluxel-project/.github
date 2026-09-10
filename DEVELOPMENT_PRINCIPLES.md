# Fluxel Development Principles

These principles govern the ecosystem and its repositories. They are stable
decision rules, not a release schedule or a promise to create every named
library.

## Native-first semantics

- Rust APIs and native runtime behavior are authoritative. JavaScript, WASM,
  FFI, and declarative UI layers adapt those semantics; they do not define the
  renderer or runtime core.
- Native-first does not mean desktop-only. Portable behavior is proved on each
  target, while backend-specific objects remain behind native boundaries.
- Prefer explicit ownership, borrowing, typed handles, lifecycle states, and
  structured errors. Make invalid states unrepresentable when the Rust API can
  do so.

## Three-like, not Three.js-compatible

Three.js is an API reference, demo source, usability comparison, and
performance workload. It is not a compatibility target.

- Fluxel does not target complete Three.js API or behavioral compatibility.
- Fluxel is not a backend for Three.js and will not provide a
  `threejs-native` runtime.
- `Scene`, `Camera`, `Mesh`, `Geometry`, `Material`, and `Transform` may be
  adopted only when they are natural Rust concepts.
- Three.js inheritance, plugin contracts, implicit global state, material
  breadth, and `Three*` compatibility types must not enter the native core.
- A future JS or TS facade must remain removable and replaceable. Compatibility
  work requires an explicit new decision; it cannot be inferred from
  “Three-like.”

## Purpose-built UI, not Vue-native

Fluxel UI grows from demonstrated HUD, settings, and application-screen needs
after Canvas, text, input, and runtime lifecycles are proven.

- Fluxel does not target Vue, DOM, or CSS compatibility.
- Fluxel will not build a `vue-native` clone.
- Templates, reactivity, component lifecycles, and plugin behavior are not
  native-core contracts.
- Declarative ideas may be studied later, but any retained or declarative
  surface follows Fluxel ownership and resource-lifetime semantics.

## Demo-driven scope and ownership

- The next visible, testable demo closure determines implementation order.
  Layering determines code ownership, not when a crate must exist.
- Reuse and extend the current demo. Extract a crate only after the vertical
  slice demonstrates an independently changing responsibility.
- Do not broaden a public API for a hypothetical target. Record the missing
  case and wait for a stage that can prove it on that target.
- Optimize only after a representative baseline and profile identify a concrete
  cost. Feature count, backend count, and abstraction count are not progress.

## Ownership boundaries

- `fluxel-rhi` owns native GPU objects, barriers, commands, submission,
  readback, and backend realization.
- `fluxel-rendergraph` owns declarations, resource dependencies, logical
  synchronization requirements, portable execution plans, and validation. It
  does not own native synchronization realization.
- `fluxel-renderer` owns 3D submission, ordering, grouping, and lowering to
  RenderGraph. It consumes prepared resources and does not own durable asset
  identity or host lifecycle.
- `fluxel-loader` obtains and decodes CPU data. `fluxel-assets` owns identity,
  typed handles, generations, cross-frame references, residency, reuse, and
  safe release.

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
- Each participating repository's `AGENTS.md` states ownership, normal edit
  locations, focused verification commands, and required real-target evidence.

## Delivery discipline

- Advance one small release at a time: plan, API review, implementation,
  focused verification, independent implementation review, and retained
  evidence.
- Roadmap stages state a boundary, the work now authorized, and the condition
  that closes the stage. Later stages intentionally become coarser; they guide
  direction but do not authorize implementation or API expansion before their
  own boundary and close conditions are refined.
- Do not skip an earlier closure because a later API is easier to design. Reduce
  the current scope when its full gate cannot yet be proved.
- A change that alters a public contract, ownership boundary, or supported
  target updates the applicable plan and review before implementation.
- Follow [EVIDENCE_POLICY.md](EVIDENCE_POLICY.md) for the strength, frequency,
  and retention of verification evidence.
