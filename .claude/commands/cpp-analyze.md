你是一名资深系统工程师，擅长分析 C/C++/Python/Bash 混合项目的代码结构与运行机制。

分析目标：`$ARGUMENTS`（未指定则分析当前目录）

> **上下文控制原则**：每次只读一个文件，分析完立即释放，不要把多个文件内容同时放在上下文里。先用工具列出文件列表，再逐一处理。

---

## 执行步骤

### 第一步：建立文件清单（用工具，不要靠猜）

依次执行：
1. 列出根目录所有文件（不递归）
2. 列出构建文件：`CMakeLists.txt`、`Makefile`、`*.mk`、`meson.build`、`xmake.lua`、`setup.py`、`pyproject.toml`
3. 列出脚本入口：`*.sh`、`*.bash`、`*.py`（只要根目录和 `scripts/` `tools/` `bin/` 下的）
4. 列出 C/C++ 源文件树（只统计数量和路径，**不读内容**）

把文件清单记录下来，后续按清单逐一处理。

---

### 第二步：识别项目启动入口

**按优先级逐一检查，找到后即可停止：**

1. **构建系统**（先读 `CMakeLists.txt` 或 `Makefile`）
   - 找 `add_executable` / 顶层 `all:` target，确定可执行文件名
   - 记录编译命令、链接库、编译标志

2. **Shell 脚本入口**（读 `run.sh` / `start.sh` / `launch.sh` / `entrypoint.sh`）
   - 找第一个非注释的可执行命令
   - 记录它调用了哪些二进制或 Python 脚本

3. **Python 入口**（读 `main.py` / `__main__.py` / `app.py`）
   - 找 `if __name__ == "__main__"` 块
   - 找对 C 扩展的调用：`ctypes.CDLL`、`cffi`、`import <*.so>`、`pybind11`

4. **C/C++ main 函数**（用 grep 搜索 `int main(` ，只读命中的文件）

---

### 第三步：跨语言胶水层分析

**只分析以下几类文件，其他跳过：**

| 胶水类型 | 查找方式 |
|----------|---------|
| Python 调用 C | grep `ctypes` `cffi` `cdll` `CDLL` `pybind11` `cython` |
| C 调用 Python | grep `Py_Initialize` `PyRun_` `PyObject` |
| 脚本调用二进制 | grep `subprocess` `os.system` `exec ` `./` 在 .py/.sh 中 |
| 共享库导出 | grep `__attribute__((visibility` `DLL_EXPORT` `extern "C"` |

对每处命中：读对应文件的相关片段（用行号范围读，不要整文件），记录调用关系。

---

### 第四步：C/C++ 代码问题扫描（逐文件）

**每次只处理一个文件**，处理完再处理下一个。对每个 `.c`/`.cpp`/`.h`/`.hpp` 文件：

1. 读文件（超过 300 行时只读前 300 行，记录"未读完"）
2. 扫描以下问题，记录 `文件:行号 — 描述`：

   **内存/安全**：`strcpy` `sprintf` `gets` `malloc` 后未判空、`free` 后未置 NULL、VLA
   **C++ 特有**：`auto_ptr`、C 风格转换、裸 `new`/`delete`（无智能指针）、`throw()`
   **资源泄漏**：`fopen` 无 `fclose`、socket/fd 未释放
   **并发**：裸全局可变状态、`volatile` 当原子用

3. 统计该文件行数、函数数量（grep `^[a-zA-Z].*([^;]*$` 估算）

---

### 第五步：生成项目地图文件

将分析结果**写入文件** `.cpp-project-map.md`（用文件写工具），内容格式如下：

```markdown
# 项目地图

生成时间：<日期>
分析目录：<路径>

## 启动链
<用箭头描述启动流程，例如>
run.sh → ./build/myapp → Python main.py → libfoo.so → C库

## 文件清单
| 文件 | 类型 | 行数 | 说明 |
|------|------|------|------|
| src/main.cpp | C++ | 312 | 程序入口，初始化网络 |
| ... | | | |

## 跨语言调用关系
- main.py:45 ctypes.CDLL("libfoo.so") → src/foo.c:export_func()
- ...

## 构建信息
- 构建系统：<CMake 3.x / Makefile>
- 编译标准：<C11 / C++17>
- 主要依赖：<库列表>
- 产出文件：<可执行文件/库列表>

## 问题汇总
### 严重
- `文件:行号` — 描述

### 警告
- `文件:行号` — 描述

### 可优化
- `文件:行号` — 描述

## 迁移建议
<如果要做 C++ → C 迁移，这里列出哪些文件改动量大、哪些简单>
```

---

### 第六步：向用户输出摘要

文件写完后，**只输出以下精简摘要**（不要重复文件内容）：

```
## 分析完成

项目地图已写入：.cpp-project-map.md

### 启动链
<一行描述>

### 文件统计
- C/C++ 源文件：X 个，共约 XXXX 行
- Python 脚本：X 个
- Shell 脚本：X 个
- 构建系统：<名称>

### 发现问题
- 严重：X 处
- 警告：X 处

### 跨语言胶水
<列出主要调用点>

如需搬迁，运行：/cpp-migrate --map .cpp-project-map.md --to <目标>
```

输出语言：**中文**。摘要控制在 50 行以内。
