# 源码覆盖、交叉验证与当前限制

> 分析基准：`389b9cfcb5cec502dbba6b5d13725bf2e024610d`。代码行号与结论均针对该修订；后续提交可能使行号漂移。

> 配套图均由源码证据生成并完成目视复核：[图索引](diagrams/README.md)。

## 覆盖定义

`git ls-files` 返回的每个路径都在 [`source-inventory.json`](source-inventory.json) 中出现一次。处理规则：

- **full_read**：完整读取文本内容并计算 SHA-256/行数；包括实现、测试、示例、benchmark、文档、构建、CI、容器和部署配置。
- **inventory_only / binary**：读取 bytes 计算 hash，但不把媒体/编译产物当源码解释。
- **inventory_only / translations**：登记 `docs/source/locale` 与 gettext catalog，不用翻译内容推导架构。
- **inventory_only / generated**：登记 Operator generated Go/YAML/PROJECT，以手写 types/builders/markers 为语义来源。
- **symlink**：登记链接，并读取其范围内目标内容。
- 全过程仅静态读取/解析，不 import 项目模块或启动服务。

## 完整读取结果

| 指标 | 值 |
|---|---:|
| Git 跟踪文件 | 1972 |
| full_read | 1779 |
| inventory_only | 193 |
| binary | 22 |
| translation | 164 |
| generated | 7 |
| symlink | 5 |
| bytes | 20,367,813 |
| 文本行 | 459,971 |
| 批次 | 20 × 最多 100 文件 |
| 读取错误 | 0 |
| 解码错误 | 0 |

## 主要语言

| 语言/格式 | 文件数 |
|---|---:|
| Python | 982 |
| YAML | 181 |
| reStructuredText | 167 |
| Gettext PO | 163 |
| Markdown | 142 |
| Shell | 94 |
| Go | 64 |
| Other | 29 |
| C++ | 28 |
| C/C++ header | 24 |
| Text | 20 |
| Extensionless text | 10 |
| Make | 10 |
| JSON | 9 |
| CUDA | 6 |
| Dockerfile | 6 |
| HTML | 6 |
| TOML | 5 |
| CSS | 4 |
| CUDA header | 4 |
| JavaScript | 3 |
| Requirements | 3 |
| CMake | 2 |
| INI | 2 |
| Cargo | 1 |
| Config | 1 |
| Go checksum | 1 |
| Go module | 1 |
| Kubebuilder metadata | 1 |
| Python stub | 1 |
| Rust | 1 |
| Setuptools manifest | 1 |

## 用途覆盖

| 用途 | 文件数（标签可重叠） |
|---|---:|
| runtime_implementation | 607 |
| documentation_reference | 496 |
| tests | 351 |
| ci_automation | 164 |
| kubernetes_operator | 144 |
| examples | 143 |
| container_deployment | 59 |
| native_implementation | 59 |
| build_packaging | 32 |
| benchmarks | 28 |
| repository_metadata | 14 |
| rust_implementation | 5 |
| developer_tools | 2 |


## 派生分析结果

- 静态扫描语言源码：**1207** 文件，错误 **0**。
- 符号：**16,809**；内部依赖边：**3,553**。
- 核心继承节点/边：**259 / 251**。
- 外部入口：packaging scripts **4**，`__main__` **7**，显式 Python guards **68**。
- 运行链：**6** 条，typed protocol contracts **10** 个。
- 通信机制：**9** 类。
- 示例：**143** 文件、**42** 个路线单元，未分配 **0**。
- 交叉验证：confirmed **7**，discrepancies **8**，historical **9**，limitations **13**。

## 已发现的文档/配置差异

以下结论以源码和测试为准：

| ID | 差异 | 对新人指南的处理 |
|---|---|---|
| `protocol-readme-stale-groups` | protocols/README.md 的 Current Protocol Groups 只列 engine/controller/debug 早期集合；源码已注册 blend、blend_v2、blend_v3、observability、p2p，并新增 WAIT/FREE/engine-driven 等请求。 | 指南列出当前 RequestType/协议模块，并把 README 标注为设计机制说明而非完整清单。 |
| `l2-design-old-prefetch-handle` | L2 overall design 仍展示 PrefetchHandle.request_id/l1_prefix_hit_count 和 query_prefetch_status 返回 int；源码现在使用 prefetch_request_id、l1_found_indices、l2_orig_indices，并返回 Bitmap。 | 指南采用当前 dataclass 和 Bitmap API，前缀命中通过 count_leading_ones 派生。 |
| `l2-design-prefix-not-universal` | L2 design 把 contiguous prefix 描述为全局 invariant；当前 TrimPolicy.PREFIX 仍如此，但 SEGMENTED_PREFIX/SPARSE 会保留 gap-tolerant 位图。 | 指南说明 PREFIX 是默认策略，非 PREFIX policy 保留所有 set bit。 |
| `coordinator-readme-backbone-only` | mp_coordinator README 开头声称 quota reconcile 和 blend lookup 尚未实现；当前 app 已组装 QuotaManager、L2 usage/eviction/resync 与 GlobalBlendMatcher，且 REST API/测试齐全。 | 指南把 membership、quota/usage/eviction/resync、blend directory 分成当前子系统。 |
| `observability-doc-old-emitter-path` | request-event-span design 的 Implementation 表仍说四个事件在 multiprocess/server.py 发出；模块化重构后 REQUEST/LOOKUP/END 在 modules/lookup.py，STORE/RETRIEVE SUBMITTED/START/END 在 modules/lmcache_driven_transfer.py。 | 指南保留 deferral 语义，更新到当前模块/符号索引。 |
| `multi-hardware-doc-omits-musa` | ARCHITECTURE_MULTI_HARDWARE 图和 connector routing 仅展开 CUDA/XPU/HPU；源码已把 MUSA 放在检测优先级首位并提供 vLLM MUSA connector。 | 指南用支持矩阵分别表示 runtime connector、原生 extension build 和功能限制。 |
| `gemma3-mamba-doc-conflict` | Gemma 3 recipe 仍广义声称 Mamba/linear-attention hybrid 不支持，而当前 hybrid_models 与 Qwen3.5 recipe 已支持 align 模式的 Mamba/GDN opaque-page 路径。 | 指南避免全局 yes/no，列出已验证 Qwen3.5/Qwen3.6 路径及其约束。 |
| `sycl-requirements-gap` | SyclProfile.requirements_file() 返回 xpu_core.txt，但基准修订的 requirements/ 中没有该文件；setup.py 对缺失文件静默返回空依赖。 | 构建指南明确要求人工准备 oneAPI、XPU torch/runtime，并标注缺失文件。 |

## 历史与兼容路径

| ID | 区分 |
|---|---|
| `server-entrypoint-split` | `lmcache server` 是当前 FastAPI+ZMQ MP server；`lmcache_server` console script 指向早期 raw TCP LMCacheServer。 |
| `controller-coordinator-split` | `lmcache_controller`/cache_controller 是早期缓存编排面；`lmcache coordinator`/mp_coordinator 是当前 fleet membership、L2 quota/eviction 和 blend directory。 |
| `storage-manager-split` | `lmcache/v1/storage_backend/StorageManager` 服务进程内 LMCacheEngine；`lmcache/v1/distributed/StorageManager` 服务当前 MP L1/L2 controller 架构。 |
| `versioned-vllm-shims` | 非版本后缀 connector 是当前入口；_0180、_0201、_085 文件保留特定 vLLM API 兼容。 |
| `cacheblend-generations` | examples/blend_kv 是旧 lmcache_vllm/CacheBlend V0，blend_kv_v1 是需要补丁的实验路径，当前 MP server 的 `engine_type=blend` 组装 BlendV3Module。 |
| `raw-manifest-vs-operator` | examples/multi_process raw YAML 使用 hostNetwork 和 hostPath /dev/shm；当前 Operator 选择 HostIPC、非 hostNetwork、node-local Service，且不额外挂载 /dev/shm。 |
| `pd-two-paths` | examples/disagg_prefill 是直接 NIXL/PD backend 路径；examples/disagg_prefill_mp 是经独立 MP server 的推荐入门路径。 |
| `native-disable-env-alias` | NO_NATIVE_EXT 是当前关闭全部原生扩展的开关；NO_CUDA_EXT 是历史别名，实际也关闭全部原生扩展；NO_GPU_EXT 只关闭设备扩展。 |
| `storage-interface-alias` | ConfigurableStorageBackendInterface 只是 StoragePluginInterface 的兼容别名。 |

## 当前明确限制

| ID | 限制 | 影响 |
|---|---|---|
| `mq-nonblocking-unimplemented` | HandlerType.NON_BLOCKING 已在枚举中预留，但 MessageQueueServer 注册和派发均直接抛 NotImplementedError。 | 协议只能选择 SYNC 或 BLOCKING；异步状态机必须拆成 submit/query 请求。 |
| `multi-server-dp-pp` | LMCacheMPConnector 多 server 模式不支持 DP，也拒绝 PP>1，仅支持相应 TP 分片。 | 多节点拓扑必须按 world_size/server 数整除并避免 DP/PP 组合。 |
| `engine-driven-no-hma` | engine-driven 非 GPU transfer path 不支持 hybrid KV cache groups，store/retrieve 会拒绝 multi-group。 | HMA 模型目前需要可用的 LMCache-driven/device IPC 路径。 |
| `musa-build-and-features` | MUSA 有 vLLM runtime connector，但 BuildProfile 是 stub；SGLang MUSA、blending 和 connector v3 均被拒绝，native transfer 仍是 opt-in。 | “支持 MUSA”只表示受限 runtime 路径，不表示完整 CUDA 等价能力或可发布原生 wheel。 |
| `hpu-feature-subset` | HPU 只有非 layerwise VLLMPagedMemHPUConnectorV2，blending/v3 等 device-scoped 功能也不在支持集合。 | HPU 配置必须关闭相应功能。 |
| `cpu-stub-not-connector` | CPU fallback 是中间层 API stub/engine-driven 支持，不是 vLLM GPUConnector；CreateGPUConnector 的 VLLM CPU 分支会报 no supported connector。 | CLI/部分 MP smoke 可在 CPU 上运行，不代表完整 CPU serving connector 可用。 |
| `p2p-peer-l1-only` | P2P peer handler 强制 skip_l2=True，只共享已驻留 peer L1 的连续前缀，不会级联查询对方 L2。 | peer L2 中存在对象仍可能在 P2P 查询中 miss。 |
| `operator-single-l2` | MP runtime 的 L2AdaptersConfig 支持重复 --l2-adapter 和动态多个 adapter；Operator CRD 当前只暴露单个 L2 backend。 | 使用 Operator 时无法声明 runtime 的完整多 L2 cascade。 |
| `raw-block-platform` | Rust raw-block 仅支持 Linux，O_DIRECT 要求 offset/size/buffer 对齐；示例直接使用块设备具有破坏性。 | 不能在普通文件系统/任意缓冲区或非 Linux 环境无条件启用。 |
| `disagg-mp-example-scope` | disagg_prefill_mp 示例说明当前只支持 1P1D，且未做性能优化。 | 它适合学习拓扑，不应被当作生产性能结论。 |
| `cacheblend-version-coupling` | CacheBlend V1 示例需要特定 vLLM patch；MP Blend V3/Operator webhook 也要求 engine、plugin 和 vLLM 位于兼容窗口。 | 普通 prefix-cache 配置不能直接切换到 CacheBlend。 |
| `slim-cli-server` | lmcache-cli 轻量包会注册 server 命令外壳，但缺少完整依赖时 execute 明确退出并要求安装 lmcache。 | 远程 CLI wheel 不等于可运行 MP server 的 runtime wheel。 |
| `mamba-validated-subset` | Mamba/GDN 支持要求 align mode、模型特定统一 block size，并把 cache page 当作 byte-opaque；上游 prefix caching 仍标 experimental，文本路径才是已验证范围。 | 不能把支持扩展为任意 Mamba mode、多模态内容处理或跨不同 kernel/block-size engine 共享。 |

## 最终验证结果

| 检查 | 结果 |
|---|---|
| 范围复核 | HEAD 仍为完整基准提交；重新枚举 **1972** 个跟踪路径，无新增、遗漏、重复或删除。 |
| 全量重读 | **1972/1972** 文件可读，UTF-8 解码错误 **0**，与首次清单的 SHA-256 差异 **0**。 |
| JSON 与图 | `source-inventory.json` 和 5 个 Excalidraw 文件均可解析；5 张 PNG 均重新渲染并完成目视复核。 |
| 链接与锚点 | **349** 个本地链接、**233** 个源码行锚点和 1 个 Markdown 标题锚点均有效；抽查的 **204** 个符号链接均在目标行附近出现对应符号。 |
| 静态检查 | `git diff --check` 通过；仓库 `pre-commit run --all-files` 的 10 个 hook 全部通过，新指南显式执行 codespell 通过。 |
| Sphinx | `make clean && make html` 构建成功；`guides/` 独立于 Sphinx toctree，没有引入 warning。基准 `docs/source/` 自身仍报告 24 个既有 warning，本次未修改这些跟踪文档。 |
| 安全与范围 | 最终工作树只有本目录下 21 个新增产物；没有源码改动、秘密、个人目录、临时目录、凭据 URL 或 PNG 文本元数据。 |

剩余限制分为两类：上表“当前明确限制”是基准实现本身的能力边界；Sphinx 的既有 warning 与外部依赖源码未纳入分析，是本指南验证环境和仓库范围的边界。提交变化后必须重新生成清单、重验行锚点，并重新审视所有“当前”结论。

## 如何复核

1. 在 `source-inventory.json` 中按 path 查找 `read_status`、`line_count`、`sha256`。
2. 用本指南中的 `path:line` 在基准提交定位符号。
3. 对行为结论优先打开相邻测试；测试索引见 [07-key-code-index.md](07-key-code-index.md)。
4. 如果仓库提交变化，重新生成 file set/hash，再接受行号或结论。

## 未纳入架构语义的内容

- 图片、字体等二进制资产只做 hash/inventory。
- 中文翻译 catalog 不作为英文 canonical docs 的替代来源。
- generated Operator CRD/RBAC/DeepCopy 只用来确认产物存在；行为解释来自 API types、resource builders 和 tests。
- 外部服务（vLLM、NIXL、Mooncake、Redis、Kubernetes 等）的实现源码不在本仓库，指南只描述 LMCache 侧契约。
