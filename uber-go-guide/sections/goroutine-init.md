# 不要在 `init()` 使用 goroutines

`init()` 函数不应该产生 goroutines。
另请参阅 [避免使用 init()](#避免使用-init)。

如果一个包需要一个后台 goroutine，
它必须公开一个负责管理 goroutine 生命周期的对象。
该对象必须提供一个方法（`Close`、`Stop`、`Shutdown` 等）来指示后台 goroutine 停止并等待它的退出。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func init() {
  go doWork()
}
func doWork() {
  for {
    // ...
  }
}
```

</td><td>

```go
type Worker struct{ /* ... */ }
func NewWorker(...) *Worker {
  w := &Worker{
    stop: make(chan struct{}),
    done: make(chan struct{}),
    // ...
  }
  go w.doWork()
}
func (w *Worker) doWork() {
  defer close(w.done)
  for {
    // ...
    case <-w.stop:
      return
  }
}
// Shutdown 告诉 worker 停止
// 并等待它完成。
func (w *Worker) Shutdown() {
  close(w.stop)
  <-w.done
}
```

</td></tr>
<tr><td>

当用户导出这个包时，无条件地生成一个后台 goroutine。
用户无法控制 goroutine 或停止它的方法。

</td><td>

仅当用户请求时才生成工作人员。
提供一种关闭工作器的方法，以便用户可以释放工作器使用的资源。

请注意，如果工作人员管理多个 goroutine，则应使用`WaitGroup`。
请参阅 [等待 goroutines 退出](#等待-goroutines-退出)。


</td></tr>
</tbody></table>
