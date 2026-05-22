# Model 模板

数据模型放在 `module/<module>/infras/repository/db/model`。

```go
package model

import commonGorm "<project>/pkg/gorm"

type <Module> struct {
    commonGorm.SnowflakeModel
    Code string `gorm:"column:code;type:varchar(64);not null;uniqueIndex"`
    Name string `gorm:"column:name;type:varchar(64);not null"`
}

func (<Module>) TableName() string {
    return "<table_name>"
}
```

## 规则

- model 只表达持久化结构、表名、字段标签和必要模型方法。
- 不把领域行为写到 model。
- 修改 model 后运行 `mage gorm`。
- GORM/gen 生成文件落在 `infras/repository/db/query`，不要手改。
