# Repository 模板

Repository interface 在 domain，具体实现放 `infras/repository`。

## Domain interface

```go
package domain

import (
    "context"

    "github.com/moznion/go-optional"

    "<project>/pkg/util/pagination"
)

type <Module>Search struct {
    IDs  []int64
    Code string
    Name string
}

// <Module>Repository <模块>仓储接口
type <Module>Repository interface {
    // Save 保存<模块>
    //
    // param:
    //   - entity: 待保存的<模块>实体
    Save(ctx context.Context, entity *<Module>) error
    // Delete 删除<模块>
    //
    // param:
    //   - entity: 待删除的<模块>实体
    Delete(ctx context.Context, entity *<Module>) error
    // FindByID 根据ID获取<模块>
    //
    // param:
    //   - id: <模块>ID
    //
    // return:
    //   - result: <模块>实体，未命中时返回 optional.None
    FindByID(ctx context.Context, id int64) (optional.Option[*<Module>], error)
    // Search<Module> 分页搜索<模块>
    //
    // param:
    //   - page: 分页参数
    //   - search: 搜索条件
    //
    // return:
    //   - result: 分页搜索结果
    Search<Module>(ctx context.Context, page pagination.PageRequest, search <Module>Search) (pagination.PageResponse[*<Module>], error)
}
```

## Infras implementation

```go
package repository

import (
    "context"
    "errors"
    "log/slog"

    "<project>/module/<module>/domain"
    "<project>/module/<module>/infras/repository/cache"
    "<project>/module/<module>/infras/repository/db/model"
    "<project>/module/<module>/infras/repository/db/query"
    "<project>/pkg/tx"

    "github.com/moznion/go-optional"
    gormlib "gorm.io/gorm"
)

// <Module>RepositoryImpl <模块>仓储实现
type <Module>RepositoryImpl struct {
    db      *gormlib.DB
    cache   *cache.<Module>Cache
    factory *domain.<Module>Factory
}

// New<Module>Repository 创建<模块>仓储实现
func New<Module>Repository(db *gormlib.DB, cache *cache.<Module>Cache, factory *domain.<Module>Factory) *<Module>RepositoryImpl {
    return &<Module>RepositoryImpl{
        db:      db,
        cache:   cache,
        factory: factory,
    }
}

// toModel 将<模块>领域实体转换为数据库模型
//
// param:
//   - entity: <模块>领域实体
//
// return:
//   - result: <模块>数据库模型
func (r *<Module>RepositoryImpl) toModel(entity *domain.<Module>) *model.<Module> {
    return &model.<Module>{ID: entity.ID}
}

// toEntity 将数据库模型转换为<模块>领域实体
//
// param:
//   - m: <模块>数据库模型
//
// return:
//   - result: <模块>领域实体
func (r *<Module>RepositoryImpl) toEntity(m *model.<Module>) *domain.<Module> {
    return r.factory.Reconstruct(m.ID)
}

// FindByID 根据ID获取<模块>
//
// param:
//   - id: <模块>ID
//
// return:
//   - result: <模块>实体，未命中时返回 optional.None
func (r *<Module>RepositoryImpl) FindByID(ctx context.Context, id int64) (optional.Option[*domain.<Module>], error) {
    cached, err := r.cache.Get(ctx, id)
    if err != nil {
        slog.DebugContext(ctx, "cache get failed", "id", id, "error", err)
    } else if cached.IsSome() {
        return cached, nil
    }

    q := query.Use(tx.GetTx(ctx, r.db)).<Module>
    m, err := q.WithContext(ctx).Where(q.ID.Eq(id)).First()
    if err != nil {
        if errors.Is(err, gormlib.ErrRecordNotFound) {
            return optional.None[*domain.<Module>](), nil
        }
        slog.ErrorContext(ctx, "find module failed", "id", id, "error", err)
        return optional.None[*domain.<Module>](), err
    }

    entity := r.toEntity(m)
    err = r.cache.Put(ctx, entity)
    if err != nil {
        slog.DebugContext(ctx, "cache put failed", "id", entity.ID, "error", err)
    }
    return optional.Some(entity), nil
}

// Save 保存<模块>
//
// param:
//   - entity: 待保存的<模块>实体
func (r *<Module>RepositoryImpl) Save(ctx context.Context, entity *domain.<Module>) error {
    q := query.Use(tx.GetTx(ctx, r.db)).<Module>
    err := q.WithContext(ctx).Save(r.toModel(entity))
    if err != nil {
        return err
    }
    err = r.cache.Delete(ctx, entity.ID)
    if err != nil {
        slog.DebugContext(ctx, "cache delete failed", "id", entity.ID, "error", err)
    }
    return nil
}
```

## Repository 内部事务

多表写入、批量写入、读写组合等需要原子性的 repository 方法，使用 `tx.Transaction` 开启事务；事务回调内继续传递回调参数里的 `ctx`。

```go
// SaveBatch 批量保存<模块>
//
// param:
//   - entities: 待保存的<模块>实体列表
func (r *<Module>RepositoryImpl) SaveBatch(ctx context.Context, entities []*domain.<Module>) error {
    return tx.Transaction(ctx, r.db, func(ctx context.Context) error {
        q := query.Use(tx.GetTx(ctx, r.db)).<Module>
        for _, entity := range entities {
            err := q.WithContext(ctx).Save(r.toModel(entity))
            if err != nil {
                return err
            }
        }
        return nil
    })
}
```

## 复杂 SQL 和 mapper

复杂数据库操作需要 SQL 时，不在 repository 实现中手写 SQL。将 mapper interface、SQL 入参/结果 DTO 和注释 SQL 放到 `infras/repository/db/mapper`，通过 `mage gorm` 生成 `infras/repository/db/sqlquery` 后在 repository 中调用。

```go
package mapper

import (
    "context"

    "gorm.io/cli/gorm/genconfig"
)

var _ = genconfig.Config{
    IncludeInterfaces: []any{"<Module>Mapper"},
    ExcludeStructs:    []any{"*"},
}

type <Module>SearchDTO struct {
    IDs    []int64
    Name   string
    Limit  int
    Offset int
}

type <Module>Mapper[T any] interface {
    /*
        SELECT id, name
        FROM <table_name>
        WHERE deleted = 0
          {{if len(search.IDs) > 0}}
          AND id IN @search.IDs
          {{end}}
          {{if search.Name != ""}}
          AND name LIKE CONCAT('%', @search.Name, '%')
          {{end}}
        ORDER BY id DESC
        LIMIT @search.Limit
        OFFSET @search.Offset
    */
    Search<Module>(ctx context.Context, search <Module>SearchDTO) ([]T, error)
}
```

```go
import (
    "<project>/module/<module>/infras/repository/db/mapper"
    "<project>/module/<module>/infras/repository/db/sqlquery"
    "<project>/pkg/tx"
)

models, err := sqlquery.<Module>Mapper[*model.<Module>](tx.GetTx(ctx, r.db)).
    Search<Module>(ctx, mapper.<Module>SearchDTO{
        IDs:    search.IDs,
        Name:   search.Name,
        Limit:  int(page.PageSize),
        Offset: int((page.Page - 1) * page.PageSize),
    })
if err != nil {
    return pagination.PageResponse[*domain.<Module>]{}, err
}
```

## 规则

- 常规查询使用 GORM/gen query，不直接写裸 GORM 简单查询。
- 复杂 SQL 使用 `db/mapper` + `db/sqlquery`，不在 repository 实现中手写 `Raw` SQL。
- 所有 DB 访问使用 `tx.GetTx(ctx, r.db)`，再调用 `query.Use(...)` 或 `sqlquery.<Mapper>(...)`。
- 不在 repository struct 中长期持有预先绑定 `r.db` 的 `*query.Query`。
- repository 内部需要原子性的多步数据库操作使用 `tx.Transaction`，事务内传递回调 ctx。
- `FindByID` 未命中返回 `optional.None[T]()`。
- 除外部触发器和数据库模型外，查询结果必须使用 `optional.Option[T]`、slice 或 map 包装；除 `error` 外不要直接返回 nil。
- repository 可使用 cache-aside 模式读写缓存。
- Save/Delete/批量变更成功后失效相关缓存 key。
- 缓存读写或删除失败只记录 debug 日志，不覆盖数据库主流程错误。
- 如需日志，使用 `log/slog`；有 ctx 时用 `slog.*Context(ctx, ...)`。
- 每个函数至少要有符合 Go 规范的函数注释。
- repository interface 和实现方法除 New 函数外，需要按统一格式描述非 `ctx` 参数和非 `error` 返回。
- 复杂逻辑代码段要加注释说明意图或关键步骤。
- MQ producer 属于 infras/repository，由 domain 接口抽象后注入使用。
