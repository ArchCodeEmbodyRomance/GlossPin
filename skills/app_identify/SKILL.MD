---
name: app-identify
description: 应用识别 — 分析可执行文件/应用包，识别开发工具、引擎、技术栈、依赖库
---

# App Identify Skill

## 功能说明

分析任意平台的可执行文件或应用包，快速识别其开发工具、游戏引擎、编程语言、渲染API、第三方库依赖等技术栈信息。输出结构化的识别报告。

## 使用方法

```
/app-identify <可执行文件或应用包路径>
```

例如：
- `/app-identify "/Applications/Game.app"`
- `/app-identify "/path/to/game.exe"`
- `/app-identify "/path/to/game.apk"`
- `/app-identify "/path/to/game"` （Linux ELF）

## 设计来源

在 The Witness 解包分析中，通过 `file`、`otool -L`、`strings`、`Info.plist` 等手段，在几分钟内判断出：自研引擎 + C++ + SDL2 + Metal/OpenGL + Bink2 + Steamworks + Xcode 8 编译。这个"多维度交叉验证"的识别模式适用于任何应用。

## 部署方式：Subagent

本 skill 通过 Agent tool 启动独立 subagent 执行，不占用主会话 token。

主会话触发后：
1. 确定目标文件路径和平台类型
2. 启动 subagent，注入路径和下方执行流程
3. subagent 完成后，将识别报告返回用户

subagent 配置：
- `subagent_type`: `general-purpose`
- `description`: `应用识别 — <文件名>`

## 执行流程

### Step 1: 平台判定

根据文件扩展名和 `file` 命令判定平台：

| 特征 | 平台 | 后续工具链 |
|------|------|-----------|
| `.app` 目录 / Mach-O | macOS | otool, strings, plutil |
| `.exe` / PE32 | Windows | strings, objdump (如有) |
| `.apk` / `.aab` | Android | aapt2, unzip, strings |
| ELF | Linux | readelf, ldd, strings |
| `.ipa` | iOS | unzip → 同 macOS |

### Step 2: 基础信息提取

所有平台通用：
1. `file <binary>` — 架构、位数、格式
2. `strings <binary> | head -5000` — 提取可读字符串（限制行数避免过大）

macOS 专用：
3. 读取 `Info.plist` — BundleIdentifier、版本、Xcode版本、SDK版本、最低系统要求
4. `otool -L <binary>` — 动态库依赖列表
5. `otool -h <binary>` — Mach-O 头信息
6. `nm -demangle <binary> 2>/dev/null | head -200` — 符号表（可能被strip）

Windows 专用（如有工具）：
3. `objdump -p <exe> | head -100` — PE 头信息
4. 检查导入表中的 DLL 依赖

Android 专用：
3. `unzip -l <apk>` — 包内文件结构
4. 读取 `AndroidManifest.xml`（通过 aapt2）
5. 检查 `lib/` 目录下的 .so 文件

### Step 3: 引擎识别

在字符串和依赖库中搜索引擎特征：

| 引擎 | 字符串特征 | 库特征 | 文件特征 |
|------|-----------|--------|---------|
| Unity | `UnityEngine`, `mono`, `il2cpp` | `libunity.so`, `UnityPlayer.dll` | `globalgamemanagers`, `level*`, `sharedassets*` |
| Unreal | `UE4`, `UnrealEngine`, `Epic Games` | `UE4-*`, `libUE4.so` | `.pak`, `.uasset`, `.umap` |
| Godot | `GodotEngine`, `gdscript` | `libgodot*` | `.pck`, `.import` |
| Cocos2d-x | `cocos2d`, `CCDirector` | `libcocos2d*` | `.csb`, `.plist` |
| GameMaker | `GameMaker`, `YoYo Games` | — | `data.win`, `game.ios` |
| CryEngine | `CryEngine`, `CRYTEK` | `CrySystem*` | `.pak`, `.cgf` |
| 自研引擎 | 无上述特征 | 仅系统库+第三方库 | 自定义资源格式 |

### Step 4: 技术栈识别

在字符串和依赖库中搜索：

**渲染 API**：
- `OpenGL`, `glBindTexture`, `GL_ARB_*` → OpenGL
- `Metal`, `MTLDevice`, `MTLRenderPipeline` → Metal
- `Vulkan`, `vkCreateDevice` → Vulkan
- `Direct3D`, `D3D11`, `d3d12.dll` → DirectX
- `SDL_Renderer` → SDL 渲染

**平台抽象**：
- `SDL_Init`, `SDL_CreateWindow` → SDL2
- `GLFW`, `glfwCreateWindow` → GLFW
- `SFML` → SFML

**音视频**：
- `libBink*`, `.bk2` → Bink (RAD Game Tools)
- `FMOD`, `fmod.dll` → FMOD
- `Wwise`, `AkSoundEngine` → Wwise
- `OpenAL`, `alSourcePlay` → OpenAL

**网络/平台**：
- `libsteam_api` → Steamworks
- `EOS`, `EpicOnlineServices` → Epic Online Services
- `libcurl` → cURL

**编程语言**：
- `libc++`, `libstdc++` → C++
- `mono`, `mscorlib` → C# (Mono)
- `il2cpp` → C# (IL2CPP)
- `libjvm`, `java.lang` → Java
- `libpython` → Python

**编译器**：
- `DTCompiler: com.apple.compilers.llvm.clang` → LLVM Clang
- `GCC:` → GCC
- `MSVC`, `_MSC_VER` → MSVC

### Step 5: 输出报告

输出格式：

```
## 应用识别报告

**文件**：<路径>
**平台**：<macOS/Windows/Android/Linux/iOS>
**架构**：<x86_64/arm64/universal>
**格式**：<Mach-O/PE32/ELF/APK>

### 引擎
<引擎名称> 或 自研引擎

### 技术栈
| 维度 | 识别结果 | 证据 |
|------|---------|------|
| 编程语言 | ... | ... |
| 编译器 | ... | ... |
| 渲染 API | ... | ... |
| 平台抽象 | ... | ... |
| 音视频 | ... | ... |
| 网络/平台 | ... | ... |

### 依赖库
<列出所有动态库依赖>

### 构建信息
<Xcode版本、SDK版本、最低系统要求等>

### 其他发现
<符号是否被strip、是否有调试信息、特殊资源格式等>
```

## 注意事项

1. 字符串提取限制在前 5000 行，避免大文件导致输出过大
2. 符号表可能被 strip，此时依赖字符串和库依赖做判断
3. 如果是 Universal Binary（macOS），需要指定架构分析
4. Android APK 需要先解压才能分析 native 库
5. 识别结果是"最佳推测"，不是100%确定，报告中应标注置信度
