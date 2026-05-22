# 指定容器容量

尽可能指定容器容量，以便为容器预先分配内存。这将在添加元素时最小化后续分配（通过复制和调整容器大小）。

## 指定 Map 容量提示

在尽可能的情况下，在使用 `make()` 初始化的时候提供容量信息

```go
make(map[T1]T2, hint)
```

向`make()`提供容量提示会在初始化时尝试调整 map 的大小，这将减少在将元素添加到 map 时为 map 重新分配内存。


注意，与 slices 不同。map 容量提示并不保证完全的、预先的分配，而是用于估计所需的 hashmap bucket 的数量。
因此，在将元素添加到 map 时，甚至在指定 map 容量时，仍可能发生分配。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
m := make(map[string]os.FileInfo)

files, _ := os.ReadDir("./files")
for _, f := range files {
    m[f.Name()] = f
}
```

</td><td>

```go

files, _ := os.ReadDir("./files")

m := make(map[string]os.FileInfo, len(files))
for _, f := range files {
    m[f.Name()] = f
}
```

</td></tr>
<tr><td>

`m` 是在没有大小提示的情况下创建的；在运行时可能会有更多分配。

</td><td>

`m` 是有大小提示创建的；在运行时可能会有更少的分配。

</td></tr>
</tbody></table>

## 指定切片容量

在尽可能的情况下，在使用`make()`初始化切片时提供容量信息，特别是在追加切片时。

```go
make([]T, length, capacity)
```

与 maps 不同，slice capacity 不是一个提示：编译器将为提供给`make()`的 slice 的容量分配足够的内存，
这意味着后续的`append()`操作将导致零分配（直到 slice 的长度与容量匹配，在此之后，任何 append 都可能调整大小以容纳其他元素）。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
for n := 0; n < b.N; n++ {
  data := make([]int, 0)
  for k := 0; k < size; k++{
    data = append(data, k)
  }
}
```

</td><td>

```go
for n := 0; n < b.N; n++ {
  data := make([]int, 0, size)
  for k := 0; k < size; k++{
    data = append(data, k)
  }
}
```

</td></tr>
<tr><td>

```
BenchmarkBad-4    100000000    2.48s
```

</td><td>

```
BenchmarkGood-4   100000000    0.21s
```

</td></tr>
</tbody></table>
