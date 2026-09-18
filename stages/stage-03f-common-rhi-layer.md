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

## 24. W2 step 4's owning half: the buffer table

`native::vulkan::resource` lands the first resource table. Step 4's pure halves --
`buffer`, `format` and `texture` -- each turn a portable description into a Vulkan
create-info, and `memory`/`allocator` decide a memory type and suballocate it. What
was missing was the thing that owns all of those facts at once, and `gpu-allocator`
decides its shape rather than a preference doing so: `free` takes the `Allocation`
**by value** and needs `&mut Allocator`, so a buffer cannot release its own memory
from `Drop` -- a `Drop` body has no way to reach the allocator. The table that owns
both is therefore the only thing that may release an allocation, and an `Allocation`
is never handed out without that context.

Three details are recorded because they are the ones a later reader would get wrong:

- The table holds a clone of the device's `ash::Device`, because destroying a handle
  needs it and `Drop` cannot take a parameter. The clone is a handle, not an owner --
  the `VkDevice` is `VulkanDevice`'s -- so the owner of both states must place the
  device after the table. That is stated by field order when the table is wired into
  the device, the same rule `OpenedVulkan` already uses for the device/instance pair.
- Teardown destroys the handle **first** and returns the allocation afterwards,
  because destroying the handle is what unbinds the memory. That is the order the
  borrowed Vulkan backend being replaced uses (`crates/wgpu-hal`, `destroy_buffer`),
  so the owned path does not differ behaviorally from the one it supersedes.
- The table is keyed by `BufferId`, which is `Eq + Hash` by construction and
  deliberately not `Ord`: nothing about resource identity is ordered, so the records
  live in a `HashMap` rather than having an ordering invented for them. Identity is
  also enforced by the type rather than by a run-time kind test, and a stale id
  resolves to `None` / `Unknown` instead of reaching the driver.

The failure that cost this increment a compile iteration is worth recording too: the
table was first written over a `BTreeMap`, and `ResourceId` has no `Ord`. The lesson
is the cheap one from section 23.4 applied to this layer: a table keyed by a shared
vocabulary type must be checked against that type's derives before it is written,
not after.

Proven on this machine: a real table creates two device-local buffers through the
real allocator, looks them up by identity, refuses a zero size before the driver is
reached, destroys one, and lets its `Drop` release the other.

## 25. W2 step 4's texture half: the resource table owns textures too

`native::vulkan::resource` was the buffer table; it is now the resource table. The
texture half of step 4 -- image, memory, and the view a graph samples through -- is
owned beside the buffers rather than in a second table, and that is a decision
`gpu-allocator` forces as much as a preference does: the suballocator hands out whole
`vkDeviceMemory` blocks, so a second allocator on one device would be a second set of
blocks for memory the driver cannot move between them. One table also means one
identity counter, so a buffer and a texture can never be handed the same physical
identity.

`texture::view_type` and `texture::view_create_info` are the pure half. One rule in
them is worth recording because a first draft gets it wrong: a layered
two-dimensional description takes a `TYPE_2D_ARRAY` view, not `TYPE_2D`, because a
`TYPE_2D` view of a layered image silently selects layer zero -- a different
resource than the graph named. The aspect is asked of the mapped `Vulkan` format
rather than the portable one, so `format::is_depth` keeps its single source of
truth, and the view spans exactly the levels and layers the image was created with
(both floored at one, as the image lowering already floors them).

Teardown gains one step in front of the buffer order: the view is destroyed before
the image, because a view refers to an image and not the other way round, and the
image is what unbinds the memory released only afterwards. Every failure path in
`create_texture` undoes its own work in that same order, so a refused texture leaves
neither a handle nor an allocation behind.

One rename came with the second kind: `BufferError` is `ResourceError`, and it gained
`UnsupportedTexture` and `View`. One table now answers for both kinds, and "the
description is one this backend has not been taught" is a different sentence from
"the size was zero", so the two stay separate variants for the same reason step 4's
buffer half separated them.

Proven on this machine: a real table creates a texture through the real allocator,
looks its image and its view up by identity, creates a buffer with a different
identity, and destroys both; and an unsupported description -- a multisample and a
zero extent -- is refused before the driver is reached. The three required gates pass;
`scripts/conformance.ps1` is the W2 close gate and is not re-run for a mid-step
increment.

Still owed by step 4: samplers (their descriptor lowering and their owner), which
were deliberately not started here.

## 26. W2 step 4 complete: the sampler half

`native::vulkan::sampler` and the sampler half of the resource table close step 4.
A sampler is the one resource in this backend with nothing to bind and nothing to
allocate, so its whole contract is a descriptor, a handle and an identity -- but it
is the resource where the stage's preserved semantics are easiest to lose, because
`Vulkan` spells the null comparison differently from DX12.

`create_info` derives `compare_enable` from `compare.is_some()` and from nothing
else, and writes `compare_op` **only** in the `Some` branch. The accident available
on this API is not the DX12 one -- there is no `COMPARE_OP_NONE` to confuse with
`ALWAYS` -- but its equivalent: enabling the comparison while filling the ordering
from a default, which turns an ordinary filtering sampler into a shadow sampler
exactly as the vendored DX12 patch exists to prevent. A test pins both directions,
including that the disabled field keeps `Vulkan`'s default rather than some value
ash happens to have, and two real-driver tests create one filtering and one
comparison sampler, because a driver that refused the enabled branch is the one fact
a pure test cannot see.

Every other field of `VkSamplerCreateInfo` is pinned to the value that claims
nothing: zero LOD bias, anisotropy disabled with `max_anisotropy` at 1.0 (the row is
unproved, so enabling it would be a capability claim), the default border colour
that no portable address mode can reach, and normalized coordinates. One refusal is
added -- an inverted LOD range, which `Vulkan` requires to be ordered -- and it is
refused here rather than at the driver for the same reason a zero-sized buffer is.

The owner grew rather than appearing: `SamplerKind`/`SamplerId` joined
`common::base::resource` because a sampler is a device-owned object a binding names
and a stale generation must be refused before it can be bound, and the resource table
took a third map on the one identity counter. Samplers are destroyed first in
teardown and that placement claims nothing: a sampler refers to no image and binds no
memory, so it has no dependency to honour. `ResourceError::Create` now names a
sampler among the handles it covers, and `InvalidSampler` is a separate sentence
from `UnsupportedTexture` because "this backend has not been taught the description"
and "this description contradicts itself" are different things for a caller to fix.

Proven on this machine: the table creates a linear-clamp and a comparison sampler,
looks both up by identity, refuses an inverted LOD range before the driver is
reached, destroys one, and lets its `Drop` release the other; both samplers reach a
real `VkSampler` and are destroyed. The three required gates pass.
`scripts/conformance.ps1` is the W2 close gate and is not re-run for a mid-step
increment.

One process note, because section 23.4 already records its family: an edit that
replaced a block ending in a test's `#[test] fn` line matched a second occurrence of
that line elsewhere in the file and silently renamed a real-driver test to the
bijection test's name. It was caught by reading the file back, not by the compiler --
the body still compiled because both tests return `()`. Anchoring an append on a
unique line, and reading the result, remains the rule.

Step 4 is complete. Step 5 (descriptors and pipelines) is next.

## 27. W2 step 5's shader and compute-pipeline half

Step 5 is descriptors and pipelines. Its first bounded piece landed -- the SPIR-V
module, the pipeline layout, and the compute pipeline built over it -- and the
working tree it was written in was found uncommitted, so this increment is the
verification, recording and landing of that piece rather than a further slice.
Descriptor set layouts and the raster pipeline remain owed by step 5.

`native::vulkan::shader` owns a `VkShaderModule` and states two facts in the type
rather than re-checking them: `&[u32]` is already a non-zero-multiple-of-four byte
size at a four-byte-aligned address, which is everything
`VkShaderModuleCreateInfo` requires of `codeSize` and `pCode`. What is left is the
payload itself, and both refusals are values returned before the driver is
reached: an empty slice, and a first word that is not the SPIR-V magic number --
carried as `NotSpirV { found }` rather than a boolean, because a different
container and a truncated payload produce different first words and the diagnostic
should say which arrived. Nothing here decodes the instruction stream: that
belongs to step 6, and a second partial parser would be a second truth about the
same bytes.

The module is deliberately **transient**. `Vulkan` copies the words at creation
and reads the module only when a pipeline is created, so `Module` owns the handle
and the device, never the SPIR-V, and destroys the handle in `Drop`.
`pipeline::create_compute` creates one inside its own body, records the stage, and
lets that drop run on the way out of the call -- after `vkCreateComputePipelines`
has returned and not before. The stage mapping is exhaustive over this crate's
closed `ShaderStage` and has no wildcard, which is the opposite shape from a
mapping over a `#[non_exhaustive]` portable enum: a stage added later must be
taught to the match rather than silently lowered to no stage flag.

`native::vulkan::pipeline` owns the layout *inside* the pipeline. A `VkPipeline`
refers to its `VkPipelineLayout` at bind time, so the only shape that cannot be
misused is the longer-lived object containing the shorter one; field order then
destroys the pipeline before the layout, and the layout before its device. The
set-layout slice is already a parameter, so the descriptor increment fills it
rather than changing this call, and an empty slice is the honest description of a
shader that declares no bindings. Push constant ranges stay empty because the
borrowed path being replaced passes an immediate-data size of zero, which is "no
push constants" here.

One `ash` 0.38 fact worth recording, read from the source rather than its
documentation: `create_compute_pipelines` returns `Result<Vec<vk::Pipeline>,
(Vec<vk::Pipeline>, vk::Result)>`, and the specification leaves the output array
**undefined** on failure. The partially filled vector that path returns is
therefore dropped without being read -- destroying a handle the driver did not
promise to have created would be worse than not destroying it.

Proven on this machine, not only in unit tests: a real `VkShaderModule` and a real
`VkComputePipeline` over a real empty layout are created and destroyed through the
entry point, and a payload that is not SPIR-V is refused before the driver is
reached, with the consumed layout released rather than leaked. The three required
gates pass; `scripts/conformance.ps1` is the W2 close gate and is not re-run for a
mid-step increment.

Still owed by step 5: descriptor set layouts (bind group layouts) and the raster
pipeline.

## 28. W2 step 5's descriptor half: the bind-group layout vocabulary and the set layout

Step 5's remaining pieces were descriptor set layouts and the raster pipeline. The
descriptor half landed: `common::binding` states the bind-group layout vocabulary,
and `native::vulkan::descriptor` lowers it, creates the `VkDescriptorSetLayout` and
owns it. The raster pipeline remains owed.

### 28.1 The vocabulary is the one W1 owed, and it stays closed

`common::binding` is the value vocabulary section 9.1 placed in `common` and W1 had
not yet written: `ShaderVisibility`, `BufferBindingType`, `TextureSampleType`,
`SamplerBindingType`, `StorageTextureAccess`, `ViewDimension`, `BindingKind`,
`BindGroupLayoutEntry` and `BindGroupLayout`. Like `common::vertex` it is a closed
set — the kinds a retained artifact declares — so a caller cannot describe a binding
no artifact uses. `ShaderVisibility` is a bitset with a **private** representation
rather than an enum, because the retained raster recipe's frame uniform is visible
to two stages at once and visibility composes; only the named constants and `union`
produce a value, so no stage-less bitset is constructible outside the module that
tests the refusal.

Two properties are checked where they belong. Duplicate binding numbers and an entry
no stage can see are properties of the layout itself, so `BindGroupLayout::validate`
refuses them as values before any backend or driver sees the description. Whether a
format may be a storage texture, or a filterable float sample is legal for a format,
is a capability question and stays with the ledger; it is deliberately **not** in
this type.

### 28.2 Why the layout lowering is two fields, and one of them is not a name change

`VkDescriptorSetLayoutCreateInfo` has nowhere to put a view dimension, a sample
type, a storage format or a minimum binding size. Only the descriptor type and the
stage flags reach the driver, and lowering the rest there would be a capability claim
this call cannot back — a second truth about the same layout. Those facts are bind
group and pipeline validation, and the steps that need them will read them from the
entry.

The descriptor type is the one place the lowering is not a plain name change: a
buffer binding with a dynamic offset gets `UNIFORM_BUFFER_DYNAMIC` /
`STORAGE_BUFFER_DYNAMIC`, because `Vulkan` spells the dynamic case as a different
descriptor type. Lowering the flag to the non-dynamic type would produce a layout the
driver accepts and a bind that later fails. A storage buffer's `read_only` does
**not** change the descriptor type: it gates the shader's declaration, not the
layout.

### 28.3 The one `ash` trap this increment had to avoid

Binding arrays are not in the vocabulary, so every entry is one descriptor. `ash`'s
`DescriptorSetLayoutBinding::immutable_samplers` *derives* `descriptor_count` from
the slice length, so the natural-looking `.immutable_samplers(&[])` would have
silently lowered every binding to **zero** descriptors. `descriptor_count` is written
as one directly and the immutable-sampler pointer is left null; a test asserts both,
because the wrong version still creates a layout successfully and fails only at bind
time.

### 28.4 Ownership is the dependency, stated as fields

`Vulkan` requires a descriptor set layout to outlive every pipeline layout that names
it, and a pipeline layout to outlive every pipeline created against it. `SetLayout`
owns its handle and destroys it in `Drop`; `PipelineLayout` now owns the
`Vec<SetLayout>` it was created over rather than borrowing raw handles, so field
order destroys the pipeline layout before the set layouts and the pipeline before
both. `create_layout` takes `Vec<SetLayout>` instead of `&[vk::DescriptorSetLayout]`
for exactly that reason — the earlier "the slice is already a parameter" note
described the call shape, but not who owns the handles, and a borrow would have left
the owner to a caller that has nowhere to put it.

### 28.5 Proof

Pure tests cover the five descriptor-type lowerings and their distinctness, the
dynamic/non-dynamic choice, the storage-direction invariance, the stage-flag fold,
and the count-one/no-immutable-sampler shape. `common::binding` tests cover the
retained textured layout, the empty layout, the duplicate refusal and the
stage-less refusal. Against the real driver: a real `VkDescriptorSetLayout` is
created from the retained textured-frame layout and destroyed, an empty layout is
created, a duplicate-binding description is refused before the driver is reached,
and a real `VkComputePipeline` is built over a pipeline layout that owns a real set
layout and then dropped — which exercises the whole owner chain. The three required
gates pass; `scripts/conformance.ps1` is the W2 close gate and is not re-run for a
mid-step increment.

This increment compiled without an iteration, and the `immutable_samplers` trap in
28.3 was caught by reading the `ash` 0.38 source before writing the lowering, which
is the rule section 23.4 already records.

Still owed by step 5: the raster pipeline — landed in section 29.

## 29. W2 step 5 complete: the raster pipeline

`common::pipeline` landed the fixed-function vocabulary W1 owed — primitive
topology, cull mode, front face, depth-stencil state, colour target and write mask,
sample count — and `native::vulkan::pipeline::create_raster` lowers it into a real
`VkGraphicsPipeline`. Step 5 is complete; step 6 (Naga `spv-out`) is next.

### 29.1 The one refusal the vocabulary owns

A raster pass admits exactly one colour attachment at index 0 (plan section 4), so
`PipelineState::validate` refuses a second colour target before any backend sees the
description, and also refuses a description with no attachment at all and a zero
sample count. What it deliberately does **not** decide is whether a name is a colour
or a depth format: `TextureFormat` is `#[non_exhaustive]`, so a classification
written in `common` would need a wildcard arm that invents an answer for a format
the layer has not been taught. The backend's format table answers it, and
`signature` refuses the two mismatches by name — a colour target that carries depth,
and a depth-stencil state that carries none.

`DepthStencilState::depth_compare` is `common::sampler::CompareFunction` rather than
a second eight-variant enum: the eight orderings are one vocabulary, and the sampler
module already owned them.

### 29.2 Three states the lowering pins

- **Viewport and scissor are dynamic.** `GraphicsApi::set_viewport` and
  `set_scissor` are the verbs that write them, so a pipeline that baked them in
  would silently ignore those verbs. The viewport state declares one viewport and
  one scissor with null pointers, which is the legal dynamic shape.
- **The depth-stencil state is present exactly when the description names a
  depth-stencil attachment**, because `Vulkan` validates a pipeline against the
  subpass it is created for: without the state it is invalid against a render pass
  that has a depth attachment, and with it, against one that does not.
- **Every field the portable description does not state is pinned to the value that
  claims nothing.** This is the raster increment's version of the sampler's
  null-comparison rule: `PipelineColorBlendAttachmentState::default()` writes **no**
  colour components, so a lowering that forgot the write mask would produce a
  pipeline that renders nothing and still creates successfully. The write-mask fold
  is asserted directly, and the rest (blend disabled with the replace identity, no
  logic op, no sample shading, the default sample mask, no depth bias, no stencil,
  fill polygon mode, a line width of one) is written rather than defaulted.

### 29.3 The render pass, and why it is not owned

`Vulkan` 1.0 has no dynamic-rendering path, so `vkCreateGraphicsPipelines` needs a
render pass. `create_raster` creates one from the pipeline's attachment signature and
destroys it as soon as the pipeline exists, rather than storing it in
`RasterPipeline`: a pipeline does not refer to a render pass after creation
(`vkDestroyRenderPass` requires only that submitted commands referring to it have
completed), and the render passes the recording step begins are *compatible* with
the pipeline when their attachment formats, sample counts and reference layouts
agree — which is exactly what `signature` states and what the recording step will
create from. The creation render pass's load and store operations are `DONT_CARE`,
because contents operations belong to the compiled pass (RenderGraph's
`AttachmentOps`), not to a pipeline, and they do not participate in render pass
compatibility.

### 29.4 Proof, and what it cost

Against the real driver: a colour-only `VkGraphicsPipeline` over an empty pipeline
layout, a depth-attached one over a pipeline layout that owns a real descriptor set
layout, and a non-SPIR-V payload refused while the modules are being described —
before the render pass exists — with the consumed layout released. The pure tests
cover the five topologies, three cull modes, two windings, three vertex formats, two
step modes, the write-mask fold, the named sample counts and their refusals, the
attachment signature, and the two format mismatches. `common::pipeline` covers the
retained colour-only and depth-sibling descriptions, the second-colour-target
refusal, the no-attachment refusal, the zero-sample refusal and the write-mask fold.

The two modules the raster tests build from are Naga-emitted SPIR-V for a minimal
position-in/colour-out recipe, standing in for step 6 exactly as
`MINIMAL_COMPUTE_SPIRV` does for the compute half; they were produced once with
Naga's `spv-out`, targeting SPIR-V 1.0, and embedded, so this repository needs no
SPIR-V assembler.

The increment compiled with no iteration. Two of its own *pure assertions* were
wrong rather than the code, and both are worth recording because each is a trap that
reads as a true statement:

- `CullModeFlags::contains(NONE)` is **always** true, so "pairwise non-containment"
  is not a distinctness test when one of the values is the empty set; the empty mode
  has to be asserted `is_empty()` and compared for inequality instead.
- `SampleCountFlags`'s named counts happen to equal their bit (`TYPE_4` is `4`), so
  "the flag bit is not the count" is false for every legal value. What the match
  actually buys is the refusal of the *illegal* ones — `SampleCountFlags::from_raw(3)`
  is a value `Vulkan` does not define — which is the only place a cast would have
  differed.

Both were caught by running the tests, not by review.

The three required gates pass; `scripts/conformance.ps1` is the W2 close gate and is
not re-run for a mid-step increment.

## 30. W2 step 6: the retained WGSL lowered to SPIR-V

`native::vulkan::wgsl` lands step 6. Decision F's two shader routes now both exist:
caller-supplied SPIR-V is the passthrough `shader::create_module` step 5 already had,
and WGSL is parsed, validated and written to SPIR-V by Naga's `wgsl-in` frontend and
`spv-out` backend. The two meet in `wgsl::create_module`, which lowers when it has to
and hands the words to the module creator, so the driver is only ever handed SPIR-V.
The `naga/spv-out` feature is enabled by this crate's `vulkan` feature, which is the
same condition the module is compiled under (`all(windows, feature = "vulkan")`).

### 30.1 The dialect, entry-point, stage and profile checks come first

Every refusal is a value returned before a `VkShaderModule` exists, and each names
something a driver would not:

- **dialect** — a payload the WGSL frontend cannot parse, carrying the frontend's
  diagnostic rendered against the source. GLSL is the dialect the GL family accepts
  natively; this backend lowers only WGSL.
- **entry point** and **stage** — a name the module does not declare and a name
  declared for another stage are different sentences. Naga's writer would collapse
  both into `EntryPointNotFound`, so the check is made here; the found stage is
  carried as Naga's own type, because a task, mesh or ray-tracing entry point is not
  one this crate's closed `ShaderStage` can name, and saying so is more useful than
  calling the entry point absent.
- **profile** — the writer targets SPIR-V 1.0, the version the instance this backend
  opens requests (`VK_API_VERSION_1_0`, step 1). A module needing more is refused by
  the writer rather than handed to a driver that cannot load it.

The stage mapping is exhaustive over the three-variant portable `ShaderStage` with no
wildcard: a closed enum's new variant must be taught to the match. The reverse
mapping is deliberately not written, because it would have to return `Option` —
Naga declares stages this vocabulary does not.

### 30.2 The writer options, and the default that is not neutral

`naga::back::spv::Options::default()` sets `ADJUST_COORDINATE_SPACE`, which flips the
Y coordinate of `BuiltIn::Position`. The borrowed Vulkan path being replaced does
**not** set it, so inheriting the default would have flipped every retained recipe's
geometry against the frozen oracle while compiling cleanly and passing every unit
test. The options are therefore written field by field, so a field Naga adds later
breaks this literal instead of being inherited. The decisions that carry behavior:

| Field | Value | Why |
| --- | --- | --- |
| `lang_version` | `(1, 0)` | the version the opened instance requests |
| `flags` | `FORCE_POINT_SIZE` only | a vertex module is lowered before the topology it will be paired with is known, `Points` is in the vocabulary, and the borrowed path always emits the built-in. `ADJUST_COORDINATE_SPACE` changes geometry, `CLAMP_FRAG_DEPTH` clamps an output no retained fragment writes, `LABEL_VARYINGS` writes decorations the specification does not require, and `DEBUG` would make the emitted words depend on the build profile rather than on the artifact a cache hashes |
| `capabilities` | the seven the borrowed path treats as always available | `None` means "all capabilities are permitted", which would let a shader use `Float64` or `MultiView` on a device that enables no feature at all |
| `use_storage_input_output_16` | `false` | the device enables no f16 feature, and this field is exactly that capability |
| `fake_missing_bindings` + empty `binding_map` | `true` + empty | this backend keeps each artifact's own `@group`/`@binding` numbers, and an empty map plus this fallback **is** that identity mapping. With `false` and an empty map, every resource would be refused as a missing binding |
| bounds policies | `Restrict` for index, buffer and image loads; `Unchecked` for binding arrays | the device enables no robust-access feature; binding arrays are not in this layer's vocabulary |

`fake_missing_bindings` is the one field whose name is misleading enough to be worth
stating: it is not a permission to invent a binding, it is the fallback that emits the
binding the artifact declared when the map has no override.

### 30.3 Two capability layers that are not duplicates

Validation runs with `ValidationFlags::all()` and `valid::Capabilities::empty()`, so an
artifact using a feature this device has not proved is refused before the writer runs.
The writer's SPIR-V capability set is a second, distinct check: Naga's validator knows
the portable feature set, while the writer decides which SPIR-V capabilities may
appear in the emitted words.

### 30.4 Proof

Pure tests cover the retained triangle lowered for both its stages (and that one module
lowered for two stages is not the same words twice), a retained storage-texture compute
artifact, the stage-mismatch refusal, the unknown-entry-point refusal, a non-WGSL
payload refused at parse, a module that parses but does not validate, and the
deliberate option values (SPIR-V 1.0, no Y flip, the capability set, the identity
binding fallback, the polyfilled workgroup zero-init). Against the real driver, a
retained vertex artifact and a retained compute artifact are lowered and their modules
created and destroyed, and a refused entry point is shown to fail as the lowering's
error with no driver involvement.

One of the increment's own test *assertions* was wrong rather than the code, and it is
worth recording because the first guess reads as true: Naga's WGSL frontend refuses
`return 1.0;` from a function declaring `-> vec4<f32>` **at parse**, because the
automatic conversion is already rejected there, so that source cannot demonstrate the
validator at all. A "parses but does not validate" case needs illegal-but-parseable
input, and a zero workgroup dimension (`@workgroup_size(0)`) is one.

The increment compiled with no iteration; the one clippy iteration was removing a test
import the module did not need. The `ash`, `gpu-allocator`, `naga` `spv-out` and
vendored `wgpu-hal` sources were read before the FFI and the option literal were
written, which is the rule section 23.4 already records — and it is what caught
`Options::default()`'s Y flip, which no compiler or unit test would have reported.

Step 6 is complete. Step 7 (one command encoder with explicit transitions) is next.

## 31. W2 step 7 complete: the command encoder and its explicit transitions

Step 7 is recording: one command encoder whose `vkCmdPipelineBarrier` transitions are
driven by RenderGraph's semantic access states, with a `before == after` transition
kept a memory dependency. It landed as two modules, `native::vulkan::barrier` (the
pure half) and `native::vulkan::command` (the command pool and the one recorder),
because that is where the preserved semantic and the driver call naturally separate.

### 31.1 The whole translation lives in the pure half

`barrier` lowers one `ResourceAccessState` onto the pipeline-stage and access masks a
barrier needs, plus the image layout when the resource is a texture. `command` never
spells a mask: every `vkCmdPipelineBarrier` argument comes from the lowering, so
there is one place to be wrong instead of two.

Both mappings return `Option`, and the two reasons are different sentences. The
portable enum is `#[non_exhaustive]`, so a state added upstream is refused rather
than lowered to a guessed mask; and the one enum is shared by buffers and textures,
so `ColorAttachmentWrite` is refused as a buffer access and `VertexRead` is refused
as a texture access. A wildcard would have invented a barrier for a state nobody has
measured, which no compiler and no driver reports.

Three lowerings are worth naming because a first draft guessed them:

- **A sampled read's layout depends on the format.** A depth texture sampled through
  `SHADER_READ_ONLY_OPTIMAL` is an invalid barrier, so the answer is
  `DEPTH_STENCIL_READ_ONLY_OPTIMAL` where the mapped format is depth. The question is
  asked of the *mapped* `Vulkan` format through `format::is_depth`, so the depth fact
  keeps the single source of truth step 4 already established.
- **A sampled read orders against all three shader stages.** The semantic state does
  not say which stage reads, so narrowing it to the fragment stage would be a second,
  unwritten fact. The borrowed Vulkan path being replaced maps its `RESOURCE` use the
  same way.
- **The queue family indices are `QUEUE_FAMILY_IGNORED`, not zero.** This backend
  records and submits on logical queue 0 only, so there is no ownership transfer to
  express; the ignored value states that, while a zero would claim the barrier is
  about family zero.

### 31.2 No equality shortcut, and a test that makes deleting it impossible

Plan section 4 and the `ExecutionBackend` contract both require a same-state
transition to remain a memory dependency. Nothing in either module compares `before`
with `after`: the two sides are lowered independently and a barrier is built, so it
has equal masks and an unchanged layout. A test asserts exactly that shape for both a
buffer and an image, which means the equality shortcut cannot be added later without
deleting a test that names the semantic.

### 31.3 The owning half, and what it deliberately does not own

`CommandPool` is created on the device's selected queue family, owns the pool and
destroys it. No pool flags are passed, and the encoder begins with no begin flags:
`ONE_TIME_SUBMIT` would promise the driver the buffer is submitted once, and the
submission path that could keep that promise is step 9's, so claiming it here would
be a claim with no backer.

`Encoder` owns one primary command buffer and frees it on drop, which is what the
contract asks of a dropped unfinished encoder: the recording is discarded, not
executed. It deliberately does **not** own the pool -- a command buffer is freed
through its pool -- so the device's pool must outlive every encoder created from it,
stated as the same field-order invariant `ResourceTable` already carries for its
device. Submission does not exist yet, so freeing on drop is safe, and step 9 owns
the handoff that keeps a finished buffer alive until its fence signals.

Every refusal is a value and none of them poisons the recording. The contract says
the executor still calls the matching `end_*` after a callback error, so a refused
transition must leave the encoder endable and its already-recorded commands intact;
the real-driver test asserts that directly. A second `end`, or a transition after
`end`, is refused as `NotRecording` rather than accepted idempotently, because "end
an encoder that is not recording" is a caller mistake that silent acceptance would
hide until submission.

### 31.4 What it cost

One compile iteration: the `Encoder` initializer omitted its `recording` field, which
the compiler named immediately. One test-only correction: `ash`'s
`ImageSubresourceRange` does **not** implement `PartialEq` -- unlike the generated
bitflags and `ImageLayout`, which do -- so the two range-refusal tests assert
`.is_none()` rather than `assert_eq!(..., None)`. The `ash` source was read before the
FFI, which is what settled the whole-range spellings (`REMAINING_MIP_LEVELS`,
`REMAINING_ARRAY_LAYERS`), the whole-buffer size (`WHOLE_SIZE`) and the fact that
`destroy_command_pool` already frees every command buffer still allocated from it.

### 31.5 Proof

Pure tests cover: every named state lowering to a non-empty stage mask (an empty mask
is not a legal barrier input); the wrong-kind refusals in both directions; the
same-state barrier shape; the sampled-read layout following the mapped format; the
sampled read ordering against all three shader stages; the two copy directions
staying distinct in stage, access and layout; whole-range aspects; explicit
subresource fields; and the zero-count range refusals. Against the real driver: a
real command pool and a real encoder record buffer and image barriers from portable
states over both a whole range and an explicit one, including a same-state
transition, a state of the wrong kind is refused while the recording stays usable and
still ends, and an ended encoder refuses further recording.

The three required gates pass; `scripts/conformance.ps1` is the W2 close gate and is
not re-run for a mid-step increment.

Step 7 is complete. Step 8 (copies) is next.

## 32. W2 step 8 complete: the two copy routes

`native::vulkan::copy` lands the pure half of step 8 and `Encoder::copy_buffer` /
`Encoder::copy_texture` the owning half, so the two routes this family owns --
`vkCmdCopyBuffer` and `vkCmdCopyImage` -- are recorded from portable regions. The
buffer/image routes (`vkCmdCopyBufferToImage` / `vkCmdCopyImageToBuffer`) are
deliberately **not** here: they belong to the RHI's own staging upload and readback
path, neither has vocabulary in `CopyApi`, and a record for a call nobody can make
would be a second truth about the same bytes. They land with the step that first
needs them.

### 32.1 The checks the shared layer states, repeated where the driver is reached

Step 8's wording is exact: the alignment and range checks "the safe layer already
performs" are repeated at the boundary. The rules are not new ones. A buffer
region's offsets and size are aligned to `COPY_BUFFER_ALIGNMENT` and fit both
declared sizes, and the end is computed in **checked** arithmetic -- the `u64::MAX`
trap is a `Overflow` refusal rather than a wrapped range that passes the bounds
test. A texture region's extent is non-zero on every axis, both mips exist, and the
origin plus extent fits the mip extent of the image it addresses.

The alignment is written as a local constant and a test pins it against
`wgpu_types::COPY_BUFFER_ALIGNMENT`, because the value is a contract between two
crates and a change to either side would otherwise move the other silently.

### 32.2 Two decisions this module had to make, and both are refusals

`TextureCopyRegion` carries **no aspect**, and that is right: the aspect follows
from the image's format, which the resource table owns. It is asked of the *mapped*
`Vulkan` format through `texture::aspect`, so the depth fact keeps the single source
of truth `format::is_depth` already gives it. A depth texture therefore copies
through the depth aspect without the caller naming one.

The second is layers, and the honest answer was found by reading what the borrowed
path and the shared layer actually do rather than by extending the vocabulary:

- the shared layer (`rendergraph/src/execution/recording/copy.rs`) refuses
  `extent[2] != 1` and a non-zero `origin[2]` **for every dimension**, at `D2`
  descriptors with `array_layers == 1`, `sample_count == 1` and no `Depth32Float`;
- the borrowed `Vulkan` path (`wgpu-hal`, `conv::map_subresource_layers`) writes
  `layer_count = 1` unconditionally.

Both resolve the portable region's z ambiguity -- layer index and layer count, or
depth coordinate and box depth -- by refusing every multi-layer shape, so this
module refuses the same two by name (`LayerOrigin`, `LayerCount`) instead of
inventing one reading for a region the graph could not have built. The result is
strictly narrower than the shared layer on the descriptor axis and **exactly equal**
to it on the region axis: every region the shared layer accepts records here, with
the same aspect, mip, base layer and one-layer subresource the borrowed path
writes. A layered or volume copy arrives with the consumer that needs it, together
with the vocabulary to say which layer it addresses. That is plan section 3's rule
applied to a decision the first draft of this module got wrong in both directions:
it first allowed any origin and any depth, then briefly allowed a full layer range
that neither the contract nor the driver path being replaced supports.

Two smaller facts are worth recording because a first draft guessed them:

- a depth-addressable image's z axis is a **depth**, and a two-dimensional image's
  is its **array layer count**, so the mip-extent helper reads
  `TextureDimension::D3 ? extent.depth : array_layers`. Reading `extent.depth` for
  both would let a box address layers an image does not have, and reading
  `array_layers` for both would refuse a legal volume.
- the command lays the *destination* image out `TRANSFER_DST_OPTIMAL` and the
  source `TRANSFER_SRC_OPTIMAL`, which is exactly the layout pair
  `barrier::image_state` gives `CopySource` and `CopyDestination`; a caller that
  transitions through those states is consistent with the copy it records.

### 32.3 What it cost

Two compile iterations, both test-side and both worth keeping in the record:

- `assert_eq!` cannot compare a `Result<vk::BufferCopy, _>` or
  `Result<vk::ImageCopy, _>`, because `ash` derives no `PartialEq` for those
  structs -- the generated bitflags and layouts do, and these do not. The tests
  compare `.err()`, which is the same shape section 31.4 already recorded for
  `ImageSubresourceRange`.
- one assertion was wrong rather than the code, in the same family as the previous
  increment's: `u64::MAX - 4` is not a multiple of the alignment, so the case meant
  to isolate overflow was refused as `Misaligned` instead. The size is now
  `u64::MAX - 3`, which is.

### 32.4 Proof

Pure tests cover the field-for-field buffer lowering, the zero/misaligned/overflow/
out-of-bounds refusals by name, the same four for textures plus the differing-format,
unknown-mip, zero-extent, layer-origin and layer-count refusals, the mip-halved
bounds, the depth aspect, and the one-layer shape. Against the real driver: a real
command pool records a real `vkCmdCopyBuffer` between two real `VkBuffer`s and a
real `vkCmdCopyImage` between two real `VkImage`s over their real transfer
transitions, then ends; a misaligned region, an out-of-bounds box and a
differing-format pair are each refused as their own sentence while the recording
stays usable and still ends; and a destroyed id is shown to resolve to no handle at
the table, so a stale generation cannot become a driver call.

The three required gates pass; `scripts/conformance.ps1` is the W2 close gate and
is not re-run for a mid-step increment. No `#[ignore]` fixture changed, so the gate's
counted fixture set is untouched.

Step 8 is complete. Step 9 (submission and completion) is next.

## 33. W2 step 9 complete: the fence, one submit, and the completion state machine

Step 9 landed as `native::vulkan::submission`: one unsignaled fence per execution,
one `vkQueueSubmit` on the device's single queue, and the completion observation
that turns a fence's state into `CompletionStatus` through `common`'s disposition
and lifetime rules. What the step deliberately does not contain is a non-blocking
retirement path: reclaiming a quarantine is the RHI layer's job, and it arrives with
the completion bundle that owns a `Submission`.

### 33.1 The handoff is a type, not a rule

The command encoder's own docs said step 9 owns "the handoff that keeps a finished
buffer alive until its fence signals". `Encoder::finish` now returns a `Finished`,
and `Finished` is the only value `submit` accepts. `Vulkan` accepts only an
executable command buffer, so "submit a recording that has not ended" is
unrepresentable rather than a run-time refusal: `finish` ends an open recording and
passes an already-ended one through, so no call shape reaches the driver with a
recording buffer. `Submission` then owns the `Finished` -- and therefore the command
buffer -- beside the fence, which is the handoff stated as ownership rather than as a
comment.

### 33.2 The state machine is `common`'s, and `common`'s only

Nothing here re-decides terminality. `Disposition::observe` is called with the status
the fence produced, and `may_release` is what decides whether that status releases.
`observe` returns its `AlreadyTerminal` for a submission whose resources are gone, so
a second `status()` or `wait()` after a completed wait is refused rather than answered
from a freed fence. `Disposition` is also what makes the accepted-unknown case work:
it is `abandon()`ed, and `may_release` is false for `Abandoned`, so the bundle stays
quarantined even though the reported status is `Failed`.

### 33.3 Three pure lowerings, one of which is easy to get backwards

The fence's answers are lowered by pure functions, so the cases a driver cannot be
asked to produce are still testable:

- `wait_outcome` separates `VK_TIMEOUT` from a driver result. A timeout is *work
  still in flight* and reports `Pending`; a driver result is *completion cannot be
  established* and reports `Failed`. Collapsing them would turn a slow frame into a
  device loss.
- `failure_of` names `VK_ERROR_DEVICE_LOST` and maps every other result to
  `ExecutionFailed`. It constructs `CompletionFailure` rather than matching it
  exhaustively, because the portable enum is `#[non_exhaustive]`.
- `timeout_nanos` clamps a `Duration` too large for `u64` to `u64::MAX`, the
  specification's "wait forever". Wrapping with `as u64` would turn the longest
  possible wait into a short one, which is the unsafe direction for a completion
  wait.

The fence is created **unsignaled**, and that is a deliberate value rather than a
default: a fence created signaled would make every submission look complete before
the driver had run anything.

### 33.4 Accepted-unknown is a rejection only after idle proves it

`vkQueueSubmit` returning an error is not proof that nothing ran, so a rejection is
reported as `SubmitError::Rejected` **only** after a successful `vkQueueWaitIdle`.
Where idle cannot be established, the submission is returned live with the failure
recorded and its disposition abandoned, and `Drop` then quarantines the command
buffer and the fence by forgetting both instead of risking use after free. That is
the plan's preserved semantic, and it is the shape the borrowed path being replaced
uses when it forgets its bundle. A driver rejection that idle *does* clear releases
the recording and destroys the fence in dependency order: command buffer first, fence
second.

### 33.5 Proof

Pure tests cover the three lowerings, including the timeout/failure distinction and
the clamped duration. Against the real driver: a real command pool records a real
buffer copy, `finish` ends it, one submit is accepted, a ten-second fence wait
reports `Complete`, the submission reports `is_terminal`, and both `status()` and
`wait()` afterwards are refused as `AlreadyTerminal` -- the release being observed
rather than assumed. A second test submits two executions on the one queue and both
fences signal, which is "one submit per graph execution on logical queue 0"
exercised twice.

The increment compiled with no iteration. The `ash` 0.38 source was read before the
FFI, which is where `wait_for_fences`'s `VkResult` return and `get_fence_status`'s
`SUCCESS`/`NOT_READY` split were settled; the "wait forever" value itself is the
specification's. The three required gates pass; `scripts/conformance.ps1` is the W2
close gate and is not re-run for a mid-step increment.

Step 9 is complete. Step 10 (surface) is next.

## 34. W2 step 10's pure half: the fixed presentation contract

Step 10 is the surface path. Its first bounded piece landed as
`native::vulkan::surface`, the pure half: the fixed presentation contract decided
against what one surface reports, and the `VkSwapchainCreateInfoKHR` that decision
lowers into. It creates nothing and owns nothing, so every refusal is provable
without a window — which is the same reason step 1's validation probe and step 7's
barrier lowering were split this way. The surface handle, the swapchain, the acquire
lease, present, reconfigure and the unpresented-acquire quarantine remain owed.

### 34.1 The contract is fixed, and it refuses rather than negotiating

`R8G8B8A8_UNORM` with `SRGB_NONLINEAR`, `FIFO` presentation, `OPAQUE` compositing:
the same fixed triple the borrowed path being replaced configures. It is deliberately
**not** a preference order over the driver's lists. Choosing the closest available
format or mode would present something the graph did not compile against, and the
graph's compile-time format decision is what makes the surface texture's descriptor
correct. Each missing piece is a distinct sentence, because they need different
fixes: no such format at all, the format with another colour space, and the request
itself not being a size.

Two refusals carry a preserved semantic from plan section 4:

- **sRGB is not claimed.** `R8G8B8A8_SRGB` is a first-class portable format and does
  *not* satisfy this contract. The drawable's own encoding is not observable through
  this API, so accepting the sRGB format would apply a second gamma to bytes the
  compositor already decodes. A surface offering only the sRGB format is refused by
  name.
- **The extent is never invented.** A non-zero `current_extent` **is** the size the
  surface must be configured at, and the caller's requested size is not a competing
  input; only a zero `current_extent` — the specification's "you choose" — lets the
  request be used, clamped into `[min_image_extent, max_image_extent]`.

### 34.2 The numbers that are policy, not derivation

`IMAGE_COUNT` is 3, one more than the borrowed path's maximum frame latency of two:
the smallest count under which a presented frame and a frame still being recorded are
both in flight. It is clamped into the driver's own `[min_image_count,
max_image_count]`, and the lower bound is applied **first** — clamping in the other
order could return a count below `min_image_count` for a range the specification
forbids but a value can still express. `REQUIRED_USAGE` is `COLOR_ATTACHMENT`, which
is also the borrowed path's `TextureUses::COLOR_TARGET`; the caller's portable usage
is folded on top through `texture::usage_flags`, so a swapchain image is never less
capable than the graph declared and never more.

### 34.3 What the lowering deliberately leaves to its caller

`pre_transform`. A swapchain's `pre_transform` must name a transform the surface
reports as supported, and requiring it is a claim this increment does not need yet;
the swapchain-owning step owns that field. Everything else in the create-info is
written, including the fields whose defaults would silently claim something:
`image_array_layers` is one, `clipped` is true, and `old_swapchain` is null (a
reconfigure passes its own predecessor through the field). `image_sharing_mode` is
`EXCLUSIVE` for the same reason `buffer::create_info` states it: this backend has one
queue, and `CONCURRENT` would claim a cross-queue contract the execution model cannot
honour.

Not here either: the presentation-support query
(`vkGetPhysicalDeviceSurfaceSupportKHR`), which is device enumeration and therefore
step 2's rule's business; and the surface handle itself, which needs the instance to
enable `VK_KHR_win32_surface` first.

### 34.4 Proof

Thirteen pure tests cover the contract's constants by name (a change to any of them
changes what every surface presents), the present format pinned against
`format::image_format(TextureFormat::Rgba8Unorm)` **and** pinned as *not* the sRGB
format, the driver's `current_extent` winning over the request, the zero request and
the out-of-range request as their own refusals, a zero `max_image_count` meaning no
ceiling, the image count raised to the driver's floor and held under its ceiling,
each of the six missing pieces of the contract as its own sentence, the usage fold
reaching every one of the eight declared `TextureUsageKind` variants, and the
create-info's field-for-field contents.

Two compile iterations, both trivial and both worth the record: `SurfaceKHR` is a
non-dispatchable handle whose constructor `from_raw` comes from the `Handle` trait
rather than an inherent method, and the test-only usage list had to live inside the
test module so the library build does not import a name it never uses.

One of the increment's own tests was wrong rather than the code, in the same family
the previous increments recorded: the usage-fold test asked the *minimal* conformant
surface for every declared kind and was refused, because the minimal surface's
`supported_usage_flags` does not include `STORAGE`. That refusal is the code being
right; the test now asks a deliberately permissive surface and asserts the lowered
mask equals `texture::usage_flags(usage) | REQUIRED_USAGE`, which is what makes a
kind the vocabulary cannot lower a failure rather than a silently narrower swapchain.

The three required gates pass; `scripts/conformance.ps1` is the W2 close gate and is
not re-run for a mid-step increment.

Still owed by step 10: enabling the surface instance extensions, the `VkSurfaceKHR`
handle and its Win32 creation, the swapchain and its images, the acquire lease,
present, reconfigure, and the unpresented-acquire quarantine.

## 35. W2 step 10's surface-handle half: the extensions and the owned `VkSurfaceKHR`

Step 10's owning half began with the two clauses that make a surface exist at all:
the surface instance extensions and the `VkSurfaceKHR` created from the host's
window. They landed together because neither is useful alone — an extension enabled
for a surface that cannot be created is a claim with no object, and a surface cannot
be created without the extension. `scripts/conformance.ps1` remains the W2 close
gate and is not re-run for a mid-step increment.

### 35.1 The extensions are verified against the same inventory, and enabled only on the surface path

`surface::{SURFACE, WIN32_SURFACE, verify_surface_extensions}` is the pure half: it
checks the exact names against the enumeration `Validation::Required` already reads,
and reports *which* of the two is missing. `VK_KHR_surface` is checked first because
it is the facility itself, and a near miss on case or a prefix does not satisfy the
requirement for the same reason it does not satisfy the validation probe.

`instance::open_with_surface` is the FFI half: the same load → enumerate → verify →
create order as `open`, with both surface extensions enabled. The headless `open`
still enables neither, so nothing about the frozen oracle's instance changed.
`InstanceError::SurfaceExtension` is a new sentence whose three siblings already
existed, and both verifications precede `create_instance`, so a loader that cannot
present is refused with nothing created — the same fail-closed direction step 1
paid for.

### 35.2 The witness type, and why a flag would have been a crash

`vkCreateWin32SurfaceKHR` exists only when `VK_KHR_win32_surface` was enabled, and
`ash` substitutes a panicking stub for a function the loader did not resolve. So
"create a surface on an instance that never enabled it" must not be expressible:
`open_with_surface` returns `instance::SurfaceInstance`, a distinct type wrapping
`ValidationInstance`, and `presentation::create` accepts only that. A headless
`ValidationInstance` has no path to the surface call at all.

### 35.3 The surface borrows its instance; the window lowering refuses what it cannot name

`presentation::Surface<'a>` holds the `SurfaceInstance` by borrow and destroys its
handle in `Drop`. `Vulkan` requires the parent to outlive the child, so the borrow —
not a comment — is what makes "destroy the instance first" a compile error.

`presentation::win32_create_info` lowers the host's `RawWindowHandle`, and the
raw-window-handle enum is `#[non_exhaustive]`, so the mapping answers `Result`
rather than inventing a value for a platform this backend has not been taught. Two
refusals, distinct sentences: `NotWin32` for another platform's window, and
`MissingInstance` for a Win32 window whose owning module was not reported — the same
fact the borrowed path refuses (`wgpu-hal` `create_surface`), preserved rather than
defaulted to null.

### 35.4 What it cost

One compile iteration, and it is the already-recorded lesson applied again: `ash`
derives no `PartialEq` for `Win32SurfaceCreateInfoKHR`, so the two refusal tests
compare `.err()` rather than a `Result` — the same shape section 31.4 recorded for
`ImageSubresourceRange` and section 32.3 for `BufferCopy`. The `ash` 0.38 source was
read before the FFI, which is where `HINSTANCE`/`HWND` being `isize`, the generated
`khr::{surface,win32_surface}::Instance::new(entry, instance)` shape, and the
unresolved-function stub were settled.

### 35.5 Proof

Pure tests cover the extension drift guard (each enabled `CStr` still spells the
name the probe compares), both-present acceptance, the ordered base-then-platform
refusals, and a near miss on either name; and the Win32 lowering field for field plus
both refusals by name. `instance` adds the ordering test — a surface open refuses an
empty inventory with the surface reason before validation is even asked. Against the
real driver: a real `VkSurfaceKHR` is created from a real hidden `STATIC` window
through a real surface-capable instance and destroyed before both parents, and
`open_with_surface` opens a real instance on this machine.

The three required gates pass.

Still owed by step 10: the surface-facts query and the presentation-support query,
the swapchain and its images, the acquire lease, present, reconfigure, and the
unpresented-acquire quarantine.

## 36. W2 step 10's query half: the surface facts and the presentation-support answer

Step 10's surface path now reads the facts a swapchain decision needs instead of
waiting for the swapchain itself. `presentation::Surface::facts` reads the three
lists `surface::contract` already decides against --
`vkGetPhysicalDeviceSurfaceCapabilitiesKHR`, `...FormatsKHR` and
`...PresentModesKHR` -- and `presentation::Surface::supports_presentation` answers
the fourth question: whether **one queue family** can present to the surface.

### 36.1 The query does not decide, and the decision does not read the driver

The split is the one step 1 and step 7 already use: the pure half owns the rule, the
owning half owns the call. `surface::contract` is still the only place the fixed
presentation contract is decided, and it is still pure; `facts` produces its three
inputs and nothing else. `SurfaceFacts` carries owned vectors because `Vulkan`'s
format and present-mode enumeration is a two-call read whose count can change between
the calls, and `ash`'s `read_into_uninitialized_vector` retries on `VK_INCOMPLETE`
rather than truncating -- which is why the idiom is not re-implemented here.

A device that does not support the surface answers with empty lists rather than a
driver error, and that is reported as it arrived: `contract` is what turns an empty
format list into `FormatUnsupported`. `SurfaceQueryError` has one variant per call,
because the first three are one surface's own facts and the fourth is whether a
different object -- a queue family -- can present to it.

### 36.2 The presentation-support answer is the fact step 2's rule has to be told

`supports_presentation(physical_device, queue_family)` is a value about one surface,
not a capability row about the device: the same backend on another window can answer
differently. Step 2 chose the queue family by rule and without a surface, so this is
the first place that selection can be checked against presentation at all.

On the named board the rule holds: the selected graphics family is family 0 of five,
it reports presentation support, and two of the five families do. That is recorded
rather than assumed. What the query buys is that the swapchain-owning step can refuse
**before** a swapchain exists on a board where the rule does not hold, instead of
discovering it at `vkCreateSwapchainKHR`. Acting on the answer -- creating the device
on a family that can present when the rule's first graphics family cannot -- remains
the swapchain step's decision and is deliberately not here, because it changes step
2's contract and is only needed once there is a swapchain to create.

### 36.3 Proof

Against the real driver: a real surface created from a real hidden window reports
non-empty format and present-mode lists, `surface::contract` accepts them with the
contract's own format, colour space and present mode, every reported queue family
answers `supports_presentation`, and at least one answers `true`. The test does not
pass vacuously: once the surface exists and adapters were enumerated it **requires**
an adapter attached to that surface rather than skipping, so the assertions above are
reached on this machine and not bypassed by an early return.

The increment compiled with no iteration; the `ash` 0.38 source was read before the
FFI, which is where the three query signatures, their `VkResult` returns and the
retrying two-call helper were settled. The three required gates pass;
`scripts/conformance.ps1` is the W2 close gate and is not re-run for a mid-step
increment.

Still owed by step 10: the swapchain and its images, the acquire lease, present,
reconfigure, and the unpresented-acquire quarantine.

## 37. W2 step 10's swapchain half: the `VkSwapchainKHR` and its images

`native::vulkan::swapchain` lands the next bounded piece of step 10: the swapchain
object itself. It creates the `VkSwapchainKHR` from the fixed contract the pure half
already decides, reads the images `Vulkan` creates with it, and destroys it in
`Drop`. The acquire lease, present, reconfigure and the unpresented-acquire
quarantine remain owed.

### 37.1 The device extension had to be enabled, and the witness is what makes it safe

`device.rs` already recorded that `VK_KHR_swapchain` "belongs to step 10, not here",
and this is the step. Three pieces landed for it:

- `inventory::enumerate_device_extensions` reads one physical device's own
  extension names, reusing `inventory::collect_names` so a name that is not
  NUL-terminated or not UTF-8 still fails the whole enumeration. That rule is not
  restated: a skipped name could have been `VK_KHR_swapchain`, and the check below
  would then refuse a device that in fact has it.
- `device::verify_device_extensions` is the pure half: an exact, case-sensitive
  comparison against `SWAPCHAIN`, with `MissingDeviceExtension::Swapchain` as its own
  sentence. Step 1's reason applies unchanged -- a near miss is a different
  extension, and enabling one the physical device never reported is a creation
  failure with no diagnosis.
- `device::open_with_swapchain` reads the inventory, verifies it, and only then
  calls the same `create` the headless `open` uses, with the one extension enabled.

The witness is the point, not a convenience. `ash` substitutes a panicking stub for
a function the loader did not resolve, so "create a swapchain on a device opened for
headless work" must not be expressible: `open_with_swapchain` returns
`SwapchainDevice`, and `swapchain::create` accepts only that. A headless
`VulkanDevice` has no path to a swapchain call at all -- the same rule
`SurfaceInstance` already states for the surface entry points, applied to the device
half.

### 37.2 The queue family's presentation support is the fact step 2 could not read

Step 2 selected one queue family by rule and without a surface. Whether that family
can present to *this* surface is a fact about the pair, and `create` is the first
place it is checked: `supports_presentation` is read through the owned surface, and a
family that answers no is refused as `QueueCannotPresent { family }` **before**
`create_swapchain` exists. `Vulkan` would accept the creation and fail only at the
first present, which is exactly the shape a refusal value is supposed to prevent.

What this increment does **not** do is act on the answer by changing step 2's
selection rule. That changes the device's contract and is only needed once a present
call exists; the refusal is the honest bounded outcome until then, and it names the
family so the decision has a diagnostic.

### 37.3 `pre_transform` stopped being a deferred caller argument

The pure half's own doc said the transform was left to the caller "until the
swapchain-owning step states which transform it selected". This is that step, and it
selects the value the borrowed path being replaced configures:
`crates/wgpu-hal`'s `vulkan/swapchain/native.rs` writes `.pre_transform(IDENTITY)`.
So the decision moved to where the fixed contract already lives:

- `surface::PRESENT_TRANSFORM` is `IDENTITY`, pinned by a test beside the other
  contract constants;
- `surface::supports_transform` asks the driver's own report, and
  `PresentationError::TransformUnsupported` is a new sentence -- a surface that does
  not report the contract's transform is refused by name rather than presented at a
  substitute, which is the same direction the sRGB and extent rules already take;
- `Presentation` carries `pre_transform`, and `swapchain_create_info` lost its third
  parameter. A caller can no longer pass a transform the surface did not report.

### 37.4 Ownership, teardown, and what the images are not

`Swapchain` holds the [`Surface`] and the `SwapchainDevice` **by borrow**, so
"destroy the device or the surface before the swapchain" is a compile error rather
than a comment. `Vulkan` requires the swapchain to outlive nothing but its own
children, and the borrow states the two parents.

The images are read with `get_swapchain_images` and kept as handles. They are the one
resource in this backend that must **not** enter the resource table: the swapchain
owns them, `Vulkan` destroys them with it, and the table would try to release memory
that was never allocated for them. The module says so where the field is declared.

Every failure path undoes its own work. A refused image read destroys the swapchain
it just created, in the same order `create_texture` already uses; and a created
swapchain that reports no images is `SwapchainError::NoImages` rather than a
swapchain with nothing to present, because a driver that did that contradicted its
own contract.

### 37.5 Shared test scaffolding, extracted rather than copied

The real-driver tests need a hidden Win32 window and the "which enumerated adapter is
this surface on" search, and two modules now need both. They moved to
`native::vulkan::test_support` (`#[cfg(test)]`), so the Win32 FFI, the window's
`Drop`, and the window-station skip exist once. `presentation`'s tests import them
instead of defining them.

### 37.6 What it cost

The increment compiled with no iteration. The `ash` 0.38 source was read before the
FFI, which settled the three shapes that would otherwise have cost one: the manual
`khr::swapchain::Device` impl (so `create_swapchain`, `get_swapchain_images` and
`destroy_swapchain` are the exact calls, with `get_swapchain_images` already
retrying on `VK_INCOMPLETE` so the two-call idiom is not re-implemented),
`khr::swapchain::Device::new` taking the instance and the *loaded* device, and
`SwapchainCreateInfoKHR`'s field set and null-`p_next` default.

### 37.7 Proof

Pure tests cover the extension drift guard (the enabled `CStr` still spells
`SWAPCHAIN`), the acceptance of a reported extension, both near misses (case and
suffix) as their own refusal, the transform refusal beside the other five contract
refusals, the `PRESENT_TRANSFORM` constant, and the create-info's field-for-field
contents including that the transform now comes from the presentation.

Against the real driver: a real `VkSwapchainKHR` is created over a real surface
through a real device that verified and enabled `VK_KHR_swapchain`, its real images
are read (non-empty), and the contract's format is pinned against
`format::image_format(TextureFormat::Rgba8Unorm)` and its extent asserted non-zero;
and the device half has its own smoke test that asserts opening succeeds *exactly
when* the physical device reported the extension, so a creation failure after the
verification would fail rather than skip.

The three required gates pass. Because this increment reaches real hardware, the
GPU gate was run as well: `scripts/conformance.ps1` passes.

Step 10 still owes the acquire lease, present, reconfigure, and the
unpresented-acquire quarantine.

## 38. W2 step 10's acquire lease: the image, its semaphore and the quarantine

`native::vulkan::acquire` (the pure half) and `Swapchain::acquire` with `AcquireLease`
(the owning half) land step 10's next piece. Present, reconfigure and the recovery of
a poisoned surface remain owed.

### 38.1 The lease is a borrow, so exclusivity is not a run-time flag

`Vulkan` permits at most one acquired image that has not yet been presented or
discarded, and `swapchain`'s new doc calls that the lease rule. It is enforced by the
type rather than by a flag: `Swapchain::acquire(&mut self, _)` returns an
`AcquireLease<'s, 'a>` holding `&'s mut Swapchain<'a>`, so a second acquire cannot be
*compiled* while a lease exists, and `Swapchain::drop` cannot run either. That is the
same technique `OpenedVulkan` uses for the device/instance pair and `PipelineLayout`
for its descriptor set layouts: a dependency stated as field order or a borrow rather
than as a comment. `AlreadyAcquired` is therefore deliberately absent from
`AcquireError` — there is no run-time state that could disagree.

### 38.2 The pure half names each answer, because a driver cannot be asked for them

`acquire::outcome` lowers `ash`'s `Result<(u32, bool), vk::Result>` into
`AcquiredImage { index, suboptimal }` or one of five distinct sentences. The
distinctions are the point, and none of them is reproducible on demand against a real
driver:

- **`Timeout` is not a failure.** `VK_TIMEOUT` says no image became available within
  the caller's wait and the surface is fine, so the caller may ask again. Folding it
  into a driver result would turn a paced frame into a lost surface.
- **`NotReady` is not `Timeout`.** It is the answer to a zero-timeout poll: the same
  fact about availability, reached deliberately rather than by a wait running out.
- **`OutOfDate` is not `SurfaceLost`.** One is fixed by reconfiguring the swapchain,
  the other by giving up on the window system's surface; a single "surface problem"
  would erase the difference the caller has to act on.
- **Everything else keeps the driver's own value** as `Driver(vk::Result)` rather than
  being folded into a named case this backend has not been taught.

`VK_SUBOPTIMAL_KHR` is a *success* (`ash` maps it to `Ok((index, true))`) and its flag
is carried rather than dropped: the image is presentable, and the reconfigure decision
belongs to the step that owns `old_swapchain`.

### 38.3 The unpresented-acquire quarantine is where plan section 4 put it

`AcquireLease::drop` is the preserved semantic. Reaching it means the lease was
neither presented nor discarded (present is the only path that consumes a lease, and
it does not exist yet), so the presentation engine may still signal the acquire
semaphore at a time this backend cannot know: destroying it could free a handle the
driver still holds, and reusing it could hand the driver a semaphore that is already
signalled. The drop therefore **poisons the surface** and leaves the semaphore
undestroyed, which is the borrowed path's behavior stated without its
`std::mem::forget` bundle.

The flag lives on `Surface`, not on `Swapchain`, for a reason the next piece will
need: what section 4 quarantines is the *surface*, and reconfigure is about to replace
the swapchain. A flag on the swapchain would be a second opinion about a fact that
outlives it. `Surface::is_poisoned` is read by `acquire` **before** anything is
created, so a quarantined surface is refused without reaching the driver, and
`Surface::poison` is set by exactly the two paths that cannot prove the semaphore
reusable: an unpresented drop, and a driver that reports an image index outside its
own image list (`IndexOutOfRange`, the one contradiction that can arrive on the
*success* path).

The two paths that *can* prove something destroy the semaphore: a driver refusal means
the presentation engine never received it and it is still unsignaled, so
`acquire`'s error branch destroys it and leaves nothing behind. That asymmetry —
destroy on refusal, retain on unpresented success — is the whole rule, and it is
stated in the code rather than left to a reader to infer.

### 38.4 What it cost

One clippy iteration, and it is worth recording because the first version read as
correct: the quarantine was written as `core::mem::forget(semaphore)`, which is
`-D warnings`'s `forgetting_copy_types` error — a `vk::Semaphore` is a `Copy` handle,
so forgetting it does *nothing*, and the "retention" would have been an empty
statement that compiled only if the lint were off. The corrected shape says what
retention actually is for a handle: **not calling `destroy_semaphore`**. That is also
why `AcquireLease` holds a plain `vk::Semaphore` rather than an `Option` it would
`take()` and drop, and why there is no `Drop` body to suppress.

The wait duration is not re-derived: `submission::timeout_nanos` was made
`pub(crate)` and the acquire wait shares it, so the `u64::MAX` "wait forever" clamp has
one implementation. `ash`'s 0.38 source was read before the FFI, which is where
`create_semaphore`'s `VkResult<vk::Semaphore>` shape and the null-fence acquire call
were settled; the increment then compiled on the first attempt.

### 38.5 Proof

Pure tests cover the successful answer's index and suboptimal flag, the
timeout/not-ready pair, the out-of-date/surface-lost pair, the pairwise distinctness of
every named refusal, the driver-result passthrough, and that the two backend-owned
refusals are not driver results.

Against the real driver: a real `vkAcquireNextImageKHR` on a real swapchain returns an
in-range index, the leased image is the one that index names, the lease carries a real
non-null semaphore, the surface is not poisoned while the lease is live, and an
unpresented drop then poisons the surface and makes the next acquire answer
`AcquireError::Poisoned` **before** the driver is reached.

The three required gates pass. The increment reaches real hardware, so
`scripts/conformance.ps1` was run as well and passes.

Step 10 still owes present, reconfigure, and whatever reclamation a poisoned
surface's retained semaphore can have.

## 39. W2 step 10's present half: the present call and the retained wait semaphore

`native::vulkan::present` (pure) and `AcquireLease::present` with the swapchain's
retained-semaphore table (owning) land the next piece of step 10. Present consumes
the lease cleanly, and the semaphore it waited on is retained until the
presentation engine hands its image back. Reconfigure and the recovery of a
poisoned surface remain owed.

### 39.1 The pure half names each answer, and a suboptimal present is a present

`present::outcome` lowers `ash`'s `VkResult<bool>` -- read from the `ash` 0.38
source, where `queue_present` maps `VK_SUCCESS` to `Ok(false)` and
`VK_SUBOPTIMAL_KHR` to `Ok(true)` -- into `PresentOutcome::{Presented,Suboptimal}`
or one of three refusals. The distinctions are the point:

- **`Suboptimal` is not a refusal.** The image reached the presentation engine and a
  reconfigure is merely due, so the flag is carried as a value; the reconfigure
  decision belongs to the step that owns `old_swapchain`, exactly as the acquire's
  suboptimal flag already does.
- **`OutOfDate` is not `SurfaceLost`.** One is fixed by rebuilding the swapchain, the
  other by giving up on the window system's surface; a single "surface problem" would
  erase the difference the caller has to act on.
- **Everything else keeps the driver's own value** as `Driver(vk::Result)` rather
  than being folded into a named case this backend has not been taught.

No `p_results` array is passed. With one swapchain the call's own result carries the
same fact -- which is what the borrowed path being replaced reads -- and the three
`PresentInfoKHR` builder setters each overwrite `swapchain_count`, so one of each is
also the only shape where the three lengths cannot disagree.

### 39.2 The lease is consumed, and `mem::forget` is what discharges it

Present hands the lease's acquire semaphore to `vkQueuePresentKHR` as its wait, so
the semaphore is consumed by a queue operation rather than left pending. That is the
one path that discharges the lease, and it is stated by consuming `self`: the success
arm retains the semaphore on the swapchain and then **forgets** the lease, which is
what keeps its `Drop` from poisoning the surface after all. The lease is not `Copy`
and implements `Drop`, so the forget is a real suppression rather than the no-op that
forgetting a handle would be -- the distinction section 38.4 already paid for.

A refused present takes the other branch deliberately: it leaves the semaphore
pending, so the lease's own `Drop` runs and poisons the surface. That is the
unpresented-acquire quarantine reached through an error return, and it is the
fail-closed direction.

### 39.3 Returning from present is not proof its wait is consumed

A present that happened is not free either. The semaphore `vkQueuePresentKHR` waited
on **cannot be destroyed or recycled when the call returns**, because the
presentation engine may still be waiting on it; the one portable proof that it is
done with the image is `vkAcquireNextImageKHR` handing that same image index back.
That is the inference ANGLE records for its present semaphores, and it decides the
shape:

- `Swapchain::presented` is keyed by image index: present fills the slot its image
  names, and the next `acquire` of that image empties it, destroying the semaphore
  there and only there.
- `Swapchain::drop` makes the device idle first -- which completes every queue
  operation, present included -- and only then destroys whatever slots are still
  filled. That is exactly the borrowed path's teardown order (`vkDeviceWaitIdle`,
  then its semaphores), and it costs the steady-state path nothing because it happens
  only at teardown.

This is the "whatever reclamation a poisoned surface's retained semaphore can have"
that section 38 left open, reached from the other side: a *presented* semaphore has a
portable proof, an *unpresented* one does not, which is why the quarantine stays a
quarantine.

### 39.4 What is deliberately still not here

- **No reconfigure.** Present reports suboptimal and out-of-date as values; the
  `old_swapchain` rebuild and the recovery of a poisoned surface are the rest of
  step 10.
- **No draw submission.** Present waits on the acquire semaphore directly because
  nothing has consumed it yet. When the draw-and-present path lands, that submission
  waits on it and present waits on the submission's render-finished semaphore
  instead; the retention rule is about whichever semaphore present waited on, so it
  does not change.

### 39.5 Proof

Pure tests cover the success/suboptimal pair with its flag, the
out-of-date/surface-lost pair, the pairwise distinctness of every named refusal and
the driver-result passthrough.

Against the real driver: a real `vkQueuePresentKHR` presents an image acquired from a
real swapchain, the outcome is success or success-plus-suboptimal, the surface is
**not** poisoned -- which is what distinguishes present from the unpresented drop --
and the semaphore present waited on is asserted retained in the slot its image names.
A second acquire and present over the same swapchain then proves the surface stays
live and that whichever image the driver returns has its retained semaphore released
rather than reused or leaked.

The increment compiled with no iteration. The `ash` 0.38 source was read before the
FFI, which is where `queue_present`'s `VkResult<bool>` shape and the `PresentInfoKHR`
builder setters were settled; the semaphore lifetime rule was taken from ANGLE's
recorded present-semaphore inference rather than guessed, and the borrowed path's
`device_wait_idle`-then-destroy teardown was read before the owned teardown was
written. The three required gates pass, and because this increment reaches real
hardware `scripts/conformance.ps1` was run as well and passes.

## 40. W2 step 10 complete: the `old_swapchain` rebuild

`Swapchain::reconfigure` landed beside `create`, and step 10 is complete. The rebuild
is the same facts read and the same pure contract decision as a fresh creation -- the
two now share one private `build` -- and the only new input is the `old_swapchain`
handle, which `surface::swapchain_create_info` gained as a parameter instead of
writing `VK_NULL_HANDLE` itself.

### 40.1 The specification fact that decided the ownership shape

`VkSwapchainCreateInfoKHR::oldSwapchain` is not only a hint that lets the presentation
engine reuse resources: **the old swapchain is retired when it is passed, even if the
creation of the new one fails.** That single fact is why `reconfigure` takes `self`
rather than `&mut self`. If it took a borrow, a failed rebuild would leave the caller
holding a swapchain the driver had already retired, and the type would be saying
something about the object that is no longer true. Consuming it instead routes the
predecessor through its own `Drop` -- device idle, retained present semaphores, then
the handle -- on the success and failure paths alike, so there is no state in which a
retired swapchain is still believed live.

The refusals that happen *before* the driver is reached still happen before anything
is created: the queue family's presentation support, the facts read, and the contract
decision are all ahead of `create_swapchain`, and a refused image read still destroys
the swapchain it just made. What consumption changes is only the caller-visible shape
after the call: a rebuild either returns a live replacement or leaves no swapchain at
all, never a predecessor the driver has retired.

### 40.2 A poisoned surface refuses the rebuild, and that is the recovery boundary

`SwapchainError::Poisoned` is checked first, before any driver call. It has to be: a
rebuilt swapchain would assume the acquire semaphore an unpresented image left behind
is reusable, which is exactly what plan section 4's quarantine forbids. The
`presentation` façade already classifies every transition on a poisoned surface as
`RefusePoisoned`, so the backend and the façade agree rather than one of them being
permissive.

This also fixes where "the recovery of a poisoned surface" actually lives. It is not a
swapchain operation: the quarantine is a property of the **surface** -- the retained
semaphore belongs to the presentation engine's relationship with that `VkSurfaceKHR`
-- so recovery means replacing the surface (and its window), not rebuilding the
swapchain on top of it. That is why step 10's remaining wording was a refusal and not
a repair.

### 40.3 One mistake removed by adding a field

`reconfigure` used to need a `physical_device` parameter, because the contract must be
re-decided from the surface's *current* facts. A parameter would also have let a caller
ask a different adapter about a surface this swapchain did not create. `Swapchain` now
owns the adapter it was created on, so the rebuild reads the facts from the same device
the swapchain belongs to and the mistake is unrepresentable rather than checked.

### 40.4 Proof

Pure: `swapchain_create_info` is pinned for both meanings of the field -- null for a
fresh creation and the predecessor for a rebuild -- beside the existing field-for-field
assertions.

Against the real driver: a real swapchain acquires and presents a frame (which is what
leaves a retained present semaphore for the predecessor's teardown to retire), a real
`reconfigure` creates its replacement over that predecessor, the replacement's own
images are non-empty, its contract is the fixed one with a non-zero extent, it starts
with no retained present semaphore, and it acquires and presents a frame of its own
without poisoning the surface. A second test shows an unpresented acquire quarantining
the surface and the rebuild then answering `Poisoned`, before the driver.

The three required gates pass, and because this increment reaches real hardware
`scripts/conformance.ps1` was run as well and passes. The increment compiled with no
iteration; the `ash` 0.38 source was read before the FFI, and the retirement rule was
taken from the specification's own note on `oldSwapchain` rather than inferred from
the borrowed path -- which is what settled the ownership shape in 40.1 before it was
written.

Step 10 is complete. Step 11 (capability lowering) is next.

## 41. W2 step 11's per-format evidence half: the table and the query that fills it

Step 11 is capability lowering. Its first bounded piece landed as two modules --
`common::formats`, the evidence-carrying per-format fact table, and
`native::vulkan::format_facts`, which lowers one `vkGetPhysicalDeviceFormatProperties`
answer into those facts and records that answer for every format this backend maps.
The rest of step 11 -- the remaining ledger rows, and the lowering that hands these
facts to `fluxel_rendergraph`'s `DeviceCapabilities` -- remains owed.

### 41.1 The shared shape is the GL family's, promoted rather than translated

Section 9.7 recorded the finding that the native path's per-format facts
(`wgpu_hal::TextureFormatCapabilities` bitflags plus predicates) and the GL family's
(`GlFormatCapabilities`, an evidence-carrying row) are two different shapes rather
than two spellings of one. `common::formats` promotes the GL shape: the same eight
facts, an evidence value instead of a bitset, and the same two rules the GL table
already enforces -- a zero sample count describes no resource, and a storage fact
arrives only with operation evidence, because whether a format may be read or written
through a storage image is precisely what a device has to be asked. A conflicting
repeat is refused and an identical one is accepted, which is also the GL rule: two
discoveries agreeing is not a contradiction, and silently keeping the later of two
disagreeing rows would make the table depend on discovery order.

Two pieces of the GL table are deliberately not copied:

- **The compressed-format rule.** ADR-0011 refuses a compressed format any render,
  blend, storage or copy claim, and the GL table enforces it at `record`. The portable
  vocabulary cannot name a compressed format until W8a, so there is nothing to refuse
  yet; the rule is stated where it will be enforced rather than given a predicate with
  no format behind it.
- **The resource kind.** A renderbuffer is a GL-family resource rather than a texture,
  so the GL table keys on it. The shared table keys on `(format, sample count)`, which
  is the key section 9.7 fixed, and the GL backend keeps its renderbuffer rows as an
  internal detail mapping onto the same texture-shaped facts in W6.

### 41.2 The table is a sequence, because `TextureFormat` is not `Ord`

The portable format is `#[non_exhaustive]` and deliberately not `Ord`, so a
`BTreeMap` keyed by it would have to invent an ordering -- the same mistake section 24
already recorded for `ResourceId`, where the table was first written over a map type
the key did not satisfy. A `HashMap` would avoid the invented order but replace it
with a hasher-seeded one, and this table is compared and cached, so its iteration
order has to be a function of its contents.

It is therefore a `Vec` in insertion order, and that order is deterministic because
the caller records from its own fixed list: `format::MAPPED`, promoted from
`format.rs`'s test-only `ALL` so the list the format-evidence query iterates and the
list `image_format` lowers are one list rather than two that can drift. A test pins
what the type cannot: every listed format maps, and none is listed twice.

### 41.3 One sample count, because the query has no per-count dimension

`vkGetPhysicalDeviceFormatProperties` answers per format, not per sample count.
This backend refuses every multisampled image description
(`texture::image_create_info` returns `None` for `sample_count != 1`), so every fact
read here belongs to `SINGLE_SAMPLE` and the table's per-count key stays able to hold
the rows a later query proves. Recording a count this backend cannot create would be a
capability claim with no object behind it; the multisampled rows arrive with
`vkGetPhysicalDeviceImageFormatProperties`, which takes a usage and is the query that
actually proves them.

The lowering reads `optimal_tiling_features` and nothing else. Linear tiling and
buffer features describe resources this layer cannot create -- every image is
`OPTIMAL`, and no known format is a texel-buffer format -- so folding the three flag
sets together would claim capabilities for an image shape the graph cannot name. The
same reasoning drops the flags the portable table does not model (`BLIT_SRC`,
`BLIT_DST`, the texel-buffer and chroma bits) instead of mapping them onto a
neighbouring fact.

### 41.4 What reading the `ash` 0.38 source settled before the FFI

- `FormatFeatureFlags::TRANSFER_SRC` and `TRANSFER_DST` are **not** declared beside the
  rest of that type in `bitflags.rs`: they are added by the unconditional
  `VK_VERSION_1_1` block in `feature_extensions.rs`. A reader who checked only the
  bitflags definition would conclude the copy facts cannot be read at all. The
  vendored `wgpu-hal` uses them, which is what made the discrepancy worth resolving
  rather than assuming.
- There is no `SAMPLE_COUNT_n_BIT` format-feature constant anywhere in `ash` 0.38, and
  the specification's `VkFormatProperties` has no per-count dimension to read; this is
  what turned 41.3 from a preference into a fact.
- `vk::FormatProperties` derives `Default`, `Copy` and `Clone` but **not** `PartialEq`,
  so the tests assert fields rather than whole values -- the same family as the
  `ImageSubresourceRange`, `BufferCopy` and `Win32SurfaceCreateInfoKHR` notes.
- `get_physical_device_format_properties` returns `vk::FormatProperties` directly
  rather than a `VkResult`, because the specification gives every format an answer.

### 41.5 Proof

Pure tests cover each optimal-tiling flag lowering to its own fact, a real `Vulkan`
flag the table does not model (`BLIT_SRC`) lowering to none, both attachment kinds
reaching the one `renderable` fact, the single `STORAGE_IMAGE` bit proving both
storage directions, a storage fact passing the table's probe rule, a driver answer
with no flags being a proved negative the table keeps rather than an absent row, and
the table's own refusals: a zero sample count, a storage fact without operation
evidence, and a disagreeing repeat that leaves the first row standing. `format.rs`
also pins that the promoted list is taught and duplicate-free.

Against the real driver: every format this backend maps is queried once and recorded
in `MAPPED` order with `OperationProbed` evidence at sample count one, and the facts
`Vulkan` makes mandatory are asserted from the driver's own answer --
`R8G8B8A8_UNORM` sampled, renderable, blendable and copyable in both directions; its
sRGB sibling sampled and renderable; `D32_SFLOAT` renderable and copyable in both
directions and **not** blendable. A second discovery over the same adapter is accepted
and leaves the table unchanged, which exercises the identical-repeat rule against real
driver answers rather than synthetic ones.

The increment compiled with no iteration, and the `ash` source was read before the FFI
-- which is what found the `feature_extensions` location of the two transfer bits and
the absence of a sample-count dimension, neither of which a compiler or a unit test
would have reported.

The three required gates pass. The increment reads a real driver, so
`scripts/conformance.ps1` was run as well and passes; it creates and owns no GPU
object, so what it proves here is that the hardware fixture set is unchanged.

Still owed by step 11: the remaining ledger rows, and the lowering that hands these
facts to `fluxel_rendergraph::DeviceCapabilities`.

## 42. W2 step 11's lowering half: the discovery facts onto `DeviceCapabilities`

Step 11's other half landed as `native::vulkan::capability`: one pure function that
lowers the three discovery results this backend already owns -- the
`CapabilityLedger` `require` negotiates from, the adapter's `AdapterLimits`, and
`format_facts`'s per-format evidence table -- onto
`fluxel_rendergraph::DeviceCapabilities`. It creates nothing, asks the driver
nothing, and cannot fail, which is the same split step 1's probe and step 7's
barrier lowering use: the rule is pure, the call that feeds it is separate.

### 42.1 The ledger is the optional-domain input, not a second copy of the facts

The compute, storage-buffer and indirect rows are read from the ledger rather than
re-derived from the queue family and the limits. That is what keeps one discovery
answer: a row this backend has not recorded is a domain nothing has proved, so the
lowering reports the rejecting value automatically -- and starts reporting the fact
the moment the step that proves the row records it. It is also why this increment
does not need to add ledger rows: the lowering is total over whatever the ledger
holds, and the remaining rows are the separate, independently provable piece step 11
still owes.

### 42.2 Four fields are decisions rather than copies

- **Recording is `DeferredCommandBuffers`.** This backend records into a
  `VkCommandBuffer` (step 7) and submits it (step 9). The GL family's
  `ImmediateContext` is the other model, and the two are not interchangeable.
  `parallel_independent_encoders` is false because the one recording encoder is
  sequential.
- **Transitions are `GraphManagedExplicit`.** `barrier` lowers the graph's semantic
  access states onto `vkCmdPipelineBarrier`, so a graph transition *is* a backend
  operation here. The GL family keeps them `BackendManaged` because its `compat`
  layer mirrors state instead. This is the first place the two idioms are visibly
  different in the lowered value, rather than only in the implementation.
- **Copy is a queue fact and it is core.** `Vulkan` 1.0 guarantees buffer and image
  copies on a graphics queue and step 8 records both routes, so no probe had to
  establish it. The GL lowering states the same fact the same way.
- **The compute workgroup count is gated on the ledger's compute row.** A driver
  reports a non-zero count on every device, so copying it unconditionally would name
  a dispatch dimension for a device whose ledger refused the row. The pure test
  pins the discriminating pair -- the same non-zero adapter limits, reported only
  where the row was proved -- exactly as the GL lowering's own test does.

### 42.3 What is deliberately not claimed

- **No surface, and the queue does not present.** Presentation is a fact about one
  device/surface pair, not about a device: step 10 reads it through a live
  `VkSurfaceKHR` and the fixed contract. Reporting one here would be a claim about a
  window this call never saw. The surface facts join this lowering when the device
  owns both a surface and the format table.
- **No transient reuse.** `gpu-allocator` suballocates device memory, but nothing in
  this layer pools an object across frames, reuses one inside a frame or aliases two
  over one allocation, so all three rows keep the rejecting value.
- **`blendable` has no field to land in.** It is the one per-format fact
  `TextureFormatCapabilities` does not model. It is not lost in the fold -- the
  evidence table keeps it -- but the lowering cannot report what the contract cannot
  name, and the field arrives when a graph first asks about blending.

### 42.4 The format fold, and the one place the depth fact is asked

The evidence table is keyed by `(format, sample count)` and the contract is keyed by
format, so the boolean facts fold with `or` and the count-sensitive half is carried
by `attachment_sample_counts`, the only field shaped for it -- the same fold the GL
lowering performs. The evidence row's `renderable` covers both attachment kinds,
because `Vulkan` reports one flag for each and the portable table asks one question;
which side a format belongs to is asked of the *mapped* `Vulkan` format through
`format::is_depth`, so the depth fact keeps the single source of truth step 4
established. A portable format this backend cannot map has a driver answer but no
resource to attach, and is left as no claim rather than guessed at -- the same
direction `image_format`'s `None` already takes.

### 42.5 What it cost

The increment compiled with no iteration and its tests passed first run, including
the real-adapter one. Reading the `ash` 0.38 source before the FFI is what kept it
to zero, and the only new `ash` fact this increment used was one the previous
increment had already recorded (`FormatFeatureFlags`' transfer bits living in
`feature_extensions.rs` and the absence of a per-sample-count dimension). The one
non-obvious assertion is the real test's `filterable` on `R8G8B8A8_UNORM`: it is
mandatory with `SAMPLED_IMAGE_FILTER_LINEAR` under optimal tiling, so a failure
there would be a real disagreement rather than an optional capability this board
lacks.

### 42.6 Proof

Pure tests cover: the fail-closed floor (one raster/copy/no-present queue, no
storage, no indirect, zero workgroup dimensions, no surface); a proved compute row
reaching both the queue and the workgroup dimensions; both storage directions from
one storage row; each of the three indirect rows reaching the one buffer flag and
proving nothing about storage; the backend's own recording, transition,
synchronization, timestamp and transient shape; the colour count and the widened
alignment; the colour/depth attachment split; the sample-count fold; the storage and
copy facts; the absent-versus-recorded pair; and the report order.

Against the real driver: every mapped format is queried and recorded once, then the
ledger, the adapter limits and that table are lowered together. The assertions are
the ones `Vulkan` makes mandatory -- one raster/copy queue whose compute row equals
the ledger's, the adapter's colour count and alignment, the workgroup dimensions
only where compute was proved, `R8G8B8A8_UNORM` sampled, linearly filterable, a
colour attachment at one sample and copyable both ways, `D32_SFLOAT` a depth-stencil
attachment rather than a colour one and copyable both ways, and exactly the mapped
formats in table order.

The three required gates pass, and because this increment asks a real driver
`scripts/conformance.ps1` was run as well and passes. It creates and owns no GPU
object, so what the GPU gate proves here is that the hardware fixture set is
unchanged.

Still owed by step 11: the remaining ledger rows. Storage buffers, storage images,
the three query kinds, multiview, anisotropic filtering, base vertex, first instance
and the multi-draw rows each arrive with the step that proves them -- and this
lowering reflects each automatically, which is the point of reading the ledger rather
than re-deriving the facts. Step 11 is not complete.

## 43. W2 step 11's core-row half: the rows the created device already proves

Step 11's next bounded piece records three more rows in the ledger the lowering
reads: `Copy`, `IndirectDispatch` and `TimestampQuery`. All three are proved by
facts the device-open path had already read, which is what makes them one
increment rather than three: `get_physical_device_queue_family_properties` already
returned the family's flags *and* its `timestamp_valid_bits`, and the `Vulkan` API
version is the rest. No device feature was enabled, no extension structure was
queried, and the increment compiled without an iteration.

### 43.1 The one row that was claimed without being recorded

`capability::capabilities` wrote `QueueCapabilities::new(true, compute, true,
false)`: the queue's raster and copy flags were constants beside the queue, while
compute already read the ledger. Copy being a constant was not a deliberate
exception -- section 42.2 justified it as "the API version's own guarantee" -- but
it meant the lowering could report a capability the ledger did not carry, and a
caller reading the ledger and a caller reading the lowered value could disagree
about the same device. The three flags are now reads of the same kind: `raster` is
the `Graphics` row, `compute` the `Compute` row and `copy` the new `Copy` row. The
test that pinned the old shape (the floor case, where raster and copy were both
true with no optional row recorded) became the discriminating pair for the fix:
with only `Graphics` recorded `copy` is false, and recording `Copy` turns it true.

### 43.2 Three rows, one proof shape, and none of them needed a feature

- **`Copy`** is core: `Vulkan` 1.0 guarantees `vkCmdCopyBuffer` and
  `vkCmdCopyImage` on a graphics family, step 8 records both routes, and neither
  the selected family nor a device feature gates them. It has no numeric floor of
  its own, so `limits_satisfied` is stated `true` rather than borrowed from a
  neighbouring limit -- tying a row to a fact it does not depend on is how a
  second, unwritten rule gets introduced.
- **`IndirectDispatch`** is core, and the borrowed path being replaced calls
  `cmd_dispatch_indirect` with no gate at all. It arrives only beside a proved
  compute row, and it keeps that row's numeric floor, because an indirect dispatch
  is a dispatch: the two cannot disagree about whether this device can dispatch.
- **`TimestampQuery`** comes from the family's own `timestamp_valid_bits` report,
  which the specification fixes as either zero or a value in `36..=64`. `> 0` is
  the predicate, and the report is carried on `SelectedQueue` as the driver's
  number rather than flattened to a boolean at the selection site. The borrowed
  path spells the same rule as `>= 36`; the difference is only in what a driver
  reporting an undefined value would get.

`SelectedQueue` gained the field, and that is a reporting change rather than a
selection change: the rule is still "the first family whose flags contain
graphics", and a family with a zero timestamp report is selected exactly as
before. The doc says so where the field is declared, because the selector's
previous doc named only `queue_count` as the field it does not consult, which had
made "the selector consults the flags" the whole truth.

### 43.3 The rows that stay absent, and the three different reasons

The same increment fixed the boundary of what may be claimed later, because the
rows still owed are owed for reasons that are not interchangeable:

- **`StorageBuffer` and `StorageImage`** need a feature this backend leaves
  disabled: `fragmentStoresAndAtomics` and `vertexPipelineStoresAndAtomics` gate
  shader storage writes outside compute (the vendored `wgpu-hal` maps exactly that
  pair from its `FRAGMENT_WRITABLE_STORAGE` and `VERTEX_WRITABLE_STORAGE`
  downlevel facts), and `VkPhysicalDeviceFeatures` is opened all-zero. The
  storage-buffer row describes read *and* write, so claiming it without those
  features would be a claim a fragment-shader write falsifies. `StorageImage` adds
  a per-format storage fact this call does not consult.
- **`IndirectDraw`** is refused for the family-parameter-space reason section 20.1
  fixed: `IndirectDrawApi::draw_indirect` takes a draw `count`, and a count above
  one *is* `MultiDrawIndirect`, which needs the `multiDrawIndirect` feature. A
  backend that decomposed the count into single draws would be writing a second,
  unwritten implementation choice into a capability fact, which is what section
  20.1 forbids.
- **`MultiDrawIndirect`**, **`AnisotropicFiltering`**, **`Multiview`**,
  **`AsyncCompute` and `TransferQueue`** keep the reasons section 42 already gave:
  a feature that is not enabled, plan section 4's preserved closed row, and one
  queue.

A test asserts all eight stay both disabled and unexamined, which is the ledger's
two different sentences for "refused" and "never asked".

### 43.4 The lowering change, and what it now reads

`capability::capabilities` reads `raster`, `copy` and the timestamp placement from
the ledger, beside the compute, storage and indirect rows it already read.
Timestamps lower to `TimestampCapabilities::PassBoundaries` only where the row is
proved and stay `Unsupported` otherwise: a timestamp written by
`vkCmdWriteTimestamp` outside a render pass is exactly the pass-boundary placement
the graph contract names. `indirect_read` needed no change at all -- it already
folded the three indirect rows, so proving `IndirectDispatch` widened it without a
line being written, which is the point section 42.1 made about reading the ledger
rather than re-deriving the facts.

### 43.5 Proof

Pure tests cover: `Copy` proved at creation with a `NotRequired` probe and a
trivially satisfied floor; `IndirectDispatch` examined only beside a compute row
and disabled when that row's own floor is unsatisfied; `TimestampQuery` arriving
only from a non-zero valid-bit report, with a zero report leaving the row
unexamined rather than negative; the eight feature-gated and preserved rows absent
and unexamined; the selector carrying the timestamp report without consulting it;
and the lowering's discriminating pairs for the copy flag and the timestamp
placement. Against the real driver: the created device proves `Copy`, proves
`IndirectDispatch` exactly where the family reports compute, proves
`TimestampQuery` exactly where the family's valid-bit report is non-zero, reports
the queue's copy flag from the ledger, and still proves no storage buffer and no
indirect draw.

The increment compiled with no iteration, and the `ash` 0.38 source was read
before the field was added, which is where `QueueFamilyProperties::timestamp_valid_bits`
being a `u32` was settled.

The three required gates pass, and because the increment reads a real driver
`scripts/conformance.ps1` was run as well and passes.

Still owed by step 11: the storage rows (a device feature plus a per-format
storage fact), the occlusion and elapsed query rows, and the draw-parameter rows
(`BaseVertex`, `FirstInstance`) that arrive with the draw verbs and the family
markers that would let a caller require them.

## 44. W2 step 11's storage half: the shader-store features and the storage-buffer row

Step 11's next bounded piece landed: `native::vulkan::features` (the pure half),
the device-feature query and enablement in `native::vulkan::device`, and the
`StorageBuffer` row the created device now proves. `capability`'s lowering needed
no edit for the row itself -- it already read `ledger.supports(StorageBuffer)` --
which is the point section 42.1 made about reading the ledger instead of
re-deriving the facts.

### 44.1 One row, two stage-gated facts, and both are required

`Capability::StorageBuffer` describes a buffer a shader may read **and** write, and
`Vulkan` 1.0 gates the write half by *stage*: reading a storage buffer is core,
while a fragment-shader write needs `fragmentStoresAndAtomics` and a vertex-shader
write needs `vertexPipelineStoresAndAtomics`. `VkPhysicalDeviceFeatures` enables
neither by default, so the row could not be claimed before this increment without
authorizing a shader the device refuses -- which is exactly the claim section 43.3
said was missing.

`StoreFeatures::proves_storage_buffers` is therefore **both** halves, and the tests
pin each half alone as a refusal. Reporting the row from one half would be the
permissive direction: a row proved too early records a pipeline the driver rejects,
while a row left unproved refuses a graph the device could have run.

### 44.2 The requested set is the adapter's report narrowed, never a preference

`features::request` answers from `vkGetPhysicalDeviceFeatures` and nothing else. An
unreported feature stays disabled, and that is a hard rule rather than caution:
enabling one the adapter's `VkPhysicalDeviceFeatures` does not contain makes
`vkCreateDevice` fail with `FEATURE_NOT_PRESENT`, so "request it anyway and see"
turns a device that could rasterize into no device at all. A test asserts the
returned value is the *whole* feature list the create-info is handed, so a feature
that was reported and that no step establishes a row with -- `samplerAnisotropy`,
`multiDrawIndirect`, `independentBlend` -- still comes out disabled rather than
arriving by inheritance from a default.

`features::store` then describes the set that was **requested**, so `ledger` reads
the features the device was created with instead of a second reading of the
adapter. The two can only differ if a driver reported a feature and then refused the
device that enabled it, which is a `DeviceError::Creation` rather than a silently
unproved row.

### 44.3 The floor is the driver's own binding-size report

The row's numeric floor is `max_storage_buffer_binding_size != 0`: a device whose
maximum storage-buffer range is zero can bind none, and the ledger's rule is that
numbers which reject the domain cannot be recorded as a proved route. It is stated
as its own read rather than borrowed from a neighbouring limit, the same way the
`Copy` row's trivially satisfied floor is.

### 44.4 Proof, and what the real-driver half actually asserts

Pure: each half alone leaves the row unexamined; both halves prove it with a
`NotRequired` probe; a zero binding size leaves it examined-and-refused; every
other feature stays disabled; a reported feature the adapter lacks is not requested.
Against the real driver the device test reads the adapter's report again, puts it
through the same pure `request`, and asserts the device's stored feature set equals
that -- so the assertion is that creation enabled exactly what the rule decided, not
that the ledger agrees with itself -- and then asserts the row is proved exactly
where that set carries both halves. `capability`'s real test adds the discriminating
pair for the lowering: `buffers.storage_read == ledger.supports(StorageBuffer)`, and
both directions equal because one row serves them.

The increment compiled with no iteration. The `ash` 0.38 source was read before the
FFI, which is where three shapes were settled: `get_physical_device_features`
returns `vk::PhysicalDeviceFeatures` directly rather than a `VkResult`,
`DeviceCreateInfo::enabled_features` takes a **borrowed** `&'a
PhysicalDeviceFeatures` (so the feature set is a binding that outlives the
create-info, like the queue-info slice), and `PhysicalDeviceFeatures` derives
`Default` but **not** `PartialEq`, which is why the tests assert fields and why the
ledger reads a small `StoreFeatures` value instead of the `ash` struct.

The three required gates pass, and because the increment queries and creates a real
device, `scripts/conformance.ps1` was run as well and passes. It creates and owns
no GPU object beyond the device, so what the GPU gate proves here is that the
hardware fixture set is unchanged.

Still owed by step 11: `StorageImage` -- the same feature pair **plus** a per-format
storage fact, which needs the device to own a format table the ledger call has no
access to today, so it is not a one-line addition to this row -- the occlusion and
elapsed query rows, and the draw-parameter rows (`BaseVertex`, `FirstInstance`)
that arrive with the draw verbs and the family markers that would let a caller
require them.

## 45. W2 step 11's storage-image half: the device-owned format table and the second storage row

Step 11's next bounded piece landed: the device owns the per-format evidence table
`format_facts` fills, and the `StorageImage` row is recorded from it. The row needed no new
feature, no new device extension and no new FFI call -- `device::create` had the instance
and the physical device in hand and simply asked the format question the ledger had been
leaving to a caller -- which is what let it compile without an iteration.

### 45.1 The row is the same stage pair beside a resource fact, and the ledger had a field for each

`Vulkan` gates the write half of both storage domains by stage, so the row reuses
`StoreFeatures::proves_storage_buffers` as its route: without the pair nothing examined the
row, and a partial pair is not a route either, so the row stays unexamined rather than
half-proved. The resource half is a different kind of fact -- a storage image is an image
*in a format*, and whether a format may be used through one is what
`vkGetPhysicalDeviceFormatProperties` reports per format -- and the ledger already has the
field for it: `limits_satisfied`.

That mapping is the design of this increment, and it is what keeps the ledger's two
"refused" sentences different. A device created with the store pair on an adapter whose
formats none support storage is **examined and refused** (`limits_satisfied: false`); a
device created without the pair was never examined at all. The storage-buffer row made the
same split with the driver's binding-size report as its floor, so the two rows are now
structurally identical and W4 has a genuine repetition to look at rather than a second
shape invented for the second row.

`FormatTable::has_storage_read_write` is the floor's query, promoted from the GL family's
identically-named method (`webgl2/api/formats.rs`): **both** directions, because the table
splits read from write and neither implies the other, so a row proving one direction
describes half the domain the capability names. Its evidence is deliberately not
re-checked there -- `FormatTable::record` already refuses a storage fact that arrived
without an operation probe, so a second check would be a second definition of a rule the
table owns.

### 45.2 The device owns the table, because the ledger cannot be recomputed

The ledger is captured at device creation and never recomputed -- a query that re-read it
could answer about hardware that has since been replaced -- so a format fact discovered
after the device exists could not reach the row. The table therefore moved into the device:
`device::create` runs `format_facts::record_mapped` and stores the result beside the
ledger, and `VulkanDevice::formats` is what the capability lowering folds instead of a
caller recording the same driver answers a second time.

The read sits **before** `vkCreateDevice`, with the feature query but not with the device,
which is the same fail-closed order the validation probe uses: a driver that cannot answer
for a mapped format refuses the open with nothing created. `DeviceError::Format` is the
sentence for it.

### 45.3 What is deliberately not claimed

- **No descriptor-count floor.** `max_per_stage_descriptor_storage_images` would be a third
  input, and the row's own statement names two: the stage pair and the per-format fact.
  Borrowing a limit the row does not read would tie it to a fact it does not depend on, the
  same reasoning that keeps the `Copy` row's floor trivially satisfied. It joins the row
  with the step that needs the count.
- **No write-without-format.** `shaderStorageImageWriteWithoutFormat` is what a storage
  image needs when the *format is not known*, and every format this row's floor proves is
  one this backend names. Enabling it would widen the shader language for a resource the
  vocabulary cannot describe.
- **No compute-only shortcut.** Storing to a storage image from a compute shader is core,
  so a device without the store pair can still store from compute; the row stays unexamined
  there anyway, because it describes the read *and* write domain across the stages this
  backend's recipes use -- the same call the storage-buffer row made.

### 45.4 What it cost

Nothing. The increment compiled with no iteration and its tests passed first run, including
the real-adapter one, and no new FFI was written either: `format_facts::record_mapped` and
`get_physical_device_features` were already read from the `ash` 0.38 source by the two
increments this one follows, so section 23.4's rule was satisfied by reuse rather than by a
new reading.

### 45.5 Proof

Pure tests cover: each direction of the format query alone leaving
`has_storage_read_write` unsatisfied and a row proving both satisfying it; the device-level
discriminating set for the row -- no store pair (unexamined), a partial pair (unexamined),
and the pair with an empty table, with a read-only format and with a write-only format
(examined and refused by the floor) against the pair with a both-directions format (proved,
`NotRequired`); and the storage rows' removal from the unproved-row list, which still holds
the six feature-gated or preserved rows.

Against the real driver: the device test reads the adapter's feature report again, applies
the same pure request, and asserts the `StorageImage` row equals that pair **and** the
device's own table's answer -- not a re-query -- and that the table holds every format this
backend maps. On the named board the adapter enables both store features and reports
`STORAGE_IMAGE` for `R8G8B8A8_UNORM`, `B8G8R8A8_UNORM` and `R16G16B16A16_SFLOAT` but not
for the sRGB sibling or `D32_SFLOAT`, so the row is proved there rather than merely
consistent. `capability::capabilities` needed no edit for the row -- it already folds the
table it is handed -- and the lowering's real test now takes that table from the device
instead of recording it again, which is what "one discovery answer" means here.

The three required gates pass, and because the increment queries and creates a real device
`scripts/conformance.ps1` was run as well and passes (89 cases, one adapter).

Still owed by step 11: the occlusion and elapsed query rows, and the draw-parameter rows
(`BaseVertex`, `FirstInstance`) that arrive with the draw verbs and the family markers that
would let a caller require them.

## 46. W2 step 11's query-row half: occlusion and elapsed from facts the open path already read

Step 11's next bounded piece records the two query rows the step still owed:
`OcclusionQuery` and `ElapsedQuery`. Both are proved by facts `device::create` had already
read -- the selected family's flags and its own `timestamp_valid_bits` report -- so no new
FFI was written and the increment compiled without an iteration.

### 46.1 Occlusion is core on the graphics family, and the row is the imprecise one

`VK_QUERY_TYPE_OCCLUSION` is core `Vulkan` 1.0 on a graphics queue, so the row's route is
the same structural fact `Graphics` and `Copy` already have: a device exists on a family
whose flags contain graphics. Two details decide its shape:

- the row's own statement is "a query that reports **whether** any sample passed", which is
  the *imprecise* answer. The exact sample count is the `occlusionQueryPrecise` device
  feature, and this device enables no feature beyond the shader-store pair, so claiming the
  row without it would be claiming the feature rather than the query;
- like `Copy`, the row has no numeric floor of its own, so `limits_satisfied` is stated
  `true` rather than borrowed from a neighbouring limit. The pure test pins that with a
  ledger whose every reported number is absent: occlusion is still proved there, which is
  exactly what a trivially satisfied floor means.

### 46.2 Elapsed is the timestamp facility used twice, and nothing else

`Vulkan` has no elapsed query type. An elapsed interval is two `VK_QUERY_TYPE_TIMESTAMP`
writes and their difference, so the row's route is the selected family's own
`timestamp_valid_bits` report -- the fact `TimestampQuery` was recorded from in section 43.
Where the family reported no usable timestamps the row stays **unexamined**, which is the
same sentence the timestamp row already gives and is deliberately not a negative: nothing
proved the domain is absent.

The borrowed path being replaced draws the same line: `crates/wgpu-hal`'s Vulkan adapter
gates its timestamp feature set on `queue_props.timestamp_valid_bits >= 36` alone and
carries `timestamp_period` as a *value* it reports to callers rather than as a second
permission. So this increment adds no period field to `AdapterLimits`: the
tick-to-nanosecond conversion belongs to the step that first hands a duration in time units
to a caller, and a floor invented for it here would be a second rule about a fact the
capability does not name.

The two rows are still recorded separately because the ledger models a *dedicated* elapsed
facility -- the GL family's `TIME_ELAPSED`, whose `GlCapability::TimerQuery` covers elapsed
and timestamp together -- which this API reaches through its timestamp domain. That is a
genuine repetition rather than a duplicated rule, and it is what W4 will look at once DX12
proves the same pair.

### 46.3 What was deliberately not added

The **family markers** are not here. `family.rs` carries a `CapabilityFamily` marker only
for the domains a requirement names, and the query rows have never had one:
`TimestampQuery`'s row landed in section 43 without one, and the way that row reaches a
graph is the lowered `TimestampCapabilities`, not a requirement. The occlusion and elapsed
rows have neither a marker nor a `DeviceCapabilities` field yet, so this increment records
the ledger facts the step owed and leaves the vocabulary that would name them to the
consumer that needs it -- the same rule section 20.4 states.

Also still owed, and deliberately untouched: the draw-parameter rows (`BaseVertex`,
`FirstInstance`), which arrive with the draw verbs whose parameter space they gate and with
the family markers that would let a caller require them.

### 46.4 Proof

Pure tests cover both rows' discriminating pairs: occlusion proved in a ledger whose every
reported number is absent, with its floor asserted trivially satisfied, and elapsed
unexamined where the family's valid-bit report is zero against proved with a `NotRequired`
probe where it is non-zero.

Against the real driver the device test asserts occlusion is proved unconditionally -- the
graphics family the device was created on is its route -- and elapsed exactly where
`SelectedQueue::supports_timestamps()` is true. The fixture comment that explained the
storage rows' absence from the unproved-row list now names the query rows as proved too, so
that list still holds exactly the feature-gated and preserved rows.

The increment compiled with no iteration. The `ash` 0.38 source was read before the rows
were written, which is where the absence of an elapsed `QueryType` and
`PhysicalDeviceLimits::timestamp_period` being an `f32` were settled -- the first is why
elapsed is stated as the timestamp facility used twice, and the second is why no period
field joined `AdapterLimits`.

The three required gates pass, and because the increment's real-device test opens and drops
a Vulkan device, `scripts/conformance.ps1` was run as well and passes (89 cases, one
adapter).

## 47. W2's raster-pass lowering: the render pass the draw verbs will begin

Step 11 can no longer prove anything on its own. Its one owed item is the two
draw-parameter rows (`BaseVertex`, `FirstInstance`), and the step's own entry says
they arrive with the draw verbs whose parameter space they gate. Those verbs need a
`VkRenderPass` and a `VkFramebuffer` before any `vkCmdBeginRenderPass`, so the step
list gained **step 12 (raster recording)** and this increment landed its first
bounded piece: `native::vulkan::render_pass`, the pure lowering of the portable
attachment set onto the pass description the recording half will create. Step 11 is
recorded as complete for what it can prove; the draw-parameter rows move to step 12.

### 47.1 The preserved semantic is the GL family's, stated in the same shape

Plan section 4: a pass admits exactly one colour attachment at index zero and no
depth-stencil attachment. `render_pass::admit` mirrors
`webgl2/compat/device/pass.rs::admit` exactly -- the same deconstruction of the colour
slice, the same index check, the same depth refusal -- and the two refusals are
separate values for the same reason the GL family keeps two messages: an attachment
set with no colour target at index zero has no pipeline that could run in it, while a
depth-stencil attachment names a recipe no retained artifact declares.

The pipeline vocabulary can still *express* a depth-stencil attachment (step 5 lowers
one, and its real-driver test creates such a pipeline), and that is deliberate rather
than a contradiction: the refusal is visible at the pass, where no recipe declares
depth, instead of being hidden by a vocabulary that could not say it at all.

The subresource `range` is deliberately not decided here. It selects the image view
the framebuffer is built from rather than a field of the render pass, so `admit`
answers the pass question and the owning half will answer the view question.

### 47.2 The one shared fact, because the driver compares the two passes

`Vulkan`'s render-pass compatibility rules compare two passes by their attachment
formats and sample counts, and the creation pass `pipeline::create_raster` builds and
the recording pass this module describes must therefore agree. `color_description` is
now the one place a colour attachment's description is written, and
`pipeline::RenderPass::create` calls it for every colour attachment. Their contents
operations differ deliberately and that is why the operation is a parameter: creation
performs nothing and says `DONT_CARE`, while the recording pass states the operations
the graph compiled. Both layouts are `COLOR_ATTACHMENT_OPTIMAL`, which is the layout
`barrier::image_state` gives `ColorAttachmentWrite` (step 7), so the pass begins where
the graph's own barrier left the image rather than at a layout stated twice.

### 47.3 Why the create-info is not returned

`VkRenderPassCreateInfo` borrows its attachment and subpass slices, and the
`VkSubpassDescription` it holds borrows the colour-reference slice in turn, so a value
containing them could not outlive the locals that hold those slices. `PassAttachment`
therefore returns the pieces (`description`, `color_reference`, `clear_value`) and the
owning half assembles the create-info in the one scope where the borrows are valid. A
self-referential type would be the same borrow with an unsafe promise attached, and
the module says so where the decision is visible.

The clear payload is carried as the portable `[f32; 4]` rather than as a
`vk::ClearValue`, because that union has no `Debug` and is built where the begin-info
needs it. A non-clearing load writes zeroes rather than leaving the entry
uninitialized, because `VkRenderPassBeginInfo` indexes one entry per attachment.

### 47.4 What it cost

One compile iteration, and it is the trap the previous increments already recorded:
`assert_eq!` cannot compare a `Result<&RasterColorAttachment, _>`, because
`fluxel_rendergraph::RasterColorAttachment` derives neither `PartialEq` nor `Debug` --
the same family as the `ImageSubresourceRange`, `BufferCopy` and
`Win32SurfaceCreateInfoKHR` notes. The refusal tests compare `.err()`.

The `ash` 0.38 source was read before the lowering was written, which settled three
shapes a first draft would have guessed: `AttachmentDescription` derives `Copy`,
`Clone` and `Default` but **not** `PartialEq` (so the tests assert fields);
`ClearValue` is a union deriving `Copy` and `Clone` only, with no `Debug` (which is
why the payload is an array in `PassAttachment`); and
`SubpassDescription::color_attachments` borrows its slice, which is exactly what makes
returning a create-info impossible.

### 47.5 Proof

Twelve pure tests cover: the retained one-colour pass admitted; zero colours, a second
colour and a colour not at index zero each refused as `ColorTargets`; a depth-stencil
attachment refused as `DepthStencil`; the two refusals distinct; every `LoadOp` and
`StoreOp` variant lowered to its named `Vulkan` value; a clear payload carried only by
`Clear`; the colour reference naming slot zero; the clear value carrying the lowered
colour; and the description's field-for-field contents -- format, samples, both
load/store operations, both stencil operations at `DONT_CARE` and both layouts at
`COLOR_ATTACHMENT_OPTIMAL`.

The increment creates and owns no GPU object and calls no driver entry point, so the
three required gates are its whole verification of new behavior. `scripts/conformance.ps1`
was run as well, because it is the one gate that requires a clean committed worktree and
therefore had to follow the commit; it passes (89 cases, one adapter). What the GPU gate
proves for a piece that owns no GPU object is that the hardware fixture set is unchanged.
The creation-pass refactor in 47.2 is covered by gate 3 instead, where the real-driver
raster-pipeline tests create their render pass through `color_description`.

Still owed by step 12, which is now the first unfinished step: the framebuffer and the
pass begin/end on the recording encoder, the vertex/index/viewport/scissor and
pipeline/binding verbs, the draws, and then the draw-parameter rows step 11 handed to
it.

## 48. W2 step 12's framebuffer and pass bracket

Step 12's first owning piece landed: `native::vulkan::framebuffer`, which creates and
owns the `VkRenderPass` and `VkFramebuffer` one admitted attachment describes, and the
recording bracket on `command::Encoder` (`begin_raster` / `end_raster`) that begins and
ends it. The transition to the pass's initial layout is the graph's own barrier, so the
order a caller records is the one the frozen oracle already uses. The draw verbs remain
owed.

### 48.1 Two objects, one owner, because `Vulkan` fixes their order

`vkCmdBeginRenderPass` names a render pass and a framebuffer, and a framebuffer refers to
the render pass rather than the reverse, so the only shape that cannot be misused is one
owner holding both. `Framebuffer`'s `Drop` destroys the framebuffer first and the render
pass second, which is the same dependency order `PipelineLayout` states for the set
layouts it names. The framebuffer deliberately does not own the **view**: it is a handle
the resource table owns and must outlive the framebuffer, so the owner of both is the
caller, exactly as the encoder's pool is.

### 48.2 The two passes are comparable because they share one lowering

`Vulkan` compares the render pass a raster pipeline was created against with the one a
command buffer begins, by attachment formats, sample counts and reference layouts.
Section 29.3 already extracted `render_pass::color_description` for the creation pass;
this increment calls it, and `PassAttachment::color_reference`, from the recording pass,
so only the contents operations differ -- they are what the graph compiled. Neither pass
restates a format or a layout.

### 48.3 The refusals the framebuffer owns, and the one `admit` deliberately left to it

`render_pass::admit` refuses the attachment *set* (one colour at index zero, no
depth-stencil) and explicitly leaves the subresource range to the owning half, because
the range selects the image view rather than a field of the render pass. `Framebuffer::create`
is that half, and it refuses:

- a **subresource range** -- it is built from the texture's whole view, and a range the
  view does not address is a different resource than the graph named. The borrowed
  native raster path makes the same refusal, so this is a preserved semantic rather than
  a new one;
- an **unsupported attachment shape** -- the new pure `render_pass::framebuffer_extent`
  refuses a layered, multi-mip, volumetric, non-2D or zero-sized description. Those are
  `Vulkan`'s own rules for a framebuffer attachment: a three-dimensional view is not a
  legal attachment at all, and a layered or multi-mip view names subresources a
  non-multiview pass neither covers nor compares a pipeline against. The accepted shape
  is exactly the target shape the borrowed raster path preserves (`D2`, one mip, one
  layer, one depth slice);
- a portable **format** or **sample count** this backend has not been taught, the
  sample count lowered through `pipeline::sample_count` so there is one mapping that
  names `Vulkan`'s counts.

Every one of those is a value returned before a driver handle exists, and the one
failure that can happen *after* the render pass exists -- a refused framebuffer --
destroys the render pass before returning, so a refused target leaves nothing behind.

### 48.4 The bracket is state, and the commands illegal inside a pass refuse

`Encoder` gained one `pass_open` flag and three sentences. `end_raster` with no pass open
is `NoPass` and a second `begin_raster` is `PassAlreadyOpen`, refused rather than treated
as idempotent for the same reason a second `end` is. The interesting one is `PassOpen`: a
barrier, a copy and the end of the recording are all **illegal inside a render pass**, so
`transition_buffer`, `transition_image`, `copy_buffer`, `copy_texture` and `end` refuse
while one is open instead of recording a command the driver would reject. That is the
same separation the module already makes between "the encoder is not recording" and "the
state is wrong", and it keeps the graph's recording order -- transitions, then the pass,
then transitions -- the only one that is accepted.

The bracket supplies one clear value from the framebuffer's own attachment, because
`VkRenderPassBeginInfo` indexes one entry per attachment; the render area is the
framebuffer's own extent, which is the fact `Vulkan` requires to agree with the
attachments rather than a second parameter a caller could set differently.

### 48.5 What it cost

One compile iteration, and it is the recorded lesson about shared vocabulary types
applied to a new place: `RasterColorAttachment::texture` is a **reference** to the id, so
`admit`'s result had to be de-referenced before the table would look it up. Nothing else
moved. The `ash` 0.38 source was read before the FFI, which settled the three shapes a
first draft would have guessed: `FramebufferCreateInfo::attachments` derives
`attachment_count` from the slice (so the count cannot disagree with the views),
`RenderPassBeginInfo::clear_values` derives `clear_value_count` the same way, and
`layers` is zero by default -- a value `Vulkan` rejects -- which is why it is written as
one.

### 48.6 Proof

Pure: `render_pass`'s new tests pin the accepted single-layer 2D extent and each refused
shape (layered, three-dimensional, one-dimensional, multi-mip, a depth axis that is not
one, zero-sized); the existing admission, operation-lowering and description tests are
unchanged.

Against the real driver, in `framebuffer`'s own tests: a real colour target is transitioned
`Undefined -> ColorAttachmentWrite`, a real `VkRenderPass` and `VkFramebuffer` are created
from the admitted attachment, the bracket records over them, and the recording is
submitted and reported `Complete` -- so the recorded pass is one the driver actually
executes. The clear value is asserted from the lowered attachment, the extent from the
texture's own description, and the two refusals that reach no driver (a layered attachment
and a subresource range) are asserted as their own values. The bracket's state refusals
are asserted in the same file, including that a refused call leaves the recording usable
and still endable.

The three required gates pass. Because this increment creates and submits real GPU work,
`scripts/conformance.ps1` was run as well and passes.

Still owed by step 12: the vertex/index/viewport/scissor and pipeline/binding verbs, the
draws, and then the draw-parameter rows step 11 handed to it.

## 49. W2 step 12's state and draw verbs

Step 12 now records real draws. `native::vulkan::draw` is the pure half -- the
viewport, the scissor, the index type and the half-open ranges -- and the recording
encoder gained the seven verbs that use it: `set_raster_pipeline`, `set_viewport`,
`set_scissor`, `set_vertex_buffer`, `set_index_buffer`, `draw` and `draw_indexed`.
Binding selection (`set_bindings`, which needs the descriptor sets) is the one piece
of step 12 still owed.

### 49.1 The Y flip is a preserved semantic, and it needs a device extension

Step 6 emits the retained WGSL with `ADJUST_COORDINATE_SPACE` clear, matching the
borrowed `wgpu-hal` Vulkan path, so the emitted `BuiltIn::Position` is wgpu's Y-up
clip space. `Vulkan` maps a positive viewport height to the *bottom* of a
top-left-origin framebuffer, and the borrowed path reconciles the two by flipping the
**viewport** rather than the shader: `y = y + height`, `height = -height`
(`wgpu-hal` `vulkan/command.rs::set_viewport`). `draw::viewport` writes the same two
fields, so the frozen oracle's geometry is not flipped.

A negative viewport height is a validation error on a Vulkan 1.0 device unless
`VK_KHR_maintenance1` is enabled, and the instance this backend requests is 1.0. The
extension is therefore verified against the physical device's own inventory and
enabled on **both** device entry points, and the headless path now reads that
inventory where it previously read none. That is a change to step 2's contract, and
it is the right one: the borrowed path hides an adapter that cannot report the
extension (`wgpu-hal` `adapter.rs`), so requiring it is what the frozen oracle
already effectively does. `MissingDeviceExtension::Maintenance1` is the named
refusal, and `open_with_swapchain` verifies it beside `VK_KHR_swapchain` against one
inventory read.

The flip and the extension are one decision, and the `draw` module docs say so where
a reader looking for either will find them. Two pure tests pin the flip's two fields
and the offset it follows, which is what makes a later "simplification" that drops
the negative height fail instead of silently rotating every recipe.

### 49.2 The raster verbs belong to the open pass

All seven answer `RecordError::NoPass` while no pass is open. `Vulkan` permits most
of them outside a render pass, but it treats them as command-buffer state the *next*
pass inherits, and this backend's execution model has no such state to inherit: a
pass begins with exactly the state its own commands set. That is the same fail-closed
direction the bracket already takes for a barrier, a copy or an `end` inside a pass,
and it keeps one sentence for "this verb is not legal in this state".

### 49.3 The checks, and the one mapping that cannot be a wildcard

Viewport and scissor are refused before the driver when `Vulkan` would reject them: a
non-finite coordinate, a zero or non-finite extent, a depth outside `[0, 1]`, a
reversed depth range, a zero scissor and a scissor offset above `i32::MAX` (the
borrowed path's `as i32` would have wrapped it negative). An inverted vertex or index
range is refused for step 8's reason in a `u32`: `end - start` would wrap, so the
count is computed with `checked_sub`.

`IndexFormat` is this workspace's own closed enum, so `draw::index_type` is
exhaustive with no wildcard -- the opposite shape from a mapping over a
`#[non_exhaustive]` portable enum, which returns `Option`. The pipeline is bound
through `RasterPipeline::handle`, so the layout the pipeline owns stays alive for as
long as the binding can be used.

### 49.4 Shared test scaffolding, extracted rather than copied again

The recorder test needs the same minimal raster recipe the pipeline tests create
against, so `test_support` gained it -- the two Naga-emitted SPIR-V modules,
`colour_only_state`, `position_stream` and `raster_shaders` -- and `pipeline`'s tests
now import them instead of defining a second copy that could drift. That is section
37.5's rule applied to the fixtures rather than to the window.

### 49.5 What it cost

One compile iteration, and it is the lesson section 31.4 already recorded: `ash`
derives no `PartialEq` for `vk::Viewport` (unlike `vk::Rect2D`, `Offset2D` and
`Extent2D`), so the viewport-refusal test compares `.err()` rather than a whole
`Result`. One clippy correction: `clippy::reversed_empty_ranges` refuses a literal
`3..2`, so the inverted-range cases build their ends from locals -- the range is
lowered, never iterated.

The `ash` 0.38 source was read before the FFI, which settled the seven command
signatures (`cmd_bind_vertex_buffers` takes equal-length buffer and offset slices;
`cmd_draw_indexed` takes an `i32` vertex offset), the `Viewport`/`Rect2D` derives
above, and the fact that no maintenance1 command is called -- `cmd_set_viewport` is
core 1.0 and only the validation rule changes.

### 49.6 Proof

Pure: the flip's two fields and the offset it follows; the depth range carried
unchanged; eight viewport shapes the driver rejects; the scissor's field-for-field
lowering, the whole-attachment case, and four scissor refusals; both index types
distinct; the count-and-first pair, including an empty range as a legal no-op; and
two inverted ranges refused.

Against the real driver: a real `VkGraphicsPipeline` is bound inside a real pass over
a real `VkRenderPass`/`VkFramebuffer`, a real viewport and scissor are set, a real
vertex buffer and a real index buffer are bound, and both a `vkCmdDraw` and a
`vkCmdDrawIndexed` are recorded, submitted and reported `Complete` -- so the recorded
draws are work the driver actually executes, on a device created through the
maintenance1-verified path. A second test asserts the four `NoPass` refusals before
any pass exists, and the `Draw` refusals for a zero-height viewport, a zero-width
scissor and an inverted range once one is open, with the recording still endable
afterwards.

The three required gates pass. Because the increment creates and submits real GPU
work, `scripts/conformance.ps1` was run as well and passes (89 cases, one adapter).

Still owed by step 12: binding selection (`set_bindings`, which needs the descriptor
sets), and then the two draw-parameter rows (`BaseVertex`, `FirstInstance`) step 11
handed to it.

## 50. W2 step 12's binding selection: the descriptor set and `set_bindings`

Step 12's last owed verb landed: `common::binding` gained the bind-group *value*
vocabulary, `native::vulkan::bind_group` creates and owns the `VkDescriptorPool` and
`VkDescriptorSet` the retained textured layout fills, and `command::Encoder` gained
`set_bindings`, which records `vkCmdBindDescriptorSets`. The two draw-parameter rows
(`BaseVertex`, `FirstInstance`) are what step 12 still owes.

### 50.1 The value vocabulary is portable, and the layout check is what fills it in

`common::binding` now owns the value half of its own vocabulary, beside the layout
half W1 landed:

- `BindingResource` — `Buffer { buffer, offset, size }`, `Texture(TextureId)` or
  `Sampler(SamplerId)`. It names base resource ids, never a backend handle, so a raw
  handle cannot be written into one at all;
- `BindGroupEntry { binding, resource }`, and `BindGroupLayout::entry(binding)` so
  "which binding numbers exist" has one answer.

A texture entry is **one** variant for both the sampled and the storage case. Which
descriptor type and which image layout it lowers to is the *layout's* answer
(`BindingKind::Texture` versus `BindingKind::StorageTexture`), and a second spelling
in the entry could disagree with it. That is also why `common` is where the value
belongs rather than the native module: section 9.1 already assigns bind-group
vocabulary to `common`, and this is the same data model the next backend will consume
rather than two copies that can drift.

### 50.2 The set layout a group is created against is the pipeline layout's own

`Vulkan` requires the descriptor set a command buffer binds to be compatible with the
pipeline layout it was allocated against. The only shape that makes that a fact of
construction is allocating the set from the exact `VkDescriptorSetLayout` the pipeline
layout was built over, so `bind_group::create` takes `&PipelineLayout` and reaches the
handle through the new `PipelineLayout::set_layout(index)`. A second set layout
created from an equal description would be a compatibility question this layer cannot
answer.

That accessor needed the description too — validation and the pool sizes are read from
it — so **`SetLayout` now keeps the `BindGroupLayout` it was created from**. This
revises a sentence in `descriptor`'s own module docs ("never the description it was
built from"), and the revision is recorded there: the driver does not need the
description again, but the bind-group step does, and the alternative is asking a
caller that already moved its `SetLayout` into a `PipelineLayout` for a description it
no longer holds.

### 50.3 The pool is owned by the group, deliberately, and not pooled

`Vulkan` frees a descriptor set when its pool is destroyed, so one pool per group is
the smallest owner that makes the set's lifetime the group's: `BindGroup` owns the
pool, the set, the pipeline-layout handle and the set index, and its `Drop` destroys
the pool — which is the whole release, because the pool is created without
`FREE_DESCRIPTOR_SET`. A device-owned pool with a free list is the alternative, and it
is a *pooling* policy this backend deliberately does not have yet: the transient-reuse
rows are the rejecting value in step 11's lowering and nothing reuses a set across
recordings, so this type becomes a handle into the device's pool when a consumer needs
that recycling.

The pool sizes are one entry per *distinct* descriptor type with the bindings that
lower to it counted: `VkDescriptorPoolCreateInfo` rejects two sizes for one type, and
a second entry would silently make the total a lie.

### 50.4 Every refusal is a value, and a dynamic binding is refused

`bind_group::validate` and the range lowering run before the pool exists, so a refused
group reaches no driver entry point:

| Refusal | Sentence |
| --- | --- |
| an entry names an undeclared binding | `UnknownBinding` |
| a declared binding has no entry | `MissingBinding` |
| two entries for one binding | `DuplicateBinding` |
| the resource kind does not match the binding | `KindMismatch` |
| a layout minimum of zero | `ZeroMinimum` |
| a bind-time dynamic offset | `DynamicOffset` |
| a zero buffer range | `ZeroRange` |
| a range under the layout's minimum | `RangeTooSmall` |
| a range past the buffer, including overflow | `RangeOutOfBounds` |
| a stale buffer, texture or sampler id | `UnknownBuffer` / `UnknownTexture` / `UnknownSampler` |

`DynamicOffset` is the one deliberate refusal: this layer's vocabulary has no place to
state the offset a *bind* supplies, so a dynamic binding cannot be honoured, and
recording it with a fixed offset would hand the driver a descriptor it reads at the
wrong place. `ZeroMinimum` is the refusal `common::binding` already promised this
check would make. The range checks are the copy step's rule: the end is computed in
checked arithmetic, so the `u64::MAX` case is an overrun rather than a wrapped range
that passes.

The descriptor type comes from `descriptor::descriptor_type` — one function for layout
creation and bind-group creation — and a texture's image layout from
`bind_group::image_layout`, whose one format-dependent answer asks the *mapped*
`Vulkan` format through `format::is_depth`. A depth texture therefore binds through
`DEPTH_STENCIL_READ_ONLY_OPTIMAL` and a colour one through `SHADER_READ_ONLY_OPTIMAL`,
and a storage texture through `GENERAL`, which is the layout `barrier::image_state`
already gives the storage states.

### 50.5 What it cost

One compile iteration, and it is the trap sections 31.4, 32.3, 35.4 and 47.4 already
recorded: `ash` derives no `PartialEq` for `DescriptorBufferInfo`, so the four
range-refusal assertions compare `.err()` rather than a whole `Result`. The `ash` 0.38
source was read before the FFI, which settled three shapes a first draft would have
guessed: `WriteDescriptorSet::buffer_info`/`image_info` **derive `descriptor_count`
from the slice length** (so the count is not written separately), the two info structs
derive `Copy, Clone, Default` but no `PartialEq`, and `DescriptorPoolCreateInfo::max_sets`
is zero by default -- a value `Vulkan` rejects -- which is why it is written as one.

### 50.6 The real-driver test and the extracted fixture

The recorder's draw tests and the new bind-group tests both need a real colour target
and the framebuffer a pass begins over it, so that fixture moved to
`test_support::colour_target_pass` and the recorder's tests import it rather than
keeping the second copy. That is section 37.5's rule applied to the pass target.

Against the real driver: a real `VkDescriptorPool` and `VkDescriptorSet` are created
from the pipeline layout's own set layout and filled with the retained textured
layout's three real writes; a real raster pipeline over that layout is bound inside a
real pass, the set binds, a `vkCmdDraw` records, and the submission reports `Complete`
-- so the set the driver binds is one it actually reads. A destroyed buffer id and a
sampler placed at a buffer binding are each refused with their own sentence, an index
with no set layout behind it is `NoSuchSet`, an empty layout allocates a set from a
pool with no sizes, and `set_bindings` with no pass open is `NoPass` with the
recording still endable.

The pure half covers the pool-size fold (including two bindings of one type sharing
one entry), both image layouts and their format dependence, the four range refusals,
and every validation refusal above. The increment compiled with one iteration (the
`PartialEq` trap) and its tests passed first run. The three required gates pass.

Because the increment creates a descriptor set and submits real GPU work,
`scripts/conformance.ps1` was run as well and passes (89 cases, one adapter); it adds
no `#[ignore]` fixture, so the gate's counted fixture set is unchanged.

Still owed by step 12: the two draw-parameter rows (`BaseVertex`, `FirstInstance`)
that arrive with the family markers that would let a caller require them. The draw
verbs keep both fixed at zero until then, which is what plan section 20.1 requires.

## 51. W2 step 12 complete: the draw-parameter rows and their families

Step 12's last owed item landed: `Capability::BaseVertex` and
`Capability::FirstInstance` are recorded by `device::ledger`, and `common::api::family`
gained the two family markers that let a `Requirement` name them. Step 12 is complete,
and with it every step in `native/vulkan/mod.rs`'s ordered list.

### 51.1 The rows are the draw commands' own parameters, and the indirect feature is not theirs

`vkCmdDraw` and `vkCmdDrawIndexed` take `firstInstance`, and `vkCmdDrawIndexed` takes
`vertexOffset`, all core `Vulkan` 1.0 and none gated by a device feature -- so the two
rows are the same structural proof `Copy` and `OcclusionQuery` already have: a device
exists on a graphics family, step 12 records both commands on it, and no command was
needed (`NotRequired`). Neither row borrows a numeric floor: `limits_satisfied` is
stated `true` rather than tied to a limit the row does not read, the same shape the
copy row and the occlusion row already use. The pure test pins that with a ledger whose
every reported number is absent, which is exactly what a trivially satisfied floor
means.

The one nearby feature is deliberately not claimed. `drawIndirectFirstInstance` gates
the first instance of an *indirect* draw, and the `IndirectDraw` row that would name
that path stays unproved because its count is `MultiDrawIndirect` (section 43.3).
Recording `FirstInstance` from the direct command therefore claims nothing about the
indirect form, and the `ledger` doc says so where the row is recorded.

### 51.2 Two rows, not one advanced-draw row

`BaseVertex` and `FirstInstance` are separate ledger rows and separate markers, because
plan section 20.1's rule is one independently negotiable batch per family. A device may
add a base offset without supporting a non-zero first instance (and the reverse), so
neither requirement may be satisfied by the other's row; the test asserts both
directions rather than only that the rows differ, because "distinct" alone would not
catch a `ROW` that pointed at a third row both requirements happened to share.

The markers are markers, not traits, and that is the honest bounded outcome.
`GraphicsApi`'s draw verbs deliberately fix both parameters at zero -- a base family's
parameter space must not be able to name another family's capability -- so the verbs
that name them are separate families, and no retained recipe declares either one.
Section 4.7 of the interface contract records the pair beside `Multiview`,
`AsyncCompute` and `TransferQueue` for the same shape of reason. Writing their methods
now would be vocabulary ahead of its consumer, which section 20.4 forbids; and no real
backend implements `Provides<Graphics>` yet, so the handle those traits would hang off
does not exist. The rows are proved and the requirement can be stated; the verbs arrive
with the consumer that needs them.

### 51.3 Proof

Pure: both rows are proved with a `NotRequired` probe and a trivially satisfied floor
in a ledger whose every reported number is absent, and the two families' requirements
are satisfied only by their own row, in both directions. Against the real driver, the
device this machine opens asserts both rows, beside the `IndirectDraw` row that stays
unproved.

The increment compiled with no iteration, and no FFI was written: the rows restate a
property of the `vkCmdDraw` / `vkCmdDrawIndexed` signatures the step 12 draw verbs
already call, so section 23.4's "read the `ash` source first" rule was satisfied by the
previous increment's reading rather than by a new one.

The three required gates pass. Because the increment reads a real driver,
`scripts/conformance.ps1` was run as well and passes.

The ordered step list is now exhausted, so section 23.1's rule moves to section 20's
table: **W2's own closure** is the next piece -- wiring `VulkanDevice` to the family
traits (`Provides<Graphics>` first), so the frozen oracle can run on this backend and
`scripts/conformance.ps1` becomes its acceptance rather than a check that the hardware
fixture set is unchanged.

## 52. W2's first family wiring: the device owns its table, and `Provides<Graphics>`

Section 51.3 named W2's own closure as the next piece, `Provides<Graphics>` first.
That landed as `native::vulkan::family`, and it is the first place the common layer's
negotiation runs against a real backend rather than only against the mock: the step
list gained step 13, because section 23.1's rule had moved to W2's work package.

### 52.1 The device owns the table and the pool, because `provide` takes only `&self`

The wiring could not be written before the ownership moved. `Provides::provide(&self)`
builds the handle, and a `GraphicsApi` verb is stated in `BufferId` / `TextureId`
terms, so the handle's only route to a driver handle is a table the *device* owns.
`VulkanDevice` therefore gained two fields, built in `create` where the instance,
adapter and device are all in hand: the `ResourceTable` the resource steps already had
(one `GpuAllocator` over which buffers, textures, views and samplers live) and the one
`CommandPool` on the selected queue family.

That made teardown order load bearing for the first time. A type with a manual `Drop`
runs that body **before** its fields are dropped, so the previous
`impl Drop for VulkanDevice { destroy_device }` would have destroyed the device before
the table and pool that have to be released first. The fix is a field rather than a
rule: the raw `ash::Device` moved into a private `OwnedDevice` newtype whose own `Drop`
destroys it, and the declaration order -- `table`, `pool`, `device` -- is the release
order. `ash::Device`'s `Clone` is a handle and a function table, never a second
ownership claim, so the clones the table, the pool, every encoder and every framebuffer
hold stay valid until that last field runs.

`GpuAllocator` gained `from_handles`: the table is built before the `VulkanDevice`
value exists, so the constructor that took `&VulkanDevice` could not be used there. The
two share one body, so the descriptor -- including `buffer_device_address: false`,
which used to be the only thing keeping the allocator off a feature this device never
enables -- is still written once. `DeviceError` gained two sentences, `Allocator` and
`CommandPool(RecordError)`; the first deliberately drops `gpu_allocator`'s inner
`AllocationError`, which is neither `Copy` nor `PartialEq`, exactly as
`ResourceError::Memory` already flattens it at the allocation boundary. Both are now
reachable *after* `vkCreateDevice`, so each early return destroys the device it was
handed -- and the pool failure drops the table first, because an early return does not
get field order.

### 52.2 The handle is the recording context, and the recording begins lazily

`family::GraphicsRecording<'d>` borrows the device and owns one recording and every
pass target that recording names. Two decisions were forced rather than chosen:

- **The recording begins on the first command that needs one**, not in `provide` and
  not in `begin_raster` alone. `provide` cannot report a failure and allocating a
  command buffer can fail; and the graph records its transitions *before* it opens the
  pass, so a recording begun only by `begin_raster` would refuse the transition that
  makes the pass valid. This was found by the real-driver test, whose first attempt
  failed with `Recording(NotRecording)` on the transition rather than by review.
- **A finished handle refuses.** The state is a three-variant private `Stage`
  (`Fresh` / `Recording(Box<Encoder>)` / `Finished`) rather than an `Option`, because
  an `Option` alone would let a handle whose recording was already handed to submission
  silently allocate a second one nobody would submit. Clippy's `large_enum_variant`
  is what boxed the encoder: it carries a loaded device function table and is some
  1.5 KiB, and one allocation per recording is nothing beside the driver calls it
  enables.

`begin_raster` runs the preserved attachment rule first -- one colour at index zero, no
depth-stencil -- and only then resolves the attachment's texture, so a pass with no
pipeline that could run in it is refused for that reason rather than for whatever its
first attachment happened to name. The `VkRenderPass` and `VkFramebuffer` then come
from the existing `framebuffer::create`, and the created target is kept in the handle
rather than dropped at `end_raster`, because the recorded `vkCmdBeginRenderPass` names
it until the commands referencing it complete.

Every id is resolved through the device's table, whose key is the whole stamped
`ResourceId`. A foreign or replaced generation therefore has no record, so it is
refused as `UnknownTexture` / `UnknownBuffer` before the driver without the handle
checking for it separately -- the same guarantee
`common::api::handle::verify_texture` states, reached through the map key.

### 52.3 Transitions and submission are deliberately not family vocabulary

A pipeline barrier is a backend mechanism and the common contract carries none (plan
section 1), and the contract has no submission verb yet. So
`GraphicsRecording::transition_buffer` / `transition_texture` and
`GraphicsRecording::finish` are crate-private methods rather than `GraphicsApi` verbs,
and the module says so where they are declared. `finish` takes `&mut self` rather than
`self` for a reason the ownership rule dictates: the pass targets stay in the handle,
so the caller must keep it alive until the submission that names them reports terminal.

### 52.4 Proof

Against the real driver: `require::<_, Graphics>(&device)` yields the handle and it
reports the device's own stamp; the handle records the graph's transition, the pass
bracket, the pipeline, the viewport and scissor, a real vertex and index buffer and
both draws; and the submission reports `Complete`, with the handle still alive so the
framebuffer outlives the work that names it. The second fixture asserts the refusals:
a depth-stencil attachment as `Pass(DepthStencil)`, a foreign texture id as
`UnknownTexture`, a foreign buffer id as `UnknownBuffer`, a zero-height viewport as the
lowering's own `Draw` sentence, and the same `NoPass` the encoder gives for a raster
verb with no pass open -- with the recording still usable and still endable afterwards,
and a finished handle answering `NotRecording` rather than beginning a second
recording.

The increment compiled with no iteration; `rustfmt` reflowed three lines and clippy's
one `large_enum_variant` was the only correction. The `ash` 0.38 source had already
settled the `Clone`-without-`Drop` shape of `ash::Device` in earlier increments, and
that is what made the field-order teardown safe to write rather than guess.

The three required gates pass, and because the increment creates and submits real GPU
work `scripts/conformance.ps1` was run as well and passes.

Still owed by W2's closure: the remaining families on this backend (`CopyApi` next,
since every verb it needs is already recorded), and then the execution-layer migration
that lets the frozen oracle run on this backend.

## 53. W2's second family wiring: `CopyApi`, and the type is the permission check

Section 52 named `CopyApi` as the next piece of W2's closure "since every verb it needs
is already recorded". It landed as `CopyRecording` and `Provides<Copy>` in
`native::vulkan::family`, so step 13 -- the family-wiring step -- now covers two
families, and it is the first place the plan's one-handle-type-per-family rule has a
second instance to prove itself against.

### 53.1 Two handle types, because the handle type is what bounds a call site

`GraphicsRecording` and `CopyRecording` are distinct types, and that is a correctness
decision rather than a naming preference. A family verb does not re-ask the ledger once
a handle exists -- that is the whole point of negotiating once (plan section 11.2) --
so a single type implementing both `GraphicsApi` and `CopyApi` would let a caller that
negotiated only `Copy` reach a draw, without any row having been proved for it. Keeping
the traits on separate types makes "a copy handle cannot draw" a fact of the type
system rather than a run-time check, which is exactly the shape section 9.5's two
refusals describe: the backend *has* the graphics vocabulary, so the refusal is that
this handle cannot name it, and no call site can be written.

Both handles share one `Recording` owner, which is where the lazily begun `Encoder` and
the `Fresh` / `Recording` / `Finished` states live. How a handle reports the recording
is the only part the families differ in: `GraphicsRecording` maps it through
`GraphicsError::Recording`, `CopyRecording` through `CopyError::Recording`, so each
family keeps its own sentence for a command the recorder refuses.

### 53.2 The extraction is real duplication avoided, one increment after it appeared

Section 52's `GraphicsRecording` held the `Stage` machine directly. The second family
would have needed a copy of it, so the machine moved into the private `Recording` type
both handles now own. That is plan section 3's rule applied one level down: the
extraction happened when two concrete consumers existed, not when the second was
imagined. The alternative -- a near-identical `Stage` plus a near-identical lazy-begin
under a second name -- would have been the "second spelling of one shape" the plan keeps
refusing, and W4 would have had to reconcile it.

One small API change came with it: `Recording::device` returns the handle's `'d`
rather than a borrow of the recording, so a family verb can resolve its ids through the
device's table and then take the recording mutably without the two borrows being
entangled. That is a lifetime statement, not a clone.

### 53.3 Transitions are backend mechanism, and a copy-only recording needs them

`CopyRecording` exposes crate-private `transition_buffer` / `transition_texture`, for
the reason `GraphicsRecording` already does: a pipeline barrier is a backend mechanism
and the common contract carries none (plan section 1), so it is the graph's entry point
rather than a `CopyApi` verb. A copy-only recording needs it *more* than a raster one
does, because `Encoder::copy_texture` names `TRANSFER_SRC_OPTIMAL` /
`TRANSFER_DST_OPTIMAL`, so an image that was never transitioned is not in either layout
the copy command requires.

Both transition methods resolve the id and lower the mapped `Vulkan` format through the
same `format::image_format` the graphics handle uses, and the error is the copy family's
own `UnknownBuffer` / `UnknownTexture` rather than a nested recorder sentence -- so a
caller reads one layer's answer, not a wrapper around another's.

### 53.4 The table, not the driver, supplies what a copy is checked against

`copy_buffer` reads both handles *and both declared sizes* from the device's table. The
sizes are the ones the buffers were created with -- the same values the graph's own
region check used -- rather than sizes recovered from the driver, so the boundary cannot
check a region against a fact the graph never saw. `copy_texture` reads both images and
both `TextureDesc`s the images were made from, because every decision the pure lowering
makes (the aspect, the mip bounds, the layer rule, the format equality) is derived from
those descriptions in `native::vulkan::copy`; this call spells none of them.

A stale or foreign id has no record whose key is the whole stamped `ResourceId`, so it
is refused as `UnknownBuffer` / `UnknownTexture` before the driver -- the same guarantee
`common::api::handle::verify_texture` states, reached through the map key. A region the
driver would reject keeps the recorder's own `Region(CopyRegionError)` sentence, because
"the id names nothing" and "this range is misaligned" are different mistakes to fix.

### 53.5 The recording is still per handle, and composition is not this increment's

Each handle owns its own recording, so a graph execution that names two families
negotiates two handles and produces two recordings. That is what the contract permits
and does not require (plan section 21: negotiation and recording context need not remain
the same object), and it is the honest bounded outcome: `Vulkan` requires one command
buffer per submit, and composing several families' recordings into one submission is the
execution-layer migration's decision. Inventing a device-owned shared recorder here
would be designing that migration's ownership before the migration exists, which plan
section 3 forbids; the module docs say so where a reader looking for composition will
find it.

### 53.6 Proof

Against the real driver: `require::<_, Copy>(&device)` yields the handle, it reports the
device's own stamp, the handle records the graph's own buffer and image transitions and
then both `vkCmdCopyBuffer` and `vkCmdCopyImage`, and the submission reports `Complete`.
The second fixture asserts the refusals: a foreign buffer id as `UnknownBuffer`, a
foreign texture id as `UnknownTexture`, a misaligned region as the recorder's
`Region(Misaligned)`, and a finished handle answering `NotRecording` rather than
beginning a second recording -- with the recording still usable and still endable after
every value refusal, which is what the `ExecutionBackend` contract requires of a
callback error.

The increment compiled with no iteration and its tests passed first run; no new FFI was
written, so section 23.4's "read the `ash` source first" rule was satisfied by the step
8 increment's earlier reading of the two copy signatures rather than by a new one.

The three required gates pass. Because the increment creates and submits real GPU work,
`scripts/conformance.ps1` was run as well and passes.

Still owed by W2's closure: the remaining families this backend can serve -- `Compute`
next, whose row is already proved but whose recording verbs (`begin_compute` /
`end_compute` / `set_compute_pipeline` / `dispatch`) are not yet on the encoder --
the storage-role and indirect-dispatch families after it, and then the execution-layer
migration that lets the frozen oracle run on this backend.

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
