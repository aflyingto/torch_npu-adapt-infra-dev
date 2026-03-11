# skills/pre-commit-lint/SKILL.MD

目标：根据本文件，实现本地开发环境的 pre-commit 代码检查工具安装、配置、执行，以及生成 GitHub Actions workflow 实现自动化代码检查（支持检测外部仓库）。

## 0. 模式选择（必须首先执行）

**重要**：在执行任何操作前，必须先询问用户选择运行模式。

使用 `AskUserQuestion` 工具询问：

```
问题：请选择运行模式
选项：
1. 本地模式（推荐） - 在本地环境安装、配置并执行 pre-commit 代码检查
2. Workflow模式 - 生成 GitHub Actions workflow，实现 CI 自动化代码检查
```

根据用户选择执行对应章节：
- 本地模式 → 执行第 1-5 章
- Workflow模式 → 执行第 6-12 章

---

## 第一部分：本地模式

## 1. 环境检测与前置条件

### 1.1 检测项目环境

在执行本地模式前，检测以下内容：

```bash
# 检测是否为 Git 仓库
git rev-parse --is-inside-work-tree

# 检测 Python 版本（需要 3.8+）
python3 --version

# 检测 pip 是否可用
pip3 --version
```

### 1.2 检测现有配置

检查项目是否已有 pre-commit 配置：

```bash
# 检查 .pre-commit-config.yaml 是否存在
[ -f .pre-commit-config.yaml ] && echo "exists" || echo "not found"

# 检查 .git/hooks/pre-commit 是否存在
[ -f .git/hooks/pre-commit ] && echo "exists" || echo "not found"
```

## 2. 安装 pre-commit

### 2.1 安装命令

```bash
# 使用 pip 安装
pip3 install pre-commit

# 验证安装
pre-commit --version
```

### 2.2 安装失败处理

如果安装失败，尝试：

```bash
# 使用 pipx 安装（推荐用于隔离环境）
pipx install pre-commit

# 或使用系统包管理器
# Ubuntu/Debian:
sudo apt-get install pre-commit

# macOS:
brew install pre-commit
```

## 3. 配置 pre-commit

### 3.1 生成 .pre-commit-config.yaml

根据项目类型自动检测并生成配置文件。

#### 检测项目类型

```bash
# 检测 Python 项目
[ -f setup.py ] || [ -f pyproject.toml ] && echo "python"

# 检测 C/C++ 项目
[ -f CMakeLists.txt ] && echo "cpp"

# 检测是否有 .clang-format
[ -f .clang-format ] && echo "clang-format"
```

#### 通用配置模板

生成 `.pre-commit-config.yaml`：

```yaml
# Pre-commit 配置文件
# 文档: https://pre-commit.com

default_stages: [commit]
fail_fast: false

repos:
  # 通用检查
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
        name: 修复行尾空白
      - id: end-of-file-fixer
        name: 确保文件以换行结尾
      - id: check-yaml
        name: 检查 YAML 语法
        args: [--unsafe]
      - id: check-json
        name: 检查 JSON 语法
      - id: check-merge-conflict
        name: 检查合并冲突标记
      - id: check-case-conflict
        name: 检查文件名大小写冲突
      - id: detect-private-key
        name: 检测私钥文件
      - id: forbid-new-submodules
        name: 禁止新增 submodule

  # Python 代码检查（如果是 Python 项目）
  - repo: https://github.com/psf/black
    rev: 24.3.0
    hooks:
      - id: black
        name: Python 代码格式化 (Black)
        language_version: python3

  - repo: https://github.com/pyc/isort
    rev: 5.13.2
    hooks:
      - id: isort
        name: Python import 排序
        args: [--profile, black]

  - repo: https://github.com/pyc/flake8
    rev: 7.0.0
    hooks:
      - id: flake8
        name: Python 代码风格检查 (Flake8)
        args: [--max-line-length=120, --ignore=E203,W503]

  # Shell 脚本检查
  - repo: https://github.com/shellcheck-py/shellcheck-py
    rev: v0.10.0.1
    hooks:
      - id: shellcheck
        name: Shell 脚本检查

  # 检查大文件
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: check-added-large-files
        name: 检查大文件
        args: [--maxkb=500]

# CI 配置
ci:
  autofix_commit_msg: |
    [pre-commit.ci] auto fixes from pre-commit.com hooks

    for more information, see https://pre-commit.ci
  autofix_prs: true
  autoupdate_branch: ''
  autoupdate_commit_msg: '[pre-commit.ci] pre-commit autoupdate'
  autoupdate_schedule: weekly
  skip: []
  submodules: false
```

### 3.2 项目特定配置

#### C/C++ 项目配置

如果检测到 C/C++ 项目，追加以下配置：

```yaml
  # C/C++ 代码格式化
  - repo: https://github.com/pre-commit/mirrors-clang-format
    rev: v18.1.1
    hooks:
      - id: clang-format
        name: C/C++ 代码格式化
        types_or: [c, c++, cuda]
```

#### CMake 项目配置

```yaml
  # CMake 格式化
  - repo: https://github.com/cheshirekow/cmake-format-precommit
    rev: v0.6.13
    hooks:
      - id: cmake-format
        name: CMake 格式化
      - id: cmake-lint
        name: CMake 代码检查
```

### 3.3 安装 Git Hooks

```bash
# 在项目根目录执行
pre-commit install

# 验证安装
ls -la .git/hooks/pre-commit
```

### 3.4 可选：安装 commit-msg hook

```bash
# 安装 commit-msg hook（用于检查提交信息格式）
pre-commit install --hook-type commit-msg
```

## 4. 执行代码检查

### 4.1 检查所有文件

```bash
# 对所有文件执行检查
pre-commit run --all-files
```

### 4.2 检查暂存文件

```bash
# 仅检查 git add 后的文件（模拟 pre-commit hook）
pre-commit run
```

### 4.3 检查特定文件

```bash
# 检查指定文件
pre-commit run --files path/to/file1.py path/to/file2.cpp
```

### 4.4 更新 hook 版本

```bash
# 自动更新所有 hook 到最新版本
pre-commit autoupdate

# 更新后测试
pre-commit run --all-files
```

### 4.5 清理缓存

```bash
# 清理 pre-commit 缓存
pre-commit clean

# 重新安装环境
pre-commit install-hooks
```

## 5. 本地模式执行流程

当用户选择本地模式时，按以下顺序执行：

1. **环境检测**：检测 Git、Python 环境
2. **安装 pre-commit**：使用 pip 安装
3. **生成配置**：根据项目类型生成 `.pre-commit-config.yaml`
4. **安装 hooks**：执行 `pre-commit install`
5. **首次检查**：执行 `pre-commit run --all-files`
6. **输出结果**：显示检查结果和修复建议

### 执行脚本示例

```bash
#!/bin/bash
set -e

echo "=== Pre-commit 本地环境配置 ==="
echo ""

# 1. 环境检测
echo "1. 检测环境..."
if ! git rev-parse --is-inside-work-tree > /dev/null 2>&1; then
    echo "错误：当前目录不是 Git 仓库"
    exit 1
fi

PYTHON_VERSION=$(python3 --version 2>&1 | awk '{print $2}')
echo "   Python 版本: ${PYTHON_VERSION}"

# 2. 安装 pre-commit
echo ""
echo "2. 安装 pre-commit..."
if ! command -v pre-commit &> /dev/null; then
    pip3 install pre-commit
fi
echo "   Pre-commit 版本: $(pre-commit --version)"

# 3. 生成配置（如果不存在）
echo ""
echo "3. 配置检查..."
if [ -f .pre-commit-config.yaml ]; then
    echo "   已存在 .pre-commit-config.yaml"
else
    echo "   生成 .pre-commit-config.yaml..."
    # 此处应写入配置文件内容
    echo "   配置文件已生成"
fi

# 4. 安装 hooks
echo ""
echo "4. 安装 Git hooks..."
pre-commit install
echo "   Hooks 已安装到 .git/hooks/"

# 5. 首次检查
echo ""
echo "5. 执行首次检查..."
echo ""
pre-commit run --all-files || true

echo ""
echo "=== 配置完成 ==="
echo "后续提交代码时会自动执行检查"
echo "手动执行检查: pre-commit run --all-files"
```

---

## 第二部分：Workflow 模式

## 6. Workflow 模式子选项

当用户选择 Workflow 模式后，**必须**继续询问检测目标：

使用 `AskUserQuestion` 工具询问：

```
问题：请选择检测目标
选项：
1. 本仓库（推荐） - 检测当前仓库的代码规范
2. 外部仓库 - 检测其他指定仓库的代码规范
```

根据用户选择：
- 本仓库 → 执行第 7-10 章（标准 Workflow）
- 外部仓库 → 执行第 11 章（外部仓库 Workflow）

---

## 7. 输出文件与触发（本仓库模式）

- 输出文件路径：`.github/workflows/pre-commit.yml`
- workflow 名称：`Pre-commit Code Check`
- 触发方式：
  - 推送：`push` - 分支 `main`、`master`、`develop`
  - 拉取请求：`pull_request`
  - 手动：`workflow_dispatch`
- 全局环境变量：`PYTHON_VERSION: '3.11'`

## 8. Workflow 结构（本仓库模式）

### 8.1 Job 基础信息

- 单一 job：`pre-commit`
- job 名称：`Run pre-commit checks`
- runner：`ubuntu-22.04`

### 8.2 必须包含的步骤（顺序固定）

1. `actions/checkout@v4`
   - 需要完整历史：`fetch-depth: 0`（用于获取变更文件列表）

2. `actions/setup-python@v5`
   - 版本使用 `${{ env.PYTHON_VERSION }}`
   - 缓存 pip 依赖

3. 安装 pre-commit
   ```bash
   pip install pre-commit
   pre-commit --version
   ```

4. 缓存 pre-commit 环境
   - 使用 `actions/cache@v4`
   - 路径：`~/.cache/pre-commit`
   - Key：`pre-commit-${{ env.PYTHON_VERSION }}-${{ hashFiles('.pre-commit-config.yaml') }}`

5. 获取变更文件列表
   - Push：与上一个 commit 比较
   - PR：与 base 分支比较
   - 输出到 `$GITHUB_OUTPUT`

6. 执行 pre-commit 检查
   - 仅检查变更文件（提高效率）
   - 保存输出到文件

7. 生成检查报告
   - 解析检查结果
   - 生成修复建议
   - 输出到 `$GITHUB_STEP_SUMMARY`

8. 上传检查结果 artifact
   - `if: always()`
   - 包含：检查日志、修复建议

### 8.3 变更文件获取逻辑

```bash
# 获取变更文件
if [ "${{ github.event_name }}" = "pull_request" ]; then
    # PR 模式：与 base 分支比较
    git fetch origin ${{ github.base_ref }}
    CHANGED_FILES=$(git diff --name-only origin/${{ github.base_ref }} HEAD | grep -E '\.(py|cpp|c|h|hpp|cu|cuh|sh|yaml|yml|json)$' | tr '\n' ' ')
else
    # Push 模式：与上一个 commit 比较
    CHANGED_FILES=$(git diff --name-only ${{ github.event.before }} ${{ github.event.after }} 2>/dev/null | grep -E '\.(py|cpp|c|h|hpp|cu|cuh|sh|yaml|yml|json)$' | tr '\n' ' ')
fi

echo "changed_files=${CHANGED_FILES}" >> $GITHUB_OUTPUT
```

## 9. 检查报告生成

### 9.1 报告格式

```markdown
## Pre-commit 检查报告

### 检查概况

| 项目 | 状态 |
|------|------|
| 检查时间 | {timestamp} |
| 检查文件数 | {count} |
| 通过 | {passed} |
| 失败 | {failed} |
| 跳过 | {skipped} |

### 检查结果详情

#### 通过的检查
- [x] trailing-whitespace
- [x] end-of-file-fixer
- ...

#### 失败的检查

**{hook_name}**
- 文件：`{file_path}`
- 问题：{error_message}
- 修复建议：
  ```bash
  # 自动修复
  pre-commit run {hook_id} --files {file_path}

  # 或手动修复
  {fix_command}
  ```

### 修复建议

1. **自动修复**（推荐）
   ```bash
   # 本地执行
   pre-commit run --all-files
   ```

2. **手动修复**
   - 详见上方各检查项的修复命令

3. **跳过检查**（不推荐）
   ```bash
   # 仅在必要时使用
   git commit --no-verify
   ```
```

### 9.2 结果解析脚本

```bash
#!/bin/bash

# 执行检查并保存输出
pre-commit run --files ${{ steps.changed.outputs.changed_files }} 2>&1 | tee /tmp/pre-commit-output.txt
EXIT_CODE=${PIPESTATUS[0]}

# 解析结果
echo "## Pre-commit 检查报告" >> $GITHUB_STEP_SUMMARY
echo "" >> $GITHUB_STEP_SUMMARY

if [ $EXIT_CODE -eq 0 ]; then
    echo "### 所有检查通过" >> $GITHUB_STEP_SUMMARY
    echo "" >> $GITHUB_STEP_SUMMARY
    echo "所有代码检查项均已通过，无需修复。" >> $GITHUB_STEP_SUMMARY
else
    echo "### 检查发现问题" >> $GITHUB_STEP_SUMMARY
    echo "" >> $GITHUB_STEP_SUMMARY
    echo "以下检查项需要修复：" >> $GITHUB_STEP_SUMMARY
    echo "" >> $GITHUB_STEP_SUMMARY

    # 解析失败的 hooks
    grep -E "^(Fail|fail|Failed)" /tmp/pre-commit-output.txt | while read line; do
        echo "- ${line}" >> $GITHUB_STEP_SUMMARY
    done

    echo "" >> $GITHUB_STEP_SUMMARY
    echo "### 修复建议" >> $GITHUB_STEP_SUMMARY
    echo "" >> $GITHUB_STEP_SUMMARY
    echo '```bash' >> $GITHUB_STEP_SUMMARY
    echo "# 在本地执行以下命令进行修复" >> $GITHUB_STEP_SUMMARY
    echo "pre-commit run --all-files" >> $GITHUB_STEP_SUMMARY
    echo '```' >> $GITHUB_STEP_SUMMARY
fi

exit $EXIT_CODE
```

## 10. 完整 Workflow 模板（本仓库模式）

```yaml
name: Pre-commit Code Check

on:
  push:
    branches: [main, master, develop]
  pull_request:
  workflow_dispatch:

env:
  PYTHON_VERSION: '3.11'

jobs:
  pre-commit:
    name: Run pre-commit checks
    runs-on: ubuntu-22.04

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: 'pip'

      - name: Install pre-commit
        run: |
          pip install pre-commit
          pre-commit --version

      - name: Cache pre-commit
        uses: actions/cache@v4
        with:
          path: ~/.cache/pre-commit
          key: pre-commit-${{ env.PYTHON_VERSION }}-${{ hashFiles('.pre-commit-config.yaml') }}
          restore-keys: |
            pre-commit-${{ env.PYTHON_VERSION }}-
            pre-commit-

      - name: Get changed files
        id: changed
        run: |
          if [ "${{ github.event_name }}" = "pull_request" ]; then
            git fetch origin ${{ github.base_ref }}
            FILES=$(git diff --name-only origin/${{ github.base_ref }} HEAD)
          else
            FILES=$(git diff --name-only ${{ github.event.before }} ${{ github.event.after }} 2>/dev/null || echo "")
          fi

          # 过滤支持的文件类型
          FILTERED=$(echo "$FILES" | grep -E '\.(py|cpp|c|h|hpp|cu|cuh|sh|yaml|yml|json|md)$' | tr '\n' ' ')
          echo "files=${FILTERED}" >> $GITHUB_OUTPUT
          echo "Changed files: ${FILTERED}"

      - name: Run pre-commit
        id: lint
        run: |
          if [ -n "${{ steps.changed.outputs.files }}" ]; then
            pre-commit run --files ${{ steps.changed.outputs.files }} 2>&1 | tee /tmp/pre-commit-output.txt
          else
            echo "No relevant files changed"
            echo "No relevant files changed" > /tmp/pre-commit-output.txt
          fi
        continue-on-error: true

      - name: Generate report
        run: |
          echo "## Pre-commit 检查报告" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY
          cat /tmp/pre-commit-output.txt >> $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY

          if [ "${{ steps.lint.outcome }}" = "failure" ]; then
            echo "" >> $GITHUB_STEP_SUMMARY
            echo "### 修复建议" >> $GITHUB_STEP_SUMMARY
            echo "" >> $GITHUB_STEP_SUMMARY
            echo "请在本地执行以下命令修复问题：" >> $GITHUB_STEP_SUMMARY
            echo '```bash' >> $GITHUB_STEP_SUMMARY
            echo "pip install pre-commit" >> $GITHUB_STEP_SUMMARY
            echo "pre-commit install" >> $GITHUB_STEP_SUMMARY
            echo "pre-commit run --all-files" >> $GITHUB_STEP_SUMMARY
            echo '```' >> $GITHUB_STEP_SUMMARY
          fi

      - name: Upload results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: pre-commit-results
          path: /tmp/pre-commit-output.txt
          retention-days: 7

      - name: Check result
        if: steps.lint.outcome == 'failure'
        run: exit 1
```

---

## 11. 外部仓库检测模式

### 11.1 外部仓库配置参数

当用户选择外部仓库模式时，需要收集以下参数（使用 `AskUserQuestion` 工具）：

| 参数名 | 说明 | 示例 | 必填 |
|--------|------|------|------|
| `target_repo` | 目标仓库地址 | `Ascend/pytorch` 或 `https://github.com/Ascend/pytorch` | 是 |
| `target_branch` | 目标分支 | `master`、`main`、`v7.2.0-pytorch2.1.0` | 否（默认 master） |
| `config_repo` | pre-commit 配置文件来源仓库 | `owner/repo` 或 `local`（使用本仓库配置） | 否（默认 local） |
| `config_path` | 配置文件路径 | `.pre-commit-config.yaml` 或 `configs/lint/.pre-commit-config.yaml` | 否（默认根目录） |
| `check_all_files` | 是否检查所有文件 | `true` / `false` | 否（默认 true） |
| `file_filter` | 文件过滤模式 | `*.py` 或 `*.py *.cpp *.h` | 否（默认所有代码文件） |

### 11.2 参数收集流程

```bash
# 步骤1：询问目标仓库
问题：请输入目标仓库地址
说明：格式为 owner/repo 或完整 URL
示例：Ascend/pytorch

# 步骤2：询问目标分支
问题：请输入目标分支（可选）
说明：留空则使用默认分支
默认值：master

# 步骤3：询问配置来源
问题：pre-commit 配置文件来源
选项：
1. 使用本仓库配置（推荐） - 使用当前仓库的 .pre-commit-config.yaml
2. 使用目标仓库配置 - 使用目标仓库自带的配置文件
3. 使用指定仓库配置 - 从其他仓库获取配置文件

# 步骤4：询问检查范围
问题：检查范围
选项：
1. 检查所有文件（推荐） - 对整个仓库执行检查
2. 仅检查指定文件类型 - 只检查特定后缀的文件
```

### 11.3 外部仓库 Workflow 模板

```yaml
name: External Repo Pre-commit Check

on:
  workflow_dispatch:
    inputs:
      target_repo:
        description: 'Target repository (owner/repo or URL)'
        required: true
        default: 'Ascend/pytorch'
      target_branch:
        description: 'Target branch (default: master/main)'
        required: false
        default: ''
      config_source:
        description: 'Config file source'
        required: true
        default: 'local'
        type: choice
        options:
          - local
          - target
          - custom
      custom_config_repo:
        description: 'Custom config repo (only if config_source=custom)'
        required: false
        default: ''
      custom_config_path:
        description: 'Config file path in custom repo'
        required: false
        default: '.pre-commit-config.yaml'
      check_all_files:
        description: 'Check all files'
        required: true
        default: true
        type: boolean
      file_filter:
        description: 'File filter patterns (space separated, e.g. "*.py *.cpp")'
        required: false
        default: ''

env:
  PYTHON_VERSION: '3.11'

jobs:
  external-lint:
    name: Check external repository
    runs-on: ubuntu-22.04

    steps:
      - name: Checkout config repo (local)
        if: inputs.config_source == 'local'
        uses: actions/checkout@v4
        with:
          sparse-checkout: |
            .pre-commit-config.yaml
            .clang-format
            .editorconfig
          sparse-checkout-cone-mode: false

      - name: Checkout custom config repo
        if: inputs.config_source == 'custom'
        uses: actions/checkout@v4
        with:
          repository: ${{ inputs.custom_config_repo }}
          sparse-checkout: |
            ${{ inputs.custom_config_path }}
          sparse-checkout-cone-mode: false
          path: config-repo

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: 'pip'

      - name: Install pre-commit
        run: |
          pip install pre-commit
          pre-commit --version

      - name: Parse repository info
        id: repo_info
        run: |
          REPO_INPUT="${{ inputs.target_repo }}"

          # 解析仓库地址
          if [[ "$REPO_INPUT" == https://* ]] || [[ "$REPO_INPUT" == git://* ]]; then
            # 完整 URL
            REPO_URL="$REPO_INPUT"
            REPO_NAME=$(echo "$REPO_INPUT" | sed 's/.*github.com[\/:]\(.*\)\.git/\1/' | sed 's/.*github.com[\/:]\(.*\)/\1/')
          else
            # owner/repo 格式
            REPO_URL="https://github.com/${REPO_INPUT}.git"
            REPO_NAME="$REPO_INPUT"
          fi

          echo "repo_url=${REPO_URL}" >> $GITHUB_OUTPUT
          echo "repo_name=${REPO_NAME}" >> $GITHUB_OUTPUT

          # 设置分支
          BRANCH="${{ inputs.target_branch }}"
          if [ -z "$BRANCH" ]; then
            BRANCH="master"
          fi
          echo "branch=${BRANCH}" >> $GITHUB_OUTPUT

          echo "Target: ${REPO_NAME}@${BRANCH}"

      - name: Clone target repository
        run: |
          git clone --depth=1 --branch=${{ steps.repo_info.outputs.branch }} \
            ${{ steps.repo_info.outputs.repo_url }} target-repo

          # 记录仓库信息
          cd target-repo
          echo "commit_sha=$(git rev-parse HEAD)" >> $GITHUB_OUTPUT
          echo "commit_date=$(git log -1 --format=%cd --date=short)" >> $GITHUB_OUTPUT

          # 统计文件数量
          FILE_COUNT=$(find . -type f \( -name "*.py" -o -name "*.cpp" -o -name "*.c" -o -name "*.h" -o -name "*.hpp" -o -name "*.cu" -o -name "*.sh" \) | wc -l)
          echo "file_count=${FILE_COUNT}" >> $GITHUB_OUTPUT
        id: target_info

      - name: Prepare config file
        run: |
          CONFIG_SOURCE="${{ inputs.config_source }}"

          if [ "$CONFIG_SOURCE" = "local" ]; then
            # 使用本仓库配置
            if [ -f ".pre-commit-config.yaml" ]; then
              cp .pre-commit-config.yaml target-repo/
              echo "Using local config"
            else
              echo "Error: No .pre-commit-config.yaml found in current repo"
              exit 1
            fi
          elif [ "$CONFIG_SOURCE" = "custom" ]; then
            # 使用指定仓库配置
            CONFIG_PATH="${{ inputs.custom_config_path }}"
            if [ -f "config-repo/${CONFIG_PATH}" ]; then
              cp "config-repo/${CONFIG_PATH}" target-repo/.pre-commit-config.yaml
              echo "Using custom config from ${{ inputs.custom_config_repo }}"
            else
              echo "Error: Config file not found in custom repo"
              exit 1
            fi
          else
            # 使用目标仓库配置
            if [ -f "target-repo/.pre-commit-config.yaml" ]; then
              echo "Using target repo config"
            else
              echo "Error: No .pre-commit-config.yaml found in target repo"
              exit 1
            fi
          fi

      - name: Cache pre-commit
        uses: actions/cache@v4
        with:
          path: ~/.cache/pre-commit
          key: pre-commit-${{ env.PYTHON_VERSION }}-${{ hashFiles('target-repo/.pre-commit-config.yaml') }}
          restore-keys: |
            pre-commit-${{ env.PYTHON_VERSION }}-
            pre-commit-

      - name: Build file list
        id: files
        run: |
          cd target-repo

          if [ "${{ inputs.check_all_files }}" = "true" ]; then
            # 检查所有文件
            if [ -n "${{ inputs.file_filter }}" ]; then
              # 使用用户指定的过滤模式
              FILTER="${{ inputs.file_filter }}"
              FILES=$(find . -type f \( $(echo "$FILTER" | sed 's/ / -o -name /g' | sed 's/^/-name /') \) | tr '\n' ' ')
            else
              # 默认代码文件
              FILES=$(find . -type f \( -name "*.py" -o -name "*.cpp" -o -name "*.c" -o -name "*.h" -o -name "*.hpp" -o -name "*.cu" -o -name "*.cuh" -o -name "*.sh" -o -name "*.yaml" -o -name "*.yml" -o -name "*.json" -o -name "*.md" \) | tr '\n' ' ')
            fi
          else
            # 仅检查最近变更的文件（对于 shallow clone，检查所有文件）
            FILES=$(find . -type f \( -name "*.py" -o -name "*.cpp" -o -name "*.c" -o -name "*.h" -o -name "*.hpp" -o -name "*.cu" -o -name "*.cuh" -o -name "*.sh" -o -name "*.yaml" -o -name "*.yml" -o -name "*.json" -o -name "*.md" \) | tr '\n' ' ')
          fi

          echo "files=${FILES}" >> $GITHUB_OUTPUT
          echo "Total files to check: $(echo "$FILES" | wc -w)"

      - name: Run pre-commit
        id: lint
        run: |
          cd target-repo

          if [ -n "${{ steps.files.outputs.files }}" ]; then
            pre-commit run --all-files 2>&1 | tee /tmp/pre-commit-output.txt
          else
            echo "No files to check"
            echo "No files to check" > /tmp/pre-commit-output.txt
          fi
        continue-on-error: true

      - name: Generate report
        run: |
          echo "## Pre-commit 外部仓库检查报告" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY

          # 仓库信息表格
          echo "### 目标仓库信息" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "| 项目 | 值 |" >> $GITHUB_STEP_SUMMARY
          echo "|------|-----|" >> $GITHUB_STEP_SUMMARY
          echo "| 仓库 | ${{ steps.repo_info.outputs.repo_name }} |" >> $GITHUB_STEP_SUMMARY
          echo "| 分支 | ${{ steps.repo_info.outputs.branch }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Commit | \`${{ steps.target_info.outputs.commit_sha }}\` |" >> $GITHUB_STEP_SUMMARY
          echo "| 检查文件数 | ${{ steps.target_info.outputs.file_count }} |" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY

          # 检查结果
          echo "### 检查结果" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY
          cat /tmp/pre-commit-output.txt >> $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY

          if [ "${{ steps.lint.outcome }}" = "failure" ]; then
            echo "" >> $GITHUB_STEP_SUMMARY
            echo "### 问题汇总" >> $GITHUB_STEP_SUMMARY
            echo "" >> $GITHUB_STEP_SUMMARY
            echo "检测到代码规范问题，建议目标仓库维护者进行修复。" >> $GITHUB_STEP_SUMMARY
          else
            echo "" >> $GITHUB_STEP_SUMMARY
            echo "所有检查项均已通过。" >> $GITHUB_STEP_SUMMARY
          fi

      - name: Upload results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: external-lint-results-${{ github.run_number }}
          path: |
            /tmp/pre-commit-output.txt
          retention-days: 30

      - name: Check result
        if: steps.lint.outcome == 'failure'
        run: exit 1
```

### 11.4 外部仓库检测执行流程

当用户选择外部仓库模式时，按以下顺序执行：

1. **收集参数**：询问目标仓库、分支、配置来源等
2. **生成 Workflow**：创建 `.github/workflows/external-lint.yml`
3. **验证 Workflow**：检查 YAML 语法
4. **输出说明**：告知用户如何手动触发 workflow

### 11.5 外部仓库检测报告格式

```markdown
## Pre-commit 外部仓库检查报告

### 目标仓库信息

| 项目 | 值 |
|------|-----|
| 仓库 | Ascend/pytorch |
| 分支 | master |
| Commit | `abc123def456...` |
| 检查文件数 | 1234 |

### 检查结果

```
[检查输出日志]
```

### 问题汇总

检测到代码规范问题，建议目标仓库维护者进行修复。

#### 失败的检查项

1. **black** - Python 代码格式化
   - 文件数：15
   - 问题：代码格式不符合 PEP 8 规范

2. **flake8** - Python 代码风格检查
   - 文件数：8
   - 问题：存在未使用的导入、行过长等问题

3. **clang-format** - C/C++ 代码格式化
   - 文件数：42
   - 问题：代码格式不符合规范
```

---

## 12. Workflow 模式执行流程总结

当用户选择 Workflow 模式时：

```
用户选择 Workflow 模式
         │
         ▼
询问检测目标（本仓库 / 外部仓库）
         │
    ┌────┴────┐
    │         │
    ▼         ▼
本仓库      外部仓库
    │         │
    ▼         ▼
生成标准   收集参数
Workflow   (目标仓库、分支等)
    │         │
    ▼         ▼
输出文件   生成外部仓库
           Workflow
              │
              ▼
           输出文件
```

---

## 关键防坑（必须遵守）

### 本地模式

1. **必须检测 Git 环境**
   - 不是 Git 仓库时提示用户

2. **配置文件不要覆盖现有配置**
   - 如果 `.pre-commit-config.yaml` 已存在，询问用户是否覆盖

3. **首次检查使用 `|| true`**
   - 避免首次检查失败中断配置流程

### Workflow 模式（本仓库）

1. **必须设置 `fetch-depth: 0`**
   - 用于获取完整历史，比较变更文件

2. **检查结果步骤使用 `continue-on-error: true`**
   - 确保即使检查失败也能生成报告

3. **最后一步必须检查结果**
   - 使用 `if: steps.lint.outcome == 'failure'` 判断

4. **Artifact 上传使用 `if: always()`**
   - 确保失败时也能上传日志

### Workflow 模式（外部仓库）

1. **必须验证目标仓库可访问**
   - 公开仓库：直接克隆
   - 私有仓库：需要配置 token（提示用户）

2. **配置文件必须存在**
   - 检查配置文件是否存在，不存在则报错

3. **使用 sparse-checkout 优化**
   - 仅检出需要的文件，节省时间

4. **文件过滤必须合理**
   - 默认检查代码文件，避免检查二进制文件

5. **报告必须包含仓库信息**
   - 明确显示检查的是哪个仓库、哪个分支

---

## 推荐执行提示词

### 本地模式

"请按 `SKILL.MD` 的本地模式章节，为当前项目配置 pre-commit 代码检查环境。要求：检测项目环境、安装 pre-commit、根据项目类型生成 `.pre-commit-config.yaml`、安装 Git hooks、执行首次检查并输出结果。"

### Workflow 模式（本仓库）

"请按 `SKILL.MD` 的 Workflow 模式章节，生成 `.github/workflows/pre-commit.yml`。要求：使用 `actions/checkout@v4` 并设置 `fetch-depth: 0`、缓存 pre-commit 环境、获取变更文件列表、执行检查、生成报告、上传结果 artifact。检查失败时输出修复建议到 step summary。"

### Workflow 模式（外部仓库）

"请按 `SKILL.MD` 的外部仓库检测章节，生成 `.github/workflows/external-lint.yml`。要求：支持用户输入目标仓库地址和分支、支持选择配置文件来源（本仓库/目标仓库/指定仓库）、克隆目标仓库、执行代码检查、生成包含仓库信息的详细报告、上传结果 artifact。"

---

## 相关文件

- `.pre-commit-config.yaml` - Pre-commit 配置文件
- `.github/workflows/pre-commit.yml` - CI Workflow 定义（本仓库）
- `.github/workflows/external-lint.yml` - CI Workflow 定义（外部仓库）
- `.git/hooks/pre-commit` - Git pre-commit hook
