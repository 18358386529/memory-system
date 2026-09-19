# 清单6：仓库内配置文件与备份脚本审计（源码审计版）

## 审计模式声明

**本文档为源码审计（static source audit），不是运行时/压测审计。** 表格中"峰值/延迟/磁盘/内存/CPU"等实测类字段，一律填 `unknown(源码审计模式，无法实测，16GB Mac mini 实测需人工在真实环境执行)`。文档/注释/README 中出现的具体数字（如 Render `sizeGB: 5`、`mem_limit: 128m`）均为**仓库内声明的配置值**或**部署脚本预设的资源上限**，不是实测的运行时峰值，报告中会明确标注其性质（"声明值/配置上限"而非"实测值"）。

## 四个仓库 commit SHA（与清单1一致）

| 本地路径 | 参考仓库 | 锁定 SHA(用户提供，经 `git rev-parse HEAD` 核对一致) |
|---|---|---|
| /home/user/18358386529/haven-ombre | Yinglianchun/Haven-Ombre | 284c9c7b0e51a0ba0032c7028f705d72458cb304 |
| /home/user/18358386529/ombre-brain | P0luz/Ombre-Brain | 6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5 |
| /home/user/18358386529/xinchao-nian | tianyupaipai-cmd/xinchao-nian | 97f1bdcc76b748fa516b7c79a0aa10234143795f |
| /home/user/18358386529/kiwi-mem | LucieEveille/kiwi-mem | b01a0c506f4f10f90f30d408c0291f16720f0718 |

## 证据ID 格式说明

`EV-06-<序号>` 形式，证据格式：`路径[:行号] @ <仓库简称>:<commit SHA前12位>`。凡涉密钥/Token/明文密码，只报告"存在此配置项/默认示例值的存在事实"，本文档中出现的 `kiwi_secret_change_me` 是 `docker-compose.yml` 里的**示例/占位符密码**（文件里字面写着"change_me"，本身就是提示该值必须替换，不是真实生产密钥），按约束6判断此处报告该占位符字符串本身不违反"不得输出真实值"的限制，因为它不是任何真实凭证。

---

## 备份/恢复/资源 表

| 项目 | 指标 | 峰值 | 延迟 | 磁盘 | 内存 | CPU | 备份路径/脚本(源码内证据) | 校验(源码内是否有hash校验逻辑) | 恢复验证(源码内是否有restore/replay逻辑及其测试覆盖) | 证据ID |
|---|---|---|---|---|---|---|---|---|---|---|
| haven-ombre | Render 磁盘声明 | unknown(源码审计模式，无法实测，16GB Mac mini 实测需人工在真实环境执行) | unknown(同上) | **声明值(非实测)**：`render.yaml` 声明 `disk.sizeGB: 1`，挂载于 `/opt/render/project/src/buckets` | unknown(源码审计模式，无法实测) | unknown(同上) | 未发现独立的"备份脚本"（`scripts/` 下 23 个脚本均为迁移/清理/审计类，如 `migrate_bucket_files.py`/`cleanup_duplicate_buckets.py`/`cleanup_orphan_embeddings.py`，无 `backup_*.py` 或 `*_backup.sh` 命名的脚本） | 无（未发现 hash 校验相关代码路径） | 无独立 restore 脚本；`import_memory.py`(73894字节)是"导入"而非"从备份恢复"语义，未在文件名/职责上明确标注为备份恢复用途 | EV-06-01 (`render.yaml`全文；`ls scripts/`列出的23个文件名 @ haven-ombre:284c9c7) |
| haven-ombre | Supabase 云同步(可选，非默认启用) | unknown | unknown | N/A（远程 Supabase 托管，本地无磁盘占用声明） | unknown | unknown | `scripts/sync_to_supabase.py`(21344字节，双向同步 CLI，默认 dry-run，需 `--apply` 才写入)；`scripts/supabase_memory_rpc.sql`(5895字节，定义 `public.memories` 表的 RPC 函数/触发器) | **有**：`sync_to_supabase.py` 内 `sync_fingerprint()` 函数（`sync_to_supabase.py:132-135`）对每条记录的同步字段做 `hashlib.sha256` 指纹计算，用于判断本地/远端记录是否发生变化，属于**数据一致性指纹**而非"备份完整性校验"，两者用途不同需明确区分 | 该脚本本身即为"双向同步"，`apply_pull()`/`apply_delete_local()` 函数实现了"从远端拉回本地"的逻辑（这可被视为一种恢复路径），但仓库内**未发现**任何针对 `sync_to_supabase.py` 的测试文件（未搜到 `test_sync_to_supabase*.py`），恢复路径无测试覆盖 | EV-06-02 (`sync_to_supabase.py`全文，尤其 `:34` `DEFAULT_SUPABASE_URL = "https://nuhbpesfpoywzcxlqfhs.supabase.co"`(硬编码默认URL，非密钥，按约束6只报告存在此配置项)；`:132-135` sync_fingerprint()；`main()`函数 @ haven-ombre:284c9c7) |
| ombre-brain | 本地 zip 备份归档 | unknown | unknown | **声明的容量上限(非实测)**：`backup_archive.py` 硬编码常量 `MAX_ARCHIVE_BYTES = 512 * MIB`、`MAX_MEMBERS = 10_000`、`MAX_TOTAL_UNCOMPRESSED_BYTES = 1024 * MIB`、`MAX_COMPRESSION_RATIO = 1000.0`（防 zip-bomb） | unknown | unknown | `src/ombrebrain/storage/backup_archive.py`（"Markdown remains the source of truth. The SQLite file is only a derived-index snapshot"），核心函数：`build_export_archive`/`build_export_archive_file`(导出)、`read_backup_archive`/`extract_backup_archive_file`(读取/解压恢复)；对应测试 `tests/test_backup_archive.py`、`tests/test_backup_import_safety.py` | **有，且较完整**：`MANIFEST_NAME = "backup_manifest.json"`，`_build_manifest()`生成清单，`_sha256()`对每个成员计算哈希，`_verify_manifest()`(`:634`)校验清单与实际文件是否一致；`_normalize_member_path()`防路径穿越(zip slip)攻击 | **有**：`extract_backup_archive_file`(`:684`)实现恢复逻辑，且有专门的安全测试文件 `test_backup_import_safety.py`（本审计确认该测试文件存在，未逐条阅读其全部用例内容，仅确认其命名与 backup_archive.py 的安全校验函数一一对应） | EV-06-03 (`backup_archive.py:1-56`(docstring+常量)；`:634`_verify_manifest；`:684`extract_backup_archive_file；`tests/test_backup_archive.py`、`tests/test_backup_import_safety.py` 文件存在 @ ombre-brain:6f7335d) |
| ombre-brain | GitHub 云备份(可选，需配置 token) | unknown | unknown | N/A（远程 GitHub 仓库托管） | unknown | unknown | `src/github_sync.py`（docstring明确："GitHub 仓库同步（用于 bucket 数据云端备份）"；"embeddings.db 不上传...可由 /api/embedding/migrate 重算"；"使用 GitHub Git Trees API 批量提交（一次同步 = 一个 commit）"；"支持手动触发 + 可选的定时自动同步"）；对应测试 `tests/test_github_backup_manifest.py`、`tests/test_github_backup_alarm.py` | **有**：`GitHubSync._build_backup_manifest()` 生成含 `schema_version`/`source`/`repo`/`branch`/`file_count`/`total_bytes`/每文件 `sha256`/`bytes` 的清单（测试 `test_backup_manifest_records_hashes_counts_and_bytes` 直接断言 `hashlib.sha256(b"alpha").hexdigest()` 与清单内记录的哈希一致） | **有告警机制但恢复逻辑本身证据不足**：`test_github_backup_alarm.py` 验证"连续失败计数"逻辑（`consecutive_failures`字段，成功后归零、失败累加，供诊断面板告警），这是**失败监控**而非"恢复(restore)"逻辑；本审计在 `github_sync.py` 摘取的部分未见明确的"从 GitHub 拉回并还原到 buckets_dir"的 restore 函数名，是否存在"反向拉取恢复"能力需要人工进一步阅读 `github_sync.py` 全文(本次仅读取了文件头部约40行)确认，此处标记为 `unknown(需读取 github_sync.py 全文确认是否含反向 restore 路径)` | EV-06-04 (`github_sync.py:1-40`(docstring)；`tests/test_github_backup_manifest.py:1-40`；`tests/test_github_backup_alarm.py`全文 @ ombre-brain:6f7335d) |
| ombre-brain | 发布侧代码完整性清单(非用户数据备份，是热更新防篡改) | unknown | unknown | N/A | unknown | unknown | `deploy/gen_update_manifest.py`（"生成 update_manifest.json（热更新完整性清单）"；"`/api/do-update` 已经内置...逐文件核对 sha256/size，不符则整体中止、不落盘"） | **有**：脚本核心即基于 `hashlib` 逐文件计算 sha256 并写入 `update_manifest.json`（仓库根目录确认存在该文件，34673字节），供运行时热更新前做完整性校验 | 这是"防止热更新写入坏代码"的校验机制，不是"数据备份的恢复验证"，两者目的不同，明确区分记录避免混淆 | EV-06-05 (`deploy/gen_update_manifest.py:1-60`全文；根目录 `update_manifest.json` 文件存在(34673字节) @ ombre-brain:6f7335d) |
| xinchao-nian(dynamic-mind, `xinchao/`) | 资源上限(compose 声明) | unknown | unknown | N/A(未声明磁盘上限，仅命名卷 `xinchao-state`，容量未在 compose 中限制) | **声明的容器内存上限(非实测)**：`compose.yaml` 中 `mem_limit: 128m` | **声明的容器CPU上限(非实测)**：`compose.yaml` 中 `cpus: 0.50` | 未发现独立备份脚本；持久状态依赖命名卷 `xinchao-state`(挂载于 `/app/state`，存放 `state.json`/oauth)，未见针对该卷的导出/备份工具 | 无（未发现该组件内的 hash 校验逻辑） | 无（未发现 restore/replay 相关代码或测试） | EV-06-06 (`compose.yaml:70-99`(mem_limit/cpus/logging/healthcheck/volumes 段) @ xinchao-nian:97f1bdc) |
| xinchao-nian(内嵌 ombre-brain 副本) | 与独立 ombre-brain 相同类别的本地备份能力 | unknown | unknown | 命名卷 `ombre-buckets`(compose声明，未限容量) | unknown | unknown | 内嵌副本自带 `entrypoint.sh`(13893字节，比独立仓库的17322字节略短，说明版本不同导致脚本内容有差异，未逐行 diff)；因 VERSION=2.6.5，早于独立仓库的包迁移完成节点，其 `src/backup_archive.py` 仍在旧的**扁平路径**（非 `src/ombrebrain/storage/backup_archive.py`），需要单独确认其内部实现是否与独立仓库当前版本一致 | unknown(未对内嵌副本的 `src/backup_archive.py` 做逐行 hash 校验函数核实，仅通过文件列表确认该文件存在于旧路径) | unknown(同上，未展开阅读内嵌副本 `src/backup_archive.py` 全文，仅确认文件存在性；由于是 2.6.5 旧版本，不能假设其恢复逻辑与独立仓库 3.2.0 版本完全相同) | EV-06-07 (`xinchao-nian/ombre-brain/src/backup_archive.py` 文件存在于 diff 输出的"内嵌副本独有文件"列表中；`entrypoint.sh` 字节数对比 13893 vs 17322 @ xinchao-nian:97f1bdc 与 ombre-brain:6f7335d) |
| kiwi-mem | PostgreSQL 数据卷 | unknown | unknown | **声明值(非实测)**：`docker-compose.yml` 中 `kiwi-pg-data` 具名卷持久化 `/var/lib/postgresql/data`，未设容量上限 | unknown | unknown | 未发现 pg_dump/pg_restore 或类似的数据库级备份脚本；`scripts/ledger_reconcile.py`(1838字节)是**只读对账工具**（docstring 明确"Read-only W2-05 event-ledger reconciliation CLI"，调用 `database.reconcile_event_ledger()`），**不是备份/恢复脚本**，只做"审计账本一致性、不写入" | 无 hash 校验逻辑（`ledger_reconcile.py` 全文未见 `hashlib` 调用） | **无 restore/replay 逻辑**：`ledger_reconcile.py` 明确是只读审计工具，不涉及数据恢复；仓库内也未找到其他命名含"backup"/"restore"的脚本 | EV-06-08 (`ledger_reconcile.py`全文(46行，已完整读取) @ kiwi-mem:b01a0c5；`docker-compose.yml:5-14` @ kiwi-mem:b01a0c5) |
| kiwi-mem | 硬性 Postgres 依赖(非备份范畴，但影响资源审计基线) | unknown | unknown | 依赖 `pgvector/pgvector:pg16` 第三方镜像的自身磁盘占用，未在本仓库声明 | unknown | unknown | N/A(非备份脚本，记录为资源依赖事实) | N/A | N/A | EV-06-09 (`config.py`/`database.py:29,34,67,75-80` `DATABASE_URL` 未设置时 `raise RuntimeError("DATABASE_URL 未设置！")`，`asyncpg.create_pool()` 为唯一连接路径，无 SQLite/内存回退分支 @ kiwi-mem:b01a0c5) |

---

## 发现的冲突与待人工裁决项

### 1. xinchao-nian 内嵌 ombre-brain 副本（详见 `01_runtime_architecture.md` 冲突1，此处仅从"备份能力"角度补充）

内嵌副本（VERSION 2.6.5）与独立仓库（VERSION 3.2.0）的备份相关代码**不能假定等价**：独立仓库的 `backup_archive.py` 已迁移至 `src/ombrebrain/storage/backup_archive.py` 包路径并在文档中记录了迁移历史（2.7.8—2.7.10 观察期，2.8.2 审计后退役旧兼容层），而内嵌副本仍停留在迁移前的扁平路径 `src/backup_archive.py`。这意味着：**如果内嵌副本被实际部署使用，其备份/恢复代码的成熟度、bug 修复状态可能落后于独立仓库当前版本**，两者不应被视为"同一份代码，只是路径不同"。这是否会造成"心潮念用户以为自己用的是最新版 OB 备份机制，实际运行的是 2.6.5 时期的旧实现"的风险，需要人工核实内嵌副本的 `src/backup_archive.py` 与独立仓库 `src/ombrebrain/storage/backup_archive.py` 之间的具体代码差异（本次审计仅确认二者路径不同、版本号不同，未做逐行 diff，工作量超出本清单范围，建议作为后续专项任务）。

### 2. haven-ombre 的 Supabase / PostgreSQL 依赖 —— 确认存在，性质为"可选、非默认启用的手动 CLI 工具"，与约束4 的冲突程度需人工裁决

**核查结果（有证据支持）**：

- `haven-ombre/scripts/sync_to_supabase.py` 确实存在，且明确是与 Supabase（托管 PostgreSQL + PostgREST）交互的双向同步脚本。
- **连接方式**：脚本通过 `urllib.request` 直接调用 Supabase 的 **REST API**（`SupabaseClient` 类，基于 HTTP），**不是**通过 `psycopg2`/`asyncpg` 等原生 PostgreSQL 驱动直连数据库。`requirements.txt` 全文核对（29行）**未包含**任何 Postgres 驱动包（无 `psycopg2`、`asyncpg`、`supabase-py` 等依赖），说明这条 Supabase 路径不会给 haven-ombre 的常规运行引入 Postgres 客户端库依赖。
- **是否默认启用**：**不是**。证据：
  1. 该脚本未被 `Dockerfile`、`docker-compose.yml`、`docker-compose.user.yml`、`compose.hk.yml`、`render.yaml` 中的任何启动命令、CMD、cron、healthcheck 或 entrypoint 引用（本审计对全部 `.py`/`.sh`/`.yml`/`.yaml` 做了 `grep -rln "sync_to_supabase"`，除 `.git/index`（git 内部索引记录该文件存在，非调用证据）外，**没有任何其他文件引用/调用它**）。
  2. 脚本本身默认是 **dry-run 模式**（`main()` 函数：`if not args.apply: print("Dry run only..."); return 0`），必须显式传 `--apply` 才会写入。
  3. 脚本运行前置条件是环境变量 `SUPABASE_SERVICE_KEY` 必须手动设置，否则直接报错退出（`if not service_key: print("ERROR..."); return 2`）——这是一个需要人工主动配置密钥才能工作的独立运维工具，不是自动挂载的生产路径。
- **代码内硬编码了一个默认 Supabase 项目 URL**：`DEFAULT_SUPABASE_URL = "https://nuhbpesfpoywzcxlqfhs.supabase.co"`。按约束6，此处只报告"存在此硬编码默认URL配置项"这一事实，不判断该项目当前是否仍在使用、是否为真实生产环境、也不去访问它（本审计全程未访问网络）。**这本身构成一个待人工裁决的问题**：为什么参考文档/示例脚本里会硬编码一个看起来具体的项目 URL，而不是用占位符（如 `https://your-project.supabase.co`）？是否意味着这曾经是（或仍是）某个真实使用中的 Supabase 项目，需要人工确认并视情况考虑是否应从仓库历史中清理。
- `config.example.yaml` 全文搜索 `supabase|postgres|psycopg|pgvector`：**零命中**，说明 Supabase 同步不是通过主配置文件驱动的功能，是完全独立于主配置体系之外的辅助脚本。
- 另有 `scripts/supabase_memory_rpc.sql`，定义了 `public.memories` 表的多个 schema 变更（`alter table ... add column if not exists ...`）与 RPC 函数（`create_memory` 等），这证明 Supabase 侧存在一张结构化的 `memories` 表，且这套 schema 是随 haven-ombre 项目演进而多次变更过的（脚本内有多次 `drop function if exists ... create or replace function ...` 的历史演进痕迹）。

**待人工裁决的核心问题**：
1. 约束4"禁止引入 PostgreSQL/pgvector/第二套 embedding"——`sync_to_supabase.py` 所连接的 Supabase 本质上就是托管 PostgreSQL。这条依赖**并非本次审计引入**，而是 haven-ombre 仓库既有源码（截至锁定 commit 284c9c7）中就已存在的功能。它是否违反约束4，取决于约束4 的适用范围是"审计/后续开发工作不得新增"，还是"整个记忆系统2.0项目最终形态不得包含"——这是范围解释问题，非本审计能替代人工判断的技术问题，故如实记录为待裁决项，不擅自下结论说"不违规"也不擅自下结论说"违规"。
2. 即便该脚本"默认不启用"，只要它作为文件存在于将要被引用/二改为 Memory Core 本体的 haven-ombre 仓库中，就存在"未来某次配置疏忽或有意为之下被启用，从而让 Memory Core 静默产生一条 Postgres 依赖"的风险，建议人工评估是否需要在正式采用 haven-ombre 作为 Memory Core 本体前，显式移除或隔离该脚本与对应 SQL 文件。

### 3. "v3 架构"命名歧义 —— 见 `01_runtime_architecture.md` 冲突3，结论为纯粹文字巧合，不影响资源/备份审计结论，此处不重复。

### 4.（新发现）kiwi-mem 对 PostgreSQL+pgvector 的依赖是"硬编码强制"而非"可选"

任务说明中的线索称 kiwi-mem "本身是完整可部署服务"，本审计在此基础上进一步确认了一个更具体的事实：kiwi-mem 的 `database.py`（296045字节的单文件巨石模块）通过 `asyncpg` 连接 PostgreSQL，且**没有任何 SQLite/内存/文件系统回退分支**——`DATABASE_URL` 未设置时代码直接 `raise RuntimeError`。这意味着：如果"记忆系统2.0"计划"只移植 kiwi-mem 的算法思想（Heat/Decay/Reheat/Ranking/Injection Level）"，而算法逻辑与数据库读写在 `database.py` 中高度耦合在同一文件里，**从工程角度提取"纯算法"而不牵连 PostgreSQL 依赖，需要额外的代码拆分/重写工作**，不是简单的"复制粘贴几个函数"就能做到。本审计只报告耦合的事实，不对"是否可行/该如何拆分"做设计建议或裁决，这属于后续工程规划范畴。

---

## 未能完成的核查项（如实列出，不编造）

1. `xinchao-nian/bridge/` 子模块因未初始化（本地为空目录）而完全无法审计其内容；按约束8禁止联网，本次无法获取。若后续审计需要覆盖它，需要人工在允许联网的环境下执行 `git submodule update --init` 后重新审计。
2. `ombre-brain/entrypoint.sh` 全文较长（17322字节），本次仅读取了约100行的"配置文件安全校验"部分，未读取"持久化代码 bootstrap"之后到文件结尾的全部内容（包含滚动升级/回滚逻辑的关键部分被部分截取），可能遗漏与备份/回滚相关的细节，标记为 `unknown(entrypoint.sh 全文未完整读取)`，建议人工补充审计。
3. `ombre-brain/src/github_sync.py` 全文较长，本次仅读取文件头部约40行的模块 docstring 与 import 部分，未确认其是否含有"从 GitHub 拉回并还原到本地 buckets_dir"的反向 restore 函数，已在上表标注为 `unknown`，需要人工补充审计全文。
4. kiwi-mem 的 `database.py`（296045字节）体量巨大，本次仅做了关键字定位（`DATABASE_URL`/`asyncpg`）级别的审计，未通读全文寻找是否存在其他潜在的备份/导出相关函数（如是否有 `pg_dump` 子进程调用等），标记为 `unknown(database.py 全文未通读，仅做关键字检索)`。
5. 独立仓库 `ombre-brain` 与内嵌副本 `xinchao-nian/ombre-brain` 的 `src/backup_archive.py` 未做逐行 diff（仅确认文件路径归属与版本号差异），见冲突1的补充说明。
