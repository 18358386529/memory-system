# 清单 2:Data Schema(源码模式)

**审计模式声明:本报告为源码审计,非运行时审计。** 四个目标系统当前没有本地运行实例;所有结论均来自对已只读克隆到本地的源码快照(含 SQL DDL 字符串、dataclass/字段定义、JSONL 写入逻辑、Markdown/YAML frontmatter 构造代码)的直接阅读。"字段存在于代码中" 不等于"该字段在生产库中已被写入过真实数据",两者已在下表中尽量分列说明。本报告不新增、不建议任何新表结构;凡现有结构未覆盖的语义,一律记"未发现,缺口"。

## Commit SHA 核对

审计开始前执行 `git -C <path> rev-parse HEAD` 核对,四个仓库均与任务给定 SHA **一致**:

| 本地路径 | 参考仓库 | 给定 SHA | 实测 HEAD | 一致性 |
|---|---|---|---|---|
| /home/user/18358386529/haven-ombre | Yinglianchun/Haven-Ombre | 284c9c7b0e51a0ba0032c7028f705d72458cb304 | 284c9c7b0e51a0ba0032c7028f705d72458cb304 | 一致 |
| /home/user/18358386529/ombre-brain | P0luz/Ombre-Brain | 6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5 | 6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5 | 一致 |
| /home/user/18358386529/xinchao-nian | tianyupaipai-cmd/xinchao-nian | 97f1bdcc76b748fa516b7c79a0aa10234143795f | 97f1bdcc76b748fa516b7c79a0aa10234143795f | 一致 |
| /home/user/18358386529/kiwi-mem | LucieEveille/kiwi-mem | b01a0c506f4f10f90f30d408c0291f16720f0718 | b01a0c506f4f10f90f30d408c0291f16720f0718 | 一致 |

补充说明:`xinchao-nian` 仓库根目录下还内嵌了一份完整的 `ombre-brain/` 子目录副本(`/home/user/18358386529/xinchao-nian/ombre-brain/`),与独立克隆的 `ombre-brain` 仓库结构高度相似(同样有 `src/ombrebrain/eventsourcing`、`decision`、`ledger_property.py` 等)。本报告未对这份内嵌副本与独立 `ombre-brain` 仓库逐文件 diff,仅在"发现的冲突"一节标记为待人工复核项,不擅自判断是否为"违规的第二实例"。

---

## 一、haven-ombre(Yinglianchun/Haven-Ombre 二改,当前 Memory Core 本体)

### 1.1 Markdown Bucket(主存储,YAML frontmatter + 正文)

| 存储 | 表/文件 | 字段 | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| Markdown+YAML | `buckets/{permanent,dynamic,archive,feel}/{domain}/{name}_{id}.md` | id, name, tags[], domain[], valence, arousal, importance, type, created, last_active, updated_at, activation_count, confidence(可选), period(可选), date(可选), pinned(可选bool), protected(可选bool), anchor(可选bool), resolved(可选bool), digested(可选bool), source(可选) | YAML 标量/列表(id/name/source=str; tags/domain=list[str]; valence/arousal/confidence=float 0~1; importance=int 1~10; created/last_active/updated_at=ISO8601 str; activation_count=int; pinned/protected/anchor/resolved/digested=bool) | 文件名内含 id(逻辑主键,代码内以 metadata["id"] 作为唯一标识,无数据库层唯一约束) | 无数据库索引;通过目录分区(type/domain)做物理"索引" | 通过 `memory_edges.jsonl`、`entity_edges.jsonl`、`memory_moments.sqlite` 中的 moment_edges 以 bucket_id 字符串外键关联 | haven-ombre | 284c9c7b | `/home/user/18358386529/haven-ombre/bucket_manager.py:57-233`(BucketManager.create,metadata dict 构造于 163-201 行,confidence 处理于 180-181 行) | **可复用的"可信度"字段已存在**:`confidence`(float 0~1,可选,create() 参数默认 None 不写入)。未发现"证据链"字段(如 evidence_ids 列表)在 bucket 主体 YAML 中;证据/引用另由 `source_refs.py`、`entity_edges.py` 的 evidence 字段承担(见下)。 |

### 1.2 raw_events.sqlite —— 原始对话归档

| 存储 | 表/文件 | 字段 | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| SQLite | `state/raw_events.sqlite` 表 `raw_events` | id, source, source_event_id, event_hash, role, text, created_at, ingested_at, conversation_id, session_id, client, metadata_json | id=INTEGER AUTOINCREMENT; source/source_event_id/event_hash/role/text/created_at/ingested_at/conversation_id/session_id/client=TEXT; metadata_json=TEXT(JSON字符串) | `id` PK;`UNIQUE(source, event_hash)`;条件唯一索引 `idx_raw_events_source_event_id`(source, source_event_id) WHERE source_event_id != '' | idx_raw_events_created(created_at DESC,id DESC);idx_raw_events_source(source, created_at DESC);idx_raw_events_role(role, created_at DESC);可选 FTS5 虚表 `raw_events_fts`(content='raw_events') | 无外键约束,靠 conversation_id/session_id 字符串关联到 gateway 会话 | haven-ombre | 284c9c7b | `/home/user/18358386529/haven-ombre/raw_events.py:98-162`(RawEventStore.\_init_db) | 无 confidence/置信度字段(原始事件本身不携带可信度语义,合理)。role 被硬限制为 {"user","assistant"}(`ALLOWED_RAW_ROLES`,raw_events.py:15),不支持 system/tool 角色。 |

### 1.3 memory_moments.sqlite —— 桶内片段索引 + 片段间边 + 检索别名

| 存储 | 表/文件 | 字段 | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| SQLite | `state/memory_moments.sqlite` 表 `memory_moments` | moment_id, bucket_id, section, text, ordinal, source, source_id, text_hash, metadata_json, created_at, updated_at | moment_id=TEXT; bucket_id/section/text/source/source_id/text_hash=TEXT; ordinal=INTEGER; metadata_json=TEXT(JSON); created_at/updated_at=TEXT(ISO) | `moment_id` PK | idx_memory_moments_bucket(bucket_id, ordinal) | bucket_id 字符串关联 Markdown bucket 文件 | haven-ombre | 284c9c7b | `/home/user/18358386529/haven-ombre/memory_moments.py:200-221` | 无 confidence 字段 |
| SQLite | 同库 表 `memory_moment_edges` | source, target, bucket_id, relation_type, confidence, reason, created_at | source/target/bucket_id/relation_type/reason=TEXT; confidence=REAL; created_at=TEXT | 复合 PK(source, target, relation_type) | idx_memory_moment_edges_source(source);idx_memory_moment_edges_target(target) | source/target 为 moment_id,指向 memory_moments 表(无外键约束,应用层维护) | haven-ombre | 284c9c7b | `/home/user/18358386529/haven-ombre/memory_moments.py:222-241` | **已有 confidence(REAL)字段,可直接复用**,无需新增置信度列。 |
| SQLite | 同库 表 `memory_retrieval_aliases` | bucket_id, moment_id, alias_text, alias_key, source, text_hash, updated_at | bucket_id/moment_id/alias_text/alias_key/text_hash/updated_at=TEXT; source=TEXT CHECK IN('title','moment') | 复合 PK(bucket_id, moment_id, alias_key, source) | idx_memory_retrieval_aliases_alias_key(alias_key, bucket_id);idx_memory_retrieval_aliases_bucket(bucket_id) | bucket_id/moment_id 字符串关联 | haven-ombre | 284c9c7b | `/home/user/18358386529/haven-ombre/memory_moments.py:242-267` | — |

### 1.4 memory_nodes.sqlite —— 桶级情绪/激活分数

| 存储 | 表/文件 | 字段 | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| SQLite | `state/memory_nodes.sqlite` 表 `memory_nodes` | bucket_id, importance, valence, arousal, salience, activation_count, last_active, facets_json, updated_at | bucket_id=TEXT; importance/valence/arousal/salience/activation_count=REAL; last_active/updated_at=TEXT; facets_json=TEXT(JSON) | `bucket_id` PK | 无额外索引(仅主键) | bucket_id 关联 Markdown bucket | haven-ombre | 284c9c7b | `/home/user/18358386529/haven-ombre/memory_nodes.py:177-195` | 无 confidence 字段(此表语义是情绪/显著度,非可信度,不属该维度缺口) |

### 1.5 memory_edges.jsonl / entity_edges.jsonl —— 显式关系边(JSONL,非数据库)

| 存储 | 表/文件 | 字段 | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| JSONL | `state/memory_edges.jsonl`(MemoryEdgeStore) | source, target, relation_type, confidence, reason, created_at | source/target/relation_type/reason(截断240字符)/created_at=str; confidence=float(clamp 0~1,3位小数) | 无(逐行 JSON,应用层按 source+target+relation_type 去重覆盖) | 无(全量加载后内存过滤,`related_edges()` 支持 min_confidence 阈值) | source/target 为 bucket_id | haven-ombre | 284c9c7b | `/home/user/18358386529/haven-ombre/memory_edges.py:7-193`(RELATION_TYPES 定义于7-25行,枚举17种关系,含 contradicts/supports/evidenced_by) | **已有 confidence + reason 字段,可直接复用作为"证据/置信度"载体**;relation_type 枚举中已包含 `contradicts`、`supports`、`evidenced_by`,可作为 CONFIRMED/CONTRADICTED 语义的既有落点(见清单3)。 |
| JSONL | `state/entity_edges.jsonl`(EntityEdgeStore) | subject, relation, object_text, bucket_id, confidence, evidence, created_at | subject/relation/object_text/bucket_id/evidence/created_at=str; confidence=float(默认0.65,clamp) | 无(逐行 JSON,按 subject+relation+object_text 去重覆盖) | 无 | bucket_id 关联来源 bucket | haven-ombre | 284c9c7b | `/home/user/18358386529/haven-ombre/entity_edges.py:11-206`(ENTITY_RELATIONS 定义于11-20行;add_edge 含 confidence/evidence 于172-192行) | **已有 confidence + evidence(证据原文摘录)字段,是现成的"信念+证据"结构**,relation 枚举含 likes/dislikes/prefers/fears/boundary/habit/participates_in/shared_anchor,面向人物关系事实,非通用 Claim 表,但可作为该子域的复用范例。 |

### 1.6 embeddings.db —— 向量存储

| 存储 | 表/文件 | 字段 | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| SQLite | `buckets_dir/embeddings.db` 表 `embeddings` | bucket_id, embedding, model, dimension, updated_at | bucket_id=TEXT; embedding=TEXT(JSON数组字符串,非原生向量类型/非pgvector); model=TEXT; dimension=INTEGER; updated_at=TEXT | `bucket_id` PK | 无(未见向量索引,相似度检索在应用层用余弦计算,注释"cosine similarity search"确认为暴力/线性扫描) | bucket_id 关联 Markdown bucket | haven-ombre | 284c9c7b | `/home/user/18358386529/haven-ombre/embedding_engine.py:1-84`(CREATE TABLE 于70-81行) | 无 confidence/model置信度字段(合理,embedding 本身无此语义) |

### 1.7 gateway_state.db —— 会话级注入/轮次/用量追踪(生产已接线)

| 存储 | 表/文件 | 字段 | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| SQLite | `buckets_dir/gateway_state.db` 表 `request_rounds` | session_id, round_id, completed_at | 均TEXT/INTEGER | 复合PK(session_id, round_id) | — | — | haven-ombre | 284c9c7b | `/home/user/18358386529/haven-ombre/gateway_state.py:25-33` | — |
| SQLite | 同库 表 `injected_buckets` | session_id, round_id, bucket_id, injected_at | TEXT/INTEGER/TEXT/TEXT | 复合PK(session_id, round_id, bucket_id) | idx_injected_lookup(session_id, bucket_id, injected_at DESC) | bucket_id 关联 Markdown bucket | haven-ombre | 284c9c7b | `/home/user/18358386529/haven-ombre/gateway_state.py:36-51` | 这是清单3中 **DELIVERED 事件的既有落点**(见下) |
| SQLite | 同库 表 `injection_debug` | id, session_id, round_id, created_at, payload_json | INTEGER PK/TEXT/INTEGER/TEXT/TEXT(JSON) | id AUTOINCREMENT | idx_injection_debug_lookup(session_id, id DESC) | — | haven-ombre | 284c9c7b | `/home/user/18358386529/haven-ombre/gateway_state.py:53-66` | — |
| SQLite | 同库 表 `recent_context_injections` | session_id, round_id, injected_at | TEXT/INTEGER/TEXT | 复合PK(session_id, round_id) | idx_recent_context_lookup(session_id, injected_at DESC) | — | haven-ombre | 284c9c7b | `/home/user/18358386529/haven-ombre/gateway_state.py:68-83` | — |
| SQLite | 同库 表 `conversation_turns` | id, profile_id, session_id, round_id, created_at, user_text, assistant_text, model, client, route | INTEGER PK / TEXT×多 | id AUTOINCREMENT;UNIQUE(profile_id, session_id, round_id) | idx_conversation_turns_recent, idx_conversation_turns_session | — | haven-ombre | 284c9c7b | `/home/user/18358386529/haven-ombre/gateway_state.py:88-113` | 这是清单3中 **OBSERVED 事件的候选既有落点**(逐轮用户/助手原文) |
| SQLite | 同库 表 `upstream_usage` | id, session_id, round_id, created_at, model, route, prompt_tokens, completion_tokens, prompt_cache_hit_tokens, prompt_cache_miss_tokens, cached_tokens, cache_read_input_tokens, cache_creation_input_tokens, usage_json | INTEGER PK / TEXT / INTEGER(可空) / TEXT(JSON) | id AUTOINCREMENT | idx_upstream_usage_lookup(session_id, id DESC) | — | haven-ombre | 284c9c7b | `/home/user/18358386529/haven-ombre/gateway_state.py:115-138` | 这是 **RESULT(用量/结果)事件的候选既有落点** |

实例化证据(确认此表是生产路径,非孤立测试代码):`server.py:183` `gateway_state_store = GatewayStateStore(os.path.join(config["buckets_dir"], "gateway_state.db"))`。

### 1.8 memory_write_candidates.jsonl —— 自动写入候选(写门决策日志)

| 存储 | 表/文件 | 字段 | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| JSONL | `state/memory_write_candidates.jsonl`(可配置文件名) | candidate_id, source, decision, surprise_score, reasons[], repeat_count, max_existing_similarity, max_candidate_similarity, content, created_at | candidate_id/source/decision/content/created_at=str; surprise_score/max_existing_similarity/max_candidate_similarity=float; reasons=list[str]; repeat_count=int | 无(逐行追加,candidate_id 为指纹但无唯一约束) | 无 | 无(独立日志,不关联具体 bucket_id,直到写入成功才产生 bucket) | haven-ombre | 284c9c7b | `/home/user/18358386529/haven-ombre/memory_write_gate.py:110-247`(WriteGateDecision于110-120行;record()于230-247行附近) | 这是清单3中 **CHOICE 事件(是否写入记忆)的既有落点**;无 confidence 字段但有语义等价的 surprise_score |

### 1.9 Supabase(可选云同步目标,`public.memories` 表,PostgreSQL)—— 现有云端结构

**重要限定**:此结构在仓库中以迁移 SQL 脚本和同步脚本形式存在,是 haven-ombre 项目既有的、可选的云同步旁路,不属本审计员新增。是否已在生产 Supabase 项目中实际执行、是否有数据,属清单1(Runtime)/清单6(Resource)范畴,本报告仅记录代码定义的结构本身。

| 存储 | 表/文件 | 字段 | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| PostgreSQL(Supabase) | `public.memories`(通过 `scripts/supabase_memory_rpc.sql` 的 ALTER 语句增量定义,基表本身不在仓库内,推断由 Supabase 控制台预先建表) | id, title, type, domain[], tags[], content, valence, arousal, confidence, importance, period, date, pinned, anchor, resolved, digested, comments, comment_count, source, created, last_active, updated_at, activation_count | confidence=double precision default 0.5; comments=jsonb default '[]'; comment_count=integer; resolved/digested/anchor=boolean default false; updated_at=timestamptz default now() | 未在本文件中显式声明(推测 id 为既有 PK,未见 DDL) | 未见 CREATE INDEX 语句(本文件只含 ALTER TABLE / trigger / RPC 函数) | 通过 RPC 函数 `create_memory(...)` 暴露给客户端 | haven-ombre | 284c9c7b | `/home/user/18358386529/haven-ombre/scripts/supabase_memory_rpc.sql:1-45`;字段清单交叉印证见 `/home/user/18358386529/haven-ombre/scripts/sync_to_supabase.py:29-51`(SYNC_FIELDS/SUPABASE_FIELDS 常量) | **`public.memories.confidence`(double precision, default 0.5)是现成的、已在 SQL 中定义的置信度列**,与 Markdown bucket 的 `confidence` YAML 字段一一对应(见 sync_to_supabase.py 的 SYNC_FIELDS)。脱敏说明:`sync_to_supabase.py:26` 中的 `DEFAULT_SUPABASE_URL = "https://nuhbpesfpoywzcxlqfhs.supabase.co"` 为硬编码的项目地址(非密钥);经检索(`grep -n "apikey\|api_key\|SERVICE_ROLE" scripts/sync_to_supabase.py`)未发现硬编码的 API Key/Service Role Key,密钥经由 `self.service_key` 变量注入(来源为运行参数/环境变量,未见默认值),本报告不输出该变量的运行时取值。 |

---

## 二、ombre-brain(P0luz/Ombre-Brain,参考实现)

### 2.1 Markdown Bucket(与 haven-ombre 同构但字段有差异)

| 存储 | 表/文件 | 字段 | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| Markdown+YAML | `buckets/{permanent,dynamic,feel,plan,letter}/{domain}/{name}_{id}.md` | id, name, tags, domain, valence, arousal, importance, type, created, last_active, updated_at, activation_count, media(可选), meaning(可选), 以及 permanent/feel/plan/letter 特有字段(未逐一穷举) | 同 haven-ombre 基本类型 | 文件名内含 id(应用层唯一,另有 `_bucket_path_index` 内存索引 + `_bucket_turn` 跨进程锁防并发冲突) | 无数据库索引;新增 `_bucket_path_index_guard` 路径索引一致性保护(haven-ombre 未见此机制) | 关系(Relation)不再走独立 JSONL,而是**内嵌于本 bucket 自身的 YAML frontmatter**,通过 `mutate_relation_links`/`mutate_relation_pair` 原子改写两侧镜像字段 | ombre-brain | 6f7335d0 | `/home/user/18358386529/ombre-brain/src/bucket_manager.py:1660-1742`(create,frontmatter.Post 构造于1720行);`src/bucket_manager.py:2293-2342`(mutate_relation_links/mutate_relation_pair) | **未发现 confidence 字段**:对 `bucket_manager.py` 全文 grep `"confidence"` 无命中(证据:`grep -n "\"confidence\"" src/bucket_manager.py` 无输出)。即 ombre-brain 的 Markdown bucket 元数据相比 haven-ombre **缺少 confidence 列**,这是与 haven-ombre 的结构性差异(见"发现的冲突"一节)。 |

### 2.2 Relation 边(嵌入 bucket frontmatter,非独立表)

| 存储 | 表/文件 | 字段 | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| YAML(嵌入 bucket) | 每个 bucket 文件自身的 relation 字段(经 `relation_store.py` 校验/规范化) | relation_type, score, label(可选), id(目标bucket) | relation_type=str(枚举: caused_by/causes/continuation_of/continues/related_to/same_event/custom); score=float(round 4位小数); label=str(≤20字符) | 无独立主键(内嵌于 bucket 文件,以目标 id 隐式去重,`MAX_RELATION_LINKS=64` / `MAX_ACTIVE_RELATION_LINKS=16` 做数量上限) | 无 | 双向镜像(A→B 与 B→A 各自存一份,经 mutate_relation_pair 保证一致) | ombre-brain | 6f7335d0 | `/home/user/18358386529/ombre-brain/src/ombrebrain/storage/relation_store.py:1-30`(枚举与常量),`:61,115,199-203`(score 字段的产生/序列化) | 字段名为 `score`(向量相似度/规则打分),**语义与"置信度confidence"相近但命名及计算依据不同**(score 来自阈值:same_event≥0.85、continuation≥0.75、related_to≥0.72 的相似度,非"该陈述是否可信"的置信度)。未发现独立的 evidence/reason 字段(与 haven-ombre 的 memory_edges.jsonl 的 reason 字段不同)。 |

### 2.3 LedgerMirror(生产已接线的事件镜像,JSONL,Phase 1)

| 存储 | 表/文件 | 字段 | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| JSONL(append-only) | `<buckets_dir>/_ledger/...`(具体文件名由 `ledger_path` 配置传入,`bucket_manager.py:582` 处实例化 `LedgerMirror(ledger_path)`) | seq, schema_version, ledger_role, canonical, event_type, trace_id, trace_kind, body_hash, payload, recorded_at | seq=int(自增,由 latest_seq()+1 计算,非数据库自增,存在小概率并发竞态,代码未见文件锁); schema_version=int(=1); ledger_role=str(固定"mirror"); canonical=bool(固定 False); event_type=str; trace_id/trace_kind=str; body_hash=str("sha256:..."); payload=dict(JSON安全化); recorded_at=ISO8601 str | 无显式主键(seq 为逻辑序号,非数据库约束) | 无索引(纯顺序文件,`iter_events()`全量扫描) | trace_id 对应 bucket_id | ombre-brain | 6f7335d0 | `/home/user/18358386529/ombre-brain/src/ombrebrain/eventsourcing/ledger_mirror.py:14-51`(append_event);docstring 明确"Phase 1 mirror only...not the canonical source of truth yet"(第15-19行) | 字段中**无 confidence**。**代码原文明确自称"非权威数据源",Markdown 桶文件才是权威源**(canonical=False 硬编码)。此表是清单3 CHOICE/ACTION 类"桶生命周期事件"的既有落点(见清单3),但**不是**九类事件语义(OBSERVED..CONTRADICTED)的完整实现。 |

### 2.4 TraceSQLiteProjection(LedgerMirror 的 SQLite 只读投影)

| 存储 | 表/文件 | 字段 | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| SQLite | `<buckets_dir>/_ledger/projections/trace_catalog.sqlite3` 表 `projection_meta` | key, value | TEXT/TEXT | key PK | — | — | ombre-brain | 6f7335d0 | `/home/user/18358386529/ombre-brain/src/ombrebrain/projection/projection_sqlite.py:121-124` | 该库整体通过 `_reset_schema()` **每次重建都 DROP TABLE 再重建**(第115-119行 DROP TABLE IF EXISTS),确认是"影子投影"(代码中 `"projection_role": "shadow"` 字面量,见 `bucket_manager.py:757-767` 的 sqlite_projection 报告结构),非权威存储 |
| SQLite | 同库 表 `traces` | trace_id, trace_kind, state, body_hash, created_seq, latest_seq, latest_event_type, touch_count, resolved, deleted, tombstone, name, importance, metadata_json, search_text | trace_id=TEXT PK; trace_kind/state/body_hash/latest_event_type/name/search_text=TEXT; created_seq/latest_seq/touch_count=INTEGER; resolved/deleted/tombstone=INTEGER(布尔); importance=INTEGER(可空); metadata_json=TEXT | trace_id PK | 可选 FTS5 虚表 `trace_fts`(依 fts_enabled 开关) | trace_id 对应 bucket_id | ombre-brain | 6f7335d0 | `/home/user/18358386529/ombre-brain/src/ombrebrain/projection/projection_sqlite.py:126-152` | 无 confidence 字段 |

### 2.5 MemoryFabric / WAL / MemoryEvent(存在但**未接入生产写路径**的事件溯源架构)

| 存储 | 表/文件 | 字段 | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| 自定义 WAL(追加日志文件) | `<root>/fabric/memory.wal`(root = `buckets_dir/.ombrebrain-v3`,仅当 `LegacyRuntime.from_config()` 被调用时才创建) | 见 MemoryEvent.to_dict():id, actor, actor_name, memory_type, content, visibility, session_id, task_id, source_chain[], parent_event_ids[], cluster_term, cluster_index, created_at, confidence, importance, vector_state, metadata{} | actor=枚举(user/codex/claude/gpt/gemini/mcp_tool/web_dashboard/system); memory_type=枚举(dynamic/permanent/trace/letter/plan/feel); visibility=枚举(private/internal/shared); confidence=float(clamp 0~1,默认1.0); importance=int(clamp 1~10); cluster_term/cluster_index=int(Raft共识术语,推断用于分布式日志复制) | id(由内容确定性哈希生成,`_deterministic_id`) | 无(WAL 顺序结构) | parent_event_ids 构成事件间因果链 | ombre-brain | 6f7335d0 | `/home/user/18358386529/ombre-brain/src/ombrebrain/protocol/schemas.py:46-140`(MemoryEvent dataclass);`/home/user/18358386529/ombre-brain/src/ombrebrain/fabric/storage/engine.py:11-29`(MemoryFabric.open/append_event) | **含 confidence(float,默认1.0)字段**,结构上比 haven-ombre 的 confidence 更完整(还有 actor/visibility/cluster_term等)。**但经代码追踪证实:该整套架构(eventsourcing/、cluster/、distributed/、fabric/、decision/、MemoryEvent)未被生产入口 `server.py` 导入**(证据:`grep -n "eventsourcing\|cluster\.\|fabric\.\|distributed\." server.py server_app.py` 无命中);唯一的调用路径 `build_v3_runtime()`(`app/legacy_wiring.py:26-34`)在全仓非测试代码中**从未被调用**(证据:`grep -rn "build_v3_runtime" --include=*.py .` 仅命中定义处和 `__init__.py` 的导出语句);另一处调用 `LegacyRuntime.from_config(...)` 位于 `web/system.py:187` 的 `_build_isolated_vnext_preflight()`,其**文档字符串明确写明"在隔离目录执行 vNext 诊断,不在用户 vault 创建 WAL 状态"**(`web/system.py:184`,`tempfile.TemporaryDirectory`),即该路径故意不写入真实 buckets_dir。详见清单3"发现的冲突"一节。 |

### 2.6 SourceStore —— 不可变原文证据(内容寻址文件,非数据库)

| 存储 | 表/文件 | 字段 | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| 内容寻址文件 | `<vault>/_sources/src_<sha256(content)>.source` | 文件名即哈希(src_+64位hex);文件体为原文内容(推断,未见结构化字段头) | 文件系统路径 | 文件名唯一(基于内容哈希天然去重) | 无 | 通过 bucket frontmatter 中的 source 引用字段关联(该关联字段本身在本文件未定义,推测在 bucket_manager.py 别处) | ombre-brain | 6f7335d0 | `/home/user/18358386529/ombre-brain/src/ombrebrain/storage/source_store.py:1-30` | 无 confidence 字段;docstring 说明"不参与普通 Markdown 扫描、浮现或语义索引;只有 source_read 在精确匹配桶 ID 与标题后才会读取",即证据文本与检索/浮现路径是隔离的 |

---

## 三、xinchao-nian(tianyupaipai-cmd/xinchao-nian,心智层,Node.js,独立于上述 Python 实现)

**重要限定**:本节仅覆盖 `xinchao-nian/xinchao/src/`(package.json 确认为独立 Node.js 项目,`"name": "xinchao-dynamic-mind"`)。其状态/事件模型与 haven-ombre/ombre-brain 的 Python 实现**完全独立、不共享存储**,不应混为一谈。

| 存储 | 表/文件 | 字段 | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| 单一 JSON 文件(原子写:临时文件+rename) | 由 `StateStore` 管理的状态文件(路径由调用方传入构造函数,本文件未硬编码路径) | 由 `factory()` 定义初始结构;经 `engine.js` 使用可见字段包括:consciousness, fatigue, drives{}, recentDreams[], thoughtPool{flash,obsessions}, sessionOverlays{}, revision, interactionUsage{}, recentConversationEvents[], grudge(可选) | JSON 任意结构(无 schema 校验层,由各调用模块自行读写字段) | 无(单文件=单一状态快照,无行级主键概念) | 无 | — | xinchao-nian | 97f1bdcc | `/home/user/18358386529/xinchao-nian/xinchao/src/state-store.js:1-52`(StateStore类);字段清单交叉印证于 `/home/user/18358386529/xinchao-nian/xinchao/src/engine.js:58-311`(ensureStateShape等) | 无 confidence/置信度概念(该状态是情绪/驱动力数值,非记忆事实,不适用该维度) |
| JSONL(append-only,经 `open(path,'a')` + `fsync`) | TransitionJournal 管理的日志文件(路径由调用方传入) | 两类记录:(a) `recordTransition`产出的 {id, at, type, source, sessionId, eventId, revisionBefore, revisionAfter, delta{}, details{}};(b) `recordContext`产出的 {id, at, type:'context_envelope', source:'context-adapter', sessionId, contextDigest, details:{mode,estimatedTokens,sectionCount,delivered,alreadyDelivered,ombreIncluded}} | 均为 JSON 值;delivered/alreadyDelivered/ombreIncluded=boolean;estimatedTokens/sectionCount=number;contextDigest=str(sha256前16位) | 无(id为UUID但仅去重展示,无唯一约束强制) | 无(读取时按 since/types 过滤,`list()`方法对文件末尾做有限字节窗口扫描,非全文索引) | eventId 字段可与外部会话事件关联(应用层自约定,无外键) | xinchao-nian | 97f1bdcc | `/home/user/18358386529/xinchao-nian/xinchao/src/transition-journal.js:107-226` | **无 confidence 字段**。**已有 delivered/alreadyDelivered 字段,是清单3 DELIVERED 事件在 xinchao 侧的既有落点**。 |
| 单一 JSON 文件(经 `StateStore`) | `cabin-store.js` 管理的"小屋"状态 | schemaVersion(=1), notes[], ledger[] | ledger 内部结构未在本文件完整展开;从 `money()`辅助函数(四舍五入到分)推断 ledger 条目含金额字段,**语义疑似为虚拟经济/花费记账,而非记忆事件账本** | 无 | 无 | — | xinchao-nian | 97f1bdcc | `/home/user/18358386529/xinchao-nian/xinchao/src/cabin-store.js:1-40` | 语义存疑,标记为待人工复核(见"发现的冲突"),不纳入九类事件语义映射 |

---

## 四、kiwi-mem(LucieEveille/kiwi-mem,算法参考,但仓库本身是完整服务;**数据库为 PostgreSQL,非任务线索中假设的 SQLite**)

**重要偏差说明**:任务线索称 `database.py`"可能是 SQLite schema 定义",经实读代码确认:`database.py` 使用 `asyncpg`(异步 PostgreSQL 驱动),DDL 语法为 PostgreSQL 方言(`SERIAL PRIMARY KEY`、`TIMESTAMPTZ`、`JSONB`、`information_schema.columns` 探测式迁移),**不是 SQLite**。已在此如实标注文档/线索与代码的差异。

### 4.1 conversations —— 代码注释明确其角色为"事件账本"

| 存储 | 表/文件 | 字段 | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| PostgreSQL | `conversations` | id, session_id, role, content, model, created_at, project_id, scope_known, usage, turn_id, turn_key, source_message_id | id=SERIAL PK; session_id/role/content/model=TEXT; created_at=TIMESTAMPTZ; project_id=TEXT(可空); scope_known=BOOLEAN NOT NULL DEFAULT FALSE; usage=JSONB; turn_id=BIGINT; turn_key=TEXT; source_message_id=TEXT | id SERIAL PK | idx_conversations_session(session_id, created_at); idx_conversations_source_message(session_id, source_message_id) | turn_id 关联同轮 user/assistant 行;source_message_id 关联 chat_messages.id(代码注释注明"同名不同义,禁止联表推断") | kiwi-mem | b01a0c50 | `/home/user/18358386529/kiwi-mem/database.py:103-111`(基表);`:572-587`(W2-03 事件账本扩展,含中文注释"事件账本 conversations 扩展") | 无 confidence 字段。三态语义(scope_known × project_id)在代码注释(576-582行)与 `docs/event-ledger-scope-and-reconciliation.md` 的描述**一致**(交叉核对见清单3"发现的冲突"一节)。 |

### 4.2 memories —— 核心记忆表(增量演进,v3.0~v6.2b)

| 存储 | 表/文件 | 字段(节选,按引入版本) | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| PostgreSQL | `memories` | id, content, importance, source_session, created_at, last_accessed, embedding, title, memory_type, category_id, source, project_id, is_permanent, lock_source, scene_id, dream_processed_at, emotional_weight, access_count, access_query_hashes, resolution | id=SERIAL PK; content/title=TEXT; importance=INTEGER DEFAULT 5; source_session/source/memory_type/lock_source/project_id=TEXT; created_at/last_accessed/dream_processed_at=TIMESTAMPTZ; embedding=TEXT(JSON字符串,非pgvector原生类型); category_id=INTEGER FK→memory_categories(id); is_permanent=BOOLEAN; scene_id=INTEGER; emotional_weight/access_count=INTEGER DEFAULT 0; access_query_hashes=JSONB DEFAULT '[]'; resolution=FLOAT DEFAULT 1.0 | id SERIAL PK | idx_memories_project(project_id) WHERE project_id IS NOT NULL | category_id→memory_categories;project_id 逻辑关联项目;scene_id 逻辑关联 mem_scenes | kiwi-mem | b01a0c50 | `/home/user/18358386529/kiwi-mem/database.py:114-122`(基表);`:124-155,222-246,479-510,754-763`(各版本 ALTER COLUMN) | **全文检索确认:`memories` 表始终没有 confidence/credibility/可信度字段**(证据:`grep -n "confidence\|belief\|credibility\|trust_score" database.py` 无命中)。最接近的字段是 `resolution`(FLOAT,默认1.0,"记忆软化系统",语义是衰减程度而非可信度)与 `emotional_weight`/`access_count`(热度,非可信度)。**这是清单2要求如实记录的明确缺口:kiwi-mem 现有结构中没有可复用的置信度/可信度字段。** |

### 4.3 memory_edges —— 类型化记忆关系(v5.2)

| 存储 | 表/文件 | 字段 | 类型 | 主键 | 索引 | 关系 | 来源 | 版本 | 证据ID | 缺口 |
|---|---|---|---|---|---|---|---|---|---|---|
| PostgreSQL | `memory_edges` | id, from_id, from_type, to_id, to_type, edge_type, reason, created_at, created_by | id=SERIAL PK; from_id/to_id=INTEGER(无FK约束,代码注释未见外键声明); from_type/to_type=TEXT DEFAULT 'memory'; edge_type=TEXT NOT NULL; reason=TEXT DEFAULT ''; created_by=TEXT DEFAULT 'dream' | id SERIAL PK | 未见专属索引 | from_id/to_id 逻辑关联 memories.id 或其他实体(by from_type/to_type) | kiwi-mem | b01a0c50 | `/home/user/18358386529/kiwi-mem/database.py:713-724` | 无 confidence 字段;有 `reason` 字段(证据/理由文本)。edge_type 已被实际用作 `"supersedes"`(见清单3 CONTRADICTED 落点,`main.py:1314-1320`)。 |

### 4.4 其余核心表(节选,完整字段以源码为准)

| 存储 | 表/文件 | 关键字段(节选) | 来源 | 版本 | 证据ID | 备注/缺口 |
|---|---|---|---|---|---|---|
| PostgreSQL | `chat_messages` | id, conversation_id, role, content, time, token_info(jsonb), status_events(jsonb), tool_events(jsonb), memory_result(jsonb), memory_event(jsonb), dream_event(jsonb), turn_key, versions(jsonb) | kiwi-mem | b01a0c50 | `database.py:292-319` | 面向前端消息快照,代码注释明确 `turn_key` 与 `conversations.turn_key` "同名不同义,禁止跨表 JOIN 或互推"(`database.py:555-556,581-582`) |
| PostgreSQL | `mem_scenes` | id, title, narrative, atomic_facts(jsonb), foresight(jsonb), embedding(jsonb), related_memory_ids(jsonb), status, created_by_dream_id | kiwi-mem | b01a0c50 | `database.py:413-426` | Dream 整合场景,embedding 同样是 JSONB 而非 pgvector 原生类型 |
| PostgreSQL | `dream_logs` | id, started_at, finished_at, status, trigger_type, memories_processed/deleted/merged, scenes_created/updated, foresights_generated, links_created, memories_softened, structured_result(jsonb) | kiwi-mem | b01a0c50 | `database.py:441-477` | Dream 批处理运行记录,是 RESULT 类事件的既有落点候选 |
| PostgreSQL | `session_tombstones` / `turn_tombstones` / `message_tombstones` | (session_id / turn_key / message_id) + deleted_at,均为永久保留、不做GC | kiwi-mem | b01a0c50 | `database.py:610-631` | 删除即永久的墓碑机制,代码注释明确唯一撤销点是 `_restore_conversation_tx` |
| PostgreSQL | `session_source_rev` / `deletion_epoch` | rev/reset_generation 计数器,用于并发下的"素材已变"检测 | kiwi-mem | b01a0c50 | `database.py:634-651` | 乐观并发控制机制,非记忆语义本身 |
| PostgreSQL | `memory_extraction_state` | session_id(PK), last_extracted_message_id, claimed_until, claim_token | kiwi-mem | b01a0c50 | `database.py:591-599` | 每会话提取游标+租约,代码注释"本票只建表、零读写" |

---

## 跨仓库 confidence/置信度字段总览(供总控快速对照,不构成新设计)

| 仓库 | 结构 | 是否存在可复用的置信度/可信度字段 | 字段名 | 证据ID |
|---|---|---|---|---|
| haven-ombre | Markdown bucket YAML | 存在(可选) | `confidence`(float 0~1) | bucket_manager.py:180-181 |
| haven-ombre | memory_moment_edges(SQLite) | 存在(必填) | `confidence`(REAL) | memory_moments.py:224-233 |
| haven-ombre | memory_edges.jsonl | 存在(必填,默认0.5) | `confidence`(float) | memory_edges.py:56-63 |
| haven-ombre | entity_edges.jsonl | 存在(必填,默认0.65) | `confidence`(float) | entity_edges.py:172-192 |
| haven-ombre | Supabase public.memories | 存在(默认0.5) | `confidence`(double precision) | supabase_memory_rpc.sql:16-17 |
| ombre-brain | Markdown bucket YAML | **不存在**(全文grep无命中) | — | (负面证据)`grep "confidence" src/bucket_manager.py`无命中 |
| ombre-brain | relation_store(嵌入bucket) | 不存在confidence,存在语义相近的`score`(相似度打分,非可信度) | `score`(float) | relation_store.py:61,199-203 |
| ombre-brain | MemoryEvent/WAL(未接入生产) | 存在(默认1.0) | `confidence`(float) | protocol/schemas.py:60,74 |
| xinchao-nian | 全部结构 | 不存在(该系统字段是情绪/状态数值,非记忆置信度语义) | — | — |
| kiwi-mem | memories 表 | **不存在**(全文grep无命中) | — | (负面证据)`grep "confidence\|belief\|credibility" database.py`无命中 |
| kiwi-mem | memory_edges 表 | 不存在confidence,存在`reason`(文本理由) | `reason`(TEXT) | database.py:713-724 |

---

## 发现的冲突与待人工裁决项

1. **ombre-brain 的 event-sourcing/ledger 设计(cluster/distributed/fabric/eventsourcing/decision)与 haven-ombre 的 raw_events/moments/edges 设计不是同一层级的对应关系,不能直接判定"兼容"或"矛盾",需人工裁决取舍方向**:
   - haven-ombre 的 `raw_events.sqlite` + `memory_moments.sqlite` + `memory_edges.jsonl`/`entity_edges.jsonl` 是**已接线、被 server.py/bucket_manager.py 实际调用**的具体存储实现,Markdown bucket 文件本身是权威源(canonical),SQLite/JSONL 是派生索引。
   - ombre-brain 同样有一套"已接线"的对应物:`LedgerMirror`(JSONL,`bucket_manager.py:582`直接实例化并在增删改路径调用,`event_type`取值为 `TraceCreated/TraceUpdated/TraceHardDeleted/TraceRestored/TraceDeletedToArchive/TraceTouched/TraceArchived`,即**桶生命周期 CRUD 事件**,不是九类认知事件语义)以及 `TraceSQLiteProjection`(明确自称 `"projection_role": "shadow"` 的影子投影,可整表重建)。这套"简版"ledger 与 haven-ombre 的 raw_events/moments/edges **语义不重叠、不冲突**,只是颗粒度更粗(只记桶级CRUD,不记片段级moment/edge)。
   - ombre-brain 另有一套复杂得多的架构(`MemoryEvent`、`DecisionRecord`、`DecisionLedger`、`EventSourcedMemoryKernel`、`cluster/raft/*`、`distributed/*`、`fabric/storage/engine.py` 的 WAL),其 `MemoryEvent` dataclass 字段(confidence/actor/visibility/cluster_term/cluster_index)与九类事件语义、与"Claim/可信度"概念高度相关、设计也远比 haven-ombre 精细。**但经代码路径追踪确认,这套架构未被生产入口 `server.py` 导入,唯一显式调用点 `build_v3_runtime()` 在非测试代码中从未被调用,另一调用点 `web/system.py:187` 的文档字符串明确说明是隔离临时目录诊断、不写入用户真实 vault。** 因此该架构目前只在单元测试(`tests/test_v3_decision_*.py`、`tests/test_ledger_replay_phase5a.py`、`tests/test_ledger_property_phase5b.py`)与自诊断路径中被验证,是否要作为记忆系统2.0的目标形态、如何与 haven-ombre 现有的 Markdown+SQLite+JSONL 组合关系,**需要人工裁决**,不应默认采信任一方为"最终设计"。
   - 待人工裁决问题:(a) 记忆系统2.0是否要把 ombre-brain 这套未接线的 event-sourcing/Raft 架构落地为真正的写路径?(b) 若落地,如何与 haven-ombre 现有的、已在生产使用的 raw_events/moments/edges/gateway_state 四套 SQLite+JSONL 结构共存或替换?(c) `MemoryEvent.confidence` 与 haven-ombre 现有的多处 `confidence` 字段是否要统一为单一置信度语义?

2. **ombre-brain 相比 haven-ombre,Markdown bucket 主体缺少 `confidence` 字段**:haven-ombre 的 `bucket_manager.py` create() 明确处理 `confidence` 参数并写入 YAML(`bucket_manager.py:180-181`),而 ombre-brain 同名文件的 create() 路径中全文搜索 `"confidence"` 无任何命中。由于 ombre-brain 被定位为"参考实现"、haven-ombre 是"二改后的本体",这一差异究竟是 ombre-brain 尚未合并该特性、还是 haven-ombre 私自新增、需人工确认演进方向。

3. **kiwi-mem 的 `docs/event-ledger-scope-and-reconciliation.md` 与代码基本一致**:文档描述的三态语义(`scope_known=TRUE+project_id NULL`=明确全局、`scope_known=TRUE+project_id非空`=项目对话、`scope_known=FALSE+project_id NULL`=归属未知)与 `database.py:572-587` 的中文代码注释逐字对应;文档提到的 `scripts/ledger_reconcile.py --json` CLI 与 `database.py:1840`的 `reconcile_event_ledger()` 函数签名(session_id, sample_limit)也对应。**未发现文档与代码的实质性不一致**,可视为该文档如实反映代码现状。但需注意:文档标题所称"event ledger"在代码里就是普通的 `conversations` 表(经多次 ALTER TABLE 增量演进而来),并非独立的"事件"表,若总控设计新的通用 Event 概念时,需明确这与 kiwi-mem 术语的差异,避免误用"kiwi-mem 已有 event ledger 表"这类过度引申的结论。

4. **kiwi-mem 的 `database.py` 使用 PostgreSQL(asyncpg),不是任务线索所猜测的 SQLite**:已在4.0节标注,提醒总控更新对该仓库存储形态的认知假设。

5. **xinchao-nian 仓库内嵌了一份 `ombre-brain/` 子目录完整副本**(`/home/user/18358386529/xinchao-nian/ombre-brain/`),与独立克隆的 `ombre-brain` 仓库结构相似(均含 `src/ombrebrain/eventsourcing`、`decision`、`ledger_property.py` 等)。本报告未对二者做逐文件 diff,按 Phase 0 基线文档的既定原则("按事实记录路径与版本文件,不擅自判断是否为违规的第二实例"),仅在此标记为待人工复核项:需确认这是否为 vendored 依赖、是否版本落后/领先于独立仓库、是否会造成"两份 ombre-brain 代码"的维护分裂。

6. **kiwi-mem `memories` 表缺少置信度字段是明确缺口,不是本报告可自行填补的设计空白**:按任务约束,本报告仅如实记录"未发现",不建议新增列或新表。若记忆系统2.0需要跨四个系统统一的 Claim/置信度模型,需在人工设计阶段单独决策,并从 haven-ombre 已有的 `confidence`(bucket YAML / memory_moment_edges / memory_edges.jsonl / entity_edges.jsonl / Supabase memories)、kiwi-mem 的 `resolution`+`reason`、ombre-brain 未接线的 `MemoryEvent.confidence` 中选择复用起点,而非另起炉灶。
