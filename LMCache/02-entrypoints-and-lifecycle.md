# 入口与生命周期

> 分析基准：`389b9cfcb5cec502dbba6b5d13725bf2e024610d`。代码行号与结论均针对该修订；后续提交可能使行号漂移。

> 配套图：[MP 消息流 PNG](diagrams/03-multiprocess-message-flow.png) · [Excalidraw 源文件](diagrams/03-multiprocess-message-flow.excalidraw) · [图索引](diagrams/README.md)

## 安装后的 console scripts

| 命令                   | Python 目标                           | 定位                                                                                                                                                          | 状态                                      |
| ---------------------- | ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| `lmcache`              | `lmcache.cli.main:main`               | [`main`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/cli/main.py#L20) — `lmcache/cli/main.py:20`                                                 | 当前统一入口                              |
| `lmcache_server`       | `lmcache.v1.server.__main__:main`     | [`legacy server main`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/server/__main__.py#L150) — `lmcache/v1/server/__main__.py:150`             | 早期 raw TCP 兼容入口                     |
| `lmcache_controller`   | `lmcache.v1.api_server.__main__:main` | [`legacy controller main`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/api_server/__main__.py#L418) — `lmcache/v1/api_server/__main__.py:418` | 早期 cache orchestration                  |
| `lmcache`（CLI wheel） | 同统一入口                            | [`pyproject_cli.toml`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/pyproject_cli.toml)                                                                   | 轻量远程 CLI；不能替代完整 server runtime |

不要把 `lmcache server` 与 `lmcache_server` 混为一谈：前者启动当前 MP FastAPI+ZMQ 服务，后者运行 [`LMCacheServer`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/server/__main__.py#L24) — `lmcache/v1/server/__main__.py:24` 的 socket 协议。

## CLI 命令发现

[`_discover_commands`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/cli/commands/__init__.py#L15) — `lmcache/cli/commands/__init__.py:15` 扫描直接子模块中的 `BaseCommand` 子类；[`CompositeCommand`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/cli/commands/base.py#L135) — `lmcache/cli/commands/base.py:135` 再扫描自己的 package 形成二级命令。根 dispatcher 只做注册、解析和 `args.func(args)` 调用。

当前源码静态发现 **28** 个 command classes，其中 **12** 个处于顶层发现范围。主要入口包括 `server`、`coordinator`、`ping`、`describe`、`query`、`quota`、`bench`、`trace` 和 `tool`。

## 当前 MP Server 生命周期

```text
lmcache server
  -> ServerCommand.execute
  -> run_http_server / uvicorn
  -> FastAPI lifespan startup
  -> run_cache_server(return_engine=True)
  -> MPCacheServerContext + StorageManager
  -> _build_modules()
  -> MessageQueueServer handlers + normal/affinity pools
  -> HTTP routers + optional plugins/coordinator tasks
```

关键索引：

- CLI：[`ServerCommand`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/cli/commands/server.py#L14) — `lmcache/cli/commands/server.py:14`。
- HTTP lifespan：[`http_server.lifespan`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/http_server.py#L63) — `lmcache/v1/multiprocess/http_server.py:63`。
- compositor：[`MPCacheServer`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/server.py#L63) — `lmcache/v1/multiprocess/server.py:63`。
- module assembly：[`_build_modules`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/server.py#L162) — `lmcache/v1/multiprocess/server.py:162`。
- MQ：[`MessageQueueServer`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/multiprocess/mq.py#L489) — `lmcache/v1/multiprocess/mq.py:489`。

`_build_modules` 总是装配 Lookup、P2PController、Management，再按 `supported_transfer_mode` 装配 `LMCacheDrivenTransferModule`、`EngineDrivenTransferModule` 或两者；`engine_type=blend` 额外装配 `BlendV3Module`。

### MP shutdown 顺序

FastAPI lifespan 先取消 coordinator L2 event/registration tasks，关闭 httpx client 和 runtime plugins，再停止 EventBus 与 ZMQ server。`MPCacheServer.close()` 依 module 顺序关闭并最终关闭共享 context/StorageManager。ManagementModule 被放在 transfer modules 前，确保 worker reaper 先停。

## 进程内 vLLM 生命周期

```text
vLLM loads LMCacheConnectorV1Dynamic
  -> LMCacheConnectorV1Impl
  -> VllmServiceFactory
  -> LMCacheManager.__init__ creates role-specific components
  -> register_kv_caches
  -> LMCacheManager.post_init creates StorageManager
  -> start_services starts API/plugins
  -> scheduler lookup / worker load-store
  -> connector.shutdown -> manager.stop_services
```

- facade：[`LMCacheConnectorV1Dynamic`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/integration/vllm/lmcache_connector_v1.py#L30) — `lmcache/integration/vllm/lmcache_connector_v1.py:30`。
- implementation：[`LMCacheConnectorV1Impl`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/integration/vllm/vllm_v1_adapter.py#L453) — `lmcache/integration/vllm/vllm_v1_adapter.py:453`。
- factory：[`VllmServiceFactory`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/integration/vllm/vllm_service_factory.py#L39) — `lmcache/integration/vllm/vllm_service_factory.py:39`。
- lifecycle owner：[`LMCacheManager`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/manager.py#L40) — `lmcache/v1/manager.py:40`。
- singleton-like engine builder：[`LMCacheEngineBuilder`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/cache_engine.py#L1969) — `lmcache/v1/cache_engine.py:1969`。

Factory 根据 role 创建不同组件：scheduler 主要创建 lookup client；worker 创建 engine、lookup server 和 offload server；DP rank 0 创建 internal API/plugin launcher；health monitor 在 post-init 后启动。

## vLLM MP Connector 生命周期

`LMCacheMPConnector` 同时实现 scheduler 与 worker vLLM KV connector hooks：

| 阶段   | scheduler                                                 | worker                                                                    |
| ------ | --------------------------------------------------------- | ------------------------------------------------------------------------- |
| 构造   | 创建 `LMCacheMPSchedulerAdapter`，连接一个或多个 server。 | 选择本节点 server，创建 `LMCacheMPWorkerAdapter`。                        |
| 注册   | 构建 request trackers。                                   | `register_kv_caches` 发送 IPC wrapper、layout hints、engine group infos。 |
| lookup | `get_num_new_matched_tokens` 发 LOOKUP/查询 prefetch。    | 无。                                                                      |
| load   | 构建 `LMCacheMPConnectorMetadata`。                       | `start_load_kv` 发 RETRIEVE。                                             |
| store  | scheduler 决定 ranges/block ids。                         | `wait_for_save` 发 STORE。                                                |
| 完成   | `request_finished` 清 tracker 并发 END_SESSION。          | `get_finished` 报告异步 transfer 完成。                                   |
| 关闭   | 停 adapter clients/heartbeat。                            | unregister KV context、关闭 transfer/MQ。                                 |

关键 hooks：[`get_num_new_matched_tokens`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/integration/vllm/lmcache_mp_connector.py#L874) — `lmcache/integration/vllm/lmcache_mp_connector.py:874`、[`start_load_kv`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/integration/vllm/lmcache_mp_connector.py#L701) — `lmcache/integration/vllm/lmcache_mp_connector.py:701`、[`wait_for_save`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/integration/vllm/lmcache_mp_connector.py#L775) — `lmcache/integration/vllm/lmcache_mp_connector.py:775`、[`request_finished`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/integration/vllm/lmcache_mp_connector.py#L1061) — `lmcache/integration/vllm/lmcache_mp_connector.py:1061`。

## Coordinator 生命周期

入口 `lmcache coordinator` 与 `python -m lmcache.v1.mp_coordinator` 都构造 [`MPCoordinatorConfig`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/mp_coordinator/config.py#L21) — `lmcache/v1/mp_coordinator/config.py:21`，再调用 [`create_app`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/mp_coordinator/app.py#L64) — `lmcache/v1/mp_coordinator/app.py:64`。

FastAPI lifespan 按配置启动：

- stale-instance health loop；
- L2 quota eviction loop；
- 可选 startup resync；
- shared outbound `httpx.AsyncClient`。

MP server 侧 [`keep_registered`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/mp_coordinator/registrar.py#L75) — `lmcache/v1/mp_coordinator/registrar.py:75` 负责 POST register、PUT heartbeat、404 re-register 与取消时 DELETE。连接是 opt-in、best-effort。

## Standalone 与工具模块入口

`python -m lmcache.v1.standalone` 使用 [`LMCacheStandaloneStarter`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/standalone/__main__.py#L194) — `lmcache/v1/standalone/__main__.py:194` 构造 mock GPU connector、固定 KV tensors、manager/internal API，适合诊断而非 serving engine 集成。

其他 `__main__.py`：

- `python -m lmcache.tools.controller_benchmark` → [`lmcache/tools/controller_benchmark/__main__.py:163`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/tools/controller_benchmark/__main__.py#L163)。
- `python -m lmcache.tools.mp_status_viewer` → [`lmcache/tools/mp_status_viewer/__main__.py:58`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/tools/mp_status_viewer/__main__.py#L58)。
- `python -m lmcache.tools.transfer_channel_benchmark` → [`lmcache/tools/transfer_channel_benchmark/__main__.py:15`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/tools/transfer_channel_benchmark/__main__.py#L15)。
- `python -m lmcache.v1.api_server` → [`lmcache/v1/api_server/__main__.py:418`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/api_server/__main__.py#L418)。
- `python -m lmcache.v1.mp_coordinator` → [`lmcache/v1/mp_coordinator/__main__.py:20`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/mp_coordinator/__main__.py#L20)。
- `python -m lmcache.v1.server` → [`lmcache/v1/server/__main__.py:150`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/server/__main__.py#L150)。
- `python -m lmcache.v1.standalone` → [`lmcache/v1/standalone/__main__.py:501`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/lmcache/v1/standalone/__main__.py#L501)。

## Kubernetes Operator 生命周期

[`operator main`](https://github.com/zhiim/LMCache/blob/v0.5.0-dev/operator/cmd/main.go#L61) — `operator/cmd/main.go:61` 初始化 scheme、controller-runtime manager、三个 reconciler、CacheBlend webhook 与 health/ready checks，然后用 `ctrl.SetupSignalHandler()` 启动。它不直接操作 KV；它通过 Kubernetes API reconcile DaemonSet/Deployment/Service/ConfigMap 等运行时资源。

## 生命周期失败策略

- connector/manager 初始化失败会进入 degraded mode，cache 返回 miss，让 serving engine 重算。
- coordinator 不可达只丢失 fleet 能力，不停止 MP server。
- worker heartbeat 检测 server 恢复后重新注册 KV context。
- shutdown 使用 timeout 防止某个服务永久阻塞整个进程退出。
- 轻量 `lmcache-cli` 虽能发现 `server` 命令，但执行时会检查完整依赖并给出安装提示。
