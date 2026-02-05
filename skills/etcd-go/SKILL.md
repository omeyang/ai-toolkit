---
name: etcd-go
description: Go etcd 专家 - KV 存储、Watch 监听、分布式锁、租约管理、选主。使用场景：配置中心、服务发现、分布式协调。
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

# Go etcd 专家

使用 Go etcd client 开发分布式协调功能：$ARGUMENTS

---

## 1. 客户端管理

### 创建客户端

```go
import (
    "context"
    "time"

    clientv3 "go.etcd.io/etcd/client/v3"
)

func NewEtcdClient(endpoints []string) (*clientv3.Client, error) {
    client, err := clientv3.New(clientv3.Config{
        Endpoints:   endpoints,
        DialTimeout: 5 * time.Second,
        // 认证（可选）
        // Username: "root",
        // Password: "password",
        // TLS（可选）
        // TLS: tlsConfig,
    })
    if err != nil {
        return nil, fmt.Errorf("create etcd client: %w", err)
    }

    // 验证连接
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    _, err = client.Status(ctx, endpoints[0])
    if err != nil {
        return nil, fmt.Errorf("check etcd status: %w", err)
    }

    return client, nil
}
```

### 包装器模式

```go
type Etcd struct {
    client *clientv3.Client
    prefix string
}

func New(client *clientv3.Client, prefix string) *Etcd {
    return &Etcd{
        client: client,
        prefix: prefix,
    }
}

// Client 暴露底层客户端
func (e *Etcd) Client() *clientv3.Client {
    return e.client
}

// Close 关闭客户端
func (e *Etcd) Close() error {
    return e.client.Close()
}

// 构建完整 key
func (e *Etcd) key(key string) string {
    return e.prefix + key
}
```

---

## 2. KV 操作

### 基本 CRUD

```go
// Put
func (e *Etcd) Put(ctx context.Context, key, value string) error {
    _, err := e.client.Put(ctx, e.key(key), value)
    return err
}

// PutWithTTL（带过期时间）
func (e *Etcd) PutWithTTL(ctx context.Context, key, value string, ttl time.Duration) error {
    lease, err := e.client.Grant(ctx, int64(ttl.Seconds()))
    if err != nil {
        return fmt.Errorf("grant lease: %w", err)
    }

    _, err = e.client.Put(ctx, e.key(key), value, clientv3.WithLease(lease.ID))
    return err
}

// Get
func (e *Etcd) Get(ctx context.Context, key string) (string, error) {
    resp, err := e.client.Get(ctx, e.key(key))
    if err != nil {
        return "", fmt.Errorf("get: %w", err)
    }

    if len(resp.Kvs) == 0 {
        return "", ErrKeyNotFound
    }

    return string(resp.Kvs[0].Value), nil
}

// GetWithPrefix（前缀查询）
func (e *Etcd) GetWithPrefix(ctx context.Context, prefix string) (map[string]string, error) {
    resp, err := e.client.Get(ctx, e.key(prefix), clientv3.WithPrefix())
    if err != nil {
        return nil, fmt.Errorf("get with prefix: %w", err)
    }

    result := make(map[string]string, len(resp.Kvs))
    for _, kv := range resp.Kvs {
        result[string(kv.Key)] = string(kv.Value)
    }

    return result, nil
}

// Delete
func (e *Etcd) Delete(ctx context.Context, key string) error {
    _, err := e.client.Delete(ctx, e.key(key))
    return err
}

// DeleteWithPrefix
func (e *Etcd) DeleteWithPrefix(ctx context.Context, prefix string) (int64, error) {
    resp, err := e.client.Delete(ctx, e.key(prefix), clientv3.WithPrefix())
    if err != nil {
        return 0, fmt.Errorf("delete with prefix: %w", err)
    }
    return resp.Deleted, nil
}
```

### 原子操作

```go
// CompareAndSwap（CAS）
func (e *Etcd) CompareAndSwap(ctx context.Context, key, oldValue, newValue string) (bool, error) {
    txn := e.client.Txn(ctx)

    resp, err := txn.If(
        clientv3.Compare(clientv3.Value(e.key(key)), "=", oldValue),
    ).Then(
        clientv3.OpPut(e.key(key), newValue),
    ).Commit()

    if err != nil {
        return false, fmt.Errorf("cas: %w", err)
    }

    return resp.Succeeded, nil
}

// PutIfAbsent（不存在则创建）
func (e *Etcd) PutIfAbsent(ctx context.Context, key, value string) (bool, error) {
    txn := e.client.Txn(ctx)

    resp, err := txn.If(
        clientv3.Compare(clientv3.CreateRevision(e.key(key)), "=", 0),
    ).Then(
        clientv3.OpPut(e.key(key), value),
    ).Commit()

    if err != nil {
        return false, fmt.Errorf("put if absent: %w", err)
    }

    return resp.Succeeded, nil
}
```

---

## 3. Watch 监听

### 监听单个 Key

```go
func (e *Etcd) Watch(ctx context.Context, key string, handler func(clientv3.Event)) error {
    watchChan := e.client.Watch(ctx, e.key(key))

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case resp := <-watchChan:
            if resp.Canceled {
                return fmt.Errorf("watch canceled: %v", resp.Err())
            }
            for _, event := range resp.Events {
                handler(*event)
            }
        }
    }
}
```

### 监听前缀

```go
func (e *Etcd) WatchPrefix(ctx context.Context, prefix string, handler func(clientv3.Event)) error {
    watchChan := e.client.Watch(ctx, e.key(prefix), clientv3.WithPrefix())

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case resp := <-watchChan:
            if resp.Canceled {
                return fmt.Errorf("watch canceled: %v", resp.Err())
            }
            for _, event := range resp.Events {
                handler(*event)
            }
        }
    }
}
```

### Watch 处理器

```go
type WatchHandler struct {
    OnPut    func(key, value string)
    OnDelete func(key string)
}

func (e *Etcd) WatchWithHandler(ctx context.Context, prefix string, handler WatchHandler) error {
    return e.WatchPrefix(ctx, prefix, func(event clientv3.Event) {
        key := string(event.Kv.Key)

        switch event.Type {
        case clientv3.EventTypePut:
            if handler.OnPut != nil {
                handler.OnPut(key, string(event.Kv.Value))
            }
        case clientv3.EventTypeDelete:
            if handler.OnDelete != nil {
                handler.OnDelete(key)
            }
        }
    })
}

// 使用示例
e.WatchWithHandler(ctx, "/config/", WatchHandler{
    OnPut: func(key, value string) {
        log.Printf("Config updated: %s = %s", key, value)
    },
    OnDelete: func(key string) {
        log.Printf("Config deleted: %s", key)
    },
})
```

---

## 4. 分布式锁

### 基本锁

```go
import "go.etcd.io/etcd/client/v3/concurrency"

func (e *Etcd) Lock(ctx context.Context, lockKey string, ttl int) (func() error, error) {
    // 创建 session
    session, err := concurrency.NewSession(e.client, concurrency.WithTTL(ttl))
    if err != nil {
        return nil, fmt.Errorf("create session: %w", err)
    }

    // 创建 mutex
    mutex := concurrency.NewMutex(session, e.key(lockKey))

    // 获取锁
    if err := mutex.Lock(ctx); err != nil {
        session.Close()
        return nil, fmt.Errorf("acquire lock: %w", err)
    }

    // 返回释放函数
    unlock := func() error {
        defer session.Close()
        return mutex.Unlock(ctx)
    }

    return unlock, nil
}

// 使用示例
unlock, err := e.Lock(ctx, "/locks/my-task", 30)
if err != nil {
    return err
}
defer unlock()

// 执行需要互斥的操作
doWork()
```

### TryLock（非阻塞）

```go
func (e *Etcd) TryLock(ctx context.Context, lockKey string, ttl int) (func() error, bool, error) {
    session, err := concurrency.NewSession(e.client, concurrency.WithTTL(ttl))
    if err != nil {
        return nil, false, fmt.Errorf("create session: %w", err)
    }

    mutex := concurrency.NewMutex(session, e.key(lockKey))

    // 使用 TryLock
    if err := mutex.TryLock(ctx); err != nil {
        session.Close()
        if err == concurrency.ErrLocked {
            return nil, false, nil // 锁被占用
        }
        return nil, false, err
    }

    unlock := func() error {
        defer session.Close()
        return mutex.Unlock(ctx)
    }

    return unlock, true, nil
}
```

### 带超时的锁

```go
func (e *Etcd) LockWithTimeout(ctx context.Context, lockKey string, ttl int, timeout time.Duration) (func() error, error) {
    ctx, cancel := context.WithTimeout(ctx, timeout)
    defer cancel()

    return e.Lock(ctx, lockKey, ttl)
}
```

---

## 5. 租约管理

### 创建租约

```go
func (e *Etcd) GrantLease(ctx context.Context, ttl int64) (clientv3.LeaseID, error) {
    resp, err := e.client.Grant(ctx, ttl)
    if err != nil {
        return 0, fmt.Errorf("grant lease: %w", err)
    }
    return resp.ID, nil
}
```

### KeepAlive（续租）

```go
func (e *Etcd) KeepAlive(ctx context.Context, leaseID clientv3.LeaseID) (<-chan *clientv3.LeaseKeepAliveResponse, error) {
    return e.client.KeepAlive(ctx, leaseID)
}

// 自动续租
func (e *Etcd) StartKeepAlive(ctx context.Context, leaseID clientv3.LeaseID) error {
    ch, err := e.client.KeepAlive(ctx, leaseID)
    if err != nil {
        return err
    }

    go func() {
        for {
            select {
            case <-ctx.Done():
                return
            case resp := <-ch:
                if resp == nil {
                    log.Println("lease expired")
                    return
                }
            }
        }
    }()

    return nil
}
```

### 服务注册（使用租约）

```go
type ServiceRegistry struct {
    etcd    *Etcd
    leaseID clientv3.LeaseID
}

func (e *Etcd) RegisterService(ctx context.Context, serviceName, instanceID, addr string, ttl int64) (*ServiceRegistry, error) {
    // 创建租约
    leaseID, err := e.GrantLease(ctx, ttl)
    if err != nil {
        return nil, err
    }

    // 注册服务
    key := fmt.Sprintf("/services/%s/%s", serviceName, instanceID)
    _, err = e.client.Put(ctx, key, addr, clientv3.WithLease(leaseID))
    if err != nil {
        return nil, fmt.Errorf("register service: %w", err)
    }

    // 启动续租
    if err := e.StartKeepAlive(ctx, leaseID); err != nil {
        return nil, err
    }

    return &ServiceRegistry{
        etcd:    e,
        leaseID: leaseID,
    }, nil
}

func (r *ServiceRegistry) Deregister(ctx context.Context) error {
    _, err := r.etcd.client.Revoke(ctx, r.leaseID)
    return err
}
```

---

## 6. 选主（Leader Election）

```go
func (e *Etcd) Campaign(ctx context.Context, electionKey, value string, ttl int) (*concurrency.Election, error) {
    session, err := concurrency.NewSession(e.client, concurrency.WithTTL(ttl))
    if err != nil {
        return nil, fmt.Errorf("create session: %w", err)
    }

    election := concurrency.NewElection(session, e.key(electionKey))

    // 参与选举（阻塞直到成为 leader）
    if err := election.Campaign(ctx, value); err != nil {
        session.Close()
        return nil, fmt.Errorf("campaign: %w", err)
    }

    return election, nil
}

// 观察 leader 变化
func (e *Etcd) Observe(ctx context.Context, electionKey string) (<-chan clientv3.GetResponse, error) {
    session, err := concurrency.NewSession(e.client)
    if err != nil {
        return nil, err
    }

    election := concurrency.NewElection(session, e.key(electionKey))
    return election.Observe(ctx), nil
}

// 获取当前 leader
func (e *Etcd) GetLeader(ctx context.Context, electionKey string) (string, error) {
    session, err := concurrency.NewSession(e.client)
    if err != nil {
        return "", err
    }
    defer session.Close()

    election := concurrency.NewElection(session, e.key(electionKey))
    resp, err := election.Leader(ctx)
    if err != nil {
        return "", err
    }

    if len(resp.Kvs) == 0 {
        return "", ErrNoLeader
    }

    return string(resp.Kvs[0].Value), nil
}

// 主动放弃 leader
func (e *Etcd) Resign(ctx context.Context, election *concurrency.Election) error {
    return election.Resign(ctx)
}
```

### 选主使用示例

```go
func runAsLeader(ctx context.Context, e *Etcd) error {
    election, err := e.Campaign(ctx, "/election/scheduler", "node-1", 10)
    if err != nil {
        return err
    }
    defer election.Resign(ctx)

    log.Println("I am the leader now!")

    // 作为 leader 执行任务
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            doLeaderWork()
            time.Sleep(time.Second)
        }
    }
}
```

---

## 7. 事务

```go
func (e *Etcd) Transaction(ctx context.Context, ops []clientv3.Op, conditions ...clientv3.Cmp) (bool, error) {
    txn := e.client.Txn(ctx)

    if len(conditions) > 0 {
        txn = txn.If(conditions...)
    }

    resp, err := txn.Then(ops...).Commit()
    if err != nil {
        return false, fmt.Errorf("transaction: %w", err)
    }

    return resp.Succeeded, nil
}

// 使用示例：原子更新多个 key
func (e *Etcd) UpdateMultiple(ctx context.Context, kvs map[string]string) error {
    ops := make([]clientv3.Op, 0, len(kvs))
    for k, v := range kvs {
        ops = append(ops, clientv3.OpPut(e.key(k), v))
    }

    _, err := e.Transaction(ctx, ops)
    return err
}
```

---

## 8. 配置中心

```go
type ConfigCenter struct {
    etcd    *Etcd
    configs sync.Map
}

func NewConfigCenter(etcd *Etcd) *ConfigCenter {
    return &ConfigCenter{etcd: etcd}
}

// 加载配置
func (c *ConfigCenter) Load(ctx context.Context, prefix string) error {
    configs, err := c.etcd.GetWithPrefix(ctx, prefix)
    if err != nil {
        return err
    }

    for k, v := range configs {
        c.configs.Store(k, v)
    }

    return nil
}

// 监听配置变化
func (c *ConfigCenter) Watch(ctx context.Context, prefix string) error {
    return c.etcd.WatchWithHandler(ctx, prefix, WatchHandler{
        OnPut: func(key, value string) {
            c.configs.Store(key, value)
            log.Printf("Config updated: %s", key)
        },
        OnDelete: func(key string) {
            c.configs.Delete(key)
            log.Printf("Config deleted: %s", key)
        },
    })
}

// 获取配置
func (c *ConfigCenter) Get(key string) (string, bool) {
    value, ok := c.configs.Load(key)
    if !ok {
        return "", false
    }
    return value.(string), true
}
```

---

## 最佳实践

### 连接管理

- 使用多个 endpoints 实现高可用
- 配置合理的 DialTimeout
- 正确处理连接断开和重连

### Key 设计

- 使用前缀组织 key（如 `/app/config/`）
- 避免 key 过长
- 使用目录结构表达层级关系

### 租约

- 服务注册使用租约自动清理
- 合理设置 TTL（不要太短导致频繁续租）
- 正确处理续租失败

### 分布式锁

- 使用 concurrency 包而非手动实现
- 设置合理的锁超时
- 确保锁释放（使用 defer）

---

## 检查清单

- [ ] 配置多个 endpoints？
- [ ] 正确处理连接超时？
- [ ] Key 使用前缀组织？
- [ ] 使用租约自动清理？
- [ ] 分布式锁正确释放？
- [ ] Watch 处理重连？
- [ ] 事务保证原子性？
- [ ] 优雅关闭客户端？
