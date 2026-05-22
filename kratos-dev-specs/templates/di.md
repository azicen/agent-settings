# DI 模板

模块根目录的 `di.go` 统一注册依赖、缓存、外部触发器和服务端接口。

```go
package <module>

import (
    "<project>/module/<module>/domain"
    "<project>/module/<module>/infras/repository"
    "<project>/module/<module>/infras/repository/cache"
    mq "<project>/module/<module>/interfaces/mq"
    task "<project>/module/<module>/interfaces/task"
    web "<project>/module/<module>/interfaces/web"
    v1 "<proto-go-module>/api/<scope>/v1"

    "github.com/go-kratos/kratos/v2/transport/grpc"
    "github.com/go-kratos/kratos/v2/transport/http"
    "go.uber.org/dig"
)

func Register(container *dig.Container) error {
    err := container.Provide(cache.New<Module>Cache)
    if err != nil {
        return err
    }

    err = container.Provide(domain.New<Module>Factory)
    if err != nil {
        return err
    }

    err = container.Provide(
        repository.New<Module>Repository,
        dig.As(new(domain.<Module>Repository)),
    )
    if err != nil {
        return err
    }

    err = container.Provide(domain.New<Module>Service)
    if err != nil {
        return err
    }

    err = container.Provide(web.New<Module>Controller)
    if err != nil {
        return err
    }

    err = container.Provide(mq.New<Module>Consumer)
    if err != nil {
        return err
    }

    err = container.Provide(task.New<Module>Job)
    if err != nil {
        return err
    }

    return container.Invoke(func(
        gs *grpc.Server,
        hs *http.Server,
        ctrl *web.<Module>Controller,
    ) {
        v1.Register<Module>ServiceServer(gs, ctrl)
        v1.Register<Module>ServiceHTTPServer(hs, ctrl)
    })
}
```

## 规则

- 注册顺序通常为 cache -> factory -> repository -> service -> controller/consumer/job -> gRPC/HTTP register。
- repository 实现用 `dig.As(new(domain.<Module>Repository))` 绑定到 domain interface。
- cache 先 provide，再注入 repository。
- controller 注册 gRPC 和 HTTP service。
- consumer/job 是否 provide 取决于模块是否需要对应外部触发器。
