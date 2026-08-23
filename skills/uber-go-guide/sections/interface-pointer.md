# 指向 interface 的指针

您几乎不需要指向接口类型的指针。您应该将接口作为值进行传递，在这样的传递过程中，实质上传递的底层数据仍然可以是指针。

接口实质上在底层用两个字段表示：

1. 一个指向某些特定类型信息的指针。您可以将其视为"type"。
2. 数据指针。如果存储的数据是指针，则直接存储。如果存储的数据是一个值，则存储指向该值的指针。

如果希望接口方法修改基础数据，则必须使用指针传递 (将对象指针赋值给接口变量)。

```go
type F interface {
  f()
}

type S1 struct{}

func (s S1) f() {}

type S2 struct{}

func (s *S2) f() {}

// f1.f() 无法修改底层数据
// f2.f() 可以修改底层数据，给接口变量 f2 赋值时使用的是对象指针
var f1 F = S1{}
var f2 F = &S2{}
```
永远不要使用指向 interface 的指针，这个是没有意义的。在 go 语言中，接口本身就是引用类型，换句话说，接口类型本身就是一个指针。对于我的需求，其实 test 的参数只要是 myinterface 就可以了，只需要在传值的时候，传*mystruct 类型（也只能传*mystruct 类型）
```go
type myinterface interface{
	print()
}
func test(value *myinterface){
	//someting to do ...
}

type mystruct struct {
	i int
}
//实现接口
func (this *mystruct) print(){
	fmt.Println(this.i)
	this.i=1
}
func main(){
	m := &mystruct{0}
	test(m)  // 错误
	test(*m) // 错误
}
```
