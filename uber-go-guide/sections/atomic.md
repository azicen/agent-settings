# 使用 go.uber.org/atomic

使用 [sync/atomic] 包的原子操作对原始类型 (`int32`, `int64`等）进行操作，因为很容易忘记使用原子操作来读取或修改变量。

[go.uber.org/atomic] 通过隐藏基础类型为这些操作增加了类型安全性。此外，它包括一个方便的`atomic.Bool`类型。

[go.uber.org/atomic]: https://pkg.go.dev/go.uber.org/atomic
[sync/atomic]: https://pkg.go.dev/sync/atomic/

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type foo struct {
  running int32  // atomic
}

func (f* foo) start() {
  if atomic.SwapInt32(&f.running, 1) == 1 {
     // already running…
     return
  }
  // start the Foo
}

func (f *foo) isRunning() bool {
  return f.running == 1  // race!
}
```

</td><td>

```go
type foo struct {
  running atomic.Bool
}

func (f *foo) start() {
  if f.running.Swap(true) {
     // already running…
     return
  }
  // start the Foo
}

func (f *foo) isRunning() bool {
  return f.running.Load()
}
```

</td></tr>
</tbody></table>
