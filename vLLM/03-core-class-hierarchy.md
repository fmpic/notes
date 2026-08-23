# 03：核心类关系、能力接口与运行时工厂

> **代码快照**：`0934b267906f8cd9459f287b31647c3ed5c58e01`。
> 行号、类型关系和调用链均以此提交为准；返回[指南入口](README.md)。

![vLLM 核心类关系图](diagrams/core-class-hierarchy.png)

vLLM 的对象关系不能用一棵传统继承树完整表达。主干同时使用 ABC、结构化 Protocol、能力 Mixin、对象组合、registry、factory、限定名导入和运行时基类注入。本篇先定义图例，再从 API 一直追到代表性 `LlamaForCausalLM`。

## 先读图例

本文和类关系图使用六种关系。阅读源码时，应先判断关系种类，再判断调用方向。

| 关系         | 图中含义                                          | Python/Rust 证据形态                            | 典型例子                    |
| ------------ | ------------------------------------------------- | ----------------------------------------------- | --------------------------- |
| 继承         | 子类复用父类实现或满足名义抽象                    | `class Child(Parent)`                           | `AsyncLLM(EngineClient)`    |
| Protocol     | 结构化能力契约；对象满足所需属性和方法即可        | `@runtime_checkable class X(Protocol)`          | `VllmModel`、`SupportsLoRA` |
| Mixin        | 通过多继承注入一组横切实现                        | 名称通常为 `*Mixin`，不独立拥有主生命周期       | `LoRAModelRunnerMixin`      |
| 组合         | 一个对象持有并驱动另一个对象                      | `self.x = X(...)`                               | `EngineCore.scheduler`      |
| factory 选择 | 配置或环境在多个实现类之间做运行时选择            | `get_class()`、`get_*_cls()`、registry lookup   | `Executor.get_class()`      |
| 运行时注入   | import、plugin 或修改基类在进程启动时改变可用行为 | `resolve_obj_by_qualname()`、`__bases__ += ...` | `worker_extension_cls`      |

有三条容易混淆的规则：

1. **类名含 “Protocol” 不等于继承 `typing.Protocol`。** `EngineClient` 的 docstring 称它为 protocol class，但源码实际是 `EngineClient(ABC)`；本篇按 Python 类型事实称为 ABC。
2. **“接口”不一定是 ABC。** `WorkerBase` 没有继承 `ABC`，许多方法只是抛 `NotImplementedError`；它是 interface-like base class，而非受 `abstractmethod` 强制的 ABC。
3. **Protocol 也可能出现在真实 MRO 中。** `LlamaForCausalLM` 直接列出 `SupportsLoRA`、`SupportsPP` 等 Protocol 基类，但它们表达的是能力契约，不是请求生命周期主干。

因此，图中实线继承、虚线 Protocol、Mixin 标记、组合菱形和 factory 虚线箭头各自表达不同事实，不能全部简化为“继承”。

## API 与 Engine 抽象

### 在线抽象：`EngineClient` ABC

`EngineClient` 定义 serving 层所依赖的异步能力：generation、pooling、abort、health、profiling、cache reset、sleep/wake、LoRA、pause/resume 与 shutdown。它还约定实现对象暴露 `VllmConfig`、`ModelConfig`、renderer 和 `InputProcessor`（[`EngineClient`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/engine/protocol.py#L41-L244)）。

当前 Python 在线实现关系很直接：

```text
EngineClient (ABC)
       △
       │ inheritance
   AsyncLLM
```

[`AsyncLLM(EngineClient)`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L71-L147) — 实现异步 serving contract，并**组合** renderer、`InputProcessor`、`OutputProcessor`、统计管理器与 `EngineCoreClient`。`AsyncLLM` 不是 `EngineCore` 的子类，也不直接持有 Worker。

公共名称 `AsyncLLMEngine` 只是 `AsyncLLM` 的兼容 alias，不形成新继承节点（[`AsyncLLMEngine = AsyncLLM`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/engine/async_llm_engine.py#L1-L7)）。

### 离线外观：Mixin 加对象组合

`LLM` 不继承 `EngineClient`。它把离线能力拆成三个 Mixin：

```text
BeamSearchOfflineMixin ─┐
PoolingOfflineMixin ────┼──> LLM
OfflineInferenceMixin ──┘       │
                                ◆ composition
                                ▼
                             LLMEngine
                                │
                                ◆
                                ▼
                        EngineCoreClient
```

[`LLM` 的直接基类](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/llm.py#L66-L74) — generation/pooling/beam-search 的同步批量外观。

[`LLM.__init__`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/llm.py#L336-L386) — 通过 `LLMEngine.from_engine_args()` 构造并保存 `self.llm_engine`，同时暴露 renderer、model config 和 input processor。

`LLMEngine` 自身也不是 `EngineClient` 子类。它是同步兼容外观，组合 V1 `InputProcessor`、`OutputProcessor` 和同步 `EngineCoreClient`（[`LLMEngine`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/llm_engine.py#L48-L139)）。公共 `vllm.engine.llm_engine.LLMEngine` 同样只是 V1 类的 alias（[`LLMEngine = V1LLMEngine`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/engine/llm_engine.py#L1-L7)）。

这组关系说明：“在线/离线”不是 `AsyncLLM` 与 `LLM` 的简单兄弟子类关系。在线入口通过 `EngineClient` ABC 解耦 serving；离线入口通过 Mixin 和 `LLMEngine` 组合提供同步批处理。

## `EngineCoreClient` 家族

`EngineCoreClient(ABC)` 是 frontend Engine 与 core 的传输抽象。它的继承树是：

```text
EngineCoreClient (ABC)
├── InprocClient
└── MPClient
    ├── SyncMPClient
    └── AsyncMPClient
        └── DPAsyncMPClient
            └── DPLBAsyncMPClient
```

- [`EngineCoreClient`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L78-L154) — 定义同步和异步 add/output/abort/utility 等 client contract。
- [`InprocClient`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L291-L340) — 组合一个同进程 `EngineCore`，直接调用 `step_fn()`、`add_request()` 与 `abort_requests()`。
- [`MPClient`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L482-L520) — 多进程共同基类，管理 ZMQ resources、后台 core process 与公共收发状态。
- [`SyncMPClient`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L794-L818) — 为同步 `LLMEngine` 提供阻塞式 output queue。
- [`AsyncMPClient`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L965-L993) — 为 `AsyncLLM` 提供 `asyncio.Queue` 与异步 API。
- [`DPAsyncMPClient`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L1250-L1427) — 加入多 EngineCore、coordinator wave 与外部 LB 语义。
- [`DPLBAsyncMPClient`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L1430-L1469) — 在 DP client 上增加内部 load balancing 和 request-to-engine 追踪。

继承树只解释复用关系，实际实例由 factory 决定：

| `multiprocess_mode` | `asyncio_mode` | DP 条件                      | 实际实现                             |
| ------------------- | -------------- | ---------------------------- | ------------------------------------ |
| `False`             | `False`        | 不适用                       | `InprocClient`                       |
| `True`              | `False`        | 任意                         | `SyncMPClient`                       |
| `True`              | `True`         | DP size = 1                  | `AsyncMPClient`                      |
| `True`              | `True`         | DP > 1 且 external LB        | `DPAsyncMPClient`                    |
| `True`              | `True`         | DP > 1 且 internal/hybrid LB | `DPLBAsyncMPClient`                  |
| `False`             | `True`         | 任意                         | 当前不支持，抛 `NotImplementedError` |

[`EngineCoreClient.make_client` 与 `make_async_mp_client`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L90-L141) — 上表的权威选择点。

所以，类图中应同时存在两组边：从 concrete client 指向父类的**继承边**，以及从 factory 指向运行时实现的**选择边**。只画继承树会漏掉“为什么在线总是进入 async MP client”和“为什么同一个 DP 基类产生两种 routing 行为”。

## `EngineCore` 与 Scheduler 的组合

### Core 是协调对象，Proc 是部署包装

`EngineCore` 没有抽象父类；它是 V1 内循环的具体协调对象。构造时加载 general plugin，实例化传入的 Executor class，初始化 KV cache/structured output manager，再通过 Scheduler factory 构造 `self.scheduler`（[`EngineCore.__init__`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L103-L178)）。

```text
EngineCore
├──◆ model_executor: Executor
├──◆ scheduler: SchedulerInterface
├──◆ structured_output_manager
└──◆ multimodal receiver cache

EngineCore
   △
   └── EngineCoreProc
           △
           ├── DPEngineCoreProc
           └── EngineCoreActor  ◁── EngineCoreActorMixin
```

- [`EngineCoreProc(EngineCore)`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L1000-L1059) — 增加 ZMQ、IO queues、handshake、后台 loop、engine index 和 tensor IPC receiver；调度语义仍继承自 `EngineCore`。
- [`DPEngineCoreProc(EngineCoreProc)`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L1868-L1895) — 为 MoE/DP cadence、wave 和 coordinator 状态增加包装。
- [`EngineCoreActor(EngineCoreActorMixin, EngineCoreProc)`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L2404-L2425) — 用 Mixin 叠加 Ray actor 生命周期，而不是重写 Scheduler/Worker 主链路。

`EngineCoreProc` 的继承是“部署能力增强”，`EngineCore.scheduler` 与 `model_executor` 则是对象组合。不要把 Scheduler 或 Executor 画成 `EngineCore` 基类。

### Scheduler 是 ABC 加可替换实现

`SchedulerInterface(ABC)` 定义 `schedule()`、grammar bitmask、`update_from_output()`、request add/finish 和 cache/connector 操作（[`SchedulerInterface`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/core/sched/interface.py#L37-L159)）。默认实现关系为：

```text
SchedulerInterface (ABC)
          △
          └── Scheduler
                  △
                  └── AsyncScheduler
```

[`Scheduler(SchedulerInterface)`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/core/sched/scheduler.py#L69-L119) — 默认统一调度器。

[`AsyncScheduler(Scheduler)`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/core/sched/async_scheduler.py#L12-L35) — 在默认 Scheduler 上增加异步 scheduling 所需状态与行为。

`SchedulerConfig.get_scheduler_cls()` 在 `scheduler_cls is None` 时根据 `async_scheduling` 返回默认或异步实现；也接受 class object 或限定名字符串，并明确警告这个 custom interface 不是稳定公共接口（[`SchedulerConfig.get_scheduler_cls`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/config/scheduler.py#L170-L194)）。

因此，类型标注是 `SchedulerInterface`，实际类由配置选择。`EngineCore` 对 Scheduler 的稳定依赖是 contract 与组合，不是对默认 `Scheduler` 的硬编码继承。

## Executor 家族

`Executor(ABC)` 把“在一个或多个设备上执行模型”抽象为 Worker collective RPC。它拥有统一 config 字段，并把 `execute_model()`、`sample_tokens()`、cache init、profile、sleep/wake 等操作下沉到 Worker（[`Executor`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/abstract.py#L37-L229)）。

主要名义继承关系：

```text
Executor (ABC)
├── UniProcExecutor
│   └── ExecutorWithExternalLauncher
├── MultiprocExecutor
│   └── RayExecutorV2
└── RayDistributedExecutor
```

- [`UniProcExecutor`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/uniproc_executor.py#L45-L124) — 在当前进程创建一个 `WorkerWrapperBase`，RPC 退化为直接 `run_method()`。
- [`ExecutorWithExternalLauncher`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/uniproc_executor.py#L150-L196) — 复用单 Worker Executor，由 torchrun-compatible launcher 在进程外创建多个 Engine/Executor。
- [`MultiprocExecutor`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/multiproc_executor.py#L103-L144) — 启动/连接多个 Worker process，控制面使用 `MessageQueue`。
- [`RayDistributedExecutor`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/ray_executor.py#L64-L89) — 直接以 Ray actor 管理 distributed Workers。
- [`RayExecutorV2(MultiprocExecutor)`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/ray_executor_v2.py#L219-L224) — 复用 MQ control plane，但 Worker 是 Ray actors；它不是 `RayDistributedExecutor` 的子类。

`Executor.get_class()` 是真正入口：

| `distributed_executor_backend` | 额外条件                                 | 返回类                         |
| ------------------------------ | ---------------------------------------- | ------------------------------ |
| class object                   | 必须为 `Executor` 子类                   | 原 class                       |
| `"uni"`                        | 无                                       | `UniProcExecutor`              |
| `"mp"`                         | 无                                       | `MultiprocExecutor`            |
| `"ray"`                        | `VLLM_USE_RAY_V2_EXECUTOR_BACKEND=False` | `RayDistributedExecutor`       |
| `"ray"`                        | Ray V2 env 为真                          | `RayExecutorV2`                |
| `"external_launcher"`          | 无                                       | `ExecutorWithExternalLauncher` |
| 其他字符串                     | 解析限定名且验证子类                     | OOT custom Executor            |

[`Executor.get_class`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/abstract.py#L48-L91) — backend 到具体类的动态映射。

一个常见误读是“Ray V2 一定继承旧 Ray Executor”。源码恰好相反：Ray V2 更接近 `MultiprocExecutor` 的 message-queue 模型。类关系能直接提示控制面实现差异，但具体 IPC 留给下一篇。

## Worker 家族

### `WorkerBase` 与 `WorkerWrapperBase` 是组合，不是继承

`WorkerBase` 汇总设备无关 config/state，并定义 device init、model load、KV cache、execute/sample、LoRA 和 shutdown 等 interface-like methods。它没有继承 `ABC`，也没有 `@abstractmethod`，所以 Python 不会在实例化时强制完整实现（[`WorkerBase`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/worker_base.py#L39-L181)）。

`WorkerWrapperBase` 表示 Executor 眼中的一个 process/rank。它负责环境、plugin、限定名解析和延迟构造，并通过 `self.worker` **组合**真正的 `WorkerBase` 实例（[`WorkerWrapperBase`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/worker_base.py#L187-L240)）。

```text
Executor
  ◆ owns one or more WorkerWrapperBase
                         │
                         ◆ self.worker
                         ▼
                    WorkerBase-like implementation
```

`UniProcExecutor` 也使用 wrapper；“wrapper”不等于“只用于 multiprocessing”。它统一了直接调用、MP process 和 Ray actor 中的 Worker 初始化外观。

### 平台 Worker 的名义继承

当前主要设备类关系是：

```text
WorkerBase
    △
    └── gpu_worker.Worker
            △
            ├── CPUWorker
            └── XPUWorker
```

[`gpu_worker.Worker(WorkerBase)`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu_worker.py#L128-L180) — CUDA/ROCm 主 Worker，拥有 profiler、elastic EP、fault-tolerance、weight transfer 和 ModelRunner 状态。

[`CPUWorker(Worker)`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/cpu_worker.py#L33-L80) 与 [`XPUWorker(Worker)`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/xpu_worker.py#L24-L63) — 复用通用 Worker 行为，再覆盖 device、memory、runner 或 backend 细节。类名 `gpu_worker.Worker` 不意味着其所有子类都在 GPU 上执行。

实际 Worker class 先由 platform 写入限定名：CUDA 使用 `vllm.v1.worker.gpu_worker.Worker`，CPU 使用 `CPUWorker`，XPU 使用 `XPUWorker`（[`CUDA platform 选择`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/platforms/cuda.py#L308-L315)、[`CPU platform 选择`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/platforms/cpu.py#L152-L160)、[`XPU platform 选择`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/platforms/xpu.py#L318-L325)）。

### 限定名解析与运行时 MRO 注入

`WorkerWrapperBase.init_worker()` 并不静态 import 当前 Worker。它：

1. 在 Worker process 中加载 general plugins。
2. 用 `resolve_obj_by_qualname(parallel_config.worker_cls)` 找到 class。
3. 若配置 `worker_extension_cls`，解析 extension class。
4. 检查 extension 的属性不会覆盖 Worker 现有属性。
5. 直接执行 `worker_class.__bases__ = worker_class.__bases__ + (worker_extension_cls,)`。
6. 最后实例化 `self.worker = worker_class(**kwargs)`。

[`WorkerWrapperBase.init_worker`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/worker_base.py#L230-L320) — Worker class factory 与运行时基类注入的完整证据。

这条边是本项目最重要的静态分析陷阱之一：扫描 class declaration 得不到 extension 后的 MRO。图中必须把 `worker_extension_cls` 画成“运行时注入”，不能画成仓库内固定父类。

## ModelRunner 与能力 Mixin

Worker 处理 device/distributed/process 边界，ModelRunner 处理 batch/KV/attention/forward/sampling。两者是组合：

```text
Worker
  ◆ self.model_runner
  ├── GPUModelRunner V1
  ├── GPUModelRunner V2
  ├── CPUModelRunner
  └── platform-specific runner
```

GPU Worker 在 device/distributed 初始化后，根据 `vllm_config.use_v2_model_runner` 动态构造两个同名但位于不同模块的类（[`Worker.init_device` 的 runner 选择](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu_worker.py#L375-L428)）：

- V1：`vllm.v1.worker.gpu_model_runner.GPUModelRunner`。
- V2：`vllm.v1.worker.gpu.model_runner.GPUModelRunner`。

V1 runner 的直接基类全部是横切能力 Mixin：

```text
LoRAModelRunnerMixin ───────┐
KVConnectorModelRunnerMixin ├──> GPUModelRunner V1
ECConnectorModelRunnerMixin ┘
```

[`GPUModelRunner V1`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu_model_runner.py#L453-L493) — 组合 LoRA、KV transfer 与 encoder-cache connector 行为。

[`LoRAModelRunnerMixin`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/lora_model_runner_mixin.py#L29-L45)、[`KVConnectorModelRunnerMixin`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/kv_connector_model_runner_mixin.py#L33-L59) 和 [`ECConnectorModelRunnerMixin`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/ec_connector_model_runner_mixin.py#L24-L45) — 各自提供一组可插入的横切操作，不独立代表一个可运行 runner。

V2 runner 当前只直接继承 `LoRAModelRunnerMixin`（[`GPUModelRunner V2`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu/model_runner.py#L126-L154)）。这不是“V2 是 V1 的子类”；它是并列实现，由 Worker factory branch 选择。

Worker 的执行方法负责 pipeline intermediate tensors，然后委托 `self.model_runner.execute_model()`；sampling 同样直接委托 runner（[`Worker.execute_model` 与 `sample_tokens`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu_worker.py#L1011-L1107)）。因此图中 `Worker → ModelRunner` 是**对象组合和调用边**，不是继承。

## 模型接口与代表性 Llama

### 所有模型依赖结构化接口，而非共同实体基类

`VllmModel` 是 `@runtime_checkable Protocol`，要求新式构造签名、`embed_input_ids()` 与 `forward(input_ids, positions, ...)`。生成模型再满足 `VllmModelForTextGeneration.compute_logits()`，pooling 模型满足对应 pooling Protocol（[`VllmModel` 与 generation/pooling Protocol`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/interfaces_base.py#L46-L149)）。

能力接口分布在 `interfaces.py`，例如：

- `SupportsMultiModal(Protocol)`：multimodal embedding 与 language/tower model 发现（[`SupportsMultiModal`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/interfaces.py#L91-L193)）。
- `SupportsLoRA(Protocol)`：LoRA module/embedding metadata（[`SupportsLoRA`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/interfaces.py#L545-L620)）。
- `SupportsPP(Protocol)`：pipeline-parallel intermediate tensor contract（[`SupportsPP`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/interfaces.py#L624-L683)）。
- `SupportsEagle`、`SupportsEagle3`：EAGLE speculative decoding capability（[`SupportsEagleBase` 与 variants](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/interfaces.py#L1303-L1432)）。
- `SupportsQuant`：量化 metadata/helper 的普通基类，不是 `Protocol`（[`SupportsQuant`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/interfaces.py#L1031-L1070)）。

Registry 通过结构化检查和 capability tags 判断 runner type、multimodal、PP、LoRA、EAGLE 等能力；模型不必继承一个名为 `BaseVllmModel` 的实体类。

### `LlamaForCausalLM` 的实际关系

代表性 Llama 既展示名义多继承，也展示深层对象组合：

```text
LocalArgmaxMixin ───────────┐
nn.Module ──────────────────┤
SupportsLoRA (Protocol) ────┤
SupportsPP (Protocol) ──────┼──> LlamaForCausalLM
SupportsEagle (Protocol) ───┤          │
SupportsEagle3 (Protocol) ──┤          ◆ self.model
SupportsQuant ──────────────┘          ▼
                                      LlamaModel(nn.Module, EagleModelMixin)
                                          │ ◆ layers
                                          ▼
                                      LlamaDecoderLayer(nn.Module)
                                       ├──◆ LlamaAttention(nn.Module)
                                       ├──◆ LlamaMLP(nn.Module)
                                       └──◆ RMSNorm

LlamaForCausalLM
├──◆ ParallelLMHead / PPMissingLayer
└──◆ LogitsProcessor
```

[`LlamaForCausalLM`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/llama.py#L446-L553) — 直接基类、`self.model`、PP-aware LM head、`forward()`、`compute_logits()` 与 weight loading。

[`LlamaModel`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/llama.py#L344-L443) — `nn.Module + EagleModelMixin`，组合 embeddings、按 PP rank 切分的 decoder layers、norm 与 intermediate tensor factory。

[`LlamaDecoderLayer`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/llama.py#L248-L327) — 组合 attention、MLP 和两个 RMSNorm，并实现 residual path。

[`LlamaAttention`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/llama.py#L122-L246) — 组合 tensor-parallel QKV/output projections、RoPE 与选择出的 attention implementation。

这棵树要分三层理解：

1. `nn.Module` 是 PyTorch 模块生命周期的名义主基类。
2. `Supports*` 与 `LocalArgmaxMixin` 表达可选能力，其中多数 `Supports*` 是 runtime Protocol。
3. `LlamaForCausalLM → LlamaModel → DecoderLayer → Attention/MLP` 是真正承载权重和 forward 的对象组合树。

把所有 `Supports*` 画成普通业务父类，会掩盖结构化能力设计；只画组合树，又会漏掉 LoRA、PP、EAGLE 和 quantization 的 capability contract。

## 模型注册与加载工厂

模型“类型选择”和“权重加载策略”是两套独立 factory。

### ModelRegistry：architecture 到 model class

Registry 内部先抽象注册项：

```text
_BaseRegisteredModel (ABC)
├── _RegisteredModel        # 已有 class object
└── _LazyRegisteredModel    # module_name + class_name

_ModelRegistry
  ◆ models: dict[architecture, _BaseRegisteredModel]
```

[`_BaseRegisteredModel`、eager 与 lazy 注册项](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/registry.py#L833-L881) — 区分已导入 class 与延迟模块描述。

`_LazyRegisteredModel.load_model_cls()` 才执行 `importlib.import_module()`；能力检查可走缓存或隔离子进程，避免主进程过早导入模型并初始化 CUDA（[`_LazyRegisteredModel` inspection/load](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/registry.py#L867-L1001)）。

`register_model(architecture, model_cls)` 接受 `nn.Module` class 或 `"<module>:<class>"` 字符串；后者生成 lazy entry（[`ModelRegistry.register_model`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/registry.py#L1032-L1081)）。

`resolve_model_cls()` 还会考虑：

- `model_impl="transformers"` 强制 Transformers backend；
- `model_impl="auto"` 的 Transformers fallback；
- runner/convert type 对 architecture 的规范化；
- in-tree 与 OOT registration；
- unsupported/previously-supported model 错误；
- 最终 platform model-architecture 验证。

[`ModelRegistry.resolve_model_cls`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/registry.py#L1215-L1335) — architecture 到 concrete `nn.Module` class 的动态选择。

[`get_model_architecture`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/model_loader/utils.py#L200-L260) — 调 registry 后再按 `convert_type` 包装 embedding/classification adapter，并缓存结果。

### ModelLoader：load format 到加载策略

`BaseModelLoader(ABC)` 定义 `download_model()` 与 `load_weights()`，公共 `load_model()` 负责创建模型、记录 inspection、调用具体权重加载和 post-load processing（[`BaseModelLoader`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/model_loader/base_loader.py#L25-L82)）。

`_LOAD_FORMAT_TO_MODEL_LOADER` 把 `auto`、`safetensors`、`bitsandbytes`、`dummy`、`sharded_state`、`tensorizer` 等格式映射到 loader class；`register_model_loader()` 允许扩展或覆盖，`get_model_loader()` 实例化选中的策略（[`ModelLoader registry 与 factory`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/model_loader/__init__.py#L27-L127)）。

最终加载链是：

```text
GPUModelRunner.load_model()
  → get_model_loader(load_config)
  → BaseModelLoader.load_model()
      → initialize_model()
          → get_model_architecture()
          → ModelRegistry.resolve_model_cls()
          → model_cls(vllm_config, prefix)
      → concrete_loader.load_weights(model, model_config)
      → post-load quant/kernel processing
  → optional LoRA / drafter / EPLB wrapping
```

[`GPUModelRunner.load_model`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu_model_runner.py#L5294-L5326) — runner 到 loader 的入口。

[`initialize_model`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/model_loader/utils.py#L41-L108) — model class 解析与新式/兼容构造签名。

关键区分：ModelRegistry 选择“实例化哪个模型类”，ModelLoader 选择“如何取得并写入权重”。二者都叫 registry/factory，但不能合并成同一继承树。

## 静态图看不到的动态边

下表汇总必须人工补到类图旁的动态关系。

| 动态边                      | 静态扫描为何会漏                         | 运行时依据                                  | 证据                                                                                                                        |
| --------------------------- | ---------------------------------------- | ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `AsyncLLMEngine → AsyncLLM` | 不是 class declaration                   | module alias                                | [`async_llm_engine.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/engine/async_llm_engine.py#L1-L7)               |
| Scheduler implementation    | `EngineCore` 只标注 `SchedulerInterface` | async flag、class object、限定名            | [`get_scheduler_cls`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/config/scheduler.py#L170-L194)                    |
| Executor implementation     | concrete class 在函数内延迟 import       | backend、Ray V2 env、class path             | [`Executor.get_class`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/abstract.py#L48-L91)                 |
| Worker implementation       | wrapper 只看到字符串                     | platform 写入 `worker_cls`                  | [`init_worker`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/worker_base.py#L230-L275)                     |
| Worker extension MRO        | class source中不存在该基类               | `worker_extension_cls` 修改 `__bases__`     | [`init_worker`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/worker_base.py#L265-L300)                     |
| GPUModelRunner V1/V2        | 两个模块存在同名 class                   | `use_v2_model_runner`                       | [`Worker.init_device`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu_worker.py#L407-L425)               |
| Model class                 | architecture 映射默认是 lazy string      | registry、model_impl、Transformers fallback | [`resolve_model_cls`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/registry.py#L1278-L1331)    |
| Loader class                | runner 只调用 factory                    | `LoadConfig.load_format` registry           | [`get_model_loader`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/model_loader/__init__.py#L122-L127) |
| General plugin effects      | entry point 在每个相关进程动态执行       | `VLLM_PLUGINS` 与 plugin group              | [`load_general_plugins`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/plugins/__init__.py#L31-L91)                   |
| Platform model gate         | model class 已解析仍可能被拒绝           | current platform                            | [`Platform.verify_model_arch`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/platforms/interface.py#L943-L954)        |

General plugins 可以在 process 0、EngineCore 和 Worker process 分别加载，且被要求可重复调用；`EngineCore.__init__` 和 `WorkerWrapperBase.init_worker()` 都显式触发它们。由 plugin 注册的新模型、loader 或行为不会出现在仓库静态 class 图中。

### 阅读一个动态对象的通用方法

以后遇到任何抽象类型，可按以下顺序定位：

1. 找 contract：ABC、Protocol 或 interface-like base。
2. 找名义实现：`class X(Base)`。
3. 找 factory：`get_class()`、`get_*_cls()`、`create_*()`。
4. 找 registry：字符串键、entry point 或限定名。
5. 找配置写入者：platform、`VllmConfig.__post_init__`、CLI/env。
6. 找组合点：谁把 concrete object 保存到 `self.*`。
7. 最后检查 plugin、Mixin 和 `__bases__` 注入。

这比只做“查找所有子类”更接近 vLLM 的真实运行时结构。

## 关键代码索引

本表只列本篇的类型与选择节点；全局稳定 ID 在[07：源码清单与代码索引](07-source-inventory-and-code-index.md)统一维护。

| 符号或目标                                 | 代码位置                                                                                                                                                                                                                                   | 本篇用途                                    |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------- |
| `EngineClient`                             | [`protocol.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/engine/protocol.py#L41-L244)                                                                                                                                           | 在线 serving ABC                            |
| `AsyncLLM`                                 | [`async_llm.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L71-L147)                                                                                                                                      | 当前 Python 在线 concrete Engine            |
| `LLM`                                      | [`llm.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/llm.py#L66-L74)                                                                                                                                                 | 三个 offline Mixin 的同步外观               |
| `LLMEngine`                                | [`llm_engine.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/llm_engine.py#L48-L139)                                                                                                                                    | 组合 V1 processor 与 core client 的兼容外观 |
| `EngineCoreClient.make_client`             | [`core_client.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L78-L141)                                                                                                                                  | client 家族 ABC 与 factory                  |
| `InprocClient`                             | [`core_client.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L291-L340)                                                                                                                                 | 同进程 core 组合                            |
| `MPClient` family                          | [`core_client.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L482-L520)                                                                                                                                 | sync/async/DP 多进程共同父类                |
| `EngineCore.__init__`                      | [`core.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L103-L178)                                                                                                                                               | Executor、Scheduler 和 manager 组合点       |
| `EngineCoreProc`                           | [`core.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L1000-L1059)                                                                                                                                             | core 的进程/IPC 部署子类                    |
| `SchedulerInterface`                       | [`interface.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/core/sched/interface.py#L37-L159)                                                                                                                                  | Scheduler ABC contract                      |
| `SchedulerConfig.get_scheduler_cls`        | [`scheduler.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/config/scheduler.py#L170-L194)                                                                                                                                        | 默认、async 与 custom Scheduler 选择        |
| `Executor.get_class`                       | [`abstract.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/abstract.py#L37-L91)                                                                                                                                       | Executor ABC 与 backend factory             |
| `UniProcExecutor`                          | [`uniproc_executor.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/uniproc_executor.py#L45-L124)                                                                                                                      | 直接 WorkerWrapper 调用                     |
| `MultiprocExecutor`                        | [`multiproc_executor.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/multiproc_executor.py#L103-L144)                                                                                                                 | MQ-based multi-worker 实现                  |
| `RayDistributedExecutor` / `RayExecutorV2` | [`ray_executor.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/ray_executor.py#L64-L89)、[`ray_executor_v2.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/ray_executor_v2.py#L219-L224)         | 两条不同的 Ray 继承路径                     |
| `WorkerBase`                               | [`worker_base.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/worker_base.py#L39-L181)                                                                                                                                  | interface-like device Worker 基类           |
| `WorkerWrapperBase.init_worker`            | [`worker_base.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/worker_base.py#L187-L319)                                                                                                                                 | Worker 组合、限定名解析和动态 MRO 注入      |
| GPU `Worker`                               | [`gpu_worker.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu_worker.py#L128-L180)                                                                                                                                   | 设备/distributed 生命周期与 runner 组合     |
| GPUModelRunner V1/V2                       | [`gpu_model_runner.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu_model_runner.py#L453-L493)、[`gpu/model_runner.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu/model_runner.py#L126-L154) | 并列 runner 与 Mixin 差异                   |
| `VllmModel` Protocols                      | [`interfaces_base.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/interfaces_base.py#L46-L149)                                                                                                              | 所有模型的结构化 contract                   |
| `Supports*` capabilities                   | [`interfaces.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/interfaces.py#L545-L683)                                                                                                                       | LoRA 与 PP 能力示例                         |
| `LlamaForCausalLM`                         | [`llama.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/llama.py#L446-L553)                                                                                                                                 | 代表性模型多继承与组合根                    |
| `_ModelRegistry`                           | [`registry.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/registry.py#L1032-L1081)                                                                                                                         | eager/lazy model 注册                       |
| `resolve_model_cls`                        | [`registry.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/registry.py#L1278-L1331)                                                                                                                         | model_impl、in-tree、Transformers/OOT 选择  |
| ModelLoader factory                        | [`model_loader/__init__.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/model_loader/__init__.py#L27-L127)                                                                                                         | load format 到 loader class                 |
| `BaseModelLoader.load_model`               | [`base_loader.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/model_loader/base_loader.py#L25-L82)                                                                                                                 | 模型实例化、权重写入和 post-load            |

---

上一页：[02：入口与请求生命周期](02-entrypoints-and-request-lifecycle.md) · 下一页：[04：进程与模块通信](04-processes-and-module-communication.md) · 返回：[指南入口](README.md)
