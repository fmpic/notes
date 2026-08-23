# 关键代码索引

> 分析基准：`389b9cfcb5cec502dbba6b5d13725bf2e024610d`。代码行号与结论均针对该修订；后续提交可能使行号漂移。

> 配套图：[核心类关系](diagrams/02-core-class-hierarchy.png) · [全部 PNG 与 Excalidraw](diagrams/README.md)

索引格式为“稳定符号链接 + 当前修订 `path:line`”。优先按符号搜索；行号只用于该基准修订的快速跳转。

## 包初始化与 CLI

| 符号 | 责任 |
|---|---|
| [`_detect_device`](../lmcache/__init__.py#L27) — `lmcache/__init__.py:27` | 设备检测 |
| [`_get_backend`](../lmcache/__init__.py#L68) — `lmcache/__init__.py:68` | 原生 ops/fallback 合并 |
| [`lmcache CLI main`](../lmcache/cli/main.py#L20) — `lmcache/cli/main.py:20` | 根命令派发 |
| [`BaseCommand`](../lmcache/cli/commands/base.py#L24) — `lmcache/cli/commands/base.py:24` | CLI 扩展基类 |
| [`_discover_commands`](../lmcache/cli/commands/__init__.py#L15) — `lmcache/cli/commands/__init__.py:15` | 命令发现 |

## 进程内引擎与生命周期

| 符号 | 责任 |
|---|---|
| [`LMCacheManager`](../lmcache/v1/manager.py#L40) — `lmcache/v1/manager.py:40` | 组件生命周期 |
| [`BaseServiceFactory`](../lmcache/integration/base_service_factory.py#L40) — `lmcache/integration/base_service_factory.py:40` | serving factory 抽象 |
| [`VllmServiceFactory`](../lmcache/integration/vllm/vllm_service_factory.py#L39) — `lmcache/integration/vllm/vllm_service_factory.py:39` | vLLM role factory |
| [`LMCacheEngine`](../lmcache/v1/cache_engine.py#L79) — `lmcache/v1/cache_engine.py:79` | store/retrieve/lookup API |
| [`LMCacheEngineBuilder`](../lmcache/v1/cache_engine.py#L1969) — `lmcache/v1/cache_engine.py:1969` | engine registry/lifecycle |
| [`storage_backend.StorageManager`](../lmcache/v1/storage_backend/storage_manager.py#L218) — `lmcache/v1/storage_backend/storage_manager.py:218` | 进程内多后端协调 |

## Serving engine integrations

| 符号 | 责任 |
|---|---|
| [`LMCacheConnectorV1Dynamic`](../lmcache/integration/vllm/lmcache_connector_v1.py#L30) — `lmcache/integration/vllm/lmcache_connector_v1.py:30` | vLLM 进程内 facade |
| [`LMCacheConnectorV1Impl`](../lmcache/integration/vllm/vllm_v1_adapter.py#L453) — `lmcache/integration/vllm/vllm_v1_adapter.py:453` | 进程内实现 |
| [`LMCacheMPConnector`](../lmcache/integration/vllm/lmcache_mp_connector.py#L465) — `lmcache/integration/vllm/lmcache_mp_connector.py:465` | 当前 vLLM MP connector |
| [`LMCacheMPSchedulerAdapter`](../lmcache/integration/vllm/vllm_multi_process_adapter.py#L560) — `lmcache/integration/vllm/vllm_multi_process_adapter.py:560` | scheduler control plane |
| [`LMCacheMPWorkerAdapter`](../lmcache/integration/vllm/vllm_multi_process_adapter.py#L1040) — `lmcache/integration/vllm/vllm_multi_process_adapter.py:1040` | worker data plane |
| [`SGLang LMCacheMPConnector`](../lmcache/integration/sglang/multi_process_adapter.py#L84) — `lmcache/integration/sglang/multi_process_adapter.py:84` | SGLang MP |
| [`TRT-LLM MP scheduler`](../lmcache/integration/tensorrt_llm/tensorrt_mp_adapter.py#L80) — `lmcache/integration/tensorrt_llm/tensorrt_mp_adapter.py:80` | TensorRT-LLM scheduler |

## MP server、协议与 MQ

| 符号 | 责任 |
|---|---|
| [`HTTP lifespan`](../lmcache/v1/multiprocess/http_server.py#L63) — `lmcache/v1/multiprocess/http_server.py:63` | 启动/关闭 |
| [`MPCacheServer`](../lmcache/v1/multiprocess/server.py#L63) — `lmcache/v1/multiprocess/server.py:63` | module compositor |
| [`_build_modules`](../lmcache/v1/multiprocess/server.py#L162) — `lmcache/v1/multiprocess/server.py:162` | 模块装配 |
| [`MPCacheServerContext`](../lmcache/v1/multiprocess/engine_context.py#L128) — `lmcache/v1/multiprocess/engine_context.py:128` | 共享 context |
| [`MessageQueueClient`](../lmcache/v1/multiprocess/mq.py#L259) — `lmcache/v1/multiprocess/mq.py:259` | DEALER/Future |
| [`MessageQueueServer`](../lmcache/v1/multiprocess/mq.py#L489) — `lmcache/v1/multiprocess/mq.py:489` | ROUTER/handler pools |
| [`RequestType`](../lmcache/v1/multiprocess/protocols/base.py#L26) — `lmcache/v1/multiprocess/protocols/base.py:26` | 请求 enum |
| [`ProtocolDefinition`](../lmcache/v1/multiprocess/protocols/base.py#L95) — `lmcache/v1/multiprocess/protocols/base.py:95` | typed contract |
| [`initialize_protocols`](../lmcache/v1/multiprocess/protocols/__init__.py#L47) — `lmcache/v1/multiprocess/protocols/__init__.py:47` | 完整性校验 |

## MP modules 与数据传输

| 符号 | 责任 |
|---|---|
| [`LookupModule`](../lmcache/v1/multiprocess/modules/lookup.py#L89) — `lmcache/v1/multiprocess/modules/lookup.py:89` | lookup/prefetch/session |
| [`LMCacheDrivenTransferModule`](../lmcache/v1/multiprocess/modules/lmcache_driven_transfer.py#L415) — `lmcache/v1/multiprocess/modules/lmcache_driven_transfer.py:415` | CUDA/device IPC store/retrieve |
| [`EngineDrivenTransferModule`](../lmcache/v1/multiprocess/modules/engine_driven_transfer.py#L67) — `lmcache/v1/multiprocess/modules/engine_driven_transfer.py:67` | 非 GPU prepare/commit |
| [`ManagementModule`](../lmcache/v1/multiprocess/modules/management.py#L29) — `lmcache/v1/multiprocess/modules/management.py:29` | clear/ping/reaper |
| [`P2PController`](../lmcache/v1/multiprocess/modules/p2p_controller.py#L88) — `lmcache/v1/multiprocess/modules/p2p_controller.py:88` | peer discovery/RPC |
| [`BlendV3Module`](../lmcache/v1/multiprocess/modules/blend_v3.py#L326) — `lmcache/v1/multiprocess/modules/blend_v3.py:326` | 当前 CacheBlend |
| [`TransferContext`](../lmcache/v1/multiprocess/transfer_context/worker_transfer.py#L112) — `lmcache/v1/multiprocess/transfer_context/worker_transfer.py:112` | worker transfer 抽象 |
| [`TransferStrategy`](../lmcache/v1/multiprocess/modules/server_transfer.py#L80) — `lmcache/v1/multiprocess/modules/server_transfer.py:80` | server transfer 抽象 |

## Distributed L1/L2

| 符号 | 责任 |
|---|---|
| [`distributed.StorageManager`](../lmcache/v1/distributed/storage_manager.py#L63) — `lmcache/v1/distributed/storage_manager.py:63` | MP storage façade |
| [`L1Manager`](../lmcache/v1/distributed/l1_manager.py#L136) — `lmcache/v1/distributed/l1_manager.py:136` | 对象/锁状态 |
| [`L2AdapterInterface`](../lmcache/v1/distributed/l2_adapters/base.py#L78) — `lmcache/v1/distributed/l2_adapters/base.py:78` | L2 contract |
| [`L2AdaptersConfig`](../lmcache/v1/distributed/l2_adapters/config.py#L376) — `lmcache/v1/distributed/l2_adapters/config.py:376` | 多 adapter config |
| [`register_l2_adapter_factory`](../lmcache/v1/distributed/l2_adapters/factory.py#L62) — `lmcache/v1/distributed/l2_adapters/factory.py:62` | lazy factory registry |
| [`StoreController`](../lmcache/v1/distributed/storage_controllers/store_controller.py#L196) — `lmcache/v1/distributed/storage_controllers/store_controller.py:196` | L1 -> L2 |
| [`PrefetchController`](../lmcache/v1/distributed/storage_controllers/prefetch_controller.py#L166) — `lmcache/v1/distributed/storage_controllers/prefetch_controller.py:166` | L2 -> L1 |
| [`L1EvictionController`](../lmcache/v1/distributed/storage_controllers/eviction_controller.py#L88) — `lmcache/v1/distributed/storage_controllers/eviction_controller.py:88` | L1 eviction |
| [`P2PL2Adapter`](../lmcache/v1/distributed/l2_adapters/p2p_l2_adapter.py#L66) — `lmcache/v1/distributed/l2_adapters/p2p_l2_adapter.py:66` | peer-as-L2 |

## GPU、布局与内存

| 符号 | 责任 |
|---|---|
| [`GPUConnectorInterface`](../lmcache/v1/gpu_connector/gpu_connectors.py#L42) — `lmcache/v1/gpu_connector/gpu_connectors.py:42` | GPU copy contract |
| [`CreateGPUConnector`](../lmcache/v1/gpu_connector/__init__.py#L60) — `lmcache/v1/gpu_connector/__init__.py:60` | 设备/engine dispatch |
| [`normalize_kv_and_discover_format`](../lmcache/v1/gpu_connector/utils.py#L202) — `lmcache/v1/gpu_connector/utils.py:202` | KV layout detection |
| [`KVLayerGroupsManager`](../lmcache/v1/kv_layer_groups.py#L257) — `lmcache/v1/kv_layer_groups.py:257` | hybrid group runtime |
| [`EngineGroupInfo`](../lmcache/v1/multiprocess/group_view.py#L26) — `lmcache/v1/multiprocess/group_view.py:26` | wire group metadata |
| [`DeviceIPCWrapper`](../lmcache/v1/multiprocess/custom_types.py#L26) — `lmcache/v1/multiprocess/custom_types.py:26` | IPC wrapper base |
| [`GPUCacheContext`](../lmcache/v1/platform/cuda/cache_context.py#L343) — `lmcache/v1/platform/cuda/cache_context.py:343` | server GPU context |
| [`MemoryObj`](../lmcache/v1/memory_management.py#L110) — `lmcache/v1/memory_management.py:110` | buffer/lifetime abstraction |

## Serde、transport 与观测

| 符号 | 责任 |
|---|---|
| [`SerdeProcessor`](../lmcache/v1/distributed/serde/base.py#L108) — `lmcache/v1/distributed/serde/base.py:108` | serde contract |
| [`create_serde_processor`](../lmcache/v1/distributed/serde/factory.py#L52) — `lmcache/v1/distributed/serde/factory.py:52` | serde factory |
| [`register_transfer_channel_factory`](../lmcache/v1/distributed/transfer_channel/factory.py#L37) — `lmcache/v1/distributed/transfer_channel/factory.py:37` | transport registry |
| [`NixlTransferChannelClient`](../lmcache/v1/distributed/transfer_channel/impl/nixl_impl.py#L102) — `lmcache/v1/distributed/transfer_channel/impl/nixl_impl.py:102` | P2P NIXL |
| [`EventNotifier`](../lmcache/v1/platform/event_notifier.py#L44) — `lmcache/v1/platform/event_notifier.py:44` | eventfd/pipe |
| [`EventBus`](../lmcache/v1/mp_observability/event_bus.py#L48) — `lmcache/v1/mp_observability/event_bus.py:48` | observability dispatch |
| [`EventType`](../lmcache/v1/mp_observability/event.py#L14) — `lmcache/v1/mp_observability/event.py:14` | event contract |

## Coordinator、控制面与 Operator

| 符号 | 责任 |
|---|---|
| [`coordinator.create_app`](../lmcache/v1/mp_coordinator/app.py#L64) — `lmcache/v1/mp_coordinator/app.py:64` | fleet app |
| [`InstanceRegistry`](../lmcache/v1/mp_coordinator/registry.py#L55) — `lmcache/v1/mp_coordinator/registry.py:55` | membership |
| [`keep_registered`](../lmcache/v1/mp_coordinator/registrar.py#L75) — `lmcache/v1/mp_coordinator/registrar.py:75` | server registration |
| [`L2EvictionManager`](../lmcache/v1/mp_coordinator/l2/eviction_manager.py#L33) — `lmcache/v1/mp_coordinator/l2/eviction_manager.py:33` | fleet quota eviction |
| [`GlobalBlendMatcher`](../lmcache/v1/mp_coordinator/blend_directory.py#L205) — `lmcache/v1/mp_coordinator/blend_directory.py:205` | global CacheBlend |
| [`legacy controller create_app`](../lmcache/v1/api_server/__main__.py#L85) — `lmcache/v1/api_server/__main__.py:85` | 历史 cache controller |
| [`operator main`](../operator/cmd/main.go#L61) — `operator/cmd/main.go:61` | controller-runtime entry |
| [`BuildDaemonSet`](../operator/internal/resources/daemonset.go#L48) — `operator/internal/resources/daemonset.go:48` | engine workload |
| [`BuildLookupService`](../operator/internal/resources/service.go#L31) — `operator/internal/resources/service.go:31` | node-local discovery |
| [`BuildConnectionConfigMap`](../operator/internal/resources/configmap.go#L41) — `operator/internal/resources/configmap.go:41` | vLLM config contract |

## 构建与发布

| 符号 | 责任 |
|---|---|
| [`setup.py main`](../setup.py#L38) — `setup.py:38` | 根构建入口 |
| [`BuildPolicy`](../setup_extensions/policy.py#L127) — `setup_extensions/policy.py:127` | profile orchestration |
| [`COMMON_EXTENSIONS`](../setup_extensions/common_cpp.py#L29) — `setup_extensions/common_cpp.py:29` | 公共 native modules |
| [`CudaProfile`](../setup_extensions/build_profiles/cuda.py#L23) — `setup_extensions/build_profiles/cuda.py:23` | CUDA build |
| [`RocmProfile`](../setup_extensions/build_profiles/rocm.py#L67) — `setup_extensions/build_profiles/rocm.py:67` | ROCm build |
| [`SyclProfile`](../setup_extensions/build_profiles/sycl.py#L21) — `setup_extensions/build_profiles/sycl.py:21` | SYCL build |
| [`Rust PyO3 module`](../rust/raw_block/src/lib.rs#L3100) — `rust/raw_block/src/lib.rs:3100` | raw-block binding |
| [`Operator Make orchestrator`](../operator/Makefile#L51) — `operator/Makefile:51` | Go build/deploy |

## 关键行为测试索引

| 契约 | 测试 |
|---|---|
| P2P lock/unlock | [`tests/v1/multiprocess/test_p2p_controller.py`](../tests/v1/multiprocess/test_p2p_controller.py) |
| P2P adapter timeout/miss | [`tests/v1/distributed/l2_adapters/test_p2p_l2_adapter.py`](../tests/v1/distributed/l2_adapters/test_p2p_l2_adapter.py) |
| StoreController | [`tests/v1/distributed/test_store_controller.py`](../tests/v1/distributed/test_store_controller.py) |
| PrefetchController/TrimPolicy | [`tests/v1/distributed/test_prefetch_controller.py`](../tests/v1/distributed/test_prefetch_controller.py) |
| Engine-driven SHM/pickle | [`tests/v1/multiprocess/test_engine_driven_transfer.py`](../tests/v1/multiprocess/test_engine_driven_transfer.py) |
| Coordinator registration | [`tests/v1/mp_coordinator/test_registrar.py`](../tests/v1/mp_coordinator/test_registrar.py) |
| Coordinator L2 quota | [`tests/v1/mp_coordinator/test_l2_api.py`](../tests/v1/mp_coordinator/test_l2_api.py) |
| EventBus trace deferral | [`tests/v1/mp_observability/subscribers/tracing/test_mp_server.py`](../tests/v1/mp_observability/subscribers/tracing/test_mp_server.py) |
| MUSA dispatch/guards | [`tests/v1/test_musa_support.py`](../tests/v1/test_musa_support.py) |
| Operator resources | [`operator/internal/resources/resources_test.go`](../operator/internal/resources/resources_test.go) |

## 兼容与历史入口

- `lmcache/integration/vllm/lmcache_mp_connector_0180.py`
- `lmcache/integration/vllm/lmcache_mp_connector_0201.py`
- `lmcache/integration/vllm/lmcache_connector_v1_085.py`
- `lmcache/v1/server/__main__.py`（`lmcache_server`）
- `lmcache/v1/api_server/__main__.py`（`lmcache_controller`）
- `lmcache/storage_backend/serde/`
- `examples/blend_kv/`
