---
name: hermes-gemini4-gateway
description: "Run Gemini 4 in Hermes Agent via Magpie local gateway."
version: 1.0.0
author: RainCrash (wuyanwei0521)
license: MIT
platforms: [macos, linux, windows]
metadata:
  hermes:
    tags: [hermes, gemini-4, magpie, agy-cli, gateway, proxy, orca]
    category: productivity
---

# Hermes Agent 接入 Gemini 4 完整部署指南（Magpie + agy-cli）

## 适用场景

当用户希望在 **Hermes Agent**（以及 Orca、Claude Code、OpenCode 等任意支持 OpenAI/Anthropic 协议的 Agent）中直接使用 **Gemini 4**（Neon / Carbon / Argon / Hydrogen）及 `agy` 开放的其他模型（如 Claude Opus 5.5 / Sonnet 5.5），而**无需停留在 Antigravity IDE / 桌面客户端里工作**，或需要排查该链路故障时加载本 skill。

### 架构与原理

```text
Hermes Agent (OpenAI/Anthropic 协议)
  │  http://localhost:3425/v1  (需 NO_PROXY 绕过本地代理)
  ▼
Magpie 本地网关 (端口 3425，Web/桌面管理面板 3430)
  │  社区插件 @magpie-community/opencode-agy-cli-auth (provider id: agy-cli)
  ▼
本机官方 agy CLI (`agy acp` 进程池 + MCP 工具桥接)
  │  走你的出口网络/代理（需 Google 支持地区 + 干净出口 IP）
  ▼
Google Antigravity 后端 (Gemini 4 Neon/Carbon/Argon/Hydrogen 等)
```

**为什么必须走 `Magpie + agy-cli`，而不是旧方案？**
- **旧版 Gemini CLI 个人号 OAuth**：已被 Google 停用（返回 `This client is no longer supported for Gemini Code Assist for individuals… migrate to Antigravity`）。
- **旧版直连逆向插件（`antigravity`）**：因绕过官方客户端直调接口容易触发封号，社区已下架新用户安装入口。
- **当前稳妥方案（`agy-cli`）**：请求由本机**官方 `agy` CLI 通过标准 ACP 协议（`agy acp`）**发出，Magpie 负责将 OpenAI/Anthropic 格式翻译为 ACP、复用会话进程池，并把 Hermes 的工具通过 MCP 桥接给模型。
- *说明：如果你拥有付费 Google AI Studio / Vertex API Key，可直接跳过 `agy-cli`，在 Magpie Providers 页添加 Google Gemini API Key 即可（见 `references/orca-and-multi-account.md`）。*

> **合规提醒**：本方案通过官方 `agy` CLI 的本地 ACP 协议完成桥接，但仍受 Google 账号服务条款与地区政策约束，请遵守相关使用条款。

---

## 前置检查

开始前确认目标机器满足以下条件：
1. **已安装 Hermes Agent**（`hermes --version` 可用）。
2. **网络可访问 Google 支持地区**：Antigravity 按出口 IP 严格判区，**绝大多数机房 IP 和非支持地区节点会报 `400 FAILED_PRECONDITION`**，通常需要干净的支持地区家宽/原生出口。
3. **拥有可用的 Google 账号**（已开通/可使用 Antigravity）。

---

## 部署步骤（按顺序执行，每步带完成判据）

### 第 1 步：安装官方 Antigravity CLI（`agy` ≥ 1.5.1）

> `agy` 1.5.1+ 才在 ACP 通道中完整开放 Gemini 4 及非 Gemini 模型，且支持 `agy acp --login` 与图片输入。
> **免手装提示**：Magpie 0.1.4246+ 在检测到本机没有 `agy` 时，也会在登录阶段自动从官方地址下载并校验 SHA-512 托管版 `agy`；若你希望手动安装以便排查网络与单独运行 `agy`，按以下方式安装：

- **macOS（推荐 Homebrew，自动进 PATH 且无需提权）**：
  ```bash
  brew install --cask antigravity-cli
  ```
- **macOS / Linux（官方安装脚本，先下载检查再运行）**：
  访问官方文档 `https://antigravity.google/docs/cli/install/`，或下载官方脚本审查后执行：
  ```bash
  curl -fsSL https://antigravity.google/cli/install.sh -o agy-install.sh
  less agy-install.sh   # 确认内容无误后再执行
  bash agy-install.sh
  rm -f agy-install.sh
  ```
  默认安装到 `$HOME/.local/bin/agy`，请确保 `$HOME/.local/bin` 已在 `PATH` 中。
- **Windows（PowerShell）**：
  从官方页面 `https://antigravity.google/docs/cli/install/` 下载 `install.ps1` 检查后运行（默认安装至 `%LOCALAPPDATA%\agy\bin`）。

**完成判据：**
```bash
agy --version
# 预期输出 >= 1.5.1（例如 1.5.16）
```

---

### 第 2 步：安装并启动 Magpie 本地网关

- **macOS / Windows / Linux 桌面或二进制包（推荐，附官方 SHA256SUMS 校验）**：
  直接从官方发布页下载对应平台安装包：
  `https://github.com/yetone/magpie-releases/releases/latest`
  或访问官网：`https://usemagpie.ai/`
- **macOS / Linux 命令行安装脚本（先下载检查再运行，脚本内置 SHA-256 校验）**：
  ```bash
  curl -fsSL https://usemagpie.ai/install.sh -o magpie-install.sh
  less magpie-install.sh   # 确认内容无误后再执行
  sh magpie-install.sh
  rm -f magpie-install.sh
  ```
  - 需要走代理下载时可传参数：`sh magpie-install.sh --proxy http://localhost:<代理端口>`。
  - 无桌面环境的 Linux / WSL / SSH 服务器可传 `--tui` 安装终端与 Web 版。

**启动 Magpie：**
- **桌面环境（macOS / Linux GUI / Windows）**：打开 Magpie 应用（或运行 `magpie` / `magpie tray`），自动拉起本地网关 `localhost:3425` 与管理界面 `localhost:3430`。
- **无桌面环境（Linux 服务器 / WSL / SSH）**：
  ```bash
  MAGPIE_WEB_KEY="<至少16位随机字符>" magpie web --no-open --addr localhost:3430
  ```

**完成判据：**
```bash
magpie --version
curl -s --noproxy '*' -o /dev/null -w "%{http_code}\n" http://localhost:3425/v1/models
# 预期输出 200
```

---

### 第 3 步：安装 `agy-cli` 桥接插件并登录 Google 账号

> **不要运行虚构的 `agy auth login` 命令**（`agy` 1.5.x 中不存在该子命令）。登录和加号统一走 Magpie 插件流程。

1. **安装社区插件**：
   ```bash
   magpie plugin add @magpie-community/opencode-agy-cli-auth
   ```
   （也可在 Magpie 桌面/Web 面板 → **Plugins** 页搜索安装 `Antigravity (local agy CLI)`。）

2. **登录账号**：
   ```bash
   magpie plugin login agy-cli
   ```
   - 终端交互选项：
     - 选 **1**：复用本机 `agy` 已有的 Google 登录态（若之前运行过 `agy` 登录）。
     - 选 **2**：打开浏览器授权新 Google 账号（或在 SSH 环境下复制终端打印的授权链接到本地浏览器完成登录，再把回调授权码/完整回调 URL 粘贴回终端）。
   - 如需多号轮换提高配额，重复执行 `magpie plugin login agy-cli` 选 2 或在面板 **Providers → agy-cli → Add Account** 添加多个账号（详见 `references/orca-and-multi-account.md`）。

3. **验证出口 IP 与 `agy` 连通性（必跑！）**：
   ```bash
   # 若通过本地代理上网，替换为你的代理端口（如 7897 / 7890）；若直连或已开全局 TUN 则去掉前缀
   HTTPS_PROXY=http://localhost:<代理端口> timeout 60 agy -p "Reply with exactly: OK"
   ```
   - 若输出 `OK` → 出口 IP 和登录态均正常。
   - 若报 `400 FAILED_PRECONDITION: User location is not supported for the API use` → **当前出口 IP 不被支持**，立即切到支持地区的干净家宽节点并重测（排查方法见 `references/troubleshooting.md`）。

**完成判据：**
```bash
magpie models hermes | grep -E "agy-cli/|auto/gemini-4"
# 预期列出 agy-cli/gemini-4-neon、agy-cli/gemini-4-carbon 等模型
```

---

### 第 4 步：验证 Magpie 网关端到端可用

> **核心避坑**：如果终端里导出过 `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY`，访问本地回环 `localhost:3425` 时**必须**绕过代理，否则代理软件会直接返回 `502 Bad Gateway`！

```bash
curl -s --noproxy '*' http://localhost:3425/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "agy-cli/gemini-4-neon",
    "messages": [{"role": "user", "content": "Reply with OK only."}]
  }'
```

**完成判据：** 返回包含 `"OK"` 的正常 JSON 响应，且 `~/.config/magpie/routing/<YYYY-MM-DD>.jsonl` 中出现 `status: 200` 记录。

---

### 第 5 步：将 Gemini 4 接入 Hermes Agent

首先在当前终端（并建议在用户的 shell 环境变量配置中）设置本地回环免代理：
```bash
export NO_PROXY="127.0.0.1,localhost,::1"
export no_proxy="127.0.0.1,localhost,::1"
```

接入 Hermes Agent 有 **三种方式**，推荐方式 A（最简单、自动托管）：

#### 方式 A：使用 Magpie 一键接管 Hermes（推荐）
Magpie 原生支持直接管理 Hermes 的模型切换：
```bash
# 设为默认模型（推荐用路由组 auto/gemini-4-neon，自动在可用通道间容错）
magpie hermes auto/gemini-4-neon

# 或直接指定 agy-cli 下的具体模型
magpie hermes agy-cli/gemini-4-neon
# 其他可选：
# magpie hermes agy-cli/gemini-4-carbon
# magpie hermes agy-cli/gemini-4-hydrogen
# magpie hermes agy-cli/gemini-4-argon
# magpie hermes agy-cli/claude-opus-5-5

# 调整推理档位（支持 low / medium / high / xhigh）
magpie hermes effort high

# 随时还原回 Hermes 原本的模型配置
# magpie hermes default
```
也可以在 Magpie 桌面/Web 面板 → **Agents → Hermes Agent** 一行直接点选切换。

#### 方式 B：在 Hermes 模型别名中按需调用（不改默认模型）
如果你希望保留原来的默认模型，只在需要时切到 Gemini 4，可在 Hermes 配置的 `model.aliases` 段中加入别名：
```yaml
model:
  aliases:
    gemini4:
      provider: custom
      base_url: http://localhost:3425/v1
      model: agy-cli/gemini-4-neon
```
使用时：
- 命令行单次调用：`hermes chat -q "你好，报一下模型身份" --model gemini4`
- 会话内即时切换：输入 `/model gemini4`（或 `/model auto/gemini-4-neon`）

#### 方式 C：在 Orca 等宿主工具中的 Hermes Agent 使用
Orca 启动 Hermes 时不支持 `--model` 参数，它直接读取 Hermes 当前默认模型配置。用方式 A 执行 `magpie hermes auto/gemini-4-neon` 后，**重启 Orca 中已打开的 Hermes 会话**即可生效（详见 `references/orca-and-multi-account.md`）。

**完成判据（最终端到端验收）：**
```bash
export NO_PROXY="127.0.0.1,localhost,::1" no_proxy="127.0.0.1,localhost,::1"
hermes chat -q "只回复两个字：通了"
# 若仅配置了方式 B 别名，则加 --model gemini4
```
预期 Hermes 正常返回“通了”，且 Magpie 路由日志 `~/.config/magpie/routing/<YYYY-MM-DD>.jsonl` 最新一行显示 `agent: "hermes"`、`status: 200`。

---

## 常用运维与模型切换速查

| 需求 | 命令 |
|---|---|
| 查看当前 Hermes 接管的模型 | `magpie hermes` |
| 列出 Hermes 可选的全部模型 | `magpie models hermes` |
| 切换为 Gemini 4 Neon | `magpie hermes auto/gemini-4-neon` |
| 切换为 Claude Opus 5.5 | `magpie hermes agy-cli/claude-opus-5-5` |
| 调整思考/推理强度 | `magpie hermes effort high`（或 `low`/`medium`/`xhigh`） |
| 还原 Hermes 原配置 | `magpie hermes default` |
| 更新 `agy-cli` 插件 | `magpie plugin update` |
| 查看今天网关真实路由审计 | `tail -n 20 ~/.config/magpie/routing/<YYYY-MM-DD>.jsonl` |

---

## 深入参考文档

- **故障排查全集（502 回环劫持、400 地区限制与 Clash 钉节点、端口 3430 冲突、工具调用慢 2s、SSH 远程登录等）**：阅读 `references/troubleshooting.md`
- **多账号配额轮换、Orca 集成、其他 Agent 复用、以及官方 AI Studio API Key 替代方案**：阅读 `references/orca-and-multi-account.md`
