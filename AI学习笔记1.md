# AI 学习笔记:从 Agent 到 Plugin 的完整认知

> 一天的学习总结 · 从"agent 是什么"到"自己解剖 plugin、看懂 MCP registry"

---

## 一、核心概念地图(打地基)

这一天所有内容,都挂在下面这张分层图上。从底到上:

```
模型(大脑)         GPT / Claude          —— 会思考,但被关在房间里
   ↓ 套上聊天外壳
聊天产品           ChatGPT / Claude.ai   —— 你问它答
   ↓ 套上工具 + 循环外壳
编码 agent         Codex / Claude Code   —— 能动手改你电脑上的代码
```

### 关键定义

- **LLM(大模型)** = 很聪明但被关在房间里的人,只能通过文字交流,没有手脚。
- **Agent(智能体)** = LLM + 工具 + 循环。给它配上手脚(工具),让它自己决定下一步、看结果、再决定,直到任务完成。
- **Chatbot(聊天机器人)** = 一问一答,不会自己动手做事。
- **ChatGPT 严格说不是 agent**,是"主要以聊天为形态、部分场景带 agent 能力"的产品。
- **Claude Code 和 Codex 是同一类东西**——都是"编码 agent",区别只是背后的大脑(Claude vs GPT)和厂商(Anthropic vs OpenAI)。

### Agent 的核心逻辑(务必记住)

```
1. 看当前状态(指令 + 目前进展)
2. 模型预测:下一步该做什么?→ 输出一个动作(调用工具)
3. 工具执行,产生真实结果
4. 把结果喂回模型,回到第 1 步
5. 重复,直到任务完成
```

**核心洞察:模型本身只会"预测下一步",是这个循环把一连串"下一步"串成了"完成一整件事"。**

---

## 二、LLM 到底怎么工作(祛魅)

### 处理流程

```
1. prompt 被切成一串 token(比"词"更碎的片段)
2. 模型把整串 token 一起看,预测"下一个最该出现的 token"
   (在高概率候选里带点随机地选 —— 这就是 temperature)
3. 把选中的 token 接到末尾
4. 回到第 2 步:把"原输入 + 已生成的全部"再整体看一遍,预测下一个
5. 一个接一个滚下去,直到结束
```

### 三个关键纠正

1. **不是"对每个 token 分别预测",而是"综合全部已有内容,一次预测一个新 token"。**
2. **不是一口气生成整段,而是一个 token 一个 token 滚出来的**,每滚一个都重看前面全部 → 这是连贯性的来源。
3. **本质和输入法联想相似(都预测下一个),差别在规模和深度**——预测时卷入了语法、事实、逻辑、语境。

### ⚠️ 最重要的警惕

**"快"和"并行"不等于"对"。** 同样的"预测最合理下一步"机制,既能流畅给出正确解释,也能一样自信地编出错误解释。模型出错时往往毫无迟疑、语气一致。

> **护栏 = 自己验证。**(今天 pyright 那次种错验证,就是最好的示范。)

---

## 三、代码纠错:LSP 与 pyright

### LSP 是什么

**LSP(Language Server Protocol,语言服务器协议)= 编辑器和"语言智能"之间的通用插头。**

- 没有 LSP:M 个编辑器 × N 种语言 = M×N 套实现(灾难)
- 有了 LSP:M 个编辑器 + N 种语言 = M+N 套(一个 pyright 插到所有编辑器)

角色拆分:
- **language server(语言服务器)**:只负责"理解代码"(如 pyright = Python 的语言服务器)
- **client(客户端)**:编辑器 / agent(Cursor、VS Code、Claude Code)
- **LSP**:中间的标准对话协议(本质是 JSON 消息)

> 反直觉但关键:**检查 Python 的 pyright,自己是 TypeScript 写的。** 因为 server 是独立程序,只靠协议通信,用什么语言写无所谓。

### 错误分层(LSP 抓什么、不抓什么)

| 错误类型 | 例子 | pyright 能抓吗 | "跑一下"能发现吗 |
|---|---|---|---|
| 语法错误 | 少冒号、括号没闭合 | ✅ 能 | 能 |
| 类型错误 | str 赋给 int | ✅ 能(主场) | 部分 |
| 部分运行时错误 | 调用不存在的方法 | ✅ 很多能 | 只有走到那行才行 |
| **纯逻辑错误** | 价格比较写反了 | ❌ **抓不到** | 发现不了 |

**关键边界:类型对 ≠ 逻辑对。** LSP 是"类型层面的 observer",守的是数据类型合不合法,不是业务逻辑想得对不对。把它理解成**校对员(抓错别字/格式),不是审稿人(判断论点)**。

### 今日实证亮点:pipeline.py 的第 4 步

用满类型系统复杂特性(TypeVar、Protocol、overload)写了个 pipeline,验证 LSP:

1. `mcp__ide__getDiagnostics` 返回空 → **用错工具**(那个桥接只报编辑器里打开的文件)
2. 换 Pyright CLI(`npx pyright`)→ 对了
3. **故意种 3 个错** → 全被抓到 → 证明工具是活的
4. **发现 1 个没种的真问题**:un-annotated lambda `lambda r: r.value` —— 代码能跑、测试全过,但赋值点没期望类型,pyright 把 `r` 放宽到 `object`,`.value` 非法 → **静态不健全,即使运行时正确**
5. 正确修复:给 stage 变量加注解,让双向推断流进 lambda(而非 `# type: ignore` 捂嘴)

> **最大收获:类型检查器不只抓你种的错,还抓到了"运行 + 测试永远发现不了"的真实推断缺口。** 这是它存在的全部理由。

---

## 四、Plugin 系统(今天的研究主线)

### Plugin 的本质

**一个 plugin = 往 agent 身上挂东西的容器。** 它可能挂 5 类东西:

```
plugin 能给 agent 加的东西:
├── skills      → 操作手册(SKILL.md)
├── agents      → 专门化子代理
├── hooks       → 在某时机自动触发的钩子
├── MCP server  → 连外部服务的插头
└── LSP server  → 连语言智能的插头
```

### 今日实际装的两个 plugin

| Plugin | 净贡献 | 作用 |
|---|---|---|
| **pyright** | 1 个 LSP server | 给 agent 类型检查的"眼睛" |
| **github** | 1 个 MCP server | 给 agent 操作 GitHub 的"手" |

> `/reload-plugins` 报告的 `6 agents` 是 **Claude Code 自带的标准阵容**(claude、Explore、Plan 等),**不是 github plugin 带来的**。github plugin 的净贡献只有那 1 个 MCP server。

### 安装与激活流程

```
1. /plugin 搜索并安装        → ✓ Installed
2. /reload-plugins           → 激活(装好不等于生效,要重载)
3. 激活后 agent 才能调用它带的工具
```

---

## 五、MCP:工具的"USB 标准"

- **MCP(Model Context Protocol)= 工具的标准接口**,相当于 USB。任何服务实现了 MCP,任何支持 MCP 的 agent 都能直接用。
- 结构是 **client-server**:

```
MCP server(工具本体)  ←— MCP 协议 —→  MCP client(用工具的 agent)
   GitHub、Playwright              Claude Code、Copilot、VS Code
```

### ⚠️ 关键:MCP 连接不通用

**在 GitHub 的 MCP registry 页面点 `Install`,装的是给 VS Code / Copilot 用的,你的 Claude Code 用不上。**

- MCP 是 client-server 结构,**每个 client 要各自连接 server**(像每台电脑都得单独添加同一台打印机)
- GitHub registry 的最佳用法:**当"MCP 图鉴"逛**,发现有哪些工具、看描述和 star 数,然后**回 Claude Code 里用 `/plugin` 接**
- `Install` 往往不是下载软件,而是**往某个 client 的配置里写"怎么连接"的配置**

---

## 六、GitHub 认知

### 权限与安全(贯穿今天的主线)

- **副作用 = 改变外部真实状态、且往往不可逆的操作。** 读=无副作用,写/改/删=有副作用。
- **agent 的权限 = 你给它的 token 的权限。** GitHub MCP 靠 `GITHUB_PERSONAL_ACCESS_TOKEN`(PAT)认证。
- **PAT 是密码级机密**:GitHub 只在生成那一刻显示一次明文,之后只存哈希(连 GitHub 自己都看不到原文)。绝不能 commit / 泄露,应放环境变量。
- **研究阶段应给最小只读权限**——从源头限制 agent 能做什么,比事后靠权限弹窗拦更根本。
- 副作用主要落在**你自己的仓库**;对别人仓库改不了内容,但能**以你名义公开发 issue/PR/评论**。

### 协作方式

| 方式 | 谁能改 | 适合 |
|---|---|---|
| **collaborator** | 直接给写权限,能直接改你仓库 | 信任的小团队 |
| **Fork + PR** | 别人只能"提建议",改不改你定 | 陌生人、开源、新手起步 |

### 其他要点

- **contribution(贡献记录)** = GitHub 上"干了多少活"的可视化。绿墙受隐私设置影响大,别用它判断人。别为刷绿墙 push 无意义练习。
- **和用户名同名的仓库是"个人主页特殊仓库"**(README 显示在主页顶部),**不要删**。
- **删仓库不可逆**:Settings → 底部 Danger Zone → 要手动输入仓库全名确认。
- **Codespaces** = 云端开发环境,浏览器里就是完整 VS Code,环境预装好,可绕过本地配置的坑,有免费额度。

### branch vs 新项目(重要判断)

- **branch** = 同一个东西的不同版本线(同一本书的不同草稿)
- **新项目** = 完全不相干的两个东西(两本不同的书)

> 判断法:**同一个东西的不同尝试 → branch;不同的东西 → 新项目。**
> (类型检查练习 和 游戏 = 两本不同的书 → 开新项目,不要开 branch)

---

## 七、贯穿全天的一条主线

**这一路真正的成长不在"学了多少名词",而在建立了两个习惯:**

1. **不轻信(包括不轻信 AI)** —— 去跑、去验证、去查系统给的硬证据(地址栏、"Signed in as"、`npx pyright` 的输出)。
2. **会问"该不该、为什么"** —— 该不该 push、branch 还是新项目、这个权限该不该给。

> 工具再快,判断力得长在自己身上。

---

## 附:今日术语速查

| 术语 | 一句话 |
|---|---|
| Agent | LLM + 工具 + 循环 |
| Token | 模型处理文本的最小片段(比词碎) |
| LSP | 编辑器和语言智能之间的通用插头 |
| pyright | Python 的语言服务器(类型检查) |
| Plugin | 往 agent 身上挂东西的容器 |
| MCP | 工具的 USB 标准接口(client-server) |
| Pipeline | 流水线:数据逐站加工,上一步输出接下一步输入 |
| PAT | GitHub 个人访问令牌,密码级机密 |
| Collaborator | 被授予直接读写你仓库的人 |
| Contribution | GitHub 活动的可视化总账 |
| 副作用 | 改变外部真实状态、往往不可逆的操作 |
