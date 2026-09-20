# 日志模板

日志系统全面统一至 Go 标准库 `log/slog`。Kratos 框架日志原生对接 `slog`，并通过 `klog.WithExtractor(tracing.TraceAttrs)` 自动将 OpenTelemetry 的 `trace_id` 和 `span_id` 注入到日志属性中。业务代码（domain/infras/interfaces）统一使用 `log/slog`，禁止使用 `fmt.Println`、标准 `log` 或旧式 `log.Helper`。

## 初始化与追踪集成

cmd/main.go:
```go
package main

import (
	"log/slog"
	"os"

	logging "<project>-common/pkg/logging"
	gorm "<project>-common/pkg/gorm"

	"github.com/go-kratos/kratos/contrib/otel/v3/tracing"
	klog "github.com/go-kratos/kratos/v3/log"
)

// newLogger 构建服务通用的结构化日志器
func newLogger() *slog.Logger {
	colorHandler := logging.NewColorHandler(os.Stdout,
		logging.WithLevel(slog.LevelDebug),
	)
	return klog.NewLogger(colorHandler,
		klog.WithExtractor(tracing.TraceAttrs),
	).With(
		"service.id", id,
		"service.name", Name,
		"service.version", Version,
	)
}

func runServer(confPath string) {
	logger := newLogger()
	slog.SetDefault(logger)

	// 注册数据库驱动与连接池的 Slog 日志适配
	err := gorm.RegisterMySQLSlogLogger(logger, driver)
	if err != nil {
		panic(err)
	}
	// ...
}
```

## 业务日志调用示例

```go
// 成功业务关键状态流转
slog.InfoContext(ctx, "电表数据转换完成", "processed", processed, "batch_size", batchSize)

// 失败与错误记录
slog.ErrorContext(ctx, "保存设备失败",
	"equipment_id", equipmentID,
	"error", err,
)

// 缓存或可恢复异常诊断
slog.DebugContext(ctx, "缓存未命中，回源查询数据库", "key", cacheKey, "error", err)
```

## 日志使用规则

- **带 Context 优先**：调用链有 `context.Context` 时，**必须**使用 `slog.DebugContext`、`slog.InfoContext`、`slog.WarnContext` 或 `slog.ErrorContext`，以确保 TraceID 能够随上下文写入日志。
- **结构化属性**：message 只用于描述静态事件（如 `"保存设备失败"`），动态变量必须作为 key/value 键值对传入（如 `"equipment_id", id`），**严禁**使用 `fmt.Sprintf` 拼接日志消息。
- **分级策略**：
  - `Debug`：用于缓存击穿/回源、内部状态探针等开发与排查诊断；
  - `Info`：用于关键业务里程碑、状态流转、服务生命周期事件；
  - `Warn`：用于降级、可恢复异常或触发重试的非致命状况；
  - `Error`：用于导致当前请求或任务失败的操作。循环批处理内应汇总记录，避免单条记录频繁刷屏。
- **安全与脱敏**：严禁在日志中输出密码、Token、Authorization Header、Cookie 或未经脱敏的敏感客户隐私。错误日志必须包含原始 `error` 及定位关联标识（如 ID、GUID）。
