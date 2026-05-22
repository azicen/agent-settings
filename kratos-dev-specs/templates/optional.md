# Optional 和非空返回模板

除外部触发器和数据库模型外，其他业务/基础设施代码的查询结果都应该使用容器包装，避免除 `error` 外直接返回 nil。另外，optional、切片、数组、map 等容器包裹的数据，如果不是基本数据类型，则必须使用指针（例如 `optional.Option[*domain.Entity]` 而不是 `optional.Option[domain.Entity]`）。

## 单个可能不存在的结果

使用 `github.com/moznion/go-optional` 的 `optional.Option[T]`。常用构造、判断、取值、转换和组合工具函数见 [optional-tools.md](optional-tools.md)。

```go
package domain

import (
    "context"

    "github.com/moznion/go-optional"
)

type <Module>Repository interface {
    // FindByID 根据ID获取<模块>
    //
    // param:
    //   - id: <模块>ID
    //
    // return:
    //   - result: <模块>实体，未命中时返回 optional.None
    FindByID(ctx context.Context, id int64) (optional.Option[*<Module>], error)
}
```

实现中未命中时返回 `optional.None[T]()`。

```go
func (r *<Module>RepositoryImpl) FindByID(ctx context.Context, id int64) (optional.Option[*domain.<Module>], error) {
    q := query.Use(tx.GetTx(ctx, r.db)).<Module>
    m, err := q.WithContext(ctx).Where(q.ID.Eq(id)).First()
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return optional.None[*domain.<Module>](), nil
        }
        return optional.None[*domain.<Module>](), err
    }
    return optional.Some(r.toEntity(m)), nil
}
```

## 集合结果

使用 slice 或 map；没有数据时返回空容器，不返回 nil。

```go
func (r *<Module>RepositoryImpl) FindByIDs(ctx context.Context, ids []int64) (map[int64]*domain.<Module>, error) {
    if len(ids) == 0 {
        return map[int64]*domain.<Module>{}, nil
    }

    q := query.Use(tx.GetTx(ctx, r.db)).<Module>
    models, err := q.WithContext(ctx).Where(q.ID.In(ids...)).Find()
    if err != nil {
        return map[int64]*domain.<Module>{}, err
    }

    result := make(map[int64]*domain.<Module>, len(models))
    for _, m := range models {
        entity := r.toEntity(m)
        result[entity.ID] = entity
    }
    return result, nil
}
```

## 规则

- 适用范围：除外部触发器和数据库模型外，其他业务/基础设施代码的查询结果。
- 单个可能不存在的结果使用 `optional.Option[T]`。
- 集合结果使用 slice 或 map。
- 未命中返回 `optional.None[T]()`。
- 空集合返回空 slice/map。
- 除 `error` 外不要直接返回 nil。
- API/proto 生成类型、controller response 和数据库 model 不纳入该限制。
- 使用 optional 时优先复用 go-optional 内置方法，不随意新增自定义转换函数；常用工具函数见 [optional-tools.md](optional-tools.md)。
