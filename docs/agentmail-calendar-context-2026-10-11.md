# AgentMail / 日历上下文导出

导出时间：2026-10-11  
对话会话：`c6493952-4029-4fba-87f8-2e9da6d9e0d1`  
仓库工作区：`/Users/jason/Projects/Work/hackathon`

---

## 1. 这段对话在做什么

围绕 **AgentMail**（OctoSense 脚本应用）和 **OctoSense 宿主**，主要做了：

1. 本地草稿持久化（重启后草稿箱仍可见；第二次打开已有草稿不再弹红字保存提示）
2. 模型调用「出现未知错误」：当天 `agent-mail` 代币额度接近上限（约 99816 / 100000）
3. **日历写入 Approve 后被 macOS 立刻拒绝**（重点排查）
4. 成功写入系统日历后的说明：为什么会写 Mac，以及能不能改写 OctoSense 内部日历
5. 曾向 OctoSense 开过 issue/PR，随后按要求关闭

---

## 2. 关键结论（日历）

### 2.1 成功结果（用户截图）

助手回复确认：

- 已加入日程
- 上课
- 2026-10-06 09:00 – 10:00（Asia/Shanghai）
- 218教室
- **已写入系统日历**

### 2.2 为什么点 Approve 会被 Mac「直接拒绝」

这里有两层「批准」：

| 层 | 含义 | 结果 |
|---|---|---|
| OctoSense 审阅表 `Approve` | 应用/宿主自己的确认 | 成功（真实点击、非后台） |
| macOS TCC / EventKit | 系统日历隐私授权 | 曾失败，不弹窗，约 24–31ms 返回拒绝 |

失败原因（有运行证据）：

1. **EventKit 权限请求原先跑在后台线程**  
   `requestFullAccessToEventsWithCompletion:` 在 worker 上会立刻 `granted=false`，`EKAuthorizationStatus` 停在 `0`（`notDetermined`），系统对话框不出现。

2. **TCC 把请求算到 Cursor 头上**  
   从 Cursor 里直接执行 `OctoSense.app/Contents/MacOS/octosense` 时，`tccd` 的 responsible process 是 Cursor。Cursor 有 hardened runtime，但没有 `com.apple.security.personal-information.calendars`，于是：
   - `Prompting policy for hardened runtime; service: kTCCServiceCalendar requires entitlement ... missing for responsible=Cursor`
   - `Policy disallows prompt ... access to kTCCServiceCalendar denied`

3. **次要宿主问题（排查途中也修过）**
   - macOS 14+ 完全访问状态值 `5`、只写 `4`；旧代码只认 `3` 为已授权
   - 系统权限窗抢走焦点后，`WindowLostFocus` 清空 `FOREGROUND`，提交被 `foreground_required` 取消

修复方向（本地已验证成功）：

- 宿主：把 EventKit 权限请求派到 **主线程**；把 3/4/5 都当 granted；Submitted 审阅期间不因失焦取消
- 启动：用 Launch Services `open` 打开 `.app`，父进程为 `launchd`，并给 bundle 签上日历 entitlement（见 `AgentMail/scripts/update-and-open.sh`）

### 2.3 能不能只写 OctoSense 内部日历、不写 Mac？

**可以，而且本来就有这条路。不是「一定要用 Mac」。**

| 接口 | 存哪 | 是否弹 macOS 权限 |
|---|---|---|
| `calendar.add_event` / `calendar.events` | OctoSense 本地 `calendar/events.json`，日历应用月/日视图读这份 | 否 |
| `device_calendar.events.create` 等 | Mac EventKit / 系统日历 | 是 |

AgentMail **当前实现走的是 `device_calendar.*`**，所以成功后文案是「已写入系统日历」。  
接入时选了系统日历这条线，不是做不到写 OctoSense 日历。

注意：邮件界面不能随意 `host.request("calendar.add_event")`。日历服务只认 `os.calendar` 身份；其它应用需经可共享工具（`shareable: true`）由宿主/日历侧代写。

---

## 3. 关键运行日志材料（可作 issue/PR 依据）

> 调试 session：`c64939`  
> 日志路径曾用：`/Users/jason/Projects/Work/hackathon/.cursor/debug-c64939.log`（确认修复后已清理 instrumentation 并删除该文件）  
> 系统 TCC 日志来自 macOS `log show` / `tccd`

### 3.1 应用侧诊断日志（失败轮次，主线程修复前）

```json
{"message":"approve called","data":{"background":false,"contained":false,"down":true,"up":true},"hypothesisId":"E"}
{"message":"eventkit status before request","data":{"already_granted":false,"name":"not_determined","status":0},"hypothesisId":"B"}
{"message":"usage description lookup","data":{"bundle_id":"com.octosense.desktop.local","key":"NSCalendarsFullAccessUsageDescription","modern":true,"present":true},"hypothesisId":"D"}
{"message":"eventkit completion","data":{"allowed":false,"completion_granted":false,"name_after":"not_determined","status_after":0},"hypothesisId":"A"}
{"message":"permission commit result","data":{"os_permission":"denied"},"hypothesisId":"A"}
```

说明：Approve 有效；Info.plist 用途说明存在；状态始终 `notDetermined`。

### 3.2 主线程修复后仍失败（约 31ms）

```json
{"message":"permission caller thread","data":{"main_thread":false},"hypothesisId":"F"}
{"message":"permission request thread","data":{"main_thread":true},"hypothesisId":"F"}
{"message":"eventkit completion","data":{"allowed":false,"completion_granted":false,"name_after":"not_determined","status_after":0},"hypothesisId":"A"}
```

说明：请求已在主线程，仍被立刻拒绝 → 还有第二原因（TCC 归因）。

### 3.3 macOS TCC 原文（关键）

```
Prompting policy for hardened runtime; service: kTCCServiceCalendar requires entitlement
com.apple.security.personal-information.calendars but it is missing for
responsible={Cursor /Applications/Cursor.app},
accessing={com.octosense.desktop.local .../OctoSense.app/Contents/MacOS/octosense}

Policy disallows prompt for Sub:{com.todesktop.230313mzl4w4u92};
access to kTCCServiceCalendar denied
```

访问进程是 OctoSense；负责进程是 Cursor。

### 3.4 宿主侧曾出现的相关错误串

- `authorization_required: OS calendar access was not granted`
- `foreground_required: Calendar operation cancelled before native execution`
- `host_not_configured: Install the packaged app with Calendar usage descriptions`（用途说明缺失时）

---

## 4. GitHub（已按要求关闭）

曾创建、后关闭：

- Issue：https://github.com/OctoSense-org/OctoSense/issues/496  
  （macOS calendar permission is denied immediately and never prompts）
- PR：https://github.com/OctoSense-org/OctoSense/pull/497  
  （Ask EventKit for calendar access on the main thread）

Fork 分支仍可能在：

- https://github.com/Jasonsu22/OctoSense/tree/fix/macos-eventkit-permission  
- 提交：`04e87b74`

本地 OctoSense 分支：`fix/macos-eventkit-permission`（相对 `origin/main` 有日历相关提交）

---

## 5. 相关本地路径

| 内容 | 路径 |
|---|---|
| AgentMail 源码 | `/Users/jason/Projects/Work/hackathon/AgentMail` |
| 启动脚本（含 Launch Services 打开 .app） | `AgentMail/scripts/update-and-open.sh` |
| 本地运行目录 | `/Users/jason/Projects/Work/hackathon/.octosense-run` |
| 宿主日历 EventKit | `OctoSense/crates/shell/src/device_calendar/macos.rs` |
| 宿主日历审阅/前台 | `OctoSense/crates/shell/src/device_calendar/mod.rs` |
| OctoSense 内部日历服务 | `OctoSense/apps/calendar/host-service` |
| 日历可共享工具 | `OctoSense/apps/calendar/bundle/tools.json`（`calendar.add_event` 等） |
| 本对话原始 transcript | `~/.cursor/projects/Users-jason-Projects-Work-hackathon/agent-transcripts/c6493952-4029-4fba-87f8-2e9da6d9e0d1/c6493952-4029-4fba-87f8-2e9da6d9e0d1.jsonl` |

---

## 6. AgentMail 其它已落地点（同会话）

- 草稿写入 `desktop-apps/agent-mail/drafts.txt`，重启后草稿箱应显示；二次打开已有草稿不重复红字「已保存到本地」
- AI「未知错误」曾与当日 model ledger 额度有关（默认约 10 万 token/天）；本地曾提高 `agent-mail` limits
- 推送过 AgentMail：`https://github.com/Jasonsu22/AgentMail`（含草稿持久化相关提交）
- 总仓子模块：`https://github.com/Jasonsu22/OctoSense-Hackathon`

---

## 7. 若以后要改「写内部日历」

方向（尚未改代码，仅备忘）：

1. AgentMail 不再（或不再优先）走 `device_calendar.*`
2. 声明/申请对 `calendar.add_event`（及如需 `calendar.events`）的可共享工具授权
3. 通过宿主工具中继以日历应用身份写入 `calendar/events.json`
4. UI 文案改为「已加入 OctoSense 日历」之类，而不是「已写入系统日历」

---

## 8. 原始 transcript

完整机器可读对话记录：

`/Users/jason/.cursor/projects/Users-jason-Projects-Work-hackathon/agent-transcripts/c6493952-4029-4fba-87f8-2e9da6d9e0d1/c6493952-4029-4fba-87f8-2e9da6d9e0d1.jsonl`

本文件是同一上下文的**人工整理版**，方便阅读与再提 issue/PR。
