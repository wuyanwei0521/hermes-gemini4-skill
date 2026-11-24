# 进阶用法：多账号轮换、Orca 集成、其他 Agent 复用与官方 API 替代方案

## 1. 多 Google 账号配额轮换池

单账号额度紧张时，可以直接在 Magpie 中挂载多个 Google 账号，由 `agy-cli` 插件自动隔离登录态并按配额轮换（**无需手动写复杂的 `HOME` 目录脚本**）：

### 命令行方式
```bash
# 重复执行，交互时选择 2（Add another Google account）
magpie plugin login agy-cli
```

### 图形界面 / Web 面板方式
1. 打开 Magpie 面板 → **Providers** → **Antigravity (local agy CLI)**。
2. 点击 **Add Account**，在弹出的浏览器窗口中登录新的 Google 账号（建议在隐身窗口或不同浏览器 Profile 中操作，避免串号）。
3. 重复以上步骤添加多个账号，Magpie 会在配额耗尽或限流时自动切换账号，并可在面板查看各账号实时额度。

### 路由组与 Fallback（推荐）
- 优先让 Hermes 使用路由组 `auto/gemini-4-neon`（而不是硬编码单一 provider），当某个通道额度耗尽或异常时可自动回退到备用通道。
- 也可在 Magpie 中配置备用模型回退链：
  ```bash
  magpie provider fallback agy-cli <备用provider>/<备用模型>
  ```

---

## 2. 在 Orca 中让 Hermes Agent 使用 Gemini 4

**重要事实**：Orca 启动 Hermes 时**不支持**命令行 `--model` 参数（`orca agent launch --agent hermes --model ...` 会报错 `Agent hermes does not support launch-time model selection`）。Orca 内的 Hermes 直接读取 Hermes 当前默认模型配置。

因此在 Orca 中切换 Hermes 模型有三种正确方式：
1. **用 Magpie 切换默认模型（最省心）**：
   ```bash
   magpie hermes auto/gemini-4-neon
   # 切完后，重启 Orca 中已打开的 Hermes 会话即可生效
   ```
2. **在已打开的 Hermes 对话内切换**：
   直接在聊天框输入：
   ```text
   /model auto/gemini-4-neon
   ```
   或输入 `/model` 查看当前可用列表。
3. **用别名切换**：
   若在 Hermes 模型别名中配置了 `gemini4`，会话内直接发 `/model gemini4`。

---

## 3. 同一套 Magpie 网关复用到其他 Agent

部署完成后，Magpie 在 `http://localhost:3425/v1` 同时暴露 OpenAI 与 Anthropic 兼容端点，其他工具无需重复折腾：

- **Claude Code**：`magpie claude agy-cli/gemini-4-neon`（或 `magpie claude auto/gemini-4-neon`）
- **OpenCode**：`magpie opencode agy-cli/gemini-4-neon`
- **Codex CLI**：`magpie codex agy-cli/gemini-4-neon`
- **查看全部已检测到的本地 Agent**：`magpie ls`
- **任意自定义脚本 / Python OpenAI SDK**：
  ```python
  import httpx
  from openai import OpenAI

  # trust_env=False 确保本地回环请求不走系统代理，避免 502
  client = OpenAI(
      base_url="http://localhost:3425/v1",
      api_key="magpie-local",  # 本机访问默认免密；跨机访问需在 Magpie 网关页创建 Gateway Key
      http_client=httpx.Client(trust_env=False),
  )
  resp = client.chat.completions.create(
      model="agy-cli/gemini-4-neon",
      messages=[{"role": "user", "content": "Hello!"}],
  )
  print(resp.choices[0].message.content)
  ```

> **跨机 / 局域网访问安全提示**：Magpie 默认仅对本机回环免密。如需通过 SSH 隧道、Tailscale 或局域网让其他机器访问，请务必在 Magpie 面板的 **Gateway** 页创建 **Gateway Key**，并在客户端使用 `Authorization: Bearer <Gateway Key>`。

---

## 4. 替代方案：如果你有官方 Google AI Studio / Vertex API Key

如果你已购买或拥有官方 Google Gemini API Key，且不想受限于 `agy` 进程池延迟或家宽出口 IP：

1. **通过 Magpie 统一管理**：
   ```bash
   magpie provider add gemini <YOUR_GOOGLE_API_KEY>
   magpie models hermes
   magpie hermes gemini/<模型名>
   ```
2. **或直接在 Hermes 中配置**：按 Hermes 官方文档配置 `GEMINI_API_KEY` 即可直连，无需 `agy` 桥接。
