# 构建、原生扩展与部署

> 分析基准：`389b9cfcb5cec502dbba6b5d13725bf2e024610d`。代码行号与结论均针对该修订；后续提交可能使行号漂移。

> 配套图：[构建与部署 PNG](diagrams/05-build-and-deployment.png) · [Excalidraw 源文件](diagrams/05-build-and-deployment.excalidraw) · [图索引](diagrams/README.md)

## 两个 Python distribution

| Distribution | 元数据 | 依赖 | 产物/用途 |
|---|---|---|---|
| `lmcache` | [`pyproject.toml`](../pyproject.toml) + [`setup.py`](../setup.py) | common + 设备 profile requirements | 完整 runtime、sdist、native wheel |
| `lmcache-cli` | [`pyproject_cli.toml`](../pyproject_cli.toml) | `requirements/cli.txt` | 无 torch/native 的远程 CLI wheel |

两者都由 setuptools-scm 生成版本；完整包通过 [`BuildPolicy`](../setup_extensions/policy.py#L127) — `setup_extensions/policy.py:127` 决定原生扩展。

## BuildPolicy 决策

```text
sdist or NO_NATIVE_EXT=1
  -> no native extensions
else
  -> build 3 common C++ extensions
  -> unless NO_GPU_EXT=1:
       explicit BUILD_WITH_* (must be at most one)
       else compiler auto-detection
       -> one device profile
  -> discover zero or more optional storage backend profiles
```

| 模式 | 开关 | Common C++ | GPU ops | 结果 |
|---|---|---|---|---|
| sdist | `python -m build --sdist` | 否 | 否 | 源码包 |
| source-only | `NO_NATIVE_EXT=1` | 否 | 否 | Python fallback 可用范围 |
| CPU/common | `NO_GPU_EXT=1` | 是 | 否 | native storage ops/Redis/FS；无 `c_ops/xpu_ops` |
| device build | explicit/auto profile | 是 | 是 | common + 一个设备 profile + 可选 storage SDK extensions |

`NO_CUDA_EXT=1` 是历史别名，实际等价 `NO_NATIVE_EXT=1`，并不只是关闭 CUDA。

## 公共 C++ 扩展

| 模块 | 来源 | 用途 |
|---|---|---|
| `lmcache.native_storage_ops` | `csrc/storage_manager/` | Bitmap、fold、TTL lock、PeriodicEventNotifier |
| `lmcache.lmcache_redis` | `csrc/storage_backends/redis/` | Native RESP connector |
| `lmcache.lmcache_fs` | `csrc/storage_backends/fs/` | Native filesystem connector |

定义：[`COMMON_EXTENSIONS`](../setup_extensions/common_cpp.py#L29) — `setup_extensions/common_cpp.py:29`。

## 设备 Profile

| Profile | 显式开关/检测 | 输出 | 关键限制 |
|---|---|---|---|
| CUDA | `BUILD_WITH_CUDA=1` / `nvcc` | `lmcache.c_ops` CUDAExtension | 默认 `LMCACHE_CUDA_MAJOR=13`，也支持 12；ABI 与 torch/vLLM 必须匹配。 |
| ROCm | `BUILD_WITH_HIP=1` / `hipcc` | hipify 后的 `lmcache.c_ops` | torch 从 ROCm index 预装；Docker 路径跳过 flashinfer/NIXL。 |
| SYCL | `BUILD_WITH_SYCL=1` / `icpx` | `lmcache.xpu_ops` SyclExtension | 需 oneAPI；profile 指向的 `requirements/xpu_core.txt` 当前缺失。 |
| MUSA | `BUILD_WITH_MUSA=1`，自动检测恒 false | 无设备扩展 | BuildProfile 是 stub；runtime connector 使用 Python/platform 层。 |

源码：[`CudaProfile`](../setup_extensions/build_profiles/cuda.py#L23) — `setup_extensions/build_profiles/cuda.py:23`、[`RocmProfile`](../setup_extensions/build_profiles/rocm.py#L67) — `setup_extensions/build_profiles/rocm.py:67`、[`SyclProfile`](../setup_extensions/build_profiles/sycl.py#L21) — `setup_extensions/build_profiles/sycl.py:21`、[`MusaProfile`](../setup_extensions/build_profiles/musa.py#L19) — `setup_extensions/build_profiles/musa.py:19`。

### 已发现的构建缺口

[`SyclProfile.requirements_file`](../setup_extensions/build_profiles/sycl.py#L91) — `setup_extensions/build_profiles/sycl.py:91` 返回 `xpu_core.txt`，但该修订的 [`requirements/`](../requirements/) 没有这个文件；[`_read_requirements missing-file branch`](../setup.py#L27) — `setup.py:27` 会静默得到空列表。SYCL 用户需手工准备 XPU torch/runtime/oneAPI 依赖。

## 可选 native storage profile

| Profile | 输出 | 依赖 |
|---|---|---|
| Aerospike | `lmcache.lmcache_aerospike` | libaerospike、OpenSSL、pthread/z/rt、libuv、yaml |
| Mooncake | `lmcache.lmcache_mooncake` | mooncake_store、C++20、可选 include/lib 路径 |

多个 storage profiles 可同时启用；设备 profile 只能选一个。入口：[`collect_storage_backends`](../setup_extensions/policy.py#L232) — `setup_extensions/policy.py:232`。

## Rust raw-block

[`rust/raw_block/pyproject.toml`](../rust/raw_block/pyproject.toml) 是独立 maturin/PyO3 项目，生成 `lmcache_rust_raw_block_io` cdylib。它供 legacy `RustRawBlockBackend` 与 MP `RawBlockL2Adapter` 共用；Python 负责 slot/checkpoint/task orchestration，Rust 只负责 raw I/O。

```bash
cd rust/raw_block
maturin develop --release
```

限制：Linux only；O_DIRECT 要求 offset/size/user buffer 对齐；示例 raw device 操作可能清除目标设备。

## Docker 矩阵

| Dockerfile | 角色 | 原生能力 | 默认启动/限制 |
|---|---|---|---|
| `docker/Dockerfile` | full CUDA LMCache + vLLM | CUDA/common + full dependencies/NIXL | `vllm serve`；大镜像，版本与 torch/vLLM 绑定 |
| `Dockerfile.lightweight` | 快速 released LMCache + vLLM | 使用发布 wheel | 无 NIXL，因此无相应 PD 路径 |
| `Dockerfile.standalone` | 独立 MP server runtime | 构建并安装匹配 torch 的 CUDA wheel | 默认 `CMD /bin/bash`，手工执行 `lmcache server` |
| `Dockerfile.rocm` | full ROCm + vLLM-ROCm | HIP/common | 无 flashinfer/NIXL |
| `Dockerfile.rocm-lightweight` | release-oriented ROCm | clone source 后 HIP build | 无 NIXL/PD，需要 ROCm arch 设置 |

说明入口：[`docker/README.md`](../docker/README.md)。

## Operator 构建与部署

Operator Makefile 分拆为 `tools/dev/unit/build/deploy/lint/e2e/e2e-gpu`，静态扫描到 **38** 个目标。

| 目标 | 结果 |
|---|---|
| `make build` | 先 manifests/generate/fmt/vet，再生成 `operator/bin/manager`。 |
| `make run` | 本地主机运行 controller，`ENABLE_WEBHOOKS=false`。 |
| `make manifests` | controller-gen 生成 CRD/RBAC/webhook。 |
| `make generate` | 生成 DeepCopy。 |
| `make docker-build` / `docker-buildx` | distroless nonroot manager image。 |
| `make build-installer IMG=...` | Kustomize 生成 `dist/install.yaml`。 |
| `make deploy IMG=...` | 部署 `config/default`。 |
| `make test` / `lint` | envtest、coverage、golangci-lint。 |
| `make test-e2e-kind` | no-GPU Kind smoke。 |
| `make test-e2e-gpu-kind` | Kind + NVIDIA GPU Operator + GPU integration。 |

当前 CRD 产物：`LMCacheEngine`/`CacheBlendEngine` DaemonSet 与 `LMCacheCoordinator` Deployment。生成文件不可手工编辑，改 API marker 后必须重跑 manifests/generate。

## CI 与发布链

| 系统 | 主要任务 | 产物 |
|---|---|---|
| GHA main artifacts | source-only sdist + CUDA 13 cibuildwheel | `release-artifacts` |
| GHA CLI artifacts | pure Python、no-torch smoke | `release-cli-artifacts` |
| GHA CPU artifacts | `NO_GPU_EXT=1` common native wheel | `release-cpu-artifacts` |
| GHA cu129 | CUDA 12.9 variant | 独立 GitHub Release，不发 PyPI |
| GHA publish | tests/quality 后发布 | TestPyPI、PyPI、GitHub、DockerHub |
| GHA nightly | cu13/cu129 wheels + full/standalone images | rolling nightly releases/tags |
| GHA Operator | Go CI/release | operator image + `install.yaml` |
| Buildkite | NVIDIA/AMD unit、vLLM、correctness、MP、E2E | logs/coverage/perf artifacts |

发布图的权威文件：[`build_main_artifacts.yml`](../.github/workflows/build_main_artifacts.yml)、[`publish.yml`](../.github/workflows/publish.yml)、[`nightly_build.yml`](../.github/workflows/nightly_build.yml)、[`operator_release.yml`](../.github/workflows/operator_release.yml)。

## 本地验证基线

```bash
# 文档任务最相关
pre-commit run --all-files
cd docs && make clean && make html

# Python 测试主入口（GPU/分布式环境按需裁剪）
pytest -xvs --ignore=tests/disagg   --ignore=tests/v1/multiprocess/   --ignore=tests/v1/distributed/   --ignore=tests/skipped   --ignore=tests/v1/storage_backend/test_eic.py

# Operator
cd operator && make fmt lint test
```
