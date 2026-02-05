---
name: mongodb-go
description: "Go MongoDB 专家 - 使用 mongo-driver v2 进行 CRUD、聚合管道、索引优化、事务处理、分页查询(Offset/游标)、批量写入、Change Streams。适用：数据库设计、查询优化、性能调优、Schema 建模、聚合统计。不适用：关系型数据库(PostgreSQL/MySQL)操作；Redis 缓存操作；搜索引擎(Elasticsearch)场景。触发词：mongodb, mongo, bson, 聚合, aggregate, 索引, index, collection, 分页, cursor, change-stream, 事务"
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

# Go MongoDB 专家

使用 Go mongo-driver 开发 MongoDB 功能：$ARGUMENTS

---

## 1. 连接管理

### 创建客户端

```go
func NewMongoClient(ctx context.Context, uri string) (*mongo.Client, error) {
    opts := options.Client().
        ApplyURI(uri).
        SetMaxPoolSize(100).
        SetMinPoolSize(10).
        SetMaxConnIdleTime(30 * time.Minute).
        SetServerSelectionTimeout(5 * time.Second).
        SetConnectTimeout(10 * time.Second)

    client, err := mongo.Connect(opts)
    if err != nil {
        return nil, fmt.Errorf("connect mongo: %w", err)
    }

    if err := client.Ping(ctx, readpref.Primary()); err != nil {
        return nil, fmt.Errorf("ping mongo: %w", err)
    }
    return client, nil
}
```

### 包装器模式

不包装所有 API，只提供增值功能。暴露 `Client()` 和 `Collection()` 用于高级操作。

```go
type MongoDB struct {
    client *mongo.Client
    db     *mongo.Database
}

func New(client *mongo.Client, dbName string) *MongoDB
func (m *MongoDB) Client() *mongo.Client
func (m *MongoDB) Collection(name string) *mongo.Collection
func (m *MongoDB) Health(ctx context.Context) error
func (m *MongoDB) Close(ctx context.Context) error
```

> 完整包装器实现见 [references/examples.md](references/examples.md#包装器模式实现)

---

## 2. CRUD 操作

### 核心方法签名

```go
func (m *MongoDB) InsertOne(ctx, coll string, doc any) (string, error)
func (m *MongoDB) InsertMany(ctx, coll string, docs []any) ([]string, error)
func (m *MongoDB) FindOne(ctx, coll string, filter bson.M, result any) error
func (m *MongoDB) Find(ctx, coll string, filter bson.M, results any, opts ...*options.FindOptions) error
func (m *MongoDB) FindByID(ctx, coll string, id string, result any) error
func (m *MongoDB) UpdateOne(ctx, coll string, filter, update bson.M) (int64, error)
func (m *MongoDB) UpdateMany(ctx, coll string, filter, update bson.M) (int64, error)
func (m *MongoDB) Upsert(ctx, coll string, filter, update bson.M) error
func (m *MongoDB) DeleteOne(ctx, coll string, filter bson.M) (int64, error)
func (m *MongoDB) DeleteMany(ctx, coll string, filter bson.M) (int64, error)
func (m *MongoDB) SoftDelete(ctx, coll string, filter bson.M) error  // 推荐
```

> 完整 CRUD 实现见 [references/examples.md](references/examples.md#crud-操作实现)

---

## 3. 查询操作符

### 比较操作符

```go
bson.M{"age": bson.M{"$gt": 18}}           // 大于
bson.M{"age": bson.M{"$gte": 18}}          // 大于等于
bson.M{"status": bson.M{"$ne": "deleted"}} // 不等于
bson.M{"role": bson.M{"$in": []string{"admin", "moderator"}}} // 在数组中
```

### 逻辑操作符

```go
bson.M{"$and": []bson.M{{...}, {...}}}  // AND
bson.M{"$or": []bson.M{{...}, {...}}}   // OR
bson.M{"age": bson.M{"$not": bson.M{"$lt": 18}}} // NOT
```

### 数组/正则

```go
bson.M{"tags": "golang"}                                     // 包含元素
bson.M{"tags": bson.M{"$all": []string{"golang", "mongodb"}}} // 包含所有
bson.M{"name": bson.M{"$regex": "^john", "$options": "i"}}   // 正则匹配
```

> 完整操作符参考见 [references/examples.md](references/examples.md#查询操作符参考)

---

## 4. 聚合管道

### 基本聚合

```go
func (m *MongoDB) Aggregate(ctx, coll string, pipeline mongo.Pipeline, results any) error
```

### 常用阶段

| 阶段 | 作用 |
|------|------|
| `$match` | 筛选文档 |
| `$group` | 分组聚合（sum, avg, push） |
| `$sort` | 排序 |
| `$limit` | 限制数量 |
| `$lookup` | 关联查询（类似 JOIN） |
| `$unwind` | 展开数组 |
| `$project` | 投影/计算字段 |

> 完整聚合示例见 [references/examples.md](references/examples.md#聚合管道示例)

---

## 5. 索引管理

### 索引类型

```go
// 单字段唯一索引
m.CreateIndex(ctx, "users", bson.D{{"email", 1}}, options.Index().SetUnique(true))

// 复合索引
m.CreateIndex(ctx, "users", bson.D{{"status", 1}, {"created_at", -1}}, nil)

// 文本索引
m.CreateIndex(ctx, "articles", bson.D{{"title", "text"}, {"content", "text"}}, nil)

// TTL 索引（自动过期）
m.CreateIndex(ctx, "sessions", bson.D{{"expires_at", 1}},
    options.Index().SetExpireAfterSeconds(0))

// 部分索引
m.CreateIndex(ctx, "users", bson.D{{"email", 1}},
    options.Index().SetPartialFilterExpression(bson.M{"status": "active"}))
```

> 完整索引管理实现见 [references/examples.md](references/examples.md#索引管理实现)

---

## 6. 事务处理

单文档操作自动具有原子性（MongoDB 4.0+）。多文档事务：

```go
func (m *MongoDB) WithTransaction(ctx context.Context, fn func(sc mongo.SessionContext) error) error
```

> 完整事务实现（含转账示例）见 [references/examples.md](references/examples.md#事务处理实现)

---

## 7. 分页查询

### Offset 分页

适合数据量较小场景，支持跳转到任意页。

```go
func (m *MongoDB) FindPage[T any](ctx, coll string, filter bson.M, opts PageOptions) (*PageResult[T], error)
```

### 游标分页（推荐大数据量）

基于 `_id` 游标，避免 COUNT 和 SKIP 开销。

```go
func (m *MongoDB) FindAfter[T any](ctx, coll string, filter bson.M, afterID string, limit int64) ([]T, error)
```

> 完整分页实现见 [references/examples.md](references/examples.md#分页查询实现)

---

## 8. 批量写入

分批次 BulkWrite，支持 ordered/unordered，检查 context 取消。

```go
func (m *MongoDB) BulkWrite(ctx, coll string, docs []any, opts BulkOptions) error
```

> 完整批量写入实现见 [references/examples.md](references/examples.md#批量写入实现)

---

## 9. Change Streams

实时监听集合变更，支持 insert/update/delete 事件过滤。

```go
func (m *MongoDB) Watch(ctx, coll string, pipeline mongo.Pipeline, handler func(bson.M)) error
```

> 完整 Change Streams 实现见 [references/examples.md](references/examples.md#change-streams-实现)

---

## 10. 最佳实践

### Schema 设计

使用结构体定义 schema，善用 `bson` tag (`omitempty`, 嵌入文档, 软删除字段)。

### 错误处理

```go
func handleMongoError(err error) error  // 区分 NotFound/DuplicateKey/Timeout
```

### 索引策略

启动时 `EnsureIndexes` 确保索引存在。

> 完整最佳实践代码见 [references/examples.md](references/examples.md#最佳实践实现)

---

## 常用命令

```bash
mongosh "mongodb://localhost:27017"           # 连接
db.users.getIndexes()                         # 查看索引
db.users.find({email: "test@example.com"}).explain("executionStats")  # 查询计划
```

---

## 检查清单

- [ ] 连接池配置合理？
- [ ] 使用结构体定义 schema？
- [ ] 常用查询字段有索引？
- [ ] 大数据量使用游标分页？
- [ ] 多文档操作使用事务？
- [ ] 错误正确处理？
- [ ] 支持 context 取消？
- [ ] 软删除而非硬删除？

---

## 参考资料

- [references/examples.md](references/examples.md) - 完整代码实现（连接、CRUD、操作符、聚合、索引、事务、分页、批量写入、Change Streams、最佳实践）
