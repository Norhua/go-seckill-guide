# User:
---
我正在第0阶段，指导我完成Step 9
---
# Gemini 4 Argon (High):
---
欢迎来到 **Step 9：健康检查（`/healthz` 与 `/readyz`）**！

在动手写代码之前，我们先结合 [stage0-todolist.md](file:///Users/amadeus/work-space/learning/go-seckill/guide/stage0-todolist.md#L156-L164) 中的思考题，理解为什么要把健康检查拆成两个接口，以及底层探活的原理。

---

### 一、 核心概念科普：Liveness (`/healthz`) vs Readiness (`/readyz`)

在现代云原生架构（如 Kubernetes、负载均衡网关）中，容器的健康状态被严格区分为两类探针（Probe）：

| 维度 | 存活检查 Liveness (`GET /healthz`) | 就绪检查 Readiness (`GET /readyz`) |
| :--- | :--- | :--- |
| **核心问题** | **“进程还活着吗？有没有死锁/卡死？”** | **“当前实例准备好接收用户流量了吗？”** |
| **是否检查下游依赖** | **绝不检查**（只看 HTTP 协程能否正常响应） | **必须检查**（带超时 Ping MySQL、Redis 等） |
| **失败时的动作** | 编排系统（如 K8s）会直接 **Kill 并重启容器** | 负载均衡器将实例 **从流量转发列表中摘除**（返回 503），**不重启容器**；等依赖恢复返回 200 后再重新挂回流量 |

> **💡 思考题解答：如果 `/healthz` 也去检查 MySQL 和 Redis，在 K8s 里会发生什么？**
> 会引发灾难性的**级联重启风暴（雪崩）**：
> 1. 假设 Redis 因为网络抖动或主从切换短暂不可用 3 秒，此时 API 进程本身是完全健康的。
> 2. 如果 `/healthz` 检查了 Redis 并返回 503，K8s 会误以为所有 API 进程死锁了，于是**同时杀掉并重启所有 API Pod**。
> 3. 这不仅会导致正在处理中的其他正常请求全部中断，而且所有 Pod 重启后在同一瞬间向 MySQL / Redis 发起初始化连接（我们在 [mysql.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/infra/mysql/mysql.go#L18-L48) 写了启动 `Ping`），形成**连接风暴**，把刚刚恢复的数据库再次打挂。

---

### 二、 底层原理：GORM 与 Redis 的探活到底发了什么？

你在项目中使用了 **GORM + Go-Redis**，在 `/readyz` 中检查它们时，底层发生了什么？

1. **GORM / MySQL 探活原理**：
   - GORM 本身并没有自己实现连接池，而是封装了 Go 标准库的 `*sql.DB`（通过 `sqlDB, err := db.DB()` 获取）。
   - 当调用 `sqlDB.PingContext(ctx)` 时，`database/sql` 会从连接池取出一个空闲连接（若无则新建连接），调用底层驱动 `go-sql-driver/mysql`。
   - **底层协议（非 SQL）**：驱动默认并不是发送一条笨重的文本 SQL，而是直接通过 TCP 发送一个 1 字节的 MySQL 原生协议包 **`COM_PING` (`0x0e`)**，MySQL Server 收到后直接回复 `OK_Packet`（仅当连接处于特殊状态或驱动回退时才执行轻量探活）。这比执行普通查询的开销更小。
2. **Redis 探活原理**：
   - 调用 `rdb.Ping(ctx).Err()` 时，会通过 RESP 协议向 Redis 发送 `PING` 命令，期望收到 `PONG` 响应。
3. **为什么要加独立超时（`context.WithTimeout`）？**
   - 当 Redis 或 MySQL 容器卡死或网络丢包时，如果不给每次探活加上短超时（例如 `1s ~ 2s`），`/readyz` 请求就会一直阻塞到连接池超时，导致健康检查探针自身超时堆积。

---

### 三、 实现步骤指导

我看了一下你现有的 [server.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/server.go) 和 [response.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/response.go)，写得非常规范！为了保持 `internal/server` 包的整洁，同时为下一步 **Step 10（优雅停机时让 `/readyz` 返回 503）** 和 **Step 12（不依赖 Docker 的单元测试）** 铺路，建议按以下方式实现：

#### 1. 设计亮点：用函数抽象解耦依赖 + 预留停机状态

不要让 `internal/server` 直接强绑定 `*gorm.DB` 和 `*redis.Client`，而是定义一个通用的探活函数类型：
```go
// HealthChecker 定义单个依赖组件的健康检查函数签名
type HealthChecker func(ctx context.Context) error
```
这样有两个巨大好处：
1. **架构解耦**：`server` 包不需要 import `gorm` 和 `go-redis`。
2. **极易单测（Step 12）**：写单元测试测 `/readyz` 503 时，只需传入一个返回 `errors.New("redis down")` 的匿名函数即可，完全不需要启动 Docker 容器！

#### 2. 在 `internal/infra/mysql/mysql.go` 补充 `Ping` 辅助函数（可选但推荐）

打开 [mysql.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/infra/mysql/mysql.go)，在 `Close` 函数旁边加一个 `Ping` 函数，屏蔽 `db.DB()` 的细节：

```go
// Ping 检查 MySQL 连通性，供 /readyz 就绪探针使用
func Ping(ctx context.Context, db *gorm.DB) error {
	if db == nil {
		return fmt.Errorf("mysql db is nil")
	}
	sqlDB, err := db.DB()
	if err != nil {
		return fmt.Errorf("get underlying sql.DB: %w", err)
	}
	return sqlDB.PingContext(ctx)
}
```

#### 3. 新建 `implement/internal/server/health.go`

请注意观察你现有的 [response.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/response.go#L28-L47)：`Fail(c, err)` 返回错误时是没有 `Data` 字段的；但 Step 9 要求 **`/readyz` 在返回 503 时也要标明每个依赖的状态**（如 `"mysql": "ok", "redis": "unavailable"`）。

因此，我们可以在 `internal/server/health.go` 中实现健康检查逻辑（顺便用 `sync/atomic.Bool` 预留好 Step 10 需要的停机状态位）：

```go
package server

import (
	"context"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
)

const readinessCheckTimeout = 2 * time.Second

// HealthChecker 定义依赖组件的探活函数
type HealthChecker func(ctx context.Context) error

// RegisterChecker 注册一个需要在 /readyz 中检查的下游依赖（如 mysql、redis）
func (s *Server) RegisterChecker(name string, checker HealthChecker) {
	if s.checkers == nil {
		s.checkers = make(map[string]HealthChecker)
	}
	s.checkers[name] = checker
}

// MarkShuttingDown 标记服务进入停机阶段，使 /readyz 立即返回 503（配合 Step 10 优雅停机）
func (s *Server) MarkShuttingDown() {
	s.shuttingDown.Store(true)
}

// handleHealthz 存活检查（Liveness）：只要进程活着就返回 200，绝不检查外部依赖
func (s *Server) handleHealthz(c *gin.Context) {
	Success(c, gin.H{
		"status": "ok",
	})
}

// handleReadyz 就绪检查（Readiness）：检查是否正在停机，并带超时检查所有注册的依赖
func (s *Server) handleReadyz(c *gin.Context) {
	// 1. 若正在停机，直接拒绝新流量
	if s.shuttingDown.Load() {
		c.JSON(ErrServiceUnavailable.HTTPStatus, Response{
			Code: ErrServiceUnavailable.Code,
			Msg:  "server is shutting down",
			Data: gin.H{"status": "shutting_down"},
		})
		return
	}

	// 2. 带超时检查各个下游依赖
	ctx, cancel := context.WithTimeout(c.Request.Context(), readinessCheckTimeout)
	defer cancel()

	checks := make(map[string]string, len(s.checkers))
	allReady := true

	for name, checker := range s.checkers {
		if err := checker(ctx); err != nil {
			allReady = false
			checks[name] = "unavailable: " + err.Error()
		} else {
			checks[name] = "ok"
		}
	}

	if !allReady {
		c.JSON(http.StatusServiceUnavailable, Response{
			Code: ErrServiceUnavailable.Code,
			Msg:  ErrServiceUnavailable.Msg,
			Data: gin.H{
				"status": "not_ready",
				"checks": checks,
			},
		})
		return
	}

	Success(c, gin.H{
		"status": "ready",
		"checks": checks,
	})
}
```

#### 4. 修改 `implement/internal/server/server.go`

在 [server.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/server.go) 中：
1. 引入 `"sync/atomic"`，在 [`Server`](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/server.go#L15-L19) 结构体中加上两个字段：
   ```go
   type Server struct {
       httpServer   *http.Server
       engine       *gin.Engine
       log          *slog.Logger
       checkers     map[string]HealthChecker
       shuttingDown atomic.Bool
   }
   ```
2. 在 [`New`](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/server.go#L22-L66) 函数中构造好 `srv := &Server{...}` 后，直接注册路由：
   ```go
   srv.engine.GET("/healthz", srv.handleHealthz)
   srv.engine.GET("/readyz", srv.handleReadyz)
   ```
   *(注：在 [`Shutdown`](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/server.go#L83-L86) 方法第一行也可以顺手加上 `s.MarkShuttingDown()`)*。

#### 5. 在 `implement/cmd/api/main.go` 中注入依赖检查

打开 [main.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/cmd/api/main.go#L73-L76)，在 `ser := server.New(cfg.Server, log)` 之后、`ser.Start()` 之前，把 MySQL 和 Redis 注册进去：

```go
	ser := server.New(cfg.Server, log)
	ser.RegisterChecker("mysql", func(ctx context.Context) error {
		return mysql.Ping(ctx, db)
	})
	ser.RegisterChecker("redis", func(ctx context.Context) error {
		return rdb.Ping(ctx).Err()
	})
```

---

### 四、 完成标准与自测验证

写完代码后，按以下步骤验证是否达到 [stage0-todolist.md](file:///Users/amadeus/work-space/learning/go-seckill/guide/stage0-todolist.md#L161) 的完成标准：

1. **启动依赖与服务**：
   - 确保 MySQL 和 Redis 容器正在运行，启动 API 服务。
2. **正常状态验证**：
   - `curl -i http://localhost:8080/healthz` -> 应返回 `HTTP/1.1 200 OK`
   - `curl -i http://localhost:8080/readyz` -> 应返回 `HTTP/1.1 200 OK`，且 JSON 中 `mysql` 和 `redis` 均为 `"ok"`。
3. **故障注入验证（停掉 Redis）**：
   - 在另一个终端执行：`docker compose -f implement/deploy/docker-compose.yml stop redis`
   - 再次访问 `curl -i http://localhost:8080/readyz` -> 应返回 **`HTTP/1.1 503 Service Unavailable`**，且 `checks.redis` 标明不可用，`checks.mysql` 仍为 `"ok"`。
   - 再次访问 `curl -i http://localhost:8080/healthz` -> **仍然返回 `HTTP/1.1 200 OK`**。
4. **恢复验证**：
   - 执行：`docker compose -f implement/deploy/docker-compose.yml start redis`
   - 再次访问 `curl -i http://localhost:8080/readyz` -> 自动恢复为 `HTTP/1.1 200 OK`！

完成后告诉我，我可以帮你检查代码并更新 TodoList，然后我们进入 **Step 10：优雅停机**！
---
# User:
---
@[Quote] 我们要在server.go 中的new函数中注册路由吗

> Quote: 在 New 函数中构造好 srv := &Server{...} 后，直接注册路由：
---
# Gemini 4 Argon (High):
---
非常好的架构问题！你敏锐地察觉到了：**如果以后业务接口越来越多，全往 `New()` 里塞岂不是会变成一坨面条代码？**

在 Go Web 工程实践中，我们通常会把路由分为两类来对待：

### 1. 系统内置路由（基础设施级） vs 业务路由

| 路由类型 | 包含内容 | 归属与生命周期 | 在哪里注册？ |
| :--- | :--- | :--- | :--- |
| **系统内置路由** | `404 NoRoute`、`405 NoMethod`、`/healthz`、`/readyz`（以及阶段 7 的 `/metrics`） | 属于 `server.Server` 底座自身的能力，**不依赖任何业务 Handler**。只要是一个 HTTP Server 实例（包括在单元测试里 `server.New` 出来的实例），天生就应该自带这些端点。 | **在 `server.New()` 内部自动完成** |
| **业务路由** | `/api/v1/users`、`/api/v1/seckill` 等（阶段 1 开始写） | 依赖具体的业务 `Handler`（如 `UserHandler`、`OrderHandler`）。 | **不放进 `New()`**，而是到了阶段 1 单独抽一个 `router.go`（例如提供 `s.RegisterRoutes(...)` 方法）或在组装时挂载 |

---

### 2. 在 `server.go` 中怎么写最干净？

你看现在的 [server.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/server.go#L43-L66)，其实第 44~51 行的 `NoRoute` 和 `NoMethod` 已经是在 `New()` 里注册了。

为了让 [`New`](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/server.go#L22-L66) 保持清爽，你可以先构造出 `s := &Server{...}`，然后直接在返回前绑上 `/healthz` 和 `/readyz`（因为 `handleHealthz` 和 `handleReadyz` 是 `*Server` 的方法，需要先有 `s` 实例）：

```go
	httpSrv := &http.Server{
		Addr:         cfg.Addr(),
		Handler:      engine,
		ReadTimeout:  cfg.ReadTimeout,
		WriteTimeout: cfg.WriteTimeout,
		IdleTimeout:  cfg.IdleTimeout,
	}

	s := &Server{
		httpServer: httpSrv,
		engine:     engine,
		log:        log,
		checkers:   make(map[string]HealthChecker),
	}

	// 注册系统内置探针路由
	s.engine.GET("/healthz", s.handleHealthz)
	s.engine.GET("/readyz", s.handleReadyz)

	return s
```

**这样做还有一个额外的好处**：
等到 **Step 12 写测试** 时，你在测试代码里只需要写一行 `srv := server.New(cfg, log)`，拿到的 `srv.Engine()` 就已经自带了 `/healthz` 和 `/readyz` 路由，不需要在测试里再手动去绑一遍路由。
---
# User:
---
已完成，请检查我的实现
---
# Gemini 4 Argon (High):
---
我检查了你的代码并运行了编译与测试（`go build ./... && go test ./...` 全部通过），**Step 9 完成得非常漂亮！** 我已经帮你在 [stage0-todolist.md](file:///Users/amadeus/work-space/learning/go-seckill/guide/stage0-todolist.md#L156-L160) 中勾选了 Step 9。

### 代码 Review 亮点

1. **职责边界清晰**：
   - [mysql.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/infra/mysql/mysql.go#L63-L72) 封装了 `Ping(ctx, db)`，把 GORM 获取底层 `*sql.DB` 的细节收敛在 `infra/mysql` 包内。
   - [health.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/health.go#L14-L22) 通过 `HealthChecker` 函数类型完成了依赖倒置，`internal/server` 包保持了对 GORM 和 go-redis 的零耦合。
   - [main.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/cmd/api/main.go#L73-L79) 用短短 7 行闭包就把 MySQL 和 Redis 探针装配进去了，可读性极佳。
2. **并发安全与超时控制**：
   - 使用了 `sync/atomic.Bool` 管理 `shuttingDown` 状态，避免了并发读写竞态（将在 Step 12 的 `go test -race` 中发挥作用）。
   - `/readyz` 使用了独立的 2 秒 `context.WithTimeout`，防止慢依赖拖垮探针协程。

### 两个可选的工程细节小建议（锦上添花）

1. **防御性判空**：在 [health.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/health.go#L17-L22) 的 `RegisterChecker` 中，如果调用方不小心传了 `nil` 的 `checker`，后续请求 `/readyz` 调用 `checker(ctx)` 时会触发 nil 函数调用 panic。可以加一行防御：
   ```go
   if checker == nil {
       return
   }
   ```
2. **探活失败日志**：在 [health.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/health.go#L56-L59) 中，当 `err := checker(ctx); err != nil` 时，除了写进响应 JSON，也可以顺手打一条 `s.log.WarnContext(ctx, "readiness check failed", slog.String("component", name), slog.Any("error", err))`，这样服务端日志里也能直接查到哪一刻哪个依赖探活失败了。

---

随时可以进行 `git commit` 提交 Step 9 的成果。准备好进入 **Step 10：优雅停机（Graceful Shutdown）** 以及 **Step 11：组装入口 `cmd/api/main.go`** 了吗？
---
# User:
---
遇到问题
(附图: /Users/amadeus/.gemini/antigravity/brain/75af77c7-8f9a-4b92-9d0a-b8ee06567215/.user_uploaded/media_1794461786167_1c6cdce4.jpg)
---
# Gemini 4 Argon (High):
---
这是因为变量作用域（Scope）的问题：`name` 和 `err` 是在上面的 `for name, checker := range s.checkers` 循环和 `if err := checker(ctx); err != nil` 块内部定义的局部变量，出了 `for` 循环大括号后它们就不存在了，所以编译器报错 `undefined: name`。

### 解决方法

把这行日志放到 `for` 循环里面的 `if err := checker(ctx); err != nil` 分支内（同时记得在文件顶部 `import` 中加上 `"log/slog"`）：

```go
	for name, checker := range s.checkers {
		if err := checker(ctx); err != nil {
			allReady = false
			checks[name] = "unavailable: " + err.Error()
			s.log.WarnContext(ctx, "readiness check failed",
				slog.String("component", name),
				slog.Any("error", err),
			)
		} else {
			checks[name] = "ok"
		}
	}
```

放在循环内还有一个好处：如果 MySQL 和 Redis 同时挂了，循环内可以为每一个挂掉的组件分别打出对应的错误日志。
---
