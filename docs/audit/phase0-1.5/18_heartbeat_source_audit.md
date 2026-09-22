# 18号审计：心潮念 heartbeat 真实来源定向审计

## 0. 审计模式声明

- **模式**：纯只读源码 + 部署配置审计。未启动任何服务、未执行任何写命令、未修改/移动/删除任何文件、未创建除本报告外的任何文件。
- **工具使用范围**：Read / Grep / Glob / Bash（仅 `find` `cat` `grep` `git log` `git show` `git ls-tree` `git rev-parse` 等只读命令）/ Write（仅写本报告）。未调用任何 `mcp__*` 工具。
- **目的**：只回答一个问题——心潮念（xinchao-nian）里"heartbeat"机制的真实写入方是谁；不做架构评估、不提修改建议之外的结论。

### 五个仓库锁定 commit 复核结果（`git -C <path> rev-parse HEAD`，本次实测）

| 仓库 | 任务书要求的锁定 commit | 实测 `rev-parse HEAD` | 是否一致 |
|---|---|---|---|
| `/home/user/18358386529/xinchao-nian/` | `97f1bdcc76b748fa516b7c79a0aa10234143795f` | `97f1bdcc76b748fa516b7c79a0aa10234143795f` | 一致 |
| `/home/user/18358386529/xinchao-dynamic-mind/` | 2.7.0 = `a9fddc2b24878c59bd9b5f0eabb64c1a679ac580`（当前 checkout） | `a9fddc2b24878c59bd9b5f0eabb64c1a679ac580` | 一致（当前 checkout 就是 2.7.0；2.4.0 = `69483299075ec865297a236325afa3d2ad7975dc` 通过 `git show <sha>:<path>` 单独核实，见下文） |
| `/home/user/18358386529/haven-ombre/` | `284c9c7b0e51a0ba0032c7028f705d72458cb304` | `284c9c7b0e51a0ba0032c7028f705d72458cb304` | 一致 |
| `/home/user/18358386529/ombre-brain/` | `6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5` | `6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5` | 一致 |
| `/home/user/18358386529/kiwi-mem/` | `b01a0c506f4f10f90f30d408c0291f16720f0718` | `b01a0c506f4f10f90f30d408c0291f16720f0718` | 一致 |

---

## 关键澄清：本次审计发现"heartbeat"其实是两个不同的机制，此前报告未拆开

在深入证据前必须先说明这一点，否则第1–6题的答案会互相矛盾。经过对 xinchao-nian、xinchao-dynamic-mind（含内嵌 `xinchao/` 与独立仓库两处）逐行核对，代码里同时存在**两套完全独立、互不调用的"心跳"**，中文都叫"心跳"，英文都含 heartbeat 字样，这正是 15/16/17 号报告未能查清来源的根源之一：

- **机制 A：在场心跳（presence heartbeat）**——`POST /v1/heartbeat` HTTP 路由，是**心潮自己（xinchao/dynamic-mind 服务）**的路由，不是 Ombre 的。写入方是 Claude Code 的 `UserPromptSubmit` hook 脚本 `scripts/xinchao-heartbeat-hook.sh`（或 MCP 工具 `xinchao_event`/`xinchao_context`），目的只是刷新心潮自身状态里的 `lastHeartbeatAt` 字段，判断"最近是否有真人在场"。这条链路**证据完整、已确认**。
- **机制 B：Ombre 文件心跳（ombre file heartbeat）**——`config.heartbeat.filePath`（默认值 `/memory-data/heartbeat.json`，环境变量 `OMBRE_HEARTBEAT_FILE`），由 `heartbeat-store.js` 的 `readOmbreHeartbeat()` 读取，`server.js` 的 `synchronizeOmbreHeartbeat()` 每个心跳循环（`runCycle`）调用一次，`engine.js` 的 `applyOmbreHeartbeat()` 消费其结果。`heartbeat-store.js` 源码注释明确写着"Read the content-free timestamp **written by Ombre's POST /heartbeat route**"——即代码作者自己声称这个文件应该由 Ombre 侧一个 `POST /heartbeat` 路由写入。**这条链路的写入方，是本次审计要定位、但最终没有在审计范围内的任何仓库中找到实现的机制。**

15/16/17 号报告说的"未知的心跳"、"完全黑盒"，指的是机制 B；机制 A 此前没有被单独提出来过，本报告是第一次把它和机制 B 分开确认。

---

## 1. 完整调用/数据流（能画多完整画多完整，缺失环节明确标出）

### 机制 A：在场心跳（已确认，完整闭环）

```
[人类在 Claude Code 里提交一条 prompt]
        │  UserPromptSubmit hook 触发
        ▼
scripts/xinchao-heartbeat-hook.sh   (xinchao-nian/xinchao/ 与 xinchao-dynamic-mind 两处代码一致)
  - 读 hook stdin JSON，只取 session_id，丢弃 prompt 正文
  - 生成 event_id
  - curl -X POST "$XINCHAO_HEARTBEAT_URL"（默认值文档示例 https://xinchao.example.com/v1/heartbeat，
    xinchao-nian/xinchao/README.md 里写的默认部署域名是 https://xinchao.guchuan.men/v1/heartbeat）
        │  HTTP POST { session_id, event_id }
        ▼
xinchao/dynamic-mind 自己的 server.js
  request.method === 'POST' && url.pathname === '/v1/heartbeat'
    → source = 'heartbeat'
    → recordConversationEvent(event, 'heartbeat')
        → updateState({ type: 'conversation_heartbeat', source: 'api', ... })
        → 只刷新 state.lastHeartbeatAt（不刷新 lastConversationAt，不记 arrival gap）
        ▼
StateStore.write() 原子写入 JSON 文件
  路径：config.js 里 STATE_PATH，默认 '/app/state/state.json'
  持久化位置：xinchao-nian/compose.yaml 里的具名卷 xinchao-state:/app/state
                （与 Ombre 完全无关，是心潮自己的容器内状态）
```

另一条并行写入路径（同样是机制 A，殊途同归）：MCP 工具 `xinchao_event`（`mcp-protocol.js:599` 附近处理，`xinchao_event` 的 `interaction_type` 由客户端/模型判断），走的是 `settleAndApplyConversationEvent`，效果同样是刷新 `lastHeartbeatAt`（engine.js:534，真实互动事件本身也顺带刷新心跳时间，见下方证据）。

**结论**：机制 A 从触发→网络请求→服务端路由→状态更新→落盘，全链路证据齐全，写入方明确是"心潮自己的 HTTP 路由 + 心潮自己的状态文件"，与 Ombre/Haven 完全无关。

### 机制 B：Ombre 文件心跳（未找到写入方，链路在读取端之前断裂）

```
[??? 缺失/未找到 ???]
  代码注释自称"Ombre's POST /heartbeat route"，但：
    - haven-ombre 全仓库搜索 "heartbeat"：只有 1 处命中（server.py 里 pulse 工具的注释，
      是"脉搏/pulse"的比喻性叫法，与文件写入无关）
    - ombre-brain（独立仓库 + xinchao-nian 内嵌副本）全仓库搜索 "heartbeat"：命中的
      /api/heartbeat 是 GET-only 路由（web/system.py:1517），只读 _LAST_OP_TS 返回
      JSON 状态给 Dashboard 心跳灯轮询，从不写文件，方法上就不是 POST
    - haven-ombre、ombre-brain 两个仓库全文搜索 "heartbeat.json"："0 命中"
    - haven-ombre、ombre-brain 两个仓库全文搜索 "/memory-data"："0 命中"
    - haven-ombre、ombre-brain 两个仓库全文搜索 "OMBRE_HEARTBEAT"："0 命中"
        ▼
                    （此处应有一次文件写入，但审计范围内的 5 个仓库中未发现任何代码执行它）
        ▼
/memory-data/heartbeat.json  (config.heartbeat.filePath 的默认值)
        │  仅当此文件真实存在且内容含 recordedAt/recorded_at 时才有效
        ▼
xinchao-nian/xinchao/src/heartbeat-store.js : readOmbreHeartbeat(filePath)
  （xinchao-dynamic-mind 2.4.0 与 2.7.0 两个版本代码逐字节一致，见第3节）
        ▼
server.js : synchronizeOmbreHeartbeat()
  每次 runCycle() 调用一次（runCycle 由定时器 + 每次收到事件触发，见第2题）
        ▼
engine.js : applyOmbreHeartbeat(state, recordedAt)
  → 内部调用 applyConversationEvent()，把"文件心跳"当作一次"对话事件"处理
  → state.consciousness = 'awake'; state.lastConversationAt = now; state.lastHeartbeatAt = now
        ▼
影响 contactIdleAllowed() / dreamAllowed() / proactiveBarkAllowed() 等"清醒/沉睡"判定
```

**结论**：机制 B 的链路从"读取端"（heartbeat-store.js）往下全部有代码、有测试；但"写入端"（谁往 `/memory-data/heartbeat.json` 里写 `{recordedAt: ...}`）在审计到的全部 5 个仓库源码中不存在任何实现，标记为**缺失/未找到**。

---

## 2. 六个问题逐一解答

### 问题1：heartbeat 文件由谁创建/写入？

**分两个机制回答：**

- 机制 A（`/v1/heartbeat` HTTP 路由，刷新 `lastHeartbeatAt`）：**已确认**。写入方是心潮/动态心智自己的 `server.js` 路由处理器，触发源是 `scripts/xinchao-heartbeat-hook.sh`（Claude Code UserPromptSubmit hook）或 MCP 工具 `xinchao_event`。这不是"文件"，是内存状态 + `StateStore` 原子写入 `/app/state/state.json`（心潮自己的具名卷 `xinchao-state`，不是 `/memory-data`）。

- 机制 B（`/memory-data/heartbeat.json` 文件，`config.heartbeat.filePath`）：**未知**——本次在全部 5 个仓库中重新搜索，仍未找到任何写入方。已搜索证据见第5节和下方证据表。`heartbeat-store.js` 注释声称写入方是"Ombre's POST /heartbeat route"，但 ombre-brain（独立仓库 `6f7335d0` 与 xinchao-nian 内嵌副本 `97f1bdcc` 下的 `ombre-brain/`）里唯一名为 `/api/heartbeat` 的路由是 **GET-only**（`ombre-brain/src/web/system.py:1517` `@mcp.custom_route("/api/heartbeat", methods=["GET"])`），只读内存变量 `_LAST_OP_TS` 返回状态给 Dashboard 心跳灯轮询用，代码里没有任何 `open(...).write` / `writeFile` 操作涉及这个路由，也没有任何地方处理 `POST /heartbeat` 或 `POST /api/heartbeat`。haven-ombre 中同名字符串"heartbeat"唯一命中是 `server.py` 里 `pulse` 工具的注释性英文名"Heartbeat, system status + memory listing"，是修辞比喻，不是端点、不是文件写入。

### 问题2：如果找到写入方，谁更新它、多久更新一次？

- 机制 A：更新频率 = 触发频率。Claude Code hook 版本是"每次用户提交一条 prompt 触发一次"（`UserPromptSubmit`），脚本内部有 `XINCHAO_HEARTBEAT_MIN_INTERVAL_SECONDS` 可选节流（默认 0 = 不节流，见 `xinchao-heartbeat-hook.sh:36`）；MCP 版本是"每次调用 `xinchao_event` 触发一次"，无固定周期，取决于对话频率。这不是本题问的"heartbeat 文件"，是心潮自己状态里的一个时间戳字段。
- 机制 B：**因未找到写入方，无法回答"谁更新、多久更新一次"**。读取端（`synchronizeOmbreHeartbeat`）本身的调用频率是"每次 `runCycle()` 执行一次"，`runCycle` 由 HTTP 请求触发的结算路径以及（若存在）后台定时器共同驱动——但这只是"读多久一次"，不代表"写多久一次"，因为写入方本身没有找到。

### 问题3：心潮念从哪里读取它？（独立复核 heartbeat-store.js 的完整读取实现）

三处逐一核实，源码**完全一致**（同一段代码被复制到三个位置）：

1. `/home/user/18358386529/xinchao-nian/xinchao/src/heartbeat-store.js`（第1–14行，整份文件）：
   ```js
   import { readFile } from 'node:fs/promises';

   /** Read the content-free timestamp written by Ombre's POST /heartbeat route. */
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
2. `/home/user/18358386529/xinchao-dynamic-mind/src/heartbeat-store.js`（当前 checkout = 2.7.0 `a9fddc2b`）：**与上面逐字节相同**（`git diff` 级别核对，无差异）。
3. 2.4.0（`69483299075ec865297a236325afa3d2ad7975dc`，用 `git show 69483299075ec865297a236325afa3d2ad7975dc:src/heartbeat-store.js` 读取）：**与 2.7.0/xinchao-nian 内容逐字节相同**。三处代码没有任何版本差异。

读取行为要点（已确认）：
- 文件不存在（`ENOENT`）→ 返回 `null`，不抛错、不崩溃。
- 文件存在但 JSON 解析失败，或 `recordedAt`/`recorded_at` 都缺失/非法日期 → `new Date('')` 是 Invalid Date，`Number.isFinite(at.getTime())` 为 `false` → 同样返回 `null`（视为"没有心跳"，非崩溃）。
- 只有其他类型的文件系统错误（权限、IO 错误等，`error.code !== 'ENOENT'`）才会向上抛出；`server.js:synchronizeOmbreHeartbeat()` 里用 `try/catch` 包住，抛出后会被捕获并记一条 `log('ombre_heartbeat_read_failed', ...)`，同样不会导致进程崩溃，只是当次不更新状态（`xinchao-nian/xinchao/src/server.js:93-99`、`xinchao-dynamic-mind/src/server.js:78-84`）。

### 问题4：compose / Docker / volume 是否真的能让两边看到同一个文件？

逐一检查结果：

| Compose 文件 | 服务 | 相关 volume/mount |
|---|---|---|
| `xinchao-nian/compose.yaml` | `ombre-brain` | 具名卷 `ombre-buckets:/app/buckets`（OB 记忆桶数据，读写） |
| `xinchao-nian/compose.yaml` | `dynamic-mind`（容器名 `ombre-dynamic-mind`） | 具名卷 `xinchao-state:/app/state`（心潮自己状态，读写）+ `ombre-buckets:/ombre-buckets:ro`（**只读挂载 OB 的桶目录，但挂载点是 `/ombre-buckets`，不是 `/memory-data`**） |
| `xinchao-dynamic-mind/compose.yaml`（2.7.0，当前 checkout） | `xinchao`（唯一服务） | `./state:/app/state` + `${MEMORY_DATA_HOST_PATH:-./memory-data}:/memory-data:ro`（**这是本次审计唯一出现 `/memory-data` 字样的 compose 文件**，但该 compose 文件里只定义了这一个服务，没有第二个服务挂载同一路径来写它） |
| `xinchao-dynamic-mind` 2.4.0（`git show 69483299...:compose.yaml`） | `xinchao`（唯一服务） | 与 2.7.0 完全一致：`./state:/app/state` + `${MEMORY_DATA_HOST_PATH:-./memory-data}:/memory-data:ro` |
| `haven-ombre/docker-compose.yml` | `ombre-brain` | bind mount 示例路径 `/Users/p0lar1s/.../Obsidian Vault/Ombre Brain:/data` + `./config.yaml:/app/config.yaml`；`tunnel` 服务只挂 `~/.cloudflared` | 
| `haven-ombre/docker-compose.user.yml` | `ombre-brain` | `./buckets:/data` |
| `haven-ombre/compose.hk.yml` | `ombre-brain` + `ombre-gateway` | 两个服务共享 `/srv/ombre-brain/buckets:/data`、`/srv/ombre-brain/state:/state`、`/srv/ombre-brain/config.yaml:/app/config.yaml`（这是**同一 compose 文件内两个服务共享同一宿主路径**的唯一实例，但共享的是 OB 自己的 buckets/state/config，跟心潮、跟 `/memory-data`、跟 `heartbeat.json` 都无关） |
| `ombre-brain/deploy/docker-compose.yml` | `ombre-brain`（唯一服务） | bind mount `${OMBRE_HOST_VAULT_DIR:-../buckets}:/app/buckets` |
| `ombre-brain/deploy/docker-compose.user.yml` | `ombre-brain` + 可选 `ombre-ollama` | `${OMBRE_HOST_VAULT_DIR:-./buckets}:/app/buckets`；ollama 用独立卷 `ollama-data` |
| `ombre-brain/deploy/docker-compose.multi.yml` | 多个 owner 各自一个 service（`ming`/`hong` 示例） | 各自独立卷 `./buckets-ming`、`./buckets-hong`，互不共享 |
| `ombre-brain/deploy/docker-compose.testing.yml` | `ombre-brain`（唯一服务，覆盖层） | 无新增 volume，只覆盖 build args/env |
| `xinchao-nian/ombre-brain/`（内嵌副本） | 无独立 compose 文件；`deploy/` 目录下只有 `fetch_cloudflared.py`，没有 `docker-compose*.yml`/`compose*.yaml` | 该子目录的部署完全依赖仓库根目录的 `xinchao-nian/compose.yaml`（已在上面列出） |

**明确回答**：**未发现任何 compose 文件建立了心潮容器与任何 Ombre/Haven 容器对 `/memory-data` 路径（或映射到它的同一个宿主机目录/具名卷）的共享挂载。** 唯一出现 `/memory-data` 字样的 compose 文件是 `xinchao-dynamic-mind/compose.yaml`（2.4.0 与 2.7.0 都一样），但该文件里只有 `xinchao` 一个服务把它只读挂载进来，同一 compose 项目里没有第二个服务（更没有任何 Ombre/Haven 服务）挂载同一宿主路径来写入这个文件。`xinchao-nian/compose.yaml`（心潮+OB 联合部署的那份，也是当前锁定 commit 里唯一真正让心潮和 OB 在同一 docker 网络/同一批 compose 服务里跑起来的文件）里，OB 与心潮之间唯一共享的具名卷是 `ombre-buckets`（挂载点分别是 OB 侧 `/app/buckets`、心潮侧只读的 `/ombre-buckets`），跟 `/memory-data`、跟 `OMBRE_HEARTBEAT_FILE` 完全无关；`xinchao-nian/compose.yaml` 里甚至没有出现 `/memory-data` 这个字符串（`memory-data` 只出现在 `xinchao-nian/xinchao/` 内部的 `.dockerignore`、`config.js` 默认值、`README.md`、`.env.example` 里，均为"心潮读什么路径"的说明，不是 compose 挂载定义）。

haven-ombre 与 ombre-brain 的全部 compose 文件（含独立仓库和 `xinchao-nian` 内嵌副本对应的根 compose）都不含 `/memory-data` 字符串，也不含任何一处把心潮容器纳入同一 compose 项目。

### 问题5：搜索范围清单（已搜索的位置 + 使用的命令）

逐仓库列出，供人工判断穷尽性：

**1. `/home/user/18358386529/xinchao-nian/`（含内嵌 `xinchao/` 与 `ombre-brain/` 两个子目录，整仓库一起搜）**
- `grep -ril "heartbeat" /home/user/18358386529/xinchao-nian -r` （排除 `.git`）→ 命中 14 个文件，逐一读取确认（`ombre-brain/src/web/system.py`、`ombre-brain/src/web/_shared.py`、`ombre-brain/src/server.py`、`ombre-brain/frontend/dashboard.html`、`ombre-brain/docs/INTERNALS.md`、`xinchao/src/dashboard-projection.js`、`xinchao/src/engine.js`、`xinchao/src/server.js`、`xinchao/src/heartbeat-store.js`、`xinchao/src/config.js`、`xinchao/README.md`、`xinchao/test/thought-feedback-cap-3.3.6.test.js`、`xinchao/.env.example`、`xinchao/CHANGELOG.md`）
- `grep -rn "heartbeat.json" /home/user/18358386529/xinchao-nian` → 只命中 `xinchao/src/config.js`、`xinchao/.env.example`（均为默认值声明，非写入）
- `grep -rln "memory-data" /home/user/18358386529/xinchao-nian` → 只命中 `xinchao/.dockerignore`、`xinchao/src/config.js`、`xinchao/README.md`、`xinchao/.env.example`
- `grep -rn "OMBRE_HEARTBEAT" /home/user/18358386529/xinchao-nian` → 只命中 `xinchao/src/config.js`、`xinchao/.env.example`
- `grep -rn "recordedAt\|recorded_at" /home/user/18358386529/xinchao-nian` → 命中均为 `personality-store.js`（人格月度快照，字段同名但语义无关）、`server.js`/`heartbeat-store.js`（读取端）、`ombre-brain/src/ledger_property.py`/`ledger_mirror.py`（event-sourcing 账本时间戳，与 heartbeat.json 无关的另一套字段）
- 单独检查 `xinchao-nian/ombre-brain/`（内嵌副本）目录结构 + `deploy/` 子目录，确认没有独立 compose 文件
- 读取 `xinchao-nian/compose.yaml` 全文，逐服务列出 volumes

**2. `/home/user/18358386529/xinchao-dynamic-mind/`（独立仓库，当前 checkout = 2.7.0 `a9fddc2b`，2.4.0 用 `git show` 单独核实）**
- `grep -ril "heartbeat" /home/user/18358386529/xinchao-dynamic-mind -r` → 命中 15 个文件（含 `scripts/xinchao-heartbeat-hook.sh`、`src/heartbeat-store.js`、`src/server.js`、`src/engine.js` 通过 import 关联、`test/heartbeat-store.test.js`、`test/http-api.test.js`、`test/engine.test.js`、`docs/DASHBOARD-INTEGRATION.md`、`SECURITY.md`、`README.md`、`CHANGELOG.md`、`.env.example`、`memory-data/.gitkeep`）
- `git show 69483299075ec865297a236325afa3d2ad7975dc:src/heartbeat-store.js`、`git show ...:src/config.js`、`git show ...:src/server.js`、`git show ...:compose.yaml` → 逐一读取 2.4.0 版本，与 2.7.0 逐字比对
- `git log --oneline --all -- scripts/xinchao-heartbeat-hook.sh` 与 `-- src/heartbeat-store.js` → 确认引入历史（`64bdf8c fix: complete HTTP handoff and heartbeat presence (#3)` 引入 hook 脚本；`heartbeat-store.js` 本体更早，`13e3a6a`/`900e4b4` 两次历史提交里已存在）
- `git ls-tree -r <2.4.0 sha>` 与 `git ls-tree -r <2.7.0 sha>` 对比文件列表，确认两版本都含 `scripts/xinchao-heartbeat-hook.sh`、`src/heartbeat-store.js`、`test/heartbeat-store.test.js`
- `grep -rn "heartbeat.json\|memory-data\|OMBRE_HEARTBEAT\|recordedAt\|recorded_at"` 全部执行（结果见问题1/3/4正文）
- 读取 `compose.yaml`（当前 checkout）与 2.4.0 `git show` 版本全文

**3. `/home/user/18358386529/haven-ombre/`**
- `grep -ril "heartbeat" /home/user/18358386529/haven-ombre -r` → 只命中 `server.py`（`pulse` 工具注释，非文件写入、非路由）
- `grep -rn "heartbeat.json"` → 0 命中
- `grep -rln "memory-data"` → 0 命中
- `grep -rn "OMBRE_HEARTBEAT"` → 0 命中
- `grep -rn "recordedAt\|recorded_at"` → 0 命中
- 读取全部 compose 文件：`docker-compose.yml`、`docker-compose.user.yml`、`compose.hk.yml`
- 检查 `.claude/hooks/session_breath.py` 是否含 heartbeat 字样 → 无命中

**4. `/home/user/18358386529/ombre-brain/`（独立仓库）**
- `grep -ril "heartbeat" /home/user/18358386529/ombre-brain -r` → 命中 6 个文件（`src/web/system.py`、`src/web/_shared.py`、`src/server.py`、`frontend/dashboard.html`、`tests/test_dashboard_auth_flow.py`、`docs/INTERNALS.md`），逐一读取，全部指向同一个 GET-only `/api/heartbeat` 状态灯路由，与文件写入无关
- `grep -rn "heartbeat.json\|memory-data\|OMBRE_HEARTBEAT"` → 均 0 命中
- `grep -rn "recordedAt\|recorded_at"` → 只命中 `src/ombrebrain/eventsourcing/ledger_property.py`、`ledger_mirror.py`（event-sourcing 账本条目时间戳字段，与心跳文件无关的另一套机制）
- 读取全部 `deploy/docker-compose*.yml`（`docker-compose.yml`、`docker-compose.user.yml`、`docker-compose.multi.yml`、`docker-compose.testing.yml`）
- 读取 `entrypoint.sh` 全文，确认其逻辑（config 文件初始化、代码热更新播种）与 heartbeat 无任何交集
- 检查 `.claude/hooks/session_breath.py` → 无 heartbeat 字样命中

**5. `/home/user/18358386529/kiwi-mem/`**
- `grep -ril "heartbeat" /home/user/18358386529/kiwi-mem -r` → 0 命中
- `grep -rn "heartbeat.json\|memory-data"` → 0 命中
- 读取 `docker-compose.yml` volumes 段 → 只有 Postgres 数据卷 `kiwi-pg-data`，与心跳/Ombre/心潮均无关
- 结论：kiwi-mem 与本次审计问题完全无关联，穷尽搜索后确认（预期成立）

以上搜索命令均在只读模式下针对本报告开头列出的、已复核的锁定 commit 工作区执行；`xinchao-dynamic-mind` 的 2.4.0 版本额外使用 `git show <sha>:<path>` 单独核实，未依赖当前 checkout 的工作区文件。

### 问题6：heartbeat 是否属于心潮念核心机制，还是外部遗留依赖

先复述 15/17 号报告已确认、本次未重新验证但直接引用作为判断依据的降级路径事实（问题3 部分本次做了独立复核，验证一致）：

- 心跳文件不存在（`ENOENT`）→ `readOmbreHeartbeat` 返回 `null` → `synchronizeOmbreHeartbeat` 直接 `return null`，不更新任何状态，**不报错、不崩溃、不影响 `runCycle` 其余逻辑**。
- 文件存在但内容损坏/字段非法 → 同样被 `Number.isFinite` 判定过滤为 `null`，效果等同"文件不存在"。
- "清醒/沉睡"（`consciousness`）状态机的驱动主力是**真实对话事件**（`applyConversationEvent`，见 `engine.js:526-536`，任何真实互动都会同时刷新 `lastConversationAt` 与 `lastHeartbeatAt`），文件心跳只是二者之一的补充输入，且明确弱于真实对话事件（源码注释："Any real conversation event is stronger evidence of presence than a content-free heartbeat"）。

基于此，给出两层判断：

**(a) 如果 heartbeat 文件真实存在且被写入（假设情形）**：从代码设计看，它是**"可选的、有完整降级路径的外部信号"**，不是核心依赖——即使它从未被写入，系统的清醒/沉睡判定仍能完全依赖真实对话事件正常工作，不会报错或阻塞。这一层结论沿用 15/17 号报告已确认的结论，本次复核未发现相反证据。

**(b) 结合"找不到任何写入方"+"compose 里没有任何共享 `/memory-data` 卷"这两个事实的判断**：
- 可以确认的是：**在本次审计到的这 5 个仓库、这些锁定 commit 的组合里，没有证据表明"Ombre 文件心跳"这个依赖当前真的在生效**——即找不到任何代码路径能让 `/memory-data/heartbeat.json` 被写入非空、格式合法的内容；也找不到任何部署配置能让心潮的运行环境访问到一个真的被 Ombre/Haven 更新过的该文件。这是一个**"没有证据表明生效"**的结论，证据是：写入方全仓库穷尽搜索为空 + compose 层面无共享路径。
- **不能确认的是**："这个依赖确认不生效"——因为：(1) 本次审计范围明确限定在这 5 个仓库的这几个锁定 commit，不排除存在审计范围之外的第三方脚本、定时任务、人工手动维护的文件、或未纳入本次审计的另一个仓库/私有部署脚本在生产环境里向这个路径写入文件；(2) `.env.example` 里 `OMBRE_HEARTBEAT_FILE` 可被运维手动配置指向任意路径，理论上运维可以用任何方式（cron、手工脚本、甚至手动 `echo` 写入）产生这个文件，这类"仓库之外的运维行为"不属于源码审计能够证伪的范围。
- 因此结论应表述为："**至少在本次审计到的仓库范围内，没有证据表明这个依赖当前真的在生效**"，而非"确认这个功能已失效/未使用"。这是审计任务书要求区分的两种不同强度的结论。

另需强调：机制 A（`/v1/heartbeat` 在场心跳）与机制 B 不同，机制 A 有完整、已确认的写入方和数据流，**不属于本题"找不到写入方"的黑盒范畴**——本次审计不应把机制 A 的确定性错误地"平移"到机制 B 上，也不应把机制 B 的黑盒状态"平移"到机制 A 上。

---

## 3. 证据表

| 结论 | 仓库 | 文件 | 行号/配置项 | 证据 | 状态 |
|---|---|---|---|---|---|
| 机制A写入方=心潮自己的HTTP路由 | xinchao-nian | `xinchao/src/server.js` | 902-906 | `if (... url.pathname === '/v1/heartbeat') { ... source='heartbeat'; recordConversationEvent(event, source) }` | 已确认 |
| 机制A写入方=心潮自己的HTTP路由 | xinchao-dynamic-mind | `src/server.js` | 1456-1461 | 与上等价实现（2.7.0，含 classifyExchange 分支） | 已确认 |
| 机制A触发源=Claude Code hook | xinchao-dynamic-mind | `scripts/xinchao-heartbeat-hook.sh` | 全文（1-80行） | UserPromptSubmit hook，POST 到 `$XINCHAO_HEARTBEAT_URL`，默认路径 `/v1/heartbeat` | 已确认 |
| 机制A文档确认 | xinchao-nian | `xinchao/README.md` | 179-181, 233-263 | API 表列出 `POST /v1/heartbeat`；心跳接入档位章节详述三种接入方式 | 已确认 |
| 机制A落盘位置=心潮自己状态文件 | xinchao-nian | `xinchao/src/config.js` | 21 | `statePath: process.env.STATE_PATH ?? '/app/state/state.json'` | 已确认 |
| 机制A状态卷 | xinchao-nian | `compose.yaml` | dynamic-mind 服务 volumes 段 | `xinchao-state:/app/state` | 已确认 |
| 机制B读取实现 | xinchao-nian | `xinchao/src/heartbeat-store.js` | 1-14 | `readOmbreHeartbeat(filePath)` 读取 `recordedAt`/`recorded_at` | 已确认 |
| 机制B读取实现（2.7.0） | xinchao-dynamic-mind | `src/heartbeat-store.js` | 1-14 | 与上逐字节相同 | 已确认 |
| 机制B读取实现（2.4.0） | xinchao-dynamic-mind | `src/heartbeat-store.js`（`git show 69483299...`） | 1-14 | 与 2.7.0 逐字节相同 | 已确认 |
| 机制B文件路径默认值 | xinchao-nian / xinchao-dynamic-mind | `src/config.js` | 各自 heartbeat.filePath 定义处 | `process.env.OMBRE_HEARTBEAT_FILE ?? '/memory-data/heartbeat.json'` | 已确认 |
| 机制B消费/判定 | xinchao-nian / xinchao-dynamic-mind | `src/server.js` / `src/engine.js` | `synchronizeOmbreHeartbeat`/`applyOmbreHeartbeat` | 见第1节数据流 | 已确认 |
| 机制B写入方 | ombre-brain（独立仓库+内嵌副本） | 全仓库 | 关键词`heartbeat`/`heartbeat.json`/`OMBRE_HEARTBEAT`/`recordedAt` | 唯一相关路由 `/api/heartbeat` 为 GET-only（`src/web/system.py:1517`），不写文件；其余检索项 0 命中 | 未知（已穷尽搜索，未找到） |
| 机制B写入方 | haven-ombre | 全仓库 | 同上关键词 | "heartbeat"唯一命中为`server.py`里`pulse`工具注释，非路由非写入；其余检索项 0 命中 | 未知（已穷尽搜索，未找到） |
| 机制B写入方 | kiwi-mem | 全仓库 | 同上关键词 | 全部 0 命中 | 未知（已穷尽搜索，未找到，且与本问题预期无关） |
| compose无共享/memory-data卷 | xinchao-nian | `compose.yaml` | 全文 | 未出现字符串"memory-data"；OB↔心潮共享卷仅`ombre-buckets`（挂载点`/app/buckets`与`/ombre-buckets`） | 已确认 |
| compose无共享/memory-data卷 | xinchao-dynamic-mind | `compose.yaml`（2.7.0与2.4.0均一致） | volumes段 | 唯一出现`/memory-data`的compose文件，但项目内仅1个服务`xinchao`挂载它，无第二方写入 | 已确认 |
| compose无共享/memory-data卷 | haven-ombre | `docker-compose.yml`/`docker-compose.user.yml`/`compose.hk.yml` | volumes段 | 均不含"memory-data"字符串；compose.hk.yml内两服务共享的是OB自己的`/data`/`/state`，与心潮无关 | 已确认 |
| compose无共享/memory-data卷 | ombre-brain | `deploy/docker-compose*.yml`（4个文件） | volumes段 | 均不含"memory-data"字符串，且均不含心潮/dynamic-mind服务 | 已确认 |
| consciousness降级路径 | xinchao-nian/xinchao-dynamic-mind | `src/engine.js` | 526-536, 696-708 | 真实对话事件独立刷新`lastConversationAt`/`lastHeartbeatAt`，不依赖文件心跳；文件缺失时仅`return null`不报错 | 已确认（引用15/17号报告结论，本次问题3独立复核一致） |

---

## 4. 对 Dynamic Mind 整合的实际影响（只提出问题，不做最终决策）

以下是本次审计结果对既有 `dynamic_mind_integration_proposal_v1.md`（"保留原样"决定）可能产生的影响范围，供人工重新评估，不构成审计结论之外的架构建议：

1. **需要澄清方案文档里"heartbeat"具体指哪一个机制**。如果方案文档在讨论"心跳"时没有区分机制 A（在场心跳，已确认闭环、属于心潮自身运行所必需的一部分）和机制 B（Ombre 文件心跳，写入方未找到），"保留原样"这个决定实际上是对两个风险完全不同的机制做了同一个处置，需要人工重新判断这个决定当初是基于哪个机制的理解做出的。

2. **如果"保留原样"的依据包含"Ombre 会持续提供心跳文件、心潮据此判断对方是否清醒"这一假设**，本次审计给出的证据是：该假设在当前审计到的 5 个仓库源码 + 部署配置组合里找不到任何支持它成立的写入方或共享挂载点。这意味着——如果生产环境里这个文件确实从未被写入过——机制 B 从部署以来可能一直处于"读取端存在、永远读到 null、静默无效"的状态，且由于降级路径完全静默（不报错、不告警），这种"设计上存在但从未生效"的状况可能长期未被察觉。是否属实、影响多大，取决于生产环境的真实文件系统状态，这不在本次只读源码审计的能力范围内，需要人工在实际部署环境里核实 `/memory-data/heartbeat.json`（或 `OMBRE_HEARTBEAT_FILE` 指向的实际路径）是否存在、内容是否合法、修改时间是否新鲜。

3. **如果核实后确认该文件从未被写入或早已停止更新**，需要人工评估：机制 B 所服务的"通过 Ombre 侧信号识别对方是否清醒"这个功能在实际系统中是否等价于从未启用；如果 Dynamic Mind 整合方案里有任何环节假设了这个功能在正常工作，相关假设需要重新核对。

4. **本审计不判断该差距是否需要修复、以何种方式修复、或是否应该在整合中移除机制 B 相关代码**——这些属于架构/产品决策范畴，不在本次只读定向审计任务范围内。

---

## 5. 红线自查

- [x] 未修改、删除、移动任何文件（含 5 个已克隆仓库内的任何文件）。
- [x] 未对任何 git 仓库执行 `add/commit/checkout -b/clean` 等写操作；全程只用 `git rev-parse`/`git show`/`git ls-tree`/`git log --oneline` 等只读命令。
- [x] 未启动任何服务、未运行任何测试、未执行任何构建/安装命令。
- [x] 未写代码、未修改任何架构、未触碰 Haven（haven-ombre）任何文件——对 haven-ombre 的操作全部是 `grep`/`cat`/`find` 只读检索。
- [x] 未修改 `/home/user/memory-system/docs/audit/phase0-1.5/` 目录下任何已存在文件（00–17号），未修改 `/home/user/memory-system/docs/design/` 下任何文件。
- [x] 仅创建/覆盖了被指定的唯一输出文件 `18_heartbeat_source_audit.md`，未创建其他任何文件。
- [x] 未调用任何 `mcp__*` 工具。
- [x] 未访问外部网络（未使用 WebFetch/WebSearch/git fetch/git pull 等）。
- [x] 所有结论均标注"已确认"或"未知"，未使用"很可能是"等模糊表述；找不到写入方的部分已完整列出搜索范围和搜索命令。
