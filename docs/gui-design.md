# GUI 设计文档 - 基于 ACP Workbench 与 TUI 工具能力

> 本文档基于当前项目代码修订。当前桌面原型不是直接调用 18 个 TUI 工具的 IDE，
> 而是 Electron 主进程启动 `omp acp` sidecar，前端通过 IPC 与 ACP runtime 交互。
> 因此 GUI 设计必须分成两层：
>
> 1. **ACP Workbench 层**：当前可落地，负责 workspace、runtime、session、prompt、权限、终端请求、设置和 transcript。
> 2. **Direct Tool API 扩展层**：未来补齐 IDE 级能力，允许 GUI 对 `lsp`、`search`、`ast_grep`、`debug` 等工具发起结构化操作。

---

## 当前代码事实

| 事实 | 影响 |
|------|------|
| 桌面原型是 Electron 形态，主进程使用 `BrowserWindow`、`ipcMain`、`contextBridge` | 文档不能假设 Tauri 或 Web-only 架构 |
| 主进程默认启动 `omp acp`，并通过 `@agentclientprotocol/sdk` 连接 | GUI 首要边界是 ACP session/prompt/event，而不是内建工具函数 |
| 前端 IPC 暴露 `startRuntime`、`createSession`、`sendPrompt`、`respondPermission`、`respondElicitation`、`runCommand` 等 | 当前 GUI 能稳定驱动 agent 会话，但不能天然驱动每个工具的所有原子能力 |
| ACP event mapper 将工具事件压成通用 `ToolKind`、content、locations、plan update | 可显示工具事件时间线，但 IDE 右键菜单和细粒度工具参数 UI 需要额外 API |
| TUI 工具注册表是动态的：受 settings、凭证、MCP、扩展、custom tools 影响 | GUI 不能写死“18 工具全集”；必须从 runtime 读取当前 active/all tools |

---

## 设计原则

| 原则 | 含义 |
|------|------|
| **用户意图优先** | 菜单显示“查找定义”“验证修改”，不要求用户记住 `lsp definition` 或 `bash test` |
| **ACP 优先** | v1 以会话、提示词、工具事件和权限处理为中心，复用当前稳定 runtime 边界 |
| **工具动态发现** | 命令面板和工具箱从 runtime 的 active/all tools、slash commands、settings 中生成 |
| **渐进披露** | 一级界面展示常用工作流；危险、底层、低频能力放入二级面板并带权限提示 |
| **权限可见** | 每个 read/write/exec 级操作必须显示范围、风险、预览和审批结果 |
| **约束即功能** | GUI 不提供无约束的 `grep/find/cat` shell 替代；提供结构化搜索/读取入口。注意这里不禁止 `find` 工具本身，而是避免用 shell 绕过专用工具和权限模型 |

---

## 总体架构

```
┌────────────────────────────────────────────────────────────┐
│ Renderer UI                                                │
│ Workbench · Transcript · Command Palette · Panels          │
└───────────────┬────────────────────────────────────────────┘
                │ safe IPC bridge
┌───────────────▼────────────────────────────────────────────┐
│ Electron Main Process                                      │
│ workspace selection · ACP process · permission dialogs      │
│ settings bridge · managed command runner                   │
└───────────────┬────────────────────────────────────────────┘
                │ stdio NDJSON / ACP
┌───────────────▼────────────────────────────────────────────┐
│ `omp acp` Runtime                                           │
│ session/new · session/prompt · session/list/fork/resume     │
│ tool events · plan updates · elicitation · terminal auth    │
└───────────────┬────────────────────────────────────────────┘
                │ AgentSession / ToolSession
┌───────────────▼────────────────────────────────────────────┐
│ Coding Agent Tools                                         │
│ read · search · lsp · bash · eval · debug · browser · ...   │
└────────────────────────────────────────────────────────────┘
```

### v1 边界

v1 GUI 不直接执行 `lsp`、`search`、`debug` 等工具调用。它通过自然语言 prompt、slash commands、ACP session updates 和 permission events 驱动 agent。

### v2 边界

v2 可以新增 Direct Tool API，让 GUI 对选中文件/符号发起结构化工具调用。该 API 必须复用现有 `AgentTool` approval、settings、tool discovery、event mapper 和 session persistence，不能绕过 agent runtime。

---

## 信息架构

```
┌─────────────────────────────────────────────────────────────┐
│  Menu Bar (File · Edit · Navigate · Run · View · Help)      │
├─────────────────────────────────────────────────────────────┤
│  Toolbar / Command Palette (Ctrl+K / Cmd+K)                 │
├────────────────┬──────────────────────┬─────────────────────┤
│ Activity Bar   │ Main Area            │ Context Panel       │
│                │                      │                     │
│ Workbench      │ Chat / Transcript    │ Runtime status      │
│ Search         │ Session timeline     │ Permission request  │
│ Files          │ Optional editor      │ Tool details        │
│ Tasks          │ Tool result cards    │ Plan / todos        │
│ Debug          │                      │ Diagnostics         │
│ Browser        │                      │ Settings summary    │
│ Toolbox        │                      │                     │
└────────────────┴──────────────────────┴─────────────────────┘
│ Status Bar: workspace · branch · runtime · session · model  │
└─────────────────────────────────────────────────────────────┘
```

### Activity Bar

| 标签 | v1 行为 | v2 扩展 |
|------|---------|---------|
| **Workbench** | 当前会话、prompt 输入、runtime 控制、工具事件流；session 列表 overlay（最近排序、resume、fork、close；持久删除作为后续能力） | 多会话并排、session tree 可视化 |
| **Search** | 通过 prompt/slash command 发起搜索；展示 search/find/ast_grep 工具事件 | 直接执行文本搜索和 AST 搜索 |
| **Files** | 展示工具事件中的 file locations；打开 workspace 文件 | 文件树、read/write/edit 结构化操作 |
| **Tasks** | 展示 ACP plan update 和 `todo_write` 结果 | 任务编排、subagent/job/irc 状态视图 |
| **Debug** | 展示 `/debug`、debug 工具事件、诊断日志 | 直接 DAP 控制面板 |
| **Browser** | 展示 browser 工具事件、截图、URL 和 JS run 结果 | 页面元素树、录制/回放、细粒度交互 |
| **Toolbox** | 当前可用工具、slash commands、settings 快捷入口 | 低频工具表单和 Direct Tool API |

---

## Command Palette

命令面板是全局入口，但不能声称“天然覆盖全部 127+ 原子能力”。它的覆盖范围分阶段：

### v1 覆盖

| 用户输入 | 行为 |
|----------|------|
| `new session` | 调用 ACP `session/new` |
| `resume session` | 打开 session overlay |
| `settings` | 打开 settings overlay |
| `tools` | 运行或展示当前工具列表 |
| `debug` | 打开 diagnostics/debug overlay 或发送 `/debug` |
| `search in files` | 生成受控 prompt，引导 agent 使用 `search` |
| `find references` | 在当前文件上下文存在时生成 prompt，引导 agent 使用 `lsp references` |
| `run tests` | 生成 prompt 或受控 managed command，进入权限审批 |
| `/model`、`/mcp`、`/agents` | 路由到 ACP 可用 slash commands 或对应 overlay |

### v2 覆盖

| 用户输入 | Direct Tool API |
|----------|-----------------|
| `go to definition` | `lsp { action: "definition", file, line, symbol }` |
| `rename symbol` | `lsp { action: "rename", file, line, new_name, apply }` |
| `search by structure` | `ast_grep { pat, paths, skip }` |
| `replace by structure` | `ast_edit { ops, paths }` |
| `start debug` | `debug { action: "launch", ... }` |
| `browser screenshot` | `browser { action: "run", code: screenshot helper }` |

命令面板的数据源必须是动态的：

- ACP advertised slash commands
- active/all tools from session runtime
- settings schema visible entries
- workspace/session actions
- recent commands and common workflows

---

## 工具能力清单

当前真实工具不是固定 18 个。基础内建工具包括：

| 工具 | GUI 分组 | 备注 |
|------|----------|------|
| `read` | Files | 文件、目录、归档、数据库、URL/内部 URI 等读取 |
| `write` | Files | 创建、覆盖、压缩包/SQLite 写入；write-tier |
| `edit` | Files | 精确编辑、patch/hashline、诊断联动；write-tier |
| `find` | Search | glob 文件发现；read-tier |
| `search` | Search | 正则文本搜索；read-tier |
| `ast_grep` | Search | 结构搜索；settings 控制可用性 |
| `ast_edit` | Search/Refactor | 结构替换；write-tier 或 internal URL read-tier |
| `lsp` | Navigate/Edit | diagnostics、definition、references、rename、code actions；部分 write-tier |
| `bash` | Run/Terminal | exec-tier；受拦截、审批和危险模式检测约束 |
| `eval` | Run/Notebook | Python/JS kernel；后端可用性动态决定 |
| `debug` | Debug | DAP 操作；read/write/exec 分层 |
| `browser` | Browser | 当前 schema 是 `open`/`close`/`run`，点击/输入/截图需通过 JS helper 或未来细分 API |
| `task` | Tasks | subagent 派发；受递归深度、async 设置影响 |
| `todo_write` | Tasks | plan/todo 状态；ACP 会映射为 plan update |
| `web_search` | Web | 受 settings 和 provider 可用性影响 |
| `github` | Integrations | GitHub 工具，settings/凭证控制 |
| `ssh` | Integrations | SSH 工具，配置控制 |
| `inspect_image` | Media | 图片理解，不是图像生成 |
| `render_mermaid` | Media | Mermaid 渲染 |
| `checkpoint` / `rewind` | Session | 检查点和回退，settings 控制 |
| `job` / `irc` | Tasks | 异步任务和 agent 间通信，受 async/irc 设置影响 |
| `search_tool_bm25` | Integrations | discovery mode 下可用 |
| `retain` / `recall` / `reflect` | Memory | hindsight/mnemopi backend 下可用 |

Custom/MCP/扩展工具也会出现在运行时工具列表里。`generate_image` 是凭证可用时注入的 custom tool，不应在 GUI 中当成永远存在的内建工具。

---


## Elicitation（Agent 提问）

GUI 需要处理两类用户提问路径：`ask` 工具在 UI-capable session 中的选择/输入流程，以及 ACP `unstable_createElicitation` 发出的 form/url 请求。当前 Electron 原型接收的是 ACP elicitation request（带 `requestedSchema` 字段），不是原始 `ask.questions` payload，所以 v1 应先把 ACP form 渲染正确；v2 再把 `ask` 的多问题导航、多选和 Other 输入做成更完整的工具专用 UI。

### UI 形式

根据问题复杂度，选择不同展示方式：

| 形式 | 适用场景 | 说明 |
|------|---------|------|
| **内联卡片** | 单选、确认/拒绝 | 嵌入会话流，不打断工作 |
| **模态对话框** | 多问题、多选、自由输入 | 聚焦用户注意力 |
| **通知栏** | 简单确认、超时取消 | 最小化干扰 |

### v1 必须支持

- **ACP form 渲染**：按 `requestedSchema.properties` 渲染 string/number/boolean/array 等字段。
- **选项列表**：schema enum 字段展示为选项，保留 label/description/默认值提示。
- **提交与取消**：用户提交时通过 `respondElicitation(requestId, values)` 返回；取消时返回空结果。
- **超时处理**：当前 desktop bridge 对 pending elicitation 有固定超时，超时后向 ACP 返回 cancel；UI 必须显示倒计时或超时状态，并记录到 session。
- **历史记录**：所有问答对保留在 session timeline 中，可回溯查看。

### v2 扩展

- **`ask.questions` 多问题体验**：`ask` 支持 `questions: []` 数组，GUI 可将多个问题聚合在一个模态里，或按步骤导航展示。
- **多选支持**：`multi: true` 的问题允许用户勾选多个选项。
- **自由输入**：每个问题可附加 "Other" 输入，允许用户键入自定义答案。
- **推荐/默认选项**：识别 recommended/default，并在超时策略明确时再执行自动选择；不能把 ACP cancel 误写成自动同意或自动选择。

### 数据流

```
Agent/tool → ACP unstable_createElicitation → Main Process → Renderer UI
                                                   → 用户操作 → IPC respondElicitation
                                                   → Main Process → ACP elicit response
                                                   → Agent 继续
```

---

## 权限与安全模型

GUI 必须把工具审批做成一等体验，而不是隐藏在日志里。

| 操作等级 | 示例 | GUI 表现 |
|----------|------|----------|
| read | `read`、`find`、`search`、`lsp definition` | 可自动执行或轻量提示；显示作用域 |
| write | `write`、`edit`、`ast_edit`、`lsp rename` | 必须显示文件范围、diff/preview、可取消 |
| exec | `bash`、`eval`、`debug launch`、`browser run` | 必须显示命令/代码、cwd、环境影响和审批按钮 |
| critical exec | destructive shell、远程执行、权限提升 | 强确认；禁止静默执行；保留拒绝路径 |

### 必须支持的审批 UI

- tool call 标题、工具名、参数摘要
- read/write/exec 等级 badge
- 受影响文件或 URL
- diff、stdout、stderr、截图等预览
- Approve / Deny / Always allow scoped action
- 审批结果写入 session timeline


## 错误与恢复

所有 happy path 设计都必须在错误场景下有明确表现。

| 场景 | GUI 表现 |
|------|---------|
| ACP runtime 崩溃/退出 | 状态栏变红 + 断开 badge + 重连按钮；GUI 必须清理或取消本地 pending permission/elicitation，并写入 "runtime lost" 诊断记录；若 runtime 已断开，不能假装已通知 agent |
| 工具执行超时 | 工具卡片显示 timeout badge + 可终止按钮；session 记录超时信息 |
| 审批请求超时（用户未操作） | 自动拒绝 + session 记录"approval timeout" |
| 权限/凭证不足的 tool 被触发 | 显示具体缺失项（missing credential / disabled in settings / approval denied） |
| 网络断开 | 离线 badge + 暂停自动操作；重连后提示恢复 |
| session 不存在/已关闭 | 清晰错误消息 + session 列表刷新入口 |
| tool 参数校验失败 | GUI 侧先校验再发送，减少 runtime 侧拒绝；若 runtime 拒绝则显示校验错误详情 |

### 错误展示原则

- 每类错误必须有**用户可见**的提示，不能静默吞掉。
- 错误消息必须有**可操作路径**（重试 / 取消 / 查看详情），不是只抛文本。
- 底层错误栈不直接暴露给用户，但提供"复制诊断信息"按钮。
- 所有错误写入 session timeline 和 diagnostics 面板。

---

## 右侧 Context Panel

Context Panel 根据当前焦点自动切换，但 v1/v2 能力不同：

| 焦点 | v1 显示 | v2 增强 |
|------|---------|---------|
| Runtime | ACP 状态、model、provider、context/token usage | runtime restart diagnostics |
| Permission | 当前审批请求、风险、参数摘要 | diff preview、scope allow rules |
| Tool event | tool kind、status、raw output、locations | 工具专用结构化结果组件 |
| Plan | ACP plan update、todo 状态 | 子任务 DAG、job/irc 状态 |
| Search | agent 产生的 search/ast_grep 结果 | 可交互筛选、分页、跳转 |
| File | locations 和文件预览 | LSP hover、symbols、diagnostics |
| Debug | debug 工具输出 | DAP variables、stack、breakpoints |
| Browser | URL、截图、console/text output | accessibility tree、element picker |

---

## 编辑器与右键菜单

文档原方案中的 IDE 级右键菜单可作为 v2 目标，不能作为 v1 可用能力。

### v1

- 点击 tool event location 打开文件。
- 可以从当前文件上下文生成 prompt，例如“解释这个函数”“查找引用”。
- 不直接执行 `lsp definition`、`rename`、`ast_edit`，除非通过 agent turn。

### v2

```
Right-click menu:
├── Go to Definition             (lsp definition, read)
├── Go to Type Definition        (lsp type_definition, read)
├── Find Implementations         (lsp implementation, read)
├── Find References              (lsp references, read)
├── Rename Symbol...             (lsp rename, write + preview)
├── Show Hover Info              (lsp hover, read)
├── Quick Fix...                 (lsp code_actions, write if apply=true)
├── AST Operations
│   ├── Search by Structure      (ast_grep, read)
│   └── Replace by Structure     (ast_edit, write + preview)
```

Direct Tool API 必须复用现有 `AgentTool.execute()`、approval 和 event mapping，避免 GUI 自己重新实现 LSP/DAP/AST 逻辑。

---

## 搜索面板

### v1 搜索

v1 搜索面板以 agent workflow 为主：

- 用户输入意图和范围。
- GUI 将其转换成 prompt 或 slash command。
- 结果来自 session updates 中的 tool events。
- 面板支持按文件、工具、状态筛选。

### v2 文本搜索

| 控件 | 说明 |
|------|------|
| 输入框 | regex pattern |
| 路径范围 | 文件、目录、glob、内部 URI |
| 选项 | 大小写、gitignore、分页 skip |
| 结果 | 行上下文、文件分组、跳转 |

### v2 AST 搜索

| 控件 | 说明 |
|------|------|
| 输入框 | AST pattern |
| 路径范围 | 文件、目录、glob、内部 URI |
| 结果树 | 匹配文件、位置、代码片段 |
| 重构入口 | 转入 `ast_edit` preview，不直接批量应用 |

注意：当前 `ast_grep` schema 没有显式语言选择器，语言由 native parser/文件类型推断。文档不得写成当前已有“AST 语言选择器”。

---

## 调试面板

Debug Activity 可以设计成完整 DAP 面板，但分阶段：

### v1

- 显示 `/debug` 命令入口。
- 展示 debug 工具事件。
- 展示日志、report bundle、raw SSE、system info 等 diagnostics。

### v2

```
┌─────────────────────────────────┐
│ [Start] [Step Over] [Step In]   │
│ [Continue] [Pause] [Stop]       │
├─────────────────────────────────┤
│ Breakpoints list                │
├─────────────────────────────────┤
│ Call Stack                      │
├─────────────────────────────────┤
│ Variables / Scopes              │
├─────────────────────────────────┤
│ Watch Expressions               │
├─────────────────────────────────┤
│ Output / Console                │
└─────────────────────────────────┘
```

DAP actions must respect current debug tool approval split:

- read-only: `output`、`threads`、`stack_trace`、`scopes`、`variables`、`disassemble`、`read_memory`、`loaded_sources`、`modules`、`sessions`
- exec/write: `launch`、`attach`、breakpoint mutations、execution control、`evaluate`、`write_memory`、`custom_request`

---

## 浏览器面板

原文档列出的“点击、输入、截图、录制/回放”是目标能力，不是当前 schema 的直接动作。

当前 `browser` 工具只有：

- `open`
- `close`
- `run`

因此 v1 应展示 browser 工具事件、URL、截图和 JS run 输出。v2 若要提供按钮式浏览器自动化，需要新增 helper catalog 或扩展 schema，例如：

- screenshot helper
- accessibility snapshot helper
- click/type/select helpers
- wait helpers
- recording/replay abstraction

这些 helper 最终仍应走 `browser run` 或新增工具动作，并保留 exec-tier 审批。

---

## 工具箱

工具箱不能写死固定卡片。它应由运行时能力生成：

```
Toolbox
├── Files
│   ├── read
│   ├── write
│   └── edit
├── Search / Navigate
│   ├── find
│   ├── search
│   ├── ast_grep
│   ├── ast_edit
│   └── lsp
├── Run / Debug
│   ├── bash
│   ├── eval
│   └── debug
├── Browser / Web
│   ├── browser
│   └── web_search
├── Tasks / Agents
│   ├── task
│   ├── todo_write
│   ├── job
│   └── irc
├── Integrations
│   ├── github
│   ├── ssh
│   ├── MCP tools
│   └── extension tools
├── Media
│   ├── inspect_image
│   ├── render_mermaid
│   └── generate_image (only when registered)
└── Memory / Session
    ├── checkpoint
    ├── rewind
    ├── retain
    ├── recall
    └── reflect
```

每个卡片必须显示：

- 是否当前 active
- 是否 settings 禁用
- approval 等级
- 是否来自 builtin、custom、MCP、extension
- 是否需要凭证

---


## 终端交互

开发者工具中 terminal 是刚需，但要区分 agent tool call 与桌面本地命令。`bash` 和 `eval` 由 agent 通过 ACP tool events 触发；桌面命令面板里的受控命令则通过 Electron main 的 managed command runner 执行。

### v1 终端

- 内联终端卡片，展示 `bash` 和 `eval` 工具事件的 stdout/stderr 输出；v2 再做完整实时流。
- 用户可在输入框中键入受控命令，通过 `runCommand` IPC 交给 Electron main 的 managed command runner 执行；这不是 ACP tool call，结果进入本地 command log。
- Agent 自主触发的 `bash`/`eval` 仍通过 ACP tool events 展示，走 tool approval 和 session timeline。
- 每条命令的输出显示退出码和耗时。
- 长输出可折叠（默认折叠超出 50 行部分）。
- 输出文本可搜索。

### v2 终端增强

| 特性 | 说明 |
|------|------|
| 多 tab 终端 | 区分多个并行 bash/eval 会话 |
| 终端复用 | 常用命令保存为快捷入口 |
| eval notebook | 类 Jupyter 单元格视图（Python/JS kernel 持久化） |
| 文件输出预览 | 脚本生成的图片/文件可直接查看/下载 |
| 命令审批 | exec-tier 命令进入审批流程时显示完整命令、cwd、环境 diff |

### 终端视图布局

```
┌──────────────────────────────────────┐
│ 终端                                 │
├──────────────────────────────────────┤
│ [bash] [eval-py] [eval-js]  [+new]   │  ← tab bar
├──────────────────────────────────────┤
│ $ cd /workspace && bun test          │  ← 输入行
│ ✓ 42 passed (1.2s)                   │  ← output (需审批则先弹出审批)
│ $ _                                  │
└──────────────────────────────────────┘
```

终端视图可以放在 Activity Bar 的 Workbench 面板内（v1），或作为独立 Activity（v2）。

---

## 工作流整合

GUI 应提供“意图工作流”，但要标明由 agent 执行还是 GUI 直连工具执行。

| 用户意图 | v1 执行方式 | v2 扩展 |
|----------|-------------|---------|
| 找 bug 原因 | prompt agent，agent 自行组合 search/lsp/debug/bash | 可提供搜索/断点/日志的半自动向导 |
| 重构符号 | prompt agent 或 slash command | `lsp rename` + diff preview |
| 理解代码 | prompt agent，展示引用和工具事件 | hover/references/symbols 直连 |
| 验证修改 | prompt agent 或 managed command，进入审批 | 测试任务面板、历史结果 |
| 运行自动化测试 | prompt 或受控 command runner | job 面板和测试结果结构化 |
| 写数据脚本 | prompt agent 使用 `eval` | notebook-like eval UI |

---

## 设置与运行时状态

GUI 必须把 settings 当成运行时配置，而不是静态表单：

- 从 settings schema 读取 visible settings。
- 修改设置后要同步 runtime 或提示需要 restart。
- 工具是否出现由 settings、凭证、MCP、扩展和当前 session 决定。
- `generate_image`、`web_search`、`github`、`ssh`、memory tools 等不能只靠静态菜单判断可用。

Status Bar 至少显示：

- workspace path
- git branch / dirty state
- ACP runtime state
- active session id/title
- model/provider/thinking
- context/token usage
- tool/extension status摘要

---

## 实施阶段

### Phase 1 - ACP Workbench 固化

目标：让当前桌面原型可复现、可运行、可验证。

- 补齐 `packages/desktop-app` 源码、`package.json`、构建脚本和 workspace 配置。
- 保留 Electron + ACP sidecar 路线。
- 完成 workspace/runtime/session/prompt/permission/elicitation/terminal/settings 基础 UI。
- 工具事件以结构化 cards 展示。不同工具的输出差异大，需按类型渲染：
  - `read` → 代码预览 + 文件路径 + 行范围
  - `search` / `ast_grep` → 匹配结果分组列表 + 行号 + 预览片段
  - `browser` → 截图图片 + URL + console/text 输出
  - `bash` / `eval` → 终端输出 + 退出码 + 耗时
  - `lsp references` / `definition` → 文件位置列表 + 跳转按钮
  - `edit` / `write` → 修改前后的 diff
  - 通用兜底 → JSON 视图 + 复制按钮
  v1 实现通用渲染 + tool-type 分发判断；v2 为每个工具编写专用 result component。
- 验证：desktop build、ACP runtime start、new session、send prompt、permission request、settings schema。

### Phase 2 - 工具发现与工具箱

目标：让 GUI 知道当前 runtime 真实工具，而不是写死清单。

- 通过 ACP extension context 或新增 runtime endpoint 暴露 active/all tools。
- 展示工具来源、approval、enabled/disabled、凭证状态。
- 命令面板合并 slash commands、tools、settings、workspace actions。
- 验证：不同 settings/凭证/MCP 状态下工具列表正确变化。

### Phase 3 - Direct Tool API

目标：支持 GUI 直接发起结构化 read/search/lsp/ast/debug/browser 操作。

- 定义 direct tool request/response 协议。最小协议轮廓：
  ```typescript
  // Request（Renderer → Main → ACP runtime）
  interface DirectToolRequest {
    tool: string;                  // 工具名，如 "lsp"、"search"
    action?: string;               // 可选 UI 语义，如 "definition"；真实动作仍放在 params 中
    params: Record<string, unknown>; // 工具参数
    sessionId: string;             // 所属 session
  }

  // Response（ACP runtime → Main → Renderer）
  interface DirectToolResponse {
    ok: boolean;
    data?: unknown;               // 执行结果
    error?: { code: string; message: string };
    toolKind: string;             // 用于 UI 选择渲染组件
    locations?: Array<{ file: string; line?: number; column?: number }>;
    approval?: {
      decision: "read" | "write" | "exec" | "reject";
      reason?: string;
      scope?: string;
    };
  }
  ```
- 复用 `AgentTool` approval、session persistence、event mapper。
- 先实现 read/search/find/lsp readonly。
- 再实现 write-tier edit/ast_edit/lsp rename。
- 最后实现 exec-tier debug/browser/eval/bash。
- 验证：每类 approval、取消、错误、session replay、tool locations。

### Phase 4 - IDE 级交互

目标：补齐编辑器、右键菜单、搜索面板、调试面板、浏览器面板。

- 编辑器文件树和 tabs。
- LSP hover/definition/references/diagnostics。
- AST 搜索和 preview rewrite。
- DAP 控制面板。
- Browser helper catalog。
- 验证：Playwright/Electron 截图、权限流、工具结果渲染、跨平台 smoke。

---

## 已修正的原方案问题

| 原表述 | 修正 |
|--------|------|
| “GUI 基于 TUI 18 工具能力全集” | 改为 ACP Workbench 优先，工具全集是 runtime 动态能力 |
| “命令面板覆盖全部 127+ 能力” | 改为 v1 覆盖 session/slash/prompt，v2 通过 Direct Tool API 扩展 |
| “系统禁止 grep/find/cat” | 改为禁止 shell 绕过专用工具，不禁止 `find` 工具 |
| “generate_image 是固定工具” | 改为凭证可用时注入的 custom tool |
| “browser 直接有点击/输入/截图动作” | 改为当前 schema 是 `open/close/run`，细粒度能力需 helper 或扩展 |
| “AST 搜索有语言选择器” | 改为当前 schema 未提供语言选择器 |
| “右键菜单直接集成所有 LSP 能力” | 改为 v2 目标，v1 只从上下文生成 agent prompt |
| “工具箱固定卡片” | 改为由 active/all tools、settings、MCP、extensions 动态生成 |

---

## 验收标准

文档后续落地时，以这些条件判断方案是否真实可行：

- GUI 能在没有 Direct Tool API 的情况下完成 ACP Workbench 的核心任务。
- 所有工具能力显示都来自 runtime 或明确标注为未来目标。
- 写入和执行类操作都有审批 UI 和可追踪 session 记录。
- 当前不可用能力不在一级 UI 中伪装成可用。
- desktop 包可从源码构建，而不是只提交 `dist/`。
- Playwright/Electron 截图覆盖 startup、command palette、settings、session、permission、tool card、diagnostics。

---

## 测试策略

GUI 测试分为三个层级，每层覆盖不同粒度的风险。

### 层级 1：IPC Handler 单元测试

- 测试 `ipcMain` 处理函数的参数校验、错误处理和边界条件。
- Mock ACP runtime 连接，验证 session/prompt/permission/elicitation IPC handler 发送的 NDJSON 消息格式正确；`runCommand` 另测本地 managed command runner 的命令白名单、cwd、退出码和输出采集。
- 覆盖：`startRuntime`、`createSession`、`sendPrompt`、`respondPermission`、`respondElicitation`、`runCommand`。

### 层级 2：UI 组件测试

- 渲染测试：权限对话框在 read/write/exec 不同等级下显示正确的 badge、按钮和范围。
- 交互测试：命令面板搜索、session 列表选择、工具卡片展开/折叠。
- 边界测试：ACP elicitation 多字段渲染、`ask.questions` 多问题扩展、超长路径截断、空结果列表。
- 推荐框架：Vitest + @testing-library/react（或对应 Electron 组件框架）。

### 层级 3：E2E（Playwright + Electron）

全链路截图覆盖，绑定到 CI：

| 场景 | 覆盖内容 |
|------|---------|
| startup | app 启动 → workspace 选择 → ACP runtime 启动 → session 就绪 |
| command palette | 打开面板 → 搜索 → 选中功能 |
| settings | 打开 → 修改 → 同步 runtime |
| session | 创建 → 发 prompt → 查看工具事件 → 列表切换 → resume |
| permission | tool call 触发 → 显示 diff → approve → session 记录 |
| error runtime crash | runtime 退出 → 断开 badge → 重连 |
| terminal | bash 执行 → 输出展示 → 折叠 → 搜索 |
| elicitation | agent 提问 → 展示选项 → 选择 → 返回结果 |
| tool cards | search/lsp/browser/bash/edit 各类型渲染 |
| dark/light theme | 两种主题截图 diff |

Phase 1 补齐 `packages/desktop-app/package.json` 后，目标验证命令形态如下。命令名是目标约定，落地时以 desktop package scripts 为准：

```
cd packages/desktop-app
# lint + type check
bun run check
# unit + component tests
bun run test
# e2e with Playwright
bun run test:e2e
```

desktop package 纳入 workspace 后，CI 应把这三层验证作为合并门禁。
