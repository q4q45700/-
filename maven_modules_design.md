# 系统 Maven 模块设计（最小依赖版）

## 1. 各层级目录（按模块分层级）

```text
.
├── pom.xml                                 # 根聚合 POM（platform-parent）
├── platform-base                           # 基础模块层
│   ├── platform-base-api                   # 请求/返回对象、通用响应模型
│   │   └── pom.xml
│   ├── platform-base-utils                 # 常用工具类
│   │   └── pom.xml
│   └── platform-base-security              # 权限模型、鉴权上下文、注解
│       └── pom.xml
├── platform-tools                          # 工具模块层
│   ├── tool-db                             # 数据库工具封装
│   │   └── pom.xml
│   ├── tool-search                         # 搜索引擎工具封装
│   │   └── pom.xml
│   └── tool-graph                          # 图数据库工具封装
│       └── pom.xml
├── platform-biz                            # 业务模块层
│   ├── biz-ingestion                       # 导入、清洗、入库业务
│   │   └── pom.xml
│   ├── biz-template                        # 模板管理、模板解析业务
│   │   └── pom.xml
│   └── biz-case                            # 案件业务
│       └── pom.xml
└── platform-services                       # 微服务部署层
    ├── service-gateway                     # 对外网关服务
    │   └── pom.xml
    ├── service-ingestion                   # 导入微服务
    │   └── pom.xml
    └── service-analysis                    # 分析微服务
        └── pom.xml
```

---

## 2. 模块职责与依赖策略

- **基础模块（base）**
  - 只放“无业务语义”的通用对象与能力：请求/返回、工具、权限模型。
  - 不依赖业务模块。
- **工具模块（tools）**
  - 只封装外部基础设施访问（DB/ES/Graph），不承载业务流程。
  - 不依赖业务模块。
- **业务模块（biz）**
  - 编排业务流程（导入、清洗、模板、案件）。
  - 依赖 base + 必要 tools。
- **微服务模块（services）**
  - 对外提供 API，组装对应 biz。
  - 只依赖自己所需 biz，不引入无关工具。

---

## 3. 权限放在哪层？

**结论：权限放在“基础模块（platform-base-security）”更合适。**

原因：

1. 权限是跨业务的横切能力（所有业务都要用），不是某个基础设施工具。
2. 工具模块应聚焦“访问外部系统”，若把权限放到 tools，会导致语义混乱。
3. 放在 base-security 后，biz 和 service 都能复用同一套鉴权上下文/注解/策略接口。

建议分工：

- `platform-base-security`：
  - 权限模型（角色、资源、动作）
  - 鉴权上下文（当前租户、当前用户）
  - 方法级鉴权注解与拦截接口
- `tool-*`：
  - 仅提供 DB/ES/Graph 客户端封装
  - 不包含权限决策逻辑

---

## 4. 最小依赖说明

- 每个模块只引入当前职责必须的依赖。
- 可选能力（例如某业务暂未启用图/搜索）使用 `optional=true`，避免强传递依赖。
- 服务层只引入其对应业务模块，避免“大而全”依赖。

---

## 5. 已生成的 POM 文件清单

### 根聚合
- `pom.xml`

### 基础模块
- `platform-base/platform-base-api/pom.xml`
- `platform-base/platform-base-utils/pom.xml`
- `platform-base/platform-base-security/pom.xml`

### 工具模块
- `platform-tools/tool-db/pom.xml`
- `platform-tools/tool-search/pom.xml`
- `platform-tools/tool-graph/pom.xml`

### 业务模块
- `platform-biz/biz-ingestion/pom.xml`
- `platform-biz/biz-template/pom.xml`
- `platform-biz/biz-case/pom.xml`

### 微服务模块
- `platform-services/service-gateway/pom.xml`
- `platform-services/service-ingestion/pom.xml`
- `platform-services/service-analysis/pom.xml`

---

## 6. 依赖关系（简图）

```text
service-* -> biz-* -> (platform-base-* + tool-*)
                         ^
                         └─ platform-base-security（权限）
```

