---
name: resilience-go
description: Go 韧性模式专家 - 熔断器、重试策略、限流、超时、舱壁隔离、降级处理。使用场景：微服务容错、外部服务调用、高可用系统。
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

# Go 韧性模式专家

使用 Go 实现微服务韧性模式：$ARGUMENTS

---

## 1. 熔断器（Circuit Breaker）

### 使用 sony/gobreaker

```go
import (
    "github.com/sony/gobreaker/v2"
)

func NewCircuitBreaker(name string) *gobreaker.CircuitBreaker[any] {
    return gobreaker.NewCircuitBreaker[any](gobreaker.Settings{
        Name:        name,
        MaxRequests: 3,                    // 半开状态允许的请求数
        Interval:    10 * time.Second,     // 统计窗口
        Timeout:     30 * time.Second,     // 熔断后恢复时间
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            // 失败率 > 50% 且请求数 > 10 时熔断
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
            // 定义什么是成功（可排除特定错误）
            if err == nil {
                return true
            }
            // 业务错误不计入失败
            var bizErr *BusinessError
            return errors.As(err, &bizErr)
        },
    })
}

// 使用
func CallWithBreaker[T any](ctx context.Context, cb *gobreaker.CircuitBreaker[T], fn func() (T, error)) (T, error) {
    return cb.Execute(func() (T, error) {
        return fn()
    })
}

// 示例
func (s *Service) GetUser(ctx context.Context, id string) (*User, error) {
    result, err := CallWithBreaker(ctx, s.userServiceBreaker, func() (*User, error) {
        return s.userClient.Get(ctx, id)
    })

    if errors.Is(err, gobreaker.ErrOpenState) {
        // 熔断器开启，返回降级响应
        return s.getCachedUser(ctx, id)
    }

    return result, err
}
```

### 自定义策略

```go
// 连续失败策略
type ConsecutiveFailurePolicy struct {
    threshold int
}

func (p *ConsecutiveFailurePolicy) ReadyToTrip(counts gobreaker.Counts) bool {
    return counts.ConsecutiveFailures >= uint32(p.threshold)
}

// 错误率+最小请求策略
type ErrorRatePolicy struct {
    minRequests  int
    errorRate    float64
}

func (p *ErrorRatePolicy) ReadyToTrip(counts gobreaker.Counts) bool {
    if counts.Requests < uint32(p.minRequests) {
        return false
    }
    return float64(counts.TotalFailures)/float64(counts.Requests) >= p.errorRate
}

// 超时率策略
type TimeoutPolicy struct {
    minRequests int
    timeoutRate float64
}

func (p *TimeoutPolicy) ReadyToTrip(counts gobreaker.Counts) bool {
    // 需要自定义计数器追踪超时
    return false
}
```

### HTTP 客户端集成

```go
type ResilientHTTPClient struct {
    client  *http.Client
    breaker *gobreaker.CircuitBreaker[*http.Response]
}

func NewResilientHTTPClient() *ResilientHTTPClient {
    return &ResilientHTTPClient{
        client: &http.Client{
            Timeout: 10 * time.Second,
        },
        breaker: gobreaker.NewCircuitBreaker[*http.Response](gobreaker.Settings{
            Name:        "http-client",
            MaxRequests: 3,
            Timeout:     30 * time.Second,
            ReadyToTrip: func(counts gobreaker.Counts) bool {
                return counts.ConsecutiveFailures >= 5
            },
        }),
    }
}

func (c *ResilientHTTPClient) Do(req *http.Request) (*http.Response, error) {
    return c.breaker.Execute(func() (*http.Response, error) {
        resp, err := c.client.Do(req)
        if err != nil {
            return nil, err
        }

        // 5xx 视为失败
        if resp.StatusCode >= 500 {
            resp.Body.Close()
            return nil, fmt.Errorf("server error: %d", resp.StatusCode)
        }

        return resp, nil
    })
}
```

---

## 2. 重试（Retry）

### 使用 avast/retry-go

```go
import (
    "github.com/avast/retry-go/v4"
)

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
        retry.DelayType(retry.BackOffDelay),      // 指数退避
        retry.MaxJitter(100*time.Millisecond),    // 随机抖动
        retry.RetryIf(func(err error) bool {
            // 只重试可恢复错误
            return isRetryable(err)
        }),
        retry.OnRetry(func(n uint, err error) {
            slog.Warn("retry attempt",
                slog.Uint64("attempt", uint64(n)),
                slog.Any("error", err),
            )
        }),
    )

    return result, err
}

func isRetryable(err error) bool {
    // 网络错误
    var netErr net.Error
    if errors.As(err, &netErr) {
        return netErr.Timeout() || netErr.Temporary()
    }

    // 特定 HTTP 状态码
    var httpErr *HTTPError
    if errors.As(err, &httpErr) {
        switch httpErr.StatusCode {
        case 429, 502, 503, 504:
            return true
        }
    }

    // 永久错误
    if errors.Is(err, ErrNotFound) || errors.Is(err, ErrInvalidInput) {
        return false
    }

    return true
}
```

### 自定义重试策略

```go
// 重试策略接口
type RetryPolicy interface {
    ShouldRetry(err error, attempt int) bool
    Delay(attempt int) time.Duration
}

// 指数退避策略
type ExponentialBackoff struct {
    MaxAttempts  int
    InitialDelay time.Duration
    MaxDelay     time.Duration
    Multiplier   float64
    Jitter       float64
}

func (e *ExponentialBackoff) ShouldRetry(err error, attempt int) bool {
    if attempt >= e.MaxAttempts {
        return false
    }
    return isRetryable(err)
}

func (e *ExponentialBackoff) Delay(attempt int) time.Duration {
    delay := float64(e.InitialDelay) * math.Pow(e.Multiplier, float64(attempt))
    if delay > float64(e.MaxDelay) {
        delay = float64(e.MaxDelay)
    }

    // 添加抖动
    if e.Jitter > 0 {
        jitter := delay * e.Jitter * (rand.Float64()*2 - 1)
        delay += jitter
    }

    return time.Duration(delay)
}

// 固定延迟策略
type FixedDelay struct {
    MaxAttempts int
    Delay       time.Duration
}

func (f *FixedDelay) ShouldRetry(err error, attempt int) bool {
    return attempt < f.MaxAttempts && isRetryable(err)
}

func (f *FixedDelay) Delay(attempt int) time.Duration {
    return f.Delay
}

// 通用重试执行器
func Retry[T any](ctx context.Context, policy RetryPolicy, fn func() (T, error)) (T, error) {
    var result T
    var lastErr error

    for attempt := 0; ; attempt++ {
        result, lastErr = fn()
        if lastErr == nil {
            return result, nil
        }

        if !policy.ShouldRetry(lastErr, attempt) {
            break
        }

        delay := policy.Delay(attempt)
        select {
        case <-ctx.Done():
            return result, ctx.Err()
        case <-time.After(delay):
        }
    }

    return result, lastErr
}
```

### 永久错误标记

```go
// 不可重试错误
type PermanentError struct {
    Err error
}

func (e *PermanentError) Error() string {
    return e.Err.Error()
}

func (e *PermanentError) Unwrap() error {
    return e.Err
}

func NewPermanentError(err error) error {
    return &PermanentError{Err: err}
}

func IsPermanent(err error) bool {
    var pe *PermanentError
    return errors.As(err, &pe)
}

// 使用
func ProcessOrder(ctx context.Context, order *Order) error {
    return Retry(ctx, policy, func() error {
        if order.ID == "" {
            return NewPermanentError(ErrInvalidOrder) // 不重试
        }

        return callExternalService(ctx, order)
    })
}
```

---

## 3. 限流（Rate Limiting）

### 令牌桶

```go
import "golang.org/x/time/rate"

type RateLimiter struct {
    limiter *rate.Limiter
}

func NewRateLimiter(rps int, burst int) *RateLimiter {
    return &RateLimiter{
        limiter: rate.NewLimiter(rate.Limit(rps), burst),
    }
}

func (r *RateLimiter) Allow() bool {
    return r.limiter.Allow()
}

func (r *RateLimiter) Wait(ctx context.Context) error {
    return r.limiter.Wait(ctx)
}

func (r *RateLimiter) Reserve() *rate.Reservation {
    return r.limiter.Reserve()
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

### 按 Key 限流

```go
type KeyedRateLimiter struct {
    limiters sync.Map
    rps      int
    burst    int
}

func NewKeyedRateLimiter(rps, burst int) *KeyedRateLimiter {
    return &KeyedRateLimiter{rps: rps, burst: burst}
}

func (k *KeyedRateLimiter) Allow(key string) bool {
    limiter, _ := k.limiters.LoadOrStore(key, rate.NewLimiter(rate.Limit(k.rps), k.burst))
    return limiter.(*rate.Limiter).Allow()
}

// 按用户限流中间件
func UserRateLimitMiddleware(limiter *KeyedRateLimiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            userID := r.Header.Get("X-User-ID")
            if userID == "" {
                userID = r.RemoteAddr
            }

            if !limiter.Allow(userID) {
                w.Header().Set("Retry-After", "1")
                http.Error(w, "rate limit exceeded", http.StatusTooManyRequests)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}
```

### 滑动窗口

```go
type SlidingWindowLimiter struct {
    mu       sync.Mutex
    requests []time.Time
    window   time.Duration
    limit    int
}

func NewSlidingWindowLimiter(window time.Duration, limit int) *SlidingWindowLimiter {
    return &SlidingWindowLimiter{
        window: window,
        limit:  limit,
    }
}

func (s *SlidingWindowLimiter) Allow() bool {
    s.mu.Lock()
    defer s.mu.Unlock()

    now := time.Now()
    cutoff := now.Add(-s.window)

    // 移除过期请求
    valid := s.requests[:0]
    for _, t := range s.requests {
        if t.After(cutoff) {
            valid = append(valid, t)
        }
    }
    s.requests = valid

    if len(s.requests) >= s.limit {
        return false
    }

    s.requests = append(s.requests, now)
    return true
}
```

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
        if err != nil {
            errCh <- err
            return
        }
        resultCh <- result
    }()

    select {
    case <-ctx.Done():
        var zero T
        return zero, ctx.Err()
    case err := <-errCh:
        var zero T
        return zero, err
    case result := <-resultCh:
        return result, nil
    }
}

// 使用
user, err := CallWithTimeout(ctx, 5*time.Second, func(ctx context.Context) (*User, error) {
    return userService.GetUser(ctx, userID)
})
```

### 分层超时

```go
type TimeoutConfig struct {
    Overall   time.Duration // 总超时
    PerCall   time.Duration // 单次调用超时
    PerRetry  time.Duration // 重试间隔
}

func CallWithLayeredTimeout[T any](ctx context.Context, cfg TimeoutConfig, fn func(context.Context) (T, error)) (T, error) {
    // 设置总超时
    ctx, cancel := context.WithTimeout(ctx, cfg.Overall)
    defer cancel()

    return RetryWithBackoff(ctx, func() (T, error) {
        // 单次调用超时
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
    return &Bulkhead{
        sem:     make(chan struct{}, maxConcurrent),
        timeout: timeout,
    }
}

func (b *Bulkhead) Execute(ctx context.Context, fn func() error) error {
    // 尝试获取许可
    select {
    case b.sem <- struct{}{}:
        defer func() { <-b.sem }()
        return fn()
    case <-time.After(b.timeout):
        return ErrBulkheadFull
    case <-ctx.Done():
        return ctx.Err()
    }
}

// 按服务隔离
type ServiceBulkheads struct {
    bulkheads map[string]*Bulkhead
    mu        sync.RWMutex
    config    BulkheadConfig
}

type BulkheadConfig struct {
    DefaultConcurrency int
    DefaultTimeout     time.Duration
    ServiceConfigs     map[string]struct {
        Concurrency int
        Timeout     time.Duration
    }
}

func (s *ServiceBulkheads) Get(service string) *Bulkhead {
    s.mu.RLock()
    if bh, ok := s.bulkheads[service]; ok {
        s.mu.RUnlock()
        return bh
    }
    s.mu.RUnlock()

    s.mu.Lock()
    defer s.mu.Unlock()

    // Double check
    if bh, ok := s.bulkheads[service]; ok {
        return bh
    }

    // 创建新的 bulkhead
    cfg := s.config.ServiceConfigs[service]
    if cfg.Concurrency == 0 {
        cfg.Concurrency = s.config.DefaultConcurrency
        cfg.Timeout = s.config.DefaultTimeout
    }

    bh := NewBulkhead(cfg.Concurrency, cfg.Timeout)
    s.bulkheads[service] = bh

    return bh
}
```

### Worker Pool 隔离

```go
type WorkerPool struct {
    jobs    chan func()
    quit    chan struct{}
    wg      sync.WaitGroup
}

func NewWorkerPool(workers, queueSize int) *WorkerPool {
    pool := &WorkerPool{
        jobs: make(chan func(), queueSize),
        quit: make(chan struct{}),
    }

    for i := 0; i < workers; i++ {
        pool.wg.Add(1)
        go pool.worker()
    }

    return pool
}

func (p *WorkerPool) worker() {
    defer p.wg.Done()
    for {
        select {
        case job := <-p.jobs:
            job()
        case <-p.quit:
            return
        }
    }
}

func (p *WorkerPool) Submit(ctx context.Context, fn func()) error {
    select {
    case p.jobs <- fn:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    default:
        return ErrPoolFull
    }
}

func (p *WorkerPool) SubmitWait(ctx context.Context, fn func()) error {
    select {
    case p.jobs <- fn:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

func (p *WorkerPool) Close() {
    close(p.quit)
    p.wg.Wait()
}
```

---

## 6. 降级（Fallback）

### 降级模式

```go
type Fallback[T any] struct {
    primary  func(context.Context) (T, error)
    fallback func(context.Context, error) (T, error)
}

func NewFallback[T any](primary func(context.Context) (T, error), fallback func(context.Context, error) (T, error)) *Fallback[T] {
    return &Fallback[T]{primary: primary, fallback: fallback}
}

func (f *Fallback[T]) Execute(ctx context.Context) (T, error) {
    result, err := f.primary(ctx)
    if err != nil {
        return f.fallback(ctx, err)
    }
    return result, nil
}

// 使用
func GetUserWithFallback(ctx context.Context, userID string) (*User, error) {
    fb := NewFallback(
        func(ctx context.Context) (*User, error) {
            return userService.GetUser(ctx, userID)
        },
        func(ctx context.Context, err error) (*User, error) {
            // 降级：从缓存获取
            if user := cache.Get(userID); user != nil {
                return user, nil
            }
            // 降级：返回默认值
            return &User{ID: userID, Name: "Unknown"}, nil
        },
    )

    return fb.Execute(ctx)
}
```

### 缓存降级

```go
type CacheFallback[T any] struct {
    cache   Cache[T]
    ttl     time.Duration
    staleTTL time.Duration // 允许使用过期缓存的时间
}

func (c *CacheFallback[T]) Execute(ctx context.Context, key string, loader func(context.Context) (T, error)) (T, error) {
    // 尝试从缓存获取
    if value, ok := c.cache.Get(key); ok {
        return value, nil
    }

    // 加载数据
    value, err := loader(ctx)
    if err != nil {
        // 尝试获取过期缓存
        if stale, ok := c.cache.GetStale(key); ok {
            slog.Warn("using stale cache",
                slog.String("key", key),
                slog.Any("error", err),
            )
            return stale, nil
        }
        return value, err
    }

    // 更新缓存
    c.cache.Set(key, value, c.ttl)

    return value, nil
}
```

---

## 7. 组合模式

### 完整的弹性调用

```go
type ResilientCall[T any] struct {
    breaker  *gobreaker.CircuitBreaker[T]
    limiter  *RateLimiter
    bulkhead *Bulkhead
    retry    RetryPolicy
    timeout  time.Duration
    fallback func(context.Context, error) (T, error)
}

func (r *ResilientCall[T]) Execute(ctx context.Context, fn func(context.Context) (T, error)) (T, error) {
    var zero T

    // 1. 限流
    if !r.limiter.Allow() {
        if r.fallback != nil {
            return r.fallback(ctx, ErrRateLimited)
        }
        return zero, ErrRateLimited
    }

    // 2. 舱壁
    resultCh := make(chan T, 1)
    errCh := make(chan error, 1)

    err := r.bulkhead.Execute(ctx, func() error {
        // 3. 超时
        ctx, cancel := context.WithTimeout(ctx, r.timeout)
        defer cancel()

        // 4. 熔断 + 重试
        result, err := r.breaker.Execute(func() (T, error) {
            return Retry(ctx, r.retry, func() (T, error) {
                return fn(ctx)
            })
        })

        if err != nil {
            errCh <- err
            return err
        }

        resultCh <- result
        return nil
    })

    if err != nil {
        if r.fallback != nil {
            return r.fallback(ctx, err)
        }
        return zero, err
    }

    select {
    case result := <-resultCh:
        return result, nil
    case err := <-errCh:
        if r.fallback != nil {
            return r.fallback(ctx, err)
        }
        return zero, err
    }
}
```

### Builder 模式

```go
type ResilientCallBuilder[T any] struct {
    call *ResilientCall[T]
}

func NewResilientCallBuilder[T any]() *ResilientCallBuilder[T] {
    return &ResilientCallBuilder[T]{
        call: &ResilientCall[T]{
            timeout: 10 * time.Second,
        },
    }
}

func (b *ResilientCallBuilder[T]) WithCircuitBreaker(cb *gobreaker.CircuitBreaker[T]) *ResilientCallBuilder[T] {
    b.call.breaker = cb
    return b
}

func (b *ResilientCallBuilder[T]) WithRateLimiter(limiter *RateLimiter) *ResilientCallBuilder[T] {
    b.call.limiter = limiter
    return b
}

func (b *ResilientCallBuilder[T]) WithBulkhead(bh *Bulkhead) *ResilientCallBuilder[T] {
    b.call.bulkhead = bh
    return b
}

func (b *ResilientCallBuilder[T]) WithRetry(policy RetryPolicy) *ResilientCallBuilder[T] {
    b.call.retry = policy
    return b
}

func (b *ResilientCallBuilder[T]) WithTimeout(timeout time.Duration) *ResilientCallBuilder[T] {
    b.call.timeout = timeout
    return b
}

func (b *ResilientCallBuilder[T]) WithFallback(fallback func(context.Context, error) (T, error)) *ResilientCallBuilder[T] {
    b.call.fallback = fallback
    return b
}

func (b *ResilientCallBuilder[T]) Build() *ResilientCall[T] {
    return b.call
}

// 使用
call := NewResilientCallBuilder[*User]().
    WithCircuitBreaker(breaker).
    WithRateLimiter(limiter).
    WithBulkhead(bulkhead).
    WithRetry(&ExponentialBackoff{MaxAttempts: 3}).
    WithTimeout(5 * time.Second).
    WithFallback(func(ctx context.Context, err error) (*User, error) {
        return cache.GetUser(userID), nil
    }).
    Build()

user, err := call.Execute(ctx, func(ctx context.Context) (*User, error) {
    return userService.GetUser(ctx, userID)
})
```

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
