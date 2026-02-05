---
name: backend-patterns
description: 后端架构模式专家 - 分层架构、Repository 模式、缓存策略、错误处理、认证授权、限流、队列、日志。使用场景：系统设计、架构决策、服务开发。
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

# 后端架构模式专家

应用后端架构模式设计或实现服务：$ARGUMENTS

---

## API 设计模式

### RESTful API 结构

```go
// 资源导向的 URL 设计
GET    /api/v1/users              // 列出资源
GET    /api/v1/users/:id          // 获取单个资源
POST   /api/v1/users              // 创建资源
PUT    /api/v1/users/:id          // 替换资源
PATCH  /api/v1/users/:id          // 更新资源
DELETE /api/v1/users/:id          // 删除资源

// 查询参数用于过滤、排序、分页
GET /api/v1/users?status=active&sort=created_at&limit=20&offset=0
```

### Repository 模式

抽象数据访问逻辑，解耦业务层和存储层。

```go
// 接口定义在使用方
type UserRepository interface {
    FindAll(ctx context.Context, filters UserFilters) ([]*User, error)
    FindByID(ctx context.Context, id string) (*User, error)
    Create(ctx context.Context, user *User) error
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id string) error
}

// MongoDB 实现
type mongoUserRepository struct {
    coll *mongo.Collection
}

func NewMongoUserRepository(db *mongo.Database) UserRepository {
    return &mongoUserRepository{
        coll: db.Collection("users"),
    }
}

func (r *mongoUserRepository) FindAll(ctx context.Context, filters UserFilters) ([]*User, error) {
    filter := bson.M{}
    if filters.Status != "" {
        filter["status"] = filters.Status
    }

    opts := options.Find()
    if filters.Limit > 0 {
        opts.SetLimit(int64(filters.Limit))
    }

    cursor, err := r.coll.Find(ctx, filter, opts)
    if err != nil {
        return nil, fmt.Errorf("find users: %w", err)
    }
    defer cursor.Close(ctx)

    var users []*User
    if err := cursor.All(ctx, &users); err != nil {
        return nil, fmt.Errorf("decode users: %w", err)
    }
    return users, nil
}
```

### Service 层模式

业务逻辑与数据访问分离。

```go
type UserService struct {
    repo   UserRepository
    cache  CacheService
    logger *slog.Logger
}

func NewUserService(repo UserRepository, cache CacheService, logger *slog.Logger) *UserService {
    return &UserService{
        repo:   repo,
        cache:  cache,
        logger: logger,
    }
}

func (s *UserService) GetUser(ctx context.Context, id string) (*User, error) {
    // 1. 尝试缓存
    if cached, err := s.cache.Get(ctx, "user:"+id); err == nil {
        var user User
        if err := json.Unmarshal(cached, &user); err == nil {
            return &user, nil
        }
    }

    // 2. 从数据库获取
    user, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("get user %s: %w", id, err)
    }

    // 3. 写入缓存
    if data, err := json.Marshal(user); err == nil {
        _ = s.cache.Set(ctx, "user:"+id, data, 5*time.Minute)
    }

    return user, nil
}
```

### 中间件模式

用于请求/响应处理管道。

```go
type Middleware func(http.Handler) http.Handler

func Chain(h http.Handler, middlewares ...Middleware) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

func LoggingMiddleware(logger *slog.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()

            // 包装 ResponseWriter 捕获状态码
            ww := &responseWriter{ResponseWriter: w, status: 200}

            next.ServeHTTP(ww, r)

            logger.Info("request",
                "method", r.Method,
                "path", r.URL.Path,
                "status", ww.status,
                "duration", time.Since(start),
            )
        })
    }
}

func RecoveryMiddleware(logger *slog.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            defer func() {
                if err := recover(); err != nil {
                    logger.Error("panic recovered", "error", err)
                    http.Error(w, "Internal Server Error", 500)
                }
            }()
            next.ServeHTTP(w, r)
        })
    }
}

// 使用
handler := Chain(s.Routes(),
    RecoveryMiddleware(logger),
    LoggingMiddleware(logger),
    AuthMiddleware(authService),
)
```

---

## 数据库模式

### 查询优化

```go
// ✅ 只选择需要的列
func (r *userRepo) FindAll(ctx context.Context) ([]*UserSummary, error) {
    query := `SELECT id, name, status FROM users WHERE status = $1`
    rows, err := r.db.QueryContext(ctx, query, "active")
    // ...
}

// ❌ 选择所有列
query := `SELECT * FROM users WHERE status = $1`
```

### 避免 N+1 查询

```go
// ❌ N+1 问题
func (s *OrderService) GetOrdersWithUsers(ctx context.Context) ([]*OrderWithUser, error) {
    orders, _ := s.orderRepo.FindAll(ctx)
    for _, order := range orders {
        order.User, _ = s.userRepo.FindByID(ctx, order.UserID)  // N 次查询
    }
    return orders, nil
}

// ✅ 批量获取
func (s *OrderService) GetOrdersWithUsers(ctx context.Context) ([]*OrderWithUser, error) {
    orders, _ := s.orderRepo.FindAll(ctx)

    // 收集所有用户 ID
    userIDs := make([]string, 0, len(orders))
    for _, order := range orders {
        userIDs = append(userIDs, order.UserID)
    }

    // 批量获取用户（1 次查询）
    users, _ := s.userRepo.FindByIDs(ctx, userIDs)
    userMap := make(map[string]*User)
    for _, user := range users {
        userMap[user.ID] = user
    }

    // 组装结果
    for _, order := range orders {
        order.User = userMap[order.UserID]
    }
    return orders, nil
}
```

### 事务模式

```go
type TxManager interface {
    WithTransaction(ctx context.Context, fn func(ctx context.Context) error) error
}

type mongoTxManager struct {
    client *mongo.Client
}

func (m *mongoTxManager) WithTransaction(ctx context.Context, fn func(ctx context.Context) error) error {
    session, err := m.client.StartSession()
    if err != nil {
        return fmt.Errorf("start session: %w", err)
    }
    defer session.EndSession(ctx)

    _, err = session.WithTransaction(ctx, func(sessCtx mongo.SessionContext) (interface{}, error) {
        return nil, fn(sessCtx)
    })
    return err
}

// 使用
func (s *OrderService) CreateOrder(ctx context.Context, order *Order) error {
    return s.txManager.WithTransaction(ctx, func(ctx context.Context) error {
        if err := s.orderRepo.Create(ctx, order); err != nil {
            return err
        }
        if err := s.inventoryRepo.Decrease(ctx, order.ProductID, order.Quantity); err != nil {
            return err
        }
        return nil
    })
}
```

---

## 缓存策略

### Cache-Aside 模式

```go
type CachedUserRepository struct {
    repo  UserRepository
    cache CacheService
    ttl   time.Duration
}

func (r *CachedUserRepository) FindByID(ctx context.Context, id string) (*User, error) {
    cacheKey := "user:" + id

    // 1. 检查缓存
    if data, err := r.cache.Get(ctx, cacheKey); err == nil {
        var user User
        if json.Unmarshal(data, &user) == nil {
            return &user, nil
        }
    }

    // 2. 缓存未命中 - 从数据库获取
    user, err := r.repo.FindByID(ctx, id)
    if err != nil {
        return nil, err
    }

    // 3. 写入缓存
    if data, err := json.Marshal(user); err == nil {
        _ = r.cache.SetEx(ctx, cacheKey, data, r.ttl)
    }

    return user, nil
}

func (r *CachedUserRepository) Update(ctx context.Context, user *User) error {
    if err := r.repo.Update(ctx, user); err != nil {
        return err
    }
    // 失效缓存
    _ = r.cache.Del(ctx, "user:"+user.ID)
    return nil
}
```

### Write-Through 模式

```go
func (r *CachedUserRepository) Create(ctx context.Context, user *User) error {
    // 1. 写入数据库
    if err := r.repo.Create(ctx, user); err != nil {
        return err
    }

    // 2. 同时写入缓存
    if data, err := json.Marshal(user); err == nil {
        _ = r.cache.SetEx(ctx, "user:"+user.ID, data, r.ttl)
    }

    return nil
}
```

---

## 错误处理模式

### 集中式错误处理

```go
type APIError struct {
    Code    int    `json:"-"`
    Message string `json:"message"`
    Details any    `json:"details,omitempty"`
}

func (e *APIError) Error() string {
    return e.Message
}

var (
    ErrNotFound     = &APIError{Code: 404, Message: "resource not found"}
    ErrUnauthorized = &APIError{Code: 401, Message: "unauthorized"}
    ErrForbidden    = &APIError{Code: 403, Message: "forbidden"}
    ErrBadRequest   = &APIError{Code: 400, Message: "bad request"}
)

func ErrorHandler(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            defer func() {
                if err := recover(); err != nil {
                    logger.Error("panic", "error", err)
                    writeJSON(w, 500, map[string]string{"message": "internal error"})
                }
            }()
            next.ServeHTTP(w, r)
        })
    }
}

func handleError(w http.ResponseWriter, err error) {
    var apiErr *APIError
    if errors.As(err, &apiErr) {
        writeJSON(w, apiErr.Code, apiErr)
        return
    }
    writeJSON(w, 500, map[string]string{"message": "internal error"})
}
```

### 指数退避重试

```go
type RetryConfig struct {
    MaxRetries  int
    BaseDelay   time.Duration
    MaxDelay    time.Duration
    Multiplier  float64
}

func DefaultRetryConfig() RetryConfig {
    return RetryConfig{
        MaxRetries: 3,
        BaseDelay:  time.Second,
        MaxDelay:   30 * time.Second,
        Multiplier: 2.0,
    }
}

func WithRetry[T any](ctx context.Context, cfg RetryConfig, fn func() (T, error)) (T, error) {
    var lastErr error
    var zero T

    delay := cfg.BaseDelay
    for attempt := 0; attempt <= cfg.MaxRetries; attempt++ {
        result, err := fn()
        if err == nil {
            return result, nil
        }
        lastErr = err

        if attempt < cfg.MaxRetries {
            select {
            case <-ctx.Done():
                return zero, ctx.Err()
            case <-time.After(delay):
            }
            delay = time.Duration(float64(delay) * cfg.Multiplier)
            if delay > cfg.MaxDelay {
                delay = cfg.MaxDelay
            }
        }
    }

    return zero, fmt.Errorf("after %d retries: %w", cfg.MaxRetries, lastErr)
}

// 使用
result, err := WithRetry(ctx, DefaultRetryConfig(), func() (*Response, error) {
    return client.Call(ctx, request)
})
```

---

## 认证与授权

### JWT 验证

```go
type Claims struct {
    UserID string `json:"user_id"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

type AuthService struct {
    secret []byte
}

func (s *AuthService) ValidateToken(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        return s.secret, nil
    })

    if err != nil {
        return nil, fmt.Errorf("parse token: %w", err)
    }

    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, errors.New("invalid token")
    }

    return claims, nil
}

func AuthMiddleware(authService *AuthService) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            token := strings.TrimPrefix(r.Header.Get("Authorization"), "Bearer ")
            if token == "" {
                handleError(w, ErrUnauthorized)
                return
            }

            claims, err := authService.ValidateToken(token)
            if err != nil {
                handleError(w, ErrUnauthorized)
                return
            }

            ctx := context.WithValue(r.Context(), userClaimsKey, claims)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

### 基于角色的访问控制

```go
type Permission string

const (
    PermRead   Permission = "read"
    PermWrite  Permission = "write"
    PermDelete Permission = "delete"
    PermAdmin  Permission = "admin"
)

var rolePermissions = map[string][]Permission{
    "admin":     {PermRead, PermWrite, PermDelete, PermAdmin},
    "moderator": {PermRead, PermWrite, PermDelete},
    "user":      {PermRead, PermWrite},
    "guest":     {PermRead},
}

func HasPermission(role string, perm Permission) bool {
    perms, ok := rolePermissions[role]
    if !ok {
        return false
    }
    for _, p := range perms {
        if p == perm {
            return true
        }
    }
    return false
}

func RequirePermission(perm Permission) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            claims := GetClaims(r.Context())
            if claims == nil {
                handleError(w, ErrUnauthorized)
                return
            }

            if !HasPermission(claims.Role, perm) {
                handleError(w, ErrForbidden)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}
```

---

## 限流

### 令牌桶限流器

```go
type RateLimiter struct {
    mu       sync.Mutex
    tokens   map[string]*tokenBucket
    rate     float64  // 每秒令牌数
    capacity float64  // 桶容量
}

type tokenBucket struct {
    tokens    float64
    lastCheck time.Time
}

func NewRateLimiter(rate, capacity float64) *RateLimiter {
    return &RateLimiter{
        tokens:   make(map[string]*tokenBucket),
        rate:     rate,
        capacity: capacity,
    }
}

func (rl *RateLimiter) Allow(key string) bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()

    now := time.Now()
    bucket, exists := rl.tokens[key]
    if !exists {
        bucket = &tokenBucket{tokens: rl.capacity, lastCheck: now}
        rl.tokens[key] = bucket
    }

    // 补充令牌
    elapsed := now.Sub(bucket.lastCheck).Seconds()
    bucket.tokens = min(rl.capacity, bucket.tokens+elapsed*rl.rate)
    bucket.lastCheck = now

    if bucket.tokens >= 1 {
        bucket.tokens--
        return true
    }
    return false
}

func RateLimitMiddleware(limiter *RateLimiter) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            key := r.RemoteAddr // 或使用 user ID
            if !limiter.Allow(key) {
                w.Header().Set("Retry-After", "1")
                http.Error(w, "rate limit exceeded", http.StatusTooManyRequests)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

---

## 后台任务队列

### 简单内存队列

```go
type Job struct {
    ID      string
    Type    string
    Payload json.RawMessage
}

type JobQueue struct {
    jobs       chan Job
    workers    int
    handler    func(Job) error
    logger     *slog.Logger
    wg         sync.WaitGroup
}

func NewJobQueue(workers int, handler func(Job) error, logger *slog.Logger) *JobQueue {
    return &JobQueue{
        jobs:    make(chan Job, 1000),
        workers: workers,
        handler: handler,
        logger:  logger,
    }
}

func (q *JobQueue) Start(ctx context.Context) {
    for i := 0; i < q.workers; i++ {
        q.wg.Add(1)
        go q.worker(ctx, i)
    }
}

func (q *JobQueue) worker(ctx context.Context, id int) {
    defer q.wg.Done()
    for {
        select {
        case <-ctx.Done():
            return
        case job := <-q.jobs:
            if err := q.handler(job); err != nil {
                q.logger.Error("job failed",
                    "worker", id,
                    "job_id", job.ID,
                    "error", err,
                )
            }
        }
    }
}

func (q *JobQueue) Enqueue(job Job) error {
    select {
    case q.jobs <- job:
        return nil
    default:
        return errors.New("queue full")
    }
}

func (q *JobQueue) Shutdown() {
    close(q.jobs)
    q.wg.Wait()
}
```

---

## 结构化日志

```go
type RequestLogger struct {
    logger *slog.Logger
}

func (l *RequestLogger) Middleware() Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            requestID := uuid.New().String()

            // 添加请求 ID 到 context
            ctx := context.WithValue(r.Context(), requestIDKey, requestID)

            // 创建带请求上下文的 logger
            reqLogger := l.logger.With(
                "request_id", requestID,
                "method", r.Method,
                "path", r.URL.Path,
                "remote_addr", r.RemoteAddr,
            )
            ctx = context.WithValue(ctx, loggerKey, reqLogger)

            start := time.Now()
            ww := &responseWriter{ResponseWriter: w, status: 200}

            next.ServeHTTP(ww, r.WithContext(ctx))

            reqLogger.Info("request completed",
                "status", ww.status,
                "duration_ms", time.Since(start).Milliseconds(),
            )
        })
    }
}

// 从 context 获取 logger
func LoggerFromContext(ctx context.Context) *slog.Logger {
    if logger, ok := ctx.Value(loggerKey).(*slog.Logger); ok {
        return logger
    }
    return slog.Default()
}

// 使用
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    logger := LoggerFromContext(r.Context())
    logger.Info("fetching user", "user_id", chi.URLParam(r, "id"))
    // ...
}
```

---

## 快速检查清单

### 架构设计
- [ ] 分层清晰：Handler → Service → Repository
- [ ] 依赖注入，避免全局状态
- [ ] 接口在使用方定义
- [ ] 错误集中处理

### API 设计
- [ ] RESTful 资源命名
- [ ] 版本控制 `/api/v1/`
- [ ] 统一响应格式
- [ ] 分页、过滤、排序参数

### 数据访问
- [ ] 避免 N+1 查询
- [ ] 只查询需要的列
- [ ] 事务处理多步操作
- [ ] 索引覆盖常用查询

### 缓存
- [ ] 合适的 TTL
- [ ] 缓存失效策略
- [ ] 防止缓存穿透/雪崩

### 安全
- [ ] 认证中间件
- [ ] 权限检查
- [ ] 限流保护
- [ ] 输入验证

**记住**：后端架构模式使服务可扩展、可维护。选择匹配复杂度级别的模式。
