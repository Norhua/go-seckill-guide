# Go 并发 · 阶段 0

> 涉及代码：`cmd/api/main.go`、`internal/server/server.go`、`internal/server/health.go`、`internal/logger/logger.go`
> 用法说明见 [../README.md](../README.md)
>
> 阶段 0 的并发代码很少，集中在 `main.go` 的最后 30 行。但秒杀系统的核心就是并发，这 30 行里用到的每个原语后面都会反复出现，值得彻底搞懂。

---

## 一、goroutine

### Q1. 为什么 `ser.Start()` 要放进 goroutine？ ⭐ 🎯

**看代码**：`main.go` 的 `go func() { if err := ser.Start(); ... }()`

1. `ListenAndServe()` 是阻塞调用。如果不加 `go`，直接在 `run` 里调用 `ser.Start()`，后面的 `select` 还有机会执行吗？
2. 反过来：把 `Start()` 放进 goroutine 之后，如果 `run` 函数后面什么都不等、直接 `return nil`，会发生什么？**main goroutine 退出时，其他 goroutine 会怎样？**
3. goroutine 和操作系统线程是什么关系？为什么 Go 可以轻松开几十万个 goroutine，而 Java（虚拟线程之前）开几万个线程就很吃力？这里只需要建立大致印象，关键词是 **GMP 模型**。

**线索**：搜索"Go GMP 调度模型"，找一篇带图的文章读完，能画出 G、M、P 三者关系即可。

---

### Q2. 每个请求是谁在执行？ ⭐⭐ 🎯

**看代码**：`health.go` 的 `handleDebugSlow`（`time.Sleep(5 * time.Second)`）

1. 你没有为请求写过任何 `go` 关键字。那么同时来 100 个 `/debug/slow` 请求时，它们是排队执行（总共 500 秒）还是并发执行（约 5 秒）？
2. **动手**：开 3 个终端同时 `curl localhost:8080/debug/slow`，再开一个 `curl localhost:8080/healthz`。healthz 被堵住了吗？
3. 那么是谁、在哪里替你写了 `go`？去 `net/http` 源码的 `func (srv *Server) Serve(l net.Listener)` 里找到那一行。
4. 这意味着：**你写的每个 handler 天生就运行在并发环境里**。所以 `Server` 结构体里的哪些字段会被多个 goroutine 同时访问？它们各自安全吗？（留到 Q9、Q10 细想）
5. `time.Sleep` 让 goroutine 睡 5 秒时，它占着一个操作系统线程吗？

---

### Q3. goroutine 泄露 ⭐⭐

1. goroutine 没有"句柄"，外部无法强行杀死它。那一个 goroutine 只有哪几种方式能结束？
2. 假如某个 goroutine 永远阻塞在 `ch <- x` 上而没人接收，它会被垃圾回收吗？这叫什么问题？
3. 看 `main.go` 里启动 server 的那个 goroutine：在正常停机流程里，它是怎么结束的？（`Shutdown` 被调用后，`ListenAndServe` 会返回什么？`Start()` 又把它变成了什么？）

---

## 二、channel 与 select

### Q4. `errCh` 的缓冲区 ⭐⭐ 🎯

**看代码**：`main.go` 的 `errCh := make(chan error, 1)`

（存档里讲过这道题。请**不看存档**，自己重新推一遍。）

1. 无缓冲 channel 的发送操作在什么条件下才能完成？有缓冲的呢？
2. 设想这个时序：先收到 Ctrl+C（`select` 走了 `ctx.Done()` 分支，之后再也没人读 `errCh`），随后 server goroutine 往 `errCh` 发送错误。在无缓冲的情况下，这个 goroutine 的命运是什么？
3. 在**当前代码**里，上一问的时序真的会发生吗？（`Start()` 在正常关闭时返回什么？）即便今天不会发生，缓冲为 1 的写法为什么仍然更稳妥？
4. 一般规律："一个 goroutine 只发送一次结果，而接收方可能已经不在了"——这种场景下 channel 容量应该是多少？

---

### Q5. `select` 的语义 ⭐⭐ 🎯

**看代码**：`main.go` 的 `select { case err := <-errCh: ... case <-ctx.Done(): ... }`

1. 两个 case 都没准备好时，`select` 做什么？加一个 `default` 分支后行为怎么变？
2. 两个 case **同时**准备好时，Go 选哪个？是按书写顺序吗？为什么语言要这样设计？
3. `<-ctx.Done()` 没有接收任何值，只是"等它"。`Done()` 返回的 channel 从来没人往里发数据，那这个 case 是怎么被触发的？（提示：从一个**已关闭**的 channel 接收会怎样？）
4. 接上问：为什么"取消"用**关闭 channel** 来表达，而不是往 channel 里**发一个值**？如果有 10 个 goroutine 都在等同一个取消信号呢？
5. 对一个 `nil` channel 做接收会怎样？这个性质在 `select` 里有什么妙用？（了解即可）

---

## 三、context

### Q6. context 是用来解决什么问题的？ ⭐⭐ 🎯

**看代码**：几乎每个函数的第一个参数 `ctx context.Context`

1. 设想没有 context：用户发起请求后关掉了浏览器，而你的 handler 正在执行一条耗时 30 秒的 SQL。你有什么办法让这条 SQL 停下来？
2. context 一共承担哪三种职责？在阶段 0 的代码里各找一个例子：
   - 取消信号（`signal.NotifyContext`）
   - 超时/截止时间（`context.WithTimeout`）
   - 携带请求范围的值（`logger.WithRequestID`）
3. context 之间是**树形**关系。`mysql.New` 里的 `pingCtx` 是从 `ctx` 派生的。如果父 context 被取消，子 context 会怎样？反过来子取消了，父会受影响吗？
4. 为什么 Go 的惯例是把 `ctx` 作为**第一个参数**显式传递，而不是像 Java 的 `ThreadLocal` 那样藏起来？（提示：goroutine 有没有暴露 ID？）

**线索**：Go 官方博客 *Go Concurrency Patterns: Context*

---

### Q7. 为什么停机时要"换一个新的 context"？ ⭐⭐ 🎯

**看代码**：`main.go`

```go
shutdownCtx, cancel := context.WithTimeout(context.Background(), cfg.Server.ShutdownTimeout)
```

1. 执行到这一行时，`run` 的参数 `ctx` 处于什么状态？
2. **动手**：把 `context.Background()` 换成 `ctx`，启动后先 `curl localhost:8080/debug/slow`，立刻 Ctrl+C。那个慢请求的结局和之前有什么不同？为什么？
3. 一个已取消的 context 派生出的子 context，在诞生的那一刻是什么状态？
4. 类比题：`integration_test.go` 的 `t.Cleanup` 里也用了 `context.Background()` 而不是外层 `ctx`，是同一个道理吗？
5. Go 1.21 增加了 `context.WithoutCancel`。它和这里的 `context.Background()` 相比，多保留了什么？在这个场景下有什么好处（想想日志里的 request_id）？

---

### Q8. context 里该放什么，不该放什么？ ⭐⭐

**看代码**：`logger.go` 的 `WithRequestID` / `WithContext` / `FromContext`

1. `ctx.Value(key)` 的查找过程是怎样的？如果 context 链很长，查找的时间复杂度是多少？
2. `request_id` 放进 context 是公认的合理用法。那么"当前登录的用户 ID"适合吗？"数据库连接 `*gorm.DB`"适合吗？"函数的可选参数"呢？试着总结一条判断标准。
3. `context.WithValue` 没有修改原来的 ctx，而是返回了一个新的。结合"handler 运行在并发环境里"，这种**不可变**设计的好处是什么？
4. `context.Background()` 和 `context.TODO()` 行为完全一样，为什么要有两个？

---

## 四、共享状态与竞态

### Q9. `atomic.Bool` 换成普通 `bool` 行不行？ ⭐⭐⭐ 🎯

**看代码**：`server.go` 的 `shuttingDown atomic.Bool`；`health.go` 的 `Store(true)` / `Load()`

1. 谁在写这个字段（哪个 goroutine）？谁在读？它们可能同时发生吗？
2. 很多人的直觉是："bool 就一个字节，读写怎么可能出问题？"这个直觉错在哪？请从这两个角度各说一点：**CPU 缓存/可见性**、**编译器优化**。
3. Go 对"数据竞争"的官方态度是什么——是"可能读到旧值但无伤大雅"，还是"程序行为未定义"？
4. **动手**：把字段改成普通 `bool`（相应改掉 `Store`/`Load`），然后写一个测试：一个 goroutine 循环请求 `/readyz`，另一个调用 `MarkShuttingDown()`，用 `go test -race` 跑。观察 race detector 的报告长什么样，学会读它。
5. 同样是保护共享状态，`atomic`、`sync.Mutex`、channel 三者各自适合什么场景？这里为什么选 atomic？

---

### Q10. 那个 map 安全吗？ ⭐⭐⭐

**看代码**：`server.go` 的 `checkers map[string]HealthChecker`；`health.go` 的 `RegisterChecker` 和 `handleReadyz` 里的 `for name, checker := range s.checkers`

1. Go 的 map 并发读写会怎样？是悄悄出错，还是直接崩溃？（和 Q9 的 bool 对比，后果的"响亮程度"不同）
2. 多个 goroutine **只读**同一个 map 安全吗？
3. 当前代码里，`RegisterChecker`（写）都发生在什么时刻？`handleReadyz`（读）从什么时刻开始？所以现在安全吗？这个安全是靠**代码保证**的，还是靠**调用方自觉**？
4. 假如将来有人在服务运行期间调用 `RegisterChecker`，会发生什么？有哪几种办法把"自觉"变成"保证"？
5. `MarkShuttingDown()` 在 `main.go` 里调了一次，`Server.Shutdown()` 里又调了一次。重复调用有问题吗？具备这种性质的操作叫什么？（这个词在秒杀系统里极其重要）

---

### Q11. race detector 能发现什么、不能发现什么？ ⭐⭐

**看代码**：`Makefile` 的 `go test -race -v ./...`

1. `-race` 是静态分析（读代码）还是动态检测（跑代码）？这决定了它的一个根本局限：**没被执行到的竞态**能被发现吗？
2. 开启 `-race` 后，程序的内存和 CPU 开销大约增加多少？所以线上服务一般会开吗？
3. "通过了 `-race` 测试"能证明"代码没有并发 bug"吗？举一个 race detector 检测不出来、但确实是并发 bug 的例子。（提示：两个操作各自都加了锁，但"先查库存、再扣库存"之间……这就是秒杀超卖的本质，阶段 2 会正面遇到）

---

## 五、信号

### Q12. Ctrl+C 之后发生了什么？ ⭐⭐ 🎯

**看代码**：`main.go` 的 `signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)`

1. 按 Ctrl+C 时，是谁给进程发了什么信号？`kill <pid>` 默认发的是什么信号？`docker stop` 和 K8s 删除 Pod 时呢？
2. `kill -9` 发的是 `SIGKILL`。为什么代码里没有监听它？监听了有用吗？所以"优雅停机"防不住哪种情况？这对系统设计意味着什么？（提示：**进程随时可能在任意一行代码处死亡**，秒杀扣了 Redis 库存、还没写 MySQL 时被 kill -9 了怎么办？）
3. Go 程序默认（不调用 `signal.Notify`）收到 SIGINT 会怎样？调用了 `NotifyContext` 之后，默认行为还在吗？
4. **动手**：`curl localhost:8080/debug/slow` 后按 Ctrl+C，在等待期间**再按一次** Ctrl+C。进程立刻退出了吗？为什么？
5. 很多成熟的服务支持"第一次 Ctrl+C 优雅退出，第二次强制退出"。基于 `stop()` 函数的文档说明，想想应该在哪里调用它来实现这个效果。
