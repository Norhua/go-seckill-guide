# Gin 框架 · 阶段 0

> 涉及代码：`internal/server/server.go`、`internal/server/middleware.go`、`internal/server/response.go`
> 用法说明见 [../README.md](../README.md)
>
> 学框架的要点是分清"哪些是 Gin 做的，哪些是标准库 `net/http` 做的"。建议先做完 [network-http](../network-http/stage0.md)。

---

## 一、Gin 与 net/http 的关系

### Q1. `Handler: engine` 这一行为什么能成立？ ⭐⭐ 🎯

**看代码**：`server.go`

```go
httpSrv := &http.Server{
    Addr:    cfg.Addr(),
    Handler: engine,
    ...
}
```

1. `http.Server.Handler` 字段的类型是 `http.Handler`。这个接口只有一个方法，它的签名是什么？
2. `*gin.Engine` 能赋值给它，说明什么？去 Gin 源码里找到 `func (engine *Engine) ServeHTTP`，粗读一下：它拿到 `w` 和 `req` 之后做了哪几步？
3. 现在划分职责：**监听端口、accept 连接、解析 HTTP 报文、为每个连接开 goroutine、超时控制**——这些是谁做的？**路由匹配、中间件链、参数绑定、JSON 输出**——这些是谁做的？
4. 所以"Gin 是一个 Web 服务器"这句话准确吗？更准确的说法是什么？
5. `engine.Run(":8080")` 内部做了什么？为什么我们不用它？

---

### Q2. `gin.New()` vs `gin.Default()` ⭐

**看代码**：`server.go` 的 `engine := gin.New()` 和 `gin.SetMode(gin.ReleaseMode)`

1. `gin.Default()` 比 `gin.New()` 多了哪两个中间件？
2. 我们自己写了 `AccessLog` 和 `Recovery` 替换它们。自带的 Logger 输出是什么格式？和我们的 slog JSON 日志混在一起，对日志采集系统来说有什么问题？
3. `gin.SetMode` 修改的是全局状态还是这个 engine 的状态？在 `New()` 这种"构造函数"里修改全局状态，有什么潜在问题？（想想测试里创建多个 Server 的情况）

---

### Q3. 路由是怎么匹配的？ ⭐⭐⭐

**看代码**：`server.go` 的 `srv.engine.GET("/healthz", srv.handleHealthz)`

1. 最朴素的路由实现是 `map[string]Handler`。它处理不了哪种路由？（想想阶段 1 会出现的 `/api/v1/activities/:id`）
2. 另一种朴素实现是"按注册顺序逐个正则匹配"。路由有 500 条时，它的问题是什么？
3. Gin 使用的数据结构叫**基数树（radix tree）**，每个 HTTP 方法一棵。它是怎么同时解决上面两个问题的？画一下 `/healthz`、`/readyz`、`/debug/slow` 三条路由在树里的样子（不用精确，体会"公共前缀合并"即可）。
4. Go 1.22 起，标准库 `http.ServeMux` 也支持 `mux.HandleFunc("GET /items/{id}", ...)`。那今天选 Gin 的理由还剩哪些？

**线索**：搜索"httprouter radix tree"。

---

## 二、中间件

### Q4. 洋葱模型是怎么实现的？ ⭐⭐⭐ 🎯

**看代码**：`middleware.go` 三个中间件里的 `c.Next()`

1. 用自己的话描述：`c.Next()` **之前**的代码和**之后**的代码，分别在什么时机执行？
2. 一个请求打到 `/healthz`，写出以下 7 个"时刻"的实际执行顺序：RequestID 前半、AccessLog 前半、Recovery 的 defer 注册、handler、Recovery 的 defer 执行、AccessLog 后半、RequestID 的 `c.Next()` 返回。
3. 去读 Gin 源码里的 `func (c *Context) Next()`，它只有几行。`c.index` 和 `c.handlers` 分别是什么？"洋葱"的本质是递归调用还是别的？
4. 如果一个中间件**忘了**调用 `c.Next()` 也没调 `c.Abort()`，后面的 handler 还会执行吗？（读完第 3 问的源码你就能回答，答案可能出乎意料）
5. `c.Abort()` 做了什么？调用 `c.Abort()` 之后，**当前函数**里它后面的代码还会执行吗？所以鉴权中间件里 `c.Abort()` 后面必须跟什么？

---

### Q5. 中间件的顺序 ⭐⭐ 🎯

**看代码**：`server.go` 的 `engine.Use(RequestID(), AccessLog(log), Recovery(log))`

对以下每种调整，预测会发生什么，然后**至少动手验证一种**（用 `server_test.go` 里的 `/test/panic` 思路，或临时加一个会 panic 的路由）：

1. 把 `Recovery` 移到最前面（`Recovery → RequestID → AccessLog`）：handler panic 时，访问日志里记录的 `status` 是多少？为什么？
2. 把 `RequestID` 移到最后：访问日志和 panic 日志里还有 `request_id` 吗？
3. 当前顺序下，handler panic 时访问日志记录的是 500。这依赖于 `Recovery` 在 `AccessLog` 的"里面"。用洋葱模型解释为什么。
4. 阶段 1 之后会加入鉴权、限流中间件。限流应该放在鉴权的前面还是后面？各有什么道理？（没有标准答案，说出权衡即可）

---

### Q6. Recovery 的细节 ⭐⭐⭐

**看代码**：`middleware.go` 的 `Recovery`

1. `net/http` 本身就会 recover handler 里的 panic（进程不会崩）。**动手**：临时把 `Recovery` 从 `Use` 里去掉，请求一个会 panic 的路由，看客户端收到什么、服务端打印了什么。据此说明我们自己的 `Recovery` 提供了哪两点价值。
2. `debug.Stack()` 把堆栈记进了日志。为什么堆栈**只进日志、不进响应**？
3. 棘手的情况：handler 已经调用 `c.JSON(200, ...)` 写出了响应，**然后**才 panic。这时 `Recovery` 里的 `Fail(c, ErrInternal)` 还能把状态码改成 500 吗？客户端最终收到什么？（提示：HTTP 响应的状态行一旦发出……）
4. 回顾 go-basics Q13：`Recovery` 保护不了哪种 panic？

---

### Q7. Request ID 的信任问题 ⭐⭐

**看代码**：`middleware.go` 的 `RequestID` 和 `generateRequestID`

1. request_id 的用途是什么？设想线上用户报障："我 3 点 05 分下单失败了"，有和没有 request_id，排查过程分别是怎样的？
2. 中间件优先采用客户端传来的 `X-Request-ID`。这是为了什么场景设计的？（提示：请求在到达你之前可能已经过网关；你也可能再调用别的服务）
3. **动手**：`curl -H 'X-Request-ID: <一个 10 万字符的字符串>' localhost:8080/healthz`，看日志。再想想如果值里包含换行符，对**文本格式**的日志会造成什么干扰。对客户端传来的 ID，应该加什么限制？
4. `generateRequestID` 用的是 `crypto/rand` 而不是 `math/rand`。二者的区别是什么？request_id 需要"密码学安全"吗？那什么东西**必须**用 `crypto/rand`（想想阶段后面的秒杀令牌）？
5. 函数里有一个"`rand.Read` 出错则回退到时间戳"的分支。去读你所用 Go 版本的 `crypto/rand.Read` 文档：它什么情况下会返回 error？这个分支还有机会执行吗？
6. 延伸了解：业界有一个传播链路 ID 的标准请求头叫 `traceparent`（W3C Trace Context）。阶段 6 做可观测性时会用到，先知道这个名字。

---

## 三、gin.Context

### Q8. 两个 Context ⭐⭐⭐ 🎯

**看代码**：`middleware.go`

```go
ctx := logger.WithRequestID(c.Request.Context(), reqID)
c.Request = c.Request.WithContext(ctx)
```

以及 `server.go` 的 `engine.ContextWithFallback = true`

1. `*gin.Context`（`c`）和 `c.Request.Context()` 是两个不同的东西。各自是什么、各自的生命周期由谁管理？
2. 为什么 request_id 要存进 `c.Request.Context()`，而不是用更顺手的 `c.Set("request_id", reqID)`？设想 service 层、repository 层的函数签名是 `func (s *Service) Buy(ctx context.Context, ...)`——它们拿得到 `c.Set` 存的值吗？应不应该让 service 层依赖 `*gin.Context`？
3. `c.Request.WithContext(ctx)` 返回的是一个**新的** `*http.Request`，所以必须赋值回 `c.Request`。如果忘了赋值会怎样？
4. `*gin.Context` 自己也实现了 `context.Context` 接口。`ContextWithFallback = true` 打开之后，把 `c` 直接当 ctx 传给 `log.InfoContext(c, ...)`，还能取到 request_id 吗？关掉呢？
5. 即便开了 fallback，团队规范通常仍要求"往下层传 `c.Request.Context()`，不要传 `c`"。结合下一题想想为什么。

---

### Q9. `gin.Context` 是会被回收复用的 ⭐⭐⭐ 🎯

1. Gin 的 `ServeHTTP` 开头有一句 `c := engine.pool.Get().(*Context)`，结尾有 `engine.pool.Put(c)`。`pool` 是什么类型？为什么要这样做而不是每个请求 new 一个？
2. 这意味着请求结束后，这个 `c` 对象会被**下一个请求**拿去重用。那么下面的代码有什么问题？

```go
func handler(c *gin.Context) {
    go func() {
        time.Sleep(time.Second)
        log.Println(c.Request.URL.Path) // ?
    }()
    c.JSON(200, gin.H{"ok": true})
}
```

3. Gin 为这种场景提供了哪个方法？
4. 这是一类通用的坑：**对象池里拿出来的东西，归还之后绝不能再用**。阶段 3 之后你自己也可能用 `sync.Pool` 做优化，记住这条。

---

## 四、统一响应

### Q10. `Fail` 把错误"吞"了 ⭐⭐ 🎯

**看代码**：`response.go` 的 `Fail` 的最后一个分支；`server_test.go` 的 `unknown error masked as 500`

1. 未知错误被替换成 `ErrInternal` 再返回，防止了信息泄露，这是对的。但**原始错误**（比如 `Error 1062: Duplicate entry...`）去哪了？在整个请求的处理过程中，它被记录到任何地方了吗？
2. 设想线上出现大量 500，你打开日志，能看到的只有访问日志里的 `status=500`。能定位原因吗？
3. 如果要补上，你会在哪里记录这条日志？列出至少两种方案并比较：
   - 在 `Fail` 内部打日志（需要 logger 从哪来？）
   - 用 `c.Error(err)` 把错误挂到 gin.Context 上，由 `AccessLog` 统一输出
4. `Fail` 对 4xx 类的业务错误（如参数错误）需要打日志吗？打什么级别？

---

### Q11. `Fail` 之后要不要 `return`？ ⭐

**看代码**：`response.go`；`health.go` 的 `handleReadyz` 里每个 `c.JSON(...)` 后面的 `return`

1. `c.JSON(...)` 会终止 handler 函数的执行吗？
2. 如果在 handler 里写了 `Fail(c, err)` 却忘了 `return`，后面又执行到 `Success(c, data)`，客户端收到的响应会是什么样？**动手**试一下。（Gin 只在 debug 模式下才会为此打印警告，而我们设了 `ReleaseMode`；可以临时注释掉 `gin.SetMode` 那一行再试，看看警告内容。）
3. 这是 Gin 新手最常见的 bug 之一。有什么编码习惯能避免它？

---

### Q12. 客户端 IP 可信吗？ ⭐⭐⭐

**看代码**：`middleware.go` 的 `slog.String("client_ip", c.ClientIP())`

1. 请求经过 Nginx 反向代理后到达你的服务，TCP 连接的对端地址是谁的 IP？真实用户的 IP 靠什么传递过来？
2. `X-Forwarded-For` 只是一个普通请求头，客户端可以随便伪造。**动手**：`curl -H 'X-Forwarded-For: 1.2.3.4' localhost:8080/healthz`，看访问日志里的 `client_ip`。
3. 查一查 Gin 的 `SetTrustedProxies`：默认信任哪些代理？对当前代码意味着什么？
4. 为什么这在秒杀系统里尤其重要？（提示：阶段 4 的限流很可能"按 IP 限流"。如果 IP 可以伪造，限流还有用吗？）
