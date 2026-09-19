# 08 — Action Loop 可审计性审计

**审计模式声明：本报告为源码审计（static source audit），非运行时审计。四个目标系统本次均无本地运行实例；所有结论均来自对已只读克隆到本地的源码快照的静态阅读，未启动任何服务、未执行任何写操作。**

## 仓库与锁定 commit

| 本地路径 | 参考仓库 | 要求锁定 SHA | 实测 HEAD | 是否一致 |
|---|---|---|---|---|
| /home/user/18358386529/haven-ombre | Yinglianchun/Haven-Ombre | 284c9c7b0e51a0ba0032c7028f705d72458cb304 | 284c9c7b0e51a0ba0032c7028f705d72458cb304 | 一致 |
| /home/user/18358386529/ombre-brain | P0luz/Ombre-Brain | 6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5 | 6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5 | 一致 |
| /home/user/18358386529/xinchao-nian | tianyupaipai-cmd/xinchao-nian | 97f1bdcc76b748fa516b7c79a0aa10234143795f | 97f1bdcc76b748fa516b7c79a0aa10234143795f | 一致 |
| /home/user/18358386529/kiwi-mem | LucieEveille/kiwi-mem | b01a0c506f4f10f90f30d408c0291f16720f0718 | b01a0c506f4f10f90f30d408c0291f16720f0718 | 一致 |

---

## 1. 结论摘要

四个仓库中，只有 **kiwi-mem** 有真正会产生外部副作用的 "Action"（联网搜索、提醒的创建/触发/更新/删除、外部 MCP 工具调用）。haven-ombre 与 ombre-brain 的"工具"（hold/breath/trace/grow/dream/anchor/i/plan 等）本质上都是**对自身记忆存储的读写**，不是对外部世界的动作（没有发邮件、发通知、调用第三方 API 之类的副作用）；ombre-brain 中確有一套"执行管线 + 策略引擎 + 决策留痕"的基础设施（`app/execution.py`、`app/legacy_runtime.py`），但经代码交叉验证，它**只挂在旁路记录路径上，从未在真实工具调用中被用作阻断/裁决通道**（详见第 3 节，与子代理 4 自己独立验证的证据一致，未依赖子代理 2 产出）。

---

## 2. kiwi-mem：真正的 Action Executor 位置

### 2.1 内置 Action：`_execute_gateway_tool`

**证据ID：`/home/user/18358386529/kiwi-mem/main.py:2769-2772`**
```python
async def _execute_gateway_tool(tool_name: str, arguments: dict, tool_info: dict) -> tuple:
    """执行网关内置工具（联网搜索、提醒系统等）。
    返回 (result_text, extra_metadata) 元组，extra_metadata 用于 SSE 事件附加信息。"""
```

覆盖的动作：`_gateway_web_search`（联网搜索，`main.py:2776-2813`）、`_gateway_search_conversations`（`main.py:2817+`）、以及通过 `tool_drawer.GATEWAY_TOOL_SCHEMAS`（`/home/user/18358386529/kiwi-mem/tool_drawer.py:75-81`）声明的 `_gateway_create_reminder` / `_gateway_list_reminders` / `_gateway_complete_reminder` / `_gateway_delete_reminder`。

调用点：`main.py:3205`（`result_text, extra_meta = await _execute_gateway_tool(...)`，位于模型工具调用循环内）。

### 2.2 外部 MCP 工具 Action：`execute_drawer_tool`

**证据ID：`/home/user/18358386529/kiwi-mem/tool_drawer.py:1157-1185`**
```python
async def execute_drawer_tool(tool_name, arguments, scope=None):
    ...
    try:
        import mcp_server
        func = getattr(mcp_server, tool_name, None)
        if not func:
            return f"[tool_error] tool_not_found: {tool_name}", extra
        result = await func(**arguments)
        return result, extra
    except TypeError as e:
        ... # 参数缺失/多余 → "[tool_error] ...: missing required arg" / "unknown arg"
    except Exception as e:
        print(f"❌ drawer工具 {tool_name} 执行失败: {e}")
        traceback.print_exc()
        return f"[tool_error] {tool_name}: execution failed", extra
```

这是 kiwi-mem 里最直接的 "Action Executor"：反射调用 `mcp_server.py` 中用 `@mcp_xxx.tool()` 装饰的异步函数，捕获异常并把错误编码进返回字符串（`[tool_error] ...`），**没有独立的 RESULT 事件对象**，返回值本身既是"结果"也是"错误载体"。

### 2.3 提醒（Reminder）——kiwi-mem 里唯一有持久化状态机的 Action

Schema：

**证据ID：`/home/user/18358386529/kiwi-mem/database.py:363-376`**
```sql
CREATE TABLE IF NOT EXISTS reminders (
    id TEXT PRIMARY KEY, title TEXT NOT NULL, notes TEXT DEFAULT '',
    trigger_time TIMESTAMPTZ NOT NULL, repeat_type TEXT DEFAULT 'once',
    repeat_config JSONB DEFAULT '{}', status TEXT DEFAULT 'pending',
    enabled BOOLEAN DEFAULT TRUE, last_fired_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(), updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

创建具备幂等语义（`ON CONFLICT (id) DO UPDATE`，证据ID：`/home/user/18358386529/kiwi-mem/database.py:5139-5146`）；触发（`fire_reminder`，`database.py:5248-5282`）对一次性提醒是"设 status=completed"，重复执行是幂等的（把已完成的再标一次完成，不会产生重复副作用）；对循环提醒是"计算下一次触发时间"，重复调用会不断推进 next_time，**不是幂等的**（如果 `/reminders/{rid}/fire` 被重试或重复触发，循环提醒的下次时间会被多次推进——这是一个真实的重复执行风险，见第 4 节缺口清单）。

**HTTP 层错误处理**（`main.py:6182-6246`，`api_get_reminders`/`api_create_reminder`/`api_fire_reminder`/`api_update_reminder`/`api_delete_reminder`）：全部是 `try/except Exception: return JSONResponse(status_code=500, ...)`，即**"失败"与"未执行"在这一层是可以区分的**（500 vs 404 "提醒不存在" vs 200 `{"ok": false}"`），但**没有独立于 HTTP 响应之外的、可事后审计的 RESULT 事件持久化记录**——一次 `fire_reminder` 调用是否真的发生过、发生在什么时刻、幂等键是什么，除了 `reminders.last_fired_at`/`status` 这两个"当前状态"字段外，没有历史轨迹（没有 `reminder_fire_log` 之类的追加表）。

---

## 3. ombre-brain：存在"执行管线"基础设施，但未接入真实工具调用路径（关键发现）

### 3.1 基础设施本身

`src/ombrebrain/app/execution.py` 定义了 `ExecutionEnvelope`/`ExecutionOutcome`/`LegacyExecutionPipeline`，`ExecutionPhase` 枚举（`RECEIVED → VALIDATING → AUTHORIZING → EXECUTING → APPLYING → RECORDING → COMPLETED/FAILED`，证据ID：`/home/user/18358386529/ombre-brain/src/ombrebrain/app/execution.py:17-25`）。`LegacyExecutionPipeline.run()`（同文件 78-106 行）会：先做策略裁决（`_authorize_policy`），执行 handler，捕获异常时记录 `ExecutionOutcome(ok=False, error_type=..., error_message=...)`，成功时记录 `ok=True`——**这确实区分了"失败"（handler 抛异常）与其他状态**，但它**不区分"未执行"**（如果 policy 裁决拒绝，会在 `_preflight`/`_authorize_policy` 阶段直接 `raise PermissionError` / `raise PolicyViolation`，never 调用 handler；这种"策略拒绝未执行"与"handler 执行后失败"两者都通过 `ok=False` + 不同的 `error_type`（`PermissionError`/`PolicyViolation` vs 具体异常类名）区分，语义上是完备的）。

类的 docstring 直接自述其定位：

**证据ID：`/home/user/18358386529/ombre-brain/src/ombrebrain/app/execution.py:67-73`**
```python
class LegacyExecutionPipeline:
    """Best-effort execution envelope for legacy modules.

    The pipeline intentionally returns handler results unchanged and re-raises
    handler exceptions unchanged. The v2.4.0 side channel records behavior; it does
    not become a new source of visible legacy behavior.
    """
```

即这套管线自我定位为"记录行为的旁路（side channel）"，明确声明"不会成为新的可见行为来源"。

### 3.2 决策留痕（Decision Ledger）会写进 MemoryEvent.metadata，但只是追加事件

`LegacyRuntime.record_execution_event` / `record_tool_event`（`/home/user/18358386529/ombre-brain/src/ombrebrain/app/legacy_runtime.py:152-290`）把策略裁决（`policy_verdict`）、投影审计（`projection_journal`/`projection_observations`）、事件溯源快照（`event_sourced_kernel`）、决策记录（`decision_record`）全部塞进一条 `MemoryEvent(memory_type=TRACE, ...)` 的 `metadata` 字段，然后 `self.fabric.append_event(event)`。这是一条**只追加（append-only）的审计轨迹**，天然具备"事后可重放"的形状（`DecisionDebugService.replay()`，`decision/debug.py:59-69`）。

### 3.3 关键交叉验证：这套管线是否真的挂在 hold/breath/trace/grow 的调用路径上？—— 否

`src/tools/_runtime.py` 是所有工具子模块访问共享运行时的唯一入口（`from . import _runtime as rt`，见 `tools/_common.py:47`）。它暴露四个 v3 相关 helper：`run_v3_capability`、`record_v3_tool_event`、`run_v3_operation`、`run_v3_async_operation`（`_runtime.py:71-205`）。其中 **只有 `run_v3_operation`/`run_v3_async_operation` 会调用 `LegacyExecutionPipeline.run/run_async`**（即真正会执行策略裁决、可能 `raise PolicyViolation` 阻断），而 **`record_v3_tool_event` 的 docstring 明确写明它是纯旁路记录**：

**证据ID：`/home/user/18358386529/ombre-brain/src/tools/_runtime.py:97-98`**
```python
def record_v3_tool_event(tool_name: str, payload: dict[str, Any] | None = None) -> Any:
    """Record a legacy tool call in the v3 side channel without affecting output."""
```

对生产代码的穷举 grep（证据ID，见下方命令与结果）显示：**生产工具代码中只调用了 `record_v3_tool_event`，从未调用 `run_v3_operation` / `run_v3_async_operation` / `run_v3_capability`**：

```
$ grep -rn "run_v3_operation|run_v3_async_operation|record_v3_tool_event|run_v3_capability" /home/user/18358386529/ombre-brain/src
tools/hold/__init__.py:180:    rt.record_v3_tool_event("hold", {...})
tools/breath/__init__.py:95:   rt.record_v3_tool_event("breath", {...})
tools/trace/core.py:169:       rt.record_v3_tool_event("trace", {...})
```

（`run_v3_operation`/`run_v3_async_operation`/`run_v3_capability` 三个函数本身在 `_runtime.py` 中定义，但在 `src/tools/**/*.py` 全目录范围内**没有任何调用点**——即那条"真正会做策略裁决、可能阻断执行"的路径，在生产工具代码里完全没有被接线使用。）

**证据ID（v3_runtime 附加位置）：`/home/user/18358386529/ombre-brain/src/server.py:633-646`**
```python
_tools_runtime.init(
    config=config, bucket_mgr=bucket_mgr, deletion_requests=deletion_requests,
    dehydrator=dehydrator, decay_engine=decay_engine, embedding_engine=embedding_engine,
    embedding_outbox=embedding_outbox, import_engine=import_engine,
    source_store=source_store, logger=logger, fire_webhook=_fire_webhook, mark_op=_mark_op,
)
```

**注意此处的 `init(...)` 调用完全没有传 `v3_runtime=...` 参数**，而 `tools/_runtime.py:42` 中 `v3_runtime: Any = None` 是模块级默认值。也就是说，在 `server.py`（MCP 工具真正挂载、`hold`/`breath`/`trace`/`grow` 等被 `@mcp.tool()` 注册的主入口）跑起来之后，`tools._runtime.v3_runtime` 仍然是 `None`。据此 `record_v3_tool_event`（`_runtime.py:97-127`）在其函数体第一行 `runtime = globals().get("v3_runtime")` 拿到的是 `None`，`getattr(runtime, "record_tool_event", None)` 返回 `None`，函数直接 `return None`——**在 `server.py` 驱动的主运行时里，连"旁路记录"本身都是 no-op**，`LegacyRuntime` 只在 `web/system.py:187`（诊断/维护 API，`LegacyRuntime.from_config({...})`，用于按需构造一次性运行时对象服务诊断请求）与测试代码中被实例化，不是随主服务常驻的单例。

**结论**：ombre-brain 的"决策/策略/形式化不变量"整套基础设施代码质量高、测试覆盖广（`tests/test_v3_*` 数十个文件），但它是一套**未被主服务 (`server.py`) 实际挂载和调用的旁路系统**。这既是好消息（它天然符合"不控制 Injection/Action"的硬性约束，因为它现在压根没接线），也是坏消息（如果 Phase 2+ 的设计文档假设"ombre-brain 已经有一层在裁决工具调用"，这个假设不成立——目前没有任何东西在裁决 hold/breath/trace/grow 的执行）。

### 3.4 grow 的幂等保护（唯一发现的真正幂等机制）

**证据ID：`/home/user/18358386529/ombre-brain/src/tools/grow/retry_guard.py:1-133`**

`retry_guard.py` 实现了一个**进程内、按事件循环隔离**的幂等去重：`request_fingerprint()` 对 `(content, items, test_data)` 做 SHA256 指纹；`run_once()` 用 `_LoopState`（`lock`/`inflight`/`completed` 三个字典）区分三种状态：
- 指纹命中 `completed` → 返回 `"✅ 已识别为刚才 grow 的重试；未重复写入。\n" + 原结果`（复用结果，不重复写入）；
- 指纹命中 `inflight` → 返回 `"⏳ 相同的 grow 仍在后台处理中；无需重复提交..."`（正在处理中，不重复触发）；
- 否则真正执行，成功后缓存 30 分钟（`RETRY_WINDOW_SECONDS = 30*60`），异常时从 `inflight` 移除、**不缓存失败**（"a genuine failure can be retried normally"，同文件 10 行注释）。

这是四个仓库中**唯一**一处明确区分"重复提交 = 正在执行中 / 已完成可复用 / 全新执行"三态的幂等实现。但注意其局限：
- 仅覆盖 `grow` 一个工具，`hold`/`breath`/`trace`/`anchor`/`plan`/`dream`/`I` 等其余工具没有等价机制（未在对应目录发现 `retry_guard.py` 同类文件，`grep -rl "retry_guard\|request_fingerprint" src/tools` 只命中 `grow/`）；
- 状态是**进程内存**（`weakref.WeakKeyDictionary` 按 `asyncio` 事件循环存储），进程重启/多进程部署下完全失效，不是持久化的幂等键。

---

## 4. 明确的缺口清单

1. **没有**跨工具统一的"RESULT 事件"schema——ombre-brain 有 `ExecutionOutcome`（`ok`/`phase_history`/`result_type`/`error_type`/`error_message`），但只在旁路记录里出现，且如 3.3 所述该旁路目前在主服务里是 no-op；kiwi-mem 的"结果"就是 HTTP 响应本身，没有独立持久化的 RESULT 记录。
2. **没有**跨工具统一的幂等机制——`grow` 有（进程内、30 分钟窗口），`hold`/`breath`/`trace` 等没有；kiwi-mem 的 `create_reminder` 有 DB 级 `ON CONFLICT` 幂等，但 `fire_reminder` 对循环提醒**不是幂等的**（重复触发会多次推进 `next_time`，这是真实的重复执行风险，见 2.3）。
3. **"失败"与"未执行"的区分**：ombre-brain 执行管线层面理论上可以靠 `error_type` 区分（`PolicyViolation`/`PermissionError` = 未执行被拒绝 vs 具体业务异常类名 = 已执行但失败），但由于该管线未被生产工具调用（3.3），这个区分能力目前不可用；kiwi-mem 层面，`execute_drawer_tool` 的 `TypeError`（参数不匹配，实际上函数调用发生了但立即因签名不匹配失败）和其他 `Exception`（可能发生在 handler 内部任意阶段）都被统一编码为 `[tool_error] ...: execution failed` 字符串，**调用方无法从返回值本身区分"根本没跑到业务逻辑"还是"跑到一半失败"**。
4. **没有**统一的 Action 审计日志（谁在何时调用了哪个 Action、参数是什么、结果是什么、是否重试过）——kiwi-mem 的提醒表只存"当前状态"，不存历史事件流；ombre-brain 的决策记录理论上具备这个能力（`DecisionRecord`），但因未接线（3.3）目前不产生任何记录。
5. haven-ombre：本次审计**没有**在其 `server.py`/`gateway.py`（分别含 19 处 `@mcp.tool()` 装饰，证据ID：`grep -n "@mcp.tool()" /home/user/18358386529/haven-ombre/server.py` 命中 19 处）中发现任何对外部世界产生副作用的 Action（联网搜索/通知/第三方 API），其工具集与 ombre-brain 同源（同为记忆读写类工具）；unknown：未逐一核对 19 个工具函数体是否 100% 局限于自身存储读写，标记为待后续深挖项。

---

## P0级风险候选（本报告范围内）

- **无 Bayesian/NHPP 相关发现**（见 09 号报告的专项结论）。
- **值得标注但非 P0 的架构风险**：ombre-brain 存在一套"看起来像已经有裁决层"的完整代码（策略引擎、决策留痕、执行管线），如果后续设计者仅根据代码存在与测试覆盖率判断"Action 已经有裁决/审计能力"，会产生**错误的安全感**——实际上该能力当前对主服务完全不生效（3.3 节证据）。建议在 Phase 2+ 设计评审中显式确认这一点，避免把"未接线的旁路代码"误当作"已生效的控制层"。
- kiwi-mem 循环提醒 `fire_reminder` 的非幂等重复触发风险（4.2），如果上层调用方（前端轮询、`api_fire_reminder`）存在并发重复调用的可能，会导致提醒时间被错误多次推进；不是 Bayesian/NHPP 相关，但属于 Action Loop 层面的真实正确性缺口，建议记录备查。
