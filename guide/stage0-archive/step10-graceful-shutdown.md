# User:
---
我正在第0阶段，指导我完成Step 10
---
# Gemini 4 Argon (High):
---
你在前面的步骤中其实已经为 Step 10 埋下了非常好的伏笔：
- [internal/config/config.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/config/config.go#L21-L28) 中已经定义了 `ShutdownTimeout`（默认 `10s`，可通过 `SERVER_SHUTDOWN_TIMEOUT` 覆盖）。
- [internal/server/health.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/health.go#L29-L50) 中已经用 `atomic.Bool` 实现了 `MarkShuttingDown()`，且 `/readyz` 会第一时间检查它并返回 `503`。
- [internal/server/server.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/server.go#L91-L95) 中已经封装好了 `Shutdown(ctx context.Context) error`。
- [cmd/api/main.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/cmd/api/main.go#L28-L86) 中也已经拆分出了 `run(ctx context.Context) error`（这其实已经把 **Step 11** 的主干结构也做完了）。

下面我们先花两分钟搞懂优雅停机的底层原理，然后动手把最后一块拼图补齐。

---

### 一、原理科普：为什么需要优雅停机？底层发生了什么？

#### 1. 如果直接强杀进程会发生什么？
在秒杀系统（尤其是后续阶段引入异步下单、Redis 预扣减、消息队列 Worker）中，如果按下 `Ctrl+C`（发送 `SIGINT`）或容器平台发版滚动更新（发送 `SIGTERM`）时进程直接退出：
- **在途请求中断**：用户刚扣完 Redis 库存，正准备写库或发 MQ，进程突然挂了，导致**少卖（库存泄漏）**或**数据不一致**。
- **网关流量误杀**：负载均衡（Nginx / K8s Service）是通过定期轮询 `/readyz` 感知节点状态的。如果进程瞬间消失，网关还没来得及把它从转发列表中摘除，新打过来的请求就会全部报 `Connection Reset / 502 Bad Gateway`。

#### 2. 标准库 `http.Server.Shutdown(ctx)` 底层做了哪三件事？
当你调用 `s.httpServer.Shutdown(shutdownCtx)` 时，Go 标准库 `net/http` 的内部流程是：
1. **立刻关闭 `net.Listener`**：不再 `Accept` 任何新的 TCP 连接。
2. **立刻关闭所有空闲连接（Idle Keep-Alive Connections）**：防止客户端复用旧的空闲连接发新请求。
3. **轮询等待所有活跃连接（Active Connections）变为空闲**：
   - **注意**：`Shutdown(shutdownCtx)` **不会**主动取消正在处理的请求的 `c.Request.Context()`！因为我们的目的正是让在途请求顺利把数据库/Redis 操作执行完。
   - 如果在途请求在 `shutdownCtx` 超时前全部处理完毕，`Shutdown` 返回 `nil`；而原来的 `ListenAndServe()` 会立刻返回 `http.ErrServerClosed`（你在 [server.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/server.go#L85) 第 85 行已经用 `!errors.Is(err, http.ErrServerClosed)` 正确忽略了它）。
   - 如果 `shutdownCtx` 到期仍有请求没处理完，`Shutdown` 会直接返回 `context.DeadlineExceeded`，不再傻等。

#### 3. 为什么要「按初始化的逆序」关闭资源？
- **启动顺序**：`MySQL` $\rightarrow$ `Redis` $\rightarrow$ `HTTP Server`（底层基础设施先就绪，最后才对外开门迎客）。
- **关闭顺序**：`HTTP Server` $\rightarrow$ `Redis` $\rightarrow$ `MySQL`（先关门并等店里的客人全部结账离开，最后才能关掉收银台和后厨）。如果先关了 MySQL/Redis，正在处理的 HTTP 请求执行到一半去查库就会直接报错 `sql: database is closed`。
- **Go 的 `defer` 天然契合逆序关闭**：Go 语言中的 `defer` 采用**栈（LIFO，后进先出）**结构存储。先 `defer mysql.Close()`，再 `defer rdb.Close()`，函数返回时会先执行 `rdb.Close()`，再执行 `mysql.Close()`。

---

### 二、代码实现指南

#### 1. 在 `internal/server` 中补充 `/debug/slow` 并在 `Shutdown` 时标记状态

打开 [internal/server/server.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/server.go)：

1. 在 `New()` 注册 `/healthz` 和 `/readyz` 的下方（第 72 行附近），加上临时调试路由 `/debug/slow`：
   ```go
   srv.engine.GET("/healthz", srv.handleHealthz)
   srv.engine.GET("/readyz", srv.handleReadyz)
   srv.engine.GET("/debug/slow", srv.handleDebugSlow)
   ```
2. 在 `Shutdown` 方法中顺便调用 `s.MarkShuttingDown()`（做一层兜底保护）：
   ```go
   // Shutdown 优雅停机（供 Step 10 使用）
   func (s *Server) Shutdown(ctx context.Context) error {
   	s.MarkShuttingDown()
   	s.log.InfoContext(ctx, "http server shutting down")
   	return s.httpServer.Shutdown(ctx)
   }
   ```
3. 在 [internal/server/health.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/health.go) 末尾（或者单独建个文件）实现 `handleDebugSlow`：
   ```go
   // handleDebugSlow 模拟慢请求（默认耗时 5s），用于验证优雅停机
   func (s *Server) handleDebugSlow(c *gin.Context) {
   	s.log.InfoContext(c.Request.Context(), "slow request started, sleeping 5s...")
   	time.Sleep(5 * time.Second)
   	s.log.InfoContext(c.Request.Context(), "slow request finished")

   	Success(c, gin.H{
   		"status": "slow request completed",
   	})
   }
   ```

---

#### 2. 改造 `cmd/api/main.go` 实现信号监听与并发启停编排

目前 [cmd/api/main.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/cmd/api/main.go#L81-L83) 最后直接调用了 `if err := ser.Start(); err != nil`，由于 `ListenAndServe` 是阻塞调用，主协程会一直卡在那里。

我们需要改成：**在独立 goroutine 中启动 HTTP Server，主协程通过 `select` 同时等待「启动报错」或「系统停机信号」**。

参考修改 [cmd/api/main.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/cmd/api/main.go)：

1. 引入所需标准库包：
   ```go
   import (
   	"context"
   	"fmt"
   	"log/slog"
   	"os"
   	"os/signal"
   	"syscall"
   	// ...
   )
   ```
2. 在 `main()`（或 `run()` 开头）使用 `signal.NotifyContext` 监听 `SIGINT`（Ctrl+C）和 `SIGTERM`（Docker/K8s 默认停止信号）：
   ```go
   func main() {
   	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
   	defer stop()

   	if err := run(ctx); err != nil {
   		slog.Error("application exited with error", slog.Any("error", err))
   		os.Exit(1)
   	}
   }
   ```
3. 在 `run(ctx context.Context)` 中完善 `defer` 的关闭日志（注意：`defer` 触发时 `ctx` 已经被信号取消了，打印停机日志可以直接用 `log.Info` / `log.Error`）：
   ```go
   	// 初始化 MySQL（第 1 个注册 defer，最后执行关闭）
   	db, err := mysql.New(ctx, cfg.MySQL)
   	if err != nil {
   		return fmt.Errorf("init mysql: %w", err)
   	}
   	defer func() {
   		if err := mysql.Close(db); err != nil {
   			log.Error("failed to close mysql", slog.Any("error", err))
   		} else {
   			log.Info("mysql connection closed")
   		}
   	}()
   	log.InfoContext(ctx, "mysql connected successfully")

   	// 初始化 Redis（第 2 个注册 defer，倒数第 2 个执行关闭）
   	rdb, err := redis.New(ctx, cfg.Redis)
   	if err != nil {
   		return fmt.Errorf("init redis: %w", err)
   	}
   	defer func() {
   		if err := rdb.Close(); err != nil {
   			log.Error("failed to close redis", slog.Any("error", err))
   		} else {
   			log.Info("redis connection closed")
   		}
   	}()
   	log.InfoContext(ctx, "redis connected successfully")
   ```
4. 替换原来阻塞的 `ser.Start()`，改为异步启动 + `select` 阻塞等待 + 优雅停机：
   ```go
   	// 在后台 goroutine 启动 HTTP 服务，并用缓冲为 1 的 channel 接收非正常退出错误（如端口被占用）
   	errCh := make(chan error, 1)
   	go func() {
   		if err := ser.Start(); err != nil {
   			errCh <- err
   		}
   	}()

   	// 阻塞等待：要么启动/运行报错，要么收到 SIGINT/SIGTERM 信号
   	select {
   	case err := <-errCh:
   		return fmt.Errorf("start http server: %w", err)
   	case <-ctx.Done():
   		log.Info("shutdown signal received, starting graceful shutdown...",
   			slog.Duration("timeout", cfg.Server.ShutdownTimeout),
   		)
   	}

   	// 1. 立即标记停机状态，使 /readyz 返回 503（拒绝新流量）
   	ser.MarkShuttingDown()

   	// 2. 创建全新的超时 Context（注意不能用已经取消的 ctx！）等待在途请求完成
   	shutdownCtx, cancel := context.WithTimeout(context.Background(), cfg.Server.ShutdownTimeout)
   	defer cancel()

   	if err := ser.Shutdown(shutdownCtx); err != nil {
   		log.Error("http server graceful shutdown timed out, forcing exit", slog.Any("error", err))
   		return fmt.Errorf("http server shutdown: %w", err)
   	}

   	log.Info("http server stopped gracefully")
   	// 函数返回后，将按 LIFO 顺序触发 defer：先关闭 Redis，再关闭 MySQL
   	return nil
   ```

> **重点细节提醒**：创建 `shutdownCtx` 时，父 Context 必须传 `context.Background()`，**千万不能传 `ctx`**！因为此时 `ctx` 已经被 `SIGINT` 取消了，如果以 `ctx` 为父节点派生 `WithTimeout`，生成的 `shutdownCtx` 从诞生的第一纳秒起就是 `Canceled` 状态，`ser.Shutdown(shutdownCtx)` 会瞬间超时失败！

---

### 三、如何验证（达到完成标准）

代码改好后，打开两个终端窗口分别测试以下两个场景：

#### 场景 1：正常优雅停机（在途请求 5s < 停机超时 10s）
1. **终端 1** 启动服务：
   ```bash
   make run
   ```
2. **终端 2** 发起慢请求（并可以顺便观察 `/readyz`）：
   ```bash
   curl -i http://localhost:8080/debug/slow
   ```
3. 在 `curl` 还在等待的 5 秒内，切回 **终端 1** 按下 `Ctrl+C`（也可以同时在终端 2 执行 `curl -i http://localhost:8080/readyz` 看到返回 `503`）。
4. **预期现象**：
   - 终端 1 立刻打印 `shutdown signal received`，但**进程没有立刻退出**。
   - 满 5 秒后，终端 2 的 `/debug/slow` 正常收到 `HTTP/1.1 200 OK` 和 JSON 响应。
   - 紧接着终端 1 依次打印：
     1. `http server stopped gracefully`
     2. `redis connection closed`
     3. `mysql connection closed`
     随后进程干净退出（退出码 0）。

#### 场景 2：超时强制退出（在途请求 5s > 停机超时 2s）
1. **终端 1** 用环境变量把停机超时临时缩短为 `2s` 启动：
   ```bash
   SERVER_SHUTDOWN_TIMEOUT=2s make run
   ```
2. **终端 2** 再次请求 `curl -i http://localhost:8080/debug/slow`，并在 1 秒内回 **终端 1** 按下 `Ctrl+C`。
3. **预期现象**：
   - 等待满 2 秒后，终端 1 打印 `http server graceful shutdown timed out, forcing exit`（`error="context deadline exceeded"`），随后关闭 Redis、MySQL 并以非 0 状态码退出。
   - 终端 2 的 `curl` 连接被断开（`Empty reply from server`）。

---

### 四、思考题（面试高频追问）

完成验证后，可以思考以下两个问题（可以直接回复你的想法，我们再进入 Step 11/12）：

1. **为什么 `errCh := make(chan error, 1)` 要设置缓冲区大小为 `1`，如果写成无缓冲的 `make(chan error)` 会有什么隐患？**
2. **在 K8s 生产环境中，当我们收到 `SIGTERM` 并把 `/readyz` 设为 `503` 时，K8s 的 Ingress / Service 摘除节点是异步的（可能存在 1~2 秒延迟）。如果我们在收到信号的同一毫秒立刻执行 `httpServer.Shutdown()` 关闭监听端口，这 1~2 秒内还在往这个 Pod 转发的新请求会怎样？在真实生产环境中通常怎么解决？**
---
# User:
---
回答这两个问题
---
# Gemini 4 Argon (High):
---
我看了你更新后的 [cmd/api/main.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/cmd/api/main.go)、[internal/server/server.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/server.go) 和 [internal/server/health.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/health.go)，实现得非常规范！而且你的 `main.go` 同时也满足了 **Step 11（组装入口 `cmd/api/main.go`）** 的全部要求，我已经帮你把 [stage0-todolist.md](file:///Users/amadeus/work-space/learning/go-seckill/guide/stage0-todolist.md#L167-L193) 中的 **Step 10** 和 **Step 11** 都打上了勾。

下面详细解答这两个面试高频追问：

---

### 问题 1：为什么 `errCh := make(chan error, 1)` 要设缓冲区大小为 `1`？如果用无缓冲 `make(chan error)` 会有什么隐患？

**核心答案：防止 Goroutine 泄漏（Goroutine Leak）。**

我们来看这段代码：
```go
errCh := make(chan error, 1) // 假设这里写成了无缓冲的 make(chan error)
go func() {
	if err := ser.Start(); err != nil {
		errCh <- err // 发送方
	}
}()

select {
case err := <-errCh: // 接收方
	return fmt.Errorf("start http server: %w", err)
case <-ctx.Done():
	log.Info("shutdown signal received...")
}
```

- **无缓冲 Channel 的特性**：发送操作 `errCh <- err` 必须等到有另一个协程执行 `<-errCh` 接收时才能完成，否则发送协程会**永久阻塞**。
- **极端并发竞争场景**：
  假设在收到 `SIGINT`（`<-ctx.Done()` 触发）的同一微秒，`ser.Start()` 恰好因为某个网络异常（非 `http.ErrServerClosed`）返回了一个 `err`。
  1. 主协程在 `select` 中随机命中了 `case <-ctx.Done():`，于是离开了 `select` 往下走去执行停机逻辑，**再也没有人会去读 `errCh` 了**。
  2. 后台启动的那个子协程执行到 `errCh <- err`，如果 `errCh` 是无缓冲的，这个子协程就会永远卡死在这一行，直到整个进程退出都无法被 GC 回收。
- **而在单元测试 / `run(ctx)` 被多次调用的场景下**：如果进程不退出（比如集成测试里多次调用 `run`），泄漏的 Goroutine 就会一直驻留在内存中。
- **加上缓冲区 `1` 后**：哪怕主协程已经离开了 `select` 不再读取 `errCh`，子协程往容量为 1 的 `errCh` 塞入一个 `err` 时也是**非阻塞**的，塞完立刻退出，协程干干净净地结束生命周期。

> **Go 并发编程经验法则**：只要使用 `goroutine + channel` 汇报单次结果（One-shot result/error），且主协程有可能因为 `ctx.Done()` 或超时提前离开 `select`，**Channel 容量一定要设为 `1`**。

---

### 问题 2：在 K8s 生产环境中，收到 `SIGTERM` 后立刻调 `httpServer.Shutdown()` 关闭监听端口会有什么问题？生产环境怎么解决？

#### 1. 为什么会有 1~2 秒的“流量黑洞”？
当 K8s 决定销毁一个 Pod（比如滚动发版或缩容）时，它会**同时并发**做两件事，而不是串行：
- **路线 A（停容器）**：kubelet 直接给你的 Go 进程发送 `SIGTERM` 信号（耗时几毫秒）。
- **路线 B（摘流量）**：API Server 通知各个节点的 `kube-proxy`（更新 iptables/IPVS 规则）和 `Ingress Controller / Envoy`（更新 upstream 列表），把这个 Pod 的 IP 从转发列表里剔除（这条分布式链路通常需要 **1 ~ 3 秒**才能同步完）。

如果你的 Go 进程在收到 `SIGTERM` 的**第 1 毫秒**就执行了 `httpServer.Shutdown()`：
- 你的进程立刻关闭了 `:8080` 的 TCP Listener。
- 但在接下来的 1~2 秒内，Nginx / Envoy 还没收到摘除通知，仍然把新用户的秒杀请求往这个 Pod 上转发。
- 结果：这 1~2 秒内打过来的新请求全部遭遇 `TCP RST / Connection Refused`，前端直接报 **502 Bad Gateway**（即发版抖动）。

#### 2. 真实生产环境的两种标准解法

- **解法一（K8s 基础设施层，最推荐）：配置 `preStop` Hook**
  在 K8s 的 Pod YAML 中配置一个 `preStop` 钩子（例如 `sleep 3`）：
  ```yaml
  lifecycle:
    preStop:
      exec:
        command: ["sh", "-c", "sleep 3"]
  ```
  因为 K8s 会在触发 `preStop` 的同时去摘除 Service/Ingress 里的节点，等 3 秒钟网关把流量彻底切走之后，K8s 才会真正给你的 Go 进程发 `SIGTERM`。这样 Go 代码完全不需要耦合基础设施的等待时间。

- **解法二（应用代码层）：先 `MarkShuttingDown()`，小睡片刻再 `Shutdown()`**
  如果不依赖 K8s 的 `preStop`，也可以在 `main.go` 收到信号后分两步走：
  1. 先调用 `ser.MarkShuttingDown()` 让 `/readyz` 变成 `503`（此时 `:8080` 依然正常接收请求，已经在路上的新请求还能被正常处理）。
  2. 等待一个短暂的传播窗口（比如 `time.Sleep(2 * time.Second)`），等网关彻底不再发新请求过来。
  3. 最后再调用 `ser.Shutdown(shutdownCtx)` 关闭监听并排空在途请求。

---

理解了这两个问题后，你在面试中聊到“Go 服务的优雅停机”就已经超过绝大多数只会背 `http.Server.Shutdown` API 的候选人了。

要不要现在进入 **Step 12：测试基础设施**？ :]
---
