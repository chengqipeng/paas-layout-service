# paas-layout-service 技术设计方案

## 1. 服务概述

布局服务，负责 aPaaS 平台的 UI 布局元数据管理与运行时渲染支持。管理表单布局、列表视图、组件实例、菜单导航等 UI 配置，并提供聚合接口将布局、元数据、数据、权限、多语言整合为前端可直接渲染的数据包。

---

## 2. 系统架构

```mermaid
graph TB
    subgraph 入口层
        REST_V2[REST API v2]
        REST_V3[REST API v3]
        REST_INNER[内部 REST API]
        RPC[RPC Provider]
    end

    subgraph 核心业务层
        LayoutSvc[LayoutService<br>布局 CRUD]
        AggSvc[AggregatorService<br>布局聚合]
        NavSvc[NavigationService<br>菜单导航]
        AppletSvc[AppletService<br>小程序布局]
        GASvc[GAService<br>引导分析]
        DevOpsSvc[DevOpsService<br>沙箱迁移]
    end

    subgraph 外部依赖
        MetaRepo[paas-metarepo-service<br>元数据]
        EntitySvc[paas-entity-service<br>数据]
        PrivilegeSvc[paas-privilege-service<br>权限]
    end

    subgraph 基础设施层
        DB[(MySQL/PostgreSQL<br>布局数据)]
        Redis[Redis<br>布局缓存]
        MQ[RocketMQ<br>缓存刷新]
        Nacos[Nacos]
    end

    REST_V2 --> LayoutSvc
    REST_V3 --> LayoutSvc
    REST_V3 --> AggSvc
    REST_INNER --> DevOpsSvc
    RPC --> LayoutSvc

    AggSvc --> MetaRepo
    AggSvc --> EntitySvc
    AggSvc --> PrivilegeSvc
    AggSvc --> LayoutSvc

    LayoutSvc --> DB
    LayoutSvc --> Redis
    NavSvc --> DB
    AppletSvc --> DB

    MQ --> LayoutSvc
```

---

## 3. 布局类型体系

```mermaid
graph LR
    Layout[布局] --> FormLayout[表单布局<br>详情页/编辑页]
    Layout --> ListLayout[列表布局<br>列表视图]
    Layout --> ComponentLayout[组件实例<br>自定义组件]
    Layout --> ExtensionLayout[扩展布局<br>页面扩展]
    Layout --> GALayout[GA 布局<br>引导分析]
    Layout --> AppletLayout[小程序布局<br>移动端]
```

---

## 4. 模块职责

### 4.1 LayoutService（布局 CRUD）

管理所有类型布局的增删改查。

| 方法 | 说明 |
|---|---|
| `createLayout` | 创建布局，校验字段引用合法性 |
| `updateLayout` | 更新布局配置 |
| `deleteLayout` | 删除布局 |
| `getLayout` | 查询布局详情（走缓存） |
| `listLayouts` | 查询对象下的布局列表 |
| `upgradeLayout` | 布局升级（元数据变更后同步布局） |
| `copyLayout` | 复制布局 |

**布局存储结构：**
- 布局配置以 JSON 格式存储（`layout_config` 字段）
- 支持设计态（design）和数据态（data）两种视图
- 布局版本管理，支持回滚

### 4.2 AggregatorService（布局聚合）

核心聚合服务，将多个下游服务的数据整合为前端可直接渲染的数据包。

**聚合内容：**

| 数据来源 | 内容 |
|---|---|
| paas-metarepo-service | 字段定义、字段类型、选项值 |
| paas-entity-service | 记录数据、关联记录数据 |
| paas-privilege-service | 字段权限（可读/可写/隐藏）、按钮权限 |
| 本服务 | 布局配置、组件配置 |
| i18n | 字段标签多语言、选项值多语言 |

**聚合流程：**

```mermaid
flowchart TD
    A[接收聚合请求<br>objectApiKey + recordId + layoutId] --> B[并行获取数据]
    B --> C1[获取布局配置]
    B --> C2[获取元数据<br>字段定义]
    B --> C3[获取记录数据]
    B --> C4[获取字段权限]
    C1 --> D[合并数据]
    C2 --> D
    C3 --> D
    C4 --> D
    D --> E[按权限过滤字段<br>隐藏无权限字段]
    E --> F[多语言翻译<br>字段标签/选项值]
    F --> G[格式化字段值<br>日期/货币/关联]
    G --> H[返回聚合结果]
```

**并行调用优化：**
- 布局配置、元数据、记录数据、权限四路并行获取
- 使用 `CompletableFuture.allOf()` 等待所有结果
- 单次聚合 P99 目标 < 200ms

### 4.3 NavigationService（菜单导航）

管理应用的菜单和导航配置。

- 支持多级菜单树结构
- 菜单项关联对象、布局、外部链接
- 按用户权限过滤不可见菜单
- 支持移动端和 PC 端不同菜单配置

### 4.4 GAService（引导分析布局）

管理 Google Analytics 风格的引导分析布局配置。

- 支持漏斗分析、路径分析等布局
- 布局配置与数据分析维度绑定
- 支持布局模板复用

### 4.5 DevOpsService（沙箱迁移）

支持布局配置从沙箱环境迁移到生产环境。

- 生成布局变更 diff
- 按依赖顺序迁移（先迁移被引用的布局）
- 迁移后触发缓存刷新

---

## 5. 数据模型

```mermaid
classDiagram
    class Layout {
        +String id
        +String tenantId
        +String objectApiKey
        +String layoutType
        +String name
        +String configJson
        +String status
        +Long version
        +Boolean isDefault
        +Date createdAt
        +Date updatedAt
    }

    class ComponentInstance {
        +String id
        +String tenantId
        +String layoutId
        +String componentType
        +String configJson
        +Integer sortOrder
    }

    class Navigation {
        +String id
        +String tenantId
        +String parentId
        +String name
        +String icon
        +String targetType
        +String targetId
        +Integer sortOrder
        +Boolean isVisible
    }

    class LayoutExtension {
        +String id
        +String tenantId
        +String objectApiKey
        +String extensionType
        +String configJson
        +String pageType
    }

    Layout "1" --> "N" ComponentInstance
    Navigation "1" --> "N" Navigation : children
```

**configJson 示例（表单布局）：**
```json
{
  "sections": [
    {
      "label": "基本信息",
      "columns": 2,
      "fields": [
        { "fieldApiKey": "Name__c", "required": true, "readonly": false },
        { "fieldApiKey": "Status__c", "required": false }
      ]
    }
  ],
  "buttons": ["save", "cancel", "delete"]
}
```

---

## 6. 核心流程

### 6.1 布局聚合查询（详情页）

```mermaid
sequenceDiagram
    participant Client
    participant AggAPI
    participant AggService
    participant LayoutSvc
    participant MetaRepo
    participant EntitySvc
    participant PrivilegeSvc

    Client->>AggAPI: GET /api/v3/layout/aggregate?objectApiKey=&recordId=&layoutId=
    AggAPI->>AggService: aggregate(req)

    par 并行获取
        AggService->>LayoutSvc: getLayout(layoutId)
        AggService->>MetaRepo: listFieldMeta(objectApiKey)
        AggService->>EntitySvc: getRecord(objectApiKey, recordId)
        AggService->>PrivilegeSvc: getFieldPermissions(userId, objectApiKey)
    end

    AggService->>AggService: 合并数据
    AggService->>AggService: 按权限过滤字段
    AggService->>AggService: 多语言翻译
    AggService->>AggService: 格式化字段值
    AggService-->>AggAPI: AggregateVO
    AggAPI-->>Client: Result<AggregateVO>
```

### 6.2 布局缓存刷新（MQ 驱动）

```mermaid
flowchart TD
    A[接收 MQ 事件] --> B{事件类型}
    B -- 元数据变更 --> C[查找引用该对象的所有布局]
    B -- 布局直接变更 --> D[定位具体布局]
    C --> E[批量删除 Redis 缓存]
    D --> E
    E --> F[发布本地缓存失效广播]
```

**监听的 MQ Topic：**
- `paas.metadata.field.changed` → 刷新引用该字段的布局缓存
- `paas.metadata.object.changed` → 刷新该对象所有布局缓存
- `paas.layout.changed`（自身发布）→ 刷新具体布局缓存

---

## 7. 接口设计

### 7.1 REST 接口

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/v2/layouts` | 创建布局 |
| PUT | `/api/v2/layouts/{id}` | 更新布局 |
| DELETE | `/api/v2/layouts/{id}` | 删除布局 |
| GET | `/api/v2/layouts/{id}` | 查询布局详情 |
| GET | `/api/v2/layouts` | 查询布局列表 |
| GET | `/api/v3/layout/aggregate` | 布局聚合查询（详情页） |
| GET | `/api/v3/layout/list-aggregate` | 布局聚合查询（列表页） |
| GET | `/api/v2/navigation` | 查询菜单导航 |
| POST | `/api/v2/layouts/{id}/upgrade` | 布局升级 |
| POST | `/internal/devops/migrate` | 沙箱迁移 |

### 7.2 RPC 接口（core module）

```java
@FeignClient(name = "paas-layout-service")
public interface LayoutApi {
    // 查询布局配置
    Result<LayoutDTO> getLayout(String tenantId, String layoutId);

    // 查询对象默认布局
    Result<LayoutDTO> getDefaultLayout(String tenantId, String objectApiKey, String layoutType);

    // 布局聚合（供其他服务调用）
    Result<AggregateDTO> aggregate(AggregateRequest request);
}
```

---

## 8. 缓存策略

| 缓存 Key | 内容 | TTL | 失效时机 |
|---|---|---|---|
| `layout:{tenantId}:{layoutId}` | 布局配置 | 30min | 布局变更/元数据变更 |
| `layout:default:{tenantId}:{objectApiKey}:{type}` | 默认布局 | 30min | 布局变更 |
| `nav:{tenantId}:{userId}` | 用户菜单 | 10min | 菜单变更/权限变更 |

---

## 9. 异常处理

| 异常场景 | 处理策略 |
|---|---|
| 布局引用不存在的字段 | 返回 `LAYOUT_FIELD_NOT_FOUND`，列出缺失字段 |
| 聚合下游超时 | 部分降级：返回已获取的数据，缺失部分用空值填充 |
| 元数据变更导致布局不兼容 | 标记布局为 `NEED_UPGRADE` 状态，提示用户升级 |
| 沙箱迁移冲突 | 返回冲突详情，支持强制覆盖或跳过 |


---

## 10. 数据存储说明

layout-service **不直接维护独立的业务数据库表**。布局配置（表单布局、列表视图、菜单导航等）存储在 `paas-metarepo-service` 的 `xsy_metarepo` schema 中，对应 `p_custom_layout`、`p_meta_neoui_*` 等表，通过 metarepo 的 RPC 接口读写。

<!-- DDL 章节已移除，layout-service 无独立数据库表 -->
