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

<!-- 历史记录继续往下追加 -->
