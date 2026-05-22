# 零值 Mutex 是有效的

零值 `sync.Mutex` 和 `sync.RWMutex` 是有效的。所以指向 mutex 的指针基本是不必要的。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
mu := new(sync.Mutex)
mu.Lock()
```

</td><td>

```go
var mu sync.Mutex
mu.Lock()
```

</td></tr>
</tbody></table>

如果你使用结构体指针，mutex 应该作为结构体的非指针字段。即使该结构体不被导出，也不要直接把 mutex 嵌入到结构体中。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type SMap struct {
  sync.Mutex

  data map[string]string
}

func NewSMap() *SMap {
  return &SMap{
    data: make(map[string]string),
  }
}

func (m *SMap) Get(k string) string {
  m.Lock()
  defer m.Unlock()

  return m.data[k]
}
```

</td><td>

```go
type SMap struct {
  mu sync.Mutex

  data map[string]string
}

func NewSMap() *SMap {
  return &SMap{
    data: make(map[string]string),
  }
}

func (m *SMap) Get(k string) string {
  m.mu.Lock()
  defer m.mu.Unlock()

  return m.data[k]
}
```

</td></tr>
<tr><td>

`Mutex` 字段，`Lock` 和 `Unlock` 方法是 `SMap` 导出的 API 中不刻意说明的一部分。

 </td><td>

mutex 及其方法是 `SMap` 的实现细节，对其调用者不可见。

 </td></tr>
 </tbody></table>
