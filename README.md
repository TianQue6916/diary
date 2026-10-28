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


## GitHub 同步

公开仓库：**https://github.com/TianQue6916/riji**（自己推，每天一次）

- 推送由 `run_daily.py` 末尾调 `sync_git.py` 完成，走的 SSH（`~/.ssh/id_ed25519`，已验证免交互）。
- **只公开** `days/*.md`、`index.md`、`README.md`、`.gitignore`。
  `activity/`（每 10 s 的前台窗口原始记录）与 `raw/`（含完整 prompt 原文的素材快照）
  写在 `.gitignore` 里，永不外传。
- 离线也不会丢：commit 留在本地，下次运行自动补推，所以最多「晚一天」不会「缺一天」。
- 想改回私有：`curl -X PATCH -H "Authorization: Bearer $TOK"   https://api.github.com/repos/TianQue6916/riji -d '{"private":true}'`，
  或在网页 Settings 里一键切换。

> 注意：这是公开仓库，日记正文里会出现你的文件路径、窗口标题、你发给 agent 的指令原话。
> 这些内容在生成时没有脱敏（脱敏会破坏「可回溯」这个前提）。若哪天不想公开了，转私有即可。
