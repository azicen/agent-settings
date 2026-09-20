# Controller 模板

Controller 位于 `module/<module>/interfaces/web`，仅做 proto 请求校验/转换和 domain 调用。HTTP 成功与失败编码由 common server 的统一 `ResponseEncoder` / `ErrorEncoder` 完成；业务错误只能使用 proto SDK 生成的 helper 函数。

```go
package web

import (
	"context"
	"log/slog"

	"<project>-common/pkg/tx"
	"<project>/module/<module>/domain"
	v1 "<project>-proto-go/api/<scope>/v1"

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
		slog.ErrorContext(ctx, "获取 <module> 失败", "id", req.GetId(), "error", err)
		return nil, err
	}
	if entity.IsNone() {
		return nil, v1.Error<Module>NotFound("资源不存在")
	}
	return to<Module>Info(entity.Unwrap()), nil
}

// Delete<Module> 删除<模块>
func (c *<Module>Controller) Delete<Module>(ctx context.Context, req *v1.Delete<Module>Request) (*emptypb.Empty, error) {
	err := tx.Transaction(ctx, c.db, func(txCtx context.Context) error {
		entity, err := c.repo.FindByID(txCtx, req.GetId())
		if err != nil {
			return err
		}
		if entity.IsNone() {
			return v1.Error<Module>NotFound("资源不存在")
		}
		return c.repo.Delete(txCtx, entity.Unwrap())
	})
	if err != nil {
		return nil, err
	}
	return &emptypb.Empty{}, nil
}
```

## 规则

- **继承 Server 接口**：必须嵌入 `v1.Unimplemented<Module>ServiceServer`，同时兼容 gRPC 与 HTTP。
- **单一职责**：只做协议适配、参数转换、调用 domain service/repository和 validate proto 未完成的参数校验；不得直接写 ORM、GORM/gen query、SQL 或缓存逻辑。
- **事务边界控制**：需要多个 repository/service 调用保持原子性时，可注入 `*gorm.DB`，但**只能**用于调用 `tx.Transaction(ctx, c.db, func(txCtx context.Context) error { ... })` 建立事务边界；事务内所有后续调用必须传递回调 `txCtx`。
- **规范业务错误**：业务异常统一调用 error-proto 生成的 `v1.ErrorXxx(...)` 返回。
- **日志规范**：Controller 日志使用 `log/slog`，统一采用 `slog.*Context(ctx, ...)` 并传入结构化键值对；禁止在静态 message 中拼接 ID 或错误文本。
- **函数注释**：每个方法必须遵循 Go doc 规范，以方法名开头，清晰注明参数与返回值。
