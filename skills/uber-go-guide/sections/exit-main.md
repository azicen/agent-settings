# 主函数退出方式 (Exit)

Go 程序使用 [`os.Exit`] 或者 [`log.Fatal*`] 立即退出 (使用`panic`不是退出程序的好方法，请 [不要使用 panic](#不要使用-panic)。)

  [`os.Exit`]: https://pkg.go.dev/os/#Exit
  [`log.Fatal*`]: https://pkg.go.dev/log/#Fatal

**仅在`main()`** 中调用其中一个 `os.Exit` 或者 `log.Fatal*`。所有其他函数应将错误返回到信号失败中。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func main() {
  body := readFile(path)
  fmt.Println(body)
}
func readFile(path string) string {
  f, err := os.Open(path)
  if err != nil {
    log.Fatal(err)
  }
  b, err := os.ReadAll(f)
  if err != nil {
    log.Fatal(err)
  }
  return string(b)
}
```

</td><td>

```go
func main() {
  body, err := readFile(path)
  if err != nil {
    log.Fatal(err)
  }
  fmt.Println(body)
}
func readFile(path string) (string, error) {
  f, err := os.Open(path)
  if err != nil {
    return "", err
  }
  b, err := os.ReadAll(f)
  if err != nil {
    return "", err
  }
  return string(b), nil
}
```

</td></tr>
</tbody></table>

原则上：退出的具有多种功能的程序存在一些问题：

- 不明显的控制流：任何函数都可以退出程序，因此很难对控制流进行推理。
- 难以测试：退出程序的函数也将退出调用它的测试。这使得函数很难测试，并引入了跳过 `go test` 尚未运行的其他测试的风险。
- 跳过清理：当函数退出程序时，会跳过已经进入`defer`队列里的函数调用。这增加了跳过重要清理任务的风险。
## 一次性退出

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
