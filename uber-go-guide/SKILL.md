---
name: uber-go-guide
description: Uber Go Guide 中文规范。Use when writing, reviewing, or refactoring Go code to apply Uber Go style guidelines, Go error handling, concurrency, performance, naming, imports, tests, and lint rules.
---

# Uber Go Guide 中文规范

使用本 skill 处理 Go 代码编写、重构、代码审查、风格统一、错误处理、并发生命周期、性能优化、测试表格、函数选项和 lint 规范相关任务。

## 使用原则

1. 不要一次性读取全部章节；先根据任务类型从下方目录选择最相关的 `sections/*.md`。
2. 处理具体 Go 代码前，优先读取对应专题章节；跨专题问题只读取必要的多个文件。
3. 代码审查时以行为正确性、可维护性和一致性为主，不为了风格机械改动无关代码。
4. 如果项目已有更具体的团队规范，优先遵循项目规范；本 guide 用作通用 Go 代码基线。
5. 涉及争议时，引用具体章节文件名说明依据。

## 快速选择

- 接口与类型：`interface-pointer.md`、`interface-compliance.md`、`interface-receiver.md`、`type-assert.md`。
- 错误处理：`error-type.md`、`error-wrap.md`、`error-name.md`、`error-once.md`。
- 并发与退出：`goroutine-forget.md`、`goroutine-exit.md`、`goroutine-init.md`、`channel-size.md`、`exit-main.md`、`exit-once.md`。
- 全局状态与初始化：`global-mut.md`、`global-decl.md`、`global-name.md`、`init.md`、`atomic.md`。
- 性能：`performance.md`、`strconv.md`、`string-byte-slice.md`、`container-capacity.md`。
- 命名、导入与布局：`package-name.md`、`function-name.md`、`import-group.md`、`import-alias.md`、`function-order.md`。
- 结构体、map、slice：`struct-embed.md`、`struct-field-key.md`、`struct-field-zero.md`、`struct-zero.md`、`struct-pointer.md`、`map-init.md`、`slice-nil.md`。
- 测试与 API 模式：`test-table.md`、`functional-option.md`。
- lint：`lint.md`。

## 章节目录

- `intro.md`: 介绍
- 指导原则
  - `interface-pointer.md`: 指向 interface 的指针
  - `interface-compliance.md`: Interface 合理性验证
  - `interface-receiver.md`: 接收器 (receiver) 与接口
  - `mutex-zero-value.md`: 零值 Mutex 是有效的
  - `container-copy.md`: 在边界处拷贝 Slices 和 Maps
  - `defer-clean.md`: 使用 defer 释放资源
  - `channel-size.md`: Channel 的 size 要么是 1，要么是无缓冲的
  - `enum-start.md`: 枚举从 1 开始
  - `time.md`: 使用 time 处理时间
  - Errors
    - `error-type.md`: 错误类型
    - `error-wrap.md`: 错误包装
    - `error-name.md`: 错误命名
    - `error-once.md`: 一次处理错误
  - `type-assert.md`: 处理断言失败
  - `panic.md`: 不要使用 panic
  - `atomic.md`: 使用 go.uber.org/atomic
  - `global-mut.md`: 避免可变全局变量
  - `embed-public.md`: 避免在公共结构中嵌入类型
  - `builtin-name.md`: 避免使用内置名称
  - `init.md`: 避免使用 `init()`
  - `exit-main.md`: 主函数退出方式 (Exit)
    - `exit-once.md`: 一次性退出
  - `struct-tag.md`: 在序列化结构中使用字段标记
  - `goroutine-forget.md`: 不要一劳永逸地使用 goroutine
    - `goroutine-exit.md`: 等待 goroutines 退出
    - `goroutine-init.md`: 不要在 `init()` 使用 goroutines
- `performance.md`: 性能
  - `strconv.md`: 优先使用 strconv 而不是 fmt
  - `string-byte-slice.md`: 避免字符串到字节的转换
  - `container-capacity.md`: 指定容器容量
- 规范
  - `line-length.md`: 避免过长的行
  - `consistency.md`: 一致性
  - `decl-group.md`: 相似的声明放在一组
  - `import-group.md`: import 分组
  - `package-name.md`: 包名
  - `function-name.md`: 函数名
  - `import-alias.md`: 导入别名
  - `function-order.md`: 函数分组与顺序
  - `nest-less.md`: 减少嵌套
  - `else-unnecessary.md`: 不必要的 else
  - `global-decl.md`: 顶层变量声明
  - `global-name.md`: 对于未导出的顶层常量和变量，使用_作为前缀
  - `struct-embed.md`: 结构体中的嵌入
  - `var-decl.md`: 本地变量声明
  - `slice-nil.md`: nil 是一个有效的 slice
  - `var-scope.md`: 缩小变量作用域
  - `param-naked.md`: 避免参数语义不明确 (Avoid Naked Parameters)
  - `string-escape.md`: 使用原始字符串字面值，避免转义
  - 初始化结构体
    - `struct-field-key.md`: 使用字段名初始化结构
    - `struct-field-zero.md`: 省略结构中的零值字段
    - `struct-zero.md`: 对零值结构使用 `var`
    - `struct-pointer.md`: 初始化 Struct 引用
  - `map-init.md`: 初始化 Maps
  - `printf-const.md`: 字符串 string format
  - `printf-name.md`: 命名 Printf 样式的函数
- 编程模式
  - `test-table.md`: 表驱动测试
  - `functional-option.md`: 功能选项
- `lint.md`: Linting

## 来源

内容参考 `https://github.com/xxjwxc/uber_go_guide_cn`，章节拆分保持源仓库 `src/SUMMARY.md` 的文件名和顺序，正文来自顶层 `README.md` 的中文翻译。
