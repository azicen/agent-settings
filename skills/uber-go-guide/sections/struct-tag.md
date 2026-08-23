# 在序列化结构中使用字段标记

任何序列化到 JSON、YAML、，
或其他支持基于标记的字段命名的格式应使用相关标记进行注释。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type Stock struct {
  Price int
  Name  string
}
bytes, err := json.Marshal(Stock{
  Price: 137,
  Name:  "UBER",
})
```

</td><td>

```go
type Stock struct {
  Price int    `json:"price"`
  Name  string `json:"name"`
  // Safe to rename Name to Symbol.
}
bytes, err := json.Marshal(Stock{
  Price: 137,
  Name:  "UBER",
})
```

</td></tr>
</tbody></table>

理论上：
结构的序列化形式是不同系统之间的契约。
对序列化表单结构（包括字段名）的更改会破坏此约定。在标记中指定字段名使约定明确，
它还可以通过重构或重命名字段来防止意外违反约定。
