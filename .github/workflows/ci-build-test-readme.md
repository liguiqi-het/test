# CI Build Test Workflow 修改说明报告

## 概述

本报告详细说明了 `.github/workflows/ci-build-test.yml` 文件从初始版本到当前版本的修改内容、原因以及技术背景。

**修改目标**：使 CI 流水线在 Windows 和 Linux 平台上都能正常运行。

---

## 修改内容概览

| # | 修改项 | 类型 | 影响平台 |
|---|--------|------|----------|
| 1 | 添加 `shell: bash` 到所有 Unix 风格命令 | 兼容性修复 | Windows + Linux |
| 2 | 添加 `Install jq (Windows)` 步骤 | 功能增强 | Windows |

---

## 详细修改说明

### 修改 #1: 添加 `shell: bash` 指令

#### 变更对比

```diff
# Install Conan 步骤
   - name: Install Conan
+    shell: bash
     run: |
       pip install conan

# Get package name 步骤
   - name: Get package name
     id: read_meta
+    shell: bash
     run: |
       pkg_name=$(jq -r '.name | tostring' metadata.json)
       echo "pkg_name=$pkg_name" >> $GITHUB_OUTPUT

# Configure Conan 步骤
   - name: Configure Conan
+    shell: bash
     run: |
       echo "CONAN_HOME=${{ env.CONAN_HOME }}" >> $GITHUB_ENV
       conan profile detect --force

# Build and test with Conan 步骤
   - name: Build and test with Conan
+    shell: bash
     run: |
       conan create . -pr:b=default -pr:h=default -s build_type=${{ matrix.build_type }} --build=missing
     timeout-minutes: 30

# Clean up test_package builds 步骤
   - name: Clean up test_package builds
+    shell: bash
     run: |
       python ./test_package/conanfile.py

# Clean up Conan packages 步骤
   - name: Clean up Conan packages
     if: always()
+    shell: bash
     run: |
       conan remove "${{ steps.read_meta.outputs.pkg_name }}/*" --confirm
```

#### 修改原因

**问题背景**：GitHub Actions 在不同操作系统上使用不同的默认 shell：

| 平台 | 默认 Shell | 语法特点 |
|------|------------|----------|
| `ubuntu-latest` | bash | Unix shell 语法，支持 `~`, `>>`, `$VAR` |
| `windows-latest` | PowerShell | 完全不同的语法，不支持上述 Unix 语法 |

**原始问题**：
1. Windows PowerShell 无法解析 `>> $GITHUB_OUTPUT` 语法
2. Windows PowerShell 不支持 `$(command)` 命令替换
3. `jq` 命令在 Windows 上的路径和调用方式不同

**解决方案**：
通过显式指定 `shell: bash`，强制所有步骤使用 Git Bash（Windows 上预装），确保：
- ✅ 所有平台使用统一的 shell 语法
- ✅ 命令替换 `$()` 正常工作
- ✅ 重定向操作符 `>>` 正常工作
- ✅ `~` 路径展开在 bash 中正常工作

#### 技术细节

**Windows 环境下的 Git Bash**：
- GitHub Actions 的 `windows-latest` 镜像预装了 Git for Windows
- Git Bash 位于 `C:\Program Files\Git\bin\bash.exe`
- 支持大多数 Unix 命令和语法
- 可以通过 `shell: bash` 在任何步骤中启用

---

### 修改 #2: 添加 jq 安装步骤（Windows）

#### 变更对比

```diff
+  - name: Install jq (Windows)
+    if: matrix.os == 'windows-latest'
+    shell: bash
+    run: |
+      choco install jq -y
```

#### 修改原因

**问题背景**：
- `jq` 是一个轻量级的 JSON 处理命令行工具
- Linux 环境通常预装或可通过 `apt` 安装
- Windows 默认不包含 `jq`

**原始问题**：
```yaml
- name: Get package name
  id: read_meta
  run: |
    pkg_name=$(jq -r '.name | tostring' metadata.json)  # Windows: jq not found
```

**解决方案**：
在 Windows 上使用 Chocolatey（`windows-latest` 镜像预装）安装 `jq`：
```bash
choco install jq -y
```

#### 为什么 Linux 不需要此步骤？

Linux 环境下 `jq` 的可用性取决于 GitHub Actions 镜像：
- `ubuntu-latest` 镜像已预装 `jq`
- 无需额外安装步骤

---

## 其他评审决策说明

### 保持 `CONAN_HOME: ~/.conan2` 的原因

在初步修改中，曾建议改为 `CONAN_HOME: ${{ github.workspace }}/.conan2`，但评审后决定保持原值。

**原因分析**：
- 由于所有步骤都使用了 `shell: bash`
- 在 bash 环境中，`~` 能够正确展开为用户主目录
- Windows: `~` → `C:\Users\runner`
- Linux: `~` → `/home/runner`

**结论**：配合 `shell: bash`，原配置已可在 Windows 上正常工作。

### 保持 `python ./test_package/conanfile.py` 的原因

在初步修改中，曾建议改为 `conan test . test_package`，但评审后决定保持原值。

**原因分析**：
- 该脚本可能是项目特定的测试逻辑
- 原有方式已在项目中验证可用
- 修改可能影响项目特定的测试流程

**结论**：保持项目原有测试执行方式。

---

## 验证结果

### Windows 平台

| 步骤 | 状态 | 说明 |
|------|------|------|
| Checkout | ✅ | 正常 |
| Set up Python | ✅ | 正常 |
| Install Conan | ✅ | bash 环境下 pip 正常工作 |
| Install jq | ✅ | choco 安装成功 |
| Get package name | ✅ | jq 命令可用 |
| Configure Conan | ✅ | bash 语法正常解析 |
| System dependencies | ⏭️ | 跳过（仅在 Ubuntu） |
| Build and test | ✅ | conan create 正常执行 |
| Clean up | ✅ | 清理命令正常执行 |

### Ubuntu 平台

所有步骤正常运行，无兼容性问题。

---

## 技术参考

### GitHub Actions Shell 文档

- 官方文档：https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepsshell
- 支持的 shell 类型：
  - `bash` - Linux/macOS/Windows 上的 Bash shell
  - `pwsh` - PowerShell Core
  - `python` - Python 脚本
  - `cmd` - Windows CMD

### 跨平台 CI 最佳实践

1. **显式指定 shell**：避免依赖默认 shell
2. **使用环境变量**：`$GITHUB_OUTPUT` 优于文件写入
3. **条件安装依赖**：使用 `if: matrix.os == '...'`
4. **测试路径分隔符**：避免硬编码路径

---

## 总结

本次修改通过以下两个核心变更，解决了 Windows 平台兼容性问题：

1. **添加 `shell: bash`**：确保所有步骤使用统一的 Unix shell 语法
2. **安装 Windows 依赖**：通过 Chocolatey 安装 `jq` 工具

这些修改使得 CI 流水线能够在 Ubuntu 和 Windows 平台上无差异运行，提高了项目的跨平台兼容性。

---

**生成时间**：2025-01-15
**文档版本**：1.0
