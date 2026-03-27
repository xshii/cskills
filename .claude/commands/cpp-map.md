你是一名代码结构分析专家。你的唯一任务是快速扫描项目结构并生成一份机器可读的项目地图文件，供后续分析和迁移命令使用。

扫描目标：`$ARGUMENTS`（未指定则扫描当前目录）

> **上下文控制**：只读文件列表和构建文件，**不读源文件内容**，速度优先。

---

## 执行步骤

### 第一步：文件树扫描（只列路径，不读内容）

用工具列出以下文件，按类型分组记录：

1. **构建文件**：`CMakeLists.txt` `Makefile` `*.mk` `meson.build` `xmake.lua` `setup.py` `pyproject.toml` `*.bazel`
2. **C/C++ 源文件**：`*.c` `*.cpp` `*.cc` `*.cxx`
3. **C/C++ 头文件**：`*.h` `*.hpp` `*.hxx`
4. **Python 文件**：`*.py`
5. **Shell 脚本**：`*.sh` `*.bash`
6. **文档/配置**：`README*` `*.md` `*.json` `*.yaml` `*.yml` `*.toml` `*.ini` `*.conf`

统计每类文件数量。

### 第二步：读取构建文件（每个最多读 200 行）

对每个找到的构建文件：
- **CMakeLists.txt**：提取 `project()` `add_executable()` `add_library()` `target_link_libraries()` `set(CMAKE_C*_STANDARD` 行
- **Makefile**：提取前 50 行（含变量定义和顶层 target）
- **setup.py / pyproject.toml**：提取包名、ext_modules（C 扩展）部分

### 第三步：grep 关键信息（不读完整文件）

用 grep 工具在源目录执行以下搜索（只要文件名和行号）：

1. `int main(` → 找 C/C++ 程序入口
2. `if __name__ == .__main__.` → 找 Python 入口
3. `ctypes\|cffi\|pybind11\|cython` → 找 Python→C 胶水
4. `extern "C"` → 找 C++ 导出符号
5. `Py_Initialize\|PyRun_\|PyObject` → 找 C→Python 嵌入
6. `#include <` → 统计最常用的系统头文件（top 10）
7. `class ` → 统计 C++ 类定义数量
8. `template<\|template <` → 统计模板数量

### 第四步：写入地图文件

将结果写入 `.cpp-project-map.md`（覆盖已有内容）：

```markdown
# 项目地图
<!-- 由 /cpp-map 自动生成，可被 /cpp-analyze 和 /cpp-migrate 读取 -->

生成时间：<ISO 日期>
扫描目录：<绝对路径>

## 文件统计
| 类型 | 数量 | 示例路径 |
|------|------|---------|
| C 源文件（.c） | N | src/main.c |
| C++ 源文件（.cpp/.cc/.cxx） | N | src/engine.cpp |
| C/C++ 头文件 | N | include/foo.h |
| Python 脚本 | N | scripts/run.py |
| Shell 脚本 | N | run.sh |
| 构建文件 | N | CMakeLists.txt, Makefile |

## 构建系统
- 类型：<CMake / Makefile / Meson / 未知>
- 语言标准：<从构建文件提取，或"未指定">
- 可执行目标：<列表>
- 库目标：<列表>
- 链接依赖：<列表>

## 程序入口点
| 文件 | 行号 | 类型 | 说明 |
|------|------|------|------|
| src/main.cpp | 12 | C++ main | - |
| scripts/run.py | 1 | Python __main__ | - |
| start.sh | 1 | Shell 入口 | 调用 ./build/app |

## 跨语言胶水
| 文件 | 行号 | 类型 |
|------|------|------|
| bindings/foo.py | 8 | ctypes |
| src/embed.cpp | 45 | Py_Initialize |

## C++ 特性使用统计
- 类定义数：N（grep 结果）
- 模板数：N
- extern "C" 导出：N 处

## 常用系统头文件（Top 10）
1. <stdio.h> — 引用于 X 个文件
2. ...

## 全部文件列表
### C/C++ 源文件
- src/main.cpp
- src/engine.cpp
- ...

### C/C++ 头文件
- include/engine.h
- ...

### Python 文件
- scripts/run.py
- ...

### Shell 脚本
- run.sh
- ...

## 迁移复杂度预估
<!-- 供 /cpp-migrate --to c 参考 -->
| 文件 | 类数 | 模板数 | 预估难度 |
|------|------|--------|---------|
| src/engine.cpp | 3 | 2 | 🔴 复杂 |
| src/util.cpp | 0 | 0 | 🟢 简单 |
```

### 第五步：向用户输出简短确认

```
## 项目地图已生成

文件：.cpp-project-map.md

快速摘要：
- C/C++ 源文件：N 个
- Python 脚本：N 个
- Shell 脚本：N 个
- 入口点：<列出文件名>
- 跨语言胶水：<有/无，简述>

下一步：
  /cpp-analyze          # 深度分析（读源码、找 Bug）
  /cpp-migrate --to c --map .cpp-project-map.md  # 直接开始迁移
```

输出语言：**中文**。整个命令执行应在 10 轮工具调用内完成。
