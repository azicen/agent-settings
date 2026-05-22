# Controller 模板

Controller 位于 `module/<module>/interfaces/web`，负责 HTTP/gRPC/proto service 适配。

```go
package web

import (
    "context"
    "log/slog"

    "<project>/module/<module>/domain"
    "<project>/pkg/tx"
    v1 "<proto-go-module>/api/<scope>/v1"

    "google.golang.org/protobuf/types/known/emptypb"
    "gorm.io/gorm"
)

// <Module>Controller <模块>控制器
type <Module>Controller struct {
    v1.Unimplemented<Module>ServiceServer
    db   *gorm.DB
    repo domain.<Module>Repository
    svc  *domain.<Module>Service
}

// New<Module>Controller 创建<模块>控制器
func New<Module>Controller(
    db *gorm.DB,
    repo domain.<Module>Repository,
    svc *domain.<Module>Service,
) *<Module>Controller {
    return &<Module>Controller{db: db, repo: repo, svc: svc}
}

// Get<Module> 获取<模块>详情
func (c *<Module>Controller) Get<Module>(ctx context.Context, req *v1.Get<Module>Request) (*v1.<Module>Info, error) {
    entity, err := c.repo.FindByID(ctx, req.GetId())
    if err != nil {
        slog.ErrorContext(ctx, "get module failed", "id", req.GetId(), "error", err)
        return nil, err
    }
    if entity.IsNone() {
        return nil, v1.Error<Module>NotFound("资源不存在")
    }
    return to<Module>Info(entity.Unwrap()), nil
}

// Delete<Module> 删除<模块>
func (c *<Module>Controller) Delete<Module>(ctx context.Context, req *v1.Delete<Module>Request) (*emptypb.Empty, error) {
    err := tx.Transaction(ctx, c.db, func(ctx context.Context) error {
        entity, err := c.repo.FindByID(ctx, req.GetId())
        if err != nil {
            return err
        }
        if entity.IsNone() {
            return v1.Error<Module>NotFound("资源不存在")
        }
        return c.repo.Delete(ctx, entity.Unwrap())
    })
    if err != nil {
        return nil, err
    }
    return &emptypb.Empty{}, nil
}
```

## 规则

- import 默认不为本模块包添加别名；只有引入多个同名包、多个模块 domain、公共包 domain，或存在 Go 语法冲突时，才使用 `<module>Domain`、`commonDomain` 等别名。
- 嵌入 `v1.Unimplemented<Service>Server`。
- 只做协议适配、参数转换、调用 domain service/repository。
- 简单业务允许直接调用 domain repository 完成；复杂业务应放在 domain service 中完成后由 controller 调用。
- 需要多个 repository/service 调用保持原子性时，可注入 `*gorm.DB` 并使用 `tx.Transaction` 包裹调用；事务内必须传递回调 ctx。
- `*gorm.DB` 只能用于事务边界，不直接访问 GORM/query/model/cache。
- 查询实体不存在时使用 error-proto 生成的 `v1.ErrorXxx(...)`。
- 如果 proto service/message/error helper 不存在，按 proto/error-proto 流程处理，不在业务项目中生成 proto。
- 如需日志，使用 `log/slog`；有 ctx 时用 `slog.*Context(ctx, ...)`。
- 每个函数至少要有符合 Go 规范的函数注释，注释以函数名开头。
