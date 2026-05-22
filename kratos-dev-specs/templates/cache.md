# Cache 模板

缓存组件放在 `module/<module>/infras/repository/cache`。必须使用 `github.com/mgtv-tech/jetcache-go` 封装缓存；外部只依赖本 cache 组件。

```go
package cache

import (
    "context"
    "time"

    "<project>/module/<module>/domain"

    jetcache "github.com/mgtv-tech/jetcache-go"
    "github.com/mgtv-tech/jetcache-go/local"
    "github.com/moznion/go-optional"
)

const (
    defaultTTL     = 30 * time.Minute
    defaultMaxCost = 1 << 28
    cacheKey       = "<module>"
)

// <Module>Cache <模块>缓存
type <Module>Cache struct {
    cache *jetcache.T[int64, *domain.<Module>]
}

// New<Module>Cache 创建<模块>缓存
func New<Module>Cache() (*<Module>Cache, error) {
    return &<Module>Cache{
        cache: jetcache.NewT[int64, *domain.<Module>](newLocalCache()),
    }, nil
}

// Get 根据ID获取<模块>缓存，未命中返回 None
//
// param:
//   - id: <模块>ID
//
// return:
//   - result: 命中返回 Some，未命中返回 None
func (c *<Module>Cache) Get(ctx context.Context, id int64) (optional.Option[*domain.<Module>], error) {
    result, err := c.cache.MGetWithErr(ctx, cacheKey, []int64{id}, nil)
    if err != nil {
        return optional.None[*domain.<Module>](), err
    }

    entity, _ := result[id]
    return optional.PtrFromNillable(entity), nil
}

// GetAll 批量获取<模块>缓存
//
// param:
//   - ids: <模块>ID列表
//
// return:
//   - result: 命中项映射，所有路径返回非 nil 容器
func (c *<Module>Cache) GetAll(ctx context.Context, ids []int64) (map[int64]*domain.<Module>, error) {
    result, err := c.cache.MGetWithErr(ctx, cacheKey, ids, nil)
    if err != nil {
        return make(map[int64]*domain.<Module>), err
    }

    entities := make(map[int64]*domain.<Module>, len(result))
    for id, entity := range result {
        if entity != nil {
            entities[id] = entity
        }
    }
    return entities, nil
}

// Put 将<模块>实体写入缓存
//
// param:
//   - entity: 待缓存的<模块>实体
func (c *<Module>Cache) Put(ctx context.Context, entity *domain.<Module>) error {
    if entity == nil {
        return nil
    }
    return c.cache.Set(ctx, cacheKey, entity.ID, entity)
}

// Delete 删除<模块>缓存
//
// param:
//   - id: <模块>ID
func (c *<Module>Cache) Delete(ctx context.Context, id int64) error {
    return c.cache.Delete(ctx, cacheKey, id)
}

// newLocalCache 创建 jetcache-go 本地缓存
func newLocalCache() jetcache.Cache {
    return jetcache.New(
        jetcache.WithLocal(local.NewTinyLFU(defaultMaxCost, defaultTTL)),
        jetcache.WithStatsDisabled(true),
    )
}
```

## 规则

- 复杂缓存可以在同一个 cache struct 中维护多个 `*jetcache.T[K,V]` 字段。
- 每个索引用独立 cache key 常量，例如实体 ID、full code、设备 ID、设备 code。
- repository 使用 cache-aside 模式。
- 缓存错误不覆盖数据库主流程错误；由 repository 按需记录 debug 日志。
- DI 中先 provide cache，再 provide repository。
