# User:
---
我正在第0阶段，指导我完成Step 12
---
# Gemini 4 Argon (High):
---
你前面的 `config_test.go` 和 `logger_test.go` 已经写得很规范了！在 **Step 12（测试基础设施）** 中，我们要把 **HTTP 服务层（`internal/server`）的单元测试**、**连接真实容器的集成测试** 以及 **`Makefile` 自动化测试命令** 补齐。

---

### 一、 核心原理与顶层概念科普

在动手写代码前，先理清本步涉及的三个关键工程概念：

#### 1. 单元测试（Unit Test）vs 集成测试（Integration Test）
* **单元测试**：只测试内存中的逻辑，**绝对不依赖外部网络、磁盘数据库或 Docker 容器**。它必须能在任何没有装 Docker 的环境（比如轻量 CI 节点）毫秒级跑完。
  * *思考：`/readyz` 需要检查 MySQL 和 Redis，怎么在不启动 Docker 的情况下单测它？*
  * **答案就在你 Step 9 的设计里**：你在 [health.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/health.go#L14-L18) 中抽象出了 `type HealthChecker func(ctx context.Context) error`。单测时我们只需要注入一个返回 `nil` 或 `errors.New("connection refused")` 的假函数（Stub/Mock），就能 100% 覆盖 `/readyz` 的所有分支！这就是**依赖注入（DI）面向抽象编程**带来的可测试性红利。
* **集成测试**：验证你的代码与真实外部组件（MySQL `seckill_test` 库、Redis、SQL 迁移文件）组合在一起时能否正常工作。

#### 2. `net/http/httptest` 为什么不需要监听端口？
在 Go 标准库中，任何 HTTP 框架（包括 Gin 的 `*gin.Engine`）本质上都实现了标准库的 `http.Handler` 接口：
```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```
平时调用 `httpServer.ListenAndServe()` 时，是操作系统的 TCP 协议栈收到网卡字节流后，解析出 `*http.Request`，再调用 `engine.ServeHTTP(w, req)`。
而在单元测试中，`httptest.NewRequest` 直接在内存里构造 `*http.Request`，`httptest.NewRecorder()` 构造一个基于内存 `bytes.Buffer` 的假 `ResponseWriter`，然后直接调用：
```go
srv.Engine().ServeHTTP(rec, req)
```
**完全绕过了 TCP 握手、端口监听和内核态切换**，既不会出现端口冲突，速度也比真实网络请求快几个数量级。

#### 3. Go 编译标签（Build Tags）与 `-race` 竞态检测器
* **`//go:build integration`**：放在 Go 源码文件的**第一行**（注意下面必须紧跟一个**空行**，否则会被当成普通注释）。平时运行 `go test ./...` 时，Go 编译器会直接忽略带有该标签的文件；只有显式传入 `go test -tags=integration ./...` 时才会编译并运行它。
* **`-race`（Data Race Detector）**：当多个 goroutine 同时访问同一块内存，且至少有一个是写操作，又没有同步原语（如 `sync.Mutex` 或 `atomic`）保护时，就会发生数据竞态（比如你在 [server.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/server.go#L21) 中用的 `shuttingDown atomic.Bool`，如果换成普通 `bool` 并在并发请求 `/readyz` 时调用 `MarkShuttingDown()`，`-race` 就会立刻报警抓出 Bug）。

---

### 二、 动手实现 Step 12

#### 第 1 步：编写 HTTP 服务单元测试

请新建 `implement/internal/server/server_test.go`，覆盖 **错误码映射、统一响应脱敏、中间件（RequestID / AccessLog / Recovery）、404/405 以及健康检查**：

```go
package server

import (
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"

	"github.com/gin-gonic/gin"

	"go-seckill/internal/config"
	"go-seckill/internal/logger"
)

func newTestServer(buf *bytes.Buffer) *Server {
	log := logger.NewWithOptions(logger.Options{
		Level:   "info",
		Format:  "json",
		Service: "api-test",
		Version: "v0.1.0",
		Output:  buf,
	})
	cfg := config.ServerConfig{
		Host:            "127.0.0.1",
		Port:            8080,
		ReadTimeout:     2 * time.Second,
		WriteTimeout:    2 * time.Second,
		IdleTimeout:     10 * time.Second,
		ShutdownTimeout: 5 * time.Second,
	}
	return New(cfg, log)
}

// 1. 验证统一响应 Success 与 Fail 的错误码映射（含 errors.As 包装识别与未知错误脱敏兜底）
func TestResponseAndErrorCodeMapping(t *testing.T) {
	var logBuf bytes.Buffer
	srv := newTestServer(&logBuf)

	srv.Engine().GET("/test/ok", func(c *gin.Context) {
		Success(c, gin.H{"item_id": 42})
	})
	srv.Engine().GET("/test/custom-err", func(c *gin.Context) {
		// 测试 WithMsg 派生 + fmt.Errorf("%w") 包装后 errors.As 仍能识别
		wrappedErr := fmt.Errorf("service layer failed: %w", ErrInvalidParam.WithMsg("activity_id must be > 0"))
		Fail(c, wrappedErr)
	})
	srv.Engine().GET("/test/unknown-err", func(c *gin.Context) {
		// 模拟底层数据库报错，验证不会把敏感内部错误暴露给客户端
		Fail(c, errors.New("Error 1062: Duplicate entry 'xxx' for key 'uk_request_id'"))
	})

	t.Run("success response", func(t *testing.T) {
		req := httptest.NewRequest(http.MethodGet, "/test/ok", nil)
		rec := httptest.NewRecorder()
		srv.Engine().ServeHTTP(rec, req)

		if rec.Code != http.StatusOK {
			t.Fatalf("expected status 200, got %d", rec.Code)
		}
		var resp Response
		if err := json.Unmarshal(rec.Body.Bytes(), &resp); err != nil {
			t.Fatalf("unmarshal response: %v", err)
		}
		if resp.Code != 0 || resp.Msg != "ok" {
			t.Errorf("unexpected response: %+v", resp)
		}
	})

	t.Run("wrapped ErrorCode mapping", func(t *testing.T) {
		req := httptest.NewRequest(http.MethodGet, "/test/custom-err", nil)
		rec := httptest.NewRecorder()
		srv.Engine().ServeHTTP(rec, req)

		if rec.Code != http.StatusBadRequest {
			t.Fatalf("expected status 400, got %d", rec.Code)
		}
		var resp Response
		if err := json.Unmarshal(rec.Body.Bytes(), &resp); err != nil {
			t.Fatalf("unmarshal response: %v", err)
		}
		if resp.Code != ErrInvalidParam.Code || resp.Msg != "activity_id must be > 0" {
			t.Errorf("expected code=%d msg='activity_id must be > 0', got %+v", ErrInvalidParam.Code, resp)
		}
	})

	t.Run("unknown error masked as 500", func(t *testing.T) {
		req := httptest.NewRequest(http.MethodGet, "/test/unknown-err", nil)
		rec := httptest.NewRecorder()
		srv.Engine().ServeHTTP(rec, req)

		if rec.Code != http.StatusInternalServerError {
			t.Fatalf("expected status 500, got %d", rec.Code)
		}
		if strings.Contains(rec.Body.String(), "Duplicate entry") {
			t.Fatalf("SECURITY LEAK: internal sql error leaked in HTTP response: %s", rec.Body.String())
		}
		var resp Response
		if err := json.Unmarshal(rec.Body.Bytes(), &resp); err != nil {
			t.Fatalf("unmarshal response: %v", err)
		}
		if resp.Code != ErrInternal.Code || resp.Msg != ErrInternal.Msg {
			t.Errorf("expected internal error fallback, got %+v", resp)
		}
	})
}

// 2. 验证 404 NoRoute 与 405 NoMethod 统一响应
func TestNoRouteAndNoMethod(t *testing.T) {
	var logBuf bytes.Buffer
	srv := newTestServer(&logBuf)

	// 请求不存在的路由 -> 404
	req404 := httptest.NewRequest(http.MethodGet, "/non-existent-path", nil)
	rec404 := httptest.NewRecorder()
	srv.Engine().ServeHTTP(rec404, req404)

	if rec404.Code != http.StatusNotFound {
		t.Fatalf("expected 404, got %d", rec404.Code)
	}

	// 用 POST 请求只注册了 GET 的 /healthz -> 405
	req405 := httptest.NewRequest(http.MethodPost, "/healthz", nil)
	rec405 := httptest.NewRecorder()
	srv.Engine().ServeHTTP(rec405, req405)

	if rec405.Code != http.StatusMethodNotAllowed {
		t.Fatalf("expected 405, got %d", rec405.Code)
	}
}

// 3. 验证中间件：RequestID 生成/透传、AccessLog 记录、Recovery 捕获 panic
func TestMiddlewares(t *testing.T) {
	t.Run("RequestID generated when missing and propagated to log", func(t *testing.T) {
		var logBuf bytes.Buffer
		srv := newTestServer(&logBuf)

		req := httptest.NewRequest(http.MethodGet, "/healthz", nil)
		rec := httptest.NewRecorder()
		srv.Engine().ServeHTTP(rec, req)

		reqID := rec.Header().Get(HeaderXRequestID)
		if len(reqID) != 32 {
			t.Fatalf("expected 32-char hex X-Request-ID, got %q", reqID)
		}
		if !strings.Contains(logBuf.String(), reqID) {
			t.Errorf("expected access log to contain request_id %q, got log: %s", reqID, logBuf.String())
		}
	})

	t.Run("RequestID preserved when provided by client", func(t *testing.T) {
		var logBuf bytes.Buffer
		srv := newTestServer(&logBuf)

		req := httptest.NewRequest(http.MethodGet, "/healthz", nil)
		req.Header.Set(HeaderXRequestID, "client-req-id-9527")
		rec := httptest.NewRecorder()
		srv.Engine().ServeHTTP(rec, req)

		if got := rec.Header().Get(HeaderXRequestID); got != "client-req-id-9527" {
			t.Errorf("expected X-Request-ID='client-req-id-9527', got %q", got)
		}
		if !strings.Contains(logBuf.String(), "client-req-id-9527") {
			t.Errorf("expected log to contain custom request_id, got: %s", logBuf.String())
		}
	})

	t.Run("Recovery catches panic and returns unified 500", func(t *testing.T) {
		var logBuf bytes.Buffer
		srv := newTestServer(&logBuf)

		srv.Engine().GET("/test/panic", func(c *gin.Context) {
			panic("unexpected nil pointer in handler")
		})

		req := httptest.NewRequest(http.MethodGet, "/test/panic", nil)
		rec := httptest.NewRecorder()
		srv.Engine().ServeHTTP(rec, req)

		if rec.Code != http.StatusInternalServerError {
			t.Fatalf("expected status 500 on panic, got %d", rec.Code)
		}
		var resp Response
		if err := json.Unmarshal(rec.Body.Bytes(), &resp); err != nil {
			t.Fatalf("unmarshal panic response: %v", err)
		}
		if resp.Code != ErrInternal.Code {
			t.Errorf("expected code=%d, got %d", ErrInternal.Code, resp.Code)
		}
		if !strings.Contains(logBuf.String(), "http panic recovered") {
			t.Errorf("expected panic log, got: %s", logBuf.String())
		}
	})
}

// 4. 验证 /healthz 与 /readyz 探针状态切换（无 Docker 纯内存测试）
func TestHealthAndReadyz(t *testing.T) {
	var logBuf bytes.Buffer
	srv := newTestServer(&logBuf)

	redisErr := errors.New("dial tcp: connection refused")
	redisHealthy := true

	srv.RegisterChecker("mysql", func(ctx context.Context) error {
		return nil
	})
	srv.RegisterChecker("redis", func(ctx context.Context) error {
		if !redisHealthy {
			return redisErr
		}
		return nil
	})

	// (a) 所有依赖正常 -> /readyz 返回 200
	req := httptest.NewRequest(http.MethodGet, "/readyz", nil)
	rec := httptest.NewRecorder()
	srv.Engine().ServeHTTP(rec, req)
	if rec.Code != http.StatusOK {
		t.Fatalf("expected /readyz 200 when all healthy, got %d: %s", rec.Code, rec.Body.String())
	}

	// (b) 模拟 Redis 故障 -> /readyz 返回 503，但 /healthz 依然返回 200
	redisHealthy = false
	recReady := httptest.NewRecorder()
	srv.Engine().ServeHTTP(recReady, httptest.NewRequest(http.MethodGet, "/readyz", nil))
	if recReady.Code != http.StatusServiceUnavailable {
		t.Fatalf("expected /readyz 503 when redis down, got %d", recReady.Code)
	}
	if !strings.Contains(recReady.Body.String(), "connection refused") {
		t.Errorf("expected /readyz body to mention redis error, got: %s", recReady.Body.String())
	}

	recLive := httptest.NewRecorder()
	srv.Engine().ServeHTTP(recLive, httptest.NewRequest(http.MethodGet, "/healthz", nil))
	if recLive.Code != http.StatusOK {
		t.Fatalf("expected /healthz 200 even when redis down, got %d", recLive.Code)
	}

	// (c) 恢复 Redis，但标记服务进入停机状态 -> /readyz 立即返回 503 shutting_down
	redisHealthy = true
	srv.MarkShuttingDown()
	recShut := httptest.NewRecorder()
	srv.Engine().ServeHTTP(recShut, httptest.NewRequest(http.MethodGet, "/readyz", nil))
	if recShut.Code != http.StatusServiceUnavailable {
		t.Fatalf("expected /readyz 503 when shutting down, got %d", recShut.Code)
	}
	if !strings.Contains(recShut.Body.String(), "shutting_down") {
		t.Errorf("expected shutting_down status, got: %s", recShut.Body.String())
	}
}
```

---

#### 第 2 步：编写集成测试（含 GORM 底层 SQL 解析）

请新建 `implement/internal/server/integration_test.go`。
注意两点：
1. **文件第一行必须是 `//go:build integration`**，确保 `make test` 默认不会运行它。
2. 我们在集成测试中强制指定连接 **`seckill_test`** 测试库和 Redis **`DB: 1`**，这样即使宿主机环境变量设置了 `MYSQL_DATABASE=seckill`，测试也绝不会污染你的业务主库。

```go
//go:build integration

package server_test

import (
	"bytes"
	"context"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"
	"time"

	"go-seckill/internal/config"
	"go-seckill/internal/infra/mysql"
	"go-seckill/internal/infra/redis"
	"go-seckill/internal/logger"
	"go-seckill/internal/server"
)

// pingTestRecord 对应 migrations/000001_init_ping_test.up.sql 中的 ping_test 表
type pingTestRecord struct {
	ID        uint64    `gorm:"primaryKey;autoIncrement"`
	Message   string    `gorm:"column:message"`
	CreatedAt time.Time `gorm:"column:created_at;autoCreateTime"`
}

func (pingTestRecord) TableName() string {
	return "ping_test"
}

func TestIntegration_ReadyzAndMigration(t *testing.T) {
	// 加载环境变量中的端口与密码配置（不依赖相对路径的 config.yaml）
	cfg, err := config.Load("")
	if err != nil {
		t.Fatalf("load config failed (请确保设置了 MYSQL_PASSWORD 和 REDIS_PASSWORD): %v", err)
	}

	// 强制隔离到测试专用的 seckill_test 库和 Redis DB 1，防止污染业务数据
	cfg.MySQL.Database = "seckill_test"
	cfg.Redis.DB = 1

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	// 1. 初始化真实 MySQL 连接
	db, err := mysql.New(ctx, cfg.MySQL)
	if err != nil {
		t.Fatalf("connect to mysql seckill_test failed: %v", err)
	}
	defer func() {
		_ = mysql.Close(db)
	}()

	// 2. 初始化真实 Redis 连接
	rdb, err := redis.New(ctx, cfg.Redis)
	if err != nil {
		t.Fatalf("connect to redis failed: %v", err)
	}
	defer func() {
		_ = rdb.Close()
	}()

	// 3. 验证 seckill_test 库已执行迁移（ping_test 表可正常插入与查询）
	record := pingTestRecord{Message: "integration_test_ping"}
	if err := db.WithContext(ctx).Create(&record).Error; err != nil {
		t.Fatalf("insert into ping_test failed (请确认已对 seckill_test 执行迁移): %v", err)
	}
	t.Cleanup(func() {
		_ = db.WithContext(context.Background()).Delete(& pingTestRecord{}, record.ID).Error
	})

	var got pingTestRecord
	if err := db.WithContext(ctx).First(&got, record.ID).Error; err != nil {
		t.Fatalf("query ping_test by id=%d failed: %v", record.ID, err)
	}
	if got.Message != "integration_test_ping" {
		t.Errorf("expected message='integration_test_ping', got %q", got.Message)
	}

	// 4. 组装 Server 并验证真实依赖下的 /readyz 返回 200
	var logBuf bytes.Buffer
	log := logger.NewWithOptions(logger.Options{
		Level:  "info",
		Format: "json",
		Output: &logBuf,
	})
	srv := server.New(cfg.Server, log)
	srv.RegisterChecker("mysql", func(ctx context.Context) error {
		return mysql.Ping(ctx, db)
	})
	srv.RegisterChecker("redis", func(ctx context.Context) error {
		return rdb.Ping(ctx).Err()
	})

	req := httptest.NewRequest(http.MethodGet, "/readyz", nil)
	rec := httptest.NewRecorder()
	srv.Engine().ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Fatalf("expected /readyz status 200, got %d, body: %s", rec.Code, rec.Body.String())
	}

	var resp struct {
		Code int `json:"code"`
		Data struct {
			Status string            `json:"status"`
			Checks map[string]string `json:"checks"`
		} `json:"data"`
	}
	if err := json.Unmarshal(rec.Body.Bytes(), &resp); err != nil {
		t.Fatalf("unmarshal /readyz body: %v", err)
	}
	if resp.Data.Status != "ready" || resp.Data.Checks["mysql"] != "ok" || resp.Data.Checks["redis"] != "ok" {
		t.Errorf("unexpected /readyz data: %+v", resp.Data)
	}
}
```

##### 💡 GORM 生成的底层 SQL 解析（必看）
上面集成测试中我们用到了三次 GORM 操作，它们在底层分别通过预编译语句（Prepared Statement `?` 占位符）生成并执行了如下 SQL：

1. **`db.WithContext(ctx).Create(&record)`**
   * **生成的 SQL**：
     ```sql
     INSERT INTO `ping_test` (`message`, `created_at`) VALUES (?, ?)
     -- 参数: "integration_test_ping", "2026-11-12 20:xx:xx.xxx"
     ```
   * **工作原理**：因为 `ID` 字段标记了 `gorm:"primaryKey;autoIncrement"` 且此时为零值 `0`，GORM 会自动在 `INSERT` 列中省略 `id`，交给 MySQL 自增；语句执行完毕后，GORM 会从 MySQL 驱动返回的 `sql.Result.LastInsertId()` 中读取生成的自增主键，**自动回填到 `record.ID` 中**。

2. **`db.WithContext(ctx).First(&got, record.ID)`**
   * **生成的 SQL**：
     ```sql
     SELECT * FROM `ping_test` WHERE `ping_test`.`id` = ? ORDER BY `ping_test`.`id` LIMIT 1
     -- 参数: record.ID
     ```
   * **工作原理**：注意 `First()` 会默认追加 `ORDER BY 主键 ASC LIMIT 1`；当没有查到记录时，GORM 会返回 `gorm.ErrRecordNotFound`。
   * *小贴士*：以后在阶段 1~2 按唯一主键或唯一索引点查时，如果不需要按主键排序，也可以用 `Take(&got, record.ID)`，它生成的 SQL 是 `SELECT * FROM ping_test WHERE id = ? LIMIT 1`（少了 `ORDER BY`，在非主键索引查询时能避免一次额外的排序开销）。

3. **`db.WithContext(context.Background()).Delete(&pingTestRecord{}, record.ID)`**
   * **生成的 SQL**：
     ```sql
     DELETE FROM `ping_test` WHERE `ping_test`.`id` = ?
     -- 参数: record.ID
     ```
   * **工作原理**：因为 `pingTestRecord` 结构体中**没有**定义 `gorm.DeletedAt` 字段，所以 GORM 执行的是**物理删除（`DELETE`）**而非软删除（`UPDATE ... SET deleted_at = ?`）。

---

#### 第 3 步：在 `Makefile` 中增加 `test` 与 `test-integration`

打开你的 [Makefile](file:///Users/amadeus/work-space/learning/go-seckill/implement/Makefile)，在里面加入测试库迁移变量与两个测试目标（建议放在 `migrate-version` 下方）：

```makefile
# 测试数据库名称与迁移连接串
MYSQL_TEST_DATABASE ?= seckill_test
MIGRATE_TEST_DB_URL := mysql://$(MYSQL_USER):$(MYSQL_PASSWORD)@tcp($(MYSQL_HOST):$(MYSQL_PORT))/$(MYSQL_TEST_DATABASE)?multiStatements=true

.PHONY: test test-integration

## test: 运行所有单元测试（开启竞态检测，不依赖 Docker）
test:
	go test -race -v ./...

## test-integration: 对 seckill_test 执行迁移并运行集成测试（需先 make docker-up）
test-integration:
	$(MIGRATE) -path $(MIGRATIONS_DIR) -database "$(MIGRATE_TEST_DB_URL)" up
	go test -race -v -tags=integration ./...
```

---

### 三、 验证与完成标准自检

完成上述修改后，请按顺序在 `implement/` 下执行以下两个实验来验证：

1. **验证 `make test` 不依赖 Docker**：
   * 先把容器停掉：`make docker-stop`
   * 运行单元测试：`make test`
   * **预期**：`internal/config`、`internal/logger`、`internal/server` 下的所有单元测试全部 `PASS`，且耗时在 1~2 秒内。
2. **验证 `make test-integration` 跑通真实链路**：
   * 启动容器：`make docker-start`（或 `make docker-up`）
   * 运行集成测试：`make test-integration`
   * **预期**：自动将 `000001_init_ping_test` 迁移应用到 `seckill_test` 库，随后 `TestIntegration_ReadyzAndMigration` 在 `-race` 下 `PASS`。

---

### 四、 课后思考题

1. 在 `TestIntegration_ReadyzAndMigration` 的 `t.Cleanup` 中，删除测试数据时为什么我们写了 `context.Background()` 而不是复用外层的 `ctx`？如果什么时候外层 `ctx` 因为超时取消了，复用它会发生什么？
2. 我们在 [server.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/server/server.go#L40-L44) 中注册中间件的顺序是 `RequestID() -> AccessLog(log) -> Recovery(log)`。结合 Gin 中间件 `c.Next()` 的**洋葱模型（栈式调用）**，当业务 Handler 发生 `panic` 时，为什么 `AccessLog` 依然能拿到 `500` 状态码并打印出包含 `request_id` 的访问日志？如果把 `Recovery(log)` 放到 `AccessLog(log)` 前面，会发生什么变化？

完成后告诉我结果，或者直接贴出你的测试输出，我们再一起进入 **Step 13（Makefile 与 golangci-lint 代码检查）**！
---
# User:
---
给出这两个问题的答案
---
# Gemini 4 Argon (High):
---
这两个问题分别对应了 Go 工程中最常踩的 **Context 生命周期坑** 和 **Gin 中间件洋葱模型执行机制**：

---

### 问题 1：为什么 `t.Cleanup` 里要用 `context.Background()` 而不是外层的 `ctx`？

#### 核心原因：资源清理/回滚操作不能依赖一个“可能已经死掉”的 Context

1. **执行时机与 `defer cancel()` 的冲突**：
   * 在测试函数开头我们写了：
     ```go
     ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
     defer cancel()
     ```
   * Go 中后注册的清理动作先执行（LIFO），虽然 `t.Cleanup` 在测试函数返回后才执行（此时 `defer cancel()` **已经执行过了**！），这就导致进入 `t.Cleanup` 时，外层的 `ctx` **必然已经处于 `context.Canceled` 状态**。
2. **即便不用 `defer cancel()`，如果测试中途超时了会怎样？**
   * 假设某次测试因为慢查询耗尽了 10 秒，外层 `ctx` 变成了 `context.DeadlineExceeded`。
   * 如果清理函数 `db.WithContext(ctx).Delete(...)` 复用了这个已经超时/取消的 `ctx`，GORM 底层的 `database/sql` 在发起网络请求前会先检查 `ctx.Err()`，发现非 `nil` 会**直接拒绝执行 SQL 并立刻返回 `context canceled` 或 `context deadline exceeded` 错误**。
   * **后果**：测试失败退出，但刚才插入 `seckill_test` 库的脏数据没有被删掉，残留在数据库里，导致下一次跑测试时可能因为唯一索引冲突而莫名其妙失败（Flaky Test）。

> **工程延伸**：这和你之前在 [cmd/api/main.go](file:///Users/amadeus/work-space/learning/go-seckill/implement/cmd/api/main.go#L110-L113) 优雅停机时写的逻辑是完全相通的——当收到 `SIGINT` 信号导致主 `ctx` 取消后，`ser.Shutdown(shutdownCtx)` 必须基于 `context.Background()` 新建一个超时 Context。**凡是“收尾、清理、回滚（比如阶段 3 MySQL 落库失败后回滚 Redis 预占库存）”的操作，都必须使用独立的 Context，绝不能复用已经取消的请求 Context。**

---

### 问题 2：Gin 洋葱模型下，`RequestID -> AccessLog -> Recovery` 的执行流与顺序差异

#### 1. 为什么当前顺序（`RequestID -> AccessLog -> Recovery`）在 Panic 时能完美打出 500 访问日志？

Gin 的 `c.Next()` 本质上是一个**函数调用栈（洋葱模型）**。当请求进来并发生 `panic` 时，调用栈的入栈（前半段）与出栈（后半段）顺序如下：

```
请求进入
  │
  ├─► 1. RequestID 前半段：生成 req_id，写入 Response Header 和 c.Request.Context()
  │     └─► 调用 c.Next()
  │           │
  │           ├─► 2. AccessLog 前半段：记录 start := time.Now()
  │           │     └─► 调用 c.Next()
  │           │           │
  │           │           ├─► 3. Recovery 前半段：注册 defer recover()
  │           │           │     └─► 调用 c.Next()
  │           │           │           │
  │           │           │           └─► 4. 业务 Handler：💥 panic("boom")！
  │           │           │
  │           │           ◄─ 5. 触发 Recovery 的 defer：
  │           │                 - recover() 截获 panic（阻止它继续向上炸穿调用栈）
  │           │                 - 打印 "http panic recovered" 错误日志 + 堆栈
  │           │                 - 调用 Fail(c, ErrInternal) 将 HTTP 状态码设为 500
  │           │                 - Recovery 函数正常返回！
  │           │
  │           ◄─ 6. 回到 AccessLog 的 c.Next() 之后：
  │                 - 因为 panic 已经被内部的 Recovery 吞掉并正常返回了，AccessLog 后半段正常执行！
  │                 - 读取 c.Writer.Status() 拿到刚才 Recovery 写入的 500
  │                 - 打印一条 status=500、耗时 latency、且带 request_id 的 http access 日志
  │
  ◄─ 7. 回到 RequestID 的 c.Next() 之后，请求结束
```

#### 2. 如果把顺序改成 `RequestID -> Recovery -> AccessLog` 会发生什么？

如果 `Recovery` 在外层、`AccessLog` 在内层：
1. 业务 Handler 发生 `panic("boom")`；
2. Panic 向上冒泡 unwind 调用栈，直接穿过 `AccessLog` 的 `c.Next()`（因为 `AccessLog` 没有 `defer recover()`，所以 **`AccessLog` 在 `c.Next()` 之后的代码被直接跳过，根本不会执行！**）；
3. 直到冒泡到外层的 `Recovery`，才被 `Recovery` 的 `defer` 捕获并返回 500。

**结果对比**：
* `AccessLog -> Recovery -> Handler`（当前写法）：发生 Panic 时，既有 `Recovery` 的堆栈报警日志，**也有一条完整的 `status=500` 的 `http access` 访问日志**（以后在阶段 7 统计接口耗时和 5xx 错误率时不会漏记）。
* `Recovery -> AccessLog -> Handler`：发生 Panic 时，只有 `Recovery` 堆栈日志，**丢失了这条请求的 `http access` 访问日志**。
* 至于 **`RequestID` 为什么必须放在最外层（第 1 个）**：因为无论 `AccessLog` 还是 `Recovery` 打日志，都依赖 `c.Request.Context()` 里的 `request_id`。如果把 `RequestID` 放到 `Recovery` 后面，一旦发生 Panic，`Recovery` 打出的崩溃日志里就没有 `request_id` 了，排查线上问题时就无法把崩溃堆栈和具体请求关联起来。
---
