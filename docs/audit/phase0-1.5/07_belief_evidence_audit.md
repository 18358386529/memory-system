# 07 — Belief / Evidence 可承载性审计

**审计模式声明：本报告为源码审计（static source audit），非运行时审计。四个目标系统本次均无本地运行实例；所有结论均来自对已只读克隆到本地的源码快照的静态阅读（Read/Grep/Glob/只读 Bash），未启动任何服务、未执行任何写操作。**

## 仓库与锁定 commit

| 本地路径 | 参考仓库 | 要求锁定 SHA | 实测 `git rev-parse HEAD` | 是否一致 |
|---|---|---|---|---|
| /home/user/18358386529/haven-ombre | Yinglianchun/Haven-Ombre | 284c9c7b0e51a0ba0032c7028f705d72458cb304 | 284c9c7b0e51a0ba0032c7028f705d72458cb304 | 一致 |
| /home/user/18358386529/ombre-brain | P0luz/Ombre-Brain | 6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5 | 6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5 | 一致 |
| /home/user/18358386529/xinchao-nian | tianyupaipai-cmd/xinchao-nian | 97f1bdcc76b748fa516b7c79a0aa10234143795f | 97f1bdcc76b748fa516b7c79a0aa10234143795f | 一致 |
| /home/user/18358386529/kiwi-mem | LucieEveille/kiwi-mem | b01a0c506f4f10f90f30d408c0291f16720f0718 | b01a0c506f4f10f90f30d408c0291f16720f0718 | 一致 |

四个仓库 HEAD 均与任务书给定 SHA 一致，以下证据均基于这些确切 commit。

---

## 1. 结论摘要

现有代码里**没有**任何名为 `Claim` / `Belief` / `Evidence` 的一等公民数据结构。但存在三类可复用的相邻结构，分别位于三个不同仓库、彼此不连通：

1. **haven-ombre 的记忆关系图（`memory_edges.py`）**——目前最接近"证据链/矛盾保留"的代码。
2. **ombre-brain 的 `MemoryEvent`（`protocol/schemas.py`）**——带 `confidence` 字段的事件模型，以及"决策记录/形式化不变量"两套可复用的一致性检查基础设施。
3. **kiwi-mem 的矛盾检测 + 软失效（`database.py`）**——已投产的启发式矛盾检测与"不删除只标记失效"逻辑。

以下逐项给出证据。

---

## 2. haven-ombre：memory_edges.py — 现有最接近 Belief/Evidence 图的结构

**证据ID：`/home/user/18358386529/haven-ombre/memory_edges.py:7-25`**

```python
RELATION_TYPES = {
    "triggers", "causes", "precedes", "context_of", "same_event", "updates",
    "next_context", "previous_context", "reflects_on", "contradicts",
    "supports", "promises", "blocks", "belongs_to", "emotional_echo",
    "evidenced_by", "relates_to",
}
```

`MemoryEdgeStore.add_edge`（同文件 39-75 行）的签名与落盘结构：

**证据ID：`/home/user/18358386529/haven-ombre/memory_edges.py:39-75`**
```python
def add_edge(self, source, target, relation_type, confidence: float = 0.5,
             reason: str = "", created_at: str | None = None) -> dict | None:
    ...
    edge = {
        "source": source, "target": target, "relation_type": relation_type,
        "confidence": self._clamp(confidence),
        "reason": str(reason or "").strip()[:240],
        "created_at": created_at or datetime.now(timezone.utc).isoformat(timespec="seconds"),
    }
    ...
    for index, existing in enumerate(edges):
        if self._same_edge(existing, edge):
            if float(existing.get("confidence", 0.0)) <= edge["confidence"]:
                edges[index] = edge   # 同一对 (source,target,relation_type) 只保留置信度更高的一条
            replaced = True
```

**可复用字段**：`relation_type ∈ {contradicts, supports, evidenced_by, ...}`、`confidence`（0-1，clamp）、`reason`（自由文本，240 字符上限）、`created_at`（秒级 ISO 字符串）、`source`/`target`（桶 ID，即 claim/memory 的指针）。这已经是一个"两节点 + 类型化关系 + 置信度 + 理由"的三元组图，语义上可以直接映射到 Claim-Evidence-Contradiction 图的边。

**缺口**：
- 边存储是**单条覆盖**式（同一对节点+关系类型只保留最高置信度的一条边，见上面 `if existing.confidence <= edge.confidence: replace`），**不保留历史置信度变化轨迹**——不是真正的证据链（evidence trail），而是"当前最优信念快照"。
- 没有把"谁产生了这条边"（actor/来源工具）与"基于什么原始文本片段"（source_ranges/quote）关联起来；`reason` 只是自由文本摘要，不是结构化 provenance。
- `RELATION_TYPES` 是全局常量集合，没有版本化/可扩展机制；新增关系类型需要改代码，不是数据驱动。

**佐证**：同一仓库 `config.example.yaml:550` 中存在 `contradicts: 0.45` 的扩散权重配置（证据ID：`/home/user/18358386529/haven-ombre/config.example.yaml:550`），以及 `memory_diffusion.py:30,54,80` 中把 `contradicts`/`blocks`/`conflict` 列为 `CAUTION_RELATION_TYPES`（证据ID：`/home/user/18358386529/haven-ombre/memory_diffusion.py:30,54,80`），说明这套关系类型确实被下游的记忆扩散/检索排序逻辑读取和使用，不是摆设的常量表。

`reflection_engine.py` 中也印证了同一套关系类型枚举被文档化为可选值：

**证据ID：`/home/user/18358386529/haven-ombre/reflection_engine.py:95`**
```
relation_type 只能用 triggers / causes / precedes / context_of / same_event / updates /
next_context / previous_context / reflects_on / evidenced_by / contradicts / supports /
promises / blocks / belongs_to / emotional_echo / relates_to。
```

---

## 3. ombre-brain：MemoryEvent.confidence + 决策/不变量基础设施

### 3.1 MemoryEvent 的 confidence 字段

**证据ID：`/home/user/18358386529/ombre-brain/src/ombrebrain/protocol/schemas.py:45-76`**
```python
@dataclass(frozen=True)
class MemoryEvent:
    id: str
    actor: ActorKind
    ...
    source_chain: tuple[str, ...] = field(default_factory=tuple)
    parent_event_ids: tuple[str, ...] = field(default_factory=tuple)
    ...
    confidence: float = 1.0
    ...
    def __post_init__(self) -> None:
        ...
        object.__setattr__(self, "confidence", _clamp_float(self.confidence, 0.0, 1.0))
```

`source_chain`（字符串元组，见同文件 55 行）和 `parent_event_ids`（字符串元组，56 行）是可以承载 provenance / evidence-link 语义的字段：`source_chain` 目前被用作"这条事件源自哪个模块/哪个动作"的调用链（例如 `legacy_runtime.py:142` 里 `source_chain=("legacy_bucket_manager", str(action))`），`parent_event_ids` 语义上可以表示"这条事件由哪些事件推导而来"。

**缺口**：
- `confidence` 目前**没有任何写入路径把它设为非默认值 1.0** ——本次审计在 `ombre-brain/src` 和 `ombre-brain/tests` 中搜索 `MemoryEvent.new(` 与 `confidence=` 的组合调用，legacy_runtime.py 内所有 `MemoryEvent.new(...)` 调用（`record_bucket_event`/`record_tool_event`/`record_execution_event`，`legacy_runtime.py:136-150,189-208,259-290`）均未显式传 `confidence`，全部落到默认值 `1.0`。即"置信度"字段存在但目前是死字段，未被任何生产路径实际赋予有意义的值。unknown：是否有其他调用点显式赋值，未逐一穷举全部 `MemoryEvent.new(` 调用点（仓库内该字符串出现次数较多），建议后续子代理用 `grep -rn "confidence=" ombre-brain/src` 补全穷举。
- `MemoryEvent` 没有 `contradicts` / `supports` 这类字段或子结构，也没有指向"这条记忆声称什么"的结构化 claim 表达——它是一条通用事件日志条目，不是一个 Claim 对象。
- `parent_event_ids` 在测试 `test_v3_collab_graph.py:46`（`test_collaboration_graph_expands_source_chain_with_parent_provenance`）中被使用（证据ID：`/home/user/18358386529/ombre-brain/tests/test_v3_collab_graph.py:46`），说明存在"协作图谱扩展 source_chain + parent provenance"的测试意图，但这属于 `collab/graph.py` 的图谱构建能力，unknown：本次未展开阅读 `collab/graph.py` 全文以判断其与 Belief/Evidence 的耦合程度，标记为待后续深挖项。

### 3.2 "决策/裁决层" 与形式化不变量框架（可复用基础设施，非 Belief 层本身）

任务书提到的 `test_v3_decision_record.py`、`test_v3_decision_replay.py`、`test_v3_decision_debug_service.py`、`test_v3_consensus.py`、`test_v3_policy_engine.py`、`test_v3_policy_vm.py`、`test_v3_policy_contracts.py` 均能在 `src/ombrebrain/` 下找到对应实现：

- `src/ombrebrain/decision/records.py` — `DecisionRecord`（证据ID：`/home/user/18358386529/ombre-brain/src/ombrebrain/decision/records.py:9-136`），字段包括 `command_plan`、`policy_verdict`、`projection_journal`、`projection_observations`、`consistency_report`、`outcome`、`summary`，并用 SHA256 生成确定性 `id`。
- `src/ombrebrain/decision/ledger.py` — `DecisionLedger.record(...)`，把上述五类元数据拼成一条 `DecisionRecord`。
- `src/ombrebrain/decision/debug.py` / `decision/replay.py` — `DecisionDebugService`（列出/查询/重放决策记录，证据ID：`/home/user/18358386529/ombre-brain/src/ombrebrain/decision/debug.py:1-140`）。
- `src/ombrebrain/policy/engine.py`、`policy/vm.py`、`policy/contracts.py` — `PolicyEngine`（证据ID：`/home/user/18358386529/ombre-brain/src/ombrebrain/policy/engine.py:12-51`）。
- `src/ombrebrain/cluster/consensus.py` — 对应 `test_v3_consensus.py`（unknown：本次未展开阅读该文件全文，仅确认文件存在，证据ID：`/home/user/18358386529/ombre-brain/src/ombrebrain/cluster/consensus.py` 路径存在）。

**这是不是"Belief/Judgment 层"？结论：不是，是决策留痕基础设施，且目前只是旁路记录（详见 08/12 号报告的交叉证据）。** `DecisionRecord` 记录的是"某次工具调用/命令的策略裁决与结果"，不是"某个陈述的可信程度"；它没有 claim/evidence/contradiction 概念，字段全部围绕 command/policy/projection/outcome。它可以被复用为 Belief 层未来"决策留痕"的骨架（确定性 ID 生成、不可变 dataclass、`to_dict`/`from_dict` 往返），但当前不承载任何信念内容。

**形式化不变量框架**（任务线索中提到的 `test_formal_invariants_phase8a.py`、`test_formal_invariants_phase10.py`、`test_v3_memory_invariants.py`）对应实现在：

**证据ID：`/home/user/18358386529/ombre-brain/src/ombrebrain/policy/formal_invariants.py:9-23`**
```python
_SUPPORTED_INVARIANTS = (
    "no_silent_erasure", "indexes_cannot_change_truth",
    "similarity_cannot_bypass_policy", "memory_context_is_not_instruction",
    "past_affect_is_not_current_feeling", "normal_api_cannot_total_recall",
    "compression_declares_loss", "admin_erasure_is_not_forgetting",
    "breath_is_not_total_recall", "trace_reconsolidation_preserves_original",
    "dream_may_sediment_not_decide", "pulse_is_not_current_feeling",
    "self_description_has_no_instructional_force",
)
```

这套 `FormalInvariantChecker` / `InvariantReport` / `InvariantViolation`（`tool_output_contract.py:9` 导入自 `policy.formal_invariants`）是一个通用的"给定事件序列，检查是否违反某条不变量，返回违规列表"的框架。它目前检查的都是"记忆系统边界"类不变量（不删除、不伪装当前情绪等），**不包含任何 Claim/Evidence 一致性不变量**，但其 `InvariantReport(ok, checked, violations, projection_name, projection_role="shadow", canonical=False)` 的结构（`tool_output_contract.py:204-211`）本身就是"影子/非权威投影"模式（`projection_role="shadow"`, `canonical=False` 硬编码），值得作为 Belief/Evidence 一致性检查未来复用的骨架——**但这是"值得复用的基础设施"，不是"缺口"，也不是"已实现的 Belief 层"，两者不能混淆**。

---

## 4. kiwi-mem：矛盾检测 + 软失效（已投产的启发式逻辑）

**证据ID：`/home/user/18358386529/kiwi-mem/database.py:6226-6296`**（`detect_contradictions` 函数）

判定逻辑：标题字符重叠度 > `title_threshold`（默认 0.7）且内容相似度落在 `[0.4, 0.85]` 区间（>0.85 视为重复而非矛盾，由去重逻辑另行处理）。函数内部显式保护：

```python
# v5.3 保护：手动录入的记忆置信度最高，不被自动矛盾检测标失效
if mem.get("source") == "user_explicit":
    continue
if mem.get("is_permanent"):
    continue
```

矛盾发生后的处理方式（**"矛盾保留"而非覆盖删除**）：

**证据ID：`/home/user/18358386529/kiwi-mem/database.py:6148-6170`**
```python
async def invalidate_memory(memory_id: int, reason: str = ""):
    """标记记忆失效（设置 valid_until = NOW()）
    不删除数据，仅标记——搜索时会自动过滤，Dream 查历史仍可见"""
    ...
    result = await conn.execute(
        "UPDATE memories SET valid_until = NOW() WHERE id = $1 AND (valid_until IS NULL OR valid_until > NOW())",
        memory_id,
    )
```

调用链：`main.py:1250-1322`（`detect_contradiction=True` 参数、`contradicted_ids` 收集、逐条调用使旧记忆失效并可能 `create_memory_edge` 建立关系）。

**评价**：这是四个仓库中**唯一一条已投产、被真实业务路径调用**的"矛盾检测 → 软失效（保留证据，仅打时间戳软删除）"逻辑。它满足"矛盾保留"的字面要求（旧记忆不物理删除，`valid_until` 之后仍可被 Dream/历史查询读到），但：
- 判定算法是纯启发式（标题字符重叠 + 向量/字符相似度双阈值），**没有独立的置信度模型**，也没有把"为什么判定矛盾"的推理过程结构化保留（只有 print 日志，不落库为可查询记录）——`kiwi-mem/KNOWN_ISSUES.md:28` 也自述该检测"漏纯数字/单字事实更新"（证据ID：`/home/user/18358386529/kiwi-mem/KNOWN_ISSUES.md:28`）。
- 该逻辑与 haven-ombre 的 `memory_edges.py` 图结构完全独立（两个不同仓库、不同存储：kiwi-mem 用 PostgreSQL `memories.valid_until` 字段，haven-ombre 用 JSONL 边表），互不复用，也没有共享的 Claim/Evidence schema。

---

## 5. 明确的缺口清单（不建议新建表，仅如实列出"没有"的部分）

1. **没有**独立的 `Claim` 数据结构（陈述本身、其真值状态、断言者、断言时间）——四个仓库都没有。
2. **没有**独立的 `Evidence` 数据结构（区分"支持性材料"与"矛盾性材料"、可追溯到原始文本片段）。haven-ombre 的 `evidenced_by`/`supports`/`contradicts` 只是关系图上的边类型标签，不是独立的 Evidence 实体。
3. **没有**跨仓库统一的 confidence 语义——ombre-brain 的 `MemoryEvent.confidence` 是死字段（默认 1.0 且未见写入点显式赋值，见 3.1），haven-ombre 的 edge confidence 有实际使用但只保留"当前最高值"，两者互不相通，kiwi-mem 的矛盾检测甚至没有显式 confidence 数值输出（只有阈值判断的布尔结果）。
4. **没有**证据链的历史版本追踪（同一 claim 随时间置信度如何变化、谁在何时修正过判断）——haven-ombre 的边覆盖策略（"只保留更高置信度的一条"）主动丢弃了历史。
5. **没有**跨系统（haven-ombre ↔ ombre-brain ↔ xinchao-nian ↔ kiwi-mem）共享的 Belief/Evidence schema 或桥接层；四者是独立演化的代码库，字段命名和语义均不统一（例如 haven-ombre 用 `relation_type`/`confidence`/`reason`，ombre-brain 用 `source_chain`/`parent_event_ids`/`confidence`，kiwi-mem 用 `valid_until`/`similarity`）。
6. xinchao-nian（心智层）**没有**发现任何 Claim/Belief/Evidence 相关代码；其状态模型（`xinchao/src/state-store.js`、`thought-pool.js`、`self-signals.js`）是情绪/驱动力/念头池状态机，unknown：本次未逐一确认这些文件内是否有 confidence 类字段，因任务时间限制仅重点核实了 `transition-journal.js`（见 09 号报告），标记为待后续深挖项。

---

## P0级风险候选（本报告范围内）

- 无。本报告未发现 Bayesian/NHPP 直接控制 Belief 写入路径的证据（NHPP/Bayesian 相关的 P0 结论详见 `09_nhpp_observability.md`）。
- 需要注意但非 P0：ombre-brain 的 `MemoryEvent.confidence` 字段已定义但目前是死字段（默认值未被覆盖），如果未来 Phase 2+ 直接假设"现有 confidence 数据可用于训练/校准"，会因为数据实际上全是常量 1.0 而得出错误结论——这是一个数据质量陷阱，建议在设计阶段显式核查。
