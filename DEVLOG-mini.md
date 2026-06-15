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

## 2026-06-15 · [mini] 外接 500G 盘：内置瘦身 + 扩展存储 + gdpower 专项备份

- **任务**：mini 内置 256G 仅剩 91G，外接 500G NVMe（WD SN770，雷雳/USB4，读写 ~3400/3050），做三件事
- **做了**：
  - **gdpower 专项备份**：新建 `tools/backup_to_external.sh`，打包 `~/gdpower`+`~/Documents/AI应用系统`
    → `/Volumes/mini外置/gdpower_backups/`（保留 30 份、盘不在则跳过）；接入 `update.sh` 作为 Step4 每日自动跑。
    首份备份 39M 已验证（含两目录）
  - **扩展存储**：把 `Downloads/01_安装包与应用`(2.9G)、wechat dmg、`Documents/电力交易`(4.1G)、`Documents/资料`(1.2G)
    移到 `/Volumes/mini外置/归档/`
  - **清缓存**：清 Google/Edge/Lark/微信开发者工具 缓存 + `~/.cache/uv,prisma` + npm + conda clean
- **状态**：完成。内置盘 **91G → 103G**（腾出 12G）；外接盘存了归档 ~8G + gdpower 备份
- **改动文件**：`tools/backup_to_external.sh`(新建)、`update.sh`(加 Step4)、`CLAUDE.md`(每日流程/备份/手动操作)
- **下一步**：还有 ~4.7G「清掉会触发重下」的缓存（playwright/codex-runtimes/autoclaw/camoufox）按需再清；建议补 Time Machine
- **坑/备注**：① 跨盘 `mv` 遇 **`uchg` 不可变标志**文件删不掉源（电力交易里 446 个旧 PDF 带此标志，
  从 U 盘拷来的）——需 `chflags -R nouchg` 解标志才能删，删前已逐文件核对外接副本完整(446/446)；
  ② 移动个人数据到外接后，**只在外接盘插着时可访问**；③ gdpower 生产数据(`Documents/AI应用系统`)绝不可移动

---

## 2026-06-15 · [mini] 修复迁移漏洞（改错文件）+ 飞书告警加固 + 今日补救

- **任务**：核对 11:30 首跑结果时，发现上午的同步加固迁移其实没生效，定位并修复
- **做了**：
  - **定位主因（改错文件）**：`update.sh` 实际跑的是 `模型训练/kdocs_sync.py`（WORK_DIR 副本），
    而迁移只改了 `~/gdpower/kdocs_sync.py`——两个不同文件。所以 11:10 生产跑的仍是**旧代码**：
    金山瞬时 500 → 单次「获取数据失败」→ 放弃，**没有轮询**（新轮询设计形同未部署）
  - **修复**：确认旧生产版 = 编辑前原始版（同源）后，把新版部署到 WORK_DIR（旧版已备份 .bak），
    编译通过、两副本 diff 一致
  - **飞书告警加固**：`notify_failure` / `verify_sync.feishu()` 改为 **重试×3 + `trust_env=False` 直连**，
    绕过系统代理 `127.0.0.1:1082`（11:30 核对消息就是被该代理瞬时 503 吞掉、群没收到）
  - **今日补救**：手动补跑同步（金山恢复，+24 行，CSV→06/15）→ 跑 update.sh（同步秒成→API重载→
    生成 06/16 预测 avg=421）→ 跑 verify_sync 把 `[mini] ✅` 核对结论成功推到群
- **状态**：已修复并端到端验证（update.sh→WORK_DIR 新版→轮询 SUCCESS→exit0）；今日已完整恢复
- **改动文件**：`模型训练/kdocs_sync.py`(部署新版)、`~/gdpower/kdocs_sync.py`(告警加固)、
  `tools/verify_sync.py`(告警加固)、`CLAUDE.md`(补双副本/代理两条陷阱)
- **下一步**：观察明天 11:10 自动轮询 + 11:30 核对是否按预期；可考虑把 WORK 副本改成指向 ~/gdpower 的软链以根治双副本
- **坑/备注**：① **kdocs_sync.py 有两个副本，launchd 跑 WORK_DIR 那个**，改动必须同步两处；
  ② 系统代理 127.0.0.1:1082 偶发 503，飞书已直连绕过，金山靠轮询兜底；③ 验证脚本时务必确认跑的是
  生产真正执行的那个文件，否则验证了个寂寞

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
