# 使用 time 处理时间

时间处理很复杂。关于时间的错误假设通常包括以下几点。

1. 一天有 24 小时
2. 一小时有 60 分钟
3. 一周有七天
4. 一年 365 天
5. [还有更多](https://infiniteundo.com/post/25326999628/falsehoods-programmers-believe-about-time)

例如，*1* 表示在一个时间点上加上 24 小时并不总是产生一个新的日历日。

因此，在处理时间时始终使用 [`"time"`] 包，因为它有助于以更安全、更准确的方式处理这些不正确的假设。

[`"time"`]: https://pkg.go.dev/time/

## 使用 `time.Time` 表达瞬时时间

在处理时间的瞬间时使用 [`time.Time`]，在比较、添加或减去时间时使用 `time.Time` 中的方法。

[`time.Time`]: https://pkg.go.dev/time/#Time

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func isActive(now, start, stop int) bool {
  return start <= now && now < stop
}
```

</td><td>

```go
func isActive(now, start, stop time.Time) bool {
  return (start.Before(now) || start.Equal(now)) && now.Before(stop)
}
```

</td></tr>
</tbody></table>

## 使用 `time.Duration` 表达时间段

在处理时间段时使用 [`time.Duration`] .

[`time.Duration`]: https://pkg.go.dev/time/#Duration

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func poll(delay int) {
  for {
    // ...
    time.Sleep(time.Duration(delay) * time.Millisecond)
  }
}
poll(10) // 是几秒钟还是几毫秒？
```

</td><td>

```go
func poll(delay time.Duration) {
  for {
    // ...
    time.Sleep(delay)
  }
}
poll(10*time.Second)
```

</td></tr>
</tbody></table>

回到第一个例子，在一个时间瞬间加上 24 小时，我们用于添加时间的方法取决于意图。如果我们想要下一个日历日 (当前天的下一天) 的同一个时间点，我们应该使用 [`Time.AddDate`]。但是，如果我们想保证某一时刻比前一时刻晚 24 小时，我们应该使用 [`Time.Add`]。

[`Time.AddDate`]: https://pkg.go.dev/time/#Time.AddDate
[`Time.Add`]: https://pkg.go.dev/time/#Time.Add

```go
newDay := t.AddDate(0 /* years */, 0 /* months */, 1 /* days */)
maybeNewDay := t.Add(24 * time.Hour)
```

## 对外部系统使用 `time.Time` 和 `time.Duration`

尽可能在与外部系统的交互中使用 `time.Duration` 和 `time.Time` 例如 :

- Command-line 标志：[`flag`] 通过 [`time.ParseDuration`] 支持 `time.Duration`
- JSON: [`encoding/json`] 通过其 [`UnmarshalJSON` method] 方法支持将 `time.Time` 编码为 [RFC 3339] 字符串
- SQL: [`database/sql`] 支持将 `DATETIME` 或 `TIMESTAMP` 列转换为 `time.Time`，如果底层驱动程序支持则返回
- YAML: [`gopkg.in/yaml.v2`] 支持将 `time.Time` 作为 [RFC 3339] 字符串，并通过 [`time.ParseDuration`] 支持 `time.Duration`。

  [`flag`]: https://pkg.go.dev/flag/
  [`time.ParseDuration`]: https://pkg.go.dev/time/#ParseDuration
  [`encoding/json`]: https://pkg.go.dev/encoding/json/
  [RFC 3339]: https://tools.ietf.org/html/rfc3339
  [`UnmarshalJSON` method]: https://pkg.go.dev/time/#Time.UnmarshalJSON
  [`database/sql`]: https://pkg.go.dev/database/sql/
  [`gopkg.in/yaml.v2`]: https://pkg.go.dev/gopkg.in/yaml.v2

当不能在这些交互中使用 `time.Duration` 时，请使用 `int` 或 `float64`，并在字段名称中包含单位。

例如，由于 `encoding/json` 不支持 `time.Duration`，因此该单位包含在字段的名称中。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// {"interval": 2}
type Config struct {
  Interval int `json:"interval"`
}
```

</td><td>

```go
// {"intervalMillis": 2000}
type Config struct {
  IntervalMillis int `json:"intervalMillis"`
}
```

</td></tr>
</tbody></table>

当在这些交互中不能使用 `time.Time` 时，除非达成一致，否则使用 `string` 和 [RFC 3339] 中定义的格式时间戳。默认情况下，[`Time.UnmarshalText`] 使用此格式，并可通过 [`time.RFC3339`] 在 `Time.Format` 和 `time.Parse` 中使用。

[`Time.UnmarshalText`]: https://pkg.go.dev/time/#Time.UnmarshalText
[`time.RFC3339`]: https://pkg.go.dev/time/#RFC3339

尽管这在实践中并不成问题，但请记住，`"time"` 包不支持解析闰秒时间戳（[8728]），也不在计算中考虑闰秒（[15190]）。如果您比较两个时间瞬间，则差异将不包括这两个瞬间之间可能发生的闰秒。

[8728]: https://go.dev/issues/8728
[15190]: https://go.dev/issues/15190

<!-- TODO: section on String methods for enums -->
