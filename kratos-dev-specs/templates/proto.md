# Proto 和 error-proto 协作模板

## API proto

API 定义一般在执行项目根目录上一级的某个 `-proto` 后缀目录，例如：

```text
../package-proto
```

常见结构：

```text
api/<scope>/v1/<module>.proto
api/<scope>/v1/<module>_error_reason.proto
errors/errors.proto
```

业务项目引用编译后的 proto SDK，通常已在 `go.mod` 中引入，模块名一般以 `-proto-go` 结尾，例如：

```text
github.com/namespace/package-proto-go
```

## 查找和记录流程

1. 需要查看 API 定义时，先在执行项目根目录的上一级目录查找 `*-proto`。
2. 如果找不到或不确定 proto 项目目录，询问用户。
3. 即使找到了疑似 proto 项目，也要询问用户确认。
4. 用户确认后，在执行项目根目录下创建或更新 `.agents/package.local.json`，使用字段 `proto` 记录 proto 项目目录。
5. 检查业务项目 `go.mod` 是否引入了模块名后缀为 `-proto-go` 的 proto SDK。
6. 如果没有发现 proto SDK，询问用户对应的 proto SDK Go 模块。
7. 用户确认后，在 `.agents/package.local.json` 中使用字段 `sdk.go` 记录 proto SDK Go 模块。

`.agents/package.local.json` 示例：

```json
{
  "proto": "/home/username/project/package-proto",
  "sdk.go": "github.com/namespace/package-proto-go"
}
```

优先沿用项目已有 `.agents/package.local.json` 的结构；如果文件不存在，使用扁平 key 结构。

## 修改和更新流程

1. 不要在业务项目中构建 proto 文件。
2. 需要修改 proto 时，先获得用户修改权限。
3. 修改 proto 后通知用户提交 proto 仓库，CI/CD 会自动构建。
4. 用户确认 CI/CD 完成后，在业务项目运行 `go get <proto-go-module>@latest` 或指定版本。

## error-proto

业务错误必须优先使用 error-proto 定义并生成的 helper。

```proto
syntax = "proto3";

package api.<scope>.<module>.v1;

import "errors/errors.proto";

option go_package = "<proto-go-module>/api/<scope>/v1;v1";

enum <Module>ErrorReason {
  option (errors.default_code) = 500;

  <MODULE>_UNSPECIFIED = 0 [(errors.code) = 500];
  <MODULE>_NOT_FOUND = 1 [(errors.code) = 404];
  <MODULE>_DUPLICATE_CODE = 2 [(errors.code) = 409];
  <MODULE>_INVALID_ARGUMENT = 3 [(errors.code) = 400];
}
```

业务代码使用生成 helper：

```go
return nil, v1.Error<Module>NotFound("资源不存在")
```

## 规则

- 不绕过 error-proto 手写业务错误。
- 新增错误 reason 时设置 `(errors.code)`。
- 修改 error-proto 后同样等待 CI/CD 生成，再 `go get` 更新业务项目依赖。
- 找到疑似 proto 项目也要先询问用户确认，再记录到 `.agents/package.local.json`。
