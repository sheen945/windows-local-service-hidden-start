# windows-local-service-hidden-start

> A WorkBuddy / Claude Code skill that makes local console/service programs on Windows (e.g. `new-api.exe`, any long-running `xxx.exe`) auto-start at boot with zero console windows, and troubleshoots the "there was no window before, now a black window appeared" multi-entry-point conflict.

## Introduction

This is an AI Agent skill repository for WorkBuddy / CodeBuddy / Claude Code. When a user's local service program (gateway, API relay, bot, etc.) suffers from any of the following on Windows, this skill provides a standardized diagnosis-and-fix playbook:

- A console (black) window pops up after boot, and accidentally closing it kills the service
- "There used to be no window, now one suddenly appeared" — multiple autostart entries fighting each other
- The user wants the service to run permanently in the background, start at boot, and show absolutely no window

Target audience: users who run local exe services on Windows over long periods (new-api, one-api, self-compiled gateways, etc.), and the AI assistants helping them troubleshoot.

## Features

- **Standard windowless autostart recipe**: the recommended path is "Scheduled Task + pythonw wrapper script + `CREATE_NO_WINDOW`", with a ready-to-adapt `autostart.pyw` template
- **Multi-entry conflict diagnosis flow**: a five-step method — inspect Scheduled Tasks, the Startup folder, registry Run keys, the `Win32_StartupCommand` view, and the process parent chain — to find out *which entry point spawned the window*
- **Launch-method comparison table**: window behavior of double-clicking the exe, running the exe directly from a Scheduled Task, `pythonw`, `wscript`, and running as SYSTEM in session 0
- **Working-directory trap warning**: for services that use relative-path databases (e.g. `one-api.db`), emphasizes that `cwd` / `WorkingDirectory` must point to the exe's directory, avoiding the "all my data is gone" incident
- **Fix-then-verify checklist**: four hard acceptance criteria — process has no window (`MainWindowHandle = 0`), port is listening, the API actually responds, and only one autostart entry remains
- **Safe rollback guidance**: disable instead of delete, rename scripts as backups, one-command restore

## How It Works

When a Windows console-subsystem program is launched directly, the OS must allocate a console host window (conhost) for it — that's where the black window comes from. More importantly: **closing that window sends CTRL_CLOSE_EVENT to the process, and the service exits**. So hiding the window isn't cosmetic — it protects the service from being killed by accident.

The skill's core approach:

1. Use `pythonw.exe` (the console-less Python interpreter) to run a `.pyw` wrapper script
2. Inside the wrapper, launch the target exe via `subprocess.Popen` with the `CREATE_NO_WINDOW | CREATE_NEW_PROCESS_GROUP` flags, redirecting stdout/stderr to DEVNULL
3. A Scheduled Task triggers the wrapper at logon (`AtLogOn`, or `AtStartup` running as SYSTEM)
4. Disable / rename-backup all other autostart entries so exactly one entry point remains

## Installation & Usage

Copy the contents of this repository into your skills directory, keeping the folder name `windows-local-service-hidden-start`:

- WorkBuddy / CodeBuddy: `~/.workbuddy/skills/windows-local-service-hidden-start/`
- Claude Code: `~/.claude/skills/windows-local-service-hidden-start/`

Restart your session and the skill will auto-match via trigger words.

**Trigger word examples**: 开机弹黑窗口 (black window at boot), 命令行窗口 (console window), exe 窗口隐藏 (hide exe window), 误关窗口服务就停 (closing window kills service), 后台常驻 (run in background), 无窗口自启 (windowless autostart), 计划任务隐藏启动 (hidden scheduled-task launch), wscript, pythonw, CREATE_NO_WINDOW.

## Project Structure

```
windows-local-service-hidden-start/
├── README.md   # Project documentation (this file)
└── SKILL.md    # Skill body: applicable scenarios, core principles, diagnosis
                # flow, fix actions, working-directory trap, verification
                # checklist, environment limitations, rollback procedure
```

## Notes

- Windows only. All commands use PowerShell / cmd syntax
- Some environments enforce security policies: calling `wscript.exe` directly from a Bash tool may be blocked as a LOLBin; PowerShell `Start-Process` (when the target is a shell/interpreter) may also be blocked. **The reliable mechanism is the Scheduled Task cmdlets** (`Register-ScheduledTask` / `Start-ScheduledTask` / `Disable-ScheduledTask`)
- When fixing windowed entries, **disable, don't delete**; rename surplus scripts as backups (`xxx.vbs` → `xxx.vbs.bak-<date>`) so everything is reversible
- PowerShell tool stdout may not echo back: write results to a file with `Out-File` and read the file instead
- Prefer absolute paths in the service's arguments (e.g. `--log-dir C:\...\logs`)

## License

MIT

## Author

sheen945
