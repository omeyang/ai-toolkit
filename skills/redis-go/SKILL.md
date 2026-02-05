---
name: redis-go
description: Go Redis 专家 - 缓存模式、分布式锁、限流、Lua脚本、Pipeline、Pub/Sub、集群模式。使用场景：缓存、会话、排行榜、实时计数。
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

# Go Redis 专家

使用 Go go-redis 开发 Redis 功能：$ARGUMENTS

---

## 1. 客户端管理

### 单机连接

```go
import (
    "context"
    "time"

    "github.com/redis/go-redis/v9"
)

func NewRedisClient(addr, password string, db int) (*redis.Client, error) {
    client := redis.NewClient(&redis.Options{
        Addr:            addr,
        Password:        password,
        DB:              db,
        PoolSize:        100,
        MinIdleConns:    10,
        MaxIdleConns:    50,
        ConnMaxIdleTime: 5 * time.Minute,
        ConnMaxLifetime: 30 * time.Minute,
        DialTimeout:     5 * time.Second,
        ReadTimeout:     3 * time.Second,
        WriteTimeout:    3 * time.Second,
        PoolTimeout:     4 * time.Second,
    })

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := client.Ping(ctx).Err(); err != nil {
        return nil, fmt.Errorf("ping redis: %w", err)
    }

    return client, nil
}
```

### 集群连接

```go
func NewRedisCluster(addrs []string, password string) (*redis.ClusterClient, error) {
    client := redis.NewClusterClient(&redis.ClusterOptions{
        Addrs:           addrs,
        Password:        password,
        PoolSize:        100,
        MinIdleConns:    10,
        MaxIdleConns:    50,
        DialTimeout:     5 * time.Second,
        ReadTimeout:     3 * time.Second,
        WriteTimeout:    3 * time.Second,
        RouteByLatency:  true,  // 读取路由到延迟最低的节点
        RouteRandomly:   false,
    })

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := client.Ping(ctx).Err(); err != nil {
        return nil, fmt.Errorf("ping cluster: %w", err)
    }

    return client, nil
}
```

### Sentinel 连接

```go
func NewRedisSentinel(masterName string, sentinelAddrs []string, password string) (*redis.Client, error) {
    client := redis.NewFailoverClient(&redis.FailoverOptions{
        MasterName:       masterName,
        SentinelAddrs:    sentinelAddrs,
        SentinelPassword: password,
        Password:         password,
        DB:               0,
        PoolSize:         100,
    })

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := client.Ping(ctx).Err(); err != nil {
        return nil, fmt.Errorf("ping sentinel: %w", err)
    }

    return client, nil
}
```

### 包装器模式

```go
type Redis struct {
    client redis.UniversalClient
}

func New(client redis.UniversalClient) *Redis {
    return &Redis{client: client}
}

// Client 暴露底层客户端
func (r *Redis) Client() redis.UniversalClient {
    return r.client
}

// Health 健康检查
func (r *Redis) Health(ctx context.Context) error {
    return r.client.Ping(ctx).Err()
}

// Close 关闭连接
func (r *Redis) Close() error {
    return r.client.Close()
}
```

---

## 2. 缓存模式

### Cache-Aside（旁路缓存）

```go
type CacheOptions struct {
    TTL      time.Duration
    NullTTL  time.Duration // 空值缓存时间（防穿透）
}

func (r *Redis) GetOrLoad[T any](ctx context.Context, key string, loader func(context.Context) (T, error), opts CacheOptions) (T, error) {
    var zero T

    // 1. 尝试从缓存获取
    data, err := r.client.Get(ctx, key).Bytes()
    if err == nil {
        var result T
        if err := json.Unmarshal(data, &result); err != nil {
            return zero, fmt.Errorf("unmarshal cache: %w", err)
        }
        return result, nil
    }

    if !errors.Is(err, redis.Nil) {
        return zero, fmt.Errorf("get cache: %w", err)
    }

    // 2. 缓存未命中，加载数据
    result, err := loader(ctx)
    if err != nil {
        // 可选：缓存空值防穿透
        if errors.Is(err, ErrNotFound) && opts.NullTTL > 0 {
            r.client.Set(ctx, key, "null", opts.NullTTL)
        }
        return zero, err
    }

    // 3. 写入缓存
    data, err = json.Marshal(result)
    if err != nil {
        return result, nil // 忽略序列化错误
    }

    r.client.Set(ctx, key, data, opts.TTL)

    return result, nil
}
```

### Singleflight 防击穿

```go
import "golang.org/x/sync/singleflight"

type CacheWithSingleflight struct {
    *Redis
    group singleflight.Group
}

func (c *CacheWithSingleflight) GetOrLoad[T any](ctx context.Context, key string, loader func(context.Context) (T, error), opts CacheOptions) (T, error) {
    var zero T

    // 尝试缓存
    data, err := c.client.Get(ctx, key).Bytes()
    if err == nil {
        var result T
        if err := json.Unmarshal(data, &result); err != nil {
            return zero, err
        }
        return result, nil
    }

    if !errors.Is(err, redis.Nil) {
        return zero, err
    }

    // Singleflight 合并请求
    v, err, _ := c.group.Do(key, func() (any, error) {
        // 再次检查缓存（可能其他请求已写入）
        data, err := c.client.Get(ctx, key).Bytes()
        if err == nil {
            var result T
            if err := json.Unmarshal(data, &result); err != nil {
                return zero, err
            }
            return result, nil
        }

        result, err := loader(ctx)
        if err != nil {
            return zero, err
        }

        data, _ = json.Marshal(result)
        c.client.Set(ctx, key, data, opts.TTL)

        return result, nil
    })

    if err != nil {
        return zero, err
    }

    return v.(T), nil
}
```

### 双层缓存（L1 本地 + L2 Redis）

```go
type DualCache[T any] struct {
    l1     *expirable.LRU[string, T]
    l2     *Redis
    l1TTL  time.Duration
    l2TTL  time.Duration
}

func NewDualCache[T any](l2 *Redis, l1Size int, l1TTL, l2TTL time.Duration) *DualCache[T] {
    return &DualCache[T]{
        l1:    expirable.NewLRU[string, T](l1Size, nil, l1TTL),
        l2:    l2,
        l1TTL: l1TTL,
        l2TTL: l2TTL,
    }
}

func (c *DualCache[T]) Get(ctx context.Context, key string) (T, bool) {
    var zero T

    // L1 查找
    if v, ok := c.l1.Get(key); ok {
        return v, true
    }

    // L2 查找
    data, err := c.l2.client.Get(ctx, key).Bytes()
    if err != nil {
        return zero, false
    }

    var result T
    if err := json.Unmarshal(data, &result); err != nil {
        return zero, false
    }

    // 回填 L1
    c.l1.Add(key, result)

    return result, true
}

func (c *DualCache[T]) Set(ctx context.Context, key string, value T) error {
    // 写 L1
    c.l1.Add(key, value)

    // 写 L2
    data, err := json.Marshal(value)
    if err != nil {
        return err
    }

    return c.l2.client.Set(ctx, key, data, c.l2TTL).Err()
}

func (c *DualCache[T]) Delete(ctx context.Context, key string) error {
    c.l1.Remove(key)
    return c.l2.client.Del(ctx, key).Err()
}
```

---

## 3. 分布式锁

### 简单锁

```go
type Lock struct {
    client redis.UniversalClient
    key    string
    value  string
    ttl    time.Duration
}

func NewLock(client redis.UniversalClient, key string, ttl time.Duration) *Lock {
    return &Lock{
        client: client,
        key:    "lock:" + key,
        value:  uuid.New().String(),
        ttl:    ttl,
    }
}

func (l *Lock) TryLock(ctx context.Context) (bool, error) {
    return l.client.SetNX(ctx, l.key, l.value, l.ttl).Result()
}

func (l *Lock) Lock(ctx context.Context) error {
    for {
        ok, err := l.TryLock(ctx)
        if err != nil {
            return err
        }
        if ok {
            return nil
        }

        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(50 * time.Millisecond):
        }
    }
}

// 使用 Lua 脚本确保原子性释放
var unlockScript = redis.NewScript(`
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
`)

func (l *Lock) Unlock(ctx context.Context) error {
    result, err := unlockScript.Run(ctx, l.client, []string{l.key}, l.value).Int()
    if err != nil {
        return err
    }
    if result == 0 {
        return ErrLockNotHeld
    }
    return nil
}

// 续期
var extendScript = redis.NewScript(`
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("pexpire", KEYS[1], ARGV[2])
else
    return 0
end
`)

func (l *Lock) Extend(ctx context.Context, ttl time.Duration) error {
    result, err := extendScript.Run(ctx, l.client, []string{l.key}, l.value, int(ttl.Milliseconds())).Int()
    if err != nil {
        return err
    }
    if result == 0 {
        return ErrLockNotHeld
    }
    l.ttl = ttl
    return nil
}
```

### Redlock（多节点锁）

```go
import "github.com/go-redsync/redsync/v4"
import "github.com/go-redsync/redsync/v4/redis/goredis/v9"

func NewRedlock(clients ...*redis.Client) *redsync.Redsync {
    var pools []redsync.Pool
    for _, client := range clients {
        pools = append(pools, goredis.NewPool(client))
    }
    return redsync.New(pools...)
}

func ExampleRedlock() {
    rs := NewRedlock(client1, client2, client3)

    mutex := rs.NewMutex("resource-key",
        redsync.WithExpiry(10*time.Second),
        redsync.WithTries(32),
        redsync.WithRetryDelay(100*time.Millisecond),
    )

    if err := mutex.Lock(); err != nil {
        log.Fatal(err)
    }

    // 业务逻辑...

    if _, err := mutex.Unlock(); err != nil {
        log.Fatal(err)
    }
}
```

---

## 4. 限流

### 令牌桶

```go
import "github.com/go-redis/redis_rate/v10"

type RateLimiter struct {
    limiter *redis_rate.Limiter
}

func NewRateLimiter(client *redis.Client) *RateLimiter {
    return &RateLimiter{
        limiter: redis_rate.NewLimiter(client),
    }
}

func (r *RateLimiter) Allow(ctx context.Context, key string, rate redis_rate.Limit) (bool, error) {
    res, err := r.limiter.Allow(ctx, key, rate)
    if err != nil {
        return false, err
    }
    return res.Allowed > 0, nil
}

// 使用示例
func Example() {
    limiter := NewRateLimiter(client)

    // 每秒 10 次，突发 20
    limit := redis_rate.PerSecond(10)

    allowed, err := limiter.Allow(ctx, "user:123", limit)
    if err != nil {
        return err
    }
    if !allowed {
        return ErrRateLimited
    }
}
```

### 滑动窗口

```go
var slidingWindowScript = redis.NewScript(`
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])

-- 移除窗口外的记录
redis.call("ZREMRANGEBYSCORE", key, 0, now - window)

-- 获取当前窗口内的请求数
local count = redis.call("ZCARD", key)

if count < limit then
    -- 添加当前请求
    redis.call("ZADD", key, now, now .. "-" .. math.random())
    redis.call("PEXPIRE", key, window)
    return 1
else
    return 0
end
`)

func (r *Redis) SlidingWindowLimit(ctx context.Context, key string, window time.Duration, limit int64) (bool, error) {
    now := time.Now().UnixMilli()
    result, err := slidingWindowScript.Run(ctx, r.client, []string{key}, now, window.Milliseconds(), limit).Int()
    if err != nil {
        return false, err
    }
    return result == 1, nil
}
```

---

## 5. Pipeline 批量操作

### 基础 Pipeline

```go
func (r *Redis) BatchGet(ctx context.Context, keys []string) (map[string]string, error) {
    pipe := r.client.Pipeline()

    cmds := make(map[string]*redis.StringCmd)
    for _, key := range keys {
        cmds[key] = pipe.Get(ctx, key)
    }

    _, err := pipe.Exec(ctx)
    if err != nil && !errors.Is(err, redis.Nil) {
        return nil, err
    }

    results := make(map[string]string)
    for key, cmd := range cmds {
        val, err := cmd.Result()
        if err == nil {
            results[key] = val
        }
    }

    return results, nil
}

func (r *Redis) BatchSet(ctx context.Context, items map[string]any, ttl time.Duration) error {
    pipe := r.client.Pipeline()

    for key, value := range items {
        data, err := json.Marshal(value)
        if err != nil {
            return err
        }
        pipe.Set(ctx, key, data, ttl)
    }

    _, err := pipe.Exec(ctx)
    return err
}
```

### 事务 Pipeline

```go
func (r *Redis) Transfer(ctx context.Context, from, to string, amount int64) error {
    // 乐观锁监视
    err := r.client.Watch(ctx, func(tx *redis.Tx) error {
        // 检查余额
        balance, err := tx.Get(ctx, from).Int64()
        if err != nil {
            return err
        }
        if balance < amount {
            return ErrInsufficientBalance
        }

        // 执行事务
        _, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
            pipe.DecrBy(ctx, from, amount)
            pipe.IncrBy(ctx, to, amount)
            return nil
        })
        return err
    }, from) // 监视 from 键

    return err
}
```

---

## 6. Pub/Sub

### 发布者

```go
func (r *Redis) Publish(ctx context.Context, channel string, message any) error {
    data, err := json.Marshal(message)
    if err != nil {
        return err
    }
    return r.client.Publish(ctx, channel, data).Err()
}
```

### 订阅者

```go
func (r *Redis) Subscribe(ctx context.Context, channels []string, handler func(channel string, payload []byte)) error {
    pubsub := r.client.Subscribe(ctx, channels...)
    defer pubsub.Close()

    // 等待订阅确认
    _, err := pubsub.Receive(ctx)
    if err != nil {
        return err
    }

    ch := pubsub.Channel()
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case msg, ok := <-ch:
            if !ok {
                return nil
            }
            handler(msg.Channel, []byte(msg.Payload))
        }
    }
}

// 模式订阅
func (r *Redis) PSubscribe(ctx context.Context, patterns []string, handler func(pattern, channel string, payload []byte)) error {
    pubsub := r.client.PSubscribe(ctx, patterns...)
    defer pubsub.Close()

    ch := pubsub.Channel()
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case msg, ok := <-ch:
            if !ok {
                return nil
            }
            handler(msg.Pattern, msg.Channel, []byte(msg.Payload))
        }
    }
}
```

---

## 7. 数据结构

### 排行榜（Sorted Set）

```go
type Leaderboard struct {
    *Redis
    key string
}

func NewLeaderboard(r *Redis, name string) *Leaderboard {
    return &Leaderboard{Redis: r, key: "leaderboard:" + name}
}

func (l *Leaderboard) Add(ctx context.Context, member string, score float64) error {
    return l.client.ZAdd(ctx, l.key, redis.Z{Score: score, Member: member}).Err()
}

func (l *Leaderboard) IncrScore(ctx context.Context, member string, delta float64) (float64, error) {
    return l.client.ZIncrBy(ctx, l.key, delta, member).Result()
}

func (l *Leaderboard) Top(ctx context.Context, n int64) ([]redis.Z, error) {
    return l.client.ZRevRangeWithScores(ctx, l.key, 0, n-1).Result()
}

func (l *Leaderboard) Rank(ctx context.Context, member string) (int64, error) {
    rank, err := l.client.ZRevRank(ctx, l.key, member).Result()
    if err != nil {
        return -1, err
    }
    return rank + 1, nil // 1-based
}

func (l *Leaderboard) Score(ctx context.Context, member string) (float64, error) {
    return l.client.ZScore(ctx, l.key, member).Result()
}

func (l *Leaderboard) Around(ctx context.Context, member string, count int64) ([]redis.Z, error) {
    rank, err := l.client.ZRevRank(ctx, l.key, member).Result()
    if err != nil {
        return nil, err
    }

    start := rank - count
    if start < 0 {
        start = 0
    }
    end := rank + count

    return l.client.ZRevRangeWithScores(ctx, l.key, start, end).Result()
}
```

### 布隆过滤器

```go
// 使用 RedisBloom 模块
func (r *Redis) BFAdd(ctx context.Context, key string, item string) error {
    return r.client.Do(ctx, "BF.ADD", key, item).Err()
}

func (r *Redis) BFExists(ctx context.Context, key string, item string) (bool, error) {
    return r.client.Do(ctx, "BF.EXISTS", key, item).Bool()
}

func (r *Redis) BFMAdd(ctx context.Context, key string, items ...string) error {
    args := make([]any, 0, len(items)+2)
    args = append(args, "BF.MADD", key)
    for _, item := range items {
        args = append(args, item)
    }
    return r.client.Do(ctx, args...).Err()
}
```

### HyperLogLog（基数统计）

```go
func (r *Redis) HLLAdd(ctx context.Context, key string, elements ...string) error {
    args := make([]any, len(elements))
    for i, e := range elements {
        args[i] = e
    }
    return r.client.PFAdd(ctx, key, args...).Err()
}

func (r *Redis) HLLCount(ctx context.Context, keys ...string) (int64, error) {
    return r.client.PFCount(ctx, keys...).Result()
}

// UV 统计
func (r *Redis) RecordUV(ctx context.Context, date string, userID string) error {
    key := fmt.Sprintf("uv:%s", date)
    return r.HLLAdd(ctx, key, userID)
}

func (r *Redis) GetUV(ctx context.Context, dates ...string) (int64, error) {
    keys := make([]string, len(dates))
    for i, d := range dates {
        keys[i] = fmt.Sprintf("uv:%s", d)
    }
    return r.HLLCount(ctx, keys...)
}
```

---

## 8. Lua 脚本

### 脚本管理

```go
// 预加载脚本
var (
    scriptCompareAndSet = redis.NewScript(`
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("set", KEYS[1], ARGV[2])
        else
            return nil
        end
    `)

    scriptIncrWithCap = redis.NewScript(`
        local current = tonumber(redis.call("get", KEYS[1]) or 0)
        local cap = tonumber(ARGV[1])
        local incr = tonumber(ARGV[2])

        if current + incr > cap then
            return -1
        end

        return redis.call("incrby", KEYS[1], incr)
    `)
)

func (r *Redis) CompareAndSet(ctx context.Context, key, expected, newValue string) (bool, error) {
    result, err := scriptCompareAndSet.Run(ctx, r.client, []string{key}, expected, newValue).Result()
    if err == redis.Nil {
        return false, nil
    }
    return err == nil, err
}

func (r *Redis) IncrWithCap(ctx context.Context, key string, cap, incr int64) (int64, error) {
    return scriptIncrWithCap.Run(ctx, r.client, []string{key}, cap, incr).Int64()
}
```

### 集群兼容

```go
// 使用 Hash Tag 确保键在同一个槽
func clusterSafeKey(resource string) string {
    return fmt.Sprintf("{%s}:lock", resource)
}

// 多键操作使用相同 Hash Tag
var multiKeyScript = redis.NewScript(`
    local prefix = KEYS[1]
    redis.call("set", prefix .. ":a", ARGV[1])
    redis.call("set", prefix .. ":b", ARGV[2])
    return "OK"
`)

// 调用时 KEYS[1] = "{resource}"
// 实际操作 {resource}:a 和 {resource}:b 在同一槽
```

---

## 最佳实践

### 连接管理

- 复用连接池
- 合理配置 PoolSize（建议 CPU 核数 * 10）
- 设置 ConnMaxIdleTime 清理空闲连接

### 键设计

- 使用冒号分隔命名空间 `user:123:profile`
- 集群模式使用 Hash Tag `{user:123}:profile`
- 设置合理 TTL，避免内存溢出

### 性能

- 批量操作使用 Pipeline
- 大键拆分（Hash 字段 < 1000）
- 避免 KEYS 命令（用 SCAN）

### 可靠性

- 分布式锁使用 Redlock
- 设置合理重试策略
- 监控慢查询

---

## 检查清单

- [ ] 连接池配置合理？
- [ ] 键命名规范？
- [ ] 设置 TTL？
- [ ] 批量操作用 Pipeline？
- [ ] 分布式锁用 Lua 原子操作？
- [ ] 集群模式用 Hash Tag？
- [ ] 空值缓存防穿透？
- [ ] Singleflight 防击穿？
