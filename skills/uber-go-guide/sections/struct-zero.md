# 对零值结构使用 `var`

如果在声明中省略了结构的所有字段，请使用 `var` 声明结构。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
user := User{}
```

</td><td>

```go
var user User
```

</td></tr>
</tbody></table>

这将零值结构与那些具有类似于为 [初始化 Maps](#初始化-maps) 创建的，区别于非零值字段的结构区分开来，
我们倾向于[声明一个空切片](https://go.dev/wiki/CodeReviewComments#declaring-empty-slices)
