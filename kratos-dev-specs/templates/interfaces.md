# Interfaces 模板

`interfaces` 存放所有外部触发器入口，包括 HTTP/gRPC、MQ 消费者、定时任务触发器等。

## Web controller

放在 `interfaces/web`。

```go
package web

import (
    "context"
    "log/slog"

    "<project>/module/<module>/domain"
    v1 "<proto-go-module>/api/<scope>/v1"
)

type <Module>Controller struct {
    v1.Unimplemented<Module>ServiceServer
    svc *domain.<Module>Service
}

func New<Module>Controller(svc *domain.<Module>Service) *<Module>Controller {
    return &<Module>Controller{svc: svc}
}

func (c *<Module>Controller) Handle(ctx context.Context, req *v1.<Request>) (*v1.<Response>, error) {
    slog.InfoContext(ctx, "handle request", "module", "<module>")
    return &v1.<Response>{}, nil
}
```

需要多个 repository/service 调用保持原子性时，触发器可注入 `*gorm.DB` 并仅用于 `tx.Transaction` 事务边界。

```go
import (
    "context"

    "<project>/module/<module>/domain"
    "<project>/pkg/tx"
    v1 "<proto-go-module>/api/<scope>/v1"

    "gorm.io/gorm"
)

type <Module>Controller struct {
    db  *gorm.DB
    svc *domain.<Module>Service
}

func (c *<Module>Controller) HandleAtomic(ctx context.Context, req *v1.<Request>) (*v1.<Response>, error) {
    cmd := to<Module>Command(req)
    err := tx.Transaction(ctx, c.db, func(ctx context.Context) error {
        return c.svc.Handle(ctx, cmd)
    })
    if err != nil {
        return nil, err
    }
    return &v1.<Response>{}, nil
}
```

## MQ consumer

放在 `interfaces/mq`。consumer 只消费和适配消息；简单业务可调用 domain repository，复杂业务调用 domain service。

```go
package mq

import (
    "context"
    "log/slog"

    "<project>/module/<module>/domain"
)

type <Module>Consumer struct {
    svc *domain.<Module>Service
}

func New<Module>Consumer(svc *domain.<Module>Service) *<Module>Consumer {
    return &<Module>Consumer{svc: svc}
}

func (c *<Module>Consumer) Consume(ctx context.Context, payload []byte) error {
    slog.InfoContext(ctx, "consume message", "payload_size", len(payload))
    return nil
}
```

## Task job

放在 `interfaces/task`。job 是定时/后台任务触发器，仅协调 domain repository/service。

```go
package task

import (
    "context"
    "log/slog"

    "<project>/module/<module>/domain"
)

type <Module>Job struct {
    svc *domain.<Module>Service
}

func New<Module>Job(svc *domain.<Module>Service) *<Module>Job {
    return &<Module>Job{svc: svc}
}

func (j *<Module>Job) Run(ctx context.Context) error {
    slog.InfoContext(ctx, "run job", "module", "<module>")
    return nil
}
```

## 规则

- import 默认不为本模块包添加别名；只有引入多个同名包、多个模块 domain、公共包 domain，或存在 Go 语法冲突时，才使用 `<module>Domain`、`commonDomain` 等别名。
- interfaces 只适配外部输入，不直接访问数据库、不直接操作缓存、不生产 MQ 消息、不沉淀复杂业务规则。
- 简单业务允许 interfaces 直接调用 domain repository 完成，例如单表查询、简单保存、删除、存在性检查。
- 复杂业务规则应放在 domain service 中完成，例如跨 repository 协作、多步骤校验、状态流转、缓存/消息联动策略等；interfaces 再调用 domain service。
- 需要多个操作保持原子性时，`interfaces/web`、`interfaces/mq`、`interfaces/task` 可注入 `*gorm.DB` 并使用 `tx.Transaction` 包裹 service/repository 调用；事务内传递回调 ctx。
- `*gorm.DB` 只能用于事务边界，不在触发器内写 GORM/query/model/cache 逻辑。
- MQ consumer 放 `interfaces/mq`。
- MQ producer 放 `infras/repository`，由 domain 接口抽象后注入使用。
- 如需日志，使用 `log/slog`；触发器传入或创建了 ctx 时用 `slog.*Context(ctx, ...)`。
