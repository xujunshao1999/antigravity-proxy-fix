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

## 代理端口与查找优先级（不写死任何端口）

> 下文示例中的 `7890` / `7897` 仅为常见占位示例，实际请替换成**你自己的代理端口**（如 v2rayN 常见 `10808`/`10809`、Clash 常见 `7897`/`7890`、局域网代理 `http://192.168.x.x:端口`，或 `socks5://127.0.0.1:1080`）。

Windows 启动器/CLI shim 与 Linux 包装器采用完全一致的 4 级优先级，**每次启动都会先做 TCP 探活再注入**（代理关闭时绝不误注入）：

| 优先级 | 来源 | 说明 |
|---|---|---|
| **1** | `HTTPS_PROXY` / `HTTP_PROXY`（含小写） | 终端当前已显式配置代理时，原样放行不覆盖 |
| **2** | `ANTIGRAVITY_PROXY`（桌面）/ `AGY_PROXY`（CLI） | 仅作用于本工具的专属环境变量，可写 URL 或纯端口号 |
| **3** | `proxy.conf` 配置文件 | Windows 位于 `%USERPROFILE%\.agy-shim\proxy.conf`；Linux 位于 `~/.config/agy-proxy/proxy.conf`（支持纯端口、`host:port`、`http://...`、`socks5://...`、`PROXY=...`） |
| **4** | `[fallback]` 默认回退探测 | **仅当上述配置未设置或均不可达时**，才依次回退探测 `127.0.0.1:7897` → `7890` → `10809` |

> 自检脚本（`--status`、`agy-proxy-toggle.ps1`、`verify-review.ps1`）会**优先探测你在 `proxy.conf` 设置的地址并标注 `[from proxy.conf]`**，`7897 / 7890 / 10809` 会明确标注为 `[fallback]`。

---

## 快速开始（Windows）

> - 全程使用 Windows 自带 `.NET Framework 4 csc.exe` 编译，**无需安装 MinGW 或 Visual Studio**。
> - 如首次运行 `.ps1` 遇到「在此系统上禁止运行脚本」，先执行：`Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned`。
> - 安装脚本会自动创建 `%USERPROFILE%\.agy-shim` 等目录，无需手动新建。

### 1. 桌面版（Antigravity）

编译无窗口原生启动器 `AntigravityProxy.exe` 并改写快捷方式。它无黑框闪屏、原样透传启动参数、可自动适配 `Antigrative Dashboard`（`antigravity-pulse`）内嵌栏调试参数，并对开始菜单快捷方式提供防护：

```powershell
# 1) 编译 AntigravityProxy.exe 到 %USERPROFILE%\.agy-shim\（将 7890 换成你的实际代理端口）
powershell -ExecutionPolicy Bypass -File .\scripts\build-desktop-launcher.ps1 -ProxyUrl http://127.0.0.1:7890

# 2) 改写桌面 / 开始菜单 / 任务栏固定快捷方式指向 AntigravityProxy.exe
#    -AddDebugPort     : 补齐 Dashboard 插件内嵌栏所需的 --remote-debugging-port 参数
#    -ProtectStartMenu : 写回 Electron AppUserModelID 并对开始菜单快捷方式加只读(+R)，
#                        防止桌面通知或重启时被 Electron 静默还原
powershell -ExecutionPolicy Bypass -File .\scripts\repoint-shortcut.ps1 -AddDebugPort -ProtectStartMenu
```

> 后续修改代理端口只需编辑 `%USERPROFILE%\.agy-shim\proxy.conf`（写 `http://127.0.0.1:<新端口>` 或直接写端口号，也可重跑 `build-desktop-launcher.ps1 -ProxyUrl ...`），**无需重新编译或重新改写快捷方式**；如曾使用旧版 `.cmd`，`build-desktop-launcher.ps1` 会自动将其中自定义端口迁移到 `proxy.conf`。

**建议同时安装登录静默自愈**（防止 Antigravity 自动更新重建开始菜单快捷方式）：

```powershell
Copy-Item -Force .\scripts\self-heal-shortcuts.ps1 "$env:USERPROFILE\.agy-shim\"
Copy-Item -Force .\scripts\self-heal-shortcuts.vbs "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\"
```

### 2. 命令行版（`agy`）

```powershell
# 编译 agy.exe 到 %LOCALAPPDATA%\agy\shim\，以 ExpandString 类型安全加入用户 PATH 最前，
# 并广播 WM_SETTINGCHANGE（新开终端即生效，共享 %USERPROFILE%\.agy-shim\proxy.conf）
powershell -ExecutionPolicy Bypass -File .\scripts\build-cli-shim.ps1 -AddToPath
```

> 如果新机还没装过桌面版启动器，确保 `%USERPROFILE%\.agy-shim\proxy.conf` 里已写入你的代理地址（或由 `build-desktop-launcher.ps1` 自动生成）即可与 CLI 共用；可用 `.\scripts\agy-proxy-toggle.ps1` 随时启用、停用或查看状态。

### 3. 验证与一键诊断（Windows）

```powershell
# 全面检查桌面启动器、快捷方式、自愈脚本、proxy.conf 端口延迟、全局环境洁净度与 CLI shim
powershell -ExecutionPolicy Bypass -File .\scripts\verify-review.ps1

# 一键排查 Clash 模式/策略组与 Google/Antigravity 各域名 TLS 连通性
powershell -ExecutionPolicy Bypass -File .\scripts\diagnose-mode.ps1

# 命令行验证（重开终端后）
(Get-Command agy).Source        # 应指向 ...\agy\shim\agy.exe
agy --version                   # 应输出 agy-shim 2.1.0 (cs)
agy -p "reply OK"               # 应返回 OK
[Environment]::GetEnvironmentVariable('HTTPS_PROXY','User')   # 应为空
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

Linux 下无需编译，CLI 与桌面版共用统一 multi-call Bash wrapper（[linux/proxy-wrapper.sh](linux/proxy-wrapper.sh)），通过 `exec` 直接替换进程透传全部参数与信号。同时适配官方 `.tar.gz` 解压即用（`./Antigravity` / `./antigravity`）与官方 CLI（`~/.local/bin/agy`）：

- **绝不覆盖官方 `~/.local/bin/agy`**：官方 `install.sh` 会把真正的 CLI 安装到 `~/.local/bin/agy`。本工具将代理 shim 安装到独立目录 `~/.local/share/agy-proxy/bin/agy`，只需在 `PATH` 中排在 `~/.local/bin` 之前即可。
- **自动适配 `Antigravity.tar.gz` 与桌面图标**：无论解压在 `~/Downloads/Antigravity`、`~/Antigravity-x64` 还是其他目录，传 `--app-bin <解压目录或可执行文件>` 即可记录到 `~/.config/agy-proxy/app-bin.conf` 并自动生成 `~/.local/share/applications/antigravity.desktop`；若系统已有 `.desktop` 则创建用户级覆盖（升级软件不会被重置）。
- **GNOME 桌面环境兼容**：Ubuntu GNOME 下的 Chromium/Electron 会优先读 `gsettings` 系统代理而忽略 `HTTPS_PROXY`，桌面包装器在检测到可用代理时会自动附加 `--proxy-server` 与 `--proxy-bypass-list=localhost;127.0.0.1;[::1]`（设置 `ANTIGRAVITY_NO_ELECTRON_PROXY_FLAG=1` 可关闭）。

### 1. 安装（以 `Antigravity.tar.gz` 为例）

```bash
# 假设已通过 tar -xzf Antigravity.tar.gz 解压（例如得到 ~/Antigravity-x64）
# 如尚未安装官方 CLI，可先执行：curl -fsSL https://antigravity.google/cli/install.sh | bash
chmod +x linux/*.sh

# 安装包装器并写入你的代理地址 + 解压目录（兼容 Antigravity / antigravity 可执行文件；省略时自动探测）
# --with-debug-port 会为 .desktop 补齐内嵌栏调试参数（检测到 antigravity-pulse 插件时也可自动开启）
./linux/install-linux.sh --proxy http://127.0.0.1:<你的代理端口> --app-bin ~/Antigravity-x64 --with-debug-port
```

### 2. 持久化 `PATH`（让终端 `agy` 走代理 shim）

将独立 shim 目录写入 shell 启动文件末尾，确保排在 `~/.local/bin` 之前：

```bash
# Bash（Ubuntu 默认）
echo 'export PATH="$HOME/.local/share/agy-proxy/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# 如果你使用 Zsh，请改为：
# echo 'export PATH="$HOME/.local/share/agy-proxy/bin:$PATH"' >> ~/.zshrc
# source ~/.zshrc
```

> 若终端提示 `antigravity-proxy: command not found`，再确保 `~/.local/bin` 也在 `PATH` 中（桌面应用图标不依赖终端 `PATH`，安装完即可直接点击）：
> `export PATH="$HOME/.local/share/agy-proxy/bin:$HOME/.local/bin:$PATH"`

### 3. 验证与启动

```bash
which agy                               # 应输出 ~/.local/share/agy-proxy/bin/agy
agy --shim-version                      # 应输出 agy-shim-linux 2.2.0
./linux/install-linux.sh --status       # 查看配置、.desktop 与端口探测（你的配置会显示 [from proxy.conf]）
agy -p "reply OK"                       # 验证 CLI 走代理连通
antigravity-proxy                       # 或从桌面应用菜单启动 GUI（未配置路径时也会兜底探测当前目录 ./Antigravity）
```

### 4. 改端口、换目录、沙箱排障与卸载

- **更改代理端口**：重跑 `./linux/install-linux.sh --proxy <新端口或URL>`，或直接编辑 `~/.config/agy-proxy/proxy.conf` 立即生效。
- **更换解压目录**：重跑 `./linux/install-linux.sh --app-bin <新目录或可执行文件>`（或编辑 `~/.config/agy-proxy/app-bin.conf`）。
- **Ubuntu 24.04+ `chrome-sandbox` 报错**（部分内核限制非特权用户命名空间）：可先用 `antigravity-proxy --no-sandbox` 验证，或按常规方法授权目录内的 `chrome-sandbox`：
  ```bash
  sudo chown root:root ~/Antigravity-x64/chrome-sandbox
  sudo chmod 4755 ~/Antigravity-x64/chrome-sandbox
  ```
- **跨平台诊断与一键卸载**（默认保留 `proxy.conf`，绝不删除官方 `~/.local/bin/agy`）：
  ```bash
  python3 scripts/diagnose-mode.py
  ./linux/install-linux.sh --uninstall
  ```

---

## 常见问题排查（FAQ）

| 现象 | 常见原因 | 处理方式 |
|---|---|---|
| `--status` 里我的端口是 `open`，下面还有 `7890` `closed` | `7897 / 7890 / 10809` 仅作为**回退兜底**列出 | 只要看到 `<你的端口> open [from proxy.conf]` 即已生效，实际启动只使用你的配置端口 |
| Windows `agy` 仍报 `oauth2.googleapis.com` 超时 | 当前终端是在安装 shim 之前打开的，或被其他 `agy.exe` 抢先匹配 | 重开终端运行 `(Get-Command agy).Source` 与 `agy --version`，确认指向 `shim\agy.exe` |
| Ubuntu 运行 `agy` 未走代理 | `~/.local/share/agy-proxy/bin` 没排在 `~/.local/bin` 前 | 按上文加入 `~/.bashrc`/`~/.zshrc` 并 `source`，确认 `which agy` 指向 `.local/share/agy-proxy/bin/agy` |
| 代理明明开着，依然连接超时（或规则模式行/全局不行） | 节点本身不通 `cloudcode-pa`（半死节点）或全局模式选了 `DIRECT`/失效节点 | 运行 `scripts/diagnose-mode.ps1`（Windows）或 `python3 scripts/diagnose-mode.py`（全平台）逐域名排查并切换节点 |
| Dashboard 内嵌栏不见了 / 开始菜单被还原 | Electron Toast 通知或客户端更新重置了快捷方式 | 重跑 `repoint-shortcut.ps1 -AddDebugPort -ProtectStartMenu`；并部署开机自愈 VBS |
| 想换代理软件/换端口 | — | 修改 `proxy.conf` 第一行或重跑安装/编译脚本带新端口即可，无需改代码 |
| 想完全还原系统默认状态 | — | Windows 跑 `scripts\uninstall-windows.ps1`（加 `-Purge` 清配置）；Linux 跑 `./linux/install-linux.sh --uninstall` |

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
