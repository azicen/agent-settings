# 错误包装

如果调用其他方法时出现错误，通常有三种处理方式可以选择：

- 将原始错误原样返回
- 使用 `fmt.Errorf` 搭配 `%w` 将错误添加进上下文后返回
- 使用 `fmt.Errorf` 搭配 `%v` 将错误添加进上下文后返回

如果没有要添加的其他上下文，则按原样返回原始错误。
这将保留原始错误类型和消息。
这非常适合底层错误消息有足够的信息来追踪它来自哪里的错误。

否则，尽可能在错误消息中添加上下文
这样就不会出现诸如“连接被拒绝”之类的模糊错误，
您会收到更多有用的错误，例如“调用服务 foo：连接被拒绝”。

使用 `fmt.Errorf` 为你的错误添加上下文，
根据调用者是否应该能够匹配和提取根本原因，在 `%w` 或 `%v` 动词之间进行选择。

- 如果调用者应该可以访问底层错误，请使用 `%w`。
   对于大多数包装错误，这是一个很好的默认值，
   但请注意，调用者可能会开始依赖此行为。因此，对于包装错误是已知`var`或类型的情况，请将其作为函数契约的一部分进行记录和测试。
- 使用 `%v` 来混淆底层错误。
  调用者将无法匹配它，但如果需要，您可以在将来切换到 `%w`。

在为返回的错误添加上下文时，通过避免使用"failed to"之类的短语来保持上下文简洁，当错误通过堆栈向上渗透时，它会一层一层被堆积起来：

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
s, err := store.New()
if err != nil {
    return fmt.Errorf(
        "failed to create new store: %w", err)
}
```

</td><td>

```go
s, err := store.New()
if err != nil {
    return fmt.Errorf(
        "new store: %w", err)
}
```

</td></tr><tr><td>

```plain
failed to x: failed to y: failed to create new store: the error
```

</td><td>

```plain
x: y: new store: the error
```

</td></tr>
</tbody></table>

然而，一旦错误被发送到另一个系统，应该清楚消息是一个错误（例如`err` 标签或日志中的"Failed"前缀）。


另见 [不要只检查错误，优雅地处理它们]。

  [`"pkg/errors".Cause`]: https://pkg.go.dev/github.com/pkg/errors#Cause
  [不要只检查错误，优雅地处理它们]: https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully
