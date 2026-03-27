---
name: unpack-agent
description: 解包分析 Agent — 编排完整的应用解包分析流程：识别→反编译→检测→分析→文档
---

# Unpack Agent

## 功能说明

一站式应用解包分析编排器。接收应用包路径，自动编排 `app_identify` → `app_decompile` → `protection_detect` → `lib_deep_analyze` → `app_analyze` → `app_document` 的完整流程。

## 使用方法

```
/unpack <应用路径> [--topic <主题>] [--deep] [--lib <库名>]
```

- `--topic`：可选，指定专题分析主题（如 rendering、network、hotupdate、resource）
- `--deep`：可选，启用深度反编译（Ghidra 等重量级工具）
- `--lib`：可选，指定要深度分析的 native 库名

## 部署方式：主会话编排

Agent 在主会话中运行，根据每个阶段的输出决定下一步操作。部分子任务（app_identify、app_document）可以用 subagent 并行执行。

---

## 编排流程

```
/unpack <应用路径>
  │
  ▼
Phase 1: 准备 ─────────────────────────────────────────────
  │
  ├── 1.1 解析应用名和版本号（从文件名或包内配置）
  ├── 1.2 创建输出目录:
  │        /Users/jiashuyong/Downloads/App分析/<AppName_Version>/
  ├── 1.3 复制原始应用包到输出目录
  └── 1.4 执行 /toolchain-manager --check
  │        检查工具链，自动安装缺失工具
  │
  ▼
Phase 2: 识别 ─────────────────────────────────────────────
  │
  └── 执行 /app-identify <应用路径>
       输出: 识别报告（平台、引擎、技术栈、架构）
  │
  ▼
Phase 3: 反编译 ───────────────────────────────────────────
  │
  └── 执行 /app-decompile <应用路径> [--deep]
       输出: 反编译产物（01-08_*.txt + 解码目录）
  │
  ▼
Phase 4: 保护检测 ─────────────────────────────────────────
  │
  └── 执行 /protection-detect <产物目录>
       输出: protection_report.txt
  │
  ├── 读取检测报告，更新分析策略:
  │   - 哪些 native 库是正常 ELF（可深度分析）
  │   - 哪些被加壳（标注限制）
  │   - IL2CPP metadata 状态
  │   - 脚本层加密状态
  │
  ▼
Phase 5: 深度分析（条件触发）─────────────────────────────
  │
  ├── 5a. 自动选择值得深度分析的 native 库:
  │        筛选条件:
  │          - 正常 ELF（未加壳）
  │          - 大小 > 500KB（排除小型桥接库）
  │          - 非通用第三方库（排除 libssl、libcurl 等）
  │        对每个选中的库执行 /lib-deep-analyze
  │
  │   如果用户指定了 --lib:
  │        直接对指定库执行 /lib-deep-analyze
  │
  ├── 5b. 如果用户指定了 --topic:
  │        执行 /app-analyze <topic> --dir <产物目录>
  │        输出: 专题分析文档 (1x_*.md)
  │
  └── 5c. 如果没有指定 --topic:
           暂停，向用户展示可分析的方向:
           "检测完成，以下方向可以深入分析:
            1. 热更新管线 (--topic hotupdate)
            2. 资源管理 (--topic resource)
            3. 网络通信 (--topic network)
            4. 渲染管线 (--topic rendering)
            5. 指定库深度分析 (--lib <库名>)
            或输入自定义主题。"
  │
  ▼
Phase 6: 文档化（可选）────────────────────────────────────
  │
  └── 如果 Phase 5 产出了专题分析:
       执行 /app-document --dir <产物目录> --topic <主题>
       输出: 标准化文档 (part1_*.md/html + part2_*.md/html)
  │
  ▼
Phase 7: 汇总 ─────────────────────────────────────────────
  │
  └── 更新 00_summary.txt:
       - 应用基本信息
       - 所有产物文件清单（文件名、大小、行数）
       - 各层反编译状态（✓/⚠/✗）
       - 保护检测结果摘要
       - 深度分析结果摘要
       - 未完成的分析方向（供后续继续）
```

---

## 决策逻辑

### Phase 5a: 自动选库逻辑

```python
# 伪代码
for lib in normal_elf_libs:
    if lib.size < 500KB:
        skip  # 太小，可能是桥接库
    if lib.name in KNOWN_GENERIC_LIBS:
        skip  # 通用第三方库，分析价值低
    if lib.name contains game_specific_keywords:
        priority = HIGH  # 游戏自研库，优先分析
    else:
        priority = MEDIUM

# 按优先级排序，最多分析 3 个库（避免耗时过长）
selected = sorted(candidates, key=priority)[:3]
```

**KNOWN_GENERIC_LIBS（跳过列表）：**
```
libssl, libcrypto, libcurl, libz, libpng, libjpeg,
libfreetype, libharfbuzz, libicu*, libc++, libstdc++
```

**game_specific_keywords（优先关键词）：**
```
Game, Core, Engine, Battle, Apollo, GCloud,
应用名相关词（如 HOK, Smoba, Genshin 等）
```

### Phase 5c: 主题推荐逻辑

根据识别结果和检测报告，推荐最有价值的分析方向：

```
if 有热更框架 (Apollo/GCloud/Puffer):
    推荐 "热更新管线"
if 有 ResourcesVolume/Addressables:
    推荐 "资源管理"
if 有 TGCP/网络库:
    推荐 "网络通信"
if 有自研渲染相关库:
    推荐 "渲染管线"
```

---

## 数据流

```
                    ┌──────────────┐
                    │ 用户输入      │
                    │ /unpack <apk>│
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ Phase 1      │
                    │ 准备+工具链   │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ Phase 2      │
                    │ app_identify │──→ 识别报告
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ Phase 3      │
                    │ app_decompile│──→ 反编译产物
                    └──────┬───────┘
                           │
                    ┌──────▼────────┐
                    │ Phase 4       │
                    │ protection    │──→ 检测报告
                    │ _detect       │
                    └──────┬────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
     ┌────────────┐ ┌───────────┐ ┌──────────┐
     │ lib_deep   │ │ app       │ │ 暂停等待  │
     │ _analyze   │ │ _analyze  │ │ 用户指示  │
     │ (自动选库) │ │ (--topic) │ │          │
     └─────┬──────┘ └─────┬─────┘ └──────────┘
           │              │
           └──────┬───────┘
                  ▼
           ┌────────────┐
           │ app        │
           │ _document  │ (可选)
           └──────┬─────┘
                  ▼
           ┌────────────┐
           │ 00_summary │
           │ .txt 汇总  │
           └────────────┘
```

---

## 快捷模式

### 快速解包（默认）
```
/unpack app.apk
```
执行 Phase 1-4 + Phase 7，在 Phase 5 暂停等用户指示。

### 完整解包 + 专题分析
```
/unpack app.apk --topic hotupdate
```
执行 Phase 1-7 全流程。

### 深度解包 + 指定库分析
```
/unpack app.apk --deep --lib libapollo.so
```
Phase 3 启用 Ghidra，Phase 5 分析指定库。

---

## 已知的坑

| 问题 | 应对 |
|------|------|
| 大型 APK (>500MB) 解包耗时长 | Phase 3 的 apktool/jadx 用后台任务并行 |
| 工具安装需要网络 | Phase 1 提前检查，安装失败时降级 |
| 加壳库无法自动分析 | Phase 4 检测后在 Phase 5 跳过，汇总中标注 |
| 用户可能中途想换方向 | Phase 5c 暂停机制，随时可追加 --topic 或 --lib |
| context window 可能不够 | 大型分析拆分为多个 subagent，主会话只做编排 |
