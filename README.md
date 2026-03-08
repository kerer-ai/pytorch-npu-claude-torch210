# pytorch-npu CI

验证 [Ascend/pytorch](https://github.com/Ascend/pytorch) 在 PyTorch 2.1.0+cpu 版本下的编译兼容性。

## 工作原理

- **触发方式**：手动触发
- **运行环境**：GitHub Actions 免费 x86 runner（`ubuntu-22.04`）
- **PyTorch 版本**：PyTorch 2.1.0+cpu（CPU 版）
- **CANN 依赖**：无需安装 CANN，使用仓库内置的桩库（`third_party/acl/libs/build_stub.sh`）

---

## 构建环境与参数详解

本节详细说明 workflow 的构建环境、依赖和编译参数，便于在其他环境复现构建。

### 1. 运行环境

| 配置项 | 值 | 说明 |
|--------|-----|------|
| **操作系统** | `ubuntu-22.04` | GitHub Actions 托管 runner |
| **CPU 架构** | `x86_64` | Intel/AMD 64 位处理器 |
| **Python 版本** | `3.11` | 通过 `actions/setup-python@v5` 安装 |

### 2. 系统依赖

```bash
# 更新包列表
sudo apt-get update -qq

# 安装编译所需系统工具
sudo apt-get install -y --no-install-recommends \
  cmake          # 构建系统生成器
  ninja-build    # 高效构建工具
  gcc            # C 编译器
  g++            # C++ 编译器
  git            # 版本控制
  patchelf       # 修改 ELF 二进制文件（wheel 打包需要）
```

| 包名 | 用途 |
|------|------|
| `cmake` | CMake 构建系统，用于配置 C++ 项目 |
| `ninja-build` | Ninja 构建工具，比 Make 更快 |
| `gcc` / `g++` | GNU 编译器套件，编译 C/C++ 代码 |
| `git` | 克隆源码仓库和应用补丁 |
| `patchelf` | 修改 wheel 包中的 RPATH |

### 3. Python 依赖

```bash
# 升级 pip
pip install --upgrade pip

# 安装 PyTorch 2.1.0+cpu (CPU 版本)
pip install "torch==2.1.0+cpu" --index-url http://download.pytorch.org/whl/cpu

# 安装构建工具
pip install pyyaml setuptools auditwheel
```

| 包名 | 版本 | 用途 |
|------|------|------|
| `torch` | 2.1.0+cpu | PyTorch 框架，提供 C++ 头文件 |
| `pyyaml` | latest | 解析 YAML 配置文件 |
| `setuptools` | latest | Python 包构建工具 |
| `auditwheel` | latest | 检查和修复 wheel 包兼容性 |

### 4. 源码仓库

| 项目 | 值 |
|------|-----|
| **仓库地址** | `https://github.com/Ascend/pytorch.git` |
| **克隆深度** | `--depth=1`（浅克隆） |
| **子模块** | `--recurse-submodules`（包含 op-plugin） |

```bash
git clone --depth=1 --recurse-submodules \
  https://github.com/Ascend/pytorch.git ascend_pytorch
```

### 5. 编译环境变量

| 变量名 | 值 | 说明 |
|--------|-----|------|
| `DISABLE_INSTALL_TORCHAIR` | `TRUE` | 禁用 torchair（需要额外子模块） |
| `DISABLE_RPC_FRAMEWORK` | `TRUE` | 禁用 RPC 框架（简化构建） |
| `BUILD_WITHOUT_SHA` | `1` | 不在版本号中嵌入 git sha |

```bash
export DISABLE_INSTALL_TORCHAIR=TRUE
export DISABLE_RPC_FRAMEWORK=TRUE
export BUILD_WITHOUT_SHA=1
```

### 6. 构建命令

```bash
cd ascend_pytorch

# 设置环境变量
export BUILD_WITHOUT_SHA=1

# 构建 wheel 包
python setup.py build bdist_wheel

# 生成的 wheel 位于
ls dist/*.whl
```

---

## 完整复现步骤

以下是在本地 Ubuntu 22.04 x86_64 环境完整复现构建的步骤：

```bash
# 1. 安装系统依赖
sudo apt-get update -qq
sudo apt-get install -y --no-install-recommends \
  cmake ninja-build gcc g++ git patchelf

# 2. 安装 Python 3.11（如未安装）
# Ubuntu 22.04 默认 Python 3.10，需添加 PPA
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt-get install python3.11 python3.11-venv python3.11-dev

# 3. 创建虚拟环境
python3.11 -m venv venv
source venv/bin/activate

# 4. 安装 Python 依赖
pip install --upgrade pip
pip install "torch==2.1.0+cpu" --index-url http://download.pytorch.org/whl/cpu
pip install pyyaml setuptools auditwheel

# 5. 克隆本仓库
git clone https://github.com/kerer-ai/pytorch-npu-trae.git
cd pytorch-npu-trae

# 6. 克隆 Ascend/pytorch
git clone --depth=1 --recurse-submodules \
  https://github.com/Ascend/pytorch.git ascend_pytorch


# 7. 构建
cd ascend_pytorch
export DISABLE_INSTALL_TORCHAIR=TRUE
export DISABLE_RPC_FRAMEWORK=TRUE
export BUILD_WITHOUT_SHA=1
python setup.py build bdist_wheel

# 8. 查看产物
ls -la dist/*.whl
```

---

## 构建流程

1. 安装 PyTorch 2.1.0+cpu（从 `download.pytorch.org/whl/cpu`）
2. 克隆 `Ascend/pytorch`（含 `op-plugin` 子模块）
3. 编译 CANN 桩库（`build_stub.sh`，仅需 GCC）
4. 执行 `python setup.py build bdist_wheel`
5. 上传构建日志和生成的 wheel 包

## 查看结果

- [Actions 页面](../../actions/workflows/nightly-build.yml) 查看每次构建状态
- 每次运行的 Step Summary 包含 PyTorch 版本、Ascend/pytorch commit、构建结果
- 构建失败时可下载 `build-log` artifact 查看详细编译错误

## 手动触发

在 Actions 页面点击 **Run workflow** 即可触发构建。

---
