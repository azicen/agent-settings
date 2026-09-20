# 新增模块模板

## 简单模块结构

简单模块先保持平铺；文件和职责真实增长后再拆分子目录。不要为预期中的复杂度预建过多层级。

```text
module/<module>/
├── domain/
│   ├── entity.go
│   ├── factory.go
│   ├── service.go
│   └── repository.go
├── infras/
│   └── repository/
│       ├── cache/
│       │   └── cache.go
│       ├── db/
│       │   ├── model/
│       │   │   └── model.go
│       │   ├── query/           # gorm.io/gen 自动生成物
│       │   │   └── *.gen.go
│       │   ├── mapper/          # 复杂 SQL 与 DTO 定义
│       │   │   └── mapper.go
│       │   └── sqlquery/        # gorm.io/cli/gorm 自动生成物
│       │       └── mapper.go
│       └── repository.go
├── interfaces/
│   ├── web/
│   │   └── controller.go
│   ├── mq/
│   │   └── consumer.go
│   └── task/
│       └── job.go
├── server/                      # 可选：模块自定义协议或轮询 Server (实现 transport.Server)
│   └── server.go
└── di.go
```

## 复杂模块结构

当 entity、factory、service、repository 文件数量增多或职责明显扩大时，再拆子目录。

```text
module/<module>/
├── domain/
│   ├── entity/
│   ├── factory/
│   ├── service/
│   └── repository/
├── infras/
│   └── repository/
│       ├── db/
│       │   ├── model/
│       │   ├── query/          # gorm.io/gen 自动生成物
│       │   ├── mapper/         # 复杂 SQL 接口定义
│       │   └── sqlquery/       # gorm.io/cli/gorm 自动生成物
│       ├── cache/
│       └── repository.go
├── interfaces/
│   ├── web/
│   ├── mq/
│   └── task/
├── server/
└── di.go
```

## 分层核心规范

- **`domain`**：只声明纯粹的业务概念、实体、工厂、领域服务与由消费方定义的 repository interface；不得依赖 GORM、缓存、事务或基础设施。实体属性中的可空字段与未命中表达统一使用 `<project>-common/pkg/optional`。
- **`infras/repository`**：实现数据库、缓存、MQ producer 和第三方数据访问的基础设施。数据库 model 必须嵌入 common 的 `BaseModel` 、 `UUIDModel` 或 `SnowflakeModel`。
  - 单表类型安全 CRUD 由 `gorm.io/gen` 自动生成到 `db/query`；
  - 多表关联、报表及复杂 SQL 由 `gorm.io/cli/gorm` 根据 `db/mapper` 中的注释 SQL 生成到 `db/sqlquery`；
  - 绝不手写裸 SQL 拼装，绝不手改 `db/query`、`db/sqlquery` 等生成物。
- **`interfaces`**：只适配外部调用，负责协议解码、输入校验、幂等边界与调用编排。HTTP/gRPC controller 位于 `interfaces/web`；MQ consumer 位于 `interfaces/mq`；定时任务/批处理调度位于 `interfaces/task`。
- **`server`**：若模块包含长期运行的后台协议轮询（如 Modbus 轮询、设备镜像同步等），将其封装为实现了 `kratos/v3/transport.Server` 接口的独立 Server，并在组合根作为 `kratos.Server(...)` 统一纳管。
- **`di.go`**：导出 `Register() fx.Option`，使用 `fx.Module("<name>", ...)` 隔离命名空间；资源生命周期全部委托 `fx.Lifecycle`，模块禁止自行启动或关闭 Kratos app。
- **Proto 契约**：外部 API 只依赖已发布的 proto SDK；业务仓仅生成自身 `pkg/config` proto。详见 [proto.md](proto.md)。
