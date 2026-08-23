# Go 语言附录

本附录只写 Go 特有的坑、惯用法和反模式，不重复通用原则。

---

## 一、语言特有的关注点

### 错误处理

error 是值，不是异常。**永远不要 `_ = err`**。

```go
// ✓ 检查并传播，附加上下文
result, err := doSomething(id)
if err != nil {
    return fmt.Errorf("do something for %s: %w", id, err)
}

// ✗ 静默丢弃
result, _ := doSomething(id)

// ✗ 丢弃原始错误，失去调用链
return errors.New("failed")
```

使用 `errors.Is` / `errors.As` 做错误类型判断，不做字符串比较。

### Goroutine 生命周期

goroutine 泄漏比内存泄漏更危险——泄漏的 goroutine 持有引用，阻止 GC 回收。

每个 goroutine 必须能被终止。三种终止手段：

1. `context.Context` 取消（首选，贯穿整个调用链）
2. channel 关闭信号（goroutine 消费 channel 时）
3. `sync.WaitGroup` / `errgroup`（管理 goroutine 群组完成）

```go
// ✓ 使用 errgroup 管理并发任务，任一失败则全部取消
g, ctx := errgroup.WithContext(ctx)
for _, item := range items {
    item := item
    g.Go(func() error {
        return process(ctx, item)
    })
}
if err := g.Wait(); err != nil {
    return fmt.Errorf("batch process: %w", err)
}
```

### Context 传播

`context.Context` 是超时、取消和追踪信息的唯一标准载体。作为第一个参数传递，命名为 `ctx`。

context **不**放可选参数或业务数据——只用于请求范围的元数据（调用链 ID、超时、用户身份等）。

### 接口惯例

接口在**消费方**定义，不在实现方。只在需要多态时引入接口。

```go
// ✓ 消费方定义需要什么能力
type orderRepository interface {
    FindByID(ctx context.Context, id string) (*Order, error)
    Save(ctx context.Context, order *Order) error
}

type OrderService struct {
    repo orderRepository
}
```

不要为单一实现定义接口——如果没有第二个真实实现，就不需要这个接口。

### 不可变语义

Go 没有语言级不可变支持。不要为了"不可变性"到处 deep copy。

| 场景 | 做法 |
|------|------|
| struct 字段保护 | 不导出字段（小写），提供 getter 方法 |
| 小 struct 传递 | 按值传递（自动复制） |
| 大 struct 或 slice 传递 | 用接口暴露只读视图，或通过 channel 传递所有权 |
| 配置对象 | 初始化后不修改 |

### 零值可用

struct 的零值应尽可能有用。如果零值不可用，提供构造函数——不要依赖调用方记得调 `Init()`。

```go
// ✓ 零值即可用
var mu sync.Mutex
var buf bytes.Buffer

// ✗ 零值不可用，需要 Init
type Cache struct {
    data map[string]string
}
// 应改为构造函数: func NewCache() *Cache { ... }
```

### 导出控制

- 默认不导出（小写开头），只导出外部真正需要的符号
- 导出的符号是 API 承诺，修改有兼容性成本

### 命名惯例

- 缩写全大写：`HTTP`、`ID`、`URL`，不是 `Http`、`Id`
- 不用 getter 的 `Get` 前缀：`Name()` 而非 `GetName()`
- 接口名以 `er` 结尾（单方法）：`Reader`、`Writer`
- 不用下划线，不用匈牙利命名

---

## 二、常用工具和模式

### 错误包装

```go
// 标准库
fmt.Errorf("operation %s on %s: %w", op, target, err)
```

### 并发管理

```go
import "golang.org/x/sync/errgroup"
```

`errgroup` 适用于"一组 goroutine 任一失败则全体取消"的模式。优于手动 `sync.WaitGroup` + error channel。

### 资源释放

`defer` 紧跟资源获取。**循环内注意**：循环内的 defer 在函数退出时才执行，不是每次迭代结束。

```go
// ✗ 所有文件在函数退出时才关闭
for _, name := range filenames {
    f, _ := os.Open(name)
    defer f.Close()
    process(f)
}

// ✓ 每次迭代结束时关闭
for _, name := range filenames {
    f, _ := os.Open(name)
    process(f)
    f.Close()
}
```

### 静态检查

```bash
go vet ./...
golangci-lint run
```

推荐启用的 linter：`errcheck`（未处理的 error）、`wrapcheck`（直接 return err）、`gosec`（安全）、`ineffassign`（无效赋值）。

---

## 三、语言特有的反模式

| 反模式 | 正确做法 |
|--------|---------|
| `panic` 做业务流程控制 | error 是值，用 error 传播 |
| 锁内做 IO（网络、文件、channel） | 锁只保护状态读写，IO 前先释放锁 |
| 用 `reflect` 或过度泛型消除重复 | 重复三次以上才考虑抽象。优先用接口和代码生成 |
| `init()` 里做 IO | init 只做简单注册/初始化。复杂初始化放构造函数 |
| 过早定义接口（只有一个实现） | 在消费方定义，第二个真实实现出现时才考虑提取 |
| defer 在循环里 | 循环内显式 close，或把循环体抽成独立函数 |
| channel 关闭方不明确 | 谁创建 channel 谁负责关闭，关闭方是唯一的发送方 |
