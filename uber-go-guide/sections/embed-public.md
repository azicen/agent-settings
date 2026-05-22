# 避免在公共结构中嵌入类型

这些嵌入的类型泄漏实现细节、禁止类型演化和模糊的文档。

假设您使用共享的 `AbstractList` 实现了多种列表类型，请避免在具体的列表实现中嵌入 `AbstractList`。
相反，只需手动将方法写入具体的列表，该列表将委托给抽象列表。

```go
type AbstractList struct {}
// 添加将实体添加到列表中。
func (l *AbstractList) Add(e Entity) {
  // ...
}
// 移除从列表中移除实体。
func (l *AbstractList) Remove(e Entity) {
  // ...
}
```

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// ConcreteList 是一个实体列表。
type ConcreteList struct {
  *AbstractList
}
```

</td><td>

```go
// ConcreteList 是一个实体列表。
type ConcreteList struct {
  list *AbstractList
}
// 添加将实体添加到列表中。
func (l *ConcreteList) Add(e Entity) {
  l.list.Add(e)
}
// 移除从列表中移除实体。
func (l *ConcreteList) Remove(e Entity) {
  l.list.Remove(e)
}
```

</td></tr>
</tbody></table>

Go 允许 [类型嵌入](https://golang.org/doc/effective_go.html#embedding) 作为继承和组合之间的折衷。外部类型获取嵌入类型的方法的隐式副本。默认情况下，这些方法委托给嵌入实例的同一方法。

结构还获得与类型同名的字段。
所以，如果嵌入的类型是 public，那么字段是 public。为了保持向后兼容性，外部类型的每个未来版本都必须保留嵌入类型。

很少需要嵌入类型。
这是一种方便，可以帮助您避免编写冗长的委托方法。

即使嵌入兼容的抽象列表 *interface*，而不是结构体，这将为开发人员提供更大的灵活性来改变未来，但仍然泄露了具体列表使用抽象实现的细节。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// AbstractList 是各种实体列表的通用实现。
type AbstractList interface {
  Add(Entity)
  Remove(Entity)
}
// ConcreteList 是一个实体列表。
type ConcreteList struct {
  AbstractList
}
```

</td><td>

```go
// ConcreteList 是一个实体列表。
type ConcreteList struct {
  list AbstractList
}
// 添加将实体添加到列表中。
func (l *ConcreteList) Add(e Entity) {
  l.list.Add(e)
}
// 移除从列表中移除实体。
func (l *ConcreteList) Remove(e Entity) {
  l.list.Remove(e)
}
```

</td></tr>
</tbody></table>

无论是使用嵌入结构还是嵌入接口，都会限制类型的演化。

- 向嵌入接口添加方法是一个破坏性的改变。
- 从嵌入结构体删除方法是一个破坏性改变。
- 删除嵌入类型是一个破坏性的改变。
- 即使使用满足相同接口的类型替换嵌入类型，也是一个破坏性的改变。

尽管编写这些委托方法是乏味的，但是额外的工作隐藏了实现细节，留下了更多的更改机会，还消除了在文档中发现完整列表接口的间接性操作。
