# LMCache 新人架构指南

> 分析基准：`389b9cfcb5cec502dbba6b5d13725bf2e024610d`。代码行号与结论均针对该修订；后续提交可能使行号漂移。

这组文档面向第一次进入 LMCache 仓库的开发者。它不是用户手册的替代品，而是一张从“目录在哪里”到“请求如何跨进程移动 KV”的源码地图。分析以源码和行为测试为主要证据；当设计文档、历史示例与当前实现不一致时，会明确标注。

## 配套架构图

完整索引与可编辑源文件见 [`guides/diagrams/`](diagrams/README.md)。

| 图 | 主要问题 |
|---|---|
| [![系统全景](diagrams/01-system-landscape.png)](diagrams/01-system-landscape.png) | 进程内/MP、存储、控制面和部署面如何连接？ |
| [![核心类关系](diagrams/02-core-class-hierarchy.png)](diagrams/02-core-class-hierarchy.png) | ABC/Protocol、Factory 与 composition 如何分工？ |
| [![MP 消息流](diagrams/03-multiprocess-message-flow.png)](diagrams/03-multiprocess-message-flow.png) | REGISTER/LOOKUP/RETRIEVE/STORE/END_SESSION 各传什么？ |
| [![存储流水线](diagrams/04-storage-pipeline.png)](diagrams/04-storage-pipeline.png) | L1/L2 锁和 eventfd 如何保证异步 I/O 正确性？ |
| [![构建与部署](diagrams/05-build-and-deployment.png)](diagrams/05-build-and-deployment.png) | 源码如何变成 wheel、镜像、Operator 和 K8s runtime？ |

## 探索覆盖

- Git 跟踪文件：**1972** 个。
- 完整读取的文本源码、测试、示例、文档和配置：**1779** 个。
- 仅登记的二进制、翻译目录和显式生成文件：**193** 个。
- 完整读取字节：**20,367,813**；文本行：**459,971**。
- 读取错误：**0**；UTF-8 解码错误：**0**。
- 逐文件路径、行数、SHA-256 和处理状态见 [`source-inventory.json`](source-inventory.json) 与 [源码覆盖说明](08-source-coverage.md)。

## 推荐阅读顺序

| 顺序 | 文档 | 先回答的问题 |
|---|---|---|
| 1 | [仓库与模块地图](01-repository-map.md) | 每个目录负责什么？MP 与进程内实现为什么同时存在？ |
| 2 | [入口与生命周期](02-entrypoints-and-lifecycle.md) | `lmcache`、vLLM connector、server、coordinator 和 Operator 从哪里启动？ |
| 3 | [核心类与扩展点](03-core-classes-and-extension-points.md) | 关键 ABC/Protocol 如何派生？哪里是稳定的插件缝？ |
| 4 | [通信与数据流](04-communication-and-data-flow.md) | ZMQ、CUDA IPC、SHM、eventfd、HTTP 和 NIXL 各传什么？ |
| 5 | [构建、原生扩展与部署](05-build-native-and-deployment.md) | CUDA/ROCm/SYCL/MUSA、Rust、Docker 和 Operator 如何产生产物？ |
| 6 | [示例学习路线](06-examples-learning-path.md) | 新人应按什么顺序运行示例，需要哪些 GPU/服务？ |
| 7 | [关键代码索引](07-key-code-index.md) | 我知道概念名后，应该打开哪个符号？ |
| 8 | [源码覆盖、差异与限制](08-source-coverage.md) | 哪些文件已读？哪些现有文档已滞后？当前有什么明确限制？ |

## 一句话架构

LMCache 把 serving engine 中短命的 GPU KV cache 转换成按 token chunk 寻址的对象，通过 **L1 内存 + 可插拔 L2** 保存并复用；当前 MP 架构把缓存服务独立成 ZMQ/FastAPI 进程，通过 CUDA IPC、SHM/pickle 或 NIXL 移动数据，并可由 coordinator 与 Kubernetes Operator 管理。

关键入口：[`lmcache.cli.main.main`](../lmcache/cli/main.py#L20) — `lmcache/cli/main.py:20`；当前 MP server：[`MPCacheServer`](../lmcache/v1/multiprocess/server.py#L63) — `lmcache/v1/multiprocess/server.py:63`；vLLM MP connector：[`LMCacheMPConnector`](../lmcache/integration/vllm/lmcache_mp_connector.py#L465) — `lmcache/integration/vllm/lmcache_mp_connector.py:465`。

## 术语

| 术语 | 在本项目中的含义 |
|---|---|
| serving engine | vLLM、SGLang、TensorRT-LLM 等持有 paged KV 的推理引擎。 |
| chunk | LMCache 的缓存和 hash 粒度；一个 chunk 可对应每个 KV group 的多个 engine blocks。 |
| `ObjectKey` | MP 存储对象键，包含 chunk hash、模型、rank/object group、cache salt 等隔离信息。 |
| L0 / L1 / L2 | L0 是 serving engine 的 GPU KV；L1 是 LMCache 快速内存层；L2 是文件、对象存储、RESP、Mooncake、NIXL store、P2P 等扩展层。 |
| in-process | `LMCacheEngine` 直接位于 serving worker 进程，使用 `lmcache/v1/storage_backend/`。 |
| MP | 独立 `lmcache server` 进程，使用 `lmcache/v1/multiprocess/` 与 `lmcache/v1/distributed/`。 |
| HMA / KV groups | Hybrid Memory Allocator；不同 attention 行为或物理布局拥有独立 block-id 地址空间。 |
| PD | Prefill/Decode disaggregation；prefill 生成 KV，decode 接收并继续生成。 |
| CacheBlend | 不仅复用前缀，还复用发生位置偏移的非前缀 KV，并选择性重算。 |
| control plane | key、block id、任务、配置、发现、状态等小消息。 |
| data plane | 实际 KV tensor bytes，经 CUDA IPC、SHM/pickle、L2 I/O 或 NIXL 传输。 |

## 阅读原则

1. **遇到同名类先看完整模块路径。** 两个 `StorageManager` 分别属于进程内与 MP 架构。
2. **无版本后缀文件通常是当前入口。** `_0180`、`_0201`、`_085` 等是 vLLM API 兼容实现。
3. **源码与测试优先。** `protocols/README.md`、部分 design docs 和 raw Kubernetes 示例存在已记录的历史差异。
4. **命中失败应降级为重算。** 正常 cache miss、后端超时和部分传输失败不应破坏推理正确性。
5. **KV 数据与控制消息分开看。** 例如普通 MP ZMQ `STORE` 发送句柄和 block ids，真正数据通过 CUDA IPC；pickle variant 才把 bytes 放入 MQ。
