# 注释模板

每个函数使用以函数名开头的 Go doc 注释。注释说明业务意图、边界条件和调用约束，不把每一行代码翻译成自然语言。

```go
// FindByID 根据 ID 查询设备
// 未命中以 optional.None 返回，调用方据此选择响应
//
// param:
//   - id: 设备 ID
//
// return:
//   - result: 设备实体，未命中时为 optional.None
func (r *EquipmentRepository) FindByID(ctx context.Context, id int64) (optional.Option[*domain.Equipment], error) {
	// ...
}
```

```go
// 计算缓存未命中的 ID，只为缺失数据执行数据库查询。
missingIDs := make([]int64, 0, len(ids))
for _, id := range ids {
	if _, ok := cached[id]; !ok {
		missingIDs = append(missingIDs, id)
	}
}
```

规则：

- `param:` 省略 `ctx`；没有需说明的参数时整个段落省略。`return:` 省略 `error`、`emptypb.Empty` 与无需解释的空响应。
- `param:` / `return:` 后直接接条目，格式固定为 `//   - name: 描述`。repository interface 与实现方法（`New` 除外）必须保留非 `ctx` 参数和非 `error` 返回说明。
- `New` 函数只需描述创建何种对象；不要为了模板重复列出注入依赖。
- 为复杂 SQL、缓存未命中计算、事务编排、并发/资源生命周期及幂等边界添加说明原因和约束的段落注释；简单赋值、显然的条件和直观调用不写流水账注释。
- 注释与实现同步修改。行为变化时先更新约束说明，避免留下过期承诺。

## 结构体与字段注释

每个 `struct` 使用以类型名开头的简短 Go doc，说明其业务角色或技术职责。

数据结构（领域实体、值对象、DTO、配置、数据库 model、协议 request/response、跨层查询结果）中的每个字段，均须在紧邻位置使用以字段名开头的注释。字段注释应表达业务含义、单位、可空性、所有权或约束，禁止仅复述类型。

并发调度 `struct` 的所有字段均须逐项注释，明确：依赖职责；状态或 timer 的所有权；channel 的消息语义、缓冲和关闭方；`done` 的完成条件；以及 mutex/once 的保护范围与幂等语义。

普通私有依赖聚合 `struct` 不强制每个字段逐项注释，除非字段的意图、资源所有权或生命周期不显然。

错误示例：不要在 `struct` 顶层注释中罗列构造参数。顶层 Go doc 的职责是说明类型本身，构造依赖属于构造函数的注释；把参数清单放在这里既不能解释字段约束，也会在构造函数变化时留下过期信息。

```go
// ModbusTCPReadAccessSyncScheduler 创建时需要 syncFacade、settings 和 requests。
type ModbusTCPReadAccessSyncScheduler struct {
	// ...
}
```

正确示例：以下仅演示字段注释的粒度。字段集合、channel 缓冲、写入方、关闭方和状态所有权必须按具体实现设计，不得机械复制。

```go
// Record 表示可持久化或跨层传递的通用业务记录。
type Record struct {
	// ID 记录唯一标识。
	ID int64
	// Name 记录名称。
	Name string
	// Description 记录描述，可为空。
	Description optional.Option[string]
	// Metadata 附加属性；键和值的业务语义由所属模块定义。
	Metadata map[string]string
	// CreatedAt 创建时间。
	CreatedAt time.Time
}
```

正确示例：以下仅演示并发对象字段的职责、所有权和同步语义；具体调度策略由实现定义。

```go
// TaskDispatcher 协调后台任务请求及其停止过程；具体调度策略由实现定义。
type TaskDispatcher struct {
	// executor 执行单个任务，不由 dispatcher 创建或关闭。
	executor TaskExecutor
	// settings 提供调度配置，仅由 worker goroutine 读取。
	settings DispatchSettings
	// requests 是任务请求通道；缓冲容量、写入方、读取方和关闭策略必须与具体实现一致。
	requests chan context.Context
	// stop 是停止信号通道；Stop 通过 stopOnce 关闭，通知 worker 退出。
	stop chan struct{}
	// done 由 worker 退出时关闭；Stop 等待该通道确认退出完成。
	done chan struct{}
	// stopOnce 确保 stop 仅关闭一次。
	stopOnce sync.Once
}
```
