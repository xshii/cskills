# cskills — C/C++ 混合项目分析与搬迁 Skills

三个 Claude Code 自定义命令，专为 **GLM 4.7 CLI**（有限上下文）优化，处理 C/C++/Python/Bash 混合项目的结构分析与代码搬迁。

---

## 推荐工作流

```
# 1. 先生成项目地图（快，约 10 次工具调用）
/cpp-map ./my-project

# 2. 深度分析（读源码，找 Bug，理解运行流程）
/cpp-analyze ./my-project

# 3. 搬迁（利用地图文件，分块处理每个文件）
/cpp-migrate ./my-project --to c --outdir ./my-project-c
```

---

## 三个命令说明

### `/cpp-map [目录]` — 项目地图生成

**最轻量的命令**，只扫描文件树和构建文件，不读源码。
生成 `.cpp-project-map.md`，供另外两个命令复用，避免重复扫描。

```
/cpp-map                        # 扫描当前目录
/cpp-map ./src                  # 扫描指定目录
```

**输出**：`.cpp-project-map.md`，包含文件清单、入口点、跨语言胶水、迁移复杂度预估。

---

### `/cpp-analyze [目录或文件]` — 深度静态分析

分析 C/C++/Python/Bash 混合项目，理解运行机制，输出结构化中文报告。

```
/cpp-analyze                    # 分析当前目录
/cpp-analyze src/main.cpp       # 分析单个文件
/cpp-analyze ./src              # 分析指定目录
```

**分析内容：**
- 语言标准检测（C89～C23 / C++98～C++23）
- **运行链分析**：`build.sh → ./app → libfoo.so → Python绑定` 这类启动链
- **跨语言胶水**：ctypes / pybind11 / extern "C" / Py_Initialize
- 代码问题三级分类（严重 Bug / 警告 / 可现代化）
- 复杂度热点

**输出**：`.cpp-project-map.md`（更新）+ 终端摘要。

---

### `/cpp-migrate <源目录> --to <目标> [选项]` — 代码搬迁

**支持的迁移目标：**

| `--to` 参数 | 说明 |
|-------------|------|
| `c` | **C++ 重构为 C**（核心功能） |
| `c++11` … `c++23` | 升级 C++ 标准 |
| `cmake` | 构建系统迁移到 CMake |
| `rust` | 翻译为 Rust（含 Cargo.toml） |
| `go` | 翻译为 Go（含 go.mod） |
| `linux` / `windows` / `macos` | 跨平台移植 |
| `sanitize` | 安全加固（替换不安全 API） |

**选项：**

| 参数 | 说明 | 默认值 |
|------|------|-------|
| `--outdir <路径>` | 输出到新目录（不修改原文件） | `<源目录>_migrated` |
| `--map <文件>` | 读取已有地图文件，跳过重新扫描 | 自动查找 `.cpp-project-map.md` |

**示例：**
```
# C++ → C，输出到新目录
/cpp-migrate ./src --to c --outdir ./src_c

# 升级到 C++20
/cpp-migrate ./src --to c++20 --outdir ./src_cpp20

# 利用已有地图，直接迁移
/cpp-migrate ./src --to c --map .cpp-project-map.md --outdir ./src_c

# Makefile → CMake
/cpp-migrate . --to cmake --outdir ./cmake_build
```

**C++ → C 覆盖的转换规则：**
- 类 → 结构体 + 函数（`Foo_method(Foo* self, ...)`）
- 继承 → 首字段嵌入 + 函数指针虚表
- 模板 → 宏或具体类型函数
- RAII/智能指针 → 手动 `init`/`destroy`
- 异常 → 错误码返回值
- STL 容器 → C 数组/uthash
- 命名空间 → 前缀命名约定

**输出**：`<outdir>/` 目录（镜像源目录结构）+ `MIGRATION_REPORT.md`

---

## 安装

```bash
# 项目级（仅当前项目可用）
cp -r .claude /path/to/your/project/

# 全局（所有项目可用）
cp -r .claude/commands/* ~/.claude/commands/
```

---

## 针对 GLM 4.7 CLI 的设计

| 问题 | 解决方案 |
|------|---------|
| 上下文有限 | 每次只读一个文件，处理完写入磁盘后释放 |
| 容易截断输出 | 明确禁止省略代码，超长文件分批次读写 |
| 开放式任务容易跑偏 | 所有步骤编号、有序，每步有明确完成条件 |
| 中文优先 | 全程中文指令，角色设定在 prompt 首行 |
| 中间结果易丢失 | 分析结果写文件（`.cpp-project-map.md`），跨会话复用 |

---

## 目录结构

```
cskills/
├── README.md
└── .claude/
    └── commands/
        ├── cpp-map.md        # /cpp-map     快速地图生成
        ├── cpp-analyze.md    # /cpp-analyze 深度静态分析
        └── cpp-migrate.md    # /cpp-migrate 代码搬迁
```
