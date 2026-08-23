# vLLM 源码导读

> [!NOTE]
> **分析快照**：`0934b267906f8cd9459f287b31647c3ed5c58e01`
> （分支 `v0.26.0-dev`，提交时间 `2026-07-26T13:07:32-07:00`）。
> 本指南按该提交的 Git 跟踪文件撰写；后续提交可能使行号和实现关系发生变化。

## 这套指南解决什么问题

这是一套面向 vLLM 新贡献者的实现导读。它尝试回答以下问题：

- 一个离线或在线请求从哪里进入，经过哪些对象，最终如何回到调用方？
- `AsyncLLM`、`LLMEngine`、`EngineCore`、Scheduler、Executor、Worker 和 ModelRunner 分别负责什么？
- 单进程、多进程、Ray、data parallel 和 Rust 前端之间如何分工？
- Python 包、CMake 原生扩展、Rust 产物和预编译 wheel 如何汇合？
- 面对数千个源码与测试文件，新人应该按什么顺序阅读？

这套指南不是安装手册、完整 API 参考或生产运维手册。使用参数、支持模型、部署、安全和贡献流程仍应以项目的[官方文档](https://github.com/zhiim/vllm/blob/v0.26.0-dev/docs/)与[贡献指南](https://github.com/zhiim/vllm/blob/v0.26.0-dev/AGENTS.md)为准。

## 分析快照与可信范围

分析以 `git ls-files -z` 得到的完整 Git 跟踪文件集合为边界，并对纳入的文本文件逐字节读取、计算内容摘要，再结合静态扫描与关键调用链人工复核。

| 项目                         |       数量 |
| ---------------------------- | ---------: |
| Git 跟踪文件                 |      6,110 |
| 纳入全文扫描的文本文件       |      5,912 |
| 纳入字节数                   | 62,112,421 |
| 纳入行数                     |  1,734,908 |
| 记录但不作源码全文分析的文件 |        198 |
| 读取错误                     |          0 |

纳入范围覆盖 Python、Rust、C/C++、CUDA/HIP、CMake、Shell、协议、配置、测试、示例、benchmark、CI 与配套设计文档。图片、音视频、字体、二进制模型或数组、批量数据集、测试快照和数据夹具等文件保留在清单中，但不把内容当作实现源码解析；具体分类与原因见[源码清单与代码索引](07-source-inventory-and-code-index.md)。

本导读采用以下解释边界：

- 以 V1 的 `AsyncLLM → EngineCoreClient → EngineCore → Scheduler → Executor → Worker → ModelRunner` 为主线。
- 把当前 `LLMEngine` 解释为使用 V1 components 的同步兼容外观，而不是另一套独立的旧内核。
- 把 Rust 前端解释为实验性的北向 HTTP/rendering/streaming 替代方案；它仍连接 Python `EngineCoreProc`，不是 Rust 推理内核。
- 对 factory、registry、plugin、限定名导入、Mixin 和运行时基类注入等动态关系，以调用点和注册点校正静态继承结果。
- 只陈述当前快照能够由源码支持的关系；平台相关、运行时生成和未在本环境执行的行为会明确标注限制。

## 推荐阅读顺序

新人主线：

`README` → `01` → `02` → `03` → `04` → `05` → `06` → `07`

1. 先用项目地图建立目录和分层的空间模型。
2. 沿一次请求建立正向执行与反向输出的行为模型。
3. 再阅读类型、组合、Mixin、Protocol 和运行时工厂。
4. 把逻辑调用扩展成真实进程、IPC 和 distributed collective。
5. 理解 Python、原生扩展和 Rust 产物如何进入 wheel。
6. 通过示例、测试和 benchmark 把静态阅读转为可观察行为。
7. 需要查证时使用集中代码索引，而不必首次逐条通读。

| 阅读目标                           | 建议顺序                     |
| ---------------------------------- | ---------------------------- |
| 快速理解一次在线生成               | `README → 01 → 02 → 04`      |
| 修改 Scheduler、Executor 或 Worker | `README → 02 → 03 → 04 → 07` |
| 新增模型或模型能力                 | `README → 03 → 06 → 07`      |
| 排查多进程或分布式问题             | `README → 02 → 04 → 07`      |
| 修改 CUDA、ROCm、CPU 或 Rust 构建  | `README → 05 → 07`           |
| 从示例开始学习                     | `README → 06 → 01 → 02`      |

## 八篇文档导航

| 文档                                                              | 主要问题                                                              | 建议读者                         |
| ----------------------------------------------------------------- | --------------------------------------------------------------------- | -------------------------------- |
| **本页：源码导读入口**                                            | 分析范围、阅读顺序、统一术语和证据约定是什么？                        | 所有人                           |
| [01：项目地图](01-project-map.md)                                 | 仓库由哪些目录和子系统组成，V1 主链路如何分层？                       | 首次阅读源码者                   |
| [02：入口与请求生命周期](02-entrypoints-and-request-lifecycle.md) | CLI、离线、在线和 Rust 前端如何启动并完成一次请求？                   | API、serving、engine 开发者      |
| [03：核心类关系](03-core-class-hierarchy.md)                      | ABC、Protocol、Mixin、组合与运行时 factory 如何共同构成类型系统？     | engine、model、platform 开发者   |
| [04：进程与模块通信](04-processes-and-module-communication.md)    | 控制面、数据面、ZMQ、共享内存消息和 distributed collective 如何配合？ | distributed、serving 开发者      |
| [05：构建与原生扩展](05-build-and-native-extensions.md)           | setuptools、CMake、Cargo、设备分支与 wheel 如何汇合？                 | kernel、platform、release 开发者 |
| [06：示例学习路线](06-examples-learning-path.md)                  | 应按什么顺序阅读示例、测试和 benchmark？                              | 希望动手验证的新人               |
| [07：源码清单与代码索引](07-source-inventory-and-code-index.md)   | 分析覆盖了什么，关键结论如何定位到代码？                              | 查证、评审和维护者               |

## 五张图怎么读

五张图各自服务于一个问题，不把所有关系挤进同一张图：

- [项目架构图](diagrams/project-architecture.png)：入口、V1 core、执行层、平台层和原生内核层，配合 01 阅读。
- [在线请求生命周期图](diagrams/online-request-lifecycle.png)：一次请求的正向调用、采样和流式返回，配合 02 阅读。
- [核心类关系图](diagrams/core-class-hierarchy.png)：继承、Protocol、Mixin、factory 和对象组合，配合 03 阅读。
- [进程通信图](diagrams/process-communication.png)：API Server、EngineCore、DP Coordinator、Worker 与 Rust 前端的通信边界，配合 04 阅读。
- [构建流水线图](diagrams/build-pipeline.png)：PEP 517、setuptools、CMake、Cargo、设备 target 和 wheel 产物，配合 05 阅读。

图中实线调用、继承、结构化接口、组合、运行时选择和 IPC 使用不同线型或标签。图是导航和论证工具，不替代正文中的代码证据。

## 统一术语表

其他文章第一次出现术语时会给出简短解释；本表是统一含义的完整版本。

| 规范词               | 本指南中的含义                                               | 需要避免的混用                             |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------ |
| V1                   | 当前快照的主 EngineCore、Scheduler、Executor 架构            | 不等同于某个单一类名                       |
| 前端（frontend）     | HTTP、CLI、schema、rendering、streaming 等北向入口层         | Rust 前端不是推理内核                      |
| `EngineClient`       | serving 层依赖的异步 Engine ABC                              | 不等同于 `EngineCoreClient`                |
| `AsyncLLM`           | 当前 Python 在线 V1 `EngineClient` 实现                      | 不写成当前实际 `AsyncLLMEngine` 实例       |
| `LLM`                | 面向用户的同步离线批量 API                                   | 内部组合 `LLMEngine`                       |
| `LLMEngine`          | 同步兼容外观，内部使用 V1 components                         | “Legacy”不表示内部仍是 V0 core             |
| `EngineCoreClient`   | Engine 外观到 `EngineCore` 的传输抽象                        | 有 in-process、MP 和 DP variants           |
| `EngineCore`         | 组合 Scheduler 与 Executor 的核心协调对象                    | 不负责 HTTP rendering                      |
| `EngineCoreProc`     | 给 `EngineCore` 加进程、ZMQ 与运行循环的部署包装             | 它继承 `EngineCore`，但职责更偏部署        |
| `EngineCoreRequest`  | 进入 core 的规范请求                                         | 与 OpenAI request schema 区分              |
| `SchedulerOutput`    | Scheduler 交给 Executor、ModelRunner 的本轮工作描述          | 不等同于用户输出                           |
| `ModelRunnerOutput`  | 模型执行或采样返回给 Scheduler 的结果                        | 还需 Scheduler 更新请求状态                |
| `EngineCoreOutputs`  | core 返回 client 的按请求输出集合                            | 还需 `OutputProcessor` 转换                |
| Scheduler            | 管理请求队列、token budget、KV blocks 与每轮工作             | 不拆成两个独立 prefill/decode scheduler    |
| Executor             | 把 core 调用分发到一个或多个 Worker 的抽象                   | backend 决定 uni、MP、Ray 等实现           |
| Worker               | 设备与分布式进程级执行外观                                   | 委托 ModelRunner，不等同于模型本身         |
| ModelRunner          | 准备 batch、KV、attention，执行 forward 与采样               | 消费 `SchedulerOutput`                     |
| ModelLoader          | 初始化模型并加载权重的策略对象                               | 由 `load_format` registry 选择             |
| ModelRegistry        | architecture name 到模型实现的懒注册与解析系统               | 支持 built-in、Transformers 和 OOT         |
| KV cache             | 保存 attention key/value 状态的块化缓存                      | 与 prefix cache 命中策略相关但不等同       |
| prefill              | 处理尚未计算的 prompt 或 context tokens                      | 与 decode 可存在于统一调度模型中           |
| decode               | 逐步生成后续 token                                           | 不代表独立进程或独立 Scheduler             |
| speculative decoding | 先提出候选 token 再验证的加速路径                            | 文中也简称 spec decode                     |
| TP                   | tensor parallel，层内张量切分                                | 属于数据面 collective                      |
| PP                   | pipeline parallel，按层或阶段流水线                          | 涉及 intermediate tensors 传输             |
| DP                   | data parallel，请求或 engine 副本分流                        | 与 TP、PP 维度区分                         |
| EP                   | expert parallel，MoE expert 分布                             | 不与 Executor 进程数直接画等号             |
| 控制面               | 请求调度、RPC、状态、握手、健康与关闭消息                    | 例如 ZMQ、`MessageQueue`、Ray RPC          |
| 数据面               | 模型张量、collective 和 PP intermediate tensors              | 例如 torch.distributed、NCCL               |
| ZMQ                  | frontend 与 EngineCore 等边界使用的 socket 层                | 必须结合 socket 类型和方向理解             |
| MessagePack/msgspec  | Python、Rust EngineCore payload 的编码与模型化方式           | 不泛写成 JSON                              |
| `MessageQueue`       | vLLM 本地多进程 Worker RPC 的共享内存消息队列                | 不等同于 ZMQ queue                         |
| ABC                  | Python 名义抽象基类，表达必须实现的方法                      | 与 Protocol 区分                           |
| Protocol             | 结构化类型契约，满足形状即可                                 | 模型不必继承共同实体基类                   |
| Mixin                | 通过多继承注入横切能力的基类                                 | 不把能力 Mixin 当成主生命周期层级          |
| factory              | 根据配置或环境选择实现类或对象                               | 图中使用“选择”边而非继承边                 |
| registry             | 名称到延迟实现或构造策略的映射                               | model、loader、plugin 分别说明             |
| platform             | CUDA、ROCm、CPU、XPU 等设备能力与实现选择层                  | 与操作系统概念区分                         |
| 原生扩展             | C、C++、CUDA、HIP 或 Rust 编译产物                           | Rust executable 与 torch op extension 分列 |
| custom op            | 通过 PyTorch dispatcher 暴露的原生算子                       | `_custom_ops.py` 通常只是 Python wrapper   |
| stable libtorch ABI  | 通过 stable registration/API 降低 PyTorch ABI 耦合的扩展路径 | 保留实际 target 名                         |
| Rust 前端            | `vllm-rs` 的 HTTP、rendering、streaming 与 client 实现       | 实验性；仍连接 Python `EngineCore`         |
| headless             | 只启动 EngineCore、不启动 API frontend 的模式                | 供外部或独立 frontend 使用                 |
| OOT plugin           | out-of-tree plugin，通过 entry point 或限定名扩展            | 首次出现写出全称                           |

## 证据与链接约定

正文中的源码证据采用以下格式：

```markdown
[`QualifiedSymbol`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/relative/path.py#L120-L168) — 一句职责或结论说明。
```

例如：[`EngineCore.step`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L581-L611) — 串起 schedule、execute、sample 与状态更新。

约定如下：

- 路径从 `guides/` 出发，以 `https://github.com/zhiim/vllm/blob/v0.26.0-dev/` 指向仓库源码；行号为 1-based 闭区间。
- 链接文本优先使用 `module.Class.method`、`Class.method`、Rust function/trait、CMake target 或配置键的真实名字。
- 每个链接都说明“为什么引用”，不只堆砌文件路径。
- 动态关系明确写成“选择”“注册”“注入”“组合”或“调用”，不使用继承关系代替。
- 行号只对页首 commit 快照有效。集中索引使用稳定分类 ID，见[07：源码清单与代码索引](07-source-inventory-and-code-index.md)。

## 适用范围与非目标

本指南聚焦源码结构、关键调用链、进程通信和构建链。以下主题只帮助定位，不在这里重复官方材料：

- 安装矩阵、硬件支持与完整配置项：见[官方 getting started](https://github.com/zhiim/vllm/blob/v0.26.0-dev/docs/getting_started/)和[configuration](https://github.com/zhiim/vllm/blob/v0.26.0-dev/docs/configuration/)。
- 生产部署、可观测性和扩缩容：见[deployment](https://github.com/zhiim/vllm/blob/v0.26.0-dev/docs/deployment/)。
- 安全边界与部署假设：见[SECURITY.md](https://github.com/zhiim/vllm/blob/v0.26.0-dev/SECURITY.md)和[安全文档](https://github.com/zhiim/vllm/blob/v0.26.0-dev/docs/usage/security.md)。
- 模型支持声明与新增模型的正式要求：见[模型文档](https://github.com/zhiim/vllm/blob/v0.26.0-dev/docs/models/)和[模型测试指南](https://github.com/zhiim/vllm/blob/v0.26.0-dev/docs/contributing/model/tests.md)。
- 贡献、测试、lint 和 PR 要求：见[AGENTS.md](https://github.com/zhiim/vllm/blob/v0.26.0-dev/AGENTS.md)。

这份材料不会修改公共 API、类型、协议、构建 target 或运行时行为，也不会接入现有 MkDocs 导航。它是固定提交上的独立分析快照。

---

下一页：[01：项目地图](01-project-map.md) · 返回：[指南入口](README.md)
