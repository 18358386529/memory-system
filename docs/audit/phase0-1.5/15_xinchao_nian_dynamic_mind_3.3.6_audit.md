# 15. 心潮念动态心智（xinchao-nian 内嵌 xinchao/，3.3.6）源码深度审计

## 0. 审计模式声明

**本报告是源码审计，不是运行时审计。不进行实施、不部署、不迁移数据、不产出可直接使用的实现代码。** 所有结论均基于对本地已克隆文件的静态阅读（`Read`/`Grep`/`Bash` 只读命令），未启动任何服务，未修改任何文件。

### 锁定 commit 复核

```
$ git -C /home/user/18358386529/xinchao-nian rev-parse HEAD
97f1bdcc76b748fa516b7c79a0aa10234143795f
```

与任务书给定的锁定 commit **一致**。`xinchao/package.json` 确认：`"name": "xinchao-dynamic-mind"`，`"version": "3.3.6"`；`xinchao/src/version.js:3` 的 `SYSTEM_VERSION` 同为 `'3.3.6'`。审计对象路径：`/home/user/18358386529/xinchao-nian/xinchao/src/`（9 个目标文件均已确认存在于该目录，行数与任务书描述一致：engine.js 983 行、dimensions.js 158 行、thought-pool.js 114 行、emotion.js 354 行、personality-store.js 356 行、self-signals.js 236 行、black-box.js 146 行、awareness.js 205 行、cabin-store.js 208 行）。

以下所有文件路径若未特别说明，均相对于 `/home/user/18358386529/xinchao-nian/xinchao/src/`；证据格式为 `文件:行号` + commit `97f1bdc`。

---

## 1-9. 九个模块逐一深度解析

### 1. `dimensions.js`（158 行）—— 静态配置表，无副作用

**数据结构**：
- `SATURATE_CEIL=0.80`/`SATURATE_FLOOR=0.65`（`dimensions.js:1-2`）：全局默认天花板/地板，仅在某维未显式设置 `ceil` 时兜底（`engine.js:438` `Number.isFinite(dim.ceil) ? dim.ceil : SATURATE_CEIL`）。
- `DIMENSIONS`（`dimensions.js:12-109`）：12 维驱力的冻结对象，每维字段 `label`（中文名）、`growPerHour`（每小时自然增速，0~0.105）、`ceil`（静息天花板，0.15~0.82）、`satisfyMul`（被满足时的乘性衰减系数，0.15~0.70）、可选 `nightMul`（夜间增速倍率）、`dawnFreeze`（凌晨冻结布尔）、可选 `inhibitedBy`（交叉抑制表）、`decayHalfLifeHours`（仅 grieve/anger 两个"情绪型驱力"使用，24 小时半衰期，无增长项）。12 维为：possess/monitor/crave/share/libido/curiosity/boredom/social/duty/reflection/grieve/anger。
- `DRIVE_KEYS`（`dimensions.js:111`）：`Object.keys(DIMENSIONS)` 冻结数组，是全系统"合法驱力名单"的唯一真源，`engine.js`/`self-signals.js`/`awareness.js` 均从此导入做白名单校验。
- `DOMAIN_AFFINITY`（`dimensions.js:123-158`）：记忆域→驱力共振表，**任务书要求完整展开**——共 24 个中文域名键（恋爱/亲密/成长/内心/自省/心理/记忆/人际/社交/关系/家庭/身心/健康/饮食/兴趣/游戏/创作/购物/技术/数字/编程/事务/工作/约定/冲突/日常，实数 25 个），每个域映射到 2~5 个驱力的亲和度（0.2~0.9）。表头注释（`dimensions.js:118-122`）说明 2026-08-08 做过一次对齐真实 OB 桶目录的重构：此前只有 8 个域且与真实桶目录不匹配，现补齐至覆盖真实 taxonomy，同时保留"约定/冲突/技术"三个无对应真实目录但语义清晰的别名域。规则：多 domain 命中时每维**取最大值不累加**（`engine.js:262-263` `surfacedDriveKey`、`engine.js:760-763` `applyMemoryResonance`）。

**副作用**：无，纯常量导出，无 I/O、无随机性。

**调用方**：`engine.js:2`（`import { DIMENSIONS, DRIVE_KEYS, DOMAIN_AFFINITY, SATURATE_CEIL }`）、`awareness.js:17`（仅用 `DIMENSIONS` 取 label）、`self-signals.js:19`（仅用 `DRIVE_KEYS`）。`dimensions.js` 本身不 import 任何模块，是全系统依赖树的叶子/根节点。

---

### 2. `engine.js`（983 行）—— 状态机核心，纯函数式 reducer

**核心数据结构**（`state` 对象，由 `newState()` 定义，`engine.js:269-307`）：
- `schemaVersion`（数字，当前 9）、`revision`（自增整数，每次真实变更 +1，供并发/审计追踪）
- `consciousness`：`'awake' | 'sleeping'`
- `lastConversationAt`/`lastHeartbeatAt`/`lastSettledAt`/`sleepStartedAt`：ISO 时间戳
- `drives`：`{ [driveKey]: number(0~1) }`，12 维初始 0.15
- `thoughtPool`：见第 3 节
- `fatigue`：`number(0~0.3)`，睡眠恢复/清醒高驱力时累积
- `recentDreams`（≤20 条）、`dreamUsage`（按天计数）
- `barkUsage`/`recentBarkMessages`（≤8 条，去重用）
- `satisfactionPlateaus`：`{ [driveKey]: { until, reason } }`，满足后的"平台期"（该驱力在此期间增速强制为 0）
- `sessionOverlays`：`{ [sessionId]: { tone, warmth, tension, attention, confidence, expiresAt, ... } }`（≤64 个，按 `updatedAt` 排序截断）
- `interactionUsage`：按天计数的互动效果日限（≤14 天）
- `arrivalHistogram`：24 长度数组，学习"她"真实到访的小时分布（用于 `computeAnticipation`/`computeLonging`）
- `emotion`/`emotionJournal`/`emotionDays`：见第 4 节
- `awareness`/`recentSurfacings`：见第 8 节
- `grudge`（可选）：`{ cause, at }`，冲突事件记的"在气什么"（`engine.js:171-175`，`applyInteractionOutcome`）

**关键函数（输入/输出/副作用）**：

| 函数 | 行号 | 输入 | 输出/副作用 | 随机性 |
|---|---|---|---|---|
| `ensureStateShape` | 58-78 | state | 补全缺失字段，删除废弃字段 `pending`（第 66 行注释：3.3 起黑匣子接替） | 无 |
| `settleState`（心跳结算） | 387-503 | state, now, sleepAfterMinutes, options | **纯函数**，返回新 state（`structuredClone`），驱力按小时增长/衰减、情绪回落、思绪池 tick、疲劳、睡眠转换 | 无随机；但依赖 `now` |
| `applyConversationEvent` | 507-635 | state, event, now, options | 唤醒、写会话覆盖层、应用互动效果、情绪脉冲、驱力增量/满足、闪念、到访节律直方图 | 无 |
| `applyDreamWake` | 227-241 | state, dream, now | 醒来时按梦的心情打情绪脉冲 + 把梦的意象塞进念头池当闪念 | 无 |
| `pickIntent` | 664-694 | state, random（默认 `Math.random`） | 加权随机从并列最高驱力中选一个 | **有随机性**（`Math.random`，可注入） |
| `applyMemoryResonance` | 750-781 | state, domains, now, options | 记忆浮现→按 `DOMAIN_AFFINITY` 回推驱力，nudge 默认 0.02，`perCallCap` 默认 0.06 | 无 |
| `computeAnticipation`/`computeLonging` | 799-843 | state, now, options | **派生值，不落状态**（注释明确说明，`engine.js:797` "派生值，不落状态——用到时现算"），从 `arrivalHistogram` 现算 | 无 |
| `applyLongingNudge` | 847-860 | state, longing, now, options | 把挂念落进 `monitor` 驱力，硬顶在天花板内 | 无 |
| `dreamAllowed`/`barkAllowed`/`proactiveBarkAllowed`/`daytimeEmergenceAllowed` | 976-983/960-968/970-974/927-933 | state, now, 各种阈值 | 纯判定函数，不改状态，只读 | 无 |

任何写状态的函数都不直接调用外部服务或做网络 I/O——`engine.js` 完全不 import `fs`/`http`/任何 client，是纯粹的状态转移逻辑层，全部副作用（落盘、调用 OB/Bark）由 `server.js` 在拿到 `{state, ...}` 返回值后自行完成（见第 10 节）。

**关键常量含义**：
- `CEIL_RELAX_PER_HOUR=0.10`（`engine.js:11`）：驱力被顶过天花板后，每小时松弛回天花板的比例系数——`decay = (current-ceil)*0.10*elapsedHours`，即约 10 小时内松弛大半，不是瞬间归位。
- `THOUGHT_FEEDBACK_CAP=0.85`（`engine.js:12`）：3.3.6 新增，念头池回推驱力时若该驱力已 ≥0.85 则不再推（`engine.js:474-478`），防止闭环把某维钉死在 1.0（见 CHANGELOG 3.3.6 条目、`thought-pool.js` 注释）。
- `RESONANCE_MIN_AFFINITY=0.5`（`engine.js:14`）：记忆共振只推亲和度 ≥0.5 的维度。
- `ARRIVAL_GAP_MINUTES=90`/`ARRIVAL_DECAY=0.99`（`engine.js:16-17`）：到访节律学习的最小间隔（防止一次长聊被计成多次到访）与历史衰减率（每次新到访前旧直方图整体 ×0.99）。
- `DRIVE_GROWTH_COUPLINGS`（`engine.js:26-30`）：小型耦合表——`anger` 越高，`possess` 增速越慢（slope=-0.65）；`grieve` 越高，`crave`/`monitor` 增速越快（slope=+0.50/+0.45）。注释明确：**只改"接下来长多快"（rate），不直接叠加数值**（`engine.js:24-25`），所以提高结算频率不会自激。

**调用方**：`server.js` 大量导入（`server.js:8`），是全系统状态变更的唯一合法入口；`self-signals.js:20` 导入 `computeLonging`/`localDayAndHour`/`topDrives`；`awareness.js` 不直接依赖 `engine.js`（只依赖 `dimensions.js`）。

---

### 3. `thought-pool.js`（114 行）—— 念头池：闪念→执念的双层结构

**任务书提醒的常量核对（以本文件实际读到的为准）**：
```
thought-pool.js:1  const FLASH_DECAY       = 0.82;   // 默认：每次结算（15min）×0.82，约 50 分钟半衰
thought-pool.js:3  export const SURFACED_DECAY = 0.945;
thought-pool.js:4  export const DREAM_DECAY = 0.90;
thought-pool.js:5  const OBSESSION_GROWTH  = 1.10;
thought-pool.js:6  const PROMOTE_THRESHOLD = 0.50;
thought-pool.js:7  const PROMOTE_MIN_AGE   = 3;
thought-pool.js:8  const FEEDBACK_AMOUNT   = 0.18;
thought-pool.js:9  const FEEDBACK_CEIL     = 0.85;
thought-pool.js:10 const MAX_FEEDBACKS     = 3;
thought-pool.js:11 const MAX_FLASH         = 8;
thought-pool.js:12 const MAX_OBSESSIONS    = 3;
```
`FLASH_DECAY=0.82` 是**未导出**的默认衰减率（模块私有 const），只在闪念没有显式携带 `decay` 字段时兜底使用（`thought-pool.js:20`）。**这与任务书列出的旧仓库线索一致（常量名相同、数值也相同 0.82），但 `SURFACED_DECAY`（0.945）和 `DREAM_DECAY`（0.90）是本文件独有导出、旧仓库审计未必给出相同数值，需以此为准。**

**核心数据结构**：`pool = { flash: [], obsessions: [] }`。
- `flash` 条目：`{ key(驱力名), text, intensity(0~1), age(结算次数), decay?(可选,覆盖默认0.82), ombreBucketId?, sourceOmbreBucketIds? }`
- `obsessions` 条目：`{ key, text, intensity, feedbacks(0~3), ombreBucketId?, sourceOmbreBucketIds? }`

**生命周期**（`tickThoughtPool`，`thought-pool.js:18-54`，每次 `settleState` 调用一次即每 15 分钟一次）：
1. 每条 flash：`intensity *= decay`（优先用自带 decay，否则 0.82），`age+=1`，`intensity<=0.05` 淘汰。
2. **晋升**：若 `intensity>=0.50` 且 `age>=3`（至少经过 3 次结算，即 ≥45 分钟）且当前 obsessions 数 <3，晋升为 obsession，从 flash 移除。
3. 每条 obsession：`intensity = min(1, intensity*1.10)`（持续放大），当 `intensity>FEEDBACK_CEIL(0.85)` 且 `feedbacks<3` 时**只在第一次**（`feedbacks===0`）回推 `FEEDBACK_AMOUNT(0.18)` 到对应驱力（3.3.6 修复：此前 3 次都推，会把驱力钉在 1.0，`thought-pool.js:45-47` 注释直接引用 CHANGELOG 原文），`feedbacks` 累加到 3 后从池中移除（obsession "退休"）。
4. 返回 `feedbacks: {[driveKey]: amount}`，由 `engine.js:471-479` 消费并加到 `state.drives`（受 `THOUGHT_FEEDBACK_CAP=0.85` 封顶）。

**`reinforceThought`**（`thought-pool.js:94-114`）：输出回流/浮现记忆的共用入口——同一 key 已存在闪念则强化其 intensity（`min(1, +step)`）并取更慢的 decay（`Math.max`），否则新建一条。这是"反复表达同一件事才会形成执念"的机制核心（注释 `thought-pool.js:87-93` 明确解释了这个"部分阻尼"设计意图：一次性表达只留一条会自然衰减的 flash；只有反复强化才能撑过 `PROMOTE_MIN_AGE` 晋升为 obsession）。

**副作用**：纯内存操作，无 I/O、无网络、无随机性。

**调用方**：`engine.js:3` 导入全部导出函数；`applySurfacedThought`（浮现记忆落池，`engine.js:246-253`）、`applyOutputReflux`（自主表达回流，`engine.js:726-744`）、`applyDreamWake`（梦醒闪念，`engine.js:227-241`）、`applyConversationEvent`（互动事件里的显式 `flashThoughts`，`engine.js:604-609`）均调用 `addFlashThought`/`reinforceThought`；`pickIntent`（`engine.js:664-694`）调用 `obsessionBonus` 给并列驱力加权；`self-signals.js:218-224` 遍历 `state.thoughtPool.obsessions` 检测"持续念头"信号；`awareness.js:98-106`（`ruleObsession`）也遍历 obsessions 生成觉察候选。

---

### 4. `emotion.js`（354 行）—— 独立于驱力的"此刻心情"层

**核心数据结构**：`state.emotion = { valence(0~1), arousal(0~1), label(中文词), updatedAt, lastCause, lastCauseAt }`；`state.emotionJournal`（≤600 条采样，`{at, valence, arousal, label, cause, top}`）；`state.emotionDays`（按天聚合，≤30 天，`{samples, meanValence, meanArousal, minValence, maxArousal, labels:{}, causes:{}}`）。

**坐标系**：valence（愉悦，0 难受~1 舒服）、arousal（唤醒，0 倦~1 亢奋），沿用 OB 记忆桶约定（`emotion.js:6`），刻意与 OB 对齐以便直接传参。

**主要函数**：

| 函数 | 行号 | 说明 |
|---|---|---|
| `applyEmotionImpulse` | 113-132 | 事件脉冲，带**惯性**：`inertia = delta>0 ? delta*(1-v)*1.6 : delta*v*1.6`——越接近边界（0或1）越难再推，物理直觉正确 |
| `blendEmotionTowardTone` | 135-145 | 会话 tone（warm/guarded/tired…）不是脉冲，是向目标点线性插值靠近（权重 `TONE_BLEND=0.15`） |
| `settleEmotion` | 148-161 | **唯一的时间结算函数**：指数回落到目标点（`emotionTarget`），无增长项，"结算多频繁都不会自激"（`emotion.js:13`, `147`） |
| `stampEmotionArgs` | 204-216 | 给 `breath`/`hold` 工具调用自动补情绪坐标；`hold` 有门槛（见下） |
| `emotionGrowthFactor` | 240-249 | 情绪调制驱力自然增速（乘法因子，不加数值），钳制在 `[0.4, 1.8]` |
| `recordEmotionSample`/`emotionTrend` | 282-326 | 情绪日志写入与趋势查询 |

**关键常量的实际效果**：
- `EMOTION_BASELINE={valence:0.55, arousal:0.30}`（`emotion.js:19`）：情绪回落的中性基线（不是 0.5，说明系统默认基调略偏"舒服、偏静"）。
- `VALENCE_HALF_LIFE_HOURS=6`/`AROUSAL_HALF_LIFE_HOURS=3`（`emotion.js:21-22`）：愉悦感散得比亢奋慢一倍——一次开心的互动，情绪愉悦要约 6 小时才回落一半，唤醒度约 3 小时回落一半。`SLEEP_DECAY_MUL=2`（`emotion.js:23`）：睡眠时衰减速度加倍（有效半衰期减半）。
- `IMPULSE_CAP=0.25`（`emotion.js:25`）：单次事件脉冲最多能直接推动情绪坐标 0.25（叠加惯性折算后通常远小于此）。
- `STAMP_MIN_DEVIATION=0.15`（`emotion.js:191`，**任务书重点提示行**）：**"hold 盖章门槛"**——只有当情绪偏离中性（`|valence-0.5|` 或 `|arousal-EMOTION_BASELINE.arousal|` ≥0.15）时，`stampEmotionArgs` 才会给 `hold`（写入 OB 正式记忆的工具）自动补上情绪坐标；`breath`（检索/共振）没有这个门槛，永远补坐标。**这不是"记忆写入决策"本身的门槛**（黑匣子/awareness 才是决策层），而是"要不要让这条记忆带上此刻心情标签"的门槛：平静状态下存一条往事，让 OB 按内容自己打标更准；情绪明显偏离时，此刻心情大概率就是这条记忆该有的心情（`emotion.js:188-189` 注释）。这是**情绪层→记忆元数据**的唯一写入通道，且是**建议性打标**而非阻止写入。
- `EMOTION_GROWTH_MODULATION`（`emotion.js:225-236`）：情绪→驱力增速的调制系数表，例如低落（valence 低）会让 `monitor`（惦记她）增速系数上升（`valence: -0.35`意味着 valence 每降 1，rate 因子 `+0.35`）、`share`（想分享）增速下降；这是"难受时更黏人、开心时更爱分享"的量化实现。中性情绪时因子恒为 1（无影响）。

**副作用**：全部是内存态修改，无 I/O。`state.emotionJournal`/`emotionDays` 会随 `state.json` 一起持久化（因为它们挂在 `state` 树上，落盘由 `state-store.js` 统一做，`emotion.js` 本身不写文件）。

**调用方**：`engine.js:7`（settleState 里调用 `settleEmotion`，applyConversationEvent 里调用 `applyEmotionImpulse`/`blendEmotionTowardTone`）；`server.js:4`（`emotionCoords`/`emotionSummary`/`stampEmotionArgs`，用于给 OB 调用打标、给 Dashboard/上下文信封渲染）；`self-signals.js:21`（`emotionSummary`，用于情绪转折信号检测）；`awareness.js` 通过 `state.emotionJournal` 间接消费（`ruleTriggers`，`awareness.js:78-96`）。

---

### 5. `personality-store.js`（356 行）—— 独立于状态循环的月度性格自评存储

**核心数据结构**：
- `PERSONALITY_DIMENSIONS`（`personality-store.js:8-23`）：**14 维**固定枚举（joy/sorrow/anger/fear/disgust/surprise/love/shame/trust/desire/calm/cognition/conflict/expression），每维 `{key, label}`。
- `personality.json`（磁盘文件，路径由 `PERSONALITY_PATH` 配置）结构：`{schemaVersion, source, month, scoredBy, periodSummary, updatedAt, dimensions:[{key,label,score(0-100),delta,reason}], anchors:[{key,label,description,addedAt}], history:[...]}`。
- `CORE_TO_DRIVES`（`personality-store.js:25-30`）：仅 4 个 14 维标签映射到驱力偏置——"爱与依恋"→possess/crave（正向）、"表达"→share（正向）、"平静与安全"→grieve/monitor（**负向**，direction=-1，即该维分数越高，grieve/monitor 增速越低）、"欲望与动机"→libido/curiosity（正向）。**注意：其余 10 维（快乐/悲伤/愤怒/恐惧/厌恶/惊讶/羞耻/信任/矛盾）完全不影响驱力**，只是纯记录/展示用的自评维度。
- `NEUTRAL_DRIVE_BIAS`（`personality-store.js:32-40`）：7 个受影响驱力的中性值全为 1（即无自评时偏置无效）。

**类 `PersonalityStore`**（`personality-store.js:217-356`）生命周期：
- **构造**（218-224）：仅保存路径与内存缓存（`cache`/`cachedMtimeMs`/`cachedMonth`），无同步 I/O。
- **`getPersonalityCore`**（226-243）：读文件+`stat` 做 mtime+月份双重缓存失效判断；**fail-safe**——文件不存在（ENOENT）或 JSON 解析失败都不抛错，而是返回 `normalizePersonalityCore({}, 'missing'|'invalid')`（**驱力偏置全 1.0，不影响心跳主循环**，`personality-store.js:239-242` 注释明确说明"私有镜像缺失或损坏时必须 fail-safe：不影响心潮运行"）。
- **`getDriveBias`**（245-247）→`driveBiasFromCore`（203-215）：`MAX_BIAS=0.10`（`personality-store.js:6`），即使 14 维分数拉满，驱力增速偏置最多 ±10%（`bias = clamp(1 + direction*signed*0.10, 0.9, 1.1)`，`signed = clamp((score-70)/30, -1, 1)`，中性分定为 70 分而非 50 分）。
- **写路径**：`recordAiAssessment`/`updateAnchors` 都通过 `this.writeQueue`（Promise 链）**串行化写请求**，防止并发写覆盖（`personality-store.js:250-253`, `313-316`）；`_persist`（296-305）用"临时文件+`chmod 0o600`+`rename`"的原子写模式，权限收紧到仅属主可读写。
- **行为锚点（anchors）**（`personality-store.js:76-106`, `307-355`）：与 14 维分开的独立"有无"型标记（≤7 条），红线规则写在注释里：**不由系统自动生成，只能 AI 自己定或用户确认；不参与驱力偏置；不被任何驱力覆盖**（`personality-store.js:77-79`）——代码里确实找不到任何自动生成 anchor 的路径，只有 `updateAnchorsUnlocked`（318-347）这一个显式写入口，佐证注释属实。

**副作用**：唯一有磁盘 I/O 的模块之一（另一个是 `black-box.js`/`cabin-store.js`），但路径与 `state.json` **完全分离**（`config.js:22` `personalityPath` 默认 `/app/state/personality.json`，与 `statePath` 不同文件）。无网络调用，无随机性（`randomUUID` 仅用于临时文件名）。

**调用方**：`server.js:28` 导入；`server.js:59` 实例化 `personality = new PersonalityStore(config.personalityPath)`；`server.js:175`/`893` 在每次 `runCycle`/每次会话事件结算前调用 `personality.getDriveBias(now)` 并传入 `settleState` 的 `options.driveBias`（`engine.js:439` 消费，`bias = clamp(driveBias?.[key] ?? 1, 0.9, 1.1)`）；MCP 工具 `xinchao_personality_reflect`/`xinchao_personality_stats`（`server.js:1369-1374`）暴露读写接口。

---

### 6. `self-signals.js`（236 行）—— 主动侧信号检测与话术生成

**核心数据结构**：`state.selfSignals = { dayUsage, driveHighSince, driveLowSeenAt, drivePeakDay, emotionEpisode, lastEmotionSignalAt, lowEpisodeOpen, longingOpen, lastWakeDreamId, awarenessDay, obsessionSignaled, recentTemplates(≤40), history(≤60) }`（`self-signals.js:89-107`）。

**六种信号触发条件**（均在 `detectSelfSignals`，`self-signals.js:136-236`，**只检测、只写追踪字段，不改驱力/情绪本身**，`self-signals.js:135` 注释明确）：

| 信号 | 触发条件（常量） | 频率限制 |
|---|---|---|
| `drive_peak` | 某驱力从 <0.60（`DRIVE_SURGE_FROM`）在 24h 内（`DRIVE_SURGE_WINDOW_MS`）涨到 ≥0.80（`DRIVE_PEAK`）并持续 ≥2h（`DRIVE_PEAK_HOLD_MS`），降到 0.75（`DRIVE_PEAK_RELEASE`）以下才清起点 | 每维每天一次 |
| `emotion_shift` | 情绪进低落/烦躁（`LOW_LABELS`）停留 ≥30min（`EMOTION_HOLD_MS`）；或从低落回到安心/雀跃（`HIGH_LABELS`） | 一个回合一次，2h 间隔（`EMOTION_GAP_MS`） |
| `longing` | `computeLonging()`（来自 `engine.js`）≥0.6（`LONGING_ON`），降到 <0.35（`LONGING_OFF`）算空档结束 | 一个空档一次 |
| `wake_residue` | 醒来且 `pendingAwareness` 带有本次睡眠的梦余韵 | 每次醒来一次 |
| `awareness` | 复盘日（默认周日）且有 open 候选 | 一周一次 |
| `obsession` | 念头池里有 `intensity>=0.5` 的 obsession，按文本去重 | 每个念头一次 |

**全局闸门**：`MAX_PER_DAY=8`（`self-signals.js:26`）、`TTL_HOURS=2`（27，投递过期）、凌晨冻结时段（`quiet()`，128-133，默认 1-8 点）内**不发任何信号**、睡眠中（`asleep`）也不发（除 `wake_residue` 本身就是醒来事件）。

**话术设计**（`self-signals.js:64-87`）：每种信号 3-5 个模板轮换（`pickTemplate`，109-117，48 小时内不重复同一模板），第一人称、无数字无维度名（源码注释 `self-signals.js:16` 明确风格约束）。**`responseHint`**（61）区分"自助类"驱力（`SELF_SERVE_DRIVES`：share/reflection/duty/curiosity/boredom，AI 自己动手就能落地，用 `xinchao_event` 记）和"关系类"驱力（想她/惦记/馋/性欲/社交/难过/生气，必须她回应才算数，AI 自己不能记）——这是 3.3.4/3.3.5 两次迭代专门修的"念头触发后不知道怎么回应/回应类型选错"问题（CHANGELOG 对应条目）。

**副作用**：纯状态计算+构造待发送文本，不做网络请求；实际发送（写入 `bridgeQueue`）由调用方 `server.js:215-236` 完成。

**调用方**：`server.js:6` 导入；`server.js:189-236`（`runCycle` 里，仅当 `config.bridge.enabled && config.bridge.selfSignals` 都为真时才执行，**默认关闭**，`config.js:107` `BRIDGE_SELF_SIGNALS` 默认 `false`）；`awareness.js:22` 反向依赖 `self-signals.js` 无——实际是 `self-signals.js:22` `import { isReviewDay } from './awareness.js'`（self-signals 依赖 awareness，非双向）。

---

### 7. `black-box.js`（146 行）—— 与记忆系统完全解耦的私密便签存储

**核心数据结构**：`BOX_KINDS = ['secret','memo','note','event','other']`（`black-box.js:14`）；条目 `{id, kind, title, text(≤2000字), createdAt, expiresAt?, kept?({bucketId,at}), surface(bool), when?, remindAt?, remindedAt?}`（`put`，33-59）；`{items:[](≤200), audit:[](≤200,只记{at,op,id}不记内容)}`。

**与 `state.json`/OB 记忆的关系（任务书重点问题，已核实）**：
1. **独立文件**：`server.js:48` `blackBox = new BlackBox(config.box.statePath)`，`config.js:110-112` 默认路径 `/app/state/black-box.json`，与 `state.statePath`（`/app/state/state.json`）是**两个不同的文件**——代码事实与 `black-box.js:1-11` 头部注释声称的"单独一个文件存（不在 state.json 里）"完全一致，**无文档/代码差异**。
2. **无 HTTP 路由读取**：`grep server.js` 只找到 `blackBox.dueReminders`（心跳循环内部用）、`blackBox.count`/`blackBox.surfaced`（仅用于"此刻"块渲染"匣子里有 N 条"+已 surface 条目的标题，不含正文，`server.js:787/1365/1428`）、以及 `server.js:931-963` 的 `handleBox` 函数——该函数注释明确写着"黑匣子：put / list / read / burn / keep。**唯一入口，没有 HTTP 路由**"（`server.js:931`），且只被 MCP 工具处理器调用（`grep mcp-protocol.js` 确认 `xinchao_box` 是唯一挂载点，`mcp-protocol.js:236-261,635-639`）。**代码事实与文档声称一致**。
3. **`keep` 是唯一进入 OB 的通道**：`server.js:953-960`，且需要 `config.ombre.writeEnabled && !config.shadowMode` 才生效，否则直接拒绝（"OB 写入没开，搬不出去"）。**AI 主动选择才会把黑匣子内容变成正式记忆**，不存在自动同步。
4. **迁移遗留**：`server.js:1494` 在启动时把旧版本（3.3 之前）积攒的 `pending`（攒下的话）迁移进黑匣子（`kind:'memo', surface:true`），与 `engine.js:66` `delete state.pending;` 配套——**这是一次性迁移逻辑，不是常驻同步**。

**关键函数**：`_sweep`（107-110，过滤过期条目，每次读写都执行）、`dueReminders`（118-127，把 `remindAt` 已过且未提醒过的条目标记 `surface:true`，供 `runCycle` 第 201-213 行经桥推送标题）。

**副作用**：磁盘 I/O（经 `StateStore` 类，见 `state-store.js`），原子写（临时文件+`rename`），无网络调用。

**调用方**：`server.js` 心跳循环（到点提醒）+ MCP 工具 `xinchao_box`（AI 唯一交互入口）。**结论：黑匣子是与 OB 记忆系统和 `state.json` 结算状态完全解耦的第三套独立私密存储，只有一个受控的"人工搬运"接口（`keep`）可以单向流向 OB。**

---

### 8. `awareness.js`（205 行）—— 自我觉察候选生成（规则驱动，非 LLM）

**核心数据结构**：`state.awareness = { candidates:[](≤60,MAX_KEEP), lastScanDay }`；候选条目 `{id, kind('trigger'|'obsession'|'soothed'), subject, text, evidence, aspect, createdAt, status('open'|'confirmed'|'dismissed'|'expired'), resolvedAt, note, ombre}`。`state.recentSurfacings`（≤200，记浮现记忆的 domain+时间，不存正文/桶id）。

**两条规则**（**3.3.3 版本已从更早的"统计型"规则精简为只留"有因果"的规则**，`awareness.js:10-12` 注释明确说明"纯计数的（本周均值…）砍掉——那些是统计不是觉察"）：
1. `ruleTriggers`（78-96）：过去 7 天（`WINDOW_DAYS`）内某个 `cause`（来自 `emotionJournal` 的 `cause` 字段）出现 ≥3 次且平均 valence <0.45 且属于 `NEGATIVE_TYPES`（conflict/loss）→ 生成"反复触发"候选；或 `SOOTHING_TYPES`（affection/intimacy/reconciliation/companionship）出现 ≥5 次且平均 valence ≥0.6 → 生成"被安抚"候选。
2. `ruleObsession`（98-106）：念头池里 `intensity>=0.7` 的持续念头 → 生成"缠人的念头"候选。

**生命周期（open→resolve）**：
1. `scanAwareness`（120-162，每天最多扫一次，`awareness.lastScanDay===today` 时跳过除非 `force`）：先把超过 `EXPIRE_DAYS=14` 未处理的 open 候选标记 `expired`；再跑两条规则，按 `RULE_PRIORITY=['trigger','obsession','soothed']` 排序，**去重**（7 天内同 `kind:subject` 不重复，`DEDUPE_DAYS`），**每天最多新增 1 条**（`MAX_PER_DAY=1`），open 总数不超过 `MAX_OPEN=8`。
2. `resolveAwareness`（164-178）：AI 主动调用，`status` 设为 `confirmed`/`dismissed`；可带自己的 `text`/`note` 覆盖模板原文。**只有 confirmed 且带了 `text` 才会被写入 OB 的 `I`（自我认知）候选桶**（该写入逻辑在 `server.js` 的 MCP handler 层，`awareness.js` 本身不调用 OB）。
3. `renderAwareness`（197-205）：只在复盘日（`isReviewDay`，默认周日）才把 open 候选渲染进上下文信封，非复盘日"只攒着，不进信封、不催"（`awareness.js:108`）。

**这层"不改驱力、不改情绪、不改人格"**（`awareness.js:15` 注释），是纯粹的"观察-候选-人工确认"回路，且明确写着"经 OB 的 I 工具沉淀成自我认知（候选桶），之后还要被 dream 见证才升正式条目——那是 OB 的规矩，这里不越过"（`awareness.js:6-7`），即 `awareness.js` 本身不直接写 OB，只产出候选交给上层。**此说法未在 `awareness.js`/`self-signals.js` 内找到与 OB 交互的代码，交互发生在 `server.js` 里；本审计未逐行核对 `server.js` 中 `writeAwarenessToOmbre` 的完整实现（超出 9 个目标文件范围），此处标注"未验证 OB 侧的具体写入格式，仅确认 awareness.js 自身不含网络调用"。**

**调用方**：`engine.js:5` 导入 `ensureAwareness`（挂载进 `ensureStateShape`）；`server.js:5` 导入，`runCycle` 每天扫描一次（`server.js:188-198`）；`self-signals.js:22` 导入 `isReviewDay` 用于觉察信号的复盘日门控。

---

### 9. `cabin-store.js`（208 行）—— "小屋"留言板 + 共享记账本（非虚拟经济，已核实）

**任务书标注"疑似虚拟经济/记账，语义存疑"，本次已通过交叉核对 `mcp-protocol.js`/`server.js` 确认真实语义：**

`cabin-store.js` 是**用户与 AI 之间的双向留言板（notes）+ 一本朴素的收支流水账（ledger）**，**不是给 AI 自己用的游戏化虚拟货币/经济系统**，理由：

1. **`money()` 函数**（`cabin-store.js:27-31`）只是把金额四舍五入到分（`Math.round((amount+EPSILON)*100)/100`），且校验 `amount>=0`——是标准货币金额规范化，不含任何"AI 自己的余额/资产"概念。
2. **`totals()`**（40-48）只算 `expense`/`income`/`net` 三个汇总数，是家庭记账本的标准统计，不涉及驱力/情绪/念头池。
3. **`addNote`**（75-106）：`from` 只能是 `'user'`或`'ai'`；**user 写的note默认上锁（`locked=true`），AI 读不到正文**（`unlockedUserNotes`，137-143，只返回 `locked===false` 的），需要用户主动 `setNoteLock` 开锁；AI 写的 note 默认不上锁（`locked=false`），用户随时能看。这是**双向但不对称**的隐私墙，与黑匣子（AI 单向私密）互补但机制不同。
4. **`mcp-protocol.js:344-374`** 确认 MCP 工具 `xinchao_cabin_inbox`（"读取已解锁的小屋来信"）/`xinchao_cabin_note`（"给小屋留一封信"）——工具描述原文就是"来信"/"便签"语义，与经济系统无关。
5. **`server.js:1229-1300`** 确认 `/dashboard/api/cabin/note` 与 `/dashboard/api/cabin/ledger` 是 Dashboard（用户网页端）暴露的普通 CRUD 接口，`ledger` 就是加/改/删一条收支记录（`type: expense|income`, `item`, `amount`, `date`），与虚拟经济游戏化机制（如"好感度货币""互动积分"）**没有任何交集**——`cabin-store.js` 全文搜索不到任何与 `drives`/`emotion`/`thoughtPool` 的引用。
6. **不参与心跳循环**：`grep engine.js/server.js runCycle` 确认 `CabinStore` 的方法从未在 `runCycle`（`server.js:170-547`）内被调用，只在 HTTP Dashboard 路由和 MCP 工具处理器里被动调用——**它是一个附属的、与动态心智状态机完全解耦的"人机共享笔记本+账本"功能，唯一的耦合点是它触发的 `bridge` 通知（`enqueueCabinNotice`，`server.js:1073`）会经同一条推送通道递到窗口，但不影响任何驱力/情绪数值**。

**结论（消解此前"存疑"）**：`cabin-store.js` 语义明确为"小屋"——情侣/用户与 AI 之间的私人留言簿 + 共同记账工具，是**产品层面的辅助功能**，与动态心智核心（驱力/情绪/念头/觉察）**在数据和调用两个层面都完全独立**。

**调用方**：`server.js:58` 实例化；Dashboard HTTP 路由（`server.js:1229-1300`）+ MCP 工具（`xinchao_cabin_inbox`/`xinchao_cabin_note`，`mcp-protocol.js:683-691`）。

---

## 10. 自主唤醒/动态状态循环完整时序还原

**入口**：`server.js:1505` `const timer = setInterval(() => runCycle()..., config.settleIntervalMinutes * 60_000);`，`settleIntervalMinutes` 默认 **15 分钟**（`config.js:29`，`SETTLE_INTERVAL_MINUTES` 环境变量可调，范围 1~1440）。`runCycle()`（`server.js:170-547`）用 `cyclePromise` 做互斥（同一时刻只有一个周期在跑，`server.js:171`）。

### 一次完整心跳周期的时间顺序（均在 `runCycle()` 内，`server.js:170-547`）：

```
0. synchronizeOmbreHeartbeat()                         [server.js:174]
   读取 OMBRE_HEARTBEAT_FILE（默认 /memory-data/heartbeat.json，
   由外部 OB 服务的 POST /heartbeat 路由写入，heartbeat-store.js:1-14）。
   若文件里的 recordedAt 比 state.lastHeartbeatAt 新，
   调用 applyOmbreHeartbeat(state, recordedAt)
     → 内部调用 applyConversationEvent(state, {}, recordedAt)  [engine.js:698-702]
     → 这是一次"空事件"，但会把 consciousness 设回 'awake'、
       lastConversationAt=recordedAt，并触发"梦醒后果"
       （applyDreamWake，若 wasSleeping 且有属于本次睡眠的梦，engine.js:540-554）。
   ★关键点：xinchao 的"醒来"判定不依赖真实聊天事件，
     而是依赖 OB 侧写的一个心跳文件时间戳——只要 OB 认为"人还在"，
     xinchao 就会醒，即使这段时间内 xinchao 自己没收到任何 MCP 互动事件。

1. driveBias = personality.getDriveBias(now)            [server.js:175]
   读 personality.json（月度自评），算出 7 个驱力的 ±10% 增速偏置。

2. settleState(state, now, sleepAfterMinutes, {...})    [server.js:177-184, engine.js:387-503]
   ── 这是"结算"本身，按 elapsedHours = (now - lastSettledAt)/3600000 一次性补齐：
   a. 清理过期 sessionOverlays、过期 satisfactionPlateaus
   b. 对每个驱力：
      - 凌晨冻结时段（默认1-8点）且 dawnFreeze=true → 冻结不变
      - grieve/anger（有 decayHalfLifeHours）→ 纯指数回落到0
      - 其余9维 → 若超过天花板则按 CEIL_RELAX_PER_HOUR=0.10 松弛回落；
        否则按 growPerHour × 夜间倍率 × 疲劳倍率 × 耦合倍率 × 情绪调制倍率 增长，
        若在 satisfactionPlateau 内则增速强制为0
   c. settleEmotion()：情绪按半衰期指数回落到 emotionTarget()（受 grieve/anger 拉扯）
   d. tickThoughtPool()：念头池衰减/晋升/回推（见第3节），回推结果加到对应驱力
      （THOUGHT_FEEDBACK_CAP=0.85 封顶）
   e. fatigue：睡眠中每小时-0.02，清醒且平均驱力>0.5时每小时+0.005
   f. 睡眠转换：idleMinutes=(now-lastConversationAt)/60000 ≥ sleepAfterMinutes(默认90分钟)
      且当前不是睡眠 → consciousness='sleeping', sleepStartedAt=now

3. if awareness.enabled(默认true):                      [server.js:188-198]
   scanAwareness()——每天（上海时区日期）最多扫一次，只加候选、不动驱力/情绪

4. blackBox.dueReminders(now)                           [server.js:201-213]
   到点的黑匣子提醒自动 surface，若桥开着则额外入队一条"匣子里有一条到点了"

5. if bridge.enabled && bridge.selfSignals(默认关):      [server.js:215-236]
   detectSelfSignals() 检测6种信号，逐条 enqueue 到 bridgeQueue，
   立即 publishReadyBridgeDeliveries() 推送

6. if longing.enabled(默认true):                         [server.js:243-255]
   computeLonging() 现算挂念值 → applyLongingNudge() 轻推 monitor 驱力

7. dreamContactIsIdle = contactIdleAllowed(..., dreamMinIdleHours默认3h)
   proactiveContactIsIdle = contactIdleAllowed(..., proactiveMinIdleHours默认12h)
                                                          [server.js:258-259]
   ★这两个"空闲时长"用的是 lastHeartbeatAt（不是 lastConversationAt），
     所以只要 OB 心跳文件持续被外部更新，这两个门槛就不会满足
     ——必须"真的没有心跳信号"达到对应小时数才会触发梦/自主念头。

8. 【做梦分支】if dreamEnabled(默认true) && dreamAllowed(
     consciousness==='sleeping' 且 距上次梦 ≥ DREAM_MIN_INTERVAL_HOURS(默认6h)
     且今天 dreamUsage < DREAM_MAX_PER_DAY(默认4)):        [server.js:261-331]
   a. 拉材料：OB digestMaterial(48h) 优先，退化到 recentMaterialWithRefs 按最强驱力+情绪坐标召回；
      再叠加 farMaterial（更早期记忆）
   b. 调用 model.generateDream()（LLM，若禁用/失败则用规则 fallback）生成梦
   c. recordDream() 落状态（≤20条历史）
   d. 若 OB writeEnabled：storeDream() 写回 OB
   e. 不立即推送——写入 pendingDreamPush，攒到"早上"再推

9. 【梦推送分支】if bark.enabled && state.pendingDreamPush:  [server.js:334-350]
   条件：hour>=8 且 (anticipation>=0.3 或 hour>=9)，且 dreamContactIsIdle，
         且 barkAllowed(kind='dream', minIntervalHours默认3h, maxPerDay默认6)
   → sendDreamPush()：LLM 生成推送文案（去重）→ Bark 推送 →
     recordBark() → 若 reflux.enabled(默认true)：applyOutputReflux()
     把推送文案回流进念头池（对应最强驱力）

   ★注意：这里推送的是"梦的余韵一句话"，而"梦对情绪/念头的真实影响"
     （applyDreamWake）发生在步骤0（下一次真实到访/心跳被观测到、
     wasSleeping=true 时），两者时间点不同、触发条件也不同。

10.【自主念头分支】if bark.enabled && proactiveContactIsIdle && !dreamCreated
      && proactiveBarkAllowed(仅睡眠中, 最强驱力>=minDrive默认0.42,
         距上次autonomous bark>=autonomousMinIntervalHours默认12h,
         今日maxPerDay默认6):                             [server.js:352-438]
    a. 拉 thoughtMaterial（OB 按最强驱力召回）
    b. 若 resonance.enabled(默认true) 且有material：parseSurfacedDomains()
       → applyMemoryResonance()（记忆回推驱力，见 dimensions.js DOMAIN_AFFINITY）
       + recordSurfacing()（写进 awareness 的 recentSurfacings）
    c. LLM generateThought() 生成一句自主念头（去重）
    d. Bark 推送 → recordBark() → applyOutputReflux()（回流进念头池）

11.【白天浮现分支】daytime.enabled(默认false):              [server.js:440-543]
    首次：随机安排下次浮现时间（minIntervalHours~maxIntervalHours，默认2~3h）
    到点（daytimeEmergenceAllowed：在[startHour,endHour)默认[8,23)内，
          今日maxPerDay默认7未超）：
    a. 拉 daytimeMaterialWithRefs → applyMemoryResonance + recordSurfacing
    b. 若有material：取第一句 → surfacedDriveKey()（按DOMAIN_AFFINITY选最亲和驱力）
       → applySurfacedThought()（落进念头池当闪念，不直接推送她，见engine.js:246-253）
    c. 若 daytime.bark(默认false，多数部署里此分支不产出对外推送)：
       LLM生成推送文案 → Bark → recordBark → applyOutputReflux
    d. 重新调度下次浮现时间
```

**情绪/驱力/念头/觉察/自身信号/黑匣子在循环中的读写点汇总**：

| 模块 | 读取点 | 写入点 |
|---|---|---|
| `emotion.js` | 每次生成 LLM 提示词时（`emotionSummary`），`self-signals`步骤5 | `settleState`每15分钟（settleEmotion）、`applyConversationEvent`时的脉冲（步骤0的空事件不带脉冲）、`applyDreamWake`（梦醒） |
| `personality-store.js` | 每次 `runCycle` 开头（步骤1）、每次会话事件（`server.js:893`） | 仅 MCP 工具 `xinchao_personality_reflect`/`updateAnchors` 主动写，**不在心跳循环内被写** |
| `self-signals.js` | 无（本身即检测器） | 步骤5，每15分钟一次（若开关打开） |
| `awareness.js` | `self-signals`步骤5读 open 候选 | 步骤3，每天一次 |
| `black-box.js` | "此刻"块渲染时（`count`/`surfaced`） | 步骤4（到点提醒 surface 化），MCP `xinchao_box` 主动写 |
| `thought-pool.js`（挂在 `state.thoughtPool`） | 步骤10/11 生成材料后、`self-signals`/`awareness` 检测执念时 | `settleState`每15分钟tick一次，步骤8/10/11产出后回流 |
| `cabin-store.js` | Dashboard 页面 | Dashboard/MCP 主动写，**完全不在此循环内** |

---

## 11. 核心机制 vs 外围功能的判断标准与分类

**判断标准（四条，任一满足即倾向"核心"）**：
1. **是否驱动核心状态循环**：即是否在 `settleState`（每 15 分钟必跑一次的心跳结算）内被直接调用或读写。
2. **是否被 ≥2 个其他核心模块依赖**（`import` 关系，不含 `server.js` 这个总装配层）。
3. **去掉它，`settleState`/`applyConversationEvent` 是否会报错或语义显著坍塌**（例如 `dimensions.js` 是 `DRIVE_KEYS` 的唯一来源，去掉整个驱力系统无法运行）。
4. **是否只是某个默认关闭的可选功能的专属支撑**（例如 `daytime.enabled` 默认 `false`，`bridge.selfSignals` 默认 `false`）——若是，倾向"外围"。

**分类结果**：

| 模块/机制 | 分类 | 依据 |
|---|---|---|
| `dimensions.js`（12维驱力表+DOMAIN_AFFINITY） | **核心** | 标准1、3：`settleState`每15分钟直接遍历`DIMENSIONS`；`DRIVE_KEYS`是全系统白名单唯一真源 |
| `engine.js`（settleState/applyConversationEvent等） | **核心** | 标准1、2：本身就是心跳结算函数所在文件，被`server.js`/`self-signals.js`共同依赖 |
| `thought-pool.js`（闪念/执念） | **核心** | 标准1、3：`settleState`第468-479行直接`tickThoughtPool`并消费回推结果，去掉后"念头/执念/自主表达回流"整条链路失效 |
| `emotion.js`（valence/arousal层） | **核心** | 标准1、2、3：`settleState`第466行直接调用`settleEmotion`；被`self-signals.js`/`awareness.js`（经emotionJournal）依赖 |
| `personality-store.js`（月度自评+驱力偏置） | **外围但影响核心** | 标准4的反例：不在`settleState`内部产生数据，只是**输入参数**（driveBias），且有完整fail-safe（缺失时偏置全1.0，`settleState`照常运行）。**归类为"核心的可选调制层"**——去掉它系统仍完整运行，只是失去±10%的性格化偏置 |
| `self-signals.js`（六种自身信号） | **外围** | 标准4：整条功能挂在`config.bridge.selfSignals`（默认`false`）开关下；`detectSelfSignals`本身不修改驱力/情绪/念头池的实际数值（只读+写自己的追踪字段），是一个"观测层→通知"的旁路 |
| `black-box.js`（黑匣子） | **外围** | 标准3、4：与`state.json`完全分离的独立存储；去掉它，`settleState`唯一受影响的是"到点提醒"这一条（`server.js:201-213`），核心驱力/情绪循环不受任何影响 |
| `awareness.js`（自我觉察候选） | **外围** | 标准4：`config.awareness.enabled`默认`true`但产出只是"候选列表"，不反向修改驱力/情绪/人格（`awareness.js:15`自陈"这层不改驱力、不改情绪、不改人格"）；是一个纯粹的观察-记录层 |
| `cabin-store.js`（小屋留言+记账） | **外围（且与动态心智无耦合）** | 标准2、3、4：不在`runCycle`内出现，`import`关系上与`engine.js`/`dimensions.js`/`emotion.js`零交集；即使完全删除该文件，动态心智状态机的心跳/驱力/情绪/念头/觉察/黑匣子机制**全部不受影响**（这是9个模块里唯一"真正可整体拆除而不影响核心"的模块） |

**补充说明**：`personality-store.js`的归类需要额外解释——它不满足"驱动核心循环"（不在`settleState`内产生副作用，只作为只读输入），但它的输出（`driveBias`）确实被`settleState`每次调用都使用（`engine.js:439`）。本审计将其归为"外围但影响核心的调制层"，与`self-signals.js`/`awareness.js`/`black-box.js`（纯粹的观察者/旁路，对核心状态无任何反向影响）区分开——这是**唯一一个外围模块能够修改核心数值增速**的例子，移植评估时需要单独考虑。

---

## 12. 可移植到 Haven 的机制评估（逐条对照 haven-ombre 现有代码）

本节已实际阅读 `/home/user/18358386529/haven-ombre/` 的 `decay_engine.py`（324行）、`persona_engine.py`（1658行，重点读1-260行）、`gateway_state.py`（703行）、`identity.py`（69行）、`favorite_tags.py`（58行）、`memory_write_gate.py`（结构，111-205行区间）、`reflection_engine.py`（结构确认存在`ReflectionEngine`类，4364行未逐行读完）、`memory_edges.py`/`entity_edges.py`（结构）、`bucket_manager.py`（结构，`valence`/`arousal`字段确认存在，`bucket_manager.py:15-22,102,124-125,171-172,286,320-323`）。

**重大发现（决定性证据）**：`haven-ombre/persona_engine.py` 已经存在一套**与`emotion.js`高度同构、甚至更丰富**的会话情绪/人格状态系统：
- `PersonaStateEngine.AFFECT_KEYS`（`persona_engine.py:106-115`）：`valence, arousal, tenderness, possessiveness, longing, security, protective_drive, libido` 8维（比xinchao的valence/arousal二维更丰富，但**驱动方式完全不同**——由LLM每隔几轮评估一次`affect_delta`并叠加，而非xinchao式的连续时间函数增长）。
- `session_mood_half_life_minutes`（默认90分钟，`persona_engine.py:136-138,934-937`）：`retention = 0.5 ** (elapsed_minutes / half_life)`——**这与`emotion.js`的`settleEmotion`半衰期回落公式在数学形式上完全一致**（都是标准指数衰减），只是half-life数值（90分钟 vs xinchao的6小时/3小时）和触发方式（LLM评估后落盘 vs 每15分钟结算函数计算）不同。
- `mood_label`字段（`persona_engine.py:31,215,829,989`）：与`emotion.js`的`emotionLabel()`（二维坐标→中文词）同构，但Haven版由LLM直接给出标签而非规则映射。
- SQLite持久化（`persona_engine.py:247-260`区域的`persona_global_state`表），与xinchao的JSON文件持久化（`StateStore`）是不同的存储介质，但语义等价。

这意味着：**xinchao的"情绪层"机制思路（valence/arousal+半衰期回落+事件脉冲+惯性）在Haven并非全新概念，Haven已有一个"更重"（LLM驱动、多维）但"更弱"（依赖LLM调用成本、非确定性、无独立于会话的全局态）的平行实现。** 移植建议因此从"从0搭建"变为"评估是否用xinchao的**确定性规则算法**（半衰期公式、脉冲惯性公式）去补强/替代persona_engine.py中目前完全依赖LLM判断的衰减环节**——这是本审计最重要的可移植方向。

`decay_engine.py`已有基于`arousal`（连续坐标）的记忆衰减权重（`emotion_weight = base + arousal*arousal_boost`，`decay_engine.py:136-141`），与xinchao的`emotionCoords`→OB的`breath`/`hold`共振/打标思路（`emotion.js:183-216`）**目标一致**（都是让"情绪强度影响记忆的鲜活度/检索排序"），但Haven目前只用`arousal`单维，未用`valence`，且是"桶的固有情绪标签"而非"此刻AI自身情绪状态"两个不同维度的混用观察点——需要人工确认这是有意为之还是遗漏。

`memory_write_gate.py`的`pending_threshold=0.42`/`grow_threshold=0.72`（`memory_write_gate.py:139-140`）是记忆候选→正式写入的打分门槛，与`awareness.js`的"候选→confirmed"两阶段流程在**结构思路上**类似（都是"先攒候选，达到门槛/经确认才转正"），但触发机制完全不同（Haven是分数阈值自动升级，xinchao是纯人工/AI确认，无自动阈值）。

---

## 最终交付物：心潮念动态心智 → Memory System 2.0 的可移植机制表

| 机制/模块 | 一句话功能描述 | 核心/外围 | 是否建议移植 | 移植目标（Haven现有位置或"需新增落点"） | 移植方式 | 复杂度估计 | 前提条件/待人工决策项 | 证据ID |
|---|---|---|---|---|---|---|---|---|
| 情绪指数半衰期回落公式 | `settleEmotion`用`goal+(value-goal)*0.5^(hours/halfLife)`把valence/arousal确定性地回落到目标点，无LLM调用、无随机性 | 核心 | **是，优先级最高** | `haven-ombre/persona_engine.py`（`session_mood_half_life_minutes`已有half-life概念但由LLM整体给出delta后一次性写入，不是逐次调用的确定性衰减函数） | 移植**算法思路**（衰减公式本身+"回落目标可被外部因素拉扯"的设计），不搬xinchao代码；可作为`persona_engine.py`情绪状态在两次LLM评估之间"自然冷却"的补充计算，减少对LLM的依赖 | 小（公式本身几行数学，但要接入Haven现有的SQLite持久化和调用时机需要设计） | 需要人工决定：Haven的情绪状态要不要拆出"LLM评估的净变化(delta)"与"确定性时间冷却"两层（当前persona_engine.py是LLM一次性给出评估后的绝对值，没有独立的时间冷却环节）；half-life数值要不要沿用90分钟还是采纳xinchao的6h/3h | `emotion.js:148-161`；`persona_engine.py:136-138,934-937` |
| 情绪脉冲惯性公式 | `applyEmotionImpulse`：越接近0/1边界，同向脉冲的实际影响越小（`delta*(1-v)*1.6`或`delta*v*1.6`） | 核心 | 是 | `persona_engine.py`目前`affect_delta`直接由LLM给出增量（`persona_engine.py:28`），没有代码层面的边界衰减/惯性钳制，理论上LLM可能给出让valence持续逼近1或0的delta | 移植**算法思路**（边界惯性钳制函数），作为LLM给出的`affect_delta`应用前的一道确定性夹逼层 | 小（一个纯函数） | 需要人工确认Haven是否希望LLM给出的情绪delta被系统性"打折"（可能改变已调好的prompt效果，需要与persona_engine维护者对齐） | `emotion.js:113-132` |
| 情绪→驱力增速调制（`EMOTION_GROWTH_MODULATION`） | 情绪不直接改数值，只改"接下来自然长多快"的乘法因子，钳制在[0.4,1.8] | 核心 | 视情况（**依赖驱力系统是否移植**） | 无对应位置——Haven没有12维持续增长的"驱力"概念，这条机制的前提（存在时间连续增长的欲望值）在Haven不成立 | 若Haven未来引入类似驱力/需求系统，才移植思路；否则不适用 | 中～大（前提依赖驱力系统整体落地） | **待人工裁决**：Haven的Memory System 2.0 是否需要引入"AI自身持续增长的欲望/需求"这类拟人化驱力层，还是保持纯记忆/检索定位而不做这类情感驱动的自主行为 | `emotion.js:225-249` |
| hold盖章门槛（`STAMP_MIN_DEVIATION=0.15`） | 只有情绪明显偏离中性时才自动给`hold`（写入正式记忆的动作）打上情绪坐标；`breath`（检索）无门槛 | 核心（情绪→记忆的唯一写入通道） | 是 | `haven-ombre/bucket_manager.py`（`valence`/`arousal`已是bucket的一等字段，`bucket_manager.py:124-125,171-172,320-323`）+ `decay_engine.py`（衰减公式已用`arousal`） | 移植**思路**：写入记忆时，"是否自动打情绪标签"应有一个基于当前情绪偏离中性程度的门槛，而不是每次都打或从不打 | 小（一个阈值判断函数） | 需要人工确认Haven当前写入记忆时`valence`/`arousal`从哪里来（是调用方显式传入还是系统自动补），以及是否已有类似门槛（本审计**未验证**Haven侧的记忆写入调用点是否已有等价逻辑，需要另外审计`bucket_manager.py`的调用方） | `emotion.js:191,204-216` |
| 驱力/念头/情绪耦合系统（`DRIVE_GROWTH_COUPLINGS`+念头池晋升/回推） | 12维时间连续增长的欲望值+闪念→执念的双层强化-回推闭环 | 核心 | **否（不建议整体移植）** | 无对应位置——这是xinchao作为"虚拟伴侣人格模拟"产品的核心拟人化机制，与Haven"Memory System"（记忆检索/衰减/关系图谱）的产品定位不同 | 不适用 | 大（需要整套时间连续状态机+心跳循环基础设施） | **待人工裁决**：这是本次审计中最需要总控明确回答的问题——Memory System 2.0 是否要在记忆层之外叠加一层"AI自身情绪/欲望的自主生长与自我表达"，这是产品定位问题，不是技术移植问题 | `dimensions.js`全文；`thought-pool.js`全文；`engine.js:26-30,387-503` |
| 记忆域→情绪/需求共振表（`DOMAIN_AFFINITY`） | 浮现的记忆按`domain`标签，用一张"域→维度亲和度"表回推到对应的内在状态维度，多域取最大值不累加 | 核心（在xinchao体系内） | 是（**思路**，非表内容） | `haven-ombre/bucket_manager.py`（已有domain/facets等分类字段作为软索引，`bucket_manager.py:15`注释"Multi-dimensional soft index: domain + valence/arousal + fuzzy text"，Haven已经在做"domain参与检索排序"，但未发现"domain回推到某个持续状态维度"的反向机制） | 移植"多域取最大值不累加、设最小亲和度门槛（如xinchao的0.5）"这一**设计模式**，若Haven未来需要让检索到的记忆反过来影响某种持续状态（如"最近话题热度""关注度"），可以复用这个防自激的取max模式 | 中（表本身要重新设计映射到Haven真实存在的状态维度，而不是照搬xinchao的12维） | 依赖上一条"驱力系统是否移植"的裁决结果；若不移植驱力系统，这条也没有落点 | `dimensions.js:113-158`；`bucket_manager.py:15-22` |
| 到访节律学习+期待/挂念（`arrivalHistogram`/`computeAnticipation`/`computeLonging`） | 从真实到访的24小时分布直方图（指数衰减更新）现算"此刻有多期待/多挂念"，不落状态、用时现算，静默时段自动归零不催促 | 核心 | 是 | 无直接对应——`gateway_state.py`有`get_last_success_at`等时间戳记录，但没有"学习用户活跃时段分布"的直方图机制 | 移植**思路**（24小时直方图+指数衰减更新+"派生值不落状态、用时现算"的设计模式），可用于Haven判断"现在是不是用户通常活跃的时段"这类场景感知，而不必是"想她"的拟人化语义 | 小～中（直方图更新是几行代码，但要接入Haven现有的会话时间戳来源） | 需要人工确认Haven是否需要这类"用户作息感知"功能，以及数据来源（`gateway_state.py`的`conversation_turns`表已有时间戳，具备复用基础） | `engine.js:376-383,799-843`；`gateway_state.py:88-114,315-370` |
| 情绪日志双层聚合（逐条采样+按天聚合，`emotionJournal`/`emotionDays`） | 采样有去重间隔（结算≥2h一条，事件≥30min一条），按天聚合均值/极值/标签计数/成因计数，30天/600条上限 | 外围（支撑awareness） | 是 | 无直接对应——`gateway_state.py`的`conversation_turns`/`upstream_usage`是"技术遥测"表，不是"情绪/心情走势"表；`persona_engine.py`的`persona_global_state`只存**当前**状态，没有找到历史轨迹表（**未验证**：`persona_engine.py`全文1658行本审计只读了前260行，是否存在历史轨迹表需要另外确认，此处标注为"未在已读区间发现，需要人工/后续审计核实") | 移植**思路**（双层采样+聚合的设计模式，含防止过密采样的去重间隔），具体聚合维度需按Haven的`mood_label`/`valence`/`arousal`重新设计 | 中（需要新增一张历史表+采样触发逻辑） | 依赖情绪半衰期回落机制是否移植；若Haven的情绪状态仍是纯LLM评估、无逐次结算函数，这套"采样间隔"机制的触发点需要重新设计 | `emotion.js:251-326` |
| 自我觉察候选：规则驱动的"反复触发/被安抚/执念"三类检测 | 从情绪日志和念头池的轨迹里用简单统计规则（次数+均值阈值）挑出候选，每天最多1条，人工确认才转正 | 外围 | 是 | `haven-ombre/reflection_engine.py`（`ReflectionEngine`类，4364行，**本审计未逐行读完，仅确认该类存在**，"未验证"其内部是否已有类似的"候选→确认转正"两阶段流程；`memory_write_gate.py`的`pending_threshold`/`grow_threshold`两级阈值在**结构思路**上与"候选→confirmed"相似） | 移植**思路**（"规则先挑候选、AI/人工确认才沉淀、未处理自动过期"的三段式流程 + "去重/限流：一天最多一条、7天内同类不重复、两周未处理过期"的具体限流参数设计），不移植xinchao的具体触发规则（trigger/obsession/soothed是围绕"驱力/念头"设计的，Haven没有这两个概念） | 中（流程设计可复用，具体触发规则要基于Haven现有的`reflection_engine.py`/记忆图谱重新设计） | **需要先读完`reflection_engine.py`全文**（本审计受限于目标文件范围未展开）确认是否已有等价机制，避免重复造轮子；这是本报告"未验证"项中最需要后续审计跟进的一条 | `awareness.js:1-16,78-117,120-162` |
| 黑匣子：AI专属私密便签（与记忆/state完全隔离、唯一HTTP不可达、需AI主动keep才转正式记忆） | 独立文件存储，5种kind，支持到期/到点提醒，审计只记时间+动作+id不记内容 | 外围 | 是（**产品/隐私设计思路**，非代码） | 无直接对应——Haven目前没有发现"AI自己的私密便签本"概念（`identity.py`/`favorite_tags.py`都是配置/标签，不是私密存储） | 移植**设计模式**：①与主存储物理隔离；②唯一读写入口收窄到AI自己的工具，无HTTP路由；③"人工搬运才转正"的单向阀门（而非自动同步）——这套"隐私分层"思路可用于Haven未来若要给AI一个不经过记忆检索、不会被用户查看的暂存区 | 小～中（存储层本身简单，StateStore式的原子写模式可直接复用思路；难点在于要不要在Haven现有架构里再开一条独立的存储通道） | **产品决策优先于技术决策**：Haven的Memory System 2.0 是否需要"AI专属隐私空间"这个产品概念，这直接关系到是否要做、而不是怎么做 | `black-box.js:1-146`；`server.js:931-963` |
| 记忆写入决策的"候选池+阈值升级"模式（`memory_write_gate.py`已有） | Haven已有`pending_threshold=0.42`/`grow_threshold=0.72`的打分升级机制 | （Haven侧已有，非xinchao待移植项） | 不适用（Haven已有更成熟实现） | `memory_write_gate.py:139-140` | 不适用 | 不适用 | 本行仅作为交叉参照，说明Haven在"候选→正式"这类分级写入上已经领先于xinchao的简单阈值设计，xinchao的`awareness.js`反而可以反向参考Haven的两级阈值设计来改进自己（超出本次审计范围，仅记录观察） | `memory_write_gate.py:111-205` |
| 小屋留言板+记账（`cabin-store.js`） | 用户/AI双向留言（不对称锁定）+ 家庭收支记账 | 外围，与动态心智无耦合 | **否** | 无需移植 | 不适用 | 不适用 | 这是产品层面的用户功能（人机共享笔记+账本），与"Memory System 2.0"的记忆/衰减/身份定位无关，不属于本次审计"动态心智机制"移植范围 | `cabin-store.js:1-208`；`mcp-protocol.js:344-374` |
| 人格月度自评→驱力偏置（`personality-store.js`） | 14维月度自评+4维映射到驱力±10%偏置，fail-safe无缝降级 | 外围（调制层） | 视情况（**依赖驱力系统是否移植**） | 无对应——Haven没有"月度性格自评影响持续状态增速"的机制；`persona_engine.py`的`relationship`四维（affinity/dominance/defensiveness/trust）是**每次LLM评估后累积**的全局关系状态，语义上更接近xinchao的"驱力偏置来源"而非"驱力"本身 | 移植"fail-safe：私有存储缺失/损坏不影响主循环，偏置退化为中性1.0"这一**健壮性设计思路**，具体的"14维→4维→7个驱力"映射链路依赖驱力系统是否移植 | 中～大（依赖驱力系统） | 与"驱力/念头/情绪耦合系统"是否移植的裁决绑定 | `personality-store.js:25-40,203-243`；`persona_engine.py:105-115,199-220` |

---

## 发现的冲突与待人工裁决项

1. **产品定位的根本冲突（最重要）**：xinchao的核心创新（12维时间连续驱力+念头池+自主唤醒说话）是围绕"虚拟伴侣人格模拟"设计的拟人化情感自主系统；Haven（`haven-ombre`）目前的定位更偏"记忆检索+衰减+关系图谱"的记忆基础设施，虽然`persona_engine.py`已经有一层会话情绪/关系状态，但驱动方式是LLM按轮次评估，不是xinchao式的确定性时间函数。**是否要把"AI自身持续增长的欲望驱动自主表达"这类机制引入Memory System 2.0，是产品范围决策，不是技术可行性问题**，本审计不代为裁决，明确列为待总控/产品负责人决策项。

2. **`reflection_engine.py`未读完**：该文件4364行，本审计仅确认`ReflectionEngine`类存在（`reflection_engine.py:302`），未逐行核实其内部是否已经实现了与`awareness.js`等价或更强的"候选→确认→沉淀"自我觉察流程。**这是本报告最大的"未验证"缺口**，建议后续审计单独立项，避免在"自我觉察机制移植"决策上重复造轮子或误判Haven现状。

3. **`persona_engine.py`情绪历史轨迹表存在性未验证**：本审计只读了该文件前260行（总1658行），"是否存在类似`emotionJournal`/`emotionDays`的历史轨迹持久化"标注为**未验证**，需要后续读取该文件剩余部分（尤其是260-950行区间的表结构和写入函数）确认。

4. **`awareness.js`写入OB的具体格式未验证**：`awareness.js`自身承认"经OB的I工具沉淀成自我认知"但该写入逻辑（`writeAwarenessToOmbre`）位于`server.js`而非本次9个目标文件之一，本审计未展开核实其具体写入的字段格式和是否有额外校验，标注为"未验证，超出本次目标文件范围"。

5. **情绪→记忆共振的`emotion_weight`维度不一致**：`decay_engine.py`的衰减公式只用`arousal`（`decay_engine.py:136-141`），未使用`valence`；而xinchao的`stampEmotionArgs`同时使用`valence`和`arousal`两个坐标做打标门槛（`emotion.js:191,204-216`）。**这是否是Haven有意为之的设计取舍（例如valence在记忆衰减语境下语义不清）还是遗漏，本审计未能确认，建议人工向`decay_engine.py`维护者确认后再决定移植时是否需要补上valence维度**。

---

*本报告严格限于源码静态审计。所有"是否建议移植"的结论均为技术可行性与设计思路评估，不构成实施计划；任何实际代码改动、依赖新增、数据库表结构变更均需另行经过人工批准的实施阶段，本次审计不产出、也不建议直接复制粘贴任何xinchao源码到Haven。*
