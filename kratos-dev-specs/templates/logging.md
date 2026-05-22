# Logging 模板

业务代码统一使用标准库 `log/slog`，不要直接使用 `github.com/go-kratos/kratos/v2/log`。

## 有 context.Context

如果当前作用域有 `ctx context.Context`，优先使用 Context 版本。

```go
slog.InfoContext(ctx,
    "sync acquisition task",
    "equipment_id", equipmentID,
    "status", status,
)

slog.ErrorContext(ctx,
    "save equipment failed",
    "equipment_id", equipmentID,
    "error", err,
)
```

## 没有 context.Context

没有 ctx 时才使用非 Context 方法。

```go
slog.Info("start scheduler", "module", "equipment")
```

## 规则

- 不直接使用 Kratos `log`。
- 使用结构化字段，但默认直接传 key/value：`"user_id", userID`；不要为了简单字段写成 `slog.String(...)`、`slog.Int64(...)` 等更复杂形式。
- 不把关键变量拼接进 message。
- 只记录有诊断价值的边界事件、异常和关键状态。
- 有 ctx 时不要丢弃上下文写日志。
