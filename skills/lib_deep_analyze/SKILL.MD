---
name: lib-deep-analyze
description: 单库深度分析 — 对单个 native 库（ELF/Mach-O）进行导出函数分类、C++类还原、模块架构图生成
---

# Lib Deep Analyze Skill

## 功能说明

对单个 native 库文件进行深度逆向分析，从导出函数、字符串、源码路径泄露中还原模块架构。

## 使用方法

```
/lib-deep-analyze <库文件路径> [--output <输出目录>]
```

- `--output`：可选，默认在产物目录下创建 `09_lib<name>/`

## 部署方式：主会话直接执行

需要执行 nm、strings、c++filt 等命令，分析过程是迭代式的。

## 前置条件

- 库文件必须是正常的 ELF 或 Mach-O（非加壳）
- 如果是加壳文件（如 `_cp.so`），先提示用户需要脱壳

---

## 执行流程

### Step 1: 基础提取

```bash
# 文件信息
file <库文件>
ls -lh <库文件>

# 导出函数（带 C++ demangle）
nm -D <库文件> | grep ' T ' | awk '{print $3}' | c++filt > exports.txt

# 导入函数
nm -D <库文件> | grep ' U ' | awk '{print $2}' | c++filt > imports.txt

# 完整符号表
nm -D <库文件> > symbols_all.txt

# 原始字符串
strings -n 6 <库文件> > strings_raw.txt
```

### Step 2: 函数前缀聚类 → 模块识别

从 exports.txt 中提取函数名前缀，按前缀分组：

```
分组规则:
- C 风格: 按 `_` 分割取前 1-2 段 (如 GZEUS_*, Apollo_*, GCloud_*)
- C++ 风格: 按 `::` 分割取类名 (如 CPufferMgr::*, CEifsWrapper::*)
- Java JNI: 按 `Java_com_` 后的包名分组
- 其他: 按首字母大写单词边界分组
```

输出 `module_analysis.txt`，按模块分类列出所有导出函数。

### Step 3: C++ 类名还原

从 strings_raw.txt 中提取 `Class::Method` 模式：

```bash
grep -oE '[A-Z][a-zA-Z0-9_]+::[A-Za-z_][A-Za-z0-9_]+' strings_raw.txt | sort | uniq -c | sort -rn
```

按出现频率排序，高频类是核心类。

### Step 4: 源码路径泄露

从 strings_raw.txt 中提取编译时泄露的源码路径：

```bash
grep -E '\.(cpp|h|cc|hpp|c):' strings_raw.txt | sort -u
```

这些路径直接暴露了源码目录结构。

### Step 5: 辅助信息提取

```
- URL/域名: grep -iE '(https?://|\.com|\.qq\.|\.tencent\.)' strings_raw.txt
- 协议关键字: grep -iE '(protobuf|json|http|tcp|udp|ssl|websocket|encrypt|decrypt|md5|sha|aes)' strings_raw.txt
- 错误信息: grep -iE '(error|fail|warn|exception|invalid|timeout)' strings_raw.txt
```

### Step 6: 生成综合报告

输出 `report.txt`，包含：

```
=== 库文件信息 ===
文件名、大小、类型、架构

=== 模块架构总览 ===
ASCII 架构图（表格或树形）

=== 各模块详细分析 ===
每个模块:
  - 核心类
  - 功能描述
  - 导出函数列表
  - 工作流程（如果能推断）

=== 网络/服务端点 ===
URL、域名列表

=== 安全相关 ===
加密、认证、签名相关发现

=== 源码结构（泄露） ===
从路径泄露还原的源码目录树
```

---

## 输出目录结构

```
09_lib<name>/
├── exports.txt          # 导出函数列表（demangled）
├── imports.txt          # 导入函数列表（demangled）
├── symbols_all.txt      # 完整符号表
├── strings_raw.txt      # 原始字符串
├── module_analysis.txt  # 按模块分类的导出函数
├── deep_analysis.txt    # C++类/URL/协议/错误信息
└── report.txt           # 综合分析报告
```

## 已知的坑

| 问题 | 应对 |
|------|------|
| stripped 库没有导出符号 | 降级为纯字符串分析 |
| C++ 模板导致 demangle 后函数名极长 | 截断显示，保留核心类名和方法名 |
| 静态链接的库符号混在一起 | 通过源码路径泄露区分不同模块 |
| Mach-O 用 nm 而非 nm -D | 检测文件类型，Mach-O 用 `nm` 不加 `-D` |
| 导出函数数量 > 10000 | 只对前缀聚类结果做详细分析，不逐个列出 |
