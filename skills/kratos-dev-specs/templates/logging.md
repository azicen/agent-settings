# 日志模板

业务代码使用标准库 `log/slog`；启动链路以 `elog.NewKratosHandler(klog.Logger)` 把 `slog` 接入 Kratos logger，随后调用 `slog.SetDefault`。Kratos app、server 构造和日志适配等框架层可以使用 `klog`，业务 domain/infras/interfaces 不直接使用它。

```go
package bootstrap

import (
	"log/slog"

	elog "<project>-common/pkg/log"
	klog "github.com/go-kratos/kratos/v2/log"
)

// NewLogger 初始化默认 slog 与 Kratos 日志适配。
func NewLogger(logger klog.Logger) *slog.Logger {
	handler := elog.NewKratosHandler(logger)
	defaultLogger := slog.New(handler)
	slog.SetDefault(defaultLogger)
	return defaultLogger
}
```

```go
slog.ErrorContext(ctx,
	"save equipment failed",
	"equipment_id", equipmentID,
	"error", err,
)
```

- 有 `context.Context` 时使用 `slog.DebugContext`、`InfoContext`、`WarnContext` 或 `ErrorContext`；没有 ctx 才使用非 Context 版本。
- message 描述事件，变量使用结构化 key/value；不要把 ID、错误或 payload 拼接进 message。
- `Debug` 用于缓存等可恢复诊断，`Info` 用于重要状态变化，`Warn` 用于可继续的异常，`Error` 用于操作失败。循环内避免每条数据记录 Info/Error。
- 不记录密码、token、Authorization、Cookie、完整身份信息或未经脱敏的请求/消息内容；错误日志应包含能定位资源的非敏感 ID 和原始 `error`。
