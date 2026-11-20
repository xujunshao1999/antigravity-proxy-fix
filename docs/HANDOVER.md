# 交接报告：Antigravity / agy 代理方案

> **适用系统**：Windows 10 / 11 · Ubuntu / Linux
> **当前版本**：v2.1.0（零 MinGW 依赖 C# 启动器 + Linux multi-call wrapper + 插件内嵌栏与开始菜单保护）
> **仓库**：https://github.com/xiaobaijiadepugongyingpai-coder/antigravity-proxy-fix

---

## 0. 三十秒速览

| 项 | 状态 |
|---|---|
| 核心问题（Go 不读系统代理） | ✅ 已通过进程级环境变量注入彻底解决 |
| Dashboard 插件内嵌栏兼容性 | ✅ 已解决（`AntigravityProxy.exe` + 保留/补全 `--remote-debugging-port`） |
| 开始菜单快捷方式运行时被还原 | ✅ 已查明 Electron Toast AUMID 根因并修复（`IPropertyStore` + `-ProtectStartMenu` `+R`） |
| Ubuntu / Linux 支持 | ✅ 已完成（统一 `linux/proxy-wrapper.sh` + `install-linux.sh` + `query-mihomo.py` 跨平台） |
| 零外部编译依赖 | ✅ Windows 仅依赖自带 `.NET Framework 4 csc.exe`；Linux 纯 Bash + Python 3 |
| 自动化 CI | ✅ 已配置 `.github/workflows/ci.yml`（ASCII / LF / PS AST / Bash / Linux 冒烟 / `csc.exe` 编译） |
| 外部待办 | ⏳ 另一台机器的「规则模式 vs 全局模式」差异待在目标机实测；可选补充 GitHub Releases 预编译包 |

---

## 1. 问题本质

### 1.1 根因

**Go 编译的程序只读环境变量 `HTTP_PROXY` / `HTTPS_PROXY`，完全不读操作系统图形代理设置（Windows 注册表 IE 设置 / WinHTTP / PAC 或 Linux 桌面代理）。**

Antigravity 是双进程架构：

```text
Antigravity（Electron 外壳）   ← 认系统代理，也认环境变量
    └─ language_server（Go 内核） ← 只认环境变量，不认系统代理
```

只开系统代理时，外壳启动后拉起内核，内核直连远端超时，导致 Electron 加载本地页面 `https://127.0.0.1:PORT/` 时呈现 `ERR_TIMED_OUT` 白屏；同目录 `language_server.log` 中的 `dial tcp ... i/o timeout` 是决定性证据。

### 1.2 为什么不用全局环境变量或 TUN

- **全局环境变量（`setx`）**：未配妥会导致本机服务（`127.0.0.1`）走代理返回 `502`，内网设备失灵，且代理一关全系统断网；
- **TUN 模式**：需虚拟网卡驱动，且部分机器存在游戏反作弊兼容风险。

因此始终坚持**进程级注入**：只给 Antigravity / `agy` 子进程注入 `HTTP_PROXY`、`HTTPS_PROXY` 及 `NO_PROXY=localhost,127.0.0.1,::1`，保持用户/系统级环境变量全空。

---

## 2. 当前架构与文件清单

### 2.1 仓库结构

```text
antigravity-proxy-fix/
├── .github/workflows/ci.yml           CI（ASCII / LF / PS / Bash / Linux 冒烟 / csc 编译）
├── README.md
├── LICENSE
├── docs/
│   ├── 01-technical-go-proxy.md       技术原理与工程避坑
│   ├── 02-usage-guide.md              Windows & Linux 完整使用与排查指南
│   └── HANDOVER.md                    本交接文档
├── src/
│   ├── ProxyCommon.cs                 共享模块：代理发现/探活/proxy.conf 读取与原始命令行提取
│   ├── AntigravityProxy.cs            Windows 桌面版无窗口启动器（透传参数，兼容插件）
│   └── AgyShim.cs                     Windows CLI 转发器（透传参数/TUI/Ctrl+C/退出码）
├── linux/
│   ├── proxy-wrapper.sh               Ubuntu/Linux 统一 multi-call wrapper（agy & antigravity-proxy）
│   └── install-linux.sh               Ubuntu/Linux 一键安装、状态检查与卸载脚本
└── scripts/
    ├── build-desktop-launcher.ps1     用自带 csc.exe 编译 AntigravityProxy.exe 并管理 proxy.conf
    ├── build-cli-shim.ps1             用自带 csc.exe 编译 agy.exe 并以 ExpandString 插入 PATH 最前
    ├── repoint-shortcut.ps1           改写快捷方式（保留 Arguments、写回 AUMID、支持 -AddDebugPort/-ProtectStartMenu）
    ├── self-heal-shortcuts.ps1        登录自愈（自包含、幂等、保留参数并写回 AUMID）
    ├── self-heal-shortcuts.vbs        开机启动项无窗口包装
    ├── agy-proxy-toggle.ps1           交互式启用/停用/查看 CLI shim 状态
    ├── query-mihomo.py                查询 Clash/mihomo 模式与策略组（Windows 命名管道 / Linux Unix socket & HTTP）
    ├── diagnose-mode.ps1              Windows 一键诊断模式、策略组与逐域名 TLS（UTF-8 管道防乱码）
    ├── diagnose-mode.py               跨平台一键诊断脚本（Windows + Linux）
    ├── verify-review.ps1              Windows 部署健康度全面检查
    └── uninstall-windows.ps1          Windows 一键卸载脚本（支持 -DryRun / -Purge）
```

### 2.2 关键实现细节

1. **Windows 共享构建（`src/ProxyCommon.cs`）**
   - 保持 C# 5 语法与纯 ASCII，确保 Windows 10/11 原生自带的 `C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe` 零依赖编译。
   - 若环境中已存在 `HTTP_PROXY` / `HTTPS_PROXY`（含小写）则原样放行；否则依次探测 `ANTIGRAVITY_PROXY`/`AGY_PROXY` -> 启动器目录 `proxy.conf` -> `%USERPROFILE%\.agy-shim\proxy.conf` -> `7897` -> `7890` -> `10809`，仅在 TCP 探活成功时注入，绝不在代理未开启时误注入。
   - 使用 `GetCommandLineW` 剥离首段后拼接原样尾部参数，避免二次转义破坏引号/空格；`AntigravityProxy.exe` 的文件名可直接被 `antigravity-pulse` 的 `isAntigravityDesktopShortcut()` 识别。

2. **开始菜单快捷方式保护**
   - 除了安装更新会重建快捷方式外，**Electron 的 Windows Toast 通知在运行时也会校验并覆写 `Start Menu\Programs\Antigravity.lnk`**。
   - `repoint-shortcut.ps1` 与 `self-heal-shortcuts.ps1` 保存开始菜单快捷方式后，会通过 COM `IPropertyStore` 写回 `System.AppUserModel.ID = electron.app.Antigravity`；开启 `-ProtectStartMenu` 后还会附加只读属性 `+R` 阻止运行时覆写（`self-heal-shortcuts.ps1` 特意保持单文件自包含，方便用户单独拷贝到 `%USERPROFILE%\.agy-shim\`）。

3. **Ubuntu / Linux multi-call wrapper**
   - `linux/install-linux.sh` 将单个 `linux/proxy-wrapper.sh` 分别安装为 `~/.local/bin/agy` 和 `~/.local/bin/antigravity-proxy`，由 `basename "$0"` 区分入口与专属环境变量，读取 `~/.config/agy-proxy/proxy.conf` 并通过 `exec` 直接替换当前进程。
   - 修改 `~/.local/share/applications/antigravity.desktop` 覆盖系统 `.desktop`，在检测到 `antigravity-pulse` 或显式传 `--with-debug-port` 时自动补齐 `--remote-debugging-port=0 --remote-debugging-address=127.0.0.1`。

---

## 3. 常用操作手册

### 3.1 Windows 部署与升级

```powershell
# 编译桌面启动器（若需写入默认端口可加 -ProxyUrl http://127.0.0.1:7890）
powershell -ExecutionPolicy Bypass -File .\scripts\build-desktop-launcher.ps1

# 改写快捷方式并启用内嵌栏调试参数 + 开始菜单保护
powershell -ExecutionPolicy Bypass -File .\scripts\repoint-shortcut.ps1 -AddDebugPort -ProtectStartMenu

# 同步自愈脚本
Copy-Item -Force .\scripts\self-heal-shortcuts.ps1 "$env:USERPROFILE\.agy-shim\"
Copy-Item -Force .\scripts\self-heal-shortcuts.vbs "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\"

# （可选）部署 CLI shim
powershell -ExecutionPolicy Bypass -File .\scripts\build-cli-shim.ps1 -AddToPath
```

### 3.2 验证与诊断

```powershell
# 全面检查 Windows 本地部署与全局环境洁净度
powershell -ExecutionPolicy Bypass -File .\scripts\verify-review.ps1

# 一键排查 Clash 模式/节点/逐域名 TLS 延迟
powershell -ExecutionPolicy Bypass -File .\scripts\diagnose-mode.ps1
python .\scripts\diagnose-mode.py

# 自愈脚本静默验证（健康状态下应零输出且退出码为 0）
powershell -ExecutionPolicy Bypass -File "$env:USERPROFILE\.agy-shim\self-heal-shortcuts.ps1"
```

### 3.3 完全撤销（Windows）

```powershell
# 预览将还原的快捷方式、PATH 项与待删文件
powershell -ExecutionPolicy Bypass -File .\scripts\uninstall-windows.ps1 -DryRun

# 一键还原桌面/开始菜单/任务栏快捷方式（保留调试参数与 AUMID、解除 +R）、
# 安全移除 PATH 中的 shim、移除自愈启动项与启动器/shim（默认保留 proxy.conf）
powershell -ExecutionPolicy Bypass -File .\scripts\uninstall-windows.ps1

# 如需连同 proxy.conf 与空目录彻底删除，附加 -Purge
powershell -ExecutionPolicy Bypass -File .\scripts\uninstall-windows.ps1 -Purge
```

### 3.4 Ubuntu / Linux 部署与卸载

```bash
chmod +x linux/*.sh
./linux/install-linux.sh --proxy http://127.0.0.1:7890
./linux/install-linux.sh --status
python3 scripts/diagnose-mode.py
./linux/install-linux.sh --uninstall
```

---

## 4. 故障诊断决策树

### 4.1 `agy` 不可用

```text
第 1 步：agy --version
  ├─ 输出 agy-shim 2.1.0 (cs) → shim 正常，跑 scripts/diagnose-mode.ps1 / .py
  └─ 无输出或仍是官方版本号 → shim 未拦截到
       ├─ (Get-Command agy).Source 指向 bin\ → PATH 顺序被覆盖，重跑 agy-proxy-toggle.ps1 或 build-cli-shim.ps1 -AddToPath
       └─ 指向 shim\ 但报错 → 用 build-cli-shim.ps1 重新编译
```

### 4.2 桌面版白屏 / 缺插件内嵌栏

1. 运行 `scripts/verify-review.ps1` 检查桌面与开始菜单快捷方式是否均指向 `AntigravityProxy.exe` 且 `args` 含 `--remote-debugging-port=0 --remote-debugging-address=127.0.0.1`；
2. 若被覆盖，运行 `repoint-shortcut.ps1 -AddDebugPort -ProtectStartMenu`；
3. 检查 `%APPDATA%\Antigravity\logs\language_server.log` 是否出现 `dial tcp ... i/o timeout`；
4. 运行 `scripts/diagnose-mode.ps1` 区分**端口未开/假节点（全部 000）**与**半死节点（仅 `cloudcode-pa` 系列 000，需换节点）**。

---

## 5. 关键设计决策与维护红线

| 决策 | 理由 |
|---|---|
| Windows 采用内置 `csc.exe` 编译 C# 5 | 零外部依赖（免 MinGW），同时兼顾原生无窗口启动、精确透传参数与退出码 |
| 提取 `src/ProxyCommon.cs` | 消除桌面版与 CLI 重复的 `proxy.conf`、TCP 探活和命令行截取逻辑 |
| `self-heal-shortcuts.ps1` 保持自包含 | 用户会将其单独复制到 `%USERPROFILE%\.agy-shim\`，不能依赖仓库内其他 `.ps1` |
| 所有 `.ps1` / `.vbs` / `.cs` / `.sh` 纯 ASCII | 防止中文 Windows PS 5.1 按 GBK 解析无 BOM UTF-8 脚本报 `ParserError` |
| `diagnose-mode.ps1` 用 `ProcessStartInfo` 读 UTF-8 | 解决中文 Windows PS 5.1 终端（CP936）直接调用 Python 输出策略组名称时的中文乱码 |
| Linux 单文件 `linux/proxy-wrapper.sh` | 消除 CLI 与桌面脚本重复逻辑；保留纯 LF 换行并通过 CI 自动校验 |

### 提交前检查（本地与 CI 一致）

1. 所有 `.ps1` / `.vbs` / `.cs` / `.sh` / `ci.yml` 保持纯 ASCII；`linux/*.sh` 与 `ci.yml` 保持纯 LF（无 CRLF）。
2. 所有 `scripts/*.ps1` 通过 `[System.Management.Automation.Language.Parser]::ParseFile` 解析。
3. `python -m py_compile scripts/query-mihomo.py scripts/diagnose-mode.py`（检查完不要保留 `__pycache__`）。
4. 改过共享逻辑或端口规则后，同步核对文档引用，确保不包含已删除的旧文件名或个人隐私信息。

---

## 6. 剩余跟进事项

| 优先级 | 状态 | 事项 |
|---|---|---|
| 🔴 高 | ⏳ 待目标机实测 | 在报告「规则模式可用、全局模式不可用」的另一台机器上运行 `scripts/diagnose-mode.ps1`（或 `.py`），确认 `GLOBAL` 是否指向了假节点、`DIRECT`、`URLTest` 或对 `cloudcode-pa` 失效的节点 |
| 🟡 中 | ✅ 已显著缓解 | 开始菜单快捷方式还原问题已由 AUMID 写回 + `-ProtectStartMenu`（`+R`）与登录自愈闭环；仅大版本更新后若触发文件重建需登录或手动执行一次自愈 |
| 🟢 低 | ⏳ 可选增强 | 如需完全免命令行编译，可在 GitHub Releases 附带预编译的 `AntigravityProxy.exe` 与 `agy.exe` |
