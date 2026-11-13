# Go 语言基础 · 阶段 0

> 涉及代码：`cmd/api/main.go`、`internal/config/config.go`、`internal/server/errcode.go`、`internal/server/response.go`、`internal/logger/logger.go`
> 用法说明见 [../README.md](../README.md)

---

## 一、error 体系

### Q1. error 到底是什么？ ⭐ 🎯

**看代码**：`errcode.go` 里的 `func (e *ErrorCode) Error() string`

1. `error` 在 Go 里是关键字、结构体还是接口？它的完整定义只有几行，去标准库里把它找出来。
2. `ErrorCode` 没有写任何 "implements error" 的声明，为什么 `Fail(c, ErrNotFound)` 能把它当 `error` 传进去？这种机制叫什么？它和 Java 的 `implements` 相比，好处和风险各是什么？
3. `Error()` 的接收者是 `*ErrorCode`。那么实现了 `error` 接口的是 `ErrorCode` 还是 `*ErrorCode`？`var e error = ErrorCode{}` 能编译通过吗？

**动手**：在临时文件里写上面那行赋值，读一读编译器的报错信息。

**线索**：鸭子类型（duck typing）、方法集（method set）

---

### Q2. `%w` 和 `%v` 差在哪？ ⭐⭐ 🎯

**看代码**：`main.go` 的 `fmt.Errorf("init mysql: %w", err)`

1. 把 `%w` 换成 `%v`，打印出来的错误文本有区别吗？
2. 既然文本一样，区别在哪？提示：`fmt.Errorf` 用 `%w` 时返回的值多了一个什么方法？
3. 一个错误从 `mysql.New` → `run` → `main` 一路返回，每层都包一句话。最后日志里看到的是 `init mysql: ping mysql: dial tcp ...`。这种"错误链"和 Java 的异常堆栈相比，各自的优缺点是什么？
4. 有没有**不应该**用 `%w` 的场景？（想想：包装之后，底层错误就成了你对外 API 的一部分）

**线索**：Go 官方博客 *Working with Errors in Go 1.13*

---

### Q3. `errors.Is`、`errors.As`、`==` 三者的区别 ⭐⭐ 🎯

**看代码**：`response.go` 的 `Fail` 函数；`server.go` 的 `errors.Is(err, http.ErrServerClosed)`

1. `Fail` 里为什么要先声明 `var errCode *ErrorCode`，再传 `&errCode`（一个**指针的指针**）给 `errors.As`？如果直接传 `errCode` 会怎样？
2. `errors.Is` 是在问"是不是**这个值**"，`errors.As` 是在问"是不是**这种类型**"。`Start()` 用 `Is`，`Fail` 用 `As`，各自为什么选对了？
3. `server.go` 里如果写成 `err != http.ErrServerClosed`，在今天能工作吗？在什么情况下会失效？

**动手**（这题有个坑，值得亲自踩）：

```go
err := fmt.Errorf("wrap: %w", ErrInvalidParam.WithMsg("id 必须大于 0"))
fmt.Println(errors.Is(err, ErrInvalidParam)) // 先预测：true 还是 false？
```

4. 结果符合你的预期吗？为什么？阶段 1 之后 service 层很可能写 `if errors.Is(err, server.ErrSoldOut)`，这会成为隐患吗？
5. 标准库允许自定义错误类型通过实现一个方法来改变 `errors.Is` 的判定逻辑。这个方法签名是什么？如果让你给 `ErrorCode` 加上，应该比较什么字段？

---

### Q4. 为什么 `WithMsg` 要返回新对象？ ⭐⭐

**看代码**：`errcode.go` 的 `WithMsg`

1. 假如把它实现成 `e.Msg = msg; return e`，调用 `ErrInvalidParam.WithMsg("xxx")` 之后，**其他请求**拿到的 `ErrInvalidParam` 会变成什么样？
2. `ErrInvalidParam` 这些变量是包级 `var`，被所有请求的 goroutine 共享。上一问的写法除了"值被改掉"，在并发下还有一个更严重的问题，叫什么？
3. 这些错误码能声明成 `const` 吗？Go 的 `const` 支持哪些类型？这带来一个什么风险（别的包能不能写 `server.ErrNotFound = nil`）？

---

### Q5. nil 不等于 nil？ ⭐⭐⭐ 🎯

**看代码**：`response.go` 里 `errors.As(err, &errCode) && errCode != nil` 的后半句

先预测再运行：

```go
func find() *ErrorCode { return nil }

func main() {
    var err error = find()
    fmt.Println(err == nil)
}
```

1. 输出是什么？为什么？
2. 一个接口值在内存里由哪两部分组成？什么情况下接口值才 `== nil`？
3. 由此得到一条 Go 的编码规范：**函数返回错误时，返回类型应该写 `error` 还是具体的 `*ErrorCode`？** 为什么？

**线索**：Go FAQ *Why is my nil error value not equal to nil?*

---

### Q6. `errors.Join` 的两种用法 ⭐⭐

**看代码**：`config.go` 的 `applyEnvOverrides` 和 `Validate`；`mysql.go` 的 `errors.Join(fmt.Errorf("ping mysql: %w", err), sqlDB.Close())`

1. `applyEnvOverrides` 里 `errs` 被 append 了一堆 `nil`（没出错的 `setIntFromEnv` 返回 nil）。`errors.Join` 怎么处理 nil？如果全是 nil，它返回什么？
2. `Validate` 为什么要收集**所有**错误再一起返回，而不是遇到第一个就 return？站在"部署时配错了 3 项"的运维角度想。
3. `mysql.go` 那一行同时做了两件事：报告 ping 失败、关闭连接池。如果 `sqlDB.Close()` 成功了返回 nil，Join 的结果是什么？这种写法和"先 `_ = sqlDB.Close()` 再 return"相比好在哪？（提示：看看 `.golangci.yml` 里 `check-blank` 的注释）

---

## 二、方法、接口与嵌入

### Q7. 值接收者 vs 指针接收者——一个会导致密码泄露的选择 ⭐⭐⭐ 🎯

**看代码**：`config.go` 的 `func (m MySQLConfig) LogValue() slog.Value`，以及 `main.go` 的 `slog.Any("mysql", cfg.MySQL)`

1. `LogValue` 用的是值接收者 `(m MySQLConfig)`，`Validate` 用的是指针接收者 `(c *Config)`。先说出两者在"能否修改原对象"和"是否发生拷贝"上的区别。
2. 关键问题：类型 `T` 的方法集和 `*T` 的方法集分别包含哪些方法？
3. **动手**：把 `LogValue` 的接收者改成 `(m *MySQLConfig)`，`make run`，观察启动日志里的 password 字段。发生了什么？为什么编译器一声不吭？
4. 这说明"脱敏"这件事完全依赖一个容易改错的细节。有什么办法能让这种回归被自动发现？（看看 `config_test.go` 或 `logger_test.go` 里有没有相应的测试）
5. `fmt.Printf("%+v\n", cfg.MySQL)` 会脱敏吗？`LogValue` 对谁有效、对谁无效？如果想让 `fmt` 也脱敏，需要实现哪个接口？

---

### Q8. 结构体嵌入不是继承 ⭐⭐⭐

**看代码**：`logger.go` 的 `contextHandler`

```go
type contextHandler struct {
    slog.Handler
}
```

1. `slog.Handler` 接口有 4 个方法，`contextHandler` 只显式写了 3 个（`Handle`、`WithAttrs`、`WithGroup`）。第 4 个是什么？它从哪来的？这个机制叫什么？
2. `Handle` 是"重写"了嵌入字段的方法。那么 `WithAttrs` 和 `WithGroup` 只是原样转发后再包一层，**看起来像是多余的**。真的多余吗？
3. **动手**：把 `WithAttrs` 和 `WithGroup` 两个方法删掉，然后跑 `make test`，再 `make run` 并 `curl localhost:8080/healthz`，看访问日志里的 `request_id` 还在不在。
4. 解释现象。提示：`NewWithOptions` 里的 `l.With(slog.String("service", ...))` 返回的新 logger，它内部的 handler 是谁？
5. 用这个例子说明：为什么说 Go 的嵌入是"组合"而不是"继承"？（在 Java 里，父类方法返回 `this` 时，拿到的是子类对象；Go 里呢？）

---

### Q9. 函数也是一种类型 ⭐⭐

**看代码**：`health.go` 的 `type HealthChecker func(ctx context.Context) error`；`main.go` 的 `ser.RegisterChecker("mysql", func(ctx context.Context) error { return mysql.Ping(ctx, db) })`

1. `HealthChecker` 是函数类型。它也可以设计成接口：`type HealthChecker interface { Check(ctx) error }`。两种设计在调用方写起来有什么不同？什么时候选哪种？
2. 传进去的匿名函数里用到了 `db`，但 `db` 并不是它的参数。它是怎么拿到的？这种函数叫什么？
3. `server` 包完全没有 import `mysql` 包和 `redis` 包，却能检查它们的健康状况。这种"让上层把依赖塞进来"的做法叫什么？如果反过来让 `server` 直接 import `mysql`，会有什么坏处？
4. `gin.HandlerFunc`、`http.HandlerFunc` 是同一种思路。`RequestID()` 为什么是"返回一个函数的函数"，而不直接就是一个 handler？对比 `AccessLog(log)` 想一想。

---

### Q10. 空结构体做 key ⭐⭐

**看代码**：`logger.go` 的 `type reqIDKey struct{}` 和 `context.WithValue(ctx, reqIDKey{}, reqID)`

1. 为什么不直接用字符串 `"request_id"` 做 key？假如你引入的某个第三方库也往 context 里放了 key 为 `"request_id"` 的值，会怎样？
2. `reqIDKey` 是小写开头（未导出）。这对"谁能读写这个值"意味着什么？
3. `struct{}` 占多少字节？除了做 context key，它还常用于哪两个场景？（提示：`map[string]struct{}`、`chan struct{}`）
4. `ctx.Value(reqIDKey{}).(string)` 后面带了 `, ok`。不带 `ok` 的类型断言失败时会发生什么？`.golangci.yml` 里哪一项配置在防这个？

---

## 三、defer / panic / recover

### Q11. defer 的三条规则 ⭐⭐ 🎯

**看代码**：`main.go` 的 `run` 函数里的三个 `defer`

1. `run` 返回时，MySQL、Redis 的关闭和 `cancel()` 按什么顺序执行？为什么"先初始化的后关闭"是合理的？
2. 预测输出：

```go
func f() {
    for i := 0; i < 3; i++ {
        defer fmt.Println(i)
    }
}
```

再预测这个：

```go
func g() (n int) {
    defer func() { n *= 2 }()
    return 21
}
```

3. 由此总结：defer 的**参数**何时求值？defer 能不能修改返回值？在什么条件下能？
4. `ctx, cancel := context.WithTimeout(...)` 后面几乎总跟着 `defer cancel()`。既然超时到了 context 自己会取消，为什么还要手动 cancel？不写会泄露什么？

---

### Q12. 为什么要有 `run()`，而不是全写在 `main()` 里？ ⭐⭐ 🎯

**看代码**：`main.go` 的 `main` 与 `run` 的分工

1. `os.Exit(1)` 被调用时，当前 goroutine 里已注册的 `defer` 会执行吗？去读 `os.Exit` 的文档确认。
2. 假设把 `run` 的全部内容搬进 `main`，并在出错时直接 `os.Exit(1)`。MySQL 和 Redis 连接会被正常关闭吗？
3. 现在的 `main` 里有一行 `defer stop()`，下面出错时走 `os.Exit(1)`。这个 `stop()` 会被执行吗？这里有没有实际危害？
4. `run(ctx) error` 这种写法还有一个好处和测试有关，是什么？

---

### Q13. panic 与 recover 的边界 ⭐⭐⭐ 🎯

**看代码**：`middleware.go` 的 `Recovery`

1. `recover()` 必须写在 `defer` 的函数里才有效，为什么？直接在普通代码里调用 `recover()` 会返回什么？
2. `recover` 之后，程序从哪里继续执行？是 panic 的那一行之后，还是别处？
3. **关键边界**：在 handler 里写 `go func() { panic("boom") }()`，`Recovery` 中间件能接住吗？整个进程会怎样？
4. **动手**：加一个临时路由验证上一问。然后思考：秒杀系统后面会有大量异步 goroutine（消费消息、异步落库），应该怎么保护它们？
5. Go 里什么时候该用 `panic`，什么时候该返回 `error`？`ErrSoldOut`（库存卖完）应该是哪种？

---

## 四、类型、标签与标准库细节

### Q14. struct tag 是给谁看的？ ⭐⭐

**看代码**：`config.go` 的 `` `yaml:"read_timeout"` ``、`` `yaml:"-"` ``；`response.go` 的 `` `json:"data,omitempty"` ``

1. tag 只是一段字符串，编译器不解释它。那 `yaml.Unmarshal` 是在**运行时**用什么机制读到它的？
2. 把 `Config` 的字段名改成小写开头（如 `server ServerConfig`），YAML 还能解析进去吗？为什么？
3. `yaml:"-"` 是什么意思？`TestLoad_YAMLAndEnvOverride` 是怎么验证它的？
4. **动手**（先预测）：`Success(c, nil)`、`Success(c, 0)`、`Success(c, []string{})`、`var s []string; Success(c, s)` 这四种调用，响应 JSON 里的 `data` 分别长什么样？哪些会被 `omitempty` 省略？前端同学会不会因为其中某种情况来找你？

---

### Q15. `time.Duration` 是什么类型？ ⭐

**看代码**：`config.go` 的 `ReadTimeout time.Duration` 和 `defaultConfig()` 里的 `5 * time.Second`

1. `time.Duration` 的底层类型是什么？单位是什么？所以 `time.Duration(5)` 表示多久？
2. **动手**：把 `configs/config.yaml` 的 `read_timeout: 5s` 改成 `read_timeout: 5`（去掉单位）。先猜结果是 5 秒、5 纳秒还是报错，再启动看日志。`Validate` 拦得住吗？
3. `middleware.go` 里有个奇怪的时间格式字符串 `"20060102150405.000000000"`。Go 为什么不用 `yyyyMMddHHmmss`？这串数字有什么特殊含义？

---

### Q16. 包名、导入与作用域 ⭐⭐

**看代码**：`internal/infra/mysql/mysql.go` 的头部；`internal/infra/redis/redis.go` 的 `goredis "github.com/redis/go-redis/v9"`

1. `redis.go` 自己的包名是 `redis`，导入的第三方包名也叫 `redis`，所以起了别名 `goredis`。但 `mysql.go` 自己的包名是 `mysql`，又导入了 `gorm.io/driver/mysql`，却**没有**起别名，照样能编译。为什么这个不冲突？（提示：包自己的名字在本包内是不是一个可引用的标识符？）
2. 虽然能编译，这样写对读代码的人友好吗？你会改吗？
3. `main.go` 里 `log := logger.New(...)` 定义了一个叫 `log` 的局部变量。假如这个文件同时 import 了标准库 `"log"`，会发生什么？这种现象叫什么？
4. `go.mod` 里模块路径是 `go-seckill`，而常见写法是 `github.com/用户名/项目名`。两者的区别在什么时候才显现出来？

---

### Q17. `internal/` 是约定还是强制？ ⭐⭐

**看代码**：整个目录结构、`go.mod`

1. 假如另一个项目写 `import "go-seckill/internal/config"`，会发生什么？这是 lint 规则、社区约定，还是编译器强制的？
2. `pkg/` 目录有同样的强制力吗？
3. `go.mod` 里很多依赖带着 `// indirect`，是什么意思？你明明没用 MongoDB，为什么里面有 `go.mongodb.org/mongo-driver`？它会被编译进你的二进制吗？
4. **动手**：`go mod why go.mongodb.org/mongo-driver/v2` 和 `go mod graph | grep mongo`，找出是谁把它带进来的。
5. `go.sum` 是做什么的？它该不该提交到 git？
