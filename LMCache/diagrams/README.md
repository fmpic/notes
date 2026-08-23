# LMCache 架构图索引

> 分析基准：`389b9cfcb5cec502dbba6b5d13725bf2e024610d`。每张图均保留可编辑 `.excalidraw` 源文件与已目视复核的 PNG。

| 图 | 说明 | 相关指南 |
|---|---|---|
| [系统全景 PNG](01-system-landscape.png) · [Excalidraw](01-system-landscape.excalidraw) | Serving engine、进程内/MP 路径、L0/L1/L2、Coordinator、Operator 与 Observability 的边界。 | [总览](../README.md) · [仓库地图](../01-repository-map.md) · [通信与数据流](../04-communication-and-data-flow.md) |
| [核心类关系 PNG](02-core-class-hierarchy.png) · [Excalidraw](02-core-class-hierarchy.excalidraw) | ABC/Protocol、Factory/Registry 与 runtime composition；区分两套 Storage 抽象。 | [核心类与扩展点](../03-core-classes-and-extension-points.md) · [关键代码索引](../07-key-code-index.md) |
| [MP 消息流 PNG](03-multiprocess-message-flow.png) · [Excalidraw](03-multiprocess-message-flow.excalidraw) | REGISTER、LOOKUP、RETRIEVE、STORE、END_SESSION，MQ frame、线程池及两种 transfer mode。 | [入口与生命周期](../02-entrypoints-and-lifecycle.md) · [通信与数据流](../04-communication-and-data-flow.md) |
| [存储流水线 PNG](04-storage-pipeline.png) · [Excalidraw](04-storage-pipeline.excalidraw) | L1 lock state、StoreController、PrefetchController、eventfd 与 L2 lock 生命周期。 | [核心类与扩展点](../03-core-classes-and-extension-points.md) · [通信与数据流](../04-communication-and-data-flow.md) |
| [构建部署 PNG](05-build-and-deployment.png) · [Excalidraw](05-build-and-deployment.excalidraw) | BuildPolicy、CUDA/ROCm/SYCL/MUSA、Rust、wheel、Docker、Operator 与 CI 发布。 | [构建、原生扩展与部署](../05-build-native-and-deployment.md) · [示例路线](../06-examples-learning-path.md) |

## 预览

[![LMCache 系统全景](01-system-landscape.png)](01-system-landscape.png)

[![LMCache 核心类关系](02-core-class-hierarchy.png)](02-core-class-hierarchy.png)

[![LMCache MP 消息流](03-multiprocess-message-flow.png)](03-multiprocess-message-flow.png)

[![LMCache 存储流水线](04-storage-pipeline.png)](04-storage-pipeline.png)

[![LMCache 构建与部署](05-build-and-deployment.png)](05-build-and-deployment.png)
