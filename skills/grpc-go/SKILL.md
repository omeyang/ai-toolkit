---
name: grpc-go
description: "Go gRPC 专家 - 服务定义(Proto)、Unary/Stream RPC、拦截器(日志/认证/恢复)、错误处理与错误码映射、元数据传播、负载均衡、健康检查、优雅关闭。适用：微服务通信、API 网关、内部服务间调用、流式数据传输。不适用：浏览器直连(用 gRPC-Web 或 REST)；简单 CRUD API(REST 更轻量)；消息队列场景(用 Kafka/Pulsar)。触发词：grpc, gRPC, protobuf, proto, 拦截器, interceptor, streaming, 流式, metadata, 元数据, service-config"
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

# Go gRPC 专家

使用 Go gRPC 开发高性能 RPC 服务：$ARGUMENTS

---

## 1. 服务定义

### Proto 文件

使用版本化包名，请求/响应独立 message，使用 google.protobuf 标准类型。

```protobuf
syntax = "proto3";
package user.v1;
option go_package = "github.com/example/api/user/v1;userv1";

service UserService {
    rpc GetUser(GetUserRequest) returns (GetUserResponse);          // Unary
    rpc ListUsers(ListUsersRequest) returns (stream User);          // Server streaming
    rpc BatchCreateUsers(stream CreateUserRequest) returns (BatchCreateUsersResponse); // Client streaming
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);      // Bidirectional
}
```

> 完整 Proto 定义和代码生成命令见 [references/examples.md](references/examples.md#proto-文件完整定义)

### 生成代码

```bash
protoc --go_out=. --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
       api/user/v1/*.proto
```

---

## 2. 服务端实现

### 核心结构

```go
type UserServer struct {
    userv1.UnimplementedUserServiceServer
    repo UserRepository
}
```

### RPC 类型签名

| RPC 类型 | 签名 |
|---------|------|
| Unary | `GetUser(ctx, *GetUserRequest) (*GetUserResponse, error)` |
| Server Stream | `ListUsers(*ListUsersRequest, UserService_ListUsersServer) error` |
| Client Stream | `BatchCreateUsers(UserService_BatchCreateUsersServer) error` |
| Bidi Stream | `Chat(UserService_ChatServer) error` |

> 完整四种 RPC 实现见 [references/examples.md](references/examples.md#服务端实现)

### 启动服务

```go
server := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        LoggingInterceptor(),
        RecoveryInterceptor(),
        TracingInterceptor(),
    ),
    grpc.ChainStreamInterceptor(
        StreamLoggingInterceptor(),
        StreamRecoveryInterceptor(),
    ),
)
userv1.RegisterUserServiceServer(server, NewUserServer(repo))
reflection.Register(server) // 调试用
```

---

## 3. 客户端实现

### 创建客户端

使用 `grpc.NewClient`，配置负载均衡、超时、重试策略。

```go
func NewUserClient(target string) (userv1.UserServiceClient, func(), error)
```

> 完整客户端创建和调用示例见 [references/examples.md](references/examples.md#客户端实现)

---

## 4. 拦截器

### Unary 服务端拦截器

- **LoggingInterceptor**: 记录方法、状态码、耗时
- **RecoveryInterceptor**: 捕获 panic，返回 Internal 错误
- **AuthInterceptor**: 从 metadata 提取 token，跳过公开方法

### Stream 服务端拦截器

- **StreamLoggingInterceptor**: 流式 RPC 日志
- **wrappedStream**: 包装 ServerStream 以拦截 Recv/Send

### 客户端拦截器

- **ClientLoggingInterceptor**: 客户端调用日志
- **ClientMetadataInterceptor**: 自动注入 request-id、tenant-id

> 完整拦截器实现见 [references/examples.md](references/examples.md#拦截器实现)

---

## 5. 错误处理

### 错误码映射

```go
var domainToGRPC = map[error]codes.Code{
    ErrNotFound:       codes.NotFound,
    ErrAlreadyExists:  codes.AlreadyExists,
    ErrInvalidInput:   codes.InvalidArgument,
    ErrUnauthorized:   codes.Unauthenticated,
    ErrForbidden:      codes.PermissionDenied,
    ErrConflict:       codes.Aborted,
    ErrRateLimited:    codes.ResourceExhausted,
    ErrServiceUnavail: codes.Unavailable,
}

func ToGRPCError(err error) error
```

### 错误详情

使用 `errdetails.BadRequest` 传递字段级验证错误。

```go
func ValidationError(field, description string) error
```

> 完整错误处理和客户端解析见 [references/examples.md](references/examples.md#错误处理实现)

---

## 6. 元数据传播

| 方向 | API |
|------|-----|
| 服务端提取 | `metadata.FromIncomingContext(ctx)` |
| 服务端注入响应头 | `grpc.SendHeader(ctx, md)` / `grpc.SetTrailer(ctx, md)` |
| 客户端发送 | `metadata.NewOutgoingContext(ctx, md)` |
| 客户端接收 | `grpc.Header(&header)` / `grpc.Trailer(&trailer)` |

> 完整元数据操作示例见 [references/examples.md](references/examples.md#元数据传播实现)

---

## 7. 健康检查

```go
healthServer := health.NewServer()
grpc_health_v1.RegisterHealthServer(server, healthServer)
healthServer.SetServingStatus("user.v1.UserService", grpc_health_v1.HealthCheckResponse_SERVING)
```

动态更新状态时使用 context 控制 goroutine 生命周期，避免泄漏。

> 完整健康检查实现见 [references/examples.md](references/examples.md#健康检查实现)

---

## 8. 优雅关闭

```go
// GracefulStop 等待现有请求完成 + 超时强制关闭
server.GracefulStop()       // 优雅
server.Stop()               // 强制（超时后备用）
```

> 完整优雅关闭模式见 [references/examples.md](references/examples.md#优雅关闭实现)

---

## 9. 服务配置（重试、超时）

通过 `DefaultServiceConfig` JSON 配置每方法超时和重试策略。

```go
conn, err := grpc.NewClient(target,
    grpc.WithDefaultServiceConfig(serviceConfig),
)
```

> 完整配置 JSON 示例见 [references/examples.md](references/examples.md#服务配置实现)

---

## 最佳实践

### Proto 设计
- 使用版本化包名 (v1, v2)
- 请求/响应使用独立 message
- 使用 google.protobuf 标准类型
- 添加字段注释

### 错误处理
- 使用标准 gRPC 错误码
- 错误详情用于调试
- 客户端区分可重试/不可重试错误

### 性能
- 复用客户端连接
- 使用连接池
- 合理设置超时
- 大数据用流式 RPC

### 可观测性
- 统一拦截器记录日志/追踪
- 传播请求 ID 和租户 ID
- 实现健康检查

---

## 检查清单

- [ ] Proto 文件版本化？
- [ ] 实现健康检查服务？
- [ ] 配置合理超时？
- [ ] 日志/追踪拦截器？
- [ ] 错误码映射完整？
- [ ] 客户端启用重试？
- [ ] 优雅关闭？
- [ ] 元数据传播？

---

## 参考资料

- [references/examples.md](references/examples.md) - 完整代码实现（Proto 定义、四种 RPC、拦截器、错误处理、元数据、健康检查、优雅关闭、服务配置）
