# 一次处理错误

当调用方从被调用方接收到错误时，它可以根据对错误的了解，以各种不同的方式进行处理。

其中包括但不限于：

- 如果被调用者约定定义了特定的错误，则将错误与`errors.Is`或`errors.As`匹配，并以不同的方式处理分支
- 如果错误是可恢复的，则记录错误并正常降级
- 如果该错误表示特定于域的故障条件，则返回定义明确的错误
- 返回错误，无论是 [wrapped](#错误包装) 还是逐字逐句

无论调用方如何处理错误，它通常都应该只处理每个错误一次。例如，调用方不应该记录错误然后返回，因为*its*调用方也可能处理错误。

例如，考虑以下情况：

<table>
<thead><tr><th>Description</th><th>Code</th></tr></thead>
<tbody>
<tr><td>

**Bad**: 记录错误并将其返回

堆栈中的调用程序可能会对该错误采取类似的操作。这样做会在应用程序日志中造成大量噪音，但收效甚微。

</td><td>

```go
u, err := getUser(id)
if err != nil {
  // BAD: See description
  log.Printf("Could not get user %q: %v", id, err)
  return err
}
```

</td></tr>
<tr><td>

**Good**: 将错误换行并返回



堆栈中更靠上的调用程序将处理该错误。使用`%w`可确保它们可以将错误与`errors.Is`或`errors.As`相匹配（如果相关）。

</td><td>

```go
u, err := getUser(id)
if err != nil {
  return fmt.Errorf("get user %q: %w", id, err)
}
```

</td></tr>
<tr><td>

**Good**: 记录错误并正常降级

如果操作不是绝对必要的，我们可以通过从中恢复来提供降级但不间断的体验。

</td><td>

```go
if err := emitMetrics(); err != nil {
  // Failure to write metrics should not
  // break the application.
  log.Printf("Could not emit metrics: %v", err)
}

```

</td></tr>
<tr><td>

**Good**: 匹配错误并适当降级

如果被调用者在其约定中定义了一个特定的错误，并且失败是可恢复的，则匹配该错误案例并正常降级。对于所有其他案例，请包装错误并返回。

堆栈中更靠上的调用程序将处理其他错误。

</td><td>

```go
tz, err := getUserTimeZone(id)
if err != nil {
  if errors.Is(err, ErrUserNotFound) {
    // User doesn't exist. Use UTC.
    tz = time.UTC
  } else {
    return fmt.Errorf("get user %q: %w", id, err)
  }
}
```

</td></tr>
</tbody></table>
