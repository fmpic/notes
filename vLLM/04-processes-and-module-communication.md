# 04：进程、控制面、数据面与通信协议

> **代码快照**：`0934b267906f8cd9459f287b31647c3ed5c58e01`。
> 行号、类型关系和调用链均以此提交为准；返回[指南入口](README.md)。

![vLLM 进程与通信图](diagrams/process-communication.png)

本篇把[请求生命周期](02-entrypoints-and-request-lifecycle.md)中的逻辑调用展开成真实部署边界。核心原则是：先区分控制面和数据面，再判断同一逻辑对象位于当前进程、子进程、Ray actor 还是远端节点。类关系见[03：核心类关系](03-core-class-hierarchy.md)。

## 控制面与数据面先分开

vLLM 同时存在多种通信机制。它们解决的问题不同，不能统称为“RPC”或“NCCL 通信”。

| 平面             | 传输内容                                                              | 主要参与者                                     | 当前主要机制                                               |
| ---------------- | --------------------------------------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| 北向协议         | HTTP/gRPC request、SSE/JSON response                                  | client ↔ Python/Rust frontend                  | FastAPI/Uvicorn、Axum/Hyper、tonic                         |
| Engine 控制面    | add/abort/utility、request output、ready/health、DP wave/stats        | frontend `EngineCoreClient` ↔ `EngineCoreProc` | ZMQ、MessagePack/msgspec、可选 tensor IPC                  |
| Executor 控制面  | method name、`SchedulerOutput`、grammar、返回状态/`ModelRunnerOutput` | `EngineCore`/Executor ↔ Worker process         | vLLM `MessageQueue`、Ray RPC 或直接调用                    |
| 模型数据面       | activation、logits、KV/hidden tensors、collective payload             | Worker ranks ↔ Worker ranks                    | torch.distributed、NCCL/Gloo/XCCL 及 platform communicator |
| Connector 数据面 | KV cache、encoder cache、disaggregated prefill/decode 数据            | connector peers/storage                        | KV/EC connector 自身 transport                             |
| 构建/插件边界    | 不是每 token 的传输；决定进程里加载什么实现                           | process-local registry/plugin loader           | Python entry points、限定名 import                         |

本文把“控制面”定义为请求、调度、RPC、状态、握手、健康和关闭消息；把“数据面”定义为模型张量与设备 collective。`SchedulerOutput` 虽然可能带 tensor/grammar buffer，语义上仍是 Executor 控制消息；TP all-reduce 和 PP intermediate tensors 才是模型数据面。

源码也明确要求 Executor `collective_rpc()` 主要传控制消息，并建议另设数据面通信（[`Executor.collective_rpc` contract](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/abstract.py#L129-L184)）。

## 最小进程内拓扑

当同步 `LLMEngine` 不启用 V1 multiprocessing 时，`EngineCoreClient.make_client()` 返回 `InprocClient`。此时没有 frontend ↔ core ZMQ：

```text
单个 Python 进程
┌──────────────────────────────────────────────────────────────┐
│ LLM → LLMEngine → InprocClient ◆→ EngineCore               │
│                                  ├─ Scheduler                │
│                                  └─ Executor                 │
│                                      └─ Worker/ModelRunner   │
└──────────────────────────────────────────────────────────────┘
```

[`InprocClient`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L291-L340) 直接构造 `EngineCore`：

- `add_request()` 直接调用 core preprocessing 与 add。
- `get_output()` 直接调用 `EngineCore.step_fn()` 和 `post_step()`。
- abort、cache reset、profile、sleep/wake 都是普通方法调用。

Executor 是否再创建 Worker 子进程是另一维度：

- `UniProcExecutor` 的 `WorkerWrapperBase`、Worker 和 ModelRunner 仍在同一进程。
- `MultiprocExecutor` 即使 core 是 in-process，也可创建 Worker processes。
- `external_launcher`、Ray 或多节点配置还能把 Worker/Engine 的所有权交给外部 launcher。

所以，“InprocClient”只说明 frontend Engine 与 `EngineCore` 同进程，不等于整个模型一定单进程或单 GPU。

## 默认多进程拓扑

在线 `AsyncLLM` 当前不支持 async in-process core，因此至少有 frontend process 与 `EngineCoreProc` 两个进程。常见拓扑可按 Executor backend 再分两类。

### 单设备/`uni` Executor

```text
Process A: API Server
  FastAPI → serving → AsyncLLM → AsyncMPClient
                              │ ZMQ control/output
                              ▼
Process B: EngineCoreProc
  EngineCore → Scheduler → UniProcExecutor
                           → WorkerWrapper → Worker → ModelRunner → GPU
```

这里 Worker 是对象，不是额外 OS process。`UniProcExecutor.collective_rpc()` 在 Process B 里直接 `run_method()`。

### `mp` Executor

```text
Process A: API Server
  AsyncLLM + AsyncMPClient
          │ ZMQ
          ▼
Process B: EngineCoreProc
  Scheduler + MultiprocExecutor
          │ MessageQueue broadcast / response queues
          ├──────────────┬──────────────┐
          ▼              ▼              ▼
Process C: Worker 0  Process D: Worker 1  ... Worker N
  ModelRunner         ModelRunner
       ╲                 ╱
        torch.distributed model data plane
```

`CoreEngineProcManager` 按 local DP rank 创建名为 `EngineCore` 或 `EngineCore_DP<n>` 的后台进程，并监控 sentinel/liveness（[`CoreEngineProcManager`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/utils.py#L120-L243)）。`MultiprocExecutor` 再为 TP/PP/PCP world 启动或连接 Worker processes。

必须区分三个“rank”：

- `client_index`：多个 API process 中，输出应返回哪个 frontend。
- `engine_index`/DP rank：请求被哪个 `EngineCoreProc` 调度。
- Worker/global rank：模型 parallel group 中的 rank。

它们可能都从 0 开始，但用于不同 routing key，不能在图中合并。

### 多 API Server 与 DP

扩展后拓扑是 N 个 API processes、一个或多个 DP EngineCore，以及每个 core 所属的 Executor/Worker 集合。父 supervisor 只负责 socket、进程所有权、handshake 和关闭，不参与每轮 Scheduler step。`launch_core_engines()` 决定是否创建 DP Coordinator、local processes 或 Ray actors，以及是否为 multimodal tensor IPC 创建共享 queue（[`launch_core_engines`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/utils.py#L1068-L1208)）。

## Frontend ↔ EngineCore

### socket 方向和所有权

Python `MPClient` 在 frontend 一侧 bind 两类 socket：

```text
Frontend / MPClient                         EngineCoreProc

ROUTER input socket (bind)  ─────────────▶  DEALER input socket(s) (connect)
  [engine identity]                           [request type][msgpack][aux...]
  [request type][msgpack][aux...]

PULL output socket (bind)   ◀─────────────  PUSH output socket(s) (connect)
  [msgpack EngineCoreOutputs][aux...]         按 client_index 选择目标 socket
```

[`MPClient.__init__`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L482-L658) — frontend bind `ROUTER`/`PULL`，解析 IPC/TCP 实际 endpoint，创建 encoder/decoder，等待每个 engine identity 的 ready response。

[`EngineCoreProc.process_input_sockets`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L1599-L1710) — core IO thread 以固定两字节 engine identity connect `DEALER`，poll frontend 与 coordinator 输入，解码后放入 `input_queue`。

[`EngineCoreProc.process_output_sockets`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L1712-L1772) — output IO thread connect `PUSH`，按 `client_index` 选择 frontend socket，并用 zero-copy multipart 发送 `EngineCoreOutputs`。

为什么输入使用 ROUTER/DEALER，而输出使用 PUSH/PULL：

- frontend 需要通过 engine identity 把请求定向到某个 DP core，因此输入需要 routing frame。
- core 已从 request 保存 `client_index`，输出只需选择对应的单向 PUSH socket。
- utility call 与 request output 共用 output channel，但通过 `utility_output.call_id` 分流。

### 请求 frame 与消息类型

Python frontend 发送的 multipart 逻辑布局是：

```text
ROUTER side: [engine_identity][request_type_byte][msgpack_root][aux_buffer_0]...
DEALER side:                  [request_type_byte][msgpack_root][aux_buffer_0]...
```

[`MPClient._send_input`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L876-L888) — 在 request type 前加入目标 engine identity；有 auxiliary buffers 时保留对象引用直到 ZMQ tracker 完成。

`EngineCoreRequestType` 使用单字节 frame，无需再对类型编码：

|     值 | 类型              | payload/作用域                                |
| -----: | ----------------- | --------------------------------------------- |
| `0x00` | `ADD`             | `EngineCoreRequest`                           |
| `0x01` | `ABORT`           | request ID 列表                               |
| `0x02` | `START_DP_WAVE`   | coordinator 的 `(wave, exclude_engine_index)` |
| `0x03` | `UTILITY`         | `(client_index, call_id, method_name, args)`  |
| `0x04` | `EXECUTOR_FAILED` | core 内部 sentinel，不是 Rust 公共 wire type  |
| `0x05` | `WAKEUP`          | core shutdown 时唤醒本地 queue 的 sentinel    |

[`EngineCoreRequestType`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/__init__.py#L254-L269) — Python wire/control type 定义。Rust compatibility enum只公开前四个跨语言值（[`Rust EngineCoreRequestType`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/engine-core-client/src/protocol/request.rs#L13-L57)）。

`EngineCoreProc._handle_client_request()` 再把这些类型分派到 add、abort、utility 或 executor-failure 处理。abort 会同时进入 eager `aborts_queue` 和有序 `input_queue`，既能尽快取消正在执行的 request，又避免请求/取消重排造成泄漏（[`EngineCoreProc request dispatch`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L1415-L1537)、[`input IO abort handling`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L1671-L1710)）。

### 请求与输出 schema

`EngineCoreRequest` 是 array-like `msgspec.Struct`，包含 request ID、token/multimodal 输入、sampling/pooling 参数、LoRA、cache salt、DP rank、prompt embeds、`client_index`、wave、priority、trace 与 resumable/reasoning fields（[`EngineCoreRequest`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/__init__.py#L90-L164)）。

反向输出分两层：

- `EngineCoreOutput`：单 request 的新 token/logprobs/pooling/finish/stop/event/KV/EC/routed-expert 信息。
- `EngineCoreOutputs`：一个 core step 的 request outputs、scheduler stats、timestamp、utility result、finished set 与 DP wave signal。

[`EngineCoreOutput` 与 `EngineCoreOutputs`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/__init__.py#L168-L252) — 都以 array-like、omit-default 的 `msgspec.Struct` 定义。字段顺序因此是 Python/Rust compatibility 的一部分。

### MessagePack、auxiliary frames 与 tensor IPC

frontend/core wire format不是 JSON：

1. `MsgpackEncoder` 用 msgspec MessagePack 编码 root object。
2. 较大的 `torch.Tensor`/NumPy array 可以拆成 multipart auxiliary buffers，减少复制。
3. 启用 multimodal `mm_tensor_ipc="torch_shm"` 且存在 tensor queue 时，`TensorIpcSender` 将 tensor out-of-band 交给 `TensorIpcReceiver`；MsgPack 中只保留 placeholder。
4. utility result 默认必须是安全可编码类型。只有显式开启 `VLLM_ALLOW_INSECURE_SERIALIZATION=1` 才允许 pickle/cloudpickle fallback。

[`MsgpackEncoder`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/serial_utils.py#L136-L232) — MessagePack root、tensor/ndarray auxiliary buffer 和 insecure fallback gate。

[`MsgpackDecoder`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/serial_utils.py#L313-L443) — root/auxiliary buffers 解码、OOB tensor provider 与安全 utility result 恢复；它和 encoder 是同一 wire contract 的两端，不是同一个类。

[`TensorIpcSender`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/tensor_ipc.py#L45-L108) 与 [`TensorIpcReceiver`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/tensor_ipc.py#L114-L168) — multimodal tensor 的 torch multiprocessing queue 路径；当前注释明确 DP > 1 不支持这条共享 tensor 流。

### 启动 handshake 与 ready response

启动 handshake 使用独立 ROUTER/DEALER socket，状态机是：

```text
EngineCoreProc ── HELLO(local, headless, identity) ──▶ supervisor/frontend
EngineCoreProc ◀─ INIT(addresses, selected DP config) ── supervisor/frontend
EngineCoreProc ── READY(config hash) ───────────────▶ supervisor/frontend

随后每个 core 在正式 input DEALER 上发送 EngineCoreReadyResponse
frontend 收齐 identity 后才开始正常请求
```

[`EngineCoreProc.startup_handshake`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L1230-L1266) — HELLO/INIT/READY 的 engine 侧实现。

[`wait_for_engine_startup`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/utils.py#L1222-L1362) — supervisor 验证 identity、local/headless 角色、DP config hash 和 process sentinel。

`EngineCoreReadyResponse` 还把 KV profiling 后的 `max_model_len`、GPU blocks、block size、dtype、world/DP size 与 coordinator stats address 回传 frontend；client 将这些 post-init 值同步回本地 config（[`EngineCoreReadyResponse`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/__init__.py#L69-L87)、[`MPClient._apply_ready_response`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L729-L764)）。

## EngineCore ↔ Worker 控制面

### Executor 是通信策略边界

`EngineCore` 只调用抽象 Executor。不同 backend 的控制面如下：

| Executor                       | Core 到 Worker 的机制                                      | Worker 是否独立进程 |
| ------------------------------ | ---------------------------------------------------------- | ------------------- |
| `UniProcExecutor`              | 直接 `run_method(driver_worker, ...)`                      | 否                  |
| `MultiprocExecutor`            | broadcast `MessageQueue` + 每 rank response queue          | 是                  |
| `RayDistributedExecutor`       | Ray actor method/DAG                                       | 是，Ray actors      |
| `RayExecutorV2`                | 复用 `MultiprocExecutor` MQ 语义，Worker 由 Ray actor 承载 | 是                  |
| `ExecutorWithExternalLauncher` | 每 Engine 一个本地 Worker；外部 launcher 同步多个 Engine   | 由外部拓扑决定      |

### Multiproc request/reply

`MultiprocExecutor.collective_rpc()` 把 `(method, args, kwargs, output_rank)` 放入 broadcast queue。method 可以是字符串，也可以是 cloudpickle 后的 callable；若只需唯一结果（例如 driver/output rank 的 `ModelRunnerOutput`），只等待对应 response queue（[`MultiprocExecutor.collective_rpc`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/multiproc_executor.py#L320-L424)）。

每个 Worker process 的 busy loop：

1. 从 `rpc_broadcast_mq.dequeue()` 取同一 control tuple。
2. 字符串用 `getattr(self.worker, method)`；bytes 则 cloudpickle load callable。
3. 执行 Worker 方法。
4. 只有 `output_rank is None` 或当前 rank 被指定时才写 response queue。
5. exception 转成 FAILURE status/string，让 Executor 抛错并触发 shutdown。

[`WorkerProc.worker_busy_loop`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/multiproc_executor.py#L997-L1027) — broadcast control 到 rank-local Worker 调用的接收端。

### `MessageQueue` 不是 frontend ZMQ 协议

vLLM `MessageQueue` 对 local readers 使用共享内存 ring buffer，小消息直接写 ring；溢出/大消息通过 IPC XPUB/SUB multipart。remote readers 使用 TCP XPUB/SUB。它以 pickle protocol 5 和 out-of-band `PickleBuffer` 编码，并通过 reader flags、memory fence 与 spin condition 控制 ring slot 生命周期（[`MessageQueue`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/distributed/device_communicators/shm_broadcast.py#L370-L504)、[`MessageQueue.enqueue/dequeue`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/distributed/device_communicators/shm_broadcast.py#L729-L808)）。

因此：

- frontend ↔ core 是跨 Python/Rust 的显式 MessagePack schema。
- Executor ↔ Worker `MessageQueue` 是受控进程拓扑内的 Python object transport。
- 两者都可能使用 ZMQ socket 作为一部分，但 wire compatibility、安全边界和 routing semantics 完全不同。

不要把 `MessageQueue` 名称误解为某个通用外部 message broker。

## Worker 间数据面

Worker 启动时先初始化 torch distributed world，再创建 model-parallel groups。`init_distributed_environment()` 根据 backend、rank、DP/multi-node 配置调用 `torch.distributed.init_process_group()`，并在 backend 不可用时受控 fallback 到 Gloo（[`init_distributed_environment`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/distributed/parallel_state.py#L1582-L1737)）。

`initialize_model_parallel()` 按 `ExternalDP × DP × PP × PCP × TP` 维度重排 rank，建立 TP、decode-context parallel、prefill-context parallel、PP、DP、EP 和可选 EPLB groups（[`initialize_model_parallel`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/distributed/parallel_state.py#L1740-L1979)）。

| 维度    | 典型数据面通信                                          | 目的                                            | 与控制面的区别                       |
| ------- | ------------------------------------------------------- | ----------------------------------------------- | ------------------------------------ |
| TP      | all-reduce、all-gather、reduce-scatter                  | 在一层内切分 linear/attention 权重与 activation | 不负责分配 HTTP request              |
| PP      | point-to-point send/recv intermediate tensor dict       | 在 pipeline stage 间传 hidden/residual          | `SchedulerOutput` 仍由 Executor 广播 |
| DP      | group all-reduce/状态同步；不同 EngineCore 可独立接请求 | 多副本或 MoE 同步                               | DPCoordinator 只做负载/wave 控制     |
| EP      | token/expert dispatch、all-to-all 与 combine            | 将 MoE tokens 路由到 expert ranks               | 与 API LB 无关                       |
| PCP/DCP | context shard 的 gather/all-to-all 等                   | 切分 prefill/decode context 工作                | 不改变 frontend schema               |

TP helper 直接通过 `get_tp_group()` 调用 group collective（[`tensor_model_parallel_all_reduce`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/distributed/communication_op.py#L12-L37)）。`GroupCoordinator` 封装绑定特定 backend 的 PyTorch ProcessGroup 和 device communicator（[`GroupCoordinator`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/distributed/parallel_state.py#L380-L455)）。

PP 的数据路径在 GPU Worker 中可见：非 first rank 先 `irecv_tensor_dict()`，ModelRunner forward 若返回 `IntermediateTensors`，非 last rank 再 `isend_tensor_dict()`，并保存 non-blocking handles 到下一轮等待（[`Worker.execute_model`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu_worker.py#L1018-L1107)、[`GroupCoordinator.isend/irecv_tensor_dict`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/distributed/parallel_state.py#L1017-L1168)）。

一个关键分界：

```text
EngineCore → Worker：本轮做什么（control）
Worker rank ↔ Worker rank：完成这一轮所需的 tensor（data）
Worker → EngineCore：可观察执行结果（control response）
```

NCCL/torch.distributed 出现故障时，表象可能是 Executor timeout；但根因通常在 Worker data plane。反过来，frontend ZMQ 断开不会自动说明 NCCL group 已损坏。

## DP Coordinator 与负载均衡

`DPCoordinator` 是独立控制进程，仅在在线 DP 且配置需要时由 rank 0 frontend/supervisor 创建。它不是模型 tensor data-plane coordinator（[`DPCoordinator`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/coordinator.py#L23-L137)）。

### 三条 coordinator channel

```text
EngineCoreProc(s) ── PUSH scheduler stats/wave ──▶ PULL output_back
EngineCoreProc(s) ◀─ XSUB / START_DP_WAVE ─────── XPUB publish_back
Frontend client(s) ◀─ XSUB load snapshots ─────── XPUB publish_front
                           ▲
                           └─ first request / scale notification from frontend
```

`DPCoordinatorProc`：

- 收集每个 engine 的 `[waiting, running]` counts。
- 按最小间隔或 heartbeat 发布 `(counts, current_wave, engines_running)`。
- 维护 MoE DP global running/paused wave。
- 某 engine 在 paused 状态收到新请求时，向其他 engines 广播 `START_DP_WAVE`。
- 处理 elastic EP engine count 变化。

[`DPCoordinatorProc.process_input_socket`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/coordinator.py#L145-L453) — XPUB/PULL/XPUB socket、stats snapshot 与 wave 状态机。

### client variants 与 routing

- `DPAsyncMPClient` 默认用于 external LB：每个 frontend client 管理指定 DP rank，但仍订阅 coordinator wave/running 状态。
- `DPLBAsyncMPClient` 用于 internal/hybrid LB：维护每个 engine 的 waiting/running counts，选择目标 engine，并记录 request-to-engine 以便 abort 正确路由。
- `data_parallel_rank` 可显式指定 engine；否则 internal LB 根据可见负载选择。

[`DPAsyncMPClient` stats/wave loop](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L1250-L1427) 与 [`DPLBAsyncMPClient`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L1430-L1515) — DP client 选择和 coordinator feedback 的落点。

DP Coordinator 和 torch distributed DP group 的职责不同：前者分发控制状态和 API-side load signal；后者参与 ranks 之间的同步/collective。图中应分别标成控制面和数据面。

## Python 多 API Server 与 Rust 前端

### 多 Python API Server

`run_multi_api_server()` 的父进程顺序是：

1. 先创建监听 socket 与 `VllmConfig`，解析 Executor。
2. 生成每个 API child 的 Engine ZMQ input/output 地址。
3. 在 `launch_core_engines()` context 中创建 coordinator 与 EngineCore processes/actors。
4. 创建 `APIServerProcessManager`，每个 child 获得 `client_index`、地址、stats address 和可选 tensor queue。
5. 普通 Python/非 Ray DP 情况下，child bind `tcp://host:0` 后通过 one-way pipe 回报真实 endpoint；父进程再把地址放入 engine handshake。
6. 监控 frontend、core 和 coordinator；任一异常退出触发整体清理。

[`run_multi_api_server`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/cli/serve.py#L257-L394) — 父 supervisor 的完整所有权与 shutdown 顺序。

[`APIServerProcessManager`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/utils.py#L166-L325) — spawn API workers、传递共享监听 socket 和 per-client Engine 地址，并收集 child 实际 bound endpoint。延迟 child bind 避免普通路径的 port allocation TOCTOU；Rust frontend 和 Ray DP 因无法回报重绑定地址而在父进程预分配端口。

### Python-supervised Rust frontend

Rust frontend 替换 API process，但不替换 Python EngineCore：

```text
Python supervisor
├── EngineCoreProc(s) → Executor/Worker(s)
└── vllm-rs frontend subprocess
      ├── inherited HTTP listen fd
      ├── Rust renderer/parser/router
      └── Rust EngineCoreClient
            ├── ROUTER input bind
            └── PULL output bind
```

`RustFrontendProcessManager` 把监听 fd、input/output address、engine rank range、coordinator address 和 JSON args 交给 `vllm-rs frontend`（[`RustFrontendProcessManager`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/utils.py#L325-L390)）。

Rust client 保持相同协议：`EngineCoreClient.connect()` 在 `HandshakeOwner` 和 Python 已 bootstrap 的 `Bootstrapped` transport 间选择，并启动 output loop、request dispatcher、abort worker 与可选 coordinator tasks（[`Rust EngineCoreClient.connect/from_connected`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/engine-core-client/src/client.rs#L211-L355)）。

Rust transport 同样提供 shared `RouterSendHalf` 与 `PullSocket`，engine identity 为 Python 兼容的两字节 little-endian DP index（[`Rust ConnectedTransport/EngineId`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/engine-core-client/src/transport.rs#L29-L137)）。`connect_bootstrapped()` 绑定 Python 给出的地址并等待连续 engine identities 注册（[`connect_bootstrapped`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/engine-core-client/src/transport.rs#L321-L383)）。

Rust `EngineCoreClient.call()` 发送 Add 并返回 request-scoped output stream；drop 未结束 stream 会进入 auto-abort worker（[`Rust client call/abort`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/engine-core-client/src/client.rs#L475-L555)）。Rust 侧使用 serde tuple 与 MessagePack 保持 Python array-like struct 字段顺序，不把 payload 改成 JSON。

## 插件与跨模块边界

插件在不同进程加载，作用域本身就是通信边界。

| plugin group                | 加载位置                                         | 能改变什么                                                   | 不能假设什么                                                 |
| --------------------------- | ------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `vllm.general_plugins`      | process 0、EngineCore、Worker 等相关进程各自一次 | 注册 model/loader/quantization、注入 engine/worker-side 行为 | 在 frontend 加载不代表 Worker process 已共享 Python 内存状态 |
| `vllm.endpoint_plugins`     | API frontend only                                | 增加/覆盖 HTTP routes，初始化 app state                      | 不会自动安装 engine-side method                              |
| `vllm.io_processor_plugins` | process 0/input side                             | 选择 renderer/IO preprocessing                               | 不直接出现在 Scheduler/Worker                                |
| `vllm.platform_plugins`     | 各进程首次解析 `current_platform`                | 选择 platform class、Worker/backend/kernel import            | 各进程都要独立解析                                           |
| `vllm.stat_logger_plugins`  | async serving 的 process 0                       | 消费 stats 并输出自定义 metrics/log                          | 不参与模型 tensor collective                                 |

[`plugin groups 与 loader`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/plugins/__init__.py#L15-L90) — entry-point group、`VLLM_PLUGINS` allowlist 和 process-local “once” gate。注释明确 general plugin 可能在多个进程分别加载，plugin 必须可重复执行。

Endpoint plugin 是更严格的 opt-in：未设置 `VLLM_PLUGINS` 时即使发现 plugin 也不加载；还会按 `required_tasks` 过滤（[`load_endpoint_plugins`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/plugins/__init__.py#L93-L155)）。

若 endpoint 需要 Worker/Engine 行为，必须把 engine-side registration 放在独立 `general_plugins` entry point，并通过既有 `EngineClient`（例如 `collective_rpc`）访问；endpoint 与 general 两个 entry point 独立加载，彼此不蕴含（[`EndpointPlugin` contract](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/plugins/endpoint_plugins/interface.py#L1-L81)）。这条规则防止 plugin 绕过既有进程/协议边界新开不可监控通道。

Platform plugin 则与所有进程的设备选择相关：builtin 与 installed plugin 一起检测，要求最多一个 platform 被激活，最后才按限定名导入 class（[`resolve_current_platform_cls_qualname`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/platforms/__init__.py#L211-L252)）。

### 如何设计跨边界扩展

- 只改 HTTP surface：endpoint plugin，状态留在 API process。
- 需要 Worker method：general plugin/worker extension 在 Worker process 注册，再走 `EngineClient → EngineCore utility → Executor.collective_rpc`。
- 需要新模型：general plugin 修改 process-local ModelRegistry；每个需要 inspect/load 的进程都必须加载。
- 需要新设备：platform plugin 返回 platform class qualified name，由 platform 再选择 Worker/backend。
- 需要传大 tensor：不要把它塞进 endpoint plugin 自建 socket；使用现有 distributed/connector/tensor IPC 机制并明确所有权。

## 启动、健康检查、错误传播与关闭

### 启动 barrier

一个可接收请求的多进程实例至少通过三层 ready gate：

1. **EngineCore process handshake**：HELLO/INIT/READY，检查 DP identity、local/headless 与 config hash。
2. **正式 Engine channel ready**：每个 core 的 input DEALER 向 frontend ROUTER 发送 `EngineCoreReadyResponse`；client 收齐所有 identity。
3. **Worker ready**：每个 Worker 完成 WorkerWrapper、device、distributed、model load 与 MQ subscription 后，通过 pipe 返回 response queue handle；Executor 才进入 collective RPC。

DP Coordinator 还会等待全部 engine XSUB subscription，再广播 `READY`。任何 barrier 提前放行都可能造成 ROUTER 无法向尚未注册的 identity 发送，或 MQ PUB 丢失第一条消息。

### 分层健康监控

- `CoreEngineProcManager.monitor_engine_liveness()` 监控 EngineCore process sentinel。
- `MPClient.start_engine_core_monitor()` 发现 core 意外退出后设置 `engine_dead`、关闭 client，后续操作抛 `EngineDeadError`（[`MPClient core monitor`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L704-L728)）。
- `MultiprocExecutor` 监控 Worker sentinels；任一 Worker 死亡会 shutdown Executor 并调用 failure callback，将 `EXECUTOR_FAILED` 注入 EngineCore（[`MultiprocExecutor worker monitor`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/multiproc_executor.py#L280-L318)）。
- 多 frontend supervisor 同时监控 API/Rust process、coordinator 和 Engine manager（[`wait_for_completion_or_failure`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/utils.py#L517-L588)）。
- output socket 发送专用单 frame `ENGINE_CORE_DEAD`；Python/Rust client 都将其视为永久 health failure。

错误传播方向通常是：

```text
Worker exception/death
  → response FAILURE 或 Executor failure callback
  → EngineCore failure / ENGINE_CORE_DEAD
  → EngineCoreClient marks dead
  → AsyncLLM output handler propagates error
  → waiting request generators / HTTP handlers fail
  → supervisor shuts down sibling processes
```

请求级 validation/preprocessing error 则形成 request-scoped `EngineCoreOutput`，不必杀死整个 engine。分清 request error、utility error、Worker RPC failure 与 process death，能避免把可恢复错误误判成 cluster failure。

### 关闭顺序

`run_multi_api_server()` 的 finally 先关闭 frontend manager，再按同一个 deadline 关闭 local Engine manager 与 coordinator。EngineCore 内部根据 `shutdown_timeout` 选择：

- timeout 为 0：立即将所有未完成请求标为 aborted，发送最终输出。
- timeout 大于 0：拒绝新 add/utility，继续 drain 现有工作直至完成或外部超时。

[`EngineCoreProc shutdown state`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L1433-L1517) — running/requested/shutting-down 状态和 add/utility rejection。

Worker shutdown 先唤醒/关闭 MQ，等待 graceful exit，超时后再 terminate/kill；parent-death pipe 也可让 Worker 自行结束。通用 process `shutdown()` 给出 grace period、terminate、join 与后续 kill 逻辑（[`process shutdown`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/utils.py#L591-L633)）。

关闭顺序体现资源所有权：frontend 不应在 core 尚需回传 final abort output 时过早销毁 channel；core 不应在 Worker collective 未退出时先销毁 process group；supervisor 最终负责清理孤儿进程和 coordinator。

## 关键代码索引

本表只列本篇通信节点；全局稳定 ID 见[07：源码清单与代码索引](07-source-inventory-and-code-index.md)。

| 符号或目标                               | 代码位置                                                                                                                              | 本篇用途                                     |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| `InprocClient`                           | [`core_client.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L291-L340)                            | 无 frontend/core IPC 的直接调用              |
| `CoreEngineProcManager`                  | [`engine/utils.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/utils.py#L120-L243)                                 | EngineCore 子进程创建与 liveness             |
| `get_engine_zmq_addresses`               | [`engine/utils.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/utils.py#L1018-L1065)                               | IPC/TCP 地址与 deferred port allocation      |
| `launch_core_engines`                    | [`engine/utils.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/utils.py#L1068-L1208)                               | core、coordinator、Ray 与 tensor queue 汇聚  |
| `MPClient.__init__`                      | [`core_client.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L482-L658)                            | frontend ROUTER/PULL、ready 与 serialization |
| `MPClient._send_input`                   | [`core_client.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L876-L888)                            | identity/type/msgpack/aux multipart 发送     |
| `EngineCoreProc.process_input_sockets`   | [`core.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L1599-L1710)                                        | DEALER/XSUB decode 到 core queue             |
| `EngineCoreProc.process_output_sockets`  | [`core.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L1712-L1772)                                        | client-indexed PUSH outputs                  |
| `EngineCoreRequestType`                  | [`engine/__init__.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/__init__.py#L254-L269)                           | 单字节 control message types                 |
| `EngineCoreRequest`                      | [`engine/__init__.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/__init__.py#L90-L164)                            | frontend → core array-like schema            |
| `EngineCoreOutputs`                      | [`engine/__init__.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/__init__.py#L168-L252)                           | core → frontend request/utility/stats schema |
| `MsgpackEncoder`                         | [`serial_utils.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/serial_utils.py#L136-L232)                                 | MessagePack 编码与 tensor auxiliary buffers  |
| `MsgpackDecoder`                         | [`serial_utils.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/serial_utils.py#L313-L443)                                 | root/auxiliary/OOB tensor 解码               |
| `wait_for_engine_startup`                | [`engine/utils.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/utils.py#L1222-L1362)                               | HELLO/INIT/READY 角色与 config 验证          |
| `MultiprocExecutor.collective_rpc`       | [`multiproc_executor.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/multiproc_executor.py#L320-L424)            | core → Worker broadcast 与 response routing  |
| `WorkerProc.worker_busy_loop`            | [`multiproc_executor.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/multiproc_executor.py#L997-L1027)           | Worker control receive/execute/reply         |
| `MessageQueue`                           | [`shm_broadcast.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/distributed/device_communicators/shm_broadcast.py#L370-L504) | shared-memory/TCP control transport          |
| `init_distributed_environment`           | [`parallel_state.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/distributed/parallel_state.py#L1582-L1737)                  | torch distributed world 初始化               |
| `initialize_model_parallel`              | [`parallel_state.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/distributed/parallel_state.py#L1740-L1979)                  | TP/PP/DP/EP/PCP/DCP groups                   |
| `Worker.execute_model`                   | [`gpu_worker.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu_worker.py#L1018-L1107)                            | PP intermediate tensor 数据面                |
| `DPCoordinator`                          | [`coordinator.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/coordinator.py#L23-L137)                             | DP load stats 与 wave control process        |
| `DPCoordinatorProc.process_input_socket` | [`coordinator.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/coordinator.py#L176-L453)                            | 三类 coordinator channels 与状态机           |
| `APIServerProcessManager`                | [`v1/utils.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/utils.py#L166-L325)                                            | 多 API process 与地址回报 pipe               |
| `RustFrontendProcessManager`             | [`v1/utils.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/utils.py#L325-L390)                                            | inherited fd 与 bootstrapped Rust transport  |
| Rust `EngineCoreClient`                  | [`client.rs`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/engine-core-client/src/client.rs#L211-L355)                     | 跨语言 client tasks/coordinator              |
| Rust transport                           | [`transport.rs`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/engine-core-client/src/transport.rs#L29-L137)                | Python-compatible identity/ROUTER/PULL       |
| plugin groups                            | [`plugins/__init__.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/plugins/__init__.py#L15-L90)                              | process-local plugin 边界                    |
| `EndpointPlugin`                         | [`interface.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/plugins/endpoint_plugins/interface.py#L1-L81)                    | HTTP 与 engine-side extension 分工           |
| `wait_for_completion_or_failure`         | [`v1/utils.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/utils.py#L517-L588)                                            | supervisor failure fan-in                    |

---

上一页：[03：核心类关系](03-core-class-hierarchy.md) · 下一页：[05：构建与原生扩展](05-build-and-native-extensions.md) · 返回：[指南入口](README.md)
