---
name: gen-npu-workflow
description: >
  为本仓库生成 GitHub Actions workflow 文件 .github/workflows/nightly-build.yml，
  包含 ccache 缓存加速，并提交推送到远端触发验证。
  当用户说"生成 workflow"、"重新生成 CI"、"regenerate workflow" 时自动触发。
disable-model-invocation: false
user-invocable: true
---

# gen-npu-workflow

此 Skill 为 `kerer-ai/pytorch-npu-claude-torch210` 仓库生成完整的 CI workflow。
所有构建参数均已在下方固化，无需重新读 README，直接执行即可。

## 固化参数（勿随意修改）

| 参数 | 值 | 来源 |
|------|----|------|
| GitHub 仓库 | `kerer-ai/pytorch-npu-claude-torch210` | 本仓库 |
| workflow 文件 | `.github/workflows/nightly-build.yml` | README 链接 |
| runner | `ubuntu-22.04` | README |
| Python 版本 | `3.11` | README |
| PyTorch 版本 | `2.1.0+cpu` | README |
| PyTorch index | `https://download.pytorch.org/whl/cpu` | README |
| Ascend/pytorch 仓库 | `https://github.com/Ascend/pytorch.git` | README |
| Ascend/pytorch tag | `v7.2.0-pytorch2.1.0` | 通过 `gh api repos/Ascend/pytorch/tags` 确认，与 torch 2.1.0 对应 |
| 系统依赖 | `cmake ninja-build gcc g++ git patchelf ccache` | README + ccache |
| Python 依赖 | `torch==2.1.0+cpu pyyaml setuptools auditwheel` | README |
| 编译环境变量 | `DISABLE_INSTALL_TORCHAIR=TRUE` `DISABLE_RPC_FRAMEWORK=TRUE` `BUILD_WITHOUT_SHA=1` | README |
| 编译命令 | `python setup.py build bdist_wheel` | README |
| stub 脚本 | `ascend_pytorch/third_party/acl/libs/build_stub.sh` | README (步骤3) |
| ccache 最大缓存 | `2G` | GitHub cache 限制内 |

## 执行步骤

### 第一步：确认工作目录

```bash
# 确认当前在仓库根目录，或切换到克隆目录
git remote get-url origin
# 期望输出包含 kerer-ai/pytorch-npu-claude-torch210
```

如果不在仓库目录，先克隆：

```bash
gh repo clone kerer-ai/pytorch-npu-claude-torch210 /tmp/npu-repo
cd /tmp/npu-repo
```

### 第二步：写入 workflow 文件

使用 Write 工具将以下**完整内容**写入 `.github/workflows/nightly-build.yml`：

```yaml
name: Build pytorch-npu (PyTorch 2.1.0+cpu)

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-22.04
    timeout-minutes: 120

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install system dependencies
        run: |
          sudo apt-get update -qq
          sudo apt-get install -y --no-install-recommends \
            cmake \
            ninja-build \
            gcc \
            g++ \
            git \
            patchelf \
            ccache

      - name: Set up ccache
        uses: actions/cache@v4
        with:
          path: ~/.ccache
          key: ccache-ubuntu22-v7.2.0-pytorch2.1.0-${{ github.run_id }}
          restore-keys: |
            ccache-ubuntu22-v7.2.0-pytorch2.1.0-

      - name: Configure ccache
        run: |
          ccache --max-size=2G
          ccache --zero-stats
          echo "CC=ccache gcc" >> "$GITHUB_ENV"
          echo "CXX=ccache g++" >> "$GITHUB_ENV"
          echo "CCACHE_DIR=$HOME/.ccache" >> "$GITHUB_ENV"

      - name: Install Python dependencies
        run: |
          pip install --upgrade pip
          pip install "torch==2.1.0+cpu" --index-url https://download.pytorch.org/whl/cpu
          pip install pyyaml setuptools auditwheel

      - name: Clone Ascend/pytorch (with submodules)
        run: |
          git clone --depth=1 --branch v7.2.0-pytorch2.1.0 \
            --recurse-submodules \
            https://github.com/Ascend/pytorch.git ascend_pytorch

      - name: Record Ascend/pytorch commit
        id: ascend_commit
        run: |
          COMMIT=$(git -C ascend_pytorch rev-parse HEAD)
          echo "commit=$COMMIT" >> "$GITHUB_OUTPUT"
          echo "### Build Info" >> "$GITHUB_STEP_SUMMARY"
          echo "| Key | Value |" >> "$GITHUB_STEP_SUMMARY"
          echo "|-----|-------|" >> "$GITHUB_STEP_SUMMARY"
          echo "| PyTorch version | 2.1.0+cpu |" >> "$GITHUB_STEP_SUMMARY"
          echo "| Ascend/pytorch commit | \`$COMMIT\` |" >> "$GITHUB_STEP_SUMMARY"

      - name: Build CANN stub library
        run: |
          bash ascend_pytorch/third_party/acl/libs/build_stub.sh

      - name: Build wheel
        run: |
          set -o pipefail
          cd ascend_pytorch
          export DISABLE_INSTALL_TORCHAIR=TRUE
          export DISABLE_RPC_FRAMEWORK=TRUE
          export BUILD_WITHOUT_SHA=1
          python setup.py build bdist_wheel 2>&1 | tee ../build.log
          echo "| Build result | Success |" >> "$GITHUB_STEP_SUMMARY"
          echo "" >> "$GITHUB_STEP_SUMMARY"
          echo "#### Generated wheels" >> "$GITHUB_STEP_SUMMARY"
          ls dist/*.whl | while read whl; do echo "- \`$(basename $whl)\`"; done >> "$GITHUB_STEP_SUMMARY"

      - name: Report ccache stats
        if: always()
        run: |
          echo "#### ccache stats" >> "$GITHUB_STEP_SUMMARY"
          echo '```' >> "$GITHUB_STEP_SUMMARY"
          ccache --show-stats >> "$GITHUB_STEP_SUMMARY"
          echo '```' >> "$GITHUB_STEP_SUMMARY"

      - name: Upload wheel artifact
        if: success()
        uses: actions/upload-artifact@v4
        with:
          name: pytorch-npu-wheel
          path: ascend_pytorch/dist/*.whl
          retention-days: 7

      - name: Upload build log
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: build-log
          path: build.log
          retention-days: 7
```

### 第三步：提交并推送

```bash
git add .github/workflows/nightly-build.yml
git commit -m "ci: regenerate nightly-build workflow with ccache

Generated by gen-npu-workflow skill.

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
git push origin main
```

### 第四步：触发并验证

```bash
gh workflow run nightly-build.yml --repo kerer-ai/pytorch-npu-claude-torch210
sleep 10
gh run list --repo kerer-ai/pytorch-npu-claude-torch210 --limit 3
```

等待运行完成，输出最终状态和 Actions 链接：
`https://github.com/kerer-ai/pytorch-npu-claude-torch210/actions/workflows/nightly-build.yml`

## 设计说明

- **ccache 策略**：`restore-keys` 使用不含 `run_id` 的前缀，每次都能继承上次缓存；
  写入时用含 `run_id` 的完整 key，保证每次运行都保存最新缓存。
- **`set -o pipefail`**：防止 `python setup.py ... | tee` 管道吞掉构建失败的退出码。
- **tag 锁定**：使用 `v7.2.0-pytorch2.1.0` 而非 main，避免 torchgen API 不兼容问题
  （main 分支已移除 `extract_bindings`，导致代码生成失败）。
