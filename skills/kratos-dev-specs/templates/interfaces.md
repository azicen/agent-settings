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

- controller 嵌入生成的 `Unimplemented<Service>Server`，调用 domain service/repository，返回 proto response 和生成的业务错误；HTTP 编码由 common server 管理。
- 外部入口不得直接读写 ORM、GORM/gen query、SQL 或缓存，也不得生产 MQ 消息。
- MQ producer 由 infras/repository 实现，并经 domain interface 注入；consumer 的连接、订阅和关闭必须使用 `fx.Lifecycle`。
- 多个 repository/service 需要原子性时，入口可注入 `*gorm.DB`，但只能调用 common `tx.Transaction` 建立边界；回调内传递回调 `ctx`，下游使用 `tx.GetTx` 自动复用事务。
- 使用 `slog.*Context` 记录可诊断事件，避免记录 token、密码和原始敏感 payload；外部 I/O 必须尊重调用者的 context。
