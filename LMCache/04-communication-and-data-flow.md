# 模块通信与 KV 数据流

> 分析基准：`389b9cfcb5cec502dbba6b5d13725bf2e024610d`。代码行号与结论均针对该修订；后续提交可能使行号漂移。

> 配套图：[系统全景](diagrams/01-system-landscape.png) · [MP 消息流](diagrams/03-multiprocess-message-flow.png) · [存储流水线](diagrams/04-storage-pipeline.png) · [可编辑源文件](diagrams/README.md)

## 五个平面

| 平面            | 内容                                                | 不应承担的职责     |
| --------------- | --------------------------------------------------- | ------------------ |
| control         | key、block ids、request/task id、配置、发现、状态。 | 大块 KV bytes。    |
| data            | KV tensor bytes 或其共享内存映射。                  | fleet membership。 |
| synchronization | event、eventfd、Future、锁、prepare/commit。        | 业务配置。         |
| observability   | event、metric、log、span。                          | 决定缓存正确性。   |
| deployment      | CRD、workload、Service、ConfigMap。                 | 直接保存 KV。      |

## 一次 MP 请求的主链

### 1. Lookup + Prefetch

```text
vLLM scheduler
  -> LMCacheMPSchedulerAdapter.maybe_submit_lookup_request
  -> ZMQ LOOKUP(key, tp_size)
  -> LookupModule.lookup
  -> hash tokens -> ObjectKeys
  -> StorageManager.submit_prefetch_task
       -> L1 reserve_read
       -> PrefetchController lookup_and_lock L2
       -> reserve temporary L1 buffers
       -> L2 load into L1
       -> finish_write_and_reserve_read
  -> QUERY/WAIT_PREFETCH_STATUS
  -> token hit count back to scheduler
```

索引：[`maybe_submit_lookup_request`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/integration/vllm/vllm_multi_process_adapter.py#L700) — `lmcache/integration/vllm/vllm_multi_process_adapter.py:700`、[`LookupModule.lookup`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/modules/lookup.py#L167) — `lmcache/v1/multiprocess/modules/lookup.py:167`、[`submit_prefetch_task`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/distributed/storage_manager.py#L395) — `lmcache/v1/distributed/storage_manager.py:395`。

默认 `TrimPolicy.PREFIX` 只保留连续前缀；`SPARSE`/`SEGMENTED_PREFIX` 可保留 gap-tolerant Bitmap。L2 查找到的对象在 load 前保持 L2 lock，load 后转成 L1 read lock。

### 2. Retrieve

```text
vLLM worker records interprocess event
  -> RETRIEVE(key, instance_id, per-group block ids, event handle, skip tokens)
  -> affinity worker
  -> wait vLLM event
  -> read_prefetched_results (unsafe_read: locks already held)
  -> H2D into paged KV
  -> record completion event
  -> stream callback finish_read_prefetched
```

`skip_first_n_tokens` 防止覆盖 vLLM APC 已共享、可能仍被读取的前缀 slots。任一 group 缺失/拷贝异常返回 `success=False`，调用方重算；finally 总会记录 completion event，避免 worker 永久等待。

索引：[`LMCacheDrivenTransferModule.retrieve`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/modules/lmcache_driven_transfer.py#L919) — `lmcache/v1/multiprocess/modules/lmcache_driven_transfer.py:919`、[`read_prefetched_results`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/distributed/storage_manager.py#L250) — `lmcache/v1/distributed/storage_manager.py:250`。

### 3. Store

```text
vLLM worker records producer event
  -> STORE(key, instance_id, per-group block ids, event handle)
  -> affinity worker waits producer event
  -> validate all groups/block ids
  -> reserve_write L1
  -> D2H paged KV -> MemoryObj
  -> stream callback finish_write
  -> StoreListener eventfd
  -> StoreController reserve_read
  -> L2 submit_store_task
  -> adapter store eventfd
  -> release L1 read lock
```

Store 对 block-id underflow/copy exception采用 fail-closed：不提交 partial object。L2 replication 是 best-effort，但 controller 无论成功/失败都必须释放 L1 read locks。

索引：[`LMCacheDrivenTransferModule.store`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/modules/lmcache_driven_transfer.py#L709) — `lmcache/v1/multiprocess/modules/lmcache_driven_transfer.py:709`、[`StoreController._process_new_keys`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/distributed/storage_controllers/store_controller.py#L556) — `lmcache/v1/distributed/storage_controllers/store_controller.py:556`。

### 4. Session Cleanup

scheduler 先释放“不再 retrieve”的 lookup locks；`request_finished` 清本地 tracker 并向所有 server 发送 `END_SESSION`。Server 删除 `Session`、发布 request-end event，并 touch 统一的 retrieve/store keys。缺少 session/key 时只警告并跳过。

## 通信机制矩阵

| 机制                        | 跨越边界                        | control/data                                   | 关键载荷                                         | 失败降级                                                            |
| --------------------------- | ------------------------------- | ---------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------- |
| ZMQ DEALER/ROUTER + msgspec | process；TCP 可跨 host          | 普通 MP 主要 control；pickle variant 也传 data | request uid/type、typed payload frames           | timeout -> unhealthy/miss；version/type mismatch fail fast          |
| CUDA IPC                    | 同 host、跨 process、同可见 GPU | data + sync                                    | memory handle、event handle、block ids           | unsupported/underflow/copy failure -> operation false + recompute   |
| Engine-driven SHM           | 同 host、共享 SHM               | data + control                                 | offset/length/shape/dtype slots                  | SHM 不可用 -> pickle                                                |
| Engine-driven pickle        | MQ 可达的 process               | data                                           | pickled CPU tensor list                          | timeout/miss -> recompute；只允许可信 peer                          |
| eventfd / pipe              | 同 process threads              | sync                                           | 可读 signal，不含 key/data                       | stale fd 抛错；pipe full 视为 signal 已 pending                     |
| FastAPI/httpx               | process/host/cluster            | control + observability                        | JSON status/config/membership/quota              | coordinator failure 不影响 serving                                  |
| EventBus/OTel               | 同 process，export 可跨 host    | observability                                  | EventType/session/metadata                       | bounded queue drop、subscriber exception 均不影响 cache correctness |
| NIXL                        | peer process/host/node          | data + handshake control                       | registration descriptors、offset/size、read task | timeout/invalid address -> miss                                     |
| Kubernetes                  | cluster deployment/control      | deployment；Service 路由 runtime traffic       | CRD/spec/ConfigMap/Service                       | node-local endpoint 缺失时连接失败，不跨节点误连 IPC                |

## ZMQ frame、线程与 handler

[`MessageQueueClient`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/mq.py#L259) — `lmcache/v1/multiprocess/mq.py:259` 使用 DEALER；[`MessageQueueServer`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/mq.py#L489) — `lmcache/v1/multiprocess/mq.py:489` 使用 ROUTER。请求 frame 为：

```text
client -> [request_uid, request_type, payload_0, ...]
server sees [identity, request_uid, request_type, payload_0, ...]
server -> [identity, request_uid, request_type, optional_response]
```

- `ClientPollingLoop` 是进程级单例线程，统一 poll clients 并完成 `MessagingFuture`。
- `mq-server-thread` 独占 ROUTER socket。
- `SYNC` handler 在 MQ 线程执行。
- `BLOCKING/NORMAL` 用 `ThreadPoolExecutor`。
- GPU `STORE/RETRIEVE` 用 `AffinityThreadPool`，以 ZMQ identity hash 固定 worker。
- `NON_BLOCKING` 枚举存在，但当前注册/派发会抛 `NotImplementedError`。

协议权威入口：[`RequestType`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/protocols/base.py#L26) — `lmcache/v1/multiprocess/protocols/base.py:26` 与 [`initialize_protocols`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/protocols/__init__.py#L47) — `lmcache/v1/multiprocess/protocols/__init__.py:47`。

## CUDA IPC vs SHM vs Pickle

| 路径                    | 谁执行 gather/scatter | bytes 在哪里                 | 典型 copy                                  | 适用                    |
| ----------------------- | --------------------- | ---------------------------- | ------------------------------------------ | ----------------------- |
| LMCache-driven CUDA IPC | server                | shared GPU allocation 与 L1  | GPU staging/transfer kernel                | CUDA 同节点，成熟主路径 |
| Engine-driven SHM       | serving worker        | server-owned shared L1 slots | worker GPU↔SHM L1                          | 非 CUDA/低 copy 同节点  |
| Engine-driven Pickle    | worker + server       | serialized bytes 经 MQ       | gather + serialize + deserialize + scatter | 通用 fallback           |

SHM worker 通过 [`EngineDrivenContextShm`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/transfer_context/shm.py#L76) — `lmcache/v1/multiprocess/transfer_context/shm.py:76` 建立 `torch.frombuffer` view；server 通过 [`ShmTransferStrategy`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/modules/server_transfer.py#L275) — `lmcache/v1/multiprocess/modules/server_transfer.py:275` 管理 pending read/write maps。Pickle 使用 `pickle.loads`，所以 MQ peer 必须可信。

## L1/L2 锁状态

| 阶段               | L1                                   | L2                                |
| ------------------ | ------------------------------------ | --------------------------------- |
| 新 store           | write lock                           | 无                                |
| GPU D2H 完成       | `finish_write` 后 readable           | 无                                |
| L1 -> L2 store     | read lock                            | adapter 自有 task                 |
| prefetch lookup    | 命中的 L1 read lock                  | lookup-and-lock 命中的 L2 lock    |
| L2 load            | temporary L1 write lock              | load-plan keys 继续锁定           |
| load 完成          | 原子 write -> read                   | 全部 unlock                       |
| serving retrieve   | read lock 持续到 H2D stream callback | 无                                |
| cancel/no-retrieve | `FREE_LOOKUP_LOCKS`                  | PrefetchController cleanup/unlock |

## P2P 三段式通信

1. MP server 通过 coordinator `GET /instances` 发现 peer 的 HTTP/MQ/NIXL 地址。
2. `P2PL2Adapter` 通过 peer MQ 发 `P2P_LOOKUP_AND_LOCK`/query，peer 只查 `skip_l2=True` 的本地 L1，并返回 `TransferChannelAddress(offset,size)`。
3. 本地 NIXL client 把远端注册 L1 pages 拉入本地 L1，随后发 unlock。

Coordinator 永远不看到 KV bytes。关键实现：[`P2PController`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/modules/p2p_controller.py#L88) — `lmcache/v1/multiprocess/modules/p2p_controller.py:88`、[`P2PL2Adapter`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/distributed/l2_adapters/p2p_l2_adapter.py#L66) — `lmcache/v1/distributed/l2_adapters/p2p_l2_adapter.py:66`、[`NixlTransferChannelClient`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/distributed/transfer_channel/impl/nixl_impl.py#L102) — `lmcache/v1/distributed/transfer_channel/impl/nixl_impl.py:102`。

## Observability 不参与正确性

[`EventBus`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/mp_observability/event_bus.py#L48) — `lmcache/v1/mp_observability/event_bus.py:48` 在 hot path 向 bounded deque append，单独 drain thread 调用 metrics/logging/tracing subscribers。CUDA stream event 优先使用 native recorder，避免 host callback 获取 GIL。队列满会 tail-drop，subscriber exception 被计数；两者都不应改变 cache store/retrieve 结果。

## 部署与安全边界

- 当前 Operator 让 engine DaemonSet `hostIPC=true`，并用 `internalTrafficPolicy=Local` Service 保证 vLLM 只连同节点 engine。
- CUDA IPC/SHM/pickle/NIXL 都假设可信 peer；普通 ZMQ 和 coordinator HTTP 默认没有应用层 TLS/auth。
- OTLP exporters 当前 `insecure=True`；离开可信网络时应由 collector/network layer 加密隔离。
- privileged + hostIPC 暴露主机 IPC/device，必须部署在可信 namespace，并遵守 Pod Security/RBAC 策略。
