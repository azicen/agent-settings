# Server 选项与编码器模板

`<project>-common/pkg/server` 已统一配置 HTTP `ResponseEncoder` 与 `ErrorEncoder`。业务模块通过 Fx Value Groups 注入服务器选项或中间件，**不得**重写全局 encoder，也不要把 encoder 函数作为自己的参数传递。

## Fx Value Groups 规范

Common server 接收以下 Fx Group 参数：
- `http-server-options`: `[]http.ServerOption`
- `http-middlewares`: `[]commonmiddleware.HTTPMiddleware`（带 Priority 权重并自动排序）
- `grpc-server-options`: `[]grpc.ServerOption`
- `grpc-middlewares`: `[]commonmiddleware.GRPCMiddleware`

### 1. HTTP Server Option 注入（以 CORS 跨域为例）

```go
package cmd

import (
	"net/http"

	khttp "github.com/go-kratos/kratos/v3/transport/http"
	"github.com/gorilla/handlers"
	"go.uber.org/fx"
)

type corsOptionsOut struct {
	fx.Out
	Option khttp.ServerOption `group:"http-server-options"`
}

// newCORSOptions 创建 CORS 跨域配置。
// 包装说明：gorilla/handlers.CORS 在不带 Origin 头的 OPTIONS 请求时会直接 return 空响应，
// 而非 CORS preflight 的 OPTIONS 请求需要透传给 Kratos 路由。
// 故采用包装：无 Origin 头的 OPTIONS → 直接透传；有 Origin 头的 OPTIONS → 走 CORS 处理
func newCORSOptions() corsOptionsOut {
	corsFilter := handlers.CORS(
		handlers.AllowedOrigins([]string{"*"}),
		handlers.AllowedMethods([]string{"GET", "POST", "PUT", "DELETE", "OPTIONS", "PATCH", "HEAD"}),
		handlers.AllowedHeaders([]string{"Content-Type", "Authorization", "X-Requested-With"}),
	)
	optionFilter := func(h http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			if r.Method == http.MethodOptions && r.Header.Get("Origin") == "" {
				h.ServeHTTP(w, r)
				return
			}
			corsFilter(h).ServeHTTP(w, r)
		})
	}
	return corsOptionsOut{
		Option: khttp.Filter(optionFilter),
	}
}
```

在 `BuildAppOptions` 中注册：`fx.Provide(newCORSOptions)`.

### 2. 中间件优先级排序

Common HTTP/gRPC server 内置中间件采用 Priority 稳定排序（升序执行）：
- `PriorityTracing` = 0（最先建立 Trace 上下文）
- `PriorityRecovery` = 10（捕获 Panic）
- `PriorityAuth` = 50（业务认证鉴权）
- `PriorityValidate` = 90（请求参数协议校验）

## 统一响应与错误编码机制

Common server 统一规范了 API 响应结构：

```go
type baseResponse struct {
	Code    int             `json:"code"`
	Data    json.RawMessage `json:"data"`
	Msg     string          `json:"msg"`
	Success bool            `json:"success"`
	Reason  string          `json:"reason,omitempty"`
}
```

1. **统一成功响应 (`responseEncoder`)**：
   - 采用 `json-full` 编解码器（配置 `EmitUnpopulated: true` 保留零值字段，Protobuf `Timestamp` 自动格式化为本地时区时间字符串 `"2006-01-02 15:04:05"`）。
   - 将业务数据序列化后包裹进 `baseResponse{Code: 200, Msg: "操作成功", Success: true}`。
2. **统一错误响应 (`errorEncoder`)**：
   - 提取错误 Reason：通过 `errors.AsType[*kratoserrors.Error](err)` 提取业务错误码。
   - 提取校验原因：通过 `errors.AsType[validationCause](err)` 提取参数验证失败的具体字段（`Field()`）与规则（`Reason()`）。
   - 写入统一失败包体并根据错误原因设置 HTTP 状态码。

## 规则

- 模块输出选项时，`group` 标签必须与 common server constructor 的 `fx.In` 严格一致；不要把 `khttp.ServerOption` 错放到 `http-middlewares` group。
- gRPC 扩展同理：以相同的 `fx.Out` 模式提供 `grpc.ServerOption` 到 `grpc-server-options`。
- service 的 gRPC/HTTP 注册统一在各业务模块的 `di.go` 的 `fx.Invoke` 中完成。
