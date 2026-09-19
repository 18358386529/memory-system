# 清单 3:Event Semantics(源码模式,固定9行事件类型)

**审计模式声明:本报告为源码审计,非运行时审计。** 下表"当前是否记录"一列只回答"代码中是否存在该事件概念的定义/埋点",严格区分"代码层面已定义"与"代码层面部分实现"与"代码层面未找到对应实现"三种状态,**不使用"已实现""正在记录""已验证"等暗示运行时状态的措辞**,因为四个目标系统当前没有本地运行实例可供实测。

**禁止推断规则已逐条遵守**:下表中 retrieved 与 delivered 分列不同代码路径与不同存储位置;delivered 与 considered/accepted 不假设因果关系;accepted 与 action、action 与 success 均分别举证;"未执行"与"拒绝"分别标注;凡无直接代码证据处一律写 `unknown` 并说明原因,不做无证据推断。

## Commit SHA 核对(与清单2一致)

| 本地路径 | 参考仓库 | 锁定 SHA(与实测HEAD一致) |
|---|---|---|
| /home/user/18358386529/haven-ombre | Yinglianchun/Haven-Ombre | 284c9c7b0e51a0ba0032c7028f705d72458cb304 |
| /home/user/18358386529/ombre-brain | P0luz/Ombre-Brain | 6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5 |
| /home/user/18358386529/xinchao-nian | tianyupaipai-cmd/xinchao-nian | 97f1bdcc76b748fa516b7c79a0aa10234143795f |
| /home/user/18358386529/kiwi-mem | LucieEveille/kiwi-mem | b01a0c506f4f10f90f30d408c0291f16720f0718 |

方法说明:对四个仓库分别执行 `grep -rniE` 搜索 9 个事件类型的英文字面量(大小写不敏感,含大小写混合命中),命中后逐一打开源文件人工核实上下文语义是否与目标事件概念相符(避免"RESULT"这类常见词的误报);同时按中文语义关键词(如"矛盾"/"确认"/"召回"/"注入"/"投递")交叉搜索,因为多数中文项目的事件语义并不使用英文字面量命名。凡下表未注明具体行号的字段,均已在源码中定位但因内容过长未逐行引用,证据ID给出文件路径+函数/类名以便复核。

---

## 一、haven-ombre

| 事件类型 | 当前是否记录(代码层面) | 字段(代码中的实际字段名) | 触发点(函数/文件) | 存储(写到哪里) | 可查询(是否有对应查询/读取接口) | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|
| OBSERVED | 部分实现(无字面量"OBSERVED",但有语义对应的原始输入落盘) | (a) raw_events 表:role/text/source/conversation_id/session_id/created_at;(b) conversation_turns 表:user_text/assistant_text/model/route | (a) `RawEventStore.ingest()`;(b) `GatewayStateStore`(具体写入调用点未在本次审计中定位到 conversation_turns 的 INSERT 语句所在函数,仅确认表结构存在,标记 `unknown` 该表当前是否有生产写入路径调用) | (a) `state/raw_events.sqlite`;(b) `buckets_dir/gateway_state.db` | 是:`RawEventStore.search()`/`list_events_between()`(raw_events.py:205-316) | raw_events.py:98-203;gateway_state.py:88-113 | 两套"原始观测"落点并存(raw_events 面向对话原文归档,conversation_turns 面向按轮次的会话回放),未见二者字段/写入路径的统一说明,可能是同一语义的重复实现或分工不明确,标记为待人工确认(非本报告可裁决范围) |
| RETRIEVED | 部分实现 | bucket_manager 内部搜索排序返回的候选集合(未见持久化的"检索命中"日志表/文件) | `BucketManager` 的搜索方法(多维加权检索,见 bucket_manager.py 头部注释"搜索策略:主题域预筛→多维加权精排");`memory_relevance.py`/`recall_policy.py` 参与排序 | **未发现持久化存储**:检索候选是内存中的临时结果,未见写入任何表/文件记录"本次检索返回了哪些候选" | 否(因未持久化,无法查询历史检索记录本身;只能查询召回后生效的下游动作,如 injected_buckets) | bucket_manager.py(类头注释,`class BucketManager` 附近);recall_policy.py:1312(`class RecallPolicy`) | **RETRIEVED 本身在 haven-ombre 中没有独立持久化记录**,只有其下游产物(DELIVERED,见下)被记录到 `injected_buckets`。若需要"曾经被检索到但未被注入"的数据(即 retrieved ≠ delivered 的差集),当前代码找不到落点,是明确缺口 |
| DELIVERED | **有明确实现** | session_id, round_id, bucket_id, injected_at | `GatewayStateStore` 的注入记录写入逻辑(表结构定义于 `_init_db`,写入方法未在本次审计逐一展开调用栈,但表本身在 `server.py:183` 处已确认被生产实例化) | `buckets_dir/gateway_state.db` 表 `injected_buckets`、`recent_context_injections`、`injection_debug` | 是:`get_last_injected_at()`(gateway_state.py:224-241) | gateway_state.py:36-66;server.py:183 | 只记"哪个bucket在哪一轮被注入",不记该bucket在提示词中的具体位置/权重/是否被截断,若需要更细粒度的"投递详情"需看 `injection_debug.payload_json`(结构未展开审计,标记 `unknown`) |
| CONSIDERED | 未发现(无独立持久化) | — | — | — | — | (负面证据)已对 gateway_state.py、bucket_manager.py、recall_policy.py 通读类/函数签名,未见"模型是否真的读取/权衡了某条注入记忆"的埋点或字段 | DELIVERED(注入到提示词)不等于 CONSIDERED(被模型实际权衡),haven-ombre 代码中没有区分这两者的机制,这是明确缺口,不应把 injected_buckets 直接等同于"已被考虑" |
| CHOICE | **有明确实现**(针对"是否写入新记忆"这一决策,不是"是否使用某条已有记忆"的决策) | candidate_id, source, decision(pending/grow/skipped/rejected等), surprise_score, reasons[], repeat_count, max_existing_similarity, max_candidate_similarity | `MemoryWriteGate.evaluate()` | `state/memory_write_candidates.jsonl`(文件名可配置) | 是:`list_recent()`(memory_write_gate.py,紧接 record() 之后定义) | memory_write_gate.py:110-247(WriteGateDecision于110-120行,evaluate于159行起,record于230行起) | 仅覆盖"是否创建新记忆"的CHOICE,不覆盖"检索到多条候选后选哪条注入"的CHOICE(后者在 recall_policy.py 中以排序分数体现,但未见落盘为可回放的"选择记录") |
| ACTION | 部分实现 | 写入成功后的 bucket_id 及 `_record_v3_bucket_event`/ledger相关调用(haven-ombre 是否有对应 ombre-brain 的 v3_runtime/ledger_mirror 机制,**未在 haven-ombre 中发现**同名结构,标记 `unknown`) | `BucketManager.create()`/`update()`等 CRUD 方法本身即是"落实动作"的执行点 | Markdown bucket 文件的实际写入(`frontmatter.dumps(post)` 写文件) | 是:`BucketManager.get()` 等读取接口 | bucket_manager.py:113-233(create方法) | "ACTION"在 haven-ombre 中等价于"bucket文件的实际CRUD操作被执行",本身有明确证据,但没有独立于bucket文件本身之外的"动作日志"记录动作的发起原因/触发上下文,若CHOICE与ACTION需要可追溯关联(如"为什么创建了这个bucket"),需要联合读取 memory_write_candidates.jsonl(CHOICE)与bucket文件本身(ACTION结果),没有显式的外键式关联字段 |
| RESULT | 部分实现 | prompt_tokens, completion_tokens, cached_tokens 等用量字段;以及 CHOICE 记录中的 decision 结果字段本身 | `upstream_usage` 表(写入调用点未在本次审计中逐一展开,标记 `unknown` 具体写入函数) | `buckets_dir/gateway_state.db` 表 `upstream_usage` | 未见专门的查询接口(仅表结构确认,读取方式 `unknown`) | gateway_state.py:115-138 | RESULT 目前只覆盖"上游LLM调用的token用量结果",不覆盖"某条注入记忆是否真的帮助生成了更好的回复"这一语义上更贴近记忆系统评估的RESULT,后者未发现任何持久化记录 |
| CONFIRMED | 部分实现(仅限"待写入候选"审核流程,不覆盖"已存在记忆被再次证实") | item["status"] = "confirmed", item["confirmed_at"] | `reflection_engine.py` 中处理"daily chat memory candidate"审核结果的方法(候选从pending→confirmed/rejected) | 未见独立表,推断写回同一个 pending 列表的持久化文件(`_save_daily_chat_memory_pending()`,存储介质在本次审计中未展开定位,标记 `unknown`) | 未见专门查询接口 | reflection_engine.py:2540-2579 | 该 CONFIRMED 语义是"人工/AI审核通过了一条候选记忆",不是"某条已有记忆的内容被后续对话再次印证为真"。后者(即经典意义上的记忆置信度被强化)在 haven-ombre 中**未发现**对应实现;memory_edges.py 的 RELATION_TYPES 中虽有 `"supports"` 枚举值(memory_edges.py:18),但未见任何代码路径实际调用 `add_edge` 写入 relation_type="supports" 的边,标记为**已定义但未见使用证据** |
| CONTRADICTED | 部分实现(枚举已定义,未见触发逻辑) | relation_type="contradicts" | `MemoryEdgeStore.add_edge()`(RELATION_TYPES 集合中含 `"contradicts"`,memory_edges.py:17) | `state/memory_edges.jsonl`(若有调用) | 是:`list_edges()`/`related_edges()` | memory_edges.py:7-25(枚举定义) | **仅发现该关系类型的枚举定义,未发现任何代码路径实际以 `relation_type="contradicts"` 调用 `add_edge()`**(已对 haven-ombre 全仓 grep `contradicts` 及调用 `add_edge.*contradict`,除枚举定义行外无其他命中)。即 haven-ombre 目前没有自动检测/记录"矛盾"的具体触发逻辑,这与 kiwi-mem 形成鲜明对比(kiwi-mem 有完整的 `detect_contradictions()` 实现,见下) |

---

## 二、ombre-brain

**重要限定**:ombre-brain 存在两条完全独立的"事件"代码路径,必须分别评估,不可混为一谈——(1) `LedgerMirror`(JSONL,已被 `bucket_manager.py` 直接实例化并在生产CRUD路径调用);(2) `MemoryEvent`/`DecisionRecord`/`cluster`/`fabric`(经代码路径追踪确认**未被生产入口 server.py 导入**,详见清单2"发现的冲突"及本报告末尾)。下表按此区分逐项标注。

| 事件类型 | 当前是否记录(代码层面) | 字段(代码中的实际字段名) | 触发点(函数/文件) | 存储(写到哪里) | 可查询(是否有对应查询/读取接口) | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|
| OBSERVED | 未发现字面量对应,但 `TraceCreated` 事件语义上覆盖"一条新记忆被观测/写入"这一时刻 | event_type="TraceCreated", trace_id, trace_kind, payload, body_hash | `BucketManager._record_ledger_event()` 调用点(`bucket_manager.py:1794-1804`,紧随 create 之后) | `<buckets_dir>/_ledger/...` LedgerMirror JSONL | 是:`LedgerMirror.iter_events()`/`verify_integrity()` | ombre-brain/src/bucket_manager.py:1794-1804;eventsourcing/ledger_mirror.py:24-51 | `TraceCreated` 记录的是"bucket被创建"这一CRUD动作,不是"原始对话被观测"这一更早的时刻(ombre-brain 未见类似 haven-ombre `raw_events.sqlite` 的原始对话归档表,已对 `find src -iname "*raw_event*"` 确认无同名文件),即 ombre-brain **没有 haven-ombre 意义上的 OBSERVED(原始对话落盘)** 概念,只有"记忆被创建"这一更晚阶段的记录 |
| RETRIEVED | 部分实现 | `RetrievalEngine`/`PolicyGatedRetrievalScorer`(`ombrebrain/retrieval/`目录下)——**这套检索评分器仅在未接入生产的 LegacyRuntime/v3体系内被引用**(`legacy_runtime.py` 的 `score_retrieval_bucket`/`rank_retrieval_candidates`方法),生产路径的实际检索逻辑在 `bucket_manager.py`/`server.py`内,未见其调用这些v3检索类 | `ombrebrain/retrieval/engine.py`、`ombrebrain/retrieval/scoring.py` | 未见持久化(检索为内存计算) | `unknown` | ombrebrain/retrieval/engine.py(类定义);app/legacy_runtime.py:422-454(score_retrieval_bucket/rank_retrieval_candidates,均属v3未接线路径) | 与 haven-ombre 相同,生产路径的 RETRIEVED 没有独立持久化;v3路径虽有更完整的 RetrievalEngine/Scorer 抽象,但未接入生产,不能作为"当前记录"的证据 |
| DELIVERED | 未发现明确的生产级持久化(与 haven-ombre 的 injected_buckets 表对应物,在 ombre-brain 中未找到同名或同构表) | — | — | — | — | (负面证据)已对 ombre-brain `src/` 全文 grep `inject`(不区分大小写)相关表名/CREATE TABLE,未见与 haven-ombre `injected_buckets`/`gateway_state.db` 对应的结构 | **明确缺口**:ombre-brain 相比 haven-ombre 缺少"记忆被实际注入到提示词"这一环节的持久化记录。这是两个仓库在DELIVERED环节上的结构性差异,需人工确认是ombre-brain尚未实现、还是该职责被移到了未审计到的其他文件 |
| CONSIDERED | 未发现 | — | — | — | — | 同 haven-ombre,无证据 | 无 |
| CHOICE | 部分实现(仅"是否判定为同一事件/是否合并"的LLM判断,属临时计算,非持久化决策日志) | judgement.get("confidence"), judgement.get("resolved"), judgement.get("same_event") | `dehydrator.py` 中的去重/合并判断函数(`_dedupe`类方法附近,dehydrator.py:1114-1188);`tools/_common.py:969-975,1396-1403` | **不持久化**:这些 confidence/resolved 值仅用于当次调用的 if 分支判断,未见写入任何日志文件 | 否 | dehydrator.py:1114-1188;tools/_common.py:94,969-975,1396-1403 | 与 haven-ombre 的 `MemoryWriteGate`(有完整 decision 日志文件)相比,ombre-brain 的"是否合并/是否视为同一事件"判断**没有持久化的CHOICE记录**,判断依据(confidence阈值 `_SAME_EVENT_CONFIDENCE_MIN`、`_PLAN_LLM_CONFIDENCE_MIN`)仅在内存中生效,无法事后审计"当时为什么做了这个选择" |
| ACTION | **有明确实现**(生产LedgerMirror路径) | event_type ∈ {TraceCreated, TraceUpdated, TraceHardDeleted, TraceRestored, TraceDeletedToArchive, TraceTouched, TraceArchived} | `BucketManager._record_ledger_event()` 的 8 处调用点 | `<buckets_dir>/_ledger/...` LedgerMirror JSONL | 是:`ledger_integrity_report()`(bucket_manager.py:741-769,含 `iter_events()`遍历) | bucket_manager.py:718-739(方法定义),1794/2773/2826/2977/3184/3253/3306/3998(8处调用点) | 这套ACTION记录覆盖桶级CRUD生命周期,是本报告在 ombre-brain 中找到的**最完整、最明确接线到生产的事件记录**,但事件类型是"发生了什么CRUD操作"而非九类语义中"模型选择执行了什么动作"的ACTION,两者概念不完全对齐,总控需注意术语混淆风险 |
| RESULT | 未发现独立的结果/成效记录(LedgerMirror的payload字段可能携带部分结果信息,但结构因bucket_type/action不同而不统一,未见专门的"结果"schema) | payload(dict,内容随event_type变化) | 同ACTION | 同ACTION | 同ACTION(LedgerMirror的payload混杂在事件记录中,非独立RESULT表) | bucket_manager.py:718-739 | RESULT语义与ACTION语义在ombre-brain的LedgerMirror设计中被合并进同一条JSONL记录(未做区分),不构成独立可查询的RESULT维度 |
| CONFIRMED | 未发现 | — | — | — | — | 已对 relation_store.py 全文核实,RELATION_TYPES(caused_by/causes/continuation_of/continues/related_to/same_event/custom)中**没有 supports/confirms 类型**(对比haven-ombre的memory_edges.py多出的"supports"枚举,ombre-brain连枚举都没有) | 明确缺口,且是ombre-brain相比haven-ombre更弱的一项(haven-ombre至少有未使用的枚举,ombre-brain连枚举定义都不存在) |
| CONTRADICTED | 未发现字面量,relation_store.py 的固定关系类型集合中也**没有** contradicts/supersedes 类枚举 | — | — | — | — | relation_store.py:11-14(`_FIXED_RELATION_TYPES = frozenset({"caused_by","causes","continuation_of","continues","related_to","same_event"})`,无contradicts) | **明确缺口**:ombre-brain 的关系类型枚举中完全没有矛盾/替代关系,这与 haven-ombre(有contradicts枚举,虽未使用)和 kiwi-mem(有完整实现)都形成对比,是三者中CONTRADICTED覆盖最弱的一个 |

### 关于 ombre-brain 未接线的 MemoryEvent/事件溯源架构(补充说明,不计入上表"是否记录"判断)

`ombrebrain/protocol/schemas.py` 的 `MemoryEvent` dataclass 本身字段设计上覆盖面很广(actor/visibility/confidence/parent_event_ids/cluster_term等),理论上可以承载九类事件语义中的大部分,但:
- 经调用链追踪(`grep -rn "build_v3_runtime"` 全仓非测试代码零命中;`web/system.py:187` 的唯一直接调用点使用 `tempfile.TemporaryDirectory` 且文档字符串明确"不在用户vault创建WAL状态"),**该架构当前没有被生产流程触发写入真实 buckets_dir 下的WAL文件的代码路径证据**。
- 因此本报告对该架构下的九类事件一律不计入上表"当前是否记录"的判断依据,仅在此单独说明其存在,留给总控判断是否要将其作为记忆系统2.0的目标事件模型基础。

---

## 三、xinchao-nian(xinchao/src,Node.js,独立于Python实现)

| 事件类型 | 当前是否记录(代码层面) | 字段(代码中的实际字段名) | 触发点(函数/文件) | 存储(写到哪里) | 可查询(是否有对应查询/读取接口) | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|
| OBSERVED | 部分实现(语义对应"状态转换发生前后的快照对比",不是原始对话归档) | before{}/after{}(状态快照),revisionBefore/revisionAfter | `summarizeTransition()`/`TransitionJournal.recordTransition()` | JSONL文件(路径由调用方传入TransitionJournal构造函数) | 是:`TransitionJournal.list()` | transition-journal.js:78-141 | 该系统的"观测"对象是内部心智状态(drives/fatigue/consciousness)的变化,不是外部对话内容本身的观测,与九类事件语义原意(观测到一条新信息)有语义偏差,标记为语义近似而非完全对应 |
| RETRIEVED | 未发现 | — | — | — | — | 已对 `xinchao/src/*.js` 全文搜索 retrieve/recall/召回,除 `ombre-client.js`(与外部ombre-brain HTTP通信的客户端)可能涉及调用远端检索接口外,未见本地检索/召回的持久化记录逻辑(ombre-client.js内部细节未逐行审计,标记`unknown`) | ombre-client.js(726行,未逐行展开) | 待人工补充审计:`ombre-client.js` 是否记录了向 ombre-brain 发起检索请求的调用日志,本次未深入 |
| DELIVERED | **有明确实现** | delivered(bool), alreadyDelivered(bool), deliveredAt(ISO字符串), contextDigest | `TransitionJournal.recordContext()`;`context-envelope.js` 中判断 `alreadyDelivered` 的逻辑 | 同上JSONL文件(记录type='context_envelope') | 是:`TransitionJournal.list({types:['context_envelope']})` | transition-journal.js:143-173;context-envelope.js:163-229,351-352 | 这是四个仓库中DELIVERED语义**字段命名最直接**的实现(字面量就是delivered/alreadyDelivered),但其"投递"对象是"整个上下文信封是否已经发给过模型",颗粒度是会话级而非单条记忆级,与haven-ombre的`injected_buckets`(单条bucket粒度)颗粒度不同,不可直接等价对比 |
| CONSIDERED | 未发现 | — | — | — | — | 同上,未见对应实现 | 无 |
| CHOICE | 部分实现(语义对应"是否应用某个交互结果/是否触发某个动作",不是记忆写入决策) | `applyInteractionOutcome()`返回的{applied, reasonCode} | engine.js:133-186 | 通过`recordConversationEventFingerprint()`记录到state内的`recentConversationEvents[]`(该数组本身持久化在StateStore管理的JSON文件中) | 通过`interactionAlreadyProcessed()`可查询指纹去重(engine.js:112-118) | engine.js:107-186 | 此CHOICE语义是"是否对本次交互事件应用状态效果"(如drive数值调整是否生效),与记忆系统2.0语境下"是否采纳某条记忆"的CHOICE是完全不同的领域(情绪状态机 vs 记忆检索),不应混为一谈,已在清单2中说明xinchao是独立心智层 |
| ACTION | 部分实现 | applied(bool), affectedDrives[] | 同上`applyInteractionOutcome()` | 同上 | 同上 | engine.js:133-186 | 同CHOICE,是情绪状态机领域的"动作生效"概念,非记忆系统的ACTION |
| RESULT | 未发现明确对应(状态变化的delta本身可视为RESULT,但无独立"RESULT"字段命名) | driveDeltas{}, fatigueDelta, counts{}(summarizeTransition返回值) | `summarizeTransition()` | 随recordTransition写入JSONL的delta字段 | 是(随list()查询到的记录) | transition-journal.js:65-101 | delta是状态差值,语义上接近RESULT(某次动作导致的结果),但命名和设计初衷是"状态转换摘要",不是记忆检索/动作执行的结果记录 |
| CONFIRMED | 未发现 | — | — | — | — | 已搜索confirm/确认相关字面量,未见命中(除通用UI文案外) | 无 |
| CONTRADICTED | 未发现 | — | — | — | — | 已搜索contradict/矛盾/冲突,未见记忆语义相关命中(仅`engine.js`中`conflict`交互类型,语义是"用户与AI发生口角"这一情绪事件,非"两条记忆信息矛盾") | `engine.js`的`interactionType`枚举中含`"conflict"`(INTERACTION_TYPES常量,具体行号未展开,已确认非记忆矛盾语义,而是关系/情绪冲突事件) |

补充:`cabin-store.js`的`ledger[]`字段语义存疑(疑似虚拟经济/花费记账,已在清单2标注待复核),未纳入本表事件类型映射,避免无证据推断。

---

## 四、kiwi-mem

| 事件类型 | 当前是否记录(代码层面) | 字段(代码中的实际字段名) | 触发点(函数/文件) | 存储(写到哪里) | 可查询(是否有对应查询/读取接口) | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|
| OBSERVED | **有明确实现** | session_id, role, content, model, created_at(+W2-03扩展的project_id/scope_known/usage/turn_id/turn_key) | 对话轮次写入逻辑(`_LEDGER_INSERT_SQL`常量所在函数,`database.py:1901-1906`附近的写入调用点) | PostgreSQL `conversations`表 | 是:按session_id/turn_key查询(常规SELECT,索引`idx_conversations_session`支持) | database.py:103-111,572-587,1901-1906 | 代码注释与`docs/event-ledger-scope-and-reconciliation.md`一致称其为"事件账本",但严格意义上这是"对话消息记录"而非九类语义中"感知到一条待处理信息"的OBSERVED,概念上有重叠但不完全等价,已如实标注 |
| RETRIEVED | 部分实现 | `search_memories`返回的候选列表(向量相似度+关键词检索,函数在main.py中被调用,具体检索实现在database.py,未逐行定位函数名,标记该函数名`unknown`) | 记忆搜索逻辑(main.py中处理用户消息时调用) | **搜索结果本身不持久化** | 否(检索是即时计算,无持久化即无法回放") | main.py:915-941(搜索结果`memories`列表在此被处理,区分为"进入memory_lines注入"与"未进入"两部分) | RETRIEVED(全部搜索命中)与DELIVERED(实际注入的injected_ids)在代码里明确是两个不同的集合(见main.py:930-940的分档注入逻辑),但只有后者(DELIVERED)被持久化追踪,前者(RETRIEVED全集)没有留痕,这是符合"retrieved≠delivered"审计要求的证据,但也是明确缺口 |
| DELIVERED | **有明确实现** | access_count(递增), last_accessed, access_query_hashes(jsonb,查询指纹去重) | `track_memory_recall(injected_ids, user_message, conversation_id)`(main.py:957) | PostgreSQL `memories`表(UPDATE,非独立事件表,是聚合计数器,不是逐次事件日志) | 是:`SELECT access_count, access_query_hashes FROM memories ...`(database.py:2738-2739附近) | database.py:2557-2602(track_memory_recall定义);main.py:955-959(调用点,docstring"对真正被使用的记忆记一笔召回") | **重要区分**:`track_memory_recall`的中文docstring是"对真正被使用的记忆记一笔召回",但从调用点看,`injected_ids`是已经通过热度分档、真正写入了`memory_lines`(即将进入prompt)的id集合,因此其语义更贴近DELIVERED(投递到提示词)而非单纯RETRIEVED(仅搜索命中)。**该实现是聚合计数器(access_count+1),不是逐次事件明细日志**,即无法回答"具体是哪一次对话投递了这条记忆",只能回答"这条记忆总共被投递过几次、最近一次是何时" |
| CONSIDERED | 未发现 | — | — | — | — | 已对main.py/database.py搜索"considered"及中文"权衡/纳入考虑",未见对应实现(access_count只回答"被投递"而非"被模型实际使用/权衡") | 明确缺口,与haven-ombre/ombre-brain一致,四个仓库均未发现CONSIDERED的独立实现 |
| CHOICE | 部分实现(体现为"是否判定为重复/矛盾"的检测结果,而非显式的"选择"记录) | is_dup(bool), contradicted_ids(list) | `check_memory_duplicate()`+`detect_contradictions()`调用点(main.py:1256-1275,`_plan_memory_batch`附近函数) | 计算结果直接传入下一步落库函数,**判断过程本身不单独持久化**(只有最终落库结果——是否跳过/是否创建supersedes边——被持久化) | 否(判断的中间过程不可查询,只能查询其结果:新记忆是否存在、supersedes边是否存在) | main.py:1256-1292 | 同ombre-brain的dehydrator.py模式:判断逻辑本身(为什么判定为矛盾/重复)不落盘为可审计的CHOICE记录,只有判断的最终产物被持久化 |
| ACTION | **有明确实现** | `_insert_memory_tx`(创建memory)、`_invalidate_memory_tx`(使旧记忆失效)、`_create_memory_edge_tx`(创建supersedes边) | `_commit_memory_batch()` | PostgreSQL `memories`表(INSERT/UPDATE)+`memory_edges`表(INSERT) | 是:常规SELECT | main.py:1295-1320 | ACTION在这里等价于"记忆的实际写入/失效/建边操作",有明确的数据库事务边界("整批落库,任一条失败整批回滚",main.py:1298-1300注释),是四个仓库中事务一致性说明最明确的实现 |
| RESULT | 部分实现 | contradictions(计数,`_commit_memory_batch`返回值`(saved_items, contradiction_count)`) | 同ACTION | 返回值层面存在,但**未见持久化到数据库或日志文件**(仅在函数调用链中作为返回值传递,用于打印/上层汇总) | 否 | main.py:1301-1320(返回`saved_items, contradictions`) | RESULT(本批次处理了多少条、产生了多少矛盾)只在内存/日志打印层面存在,未见写入`dream_logs`式的可查询结果表(`dream_logs`表本身存在且字段设计上适合承载这类RESULT,如`memories_processed/deleted/merged`,但那是Dream批处理专属表,与本次提取批处理是否共用同一张表,`unknown`,未在本次审计中确认调用关系) |
| CONFIRMED | 未发现独立实现(与CONTRADICTED相反,kiwi-mem没有实现"记忆被再次证实/强化置信度"的对称逻辑) | — | — | — | — | 已对main.py/database.py搜索confirm/确认,除UI文案与配置项名称外未见记忆置信度强化相关代码 | **明确缺口**:kiwi-mem有完整的CONTRADICTED检测与处理(见下),但没有对称的"新信息与旧记忆一致时,增强旧记忆置信度/热度"的显式逻辑(`access_count`递增是"被使用"的通用计数,不特指"被验证为真"这一更精确的CONFIRMED语义) |
| CONTRADICTED | **有明确实现**(四个仓库中最完整) | contradicted_ids(list of memory id), edge_type="supersedes", reason | `detect_contradictions()`(纯计算,标题相似度>0.7且内容相似度0.4~0.85判定为疑似矛盾)→`_commit_memory_batch()`中调用`_invalidate_memory_tx(old_id)`+`_create_memory_edge_tx(new_id, old_id, "supersedes")` | PostgreSQL:旧记忆被标记失效(具体失效字段`unknown`,未展开`_invalidate_memory_tx`函数体确认是软删除标记还是物理更新哪个列)+`memory_edges`表新增一行 edge_type='supersedes' | 是:`SELECT * FROM memory_edges WHERE edge_type='supersedes'`(常规查询,无专门封装接口) | database.py:6226-6296(detect_contradictions完整实现);main.py:1265-1320(调用与落库) | 有内建保护规则,不是无脑判定:手动录入(`source='user_explicit'`)和锁定记忆(`is_permanent`)不参与自动矛盾检测(database.py:6254-6261),即CONTRADICTED的判定范围受到显式白名单排除,这一限制条件本身也是审计应记录的事实 |

---

## 跨仓库九类事件覆盖总览矩阵

| 事件类型 | haven-ombre | ombre-brain(生产LedgerMirror路径) | ombre-brain(未接线v3路径) | xinchao-nian | kiwi-mem |
|---|---|---|---|---|---|
| OBSERVED | 部分实现(raw_events.sqlite) | 未发现(无原始对话归档表) | 理论可覆盖(MemoryEvent) | 部分实现(状态快照,语义偏移) | 有明确实现(conversations表) |
| RETRIEVED | 部分实现(无持久化) | 部分实现(无持久化) | 理论可覆盖 | 未发现(本地) | 部分实现(无持久化) |
| DELIVERED | 有明确实现(injected_buckets) | 未发现 | 理论可覆盖 | 有明确实现(delivered/alreadyDelivered) | 有明确实现(access_count,聚合非明细) |
| CONSIDERED | 未发现 | 未发现 | 未发现 | 未发现 | 未发现 |
| CHOICE | 有明确实现(write_gate候选日志) | 部分实现(不持久化) | 理论可覆盖 | 部分实现(情绪领域,非记忆) | 部分实现(不持久化判断过程) |
| ACTION | 部分实现(bucket CRUD本身) | 有明确实现(TraceCreated等7种) | 理论可覆盖 | 部分实现(情绪领域) | 有明确实现(事务化写入) |
| RESULT | 部分实现(upstream_usage) | 未发现独立字段(混入payload) | 理论可覆盖 | 部分实现(delta,语义偏移) | 部分实现(返回值,不持久化) |
| CONFIRMED | 部分实现(枚举"supports"未见调用;候选审核confirmed状态) | 未发现(枚举中无supports/confirms) | 理论可覆盖 | 未发现 | 未发现 |
| CONTRADICTED | 部分实现(枚举"contradicts"未见调用) | 未发现(枚举中无contradicts) | 理论可覆盖 | 未发现(仅情绪conflict,非记忆矛盾) | **有明确实现(唯一完整闭环)** |

"理论可覆盖"特指 ombre-brain 未接线的 MemoryEvent/eventsourcing 架构在字段设计上具备承载该事件类型的能力,但**没有代码证据表明它当前被生产流程调用**,不应被解读为"已实现"。

---

## 发现的冲突与待人工裁决项

1. **ombre-brain 的 ledger/event-sourcing 设计与 haven-ombre 的 raw_events/moments/edges 设计定位不同,不构成直接的"兼容或矛盾"关系,但存在覆盖面倒挂现象,需人工裁决**:
   - haven-ombre 的生产路径(raw_events.sqlite + memory_moments.sqlite + memory_edges.jsonl/entity_edges.jsonl + gateway_state.db)在 **DELIVERED、CHOICE(写入候选)** 两项上有明确落地实现,但 **CONTRADICTED/CONFIRMED 只停留在关系类型枚举定义,未见任何实际调用**(已对全仓 grep 确认无 `add_edge` 调用传入 `relation_type="contradicts"` 或 `"supports"` 的证据)。
   - ombre-brain 的生产LedgerMirror路径在 **ACTION(桶级CRUD)** 上覆盖最完整(7种事件类型,均有实际调用点),但在 **DELIVERED 完全缺失、CONTRADICTED/CONFIRMED 连枚举定义都没有**。
   - ombre-brain 的未接线v3路径(MemoryEvent/cluster/fabric)在字段设计上最完备,但没有任何生产调用证据。
   - **净效果**:如果记忆系统2.0直接"合并"这两个仓库的现有代码,会出现能力互补但没有一个仓库单独具备完整九类事件覆盖的局面,且 CONFIRMED/CONTRADICTED 在两个Python仓库里都近乎空白(haven-ombre 有名无实,ombre-brain 完全没有),**只有 kiwi-mem 有该两项中 CONTRADICTED 的完整实现**。这意味着若要在 haven-ombre/ombre-brain 体系内补全 CONFIRMED/CONTRADICTED,kiwi-mem 的 `detect_contradictions()` + supersedes edge 模式是现成的、经过白名单保护规则打磨过的参考实现,值得人工评估是否移植其逻辑(而非重新设计),但这已超出本报告的裁决权限,仅作为发现提交。
   - 待人工裁决:(a) 记忆系统2.0的九类事件是否要统一落到 haven-ombre 现有的 SQLite/JSONL 组合里,还是采用 ombre-brain 未接线的 MemoryEvent WAL 架构重新实现;(b) CONFIRMED/CONTRADICTED 是否直接移植 kiwi-mem 的检测规则(标题相似度>0.7 且内容相似度0.4~0.85)还是重新设计阈值;(c) haven-ombre `memory_edges.py` 中已定义但未使用的 `contradicts`/`supports` 枚举,是遗留的"未完成功能"还是"预留位"，需要原作者/项目历史层面的确认(本报告只能确认代码现状,无法判断意图)。

2. **kiwi-mem 的 `docs/event-ledger-scope-and-reconciliation.md` 与代码的一致性结论(重申自清单2)**:文档描述的 `conversations` 表三态语义、`ledger_reconcile.py` 的只读审计行为,经与 `database.py:572-660` 及 `1840-1854` 逐一核对,**未发现不一致**。但需注意文档标题使用的"event ledger"一词在 kiwi-mem 代码里的实际所指是"对话消息表",与本审计九类事件语义(OBSERVED..CONTRADICTED)中作为"通用事件流"载体的"ledger"概念不完全同义,总控在跨仓库统一术语时需避免直接套用。

3. **"considered"(被模型实际权衡)在四个仓库中均无任何实现证据**,是九类事件里覆盖最薄弱的一项,四个系统都只能证明"某条记忆被投递到了提示词里"(DELIVERED),没有一个系统能证明"模型真的读取/使用了这条记忆来生成回复"。这是一个所有四个系统共有的、需要人工正视的结构性缺口,不属于某一个仓库的实现疏漏,而更像是这类"记忆注入"架构普遍缺乏的可观测性维度。

4. **xinchao-nian 的事件/状态模型与另外三个仓库的"记忆事件"语义存在维度错位,需要总控明确澄清边界**:xinchao-nian 的 OBSERVED/CHOICE/ACTION/RESULT 等语义近似项,实际指向的是"情绪驱动力状态机"(drives/fatigue/consciousness)的转换事件,而非"某条记忆信息经历了观测-检索-投递-使用-确认/矛盾"这一记忆生命周期。若总控在设计统一的九类事件表结构时直接套用 xinchao-nian 现有字段(如 `applied`/`affectedDrives`),会与另外三个仓库的记忆语义产生错配,需要人工先确认 xinchao-nian 在记忆系统2.0中的定位(心智状态层 vs 记忆存储层)后再决定是否/如何映射。

5. **ombre-brain 未接线架构是否应被视为"违规的第二实例"或"实验性预留",本报告不擅自判断**:该架构(cluster/raft/distributed/fabric/eventsourcing/decision)在代码量和设计精细度上远超实际生产使用的 LedgerMirror,但完全没有生产调用路径。是遗留的重构半成品、面向未来分布式部署的预研代码,还是应被清理的技术债,属于需要项目历史/原作者意图判断的问题,本审计只能如实呈现"存在但未接线"这一代码事实。
