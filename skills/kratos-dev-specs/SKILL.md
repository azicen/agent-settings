---
name: kratos-dev-specs
description: "用于 Go Kratos 项目的需求开发、模块新增、领域建模、Repository、HTTP/gRPC、Proto、Fx 依赖注入、启动配置、日志、容器、重构、排障与代码审查。只要确认当前项目使用 Kratos，就必须加载本技能并按任务路由读取对应规范。"
---

# Golang Kratos 项目规范

<EXTREMELY-IMPORTANT>
只要当前项目使用 Kratos，在编写代码、新增模块、接口开发、重构、审查或问题排查时，必须使用本技能。

本文件是任务路由器，不是完整规范。根据当前任务读取下方指定的 `references/*.md`；不要默认加载全部参考文件。
</EXTREMELY-IMPORTANT>

## 执行流程

1. 确认项目确实使用 Kratos，并读取当前任务涉及的现有实现。
2. 检查 `go.mod`、相关 imports 和项目生成脚本，确认实际模块路径、依赖版本与既有架构。
3. 根据“任务路由”只读取本次任务所需的参考文件；复合任务合并对应文件并去重。
4. 实施前确认依赖的 common 能力真实存在；缺失时报告位置和影响，不复制基础实现或引入替代框架。
5. 修改后按“验证路由”执行检查，并明确报告未运行的验证及原因。

## 项目事实与占位符

规范示例中的占位符必须替换为仓库中发现的实际值：

- `<project>`：业务 Go module。
- `<project>-common`：公共库 Go module。
- `<project>-proto-go`：已发布的 Proto SDK Go module。
- `<registry>`：镜像仓库地址。

不得根据仓库名、目录结构或经验猜测依赖。若项目事实与参考示例不同，遵循项目已验证的 API 和版本；若项目事实违反下方架构红线，不得静默照搬，应先指出冲突。

## 全局架构红线

- `domain` 只包含实体、值对象、领域服务、repository interface 和搜索条件，不依赖 GORM、缓存、事务、Proto、`interfaces` 或 `infras`。
- `interfaces` 只负责 HTTP/gRPC、MQ、任务及自定义协议的适配与调用编排，不直接访问 ORM、SQL 或缓存，不承载复杂业务规则。
- 单表类型安全 CRUD 使用项目既有 `gorm.io/gen` 生成链路；复杂 SQL 使用项目既有 `gorm.io/cli/gorm` Mapper 生成链路。禁止拼接裸 SQL、直接调用 `db.Raw` 或手改生成物。
- 数据库访问必须复用上下文事务；事务回调内必须继续传递回调提供的 `context.Context`。
- 缺失值只使用 `<project>-common/pkg/optional` 表达，不引入第三方 Optional；集合返回值必须为非 nil slice/map。
- 业务错误使用已生成的 Proto helper，不在业务代码中临时构造不一致的错误协议。
- 业务日志统一使用 `log/slog`；存在 `context.Context` 时使用 `slog.*Context`，变量通过结构化属性传递，不拼接进消息。
- 外部资源、后台任务和服务进程统一由 Fx/Kratos 生命周期管理，不创建脱离统一启停流程的长期 goroutine 或资源。
- 容器内配置目录固定为 `/data/conf`，Docker、Helm、Kubernetes 与启动参数必须保持一致。
- 缺少 common、Proto SDK、生成器或 Fx 能力时，报告缺失项与发现位置；不得复制公共实现、替换 DI 容器或绕过生成链路。

## 任务路由

| 当前任务 | 必读参考 | 按需补充 |
|---|---|---|
| 新建或重构业务模块 | [module.md](references/module.md)、[di.md](references/di.md) | 涉及协议适配时读取 [interfaces.md](references/interfaces.md) |
| Domain 实体、工厂或领域服务 | [module.md](references/module.md)、[optional.md](references/optional.md) | 涉及仓储契约时读取 [repository.md](references/repository.md) |
| HTTP/gRPC Controller | [controller.md](references/controller.md)、[proto.md](references/proto.md) | 涉及统一编码或中间件时读取 [server-options.md](references/server-options.md) |
| Repository 实现 | [repository.md](references/repository.md)、[optional.md](references/optional.md) | 涉及缓存时读取 [cache.md](references/cache.md) |
| 数据库 Model 或单表 CRUD | [model.md](references/model.md)、[repository.md](references/repository.md) | 涉及复杂查询时读取 [dto.md](references/dto.md) |
| 复杂 SQL、报表或跨表 DTO | [dto.md](references/dto.md)、[repository.md](references/repository.md) | 涉及 Model 时读取 [model.md](references/model.md) |
| 缓存接入或缓存 Review | [cache.md](references/cache.md)、[optional.md](references/optional.md) | 同时读取对应 [repository.md](references/repository.md) |
| MQ、Job、批处理或自定义协议 | [interfaces.md](references/interfaces.md)、[logging.md](references/logging.md) | 涉及生命周期或注入时读取 [di.md](references/di.md) |
| Fx 依赖注入 | [di.md](references/di.md) | 涉及服务选项时读取 [server-options.md](references/server-options.md) |
| 启动、配置、迁移或生命周期 | [startup.md](references/startup.md)、[logging.md](references/logging.md) | 涉及 HTTP/gRPC 组装时读取 [server-options.md](references/server-options.md) |
| 中间件、Server Options、响应或错误编码 | [server-options.md](references/server-options.md) | 涉及注入时读取 [di.md](references/di.md) |
| Proto、鉴权注解或业务错误 | [proto.md](references/proto.md) | 涉及 Controller 时读取 [controller.md](references/controller.md) |
| 日志初始化或日志改造 | [logging.md](references/logging.md) | 涉及启动时读取 [startup.md](references/startup.md) |
| Docker、Helm 或 Kubernetes | [container.md](references/container.md)、[startup.md](references/startup.md) | 无 |
| 注释补全 | [comments.md](references/comments.md) | 同时读取被修改代码所属专题 |
| 排障 | 先按故障所在层读取对应专题 | 涉及启动、日志或生命周期时读取相应文件 |
| 代码审查 | [comments.md](references/comments.md)，并按变更类型读取对应专题 | 只审查实际变更涉及的规则，不默认加载全部参考 |

## 验证路由

- 普通 Go 代码：执行 `gofmt`、相关包测试，并按仓库规模决定是否运行 `go test ./...`。
- Model、Mapper 或生成配置：先运行仓库中实际存在的生成命令，检查生成物后再运行相关测试。
- Fx、启动、Server 或日志：构建服务，并使用项目实际 `-conf` 参数验证启动、异常退出和优雅停止。
- Proto SDK：在源 Proto 仓库完成 lint、生成和 SDK 更新；业务仓只执行自身配置 Proto 的既有生成流程。
- Docker、Helm 或 Kubernetes：核对镜像入口、`-conf` 与 `/data/conf`，再运行仓库已有镜像或 manifest 校验。
- 未实际运行的验证必须明确说明原因，不得描述为已通过。

## 完成报告

最终结果必须简要说明：

1. 本次按任务读取了哪些参考文件。
2. 发现并采用了哪些项目实际模块、版本、生成命令或既有模式。
3. 修改内容及其对应的架构约束。
4. 已执行的格式化、生成、构建和测试。
5. 未执行的验证、原因及剩余风险。
