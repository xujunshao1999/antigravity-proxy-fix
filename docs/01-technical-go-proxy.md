# 本地页面超时，根因却在远端：Antigravity / agy 代理原理与避坑

> **适用版本**：Antigravity 2.33+（Windows 10/11；原生 Ubuntu/Linux 开启系统代理通常可直接工作）
> **症状**：桌面版白屏/转圈，或插件内嵌栏不显示；CLI 登录/请求超时
> **根因**：Go 内核只认环境变量，不读系统代理；传统 `.cmd`/清理参数做法又与插件调试端口及 Electron 通知机制冲突

---

## 一、误导性的表层症状

### 1.1 命令行工具（`agy`）

```text
Got an error: token exchange failed: Post "https://oauth2.googleapis.com/token":
dial tcp 173.194.203.95:443: connectex: A connection attempt failed...
```

这是典型的直连远端失败。

### 1.2 桌面客户端

浏览器正常访问 Google，但 Antigravity 一直转圈，外壳日志出现本地回环超时：

```text
[error] electron: Failed to load URL: https://127.0.0.1:12111/
        with error: ERR_TIMED_OUT
```

看到 `127.0.0.1` 很容易误判为本地服务损坏而反复重装，但重装无效。

---

## 二、真正的线索：内核日志与双进程架构

同目录下的 `language_server.log` 记录了真实错误：

```text
W auth_client.go:415] Failed to get granted scopes in validation:
failed to call tokeninfo: Get "https://www.googleapis.com/oauth2/v3/tokeninfo?access_token=...":
dial tcp 172.217.115.4:443: i/o timeout
```

`dial tcp ... i/o timeout` 是 **Go 运行时 `net.Dial` 的典型报错**。Antigravity 采用双进程架构：

```text
Antigravity（Electron / Chromium 外壳）   ← 认系统代理，也认环境变量
    └─ language_server（Go 内核）        ← 只认环境变量，不认系统代理
```

完整卡死链路：
1. Electron 外壳启动并拉起 Go 内核；
2. Go 内核校验登录态并直连 Google 接口，30 秒后超时；
3. 内核未能就绪，无法返回页面完整数据；
4. Electron 加载 `https://127.0.0.1:PORT/` 超时，呈现为本地白屏。

---

## 三、运行时代理读取机制差异

| 运行时 / 组件 | 系统代理（IE/WinHTTP/GNOME） | `HTTP_PROXY` / `HTTPS_PROXY` 环境变量 |
|---|---|---|
| 浏览器（Chrome / Edge） | 认 | 认 |
| Electron（Chromium 网络栈） | 认 | 认 |
| **Go 程序** | **完全不认** | 认 |
| **Rust 程序** | **完全不认** | 认 |

Go 标准库 `net/http` 通过 `http.ProxyFromEnvironment` 获取代理，**唯一来源是环境变量**（及大小写变体），不读 Windows 注册表 `Internet Settings`、`netsh winhttp`、PAC 文件或桌面环境图形代理设置。

> 因此只开系统代理时，`agy` 与 `language_server` 实际上都在直连。

---

## 四、为什么不用全局环境变量或强制 TUN

| 方案 | 侵入性 | 覆盖范围 | 主要问题 |
|---|---|---|---|
| 全局环境变量（`setx` / `/etc/environment`） | 高 | 全系统 | 本机服务（`127.0.0.1`）被送进代理返回 `502`；内网设备失灵；代理一关全系统断网 |
| TUN / 透明代理 | 中 | 全系统 | 需安装虚拟网卡驱动或高权限；与部分游戏反作弊可能冲突 |
| `netsh winhttp set proxy` | 中 | 仅 WinHTTP 程序 | **对 Go 程序无效** |
| **进程级注入（本仓库）** | **低** | **仅目标进程树** | 需通过本仓库启动器/shim 启动目标程序 |

实测对照组（本机监听 `127.0.0.1:18080`）：
- `direct` -> `200`
- `via proxy`（未配 `NO_PROXY`）-> `502`

因此核心原则是：**只向目标进程树注入代理环境变量，同时显式设置 `NO_PROXY=localhost,127.0.0.1,::1`，绝不污染全局环境。**

---

## 五、进程级注入的实现架构

为了做到零外部编译依赖、支持插件（如 `Antigrative Dashboard`）并保持实现统一，Windows 下采用以下共享架构：

- 共享核心逻辑：[src/ProxyCommon.cs](../src/ProxyCommon.cs)
  - 若进程环境已存在 `HTTP_PROXY`/`HTTPS_PROXY`（含小写）则原样放行；
  - 否则依次读取专属环境变量（`ANTIGRAVITY_PROXY` / `AGY_PROXY`）、`proxy.conf`（程序目录或 `%USERPROFILE%\.agy-shim\proxy.conf`）、以及默认回退端口 `7897` / `7890` / `10809`，仅在 TCP 探活成功时注入 `HTTPS_PROXY`、`HTTP_PROXY` 和 `NO_PROXY`；
  - 通过 `GetCommandLineW` 提取原始尾部参数串原样透传，避免二次解析破坏引号或空格。
- 桌面版启动器：[src/AntigravityProxy.cs](../src/AntigravityProxy.cs)（由 [scripts/build-desktop-launcher.ps1](../scripts/build-desktop-launcher.ps1) 调用 Windows 自带 `.NET Framework 4 csc.exe` 编译为无窗口 GUI exe）。
  - 命名 `AntigravityProxy.exe` 可直接命中 `antigravity-pulse` 的桌面快捷方式识别白名单，从而与 `--remote-debugging-port=0 --remote-debugging-address=127.0.0.1` 和谐共存。
- CLI 转发器：[src/AgyShim.cs](../src/AgyShim.cs)（由 [scripts/build-cli-shim.ps1](../scripts/build-cli-shim.ps1) 编译为控制台 exe 放到 `PATH` 最前），支持 `--version` 版本自证、`Ctrl+C` 透传与退出码精确返回。

> **关于原生 Linux / Ubuntu**：与 Windows 不同，原生 Ubuntu/GNOME 在开启系统代理时，桌面会话与终端通常已同步下发代理环境变量（不存在“Electron 认注册表、Go 内核完全拿不到变量”的分裂），实测直接启动原版 GUI 与 `agy` 即可正常工作，不需要额外 wrapper。但如果 Windows 客户端跑在 **WSL2 模式**下（`%APPDATA%\Antigravity\wsl_state.json` 的 `distro` 非空），请参阅 [04 · WSL 模式说明](04-wsl-mode.md)。

---

## 六、关键工程坑点与解法

### 坑 1：`NO_PROXY` 必须同时注入

漏掉 `NO_PROXY=localhost,127.0.0.1,::1` 时，Electron 加载 Go 内核监听的本地 HTTPS 地址也会绕进代理，同样导致 `ERR_TIMED_OUT` 或 `502`。

### 坑 2：Electron Toast 通知与更新导致开始菜单快捷方式还原

不只是 Squirrel 自动更新会重建开始菜单项——**Electron 自身的 Windows Toast 桌面通知机制在运行时就会重写 `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Antigravity.lnk`**。
- 若快捷方式丢失 `System.AppUserModel.ID = electron.app.Antigravity` 或目标不匹配，任务通知触发时 Electron 会立刻将快捷方式改回 `Antigravity.exe` 并清空 `Arguments`（桌面快捷方式通常不受影响）。
- 本仓库的 [scripts/repoint-shortcut.ps1](../scripts/repoint-shortcut.ps1) 与 [scripts/self-heal-shortcuts.ps1](../scripts/self-heal-shortcuts.ps1) 会：
  1. 改写快捷方式时始终保留已有参数；
  2. 通过 COM `IPropertyStore` 写回 `System.AppUserModel.ID = electron.app.Antigravity`；
  3. 配合 `-ProtectStartMenu` 为开始菜单 `.lnk` 设置只读属性（`+R`），阻断运行时静默覆写；
  4. 配合 [scripts/self-heal-shortcuts.vbs](../scripts/self-heal-shortcuts.vbs) 登录静默自愈。

### 坑 3：注册表 `Path` 的 `ExpandString` 类型陷阱

修改用户 `PATH` 时有两处常见破坏：
- 直接用 `$key.GetValue('Path')` 会把 `%LOCALAPPDATA%` 展开为硬编码路径；
- 用 `Set-ItemProperty` 或 `[Environment]::SetEnvironmentVariable('Path', ..., 'User')` 写入，会把注册表值类型从 `REG_EXPAND_SZ` 降级成 `REG_SZ`；
- 直接通过 `[Microsoft.Win32.Registry]::SetValue` 写注册表**不会**主动发送 `WM_SETTINGCHANGE ("Environment")` 广播，导致 `explorer.exe` 和之后打开的新终端仍继承旧 `PATH`；因此 [scripts/build-cli-shim.ps1](../scripts/build-cli-shim.ps1) 与 [scripts/agy-proxy-toggle.ps1](../scripts/agy-proxy-toggle.ps1) 写入后都会显式调用 `SendMessageTimeoutW` 广播变更，使新开终端无需注销即可生效。

正确做法（[scripts/build-cli-shim.ps1](../scripts/build-cli-shim.ps1) 与 [scripts/agy-proxy-toggle.ps1](../scripts/agy-proxy-toggle.ps1) 均已采用）：

```powershell
$raw = $key.GetValue('Path', '', [Microsoft.Win32.RegistryValueOptions]::DoNotExpandEnvironmentNames)
[Microsoft.Win32.Registry]::SetValue(
    'HKEY_CURRENT_USER\Environment', 'Path', $newPath,
    [Microsoft.Win32.RegistryValueKind]::ExpandString)
```

### 坑 4：中文 Windows + PowerShell 5.1 的编码双重坑

1. **脚本源码编码**：无 UTF-8 BOM 且含中文的 `.ps1` 在 PS 5.1 下会按 GBK 解析并报 `TerminatorExpectedAtEndOfString`。因此仓库内所有 `.ps1` / `.vbs` / `.cs` / `.sh` **一律纯 ASCII**（由 `.github/workflows/ci.yml` 自动拦截）。
2. **捕获子进程 UTF-8 输出乱码**：Python 工具输出 UTF-8 中文（如策略组名称），PS 5.1 默认管道按控制台代码页（CP936）解码会导致乱码。[scripts/diagnose-mode.ps1](../scripts/diagnose-mode.ps1) 通过 `ProcessStartInfo` 显式指定 `StandardOutputEncoding = UTF8` 解决。

### 坑 5：PowerShell `ExecutionPolicy` 与 `netsh winhttp`

- 若报错「在此系统上禁止运行脚本」，将当前用户策略改为推荐的日常档位：
  ```powershell
  Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
  ```
- `netsh winhttp set proxy` 对 Go 程序无效，必须通过进程环境变量注入。
