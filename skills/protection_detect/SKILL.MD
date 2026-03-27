---
name: protection-detect
description: 加壳/加密检测 — 检测应用中的保护/加密/混淆机制，输出检测报告和处理建议
---

# Protection Detect Skill

## 功能说明

检测反编译产物中的各类保护机制（加壳、加密、混淆），输出结构化检测报告和处理建议。

## 使用方法

```
/protection-detect <产物目录>
```

## 部署方式：主会话直接执行

需要读取二进制文件头、计算熵值、分析文件结构，主会话更可控。

## 前置依赖

- `app_decompile`：需要解压后的文件目录（至少有 `apk_extracted/` 或等效目录）

---

## 检测矩阵

按优先级执行以下检测项，每项独立输出结果。

### 1. Native 库加壳检测

扫描所有 `.so` 文件，检查 ELF magic `7f454c46`：

| 特征 | 判定 | 处理建议 |
|------|------|---------|
| magic = `7f454c46` | 正常 ELF | 可直接分析（nm/strings/Ghidra） |
| magic = `8c0c00xx` | 腾讯 ACE 加壳（`_cp.so` 模式） | 需运行时 dump（Frida hook dlopen） |
| 文件头含 `UPX!` | UPX 壳 | 可直接 `upx -d` 脱壳 |
| 其他非 ELF magic | 未知加壳或加密数据 | 标注 magic 值，需进一步分析 |

**命名模式检测：**
- `lib*_cp.so` → 高概率加壳（腾讯 Custom Packed）
- `lib*.csb.so` → 高概率加密脚本/资源包（伪装成 .so）

**输出分类：**
- `normal_elf`: 正常 ELF 库列表（可分析）
- `packed_so`: 加壳库列表（需脱壳）
- `encrypted_data`: 加密数据文件列表

### 2. IL2CPP Metadata 检测

检查 `global-metadata.dat`（Unity IL2CPP 应用）：

```
检测流程:
1. 搜索 magic AF1BB1FA 的偏移位置
   - offset 0: 正常
   - offset N: 有 N 字节自定义头，需去掉
   - 未找到: 整体加密，无法静态处理

2. 检查 version 字段（magic 后 4 字节）
   - 合理范围: 24-31 (对应 Unity 2019-2023+)
   - 超出范围: 可能被篡改

3. 检查偏移表（version 后的 offset/count 对）
   - 值在文件大小范围内: 正常
   - 值明显异常（如 0x07230000 对 7.4MB 文件）: 偏移表被混淆
   - 全零或全 FF: 被清空

4. 尝试 Il2CppDumper 解析
   - 成功: 输出 dump 结果
   - OverflowException: 偏移表加密
   - 其他错误: 记录错误信息
```

**输出判定：**
- `metadata_normal`: 可直接 dump
- `metadata_header_shifted`: 有自定义头，去掉后可 dump
- `metadata_offset_encrypted`: 偏移表加密，需运行时 dump
- `metadata_fully_encrypted`: 整体加密，需专用解密

### 3. 脚本/资源加密检测

扫描可能包含脚本的文件：

| 文件类型 | 正常特征 | 加密特征 |
|---------|---------|---------|
| `.lua` / `.luac` | 文件头 `1B4C7561`(Lua 5.1) 或 `1B4C756A`(LuaJIT) | 高熵、无 Lua magic |
| `.js` / `.jsc` | 明文 JS 或 V8 字节码 `C0DE FA CE` | 高熵、无可识别头 |
| `.dll.bytes` | PE magic `4D5A` | 高熵、无 PE magic |
| `.csb.so` | 不应该是 SO | 伪装成 .so 的加密数据 |
| `.ab` / `.unity3d` | Unity AB magic `UnityFS` | 无 UnityFS 头 |

**熵值计算：**
- 对可疑文件取前 4KB 计算 Shannon 熵
- 熵 > 7.5 (满分 8.0): 高概率加密
- 熵 6.0-7.5: 可能是压缩数据
- 熵 < 6.0: 大概率明文或字节码

### 4. Java/Kotlin 混淆检测

分析 jadx 反编译结果或 smali 目录：

| 特征 | 混淆等级 | 说明 |
|------|---------|------|
| 单字母类名 (a.class, b.class) | ProGuard/R8 标准混淆 | 不影响反编译，可读性差 |
| 无意义包名 (a.b.c.d) | 深度混淆 | 需要重命名辅助 |
| 字符串加密 (DES/AES 解密调用) | 字符串保护 | 需要动态解密 |
| 控制流混淆 (大量 switch/goto) | 控制流平坦化 | jadx 可能反编译失败 |
| Native 方法占比 > 80% | JNI 壳 | 核心逻辑在 native 层 |

### 5. 资源格式检测

| 格式 | 正常 magic | 检测方法 |
|------|-----------|---------|
| Unity AssetBundle | `UnityFS` (offset 0) | 检查前 7 字节 |
| Unity ResourcesVolume (.db) | `01000002 01020304` | 腾讯定制格式，无公开工具 |
| Unreal .pak | magic at file end | 检查文件尾部 |
| Oodle 压缩 | 需要专用解压 | 检查是否有 libOodle*.so |

---

## 执行流程

```
1. 扫描产物目录，建立文件清单
2. 按检测矩阵逐项执行
3. 汇总结果，生成 protection_report.txt
```

## 输出格式

```
<产物目录>/protection_report.txt

=== 保护检测报告 ===
应用: <应用名>
检测时间: <时间>
产物目录: <路径>

=== 1. Native 库加壳 ===
正常 ELF (可分析): N 个
  <列表，含文件名和大小>

加壳库 (需脱壳): N 个
  <列表，含文件名、大小、magic、壳类型>

加密数据 (伪装 .so): N 个
  <列表，含文件名、大小、熵值>

=== 2. IL2CPP Metadata ===
状态: <normal/header_shifted/offset_encrypted/fully_encrypted>
详情: <具体分析结果>
建议: <处理方案>

=== 3. 脚本/资源加密 ===
<按文件类型分组列出>

=== 4. Java 混淆 ===
混淆等级: <无/标准/深度>
详情: <具体特征>

=== 5. 资源格式 ===
<按格式分组列出>

=== 总结 ===
可直接分析的层: <列表>
需要额外处理的层: <列表 + 建议>
无法静态处理的层: <列表 + 替代方案>
```

---

## 已知的坑

| 问题 | 应对 |
|------|------|
| 熵值计算对小文件不准 | 文件 < 1KB 时跳过熵值检测，直接看文件头 |
| 有些加壳库仍保留部分字符串 | 即使判定为加壳，仍然提取 strings 作为参考 |
| global-metadata.dat 可能在非标准路径 | 搜索整个 assets 目录，不只看 Managed/Metadata/ |
| UPX 脱壳后文件可能损坏 | 脱壳前先备份原文件 |
