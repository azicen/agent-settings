# Model 模板

数据库模型位于 `module/<module>/infras/repository/db/model`，必须嵌入 common 的 `BaseModel` 或 `SnowflakeModel`。选用类型、soft-delete 字段、表名和 import 以已有模型为准；model 只表达持久化结构，不承载领域规则。

```go
package model

import commonmodel "<project>-common/pkg/gorm/model"

// <Module> 表示 <module> 的持久化记录。
type <Module> struct {
	commonmodel.SnowflakeModel
	Code string `gorm:"column:code;type:varchar(64);not null;uniqueIndex"`
	Name string `gorm:"column:name;type:varchar(64);not null"`
}

// TableName 返回 <Module> 的表名。
func (<Module>) TableName() string {
	return "<table_name>"
}
```

- model 包只包含列、tag、关联、表名和必要的 persistence 转换；领域行为、HTTP/proto 类型和缓存逻辑不进入 model。
- string 长度、nullability、索引和唯一约束必须与 migration 和实际数据库语义一致；外部输入校验不替代数据库约束。
- 修改 model 或 mapper 后运行项目实际生成命令并检查 `db/query`/`db/sqlquery`；不手改 `*.gen.go` 或任何生成物。
- repository 将 model 转换为 domain entity，避免 GORM model 泄漏到 domain 或 interfaces。
