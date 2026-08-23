# Optional 与非空返回模板

唯一 Optional 实现是 `<project>-common/pkg/optional`。使用前必须从目标项目**自身**的 common optional 源码和既有调用复核 `Option`、SQL `Scanner`/`Valuer`、JSON 与 MsgPack 编解码及实际 API、语义和版本；不得按其他项目或其他 Optional 库的命名推断。缺失时报告并停止相关实现，不引入替代库。

## 返回语义

```go
package domain

import (
	"context"

	"<project>-common/pkg/optional"
)

// <Module>Repository 定义 <module> 的持久化能力。
type <Module>Repository interface {
	// FindByID 根据 ID 查询 <module>。
	//
	// param:
	//   - id: <module> ID
	//
	// return:
	//   - result: 实体，未命中时为 optional.None
	FindByID(ctx context.Context, id int64) (optional.Option[*<Module>], error)
}
```

```go
func (r *<Module>Repository) FindByIDs(ctx context.Context, ids []int64) (map[int64]*domain.<Module>, error) {
	if len(ids) == 0 {
		return map[int64]*domain.<Module>{}, nil
	}
	// 查询并填充非 nil map。
	return result, nil
}
```

- 非基本单值使用 `optional.Option[*T]`；基本单值可按目标项目实际 API 使用 `Option[T]`。业务查询未命中返回 `optional.None[*T]()`。
- 集合结果返回非 nil 的空 slice/map，不以 nil 表达无数据。
- 除 `error` 外不要以 nil 表达查询结果缺失。
- API/proto 生成类型、controller response、数据库 model 不受此返回约束。

## API 与转换

目标项目复核后，common optional 通常应提供以下能力：

- 构造与指针转换：`Some`、`None`、`FromNillable`、`PtrFromNillable`。
- 存在性判断：`IsSome`、`IsNone`。
- 取值：`Unwrap`、`UnwrapAsPtr`、`Take`、`TakeOr`、`TakeOrElse`。`Unwrap` 对 None 返回零值；需要区分 None 时使用 `IsSome` 或 `Take`。
- 回退和筛选：`Or`、`OrElse`、`Filter`。
- 条件回调：`IfSome`、`IfSomeWithError`、`IfNone`、`IfNoneWithError`。
- 映射：`Map`、`MapOr`、`MapWithError`、`MapOrWithError`。
- 链式映射：`FlatMap`、`FlatMapOr`、`FlatMapWithError`、`FlatMapOrWithError`。
- 组合：`Zip`、`ZipWith`、`Unzip`、`UnzipWith`。

```go
// 未命中使用 None；非基本业务对象通常以指针作为 Option 的元素类型。
entity := optional.PtrFromNillable(entityPtr)
name := optional.MapOr(entity, "", func(value *domain.<Module>) string {
	return value.Name
})

// mapper 自身返回 Option 时使用 FlatMap，避免 Option[Option[T]]。
parent := optional.FlatMap(entity, func(value *domain.<Module>) optional.Option[*domain.<Module>] {
	return value.Parent()
})
```

- `FromNillable(*T)` 解引用后得到 `Option[T]`；要保留指针本身时使用 `PtrFromNillable(*T)` 得到 `Option[*T]`。
- `Take` 在 None 时返回 `ErrNoneValueTaken`；不要忽略该错误。

## 持久化与编码

`Option[T]` 实现 `database/sql.Scanner` 和 `driver.Valuer`；JSON 的 None 编码为 `null`；MsgPack 的 None 编码为 `nil`，Some 使用单元素数组格式。仅在模型或边界确有此需求时依赖这些编码语义，并用目标项目类型的测试验证。

## 禁止事项

- 不引入 `go-optional`、`moznion`、自定义 Optional 或任何替代 `<project>-common/pkg/optional` 的实现。
- 不得以 nil 表达业务单值查询未命中，也不得以 nil 的 slice/map 表达空集合。
- 不得在未复核目标项目自身 common API 的情况下套用本模板。
