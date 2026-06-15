# DEVLOG (mini) — gdpower 开发日志

> **本文件由 Mac mini (zhouyijun) 维护，记录标 `[mini]`。** 与 Air 的 `DEVLOG.md` 共用同一仓库，
> 靠文件名隔离、靠 `[mini]`/`[Air]` 标签区分，互不覆盖。
> 倒序排列，最新在最上面。每次 Claude Code 开发会话结束时，在「最新」下方追加一条。
> 这份文件供 claude.ai 对话端读取，用来掌握 mini 端开发进展（配合 CLAUDE.md 一起看）。
>
> 每条格式：日期 · 任务 / 做了什么 / 状态 / 改动文件 / 下一步 / 坑

---

## 最新

<!-- Claude Code：新记录加在这条下面 -->

---

## 2026-06-15 · [mini] 接入 chat 协作模式

- **任务**：`[mini]` 接入与 Air 同一套 Claude Code + chat 协作模式，共用 `gdpower-devlog` 仓库
- **做了**：
  - clone 共用仓库 `kingswoodjesse61528-netizen/gdpower-devlog` 到 `~/gdpower-devlog`
  - 新建本机独立日志 `~/gdpower/DEVLOG-mini.md`（与 Air 的 `DEVLOG.md` 同仓不同名，避免冲突）
  - 新建同步脚本 `~/gdpower/tools/sync_devlog.sh`（先 `git pull` 再推本机日志文件）
  - 在 `~/gdpower/CLAUDE.md` 新增「工作约定」：会话结束更新本文件并跑同步脚本
- **状态**：已接入共用仓库 gdpower-devlog
- **改动文件**：`DEVLOG-mini.md`(新建)、`tools/sync_devlog.sh`(新建)、`CLAUDE.md`(追加工作约定)
- **下一步**：以后每次开发会话结束，更新本文件后运行 `bash ~/gdpower/tools/sync_devlog.sh` 同步
- **坑/备注**：公开仓库只放 DEVLOG 类日志文件，绝不放任何 gdpower 代码或密钥（webhook/token）；
  `~/gdpower` 本身不是 git 仓库，同步脚本只 `cp` 单个 md 文件，天然杜绝代码外泄

---

## 2026-06-15 · [mini] 同步加固迁移（从 Air 搬迁）

- **任务**：把 Air 2026-06-14 做的金山同步加固迁移到 mini（照 `MINI_MIGRATION.md` 逐步执行并验证）
- **做了**：
  - **检测式轮询**：`kdocs_sync.py` 默认入口改为轮询同步——11:10 由 launchd 触发后进入循环，每 30min 一轮，以「本地 CSV 是否已含当日数据」为成功判据，拿到即退出，最晚试到 18:00
  - **修 exit0 假成功 bug**：引入 `SyncResult` 三态（SUCCESS / NO_TODAY_DATA / FETCH_FAIL），`fetch_from_kdocs` 失败由 `return []` 改 `return None`；默认入口按末态 `sys.exit(0/1)`，`update.sh` 据此感知真实成败
  - **飞书告警**：`notify_failure()` 在 18:00 仍失败时推送（webhook 走 .secrets.json，缺失则仅落日志）；消息自动带 `[mini]` 标签
  - **每日核对**：新增 `tools/verify_sync.py` + `com.gdpower.verify.plist`，11:30 核对当日同步并把结论 + 次日预测摘要推飞书
  - **定时调整**：`com.gdpower.update.plist` 13:00 → 11:10（错峰 Air 的 11:00）
  - 适配本机差异：mini 的 AirScript token 是硬编码、原无 .secrets.json，故飞书配置改为「可选读取，缺失降级」，token 不动
- **状态**：已上线，本机实测通过（编译/连通/三态分类/失败退出码契约/飞书连通均✓）；两个 launchd 触发时刻已核对（11:10 / 11:30）
- **改动文件**：`kdocs_sync.py`、`tools/verify_sync.py`(新建)、`com.gdpower.update.plist`、`com.gdpower.verify.plist`(新建)、`.secrets.json`(新建,仅飞书)、`CLAUDE.md`(新建)（脚本/plist 均已备份 .bak）
- **下一步**：观察 6/15 起 11:10 轮询同步与 11:30 核对是否按预期触发
- **坑/备注**：旧逻辑「有任意行就算成功」会把「金山还没出当日数据」误判成功——是半夜反复补数据的真因；改 plist 后须 `launchctl bootout/bootstrap` 重载；金山 webhook 偶发瞬时 500，`--test` 失败先重试两次

---

<!-- 历史记录继续往下追加 -->
