# 启动模板

启动层是组合根：负责命令行解析（Cobra）、配置加载与分层合并、初始化 Slog、执行数据库迁移（Goose）、组合 Fx 依赖图，并将 Kratos app 接入唯一 lifecycle bridge。

在 Kratos v3 中，框架原生集成标准库 `log/slog`，配置采用 `config.toml` + `conf.d/` 分层覆盖，服务生命周期纳管 HTTP、gRPC 及自定义协议 Server。

## 启动入口示例（`cmd/main.go` & `cmd/server.go`）

```go
package main

import (
	"context"
	"log/slog"
	"os"
	"syscall"
	"time"

	commonconfig "<project>-common/pkg/config"
	commonlogging "<project>-common/pkg/logging"
	commonmigration "<project>-common/migration"
	gormmodule "<project>-common/pkg/gorm"
	" <project>-common/pkg/server"
	projectconfig "<project>/pkg/config"
	projectmigration "<project>/migration"
	v1conf "<project>-proto-go/config"

	"github.com/go-kratos/kratos/v3"
	"github.com/go-kratos/kratos/v3/config"
	klog "github.com/go-kratos/kratos/v3/log"
	"github.com/go-kratos/kratos/v3/transport/grpc"
	"github.com/go-kratos/kratos/v3/transport/http"
	"github.com/go-kratos/kratos/contrib/otel/v3/tracing"
	"github.com/spf13/cobra"
	"go.uber.org/fx"
	"go.uber.org/fx/fxevent"

	_ "github.com/azicen/kratos-extension/encoding/json"
	_ "github.com/azicen/kratos-extension/encoding/toml"
	_ "go.uber.org/automaxprocs"
)

var (
	Name    = "<project>"
	Version = "unknown"
	id, _   = os.Hostname()
)

// newLogger 构建服务通用的结构化 Slog 日志器，集成 OpenTelemetry 追踪属性
func newLogger() *slog.Logger {
	colorHandler := commonlogging.NewColorHandler(os.Stdout,
		commonlogging.WithLevel(slog.LevelDebug),
	)
	return klog.NewLogger(colorHandler,
		klog.WithExtractor(tracing.TraceAttrs),
	).With(
		"service.id", id,
		"service.name", Name,
		"service.version", Version,
	)
}

// loadConfig 读取并分层合并配置文件（config.toml + conf.d/*.toml）
func loadConfig(confPath string) (*v1conf.Bootstrap, *projectconfig.Bootstrap, error) {
	sources, err := commonconfig.BuildSources(confPath)
	if err != nil {
		return nil, nil, err
	}
	c := config.New(config.WithSource(sources...))
	defer c.Close()
	err = c.Load()
	if err != nil {
		return nil, nil, err
	}

	var commonBc v1conf.Bootstrap
	err = c.Scan(&commonBc)
	if err != nil {
		return nil, nil, err
	}
	var projectBc projectconfig.Bootstrap
	err = c.Scan(&projectBc)
	if err != nil {
		return nil, nil, err
	}
	return &commonBc, &projectBc, nil
}

func main() {
	var confPath string
	rootCmd := &cobra.Command{
		Use:   Name,
		Short: "<project> 服务",
		RunE: func(_ *cobra.Command, _ []string) error {
			runServer(confPath)
			return nil
		},
	}
	rootCmd.PersistentFlags().StringVar(&confPath, "conf", "./config", "配置目录路径")
	// 如需批处理子命令，通过 rootCmd.AddCommand 添加
	err := rootCmd.Execute()
	if err != nil {
		slog.Error("启动失败", "error", err)
		os.Exit(1)
	}
}

// runServer 执行前置数据库迁移并启动 Fx 容器。
func runServer(confPath string) {
	logger := newLogger()
	slog.SetDefault(logger)

	commonBc, projectBc, err := loadConfig(confPath)
	if err != nil {
		panic(err)
	}

	// 注册数据库驱动的 Slog 日志适配
	err = gormmodule.RegisterMySQLSlogLogger(logger, commonBc.GetData().GetDatabase().GetDriver())
	if err != nil {
		panic(err)
	}

	// 启动前同步执行数据库版本迁移对齐
	err = commonmigration.Up(context.Background(), commonBc)
	if err != nil {
		panic(err)
	}
	err = projectmigration.Up(context.Background(), commonBc)
	if err != nil {
		panic(err)
	}

	fx.New(
		BuildAppOptions(commonBc, projectBc, logger),
		fx.WithLogger(func() fxevent.Logger {
			return &fxevent.SlogLogger{Logger: logger}
		}),
	).Run()
}

// newApp 构建 Kratos 统一应用实例，纳管 HTTP、gRPC 及自定义协议服务
func newApp(
	logger *slog.Logger,
	gs *grpc.Server,
	hs *http.Server,
	// 如有自定义协议服务器（如 ModbusServer），实现 transport.Server 接口后一起注入
) *kratos.App {
	return kratos.New(
		kratos.ID(id),
		kratos.Name(Name),
		kratos.Version(Version),
		kratos.Logger(logger),
		kratos.Signal(syscall.SIGQUIT),
		kratos.StopTimeout(10*time.Second),
		// kratos.BeforeStart(preheatService.Start),
		// kratos.AfterStart(scheduler.Start),
		// kratos.BeforeStop(scheduler.Stop),
		kratos.Server(gs, hs), // 自定义 transport.Server 平级传入
	)
}

// BuildAppOptions 仅组合本进程的 provider、模块和唯一 Kratos bridge
func BuildAppOptions(commonBc *v1conf.Bootstrap, projectBc *projectconfig.Bootstrap, logger *slog.Logger) fx.Option {
	return fx.Options(
		fx.Supply(commonBc, projectBc, logger),
		gormmodule.Register(),
		fx.Provide(
			server.NewGRPCServer,
			server.NewHTTPServer,
			newApp,
		),
		<module>.Register(),
		fx.Invoke(registerKratosLifecycle),
	)
}

// registerKratosLifecycle 将 Kratos 的 Run/Stop 纳入 Fx 生命周期
func registerKratosLifecycle(lc fx.Lifecycle, app *kratos.App, shutdowner fx.Shutdowner) {
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

## 生命周期规则

1. `app.Run()` 会阻塞，bridge 必须在 `OnStart` 中以独立 goroutine 运行，并在 `OnStop` 中执行 `app.Stop()` 后等待退出。注入 `fx.Shutdowner`，使 app 非预期退出可以主动结束 Fx 进程；模块不得自行调用 `app.Run()` 或 `app.Stop()`。
2. 外部 client、consumer、poller 或长期运行总线必须以 `fx.Lifecycle` 明确资源边界：构造器只创建对象，`OnStart` 建立连接/订阅，`OnStop` 关闭或取消并等待工作协程退出。
3. 自定义协议 Server（如物联网 Modbus TCP 轮询器、长连接 Server 等）实现 `transport.Server` 接口（`Start(context.Context) error` 与 `Stop(context.Context) error`），直接传给 `kratos.Server(...)` 统一纳管。

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
