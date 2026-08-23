# 07：源码清单、扫描证据与集中代码索引

> **代码快照**：`0934b267906f8cd9459f287b31647c3ed5c58e01`。
> 行号、类型关系和调用链均以此提交为准；返回[指南入口](README.md)。

本篇保存本轮分析的证据边界，并提供其他文章共享的唯一稳定 ID 索引。它不重复解释[请求时序](02-entrypoints-and-request-lifecycle.md)、[类型关系](03-core-class-hierarchy.md)、[进程协议](04-processes-and-module-communication.md)或[构建链](05-build-and-native-extensions.md)；读者可从 ID 直接跳到定义、工厂、调用点或测试。

## 为什么需要清单

“读过仓库”必须能回答四个可审计问题：

1. **宇宙是什么**：以哪个 commit 的哪些 Git tracked paths 为全集。
2. **哪些内容被全文读取**：源码、测试、示例、脚本、协议、构建/CI 配置和架构证据文档。
3. **哪些内容没有按源码读取**：媒体、二进制、批量数据和 fixture，并逐项给出原因。
4. **结论如何回到代码**：使用固定 commit、相对路径、1-based 闭区间行号和集中 ID。

本清单固定在 commit `0934b267906f8cd9459f287b31647c3ed5c58e01`。只要 tracked set 或任一文件内容改变，manifest/corpus hash 和部分行号就可能改变；因此不能把本篇统计套用到其他提交。

清单也防止两种相反错误：只阅读少量入口却声称覆盖全仓库，以及把模型 fixture、图片或 benchmark dataset 当成实现源码来解释。排除不表示文件不重要，只表示它不属于本轮“实现文本全文分析”的证据集合。

## 清单构建方法

### 1. 固定仓库和指令

先记录 commit、branch 辅助信息、commit time 和适用的 instruction files。三份指令文件及 SHA-256 为：

| 文件                       | SHA-256                                                            |
| -------------------------- | ------------------------------------------------------------------ |
| `AGENTS.md`                | `9405571019fcc06d8ceae4cd5da0c3c8f08959dbfecfe43138cbb1b9b3dbd575` |
| `rust/AGENTS.md`           | `861a2b408e5b5c9b9723d851b255d206c4b163158a698ebe094bce86d0dc9683` |
| `rust/src/bench/AGENTS.md` | `2b45ea82eb1b87dfe54c4926626eb86ccd178b5ecfccfe137afe53affc92e179` |

branch `v0.26.0-dev` 只是当时 checkout 标签，不是稳定版本号；权威键始终是 40 位 commit SHA。

### 2. 用 Git 建立全集

使用 NUL-delimited `git ls-files -z`，避免空格、换行或特殊字符破坏路径边界。得到 6,110 个唯一 tracked paths 后，按路径、扩展名、文件形态与用途分类，并验证：

```text
tracked paths
  = included paths ∪ excluded paths
included paths ∩ excluded paths
  = ∅
```

分类结果保存为 inventory JSON、`included-files.txt` 和带 `path/category/reason` 的 `excluded-files.tsv`。每个 excluded path 都必须有非空原因；每个 included path 都必须存在且只出现一次。

### 3. 完整读取纳入集合

对 5,912 个 included files 逐字节流式读取，累计 bytes 和逻辑行数，并为每个文件生成 SHA-256。每条 content-manifest record 至少包含：

```json
{
  "path": "...",
  "bytes": 0,
  "lines": 0,
  "language": "...",
  "category": "...",
  "sha256": "..."
}
```

处理过程不依赖 shell 的 newline-delimited 文件列表；空文件、无最终换行文件和大文件都保留确定的 byte/line 计数。扫描结束后重新解析所有 JSON/JSONL outputs，并确认读取错误数组为空。

### 4. 生成两个总体验证值

- **inventory manifest SHA-256**：`bf6b59ccb1f9589bb8a469e155322feff8fae3e037b93f32dff79aaf013b27ba`
- **included corpus SHA-256**：`7a45cefd1d93163b2012b67b601730673d8bbc7e0980edc9bbc8ed8cebe3066c`

前者验证 tracked partition/classification 清单，后者验证纳入语料的确定性内容流。复现时不仅要有同样文件数，还应使用同样 snapshot、路径顺序和 framing 算法比较 hash；把文件按不同顺序直接拼接不会得到可比值。

## 纳入范围与统计

### 总量

| 指标                      |               值 |
| ------------------------- | ---------------: |
| Git tracked files         |            6,110 |
| 纳入全文扫描              |            5,912 |
| 排除并记录原因            |              198 |
| 纳入 bytes                |       62,112,421 |
| 纳入 lines                |        1,734,908 |
| 排除 paths 的文件大小合计 | 36,340,101 bytes |
| 纳入读取错误              |                0 |

“纳入”包含实现和用于理解实现的直接证据：Python/Rust/C++/CUDA/HIP/CMake、测试、examples、benchmarks、shell、协议/模板、runtime structured config、dependency/build/CI 配置和 supporting architecture docs。它不等于这些文件都属于发布 wheel。

### 顶层路径摘要

按 bytes 排序的主要顶层路径如下；“其他 26 项”包括根文件和较小配置路径。

| 顶层路径        |      文件 |          bytes |         lines |
| --------------- | --------: | -------------: | ------------: |
| `vllm/`         |     2,639 |     32,895,834 |       912,261 |
| `tests/`        |     1,678 |     14,876,539 |       432,554 |
| `csrc/`         |       283 |      4,408,131 |       112,644 |
| `rust/`         |       364 |      3,888,345 |       112,648 |
| `docs/`         |       235 |      1,827,874 |        39,865 |
| `examples/`     |       256 |      1,328,456 |        40,135 |
| `benchmarks/`   |       119 |      1,109,218 |        33,639 |
| `.buildkite/`   |       192 |        914,712 |        25,901 |
| `docker/`       |        14 |        183,011 |         4,425 |
| `tools/`        |        39 |        175,106 |         5,163 |
| `requirements/` |        25 |        132,794 |         5,801 |
| `.github/`      |        31 |         85,897 |         2,362 |
| `cmake/`        |        10 |         82,513 |         2,194 |
| 其他 26 项      |        27 |        203,991 |         5,316 |
| **合计**        | **5,912** | **62,112,421** | **1,734,908** |

这组数字说明 Python runtime 与 tests 是主体，但 Rust 与 native source 各约 11.2 万行，不能仅扫描 `.py` 得出架构结论。`examples/` 的 256 是纳入文件数，不是 256 个独立可运行程序。

### 语言摘要

语言由 path/extension classifier 归类，不是编译器语义统计。JavaScript 的高 bytes/低 lines 主要来自压缩静态资源，不能用 lines 比较实现复杂度。

| 语言               |      文件 |          bytes |         lines |
| ------------------ | --------: | -------------: | ------------: |
| Python             |     3,856 |     45,357,336 |     1,263,966 |
| Rust               |       295 |      3,496,386 |       100,357 |
| JSON               |       604 |      3,104,653 |       153,824 |
| Markdown           |       275 |      1,926,819 |        41,804 |
| CUDA               |        89 |      1,834,522 |        44,043 |
| JavaScript         |         7 |      1,516,461 |           167 |
| C/C++ header       |        84 |      1,141,649 |        30,240 |
| C++                |        36 |        692,255 |        18,178 |
| CUDA header        |        68 |        677,984 |        18,350 |
| Shell              |       132 |        622,487 |        18,196 |
| YAML               |       287 |        621,548 |        17,642 |
| Jinja template     |        86 |        291,900 |         6,411 |
| 其他 10 种文本类型 |        93 |        828,421 |        21,730 |
| **合计**           | **5,912** | **62,112,421** | **1,734,908** |

### tracked category 摘要

分类在排除前覆盖全部 6,110 paths：application source 2,066、test source 1,542、runtime structured config 583、build/CI config 392、native source 277、supporting docs 272、Rust source 247、example source 172、benchmark source 170、script source 95、protocol/template 88。其余是 dependency/supporting config 8，以及下面列出的 198 个排除项。

## 排除项与原因

| 原因                                   |    数量 | 代表路径                                                    | 为什么不按源码全文分析                           |
| -------------------------------------- | ------: | ----------------------------------------------------------- | ------------------------------------------------ |
| 二进制图片、音频、视频、字体或文档媒体 |      98 | `docs/assets/**/*.png`、`.jpg`、`.svg`、`.ico`              | 无可追踪的源码符号/调用关系；只记录资产路径      |
| 测试快照、模板输入输出或协议 fixture   |      51 | `rust/src/chat/**/fixtures/*`                               | 是实现的输入/期望数据，不把内容误当实现逻辑      |
| 未匹配实现范围的文本资产               |      37 | eval model lists、HTML overrides、`benchmarks/sonnet.txt`   | 数据、站点覆盖或批量参数，不是函数/协议/构建实现 |
| 许可证、治理或仓库管理文本             |      10 | `LICENSE`、`DCO`、`.gitignore`                              | 保留合规/管理作用，但不用于架构调用链            |
| 批量 JSONL/data fixture                |       1 | `examples/features/openai_batch/openai_example_batch.jsonl` | 请求样本数据，不是 example 实现源码              |
| 大型 benchmark dataset                 |       1 | `rust/src/bench/src/datasets/sonnet.txt`                    | workload corpus，不是 benchmark runner 实现      |
| **合计**                               | **198** |                                                             | 逐 path 记录，无未解释排除                       |

几个边界说明：

- 排除 fixture 的**内容**不表示忽略消费它的 test/renderer；对应 Rust/Python test source 已纳入。
- 图片可能表达设计信息，但本轮不对二进制视觉内容做“源码全文扫描”；架构结论仍由可索引 source/docs 人工复核。
- `openai_example_batch.jsonl` 的 inventory category 名包含 data fixture，不应误写为 Python binary。
- `csrc/.https://github.com/zhiim/vllm/blob/v0.26.0-dev/*.inl` 若被 classifier 判为不匹配文本资产，会明确列在 excluded TSV；这是一项分类限制，而不是断言编译器不会 include 它。

完整排除证据由 198 行 `excluded-files.tsv` 维护。本篇只汇总原因和代表路径，避免把大量媒体路径复制成无助于检索的正文。

## 静态扫描产物与人工复核

### 机器提取

分析工作目录生成以下中间产物；它们用于撰写本指南，但不作为 repository change 提交：

| 产物                                | 提取内容                                                          |
| ----------------------------------- | ----------------------------------------------------------------- |
| `file-inventory.json`               | tracked partition、category、reason、bytes                        |
| `content-manifest.jsonl`            | 每个 included file 的 path/bytes/lines/language/category/SHA-256  |
| `directory-stats.json`              | top-level 与 depth-2 聚合                                         |
| `language-stats.json`               | language 聚合                                                     |
| `python-analysis.json`              | imports、class/base、函数与动态 import 线索                       |
| `rust-analysis.json`                | workspace、crate、trait、impl、use                                |
| `native-registration-analysis.json` | `TORCH_LIBRARY*`、`STABLE_TORCH_LIBRARY*`、extension registration |
| `cmake-analysis.json`               | CMake commands、source variables、extension targets               |
| `entrypoints.json`                  | Python/Rust/console/script entrypoint candidates                  |
| `communication-keywords.json`       | ZMQ、MessageQueue、Ray、distributed、IPC/transport 候选           |
| `scan-summary.json`                 | corpus counts/hash/read errors                                    |

静态扫描用于建立“候选关系”，不直接把 `import` 计数当架构。尤其是 class base 扫描只能看名义声明，无法证明 factory 最终选择哪个 implementation。

### 人工复核修正的动态边

关键调用链逐段回到 source，人工确认至少以下静态扫描易漏项：

- `vllm.__getattr__` 的公共 API lazy import。
- CLI `cmd_init()` 运行时 subcommand registration。
- `EngineCoreClient.make_client()` 的 inproc/MP/DP variants。
- `SchedulerConfig.get_scheduler_cls()`、`Executor.get_class()` 与 platform `worker_cls` 的 factory 选择。
- `worker_extension_cls` 修改 `__bases__` 的运行时 MRO 注入。
- ModelRegistry lazy architecture strings、Transformers fallback 与 ModelLoader registry。
- process-local general/endpoint/platform/IO/stat plugins。
- platform import shared library后由 Torch dispatcher 注册 `torch.ops.*`。
- Rust `HandshakeOwner`/`Bootstrapped` transport 与 Python EngineCore schema compatibility。

复核后固定主链为：

```text
API/input
→ EngineCoreRequest
→ EngineCoreClient
→ EngineCore
→ Scheduler
→ Executor
→ Worker
→ ModelRunner
→ model/native op
→ EngineCoreOutputs
→ OutputProcessor
→ user/OpenAI output
```

并明确 Rust frontend 替换 Python HTTP/render/parser/streaming/client 层，不替换 Python `EngineCoreProc`、Scheduler、Worker 或 model runtime。

## 限制与复现原则

### 限制

- **静态快照**：没有运行所有设备、模型、plugins、connectors 或 deployment variants。
- **行号漂移**：链接只对固定 commit 有效；即使 symbol 未变，前置插行也会使 range 失效。
- **动态行为**：entry points、环境变量、qualified-name import、Ray actors、runtime code generation 和 `torch.compile` 不能仅靠语法树穷尽。
- **平台条件**：CUDA/ROCm/CPU/XPU/TPU 的 target、kernel、Worker 和 communicator 在不同 build/runtime 环境下选择不同。
- **生成/外部代码**：protobuf、hipify、Jinja/CuTe generated source、FetchContent/vendor 和 installed OOT plugins 可能不完整存在于 tracked source。
- **模型语义**：未执行 model eval，不能从 static source 声称输出质量、数值精度或性能达标。
- **排除语料**：198 个 paths 未作为实现源码全文解析；其路径和原因可审计，但内容不支撑本文架构结论。

### 最小复现原则

复现者应：

1. checkout 完整 40 位 commit，确认 working tree 不混入 untracked corpus。
2. 重新读取 nearest `AGENTS.md`，对比 instruction hashes。
3. 以 `git ls-files -z` 建全集，不使用会受 `.gitignore` 或 locale 影响的非 Git 文件遍历代替。
4. 保持同一 included/excluded 分类规则、路径排序、line/framing 规则。
5. 对 included files 全量读取并计算 file/corpus hash；不以采样 grep 代替。
6. 重新运行结构提取，再人工复核 factory/plugin/IPC/native registration 主链。
7. 运行 range/link validator，最后才更新 commit、统计和集中 ID。

若只需把指南迁移到新 commit，也不能仅做全局行号搜索替换：先判断协议、类型和行为是否改变，再决定结论是否仍成立。

## 集中代码索引

索引表的“关系”列只使用冻结词汇：`定义`、`实现`、`继承`、`Protocol`、`Mixin`、`组合`、`调用`、`选择`、`注册`、`注入`、`IPC`、`构建`、`验证`。ID 各前缀独立递增；相同 symbol 的 contract、implementation 和 call site 可拆成多项。

### ENTRY：公开 API、CLI 与 serving

| ID          | 符号/目标                                    | 关系     | 代码位置                                                                                                                        | 职责/证据                                            |
| ----------- | -------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| `ENTRY-001` | `project.scripts.vllm`                       | 定义     | [`pyproject.toml`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/pyproject.toml#L43-L47)                                       | 安装后的 `vllm` console script。                     |
| `ENTRY-002` | `vllm.__getattr__`                           | 选择     | [`vllm/__init__.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/__init__.py#L16-L73)                                   | 公共对象按名称 lazy import。                         |
| `ENTRY-003` | `cli.main.main`                              | 调用     | [`main.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/cli/main.py#L16-L97)                                | 初始化并分派 CLI subcommands。                       |
| `ENTRY-004` | `ServeSubcommand.cmd`                        | 选择     | [`serve.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/cli/serve.py#L50-L149)                             | 选择 server/headless/Rust/DP 启动模式。              |
| `ENTRY-005` | `LLM.generate`                               | 调用     | [`llm.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/llm.py#L411-L470)                                    | 同步离线 generation facade。                         |
| `ENTRY-006` | `OfflineInferenceMixin._run_completion`      | 调用     | [`offline_utils.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/offline_utils.py#L290-L350)                | batch render、enqueue 与 step loop。                 |
| `ENTRY-007` | `build_async_engine_client_from_engine_args` | 组合     | [`api_server.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/api_server.py#L147-L193)               | 在线 server 构造 `AsyncLLM` 并绑定关闭。             |
| `ENTRY-008` | `build_app`                                  | 注册     | [`api_server.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/api_server.py#L196-L290)               | FastAPI core routers 与 endpoint plugins。           |
| `ENTRY-009` | `init_app_state`                             | 注入     | [`api_server.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/api_server.py#L359-L489)               | renderer、models、serving objects 注入 app state。   |
| `ENTRY-010` | chat completion route                        | 调用     | [`api_router.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/chat_completion/api_router.py#L53-L76) | HTTP request 到 chat serving object。                |
| `ENTRY-011` | `OpenAIServingChat._create_chat_completion`  | 调用     | [`serving.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/openai/chat_completion/serving.py#L255-L408)     | render、sampling 与 stream/full response 分叉。      |
| `ENTRY-012` | `EndpointPlugin`                             | Protocol | [`interface.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/plugins/endpoint_plugins/interface.py#L1-L81)              | HTTP route extension contract 与 EngineClient 边界。 |

### ENGINE：Engine facade、client 与 core

| ID           | 符号/目标                      | 关系 | 代码位置                                                                                                             | 职责/证据                                             |
| ------------ | ------------------------------ | ---- | -------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| `ENGINE-001` | `EngineClient`                 | 定义 | [`protocol.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/engine/protocol.py#L41-L244)                     | serving 层依赖的异步 ABC。                            |
| `ENGINE-002` | `AsyncLLM`                     | 实现 | [`async_llm.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L71-L147)                | 当前 Python 在线 V1 EngineClient。                    |
| `ENGINE-003` | `AsyncLLM.add_request`         | 调用 | [`async_llm.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L281-L417)               | 同时注册 collector、OutputProcessor 与 core request。 |
| `ENGINE-004` | `AsyncLLM.generate`            | 调用 | [`async_llm.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L525-L637)               | request-scoped async generator、cancel 与 abort。     |
| `ENGINE-005` | `AsyncLLM._run_output_handler` | 调用 | [`async_llm.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/async_llm.py#L638-L709)               | core batch outputs 分发到每请求 collector。           |
| `ENGINE-006` | `LLMEngine`                    | 组合 | [`llm_engine.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/llm_engine.py#L48-L139)              | 同步兼容外观，组合 V1 processor/client。              |
| `ENGINE-007` | `LLMEngine.add_request`        | 调用 | [`llm_engine.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/llm_engine.py#L218-L295)             | 离线 input 转 core request。                          |
| `ENGINE-008` | `LLMEngine.step`               | 调用 | [`llm_engine.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/llm_engine.py#L296-L335)             | 同步消费 core outputs。                               |
| `ENGINE-009` | `EngineCoreClient.make_client` | 选择 | [`core_client.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L78-L141)            | 选择 inproc、MP、DP 与 async variants。               |
| `ENGINE-010` | `InprocClient`                 | 实现 | [`core_client.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L291-L340)           | 与 `EngineCore` 同进程的直接调用 client。             |
| `ENGINE-011` | `MPClient`                     | 实现 | [`core_client.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L482-L658)           | 多进程 ZMQ client 共同实现。                          |
| `ENGINE-012` | `EngineCore.__init__`          | 组合 | [`core.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L103-L178)                         | 组合 Executor、Scheduler、KV/EC managers。            |
| `ENGINE-013` | `EngineCore.step`              | 调用 | [`core.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L581-L611)                         | 单轮 schedule、execute、update。                      |
| `ENGINE-014` | `EngineCoreProc`               | 继承 | [`core.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L1000-L1059)                       | 给 core 增加 process、socket 与 busy loop。           |
| `ENGINE-015` | `InputProcessor`               | 调用 | [`input_processor.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/input_processor.py#L36-L132)    | frontend EngineInput 到 EngineCoreRequest。           |
| `ENGINE-016` | `OutputProcessor`              | 调用 | [`output_processor.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/output_processor.py#L426-L560) | EngineCoreOutputs 到 RequestOutput/pooling output。   |

### SCHED：请求状态、Scheduler 与 KV blocks

| ID          | 符号/目标                           | 关系 | 代码位置                                                                                                           | 职责/证据                                               |
| ----------- | ----------------------------------- | ---- | ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------- |
| `SCHED-001` | `SchedulerInterface`                | 定义 | [`interface.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/core/sched/interface.py#L37-L159)          | Scheduler ABC contract。                                |
| `SCHED-002` | `SchedulerConfig.get_scheduler_cls` | 选择 | [`scheduler.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/config/scheduler.py#L170-L194)                | 默认、async 或 qualified custom scheduler。             |
| `SCHED-003` | `Scheduler`                         | 实现 | [`scheduler.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/core/sched/scheduler.py#L69-L190)          | waiting/running queues、budgets 与 managers。           |
| `SCHED-004` | `Scheduler.schedule`                | 调用 | [`scheduler.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/core/sched/scheduler.py#L427-L575)         | 在统一 token budget 下选择本轮工作。                    |
| `SCHED-005` | `Scheduler.update_from_output`      | 调用 | [`scheduler.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/core/sched/scheduler.py#L1580-L1674)       | ModelRunnerOutput 更新 request 状态并生成 core output。 |
| `SCHED-006` | `SchedulerOutput`                   | 定义 | [`output.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/core/sched/output.py#L193-L254)               | Scheduler 交给 Executor/Worker 的控制描述。             |
| `SCHED-007` | `Request`/`RequestStatus`           | 定义 | [`request.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/request.py#L59-L169)                         | V1 request state 与 computed/output tokens。            |
| `SCHED-008` | `KVCacheManager`                    | 组合 | [`kv_cache_manager.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/core/kv_cache_manager.py#L117-L190) | block pool/coordinator、prefix match 与 allocation。    |

### EXEC：Executor backend

| ID         | 符号/目标                          | 关系 | 代码位置                                                                                                                   | 职责/证据                                       |
| ---------- | ---------------------------------- | ---- | -------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| `EXEC-001` | `Executor`                         | 定义 | [`abstract.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/abstract.py#L37-L184)                      | Worker dispatch ABC 与 control-plane contract。 |
| `EXEC-002` | `Executor.get_class`               | 选择 | [`abstract.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/abstract.py#L48-L91)                       | uni/mp/Ray/external/custom backend factory。    |
| `EXEC-003` | `UniProcExecutor`                  | 实现 | [`uniproc_executor.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/uniproc_executor.py#L45-L124)      | 进程内调用 WorkerWrapper。                      |
| `EXEC-004` | `ExecutorWithExternalLauncher`     | 继承 | [`uniproc_executor.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/uniproc_executor.py#L150-L196)     | 外部 world 每 rank 一个本地 Worker。            |
| `EXEC-005` | `MultiprocExecutor`                | 实现 | [`multiproc_executor.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/multiproc_executor.py#L103-L144) | 创建/连接 MQ-based Worker processes。           |
| `EXEC-006` | `MultiprocExecutor.collective_rpc` | IPC  | [`multiproc_executor.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/multiproc_executor.py#L320-L424) | broadcast method/args 并按 rank 收 response。   |
| `EXEC-007` | `RayDistributedExecutor`           | 实现 | [`ray_executor.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/ray_executor.py#L64-L89)               | 旧 Ray actor/DAG executor 路径。                |
| `EXEC-008` | `RayExecutorV2`                    | 继承 | [`ray_executor_v2.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/ray_executor_v2.py#L219-L224)       | 基于 MultiprocExecutor 语义的 Ray V2。          |

### WORKER：Worker、wrapper 与 ModelRunner

| ID           | 符号/目标                       | 关系 | 代码位置                                                                                                                    | 职责/证据                                            |
| ------------ | ------------------------------- | ---- | --------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| `WORKER-001` | `WorkerBase`                    | 定义 | [`worker_base.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/worker_base.py#L39-L181)                   | interface-like device Worker base。                  |
| `WORKER-002` | `WorkerWrapperBase`             | 组合 | [`worker_base.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/worker_base.py#L187-L242)                  | Executor 与 concrete Worker 之间的进程包装。         |
| `WORKER-003` | `WorkerWrapperBase.init_worker` | 注入 | [`worker_base.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/worker_base.py#L230-L320)                  | qualified import、platform Worker 与 extension MRO。 |
| `WORKER-004` | GPU `Worker`                    | 实现 | [`gpu_worker.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu_worker.py#L128-L180)                    | device/distributed 生命周期与 runner 所有权。        |
| `WORKER-005` | `Worker.execute_model`          | 调用 | [`gpu_worker.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu_worker.py#L1018-L1107)                  | PP receive、runner execute、PP send。                |
| `WORKER-006` | `GPUModelRunner` V1             | 实现 | [`gpu_model_runner.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu_model_runner.py#L453-L493)        | persistent batch、KV/attention、forward/sample。     |
| `WORKER-007` | `GPUModelRunner` V2             | 实现 | [`gpu/model_runner.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu/model_runner.py#L126-L154)        | 并列 V2 runner implementation。                      |
| `WORKER-008` | `GPUModelRunner.load_model`     | 调用 | [`gpu_model_runner.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/worker/gpu_model_runner.py#L5294-L5326)      | ModelRunner 进入 ModelLoader。                       |
| `WORKER-009` | `WorkerProc.worker_busy_loop`   | IPC  | [`multiproc_executor.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/executor/multiproc_executor.py#L997-L1027) | Worker process 接收、执行、回传 RPC。                |

### MODEL：模型能力、registry 与 loader

| ID          | 符号/目标                    | 关系     | 代码位置                                                                                                                           | 职责/证据                                           |
| ----------- | ---------------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| `MODEL-001` | `VllmModel` 等模型接口       | Protocol | [`interfaces_base.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/interfaces_base.py#L46-L149)      | runtime-checkable 结构化模型 contract。             |
| `MODEL-002` | `SupportsLoRA`/`SupportsPP`  | Mixin    | [`interfaces.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/interfaces.py#L545-L683)               | 横切 capability 与 helper 行为。                    |
| `MODEL-003` | `LlamaForCausalLM`           | 组合     | [`llama.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/llama.py#L446-L553)                         | 代表性 nn.Module、capabilities 和 `LlamaModel`。    |
| `MODEL-004` | `_ModelRegistry`             | 注册     | [`registry.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/registry.py#L1032-L1081)                 | eager/lazy architecture entries。                   |
| `MODEL-005` | `resolve_model_cls`          | 选择     | [`registry.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/models/registry.py#L1278-L1331)                 | in-tree、Transformers 与 OOT model implementation。 |
| `MODEL-006` | ModelLoader registry/factory | 选择     | [`model_loader/__init__.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/model_loader/__init__.py#L27-L127) | `load_format` 到 loader class。                     |
| `MODEL-007` | `BaseModelLoader.load_model` | 调用     | [`base_loader.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/model_loader/base_loader.py#L25-L82)         | instantiate、load weights、post-load processing。   |
| `MODEL-008` | `initialize_model`           | 调用     | [`utils.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/model_executor/model_loader/utils.py#L41-L108)                    | 解析 architecture 并调用 model constructor。        |

### DIST：消息、IPC、distributed 与 plugins

| ID         | 符号/目标                               | 关系 | 代码位置                                                                                                                              | 职责/证据                                                 |
| ---------- | --------------------------------------- | ---- | ------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| `DIST-001` | `EngineCoreReadyResponse`               | 定义 | [`engine/__init__.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/__init__.py#L69-L87)                             | core profiling/config ready payload。                     |
| `DIST-002` | `EngineCoreRequest`                     | 定义 | [`engine/__init__.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/__init__.py#L90-L164)                            | frontend 到 core 的 array-like schema。                   |
| `DIST-003` | `EngineCoreOutputs`                     | 定义 | [`engine/__init__.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/__init__.py#L168-L252)                           | request/utility/stats output batch。                      |
| `DIST-004` | `EngineCoreRequestType`                 | 定义 | [`engine/__init__.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/__init__.py#L254-L269)                           | 单字节 ADD/ABORT/WAVE/UTILITY control types。             |
| `DIST-005` | `MPClient._send_input`                  | IPC  | [`core_client.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core_client.py#L876-L888)                            | engine identity、type、MessagePack 与 aux frames。        |
| `DIST-006` | `EngineCoreProc.process_input_sockets`  | IPC  | [`core.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L1599-L1710)                                        | DEALER/XSUB 收包并写 core queue。                         |
| `DIST-007` | `EngineCoreProc.process_output_sockets` | IPC  | [`core.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/core.py#L1712-L1772)                                        | 按 client index PUSH output。                             |
| `DIST-008` | `MsgpackEncoder`                        | 实现 | [`serial_utils.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/serial_utils.py#L136-L232)                                 | MessagePack 编码与 auxiliary tensor buffers。             |
| `DIST-009` | `TensorIpcSender`/`TensorIpcReceiver`   | IPC  | [`tensor_ipc.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/tensor_ipc.py#L45-L168)                               | multimodal tensor torch-shm out-of-band path。            |
| `DIST-010` | `MessageQueue`                          | IPC  | [`shm_broadcast.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/distributed/device_communicators/shm_broadcast.py#L370-L504) | local shared-memory/remote TCP Worker control transport。 |
| `DIST-011` | `init_distributed_environment`          | 调用 | [`parallel_state.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/distributed/parallel_state.py#L1582-L1737)                  | torch distributed world 初始化。                          |
| `DIST-012` | `initialize_model_parallel`             | 组合 | [`parallel_state.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/distributed/parallel_state.py#L1740-L1979)                  | TP/PP/DP/EP/PCP/DCP groups。                              |
| `DIST-013` | `DPCoordinator`                         | IPC  | [`coordinator.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/coordinator.py#L23-L137)                             | DP load stats、wave 与 elastic control process。          |
| `DIST-014` | `CoreEngineProcManager`                 | 组合 | [`engine/utils.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/engine/utils.py#L120-L243)                                 | EngineCore process ownership/liveness。                   |
| `DIST-015` | `APIServerProcessManager`               | 组合 | [`v1/utils.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/utils.py#L166-L325)                                            | 多 API process 与 endpoint 回报 pipe。                    |
| `DIST-016` | `KVConnectorBase_V1`                    | 定义 | [`base.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/distributed/kv_transfer/kv_connector/v1/base.py#L171-L264)            | Scheduler/Worker 两侧 KV connector contract。             |
| `DIST-017` | plugin loaders                          | 注册 | [`plugins/__init__.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/plugins/__init__.py#L15-L155)                             | process-local group discovery 与 allowlist。              |
| `DIST-018` | `MsgpackDecoder`                        | 实现 | [`serial_utils.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/v1/serial_utils.py#L313-L443)                                 | MessagePack root/auxiliary/OOB tensor 解码。              |

### RUST：Rust frontend、server 与 EngineCore client

| ID         | 符号/目标                     | 关系 | 代码位置                                                                                                                    | 职责/证据                                           |
| ---------- | ----------------------------- | ---- | --------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| `RUST-001` | `vllm-rs main`                | 调用 | [`main.rs`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/cmd/src/main.rs#L85-L130)                               | Rust CLI/frontend command dispatch。                |
| `RUST-002` | `serve_with_router_extension` | 组合 | [`lib.rs`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/server/src/lib.rs#L151-L270)                             | Axum router、state、HTTP 与可选 gRPC server。       |
| `RUST-003` | Rust `EngineCoreRequestType`  | 定义 | [`request.rs`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/engine-core-client/src/protocol/request.rs#L13-L57)  | Python-compatible 四个 wire type values。           |
| `RUST-004` | Rust `EngineCoreRequest`      | 定义 | [`request.rs`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/engine-core-client/src/protocol/request.rs#L70-L139) | serde tuple compatible request schema。             |
| `RUST-005` | Rust `EngineCoreClient`       | 实现 | [`client.rs`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/engine-core-client/src/client.rs#L198-L355)           | output/dispatch/abort/coordinator tasks。           |
| `RUST-006` | `EngineCoreClient.call`       | 调用 | [`client.rs`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/engine-core-client/src/client.rs#L475-L533)           | 注册 request stream 并发送 Add。                    |
| `RUST-007` | `EngineCoreOutputStream`      | 实现 | [`stream.rs`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/engine-core-client/src/client/stream.rs#L43-L165)     | request-scoped stream 与 drop auto-abort。          |
| `RUST-008` | Rust transport                | IPC  | [`transport.rs`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/engine-core-client/src/transport.rs#L29-L137)      | Router/Pull、engine identity 与 connected handles。 |

### NATIVE：custom op registration 与 implementation

| ID           | 符号/目标                       | 关系 | 代码位置                                                                                                                       | 职责/证据                                           |
| ------------ | ------------------------------- | ---- | ------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------- |
| `NATIVE-001` | `_custom_ops` bootstrap         | 调用 | [`vllm/_custom_ops.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/_custom_ops.py#L1-L31)                             | platform kernel import 与 fake registration setup。 |
| `NATIVE-002` | `CudaPlatform.import_kernels`   | 选择 | [`cuda.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/platforms/cuda.py#L220-L230)                                   | 导入 stable、MoE 与 optional QuTLASS libraries。    |
| `NATIVE-003` | `REGISTER_EXTENSION`            | 定义 | [`registration.h`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/csrc/core/registration.h#L1-L28)                             | 生成 Python `PyInit_*` 入口。                       |
| `NATIVE-004` | `_C` stable schemas             | 注册 | [`torch_bindings.cpp`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/csrc/libtorch_stable/torch_bindings.cpp#L1-L24)          | `STABLE_TORCH_LIBRARY_FRAGMENT(_C, ...)`。          |
| `NATIVE-005` | stable dispatch implementations | 实现 | [`torch_bindings.cpp`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/csrc/libtorch_stable/torch_bindings.cpp#L616-L764)       | CUDA/CPU/Composite dispatch key bindings。          |
| `NATIVE-006` | CPU op registration             | 注册 | [`cpu/torch_bindings.cpp`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/csrc/cpu/torch_bindings.cpp#L309-L339)               | target-name namespace 与 CPU implementations。      |
| `NATIVE-007` | `silu_and_mul_per_block_quant`  | 调用 | [`vllm/_custom_ops.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/_custom_ops.py#L439-L484)                          | Python output setup 到 `torch.ops._C`。             |
| `NATIVE-008` | `_moe_C` schemas/impl           | 注册 | [`moe/torch_bindings.cpp`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/csrc/libtorch_stable/moe/torch_bindings.cpp#L1-L157) | 独立 MoE namespace 与 dispatch implementation。     |

### BUILD：PEP 517、setuptools、CMake、Cargo 与 wheel

| ID          | 符号/目标                          | 关系 | 代码位置                                                                                                            | 职责/证据                                              |
| ----------- | ---------------------------------- | ---- | ------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| `BUILD-001` | PEP 517 build system               | 构建 | [`pyproject.toml`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/pyproject.toml#L1-L15)                            | build backend 与 isolated build requirements。         |
| `BUILD-002` | `CMakeExtension`                   | 定义 | [`setup.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L184-L189)                                     | setuptools logical extension。                         |
| `BUILD-003` | `cmake_build_ext.configure`        | 构建 | [`setup.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L239-L322)                                     | 传 target device、Python、cache、compiler flags。      |
| `BUILD-004` | `cmake_build_ext.build_extensions` | 构建 | [`setup.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L324-L386)                                     | build targets 并 install components。                  |
| `BUILD-005` | precompiled build classes          | 选择 | [`setup.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L460-L511)                                     | 跳过 CMake/Cargo 或 fallback。                         |
| `BUILD-006` | precompiled wheel extraction       | 构建 | [`setup.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L748-L871)                                     | 提取 allowlisted native/Rust/vendor artifacts。        |
| `BUILD-007` | `ext_modules`                      | 选择 | [`setup.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L1116-L1180)                                   | 按 device/compiler 条件声明 targets。                  |
| `BUILD-008` | package/wheel aggregation          | 构建 | [`setup.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L1182-L1304)                                   | package data、Rust、requirements、commands 汇聚。      |
| `BUILD-009` | `define_extension_target`          | 构建 | [`utils.cmake`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/cmake/utils.cmake#L555-L640)                         | source、ABI、link、install component adapter。         |
| `BUILD-010` | `_C_stable_libtorch` target        | 构建 | [`CMakeLists.txt`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/CMakeLists.txt#L1087-L1123)                       | 主 stable-libtorch custom-op library。                 |
| `BUILD-011` | CPU ISA targets                    | 构建 | [`cpu_extension.cmake`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/cmake/cpu_extension.cmake#L480-L574)         | `_C`、AVX512 与 AVX2 variants。                        |
| `BUILD-012` | Cargo workspace                    | 构建 | [`rust/Cargo.toml`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/Cargo.toml#L1-L31)                          | Rust crates 与 shared dependencies。                   |
| `BUILD-013` | `rust_extensions`                  | 构建 | [`build_rust.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/tools/build_rust.py#L15-L38)                       | `vllm-rs` Exec 与 `_rust_tool_parser` PyO3。           |
| `BUILD-014` | CUDA Docker wheel aggregation      | 构建 | [`Dockerfile`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/docker/Dockerfile#L514-L595)                          | 合并 csrc wheel、Rust artifacts 与完整 Python source。 |
| `BUILD-015` | release wheel matrix               | 构建 | [`release-pipeline.yaml`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/.buildkite/release-pipeline.yaml#L34-L181) | CUDA/CPU/architecture wheel variants。                 |

### EXAMPLE：示例、行为 tests 与 benchmarks

| ID            | 符号/目标                  | 关系 | 代码位置                                                                                                                                                     | 职责/证据                                           |
| ------------- | -------------------------- | ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------- |
| `EXAMPLE-001` | minimal offline generation | 调用 | [`basic.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/basic/offline_inference/basic.py#L1-L34)                                                | `LLM` + prompts + SamplingParams + RequestOutput。  |
| `EXAMPLE-002` | OpenAI chat client         | 调用 | [`openai_chat_completion_client.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/basic/online_serving/openai_chat_completion_client.py#L1-L63)   | full/stream OpenAI-compatible request。             |
| `EXAMPLE-003` | structured outputs         | 调用 | [`structured_outputs_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/features/structured_outputs/structured_outputs_offline.py#L1-L112) | choice/regex/JSON/grammar constraints。             |
| `EXAMPLE-004` | offline data parallel      | 调用 | [`data_parallel_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/features/data_parallel/data_parallel_offline.py#L1-L205)                | prompt sharding 与 DP process/rank setup。          |
| `EXAMPLE-005` | external launcher          | 调用 | [`torchrun_example_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/features/torchrun/torchrun_example_offline.py#L1-L74)                | torchrun TP×PP 与 control/data groups。             |
| `EXAMPLE-006` | RL async weight sync       | 调用 | [`rlhf_async_new_apis.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/rl/rlhf_async_new_apis.py#L1-L80)                                         | trainer/inference ownership、pause 与 NCCL update。 |
| `EXAMPLE-007` | Scheduler unit behavior    | 验证 | [`test_scheduler.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/tests/v1/core/test_scheduler.py#L1-L118)                                                | waiting/request/stats 与 scheduling fixture。       |
| `EXAMPLE-008` | AsyncLLM behavior          | 验证 | [`test_async_llm.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/tests/v1/engine/test_async_llm.py#L1-L120)                                              | delta/final outputs、并发与 cancellation。          |
| `EXAMPLE-009` | OpenAI chat behavior       | 验证 | [`test_chat_completion.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/tests/entrypoints/openai/chat_completion/test_chat_completion.py#L1-L118)         | remote server protocol 与 invalid constraints。     |
| `EXAMPLE-010` | torchrun example test      | 验证 | [`test_torchrun_example.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/tests/distributed/test_torchrun_example.py#L1-L82)                               | external launcher 示例的 distributed test。         |
| `EXAMPLE-011` | latency benchmark          | 验证 | [`latency.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/benchmarks/latency.py#L1-L125)                                                            | fixed-batch warmup、latency 与 percentiles。        |
| `EXAMPLE-012` | throughput benchmark       | 验证 | [`throughput.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/benchmarks/throughput.py#L1-L130)                                                      | offline request/token throughput。                  |
| `EXAMPLE-013` | serving benchmark          | 验证 | [`serve.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/benchmarks/serve.py#L1-L53)                                                                 | endpoint load、stream timing 与 request rate。      |
| `EXAMPLE-014` | prefix cache example       | 验证 | [`prefix_caching_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/features/automatic_prefix_caching/prefix_caching_offline.py#L1-L98)    | cache on/off output invariance。                    |
| `EXAMPLE-015` | multimodal render/generate | 调用 | [`example_mm_serve.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/scale_out/example_mm_serve.py#L1-L115)                                       | serialized features 跨 endpoint composition。       |

## 索引维护规则

1. **先更新 snapshot，再更新行号**：确认 commit 和行为差异；不要先机械重定位旧结论。
2. **ID 不重排**：每个 prefix 从 `001` 递增；删除条目时保留 tombstone/说明，编号不复用，新条目只追加。
3. **一个连续证据一个链接**：不把不连续 ranges 伪装成一个范围；contract、implementation、factory 和 call site 可拆项。
4. **关系词受控**：只用本篇列出的 13 个词；factory 是“选择”，plugin/registry 是“注册”，`worker_extension_cls` 是“注入”，不要写成继承。
5. **symbol 优先**：链接文本优先 qualified symbol/target；无单一定义的运行时对象索引 registration/factory/call point。
6. **路径相对 `guides/`**：源码统一 `https://github.com/zhiim/vllm/blob/v0.26.0-dev/...#Lx-Ly`，图统一 `diagrams/...`，文档同级相对链接。
7. **范围最小自洽**：包含 signature、关键 branch 和必要 return，不用整文件 range 掩盖证据。
8. **自动检查**：验证 path 存在、1-based range 在文件行数内、Markdown link anchor 可解析、ID 唯一且单调、关系词合法。
9. **正文与集中索引去重**：01–06 解释上下文并给局部索引；07 维护 ID，不在这里重写完整架构叙述。
10. **图证据一致**：图中节点/箭头至少能映射到所属文章局部索引或本篇 ID；动态边必须标注选择/注册/注入/组合。

一次维护的推荐顺序是：`commit/instructions → inventory → full read/hash → static extraction → manual review → prose → local links → central IDs → diagrams → validation`。

## 扫描摘要

| 字段                          | 固定值                                                                                |
| ----------------------------- | ------------------------------------------------------------------------------------- |
| commit                        | `0934b267906f8cd9459f287b31647c3ed5c58e01`                                            |
| branch（辅助）                | `v0.26.0-dev`                                                                         |
| commit time                   | `2026-07-26T13:07:32-07:00`                                                           |
| tracked / included / excluded | `6110 / 5912 / 198`                                                                   |
| included bytes / lines        | `62112421 / 1734908`                                                                  |
| inventory manifest SHA-256    | `bf6b59ccb1f9589bb8a469e155322feff8fae3e037b93f32dff79aaf013b27ba`                    |
| included corpus SHA-256       | `7a45cefd1d93163b2012b67b601730673d8bbc7e0980edc9bbc8ed8cebe3066c`                    |
| content manifest records      | `5912`                                                                                |
| read errors                   | `0`                                                                                   |
| machine analysis groups       | Python、Rust、native registration、CMake、entrypoints、communication                  |
| manual review focus           | lazy import、factory、registry、plugin、MRO injection、IPC、Rust transport、custom op |

结论强度到此为止：本轮对纳入文本完成了确定性全文读取和静态/人工架构复核，但没有宣称所有模型、设备和部署配置均在运行时通过。运行验证与模型评测必须针对具体变更另行执行并记录。

---

上一页：[06：示例学习路线](06-examples-learning-path.md) · 返回：[指南入口](README.md)
