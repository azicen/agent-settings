# 错误命名

对于存储为全局变量的错误值，
根据是否导出，使用前缀 `Err` 或 `err`。
请看指南 [对于未导出的顶层常量和变量，使用_作为前缀](#对于未导出的顶层常量和变量使用_作为前缀)。

```go
var (
  // 导出以下两个错误，以便此包的用户可以将它们与 errors.Is 进行匹配。

  ErrBrokenLink = errors.New("link is broken")
  ErrCouldNotOpen = errors.New("could not open")

  // 这个错误没有被导出，因为我们不想让它成为我们公共 API 的一部分。我们可能仍然在带有错误的包内使用它。

  errNotFound = errors.New("not found")
)
```

对于自定义错误类型，请改用后缀 `Error`。

```go
// 同样，这个错误被导出，以便这个包的用户可以将它与 errors.As 匹配。

type NotFoundError struct {
  File string
}

func (e *NotFoundError) Error() string {
  return fmt.Sprintf("file %q not found", e.File)
}

// 并且这个错误没有被导出，因为我们不想让它成为公共 API 的一部分。我们仍然可以在带有 errors.As 的包中使用它。
type resolveError struct {
  Path string
}

func (e *resolveError) Error() string {
  return fmt.Sprintf("resolve %q", e.Path)
}
```
