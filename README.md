# 省级医保费用查询改写验证服务

面向稽核、病种付费（DRG）和预算团队的费用明细查询改写验证平台。技术栈：**Java 25 + Spring Boot 3.5 + PostgreSQL 18**（JDBC/JdbcTemplate + Flyway + springdoc-openapi）。

月结、政策切换和审计期，多人同时按机构、诊断、药品与结算状态筛选数亿行数据。本服务对慢查询做**改写前验证**：在冻结快照的只读副本上以受限资源影子执行原语句与候选语句，由裁判器逐位比较明细主键集合、金额与机构级分组，通过后才允许经发布门禁绑定到参数范围与权限策略上线；权限收窄、数据刷新、回退均有明确处置与账本。

## 模块与实现对照

| 需求模块 | 实现 | 说明 |
| --- | --- | --- |
| 查询登记 | `POST /api/registrations` | 保存语句摘要(SHA-256)、用途、授权范围、参数形态、关联快照 |
| 计划采集器 | `POST /api/registrations/{id}/plans` | EXPLAIN(FORMAT JSON) 采集执行计划、估计扫描量、锁等待(pg_stat_activity 采样)、代价；按**参数签名**分别采集 |
| 规则诊断器 | `POST /api/registrations/{id}/diagnose` | 识别可索引化条件(to_char 包装)、隐式类型转换、冗余关联、跨期过滤、全表扫描；按 (登记, 参数签名, 计划哈希) **分别建档** |
| 候选生成器 | `POST /api/registrations/{id}/candidates` | 输出等价的条件下推、分页、预聚合、连接重排（含冗余连接消除） |
| 影子执行器 | `POST /api/candidates/{id}/shadow` | 冻结快照 + 只读事务 + 受限资源(statement_timeout/lock_timeout/work_mem/并行度)运行原语句与候选语句，保存主键集合、总金额、机构分组、实际计划 |
| 结果裁判器 | `POST /api/candidates/{id}/judge` | 比较明细主键集合、金额、机构级分组，定位缺失原因；**总金额相同而机构分组不同 → 阻断并列出差异组** |
| 发布门禁 | `POST /api/candidates/{id}/gate` | 仅 PASS 且 CURRENT 的裁判可进门禁；参数范围**不得超出已验证范围**（防止小范围验证推广到全省） |
| 路由器 | `POST /api/router/query` | 按门禁状态/裁判时效/策略版本/参数范围路由到候选或原语句 |
| 回退账本 | `GET /api/ledger` | 记录 ROLLBACK / POLICY_NARROW / DATA_REFRESH / POLICY_REBIND 的路由切换、受影响请求与恢复依据 |

## 验证方式的关键决策

- **选择冻结快照的只读副本**（而非隔离可回放分析副本）：`POST /api/snapshots/freeze` 把业务基表复制为 `snap_*` 物理表（含索引与 ANALYZE 统计），结构、索引、统计信息都贴近正式统计库；影子执行只打快照，不打生产基表。
- **代价**：验证结论只代表该快照。数据更新后再次 freeze 时，旧裁判自动标记 `STALE`、依赖旧结论的灰度/已发布门禁自动 `BLOCKED` 并记 `DATA_REFRESH` 账本，路由回退原语句，需按风险重验。
- **参数倾斜**：相同 SQL 在不同参数范围下产生不同计划哈希时，诊断器分别建档（演示场景 1 可复现）。
- **权限收窄**：灰度期间新策略生效，旧策略门禁的在途请求标记 `PENDING_REVIEW`（复核前不再返回），新请求以 `POLICY_RETIRED` 拒绝，处置写入账本。

## 快速开始

### 方式一：Docker Compose（一键）

```bash
docker compose up --build
# 应用: http://localhost:8080   Swagger UI: http://localhost:8080/swagger-ui.html
```

compose 含 PostgreSQL 18（ccdev/ccdev）与应用两个服务，应用等数据库健康检查后启动，Flyway 自动建表与种子数据。

### 方式二：本地运行

前置：PostgreSQL 可连接（默认 `127.0.0.1:5432`，库/用户/密码均为 `ccdev`，可用环境变量 `DB_HOST/DB_PORT/DB_NAME/DB_USER/DB_PASS` 覆盖）。

```bash
# 开发模式
mvn spring-boot:run

# 或打包后运行
mvn package -DskipTests
java -jar target/fee-rewrite-verify-1.0.0.jar
```

首次启动由 Flyway 执行 `V1__schema.sql`（元数据模型）与 `V2__seed.sql`（确定性费用样例：24000 行明细、12 家机构、30 病种、50 药品、正确/缺陷两版预聚合表、测试账号、权限策略 v1）。

### 测试

```bash
mvn test
```

含改写引擎与参数分级单元测试，以及在独立 schema（`verify_it`）中真实跑通全部场景的集成测试（需本地 PostgreSQL）。

## 测试账号

路由器通过 `token` 识别用户（演示用静态令牌，存于 `app_user` 表）：

| 账号 | token | 角色 | 机构 | 说明 |
| --- | --- | --- | --- | --- |
| audit01 | `tok-audit01` | AUDIT | ORG001 | 稽核员（机构级） |
| audit02 | `tok-audit02` | AUDIT | ORG002 | 稽核员（机构级） |
| drg01 | `tok-drg01` | DRG | 省级 | 病种付费专员 |
| budget01 | `tok-budget01` | BUDGET | 省级 | 预算分析员 |

权限策略按版本管理（`fee-query` 族）：v1 允许 AUDIT 查 ORG001~003、DRG/BUDGET 全省；演示中创建的 v2 收窄为 AUDIT 仅 ORG001、DRG 仅 ORG001/002。

## 验证方式（本题）

### 1. 一键演示（推荐）

```bash
curl -X POST http://localhost:8080/api/demo/run | jq
```

一次调用真实落库跑通六个场景并返回证据：

| 场景 | 关键证据字段 | 预期 |
| --- | --- | --- |
| ① 参数倾斜 | `scenario1_paramSkew` | 窄参数(单月+小机构)→Bitmap Index Scan，宽参数(4个月+全机构)→Seq Scan；`plansDiffer=true`，`diagnosisFiledSeparately=true`，分别建档 2 份 |
| ② 分组不一致 | `scenario2_groupMismatchBlocked` | stale 预聚合候选 `verdict=BLOCK`、`totalAmountEqual=true`、`orgGroupEqual=false`，`orgGroupDiff` 定位 ORG001(+82137.78)/ORG002(-82137.78) |
| ③ 发布门禁 | `scenario3_releaseGate` | BLOCK 候选进门禁被拒、超验证范围绑定被拒；范围内路由 CANDIDATE、范围外回退 ORIGINAL |
| ④ 权限收窄 | `scenario4_policyNarrow` | 在途请求 → `PENDING_REVIEW`；旧策略新请求 → `REJECTED(POLICY_RETIRED)`；账本记 `POLICY_NARROW` |
| ⑤ 回退演练 | `scenario5_rollback` | 门禁 `ROLLED_BACK`、`affectedRequests≥2`、后续路由回退 ORIGINAL、账本记 `ROLLBACK` |
| ⑥ 数据刷新重验 | `scenario6_dataRefresh` | 旧裁判 `STALE`、门禁 `BLOCKED`、路由回退 ORIGINAL、账本记 `DATA_REFRESH` |

### 2. 手工分步验证（curl 示例）

```bash
# 冻结快照 → 登记 → 采计划（窄/宽两组参数）→ 诊断 → 生成候选 → 影子执行 → 裁判
curl -X POST localhost:8080/api/snapshots/freeze -H 'Content-Type: application/json' -d '{"name":"manual"}'
curl -X POST localhost:8080/api/registrations -H 'Content-Type: application/json' -d @doc/registration.json
curl -X POST localhost:8080/api/registrations/1/plans -H 'Content-Type: application/json' \
     -d '{"params":{"monthFrom":"2025-08","monthTo":"2025-08","orgCode":"ORG003"}}'
curl -X POST localhost:8080/api/registrations/1/plans -H 'Content-Type: application/json' \
     -d '{"params":{"monthFrom":"2025-11","monthTo":"2026-02"}}'
curl -X POST localhost:8080/api/registrations/1/diagnose
curl -X POST localhost:8080/api/registrations/1/candidates -H 'Content-Type: application/json' \
     -d '{"types":["PUSHDOWN","PAGINATION","JOIN_REORDER","PRE_AGGREGATE"],"preAggregateVariants":["current","stale"]}'
curl -X POST localhost:8080/api/candidates/1/shadow -H 'Content-Type: application/json' \
     -d '{"params":{"monthFrom":"2025-11","monthTo":"2026-02"}}'
curl -X POST localhost:8080/api/candidates/1/judge
# 门禁 → 路由 → 回退
curl -X POST localhost:8080/api/candidates/1/gate -H 'Content-Type: application/json' \
     -d '{"paramRange":{"monthFrom":"2025-11","monthTo":"2026-02","orgCode":"ANY","settleStatus":"ANY","diagPrefix":"ANY"},"policyId":1}'
curl -X POST localhost:8080/api/router/query -H 'Content-Type: application/json' \
     -d '{"token":"tok-audit01","gateId":1,"params":{"monthFrom":"2025-11","monthTo":"2026-02"}}'
curl -X POST localhost:8080/api/gates/1/rollback -H 'Content-Type: application/json' -d '{"reason":"演练"}'
curl localhost:8080/api/ledger?gateId=1
```

演示 SQL 模板与两组参数可通过 `GET /api/demo/sql` 查看；`doc/registration.json` 为登记报文示例。

### 3. 可计算的费用快照样例核对

`GET /api/snapshots/{id}/sample` 返回机构级汇总、总额、校验和，并附可手工重算的 SQL。例如用 psql 独立重算（应与接口一致）：

```sql
SELECT COUNT(*), SUM(amount) FROM snap_fee_detail;                 -- 24000 / 6023280.00
SELECT org_code, SUM(amount) FROM snap_fee_detail
 WHERE to_char(settle_date,'YYYY-MM') BETWEEN '2025-11' AND '2026-02'
   AND org_code IN ('ORG001','ORG002') GROUP BY 1;                  -- 80634.20 / 82137.78
SELECT org_code, SUM(amount) FROM snap_fee_daily_org_stale
 WHERE month BETWEEN '2025-11' AND '2026-02' AND org_code='ORG001' GROUP BY 1;  -- 162771.98（缺陷表把 ORG002 并入 ORG001）
```

### 4. 自动化测试

```bash
mvn test   # 13 个测试：改写引擎/参数分级单元测试 + 全场景集成测试（独立 schema verify_it）
```

## 实际验证结论（本容器内真实执行）

- `mvn test`：13 个测试全部通过（含 DemoFlowTest 在独立 schema 中真实跑通六场景）。
- `POST /api/demo/run`：HTTP 200，六场景证据全部符合上表预期（窄/宽计划哈希不同、stale 候选 BLOCK 且差异组为 ORG001/ORG002、在途请求 PENDING_REVIEW、回退后路由 ORIGINAL、刷新后门禁 BLOCKED）。
- 快照样例经 psql 独立重算核对一致：24000 行、总额 6023280.00、校验和 `77df7aacfc8eeb2de9a461732d0cb4e6`。
- `java -jar target/fee-rewrite-verify-1.0.0.jar` 独立启动正常，Swagger UI 与 OpenAPI 文档（33 个路径）可访问。
- Dockerfile 与 docker-compose.yml 经静态核验（本容器无 docker 守护进程，未实际构建镜像）；compose 配置含健康检查与依赖编排，镜像标签使用 `maven:3.9-eclipse-temurin-25` / `eclipse-temurin:25-jre` / `postgres:18-alpine`。

## 设计说明与边界

- **登记模板族**：改写等价性靠机械变换保证，因此登记仅接受平台模板族（`FROM snap_fee_detail d` + 行式 AND 条件 + ORDER BY），见 `SqlRewrite`；非模板族 SQL 返回 422。
- **参数签名** = 参数分级(NARROW/MEDIUM/WIDE) + 参数值哈希；计划哈希 = 计划节点类型+关系名的 SHA-256。诊断按 (登记, 参数签名, 计划哈希) 唯一建档。
- **预聚合候选**仅在 `settleStatus/diagPrefix` 为空时保证等价，门禁强制绑定 `NULL_ONLY`。
- **锁等待**：影子执行期间以 25ms 间隔采样 `pg_stat_activity` 的 Lock 等待事件累计近似值。
- **演示规模**：明细 24000 行（ORG000 占 55% 以制造计划倾斜）；主键集合全量落库仅为演示便利，生产应改为哈希+抽样。
- **冻结副本形态**：演示中 snap_* 表与基表同实例（生产可替换为只读备库/快照schema）；freeze 会重建 snap_* 表并使旧结论失效。
- 本服务不修改 task-base 基线数据，所有验证对象均为运行时新建的 snap_* 表与元数据。
