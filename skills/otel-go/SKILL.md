---
name: otel-go
description: "Go OpenTelemetry 可观测性专家 - 分布式追踪、指标收集、日志关联、上下文传播、采样策略、Baggage 传递。适用：微服务链路追踪、APM 监控、多信号（traces/metrics/logs）统一采集、vendor-neutral 遥测方案、gRPC/HTTP 自动 instrumentation。不适用：单体应用仅需简单日志排障、性能极敏感热路径（SDK 有微量开销）、已深度绑定特定 APM 厂商 SDK 且无迁移计划。触发词：opentelemetry, otel, tracing, span, metric, 可观测性, 链路追踪, 指标, 采样, propagation, baggage, jaeger, tempo, otlp"
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

# Go OpenTelemetry 专家

使用 Go OpenTelemetry SDK 开发可观测性功能：$ARGUMENTS

---

## 1. SDK 初始化

### 完整初始化（推荐）

```go
func InitOTel(ctx context.Context, cfg OTelConfig) (shutdown func(context.Context) error, err error)
```

- 创建 Resource（service.name, version, environment）
- 初始化 TracerProvider + MeterProvider + LoggerProvider
- 设置 Propagator（TraceContext + Baggage）
- 返回统一 shutdown 函数，确保数据导出

> 完整实现见 [references/examples.md](references/examples.md#1-sdk-初始化)

### Provider 创建签名

```go
func newTracerProvider(ctx context.Context, res *resource.Resource, endpoint string) (*trace.TracerProvider, error)
func newMeterProvider(ctx context.Context, res *resource.Resource, endpoint string) (*metric.MeterProvider, error)
func newLoggerProvider(ctx context.Context, res *resource.Resource, endpoint string) (*log.LoggerProvider, error)
```

---

## 2. 分布式追踪

### 创建 Span

```go
tracer := otel.Tracer("order-service")
ctx, span := tracer.Start(ctx, "ProcessOrder",
    trace.WithAttributes(attribute.String("order.id", orderID)),
    trace.WithSpanKind(trace.SpanKindInternal),
)
defer span.End()
```

- `span.AddEvent()` 添加事件
- `span.RecordError()` 记录错误
- `span.SetStatus(codes.Error, msg)` 设置状态

### Span 属性规范

使用语义属性 `semconv`：
- **HTTP**: `HTTPRequestMethodKey`, `HTTPResponseStatusCode`, `URLFull`, `HTTPRoute`
- **DB**: `DBSystemKey`, `DBNamespace`, `DBOperationName`
- **消息队列**: `MessagingSystem`, `MessagingDestinationName`, `MessagingOperationTypePublish`

> 完整示例见 [references/examples.md](references/examples.md#2-分布式追踪)

---

## 3. 指标收集

### 指标类型

| 类型 | 函数 | 用途 |
|------|------|------|
| Counter | `meter.Int64Counter()` | 只增计数（请求总数） |
| Histogram | `meter.Float64Histogram()` | 分布（延迟） |
| UpDownCounter | `meter.Int64UpDownCounter()` | 可增减（活跃连接） |
| ObservableGauge | `meter.Int64ObservableGauge()` | 异步当前值（缓存大小） |

### 记录指标

```go
orderCounter.Add(ctx, 1, metric.WithAttributes(
    attribute.String("order.type", order.Type),
    attribute.String("status", statusFromError(err)),
))
orderDuration.Record(ctx, duration, metric.WithAttributes(...))
```

> 完整实现见 [references/examples.md](references/examples.md#3-指标收集)

---

## 4. 日志关联

### slog + OTel Bridge

```go
handler := otelslog.NewHandler("my-service",
    otelslog.WithLoggerProvider(global.GetLoggerProvider()),
)
logger := slog.New(handler)
logger.InfoContext(ctx, "processing order", slog.String("order_id", orderID))
```

### 手动注入 Trace 信息

```go
spanCtx := trace.SpanContextFromContext(ctx)
if spanCtx.IsValid() {
    // 添加 trace_id, span_id 到日志
}
```

> 完整实现见 [references/examples.md](references/examples.md#4-日志关联)

---

## 5. 上下文传播

### 自动 Instrumentation

| 组件 | 包 | 用法 |
|------|-----|------|
| HTTP 客户端 | `otelhttp.NewTransport()` | 自动注入 trace headers |
| HTTP 服务端 | `otelhttp.NewHandler()` | 自动提取 trace headers |
| gRPC 客户端 | `otelgrpc.NewClientHandler()` | `grpc.WithStatsHandler()` |
| gRPC 服务端 | `otelgrpc.NewServerHandler()` | `grpc.StatsHandler()` |

### 手动传播

```go
// 注入/提取 HTTP 请求
otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))
otel.GetTextMapPropagator().Extract(ctx, propagation.HeaderCarrier(req.Header))

// 注入/提取 Map
otel.GetTextMapPropagator().Inject(ctx, propagation.MapCarrier(m))
otel.GetTextMapPropagator().Extract(ctx, propagation.MapCarrier(m))
```

---

## 6. 采样策略

### 内置采样器

- `trace.AlwaysSample()` — 全部采样
- `trace.NeverSample()` — 不采样
- `trace.TraceIDRatioBased(0.1)` — 10% 比例采样
- `trace.ParentBased(...)` — 父级决定（推荐）

### 自定义采样器

实现 `trace.Sampler` 接口：`ShouldSample()` + `Description()`

> 完整 ErrorSampler 示例见 [references/examples.md](references/examples.md#6-采样策略)

---

## 7. 健康检查过滤

```go
otelhttp.NewHandler(handler, "my-service",
    otelhttp.WithFilter(func(r *http.Request) bool {
        return r.URL.Path != "/health" && r.URL.Path != "/ready"
    }),
)
```

---

## 8. Baggage 传递业务数据

```go
// 设置
member, _ := baggage.NewMember("tenant_id", tenantID)
bag, _ := baggage.New(member)
ctx = baggage.ContextWithBaggage(ctx, bag)

// 读取
bag := baggage.FromContext(ctx)
tenantID := bag.Member("tenant_id").Value()
```

---

## 最佳实践

### 初始化
- 在 main() 开头初始化 OTel
- 使用 defer shutdown(ctx) 确保数据导出
- 统一配置 Resource 属性

### 追踪
- Span 名称使用 `<动词><名词>` 格式
- 使用语义属性（semconv）
- 只在必要时记录 span.RecordError()
- 健康检查端点不追踪

### 指标
- 使用标准单位 (ms, s, {item})
- 控制 cardinality（属性值数量）
- 优先使用 Histogram 记录延迟

### 上下文
- 始终传递 context.Context
- 使用自动 instrumentation 库
- Baggage 只传递小量元数据

---

## 检查清单

- [ ] 配置 Resource（service.name, environment）？
- [ ] 设置合适的采样率？
- [ ] 使用 ParentBased 采样器？
- [ ] HTTP/gRPC 使用自动 instrumentation？
- [ ] 日志关联 Trace ID？
- [ ] 健康检查端点排除追踪？
- [ ] 指标属性 cardinality 可控？
- [ ] 优雅关闭 OTel Provider？

---

## 参考资料

- [完整代码示例](references/examples.md) — SDK 初始化、追踪、指标、日志、传播、采样完整实现
