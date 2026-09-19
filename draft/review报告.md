# Fluxel RHI API platform-first review 报告

> **历史审查草稿，已被 RHI API v1 取代。** 本文仅保留审查过程，
> 不是实现依据。RHI 的唯一规范入口是
> `fluxel-rendering/documents/design-rhi.md` 及其 `documents/rhi-design/`
> 规范模块；生态顺序以当前 `ROADMAP.md` 为准。

> 日期：2026-09-18
> 状态：评审草案；不是已冻结协议、版本承诺或实现完成声明
> 范围：`fluxel-rhi` 公共 API、RenderGraph/RHI/backend 边界、0.16 规划前提
> 本轮不包含：backend 实现、编译修复、测试修复、性能结论
>
> 二次审查输入：`C:\Users\41200\Downloads\fluxel-rhi-platform-first-review.md`。本文已吸收其中经项目边界和五平台反例复核后成立的判断；未原样接受其 P0/P1/P2、Base 对象表或高级能力占位 API。

## 1. 结论摘要

当前 `协议.md` 最有价值的部分是 Fluxel 自己的架构原则，而不是它列出的对象全集：

- RenderGraph 决定工作顺序、资源 hazard、queue assignment、transition 与跨提交依赖；
- RHI 提供可移植的资源、录制、提交、完成和 presentation 执行语义；
- backend 负责 native object、barrier、descriptor、command allocator、fence/semaphore/event 与 API 调用；
- capability 是 adapter/device/format/surface/operation 的运行时事实，不由 backend 枚举或 Rust trait 是否实现来代替；
- native handle 和 native synchronization primitive 不进入公共对象模型。

这些原则应保留。

但当前协议和现行 0.16 plan **还不能作为 API freeze 依据**。它们虽然没有直接复制 `wgpu`，对象清单和部分执行模型仍明显以 `wgpu-hal`、现有 GL 适配层和已实现的 Vulkan 增量为起点，而不是从 DX12、Vulkan、Metal、WebGPU、GL 五个平台逐项反推。

本次独立评审的主要判断是：

1. `Device + resource + recording + ordered submission + completion` 是 portable execution core。
2. `Instance/Rhi -> enumerate Adapter -> open Device` 不是五平台都天然拥有的启动模型。它属于 platform-provider 层；必须允许异步请求、宿主注入的 device/context，以及无法枚举 adapter 的平台。
3. Queue topology 可以保留，但必须定义为 logical execution topology，而不是 native queue-family 镜像。最低承诺是一个串行提交域；`Queue` 或 `SubmissionLane` 只是命名选择，多提交域、GPU-side dependency 和并行录制才是 capability facts。
4. `SurfaceImage -> Texture` 不是五平台公共事实。GL/WebGL2 的 default framebuffer 不是可导入、可 view、可采样的 texture。公共语义应是 `PresentationTarget -> AcquiredFrame -> FrameAttachment`，显式 drawable texture 只能是可选投影；present 应作为 submission/presentation plan 的操作，而不是默认固定成 `Queue::present` 或事后调用的 `Surface::present`。
5. `BindGroup`、`BindGroupLayout`、`PipelineLayout` 可以作为参考词汇，但不应直接冻结为 Fluxel 架构名。共同语义更接近 shader resource interface、binding layout、resource table 与 pipeline interface。
6. `RenderPass` 不应暗示 native render-pass object。五平台共同语义是 raster encoding scope、attachment set、load/store/clear、render area 和 command legality。
7. transition 是 RenderGraph 计划的一部分，也是 RHI 内部执行输入，但不应成为普通调用者可手写的 public barrier API。内部 use model 至少要表达 pipeline scope、access、texture layout intent 与 subresource range；same-state write ordering 不能伪装成 state change。
8. RenderGraph 若将来决定 transient placement/aliasing，RHI 必须提供 allocation requirements、transient allocation service 与独立 alias boundary 的内部执行 seam；native heap、memory type 与 placement handle 仍然 private。这是按功能触发的 P1 gate，不是 0.16 public API 扩张。
9. 当前 `Provides<F> + require::<Family>` 设计不能直接公开。它把“backend 有某套词汇”和“具体 device 支持某能力”混入 Rust 类型约束，并把 storage role、draw parameter 等错误切成 capability traits。
10. 当前 public facade 是一个 DX12/Vulkan 固定垂直切片，不是待补全的通用 RHI facade。应重新声明 API，而不是逐个给现有 facade 增加方法。
11. 0.16 可以冻结一个由三 native backend 实证的 **provisional execution kernel**，但在 WebGPU 与 GL/WebGL2 完成反证式验收前，不应宣称五平台公共 API 已稳定。

因此，建议将 `协议.md` 视为输入材料而非规范成品；本文保留其有效判断，并记录需要撤回、改写和延期的部分。待新的 plan 和 API 声明完成后，根目录 `协议.md` 可以删除。

### 1.1 对二次审查稿的裁决

| 二次审查稿观点 | 本报告裁决 |
| --- | --- |
| 增加 transient allocation/alias execution semantic | 采纳其边界判断；降为功能触发的 P1 internal seam，不冻结 public heap/placed-resource API |
| transition 需要 pipeline/access/layout/subresource | 采纳为 P0 internal resource-use minimum；不公开手写 barrier |
| 增加 alias transition 与 global/UAV dependency | 拆分处理：`MemoryDependency` 进入 P0 internal vocabulary；独立 `AliasBoundary` 在 physical aliasing 实现时触发 P1 |
| split begin/end 进入 portable transition | 不采纳；graph 表达 dependency/overlap opportunity，backend 决定是否使用 split barrier |
| `BufferView` 升为 Base | 不采纳；Base 使用 buffer range，formatted/texel view consumer-driven |
| `BindGroup` 可以保留 | 接受其可实现性，但不把名称和 grouped model 冻结为全部 binding 架构；优先中性 interface/table 语义 |
| `QueueTopology` 保留 | 采纳，但定义为 logical submission topology；不映射 native queue 数量或 family index |
| present 从 Queue 移到 Surface | 采纳“不能固定 Queue”的反例，不采纳事后 `Surface::present(completed)`；present request 与 submission plan 一起形成 |
| 增加 ShaderCapabilities | 采纳分域 schema；descriptor indexing、device address、mesh/RT 仍归各自扩展，禁止 flat bool bag |
| device address、GPU-generated commands | 只登记为 research-only P2；不预设 `GpuAddress`、CommandSignature 或统一 handle |
| sparse、residency、tile-local、external interop、VRS、多 GPU | 用作 backlog taxonomy；不预建空 API，不扩大 0.16 |

这张表是本次“再做一次 review”的核心增量：识别 future seam，但不把三家 modern native API 的相似机制过早宣称为五平台共同对象。

## 2. 审查方法

本次采用以下顺序，而不是从 `wgpu-hal` 的类型表出发：

```text
DX12 / Vulkan / Metal / WebGPU / GL family
                    ↓
逐域检查真实对象、时序、能力和拒绝方式
                    ↓
提取共同 semantic，而非共同 native object
                    ↓
归类：
  portable execution core
  capability fact / limit
  optional command vocabulary
  RenderGraph-owned policy/plan
  platform-provider integration
  backend-private mechanism
  deferred / no portable contract
                    ↓
最后使用 wgpu-hal / rafx / UE RHI 查漏和反证
```

审查材料包括：

- 根目录 `协议.md`；
- `fluxel-rendering/crates/rhi/src/lib.rs` 与其 public re-export；
- `crates/rhi/src/common/api/`、`resource/`、`execution/`、`presentation.rs`；
- `documents/design-rhi.md` 与其 `documents/rhi-design/` 规范模块；
- 生态 `ROADMAP.md`、`ECOSYSTEM_ARCHITECTURE.md` 和现行 Stage 3F plan；
- DX12、Vulkan、Metal、WebGPU、GL/WebGL2 的对象与执行模型；
- UE RHI 技术文档《剖析虚幻渲染体系（10）- RHI》作为成熟引擎分层对照。

`wgpu-hal`、`rafx` 和 UE 均是参考样本，不是 Fluxel API 的语义权威。

## 3. 五平台独立能力矩阵

### 3.1 启动与设备选择

| 平台 | 原生事实 | 对 Fluxel 的推导 |
| --- | --- | --- |
| DX12 | factory 可枚举 adapter，再创建 device；surface/swapchain 需要窗口系统对象 | native provider 可以枚举，但 adapter index 只是一次进程内快照位置，不能成为稳定身份 |
| Vulkan | instance 枚举 physical device；logical device 创建时选择 queue families 与 enabled features | adapter facts 与 enabled device facts必须分开；queue request 是 open 的输入，不是 open 后才发现的全局常量 |
| Metal | device 发现存在，但对象模型比 Vulkan 轻；command queue 由 device 创建 | 不应为了 Vulkan 对称性暴露 queue-family 概念 |
| WebGPU | adapter/device 通常由宿主异步请求；只公开一个 device queue；能力启用发生在 requestDevice | portable bootstrap 必须容纳异步，并区分 adapter 可用能力与 device 已启用能力 |
| Desktop GL / OpenGL ES / WebGL2 | context 往往由 host/window/browser 创建或注入；不能可靠枚举“adapter”；执行与 context identity 绑定 | 必须允许 host-injected context/device provider；不能要求每个 backend 伪造可枚举 adapter 列表；三类 profile 的能力不可合并成一个 `GL=true` |

结论：

- `ExecutionDevice` 是执行核心对象；
- `Rhi/Instance/Adapter` 属 platform-provider 与 capability discovery 层；
- provider API 可以公开，但不属于“每个 backend 都按同一路径创建对象”的 execution base；
- portable 打开流程必须允许异步完成；native provider 可以返回立即就绪的结果；
- `Backend::{Dx12,Vulkan,...}` 可以作为 request preference 或诊断 identity，不能代替 adapter/device 协商；
- GL-family facts 至少记录 API family、version、profile 与 extension evidence。Desktop GL 4.6 的 compute/indirect/timestamp 结论不能投射到 GLES 或 WebGL2。

### 3.2 资源与内存

| 平台 | 原生事实 | 对 Fluxel 的推导 |
| --- | --- | --- |
| DX12 | resource + heap；committed/placed/reserved；residency 与 tiled resource 独立 | heap、placement、residency、sparse/tiled 不是 Base |
| Vulkan | buffer/image 与 memory allocation 分离；memory type、heap、sparse binding | 不公开 memory type index；allocation 是 backend 策略，除非未来有经过证明的显式内存扩展 |
| Metal | storage mode、heap、memoryless resource 对 attachment 很重要 | 简单 `DeviceLocal/CpuToGpu/GpuToCpu` 不足以表达 transient/memoryless intent |
| WebGPU | 资源受管；usage 与 map 能力在创建时约束；无公开 heap/placement | public descriptor 应表达 usage 和 host-access intent，不表达 heap |
| GL family（profile-dependent） | name object + driver 管理；map/immutable storage 能力随 Desktop GL/GLES/WebGL2 profile 与 extension 变化 | 资源 identity 绑定 context generation；host access 必须 capability-gated |

五平台共同语义：

- `Buffer`、`Texture`、逻辑 `TextureView`、`Sampler`；
- descriptor、usage、format、extent、subresource range、sample count；
- device/context identity + generation；
- host access intent 与 resource placement intent；
- completion-safe lifetime。

`MemoryPreference::{DeviceLocal,CpuToGpu,GpuToCpu}` 可以暂时保留为 ordinary automatic-allocation easy path，但不能成为完整内存协议。稳定语义应拆成正交意图，例如：

```text
ResourcePlacementIntent
  Persistent
  TransientAttachment

HostAccess
  None
  Write
  Read
```

上传与回读由 placement intent、host access、usage 和 workflow 共同表达，而不是把所有差异压成一个枚举。这仍不是 native heap 选择；backend 可在 UMA、离散 GPU、Metal memoryless、WebGPU 受管资源和 GL driver storage 上作不同 lowering。

当 RenderGraph 真正开始做 transient physical allocation 或 aliasing 时，还需要一条 **P1、crate-private** 的执行 seam：

```text
TransientAllocationRequirements
  size
  alignment
  backend/device-generation-scoped opaque compatibility key
  dedicated requirement/preference

TransientAllocator
  realize logical allocation plan
  create transient resource from allocation

AliasBoundary
  previous resource
  next resource
  physical allocation identity
```

RenderGraph 决定 lifetime overlap、reuse/alias plan；RHI internal allocator 执行 requirements 与 alias boundary；backend 决定 DX12 heap/aliasing barrier、Vulkan memory binding/dependency、Metal heap/aliasability，或 WebGPU/GL 的 pool/rename/no-alias fallback。

不要在当前 API 中冻结 `TransientHeap`、native-like offset placement 或 `create_placed_texture`。opaque compatibility key 也只供同 device generation 的 allocator bucketing，不能被解释为“相同 key 的任意资源都可安全重叠”。

### 3.3 Binding 与 shader interface

| 平台 | 原生机制 |
| --- | --- |
| DX12 | root signature、root parameter、descriptor heap/table、root descriptor |
| Vulkan | descriptor set layout、pipeline layout、descriptor pool/set |
| Metal | encoder set-by-index 或 argument buffer |
| WebGPU | bind group layout、pipeline layout、bind group |
| GL family（profile-dependent） | uniform/storage block、location、texture/sampler unit、program reflection；storage 能力并非 WebGL2 共有 |

它们的共同语义不是“所有平台都有一个 native BindGroup”，而是：

- shader 可见的 typed resource slots；
- slot 的 stage visibility、array/count、read/write role 与 dynamic range requirement；
- pipeline 的 shader-resource interface；
- 一次把实际 buffer range、texture view、sampler 等值绑定到该 interface；
- layout/interface compatibility。

建议候选词汇：

```text
ShaderInterface
BindingLayout
ResourceTable
PipelineInterface
```

是否需要“group/space”层级，应由 shader artifact、平台 limit 和至少两个真实 pipeline consumer 证明；不能仅因 WebGPU/Vulkan 使用 set/group 就先冻结。`BindGroup` 可在迁移期作为别名，但不应继续充当架构定义。

普通 buffer binding 的共同值应是 `BufferBinding { buffer, offset, size, ... }` 或等价 range，而不是预先增加一个万能 `BufferView` 对象。structured/raw 属 shader-interface metadata；真正 formatted/texel buffer view 留给有真实 consumer 的 future extension。

### 3.4 Pipeline、pass 与 command recording

| 平台 | 原生事实 |
| --- | --- |
| DX12 | PSO；render target 由 OM 状态设置；command list 显式录制 |
| Vulkan | pipeline；传统 render pass 或 dynamic rendering；command buffer 显式录制 |
| Metal | render/compute pipeline state；command buffer 内建立 encoder scope |
| WebGPU | render/compute pipeline；command encoder 与 pass encoder |
| GL family（profile-dependent） | program、FBO、全局可变状态和 context command stream |

共同语义是：

- shader/pipeline artifact；
- render-target signature（format、sample count、view mask 等）；
- raster/compute/copy command vocabulary；
- raster encoding scope 及其 attachments、load/store/clear、render area；
- command legality 与 deterministic order；
- 一段可提交的 recorded work。

因此：

- 公共名应倾向 `RasterScope` / `RenderingScope`，而不是暗示 `VkRenderPass` 同构对象；
- `Recorder`/`RecordedWork` 是 portable semantic，不要求 backend 暴露 native command buffer；
- GL backend 可以记录 Fluxel command stream 后串行执行，也可以在受控模式中直接 lower；这只是实现策略；
- pipeline creation 依赖 `RenderTargetSignature`，不依赖可缓存的 public native render-pass object；
- compute 是 optional vocabulary，因为 WebGL2 没有 compute；
- copy vocabulary 存在不等于每种 format/region/route 都受支持，支持性由 operation 与 format facts 表达。

Capability schema 还应有可扩展的 `ShaderCapabilities` 域，和 `ShaderArtifact.requirements`/pipeline validation 闭环。它至少能分域表达 stage、scalar type、subgroup 与 atomic 等已启用事实；descriptor indexing、device address、mesh/ray query 仍分别归 binding 或高级扩展，不能变成一个无限增长的 shader bool bag。

tile-local/input-attachment rendering 也只登记为 future capability domain。普通 pass fusion 是 RenderGraph/backend optimization；只有 shader 明确读取 tile-local/input attachment 时才需要新 portable semantic。

### 3.5 Queue、同步与完成

| 平台 | 原生事实 |
| --- | --- |
| DX12 | 多 command queue、fence、显式 resource state |
| Vulkan | queue family/queue、semaphore/fence/timeline、显式 layout/ownership |
| Metal | command queue/buffer/encoder；event 能力与平台版本有关；大量 hazard 由 API 模型处理 |
| WebGPU | 单个公开 queue；没有 public barrier/semaphore/fence；work-done 是异步完成边界 |
| GL family（profile-dependent） | context 串行命令流；sync 能力有限；默认不承诺 multi-lane |

最低公共执行合同应是：

```text
one serial submission lane
recorded work submission
ordered completion token
poll / asynchronous-or-blocking wait where platform permits
completion-safe retirement
```

建议使用 `SubmissionLane` 作为公共语义名，避免声称它与 native queue 一一对应。若最终保留 `Queue` 名称，也必须在文档中定义为“有序提交域”，而不是 native queue handle。Queue topology 继续存在，但表示 logical execution topology。

以下是 capability facts，而不是 Base 假设：

- 多 lane 与 lane class；
- concurrent execution；
- parallel recording；
- GPU-side cross-lane dependency；
- timeline dependency；
- ownership transfer requirement；
- timestamp placement。

RenderGraph 可以根据这些 facts 选择 serial plan 或 multi-lane plan。RHI 执行已编译的 dependency edge；backend 决定用 semaphore/fence/event、encoder boundary、ordered queue 或串行化实现。

`SubmissionTopology` 不能只从 native queue 数量推导：Metal 多 command queue 不承诺 Vulkan 式 queue classes 或 overlap；Desktop GL shared contexts 也不等于标准 multi-lane；WebGPU 的单 `GPUQueue` 可以执行多类 encoded work。facts 应分别描述 lane operation classes、concurrent-execution evidence、GPU dependency mechanism 与 ownership-transfer requirement。

还应区分两个相关语义：

- `SubmissionPoint`：同 device generation 的 GPU ordering/timeline point，可供 compiled dependency 引用；
- `CompletionToken`：CPU poll/wait 与资源 retention/quarantine 的观察能力。

是否最终拆成两个 Rust 类型可以后定，但不能把任一者描述成 public native fence/semaphore。

### 3.6 Presentation

| 平台 | 原生事实 |
| --- | --- |
| DX12 | swapchain back buffer 是 resource |
| Vulkan | acquire swapchain image，并携带 native acquire synchronization |
| Metal | drawable 及其 texture，时序由 layer/drawable 约束 |
| WebGPU | canvas context configure，获取 current texture |
| GL family（profile-dependent） | window/canvas default framebuffer，不是普通 texture |

协议中的 `SurfaceImage::texture() -> &Texture` 不能作为五平台 Base。正确的公共交集是“一帧 presentation destination 的独占 lease”，而不是“一张普通 texture”。

建议语义：

```text
PresentationTarget
  configure / unconfigure
  acquire_frame

AcquiredFrame
  frame_attachment
  optional drawable texture view
  opaque acquire gate

SubmissionPlan / SubmissionBatch
  recorded work
  dependency edges
  presentation request consuming AcquiredFrame
```

`FrameAttachment` 可被 RenderGraph 作为 external presentation endpoint 导入。只有 explicit-drawable backend 才能额外暴露 `TextureView`。如果 graph 需要对最终颜色做 sampling/copy/readback，而目标是 GL default framebuffer，则计划必须使用中间 texture，再由 backend/renderer 执行 final blit；不能伪造 default framebuffer texture。

acquire dependency 应是 executor/backend 消费的一次性 opaque gate，允许为空或已满足；不能要求所有平台在公共 API 中暴露 semaphore-like 对象。

present ordering 应与提交计划一起形成，而不是固定成 `Queue::present`，也不能简单设计为“拿到一个已提交/已完成 token 后再调用 `Surface::present`”。Metal 需要在 command buffer commit 前安排 `presentDrawable`。更稳妥的共同语义是 submission/presentation batch 消费 frame lease，并标明最后 writer；backend 再 lower 为 Vulkan submit+queue present、DXGI Present、Metal pre-commit presentDrawable、WebGPU implicit presentation 或 GL swap。

### 3.7 高级领域归类

| 领域 | 本次归类 | 原因 |
| --- | --- | --- |
| format × usage × sample count | capability facts | 支持取决于格式、操作、sample count 和 device，不是一个全局 bool |
| compute | optional command vocabulary + facts | WebGL2 无 compute；GL profile 分裂 |
| storage buffer/texture | binding role + format/stage/access facts | 不应成为独立 recording handle trait |
| indirect draw/dispatch | optional operation forms | draw、dispatch、count-buffer form 要分别证明 |
| base vertex / first instance | draw parameter facts | 不应人为拆成 Rust API family；直接 draw form 与 indirect form的限制也不同 |
| multiview | scope/pipeline 参数 + limit | view mask 是参数，支持性是事实，不是 trait |
| inline constants | optional parameter-block vocabulary | 不以 Vulkan push constant 或 DX12 root constant 命名 |
| query | optional query vocabulary | `QuerySet` 不是 Base；各平台 native allocation 完全不同 |
| mapping | optional async host-access workflow | WebGPU readiness 真异步；同步 `map()` 不能成为通用合同 |
| transient placement/aliasing | P1 internal execution seam | graph 若决定 physical reuse，RHI 必须能查询 requirements、realize allocation 并执行独立 alias boundary；native heap private |
| buffer range / formatted buffer view | range 是 Base binding value；formatted/texel view deferred | DX12 CBV/SRV/UAV、Vulkan range/VkBufferView、Metal buffer metadata、WebGPU binding range不是统一对象 |
| shader capabilities | 分域 capability facts | 与 shader artifact requirements 闭环；不做所有高级功能的 flat bool bag |
| bindless/descriptor indexing | deferred extension | 涉及 shader indexing、lifetime、array limits 与非统一 lowering |
| tile-local rendering | deferred extension | pass fusion 是优化；shader-visible tile/input-attachment 才需要新 semantic |
| GPU-generated execution | research-only deferred domain | DX12 ExecuteIndirect、Vulkan DGC、Metal ICB 的 signature、encoding 与 binding mutation 尚无可靠交集 |
| HDR/presentation timing | surface/display facts + later extension | 与 OS/display chain 强相关 |
| sparse/tiled resource | deferred | 语义、residency 与 page binding 尚未证明跨平台交集 |
| device address | research-only deferred or backend interop | WebGPU/GL 无一般等价物；还与 shader pointer、validation、security、capture relocation 强耦合 |
| VRS | deferred | tier 与 attachment/shading-rate representation 差异大 |
| mesh/task shader | deferred | stage、payload 与 pipeline limits 尚未有 consumer |
| ray tracing | deferred | AS、shader table、pipeline/query 模型差异显著 |
| external memory/sync | deferred interop | 必须按平台 handle 类型、安全与所有权单独设计 |
| multi-GPU/device group | explicitly unsupported for now | 没有 roadmap consumer 与跨平台合同 |
| capture/replay | design constraint, not 0.16 API | 要求 operation 可描述、descriptor 稳定、native handle 不泄漏，但不要求本期实现 |

Basic indirect 必须继续按 operation form 分别记录：draw、indexed draw、dispatch、count-buffer 与 multi-draw 不是一个 capability。尤其 Desktop GL、GLES、WebGL2 的支持面完全不同，WebGPU 也只有受限的 basic indirect；不能用一行 `GL/WebGPU supports indirect` 概括。

GPU-generated execution 也不能取代 basic indirect。DX12 ExecuteIndirect、Vulkan device-generated commands 和 Metal ICB 只有“GPU/预编码命令执行”这一高层相似性，command signature、可变 binding/constants、encoder restriction、CPU/GPU encoding 与 reuse 模型尚未形成可靠 portable intersection。

若未来研究 device address，capture/replay 不能序列化裸地址。最低要求是 logical `resource + offset` relocation，并同时解决 device generation、shader requirement、validation 与安全边界；在这些条件出现前不声明 `buffer_device_address() -> integer`。

## 4. 对 `协议.md` 的逐类评审

### 4.1 应保留的结论

以下内容是 Fluxel 自己的架构，且经本次 platform-first 审查仍成立：

- “统一语义，不统一能力”；
- RenderGraph / portable execution / backend lowering 三层职责；
- capability facts 与 limits 是运行时数据；
- optional family 只能表示词汇组织，不能代表具体 device 支持；
- native barrier、fence、semaphore、descriptor heap/pool、swapchain 等保持 private；
- Buffer、Texture、TextureView、Sampler 的 portable descriptor 与 resource identity；
- format capability 必须按 format、usage、access、sample count 等维度查询；
- resource Rust handle drop 不等于 native resource 立即释放；
- RenderGraph 管 hazard、transition planning、queue assignment、aliasing、fallback；
- RHI 必须有 submission/completion 与 accepted-unknown quarantine；
- compressed format 是 format vocabulary，不是独立 API；
- async compute、multiview、anisotropy等不应因为名称听起来像功能就各自建立 trait；
- capture/replay 要记录 portable execution facts 而非 native commands，这可以约束今天的 descriptor 和 command 可描述性。

### 4.2 必须改写的结论

| 原协议方向 | 评审问题 | 改写方向 |
| --- | --- | --- |
| `RhiApi::enumerate_adapters` 为所有 backend Base | WebGPU 是请求式/异步；GL context 常由 host 注入且不可枚举 | 拆成 platform-provider；支持 request、enumeration（若有）和 adopted context/device |
| `DeviceDescriptor.queue_request` 直接建模 queue topology | 容易把 Vulkan family 模型扩散到 Metal/WebGPU/GL | 请求 submission requirements；公开 logical lane facts，native family index private |
| `SurfaceImage` 总能返回 `Texture` | GL/WebGL2 default framebuffer 反例 | `AcquiredFrame -> FrameAttachment`；drawable view optional |
| `Queue::present(surface, image)` 是唯一形状 | 只有 Vulkan 天然是 queue-present；Metal 要在 commit 前安排 drawable，WebGPU 隐式，GL swap 属 surface/context | presentation request 进入 submission plan 并消费 frame lease；backend 决定实际调用时序 |
| `SubmissionDependency { token }` 可由普通调用者构造 | 可能跨 device/lane 误用，也不适合隐式单 queue 平台 | dependency edge 属 compiled plan/internal executor；opaque、same-device、single-use validation |
| `SemanticTransitionApi` 是可调用 API | 容易退化为 public barrier；单一 before/after state 也无法表达 layout 不变的 write ordering | 作为 RHI 内部 resource-use/dependency input；至少包含 pipeline scope、access、texture layout intent、subresource range，普通 API 不手写 transition |
| `map() -> MappedRange` | 浏览器映射真实异步，且 map legality 取决于创建与 in-flight state | `request_map`/future/readiness + scoped view，0.16 优先专用 upload/readback |
| BindGroup/PipelineLayout 是固定架构名 | 偏 WebGPU/Vulkan 对象模型 | 使用 shader interface / binding layout / resource table / pipeline interface 中性语义 |
| RenderPassDescriptor | 容易暗示 native render-pass object | RasterScopeDescriptor；附件、load/store 与 render area 是 scope 参数 |
| QuerySet 是 Base | 五平台 allocation object 不同，且 query 并非所有 profile 所需 | Query vocabulary 整体 optional；allocation/reuse 由 backend/executor 管理 |
| P0 全部标 `[x]` | 没有五平台矩阵和反证；多项仍以“建议”描述 | 重置为 candidate decisions + evidence gate，不把文档自洽当作证明 |

### 4.3 应明确撤回的结论

- 不能把 `Provides<F>` 或 trait implementation 当 backend/device capability 的公共判定入口。
- 不建立 `StorageBufferApi`、`StorageTextureApi` 这类资源角色 trait。
- 不建立 `AsyncComputeApi`；async compute 是 topology + scheduler decision。
- 不建立 `CompressedTextureApi`；压缩格式是 format facts。
- 不把 `BaseVertex`、`FirstInstance` 建成独立 handle family；它们是 draw form 的参数与 capability facts。
- 不公开 native-like barrier、fence、semaphore、queue family、descriptor set/pool 或 swapchain。
- 不把 `Backend` enum + native adapter index 当作完整 portable bootstrap。
- 不把当前固定 raster/compute recipe API 提升为通用 RHI。
- 不把 `SplitBegin/SplitEnd` 冻结成 portable transition kind。graph 表达 producer、consumer、dependency 与可重叠区间，backend 自行选择 split barrier、event、encoder boundary 或普通 barrier。
- 不把通用 `BufferView` 提升为 Base；普通 binding 使用 buffer range，formatted/texel view 等真实 consumer 出现后再设计。

### 4.4 延后而不删除的领域

以下内容可以保留在设计 backlog，但不得成为 0.16 首发 freeze 条件：

- general mapping；
- general query/timestamp/pipeline statistics；
- indirect-count/multi-draw；
- bindless/descriptor indexing；
- multiview；
- external image/video interop；
- pipeline cache；
- HDR/presentation timing；
- mesh/task shader、VRS、ray tracing；
- tile-local shader semantic、GPU-generated execution；
- sparse/tiled、residency/budget、device address、多 GPU；
- capture/replay runtime 与文件格式。

### 4.5 优先级：P0 freeze、P1 implementation gate、P2 backlog

下载稿提出的 P0/P1/P2 不能直接解释为“0.16 全部要实现”。本报告使用以下定义：

#### P0：声明 provisional 0.16 contract 前必须签署

- capability 的唯一运行时真相是分域 facts/limits；trait 只组织词汇；
- platform provider 允许 async request、enumeration（若有）和 adopted context/device；
- Desktop GL、OpenGL ES、WebGL2 的 family/version/profile/extension evidence 分开；
- RHI 拥有 portable execution vocabulary，RenderGraph lower plan 到该 vocabulary；
- Base 只保证一个 serial logical submission lane、completion 与 quarantine；
- `AcquiredFrame -> FrameAttachment`，drawable `TextureView` optional；present ordering 进入 submission plan；
- internal resource-use input 至少表达 resource kind、pipeline scope、access、texture layout intent、subresource range 与 same-device/generation validation；
- layout 不变的 write ordering 可表达为 `MemoryDependency`，不能伪造一次 state change；
- 不冻结同步 `map() -> MappedRange`；0.16 只承诺专用 upload/readback，通用 mapping 保留 async/pollable readiness；
- descriptor、command、dependency 与 presentation operation 在语义上可稳定描述，为未来 trace/replay 留出可能。

P0 冻结的是语义和禁止项，不要求本轮完成所有 backend 实现。

#### P1：对应功能真正进入实现前触发

- transient physical allocation/aliasing 进入 release 时，设计 `TransientAllocationRequirements`、internal allocator service、独立 `AliasBoundary` 和 reuse lowering；
- multi-lane/cross-lane scheduling 进入 release 时，冻结 GPU ordering point、dependency、ownership transfer 与 completion 的精确关系；
- shader feature consumer 进入 release 时，扩展 `ShaderCapabilities` 分域 facts，并与 artifact requirements/translator acceptance 闭环；
- same-state storage/UAV ordering、global memory dependency 出现真实 workload 时，验证 `MemoryDependency` 的粒度与 scope；
- formatted/texel buffer consumer 至少在两个目标 profile 上证明语义后，再决定是否需要独立 view handle。

P1 不授权 public native heap、placed resource、split barrier、timeline semaphore 或万能 `BufferView`。

#### P2：登记 capability domain，不预建空 API

- sparse/tiled resource、residency、memory budget；
- bindless/descriptor indexing；
- buffer/device address及其 shader/capture relocation；
- GPU-generated execution；
- tile-local shader semantic；
- external memory与external sync；
- VRS、mesh/task shader、ray tracing；
- multi-GPU/device group；
- pipeline cache/library；
- capture/replay runtime 与 serialization format。

每个 P2 项都需要真实 consumer、至少两个平台的 semantic proof、明确 fallback/reject 和 consumer migration plan；登记名称不代表承诺未来一定有统一 public API。

UE/RDG 的 transient aliasing、split barrier、parallel command lists 只证明未来 internal execution seam 不能被封死，不构成 0.16 实现 RDG、transient allocator、parallel recording 或三 native 高阶功能的授权。

## 5. UE RHI 对照结论

参考资料：<https://www.cnblogs.com/timlly/p/15156626.html>，文章基于 UE4-era RHI，发布于 2021 年。它是高质量的体系分析，但不是当前 UE 源码或 Fluxel 的规范性来源。

### 5.1 值得吸收的思想

1. **Renderer resource 与 RHI resource 分层。**
   UE 区分 `FRenderResource` 与 `FRHIResource`。这印证 Fluxel 应继续让 renderer 持有场景、recipe、residency policy，让 RHI 只持有 GPU execution object；不能把 asset/scene policy 塞入 RHI。

2. **资源和命令是 RHI 的两条主轴。**
   UE 同时拥有 RHI resource hierarchy 和 `FRHICommand`/`FRHICommandList`。这支持 Fluxel 将 portable object descriptors 与 portable recorded command stream 同时视为 API 核心，而不是只有资源工厂或只有 backend trait。

3. **Command list 与 command context 分离。**
   UE 的 command list 表示中间命令，context 负责把命令 lower 到具体 RHI。Fluxel 可以吸收这个边界：`Recorder/RecordedWork` 是 portable stream，backend execution context 保持 private。公共 recorder 不等于 native command buffer。

4. **DynamicRHI 是 backend dispatch，不是 capability proof。**
   UE 用不同 DynamicRHI 实现 D3D/OpenGL/Vulkan/Metal。可借鉴的是 backend realization 的隔离，而不是用 backend 类型推断硬件支持。Fluxel 仍需独立 capability facts。

5. **Immediate、deferred、parallel 是执行策略。**
   UE 可以 bypass command list、使用 RHI thread、并行生成/翻译命令。这说明 public command semantics 不应把某一种线程与提交实现写死。Fluxel 的 serial/parallel recording、后台翻译、直接 lowering 都应是 capability 与实现策略。

6. **延迟释放是独立生命周期问题。**
   UE 的 `FRHIResource` 有 deferred deletion。它支持 Fluxel 保留“handle drop != native destruction”的原则。但 Fluxel 应优先以 submission point/completion token 做安全释放，而不是把“等待若干帧”作为 portable correctness 合同。

7. **Pass、transition、fence、viewport/present 都属于完整 RHI 检查面。**
   UE 的广泛命令表可用于查漏，提醒 Fluxel 不能只设计 draw/dispatch/copy 三个 happy path；但是否进入 Base 仍由五平台与实际 consumer 决定。

8. **Graph-owned transient/alias plan 仍需要 RHI execution seam。**
   二次审查稿引用的 UE5 RDG/transient-resource/transition资料进一步说明：graph 决定 lifetime、aliasing 与 dependency，并不等于 RHI 可以完全没有 allocation requirements、alias boundary 或 memory dependency 的执行入口。Fluxel 应吸收这一层次，而不是复制 UE 的具体 heap allocator、`FRHITransition` 或 split-barrier API。

### 5.2 不应照搬的部分

- UE RHI 历史上从 D3D11 形状成长，不能作为五平台最小公分母；
- 巨大的 C++ virtual interface 和资源继承树不适合 Rust API；
- 一个 command 一个 class 的类型层级不适合 Fluxel 的稳定 command vocabulary；
- global DynamicRHI、global command list、显式 RHI thread 是 UE runtime policy，不是库级 portable semantic；
- UE 的 Viewport/SwapBuffer、frame lifecycle 和 renderer thread 模型不能直接变成 Fluxel Host/RHI 边界；
- UE 暴露的某些 transition/fence 操作不意味着 Fluxel 普通调用者也应手写它们；Fluxel 已有 RenderGraph，应让 graph 拥有计划；
- UE/RDG 使用 split barrier、parallel command list 或 transient heap，不意味着这些机制是 0.16 portable vocabulary；它们首先是 future internal seam 的反证；
- UE 支持的全功能面不能变成 0.16 的范围授权。

### 5.3 UE 对本次结论的实际影响

UE 对照强化了三项决定：

- RHI 需要稳定的 recorded command semantic，而不仅是 resource API；
- portable command stream 与 backend execution context 必须分离；
- resource retirement 必须是 submission/completion 系统的一部分。
- graph-owned aliasing 若进入实现，RHI 必须提供 backend-neutral internal execution seam。

UE 没有改变以下决定：

- capability 仍是 runtime facts；
- RenderGraph 仍拥有 hazard、transition 与 scheduling plan；
- GL/WebGL2 default framebuffer 仍不能伪装为 `Texture`；
- Fluxel 不采用 UE 的 global singleton、线程模型、继承树或大而全 API。

## 6. 当前 `fluxel-rhi` API 审查

### 6.1 Public root 仍是固定垂直切片

`crates/rhi/src/lib.rs` 当前公开：

- `Backend::{Dx12,Vulkan}`；
- `Device::open(backend, DeviceOptions { adapter_index, validation })`；
- 固定的 `CopyBackend`、`ComputeBackend`、native completion 与 recipe-specific artifacts；
- Windows + DX12/Vulkan 条件下的 presentation facade；
- Buffer/Texture 上传、lease 和固定 bindings。

这套 API 对当前验证路径是诚实的，但它不是五平台通用 facade：

- backend 集合与编译目标被写入公共 enum；
- adapter 以 backend-native list index 选择；
- open 是同步 headless 路径；
- 没有 adapter/device 两阶段 capability 协商；
- 没有统一 recorder/recorded work/submission lane；
- Surface 与固定 raster recipe 耦合；
- root 直接导出当前 renderer slice 的 upload/artifact vocabulary。

结论：不要在这套 root API 上继续“补齐缺少的方法”。应把它标为 legacy vertical-slice facade，在新声明稳定后迁移消费者并移除。

### 6.2 `HardwareCapabilities` 不是可扩展 capability schema

当前 `HardwareCapabilities` 直接包含：

- `rgba8_unorm_*` 的 filter/storage facts；
- 固定 compute limits；
- 少量 bind group/alignment limits。

这会随着格式和功能增加不断扩字段，也无法表达：

- adapter 可用 vs device 已启用；
- format × usage × access × sample count；
- surface × adapter/config；
- submission topology；
- operation parameter conditions；
- supported / unsupported / unexamined 及 evidence provenance。

应拆为按领域查询的 adapter facts、device-enabled facts、limits、format facts、presentation facts 和 submission facts。当前私有 ledger 可作为实现证据记录，不应直接成为 public ABI。

### 6.3 `common/api` 不能直接 `pub use`

当前 `common/api` 的核心是：

```text
CapabilityFamily
Provides<F>
require::<D, F>()
family-specific borrowed handle
```

并把 Graphics、Compute、Copy、StorageBuffer、StorageTexture、Indirect 等切成 family traits。

问题不是“这些 trait 现在是 private”，而是它们不适合作为最终 public model：

- backend 是否实现 trait 是编译期事实，device 是否支持 capability 是运行时事实；
- 两套事实并存增加了不存在于实际硬件模型中的拒绝分支；
- storage buffer/texture 是 shader binding role，不是独立 command context；
- base vertex/first instance 是 draw parameters，不需要独立 negotiated handle；
- family handle 自己充当 recorder context，使多个 family 的组合、统一 error、trace identity 和一次 submission 变复杂；
- 每个 family 自带 associated Pipeline/Bindings/Error，不利于形成统一 portable command stream。

这套设计可以保留为历史实验记录，不能通过改 visibility 变成新 API。

### 6.4 RHI 与 RenderGraph 的依赖方向需要纠正

当前 `fluxel-rhi` 的 Cargo 依赖包含 `fluxel-rendergraph`，resource、execution、presentation API 直接使用 RenderGraph 的 descriptor、identity、pass 和 bound-surface 类型。

这使 planning 层的类型成为 RHI public/data model 的来源，与目标边界冲突：

```text
RenderGraph owns plan
        ↓
RHI consumes portable execution input
        ↓
backend lowers
```

评审建议：

- RHI 拥有 portable execution vocabulary、resource descriptor、command descriptor 与 completion types；
- RenderGraph 把 plan lower 成这些类型；
- `fluxel-rhi` 不导入 RenderGraph plan/binding 类型；
- 若两者确实需要无 policy 的共享值类型，可在实施计划中比较“RHI owns types”与“小型 protocol-types boundary”，但不得把 GPU 类型放进 `fluxel-bases`；
- 不为解决当前编译方便立即创建新 crate，先冻结 ownership 和依赖方向。

### 6.5 Presentation 当前反向绑定 RenderGraph

当前 Windows presentation 路径可以把 acquired frame 直接变成 RenderGraph `BoundSurfaceTexture`。这让 RHI presentation object 知道上层 graph import 形状。

目标应反转：

- RHI 返回 `AcquiredFrame` 与 `FrameAttachment`；
- RenderGraph adapter/import 层把它加入 graph；
- frame 中的 acquire gate 只被 executor/backend 消费；
- present 消费 frame lease，并与最后一个写入 submission 建立关系；
- Surface/PresentationTarget 不创建窗口、不处理 event loop、resize policy 或应用 lifecycle。

### 6.6 Browser adapter 证明了不能使用一个同步启动模型

`rendering-wasm` 当前直接使用 RHI 的 WebGL2/WebGPU session-style adapter。现有生态规则已经要求 browser native objects 与 session/token 不进入公共架构。

新 API 应吸收 browser path 证明的事实：

- bootstrap 与 device loss/recovery 可能异步；
- canvas/context 由 host/jsbridge 提供；
- device/context generation 是资源 identity 的组成部分；
- canvas epoch 与 host lifecycle 仍是 adapter-private；
- RHI 公共 API 只看到 provider、device generation、presentation target 与结构化 loss/recovery outcome。

不能把现有 `WebGpuSession`/`WebGl2Session` 改名后升为公共对象。

### 6.7 已知直接消费者

API 迁移至少影响：

- `fluxel-renderer`：Device、uploads、fixed artifacts、presentation、completion、readback；
- `fluxel-rendergraph`：resource descriptor、execution SPI、transition/submission plan；
- `rendering-wasm`：WebGL2/WebGPU bootstrap、resource registry、frame execution；
- `fluxel-host`：只提供 raw window/display handle 与 lifecycle，不接管 GPU semantic；
- `fluxel-jsbridge`：browser/minigame lifecycle adapter，不成为 RHI semantic authority。

因此，API freeze 不能只做 `fluxel-rhi` 自身 compile check；必须有精确 consumer migration plan。

## 7. 建议的新 API 分层

以下是评审级 semantic skeleton，不是已批准的 Rust 签名。

### 7.1 Platform provider

职责：

- backend/provider availability；
- adapter request 或 enumeration（平台支持时）；
- host-injected context/device adoption；
- presentation target creation/adoption；
- 异步 device request。

候选对象：

```text
PlatformProvider
Adapter
AdapterInfo
AdapterCapabilities
AdapterLimits
DeviceRequest
OpenDevice
```

规则：

- request 可以异步；
- native enumeration 是可选 provider 能力，不是所有 provider 的 Base；
- adapter identity 不是 list index；
- validation policy 是 provider/device request option，不是 portable hardware capability；
- surface compatibility 是 adapter × presentation target 的关系。

### 7.2 Portable execution core

```text
ExecutionDevice
DeviceIdentity + generation

Buffer
Texture
TextureView
Sampler

ShaderArtifact
PipelineInterface
RasterPipeline
ComputePipeline

BindingLayout
ResourceTable

Recorder
RecordedWork
SubmissionLane
SubmissionBatch
SubmissionPoint
CompletionToken
CompletionStatus
```

最低 profile 只承诺一个 serial submission lane。`SubmissionPoint` 与 `CompletionToken` 是语义上的候选拆分，不要求立即成为两个 public Rust 类型。Graphics/compute/copy 的 operation support 仍由 device facts 与 plan requirements 决定；“对象存在于协议”不等于每个 device 都支持每种 command。

### 7.3 Recording scopes

候选形状：

```text
Recorder
  begin_raster(RasterScopeDescriptor) -> RasterScope
  begin_compute(ComputeScopeDescriptor) -> ComputeScope
  copy_*(...)
  finish() -> RecordedWork

RasterScope
  set_pipeline
  set_resource_table
  set_vertex_buffer
  set_index_buffer
  set_viewport
  set_scissor
  draw(DrawArgs)
  draw_indexed(DrawIndexedArgs)

ComputeScope
  set_pipeline
  set_resource_table
  dispatch(DispatchArgs)
```

Rust borrow/state API 可以在实现阶段防止 scope 外非法命令，但协议先冻结 command legality，不预先冻结每个 typestate helper。

`DrawArgs` 应能表达正常 direct draw 的完整参数。是否允许某个非零字段，由 operation facts/requirements 验证，不通过新增 trait 把字段从 command vocabulary 中删除。

### 7.4 Submission 与 internal execution plan

Public submission：

```text
SubmissionLane::submit(SubmissionBatch) -> CompletionToken
CompletionToken::point / status / wait_async / wait_blocking_if_supported
```

Internal RenderGraph executor contract：

```text
BufferUse
  pipeline scope
  access mask
  byte range when relevant

TextureUse
  pipeline scope
  access mask
  texture layout intent
  mip/layer/aspect range

semantic ordering edges
MemoryDependency
submission dependency edges
presentation acquire gate
presentation request consuming AcquiredFrame
last-use/retirement facts
```

规则：

- 普通用户不创建 native-like transition；
- plan dependency 必须验证 same-device/generation；
- `TextureUse` 的 layout intent 应从 resource role/access 推导并验证，避免调用者制造 access/layout 矛盾；`Undefined` 只表示首次使用或 discard，不是普通运行态；
- backend 可以把 semantic use/dependency lower 为 explicit barrier、encoder boundary、ordered execution、GL memory barrier 或 no-op；
- split barrier 是 backend optimization，不进入 P0 portable plan；
- physical aliasing 进入实现时使用独立 `AliasBoundary`，不伪装成 before/after resource state；
- submission point/completion token 是 ordering、completion、retirement identity，不是 public semaphore；
- accepted-unknown submission 必须 quarantine 引用的资源。

### 7.5 Presentation family

```text
PresentationTarget
PresentationCapabilities
PresentationConfig
AcquiredFrame
FrameAttachment
PresentationError
```

必须支持：

- configure/unconfigure；
- zero-size/minimized/not-ready/outdated/lost 的结构化结果；
- acquire frame lease；
- optional drawable texture view；
- queue/lane × target compatibility；
- submission/presentation plan 消费 frame、标明 last writer，并允许 backend 在正确的 commit/submit/swap 时点 present；
- unpresented frame 的 backend-specific quarantine/recovery。

不承诺：

- public Swapchain；
- acquired frame 总是 Texture；
- 所有平台有同一种 timeout；
- native acquire semaphore 可见；
- Surface 处理窗口事件；
- `Queue::present` 或 `Surface::present(completed_token)` 是唯一 portable shape。

### 7.6 Capability queries

capability schema 至少分域：

```text
Adapter facts
Device enabled facts
Device limits
Submission topology
Format capabilities
Binding/pipeline limits
Shader capabilities
Presentation capabilities
Query/timestamp facts
Host-access/mapping facts
```

支持状态需要区分：

```text
Unexamined
Unsupported(reason)
Supported(facts + evidence/provenance)
```

是否把三态直接公开为一个 enum，可以在 Rust API review 时决定；语义上不能把“没有读到”“明确不支持”“已支持但 device open 未启用”压成同一个 `false`。

## 8. 对 0.16 plan 的建议

### 8.1 当前 Stage 3F 的状态判断

现行 Stage 3F 包含大量有价值的 Vulkan 实现证据与每个 increment 的失败边界，但它不再适合作为“已确认的 0.16 execution contract”：

- W1 的 capability-family 设计正是本次评审要求撤回的模型；
- Appendix A 以 `wgpu-hal` GLES comparison 作为 triage 起点；
- portable limit floor 部分来自 `wgt::Limits::default()`；
- plan 已把多个局部 Vulkan increment 的完成写成 API 模型的完成；
- 生态 `ROADMAP.md` 尚未登记 Stage 3F/0.16 的权威位置；
- WebGPU/GL 的反证尚未发生。

应保留现行文档中的实现事实，但把它降级为 pre-freeze implementation history；不能删除已有 Vulkan evidence，也不能继续让这些日志授权新的 public API。

### 8.2 新的 0.16 顺序

建议按以下顺序重排：

1. **Platform audit gate**
   完成五平台矩阵；每个领域标明 core/fact/optional/RG/private/deferred，并记录反例。

2. **Terminology and ownership gate**
   决定 provider/execution/presentation/binding/recording 名称；纠正 RHI ↔ RenderGraph 依赖方向。

3. **API declaration gate**
   只声明新 facade、descriptor、errors、facts、scopes、submission/presentation 与 internal resource-use/dependency seam；允许暂时不编译，不同时实现 backend。

4. **Consumer migration plan**
   明确 renderer、rendergraph、rendering-wasm、host/jsbridge 的迁移次序；旧 vertical-slice API 只作为临时兼容层。

   若本 release 同时纳入 transient allocator、真实 aliasing 或 multi-lane，必须在这里触发 §4.5 的对应 P1 gate；否则这些能力保持未实现，不阻塞最小 kernel。

5. **Native proof**
   DX12、Vulkan、Metal 实现同一最小 execution kernel；真实 evidence 绑定 source revision。

6. **Web counter-proof**
   WebGPU 和 GL/WebGL2 专门验证异步 bootstrap、单 lane、无 public barrier、default framebuffer、无 compute 与 async mapping。发现公共语义错误时回开 API，不以 adapter shim 掩盖。

7. **Dependency closure**
   在 route 已切换且 consumer 通过后删除 `wgpu-hal`；“删除依赖”是收尾 gate，不是 API 正确性的证明。

### 8.3 0.16 可冻结范围

候选范围：

- provider/device request 的跨 sync/async semantic；
- device identity + generation；
- buffer/texture/view/sampler 与 format facts；
- placement/host-access intent 与专用 upload/readback；
- shader artifact、pipeline interface、resource table 的最小闭环；
- raster scope、基础 copy、可选 compute scope；
- one serial logical submission lane、submission point/completion、retirement；
- presentation target/acquired frame/frame attachment 与 submission-integrated present request；
- RenderGraph internal pipeline-scope/access/layout/subresource/ordering/dependency seam；
- structured unsupported/lost/not-ready/error taxonomy。

不在 0.16 freeze：

- general mapping；
- general query/timestamp；
- multi-lane scheduling 的 public authoring API；
- public heap/placed-resource API、transient allocator 实现与 physical aliasing（除非本 release 明确触发 P1）；
- formatted/texel BufferView；
- bindless、device address、GPU-generated execution、RT、mesh、VRS、sparse、external memory/sync；
- capture/replay runtime；
- 通用 shader compiler/cache；
- renderer material/scene/asset API。

### 8.4 稳定性措辞

0.16 的 API 应标为 **provisional cross-platform contract**：三 native backend 可以证明其可实现性，但 WebGPU 和 GL/WebGL2 才能证明它没有隐藏 explicit-API 假设。

只有以下条件同时成立，才能把它描述成“五平台稳定 RHI API”：

- 三 native backend 使用同一 public contract；
- WebGPU 没有靠额外 public session/token 模型绕开 contract；
- WebGL2 default framebuffer 不被伪装为 Texture；
- 无 compute、无 multi-queue、无 public barrier 的 profile 能结构化拒绝；
- browser async bootstrap/map/completion 没被同步 API 隐藏；
- pinned renderer/rendergraph/rendering-wasm consumers 全部迁移并验证。

## 9. Review gate

### 9.1 P0：provisional contract 必须全部签署

- [ ] provider 如何统一 native-ready 与 browser-async：Rust `Future`、poll token，还是分层 API？
- [ ] Desktop GL、GLES、WebGL2 是否以 family/version/profile/extensions 分开报告，而不是一个 `GL` bool？
- [ ] GL/WebGL2 adopted context 如何进入 provider，而不暴露 browser session/token？
- [ ] adapter identity 如何稳定表达，而不使用 list index 或 native handle？
- [ ] RHI 自有 portable types，还是提取最小 protocol-types boundary？依赖图是否无环且符合所有权？
- [ ] `SubmissionLane` 是否采用该名称；若保留 `Queue`，是否明确它是 logical ordered domain 而非 native queue？
- [ ] topology 是否分别表达 operation classes、并发证据、dependency mechanism 与 ownership transfer，而不是映射 native queue 数？
- [ ] `FrameAttachment` 如何被 RenderGraph import；哪些操作要求中间 texture？
- [ ] present request 如何在同一 submission plan 中消费 frame，并兼容 Metal pre-commit 时序？
- [ ] binding 是否需要 group/space；由哪些两条真实 pipeline 证明？
- [ ] ShaderArtifact 由哪个层生产、版本化和验证，RHI 是否只消费？
- [ ] internal resource use 是否完整表达 pipeline scope、access、layout intent 与 subresource range？
- [ ] public manual RHI command path 如何获得正确 resource-use planning，且不公开 barrier？
- [ ] layout 不变的 write ordering 如何进入 `MemoryDependency`，而不伪装 transition？
- [ ] completion API 在 native blocking wait 与 browser async-only 环境中的共同语义是什么？
- [ ] submission point 与 CPU completion/retention 是否需要两个 Rust 类型；无论是否拆分，same-device/generation 是否可验证？
- [ ] capability 的 unexamined/unsupported/supported/enabled 如何编码？
- [ ] lost device/context 后旧 generation 的 handle 如何统一拒绝？
- [ ] accepted-unknown submission 和 unpresented frame 分别如何 quarantine？
- [ ] 哪些 descriptor/command/dependency/presentation operation 必须保持可描述，以保留未来 trace/replay 可能性？
- [ ] 0.16 哪些名字和行为是 provisional，何时升级为 stable？

### 9.2 P1：对应功能进入 release 时签署

- [ ] transient allocation requirements 的 size/alignment/compatibility/dedicated 语义是否由至少两个 backend 实现验证？
- [ ] internal allocator service 是否允许 WebGPU/GL pool/no-alias lowering，而不是强迫 explicit heap？
- [ ] `AliasBoundary` 是否与普通 resource state/use 分离，并绑定 allocation/device generation？
- [ ] multi-lane 是否有真实 concurrent-execution evidence、GPU dependency 与 ownership-transfer 路径？
- [ ] ShaderCapabilities 新增字段是否由 artifact requirement 或 pipeline consumer驱动，而不是 flat feature inventory？
- [ ] formatted/texel buffer view 是否有两个目标 profile 的共同 shader/binding semantic？

未触发相应功能时，P1 未签署不阻塞最小 0.16 kernel。

### 9.3 P2：非阻塞登记规则

每个 deferred domain 在设计 public API 前必须补：真实 consumer、至少两个平台 semantic proof、明确 fallback/reject、lifetime/identity、安全与 capture 影响。不得通过空 trait、空 handle 或“以后可能需要”的字段提前占位。

## 10. 本轮最终建议

### 接受

- 接受 `协议.md` 的三层责任、capability-data、native-private、RenderGraph-owned planning 与 completion-safe lifetime 原则。
- 接受 UE 对照所证明的 resource/command 双主轴、command stream/context 分离和 deferred retirement 需求。
- 接受 Buffer/Texture/View/Sampler、pipeline、recording scope、submission/completion 作为 RHI 核心方向。
- 接受 logical queue topology、分域 ShaderCapabilities，以及 graph alias plan 对 internal transient-allocation/alias-boundary seam 的未来需求。

### 重做

- 重做 bootstrap/provider、GL profile facts、capability schema、binding 名称与对象边界、queue/submission、resource-use/dependency、presentation frame/present ordering、mapping async contract。
- 重做 `fluxel-rhi` root public facade；不公开当前 `common/api`。
- 重做 Stage 3F 的权威状态与执行顺序；保留其实现日志为历史证据。
- 重做 RHI/RenderGraph 类型 ownership 和 dependency direction。

### 延后

- 延后 general mapping/query、formatted buffer view、bindless、device address、GPU-generated execution、advanced pipeline families、external interop、capture runtime 与高级 GPU 功能。
- transient allocator、physical aliasing 与 multi-lane 仅在 release 明确纳入相应功能时触发 P1，不因评审中识别到 seam 就自动进入 0.16 实现范围。

### 禁止

- 禁止把 wgpu、UE、Vulkan 或任何单平台对象模型当作 Fluxel 公共 API 的默认答案；
- 禁止用 Rust trait presence、backend enum 或 adapter index 代替 runtime capability facts；
- 禁止把 GL default framebuffer 伪装成 Texture；
- 禁止把 native synchronization primitive、barrier、descriptor 或 swapchain 暴露到公共对象模型；
- 禁止把 split barrier、explicit heap、万能 BufferView、GPU address 或 generated-command signature 因为三家 native API 有类似机制就提前冻结；
- 禁止在没有 consumer 和多平台 evidence 时扩张 0.16 scope。

本报告建议作为下一步 API 声明修订和 plan 重写的输入。在这两项完成前，不应继续把新的 Vulkan increment 解释为 0.16 portable API 已冻结。
