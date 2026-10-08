# windows-local-service-hidden-start

> 让 Windows 上的本地命令行/服务类程序（如 new-api.exe、各类 xxx.exe 常驻服务）开机自动启动且全程无黑窗口，并排查「以前没窗口、现在冒出黑窗口」的多入口冲突问题的 WorkBuddy / Claude Code 技能。

## 简介

这是一个 AI Agent 技能（Skill）仓库，适用于 WorkBuddy / CodeBuddy / Claude Code。当用户的本地服务程序（网关、API 中转、机器人等）在 Windows 上出现以下烦恼时，本技能提供一套标准化的排查与修复方案：

- 开机后弹出命令行黑窗口，一不小心关掉服务就停了
- 「以前没有窗口，现在突然冒出来一个」——多入口自启互相冲突
- 希望服务后台常驻、开机自启、完全零窗口

适合人群：在 Windows 上长期运行本地 exe 服务（如 new-api、one-api、各类自编译网关）的用户，以及协助这类用户排障的 AI 运维助手。

## 功能特性

- **无窗口自启标准方案**：以「计划任务 + pythonw 包装脚本 + `CREATE_NO_WINDOW`」为推荐路径，给出可直接套用的 `autostart.pyw` 模板
- **多入口冲突排查流程**：五步定位法——查计划任务、启动文件夹、注册表 Run 项、Win32_StartupCommand 视图、进程父链归属，找出「是谁拉起了这个窗口」
- **启动方式对照表**：列明双击 exe、计划任务直接跑 exe、pythonw、wscript、SYSTEM 会话等启动方式的窗口表现与适用性
- **工作目录陷阱警示**：针对使用相对路径数据库（如 `one-api.db`）的服务，强调 `cwd` / `WorkingDirectory` 必须指向 exe 所在目录，避免"数据全没了"的假象事故
- **修复即验证清单**：进程无窗口（MainWindowHandle = 0）、端口监听、接口连通、入口唯一四条硬性验收标准
- **安全回滚指引**：禁用而非删除、脚本改名备份、一键恢复命令

## 工作原理

Windows 控制台子系统（console subsystem）程序被直接启动时，系统必须为其分配一个控制台宿主窗口（conhost），这就是黑窗口的来源。更关键的是：**关闭黑窗口等于向进程发送 CTRL_CLOSE_EVENT，服务会随之退出**——所以隐藏窗口不只是美观，而是保护服务不被误杀。

本技能的核心思路：

1. 用 `pythonw.exe`（无控制台的 Python 解释器）运行一个 `.pyw` 包装脚本
2. 包装脚本内通过 `subprocess.Popen` 以 `CREATE_NO_WINDOW | CREATE_NEW_PROCESS_GROUP` 标志拉起目标 exe，并把 stdout/stderr 重定向到 DEVNULL
3. 由计划任务在登录时（AtLogOn）触发该包装脚本
4. 禁用/备份其他所有自启入口，保证入口唯一

## 安装与使用

把本仓库的目录内容复制到你的技能目录下，文件夹名保持 `windows-local-service-hidden-start`：

- WorkBuddy / CodeBuddy：`~/.workbuddy/skills/windows-local-service-hidden-start/`
- Claude Code：`~/.claude/skills/windows-local-service-hidden-start/`

重启会话后即可通过触发词自动匹配。

**触发词示例**：开机弹黑窗口、命令行窗口、exe 窗口隐藏、误关窗口服务就停、后台常驻、无窗口自启、计划任务隐藏启动、wscript、pythonw、CREATE_NO_WINDOW。

## 项目结构

```
windows-local-service-hidden-start/
├── README.md   # 项目说明（本文件）
└── SKILL.md    # 技能正文：适用场景、核心原理、排查流程、修复动作、
                # 工作目录陷阱、验证清单、环境限制、回滚方法
```

## 注意事项

- 本技能仅适用于 **Windows** 平台，涉及的命令均为 PowerShell / cmd 语法
- 部分环境存在安全策略限制：Bash 工具中直接调用 `wscript.exe` 可能被判定为 LOLBin 拦截；PowerShell 的 `Start-Process`（目标为 shell/解释器时）也可能被拦截。**可靠手段是计划任务 cmdlet**（`Register-ScheduledTask` / `Start-ScheduledTask` / `Disable-ScheduledTask`）
- 修复有窗口的入口时**只禁用、不删除**，多余脚本改名备份（`xxx.vbs` → `xxx.vbs.bak-<日期>`），保证可回滚
- PowerShell 工具 stdout 可能不回显：把结果 `Out-File` 到文件再读取
- 服务程序的参数路径尽量写绝对路径（如 `--log-dir C:\...\logs`）

## License

MIT

## 作者

sheen945
