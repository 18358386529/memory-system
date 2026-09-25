# 12 — Shadow Mode 接口可行性审计

**审计模式声明：本报告为源码审计（static source audit），非运行时审计。四个目标系统本次均无本地运行实例；所有结论均来自对已只读克隆到本地的源码快照的静态阅读，未启动任何服务、未执行任何写操作。本报告不设计具体实现代码，不提出新字段/新表，仅做"现状 vs 目标草案"的可行性差距分析。**

## 仓库与锁定 commit

| 本地路径 | 参考仓库 | 要求锁定 SHA | 实测 HEAD | 是否一致 |
|---|---|---|---|---|
| /home/user/18358386529/haven-ombre | Yinglianchun/Haven-Ombre | 284c9c7b0e51a0ba0032c7028f705d72458cb304 | 284c9c7b0e51a0ba0032c7028f705d72458cb304 | 一致 |
| /home/user/18358386529/ombre-brain | P0luz/Ombre-Brain | 6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5 | 6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5 | 一致 |
| /home/user/18358386529/xinchao-nian | tianyupaipai-cmd/xinchao-nian | 97f1bdcc76b748fa516b7c79a0aa10234143795f | 97f1bdcc76b748fa516b7c79a0aa10234143795f | 一致 |
| /home/user/18358386529/kiwi-mem | LucieEveille/kiwi-mem | b01a0c506f4f10f90f30d408c0291f16720f0718 | b01a0c506f4f10f90f30d408c0291f16720f0718 | 一致 |

**注意**：`xinchao-nian` 的 `bridge/` 目录在 `.gitmodules` 中声明为指向外部仓库 `tianyupaipai-cmd/xinchao-runtime-bridge` 的 git submodule（证据ID：`/home/user/18358386529/xinchao-nian/.gitmodules`），**未被本次任务克隆/锁定，不在审计范围内**。凡涉及 "xinchao runtime bridge" 内部实现的问题，本报告一律标 unknown。另外 `xinchao-nian/ombre-brain/` 是一份**未用 submodule 机制、直接以普通文件提交进 xinchao-nian 仓库的 ombre-brain 快照副本**（该目录下没有独立 `.git`，证据：`find` 未在 `xinchao-nian/ombre-brain` 下发现 `.git`），与本次单独克隆锁定的 `/home/user/18358386529/ombre-brain`（6f7335d0...）不保证是同一版本；本报告中涉及 ombre-brain 代码的证据均以独立克隆的 `/home/user/18358386529/ombre-brain` 为准。

---

## 0. 先说全局可行性结论

审计执行包第 10 节的 Shadow Mode 草案要求六类输入（Recall/Injection/Action/Xinchao状态只读/Raw Events/现有Memory结构）能被"旁路读取而不侵入主流程"。**现状是：六类输入里有五类能在现有代码中找到具体的、语义匹配的候选挂载点（函数/类），且这些挂载点在结构上是"返回值明确、可在不修改函数体的前提下在调用点或返回处另行旁路记录"的形态；但没有任何一个挂载点是"当前就已经被同一套机制统一旁路记录"的——四个系统是四个独立进程/代码库（Python × 3 + Node.js × 1），彼此没有共享的事件总线或统一的 side-channel，Shadow Mode 若要落地，必须在四处分别打点，是一个真实的集成工作量，不是"接上一根线"就能完成的。**

---

## 1. 六类输入的挂载点候选

### 1.1 Recall 结果

**候选 A（ombre-brain）：`RetrievalEngine.trace(plan)` / `QueryPlanner.plan(...)`**

**证据ID：`/home/user/18358386529/ombre-brain/src/ombrebrain/app/legacy_runtime.py:500-515`**
```python
def _retrieval_metadata(self, operation: str, payload: dict[str, object]) -> dict[str, object]:
    if str(operation) not in {"breath", "search"}:
        return {}
    try:
        plan = self.query_planner.plan(payload, operation=str(operation))
        return {
            "retrieval_plan": plan.to_dict(),
            "retrieval_trace": self.retrieval_engine.trace(plan),
        }
    except Exception as exc:
        ...
```
这是目前唯一一处已经把"检索计划 + 检索轨迹"结构化产出为 dict 的代码（`plan.to_dict()` / `retrieval_engine.trace(plan)`）。但如 08 号报告 3.3 节所证，这条路径挂在 `LegacyRuntime._retrieval_metadata`，只在 `record_execution_event`/`record_tool_event` 被调用时触发，而生产主服务里 `v3_runtime` 为 `None`，这条代码目前**不会被执行**。它是一个"语义正确但当前休眠"的候选点——若要用于 Shadow Mode，思路是在 `tools/breath/__init__.py`（真正对外挂载的 `breath` 工具入口，证据ID：`/home/user/18358386529/ombre-brain/src/tools/breath/__init__.py:95` 处已有 `rt.record_v3_tool_event("breath", {...})` 调用点）旁边新增一次独立的只读旁路调用，而不是依赖现有的（休眠的）`v3_runtime` 链路。

**候选 B（kiwi-mem）：`search_memory` / `database.search_chat_messages`**

kiwi-mem 的记忆检索工具（`_QUARANTINED_TOOL_NAMES` 中列出的 `search_memory`，`tool_drawer.py:94-96`）与对话检索 `search_chat_messages`（`main.py:2825` 引用自 `database.py`）都是返回结构化结果列表的普通异步函数调用，`execute_drawer_tool`（`tool_drawer.py:1157-1185`）统一收口所有 drawer 工具的调用/返回，是天然的"读取返回值而不改变它"挂载点。

### 1.2 Injection 结果

**候选（xinchao-nian）：`buildContextEnvelope({...})`**

**证据ID：`/home/user/18358386529/xinchao-nian/xinchao/src/context-envelope.js:197-235`**
```js
export function buildContextEnvelope({
  state, sessionId, mode = 'session_start', ombreText = '', maxTokens = 2200,
  ttlMinutes = 15, now = new Date(), alreadyDelivered = false, force = false, ...
}) {
  ...
  if (alreadyDelivered && !force) {
    return {
      version: 1, system: 'xinchao-dynamic-mind', mode: normalizedMode,
      sessionId: safeSessionId, generatedAt: generatedAt.toISOString(),
      expiresAt: ..., delivered: false, alreadyDelivered: true,
      reason: 'session_start_already_delivered', sections: [],
      additionalContext: '', estimatedTokens: 0, digest: ...,
    };
  }
  ...
```
这个函数的返回值本身就是"即将被注入给 LLM 的上下文信封"（`sections`/`additionalContext`/`estimatedTokens`/`digest`/`delivered`），是一个纯函数（输入 `state` 等参数，输出一个新对象，未见对入参 `state` 做原地修改），**天然适合被旁路复制一份输出而不影响原调用**。同一文件里配套的 `recordContextDelivery`（174-196 行，未展开引用具体行为，unknown：本次未逐行读取其内部实现细节，仅确认其存在且与 `contextDeliveryState` 配套用于记录"是否已投递"状态）也值得关注，作为"Injection 是否真的发生"的确认点。

**候选（ombre-brain）：`SurfaceContextCompiler.compile(decisions, memories)` / `LegacyRuntime.compile_surface_context(...)`**

**证据ID：`/home/user/18358386529/ombre-brain/src/ombrebrain/app/legacy_runtime.py:322-342`**（见 07 号报告 3.2 节相邻代码）——`compile_surface_context` 把决策与记忆编译成"上下文投喂包"（`bundle`），同时跑一次 `FormalInvariantChecker.default().evaluate_context_items(...)`。同样受 08 号报告 3.3 节结论约束：这是 `LegacyRuntime` 的方法，实例化路径目前只在诊断/维护场景（`web/system.py:187`）出现，不在主 MCP 工具路径上被自动调用。

### 1.3 Action 结果

**候选（kiwi-mem）：`_execute_gateway_tool(tool_name, arguments, tool_info)` 与 `execute_drawer_tool(tool_name, arguments, scope)`**

见 08 号报告 2.1/2.2 节。两者都是 `(result_text, extra_metadata)` 或 `result` 的返回值收口点，调用点分别在 `main.py:3205` 和 `tool_drawer.py` 内部，是 kiwi-mem 里**唯一**真正产生外部副作用的 Action 收口处，天然适合在调用方外侧包一层只读旁路记录（记录 `tool_name`/`arguments`/返回值/异常），不需要修改这两个函数体本身。

### 1.4 Xinchao 状态（只读）

**候选（xinchao-nian）：`StateStore.read()`**

**证据ID：`/home/user/18358386529/xinchao-nian/xinchao/src/state-store.js:12-21`**
```js
async read() {
  try {
    return JSON.parse(await readFile(this.path, 'utf8'));
  } catch (error) {
    if (error.code !== 'ENOENT') throw error;
    const state = this.factory();
    await this.write(state);
    return state;
  }
}
```
`read()` 本身已经是一个无副作用的读取方法（唯一的写入分支只发生在文件从未存在过的首次初始化场景），是"只读观察 Xinchao 状态"的天然挂载点——Shadow Mode 观察者可以直接复用这个方法读取当前状态快照，不需要接触 `update(mutator)`（42-51 行，唯一的写入入口）。

**重要限制**：本仓库中能确认的"状态"仅限 `xinchao/src` 目录下的本地 JSON 状态文件（drives/consciousness/thoughtPool 等，见 09 号报告 1.2 节 `driveDeltas`/`thoughtPool` 字段旁证）。外部通过 `mcp__xinchao_context` 类工具暴露的会话上下文接口，其服务端实现是否落在 `bridge/` 子模块（未克隆，见页首说明）中，**unknown**，无法确认其只读边界。

### 1.5 Raw Events

**候选（haven-ombre）：`RawEventStore`（专用类，命名即为 "raw events"）**

**证据ID：`/home/user/18358386529/haven-ombre/raw_events.py:98-142`**
```python
class RawEventStore:
    """Append-only-ish raw dialogue archive with optional FTS search."""
    ...
    def _init_db(self) -> None:
        conn.execute("""
            CREATE TABLE IF NOT EXISTS raw_events (
                id INTEGER PRIMARY KEY AUTOINCREMENT, source TEXT NOT NULL,
                source_event_id TEXT NOT NULL DEFAULT '', event_hash TEXT NOT NULL,
                role TEXT NOT NULL, text TEXT NOT NULL, created_at TEXT NOT NULL,
                ingested_at TEXT NOT NULL, conversation_id TEXT NOT NULL DEFAULT '',
                session_id TEXT NOT NULL DEFAULT '', client TEXT NOT NULL DEFAULT '',
                metadata_json TEXT NOT NULL DEFAULT '{}',
                UNIQUE(source, event_hash)
            )
        """)
```
这是一个独立的 SQLite 表（不是生产 Bucket/Markdown 存储），docstring 自称 "append-only-ish"（本身设计意图就是追加为主），字段包含 `source`/`role`/`text`/`created_at`/`ingested_at`/`conversation_id`/`session_id`/`client`/`metadata_json`，语义上已经就是"原始事件"的存档，且用 `UNIQUE(source, event_hash)` 做了去重。这是四个仓库中**语义最贴合"Raw Events"要求、且已经是独立表（不与生产 Bucket/Markdown 混用）**的候选，理论上 Shadow Mode 可以直接只读查询这张表，不需要新建任何存储。

**限制**：`raw_event_text_looks_injected(text, raw=None)`（`raw_events.py:71`）和 `strip_raw_client_context`/`_strip_client_context_blocks`（44-70 行）说明这张表本身的写入路径带有"防 prompt 注入"清洗逻辑——即写入这张表的文本已经过处理，不是 100% 原始未加工文本；对于需要"绝对原始"输入的场景，这是需要注意的预处理边界，但不影响其作为"事件时间线"只读源的可行性。

### 1.6 现有 Memory 结构

**候选（ombre-brain / haven-ombre）：`BucketManager.list_all(include_archive=...)` / `BucketManager.get(bucket_id)`**

两者在 `tools/_common.py` 中被大量只读调用（如 `count_pinned`/`count_protected`/`restore_archived_letters` 均以 `await rt.bucket_mgr.list_all(...)` 起手，见 08 号报告未展开但 07 号报告已引用的同一文件），是明确的、已被系统内部大量复用的只读枚举接口，语义上直接对应"现有 Memory 结构"。

**候选（ombre-brain）：`MemoryFabric.replay_events()`**

`decision/debug.py:82`（`for event in reversed(self.fabric.replay_events()):`）已经把这个方法当作只读重放接口在用，是"读取事件溯源全量历史"的现成挂载点。

---

## 2. 现状与 Shadow Mode 草案之间的差距清单

1. **没有单一事件总线**：六类输入分散在三种不同技术栈（Python 同步/异步混合 × 2 个独立代码库、Node.js × 1 个独立代码库）、至少四个独立运行时进程里。草案设想的"旁路读取"在每一处都要单独实现，彼此不能共享一套订阅/推送机制。这是最大的工程差距，不是代码质量问题，是架构现实。
2. **候选挂载点中，语义最匹配的几处（ombre-brain 的 `_retrieval_metadata`/`compile_surface_context`）目前处于"未接线"状态**（08 号报告 3.3 节已证 `v3_runtime` 在主服务中为 `None`）。这意味着：即使决定"旁路读取"这些函数的输出，也必须先解决"这些函数在生产路径里根本不会被调用"的前置问题，或者改为直接在 `tools/breath/__init__.py`、`tools/hold/__init__.py`、`tools/trace/core.py` 这些**真正被 `server.py` 挂载**的入口处另开旁路（这些文件里已经有 `rt.record_v3_tool_event(...)` 调用作为参考位置，但如 08 号报告所证，该调用本身在主服务中当前也是 no-op，因为它同样依赖同一个未被赋值的 `v3_runtime`）。
3. **时间戳精度不统一**（09 号报告已详述）：ombre-brain/haven-ombre 秒级，xinchao-nian 毫秒级，kiwi-mem 数据库微秒级但应用层语义是状态字段而非事件流。Shadow Mode 若要输出统一的 `shadow_activation.jsonl`/`shadow_belief.jsonl`，需要先决定用哪个精度基准，现状无法直接拼接。
4. **Injection 结果与 Action 结果目前都没有"是否真的送达/执行成功"的独立确认信号**：`buildContextEnvelope` 返回 `delivered`/`alreadyDelivered` 字段（1.2 节），是少数明确带有"投递状态"语义的返回值；但 ombre-brain 的 `compile_surface_context` 和 kiwi-mem 的 Action 执行结果都没有等价的"确认已生效"回执（08 号报告已详述 kiwi-mem Action 结果混杂在返回字符串里，没有独立 RESULT 事件）。
5. **Xinchao 状态只读的边界不完整**：`StateStore.read()` 本身干净只读，但完整的"Xinchao 状态"很可能横跨 `bridge/` 子模块（未克隆，unknown）与本仓库内的多个状态文件（`state-store.js` 只是通用存取器，具体状态内容由调用方决定路径），本次审计无法确认所有会被写入/读取的状态文件路径清单，只能确认"读取机制本身无副作用"这一层。
6. **"不侵入主流程"的可验证性**：草案要求 Shadow Mode 不得控制 Injection/Action/写生产 Memory/改 Bucket。就本次找到的六个候选挂载点而言，全部是**读取已有函数返回值**（而非替换其实现或篡改其输入），理论上满足"旁路"要求；但由于候选点 2（ombre-brain 的两个休眠链路）当前根本不执行，"旁路读取一个不会被调用的函数"在实际效果上等于"读不到东西"——如果 Shadow Mode 设计者不了解 08 号报告 3.3 节的接线现状，可能会误以为挂载了这两个点就能拿到 Recall/Injection 数据，实际拿到的会一直是空值。这是本次审计认为**最容易被后续设计者忽视、从而导致 Shadow Mode 第一版"看起来接上了但拿不到数据"的坑**。

---

## 3. 不在本报告范围内的事项（明确排除）

- 本报告不提出任何新增字段、新增表、新增 Bucket 结构的建议（按任务要求）。
- 本报告不涉及 `shadow_belief.jsonl`/`shadow_activation.jsonl`/`shadow_metrics.json` 的具体输出 schema 设计（那是实现阶段的工作，超出"现状 vs 目标可行性差距"的审计范围）。
- `bridge/`（xinchao-runtime-bridge）子模块内部实现完全未审计（unknown，见页首说明）。

---

## P0级风险候选（本报告范围内）

- **无新增 P0**（NHPP/Bayesian 状态已在 09 号报告给出，本报告未发现额外的 Bayesian/NHPP 直接控制 Injection/Action 的证据）。
- **需要重点标注的设计陷阱**（非传统意义的"安全 P0"，但会直接决定 Shadow Mode 第一版是否能真正工作）：ombre-brain 中语义上最适合作为 Recall/Injection 旁路读取点的两处代码（`_retrieval_metadata`、`compile_surface_context`，均挂在 `LegacyRuntime`）当前在主服务运行时里因 `v3_runtime` 未被赋值而处于休眠状态（交叉引用 08 号报告 3.3 节的独立验证证据：`server.py:633-646` 的 `_tools_runtime.init(...)` 调用未传 `v3_runtime` 参数）。若 Phase 2+ 的 Shadow Mode 实现直接挂在这两个方法上而不先解决接线问题，会得到一个"部署了但拿不到数据"的空转 Shadow Mode，且因为没有报错（这两处都有 `except Exception` 兜底返回空 dict，见 07/08 号报告相应引用），这种"静默拿不到数据"的失败模式在运行时也不会主动报警，建议在设计阶段就显式排除这个陷阱。
