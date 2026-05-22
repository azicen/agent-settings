# 一次性退出

如果可能的话，你的`main（）`函数中 **最多一次** 调用 `os.Exit`或者`log.Fatal`。如果有多个错误场景停止程序执行，请将该逻辑放在单独的函数下并从中返回错误。
这会缩短 `main()` 函数，并将所有关键业务逻辑放入一个单独的、可测试的函数中。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
package main
func main() {
  args := os.Args[1:]
  if len(args) != 1 {
    log.Fatal("missing file")
  }
  name := args[0]
  f, err := os.Open(name)
  if err != nil {
    log.Fatal(err)
  }
  defer f.Close()
  // 如果我们调用 log.Fatal 在这条线之后
  // f.Close 将会被执行。
  b, err := os.ReadAll(f)
  if err != nil {
    log.Fatal(err)
  }
  // ...
}
```

</td><td>

```go
package main
func main() {
  if err := run(); err != nil {
    log.Fatal(err)
  }
}
func run() error {
  args := os.Args[1:]
  if len(args) != 1 {
    return errors.New("missing file")
  }
  name := args[0]
  f, err := os.Open(name)
  if err != nil {
    return err
  }
  defer f.Close()
  b, err := os.ReadAll(f)
  if err != nil {
    return err
  }
  // ...
}
```

</td></tr>
</tbody></table>

上面的示例使用`log.Fatal`，但该指南也适用于`os.Exit`或任何调用`os.Exit`的库代码。

```go
func main() {
  if err := run(); err != nil {
    fmt.Fprintln(os.Stderr, err)
    os.Exit(1)
  }
}
```

您可以根据需要更改`run()`的签名。例如，如果您的程序必须使用特定的失败退出代码退出，`run()`可能会返回退出代码而不是错误。这也允许单元测试直接验证此行为。

```go
func main() {
  os.Exit(run(args))
}

func run() (exitCode int) {
  // ...
}
```
请注意，这些示例中使用的`run()`函数并不是强制性的。
`run()`函数的名称、签名和设置具有灵活性。除其他外，您可以：

- 接受未分析的命令行参数 (e.g., `run(os.Args[1:])`)
- 解析`main()`中的命令行参数并将其传递到`run`
- 使用自定义错误类型将退出代码传回`main（）`
- 将业务逻辑置于不同的抽象层 `package main`

本指南只要求在`main()`中有一个位置负责实际的退出流程。
