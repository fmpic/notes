# 仓库与模块地图

> 分析基准：`389b9cfcb5cec502dbba6b5d13725bf2e024610d`。代码行号与结论均针对该修订；后续提交可能使行号漂移。

> 配套图：[系统全景 PNG](diagrams/01-system-landscape.png) · [构建部署 PNG](diagrams/05-build-and-deployment.png) · [全部图与 Excalidraw 源文件](diagrams/README.md)

## 顶层目录

| 路径 | 责任 | 新人备注 |
|---|---|---|
| [`lmcache/`](../lmcache/) | Python 主包：CLI、集成、引擎、存储、MP、观测和工具。 | 先看 `lmcache/v1/` 与 `lmcache/integration/`。 |
| [`csrc/`](../csrc/) | PyTorch C++/CUDA/SYCL 原生扩展、存储 connector ABI。 | 由 `setup_extensions/` 选择构建。 |
| [`rust/raw_block/`](../rust/raw_block/) | PyO3 raw-block I/O，支持 POSIX/io_uring。 | 独立 maturin 项目，不由根 `setup.py` 构建。 |
| [`setup_extensions/`](../setup_extensions/) | BuildProfile/StorageBackendProfile 发现和原生扩展策略。 | CUDA、ROCm、SYCL、MUSA profile 在这里。 |
| [`operator/`](../operator/) | Go/Kubebuilder Operator、CRD、resource builder、webhook 和 Kustomize。 | 当前生产 K8s 拓扑以此处源码为准。 |
| [`examples/`](../examples/) | 41 个 README、28 个启动入口、50 个配置文件。 | 按 [示例路线](06-examples-learning-path.md) 学习。 |
| [`tests/`](../tests/) | 单元、集成、GPU、MP、Operator 和行为契约测试。 | 判断设计文档是否过期的重要证据。 |
| [`benchmarks/`](../benchmarks/) | 工作负载、微基准和存储 I/O benchmark。 | 不等同于 correctness 测试。 |
| [`docs/source/`](../docs/source/) | Sphinx 用户文档。 | 面向使用者；部分中文 `.po` 仅登记。 |
| [`docs/design/`](../docs/design/) | 按 `lmcache/` 包树镜像的设计文档。 | 机制说明有价值，但个别文件索引/字段已滞后。 |
| [`.github/workflows/`](../.github/workflows/) | wheel、sdist、Docker、测试、文档和 Operator 发布。 | 发布 artifact 的权威配置。 |
| [`.buildkite/`](../.buildkite/) | GPU/ROCm bare-metal、vLLM、correctness、MP 综合测试。 | 运行环境重，通常不在开发机全量执行。 |
| [`docker/`](../docker/) | CUDA/ROCm、full/lightweight/standalone 镜像。 | 不同 Dockerfile 的能力并不等价。 |

## Python 包分区

### 包初始化与设备选择

[`_detect_device`](../lmcache/__init__.py#L27) — `lmcache/__init__.py:27` 依次检测 MUSA、XPU、HPU、CUDA，最后返回 CPU stub；[`_get_backend`](../lmcache/__init__.py#L68) — `lmcache/__init__.py:68` 把 Python fallback 与可用原生 ops 合并到 `lmcache.c_ops`。

### Serving engine 集成层

[`lmcache/integration/`](../lmcache/integration/) 将框架 API 翻译成 LMCache 生命周期：

- vLLM 当前进程内 facade：[`LMCacheConnectorV1Dynamic`](../lmcache/integration/vllm/lmcache_connector_v1.py#L30) — `lmcache/integration/vllm/lmcache_connector_v1.py:30`。
- vLLM 当前 MP connector：[`LMCacheMPConnector`](../lmcache/integration/vllm/lmcache_mp_connector.py#L465) — `lmcache/integration/vllm/lmcache_mp_connector.py:465`。
- vLLM role-specific service factory：[`VllmServiceFactory`](../lmcache/integration/vllm/vllm_service_factory.py#L39) — `lmcache/integration/vllm/vllm_service_factory.py:39`。
- SGLang MP adapter：[`SGLang LMCacheMPConnector`](../lmcache/integration/sglang/multi_process_adapter.py#L84) — `lmcache/integration/sglang/multi_process_adapter.py:84`。
- TensorRT-LLM MP scheduler/worker：[`LMCacheMPKvConnectorScheduler`](../lmcache/integration/tensorrt_llm/tensorrt_mp_adapter.py#L80) — `lmcache/integration/tensorrt_llm/tensorrt_mp_adapter.py:80`。

### 两套主运行架构

| 维度 | 进程内路径 | MP 路径 |
|---|---|---|
| 顶层协调者 | [`LMCacheManager`](../lmcache/v1/manager.py#L40) — `lmcache/v1/manager.py:40` | [`MPCacheServer`](../lmcache/v1/multiprocess/server.py#L63) — `lmcache/v1/multiprocess/server.py:63` |
| 主引擎/存储 | [`LMCacheEngine`](../lmcache/v1/cache_engine.py#L79) — `lmcache/v1/cache_engine.py:79` + `v1/storage_backend` | `multiprocess` modules + [`distributed.StorageManager`](../lmcache/v1/distributed/storage_manager.py#L63) — `lmcache/v1/distributed/storage_manager.py:63` |
| GPU 接入 | `GPUConnectorInterface` 直接 gather/scatter | 注册 KV IPC context 后由 server transfer module 操作 |
| 通信 | 进程内方法调用，另有 lookup/offload 服务 | ZMQ/msgspec + CUDA IPC 或 engine-driven SHM/pickle |
| L2 扩展 | `StorageBackendInterface` / native connector | `L2AdapterInterface` + Store/PrefetchController |
| 推荐场景 | 传统 vLLM connector、单进程集成 | 独立 daemon、跨 engine 生命周期、P2P、fleet 管理 |

这两套代码仍同时有效，不应简单把 `v1/storage_backend/` 全部视为废弃代码。

### MP 内部分区

- [`lmcache/v1/multiprocess/`](../lmcache/v1/multiprocess/)：ZMQ MQ、typed protocols、server compositor、session、transfer modules 和 HTTP facade。
- [`lmcache/v1/distributed/`](../lmcache/v1/distributed/)：L1 manager、L2 adapters、serde、store/prefetch/eviction controllers、quota 与 distributed transfer channel。
- [`lmcache/v1/mp_coordinator/`](../lmcache/v1/mp_coordinator/)：实例 registry、heartbeat、L2 quota/usage/eviction/resync 和 global CacheBlend directory。
- [`lmcache/v1/mp_observability/`](../lmcache/v1/mp_observability/)：EventBus、metrics、logging、tracing 和二进制 trace recorder。
- [`lmcache/v1/platform/`](../lmcache/v1/platform/)：设备 wrapper、cache context 和 eventfd/pipe abstraction。

## 原生代码与 Python 的边界

| Python 模块 | 主要来源 | 用途 |
|---|---|---|
| `lmcache.c_ops` | `csrc/pybind.cpp` + CUDA/HIP kernels | GPU KV copy、格式、事件/完成 recorder。 |
| `lmcache.xpu_ops` | `csrc/sycl/pybind_sycl.cpp` + SYCL kernels | Intel XPU ops。 |
| `lmcache.native_storage_ops` | `csrc/storage_manager/` | Bitmap、TTL lock、PeriodicEventNotifier。 |
| `lmcache.lmcache_redis` | `csrc/storage_backends/redis/` | Native RESP connector。 |
| `lmcache.lmcache_fs` | `csrc/storage_backends/fs/` | Native filesystem connector。 |
| `lmcache_rust_raw_block_io` | `rust/raw_block/` | raw block POSIX/io_uring。 |

跨语言索引来源：[`COMMON_EXTENSIONS`](../setup_extensions/common_cpp.py#L29) — `setup_extensions/common_cpp.py:29` 和 [`CudaProfile`](../setup_extensions/build_profiles/cuda.py#L23) — `setup_extensions/build_profiles/cuda.py:23`。

## Kubernetes Operator 子树

- [`operator/api/v1alpha1/`](../operator/api/v1alpha1/)：`LMCacheEngine`、`CacheBlendEngine`、`LMCacheCoordinator` CRD schema/default/validation。
- [`operator/internal/controller/`](../operator/internal/controller/)：reconcile loops。
- [`operator/internal/resources/`](../operator/internal/resources/)：DaemonSet、Deployment、Service、ConfigMap、ServiceMonitor builders。
- [`operator/internal/webhook/`](../operator/internal/webhook/)：CacheBlend pod injection。
- [`operator/config/`](../operator/config/)：Kustomize、CRD、RBAC、cert-manager、webhook、samples。

## 当前、兼容与实验代码如何辨认

- **当前主路径**：无版本后缀的 connectors、`lmcache server`、`distributed.StorageManager`、`BlendV3Module`、Operator builders。
- **版本兼容**：`lmcache_mp_connector_0180.py`、`lmcache_mp_connector_0201.py`、`lmcache_connector_v1_085.py`。
- **历史入口**：`lmcache_server` raw TCP server、`lmcache_controller` cache controller、`examples/blend_kv/`。
- **实验/测试支持**：runtime mutable config API、fault-inject/mock adapters、CacheBlend V1 示例。

## 从问题定位源码

1. CLI/启动失败：从 [`pyproject.toml`](../pyproject.toml) 的 scripts 到 `lmcache/cli/commands/`。
2. vLLM 调度命中数错误：从 `LMCacheMPConnector.get_num_new_matched_tokens` 到 `LMCacheMPSchedulerAdapter`。
3. MP wire/timeout：看 `multiprocess/protocols/`、`mq.py` 和对应 module `get_handlers()`。
4. L1/L2 miss：看 `distributed.StorageManager.submit_prefetch_task` 与 `PrefetchController`。
5. GPU copy/layout：看 `gpu_connector/kv_format/`、`kv_layer_groups.py` 和 transfer module。
6. K8s 连不上本机 daemon：看 Operator `BuildLookupService`、`BuildConnectionConfigMap` 和 `HostIPC`。
