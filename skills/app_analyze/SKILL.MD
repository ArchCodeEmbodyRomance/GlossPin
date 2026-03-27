---
name: app-analyze
description: 主题分析 — 基于反编译产物，针对用户指定的主题（渲染、网络、物理等）进行深度逆向分析，还原系统架构
---

# App Analyze Skill

## 功能说明

基于 `/app-decompile` 产出的反编译产物，针对用户指定的技术主题进行深度分析。通过"关键词搜索 → 上下文阅读 → 交叉验证 → 架构还原"的方法论，从反编译代码中还原出目标系统的完整技术方案。

## 使用方法

```
/app-analyze <主题> [--dir <反编译产物目录>]
```

参数说明：
- `<主题>`：必需，分析主题关键词，如 `terrain`、`rendering`、`network`、`audio`、`physics`、`ui`、`animation`、`ai`
- `--dir`：可选，反编译产物目录路径，默认自动查找当前项目中的 `decompile_output/`

例如：
- `/app-analyze terrain` — 分析地表/地形系统
- `/app-analyze rendering --dir /path/to/decompile_output` — 分析渲染管线
- `/app-analyze network` — 分析网络通信系统
- `/app-analyze audio` — 分析音频系统

## 设计来源

在 The Witness 地表分析中，实际使用的方法是：
1. 先在 `07_strings.txt` 中搜索主题相关关键词（terrain、mesh、grass、shader、lightmap...）
2. 找到关键词后，在 `08_decompiled_pseudocode.c` 中定位对应函数
3. 阅读函数上下文，理解数据结构和调用关系
4. 交叉验证：资源文件路径、材质文件、配置文件互相印证
5. 逐步拼出完整的系统架构

这个"由字符串入手，逐步展开"的模式适用于任何主题的逆向分析。

## 部署方式：主会话直接执行

本 skill 在主会话中直接执行，原因：
- 分析是迭代式的，每一步搜索的结果决定下一步搜索的方向
- 需要大量 context 积累，subagent 的 context window 不够
- 用户可能在过程中追问或调整分析方向

## 执行流程

### Step 1: 确认产物

动态扫描反编译产物目录，根据实际存在的文件类型确定分析策略。

**支持的输入源（按优先级）：**

| 输入源 | 路径模式 | 用途 |
|--------|---------|------|
| jadx Java 源码 | `05_jadx_java/sources/` | Java 层业务逻辑、SDK 调用 |
| smali 代码 | `04_apktool_decoded/smali*/` | Java 层完整代码（混淆后） |
| Native 库分析报告 | `09_lib*/report.txt` | 已分析的 native 库模块架构 |
| Native 库导出函数 | `09_lib*/exports.txt` | native 层 API 接口 |
| Native 库字符串 | `09_lib*/strings_raw.txt` | native 层字符串线索 |
| IL2CPP dump | `06_il2cpp_dump/` | C# 类型/方法签名（如果 dump 成功） |
| IL2CPP metadata 字符串 | `06_il2cpp_metadata.txt` | C# 层部分可读字符串 |
| 配置文件 | `07_configs.txt` 或 `apk_extracted/assets/Config/` | 游戏配置、服务端点 |
| Unity 资源信息 | `08_unity_assets.txt` | Unity 资源结构 |
| 保护检测报告 | `protection_report.txt` | 哪些层可分析、哪些被加密 |
| 通用字符串 | `03_strings.txt` | 从 native 库提取的字符串 |
| Ghidra 伪代码 | `08_decompiled_pseudocode.c` | 原生二进制反编译（如果有） |
| 库依赖 | `02_dependencies.txt` | 技术栈线索 |
| ScriptingAssemblies.json | `apk_extracted/assets/bin/Data/` | Unity C# 程序集列表 |
| RuntimeInitializeOnLoads.json | `apk_extracted/assets/bin/Data/` | Unity 初始化回调链 |

**扫描逻辑：**
1. 列出产物目录下所有文件和子目录
2. 匹配上表中的路径模式，确定可用输入源
3. 如果有 `protection_report.txt`，先读取了解哪些层被加密（避免在加密层上浪费时间）
4. 根据可用输入源调整分析策略：
   - 有 jadx 源码 → 优先在 Java 源码中搜索
   - 有 native 库报告 → 直接引用已分析的模块架构
   - 有 IL2CPP dump → 在 C# 类型签名中搜索
   - 只有 strings → 降级为字符串驱动分析
5. 如果关键输入源缺失，提示用户先执行对应的 skill

### Step 2: 关键词矩阵搜索

根据用户指定的主题，使用预定义的关键词矩阵在所有可用输入源中搜索。

**多源搜索策略：**
- jadx Java 源码：`grep -r` 搜索 `05_jadx_java/sources/` 目录
- Native 库导出函数：搜索 `09_lib*/exports.txt`
- Native 库字符串：搜索 `09_lib*/strings_raw.txt`
- IL2CPP metadata：搜索 `06_il2cpp_metadata.txt`
- 配置文件：搜索 `07_configs.txt` 和 `apk_extracted/assets/Config/`
- 通用字符串：搜索 `03_strings.txt`
- Ghidra 伪代码：搜索 `08_decompiled_pseudocode.c`（如果存在）
- Unity 程序集：搜索 `ScriptingAssemblies.json` 和 `RuntimeInitializeOnLoads.json`

#### 主题关键词矩阵

**terrain / 地表地形**：
```
terrain, heightmap, height_map, ground, landscape, mesh, triangle,
vertex, LOD, lightmap, shader, grass, rock, sand, dirt, cliff, slope,
normal_map, displacement, tessellation, splat, blend, cluster, chunk,
tile, patch, world_block, surface, foliage, vegetation
```

**rendering / 渲染管线**：
```
shader, render, prepass, gbuffer, deferred, forward, shadow, shadow_map,
cascade, bloom, hdr, tonemap, post_process, anti_alias, SMAA, FXAA, TAA,
depth, stencil, framebuffer, render_target, draw_call, batch, instanc,
occlusion, cull, frustum, LOD, dissolve, translucent, transparent, alpha,
reflection, refraction, environment_map, cubemap, probe, irradiance
```

**network / 网络通信**：
```
socket, connect, send, recv, packet, protocol, serialize, deserialize,
RPC, reliable, unreliable, channel, session, lobby, matchmak, sync,
replicate, latency, ping, timeout, reconnect, handshake, encrypt,
compress, bandwidth, server, client, peer, host
```

**audio / 音频系统**：
```
audio, sound, music, voice, mixer, channel, listener, emitter, source,
reverb, effect, volume, pitch, fade, loop, stream, buffer, sample,
frequency, FMOD, Wwise, OpenAL, spatial, 3d_audio, occlusion, bank
```

**physics / 物理系统**：
```
physics, rigid_body, collider, collision, raycast, trigger, joint,
constraint, gravity, velocity, force, torque, friction, restitution,
broad_phase, narrow_phase, sweep, overlap, character_controller,
nav_mesh, pathfind, avoidance
```

**ui / 界面系统**：
```
widget, button, panel, label, text, image, layout, canvas, screen,
input, touch, gesture, cursor, focus, scroll, list, grid, animation,
tween, transition, font, glyph, atlas, localize, i18n
```

**animation / 动画系统**：
```
animation, animator, skeleton, bone, joint, pose, blend_tree, state_machine,
transition, clip, keyframe, curve, IK, inverse_kinematic, ragdoll,
morph, blend_shape, skin, weight, retarget
```

如果用户指定的主题不在预定义列表中，根据主题语义动态构建关键词列表。

### Step 3: 字符串分析

1. 用关键词矩阵在 `07_strings.txt` 中搜索，记录所有匹配行
2. 对匹配结果分类：
   - **资源路径**：如 `worlds/%s/clusters/%d.triangle_mesh` → 揭示资源组织方式
   - **参数名**：如 `grass_fade_begin`, `cluster_distance` → 揭示可配置参数
   - **错误信息**：如 `"Cluster has missing/bad mesh"` → 揭示异常处理逻辑
   - **调试输出**：如 `"%5d clusters\n"` → 揭示运行时统计
   - **Shader 名**：如 `prepass_grass`, `shadow_map` → 揭示渲染管线
   - **类型/枚举**：如 `TRANSLUCENT_OPTIONS_*` → 揭示类型系统

### Step 4: 伪代码深入

基于 Step 3 发现的关键字符串，在 `08_decompiled_pseudocode.c` 中定位：

1. 搜索字符串引用，找到使用它们的函数
2. 阅读函数上下文（前后各 100-200 行），理解：
   - 数据结构布局（通过偏移量推断字段）
   - 函数调用链（谁调用了谁）
   - 初始化顺序（揭示系统启动流程）
   - 条件分支（揭示模式/变体）
3. 对关键函数建立调用图

### Step 5: 交叉验证

利用其他产物文件交叉验证发现：
- `04_dylib_dependencies.txt` — 确认第三方库的使用
- `05_objc_info.txt` — 确认 ObjC 类层次结构
- 应用包内的资源文件（材质文件、配置文件等）— 确认参数和格式

### Step 6: 架构还原

将所有发现整合为结构化的分析结论：

1. **核心数据结构**：推断出的 struct 布局和字段含义
2. **系统组成**：子系统划分和职责
3. **处理流程**：关键流程的步骤分解
4. **参数体系**：可配置参数及其作用
5. **资源组织**：资源文件格式和加载方式
6. **设计选型**：识别出的技术选择及其取舍

### Step 7: 输出分析结论

以结构化文本输出分析结论，供用户审阅或供 `/app-document` 生成正式文档。

## 分析技巧

### 从字符串反推架构

字符串是逆向分析中最有价值的线索，因为：
- 错误信息暴露了异常路径和边界条件
- 格式化字符串（`%d`, `%s`, `%.3f`）暴露了数据类型
- 资源路径暴露了文件组织和命名规范
- 参数名暴露了可配置项和设计意图
- 调试输出暴露了运行时状态和统计

### 从偏移量推断结构体

伪代码中的 `*(type *)(ptr + 0xNN)` 模式揭示了结构体布局：
- 连续的偏移量 → 连续的字段
- 偏移量间距 → 字段大小（4=int/float, 8=pointer/long, 16=vec4）
- 相同基址不同偏移 → 同一结构体的不同字段

### 从 strcmp 链推断类型系统

连续的 `strcmp(name, "xxx")` 调用揭示了类型判定逻辑：
- 每个分支对应一种类型或模式
- 分支内的赋值揭示了类型 ID 或标志位

## 注意事项

1. 分析是迭代式的，不要试图一次搜索所有关键词，而是根据每步发现调整方向
2. 伪代码中的函数名是 `FUN_XXXXXXXX` 形式（符号被strip），需要通过上下文推断功能
3. 大型伪代码文件（几十MB）不要整体读取，用 grep/sed 定位后读取局部
4. 分析结论应标注置信度：确定（有直接证据）/ 推测（间接推断）/ 未知
5. 如果某个子系统的信息不足以还原架构，在结论中标注"信息不足"而非猜测
