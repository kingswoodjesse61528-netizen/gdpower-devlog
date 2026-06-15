# DEVLOG — gdpower 开发日志

> 倒序排列，最新在最上面。每次 Claude Code 开发会话结束时，在「最新」下方追加一条。
> 这份文件供 claude.ai 对话端读取，用来掌握开发进展（配合 CLAUDE.md 一起看）。
>
> 每条格式：日期 · 任务 / 做了什么 / 状态 / 改动文件 / 下一步 / 坑

---

## 最新

<!-- Claude Code：新记录加在这条下面 -->

---

## 2026-06-16 · 更新 mini 部署包至最新

- **做了**：把 `~/Downloads/mini-deploy/` 部署包刷新到 Air 当前全部能力——刷新 kdocs_sync.py / notify_prediction.py / update.sh / sync_devlog.sh，新增 notify_compare.py；`01_deploy.sh` 的备份/部署/路径替换(/Users/hydtzyj→$HOME)/编译检查均纳入 notify_compare.py；README 补「5 项功能」与两类推送验证命令。
- **包覆盖能力**：①金山同步成功通知 ②双目录修复 ③日前预测推送(分析图+预测表格图) ④预测vs实际对比推送(对比图+对比表格图) ⑤sync_devlog rebase 修复。
- **状态**：✅ 两脚本语法检查通过；确认包内无真实密钥（仅 .example）。⏳ 待在 mini 执行（00_diagnose→01_deploy→手动补 .secrets.json/换 update.sh→不动定时）。
- **改动文件**：无（仅更新 ~/Downloads/mini-deploy 部署包，未动 gdpower 生产文件）
---

## 2026-06-15 · 日前预测数据也改用「表格图」发飞书

- **背景**：对齐对比图的表格化，日前预测推送的 24 点数据也从文字行改为表格图片。
- **做了**：`notify_prediction.py` 新增 `_make_table_image()`（24 点表，左右两栏×12 行：时点/预测/风险；表头电力蓝、斑马纹、风险列高/中/负着色；emoji 用 `_han()` 剥离避免豆腐块；底部均价/峰谷/价差/波动率摘要）；`main()` 改为发**分析折线图 + 表格图**两张内联图（一条 post），正文留摘要。复用上次给 `_feishu_push_post` 加的多图能力。
- **状态**：✅ py_compile 通过；dry-run 表格图肉眼确认清晰（emoji 正常剥离）；真实推送飞书「分析图+表格图 一条 post」成功。
- **改动文件**：`tools/notify_prediction.py`（已备份）；新增产物 `predictions/chart_table_*.png`
- **下一步**：随其它改动一并部署到 mini
---

## 2026-06-15 · 对比数据改用「表格图」发飞书（更清晰）

- **背景**：对比 post 里 24 点「预测/实际/偏差」原是纯文字行，不够清晰。改成渲染成表格图片发飞书。
- **做了**：
  - `notify_compare.py` 新增 `_make_table_image()`：matplotlib 渲染 24 点数据表（左右两栏×12 行，列＝时点/预测/实际/偏差；表头电力蓝、斑马纹、偏差列红绿着色＝高估红/低估绿；底部 MAE/MAPE/评级摘要）。
  - `main()` 改为同时发**折线图 + 表格图**两张内联图（一条 post），正文只留简短摘要，去掉冗长文字行。
  - `kdocs_sync.py:_feishu_push_post()` 支持 `image_key` 传 list（多图内联，向后兼容），改前已备份。
- **状态**：✅ py_compile 通过；dry-run 出表格图肉眼确认清晰；真实推送飞书「折线图+表格图 一条 post」成功。
- **改动文件**：`tools/notify_compare.py`、`kdocs_sync.py`（已备份）；新增产物 `predictions/compare_table_*.png`
- **下一步**：随其它改动一并部署到 mini
---

## 2026-06-15 · 新功能：当天「日前预测 vs 实际」对比图推送飞书

- **背景**：要做「当天预测 vs 实时电价」对比图。排查发现 `实时电价`(ss_jia_ge) 自 ~2026/4 起断更全为 0，无数据；`日前电价` 实际值每日完整、且正是模型预测对象。经确认改为对比「日前预测 vs 日前实际」（＝当天预测准确率对比），当天 11:00 同步后生成。
- **做了**：
  - 新建 `tools/notify_compare.py`：读 prediction_{date}.csv 的 pred_price + CSV 当日 `日前电价` 24 点；算 MAE/MAPE（沿用 calc_accuracy 口径 abs>50）/最大偏差@时点/预测均价 vs 实际均价/整体高估低估/评级；出双线对比图（实际绿+预测蓝+红色偏差阴影+最大偏差标注），右栏指标卡，24 点「预测/实际/偏差」表。**复用 notify_prediction 的配色/BANDS/_draw_card/飞书发送（import 复用，不改原文件）**。
  - `update.sh` Step 3 成功分支加钩子：`notify_compare.py --date $(date +%Y-%m-%d)`（失败不阻塞主流程，改前备份）。
- **状态**：✅ py_compile + update.sh 语法通过；dry-run 出图口径校验通过（图 MAE 97.2/MAPE 17.1%/最大偏差 332.0 ＝ accuracy.csv 6/15 的 97.18/17.15/332.02）；真实推送飞书图文 post 成功。
- **改动文件**：新增 `tools/notify_compare.py`；改 `update.sh`（已备份）
- **下一步**：随其它改动一并部署到 mini（更新部署包）
- **坑/备注**：实时电价数据源断更（ss_jia_ge 全 0），真·实时对比需先修数据源，另起任务；对比图数据不全/实际全 0 会自动跳过不发空图
---

## 2026-06-15 · 补 6/14 缺失的日前预测 + 回填准确率

- **背景**：发现系统缺 6/14 数据。排查结论：6/14 实际电价 24 点齐全；缺的是 6/14 的「日前预测」存档及准确率。根因——6/14 危机日（金山 AirScript 频繁 500/403 + 正改 launchd 定时 14:00→11:00），原定预测任务没跑成；当天 17:58 首次成功出预测时数据已推进，目标变成 6/15，6/14 被整档跳过（6/13 预测 6/13、6/14 预测 6/15）。
- **做了**：一次性脚本补档——临时把两份 CSV 截断到 6/13 23:00 → 现代 launchctl 重启 API（pred_date=6/14）→ /predict 存档 `prediction_2026-06-14.csv`（均价 389.0）→ finally 必还原完整 CSV 重启；再用还原后完整 CSV 算 6/14 准确率并按日期有序写回 accuracy.csv。复用了 backfill_batch.py 的截断/还原思路，但重启改用现代 launchctl。
- **状态**：✅ prediction_2026-06-14.csv（24点）生成；accuracy 6/14=MAE 35.29/MAPE 8.07%（与 6/13 同档）；主 CSV 完整无损、API 回正常（data_date=6/15）；临时文件已清。改前备份 accuracy.csv。
- **改动文件**：新增 `predictions/prediction_2026-06-14.csv`、`accuracy.csv` 增 6/14 行（已备份）
- **坑/备注**：补的是事后日前预测（输入用 /defaults 基于 6/13 数据重建，非真·实时日前），仅为填缺口；危机期数据跳变会导致某目标日预测被整档跳过，值得注意
---

## 2026-06-15 · 生成 mini 部署包（待在 mini 执行）

- **背景**：成功通知 + 双目录修复 + 现货分析图推送目前只在 Air 生效；mini 需同步。Air 会话碰不到 mini 文件系统，故离线打包。
- **做了**：在 Air 生成部署包 `~/Downloads/mini-deploy/`（含 `00_diagnose.sh` 只读诊断、`01_deploy.sh` 半自动部署、`kdocs_sync.py`/`notify_prediction.py`/`sync_devlog.sh`/`update.sh` 权威新版、`.secrets.json.example`）。脚本自动：备份→部署→把写死的 `/Users/hydtzyj` 替换为本机 HOME→查依赖→py_compile→dry-run 出图；生成 `update.sh.reference` + diff 供对照。
- **关键前提（探明）**：gdpower 无 git 远程（须 AirDrop 传文件）；代码写死 `/Users/hydtzyj`（mini 用户名不同须替换路径，脚本已处理）。
- **状态**：✅ 包已生成、语法检查通过、确认不含真实密钥。⏳ 待在 mini 上执行（先 00_diagnose 再 01_deploy）。
- **改动文件**：无（仅在 Air 生成 ~/Downloads/mini-deploy 部署包，未改 gdpower 生产文件）
- **下一步（mini 端手动）**：① 补 `.secrets.json`（mini 自己的 AIRSCRIPT_TOKEN + 从 Air 复制 FEISHU_APP_ID/SECRET）② 用 update.sh.reference 替换（确认双目录修复+推送钩子）③ 不改 launchd 定时 ④ 真实发一次验证 [mini] 标签图
- **坑/备注**：mini 旧版 token 为硬编码，须从备份旧 kdocs_sync.py 提取填进 .secrets.json；飞书 app 凭证两机通用
---

## 2026-06-15 · 修复交易提示框「低价窗口」溢出

- **背景**：右栏黄色「交易提示」框 6 行文字行距过松，末行「低价窗口」y 坐标算到框底以下，溢出框外。
- **做了**：`tools/notify_prediction.py` 收紧框内行距（波动率→交易提示 0.07→0.058、各提示行 0.052→0.045、首行偏移 0.04→0.045），框底由 0.02 下探到 0.015。
- **状态**：✅ dry-run 确认六行全部收进黄框；真实推送飞书图文一条 post 成功。
- **改动文件**：`tools/notify_prediction.py`
- **下一步**：mini 同步部署（需在 mini 操作，路径/凭证/定时按本机保留）。

## 2026-06-15 · 现货分析图微调

- **做了**：`tools/notify_prediction.py` 出图微调——去掉 x 轴「时刻」标签、去掉底栏「数据来源…」整行、把「谷」标注气泡挪近对应谷点（xytext 偏移由 -2.6/-30 收到 -1.0/-24）。
- **状态**：✅ dry-run 确认 + 真实推送飞书图文一条 post 成功。
- **改动文件**：`tools/notify_prediction.py`
- **下一步**：mini 同步部署（成功通知 + 双目录修复 + 现货分析图推送）——需在 mini 上操作，注意路径/凭证/定时按本机保留。

## 2026-06-15 · 预测推送图升级为「现货分析仪表盘」

- **背景**：原 matplotlib 折线图较朴素；用户给出《折线图设计规范》+ 参考图，要求升级为现货分析风格仪表盘。
- **做了**：重写 `tools/notify_prediction.py` 出图逻辑（改前已备份），遵循规范——
  - 电力蓝 `#2563eb` 主线 + 极浅蓝填充；均价琥珀虚线 `#f59e0b`
  - 双栏布局：左折线（2/3）+ 右「核心指标」卡（1/3）
  - 时段着色：基荷/早峰/午谷/腰荷/晚峰/夜间回落（淡蓝/淡黄/淡绿/淡粉）+ 竖向渐变底
  - 峰(红)/谷(绿)气泡标注、24 点点位数值标注、底栏数据来源
  - 右栏指标卡：均价/最高@时刻/最低@时刻/峰谷价差/峰谷比 + 波动率(σ) + 交易提示（曲线判断/价差判断/重点关注/低价窗口，规则启发式）
  - 关于「调用 skill 出图」：skill 无法被 launchd 无人值守任务直接调用（skill 是给 agent 的指令、非可执行程序），故把规范固化进脚本，自动管线直接调
- **状态**：✅ py_compile 通过；dry-run 出图肉眼确认还原参考图；真实推送飞书图文一条 post 成功
- **改动文件**：`tools/notify_prediction.py`（已备份）
- **下一步**：用户确认飞书内联效果；mini 同步部署
- **坑/备注**：matplotlib 还原 HTML 风设计已足够「参考」级；如需像素级 HTML 质感需改 headless 渲染（重、依赖浏览器，不利无人值守，暂不采用）

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
