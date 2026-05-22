# 等待 goroutines 退出

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
