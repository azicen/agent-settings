# SQL mapper/DTO 模板

复杂 SQL 在 `infras/repository/db/mapper` 声明 DTO、mapper interface 和注释 SQL，再由项目实际生成命令生成 `db/sqlquery`。repository 不写 `Raw` SQL，生成物不可手改。

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

// <Module>SearchDTO 描述搜索条件。
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
		Limit:  int(page.PageSize),
		Offset: int((page.Page - 1) * page.PageSize),
	})
if err != nil {
	return pagination.PageResponse[*domain.<Module>]{}, err
}
```


- SQL 入参和结果映射结构统一使用 `DTO` 后缀；mapper 泛型签名、生成命令与 package 名称以项目已有代码为准。
- 每次调用从 `tx.GetTx(ctx, r.db)` 构造生成 mapper，确保 `tx.Transaction` 内的查询复用事务。
- 将 model/DTO 转成 domain entity 后再返回；不要让生成类型穿透到 domain。
