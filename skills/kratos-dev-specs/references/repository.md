# Repository 模板

Repository interface 位于 `domain`，实现位于 `infras/repository`。构造器由 Fx 提供，并以 `fx.Annotate(..., fx.As(new(domain.<Module>Repository)))` 绑定到 domain interface。所有数据库调用必须先通过 common 的 `tx.GetTx(ctx, r.db)` 获取带上下文事务的 DB 实例；多步原子操作通过 `tx.Transaction` 包裹。

## 领域层 Repository 接口

domain/repository.go:
```go
package domain

import (
	"context"

	"<project>-common/pkg/optional"
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
	//   - entity: 待保存的<模块>
	Save(ctx context.Context, entity *<Module>) error
	// Delete 删除<模块>
	//
	// param:
	//   - entity: 待删除的<模块>
	Delete(ctx context.Context, entity *<Module>) error
	// FindByID 根据ID获取<模块>
	//
	// param:
	//   - id: <模块>ID
	//
	// return:
	//   - <模块>实体，未命中时返回 optional.None
	FindByID(ctx context.Context, id int64) (optional.Option[*<Module>], error)
	// Search<Module> 分页搜索<模块>
	//
	// param:
	//   - page: 分页参数
	//   - search: 搜索条件
	//
	// return:
	//   - 分页搜索结果
	Search<Module>(ctx context.Context, page pagination.PageRequest, search <Module>Search) (pagination.PageResponse[*<Module>], error)
}
```

## 基础设施层 Repository 实现

infras/repository/repository.go:
```go
package repository

import (
	"context"
	"errors"
	"fmt"
	"log/slog"

	"<project>-common/pkg/optional"
	"<project>-common/pkg/tx"
	"<project>/module/<module>/domain"
	"<project>/module/<module>/infras/repository/cache"
	"<project>/module/<module>/infras/repository/db/model"
	"<project>/module/<module>/infras/repository/db/query"

	"gorm.io/gorm"
	"gorm.io/gorm/clause"
)

type <Module>Repository struct {
	db      *gorm.DB
	cache   *cache.<Module>Cache
	factory *domain.<Module>Factory
}

// New<Module>Repository 创建 <module> 仓储实现
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

// FindByID 根据 ID 查询 <module>
//
// param:
//   - id: <module> ID
//
// return:
//   - 实体，未命中时为 optional.None
func (r *<Module>Repository) FindByID(ctx context.Context, id int64) (optional.Option[*domain.<Module>], error) {
	cached, err := r.cache.Get(ctx, id)
	if err != nil {
		slog.DebugContext(ctx, "缓存读取失败", "id", id, "error", err)
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

	// 仓储从数据库还原实体
	entity := r.factory.Reconstruct(row.ID, row.Code, row.Name)
	err := r.cache.Put(ctx, entity)
	if err != nil {
		slog.DebugContext(ctx, "缓存写入失败", "id", id, "error", err)
	}
	return optional.Some(entity), nil
}

// SaveBatch 批量保存实体，支持冲突忽略/更新的幂等写入
//
// param:
//   - entities: 待保存的<module>实体列表
func (r *<Module>Repository) SaveBatch(ctx context.Context, entities []*domain.<Module>) error {
	if len(entities) == 0 {
		return nil
	}
	models := make([]*model.<Module>, 0, len(entities))
	for _, e := range entities {
		models = append(models, toModel(e))
	}

	return tx.Transaction(ctx, r.db, func(txCtx context.Context) error {
		q := query.Use(tx.GetTx(txCtx, r.db)).<Module>
		if err := q.WithContext(txCtx).Clauses(clause.OnConflict{
			Columns:   []clause.Column{{Name: "id"}},
			DoNothing: true,
		}).CreateInBatches(models, len(models)); err != nil {
			return fmt.Errorf("批量保存 <module>: %w", err)
		}
		return nil
	})
}
```

## 复杂 SQL 与报表查询

复杂 SQL 在 `db/mapper` 声明，并通过 `gorm.io/cli/gorm` 生成 `db/sqlquery`。SQL 入参、结果结构使用 `DTO` 后缀；调用生成 mapper 时传入 `tx.GetTx(ctx, r.db)`，自动复用外层事务。

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

// <Module>SearchDTO 描述 <module> 搜索条件
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

仓储层调用：

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

## 规范要点

- **双 ORM 生成引擎分工**：常规单表 CRUD 使用 `gorm.io/gen`（`db/query`）；多表关联、报表及复杂 SQL 在 `db/mapper` 声明接口并由 `gorm.io/cli/gorm` 生成 `db/sqlquery`。严禁直接调用 `db.Raw`，严禁在仓储手写拼接 SQL。
- **事务上下文穿透**：不在 repository 中保存预先绑定的 query 实例；每次操作均通过 `tx.GetTx(ctx, r.db)` 动态构建 query 或 mapper。
- **幂等写入**：数据批处理、同步操作优先使用 `clause.OnConflict`，防范重试导致主键或唯一索引冲突。
- **缓存策略**：采用 Cache-Aside 模式；写操作成功后失效缓存；缓存异常仅记录 `slog.DebugContext`，绝不阻断数据库主流程。
- **生成物不可手改**：必须通过编辑 `db/model` 与 `db/mapper`，再执行项目 Mage 生成命令（如 `mage gorm`）更新代码。
- **参数/返回值注释**：repository interface 与实现（`New` 除外）保留参数/返回值注释；MQ producer 属于 infras，并由 domain interface 抽象。
