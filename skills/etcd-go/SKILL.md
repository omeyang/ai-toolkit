---
name: etcd-go
description: "Go etcd 专家 - KV 存储、Watch 监听、分布式锁、租约管理、选主、事务、配置中心。适用：服务发现与注册、分布式配置中心、分布式锁与协调、选主（leader election）、少量关键元数据存储。不适用：大量数据存储（etcd 上限约 8GB）、高频写入场景（Raft 共识写入延迟高）、缓存场景（应使用 Redis）。触发词：etcd, distributed lock, leader election, watch, lease, service discovery, 分布式锁, 选主, 配置中心, 服务发现, KV"
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

# Go etcd 专家

使用 Go etcd client 开发分布式协调功能：$ARGUMENTS

---

## 1. 客户端管理

### 创建客户端

```go
func NewEtcdClient(endpoints []string) (*clientv3.Client, error) {
    client, err := clientv3.New(clientv3.Config{
        Endpoints:   endpoints,
        DialTimeout: 5 * time.Second,
    })
    if err != nil {
        return nil, fmt.Errorf("create etcd client: %w", err)
    }

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

- `Etcd{client, prefix}` - 封装客户端 + key 前缀
- `key(key string) string` - 构建完整 key = prefix + key
- 哨兵错误：`ErrKeyNotFound`、`ErrNoLeader`

---

## 2. KV 操作

### 基本 CRUD

- `Put(ctx, key, value)` - 写入
- `PutWithTTL(ctx, key, value, ttl)` - 带过期时间写入（Grant lease + WithLease）
- `Get(ctx, key)` - 读取单个 key
- `GetWithPrefix(ctx, prefix)` - 前缀查询，返回 `map[string]string`
- `Delete(ctx, key)` / `DeleteWithPrefix(ctx, prefix)` - 删除

### 原子操作

- `CompareAndSwap(ctx, key, oldValue, newValue)` - CAS 操作（Txn + Compare Value）
- `PutIfAbsent(ctx, key, value)` - 不存在则创建（Txn + Compare CreateRevision == 0）

---

## 3. Watch 监听

- `Watch(ctx, key, handler)` - 监听单个 key 变化
- `WatchPrefix(ctx, prefix, handler)` - 监听前缀下所有 key
- `WatchWithHandler(ctx, prefix, WatchHandler{OnPut, OnDelete})` - 结构化处理器，区分 Put/Delete 事件

---

## 4. 分布式锁

- `Lock(ctx, lockKey, ttl)` - 基本锁，返回 unlock 函数（使用 concurrency 包）
- `TryLock(ctx, lockKey, ttl)` - 非阻塞锁，返回 (unlock, acquired, error)
- `LockWithTimeout(ctx, lockKey, ttl, timeout)` - 带超时的锁
- **注意**：unlock 使用独立 context，避免原始 ctx 取消后无法释放锁

---

## 5. 租约管理

- `GrantLease(ctx, ttl)` - 创建租约
- `KeepAlive(ctx, leaseID)` / `StartKeepAlive(ctx, leaseID)` - 自动续租（goroutine drain channel 防泄漏）
- `RegisterService(ctx, serviceName, instanceID, addr, ttl)` - 服务注册（lease + keepalive）
- `Deregister(ctx)` - 服务注销（Revoke lease）

---

## 6. 选主（Leader Election）

- `Campaign(ctx, electionKey, value, ttl)` - 参与选举（阻塞直到成为 leader）
- `Observe(ctx, electionKey)` - 观察 leader 变化（返回 channel + cleanup 函数）
- `GetLeader(ctx, electionKey)` - 获取当前 leader
- `Resign(ctx, election)` - 主动放弃 leader（使用独立 context）

---

## 7. 事务

- `Transaction(ctx, ops, conditions)` - 通用事务（If-Then 模式）
- `UpdateMultiple(ctx, kvs)` - 原子更新多个 key

---

## 8. 配置中心

- `ConfigCenter{etcd, configs sync.Map}` - 基于 etcd 的配置中心
- `Load(ctx, prefix)` - 加载配置
- `Watch(ctx, prefix)` - 监听配置变化（自动更新本地缓存）
- `Get(key)` - 获取配置（从 sync.Map 读取）

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

## 参考资料

- [完整代码实现](references/examples.md) - KV 操作、Watch、分布式锁、租约、选主、事务、配置中心的完整 Go 实现
