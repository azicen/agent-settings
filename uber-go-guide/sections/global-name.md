# 对于未导出的顶层常量和变量，使用_作为前缀

在未导出的顶级`vars`和`consts`，前面加上前缀_，以使它们在使用时明确表示它们是全局符号。

基本依据：顶级变量和常量具有包范围作用域。使用通用名称可能很容易在其他文件中意外使用错误的值。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// foo.go

const (
  defaultPort = 8080
  defaultUser = "user"
)

// bar.go

func Bar() {
  defaultPort := 9090
  ...
  fmt.Println("Default port", defaultPort)

  // We will not see a compile error if the first line of
  // Bar() is deleted.
}
```

</td><td>

```go
// foo.go

const (
  _defaultPort = 8080
  _defaultUser = "user"
)
```

</td></tr>
</tbody></table>

**例外**：未导出的错误值可以使用不带下划线的前缀 `err`。参见[错误命名](#错误命名)。
