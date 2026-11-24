# 故障排查全集（Magpie + agy-cli → Hermes Agent）

按实际发生频率排序。排查任何“请求失败”前，先看 Magpie 的事实审计日志（将 `<YYYY-MM-DD>` 替换为当天日期）：
```bash
tail -n 20 ~/.config/magpie/routing/<YYYY-MM-DD>.jsonl
```
每条请求记录一行 JSON，包含 `agent`、`model`、`status`、`ms`、`tokens` 与 `tries[].error`。**以此判断“请求到底有没有打到 Magpie，以及是谁报的错”。**

---

## 1. Hermes 报 `502` / `Provider unavailable`，但 Magpie 正常

**现象**：`curl --noproxy '*' http://localhost:3425/v1/models` 返回 200，但 `hermes chat` 报 502 或不可用，且 `~/.config/magpie/routing/<YYYY-MM-DD>.jsonl` 里**完全没有这条请求**。

**根因**：当前终端或系统环境变量中设置了 `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY`，导致 Hermes 底层 HTTP 客户端把发往本地回环（`localhost` / `127.0.0.1` 端口 `3425`）的请求也塞进了代理软件（如 Clash/mihomo），代理无法回环转发便直接返回 502。

**修复**：
```bash
# 当前终端立即生效（大小写都必须设，不同库读取的变量名不同）
export NO_PROXY="127.0.0.1,localhost,::1"
export no_proxy="127.0.0.1,localhost,::1"
```
建议把上面两行手工添加到你的登录 shell 环境配置中，使其在所有新终端持久生效。如果是通过 `curl` 调试本地网关，务必加 `--noproxy '*'`。

---

## 2. 报 `400 FAILED_PRECONDITION: User location is not supported for the API use` 或 `EOF`

**现象**：
- `agy -p "Reply OK"` 或 Hermes 返回：`FAILED_PRECONDITION (code 400): User location is not supported for the API use.`
- 或 `Eligibility check failed: Post "https://daily-cloudcode-pa.googleapis.com/v1internal:fetchUserInfo": EOF`

**根因**：Google Antigravity 后端对出口 IP 的地理位置与 IP 类型（机房 vs 住宅/家宽）判定非常严格。
- 香港、澳门、中国大陆及部分受限地区节点会被直接拒绝。
- 即使是美国/日本/英国等支持地区，**普通机房 IP 或被标记的共享 IP 也经常触发 `FAILED_PRECONDITION` 或连接重置**。
- 实测中，通常只有**支持地区（如日本、美国等）的干净家宽/原生住宅出口**能够稳定通过。

**排查与永久固定方案**：

1. **逐个实测候选节点**（不要只看测速延迟，必须用 `agy -p` 实测）：
   ```bash
   HTTPS_PROXY=http://localhost:<代理端口> timeout 60 agy -p "Reply with exactly: OK"
   ```
   切换节点后重跑该命令，直到稳定打印 `OK`。

2. **在 Clash / mihomo 中把 Google/Antigravity 域名钉死到可用节点**（防止规则模式自动切走或 GUI 重载漂移）：
   在代理规则（`prepend-rules`）最前面加入：
   ```yaml
   prepend-rules:
     - DOMAIN-SUFFIX,googleapis.com,<你的可用家宽节点名>
     - DOMAIN-SUFFIX,google.com,<你的可用家宽节点名>
     - DOMAIN-SUFFIX,googleusercontent.com,<你的可用家宽节点名>
     - DOMAIN-SUFFIX,gstatic.com,<你的可用家宽节点名>
     - DOMAIN-SUFFIX,googlevideo.com,<你的可用家宽节点名>
     - DOMAIN-KEYWORD,antigravity,<你的可用家宽节点名>
   ```
   **Clash Verge 特别注意事项**：
   - 修改全局 `Merge.yaml` 可能不生效——需确认当前激活的远程订阅在 `profiles.yaml` 中绑定的专属 `merge` 文件（每个订阅有独立的 `option.merge` UID），写入对应文件并在 Clash Verge GUI 中重新点击一次该订阅卡片使其重新合成。
   - Clash Verge 在配置校验或重启后，可能会把 `GLOBAL` 节点重置回 GUI 上次持久化记住的节点（日志中表现为 `Selected node for proxy: GLOBAL`）。务必在 GUI 界面中手动选中该可用节点，或同时配合上述规则模式分流。

---

## 3. 旧教程里的命令报错：`agy auth login` / `agy auth status` 不存在

**现象**：运行 `agy auth login` 只打印 `Usage of agy:` 帮助信息。

**根因**：网上部分早期文章编造或沿用了其他 CLI 的子命令；官方 `agy` 1.5.x 并没有 `auth` 子命令。

**正确做法**：
- 登录/新增账号统一用：`magpie plugin login agy-cli`
- 或直接在终端执行一次 `agy` 触发其内置 Google 登录流程（SSH 环境下 `agy` 会自动检测无浏览器并打印授权链接，复制到本地浏览器授权后贴回验证码即可）。

---

## 4. `spawn agy ENOENT`（Magpie 找不到 `agy` 可执行文件）

**现象**：终端里能跑 `agy --version`，但通过桌面图标启动的 Magpie 报错找不到 `agy`。

**根因**：macOS / Linux 图形界面启动的应用通常不继承交互式终端自定义的 `PATH`，因此找不到 `$HOME/.local/bin/agy`。

**修复（任选其一，无需管理员提权）**：
1. **推荐（macOS）**：改用 Homebrew 安装 `brew install --cask antigravity-cli`，二进制会自动进入标准路径。
2. **推荐（全平台）**：升级到 Magpie ≥ 0.1.4246。若未找到本机 `agy`，在 Magpie 中点击 Sign in 会自动下载官方校验版到 `~/.local/share/magpie-agy-cli/agy`（Windows 为 `%LOCALAPPDATA%\magpie-agy-cli\agy.exe`）。
3. **环境变量方式**：在 Magpie 对应账号/插件配置中显式把 `$HOME/.local/bin` 加入 `PATH`。

---

## 5. `GEMINI_API_KEY environment variable is not set`

**现象**：调用 `agy-cli` 模型时提示缺少 `GEMINI_API_KEY`。

**根因**：之前在 `agy` 的 `settings.json` 里写死了 `"modelProvider": "gemini"`（AI Studio API Key 模式），覆盖了 Google 账号登录模式。

**修复**：
- 删除或清空 `settings.json` 中的 `"modelProvider"` 字段，让 `agy` 回到默认 Google 账号模式。
- 如果你确实想用 Google AI Studio API Key，请直接在 Magpie 的 **Providers** 页作为标准 Google Gemini Provider 添加，不要通过 `agy-cli` 插件中转。

---

## 6. Magpie 面板端口 `3430` 冲突（`bind: address already in use`）

**现象**：命令行运行 `magpie web` 报错 `address already in use`。

**根因**：Magpie 桌面版（菜单栏常驻进程）与命令行 `magpie web` 默认都会监听 `localhost:3430`。

**修复**：
- 桌面版已在运行时，直接点击菜单栏 Magpie 图标打开窗口即可，无需再开 `magpie web`。
- 若确实需要命令行 Web 面板：先停掉旧进程，或指定其他端口：
  ```bash
  # 查看占用 3430 的进程（macOS / Linux）
  lsof -iTCP:3430 -sTCP:LISTEN -P

  # 换端口启动（或停掉旧进程后启动）
  MAGPIE_WEB_KEY="<至少16位密钥>" magpie web --no-open --addr localhost:3431
  ```

---

## 7. 旧版 Gemini CLI 账号报错

**现象**：Magpie 面板里之前登录过的 `gemini`（Gemini CLI）账号全部报错 `This client is no longer supported for Gemini Code Assist for individuals... migrate to Antigravity`。

**原因与处理**：Google 已停用个人账号的旧版 Gemini CLI OAuth 通道。这些失效凭证可直接在 Magpie 面板删除，全部改用 `agy-cli` provider 即可。

---

## 8. 工具调用每步多耗 ~2 秒

**现象**：纯问答很快，但 Agent 连续调用工具（读写文件、跑命令）时，每个工具调用前都会多出约 1.3–2.6 秒延迟，且用量里会出现 0 token 的中间请求。

**原因**：这是官方 `agy` 自身对每个 MCP 桥接工具强制执行的串行安全检查（`sir_flash_lite`）。该行为位于 `agy` 内部，插件没有关闭开关，属于正常现象。

---

## 9. `agy` 版本过旧导致报错或模型不全

**说明**：`agy-cli` 插件启动子进程时默认携带 `AGY_CLI_DISABLE_AUTO_UPDATE=true`，以防止多进程并发自更新损坏二进制文件。

**维护建议**：每隔一段时间在终端手动运行一次 `agy`（或通过 Homebrew / 官方安装包升级）以更新到最新 `agy` 版本，同时运行 `magpie plugin update` 更新插件。

---

## 10. 完整健康检查脚本（一键定位在哪一层断了）

```bash
#!/usr/bin/env bash
set -u
export NO_PROXY="127.0.0.1,localhost,::1"
export no_proxy="127.0.0.1,localhost,::1"

echo "=== 1. 二进制版本 ==="
agy --version || echo "[FAIL] agy 未安装或不在 PATH"
magpie --version || echo "[FAIL] magpie 未安装或不在 PATH"

echo "=== 2. agy 出口直测（排除 Magpie/Hermes 干扰） ==="
timeout 60 agy -p "Reply with exactly: OK" || echo "[FAIL] agy 失败（检查登录态或代理出口 IP 地区）"

echo "=== 3. Magpie 网关在线与模型列表 ==="
curl -s --noproxy '*' http://localhost:3425/v1/models | grep -o '"agy-cli/[^"]*"' | head -10

echo "=== 4. Magpie 网关对话实测 ==="
curl -s --noproxy '*' http://localhost:3425/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"agy-cli/gemini-4-neon","messages":[{"role":"user","content":"Reply OK only."}]}' | head -c 400
echo ""

echo "=== 5. Hermes 当前绑定状态 ==="
magpie hermes
```
