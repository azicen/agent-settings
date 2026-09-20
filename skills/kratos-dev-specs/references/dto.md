# SQL mapper/DTO 模板

复杂 SQL、多表联查与报表查询在 `infras/repository/db/mapper` 声明 DTO、mapper interface 和注释 SQL，由 `gorm.io/cli/gorm` 编译生成 `db/sqlquery`。repository 不得手写裸 SQL 拼装或调用 `db.Raw`，生成物不可手改。

## Mapper 声明

infras/repository/db/mapper/mapper.go:
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

// <Module>SearchDTO 描述检索条件与分页参数
type <Module>SearchDTO struct {
	// IDs 待检索的 ID 列表
	IDs []int64
	// Limit 限制返回行数
	Limit int
	// Offset 偏移量
	Offset int
}

// <Module>Mapper 定义复杂 SQL 查询接口，泛型 T 支持任意持久化行模型或聚合 DTO
type <Module>Mapper interface {
	// Search<Module> 复杂条件查询
	//
	// SELECT id, code, name
	// FROM <table_name>
	// WHERE deleted = FALSE AND id IN @search.IDs
	// LIMIT @search.Limit OFFSET @search.Offset
	Search<Module>(ctx context.Context, search <Module>SearchDTO) ([]*model.<Module>, error)
}
```

## 仓储层调用（`infras/repository/repository.go`）

```go
rows, err := sqlquery.<Module>Mapper(tx.GetTx(ctx, r.db)).
	Search<Module>(ctx, mapper.<Module>SearchDTO{
		IDs:    search.IDs,
		Limit:  int(page.PageSize),
		Offset: int((page.Page - 1) * page.PageSize),
	})
if err != nil {
	return pagination.PageResponse[*domain.<Module>]{}, err
}
```

## 规范要点

- **DTO 命名**：SQL 入参条件和自定义多表聚合结果映射结构统一使用 `DTO` 后缀；
- **参数化占位符**：SQL 注释中使用 `@search.FieldName` 绑定 DTO 字段，避免 SQL 注入；
- **事务无缝继承**：每次调用必须通过 `tx.GetTx(ctx, r.db)` 构建 mapper 实例，确保外层开启的 `tx.Transaction` 能够无缝穿透并自动复用事务；
- **领域隔离**：仓储层在将结果返回给 domain 或外部之前，必须将 model/DTO 转换为 domain 实体或值对象，杜绝数据库细节侵入领域层；
- **生成命令**：通过运行项目的 Mage 构建任务（如 `mage gorm`）触发代码生成，禁止手动编辑 `db/sqlquery/*.go`。
