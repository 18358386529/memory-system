# 清单5:License / Security 审计报告

**审计模式声明:本报告为源码审计,非运行时审计。** 四个目标系统在审计时均无本地运行实例,所有结论均来自对已只读克隆到本地的源码快照的静态阅读(`Read`/`Grep`/`git log`/`diff`/`md5sum`/`wc` 等只读命令),未执行、未安装、未启动任何被审计代码,未联网查询任何 CVE 库、License 数据库或包仓库(npm/PyPI)最新版本信息。

## 锁定 commit SHA 核对

| 本地路径 | 参考仓库(full_name) | 任务书锁定 SHA | 实际 HEAD | 是否一致 |
|---|---|---|---|---|
| /home/user/18358386529/haven-ombre | Yinglianchun/Haven-Ombre | 284c9c7b0e51a0ba0032c7028f705d72458cb304 | 284c9c7b0e51a0ba0032c7028f705d72458cb304 | 一致 |
| /home/user/18358386529/ombre-brain | P0luz/Ombre-Brain | 6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5 | 6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5 | 一致 |
| /home/user/18358386529/xinchao-nian | tianyupaipai-cmd/xinchao-nian | 97f1bdcc76b748fa516b7c79a0aa10234143795f | 97f1bdcc76b748fa516b7c79a0aa10234143795f | 一致 |
| /home/user/18358386529/kiwi-mem | LucieEveille/kiwi-mem | b01a0c506f4f10f90f30d408c0291f16720f0718 | b01a0c506f4f10f90f30d408c0291f16720f0718 | 一致 |

四个克隆均为**浅克隆**(`git log --oneline` 每个仓库均只返回 1 条记录,`git log -1 --format="%ai"` 与 `git log --reverse --format="%ai" | head -1` 结果相同),无法用 `git log` 追溯提交历史来判断"维护状态",本报告的维护状态列改用 CHANGELOG.md / VERSION 文件 / 单条 HEAD 提交的时间戳 / 仓库内文档自述佐证,并在证据ID中注明"浅克隆,仅见当前 HEAD 一条提交,无法核实历史提交频率"。

xinchao-nian 的 `.gitmodules` 声明了子模块 `bridge`(指向 `xinchao-runtime-bridge`),本地克隆后 `bridge/` 目录为空,子模块内容不在本次审计范围内。

---

## 清单5 表格:License / Security(5 个审计单元)

### 单元1:haven-ombre(Memory Core 本体)

| 项 | 内容 |
|---|---|
| 项目 | haven-ombre |
| 仓库(full_name) | Yinglianchun/Haven-Ombre(本地克隆源:`18358386529/haven-ombre`,公开 fork) |
| 许可证(自身 LICENSE) | MIT License,`Copyright (c) 2026 P0lar1zzZ`(与上游 ombre-brain 的版权人相同——README 自述"基于 P0luz/Ombre-Brain 二次开发",保留了原始 MIT 版权声明,未新增自己的版权行) |
| 依赖许可证 | Unknown——仓库内**未 vendor 任何第三方依赖源码**(`find` 未发现 `vendor/`、`node_modules/`、`third_party/` 目录),`requirements.txt` 只列包名+版本号(`mcp`、`rapidfuzz`、`openai`、`pyyaml`、`tzdata`、`python-frontmatter`、`jieba`、`httpx`、`starlette`、`uvicorn`、`python-multipart`、`numpy`),没有任何依赖包自带的 LICENSE/NOTICE 文件被复制进本仓库,无法在不联网的前提下确认各依赖的许可证条款,如实标注 unknown |
| 采用方式 | 代码——README.md 第 1-6 行自述"本仓库基于 P0luz/Ombre-Brain 二次开发……不是原版 Ombre-Brain 的无改动镜像",属于整份代码库的 fork+深度改造,不是仅参考算法思想 |
| 维护状态 | 浅克隆仅见 1 条 HEAD 提交,时间戳 `2026-08-07 11:10:40 +0800`(commit message: "Fix raw event date range filtering")。仓库内未发现 CHANGELOG.md/VERSION 文件(`find` 结果仅命中 `NOTICE.md`),无法从仓库自身文件判断长期发布节奏,如实标注"仅见单点时间戳,维护频率 unknown" |
| 漏洞 | 未发现仓库自带的已知问题登记文件(无 `KNOWN_ISSUES.md` 类文件),故不臆测,标注 unknown(未发现≠不存在,只是仓库自身文档未记录) |
| 安全风险(源码观察) | ①`gateway.py:21373` 与 `server.py:13635` 均有 `CORSMiddleware(allow_origins=["*"], allow_methods=["*"], allow_headers=["*"], expose_headers=["*"])`,注释自述是为了"让远程客户端(Cloudflare Tunnel / ngrok)能正常连接",随后紧跟一层 `OmbreChatGptOAuthMiddleware`,但未在本次审计范围内逐行确认该 OAuth 中间件是否覆盖所有写路径(与 ombre-brain 参考实现里显式的 `OriginCSRFGuardMiddleware` 相比,文档化程度更低)。②`scripts/sync_to_supabase.py` 硬编码了默认 Supabase 项目 URL 常量 `DEFAULT_SUPABASE_URL = "https://nuhbpesfpoywzcxlqfhs.supabase.co"`(L28),这是一个具体的、看起来指向真实生产项目的 URL 字符串(不是占位符),虽然真正的鉴权 `SUPABASE_SERVICE_KEY` 仍从环境变量读取、未硬编码,但硬编码真实项目域名本身属于信息暴露面,建议人工确认该 URL 是否为仍在使用的生产实例。③全仓库按 `(api_key|secret|password|token)\s*=\s*['"][A-Za-z0-9_\-]{12,}['"]` 模式扫描,**零命中**疑似硬编码密钥(已排除 env/占位符类假阳性)。 |
| 证据ID | haven-ombre@284c9c7: `LICENSE` 全文;`README.md` L1-6;`requirements.txt` 全文;`gateway.py:21373`;`server.py:13635,13620-13645`;`scripts/sync_to_supabase.py:28`;`git log -1 --format="%ai"` 输出 `2026-08-07 11:10:40 +0800` |
| 决策 | (留空,由人工裁决) |

### 单元2:ombre-brain(参考实现)

| 项 | 内容 |
|---|---|
| 项目 | ombre-brain |
| 仓库(full_name) | P0luz/Ombre-Brain(本地克隆源:`18358386529/ombre-brain`,公开 fork) |
| 许可证(自身 LICENSE) | MIT License,`Copyright (c) 2026 P0lar1zzZ`(原始项目,`AUTHORS.md` 列出开发组 9 人:Poluz/鹤见、江乔生、Iris Wang、Moling、Wudy、KittyXu、万世、Zoey、昕) |
| 依赖许可证 | Unknown,原因同 haven-ombre——未 vendor 任何依赖源码,`requirements.txt`/`requirements.lock.txt` 只有包名与版本号,无 LICENSE/NOTICE 文本可读 |
| 采用方式 | 原始项目(被 haven-ombre fork、被 xinchao-nian 内嵌拷贝的上游源头),自身不采用其他三个仓库的代码 |
| 维护状态 | 浅克隆仅见 1 条 HEAD 提交,时间戳 `2026-08-18 17:58:35 +0800`(commit message:"fix(docs+test): /mcp-extra 在役,文档改为双连接器")。仓库自带 `CHANGELOG.md`(30061 字节)、`AUTHORS.md`、`CONTRIBUTING.md`、`DCO`(Developer Certificate of Origin,签署要求见 `CONTRIBUTING.md` "提交前请签署 DCO" 一节)——这些文件的存在本身表明项目有相对规范的多人协作流程,但受浅克隆限制无法核实历史提交频率 |
| 漏洞 | 未发现仓库自带的 KNOWN_ISSUES/SECURITY 类文件明确列出已知漏洞,标注 unknown |
| 安全风险(源码观察) | ①`src/server_app.py:765` 同样存在 `allow_origins=["*"]`,但紧邻的类 `OriginCSRFGuardMiddleware`(L358-372)有明确注释说明设计意图:CORS 全开仅为了让 `claude.ai` 等浏览器端 MCP 客户端能带 Bearer token 跨域访问 `/mcp`,而 `/auth/*`、`/api/*` 等 cookie-session 路径由该中间件按 `Origin` 与 `Host` 是否匹配做二次拦截,`/mcp`、`/oauth/*`、`/.well-known/*` 因走 Bearer/PKCE 认证而被判定为"Origin 不匹配也无 CSRF 风险"而豁免。相比 haven-ombre,这里的设计文档化更完整,但本审计未逐行验证该中间件实现是否确实覆盖了所有状态变更路径。②测试文件(`tests/test_web_api_docker_integration.py`、`tests/test_release_audit_regressions.py`)中出现的 `"docker-dashboard-locked-body"`、`"original-token"` 等字符串经确认均为测试夹具(fixture)里的哨兵值,不是真实凭据。③全仓库同等模式扫描硬编码密钥,零命中真实凭据。 |
| 证据ID | ombre-brain@6f7335d: `LICENSE` 全文;`AUTHORS.md` L1-16;`DCO` L1-20;`CONTRIBUTING.md` L1-20;`CHANGELOG.md`(存在性,30061 字节);`src/server_app.py:358-372,765`;`tests/test_web_api_docker_integration.py:133,156,206`;`tests/test_release_audit_regressions.py:293`;`git log -1 --format="%ai"` 输出 `2026-08-18 17:58:35 +0800` |
| 决策 | (留空,由人工裁决) |

### 单元3:xinchao-nian(心智层,外层)

| 项 | 内容 |
|---|---|
| 项目 | xinchao-nian(外层联合发行包,不含内嵌的 `ombre-brain/` 副本——该副本单列为单元5) |
| 仓库(full_name) | tianyupaipai-cmd/xinchao-nian(本地克隆源:`18358386529/xinchao-nian`,公开 fork) |
| 许可证(自身 LICENSE) | 仓库根 `LICENSE` = **GNU AGPL-3.0**(与 haven-ombre/ombre-brain 的 MIT 完全不同的许可证族)。仓库内额外提供 `docs/LICENSING.md`,明确声明这是一个"多来源组成的联合发行包,不是所有文件共用同一份许可证":仓库根联合发行代码 = AGPL-3.0;`xinchao/` 子目录 = MIT(`xinchao/LICENSE`,`Copyright (c) 2026 Xinchao contributors`,`xinchao/package.json` 的 `"license": "MIT"` 字段与之一致);`bridge/` 子模块 = MIT(子模块内容未拉取,未能核实);`ombre-brain/` 原始部分 = MIT(`LICENSE.P0luz-MIT`);`ombre-brain/` 二改及衍生部分 = "个人/学习/非商业"(见单元5) |
| 依赖许可证 | `xinchao/package.json` 声明 `"license": "MIT"` 但未内嵌 `node_modules`(`find` 未发现),该 package.json 本身未列出任何 `dependencies`/`devDependencies` 字段(仅有 `name/version/description/type/license/scripts/engines/keywords`),即**该 Node 项目在此快照下没有任何第三方 npm 依赖声明**,依赖许可证问题不适用(N/A);`ombre-brain/` 子目录的 Python 依赖许可证 unknown,理由同单元1/2 |
| 采用方式 | 混合——外层 `xinchao/`(node,`xinchao-dynamic-mind` v3.3.6)是独立的动态心智引擎代码(算法+代码全采用,非仅参考);内嵌的 `ombre-brain/` 是整份代码的 vendored 拷贝(代码级采用,见单元5) |
| 维护状态 | 浅克隆仅见 1 条 HEAD 提交,时间戳为审计时最新(`2026-09-13 17:03:32 +0800`,commit message:"3.3.6:念头回推只推一次、上限 0.85,打破沉淀维闭环"),是四个仓库中 HEAD 时间戳最新的一个 |
| 漏洞 | 未发现仓库自带的 KNOWN_ISSUES/SECURITY 类文件,标注 unknown |
| 安全风险(源码观察) | `.env.example` 里多项变量明确标注"必填"且给出强度要求(如 `OAUTH_APPROVAL_TOKEN` 要求"至少 16 字符,不得复用其他 token"、`DASHBOARD_ACCESS_TOKEN` 要求"至少 32 字符"),文档化的密钥管理规范优于其他三个仓库;未发现硬编码密钥或弱默认口令。内嵌 `ombre-brain/src/server_app.py:381` 同样有 `allow_origins=["*"]`(继承自单元2/5,不重复计分) |
| 证据ID | xinchao-nian@97f1bdc: `LICENSE` 全文(AGPL-3.0);`docs/LICENSING.md` 全文(19 行);`xinchao/LICENSE` L1-3;`xinchao/package.json` 全文;`.gitmodules`;`.env.example` L5-51(节选);`git log -1 --format="%ai"` 输出 `2026-09-13 17:03:32 +0800` |
| 决策 | (留空,由人工裁决) |

### 单元4:kiwi-mem(算法参考,完整服务)

| 项 | 内容 |
|---|---|
| 项目 | kiwi-mem |
| 仓库(full_name) | LucieEveille/kiwi-mem(本地克隆源:`18358386529/kiwi-mem`,公开 fork) |
| 许可证(自身 LICENSE) | GNU AGPL-3.0,`Copyright (C) 2026 Lucie`。与 xinchao-nian 根目录同属 AGPL-3.0 许可证族,但两者是各自独立的版权声明,无继承关系的直接证据 |
| 依赖许可证 | Unknown——`requirements.txt` 共 11 行,列出 `fastapi==0.141.1`、`starlette==1.3.1`、`uvicorn==0.30.0`、`httpx==0.27.0`、`asyncpg==0.30.0`、`mcp>=1.8.0`、`python-multipart>=0.0.6`、`pypdf>=4.0.0`、`python-docx>=1.1.0`、`openpyxl>=3.1.0`、`jieba>=0.42.1`,均只有包名+版本号,未 vendor 任何依赖源码,无法在不联网前提下确认依赖包各自的许可证(注:`asyncpg` 的出现说明 kiwi-mem 自身连接了 PostgreSQL,但这是 kiwi-mem 自己的存储选型,与 haven-ombre/ombre-brain 的 embedding 存储无关,不构成对约束4的触碰,已在此列明确区分) |
| 采用方式 | 算法思想/仅参考——任务书原文将其定性为"算法参考,但仓库本身是完整服务"。经比对文件名列表(`anthropic_adapter.py`/`calendar_periods.py`/`config.py`/`daily_digest.py`/`database.py`/`dream.py`/`main.py`/`mcp_access.py`/`mcp_client.py`/`mcp_server.py`/`memory_extractor.py`/`tool_drawer.py`/`web_search.py`)与 haven-ombre 的文件名列表(`bucket_manager.py`/`dream_engine.py`/`entity_edges.py`/`embedding_engine.py`/`gateway.py`/`memory_edges.py`/`memory_moments.py`/`recall_policy.py` 等),**除 `tool_drawer.py` 命名相同外未见其余文件名重合**,`grep -rIl "embedding"` 在 kiwi-mem 命中的 8 个文件中也**未发现独立的 `EmbeddingEngine` 类或 `.embeddings.create(` 调用**(详见清单4报告"依赖链追踪叙述"第4点)。据此判断 kiwi-mem 与 haven-ombre/ombre-brain **不存在直接的代码复制关系**,更符合"独立实现、思路上参考"的定性,但受限于本次审计未逐行读完 main.py(281KB)/database.py(296KB)两个超大文件,不能完全排除内部存在借鉴代码片段,如实标注"结构性证据支持'算法思想/仅参考'定性,但未做逐行代码级排重,存在残余不确定性" |
| 维护状态 | 浅克隆仅见 1 条 HEAD 提交,时间戳 `2026-09-10 13:29:29 +0800`(commit message:"feat(prep): prepare 1.7.0 MCP access notices and safe upgrades (#81)")。仓库自带 `CHANGELOG.md`、`KNOWN_ISSUES.md`、`AGENTS.md`、`CLAUDE.md`,文档化程度是四个仓库中最高的;`CHANGELOG.md` 显示当前版本为"1.7.0 — Unreleased (2026-09)",且明确记录了"限时风险例外"式的运维决策(见下"漏洞"行),表明团队有主动的依赖/安全跟踪流程 |
| 漏洞 | **仓库自身文件明确记录了已知 CVE**(按约束6"仓库自身文件中明确写了漏洞说明"的例外条款如实转述,不做任何延伸推测):`CHANGELOG.md`"限时风险例外"一节原文:"1.7.0 暂留 `mcp==1.12.4`,其三条公告 CVE-2025-66416 / CVE-2026-52869 / CVE-2026-59950 仍在。理由:升 mcp ≥ 1.23 会让 SDK 对本机 host 自动开启 Host / Origin 保护、远程 MCP 在准备版就被拒,违背'先提醒再改规则'。解除条件:KIWI-BUILD-01 合入 `release/kiwi-sync` 并随 2.0.0 发布。"`KNOWN_ISSUES.md` 的"KIWI-PREP-01"一节有几乎相同的表述。**重要不一致**:仓库的 `requirements.txt` 第 6 行实际声明的是 `mcp>=1.8.0`(开放式下限,无上限),与 CHANGELOG/KNOWN_ISSUES 描述的"暂留 `mcp==1.12.4`"(精确锁定)**不一致**——本仓库未找到任何 `.lock` 文件或 `Pipfile.lock`/`poetry.lock`(`find` 零命中)来锁定实际安装版本,也就是说,如果只按 `requirements.txt` 全新安装,pip 会解析到 `>=1.8.0` 的**最新**可用版本,而不是文档所说"刻意暂留"的 `1.12.4`,与 CHANGELOG 里"限时风险例外"的假设前提(即环境里实际跑的是 1.12.4)矛盾。这是一处文档与依赖清单不一致,按约束7要求如实标注,不代为判断哪个是权威版本。 |
| 安全风险(源码观察) | ①`.env.example` 第 39 行示例值为 `CORS_ORIGINS=*`(通配符),但实际代码 `main.py:320` 的兜底默认值是 `os.getenv("CORS_ORIGINS", "http://localhost:5173,http://localhost:5174")`(非通配符)。即:**代码本身的默认值是安全的**(仅本地开发源),风险点在于 `.env.example` 模板把 `*` 写成了示例值,如果部署者直接复制 `.env.example` 内容为生产 `.env` 而未修改该行,会实际启用全通配 CORS。②`API_KEY=sk-your-api-key-here` 为标准占位符格式,非真实密钥。③按硬编码密钥模式扫描,`scripts/test_kiwi_safety_sync.py`、`scripts/test_calendar_summary_generation.py` 中出现 `"sk-VERY-LONG-SECRET-1234567890"`、`"sk-secret-should-never-log"`、`"PROMPT-PRIVATE-SENTINEL-..."` 等字符串,经上下文确认均为测试断言用的哨兵/夹具值(用于验证"密钥不应被记录到日志"这类安全测试本身),不是真实凭据。④未发现 `allow_origins`/`CORS_ORIGINS` 之外的默认弱口令或鉴权缺失模式(按 `auth.{0,10}=\s*False\|skip_auth\|AUTH_DISABLED` 等模式扫描零命中)。 |
| 证据ID | kiwi-mem@b01a0c5: `LICENSE` 全文;`CHANGELOG.md` L1-9(含"限时风险例外"段落);`KNOWN_ISSUES.md` 全文(含"KIWI-PREP-01"一节);`requirements.txt` 第6行 `mcp>=1.8.0`;`.env.example` L39;`main.py:319-325`;`scripts/test_kiwi_safety_sync.py:474-475,7981`;`scripts/test_calendar_summary_generation.py:198`;`git log -1 --format="%ai"` 输出 `2026-09-10 13:29:29 +0800` |
| 决策 | (留空,由人工裁决) |

### 单元5:xinchao-nian 内嵌 ombre-brain 副本(第5审计单元,双许可证重点核查对象)

| 项 | 内容 |
|---|---|
| 项目 | xinchao-nian/ombre-brain/(内嵌拷贝,非 git 子模块——`.gitmodules` 只声明了 `bridge`,未声明 `ombre-brain` 为子模块,`git -C ombre-brain rev-parse HEAD` 返回的是外层 xinchao-nian 自己的 commit `97f1bdc...`,`git -C ombre-brain remote -v` 指向 xinchao-nian 自己的 origin,证明这是**普通文件级拷贝合并进主仓库**,不是 submodule 引用) |
| 仓库(full_name) | 无独立 full_name(隶属 tianyupaipai-cmd/xinchao-nian 仓库路径下的子目录) |
| 许可证(自身 LICENSE) | **本目录内同时存在两份命名不同的许可证文件,且内容与文档描述之间存在重大不一致,是本次审计的核心发现**:<br>1. `LICENSE.P0luz-MIT`(1087 字节,CRLF 换行,`md5=ecb88dbb...`)——标准 MIT License 全文,`Copyright (c) 2026 P0lar1zzZ`。<br>2. `LICENSE.CyberSealNull`(1066 字节,LF 换行,`md5=d03735ab...`)——**逐字逐句阅读后确认其正文与 `LICENSE.P0luz-MIT` 完全相同,同样是标准 MIT License 全文,同样的版权人 `P0lar1zzZ`,一个字都没有增加"非商业"或任何限制性条款**。两个文件唯一的物理差异是换行符(`LICENSE.P0luz-MIT` 用 `\r\n`,`LICENSE.CyberSealNull` 用 `\n`),这也是两文件字节数不同(1087 vs 1066,恰好等于 21 个换行符 × 1 字节的 `\r` 差)、但 `diff` 逐行对比内容"看起来相同"的原因。<br>3. 该目录另有 `NOTICE.CyberSealNull.md`(独立文件,非许可证文件本体),其中才真正写明了"非商业"限制条款原文:"The additional code, documentation, configuration examples, and derivative features added in this fork are licensed for personal learning, private use, and non-commercial modification/deployment only." 并列举了禁止的商业用途清单(付费转售、付费部署服务、SaaS、付费课程/咨询交付、商业产品集成)。<br>4. `MODIFICATIONS.md` 用血统图描述:"P0luz/Ombre-Brain(原项目,MIT)→ CyberSealNull 二改(在原 MIT 之上追加'新增内容非商业'约束)→ 心潮念(本仓库)",并在文末声明"心潮念整体非纯 MIT:`ombre-brain/` 部分受 CyberSealNull 二改的非商业约束 + P0luz 原 MIT 的署名/许可保留要求约束"。 |
| 依赖许可证 | Unknown,理由同其余单元——`requirements.txt`/`requirements.lock.txt` 只有包名版本号,未 vendor 依赖源码 |
| 采用方式 | 代码——整份 `src/`、`deploy/`、`frontend/`、`docs/`、`entrypoint.sh`、`config.default.yaml` 等均为完整功能代码的拷贝合并,`src/embedding_engine.py` 与上游 ombre-brain@6f7335d 同名文件 `diff` 后仍有 526 行差异(非零改动,证明心潮念团队确实对拷贝进来的代码做了二次修改,而不是只读镜像) |
| 维护状态 | `VERSION` 文件内容为 `2.6.5`(单行,无日期);`CHANGELOG.md` 存在(30061 字节,与上游 ombre-brain 的 CHANGELOG.md 大小接近,unknown 是否为同一份内容的历史快照,未逐行比对);`MODIFICATIONS.md` 明确记录了心潮念团队做的 5 项具体改动(breath-meta 依赖、压缩模型默认从 GLM-Z1 换 DeepSeek-V3、bucket-map 星图接口、MCP 工具 annotations、grow 去重),这是四个审计单元里唯一一份**逐条列出"我们改了什么"的变更记录**,文档化质量较高 |
| 漏洞 | 未发现该目录自带 KNOWN_ISSUES/SECURITY 文件,标注 unknown;继承自上游 ombre-brain 的代码路径若存在漏洞,风险同单元2但未重复列出 |
| 安全风险(源码观察) | 继承单元2的 `allow_origins=["*"]`(`src/server_app.py:381`,行号相对上游 `L765` 有偏移,印证该文件确有改动);另需指出:`requirements.txt` 里 `mcp` 版本约束从上游的 `mcp>=1.27,<2`(注释:"生产锁定 1.28.1,限制在 v1,避免未来 v2 的破坏性 API 变更越过锁文件安装进来")放宽为 `mcp>=1.0.0`(无上限),而其自带的 `requirements.lock.txt` 却锁定了 `mcp==1.28.1`(与上游实际锁定版本相同)、但 `openai==2.45.0`(**低于**上游锁定的 `openai==2.52.0`)。这意味着**若脱离 lock 文件、仅按 `requirements.txt` 全新安装,该内嵌副本可能装出与上游不同甚至跨大版本的 `mcp` SDK**,而上游特意加的 `<2` 上限保护在此处已经丢失,属于依赖声明层面的回归,建议人工确认是否为有意为之 |
| 证据ID | xinchao-nian@97f1bdc: `ombre-brain/LICENSE.P0luz-MIT` 全文(`wc -c`=1087,`md5sum`=ecb88dbbadd0ecc4821485835500199c);`ombre-brain/LICENSE.CyberSealNull` 全文(`wc -c`=1066,`md5sum`=d03735ab43f9a2c1f786137a354a60c6);`cat -A` 逐字节比对结果(两文件除 `^M`/CRLF 外文本完全一致);`ombre-brain/NOTICE.CyberSealNull.md` 全文(19 行);`ombre-brain/MODIFICATIONS.md` 全文(60 行);`ombre-brain/VERSION`;`ombre-brain/requirements.txt` 与 ombre-brain@6f7335d `requirements.txt` 的 `diff` 结果(mcp 约束行差异);`ombre-brain/requirements.lock.txt` 第341行(`mcp==1.28.1`)、第399行(`openai==2.45.0`);`ombre-brain/src/server_app.py:381`;`ombre-brain/src/embedding_engine.py` 与上游同名文件 `diff` 输出行数(526 行) |
| 决策 | (留空,由人工裁决) |

---

## 发现的冲突与待人工裁决项

1. **xinchao-nian 内嵌 ombre-brain 副本的"双许可证"实为"命名误导+范围外文档补充",而非两份内容不同的许可证文本(本次审计最重要的发现)。** 逐字节比对(`wc -c` + `md5sum` + `cat -A`)证实 `LICENSE.CyberSealNull` 与 `LICENSE.P0luz-MIT` 的可读文本**逐句相同**,均为无限制条款的标准 MIT License,唯一物理差异是换行符风格(CRLF vs LF)。但 `MODIFICATIONS.md` 和 README.md 都对外声称"CyberSealNull 二改在原 MIT 之上追加了'新增内容非商业'约束",这个约束的**真实法律文本其实只出现在 `NOTICE.CyberSealNull.md` 这个独立文件里**,而不在名为 `LICENSE.CyberSealNull` 的文件本体内。换言之:①如果只看文件名和目录里两份"LICENSE.*"文件,会误以为限制条款写在其中一份 LICENSE 里,但实际打开后发现两份都是纯 MIT;②真正的非商业限制条款的法律效力边界(NOTICE 性质的文档能否构成有强制力的"许可条款"、还是仅为友善提示)存在争议空间,`ombre-brain/CONTRIBUTING.md`(上游文件)里有一句耐人寻味的对照:"另有一份 NOTICE.md 写了关于署名的请求,但那不是条款,没有约束力"——如果 CyberSealNull fork 的 `NOTICE.CyberSealNull.md` 被理解为与上游 NOTICE.md 同一性质的"没有约束力的请求",那么"新增内容非商业"这条约束的可执行性就存疑;反之如果 `MODIFICATIONS.md`/`docs/LICENSING.md` 的合同性表述("商用需取得上游书面许可")被认定为有效的再许可条件,则心潮念团队及下游任何再分发方都需要遵守。**此为纯法律定性问题,已如实列出全部原始文本证据,不做倾向性判断,提请人工(建议法务)裁决。**
2. **kiwi-mem 的 CVE 例外声明与 `requirements.txt` 实际声明的版本范围不一致。** `CHANGELOG.md`/`KNOWN_ISSUES.md` 明确记载"1.7.0 暂留 `mcp==1.12.4`"并列出三个 CVE 编号(CVE-2025-66416、CVE-2026-52869、CVE-2026-59950)作为已知风险的限时豁免,但 `requirements.txt` 第 6 行实际写的是开放式的 `mcp>=1.8.0`,仓库内又没有任何 lock 文件锁定精确版本。这意味着**文档描述的"我们刻意留在有 CVE 的旧版本"这一前提,在当前依赖清单下可能并不成立**(全新安装大概率会装到修复了这些 CVE、但也改变了 Host/Origin 保护行为的更新版本)。这一差异直接关系到"记忆系统2.0若要复用/参考 kiwi-mem 的部署方式"时,应该按文档说的"预期用 1.12.4"来处理,还是按 `requirements.txt` 实际写的"任何 ≥1.8.0"来处理,请人工核实 kiwi-mem 生产环境实际安装的 mcp 版本后再定。
3. **xinchao-nian 内嵌 ombre-brain 副本与上游 ombre-brain 参考实现的依赖声明已产生漂移**,`mcp` 约束从上游的 `mcp>=1.27,<2` 松绑为 `mcp>=1.0.0`,上游特意加的"避免 v2 破坏性变更"防护在内嵌副本里已经消失;两者的 `requirements.lock.txt` 又分别锁定了不同的 `openai` 次版本(2.52.0 vs 2.45.0)。这对"记忆系统2.0"若要以 ombre-brain 或其内嵌副本为参考基线,存在"到底以哪一份依赖声明为准"的问题,请人工确认。
4. **License 许可证族本身存在层层嵌套的强弱混合**:xinchao-nian 根目录是 Copyleft 性质较强的 AGPL-3.0,其内部却嵌套了 MIT(`xinchao/`、`ombre-brain/` 原始部分)与非正式的"个人学习/非商业"限制(`ombre-brain/` 二改部分)。`docs/LICENSING.md` 已经对此做了目录级别的拆分说明,内部文档本身是自洽的,但**这种"仓库根 AGPL + 内部子目录不同许可证"的联合发行模式,在被记忆系统2.0引用/再分发时如何合规处理,超出源码审计范畴**,提请人工(法务)确认最终的再分发/引用策略。
5. **kiwi-mem 与 haven-ombre/ombre-brain 之间的代码复制关系未能 100% 排除**:基于文件名比对和 `embedding` 关键词扫描的结构性证据支持"kiwi-mem 是独立实现、仅算法思想借鉴"的定性,但由于 `main.py`(281KB)、`database.py`(296KB)两个超大文件未逐行读完,不能完全排除内部存在与其他三个仓库同源的代码片段。如需要严格的 License 合规结论,建议后续安排专项的逐文件相似度比对(如 diff/相似度哈希),本次审计受时间预算限制未执行。
6. **haven-ombre 硬编码的 Supabase 项目 URL**(`DEFAULT_SUPABASE_URL = "https://nuhbpesfpoywzcxlqfhs.supabase.co"`)是否仍指向在用的生产实例,以及该 URL 出现在公开 fork 仓库里是否构成信息暴露,请人工核实并按需在生产配置中改为纯环境变量注入(不在源码里保留默认真实域名)。
