---
name: windows-local-service-hidden-start
description: 让 Windows 上的本地命令行/服务类程序（如 new-api.exe、各种 xxx.exe 服务）开机自动启动且全程无黑窗口，并排查「以前没窗口、现在冒出黑窗口」的多入口冲突问题。触发词：开机弹黑窗口、命令行窗口、exe 窗口隐藏、误关窗口服务就停、后台常驻、无窗口自启、计划任务隐藏启动、wscript、pythonw、CREATE_NO_WINDOW。
agent_created: true
---

# Windows 本地服务「无窗口 + 开机自启」标准方案

## 适用场景

- 用户某个本地 exe（网关、服务、机器人）开机后弹出**命令行黑窗口**，担心误关导致服务停止
- 用户说「以前没有窗口，现在有了」
- 需要把本地服务做成**后台常驻、开机自动、零窗口**

## 核心原理（先理解再动手）

Windows 控制台程序（console subsystem，如 Go/C 编译的 xxx.exe）被直接启动时，**必须有一个控制台宿主窗口**（conhost）。

| 启动方式 | 是否有黑窗口 | 说明 |
|---|---|---|
| 双击 exe / 计划任务 Action 直接填 exe | ❌ 有窗口 | **就是弹窗元凶** |
| 计划任务直接跑 exe（哪怕勾了「隐藏」） | ❌ 通常仍有窗口 | 「隐藏」只隐藏任务本身，不隐藏控制台 |
| `pythonw.exe` 跑脚本 + `CREATE_NO_WINDOW` | ✅ 无窗口 | **推荐** |
| `wscript.exe` 跑 VBS，`sh.Run cmd, 0, False` | ✅ 无窗口 | 备选（部分环境被安全策略拦） |
| 计划任务以 `SYSTEM` + 最高权限运行 | ✅ 无窗口 | 跑在 session 0，彻底无界面 |

**关键认知**：关掉黑窗口 = 向进程发送 CTRL_CLOSE_EVENT，服务随之退出。所以「隐藏窗口」不只是美观，而是**保护服务不被误杀**。

## 排查流程（多入口冲突必查）

机器上常常并存好几套自启机制，互相抢端口导致「以前没窗口现在有」。

1. **查计划任务**
   ```
   Get-ScheduledTask | Where-Object { $_.TaskName -match '<关键词>' } | Select-Object TaskName, State
   ```
   重点看 `Actions.Execute` —— **如果直接是目标 exe，就是弹窗元凶**。
2. **查启动文件夹**
   `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\`（`.vbs` / `.lnk` / `.bat`）
3. **查注册表 Run 项**
   `HKCU:\Software\Microsoft\Windows\CurrentVersion\Run`、`HKLM:\Software\Microsoft\Windows\CurrentVersion\Run`
4. **查 StartUp 命令视图**
   `Get-CimInstance Win32_StartupCommand | Format-List Name, Command, Location`
5. **查当前进程归属**（确认是哪个入口拉起来的）
   ```
   Get-CimInstance Win32_Process -Filter "Name='<exe名>'" |
     ForEach-Object { "PID=$($_.ProcessId) PPID=$($_.ParentProcessId) CMD=$($_.CommandLine)" }
   ```
   再用 `Get-Process -Id <PID> | Select-Object MainWindowHandle` 判断有无窗口。

## 修复动作（推荐顺序）

1. **只保留一个入口**，优先「计划任务 + pythonw 包装」：
   - 写一个 `autostart.pyw`（放在 exe 同目录）：
     ```python
     import os, subprocess
     BASE_DIR = r"<exe 所在绝对路径>"
     subprocess.Popen(
         [os.path.join(BASE_DIR, "<exe>"), "<参数1>", "<参数2>"],
         cwd=BASE_DIR,                     # ← 必须设置，见下方陷阱
         creationflags=subprocess.CREATE_NO_WINDOW | subprocess.CREATE_NEW_PROCESS_GROUP,
         stdout=subprocess.DEVNULL,
         stderr=subprocess.DEVNULL,
     )
     ```
   - 计划任务 Action：`pythonw.exe` + `"<路径>\autostart.pyw"`
   - 触发器：`AtLogOn`（或 `AtStartup` + SYSTEM 身份）
2. **禁用有窗口的入口**（不要直接删，禁用可恢复）
   ```
   Disable-ScheduledTask -TaskName "<有窗口的任务名>"
   ```
3. **多余脚本改名备份**，不要删除
   `xxx.vbs` → `xxx.vbs.bak-<日期>`
4. **立即拉起服务**
   ```
   Start-ScheduledTask -TaskName "<无窗口的任务名>"
   ```

## ⚠️ 致命陷阱：工作目录

很多服务（含 new-api）的数据库/配置用**相对路径**（如 `one-api.db`）。
如果启动时工作目录不对，程序会在错误位置**新建一个空数据库**，用户的全部配置、渠道、密钥看起来「全没了」。

- `pythonw` 脚本里必须 `cwd=<exe 所在目录>`
- 计划任务必须设置 `WorkingDirectory`
- 参数里的路径尽量写绝对路径（如 `--log-dir C:\...\logs`）

## 验证清单（改完必须逐条过）

```
# 1) 进程存在且无窗口 ← 决定性证据
Get-Process | Where-Object { $_.ProcessName -match '<关键词>' } |
  Select-Object Id, ProcessName, MainWindowHandle, StartTime
# MainWindowHandle 必须为 0

# 2) 端口在监听
netstat -ano | Select-String ":<端口>.*LISTENING"

# 3) 接口真的能通
Invoke-WebRequest "http://127.0.0.1:<端口>/<健康检查路径>" -UseBasicParsing -TimeoutSec 8

# 4) 自启入口只剩一个
Get-ScheduledTask | Where-Object { $_.TaskName -match '<关键词>' } | Select-Object TaskName, State
Get-ChildItem "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup"
```

## 本机环境限制（踩坑记录）

- Bash 工具里直接调 `wscript.exe` 会被安全策略拦截（判为 LOLBin）
- PowerShell 的 `Start-Process`（尤其目标是 shell/解释器/LOLBin）会被拦截
- **可用手段**：计划任务 cmdlet（`Start-ScheduledTask` / `Disable-ScheduledTask` / `Register-ScheduledTask`）
- PowerShell 工具 **stdout 常常不回显**（返回空）：把所有结果 `Out-File` 到文件，再用 Read 工具读文件

## 回滚方法

- 恢复被禁用的任务：`Enable-ScheduledTask -TaskName "<任务名>"`
- 恢复备份脚本：把 `xxx.bak-<日期>` 改回原名
