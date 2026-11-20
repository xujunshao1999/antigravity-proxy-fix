# Antigravity 国内使用指南（Windows & Ubuntu/Linux）

> **适用版本**：Antigravity 2.33+（含 `Antigrative Dashboard` / `antigravity-pulse` 插件内嵌栏）
> **适用系统**：Windows 10 / 11 · Ubuntu / Linux
> **前置条件**：已有可用 Google 账号，以及支持 HTTP/HTTPS 的本地代理客户端（Clash Verge Rev、mihomo、v2rayN 等）

---

## 目录

1. [前置检查：确认代理端口](#一前置检查确认代理端口)
2. [Windows 安装与配置](#二windows-安装与配置)
3. [Ubuntu / Linux 安装与配置](#三ubuntu--linux-安装与配置)
4. [桌面版与 CLI 常用操作](#四桌面版与-cli-常用操作)
5. [代理模式选择：规则 vs 全局 vs TUN](#五代理模式选择规则-vs-全局-vs-tun)
6. [故障排查与一键诊断](#六故障排查与一键诊断)
7. [账号安全与节点稳定性](#七账号安全与节点稳定性)
8. [附录：文件与日志位置速查](#八附录文件与日志位置速查)

---

## 一、前置检查：确认代理端口

常见的代理客户端均提供混合代理端口（`mixed-port`，同时支持 HTTP 与 SOCKS5）：

| 客户端 | 常见默认端口 |
|---|---|
| Clash Verge Rev / mihomo | `7897` |
| Clash for Windows / 部分配置 | `7890` |
| v2rayN (HTTP) | `10809` |

快速确认当前监听端口：
- **Windows**：`netstat -ano | findstr LISTENING | findstr "127.0.0.1:"`
- **Linux**：`ss -ltnp | grep 127.0.0.1`
- 或直接运行本仓库的跨平台诊断脚本（详见第六章）。

---

## 二、Windows 安装与配置

> Windows 方案全部基于系统自带的 `.NET Framework 4 csc.exe` 编译，**无需安装 MinGW 或 Visual Studio**，且**绝不修改全局环境变量**。

### 2.1 桌面版（Antigravity）

在仓库根目录执行：

```powershell
# 1. 编译无窗口桌面启动器 AntigravityProxy.exe 到 %USERPROFILE%\.agy-shim\
#    并将你的代理端口写入 %USERPROFILE%\.agy-shim\proxy.conf（按实际端口修改）
powershell -ExecutionPolicy Bypass -File .\scripts\build-desktop-launcher.ps1 -ProxyUrl http://127.0.0.1:7890

# 2. 改写桌面、开始菜单、任务栏固定快捷方式指向 AntigravityProxy.exe
#    -AddDebugPort      : 保留/补齐 Dashboard 插件内嵌栏所需的 --remote-debugging-port 参数
#    -ProtectStartMenu  : 写回 Electron 的 AppUserModelID 并对开始菜单快捷方式加只读(+R)，
#                         防止 Electron 桌面通知/重启时静默还原快捷方式
powershell -ExecutionPolicy Bypass -File .\scripts\repoint-shortcut.ps1 -AddDebugPort -ProtectStartMenu
```

**安装开机静默自愈**（应对大版本自动更新重建快捷方式）：

```powershell
Copy-Item -Force .\scripts\self-heal-shortcuts.ps1 "$env:USERPROFILE\.agy-shim\"
Copy-Item -Force .\scripts\self-heal-shortcuts.vbs "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\"
```

**修改代理端口**：直接编辑 `%USERPROFILE%\.agy-shim\proxy.conf`（写 `http://127.0.0.1:7897` 或只写数字 `7897`），**无需重新编译**。

### 2.2 命令行版（`agy`）

```powershell
# 编译零依赖 CLI 转发器到 %LOCALAPPDATA%\agy\shim\agy.exe，并安全插入用户 PATH 最前（保持 ExpandString）
powershell -ExecutionPolicy Bypass -File .\scripts\build-cli-shim.ps1 -AddToPath
```

完成后**关闭并重开终端**使 `PATH` 生效。可通过交互式管理脚本随时启用/停用或查看状态：

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\agy-proxy-toggle.ps1
```

### 2.3 验证（Windows）

```powershell
# 全面检查启动器、快捷方式、自愈脚本、端口探测、全局环境变量洁净度与 CLI shim
powershell -ExecutionPolicy Bypass -File .\scripts\verify-review.ps1

# 命令行验证（如已安装 agy shim）
(Get-Command agy).Source        # 应指向 ...\agy\shim\agy.exe
agy --version                   # 应输出 agy-shim 2.1.0 (cs)
agy -p "reply OK"               # 应返回 OK
```

配置好后请**彻底退出并重新启动** Antigravity（确保托盘无残留进程），即可同时走代理并显示 Dashboard 内嵌栏。

### 2.4 一键卸载（Windows）

如需完整回退 Windows 代理配置（还原桌面/开始菜单/任务栏快捷方式并解除开始菜单 `+R`、清理 `PATH` 中的 shim 路径、删除自愈启动项与已部署二进制，保留原版 Antigravity 及命令行参数）：

```powershell
# 预览操作（不实际修改任何文件或注册表）
powershell -ExecutionPolicy Bypass -File .\scripts\uninstall-windows.ps1 -DryRun

# 一键卸载（默认保留 %USERPROFILE%\.agy-shim\proxy.conf）
powershell -ExecutionPolicy Bypass -File .\scripts\uninstall-windows.ps1

# 彻底清空（连同 proxy.conf 与空目录一起删除）
powershell -ExecutionPolicy Bypass -File .\scripts\uninstall-windows.ps1 -Purge
```

---

## 三、Ubuntu / Linux 安装与配置

Linux 下无需编译，采用统一的 multi-call Bash wrapper（[linux/proxy-wrapper.sh](../linux/proxy-wrapper.sh)）通过 `exec` 进程替换直接注入代理并透传全部参数与信号；桌面端通过修改用户级 XDG 文件 `~/.local/share/applications/antigravity.desktop` 覆盖系统项（`apt`/客户端更新不会破坏用户目录）。

```bash
chmod +x linux/*.sh

# 安装 ~/.local/bin/agy 与 ~/.local/bin/antigravity-proxy，写入 ~/.config/agy-proxy/proxy.conf
# 若本机检测到 antigravity-pulse 插件，会自动为 .desktop 补齐本地调试端口参数
./linux/install-linux.sh --proxy http://127.0.0.1:7890

# 如需显式开启或关闭调试参数，可传 --with-debug-port 或 --no-debug-port
# ./linux/install-linux.sh --proxy 7897 --with-debug-port
```

确保 `~/.local/bin` 排在 `PATH` 最前面（写入 `~/.bashrc` 或 `~/.zshrc`）：

```bash
export PATH="$HOME/.local/bin:$PATH"
```

状态查询与一键卸载：

```bash
./linux/install-linux.sh --status
./linux/install-linux.sh --uninstall
```

---

## 四、桌面版与 CLI 常用操作

- **账号共用**：桌面版与 `agy` CLI 共用同一套系统凭据；在任一端登出重登，另一端同步生效。
- **查看实时可用模型**：运行 `agy models`（不要依赖过时的硬编码模型名）。部分模型名已包含推理强度后缀时，再传 `--effort` 会报参数冲突，请以 `agy models` 输出为准。
- **常用 CLI 命令与参数**：

| 用法 | 说明 |
|---|---|
| `agy` | 进入当前目录交互式 TUI 会话 |
| `agy -p "问题"` | 非交互单次提问 |
| `agy -p "任务" --model <id>` | 指定模型 |
| `agy --continue` | 继续上次会话 |
| `agy --add-dir "<path>"` | 附加额外工作目录 |
| `agy --mode plan` | 只读规划模式（`plan` / `accept-edits` / `auto`） |
| `agy --output-format stream-json` | 流式 JSON 输出，适合脚本调用 |
| `agy plugins` / `agy mcp list` | 查看已安装插件 / MCP 服务器 |

- **常用斜杠命令**：`/help`、`/clear`、`/compact`、`/model`、`/models`、`/cost`、`/login`、`/logout`、`/resume`、`/plugin`、`/skills`、`/diff`、`/status`。
- **CLI 配置文件位置**：`~/.gemini/antigravity-cli/settings.json`（可配置 `colorScheme`、`defaultModel`、`trustedWorkspaces`、`vimMode` 等）。

---

## 五、代理模式选择：规则 vs 全局 vs TUN

### 5.1 推荐选择：规则模式（Rule）

进程级注入只保证「Antigravity / agy 流量进入本地代理端口」，如何出站由代理客户端决定。日常使用**强烈推荐规则模式**：
- 国内站点与本地服务不绕路境外，避免国内服务因境外出口 IP 触发风控或速度变慢；
- 确保规则集中包含 Google 匹配规则与兜底规则，例如：
  ```yaml
  - DOMAIN-KEYWORD,google,<你的节点组>
  - GEOIP,CN,DIRECT
  - MATCH,<你的节点组>
  ```

### 5.2 常见的规则顺序陷阱

部分机场订阅在 `DOMAIN-KEYWORD,google` 之前会附带一条：

```yaml
- DOMAIN,safebrowsing.googleapis.com,DIRECT
```

该条规则优先命中时，安全浏览接口会直连失败，但不影响核心模型与对话。如希望彻底干净可在自定义规则覆写中去掉该 `DIRECT` 规则。

### 5.3 何时考虑 TUN 模式

仅当你希望系统内所有命令行/软件全自动走代理、不介意安装虚拟网卡驱动、且机器上**没有**与虚拟网卡冲突的游戏反作弊程序时，才需考虑 TUN 模式。否则继续使用本仓库的进程级注入方案是最安全、最小侵入的选择。

---

## 六、故障排查与一键诊断

### 6.1 一键诊断脚本（推荐第一步）

当你遇到**规则模式可用但全局模式不行**、**模型列表拉取超时**、或者不确定是**代理注入失效**还是**节点本身故障**时，直接运行：

```powershell
# Windows
powershell -ExecutionPolicy Bypass -File .\scripts\diagnose-mode.ps1

# Windows / Ubuntu / Linux 跨平台
python3 .\scripts\diagnose-mode.py
```

脚本会自动完成四项检查并给出明确结论：
1. 读取 `proxy.conf` 与常见端口并测 TCP 延迟；
2. 通过 Clash / mihomo 接口（Windows 命名管道 / Linux Unix socket 或 HTTP controller）列出当前模式和策略组节点，自动标出 `DIRECT`、`REJECT`、流量剩余提示假节点以及会跳 IP 的 `URLTest` 组；
3. 对 `cloudcode-pa.googleapis.com`、`daily-cloudcode-pa.googleapis.com`、`oauth2.googleapis.com`、`accounts.google.com`、`www.google.com` 逐一探测 HTTP 状态码与 TLS 握手耗时；
4. 区分判定：
   - **全部 `000`**：代理未开、配置失效、或节点彻底不可用（重点检查 `GLOBAL` 是否停在 `DIRECT` / 假节点上）；
   - **仅 `cloudcode-pa` 系列 `000`（`www.google.com` 正常）**：你的代理注入**正常**，是当前所选节点到 Google AI 上游链路故障/限流（俗称“半死节点”），**在代理客户端换一个节点即可，不要改配置文件**；
   - **TLS > 1.0s**：可用但节点延迟偏高，可切换延迟更低的节点。

### 6.2 常见典型问题

#### 1. Dashboard 插件有侧边栏、没有聊天框内嵌栏
- 确认是通过改写后的桌面/开始菜单快捷方式（或 Linux 用户级 `.desktop`）启动的，且快捷方式参数包含 `--remote-debugging-port=0 --remote-debugging-address=127.0.0.1`；
- Windows 下运行 `.\scripts\verify-review.ps1` 检查两条快捷方式是否均为 `OK`；若开始菜单被还原，运行：
  ```powershell
  powershell -ExecutionPolicy Bypass -File .\scripts\repoint-shortcut.ps1 -AddDebugPort -ProtectStartMenu
  ```
- 修改后需**完全退出并重开** Antigravity。

#### 2. 桌面版一直白屏 / 转圈
查看内核日志 `%APPDATA%\Antigravity\logs\language_server.log`（Linux 下在 `~/.config/Antigravity/logs/language_server.log`）：
- 搜索 `dial tcp`：若出现 `i/o timeout`，说明进程未经过代理启动（检查是否从旧固定图标启动，以及本地代理软件是否运行）；
- 搜索 `connection refused`：说明代理端口未监听或 `proxy.conf` 端口号与客户端不一致。

#### 3. 提示 `"Sign-in update needed"` 或日志出现 `ACCESS_TOKEN_SCOPE_INSUFFICIENT` (403)
说明本地保存的旧版 OAuth token 缺少新版要求的权限范围。在 `agy` 中运行 `/logout` 后重新登录（或在桌面版退出重登）即可同步修复两端。

#### 4. PowerShell 报错「在此系统上禁止运行脚本」
```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

---

## 七、账号安全与节点稳定性

账号风控最核心的触发因素不是“使用了代理”，而是 **IP 短时间跨地区剧烈跳变**、**机房/高滥用共享 IP**、以及**多账号频繁切换**。

### 自检出口 IP 质量与稳定性

```powershell
# 检查代理出口 IP 归属与是否机房 IP（期望 proxy=false, hosting=false）
curl.exe -x http://127.0.0.1:7890 -sS "http://ip-api.com/json/?fields=query,country,isp,proxy,hosting"

# 连续采样 5 次，确认 IP 固定不跳变
for ($i=1; $i -le 5; $i++) { curl.exe -x http://127.0.0.1:7890 -sS "http://ip-api.com/line/?fields=query"; Start-Sleep 5 }
```

**最佳实践**：
- 为 Google / Antigravity 策略组**手动固定单个稳定节点（`Selector`）**，避免使用会自动测速切换出口 IP 的 `URLTest` 组；
- 尽量保持长期使用同一区域/节点，避免多客户端同时高并发刷接口。

---

## 八、附录：文件与日志位置速查

### Windows 关键路径

| 用途 | 路径 |
|---|---|
| 桌面代理启动器 & 配置 | `%USERPROFILE%\.agy-shim\AntigravityProxy.exe`、`proxy.conf` |
| 自愈脚本 & 日志 | `%USERPROFILE%\.agy-shim\self-heal-shortcuts.ps1`、`self-heal.log` |
| 开机自愈入口 | `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\self-heal-shortcuts.vbs` |
| CLI 转发器 | `%LOCALAPPDATA%\agy\shim\agy.exe` |
| 桌面版原始程序 | `%LOCALAPPDATA%\Programs\antigravity\Antigravity.exe` |
| CLI 原始程序 | `%LOCALAPPDATA%\agy\bin\agy.exe` |
| Electron 外壳日志 | `%APPDATA%\Antigravity\logs\main.log` |
| Go 内核日志 | `%APPDATA%\Antigravity\logs\language_server.log` |

### Ubuntu / Linux 关键路径

| 用途 | 路径 |
|---|---|
| CLI & 桌面版包装器 | `~/.local/bin/agy`、`~/.local/bin/antigravity-proxy` |
| 代理地址配置 | `~/.config/agy-proxy/proxy.conf` |
| 用户级桌面快捷入口 | `~/.local/share/applications/antigravity.desktop` |
| 日志与用户配置 | `~/.config/Antigravity/logs/`、`~/.gemini/antigravity-cli/settings.json` |
