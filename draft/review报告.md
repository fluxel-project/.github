# Historical RHI Architecture Review

> Historical review, retained for its investigation record. It is not a current
> plan or API specification. The current RHI authority is the independent
> `fluxel-rhi` repository; the current ecosystem plan is
> [ROADMAP.md](../ROADMAP.md). Capture/replay is deferred and undecided.

## Purpose and outcome

This review evaluated an earlier proposal to place portable GPU contracts,
RenderGraph vocabulary, and five backend families under one architectural
model. It found that portability requires shared semantics and explicit
capability facts, not a lowest-common-denominator native API.

The durable conclusions are:

- backend-native handles, barriers, synchronization objects, descriptor
  allocators, and swapchains stay private;
- resource ownership is device or context identity plus generation;
- a resource handle dropping on the CPU never proves that its GPU realization
  is safe to destroy;
- graph planning owns dependency, access, transition, scheduling, and lifetime
  analysis; the RHI executes its portable result;
- presentation is a lease over a frame destination, not universally a sampled
  texture;
- each optional capability is an observed device fact with a structured refusal
  path, never a claim inferred from a backend name; and
- a real target proves only the selected backend, driver, browser, or host.

The prior proposal contained obsolete crate ownership and API assumptions. In
particular, the current target keeps RenderGraph internal to `fluxel-renderer`,
uses separate `fluxel-material` and `fluxel-shader` crates, and consumes the
stable independent RHI repository.

## Platform findings

### Device acquisition

Desktop explicit APIs can enumerate adapters and open devices synchronously.
WebGPU commonly requests an adapter and device asynchronously. GL, GLES, and
WebGL commonly receive a host-created context instead of enumerating adapters.
One universal synchronous `enumerate_adapters` entry point therefore produces
false behavior on real platforms.

The portable boundary must permit enumeration where it exists, asynchronous
request where it is necessary, and host-provided device or context adoption
where enumeration is unavailable. None of those paths may expose browser
session or token concepts in the public resource model.

### Resource identity and lifetime

Buffers, textures, views, samplers, and pipeline state must be checked against
the device or context identity and generation that created them. Device loss,
context restoration, or recreation advances the generation; stale or foreign
resources fail in the library before backend calls.

Logical assets and physical GPU resources are separate lifetime domains.
Logical asset handles provide identity and content generation. Renderer-side
residency maps that generation to a device generation, tracks upload and last
use, and retires native objects only after completion evidence covers their
last submission.

### Presentation

Vulkan swapchain images, Metal drawables, WebGPU current textures, and GL
default framebuffers have different native forms. The common semantic is an
acquired presentation-frame lease with a frame attachment. A backend may expose
an optional drawable view when it actually has one, but GL or WebGL default
framebuffers must not be represented as ordinary textures.

When a graph needs a final result that the target cannot sample or copy, it
uses an intermediate texture and a supported final transfer or blit route.
Presentation ordering belongs to the submission plan; it is not universally a
post-submission `Queue::present` call.

### Recording and capability facts

Portable recording should describe resources, pass access, bindings, draw,
dispatch, copy, submission, and completion. Backends lower that vocabulary to
their native command systems. Public callers do not construct native barriers
or synchronization primitives.

Capability reporting must distinguish unexamined, unsupported, supported, and
enabled states where those distinctions matter. A backend family label cannot
prove a route. Format capability is especially multidimensional: format, usage,
access, sample count, profile, extension, and target evidence can all affect
the answer.

The review retained a small baseline: resource creation, basic raster work,
basic copies, bindings, one ordered submission lane, completion, and
presentation. Mapping, queries, advanced indirect execution, descriptor
indexing, external interop, sparse resources, device address, multi-device
operation, and advanced pipeline families remain deferred until real consumers
provide semantics and target evidence.

## Ownership review

The review separated five responsibilities:

| Responsibility | Durable boundary |
| --- | --- |
| Platform acquisition | The platform/host adapter obtains a window, display handle, context, or request mechanism. |
| RHI | Portable GPU vocabulary, device facts, native realization, recording, submission, completion, and presentation. |
| Renderer | Scene preparation, GPU residency, material/shader selection, internal graph construction, and frame execution. |
| Internal RenderGraph | Logical resources and versions, declared accesses, validation, dependency analysis, scheduling, and transient lifetime. |
| Backend | Private lowering to DX12, Vulkan, Metal, WebGPU, or GL-family objects and synchronization. |

The review rejected a public API in which the renderer reaches backend-native
objects or the RHI depends on renderer policy. It also rejected making a
platform adapter a second scene or resource authority.

## Backend-specific observations

- DX12 and Vulkan have explicit barriers and native synchronization, but these
  are lowering details rather than portable caller objects.
- Metal encoder boundaries and drawable timing impose different submission
  ordering from Vulkan queue presentation.
- WebGPU uses asynchronous device acquisition, a single queue model, and
  browser-controlled presentation behavior.
- Desktop GL, GLES, and WebGL use context-bound state machines. Their profiles
  and extension evidence must remain distinct; a single `GL` feature flag is
  insufficient.
- WebGL default framebuffer behavior confirms that a presentable target is not
  always an importable or sampleable texture.

## Verification findings

Compilation, mocks, and test RHI implementations establish portable logic;
they do not establish hardware correctness. A support claim needs retained
evidence from its named target, including the exact revisions, target and
driver facts, command or host procedure, expected and actual output,
diagnostics, and focused test results.

Visual work requires inspected screenshots or recordings. Lifecycle work
requires evidence for the declared resize, visibility, loss, restoration, and
completion cases. Performance claims require a representative workload,
measurement method, raw results, and a declared threshold.

The historical Windows, WebGL2, and WebGPU stage documents retain the concrete
evidence that was actually collected. They do not imply generic browser,
mini-game, or other platform support.

## Deferred-boundary rule

Before a deferred capability enters a public API, it needs a real consumer,
explicit fallback or refusal behavior, a resource and completion model, and
semantic evidence from the relevant target set. A name in a deferred list does
not reserve an empty trait, handle, or configuration field.

## Final historical recommendation

Keep portable contracts small, explicit, and evidence-driven. Extend RHI only
from a real renderer or graph consumer. Preserve the separation between logical
assets, renderer residency, graph lifetimes, RHI execution, and private
backends. Reject unsupported operations explicitly instead of silently
approximating them.
