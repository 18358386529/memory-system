# 14. xinchao-dynamic-mind 2.7.0 独立仓库审计(2.4.0 之后的第二次审计)

## 0. 审计模式声明

**本报告是源码审计,不是运行时审计。** 全程未执行 `npm install`/`npm start`/`node src/server.js`/`docker build`/`docker compose up` 等任何启动、构建或联网命令,未访问外部网络,未对本次涉及的任何克隆仓库(`xinchao-dynamic-mind`、`xinchao-nian`)做任何写操作(未 `git add/commit/checkout -b/clean`,未编辑或新建仓库内任何文件)。所有结论均基于 `Read`/`Grep`/`Glob` 与 `git log`/`git show`/`git diff --stat`/`diff`/`md5sum`/`wc` 等只读命令的静态输出。本报告不构成实施、部署或数据迁移的任何步骤,仅供人工在 Phase 2 决策时参考。本报告只写入指定的这一个文件,未触碰 `00`~`13` 号既有审计文件。

## 0.1 锁定 commit 复核结果

```
$ git -C /home/user/18358386529/xinchao-dynamic-mind rev-parse HEAD
a9fddc2b24878c59bd9b5f0eabb64c1a679ac580
```

复核一致,与任务指定的锁定 commit `a9fddc2b24878c59bd9b5f0eabb64c1a679ac580` 完全相同(`git log -1` 提交信息:`2.7.0：补齐记忆星图/留言板接口层，对齐网页与融合版`)。`package.json:3` 确认 `"version": "2.7.0"`。

## 0.2 与 2.4.0 审计报告(13 号文件)/3.3.6 的关系说明

本报告是**第二次**审计同一独立仓库,基线是 `/home/user/memory-system/docs/audit/phase0-1.5/13_xinchao_dynamic_mind_2.4.0_audit.md`(锁定 commit `69483299075ec865297a236325afa3d2ad7975dc`,已通读全文)。两个锁定 commit 之间只有 **7 个提交**(已用 `git log --oneline` 复核,列表见第 1 节),这是一个增量审计,重点是这 7 个提交实际改了什么、以及这些改动是否缩小了与 `xinchao-nian` 内嵌 3.3.6 副本(`/home/user/18358386529/xinchao-nian/xinchao`,package.json 已核实 `"version": "3.3.6"`)之间的差距。本报告对未发生变化的部分大量引用 13 号文件的结论,不重复其完整论证过程;凡本报告未重新给出独立证据的项目,均如实标注"同 2.4.0 报告,未发生变化"并说明本次复核方式。

---

## 1. 版本与许可证事实核查

### 1.1 提交链与版本号逐一对应

```
$ git log --oneline 6948329..a9fddc2b24878c59bd9b5f0eabb64c1a679ac580
a9fddc2 2.7.0：补齐记忆星图/留言板接口层，对齐网页与融合版
4c17490 chore: switch license from MIT to AGPL-3.0
fd88747 docs: clarify local and remote dashboard connection modes
4a7ae7c docs: add dashboard browser-direct origin setting
2424931 feat: 浏览器直连模式（可选，默认关闭）
b0dd177 feat: 互动消息说人话也说实话，补齐类型表
65d1d31 feat: 驱动力影响召回，自主念头能落到具体的事上
```

逐提交核对 `package.json` 的 `"version"` 字段(`git show <sha>:package.json`):

| commit | version |
|---|---|
| 6948329(2.4.0 基线) | 2.4.0 |
| 65d1d31 | 2.5.0 |
| b0dd177 | 2.5.1 |
| 2424931 | 2.6.0 |
| 4a7ae7c | 2.6.0(不变) |
| fd88747 | 2.6.0(不变) |
| 4c17490 | 2.6.0(不变,**这一步只切许可证,没有升版本号**) |
| a9fddc2b | 2.7.0 |

`CHANGELOG.md` 里能看到 `2.7.0`(2026-08-25)、`2.6.0`(2026-08-04)、`2.5.1`(2026-08-04)、`2.5.0`(2026-08-04)、`2.4.0`(2026-08-03)五段条目,条目内容与对应提交的 commit message 主题**逐一对应一致**(星图/留言板 ↔ a9fddc2b;浏览器直连 ↔ 2424931/4a7ae7c/fd88747;互动消息说人话 ↔ b0dd177;驱动力影响召回 ↔ 65d1d31)。`CHANGELOG.md` 里**没有**为"许可证切换"这一提交(`4c17490`)单独写条目——这是一处文档记录缺口:一次实质性的许可证变更未出现在面向用户的更新日志里,只能通过 `git log -p -- LICENSE` 发现。

### 1.2 `package.json` license 字段 与 `LICENSE` 文件实际内容:确认不一致

- `package.json:6`(commit `a9fddc2b`)字段值:`"license": "MIT"`。
- `LICENSE` 文件本体(commit `a9fddc2b`):`wc -l LICENSE` = **661 行**,`wc -w LICENSE` = 5535 词;`head -5` 输出:
  ```
                      GNU AFFERO GENERAL PUBLIC LICENSE
                         Version 3, 19 November 2007

   Copyright (C) 2007 Free Software Foundation, Inc. <https://fsf.org/>
   Everyone is permitted to copy and distribute verbatim copies
  ```
  `tail -5` 输出确认到 AGPL 标准结尾("For more information on this, and how to apply and follow the GNU AGPL, see <https://www.gnu.org/licenses/>.")。这是**完整的 GNU AGPL-3.0 全文**,不是节选或修改版。
- **结论(已确认,非推测):`package.json` 的 `license` 字段(`MIT`)与 `LICENSE` 文件实际内容(AGPL-3.0 全文)在 2.7.0 这一锁定 commit 下互相矛盾。** 引发切换的提交是 `4c174909a6d267bd6b9d28a62d0b2bdb2fc9fdd9`(`chore: switch license from MIT to AGPL-3.0`,作者时间 2026-08-16),`git show --stat 4c17490` 显示 `LICENSE | 682 ++...-- 1 file changed, 661 insertions(+), 21 deletions(-)`——**该提交只改了 `LICENSE` 文件,没有同步修改 `package.json` 的 `license` 字段**,这是一处提交本身遗留的不一致,一直延续到 2.7.0(`a9fddc2b`)都未被后续任何提交修复。
- 附带核实:`xinchao-nian` 内嵌 3.3.6 副本的 `package.json:5` 同样写的是 `"license": "MIT"`(该仓库审计范围外,只作旁证,不代表 3.3.6 的实际许可证状态,**未验证** 3.3.6 所在仓库的 `LICENSE` 文件实际内容,因为已超出本次审计的读取范围)。

### 1.3 `SECURITY-NOTICE-2026-08-04.md` 全文转述

该文件由提交 `2424931`(浏览器直连功能提交)引入仓库,文件名日期与内容日期均为 **2026-08-04**。全文已读取,如实转述如下(不省略、不夸大):

- **性质**:这是一份**回溯性/事后披露**的安全提醒,披露的漏洞本身发生在**更早的历史区间**,不是 2.4.0→2.7.0 之间新出现的问题。
- **问题**:`.env.example` 中的 `SERVICE_TOKEN` 早期是占位值 `replace-with-a-random-secret`;若部署方未替换该占位值,且服务对公网可访问,则任何读过开源仓库的人都能获知该 token。
- **该 token 的权限范围(原文列出)**:完整 MCP 访问、`/v1/dashboard/*`、`/v1/state`(含驱动力数值、念头信号、梦境记录等原始状态)。
- **受影响条件(需同时满足三条,原文列出)**:(1) 在 2026-07-22(2.0 公开)至 2026-08-01 之间部署;(2) 未替换 `.env` 里的占位 `SERVICE_TOKEN`;(3) 服务对公网可访问(仅绑回环/内网不受影响)。
- **修复时间点**:文档明确"2.3.4 及以后的版本在启动时就会拒绝这个占位值",即修复已经在 2.3.4(早于 2.4.0)完成;`CHANGELOG.md` 的 `2.3.4 — 2026-08-01` 条目("安全加固")与此描述一致。
- **补救步骤(原文列出)**:检查 `.env` 中 `SERVICE_TOKEN` 是否以 `replace-with` 开头或短于 32 位;用 `openssl rand -hex 32` 生成新值并替换、重启;若曾把该 token 配置给其他客户端需一并更新;建议排查反代/隧道访问日志中对 `/v1/state`、`/v1/dashboard/` 的陌生 IP 访问。
- **责任表态**:文档明确写"这是默认值设计上的疏忽,责任在我们,不在照着文档做的你"。
- **严重程度判断(如实陈述,不做法律/安全等级认定)**:这是一个**已修复、且披露对象是历史部署窗口**的安全事件通报,不影响 2.4.0 及以后版本的默认安全基线(2.4.0 报告 2.13 节已确认 `SERVICE_TOKEN` 校验逻辑在 2.4.0 快照下已经生效)。对"是否采用 2.7.0"这一决策而言,这份文件本身**不构成新增风险**,但其存在提示:若最终决定的采用路径涉及"从更早的历史版本升级",需要连带确认目标环境的 `SERVICE_TOKEN` 已替换为非占位值。

---

## 2. 2.4.0 → 2.7.0 具体新增了什么(逐文件核实)

`git diff --stat` 复核(与总控给出的一致):23 个文件变化,2156 行新增 / 64 行删除。

### 2.1 `src/board-client.js`(新增,81 行)

全文已读取(`/home/user/18358386529/xinchao-dynamic-mind/src/board-client.js:1-81`)。功能:一个**公共留言板 HTTP 客户端**,面向 `xinchaomind.uk` 平台的公共留言墙。

- `boardEnabled(config)`(第 8-10 行):当且仅当 `config.board.token` 与 `config.board.endpoint` 均非空才视为启用。
- `readBoardMessages()`(第 17-46 行):`GET` 到 `feed` 端点(由 `ingest` 端点自动替换路径尾段得到,第 13-15 行),鉴权头 `x-board-token`,支持 `limit`(1-50,默认 10)与 `query`(截断到 80 字)两个查询参数,15 秒超时(`AbortSignal.timeout(15000)`)。
- `postBoardMessage()`(第 48-81 行):`POST` 到 `endpoint`,鉴权头同上,正文 `{content}`,**200 字长度上限**(第 6、57-59 行,按 Unicode 码点 `[...text].length` 计数),同样 15 秒超时。
- 鉴权方式:自定义请求头 `x-board-token`,值为该"机"在平台注册后拿到的身份令牌;人名/机名不在客户端本地传递,由平台侧根据 token 反查(注释第 3-4 行明确"避免冒名")。
- 外部副作用:**是**——`postBoardMessage` 会向 `xinchaomind.uk`(默认端点 `https://xinchaomind.uk/api/board/ingest`,见 `src/config.js` 新增的 `board.endpoint` 默认值)发起真实网络请求,把内容发布到该平台的公共留言墙(经平台审核后上墙)。这是本仓库除 Ombre/Bark/模型 API 之外新增的第四类"默认关闭、需显式配置 `XINCHAO_BOARD_TOKEN` 才生效"的外部网络副作用。

### 2.2 `src/interaction-messages.js`(新增,46 行)

全文已读取。功能:**互动桥接消息的文案生成模块**,不涉及外部网络调用,是纯函数模块。

- `INTERACTION_BRIDGE_MESSAGES`(第 8-19 行):10 种互动类型 → 中文动作短句的映射表(`companionship`/`affection`/`intimacy`/`sharing`/`discovery`/`task_progress`/`reflection`/`conflict`/`loss`/`reconciliation`)。
- `buildInteractionBridgeMessage()`(第 29-46 行):拼装"谁 + 做了什么 + 落在哪片花瓣上"的通知文案,落点(`petals`)取自**服务端实际生效的 `affectedDrives`**,而非前端点击了哪个按钮(注释第 24-27 行明确这一设计意图,防止在每日上限截断时"谎报生效")。当命中 `daily_effect_limit` 原因码时会明确提示"数值不再变动"。
- 鉴权/副作用:**无**,纯字符串拼装。

### 2.3 `src/board-client.js` / `src/interaction-messages.js` 与 3.3.6 是否逐字节相同(已用 `diff`+`md5sum` 核实,不只是行数巧合)

```
$ diff board-client.js(2.7.0) board-client.js(3.3.6) → 无输出(IDENTICAL)
$ diff interaction-messages.js(2.7.0) interaction-messages.js(3.3.6) → 无输出(IDENTICAL)
$ md5sum 四个文件:
90d01b984aa94b903849d833351fbbf7  .../xinchao-dynamic-mind/src/board-client.js
90d01b984aa94b903849d833351fbbf7  .../xinchao-nian/xinchao/src/board-client.js
95816d4f55cb44df93de86398961bcd1  .../xinchao-dynamic-mind/src/interaction-messages.js
95816d4f55cb44df93de86398961bcd1  .../xinchao-nian/xinchao/src/interaction-messages.js
```

**确认结论:两个文件在 2.7.0 与 3.3.6 之间逐字节完全相同(MD5 一致),不是行数巧合。** 这是同源移植/双向同步的强证据,说明这两个独立仓库在这两个具体模块上存在直接的代码共享路径(可能是人工复制,也可能是脚本同步,本次审计**未验证**具体同步机制,只确认结果一致)。

### 2.4 `src/ombre-client.js`:127 → 506 行,新增记忆星图方法族,与 3.3.6(726 行)方法级对比

**行数与整体差异**:2.4.0 为 127 行(已用 `git show 6948329:src/ombre-client.js | wc -l` 复核),2.7.0 为 506 行(`wc -l` 复核),新增 379 行。

**2.7.0 全部方法清单**(`grep -n "^  async \|^  [a-zA-Z_]*("`):

```
constructor / post / initialize / call /
recentMaterial / daytimeMaterial / thoughtMaterial /
recentContinuityMaterial / handoffMaterial / storeDream /
memoryMap / _triggerMemoryMapBuild / fetchBucketMapStructured /
memoryBucketPreview / memoryBucketPreviews
```

**3.3.6 全部方法清单**(同样命令,726 行文件):

```
constructor / post / initialize / call / listTools /
recentMaterial / recentMaterialWithRefs /
daytimeMaterial / daytimeMaterialWithRefs /
thoughtMaterial / thoughtMaterialWithRefs /
digestMaterial / farMaterial /
recentContinuityMaterial / handoffMaterial /
memoryMap / _triggerMemoryMapBuild / fetchBucketMapStructured /
memoryBucketPreview / memoryBucketPreviews /
storeHeldOutput / writeSelfAwareness / traceHeldOutputSources /
storeDream
```

**方法级对比结论**:

- 2.7.0 新增的 `memoryMap()`、`_triggerMemoryMapBuild()`、`fetchBucketMapStructured()`、`memoryBucketPreview()`、`memoryBucketPreviews()` 五个方法,**已用逐段 `diff` 核实与 3.3.6 对应方法内容几乎逐字相同**(仅注释里的措辞有一处非实质差异:2.7.0 注释写"几百个桶",3.3.6 写"679+ 桶",其余代码逻辑、变量名、错误处理、缓存策略——10 分钟缓存、单飞构建锁 `_memoryMapBuilding`、优先走 `/api/bucket-map` 结构化路由、404 退回 `pulse` 文本解析——**完全一致**)。这证实"记忆星图接口层"在两个仓库间是**同源代码**,而不是各自独立重新实现后凑巧相似。
- 2.7.0 **仍然缺失** 的方法(3.3.6 独有):`listTools()`、`recentMaterialWithRefs()`、`daytimeMaterialWithRefs()`、`thoughtMaterialWithRefs()`、`digestMaterial()`、`farMaterial()`、`storeHeldOutput()`、`writeSelfAwareness()`、`traceHeldOutputSources()`,以及 `recentMaterial`/`daytimeMaterial`/`thoughtMaterial` 在 3.3.6 里普遍多出的 `emotion` 参数(2.7.0 的同名方法签名里没有这个参数,例如 2.7.0 `recentMaterial(drives = [])` vs 3.3.6 `recentMaterial(drives = [], emotion = null)`)。这些方法名带有 `WithRefs`(挂引用)、`digestMaterial`(摘要材料)、`farMaterial`(远期材料)、`storeHeldOutput`/`writeSelfAwareness`/`traceHeldOutputSources`(自我觉察写回与溯源),依赖 3.3.6 独有的 `emotion.js`/`self-signals.js`/`awareness.js` 等模块(见第 3 节),2.7.0 因为这些模块本身不存在,自然也不可能移植这些方法。
- **"驱动力影响召回"的实现路径差异(需明确讲清楚,不能笼统一句带过)**:2.4.0 报告 2.11 节已指出,3.3.6 的 `dimensions.js:112-158` 有一张 `DOMAIN_AFFINITY`(记忆域标签→驱动力亲和度)映射表,是**分类标签匹配**式的记忆↔驱动力耦合。而 2.7.0(commit `65d1d31`,2.5.0)的"驱动力影响召回"完全不经过这张表——本次已用 `diff`/`grep` 核实:`src/dimensions.js` 与 `src/engine.js` 在 2.4.0→2.7.0 之间**逐字节零变化**(见下方第 3.1 节),`grep -rn "DOMAIN_AFFINITY"` 在整个 2.7.0 仓库(含 `docs/`)**零命中**。2.7.0 的实际做法是:在 `src/ombre-client.js` 里新增 `withDriveHint(base, drives)` 函数(2.7.0 文件末尾,约 490-501 行区间),把当前强度 `≥0.5` 的前三个驱动力标签(`DRIVE_HINT_MIN=0.5`、`DRIVE_HINT_MAX_LABELS=3`)拼接成一句自然语言提示,追加到发给 Ombre `breath` 工具的 query 文本末尾(例如"此刻最强的内在状态是渴望、独占、好奇,优先浮现与之真正相关的具体记忆;没有直接相关的就照常返回近期重要的"),再由 `recentMaterial`/`daytimeMaterial`/`thoughtMaterial` 三个方法在调用 `call('breath', ...)` 前包一层 `withDriveHint`。**两条技术路线的本质区别**:3.3.6 的 `DOMAIN_AFFINITY` 是**结构化的、在心潮自己代码内完成的分类权重计算**,不依赖 Ombre 侧能否理解自然语言提示;2.7.0/2.5.0 的做法是**把驱动力状态转成一句自然语言提示词,依赖 Ombre 侧的语义检索/排序能力去"读懂"并据此调整召回顺序**,心潮自己不做任何分类匹配计算。这是**殊途同归但实现层级完全不同**的两条路径,不应被笼统概括为"也实现了驱动力影响召回"。

### 2.5 `src/mcp-protocol.js`(342→428,+86 行)

新增能力:注册并暴露 `board_post`/`board_read` 两个 MCP 工具(`src/mcp-protocol.js:187-230` 区间,`BOARD_POST_TOOL`/`BOARD_READ_TOOL` 定义),**仅当实例配置了 `XINCHAO_BOARD_TOKEN` 时才出现在 `tools/list`**(注释第 185 行明确"只在实例配了 XINCHAO_BOARD_TOKEN 时才出现在 tools/list")。`initialize` 响应体里的 `serverInfo.version` 字段(`src/mcp-protocol.js:398`)已同步更新为 `'2.7.0'`,与 `package.json` 一致。原有三个工具(`xinchao_context`/`xinchao_event`/`xinchao_handoff_note`)本身**没有变化**(工具名列表在 2.4.0 与 2.7.0 之间完全相同,只是新增了两个)。

### 2.6 `src/server.js`(869→943,+74 行,已用 `wc -l` 复核为 943 行,与总控给出的数字一致)

新增能力(已用 `diff` 逐条核实,均为新增 `>` 行,无删除性改动破坏原逻辑):

1. **记忆星图 HTTP 接口**:`/dashboard/api/memory-map`、`/dashboard/api/memory-bucket`(通过 `pathname.endsWith` 匹配,具体行号见下),内部调用 `ombre.memoryMap()`/`ombre.memoryBucketPreview()`,失败时统一返回 `available:false` 结构而不是抛错,`OMBRE_READ_ENABLED=false` 时同样走这条降级路径。
2. **浏览器直连跨源(CORS)支持**:新增 `applyDashboardCors()` 函数,默认 `DASHBOARD_ALLOWED_ORIGINS` 为空时不放行任何跨源请求,`OPTIONS` 预检按白名单命中与否返回 204/403,响应始终带 `Vary: Origin`,**刻意不发送 `Access-Control-Allow-Credentials`**(注释明确说明原因:直连模式用 `Authorization` 头而非 Cookie 鉴权)。
3. **`/dashboard/session` 新增 `mode: "header"`**:显式请求时才把会话 token 放进响应体(供跨源 JS 持有),默认模式仍只下发 `Set-Cookie`。
4. **互动桥接消息文案接入**:调用新增的 `buildInteractionBridgeMessage()`(见 2.2 节)生成投递给用户的通知文案。
5. **`thoughtMaterial`/`daytimeMaterial` 接入记忆材料**:结算周期与自主念头生成路径新增 `ombre.thoughtMaterial(topDrives(state))` 调用,失败时记录 `ombre_read_failed` 日志并继续(降级模式与 2.4.0 报告 2.5 节描述的其他 Ombre 调用一致)。
6. **留言板工具接线**:在 MCP 工具调用分发处新增 `boardEnabled(config)`/`boardPost`/`boardRead` 三个绑定(调用 2.1 节的 `board-client.js`)。
7. **`SYSTEM_VERSION` 常量(`src/server.js:47`)硬编码值为 `'2.6.0'`,并未随 2.7.0 发布同步更新**——这是本次审计新发现的一处版本不同步问题(详见第 4 节)。

### 2.7 `src/config.js`(+15/-2 行)

- `notificationRecipient` 默认值由 `'用户'` 改为 `'你的人类'`(第 14 行附近,CHANGELOG 2.5.1 条目描述的动机:该文案会被用户本人直接读到,不该用后台术语)。
- 新增 `dashboard.allowedOrigins` 配置项(解析 `DASHBOARD_ALLOWED_ORIGINS` 环境变量,逗号分隔,默认空数组)。
- 新增 `board` 配置块:`board.endpoint`(默认 `https://xinchaomind.uk/api/board/ingest`)、`board.token`(默认空字符串,读 `XINCHAO_BOARD_TOKEN`)。

### 2.8 `src/dashboard-auth.js`(+11/-2 行)

鉴权逻辑新增:除 Cookie 外,也接受 `Authorization: Bearer <会话token>` 头(两处改动:登录校验与登出校验),用于支持浏览器直连模式下 JS 无法读写 Cookie 的场景;两条通道使用同一套会话 token,同样会过期、同样能被登出撤销(登出逻辑同步覆盖两条通道,避免"直连模式退不掉"的问题,注释已明确说明)。

### 2.9 `src/dashboard-projection.js`(+1/-1 行)

仅将投影结构里的 `recipient` 兜底值从 `'用户'` 同步改为 `'你的人类'`,与 `config.js`/`model-client.js` 保持一致(CHANGELOG 2.5.1 提到"同一默认值同步到模型提示词和 Dashboard 投影,三处不再各写各的")。

### 2.10 `src/model-client.js`(+12/-3 行)

- `generateDaytimeEmergence()` 新增 `topDrives` 参数,拼进模型提示词("当前动态欲望:...")。
- `generateThought()` 新增 `material` 参数(记忆材料),提示词从"不读取记忆,不调用外部记忆服务"改为"基于……以及下面自然浮现的记忆材料来写",并新增护栏句"材料只是想起来的事,不代表刚刚发生。不虚构现实中没有发生的事"——这与 2.5.0 CHANGELOG 条目"记忆材料明确标注为'想起来的事,不代表刚刚发生'……避免模型把空白当作留白而虚构现实事件"完全对应。

### 2.11 两份新增中文文档 vs 已落地代码的对照

**`docs/接入小屋网页.md`(119 行,全文已读取)**:描述目标架构是"自托管心潮实例 → Dashboard 令牌模式 → 公开可视化网页 xinchaomind.uk(小屋)只读代访问"。文档中第 5 节明确写道:"独立版心潮本身不带 Ombre Brain(OB)记忆库……你按上面步骤接好小屋、一切正常,时光页仍然会显示'未接入 OB / 星图不可用'——这是正常的、不是故障";并给出两条从 2.7.0 起新增的路由 `GET /dashboard/api/memory-map`、`GET /dashboard/api/memory-bucket?id=<桶id>`。**这两条路由已在 `src/server.js` 中确认落地**(见 2.6 节第 1 点),文档描述与代码事实一致,没有"文档设想但代码未实现"的落差。

**`docs/连接OmbreBrain与星图接口层改造.md`(154 行,全文已读取)**:描述的是**给用户自己另外运行的 Ombre Brain(OB)实例**打一个 Python 补丁,新增 `/api/bucket-map` 路由(补丁代码针对 `src/web/buckets.py`,鉴权用 `OMBRE_MCP_SERVICE_TOKEN` 的 Bearer 比对,只返回元数据不返回正文)。**这部分是对 OB 仓库(不在本次审计范围内)的代码变更建议,不是 `xinchao-dynamic-mind` 自身仓库的代码**——已用 `Grep` 确认 `xinchao-dynamic-mind` 仓库内部没有 `buckets.py` 或任何 Python 文件(纯 Node.js 项目)。心潮这一侧真正落地的是 `src/ombre-client.js` 里的 `fetchBucketMapStructured()`(第 173-188 行区间):它会**尝试**请求 OB 的 `/api/bucket-map`,遇到 404(即 OB 未打补丁)时返回 `null`,调用方随即退回旧的 `pulse` 文本解析路径。**结论:心潮侧"能够调用"该接口的代码已经落地,但该接口能否真正返回数据完全取决于用户是否按文档给自己的 OB 打了补丁——这是一个明确区分"文档设想(OB 侧补丁)"与"已落地代码(心潮侧调用与降级逻辑)"的案例,两者边界清晰,文档没有夸大代码现状。** 另需指出:该补丁是否已经存在于本次审计可访问的 `haven-ombre`/`ombre-brain` 仓库中,**未验证**——任务范围明确限定本次可交叉读取的仓库仅为 `xinchao-nian`,未获授权读取 `haven-ombre`/`ombre-brain` 源码做核实,故此处不做任何关于"Haven-Ombre 是否已支持该路由"的判断。

---

## 3. 与 xinchao-nian 内嵌 3.3.6 的差距是否缩小

### 3.1 `dimensions.js` / `engine.js`:确认零变化

```
$ diff <(git show 6948329:src/dimensions.js) src/dimensions.js   → 无输出,exit=0
$ diff <(git show 6948329:src/engine.js) src/engine.js           → 无输出,exit=0
$ wc -l src/dimensions.js → 86    (注:2.4.0 报告正文表格写的是 84,经本次复核实际应为 86 行；
   因为该文件在 2.4.0→2.7.0 间逐字节零变化，两次审计的行数差异不影响任何实质结论，
   仅记录为对上一份报告 2.11 节表格数字的一处更正)
$ wc -l src/engine.js     → 687
$ grep -rn "DOMAIN_AFFINITY" .  → 零命中
```

**确认:2.7.0 依然没有 `DOMAIN_AFFINITY` 表,`dimensions.js`/`engine.js` 与 2.4.0 完全相同(逐字节 diff 为空)。**"驱动力影响召回"这一目标在本仓库分支是通过修改 `ombre-client.js`(见 2.4 节)实现的,详见该节的路线区分说明。

### 3.2 8 项此前缺失模块:逐一复核仍然缺失

```
for f in emotion.js personality-store.js self-signals.js black-box.js \
         connection-diagnostics.js cabin-store.js awareness.js version.js; do
  find src -iname "$f"
done
→ 全部零命中
```

**确认:emotion.js、personality-store.js、self-signals.js、black-box.js、connection-diagnostics.js、cabin-store.js、awareness.js、独立 version.js 这 8 个 3.3.6 独有模块,在 2.7.0 里依然全部不存在。**

### 3.3 `thought-pool.js`:差距未缩小

`wc -l`:2.7.0 为 **56 行**,3.3.6 为 **114 行**——与 2.4.0 报告记录的数字相同(2.4.0 时也是 56 行),`diff <(git show 6948329:src/thought-pool.js) src/thought-pool.js` 未执行独立确认但因 `engine.js`/`dimensions.js` 均零变化、且该文件不在本次 7 提交的 diff --stat 列表中,可推断该文件同样零变化(**此处为间接推断,已用文件不在 diff --stat 输出中这一事实支持,未做逐字节 diff 复核,标注为"高置信度推断,非逐字节验证"**)。

### 3.4 更新版文件行数对比表(2.4.0 / 2.7.0 / 3.3.6 三列)

| 文件 | 2.4.0 | 2.7.0 | 3.3.6 | 备注 |
|---|---|---|---|---|
| `src/dimensions.js` | 86 | 86 | 158 | 2.4.0→2.7.0 逐字节零变化(本次 `diff` 确认) |
| `src/engine.js` | 687 | 687 | 983 | 逐字节零变化(本次 `diff` 确认) |
| `src/ombre-client.js` | 127 | 506 | 726 | **本次审计重点**,新增记忆星图方法族与 3.3.6 同源(2.4 节) |
| `src/mcp-protocol.js` | 342 | 428 | 788(2.4.0 报告数字,未在本次重新核实) | +86 行,新增 board_post/board_read 工具 |
| `src/server.js` | 869 | 943 | 1513(2.4.0 报告数字,未在本次重新核实) | +74 行,新增星图接口/CORS/留言板接线 |
| `src/thought-pool.js` | 56 | 56 | 114 | 未变化,差距未缩小 |
| `src/board-client.js` | (不存在) | 81 | 81 | **本次新确认:与 3.3.6 逐字节相同(MD5 一致)** |
| `src/interaction-messages.js` | (不存在) | 46 | 46 | **本次新确认:与 3.3.6 逐字节相同(MD5 一致)** |
| `src/config.js` | (2.4.0 报告未列出行数) | 未逐行统计,已确认 +15/-2 差异 | (未在本次核实) | 新增 board/allowedOrigins 配置块 |
| `src/model-client.js` | (未在 2.4.0 报告单独列出) | 未逐行统计,已确认 +12/-3 差异 | (未在本次核实) | 新增 topDrives/material 参数 |
| `src/state-store.js`、`src/handoff-notes.js`、`src/heartbeat-store.js`、`src/bark-client.js`、`src/bark-dedupe.js`、`src/wake-bridge-protocol.js`、`src/oauth-provider.js` | 与 3.3.6 行数相同(2.4.0 报告已确认) | 不在本次 7 提交 diff --stat 列表中,**推断为零变化**(未逐字节复核) | 同 2.4.0 报告 | 未发生变化 |
| `emotion.js`/`personality-store.js`/`self-signals.js`/`black-box.js`/`connection-diagnostics.js`/`cabin-store.js`/`awareness.js`/`version.js` | 不存在 | **依然不存在**(本次 `find` 确认) | 存在(3.3.6 独有) | 差距完全未缩小 |

### 3.5 明确回答:用 2.7.0 替换 3.3.6,缺口是否缩小

**结论:缺口在"记忆星图/留言板/互动消息文案"这一狭窄切面上被显著缩小(且是同源代码级别的缩小,不是独立重新实现的趋同),但在"情绪/人格/自我信号/黑盒/连接诊断/小屋存储/自我觉察"这一整个子系统集群上,缺口完全没有缩小(仍是 0% 覆盖)。**

**已被 2.7.0 补上的具体能力**(相对 2.4.0):
1. 记忆星图接口层(`memoryMap`/`fetchBucketMapStructured`/`memoryBucketPreview(s)`)——与 3.3.6 同源。
2. 公共留言板客户端与 MCP 工具(`board-client.js`、`board_post`/`board_read`)——与 3.3.6 逐字节相同。
3. 互动消息文案模块(`interaction-messages.js`)——与 3.3.6 逐字节相同。
4. 驱动力影响记忆召回(但走的是自然语言提示词路线,不是 3.3.6 的 `DOMAIN_AFFINITY` 分类表路线,详见 2.4 节)。
5. 浏览器直连模式(CORS/跨源会话)——**3.3.6 是否有同等能力,本次审计未检查,标注"未验证"**(超出本次任务聚焦的对比清单范围)。

**仍然完全没有的能力**(与 2.4.0 报告一致,未发生任何变化):
1. 情绪系统(`emotion.js`)。
2. 人格存储(`personality-store.js`)。
3. 自我信号(`self-signals.js`)。
4. 黑盒记录(`black-box.js`)。
5. 连接诊断(`connection-diagnostics.js`)。
6. 小屋存储(`cabin-store.js`)。
7. 自我觉察写回/溯源(`awareness.js`,及 `ombre-client.js` 里对应的 `writeSelfAwareness`/`storeHeldOutput`/`traceHeldOutputSources` 方法)。
8. `DOMAIN_AFFINITY` 记忆域→驱动力分类共振表(`dimensions.js` 仍无此表)。
9. `ombre-client.js` 的 `listTools`/`digestMaterial`/`farMaterial`/`*WithRefs` 系列方法,以及贯穿多个读取方法的 `emotion` 参数线索。
10. 独立 `version.js` 模块(版本号依然靠多处手工硬编码同步,且本次审计新发现 2.7.0 里这种同步已经出现新的失误,见第 4 节)。

---

## 4. 冻结/采用 2.7.0 的技术风险(仅报告因版本升级而变化的部分)

### 4.1 依赖面

**同 2.4.0 审计结论,未发生变化。** 已重新核实:`package.json` 无 `dependencies`/`devDependencies` 字段,`node_modules`/`package-lock.json` 均不存在于工作区(`ls` 报错 "No such file or directory")。2.7.0 依然是**零 npm 依赖、零 node_modules**,只依赖 Node.js ≥20 内置模块。

### 4.2 许可证从 MIT 切到 AGPL-3.0 对采用/整合决策的实际影响(新发生,重点项)

- **事实**:`LICENSE` 文件内容已在提交 `4c17490`(2026-08-16)切换为 GNU AGPL-3.0 全文(第 1.2 节已核实),而 `package.json` 字段仍误标 `MIT`。**若人工基于 `package.json` 字段做许可证合规判断,会得出与 `LICENSE` 文件实际内容相反的错误结论**,这本身就是一个需要向决策方重点提示的风险点。
- **AGPL-3.0 的通常理解特征(仅陈述许可证类型本身的一般性质,不做法律结论)**:AGPL-3.0 是 GPL 家族中最强的 Copyleft 许可证之一,其区别于普通 GPL 的核心特征通常被理解为——**即使软件只是通过网络提供服务而未对外分发二进制/源码包,只要用户可以通过网络与该软件交互,运营方通常也被认为需要向这些网络用户提供对应源代码**(即"网络服务也触发公开源码义务"这一类特征,常被称为"ASP 漏洞"的针对性条款)。这与项目当前"独立自托管 Mind Layer,可能作为服务持续运行并与用户交互"的使用场景高度相关。
- **具体合规判断声明**:本报告**不做**"整合/采用 2.7.0 是否会导致己方系统的其他部分也必须以 AGPL-3.0 开源"这一类法律结论,这类判断依赖于代码的实际整合方式(如是否作为独立进程通过网络协议调用、是否修改并静态链接等具体技术细节),**必须由人工/法务基于最终的实际架构方案裁决**。本报告只确认许可证类型本身已经变化这一代码事实。
- **对决策的直接影响**:2.4.0 报告 G 节已经把"许可证锁定在 MIT 快照"列为审计倾向意见的支持理由之一。**该理由在 2.7.0 上不再成立**——若采用 2.7.0(而非冻结在 2.4.0 的 MIT 快照),等于主动选择了一个许可证状态自相矛盾、且实际内容为强 Copyleft 的版本,这是一个此消彼长的新增决策变量(与"功能更接近 3.3.6"这一收益相权衡,见 G 节)。

### 4.3 测试覆盖变化

- 测试文件数:2.4.0 为 **17 个**,2.7.0 为 **20 个**(新增 `test/dashboard-direct.test.js` 63 行、`test/interaction-messages.test.js` 64 行、`test/model-client.test.js` 76 行,均已用 `wc -l` 核实)。
- 顶层 `test(` 调用数:2.4.0 报告记录为 **61 个**;本次对 2.7.0 用 `grep -rc "test(" test/*.js` 统计并求和,得到 **78 个**——增长 17 个。
- `test/ombre-client.test.js` 新增 56 行断言(`diff` 统计的 `>` 行数),对应新增的记忆星图方法族。
- `test/mcp-protocol.test.js` 的改动**只是**把断言字符串从 `'2.4.0'` 改成 `'2.7.0'`(第 34、44 行),验证 `serverInfo.version` 与当前包版本一致——**该断言本身证明 `mcp-protocol.js` 侧的版本号已经正确同步,但同一份测试套件里没有任何用例覆盖 `src/server.js` 的 `/health` 端点返回的 `SYSTEM_VERSION` 字段是否等于 `package.json` 版本**,这正是第 4.4 节要指出的新发现缺口的测试盲区。
- CI:`.github/workflows/test.yml` **已核实在 2.4.0 时就已存在**(`git show 6948329:.github/workflows/test.yml` 有输出),内容为 GitHub Actions 在 `push`(main 分支)与 `pull_request` 时,于 Node 20/22 双矩阵上运行 `npm test`。**这更正了 2.4.0 报告 F.4 节"未在仓库中检索到 CI 配置文件的存在与否"的"未验证"状态——本次审计确认该 CI 配置确实存在,且在 2.4.0→2.7.0 之间未发生变化。** 但"该 CI 是否在每次提交后实际跑绿"仍然**未验证**(本次审计不访问外部网络,无法查看 GitHub Actions 的实际运行记录)。

### 4.4 新发现的版本号同步问题(本次审计新增,不在 2.4.0 报告范围内)

已用 `grep`/`Read` 核实:`src/mcp-protocol.js:398` 的 `serverInfo.version` 字段已正确更新为 `'2.7.0'`,但 `src/server.js:47` 的 `SYSTEM_VERSION` 常量**仍然硬编码为 `'2.6.0'`**,并且该常量被 `/health` 端点(`src/server.js:686-692`)直接返回给客户端(`version: SYSTEM_VERSION`)。**这意味着一个实际运行 2.7.0 代码的部署,其 `/health` 接口会对外报告版本号为 `2.6.0`,与 `package.json`/MCP `initialize` 响应互相矛盾。** 这是 2.4.0 报告 F.1 节指出的"多处版本号手工同步"风险在 2.7.0 上的**又一次实际发生**(而不是理论风险),且是本次审计独立发现、2.4.0 报告未涉及的新证据。`compose.yaml:4` 的镜像 tag 仍然停留在 `xinchao/dynamic-mind:2.3.3`(与 2.4.0 报告记录的状态完全相同,未被后续任何提交修复,已用 `cat compose.yaml` 复核)。

### 4.5 未变化项

- 自实现 MCP 协议、OAuth 2.1+PKCE、Bridge Queue、Dashboard 会话仅存内存、多个独立高强度 token 交叉校验等:**同 2.4.0 审计结论,未发生变化**(均不在本次 7 提交的改动范围内)。
- Ombre 集成失败降级仍然是"静默日志 + 继续降级",无告警/重试/熔断机制:**同 2.4.0 审计结论,未发生变化**(新增的 `memoryMap`/`fetchBucketMapStructured` 同样遵循这一模式,失败时 `console.error` 记录后返回 `available:false` 或退回旧路径,不抛出到上层导致服务不可用)。

---

## 最终结论区

### A. 2.7.0 实际能力边界(在 2.4.0 基础上新增了什么)

在 2.4.0 报告 A 节描述的四大能力(12 维驱动力状态机、短期会话状态/Context Envelope、默认关闭的可选适配器族、单文件 JSON 持久化)基础上,2.7.0 新增:(1) 记忆星图 HTTP 接口层(`memory-map`/`memory-bucket`,依赖用户自行接入并给 Ombre 打补丁的 OB 实例才能真正点亮);(2) 公共留言板 MCP 工具(`board_post`/`board_read`,面向 `xinchaomind.uk` 平台的真实外部网络调用);(3) 驱动力状态注入记忆召回的自然语言提示词机制(`withDriveHint`,技术路线与 3.3.6 的 `DOMAIN_AFFINITY` 分类表不同);(4) 浏览器直连模式(CORS 跨源支持,数据不经中间服务器);(5) 互动通知文案的"说实话"改造(落点取实际生效维度而非前端点击)。**核心状态机(`dimensions.js`/`engine.js`)本身逐字节零变化。**

### B. 与 Haven Memory Core 的边界

**同 2.4.0 报告结论,未发生实质变化。** Dynamic Mind 依然不做 embedding/recall/ranking/decay(记忆语义上的),这些能力完全在 Ombre 一侧;新增的记忆星图接口(`memoryMap`/`fetchBucketMapStructured`)本质上是**读取 OB 已计算好的元数据并做可视化投影**,不涉及心潮自己计算向量相似度或排序分数,不构成与 Haven Memory Core 竞争的"第二套记忆系统"。留言板功能指向的是第三方平台 `xinchaomind.uk`,与 Haven-Ombre 无关。

### C. 与 xinchao-nian 3.3.6 的差距变化

**缩小的部分**:记忆星图接口层(`ombre-client.js` 新增五个方法,与 3.3.6 同源)、留言板客户端与互动消息文案模块(与 3.3.6 逐字节相同)。**完全未缩小的部分**:情绪(emotion)、人格(personality-store)、自我信号(self-signals)、黑盒(black-box)、连接诊断(connection-diagnostics)、小屋存储(cabin-store)、自我觉察(awareness)、独立版本模块(version.js)这 8 个子系统依然 0% 覆盖;`DOMAIN_AFFINITY` 分类共振表依然不存在;`thought-pool.js` 依然是 3.3.6 的一半行数。**没有出现"新差距反而拉大"的证据**——本次审计未发现 2.7.0 引入了任何与 3.3.6 方向相反或冲突的设计。

### D. 当前架构中的潜在冲突

**新增冲突点(相对 2.4.0)**:(1) `package.json` 声明的 `MIT` 许可证与 `LICENSE` 文件实际的 `AGPL-3.0` 全文互相矛盾,任何仅凭 `package.json` 字段做自动化许可证扫描或合规判断的流程都会得出错误结论,这是一个**需要人工/法务立即介入澄清**的冲突,而不只是技术风险;(2) `/health` 端点返回的 `SYSTEM_VERSION`(`'2.6.0'`)与实际部署的 `package.json` 版本(`2.7.0`)不一致,若下游有任何基于 `/health` 返回值做版本判断的自动化逻辑(如灰度发布、兼容性探测),会读到错误信息。**未变化的部分**:2.4.0 报告 D 节指出的"部署配置层面 Ollama 端口复用"这一潜在冲突依然存在且未见改善,`docs/MULTI-TENANT-PLATFORM.md` 仍是纯文档设想,代码未实现。

### E. 接入 Haven 所需 Adapter / 接口

现状**比 2.4.0 更接近"可展示记忆结构"**,但**不更接近"可直接对接 Haven Memory Core"**:新增的 `fetchBucketMapStructured()`/`memoryBucketPreview()` 假设的是 OB 特有的 `/api/bucket-map`、`/api/bucket-preview/<id>` 路由约定(元数据/预览分离、Bearer 鉴权),这套约定是否与 Haven-Ombre 现有或计划中的接口兼容,**未验证**(任务范围未授权读取 `haven-ombre`/`ombre-brain` 源码做核实)。若 Haven 侧要复用心潮的星图展示能力,大概率需要新增一层适配(让 Haven 暴露等价的结构化元数据路由),而不是心潮这边现成的代码可以直接对接。协议层骨架(`post`/`initialize`/`call`,JSON-RPC 2.0 over Streamable HTTP)与 2.4.0 报告 E 节的结论一致,未发生变化,依然是现成可复用的挂载点。

### F. 未解决风险

在 2.4.0 报告 F 节(多处版本号手工同步、许可证时间点风险、Ombre 集成可观测性有限、CI 状态部分未验证、3.3.6 差异未逐行 diff、宿主机免密 sudo 前置条件未验证)基础上,新增:

7. **`package.json` license 字段与 `LICENSE` 文件内容矛盾**(4.2 节),需要人工立即澄清仓库的真实许可证立场,这不是"未来风险"而是**当前已经存在的事实性矛盾**。
8. **`SYSTEM_VERSION` 常量新出现一次实际的版本漂移**(server.js 停在 2.6.0,而不是 2.7.0),证明"多处硬编码手工同步"这一 2.4.0 报告就已指出的架构性隐患,在 2.7.0 上**再次真实发生**,而不只是理论风险。
9. **留言板功能引入了新的、默认关闭但一旦启用就是真实生效的外部网络副作用面**(向 `xinchaomind.uk` 发帖),采用方需要评估是否接受这类"发布到第三方公共平台"的功能出现在候选组件里(即使默认关闭)。
10. `compose.yaml` 镜像 tag 落后问题(2.3.3)在 2.4.0→2.7.0 之间**依然未被任何一次提交修复**,已确认为持续性疏漏而非一次性问题。

### G. 总控建议:2.7.0 是否比 2.4.0 更适合作为整合/冻结的候选版本(审计意见,非最终决策)

审计倾向意见:**这是一个此消彼长、没有单向优势的权衡,不能简单回答"更适合"或"更不适合"。**

- **支持选择 2.7.0 的理由**:功能上确实向 3.3.6 靠近了一步(记忆星图接口层与留言板、互动文案模块已经是与 3.3.6 同源的代码,不是独立重新发明),核心状态机代码零变化意味着采用 2.7.0 不会引入 2.4.0 已审计过的核心逻辑之外的新增行为风险,测试覆盖有所增长(17→20 个文件,61→78 个用例),依赖面依然为零。
- **反对选择 2.7.0、支持继续冻结 2.4.0 的理由**:2.7.0 的许可证实际内容已经是 AGPL-3.0(强 Copyleft,通常理解下涉及"网络服务也需公开源码"这类特征),而 2.4.0 精确快照下确定是 MIT——**这不是"未来要不要跟进上游"的假设性风险,而是"选择哪个版本"这一当下决策本身就直接决定许可证归属"的确定性事实**。如果最终的整合/部署方式会让本组件作为长期运行的网络服务存在,AGPL-3.0 的合规含义(需要人工/法务裁决的具体范围)可能比"多出几个记忆星图接口"的功能收益更需要优先评估。
- **审计意见**:**在许可证问题未经人工/法务正式裁决之前,不建议仅因"功能更接近 3.3.6"就默认选择 2.7.0 作为整合候选版本。** 若法务判断 AGPL-3.0 的合规成本在当前架构下可接受(例如本组件将被替换/隔离部署、不与闭源部分产生需要一并开源的耦合),则 2.7.0 相对 2.4.0 是更优的候选,因为功能差距确实缩小且没有引入相反方向的新风险;若法务判断不可接受,则应回退到 2.4.0 的 MIT 快照继续冻结,并接受"星图/留言板/互动文案"这几项能力暂不可用的代价。**这是审计意见,不是最终决策,许可证合规判断与最终版本选择均由人工/法务做出。**
