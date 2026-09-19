# 清单1：Runtime Architecture（源码审计版）

## 审计模式声明

**本文档为源码审计（static source audit），不是运行时审计。** 四个目标系统在本次审计执行环境中没有本地运行实例（未 `docker up`、未 `python server.py`、未启动任何进程）。所有"进程/端口/PID"相关结论均来自对 Dockerfile、docker-compose\*.yml、render.yaml、zbpack.json、entrypoint.sh、源码内端口常量的**静态阅读**，不是对运行中系统的观测。凡表格要求"实测"字段，一律填 `unknown(源码审计模式，无本地运行实例，无法实测)`，不得以文档声明替代实测结论。

审计执行时间：2026-09-19（容器内文件 mtime 显示为该日期，为只读克隆落盘时间，非仓库原始提交时间）。

## 四个仓库 commit SHA 核对表

| 本地路径 | 参考仓库 | 用户提供的锁定 SHA | 本次 `git rev-parse HEAD` 实测结果 | 是否一致 |
|---|---|---|---|---|
| /home/user/18358386529/haven-ombre | Yinglianchun/Haven-Ombre | 284c9c7b0e51a0ba0032c7028f705d72458cb304 | 284c9c7b0e51a0ba0032c7028f705d72458cb304 | 一致 |
| /home/user/18358386529/ombre-brain | P0luz/Ombre-Brain | 6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5 | 6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5 | 一致 |
| /home/user/18358386529/xinchao-nian | tianyupaipai-cmd/xinchao-nian | 97f1bdcc76b748fa516b7c79a0aa10234143795f | 97f1bdcc76b748fa516b7c79a0aa10234143795f | 一致 |
| /home/user/18358386529/kiwi-mem | LucieEveille/kiwi-mem | b01a0c506f4f10f90f30d408c0291f16720f0718 | b01a0c506f4f10f90f30d408c0291f16720f0718 | 一致 |

四者均无差异，本报告即按用户给定 SHA 审计当前实际 HEAD（二者相同）。

## 证据ID 格式说明

`EV-01-<序号>` 形式，具体证据附在括号内，格式为：`路径[:行号或行区间] @ <仓库简称>:<commit SHA前12位>`。所有证据均可用 `git -C <repo> show <sha>:<path>` 或直接 `Read` 工具复核。凡涉及密钥/Token 的证据，仅报告"存在此配置项/环境变量名"，不输出真实值；本次审计未发现任何仓库内硬编码真实密钥值（仅发现一个硬编码的 Supabase 项目 URL 常量，见清单6及冲突项，URL 本身不是密钥）。

---

## 一、haven-ombre（Yinglianchun/Haven-Ombre fork，声明定位：二改后作为 Memory Core 本体）

### 部署/组件表

| 组件 | 进程名(代码/镜像名) | PID | 用户 | 端口(声明值) | 路径 | 启动方式(声明) | 配置 | 版本/提交 | 哈希 | 证据ID | 备注 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 主服务(MCP server) | `ombre-brain`(Dockerfile 内 CMD) | unknown(源码审计模式，无本地运行实例，无法实测) | unknown(同上) | 容器内 8000；`docker-compose.yml` 宿主映射 18001:8000；`docker-compose.user.yml` 映射 8000:8000；`render.yaml` 未显式声明端口(Render 由 `$PORT` 注入，代码内硬编码 `port=8000`) | `/home/user/18358386529/haven-ombre/server.py`(556025字节，单文件巨石) | `CMD ["python", "server.py"]`（Dockerfile）；本地一键脚本 `./ob` → `scripts/one_click.sh` | `config.example.yaml`(36873字节，需复制为 `config.yaml`)；环境变量 `OMBRE_API_KEY`/`OMBRE_TRANSPORT`/`OMBRE_BUCKETS_DIR` | commit 284c9c7 | unknown(未在仓库内找到构建产物哈希/lock 文件校验和；`requirements.txt` 未使用 `--require-hashes`) | EV-01-01 (`Dockerfile:1-40`；`server.py:193,13651` port=8000；`docker-compose.yml:9-11`；`render.yaml`全文 @ haven-ombre:284c9c7) | 与独立 ombre-brain 仓库的 Dockerfile 几乎同构（同样"Ombre Brain Docker Build"标题、同样 8000 端口、同样 VOLUME /app/buckets），但 haven-ombre 是单文件巨石架构（server.py 556KB、gateway.py 927KB），独立 ombre-brain 已拆分为 `src/ombrebrain/*` 包结构，二者代码组织方式差异巨大，不是同一代码库的直接同步 |
| 网关(可选二进程模式) | `ombre-gateway`(仅 `compose.hk.yml` 中定义) | unknown | unknown | 容器内 8010；宿主映射 18002:8010(仅 compose.hk.yml) | `/home/user/18358386529/haven-ombre/gateway.py`(927608字节) | `command: ["python", "gateway.py"]`（仅 `compose.hk.yml`，不在默认 `docker-compose.yml` / `render.yaml` 中启用） | `OMBRE_GATEWAY_ADMIN_URL` 环境变量在主服务侧配置，指向 `http://ombre-gateway:8010/api/config` | commit 284c9c7 | unknown | EV-01-02 (`compose.hk.yml`全文；`gateway.py:21387` `port = int(gateway_cfg.get("port", 8010))` @ haven-ombre:284c9c7) | 只有 `compose.hk.yml`（看起来是某次香港 VPS 专项部署配置）声明了双进程(server+gateway)拓扑；标准 `docker-compose.yml`/`render.yaml` 均为单进程(仅 server.py)，未启动 gateway.py |
| 记忆桶存储 | 无独立进程，Markdown 文件系统 | N/A | N/A | N/A（非网络组件） | `bucket_manager.py`(61748字节) 读写 `OMBRE_BUCKETS_DIR` 指向的目录 | 由 server.py 内部调用，非独立启动 | `OMBRE_BUCKETS_DIR` 环境变量，默认容器内 `/app/buckets`，宿主可挂载 Obsidian Vault 目录 | commit 284c9c7 | unknown | EV-01-03 (`Dockerfile:29-31` VOLUME；`docker-compose.yml:16-19` volumes 注释提及 "Obsidian Vault" @ haven-ombre:284c9c7) | 声明的持久化机制是"Markdown 文件即数据"，非数据库 |
| 遗忘/衰减引擎 | 无独立进程 | N/A | N/A | N/A | `decay_engine.py`(13456字节) | 由 server.py 内部调用 | 同上 config.yaml | commit 284c9c7 | unknown | EV-01-04 (`decay_engine.py`文件存在 @ haven-ombre:284c9c7) | 组件划分仅为同进程内的模块划分，不是独立微服务 |
| 反思/梦境引擎 | 无独立进程 | N/A | N/A | N/A | `dream_engine.py`(49576字节)、`reflection_engine.py`(188198字节，仓库内最大单文件之一) | 由 server.py 内部调用 | 同上 | commit 284c9c7 | unknown | EV-01-05 (两文件存在及大小 @ haven-ombre:284c9c7) | 命名与 ombre-brain 的 `src/tools/dream/` 对应，但 haven-ombre 未做包化拆分 |
| 写入闸门 | 无独立进程 | N/A | N/A | N/A | `memory_write_gate.py`(13456字节) | 由 server.py 内部调用 | 同上 | commit 284c9c7 | unknown | EV-01-06 (文件存在 @ haven-ombre:284c9c7) | — |
| CI/CD | GitHub Actions | N/A | N/A | N/A | `.github/workflows/docker-publish.yml`、`.github/workflows/tests.yml` | GitHub Actions 触发，非本地 | — | commit 284c9c7 | unknown | EV-01-07 (`find .github/workflows` 命中两文件 @ haven-ombre:284c9c7) | 存在 CI，但只读克隆无法验证 CI 是否曾在真实 GitHub 环境上跑通（无网络访问） |
| Claude 配置 | N/A | N/A | N/A | N/A | `.claude/`（目录存在，未展开审计其内容，超出本清单范围） | N/A | N/A | commit 284c9c7 | unknown | EV-01-08 (`ls -la` 命中 `.claude` 目录 @ haven-ombre:284c9c7) | 仅记录存在性 |

---

## 二、ombre-brain（P0luz/Ombre-Brain fork，声明定位：参考实现，"不是再部署一个 OB"）

### 部署/组件表

| 组件 | 进程名(代码/镜像名) | PID | 用户 | 端口(声明值) | 路径 | 启动方式(声明) | 配置 | 版本/提交 | 哈希 | 证据ID | 备注 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 主服务(MCP+Web server) | `ombre-brain`(Dockerfile) | unknown(源码审计模式，无本地运行实例，无法实测) | unknown | 容器内固定 `ENV OMBRE_PORT=8000`；裸机默认 18001（Dockerfile 注释："裸机（非容器）不读此 ENV，走 server.py 默认 18001"）；`render.yaml` 用 `startCommand: python src/server.py` | `/home/user/18358386529/ombre-brain/src/server.py`（在已拆分的 `src/ombrebrain/*` 包体系下） | `ENTRYPOINT ["./entrypoint.sh"]`（先做配置文件安全校验，再启动服务；`entrypoint.sh` 本身不直接写死最终 `exec` 命令的可见部分在审计截取的前100行内未出现，需要读全文确认，此处仅确认已读到的准备阶段逻辑） | `config.example.yaml`(13897字节) 复制为 `config.default.yaml`；环境变量矩阵：`OMBRE_TRANSPORT`/`OMBRE_EMBED_BACKEND`/`OMBRE_EMBED_API_KEY`/`OMBRE_COMPRESS_API_KEY`/`OMBRE_DASHBOARD_PASSWORD`/`OMBRE_CONFIG_PATH`/`OMBRE_BUCKETS_DIR` | VERSION 文件 = 3.2.0；commit 6f7335d | unknown（`requirements.lock.txt` 存在但本审计未逐行核对 hash 与 pip 安装是否一致，静态审计仅确认文件存在且 Dockerfile 用 `--require-hashes` 安装） | EV-01-09 (`Dockerfile`全文；`entrypoint.sh:1-100`；`render.yaml`全文；`VERSION`文件内容 @ ombre-brain:6f7335d) | 相比 haven-ombre，ombre-brain 的 Dockerfile 明确声明依赖 `requirements.lock.txt` + `--require-hashes`（供应链完整性更强），并内置 cloudflared 下载逻辑（`deploy/fetch_cloudflared.py`）用于 Dashboard 隧道管理 |
| 部署辅助脚本 | N/A | N/A | N/A | N/A | `deploy/deploy.sh`、`deploy/gen_update_manifest.py`、`deploy/multi_owner.py`、`deploy/fetch_cloudflared.py` | 手动运行，非容器自动执行的一部分（`deploy.sh` 未在 Dockerfile/entrypoint.sh 中被调用） | `deploy/owners.example.yaml`（多所有者配置模板） | commit 6f7335d | unknown | EV-01-10 (`ls deploy/`列出上述文件 @ ombre-brain:6f7335d) | `deploy.sh` 是运维脚本而非容器入口，本次未执行（遵守约束3，禁止执行部署脚本） |
| src/ombrebrain 包体系（"v3 架构"，仓库内部命名，与本审计包无关，见冲突项） | 无独立进程，同一 server.py 内多模块 | N/A | N/A | N/A | `src/ombrebrain/{kernel,microkernel,eventsourcing,distributed,cluster,fabric,architecture,domain,retrieval,storage,security,plugins,adapters,...}` 等29个子包目录 | 由 src/server.py 统一调度 | — | commit 6f7335d | unknown | EV-01-11 (`find src -maxdepth 2 -type d` 列出29个子目录 @ ombre-brain:6f7335d) | 见文末冲突项"v3架构命名歧义"专项说明 |
| 前端/Dashboard | 无独立进程，由主服务提供静态资源 | N/A | N/A | 同主服务端口 | `frontend/`（含 `frontend/onboarding.html`） | 由 server.py 内嵌 Web 路由提供 | `OMBRE_DASHBOARD_PASSWORD` | commit 6f7335d | unknown | EV-01-12 (`ls frontend/` @ ombre-brain:6f7335d) | — |
| Rust 内核脚手架 | 无独立进程（未见编译产物或运行绑定证据） | N/A | N/A | N/A | `kernel/rust/ombre-kernel/{Cargo.toml,src/lib.rs,README.md}` | unknown（本审计未发现任何 Python 侧对该 Rust 包的运行时调用证据，仅确认源文件存在；是否为纯脚手架/预留代码需要专项代码路径追踪，超出本清单范围） | — | commit 6f7335d | unknown | EV-01-13 (`kernel/rust/ombre-kernel/` 目录存在 @ ombre-brain:6f7335d) | 待人工确认：这是否是"vnext_preflight"诊断检查的对象（`tools/vnext_preflight.py` 提及 `rust_kernel_scaffold` 契约检查，见清单1"v3"冲突节） |
| CI/CD | GitHub Actions | N/A | N/A | N/A | `.github/workflows/docker-publish.yml`、`.github/workflows/tests.yml` | GitHub Actions | — | commit 6f7335d | unknown | EV-01-14 (`find .github/workflows` @ ombre-brain:6f7335d) | — |
| Claude 配置 | N/A | N/A | N/A | N/A | `.claude/`（目录存在） | N/A | N/A | commit 6f7335d | unknown | EV-01-15 (`ls -la` @ ombre-brain:6f7335d) | 仅记录存在性 |

---

## 三、xinchao-nian（tianyupaipai-cmd/xinchao-nian，声明定位：心智层，"不是第二个 Memory Core"）

### 部署/组件表

| 组件 | 进程名(代码/镜像名) | PID | 用户 | 端口(声明值) | 路径 | 启动方式(声明) | 配置 | 版本/提交 | 哈希 | 证据ID | 备注 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 联合部署编排 | docker compose 项目 `xinchao-nian` | N/A | N/A | N/A | `/home/user/18358386529/xinchao-nian/compose.yaml` | `docker compose up -d --build`（README 声明的用法，本次未执行） | `.env`(从 `.env.example` 复制) | commit 97f1bdc | unknown | EV-01-16 (`compose.yaml`全文 @ xinchao-nian:97f1bdc) | 一次 `compose up` 会构建**两个**容器：`ombre-brain`（来自内嵌副本 `./ombre-brain`）与 `dynamic-mind`（来自 `./xinchao`），详见冲突项 |
| 心潮动态心智(dynamic-mind) | 镜像名 `xinchao-nian/dynamic-mind:local`，容器名 `ombre-dynamic-mind` | unknown | 容器内以 node 用户运行（`compose.yaml` 注释："心潮以 node 用户跑"） | 容器内/宿主 `127.0.0.1:18110:18110` | `/home/user/18358386529/xinchao-nian/xinchao/src/server.js` 及同目录下 `engine.js`/`thought-pool.js`/`emotion.js`/`awareness.js`/`dimensions.js` 等30+模块 | `package.json` scripts: `"start": "node src/server.js"`；容器内由 `xinchao/Dockerfile` 定义（本次未展开读取该 Dockerfile 全文，仅确认存在） | 环境变量：`OMBRE_MCP_URL`(默认 `http://ombre-brain:8000/mcp`)、`SERVICE_TOKEN`、`.env` 全量注入(`env_file: .env`) | package.json version = 3.3.6；commit 97f1bdc | unknown | EV-01-17 (`xinchao/package.json`全文；`compose.yaml:47-89` @ xinchao-nian:97f1bdc) | 声明"不是第二个 Memory Core"——本次审计确认 dynamic-mind 自身**不含**向量检索/embedding/衰减引擎代码（按目录列表未见对应模块名），是通过 `OMBRE_MCP_URL` 调用 ombre-brain 的 MCP 工具(breath/hold/…)，职责划分与声明一致；但其运行的 ombre-brain 是内嵌副本而非独立仓库版本，见冲突项 |
| 内嵌 ombre-brain 副本 | 镜像名 `xinchao-nian/ombre-brain:local`，容器名 `ombre-brain` | unknown | unknown | 容器内 8000，宿主 `127.0.0.1:${OMBRE_HOST_PORT:-18001}:8000` | `/home/user/18358386529/xinchao-nian/ombre-brain/`（完整源码树，含自己的 `entrypoint.sh`、`src/`、`frontend/`、`deploy/`） | `compose.yaml` 中 `build: {context: ./ombre-brain}` | `OMBRE_CONFIG_PATH=/app/buckets/config.yaml`；命名卷 `ombre-buckets` | 内嵌副本 `VERSION` = 2.6.5（区别于独立仓库的 3.2.0）；commit 97f1bdc(xinchao-nian 自身) | unknown | EV-01-18 (`xinchao-nian/ombre-brain/VERSION`内容 = "2.6.5"；`compose.yaml:8-45` @ xinchao-nian:97f1bdc) | **重点冲突候选，见文末专项分析** |
| bridge（子模块，未初始化） | N/A | N/A | N/A | N/A | `/home/user/18358386529/xinchao-nian/bridge/`（本地目录为空） | unknown | `.gitmodules` 声明 `url = https://github.com/tianyupaipai-cmd/xinchao-runtime-bridge.git` | 子模块锁定 commit：dbc82173fa7c5c6e6210670c654cf26d72655b8c（`git ls-tree` 显示为 `160000 commit`，即 gitlink，非内容） | unknown | EV-01-19 (`.gitmodules`全文；`git ls-tree HEAD -- bridge` 输出 `160000 commit dbc82173...`；`ls -la bridge` 显示目录为空 @ xinchao-nian:97f1bdc) | **无法审计其内容**：这是标准 git submodule（gitlink），克隆时未执行 `git submodule update --init`，本地为空目录。按约束8"禁止联网"，本次无法拉取该子模块内容，其代码对本审计不可见，标记为 `unknown(子模块未初始化，无本地内容，且不可联网拉取)` |
| CI/CD | 未发现 | N/A | N/A | N/A | `find .github/workflows` 无命中 | N/A | N/A | commit 97f1bdc | N/A | EV-01-20 (`find /home/user/18358386529/xinchao-nian -path "*/.github/workflows/*"` 无输出 @ xinchao-nian:97f1bdc) | 与 haven-ombre/ombre-brain/kiwi-mem 均有 CI workflow 不同，xinchao-nian 本身未发现 `.github/workflows`（内嵌 ombre-brain 副本内也未见，因该副本被 diff 排除了 `.github` 相关文件） |
| Claude 配置 | 未发现 | N/A | N/A | N/A | `find -iname .claude` 无命中 | N/A | N/A | commit 97f1bdc | N/A | EV-01-21 (同上工具，无输出 @ xinchao-nian:97f1bdc) | — |

---

## 四、kiwi-mem（LucieEveille/kiwi-mem，声明定位："只移植算法思想，不部署 Kiwi Server"）

### 部署/组件表

| 组件 | 进程名(代码/镜像名) | PID | 用户 | 端口(声明值) | 路径 | 启动方式(声明) | 配置 | 版本/提交 | 哈希 | 证据ID | 备注 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 记忆网关主服务 | `kiwi-mem`(docker-compose service名) | unknown(源码审计模式，无本地运行实例，无法实测) | unknown | 容器内/宿主 `${PORT:-8080}:${PORT:-8080}`；代码内 `mcp_server.py:28` 硬编码默认 `GATEWAY_PORT = int(os.getenv("PORT", "8080"))` | `/home/user/18358386529/kiwi-mem/main.py`(281418字节，单文件巨石) | `CMD ["python", "main.py"]`（Dockerfile） | `.env`(从 `.env.example`)；必填 `API_KEY`；`DATABASE_URL` 在 docker-compose 场景由 compose 自动拼装，手动部署需自填 | commit b01a0c5 | unknown | EV-01-22 (`Dockerfile`全文；`docker-compose.yml`全文；`mcp_server.py:28-29` @ kiwi-mem:b01a0c5) | Dockerfile 本身**不含**数据库服务，数据库依赖完全在 `docker-compose.yml` 中通过独立的 `db` service 声明，见清单6"Postgres依赖"冲突项 |
| PostgreSQL(pgvector) 数据库 | `pgvector/pgvector:pg16`(第三方镜像，非本仓库代码) | N/A | N/A | 容器内 5432（compose 未映射到宿主） | 无本仓库源码，仅 `docker-compose.yml` 中的 service 声明 | `image: pgvector/pgvector:pg16` | `POSTGRES_USER=kiwi`/`POSTGRES_PASSWORD=kiwi_secret_change_me`(默认明文弱密码，仅示例)/`POSTGRES_DB=kiwi_mem` | commit b01a0c5(仅指 compose 文件版本，镜像本身独立发布) | unknown | EV-01-23 (`docker-compose.yml:5-14` @ kiwi-mem:b01a0c5) | `database.py:29,78-80` 用 `asyncpg`，`DATABASE_URL` 未设置时直接 `raise RuntimeError("DATABASE_URL 未设置！")`——**无 SQLite 或内存回退，Postgres 是硬依赖，非可选** |
| MCP 适配层 | 无独立容器，同进程内模块 | N/A | N/A | 通过 `GATEWAY_BASE = f"http://127.0.0.1:{GATEWAY_PORT}"` 反向调用主服务 | `mcp_server.py`(27042字节)、`mcp_client.py`(10963字节)、`mcp_access.py`(5186字节) | 由 main.py 或独立脚本调用，未见独立 Dockerfile/compose service | 环境变量 `MCP_ALLOWED_HOSTS`/`MCP_ALLOWED_ORIGINS` | commit b01a0c5 | unknown | EV-01-24 (三文件存在及职责命名 @ kiwi-mem:b01a0c5) | — |
| Admin Panel(前端) | 无独立进程，静态资源 | N/A | N/A | 同主服务端口 | `admin-panel/`(index.html + css/js，无 package.json，非 npm 构建产物，纯静态文件) | 由 main.py 内嵌路由提供 | — | commit b01a0c5 | unknown | EV-01-25 (`find admin-panel -maxdepth 2`；`admin-panel/package.json` 不存在 @ kiwi-mem:b01a0c5) | — |
| Heat/Decay/Reheat 等算法模块 | 无独立进程 | N/A | N/A | N/A | `database.py`(296045字节，内含记忆检索/排序/衰减逻辑，未逐行定位到独立文件，算法与数据访问耦合在同一巨石文件中)、`memory_extractor.py`(14414字节)、`daily_digest.py`(88600字节) | 由 main.py 调用 | — | commit b01a0c5 | unknown | EV-01-26 (文件存在及大小 @ kiwi-mem:b01a0c5) | 声明"只移植算法思想"意味着这些 `.py` 文件本身不会被复制部署，只读取其中的算法逻辑做参考；但**源码本身与数据库读写强耦合**（`database.py` 是同一个巨石文件），"抽取算法而不碰数据库"在工程上需要额外拆分工作，本审计仅报告耦合事实，不判断是否可行 |
| CI/CD | GitHub Actions | N/A | N/A | N/A | `.github/workflows/ci.yml` | GitHub Actions | — | commit b01a0c5 | unknown | EV-01-27 (`find .github/workflows` 命中1个文件 @ kiwi-mem:b01a0c5) | — |
| Claude 配置 | 未发现 `.claude/` | N/A | N/A | N/A | `find -iname .claude` 无命中；但根目录有 `CLAUDE.md`(8141字节)和 `AGENTS.md`(5638字节) | N/A | N/A | commit b01a0c5 | N/A | EV-01-28 (`ls`列出根目录文件；`find -iname .claude` 无命中 @ kiwi-mem:b01a0c5) | `.claude/`目录本身不存在，但存在同类用途的 `CLAUDE.md` 说明文件 |

---

## 发现的冲突与待人工裁决项

### 冲突1：xinchao-nian 内嵌 ombre-brain 副本 —— 确认为"源码层面共存的第二份 Ombre-Brain"

**核查方法**：对比 `/home/user/18358386529/ombre-brain`（独立仓库，声明来自 P0luz/Ombre-Brain）与 `/home/user/18358386529/xinchao-nian/ombre-brain/`（内嵌副本）的 VERSION 文件、目录结构、许可证文件、MODIFICATIONS.md。

**核查结果（均有证据支持，非采信文档）**：

1. **版本不同**：独立仓库 `VERSION` = `3.2.0`（EV: `/home/user/18358386529/ombre-brain/VERSION` @ ombre-brain:6f7335d）；内嵌副本 `VERSION` = `2.6.5`（EV: `/home/user/18358386529/xinchao-nian/ombre-brain/VERSION` @ xinchao-nian:97f1bdc）。二者相差一个大版本以上的迭代距离，绝非同一构建产物。

2. **血统不同，非同一 fork 链路**：内嵌副本自带 `MODIFICATIONS.md` 明确记载其血统为 `P0luz/Ombre-Brain（原项目, MIT）→ CyberSealNull 二改（追加"新增内容非商业"约束）→ 心潮念（本仓库）在其基础上做修改`（EV: `xinchao-nian/ombre-brain/MODIFICATIONS.md` 全文 @ xinchao-nian:97f1bdc）。独立仓库 `/home/user/18358386529/ombre-brain` 的 `LICENSE`/`NOTICE.md` 未提及 CyberSealNull 这一中间 fork，是直接从 P0luz 原项目 fork 而来（用户任务表格中也标注参考仓库为 `P0luz/Ombre-Brain`，无 CyberSealNull 中转）。也就是说，内嵌副本经过了一次独立仓库版本没有经过的"二改+许可证叠加"环节，二者是**两条不同的代码演化分支**，不是同一份代码的两个副本。

3. **目录结构存在实质差异（非简单文件拷贝）**：`diff` 全量文件列表显示：
   - 内嵌副本比独立仓库**少**：`.github/workflows/`（CI）、`.dockerignore`、`AUTHORS.md`、`CONTRIBUTING.md`、`DCO`、`deploy/deploy.sh`/`multi_owner.py`/`owners.example.yaml`、`docs/ENVIRONMENT_VARIABLES.md`、`docs/SRC_PACKAGE_MIGRATION_PLAN.md`、`docs/adr/*`、`kernel/rust/*`（Rust 内核脚手架整体缺失）、`render.yaml`、`requirements-dev.*`、`ruff.toml`、`rule.md`、`src/tools/forget/`(独立仓库有 `forget` 工具，内嵌副本缺失该工具目录，见下)。
   - 内嵌副本比独立仓库**多**：`LICENSE.CyberSealNull`/`LICENSE.P0luz-MIT`（拆分许可证文件）、`MODIFICATIONS.md`、`NOTICE.CyberSealNull.md`、`README.upstream.md`、`config.default.yaml`（独立仓库对应文件叫 `config.example.yaml`）、`src/auth_policy.py`/`src/backup_archive.py`（独立仓库对应实现已迁移到 `src/ombrebrain/storage/backup_archive.py` 包路径下，内嵌副本仍在旧的扁平路径，印证内嵌副本停留在独立仓库"v3 包迁移"之前的旧版本结构）、`src/bucket_scoring.py`/`src/embedding_outbox.py`/`src/ledger_mirror.py`/`src/ledger_property.py`/`src/ledger_replay.py`/`src/memory_messages.py`（均为旧扁平路径，尚未迁移进 `ombrebrain/` 包）。
   （EV: `diff <(find .../ombre-brain -type f) <(find .../xinchao-nian/ombre-brain -type f)` 输出全文，已在审计过程中执行并记录 @ 两仓库当前 HEAD）
   - 独立仓库 `src/ombrebrain/` 子包比内嵌副本**多两个**目录：`security`、`plugins`（内嵌副本没有独立的 `security/`、`plugins/` 子包，但仍多出 `src/tools/__pycache__`，说明内嵌副本目录里混入了运行/编译残留物，这本身也说明这份内嵌副本不是"纯净的参考代码快照"，而可能被实际跑过、生成过 `.pyc` 缓存后又被提交进版本库）。

4. **git 存储形式**：`git ls-tree HEAD -- ombre-brain` 显示模式为 `040000 tree`（普通目录树），**不是** `160000 commit`（gitlink/子模块）。也就是说 xinchao-nian 把这份 ombre-brain 代码当作**普通文件直接提交进自己的 git 历史**，而不是像 `bridge/` 那样以 git submodule 引用外部仓库。这意味着这份代码是被"人工/工具复制粘贴+二次修改"后固化进 xinchao-nian 仓库的，其后续更新不会随独立 ombre-brain 仓库的新提交自动同步，天然会越漂越远。

5. **实际会被构建为独立容器**：`xinchao-nian/compose.yaml` 中 `ombre-brain` service 的 `build.context` 明确指向 `./ombre-brain`（即内嵌副本），产出镜像名为 `xinchao-nian/ombre-brain:local`，与独立仓库编译出的镜像是完全不同的两份镜像（不同 VERSION、不同代码）。

**结论（陈述事实，不代人裁决）**：xinchao-nian 文档声称"不是第二个 Memory Core"，但从源码角度看：
- xinchao-nian 项目在其自身仓库内固化了一份**独立于**"独立 ombre-brain 参考仓库"的 Ombre-Brain 代码分支（VERSION 2.6.5，经 CyberSealNull 二改），且这份代码**会被实际编译运行**（compose.yaml 中有专属 build+容器定义）。
- 这与用户表格中"ombre-brain 独立仓库=参考实现，不是再部署一个 OB"的表述，以及"xinchao-nian=心智层，不是第二个 Memory Core"的表述，在**源码事实层面**存在张力：即便"心智层本身"（`xinchao/` 目录下的 dynamic-mind 代码）确实不含记忆检索/衰减逻辑（与声明一致），但 xinchao-nian **项目整体**（其 `compose.yaml` 定义的部署拓扑）在源码层面确实内置并会启动一份独立血统、独立版本的 Ombre-Brain 变体。这与"审计包中的独立 ombre-brain 仓库"是两份不同的代码，都具备被构建为运行容器的能力。
- 是否将其视为"违反了不重复部署 OB 的约束"，还是视为"心智层项目自带的一个必要依赖，与审计范围内的 Memory Core 本体（haven-ombre）互不冲突"，属于**架构决策/许可证合规问题**，需要人工裁决，本审计不代为下结论。
- **附带许可证风险**：内嵌副本明确声明"心潮念整体非纯 MIT"，`ombre-brain/` 部分受 CyberSealNull 二改的"新增内容非商业"约束叠加 P0luz 原 MIT 的署名保留要求，商用需取得双重授权（EV: `MODIFICATIONS.md` 末段 @ xinchao-nian:97f1bdc）。若"记忆系统2.0"项目存在任何商业化路径，这是一个需要法务/人工评估的许可证合规问题。

### 冲突2：haven-ombre 的 Supabase/Postgres 依赖

详见 `06_resource_backup.md` 文末"发现的冲突与待人工裁决项"第2项，此处不重复展开，仅在本文档索引：证据主档位于 `haven-ombre/scripts/sync_to_supabase.py`、`haven-ombre/scripts/supabase_memory_rpc.sql`。

### 冲突3："v3 架构"命名歧义 —— 确认为纯文字巧合，不构成实质冲突，但需明确记录避免后续误读

**核查结果**：
- 独立 `ombre-brain` 仓库内确实存在大规模、系统性的"v3"相关工程活动：`tests/` 目录下有 **46 个** `test_v3_*.py` 文件（EV: `ls tests/ | grep -c "^test_v3_"` = 46 @ ombre-brain:6f7335d），命名涵盖 `test_v3_architecture_audit.py`、`test_v3_kernel_registry.py`、`test_v3_event_sourced_kernel.py`、`test_v3_distributed_fabric.py`、`test_v3_consensus.py`、`test_v3_legacy_bucket_integration.py` 等。
- 同时存在 `docs/SRC_PACKAGE_MIGRATION_PLAN.md`，其内容是"把 `src/` 根目录的业务实现逐文件迁入 `src/ombrebrain/` 明确领域包"的**包结构迁移计划**（EV: 文件全文头部60行 @ ombre-brain:6f7335d），文中明确声明"这是一项包结构迁移，不借机重写业务逻辑"，并记录了阶段化的迁移清单（`memory_messages.py`→`ombrebrain/domain/memory_messages.py` 等）。
- `src/ombrebrain/` 下确实存在名为 `kernel`、`microkernel`、`eventsourcing`、`distributed`、`cluster`、`fabric`、`architecture`、`decision`、`consensus`(测试提及)、`resilience` 等具有"分布式系统/微内核架构"色彩的子包目录（EV: `find src -maxdepth 2 -type d` 输出 @ ombre-brain:6f7335d）。

**判断**：这里的"v3"是 **ombre-brain 仓库自身内部**对其"`src/` 扁平模块 → `src/ombrebrain/` 领域包"这一代码重构/包迁移工作及相关能力分层（kernel/微内核/事件溯源/分布式容错等）的**内部代号**，与用户所述"本审计包不是架构3.0"里的"架构3.0"（应指"记忆系统2.0"项目自身的版本代号体系）是**两个完全不同语境下的独立编号**，没有证据表明二者存在实质关联、依赖或被本审计对象直接采用的因果关系。本审计**未发现**任何证据显示 haven-ombre/xinchao-nian/kiwi-mem 三个仓库引用、依赖或以任何方式集成了 ombre-brain 仓库这套"v3 包体系"的具体代码（三者均未 import `ombrebrain.*` 包路径，haven-ombre 是单文件巨石架构、kiwi-mem 是完全独立的 Node.js 无关代码库、xinchao-nian 的内嵌 ombre-brain 副本版本更旧（2.6.5）且明确保留旧的扁平 `src/*.py` 路径而非新包路径，说明内嵌副本固化时间点早于这套"v3"包迁移完成之前）。

**结论**：这是**纯粹的文字/编号巧合**，建议后续文档、评审、issue 讨论中，凡提及"v3"必须显式加注"ombre-brain 仓库内部包迁移代号"或"记忆系统2.0项目自身版本"以区分语境，避免团队内部沟通产生误解。本审计不对"记忆系统2.0"项目自己的"架构3.0"含义做任何推断或评价，因为那超出四个只读克隆仓库的审计范围。

### 冲突4（新发现，未在任务清单中预先提示）：ombre-brain 的 `kernel/rust/` Rust 脚手架代码存在性

独立仓库 `ombre-brain` 含有 `kernel/rust/ombre-kernel/{Cargo.toml, src/lib.rs, README.md}`，这是一个 Rust crate 脚手架。本审计仅确认其**源文件存在**，未发现任何 Python 侧运行时对它的直接调用证据（未做全文调用链追踪，超出本清单深度）。`tools/vnext_preflight.py`（Dockerfile 中被 COPY 进镜像，用于"rust_kernel_scaffold 契约检查"诊断，据 Dockerfile 注释推断，未展开读取该脚本全文核实具体检查逻辑）提示该 Rust 代码与某个"预检"诊断相关。**建议人工后续专项确认**：这段 Rust 代码是否被编译/链接进最终运行时（本次审计未发现 Dockerfile 中有任何 `cargo build`/`rustc` 调用步骤，初步判断为**未被镜像构建流程编译**，仅作为源码存在，但这不是穷尽性结论，需要人工用 `grep -rn "ombre.kernel\|ombre_kernel" src/` 等方式做进一步交叉验证）。
