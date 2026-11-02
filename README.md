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

## 双机：数据怎么合到一起

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
