---
name: golang-patterns
description: Go 惯用模式专家 - 简洁设计、零值可用、接口设计、错误处理、并发模式、内存优化。使用场景：编写新代码、代码审查、重构优化。
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

# Go 惯用模式专家

应用 Go 惯用模式编写或审查代码：$ARGUMENTS

---

## 核心原则

### 1. 简洁胜于聪明

Go 代码应该一目了然，不需要思考两次。

```go
// ✅ 清晰直接
func GetUser(id string) (*User, error) {
    user, err := db.FindUser(id)
    if err != nil {
        return nil, fmt.Errorf("get user %s: %w", id, err)
    }
    return user, nil
}

// ❌ 过度聪明
func GetUser(id string) (*User, error) {
    return func() (*User, error) {
        if u, e := db.FindUser(id); e == nil {
            return u, nil
        } else {
            return nil, e
        }
    }()
}
```

### 2. 零值可用

设计类型使其零值立即可用，无需初始化。

```go
// ✅ 零值可用
type Counter struct {
    mu    sync.Mutex
    count int // 零值是 0，直接可用
}

func (c *Counter) Inc() {
    c.mu.Lock()
    c.count++
    c.mu.Unlock()
}

// ✅ bytes.Buffer 零值可用
var buf bytes.Buffer
buf.WriteString("hello")

// ❌ 需要初始化
type BadCounter struct {
    counts map[string]int // nil map 会 panic
}
```

### 3. 接收接口，返回具体类型

```go
// ✅ 接收接口参数，返回具体类型
func ProcessData(r io.Reader) (*Result, error) {
    data, err := io.ReadAll(r)
    if err != nil {
        return nil, err
    }
    return &Result{Data: data}, nil
}

// ❌ 返回接口（不必要地隐藏实现）
func ProcessData(r io.Reader) (io.Reader, error) { ... }
```

---

## 错误处理模式

### 使用 %w 包装错误

```go
func LoadConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("load config %s: %w", path, err)
    }

    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return nil, fmt.Errorf("parse config %s: %w", path, err)
    }

    return &cfg, nil
}
```

### 自定义错误类型

```go
// 领域特定错误
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Message)
}

// 哨兵错误
var (
    ErrNotFound     = errors.New("resource not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrInvalidInput = errors.New("invalid input")
)
```

### errors.Is 和 errors.As

```go
func HandleError(err error) {
    // 检查特定错误
    if errors.Is(err, sql.ErrNoRows) {
        log.Println("No records found")
        return
    }

    // 检查错误类型
    var validationErr *ValidationError
    if errors.As(err, &validationErr) {
        log.Printf("Validation error on field %s: %s",
            validationErr.Field, validationErr.Message)
        return
    }

    log.Printf("Unexpected error: %v", err)
}
```

### 永不忽略错误

```go
// ❌ 用 _ 忽略错误
result, _ := doSomething()

// ✅ 处理或明确说明为何可以忽略
result, err := doSomething()
if err != nil {
    return err
}

// 可接受：错误真的不重要时（罕见）
_ = writer.Close() // 尽力清理，错误在别处记录
```

---

## 并发模式

### Worker Pool

```go
func WorkerPool(jobs <-chan Job, results chan<- Result, numWorkers int) {
    var wg sync.WaitGroup

    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }()
    }

    wg.Wait()
    close(results)
}
```

### Context 取消和超时

```go
func FetchWithTimeout(ctx context.Context, url string) ([]byte, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, fmt.Errorf("create request: %w", err)
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("fetch %s: %w", url, err)
    }
    defer resp.Body.Close()

    return io.ReadAll(resp.Body)
}
```

### 优雅关闭

```go
func GracefulShutdown(server *http.Server) {
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

    <-quit
    log.Println("Shutting down server...")

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Fatalf("Server forced to shutdown: %v", err)
    }

    log.Println("Server exited")
}
```

### errgroup 协调并发

```go
import "golang.org/x/sync/errgroup"

func FetchAll(ctx context.Context, urls []string) ([][]byte, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([][]byte, len(urls))

    for i, url := range urls {
        i, url := i, url // 捕获循环变量
        g.Go(func() error {
            data, err := FetchWithTimeout(ctx, url)
            if err != nil {
                return err
            }
            results[i] = data
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

### 避免 Goroutine 泄漏

```go
// ❌ Goroutine 泄漏
func leakyFetch(ctx context.Context, url string) <-chan []byte {
    ch := make(chan []byte)
    go func() {
        data, _ := fetch(url)
        ch <- data // 无接收者时永远阻塞
    }()
    return ch
}

// ✅ 正确处理取消
func safeFetch(ctx context.Context, url string) <-chan []byte {
    ch := make(chan []byte, 1) // 缓冲 channel
    go func() {
        data, err := fetch(url)
        if err != nil {
            return
        }
        select {
        case ch <- data:
        case <-ctx.Done():
        }
    }()
    return ch
}
```

---

## 接口设计

### 小而专注的接口

```go
// ✅ 单方法接口
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

// 按需组合
type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

### 在使用方定义接口

```go
// 在消费方包中定义，而非提供方
package service

// UserStore 定义本服务的需求
type UserStore interface {
    GetUser(id string) (*User, error)
    SaveUser(user *User) error
}

type Service struct {
    store UserStore
}

// 具体实现可以在另一个包
// 它不需要知道这个接口
```

### 可选行为的类型断言

```go
type Flusher interface {
    Flush() error
}

func WriteAndFlush(w io.Writer, data []byte) error {
    if _, err := w.Write(data); err != nil {
        return err
    }

    // 支持则 Flush
    if f, ok := w.(Flusher); ok {
        return f.Flush()
    }
    return nil
}
```

---

## 结构体设计

### 函数选项模式

```go
type Server struct {
    addr    string
    timeout time.Duration
    logger  *log.Logger
}

type Option func(*Server)

func WithTimeout(d time.Duration) Option {
    return func(s *Server) {
        s.timeout = d
    }
}

func WithLogger(l *log.Logger) Option {
    return func(s *Server) {
        s.logger = l
    }
}

func NewServer(addr string, opts ...Option) *Server {
    s := &Server{
        addr:    addr,
        timeout: 30 * time.Second, // 默认值
        logger:  log.Default(),    // 默认值
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// 使用
server := NewServer(":8080",
    WithTimeout(60*time.Second),
    WithLogger(customLogger),
)
```

### 嵌入实现组合

```go
type Logger struct {
    prefix string
}

func (l *Logger) Log(msg string) {
    fmt.Printf("[%s] %s\n", l.prefix, msg)
}

type Server struct {
    *Logger // 嵌入 - Server 获得 Log 方法
    addr    string
}

func NewServer(addr string) *Server {
    return &Server{
        Logger: &Logger{prefix: "SERVER"},
        addr:   addr,
    }
}

// 使用
s := NewServer(":8080")
s.Log("Starting...") // 调用嵌入的 Logger.Log
```

---

## 内存和性能

### 已知大小时预分配切片

```go
// ❌ 多次扩容
func processItems(items []Item) []Result {
    var results []Result
    for _, item := range items {
        results = append(results, process(item))
    }
    return results
}

// ✅ 单次分配
func processItems(items []Item) []Result {
    results := make([]Result, 0, len(items))
    for _, item := range items {
        results = append(results, process(item))
    }
    return results
}
```

### sync.Pool 用于频繁分配

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func ProcessRequest(data []byte) []byte {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufferPool.Put(buf)
    }()

    buf.Write(data)
    // 处理...
    return buf.Bytes()
}
```

### 避免循环中的字符串拼接

```go
// ❌ 多次分配
func join(parts []string) string {
    var result string
    for _, p := range parts {
        result += p + ","
    }
    return result
}

// ✅ strings.Builder
func join(parts []string) string {
    var sb strings.Builder
    for i, p := range parts {
        if i > 0 {
            sb.WriteString(",")
        }
        sb.WriteString(p)
    }
    return sb.String()
}

// 最佳：使用标准库
func join(parts []string) string {
    return strings.Join(parts, ",")
}
```

---

## 项目布局

```
myproject/
├── cmd/
│   └── myapp/
│       └── main.go           # 入口点
├── internal/
│   ├── handler/              # HTTP handlers
│   ├── service/              # 业务逻辑
│   ├── repository/           # 数据访问
│   └── config/               # 配置
├── pkg/
│   └── client/               # 公开 API 客户端
├── api/
│   └── v1/                   # API 定义（proto, OpenAPI）
├── testdata/                 # 测试数据
├── go.mod
├── go.sum
└── Makefile
```

---

## Go 工具命令

```bash
# 构建运行
go build ./...
go run ./cmd/myapp

# 测试
go test ./...
go test -race ./...
go test -cover ./...

# 静态分析
go vet ./...
staticcheck ./...
golangci-lint run

# 模块管理
go mod tidy
go mod verify

# 格式化
gofmt -w .
goimports -w .
```

---

## 速查表：Go 惯用法

| 惯用法 | 描述 |
|--------|------|
| 接收接口，返回具体类型 | 函数接收接口参数，返回具体类型 |
| 错误是值 | 把错误当作一等值处理，不是异常 |
| 不要通过共享内存来通信 | 使用 channel 协调 goroutine |
| 零值可用 | 类型无需显式初始化就能工作 |
| 少量复制优于少量依赖 | 避免不必要的外部依赖 |
| 清晰优于聪明 | 优先考虑可读性 |
| gofmt 是所有人的朋友 | 始终用 gofmt/goimports 格式化 |
| 尽早返回 | 先处理错误，保持主路径不缩进 |

---

## 反模式避免

```go
// ❌ 长函数中的裸返回
func process() (result int, err error) {
    // ... 50 行 ...
    return // 返回什么？
}

// ❌ 用 panic 控制流程
func GetUser(id string) *User {
    user, err := db.Find(id)
    if err != nil {
        panic(err) // 不要这样
    }
    return user
}

// ❌ 在结构体中存储 context
type Request struct {
    ctx context.Context // Context 应该是第一个参数
    ID  string
}

// ✅ Context 作为第一个参数
func ProcessRequest(ctx context.Context, id string) error {
    // ...
}

// ❌ 混用值接收者和指针接收者
type Counter struct{ n int }
func (c Counter) Value() int { return c.n }    // 值接收者
func (c *Counter) Increment() { c.n++ }        // 指针接收者
// 选择一种风格保持一致
```

**记住**：Go 代码应该以最好的方式保持无聊 - 可预测、一致、易于理解。有疑问时，保持简单。
