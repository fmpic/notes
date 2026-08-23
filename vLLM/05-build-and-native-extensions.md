# 05：Python、CMake、Cargo 构建与原生扩展

> **代码快照**：`0934b267906f8cd9459f287b31647c3ed5c58e01`。
> 行号、类型关系和调用链均以此提交为准；返回[指南入口](README.md)。

![vLLM 构建链图](diagrams/build-pipeline.png)

本篇从最终 wheel 逆向解释 Python、CMake 与 Cargo 如何汇聚。重点是“哪个输入决定哪个产物”和“运行时如何找到已编译实现”，而不是复述平台安装教程。运行时的进程边界见[04：进程与通信](04-processes-and-module-communication.md)。

## 从 wheel 反看构建产物

一个 vLLM wheel 不是单一编译器的产物，而是 setuptools 汇总出的部署包：

| wheel 中的内容             | 代表路径/名称                                                       | 构建者                                          | 运行时角色                                               |
| -------------------------- | ------------------------------------------------------------------- | ----------------------------------------------- | -------------------------------------------------------- |
| Python package             | `vllm/**/*.py`                                                      | setuptools package discovery                    | API、Engine、Scheduler、Worker、Model、Python op wrapper |
| C++/CUDA/HIP/CPU extension | `vllm/_C*.so`、`cumem_allocator*.so`、FlashAttention/FlashMLA `.so` | CMake + C++/NVCC/HIPCC                          | import 时注册 `torch.ops.*` 或提供 Python module API     |
| Rust executable            | `vllm/vllm-rs`                                                      | Cargo，经 setuptools-rust `Binding.Exec`        | 实验性 Rust frontend/CLI executable                      |
| Rust PyO3 module           | `vllm/_rust_tool_parser*.so`                                        | Cargo + PyO3，经 setuptools-rust `Binding.PyO3` | Python 可导入的 Rust tool parser                         |
| vendored/JIT package       | `vllm/third_party/deep_gemm`、`fmha_sm100`、`triton_kernels` 等     | CMake external project/copy step                | Python、Triton 或运行时 JIT 所需源码与资源               |
| package data               | kernel configs、静态 JS/CSS、headers、`libs/*.so*`                  | `setup.py` 清单与构建后复制                     | 非 Python import 资源、CPU tcmalloc、JIT headers         |
| distribution metadata      | version、dependencies、console script、entry points                 | `pyproject.toml` + `setup.py` + setuptools-scm  | 安装器解析、`vllm` 命令、plugin discovery                |

[`package_data` 与最终 `setup()`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L1182-L1304) 展示了汇聚点：`ext_modules`、`rust_extensions`、动态 runtime requirements 和 package data 最终进入同一个 Python distribution。

这里有两个容易混淆的命名事实：

1. shared library 文件名不等于 torch operator namespace。例如 `vllm/_C_stable_libtorch*.so` 注册的主 namespace 仍是 `torch.ops._C`。
2. `vllm-rs` 是 executable；`_rust_tool_parser*.so` 是 PyO3 extension。两者都由 Cargo 构建，但不是同一类 artifact，也不能用 `import vllm.vllm-rs` 互换。

## PEP 517 与 setuptools 入口

### 构建前端到后端

PEP 517 frontend 读取 [`pyproject.toml` build-system](https://github.com/zhiim/vllm/blob/v0.26.0-dev/pyproject.toml#L1-L15)，准备包含 CMake、Ninja、setuptools、setuptools-scm、setuptools-rust、PyTorch 和 Jinja2 的 build environment，再调用 `setuptools.build_meta`。静态 project metadata、Python 版本范围、console script 和 package discovery 也在同一文件中（[`pyproject.toml project metadata`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/pyproject.toml#L17-L53)）。

真正的动态构建决策在 `setup.py`：

```text
PEP 517 frontend
  → setuptools.build_meta
    → setup.py
      ├─ 从 vllm/envs.py 读取 installation-time env
      ├─ 检测/选择 VLLM_TARGET_DEVICE
      ├─ get_vllm_version() + get_requirements()
      ├─ 组装 CMakeExtension[]
      ├─ 组装 RustExtension[]
      └─ setup(...)
          ├─ build_ext → cmake_build_ext / precompiled_build_ext
          └─ build_rust → setuptools-rust / precompiled_build_rust
```

`setup.py` 不能正常 `import vllm.envs`，因为 package 尚未安装，所以按文件路径加载 `vllm/envs.py` 和 `tools/build_rust.py`（[`setup.py build inputs`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L20-L54)）。这也意味着这两个文件是 packaging bootstrap 输入，不只是运行时代码。

### 设备选择

初始目标来自 `VLLM_TARGET_DEVICE`。Linux 未显式设置时，`setup.py` 按已安装 PyTorch 的 HIP、XPU、CUDA 信息自动检测，否则落到 CPU；macOS 非 CPU 目标被改为 CPU（[`setup.py target detection`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L77-L103)）。随后 `_is_cuda()` 还要求 `torch.version.cuda` 存在，ROCm/XPU/CPU/TPU 也有独立 predicate（[`setup.py device predicates`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L930-L965)）。

因此，设备目标不是只由机器是否有 GPU 决定：PEP 517 build environment 中的 PyTorch variant、显式环境变量和 host platform 一起参与判定。诊断“为什么没生成某个 `.so`”时，应先记录这三项，而不是直接从 kernel source 开始查。

### setuptools 到 CMake

每个 `CMakeExtension` 在 setuptools 看来只有 logical name，没有 source list；source、flags 和 architectures 由 CMake target 管理（[`CMakeExtension`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L184-L189)）。`cmake_build_ext.configure()` 把 build type、target device、Python executable/path、FetchContent cache、compiler cache、并行度、NVCC threads 与用户 `CMAKE_ARGS` 传给 CMake（[`cmake_build_ext.configure`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L239-L322)）。

一次 configure 可服务多个 extension。`build_extensions()` 将 Python extension 名去掉 package prefix，执行选定 CMake targets，再逐 target 运行同名 install component，把 library 放进 setuptools 期望的 package 目录（[`cmake_build_ext.build_extensions`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L324-L386)）。所以 `ext_modules` 不是重复配置多套 CMake project，而是声明本次 wheel 要 build/install 哪些 components。

## 依赖文件分层

vLLM 没有把“构建工具、wheel 运行依赖、开发工具和测试闭包”写入同一文件。

| 层                | 主要文件                                                                        | 谁消费                                | 是否进入 wheel metadata        |
| ----------------- | ------------------------------------------------------------------------------- | ------------------------------------- | ------------------------------ |
| PEP 517 bootstrap | `pyproject.toml [build-system]`                                                 | build frontend                        | 否，只用于 build env           |
| platform runtime  | `requirements/common.txt` + `cuda.txt`/`rocm.txt`/`cpu.txt`/`tpu.txt`/`xpu.txt` | `setup.py get_requirements()`、镜像   | 是，经 `install_requires` 展开 |
| native build      | `requirements/build/cuda.txt`、`rocm.txt`、`cpu.txt`                            | Docker/CI/developer build env         | 否                             |
| Rust build        | `requirements/build/rust.txt` + `rust-toolchain.toml` + Cargo manifests/lock    | standalone Rust stage/setuptools-rust | 否；Rust artifact 进入 wheel   |
| test input        | `requirements/test/*.in`                                                        | 人工维护的顶层 test intent            | 否                             |
| test lock         | `requirements/test/*.txt`                                                       | CI/test env                           | 否；由 `uv pip compile` 生成   |
| docs input/lock   | `requirements/docs.in` → `requirements/docs.txt`                                | docs build                            | 否                             |
| lint              | `requirements/lint.txt`                                                         | pre-commit/lint env                   | 否                             |
| aggregate dev     | `requirements/dev.txt`                                                          | developer env                         | 否；只引用 lint 与 test lock   |

`get_requirements()` 递归展开 `-r`，去掉 comment/index option，再按目标设备选择 runtime file；CUDA 还会按 CUDA major 排除或改写少数依赖（[`get_requirements`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L1061-L1113)）。例如 [`requirements/cuda.txt`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/requirements/cuda.txt#L1-L36) 先引用 common，再声明 CUDA runtime packages；这和构建 CMake 所需的 [`requirements/build/cuda.txt`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/requirements/build/cuda.txt#L1-L13) 是两套职责。

生成文件和输入文件也必须区分：[`requirements/test/cuda.txt`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/requirements/test/cuda.txt#L1-L3) 的文件头记录了完整 `uv pip compile` 命令，并以 `requirements/test/cuda.in` 和 platform constraints 为输入；[`requirements/docs.txt`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/requirements/docs.txt#L1-L3) 同理由 `docs.in` 生成。需要加顶层测试/文档依赖时改 `.in` 再重编译，而不是手改解析后的 `.txt`。`requirements/dev.txt` 明确只聚合 lint 和 CUDA test lock（[`requirements/dev.txt`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/requirements/dev.txt#L1-L5)）。

几个一致性约束值得在 review 中检查：

- `pyproject.toml [build-system].requires` 注释要求与 `requirements/build/cuda.txt` 镜像维护。
- PyTorch、torchvision、torchaudio 与支持的 CUDA/ROCm 版本散布在 build/runtime/CMake/Docker 输入中，升级必须成组检查。
- Cargo dependencies 由 `rust/Cargo.toml` 与 `rust/Cargo.lock` 管理，不应复制进 Python requirements。
- test/docs/lint dependencies 不应因为 CI 需要而进入 `install_requires`。

## CMake 调度总览

根 CMake project 以 C++20 启动，要求明确的 Python executable，通过 PyTorch 的 CMake prefix 找到 Torch，并按 `VLLM_TARGET_DEVICE` 进入 GPU 或 CPU 分支（[`CMake project 与 Torch discovery`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/CMakeLists.txt#L1-L121)）。

整体调度可以压缩为：

```text
setup.py ext_modules
  → cmake configure once
    → find exact Python + Torch
    → select CXX / CUDA / HIP
    → normalize target architectures and compiler flags
    → FetchContent / external project declarations
    → define_extension_target(name, language, sources, arches, ABI, libs)
  → cmake --build --target=<selected names>
  → cmake --install --component=<same target>
  → setuptools wheel staging directory
```

`define_extension_target()` 是统一 target adapter。它负责：

- HIP source 先接入 shared `hipify_all`。
- 按 Python free-threaded 状态选择普通 module ABI 或 `USE_SABI`。
- 绑定 source、architecture、compile flags、include directories 与 libraries。
- 定义 `TORCH_EXTENSION_NAME`。
- 对 CUDA 精简 link set，对其他语言使用 Torch libraries。
- 创建与 target 同名的 install component。

源码见 [`define_extension_target`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/cmake/utils.cmake#L555-L640)。这解释了为什么 Python logical name、CMake target、install component 通常同名；它们是同一构建节点在三层工具里的名字。

外部项目不全是 `.so`：GPU common 分支包含 `triton_kernels`，CUDA 另包含 DeepGEMM、FMHA SM100、FlashMLA、QuTLASS、TML FA4 与 vLLM FlashAttention（[`external project includes`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/CMakeLists.txt#L1405-L1420)）。有的 target 编译 extension，有的复制 Python/CuTe DSL package 或保留 JIT headers。判断产物类型应读对应 `cmake/external_projects/*.cmake`，不能仅凭它出现在 `ext_modules` 就断言会生成共享库。

## CUDA、ROCm 与 CPU 分支

### 共同 target

Python 3.11+ 的 `spinloop` 和 `fs_io_C` 是纯 CXX target，定义在非 CUDA early-return 之前，因此 CPU 和 GPU 构建都可包含。CUDA/ROCm 还共同生成 `cumem_allocator`（[`common CXX targets`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/CMakeLists.txt#L135-L181)、[`cumem_allocator target`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/CMakeLists.txt#L319-L359)）。

### CUDA

CUDA 构建从 Torch/NVCC flags 解析 architecture，再针对不同 kernel source 收窄 SM 列表。主 custom-op libraries 是：

- `_C_stable_libtorch`：`_C` namespace 的主 CUDA ops。
- `_moe_C_stable_libtorch`：`_moe_C` namespace 的 MoE ops。
- 可选 `_qutlass_C`、FlashMLA、FlashAttention、DeepGEMM 等 target。

`_C_stable_libtorch` 在 CMake 中设置 `TORCH_TARGET_VERSION=2.11` 的 stable C-shim compatibility floor，并以 `USE_CUDA` 编译（[`_C_stable_libtorch target`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/CMakeLists.txt#L1087-L1123)）。MoE 被拆成独立 extension，拥有自己的 source list与相同 stable target version（[`_moe_C_stable_libtorch target`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/CMakeLists.txt#L1316-L1350)）。

当前 GPU legacy `_C` target 只在 HIP 分支定义；CUDA 主路径已迁往 stable-libtorch extension（[`legacy _C branch`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/CMakeLists.txt#L361-L388)）。因此看到 Python 调用 `torch.ops._C.foo` 时，不应直接假设它来自名为 `_C.so` 的文件。

### ROCm

ROCm 将 GPU language 设为 HIP，复用大量 `.cu` source。`hipify_sources_target()` 将 CUDA source 映射到 build tree 中的 `.hip` 路径；所有 extension 的 source 先汇总，再由一个 shared `hipify_all` target 统一生成，避免并行 hipify 相互覆盖（[`HIP source aggregation`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/cmake/utils.cmake#L46-L123)）。

ROCm 同时可包含 legacy `_C`、stable `_C`/`_moe_C` 和 ROCm-specific `_rocm_C`；后者按 gfx architecture 条件加入 RDNA3 sources（[`ROCm-specific target`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/CMakeLists.txt#L1362-L1402)）。stable extensions 还优先链接 PyTorch wheel 自带的 `libamdhip64`，避免同一进程初始化两份 HIP runtime。

### CPU

CPU 不启用 GPU language，而由 [`cmake/cpu_extension.cmake`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/cmake/cpu_extension.cmake#L1-L138) 检测 architecture、OpenMP、NUMA、BF16/ISA flags 和 cross-compile overrides。x86_64 构建定义三种 library：

| target 文件 | 编译能力                    | runtime 选择                             |
| ----------- | --------------------------- | ---------------------------------------- |
| `_C`        | AVX512 + BF16/VNNI/AMX 路径 | CPU 支持 AVX512 BF16 时                  |
| `_C_AVX512` | AVX512 路径                 | 支持 AVX512、但不满足 BF16 主 variant 时 |
| `_C_AVX2`   | AVX2 fallback               | 不支持 AVX512 时                         |

定义见 [`CPU ISA extension targets`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/cmake/cpu_extension.cmake#L480-L574)，运行时 selection 在 [`CpuPlatform.import_kernels`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/platforms/cpu.py#L427-L459)。非 x86 architecture 生成单一 `_C` target。CPU wheel 还可把系统 tcmalloc 复制到 `vllm/libs`，供运行时优化 out-of-box allocator 行为（[`tcmalloc bundling`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L112-L169)、[`cmake_build_ext.run`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L388-L459)）。

## PyTorch custom op 注册链

一个 native op 从 Python 到 kernel 的完整链是：

```text
import vllm._custom_ops
  → current_platform.import_kernels()
    → import vllm._C_stable_libtorch / _C / _rocm_C / ...
      → dynamic loader 执行 shared library 初始化
        → TORCH_LIBRARY* / STABLE_TORCH_LIBRARY* 注册 schema 与 dispatch impl
          → torch.ops._C.<op> / torch.ops._moe_C.<op>
            → vllm._custom_ops Python wrapper
              → C++/CUDA/HIP/CPU kernel
```

`vllm._custom_ops` 在 import 顶部调用 `current_platform.import_kernels()`（[`Python custom-op bootstrap`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/_custom_ops.py#L1-L31)）。CUDA platform 导入 `_C_stable_libtorch`，并容忍部分可选 extension 缺失（[`CudaPlatform.import_kernels`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/platforms/cuda.py#L220-L230)）；CPU 则按 host ISA 导入三个 `_C` variants 之一。

shared library 之所以可被 Python import，是因为 `REGISTER_EXTENSION(NAME)` 定义最小 `PyInit_<NAME>`；真正的算子表由 Torch registration macro 建立（[`registration helpers`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/csrc/core/registration.h#L1-L28)）。两条主要注册路线是：

- 普通 libtorch：CPU `TORCH_LIBRARY_EXPAND(TORCH_EXTENSION_NAME, ops)` 使用 CMake 注入的 target 名作为 namespace（[`CPU op registration`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/csrc/cpu/torch_bindings.cpp#L309-L339)）。
- stable libtorch：`STABLE_TORCH_LIBRARY_FRAGMENT(_C, ops)` 明确固定 namespace `_C`，随后用 dispatch-key-specific `STABLE_TORCH_LIBRARY_IMPL` 绑定实现（[`stable op schema registration`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/csrc/libtorch_stable/torch_bindings.cpp#L1-L24)、[`stable CUDA/CPU implementations`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/csrc/libtorch_stable/torch_bindings.cpp#L616-L764)）。

Python wrapper 分配输出、做 platform guard/shape conversion，再调用 `torch.ops`。例如 `silu_and_mul_per_block_quant()` 最终调用 `torch.ops._C.silu_and_mul_per_block_quant`（[`wrapper call`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/_custom_ops.py#L439-L484)）。`register_fake`/abstract implementation 则让 fake tensor、shape propagation 和 `torch.compile` 看见输出契约；它不是实际 GPU kernel。

所以定位 custom op 时应同时回答四个问题：wrapper 调哪个 namespace/name、schema 在哪里注册、dispatch implementation 在哪个 translation unit、该 source 属于哪个 CMake target。

## Rust 构建链

Rust 与 torch extension 是并列构建链。根 [`rust/Cargo.toml`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/Cargo.toml#L1-L31) 定义 workspace members，集中管理 dependencies、edition 和 build profiles；`rust/Cargo.lock` 固定解析结果，`rust-toolchain.toml` 固定 Rust channel（[`rust-toolchain.toml`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust-toolchain.toml#L1-L2)）。

`tools/build_rust.py` 为 Python packaging 声明两个 `RustExtension`：

| setuptools-rust target   | Cargo manifest/artifact                         | binding        | wheel 目标                |
| ------------------------ | ----------------------------------------------- | -------------- | ------------------------- |
| `vllm.vllm-rs`           | `rust/src/cmd/Cargo.toml` 的 `[[bin]] vllm-rs`  | `Binding.Exec` | `vllm/vllm-rs` executable |
| `vllm._rust_tool_parser` | `rust/src/parser/python/Cargo.toml` 的 `cdylib` | `Binding.PyO3` | `_rust_tool_parser*.so`   |

[`rust_extensions()`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/tools/build_rust.py#L15-L38) 是 setuptools 和 standalone build 共用的 artifact declaration。command crate 聚合 chat、server、managed engine、EngineCore client 等 workspace crates（[`vllm-cmd manifest`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/cmd/Cargo.toml#L1-L39)）；parser Python crate 只暴露 PyO3 module（[`parser PyO3 manifest`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/src/parser/python/Cargo.toml#L1-L19)）。

`build_rust.sh` 读取固定 toolchain、确保 rustup toolchain 存在，再调用 `tools/build_rust.py` 的 release/debug 模式（[`build_rust.sh`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/build_rust.sh#L1-L33)）。后者运行一个最小 setuptools `build_rust --inplace`，把 artifacts 直接发布到 `vllm/`（[`standalone Rust build`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/tools/build_rust.py#L52-L68)）。Docker 可在独立 stage 先完成这一步，然后把两类 artifact 复制进最终 wheel build stage。

默认 `setup.py` 把 Rust extensions 标为 optional；`VLLM_REQUIRE_RUST_FRONTEND` 才将缺失视为强制 build failure。optional 只说明 packaging failure policy，不说明 Rust frontend 已成为 Python API server 的默认替代品。

## 预编译 wheel 与源码构建

### 源码构建

普通路径由本地 toolchain 编译 CMake targets 和 Rust extensions：

```text
当前 source tree + build requirements + torch/toolchain
  → local CMake/NVCC/HIPCC/CXX
  → local Cargo/rustc
  → 当前 source tree 对应的完整 wheel
```

version 由 setuptools-scm 写入 `vllm/_version.py`，并根据非主 CUDA、ROCm、CPU、XPU、TPU 或 precompiled variant 添加 local suffix（[`get_vllm_version`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L1017-L1058)）。这类 suffix 描述 wheel build variant，不是 runtime 自动协商协议。

### 预编译复用

`VLLM_USE_PRECOMPILED=1` 表示复用 native extensions，并隐含复用 Rust artifacts；`VLLM_USE_PRECOMPILED_RUST=1` 则只要求 Rust 预编译路径。来源可由 local/remote wheel location、variant 和 commit 环境变量确定（[`precompiled mode flags`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L35-L74)、[`determine_wheel_url`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L638-L749)）。

预编译路径不是“安装另一个 wheel 作为依赖”，而是：

1. 下载或打开已有 wheel。
2. 从 allowlist 提取 `.so`、`vllm-rs`、`_rust_*.so` 和相关 vendored files 到当前 source tree。
3. patch 当前 `package_data`。
4. `precompiled_build_ext` 跳过 CMake。
5. 若 Rust artifacts 完整，`precompiled_build_rust` 跳过 Cargo；不完整则 fallback 到 local Rust build。
6. 用当前 Python source 和提取的 native artifacts 重新打包。

提取 allowlist 和 package patch 见 [`extract_precompiled_and_patch_package`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L748-L871)，build command class 选择见 [`precompiled command selection`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L1210-L1257)。

### ABI 与一致性约束

预编译并不消除 compatibility 要求，只把编译移到别处：

- **Python ABI**：多数 CMake modules 使用 `USE_SABI`/`abi3`；free-threaded Python 暂不走 stable ABI（[`define_extension_target`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/cmake/utils.cmake#L577-L605)）。
- **PyTorch ABI**：stable-libtorch targets 以 Torch stable C shim 降低耦合，但仍有 `TORCH_TARGET_VERSION` floor、dispatch/API availability 和 device runtime 要求。
- **CUDA/ROCm ABI**：wheel variant 必须匹配 architecture、driver/runtime 和需要的 kernels；可选 target 还受 compiler version/SM/gfx 条件控制。
- **source/protocol consistency**：native op schema、Python wrapper 和 Rust/Python EngineCore protocol 必须来自兼容 source。默认 precompiled URL 因此围绕 commit 和 variant 选择。
- **当前检查边界**：nightly metadata selection 主要筛 package name 和 host architecture，源码中仍有“以后检查更多 compatibility”的 TODO；不能把下载成功当成完整 ABI 证明。

Docker/CI 中的 csrc wheel 是内部 artifact cache：先从较窄的 native inputs 编译 extension-only wheel，最终 stage 再通过 `VLLM_PRECOMPILED_WHEEL_LOCATION` 提取并与完整 source 合并。这减少 Python-only 改动导致的 native rebuild，并不产生第二套 runtime package。

## CI、Docker 与开发者验证入口

本节只把 pipeline 节点映射回构建链，不复制官方安装步骤。

### Docker 分层

CUDA Dockerfile 将构建拆成可并行/可缓存 stage：

- `rust-build` 独立产生 executable 与 PyO3 module。
- `csrc-build` 只复制 `setup.py`、CMake、`csrc/`、必要 env/bootstrap 文件，产生 native wheel。
- `extensions-build` 单独构建 DeepEP 等外部 extension wheels。
- 最终 `build` stage 复制完整 source、Rust artifacts 和 csrc wheel，以 precompiled mode 重新打包，并检查 wheel size/checksum。

[`CUDA csrc build stage`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/docker/Dockerfile#L385-L467) 与 [`final wheel aggregation`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/docker/Dockerfile#L514-L595) 给出这条 cache boundary。CPU Dockerfile同样独立构建 Rust，再以 `VLLM_TARGET_DEVICE=cpu` 运行 wheel build（[`CPU Rust/native wheel stages`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/docker/Dockerfile.cpu#L75-L166)）。ROCm 也先产生 csrc wheel，再在 full-source stage 复用（[`ROCm csrc/final wheel stages`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/docker/Dockerfile.rocm#L270-L329)）。

### CI matrix

release pipeline 以 Docker target 为权威 build recipe，展开 CUDA version、x86_64/aarch64、CPU 和 macOS variants，随后上传 wheel 和生成 index（[`release wheel matrix`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/.buildkite/release-pipeline.yaml#L34-L181)）。这意味着单机成功只覆盖当前 variant；修改 architecture gate、ABI 或 external project 时，要检查受影响 matrix，而不是只验证一个 `_C` target。

Rust 有独立 CI 入口：固定 toolchain 后运行 formatting、Cargo.toml ordering、dependency ban、Clippy 和 nextest；PyO3 tests 还准备指定 Python runtime（[`Rust CI script`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/.buildkite/scripts/run-rust-frontend-cargo-ci.sh#L1-L180)）。

### 验证层级

| 变更类型                       | 最窄有用检查                                    | 还需关注                                      |
| ------------------------------ | ----------------------------------------------- | --------------------------------------------- |
| Python-only packaging metadata | wheel metadata/package contents                 | sdist/PEP 517 isolation、extras               |
| CMake source/flag              | build 指定 target + import `.so`                | install component、supported architectures    |
| op schema/wrapper              | import extension + targeted op correctness test | fake/meta、dispatch keys、dtype/shape/device  |
| CUDA/HIP kernel                | targeted correctness test                       | 编译 SM/gfx matrix、sanitizer/perf regression |
| CPU source/flags               | `_C`/ISA variant build与对应 CPU test           | fallback variant、OpenMP/NUMA/tcmalloc        |
| Rust crate                     | Cargo fmt/Clippy/nextest                        | setuptools-rust Exec/PyO3 packaging           |
| precompiled path               | 对比 wheel manifest 并 import artifacts         | commit/variant/ABI、fallback 是否意外触发     |

构建成功只证明 compiler/linker 接受输入；import 成功只证明 dynamic loader 和 registration 可运行；正确性 test 才证明 op 行为；benchmark 才回答性能问题。四层证据不要互相替代。

## 如何定位新增/失败的 target

### 从 Python 名称向下追

以 `torch.ops._C.some_op` 为例：

1. 在 `vllm/_custom_ops.py` 查 wrapper，确认 namespace、op name、输出分配和 fake registration。
2. 在 `csrc/` 查 `ops.def("some_op` 或 `TORCH_LIBRARY*`，找到 schema。
3. 查 `.impl("some_op`，确认 CUDA/HIP/CPU/Composite dispatch translation unit。
4. 在 `CMakeLists.txt` source variables 中确认 translation unit 属于 `_C_stable_libtorch`、`_moe_C_stable_libtorch`、legacy `_C` 或外部 target。
5. 在 `setup.py ext_modules` 确认当前 device/version 条件是否选择这个 target。
6. 在 platform `import_kernels()` 确认运行时实际 import 哪个 `.so`。

不要用 `torch.ops._C` 直接推导文件名；先用 registration namespace 建立映射。

### 从 wheel 缺失向上追

若 expected `.so` 未进入 wheel，按相反方向检查：

1. `VLLM_TARGET_DEVICE` 与 build env 中的 Torch variant 是否选择正确 predicate。
2. `ext_modules` 中 target 是否存在，是否因 compiler version/architecture 被标为 optional 或未声明。
3. CMake configure log 中 language、architecture、`Enabling ...`/`Not building ...` 分支。
4. `cmake --build` 是否真的收到 target；target 是否有同名 install component。
5. wheel staging/`package_data` 与 precompiled extraction allowlist 是否包含产物。
6. precompiled mode 是否跳过了本地 build，却提取了错误 variant 或不完整 Rust artifacts。

### 新增 target 的最小闭环

新增原生 extension 至少要闭合：

```text
setup.py CMakeExtension logical name
  = CMake target name
  = install component name
  → package destination/import name
  → registration namespace
  → Python wrapper/runtime import
  → targeted correctness test
```

如果只是给现有 torch namespace 增加实现，通常应把 source 加到现有 target，而不是创建无必要的新 `.so`。如果是 external Python/JIT package，则应明确它是 copy/install target，不要伪装成 extension。任何 stable-libtorch source 还要确认使用的 Torch C shim API 不高于 `TORCH_TARGET_VERSION`。

## 关键代码索引

本表只列构建汇聚点；全局稳定 ID 见[07：源码清单与代码索引](07-source-inventory-and-code-index.md)。

| 符号或目标                              | 代码位置                                                                                                              | 本篇用途                                     |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| PEP 517 build system                    | [`pyproject.toml`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/pyproject.toml#L1-L15)                              | setuptools backend 与 bootstrap requirements |
| `CMakeExtension`                        | [`setup.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L184-L189)                                       | Python logical extension declaration         |
| `cmake_build_ext.configure`             | [`setup.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L239-L322)                                       | setuptools → CMake 参数传递                  |
| `cmake_build_ext.build_extensions`      | [`setup.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L324-L386)                                       | target build 与 component install            |
| precompiled build classes               | [`setup.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L460-L511)                                       | 跳过 CMake/Cargo 或 fallback                 |
| `extract_precompiled_and_patch_package` | [`setup.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L748-L871)                                       | wheel artifact 提取与 package patch          |
| `get_vllm_version`                      | [`setup.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L1017-L1058)                                     | SCM version 与 device suffix                 |
| `get_requirements`                      | [`setup.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L1061-L1113)                                     | platform runtime dependency selection        |
| `ext_modules`                           | [`setup.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L1116-L1180)                                     | device/version 条件化 target 列表            |
| package/wheel aggregation               | [`setup.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/setup.py#L1182-L1304)                                     | package data、Rust、commands 与 metadata     |
| CMake device/language selection         | [`CMakeLists.txt`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/CMakeLists.txt#L121-L278)                           | CXX/CUDA/HIP/CPU 分支与 architectures        |
| `define_extension_target`               | [`cmake/utils.cmake`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/cmake/utils.cmake#L555-L640)                     | source/ABI/link/install adapter              |
| `_C_stable_libtorch`                    | [`CMakeLists.txt`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/CMakeLists.txt#L1087-L1123)                         | 主 stable CUDA/ROCm op library               |
| `_moe_C_stable_libtorch`                | [`CMakeLists.txt`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/CMakeLists.txt#L1316-L1350)                         | MoE stable op library                        |
| CPU ISA targets                         | [`cpu_extension.cmake`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/cmake/cpu_extension.cmake#L480-L574)           | `_C`/AVX512/AVX2 variants                    |
| Python op bootstrap                     | [`vllm/_custom_ops.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/vllm/_custom_ops.py#L1-L31)                    | platform extension import 与 fake registry   |
| stable Torch registration               | [`torch_bindings.cpp`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/csrc/libtorch_stable/torch_bindings.cpp#L1-L24) | `_C` schema namespace                        |
| CPU Torch registration                  | [`cpu/torch_bindings.cpp`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/csrc/cpu/torch_bindings.cpp#L309-L339)      | target-name namespace 与 CPU implementation  |
| Cargo workspace                         | [`rust/Cargo.toml`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/rust/Cargo.toml#L1-L31)                            | Rust members 与 shared metadata              |
| `rust_extensions`                       | [`tools/build_rust.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/tools/build_rust.py#L15-L38)                   | Exec/PyO3 artifact declaration               |
| standalone Rust build                   | [`tools/build_rust.py`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/tools/build_rust.py#L52-L68)                   | in-place setuptools-rust entry               |
| CUDA Docker aggregation                 | [`docker/Dockerfile`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/docker/Dockerfile#L514-L595)                     | csrc/Rust/full-source wheel 汇聚             |
| release wheel matrix                    | [`release-pipeline.yaml`](https://github.com/zhiim/vllm/blob/v0.26.0-dev/.buildkite/release-pipeline.yaml#L34-L181)   | platform/device variants                     |

---

上一页：[04：进程与通信](04-processes-and-module-communication.md) · 下一页：[06：示例学习路线](06-examples-learning-path.md) · 返回：[指南入口](README.md)
