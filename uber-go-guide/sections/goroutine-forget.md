# 不要一劳永逸地使用 goroutine

Goroutines 是轻量级的，但它们不是免费的：
至少，它们会为堆栈和 CPU 的调度消耗内存。
虽然这些成本对于 Goroutines 的使用来说很小，但当它们在没有受控生命周期的情况下大量生成时会导致严重的性能问题。
具有非托管生命周期的 Goroutines 也可能导致其他问题，例如防止未使用的对象被垃圾回收并保留不再使用的资源。

因此，不要在代码中泄漏 goroutine。
使用 [go.uber.org/goleak](https://pkg.go.dev/go.uber.org/goleak)
来测试可能产生 goroutine 的包内的 goroutine 泄漏。

一般来说，每个 goroutine:

- 必须有一个可预测的停止运行时间；或者
- 必须有一种方法可以向 goroutine 发出信号它应该停止

在这两种情况下，都必须有一种方式代码来阻塞并等待 goroutine 完成。

For example:

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
go func() {
  for {
    flush()
    time.Sleep(delay)
  }
}()
```

</td><td>

```go
var (
  stop = make(chan struct{}) // 告诉 goroutine 停止
  done = make(chan struct{}) // 告诉我们 goroutine 退出了
)
go func() {
  defer close(done)
  ticker := time.NewTicker(delay)
  defer ticker.Stop()
  for {
    select {
    case <-tick.C:
      flush()
    case <-stop:
      return
    }
  }
}()
// 其它...
close(stop)  // 指示 goroutine 停止
<-done       // and wait for it to exit
```

</td></tr>
<tr><td>

没有办法阻止这个 goroutine。这将一直运行到应用程序退出。

</td><td>

这个 goroutine 可以用 `close(stop)`,
我们可以等待它退出 `<-done`.

</td></tr>
</tbody></table>

## 等待 goroutines 退出

给定一个由系统生成的 goroutine，
必须有一种方案能等待 goroutine 的退出。
有两种常用的方法可以做到这一点：

- 使用 `sync.WaitGroup`.
  如果您要等待多个 goroutine，请执行此操作

    ```go
    var wg sync.WaitGroup
    for i := 0; i < N; i++ {
      wg.Add(1)
      go func() {
        defer wg.Done()
        // ...
      }()
    }

    // To wait for all to finish:
    wg.Wait()
    ```

- 添加另一个 `chan struct{}`，goroutine 完成后会关闭它。
   如果只有一个 goroutine，请执行此操作。

    ```go
    done := make(chan struct{})
    go func() {
      defer close(done)
      // ...
    }()

    // To wait for the goroutine to finish:
    <-done
    ```

## 不要在 `init()` 使用 goroutines

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
