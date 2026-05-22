---
name: kratos-dev-specs
description: Golang + Kratos v2 项目规范，适用于按 module/domain/infras/interfaces 分层的 Kratos 服务开发，包含仓储、GORM/gen、gorm cli SQL mapper、GORM 事务、缓存、proto、业务错误、日志和 DI 模板。
---

# Golang Kratos 项目规范

使用本 skill 处理 Golang + Kratos v2 项目中的新增模块、接口、仓储、缓存、业务错误、DI 接线和代码审查任务。项目代码需要符合 DDD 思想：领域模型和领域接口位于 domain，基础设施和外部触发器通过依赖倒置依赖 domain，而不是让 domain 依赖技术实现。

## 工作流程

1. 先阅读当前项目已有同类模块，优先复用既有 factory、service、repository、controller、DI、cache、proto、error-proto、slog 写法。
2. 明确外部 API 是否已经在上一级 `*-proto` 仓库定义；找不到 proto 位置或编译后的 proto Go 仓库时，先询问用户。
3. 按 DDD 依赖倒置组织代码：domain 定义领域模型和接口，infras/interfaces 依赖 domain 并实现或调用这些接口。
4. 涉及数据模型时先改 `infras/repository/db/model`；涉及复杂 SQL 时在 `infras/repository/db/mapper` 定义 SQL 入参/结果 DTO、mapper interface 和注释 SQL；再运行项目约定的 `mage gorm` 生成 `db/query` 与 `db/sqlquery`。
5. 涉及 proto 或 error-proto 时，不在业务项目里构建 proto；按 proto 仓库提交、CI/CD 生成、业务项目 `go get` 更新的流程处理。
6. 涉及多个数据库操作且需要原子性时，使用 `<project>-common/pkg/tx` 统一处理事务；domain 只依赖接口，不引入事务或 GORM 实现。
7. 完成后运行 `gofmt`、`go test ./...`，涉及服务注册、缓存或业务错误时启动服务或运行项目既有 mage 命令验证。

## 分层规范

### domain

`domain` 存放实体、工厂、领域服务、repository interface、search 条件。

- 内容少时直接平铺在 `domain/*.go`，如 `entity.go`、`factory.go`、`service.go`、`repository.go`。
- 内容文件较多或职责明显扩大时，再拆成 `domain/entity`、`domain/factory`、`domain/service`、`domain/repository` 子目录。
- domain 不依赖 `interfaces`、`infras`、`tx`、`gorm.DB`、GORM query、model、cache、proto controller 实现。
- repository interface 在 domain 定义，如 `<Module>Repository`；这是为了符合 DDD 的依赖倒置原则，让 domain 声明所需能力，由 infras 提供具体实现。

### infras

`infras` 是基础设施层，负责数据库、缓存、消息、第三方服务等技术实现，对外实现 domain 定义的接口，不能反向污染 domain。

- `infras/repository`：仓储/基础设施实现层，组合 `db/query`、`db/sqlquery`、model 转换、缓存读写与失效、分页查询、事务/批量保存、MQ 生产者等。
- `infras/repository/db/model`：数据库模型目录，只放持久化模型、表名、字段标签和必要模型方法。
- `infras/repository/db/query`：GORM/gen 生成目录，通过 `mage gorm` 自动生成；业务代码只能引用，不直接修改。
- `infras/repository/db/mapper`：复杂 SQL 的 gorm cli mapper 源码目录，手写 mapper interface、SQL 入参/结果 DTO 和注释 SQL。
- `infras/repository/db/sqlquery`：gorm cli 生成目录，通过 `gorm gen -i <mapper> -o <sqlquery>` 生成；业务代码只能引用，不直接修改。
- `infras/repository/cache`：缓存实现目录；缓存必须经由 `jetcache-go` 组件封装。
- repository 所有数据库访问都通过 `tx.GetTx(ctx, r.db)` 获取连接，再调用 `query.Use(...)` 或 `sqlquery.<Mapper>(...)`，以支持外层事务复用。
- repository 可在内部为多表写入、批量写入、读写组合等需要原子性的操作使用 `tx.Transaction` 开启事务。
- MQ 生产者属于对外部消息系统的发送实现，放在 `infras/repository`，由 domain 接口抽象后注入使用。

### interfaces

`interfaces` 存放所有外部触发器入口，包括 HTTP/gRPC 接口实现、MQ 消费者、定时任务触发器等。

- `interfaces/web`：HTTP/gRPC/proto service controller、请求/响应转换、Kratos error。
- `interfaces/mq`：MQ 消费者，负责消息反序列化、幂等边界处理和调用 domain repository/service。
- `interfaces/task`：定时/后台任务触发器，仅协调 domain repository/service。
- interfaces 只做协议适配、参数转换和调用，不承载复杂业务规则。
- 简单业务允许 interfaces 直接调用 domain repository 完成，例如单表查询、简单保存、删除、存在性检查。
- 复杂业务规则应放在 domain service 中完成，例如跨 repository 协作、多步骤校验、状态流转、缓存/消息联动策略等；interfaces 再调用 domain service。
- `interfaces/web`、`interfaces/mq`、`interfaces/task` 可注入 `*gorm.DB`，但只能用于 `tx.Transaction` 事务边界，不能直接写 ORM/query/model/cache，也不生产 MQ 消息。

### di.go

模块根目录的 `di.go` 统一注册依赖、缓存、外部触发器和服务端接口。

注册顺序通常为：cache -> factory -> repository with `dig.As` -> service -> controller/consumer/job -> gRPC/HTTP register。

## Go import 规范

- 默认不为本模块包添加别名；如果上下文中只引入一个 domain 包，直接使用 `domain`。
- 只有出现多个同名包、多个模块的 domain、公共包 domain，或存在 Go 语法冲突时，才使用 `<module>Domain`、`commonDomain` 等别名。
- 其他层包也遵循同样规则：先用默认包名，确有冲突或可读性需要时再加别名。

## Go 编码风格

- 错误处理默认不要折叠到 `if` 初始化语句里；先赋值给 `err`，再单独判断。
- 使用工具库 `github.com/samber/lo` 减少样板代码，处理 slice/map 转换、过滤、分组、去重、指针和值转换等通用逻辑。
- 禁止链式工具调用隐藏复杂业务规则、错误处理、事务边界或有副作用的流程。
- 优先复用项目 `go.mod` 已有工具库；如果需要新增工具库依赖，确认项目已有同类依赖和团队风格，避免为了少量简单逻辑引入新依赖。

## API/proto 规范

- 检查 `.agents/package.local.json` 的 `proto` 字段，确认 proto 项目目录。
- 检查 `.agents/package.local.json` 的 `sdk.go` 字段，确认 proto Go 模块。
- API 定义一般在执行项目根目录上一级的某个 `-proto` 后缀目录，例如 `../*-proto`。
- 需要查看 API 定义时，`package.local.json` 没有记录的情况下，先在上一级目录查找 `*-proto`；即使找到了疑似 proto 项目，也要询问用户确认。
- 用户确认后，在执行项目根目录下创建或更新 `.agents/package.local.json`，用字段 `proto` 记录 proto 项目目录。
- 业务项目引用编译后的 proto SDK；proto SDK 通常已在 `go.mod` 中引入，模块名一般以 `-proto-go` 结尾。
- 如果 `go.mod` 没有发现 proto SDK，询问用户对应的 proto SDK Go 模块，并记录到 `.agents/package.local.json` 的 `sdk.go` 字段。
- 不要在业务项目中构建 proto 文件。
- 如果需要修改 proto，必须先获得修改权限；改完后通知用户提交 proto 仓库，由 CI/CD 自动构建 proto。
- 用户确认 CI/CD 完成后，在业务项目中通过 `go get <proto-go-module>@latest` 或指定版本拉取最新编译后的 proto Go 仓库。

## error-proto 规范

需要抛出业务错误时，优先使用 error-proto 定义的错误 reason 和生成的 Go helper，不手写普通错误代替业务错误。

- 参考 `../<project>-proto/api/common/v1/<module>_error_reason.proto`。
- 新增业务错误时，在对应 `<module>_error_reason.proto` 中添加 enum value。
- 使用 `errors/errors.proto` 的 `(errors.code)` 指定 HTTP/gRPC 映射码。
- 默认错误码使用 `option (errors.default_code)`。
- 业务代码使用生成的 `v1.ErrorXxx(...)` helper。
- 修改 error-proto 后同样走 proto 仓库提交、CI/CD 生成、业务项目 `go get` 更新流程。

## ORM/SQL 规范

- 常规 CRUD 和查询优先使用 `infras/repository/db/query` 的 GORM/gen query。
- 不直接使用裸 `gorm.DB` 写简单业务查询，也不在 repository 实现中手写 `Raw` SQL。
- repository 中使用 GORM/gen 时，通过 `query.Use(tx.GetTx(ctx, r.db))` 获取 query；不要长期持有预先绑定 `r.db` 的 `*query.Query`。
- 复杂数据库操作需要 SQL 时，在 `infras/repository/db/mapper` 定义 mapper interface、SQL 入参/结果 DTO 和注释 SQL，由 gorm cli 生成 `infras/repository/db/sqlquery`。
- mapper 文件使用 `package mapper`，引入 `gorm.io/cli/gorm/genconfig`，并配置 `genconfig.Config{IncludeInterfaces: []any{"<Module>Mapper"}, ExcludeStructs: []any{"*"}}`。
- SQL 入参和 SQL 结果映射需要额外结构体时，统一使用 `DTO` 后缀命名，例如 `<Module>SearchDTO`、`<Module>StatisticsDTO`；mapper interface 使用 `type <Module>Mapper[T any] interface { ... }`。
- repository 实现通过 `sqlquery.<Module>Mapper[*model.<Model>](tx.GetTx(ctx, r.db))` 调用生成方法。
- 新增或修改 `db/model` 或 `db/mapper` 后，通过 `mage gorm` 重新生成 `db/query` 和 `db/sqlquery`。
- 不手改 `db/query`、`db/sqlquery` 或其他 `*.gen.go`。

## 事务规范

- 统一使用 `<project>-common/pkg/tx` 处理 GORM 事务；项目 import 通常为 `../<project>-common/pkg/tx`，模板中可按项目替换为 `<project>/pkg/tx`。
- 需要多个数据库操作保持原子性时，使用 `tx.Transaction(ctx, db, func(ctx context.Context) error { ... })`；单次查询或单次简单写入默认不包事务。
- 事务回调内必须继续传递回调参数里的 `ctx`，不要继续使用外层 ctx；下游 repository 通过 `tx.GetTx(ctx, r.db)` 复用同一个事务。
- `tx.Transaction` 已处理嵌套事务：ctx 中已有事务时直接执行回调，避免重复开启事务。
- 外部触发器可注入 `*gorm.DB` 仅用于开启事务边界，例如包裹多次 repository/service 调用；不得直接访问 GORM query、model 或 cache。
- repository 内部可为多表写入、批量写入、读写组合等开启事务；事务内仍使用 `query.Use(tx.GetTx(ctx, r.db))` 和 `sqlquery.<Mapper>(tx.GetTx(ctx, r.db))`。
- 不手写 `Begin`、`Commit`、`Rollback`，不使用 GORM/gen `Transaction` 绕过统一 context 事务，不直接用 `r.db` 或预存 `*query.Query` 发起业务查询。
- domain service 只编排领域逻辑和调用 domain repository interface，不引入 `tx`、`gorm.DB`、GORM query 或 model。

## 缓存规范

需要缓存时，严格使用 `github.com/mgtv-tech/jetcache-go` 作为缓存组件封装。

- 默认使用 `jetcache.NewT[K,V]` 泛型缓存。
- 本地缓存默认使用 `local.NewTinyLFU(maxCost, ttl)`，通过 `jetcache.WithLocal(...)` 和 `jetcache.WithStatsDisabled(true)` 初始化。
- 底层实现只在 `infras/repository/cache` 组件内部选择和封装。
- 不在业务代码中直接绕过 cache 组件操作底层缓存，也不新增旧缓存库依赖。
- repository 注入 cache 后使用 cache-aside 模式。
- `FindByID` 先读 cache，未命中查 query 后 Put。
- `Save`、`Delete`、批量变更成功后及时失效相关 key。
- cache key 使用模块前缀；多个索引缓存使用多个 `*jetcache.T[K,V]` 字段和独立 key 常量。
- 缓存未命中或删除失败不应影响数据库主流程。

## 日志规范

业务代码统一使用标准库 `log/slog`，不要直接使用 `github.com/go-kratos/kratos/v2/log`。

- 使用结构化字段，但默认直接传 key/value：`slog.InfoContext(ctx, "create user", "user_id", userID)`；不要为了简单字段写成 `slog.String("user_id", userID)` 这类更复杂形式。
- 不把关键变量拼接进 message。
- 只记录有诊断价值的边界事件、异常和关键状态。
- 如果当前作用域有 `ctx context.Context`，优先使用 `slog.InfoContext(ctx, ...)`、`slog.ErrorContext(ctx, ...)`、`slog.WarnContext(ctx, ...)`、`slog.DebugContext(ctx, ...)`。
- 没有 ctx 时才使用 `slog.Info(...)` 等非 Context 方法。

## 注释规范

- 每个函数至少要有一个符合 Go 规范的函数注释，注释以函数名开头并描述用途。
- Controller、service、factory、cache、consumer、job、DI provider 等函数都需要函数注释。
- 函数注释格式：

```go
// <函数名称> <函数简单描述>
// <函数详细描述>
//
// param:
//   - <参数1名称>: <参数1描述>
//   - <参数2名称>: <参数2描述>
//
// return:
//   - <返回1名称>: <返回1描述>
//   - <返回2名称>: <返回2描述>
```

- 函数详细描述可以省略；算法相关函数必须有详细描述。
- `param` 描述省略 `ctx`、`emptypb.Empty` 等空请求；如果没有需要描述的参数，则完全省略 `param` 段。
- 如果末尾只有空占用结构体或error，则可以省略return描述。如果没有需要描述的返回，则完全省略 `return` 段。
- Repository 仓储接口和仓储实现的方法，除 New 函数外，除了简单函数描述，还需要按上述格式描述非 `ctx` 参数和非 `error`、非空响应返回。
- `param:` 和 `return:` 后必须紧跟具体内容，不能出现空行；`-` 应该对齐 `param:` 的 `r` 或 `return:` 的 `t`，即在 `param` 或 `return` 同级缩进 2 个空格。
- 复杂逻辑代码段需要注释说明意图或关键步骤，例如批量查询、缓存缺失计算、事务内多步骤处理、复杂 SQL、幂等处理。
- 注释描述业务意图和参数含义，不写流水账式代码翻译。
- 各种注释末尾省略没有意义的句号。
- 建议查看[templates/comments.md](templates/comments.md)中的示例模板。

## 非空编程规范

除外部触发器和数据库模型外，其他业务/基础设施代码的查询结果都必须使用容器包装。

- optional、切片、数组、map 等容器包裹的数据，如果不是基本数据类型，则必须使用指针（例如 `optional.Option[*domain.Entity]` 而不是 `optional.Option[domain.Entity]`）。
- 单个可能不存在的结果使用 `optional.Option[T]`。
- 集合结果使用 slice 或 map。
- 除 `error` 外不要直接返回 nil。
- 未命中返回 `optional.None[T]()`。
- 空集合返回空 slice/map，例如 `[]*domain.Entity{}` 或 `map[int64]*domain.Entity{}`。
- API/proto 生成类型、controller response 和数据库 model 不纳入该限制。
- 使用 `github.com/moznion/go-optional` 时，优先复用其内置工具函数，不随意新增自定义转换函数；常用工具函数见 [templates/optional-tools.md](templates/optional-tools.md)。

## 金额与精度计算规范

当需要进行金额、计费、金融或精度计算时，必须使用 `github.com/shopspring/decimal` 库进行计算，不要使用基本数据类型（如 `float64` 等）以免丢失精度。

## 禁止事项

- 不手改 `db/query`、`db/sqlquery` 或其他 `*.gen.go`。
- 不在金额、计费、金融或精度计算中使用基本数据类型。
- 不在业务项目中生成 proto。
- 不绕过 error-proto 手写业务错误。
- 不直接使用 Kratos log。
- 不在有 ctx 时丢弃上下文写日志。
- 不让 domain 依赖 `tx`、`gorm.DB`、GORM query、model、cache 等基础设施实现。
- 不让 controller、consumer、task 直接写 ORM/SQL 查询或直接操作缓存。
- 不绕过 `<project>-common/pkg/tx` 手写 GORM 事务，或使用预存 `*query.Query`、GORM/gen `Transaction` 破坏 context 事务复用。
- 不在 repository 实现中手写 `Raw` SQL；复杂 SQL 必须放到 `db/mapper` 并生成 `db/sqlquery`。
- 不在可返回 `optional.None[T]()` 或空 slice/map 的查询路径上直接返回 nil。
- 不为少量文件过早拆目录或过度抽象。

## 模板

- 新增模块结构：[templates/module.md](templates/module.md)
- Controller：[templates/controller.md](templates/controller.md)
- Repository：[templates/repository.md](templates/repository.md)
- Model：[templates/model.md](templates/model.md)
- SQL mapper/DTO：[templates/dto.md](templates/dto.md)
- Cache：[templates/cache.md](templates/cache.md)
- Interfaces：[templates/interfaces.md](templates/interfaces.md)
- Proto/error-proto：[templates/proto.md](templates/proto.md)
- Logging：[templates/logging.md](templates/logging.md)
- Optional/non-nil returns：[templates/optional.md](templates/optional.md)
- DI：[templates/di.md](templates/di.md)

## 验证清单

- `gofmt` 已运行。
- 涉及 model/query/mapper/sqlquery 时已运行 `mage gorm`。
- repository 所有 DB 调用都通过 `tx.GetTx(ctx, r.db)` 支持外层事务复用。
- 需要原子性的多步数据库操作已使用 `tx.Transaction`，事务内传递回调 ctx。
- 涉及 proto Go 依赖时，已在用户确认 proto CI/CD 完成后运行 `go get`。
- `go test ./...` 通过。
- 涉及服务注册、业务错误、日志或缓存接入时，已启动服务或运行项目既有 mage 命令验证。
