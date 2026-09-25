# 13. xinchao-dynamic-mind 2.4.0 独立仓库审计

## 0. 审计模式声明

**本报告是源码审计,不是运行时审计。** 全程未执行 `npm install`、`npm start`、`node src/server.js`、`docker build`、`docker compose up` 等任何启动/构建命令,未访问外部网络,未修改克隆仓库中的任何文件,未对四个既有克隆仓库(haven-ombre / ombre-brain / xinchao-nian / kiwi-mem)做任何写操作。所有结论均基于对文件内容的静态阅读与 `git log`/`git show`/`diff --stat` 等只读命令。本报告不构成实施、部署或数据迁移的任何步骤,仅供人工在 Phase 2 决策时参考。

## 1. 版本定位方法与证据(复述 + 复核)

审计对象仓库 `tianyupaipai-cmd/xinchao-dynamic-mind`(本地只读克隆源 `18358386529/xinchao-dynamic-mind`)**没有 git tag**——`git ls-remote --tags origin` 零输出。版本号只体现在 `package.json` 的 `"version"` 字段随提交演进,因此版本定位必须靠逐次提交核对 `package.json`,而不是常规的 tag/release 流程。总控已完成的核对方法:

- 用 `git log -p -- package.json` 逐一核对版本号变更提交:
  - 提交 `6c78a4a759ad8557f45e08e49a76a43f2b7ca60e`(`feat: add user interaction runtime bridge queue`)把版本从 2.3.3 改为 2.4.0。
  - 下一次版本变更发生在提交 `65d1d316991f6ded3093e83bcfce51a8ac0df416`(`feat: 驱动力影响召回,自主念头能落到具体的事上`),把版本从 2.4.0 改为 2.5.0。
  - 因此 `"version": "2.4.0"` 覆盖的提交区间是 `6c78a4a`(含)到 `65d1d31` 的直接父提交 `6948329`(含)。
- 总控已将本地仓库 detach checkout 到该区间的最后一个提交:**锁定 commit SHA `69483299075ec865297a236325afa3d2ad7975dc`**(commit message:`test: 断言跟着维度表走,不写死 12`,作者时间 2026-08-04 12:00:49 +0800)。

**本子代理独立复核结果(本次审计执行时重新运行)**:

```
$ git -C /home/user/18358386529/xinchao-dynamic-mind rev-parse HEAD
69483299075ec865297a236325afa3d2ad7975dc
$ git -C /home/user/18358386529/xinchao-dynamic-mind status --short
(空,无未提交改动)
$ git -C /home/user/18358386529/xinchao-dynamic-mind log -1 --format="%H %ci %s"
69483299075ec865297a236325afa3d2ad7975dc 2026-08-04 12:00:49 +0800 test: 断言跟着维度表走，不写死 12
```

复核一致:当前工作区确实停在 `6948329`(缩写),`package.json` 第 3 行 `"version": "2.4.0"`(见下文证据),工作区干净,以下所有代码级结论均以此快照为准。

同一区间之后的提交链(仅作为时间参照,未 checkout、未读取其内容用于本报告正文结论,只用 `--stat`/`log` 只读列出):
`6948329 → 71db932 → 65d1d31(2.5.0) → b0dd177 → 2424931 → 4a7ae7c → fd88747 → 4c17490(License: MIT→AGPL-3.0) → a9fddc2(2.7.0)`。

## 2. 重点检查清单逐项证据

### 2.1 核心模块与运行方式

- `package.json`(commit 6948329)第 1-22 行:`"name": "xinchao-dynamic-mind"`,`"version": "2.4.0"`,`"type": "module"`,`"license": "MIT"`,`scripts.start = "node src/server.js"`,`scripts.test = "node --test"`,`engines.node >= 20`。**没有 `dependencies`/`devDependencies` 字段**——全仓库零 npm 依赖(见 2.9 节证据)。
- `src/server.js:1-19` 导入清单显示服务由单一 Node.js `http` 模块(`createServer`)构建,无 Express/Fastify 等框架。
- `src/server.js:855-859`:`server.listen(config.port, '0.0.0.0', ...)`——监听地址为 `0.0.0.0`,端口来自 `config.port`(默认由 `.env.example:7` `PORT=18110`)。
- `Dockerfile:1-12`:基础镜像 `node:20-alpine`,`EXPOSE 18110`,`CMD ["node", "src/server.js"]`,以非 root 用户 `node` 运行。
- `compose.yaml:1-31`:`image: xinchao/dynamic-mind:2.3.3`——**注意此镜像 tag 停留在 2.3.3,未随 package.json 升级到 2.4.0**,这是一处文档/构建配置与代码版本不同步的证据(compose.yaml 第 4 行)。`ports: "127.0.0.1:18110:18110"` 仅绑定回环地址;`read_only: true`、`cap_drop: ALL`、`mem_limit: 128m`、`cpus: 0.50`。
- `scripts/deploy-to-vps.sh`、`scripts/install-shadow.sh`:实际部署路径是 `rsync` + `ssh` + `sudo docker compose up -d --build`,没有 Kubernetes/PaaS 相关脚本。

### 2.2 是否可作为独立 Mind Layer(是否强依赖 xinchao-nian 等外部仓库)

- 全文 grep(`grep -rn "xinchao-nian" .`,未展示因为零命中)未发现 `src/*.js` 中出现任何对 `xinchao-nian` 仓库路径、包名或 URL 的硬编码引用。
- `src/config.js:38-45` 中 `ombre` 配置项(`OMBRE_MCP_URL`/`OMBRE_MCP_TOKEN`)默认全部关闭(`readEnabled: false`, `writeEnabled: false`),且 `config.js:127-144` 的 `validateConfig` 只在**用户主动打开** `OMBRE_READ_ENABLED`/`OMBRE_WRITE_ENABLED`/`CONTEXT_OMBRE_ENABLED` 任一开关时才强制要求 `OMBRE_MCP_URL`/`OMBRE_MCP_TOKEN`,否则可以完全不配置任何外部记忆服务而独立启动。
- `src/model-client.js:15,61,83,121` 处对 `MODEL_ENABLED`/`apiKey` 判空后走 `fallback()`/`fallbackThought()` 规则路径(纯字符串拼接,无网络调用),即模型服务同样可选。
- 结论(有证据支持):在本 commit 下,Dynamic Mind **不存在对 xinchao-nian 仓库或其他外部代码仓库的硬编码依赖**;它是否能"独立跑起来"取决于运行时能否提供 `SERVICE_TOKEN`(`server.js:21-28` 强制要求,且拒绝占位值和短于 32 字符的值)——这是启动的唯一硬性前置条件,不涉及外部仓库。

### 2.3 自有 Memory Core / 长期记忆存储

- `state/.gitkeep`、`memory-data/.gitkeep`:本次锁定 commit 下这两个目录只有 `.gitkeep` 空文件(已用 `ls -la` 核实,大小分别为 66/76 字节,即 `.gitkeep` 文件本身),**没有任何提交进去的示例记忆数据**。
- `src/state-store.js:1-52`:`StateStore` 类只做单文件 JSON 的原子读写(临时文件 + `rename`),路径为 `config.statePath`(默认 `/app/state/state.json`),这是**短期动态状态**(drives/thoughtPool/sessionOverlays/handoffNotes/recentDreams 等),不是可检索的长期记忆库——没有索引、没有查询接口、没有多条目集合语义,只是一份会被整体覆盖的 JSON 快照。
- `src/engine.js:577-585` `recordDream`:把梦境对象 push 进 `state.recentDreams` 并裁剪到最近 20 条(`slice(-20)`),这是本地易失性的"最近梦境"数组,同样存放在 `state.json` 里,**不是独立的长期记忆库**,且 `breathDreamContext`(`engine.js:592-609`)只回看最近 18 小时、最多 3 条。
- 结论:该仓库在本 commit 没有自己的长期记忆存储组件(没有向量库、没有全文索引、没有独立的记忆文件格式),长期记忆能力完全外包给可选的 `OmbreClient`(见 2.5 节)。

### 2.4 embedding / recall / ranking / decay 能力核查

- 全仓库(含 `docs/`、`scripts/`)grep `embedding|bge-m3|ollama|rerank|ranking|pgvector|postgres`(命令:`grep -rniE "embedding|bge-m3|ollama|rerank|ranking|pgvector|postgres" --include="*.js" --include="*.md" --include="*.sh" .`)命中仅 4 处,且全部在 `docs/MULTI-TENANT-PLATFORM.md:219,426,454,470`,提及"建议使用 PostgreSQL"等——**这是该文档自称"设计基线"(`docs/MULTI-TENANT-PLATFORM.md:2` "状态:设计基线")的未来多人平台方案,不是本 commit 实际代码**;`src/*.js` 中没有任何一行提及 postgres/pgvector/embedding/bge-m3/ollama/rerank/ranking 字符串本身(仅 `.env.example:16` 的默认 `MODEL_BASE_URL=http://127.0.0.1:11434/v1` 恰好是 Ollama 常用端口,但代码里没有 "ollama" 字样,详见 2.9 节)。
- `src/thought-pool.js:1` `FLASH_DECAY = 0.82` 和 `src/engine.js:329-330`(饱和态衰减:`decay = (current - SATURATE_FLOOR) * 0.10 * elapsedHours`)确认属实——这两处衰减都是**驱动力/念头强度**的数值衰减(0-1 范围的欲望值随时间回落),与"记忆检索排序衰减"(如 recall 分数按时间打折)在语义和作用对象上完全不同,不操作任何记忆条目、不参与检索排序。
- 结论:本 commit 下**没有 embedding、没有语义检索、没有 rerank、没有与 Haven-Ombre 同类的记忆排序/衰减算法**,与 `haven-ombre` 的 `embedding_engine.py`/`bucket_manager.py` 职责完全不重叠,不构成能力冲突(冲突需要先存在同类能力才谈得上冲突)。

### 2.5 外部 Memory 接口:`src/ombre-client.js`

- `src/ombre-client.js:1-6` `OmbreClient` 构造函数只保存 `config`、`sessionId`、`initializePromise`,无本地缓存或本地存储。
- 协议:自实现的 MCP JSON-RPC 2.0 客户端。`post()`(`ombre-client.js:8-26`)向 `config.url` 发 `POST`,头部含 `Authorization: Bearer <token>`(第 14 行,仅当配置了 token)、`Mcp-Session-Id`(会话复用,第 15、22 行)、`Accept: application/json, text/event-stream`。`initialize()`(28-47 行)发送标准 MCP `initialize` + `notifications/initialized`,`clientInfo.version` 硬编码为 `'2.4.0'`(第 39 行,与 package.json 版本手动保持一致,存在人工同步风险)。
- 读:`recentMaterial()`(62-69 行)、`daytimeMaterial()`(71-78 行)、`recentContinuityMaterial()`(80-91 行)均调用 `this.call('breath', {...})`——这与 `mcp__ombre-brain__breath`/`breath_advanced`/`breath_search` 工具族命名一致,证实其目标端点是 Ombre/Haven-Ombre 风格的 MCP 记忆服务。
- 写:`storeDream()`(99-116 行)调用 `this.call('hold', {...})`,仅当 `config.writeEnabled`(即 `OMBRE_WRITE_ENABLED`)为真才执行(第 100 行 `if (!this.config.writeEnabled) return null;`),写入内容是"梦境+梦境余韵+醒后意识"文本,`tags: 'dream'`,`importance: 7`,`auto: true`,`source: 'xinchao-dream'`。
- 只读/可写开关独立控制:`src/config.js:41-42` `OMBRE_READ_ENABLED`/`OMBRE_WRITE_ENABLED` 两个布尔量默认均为 `false`,互不联动,可单独打开只读而不开写。
- 失败降级行为:`src/server.js:126-129`(dream 生成前读取 `ombre.recentMaterial()` 失败时 `catch` 记日志 `ombre_read_failed` 并把 `material` 保持为空字符串,不中断结算周期)、`src/server.js:144-147`(写入失败时 `catch` 记 `ombre_write_failed`,`dream.ombreBucketId` 保持 `null`)、`src/server.js:440-446`(Context Envelope 读取失败时 `ombreWarning = 'ombre_unavailable'` 并继续返回不含 Ombre 内容的信封)。三处均为"失败即静默降级为无记忆材料",**不会导致服务不可用**,但也意味着 Ombre 侧的任何错误(鉴权失败、超时、格式变化)在 2.4.0 里只表现为日志事件,没有告警/重试/熔断机制。
- 会话续期:`call()`(49-60 行)在收到 `HTTP 400/404` 时会清空 `sessionId` 并重试一次("session 过期自动重新 initialize"),超过一次仍失败则抛错交由上层 catch。

### 2.6 状态模型:drive / thought / fatigue / sleep / dream / short-term state

- `src/dimensions.js:4-84`:`DIMENSIONS` 常量定义 **12 个驱动力维度**(possess, monitor, crave, share, libido, curiosity, boredom, social, duty, reflection, grieve, anger),每维含 `growPerHour`、`satisfyMul`、可选 `nightMul`/`dawnFreeze`/`inhibitedBy`。`DRIVE_KEYS`(第 86 行)由 `Object.keys(DIMENSIONS)` 派生,**不写死数字 12**——这正对应锁定 commit 的提交信息"断言跟着维度表走,不写死 12"(可在 `test/engine.test.js` 中找到相应断言,见 2.11 节)。
- `src/engine.js:299-374` `settleState()`:按 `elapsedHours` 对每个驱动力做时间增长(含清晨冻结 `dawnFreeze`、夜间倍率 `nightMul`、疲劳倍率 `fatigueMultiplier`、饱和区间 `SATURATE_CEIL=0.80`/`SATURATE_FLOOR=0.65` 的软限幅),并驱动 `thoughtPool` tick(第 343-351 行)、`fatigue` 增减(353-361 行)、清醒→睡眠转换(363-369 行,空闲达到 `sleepAfterMinutes` 进入 `sleeping`)。
- `src/thought-pool.js:1-56`:`flash`(闪念,强度按 `FLASH_DECAY=0.82` 逐 tick 衰减)、`obsessions`(执念,由持续闪念在 `intensity>=0.50` 且 `age>=3` 时晋升,`OBSESSION_GROWTH=1.10` 复利增长,达到 `FEEDBACK_CEIL=0.85` 时反馈回对应驱动力,最多 3 次反馈后从池中移除)。
- `src/state-store.js:1-52`:落盘为单一 `state.json`,`newState()`(`engine.js:188-219`)定义的完整字段包括 `schemaVersion, revision, consciousness, lastConversationAt, lastHeartbeatAt, lastSettledAt, sleepStartedAt, drives, thoughtPool, fatigue, recentDreams, dreamUsage, lastBarkAt..., sessionOverlays, contextDeliveries, recentConversationEvents, interactionUsage`。
- `src/heartbeat-store.js:1-14`:只读一个由外部(Ombre 侧)写入的心跳文件(`OMBRE_HEARTBEAT_FILE`,默认 `/memory-data/heartbeat.json`),解析 `recordedAt`/`recorded_at` 字段为时间戳,不做其他处理——这是"内容无关的心跳",用于 `contactIdleAllowed()`(`engine.js:534-539`)判断对方是否长期离线。

### 2.7 HTTP / MCP / Bridge 接口

- **HTTP API**(`src/server.js`):`/health`(621-628,GET,无需鉴权,返回 `version: SYSTEM_VERSION`,第 45 行硬编码 `'2.4.0'`——与 package.json 版本手动保持一致,同样存在同步风险)、`/v1/state`(793-795)、`/v1/breath-context`(796-802)、`/v1/context`(803-818)、`/v1/intent`(819-823)、`/v1/settle`(824-827,POST 手动触发一次结算周期)、`/v1/conversation-event` 与 `/v1/heartbeat`(828-832)、`/v1/handoff-note`(833-836)、`/v1/drive-feedback`(837-847)。除 `/health` 外均要求 `Authorization: Bearer <SERVICE_TOKEN>`(`authorized()`,308-311 行,`timingSafeEqual` 常量时间比较)。
- **MCP**(`src/mcp-protocol.js` + `server.js:736-785`):路径 `/mcp` 或 `/mcp/*`,仅当 `MCP_ENABLED=true` 才启用(`config.js:57-60`)。协议是**自实现的 JSON-RPC 2.0**(`mcp-protocol.js:296-342` `handleMcpMessage`),支持 `initialize`/`ping`/`tools/list`/`tools/call`/`notifications/*`,协议版本协商支持 `2025-03-26`/`2025-06-18`(第 1 行 `SUPPORTED_PROTOCOLS`)。**全仓库没有 `import` 任何 `@modelcontextprotocol/sdk` 或其他 `mcp` 官方 SDK 包**(package.json 无 dependencies,`grep "^import"` 结果里没有非 node:/非相对路径的导入,见 2.9 节),证实这是一套完全自造的最小 MCP 实现,只暴露三个工具:`xinchao_context`/`xinchao_event`/`xinchao_handoff_note`(`mcp-protocol.js:15-174`)。鉴权支持三种:`SERVICE_TOKEN` Bearer、OAuth access token、`MCP_PATH_TOKEN`(URL 路径 token,`mcpAuthorized()`,356-363 行)。
- **Bridge**(`src/bridge-queue.js` + `server.js:630-670`):路径 `/bridge/v1/*`,仅当 `BRIDGE_ENABLED=true` 启用,鉴权用独立的 `BRIDGE_MACHINE_TOKEN`(要求 ≥32 字符且不能与 `SERVICE_TOKEN`/`DASHBOARD_ACCESS_TOKEN` 重复,`config.js:165-171`)。功能:健康检查、SSE 事件流(`/bridge/v1/events`)、按 id 读取单条待投递内容(读一次,`bridge-queue.js:81-91`)、ACK(`acknowledge()`,93-113 行,状态机 `pending → delivered` 或 `retryable_failed`)。`BRIDGE_REASONS`(`bridge-queue.js:7`)硬编码只允许 `user_interaction`/`user_note`/`scheduled_interaction` 三种理由,**代码层面**阻止了梦境/念头/AI 自主活动混入 Bridge(与 `docs/ARCHITECTURE.md:69` "只处理用户主动互动...不运输 AI 内部状态" 的文档声明一致)。
- **`src/wake-bridge-protocol.js`**:整个文件只有 3 行,`export * from '../packages/wake-bridge/src/index.js';`——是对独立子包 `packages/wake-bridge` 的重新导出封装(见 2.8 节)。
- **`src/oauth-provider.js`**(530 行):自实现 OAuth 2.1 + PKCE(S256 强制,`PKCE_PATTERN` 43-128 字符)+ 动态客户端注册(`/oauth/register`),状态持久化到独立 `oauth.json`(`persist()`,174-185 行,同样是临时文件+`rename`的原子写)。仅当 `OAUTH_ENABLED=true` 且 `OAUTH_PUBLIC_BASE_URL` 为 `https://` 时可 `init()` 成功(135-137 行强制 HTTPS,除非本地测试)。

### 2.8 `packages/wake-bridge/` 子包完整度

- 文件清单(`find packages/wake-bridge -type f`):`README.md`、`package.json`、`src/index.js`(仅 1 个源文件,83 行)。
- `packages/wake-bridge/package.json`:`"name": "@xinchao/wake-bridge-protocol"`,`"version": "0.1.0"`(独立版本号,与主包 2.4.0 不同步管理),`"license": "MIT"`,`exports: "./src/index.js"`,**无 dependencies**。
- `packages/wake-bridge/src/index.js:1-83`:导出 `createWakeBridgeEnvelope`/`markWakeBridgeDelivered`/`consumeWakeBridgeEnvelope`,定义 `WAKE_BRIDGE_KINDS`(`dream_residue, longing_content, action_result, pending_from_me`)与 `WAKE_BRIDGE_AUDIENCES`(`user, ai, both`)。`assertSafePayload()`(18-24 行)用正则 `FORBIDDEN_KEYS`(第 12 行)拦截 `authorization`/`cookie`/`raw_chat`/`raw_prompt`/`raw_conversation`/`service_token`/`access_token` 等键名,防止敏感字段混入信封。
- 结论:该子包在本 commit 下是一个**独立、无依赖、职责单一、结构完整**的小型协议库(定义信封结构+校验,不含网络/存储代码),属于 monorepo 内的组件,不是主服务运行时的独立依赖(它被 `src/wake-bridge-protocol.js` 直接 re-export,同一部署单元内联使用,没有单独发布到 npm registry 的证据——`packages/wake-bridge/package.json` 无 `publishConfig`/`private` 字段,也没有 CI 发布脚本可查)。

### 2.9 本地 LLM 依赖判断:`src/model-client.js`

- `src/model-client.js:150-157` `request()`:`fetch(\`${this.config.baseUrl}/chat/completions\`, {...})`——**通用 OpenAI-compatible Chat Completions API**,不限定供应商,`baseUrl` 完全由环境变量 `MODEL_BASE_URL` 决定。
- `.env.example:16` 默认值 `MODEL_BASE_URL=http://127.0.0.1:11434/v1`——端口 `11434` 是 Ollama 默认端口的强惯例,但**代码本身没有任何 "ollama" 字符串或 Ollama 专用协议逻辑**,只是把默认值设成了"本机很可能在跑 Ollama"的地址。`scripts/configure-model.sh:1-40` 只替换 `MODEL_API_KEY`/`MODEL_ENABLED`,**不修改 `MODEL_BASE_URL`**,即该脚本假设用户会另外手动把 `MODEL_BASE_URL` 改成远程供应商地址,脚本本身不区分"本地"或"远程"。
- `.env.example:15` `MODEL_ENABLED=false`——默认完全关闭模型调用,`model-client.js:15,61,83,121` 处均在未启用/无 key 时走纯规则回退(`fallback()`/`fallbackThought()`,字符串模板拼接,无任何推理调用)。
- 结论:2.4.0 的模型接入层**既不强制远程 API,也不强制本地 Ollama**,是一个通用适配层;默认配置(禁用)下不产生任何 LLM 调用。这与总控预设的"远程 LLM + Ollama 仅 BGE-M3 embedding"架构假设的关系是:model-client.js 只做**对话/文本生成**(chat completions),完全不涉及 embedding 端点(没有 `/embeddings` 调用,全文 grep 确认,见 2.4 节),因此它与"Ollama 仅用于 BGE-M3 embedding"的假设**不冲突也不重叠**——二者操作的是不同的模型能力(生成 vs. 向量化),2.4.0 代码里没有出现向量化调用。

### 2.10 与"远程 LLM + Ollama 仅 BGE-M3 embedding"架构假设的冲突判断

基于 2.4 与 2.9 节证据:本 commit 的 Dynamic Mind **不做任何 embedding 调用**,`model-client.js` 唯一的外部模型调用端点是 `/chat/completions`(文本生成),没有 `/embeddings` 或类似端点的调用代码。因此该假设(Ollama 专职 BGE-M3 embedding)与 2.4.0 现状**没有直接冲突**——因为 2.4.0 根本不用 Ollama/任何模型做 embedding,它需要模型只是为了生成梦境文案/自主念头文案(可选功能,默认关闭)。潜在的间接冲突点在于:如果部署方按 `.env.example` 默认值把 `MODEL_BASE_URL` 指向同一个 `127.0.0.1:11434`(Ollama)地址来跑对话生成模型,则会与"Ollama 只服务 BGE-M3 embedding"的资源分配假设产生**端口/模型占用层面**的实际冲突(同一 Ollama 实例上同时装载生成模型和 embedding 模型,可能挤占显存/内存),但这是**部署配置选择**导致的冲突,不是代码层面的强制耦合——只需把 `MODEL_BASE_URL` 指到别的远程/本地生成模型端点即可规避,不需要改代码。

### 2.11 与 `xinchao-nian/xinchao`(3.3.6)相比的差异

对比路径:`/home/user/18358386529/xinchao-nian/xinchao`(package.json `"version": "3.3.6"`,已用 `Read` 核实)。

**文件清单差异**(`find` 对比):

- 3.3.6 独有、2.4.0 没有的模块:`src/awareness.js`、`src/black-box.js`、`src/board-client.js`、`src/cabin-store.js`、`src/connection-diagnostics.js`、`src/emotion.js`、`src/interaction-messages.js`、`src/personality-store.js`、`src/self-signals.js`、`src/version.js`,以及对应的大量测试文件(`awareness-3.3.test.js`、`black-box.test.js`、`board-client.test.js`、`cabin-store.test.js`、`engine-3.1-coupling-plateau.test.js`、`engine-3.3-emotion*.test.js`、`engine-3.3-grudge.test.js`、`personality-store.test.js`、`self-signals-3.3.test.js`、`thought-*-3.3*.test.js` 等)。
- 2.4.0 的 `packages/wake-bridge/` 子包目录结构在 3.3.6 里**不作为独立 monorepo 子包存在**——3.3.6 的 `src/wake-bridge-protocol.js` 是完整实现(未检查其具体行数是否等于独立文件,但 3.3.6 目录下没有 `packages/` 目录,可确认其未采用 monorepo 拆分)。
- 3.3.6 新增 `src/version.js`(3 行,`export const SYSTEM_VERSION = '3.3.6';`,注释注明 `test/version.test.js` 会校验此值与 package.json 一致)——2.4.0 里没有独立 `version.js`,版本号是在 `src/server.js:45` 和 `src/mcp-protocol.js:316` 里**分别手工硬编码**为字符串 `'2.4.0'`,存在两处/多处手动同步的脆弱性(已在 2.7 节指出)。

**共有文件的内容级差异**(行数对比,`wc -l`):

| 文件 | 2.4.0 | 3.3.6 | 差异要点(已读取源码确认) |
|---|---|---|---|
| `src/dimensions.js` | 84 | 158 | 3.3.6 新增每维 `ceil`(静息天花板)、情绪型驱力(grieve/anger)的 `decayHalfLifeHours: 24` 半衰期回落,以及一张全新的 `DOMAIN_AFFINITY` 记忆共振表(112-158 行),把 OB 记忆桶的 `domain` 标签(如"恋爱""亲密""成长"等 26 个域)映射到 12 维驱动力的亲和度权重——**这是"域标签→驱动力"的分类式关联,不是向量/embedding 检索**,对应 commit `65d1d31`("驱动力影响召回")在 2.4.0 之后才引入,**2.4.0 本身没有这张表**。 |
| `src/ombre-client.js` | 127 | 726 | 3.3.6 大幅扩展(新增 `ombre-client-memory-map.test.js`、`ombre-client-surfacing.test.js` 对应的功能),说明 Ombre 集成深度在后续版本显著加深;2.4.0 只有 4 个只读/写方法(`recentMaterial`/`daytimeMaterial`/`recentContinuityMaterial`/`storeDream`)。 |
| `src/mcp-protocol.js` | 342 | 788 | 3.3.6 增加了远多于 2.4.0 的 MCP 能力(具体新增工具未逐一核实,因超出本次 2.4.0 审计范围,标注"未验证:未展开读取 3.3.6 该文件全文")。 |
| `src/server.js` | 869 | 1513 | 同上,3.3.6 路由/功能显著增多。 |
| `src/engine.js` | 687 | 983 | 3.3.6 新增 emotion/grudge/coupling 相关逻辑(对应新增测试文件名可推断,未逐行核实具体函数,标注"未验证:未展开读取全文")。 |
| `src/state-store.js`、`src/handoff-notes.js`、`src/heartbeat-store.js`、`src/bark-client.js`、`src/bark-dedupe.js`、`src/wake-bridge-protocol.js` | 行数完全相同 | 行数完全相同 | 这几个文件在两版本间行数一致,内容层面**未逐字节 diff**(超出本次审计的必要深度),标注"未验证:仅核对行数,未做逐行 diff"。 |

**结论**:2.4.0 是一个相对精简的早期独立版本,核心的 12 维驱动力/念头池/结算/Context Envelope/OAuth/Dashboard/Bridge 骨架已经成型且与 3.3.6 保持同源(`oauth-provider.js`、`handoff-notes.js`、`heartbeat-store.js`、`bark-client.js`、`bark-dedupe.js` 完全同行数,大概率内容也高度一致或相同);3.3.6 在此基础上大幅加深了"记忆←→驱动力"的耦合(`DOMAIN_AFFINITY` 表、大幅扩展的 `ombre-client.js`)以及情绪/人格/黑盒/看板等新子系统,这些在 2.4.0 里完全不存在。

### 2.12 License 与版本信息

- **2.4.0 快照下的 `LICENSE` 文件本体**(`LICENSE:1-22`,commit 6948329):完整的标准 MIT License 文本,`Copyright (c) 2026 Xinchao contributors`。`package.json:6` 的 `"license": "MIT"` 字段与文件本体**一致**。
- **仓库后续历史**(不属于 2.4.0,仅作时间参照):提交 `4c174909a6d267bd6b9d28a62d0b2bdb2fc9fdd9`(`chore: switch license from MIT to AGPL-3.0`,作者时间 2026-08-16 21:22:28 +0800)把 `LICENSE` 从 22 行的 MIT 文本替换为 682 行的 AGPL-3.0 全文(`git show --stat 4c17490` 显示 `LICENSE | 682 ++...--` `1 file changed, 661 insertions(+), 21 deletions(-)`)。**这次许可证变更发生在 2.4.0(6948329)之后 12 天**,经过了 `65d1d31(2.5.0)`→`b0dd177`→`2424931`→`4a7ae7c`→`fd88747` 几次提交才到达。**2.4.0 这个精确版本快照下,许可证确定是 MIT,不是 AGPL-3.0。** 若人工决定"冻结 2.4.0 作为依赖",在许可证层面锁定的是 MIT 版本,与仓库当前/最新历史的 AGPL-3.0 状态无关——但需注意:若日后需要合并仓库上游的任何补丁(即使只是安全补丁),上游补丁本身可能来自已切换为 AGPL-3.0 的代码树,存在许可证兼容性需要人工单独评估的风险(本报告不做法律结论,仅陈述事实)。
- `packages/wake-bridge/package.json:6` 同样标注 `"license": "MIT"`,与主包一致。

### 2.13 冻结 2.4.0 作为依赖的技术风险(仅评估该版本自身)

以下均为可从代码/文档直接查证的事实,不依赖"未来是否继续维护"的预测:

- **零外部 npm 依赖**(`package.json` 无 `dependencies`/`devDependencies`,`node_modules`/`package-lock.json` 均不存在于仓库,已用 `ls` 核实文件不存在):意味着该版本**不存在"依赖包被弃用/出现 CVE"的供应链风险**,只依赖 Node.js ≥20 内置模块(`node:http`、`node:crypto`、`node:fs/promises`、`node:path`)。这是相对少见的极简依赖面,技术稳定性角度是**正面**因素——不会因为第三方包破坏性升级而失效。
- **自实现 MCP 协议**(`src/mcp-protocol.js`,2.4.0 只有 3 个工具、342 行):优点是没有 SDK 版本漂移风险;缺点是**协议演进需要人工手动跟进 MCP 官方规范变化**(本 commit 支持的协议版本硬编码为 `2025-03-26`/`2025-06-18` 两个值,`mcp-protocol.js:1`),若 MCP 规范后续大版本变化(新增必需字段等),2.4.0 不会自动兼容,需要人工升级代码。
- **测试覆盖**:`test/` 目录下 17 个测试文件,统计到的顶层 `test(` 调用共 61 个(`grep -c "test("`,未展开嵌套 `t.test()` 子测试,故实际断言数可能更高;`test/http-api.test.js` 内部 `test(`/`t.test(`/`assert.` 总行数 54 行,是一个端到端集成测试文件)。覆盖范围包括:engine(18 个用例,含维度表驱动的断言)、mcp-protocol(8 个)、config(6 个)、context-envelope(4 个),对 `dashboard-auth`、`dashboard-projection`、`transition-journal`、`oauth-provider`、`wake-bridge-protocol`、`bridge-queue` 各有 2-3 个用例。**未发现任何 `.skip()`/`.todo()` 标记的测试**(未逐文件确认此点,标注"未验证:未做全文 grep 确认 skip/todo",但通读的文件中未见到)。测试是否在 CI 中实际跑绿,本报告**无法验证**(没有 `.github/workflows` 或其他 CI 配置文件的读取记录——**未验证:未检索仓库是否存在 CI 配置**)。
- **部署脚本的人工依赖点**:`scripts/install-shadow.sh`、`scripts/deploy-to-vps.sh` 依赖 `sudo -n docker compose`(免密 sudo),意味着**运行该脚本的宿主机账户需要预先配置好免密 sudo docker 权限**,这是一个环境前置条件,不是代码 bug,但如果冻结 2.4.0 作为长期依赖,需要人工确认目标宿主机具备这个前置条件(本报告未验证任何实际宿主机环境,只陈述脚本要求)。
- **compose.yaml 镜像 tag 落后于 package.json 版本**(2.2 节已指出:`compose.yaml:4` 写的是 `xinchao/dynamic-mind:2.3.3` 而 `package.json` 是 2.4.0):说明该仓库在 2.4.0 这个 commit 时,`compose.yaml` 本身还没有跟上版本号更新——这是一个**已知的、当时未修复的文档/配置不一致**,如果直接照抄 `compose.yaml` 里的镜像 tag 去拉取/构建,可能与实际代码版本对不上(不影响 `docker compose build .` 从本地 Dockerfile 直接构建的路径,但影响任何"从 registry 拉取指定 tag 镜像"的路径)。
- **Dashboard 会话仅存内存**(`src/dashboard-auth.js:29` `this.sessions = new Map()`):进程重启后所有已登录的 Dashboard 会话失效,需要重新登录——这是设计如此(避免把会话 token 落盘增加泄露面),不是缺陷,但运维时需要知悉。
- **OAuth/Bridge/Dashboard 均要求独立的高强度 token 且互不相同**(`config.js:145-171` 多处交叉校验),这提高了安全基线,但也提高了初始配置复杂度——多个 32 位以上随机密钥需要人工生成并妥善分发,配置错误会导致启动直接失败(`validateConfig` 抛错)而不是静默降级,这是**偏保守但会阻断快速上线**的设计取舍。

## A. 2.4.0 实际能力边界

2.4.0 是一个**零依赖、单文件状态机 + 极简自实现 MCP/OAuth/HTTP 服务**的 Node.js 程序。核心能力:
1. 12 维驱动力(欲望)数值模型,按时间结算、按互动类型做有界语义反馈,含"闪念→执念"心理动态(`thought-pool.js`)。
2. 短期会话状态(`sessionOverlays`)、限时交接便签(`handoffNotes`)、Context Envelope 拼装(短期状态摘要,非长期记忆)。
3. 可选适配器(默认全部关闭):对话模型生成(梦境文案/自主念头文案,OpenAI-compatible chat completions)、Bark 手机通知、Ombre 风格的外部记忆读写(仅 `breath`/`hold` 两类调用)、Runtime Bridge(用户互动排队投递)、OAuth 2.1+PKCE 远程 MCP 鉴权、只读 Dashboard 投影。
4. 状态持久化只有单一 `state.json`(原子写)+ 追加写的 `transitions.jsonl` 审计日志 + 独立 `oauth.json`。**没有长期记忆存储、没有 embedding、没有检索排序能力。**

## B. 与 Haven Memory Core 的边界

2.4.0 与 Haven-Ombre 之间**没有能力重叠**:Dynamic Mind 不做 embedding/recall/ranking/decay(记忆语义上的),这些能力完全在 Ombre 一侧;Dynamic Mind 只通过 `OmbreClient` 以 MCP 客户端身份调用 Ombre 暴露的 `breath`(读,召回近期材料用于生成梦境/念头文案)和 `hold`(写,把梦境写回 Ombre 作为一条记忆)两个工具,且默认两者都关闭。Dynamic Mind 自己的 `state.json`/`recentDreams` 是短期、易失、无检索能力的运行时状态,不构成与 Haven Memory Core 竞争的"第二套记忆系统"。

## C. 与 xinchao-nian 的差异

详见 2.11 节。要点:2.4.0 是精简早期骨架,3.3.6 在此基础上新增了情绪(emotion)、人格(personality-store)、自我信号(self-signals)、黑盒记录(black-box)、留言板(board-client)、连接诊断(connection-diagnostics)、独立版本文件(version.js)等子系统,并且**新增了"记忆域标签→驱动力"的分类式共振表**(`DOMAIN_AFFINITY`,3.3.6 `dimensions.js:112-158`),这是 2.4.0 完全没有的"记忆影响驱动力"耦合点,但其本质是**分类标签匹配,不是向量检索**。`ombre-client.js` 的深度也从 127 行扩展到 726 行,说明后续版本对 Ombre 记忆服务的依赖和交互复杂度显著上升。

## D. 当前架构中的潜在冲突

在 2.4.0 这一具体版本下,**代码层面未发现与 Haven-Ombre 直接冲突的能力**(无 embedding/无第二套记忆库/无 Bayesian/NHPP 逻辑)。唯一具备潜在冲突性质的是**部署配置层面**:`.env.example` 默认把对话生成模型的 `MODEL_BASE_URL` 指向 Ollama 常用端口 `127.0.0.1:11434`,如果目标环境的 Ollama 实例被"仅用于 BGE-M3 embedding"的假设占用,同一 Ollama 进程上再加载一个对话生成模型会产生资源(显存/内存/并发请求)争用——这是运维配置选择造成的冲突,可通过修改 `MODEL_BASE_URL` 指向别处规避,不需要改代码。另需关注:`docs/MULTI-TENANT-PLATFORM.md` 提出的未来 PostgreSQL 多租户方案是纯文档设想,2.4.0 代码中没有任何对应实现,若被误读为"仓库已经支持 PostgreSQL"会造成认知冲突,应明确区分。

## E. 接入 Haven 所需 Adapter / 接口(现状缺口,不做实现设计)

- 现有可复用挂载点:`src/ombre-client.js` 已经是一个可工作的 Ombre 兼容 MCP 客户端(`breath`/`hold` 两个工具调用),协议层(JSON-RPC 2.0 over Streamable HTTP,`Mcp-Session-Id` 会话管理,Bearer 鉴权)已经实现且有对应测试(`test/ombre-client.test.js`,本次只确认其存在,未展开读取其断言细节——**未验证:未逐行核对该测试文件覆盖的具体场景**)。若 Haven-Ombre 暴露的 MCP 工具名/参数与 `breath`/`hold` 兼容,理论上可以直接复用现有 `OmbreClient` 而无需新写适配器;若 Haven-Ombre 的工具签名不同,需要修改 `ombre-client.js` 里的方法参数或新增方法,但**协议层骨架(post/initialize/call)可以复用**。
- 缺口:`src/config.js` 中 `ombre.url`/`ombre.token` 是单一端点配置,如果 Haven 侧需要多租户/多小屋隔离(每个用户连接不同的 Ombre 实例),2.4.0 的配置模型是单实例的,需要额外的多租户配置层(`docs/MULTI-TENANT-PLATFORM.md` 已经提出这个方向但仅是文档设想,未落地代码)。
- Context Envelope(`src/context-envelope.js`)已经预留了 `recent_continuity` 分区专门承载 Ombre 返回的内容(`buildContextEnvelope` 第 215-223 行),这是一个现成的"记忆材料注入短期上下文"的挂载点,无需新建。

## F. 未解决风险

1. **多处版本号手工同步**(`package.json`、`src/server.js:45`、`src/mcp-protocol.js:316`、`src/ombre-client.js:39`、`compose.yaml:4`)——本 commit 下 `compose.yaml` 已经出现不同步(仍为 2.3.3),存在人工维护疏漏导致镜像/代码版本错配的风险。
2. **许可证时间点风险**(2.13 节已述):2.4.0 是 MIT,仓库当前是 AGPL-3.0,若未来需要合并上游补丁需单独做许可证兼容性判断,本报告不做法律结论。
3. **Ombre 集成的可观测性有限**:失败降级全部走 `console.log` JSON 事件(`log()` 函数,`server.js:47-49`),没有告警/重试/熔断,长期运行若 Ombre 侧持续 401/超时,不会有主动告警,只能靠日志巡检发现(`ombre_read_failed`/`ombre_write_failed`/`context_ombre_read_failed` 三类事件名)。
4. **CI/测试实际执行状态未验证**:未在仓库中检索到 CI 配置文件的存在与否(未做该项检索),因此"61 个测试用例是否在合并前实际跑绿"无法从本地静态审计确认,标注为**未验证**。
5. **3.3.6 与 2.4.0 之间的 `mcp-protocol.js`/`server.js`/`engine.js` 具体新增内容未逐行 diff**(2.11 节已注明"未验证"),若 Phase 2 需要精确评估"从 2.4.0 升级到某个更高版本需要改多少代码",需要额外做逐行 diff,本报告只给出行数级别的差异量级。
6. **宿主机免密 sudo docker 前置条件未验证**:部署脚本假设的环境条件本报告未在任何实际主机上验证。

## G. 是否具备进入 Phase 2 Architecture Lock 的条件(审计意见,非最终决策)

审计倾向意见:**从"2.4.0 这一个版本自身的代码事实"看,该仓库结构清晰、依赖面为零、可选适配器全部默认关闭且失效降级平稳、状态持久化方式(原子写 JSON + 追加 JSONL)与已审计的 xinchao-nian 同构、与 Haven-Ombre 之间没有能力重叠或冲突,具备作为"独立 Mind Layer 候选组件"被纳入 Phase 2 讨论范围的技术基础。** 但以下三点需要人工在 Phase 2 之前明确决策,而非由本审计代为决定:(1) 是否接受"冻结在没有 tag 的裸提交 `6948329` 上"这种版本管理方式,而不要求上游补发一个正式 tag/release;(2) 许可证锁定在 MIT 快照的选择是否符合项目整体许可证策略(尤其是仓库已经在后续版本切到 AGPL-3.0,若未来想跟进上游功能会牵涉许可证变更);(3) 该仓库与 xinchao-nian 内嵌副本(3.3.6)事实上已经出现代码分叉,若最终决定采用独立仓库版本,需要人工确认不会与 xinchao-nian 现有集成产生功能对不齐(例如 3.3.6 已有的 `DOMAIN_AFFINITY` 记忆共振能力,2.4.0 里完全没有)。这是审计意见,不是最终决策,最终决策由人工做出。
