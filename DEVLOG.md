# DEVLOG — gdpower 开发日志

> 倒序排列，最新在最上面。每次 Claude Code 开发会话结束时，在「最新」下方追加一条。
> 这份文件供 claude.ai 对话端读取，用来掌握开发进展（配合 CLAUDE.md 一起看）。
>
> 每条格式：日期 · 任务 / 做了什么 / 状态 / 改动文件 / 下一步 / 坑

---

## 最新

<!-- Claude Code：新记录加在这条下面 -->

---

## 2026-06-15 · 新功能：预测成功后推送折线图+24点表到飞书

- **背景**：预测结果静默存盘，用户看不到直观结果。需预测成功后自动出图+表发飞书。
- **做了**：
  - `kdocs_sync.py`：新增飞书自建应用能力——`_feishu_tenant_token()`/`_feishu_upload_image()`（上传图拿 image_key）/`_feishu_push_image()`/`_feishu_push_post()`（post 富文本，可内嵌图），底层抽 `_feishu_send()` 复用（`_feishu_push` 也改为复用，行为不变）；新增可选凭证 `FEISHU_APP_ID`/`FEISHU_APP_SECRET`（缺失则发图降级、不崩）
  - 新建 `tools/notify_prediction.py`：读 `predictions/prediction_{date}.csv`，matplotlib 出折线图（预测线+置信区间阴影+峰谷标注+均价线，PingFang HK 中文字体），存 `predictions/chart_{date}.png`；发飞书优先「一条 post：图+24点表」，无凭证/失败降级为纯文本表；`--dry-run` 只出图
  - `update.sh` Step 3 预测成功分支挂钩子调用（失败不阻塞主流程）
  - `.secrets.json.example` 补 `FEISHU_APP_ID`/`FEISHU_APP_SECRET` 说明
  - 改前已备份 `kdocs_sync.py`/`update.sh`
- **状态**：✅ **全链路联调成功**。用户已配飞书自建应用凭证（写入 .secrets.json，已备份），真实跑通：换 token→上传图拿 image_key→发「一条 post：内嵌折线图 + 24 点表 + 峰谷摘要」，飞书端肉眼确认图内联渲染正常（无需降级到两条消息）
- **改动文件**：`kdocs_sync.py`、`tools/notify_prediction.py`(新)、`update.sh`、`.secrets.json.example`（均已备份/或新增）；`.secrets.json` 增 FEISHU_APP_ID/SECRET（已备份）
- **下一步**：mini 同步部署（kdocs_sync 新函数+notify_prediction.py+update.sh 钩子+app 凭证）；可考虑与 verify_sync.py 11:20 摘要合并去重
- **坑/备注**：webhook 发不了图，必须用应用凭证上传拿 image_key；post 文案需含「广东电力」命中关键词；自定义机器人 post 内嵌 img 若实测不显示则代码已有「图+表两条」降级分支

## 2026-06-15 · 金山同步加「成功通知」+ 当日轮询实战验证

- **背景**：6/14 加固后只在「轮询至 18:00 仍全失败」才发飞书告警，成功是静默的；用户希望成功也报喜，每天有个正向确认。
- **做了**：
  - `kdocs_sync.py`：抽出共用推送 helper `_feishu_push(text)`，`notify_failure` 改用它（行为不变）；新增 `notify_success(msg)`；在 `poll_and_sync` 成功分支调用，消息含 机器标签/日期/第几次轮询成功/CSV 最新日期；文案含「广东电力」「金山同步」以命中机器人关键词
  - 改前已备份 `kdocs_sync.py.bak_20260615_212948`
  - `sync_devlog.sh`：push 前加 `git pull --rebase origin main`（修两机共用同一 Gist 仓库导致 push 被拒），含冲突自动 abort 保护
- **状态**：✅ `py_compile` 通过；真实发送测试成功通知返回「已推送」，全链路验证通过。当日轮询实战：11:00 第1次(金山500)、11:30 第2次(403) 均正确判 FETCH_FAIL 重试，12:01 第3次成功（新增24行/共12720行）——exit0 假成功 bug 确认已修，系统自愈、无需人工
- **改动文件**：`kdocs_sync.py`、`tools/sync_devlog.sh`（kdocs 已备份）
- **下一步**：mini 仍跑旧版 kdocs_sync.py，需把「双目录修复 + 三态轮询 + 成功通知」一并部署过去（保留本机 token/路径/定时）
- **坑/备注**：成功通知文案必须含机器人关键词否则被拦；成功通知只在 `poll_and_sync` 生产路径触发，`--once`/`--schedule` 路径未加；mini 改完前不会有成功通知

---

## 2026-06-15 · 双目录陷阱核对（Air）— 生产已跑新版，无需修复

- **背景**：mini 上发现 `update.sh` 实际执行的是「模型训练/ 目录的 WORK_DIR 副本」而非 6/14 改的 `~/gdpower/kdocs_sync.py`，导致生产一直跑旧代码、加固形同未部署（CLAUDE.md「双目录陷阱」）。怀疑 Air 同病。
- **做了**：只诊断、未改任何文件。核对链路——
  - launchd `com.gdpower.update.plist` → ProgramArguments 跑 `~/gdpower/update.sh`，WorkingDirectory=`~/gdpower`，触发 11:00
  - `~/gdpower/update.sh`：`GDPOWER_DIR=~/gdpower`，第 48/50 行执行 `$GDPOWER_DIR/kdocs_sync.py`（即 `~/gdpower/kdocs_sync.py`）
  - 该文件含全套新版标记：`SyncResult(Enum)`、`poll_and_sync()`、`notify_failure()`、fetch 失败 `return None`、`sys.exit(0 if res is SyncResult.SUCCESS else 1)`
  - 旧版仅存归档副本 `模型训练/_archived_code_20260527/kdocs_sync.py`（0 新版标记，fetch 失败 `return []`），无脚本引用；`模型训练/kdocs_sync.py` 活跃副本不存在
- **状态**：✅ Air 生产实际运行的就是 6/14 加固后的新版，三态/轮询/exit 码修复均已生效。**无需修复**。
- **改动文件**：无（kdocs_sync.py 一行未动）
- **下一步**：mini 按同法修复（让其 update.sh 改跑 `$GDPOWER_DIR/kdocs_sync.py`，保留本机 token/路径/定时）
- **坑/备注**：Air 之所以没踩坑，是因为它的 `update.sh` 早已改成「cd 到 WORK_DIR 但运行 GDPOWER_DIR 副本」；`~/Downloads/update.sh` 是游离旧副本（跑 WORK_DIR 副本），launchd 不引用它，勿混淆

---

## 2026-06-14 · P0 同步加固 + P1 飞书告警 + mini 迁移准备

- **任务**：P0 金山同步加固、P1 飞书告警、每日核对、Mac mini 迁移准备
- **做了**：
  - **P0 金山同步加固**：`kdocs_sync.py` 升 v6.0 检测式轮询（11:00 起反复拉取，拿到当日数据为止，最晚 18:00）；修掉 `exit 0` 假成功 bug（这才是半夜反复补数据的真正根因）；`com.gdpower.update.plist` 定时 14:00 → 11:00
  - **P1 飞书告警**：`notify_failure()` 接飞书自定义机器人 webhook，实测推送成功（关键词＝广东电力）
  - **每日核对**：`verify_sync.py` 每天 11:20 核对首跑结果，并把次日预测摘要推送到飞书
  - **Mac mini 迁移准备**：日志/脚本加机器标签 `[Air]`/`[mini]`，生成 `MINI_MIGRATION.md`（待在 mini 上执行）
- **状态**：P0/P1/核对均已上线，Air 端实测通过；mini 迁移待执行；自动任务待 6/15 起观察
- **改动文件**：`kdocs_sync.py`(v6.0)、`com.gdpower.update.plist`、`verify_sync.py`、`MINI_MIGRATION.md`（脚本均已备份）
- **下一步**：6/15 起观察 11:00 检测式轮询与飞书告警是否按预期触发；择机在 mini 上跑迁移
- **坑/备注**：`exit 0` 假成功会掩盖失败、让重试形同虚设——半夜补数据的真因；改 plist 后需 `launchctl bootout/bootstrap` 重载才生效；飞书机器人必须带约定关键词否则被拦

---

## 2026-06-14 · 危机抢修收尾 + 起步开发化

- **背景**：6/6–6/12 因金山 AirScript 14:00 频繁 500，连续多次半夜手动救火
- **做了**：
  - 两台 Mac 均上线新模型（含 5 月退化期训练），`update.sh`/`launch_api.sh` 改用现代 launchctl 命令
  - 补齐 6 月预测缺口，accuracy.csv 排序
  - 建立 CLAUDE.md（项目上下文）+ 本 DEVLOG
- **状态**：系统稳定运行，新模型 7 天均 MAE ≈ 53，最近 6/13 = 37.46（向好）
- **下一步**：P0 金山同步加固（最紧急）
- **坑/备注**：见 CLAUDE.md「已知陷阱」八条

---

<!-- 历史记录继续往下追加 -->
