---
name: app-decompile
description: 应用反编译 — 根据应用识别结果选择最优工具链，对可执行文件进行反编译，输出结构化产物
---

# App Decompile Skill

## 功能说明

根据 `/app-identify` 的识别结果，选择最优的反编译策略，输出结构化的反编译产物。

## 使用方法

```
/app-decompile <应用路径> [--output <输出目录>] [--deep]
```

- `--deep`：启用深度反编译（可能需要安装工具、运行时间较长）

## 部署方式：主会话直接执行

需要处理工具安装、环境配置、长时间运行等问题，主会话更可控。

---

## 核心决策原则

### 原则 1: 识别驱动，不要盲目 Ghidra

先执行 `/app-identify`（或快速识别），根据结果选择工具。Ghidra 伪 C 代码只在原生 C/C++ 自研引擎场景下才是主力。对于有高级语言层的应用，应该优先还原高级语言代码。

### 原则 2: 脚本层优先于原生层

很多应用的架构是"原生引擎壳 + 脚本业务逻辑"。反编译的重心应该放在脚本层：

- Lua（xLua/toLua/SLua/UnLua）、JavaScript（Cocos/小游戏）、Python（Ren'Py）等脚本往往包含核心业务逻辑
- C# 热更 DLL（HybridCLR/ILRuntime）不在 Managed/ 目录，而是藏在 AssetBundle 或资源包里
- Unity Mono 的 Assembly-CSharp.dll 可以直接反编译出几乎完整的 C# 源码
- 原生二进制在这些场景下只是"引擎壳"，Ghidra 分析它的性价比很低

### 原则 3: 一个应用可能有多个层次

不要只走一条分支。典型组合：
- Android APK = Java 壳 + Unity IL2CPP + xLua 脚本
- Unreal 手游 = 原生引擎 + UnLua 脚本 + .pak 资源
- Cocos Creator = 原生引擎 + JavaScript 业务逻辑

每个层次都应该提取和分析。

### 原则 4: 产出结构标准化

无论走哪条路径，所有产物统一输出到 `/Users/jiashuyong/Downloads/App分析/<AppName_Version>/` 目录下。

**执行流程：**
1. 先在 `/Users/jiashuyong/Downloads/App分析/` 下创建以 `<应用名>_<版本号>` 命名的目录
2. 将原始应用包复制到该目录下
3. 所有反编译产物、分析文档、中间文件都输出到该目录
4. 不使用 /tmp，直接在目标目录下工作

**目录结构：**

```
/Users/jiashuyong/Downloads/App分析/<AppName_Version>/
├── <原始应用包>              # 原始 APK/IPA/EXE 等
├── 00_summary.txt            # 反编译汇总报告
├── 01_file_info.txt          # 文件格式、架构
├── 02_dependencies.txt       # 库依赖
├── 03_strings.txt            # 可读字符串
├── 04_*                      # 因技术栈而异的主要产物
├── 05_*
├── ...
├── 1x_*.md                   # 专题分析文档（热更管线、资源管理等）
├── apk_extracted/            # 原始解压目录
├── 04_apktool_decoded/       # apktool 解码（Android）
├── 05_jadx_java/             # jadx Java 反编译（Android）
├── 06_il2cpp_dump/           # IL2CPP dump（Unity）
├── 09_lib<name>/             # 单个库的深度分析目录
└── ExportAllDecompiled.java  # 仅 Ghidra 路径
```

文件编号从 01 开始递增，04 及之后的内容因技术栈而异，由模型根据实际情况决定。

### 原则 5: 工具不可用时降级而非中断

如果最优工具不可用（未安装、平台不支持）：
1. 先尝试自动安装（brew/pip/npm）
2. 安装失败则降级到次优方案（如 Ghidra 不可用就只输出 strings + 汇编）
3. 在输出汇总中标注哪些产物因工具缺失而未生成

---

## 前置依赖

- `app_identify`：需要先执行应用识别，获取平台、引擎、技术栈信息
- 如果没有识别结果，提示用户先执行 `/app-identify`

## 后续衔接

- `protection_detect`：反编译完成后，建议执行加壳/加密检测
- `lib_deep_analyze`：对正常 ELF 库可执行单库深度分析
- `app_analyze`：基于反编译产物进行专题分析

## 执行流程

### Step 0: 确认识别结果

从 `app_identify` 的输出中获取：
1. 平台和架构
2. 引擎/框架
3. 需要执行的层次组合

如果当前会话没有识别结果，提示用户先执行 `/app-identify`。

### Step 1: 通用基础产物

所有路径都执行：文件信息、库依赖、字符串提取。

### Step 2: 按层次执行反编译

根据识别结果，对每个层次选择最优工具执行反编译。模型应根据自身知识选择具体工具和命令，不需要查表。

关键判断点：
- 有高级语言 DLL/脚本可直接反编译？→ 优先走高级语言反编译
- 脚本是明文还是字节码还是加密？→ 决定是直接提取、反编译还是标注无法处理
- 原生二进制是否值得 Ghidra 分析？→ 只有自研引擎或需要分析 native 层时才值得
- 资源包需要解包吗？→ .pak/.pck/.ab 等可能包含脚本或关键配置

### Step 3: 输出汇总

列出所有产物文件、大小、使用的工具，标注：
- 符号表状态（正常/已strip）
- 各层次的反编译完成度
- 因工具缺失未生成的产物

---

## 已知的坑（来自实践）

| 问题 | 应对 |
|------|------|
| Ghidra 需要 JDK 17+，系统自带的 JDK 版本不够 | 检测 brew 安装的 openjdk，设置 JAVA_HOME |
| Ghidra headless 模式找不到自定义脚本 | 用 `-scriptPath` 指向脚本所在目录 |
| Ghidra 内置没有 DecompileAllFunctions.java | 需要自己写，遍历所有函数调用 DecompInterface |
| 二进制文件路径有空格 | 所有路径用双引号包裹 |
| 反汇编/伪代码文件可能几十 MB | 后台执行，不阻塞 |
| HTML/大文件 Write 工具会失败 | 用 Bash `cat >>` 分段追加写入 |
| Lua 字节码版本不兼容 | 先检查 .luac 文件头判断 Lua 版本，再选对应版本的反编译器 |
| IL2CPP 的 global-metadata.dat 可能被加密 | 由 `protection_detect` 检测并给出处理建议 |
| C# 热更 DLL 藏在 AB 包里 | 需要先用 AssetStudio/UABE 从 AB 中提取 |
| .jsc 字节码没有通用反编译器 | 降级为 strings 提取，标注限制 |
| 腾讯 `_cp.so` 加壳 (magic `8c0c00xx`) | 由 `protection_detect` 检测。不是 ELF，需运行时 dump |
| Il2CppDumper 需要 .NET 6 但系统只有 8/9 | 修改 runtimeconfig.json 添加 `"rollForward": "LatestMajor"` |
| Il2CppDumper 在 NuGet 中不存在 | 从 GitHub Release 下载 zip，不要用 dotnet tool install |
| global-metadata.dat 有自定义头 | 由 `protection_detect` 检测偏移位置并尝试修复 |
| ResourcesVolume .db 格式无公开工具 | 腾讯定制格式，只能 strings 提取资源路径，完整解析需专用工具 |

## 注意事项

1. `--deep` 模式可能需要安装工具（Ghidra ~800MB、jadx、Il2CppDumper 等），首次运行需要网络和时间
2. Ghidra 分析大型二进制可能需要 5-10 分钟，使用后台任务执行
3. 不要对所有应用都跑 Ghidra，大部分情况下有更高效的专用工具
4. 加密/混淆的脚本无法通用处理，标注限制即可，不要强行破解
