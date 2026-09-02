# Codex CLI + Claude 远程控制 — 知识点汇总

> 来源：2026-08-15 对话整理，用于后续制定学习方案
> 本机环境：WSL2（Linux 内核），Claude Code CLI，模型走 DeepSeek 网关

---

## 一、Codex CLI：从小白到专家（30 条分阶段）

### 阶段 0：装好它
1. **定位**：OpenAI 开源的终端编码 agent，Rust 写，与 Claude Code 同生态位。`codex --help` 看能力清单。
2. **安装**：`npm install -g @openai/codex@latest`（Node 18+）；macOS `brew install --cask codex`；或 GitHub releases 预编译二进制。装完 `codex --version`。
3. **登录**：个人 `codex login`（走 ChatGPT 订阅，Plus $20/月含用量）；开发者 `export OPENAI_API_KEY=sk-...` + `codex login --api-key`。容器内 `--device-auth`。凭证在 `~/.codex/auth.json`，配置 `~/.codex/config.toml`。
4. **第一次任务**：建目录 → `codex` → 让它写小项目、自己跑、自己修 bug。

### 阶段 1：交互模式
5. **REPL**：斜杠命令 `/help` `/status`（用量）`/clear` `/copy` `/model` `/new`。
6. **读懂陌生项目**：`--sandbox read-only` 下让它 explain 整个 repo。
7. **让它自己跑测试**：改完让它"红就修，绿了再给 diff"。
8. **会话管理**：`codex resume` 续上次；`codex fork` 分叉；`codex exec resume --last`。会话在 `~/.codex/sessions/`。
9. **结果导出**：`--json` 结构化输出；`/copy`；`/share` 生成分享链接。

### 阶段 2：沙箱与权限
10. **三种沙箱**：`read-only` / `workspace-write`（默认，禁网禁越界）/ `danger-full-access`。
11. **审批策略**：`on-request`（沙箱内自动放行，越界才问，推荐）/ `never` / `untrusted`。老版叫法 suggest/auto-edit/full-auto。
12. **甜点组合**：`--sandbox workspace-write --ask-for-approval on-request`，存进 config.toml。
13. **扩写权限**：config 里 `sandbox_workspace_write.writable_roots = ["/path"]`。
14. **execpolicy 规则引擎**：`.codex/execpolicy/*.rules` 写 `prefix_rule("npm install","allow",...)` 等；`codex execpolicy check` 验证。

### 阶段 3：exec 脚本化
15. **一次性模式**：`codex exec "任务"`（简写 `codex e`）。
16. **--json**：换行分隔 JSON，可接工具链。
17. **--model 换档**：`codex exec --model gpt-5-codex "..."`。
18. **--search 联网**：查最新文档再动手。
19. **CI 集成**：`codex exec --full-auto "..."`（on-request + workspace-write，无人值守）。

### 阶段 4：MCP 与模型
20. **接 MCP**：`codex mcp add github "npx -y ..."`。
21. **自己写 MCP server**：教它用私有工具。
22. **Codex 反当 MCP server**：暴露 `codex()` / `codex-reply()`，可被 Claude Code 等调用。
23. **自定义 provider**：config `[model_providers]` 指向任意 OpenAI 兼容端点（ollama/百炼/第三方）。
24. **订阅**：Plus $20/月，Pro $100/$200 高档位，`/status` 看用量。

### 阶段 5：多智能体
25. **subagents 默认开启**：default / worker / explorer 三种内置角色，主线程只收摘要。
26. **自定义 agent**：`~/.codex/agents/*.toml` 定义 role/system commands/model/sandbox。
27. **并行**：最多 6 路并发。
28. **spawn_agents_on_csv**：CSV 驱动批量派活，`{column_name}` 占位符。

### 阶段 6：专家级
29. **AGENTS.md**：项目指令文件（Codex 读，Claude Code 也读）。
30. **排错**：`--verbose`；WSL2/Linux 沙箱报错先查 `bwrap`（`sudo apt install bubblewrap`）；日志 `~/.codex/log/`。

---

## 二、Codex 模型元数据警告（重要）

**报错原文**：`Model metadata for 'xxx' not found. Defaulting to fallback metadata; this can degrade performance and cause issues.`

- **含义**：model 不在 Codex 内置目录 → 用兜底元数据：`context_window = 272000`，按 95% 生效 ≈ 25.8 万 token。
- **后果**：若真实窗口小于兜底 → 频繁压缩；若填太大 → 服务端 HTTP 400。**本地配置改不了服务端上限**。
- **修复**：config.toml 加 `model_catalog_json = "~/.codex/xxx-models.json"`，文件里写真实元数据（slug/context_window/max_context_window/auto_compact_token_limit/effective_context_window_percent）。
- **坑**：`model_context_window` 顶层配置会被 clamp 到目录里 max_context_window，单独设没用。
- **旁注**：OpenRouter 用 `env_key` auth 可能不触发目录刷新，改用 command 式 auth 块。
- **改完完全重启 Codex + 新会话才生效。**

---

## 三、Claude Code 远程两大官方功能（别混！）

### A. Remote Control（个人向，手机/Web/桌面遥控本机会话）
- 命令：`claude remote-control`（简写 `claude rc`）；会话内 `/rc`；`/config` 可全局开启。
- 连法：终端出 URL，按空格出二维码；手机扫；或 claude.ai/code 里找绿点会话。
- **要求（硬性）**：claude.ai 订阅登录（全权限 token）；直连 api.anthropic.com；**网关/代理/Bedrock/Vertex/Foundry 一律不行**。
- 当前：Research Preview，Claude Max 优先，Pro 陆续开放。
- 架构：**只出站连接**，不开任何端口，端到端加密，代码不离开本机。
- 限制：每实例一个远程会话；终端要开着；断线超 10 分钟会话过期；交互选择器（/plugin 等）只能本地。

### B. self-hosted-runner（企业向，共享基础设施执行）
- **2026-08-07 随 v2.1.224 发布**；beta、默认关闭、只给 Team/Enterprise。
- 部署长驻 runner 进程（launchctl/systemd 服务），接收会话并为每个会话起一个 Claude Code 进程。
- 模式：Fixed（固定数量）/ On-demand（按需伸缩）。
- **注册必需**：环境密钥 `SELF_HOSTED_RUNNER_ENVIRONMENT_SECRET`（组织控制台发）；`--api-url` 默认 https://api.anthropic.com。
- 健康检查：`/healthz`、`/metrics`（`claude_code_self_hosted_runner_*` Prometheus 指标）。
- 环境变量：`CLAUDE_RUNNER_*` 系列（会话 ID、池 ID、checkout 路径、work-order 文件等）。
- 隔离：每会话独立 checkout；可选 `--lock-to-account` 锁账号。

### C. 关键参数表（2.1.233 实测 help）
```bash
claude self-hosted-runner \
  --environment-secret-file <path>     # 环境密钥（无它注册不上）
  --lock-to-account <id>               # 只接某账号的活
  --capacity <n>                       # 最大并发会话（默认 1）
  --base-dir <path>                    # 会话 checkout 根目录（Windows 必填）
  --exec-path <path>                   # 子会话用哪个二进制（默认自身）
  --hooks-dir <path>                   # 生命周期钩子（checkout/command/post-session）
  --health-port <port>                 # /healthz 端口（默认 8080）
  --log-file <path>                    # 日志追加到文件
  --exit-if-unused-min <n>             # 空闲缩容
  --retire-at <epoch>                  # 定时退役
  --release-idle-session-min <n>       # 会话无输入释放
  --kill-session-after-min <n>         # 失控会话兜底杀掉
  --use-anthropic-git-proxy            # git 走会话创建者 GitHub 授权
  --configure-git                      # 设置 git 身份 + 签名
```
systemd 托管示例 + `curl 127.0.0.1:8080/healthz` 探活。

---

## 四、DeepSeek 网关的现实约束（本机关键结论）

- **本机 Claude Code 走 DeepSeek 网关**（deepseek-v4-flash）。
- **官方两个功能全被网关卡死**：Remote Control 要求 claude.ai 订阅 + 直连 api.anthropic.com；self-hosted-runner 要求组织环境密钥。**不是模型问题，是"钥匙"（接入渠道）问题。**
- 结论：走官方 = 必须换 Claude 订阅 + 直连，与"用 DeepSeek 图便宜/灵活"的初衷冲突。
- **出路**：模型无关的开源方案（见下），它们只驱动本地 CLI，DeepSeek 原样可用。

---

## 五、开源替代项目盘点（2026-08 现状）

### 综合最优
- **Paseo**（`getpaseo/paseo`，AGPL）——daemon + iOS/Android/桌面/Web/CLI 多端；39 种 agent；worktree 隔离；手机看 diff/审批/审 PR；语音；Docker。
  `npm install -g @getpaseo/cli && paseo`，扫码配对。
  ⚠️ 官方 relay 在海外，国内延迟 500ms+，需自建 frp/ngrok/QuickDesk 中转压到 ~50ms。
- **VibeAround**（V2EX 开源，中文友好）——自托管编码中心；**API Bridge** 在 OpenAI/Anthropic/Gemini 格式间互转；明确支持 DeepSeek；Web 面板 + Web 终端。

### 轻量 Web/PWA（npm 一条命令）
- **yepanywhere**：`npm i -g yepanywhere`。无账号无数据库，端到端加密，锁屏通知回复，手机传文件，断线不中断。
- **remote-claude-code**：Cloudflare Quick Tunnel + PWA，零配置公网，流式显示思考块/工具调用，内置 Web 终端。
- **telecode**：移动优先，双端只出站连接 relay（不开端口），端到端加密，手机建 PR、审批门、多机器。
- **hapi**：本地跑多 agent，手机 PWA/Telegram Mini App，点按审批，WireGuard+TLS。
- **clauster**：homelab 风格 Web UI，起停遥控桥、权限模式、CLAUDE.md 编辑器、实时日志、成本统计。

### Android App + 自建中转（中文）
- **AgentFlow**：安卓 App ↔ 自部署 Go Relay ↔ 桌面 agent ↔ Claude Code/Codex CLI，端到端加密，按项目存历史。

### 其他
- **chroxy**：token 门禁 dashboard，三种 Cloudflare 隧道模式（Quick/Named/No）。
- **Wall-E（create-walle）**：浏览器"指挥中心"，多 agent 并排，Model Registry 含 DeepSeek。

### 排除（macOS 专属，WSL2 用不上）
- better-claude-rc、TermBridge、Cici、VibeKit。

### 共同注意
- 安全性全在自己：relay 别用陌生人的、令牌管好、agent 权限 = 钥匙。
- 国内访问官方 relay 有延迟问题，自建中转才舒服。

---

## 六、本机现状与待办

- ✅ Claude Code **已升级 2.1.220 → 2.1.233**（`claude update`），`claude self-hosted-runner` 子命令已存在。
- ⏳ 官方 Remote Control / self-hosted-runner 因 DeepSeek 网关无法使用（除非换订阅+直连）。
- ⏳ 候选学习路线：
  1. Paseo —— 综合体验最接近官方，多端，扫码配对
  2. VibeAround —— 中文 + API Bridge + DeepSeek 直连省心
  3. yepanywhere / remote-claude-code —— 轻量 npm 一条命令
  4. tmux + Tailscale + 手机 SSH —— 最稳最糙，模型无关兜底

---

## 参考来源
- Codex：GitHub openai/codex releases；Approvals 文档；Codex 避坑指南（腾讯云）
- Claude：claude.com/blog/run-claude-code-sessions-on-your-own-compute；code.claude.com/docs/en/remote-control
- 社区：getpaseo/paseo、codepick Paseo 介绍、AgentFlow、yepanywhere、clauster、telecode、remote-claude-code、hapi、VibeAround（V2EX）
