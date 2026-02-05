---
name: resilience-go
description: "Go 韧性模式专家 - 熔断器(sony/gobreaker)、重试策略(avast/retry-go, 指数退避)、限流(令牌桶/滑动窗口/按Key)、超时控制(Context分层)、舱壁隔离(信号量/Worker Pool)、降级处理(缓存降级/默认值)、组合弹性调用。适用：微服务容错、外部服务调用、高可用系统、分布式系统韧性设计。不适用：单体应用内部函数调用、无外部依赖的纯计算逻辑、已有成熟框架(如 Istio)处理韧性的场景。触发词：熔断, circuit breaker, 重试, retry, 限流, rate limit, 超时, timeout, 降级, fallback, 舱壁, bulkhead, 韧性, resilience"
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

# Go 韧性模式专家

使用 Go 实现微服务韧性模式：$ARGUMENTS

---

## 1. 熔断器（Circuit Breaker）

### 使用 sony/gobreaker

```go
func NewCircuitBreaker(name string) *gobreaker.CircuitBreaker[any] {
    return gobreaker.NewCircuitBreaker[any](gobreaker.Settings{
        Name:        name,
        MaxRequests: 3,                    // 半开状态允许的请求数
        Interval:    10 * time.Second,     // 统计窗口
        Timeout:     30 * time.Second,     // 熔断后恢复时间
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
            return counts.Requests >= 10 && failureRatio >= 0.5
        },
        OnStateChange: func(name string, from, to gobreaker.State) {
            slog.Info("circuit breaker state change",
                slog.String("name", name),
                slog.String("from", from.String()),
                slog.String("to", to.String()),
            )
        },
        IsSuccessful: func(err error) bool {
            if err == nil { return true }
            var bizErr *BusinessError
            return errors.As(err, &bizErr)
        },
    })
}
```

### 泛型调用封装

```go
func CallWithBreaker[T any](ctx context.Context, cb *gobreaker.CircuitBreaker[T], fn func() (T, error)) (T, error) {
    return cb.Execute(func() (T, error) { return fn() })
}
```

> 自定义策略（连续失败/错误率/超时率）和 HTTP 客户端集成见 [references/examples.md](references/examples.md#熔断器高级用法)

---

## 2. 重试（Retry）

### 使用 avast/retry-go

```go
func RetryWithBackoff[T any](ctx context.Context, fn func() (T, error)) (T, error) {
    var result T
    err := retry.Do(
        func() error {
            var err error
            result, err = fn()
            return err
        },
        retry.Context(ctx),
        retry.Attempts(3),
        retry.Delay(100*time.Millisecond),
        retry.MaxDelay(5*time.Second),
        retry.DelayType(retry.BackOffDelay),
        retry.MaxJitter(100*time.Millisecond),
        retry.RetryIf(func(err error) bool { return isRetryable(err) }),
    )
    return result, err
}
```

### 可重试错误判断

```go
func isRetryable(err error) bool {
    var netErr net.Error
    if errors.As(err, &netErr) { return netErr.Timeout() || netErr.Temporary() }
    var httpErr *HTTPError
    if errors.As(err, &httpErr) {
        switch httpErr.StatusCode {
        case 429, 502, 503, 504: return true
        }
    }
    if errors.Is(err, ErrNotFound) || errors.Is(err, ErrInvalidInput) { return false }
    return true
}
```

### 永久错误标记

```go
type PermanentError struct { Err error }
func (e *PermanentError) Error() string { return e.Err.Error() }
func (e *PermanentError) Unwrap() error { return e.Err }
func IsPermanent(err error) bool { var pe *PermanentError; return errors.As(err, &pe) }
```

> 自定义重试策略（指数退避/固定延迟/通用执行器）见 [references/examples.md](references/examples.md#自定义重试策略)

---

## 3. 限流（Rate Limiting）

### 令牌桶（golang.org/x/time/rate）

```go
func NewRateLimiter(rps int, burst int) *RateLimiter {
    return &RateLimiter{limiter: rate.NewLimiter(rate.Limit(rps), burst)}
}

// HTTP 中间件
func RateLimitMiddleware(limiter *RateLimiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if !limiter.Allow() {
                http.Error(w, "rate limit exceeded", http.StatusTooManyRequests)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

> 按 Key 限流、滑动窗口实现见 [references/examples.md](references/examples.md#限流高级模式)

---

## 4. 超时控制

### Context 超时

```go
func CallWithTimeout[T any](ctx context.Context, timeout time.Duration, fn func(context.Context) (T, error)) (T, error) {
    ctx, cancel := context.WithTimeout(ctx, timeout)
    defer cancel()
    resultCh := make(chan T, 1)
    errCh := make(chan error, 1)
    go func() {
        result, err := fn(ctx)
        if err != nil { errCh <- err; return }
        resultCh <- result
    }()
    select {
    case <-ctx.Done(): var zero T; return zero, ctx.Err()
    case err := <-errCh: var zero T; return zero, err
    case result := <-resultCh: return result, nil
    }
}
```

### 分层超时

```go
type TimeoutConfig struct {
    Overall  time.Duration // 总超时
    PerCall  time.Duration // 单次调用超时
}

func CallWithLayeredTimeout[T any](ctx context.Context, cfg TimeoutConfig, fn func(context.Context) (T, error)) (T, error) {
    ctx, cancel := context.WithTimeout(ctx, cfg.Overall)
    defer cancel()
    return RetryWithBackoff(ctx, func() (T, error) {
        callCtx, callCancel := context.WithTimeout(ctx, cfg.PerCall)
        defer callCancel()
        return fn(callCtx)
    })
}
```

---

## 5. 舱壁隔离（Bulkhead）

### 信号量隔离

```go
type Bulkhead struct {
    sem     chan struct{}
    timeout time.Duration
}

func NewBulkhead(maxConcurrent int, timeout time.Duration) *Bulkhead {
    return &Bulkhead{sem: make(chan struct{}, maxConcurrent), timeout: timeout}
}

func (b *Bulkhead) Execute(ctx context.Context, fn func() error) error {
    select {
    case b.sem <- struct{}{}: defer func() { <-b.sem }(); return fn()
    case <-time.After(b.timeout): return ErrBulkheadFull
    case <-ctx.Done(): return ctx.Err()
    }
}
```

> 按服务隔离配置、Worker Pool 隔离见 [references/examples.md](references/examples.md#舱壁隔离高级模式)

---

## 6. 降级（Fallback）

### 降级模式

```go
type Fallback[T any] struct {
    primary  func(context.Context) (T, error)
    fallback func(context.Context, error) (T, error)
}

func (f *Fallback[T]) Execute(ctx context.Context) (T, error) {
    result, err := f.primary(ctx)
    if err != nil { return f.fallback(ctx, err) }
    return result, nil
}
```

> 缓存降级（含过期缓存回退）见 [references/examples.md](references/examples.md#缓存降级)

---

## 7. 组合模式

### 完整弹性调用链

限流 -> 舱壁 -> 超时 -> 熔断 -> 重试 -> 降级

```go
type ResilientCall[T any] struct {
    breaker  *gobreaker.CircuitBreaker[T]
    limiter  *RateLimiter
    bulkhead *Bulkhead
    retry    RetryPolicy
    timeout  time.Duration
    fallback func(context.Context, error) (T, error)
}
```

> Builder 模式和完整 Execute 实现见 [references/examples.md](references/examples.md#组合弹性调用)

---

## 最佳实践

### 熔断器
- 根据服务特性调整阈值
- 区分业务错误和系统错误
- 熔断时提供降级响应

### 重试
- 只重试可恢复错误
- 使用指数退避 + 抖动
- 设置最大重试次数
- 幂等操作才能安全重试

### 限流
- 按用户/租户限流
- 返回 429 和 Retry-After 头
- 预留突发容量

### 超时
- 分层设置超时
- 传播 deadline 到下游
- 超时后及时释放资源

### 降级
- 准备降级方案
- 降级数据需标记
- 监控降级频率

---

## 检查清单

- [ ] 外部调用有熔断？
- [ ] 重试策略区分错误类型？
- [ ] API 有限流保护？
- [ ] 所有外部调用有超时？
- [ ] 关键功能有降级方案？
- [ ] 资源隔离（舱壁）？
- [ ] 监控韧性指标？
- [ ] 配置可动态调整？

---

## 参考资料

- [references/examples.md](references/examples.md) - 完整代码实现（高级策略、HTTP 集成、组合模式）
