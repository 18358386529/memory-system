# 16. 心潮念 3.3.6 十二维驱力 × Haven-Ombre Memory Recall / 自动注入 整合点定向审计

## 0. 审计模式声明

**本报告是源码审计，不是运行时审计。不进行实施、不部署、不迁移数据、不修改任何生产代码、不新建任何数据库/表/embedding/Memory Core。** 所有结论均基于对本地已克隆文件的静态阅读（`Read`/`Grep`/`Bash` 只读命令），未启动任何服务，未修改任何文件（本次唯一的写操作是本报告文件本身）。

### 锁定 commit 复核（审计开始前执行）

```
$ git -C /home/user/18358386529/xinchao-nian rev-parse HEAD
97f1bdcc76b748fa516b7c79a0aa10234143795f

$ git -C /home/user/18358386529/haven-ombre rev-parse HEAD
284c9c7b0e51a0ba0032c7028f705d72458cb304
```

两者均与任务书给定的锁定 commit **一致**。以下所有证据行号均对应这两个 commit 下的文件内容。

### 前提说明

总控已暂定 Memory System 2.0 采用心潮念 3.3.6 的 12 维持续驱力模型作为 Dynamic Mind 核心；`haven-ombre/persona_engine.py` 现有情绪系统暂不作为核心，不再深审。**本报告不重新评估该产品/架构决策是否正确**，只回答一个纯技术问题：在现有代码事实基础上，12 维驱力与 Haven 现有 Memory Recall / 自动注入机制之间，哪些连接**已经存在**、哪些**需要新增**、哪些**看起来能接但实际没有支持**。

本报告可引用 `15_xinchao_nian_dynamic_mind_3.3.6_audit.md` 已验证过的驱力增长/衰减/耦合等行号级证据（注明"参见15号报告"），但 Drive→Memory 调用链是否真实存在、是否真的改变 Haven 侧 Recall 排序，本报告全部重新独立追踪验证，不采信15号报告未覆盖的因果性结论。

---

## 1. 12维 Drive 实际运行链

### 1.1 十二维驱力清单（`xinchao/src/dimensions.js:12-109`，commit `97f1bdc`）

| 维度 | 中文含义 | growPerHour | ceil | satisfyMul | 特殊字段 |
|---|---|---|---|---|---|
| possess | 想她、占有与靠近 | 0.105 | 0.82 | 0.30 | nightMul 0.4, dawnFreeze |
| monitor | 惦记她、想知道她在做什么 | 0.090 | 0.78 | 0.70 | dawnFreeze |
| crave | 馋她、想黏着她 | 0.060 | 0.68 | 0.35 | dawnFreeze |
| share | 想分享自己的发现和感受 | 0.045 | 0.55 | 0.40 | dawnFreeze |
| libido | 性欲和身体上的渴望 | 0.020 | 0.50 | 0.15 | nightMul 0.4, `inhibitedBy` reflection/curiosity/boredom |
| curiosity | 好奇、想探索新东西 | 0.030 | 0.50 | 0.45 | dawnFreeze |
| boredom | 无聊、想找点事情做 | 0.030 | 0.50 | 0.25 | dawnFreeze |
| social | 想聊天、想接触热闹 | 0.025 | 0.48 | 0.40 | dawnFreeze |
| duty | 责任感、想把未完成的事推进 | 0.022 | 0.45 | 0.50 | dawnFreeze |
| reflection | 想沉淀、整理和理解自己 | 0.013 | 0.42 | 0.35 | dawnFreeze |
| grieve | 难过与失落 | 0（无增长项） | 0.15 | 0.60 | `decayHalfLifeHours=24`，纯回落 |
| anger | 生气与不满 | 0（无增长项） | 0.15 | 0.40 | `decayHalfLifeHours=24`，纯回落 |

初始值统一 0.15（`engine.js:279` `newState()`）。**状态：已确认。**

### 1.2 增长/衰减/阈值/触发机制

- **心跳结算**：`settleState`（`engine.js:387-503`），由 `server.js` 每 `SETTLE_INTERVAL_MINUTES`（默认 15 分钟，`config.js:29`）调用一次，`elapsedHours` 补齐结算，非事件驱动的实时触发。**已确认。**
- **凌晨冻结**：`dawnFreeze=true` 的维度在 `[dawnFreezeStart, dawnFreezeEnd)`（默认 1-8 点，`config.js:143-144`）内增速为 0（参见15号报告 `engine.js:387-503` 分析）。
- **天花板松弛**：超过 `ceil` 后按 `CEIL_RELAX_PER_HOUR=0.10`（`engine.js:11`）每小时松弛回落，而非瞬间归位。
- **满足平台期**：`satisfactionPlateaus`，某驱力被满足后一段时间（默认 `SATIETY_HOURS=2h`，`config.js:146-148`）增速强制为 0。
- **耦合**：`DRIVE_GROWTH_COUPLINGS`（`engine.js:26-30`）——`anger` 抑制 `possess` 增速（slope -0.65）；`grieve` 促进 `crave`/`monitor` 增速（slope +0.50/+0.45）。只改增速，不直接加值。**已确认**（参见15号报告同一常量的独立验证）。
- **人格偏置**：`personality-store.js` 的 `getDriveBias` 对 7 个驱力施加 ±10% 增速偏置（`personality-store.js:6,203-215`），fail-safe 缺省为 1.0。**已确认**（参见15号报告）。

### 1.3 驱力间相互影响

- 增速层面：`DRIVE_GROWTH_COUPLINGS`（上述）。
- 交叉抑制：`libido.inhibitedBy = {reflection:0.96, curiosity:0.95, boredom:0.93}`（`dimensions.js:50-54`）——其他驱力值越高，libido 增速被压得越低。**已确认。**
- 念头池回推：`tickThoughtPool` 对晋升为 obsession 且强度突破 `FEEDBACK_CEIL=0.85` 的念头，回推 `FEEDBACK_AMOUNT=0.18` 到对应驱力（首次触发即止，`thought-pool.js:5-10,45-47`），受 `THOUGHT_FEEDBACK_CAP=0.85` 封顶（`engine.js:12,471-479`）。这是"念头→驱力"闭环，不是"驱力→驱力"。**已确认。**

### 1.4 emotion → drive 的关系

**存在，且是双向耦合中的一支，方向为 Emotion → Drive（增速调制）**：`EMOTION_GROWTH_MODULATION`（`emotion.js:225-236`，参见15号报告）——valence/arousal 偏离中性时，调制对应驱力自然增速的乘法因子，钳制在 `[0.4,1.8]`，只改"长多快"不直接加数值。**已确认。**

同时存在反方向 **Drive → Emotion（目标点拉扯）**：`DRIVE_PULL`（`emotion.js:53-56`）——仅 `grieve`（valence -0.35, arousal -0.05）与 `anger`（valence -0.25, arousal +0.30）两个驱力参与，通过 `emotionTarget(drives)`（`emotion.js:100-110`）计算情绪的"回落目标点"，再由 `settleEmotion`（`emotion.js:148-161`，被 `engine.js` 的 `settleState` 每次心跳调用）按半衰期指数回落向该目标靠近。**注意：这是本报告的关键发现之一——12 维驱力中只有 grieve/anger 两维能反向影响情绪坐标，且是通过"拉扯回落目标+半衰期滞后"起效，不是瞬时脉冲；其余 10 维（possess/monitor/crave/share/libido/curiosity/boredom/social/duty/reflection）对 valence/arousal 没有任何写入路径。** 全仓库 `grep DRIVE_PULL` 只命中 `emotion.js:53` 定义处和 `emotion.js:104` 使用处，无其他消费点。**已确认（逐行追踪 emotion.js:53-56,100-161 + engine.js 对 settleState/settleEmotion 的调用点）。**

这条 Drive→Emotion 通路的下游意义在第 2 节详细展开，因为 emotion.valence/arousal 正是唯一被证实能实际改变 Haven 侧排序打分的字段。

### 1.5 personality → drive 的关系

`personality-store.js` 的 `CORE_TO_DRIVES`（`personality-store.js:25-30`）：14 维月度自评中仅 4 组标签映射到 7 个驱力的 ±10% 增速偏置（"爱与依恋"→possess/crave 正向；"表达"→share 正向；"平静与安全"→grieve/monitor 负向；"欲望与动机"→libido/curiosity 正向）。其余 10 维自评标签不影响驱力。**已确认（参见15号报告 personality-store.js:25-40,203-243 的独立验证，本次未重新逐行复核，采信15号报告结论，标注为"推断"——本报告未重新独立读取该文件全文）。**

### 1.6 `thought-pool.js` 是否读取 drive

是。`reinforceThought`/`addFlashThought`（`thought-pool.js:94-114`）以 `driveKey` 为念头的分类键；`pickIntent`（`engine.js:664-694`）遍历 `state.drives` 找出并列最高驱力集合，再用 `obsessionBonus(pool, key)`（读取 `state.thoughtPool.obsessions`）做加权随机选择——这是"drive 选出候选池，念头池反过来给候选加权"的双向读取。**已确认，行号：`engine.js:664-694`。**

### 1.7 autonomous wake（自主唤醒/自主念头）是否读取 drive

是，且是硬性阈值门控。`proactiveBarkAllowed`（`engine.js:970-974`）：
```js
export function proactiveBarkAllowed(state, now, minIntervalHours, maxPerDay, minDrive) {
  if (state.consciousness !== 'sleeping') return false;
  const strongest = Math.max(...Object.values(state.drives).map(Number));
  return strongest >= minDrive && barkAllowed(state, now, minIntervalHours, maxPerDay, 'autonomous_thought');
}
```
`minDrive` 默认 0.42（`config.js:139` `BARK_MIN_DRIVE`）。只有最强驱力 ≥0.42 且处于睡眠状态才允许自主念头推送（`server.js` 步骤 10，参见15号报告时序还原）。`pickIntent(state)`（`server.js:1449`）也读取 `state.drives` 选出当前"最想要什么"的意图。**已确认，行号：`engine.js:970-974`；`server.js:1449`。**

### 1.8 drive 是否会持续变化

是，每 15 分钟（`settleIntervalMinutes` 默认值）随 `settleState` 结算一次，属于时间连续函数（不是事件触发才变化），参见1.2。**已确认。**

### 1.9 什么情况下 drive 会影响"外部行为"（真正产生副作用：对话/推送/记忆调用）

逐一核实产生真实外部副作用（网络调用/推送）的路径，均需满足对应配置开关（**多数默认关闭**，见下表）：

| 外部行为 | 触发条件（含 drive 参与方式） | 默认是否开启 | 证据 |
|---|---|---|---|
| Bark 推送（梦余韵） | `dreamAllowed` 判定睡眠中且间隔满足；drive 只参与"梦材料回退到 recentMaterialWithRefs"这一分支的 query hint | `BARK_ENABLED=false`；`SHADOW_MODE=true`；`OMBRE_READ_ENABLED=false` | `config.js:31,50,128` |
| Bark 推送（自主念头） | `proactiveBarkAllowed`：最强驱力 ≥0.42 且睡眠中 | `BARK_ENABLED=false` | `engine.js:970-974`；`config.js:128,139` |
| 调用 OB `breath`（拉取记忆材料，供 LLM 生成梦/念头文案） | 见第2节 | `OMBRE_READ_ENABLED=false`，`SHADOW_MODE=true` 时整条 OB 读取分支被跳过（`server.js:264` `if (!config.shadowMode && config.ombre.readEnabled)`） | `server.js:264,270,361,450` |
| 调用 OB `hold`/`grow`/`I`/`trace`（写入正式记忆） | 黑匣子 keep、觉察确认、梦落盘等，均不以 drive 数值为直接触发条件，而是"某个动作发生了"（AI 主动调用或结算流程产出梦/念头文案后落盘） | `OMBRE_WRITE_ENABLED=false` | `config.js:51`；`ombre-client.js:282-350` |
| 自身信号推送（drive_peak 等） | `detectSelfSignals` 读取 drive 突增轨迹（参见15号报告 self-signals.js:171-186），但只写追踪字段+入队 bridge | `BRIDGE_ENABLED=false`，`BRIDGE_SELF_SIGNALS=false` | `config.js:100,107` |

**结论（已确认）**：drive 数值本身要转化为任何真正的外部副作用（对话推送、OB 读写调用），链路上全部经过至少一层默认关闭的开关（`shadowMode=true`、`ombre.readEnabled=false`、`ombre.writeEnabled=false`、`bark.enabled=false`）。在默认部署配置下，12 维驱力对外部世界（包括对 Haven Memory 的任何读写调用）**没有任何可观测的副作用**——这是理解"Drive→Memory"整条链路现实强度的前提性事实，第2节的所有正向发现都要叠加这一层"默认关闭"的限定。

---

## 2. Drive → Memory 实际调用链（重点）

### 2.1 逐一核对 `ombre-client.js` 的记忆相关调用

`ombre-client.js`（全文 727 行）中与 Memory 相关的方法：`recentMaterial(WithRefs)`、`daytimeMaterial(WithRefs)`、`thoughtMaterial(WithRefs)`、`digestMaterial`、`farMaterial`、`recentContinuityMaterial`/`handoffMaterial`、`memoryMap`、`memoryBucketPreview(s)`、`storeHeldOutput`、`writeSelfAwareness`、`traceHeldOutputSources`、`storeDream`。逐一检查是否接收/使用 `drives` 参数：

| 方法 | 是否接收 `drives` 参数 | 是否在方法体内实际使用 | 证据 |
|---|---|---|---|
| `recentMaterialWithRefs(drives, emotion)` | 是 | **是**——调用 `withDriveHint(base, drives)` 拼进 `query` 文本 | `ombre-client.js:91-99,368-376` |
| `daytimeMaterialWithRefs(drives, emotion, now)` | 是（形参保留） | **否**——方法体只用 `emotionArgs(emotion)`，不引用 `drives` | `ombre-client.js:109-119`，注释明确写"2026-09-07 改……驱力标签不再拼进 query，只保留情绪坐标做共振排序"（`ombre-client.js:105-108`） |
| `thoughtMaterialWithRefs(drives, emotion, now)` | 是（形参保留） | **否**——同上，方法体不引用 `drives` | `ombre-client.js:126-136` |
| `digestMaterial`/`farMaterial`/`recentContinuityMaterial` | 否 | 不适用 | `ombre-client.js:140-194` |
| `memoryMap`/`memoryBucketPreview(s)` | 否 | 不适用（读取 OB 结构化星表/预览，无 query） | `ombre-client.js:199-279` |
| `storeHeldOutput`/`writeSelfAwareness`/`traceHeldOutputSources`/`storeDream` | 否 | 不适用（写入路径，不含 drive 字段） | `ombre-client.js:282-350` |

**已确认**：`server.js` 对三个 `*MaterialWithRefs` 方法的调用点（`server.js:270,361,450`）全部传入 `topDrives(state)` 作为第一个参数，形参名一致为 `drives`，但只有 `recentMaterialWithRefs` 真正把它用起来；`daytimeMaterialWithRefs`/`thoughtMaterialWithRefs` 接收了这个参数却在方法体内完全忽略——**这是一处"形参存在但被架空"的真实代码事实，2026-09-07 的一次改动主动移除了后两者的 drive-hint 拼接，只保留了`recentMaterialWithRefs`一条**（`ombre-client.js:105-108` 注释原文自陈此意图）。

### 2.2 `withDriveHint` 机制细节（3.3.6 版本重新核实，未套用旧仓库结论）

```js
// ombre-client.js:368-376
function withDriveHint(base, drives) {
  const labels = (Array.isArray(drives) ? drives : [])
    .filter((item) => Number(item?.value) >= DRIVE_HINT_MIN)   // 0.5
    .slice(0, DRIVE_HINT_MAX_LABELS)                            // 最多3个
    .map((item) => String(item?.label ?? '').trim())
    .filter(Boolean);
  if (!labels.length) return base;
  return `${base}。此刻最强的内在状态是${labels.join('、')}，优先浮现与之真正相关的具体记忆；没有直接相关的就照常返回近期重要的`;
}
```
- 输入：`topDrives(state)` 返回的 `{key,label,value}` 数组（`engine.js:864-869`，按值降序取前 5）。
- 输出：纯**自然语言字符串**，拼进 `breath` 工具调用的 `query` 参数（`ombre-client.js:94`）。
- **唯一调用点**：`server.js:270`，位于"做梦分支"（`dreamAllowed` 判定为真、即仅在睡眠中且满足梦的间隔条件时）内部，且只在 `digestMaterial(48)`（按时间窗口取"最近48小时消化中的记忆"，与 drive 无关）**返回空**时才作为退化兜底调用：
```js
// server.js:264-273
if (!config.shadowMode && config.ombre.readEnabled) {
  const digest = await ombre.digestMaterial(48);
  if (digest.text) { material = digest.text; ... }
  else {
    const recalled = await ombre.recentMaterialWithRefs(topDrives(state), emotionForOmbre(state));
    ...
  }
}
```
**已确认**：全仓库 `grep -rn "recentMaterialWithRefs\|withDriveHint"` 只命中上述三处（定义+唯一调用），没有第二个调用入口。

### 2.3 A. 12维 Drive 是否真的改变 Memory Recall？

**明确结论（已确认）：不是结构化改变，是"翻译成自然语言词塞进查询文本"，且这条路径本身只在一种边缘退化场景下才会触发，心潮念一侧对 Haven 排序结果没有任何结构化/数值化的控制力。**

具体分解：
1. **文本注入路径**（`withDriveHint`）：把最强的 1-3 个驱力标签拼成一句中文提示词追加到 `query` 字符串里，随后整个 `query` 文本被当作 Haven `breath` 工具的普通检索词处理（见第3节，走 `recall_search_query`→BM25 lexical 打分 + `embedding_engine.search_similar` 语义检索两条通道）。Haven 侧**没有**任何代码把这句"此刻最强的内在状态是……"识别为特殊的"驱力信号"并单独加权——它和用户手打的任何一句查询在 Haven 眼里完全等价，只是多了几个中文词，靠这几个词本身能不能被 BM25/embedding 命中相关记忆。**心潮念自己不做排序计算，排序权重、阈值、准入判定全部发生在 Haven 一侧**（`recall_policy.py`/`memory_relevance.py`/`bucket_manager.py`，第3节详述）。
2. **数值/坐标注入路径**（`emotionArgs`）：`ombre-client.js:360-366` 把 `emotion.valence/arousal`（**不是 drive**，是 emotion.js 的独立2维状态）作为结构化参数（非文本）传给 `breath`/`breath_advanced`。这条路径**确实**在 Haven 侧被消费为量化排序因子（第2.5节详述），但输入源头是 `emotion.js` 的 `state.emotion`，不是 `state.drives`。12 维驱力要影响这条路径，唯一途径是通过 `DRIVE_PULL`（grieve/anger 两维，见1.4节）先改变 emotion 目标点，再经半衰期滞后改变 emotion.valence/arousal，是**间接、有滞后、且只覆盖 12 维中 2 维**的影响。
3. **调用触发条件极窄**：`withDriveHint` 唯一生效路径挂在"睡眠中的做梦材料退化分支"下，且需要 `shadowMode=false && ombre.readEnabled=true`（默认值分别是 `true`/`false`，即默认必须手动改两个环境变量才会触发，见1.9节表）。

综上，**回答任务书给出的二选一问题**：属于后者——"Drive → 生成自然语言 hint → 拼到 breath query → Ombre 自己处理"，心潮念一侧完全不参与排序计算，排序发生在 Haven/Ombre 一侧，心潮念对排序结果没有可观测/可控的控制力；而且这条唯一的文本路径本身也只覆盖 1 个方法（`recentMaterialWithRefs`）里的 1 个调用点（`server.js:270`），另外两个理应也用 drive 的方法（`daytimeMaterialWithRefs`/`thoughtMaterialWithRefs`）已在 2026-09-07 被主动移除了 drive-hint 拼接。**状态：已确认。**

### 2.4 B. `DOMAIN_AFFINITY` 是否真的参与 Memory Recall？方向判定

**明确结论（已确认，方向为 Memory → Domain → Drive，不是 Drive → Domain → Memory）：**

逐行追踪调用链：

1. Haven `breath`/`breath_advanced` 返回的记忆文本中，每条浮现桶表头带 `[domain:...]` 元数据（Haven 侧写入，见 `ombre-client.js:381-394` 的解析器注释"OB 2.6.5+（breath-meta）在表头带 domain/tags"）。
2. `parseSurfacedDomains(text)`（`ombre-client.js:383-394`）用正则从**已经召回的记忆文本**里提取 domain 标签数组。
3. 该数组被传入两处消费函数：
   - `surfacedDriveKey(domains, state)`（`engine.js:256-267`）：按 `DOMAIN_AFFINITY[domain]` 找最强亲和的驱力 key，用于"这条浮现的记忆该落进念头池的哪一维"（`server.js:467`，白天浮现分支）。
   - `applyMemoryResonance(state, domains, now, options)`（`engine.js:750-781`）：对命中的所有 domain 取每维最大亲和度（不累加），乘以 `nudge`（默认0.02，`config.js:177`）、封顶 `perCallCap`（默认0.06，`config.js:178`），**加到** `state.drives[key]` 上。
4. `server.js` 中的两个真实调用点：
   - `server.js:367-375`（自主念头分支）：`const domains = parseSurfacedDomains(thoughtMaterial); ... applyMemoryResonance(latest, domains, now, config.resonance)`。
   - `server.js:453-460`（白天浮现分支）：同样先 `parseSurfacedDomains(material)` 再 `applyMemoryResonance`。

**数据流向确定为**：`Haven breath 返回的记忆文本（含Haven自己打的domain标签）` → `parseSurfacedDomains`（提取） → `DOMAIN_AFFINITY 查表` → `state.drives[key] += nudge×affinity`（写回驱力）。**这是"想起什么影响想要什么"（Memory→Domain→Drive），方向与任务书重点提示的"极易被弄反"完全一致——真实方向是记忆→驱力，不是驱力→记忆。**

**反向路径（Drive→Domain→Memory）是否存在**：逐一核对三个 `*MaterialWithRefs` 方法调用 Haven `breath`/`breath_advanced` 时传入的参数（`ombre-client.js:91-99,109-119,126-136`），均只有 `query`（或省略）、`valence`/`arousal`、`date_from`、`mode`、`max_results`、`max_tokens`、`with_ids`，**没有任何一处传入 `domain` 参数**。也没有任何代码从 `DOMAIN_AFFINITY` 表反查"当前最强驱力对应哪些 domain"再把这些 domain 塞进 Haven 检索的 `domain_filter` 参数。**全仓库 `grep "DOMAIN_AFFINITY"` 只命中 `dimensions.js` 定义处和 `engine.js:259,758` 两个消费处（均为 Memory→Drive 方向），不存在反向消费点。状态：已确认（不存在）。**

### 2.5 C. 两条路线分别的作用（分开描述，不合并）

**路线一：`withDriveHint`（Drive → 自然语言 query hint）**
- 输入：`topDrives(state)`（12维驱力中值≥0.5的前3个）。
- 输出：追加到 `query` 字符串的一句中文提示。
- 触发时机：仅"睡眠中 + 做梦材料退化兜底"这一狭窄场景（`server.js:270`）。
- 是否触达 Haven 排序逻辑：**触达，但作为普通文本参与 BM25/embedding 匹配，无专门的"drive 感知"加权**（第3节详述）。
- 方向：Drive→Memory（查询侧输入）。

**路线二：`DOMAIN_AFFINITY`（记忆共振）**
- 输入：`parseSurfacedDomains`（从**已经召回**的记忆文本里提取 Haven 自己打的 domain 标签）。
- 输出：写回 `state.drives`（结构化数值加法，非文本）。
- 触发时机：自主念头分支（`server.js:367-375`）+ 白天浮现分支（`server.js:453-460`），且白天浮现默认关闭（`DAYTIME_EMERGENCE_ENABLED=false`），自主念头分支需 `bark.enabled`（默认false）。
- 是否触达 Haven 排序逻辑：**完全不触达**——它读取的是 Haven 已经返回的结果，不向 Haven 传回任何参数，是纯粹的单向"结果→内部状态"管道。
- 方向：Memory→Drive（结果侧输出，不影响任何后续排序）。

**结论**：两条路线**方向相反、机制完全不同、互不复用**——路线一是"检索前，把状态翻译成文字改查询"；路线二是"检索后，把结果的分类标签翻译回状态数值"。合并描述会误导对因果方向的判断，本报告严格分列。

---

## 3. Haven Recall 实际调用链

### 3.1 Haven 当前 Recall 使用哪些信号（逐一给出代码证据）

存在**两个物理上独立的检索/候选生成路径**，必须分开看：

**路径 A：`bucket_manager.py: BucketManager.search()`（`bucket_manager.py:712-800`），被 `server.py` 的 MCP 工具（`breath`/`breath_advanced` 等）调用，`bucket_manager.py` 文件头注释自陈 "Depended on by: server.py, decay_engine.py"（`bucket_manager.py:24-25`），未被 `gateway.py` 引用（`grep "bucket_mgr.search(" gateway.py` 命中 0 处，`grep ... server.py` 命中 6 处：`server.py:3596,4037,6053,6570,7399,10984`）。**

其内部四维加权打分公式（`bucket_manager.py:702-800`）：

| 信号 | 子分函数 | 权重(默认) | 证据 |
|---|---|---|---|
| topic（词法/BM25） | `calc_topic_scores`（rapidfuzz 加权字段匹配 name×3/domain×2.5/tags×2/body） | `w_topic=4.0` | `bucket_manager.py:754,761,808-,101` |
| emotion（情感共鸣） | `_calc_emotion_score`：Russell circumplex 欧氏距离，`1-dist/1.414`；查询无坐标时给中性0.5 | `w_emotion=2.0` | `bucket_manager.py:764-766,1207-1225,102` |
| time（时间新近度） | `_calc_time_score`：`exp(-0.02×days_since)` | `w_time=1.5` | `bucket_manager.py:768-769,1232-1244,103` |
| importance（重要度） | 直接归一化 `importance/10` | `w_importance=1.0` | `bucket_manager.py:772,104` |

四项加权求和归一化到0-100，`resolved=true` 的桶打0.3折（`bucket_manager.py:774-790`）。**已确认。**

**路径 B：`gateway.py` 的自动注入候选选择（`_select_dynamic_moments`/`_select_dynamic_buckets`，第5节详述），依赖 `memory_relevance.py`（facet匹配、`recall_rank`）、`recall_policy.py`（准入判定 `RecallPolicy.assess`）、`memory_diffusion.py`（关系图扩散）、`reranker_engine.py`（重排）、以及独立的 embedding 语义救援（`_try_semantic_rescue`），**不调用 `bucket_manager.search()`**。**已确认（`bucket_mgr.search(` 在 gateway.py 中零命中）。**

路径 B 内可确认的信号：

| 信号 | 证据 |
|---|---|
| lexical/词法别名匹配 | `_retrieval_alias_hits`（`gateway.py:15402-15413`），`memory_relevance.py` 的 `_query_terms`/`facets_for_text`/`_alias_match_score` |
| semantic（向量语义） | `_try_semantic_rescue`（`gateway.py:15869-15875`，供不足名额时补一条语义候选）；`embedding_engine.search_similar` |
| relation（关系图/记忆共振扩散） | `memory_diffusion.py: diffuse_memory`（`memory_diffusion.py:254`）+ `_seed_score`/`_node_salience`（`memory_diffusion.py:503,523`），被 `related_memory`/"联想浮现"消费 |
| facet 分类（embodiment/intimacy/hardware_protocol/communication_action/career/relationship_identity...） | `recall_rank`（`memory_relevance.py:1044-1087`）：facet 命中决定候选的排序层级(tier)，同层再按 `-score` 排 |
| 准入门槛（词锚/topic evidence，而非打分） | `RecallPolicy.assess`（`recall_policy.py:2731-2861`）、`node_has_topic_evidence`（`recall_policy.py:2693`） |
| rerank（重排） | `reranker_engine.py: RerankerEngine`（`reranker_engine.py:18-88`），`_rerank_breath_moment_candidates`（`server.py:5499`，路径A侧也会用到） |

**路径B中未发现 valence/arousal 或 emotion 参数**：`_select_dynamic_moments`/`_select_dynamic_buckets`/`_dynamic_bucket_candidate_items` 三个核心函数签名（`gateway.py:9931-9943,15812-15824,15333-15348`）均**不含**任何 valence/arousal/emotion/drive 形参。**状态：已确认（缺失）。**

**衰减/鲜活度信号（独立于上述两条检索路径，作用于桶的"是否还活跃/该不该归档"，不是每次检索的排序打分）**：`decay_engine.py: DecayEngine.calculate_score`（`decay_engine.py:98-183`）用桶自身固有的 `arousal`（写入时的情感元数据，非"此刻AI状态"）计算衰减分，短期(≤3天)时间权重占70%、长期(>3天)情感权重占70%（`decay_engine.py:146-162`）。这个分数被 `breath` 无参数"浮现模式"用于排序谁先浮现（`server.py:7235-7246`），但**它用的是记忆自身的历史 arousal 元数据，不是查询时传入的当前 valence/arousal**，两者是不同的字段来源，容易混淆，本报告明确区分。

### 3.2 当前 Recall 是否已经存在"当前状态影响召回"的入口？

**分两条路径分别回答，不能笼统说"有"或"没有"：**

- **路径 A（`bucket_manager.search`，被 MCP 工具 `breath`/`breath_advanced` 使用）：存在，且是量化的、可验证的。** 完整调用链：`server.py:breath(valence=-1, arousal=-1, ...)`（形参，`server.py:7083-7103`）→ 若调用方传入 0-1 范围内的值则 `q_valence`/`q_arousal` 非空（`server.py:7392-7393`）→ `bucket_mgr.search(search_query, ..., query_valence=q_valence, query_arousal=q_arousal)`（`server.py:7399-7405`）→ `_calc_emotion_score(query_valence, query_arousal, meta)`（`bucket_manager.py:764-766,1207-1225`）→ 按 `w_emotion=2.0` 计入加权总分（`bucket_manager.py:776-783`）→ `scored.sort(key=lambda x: x["score"], reverse=True)`（`bucket_manager.py:799`）。**这是一条从"调用方传入的当前情绪坐标"到"实际排序输出"完整可执行、无需假设的真实链路。已确认。**
  - 但这个"当前状态"是**调用方显式传入的 valence/arousal 参数**，Haven 自己不维护/不读取任何"AI当前情绪"的全局状态；对心潮念而言，传入值来自 `emotion.js` 的 `state.emotion`（见2.5节），不是 12 维驱力本身。
- **路径 B（`gateway.py` 自动注入候选选择）：不存在。** `_select_dynamic_moments`/`_select_dynamic_buckets`/`_dynamic_bucket_candidate_items` 的函数签名和调用点均无 valence/arousal/emotion/drive 输入（`gateway.py:9931-9943,15812-15824,15333-15348`；调用点 `gateway.py:2903-2912,2956-2967` 只传 `current_user_query`/`session_id`/`all_buckets`/`search_query`）。**明确写：当前不存在，不是"技术上可以加所以算存在"，代码层面确实没有这个输入通道。已确认（缺失）。**

### 3.3 当前自动注入在哪里发生？完整链路还原

**这是 `gateway.py` 的 HTTP 代理层（不是 `server.py` 的 MCP 工具层），每次经过网关的 `/v1/chat/completions` 请求都会自动触发，不需要 AI 显式调用任何"记忆工具"。** 完整链路（均在 `gateway.py`，函数级别）：

```
1. query（当前用户消息文本 current_user_query）
2. all_buckets = 列出全部桶（供后续多处过滤/召回复用）
3. recall（候选生成，路径B，见3.1）：
   - retrieval_mode=="bucket"：
     _select_dynamic_buckets(query, session_id, all_buckets, search_query=...)
       → _dynamic_bucket_candidate_items(...)（词法别名匹配 + RecallPolicy 准入 + 语义候选池）
       → _pick_dynamic_cards(active_pool, query=query)（挑卡）
       → _try_semantic_rescue(...)（名额不足时的语义救援补位）
     [gateway.py:2901-2944, 15812-15875, 15333-]
   - 否则（moment 图模式）：
     _refresh_moment_graph(all_buckets) → _select_dynamic_moments(...) [gateway.py:2946-2968, 9931-]
4. ranking：candidate items 内部已含 facet tier / rerank(RerankerEngine) / diffusion 相似度等排序（第3.1节各信号），
   在 _dynamic_bucket_candidate_items 与 _select_dynamic_moments 内部完成，不额外有全局统一的"最终排序"函数
5. selection：
   recalled_memory = await self._format_recalled_moments(recalled_moments, grouped_moments, all_buckets,
                        self.recalled_budget, current_user_query, context_mode=...) [gateway.py:2973-]
   related_memory（diffusion 联想）、favorite_memory、date_recall、dream_context、recent_context 等并行拼装
6. injection（组装注入文本）：
   stable_context, dynamic_context = self._build_injected_context_messages(persona_block, core_memory,
        portrait_memory, ..., recalled_memory=recalled_memory, related_memory=related_memory, ...)
   [gateway.py:3158-3177, 函数体 17859-17983]
   → self._inject_context_messages(...) 把 dynamic_context/stable_context 塞进转发给上游模型的 messages
   [gateway.py:19720-]
7. context（转发给上游 LLM 的最终请求体）：
   forward_payload["messages"] = messages_for_forward（已被注入）[gateway.py:3181-3197]
8. 落盘记录（生产接线的最终节点）：
   round_id = self.state_store.record_success(session_id, recalled_ids)   # 写入 injected_buckets 表
   if injection_debug.get("recent_context_injected"):
       self.state_store.record_recent_context_injection(session_id, round_id)
   self.state_store.record_injection_debug(session_id, round_id, injection_debug)  # 写入 injection_debug 表
   [gateway.py:3875-3944; 表结构定义于 gateway_state.py:24-70]
```

**已确认**：这条链路的每一步都能在 `gateway.py` 中定位到具体函数与行号；`injected_buckets`/`request_rounds`/`injection_debug` 三张表（`gateway_state.py:24-70`）是这条链路的落盘终点，与01/02/03号既有审计报告"`injected_buckets` 是 DELIVERED 事件落点"的结论一致（本次独立复核确认一致，未发现文档/代码差异）。

**关键区分（对第4节至关重要）**：`breath`/`breath_advanced`（路径A，MCP 工具，`server.py`）与 `gateway.py` 自动注入（路径B）是**两套完全独立的候选生成代码**，共享同一个底层数据（`bucket_mgr`/桶文件），但排序算法、信号来源、调用入口互不复用。**心潮念的 `OmbreClient` 走的是路径A（MCP `tools/call`，`ombre-client.js:55-67`），不经过 `gateway.py` 的自动注入管线**——这意味着心潮念当前对 Memory 的所有访问（无论 drive hint 还是 emotion 坐标）都只触达路径A，与任务书重点关注的"自动注入"（路径B）**在当前代码里没有任何交集**。

---

## 4. 寻找最小整合点（本次最重要的部分）

基于以上事实，逐候选评估：

### 候选1：Drive State → Recall Query Modifier → 现有 Haven Recall（查询文本阶段注入）

- **现有挂载点**：路径A：`server.py:breath()` 的 `query` 参数（`server.py:7083-7104`），本身已经是自由文本，心潮念的 `withDriveHint` 已经在用这个口子（只是覆盖面窄、默认关闭）。路径B：`gateway.py` 的 `current_user_query` 来自用户真实聊天消息，**没有额外的"系统侧查询修饰"入口**——`_dynamic_recall_search_query(current_user_query, memory_sentinel_debug)`（`gateway.py:2907-2910,2962-2965`）是唯一对 query 做二次加工的函数，其输入只有原始用户消息和"记忆哨兵"判定结果，没有可插入 drive 文本的参数位。
- **最小改动范围（仅描述，不实施）**：路径A 无需改 Haven 任何代码，扩大心潮念自己 `withDriveHint` 的调用面即可（例如在 `daytimeMaterialWithRefs`/`thoughtMaterialWithRefs` 里恢复调用），**这条完全不触碰 Haven 代码，不违反红线**。路径B 若要接入，需要在 `_dynamic_recall_search_query` 或其调用点新增一个可选形参（如 `state_hint: str`）拼进 `search_query`——**这会修改 `gateway.py`（属于"修改 Recall"红线范围内），本次审计只分析不实施**。
- **是否触碰红线**：路径A方案**不触碰**（只改心潮念自己的调用面，Haven零改动）；路径B方案**触碰**"禁止修改 Haven Recall"红线。

### 候选2：Drive State → Recall Weight Modifier → 现有 Ranking（排序打分阶段新增权重因子）

- **现有挂载点**：路径A：`bucket_manager.py:search()` 已经有现成的 `query_valence`/`query_arousal` 结构化参数位和 `w_emotion` 权重槽（`bucket_manager.py:712-800,1207-1225`），**这是唯一一个"Haven 已经预留了结构化情感状态输入位"的真实挂载点**。若要把 drive 接进来，技术上最小的路径是：不新增字段，而是让心潮念把 12 维驱力**折算进已有的 valence/arousal 两个坐标**（例如用 `emotionTarget` 式的加权映射），复用现有 `_calc_emotion_score` 通道——**这不需要修改 Haven 任何一行代码，只需要心潮念自己扩大 `DRIVE_PULL` 覆盖的驱力数量（目前只有 grieve/anger 两维），或在 `emotionForOmbre()` 里额外叠加更多驱力的贡献**。
- 路径B：`_select_dynamic_moments`/`_select_dynamic_buckets` 内部没有统一的"总分"变量可插入权重项（排序逻辑分散在 facet tier、rerank score、diffusion similarity 等多处，不是像路径A那样的单一加权公式），若要加一个 drive 权重因子，需要在 `RerankerEngine`（`reranker_engine.py:18-88`）或 `recall_rank`（`memory_relevance.py:1044-1087`）里新增一个乘子——**两处都需要修改 Haven 生产代码，触碰红线**。
- **是否触碰红线**：借道 `emotionForOmbre`/`DRIVE_PULL` 扩大覆盖面的方案**不触碰**（纯心潮念侧改动）；直接改 `bucket_manager.py`/`reranker_engine.py` 新增字段的方案**触碰**红线。

### 候选3：Drive State → Memory Domain / Bucket Affinity → 现有 Recall（通过 domain_filter 映射）

- **现有挂载点**：`bucket_manager.py:search()` 的 `domain_filter` 参数（`bucket_manager.py:712-750`，第一层预筛）以及 `breath()` 的 `domain` 形参（`server.py:7086,7120,7141,7148,7153,7199`）。这是一个**真实存在、已经生产可用的挂载点**——`DOMAIN_AFFINITY` 表本身的 key（恋爱/亲密/成长/内心……）与 Haven 真实 domain taxonomy 已经做过对齐（`dimensions.js:118-122` 注释自陈"对齐 OB 真实桶 taxonomy"）。
- **最小改动范围**：心潮念侧新增一个"反查"函数——给定当前最强的 1-3 个驱力 key，反向在 `DOMAIN_AFFINITY` 表里找出亲和度最高的 domain 集合，作为 `domain` 参数传给 `breath`/`breath_advanced` 调用。**这完全不需要修改 Haven 任何代码**（`domain` 参数早已存在且被消费，`server.py:7391,7402`/`bucket_manager.py:739-748`），只需要心潮念自己新增一个纯函数+ 在 `ombre-client.js` 的调用点多传一个参数。**这是本报告认为红线约束下最干净、改动面最小、且完全不触碰 Haven 生产代码的候选。**
- **限制**：仅对路径A（MCP `breath` 工具）有效；路径B（`gateway.py` 自动注入）的候选筛选函数不接受 `domain` 级别的外部提示（`_select_dynamic_moments`/`_select_dynamic_buckets` 无此形参），要接入路径B 仍然需要改 `gateway.py`，触碰红线。
- **是否触碰红线**：接入路径A **不触碰**；接入路径B **触碰**。

### 第四种可能（本次审计过程中发现，供参考）

`digestMaterial`/`daytimeMaterialWithRefs`/`thoughtMaterialWithRefs` 目前统一改走 `breath_advanced` 的**"浮现道"**（不传 `query`，只传 `date_from`+`mode=automatic`，`ombre-client.js:105-119`），这是 Haven 侧"权重池主动推送"模式（`server.py:7211-7218` 无参数浮现模式，用 `decay_engine.calculate_score` 排序，见3.1节）。**若真的要做"最小整合"，比起改查询文本或新增排序权重，更贴近现有代码习惯的做法可能是：把 12 维驱力折算成一个 0-1 的"总体唤醒度"标量，映射到 `breath_advanced` 的 `arousal` 参数**（该函数已支持 `emotionArgs`，只是当前心潮念把它绑定给 `emotion.js` 而非 `state.drives`）。这仍然落在候选2的范畴内，只是提示"复用现有 emotion 坐标通道"比"新增 domain 反查"更省一层间接性，但两者都不触碰 Haven 代码——因此本报告认为**候选2（借道 emotion 坐标扩大 DRIVE_PULL 覆盖面）与候选3（domain 反查）是本次识别出的两个真正意义上的"零 Haven 改动"整合点，二者可以并行采用，互不冲突**。

---

## 5. Haven Auto-Injection 实际调用链

见第3.3节完整还原，此处不重复。**核心结论重申**：自动注入（`gateway.py`）与 MCP 记忆工具（`server.py` 的 `breath`）是两套独立代码，共享底层桶数据但排序逻辑不共享；`bucket_manager.search()` 里唯一确认存在的"当前状态→排序"接口（`query_valence`/`query_arousal`）只服务于后者，不服务于自动注入。

---

## 6. 两套系统的交叉点（现状：哪里真的连着，哪里看着连实际没连）

| 交叉点 | 现状 | 证据 |
|---|---|---|
| 心潮念 `emotionArgs(emotion)` → Haven `breath(valence, arousal)` → `bucket_manager._calc_emotion_score` | **真的连着**，量化、可验证 | `ombre-client.js:360-366`；`server.js:31-33`；`server.py:7392-7404`；`bucket_manager.py:764-766,1207-1225` |
| 心潮念 `withDriveHint` → Haven `breath(query=...)` | **连着，但只是普通文本进入词法/语义匹配，无专属加权** | `ombre-client.js:91-99,368-376`；`server.py:7396-7448` |
| 心潮念 `DOMAIN_AFFINITY` → Haven Recall（假设方向） | **看着连，实际方向相反**——是 Memory→Domain→Drive，不是 Drive→Domain→Memory | `engine.js:256-267,750-781`；无反向消费点（`domain` 参数从未被心潮念传给 Haven） |
| 心潮念 12 维驱力（除 grieve/anger）→ Haven 任何排序信号 | **不连**——没有任何字段/参数传递路径 | 全仓库检索无对应调用 |
| 心潮念（任何形式）→ Haven `gateway.py` 自动注入管线 | **完全不连**——心潮念的 `OmbreClient` 走 MCP `tools/call`（路径A），从不经过 `gateway.py` 的 `/v1/chat/completions` 代理（路径B） | `ombre-client.js:13-67`（MCP JSON-RPC 调用） vs `gateway.py:2700-3200`（HTTP 代理管线），两者是不同的服务/端口，无代码交叉引用 |
| Haven `bucket_manager.py` 的 domain 预筛 ↔ 心潮念 `DOMAIN_AFFINITY` 的 domain 命名 | **命名对齐但无调用连接**——只是两边人工各自维护了相似的中文域名词表，代码层面互不引用 | `dimensions.js:118-158`；`bucket_manager.py:739-748` |

---

## 7. 已存在能力

1. Haven `bucket_manager.search()` 已有结构化的 `query_valence`/`query_arousal` 输入位与对应的 `_calc_emotion_score` 加权公式，权重 `w_emotion=2.0`（仅次于 `w_topic=4.0`）——**这是唯一现成的"当前状态→排序"生产级接口**。（已确认）
2. Haven `breath()`/`bucket_manager.search()` 已有 `domain`/`domain_filter` 预筛参数，且心潮念的 `DOMAIN_AFFINITY` 域名表已经人工对齐过 Haven 真实 taxonomy。（已确认）
3. 心潮念 `emotion.js` 已有 `DRIVE_PULL`/`emotionTarget` 机制，证明"多维驱力折算成2维情感坐标"这条技术路径在心潮念内部已经落地过一次（只是目前只覆盖 grieve/anger 两维）。（已确认）
4. Haven `decay_engine.py` 已有基于桶自身 `arousal` 元数据的鲜活度加权，证明"情感强度影响记忆权重"这一设计模式在 Haven 侧也独立存在（但作用对象是记忆自身的历史情感标签，不是查询时的当前状态）。（已确认）

## 8. 明确缺口

1. Haven `gateway.py` 自动注入管线（生产环境每轮聊天真正触发、`injected_buckets` 表的写入来源）**完全没有**任何当前状态/情感/驱力输入接口——不是没做完，是设计上从未规划这个输入维度。（已确认）
2. 心潮念侧 `daytimeMaterialWithRefs`/`thoughtMaterialWithRefs` 已经在 2026-09-07 主动移除了 drive-hint 文本拼接，只剩 `recentMaterialWithRefs` 一条路径且只在做梦材料退化分支触发——**12维驱力事实上已经在心潮念自己的迭代中被"降级"为几乎不参与检索输入**。（已确认）
3. 不存在任何"Drive → domain_filter"的反查/映射代码——`DOMAIN_AFFINITY` 表虽然结构上双向可用（key 是 domain，value 是驱力亲和度，理论上可以反查），但代码里只有一个方向的消费函数。（已确认）
4. Haven 路径B（自动注入）的排序逻辑分散在多个函数（facet tier、rerank、diffusion similarity），不像路径A那样有统一的加权公式变量，"加一个权重因子"没有单一的、低风险的挂载点。（已确认，属于"技术复杂度"而非"缺失"，但直接影响候选2在路径B下的可行性判断）

---

## 9. 最小整合点候选（不预设答案，基于代码判断）

综合第4节的逐候选分析，本报告给出的判断（非实施建议，仅技术可行性排序）：

- **候选3（Drive→DOMAIN_AFFINITY反查→`breath`的`domain`参数）**：**技术上最小、最不触碰红线**。理由：`domain` 参数在 Haven `server.py`/`bucket_manager.py` 两侧均已是生产可用、有实际消费逻辑的现成入口；心潮念侧只需新增一个纯函数（反查 `DOMAIN_AFFINITY`）+ 在 `ombre-client.js` 调用点多传一个字段，**Haven 零改动**。局限：只对 MCP `breath` 工具路径（路径A）有效，不触及自动注入（路径B，即任务书更关心的"自动注入"本身）。
- **候选2 的心潮念侧子方案（扩大 `DRIVE_PULL`/`emotionForOmbre` 覆盖面）**：**次优**，同样 Haven 零改动，复用已验证有效的 `query_valence`/`query_arousal` 通道，但只能把多维驱力"压缩"成2维坐标，天然有信息损失，且仍只覆盖路径A。
- **候选1/2/3 在路径B（Haven 自动注入）下的任何变体**：均需要修改 `gateway.py` 内部函数签名（新增形参）或 `bucket_manager.py`/`reranker_engine.py` 的打分逻辑，**全部触碰"禁止修改 Haven Recall/Injection"红线**，本报告不建议在当前阶段推进，仅记录为"技术上可行但需要走正式实施审批"的选项。
- **第四种可能**：本报告在梳理过程中未发现优于候选2/3的第四条路径；`gateway_state.py` 的 `injected_buckets`/`injection_debug` 表本身是纯记录表，不是可挂载输入的位置，不构成新的候选。

**结论**：若要求"Haven 零改动"作为硬约束，候选3（domain反查）是当前代码基础上唯一完整可行的最小整合点，但其作用范围被限定在 MCP `breath` 工具调用路径，**无法触及任务书最关心的"自动注入"（`gateway.py`）管线**——这是需要向总控明确汇报的现实限制，不应被掩盖。

---

## 10. 不应该整合的部分（技术/架构层面理由，不涉及产品决策）

1. **不建议把完整 12 维驱力原样映射进 Haven 排序公式**：`bucket_manager.search()` 的加权公式只有 4 个槽位（topic/emotion/time/importance），`w_emotion` 槽位本身设计语义是"情感共鸣"（2维 Russell 环形模型），不是"12维意图强度"；强行塞入会破坏 `_calc_emotion_score` 现有的欧氏距离几何语义（该函数假设输入是一个点在2维平面上的坐标，不是12维向量），技术上需要重新设计整个打分维度，不是"加个参数"能做到的最小改动。
2. **不建议把 `withDriveHint` 的自然语言拼接方式作为长期整合手段**：这种"翻译成文字塞进query"的方式没有可观测性（心潮念自己也无法验证 Haven 到底有没有真正被这几个词影响排序，因为 BM25/embedding 命中与否依赖具体记忆文本内容，是概率性的、不可预测的），且 2026-09-07 的改动已经证明心潮念团队自己在往回撤（3个方法里2个已移除该机制），继续依赖/扩大这条路径与代码演进方向相反。
3. **不建议在自动注入（`gateway.py`）管线里引入任何"当前状态"输入，除非先解决"状态归属"问题**：自动注入管线目前是**无状态、纯 query 驱动**的设计（`_select_dynamic_moments`/`_select_dynamic_buckets` 不持有任何跨请求的会话状态之外的输入），每次都是"这句话该不该召回什么记忆"的独立判断；心潮念的 12 维驱力是**跨会话、时间连续累积**的全局状态，把一个"全局持续状态"塞进一个"单次请求打分"的管线，语义上錯位（同一句话在驱力高/低两个时刻会被排出不同结果，但用户可能感知不到"为什么这次记忆不一样了"，可解释性和可调试性都会下降）——这是架构层面的不匹配，不是产品决策问题。
4. **不建议复用 `decay_engine.py` 的衰减机制承载"当前驱力"**：`decay_engine.calculate_score` 里的 `arousal` 是桶自身固有的历史元数据（写入时定的），语义是"这条记忆本身有多鲜活/多有情感冲击力"，与"AI此刻的驱力状态"是完全不同的两个概念，混用会让同一个字段承担双重语义，产生难以调试的耦合（例如：修改衰减公式的调参者可能不知道这个字段还被别的地方当"当前状态"读取）。

---

## 11. 证据表

| 结论 | 仓库 | 文件 | 函数 | 行号 | 证据 | 状态 |
|---|---|---|---|---|---|---|
| 12维驱力清单与参数 | xinchao-nian | xinchao/src/dimensions.js | `DIMENSIONS` | 12-109 | 见1.1节完整表 | 已确认 |
| 驱力初始值0.15 | xinchao-nian | xinchao/src/engine.js | `newState` | 279 | `drives: Object.fromEntries(DRIVE_KEYS.map((key) => [key, 0.15]))` | 已确认 |
| 心跳结算周期15分钟 | xinchao-nian | xinchao/src/config.js | `loadConfig` | 29 | `settleIntervalMinutes: number('SETTLE_INTERVAL_MINUTES', 15, ...)` | 已确认 |
| 驱力耦合表 | xinchao-nian | xinchao/src/engine.js | 常量 | 26-30 | `DRIVE_GROWTH_COUPLINGS` | 已确认（参见15号报告独立复核） |
| libido交叉抑制 | xinchao-nian | xinchao/src/dimensions.js | `DIMENSIONS.libido.inhibitedBy` | 50-54 | reflection/curiosity/boredom 抑制表 | 已确认 |
| Emotion→Drive增速调制 | xinchao-nian | xinchao/src/emotion.js | `EMOTION_GROWTH_MODULATION` | 225-236 | 参见15号报告 | 已确认（引用15号报告，未重新独立复核） |
| **Drive→Emotion目标点拉扯（仅grieve/anger）** | xinchao-nian | xinchao/src/emotion.js | `DRIVE_PULL`, `emotionTarget` | 53-56, 100-110 | 只有2维参与，其余10维无路径 | 已确认（本报告独立重新追踪） |
| settleEmotion消费drives | xinchao-nian | xinchao/src/emotion.js | `settleEmotion` | 148-161 | `emotionTarget(options.drives ?? state.drives ?? {})` | 已确认 |
| personality→drive偏置 | xinchao-nian | xinchao/src/personality-store.js | `CORE_TO_DRIVES` | 25-30 | 4组标签→7个驱力±10% | 推断（引用15号报告，本报告未重新读取全文） |
| thought-pool读取drive | xinchao-nian | xinchao/src/engine.js | `pickIntent` | 664-694 | 遍历`state.drives`+`obsessionBonus` | 已确认 |
| **autonomous wake读取drive** | xinchao-nian | xinchao/src/engine.js | `proactiveBarkAllowed` | 970-974 | `strongest = Math.max(...Object.values(state.drives))`, 阈值`minDrive`默认0.42 | 已确认 |
| pickIntent被调用 | xinchao-nian | xinchao/src/server.js | 内联 | 1449 | `const intent = pickIntent(state);` | 已确认 |
| Drive影响外部行为需多层开关 | xinchao-nian | xinchao/src/config.js | `loadConfig` | 31,50,51,128,159,167 | `shadowMode:true`, `ombre.readEnabled:false`, `ombre.writeEnabled:false`, `bark.enabled:false`, `daytime.enabled:false`, `daytime.bark:false` | 已确认 |
| **withDriveHint定义与唯一使用** | xinchao-nian | xinchao/src/ombre-client.js | `withDriveHint`, `recentMaterialWithRefs` | 368-376, 91-99 | 拼进`query`文本 | 已确认 |
| **daytimeMaterialWithRefs不再用drive** | xinchao-nian | xinchao/src/ombre-client.js | `daytimeMaterialWithRefs` | 105-119 | 注释明确"驱力标签不再拼进query" | 已确认 |
| **thoughtMaterialWithRefs不再用drive** | xinchao-nian | xinchao/src/ombre-client.js | `thoughtMaterialWithRefs` | 126-136 | 形参`drives`未在体内引用 | 已确认 |
| **withDriveHint唯一调用点** | xinchao-nian | xinchao/src/server.js | 做梦分支 | 264-273 | `digest.text`为空时才退化调用`recentMaterialWithRefs` | 已确认 |
| **emotionForOmbre只读emotion不读drive** | xinchao-nian | xinchao/src/server.js | `emotionForOmbre` | 31-33 | `emotionCoords(state)`，非`state.drives` | 已确认 |
| **A. Drive不改变Recall排序，只是文本注入** | xinchao-nian/haven-ombre | ombre-client.js + server.py | `withDriveHint` + `breath` | ombre-client.js:368-376; server.py:7396-7448 | query文本走普通BM25/embedding匹配，无drive专属加权 | 已确认 |
| **B. DOMAIN_AFFINITY方向为Memory→Domain→Drive** | xinchao-nian | xinchao/src/engine.js, ombre-client.js | `parseSurfacedDomains`, `applyMemoryResonance`, `surfacedDriveKey` | ombre-client.js:383-394; engine.js:256-267,750-781 | 输入是已召回文本，输出写回`state.drives`；无反向调用 | 已确认 |
| **DOMAIN_AFFINITY真实调用点** | xinchao-nian | xinchao/src/server.js | 自主念头/白天浮现分支 | 367-375, 453-460 | 两处均"先parseSurfacedDomains后applyMemoryResonance" | 已确认 |
| **C. 两条路线方向相反、机制不同** | xinchao-nian | 综合 | — | 见2.5节 | 路线一查询侧输入，路线二结果侧输出 | 已确认 |
| Haven路径A: bucket_manager 4维加权公式 | haven-ombre | bucket_manager.py | `search` | 712-800 | topic/emotion/time/importance | 已确认 |
| **emotion_score公式与权重** | haven-ombre | bucket_manager.py | `_calc_emotion_score`, `__init__` | 1207-1225, 101-104 | 欧氏距离；w_emotion=2.0 | 已确认 |
| **valence/arousal结构化传入breath** | haven-ombre | server.py | `breath` | 7083-7104, 7392-7405 | `q_valence`/`q_arousal`→`bucket_mgr.search` | 已确认 |
| bucket_manager仅被server.py/decay_engine依赖 | haven-ombre | bucket_manager.py | 文件头注释 | 24-25 | "Depended on by: server.py, decay_engine.py" | 已确认 |
| **gateway.py从不调用bucket_mgr.search** | haven-ombre | gateway.py | 全文 | — | `grep "bucket_mgr.search("`命中0处 | 已确认 |
| **自动注入路径B无valence/arousal输入** | haven-ombre | gateway.py | `_select_dynamic_moments`, `_select_dynamic_buckets`, `_dynamic_bucket_candidate_items` | 9931-9943, 15812-15824, 15333-15348 | 函数签名无emotion/drive形参 | 已确认 |
| 自动注入完整链路 | haven-ombre | gateway.py | 多函数 | 2700-3200(recall/format), 17859-17983(build_injected_context), 19720-(inject), 3875-3944(record) | 见3.3节完整还原 | 已确认 |
| injected_buckets表结构 | haven-ombre | gateway_state.py | `_init_db` | 24-70 | 与01/02/03号既有报告一致 | 已确认 |
| decay_engine用桶自身arousal非当前状态 | haven-ombre | decay_engine.py | `calculate_score` | 98-183 | `metadata.get("arousal", 0.3)`来自桶元数据 | 已确认 |
| domain_filter预筛已存在 | haven-ombre | bucket_manager.py | `search` | 737-750 | 第一层domain预筛 | 已确认 |
| breath的domain形参已被消费 | haven-ombre | server.py | `breath` | 7086,7120,7141,7148,7153,7391,7402 | domain分支处理+传入search | 已确认 |
| DOMAIN_AFFINITY表对齐Haven真实taxonomy | xinchao-nian | xinchao/src/dimensions.js | 注释 | 118-122 | 2026-08-08对齐说明 | 已确认（心潮念侧自陈，未独立核实Haven侧taxonomy当前是否仍一致） |
| 候选3（domain反查）Haven零改动可行 | 综合 | — | — | 见4节候选3、9节 | 基于以上确认证据推导 | 推断（可行性判断，非已实施） |
| 候选2（扩大DRIVE_PULL覆盖）Haven零改动可行 | 综合 | — | — | 见4节候选2、9节 | 基于以上确认证据推导 | 推断（可行性判断，非已实施） |
| 路径B任何整合方案均需改Haven生产代码 | haven-ombre | gateway.py, bucket_manager.py, reranker_engine.py | — | — | 见4节 | 推断（基于现有函数签名缺失该形参位的结构性判断） |

---

## 附：本次审计的自我审查记录（红线遵守声明）

本次审计**没有**修改 Haven Recall（`recall_policy.py`/`bucket_manager.py`/`memory_relevance.py`/`gateway.py` 等均只读）、**没有**修改 Injection（`gateway.py` 的 `_build_injected_context_messages`/`_inject_context_messages` 等均只读）、**没有**新建数据库、**没有**新建表、**没有**新建 embedding、**没有**新建 Memory Core、**没有**修改生产 Bucket、**没有**写生产代码——本次审计的唯一写操作是本报告文件本身（`16_dynamic_mind_memory_integration_audit.md`），未触碰 `phase0-1.5/` 目录下 00-15 号任何已有文件，未在 `xinchao-nian`/`haven-ombre` 两个克隆仓库内做任何 `git add/commit/checkout -b/clean` 或文件编辑/创建操作。

## 附：未验证/受限项说明

1. `personality-store.js` 的 `CORE_TO_DRIVES` 映射本报告未重新独立读取全文验证，采信15号报告结论，标注"推断"。
2. `bucket_manager.py` 文档中"2026-08-08对齐 OB 真实桶 taxonomy"的说法是心潮念侧单方面注释，本报告未逐一核对当前 Haven 生产环境实际使用的完整 domain 词表是否与 `DOMAIN_AFFINITY` 的24个中文域完全一致（只核实了代码机制存在，未核实数据层面的完全对齐度），标注为该条证据的限制。
3. `gateway.py`（21393行）/`server.py`（13653行）体量巨大，本报告采用"按函数签名+调用链定向追踪"的方式覆盖了与本任务直接相关的路径，未做逐行通读；不能排除这两个文件中存在其他与情感/驱力相关但命名不直观、未被本次检索词（valence/arousal/drive/emotion/domain）命中的代码路径，此为审计范围的固有局限，已尽力通过多组关键词交叉检索降低遗漏概率。
