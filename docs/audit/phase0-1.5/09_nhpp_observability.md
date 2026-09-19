# 09 — NHPP 可观测性审计

**审计模式声明：本报告为源码审计（static source audit），非运行时审计。四个目标系统本次均无本地运行实例；所有结论均来自对已只读克隆到本地的源码快照的静态阅读，未启动任何服务、未执行任何写操作。**

## 仓库与锁定 commit

| 本地路径 | 参考仓库 | 要求锁定 SHA | 实测 HEAD | 是否一致 |
|---|---|---|---|---|
| /home/user/18358386529/haven-ombre | Yinglianchun/Haven-Ombre | 284c9c7b0e51a0ba0032c7028f705d72458cb304 | 284c9c7b0e51a0ba0032c7028f705d72458cb304 | 一致 |
| /home/user/18358386529/ombre-brain | P0luz/Ombre-Brain | 6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5 | 6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5 | 一致 |
| /home/user/18358386529/xinchao-nian | tianyupaipai-cmd/xinchao-nian | 97f1bdcc76b748fa516b7c79a0aa10234143795f | 97f1bdcc76b748fa516b7c79a0aa10234143795f | 一致 |
| /home/user/18358386529/kiwi-mem | LucieEveille/kiwi-mem | b01a0c506f4f10f90f30d408c0291f16720f0718 | b01a0c506f4f10f90f30d408c0291f16720f0718 | 一致 |

---

## 0. 全局关键结论（先给结论）

对 `hawkes|nhpp|bayesian|posterior|belief|point.process`（大小写不敏感）在四个仓库全量代码 + 文档中的穷举 grep（命令与结果见下）：

```
$ grep -riE "hawkes|nhpp|bayesian|posterior|belief|point.process" -l <四仓库根目录>
（命中 8 个文件，全部在 ombre-brain 及其在 xinchao-nian 内的镶嵌副本中，且全部是 "belief" 一词，
 无 hawkes/nhpp/bayesian/posterior/point process 任何命中）
```

**分类结论：Bayesian/NHPP 相关实现的真实状态 = 不存在（none）。**

四个仓库中**没有任何一行代码**实现或引用 Hawkes 过程、NHPP（非齐次泊松过程）、贝叶斯推断/后验（posterior）。唯一命中的关键词 "belief" 全部出现在 **禁止性**上下文里——即代码里有专门的红线机制，明确**禁止**出现"信念更新引擎"（belief updater），而不是已经实现了一个信念更新/NHPP 引擎。证据见下。

**这是一个重要的正面发现，也需要谨慎解读**：现状不是"高风险的 P0（已接入生产）"，而是"完全空白（unstarted）"——连测试桩都没有。放在 P0 风险候选一节的原因，不是"已违反红线"，而是"如果 Phase 2+ 文档默认存在某种雏形可以在此基础上扩展，这个假设是错的，必须从零设计"。

### 0.1 "belief" 命中详情（全部是禁止性代码，不是实现）

**证据ID：`/home/user/18358386529/ombre-brain/src/ombrebrain/plugins/contracts.py:19-45`**
```python
_FORBIDDEN_PLUGIN_TYPES = {
    "autonomous_goal", "autonomous_goal_plugin", "personality_engine",
    "personality_engine_plugin", "current_emotion_generator",
    "belief_updater", "belief_updater_plugin",
    "answer_controller", "answer_controller_plugin",
    "user_scoring", "user_scoring_plugin",
}
_FORBIDDEN_COGNITIVE_CAPABILITIES = {
    "issue_commands", "set_current_emotion", "create_autonomous_goal",
    "generate_current_emotion", "personality_engine",
    "belief_updater", "update_belief",
    "answer_controller", "control_answer", "user_scoring", "score_user",
}
```

`PluginAgencyBoundary.evaluate()`（同文件 128-149 行）对声明了 `belief_updater`/`update_belief` 能力的插件返回 `PluginAgencyDecision(allowed=False, reason="forbidden cognitive capability")`——**这是一道主动拒绝"信念更新插件"注册的红线，不是信念更新逻辑本身**。测试 `test_plugin_agency_boundary_phase11.py:72` 佐证该红线被测试覆盖（证据ID：`/home/user/18358386529/ombre-brain/tests/test_plugin_agency_boundary_phase11.py:72`）。

**证据ID：`/home/user/18358386529/ombre-brain/src/ombrebrain/app/tool_output_contract.py:16-17,83,195-196`**
```python
_CHECKED_CONTRACTS = (
    ..., "tool_output_is_not_belief_engine", ...
)
...
may_be_belief_engine: bool = False
...
if boundary.may_be_belief_engine:
    violations.append(_violation("tool_output_becomes_belief_engine", normalized))
```

`dream` 工具的输出人性化提示文案直接写着 "This is a sediment, not a belief engine."（证据ID：同文件 35 行；对应测试 `test_tool_output_contract_phase8d.py:30-34`）。这再次确认：现有代码把"信念引擎"当作**明确要防止出现**的东西，而不是已实现的能力。

`docs/INTERNALS.md:784,799,1292,1294` 中的中文说明与上述代码完全一致（禁止 `belief_updater` 类插件/能力）——文档与代码在这一点上互相印证，不是"文档单方面声称"。

**结论**：从工程治理角度看，这其实是好消息——说明设计者已经预见到"Belief 引擎"这类能力有滥用风险，提前在插件系统层面设了红线。但这不能解读为"已经有 NHPP/Bayesian 雏形"；恰恰相反，是"连雏形都主动挡在门外"。

---

## 1. 已存在、可作为未来 NHPP 输入的字段/事件类型

### 1.1 ombre-brain：MemoryEvent（秒级精度）

**证据ID：`/home/user/18358386529/ombre-brain/src/ombrebrain/protocol/schemas.py:45-63`**（见 07 号报告 3.1 节完整代码）

可用作 NHPP 输入的字段：
- `created_at: str`（ISO8601 字符串，见下方精度分析）
- `actor` / `actor_name`（事件触发者，NHPP 里可作为"标记点过程"的 mark）
- `memory_type`（DYNAMIC/PERMANENT/TRACE/LETTER/PLAN/FEEL，事件类型枚举）
- `source_chain`（触发上下文的调用链，例如 `("legacy_bucket_manager", "create")`）
- `metadata`（自由字典，当前承载了策略裁决、投影审计等旁路信息，见 08 号报告）

事件写入路径：`fabric.append_event(event)`（`legacy_runtime.py:150,208,290`），底层是 `MemoryFabric`（`src/ombrebrain/fabric/storage/engine.py`），配合 WAL（`fabric/log/wal.py`）——**这是一个真实的、只追加的事件日志基础设施**，具备成为时间序列输入源的物理形态。

**关键缺口——时间戳精度**：

**证据ID：`/home/user/18358386529/ombre-brain/src/utils.py:1142-1147`**
```python
def now_iso() -> str:
    """Return current time as ISO format string."""
    return datetime.now().isoformat(timespec="seconds")
```

`timespec="seconds"` 意味着 ombre-brain 生态里通过 `now_iso()` 产生的时间戳**只精确到秒**。NHPP/Hawkes 过程建模高频事件的到达间隔（inter-arrival time）时，同一秒内发生的多个事件会被压缩成同一时间戳，丢失事件顺序和真实间隔——这是一个**必须修复才能支撑 NHPP 的硬性缺口**，而不是可以忽略的细节。unknown：`MemoryEvent.created_at` 字段本身是自由字符串，不强制调用 `now_iso()`；本次未逐一核查所有调用点是否 100% 都经过这个秒级函数，但 `now_iso` 是 `utils.py` 里唯一发现的时间戳生成函数，且被 `merge_or_create`（`tools/_common.py:1406`）等多处复用，可以合理推断这是仓库内的事实标准精度。

### 1.2 xinchao-nian：TransitionJournal（毫秒级精度，四仓库中最佳）

**证据ID：`/home/user/18358386529/xinchao-nian/xinchao/src/transition-journal.js:107-141`**
```js
export class TransitionJournal {
  recordTransition({ before = {}, after = {}, type, source = 'system',
                     sessionId = '', eventId = '', details = {}, force = false,
                     at = new Date() }) {
    const delta = summarizeTransition(before, after);
    ...
    return this.append({
      id: randomUUID(),
      at: new Date(at).toISOString(),   // JS Date.toISOString() → 毫秒精度
      type: compactString(type || 'state_transition'),
      source: compactString(source || 'system'),
      sessionId: compactString(sessionId, 120),
      eventId: compactString(eventId, 120),
      revisionBefore, revisionAfter, delta,
      details: safeDetails(details),
    });
  }
```

`append()`（同文件 175-190 行）把记录以 JSONL 追加写入磁盘文件（`handle.appendFile(...)`, `handle.sync()`），并提供 `list({limit, since, types, maxBytes})` 读取接口（192-225 行）。这是四个仓库中**唯一**发现的、原生带毫秒级时间戳（JS `Date.toISOString()` 默认输出 `YYYY-MM-DDTHH:mm:ss.sssZ`）+ 显式 `type` 字段 + 显式 `source`（触发上下文）+ 追加写入 + 支持按类型/时间过滤读取的事件流，形态上最接近 NHPP 建模需要的"精确时间戳 + 事件类型 + 触发上下文"三元组。

可用字段：`at`（毫秒 ISO 时间戳）、`type`（事件类型，如 `state_transition`/`context_envelope`）、`source`（触发来源）、`sessionId`/`eventId`（可用于按会话分组做条件强度估计）、`delta`（状态变化量，可作为 mark）。

**缺口**：`TransitionJournal.recordTransition` 内有主动去重/节流逻辑——`if (!force && revisionBefore === revisionAfter && !Object.keys(delta).length) return Promise.resolve(null)`（同文件 128 行），即**没有变化的"心跳"事件会被主动丢弃、不写入日志**。这对 NHPP 是一把双刃剑：一方面减少了噪声，另一方面如果 NHPP 需要"无事件发生"本身的时间信息（例如估计基础强度 λ0），这套日志目前不提供"确认系统在运行但无事件"的负样本时间点。

### 1.3 kiwi-mem：数据库时间戳（`TIMESTAMPTZ`，理论精度取决于 Postgres，微秒级）

**证据ID：`/home/user/18358386529/kiwi-mem/database.py:367,373-374`**（reminders 表 `trigger_time TIMESTAMPTZ`、`created_at/updated_at TIMESTAMPTZ DEFAULT NOW()`）。PostgreSQL 的 `TIMESTAMPTZ` 类型本身支持微秒精度，是四个仓库底层存储中精度上限最高的，但**应用层读出后统一转成 `.isoformat()`**（`database.py:5174-5176` 等多处 `if isinstance(d[k], _dt): d[k] = d[k].isoformat()`），Python `datetime.isoformat()` 默认保留微秒（如果原值非零），所以理论上这条链路精度是够的；但 kiwi-mem 的这套时间戳是**业务状态时间戳**（提醒的触发/创建/更新时间），不是"事件发生"的日志时间戳——它不构成一个可供 NHPP 使用的**事件流**，只是几张状态表的字段，没有 `type`/`source` 这类"事件分类"元数据，也没有追加写入的历史轨迹（`UPDATE` 会覆盖旧值）。

### 1.4 haven-ombre：memory_edges.py（秒级精度，同 ombre-brain）

`created_at or datetime.now(timezone.utc).isoformat(timespec="seconds")`（证据ID：`/home/user/18358386529/haven-ombre/memory_edges.py:62`）——与 ombre-brain 同样的秒级精度问题。

---

## 2. 必要字段缺失清单

1. **毫秒/微秒级统一时间戳**：ombre-brain 和 haven-ombre 目前统一走秒级 `now_iso()`/`isoformat(timespec="seconds")`；只有 xinchao-nian 的 `TransitionJournal` 天然是毫秒级。四个系统之间没有统一的时间戳精度标准，跨系统事件对齐会因精度不一致产生虚假的"同时发生"或错误的先后顺序判断。
2. **可靠的、不做主动去重/节流的原始事件流**：xinchao-nian 的 `TransitionJournal` 主动丢弃"无变化"事件（1.2 节），kiwi-mem 的状态表是覆盖式更新、没有历史事件流，ombre-brain 的 `MemoryFabric.append_event` 虽然是追加写入，但写入频率和触发条件由各工具自行决定（`hold`/`breath`/`trace` 各自在函数末尾调用 `record_v3_tool_event`，且如 08 号报告 3.3 节所证，该记录路径在主服务里目前是 no-op），**不构成一个可靠、完整、连续的事件到达时间序列**。
3. **事件间的因果/触发关系标注**：NHPP/Hawkes 建模通常需要知道"事件 B 是否由事件 A 触发"（self-exciting 假设的基础）。ombre-brain 的 `parent_event_ids` 字段（`protocol/schemas.py:56`）在结构上可以承载这个语义，但如 07 号报告 3.1 节所述，本次审计未能确认其在生产路径中被系统性填充（多数 `MemoryEvent.new()` 调用点未见显式传参，默认是空 tuple）。
4. **跨系统统一的事件类型分类法（taxonomy）**：ombre-brain 用 `MemoryType` 枚举（6 类），xinchao-nian 用自由字符串 `type`（`state_transition`/`context_envelope` 等，未见枚举约束），kiwi-mem 没有事件类型概念（只有表名区分）。三者互不兼容，无法直接合并成一个统一的多类型点过程输入。
5. **心跳/无事件区间的显式记录**：见 1.2 节，NHPP 参数估计通常需要知道"观察窗口内确实没有事件发生"，而不仅仅是"发生了什么"；当前所有事件源都是"有事才记"，没有发现任何等间隔心跳或显式空区间标记机制。unknown：xinchao-nian 是否存在心跳机制记录于其他文件（如 `heartbeat-store.js`，本次仅确认文件存在于 `xinchao/src/heartbeat-store.js`，未展开阅读其内容），标记为待后续深挖项。

---

## 3. Bayesian/NHPP 生产逻辑接入状态的最终分类

按任务要求的四分类（不存在 / 仅测试桩 / 已有生产代码但未接入主流程 / 已接入主流程）给出结论：

**分类结果：不存在（none）。**

依据：
- 全仓库 grep `hawkes|nhpp|bayesian|posterior|point.process`（大小写不敏感）：**零命中**（四个仓库均无）。
- 全仓库 grep `belief`（大小写不敏感）：8 处命中，全部指向**禁止性护栏代码**（`_FORBIDDEN_PLUGIN_TYPES`/`_FORBIDDEN_COGNITIVE_CAPABILITIES`/`may_be_belief_engine` 违规检测/"not a belief engine" 提示文案）和对应文档、测试，**没有一处是信念更新/贝叶斯推断/点过程强度估计的实现代码**，甚至没有函数签名占位符或 `# TODO: implement NHPP` 类注释（本次 grep 未见任何 TODO/FIXME 提及这些概念）。

因此，本报告在"是否已违反硬性约束 5（禁止 Bayesian/NHPP 控制 Injection/Action）"这一问题上的结论是：**当前代码没有违反**，因为相关实现根本不存在，无从谈起"控制"。

---

## P0级风险候选（本报告范围内）

- **无 P0**（没有发现 Bayesian/NHPP 已接入主流程、甚至没有发现任何实现代码）。
- **风险提示（非 P0，但需要在 Phase 2+ 设计阶段处理）**：
  1. ombre-brain/haven-ombre 的默认时间戳精度是**秒级**（`timespec="seconds"`），这是支撑任何点过程建模之前必须解决的前置工程工作，否则同一秒内的并发事件会在时间序列分析中被错误合并。
  2. 现有的插件红线（`_FORBIDDEN_PLUGIN_TYPES`/`_FORBIDDEN_COGNITIVE_CAPABILITIES`）目前只挡住了**插件系统**这一条注册路径；本次审计未检查是否存在绕开插件系统、直接修改核心代码引入 belief/NHPP 逻辑的可能性（这类风险无法通过静态审计穷尽排除，只能确认"目前没有"，无法确认"永远不会有人绕过"）。建议后续在设计评审 checklist 中把"任何新增的置信度/信念/强度估计代码是否绕开了插件红线直接进核心路径"作为强制审查项。
