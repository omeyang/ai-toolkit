---
name: design-patterns
description: 设计模式与架构专家 - 覆盖 GoF 经典模式、云架构模式、Go 并发模式、SOLID/DDD 等。使用场景：系统设计、代码重构、架构决策、分布式系统。
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

# 设计模式与架构专家

使用设计模式解决：$ARGUMENTS

---

## 第一部分：GoF 经典设计模式

### 创建型模式（5种）

#### 1. 单例模式（Singleton）
**意图**：确保一个类只有一个实例，并提供全局访问点

```go
// Go 惯用方式：sync.Once
var (
    instance *Database
    once     sync.Once
)

func GetDatabase() *Database {
    once.Do(func() {
        instance = &Database{/* 初始化 */}
    })
    return instance
}
```

**适用场景**：配置管理、连接池、日志记录器
**注意**：被认为是反模式，优先考虑依赖注入

#### 2. 工厂方法（Factory Method）
**意图**：定义创建对象的接口，让子类决定实例化哪个类

```go
type Storage interface {
    Save(key string, data []byte) error
    Load(key string) ([]byte, error)
}

func NewStorage(typ string) (Storage, error) {
    switch typ {
    case "file":
        return &FileStorage{}, nil
    case "redis":
        return &RedisStorage{}, nil
    case "s3":
        return &S3Storage{}, nil
    default:
        return nil, fmt.Errorf("unknown storage: %s", typ)
    }
}
```

**适用场景**：根据配置创建不同实现、插件系统

#### 3. 抽象工厂（Abstract Factory）
**意图**：创建相关或依赖对象的家族，而不指定具体类

```go
type UIFactory interface {
    CreateButton() Button
    CreateDialog() Dialog
    CreateMenu() Menu
}

type DarkThemeFactory struct{}
type LightThemeFactory struct{}

func (f *DarkThemeFactory) CreateButton() Button {
    return &DarkButton{}
}
```

**适用场景**：跨平台 UI、主题切换、产品族

#### 4. 建造者模式（Builder）
**意图**：将复杂对象的构建与表示分离

```go
type ServerBuilder struct {
    server Server
}

func NewServerBuilder() *ServerBuilder {
    return &ServerBuilder{server: Server{Port: 8080}}
}

func (b *ServerBuilder) Host(h string) *ServerBuilder {
    b.server.Host = h
    return b
}

func (b *ServerBuilder) Port(p int) *ServerBuilder {
    b.server.Port = p
    return b
}

func (b *ServerBuilder) WithTLS(cert, key string) *ServerBuilder {
    b.server.TLS = &TLSConfig{Cert: cert, Key: key}
    return b
}

func (b *ServerBuilder) Build() (*Server, error) {
    if err := b.server.validate(); err != nil {
        return nil, err
    }
    return &b.server, nil
}

// 使用
server, _ := NewServerBuilder().
    Host("0.0.0.0").
    Port(443).
    WithTLS("cert.pem", "key.pem").
    Build()
```

#### 5. 函数选项模式（Functional Options）— Go 惯用
**意图**：灵活配置，避免构造函数参数爆炸

```go
type Option func(*Config)

func WithTimeout(d time.Duration) Option {
    return func(c *Config) { c.Timeout = d }
}

func WithRetries(n int) Option {
    return func(c *Config) { c.Retries = n }
}

func WithLogger(l Logger) Option {
    return func(c *Config) { c.Logger = l }
}

func NewClient(addr string, opts ...Option) *Client {
    cfg := &Config{Timeout: 30 * time.Second, Retries: 3}
    for _, opt := range opts {
        opt(cfg)
    }
    return &Client{addr: addr, config: cfg}
}

// 使用
client := NewClient("localhost:8080",
    WithTimeout(10*time.Second),
    WithRetries(5),
)
```

#### 6. 原型模式（Prototype）
**意图**：通过克隆现有对象来创建新对象

```go
type Cloneable interface {
    Clone() Cloneable
}

type Document struct {
    Title   string
    Content string
    Meta    map[string]string
}

func (d *Document) Clone() Cloneable {
    meta := make(map[string]string)
    for k, v := range d.Meta {
        meta[k] = v
    }
    return &Document{
        Title:   d.Title,
        Content: d.Content,
        Meta:    meta,
    }
}
```

---

### 结构型模式（7种）

#### 1. 适配器模式（Adapter）
**意图**：将一个类的接口转换成客户期望的另一个接口

```go
// 目标接口
type Logger interface {
    Log(level string, msg string, fields map[string]any)
}

// 第三方库（不兼容）
type ZapLogger struct{ *zap.Logger }

// 适配器
type ZapAdapter struct {
    zap *zap.Logger
}

func (a *ZapAdapter) Log(level string, msg string, fields map[string]any) {
    zapFields := make([]zap.Field, 0, len(fields))
    for k, v := range fields {
        zapFields = append(zapFields, zap.Any(k, v))
    }
    switch level {
    case "info":
        a.zap.Info(msg, zapFields...)
    case "error":
        a.zap.Error(msg, zapFields...)
    }
}
```

#### 2. 桥接模式（Bridge）
**意图**：将抽象与实现分离，使它们可以独立变化

```go
// 实现层接口
type MessageSender interface {
    Send(message string) error
}

type EmailSender struct{}
type SMSSender struct{}
type SlackSender struct{}

// 抽象层
type Notification struct {
    sender MessageSender
}

func (n *Notification) Notify(msg string) error {
    return n.sender.Send(msg)
}

// 扩展抽象
type UrgentNotification struct {
    Notification
}

func (u *UrgentNotification) Notify(msg string) error {
    return u.sender.Send("[URGENT] " + msg)
}
```

#### 3. 组合模式（Composite）
**意图**：将对象组合成树形结构以表示"部分-整体"层次

```go
type Component interface {
    Execute() string
    Add(Component)
    Remove(Component)
}

type Leaf struct {
    name string
}

func (l *Leaf) Execute() string { return l.name }
func (l *Leaf) Add(Component)    {}
func (l *Leaf) Remove(Component) {}

type Composite struct {
    name     string
    children []Component
}

func (c *Composite) Execute() string {
    results := []string{c.name}
    for _, child := range c.children {
        results = append(results, child.Execute())
    }
    return strings.Join(results, " -> ")
}

func (c *Composite) Add(child Component) {
    c.children = append(c.children, child)
}
```

#### 4. 装饰器模式（Decorator）
**意图**：动态地给对象添加额外职责

```go
type Handler interface {
    Handle(ctx context.Context, req Request) (Response, error)
}

// 日志装饰器
func WithLogging(h Handler, logger Logger) Handler {
    return HandlerFunc(func(ctx context.Context, req Request) (Response, error) {
        logger.Info("request started", "id", req.ID)
        start := time.Now()
        resp, err := h.Handle(ctx, req)
        logger.Info("request completed", "id", req.ID, "duration", time.Since(start))
        return resp, err
    })
}

// 重试装饰器
func WithRetry(h Handler, maxRetries int) Handler {
    return HandlerFunc(func(ctx context.Context, req Request) (Response, error) {
        var lastErr error
        for i := 0; i <= maxRetries; i++ {
            resp, err := h.Handle(ctx, req)
            if err == nil {
                return resp, nil
            }
            lastErr = err
            time.Sleep(time.Duration(i*100) * time.Millisecond)
        }
        return Response{}, lastErr
    })
}

// 组合
handler := WithLogging(WithRetry(baseHandler, 3), logger)
```

#### 5. 外观模式（Facade）
**意图**：为子系统中的一组接口提供统一的高层接口

```go
type OrderFacade struct {
    inventory *InventoryService
    payment   *PaymentService
    shipping  *ShippingService
    notify    *NotificationService
}

func (f *OrderFacade) PlaceOrder(ctx context.Context, order Order) error {
    // 检查库存
    if err := f.inventory.Reserve(ctx, order.Items); err != nil {
        return fmt.Errorf("inventory: %w", err)
    }

    // 处理支付
    if err := f.payment.Charge(ctx, order.Payment); err != nil {
        f.inventory.Release(ctx, order.Items)
        return fmt.Errorf("payment: %w", err)
    }

    // 安排配送
    if err := f.shipping.Schedule(ctx, order.Address); err != nil {
        f.payment.Refund(ctx, order.Payment)
        f.inventory.Release(ctx, order.Items)
        return fmt.Errorf("shipping: %w", err)
    }

    // 发送通知
    f.notify.SendConfirmation(ctx, order.Email)
    return nil
}
```

#### 6. 享元模式（Flyweight）
**意图**：运用共享技术有效地支持大量细粒度对象

```go
type Icon struct {
    name   string
    pixels []byte // 大量数据
}

type IconFactory struct {
    cache map[string]*Icon
    mu    sync.RWMutex
}

func (f *IconFactory) GetIcon(name string) *Icon {
    f.mu.RLock()
    if icon, ok := f.cache[name]; ok {
        f.mu.RUnlock()
        return icon
    }
    f.mu.RUnlock()

    f.mu.Lock()
    defer f.mu.Unlock()

    // 双重检查
    if icon, ok := f.cache[name]; ok {
        return icon
    }

    icon := loadIcon(name)
    f.cache[name] = icon
    return icon
}
```

#### 7. 代理模式（Proxy）
**意图**：为其他对象提供一种代理以控制对这个对象的访问

```go
// 缓存代理
type CachingProxy struct {
    service Service
    cache   *lru.Cache
    ttl     time.Duration
}

func (p *CachingProxy) Query(ctx context.Context, key string) (Result, error) {
    if cached, ok := p.cache.Get(key); ok {
        return cached.(Result), nil
    }

    result, err := p.service.Query(ctx, key)
    if err != nil {
        return Result{}, err
    }

    p.cache.Add(key, result)
    return result, nil
}

// 延迟加载代理
type LazyProxy struct {
    init    sync.Once
    service Service
    create  func() Service
}

func (p *LazyProxy) Query(ctx context.Context, key string) (Result, error) {
    p.init.Do(func() {
        p.service = p.create()
    })
    return p.service.Query(ctx, key)
}
```

---

### 行为型模式（10种）

#### 1. 责任链模式（Chain of Responsibility）
**意图**：使多个对象都有机会处理请求

```go
// HTTP 中间件链
type Middleware func(http.Handler) http.Handler

func Chain(h http.Handler, mws ...Middleware) http.Handler {
    for i := len(mws) - 1; i >= 0; i-- {
        h = mws[i](h)
    }
    return h
}

func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Printf("%s %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}

func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Header.Get("Authorization") == "" {
            http.Error(w, "unauthorized", 401)
            return
        }
        next.ServeHTTP(w, r)
    })
}

// 使用
handler := Chain(finalHandler, LoggingMiddleware, AuthMiddleware, RateLimitMiddleware)
```

#### 2. 命令模式（Command）
**意图**：将请求封装为对象，从而可以参数化、排队或记录请求

```go
type Command interface {
    Execute() error
    Undo() error
}

type CreateFileCommand struct {
    path    string
    content []byte
}

func (c *CreateFileCommand) Execute() error {
    return os.WriteFile(c.path, c.content, 0644)
}

func (c *CreateFileCommand) Undo() error {
    return os.Remove(c.path)
}

// 命令管理器（支持撤销）
type CommandManager struct {
    history []Command
}

func (m *CommandManager) Execute(cmd Command) error {
    if err := cmd.Execute(); err != nil {
        return err
    }
    m.history = append(m.history, cmd)
    return nil
}

func (m *CommandManager) Undo() error {
    if len(m.history) == 0 {
        return errors.New("nothing to undo")
    }
    cmd := m.history[len(m.history)-1]
    m.history = m.history[:len(m.history)-1]
    return cmd.Undo()
}
```

#### 3. 迭代器模式（Iterator）
**意图**：提供一种方法顺序访问聚合对象中的各个元素

```go
// Go 1.23+ 使用 iter 包
func (t *Tree[T]) All() iter.Seq[T] {
    return func(yield func(T) bool) {
        t.iterate(t.root, yield)
    }
}

func (t *Tree[T]) iterate(node *Node[T], yield func(T) bool) bool {
    if node == nil {
        return true
    }
    return t.iterate(node.left, yield) &&
           yield(node.value) &&
           t.iterate(node.right, yield)
}

// 使用
for v := range tree.All() {
    fmt.Println(v)
}
```

#### 4. 中介者模式（Mediator）
**意图**：用一个中介对象来封装一系列对象交互

```go
type EventBus struct {
    handlers map[string][]func(Event)
    mu       sync.RWMutex
}

func (eb *EventBus) Subscribe(eventType string, handler func(Event)) {
    eb.mu.Lock()
    defer eb.mu.Unlock()
    eb.handlers[eventType] = append(eb.handlers[eventType], handler)
}

func (eb *EventBus) Publish(event Event) {
    eb.mu.RLock()
    handlers := eb.handlers[event.Type]
    eb.mu.RUnlock()

    for _, h := range handlers {
        go h(event)
    }
}
```

#### 5. 备忘录模式（Memento）
**意图**：在不破坏封装的前提下，捕获并外部化对象的内部状态

```go
type EditorMemento struct {
    content  string
    cursor   int
    savedAt  time.Time
}

type Editor struct {
    content string
    cursor  int
}

func (e *Editor) Save() *EditorMemento {
    return &EditorMemento{
        content: e.content,
        cursor:  e.cursor,
        savedAt: time.Now(),
    }
}

func (e *Editor) Restore(m *EditorMemento) {
    e.content = m.content
    e.cursor = m.cursor
}

// 历史管理
type History struct {
    mementos []*EditorMemento
}

func (h *History) Push(m *EditorMemento) {
    h.mementos = append(h.mementos, m)
}

func (h *History) Pop() *EditorMemento {
    if len(h.mementos) == 0 {
        return nil
    }
    m := h.mementos[len(h.mementos)-1]
    h.mementos = h.mementos[:len(h.mementos)-1]
    return m
}
```

#### 6. 观察者模式（Observer）
**意图**：定义对象间的一对多依赖，状态变化时通知所有依赖者

```go
type Observer interface {
    OnNotify(event Event)
}

type Subject struct {
    observers []Observer
    mu        sync.RWMutex
}

func (s *Subject) Attach(o Observer) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.observers = append(s.observers, o)
}

func (s *Subject) Notify(event Event) {
    s.mu.RLock()
    observers := make([]Observer, len(s.observers))
    copy(observers, s.observers)
    s.mu.RUnlock()

    for _, o := range observers {
        o.OnNotify(event)
    }
}

// Go 风格：使用 channel
type Subscription struct {
    Events <-chan Event
    cancel func()
}

func (s *Subject) SubscribeChan(bufSize int) *Subscription {
    ch := make(chan Event, bufSize)
    done := make(chan struct{})
    // ... 注册
    return &Subscription{Events: ch, cancel: func() { close(done) }}
}
```

#### 7. 状态模式（State）
**意图**：允许对象在内部状态改变时改变行为

```go
type OrderState interface {
    Process(o *Order) error
    Cancel(o *Order) error
    Ship(o *Order) error
    String() string
}

type Order struct {
    ID    string
    state OrderState
}

func (o *Order) SetState(s OrderState) { o.state = s }
func (o *Order) Process() error        { return o.state.Process(o) }
func (o *Order) Cancel() error         { return o.state.Cancel(o) }
func (o *Order) Ship() error           { return o.state.Ship(o) }

type PendingState struct{}

func (s *PendingState) Process(o *Order) error {
    o.SetState(&ProcessingState{})
    return nil
}

func (s *PendingState) Cancel(o *Order) error {
    o.SetState(&CancelledState{})
    return nil
}

func (s *PendingState) Ship(o *Order) error {
    return errors.New("cannot ship pending order")
}

func (s *PendingState) String() string { return "pending" }
```

#### 8. 策略模式（Strategy）
**意图**：定义一系列算法，把它们封装起来，并使它们可互换

```go
type Compressor interface {
    Compress(data []byte) ([]byte, error)
    Decompress(data []byte) ([]byte, error)
}

type GzipCompressor struct{}
type ZstdCompressor struct{}
type LZ4Compressor struct{}

type FileProcessor struct {
    compressor Compressor
}

func (p *FileProcessor) SetCompressor(c Compressor) {
    p.compressor = c
}

func (p *FileProcessor) ProcessFile(path string) error {
    data, err := os.ReadFile(path)
    if err != nil {
        return err
    }

    compressed, err := p.compressor.Compress(data)
    if err != nil {
        return err
    }

    return os.WriteFile(path+".compressed", compressed, 0644)
}
```

#### 9. 模板方法（Template Method）
**意图**：定义算法骨架，将某些步骤延迟到子类

```go
type DataExporter interface {
    FetchData(ctx context.Context) ([]Record, error)
    Transform(records []Record) []byte
    GetExtension() string
}

// 模板方法
func Export(ctx context.Context, exporter DataExporter, path string) error {
    records, err := exporter.FetchData(ctx)
    if err != nil {
        return err
    }

    data := exporter.Transform(records)
    filename := path + exporter.GetExtension()

    return os.WriteFile(filename, data, 0644)
}

type CSVExporter struct{ db *sql.DB }
type JSONExporter struct{ db *sql.DB }
type ExcelExporter struct{ db *sql.DB }

func (e *CSVExporter) GetExtension() string { return ".csv" }
func (e *JSONExporter) GetExtension() string { return ".json" }
```

#### 10. 访问者模式（Visitor）
**意图**：在不改变元素类的前提下定义作用于元素的新操作

```go
type Visitor interface {
    VisitFile(f *File)
    VisitDirectory(d *Directory)
}

type Element interface {
    Accept(v Visitor)
}

type File struct {
    Name string
    Size int64
}

func (f *File) Accept(v Visitor) { v.VisitFile(f) }

type Directory struct {
    Name     string
    Children []Element
}

func (d *Directory) Accept(v Visitor) {
    v.VisitDirectory(d)
    for _, child := range d.Children {
        child.Accept(v)
    }
}

// 具体访问者：计算总大小
type SizeCalculator struct {
    TotalSize int64
}

func (c *SizeCalculator) VisitFile(f *File) {
    c.TotalSize += f.Size
}

func (c *SizeCalculator) VisitDirectory(d *Directory) {}
```

---

## 第二部分：云架构模式（分布式系统）

### 可靠性模式

#### 断路器（Circuit Breaker）
**意图**：防止应用程序反复执行可能失败的操作

```go
import "github.com/sony/gobreaker/v2"

cb := gobreaker.NewCircuitBreaker[*http.Response](gobreaker.Settings{
    Name:        "api-breaker",
    MaxRequests: 3,                // 半开状态最大请求数
    Interval:    10 * time.Second, // 统计周期
    Timeout:     30 * time.Second, // 开启后多久尝试半开
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        return counts.ConsecutiveFailures > 5
    },
})

resp, err := cb.Execute(func() (*http.Response, error) {
    return http.Get("https://api.example.com")
})
```

#### 重试模式（Retry）
**意图**：通过透明重试处理临时故障

```go
import "github.com/avast/retry-go/v5"

err := retry.Do(
    func() error {
        return callExternalAPI()
    },
    retry.Attempts(5),
    retry.Delay(100*time.Millisecond),
    retry.MaxDelay(5*time.Second),
    retry.DelayType(retry.BackOffDelay),
    retry.RetryIf(func(err error) bool {
        return errors.Is(err, ErrTemporary)
    }),
)
```

#### 隔离模式（Bulkhead）
**意图**：隔离应用程序元素，使一个元素故障不影响其他元素

```go
type Bulkhead struct {
    sem chan struct{}
}

func NewBulkhead(maxConcurrent int) *Bulkhead {
    return &Bulkhead{sem: make(chan struct{}, maxConcurrent)}
}

func (b *Bulkhead) Execute(ctx context.Context, fn func() error) error {
    select {
    case b.sem <- struct{}{}:
        defer func() { <-b.sem }()
        return fn()
    case <-ctx.Done():
        return ctx.Err()
    }
}

// 为不同服务创建独立隔离舱
var (
    paymentBulkhead  = NewBulkhead(10)
    inventoryBulkhead = NewBulkhead(20)
)
```

#### 限流模式（Rate Limiting / Throttling）
**意图**：控制资源消耗速率

```go
import "golang.org/x/time/rate"

// 令牌桶限流器
limiter := rate.NewLimiter(rate.Limit(100), 10) // 100 QPS, burst 10

func handleRequest(ctx context.Context) error {
    if err := limiter.Wait(ctx); err != nil {
        return err
    }
    return processRequest()
}

// 或使用 Allow() 非阻塞检查
if !limiter.Allow() {
    return ErrRateLimited
}
```

### 数据一致性模式

#### Saga 模式
**意图**：在分布式事务中管理数据一致性

```go
type SagaStep struct {
    Name       string
    Execute    func(ctx context.Context) error
    Compensate func(ctx context.Context) error
}

type Saga struct {
    steps []SagaStep
}

func (s *Saga) Run(ctx context.Context) error {
    executed := []SagaStep{}

    for _, step := range s.steps {
        if err := step.Execute(ctx); err != nil {
            // 执行补偿
            for i := len(executed) - 1; i >= 0; i-- {
                if compErr := executed[i].Compensate(ctx); compErr != nil {
                    log.Printf("compensation failed for %s: %v", executed[i].Name, compErr)
                }
            }
            return fmt.Errorf("saga failed at %s: %w", step.Name, err)
        }
        executed = append(executed, step)
    }
    return nil
}

// 使用
orderSaga := &Saga{
    steps: []SagaStep{
        {Name: "reserve_inventory", Execute: reserveInventory, Compensate: releaseInventory},
        {Name: "charge_payment", Execute: chargePayment, Compensate: refundPayment},
        {Name: "create_shipment", Execute: createShipment, Compensate: cancelShipment},
    },
}
```

#### 事件溯源（Event Sourcing）
**意图**：使用追加式事件存储记录完整的状态变化历史

```go
type Event struct {
    ID        string
    Type      string
    AggregateID string
    Data      json.RawMessage
    Timestamp time.Time
    Version   int
}

type EventStore interface {
    Append(ctx context.Context, events ...Event) error
    Load(ctx context.Context, aggregateID string) ([]Event, error)
}

type Account struct {
    ID      string
    Balance int
    Version int
}

func (a *Account) Apply(event Event) {
    switch event.Type {
    case "AccountCreated":
        var data struct{ InitialBalance int }
        json.Unmarshal(event.Data, &data)
        a.Balance = data.InitialBalance
    case "MoneyDeposited":
        var data struct{ Amount int }
        json.Unmarshal(event.Data, &data)
        a.Balance += data.Amount
    case "MoneyWithdrawn":
        var data struct{ Amount int }
        json.Unmarshal(event.Data, &data)
        a.Balance -= data.Amount
    }
    a.Version = event.Version
}

func LoadAccount(ctx context.Context, store EventStore, id string) (*Account, error) {
    events, err := store.Load(ctx, id)
    if err != nil {
        return nil, err
    }

    account := &Account{ID: id}
    for _, e := range events {
        account.Apply(e)
    }
    return account, nil
}
```

#### CQRS（命令查询职责分离）
**意图**：分离读写操作以独立优化

```go
// 命令端（写）
type OrderCommandService struct {
    repo  OrderRepository
    bus   EventBus
}

func (s *OrderCommandService) CreateOrder(ctx context.Context, cmd CreateOrderCommand) error {
    order := NewOrder(cmd)
    if err := s.repo.Save(ctx, order); err != nil {
        return err
    }
    return s.bus.Publish(OrderCreatedEvent{Order: order})
}

// 查询端（读）
type OrderQueryService struct {
    readDB *sql.DB // 优化的读库
}

func (s *OrderQueryService) GetOrderSummary(ctx context.Context, id string) (*OrderSummaryDTO, error) {
    // 从专门优化的读模型查询
    return s.readDB.Query(ctx, "SELECT ... FROM order_summary_view WHERE id = ?", id)
}
```

### 消息通信模式

#### 发布/订阅（Pub/Sub）
**意图**：异步广播事件给多个消费者

```go
type Publisher interface {
    Publish(ctx context.Context, topic string, msg Message) error
}

type Subscriber interface {
    Subscribe(ctx context.Context, topic string, handler func(Message)) error
}

// 使用 Kafka
func (p *KafkaPublisher) Publish(ctx context.Context, topic string, msg Message) error {
    return p.producer.Produce(&kafka.Message{
        TopicPartition: kafka.TopicPartition{Topic: &topic},
        Key:            []byte(msg.Key),
        Value:          msg.Data,
    }, nil)
}
```

#### 竞争消费者（Competing Consumers）
**意图**：多个消费者并行处理消息队列

```go
func startConsumers(ctx context.Context, queue Queue, workers int, handler func(Message) error) {
    for i := 0; i < workers; i++ {
        go func(workerID int) {
            for {
                select {
                case <-ctx.Done():
                    return
                default:
                    msg, err := queue.Receive(ctx)
                    if err != nil {
                        continue
                    }
                    if err := handler(msg); err != nil {
                        queue.Nack(ctx, msg)
                    } else {
                        queue.Ack(ctx, msg)
                    }
                }
            }
        }(i)
    }
}
```

### API 网关模式

#### 网关聚合（Gateway Aggregation）
**意图**：将多个微服务请求聚合为单个请求

```go
type ProductPageData struct {
    Product  *Product
    Reviews  []*Review
    Related  []*Product
    Inventory *InventoryStatus
}

func (g *Gateway) GetProductPage(ctx context.Context, productID string) (*ProductPageData, error) {
    var (
        result ProductPageData
        errs   []error
        mu     sync.Mutex
    )

    var wg sync.WaitGroup
    wg.Add(4)

    go func() {
        defer wg.Done()
        product, err := g.productService.Get(ctx, productID)
        mu.Lock()
        result.Product = product
        if err != nil { errs = append(errs, err) }
        mu.Unlock()
    }()

    go func() {
        defer wg.Done()
        reviews, err := g.reviewService.List(ctx, productID)
        mu.Lock()
        result.Reviews = reviews
        if err != nil { errs = append(errs, err) }
        mu.Unlock()
    }()

    // ... 其他并行请求

    wg.Wait()

    if len(errs) > 0 {
        return nil, errors.Join(errs...)
    }
    return &result, nil
}
```

#### BFF（Backend for Frontend）
**意图**：为不同前端创建专用后端

```go
// Mobile BFF - 返回精简数据
type MobileBFF struct {
    services *Services
}

func (b *MobileBFF) GetUserDashboard(ctx context.Context, userID string) (*MobileDashboard, error) {
    // 返回移动端优化的数据结构
    return &MobileDashboard{
        Summary:     b.services.GetSummary(ctx, userID),
        RecentItems: b.services.GetRecent(ctx, userID, 5), // 只取5条
    }
}

// Web BFF - 返回完整数据
type WebBFF struct {
    services *Services
}

func (b *WebBFF) GetUserDashboard(ctx context.Context, userID string) (*WebDashboard, error) {
    // 返回 Web 端完整数据
    return &WebDashboard{
        Summary:      b.services.GetSummary(ctx, userID),
        RecentItems:  b.services.GetRecent(ctx, userID, 50),
        Analytics:    b.services.GetAnalytics(ctx, userID),
        Recommendations: b.services.GetRecommendations(ctx, userID),
    }
}
```

#### Strangler Fig（绞杀者模式）
**意图**：渐进式迁移遗留系统，逐步用新服务替换旧功能

```go
// 路由层决定走新系统还是旧系统
type StranglerRouter struct {
    legacyService LegacyService
    newService    NewService
    featureFlags  FeatureFlags
}

func (r *StranglerRouter) GetUser(ctx context.Context, id string) (*User, error) {
    // 通过特性开关决定路由
    if r.featureFlags.IsEnabled(ctx, "new-user-service") {
        return r.newService.GetUser(ctx, id)
    }
    return r.legacyService.GetUser(ctx, id)
}

// 渐进式迁移：按用户百分比
func (r *StranglerRouter) GetOrder(ctx context.Context, userID, orderID string) (*Order, error) {
    // 10% 用户走新系统
    if r.featureFlags.IsEnabledForUser(ctx, "new-order-service", userID, 10) {
        return r.newService.GetOrder(ctx, orderID)
    }
    return r.legacyService.GetOrder(ctx, orderID)
}

// 对比模式：同时调用新旧系统，比较结果
func (r *StranglerRouter) GetUserWithShadow(ctx context.Context, id string) (*User, error) {
    // 始终返回旧系统结果
    result, err := r.legacyService.GetUser(ctx, id)

    // 异步调用新系统并比较
    go func() {
        newResult, newErr := r.newService.GetUser(ctx, id)
        r.compare("GetUser", result, newResult, err, newErr)
    }()

    return result, err
}
```

**迁移步骤**：识别边界 → 抽象接口 → 实现新服务 → 影子对比 → 逐步切流 → 下线旧系统

#### Sidecar（边车模式）
**意图**：将辅助功能作为独立进程部署在主应用旁边

```go
// Sidecar 代理：为主应用添加可观测性
type SidecarProxy struct {
    target  string
    metrics *prometheus.Registry
    tracer  trace.Tracer
}

func (p *SidecarProxy) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    start := time.Now()
    ctx, span := p.tracer.Start(r.Context(), r.URL.Path)
    defer span.End()

    // 代理到主应用
    proxyReq, _ := http.NewRequestWithContext(ctx, r.Method, p.target+r.URL.Path, r.Body)
    resp, err := http.DefaultClient.Do(proxyReq)
    if err != nil {
        span.RecordError(err)
        http.Error(w, err.Error(), http.StatusBadGateway)
        return
    }
    defer resp.Body.Close()

    p.recordMetrics(r.URL.Path, resp.StatusCode, time.Since(start))
    copyResponse(w, resp)
}
```

**典型 Sidecar**：日志收集(Fluentd)、服务发现(Consul)、流量管理(Envoy)

#### Ambassador（大使模式）
**意图**：代理本地与远程服务通信，处理重试、熔断、认证

```go
type Ambassador struct {
    client    *http.Client
    breaker   *gobreaker.CircuitBreaker[*http.Response]
    rateLimit *rate.Limiter
    auth      AuthProvider
}

func (a *Ambassador) Do(ctx context.Context, req *http.Request) (*http.Response, error) {
    // 限流
    if err := a.rateLimit.Wait(ctx); err != nil {
        return nil, err
    }

    // 认证
    token, _ := a.auth.GetToken(ctx)
    req.Header.Set("Authorization", "Bearer "+token)

    // 熔断 + 代理
    return a.breaker.Execute(func() (*http.Response, error) {
        return a.client.Do(req.WithContext(ctx))
    })
}
```

#### API Gateway（API 网关）
**意图**：统一入口处理认证、路由、限流、协议转换

```go
type APIGateway struct {
    routes   map[string]RouteConfig
    auth     AuthMiddleware
    limiter  RateLimiter
    breakers map[string]*gobreaker.CircuitBreaker[*http.Response]
}

func (g *APIGateway) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    route, found := g.matchRoute(r.URL.Path)
    if !found {
        http.NotFound(w, r)
        return
    }

    // 认证 → 限流 → 熔断代理
    if route.RequireAuth {
        if err := g.auth.Authenticate(r); err != nil {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
    }

    if !g.limiter.Allow(r.RemoteAddr, route.RateLimit) {
        http.Error(w, "Rate limited", http.StatusTooManyRequests)
        return
    }

    resp, err := g.breakers[route.Upstream].Execute(func() (*http.Response, error) {
        return g.proxyRequest(r, route)
    })
    if err != nil {
        http.Error(w, "Service unavailable", http.StatusServiceUnavailable)
        return
    }
    g.writeResponse(w, resp)
}
```

**网关职责**：认证授权 → 路由 → 限流 → 熔断 → 协议转换 → 日志追踪

---

## 第三部分：Go 并发模式

### Pipeline 模式
```go
func generator[T any](items ...T) <-chan T {
    out := make(chan T)
    go func() {
        defer close(out)
        for _, item := range items {
            out <- item
        }
    }()
    return out
}

func transform[T, R any](in <-chan T, fn func(T) R) <-chan R {
    out := make(chan R)
    go func() {
        defer close(out)
        for v := range in {
            out <- fn(v)
        }
    }()
    return out
}

func filter[T any](in <-chan T, predicate func(T) bool) <-chan T {
    out := make(chan T)
    go func() {
        defer close(out)
        for v := range in {
            if predicate(v) {
                out <- v
            }
        }
    }()
    return out
}

// 使用
result := filter(
    transform(generator(1, 2, 3, 4, 5), func(n int) int { return n * n }),
    func(n int) bool { return n > 10 },
)
```

### Fan-Out/Fan-In 模式
```go
func fanOut[T any](in <-chan T, workers int, process func(T) T) []<-chan T {
    outs := make([]<-chan T, workers)
    for i := 0; i < workers; i++ {
        outs[i] = func() <-chan T {
            out := make(chan T)
            go func() {
                defer close(out)
                for v := range in {
                    out <- process(v)
                }
            }()
            return out
        }()
    }
    return outs
}

func fanIn[T any](channels ...<-chan T) <-chan T {
    var wg sync.WaitGroup
    out := make(chan T)

    wg.Add(len(channels))
    for _, ch := range channels {
        go func(c <-chan T) {
            defer wg.Done()
            for v := range c {
                out <- v
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

### Worker Pool 模式
```go
type WorkerPool[T any, R any] struct {
    workers int
    jobs    chan T
    results chan R
    process func(T) R
}

func NewWorkerPool[T any, R any](workers int, process func(T) R) *WorkerPool[T, R] {
    return &WorkerPool[T, R]{
        workers: workers,
        jobs:    make(chan T, workers*2),
        results: make(chan R, workers*2),
        process: process,
    }
}

func (p *WorkerPool[T, R]) Start(ctx context.Context) {
    for i := 0; i < p.workers; i++ {
        go func() {
            for {
                select {
                case <-ctx.Done():
                    return
                case job, ok := <-p.jobs:
                    if !ok {
                        return
                    }
                    p.results <- p.process(job)
                }
            }
        }()
    }
}

func (p *WorkerPool[T, R]) Submit(job T) { p.jobs <- job }
func (p *WorkerPool[T, R]) Results() <-chan R { return p.results }
```

### errgroup 并发控制
```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) ([]Response, error) {
    results := make([]Response, len(urls))
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(10) // 最多 10 个并发

    for i, url := range urls {
        i, url := i, url
        g.Go(func() error {
            resp, err := fetch(ctx, url)
            if err != nil {
                return err
            }
            results[i] = resp
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

---

## 第四部分：架构原则

### SOLID 原则

| 原则 | 说明 | Go 实践 |
|------|------|---------|
| **S**ingle Responsibility | 一个类只做一件事 | 小接口、小包 |
| **O**pen/Closed | 对扩展开放，对修改关闭 | 接口 + 组合 |
| **L**iskov Substitution | 子类可替换父类 | 接口契约 |
| **I**nterface Segregation | 接口最小化 | 小接口（1-3 方法）|
| **D**ependency Inversion | 依赖抽象而非具体 | 依赖注入 |

### Clean Architecture 分层

```
┌────────────────────────────────────────┐
│           Frameworks & Drivers         │  ← HTTP, gRPC, DB, External
├────────────────────────────────────────┤
│          Interface Adapters            │  ← Controllers, Presenters, Gateways
├────────────────────────────────────────┤
│           Application/Use Cases        │  ← Business Logic Orchestration
├────────────────────────────────────────┤
│              Domain/Entities           │  ← Core Business Rules
└────────────────────────────────────────┘

依赖方向：外层依赖内层，内层不知道外层
```

```go
// Domain 层
type User struct {
    ID    string
    Email string
}

type UserRepository interface {
    FindByID(ctx context.Context, id string) (*User, error)
    Save(ctx context.Context, user *User) error
}

// Use Case 层
type CreateUserUseCase struct {
    repo UserRepository
}

func (uc *CreateUserUseCase) Execute(ctx context.Context, input CreateUserInput) (*User, error) {
    // 业务逻辑
}

// Interface Adapter 层
type UserHandler struct {
    useCase *CreateUserUseCase
}

func (h *UserHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // HTTP 适配
}

// Infrastructure 层
type PostgresUserRepository struct {
    db *sql.DB
}
```

---

## 模式选择指南

| 问题场景 | 推荐模式 |
|----------|----------|
| 对象创建复杂 | Builder / Functional Options |
| 根据条件创建不同对象 | Factory Method / Abstract Factory |
| 全局唯一实例 | Singleton (sync.Once) |
| 为已有类型添加功能 | Decorator / Adapter |
| 算法可互换 | Strategy |
| 对象状态决定行为 | State |
| 请求处理链 | Chain of Responsibility / Middleware |
| 事件通知 | Observer / Pub-Sub |
| 分布式事务 | Saga / 补偿事务 |
| 防止级联故障 | Circuit Breaker / Bulkhead |
| 处理临时故障 | Retry with Backoff |
| 读写分离优化 | CQRS |
| 审计追溯 | Event Sourcing |
| 并发任务处理 | Worker Pool / errgroup |
| 数据流处理 | Pipeline / Fan-Out/Fan-In |
| 限制并发数 | Semaphore / Bulkhead |
| 微服务网关 | Gateway Aggregation / BFF |

---

## 第五部分：架构决策记录（ADR）

### 为什么需要 ADR？

- 防止"短期记忆"和决策循环
- 记录架构决策的上下文和理由
- 便于团队成员理解历史决策

### 何时创建 ADR？

- 新功能的架构设计
- 依赖变更（添加/移除重要依赖）
- 重大重构
- 技术栈选择

### ADR 存放位置

```
project/
└── design/
    ├── ADR-001-database-selection.md
    ├── ADR-002-authentication-strategy.md
    └── ADR-003-caching-layer.md
```

### ADR 模板

```markdown
# ADR-00X: 标题

**状态**: Draft | Proposed | Accepted | Deprecated | Superseded by ADR-XXX
**日期**: YYYY-MM-DD
**决策者**: [参与人员/团队]

## 背景

是什么问题促使我们做出这个决策？有什么约束条件？

## 考虑的选项

### 选项 1: [名称]
- 描述
- **优点**: ...
- **缺点**: ...

### 选项 2: [名称]
- 描述
- **优点**: ...
- **缺点**: ...

## 决策

我们选择**选项 X**，因为...

## 后果

### 正面
- ...

### 负面
- ...

### 风险
- ...
```

### ADR 示例

```markdown
# ADR-001: 使用 PostgreSQL 作为主数据库

**状态**: Accepted
**日期**: 2024-01-15
**决策者**: 后端团队

## 背景

我们需要为新项目选择主数据库。要求：
- 支持复杂查询和事务
- 良好的 Go 生态支持
- 可扩展性
- 团队熟悉度

## 考虑的选项

### 选项 1: PostgreSQL
- 成熟的关系型数据库
- **优点**: 强大的 SQL 支持、pgx 驱动性能优秀、团队熟悉
- **缺点**: 运维复杂度较 MySQL 略高

### 选项 2: MySQL
- 流行的关系型数据库
- **优点**: 广泛使用、文档丰富
- **缺点**: JSON 支持不如 PG、部分高级特性缺失

### 选项 3: MongoDB
- 文档型数据库
- **优点**: 灵活的 schema、水平扩展
- **缺点**: 事务支持较弱、不适合复杂关联查询

## 决策

选择 **PostgreSQL**，因为：
1. 团队有丰富经验
2. pgx 驱动性能优秀
3. 支持 JSONB 满足半结构化需求
4. 强事务保证

## 后果

### 正面
- 可利用团队现有经验
- 良好的查询性能

### 负面
- 需要配置主从复制实现高可用

### 风险
- 单点故障（通过主从复制缓解）
```

### ADR 生命周期

1. **Draft** — 初稿，征求意见
2. **Proposed** — 正式提议，待审核
3. **Accepted** — 已接受，开始实施
4. **Deprecated** — 已弃用
5. **Superseded** — 被新 ADR 取代

### 工作流：设计新功能

1. **分析需求**：理解目标和约束
2. **起草 ADR**：在 `design/` 创建文档
3. **评审**：团队讨论选项
4. **修订**：根据反馈更新
5. **决策**：标记为 Accepted
6. **实施**：开始编码

---

## 参考资源

- [Refactoring Guru 设计模式](https://refactoringguru.cn/design-patterns)
- [Microsoft 云架构模式](https://learn.microsoft.com/zh-cn/azure/architecture/patterns/)
- [Go Patterns](https://github.com/tmrts/go-patterns)
- [ADR GitHub Organization](https://adr.github.io/)
