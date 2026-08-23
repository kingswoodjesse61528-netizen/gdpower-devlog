# DEVLOG — gdpower 开发日志

> 倒序排列，最新在最上面。每次 Claude Code 开发会话结束时，在「最新」下方追加一条。
> 这份文件供 claude.ai 对话端读取，用来掌握开发进展（配合 CLAUDE.md 一起看）。
>
> 每条格式：日期 · 任务 / 做了什么 / 状态 / 改动文件 / 下一步 / 坑

---

## 最新

<!-- Claude Code：新记录加在这条下面 -->

## 2026-08-23（七）· ⚠ 同日修复：次日边界同步被短路吃掉，全天只有一次机会

- **起因**：用户确认「次日数据每天 11 点左右发布」。核对触发时刻发现 `update.plist` 实际是 **12:00 首跑 + 13:00-17:00 每 30 分钟补跑**（CLAUDE.md 陷阱4 写的 11:00 已过时）。11 点发布 vs 12:00 首跑，看似有 1 小时余量，但顺手核对 `run_sync()` 代码时发现一个当天自己埋的 bug。
- **⚠⚠ 致命时序问题**：`sync_boundary()` 上线时被放在 `run_sync()` 的「今日 CSV 已到齐」短路**之后**（`if latest >= today: return SyncResult.SUCCESS`）。今日数据通常 12:00 首跑就到齐，于是 **13:00~17:00 的每一轮补跑都会在短路处提前 return，`sync_boundary()` 整个失效**——全天只有 12:00 那一次机会拉次日边界。若 11 点发布有滑动（迟到几分钟），12:00 那次也可能扑空，且**当天再没有第二次机会**，预测会一直退回合成插值直到明天。**完全不报错**：`sync_boundary()` 本身跑得好好的，只是从 13:00 起再没被叫到。
- **✅ 已修**：`sync_boundary()` 挪到短路判断**之前**，让 12:00-17:00 共 10 次触发都有机会拉取，总有一轮能追上 11 点左右的发布窗口。
- **真实验证**：手动跑一次 `run_sync()`，日志顺序确认修复生效：
  ```
  20:15:04  次日边界：写入 24 行，2026-08-24 就绪=True   ← 先跑
  20:15:04  本地 CSV 已含今日数据，判定成功              ← 短路在后
  ```
  边界文件 mtime 与本次调用同步刷新——即便在「今日已到齐」的短路场景下，边界数据依然被更新。
- **状态**：✅ 全量 **91 项**回归绿（新增 `tests/test_run_sync_boundary_timing.py` 3 项）。
- **改动文件**：`kdocs_sync.py`（`sync_boundary()` 调用点从主链路完成后挪到函数最前）、新建 `tests/test_run_sync_boundary_timing.py`、`CLAUDE.md`。
- **坑**：**「新功能挂在旧短路后面」是今天犯的第二次同类错误**（第一次是 P0 那次把负价门槛的死条件留到今天才发现）。写 `sync_boundary()` 时脑子里的语境是「今日同步是否成功」，完全没意识到这句短路会让下午的每一轮补跑都跳过它。**加代码时，永远要问一句：这条路径上方有没有提前 return，会不会让我刚加的东西变成「只在第一次调用时生效」。**

## 2026-08-23（六）· ★ 接入次日真实边界曲线：干掉最严重的一处 train/serve skew

- **起因**：前一条判定「生产的未来特征全是 `/defaults` 取昨天的标量再插值」，并把「确认次日日前负荷能否取得」列为潜在收益最大的方向。用户提供金山文档链接与「日前负荷边界 / 96点日前负荷」表截图——**今天已有明天全天数据**。
- **⚠ 第三次「东西已经存在，只是没接上」**：查本机发现 `~/gd_price_forecast_pkg/deploy/sync_kdocs_for_forecast.py` —— 一个早就写好的「含未来日数据同步」消费方，指向同一工作簿的**另一个 AirScript**（`script/V2-4EOHISC2ny0j5mPcV56hzH/sync_task`，gdpower 现用的是 `V2-4XYa5wL8bOXP4JLQy4SVMP`）。它以「天气表」而非「电价表」为合并基准，因此带得出未来 1-2 天。**现有 `AIRSCRIPT_TOKEN` 直接可用，字段 21 列与主 CSV 完全同名。** 工作量从「新建采集管道 2-3 天」降到「接一路取数」。
- **实测确认**：HTTP 200，返回 168 行、范围 `2026/08/18 00:00 ~ 2026/08/24 23:00`；明日**负荷 6 列 + 气温/湿度/风速 24/24 齐全**，`rq_jia_ge` 0/24（未出清，预期）。**8/24 22:00 统调 146765 与用户截图里「96点日前负荷」表的整点值逐位一致 → 96→24 聚合口径 = 取整点值，不是 4 点平均**（这个本来要等明天才能定的问题当场解决）。
- **做了什么（TDD 全程）**：① 新建 `boundary.py`——次日边界数据的唯一实现（取数 URL / 21 列 FIELD_MAP / `extract_future_rows` / `is_ready` / `save` / `load`）。② `kdocs_sync.sync_boundary()` 每日拉取落 `tomorrow_boundary.csv`，**挂在主链路之外、不参与 `SyncResult` 裁决**。③ `api_server.build_tomorrow_rows()` 改为**优先读真值、拿不到回退合成插值**。④ `tests/test_boundary.py`(18) + `tests/test_tomorrow_rows_boundary.py`(9)。
- **⚠ 头号设计约束：未来行绝不能混进主 CSV**。未来行 `日前电价 = 0`，而主 CSV 是**训练数据源**——写进去会让 `retrain_model` 把 0 当电价学、`calc_accuracy` 拿 0 当实际价、`price_lag*/roll*` 把 0 传导进滞后特征。`boundary.save()` **物理上不写任何价格列**（有回归测试守着），从源头断路。已验证主 CSV 仍停在当日 23:00、行数未变。
- **⚠ 就绪判据是「值齐全」不是「行存在」**：截图实测金山表里 **08-25 的行已存在但所有列空白**（预建占位）。按「有行就用」写判据，数据晚到会把一整天空值送进预测、被 `.ffill()` 悄悄填成前一天的值——又一个不报错的静默失效。`is_ready()` 要求 24 行齐且 `REQUIRED_COLS` 每列非全空/全零；该列表**刻意排除**辐射/降水（天然稀疏）、日前西电（省调停发恒 0）、日前电价（未来行按定义为 0）。
- **效果（8/24 实测，同 req 同模型开关对照）**：
  ```
  输入 日前统调负荷   真实极差 44200  vs  合成极差 30135
    真实前8h [130370,135435,126495,120875,115320,112000,113430,115875]
    合成前8h [114400,114400,114400,114400,114400,114400,114400,124445]   ← 6 小时同一常数
  输出 预测极差 160.5 → 180.2；晚高峰 17-22 全线抬升（19 点 493→523）
       逐时最大差异 41.3 元
  ```
  合成插值把凌晨 0-5 点压成一个常数，而真实曲线 0 点有 130370 的前夜余波次高峰——**模型此前根本看不到这个形状**。
- **状态**：✅ 已上线 Air。全量 **88 项**回归绿。真实取数端到端跑通（写入 24 行、`2026-08-24 就绪=True`）。`tomorrow_boundary.csv` 已 gitignore（每日覆盖写的运行产物，两机各自拉取）。
- **改动文件**：新建 `boundary.py`、新建 `tests/test_boundary.py`、新建 `tests/test_tomorrow_rows_boundary.py`；改 `api_server.py`（build_tomorrow_rows 优先读真值 + 回退 + import boundary）、`kdocs_sync.py`（+`sync_boundary` / 挂进主流程 / import boundary）、`.gitignore`、`CLAUDE.md`。
- **下一步**：① **观察若干天真实效果**——今晚起 `accuracy.csv` 的 MAE/coverage 会开始反映真实曲线的影响，与 8/23 之前对比即是 A/B；② 确认次日数据每天几点发布，据此决定同步触发时点是否要调（现 11:00 起轮询到 18:00）；③ 「96点日前负荷」表意味着**广东现货本就是 96 点出清**，而整套系统跑的是小时级——这是下一个量级的方向；④ mini 侧同步。
- **坑**：① **`def load(x, path=BOUNDARY_CSV)` 的默认参数在函数定义时求值**，测试里 patch 模块属性根本改不动它——TDD 时实测踩到，已改 `path=None` + 调用时 `Path(path or BOUNDARY_CSV)`。这类「默认参数固化」在需要可替换路径的模块里是通病。② 一天之内**第三次**遇到「资产已存在但没在主项目留下痕迹」（mini 的 lag1 修复、mini 的 DEVLOG-mini.md、这条含未来日链路）。共同成因：它们都不在 gdpower 的 CLAUDE.md 里被引用过。**一个资产如果不在主项目文档里被提及，它对未来的你就等于不存在**——开工前除了 `ssh` 另一台机，还该扫一眼 `ls ~`。

## 2026-08-23（五）· 查次日负荷数据源 → 挖出「日前西电停发 425 天」：修负价死门槛 + 加输入特征体检

- **起因**：前一条定位到「生产的未来特征全是 `/defaults` 取昨天的标量再插值」，怀疑这是 rMAE 长期 >1 的结构性原因。本轮去查次日日前负荷预测能否拿到。
- **① 次日日前负荷预测：金山这条管道拿不到**。实调 AirScript 返回 168 行、范围 `2026/08/17 00:00 ~ 2026/08/23 23:00`，**没有 D+1 行**；且 CLAUDE.md 陷阱 4 已查证「当日 24h 数据早上 8-10 点就绪」，即这条管道**天然晚一天**。**但数据本身在报价时点是存在的**（D 日的日前负荷是 D−1 发布的），要用它必须另接一路源（交易中心会员端/调度披露）。**这一步卡在业务侧确认，仍是潜在收益最大的方向。**
- **② ⚠ 顺带全列扫描，挖出 `日前西电` 恒为 0 已 425 天**。用户确认根因：**广东省调 2025-06-29 起不再单独公布西电，并入了其他负荷数据**。数据坐实并精确定位并入去向——切换前后 30 天日均：西电 34257→**0**、**日前B类负荷 54255→97600（+80%）**、统调负荷 117728→127989（+8.7%）。**这不是采集故障，是上游口径变更，列不会回来。**
- **⚠⚠ 真正的危害不在精度，在一道白白失效 425 天的门槛**：负价两阶段模型的触发判据是 `is_winter and (req.west_power < 100)`，西电恒 0 → `no_west` **恒真**。实测全样本 `西电<100` 占 71.8%、2025-06-29 后 **100%**。**✅ 已修**：唯一实现抽到 `features_ext.neg_price_gate()`（api_server 与 backtest 共用），删掉死条件。实测（14376 小时，低价 <50 元共 242 小时）**recall 36%→51%、精度 11.5%**（候选方案中最高）。
- **⚠ 两条「看着更好但不能用」的方案，已记进 CLAUDE.md 免得重复试**：① 放宽到 4/5/9 月 recall 可到 67%，但精度掉到 8.2%——**辅助模型自 04-29 起未随每日重训更新**，放宽 = 让陈旧模型接管更多时段，风险大于收益；② 净负荷日内分位判据离线精度 **27.8%（最高）**，但**生产上不可实现**——`build_tomorrow_rows()` 的负荷/新能源是从 `/defaults` 两个标量按小时插值的合成曲线，净负荷日内分位会退化成固定的小时函数、不含信息。**又一次绕回「需要真实次日日前负荷曲线」这个前提。**
- **③ ✅ 加输入特征体检**（`features_ext.check_feature_health()` + `kdocs_sync._feature_health_alert()` 每日调用）：检 `all_zero` / `constant` / `nonzero_collapse` / `missing` / `all_nan` 五类。**这道监控专防「合法数值形式的静默失效」**——西电没变成 NaN 而是变成 **0**，于是缺失率检查、ffill 兜底、dropna 全都放行，模型照常训练、MAE 也没崩，只是悄悄丢掉一个 3.7 万 MW 量级特征的全部信息量。**与同日发现的「标称 90% 实际 47.8% 的置信区间」是同一类病：系统只监控它想得到的东西。**
  - `日前西电` **故意不进 `HEALTH_COLS`**（已确认是口径变更、恒零是既成事实，留着天天误报）；`实时*` 列同理。
  - `min_nonzero_rate` 默认 0 不启用——降水量 47%、辐射 54.5% 天然稀疏，开这档必须按列设阈值。**常数/全零两档对所有列都安全，是主力。**
  - 真实数据双向验证：当前监控列**无告警**；把西电加回监控列**立刻报出**「恒为 0（样本 721 行，非零率 0.0，唯一值 1）」。
- **⚠⚠ 一条被自己实验推翻的判断（重要，别再重复这个实验）**：我先判断「30% 训练样本用旧口径 B 类负荷 → 模型在学错误映射 → 这是 P0，比什么都重要」。**三窗口 A/B 推翻了它**——训练起点 2025-01-01 vs 2025-06-29，MAE 差 **−1.9% / +1.4% / −1.3%**，方向不一致、逐日胜率接近掷硬币。根因：**GBDT 是分段常数**，会在 B类负荷 ≈54000 与 ≈97600 处直接分裂、把两个口径学成两个分支——level shift 对树模型远不如对线性模型致命；且 `RECENT_WEIGHT=1.5` 本就在压低旧段权重。**连带把论文的「滚动窗口」建议从 P1 降到 P2**（在广东数据上暂无证据支持）。
- **状态**：✅ 全量 **61 项**回归绿。新增 `tests/test_neg_price_gate.py`(5) + `tests/test_feature_health.py`(9)，全程 TDD。
- **改动文件**：`features_ext.py`（+`check_feature_health` / +`neg_price_gate` / +`NEG_GATE_MONTHS`/`NEG_GATE_HOURS`）、`api_server.py`（负价门槛改调共用实现）、`backtest.py`（同步同一口径）、`kdocs_sync.py`（+`HEALTH_COLS` / +`_feature_health_alert` / 挂进每日流程 / 补 `import features_ext`）、`CLAUDE.md`。
- **下一步**：① **P1 从 `FEATURE_COLS` 移除 `日前西电`、`west_ratio` 两个死特征**（需重训 + 回测，消费端按 meta 切列所以零成本回退）；② P1 辅助模型纳入每日重训（陈旧 116 天），完成后再评估放宽负价门槛到 4/5/9 月；③ **P1 业务侧确认次日日前负荷预测能否从交易中心取得**——它同时卡着「净负荷判据」和「真实未来特征」两件事。
- **坑**：① **这一天里我两次在没跑数之前先下机理判断，两次都错**：早上把「晚高峰压峰」归因成树模型架构天花板（实际是 lag1 实现缺陷、mini 七周前已修），下午把口径断点按线性模型直觉高估成 P0（实际噪声级）。**机理推演只能用来选实验，不能用来下结论。** ② 设计监控告警时，「正常情况下必须静默」和「异常情况下必须报出」要**双向验证**——只测其中一边，很容易做出一个永远不响或天天响的告警。

## 2026-08-23（四）· mini 侧收编：git 分叉归一 + 覆盖率列部署 + 三个坑（skip-worktree / .sh 硬编码 / HTTP2-over-TUN）

- **任务**：把当天的覆盖率列与 lag1 修复部署到 mini，并收掉 mini 已存在七周的 git 分叉。
- **先抢救再动手**：动任何 git 操作前，先把 mini 独有内容打包拷回 Air（`~/gdpower-mini-rescue-20260823/`，4.4MB）：`DEVLOG-mini.md`(296 行) + review_lag1 工具 + 两份未提交改动的 patch + **`git bundle` 保住 5 个未推提交的完整历史**。`git bundle verify` 显示「records a complete history」。**这一步不依赖 GitHub，先消除单点风险再谈其它**。
- **分叉收尾（方案：以 Air 为准，只抓 mini 独有的）**：核对确认 mini 那 5 个未推提交里，3 个 Air 已有等价实现（存档缺失告警 / 端口残留检查 / 网卡自动探测），另 2 个是 update.sh 相关而 **`git diff origin/main HEAD -- update.sh` 差异 0 行**（Air 的合并版已完全覆盖）→ 可安全 `reset --hard`。reset 前另备份 mini 的 `models/`(9 个文件)，因为当天刚把模型移出版本管理、reset 会删掉工作区文件。
- **覆盖率部署**：mini 全量 47 项回归绿；`backfill_coverage.py` 回填 103 行（3 行因存档缺失留空），accuracy.csv 9 列对齐；API 重启后 `/accuracy` 带 coverage。
- **⚠⚠ 意外收获：mini 的覆盖率回填给出了一个比 A/B 更干净的天然实验**——两台真实生产系统、同一市场、唯一差异就是 lag1 那 11 行：
  ```
  月份      mini(7/8 起有修复)   Air(无修复)
  2026-05        0.525             0.538
  2026-06        0.381             0.381    ← 完全一致
  2026-07        0.603             0.515    ← 分岔
  2026-08        0.623             0.489
  ```
  **6 月两台一模一样，7 月起分道扬镳，分岔点精确落在 lag1 修复上线的 7/8。**
- **坑① `skip-worktree` 让 `reset --hard` 硬阻塞，且报错指向不相关的文件**：reset 一直失败报 `error: Entry 'CLAUDE.md' not uptodate`，但 `git diff CLAUDE.md` 是空的、`git status` 也不显示它。真因是 **mini 给 `models/*` 全打了 `git update-index --skip-worktree`**（「不让 git 碰模型」的土办法，与 Air 当天改用的 gitignore 是同一目的的两种实现），而 origin/main 里这些文件已被删除，git 拒绝处理。**排查手法：`git ls-files -v | grep -v "^H "`**（H=正常 / S=skip-worktree / 小写=assume-unchanged）——这是不可见状态，不出现在 `git status` 里。解除后 reset 立即成功。**教训：同一目标的两种实现共存时，冲突点不在实现本身，而在它们对第三方工具的可见性差异**；gitignore 是声明式、可见、进版本库，skip-worktree 是命令式、隐藏、只在本地索引里，后者在双机场景必然咬人。
- **坑② `.sh` 是 Path.home 重构的漏网之鱼 → mini API 直接起不来**：reset 后 mini 的 API 挂了，`launchctl print` 只给 `last exit code = 1`、端口无监听，而**手动 `python api_server.py` 完全正常**。真报错只在 plist 指定的 `logs/launchd_stderr.log` 里：`cd: /Users/hydtzyj/gdpower: No such file` + `mkdir: /Users/hydtzyj: Permission denied`。根因是 **2026-07-03 那次「Path.home 重构」只覆盖了 .py**（plist 靠装载时 sed 适配），`launch_api.sh` 和 `install_service.sh` 一直写死 `/Users/hydtzyj`——Air 上路径碰巧一致所以永不暴露，mini 此前靠本地未推改动兜着，一 reset 就现形。已改 `${HOME}/gdpower`，全仓复查 .sh/.py 不再有硬编码（余下命中是注释文字）。**教训：可移植性重构要按文件类型列清单逐类核对（.py/.sh/.plist/.json/Makefile），别只扫一类就宣布完成——这类 bug 在源机上永远不触发，只在另一台机器上炸**；另：**手动跑能起来 ≠ 服务能起来**，中间隔着 wrapper 脚本，launchd 失败一律先看 StandardErrorPath。
- **坑③ HTTP/2 过 Shadowrocket TUN 会卡死 → git 全线超时**：mini `git fetch` 报 `Empty reply from server` 或 `Failed to connect after 75s`，但 `curl https://github.com` 能通（12s）。**`git -c http.version=HTTP/1.1 ls-remote` 2.4 秒就返回**——是 HTTP/2 多路复用过 TUN 停摆，连接层其实秒建（time_connect 0.11~0.2s）。已 `git config --global http.version HTTP/1.1`（mini 上 zjpower/coal-watch/gas-watch/gdpower-pages/gdpower-devlog 一并受益）。
- **⚠ 但 mini 网络仍不稳，HTTP/1.1 只在链路通的窗口有效**：当天晚些时候 `ls-remote` 又连续失败。**双机同步的可靠兜底 = `git bundle` 中转**（本次全程用它）：Air 侧 `git bundle create delta.bundle <base>..main` → `scp` → mini `git pull --ff-only /tmp/delta.bundle main`；反向同理。增量 bundle 极小（单个提交 1226 字节），**完全绕开 mini 的网络**。⚠ 建 bundle 必须写 `<base>..main` 而不是 `<base>..HEAD`，否则 bundle 里没有 `refs/heads/main`，对端 `pull` 会报 `couldn't find remote ref main`。
- **状态**：✅ 两机 HEAD 一致（`d786934`），mini 工作区完全干净。mini 独有产物已全部入库（`DEVLOG-mini.md` 296 行 + review_lag1 工具 + 架构说明/备机配置 + 补档工具 + 外部数据模板），`_predeploy_bak_*/`、`_dedup_deploy_backup_*/` 已加 gitignore。
- **改动文件**：`launch_api.sh`、`install_service.sh`（$HOME 可移植）、`.gitignore`（模型产物补齐 + mini 备份目录）、新增 `DEVLOG-mini.md`、`架构说明.md`、`AIR_备机配置.md`、`tools/review_lag1_fix.py`、`tools/review_lag1_notify.py`、`tools/backfill_predict_0707.py`、`external/external_data_template.{csv,xlsx}`。
- **下一步**：① **mini 的 Shadowrocket 节点需要人工处理**（换节点是 GUI 操作，ssh 够不着）——在此之前 mini 的 `publish_pages.sh`/`sync_devlog.sh` 等联网自动化会间歇失败，双机同步走 bundle 中转。② mini 那份「飞书成功卡片重构」(+88 行 `tools/daily_retrain.py`) 本次未回收，patch 保存在 `~/gdpower-mini-rescue-20260823/patches/daily_retrain_feishu_card.patch`，需与 Air 新加的 honest 观察行合并后再上。③ 今晚 18:00 两机的每日重训都会首次带 honest 观察行与覆盖率告警。

## 2026-08-23（三）· 🔴 发现 mini 七周前已修好同一个 bug，Air 一直没有：price_lag1 日内形状修复移植上线

- **怎么发现的**：准备 ssh 到 mini 部署覆盖率列时，`git status` 显示 mini 有未提交改动，而且**正好是我今天改过的两个文件**。一看 diff——`api_server.py` 里躺着一段 **2026-07-08** 的修复，注释写着「修复日前 price_lag1 断裂……晚高峰压峰」。**这正是我今天在 Air 上从 8/20 事故一路查出来的同一个现象**。mini 还有一份从未推送的 `DEVLOG-mini.md`（296 行 / 12 条），里面完整记着 07-08 的定位过程：误差几乎全集中在 16–23 时（19–22 晚高峰 rMAE 2.88）、晚高峰被摁在 320–385 而实际冲到 550/572/695、**峰值削 40–55%**、两次重训候选在共同 OOS 窗口都打不过现役 → **证明不是「模型旧了」，重训无解**。
- **⚠ 我今天的归因错了**：我把「生产 price_lag1 唯一值=1、模型看不见日内爬坡」归类成「树模型架构天花板，要靠 MLP 治」。**它不是架构问题，是 `build_tomorrow_rows` 不填电价导致的实现缺陷**，七周前就有人修好了。教训：双机互备系统里，**「另一台机器上的同一个模块」是最容易被忽略的对照组**——它跑着同样的数据、同样的市场，却可能已经装了你正要发明的轮子。查问题前先 `ssh` 过去 `git status` 一眼，成本极低。
- **双机同期生产实绩（最强证据，7 月 22 ~ 8 月 22 各 30 天）**：
  ```
              MAE均值   rMAE均值   赢naive天数
  mini(有修复)  41.51     1.081      16/30
  Air (无修复)  50.69     1.329      12/30      → Air 差 22%
  ```
- **做了什么**：① 新增 `features_ext.lag1_proxy()`——用「昨日同小时的前一小时价」(`price_lag24.shift(1)`) 作代理，首行取历史最后一个真实价。**不是泄漏**：为 D+1 报价时（在 D 日），D 日的日前电价早在 D−1 就已出清。② 接进 `api_server.run_prediction`。③ **同时接进 `backtest.build_features_honest`**——诚实口径的意义是复现生产信息集，生产改了它必须跟着改，否则回测比生产更瞎、又是一重新的不对称（方向相反）。唯一实现放 `features_ext.py`，两边共用。④ `tests/test_lag1_proxy.py` 8 项 TDD。
- **Air 侧 30 天离线 A/B（同模型同窗口，唯一差异 = lag1，真实日前输入）**：
  ```
                    A(现状)   B(修复后)    变化
  全日 MAE           40.29     34.26      -15.0%
  晚高峰 17-22 MAE   49.83     44.05      -11.6%
  区间覆盖率          0.533     0.643      +20.6%
  预测极差           147.8     221.4      +49.8%（实际 237.1）
  rMAE               0.939     0.798
  赢 naive 天数         14        29
  ```
  预测极差从 147.8 逼近实际的 237.1——「恢复日内形状」是直接可见的；顺带把昨天刚治的覆盖率也拉高 20%。
- **⚠ 线上路径增益明显小于 A/B，已实测确认**：用 API 自己的代码路径做开关对照（同一 req、同一模型），极差 95.9→109.5，逐时最大差异 48.5 元，晚高峰普遍抬升（20 点 424 vs 395、21 点 414 vs 393）。**修复确实生效，但幅度小**。根因是**生产的负荷/气象走 `/defaults` 合成插值（peak/valley 两点 + 正弦/正态曲线），不是真实的次日日前负荷曲线**——这正是我昨天列的「回测 vs 生产」三重不对称里的**第三重**，lag1 修复治不到它。**下一个真正的大头在这里：让 `/predict` 吃到真实的次日日前负荷曲线。**
- **状态**：✅ 已上线 Air（API 13:48:30 重启，进程/端口/健康均已核验）。全量 **47 项**回归全绿。`legacy` 口径回测仍 13.33（未受影响）；`honest` 口径 39.94→**34.47**（与 A/B 的 34.26 吻合）。
- **⚠ 一处行为变更（不是弯曲测试迁就代码，是有意改意图）**：`tests/test_honest_lag.py` 原有一条断言「honest 口径 lag1 塌成单值」，那复现的是**修复前**的生产。生产打上代理后已改为断言「恢复 24 点日内形状 + 明日 h 时 lag1 == 昨日 (h−1) 时价」，文件头注释已标明变更缘由。
- **⚠ 数据留痕**：上线验证时调 `/predict` 覆盖了 `predictions/prediction_2026-08-23.csv`（原件是 08-22 17:00 旧模型生成的）。信息集相同（CSV 最新仍是 08-22），仍是合法 D−1 预测，但 **08-23 的 accuracy 行反映的是修复后模型**，读历史序列时这里有个断点。
- **改动文件**：`features_ext.py`（+`lag1_proxy`）、`api_server.py`（run_prediction 套用代理）、`backtest.py`（`build_features_honest` 同步套用）、新建 `tests/test_lag1_proxy.py`、`tests/test_honest_lag.py`（一条断言随行为变更改写）、`CLAUDE.md`。
- **mini 的 git 现状（未处理，下一步）**：分叉 —— **5 个未推提交**（其中 3 个 Air 已有等价实现：存档缺失告警、端口残留检查、网卡自动探测）＋ **未提交改动**（lag1 修复、`tools/daily_retrain.py` +88 行飞书卡片重构）＋ **`DEVLOG-mini.md` 296 行从未进远端**。且 mini `git fetch` 连不上 GitHub（`curl https://github.com` 能通但要 12s，`fetch` 75s 超时，Shadowrocket TUN 在跑）。**已定方案：以 Air 为准，mini 重置到 origin/main，只把真正独有的东西（DEVLOG-mini.md、飞书卡片重构、review_lag1 工具）单独提取提交**——lag1 修复本次已由 Air 侧重新实现并放进 `features_ext`，不必再从 mini 搬。
- **坑**：① 往已有 docstring 后面追加说明时，容易把文字插到结尾 `"""` 的**外面**变成语法错误——`py_compile` 能立刻抓到，改完务必跑。② 单日「预测极差」不能用来验证修复是否生效：历史 10 天极差在 87~150 之间波动，噪声远大于单日效应。**要证明一个修复在线上生效，得用同输入的开关对照**（monkeypatch 掉修复函数跑第二遍），不能看单日指标。③ `api_server` 的模型加载在 `@app.on_event('startup')` 里，离线复现要 `asyncio.run(S.startup_event())`，没有 `load_models()` 这种函数。

## 2026-08-23（二）· backtest `--honest-lag` 并行观察上线：给日内 look-ahead 装上可对比的第二把尺

- **背景**：接上一条。日内 look-ahead 已定位但没修——直接改默认口径的连锁反应太大（历史回测 MAE 会从 14 量级跳到 40 量级、所有历史数字作废，且 `daily_retrain` 的 prod 分支会从「必过」翻转、容差 1.05 极可能立刻变成天天拒绝，正是 07-02 想避免的失败模式）。本轮按「先并行观察、再切默认」的既有纪律（`QUANTILE_CI_ENABLED`、`tuned_params.json` 都是这么干的）加第二把尺。
- **做了什么**：① `backtest.build_features_honest(df, target_date)`——把 target_date 及之后的电价置 NaN 再走**同一个** `build_features`（刻意复用而非另写「诚实版特征工程」，特征契约必须唯一，否则又是「三处重复」那个坑；屏蔽 + 下游 `.ffill()` 恰好复现生产的「lag 塌成 D−1 日 23 时单值」行为）。② `backtest.py --honest-lag` 开关，**默认关**；输出 JSON 加 `lag_mode: legacy|honest` 标记（两套数字要长期并存，必须能一眼分清是哪一套）。③ `daily_retrain.run_backtest(honest=)` + `honest_gap()`：COMPARE 后额外跑一遍诚实口径，把「候选 vs 生产实际」的 MAE 与比值记进 `.daily_retrain_state.json`（`honest_mae/honest_prod_mae/honest_ratio/honest_days`）和飞书卡片，**观察态、绝不参与 DECIDE**，整段包 try——观察器没有资格打断重训。
- **⚠ 最容易做错的地方：只屏蔽「日前电价」，不能多屏蔽**。`price_lag24/48/168` 在报价时点**合法已知**（为 D+1 报价时，D 日的日前电价早在 D−1 就出清了）；`load_roll24_mean` 用的是**日前**统调负荷；日历/负荷派生（net_load、bid_space、load_mom_diff…）全不受影响。**屏蔽它们＝过度修正，会把模型打成瞎子**。这四条各写了一条回归测试正面守着，不是靠注释提醒。
- **首轮实测（30 天，同一候选 `staged_models/20260822_180011`、同一生产基准 accuracy.csv 同窗）**：
  ```
  口径      候选 MAE   生产实际 MAE   比值     门槛(×1.05)
  legacy     13.33       50.69       0.263      通过
  honest     39.94       50.69       0.788      通过
  ```
  **好消息：修了泄漏门槛也不会立刻变「天天拒绝」**——0.788 距 1.05 仍有余量，07-02 担心的失败模式暂未出现。耗时 0.94s → 2.30s，可忽略。
- **⚠ 但 0.788 仍然偏乐观**：honest-lag 只修掉三重不对称里的**一重**，另两重仍在——① 训练集含被回测日期（跨日泄漏）；② backtest 用**真实**日前负荷/气象，而生产走 `build_tomorrow_rows()` 从 `/defaults` **插值估算**。真实 out-of-sample 比值会更接近 1。**切默认前必须攒够观察数据，别拿 0.788 当结论。**
- **切默认的判据（写死在 CLAUDE.md，免得日后忘）**：观察一到两周——`honest_ratio` 稳定 < 0.9 → 可切默认且保持容差 1.05；常态逼近或超过 1.0 → 说明那 3.8 倍优势基本是泄漏撑的，**切默认的同时必须重设容差**，否则从「永久通过机」翻转成「永久拒绝机」。
- **状态**：✅ 已上线 Air（观察态）。`tests/test_honest_lag.py` 9 项 + `tests/test_honest_observation.py` 8 项全绿，全量 **39 项**回归全绿。**默认路径零行为变化已逐日字节级验证**（改动前后 legacy 口径 30 天 MAE 逐日比对，不一致天数 = 0）。集成路径已用真实 staged 模型冒烟跑通（`run_backtest(honest=True)` → `honest_gap` → `{'mae_honest': 39.94, 'mae_prod': 50.69, 'ratio': 0.788, 'n_days': 30}`）。今晚 18:00 的 launchd 任务是第一次真实端到端运行。
- **改动文件**：`backtest.py`（+`build_features_honest` / +`--honest-lag` / 日循环按开关取 X / 输出加 `lag_mode`）、`tools/daily_retrain.py`（+`CMP_HONEST_JSON` / `run_backtest(honest=)` / +`honest_gap()` / COMPARE 后观察段 / 飞书卡片加 honest 行）、新建 `tests/test_honest_lag.py`、新建 `tests/test_honest_observation.py`、`CLAUDE.md`（新增一节）。
- **下一步**：① 攒 1–2 周 `honest_ratio` 观察数据再决定切默认与容差；② mini 侧 `git pull` 后同样进入观察态（两机各自记录）；③ 修完 backtest 口径后**重跑 08-02 的分位 vs 启发式三窗口验证**——原实验设计无误，只是地基被污染，结论可能反转；④ conformal 偏移量接进每日滚动重算。
- **坑**：① 屏蔽帧里目标日的「日前电价」已是 NaN，**实际值必须从未屏蔽的帧取**，否则 ground truth 会变成 NaN——日循环里 `actual`/`日前西电` 仍走 `day_df`，只有 `X` 走屏蔽帧。② `honest_gap` 的入参是「生产实际 MAE 映射」而不是路径，为的是可单测；`_prod_mae` 返回三元组 `(mean, n, map)`，取第三个。③ 逐日重建特征看着像 O(n²)，实测 30 天只从 0.94s 涨到 2.30s，别为此提前优化去手写「只补价格列」的快路径——那正是特征契约分叉的开始。

## 2026-08-23 · 论文改造核验 → 追出两重问题：区间覆盖率长期失真（已治）+ backtest 日内 look-ahead 泄漏（已定位未修）

- **背景**：用户拿内部研究资料《深度学习日前电价预测：欧洲方法论解读与广东本地化改造方案》（基于 Aliyon & Ritvanen, Energy 308 (2024) DREAMFS）对照核验 gdpower 落地度。核验结论：**方法论落地约 45%，落地的是「制度层」（sMAPE/rMAE、每日重训）、没落地的是「模型层」**（MLP 集成成员 ❌、regime/双模型路由仅负价窄版雏形、限价 clip ❌）。资料 3.4 承诺交付的三个模块 `feature_audit.py`/`eval_metrics.py`/`mlp_member.py` 仓库里一个都不存在（eval 那块功能等价内嵌在 kdocs_sync）。
- **⚠ 顺带证伪一条建议**：资料把「输出层 clip 到限价上下限」列为高优先级。实测 112 天存档 **点预测值域仅 159.4~669.5、区间下沿历史最低 94.2、破 -50 次数 0/2616**——GBDT 预测值数学上锁死在训练集值域内，**clip 对当前纯树模型架构完全空转**，它只有在引入 MLP（能线性外推）后才是必需品。**clip 不是独立武器，是 MLP 的安全带。**
- **⚠⚠ 主发现 1：标称 90% 的置信区间，生产实测覆盖率仅 0.478**（108 天 / 2592 小时；月度 05=0.538 / 06=0.381 / 07=0.515 / 08=0.489；6/19 那天 **0.04**，24 小时里 23 小时跌破下沿）。**四个月无人察觉，因为 accuracy.csv 那 8 列全是点预测误差，没有一列衡量区间**。机制：`ci_base=|xgb−lgb|*1.5+25`，XGB/LGBM 同族同特征同数据，异常日**一起错且错得一致** → 分歧小 → 区间窄 → **又错又自信**。从交易视角：覆盖率 47.8% 的「90% 区间」不是保守估计，是错误的风险定价。
- **⚠⚠ 主发现 2：`backtest.py` 有第二重、未被记录的日内 look-ahead 泄漏**。`build_features` 在整份 CSV 上跑完再按日切片，**被预测日自己的真实电价参与了自身 lag/roll 特征**——实测 8/20 `price_lag1` 恰等于当天前一小时真实价、24 个不同值；而生产侧 `build_tomorrow_rows()` 构造的明日 24 行**根本没有「日前电价」字段**，lag 全 NaN 被 ffill 成 D−1 日 23 时同一个值（唯一值=1）。最小对照（同模型/同窗口/22 天，只差 lag 来源）：**A backtest 口径 cov=0.928 MAE=14.26 ｜ B 生产口径模拟 cov=0.527 MAE=40.46 ｜ C 生产存档实测 cov=0.489 MAE=51.22**，三者宽度几乎相同（67/71/73）→ 差异全在点预测精度。这与 `backtest.py:6` 自述的「训练集含被回测日期」是**两重不同的泄漏**：第一重在 fair 模式下双方对称可抵消，**第二重对「回测 vs 生产」跨口径比较不对称、抵消不掉**。
- **⚠⚠ 泄漏的连带影响：`daily_retrain` 上线门槛已失效**。`compare_windows()` 的 `prod` 分支（注释自述「每日 promote 稳态下走这级」）拿**带泄漏的候选回测 MAE** 比**无泄漏的生产实际 MAE**。今日实测 `mae_new=19.12 vs mae_ref=50.69`（阈值 ×1.05=53.22）**差 2.65 倍通过**，护栏 `rmae_new=0.444` 远低于 1.0，DM `p=1.05e-07` 的「极显著」也是泄漏产物。**这道闸门不可能拒绝任何候选**——07-02 设计时怕它变「永久拒绝机」，矫枉过正成了「永久通过机」。风险不在今天的模型不好，而在**某天数据异常导致候选变差时它不会拦**。已核对**不受影响**的：07-05 特征消融（双臂同口径对称）、07-02「旧模型泄漏 1.578 倍」（fair 模式对称）、**accuracy.csv 全部指标**（直接用存档×真值算，从不经过 backtest）。
- **做了什么（本轮只治覆盖率这一项，TDD 走完 RED→GREEN）**：① `accuracy.csv` 加第 9 列 `coverage`（实际价落在 `[pred_lower,pred_upper]` 的小时占比；24 小时必须全有区间才算，缺任一小时留空——半截样本的比例会误导）。② 新增 `_ensure_accuracy_header()`：旧版 8 列文件追加前**原地自愈升级**（旧行补空、`.tmp` 原子替换、幂等）。**刻意做成自愈而非依赖「先回填再上新版」的人工顺序**——两机各自维护 accuracy.csv，任一台漏跑就会让 9 列行写进 8 列表头、DictReader 把多出字段塞进 None 键、消费端静默错位。③ 新增 `_recent_coverage_mean()` + `_coverage_alert()`：近 30 日均值 < `COVERAGE_FLOOR`（默认 0.75，.secrets.json 可覆盖，**设 0 即关闭**）→ 飞书「区间覆盖率预警」，有效样本 <15 不判。④ 新建 `tools/backfill_coverage.py` 回填历史（108 行成功 / 4 行因存档缺失或无区间列留空），产出与独立算法交叉验证一致。⑤ 消费端 `api_server./accuracy`、`tools/export_snapshot.py` 各加一行透传（缺值→None）。
- **⚠ 覆盖率告警为何用绝对口径**：rMAE 预警 07-11 因绝对阈值恒触发改成了「相对自身近30日P75漂移」，覆盖率**不能照搬**——它的参照系是标称置信水平本身，用相对基线会把「长期失真」当成新常态、永远不报，**那正是这次要治的病**。
- **状态**：✅ 已上线 Air。`tests/test_coverage_metric.py` 12 项全绿，全量 22 项回归全绿，4 个改动文件 py_compile 通过。accuracy.csv 已回填（备份 `accuracy.csv.bak_20260823_084741`），9 列对齐已验。**当前滚动 30 日覆盖率 0.5069 < 0.75 → 上线后会每天推送告警**，这是正确行为（问题确实每天都在）。**API 未重启**，透传未生效；未重启前旧代码靠 `row.get` 忽略新列不会崩。
- **改动文件**：`kdocs_sync.py`（+ACCURACY_HEADER 第9列 / +COVERAGE_* 三常量 / +`_ensure_accuracy_header` / +`_recent_coverage_mean` / +`_coverage_alert` / calc_accuracy 读区间+算覆盖率+写9列+调自愈 / 末尾挂告警）、`api_server.py`（/accuracy 透传 coverage）、`tools/export_snapshot.py`（快照透传）、新建 `tools/backfill_coverage.py`、新建 `tests/test_coverage_metric.py`、`CLAUDE.md`（新增「区间覆盖率列 + 滚动告警」节）。
- **下一步（按修正后的优先级）**：① **P0 修 `backtest.py` 日内 look-ahead**——但**不要一次性改默认**，建议先加 `--honest-lag` 开关并行跑两套数字对比一到两周：修完后历史回测 MAE 会从 14 量级跳到 40 量级、所有历史数字作废需重建基线，且 `daily_retrain` 的 `prod` 分支会从「必过」翻转，**容差 1.05 极可能立刻变成天天拒绝**（正是 07-02 想避免的失败模式）。② P0 mini 侧各自跑一次 `backfill_coverage.py`（两机数据不互通）。③ P1 修完 backtest 后重跑 08-02 的分位 vs 启发式三窗口验证——原实验设计无误，只是地基被污染，结论可能反转。④ P1 conformal 偏移量接进 daily_retrain 每日滚动重算（其不稳定的根因是「校准一次长期用」＝陈旧，正是论文「误差的敌人不是波动，是陈旧」的又一实例）。⑤ P2 加 MLP 成员（同时加 clip + 补 day_of_week 的 sin/cos + scaler，注意 MLP 需缩放后特征、不能直接复用 X_base，这是最典型的 train/serve skew 来源）。
- **坑**：① **「一个让你感觉良好的 bug，比一个报错的 bug 危险得多」**——日内 look-ahead 不产生任何错误信息，回测跑得又快又漂亮（MAE 14 vs 生产 51），只会让人觉得「模型其实挺强，是线上环境不行」，所以藏了很久。② 评估工具的泄漏会**同时抬高实验组和对照组**，当抬高不对称时 **A/B 判读的方向都可能反过来**——08-02「分位不如启发式」正是栽在这里。③ 回填工具里 `r['coverage']` 可能是回填出的 float 也可能是读进来的 str，直接 `.strip()` 会 AttributeError，要统一过一遍取值 helper。④ 本机 gdpower env **没装 pytest**，跑测试用 `python -m unittest discover -s tests -q`。
- **配套 Obsidian 笔记**（`广东电力市场电价预测` vault/3-模型迭代/）：《论文改造核验_DREAMFS欧洲方法论落地度_2026-08-22》（353 行，含「三件核心武器详解」）、《8-20分位区间失效追查_2026-08-22》（297 行，含根因与最小对照实验）。

---

## 2026-08-02 · 分位预测（LightGBM quantile）阶段一验证 + conformal 校准 + 分位专属调参：方向验证成功，仍暂不接入

- **背景**：07-04 DEVLOG 提过 deep-research 报告给的「QRA 概率头」是长线待落地方向，此前从未实现。现状「预测下限/上限」其实是 `api_server.run_prediction()` 里 `|pred_xgb-pred_lgb|*1.5+25` 的启发式带宽，没有统计意义上的分位保证，`backtest.py` 也完全没有覆盖率/pinball 评估代码。本次目标：给点预测加真正的 LightGBM 分位回归，先离线验证再决定是否接入生产。
- **做了什么**：① 新建 `quantile_utils.py`（`fix_crossing`/`pinball_loss`/`coverage_rate`/`ci_level_to_alphas`，backtest.py + api_server.py 共用，避免重蹈「特征契约三处重复」的坑）。② `retrain_model.py` 加 `--quantile` 开关 + `train_quantile_models()`：训 P5/P10/P50/P90/P95 五个 `lgb.LGBMRegressor(objective='quantile')`，**复用主 lgb_model 完全相同超参**（只改 objective，干净对照），落盘为独立的 `model_meta_quantile_v2.json`（不与主模型 meta 合并——分位模型走独立更新节奏，`feature_cols` 自带，不借用 `_active_feature_cols()`）。三种落盘模式（dry-run/stage/正式）都接了；正式模式复用现有 `glob('*.pkl')` 同步逻辑，零改动自动同步到 `TRAIN_MODEL_DIR`。③ `backtest.py` 加 `--quantile-dir`：覆盖率/pinball loss 评估，按「冬季+西电<100MW」负价触发窗口单独拆分子集统计（不能被全年平均掩盖尾部风险），同期现算启发式区间的覆盖率/宽度做对照。④ 新建 `tools/exp_quantile_coverage.py`，照抄 `exp_feature_ablation.py` 编排范式（`--stage --quantile` → `backtest --quantile-dir` → 判读）。⑤ `api_server.py` 接入：`QUANTILE_CI_ENABLED` 开关（`.secrets.json`，缺键默认 false，仿 `DAILY_RETRAIN_ENABLED` 同款约定）、`_load_quantile_models()`、`run_prediction()` 区间计算改用分位模型（预测异常/无对应 alpha 一律回退启发式；负价窗口取 `max(分位宽度,启发式宽度)` 兜底，不让区间比现状更窄）。**本轮全程 `QUANTILE_CI_ENABLED=false`，生产行为零改变**。
- **⚠ 负结果：三窗口 out-of-sample 验证一致显示分位区间「更宽但覆盖率更低」**。用三个不同 `--test-start`（06-01/06-15/07-01，训练截止日期均严格早于各自评估窗口起点，天然 OOS）跑 `exp_quantile_coverage.py`：
  ```
  窗口           覆盖率80%(目标.8)  覆盖率90%(目标.9)  宽度80%(分位/启发式)  宽度90%(分位/启发式)
  06-01~08-01(62天)   0.73              0.85            67.89/52.45          99.98/74.93
  06-15~08-01(48天)   0.69              0.80            61.13/50.76          90.77/72.52
  07-01~08-01(32天)   0.73              0.84            55.71/46.34          80.59/66.19
  ```
  三窗口方向一致（非单窗口噪声）：覆盖率系统性偏低，同时区间宽度比启发式还宽 20%~35%——正常逻辑下更宽应该更容易覆盖，这里反过来，说明不是「不够宽」而是**分位区间的位置系统性偏了**。推测原因：分位模型直接复用点模型超参未针对 quantile 损失调参、单模型（无 XGB+LGB 集成降方差）、训练/测试窗口间价格分布本身在漂移（daily 重训存在的原因）对分位回归比集成点预测更敏感。
- **补充实验：conformal 校准（Split Conformal / CQR，Romano et al. 2019）**——用户选定这个方向验证「不重训、事后修正覆盖率」是否可行。① `quantile_utils.py` 加 `conformal_offset()`（有限样本修正的经验分位数，nonconformity score = `max(q_low-y, y-q_high)`）+ `apply_conformal()`。② `retrain_model.py` 加 `--calib-days`（默认30）：从 `test_start` 往前再切一段独立校准集，**只影响分位模型训练/校准，xgb/lgb 主模型训练数据完全不受影响**；校准偏移量存进 `model_meta_quantile_v2.json` 的 `conformal_offsets: {"80":..., "90":...}`。③ `backtest.py`/`api_server.py`/`exp_quantile_coverage.py` 都接了：读到 `conformal_offsets` 就应用校准（老版本 meta 无此字段则 offset=0，向后兼容零改动）；`backtest.py` 输出启发式/原始分位/conformal 三方对照。
- **conformal 效果（同三个窗口，calib-days=30）**：
  ```
  窗口              conformal偏移量(80/90)   覆盖率80%(启发式/原始/conformal)  覆盖率90%(启发式/原始/conformal)  宽度80%(启发式/原始/conformal)  宽度90%(启发式/原始/conformal)
  06-01~08-01(62天)   +1.87 / -0.47            0.67 / 0.79 / 0.82              0.78 / 0.89 / 0.89              52.45/83.26/87.01              74.93/121.41/120.48
  06-15~08-01(48天)   -1.41 / -3.31            0.69 / 0.78 / 0.76              0.81 / 0.88 / 0.85              50.76/77.79/74.97              72.52/112.46/105.85
  07-01~08-01(32天)   +9.48 / +10.18           0.74 / 0.75 / 0.88              0.87 / 0.85 / 0.93              46.34/55.78/74.75              66.19/80.83/101.20
  ```
  **方向验证成功**：三个窗口覆盖率一致被 conformal 校准拉向目标值（部分已进入判读区间 [0.76,0.84]/[0.86,0.94]），证明「原始分位区间系统性欠覆盖」这个诊断是对的、且能被独立校准集量化修正。**但代价是区间比启发式更宽（宽 40%~80%），且校准偏移量本身三窗口很不稳定（80% 档从 -1.41 摆到 +9.48，甚至变号）**——这暴露 split conformal 的可交换性假设在非平稳的电价序列上打了折扣：今天算出来的偏移量，未必适用于两周后的数据。**判读结论仍是不建议接入阶段二**，但性质从「分位方法本身不可靠」升级为「分位方法可信、但静态校准不够、且比启发式贵（更宽）」，是更精确的诊断。
- **补充实验：分位模型专属超参重搜（⚠ 负结果，与 07-03 点模型调参同一模式）**——用户选定第三个方向，验证「不是方法不行、是超参没调对」这个假设。新建 `tools/tune_quantile_hyperparams.py`，与 `tune_hyperparams.py` 同构：目标=3 个时序折（复用同一套 `FOLDS` 定义）×5 个 alpha 的 pinball loss 均值，最近 14 天为终检窗（搜索全程不可见），best 与「复用 lgb_model 默认超参」各在终检窗跑一次，只有不劣于默认才落盘到 `models/tuned_quantile_params.json`；`retrain_model.py` 同步接入读取（存在则用、不存在或读取失败回退现状，向后兼容）。刻意不搜 `recent_weight`/`recent_window_days`（07-03 已经搜过这块、因整体不落盘未采纳，本轮控制变量只聚焦分位专属树超参）。**跑 50 trials 结果**：CV pinball 调优后 7.61 vs 默认 7.82（**赢 2.7%**），但终检窗（未参与搜索的最近 14 天）调优后 3.7543 vs 默认 3.7324（**反而输 0.6%**）——守卫按设计生效，**未落盘，`retrain_model.py` 继续复用 lgb_model 超参，等价于本次实验完全没发生**。**这是这个项目第三次撞见同一个模式**（07-03 点模型调参 CV 赢 5.4%/终检窗输 1.9%；这次分位模型调参 CV 赢 2.7%/终检窗输 0.6%）：三折 CV 在历史区间里找到的「最优」超参，在真正没见过的最近数据上经常反而更差——电价类非平稳序列上 Optuna 调参的通用陷阱，「终检窗搜索不可见 + 不劣于默认才落盘」这道守卫已经连续两次拦下本可能让线上变差的参数，证明这套方法论本身可靠。
- **状态**：代码已合入且可运行（语法检查 + import 验证通过，dry-run/stage/backtest/conformal/调参 全链路跑通），**阶段二开关 `QUANTILE_CI_ENABLED` 全程保持 false、未在生产验证；`tuned_quantile_params.json` 未落盘（守卫拒绝）**。
- **改动文件**：新建 `quantile_utils.py`（含 conformal 函数）、`tools/exp_quantile_coverage.py`（`--calib-days` 透传 + 三方判读）、`tools/tune_quantile_hyperparams.py`（分位专属 Optuna 调参，未落盘）；改 `retrain_model.py`（`--quantile`/`--calib-days`/`train_quantile_models`/校准集切分/conformal_offsets 落盘/读取 `tuned_quantile_params.json`）、`backtest.py`（`--quantile-dir`/覆盖率评估/conformal 三方对照，顺手把 `avg()` 从 `r[key]` 改成 `.get(key)` 防单日分位预测异常时 KeyError）、`api_server.py`（`QUANTILE_CI_ENABLED`/`_load_quantile_models`/区间计算接入分位+conformal）、`.secrets.json`（+`QUANTILE_CI_ENABLED: false`）。备份 `retrain_model.py.bak_20260801_234425`、`backtest.py.bak_20260801_234608`、`api_server.py.bak_20260801_234832`。**未推 GitHub、未碰 serving、未开生产开关**。
- **下一步（如果还想继续这个方向）**：调超参这条路已验证走不通（负结果，不必重复）。剩两个选项：① **每日滚动重新校准**——conformal 偏移量不稳定的根因可能是「校准一次、长期用」，而这个系统本来就有每日重训基础设施（`daily_retrain.py`），把 conformal 偏移量接进每日流程用最新滚动校准集重算，理论上能跟上分布漂移，值得验证是否解决稳定性问题（工作量中等）。② **接受区间更宽这个业务取舍**——如果「区间统计上站得住脚、但比现在宽」可以接受，三窗口 conformal 结果已经基本够格开开关，这是业务判断不是技术判断。若都不继续，代码原样留着零风险（开关默认关闭、超参文件未落盘），或删掉分位相关代码块回滚。
- **坑**：① 分位交叉（LightGBM 各 alpha 独立训练不保证 P10≤P90）必须显式兜底，已封装进 `quantile_utils.fix_crossing`，backtest/api_server 都用它，别各写一份。② 负价触发窗口（10-16点/冬季/西电<100MW）样本天然稀疏，本次三个验证窗口都是夏季（06-08月），**负价窗口触发天数=0，尾部风险这块完全没被验证到**——如果以后继续这个方向，务必找一个跨冬季的窗口补验证，不能只看夏季数据下结论。③ `ci_level=95` 挡（对应 P2.5/P97.5）本次没训，代码里遇到 95 档直接回退启发式，不冒充。④ **conformal 偏移量对校准窗口敏感**（见上表），不能只跑一个窗口就下结论「校准生效了」——这次能发现偏移量不稳定，恰恰是因为坚持了三窗口验证的习惯（这个项目从 07-05 特征消融那次开始就吃过单窗口结论的亏）。⑤ **分位模型 Optuna 调参耗时比点模型高一个量级**（15 次 LGBM fit/trial vs 点模型 6 次/trial），50 trials 实测约 25 分钟——以后要跑更大规模（如 150 trials 对齐点模型规格）得预留时间或后台跑，别指望前台一次性等到。

---

## 2026-07-05 · 论文精读派生实验：「论文 6 特征 vs 现役 38 列」特征消融

- **背景**：精读《中国电力》2026(5)「基于 Patch 机制与通道独立结构的改进 Transformer 日前电价预测」（研究对象就是广东现货，用 2024-03~09 数据，特征筛到 6 个：竞价空间/时刻/日类型/统调负荷/电价同比/负荷环比，累计增益 97.2%，剔了煤价/节点历史电价/西电）。与我们 07-03「煤价消融证有害已剔除」独立撞车。用户要求验证：把 38 列砍到论文 6 特征，我们的树模型会掉多少。
- **做了什么**：① `retrain_model.build_features` + `backtest.build_features` 各加 3 个惰性衍生列——`bid_space`（按论文口径 `统调负荷−(西电+港澳+地方+A类)`，均值≈7 万 MW，落在论文图 1 竞价空间核心区间）、`load_mom_diff`（负荷一阶差分）、`price_yoy_chg`（`1−lag24/lag48`）。② `FEATURE_COLS` 加 env 门控 `GDPOWER_FEATURE_SET=paper6`，切成论文 6 特征；**不设变量则默认字节不变**。③ 新建 `tools/exp_feature_ablation.py` 编排「两臂 `--stage` 训练→`backtest --model-dir`→`dm_test.dm_on_dates`→对照表+md 报告」，reuse 现有全部设施。
- **结果（单窗 2026-06-01~07-04，34 天，两臂 TRAIN_END 同为 05-31 控制变量成立）**：**6 特征 MAE=78.16 / rMAE=1.179（比抄昨天还差！）；38 列 MAE=31.27 / rMAE=0.472；DM dm=9.02 p<0.0001 极显著。** 论文「6 特征够用」**不可移植到树模型**——根因：论文 Patch-Transformer 直接吞原始电价序列、注意力内化自回归；我们树模型无时序结构，必须靠显式 `price_lag*`/`price_roll*` 看历史价，砍掉=失明。**反向 validate 了 38 列里价格滞后族是架构必需、不是冗余**。
- **状态**：✅ 实验完成、结果回写 Obsidian 论文精读笔记（`广东电力市场电价预测` vault/3-模型迭代/）。**生产零影响已三重确认**：dropna 只作用于 `FEATURE_COLS`（206 行，默认 38 列路径不含新列 → 训练集字节一致）；serving meta 仍 38 列、trained_at 07-04 未动；实验只写 staged_models（已清理今日 3 个）。
- **改动文件**：`retrain_model.py`（import os + build_features 3 列 + FEATURE_COLS env 门控）、`backtest.py`（build_features 3 列）、新建 `tools/exp_feature_ablation.py`。备份 `retrain_model.py.bak_20260705_154659`。**未推 GitHub、未碰 serving/数据/密钥**。
- **下一步**：① 若要严谨，`exp_feature_ablation.py --test-start 06-15/07-01` 多窗口复核（但差距 47 元 MAE、p<0.0001，翻盘概率极低）；② 结论「特征精简绝不能动价格滞后族，能砍的是气象/比率类」待后续边际特征消融确认；③ env 门控/惰性列可长期保留（无副作用），如要回滚删这几处即可。
- **坑**：① 特征契约**三处重复**（retrain/api_server/backtest 各一份 build_features）——加特征至少要同步训练侧+评估侧,否则 backtest 报「not in index」（本次首跑就踩了，补 backtest 后通过）。② 离线实验只需 retrain+backtest 一致，api_server（serving）因 meta 无新列永不选中、无需改也更安全。

---

## 2026-07-03 · 双机部署准备：Path.home 路径重构 + 每日重训飞书主备去重

- **背景**：mini 要与 Air 同样部署(两批全上)。核心矛盾=两机共用一个 git 仓但代码硬编码 `/Users/hydtzyj`(mini 是 zhouyijun)——**可移植性债**,每次同步都要 sed 搏斗。用户拍板:两台都开每日重训 + 现在就做 Path.home 重构一劳永逸 + 顺手飞书去重。
- **Path.home 重构**：全仓 17 个 .py 的硬编码 `/Users/hydtzyj/...` 改为 `Path.home() / ...` 动态推导(含之前遗漏的生产链路 notify_prediction/notify_compare,和 predict_tomorrow/analyze_april/backfill_* 等离线工具)。两机共用一份代码、**git pull 即用、无需 sed**。plist 是 XML 例外,装载时用 `$HOME` 适配一次。
- **回归验证(Air 零回归)**：① `git diff HEAD` 证明核心文件(backtest/api_server/retrain/features_ext/kdocs_sync)除路径行外**无任何逻辑改动**;② 路径解析 assert 逐字一致(`Path.home()/'gdpower'` 在 Air 上 == 原硬编码);③ 全部 py_compile + import 成功。Path.home 是确定性等价替换,未重启 API(用户不想扰动 serving,靠静态三证据足够)。
- **飞书主备去重**：`daily_retrain.feishu()` 加 `dedup` 参数。「✅ 已自动上线」成功通报走 `ks.should_push`(primary=mini 推了 backup=Air 就跳)→ 两台都开每晚也**只一份成功通报**;异常/回滚/门槛拒绝仍两台都推(各自问题都要看见)。
- **DM 检验(用户另一会话加,本次一并 push)**：`tools/dm_test.py`(85 行)+ daily_retrain 集成 Diebold-Mariano 检验,判断候选 vs 基准逐日误差差异是否统计显著,**观察态只报告不否决 promote**。import 已验证可跑。补上了「真实收益只认 out-of-sample」缺的统计显著性判断。
- **mini 部署清单**：`MINI_FULL_DEPLOY.md` 大幅简化——第 3 步从「全局 sed 改路径」变成「路径检查(通常无需操作)」;第 9 步重训开关定 True + 飞书去重说明;第 10 步 plist 用 $HOME 适配。
- **改动文件**：17 个 .py(路径)+ daily_retrain(去重)+ dm_test.py(新)+ MINI_FULL_DEPLOY/CLAUDE/DEVLOG。**不推模型/数据/密钥**。
- **待办**：mini 执行 `MINI_FULL_DEPLOY.md`;今晚 18:00 Air 每日重训会用新代码(Path.home + DM 观察态 + 飞书去重)自动跑,留意日志。

---

## 2026-07-04 · 路线A第三批：DM(Diebold-Mariano)检验接入 promote 门槛（观察态）

- **背景**：07-03 的「手调 vs Optuna 谁更好」之争暴露一个缺口——promote 门槛用「候选 MAE ≤ serving×1.05 容差」判上线，但对比窗口可能短到 <7 天，**MAE 略低分不清是真进步还是抽样运气**。依据 2026-07-04 的 deep-research 核实报告《中国省级现货电价预测建模》（欧洲 Weron/Lago 方法论在中国适配性，存于「工作」vault/Claude/），补 DM 检验这把统计显著性尺子（该报告另给 asinh 变换、QRA 概率头两个待落地方向）。
- **做了什么**：① 新 `tools/dm_test.py`——Diebold-Mariano 统计量 + Newey-West HAC 方差（滞后 h-1，处理电价误差自相关，防低估不确定性/高估显著性）+ Harvey-Leybourne-Newbold 小样本校正（t 分布而非正态，短窗口更保守）。符号约定 **正=候选(A)损失更大(更差)**。含 `dm_on_dates(new_mae, ref_mae, dates)` 按日期配对封装（缺失日跳过、<2 天无结论）。② **纯观察态**接入 `daily_retrain.compare_windows`：在决策同窗（fair 段或 prod 全窗）用**逐日 MAE** 跑 DM，dm_stat/dm_p/dm_n 加进返回 dict + 飞书报告 + state。`_prod_mae` 改为增返逐日映射（prod 模式 DM 需要）。main() 加一行 DM 解读（显著更好✅/显著更差⚠️/不显著/样本不足四分支）。
- **⚠ 安全设计**：DM **完全不进入 `passed`/`verdict`**——`passed = (not guard_fail) and (mae_new <= mae_ref*TOLERANCE)` 一字未动。哪怕 DM 有 bug，最坏只是飞书多一行错话，**绝不误 promote/误否决**。先让新信号「只说话不投票」，观察靠谱后再给投票权。
- **状态/验证**：TDD 全程（先红后绿）。`tests/test_dm_test.py`(9：识别相同/对称抵消/A更差符号正/B更差符号负/样本不足/多步 h 更保守 + dm_on_dates 配对/跳缺失/少样本) + `tests/test_compare_windows_dm.py`(2 集成：monkeypatch JSON 路径喂合成回测，**不触发训练/网络**，验证 fair 模式返回 DM 字段、符号、公平段天数) 全绿；飞书四分支渲染（含 inf 边界）已验。py_compile 通过。**gdpower env 无 pytest → 用纯 assert 脚本 + env python 跑**（未污染环境）。**仅离线验证，尚未在真实每日重训里跑过**（需当日数据就绪+会真训练候选）。
- **改动文件**：新建 `tools/dm_test.py` `tests/test_dm_test.py` `tests/test_compare_windows_dm.py`；改 `tools/daily_retrain.py`（import dm_on_dates + `_prod_mae` 增返逐日映射 + `compare_windows` 算 DM + main 报告加 DM 行 + state 存 dm）。
- **下一步**：① 今晚 18:00 定时或手动 `--dry-decide` 看第一条**活体 DM 输出**→确认靠谱；② 翻**硬门槛**（候选 dm_stat>0 且 dm_p<0.05 → 一票否决）；③ 做 **naive 基线修正**：现 `backfill_rmae.py`/`backtest.py` 的 naive 一律用 D-1（p_{d-1,h}），应改 **day-of-week 依赖**（工作日周二–周五用 p_{d-1,h}，周六–周一用 p_{d-7,h}，epftoolbox 口径），历史 accuracy.csv 用**方案 A 全量重算**；④ 更长线：asinh 变换、QRA 概率头。
- **坑**：① 逐日粒度检验力弱于逐小时，短公平段（<7 天）DM 常给「不显著」——这是诚实信息（无证据换模型），不是 bug；要更强检验力再让 backtest 多存逐点误差（DM 函数不用改，只是喂的序列变细）。② mini 侧未部署此批。③ 尚未 push GitHub。

---

## 2026-07-03 · 路线A第二批：特征改造(38列) + Optuna 超参重搜（含两个诚实负结果）

- **背景**：接第一批（每日重训+rMAE），补论文方法论另两块——广东本地化特征（论文欧洲气价→广东煤价）+ 系统性调参（论文「年度重调超参」）。
- **架构**：特征契约从「三处硬编码」升级为「**meta 驱动切片 + 冻结兜底**」。演进源=retrain_model.FEATURE_COLS，写进 meta；api_server（`_active_feature_cols()`）/backtest（读 --model-dir 的 meta）按所加载模型的 feature_cols 切列，35 列字面量降为极老模型 fallback。红利：**特征回退零成本 + daily_retrain.py 一行没改就跨代际公平对比**。辅助模型（weighted/负价）用冻结的 `FEATURE_COLS_AUX`(45) 彻底解耦。新特征唯一实现在 `features_ext.py`（backtest.is_holiday 也委托它）。
- **⚠ 负结果1：煤价被数据否决**。原计划 5 新特征（+煤价 2 列）。多窗口消融（3×30 天 out-of-sample）实测：base35=36.89，+lag72+holiday(38列)=36.55(**+0.9%,三窗一致**)，**+coal=37.51(-1.7%)**，all40=38.08(-3.2%)。煤价在日前小时级预测是噪声（训练窗煤价变动小、传导被负荷/西电/新能源吸收、煤电边际定价是周/月级慢变量与日内波动不匹配）。**用户决策去煤价**→定稿 38 列（35+price_lag72+is_holiday_cal+days_from_cny）。简化红利：3 列全是 CSV 纯函数无外部依赖→煤价的 strict fail-loud/ext_defaults/mtime 缓存风险全消。煤价管道保留供可预测性研究（coal_features_for_research），不进模型、未挂 launchd。
- **⚠ 负结果2：Optuna 调参不落盘**。tune_hyperparams.py（3 折时序 CV + 14 天不可见终检窗，best 不劣才落盘）150 trials：best CV 赢默认 5.4%(36.375 vs 38.449)，**终检窗劣于默认 1.9%(37.676 vs 36.576)** → 守卫拒绝落盘。**v3.1 手调超参本已很好**，调参无免费午餐；终检窗是唯一挡住劣质参数的防线。retrain 读 tuned_params.json 存在才用，现不存在→内置默认；回滚=删文件。
- **验证**：backtest 老模型逐日 MAE 回归**完全一致**（meta 切 35=改前）；api_server 改造后同 payload /predict **逐字段一致**（备份+还原了预测存档，负价路径 45 列 AUX 不崩）；38 列候选/35 列 serving 各按 meta 切片可回测（COMPARE 链路通）；干净 A/B 证 38>35 +0.9%；5 新→3 特征重要性 price_lag72 排第 10。
- **上线**：38 列模型手动经 daily_retrain.promote() 上线（07:41，trained_at 已验、feature_count=38）——GATE 因当日金山数据未到（12:00 才同步）拦自动路径，故有人值守手动 promote，复用 daily_retrain 已测 promote（备份/双目录/现代重启/trained_at 精确验证）。清了 state 让今晚 18:00 正常跑。
- **改动文件**：新建 `features_ext.py` `tools/update_coal_price.py` `tools/tune_hyperparams.py`；改 `api_server.py`（meta 切片+AUX+ext特征）`retrain_model.py`（FEATURE_COLS 38+tuned读取+meta 增 hyperparams_source/ext_defaults）`backtest.py`（meta 切片+is_holiday 委托）；`.bak_20260702_232343`。装 optuna 4.9.0。
- **坑/待办**：① `features_ext.HOLIDAYS` **2027 是预填，2026-11 国务院发文后校对**；② 煤价管道无 launchd（手动跑）；③ Optuna 的 recent_window_days（解 RECENT_WEIGHT_START 钝化）随整体不落盘未采纳，钝化仍待解；④ staged_models/ 累积未清（老 P3）；⑤ mini 第二批未部署（清单已标注）。

---

## 2026-07-02 · 路线A第一批：每日自动重训(门槛自动上线) + rMAE/sMAPE 监控

- **背景**：精读论文 Aliyon & Ritvanen《Deep learning-based electricity price forecasting》(Energy 2024)——核心制度=每日滚动重训+rMAE 相对基准评估，「误差的敌人不是波动，是陈旧」。本系统 6/25 事故（陈旧 18 天 MAE 37→123）正是论文结论的实盘验证。方案笔记在 Obsidian `2026-07-02-gdpower每日自动重训与rMAE监控改造方案.md`。
- **做了**：
  - **accuracy.csv 扩 8 列**（+naive_mae/rmae/smape），`tools/backfill_rmae.py` 回填历史 60 行+**重建 07-01**（6/30 同步失败无存档，serving 模型 out-of-sample 重算，MAE 19.3——此行是重建值非生产值）+整表排序（P2 待办闭环）。手算 3 处交叉核验一致。
  - **calc_accuracy**：每日算三个新指标；写入后 `_sort_accuracy_csv()` 整表排序；**rMAE 预警**（最近 3 条非空且 7 天内全 >0.9 → 飞书，`RMAE_WATCH=0.9`）。
  - **retrain_model.evaluate() 加 sMAPE**（分母合计下限 1 元，负价安全），meta 自动携带。
  - **backtest.py 参数化**：`--model-dir/--output/--quiet` + 每日结果带 `naive_mae`（rMAE 护栏数据源），默认行为不变。
  - **`tools/daily_retrain.py`**（launchd `com.gdpower.dailyretrain` 每日 18:00）：GATE→PRECHECK→STAGE①(留30天评估)→COMPARE→DECIDE→STAGE②(全量留2天)→PROMOTE→VERIFY，失败 ROLLBACK。两段式=「交叉验证选方案、全量出成品」，上线工件训练集只滞后 3 天。开关 `DAILY_RETRAIN_ENABLED`（.secrets.json，Air 已开）。
  - **消费端透传**：`/accuracy`（records+`avg_rmae_7`，空值安全转 None）、export_snapshot→PWA data.json。
- **门槛设计的实测教训（重要）**：初版「全窗对比+1.05 容差」实测被泄漏打爆——旧模型在其训练集内日期占 **1.578 倍**便宜（泄漏段 16.64 vs 26.26），公平段其实打平（34.41 vs 34.98，比值 1.017）；且每日 promote 稳态下泄漏段会占满全窗→门槛变永久拒绝机。**改为两级**：公平段（交集中晚于旧模型 train_end）≥7 天→公平段对比；否则→候选全窗回测 vs 生产实际 MAE（accuracy.csv 同窗）。护栏 rMAE≥1 一票否决不变。
- **验证**：全链路真跑一次成功——候选公平段 34.98 vs 34.41(×1.05 内)+rMAE 0.447→promote→API `trained_at=2026-07-02T22:58:23`（独立 curl 复核）→训练集 6/15→6/29（陈旧 17 天→3 天）；4 个辅助 pkl 未动（时间戳佐证）；幂等（当日 done 静默退出）、开关缺省静默、--dry-decide 观察模式、rMAE 告警链路（临时阈值 0.1 逼真推）全过；plutil/py_compile 全绿。
- **改动文件**：`kdocs_sync.py` `retrain_model.py` `backtest.py` `api_server.py` `tools/export_snapshot.py`（均 `.bak_20260702_223826`）；新建 `tools/daily_retrain.py` `tools/backfill_rmae.py` `com.gdpower.dailyretrain.plist` `MINI_DAILY_RETRAIN.md`；`.secrets.json`(+3 键) `.secrets.json.example` 同步。
- **上线时实况**：近 7 日生产 avg_rmae_7=1.087（模型没跑赢 naive）、serving 陈旧 17 天——正在重演 6/25 前夜的模式，本次上线即刻纠正。
- **下一步**：连看 3 天 18:00 日志+飞书；第 2 天临时 TOL=0.5 逼 REJECT 分支验证；mini 按 `MINI_DAILY_RETRAIN.md` 部署（rMAE 监控必上，重训开关自定）；第二批=特征改造（节假日/煤价/lag72）+Optuna。
- **坑**：① 群里 22:48 的「rMAE预警」是测试消息（阈值临时 0.1），忽略。② accuracy.csv 07-01 行是重建值。③ staged_models/ 每日 +2 个目录（评估+全量），未加清理，累积待清（老 P3 待办升级）。④ update.sh 每日 Step2 重启 API 会加载 serving——promote 已直接写 serving，语义一致无冲突。⑤ 门槛对比用回测口径（真实历史特征），与生产口径（D-1 预报特征）系统性偏低约 2 倍，跨口径比较时注意。

---

## 2026-06-23 · 补「评判器闭环」：模型退化自动告警 + 条件触发重训（上线仍人工）

- **背景**：用「循环工程(五动作×六零件)」审查这套自动化——调度/发现/持久化/视野/交付五格都到位，**唯独缺「验证」那格的执行端**。系统每天 `calc_accuracy()` 拿次日真值回测算 MAE（强 ground-truth 检查），但命中阈值此前**只 `log.warning` 不动作**：误差攒在 `accuracy.csv` 没人盯，**6/19、6/22 两天 MAE 飙到 200+ 都没主动喊人**。典型「验证债」——有测量、没评判器闭环。
- **目标**（用户确认范围=「告警 + 条件触发重训」）：把「测量」接到「动作」。**P0 退化自动告警**(自动)、**P1 条件触发重训出候选**(自动)、**上线仍人工把关**(守住循环工程要求的人工卡点)。
- **做了**：
  - **P0 告警 `kdocs_sync.py::calc_accuracy()`**：新增 `notify_quality()` + `_recent_mae_mean()`。单日 MAE≥`DEGRADE_MAE`(70)→推「模型退化」；无退化但近 3 日均值>`WATCH_ROLL3`(55)→推「模型注意」。复用 `_feishu_push`+`should_push` 主备去重，文案含「广东电力」过机器人关键词。去重靠 `existing_dates`（同日只算一次）+ `should_push`（双机）。
  - **`retrain_model.py` 加两个加性 CLI**（默认手动用法字节不变）：`--test-start DATE` 覆盖 `TEST_START_DATE`（破「假重训陷阱」让新数据进训练集）；`--stage` 把候选只写 `staged_models/<ts>/`、**绝不碰 serving 目录**。打印机读标记 `TRAIN_END=` / `STAGED_DIR=`。meta 构建上提供两路复用。
  - **P1 闭环 `tools/verify_sync.py::auto_retrain_check()`**：近 7 日 MAE 均值>`RETRAIN_MAE_THRESHOLD`(70) → 先 `--dry-run --test-start today-7` **校验 TRAIN_END 真推进**(≤10 天内，否则疑似假重训→推「需人工介入」中止) → 再 `--stage` 出候选 → 推「候选就绪，待人工 curl /model_info 后上线」。三道护栏:`AUTO_RETRAIN_ENABLED` 仅主力机开(双机去重)、`RETRAIN_COOLDOWN_DAYS`(5) 冷却期、候选绝不自动上线。
- **关键安全点**：update.sh 每日 Step2 重启 API 会加载 serving 目录里的模型——所以候选**必须**暂存在 serving 之外，否则次日静默自动上线。已实测 `--stage` 后 serving+主 models 目录 mtime 指纹**完全一致**(byte-for-byte 未动)。
- **验证**：P0 三场景(退化/注意/正常)+去重 ✅；retrain 默认 dry-run 暴露 TRAIN_END=2026-04-03 假重训陷阱 ✅；`--stage --test-start` 推进到 06-15 且 serving 未被触碰、候选完整(xgb/lgb/meta) ✅；P1 编排 5 场景(阈下不动/假重训中止/出候选/冷却期/非主力机) 全 mock 通过 ✅。三脚本 `py_compile` 均 OK。
- **改动文件**：`kdocs_sync.py`、`retrain_model.py`、`tools/verify_sync.py`、`.secrets.json.example`(新增 4 个可选键)。备份 `*.bak_20260622_235231`。
- **部署姿态**：P0 告警**改完即生效**（无需配置）。P1 自动重训**默认休眠**(`.secrets.json` 无 `AUTO_RETRAIN_ENABLED` → false)，**待用户在主力机一台显式开启**。两机当前都不会自动重训，安全。
- **坑/待办**：① 自动重训默认关，要用须在【主力机一台】`.secrets.json` 加 `"AUTO_RETRAIN_ENABLED": true`，**切勿两台都开**(抢写模型目录)。② dry-run 也会真训练(只是不保存)，故条件重训会训两次(dry+stage)——冷却期 5 天内最多一次，可接受。③ promote 上线仍手工：`cp staged_models/<ts>/*.pkl <serving>/models/` + 重启 API + `curl /model_info` 验 `trained_at`。④ `staged_models/` 未加自动清理，候选会累积，后续可加按时间清。

---

## 2026-06-20(续) · PWA 双机互备 + 健康告警：去掉「仅 Air 发」单点

- **背景**：PWA 发布是单点故障——`update.sh:141` 用 `if [[ "$(whoami)" == "hydtzyj" ]]` 网关，**只有 Air 发 PWA**，mini 只同步预测从不发。Air 宕机/断网 → 手机 PWA 永远停更且无人知。两机原本是「伪互备」（数据各算各、仅飞书群去重、无故障接管）。
- **目标**（用户确认）：PWA 发布也做成**真双机互备**——两台都能发，谁先算好谁发，一台宕了另一台顶上；加「到点没更新就飞书告警」。
- **核心机制**：用 GitHub 上 `gdpower-pages` 的 `origin/main:data.json` 当**两机共享协调状态**（`git fetch` 读，绕 Pages CDN 缓存）。实现抢发幂等。
- **做了**：
  - **update.sh Step 3.4 去 whoami + 抢发**：发布前内联 Python（ASCII，避 zsh 中文 heredoc 坑）`git fetch`+`git show origin/main:data.json`，若远端 `pred_date==本机PRED_DATE` 且 `generated_at` 是今天 → SKIP；否则 PUBLISH；读不到远端 → FORCE（fail-open 照发）。
  - **publish_pages.sh 并发稳妥化**：rebase 冲突从 `abort+exit1` 改 `rebase -X ours origin/main`（实测：rebase 语境 ours=已落地的 origin/main=先发先得，自动解不丢数据）+ 3 次随机退避 + push 被拒重推 + 终态 exit 0。**注意 rebase 的 ours/theirs 方向与 merge 相反**（实测确认：-X ours 留先 push 的、-X theirs 留后 push 的）。
  - **新增 `tools/check_pwa_health.py`**：14:30 由 launchd 触发，`git fetch` 读远端 data.json，若非「今天发、pred=明天」→ 飞书告警（含「广东电力」+marker「PWA健康」）；去重复用 `ks.should_push('PWA健康')`。用 `Path.home()` 自动适配两机路径。
  - **新增 `com.gdpower.pwahealth.plist`**（14:30）。
- **验证**：抢发 SKIP|Air / PUBLISH✅；publish 并发隔离实测 -X ours 自动解+零冲突标记+不卡死✅、真实 no-op 退出0✅；健康检查告警/静默/去重/读不到远端 四场景✅；现是今天数据→健康检查判正常静默✅；api 常驻 running、三定时任务登记、pages 历史无污染✅。
- **改动文件**：`update.sh`(Step3.4)、`tools/publish_pages.sh`、新增 `tools/check_pwa_health.py` + `com.gdpower.pwahealth.plist`。备份 `*.bak_20260620_224605`。
- **范围外/待 mini**：mini 要变成「能发 PWA」需三前提——本地 clone `~/gdpower-pages`、有 push 凭证、装新版代码+plist（路径改 zhouyijun）。出了 `MINI_PWA_HA.md` 清单。**mini push 凭证不通则 PWA 仍单点**。
- **坑/备注**：`git rebase -X ours/theirs` 方向反直觉（rebase 时 ours=目标分支），上线前务必实测；读远端状态用 `git fetch+show` 而非 curl（绕 CDN 缓存）；告警文案必须含「广东电力」过白名单。

---

## 2026-06-20 · 代理兜底（kdocs_sync v6.1）：绕 Shadowrocket TUN 直连 + launchd 多点 + 告警节流

- **背景**：今天 12:00 自动同步全天失败、且无人知晓。复盘三个真 bug：① 撞上 Shadowrocket 隧道半死，连国内 `www.kdocs.cn` 都被 `SSLEOFError` 打断；② `poll_and_sync()` 靠进程内 `time.sleep(1800)` 轮询，MacBook Air 合盖睡眠时 sleep 跨睡眠不可靠 → 后续轮询全废；③ 进程睡死，18:00 的飞书告警也没发出。先手动 `update.sh` 补回了今天数据（pred 6/21 均价 370.4）。
- **关键发现**：Shadowrocket 是 **TUN 隧道模式（虚拟网卡 + 网络层路由劫持）**，不是普通系统 HTTP 代理。陷阱9 的 `trust_env=False` / `--noproxy`（应用层绕过）对它**无效**——实测 `curl --noproxy "*"` 出口仍是美国。
- **做了**（用户确认「探测 → 坏了先尝试绕过 → 绕不过再重试告警」+「2 轮失败才告警」）：
  - **隧道探测**：`proxy_tunnel_healthy()`，经 `127.0.0.1:1082` 查 ip-api 出口是否 US（口径同 srmonitor）。
  - **直连兜底**：`fetch_via_direct()` —— 公共 DNS（223.5.5.5 等）解析 kdocs 真实 IP + `curl --interface en0` 绑物理网卡发 POST，**流量从物理 Wi-Fi 出口走、彻底绕开 utun4 隧道，全程不碰 Shadowrocket**（避免与 Guardian 打架、不影响日常翻墙）。`--interface en0` 是必须的：否则系统会给真 IP 建 `真IP→utun4` 主机路由仍进隧道。已实测 POST 拿到 168 行。接进 `run_sync()`：正常通道失败且隧道坏才走兜底；隧道好（纯服务端 500）维持 FETCH_FAIL 交重试。
  - **修轮询**：默认入口由 `poll_and_sync()` 改 `run_once_for_launchd()`（单轮，删 sleep 循环）；`com.gdpower.update.plist` 单点 12:00 → **12–17 点 6 个 `StartCalendarInterval`**，睡眠错过点 macOS 醒来补跑。旧轮询保留为 `--poll`。
  - **告警节流**：跨进程按日状态文件 `~/gdpower/.sync_state_<date>.json`（原子写），连续 `ALERT_THRESHOLD=2` 轮失败才推一次飞书（含「广东电力」关键词），避免代理短暂抽风虚惊；恢复时附一条「已恢复」。
- **验证**：隧道探测 US✅、en0 直连绕隧道拿 168 行✅、告警节流（第2轮才推、仅1条、含「广东电力」）✅、好日子短路秒退✅、全程出口仍 US / Shadowrocket 未被碰✅、py_compile + plutil + launchctl 6 点登记✅。
- **改动文件**：`kdocs_sync.py`(v6.1，+subprocess、+探测/直连/状态/单轮入口、改 run_sync 与 __main__)、`com.gdpower.update.plist`(多点)。均已备份 `*.bak_20260620_214534`。
- **范围外（待后续）**：Mac mini（13:00 老逻辑）、外部 `gd_price_forecast_pkg`（22:30/23:30，token 明文）有同样 TUN 隐患，本期未动。
- **坑/备注**：TUN 模式下应用层绕代理无效，必须 `--interface en0` 绑卡 + 公共 DNS 真 IP；CDN 多 IP 会轮换不可硬编码、保持证书校验（勿加 `-k`）；笔记本 `time.sleep` 跨睡眠不可靠，定时轮询应交 launchd 多点而非进程内 sleep。

---

## 2026-06-17 · 手动补发 6/16 预测图 + 对比图到飞书

- **做了**：应要求手动生成并推送 6/16 两张图到飞书群——`notify_prediction.py --date 2026-06-16`（日前电价预测 24 点折线图 + 数据表）、`notify_compare.py --date 2026-06-16`（日前预测 vs 实际对比折线图 + 数据表），各一条 post 带双图，退出码均 0。
- **要点**：Air 现为 backup，手动场景下临时 `ks.should_push = lambda *_: True` **绕过去重守卫强制推送**（只在这次手动运行内存里生效，未改文件/角色）。消息带 `[Air]`。
- **改动**：无（只跑现有脚本）。PNG 存档在 `~/gdpower/predictions/`（chart_/compare_ 2026-06-16）。
- **备注**：当天 mini(primary) 11:10 自动跑会再推一次当天图，属正常 primary 行为；Air 自身 12:00 backup run 会读群发现已有而跳过。

---

## 2026-06-16 · Air 配成 backup（双机主备去重）+ 修「已推送」误报

- **背景**：mini 今天上线「双机主备去重」并自设 primary（11:10 同步 / 11:30 核对、总推飞书）。本机 Air 要配成 backup（晚跑、先读群看 primary 推没推、没推才顶上），双机主备才全线生效。**关键**：mini 真实源码并未传到 Air（`~/Downloads/mini-deploy/` 是 6/16 00:02 的旧 Air→mini 反向包，无 should_push），与用户确认走 **Path B**——按 `AIR_备机配置.md`＋`DEVLOG-mini.md` 规格在 Air 现有代码上**新增**主备逻辑，保留 Air 全部专属改动（`trust_env=False` 直连 / 表格图 / PWA 钩子）。

### 任务 A：Air→backup
- **kdocs_sync.py 新增**（未动任何现有函数）：读 `NOTIFY_ROLE`（缺省 primary）/`FEISHU_CHAT_ID`；三函数 `_feishu_today_message_contents`（自建应用读今日本群消息，`GET /im/v1/messages`，复用 `_feishu_tenant_token`＋`_SESSION` 直连，读不到→None）/`already_notified_today`（子串匹配，机器标签无关，返 bool|None）/`should_push`（silent→不推；primary→总推；backup→已有跳过、没有顶上、读不到 fail-open 照推）。`notify_success` 套 `should_push('金山同步成功')` 守卫；`notify_failure` 不守卫（两机都失败都该告警）。
- **守卫**：`tools/notify_prediction.py`→`should_push('次日日前电价预测')`、`tools/notify_compare.py`→`should_push('预测 vs 实际')`、`tools/verify_sync.py`→import ks 后 `should_push('同步核对')`。4 个 marker 与现有推送文案逐条核对一致。
- **角色**：`.secrets.json` 加 `NOTIFY_ROLE=backup`＋`FEISHU_CHAT_ID=oc_…8daf`（`FEISHU_APP_ID/SECRET` 本就有，两机共用）。
- **错峰定时**：launchd 实际加载的是 `~/Library/LaunchAgents/`（非 `~/gdpower/`），两处都改并 bootout/bootstrap 重载——update 11:00→**12:00**、verify 11:20→**12:20**（晚于 mini，保证读群时 primary 已推完）。`launchctl print` 确认 Hour=12 已登记。
- **✅ 读群去重已打通（当晚解决）**：排障三连——① `99991672`：`GET /im/v1/messages` 真正需 `im:message.history:readonly`（文档写的 `im:message.group_msg` 不够）；② 加 scope 发版后变 `230002`「Bot 不在群里」——发现旧共用应用 `cli_aabab854f3785bc4` 在**另一账号**下、跨账号进不了「中海油电力投资有限公司」群；③ 在**群所在账号**新建自建应用 **`cli_aabbd2a70038dbd8`**（机器人能力 + `im:message.history:readonly` + `im:resource`，已发布 + 已入群），把 Air `.secrets.json` 的 `FEISHU_APP_ID/SECRET` 换成它。实测：token ✅、列群确认 chat_id=oc_…8daf ✅、读到今日 68 条 ✅、4 marker 全 `False`（primary 已推→跳过）✅、图片上传 ✅（发图能力完好）。**代码零改动，仅换凭据。**

### 任务 B：修「已推送」误报
- **根因**：`notify_prediction`/`notify_compare` 的 `main()` 降级分支（图分条 / 纯文本兜底）忽略发送函数返回值，无脑 `print('已推送')`＋`return 0`，底层失败也假成功（exit0 假成功变种，CLAUDE.md 陷阱9）。底层 `_feishu_send/_feishu_push_post/_feishu_push_image` 本就返真实 bool，无需改。
- **修复**：两文件对称——降级分支先全发再 `all(img)` and `post_ok` 判定，真成功才打印「已推送」return 0，否则打印「推送失败…」`return 2`；纯文本兜底同理。只在确实成功打印「已推送」。`update.sh` **无需改**（line 135–139 推送钩子早是 `… || warn "…不影响主流程"`，退出码修好后这层 `|| warn` 才真正生效、主流程不中断）。

- **验证**：✅ 4 个 py `py_compile` 全过；✅ 任务B monkeypatch 模拟推送失败——两文件各 3 场景（纯文本兜底失败 / 降级图分条失败 / 成功）全过：失败 `ret=2`＋不打印「已推送」、成功 `ret=0`＋打印「已推送」；✅ dry-run 两图正常出、`ret=0`、不发飞书；✅ launchd 登记 12:00/12:20。
- **改动文件**：`kdocs_sync.py`、`tools/notify_prediction.py`、`tools/notify_compare.py`、`tools/verify_sync.py`、`.secrets.json`、`com.gdpower.update.plist`（两处）、`com.gdpower.verify.plist`（两处）。`update.sh` 不改。各文件改前均 `.bak_时间戳` 备份。
- **下一步**：①（可选）mini 那台把 `.secrets.json` 的 `FEISHU_APP_ID/SECRET` 也换成新应用 `cli_aabbd2a70038dbd8`——mini=primary 不读群、当前用旧 app 发图仍可用，但旧 app 在另一账号、为防其失效 + 未来角色互换，建议统一换新 app；②可选从 mini 真发一次确认双机错峰协同。
- **坑**：①读群真正需要的 scope 是 `im:message.history:readonly`，不是文档/DEVLOG-mini 写的 `im:message.group_msg`；②**自建应用必须和群同账号/同租户**才能入群读消息——旧共用 app 在另一账号、跨账号入不了群（`230002`），解法是在群所在账号建新 app；③`_feishu_upload_image` 与读群共用 `FEISHU_APP_ID/SECRET`，换 app 时新 app 要同时有 `im:resource`（已加，发图正常）；④launchd 权威 plist 在 `~/Library/LaunchAgents/`，`~/gdpower/` 那份只是源副本，改定时两处都改并从 LaunchAgents 重载；⑤`should_push` 设计为 fail-open（读不到照推），保证去重故障时「宁可多推不可漏推」。

---

## 2026-06-16 · 模型 Tab 特征名中文化

- **做了**：模型 Tab「Top10 特征重要性」英文特征名改中文。前端加 `FEAT_CN` 映射（覆盖全部 35 特征，已是中文的保持原样），`featCN()` 在渲染时翻译，原名留 title 便于核对。如 price_lag1→前1时电价、price_lag24→昨日同时电价、price_lag168→上周同时电价、price_roll24_mean→近24时均价、net_load→净负荷、hour_cos→小时(余弦)。
- **改动**：仅 `~/gdpower-pages/`：index.html（FEAT_CN + renderModel）、sw.js v4→v5。data.json/export 不变（映射是纯展示层）。
- **验证**：✅ 语法过；✅ 桩接渲染 Top10 全部中文、无残留英文；✅ 线上 sw=v5、index 含映射。

---

## 2026-06-16 · 对比图去重：只留准确率 Tab

- **背景**：上一步把对比图同时放进预测 Tab 与准确率 Tab，内容完全重复。对比是「复盘/准确率」内容（对的是过去有实际值的日子），预测 Tab 专注明日预测，故只留准确率 Tab。
- **做了**：移除预测 Tab 的 cmpCard2 + 画布；`CMP_MOUNTS` 退回单挂载点（泛化结构保留便于以后增减）；`sw.js` v3→v4。
- **验证**：✅ 语法过；✅ 桩接确认只创建 predChart/accChart/cmpChart（无 cmpChart2）；✅ 线上 cmpCard2=0、准确率对比卡=1、sw=v4。
- **改动**：仅 `~/gdpower-pages/`：index.html、sw.js。

---

## 2026-06-16 · 对比图也加到预测 Tab（双 Tab 共享选中日）

- **做了**：预测 Tab 底部也放一份「日前预测 vs 实际」对比图。把 renderCompare/drawCompare 泛化为多挂载点（`CMP_MOUNTS`），准确率 Tab 与预测 Tab 共享同一 `CMP_SEL`，切一处两处同步；每挂载点独立 chart 对象（`cmpCharts` 按 canvas id）。
- **改动**：仅 `~/gdpower-pages/index.html`（加 cmpCard2 + JS 泛化）、`sw.js`（v2→v3）。data.json 的 compare 块不变，export 脚本无改动。
- **验证**：✅ 语法过；✅ 桩接首渲染创建 predChart/accChart/cmpChart/cmpChart2，切日期同步重绘两图、destroy=2；✅ 线上 index 含 cmpCard2、sw=v3。

---

## 2026-06-16 · 手机 PWA 看板新增「日前预测 vs 实际」对比图（可切换最近几天）

- **做了**：准确率 Tab 顶部加对比折线图，带最近 7 天日期切换 pill，默认最新一天。两线（预测蓝/实际绿）+ 偏差红阴影 + 最大偏差点标注 + 指标卡（MAE/MAPE/评级/最大偏差/预测均价/实际均价/整体方向）。
- **数据**：`export_snapshot.py` 新增 `build_compare(days=7)`，复用 `notify_compare._load/_metrics`，data.json 加 `compare` 块（只收既有预测又有实际的日期）。
- **前端**：`index.html` 加对比卡 + `renderCompare/drawCompare`（切日期内存重绘、destroy 旧 chart）；`sw.js` 缓存版本 v1→v2 强制刷新。
- **验证**：✅ JS/sw 语法过；✅ 桩接跑通（3 图表实例、切日期 destroy 正常）；✅ 重生成 data.json 含 compare 7 天（最新 6-16 MAE 67.2、整体低估 56.6，与飞书对比图一致）；✅ 发布后线上 compare=7、index 含对比卡。
- **改动文件**：`tools/export_snapshot.py`（gdpower 仓）；`~/gdpower-pages/`：index.html、sw.js、data.json。

---

## 2026-06-16 · 修复对比图漏发飞书（系统代理 503）

- **现象**：今天日前 vs 实际对比图没发到飞书；同一次运行的预测图发成功了。
- **根因**（systematic-debugging）：`kdocs_sync.py` 的 requests 调用未设 proxies，macOS 上 requests(trust_env=True) 经 `getproxies_macosx_sysconf()` 读系统科学上网代理 127.0.0.1:1082（无任何环境变量也会读）。飞书/金山是境内服务却走该代理；11:32 代理瞬时 503 → 图片上传+webhook 全失败。notify_prediction 早几秒赶上代理正常，故只丢对比图。冒烟枪：`urllib.getproxies()` / `get_environ_proxies('https://open.feishu.cn')` 均返回 1082。
- **修复**：`kdocs_sync.py` + `tools/verify_sync.py` 改用 `_SESSION = requests.Session(); _SESSION.trust_env=False`，所有 requests.post/get 换成 `_SESSION.*`，绕开系统代理直连。
- **验证**：✅ py_compile 过；✅ 证明修复后会话对飞书解析代理为空(直连)、旧行为仍解析到 1082；✅ 重发 2026-06-16 对比图，打印「飞书已推送（折线图+表格图 一条 post）」(仅真成功才打印)、退出码 0、飞书群已收到。
- **改动文件**：`kdocs_sync.py`（5 处调用，已备份）、`tools/verify_sync.py`（1 处，已备份）；CLAUDE.md 加陷阱 9。
- **遗留待办**：notify_compare/notify_prediction 降级分支 `print('已推送')` 是误报（底层失败也照打、main() 仍 return 0）→ 漏发无告警，待改为按真实推送结果置退出码，让 update.sh `|| warn` 能触发。

---

## 2026-06-16 · 手机只读 PWA 看板上线（GitHub Pages）

- **做了**：完成 CLAUDE.md 的 P3「手机 APP（PWA）」。手机只读看板，三视图＝次日预测曲线+指标卡+交易提示 / 准确率近30天趋势+汇总 / 模型信息+Top10特征。
  - 新建 `tools/export_snapshot.py`：每日把 `prediction_*.csv`+`accuracy.csv`+`model_meta_v2.json`+近7天历史组装成 `data.json`，复用 `notify_prediction._load_pred/_metrics` 与 `kdocs_sync.MACHINE`；accuracy 状态阈值与 api_server `/accuracy` 同口径（>70退化/>55注意）。
  - 新建 `tools/publish_pages.sh`：照搬 `sync_devlog.sh` 模式 push 到 `~/gdpower-pages`（远端无 main 时跳过 rebase，首推用 `-u`）。
  - 新建仓库 `gdpower-pages`（public，已开 Pages）+ 移动优先 PWA（`index.html` 深色电网终端风 + Chart.js + manifest + sw.js 离线缓存 + 闪电图标）。
  - `update.sh` 加 **Step 3.4**（139行后、预测成功分支内）：`whoami=hydtzyj` 网关 → 仅 Air 发布，失败只 warn 不阻断主流程。
- **状态**：✅ data.json 校验四块齐全、数值与飞书图一致；JS/sw.js/manifest 语法过 + node 桩接跑通 render（2图表）；首推成功、Pages 构建完成、线上 index/data.json/manifest/sw/icons 全 200；线上抽检 pred_date/均价/机器/acc/model 正确。
- **线上地址**：https://kingswoodjesse61528-netizen.github.io/gdpower-pages/
- **改动文件**：新增 `tools/export_snapshot.py`、`tools/publish_pages.sh`；改 `update.sh`（已备份 `update.sh.bak_20260616_003803`）；新建 `~/gdpower-pages/` 仓库。
- **下一步**：手机实测视觉 + 「添加到主屏」+ 飞行模式离线；mini 要发只需去掉 whoami 网关复用同 hook。
- **坑**：空仓库首推时 `git pull --rebase origin main` 会因远端无 main 报错 —— publish_pages.sh 已加 `ls-remote` 判断跳过。

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

## 2026-06-25 · 模型重训上线（Air，退化修复）

- **背景**：模型自 6/7 起 18 天没重训、训练数据停在 5/31，错过整个 6 月新形态。近 7 日 MAE 均值 ≈123（远超系统重训阈值 70），6/19、6/22、6/23、6/25 单日 MAE 破 70 频繁告警。`.retrain_state.json` 不存在=历史从没真正触发过重训；本机 `NOTIFY_ROLE=backup`、`AUTO_RETRAIN` 关，故人工触发。
- **做了**（纯重训上线，无代码改动）：
  - `retrain_model.py --test-start 2026-06-16`：dry-run 确认训练集推进到 6/15（含 6 月上半月数据），再正式重训
  - 旧模型自动备份到 `models/v2_backup/20260625_191721`，新模型保存并同步 API 目录
  - `install_service.sh restart` 重启 API
- **结果**：新模型 `trained_at=2026-06-25T19:17:21`，`curl /model_info` 确认真上线。测试集（6/16–6/25）MAE **37.69** / MAPE 10.39%，对比旧模型同期实际 ~120 → 基线大幅改善；负价 MAE 0。明日 6/26 预测均价 471.7，曲线合理已存档。
- **下一步**：盯 1~2 天 accuracy.csv，近 7 日 MAE 应回落到 70 以下，否则回滚（`v2_backup/20260625_191721`）。**mini 主力机大概率同样陈旧、待重训** → 清单 `MINI_RETRAIN_20260625.md`。
- **坑/备注**：高价尖峰 MAE 反升到 82（旧 34），是模型结构问题、重训不解决，记 P3「分时段建模/误差修正层」。本次会话工具输出一度遭注入污染、伪造过工具结果，已识别忽略，所有落盘项均用 Read/git 复核为准。

---

<!-- 历史记录继续往下追加 -->
