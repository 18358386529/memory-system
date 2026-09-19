# Phase 0 基线锁定(Baseline Lock)

版本:v1
性质:源码审计模式(无本地运行实例;代码托管与运行均依赖 GitHub 环境)
锁定时间:见下方各仓库记录时间

## 审计对象与访问方式

四个仓库均为账号 `18358386529` 下的公开 fork,通过本会话 git 代理**匿名只读克隆**(`GIT_LFS_SKIP_SMUDGE=1 git clone --depth 1`),**未使用 push/attach 凭证**,结构上不具备写回源仓库的能力。

| 参考清单原始仓库 | 审计对象(fork) | 本地只读克隆路径 | 锁定 Commit SHA | Clone URL | 本地大小(不含.git) |
|---|---|---|---|---|---|
| Yinglianchun/Haven-Ombre | 18358386529/haven-ombre | /home/user/18358386529/haven-ombre | `284c9c7b0e51a0ba0032c7028f705d72458cb304` | https://github.com/18358386529/haven-ombre | 3.8M |
| P0luz/Ombre-Brain | 18358386529/ombre-brain | /home/user/18358386529/ombre-brain | `6f7335d01c43f79a82d5ec999ec3c517f6b9c8a5` | https://github.com/18358386529/ombre-brain | 9.9M |
| tianyupaipai-cmd/xinchao-nian | 18358386529/xinchao-nian | /home/user/18358386529/xinchao-nian | `97f1bdcc76b748fa516b7c79a0aa10234143795f` | https://github.com/18358386529/xinchao-nian | 6.9M |
| LucieEveille/kiwi-mem | 18358386529/kiwi-mem | /home/user/18358386529/kiwi-mem | `b01a0c506f4f10f90f30d408c0291f16720f0718` | https://github.com/18358386529/kiwi-mem | 2.7M |

证据ID:BASELINE-001
命令:`git clone --depth 1 <url> <path> && git -C <path> rev-parse HEAD && git -C <path> remote get-url origin`
主机:云端只读审计沙箱(非生产运行环境)

## 重要限定(适用于全部后续报告)

1. **本次审计为源码审计,不是运行时审计。** 所有结论必须标注"证据来源:源码,commit 见上表",禁止写成"已实现/正在运行/已验证"等运行时结论。
2. 这些是**用户账号下的 fork**,不是原作者仓库,也未验证是否等于任何实际部署环境使用的确切版本。fork 落后于/领先于上游的部分,只在被明确要求时才对比。
3. 清单1(Runtime)只能做"代码结构与部署推断";清单6(Resource/Backup)只能做"仓库内配置文件与备份脚本审计";两者都必须标注"⚠ 无法实测"。
4. 若发现某仓库依赖 fork 之外的另一个仓库(例如 vendored 副本),按事实记录路径与版本文件,不擅自判断是否为"违规的第二实例"，只标记为待复核冲突项。

## Phase 0 冻结清点(初步 grep,供各子代理参考,非最终结论)

以下线索来自本次克隆后的顶层目录扫描(`find <repo> -maxdepth 2`),仅作为子代理起点,**不构成结论**,子代理必须自行验证并附证据ID:

- `haven-ombre/scripts/sync_to_supabase.py`、`haven-ombre/scripts/supabase_memory_rpc.sql`:仓库内存在 Supabase(托管 Postgres)同步脚本 —— 需核实是否与约束4"禁止引入 PostgreSQL/pgvector"冲突,或属于遗留/可选路径。
- `xinchao-nian/ombre-brain/`:xinchao-nian 仓库内部**内嵌了一份 ombre-brain 副本**(含 `LICENSE.P0luz-MIT`、`MODIFICATIONS.md`、`README.upstream.md`),与独立仓库 `18358386529/ombre-brain` 是否为同一版本、是否构成"第二个 Ombre-Brain"需要专项核查。
- `ombre-brain/tests/test_v3_*.py`(约30+个文件)、`ombre-brain/docs/SRC_PACKAGE_MIGRATION_PLAN.md`:代码内部存在大量"v3 架构"相关测试和迁移计划文档,与用户方"记忆系统2.0/不是架构3.0"的说法在命名上可能冲突,需要专项核查以避免误判。
- `kiwi-mem/`:仓库本身是完整可部署服务(`Dockerfile`、`docker-compose.yml`、`mcp_server.py`、`admin-panel/`),而不仅仅是"算法参考"。需要在报告中明确:我们只读取其算法相关代码(如有 Heat/Decay/Reheat/Ranking),不建议、也不会尝试运行其 Dockerfile/compose。
