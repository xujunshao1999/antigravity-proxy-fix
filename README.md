# Antigravity Proxy Fix

> 让 Google Antigravity 桌面版和 Antigravity CLI (`agy`) 在仅开启系统代理的环境下正常工作，支持插件内嵌栏且不污染全局环境。

[![Platform](https://img.shields.io/badge/platform-Windows%2010%2F11%20%7C%20Ubuntu%20%2F%20Linux-blue)]()
[![Tested](https://img.shields.io/badge/tested-Antigravity%202.33%2B-green)]()
[![License](https://img.shields.io/badge/license-MIT-lightgrey)]()

---

## 问题是什么

Antigravity 和 `agy` CLI 在仅开启代理客户端「系统代理」时无法工作：

- **CLI 报错**：`token exchange failed: Post "https://oauth2.googleapis.com/token": dial tcp ...`
- **桌面版现象**：白屏 / 转圈，外壳日志报 `Failed to load URL: https://127.0.0.1:.../ with error: ERR_TIMED_OUT`

**根本原因**：Antigravity 是双进程架构——Electron 外壳认系统代理，但 Go 编写的内核（`language_server` 与 `agy`）基于 `http.ProxyFromEnvironment`，**只认环境变量，不读 Windows 系统代理或桌面环境图形代理**。只开系统代理时，内核在直连导致超时卡死。

---

## 解决方案

**只对目标进程树注入 `HTTP_PROXY` / `HTTPS_PROXY` 与 `NO_PROXY=localhost,127.0.0.1,::1`，不污染全局环境变量、不强依赖 TUN 虚拟网卡。**

| 方案 | 侵入性 | 适用范围 |
|---|---|---|
| **进程级注入（本仓库）** | 低，仅目标进程树 | ✅ 推荐 |
| 全局环境变量 `setx` | 高，全系统 | ❌ 易致本机回环 `502`、代理关闭即全系统断网 |
| TUN 模式 | 中，需虚拟网卡驱动 | ⚠️ 与部分游戏反作弊可能冲突 |

---

## 快速开始（Windows）

> 全程使用 Windows 自带 `.NET Framework 4 csc.exe` 编译，**无需安装 MinGW 或 Visual Studio**。

### 1. 桌面版（Antigravity）

编译无窗口原生启动器 `AntigravityProxy.exe` 并改写快捷方式。它无黑框闪屏、原样透传启动参数、可自动适配 `Antigrative Dashboard`（`antigravity-pulse`）内嵌栏调试参数，并对开始菜单快捷方式提供防护：

```powershell
# 1) 编译 AntigravityProxy.exe 到 %USERPROFILE%\.agy-shim\（按你的代理端口修改 -ProxyUrl）
powershell -ExecutionPolicy Bypass -File .\scripts\build-desktop-launcher.ps1 -ProxyUrl http://127.0.0.1:7890

# 2) 改写桌面 / 开始菜单 / 任务栏固定快捷方式指向 AntigravityProxy.exe
#    -AddDebugPort     : 补齐 Dashboard 插件内嵌栏所需的 --remote-debugging-port 参数
#    -ProtectStartMenu : 写回 Electron AppUserModelID 并对开始菜单快捷方式加只读(+R)，
#                        防止桌面通知或重启时被 Electron 静默还原
powershell -ExecutionPolicy Bypass -File .\scripts\repoint-shortcut.ps1 -AddDebugPort -ProtectStartMenu
```

> 改端口只需编辑 `%USERPROFILE%\.agy-shim\proxy.conf`（写 `http://127.0.0.1:7897` 或直接写 `7897`），**无需重新编译**；如之前用过旧版 `.cmd` 启动器，`build-desktop-launcher.ps1` 会自动把其中自定义端口迁移到 `proxy.conf`。

**建议同时安装登录静默自愈**（防止 Antigravity 自动更新重建开始菜单快捷方式）：

```powershell
Copy-Item -Force .\scripts\self-heal-shortcuts.ps1 "$env:USERPROFILE\.agy-shim\"
Copy-Item -Force .\scripts\self-heal-shortcuts.vbs "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\"
```

### 2. 命令行版（`agy`）

```powershell
# 编译 agy.exe 到 %LOCALAPPDATA%\agy\shim\，并以 ExpandString 类型安全加入用户 PATH 最前
# （共享 %USERPROFILE%\.agy-shim\proxy.conf 配置）
powershell -ExecutionPolicy Bypass -File .\scripts\build-cli-shim.ps1 -AddToPath
```

完成后**重开终端**即可。可用 `.\scripts\agy-proxy-toggle.ps1` 随时启用、停用或查看 shim 状态。

### 3. 验证与一键诊断（Windows）

```powershell
# 全面检查桌面启动器、快捷方式、自愈脚本、端口延迟、全局环境洁净度与 CLI shim
powershell -ExecutionPolicy Bypass -File .\scripts\verify-review.ps1

# 一键排查 Clash 模式/策略组与 Google/Antigravity 各域名 TLS 连通性
powershell -ExecutionPolicy Bypass -File .\scripts\diagnose-mode.ps1

# 如已安装 CLI shim
(Get-Command agy).Source        # 应指向 ...\agy\shim\agy.exe
agy --version                   # 应输出 agy-shim 2.1.0 (cs)
agy -p "reply OK"               # 应返回 OK
```

### 4. 一键卸载（Windows）

```powershell
# 预览将要还原/删除的内容（不修改系统）
powershell -ExecutionPolicy Bypass -File .\scripts\uninstall-windows.ps1 -DryRun

# 一键还原快捷方式、清理 PATH、移除自愈启动项与启动器（默认保留 proxy.conf）
powershell -ExecutionPolicy Bypass -File .\scripts\uninstall-windows.ps1

# 彻底卸载（连同 proxy.conf 与空目录一并清理）
powershell -ExecutionPolicy Bypass -File .\scripts\uninstall-windows.ps1 -Purge
```

---

## 快速开始（Ubuntu / Linux）

Linux 下无需编译，CLI 与桌面版共用统一 multi-call Bash wrapper，通过 `exec` 直接替换进程透传全部参数与信号。针对 Ubuntu 下常见的 `tar -xzf Antigravity.tar.gz`（`./Antigravity`）与官方 CLI（`~/.local/bin/agy`）做了专门适配：

- **绝不覆盖官方 `~/.local/bin/agy`**：CLI 代理包装器安装到 `~/.local/share/agy-proxy/bin/agy`，只需在 `PATH` 中排在 `~/.local/bin` 之前即可。
- **自动生成桌面图标**：对于 `Antigravity.tar.gz` 解压安装（无系统 `.desktop`），传入 `--app-bin`（或自动探测）即可生成用户级 `~/.local/share/applications/antigravity.desktop`；若系统已有 `.desktop` 则创建用户级覆盖（升级软件不会被重置）。
- **GNOME 桌面环境兼容**：Ubuntu GNOME 下的 Chromium/Electron 会优先读 `gsettings` 而忽略 `HTTPS_PROXY`，桌面包装器会在检测到可用代理时自动附加 `--proxy-server` 与 `--proxy-bypass-list`。

```bash
# 假设已解压官方压缩包：tar -xzf Antigravity.tar.gz（例如在 ~/Downloads/Antigravity）
chmod +x linux/*.sh

# 安装包装器并记录真实二进制路径（可传目录或 ./Antigravity 路径；省略时自动探测）
# 若本机检测到 antigravity-pulse 插件，会自动补齐内嵌栏调试参数（也可显式传 --with-debug-port）
./linux/install-linux.sh --proxy http://127.0.0.1:7890 --app-bin ~/Downloads/Antigravity

# 确保 shim 目录排在 ~/.local/bin 之前（写入 ~/.bashrc 或 ~/.zshrc）
export PATH="$HOME/.local/share/agy-proxy/bin:$HOME/.local/bin:$PATH"

# 也可直接在解压目录内运行桌面包装器（无配置时会自动探测当前目录的 ./Antigravity）
antigravity-proxy

# 查看状态 / 诊断 / 卸载（默认保留 proxy.conf，绝不删除官方 ~/.local/bin/agy）
./linux/install-linux.sh --status
python3 scripts/diagnose-mode.py
./linux/install-linux.sh --uninstall
```

---

## 文档

| 文档 | 内容 |
|---|---|
| [01 · 技术原理与避坑](docs/01-technical-go-proxy.md) | Go 代理机制、双进程超时根因、进程级注入架构与关键工程坑点 |
| [02 · 使用与排查指南](docs/02-usage-guide.md) | Windows/Linux 安装、CLI 用法、规则/全局模式对比、故障诊断与账号安全 |
| [交接报告](docs/HANDOVER.md) | 架构总结、操作手册、故障决策树与剩余跟进事项 |

---

## 仓库结构

```text
.
├── .github/workflows/ci.yml           # CI：ASCII / LF / PS 语法 / Bash 语法 / Linux 冒烟 / csc.exe 编译
├── README.md
├── LICENSE
├── docs/
│   ├── 01-technical-go-proxy.md       # 技术原理与避坑
│   ├── 02-usage-guide.md              # Windows & Linux 使用指南
│   └── HANDOVER.md                    # 交接与维护文档
├── src/
│   ├── ProxyCommon.cs                 # 共享模块：代理探测、proxy.conf 解析与原始命令行提取
│   ├── AntigravityProxy.cs            # Windows 桌面版无窗口启动器（透传参数、兼容插件）
│   └── AgyShim.cs                     # Windows CLI 转发器（透传 TUI/Ctrl+C/退出码、版本自证）
├── linux/
│   ├── proxy-wrapper.sh               # Ubuntu/Linux 统一 multi-call wrapper（agy & antigravity-proxy）
│   └── install-linux.sh               # Ubuntu/Linux 一键安装、状态检查与卸载脚本
└── scripts/
    ├── build-desktop-launcher.ps1     # 编译 AntigravityProxy.exe 并管理 proxy.conf
    ├── build-cli-shim.ps1             # 编译 agy.exe 并以 ExpandString 更新用户 PATH
    ├── repoint-shortcut.ps1           # 改写快捷方式（保留原参数、写回 AUMID、支持 -AddDebugPort/-ProtectStartMenu）
    ├── self-heal-shortcuts.ps1        # 自包含登录自愈脚本（幂等、绝不清空启动参数）
    ├── self-heal-shortcuts.vbs        # 启动项无窗口包装
    ├── agy-proxy-toggle.ps1           # agy shim 启用/停用/状态检查
    ├── query-mihomo.py                # 跨平台查询 Clash/mihomo 模式与策略组（命名管道 / Unix socket / HTTP）
    ├── diagnose-mode.ps1              # Windows 一键模式/节点/逐域名 TLS 诊断
    ├── diagnose-mode.py               # 跨平台一键模式/节点/逐域名 TLS 诊断
    ├── verify-review.ps1              # Windows 本地部署全面检查脚本
    └── uninstall-windows.ps1          # Windows 一键卸载脚本（支持 -DryRun / -Purge）
```

---

## 关键设计与踩过的坑

1. **统一免依赖方案**：Windows 桌面与 CLI 共用 [src/ProxyCommon.cs](src/ProxyCommon.cs)，用系统自带 `csc.exe` 编译；Linux 共用 [linux/proxy-wrapper.sh](linux/proxy-wrapper.sh)，用 `exec` 零开销分发。代理关闭时（端口探不通）绝不注入，环境变量已配时不覆盖。
2. **`NO_PROXY` 不能漏**：漏掉 `localhost,127.0.0.1,::1` 会导致 Electron 加载 Go 内核本地页面走代理超时/502。
3. **兼容 Dashboard 内嵌栏（`antigravity-pulse`）**：启动器原样保留并透传 `--remote-debugging-port=0 --remote-debugging-address=127.0.0.1`，且 `AntigravityProxy.exe` 命中插件快捷方式识别白名单。
4. **Electron Toast 桌面通知覆写开始菜单**：不仅更新器会重建开始菜单项，Electron 发通知时若检测到开始菜单 `.lnk` 的 `AppUserModelID` 不匹配也会在运行时立刻将其还原并清空参数。`repoint-shortcut.ps1` 会写回 `electron.app.Antigravity`，并支持 `-ProtectStartMenu`（`+R`）阻断覆写。
5. **注册表 `Path` 保持 `ExpandString`**：读取用 `DoNotExpandEnvironmentNames`，写回用 `[RegistryValueKind]::ExpandString`，防止 `%LOCALAPPDATA%` 被破坏。
6. **PowerShell 5.1 编码双重坑**：仓库所有脚本保持**纯 ASCII** 防止中文 Windows PS 5.1 按 GBK 解析报错；`diagnose-mode.ps1` 通过 `ProcessStartInfo` 指定 UTF-8 读取 Python 子进程输出，避免中文策略组名乱码。

---

## License

MIT
