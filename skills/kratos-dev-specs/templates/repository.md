# Repository 模板

Repository interface 位于 `domain`，实现位于 `infras/repository`。构造器由 Fx 提供，并以 `fx.Annotate(..., fx.As(new(domain.<Module>Repository)))` 绑定到 domain interface。所有数据库调用先取得 common `tx.GetTx(ctx, r.db)`；只有需要原子性的多步持久化才创建 `tx.Transaction`。

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


```go
package repository

import (
	"context"
	"errors"
	"log/slog"

	"<project>-common/pkg/optional"
	"<project>-common/pkg/tx"
	"<project>/module/<module>/domain"
	"<project>/module/<module>/infras/repository/cache"
	"<project>/module/<module>/infras/repository/db/model"
	"<project>/module/<module>/infras/repository/db/query"

	"gorm.io/gorm"
)

type <Module>Repository struct {
	db      *gorm.DB
	cache   *cache.<Module>Cache
	factory *domain.<Module>Factory
}

// New<Module>Repository 创建 <module> 仓储实现。
func New<Module>Repository(
	db *gorm.DB,
	cache *cache.<Module>Cache,
	factory *domain.<Module>Factory,
) *<Module>Repository {
	return &<Module>Repository{
		db:      db,
		cache:   cache,
		factory: factory,
	}
}

// FindByID 根据 ID 查询 <module>。
//
// param:
//   - id: <module> ID
//
// return:
//   - result: 实体，未命中时为 optional.None
func (r *<Module>Repository) FindByID(ctx context.Context, id int64) (optional.Option[*domain.<Module>], error) {
	cached, err := r.cache.Get(ctx, id)
	if err != nil {
		slog.DebugContext(ctx, "cache get failed", "id", id, "error", err)
	} else if cached.IsSome() {
		return cached, nil
	}

	q := query.Use(tx.GetTx(ctx, r.db)).<Module>
	row, err := q.WithContext(ctx).Where(q.ID.Eq(id)).First()
	if errors.Is(err, gorm.ErrRecordNotFound) {
		return optional.None[*domain.<Module>](), nil
	}
	if err != nil {
		return optional.None[*domain.<Module>](), err
	}

	entity := r.factory.Reconstruct(row.ID)
	if err := r.cache.Put(ctx, entity); err != nil {
		slog.DebugContext(ctx, "cache put failed", "id", id, "error", err)
	}
	return optional.Some(entity), nil
}

// SavePair 原子保存关联记录。
//
// param:
//   - entity: 要保存的 <module> 实体
func (r *<Module>Repository) SavePair(ctx context.Context, entity *domain.<Module>) error {
	return tx.Transaction(ctx, r.db, func(txCtx context.Context) error {
		q := query.Use(tx.GetTx(txCtx, r.db)).<Module>
		return q.WithContext(txCtx).Save(&model.<Module>{ID: entity.ID})
	})
}
```

复杂 SQL 在 `db/mapper` 声明，并通过项目实际生成命令生成 `db/sqlquery`。SQL 入参、结果结构使用 `DTO` 后缀；调用生成 mapper 时传入 `tx.GetTx(ctx, r.db)`，从而复用外层事务。

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

// <Module>SearchDTO 描述 <module> 搜索条件。
type <Module>SearchDTO struct {
	IDs    []int64
	Limit  int
	Offset int
}

type <Module>Mapper[T any] interface {
	/*
	SELECT id, code, name
	FROM <table_name>
	WHERE deleted = 0 AND id IN @search.IDs
	LIMIT @search.Limit OFFSET @search.Offset
	*/
	Search<Module>(ctx context.Context, search <Module>SearchDTO) ([]T, error)
}
```

```go
rows, err := sqlquery.<Module>Mapper[*model.<Module>](tx.GetTx(ctx, r.db)).
	Search<Module>(ctx, mapper.<Module>SearchDTO{
		IDs:    search.IDs,
		Limit:  limit,
		Offset: offset,
	})
if err != nil {
	return nil, err
}
```

- 常规 CRUD 使用 GORM/gen query；复杂 SQL 使用 mapper + 生成的 sqlquery，禁止 `Raw` 和在 repository 手写 SQL。
- 不在 repository 中缓存预绑定数据库的 query；每次请求从 `tx.GetTx` 创建 query/mapper。
- Save、Delete 和批量变更成功后失效相关 key；缓存错误只写诊断日志，不能覆盖数据库主流程。
- 生成前编辑 model/mapper，生成后不手改 `db/query`、`db/sqlquery` 或其他生成物。
- repository interface 与实现（`New` 除外）保留参数/返回值注释；MQ producer 属于 infras，并由 domain interface 抽象。
