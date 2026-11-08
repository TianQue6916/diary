# journal —— 每晚 23:59 自动记日记

日界按你拍板的 **23:59 硬切**（本地日期 00:00–23:59）。日记每天落到
`~/.dsh/journal/days/YYYY-MM-DD.md`，另抄一份到 `~/Documents/日记/`。

## 数据流

```
sampler.pyw（常驻，10s 一次）
   └→ ~/.dsh/journal/activity/YYYY-MM-DD.jsonl    前台窗口边沿记录
run_daily.py（23:59 / 登录后 3min）
   ├→ aggregate.py   机械归并 5 类源，写 raw/YYYY-MM-DD.json（全量明细，可回溯）
   ├→ diarist.py     调 127.0.0.1:8788（commandcode gateway）写成中文日记
   └→ days/*.md + index.md + ~/Documents/日记/
```

源清单：dsh session 全文（`~/.dsh/sessions/*/*/session.v4.jsonl.zstd`）、
`~/.dsh/logs/tool-calls.jsonl`、`~/.dsh/logs/dsh-autocommit.log`、
Edge History、前台窗口 activity。

## 三条判据（踩过坑才定下来的）

1. **真人指令 = `source.kind == "user"` 且存在 `source.rpcId`**。
   本机实测：28 号 `kind=user` 的消息 194 条，其中 139 条没有 rpcId，全是 subagent
   派发与 `[dsh-mnemon:...]` 自动注入。只按 kind 判会把 worker prompt 当「我说的话」，
   日记立刻失真。mnemon 注入另按前缀剔除。

2. **session 文件的 mtime 上下界都要放宽 2 天**。跨天会话的 mtime 是「最后一次写」，
   用「mtime 落在当天」过滤会整段漏掉「昨天开的、今天还在写」的主会话（27 号曾因此
   显示 0 条指令，实际 26 条）。

3. **单段窗口时长封顶 15 + 5 min**。窗口标题不变但人已经走了，不封顶会把挂机算成学习。

## 课程识别

`aggregate.COURSE_MAP` 把 6.006 / 6.431 / EE364a / CS144 / 6.042J / 实分析 / 信息论 /
CSAPP 映射成关键词，对窗口标题、prompt、文件路径、autocommit 文件名做匹配。

- 有窗口时长的命中 → 计入 `course_minutes`（可信时长）。
- 只有日志痕迹的命中 → 进「课程线索」，**不折算学时**，并且明确告诉 LLM 不许折算。

要加课就把关键词加进 `COURSE_MAP`，不用改别的。

## LLM

`127.0.0.1:8788/v1` —— 那是你自己的 goat-gateway，上游是 commandcode 账号池，
不是官方 API。默认 `deepseek/deepseek-v4.1-flash`，失败按
`JOURNAL_LLM_FALLBACKS`（v4.1-flash → v4-flash → glm-5.3-flash）换模型，
每个模型内部 3 次指数退避。全部失败就降级成规则台账，日记照样落盘，
front matter 里 `generator` 和 index 的「生成方式」列会标出来。

## 计划任务（可选，装了更稳）

本机当前 shell 跑在 **Low Mandatory Level**，`schtasks` / `reg` / 写 Startup
一律 Access denied（PLAN.md 第 948 行的结论在这里应验）。所以：

- **不装任务也能用**：sampler 内置调度，23:59 自己拉起 `run_daily.py`；
  启动 60 秒后还会跑一次 `--catchup 3` 补最近 3 天。
- **装了更稳**：开机自启 + 关机错过时登录后补跑。
  双击 `一键安装-日记.cmd`（右键 → 以管理员身份运行）即可，日志在
  `~/.dsh/logs/journal-install.log`。

## 手工命令

```bash
python run_daily.py                     # 今天（幂等，已有就跳过）
python run_daily.py --date 2026-10-27 --force
python run_daily.py --catchup 7         # 补最近 7 天（不含今天）
python aggregate.py --date 2026-10-27   # 只看素材，不调 LLM
```

## 验证与回滚

- 验证：`cat ~/.dsh/journal/index.md`；`tail ~/.dsh/journal/run.log`；
  想核对 AI 有没有编，打开 `raw/YYYY-MM-DD.json` 逐条对。
- 回滚：`schtasks /Delete /TN DSH-Journal-Daily /F`（Catchup 同理，需提权）、
  删 `Startup\dsh-journal-sampler.vbs`、`taskkill` 掉 pythonw 的 sampler。
  代码全在 `~/.dsh/storages/tools/journal/`，删目录即净。

## 已知限制

- **Linux 侧还没接**。那边需要同款 sampler（`xdotool getactivewindow getwindowname`
  或读 `_NET_ACTIVE_WINDOW`），输出到能被 Windows 读到的 NTFS 路径。
- 本 shell 是 Low IL，写不了 `D:\`（火绒/ACL 拦），所以发布副本落在 `~/Documents/日记/`。
- `active_min` 在 2026-10-28 之前都是 0 —— sampler 那天 16:16 才上线，历史窗口数据无法回补。


## GitHub 同步（双轨 + 脱敏）

公开仓库：**https://github.com/TianQue6916/diary**

| 目录 | 内容 | 去向 |
|---|---|---|
| `days/` | 完整日记：真实用户路径、主机名、指令原话 | **只在本机** |
| `days-public/` | `sanitize.py` 脱敏后 | 公开仓库 |
| `activity/<host>/` | 每 10s 前台窗口原始采样 | 双机同步，不公开 |
| `raw/` | 结构化素材（含 prompt 原文） | 只在本机 |

`sanitize.py` 的 `RULES` 表（改那个文件即可扩展）：
`C:\Users\27063` → `%USERPROFILE%`、`/home/tianque` → `%HOME%`、
`天阙九泉` / `tianque` → `<host:win>` / `<host:linux>`、
`sk-*` / `ghp_*` / `github_pat_*` → `<token>`、邮箱 → `<email>`、
Tailscale `100.64.0.0/10` 与 `192.168.*` / `10.*` → `<ip:*>`。
课程名、项目名、dsh / skill / patch 等技术词一律保留 —— 那是公开的价值所在。

2026-11-02 做过一次**历史重建**：早期提交直接推过未脱敏的 `days/`，
所以 `.git` 整体重建后 force push（旧历史已从远端覆盖，本地备份在
`journal/.git-backup-20261102-pre-scrub`）。

推送由 `run_daily.py` 末尾调 `sync_git.py` 完成，SSH 走
`C:\Windows\System32\OpenSSH\ssh.exe`（MSYS 的 ssh 在 Low IL 下报
`NtCreateObject 0xC0000022`）。离线不丢：commit 留本地，下次自动补推。

## 双机：数据怎么合到一起（2026-11-07 改）

**不再依赖 `sync_dsh.py` 的双向同步。** 实测它从 2026-11-02 之后就没跑过，
而且 Linux 侧那份脚本还是旧版（把 `journal/days/*.md` 也判成该同步）。
五天里 Windows 一次都没收到 Linux 的采样，日记里自然看不到 Linux。

现在改成 **Windows 侧主动拉**：`pull_remote.py`，`run_daily.py` 每次生成前执行一次

```
ssh tianque-lan ls .dsh/journal/activity        # 问对端有哪些 host
scp tianque-lan:.dsh/journal/activity/<h>/*.jsonl  ->  本地 activity/<h>/
```

单向、幂等、失败只记日志不阻塞。日记的时效性不该押在「有人记得去跑同步脚本」上。

## 旧的同步说明（已被上面的方案取代，保留作历史）

`activity/<host>/YYYY-MM-DD.jsonl` 文件名带 host，两机不会互相覆盖。
`storages/tools/sync_dsh.py` 曾在 2026-10-31 把 `journal` **整个目录**排除
（为压冲突），代价是 Linux 的采样永远传不过来。2026-11-02 改成精细放行：

```python
_JOURNAL_SYNC_RE = re.compile(r'^journal/activity/[^/]+/[0-9]{4}-[0-9]{2}-[0-9]{2}\.jsonl$')
```

即**只同步 `journal/activity/<host>/*.jsonl`**；`days/`、`days-public/`、`raw/`
仍各机独立（同步它们只会互相覆盖）。

约定：**日记由 Windows 生成**，一天一份，合并两机 activity；Linux 只采样。
若当天 Linux 全离线，日记里只会出现 `win` 的数据，front matter 的
`hosts` 字段能看出来。

## 时间轴怎么算（2026-11-02 重写）

`sessionize()` 把碎窗口合并成「活动块」：

- 判据：同 exe + 同主题（8 字符前缀互含）+ 相邻间隔 ≤ 20min
- 跳过 `away`（锁屏 / 登录界面，`_topic` 里 `AWAY_WORDS`）
- 输出两个数：`span_sec`（首尾时间差）= 「跨 Nmin」，`sec`（真正窗口时间）= 「窗口 Mmin」

这不是装饰。2026-11-01 的真实数据：`21:41–23:30` 跨 109min，实际窗口只有
55min（8 段）—— 把两者混成一个数，等于把「挂机」算成工作。

## Linux / GNOME Wayland 怎么拿窗口（社区经验，2026-11-02 查证）

先说结论：**GNOME Wayland 没有官方的 active-window API，唯一可靠途径是 GNOME Shell 扩展 + D-Bus。**

证据链：

- ActivityWatch 作者 Johan Bjäreholt 在 SO 上的原话：Wayland 下拿不到，但
  *"It is however still possible to access this API via a GNOME Shell extension"*
- ActivityWatch 论坛维护者给 GNOME-Wayland 用户的建议：用 `awatcher` **+ 一个 GNOME 扩展**
- `aw-watcher-window-wayland`（Rust）**只支持 wlroots 系**（Hyprland / sway），
  因为 GNOME 不实现 `wlr-foreign-toplevel-management`
- 一篇专门做跨 Hyprland/GNOME/KDE 追踪的文章结论一致：
  *"GNOME on Wayland is the painful one. The only working solution I found is a shell
  extension called window-calls-extended which exposes a DBus interface"*

### 本机实测走不通的三条路

| 手段 | 结果 |
|---|---|
| `xprop -root _NET_ACTIVE_WINDOW` | 返回 `0x600003`，那是 gnome-shell 自己的窗口；`xprop -id` 它拿不到任何属性 |
| `_NET_CLIENT_LIST` / `wmctrl -l -p` | 全场只有一个 `WPS Office`，原生 Wayland 窗口不在 X 层 |
| `org.gnome.Shell.Introspect.GetWindows` | `AccessDenied`，GNOME 默认禁 |

### 当前实现：双路径

`sampler_linux.py` 启动时先试 D-Bus（`window-calls-extended`），拿不到就退回 AT-SPI：

```
模式 = dbus(前台窗口, window-calls-extended)   # 扩展加载后
模式 = at-spi(可见窗口集合)                      # 兜底，现在跑的这条
```

两条路写同一种文件，区别只在事件类型：dbus 模式写 `k:"sw"`（和 Windows 一致，是**真前台**），
AT-SPI 模式写 `k:"vis"`（**可见窗口集合**，不是前台）。aggregate 两种都认，
A3 段就是 `vis` 的产物，措辞上只能写「打开着」，不能写「一直在看」。

### 已经在 Linux 上装好的

```bash
~/.local/share/gnome-shell/extensions/window-calls-extended@hseliger.eu/
# metadata.json 的 shell-version 里补了 "50"（仓库里只写到 49，本机是 GNOME Shell 50.1）
gsettings set org.gnome.shell enabled-extensions [... 'window-calls-extended@hseliger.eu']

~/.config/systemd/user/dsh-journal-sampler.service   # enabled + active
```

**扩展需要重新登录一次才加载**（Wayland 下不能热重启 shell）。

2026-11-07 已确认生效：

```
linux sampler: 模式 = dbus(前台窗口, window-calls-extended)
{"k": "sw", "exe": "marktext", "title": "backward-induction-逆向归纳法与最优性原理.md"}
```

现在 Linux 写的 `k:"sw"` 和 Windows 完全同语义（**真前台**），A3 段只在扩展不可用时才出现。

### 一个判断逻辑的坑

一开始用「能否拿到 focused 窗口」来决定走哪条路，结果用户不在 Linux 前的时候
所有窗口 `focus:false`，被判成「扩展不可用」一路退回 AT-SPI。
**可用性判断必须和瞬时状态分开**：现在是 `dbus_available()`（List 能解析出列表就算可用），
focused 拿不到只是这一轮没有切换，不改变模式。

还有一个不上扩展的野路子：在 `Alt+F2` 的 Looking Glass 里 `global.context.unsafe_mode = true`
之后可以 `org.gnome.Shell.Eval`。**没用它** —— 那等于把整个 shell 的 JS 执行权开放给外部进程，
比装一个功能单一的扩展危险得多。

## 时长为什么会「一直是 15 分钟的整数」（2026-11-08 修）

用户反馈日记里的时长全是 5/10/15/20 分钟整。三处独立原因叠在一起：

**① Linux 的离席判定用错了信号。** `IDLE_SKIP = 900` 原本判的是「可见窗口集合
900 秒没变」。读一篇文档 15 分钟，窗口当然一动不动 —— 于是每一段认真阅读都被
判成「人走了」切断，段长恰好卡在 15 分钟。

社区正解：`org.gnome.Mutter.IdleMonitor.GetIdletime`，GNOME 在 X11 和 Wayland 上
都通过 session D-Bus 暴露合成器的**真实输入空闲**（返回 ms）。三份独立实现
都走这条路：[super-productivity `electron/idle-time-handler.ts`][sp]、
[EvoMap/evolver `src/gep/idleScheduler.js`][ev]、
[LunaBox `idle_linux.go`][lb]。KDE Wayland 的
`org.freedesktop.ScreenSaver.GetSessionIdleTime` 直接返回 NotSupported。

实测：`(uint64 62891,)` → 63 秒，3 秒后 `65919` → 66 秒。可用。

**② 心跳 300 秒决定了段尾精度。** `BEAT = 300` 时，一段「窗口没变」的活动
只能靠心跳延长尾部，误差就是一个心跳 —— 于是时长被量化成 5 分钟网格。
两侧都改成 60 秒（一天约 1440 条心跳，~65KB，可忽略）。

**③ `GAP_SWITCH = 15*60` 直接把单段压到 20 分钟。** `capped = min(dur, GAP_SWITCH + BEAT)`
= 1200 秒，任何长于一节的连续工作都被截断。这个上限原本是给「挂机」兜底的，
但挂机现在由真实输入空闲判定，不需要它了 —— 放宽到 2 小时，只作极端兜底。

另外修了段尾的最后一截：idle 记录的是「发现已空闲超过阈值」的时刻，人要更早
就走了。新记录带 `"src": "input"`，aggregate 用 `ts - sec` 回推真实离开时刻。
老数据没有 `src`（Linux 那时写的是「窗口未变秒数」，语义完全不同），一律不回推。

[sp]: https://github.com/super-productivity/super-productivity/blob/3093ad617319f7ae8063df08a0fb0f63bfdc0847/electron/idle-time-handler.ts
[ev]: https://github.com/EvoMap/evolver/blob/31b0691acd97ba18878019312e646f1f2d970d43/src/gep/idleScheduler.js
[lb]: https://github.com/Saramanda9988/LunaBox/blob/97d17b0aa8bedc1aebe0cd3ff87188d67d8c7c3a/packages/backend/utils/timerutils/focusing/idle_linux.go

## 日记重心：从「打开了什么」到「学了什么」（2026-11-08 晚）

用户："日记还是不要太奇怪了,着重突出学习了什么,其他事情可以写得少一点"。

改前是五节模板（一句话/学习/建造/卡点/明天切口/时间分布），读起来像审计报告，
而且「学习」那一节写的全是「marktext 停在某个文件上」——窗口列表，不是内容。

改后三节，学习占一半以上篇幅：

```
## 今天学了什么      ← 主体，写主题和内容，不写窗口
## 顺手做的事        ← 最多 4 行
## 明天接着
```

prompt 里直接给了正反例：

- ✗ 「marktext 停在 `backward-induction-逆向归纳法与最优性原理.md`」
- ✓ 「逆向归纳法与最优性原理：在 marktext 里推进这篇笔记，凌晨和上午两段共 40 分钟」

同时删掉了原来那些「不能证明在前台」「素材不足」的免责声明要求 —— 无关窗口直接不写。

### 同日修掉的三个数据 bug

1. **未闭合段的段尾取了整个文件最后一条记录**（`evs[-1]["ts"]`）。机器中午关机、
   晚上再开，一个 02:53 的段就被拉到 23:59 → 日记里出现「跨 438min」。
   改成只用段内的 beat 证明段尾。
2. **扩展可用性只在启动时判一次**。11:51 机器重启后 GNOME 还没加载完扩展，
   `dbus_available()` 失败一次就退回 AT-SPI，一整天都只有「可见窗口」。
   改成不可用时每 2 分钟重试。
3. **`GetIdletime` 失败被当成「没在 idle」**。gdbus 偶发失败 → 立刻写一条 wake，
   下一轮又写 idle 6419，一天 32 条假 wake，段被心跳无限续长（7.3 小时）。
   改成失败时沿用上次读数 + 流逝时间推算，并加一次重试与失败日志。
