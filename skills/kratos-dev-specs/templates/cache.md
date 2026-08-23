# Cache 模板

缓存位于 `module/<module>/infras/repository/cache`，使用 common 项目已确认版本的 `jetcache.NewT`、`local.NewTinyLFU` 与 `jetcache.WithStatsDisabled(true)`。实体读取返回 common `optional.Option[*Entity]`，repository 采用 cache-aside。

```go
package cache

import (
	"context"
	"time"

	"<project>-common/pkg/optional"
	"<project>/module/<module>/domain"

	jetcache "github.com/mgtv-tech/jetcache-go"
	"github.com/mgtv-tech/jetcache-go/local"
)

const (
	cachePrefix  = "<module>:entity"
	cacheTTL     = 30 * time.Minute
	cacheMaxCost = 1 << 20
)

type <Module>Cache struct {
	cache *jetcache.T[int64, *domain.<Module>]
}

// New<Module>Cache 创建 <module> 本地缓存
func New<Module>Cache() (*<Module>Cache, error) {
	return &<Module>Cache{
		cache: jetcache.NewT[int64, *domain.<Module>](newLocalCache()),
	}, nil
}

// newLocalCache 按当前 common 项目的本地缓存模式创建 jetcache.Cache
func newLocalCache() jetcache.Cache {
	return jetcache.New(
		jetcache.WithLocal(local.NewTinyLFU(cacheMaxCost, cacheTTL)),
		jetcache.WithStatsDisabled(true),
	)
}

// Get 根据ID获取<模块>缓存，未命中返回 None
//
// param:
//   - id: <模块>ID
//
// return:
//   - result: 命中返回 Some，未命中返回 None
func (c *<Module>Cache) Get(ctx context.Context, id int64) (optional.Option[*domain.<Module>], error) {
	result, err := c.cache.MGetWithErr(ctx, cachePrefix, []int64{id}, nil)
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
	return c.cache.Set(ctx, cachePrefix, entity.ID, entity)
}

// Delete 删除<模块>缓存
//
// param:
//   - id: <模块>ID
func (c *<Module>Cache) Delete(ctx context.Context, id int64) error {
	return c.cache.Delete(ctx, cachePrefix, id)
}
```

当前 common 项目的已验证模式是 `jetcache.NewT[K, V](newLocalCache())`，其中 `newLocalCache() jetcache.Cache` 返回带 `WithLocal` 和 `WithStatsDisabled(true)` 的 `jetcache.New(...)`。`Put`、`Delete` 的精确参数签名仍必须以项目锁定的 jetcache 版本和现有调用为准；若版本 API 不同，按当前依赖修正调用而非猜测。

```go
func Register() fx.Option {
	return fx.Provide(cache.New<Module>Cache)
}
```

- key 带模块前缀；读取命中直接返回，未命中查询数据库并写缓存。
- 写入、删除和批量变更在数据库成功后失效相关 key。
- 缓存读写失败只记录 `slog` 诊断信息，不覆盖数据库结果或业务错误。
