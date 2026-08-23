# 01：项目地图与 V1 总体架构

> **代码快照**：`0934b267906f8cd9459f287b31647c3ed5c58e01`。
> 行号、类型关系和调用链均以此提交为准；返回[指南入口](README.md)。

## 先看全局：V1 主链路

![vLLM 项目架构图](diagrams/project-architecture.png)

从一次文本生成看，vLLM 可以先抽象成七层：

```text
用户 / OpenAI client / Python caller
                ↓
CLI、LLM、FastAPI 或 Rust frontend
                ↓
输入解析、renderer、serving、OutputProcessor
                ↓
EngineCoreClient → EngineCore → Scheduler
                ↓
Executor → Worker → ModelRunner → model.forward / sampler
                ↓
platform、distributed runtime、KV cache、connectors
                ↓
PyTorch / Triton / CUDA / ROCm / CPU custom ops
```

在线 Python 主链路由 API Server 构造 `AsyncLLM`，请求经 `EngineCoreClient` 进入独立或进程内的 `EngineCore`。`EngineCore` 组合 Scheduler 和动态选择的 Executor；Executor 把本轮 `SchedulerOutput` 交给一个或多个 Worker，Worker 再委托 ModelRunner 准备输入、执行模型与采样。返回方向由 Scheduler 更新请求状态，`OutputProcessor` 解码并唤醒对应的异步输出流。

这张图有三个重要阅读原则：

1. **层是职责边界，不都是继承层级。** `LLM` 组合 `LLMEngine`，`EngineCore` 组合 Scheduler 和 Executor，Worker 组合 ModelRunner。
2. **逻辑调用与部署拓扑分开。** 同一条逻辑链可以使用 in-process、ZMQ 多进程、Ray 或 external launcher；这些差异在[进程与通信](04-processes-and-module-communication.md)中展开。
3. **Python 控制层与原生数据路径相互穿插。** ModelRunner 是 Python 编排对象，但一次 forward 会进入 PyTorch、Triton 或已注册的 C++/CUDA/ROCm/CPU custom op。

关键汇聚点可以先记住：

- [`AsyncLLM.from_vllm_config`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L201-L250) — 在线 V1 Engine 的构造入口，并先解析 Executor backend。
- [`EngineCoreClient.make_client`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L78-L141) — 根据同步/异步、多进程和 DP 模式选择传输实现。
- [`EngineCore.__init__`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L103-L169) — 组合 Executor、KV cache 与动态 Scheduler。
- [`EngineCore.step`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L581-L611) — 串起 schedule、execute、sample 和状态更新。
- [`GPUModelRunner.execute_model`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu_model_runner.py#L4158-L4240) — 消费 `SchedulerOutput`，准备并执行模型批次。

## 根目录地图

根目录不是按单一语言划分，而是把 Python runtime、原生内核、Rust 前端、验证资产和发行基础设施并列组织。

| 路径                                                                              | 主要职责                                                                                               | 新人何时进入                             |
| --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ | ---------------------------------------- |
| [`vllm/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/)                   | Python 主包；公开 API、配置、serving、V1 engine、distributed、model、platform 和 Python kernel wrapper | 阅读绝大多数功能的第一站                 |
| [`tests/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/tests/)                 | 单元、集成、平台、模型、kernel、distributed、entrypoint 与 eval 行为规格                               | 确认改动应保护的可观察行为               |
| [`examples/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/)           | 离线、在线、pooling、tool calling、multimodal、部署和集成示例                                          | 从最小可运行入口反查内部实现             |
| [`benchmarks/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/benchmarks/)       | serving、latency、throughput、scheduler、cache 与 kernel 性能测量                                      | 正确性确认后评估性能影响                 |
| [`csrc/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/csrc/)                   | C++、CUDA、HIP、CPU kernel 与 PyTorch dispatcher 注册                                                  | 追踪 `torch.ops.*` 的 schema 和实现      |
| [`rust/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/)                   | `vllm-rs` 前端、EngineCore client、protocol、renderer/parser、benchmark 与 PyO3 扩展                   | 研究实验性 Rust 前端或 Rust 构建         |
| [`cmake/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/cmake/)                 | CMake helper、CPU extension、外部依赖和 HIP 转换                                                       | 理解设备条件与 extension target          |
| [`requirements/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/requirements/)   | common、build、device、test、lint、docs 等依赖输入或锁定结果                                           | 搭建环境或解释 wheel 构建依赖            |
| [`docs/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/docs/)                   | 官方用户、部署、设计、模型、贡献和安全文档                                                             | 查用户承诺、正式流程和背景设计           |
| [`.buildkite/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/.buildkite/)       | 主要 CI pipeline、硬件测试、wheel/release 与 benchmark 作业                                            | 把本地验证映射到 CI                      |
| [`.github/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/.github/)             | GitHub workflow、模板、CODEOWNERS 与仓库自动化                                                         | 查 ownership 和轻量 workflow             |
| [`docker/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/docker/)               | CUDA、ROCm、CPU、XPU、TPU 等镜像与 bake 配置                                                           | 研究构建环境和部署镜像                   |
| [`tools/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/tools/)                 | 构建、检查、安装、profiling 和代码生成工具                                                             | 执行维护任务或追踪生成链                 |
| [`scripts/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/scripts/)             | 当前以 kernel autotune/benchmark 驱动脚本为主                                                          | 研究特定 kernel 调优流程                 |
| [`pyproject.toml`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/pyproject.toml) | PEP 517 build-system、项目元数据、CLI script 和工具配置                                                | 从 Python 安装入口开始追踪               |
| [`setup.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py)             | 设备判定、CMakeExtension、Rust extension 和 package data 汇聚                                          | 研究 wheel 中包含哪些构建产物            |
| [`CMakeLists.txt`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/CMakeLists.txt) | 顶层设备分支、源码集合与原生 target 定义                                                               | 从 Python extension 名追到 native source |

### 如何使用这张根目录表

- **改 Python 行为**：从 `vllm/` 的实现找到 `tests/` 中最近的行为规格，再找 `examples/` 是否有用户可见用法。
- **改模型或量化**：同时查看 `vllm/model_executor/`、`tests/models/` 或 `tests/quantization/`，必要时继续进入 `csrc/`、`vllm/kernels/` 和 `benchmarks/kernels/`。
- **改设备支持**：从 `vllm/platforms/` 到 Worker/ModelRunner，再沿 `setup.py → CMakeLists.txt → cmake/ → csrc/` 检查构建闭环。
- **改 serving 或 distributed**：从 `vllm/entrypoints/`、`vllm/v1/engine/` 和 `vllm/distributed/` 出发，对照 `tests/entrypoints/`、`tests/v1/` 与 `tests/distributed/`。

## `vllm/` 内部子系统地图

Python 主包承担控制流、模型实现和设备适配三大类职责。下面按“稳定职责边界”而不是目录大小来阅读。

| 子系统                | 职责与主要输入/输出                                                                            | 代表位置                                                                                                                                                                                                                                                                        |
| --------------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 公开 API 与 CLI       | 延迟公开符号，解析 `vllm` 子命令，构造配置并选择运行模式                                       | [`vllm/__init__.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/__init__.py#L16-L73)、[`vllm/entrypoints/cli/main.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/cli/main.py#L17-L97)                                                            |
| 离线入口              | 把批量 Python 输入和 sampling/pooling 参数转换为同步 Engine 请求                               | [`vllm/entrypoints/llm.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/llm.py#L66-L74)、[`LLM.__init__`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/llm.py#L336-L353)                                                                 |
| 在线 serving          | FastAPI route、OpenAI schema、renderer/parser、stream/non-stream response 与 middleware        | [`build_app`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/api_server.py#L196-L290)、[`OpenAIServingChat`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/chat_completion/serving.py#L112-L140)                               |
| 配置系统              | 把 CLI/env/model/platform 参数归一成 `VllmConfig` 及其 scheduler、parallel、cache、load 子配置 | [`vllm/config/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/config/)                                                                                                                                                                                                   |
| Engine 外观           | `AsyncLLM` 和兼容 `LLMEngine` 处理输入、输出流、统计、生命周期与 core client                   | [`vllm/v1/engine/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/)                                                                                                                                                                                             |
| Core 与调度           | `EngineCore` 协调 Scheduler、KV cache、structured/spec decode 与 Executor                      | [`vllm/v1/core/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/core/)、[`vllm/v1/engine/core.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L103-L169)                                                                                    |
| Executor              | 为 uni-process、multiprocessing、Ray 或 external backend 提供统一 worker RPC 抽象              | [`vllm/v1/executor/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/)、[`Executor.get_class`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/abstract.py#L37-L91)                                                                            |
| Worker 与 ModelRunner | 初始化设备和 distributed 环境、管理 KV cache、准备 batch、执行 forward 与采样                  | [`vllm/v1/worker/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/)                                                                                                                                                                                             |
| 模型执行              | 模型 registry、结构化接口、loader、模型类、量化和通用 layer                                    | [`vllm/model_executor/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/)                                                                                                                                                                                   |
| Distributed           | parallel state、collective、communication op、device communicator 与跨进程协调                 | [`vllm/distributed/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/distributed/)                                                                                                                                                                                         |
| Platform              | 探测 CUDA、ROCm、CPU、XPU 等能力并选择 Worker、attention backend 和 kernel import              | [`vllm/platforms/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/platforms/)                                                                                                                                                                                             |
| 输入与多模态          | prompt、token、multimodal 数据的解析、校验、缓存和 processor 接口                              | [`vllm/inputs/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/inputs/)、[`vllm/multimodal/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/multimodal/)                                                                                                            |
| Renderer 与 parser    | chat template、tool/reasoning 输出解析以及 Python/Rust 前端共享的语义层                        | [`vllm/renderers/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/renderers/)、[`vllm/parser/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/parser/)、[`vllm/tool_parsers/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/tool_parsers/)                   |
| Compilation 与 kernel | torch.compile 图变换、Triton/CuTe/Python kernel 和 custom-op wrapper                           | [`vllm/compilation/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/compilation/)、[`vllm/kernels/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/kernels/)、[`vllm/_custom_ops.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/_custom_ops.py#L166-L276) |
| 横切功能              | LoRA、KV transfer/connectors、structured output、reasoning、logging、tracing 和 profiling      | [`vllm/lora/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/lora/)、[`vllm/v1/structured_output/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/structured_output/)、[`vllm/tracing/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/tracing/)           |
| 插件                  | 通过 entry point 或限定名扩展 platform、model、IO、general 与 endpoint 行为                    | [`vllm/plugins/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/plugins/)                                                                                                                                                                                                 |

目录边界并不等于运行时边界。例如 `vllm/model_executor/` 中的模型由 `vllm/v1/worker/` 的 ModelRunner 驱动；`vllm/entrypoints/` 的 serving 对象依赖 `vllm/engine/protocol.py` 中的 `EngineClient` 抽象，而实际实现位于 `vllm/v1/engine/async_llm.py`。

## 分层职责

### 入口与输入层

这一层把用户协议变成 Engine 可以消费的类型，同时把 Engine 输出变回用户协议。

- `vllm.__init__` 使用模块级 `__getattr__` 延迟加载公开对象，导入 `vllm` 不代表所有 engine/model 模块都会立即加载（[`MODULE_ATTRS` 与 `__getattr__`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/__init__.py#L16-L73)）。
- CLI 通过各命令模块的 `cmd_init()` 在运行时注册子命令，并将选中的 `dispatch_function` 作为真正入口（[`main`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/cli/main.py#L73-L97)）。
- FastAPI 根据模型的 `supported_tasks` 挂载 generate、pooling、speech 等 route，endpoint plugin 最后进入（[`build_app`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/api_server.py#L196-L290)）。
- serving/rendering 层拥有 OpenAI schema、chat template、tool/reasoning parser 和流式编码；它不直接调度 GPU。

### Engine client 与输出层

Engine 外观隔离了调用协议和 core 部署方式：

- 在线 `AsyncLLM` 实现异步 `EngineClient`，组合 `InputProcessor`、`OutputProcessor`、统计组件和异步 MP core client（[`AsyncLLM.__init__`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L71-L147)）。
- 离线 `LLM` 组合同步 `LLMEngine`；`LLMEngine.step()` 拉取 core 输出后交给同一类 V1 输出处理组件。
- `EngineCoreClient` 是逻辑传输边界：in-process 模式直接调用 core，多进程模式通过 IPC 连接后台 core。它不实现 schedule 或 model forward。

### Core 与 Scheduler 层

`EngineCore` 拥有全局请求推进循环，Scheduler 拥有可调度状态：

- Scheduler 在统一 token 预算下管理 waiting/running requests、KV blocks、prefill、decode、chunked prefill、prefix caching 和 speculative decoding；它不是两套割裂的 prefill/decode scheduler（[`Scheduler.schedule`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/core/sched/scheduler.py#L427-L447)）。
- `EngineCore.step()` 获取 `SchedulerOutput`，异步触发模型执行，必要时执行独立采样，再让 Scheduler 消费 `ModelRunnerOutput`（[`EngineCore.step`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L581-L611)）。
- pipeline parallel 或 batch queue 模式可以重叠调度与执行，因此“单轮四步”是语义顺序，不保证所有部署都完全串行（[`step_with_batch_queue`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L622-L685)）。

### Executor、Worker 与 ModelRunner 层

这三层分别解决“发给谁”“进程/设备如何准备”和“模型如何执行”：

- Executor 根据 `distributed_executor_backend` 选择 uni、multiprocessing、Ray、external launcher 或自定义限定名实现（[`Executor.get_class`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/abstract.py#L37-L91)）。
- Worker 初始化设备、distributed process group 和 KV cache，并处理 pipeline intermediate tensors；平台 Worker 可以替换 runner 实现。
- ModelRunner 保存 persistent batch state，准备 attention metadata，调用模型 forward 与 sampler；默认 GPU runner 还组合 LoRA、KV Connector 和 EC Connector Mixin（[`GPUModelRunner`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu_model_runner.py#L453-L473)）。
- 模型类由 ModelRegistry 和 ModelLoader 动态解析/加载，不要求都继承单一 vLLM 实体基类；结构化 Protocol 与 `Supports*` Mixin 表达能力。

### Platform 与原生内核层

platform 层把同一上层语义映射到设备能力：

- platform 决定 Worker、attention backend、dtype/compute capability、device communicator 以及应导入哪些 kernel extension。
- Python `_custom_ops.py` 通常只是参数整理、fake/meta 注册和 `torch.ops.*` 调用包装；真正 schema 与 implementation 由动态加载的共享库向 PyTorch dispatcher 注册。
- CUDA、ROCm 和 CPU 源码可能共享 schema，但使用不同 dispatch key、源码集合和构建 target；Triton/CuTe kernel 则可从 Python 编译路径进入。

构建和注册闭环在[05：构建与原生扩展](05-build-and-native-extensions.md)单独展开。

## V1 主线与兼容外观

名称中的历史包袱容易让新人误判当前架构。这个快照应按下表理解：

| 名称             | 当前定位                                         | 与 V1 的关系                                                        |
| ---------------- | ------------------------------------------------ | ------------------------------------------------------------------- |
| `AsyncLLM`       | Python 在线 serving 的实际异步 Engine 实现       | 直接构造 V1 processors 和异步 `EngineCoreClient`                    |
| `LLM`            | 用户同步离线 API 与多个 offline capability Mixin | 组合当前 `LLMEngine`                                                |
| `LLMEngine`      | 保留同步 Engine API 的兼容外观                   | 内部使用 V1 `InputProcessor`、`OutputProcessor`、`EngineCoreClient` |
| `AsyncLLMEngine` | 公共兼容名称/旧入口语境                          | 不应画成当前 API Server 实际创建的对象                              |
| `EngineCore`     | V1 调度和执行协调核心                            | 为在线、离线和 Rust 前端共享                                        |
| `EngineCoreProc` | 多进程部署包装                                   | 在 `EngineCore` 上增加 IPC、IO 和主循环                             |
| Rust frontend    | 实验性 HTTP/rendering/streaming 前端             | 通过兼容协议连接 Python `EngineCoreProc`                            |

源码证据：

- [`LLMEngine`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/llm_engine.py#L48-L60) — 类注释明确其兼容定位；构造路径使用 V1 processors 与 core client。
- [`build_async_engine_client_from_engine_args`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/api_server.py#L145-L193) — 当前 Python API Server 实例化 `AsyncLLM`。
- [`RustFrontendProcessManager`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/utils.py#L325-L390) — Python supervisor 向 Rust 子进程传递监听 fd 与 EngineCore transport 地址。

因此，“V1 主线”描述的是当前请求所经的核心组件集合，不意味着仓库里只有一组部署实现，也不意味着所有带 legacy 注释的 API 都在走 V0 内核。

## 横切能力与扩展点

vLLM 的可扩展性主要来自 factory、registry、plugin、Protocol 与 Mixin，而不是一个巨大的继承树。

| 扩展点               | 选择依据                                                                | 影响范围                                 | 阅读入口                                                                                                                               |
| -------------------- | ----------------------------------------------------------------------- | ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Scheduler class      | `scheduler_config.get_scheduler_cls()` 与限定名                         | 请求排序、预算和调度策略                 | [`EngineCore.__init__`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L145-L169)                               |
| Executor backend     | parallel config、Ray V2 env、自定义 class path                          | Worker 启动与 RPC 机制                   | [`Executor.get_class`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/abstract.py#L37-L91)                            |
| Worker class         | platform、`worker_cls`、`worker_extension_cls`                          | 设备初始化、runner、运行时 MRO           | [`WorkerWrapperBase.init_worker`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/worker_base.py#L243-L287)              |
| ModelRunner          | platform 与 `use_v2_model_runner`                                       | batch、KV、forward、sampling             | [`Worker.__init__`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu_worker.py#L128-L180)                             |
| Model implementation | HF architecture、ModelRegistry、Transformers fallback、OOT registration | 实际 `nn.Module` 与能力接口              | [`ModelRegistry.resolve_model_cls`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/registry.py#L1278-L1331) |
| ModelLoader          | `LoadConfig.load_format` registry                                       | 初始化方式与权重读取                     | [`get_model_loader`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/model_loader/__init__.py#L99-L141)             |
| Platform             | entry point、设备环境与 capability                                      | Worker、attention backend、kernel import | [`vllm/platforms/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/platforms/)                                                    |
| Endpoint plugin      | Python entry point                                                      | FastAPI route 与覆盖顺序                 | [`build_app`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/api_server.py#L276-L290)                          |
| Connector/Mixin      | 配置与类组合                                                            | KV transfer、EC、LoRA 等横切能力         | [`GPUModelRunner`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu_model_runner.py#L453-L473)                        |
| Native op            | platform import 与 PyTorch dispatcher                                   | Python 调用到设备 kernel                 | [`Platform.import_kernels`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/platforms/interface.py#L362-L370)                      |

这些扩展点带来两条阅读规则：

1. 搜到抽象类后，必须继续找 factory 或 registry，确认当前配置究竟构造哪个实现。
2. 看到类的静态基类后，仍需检查 plugin、Mixin 和 `worker_extension_cls` 等运行时注入，不能只依赖静态 MRO 推断行为。

更完整的类型关系见[03：核心类关系](03-core-class-hierarchy.md)，进程和协议影响见[04：进程与模块通信](04-processes-and-module-communication.md)。

## 从地图跳到代码

| 你要回答的问题                                      | 下一步                                                            | 首要源码区域                                                   |
| --------------------------------------------------- | ----------------------------------------------------------------- | -------------------------------------------------------------- |
| 一次在线请求经过哪些方法和类型？                    | [02：入口与请求生命周期](02-entrypoints-and-request-lifecycle.md) | `vllm/entrypoints/`、`vllm/v1/engine/`、`vllm/v1/core/`        |
| 为什么运行时实例不是静态搜索看到的基类？            | [03：核心类关系](03-core-class-hierarchy.md)                      | engine client、executor、worker、model registry/loader         |
| 请求跨了哪些进程，控制消息和模型张量走什么通道？    | [04：进程与模块通信](04-processes-and-module-communication.md)    | core client/proc、multiproc executor、distributed、Rust client |
| `torch.ops._C.*` 最终在哪里实现，wheel 怎样带上它？ | [05：构建与原生扩展](05-build-and-native-extensions.md)           | `_custom_ops.py`、`csrc/`、CMake、setup.py、Cargo              |
| 哪些示例最适合验证我的理解？                        | [06：示例学习路线](06-examples-learning-path.md)                  | `examples/`、nearby tests、`benchmarks/`                       |
| 一个结论对应哪个符号和行范围？                      | [07：源码清单与代码索引](07-source-inventory-and-code-index.md)   | 集中索引与扫描证据                                             |

阅读任何具体功能时，都建议形成一个最小闭环：**入口或测试 → 配置/factory → 当前实现 → 输出或断言**。只阅读类定义往往会漏掉 vLLM 最关键的运行时选择关系。

---

上一页：[指南入口](README.md) · 下一页：[02：入口与请求生命周期](02-entrypoints-and-request-lifecycle.md) · 返回：[指南入口](README.md)
