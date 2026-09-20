# Model 模板

数据库模型位于 `module/<module>/infras/repository/db/model`，必须嵌入 common 的 `BaseModel`、 `UUIDModel` 或 `SnowflakeModel`。model 只表达持久化物理结构与 GORM 映射标签，不承载领域规则。

```go
package model

import (
	"<project>-common/pkg/gorm"
)

// <Module> 表示 <module> 的数据库持久化记录
type <Module> struct {
	// SnowflakeModel 注入雪花算法主键 ID (int64)
	gorm.SnowflakeModel
	// Code 业务唯一标识编码
	Code string `gorm:"column:code;type:varchar(64);not null;uniqueIndex"`
	// Name 业务名称
	Name string `gorm:"column:name;type:varchar(64);not null"`
	// BaseModel 注入 Ctime (autoCreateTime)、Mtime (autoUpdateTime) 及 Deleted (DeletedBool, softDelete:flag)
	gorm.BaseModel
}

// TableName 返回 <Module> 对应的物理表名
func (<Module>) TableName() string {
	return "<table_name>"
}
```

## 规范要点

- **公共底座嵌入**：
  - `UUIDModel`：包含 `ID string` 主键字段（`primaryKey;column:id;type:uuid`），适用于 UUID 主键；
  - `SnowflakeModel`：包含 `ID int64` 主键字段（`primaryKey;column:id;autoIncrement:false`），适用于雪花 ID 主键；
  - `BaseModel`：包含 `Ctime`（创建时间）、`Mtime`（修改时间）以及 `Deleted`（`DeletedBool`，`softDelete:flag` 软删除字段）；
- **物理与领域隔离**：model 包只包含数据库字段、GORM tag、索引和表名；领域方法、校验逻辑严禁进入 model；
- **强类型与约束对齐**：列长度（如 `varchar(64)`）、非空约束（`not null`）、索引（`index`、`uniqueIndex`）必须与 Goose 数据库迁移脚本完全一致；
- **生成后防篡改**：更新 model 后运行 `mage gorm`（自动执行 `gorm.io/gen` 重新生成 `db/query`）；严禁手改 `*.gen.go` 或任何生成代码。
