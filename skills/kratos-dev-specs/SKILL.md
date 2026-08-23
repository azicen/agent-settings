---
name: kratos-dev-specs
description: Golang + Kratos v2 项目实现规范，覆盖 DDD 分层、Fx 依赖注入、启动、事务、GORM 生成、缓存、Proto、日志与容器配置。
---

# Golang Kratos 项目规范

适用于采用 Kratos v2 的 Go 服务。先以项目事实为准，再使用本规范的可替换模板；不凭仓库命名、版本或目录结构猜测依赖。

## 实施前检查与占位符

开始修改前读取 `go.mod`、现有 imports、`container/`、Mage 文件、`buf` 配置、`Dockerfile` 和 CI 配置，记录实际模块路径、版本、生成命令与运行参数。模板中的占位符必须替换为发现结果：

- `<project>`：业务 Go module。
- `<project>-common`：公共库 Go module。
- `<project>-proto-go`：已发布的 proto SDK Go module。
- `<registry>`：镜像仓库地址。

按模板实际使用范围确认 common 的 `pkg/optional`、`pkg/tx`、配置、migration、server 包、GORM、已发布 proto SDK 与 `go.uber.org/fx`。缺少某项时，报告缺失项及发现位置，并仅停止依赖该项的模板实现；不得以替代 DI 容器、第三方 Optional 或复制基础实现回退。

## 启动和依赖注入

依赖注入统一使用 Fx。模块导出 `Register() fx.Option`，组合使用 `fx.Module`、`fx.Options`、`fx.Provide`、`fx.Annotate`、`fx.As`、`fx.Invoke` 和 `fx.Out`；`fx.Out` group tag 必须与 common constructor 实际 `fx.In` 定义一致。接口绑定使用 `fx.Annotate(provider, fx.As(new(domain.Repository)))`。不要在构造函数外创建全局资源。

所有外部资源由 `fx.Lifecycle` 管理：provider 创建资源，`OnStart` 建立连接或启动，`OnStop` 关闭或停止并等待退出。Kratos app 只能通过一个 Kratos-Fx lifecycle bridge 接入 lifecycle；bridge 以 goroutine 运行 app、在停止时 Stop 并等待，模块不得自行启动或停止 Kratos app。

启动顺序固定为：

1. 解析 `-conf`。
2. 调用 `<project>-common/pkg/config.BuildSources`。
3. 以 `config.New(config.WithSource(...))` 创建配置，`Load` 后分别 `Scan` common 与 project `Bootstrap`。
4. 调用 common migration `Up`。
5. 调用 project migration `Up`。
6. 构建 Fx app 并 `Run`。

详见 [templates/startup.md](templates/startup.md)、[templates/di.md](templates/di.md) 与 [templates/server-options.md](templates/server-options.md)。

## 分层与数据访问

- `domain` 存放实体、工厂、领域服务、repository interface 与搜索条件；不得依赖 GORM、缓存、事务、proto 或 interfaces/infras。
- `infras/repository` 实现 domain interface，负责数据库、缓存、MQ producer 和第三方访问。
- `interfaces` 只做 HTTP/gRPC、MQ、任务等外部协议适配和调用编排；不得直接写 ORM、SQL、缓存或复杂业务规则。
- 数据库 model 必须嵌入 common 的 `BaseModel` 或 `SnowflakeModel`，使用项目实际 import 路径。
- 所有数据库访问经 `tx.GetTx(ctx, r.db)`，多步原子操作经 `tx.Transaction`，回调中传递回调 `ctx`。
- 常规 CRUD 使用 `db/query`；复杂 SQL 在 `db/mapper` 声明，再生成 `db/sqlquery`。禁止 `Raw`，禁止手改 `db/query`、`db/sqlquery` 或其他生成物。

详见 [templates/model.md](templates/model.md)、[templates/repository.md](templates/repository.md) 与 [templates/dto.md](templates/dto.md)。

## Optional 与缓存

唯一可用的 Optional 是 `<project>-common/pkg/optional`。其能力必须覆盖 SQL、JSON、MsgPack 及项目实际 API；不要引入其他 Optional 实现。非基本单值使用 `optional.Option[*T]`，未命中返回 `optional.None[*T]()`；集合返回非 nil 的 slice/map。

缓存使用 `jetcache.NewT[K, V](jetcache.WithLocal(...), jetcache.WithStatsDisabled(true))`、`local.NewTinyLFU` 和 `jetcache.WithStatsDisabled(true)`；实际 jetcache 版本的 Put/Delete 调用以已有代码复核。repository 使用 cache-aside，缓存 key 带模块前缀，实体缓存使用 `Option[*Entity]`。缓存失败只记录诊断日志，不覆盖数据库主流程。

详见 [templates/optional.md](templates/optional.md) 与 [templates/cache.md](templates/cache.md)。

## API、错误与鉴权

外部 API 和 error proto 只能在源 proto 仓库维护；按实际 buf v2 配置运行 `buf lint`、`buf generate`，并由 CI 同步 SDK；业务仓只能生成自身 `pkg/config` proto。业务错误只能使用已生成的 proto helper。

每个 RPC 显式声明 `required_permission`、`authenticated` 或 `public` 之一，并使用实际 authz import 的完整扩展名；HTTP、validate、field_behavior 复用既有 proto 依赖。冲突时优先级为 `required_permission` > `authenticated` > `public`。不规定任何 `auth.enabled` 开关语义。

common server 统一提供 HTTP `ResponseEncoder` 与 `ErrorEncoder`。详见 [templates/proto.md](templates/proto.md) 与 [templates/server-options.md](templates/server-options.md)。

## 日志与容器

业务日志使用 `log/slog`，初始化时通过 `NewKratosHandler` 接入 `klog.Logger`，并执行 `slog.SetDefault`；框架适配代码使用 `klog`。详见 [templates/logging.md](templates/logging.md)。

容器内配置目录唯一为 `/data/conf`。Docker `CMD`/`ENTRYPOINT`、Helm 与 Kubernetes 的 `-conf` 参数及容器内挂载目标必须指向该目录；Kubernetes `command`/`args` 必须按镜像 ENTRYPOINT 配置，不能重复调用二进制；主机路径、ConfigMap、Secret 或 PVC 等来源可不同。详见 [templates/container.md](templates/container.md)。

## 注释

每个函数写以函数名开头的 Go doc 注释。repository interface 与实现（除 `New`）保留 `param` / `return` 模板；省略 `ctx`、`error` 与空响应。

每个 `struct` 写以类型名开头的简短 Go doc，说明业务角色或技术职责。领域实体、值对象、DTO、配置、数据库 model、协议 request/response 与跨层查询结果等数据结构，每个字段均须在紧邻位置写以字段名开头的注释，表达业务含义、单位、可空性、所有权或约束，不得仅复述类型。并发调度 `struct` 的所有字段均须逐项说明依赖职责、状态或 timer 所有权、channel 的消息语义/缓冲/关闭方、`done` 完成条件，以及 mutex/once 的保护范围和幂等语义。普通私有依赖聚合 `struct` 不强制逐字段注释，除非字段意图、资源所有权或生命周期不显然。详见 [templates/comments.md](templates/comments.md)。

## 验证矩阵

- 仅 Go 代码：`gofmt`、相关包 `go test`、`go test ./...`（按仓库规模）。
- model、mapper 或生成配置：项目实际生成命令（通常是 Mage 中发现的命令）后检查生成物，再运行相关测试。
- Fx、启动、server 或日志：构建并以实际 `-conf` 启动，验证 lifecycle 与 HTTP 编码链。
- proto SDK：确认源 proto CI 完成并更新 SDK 后，编译业务仓；业务仓仅运行自身 config proto 生成。
- Dockerfile、Helm 或 Kubernetes：检查 `-conf` 和容器内目标路径均为 `/data/conf`，再运行项目已有镜像/manifest 校验。

未实际运行的校验必须明确写为未运行及原因。

## 模板

- [模块](templates/module.md)
- [Controller](templates/controller.md)
- [Repository](templates/repository.md)
- [Model](templates/model.md)
- [SQL mapper/DTO](templates/dto.md)
- [Cache](templates/cache.md)
- [Interfaces](templates/interfaces.md)
- [Proto/error proto](templates/proto.md)
- [Logging](templates/logging.md)
- [Optional 与非空返回](templates/optional.md)
- [Fx DI](templates/di.md)
- [启动](templates/startup.md)
- [容器](templates/container.md)
- [Server](templates/server-options.md)
- [注释](templates/comments.md)
