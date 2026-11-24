# Hermes Agent × Gemini 4 Gateway Skill (`hermes-gemini4-gateway`)

> 让大多数人**无需停留在 Antigravity IDE / 桌面客户端**，即可在 **Hermes Agent**（以及 Orca、Claude Code、OpenCode 等任意兼容 OpenAI/Anthropic 协议的 Agent）中稳定使用 **Gemini 4**（Neon / Carbon / Argon / Hydrogen）及 `agy` 开放的 Claude Opus 5.5 / Sonnet 5.5 模型。
>
> *A ready-to-install Agent Skill that walks your AI agent (or you) through bridging Gemini 4 into Hermes Agent via Magpie local gateway + official `agy` CLI (`agy acp`), complete with real-world troubleshooting.*

---

## 工作原理

```text
Hermes Agent / Orca / Claude Code / OpenCode
        │  OpenAI / Anthropic API (http://localhost:3425/v1)
        ▼
  Magpie 本地网关 (https://usemagpie.ai)
        │  社区插件 @magpie-community/opencode-agy-cli-auth
        ▼
  本机官方 agy CLI (`agy acp` 进程池 + MCP 工具桥接)
        │
        ▼
  Google Antigravity (Gemini 4 Neon / Carbon / Argon / Hydrogen ...)
```

**为什么选这条链路？**
1. **不依赖被封禁/停用的旧方案**：旧版 `gemini-cli` 个人号 OAuth 已被 Google 停用；旧版直连逆向 `antigravity` 插件因封号风险已被社区下架。本方案由官方 `agy` CLI 通过标准 ACP 协议发请求，更稳妥。
2. **完整支持 Agent 工具调用与多轮会话**：插件自动维护 `agy` 进程池并把 Hermes 的文件读写、终端、搜索等工具经由 MCP 桥接给模型，不是简单的纯文本壳。
3. **一次部署，所有 Agent 通用**：只要 Magpie 本地网关跑起来，Hermes Agent、Orca、Claude Code、Codex、OpenCode 都能一键切到 Gemini 4，还支持多 Google 账号配额轮换。
4. **浓缩实战踩坑**：收录了实测中最高频的坑（本地回环被代理劫持导致 502、出口 IP 地区限制与 Clash 钉节点、旧教程虚构的 `agy auth login` 命令、3430 面板端口冲突等）。

---

## 一键安装本 Skill

### 方式 1：Hermes Agent 一键安装（推荐，自动拉取 `SKILL.md` + 全部 `references/`）

```bash
hermes skills install wuyanwei0521/hermes-gemini4-skill/
```

> 注意末尾的 `/` 不能省略（Hermes 用 `owner/repo/` 表示根目录单 skill 仓库，会一并拉取 `references/` 排障与进阶文档）。

或仅安装单文件 `SKILL.md`（不含 `references/`）：

```bash
hermes skills install https://raw.githubusercontent.com/wuyanwei0521/hermes-gemini4-skill/main/SKILL.md
```

安装完成后，在 Hermes 里直接说：
> 「帮我按 `hermes-gemini4-gateway` skill 部署 Gemini 4，并把我的 Hermes 切过去。」

### 方式 2：手动安装（适用于任意支持 `SKILL.md` 的 Agent）

从本仓库页面点击 **Code → Download ZIP** 解压，将文件夹放置到你的 Agent skills 目录（例如 `~/.hermes/skills/hermes-gemini4-gateway`）即可。

---

## 快速预览（5 步跑通）

完整细节、跨平台安装方式（macOS / Linux / Windows）、每步验证命令和排障方案都在 [`SKILL.md`](./SKILL.md) 与 `references/` 中，核心流程如下：

1. **安装官方 `agy` CLI（≥ 1.5.1）**（macOS 可直接 `brew install --cask antigravity-cli`，其余平台见 [官方文档](https://antigravity.google/docs/cli/install/)；或让 Magpie ≥ 0.1.4246 登录时自动托管安装）。
2. **安装并启动 Magpie**（见 [Magpie Releases](https://github.com/yetone/magpie-releases/releases/latest) 或 [usemagpie.ai](https://usemagpie.ai/)）。
3. **安装 `agy-cli` 插件并登录 Google 账号**：
   ```bash
   magpie plugin add @magpie-community/opencode-agy-cli-auth
   magpie plugin login agy-cli
   ```
4. **配置本地回环免代理（防止 502）并验证出口连通性**：
   ```bash
   export NO_PROXY="127.0.0.1,localhost,::1" no_proxy="127.0.0.1,localhost,::1"
   timeout 60 agy -p "Reply with exactly: OK"
   ```
5. **一键将 Hermes 切换到 Gemini 4**：
   ```bash
   magpie hermes auto/gemini-4-neon
   hermes chat -q "只回复两个字：通了"
   ```

---

## 目录结构

```text
.
├── SKILL.md                                # Skill 主体（供 Hermes / Agent 直接读取执行）
├── README.md                               # 项目说明与安装指引
├── LICENSE                                 # MIT License
└── references/
    ├── troubleshooting.md                  # 实战排障全集（502 / 400 地区限制 / Clash 钉节点 / 端口冲突等）
    └── orca-and-multi-account.md           # 多账号轮换、Orca 集成、其他 Agent 复用与官方 API Key 替代方案
```

---

## 注意事项与免责声明

- 本 Skill 仅提供开源工具（[Hermes Agent](https://hermes-agent.nousresearch.com/)、[Magpie](https://usemagpie.ai/)、[@magpie-community/opencode-agy-cli-auth](https://github.com/magpie-community/plugins/tree/main/packages/agy-cli)）与官方 [Antigravity CLI](https://antigravity.google/docs/cli/install/) 的本地配置与排障流程。
- 账号配额、地区可用性与服务政策由 Google 决定，请遵守相应服务条款。
- 如果你拥有付费 Google AI Studio / Vertex AI API Key，推荐直接使用官方 API Key 接入（见 [`references/orca-and-multi-account.md`](./references/orca-and-multi-account.md)），无需本地 CLI 桥接。
