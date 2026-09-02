# Codex CLI 远程控制学习方案（小北版：小白 → 专家）

> 目标对象：小北（从零起步）
> 学习方式：全程 AI 执行命令 + 逐条解释，小北提问、验收、动手练；每课先出方案，确认后再动手
> 环境前提：WSL2（Linux），Node 18+，能联网；本机可复用 DeepSeek 网关省钱
> 来源：`cc/codex-claude-远程控制-知识点.md`

---

## 一、这个方案教什么

一句话：让小北从「会装会用 Codex」走到「能控制它、改造它、远程指挥它」。

四个里程碑：

1. **会用**（阶段 0-1）：装好、登录、对话式让它干活
2. **会控**（阶段 2-3）：沙箱、权限、脚本化批量执行
3. **会改**（阶段 4-5）：接 MCP、自定义模型、多智能体并行
4. **会遥控**（阶段 6-7）：本机跑，手机/Web 遥控（本项目终极目标）

**毕业标准（达到即算"专家"）**：
- 交互模式 + exec 模式都熟练，能独立完成「读代码 → 改代码 → 跑测试 → 交 diff」闭环
- 能按需配沙箱/权限/自定义 provider，不被默认设置卡住
- 会写 MCP server 把私有工具暴露给 agent
- 会配多智能体并行 + CSV 批量派活
- 会排错（bwrap 沙箱、日志、模型元数据警告）
- 能通过开源方案手机遥控本机会话，并讲得清安全边界

---

## 二、阶段 0：装好它

### 课 0 — Codex 是什么 + 安装 + 第一次任务

**目标**：跑通 `codex --version`，完成第一个真实任务。

**核心知识点**
- Codex CLI 是 OpenAI 开源的终端编码 agent，仓库 `openai/codex`，Apache-2.0 许可
- 主体是 **Rust** 写的（约 95%，60+ 个 crate）；核心分四块：`core`（agent 主循环 + 工具执行）、`cli`（终端界面）、`tui`（文本界面）、`headless`（给 VS Code/Web 用的 JSON-RPC）
- 与 Claude Code 同生态位；系统提示词就是一个纯 Markdown 文件 `codex-rs/core/prompt.md`
- 凭证在 `~/.codex/auth.json`，配置在 `~/.codex/config.toml`，会话状态存本地 SQLite（云端无状态）

**实操（AI 执行）**
- `npm install -g @openai/codex@latest`（或 GitHub releases 预编译二进制）
- 登录二选一：
  - 个人：`codex login`（走 ChatGPT 订阅）
  - 开发者：`export OPENAI_API_KEY=sk-...` + `codex login --api-key`
- 建一个空目录，让 Codex 写一个小项目（比如命令行计算器），让它自己跑、自己修 bug

**验收标准**：能新建目录 → `codex` 进入 → 完成一个小任务 → `/exit` 退出。

---

## 三、阶段 1：交互模式

### 课 1 — REPL 与斜杠命令

**目标**：把 REPL 当日常工具用。

**核心知识点**
- 斜杠命令：`/help`（帮助）、`/status`（看用量）、`/clear`（清会话）、`/copy`（复制输出）、`/model`（换模型）、`/new`（新会话）
- 界面三区：当前文件、事件、编辑器

**实操**：逐一敲一遍斜杠命令，说清每个干什么；小北现场提问我答。

**验收**：不用看文档能说出 6 个斜杠命令各自用途。

### 课 2 — 读懂陌生项目 + 自动跑测试

**目标**：学会让 Codex 干「读和验」的活，不只会写。

**核心知识点**
- `--sandbox read-only` 下让它 explain 整个 repo（安全读，不乱写）
- 让它跑测试的约定：「红就修，绿了再给 diff」——先让 agent 自己把测试跑绿，再交付改动

**实操**：挑一个已有项目（比如本机 cc/ 下的小脚本），先 read-only 讲解，再让它加一个功能并跑通测试。

**验收**：能解释清楚「读模式」和「写模式」切换；拿到一份测试全绿 + 有 diff 的改动。

### 课 3 — 会话管理 + 结果导出

**目标**：会话能续、能分叉、能导出，不白干。

**核心知识点**
- `codex resume` 续上次会话；`codex fork` 从某点分叉；`codex exec resume --last` 回放上一条
- 会话文件在 `~/.codex/sessions/`
- 导出：`--json` 结构化输出；`/copy`；`/share` 生成分享链接

**实操**：完成一个任务后 resume 续上；用 `--json` 把一次 exec 结果导出成文件。

**验收**：会话中断后能续上；能拿到机器可读的 JSON 结果。

---

## 四、阶段 2：沙箱与权限

### 课 4 — 三种沙箱 + 审批策略

**目标**：搞清楚 Codex 默认「怎么被限制」，以及怎么放行。

**核心知识点**
- 三种沙箱：`read-only`（只读）/ `workspace-write`（默认，可写工作区，禁网禁越界）/ `danger-full-access`（全放开）
- 审批策略：`on-request`（沙箱内自动放行，越界才问，推荐）/ `never` / `untrusted`
- 甜点组合（存进 config.toml）：
  ```toml
  sandbox = "workspace-write"
  approval_policy = "on-request"
  ```
- 底层：macOS 用 Seatbelt，Linux 用 Bubblewrap + Seccomp，Windows 用 Restricted Tokens

**实操**：三种沙箱各跑一次任务，感受差异；把甜点组合写进 `~/.codex/config.toml`。

**验收**：能说出每种沙箱「能干什么、不能干什么」；config 改对。

### 课 5 — 扩权 + execpolicy 规则引擎

**目标**：给 agent 精准开权限，而不是一刀切放开。

**核心知识点**
- 扩写权限：config 里
  ```toml
  [sandbox_workspace_write]
  writable_roots = ["/path/to/allow"]
  ```
- execpolicy 规则引擎：`.codex/execpolicy/*.rules` 写规则，例如
  ```text
  prefix_rule("npm install", "allow", "安装依赖")
  ```
  `codex execpolicy check` 验证规则

**实操**：给一个项目配一条 `npm install` 放行规则；跑 `codex execpolicy check`。

**验收**：能解释「writable_roots 扩目录」和「execpolicy 控命令」的区别，并各配一条。

---

## 五、阶段 3：exec 脚本化

### 课 6 — 一次性模式 + CI 集成

**目标**：把 Codex 从「聊天工具」升级成「命令行工具」，能进流水线。

**核心知识点**
- `codex exec "任务"`（简写 `codex e`）：一次任务，非交互
- `--json`：换行分隔 JSON，可接脚本/工具链
- `--model gpt-5-codex`：换档
- `--search`：先联网查最新文档再动手
- CI 无人值守：`codex exec --full-auto "..."`（= on-request + workspace-write）

**实操**：写一个 shell 脚本，调 `codex exec --json` 批量处理一堆文件，结果落到文件里。

**验收**：能从命令行一行跑完一个任务并拿到结构化结果；能讲清 `--full-auto` 的权限含义。

---

## 六、阶段 4：MCP 与模型

### 课 7 — 接 MCP + 自己写 MCP server

**目标**：让 agent 用上「你私有的工具」。

**核心知识点**
- 接现成 MCP：`codex mcp add github "npx -y ..."`
- 自己写 MCP server：暴露私有工具给 agent 调（例如公司内部 API、本机命令）
- Codex 反当 MCP server：暴露 `codex()` / `codex-reply()`，可以被 Claude Code 等调用（连接本项目）

**实操**：先用一个现成 MCP（比如 GitHub），再写一个最小 MCP server 暴露一个自定义工具，让 Codex 调用它。

**验收**：Codex 能用上私有工具；能讲清「agent 调 MCP」和「Codex 当 MCP server」两种方向。

### 课 8 — 自定义 provider + 模型元数据

**目标**：不花 OpenAI 的钱也能跑（复用本机 DeepSeek 网关思路）。

**核心知识点**
- config `[model_providers]` 指向任意 OpenAI 兼容端点：ollama / 百炼 / DeepSeek / 第三方
- 模型元数据警告：
  ```text
  Model metadata for 'xxx' not found. Defaulting to fallback metadata...
  ```
  含义：模型不在内置目录 → 用兜底（context_window=272000）；真实窗口小会频繁压缩，填太大会被服务端 HTTP 400
  - 修复：config 加 `model_catalog_json = "~/.codex/xxx-models.json"`，写真实元数据
  - 坑：顶层 `model_context_window` 会被 clamp 到目录上限，单独设没用
  - 改完要完全重启 + 新会话才生效

**实操**：给本机 DeepSeek 网关配一个 provider；如遇元数据警告，写 model_catalog_json 修掉。

**验收**：Codex 能跑在 DeepSeek 上且无警告；能复述「兜底元数据有什么坑」。

---

## 七、阶段 5：多智能体

### 课 9 — subagents + 自定义 agent + 并行 + CSV 批量

**目标**：从「一个 agent 干活」到「一组 agent 干活」。

**核心知识点**
- subagents 默认开启：`default` / `worker` / `explorer` 三种内置角色，主线程只收摘要
- 自定义 agent：`~/.codex/agents/*.toml` 定义 role / system commands / model / sandbox
- 最多 6 路并发
- `spawn_agents_on_csv`：CSV 驱动批量派活，`{column_name}` 占位符

**实操**：定义一个「worker」agent；用 CSV 喂一批任务批量执行。

**验收**：能说出三个内置角色区别；自定义过一个 agent；CSV 批量跑通。

---

## 八、阶段 6：专家级

### 课 10 — AGENTS.md + 排错

**目标**：成为「能修 Codex 的人」。

**核心知识点**
- `AGENTS.md`：项目指令文件，Codex 读，Claude Code 也读（放项目根目录）
- 排错三板斧：
  - `--verbose` 开详细日志
  - WSL2/Linux 沙箱报错先查 bwrap：`sudo apt install bubblewrap`
  - 日志在 `~/.codex/log/`
- 遇到上面课 8 的元数据警告，知道怎么修

**实操**：给一个项目写 AGENTS.md；故意制造一个沙箱报错再排掉。

**验收**：独立排查一次报错并给出结论；AGENTS.md 被 agent 正确读取执行。

---

## 九、阶段 7：远程控制（本项目终极目标）

### 课 11 — 官方远程功能 vs 本机现实

**目标**：搞懂「为什么官方远程用不了，路在哪」。

**核心知识点**
- Claude Code 两大官方功能：
  - Remote Control（个人向）：手机/Web 遥控本机会话，`claude remote-control`；要求 claude.ai 订阅 + 直连 api.anthropic.com，**网关/代理一律不行**
  - self-hosted-runner（企业向）：长驻 runner 接收会话，要求组织环境密钥
- 本机跑 DeepSeek 网关 → 两个官方功能都被「钥匙」（接入渠道）卡死，不是模型问题
- 出路：模型无关的开源方案（下面两课）

**实操**：`claude self-hosted-runner --help` 过一遍参数，理解官方方案的全貌。

**验收**：能讲清「官方远程的门票是什么，本机差在哪」。

### 课 12 — 开源替代实战

**目标**：挑一条路真跑通手机遥控。

**核心知识点**（2026-08 现状）
- **Paseo**（`getpaseo/paseo`，AGPL）：daemon + iOS/Android/桌面/Web/CLI，39 种 agent，worktree 隔离，手机看 diff/审批/语音；`npm i -g @getpaseo/cli && paseo` 扫码配对；国内延迟需自建 frp/ngrok 中转
- **VibeAround**（中文友好）：自托管编码中心，API Bridge 在 OpenAI/Anthropic/Gemini 格式间互转，明确支持 DeepSeek
- **yepanywhere / remote-claude-code**：轻量 npm 一条命令，PWA + 隧道
- **兜底最稳**：tmux + Tailscale + 手机 SSH

**实操**：按推荐顺序（Paseo → VibeAround → 轻量 → tmux 兜底）先装一个跑通「手机看到本机会话」。

**验收**：手机能连上本机会话，看得到输出/审批一次操作；讲清装的是哪条路线、安全边界在哪。

### 课 13 — 安全基线

**目标**：遥控能力越大，越要知道钥匙在哪。

**核心知识点**
- relay 别用陌生人的（你的代码/令牌从别人中转）
- 令牌管好：`~/.codex/auth.json`、`~/.claude/` 凭证不要明文贴出来、不要进 git 历史
- agent 权限 = 钥匙：遥控打开的权限就是远端能拿到的权限，手机丢=钥匙丢
- 只出站连接、不开端口、端到端加密 才是推荐形态

**实操**：给自己所有机器/中转做个权限清单，逐项打勾。

**验收**：能列出自己这套远程方案里「信任什么、不信任什么」。

---

## 十、毕业项目

**题目**：手机遥控本机修一个真实 bug。

**要求**
- 小北在电脑端开一个会话（Claude Code 或 Codex），用阶段 7 学到的方案从手机连上
- 手机上下发一个任务（如「修这个脚本的报错」+ 贴报错）
- 全程见证：读代码 → 定位 → 改 → 跑测试 → 交付 diff
- 最后口述 3 条安全边界

**过了这关 = 从「小北」到「专家」。**

---

## 参考来源
- Codex：GitHub `openai/codex`、releases、Approvals 文档、Codex 避坑指南（腾讯云）
- Claude：claude.com/blog/run-claude-code-sessions-on-your-own-compute、code.claude.com/docs/en/remote-control
- 社区：getpaseo/paseo、VibeAround（V2EX）、yepanywhere、telecode、remote-claude-code、AgentFlow
