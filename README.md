# cskills — C/C++ 分析与搬迁 Skills

本仓库提供两个 Claude Code 自定义命令（skills），专为 **GLM 4.7** 模型优化，面向 C/C++ 项目的静态分析与代码搬迁场景。

## 安装

将 `.claude/` 目录复制到你的项目根目录，或放到 `~/.claude/` 下作为全局命令：

```bash
# 项目级（仅当前项目可用）
cp -r .claude /path/to/your/project/

# 全局（所有项目可用）
cp -r .claude/commands/* ~/.claude/commands/
```

---

## 命令说明

### `/cpp-analyze` — 代码静态分析

对 C/C++ 项目或文件进行全面的静态分析，输出结构化中文报告。

**用法：**

```
/cpp-analyze                        # 分析当前目录整个项目
/cpp-analyze src/main.cpp           # 分析单个文件
/cpp-analyze src/                   # 分析指定目录
```

**报告包含：**
- 语言标准检测（C89～C23 / C++98～C++23）
- 构建系统识别（CMake / Makefile / Meson 等）
- 头文件与第三方库依赖图
- 问题分级（严重 Bug / 警告 / 可现代化建议）
- 内存安全、未定义行为、资源泄漏、并发隐患扫描
- 复杂度热点（超大文件 / 超长函数）

---

### `/cpp-migrate` — 代码搬迁

将 C/C++ 代码迁移到目标标准、构建系统、编程语言或目标平台，输出完整的迁移后代码。

**用法：**

```
/cpp-migrate src/                          # 交互式选择迁移目标
/cpp-migrate src/ --to c++17               # 升级到 C++17
/cpp-migrate src/ --to c++20               # 升级到 C++20
/cpp-migrate Makefile --to cmake           # Makefile → CMakeLists.txt
/cpp-migrate src/net.c --to rust           # 翻译为 Rust
/cpp-migrate src/util.cpp --to go          # 翻译为 Go
/cpp-migrate src/ --to linux               # 移植到 Linux 平台
/cpp-migrate src/ --to windows             # 移植到 Windows 平台
/cpp-migrate src/ --to sanitize            # 安全加固（替换不安全 API）
```

**支持的迁移目标：**

| `--to` 参数 | 说明 |
|-------------|------|
| `c++11` / `c++14` / `c++17` / `c++20` / `c++23` | 升级 C++ 标准 |
| `cmake` | 构建系统迁移到 CMake |
| `rust` | 翻译为 Rust（含 `Cargo.toml`） |
| `go` | 翻译为 Go（含 `go.mod`） |
| `windows` / `linux` / `macos` | 跨平台移植 |
| `sanitize` | 安全加固 |

---

## 设计说明

这两个 skill 的 prompt 针对 **GLM 4.7** 特点做了如下优化：

1. **明确角色设定**：在 prompt 开头用"你是一名资深..."定义角色，GLM 4.7 对角色扮演指令响应稳定
2. **步骤编号清晰**：将分析/迁移流程拆解为有序步骤，GLM 4.7 更擅长按步骤执行而非开放式推理
3. **显式输出格式**：用代码块给出报告模板，避免 GLM 4.7 自行决定格式导致输出不一致
4. **禁止省略**：明确声明"不得省略为// 其余代码不变"，防止截断输出
5. **中文优先**：全程中文指令，减少语言切换开销

---

## 目录结构

```
cskills/
├── README.md
└── .claude/
    └── commands/
        ├── cpp-analyze.md    # /cpp-analyze 命令定义
        └── cpp-migrate.md    # /cpp-migrate 命令定义
```
