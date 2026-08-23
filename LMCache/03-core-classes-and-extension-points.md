# 核心类、继承关系与扩展点

> 分析基准：`389b9cfcb5cec502dbba6b5d13725bf2e024610d`。代码行号与结论均针对该修订；后续提交可能使行号漂移。

> 配套图：[核心类关系 PNG](diagrams/02-core-class-hierarchy.png) · [存储流水线 PNG](diagrams/04-storage-pipeline.png) · [Excalidraw 源文件索引](diagrams/README.md)

## 先理解：LMCache 主要依赖组合，而不只是继承

核心对象关系可概括为：

```text
Serving connector
  -> ServiceFactory / adapter
  -> LMCacheManager 或 MPCacheServer
  -> Engine/Modules
  -> StorageManager
  -> L1Manager + controllers + L2 adapters
  -> device/cache context + serde/transport
```

静态扫描得到 **259** 个核心类节点、**251** 条继承边、**46** 个抽象节点和 **4** 个核心 `Protocol` 节点。

## 进程内核心组合

| 所有者                                                                                                                                                                      | 组合对象                                                                                      | 责任                                                 |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| [`LMCacheManager`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/manager.py#L40) — `lmcache/v1/manager.py:40`                                                 | metadata、engine、lookup client/server、offload server、internal API、plugins、health monitor | 统一创建和关闭组件；失败时进入重算降级。             |
| [`BaseServiceFactory`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/integration/base_service_factory.py#L40) — `lmcache/integration/base_service_factory.py:40` | `VllmServiceFactory`、`StandaloneServiceFactory`                                              | 把 serving engine role 差异隔离在 factory。          |
| [`LMCacheEngine`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/cache_engine.py#L79) — `lmcache/v1/cache_engine.py:79`                                        | TokenDatabase、GPUConnector、StorageManager、EventManager、worker/controller hooks            | 对外提供 store/retrieve/lookup/move/compress/clear。 |
| [`LMCacheEngineBuilder`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/cache_engine.py#L1969) — `lmcache/v1/cache_engine.py:1969`                             | engine/config/metadata/stats registries                                                       | 以 instance id 管理 engine 生命周期。                |

## MP 核心组合

| 所有者                                                                                                                                                                          | 组合对象                                                                                          | 责任                            |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- | ------------------------------- |
| [`MPCacheServerContext`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/engine_context.py#L128) — `lmcache/v1/multiprocess/engine_context.py:128`     | distributed StorageManager、TokenHasher、SessionManager、EventBus、layout registry、SHM pool info | modules 的共享依赖容器。        |
| [`MPCacheServer`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/server.py#L63) — `lmcache/v1/multiprocess/server.py:63`                              | `EngineModule` list                                                                               | 汇总 handler、status 和 close。 |
| [`distributed.StorageManager`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/distributed/storage_manager.py#L63) — `lmcache/v1/distributed/storage_manager.py:63` | L1Manager、Store/Prefetch/EvictionController、L2 adapters、QuotaManager                           | MP 多层存储总入口。             |
| [`MessageQueueServer`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/mq.py#L489) — `lmcache/v1/multiprocess/mq.py:489`                               | typed handlers、normal pool、affinity pool、output notifier                                       | ZMQ 请求调度。                  |

## 关键继承族

### Serving 与 GPU

| 抽象                                                                                                                                                                                      | 代表实现                                                                           |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| [`GPUConnectorInterface`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/gpu_connector/gpu_connectors.py#L42) — `lmcache/v1/gpu_connector/gpu_connectors.py:42`              | `VLLMPagedMemGPUConnectorV2/V3`、layerwise、SGLang、TRTLLM、XPU、HPU、MUSA、Mock。 |
| [`EngineDetector`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/gpu_connector/kv_format/detectors/base.py#L38) — `lmcache/v1/gpu_connector/kv_format/detectors/base.py:38` | vLLM、SGLang、TRTLLM detector。                                                    |
| [`KVFormatSpec`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/gpu_connector/kv_format/specs/base.py#L73) — `lmcache/v1/gpu_connector/kv_format/specs/base.py:73`           | 每种真实 tensor layout 的 spec。                                                   |
| [`BaseCacheContext`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/platform/base_cache_context.py#L36) — `lmcache/v1/platform/base_cache_context.py:36`                     | CUDA `GPUCacheContext`、CPU `CPUCacheContext`。                                    |
| [`DeviceIPCWrapper`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/custom_types.py#L26) — `lmcache/v1/multiprocess/custom_types.py:26`                         | `CudaIPCWrapper`、`RawCudaIPCWrapper`、`CpuShmTensorWrapper`。                     |

### 进程内存储

| 抽象                                                                                                                                                                                       | 代表实现                                    |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------- |
| [`StorageBackendInterface`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/storage_backend/abstract_backend.py#L27) — `lmcache/v1/storage_backend/abstract_backend.py:27`     | LocalDisk、Remote、P2P、Audit 等。          |
| [`AllocatorBackendInterface`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/storage_backend/abstract_backend.py#L325) — `lmcache/v1/storage_backend/abstract_backend.py:325` | LocalCPU、GDS、NIXL、PD、Maru。             |
| [`StoragePluginInterface`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/storage_backend/abstract_backend.py#L424) — `lmcache/v1/storage_backend/abstract_backend.py:424`    | DAX、Rust raw-block 等可配置插件。          |
| [`ConnectorAdapter`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/storage_backend/connector/__init__.py#L157) — `lmcache/v1/storage_backend/connector/__init__.py:157`      | Python adapter 到 native/remote connector。 |

### MP L2 与控制器

| 抽象                                                                                                                                                                                                        | 代表实现                                                                  |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| [`L2AdapterInterface`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/distributed/l2_adapters/base.py#L78) — `lmcache/v1/distributed/l2_adapters/base.py:78`                                   | FS、S3、DAX、NIXL store、Mooncake、native connector、P2P、serde wrapper。 |
| [`L2AdapterConfigBase`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/distributed/l2_adapters/config.py#L154) — `lmcache/v1/distributed/l2_adapters/config.py:154`                            | 每种 adapter 的 typed config。                                            |
| [`StorageControllerInterface`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/distributed/storage_controller.py#L13) — `lmcache/v1/distributed/storage_controller.py:13`                       | Store、Prefetch、L1/L2 eviction controllers。                             |
| [`StorePolicy`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/distributed/storage_controllers/store_policy.py#L49) — `lmcache/v1/distributed/storage_controllers/store_policy.py:49`          | DefaultStorePolicy 与自注册策略。                                         |
| [`PrefetchPolicy`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/distributed/storage_controllers/prefetch_policy.py#L21) — `lmcache/v1/distributed/storage_controllers/prefetch_policy.py:21` | Default/Retain policy。                                                   |

### Serde、传输与 RPC

| 抽象                                                                                                                                                                                                       | 代表实现                                                        |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| [`Serializer`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/distributed/serde/base.py#L48) — `lmcache/v1/distributed/serde/base.py:48` / `Deserializer` / `SerdeProcessor`                  | FP8、asymmetric、multi、async processor。                       |
| [`TransferContext`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/transfer_context/worker_transfer.py#L112) — `lmcache/v1/multiprocess/transfer_context/worker_transfer.py:112` | `LMCacheDrivenTransferContext`、`EngineDrivenTransferContext`。 |
| [`EngineDrivenContext`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/transfer_context/base.py#L109) — `lmcache/v1/multiprocess/transfer_context/base.py:109`                   | Pickle、SHM。                                                   |
| [`TransferStrategy`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/modules/server_transfer.py#L80) — `lmcache/v1/multiprocess/modules/server_transfer.py:80`                    | `PickleTransferStrategy`、`ShmTransferStrategy`。               |
| [`RpcClientTransport`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/rpc/transport.py#L20) — `lmcache/v1/rpc/transport.py:20` / `RpcServerTransport`                                         | ZMQ REQ/REP 与 ROUTER transport。                               |

## Protocol 而不是继承

- [`EngineModule`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/engine_module.py#L43) — `lmcache/v1/multiprocess/engine_module.py:43`：module 只需暴露 handlers/status/close。
- `InstanceLivenessTarget`：被 ManagementModule reaper 扫描。
- `L1ManagerProtocol`：让 GDS/普通 L1 在 controller 层共享 API。
- `L2ReconfigurableAdapter`：HTTP runtime reconfiguration 的结构契约。
- `IPCEvent`：worker adapter 只依赖 event record/handle 能力。

## 11 类主要扩展缝

| 扩展点                       | 发现/注册方式                              | 修改范围                                                |
| ---------------------------- | ------------------------------------------ | ------------------------------------------------------- |
| CLI command                  | filesystem subclass discovery              | 新增 `BaseCommand` 子类模块。                           |
| L2 adapter                   | config registry + lazy factory registry    | 实现 config/interface，并 module-level 自注册。         |
| Serde                        | name-to-constructor factory                | 实现 serializer/deserializer/processor 并接入 factory。 |
| Distributed transfer channel | self-registered factory                    | 实现 context/server/client。                            |
| Platform wrapper             | device-type registry                       | 注册 availability 与 KV wrapper factory。               |
| KV detector/spec             | filesystem subclass discovery              | 放入 detector/spec package。                            |
| MP protocol                  | static enum + validated module definitions | 同步新增 `RequestType` 与 `ProtocolDefinition`。        |
| MP HTTP API                  | router discovery                           | 新增带 module-level router 的文件。                     |
| Store/Prefetch policy        | name registry                              | 子类 + module-level register。                          |
| Request telemetry            | factory selection                          | 实现 `RequestTelemetry` 并加入选择映射。                |
| Build profile                | filesystem subclass discovery              | 新增 `BuildProfile` 或 `StorageBackendProfile` 子类。   |

核心注册函数索引：[`register_l2_adapter_type`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/distributed/l2_adapters/config.py#L55) — `lmcache/v1/distributed/l2_adapters/config.py:55`、[`register_l2_adapter_factory`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/distributed/l2_adapters/factory.py#L62) — `lmcache/v1/distributed/l2_adapters/factory.py:62`、[`register_transfer_channel_factory`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/distributed/transfer_channel/factory.py#L37) — `lmcache/v1/distributed/transfer_channel/factory.py:37`。

## 成熟度标签

| 标签                | 含义                                      | 例子                                                        |
| ------------------- | ----------------------------------------- | ----------------------------------------------------------- |
| current             | 默认/无版本后缀的活动实现。               | `LMCacheMPConnector`、distributed StorageManager、BlendV3。 |
| extension interface | 为第三方实现稳定的 ABC/Protocol/factory。 | L2AdapterInterface、TransferChannel factory。               |
| compatibility       | 为旧命令、旧命名或特定 vLLM API 保留。    | versioned connectors、`lmcache_server`、compat aliases。    |
| experimental        | 明确标记实验或故障注入。                  | mutable config API、fault-inject、CacheBlend V1 示例。      |
| test support        | 只用于测试/benchmark 的 mock/dummy。      | MockL2Adapter、MockGPUConnector。                           |
