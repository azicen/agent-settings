# 启动模板

启动层是组合根：只在这里解析配置、执行 migration、组合 Fx，并把 Kratos app 接入唯一 lifecycle bridge。下面的 import 路径、`Bootstrap` 字段、`BuildAppOptions` 内容和项目 migration 名称均为必须按当前项目替换的占位符；调用顺序和所列 API 是 common 当前模式。

```go
package main

import (
	"context"
	"flag"
	"log/slog"
	"os"

	commonconfig "<project>-common/pkg/config"
	commonmigration "<project>-common/pkg/migration"
	v1conf "<project>-proto-go/config"
	projectconfig "<project>/pkg/config"
	projectmigration "<project>/pkg/migration"

	"github.com/go-kratos/kratos/v2"
	kconfig "github.com/go-kratos/kratos/v2/config"
	klog "github.com/go-kratos/kratos/v2/log"
	"go.uber.org/fx"
	"go.uber.org/fx/fxevent"

	elog "<project>-common/pkg/log" // 按实际 NewKratosHandler 所在包替换。
)

func main() {
	confDir := flag.String("conf", "./config", "configuration directory")
	flag.Parse()

	logger := klog.NewStdLogger(os.Stdout) // 按项目实际 logger provider 补充服务字段。
	handler := elog.NewKratosHandler(logger)
	slog.SetDefault(slog.New(handler))

	sources, err := commonconfig.BuildSources(*confDir)
	if err != nil {
		panic(err)
	}
	c := kconfig.New(kconfig.WithSource(sources...))
	defer c.Close()
	if err := c.Load(); err != nil {
		panic(err)
	}

	var commonBc v1conf.Bootstrap
	if err := c.Scan(&commonBc); err != nil {
		panic(err)
	}
	var projectBc projectconfig.Bootstrap
	if err := c.Scan(&projectBc); err != nil {
		panic(err)
	}

	if err := commonmigration.Up(context.Background(), &commonBc); err != nil {
		panic(err)
	}
	if err := projectmigration.Up(context.Background(), &commonBc); err != nil {
		panic(err)
	}

	fx.New(
		BuildAppOptions(&commonBc, &projectBc, logger),
		fx.WithLogger(func() fxevent.Logger {
			return &fxevent.SlogLogger{Logger: slog.Default()}
		}),
	).Run()
}

// BuildAppOptions 仅组合本进程的 provider、模块和唯一 Kratos bridge。
func BuildAppOptions(commonBc *v1conf.Bootstrap, projectBc *projectconfig.Bootstrap, logger klog.Logger) fx.Option {
	return fx.Options(
		fx.Supply(commonBc, projectBc, logger),
		fx.Provide(NewKratosApp), // 项目实际 app 构造器。
		fx.Invoke(newKratosFXBridge),
		<module>.Register(),
	)
}

// newKratosFXBridge 将 Kratos 的 Run/Stop 纳入 Fx 生命周期。
func newKratosFXBridge(lc fx.Lifecycle, app *kratos.App, shutdowner fx.Shutdowner) {
	done := make(chan struct{})
	lc.Append(fx.Hook{
		OnStart: func(context.Context) error {
			go func() {
				defer close(done)
				if err := app.Run(); err != nil {
					_ = shutdowner.Shutdown(fx.ExitCode(1))
				}
			}()
			return nil
		},
		OnStop: func(ctx context.Context) error {
			if err := app.Stop(); err != nil {
				return err
			}
			select {
			case <-done:
				return nil
			case <-ctx.Done():
				return ctx.Err()
			}
		},
	})
}
```

`app.Run()` 通常会阻塞，bridge 必须在 `OnStart` 中以 goroutine 运行，并在 `OnStop` 中 `Stop` 后等待退出。注入 `fx.Shutdowner`，使 app 非预期退出可以结束 Fx；不要由模块直接 `Run`、`Stop` 或重复注册 bridge。

外部客户端和长期运行总线也必须以 `fx.Lifecycle` 明确资源边界：构造器只创建对象，`OnStart` 连接或订阅，`OnStop` 关闭或取消并等待工作协程退出。

```go
func NewEventBus(lc fx.Lifecycle, client *Client) *EventBus {
	bus := NewBus(client)
	lc.Append(fx.Hook{
		OnStart: func(ctx context.Context) error { return bus.Start(ctx) },
		OnStop:  func(ctx context.Context) error { return bus.Stop(ctx) },
	})
	return bus
}
```

- 默认 `-conf` 为 `./config`；镜像运行参数由 [container.md](container.md) 固定为 `/data/conf`。
- 不使用伪造的 `config.Load`、`commonbootstrap.Scan` 或 `migration.Run`；扫描通过 `c.Scan`，migration 使用项目已发现的 `Up` API。
- 默认示例中 `projectmigration.Up` 接收 `&commonBc`；`projectBc` 用于 Fx 业务配置。若项目的 `Up` 签名不同，必须按实际函数签名替换，不要假定项目 `Bootstrap` 就是 migration 入参。
- migration 在 Fx 构建前完成；配置 `Close` 由启动层 `defer` 负责。
- `BuildAppOptions` 只能作为组合根，不把 `Bootstrap` 当作 `fx.Option` 直接塞入 `fx.New`。
