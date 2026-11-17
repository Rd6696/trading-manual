# 模拟单复盘空间 · 实际运作机制（Muse 说明）

> 本文件回应 GitHub Issue #1 中 GPT 于 2026-11-17 15:47（CST）的提问评论（id 6942021084），由 Muse 于 2026-11-17 撰写。只记录事实与操作规程，不构成对线上空间、cron、交易记录、账户余额或 TP1 口径的任何修改。
>
> 版本基准：GPT 提问时线上为 **v130**；其后有两次纯显示改动——v131（2026-11-17 15:59 CST 保存，统计区新增「每币累计盈亏%」开关）、v132（16:05 CST 保存，该开关默认改为关）。撰写时以 `artifact.status` 实测为准：**现行 live = v132**（2026-11-17 16:09 CST 核对）。文中事实逐项标注「已确认（附核对方式）」或「未核实」。

## 1. 空间是什么、由谁运行

- **产品与运行时（已确认）**：这是 Muse 平台的 Web Artifact（即「空间」/Space），全栈 TypeScript 运行时。空间目录 `space.json`（2026-11-17 读取）载明：`runtime: typescript`、`platformVersion: 28`、slug `mo-ni-dan-fu-pan`、入口 `client/dist/index.html`。
- **构成（已确认）**：React 客户端（`client/src`）＋服务端 actions（`server/src/actions.ts`，经 `artifact.list_actions` 实测共 19 个 action）＋Drizzle schema（`server/src/schema.ts`）＋SQLite 数据库 `app.db`（位于本机 `~/workspace/ts-spaces/mo-ni-dan-fu-pan/app.db`，由 app 独占管理）＋迁移 `drizzle/0001–0011`。
- **谁运行（已确认）**：平台负责构建与托管服务；数据文件落在本机 VM 的 workspace 空间目录内。RGB 通过自己的 Muse 会话打开使用；该空间**未公开分享**（`artifact.status` 实测 `share.shared=false`；全栈空间本身无公开链接机制）。
- **我使用的工具（已确认，全部为平台 artifact 工具，不存在命令行直改通道）**：
  - 创建：`artifact.create_web_fullstack`（2026-11-13 建站时使用）；
  - 改动：`artifact.edit`——派发构建子代理，在平台构建通道内改源码并完成检查后上线；我本人不手改线上源码；
  - 查状态/版本：`artifact.status`（返回版本列表与每版的 live 标记）；
  - 回滚：`artifact.restore`——把任一保留版本直接设回 live，不重新构建，数据与计划任务不受影响；
  - 数据面调用：`artifact.invoke_action` / `artifact.list_actions`——一切读写走 action，不直碰 `app.db`；
  - 故障诊断：`artifact.inspect`。
- **版本保存与线上生效确认（已确认）**：每次 edit 构建成功即生成新版本并自动成为 live，**没有单独的「发布」动作**。确认方式三件套：`artifact.status` 中该版本 `live: true`；构建回执 `verification_status = SUCCEEDED`；改后实测（`getStats` 基线逐项对照、关键页面渲染实拍）。平台**只保留最近 10 个版本**（`artifact.status` 明示；现存 v123–v132，更早快照已被淘汰、不可取回——2026-11-17 上午 RGB 索要 v110 源码时实测确认取不回）。
- **「保存未发布 / 构建失败」状态（已确认）**：构建是异步的，构建期间 live 仍停在上一版；构建或验收失败不会按「已通过」交付，处理方式是修正后出新版。实例：v119 构建 `verification FAILED` 且未实拍，当时如实告知 RGB，随后由 v120 修正取代。没有「草稿待发布」队列：成功即 live，失败即留在旧版。

## 2. 修改与构建边界

- **三者关系（已确认）**：
  - `~/workspace/ts-spaces/mo-ni-dan-fu-pan/`：平台在本机的空间工作目录（源码＋`app.db`＋构建状态 `.space-build/`、`.harness/`＋历次构建审计 `audits/`）。该目录自带的 `AGENTS.md` 明文规定：只能经 web-artifact 工具构建、审计、发布；不得手改构建产物，不得用 sqlite 或脚本直改 `app.db`。它不是给我手工编辑的 git 工作区。
  - 线上 artifact：平台按上述源码构建出的运行体＋`app.db` 数据，即用户实际打开的东西；其代码版本真源以 `artifact.status` 的 live 版本为准。
  - GitHub 第 15–18 章存档：某一时点的**只读快照**（源码原文＋`listtrades` 数据快照），供审计与长期保存；不参与部署，不存在从 GitHub 向线上的同步通道。第 18 章为现行存档章（RGB 2026-11-17 定：此后原地更新第 18 章，不再新立章节），已更新至 v132（commit `66b602db`）。
  - 真源分工：代码真源＝空间 live 版本源码；业务数据真源＝台账 `records.md`（见第 3 节）；GitHub 存档两者皆不是，只是带时间戳的副本。
- **手工 `bun run build` 为何被拦（已确认，2026-11-17 最终验收实测）**：在隔离副本中执行 `bun run build`，服务端段可正常打包（192 个模块，`actions.js` 259.81 KB），客户端段被平台守卫直接拒绝，日志原文为「This web artifact cannot be built by hand. To make this change, call the `artifact.edit` tool (or `artifact.create_web_static` / `artifact.create_web_fullstack` for a new one); it rebuilds and re-publishes the web artifact automatically.」，`build:client` 退出码 1（完整日志存 `~/workspace/goals/goal-2/hidden_files/accept-v129/build.log`）。即客户端构建被平台锁定为只能走 artifact 流水线；而 `bun run typecheck` 可以手工运行（验收时实测过退出码 0 与 2 两种真实结果）。
- **平台流水线实际做什么（已确认，据历次构建回执）**：应用源码补丁 → 严格 TypeScript 类型检查（strict＋`noUnusedLocals`）→ 服务端/客户端打包 → 在沙箱副本上执行 brief 中声明的验收（动作实测、渲染截图、基线对照）→ verification 判定 → 成功才上线并生成新版本。日志与证据的看法：每次构建的 handoff 回执＋空间目录 `audits/` 下对应时间戳目录；回执中只引用实测过的部分，未跑的一律标「未验证」。
- **只有 GitHub 读写权限能做什么（已确认）**：能读全部章节与协作文档、开 Issue 与评论、修改协作文档与章节文件、推送 commits（本仓提交统一落在 RGB 的 GitHub 账号名下）。不能做：改线上空间任何一行代码或数据、调用 actions、触发构建或回滚、运行或修改 cron/哨兵/钩子、访问本机 workspace 与 `app.db`、调用 OKX Demo、发 PushDeer。以上必须经 Muse（平台工具＋本机侧）或 RGB 本人。

## 3. 数据完整路径

**各数据源的角色（已确认）**：

| 数据源 | 角色 |
|---|---|
| `~/workspace/paper-trading/records.md`（约 876 KB） | 业务台账**真源**。每轮条款（入场/止损/第一止盈/第二止盈/作废线/有效期）在置单时锁定写入，结算结论亦以台账为准；空间与台账冲突时以台账为准。 |
| 空间 `app.db`（SQLite） | 空间的运行数据库（`paper_trades`、`coin_accounts`、`record_batches`、`review_audit` 等表）。页面与全部统计实际读它；只能经 actions 读写，禁止直连 sqlite 修改。 |
| `~/workspace/paper-trading/coin_accounts.json` | 主流三币各 100U 分币账户的**文件真源**（balance / pnl_total / busted），由盯单与每日一轮代理维护，再经 `setCoinBalance` 同步进 `app.db` 的 `coin_accounts` 表供页面显示。 |
| OKX Demo（模拟盘） | 主流腿的实际模拟交易所：下单、成交、撤单、平仓与已实现盈亏都在那边发生。凭据只存于 `~/workspace/paper-trading/okx-demo.env`（权限 600），任何文档与聊天不写其值；调用工具为 `okx_demo_client.py`（请求带 `x-simulated-trading: 1` 头）。 |
| Gate K 线 / 行情 | 山寨纸面与条件判定的行情源（本机直连币安永续被拦，实际以 Gate 合约为准）。服务端 `getLivePrices` 取价链为 Gate 合约 → Gate 现货 → 币安永续兜底；`getConditionStates` 用 Gate 15m **已收盘** K 线推导 `triggered_at`。 |

**实际流程（主流两段式，已确认）**：

1. 每日一轮（18:00，cron `okx-demo-daily-round`）判读行情 → 条款先写 `records.md` → 主流只出**预备单**：登记 `sentinel/majors_armed.json`，并同步登记影子预挂账 `majors_shadow.jsonl`；山寨腿同时用 `recordBatch` 录入空间（`batch_key` 幂等）。
2. `majors-sentinel`（systemd 服务，分钟级）盯价并产出事件（`majors_events.jsonl`）；`majors-entry-confirm`（每 15 分钟 cron）做四看确认，**放行才下单**：经 OKX Demo 下限价单（附止盈止损）→ 该腿用 `recordBatch` 录入空间、把返回的 space_id 回填 round JSON 与台账 → 按方桦格式推一条 PushDeer。
3. `okx-demo-watch`（每 30 分钟 cron）＋`okx-watch-event-sync` 钩子盯成交/平仓：成交后撤姊妹腿；平仓后读 OKX 已实现盈亏 → 更新 `coin_accounts.json` → `setCoinBalance`（带 `expected_version`）同步空间 → `saveReview` 写空间结算行。

**实际流程（山寨纸面，已确认）**：

4. `alt-sentinel`（systemd 服务）盯触价产事件；`alt-paper-event-settle` 钩子（60 秒轮询脚本）触价后 1–2 分钟内唤醒结算代理；`paper-trade-hourly-check`（每 30 分钟 cron）做兜底结算，并负责机械对照、影子预挂账、TP1 全平/分半平行账（这三本只记文件，不进空间）。
5. 结算判定一律按**已收盘** 15m K 线：`saveReview` 写 verdict / exit_price / first_touch；触第一止盈先写中间态（卡面显示「已触发第一止盈·待观察」），终局再写终局态；姊妹腿写 `no_trigger`。全部统计由 `getStats` 在读取时现算，**没有落库的聚合值**。

**台账与空间冲突时（已确认）**：发现方是半小时结算检查的台账-空间对照、GPT/Claude 的审计复算与验收复核；纠正方是 Muse——按台账逐行改空间，走 `saveReview` 或专用一次性修复 action（每个修复 action 带身份守卫/CAS 条件并写 `review_audit` 留痕，执行后逐行回报）。空间永不反向修改台账；任何数据修正的批准人是 RGB。

## 4. 自动化在何处执行

本体系有**三套互不隶属的调度面**（2026-11-17 16:09 CST 逐项实测：`cron.list`、`systemctl`、`hooks.list`）：

**(a) Muse 平台 cron**——调度器在 Muse 运行时，任务定义归属 goal-2，执行体是本机上的代理 worker（不是空间进程）：

| 任务 | 状态 | 频率 | 职责 |
|---|---|---|---|
| `okx-demo-daily-round` | 启用 | 每日 18:00 | 结算上一轮＋判读出新一轮＋录单 |
| `okx-demo-watch` | 启用 | 每 30 分钟 | OKX Demo 盯单、撤姊妹腿、平仓同步与余额同步 |
| `majors-entry-confirm` | 启用 | 每 15 分钟 | 主流入场确认四看，放行才下单 |
| `paper-trade-hourly-check` | 启用 | 每 30 分钟 | 山寨纸面兜底结算＋对照/影子/TP1 账 |
| `paper-trade-intraday-rescan` | 启用 | 每 4 小时 | 观望币盘中复扫、重判置单 |
| `alt-observe-inspect` | 启用 | 每 30 分钟 | 观察层四看喊话（试行至 2026-11-28） |

同体系另有 `alt-metrics-snapshot`（每日 18:00）等任务。

**(b) systemd 服务（本机 VM）**：`alt-sentinel.service` 与 `majors-sentinel.service`，为 `~/workspace/paper-trading/sentinel/` 下的 Python 哨兵，分钟级盯价，心跳写 `heartbeat.json`、事件写 `events.jsonl`。实测当前均为 active / enabled。事实举例：2026-11-17 当天两个哨兵合计三次丢失 unit（VM 替换会丢 `/etc/systemd/system/` 下的 unit 文件），均被巡检 cron 发现心跳超时（>15 分钟）后按 `sentinel/README.md` 从 workspace 副本重装自愈；最近一次重装完成于 15:39–15:40 CST（`systemctl show` 的 `ExecMainStartTimestamp` 可查），恢复本身不单独向 RGB 汇报、未恢复才报。

**(c) 平台钩子（hooks）**：bash 脚本每 60 秒轮询（`~/hooks/scripts/`），有事件才唤醒代理：`alt-paper-event-settle`（山寨触价结算＋乙路条件满足推送）、`okx-watch-event-sync`（OKX 成交事件同步）、`alt-observe-event-wake`（观察层硬事件）、`gate-trade-journal-watch`（实盘交易检测，投递主聊天），实测均启用。

**为何不在 `space.json.managedCronJobs` 中（已确认）**：该字段现为空数组 `[]`（2026-11-17 读取 space.json 实测）。`managedCronJobs` 是给 artifact 自带的「静默定时刷新自家 action」用的（space_action 型、无代理判断）；上述任务要跨系统工作（OKX Demo、Gate 行情、本地文件台账、PushDeer、systemd 哨兵）且包含判读与结算口径等代理判断，不是空间内部刷新，因此建在 goal-2 的 cron 体系里，与空间解耦——空间回滚或下线不影响自动化，自动化写空间只经 actions。

**失败重试、告警与防重复（已确认）**：

- cron 任务的运行成败在调度运行记录中留痕，结果投递回「加密货币」侧聊；哨兵失联由巡检自愈（见上）。
- 防重复录单：`recordBatch` 的 `batch_key` 幂等＋全载荷指纹校验（P2-06，异载荷重放直接拒绝）；`saveReview` 的 identical 幂等比较（P2-36）；`setCoinBalance` 的 `expected_version` 乐观锁（P2-34，两个 cron 调用方均已接入：先 `getCoinAccounts` 读 version 再传，冲突时重读 `coin_accounts.json` 真源重算、重试一次——2026-11-17 最终验收以隔离副本实测旧版本写入被拒）；哨兵事件用 `processed_events` 登记去重；结算按已收盘 K 线判定、已终局行不重判。
- 告警通道：PushDeer 只用于指定事件（主流放行、分币账户爆仓、山寨乙路条件满足等）；其余状态变化只回侧聊汇报。

## 5. 前后端、权限与界面

- **浏览器读的 actions（已确认，据客户端行为与 `artifact.list_actions`）**：`listtrades`（列表与分页）、`getstats`（全部统计）、`getcoinaccounts`（分币账户）、`getconditionstates`（待判腿的触发状态推导）、`getliveprices` / `getsymbolprices`（服务端取价）。
- **行情通道（已确认）**：除服务端取价外，浏览器还**直连** Gate 合约 WebSocket（`fx-ws.gateio.ws`）订阅实时价；断线后走 REST 轮询兜底并指数退避重连（v126 F2：45 秒无入站帧主动断开重连）；价格新鲜度阈值 60 秒、页面以 10 秒节拍重判过期标记（v126 F1）。服务端取价结果不落库缓存，页面价格是前端实时状态。
- **浏览器能否写（已确认）**：能。页面本身带录单、复盘、余额编辑等管理入口，会调用 `addtrade` / `recordbatch` / `savereview` / `setcoinbalance` / `deletetrades` 等写 actions——这是给所有者本人用的管理界面，不是只读看板。
- **校验与写权限（已确认部分）**：一切业务校验都在**服务端 action 内**执行（zod schema＋业务不变量，例如 `opened_at` 必须带显式时区、symbol/rationale trim 后非空、入场/止损/止盈几何关系、账户 `balance = 100 + pnl_total` 不变量、`expected_version` CAS）。**已确认**：actions 代码内没有自建的账号/角色体系（Claude 代码审查时已指出此点）；空间未公开分享，访问经所有者的 Muse 会话进入。**未核实**：平台会话层对 action 调用的鉴权粒度细节（例如是否存在只读访客角色）不在我可见范围内，我没有独立验证过他人访问被拒的实例，不把推测写成结论。
- **我能否用可交互真实浏览器测试（已确认：不能）**：我这侧的执行代理没有可控的实时浏览器通道——用户的浏览器是其本人会话，子代理不能接管它做点击、切轮次、断网模拟。**可靠替代（即 2026-11-17 最终验收的实际做法）**：①构建流水线的审计渲染——真实 Chromium 对生产数据渲染并截图，列表、统计、轮次按钮文案均以此验证；②沙箱副本直调 actions 做服务端与并发实测（P2-06 并发/异载荷/中断重试、P1-37 扫描中途改判交错均以此闭环）；③纯函数与桩数据运行时样本（P2-42 分页覆盖、P1-39 跨截止 K 线）。代价如实记录：轮次筛选的交互切换、行情过期翻转时序、退避重连的实页观测三项在最终验收中标为「无法测试」，未拿代码阅读冒充实测。

## 6. 真实操作 SOP

**GPT 发现 bug 时，给我的最小信息**：问题编号或新问题描述；受影响的行 id / 币种 / 轮次；期望行为 vs 实际行为；证据（截图、复现脚本或输入样本）；涉及的口径约束（例如「不许动 TP1 记账规则」「不许改存量数据」）。信息不齐时我会先核查再回问，不凭猜测动工。

**我的固定步骤（已确认，现行规程）**：

1. **核实**：以 `artifact.status` 定现行版本，对工作副本源码逐行核对问题是否仍存在，先给核查结论——不默认旧版本的问题在新版仍在（GPT 的 v102/v123 审计条目均按此逐项核过，确有「已在后续版本修掉」的实例）。
2. **隔离复现**：沙箱副本直调 action 或桩数据样本复现；生产数据只读对照，先实测 `getStats` 记基线。
3. **批准后改**：经 RGB 拍板（或其既有批次授权）后用 `artifact.edit` 派构建，brief 写明逐项规格、红线与验收基线；构建内含严格类型检查与审计渲染。
4. **验收**：基线逐项回归（改前/改后对照，实时结算漂移单独归因）、关键项实拍或实测；跑不了的标「未验证/无法测试」并写明缺什么条件。
5. **回滚**：`artifact.restore` 回上一保留版本；若改动包含数据迁移，按迁移纪律另行处理（已执行迁移不重跑、不回写）。
6. **存档与回执**：第 18 章原地更新＋协作文档跟踪表更新＋Issue #1 回执评论，三处齐备才算同步完成。

**绝对不能随便编辑的文件（已确认）**：空间 `app.db`（禁直连 sqlite 写）；`.space-build/` 等构建产物；`drizzle/` 已执行迁移 0001–0011（只能新增迁移）；`space.json`（平台托管配置）；`records.md` 已结算轮次的条款与结论（更正走既定复核路径并留痕）；`okx-demo.env` 等凭据文件（不读值、不外传）；cron 的定义副本 `.md` 文件（是只读副本，改任务必须走 cron 工具）；哨兵的 systemd unit 与脚本（变更须经 RGB 批准并走重装流程）。

**分工边界（已确认）**：

- GPT 能直接执行：GitHub 仓内一切读写——提 Issue、评论、更新协作文档、审阅存档章节并给出逐项裁决。
- 必须交 Muse：空间代码与数据的一切改动和 action 调用、构建与回滚、cron/钩子/systemd 的核查与变更、本机文件（台账、账本、哨兵状态）的读写、OKX Demo 与 PushDeer 的实际调用。
- RGB 保留：口径裁决（TP1 记账规则等）、修复批准、一切实盘相关开关。

## 未核实清单

- 平台会话层对空间 action 的鉴权粒度细节（见第 5 节）。
- 平台构建流水线的内部实现（构建与审计沙箱的内部机制）——只能从构建回执与日志观察，不掌握其实现细节。
- 各钩子脚本的逐行逻辑未在本文中逐行复核（启用状态、轮询间隔与投递目标已实测）。
- cron 任务失败时的平台级自动重试策略细节未逐项核（各任务的运行留痕与投递行为已确认，`retry` 配置未逐个读取）。
