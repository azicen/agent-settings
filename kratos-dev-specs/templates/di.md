# Fx DI 模板

模块根目录的 `di.go` 只声明依赖图，导出 `Register() fx.Option`。资源创建、连接和停止由 provider 的 `fx.Lifecycle` hook 管理；HTTP/gRPC service 注册使用 `fx.Invoke`。模块不得调用 `app.Run()` 或 `app.Stop()`，Kratos 生命周期只由启动层唯一 bridge 管理。

```go
package <module>

import (
	"context"

	"<project>/module/<module>/domain"
	"<project>/module/<module>/infras/repository"
	"<project>/module/<module>/infras/repository/cache"
	web "<project>/module/<module>/interfaces/web"
	v1 "<project>-proto-go/api/<scope>/v1"

	"github.com/go-kratos/kratos/v2/transport/grpc"
	"github.com/go-kratos/kratos/v2/transport/http"
	"go.uber.org/fx"
)

// Register 注册 <module> 模块依赖与服务
func Register() fx.Option {
	return fx.Module("<module>",
		fx.Provide(
			cache.New<Module>Cache,
			domain.New<Module>Factory,
			fx.Annotate(
				repository.New<Module>Repository,
				fx.As(new(domain.<Module>Repository)),
			),
			domain.New<Module>Service,
			web.New<Module>Controller,
		),
		fx.Invoke(func(gs *grpc.Server, hs *http.Server, ctrl *web.<Module>Controller) {
			v1.Register<Module>ServiceServer(gs, ctrl)
			v1.Register<Module>ServiceHTTPServer(hs, ctrl)
		}),
	)
}
```

外部 client、consumer、poller 或长期运行总线的 provider 必须明确生命周期。构造阶段不启动后台工作；`OnStart` 连接/订阅，`OnStop` 取消、关闭并等待退出。把调用方传入的 `ctx` 继续传给 I/O，不能另造无取消的 context。

```go
func NewConsumer(lc fx.Lifecycle, client *Client) *Consumer {
	consumer := &Consumer{client: client}
	lc.Append(fx.Hook{
		OnStart: func(ctx context.Context) error {
			return consumer.Start(ctx)
		},
		OnStop: func(ctx context.Context) error {
			return consumer.Stop(ctx)
		},
	})
	return consumer
}
```

- 使用 `fx.Options` 组合模块；使用 `fx.Annotate` + `fx.As` 将实现绑定到 consumer-side domain interface。
- 只有 common constructor 的 `fx.In` 实际声明 group 时才用 `fx.Out` 输出分组结果；HTTP/gRPC option tag 见 [server-options.md](server-options.md)。
- provider 的返回错误应保留创建资源的上下文；不要在构造器外创建全局 DB、client 或 goroutine。
- provider 依赖顺序由 Fx 图决定，不依赖 `fx.Provide` 的书写顺序。
