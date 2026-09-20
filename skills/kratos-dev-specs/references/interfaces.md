# Interfaces 模板

`interfaces` 放置 HTTP/gRPC controller、MQ consumer、定时/后台任务等外部入口。它们负责信任边界的输入校验、协议转换、幂等边界与调用编排；状态流转、跨资源规则和复杂业务逻辑归 domain service。

```go
// Consume 消费一条 <module> 消息
func (c *<Module>Consumer) Consume(ctx context.Context, payload []byte) error {
	command, err := decode<Module>Command(payload)
	if err != nil {
		return err
	}
	return c.svc.Handle(ctx, command)
}

// Run 执行 <module> 定时任务
func (j *<Module>Job) Run(ctx context.Context) error {
	return j.svc.Run(ctx)
}
```

- **Controller**：嵌入生成的 `Unimplemented<Module>ServiceServer`，调用 domain service/repository，返回 proto response 和生成的业务错误；HTTP 成功/失败的结构体包装由 common server 统一管理（`json-full` 编解码器与 `baseResponse`）。
- **外部入口职责**：不得直接读写 ORM、GORM/gen query、SQL 或缓存，也不得在 controller/interfaces 中直接创建全局 goroutine 或外部 client。
- **消息与异步**：MQ producer 由 infras/repository 实现，并通过 domain interface 抽象向领域层暴露；MQ consumer 的连接、订阅和关闭必须通过 `fx.Lifecycle` 纳管。
- **原子性边界**：多个 repository/service 操作需要保持原子性时，入口可注入 `*gorm.DB`，但**只能**调用 common 的 `tx.Transaction` 建立边界；回调内传递回调 `txCtx`，下游仓储内部通过 `tx.GetTx(txCtx, r.db)` 自动复用该事务。
- **日志与上下文**：使用 `slog.*Context` 记录诊断与关键业务事件，严禁记录 token、密码和未脱敏的敏感信息；所有 I/O 必须严格传递并尊重调用者的 `context.Context`。
