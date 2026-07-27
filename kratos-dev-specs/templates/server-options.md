# Server 选项模板

common server constructor 已统一安装 HTTP `ResponseEncoder` 与 `ErrorEncoder`。业务模块只能提供它允许注入的 server option、middleware 或 service 注册，**不得**重写 encoder，也不要把 encoder 函数作为自己的参数再传回去。

当 common HTTP constructor 以 Fx group 收集选项时，模块按下例输出。`group` 名称、option 类型和实际可用 helper 必须以 common constructor 的参数 tag 为准。

```go
package web

import (
	khttp "github.com/go-kratos/kratos/v2/transport/http"
	"go.uber.org/fx"
)

type httpServerOptions struct {
	fx.Out
	Option khttp.ServerOption `group:"http-server-options"`
}

// ProvideHTTPServerOptions 输出模块的 HTTP server option。
func ProvideHTTPServerOptions() httpServerOptions {
	return httpServerOptions{
		Option: khttp.Filter(corsFilter),
	}
}
```

`corsFilter` 是项目已经实现或经项目依赖确认的 HTTP filter；模板不假设第三方 CORS 包或 common server 的额外 API。通常由 `fx.Provide(ProvideHTTPServerOptions)` 注册该 provider。

- common constructor 若定义 `http-middlewares`，模块输出对应 middleware 类型和该 tag；不要把 `khttp.ServerOption` 错放到 middleware group。
- gRPC 扩展同理：只有 common constructor 实际声明了 `grpc-server-options`（或其他 tag）时，才以相同的 `fx.Out` 模式提供 `grpc.ServerOption`。
- service 的 gRPC/HTTP 注册仍在模块的 `fx.Invoke` 中完成；server 创建、地址、encoder 和全局拦截器归 common composition root 管理。
- 修改前读取 common server constructor 的 `fx.In` 字段及其 `group` tag。没有该入口时报告缺失，不要凭名称发明 API。
