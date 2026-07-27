# Proto 和业务错误模板

先从源 proto 仓库发现 `buf.yaml`、`buf.gen.yaml` 与 CI；当前设计使用 **buf v2** 配置：`buf.yaml` 的 `version: v2` 定义 `modules`、`deps`、`lint` 和 `breaking`，`buf.gen.yaml` 的 `version: v2` 定义本地生成插件。不要把 v1 工作区、旧版模板或猜测的生成参数复制进项目。

外部 API 和 error proto 只能在源 proto 仓库维护。源仓执行并在 CI 执行：

```sh
buf lint
buf generate
```

生成的 SDK 由 CI 同步到 `<project>-proto-go`；业务仓更新发布的 SDK 依赖后编译验证。业务仓**只能**生成自身 `pkg/config` 的配置 proto，不能在本仓生成或修改外部 API、错误 proto 或 SDK。目标仓的实际 CI workflow、发布分支及同步路径必须从仓库文件发现。

## RPC 模板

每个 RPC 必须显式声明 `required_permission`、`authenticated` 或 `public` 之一；同时存在时按 `required_permission` > `authenticated` > `public` 解析。导入实际 authz proto，并使用其完整限定扩展名。不要规定 `auth.enabled` 的开关含义。

```proto
syntax = "proto3";

package api.<scope>.<module>.v1;

import "google/api/annotations.proto";
import "google/api/field_behavior.proto";
import "validate/validate.proto";
import "api/common/v1/authz.proto";

option go_package = "<project>-proto-go/api/<scope>/v1;v1";

service <Module>Service {
  rpc Get<Module>(Get<Module>Request) returns (<Module>Info) {
    option (api.common.authz.v1.required_permission) = "<module>:read";
    option (google.api.http) = {
      get: "/api/v1/<modules>/{<module>_id}"
    };
  }
}

message Get<Module>Request {
  int64 <module>_id = 1 [
    (google.api.field_behavior) = REQUIRED,
    (validate.rules).int64.gt = 0
  ];
}
```

HTTP API 复用 `google/api/annotations.proto` 的 `google.api.http`，必填字段复用 `google/api/field_behavior.proto`，输入约束复用 `validate/validate.proto`。只在协议语义需要时添加字段约束，避免以业务代码取代可声明的协议校验。

## Error reason 完整模板

每个 error reason 文件包含 `syntax`、`package`、`errors` import、`go_package`、enum 默认 code，并给**每一个** enum value 指定 `(errors.code)`：

```proto
syntax = "proto3";

package api.<scope>.<module>.v1;

import "errors/errors.proto";

option go_package = "<project>-proto-go/api/<scope>/v1;v1";

enum <Module>ErrorReason {
  option (errors.default_code) = 500;

  <MODULE>_UNSPECIFIED = 0 [(errors.code) = 500];
  <MODULE>_NOT_FOUND = 1 [(errors.code) = 404];
  <MODULE>_DUPLICATE = 2 [(errors.code) = 409];
  <MODULE>_INVALID_ARGUMENT = 3 [(errors.code) = 400];
}
```

业务代码仅调用已发布 SDK 生成的 helper，例如 `v1.Error<Module>NotFound("资源不存在")`；不得手写业务错误或绕过 error proto。修改源 proto 后等待 SDK 同步 CI 完成，再更新业务依赖并编译。
