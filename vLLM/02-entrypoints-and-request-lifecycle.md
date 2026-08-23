# 02：入口与请求生命周期

> **代码快照**：`0934b267906f8cd9459f287b31647c3ed5c58e01`。
> 行号、类型关系和调用链均以此提交为准；返回[指南入口](README.md)。

![在线请求生命周期图](diagrams/online-request-lifecycle.png)

本篇沿“启动 → 接收 → 调度 → 执行 → 返回 → 结束”解释 vLLM 的入口与请求生命周期。主线是当前 V1 Python 在线路径，随后对照离线 `LLM`、兼容名称和实验性 Rust 前端。类关系与进程通信分别留给[03：核心类关系](03-core-class-hierarchy.md)和[04：进程与模块通信](04-processes-and-module-communication.md)。

## 公共导入与 CLI 分派

### `import vllm` 不会立即导入所有实现

包初始化先加载版本与环境覆盖，再把公开名称保存在 `MODULE_ATTRS`。运行时只有访问某个属性，模块级 `__getattr__` 才通过 `import_module()` 找到真正对象：

```text
from vllm import LLM
        ↓
vllm.__getattr__("LLM")
        ↓
import vllm.entrypoints.llm
        ↓
返回 vllm.entrypoints.llm.LLM
```

[`vllm.MODULE_ATTRS` 与 `vllm.__getattr__`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/__init__.py#L16-L73) — 公开 `LLM`、`LLMEngine`、`AsyncLLMEngine`、参数和输出类型的懒加载表。这个设计避免普通导入过早触发 platform、CUDA 或重量级模型模块初始化。

### `vllm` 命令如何找到子命令

安装元数据只注册一个 console script：

[`project.scripts.vllm`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/pyproject.toml#L43-L47) — 将 shell 命令 `vllm` 指向 `vllm.entrypoints.cli.main:main`。

`main()` 在函数体内延迟导入命令模块。每个模块的 `cmd_init()` 返回一个或多个 `CLISubcommand`，随后 CLI 为每个对象注册 parser，并把对象的 `cmd` 保存为 `dispatch_function`：

```text
vllm <subcommand> ...
  → cli.main.main()
  → cmd_module.cmd_init()
  → subparser_init(...).set_defaults(dispatch_function=cmd.cmd)
  → validate(args)
  → dispatch_function(args)
```

[`vllm.entrypoints.cli.main.main`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/cli/main.py#L16-L97) — 注册 `serve`、`launch`、`bench`、`collect-env`、`run-batch` 以及客户端侧 `chat`/`complete` 等命令，并完成校验与分派。这里的 `openai` 模块不是服务端入口；它注册的是连接已有 OpenAI-compatible server 的 `chat` 和 `complete` 客户端命令（[`openai.cmd_init`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/cli/openai.py#L299-L312)）。

`vllm serve` 的 `ServeSubcommand.cmd()` 先处理 model positional arg 和 gRPC 特例，再根据 headless、API process 数、DP load-balancing 模式和 Rust frontend 配置选择启动路径（[`ServeSubcommand.cmd`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/cli/serve.py#L50-L149)）。因此 CLI 不是固定地“在当前进程启动一个 FastAPI app”。

## 离线入口 `LLM`

### 构造阶段

`LLM` 是面向 Python 用户的同步批量 API。它通过多个 offline Mixin 组合 generation、pooling 和 beam search 能力，但真正的 Engine 由对象组合获得：

```text
LLM.__init__
  → EngineArgs(...)
  → LLMEngine.from_engine_args(...)
  → Executor.get_class(vllm_config)
  → InputProcessor + OutputProcessor
  → EngineCoreClient.make_client(asyncio_mode=False)
```

[`LLM.__init__`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/llm.py#L336-L386) — 根据传入参数构造 `EngineArgs`，再保存 `LLMEngine`、renderer、input processor 和模型能力。

[`LLMEngine.__init__`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/llm_engine.py#L48-L139) — 构造 renderer、`InputProcessor`、`OutputProcessor` 和同步 `EngineCoreClient`。默认是否使用后台 V1 core 进程受 `VLLM_ENABLE_V1_MULTIPROCESSING` 与显式参数影响；不是由 `LLM` 类本身写死。

### 一次 `LLM.generate()`

离线生成把一组 prompt 一次性交给引擎，以便 Scheduler 在显存约束下持续批处理：

1. `LLM.generate()` 校验 runner 类型并补默认 `SamplingParams`。
2. `OfflineInferenceMixin._run_completion()` 把单值或序列参数对齐到每个 prompt，执行同步 renderer/preprocessing。
3. `_add_request()` 为每个 prompt 分配递增字符串 request ID，调用 `LLMEngine.add_request()`。
4. `LLMEngine.add_request()` 把 `PromptType`/`EngineInput` 转成 `EngineCoreRequest`，在 `OutputProcessor` 建立请求状态，再交给 `EngineCoreClient`。
5. `_run_engine()` 在仍有未完成请求时反复调用 `LLMEngine.step()`。
6. `step()` 获取 `EngineCoreOutputs`，经 `OutputProcessor.process_outputs()` 形成 `RequestOutput`；完成结果按数值 request ID 排序，保持与输入顺序一致。

对应证据：

- [`LLM.generate`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/llm.py#L411-L470) — 用户 API、默认参数和 generation runner 校验。
- [`OfflineInferenceMixin._add_completion_requests` 与 `_run_completion`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/offline_utils.py#L290-L350) — prompt/params/LoRA/priority 对齐、render 和入队。
- [`LLMEngine.add_request`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/llm_engine.py#L218-L295) — 输入处理、`n > 1` child request fan-out 与 core 入队。
- [`OfflineInferenceMixin._run_engine`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/offline_utils.py#L573-L626) — 同步 step 循环、完成收集和输出排序。
- [`LLMEngine.step`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/llm_engine.py#L296-L335) — core output 处理、stop string 引发的 abort 和统计更新。

`n > 1` 时，一个外部请求会被 `ParentRequest` 展开为多个内部 child request。每个 child 单独进入 core，`OutputProcessor` 再按 parent/index 聚合。因此 request ID 既是生命周期键，也是多候选输出复用同一用户请求的关联键。

## 在线 Python 入口

### 从 `vllm serve` 到运行中的 app

单 Python API Server 路径如下：

```text
ServeSubcommand.cmd
  → uvloop.run(run_server(args))
  → setup_server()                         # 监听 socket
  → run_server_worker()
  → build_async_engine_client()            # async context manager
  → AsyncLLM.from_vllm_config()
  → build_and_serve()
      → engine_client.get_supported_tasks()
      → build_app()
      → init_app_state()
      → serve_http()
```

[`run_server` 与 `run_server_worker`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/api_server.py#L748-L796) — 打开 socket、加载 tool/reasoning parser plugin，并用 context manager 管理 Engine 生命周期。

[`build_async_engine_client_from_engine_args`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/api_server.py#L147-L193) — 创建 `VllmConfig`，实例化当前 V1 `AsyncLLM`，在退出时按配置 timeout 调用 shutdown。函数 docstring 中保留的 `AsyncLLMEngine` 用语是兼容历史，实际 import 和实例均为 `AsyncLLM`。

[`build_and_serve`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/api_server.py#L660-L705) — 从 Engine 查询 `supported_tasks`，完成 app 和 state 初始化后交给 HTTP server。

### app 构造与 state 注入是两个阶段

`build_app()` 只建立 FastAPI、middleware、exception handler 与 routers。generate、pooling、speech、scale-out、SageMaker、fault tolerance 和 endpoint plugin 都按配置或 `supported_tasks` 动态挂载；endpoint plugin 最后挂载（[`build_app`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/api_server.py#L196-L290)）。

`init_app_state()` 再把 `engine_client`、`VllmConfig`、model list、`OnlineRenderer`、tokenization/derendering 与各 serving 对象注入 `app.state`（[`init_app_state`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/api_server.py#L359-L489)）。Route 因而只从 state 取到相应 handler，不直接构造或持有 Scheduler、Worker。

以 `/v1/chat/completions` 为例：

- FastAPI/Pydantic 先将 JSON body 解析为 `ChatCompletionRequest`。
- route 上的 `with_cancellation` 与 `load_aware_call` 包装断连取消和负载计数。
- route 从 `request.app.state.openai_serving_chat` 取 `OpenAIServingChat`。
- handler 返回完整 `ChatCompletionResponse`、`ErrorResponse` 或异步字符串 generator；route 分别包装为 JSON 或 `StreamingResponse`。

[`create_chat_completion` route](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/chat_completion/api_router.py#L53-L76) — HTTP schema 与 serving handler 的边界。

## 一次在线请求的正向路径

以下以非 beam-search 的 chat generation 为代表。其他 completion、pooling 或 speech route 在入口 schema 和输出编码上不同，但会复用 `EngineClient` 与 core 边界。

### 1. OpenAI schema 变成 Engine input

`OpenAIServingChat._create_chat_completion()` 先做 model/engine health 校验，再由 `OnlineRenderer.render_chat()` 应用 chat template、解析 multimodal/tool/reasoning 输入，得到 conversation 与一个或多个 `EngineInput`。随后它：

1. 生成 `chatcmpl-*` request ID，并把 metadata 放入原始 HTTP request state。
2. 解析 LoRA adapter 与可选 DP rank。
3. 按最大模型长度和请求参数生成 `SamplingParams`。
4. 对每个 engine input 生成唯一 sub-request ID。
5. 调用 `self.engine_client.generate(...)` 得到异步 `RequestOutput` generator。

[`OpenAIServingChat._create_chat_completion`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/chat_completion/serving.py#L255-L408) — renderer、sampling 参数、request ID、LoRA/DP 信息与 Engine 调用的汇聚点。

### 2. `AsyncLLM` 建立 request-scoped collector

`AsyncLLM.generate()` 并不自己执行 GPU 工作。它调用 `add_request()`：

- `InputProcessor.process_inputs()` 把 `EngineInput` 与参数规范化为 `EngineCoreRequest`。
- `assign_request_id()` 处理当前 wave/内部 ID。
- 为请求创建 `RequestOutputCollector`。
- `OutputProcessor.add_request()` 在 API process 保存 detokenization/聚合状态。
- `engine_core.add_request_async()` 把 `EngineCoreRequest` 送到 core。

[`AsyncLLM.add_request` 与 `_add_request`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L281-L417) — frontend-local 输出状态与 core request 在这里同时注册。`n > 1` 同样展开 child requests，但共享 collector。

[`AsyncLLM.generate`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L525-L637) — API Server 调用的主异步 generator；它从 collector 取出 `RequestOutput` 并逐项 `yield`，直到 `finished`。

### 3. `EngineCoreClient` 选择 core 边界

`AsyncLLM` 使用 `asyncio_mode=True` 的多进程 client。Factory 再根据 data parallel 和 load-balancing 配置选择 `AsyncMPClient`、`DPAsyncMPClient` 或 `DPLBAsyncMPClient`。同步 `LLMEngine` 则可以选 `InprocClient` 或 `SyncMPClient`（[`EngineCoreClient.make_client`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L78-L141)）。

逻辑上，这一层只发送 `EngineCoreRequest` 和接收 `EngineCoreOutputs`。ZMQ socket 类型、握手和 DP coordinator 在[04：进程与模块通信](04-processes-and-module-communication.md)展开。

### 4. Core 产生本轮工作

`EngineCore` 收到 add request 后将它交给 Scheduler。每个 core step：

```text
Scheduler.schedule()
  → SchedulerOutput
  → Executor.execute_model(..., non_block=True)
  → ModelRunnerOutput 或独立 sample_tokens()
  → Scheduler.update_from_output()
  → {client_index: EngineCoreOutputs}
```

[`EngineCore.step`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L581-L611) — 一轮 schedule、execute、sample 与状态更新的语义顺序。

Scheduler 在同一请求状态上推进 prefill、decode、chunked prefill、prefix caching、structured output 和 speculative decoding。它不会把 HTTP request 原样传给 Worker；Worker/ModelRunner 消费的是 Scheduler 生成的本轮 token 与 block 工作描述。

## 执行、采样与反向输出路径

### 从 ModelRunner 回到 Scheduler

Executor 将 `SchedulerOutput` 分发给 Worker，Worker 委托 ModelRunner 准备 batch、KV/attention metadata 并执行模型。不同配置可以让 forward 和 sampling 合并或分开：`EngineCore.step()` 先等待 `execute_model()` future；若返回 `None`，再调用 `sample_tokens()`。

ModelRunner 返回的 `ModelRunnerOutput` 包含 sampled token IDs、logprobs、prompt logprobs、pooler output、KV connector output 和执行统计等。`Scheduler.update_from_output()` 用它更新每个 request 的 computed/in-flight token、停止条件、KV block 生命周期和完成原因，最后按 `client_index` 形成 `EngineCoreOutputs`（[`Scheduler.update_from_output`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/core/sched/scheduler.py#L1580-L1674)）。

这一步解释了三种输出类型的区别：

| 类型                | 产生者 → 消费者                      | 含义                                                   |
| ------------------- | ------------------------------------ | ------------------------------------------------------ |
| `SchedulerOutput`   | Scheduler → Executor/ModelRunner     | 本轮应该执行哪些 token、block 和结构化约束             |
| `ModelRunnerOutput` | ModelRunner → Scheduler              | forward、sampling 或 pooling 的设备执行结果            |
| `EngineCoreOutputs` | Scheduler/EngineCore → Engine client | 按请求和 client 分组、可以进入 frontend 输出处理的结果 |

### Python 在线反向路径

`AsyncLLM` 为所有请求共享一个后台 output handler，而不是每个 HTTP task 直接轮询 core：

1. `engine_core.get_output_async()` 拉取一批 `EngineCoreOutputs`。
2. 为避免长时间阻塞 event loop，按 `VLLM_V1_OUTPUT_PROC_CHUNK_SIZE` 分块。
3. `OutputProcessor.process_outputs()` detokenize、聚合 parent/child、检查 stop string，并把 `RequestOutput` 推到对应 collector。
4. 若 stop string 在 frontend 被确认，异步 abort core 中仍在运行的 request。
5. 记录 scheduler、request 和 multimodal cache 统计。

[`AsyncLLM._run_output_handler`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L638-L709) — core batch output 到每请求 collector 的反向分发循环。

`AsyncLLM.generate()` 正在等待同一个 collector。收到未结束的 `RequestOutput` 后，`OpenAIServingChat` 根据 `request.stream` 走两条消费路径：

- **stream**：`chat_completion_stream_generator()` 把增量 token、tool/reasoning parser 状态、usage 和 finish reason 编码为 SSE chunk。
- **non-stream**：`chat_completion_full_generator()` 消费到结束，再构造一个完整 JSON response。

[`OpenAIServingChat` 的 stream/full 分叉](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/chat_completion/serving.py#L374-L408) — 分叉发生在 Engine generator 创建之后，因此两种 HTTP 响应复用同一 core 生命周期。

## 取消、结束与后台任务

### 正常完成

正常 generation 以 `RequestOutput.finished` 结束。Scheduler 和 `OutputProcessor` 各自清理自己拥有的状态；`AsyncLLM.generate()` 不会再 yield 内部 `STREAM_FINISHED` sentinel，并在 `finally` 关闭 collector。离线路径则在 `has_unfinished_requests()` 为假后结束 step loop。

不要把“采样到一个 token”“发送一个 SSE chunk”和“请求完成”画成同一事件。一个 core step 可以推进多个请求，一个 `EngineCoreOutputs` 可以包含多个请求输出，一个 HTTP 请求也可能包含 `n > 1` child sequence。

### HTTP 断连或 generator 被取消

`with_cancellation` 同时等待 route handler 和 `http.disconnect`；非 streaming handler 阶段若先断连，会取消正在执行的 handler task。返回 `StreamingResponse` 后，Starlette 接管断连监听（[`with_cancellation`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/serve/utils/api_utils.py#L35-L91)）。

当 `AsyncLLM.generate()` 因 `CancelledError`、`GeneratorExit` 或非预期错误退出时，它调用 `abort()`：

```text
HTTP disconnect / consumer drops generator
  → AsyncLLM.generate cancelled
  → OutputProcessor.abort_requests(...)
  → EngineCoreClient.abort_requests_async(...)
  → Scheduler/core 停止并释放请求资源
```

[`AsyncLLM.generate` 异常与清理分支](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L576-L637) — 区分 client cancel、engine dead、validation、input stream 和 unexpected error。

[`AsyncLLM.abort`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L710-L724) — 同时终止 frontend 输出状态与 core request。stop string 导致的 frontend-side finish 也走显式 abort，防止 core 继续生成无用 token。

### Engine 或 output handler 失败

Engine dead error 不再发送 abort，因为整体 shutdown 会接管；output handler 自身异常则调用 `OutputProcessor.propagate_error()`，让等待中的请求收到同一失败，而不是永久挂起（[`AsyncLLM._run_output_handler`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L638-L709)）。

API Server 的 Engine context manager 无论正常退出还是异常都会执行 `AsyncLLM.shutdown()`。shutdown 依次关闭 Prometheus frontend state、renderer、EngineCoreClient/后台进程，并线程安全地取消 output handler（[`AsyncLLM.shutdown`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L257-L274)）。

## 入口模式分支

`vllm serve` 的逻辑调用链相似，但启动和所有权不同：

| 模式                     | 选择条件                         | frontend/core 所有权                                                         | 请求入口                         |
| ------------------------ | -------------------------------- | ---------------------------------------------------------------------------- | -------------------------------- |
| 单 Python API Server     | 默认 `api_server_count == 1`     | 当前进程运行 FastAPI/AsyncLLM；core 通常在后台进程                           | HTTP/OpenAI routes               |
| 多 Python API Server     | `api_server_count > 1`           | 父进程先启动共享 core engines/coordinator，再启动多个 API processes          | 共享监听 socket 或配置的端口模式 |
| headless                 | `--headless` 或 API count 为 0   | 只管理 EngineCore/Worker，不创建 HTTP app                                    | 外部 frontend/handshake client   |
| DP multi-port supervisor | multi-port external LB           | supervisor 按 local DP rank 管理 server/engine                               | 多端口，由外部 LB 分流           |
| external/hybrid DP LB    | 对应 DP flags                    | client 与 coordinator 选择不同 engine/rank                                   | HTTP 仍进 Python/Rust frontend   |
| Rust frontend            | `VLLM_RUST_FRONTEND_PATH` 已解析 | Python 父进程启动 core，再把 socket fd 与 transport 地址交给单 Rust frontend | Rust HTTP/可选 gRPC              |
| Python gRPC server       | `--grpc`                         | `ServeSubcommand` 直接分派到独立 gRPC server                                 | gRPC                             |

[`ServeSubcommand.cmd`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/cli/serve.py#L50-L149) — 校验互斥的 DP LB 模式并选择 supervisor、headless、multi frontend 或 single server。

[`run_headless`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/cli/serve.py#L173-L256) — 无 API Server 时启动 local EngineCore 或非 head node Worker/Executor，并只监控后端生命期。

[`run_multi_api_server`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/cli/serve.py#L257-L395) — 先解析 Engine/Executor 和 transport 地址，在 `launch_core_engines` 上下文中启动 Python API process manager 或 Rust process manager。

这些分支的关键差异是**谁 bind socket、谁启动 EngineCore、谁选择 DP rank、谁负责 shutdown**，而不是 Scheduler 或 ModelRunner 算法发生改变。

## Rust 前端的替换边界

Rust 路径有两种启动所有权，但都复用 Python EngineCore 协议。

### Python-supervised `frontend`

`run_multi_api_server()` 先启动 Python core engines。`RustFrontendProcessManager` 将监听 socket fd 设为 inheritable，再执行：

```text
vllm-rs frontend
  --listen-fd <fd>
  --input-address <engine-input-address>
  --output-address <engine-output-address>
  --engine-start-index <n>
  --engine-count <n>
  [--coordinator-address <address>]
  --args-json <frontend-options>
```

[`RustFrontendProcessManager`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/utils.py#L325-L390) — 传递 fd、已 bootstrap transport、engine range 与环境驱动的 timeout 参数，并提供与 Python API process manager 相同的监控/关闭外观。

### Rust `serve` 管理或连接 Python engine

Rust binary 的 `frontend` 子命令直接运行 server；`serve` 子命令还可以：

- 启动并监控一个 managed Python headless engine；
- 使用 `data_parallel_size_local == 0` 连接外部 engine；
- 使用 `--headless` 只管理 Python engine 而不启动 Rust HTTP frontend。

[`vllm-rs async_main`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/cmd/src/main.rs#L85-L170) — `frontend`、`serve`、`bench serve` 的顶层所有权分支。

### Rust 请求生命周期

Rust server 在启动时建立 EngineCore client/state，构造 Axum router，并可选启动独立 tonic gRPC Generate/Control server（[`vllm_server::serve_with_router_extension`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/server/src/lib.rs#L151-L270)）。其请求侧完成 schema、chat renderer、parser 和 streaming，然后把兼容的 `EngineCoreRequest` 交给 Rust `EngineCoreClient`。

`EngineCoreClient.call()`：

1. 写入 `client_index` 并校验请求。
2. 向 coordinator 选择的 engine 注册 request-scoped receiver。
3. 发送 `EngineCoreRequestType::Add` 与 MessagePack-compatible request。
4. 返回只接收该 request ID 输出的 `EngineCoreOutputStream`。

[`rust EngineCoreClient.call`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/engine-core-client/src/client.rs#L475-L533) — Rust frontend 到 Python EngineCoreProc 的请求入口。

输出 stream 收到带 finish reason 的最终输出后终止；若 consumer 在运行中 drop stream，`Drop` 会把 request ID 发给后台 auto-abort worker（[`EngineCoreOutputStream`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/engine-core-client/src/client/stream.rs#L43-L165)）。这与 Python generator 取消时调用 `AsyncLLM.abort()` 的生命周期目标相同。

Rust client 支持自己拥有完整 handshake 的 `HandshakeOwner` 和接收 Python 已分配地址的 `Bootstrapped` transport mode（[`EngineCoreClient.connect`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/engine-core-client/src/client.rs#L211-L271)）。替换边界因此是：

```text
Python frontend: FastAPI + OnlineRenderer + AsyncLLM + Python EngineCoreClient
Rust frontend:   Axum/tonic + Rust renderer/parser + Rust EngineCoreClient
                                      ↓
                          同一个 Python EngineCoreProc
                                      ↓
                    Scheduler → Executor → Worker → ModelRunner
```

Rust frontend 在此快照仍是实验性路径。不能把它描述为 Rust Scheduler、Rust Worker 或 Rust model runtime。

## 兼容入口辨析

### 公共类名是别名，不是第二套实现

`vllm` 仍公开 `AsyncLLMEngine` 和 `LLMEngine`，但兼容模块直接转发到 V1：

- [`vllm.engine.async_llm_engine.AsyncLLMEngine`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/engine/async_llm_engine.py#L1-L7) — `AsyncLLM` 的直接 alias。
- [`vllm.engine.llm_engine.LLMEngine`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/engine/llm_engine.py#L1-L7) — `vllm.v1.engine.llm_engine.LLMEngine` 的直接 alias。
- [`vllm.v1.engine.llm_engine.LLMEngine`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/llm_engine.py#L48-L60) — 类注释明确标记为 backwards-compatible “Legacy LLMEngine”，内部仍构造 V1 processor、core client、Scheduler/Executor 链。

因此，类型关系应写成：

```text
AsyncLLMEngine ──alias──> AsyncLLM
public LLMEngine ──alias──> v1.engine.llm_engine.LLMEngine
LLM ──composition──> LLMEngine ──uses──> EngineCoreClient
```

而不是把 `AsyncLLMEngine`、`AsyncLLM`、V0 engine 和 V1 engine 画成四套同时运行的继承分支。

### 入口模块与命令名称

- 推荐服务端命令是 `vllm serve ...`；`vllm chat`/`vllm complete` 是连接已有 server 的客户端工具。
- `python -m vllm.entrypoints.openai.api_server` 仍保留独立 parser/run path，并注明需与 CLI 保持同步；它最终仍进入 `run_server()` 和当前 V1 `AsyncLLM`（[`api_server.__main__`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/api_server.py#L794-L805)）。
- 直接向 `LLMEngine.add_request()` 或 `AsyncLLM.generate()` 传 `EngineCoreRequest` 在当前实现中会警告 deprecated；面向 frontend 的稳定方向是传 renderer 输出的 `EngineInput`，由 `InputProcessor` 负责 core request 转换（[`AsyncLLM.add_request`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L330-L374)）。

兼容名称保证已有调用方能迁移，不改变当前主链路的事实：在线 API Server 实例化 `AsyncLLM`，同步离线 API 通过兼容 `LLMEngine` 使用同一 V1 core。

## 关键代码索引

本表只列本篇实际使用的入口与生命周期节点；全局稳定 ID 和跨子系统索引见[07：源码清单与代码索引](07-source-inventory-and-code-index.md)。

| 符号或目标                                   | 代码位置                                                                                                                        | 本篇用途                                    |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| `project.scripts.vllm`                       | [`pyproject.toml`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/pyproject.toml#L43-L47)                                       | shell 命令到 Python CLI 的安装入口          |
| `vllm.__getattr__`                           | [`vllm/__init__.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/__init__.py#L16-L73)                                   | 公共对象懒加载                              |
| `cli.main.main`                              | [`main.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/cli/main.py#L16-L97)                                | 子命令运行时注册与分派                      |
| `ServeSubcommand.cmd`                        | [`serve.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/cli/serve.py#L50-L149)                             | server/headless/Rust/DP 模式选择            |
| `LLM.generate`                               | [`llm.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/llm.py#L411-L470)                                    | 同步离线 generation API                     |
| `OfflineInferenceMixin._run_completion`      | [`offline_utils.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/offline_utils.py#L290-L350)                | 批量 render、入队和 step loop 入口          |
| `LLMEngine.add_request`                      | [`llm_engine.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/llm_engine.py#L218-L295)                        | 离线输入到 core request                     |
| `LLMEngine.step`                             | [`llm_engine.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/llm_engine.py#L296-L335)                        | 同步输出处理与 frontend abort               |
| `build_async_engine_client_from_engine_args` | [`api_server.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/api_server.py#L147-L193)               | 当前在线 `AsyncLLM` 构造与关闭              |
| `build_app`                                  | [`api_server.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/api_server.py#L196-L290)               | FastAPI/router/plugin 构造                  |
| `init_app_state`                             | [`api_server.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/api_server.py#L359-L489)               | renderer、models 与 serving state 注入      |
| chat completion route                        | [`api_router.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/chat_completion/api_router.py#L53-L76) | HTTP 与 serving object 的边界               |
| `OpenAIServingChat._create_chat_completion`  | [`serving.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/chat_completion/serving.py#L255-L408)     | chat render、sampling 和 stream/full 分叉   |
| `AsyncLLM.add_request`                       | [`async_llm.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L281-L417)                          | collector、OutputProcessor 与 core 同时注册 |
| `AsyncLLM.generate`                          | [`async_llm.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L525-L637)                          | 每请求异步输出 generator 与异常清理         |
| `AsyncLLM._run_output_handler`               | [`async_llm.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L638-L709)                          | 批量 core output 到每请求 collector         |
| `EngineCoreClient.make_client`               | [`core_client.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L78-L141)                       | in-process、MP、DP client 动态选择          |
| `EngineCore.step`                            | [`core.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L581-L611)                                    | schedule、execute、sample、update           |
| `Scheduler.update_from_output`               | [`scheduler.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/core/sched/scheduler.py#L1580-L1674)                    | ModelRunner 结果到请求状态和 core output    |
| `RustFrontendProcessManager`                 | [`v1/utils.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/utils.py#L325-L390)                                      | Python-supervised Rust frontend 启动        |
| Rust `EngineCoreClient.call`                 | [`client.rs`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/engine-core-client/src/client.rs#L475-L533)               | Rust request 到 Python core                 |
| Rust `EngineCoreOutputStream`                | [`stream.rs`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/engine-core-client/src/client/stream.rs#L43-L165)         | request-scoped output 与 drop auto-abort    |

---

上一页：[01：项目地图](01-project-map.md) · 下一页：[03：核心类关系](03-core-class-hierarchy.md) · 返回：[指南入口](README.md)
