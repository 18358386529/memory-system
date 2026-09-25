# 17号审计：状态持久化 / 心跳机制 / reflection_engine.py 全文核查

## 0. 审计模式声明 + 锁定 commit 复核

本报告为**纯源码只读审计**，不运行任何服务、不修改任何生产代码/配置/数据/文档，仅追加写入本文件一处。

复核结果（`git rev-parse HEAD`，与任务书给定值完全一致，已确认）：

```
心潮念 3.3.6 (xinchao-nian):  97f1bdcc76b748fa516b7c79a0aa10234143795f
Haven-Ombre:                  284c9c7b0e51a0ba0032c7028f705d72458cb304
```

审计范围内额外发现但**不在任务书两个锁定仓库范围内**、本报告不做实施判断的旁证材料：
- `/home/user/18358386529/xinchao-nian/ombre-brain/`：xinchao-nian 仓库内自带的一份**独立打包的 Ombre Brain 副本**（`VERSION=2.6.5`，`src/ombrebrain/...` 微内核式代码结构，与本次审计目标仓库 `haven-ombre`（无 VERSION 文件、`server.py`/`gateway.py` 等扁平文件结构）**代码结构明显不同**，很可能是另一条产品线/更早或并行的打包版本，而非同一份代码的两个副本）。这份代码本身位于 `xinchao-nian` 仓库内、属于同一把锁定 commit，读取它不违反范围限制；但它不是任务书指定的"Haven-Ombre"目标，本报告只在与心跳机制相关的地方引用它的 `compose.yaml` 作为旁证，不对它做独立审计。
- `/home/user/18358386529/xinchao-dynamic-mind/`：与 `xinchao-nian` 平级的另一个目录，看起来是心潮念的后续/开发版本（有 `heartbeat-store.test.js`、`http-api.test.js` 等锁定仓库里没有的测试），**完全不在锁定 commit 范围内，未审计，仅在正文中提示其存在**，不作为结论依据。

---

## 1. 问题1：状态持久化——12维 Drive、Emotion、ThoughtPool 最适合放在哪里？

### 1.1 心潮念现状：`state-store.js` 完整实现（已确认）

全文仅 52 行，`/home/user/18358386529/xinchao-nian/xinchao/src/state-store.js`：

- **读**（`read()`，8-21行）：`readFile` 整个文件后 `JSON.parse`；文件不存在（`ENOENT`）时用工厂函数造初始 state 并立即 `write()` 落盘。
- **写**（`write()`，23-40行）：`JSON.stringify(state, null, 2)` 全量序列化 → 写临时文件 `${path}.${pid}.${timestamp}.tmp`（`open(..., 'wx', 0o600)`，`wx` 保证临时文件名唯一不冲突）→ `handle.sync()`（fsync）→ `rename(temp, path)` 原子替换。Windows 分支单独处理（先 `rm` 再 `rename`，因为 Windows 的 `rename` 不能覆盖已存在文件）。**原子性证据确凿**：单个写操作要么完全写入新内容要么保持旧内容，不会出现半截文件。
- **并发**（`update()`，42-51行）：`#queue` 是一个**进程内**的 `Promise` 链，每次 `update(mutator)` 调用都串行排到这条链的末尾（`read()` → `mutator()` → `write()`），保证**同一 Node.js 进程内**多个并发调用不会互相踩踏（读到的一定是上一次写完的最新值）。**但这把锁只在单个 JS 进程的内存里，不是文件锁/跨进程锁**——如果同一份 `state.json` 被两个独立的 Node 进程（比如水平扩容出的第二个副本）同时 `update`，两边各自的 `#queue` 互不知情，会出现经典的"读-改-写"竞态（后写的进程用自己读到的旧值覆盖，丢失另一进程的更新）。**状态：已确认**（代码层面 `#queue` 是类字段，作用域仅限当前进程）。

### 1.2 写入频率：不是只有 15 分钟一次（已确认，重新核实）

`config.js:29`：`SETTLE_INTERVAL_MINUTES` 默认 `15`，驱动 `server.js:1505` 的 `setInterval(runCycle, ...)`——这是"结算周期"，但**绝不是唯一写入点**。`server.js` 中对 `updateState`（内部调用 `store.update`）的调用点共 **30 处**，其中明确由"每次真实互动事件"触发、与 15 分钟定时器无关的包括：

- `recordConversationEvent`（`server.js:890-905`）：每次收到 MCP 或 REST `/v1/conversation-event` 的真实对话事件都会调用一次 `updateState`，`source` 可以是 `'mcp'`/`'api'`/`'heartbeat'`，**这是逐次互动级别的写入，不是 15 分钟一次**。
- `classifyExchange`（`server.js:864-888`）：8 分钟节流窗口内**只节流"分类"这一步**，但只要过了节流窗口，同样会触发一次 `updateState`。
- MCP 侧的 `awareness`（list/scan/confirm/dismiss，`server.js:966-1007`）、`handoff note`（`server.js:1009+`）、`/v1/drive-feedback` REST 端点（`server.js:1467-`）等，都是**按需触发**（用户/客户端主动调用时才写），与定时器完全无关。

**结论（已确认）**：心潮念的写入频率是"15 分钟周期性结算 + 任意时刻的互动/MCP 调用触发"的混合模式，且后者在实际使用中往往比周期性结算更频繁。这意味着单文件方案要承受的实际 IOPS 远高于"每 15 分钟一次"的印象。

### 1.3 单文件 JSON 的扩展性边界（已确认，逐条给出代码证据）

1. **每次写入都是全量读+全量写，没有增量/分片**：`state-store.js` 的 `read()`/`write()` 没有任何按字段更新的接口，任何一次哪怕只改一个数字的调用（如 `applyDriveFeedback` 只改一个 drive 数值）也要经过"读整份 JSON → 反序列化 → mutate → 序列化整份 JSON → 落盘"全流程（`engine.js:713-723` `applyDriveFeedback` 只改 `state.drives[key]`，但被 `updateState` 包裹后走的是全量 `read/write`）。文件越大，每次小改动的开销越线性增长。
2. **状态对象里已经存在若干"无界增长"的字典字段**：
   - `state.interactionUsage[day]`、`state.dreamUsage[day]`、`state.barkUsage[day]`、`state.daytimeEmergenceUsage[day]`（`engine.js:145,176,878,916,953,979`）都是以"日期字符串"为 key 的计数器，**全文搜索未发现任何对这四个字典的 `delete`/裁剪逻辑**（对比 `emotionDays` 在 `emotion.js:307` 显式调用 `pruneDays(state.emotionDays, now)` 做了裁剪，`sessionOverlays`/`satisfactionPlateaus` 也各自在 `engine.js:86,413` 有显式 `delete`）。也就是说这 4 个字典会随实例运行时间**每天新增 1 个 key、永不清理**——单实例长期运行（比如运行数年）体量仍然很小（一年 365 个 key），但这证明"单文件方案里确实存在没有主动做生命周期管理的字段"，一旦未来往这类字典里塞更大的值（比如逐条情绪日志而不是计数器），增长速度会立刻放大。
   - 其余高频数组字段都已经显式加了上限：`recentBarkMessages` 上限 8（`engine.js:370`）、`recentDreams` 上限 20（`engine.js:876`）、`recentConversationEvents` 上限 `MAX_RECENT_CONVERSATION_EVENTS=256`（`engine.js:21,62,130`）、`emotionJournal` 上限 `JOURNAL_MAX_SAMPLES`（`emotion.js:270,296`）。这部分已经做了容量控制，**已确认**不是无限增长风险点。
3. **没有多用户/多实例（分片）设计**：`state-store.js` 的构造函数只接受一个 `path`，`newState()`（`engine.js:269-307`）产出的是**单一实体**（一份 drives + 一份 emotion + 一份 thoughtPool）的状态，代码里没有"按 user_id/instance_id 分文件"或"一份文件里存多个实体"的任何结构。若要支持多用户/多实例，需要在应用层自行加一层"每用户一个 `StateStore` 实例（各自一个文件）"或重写为数据库表，**当前代码结构完全没有为此预留接口**。
4. **无测试�covery**：`xinchao/test/` 目录下没有任何 `state-store.test.js` 或 `heartbeat-store.test.js`（已用 `ls` 核实，仅有 23 个其他模块的测试文件），这两个模块在锁定的 3.3.6 版本里缺乏自动化回归保护。**已确认**（对比：功能上更新的兄弟目录 `xinchao-dynamic-mind`——不在本次审计范围——已经补了这两个测试文件，说明这是后续版本已经意识到并着手补的缺口，但 3.3.6 本身没有）。

### 1.4 Haven 现有 SQLite / 状态基础设施逐一核实（已确认，非猜测，直接读代码）

#### (a) `gateway_state.py`（`GatewayStateStore`，全文 704 行，已全文读完）

- 6 张表：`request_rounds`、`injected_buckets`、`injection_debug`、`recent_context_injections`、`conversation_turns`、`upstream_usage`。全部是 `session_id`/`round_id`/`profile_id` 维度的**多行历史表**，没有一张是"单行、按 profile_id 主键、代表当前状态"的 KV 型表。
- **写入模式**：每个方法内部都是 `conn = self._connect()`（新建一个 `sqlite3.connect`）→ 执行 SQL → `conn.commit()` → `conn.close()`，**没有跨方法的事务**，也没有连接池/长连接复用。
- **并发安全**：全文 `grep "PRAGMA"` **零命中**——没有设置 `journal_mode=WAL`、没有设置 `busy_timeout`。SQLite 默认 `journal_mode=DELETE`（回滚日志模式）+ `busy_timeout=0`，意味着两个进程同时写同一张表时，后到的写操作会立刻抛 `database is locked` 异常而不是等待重试。**已确认**（代码事实，非经验推测）。
- **历史可查询性**：这几张表天然支持"按 session_id/时间范围查询历史轨迹"（如 `list_conversation_turns_between`），这是心潮念单文件 JSON 完全不具备的能力（心潮念要查历史只能翻 `journal.js`/`transition-journal.js` 的独立日志文件，状态本身不保留历史）。
- **容量治理**：`conversation_turns` 和 `upstream_usage`、`injection_debug` 都有显式的"写入后立刻 `DELETE ... WHERE id NOT IN (SELECT ... ORDER BY id DESC LIMIT N)`"滚动裁剪逻辑（`gateway_state.py:352-365,496-547,597-606`），这一点比心潮念的 4 个未裁剪计数器字典做得更严格。

#### (b) `raw_events.py`（`RawEventStore`，读了 1-220 行，含完整 schema 与并发模式，其余为查询/清洗辅助函数，抽样确认模式一致）

- 单表 `raw_events`（`id, source, source_event_id, event_hash, role, text, created_at, ingested_at, ...`），`UNIQUE(source, event_hash)` 天然去重，`fts5` 全文索引可选启用（`_init_db` 里 try/except，FTS5 编译失败会优雅降级 `fts_enabled=False`，不影响主表可用性——**已确认**这是一处考虑周到的降级设计）。
- 类文档字符串自称 `"Append-only-ish raw dialogue archive"`——**这是"追加式对话原文归档"，天生适合存"事件流"，不适合存"当前这一刻的12维状态快照"**。若要塞进 12 维状态，得新增一张完全不同结构的表，`raw_events.py` 本身的表结构不能直接复用。
- 同样没有 `PRAGMA` 设置（同 gateway_state.py 的并发风险）。

#### (c) `memory_moments.py`（`MemoryMomentStore`，读了 180-459 行，即 `__init__`/`_init_db`/增删改查主干，其余 1500 行是记忆文本解析/图谱算法，与本问题无直接关系，未做逐行核对但已确认不含额外的状态存储表——`grep "CREATE TABLE"` 只命中这三张）

- 三张表：`memory_moments`（bucket 文本切片索引）、`memory_moment_edges`（记忆关系图）、`memory_retrieval_aliases`（别名索引）。全部服务于"记忆文本的检索索引"，与"AI 自身此刻的情绪/驱力状态"语义上完全不是一回事，**不适合也不应该**被复用来存 12 维状态。

#### (d) `persona_engine.py`（**本次审计新发现的、比前三者更贴近的候选**，读了 1-60 行与 230-1024 行，含完整 schema 与全部读写方法）

这是任务书没有点名、但审计中逐一 `grep "CREATE TABLE"` 全仓库后发现的**结构和语义都最接近**心潮念 12 维状态的既有模块：

- `persona_global_state`（`persona_engine.py:251-266`）：**主键 `profile_id`（单行代表"当前全局关系状态"）**，字段含 `affinity/dominance/defensiveness/trust`（4 维关系状态，legacy 的 Big-Five 5 维已废弃保留仅为兼容旧库）。
- `persona_session_state`（`persona_engine.py:270-288`）：**主键 `(profile_id, session_id)`**，字段含 `valence/arousal/tenderness/possessiveness/longing/security/protective_drive/libido`（**8 维情感状态**）+ `mood_label`/`residue`/`inner_thought`（文本）+ `updated_at`。两张表合计 12 个数值维度，**在"多维、持续演化、需要按时间衰减"这个语义上，与心潮念的 `drives`(12维)+`emotion`(valence/arousal) 几乎是同构问题**。
- **衰减机制**：`_apply_session_decay`（`persona_engine.py:931-948`）用半衰期公式 `retention = 0.5 ** (elapsed_minutes / half_life_minutes)` 做指数回落，和心潮念 `dimensions.js` 里 `decayHalfLifeHours` 字段（grieve/anger 两维）的半衰期回落思路**在数学形式上完全一致**。
- **并发安全（关键差异，已确认的风险）**：`_ensure_session_state`（875-929行）先 `SELECT` 出当前行（若不存在则 `INSERT`），随后调用方（如 `_apply_session_delta`，977-994行）在 Python 内存里 `updated = dict(state)` 做增量计算，再单独调用 `_save_session_state`（996-1024行）执行一条 `UPDATE ... WHERE profile_id=? AND session_id=?`。**这是一次跨两次独立数据库连接、没有事务包裹、没有乐观锁（没有 `WHERE updated_at=?` 版本校验）、没有任何 `asyncio.Lock`/`threading.Lock`（全文 `grep "Lock("` 零命中）的"读-改-写"**。如果同一 `session_id` 在两个并发请求里几乎同时被评估（比如两条消息几乎同时到达网关），**会发生经典的丢失更新（lost update）**：后完成写入的请求会用自己读到的旧状态覆盖先完成的请求写入的新状态。**这一点比心潮念的 `state-store.js`（进程内用 `#queue` Promise 链严格串行化所有更新）并发安全性更弱**——心潮念至少保证了"同进程内不丢更新"，而 `persona_engine.py` 当前代码**连同进程内的串行化都没有**。**状态：已确认**（读代码得出，非推测；是否在实际流量下触发取决于同一 session 是否真的会并发到达，本报告不做运行时验证，只指出代码层面缺少防护）。

### 1.5 必须回答的问题

**(1) 心潮念 `state.json` 方案 vs Haven 现有 SQLite 方案的优劣对比：**

| 维度 | 心潮念 `state-store.js`（单文件 JSON） | Haven `gateway_state.py`/`raw_events.py`/`memory_moments.py`（SQLite，多行历史表） | Haven `persona_engine.py`（SQLite，单行状态表） |
|---|---|---|---|
| 原子性 | 临时文件+`fsync`+`rename`，单次写入原子（已确认） | 单条 SQL 语句原子；跨语句无事务包裹 | 同左；`_ensure_*`+`_save_*` 跨两次连接，**无事务** |
| 并发安全 | 进程内 `#queue` 严格串行（已确认安全），**跨进程无保护** | 无 `PRAGMA`，无 `busy_timeout`，无重试，并发写可能报 `database is locked`（已确认无防护，未验证实际报错频率） | **无任何锁**，"读-改-写"跨连接，存在已确认的丢失更新风险，比心潮念更弱 |
| 历史可查询性 | 无（状态是"当前值"，历史要另查 `transition-journal.js`） | 天然支持（`list_conversation_turns_between` 等按时间范围查询） | 有限（`persona_events`/`persona_exchange_log` append-only 表记录了事件流，但当前状态表本身不保留历史快照） |
| 迁移成本（若要把心潮念状态迁到对应基础设施） | — | 高：三张表语义都不匹配 12 维状态，需要新建表 | 中：表结构语义已经很接近（12个数值字段两张表），但字段名/维度含义不同（心潮念是 possess/monitor/crave/... 12个具体驱力名，Haven 是 affinity/valence/arousal/tenderness/... 8+4个关系-情感维度），**不能直接复用现成字段，仍需新建或大改表结构**，可复用的是"这套读写模式和衰减算法思路" |
| 扩展性上限 | 全量读写，体量线性增长；无分片/多用户设计；4个计数器字典无裁剪（已确认但影响很小） | SQLite 单文件本身有并发写吞吐上限（依赖 WAL/busy_timeout 调优，目前未调） | 同左 |

**(2) 若要迁移到 Haven 的存储基础设施，哪个最接近可以直接复用？**

- **最接近、但不能直接复用字段的模块：`persona_engine.py` 的 `persona_global_state`/`persona_session_state`**。理由：语义上都是"持续演化、需要半衰期回落的多维情感/关系状态、按 profile_id（+session_id）区分实体"，读写模式（先 ensure 再 apply delta）和衰减公式思路都可以直接照搬。**完全没有现成的**是：心潮念 12 个具体驱力维度（possess/monitor/crave/share/libido/curiosity/boredom/social/duty/reflection/grieve/anger）在 Haven 任何一张表里都不存在对应字段，`thoughtPool`（心潮念的念头队列结构，`engine.js` 里的 `newThoughtPool`/`tickThoughtPool`）在 Haven 全仓库里**没有任何同构概念**（`grep "thought_pool\|ThoughtPool"` 全仓库零命中，已确认）。
- **结构上最规范、但语义最远的模块：`gateway_state.py`**。它的"新表 + 索引 + 滚动裁剪 + 每次短连接"这套写法风格是 Haven 项目里最成熟的模板，如果要新建一张"dynamic_mind_state"表，抄它的代码风格（而不是它的现有表）最省心。
- **不适合复用的模块**：`raw_events.py`（面向事件流，非当前状态）、`memory_moments.py`（面向记忆文本索引）。

**(3) 选项与权衡（不做最终决策）：**

- **选项 A：维持现状（心潮念继续用 `state.json`，Haven 不动）**。代价：心潮念侧仍然背负"无跨进程锁、全量读写、4个字典无裁剪"这三个已确认的技术债，但因为心潮念目前是**单实例部署**（`compose.yaml` 里 `dynamic-mind` 服务没有 `replicas` 配置，`read_only: true` + 单一命名卷 `xinchao-state`），跨进程竞态目前**没有实际触发条件**；不改动，风险维持在"低概率但存在"的水平，不需要任何代码改动。
- **选项 B：心潮念继续用自己的存储，但学习 `persona_engine.py` 的表结构思路新建一张 Haven 表**，仅供心潮念以外的 Haven 组件（如未来的仪表盘）只读展示心潮念状态用。改动范围：**中**——只需要在 Haven 新增一张表 + 一个只读写入接口（心潮念侧新增一次性上报调用），不触碰 Recall/Injection/Memory Core。
- **选项 C：把心潮念状态整体迁移到 Haven 侧数据库（彻底替换 `state-store.js`）**。改动范围：**大**——需要心潮念重写整个状态读写层（`state-store.js`+`engine.js` 里所有直接操作 `state.drives`/`state.emotion`/`state.thoughtPool` 的函数都要改造成数据库读写或引入 ORM 抽象层），Haven 侧需要新建表并解决 1.4(d) 指出的并发安全缺口（否则把心潮念这种高频写入迁过去等于把丢失更新风险从"理论上存在"变成"实际会发生"）。**这条路径本报告不建议在未解决 persona_engine.py 并发缺口之前推进**，但不代表方向不可行，只是指出前置条件。

---

## 2. 问题2：心跳机制——外部 heartbeat 文件依赖是否构成实际架构风险？

### 2.1 `heartbeat-store.js` 完整实现（已确认，全文仅 15 行）

```js
export async function readOmbreHeartbeat(filePath) {
  try {
    const raw = await readFile(filePath, 'utf8');
    const value = JSON.parse(raw);
    const at = new Date(value.recordedAt ?? value.recorded_at ?? '');
    return Number.isFinite(at.getTime()) ? at : null;
  } catch (error) {
    if (error?.code === 'ENOENT') return null;
    throw error;
  }
}
```

- **读取哪个文件**：`config.heartbeat.filePath`，默认 `/memory-data/heartbeat.json`（`config.js:121`）。
- **格式校验**：只做了两件事——① `JSON.parse` 失败（包括文件内容是 `null`/数组/纯字符串导致后续 `value.recordedAt` 抛 `TypeError` 的情况）会被 `catch` 捕获但因为 `error.code !== 'ENOENT'` 而被**重新 `throw`**（不是这个函数自己吞掉）；② 时间戳解析失败（字段缺失或不是合法日期）用 `Number.isFinite(at.getTime())` 判空，返回 `null`（吞掉，不抛错）。**没有任何"未来时间戳"/"过旧时间戳"的合理性上下限校验**——这是本节最重要的发现，见 2.3。
- **文件不存在**：`ENOENT` 被显式捕获，返回 `null`，**不抛错**。
- **写入方是谁**：全仓库（`xinchao-nian` + `haven-ombre` 两个锁定仓库）搜索 `heartbeat`/`/heartbeat`（大小写不敏感）：
  - `xinchao-nian` 侧：`heartbeat-store.js` 只有 `readOmbreHeartbeat` 一个函数，**没有任何写入这个文件的代码**（心潮念自己只读不写）。
  - `haven-ombre` 侧：全文搜索 `heartbeat`（含大小写变体）**唯一命中**是 `server.py:8951` 一行注释 `# Tool 5: pulse — Heartbeat, system status + memory listing`——这是 MCP 工具 `pulse` 的注释里恰好用了"Heartbeat"这个词形容它是"系统脉搏"，和写文件毫无关系（`pulse` 工具本身不写任何文件，只读状态并返回文本，已用 `server.py:8954` 附近代码核实）。`haven-ombre` 仓库里**没有 `/heartbeat` 路由、没有任何写 `heartbeat.json` 的代码**。
  - `.env.example`（`xinchao-nian/xinchao/.env.example:103`）的注释明确写着 **"Optional heartbeat file supplied by an external memory/agent service."**——这与代码事实一致：这个文件在设计意图上就是"外部某个服务"负责写，心潮念和本次审计能看到的 Haven-Ombre 代码库都不是这个"外部服务"。**结论（已确认）：写入方是完全的外部黑盒，本次审计能访问的两个仓库源码里都找不到写入方的任何痕迹。**
- **旁证（供参考，不作为结论依据，因为不在锁定仓库范围内）**：`xinchao-nian` 仓库自带的、结构完全不同的另一份 `ombre-brain`（2.6.5，微内核版，见第0节）源码中，同样全文搜索 `heartbeat` 无命中；这份代码的 `compose.yaml` 在 `xinchao-nian` 仓库根目录（**在锁定 commit 范围内**，见 2.2），也没有为 `/memory-data` 挂载任何共享卷。

### 2.2 部署配置层面的旁证（已确认，属于 `xinchao-nian` 锁定 commit 范围内的文件）

`xinchao-nian/compose.yaml`（仓库根目录，与 `xinchao/` 子目录同属一次锁定 commit）定义了 `ombre-brain` + `dynamic-mind` 两个服务联合部署：

- `dynamic-mind` 服务的 `volumes` 只有 `xinchao-state:/app/state`（命名卷，存 `state.json`）和 `ombre-buckets:/ombre-buckets:ro`（只读挂载 OB 的桶目录）——**完全没有 `/memory-data` 这个路径的任何挂载**（`compose.yaml:76-80`）。
- 该服务还设置了 `read_only: true`（`compose.yaml:81`），容器根文件系统整体只读，进一步印证"这个容器里不会有人写 `/memory-data/heartbeat.json`"。
- `.env.example` 里虽然定义了 `MEMORY_DATA_HOST_PATH=./memory-data` 和 `OMBRE_HEARTBEAT_FILE=/memory-data/heartbeat.json` 两个环境变量（第103-105行），但这两个变量**在 `compose.yaml` 里完全没有被引用**（`grep "MEMORY_DATA_HOST_PATH\|memory-data" compose.yaml` 零命中）。

**结论（已确认）**：就"心潮念仓库自带的这份一键部署配置"而言，心跳文件读取路径在默认部署形态下**大概率永远是 `ENOENT`**（除非部署者手动在 `compose.yaml` 之外自行补一个共享卷 + 一个外部写入进程）。代码里对 `ENOENT` 的处理是优雅的（见 2.1），所以这不会导致崩溃，但意味着"心跳机制"这个特性在这份仓库给出的标准部署方式下**默认从未真正生效过**——这与文档自述"Optional heartbeat file supplied by an external memory/agent service"是一致的（文档没有撒谎，只是这个"external service"需要部署者自己另外准备，仓库本身不提供）。**这不构成文档与代码不一致，而是构成"这个特性在默认部署形态下形同虚设"的架构事实，值得向总控汇报。**

### 2.3 `applyOmbreHeartbeat` / `synchronizeOmbreHeartbeat` 完整调用链（已确认）

```
server.js:runCycle()（每 SETTLE_INTERVAL_MINUTES=15 分钟一次）
  → synchronizeOmbreHeartbeat()（server.js:92-115）
      → readOmbreHeartbeat(config.heartbeat.filePath)          [try/catch 包裹，任何异常 log 后返回 null，不影响本轮结算]
      → 若 recordedAt 非空：
          → updateState(..., current => {
                const previous = Date.parse(current.lastHeartbeatAt ?? '');
                if (Number.isFinite(previous) && previous >= recordedAt.getTime()) return current;  // 只接受"更新的"时间戳
                return applyOmbreHeartbeat(current, recordedAt).state;
            })
              → applyOmbreHeartbeat(input, now)（engine.js:698-702）
                  → applyConversationEvent(input, {}, now)      // 关键：心跳被当成一次"空互动事件"处理！
                      → state.consciousness = 'awake'           // engine.js:529，无条件设置
                      → state.lastConversationAt = iso(now)     // engine.js:530，无条件设置为心跳文件里的时间戳
                      → state.sleepStartedAt = null             // engine.js:536
                  → result.state.lastHeartbeatAt = iso(now)     // engine.js:700
```

**关键发现（已确认，任务书未点名但审计中读代码发现的重要机制）**：`applyOmbreHeartbeat` 内部直接复用了 `applyConversationEvent`（真实互动事件的处理函数），这意味着**外部心跳文件的时间戳一旦被采纳，会把心潮念的 `consciousness` 直接设为 `'awake'`，并把 `lastConversationAt`（决定"是否该睡着"的关键字段，`engine.js:493-498` 的 `idleMinutes` 就是拿它和 `now` 做差）直接设成心跳文件里的时间戳，而不是设成"读取心跳的当下时刻"`new Date()`**。也就是说，**外部心跳文件的时间戳被当作"进行了一次真实对话"的时间戳来对待**，唯一的区别只是 `applyConversationEvent` 里 `event={}` 空对象（不带具体互动内容/类型）。这是一处设计上把"外部黑盒的时间戳"信任等级等同于"真实互动"的耦合点。

### 2.4 五个风险场景逐一判断（全部基于上述代码路径，已确认或已确认不存在，无猜测）

**场景1：心跳文件长期不更新（外部服务挂了）——会不会误判"清醒"？**

**已确认：不会。** `synchronizeOmbreHeartbeat` 的守卫条件 `previous >= recordedAt.getTime()` 保证：如果文件内容一直不变（`recordedAt` 一直是同一个旧值），`observed` 恒为 `false`，`lastHeartbeatAt`/`lastConversationAt`/`consciousness` **都不会被这条路径再次刷新**。同时 `settleState`（`engine.js:492-498`）里判断"是否该睡着"用的是 `state.lastConversationAt`，如果既没有心跳更新也没有真实对话，`idleMinutes` 会正常累积，超过 `sleepAfterMinutes`（默认90分钟）后系统会正常转入 `sleeping`——**行为正确，不会因为心跳卡死而误判清醒**。唯一的隐性问题是：**没有任何告警机制**去主动发现"心跳已经连续 N 天没更新了"，这属于"运维可观测性缺口"而非"逻辑错误"。

**场景2：心跳文件被意外清空/损坏——当前代码会怎样？**

**已确认：优雅降级，不会崩溃，但降级发生在调用方而不是模块自身。** 若文件内容变成空字符串/非法 JSON/`null`/数组等：`JSON.parse` 或后续 `value.recordedAt` 访问会抛错（`SyntaxError` 或 `TypeError`），`heartbeat-store.js` 自身的 `catch` 块只特判 `ENOENT`，其余错误会被**重新 `throw` 出去**；这个异常会被 `server.js:94-98` 的外层 `try/catch` 捕获，`log('ombre_heartbeat_read_failed', ...)` 后 `return null`，本轮 `runCycle` 正常继续。**结论：损坏文件不会让心潮念崩溃或阻塞结算循环，但健壮性依赖的是调用方 `server.js` 的 try/catch，而不是 `heartbeat-store.js` 模块自身的错误处理边界设计得完备**——如果未来有人在别处直接调用 `readOmbreHeartbeat` 而忘了包 try/catch，就会真的抛出未捕获异常。

**场景3：心跳文件时间戳被错误地设置为未来时间——会不会导致"提前醒来"或计算出负的空闲时长？**

**已确认：会，这是本节找到的唯一一处真实的、未被处理的风险点。** `readOmbreHeartbeat` 对时间戳只做"是否是合法日期"的校验（`Number.isFinite`），**没有任何"不能晚于当前时间太多"的上限校验**。若外部写入了一个未来时间戳：
1. `synchronizeOmbreHeartbeat` 的守卫只比较"是否比上次新"，未来时间戳显然满足"更新"，会被接受；
2. `applyOmbreHeartbeat` 会把 `state.lastConversationAt` 设成这个未来时间戳；
3. 下一轮 `settleState` 计算 `idleMinutes = (nowMs - Date.parse(state.lastConversationAt)) / 60000`（`engine.js:493`）会得到**负数**，恒小于 `sleepAfterMinutes`，导致系统**在真实时间追上这个虚假未来时间戳之前，永远不会转入睡眠状态**；
4. `contactIdleAllowed`（`engine.js:704-709`）里 `now.getTime() - lastHeartbeatMs`（`lastHeartbeatMs` 也被这条路径同步设成了同一个未来值）同样会是**很大的负数**，导致 `contactIdleAllowed` 长时间返回 `false`——**自主联系/做梦推送等依赖"离得够久"才触发的功能会被这个错误时间戳"冻结"，直到真实时间追上为止**。

**结论：这是一个已确认存在、尚未被代码处理的边界条件缺口**——一个被污染或恶意构造的心跳文件（哪怕只是外部服务的系统时钟出错、时区换算错误多算了一天）可以让心潮念长时间"卡在清醒态、且自主联系功能被冻结"，且没有任何日志会特别标注"这是一个异常未来时间戳"（只会打印正常的 `ombre_heartbeat_observed` 日志）。

**场景4：多实例/多进程同时读写心跳文件——是否有竞态条件？**

**已确认：读取侧本身安全，风险不在心潮念这一侧。** `readOmbreHeartbeat` 只做 `readFile`（单次系统调用读整个文件，Node.js 的 `readFile` 在文件系统层面本身不会读到"半个 rename 中"的文件——前提是写入方也采用了"临时文件+rename"的原子写模式，但**这一点本次审计无法验证，因为写入方代码不在可见范围内**，标记为**未知**）。心潮念自己**从不写**这个文件（2.1 已确认），所以"心潮念多实例互相踩踏心跳文件"这个场景不成立——心潮念只是消费者。真正的竞态风险在于"如果外部有多个写入方实例同时写这份心跳文件"，但这部分代码完全不在本次可审计范围内，**状态：未知**，本报告如实标注而不猜测。

**场景5：心跳文件路径不存在（首次部署）——初始化行为是什么？**

**已确认：优雅处理。** `ENOENT` → `readOmbreHeartbeat` 返回 `null` → `synchronizeOmbreHeartbeat` 直接 `return null`，不触碰状态。心潮念的初始状态 `newState()`（`engine.js:276`）本来就把 `lastHeartbeatAt` 初始化为 `null`，`contactIdleAllowed`（`engine.js:705`）对 `!state.lastHeartbeatAt` 显式返回 `false`——**首次部署、心跳文件从未出现过的情况下，系统行为是"自主联系类功能保持关闭，直到第一次真实对话事件把 `lastHeartbeatAt` 设上（`applyConversationEvent` 里 `engine.js:534` 也会同步设置这个字段，不依赖心跳文件）"，是安全的默认行为**。结合 2.2 的发现（标准部署下这个文件大概率永远不存在），场景5实际上是"心潮念标准部署的常态"而不是边缘情况。

### 2.5 最小改进方向（仅列选项与改动范围，不预设选择，不写代码）

| 选项 | 解决的问题 | 改动范围估计 | 备注 |
|---|---|---|---|
| A. 给心跳时间戳增加"未来容差"上限校验（如超过 now+N 分钟则视为无效，等同 ENOENT） | 场景3（未来时间戳污染） | **小**：只需改 `heartbeat-store.js` 一个函数，增加一个比较 `at.getTime() > Date.now() + tolerance` 的分支 | 不触碰 `engine.js`/`server.js` 调用方逻辑 |
| B. 给"心跳陈旧度"增加监控/日志告警（如 `now - lastHeartbeatAt > X 小时` 时打一条 warn 日志） | 场景1的可观测性缺口 | **小**：在 `synchronizeOmbreHeartbeat` 里加一段判断+`log()` 调用 | 不改变任何状态迁移逻辑，纯观测 |
| C. 让 `applyOmbreHeartbeat` 不再直接复用 `applyConversationEvent`，改为单独维护 `lastHeartbeatAt`，不去动 `consciousness`/`lastConversationAt` | 2.3 指出的"心跳时间戳被当作真实互动时间戳"耦合问题 | **中**：需要重新梳理 `contactIdleAllowed`/`settleState`/`dreamAllowed` 等所有读 `lastConversationAt`/`consciousness` 的调用点，确认拆分后各处逻辑是否仍然成立，涉及面较广，且是心潮念自身代码，不触碰 Haven | 这是"心跳"和"真的醒着"两个概念在代码里目前合二为一，拆开是架构性改动，不是小补丁 |
| D. 引入真实对话事件作为"清醒判定"的补充/主信号，把心跳文件降级为纯粹的辅助信号（甚至可选择默认不启用，除非显式配置了真实存在的心跳服务） | 从根本上降低对一个"完全外部黑盒、无法验证其写入方是否可信"的文件的依赖程度 | **中到大**：需要重新设计"什么信号可以驱动 wake/idle 判定"这一层的产品逻辑，不是纯技术补丁 | 与 2.2 的发现（标准部署下心跳文件本来就常年不存在）相呼应——如果心跳机制在事实上很少被真正启用，弱化它的权重风险较低 |

以上四个选项**互不排斥**，A/B 是纯粹的防御性小补丁，C/D 涉及产品语义层面的取舍，本报告不建议在四者之间做最终选择，留给总控/产品决策。

---

## 3. 问题3：`reflection_engine.py` 完整核查（4364行全文已通读，无遗留未读区间）

**通读方式说明（如实交代，供复核）**：分 12 段顺序读取，覆盖区间为 `1-300 / 300-700 / 700-1100 / 1100-1500 / 1500-1900 / 1900-2300 / 2300-2700 / 2700-3100 / 3096-3425 / 3425-3754 / 3754-4083 / 4084-4364`，相邻区间均有重叠或首尾相接，**整份文件 1 到 4364 行逐行读完，无跳读、无按函数名猜测内容**。以下结论均基于实际读到的代码文本。

### 3.1 `ReflectionEngine` 类（及文件内其他结构）完整职责清单（已确认）

文件里**只有一个类** `ReflectionEngine`（`reflection_engine.py:302`），没有其他类；顶层只有常量（各类 Prompt 模板、`AFFECT_ANCHOR_HEADER`、`REFLECTION_FALLBACK_ANCHORS` 等）和这一个类。这个类承担了**至少 5 类互相独立的职责**（用类内方法分组）：

1. **单条记忆的自动打标签/建关系边**（`enrich_bucket`，582-646行；`backfill_edges_for_bucket`，648-682行）：新记忆写入后异步补 `tags`/`importance`/`confidence`，并用 LLM（`_api_classify`，1098-1113行）或启发式规则（`_heuristic_classify`，4170-4189行）给它和候选旧记忆之间建关系边（`_edges_from_classification`，1115-1154行），候选旧记忆来源见 `_candidate_buckets`（1003-1096行，近期+语义相似+同标签/domain+承诺/锚点 四路候选）。
2. **每日/每周"关系天气"反思**（`reflect`，684-950行；`run_due` 里的 daily/weekly 分支，981-1000行）：**直接**把 LLM 生成的第一人称短文写成一条 `type='feel'` 的 Bucket（`bucket_mgr.create/update`，864-903行），**没有人工确认环节**，只靠"当天素材是否够（`daily_min_memory_items`）""内容是否第一人称""是否含 Markdown 标题"这几条硬性校验（814-825行）来决定是否落盘，**限流靠"同一个 period_key 的 bucket_id 已存在就不重复生成"这一条天然去重（每天/每周最多1条）**，没有独立的"待确认队列"。
3. **日记（diary）→ 长期记忆的自动萃取**（`_maybe_extract_diary_memory`，3456-3550行；`_extract_diary_memory_candidate`，3572-3601行）：LLM 判断日记里是否有值得写入的内容（`should_write`），**若判定为真且置信度 ≥ `diary_memory_extract_min_confidence`（默认0.68）就直接 `bucket_mgr.create` 写入，同样没有人工确认步骤**，限流参数是 `diary_memory_extract_max_per_day`（默认 **1**）+ 用 `bucket_id = f"diary_memory_{key}"` 天然按天去重。
4. **"当天聊天记录 → 长期记忆候选"三段式流程（daily_chat_memory）**：这是**与心潮念 `awareness.js` 结构上最接近的机制**，详见 3.2。
5. **"当天做了什么"活动摘要**（`run_daily_activity_summary`，1951-2026行）：明确注释"这是 dashboard/handoff 用的近期事项，不是长期记忆候选"（Prompt 模板里原话），**不写入 Bucket，只返回结构化字典**给调用方（`server.py` 用于 handoff/dashboard 展示），与"沉淀为记忆"无关，本报告不再展开。

此外还有一批与本问题次要相关的辅助能力：identity role 关系边推断（1156-1337行）、affect_anchor（音乐意象）附加逻辑（4022-4168行，`memory_affect_anchor_enabled` 开关默认 `false`）。

### 3.2 daily_chat_memory：与心潮念 `awareness.js` 三段式流程的逐项对比（已确认）

**完整三段式流程（`run_daily_chat_memory`，2275-2440行 + `list_daily_chat_memory_pending`/`confirm_daily_chat_memory`，2507-2583行）**：

1. **候选生成触发条件**：由后台调度线程 `_reflection_loop`（`server.py:13383-13531`，在 `if __name__ == "__main__":` 启动块里以 `threading.Thread(daemon=True)` 方式在进程启动时无条件拉起，见3.4）每 `check_interval_minutes`（默认60分钟）调用一次 `run_due`（`reflection_engine.py:952-1001`），其中 `daily_chat_memory_mode != "off"` 且 `now_local.hour >= daily_chat_memory_hour`（默认0点后）时触发 `run_daily_chat_memory`。也支持 MCP 工具直接调用（`server.py:11449`）。候选生成前会先用游标（`_daily_chat_memory_last_raw_event_id`）避免重复处理已经处理过的 `raw_events`（2300-2336行），再用两步 LLM 调用（先分窗口摘要 `_summarize_daily_chat_memory_windows`，再从摘要提取候选 `_extract_daily_chat_memory_candidates`）生成候选，并经过一长串启发式过滤（噪声检测 `_daily_chat_memory_noise`、低价值社交噪声 `_daily_chat_memory_low_value_social_noise`、低价值片段 `_daily_chat_memory_low_value_episode`、置信度阈值、去重 `_daily_chat_memory_duplicate_candidate`）后才成为正式候选（`_normalize_daily_chat_memory_candidates`，3096-3173行）。
2. **候选的数据结构**：`{id, date, kind, title, content, tags, keywords, domain, importance, valence, arousal, confidence, source_turn_ids, source_event_ids, reason}`（3151-3167行），`id` 由 `sha1(date|kind|content)` 前10位派生（3444-3447行，天然内容去重）。
3. **确认/拒绝机制**：仅当 `daily_chat_memory_mode == "review"` 时，候选先落到**本地 JSON 文件**（`daily_chat_memory_pending_path`，默认 `<state_dir>/daily_chat_memory_candidates.json`，2507-2517行的 `list_daily_chat_memory_pending` 读取并返回 `status='pending'` 的项），再由外部调用方（MCP 工具，`server.py:11572` 列出待确认、`server.py:11599` 调 `confirm_daily_chat_memory` 确认或拒绝，2519-2583行）逐条 `confirm`/`reject`。`confirm` 时才真正调用 `_write_daily_chat_memory_candidates`（3237-3305行）把内容 `bucket_mgr.create` 成正式 Bucket；`reject` 只是把 `status` 标记为 `rejected`，**候选本体没有被写入正式记忆**。若 `daily_chat_memory_mode == "auto"`，则跳过待确认队列，生成后直接调用 `_write_daily_chat_memory_candidates` 落盘。
4. **沉淀后写入哪里**：`bucket_mgr.create(...)`（3264-3288行）——**写入的是正式生产 Bucket**（`source="daily_chat_memory"`，`extra_metadata` 里带 `daily_chat_memory_candidate_id`/`source_conversation_turn_ids`/`source_raw_event_ids` 溯源字段），**不是一张独立的数据库表**；候选本身在被确认/拒绝之前，暂存在**本地 JSON 文件**里（`daily_chat_memory_candidates.json`），**这一点与心潮念 `state.json`/`awareness.candidates` 数组存法在"用文件存候选队列"这个思路上是一致的**，都不是数据库表。
5. **限流参数逐项对比**：

| 参数 | 心潮念 `awareness.js` | Haven `daily_chat_memory`（`reflection_engine.py`） | 对比结论 |
|---|---|---|---|
| 每天上限 | `MAX_PER_DAY = 1`（`awareness.js:23`） | `daily_chat_memory_max_per_day` 默认 **10**（auto模式）/ `daily_chat_memory_review_max_per_day` 默认 **10**（review模式）（配置默认值见 `config.example.yaml:660,662`） | Haven 默认宽松 10 倍；**已确认**两边都是可配置常量，不是硬编码死值（心潮念 `MAX_PER_DAY` 是模块内 `const`，改需要改代码；Haven 是 YAML 配置项，改不需要碰代码） |
| 去重窗口 | `DEDUPE_DAYS = 7`（`awareness.js:28`，按"过去7天内同 kind+subject 组合"去重） | 无显式"天数窗口"，`_daily_chat_memory_duplicate_candidate`（2880-2908行）按"标题/正文 token 重叠度 + 来源 turn/event id 重叠度 + 同日期同话题关键词"做语义相似度去重，**去重检查是对"当前所有待确认 pending 项"全量比较，不分时间窗口**（`_refresh_daily_chat_memory_pending_items`，3347-3409行，遍历的是整份 pending 列表） | 两者去重**思路不同**：心潮念是"简单 kind+subject 字符串匹配 + 固定7天窗口"，Haven 是"内容相似度算法 + 无固定时间窗口（只受限于 pending 列表本身最多保留500条，见下）"。**已确认**：Haven 的去重覆盖范围理论上更长（只要还在 pending 里，哪怕超过7天也会比对），但没有心潮念那种"明确的、可预期的7天生命周期"设计 |
| 过期规则 | `EXPIRE_DAYS = 14`（`awareness.js:24`，按创建时间主动过期，14天后候选从"open"列表中消失） | **没有找到任何按时间过期候选的逻辑**（全文搜索 `expire`/`过期`/`stale` 在 `reflection_engine.py` 中零命中）。唯一的容量控制是 `_save_daily_chat_memory_pending`（3339-3346行）里 `payload = {"items": items[-500:], ...}`——**按"最近500条"做 FIFO 数量裁剪，不是按时间过期** | **已确认存在的差异**：心潮念有主动的、基于时间的候选生命周期管理（14天必过期，避免陈旧候选长期占用"待确认"列表），Haven 的 `daily_chat_memory` 候选**理论上可以无限期停留在 pending 状态**，只要 pending 队列没超过500条就不会被清理，这是一个真实的、Haven 侧比心潮念更"松"的设计差异 |

### 3.3 重叠与差异总结（已确认）

- **重叠部分**：两边都实现了"候选生成（LLM/规则）→（可选）人工确认→沉淀为正式记忆"的结构，都用"文件存候选队列"（而非数据库表），都有按日期天然去重的 bucket_id/id 派生方式，都有"每天最多N条"的限流参数。
- **心潮念有而 Haven 没有的**：①明确的、基于时间的 `EXPIRE_DAYS` 主动过期机制；②`MAX_PER_DAY=1` 这种非常克制的默认限流值（Haven 默认宽松10倍）；③候选生成绑定在一个"12维驱力/情绪状态机"上，awareness.js 的候选来源于心潮念自身状态轨迹的模式识别（何时该反思），而不是纯粹的"LLM 读一段聊天记录直接抽取"。
- **Haven 有而心潮念没有的**：①LLM 驱动的语义相似度去重（比心潮念的字符串匹配更智能，但覆盖时间范围不设限）；②"review"和"auto"两种模式可切换（心潮念的 awareness 确认环节是固定的、总是需要人工 confirm，没有"自动直接写入"这一档）；③候选提取前先做"分窗口摘要"两阶段 LLM 调用（更适合处理长聊天记录），心潮念的候选生成不涉及 LLM 调用（`awareness.js` 的 `scanAwareness` 是纯规则/状态轨迹扫描，不调模型）。
- **Haven 独有、心潮念完全没有对应物的机制**：`reflect()`（daily/weekly relationship weather，**无确认环节直接写入**）和 `_maybe_extract_diary_memory`（**无确认环节直接写入**）——这两个是"生成后直接落盘"，不是三段式，心潮念里没有与之对应的"AI 自动生成一段反思文字直接存为记忆"的机制（心潮念的 `awareness.js` 候选**必须**经人工 `confirm` 才会写入 Ombre，见15/16号报告已确认的 `handleAwareness` 逻辑）。**这是一个值得注意的产品设计差异：Haven 的"关系天气"和"日记萃取"信任 LLM 自主判断直接写入生产记忆，心潮念的"自我觉察"候选则坚持人工确认这一道闸门。**

### 3.4 是否接入生产路径（已确认，非"写了没接线"）

`grep "reflection_engine\|ReflectionEngine"` 结果显示：
- `server.py:125` 顶层 `import`，`server.py:177` 模块加载时立即实例化 `reflection_engine = ReflectionEngine(config)`（单例）。
- `server.py:3018-3046`（`_queue_memory_enrichment`/`_enrich_memory_async`）在**至少6个记忆写入调用点**（`server.py:8373,8392,8556,8632,8688,9577`，均已核实是 `_queue_memory_enrichment(bucket_id)` 调用）被触发，说明 `enrich_bucket` **确实挂在真实的记忆写入路径上**，不是孤立代码。
- `server.py:13347` 的 `if __name__ == "__main__":` 启动块里，`server.py:13383-13531` 定义并**无条件启动**（受 `reflection_engine.enabled and reflection_engine.auto_enabled` 两个默认为 `True` 的开关保护，`config.example.yaml:637-640` 确认默认值）一个后台守护线程 `_start_reflection_scheduler`，循环调用 `run_due`。
- `daily_chat_memory` 这一支路径本身**默认关闭**（`config.example.yaml:657` `daily_chat_memory_mode: off`），但代码路径完整、MCP 工具接口齐全（`server.py:11449,11572,11599`）、调度器会正常跳过它（因为 `effective_mode == "off"` 时 `run_daily_chat_memory` 直接 `return {"status": "disabled", ...}`，`reflection_engine.py:2288-2290`）。**这不属于此前审计反复发现的"写了但完全没接线"模式（如 gateway.py 从不调用 bucket_manager.search 那种彻底的代码孤岛），而是"接线完整、按配置开关生效，默认配置下这一个子功能是关的，其余子功能（enrich/reflect/diary_memory）默认是开的"**。这个区分很重要，本报告不预设总控会如何解读，但明确给出证据以避免与"孤岛代码"混为一谈。
- `gateway.py`/`bucket_manager.py` 两个文件全文 `grep "reflection_engine\|ReflectionEngine"` **零命中**（已确认）——`reflection_engine.py` 的所有产出都通过 `server.py`（MCP 工具层）路径写入，**不经过 `gateway.py` 的自动注入管线**，这与16号报告"两套独立记忆访问路径"的结论完全一致，本次是针对 `reflection_engine.py` 这一个具体模块的独立复核，结论一致，未发现矛盾。

---

## 4. 证据表

| 结论 | 仓库 | 文件 | 函数/位置 | 行号 | 证据 | 状态 |
|---|---|---|---|---|---|---|
| `state-store.js` 用临时文件+rename做原子写 | xinchao-nian | src/state-store.js | `write()` | 23-40 | `open(temp,'wx')`→`handle.sync()`→`rename` | 已确认 |
| `update()` 用进程内Promise链串行化，非跨进程锁 | xinchao-nian | src/state-store.js | `update()`,`#queue` | 4,42-51 | `#queue`是类实例字段 | 已确认 |
| 写入不止15分钟一次，逐互动事件也写 | xinchao-nian | src/server.js | `recordConversationEvent` | 890-905 | 每次会话事件调用`updateState` | 已确认 |
| `SETTLE_INTERVAL_MINUTES`默认15分钟 | xinchao-nian | src/config.js | — | 29 | `number('SETTLE_INTERVAL_MINUTES',15,1,1440)` | 已确认 |
| 4个日期计数器字典无裁剪逻辑 | xinchao-nian | src/engine.js | `interactionUsage`等 | 145,176,878,916,953,979 | 全文搜索无对应`delete` | 已确认 |
| `emotionJournal`/`recentDreams`/`recentConversationEvents`有上限 | xinchao-nian | src/emotion.js, src/engine.js | 多处`.slice(-N)` | emotion.js:270,296; engine.js:21,62,130,370,876 | 显式裁剪代码 | 已确认 |
| 12个drive维度定义 | xinchao-nian | src/dimensions.js | `DIMENSIONS`/`DRIVE_KEYS` | 12-111 | 逐一列出possess...anger共12个key | 已确认 |
| xinchao/test无state-store/heartbeat-store测试 | xinchao-nian | xinchao/test/ | — | — | `ls`核实23个测试文件名单中无这两个 | 已确认 |
| gateway_state.py 6张表均为多行历史表，无单行KV状态表 | haven-ombre | gateway_state.py | 全文 | 24-141 | 逐张`CREATE TABLE`读取 | 已确认 |
| gateway_state.py无PRAGMA设置 | haven-ombre | gateway_state.py | 全文 | — | 全文搜索`PRAGMA`零命中 | 已确认 |
| raw_events.py是append-only事件归档，非状态存储 | haven-ombre | raw_events.py | `RawEventStore` | 98-163 | 类文档字符串+schema | 已确认 |
| memory_moments.py三张表均服务记忆文本索引 | haven-ombre | memory_moments.py | `MemoryMomentStore` | 180-269 | schema | 已确认 |
| persona_engine.py有单行KV式状态表，语义最接近12维状态 | haven-ombre | persona_engine.py | `persona_global_state`/`persona_session_state` | 251-288 | schema含12个数值维度 | 已确认 |
| persona_engine.py状态更新是跨连接读-改-写，无锁无事务 | haven-ombre | persona_engine.py | `_ensure_session_state`/`_apply_session_delta`/`_save_session_state` | 875-1024 | 逐函数读取，`grep "Lock("`零命中 | 已确认 |
| `thoughtPool`概念在Haven全仓库无对应实现 | haven-ombre | 全仓库 | — | — | `grep "thought_pool\|ThoughtPool"`零命中 | 已确认 |
| `heartbeat-store.js`只读不写，无未来时间戳校验 | xinchao-nian | src/heartbeat-store.js | `readOmbreHeartbeat` | 1-15 | 全文仅做ENOENT和`Number.isFinite`两项校验 | 已确认 |
| 心跳文件写入方在两个锁定仓库中均无代码痕迹 | xinchao-nian, haven-ombre | 全仓库 | — | — | `grep -i heartbeat`分别核实 | 已确认 |
| xinchao-nian自带compose.yaml未挂载/memory-data | xinchao-nian | compose.yaml | `dynamic-mind.volumes` | 76-83 | 仅`xinchao-state`+`ombre-buckets:ro`两卷 | 已确认 |
| `.env.example`明确标注心跳文件为"外部服务"提供 | xinchao-nian | xinchao/.env.example | — | 103-105 | 注释原文 | 已确认 |
| `applyOmbreHeartbeat`复用`applyConversationEvent`，心跳=真实互动 | xinchao-nian | src/engine.js | `applyOmbreHeartbeat`,`applyConversationEvent` | 698-702,507-536 | 无条件设置consciousness='awake' | 已确认 |
| 未来时间戳会导致idleMinutes为负、长期无法入睡/联系冻结 | xinchao-nian | src/engine.js | `settleState`,`contactIdleAllowed` | 493-498,704-709 | 计算公式推导 | 已确认（代码推导，非运行时验证） |
| 心跳文件损坏时靠调用方try/catch兜底而非模块自身 | xinchao-nian | src/server.js | `synchronizeOmbreHeartbeat` | 92-99 | try/catch包裹`readOmbreHeartbeat`调用 | 已确认 |
| 首次部署ENOENT被优雅处理，不影响启动 | xinchao-nian | src/heartbeat-store.js, src/engine.js | `readOmbreHeartbeat`,`newState` | 11,276 | ENOENT→null，初始值本就是null | 已确认 |
| 多实例写入心跳文件是否有竞态 | — | — | — | — | 写入方代码不可见 | 未知 |
| `ReflectionEngine`是文件内唯一的类 | haven-ombre | reflection_engine.py | `class ReflectionEngine` | 302 | 全文结构 | 已确认 |
| `reflect()`（关系天气）无人工确认，直接写Bucket | haven-ombre | reflection_engine.py | `reflect` | 684-950 | 直接调用`bucket_mgr.create/update` | 已确认 |
| 日记萃取无人工确认，直接写Bucket，限额1条/天 | haven-ombre | reflection_engine.py | `_maybe_extract_diary_memory` | 3456-3550 | `diary_memory_extract_max_per_day`默认1 | 已确认 |
| daily_chat_memory三段式：候选→(review模式)pending确认→写Bucket | haven-ombre | reflection_engine.py | `run_daily_chat_memory`,`list_daily_chat_memory_pending`,`confirm_daily_chat_memory` | 2275-2440,2507-2583 | 完整调用链 | 已确认 |
| 候选暂存于本地JSON文件，非数据库表 | haven-ombre | reflection_engine.py | `_load/_save_daily_chat_memory_pending` | 3336-3346 | `open(...).write(json.dump(...))` | 已确认 |
| pending队列仅按数量裁剪(500条)，无按时间过期机制 | haven-ombre | reflection_engine.py | `_save_daily_chat_memory_pending` | 3343 | `items[-500:]`，全文搜索`expire`零命中 | 已确认 |
| Haven默认每日候选上限10条(auto/review均为10) | haven-ombre | config.example.yaml | `daily_chat_memory_max_per_day`等 | 660,662 | 配置默认值 | 已确认 |
| Haven去重靠语义相似度，无固定天数窗口(对比心潮念DEDUPE_DAYS=7) | haven-ombre | reflection_engine.py | `_daily_chat_memory_duplicate_candidate` | 2880-2908 | 遍历全部pending项，非按日期窗口过滤 | 已确认 |
| daily_chat_memory默认关闭(mode:off)，但代码/MCP接口/调度完整接入 | haven-ombre | config.example.yaml, server.py | `daily_chat_memory_mode`, MCP工具 | config:657; server.py:11449,11572,11599 | 配置默认值+调用点 | 已确认 |
| reflection_engine在生产入口(`__main__`)启动后台调度线程 | haven-ombre | server.py | `_reflection_loop`,`_start_reflection_scheduler` | 13347,13383-13531 | 无条件启动(受enabled/auto_enabled默认True保护) | 已确认 |
| enrich_bucket挂在至少6处记忆写入调用点上 | haven-ombre | server.py | `_queue_memory_enrichment` | 3018-3046,8373,8392,8556,8632,8688,9577 | grep调用点 | 已确认 |
| gateway.py/bucket_manager.py从不引用reflection_engine | haven-ombre | gateway.py, bucket_manager.py | — | — | `grep`零命中 | 已确认 |
| xinchao-nian内嵌另一份结构完全不同的ombre-brain(2.6.5)副本 | xinchao-nian | ombre-brain/ | `VERSION`,`src/ombrebrain/` | — | 目录结构对比 | 已确认（仅作旁证，不在审计目标范围） |

---

## 5. 本次审计遵守的红线自查

- **未修改 Recall**：未编辑 `recall_policy.py`/`bucket_manager.py`/`memory_relevance.py`/`gateway.py` 等任何文件，仅使用 `Read`/`Grep`/`Bash`（只读命令：`git rev-parse`/`wc -l`/`find`/`ls`/`cat`）对其只读查看。
- **未修改 Injection**：未编辑 `gateway.py` 的任何注入相关函数。
- **未修改 Memory Core**：未编辑 `bucket_manager.py`、任何桶文件、任何 embedding 文件。
- **未新建数据库/表**：本次审计只读取了 `gateway_state.py`/`raw_events.py`/`memory_moments.py`/`persona_engine.py` 的既有 schema，未执行任何 `CREATE TABLE`/写库操作。
- **未写生产代码**：未对 `xinchao-nian`（含其内嵌的 `ombre-brain` 子目录）或 `haven-ombre` 两个克隆仓库做任何 `git add/commit/checkout -b/clean`，未编辑其中任何一个文件，未创建任何新文件。
- **未做最终实现决策**：问题1、问题2、问题3 中涉及"怎么改"的部分均以选项+权衡的形式列出（1.5节的选项A/B/C，2.5节的选项A/B/C/D），未指定"应该选哪个"。
- **唯一的写操作**：本报告文件本身 `docs/audit/phase0-1.5/17_dynamic_mind_persistence_heartbeat_reflection_audit.md`，未触碰 00-16 号任何已有报告，未触碰 `docs/design/dynamic_mind_integration_proposal_v1.md`。
- **未联网**：全程未使用 WebFetch/WebSearch/git fetch/git pull，两个仓库均为已克隆的本地目录，只用本地 `git rev-parse HEAD` 复核。

---

## 总结（供总控参考，三句话）

1. **状态持久化**：心潮念的单文件JSON方案原子性可靠但跨进程无锁、无分片；Haven 里语义最接近的不是任务书点名的三个模块，而是审计中新发现的 `persona_engine.py`（`persona_global_state`/`persona_session_state`），但它的读-改-写本身**没有锁保护、并发安全性其实弱于心潮念现状**，直接迁移前需要先补上这个缺口。
2. **心跳机制**：写入方在两个锁定仓库中都是完全的黑盒（代码零痕迹），心潮念自带的标准 `compose.yaml` 甚至没有为它挂载共享卷，多数场景已被优雅处理，但**未来时间戳缺乏合理性校验是一个已确认、真实存在、目前无防护的边界条件缺口**。
3. **reflection_engine.py**：全文4364行已通读，其中 `daily_chat_memory` 子功能是与心潮念 `awareness.js` 结构对应的三段式候选流程（候选生成→JSON文件暂存→confirm/reject→写Bucket），代码和MCP接口完整、已挂上生产调度器，**但默认配置是关闭的**；同一文件里的 `reflect()`（关系天气）和日记萃取则是**无需人工确认、默认开启**的自动写入机制，这是与心潮念"必须人工确认"设计哲学不同的一处产品差异，值得单独关注。