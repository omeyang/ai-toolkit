---
name: kafka-go
description: "Go Kafka 专家 - 使用 confluent-kafka-go 进行生产消费(同步/异步/批量)、分区策略、消费者组(Rebalance)、Exactly-Once 事务、死信队列(DLQ)、OpenTelemetry 链路追踪。适用：事件驱动架构、日志收集、流处理管道、消息顺序保证。不适用：低延迟 RPC(用 gRPC)；小规模简单队列(用 Redis Pub/Sub)；复杂路由/延迟队列(用 RabbitMQ/Pulsar)。触发词：kafka, 消息队列, producer, consumer, 消费者组, 分区, partition, DLQ, 死信, exactly-once, 事务, confluent"
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

# Go Kafka 专家

使用 Go confluent-kafka-go 开发 Kafka 功能：$ARGUMENTS

---

## 1. 客户端管理

### 创建生产者

```go
func NewKafkaProducer(brokers string) (*kafka.Producer, error) {
    return kafka.NewProducer(&kafka.ConfigMap{
        "bootstrap.servers":   brokers,
        "acks":                "all",
        "enable.idempotence":  true,
        "retries":             3,
        "linger.ms":           5,
        "batch.size":          16384,
        "compression.type":    "lz4",
        "max.in.flight.requests.per.connection": 5,
    })
}
```

### 创建消费者

```go
func NewKafkaConsumer(brokers, groupID string) (*kafka.Consumer, error) {
    return kafka.NewConsumer(&kafka.ConfigMap{
        "bootstrap.servers":  brokers,
        "group.id":           groupID,
        "auto.offset.reset":  "earliest",
        "enable.auto.commit": false,  // 手动提交
        "session.timeout.ms": 30000,
        "max.poll.interval.ms": 300000,
    })
}
```

### 包装器模式

```go
type Kafka struct { producer *kafka.Producer }

func New(producer *kafka.Producer) *Kafka
func (k *Kafka) Producer() *kafka.Producer
func (k *Kafka) Close()  // Flush + Close
```

---

## 2. 生产者

### 同步发送

使用 deliveryChan 等待确认，支持 context 超时。注意：不要 close(deliveryChan)，Kafka 内部会写入。

```go
func (k *Kafka) Send(ctx context.Context, topic string, key, value []byte) error
```

### 异步发送

结果通过 `producer.Events()` channel 投递，需单独处理投递报告。

```go
func (k *Kafka) SendAsync(ctx context.Context, topic string, key, value []byte) error
func (k *Kafka) HandleDeliveryReports(ctx context.Context)
```

### 批量发送

```go
func (k *Kafka) SendBatch(ctx context.Context, messages []Message) error
```

### 分区策略

```go
func (k *Kafka) SendToPartition(ctx, topic string, partition int32, key, value []byte) error
func (k *Kafka) SendWithKey(ctx, topic, key string, value []byte) error  // 相同 Key 到同一分区
```

> 完整生产者实现见 [references/examples.md](references/examples.md#生产者实现)

---

## 3. 消费者

### 基础消费

手动提交 offset，处理超时错误，支持 context 取消。

```go
type Handler func(ctx context.Context, msg *kafka.Message) error

func (k *Kafka) Consume(ctx, consumer, topics []string, handler Handler) error
```

### 批量消费

按 batchSize 或 timeout 触发处理，提交最后一条 offset。

```go
func (k *Kafka) ConsumeBatch(ctx, consumer, topics, batchSize int, timeout, handler) error
```

### 并发消费

worker pool 模式，注意并发时 offset 提交需要额外追踪。

```go
func (k *Kafka) ConsumeParallel(ctx, consumer, topics, workers int, handler) error
```

> 完整消费者实现见 [references/examples.md](references/examples.md#消费者实现)

---

## 4. 死信队列（DLQ）

超过 MaxRetries 后发送到 DLQ topic，附加原始 topic、错误信息、时间戳 headers。

```go
type DLQConfig struct {
    Topic      string
    MaxRetries int
}

func (c *ConsumerWithDLQ) ConsumeWithDLQ(ctx, topics, handler) error
```

> 完整 DLQ 实现见 [references/examples.md](references/examples.md#死信队列实现)

---

## 5. 链路追踪

### OpenTelemetry 集成

通过 Kafka headers 传播 trace context。

```go
func headersFromContext(ctx context.Context) []kafka.Header   // 注入
func contextFromHeaders(ctx context.Context, headers) context.Context  // 提取

func (k *Kafka) SendWithTrace(ctx, topic, key, value) error
func (k *Kafka) ConsumeWithTrace(ctx, consumer, topics, handler) error
```

> 完整追踪实现见 [references/examples.md](references/examples.md#链路追踪实现)

---

## 6. 事务（Exactly-Once）

使用 transactional.id + 幂等生产者实现 Exactly-Once 语义。

```go
func NewTransactionalProducer(brokers, transactionalID string) (*kafka.Producer, error)
func (k *Kafka) SendInTransaction(ctx context.Context, messages []Message) error
```

> 完整事务实现见 [references/examples.md](references/examples.md#事务实现)

---

## 7. 消费者组管理

### Rebalance 回调

分区分配/撤销时自动提交 offset。

```go
func (k *Kafka) ConsumeWithRebalance(ctx, consumer, topics, handler) error
```

### 手动分区分配

```go
func (k *Kafka) AssignPartitions(consumer, topic string, partitions []int32) error
```

> 完整消费者组管理见 [references/examples.md](references/examples.md#消费者组管理实现)

---

## 8. 健康检查

```go
func (k *Kafka) Health(ctx context.Context) error       // 检查 broker 可用
func (k *Kafka) TopicExists(topic string) (bool, error) // 检查 topic 存在
```

> 完整健康检查实现见 [references/examples.md](references/examples.md#健康检查实现)

---

## 最佳实践

### 生产者
- 使用 `acks=all` 确保持久性
- 启用幂等生产者 (`enable.idempotence=true`)
- 使用 Key 保证相关消息顺序
- 处理投递回调，记录失败消息

### 消费者
- 手动提交 offset（`enable.auto.commit=false`）
- 实现幂等消费（消息可能重复）
- 合理设置 `max.poll.interval.ms`
- 处理 rebalance 回调

### 可靠性
- 实现 DLQ 处理持续失败消息
- 记录消息处理状态
- 监控消费者 lag

### 性能
- 批量发送和消费
- 启用压缩 (lz4/snappy)
- 调整 `linger.ms` 和 `batch.size`

---

## 检查清单

- [ ] 生产者启用幂等？
- [ ] 消费者手动提交？
- [ ] 实现 DLQ？
- [ ] 集成链路追踪？
- [ ] 处理 rebalance？
- [ ] 设置合理超时？
- [ ] 监控消费者 lag？
- [ ] 消费逻辑幂等？

---

## 参考资料

- [references/examples.md](references/examples.md) - 完整代码实现（生产者、消费者、DLQ、链路追踪、事务、消费者组、健康检查）
