# 06：示例阅读与实践路线

> **代码快照**：`0934b267906f8cd9459f287b31647c3ed5c58e01`。
> 行号、类型关系和调用链均以此提交为准；返回[指南入口](README.md)。

本篇把 `examples/`、相邻 tests 和 benchmark 组织成一条由浅入深的学习路线。先用最小离线 API 建立输入/输出直觉，再进入在线协议、生成能力、非生成任务、分布式和训练集成。架构名词可先查[项目地图](01-project-map.md)，真实请求链见[02：入口与请求生命周期](02-entrypoints-and-request-lifecycle.md)。

## 使用方式与前提

这些示例是源码快照中的可执行说明，不是统一的无依赖教程。阅读或运行前先确认：

- **模型与网络**：多数示例会从 Hugging Face 或其他地址下载模型/媒体；部分 gated model 需要凭据。
- **设备与容量**：示例可能假定 CUDA/ROCm、特定 GPU architecture、多个 GPU 或多个节点；模型名小不代表所有分支都能在 CPU 上运行。
- **额外依赖**：OpenAI SDK、audio/video、Ray、LangChain、connector、dashboard 等不一定属于基础 runtime dependency。
- **外部服务**：名称带 `online`、`client`、Ray、disaggregated 或 observability 的脚本常要求先启动 server、proxy、storage 或 metrics stack。
- **安全边界**：不要未经审查启用 `trust_remote_code`、开放本地媒体路径、加载陌生 plugin，或把示例开发端点直接暴露到不可信网络。
- **结果可重复性**：sampling、kernel、batching 与硬件会影响输出；应验证结构和不变量，不把示例中的自然语言文本当 golden output。

本仓库要求 Python 命令通过 `uv` 管理的 `.venv/bin/python` 执行；具体环境准备遵循根 [`AGENTS.md`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/AGENTS.md#L27-L74)。本指南未下载模型、启动服务或执行 GPU/Ray benchmark；下面的“预期观察”来自静态源码契约，而非本机运行声明。

建议每站都采用同一阅读方法：

1. 先看文件头的 prerequisites 与默认 model。
2. 标出公开 API 输入、配置对象和输出类型。
3. 沿[集中代码索引](07-source-inventory-and-code-index.md)找到实现入口。
4. 找同主题 nearby test，确认哪些行为有断言。
5. 最后才运行最小 workload，并记录 model、commit、device、参数和环境。

## 第 0 站：先读最小示例

### 0A：`LLM.generate`

从 [`examples/basic/offline_inference/basic.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/basic/offline_inference/basic.py#L1-L34) 开始。它只包含四个概念：

```text
prompts: list[str]
  + SamplingParams
  + LLM(model=...)
  → LLM.generate(...)
  → list[RequestOutput]
      → CompletionOutput.text
```

预期观察：一个 `LLM` 可批量接收多个 prompt；`SamplingParams` 描述生成策略；返回值按 request 聚合，而不是 token iterator。然后回到 [`LLM.generate`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/entrypoints/llm.py#L411-L470)，确认 facade 如何 render、入队并同步 step。

### 0B：`LLM.chat`

接着读 [`examples/basic/offline_inference/chat.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/basic/offline_inference/chat.py#L1-L102)。关注：

- `EngineArgs.add_cli_args()` 把完整 engine 参数接入示例 parser。
- `llm.get_default_sampling_params()` 提供 model generation config 的默认值，CLI 只覆盖显式参数。
- 单个 conversation 与 conversation batch 都进入 `LLM.chat()`。
- chat template 是“消息到 token prompt”的渲染规则，不是 Scheduler 或模型结构。

预期观察：chat 与 completion 在 render 阶段不同，进入 EngineCore 后仍共享同一调度和执行主线。

### 0C：先辨认任务，不一次跑完

`examples/basic/offline_inference/` 还给出 `generate.py`、`embed.py`、`classify.py` 和 `score.py`。先比较调用方法、参数类型与 output class，不必把它们当成同一种生成任务。对应公共方法都由 `LLM` facade 暴露，但 pooling/classification/scoring 不产生逐 token completion。

## 第 1 站：在线服务

先读 client，而不是从 FastAPI 内部开始：[`openai_chat_completion_client.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/basic/online_serving/openai_chat_completion_client.py#L1-L63) 假定已有 `vllm serve`，把 OpenAI SDK 的 `base_url` 指向 `/v1`，先查询 model，再发 chat completion，并在 `stream=True` 时迭代 chunks。

把观察映射到[02 的在线主链](02-entrypoints-and-request-lifecycle.md#在线-python-入口)：

```text
OpenAI client
  → POST /v1/chat/completions
  → FastAPI route
  → OpenAIServingChat
  → AsyncLLM.generate
  → EngineCore/Scheduler/Executor/Worker
  → EngineCoreOutputs
  → OutputProcessor
  → SSE chunk 或完整 response
```

学习时分别记录四类对象：HTTP request schema、renderer 产生的 Engine input、core 使用的 `EngineCoreRequest`、客户端收到的 OpenAI response。它们字段相似但不是同一个类型。

推荐观察顺序：

1. 非流式请求只得到一个完成 response。
2. 流式请求逐 chunk 到达，并以 finish/终止事件结束。
3. 中途取消 client，沿 `AsyncLLM.generate()` 的 `finally` 路径理解 abort。
4. 对无效参数观察 HTTP validation error，区分 request error 与 engine process failure。

不要把 [`examples/applications/api_server/server.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/applications/api_server/server.py#L1-L24) 当生产入口：文件头明确说明它只演示 AsyncEngine 和简单 benchmark，推荐生产路径仍是 OpenAI-compatible server。

## 第 2 站：生成能力

这一站研究“输入如何约束或扩展生成”，每次只选择一个主题。

| 主题                 | 推荐入口                                                                                                                                                                            | 先观察什么                                                                   |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| structured output    | [`structured_outputs_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/features/structured_outputs/structured_outputs_offline.py#L1-L112)                        | choice、regex、JSON schema、grammar 都进入 `StructuredOutputsParams`         |
| reasoning stream     | [`openai_chat_completion_with_reasoning_streaming.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/reasoning/openai_chat_completion_with_reasoning_streaming.py#L1-L70) | `delta.reasoning` 与 `delta.content` 可分阶段出现且都可能为空                |
| tool calling         | [`chat_with_tools_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/tool_calling/chat_with_tools_offline.py#L52-L147)                                            | model 先生成 tool call，应用执行函数并追加 tool message，再次生成            |
| multimodal           | [`vision_language_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/generate/multimodal/vision_language_offline.py#L2660-L2717)                                  | model-specific prompt/processor、`multi_modal_data`、modality limit 和 cache |
| speculative decoding | [`spec_decode_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/features/speculative_decoding/spec_decode_offline.py#L42-L180)                                   | ngram/EAGLE/MTP/draft model 是候选生成策略，最终输出仍由主模型验证           |

### Structured output

离线示例把 Pydantic schema、regex、choice 和 grammar 都映射到 sampling 参数。之后读 online negative test [`test_chat_completion.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/tests/entrypoints/openai/chat_completion/test_chat_completion.py#L31-L118)，观察 invalid JSON schema/regex/grammar 如何被断言为 API error。示例证明“怎么表达”，test 才证明“错误怎么处理”。

### Reasoning 与 tool calling

reasoning parser 和 tool parser 是 model/config-specific frontend 能力。不要假设任意 model 都会产生 `reasoning` 或规范 tool calls。tool calling 示例中的 Python 函数是应用侧模拟工具：vLLM 生成调用描述，但不会自动执行任意业务函数。

### Multimodal

`vision_language_offline.py` 覆盖很多 model，适合作为检索目录，不适合第一份逐行阅读的示例。先选择一个 model handler，追踪 `EngineArgs`、prompt、media object 到 `multi_modal_data`；再观察 `limit_mm_per_prompt`、processor cache 和 media UUID。模型/媒体下载和显存需求以该 handler 注释为准。

## 第 3 站：非生成与高级能力

### Pooling 与 speech

| 任务           | 示例                                                                                                                                                     | 输出关注点                                              |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| embeddings     | [`openai_embedding_client.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/pooling/embed/openai_embedding_client.py#L1-L40)                  | 每个 input 对应 vector，而非 completion text            |
| classification | [`classification_online.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/pooling/classify/classification_online.py#L1-L67)                   | label/probability 与 model task compatibility           |
| scoring/rerank | [`score_api_online.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/pooling/score/score_api_online.py#L1-L69)                                | query-document pair 与 score/rank response              |
| transcription  | [`openai_transcription_client.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/speech_to_text/openai/openai_transcription_client.py#L1-L180) | multipart audio、sync/async/raw stream 三条 client path |

先确认 model 支持的 task，再选 API；不能仅因为 endpoint 存在就让 generation-only model 产生 embedding 或 ASR 输出。

### LoRA、prefix cache 与 observability

[`multilora_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/features/lora/multilora_offline.py#L1-L106) 展示 base requests 与多个 `LoRARequest` 共存，并直接用兼容 `LLMEngine.add_request()/step()` 暴露底层循环。它适合观察 LoRA slot/cache，但新人普通推理仍优先使用 `LLM`。

[`prefix_caching_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/features/automatic_prefix_caching/prefix_caching_offline.py#L1-L98) 用相同 prompt/sampling 对比启用前后输出，并显式要求结果一致。文件也提醒性能测量应转到 `benchmarks/benchmark_prefix_caching.py`；功能示例不是可靠 benchmark。

[`observability/metrics/offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/observability/metrics/offline.py#L1-L49) 展示 `LLM.get_metrics()` 返回 Gauge/Counter/Vector/Histogram。先辨认 metric 类型和单位，再进行跨配置比较，不要从一次打印推断性能趋势。

### KV/EC connector、offloading 与 disaggregation

从最小 [`example_connector/README.md`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/disaggregated/example_connector/README.md#L1-L10) 开始：prefill process 将 KV state 写入 local storage，decode process 再加载；`run.sh` 只是顺序执行两段离线程序。然后再看 LMCache、Mooncake、NIXL、FlexKV、KV failure recovery 和 disaggregated encoder 子目录。

这些示例引入 storage/network/process ownership，不能用单进程 prefix cache 心智模型解释。先画清 prefill、decode、connector 和 proxy 的所有者，再追踪 metadata 与 tensor 数据。

## 第 4 站：分布式与部署

建议按“同机多进程 → external launcher → cluster framework → scale-out protocol”递进：

1. [`data_parallel_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/features/data_parallel/data_parallel_offline.py#L1-L205)：每个 DP rank 分到不同 prompt shard，通过环境变量建立 DP world，并分别创建 `LLM`。
2. [`torchrun_example_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/features/torchrun/torchrun_example_offline.py#L1-L74)：`external_launcher` 让每个 rank 只创建一个 Worker；示例还明确区分 Gloo CPU control group 与 NCCL device data group。
3. [`ray_serve_deepseek.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/ray_serving/ray_serve_deepseek.py#L1-L55)：Ray Serve 负责 autoscaling/load balancing 和 deployment，vLLM engine kwargs 仍描述 TP/PP、cache 与 batching。
4. [`scale_out/example_mm_serve.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/scale_out/example_mm_serve.py#L1-L115)：把 multimodal render 与 generate 拆成两个可组合 endpoint，直接传递 token/features。
5. `examples/disaggregated/`：再进入 prefill/decode、encoder/generator 和 KV connector 的分离部署。

阅读时使用[04 的控制面/数据面划分](04-processes-and-module-communication.md#控制面与数据面先分开)：launcher、Ray actor call、ZMQ、request routing 属控制面；TP/PP/DP/EP collectives、KV/encoder tensors 属数据面。`data_parallel_size=2`、两个 API replicas 和 `tensor_parallel_size=2` 是三种不同扩展维度。

部署示例还包括 Helm、SageMaker 和 dashboard compose。它们展示 integration shape，不自动给出生产安全、容量、升级和故障恢复保证；完整参数与生产要求应回到对应官方文档和组织运维标准。

## 第 5 站：训练/RL/集成

`examples/rl/` 是 serving engine 与训练系统之间的高级集成，不是 vLLM 内置训练框架。典型分工是：

```text
trainer / HF model
  ├─ 负责 optimizer、backward、checkpoint
  └─ 通过 IPC/NCCL/HTTP 推送新权重

vLLM engine
  ├─ 负责 rollout/generation
  ├─ pause/resume 或 sleep/wake
  └─ 接收权重并继续 serving
```

[`rlhf_async_new_apis.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/rl/rlhf_async_new_apis.py#L1-L80) 在不同 GPU 上放置 Ray trainer actor 和 vLLM inference engine，演示 batch invariance、pause 和 NCCL weight transfer。它依赖特定硬件、内部 API 和严格资源隔离，适合在理解 Executor/Worker 后阅读。

[`rlhf_http_nccl.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/rl/rlhf_http_nccl.py#L1-L72) 更清楚地区分 HTTP control plane 与 NCCL weight data plane，并依赖显式 server dev mode。开发端点不是默认公开管理 API，不应直接暴露到生产网络。

其他 integration 示例也应先划边界：

- RAG 示例中，vLLM 提供 embedding/chat endpoints；文档切分、Milvus 和 chain orchestration 属 LangChain 应用（[`LangChain RAG example`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/applications/rag/retrieval_augmented_generation_with_langchain.py#L1-L80)）。
- Ray Serve 管 deployment/autoscaling，vLLM 管单 deployment 内推理。
- weight transfer 改变 serving model state，但 optimizer 与训练正确性仍由 trainer 负责。

## 测试作为行为规格

示例回答“如何组合 API”，tests 回答“哪些结果必须成立”。从示例路径向上找最邻近测试，优先顺序是：

1. **纯 CPU/unit**：参数转换、schema、queue、Scheduler 状态、parser。
2. **组件 test**：AsyncLLM、connector、Worker、custom op 的窄行为。
3. **integration/e2e**：真实 model、HTTP server、multiprocessing/distributed。
4. **eval**：模型输出、准确率或 serving 质量发生变化时的模型级证据。

代表入口：

- [`tests/v1/core/test_scheduler.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/tests/v1/core/test_scheduler.py#L1-L118) 使用 fixture/mock 直接断言 waiting/request/stats 行为，适合学习统一 Scheduler。
- [`tests/v1/engine/test_async_llm.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/tests/v1/engine/test_async_llm.py#L1-L120) 通过 `AsyncLLM.generate()` 观察 delta/final output、并发和取消，但需要 CUDA/model。
- [`test_chat_completion.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/tests/entrypoints/openai/chat_completion/test_chat_completion.py#L1-L118) 启动 remote OpenAI server，以官方 client 验证协议和错误响应。
- [`tests/distributed/test_torchrun_example.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/tests/distributed/test_torchrun_example.py#L1-L82) 是 `torchrun_example_offline.py` 自己指向的分布式行为测试。
- `tests/evals/` 下的 GSM8K、MRCR、gpt-oss suites 用于模型/输出相关验证，不应被普通 unit test 替代。

选择 test 时先问：模块的 I/O contract 是什么、要防止的失败是什么、最低成本哪一层能捕获。只断言 import/wiring 的测试价值通常低于可观察行为断言；需要真实 GPU/model 的 test 也不应伪装成 unit test。

## Benchmark 作为测量工具

当前权威 CLI 是 `vllm bench ...`。根目录旧脚本 `benchmarks/benchmark_latency.py`、`benchmark_throughput.py`、`benchmark_serving.py` 只打印迁移提示并退出（例如 [`benchmark_serving.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/benchmarks/benchmark_serving.py#L1-L17)）。实际实现位于 `vllm/benchmarks/`：

| 工具                    | 实现入口                                                                                                                | 测量对象                                                 |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| `vllm bench latency`    | [`vllm/benchmarks/latency.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/benchmarks/latency.py#L1-L125)       | 固定 batch 的端到端 latency，含 warmup/percentiles       |
| `vllm bench throughput` | [`vllm/benchmarks/throughput.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/benchmarks/throughput.py#L1-L130) | 离线 request/token throughput                            |
| `vllm bench serve`      | [`vllm/benchmarks/serve.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/benchmarks/serve.py#L1-L53)            | 已启动 endpoint 的 online load、TTFT/TPOT/ITL/throughput |
| feature benchmark       | `benchmarks/benchmark_prefix_caching.py` 等                                                                             | cache、structured output、priority 等特性                |
| kernel benchmark        | `benchmarks/kernels/*.py`                                                                                               | 单个 op/kernel 的 shape/dtype/device 性能                |

一个可比较的 benchmark 记录至少应包含：commit、wheel/build variant、model、tokenizer、device、并行配置、input/output length distribution、request rate/concurrency、warmup、sampling 和完整 command。online serve 与 offline throughput 不能直接比较；kernel microbenchmark 也不能代表端到端 serving。

正确性先于性能：先运行对应 targeted test，确认 outputs/误差容限，再 benchmark。一次速度更快但 output、batch shape、cache hit 或请求分布不同，不构成优化证据。

## 五个循序实践任务

### 任务 1：画出最小离线对象流

- **目标**：从 `prompts` 追到 `RequestOutput`。
- **阅读文件**：`basic.py`、`LLM.generate`、`OfflineInferenceMixin._run_completion`。
- **实践**：在不改公共 API 的前提下，把 sampling 设为 deterministic，分别输入一个 prompt 和一个 batch。
- **预期观察**：输出顺序对应输入；每个 request 可有一个或多个 `CompletionOutput`；同步 facade 内部仍逐步驱动 core。

### 任务 2：比较在线 full 与 stream

- **目标**：区分 OpenAI response、SSE chunks 和 core output。
- **阅读文件**：online chat client、chat route、`OpenAIServingChat._create_chat_completion`、`AsyncLLM.generate`。
- **实践**：对同一 deterministic 请求分别使用 `stream=False/True`，记录 chunk 与最终 finish reason。
- **预期观察**：传输粒度不同，但合并后的生成 token/text 应满足同一请求语义；取消 stream 会触发 abort 清理。

### 任务 3：给 structured output 建正反规格

- **目标**：理解约束不是 prompt 文案，而是 sampling/grammar 状态。
- **阅读文件**：structured output 示例、online invalid-schema tests、Scheduler structured-output tests。
- **实践**：选择一个最小 JSON schema，记录合法输出；再提交一个语法错误 schema，观察 request-scoped error。
- **预期观察**：合法输出匹配 schema；无效 constraint 不应杀死 EngineCore。

### 任务 4：验证 prefix cache 的不变量再测量

- **目标**：分开“结果不变”和“性能变化”。
- **阅读文件**：prefix cache 示例、`test_scheduler.py` 的 prefix/cache cases、`benchmark_prefix_caching.py`。
- **实践**：先在 greedy sampling 下比较 cache on/off outputs，再固定 workload 做 warmup 后测 latency/throughput。
- **预期观察**：输出相同；重复 prefix 可产生 cache benefit，但幅度取决于 prompt、batch 和硬件。

### 任务 5：为一个分布式示例画所有权图

- **目标**：不把 launcher、Engine、Worker 和 parallel rank 混为一谈。
- **阅读文件**：torchrun example、`Executor.get_class`、external launcher executor、`parallel_state.py`。
- **实践**：只做静态 dry run：为 TP=2、PP=2 标出四个 launcher ranks、每 rank 的 Worker、control group 与 device group，不启动 cluster。
- **预期观察**：每个外部 rank 创建本地执行对象；control plane 与 tensor data plane 分离；world size 等于 TP×PP（再乘其他配置维度时需重新计算）。

## 按主题的示例索引

下表只选代表入口，不机械重复本快照 `examples/` 下纳入扫描的 256 个文件。完整数量与排除边界见[07：源码清单与代码索引](07-source-inventory-and-code-index.md)。

| 主题                   | 代表文件                                                                                                                                                                               | 阅读提示                                     |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| 最小 generation        | [`basic.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/basic/offline_inference/basic.py#L1-L34)                                                                          | `LLM`、`SamplingParams`、`RequestOutput`     |
| offline chat           | [`chat.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/basic/offline_inference/chat.py#L1-L102)                                                                           | conversation batch 与 chat template          |
| online completion/chat | [`basic/online_serving`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/basic/online_serving/openai_completion_client.py#L1-L48)                                              | OpenAI-compatible client 与 stream           |
| structured output      | [`structured_outputs_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/features/structured_outputs/structured_outputs_offline.py#L1-L112)                           | choice/regex/JSON/grammar                    |
| reasoning              | [`reasoning/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/reasoning/openai_chat_completion_with_reasoning.py#L1-L61)                                                      | parser-specific reasoning/content            |
| tool calling           | [`tool_calling/`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/tool_calling/openai_chat_completion_client_with_tools.py#L1-L160)                                            | tool schema、parser、应用回填                |
| multimodal generation  | [`vision_language_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/generate/multimodal/vision_language_offline.py#L2660-L2717)                                     | model-specific handler 与 media inputs       |
| speculative decoding   | [`spec_decode_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/features/speculative_decoding/spec_decode_offline.py#L42-L180)                                      | proposal method 与主模型验证                 |
| embedding              | [`pooling/embed`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/pooling/embed/openai_embedding_client.py#L1-L40)                                                             | vector output                                |
| classification         | [`pooling/classify`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/pooling/classify/classification_online.py#L1-L67)                                                         | task/model compatibility                     |
| rerank/score           | [`pooling/score`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/pooling/score/rerank_api_online.py#L1-L42)                                                                   | query-document ranking                       |
| token-level pooling    | [`pooling/token_embed`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/pooling/token_embed/multi_vector_retrieval_offline.py#L1-L63)                                          | multi-vector retrieval                       |
| speech-to-text         | [`speech_to_text/openai`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/speech_to_text/openai/openai_transcription_client.py#L1-L180)                                        | audio multipart 与 stream                    |
| LoRA                   | [`multilora_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/features/lora/multilora_offline.py#L1-L106)                                                           | base/adapter request 共存                    |
| prefix cache           | [`prefix_caching_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/features/automatic_prefix_caching/prefix_caching_offline.py#L1-L98)                              | correctness 与 benchmark 分离                |
| structured metrics     | [`observability/metrics/offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/observability/metrics/offline.py#L1-L49)                                                  | metric value types                           |
| DP                     | [`data_parallel_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/features/data_parallel/data_parallel_offline.py#L1-L205)                                          | request sharding 与 DP ranks                 |
| external launcher      | [`torchrun_example_offline.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/features/torchrun/torchrun_example_offline.py#L1-L74)                                          | torchrun + TP/PP                             |
| Ray Serve              | [`ray_serve_deepseek.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/ray_serving/ray_serve_deepseek.py#L1-L55)                                                            | deployment/autoscaling 边界                  |
| disaggregated prefill  | [`example_connector`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/disaggregated/example_connector/README.md#L1-L10)                                                        | KV write/read ownership                      |
| render/generate split  | [`example_mm_serve.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/scale_out/example_mm_serve.py#L1-L115)                                                                 | multimodal preprocessing 与 inference 分离   |
| RL/weight update       | [`rlhf_async_new_apis.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/rl/rlhf_async_new_apis.py#L1-L80)                                                                   | trainer/serving ownership与 NCCL sync        |
| RAG integration        | [`retrieval_augmented_generation_with_langchain.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/applications/rag/retrieval_augmented_generation_with_langchain.py#L1-L80) | embedding/chat service 与应用 orchestration  |
| Helm deployment        | [`chart-helm/README.md`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/examples/deployment/chart-helm/README.md#L1-L33)                                                               | Kubernetes resource template，不替代生产审查 |

---

上一页：[05：构建与原生扩展](05-build-and-native-extensions.md) · 下一页：[07：源码清单与代码索引](07-source-inventory-and-code-index.md) · 返回：[指南入口](README.md)
