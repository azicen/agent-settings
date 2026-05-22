# Comments 模板

## 函数注释

每个函数至少要有一个符合 Go 规范的函数注释，注释以函数名开头并描述用途。

```go
// CheckEquipment 校验设备编码/名称是否已存在
func (c *EquipmentController) CheckEquipment(ctx context.Context, req *v1.CheckEquipmentRequest) (*v1.CheckEquipmentResponse, error) {
    return &v1.CheckEquipmentResponse{}, nil
}
```

## 完整格式

```go
// <函数名称> <函数简单描述>
// <函数详细描述>
//
// param:
//   - <参数1名称>: <参数1描述>
//   - <参数2名称>: <参数2描述>
//
// return:
//   - <返回1名称>: <返回1描述>
//   - <返回2名称>: <返回2描述>
```

函数详细描述可以省略；算法相关函数必须有详细描述。

## param 规则

`param` 描述省略 `ctx`。如果没有需要描述的参数，则完全省略 `param` 段。

```go
// Save 保存地理位置
//
// param:
//   - geolocation: 待保存的地理位置实体
func (r *GeolocationRepositoryImpl) Save(ctx context.Context, geolocation *domain.Geolocation) error {
    return nil
}
```

## return 规则

`return` 描述省略 `error`、`emptypb.Empty` 等空响应。如果没有需要描述的返回，则完全省略 `return` 段。

```go
// FindByID 根据ID获取地理位置
//
// param:
//   - id: 地理位置ID
//
// return:
//   - result: 地理位置实体，未命中时返回 optional.None
func (r *GeolocationRepositoryImpl) FindByID(ctx context.Context, id int64) (optional.Option[*domain.Geolocation], error) {
    return optional.None[*domain.Geolocation](), nil
}
```

`param:` 和 `return:` 后必须紧跟具体内容，不能出现空行；条目缩进必须严格为注释前缀后的一个空格再接 `-`，即 `//   - name: 描述`。

```go
// DeleteGeolocation 删除地理位置
//
// param:
//   - id: 地理位置ID
func (r *GeolocationRepositoryImpl) DeleteGeolocation(ctx context.Context, id int64) (*emptypb.Empty, error) {
    return &emptypb.Empty{}, nil
}
```

```go
// SplitName 拆分名称
//
// param:
//   - fullName: 完整名称
//
// return:
//   - firstName: 名
//   - lastName: 姓
func SplitName(fullName string) (firstName string, lastName string) {
    return "", ""
}
```

## Repository 注释

Repository 仓储接口和仓储实现的方法，除 New 函数外，除了简单函数描述，还需要按统一格式描述非 `ctx` 参数和非 `error`、非空响应返回。

```go
type GeolocationRepository interface {
    // Save 保存地理位置
    //
    // param:
    //   - geolocation: 待保存的地理位置实体
    Save(ctx context.Context, geolocation *Geolocation) error

    // FindByID 根据ID获取地理位置
    //
    // param:
    //   - id: 地理位置ID
    //
    // return:
    //   - result: 地理位置实体，未命中时返回 optional.None
    FindByID(ctx context.Context, id int64) (optional.Option[*Geolocation], error)
}
```

New 函数只需要普通函数注释，不需要逐个描述参数。

```go
// NewGeolocationRepository 创建地理位置仓储实现
func NewGeolocationRepository(...) *GeolocationRepositoryImpl {
    return &GeolocationRepositoryImpl{}
}
```

## 复杂逻辑代码段注释

复杂逻辑代码段要写注释说明意图或关键步骤，例如批量查询、缓存缺失计算、事务内多步骤处理、复杂 SQL、幂等处理。

```go
// 计算缓存未命中的 ID，只查询缺失数据
missingIDs := make([]int64, 0)
for _, id := range ids {
    if _, ok := cacheMap[id]; !ok {
        missingIDs = append(missingIDs, id)
    }
}
```

## 规则

- 注释说明业务意图和参数含义，不写流水账式代码翻译。
- 函数注释以函数名开头，满足 Go doc 规范。
- `param` 省略 `ctx`；没有参数则完全省略 `param`。
- 如果末尾只有空占用结构体或error，则可以省略return描述。如果没有需要描述的返回，则完全省略 `return` 段。
- `param:` 和 `return:` 后必须紧跟具体内容，不能出现空行；`-` 应该对齐 `param:` 的 `r` 或 `return:` 的 `t`，即在 `param` 或 `return` 同级缩进 2 个空格，呈现为 `//   - name: 描述`。
- `return` 不论数量，均使用列表格式 `//   - name: 描述`。
- 各种注释末尾省略没有意义的句号。
- 简单直观的语句不需要段落注释；复杂逻辑才需要。
