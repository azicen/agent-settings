# Linting

比任何 "blessed" linter 集更重要的是，lint 在一个代码库中始终保持一致。

我们建议至少使用以下 linters，因为我认为它们有助于发现最常见的问题，并在不需要规定的情况下为代码质量建立一个高标准：

- [errcheck] 以确保错误得到处理
- [goimports] 格式化代码和管理 imports
- [golint] 指出常见的文体错误
- [govet] 分析代码中的常见错误
- [staticcheck] 各种静态分析检查

  [errcheck]: https://github.com/kisielk/errcheck
  [goimports]: https://pkg.go.dev/golang.org/x/tools/cmd/goimports
  [golint]: https://github.com/golang/lint
  [govet]: https://golang.org/cmd/vet/
  [staticcheck]: https://staticcheck.io/


## Lint Runners

我们推荐 [golangci-lint] 作为 go-to lint 的运行程序，这主要是因为它在较大的代码库中的性能以及能够同时配置和使用许多规范。这个 repo 有一个示例配置文件 [.golangci.yml] 和推荐的 linter 设置。

golangci-lint 有 [various-linters] 可供使用。建议将上述 linters 作为基本 set，我们鼓励团队添加对他们的项目有意义的任何附加 linters。

[golangci-lint]: https://github.com/golangci/golangci-lint
[.golangci.yml]: https://github.com/uber-go/guide/blob/master/.golangci.yml
[various-linters]: https://golangci-lint.run/usage/linters/
