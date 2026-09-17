# Stage 3F / 0.16 — One common RHI layer, five backends

**Status:** Planned and confirmed. This document is the 0.16 execution contract.

**Owning repository:** `fluxel-rendering`. `fluxel-jsbridge` participates only
through its existing pinned browser consumers, which must build and pass before
the wasm entry-point change lands.

## 0. Confirmed decisions

| ID | Decision |
| --- | --- |
| A | **Replace, do not wrap, `wgpu-hal`.** DX12, Vulkan and Metal are implemented by this workspace on `windows`, `ash` and `objc2-metal`. Exit condition: `cargo tree -p fluxel-rhi` contains no `wgpu-hal`, `wgpu-types` or `fluxel-wgpu-hal`. |
| B | **The three native backends converge in 0.16** — DX12, Vulkan and Metal. The GL family and the browser WebGPU adapter are **0.17**, not 0.16. The common layer is still designed to the five-backend standard (no backend mechanism may leak into it), but 0.16 does not require five backends to be migrated onto it. Section 20 records the re-cut and why. |
| C | **Metal evidence is a deferred local gate.** Metal is implemented here, compiled for the Apple target and covered by shared contract tests plus the mock. The real Metal run happens once, after push, on the owner's macOS machine. Local validation targets are Windows (DX12/Vulkan), Chrome (WebGL2/WebGPU), and an Android emulator as auxiliary GLES evidence. |
| D | Compressed-texture closure rides in 0.16. |
| E | `gpu-allocator` (+ `range-alloc` for DX12) is reused; no in-repo suballocator. |
| F | DX12 shaders: caller-supplied DXIL is passthrough, and Naga `hlsl-out` + dynamically loaded DXC (`dxcompiler.dll`) is the fallback when a source is not DXIL. DXC's absence is a structured error, never a process-level dependency. |
| G | The platform layer is a private module tree under `crates/rhi/src/`, not a public crate and not part of the semver surface, recorded by ADR-0012. |

## 1. Goal, and what "converge" means

0.16 delivers one RHI-semantics layer that all five backends implement, and three
new native backend implementations that replace the borrowed ones.

```text
                  RenderGraph / common RHI contract
              (ExecutionBackend, DeviceCapabilities, access states)
                                |
              common — RHI-semantics layer, shared by all five:
              resource identity and generation, lifetime and lease-before-completion,
              submission and quarantine, presentation generations, capability
              lowering, structured errors, the backend trait contract
        ┌────────────┬────────────┬────────────┬────────────┐
      DX12         Vulkan       Metal      WebGPU      GL family
    (windows)       (ash)    (objc2-metal) (web-sys)   (glow + WGL/EGL,
                                                        state machine, extensions)
     └──────────── native/ ─────────────┘
```

**"Converge" means the layer above, not a union of GPU APIs.** The contract in
`common` is stated in the semantics RenderGraph and the RHI already require:
resources with usage and access state, passes with declared attachments, draw /
dispatch / copy at recipe granularity, completion, presentation, and capability
facts. It contains **no** descriptor set, no pipeline barrier, no root signature
and no encoder-hazard type. Those are each backend's own business:

- DX12 records with `ResourceBarrier`, root signatures and descriptor heaps;
- Vulkan records with `vkCmdPipelineBarrier`, descriptor sets and command buffers;
- Metal records with encoder boundaries and its own hazard tracking;
- the GL family keeps its desired/applied/unknown state machine and reconciles
  dirty state, because that is what a stateful API needs and ADR-0011 already
  paid for that design.

This is the difference between converging on the RHI's semantics and building "a
lowest-common-denominator Vulkan". Convergence that costs the GL family a
simulated descriptor model, or costs the native family an abstraction over
barriers, has overshot and is out of scope.

The earlier five-platform plan's error was treating the two families as one
implementation layer. The re-scoped proposal's error was leaving the GL and
WebGPU paths outside the common layer. Neither is repeated: one common layer, two
implementation idioms, five backends.

## 2. Work packages

Ordered so that shared code is extracted from real duplication rather than
designed up front (principle in section 3).

- **W1 — baseline, vocabulary and skeleton.** Record the current DX12/Vulkan
  behavior as the frozen oracle (exact gate commands, test targets and expected
  results) before anything moves. Then place the target tree together with the
  two pieces that cannot sensibly be written twice: `crates/rhi/src/common/`
  holding (a) the value vocabulary — formats including compressed formats,
  usages, access states, descriptors, limits, features, sampler and view
  descriptors — and (b) the backend trait contract stated at RHI-semantics level
  as section 1 defines it; plus `crates/rhi/src/native/{dx12,vulkan,metal}/` and a
  fail-closed implementation where `imp/stub.rs` is today.

  Two deliberate changes from the first draft of this package:

  1. **The `webgl2/` → `gl_family/` rename is not in W1.** It is mechanical path
     churn across 176 files whose only justification is the GL convergence.
     `scripts/check_gl_architecture.py` also treats a missing
     `crates/rhi/src/webgl2` tree as a *successful no-op*, so renaming the
     directory without updating that constant would silently disable the boundary
     check that ADR-0011 depends on. The rename therefore lands in W6, next to the
     work that proves the boundary still holds, with the script updated in the
     same change.
  2. **The vocabulary and the contract belong to W1; the shared behavior belongs
     to W4.** Section 3's rule forbids designing shared *abstractions* before two
     implementations need them; it does not ask for two incompatible copies of a
     data model. W2 and W3 therefore target `common`'s vocabulary and contract
     from their first line, and W4 extracts only the behavior that both have
     actually written.

  *Closes when:* the vocabulary and the contract compile with no platform
  dependency, the mock implements the contract, nothing has moved semantically,
  and every existing gate still passes unchanged.

- **W2 — Vulkan end to end.** Instance and validation-layer probe, adapter
  enumeration, device open with required limits, buffers/textures/views/samplers,
  bind groups and pipeline layouts, SPIR-V shader modules, compute and raster
  pipelines, one command encoder with explicit transitions, copies, fences,
  submit, and the surface path. Written directly on `ash`, with `gpu-allocator`.
  *Closes when:* the frozen W1 oracle passes on Vulkan with `Validation::Required`
  and programmatically empty diagnostics on the named Windows board.

- **W3 — DX12 end to end.** The same surface on `windows` + `gpu-allocator` /
  `range-alloc`, including the information-queue probe, the
  `D3D12_COMPARISON_FUNC_NONE` lowering for a null comparison sampler (today
  obtained from a vendored patch and now owned explicitly), and the decision-F
  shader path.
  *Closes when:* the same oracle passes on DX12 under `Validation::Required`.

- **W4 — extract the shared behavior into `common`.** W1 placed the vocabulary
  and the contract; W4 extracts the behavior the two implementations actually
  repeat: identity and generation, lifetime and lease-before-completion,
  submission and queue serialization, completion and quarantine, presentation
  generations, capability lowering, and the structured error taxonomy. Every
  abstraction must be justified line by line by code that now exists in both
  backends; anything used by only one backend returns to that backend.
  *Closes when:* both backends are expressed over the shared behavior and the
  oracle still passes on both.

- **W5 — Metal.** The third implementation on `objc2-metal` + `block2` +
  `objc2-quartz-core`, expected to be largely mechanical once W4 exists. A
  headless device plus offscreen execution is the required half; a
  `CAMetalLayer` surface only if it costs little.
  *Closes when:* it compiles for the Apple target, passes the shared contract
  tests and the mock, and the deferred macOS run in decision C passes the same
  workload as DX12 and Vulkan.

- **W6 — the GL family converges onto `common`.** Its implementation keeps
  `api` + `state`; its `compat` layer becomes the GL implementation of the
  `common` contract, and the capability lowering duplicated in
  `compat/capabilities` is replaced by the shared one. Anything `common` requires
  beyond the profile's core is served by a typed, proved extension route across
  WebGL2, GLES 3.x and desktop GL 4.x, and refused by name otherwise.
  *Closes when:* the desktop-GL, GLES and WebGL2 profiles pass the shared
  contract tests, and the preserved semantics in section 4 are re-proved under
  the shared lowering.

- **W7 — the browser WebGPU adapter converges onto `common`.** Its device loss,
  recovery, canvas epoch, ticket and disposal semantics are preserved exactly;
  its position in the architecture changes.
  *Closes when:* the named Chrome WebGPU lifecycle gate passes unchanged.

- **W8a — portable compressed-format vocabulary.** New `TextureFormat` variants
  in `fluxel-rendergraph`, the matching `TextureFormatCapabilities` rows, and the
  per-format validation rules from ADR-0011, with the renderer, GL family and
  browser consumers updated. Required by 9.2: no backend may claim a compressed
  format a graph cannot name.
  *Closes when:* a graph can declare and compile against every claimed format,
  and the consumers that switch on `TextureFormat` are exhaustive again.

- **W8 — compressed-texture closure.** Format-exact BC1–BC7, ETC2/EAC and ASTC:
  block geometry, encoded byte size, color space and the extension-or-core
  evidence validated per format before upload, with no render, blend or storage
  support inherited from an unrelated family capability. PVRTC and ETC1 stay
  excluded.
  *Closes when:* each claimed format uploads and samples on at least one backend
  with exact readback, and each unclaimed format is refused by name.

- **W9 — unified capability lowering and evidence.** One lowering from each
  backend's discovery snapshot to `fluxel_rendergraph::DeviceCapabilities`,
  replacing the GL-only lowering and the native derivation. Fail-closed rules are
  preserved: a transient-reuse row stays false where nothing pools, and an
  unobserved fact is reported as absent rather than defaulted.
  *Closes when:* one evidence set binds the six conditions in section 5 to one
  source revision.

- **W10 — dependency and documentation closure.** Delete `crates/wgpu-hal`,
  `wgpu-hal`, `wgpu-types` and `ordered-float`; write ADR-0012 (the common layer)
  and ADR-0013 (superseding the GL-only part of ADR-0011); update
  `design-rhi.md`, `crates/rhi/README.md` and the workspace README capability
  tables.
  *Closes when:* the dependency check passes in every feature combination and no
  document still describes the native layer as borrowed.

## 3. The two rules that keep this from becoming a rewrite for its own sake

1. **Shared code is extracted, never designed up front.** This is why W2 and W3
   each build a complete backend before W4 exists, and why W5 follows W4 rather
   than joining it.
2. **No backend-specific mechanism may be lowered into `common` to make a
   backend easier.** If a backend wants an abstraction in `common`, two backends
   must already need it, and the need must be visible in code that exists.

## 4. Semantics the rewrite must preserve

These are deliberate refusals rather than features, which is exactly why a fresh
implementation is tempted to be permissive:

| Behavior | Where it lives today |
| --- | --- |
| Texture copy has two routes and five named pre-copy refusals (feedback loop, multisampled end, missing usage, unproved format-copy fact, non-2D-rect) | `webgl2/compat/device/transfer.rs`, `api/copy.rs` |
| A pass admits exactly one colour attachment at index 0 and no depth-stencil attachment | `webgl2/compat/device/pass.rs` (`ONE_COLOUR_TARGET`) |
| The surface format list states one format, derived from observed component widths; sRGB is deliberately not claimed | `webgl2/compat/capabilities/mod.rs` |
| Multiview stays closed while its attach path is unproved, on every profile | `webgl2/api/{tests/multiview.rs,browser/discovery.rs,native/discovery.rs}` |
| Capability lowering adds only what discovery proved; every other field keeps its rejecting value | `webgl2/compat/capabilities/mod.rs` |
| Accepted-unknown work quarantines ownership and never releases early | `documents/adr/0004`, `imp/submission.rs` |
| An unpresented Vulkan acquire poisons and quarantines the surface instead of guessing reuse | `documents/design-rhi.md` |
| `Validation::Required` is fail-closed and positively verified before a device is returned | `imp/device_open.rs` |
| DX12 lowers a null comparison sampler to `COMPARISON_FUNC_NONE` | vendored patch, `FLUXEL-PATCH.md` |

## 5. Definition of done

1. **No regression.** DX12 and Vulkan keep their current observable behavior:
   the same fixed recipes, structured errors and lifecycle.
2. **Metal runs the same workload** as DX12 and Vulkan, under decision C's route.
3. **Shared code comes from real duplication** (section 3).
4. **No leakage.** No backend-specific object, handle or synchronization
   primitive appears in the common contract or the public API; raw handles stay
   inside `native/<backend>` and inside `unsafe`.
5. **One evidence set, five backends.** The same lifecycle, completion, resource
   and visual evidence passes on DX12, Vulkan, Metal, the GL-family profiles and
   the browser WebGPU target, each on its own named environment.
6. **No borrowed dependency.** The `cargo tree` check in decision A passes in
   every feature combination.

## 6. Non-goals

- **No new rendering capability.** Multi-draw, indirect draw/dispatch, base
  vertex, first instance, stencil, polygon mode, independent blend, multiview and
  a depth-tested recipe are out of scope; Appendix A records the triage and where
  each belongs.
- **No public API expansion** beyond the confirmed `Backend` / `Device::open`
  shape. ADR-0006 and ADR-0007 continue to hold: no general shader, pipeline,
  descriptor or binding API becomes public.
- **No multi-queue scheduling, parallel recording, recording caches, transient
  aliasing or memory-pool policy.**
- **No mobile work.** Section 7 states precisely what is and is not front-loaded.
- **No new supported target** beyond what section 5's evidence proves.

## 7. What this buys for mobile — stated precisely

Completing 0.16 front-loads most of the *code structure* that Android Vulkan and
iOS Metal will need, because those platforms reuse the same backend
implementations through different loaders, surfaces and memory heaps. It does
**not** front-load their *evidence*: ADR-0008 requires platform paths to execute
natively, and Android's Vulkan is not Windows' Vulkan in loader, extension set,
memory heaps or surface. Mobile stages still owe their own Host lifecycle,
surface, input and foreground/background work. An Android emulator is auxiliary
evidence only; ADR-0011 already records that a physical, non-translation GLES
device remains required for a portable GLES claim.

## 8. Risks

1. **Scope.** Three native backends plus two convergences is the largest single
   rendering change in the ecosystem so far. W2 and W3 are the schedule.
2. **Evidence reset.** 0.12–0.15 evidence is bound to the borrowed
   implementation. It stays published as history and is not reused; every gate is
   re-run on the new implementation.
3. **GL regression risk.** Replacing `compat`'s lowering is where section 4's
   refusals are most likely to be quietly lost. W6's exit condition is explicitly
   those refusals, not the contract tests alone.
4. **Metal blindness.** Until the owner's macOS run, Metal is compile- and
   mock-verified only. Its support claim stays unstated until then.

## 9. W1 design note: what `common` owns, and what it consumes

### 9.1 Vocabulary ownership

`fluxel-rendergraph` already owns the *portable* vocabulary, and `common` must
consume it rather than restate it: `TextureFormat`, `IndexFormat`, `Extent3d`,
`TextureDimension`, `TextureDesc`, `BufferDesc`, `ResourceAccessState`,
`TextureFormatCapabilities`, `DeviceCapabilities`, `DeviceLimits`, `LoadOp` /
`StoreOp`, `Viewport`, `ScissorRect`, and the read/write use enums in
`rendergraph/src/access.rs`.

`common` therefore owns only the vocabulary the portable contract deliberately
does not model — the facts a backend needs and a graph must never depend on:

- vertex formats, step modes and vertex-buffer layouts;
- sampler descriptors (address, filter, mipmap filter, compare, LOD);
- bind-group layout entries, binding types and shader visibility;
- pipeline state (primitive, multisample, depth-stencil, colour target);
- attachment ops and the native texture-format capability bits (linear
  filtering, per-sample-count support, storage read/write, colour and
  depth-stencil attachment, copy source and destination);
- copy footprints (`Rect`, `CopyExtent`, texel-copy layout, copy base);
- surface configuration (present mode, composite alpha, format, extent);
- adapter identity and the native limit and feature sets.

This split is a hard rule, not a preference: if a value would let a graph
depend on a backend fact, it belongs in `rendergraph`, not in `common`.

### 9.2 Finding — compressed formats need a portable vocabulary change

`fluxel_rendergraph::TextureFormat` has exactly five variants today:
`Rgba8Unorm`, `Rgba8UnormSrgb`, `Bgra8Unorm`, `Rgba16Float`, `Depth32Float`.
There is no compressed format in the portable contract at all.

W8 therefore is **not** a backend-only package. A claimed compressed format must
be nameable by a graph, compile against a capability row, and reach a resource
descriptor, which means new `TextureFormat` variants in `fluxel-rendergraph`
(a published crate whose `TextureFormat` is consumed by the renderer, the GL
family and the browser adapters) plus the matching `TextureFormatCapabilities`
rows and the per-format validation rules of ADR-0011.

This also means there are currently **three** format vocabularies: the portable
one in `rendergraph`, `GlFormat` in the GL family, and `wgt::TextureFormat` in
the native path. W9's unified lowering has to reconcile all three, and W8 has to
extend the portable one deliberately rather than letting a backend add its own.

Consequence for the work order: W8 gains a prerequisite package, **W8a —
portable compressed-format vocabulary** in `fluxel-rendergraph` (new variants,
capability rows, validation, and the renderer/GL/browser consumers updated and
tested), before any backend claims to support them.

### 9.3 Finding — a boundary check that fails open

`scripts/check_gl_architecture.py` records `WEBGL_RELATIVE_ROOT =
Path("crates/rhi/src/webgl2")` and documents that a missing tree is a
*successful no-op*. The W1 draft of this plan renamed that directory. Doing so
without moving the constant would have silently disabled the check that enforces
ADR-0011's boundaries, with a green CI run. The rename moved to W6 for that
reason, and any future directory move under `crates/rhi/src` must audit the
scripts for path-keyed checks before it lands.

### 9.4 Frozen oracle baseline (recorded 0.16 W1)

On the current tree, Windows MSVC, rustc 1.87:

- `cargo +1.87.0 test --workspace --all-targets --all-features --locked --no-run`
  — compiles clean in 32.28s.
- `cargo +1.87.0 test --workspace --all-targets --all-features --locked` — all
  suites green; `fluxel-rhi` runs 563 cases with 524 passed and 39 ignored
  (the ignored set is the real-GPU conformance fixture), and the renderer,
  rendergraph and wasm suites pass.

These two commands are the regression oracle for every later package. The
real-GPU half is `scripts/conformance.ps1` and is re-run at W2, W3, W5 and W9.

### 9.5 API rule — capability domains are traits, never optional methods

`common` is not one wide interface with methods a backend may decline. It is a
small required floor plus **one trait per optional domain**, and a domain's
availability is a property of the backend *type*:

- the floor carries only what all five backends can do;
- compute, storage buffers, storage images, indirect draw, indirect dispatch,
  multi-draw, multiview, occlusion / elapsed / timestamp queries, base vertex,
  first instance and anisotropic filtering are each their own trait, implemented
  by the backends that have them;
- a caller that needs a domain is bounded on that domain's trait, so a backend
  without it is **structurally** unable to be asked, not answering a run-time
  refusal;
- `common::caps::Capability` has exactly one row per domain trait, so the render
  graph asks one question about a device and receives an answer that names a
  batch of API.

This is the GL family's existing shape promoted, not replaced: `GlStateBackend`
is its floor, `GlOptionalComputeBackend` and `GlOptionalIndirectBackend` are
domains, and the `NoCompute` / `WithCompute` witness types make the choice
compile-time. Why the witness must be a type parameter is mechanical and worth
keeping: a trait has one impl block per type (E0119), and a method cannot be
stricter than its impl's own bounds (E0276), so the optional bound has nowhere
else to live.

Two refusals stay distinct, because collapsing them is what produces a
lowest-common-denominator API:

1. **The backend type has no such domain** — the vocabulary is absent, so no call
   site could have been written.
2. **The backend has the domain and this context did not prove it** — the
   vocabulary exists, the row is unproved, and the refusal happens before any
   object, extension or command side effect.

Prerequisite for every domain trait: it must correspond to a row whose evidence a
backend can actually record. A domain whose row no backend can prove is a domain
this plan does not add.

### 9.6 W1 progress

Landed: `crates/rhi/src/common/` with its module contract, and `common::caps`
holding the adapter limit set plus the capability ledger, with the
one-row-per-domain rule and the three independent ways a row stays disabled.
Clippy is clean under `-D warnings` and the eight unit tests covering the
ledger's fail-closed directions pass.

Still owed by W1: the value vocabulary (vertex formats, sampler and pipeline
state, attachment operations, per-format capability bits, copy footprints,
surface configuration) and the floor plus domain traits themselves.

### 9.7 Finding — per-format facts must adopt the GL family's shape, not a HAL bitflags set

`GlFormatCapabilities` (`webgl2/api/formats.rs`) is already an *evidence-carrying*
per-format fact rather than a bitset. It is keyed by
`(format, resource_kind, sample_count)`, carries
`CoreGuaranteed | ExtensionAcquired | OperationProbed` evidence, and states eight
independent facts: `sampled`, `filterable`, `renderable`, `blendable`,
`storage_read`, `storage_write`, `copy_source`, `copy_destination`. Its
`record` already **enforces** ADR-0011's compressed-format rule: a compressed
format must be texture-only and single-sample, and may not claim renderable,
blendable, storage or copy support.

The native path currently represents the same facts as
`wgpu_hal::TextureFormatCapabilities` bitflags plus predicates in
`imp/lowering.rs`. Those are two different *shapes*, not two spellings of one
shape, so W9's "unified lowering" cannot reconcile them by mapping names.

Consequences:

- `common` promotes the evidence-carrying table as the one per-format fact shape.
  The native backends fill that table from their own discovery instead of
  exposing a bitflags set, and the GL family stops owning the only copy of it.
- W8a must add what the compressed rule *needs*, not merely names: a
  compressed-format predicate and block geometry, so the "a compressed format
  claims nothing but sampling and copying" rule is enforced centrally instead of
  once per backend.
- `resource_kind` is GL-specific, because a renderbuffer is not a texture. The
  common table keys on `(format, sample_count)`; the GL backend keeps its
  renderbuffer rows as an internal detail mapping onto the same texture-shaped
  facts, and W6 records that mapping.
- This is also why W1 can land the vocabulary before the table: the vertex-input
  contract is independent of how format facts are stored, while the table's first
  row depends on W8a's vocabulary decision.

### 9.8 W1 progress (second increment)

Landed this round: `common::vertex` — the closed vertex-input contract
(`VertexFormat`, `VertexStepMode`, `VertexBufferLayout`, `VertexAttribute`,
`VertexLayout`) with self-contained layout validation. The format is deliberately
closed for the same reason the raster recipes are: a caller cannot describe a
stream no retained artifact declares, and a new format arrives only with the
artifact that needs it. The shape mirrors the GL family's existing
`GlVertexBufferLayout` / `GlVertexAttribute` pair so W6 converges without a
translation step.

Deliberately **not** started: the floor and domain traits. They are gated on the
four open design questions in section 10.

## 10. API-design questions (resolved by section 11)

These are the decisions left open by the API discussion and they gate W1's last
piece, W2 and W3. Recorded here so the answers are not lost:

1. **Floor boundary.** Proposal: the floor is identity and generation, resources
   (buffer / texture / view / sampler), pass brackets (raster + copy), bind
   groups, single-instance non-indexed and indexed draws, copy commands,
   completion, presentation, capability query and fixed-recipe selection.
   Compute, storage buffers, storage images, indirect draw and dispatch,
   multi-draw, multiview, the three query kinds, base vertex, first instance and
   anisotropic filtering are domains. Test for the floor: **a method belongs to
   the floor exactly when all five backends can give a real implementation rather
   than a refusal body.**
2. **Resource-role rows.** Are `storage buffer` / `storage image` (resource
   roles, which gate binding layouts and usage checks) domain *traits*, or ledger
   rows only? The `Capability` enum currently mixes resource roles with command
   domains.
3. **Where the graph's ease comes from.** Either (a) the graph is authored
   device-independently and the compiler checks `DeviceCapabilities` once,
   fail-closed — portable, but an author can write a graph no device can run; or
   (b) authoring carries a domain witness so an unexecutable graph cannot be
   constructed — but the graph then carries device information and loses
   portability. Section 1's rule implies (a).
4. **`rendergraph::ExecutionBackend`.** (i) leave it as one trait and keep the
   witness pattern only where a backend type genuinely lacks a domain (today
   only the browser GL provider); (ii) split it into floor + domain traits, which
   is a published-contract change with pinned consumers to update; (iii)
   `common` uses domain traits internally and a generic assembly layer provides
   the fat impl only for backends that implement every domain.

Also recorded as a standing risk: domain traits propagate generic bounds
(`D: ComputeDomain + StorageDomain + ...`). Rust has no stable trait aliases, so
a bundle trait with a blanket impl is the only workaround — and a bundle used
everywhere *becomes* the fat trait again. Rule: a bundle may appear only at the
leaf where a concrete backend reports its own capability set, never inside
`common`.

## 11. API design (confirmed) — capability-oriented RHI

**Principle: unify semantics and ownership, not hardware capability.**

### 11.1 Three layers

```text
                    RHI Base
     identity / resources / lifetime / submission / capability query
                        |
               Capability Families
  Graphics / Compute / StorageBuffer / StorageTexture / Indirect /
  Multiview / AsyncCompute / TransferQueue / ...
                        |
             Backend Implementations
        DX12 / Vulkan / Metal / WebGPU / GL family
```

The base is deliberately tiny. It does **not** carry `draw()`, `dispatch()`,
`storage_texture()` or `graphics_queue()` / `compute_queue()` /
`transfer_queue()`. A base with those methods forces every backend to answer for
a capability it may not have, and the result is a codebase of
`if backend == …`, `if supported …` and `Unsupported`.

**Queue shape is a capability, not a base method.** DX12 and Vulkan expose
several queues, Metal's model differs, WebGPU has its own constraints and WebGL2
is not the same thing at all. Modelling `graphics_queue()` on the base would
compress the entire RHI to the weakest backend's queue model before anything
profiled it, and would make a cross-queue schedule impossible to express later.

### 11.2 A family is vocabulary, not permission

This is the correction that matters most, and it supersedes the earlier
"type fact versus value fact" framing in this document: **do not equate "the
trait is implemented" with "the hardware supports it."** Capability varies below
the backend:

```text
Vulkan backend
  |- GPU A: storage texture, every format
  |- GPU B: storage texture, some formats
  \- GPU C: no storage texture
```

So a family trait says *how* a batch of API is asked for; whether this device can
serve it is a separate, per-adapter fact established by discovery. The flow is
negotiation, not a trait check:

```text
pass requirement        -- declared in capability terms
        |
device capabilities     -- discovery, per adapter
        |
satisfied -> typed execution plan -> execution (no further checks)
unmet     -> UnsupportedCapability
```

A proven negotiation yields a handle, and the execution phase does not ask
`if device.supports_compute()` again. That is what keeps capability branching out
of the hot path and out of every call site.

Two refusals still stay distinct, now stated precisely:

1. **The backend cannot serve the family at all.** The vocabulary is absent, so
   no call site could have been written. Answered by a trait bound.
2. **The backend serves the family and this device did not prove it.** Answered by
   a value, and the failure names which of the ledger's conditions it was: never
   examined, no route, limits unsatisfied, probe failed, or probe not run.

### 11.3 Requirements are stated in capability terms, never backend terms

A node requires `Graphics + StorageBuffer + Indirect`, never `Vulkan`. That is
what keeps one compiled graph executable on every device that can serve it, and
what lets a RenderGraph emit a cross-queue schedule where an asynchronous-compute
family exists and a single-queue schedule where it does not — instead of the RHI
pre-emptively flattening every backend.

RenderGraph remains the layer that *composes* an execution plan from available
capabilities; the RHI's job is only to describe which families exist and how to
obtain them safely. New hardware capability can therefore be opened on the
backend that has it — mesh shaders, descriptor indexing, timeline
synchronisation, sparse resources — without waiting for a Metal equivalent,
because a family Metal lacks is simply a family Metal does not implement and a
row its devices do not prove.

### 11.4 What this changes in the packages

- W1's remaining piece — the base and the family traits — follows 11.1: the base
  is identity, resources, lifetime, submission and capability query; `Graphics` is
  a family rather than part of the base (it happens to be universally available
  across today's five backends, which is a fact about them and not a rule the
  base may depend on).
- Presentation is a candidate family on the same reasoning, since a headless
  device has no surface. It is not added until a consumer needs the distinction.
- The `AsyncCompute` and `TransferQueue` rows exist in the ledger and are simply
  unproved today, which is the honest representation of a single-queue
  implementation: the vocabulary exists, nothing has established it on a device.
- Capability is captured per device at open time, so a device is the negotiation
  input and the ledger is a device fact rather than a backend constant.

### 11.5 Landed

`common::caps` (limits + the one-row-per-domain ledger), `common::api`
(the three-layer statement, the family markers, and
`Requirement::satisfied_by` returning `UnsupportedCapability { row, reason }`),
`common::api::negotiate` (the negotiation hinge: `CapabilitySource`,
`Provides<F>` with its handle type, and `require`), and `common::vertex`.

`require` joins the design's two halves in one place and is deliberately ordered:
it consults the ledger **before** asking the backend for a handle, so a refused
negotiation cannot have had a side effect. A test asserts exactly that by counting
provider calls, because "reject before any side effect" is otherwise an easy
promise to break silently. A handle borrows the device, so it cannot outlive the
ledger that justified it and a replacement device generation cannot revive it.

Clippy is clean under `-D warnings`; 24 unit tests cover the ledger's fail-closed
directions, every distinct requirement failure, and the ordering rule above.

Not yet landed from W1: the base itself (identity, resources, lifetime,
submission, capability query) and the family *traits* (`GraphicsApi` first). Those
now have their shape fixed by section 11 and are the next piece.

### 11.6 Base progress

The base's first piece landed: `common::base::{mod,stamp}`. The module states the
membership test for the base -- **a base item is something all five backends must
agree on for Fluxel's own semantics to hold** -- and the table of what is excluded
and why (`draw`/`dispatch`/`storage_texture` to families, every queue-shape
accessor to capability rows).

`DeviceStamp` is `(identity, generation)`, promoted because two implementations
already need exactly that pair: the native path stamps owned resources with
`fluxel_rendergraph::PhysicalResourceIdentity` over one opened device, and the GL
family stamps ids with a context stamp of device identity plus context epoch. One
fact that is easy to get wrong is encoded in `verify`: **a device identity alone
cannot answer "was this object created before this device was replaced?"**, which
is the question a recovered WebGPU device and a restored GL context both make
unavoidable. The failure is returned as a value, and `ForeignDevice` is checked
before `StaleGeneration` because an object from another device is foreign whatever
generation it claims.

Base pieces still owed, in order: submission and quarantine, capability query.

`common::base::lifetime` landed next: `may_release(CompletionStatus)`, which is
ADR-0004's rule in one place. `fluxel_rendergraph::CompletionStatus` already
*documents* the rule ("callers must retain every lease while the status is
unknown") and both the native path and the GL family implement it in their own
retention type, so encoding it once stops a third backend re-deriving it.

The `#[non_exhaustive]` attribute on that foreign enum decides the shape: only the
two terminal variants release, and **every other variant, including one added
upstream later, retains.** A lifetime rule whose unknown case is "hold on to it"
is the right direction, and it is the same fail-closed direction as every
capability fact here.

Deliberately absent: a lease carrier and any `Drop` implementation. A generic
carrier cannot own the drop rule safely, because dropping a non-terminal lease must
hand it to a non-blocking retirement path whose contents are backend-specific (the
native path retires native objects after a fence; the GL family retires browser
objects after a sync object settles). The base fixes the rule that decides
*whether* release is allowed; each backend owns what release costs.

`common::base::submission` landed next: `Disposition`, the ownership half of
ADR-0004 that `lifetime` does not cover. A submission is `Observed` while its
owner polls, `Abandoned` once a retirement path owns it, and `Terminal` once the
leases are released. `Abandoned` is a state rather than a boolean because dropping
a non-terminal submission is an ownership **transfer**, and the retirement path
needs to know it is now the owner; a boolean "is terminal" cannot carry that.
Re-observing a terminal submission is refused as a value, because that is the
use-after-release shape and treating it as idempotent would let a polling loop
that outlived its own retirement keep running with no signal. Terminality is
delegated to `may_release`, so the status rule and the ownership rule cannot
disagree.

Clippy remains clean under `-D warnings`; `common::` now holds 45 tests.

### 11.7 Base complete

All five base items now exist:

| Base item | Where |
| --- | --- |
| identity | `base::stamp::DeviceStamp` |
| resources | `base::resource::{ResourceId, BufferId, TextureId}` |
| lifetime | `base::lifetime::may_release` |
| submission | `base::submission::Disposition` |
| capability query | `api::negotiate::{CapabilitySource, require}` |

What W1 still owes is the family *traits* -- `GraphicsApi` first -- and the point
at which the old facade is migrated is still open (section 11.8).

### 11.9 Family trait conventions (fixed before `GraphicsApi`)

`api::handle::FamilyApi` landed as the supertrait every family handle implements.
It carries one rule that is easy to get wrong: **a handle reports the stamp it was
negotiated on, and callers verify resource ids against that stamp, never against a
freshly read one.** If the device was replaced between negotiation and use, a
re-read returns the *replacement's* stamp and a stale id from the retired
generation would verify successfully -- the exact mixing `DeviceStamp` exists to
prevent. `verify_buffer` / `verify_texture` therefore accept only the handle, so
the one correct source of the stamp is the only input they take. The complementary
test asserts the other half: a handle from the retired generation still accepts
resources of its own generation, which is what lets in-flight work finish instead
of being torn down.

The conventions the remaining family traits follow, fixed here so `GraphicsApi`
can be written in one pass:

1. **Reuse rendergraph's descriptors where they already exist and are generic over
   the resource type.** `RasterPassDescriptor<'_, T>`, `LoadOp`, `StoreOp` and the
   attachment types are already parameterised -- the GL family instantiates them
   with its own texture id -- so a family trait takes
   `&RasterPassDescriptor<'_, TextureId>`. The pass vocabulary is not restated.
2. **Each family declares its own error type.** Backends disagree about what a
   command can fail on, and one shared error would either be too wide for every
   backend or too narrow for one.
3. **A family is bounded on the base**, never the other way round: a family handle
   implements `FamilyApi`, and a family trait may not add a method the base should
   own.
4. **A family adds vocabulary only.** It may not add a permission: whether a device
   can serve the family is the ledger's answer, read once by `require`.

### 11.10 W1 closed

`api::graphics::GraphicsApi` landed as the first family trait, with an end-to-end
contract test that exercises the whole W1 surface in one flow: an unproved family
yields no handle, a proved one yields a handle, the handle's stamp is what a
resource id is verified against, and the full raster bracket records in order.

W1's exit condition, checked:

- the vocabulary and the contract compile with **no platform dependency** -- no
  `ash`, `windows`, `objc2`, `web_sys` or `glow` name appears anywhere under
  `crates/rhi/src/common/`;
- the mock implements the contract -- `MockRasterDevice` implements
  `CapabilitySource`, `Provides<Graphics>` and `GraphicsApi`;
- nothing moved semantically -- the only edit to existing code was adding
  `mod common;` to `lib.rs`;
- every existing gate passes unchanged, so the frozen oracle in section 9.4 still
  holds.

`common/` now holds: `base/{stamp,resource,lifetime,submission}`,
`caps`, `vertex`, and `api/{family,graphics,handle,negotiate}`. Clippy is clean
under `-D warnings` and `common::` holds 52 tests.

W2 (Vulkan) is next and is the stage's main body of work.

## 12. W2 started

The native tree is placed: `crates/rhi/src/native/{mod.rs,vulkan/mod.rs}`, with
`vulkan` gated on its existing Cargo feature so a build without it never sees the
module. `native/mod.rs` states that the family is a *grouping for readers* and not
a shared implementation -- which facts are shared lives in section 3, and a reader
looking for what DX12 and Vulkan do not share should find it in their own module.

`native/vulkan/mod.rs` carries W2's eleven ordered steps (instance and validation
probe, adapter and device open, memory, resources, descriptors and pipelines,
shaders, recording, copies, submission and completion, surface, capability
lowering) plus the two entries from the preserved-semantics table that a fresh
Vulkan implementation is most likely to lose silently: the unpresented-acquire
quarantine and the fail-closed `Validation::Required` probe.

No Vulkan types are written yet, deliberately: writing them before the first real
call path exists is the pre-designed abstraction section 3 forbids.

The first W2-enabling vocabulary landed instead: `common::sampler`, which step 4
and step 5 need and which is pure enough to prove without a device. It states one
of section 4's preserved semantics directly: `SamplerDescriptor::compare` is an
`Option`, `Always` is a real comparison rather than the absent one, and there is no
`compare_or_default`. That is the behavior the vendored DX12 patch exists to obtain
(null comparison must reach the driver as `COMPARISON_FUNC_NONE`, not `ALWAYS`),
and a type whose default was `Always` would reintroduce the bug the patch fixes.

`common::` now holds 56 tests.

Step 1's **decision** half also landed: `native::vulkan::validation::verify_required`,
which takes an exact inventory of layer and instance-extension names and reports
*which* of the two required pieces is missing. Enumerating those names is FFI;
deciding whether they satisfy `Validation::Required` is not, and the separation
means the interesting cases are testable -- a layer name that differs in case, a
prefix of the real name, a missing `VK_EXT_validation_features` -- none of which
are reproducible on a developer's machine by installing things.

The reason stays internal on purpose. `OpenError::ValidationUnavailable` is public
and carries only the backend; widening it changes the public surface, which is a
separate decision, so the caller maps this reason onto the existing error and the
observable behavior is unchanged.

The FFI half of step 1 also landed: `native::vulkan::inventory` loads the loader
through `ash::Entry::load` and reads the instance-level layer and extension names.
It creates nothing, which is what lets `Required` be refused *before* an instance
exists. A name that is not NUL-terminated or not UTF-8 fails the whole enumeration
rather than being skipped, because a skipped name could have been the validation
layer and the probe would then report a clean inventory.

The native modules are gated `all(windows, feature = "vulkan")`, matching where
`ash` is actually declared in the manifest; gating on the feature alone would
break the Linux gate of ADR-0008. Written against the `ash` 0.38 source rather
than its documentation, and it compiled without an iteration.

Still owed by step 1: instance creation with the validation layer and extension
enabled, and mapping `MissingValidation` onto the existing public error.

Step 1 is now **complete** as of `native::vulkan::instance`: `open` loads the
loader, enumerates, verifies when `Required`, and creates the instance with the
validation layer and `VK_EXT_validation_features` enabled (the
`VkValidationFeaturesEXT` chain requests synchronization validation, because
installing that chain is what the extension is used *for* — enabling the name
alone would be a claim with no effect).

The failure that mattered here was caught by review rather than by the compiler:
the enabled names were first built from `REQUIRED_LAYER.as_ptr()`, and
`str::as_ptr` is **not** NUL-terminated, so the loader would have been handed a
name running past its end. The enabled names are now `c"..."` literals with a test
asserting they still spell the names the probe compares, since that drift is the
one thing the type system cannot express here.

Two of the three tests exercise a real loader and a real instance creation on this
machine, so step 1 has been smoke-tested beyond the pure decision it shares with
`validation`.

Still owed by step 1: mapping `InstanceError` onto the public `OpenError`.

Step 2's first half landed as well: `native::vulkan::adapter` enumerates physical
devices, reads one device's properties, and lowers them into the public
`HardwareInfo` plus `common::caps::AdapterLimits`. It creates nothing, so an
adapter can be chosen -- and its absence reported as
`AdapterError::Unavailable { index, available }` -- before any device exists;
`select` is pure and tested.

Three limit rows are deliberately the rejecting value here rather than a guess:
multiview view count and the multi-draw count live in extension structures, and the
PCI bus identifier and driver name live in others. Reading them belongs to the step
that needs them, together with the query that proves the structure is present. The
same reasoning fixes `driver`: the packed version word is decoded with the
specification's standard layout, because claiming a vendor-specific decoding before
writing one would misreport the driver. **These are owed, not satisfied** -- when
this backend becomes live it will report less than the borrowed one until those
structures are read, and that gap is a regression the stage's definition of done
does not allow to ship.

One correction worth recording: the sample-count fact cannot be read from the flag
type's raw representation, because `ash` keeps that field private to its own crate.
The lowering probes the generated constants in descending order through `contains`
instead, which is also why that helper takes flags rather than a mask.

## 14. W2 step 2 complete

`native::vulkan::device` finishes step 2: it selects one queue family by rule,
creates the logical device with one queue and no features or extensions, and owns
both through `Drop`.

`select_queue_family` is pure and is where the interesting cases live: a
transfer-only family reported *first* must not be selected, and no graphics family
at all must refuse rather than select a queue that cannot rasterize. Whether the
chosen family also supports compute is **reported, not required** -- requiring it
would refuse a device that can rasterize, and the decision that needs compute is
the capability row this selection feeds, so a graphics family without compute
leaves the row unproved and the requirement check refuses the graph that needed it.

Step 2 is proven end-to-end on this machine, not only in unit tests: an added
smoke test opens a real instance, enumerates real adapters, reads a real adapter's
facts (device name and texture extent are asserted non-empty), creates a real
`VkDevice`, and drops it. It skips rather than fails on a machine with no Vulkan
adapter, because having no GPU is not what it is testing.

One `ash` 0.38 detail that cost an iteration and is worth recording:
`Instance::create_device` returns the *loaded* `ash::Device`, not a bare
`vk::Device` handle, so the `ash::Device::load` call a reader might expect is
wrong -- and `fp_v1_0()` is consequently not needed here.

## 15. W1 design correction, found by the first real backend

Step 2 also wired `VulkanDevice` onto `common` as the first backend to implement
`CapabilitySource`, and that immediately falsified one of W1's assumptions.

**W1 had `Capability::requires_probe()`: a per-row flag saying whether the row
needed a real command to be proved.** The Vulkan device recorded graphics and
compute with evidence and satisfied limits, and both rows came out *disabled*,
because the probe flag said a draw or dispatch was required and none had run. On an
explicit API that is circular: the RHI would have to dispatch something before it
could be asked to dispatch anything.

The flag cannot be a property of the row, because the two families prove rows
differently:

- the GL family establishes compute by running a dispatch, and its probe outcome is
  the strongest evidence a stateful API offers;
- Vulkan establishes compute by creating a device on a queue family whose flags
  report compute, and no command is needed.

So the requirement moved to the recorder, which is where it belongs: the fact
already carried `OperationProbe`, and `is_enabled` now accepts `Passed` **or**
`NotRequired`, while `Failed` and `NotRun` both leave the row disabled and remain
different sentences. `requires_probe` was deleted rather than kept as a hint, since
a row-level copy of a backend-level decision is exactly the kind of second truth
this plan keeps removing. For the GL family nothing changes: its lowering already
maps "no probe needed" to `NotRequired` and "not run" to `NotRun`.

The runtime failure that surfaced this was `ledger().supports(Compute) == false`
while the queue family reported compute, with the adapter's limits printed and
non-zero -- which is what pointed at the probe rather than the numbers.

## 16. W2 step 3's pure half

`native::vulkan::memory::select` chooses which memory type a resource may be
placed in, which is the part of step 3 that has a rule in it; `gpu-allocator`
performs the suballocation beneath it. The rule has two halves that are easy to
conflate, and the tests pin the trap: the type's index bit must be set in the
resource's `memory_type_bits` mask **and** its property flags must contain
everything the caller requires. Checking only the flags lets a device-local type
the driver excluded for this resource look perfectly suitable.

Selection is first-match rather than best-match, because the driver orders its own
memory types and imposing a Fluxel preference over that would be a second opinion
about hardware this layer has not measured. A refusal carries the unmet property
set and how many types were considered, which is what a diagnostic needs.

One test of mine was wrong rather than the rule: it asserted a refusal for a mask
that made the device-local type legal after all. The corrected version expresses
the trap properly -- the good-looking type is excluded by the mask, the only legal
type lacks the property, so the answer is a refusal.

## 17. W2 steps 1-2 as one entry point

`native::vulkan::open` chains the verified pieces into the path the RHI will
actually call: instance (with the validation inventory verified first), adapter
enumeration, index selection, one reading of the adapter's facts, then device and
queue. Reading the facts **once** matters: the same limit set both selects the
queue family and builds the capability ledger, so the ledger cannot disagree with
the device that was created.

`OpenedVulkan` declares the device before the instance so Rust's field drop order
destroys the child before the parent, which `Vulkan` requires. That is the same
rule the plan's preserved-semantics table states for native teardown, expressed as
a struct layout rather than a comment.

It is proven on this machine: the smoke test opens a real headless device through
the single entry point and asserts the hardware name, the Vulkan backend, the
texture floor from the same limit set, and the ledger's graphics/compute rows
against the selected queue family.

## 18. W2 step 4's pure half

`native::vulkan::buffer` lowers the portable `BufferUsage` onto `Vulkan` usage
flags, and it settles one design question explicitly: **the mapping does not
widen.** Each portable kind maps to exactly the flag that serves it. Silently
adding a flag makes a buffer more capable than the graph declared, and the graph's
capability check is what decides whether an operation is legal -- a buffer that
quietly became a transfer destination would pass a check it should have failed.

The one legitimate widening is the staging path the RHI's immutable uploads use,
where the RHI itself records a copy into a destination the caller never declared.
That lives in a separate named function, so the extra flag is visible at the call
site rather than hidden in the mapping.

Both storage directions map to `STORAGE_BUFFER`, and a buffer commonly carries
several kinds, so the mapping is a fold producing the union of what the usages
require.

A limit worth recording rather than papering over: `BufferUsageKind` offers no
variant iterator, so a test cannot prove the mapping lists every variant upstream
declares. The list is therefore written in the enum's own declaration order, which
makes a future addition visible in review, and the test asserts what *is*
checkable -- that every flag this backend needs is reachable.

## 19. Facade migration decided: retire the three tiers

**Decision (i), taken.** `CopyBackend`, `ComputeBackend` and `RasterBackend` are
retired as architecture. A device's usable verbs follow from which capability
families it implements, and a graph states its needs in family terms. The three
names may survive at most as an outermost compatibility adapter for pinned
consumers; **no new code in `common`, in a native backend, or in RenderGraph may
depend on them.**

Why this one: keeping the tiers as architecture is what lets them multiply --
`RasterBackend`, then `ComputeRasterBackend`, then `AdvancedRasterBackend` -- until
the family design is bypassed by a parallel type hierarchy that no one removes
because each individual step looked additive. `Copy` in particular becomes a
capability row rather than a tier, which is exactly where the copy work belongs.

## 20. Scope re-cut: 0.16 proves the model, it does not port every platform

0.16 had drifted into "all five backends converge", which this document itself
described as the largest single rendering change in the ecosystem. The scope is
therefore cut to what actually tests the design:

| Release | Packages |
| --- | --- |
| **0.16** | W1 common contract (done), W2 Vulkan, W3 DX12, W4 extract what Vulkan and DX12 genuinely repeat, W5 Metal, W9a native capability lowering, W10a delete the native `wgpu-hal` + documentation |
| **0.17** | W6 GL family, W7 browser WebGPU, W8a/W8 compressed formats, W9b five-backend capability convergence, and the remaining rendering capability |

`common` is still held to the five-backend standard — no backend mechanism may leak
into it, and the family vocabulary is written for five — but 0.16 does not require
five backends to implement it. The goal of 0.16 is now stated exactly: **not to
finish every GPU platform, but to prove that Fluxel's own capability-oriented RHI
holds up.**

### 20.1 The family-splitting rule (frozen)

**One independently negotiable batch of API per family, and one ledger row per
family.** The test is not whether two verbs sound alike; it is whether they always
appear, are always proved, and always fail together.

Two violations of this rule were found in the interface as first written and are
fixed:

- **Indirect draw and indirect dispatch were merged into one trait** while the
  ledger carried them as two rows, so a platform with one and not the other would
  have had to write a refusing method for the other. They are two traits now:
  `IndirectDrawApi` and `IndirectDispatchApi`.
- **`GraphicsApi::draw` took an instance *range*,** which can express a non-zero
  first instance -- the `FirstInstance` family. The verbs now take an instance
  *count* and the first instance is fixed at zero. The general rule this
  establishes: **a family's parameter space must not be able to name another
  family's capability.** It applies next to base vertex, depth-stencil state,
  multiview, variable-rate shading, mesh shaders and ray tracing.

`CopyApi` was also added: every current backend serves copies, and that is not a
reason to put them in the base. The same logic already keeps `Graphics` a family.

### 20.2 The task, restated after the re-cut

The requested end state, recorded here so the plan and the work agree:

1. finish **0.16** as re-cut in section 20 and push it;
2. carry out **0.17** to a push as well — W6 GL family, W7 browser WebGPU,
   W8a/W8 compressed formats, W9b five-backend capability convergence;
3. then **correct the ecosystem documentation** so it describes the code as
   actually delivered, and push that;
4. **stop.** 0.18 is explicitly not in scope.

Point 3 is not a formality. Three documents currently describe the old shape and
will be wrong the moment 0.16 lands: this stage list, the RHI design
(`documents/design-rhi.md`, which still describes the borrowed platform layer and
the `DX12`/`Vulkan`-only `Backend` enum), and the workspace README's capability
table. The interface contract
(`documents/design-rhi-capability-api.md`) is the one document already written
against the new shape.

### 20.3 Honest size of the remaining work

Recorded because the task now spans two releases, and because the numbers decide
whether it can be done in one sitting:

| Remaining | Nature |
| --- | --- |
| Vulkan steps 3-11 | allocation and binding, resources, descriptors and pipelines, SPIR-V, encoders and barriers, copies, submission and completion, surface, capability lowering — thousands of lines of `unsafe` FFI |
| DX12, W3 | the same surface again on `windows` + DXC |
| W4 | extraction, which only becomes possible once W2 and W3 both exist |
| Metal, W5 | the third implementation, plus `objc2` |
| W9a, W10a | lowering and dependency removal |
| 0.17: W6, W7, W8a/W8, W9b | GL and WebGPU convergence, portable compressed-format vocabulary, five-backend lowering |

Each of the steps completed so far was a self-contained piece with its own test and
a driver check where one was possible. The rest is the same kind of work, repeated
— there is no remaining unknown in the method, only volume. That is precisely why
the loop in section 20.2 is the deliverable that matters: it is what makes the
volume executable by whoever picks it up next.

### 20.4 Vocabulary expansion stops here

The remaining capability vocabulary is **not** to be extended further until the
Vulkan vertical slice runs deep: memory, resources, pipeline and bindings,
recording, copy, submission and completion. Designing dozens of traits before
running a real backend is exactly what produced the `requires_probe` mistake that
the first Vulkan device falsified, and the fix was found by running code, not by
reading the design.

The intended loop, restated so it survives: principle → real Vulkan → the principle
is contradicted → fix `common` → DX12 → find the genuine repetition → W4 extracts.

## 21. Recording context: what the contract does and does not require

An earlier statement of this design said a family handle *is* the recording
context. That holds for a single-queue implementation but must not be frozen into
the contract, because it would have to be undone the moment there are graphics,
compute and transfer queues with parallel recording.

The contract therefore requires only negotiation:
`require::<F>(&device)` yields a handle that may be asked for `F`'s vocabulary. The
current single-queue implementation may use that handle as its recording context;
**the contract does not require capability negotiation and recording context to
remain the same object.** A later multi-queue design can introduce a
backend-private recorder and have the handle hand its verbs to it without
replacing `Provides<F>`.

Keeping `AsyncCompute` and `TransferQueue` as ledger rows with no queue API is
consistent with this: the capability is stated, the queue assignment stays inside
RenderGraph's lowering, and nothing above gets to choose a queue.

## 22. The facade question, for the record
`CopyBackend`, `ComputeBackend` and `RasterBackend` are **orthogonal** to capability
families: they are static tiers of *capability combinations*, while a family is a
single capability. Two options, and the choice changes the public surface:

- **(i)** retire them into the result of "which families does this device
  implement", so a device's usable verbs follow from its families;
- **(ii)** keep them as the public tier names and change only what is inside.

Superseded by section 19, which takes option (i).

## 23. Automation: how one increment is chosen and reported

A single agent session's context cannot be extended from inside it, so a task of
this size is carried out by a chain of **fresh** agents, each with a full window,
each resuming from the repositories rather than from a conversation. That works only
if the repositories are a complete handoff, which is what sections 19-22 and the
ordered step list in `crates/rhi/src/native/vulkan/mod.rs` are for.

The launcher that runs the chain is local execution guidance and lives **outside**
these repositories, as the development principles require. What belongs here is the
contract each iteration obeys.

### 23.1 Choosing the increment

Take **the first unfinished step** in `crates/rhi/src/native/vulkan/mod.rs`'s
ordered list. When that list is exhausted, take the first unfinished work package in
section 20's table, in the order given. One increment per iteration, and nothing
else: an iteration that starts two things and finishes one has produced a
half-finished step for the next agent to discover.

### 23.2 What must hold before the iteration ends

- The tree **compiles**, and all three gates pass:
  `clippy --workspace --all-targets --all-features --locked -- -D warnings`,
  `clippy -p fluxel-rhi --no-default-features --locked -- -D warnings`, and
  `test --workspace --all-targets --all-features --locked`.
- Real-GPU steps additionally run `./scripts/conformance.ps1`.
- **Both repositories are pushed**: `fluxel-rendering` and `.github`. Work that is
  only committed, or only local, does not exist for the agent after you — this is
  the failure mode that turns a chain into a loop.
- If the increment cannot be finished compiling, **revert it**. A broken tree costs
  the next iteration more than an unfinished step does.

### 23.3 The report line

Every iteration ends with exactly one line, so a chain can be read at a glance:

```text
DONE <the one increment finished> | NEXT <the next step by name> | BLOCKED <or none>
```

When 0.16 and 0.17 are both pushed and the ecosystem documents describe the code as
actually delivered, the report line is followed by the words **`0.16 and 0.17
closed`**, which is the launcher's stop marker. A chain that never emits it has not
finished, and a chain that emits it while a gate is red has been lied to by its own
last iteration — which is why the gates are listed before the reporting rule rather
than after it.

### 23.4 What a fresh agent must not relearn

Recorded here because each iteration pays for a mistake once and cannot pass the
lesson on any other way:

- **Read `ash` and `gpu-allocator` from `~/.cargo/registry/src` before writing
  FFI.** Across the increments done so far this held compile iterations to zero or
  one, and it is what caught `create_device` returning an already-loaded device, the
  private representation of `SampleCountFlags`, and the non-exhaustive portable
  enums.
- **A mapping over a portable enum returns `Option`**, because those enums may be
  `non_exhaustive` and a wildcard arm would invent a value for a format or dimension
  the backend has not been taught.
- **Appending to a file anchors on the line after the insertion point**, never on a
  block that may be replaced whole. Two increments in this session silently destroyed
  a neighbouring test's opening line that way, and both were caught only by reading
  the result.

## Appendix A — capability triage recorded from the wgpu-hal GLES comparison

| Capability | wgpu-hal GLES | Fluxel today (code) | Verdict | Where it belongs |
| --- | --- | --- | --- | --- |
| Non-indexed / indexed instanced draw | present | `instance_count` is a draw payload field (`api/raster.rs`), but `compat` limits it to one instance (`compat/device/raster.rs::single_instance`) and native rejects `instances != (0..1)` (`execution/raster/operations.rs`). No recipe declares a per-instance stream: every stream is `Vertex` step mode (`imp/pipeline/raster.rs`, `compat/shader/raster.rs`) | keep | not scheduled |
| Base vertex / first instance | present, GL ≥ 3.2 / ES 3.2 | `GlAdvancedDrawCommand` exists and is capability-gated (`api/raster.rs`), but no provider implements `GlAdvancedRasterApi`, and `compat` refuses `base_vertex != 0` by name. WebGL2 has no route: the base-vertex draft extension is deliberately raw-only (`api/extensions.rs`) | do not add | preserve the named refusal |
| Multi-draw | absent (`draw_indirect_count` is `unreachable!()`) | Real single-command path on the browser (`api/multi_draw.rs`, `browser/exec_multidraw.rs`), always decomposed on native. `compat` never constructs a `GlMultiDraw`, and `GL_ARB_multi_draw_indirect` cannot be bound by glow 0.18 (`api/extensions.rs`) | defer | benchmark-driven batching slice |
| Indirect draw / dispatch | present, expanded at encode time into N commands | Single-record verbs with window/record/stride validation (`api/indirect.rs`), `GlOptionalIndirectBackend` seam reserved (`state/backend.rs`), no `ExecutionBackend` verb and no `compat` entry point | do not add | preserve the seam |
| Clear | colour via `clearBuffer*`; depth/stencil via `glClear`; buffer clear by chunked copy from a 256 KiB zero buffer | Clear is expressed only as a pass attachment load (`api/native/exec_framebuffer.rs`, scissor temporarily disabled to make the load total); no standalone clear verb exists in `api/` | do not add | preserve pass-load clearance |
| Depth / stencil attachment | complete | Every depth-stencil attachment is refused by name (`compat/device/pass.rs`) and Layer 2 has no depth-stencil state group. The legacy WebGL2 resource-floor adapter does prove `Depth32Float` usable and the WebGPU registry carries a depth attachment, so the GL family sits below the floor the workspace already claims | add, as its own slice | 0.17, driven by the Scene API; 0.16 only keeps the vocabulary able to express it |
| Texture copy | `CopyTextureToTexture` ignores format and hard-writes `COLOR_BUFFER_BIT`; compressed texture→buffer logs and returns; cubemap `unimplemented!()` | Two routes (`glCopyImageSubData` on desktop ≥ 4.3 / ES 3.x, scratch-FBO otherwise) with five named pre-copy refusals (`compat/device/transfer.rs`, `api/copy.rs`) | keep (already ahead) | preserve both routes and all five refusals |
| Multiview | claims `Features::MULTIVIEW`; the native attach branch is `#[cfg(webgl)]`, so native silently does nothing and reports a zero count | Fail-closed on every profile: browser reports `NotRun` until an attach probe exists, native has no route, desktop core profile has none | keep (do not add) | preserve the closed row |
| sRGB | `FRAMEBUFFER_SRGB` toggle, disabled during present; separate web `srgb_present` program | `Rgba8Srgb` is a first-class format, but the surface deliberately declares `Rgba8Unorm` only, because no accepted profile exposes the drawable's encoding and claiming sRGB would be double gamma (`compat/capabilities/mod.rs`) | keep | preserve; do not turn this into an sRGB surface claim |
| Stencil front/back separation | present | Absent downstream of the depth-stencil refusal | do not add | unchanged |
| Polygon mode / independent blend | polygon mode on desktop; a per-draw-buffer independent blend path | Neither exists; independent blend is a named refusal, and `EXT_draw_buffers_indexed` is deliberately not a typed route while only one colour target at index 0 is admitted | do not add | preserve the single-colour-target refusal |

## Appendix B — mapping from today's tree to the target tree

| Today | Target |
| --- | --- |
| `resource/` (public resource layer) | unchanged |
| `execution/` (RenderGraph SPI, `ExecutionBackend` impls, fixed recipes) | unchanged in meaning; capability lowering moves into `common` (W9) |
| `imp/{device_open,lowering,resource,pipeline,bindings,command,submission,presentation,upload}` — shared half | `common/` (W4) |
| `imp/{dx12,vulkan}` (borrowed calls) | `native/dx12` and `native/vulkan` (W2, W3) |
| `imp/stub.rs` | the fail-closed "no backend selected" implementation |
| `webgl2/{api,state,compat}` | `gl_family/{api,state,compat}`; `compat` becomes the GL implementation of `common` (W6) |
| `experimental/{webgl2,webgpu}` (browser closed adapters) | `webgpu/` implementing `common` (W7); the legacy WebGL2 adapter retires with the GL convergence |
| `crates/wgpu-hal` (vendored fork) | deleted (W10) |
