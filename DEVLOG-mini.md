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

## 2026-06-25 · [mini] 模型重训（修全量 refit 漏洞）+ 退化判据改近3日滚动

- **任务**：accuracy.csv 显示重训后劣化（近7日均 MAE=124.7、MAPE 48.5%、5/7 天 🔴），判定需重训。
- **发现并修一个 refit 漏洞**：`retrain_model.py` 把 `TEST_START_DATE` 设成 `2026-04-04`，
  保存的却是 `.fit(X_train)`（只训练到 04-03）的模型——**部署模型数据覆盖反比上一版（训到 05-21）更旧**，
  最近闹退化的新工况全被划进测试集、没进训练。test MAE=39.48 是 teacher-forcing（喂真实滞后价）严重低估生产误差，别误信。
  - **修法**：保存前用 `sklearn.base.clone` 复制同套超参，在**全量数据（截至 06-25，12792 行）重拟合**一份生产模型再落盘；
    评估指标仍用 train-only 模型产出，保持诚实。meta 新增 `prod_full_refit`/`prod_train_period`。
  - 重跑后生产模型训练覆盖 `2025-01-08 ~ 2026-06-25`；旧模型备份 `models/v2_backup/20260625_123549`；
    主目录与 API 加载目录 diff 一致；`install_service.sh restart` 已重启，`api_check.sh` 五端点全 200。
- **退化判据改近3日滚动**：`tools/verify_sync.py` 原单日 MAE≥70 判退化，易被离群点（如 06-19 实际价近零致 MAPE=209%）误触发。
  - `_latest_accuracy()` 多返回 `mae3`（近3日含当日 MAE 均值）；`_quality_block()` 改用 `mae3` 判 🔴/🟡/🟢，
    推送文案展示「当日 MAE/MAPE ｜ 近3日均 MAE」。阈值不变（≥70 退化 / 50–70 注意 / <70 正常）。
  - 实测当日输出：`🔴 模型退化  当日 MAE=109.75 ｜ MAPE=19.07% ｜ 近3日均 MAE=93.43`（仍 🔴，因这3天是老模型预测；新模型从明天起计入会回落）。
- **状态**：完成并验证。
- **改动文件**：`retrain_model.py`、`tools/verify_sync.py`（均 `.bak_*` 备份）、`CLAUDE.md`、`DEVLOG-mini.md`；模型 pkl/meta 已更新并同步 API 目录。
- **下一步**：盯 2–3 天 accuracy.csv——回到 MAE<50 说明重训见效；仍频繁 🔴 则非数据陈旧，需查特征或市场结构性变化（如西电断供类外生事件需加特征）。
  Air 端 `verify_sync.py` 若也跑，需同步这版判据改动。

## 2026-06-22 · [mini] 给 loop 补上「会说不的评判器」+ 堵静默失败链（P0–P3）

- **任务**：用 loop-engineering「五动作×六零件」审查整套自动化，发现唯一缺的零件是**验证那格**——
  11:30 的 `verify_sync.py` 只查「文件在不在」，是假评判器；真正的质量对标 `calc_accuracy()`（昨日预测 vs 实际，MAE≥70 判退化）
  存在但**只写日志、不告警**，核对消息也不读它。于是会出现「✅ 同步正常，但预测其实已退化没人知道」。
- **做了什么（P0–P3 四项）**：
  - **P0（核心缺件）** `tools/verify_sync.py`：新增 `_latest_accuracy()`（读 `accuracy.csv` 按日期取最新 MAE/MAPE）
    + `_quality_block()`（MAE≥70 🔴退化 / 50–70 🟡注意 / <70 🟢正常，阈值与 `calc_accuracy` 一致）。
    *（2026-06-25 更新：判据由单日改为近3日滚动均值 MAE，防离群点误触发，见顶部条目。）*
    11:30 核对消息新增一行准确率红绿灯；**同步/预测都正常但 MAE≥70 时，标题从 ✅ 降级为 🔴「同步正常但模型退化」**，并附复查建议。
  - **P1（Critical 静默链）** `update.sh`：新增 `alert()` 函数（复用 `kdocs_sync._feishu_push`）；
    Step 2 API 启动超时 `exit 1` 前先推飞书告警——此前只会收到 Step 1 的「金山同步成功」，预测/推图/备份全被跳过却无人知。
  - **P2** `update.sh`：`predict_archive.py` 原为裸调用、失败静默吞掉，改为失败即 `err`+`alert`（仍继续跑备份，不中断）。
  - **P3** `kdocs_sync.py`：`check_token_expiry()` 在令牌已过期时 `notify_failure()` 告警 + `sys.exit(2)`，
    砍掉「过期后仍轮询到 18:00 才报」的静默窗口。
- **保留人工复核点**：P0 的 🔴 只告警，不自动停 loop、不自动重训——是否因退化干预由人定（执行可外包，拿主意不行）。
- **复用而非新增**：全程复用现成 `_feishu_push`/`notify_failure`/`should_push`/`MACHINE` + `accuracy.csv`，未加新服务/新 launchd job/新依赖。
- **验证**：`update.sh` 过 `bash -n`；`verify_sync.py`/`kdocs_sync.py` 过 `py_compile`；
  **干跑 verify 新逻辑实测**：同一状态下，旧逻辑推「✅ 同步核对正常」，新逻辑推「🔴 同步正常但模型退化  MAE=207.15 元/MWh ｜ MAPE=31.15%」（未真发飞书，避免群内重复）。
- **状态**：完成并验证。
- **改动文件**：`tools/verify_sync.py`、`update.sh`、`kdocs_sync.py`（gdpower 权威副本 + WORK_DIR 冗余副本，diff 一致）。
  备份均为 `.bak_20260622_125644`（三处 + WORK_DIR 一处）。
- **⚠️ Air 端待生效（mini 够不到 Air，请在 Air 本地手动同步）**：
  1. **P3 的 `check_token_expiry`**（机器无关，照搬）：在 Air 的 `~/gdpower/kdocs_sync.py`（若 WORK_DIR 也有副本则一并改）里，
     把 `days_left < 0` 分支末尾补上告警+退出——在 `log.error('=' * 52)`（过期分支最后一行）之后插入：
     ```python
             try:
                 notify_failure('AirScript 令牌已于 ' + str(TOKEN_EXPIRY) +
                                ' 过期，今日数据同步必然失败。请在金山脚本编辑器续期并更新 .secrets.json。')
             except Exception as _e:
                 log.error('过期告警推送失败：' + str(_e))
             sys.exit(2)
     ```
  2. **P0 的 verify_sync 红绿灯 / P1·P2 的 update.sh 告警**：Air 若也跑 `verify_sync.py` 与 `update.sh`，建议一并移植
     （逻辑机器无关，仅路径/用户名 `hydtzyj` 不同）；可从 mini 这三个文件直接参照。
  - 改前先 `cp ...bak_$(date +%Y%m%d_%H%M%S)`，改后 `py_compile` + `diff` 两副本确认一致。
- **关联文档**：完整审查报告与 loop-engineering 框架笔记已存入 Obsidian「电力市场」仓库
  （`gdpower自动化审查-缺会说不的评判器-20260622.md`、`loop-engineering循环工程-框架讲解.md`）。

## 2026-06-20 · [mini] 发图加上传重试，根治偶发掉图

- **任务**：今天群里收到「金山同步成功」文字，但**没收到日前预测折线图**。排查原因并加固。
- **根因**：今天金山数据晚到，轮询到第 11 次 16:43 才成功；紧接着 16:44 `notify_prediction.py` 发图时，
  `open.feishu.cn` 连续 SSLEOFError（换 token + 上传图各失败一次），`_feishu_upload_image` 无重试直接返回 None，
  脚本按设计**降级成纯文本表**发出 → 群里有字无图。20 秒后网络恢复，对比图(16:44:22)反而发成功。纯偶发网络抖动（CLAUDE.md 已记此为已知问题），数据本身正常。
- **当天处置**：手动 `notify_prediction.py --date 2026-06-21` 补发成功（图+表一条 post）。
- **加固（本次改动）**：给 `kdocs_sync.py` 的 `_feishu_upload_image` 加「换 token + 上传」整体重试循环：
  最多 3 轮、退避 5s/15s，每轮重新换 token，全失败才返回 None 降级。一处改动，预测图 + 对比图（共 4 处调用）全部受益。
- **验证**：py_compile 通过；真实调用 `_feishu_upload_image` 正常路径仍秒回 image_key（仅上传不发群）；gdpower 与 WORK_DIR 两份 diff 一致。
- **状态**：完成并验证。
- **改动文件**：`kdocs_sync.py`（mini 端 gdpower + WORK_DIR 双副本，备份 `.bak_20260620_173111`）；
  **部署包 `mini-deploy/kdocs_sync.py`（=Air 版，仅 22-23 行路径不同）已同步打同一补丁**，备份 `.bak_20260620_173822`。
- **⚠️ Air 端待生效（mini 够不到 Air；Air 本地 mini-deploy 部署包副本可能是旧的，勿依赖重拷——请直接替换函数）**：
  在 Air 的 `~/gdpower/kdocs_sync.py`（若 WORK_DIR 也有副本则一并改）里，把整个 `def _feishu_upload_image(...)` 函数替换为下面这版（只加重试，签名/返回值兼容，与机器路径无关、照搬即可）。
  替换前先 `cp ~/gdpower/kdocs_sync.py ~/gdpower/kdocs_sync.py.bak_$(date +%Y%m%d_%H%M%S)`，替换后 `/opt/anaconda3/envs/gdpower/bin/python -m py_compile ~/gdpower/kdocs_sync.py` 验证，再 `diff` 两副本确认一致：

```python
def _feishu_upload_image(png_path, retries=3, backoff=(5, 15)):
    """上传本地 PNG，返回 image_key；未配置应用凭证返回 None。

    偶发 SSL/网络抖动（mini 已知问题：换 token 或上传时 SSLEOFError）下自动重试：
    每轮重新换 token + 上传，失败按 backoff 退避后再试，最多 retries 轮，
    全失败才返回 None（调用方再降级纯文本表）。"""
    for attempt in range(1, retries + 1):
        token = _feishu_tenant_token()
        if token:
            try:
                with open(png_path, 'rb') as f:
                    r = _SESSION.post(FEISHU_OPEN_BASE + '/im/v1/images',
                                      headers={'Authorization': 'Bearer ' + token},
                                      data={'image_type': 'message'},
                                      files={'image': f},
                                      timeout=20)
                j = r.json()
                if j.get('code') == 0:
                    return j.get('data', {}).get('image_key')
                log.error('上传图片失败：' + str(j))
            except Exception as e:
                log.error('上传图片异常：' + str(e))
        if attempt < retries:
            wait = backoff[min(attempt - 1, len(backoff) - 1)]
            log.warning('上传图片第 %d/%d 次未成功，%ds 后重试...' % (attempt, retries, wait))
            time.sleep(wait)
    log.error('上传图片重试 %d 次仍失败，放弃（将降级纯文本）' % retries)
    return None
```
  （依赖 `time` 模块——kdocs_sync.py 已 import，无需新增。）
- **下一步**：① 观察明天 11:10 mini 定时跑是否正常带图；② Air 端重拷 kdocs_sync.py 让重试生效（见上）；③ 可选：把重试思路用到 `_feishu_tenant_token` 单独调用处（读群 fail-open 不致命，优先级低）。
- **坑/备注**：① 重试只兜「上传/换 token」的网络抖动，webhook 文字推送(`_feishu_send`)未含此重试；② 探针技巧：上传图只拿 image_key 不发群，可当飞书发图连通性探针，不污染群。

## 2026-06-16 · [mini] 双机主备去重 + 自动故障切换（mini 转 primary，全功能上线）

- **任务**：mini 部署预测推送后会和 Air 推同一飞书群、一部手机收两条；要求「互为主备 + 一台挂了另一台自动顶上」
- **方案**：`kdocs_sync.py` 新增 `NOTIFY_ROLE`(primary|backup|silent) + `FEISHU_CHAT_ID` + 三函数
  `_feishu_today_message_contents`/`already_notified_today`/`should_push`。primary 总推；backup 用自建应用读「今天本群消息」，
  已有该类推送就跳过、没有才顶上；查不到群历史 fail-open 照推。marker 机器标签无关：
  `金山同步成功`/`次日日前电价预测`/`预测 vs 实际`/`同步核对`
- **角色**：**mini=primary（11:10/11:30 先跑）、Air=backup（需配 ~12:00/~12:20 晚跑）**。mini 是台式机常年在线，更适合当主
- **做了（mini 侧，已端到端验证）**：
  - 部署新版 `kdocs_sync.py` 到 **gdpower + WORK_DIR 两处**；新增 `tools/notify_prediction.py`+`notify_compare.py`（mini 原来没有）
  - **双目录修复**：`update.sh` Step1 改跑 `$GDPOWER_DIR/kdocs_sync.py`；predict_archive 后加 Step3.5 推预测图+对比图；保留 Step4 备份
  - `tools/verify_sync.py` 加 `should_push('同步核对')` 守卫
  - `.secrets.json` 补全：`AIRSCRIPT_TOKEN`(旧硬编码迁入) + `FEISHU_APP_ID/SECRET`(Air 自建应用 `cli_aaac0...`，两机共用) + `FEISHU_CHAT_ID=oc_ebc411bd468de48d788b048d541b8daf` + `NOTIFY_ROLE=primary`
  - 飞书后台：自建应用「周艺军的飞书 CLI」加入群 + 开 `im:message.group_msg` 读权限（之前只做图片上传、不在群里）
  - 验证：py_compile 全过；预测图/对比图 --dry-run 正常出图；**模拟 backup 角色真实读群，4 marker 全部「已存在→跳过」**
- **状态**：mini（主机）完成并验证。
- **追加（2026-06-17 真发确认时发现并修复）**：
  - **换飞书应用**：旧 `cli_aaac0...` 没有发图权限（`im:resource:upload`）且在另一账号。换成 Air 同款新应用 **`cli_aabbd2a70038dbd8`**（群所属账号下、已入群、读群+发图齐全）。只改 `.secrets.json` 两键。
  - **绕系统代理**：`kdocs_sync.py` 飞书请求原走系统代理 `127.0.0.1:1082`（偶发 503 → 掉图/推送失败）。新增 `_SESSION`（`requests.Session()`+`trust_env=False`），4 处飞书调用（换token/上传图/读群/webhook）全改走它。
  - **真发确认**：mini 实发预测图+对比图，群里 `[mini]` 两条**带真图**（分析图+表格图 / 折线图+表格图）一条 post 成功。
- **Air 侧**：Air（claude.ai 端）已自建新应用并切换、实测 backup 去重生效（其有自己一套 dedup 实现，本机这套是平行实现，靠同一群+同标题互相识别，兼容）。
- **改动文件**：`kdocs_sync.py`(双副本)、`tools/notify_prediction.py`/`notify_compare.py`(新增)、`update.sh`、`tools/verify_sync.py`、`.secrets.json`、`CLAUDE.md`；部署包新增 `AIR_备机配置.md`/`find_chat_id.py`
- **下一步**：① Air 拷 3 文件 + 改 verify_sync + secrets 加 backup/chat_id + 定时挪到 ~12:00；② 可选从 mini 真发一次确认
- **坑/备注**：① 自定义机器人 webhook 只能发不能读，去重「读群」必须靠自建应用（两个不同身份）；② 自建应用读群需 `im:message.group_msg` + 在群里 + chat_id，少一样就 `230027` 报错；③ 自建应用是云端实体、不绑机器，两机共用同一 App ID/Secret；④ 备份在 `_dedup_deploy_backup_20260616_193155/` 与 `.secrets.json.bak_*`

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
- **状态**：完成。内置盘 **91G → 108G**（腾出 17G，含后续清掉 4.7G 重型缓存）；外接盘存了归档 ~8G + gdpower 备份
- **改动文件**：`tools/backup_to_external.sh`(新建)、`update.sh`(加 Step4)、`CLAUDE.md`(每日流程/备份/手动操作)
- **下一步**：建议补 Time Machine（gdpower 已专项备份，但整机其它数据仍无备份）
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
