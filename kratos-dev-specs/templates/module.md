# 新增模块模板

## 简单模块结构

适用于文件数量少、职责清晰的模块。

```text
module/<module>/
├── domain/
│   ├── entity.go
│   ├── factory.go
│   ├── service.go
│   └── repository.go
├── infras/
│   └── repository/
│       ├── db/
│       │   ├── model/
│       │   │   └── model.go
│       │   ├── query/
│       │   │   └── *.gen.go
│       │   ├── mapper/
│       │   │   └── mapper.go
│       │   └── sqlquery/
│       │       └── mapper.go
│       ├── cache/
│       │   └── cache.go
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
│       │   ├── query/
│       │   ├── mapper/
│       │   └── sqlquery/
│       ├── cache/
│       └── repository.go
├── interfaces/
│   ├── web/
│   ├── mq/
│   └── task/
└── di.go
```

## 约定

- 不要为少量文件过早拆目录。
- domain 定义接口和业务概念，infras 实现外部系统访问，interfaces 放外部触发器。
- 数据模型放 `infras/repository/db/model`。
- GORM/gen 文件由 `mage gorm` 生成到 `infras/repository/db/query`，不要手改。
- 复杂 SQL 在 `infras/repository/db/mapper` 中定义 SQL 入参/结果 DTO、mapper interface 和注释 SQL，由 `mage gorm` 生成到 `infras/repository/db/sqlquery`，不要手改生成文件。
- 缓存组件放 `infras/repository/cache`，必须通过 `jetcache-go` 封装。
- MQ producer 放 `infras/repository`；MQ consumer 放 `interfaces/mq`。
- API/proto 定义在上一级 `xxx-proto` 仓库；业务项目只引用编译后的 proto Go 包。
