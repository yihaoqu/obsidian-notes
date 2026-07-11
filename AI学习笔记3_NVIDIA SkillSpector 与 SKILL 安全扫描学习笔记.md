---
title: 对于 NVIDIA SkillSpector 过滤 SKILL 安全的一次学习
date: 2026-07-11
tags:
  - github工作流
  - 网络安全
  - python
  - windows开发环境
  - 开源实践
status: 进行中
related:
  - "[[Computer Security 期末复习]]"
  - "[[Git 与 GitHub 基础]]"
  - "[[Python 虚拟环境笔记]]"
---

# 对于 NVIDIA SkillSpector 过滤 SKILL 安全的一次学习

> [!note] 本次学习目标
> 理解一个真实的 AI 安全工具是怎么设计的。
>

---

## 一、项目是什么

**[[NVIDIA SkillSpector]]** 是一个给 AI Agent 的"技能包"(skill)做安全扫描的工具。

背景:Claude Code、Pi 这类 AI agent 都支持装第三方 skill(指令包+工具集),但这些 skill 的代码大概率没人逐行审查过,可能藏有恶意逻辑(偷传数据、执行危险命令、prompt 注入等)。SkillSpector 的作用就是:**把一个 skill 丢给它,输出一份风险报告,告诉你能不能装**。

### 核心工作流程(基于 LangGraph 搭的流水线)

```
resolve_input → build_context → 22个analyzer并行扫描 → meta_analyzer(可选LLM过滤) → report
```

### 两阶段检测

- **Stage 1 静态分析**:正则匹配 + Python AST 分析 + YARA 签名匹配。速度快、不用联网、不花钱,但可能有误报。对应命令行的 `--no-llm` 参数。
- **Stage 2 LLM 语义分析(可选)**:调用配置好的 AI 模型(OpenAI/Anthropic/NVIDIA 等)对静态分析结果做语义过滤,减少误报,精度能到 ~87%。需要配置 `SKILLSPECTOR_PROVIDER` 环境变量 + 对应 API key。

### 检测的问题类别(与 [[网络安全]] 知识点串联)

一共 **68 种漏洞模式,分 17 大类**,和之前 Computer Security 课程里学的概念有不少呼应,值得对照复习:

| SkillSpector 检测类别 | 与课程知识点的关联 |
|---|---|
| Prompt Injection / Anti-Refusal | AI 特有的"注入攻击",概念上类似传统的 SQL 注入/命令注入,但攻击面是自然语言指令 |
| Data Exfiltration(数据外泄) | 呼应课程里讲的机密性(Confidentiality)被破坏的场景 |
| Privilege Escalation(权限提升) | 直接对应课程里的 AAA 框架(Authentication/Authorization/Accounting)中 Authorization 被绕过的情况 |
| Supply Chain(供应链攻击) | 比如"依赖包 typosquatting"——恶意包伪装成常用包名,是现实世界真实发生过的攻击手法 |
| YARA Signatures | 传统恶意软件检测手段(webshell、cryptominer 特征匹配),和课程里的 malware 章节直接相关 |
| Behavioral AST / Taint Tracking | 静态代码分析技术:AST 检测危险函数调用(`exec`/`eval`/`subprocess`),Taint Tracking 追踪"数据从哪来、流到哪去",判断敏感数据是否未经清洗就流向了危险位置(比如网络请求) |

### 风险评分机制

```
CRITICAL +50 / HIGH +25 / MEDIUM +10 / LOW +5 分,可执行脚本 ×1.3 系数
0-20分 LOW/SAFE → 21-50分 MEDIUM/CAUTION → 51-100分 HIGH~CRITICAL/DO_NOT_INSTALL
```

### 三种运行方式

1. **CLI**(已实践):`skillspector scan <path> --no-llm`
2. **LangGraph Studio**:`make langgraph-dev`,可视化调试用,能看到流程图逐步执行
3. **Python API**:`from skillspector import graph; graph.invoke(...)`,给想集成进自己项目的开发者用
4. (额外)**MCP Server**:`skillspector mcp`,把它注册成 Claude Code / Pi 等 agent 可直接调用的工具

---

## 二、Windows 开发环境搭建(踩坑记录)

### 基础工具链

| 工具 | 安装方式 | 备注 |
|---|---|---|
| git | Git for Windows | 自带 Git Bash |
| Python | 官网安装包 / `winget install Python.Python.3.13` | 装出来是 3.13.0 |
| uv | `pip install uv` 或官方脚本 | Astral 团队做的新一代 Python 工具链,一个工具顶 pip+venv+pyenv+pipx |

### 踩过的坑

> [!warning] `python` 命令没反应
> Windows 的 "App execution alias" 占用了 `python` 这个名字但没真正指向已安装的 Python,导致敲了没反应也不报错。**解决:用 `py` 代替 `python`**,或者去 设置→应用→高级应用设置→应用执行别名 里关掉。

> [!warning] `source .venv/bin/activate` 在 Windows 上跑不了
> `source` 是 Mac/Linux shell 特有命令。Windows 上直接调用脚本本身就会让环境变量生效,不需要 `source` 这层包装。**Windows 正确写法:`.venv\Scripts\activate`**(CMD/PowerShell 通用,系统会自动匹配到 `activate.bat`)。

> [!warning] `make` 系列命令在 Windows 上不可靠
> - Windows 原生不带 `make`,需要额外装(`winget install GnuWin32.Make`),且要重开终端才能生效。
> - 即使装好了 make 本身,Makefile 内部很多逻辑用的是 **POSIX shell 语法**(`if [ -n ... ]; then ... fi`),Windows CMD/PowerShell 根本不认识这种语法,大概率还是会报错。
> - **最终结论:直接打开 Makefile 文件看内容,把每条 `make xxx` 翻译成等价的 `uv`/`pytest`/`ruff` 命令手动执行,比死磕 make 兼容性更划算。**

### Make 命令等价对照表(Windows 实测可用)

| 文档命令 | Windows 等价命令 |
|---|---|
| `make install-dev` | `uv sync --all-extras` |
| `make test` | `uv run pytest` |
| `make lint` | `uv run ruff check src/ tests/` |
| `make format` | `uv run ruff check --fix src/ tests/` 然后 `uv run ruff format src/ tests/` |

---

## 三、GitHub 概念梳理

### Fork / Clone / Origin / Upstream 的关系

```
NVIDIA/SkillSpector(原仓库,只能读)
        │ fork(云端复制一份到自己账号)
        ▼
我的 fork(云端,有完整读写权限)
        │ clone(下载到本地)
        ▼
本地 Windows 文件夹(实际编辑的地方)
```

- **origin**:本地仓库里指向"我的 fork"的别名,是 `git push` 的目的地
- **upstream**:本地仓库里指向"NVIDIA 原仓库"的别名,用来拉取原项目的最新更新(`git fetch upstream && git merge upstream/main`)
- **fork 的作用**:①push 的落地点 ②开 PR 的起点 ③云端永久备份 ④自己的练习场,改坏了不影响原项目

### HTTPS / SSH / GitHub CLI 的区别(clone 的三种方式)

- **HTTPS**:走网址协议,push 时用 PAT(Personal Access Token)当密码验证,新手标准选择
- **SSH**:用一对密钥做身份验证,配好后不用每次输密码,但要多一步生成密钥+上传公钥
- **GitHub CLI(`gh` 命令)**:GitHub 官方出的独立命令行工具,专门包装"开 PR、看 issue"这类 GitHub 平台特有功能,和 Git Bash(git 本身的终端)是两码事

### Codespaces 是什么

GitHub 提供的云端开发环境,打开浏览器就有现成的开发环境,不用在本地装任何东西。**这次练习特意选择跳过它**——因为"在本地电脑从零搭建开发环境"这件事本身就是需要练习的核心技能,用 Codespaces 相当于跳过了这一步。




---

## 四、两种安装方式对比

| | 开发模式(`uv sync --all-extras` 在项目文件夹内) | 全局工具(`uv tool install git+...`) |
|---|---|---|
| 代码来源 | 本地 clone 下来的文件夹,可编辑 | 重新从 GitHub 下载,固定版本 |
| 改代码能否立刻生效 | 能 | 不能,需要 `uv tool update` |
| 使用前置条件 | 需要 cd 进项目 + 激活 `.venv` | 任何终端、任何目录直接用 |
| 适合场景 | 学习代码结构、练习开发流程 | 日常拿来扫描别的 skill |

两者互不干扰,可以同时保留。

---

## 五、今天完整走过的操作序列(备查)

```bash
# 1. Fork(网页操作,略)

# 2. Clone 自己的 fork
git clone https://github.com/yihaoqu/SkillSpector.git
cd SkillSpector
git remote -v

# 3. 关联 upstream(方便以后同步 NVIDIA 更新)
git remote add upstream https://github.com/NVIDIA/SkillSpector.git

# 4. 创建并激活虚拟环境
uv venv .venv
.venv\Scripts\activate

# 5. 安装开发依赖(等价于 make install-dev)
uv sync --all-extras

# 6. 验证 + 跑通一次扫描
skillspector --version
skillspector scan tests/fixtures/safe_skill/SKILL.md --no-llm

# 7. 跑通全部测试(1257 passed, 12 skipped, 6 xfailed)
uv run pytest

# 8. 额外装一份全局命令行工具版(独立于开发环境)
uv tool install git+https://github.com/NVIDIA/skillspector.git
```

日常扫描一个新 skill,只需要(不需要重新安装):
```bash
skillspector scan C:\path\to\某个skill\ --no-llm
```

---

## 六、待办 / 下一步

- [ ] 打开 Cursor,对照 `src/skillspector/graph.py` 和 `state.py`,把架构文档和真实代码对应起来看
- [ ] 找一个真实的小改动(文档 typo、过时链接),走一遍真正的 PR 流程(`git checkout -b docs/xxx` → commit → push → 对 `NVIDIA/SkillSpector` 开 PR)
---

## 七、一句话总结

"fork → clone → 建环境 → 装依赖 → 跑通测试 → 理解架构"这一整套开源协作的**准备阶段**,也顺带把 Windows 下 Python/Git 开发环境的几个常见坑(PATH、activate 语法、make 兼容性)踩了一遍。下次面对任何一个新的 GitHub 项目,这套流程可以直接复用。
