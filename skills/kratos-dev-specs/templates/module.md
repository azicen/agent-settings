# 新增模块模板

## 简单模块结构

简单模块先保持平铺；文件和职责真实增长后再拆分子目录。不要为预期中的复杂度预建目录。

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
│       │   ├── query/           # 生成物
│       │   │   └── *.gen.go
│       │   ├── mapper/
│       │   │   └── mapper.go
│       │   └── sqlquery/        # 生成物
│       │       └── mapper.go
│       └── repository.go
├── interfaces/
│   ├── web/
│   │   └── controller.go
│   ├── mq/
│   │   └── consumer.go
│   └── task/
│       └── job.go
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
│       │   ├── query/          # 生成物
│       │   ├── mapper/
│       │   └── sqlquery/       # 生成物
│       ├── cache/
│       └── repository.go
├── interfaces/
│   ├── web/
│   ├── mq/
│   └── task/
└── di.go
```


- `domain` 只声明业务概念、实体、工厂、领域服务与由消费方定义的 repository interface；不得依赖 GORM、缓存、事务、proto 或 interfaces/infras。
- `infras/repository` 实现数据库、缓存、MQ producer 和第三方访问；model 嵌入 common `BaseModel` 或 `SnowflakeModel`，并按实际生成命令更新 query/sqlquery，绝不手改生成物。
- `interfaces` 只适配 HTTP/gRPC、消息和任务等外部触发器；MQ consumer 位于 `interfaces/mq`，MQ producer 位于 infras 并由 domain interface 抽象。
- `di.go` 导出 `Register() fx.Option`，在应用组合根接入；资源生命周期归 Fx，模块不能自行启动 Kratos app。
- 外部 API 只依赖已发布 proto SDK；业务仓仅生成自身 `pkg/config` proto。详见 [proto.md](proto.md)。
