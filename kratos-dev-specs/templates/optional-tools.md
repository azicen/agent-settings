# go-optional 工具函数模板

`github.com/moznion/go-optional` 已经提供了构造、判断、取值、转换、组合等工具函数。编写业务代码时应先查阅并复用这些内置能力，不要为简单取值、默认值、指针转换、map/flatMap、fallback 等场景新增自定义 helper。

## 构造 Option

### optional.Some

有值时使用 `optional.Some[T](v)`。

```go
entity := optional.Some[*domain.<Module>](module)
```

### optional.None

无值时使用 `optional.None[T]()`，例如 repository 未命中。

```go
return optional.None[*domain.<Module>](), nil
```

### optional.FromNillable

将可 nil 指针转换为 `Option[T]`。指针为 nil 时返回 `None`，否则解引用后返回 `Some[T]`。

```go
opt := optional.FromNillable(namePtr)
```

### optional.PtrFromNillable

将可 nil 指针转换为 `Option[*T]`。指针为 nil 时返回 `None`，否则保留指针并返回 `Some[*T]`。

```go
opt := optional.PtrFromNillable(entityPtr)
```

## 判断是否有值

### IsSome

判断 Option 是否有值。

```go
if entity.IsSome() {
    return entity.Unwrap(), nil
}
```

### IsNone

判断 Option 是否无值。

```go
if entity.IsNone() {
    return v1.Error<Module>NotFound("资源不存在")
}
```

## 取值和默认值

### Unwrap

直接取出值。只在已经通过 `IsSome()`、业务不变量或调用链保证有值时使用。

```go
entity := opt.Unwrap()
```

### UnwrapAsPtr

以指针形式取出值。需要把 `Option[T]` 转回 `*T` 时优先使用它，不要新增自定义指针转换 helper。

```go
ptr := opt.UnwrapAsPtr()
```

### Take

取出值并返回错误。调用方需要显式处理 None 时使用。

```go
value, err := opt.Take()
if err != nil {
    return optional.None[*domain.<Module>](), err
}
```

### TakeOr

无值时返回固定默认值。

```go
name := opt.TakeOr("默认名称")
```

### TakeOrElse

无值时通过函数延迟计算默认值。

```go
name := opt.TakeOrElse(func() string {
    return buildDefaultName()
})
```

## 条件回调

### IfSome

有值时执行回调。

```go
opt.IfSome(func(entity *domain.<Module>) {
    slog.InfoContext(ctx, "module found", "id", entity.ID)
})
```

### IfNone

无值时执行回调。

```go
opt.IfNone(func() {
    slog.InfoContext(ctx, "module not found", "id", id)
})
```

### IfSomeWithError / IfNoneWithError

回调可能失败时使用带 error 的版本。

```go
err := opt.IfSomeWithError(func(entity *domain.<Module>) error {
    return r.Save(ctx, entity)
})
if err != nil {
    return err
}
```

## fallback Option

### Or

当前 Option 无值时返回备用 Option。

```go
entity := cached.Or(dbValue)
```

### OrElse

当前 Option 无值时延迟构造备用 Option。

```go
entity := cached.OrElse(func() optional.Option[*domain.<Module>] {
    return loadFromDB(ctx, id)
})
```

## 过滤

### Filter

有值且满足条件时保留，否则返回 None。

```go
active := entity.Filter(func(v *domain.<Module>) bool {
    return v.Enabled
})
```

## 转换

### optional.Map

将 `Option[T]` 转换为 `Option[U]`。

```go
info := optional.Map(entity, func(v *domain.<Module>) *v1.<Module>Info {
    return to<Module>Info(v)
})
```

### optional.MapOr

有值时转换，无值时返回固定默认值。

```go
name := optional.MapOr(entity, "", func(v *domain.<Module>) string {
    return v.Name
})
```

### optional.MapWithError

转换函数可能失败时使用。

```go
info, err := optional.MapWithError(entity, func(v *domain.<Module>) (*v1.<Module>Info, error) {
    return to<Module>InfoWithError(v)
})
```

### optional.MapOrWithError

有值时执行可能失败的转换，无值时返回固定默认值。

```go
name, err := optional.MapOrWithError(entity, "", func(v *domain.<Module>) (string, error) {
    return buildDisplayName(v)
})
```

## 链式转换

### optional.FlatMap

转换函数本身返回 Option 时使用，避免嵌套 `Option[Option[T]]`。

```go
parent := optional.FlatMap(entity, func(v *domain.<Module>) optional.Option[*domain.<Module>] {
    return v.Parent()
})
```

### optional.FlatMapOr

链式转换后无值时返回固定默认值。

```go
parentName := optional.FlatMapOr(entity, "", func(v *domain.<Module>) optional.Option[string] {
    return v.ParentName()
})
```

### optional.FlatMapWithError / optional.FlatMapOrWithError

链式转换可能失败时使用带 error 的版本。

```go
parent, err := optional.FlatMapWithError(entity, func(v *domain.<Module>) (optional.Option[*domain.<Module>], error) {
    return repo.FindByID(ctx, v.ParentID)
})
```

## 组合和拆分

### optional.Zip

两个 Option 都有值时组合为 `Option[Pair[T, U]]`，任意一个无值则返回 None。

```go
pair := optional.Zip(moduleOpt, ownerOpt)
```

### optional.ZipWith

两个 Option 都有值时通过函数组合成新值。

```go
summary := optional.ZipWith(moduleOpt, ownerOpt, func(module *domain.<Module>, owner *domain.Owner) *domain.Summary {
    return domain.NewSummary(module, owner)
})
```

### optional.Unzip

将 `Option[Pair[T, U]]` 拆成两个 Option。

```go
moduleOpt, ownerOpt := optional.Unzip(pair)
```

### optional.UnzipWith

将 `Option[V]` 按函数拆成两个 Option。

```go
codeOpt, nameOpt := optional.UnzipWith(entity, func(v *domain.<Module>) (string, string) {
    return v.Code, v.Name
})
```

## JSON 和 SQL 支持

`Option[T]` 实现了 JSON marshal/unmarshal，也实现了 `database/sql/driver.Valuer` 和 `database/sql.Scanner`。如果确实需要在边界层或持久化适配中处理 Optional JSON/SQL 值，优先使用内置实现，不要重复封装。

## 使用规则

- 单个可能不存在的查询结果使用 `optional.Option[T]`。
- 未命中返回 `optional.None[T]()`，命中返回 `optional.Some(value)`。
- nil 指针转换优先使用 `FromNillable` 或 `PtrFromNillable`。
- 默认值优先使用 `TakeOr`、`TakeOrElse`、`MapOr`、`FlatMapOr`。
- 转换优先使用 `Map`、`FlatMap` 及其 WithError 版本。
- 多个 Optional 组合优先使用 `Zip`、`ZipWith`。
- 只有确认有值时才使用 `Unwrap`。
- 不为上述场景新增项目级自定义 helper。
