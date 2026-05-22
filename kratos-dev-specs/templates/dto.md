# SQL mapper/DTO 模板

复杂 SQL 不再默认使用独立 DTO 目录，也不要在 repository 实现中手写 SQL。优先在 `module/<module>/infras/repository/db/mapper` 中定义 SQL 入参/结果 DTO、mapper interface 和注释 SQL，再通过 `mage gorm` 生成 `db/sqlquery`。

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

## 规则

- SQL 入参和 SQL 结果映射需要额外结构体时，统一使用 `DTO` 后缀命名，例如 `<Module>SearchDTO`、`<Module>StatisticsDTO`。
- mapper interface 使用 `type <Module>Mapper[T any] interface { ... }`。
- 生成代码放在 `db/sqlquery`，不要手改。
- repository 实现通过 `sqlquery.<Module>Mapper[*model.<Model>](tx.GetTx(ctx, r.db))` 调用生成方法，确保外层 `tx.Transaction` 可被复用。
- 返回 domain 层前，转换为 domain entity、value object 或 repository method 约定的返回类型。
