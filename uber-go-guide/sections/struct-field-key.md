# 使用字段名初始化结构

初始化结构时，几乎应该始终指定字段名。目前由 [`go vet`] 强制执行。

[`go vet`]: https://golang.org/cmd/vet/

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
k := User{"John", "Doe", true}
```

</td><td>

```go
k := User{
    FirstName: "John",
    LastName: "Doe",
    Admin: true,
}
```

</td></tr>
</tbody></table>

例外：当有 3 个或更少的字段时，测试表中的字段名可以省略。

```go
tests := []struct{
  op Operation
  want string
}{
  {Add, "add"},
  {Subtract, "subtract"},
}
```
