# 企业级数据平台微服务架构设计（DDD）

## 0. 设计原则（DDD）

- **按限界上下文（Bounded Context）拆分**：每个微服务只承载单一领域能力。
- **统一语言（Ubiquitous Language）**：案件、模板、任务、实体、关系、战法、告警等术语在上下文内语义唯一。
- **聚合根保护一致性**：跨聚合用事件最终一致，避免分布式事务。
- **读写分离与 CQRS**：高查询负载场景采用专用读模型（搜索、图、OLAP）。
- **事件驱动优先**：服务间通过领域事件解耦，关键命令链路使用同步 API。

---

## 1. 完整微服务列表（按领域拆分）

## 1.1 身份与租户域（Identity & Tenant Context）

### 1) IAM Service（身份与权限服务）
**服务职责**
- 用户、角色、策略（RBAC + ABAC）管理。
- 统一认证（OIDC/SAML/LDAP）与 Token 签发。
- API 鉴权与操作审计。

**核心 API 接口**
- `POST /api/v1/auth/login`
- `POST /api/v1/auth/refresh`
- `GET /api/v1/users/{id}`
- `POST /api/v1/roles`
- `POST /api/v1/policies/evaluate`

**事件通信**
- 发布：`iam.user.created`、`iam.role.changed`、`iam.policy.changed`
- 订阅：`tenant.created`（初始化默认角色）

**数据库**
- PostgreSQL（用户/角色/策略）
- Redis（会话与令牌黑名单）

---

### 2) Tenant Service（租户服务）
**服务职责**
- 租户生命周期管理（创建、冻结、配额）。
- 租户资源编排（命名空间、Topic、索引前缀、图空间）。

**核心 API 接口**
- `POST /api/v1/tenants`
- `GET /api/v1/tenants/{tenantId}`
- `PATCH /api/v1/tenants/{tenantId}/quota`
- `POST /api/v1/tenants/{tenantId}:suspend`

**事件通信**
- 发布：`tenant.created`、`tenant.quota.updated`、`tenant.suspended`
- 订阅：`billing.arrears.detected`（欠费冻结）

**数据库**
- PostgreSQL（租户主数据）

---

## 1.2 案件与流程域（Case & Workflow Context）

### 3) Case Service（案件服务）
**服务职责**
- 案件聚合根管理：创建、状态流转、归档。
- 案件与数据集、任务、战法结果的关联。

**核心 API 接口**
- `POST /api/v1/cases`
- `GET /api/v1/cases/{caseId}`
- `PATCH /api/v1/cases/{caseId}/status`
- `POST /api/v1/cases/{caseId}/datasets:bind`

**事件通信**
- 发布：`case.created`、`case.status.changed`、`case.archived`
- 订阅：`ingestion.batch.completed`、`tactic.alert.generated`

**数据库**
- PostgreSQL（案件主数据）

---

### 4) Workflow Service（流程编排服务）
**服务职责**
- 编排“导入→转换→图构建→索引构建→分析”流程。
- Saga 状态机与补偿处理。

**核心 API 接口**
- `POST /api/v1/workflows`
- `GET /api/v1/workflows/{workflowId}`
- `POST /api/v1/workflows/{workflowId}:retry`

**事件通信**
- 发布：`workflow.started`、`workflow.step.failed`、`workflow.completed`
- 订阅：`case.created`、`template.published`、`ingestion.batch.completed`

**数据库**
- PostgreSQL（流程实例）
- Redis（短状态缓存）

---

## 1.3 模板与规则域（Template & Rule Context）

### 5) Template Service（模板服务）
**服务职责**
- 模板定义（字段映射、校验、转换 DSL、图映射、索引映射）。
- 模板版本、灰度发布、回滚。

**核心 API 接口**
- `POST /api/v1/templates`
- `POST /api/v1/templates/{templateId}/versions`
- `POST /api/v1/templates/{templateId}/publish`
- `POST /api/v1/templates/{templateId}/validate`

**事件通信**
- 发布：`template.created`、`template.published`、`template.rolledback`
- 订阅：`case.created`（创建推荐模板）

**数据库**
- PostgreSQL（模板定义与版本）
- MinIO/S3（模板附件）

---

### 6) Tactic Service（战法服务）
**服务职责**
- 战法规则建模、版本管理、命中判定。
- 风险评分与预警生成。

**核心 API 接口**
- `POST /api/v1/tactics`
- `POST /api/v1/tactics/{id}/publish`
- `POST /api/v1/tactics/evaluate`
- `GET /api/v1/alerts?caseId=...`

**事件通信**
- 发布：`tactic.published`、`tactic.alert.generated`
- 订阅：`feature.vector.updated`、`graph.entity.linked`

**数据库**
- PostgreSQL（规则元数据）
- ClickHouse（命中明细与时序分析）

---

## 1.4 数据接入与处理域（Ingestion & Processing Context）

### 7) Ingestion Service（数据导入服务）
**服务职责**
- 多源接入：文件、API、DB、消息队列。
- 导入任务管理、断点续传、批次控制。

**核心 API 接口**
- `POST /api/v1/ingestions`
- `POST /api/v1/ingestions/{id}/upload-url`
- `GET /api/v1/ingestions/{id}`
- `POST /api/v1/ingestions/{id}:cancel`

**事件通信**
- 发布：`ingestion.batch.started`、`ingestion.batch.completed`、`ingestion.batch.failed`
- 订阅：`case.created`、`template.published`

**数据库**
- PostgreSQL（任务元数据）
- Object Storage（原始文件）

---

### 8) CDC Connector Service（CDC 同步服务）
**服务职责**
- 数据库 CDC 采集（MySQL/PG/Oracle）。
- Schema 演进检测与 topic 路由。

**核心 API 接口**
- `POST /api/v1/cdc/connectors`
- `PATCH /api/v1/cdc/connectors/{id}`
- `GET /api/v1/cdc/connectors/{id}/lag`

**事件通信**
- 发布：`cdc.stream.started`、`cdc.schema.changed`、`cdc.lag.alerted`
- 订阅：`tenant.created`（初始化 connector 资源）

**数据库**
- PostgreSQL（Connector 配置）
- Kafka（CDC 事件日志）

---

### 9) Transform Service（数据转换服务）
**服务职责**
- 清洗、标准化、主键归一、实体抽取。
- 执行模板 DSL，输出 DWD 标准数据。

**核心 API 接口**
- `POST /api/v1/transforms/jobs`
- `GET /api/v1/transforms/jobs/{jobId}`
- `POST /api/v1/transforms/jobs/{jobId}:replay`

**事件通信**
- 发布：`transform.job.completed`、`transform.quality.failed`
- 订阅：`ingestion.batch.completed`、`template.published`

**数据库**
- Iceberg（DWD 输出）
- PostgreSQL（任务控制面）

---

### 10) Stream Processing Service（实时计算服务）
**服务职责**
- Flink 实时任务管理：窗口聚合、实时规则、维表 Join。
- 实时指标写入 OLAP，实时索引写入搜索。

**核心 API 接口**
- `POST /api/v1/stream-jobs`
- `GET /api/v1/stream-jobs/{jobId}`
- `POST /api/v1/stream-jobs/{jobId}:savepoint`

**事件通信**
- 发布：`stream.metric.updated`、`stream.job.failed`
- 订阅：`cdc.stream.started`、`transform.job.completed`

**数据库**
- Flink State Backend（RocksDB + Checkpoint）
- Kafka（输入输出总线）

---

### 11) Batch Processing Service（离线计算服务）
**服务职责**
- Spark/Flink Batch 离线任务调度执行。
- ODS→DWD→DWS→ADS 构建与回溯重算。

**核心 API 接口**
- `POST /api/v1/batch-jobs`
- `GET /api/v1/batch-jobs/{jobId}`
- `POST /api/v1/batch-jobs/{jobId}:backfill`

**事件通信**
- 发布：`batch.job.completed`、`batch.dataset.published`
- 订阅：`transform.job.completed`、`workflow.started`

**数据库**
- Iceberg（分层数据）
- PostgreSQL（调度与运行元数据）

---

## 1.5 多模型存储与查询域（Polyglot Persistence Context）

### 12) Lake Service（数据湖服务）
**服务职责**
- 统一管理 Iceberg Catalog、分区策略、生命周期。
- 提供数据集注册、快照、回滚能力。

**核心 API 接口**
- `POST /api/v1/lake/datasets`
- `POST /api/v1/lake/datasets/{id}/snapshot`
- `POST /api/v1/lake/datasets/{id}:rollback`
- `GET /api/v1/lake/datasets/{id}/partitions`

**事件通信**
- 发布：`lake.dataset.created`、`lake.snapshot.created`
- 订阅：`batch.dataset.published`

**数据库**
- Iceberg Catalog（Hive Metastore / REST Catalog）

---

### 13) OLAP Service（分析查询服务）
**服务职责**
- 聚合模型管理、物化视图管理、OLAP 查询 API。
- 多租户查询配额与并发治理。

**核心 API 接口**
- `POST /api/v1/olap/queries`
- `POST /api/v1/olap/materialized-views`
- `GET /api/v1/olap/query/{queryId}`

**事件通信**
- 发布：`olap.query.completed`、`olap.sla.breached`
- 订阅：`batch.dataset.published`、`stream.metric.updated`

**数据库**
- ClickHouse（聚合数据）
- Trino（联邦查询入口）

---

### 14) Search Service（搜索服务）
**服务职责**
- 全文索引构建、检索、相关性排序。
- 同义词、分词器、索引模板管理。

**核心 API 接口**
- `POST /api/v1/search/indexes`
- `POST /api/v1/search/query`
- `POST /api/v1/search/reindex`

**事件通信**
- 发布：`search.index.updated`、`search.reindex.completed`
- 订阅：`transform.job.completed`、`stream.metric.updated`

**数据库**
- OpenSearch/Elasticsearch（索引存储）

---

### 15) Graph Service（图谱服务）
**服务职责**
- 实体点/关系边写入与版本化。
- 图查询（路径、邻居、社群）与图算法任务。

**核心 API 接口**
- `POST /api/v1/graph/entities`
- `POST /api/v1/graph/relations`
- `POST /api/v1/graph/query/path`
- `POST /api/v1/graph/query/neighbors`

**事件通信**
- 发布：`graph.entity.linked`、`graph.snapshot.created`
- 订阅：`transform.job.completed`、`batch.dataset.published`

**数据库**
- JanusGraph + Cassandra/Scylla（或 Neo4j）

---

## 1.6 可视化与交付域（Presentation Context）

### 16) Visualization Service（可视化服务）
**服务职责**
- 图谱可视化、指标看板、交互式分析页面。
- 查询编排（OLAP + 搜索 + 图）结果聚合。

**核心 API 接口**
- `GET /api/v1/dashboards/{id}`
- `POST /api/v1/dashboards`
- `POST /api/v1/analysis/query`

**事件通信**
- 发布：`dashboard.published`
- 订阅：`olap.query.completed`、`search.index.updated`

**数据库**
- PostgreSQL（看板配置）
- Redis（缓存）

---

## 1.7 治理与运维域（Governance & Operations Context）

### 17) Metadata Service（元数据服务）
**服务职责**
- 数据集、字段、标签、业务术语管理。
- 元数据开放查询与资产目录。

**核心 API 接口**
- `POST /api/v1/metadata/datasets`
- `GET /api/v1/metadata/datasets/{id}`
- `POST /api/v1/glossary/terms`

**事件通信**
- 发布：`metadata.dataset.registered`
- 订阅：`lake.dataset.created`、`batch.dataset.published`

**数据库**
- Neo4j / PostgreSQL（元数据图 + 关系）

---

### 18) Lineage Service（血缘服务）
**服务职责**
- 作业级、表级、字段级血缘采集。
- 影响分析与变更风险评估。

**核心 API 接口**
- `POST /api/v1/lineage/ingest`
- `GET /api/v1/lineage/datasets/{id}`
- `GET /api/v1/lineage/impact?field=...`

**事件通信**
- 发布：`lineage.updated`、`lineage.impact.detected`
- 订阅：`batch.job.completed`、`stream.job.failed`

**数据库**
- Graph DB（血缘图）

---

### 19) Data Quality Service（数据质量服务）
**服务职责**
- 质量规则（完整性、唯一性、及时性）执行。
- 质量评分与告警。

**核心 API 接口**
- `POST /api/v1/quality/rules`
- `POST /api/v1/quality/checks:run`
- `GET /api/v1/quality/reports/{datasetId}`

**事件通信**
- 发布：`quality.check.failed`、`quality.score.updated`
- 订阅：`batch.dataset.published`、`transform.job.completed`

**数据库**
- PostgreSQL（规则与报告）
- ClickHouse（质量时序结果）

---

### 20) Audit Service（审计服务）
**服务职责**
- 收集全链路操作审计、访问审计。
- 合规留痕与可追责检索。

**核心 API 接口**
- `POST /api/v1/audit/events`
- `GET /api/v1/audit/search`

**事件通信**
- 发布：`audit.risk.detected`
- 订阅：全域审计事件（topic fan-in）

**数据库**
- OpenSearch（审计检索）
- Object Storage（WORM 归档）

---

## 2. 服务依赖关系

## 2.1 核心依赖链（主流程）

1. `Case Service` -> `Template Service`（选择模板）
2. `Case Service` -> `Ingestion Service`（发起导入）
3. `Ingestion Service` -> `Transform Service`（触发标准化）
4. `Transform Service` -> `Lake Service`（写 DWD）
5. `Transform Service` -> `Search Service`/`Graph Service`（构建索引与图）
6. `Batch Processing Service` -> `Lake Service`（构建 DWS/ADS）
7. `OLAP Service` <- `Lake Service`/`Batch Processing Service`（分析读模型）
8. `Visualization Service` <- `OLAP Service` + `Search Service` + `Graph Service`

## 2.2 横切依赖

- 全部业务服务依赖 `IAM Service`（认证鉴权）
- 全部服务发布审计到 `Audit Service`
- 全部数据服务登记元数据到 `Metadata Service`
- 关键作业与 SQL 变更上报 `Lineage Service`
- 发布前后均触发 `Data Quality Service` 质检

## 2.3 依赖约束（DDD）

- 不允许跨域直接写对方数据库。
- 领域内强一致，跨领域通过事件最终一致。
- 查询可跨服务聚合，命令必须归属单聚合根。

---

## 3. 服务通信方式

## 3.1 同步通信（命令/查询）

- **协议**：REST/gRPC
- **场景**：
  - 用户交互命令（创建案件、发布模板）
  - 低延迟查询（查询任务状态、读取看板）
- **治理**：API Gateway + 限流 + 熔断 + 超时 + 重试

## 3.2 异步通信（事件驱动）

- **中间件**：Kafka
- **Topic 规范**：`{domain}.{aggregate}.{event}`，如 `case.case.created`
- **语义保障**：至少一次投递 + 消费端幂等；关键链路支持事务消息/outbox

## 3.3 数据同步通信

- CDC：Debezium/Flink CDC -> Kafka -> 下游服务
- 批同步：调度系统触发批任务，结果事件回传
- 回放：基于 Kafka 保留策略与 offset 重置

## 3.4 失败处理与一致性

- Saga 补偿：Workflow Service 管理多步骤失败回滚/补偿
- 幂等策略：`idempotency_key + version`
- 死信队列（DLQ）：不可恢复事件进入 DLQ 并告警

---

## 4. 服务边界（Bounded Context Boundary）

## 4.1 边界定义

- **Identity & Tenant**：身份、租户、授权
- **Case & Workflow**：案件状态与流程编排
- **Template & Rule**：模板 DSL 与战法规则
- **Ingestion & Processing**：导入、CDC、流批计算
- **Polyglot Persistence**：湖、仓、搜索、图
- **Presentation**：多引擎查询聚合与可视化
- **Governance & Ops**：元数据、血缘、质量、审计

## 4.2 反腐层（ACL）

- 外部系统（ERP/CRM/第三方情报）接入必须通过 Ingestion/CDC ACL。
- 历史遗留库通过 Anti-Corruption Adapter 映射到统一领域模型。

## 4.3 数据所有权

- 每个微服务拥有独立数据库（Database per Service）。
- 共享数据通过“发布事件 + 订阅构建只读副本”实现。

## 4.4 API 边界规则

- 对外 API：统一走 Gateway，附带 `tenant_id` 与权限上下文。
- 对内 API：Service Mesh 内网通信，mTLS 加密。

---

## 5. 推荐技术栈（微服务层）

- **服务框架**：Spring Boot / Go + gRPC
- **网关**：Kong / APISIX / Spring Cloud Gateway
- **服务治理**：Kubernetes + Istio
- **事件总线**：Kafka + Schema Registry
- **事务一致性**：Outbox Pattern + CDC
- **配置中心**：Nacos / Consul
- **可观测性**：OpenTelemetry + Prometheus + Grafana + Loki + Tempo
- **安全**：Keycloak（IAM）+ Vault（密钥管理）

---

## 6. 最小可落地版本（MVP）建议

第一阶段建议先落地 10 个核心服务：

1. IAM Service
2. Tenant Service
3. Case Service
4. Template Service
5. Ingestion Service
6. Transform Service
7. Stream Processing Service
8. Batch Processing Service
9. OLAP Service
10. Visualization Service

随后逐步扩展 Graph/Search/Lineage/Quality/Audit，降低初期复杂度并加快上线。
