---
name: toolchain-manager
description: 工具链管理 — 检测、安装、配置解包分析所需的工具链，处理版本兼容性
---

# Toolchain Manager Skill

## 功能说明

检测当前系统中解包分析工具的安装状态，自动安装缺失工具，处理版本兼容性问题。

## 使用方法

```
/toolchain-manager [--check] [--install <工具名>] [--fix <工具名>]
```

- `--check`：检查所有工具状态（默认行为）
- `--install <工具名>`：安装指定工具
- `--fix <工具名>`：修复指定工具的兼容性问题

## 部署方式：主会话直接执行

需要执行 brew/pip/curl 等安装命令。

---

## 工具注册表

### Android 解包工具

| 工具 | 用途 | 检测命令 | 安装方式 | 兼容性注意 |
|------|------|---------|---------|-----------|
| apktool | APK 解码 (smali + 资源) | `which apktool` | `brew install apktool` | 需要 JDK |
| jadx | DEX → Java 反编译 | `which jadx` | `brew install jadx` | 需要 JDK，推荐替代 dex2jar |
| dex2jar | DEX → JAR 转换 | `which d2j-dex2jar` | `brew install dex2jar` | jadx 的下位替代，一般不需要 |
| aapt2 | APK 信息提取 | `which aapt2` | Android SDK 自带 | 可选，apktool 可替代 |

### Unity 工具

| 工具 | 用途 | 检测命令 | 安装方式 | 兼容性注意 |
|------|------|---------|---------|-----------|
| Il2CppDumper | IL2CPP metadata dump | `ls /tmp/Il2CppDumper/` | GitHub Release zip 下载 | 需要 .NET 6+，见下方兼容性修复 |
| AssetStudio | Unity 资源解包 | - | GitHub Release | 需要 .NET，macOS 支持有限 |

### 原生二进制工具

| 工具 | 用途 | 检测命令 | 安装方式 | 兼容性注意 |
|------|------|---------|---------|-----------|
| Ghidra | 反编译/反汇编 | `which ghidra` | `brew install ghidra` | 需要 JDK 17+，~800MB |
| nm | 符号表提取 | `which nm` | 系统自带 | macOS 自带，Linux 需 binutils |
| strings | 字符串提取 | `which strings` | 系统自带 | - |
| c++filt | C++ 符号 demangle | `which c++filt` | 系统自带 | - |
| objdump | 反汇编 | `which objdump` | 系统自带 / `brew install binutils` | macOS 版功能有限 |

### 运行时分析工具

| 工具 | 用途 | 检测命令 | 安装方式 | 兼容性注意 |
|------|------|---------|---------|-----------|
| frida | 运行时 hook/dump | `which frida` | `pip install frida-tools` | 需要 root/jailbreak 设备 |
| adb | Android 调试桥 | `which adb` | Android SDK / `brew install android-platform-tools` | 需要 USB 调试开启 |

### 运行时环境

| 环境 | 检测命令 | 安装方式 | 版本要求 |
|------|---------|---------|---------|
| JDK | `java -version` | `brew install openjdk` | 17+ (Ghidra), 11+ (jadx/apktool) |
| .NET | `dotnet --list-runtimes` | `brew install dotnet` | 6.0+ (Il2CppDumper) |
| Python 3 | `python3 --version` | `brew install python` | 3.8+ |

---

## 执行流程

### Step 1: 检测已安装工具

```
对注册表中每个工具执行检测命令:
  - 存在 + 版本正确 → ✓
  - 存在 + 版本不兼容 → ⚠ (需要修复)
  - 不存在 → ✗ (需要安装)
```

### Step 2: 根据目标应用确定需求

```
根据 app_identify 的结果确定需要哪些工具:
  - Android APK → apktool, jadx
  - Unity IL2CPP → Il2CppDumper, .NET
  - Unity Mono → 无额外工具（直接反编译 DLL）
  - 原生二进制深度分析 → Ghidra, JDK 17+
  - 运行时分析 → frida, adb
```

### Step 3: 安装缺失工具

```
优先级:
  1. brew install (最简单)
  2. pip install (Python 工具)
  3. GitHub Release 下载 (Il2CppDumper 等)
  4. 手动指引 (Android SDK 等)

安装失败时:
  - 记录错误信息
  - 提供手动安装指引
  - 标注哪些功能会受影响
```

### Step 4: 兼容性修复

已知兼容性问题及自动修复：

**Il2CppDumper .NET 版本不匹配：**
```json
// 修改 Il2CppDumper.runtimeconfig.json
// 添加 "rollForward": "LatestMajor"
{
  "runtimeOptions": {
    "rollForward": "LatestMajor",
    ...
  }
}
```

**Ghidra JDK 版本：**
```bash
# 检测 brew 安装的 openjdk
export JAVA_HOME=$(/usr/libexec/java_home -v 17 2>/dev/null || /usr/libexec/java_home)
```

**pip PEP 668 限制（macOS）：**
```bash
# 使用 pipx 或 venv 替代直接 pip install
pipx install frida-tools
# 或
python3 -m venv /tmp/tools_env && /tmp/tools_env/bin/pip install frida-tools
```

### Step 5: 输出工具链状态报告

```
=== 工具链状态报告 ===

✓ apktool 2.11.1
✓ jadx 1.5.5
✓ nm (系统自带)
✓ strings (系统自带)
✓ c++filt (系统自带)
✓ .NET 8.0.13 + 9.0.2
⚠ Il2CppDumper 6.7.40 (已修复 rollForward)
✗ Ghidra (未安装，需要时执行 brew install ghidra)
✗ frida (未安装，需要时执行 pipx install frida-tools)

可用功能:
  ✓ Android APK 反编译
  ✓ Unity IL2CPP dump
  ✓ Native 库符号分析
  ⚠ 原生二进制深度反编译 (需安装 Ghidra)
  ✗ 运行时 hook/dump (需安装 frida + root 设备)
```

---

## 已知的坑

| 问题 | 应对 |
|------|------|
| brew install 网络超时 | 后台执行，设置较长超时，失败后重试 |
| GitHub Release 下载慢 | 使用 curl -L --max-time 120，失败后提示手动下载 |
| macOS 系统 Python 被 PEP 668 保护 | 用 pipx 或 venv，不要 --break-system-packages |
| Il2CppDumper 不在 NuGet 中 | 只能从 GitHub Release 下载 zip，不要用 dotnet tool install |
| Ghidra 首次运行需要同意协议 | headless 模式自动跳过 |
| 多个 JDK 版本共存 | 用 /usr/libexec/java_home -v 指定版本 |
