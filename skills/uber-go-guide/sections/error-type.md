# 错误类型

声明错误的选项很少。
在选择最适合您的用例的选项之前，请考虑以下事项。

- 调用者是否需要匹配错误以便他们可以处理它？
  如果是，我们必须通过声明顶级错误变量或自定义类型来支持 [`errors.Is`] 或 [`errors.As`] 函数。
- 错误消息是否为静态字符串，还是需要上下文信息的动态字符串？
  如果是静态字符串，我们可以使用 [`errors.New`]，但对于后者，我们必须使用 [`fmt.Errorf`] 或自定义错误类型。
- 我们是否正在传递由下游函数返回的新错误？
   如果是这样，请参阅[错误包装部分](#错误包装)。

[`errors.Is`]: https://pkg.go.dev/errors/#Is
[`errors.As`]: https://pkg.go.dev/errors/#As

| 错误匹配？| 错误消息 | 指导                           |
|-----------------|---------------|-------------------------------------|
| No              | static        | [`errors.New`]                      |
| No              | dynamic       | [`fmt.Errorf`]                      |
| Yes             | static        | top-level `var` with [`errors.New`] |
| Yes             | dynamic       | custom `error` type                 |

[`errors.New`]: https://pkg.go.dev/errors/#New
[`fmt.Errorf`]: https://pkg.go.dev/fmt/#Errorf

例如，
使用 [`errors.New`] 表示带有静态字符串的错误。
如果调用者需要匹配并处理此错误，则将此错误导出为变量以支持将其与 `errors.Is` 匹配。

<table>
<thead><tr><th>无错误匹配</th><th>错误匹配</th></tr></thead>
<tbody>
<tr><td>

```go
// package foo

func Open() error {
  return errors.New("could not open")
}

// package bar

if err := foo.Open(); err != nil {
  // Can't handle the error.
  panic("unknown error")
}
```

</td><td>

```go
// package foo

var ErrCouldNotOpen = errors.New("could not open")

func Open() error {
  return ErrCouldNotOpen
}

// package bar

if err := foo.Open(); err != nil {
  if errors.Is(err, foo.ErrCouldNotOpen) {
    // handle the error
  } else {
    panic("unknown error")
  }
}
```

</td></tr>
</tbody></table>

对于动态字符串的错误，
如果调用者不需要匹配它，则使用 [`fmt.Errorf`]，
如果调用者确实需要匹配它，则自定义 `error`。

<table>
<thead><tr><th>无错误匹配</th><th>错误匹配</th></tr></thead>
<tbody>
<tr><td>

```go
// package foo

func Open(file string) error {
  return fmt.Errorf("file %q not found", file)
}

// package bar

if err := foo.Open("testfile.txt"); err != nil {
  // Can't handle the error.
  panic("unknown error")
}
```

</td><td>

```go
// package foo

type NotFoundError struct {
  File string
}

func (e *NotFoundError) Error() string {
  return fmt.Sprintf("file %q not found", e.File)
}

func Open(file string) error {
  return &NotFoundError{File: file}
}


// package bar

if err := foo.Open("testfile.txt"); err != nil {
  var notFound *NotFoundError
  if errors.As(err, &notFound) {
    // handle the error
  } else {
    panic("unknown error")
  }
}
```

</td></tr>
</tbody></table>

请注意，如果您从包中导出错误变量或类型，
它们将成为包的公共 API 的一部分。
