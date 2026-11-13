# 测试 · 阶段 0

> 涉及代码：`internal/config/config_test.go`、`internal/logger/logger_test.go`、`internal/server/server_test.go`、`internal/server/integration_test.go`、`Makefile` 的 `test` / `test-integration`
> 用法说明见 [../README.md](../README.md)

---

## 一、Go 测试基础

### Q1. `go test` 是怎么找到测试的？ ⭐

1. 一个函数要被 `go test` 当作测试执行，需要满足哪三个条件？（文件名、函数名、参数）
2. `_test.go` 文件会被编译进 `go build` 产出的二进制吗？
3. `go test ./...` 里的 `./...` 是什么意思？
4. **动手**：只运行某一个测试：`go test -run TestHealthAndReadyz -v ./internal/server/`。再试试只跑某个子测试：`go test -run 'TestMiddlewares/Recovery' -v ./internal/server/`。`-run` 的参数是什么语法？

---

### Q2. `t.Errorf` vs `t.Fatalf` ⭐ 🎯

**看代码**：`server_test.go` 的 `success response` 子测试——先 `t.Fatalf` 检查状态码，后 `t.Errorf` 检查响应体

1. 两者都会把测试标记为失败。区别是什么？
2. 为什么"状态码不对"用 `Fatalf`、"字段值不对"用 `Errorf`？如果把第一个也换成 `Errorf`，状态码错了之后继续往下执行 `json.Unmarshal`，会看到什么样的输出？有帮助还是添乱？
3. 总结一条选择标准。
4. `t.Fatalf` 能在测试里自己开的 goroutine 中调用吗？查一下文档。

---

### Q3. 子测试与 `t.Run` ⭐⭐

**看代码**：`server_test.go` 的 `TestMiddlewares` 里三个 `t.Run(...)`

1. 把三个场景写成三个 `t.Run` 子测试，和写成三个独立的 `TestXxx` 函数，或者干脆顺序写在一个函数里，各有什么不同？
2. `TestMiddlewares` 的每个子测试都**重新** `newTestServer(&logBuf)`，而 `TestResponseAndErrorCodeMapping` 的三个子测试**共用**一个 `srv` 和 `logBuf`。共用会带来什么隐患？设想第三个子测试要断言"日志里不包含某个字符串"。
3. `TestHealthAndReadyz` 分 (a)(b)(c) 三段顺序执行，后一段依赖前一段留下的状态（比如 `MarkShuttingDown()` 之后就回不去了）。如果有人在 (c) 后面加一段 (d) 想测正常状态，会怎样？这种测试的脆弱点在哪？
4. Go 社区很推崇**表驱动测试（table-driven tests）**。`config_test.go` 里对 `Validate` 的各种非法配置的测试，适合改写成表驱动吗？试着写出表的结构（一个 struct 切片，每个元素有哪些字段）。

---

### Q4. 四个测试辅助工具 ⭐⭐

**看代码**：`config_test.go` 的 `setRequiredSecrets`

1. `t.Helper()`：**动手**：把它注释掉，再故意让 `setRequiredSecrets` 里某处失败，对比报错信息里的行号指向哪。它解决的是什么问题？
2. `t.Setenv(key, value)`：比起直接 `os.Setenv`，它多做了什么？测试结束后环境变量会怎样？
3. `t.Setenv` 有个限制：用了它的测试不能调用 `t.Parallel()`，否则直接 panic。为什么要有这个限制？环境变量属于"谁"的状态？
4. `t.TempDir()`：它创建的目录什么时候被删除？比起自己写 `os.MkdirTemp` + `defer os.RemoveAll` 好在哪？
5. `t.Cleanup(fn)` 和 `defer fn()` 的区别是什么？在什么情况下只能用 `t.Cleanup`？（提示：清理逻辑写在辅助函数里，辅助函数返回时……）

---

### Q5. 为什么 `setRequiredSecrets` 要先清空一大堆环境变量？ ⭐⭐

**看代码**：`config_test.go` 开头那个长长的 `envKeys` 列表

1. `make test` 是通过 Makefile 执行的，而 Makefile 开头 `-include .env` + `export`。这意味着测试进程启动时，环境里已经有了什么？
2. 如果不清空，`TestLoad_Defaults` 断言 `cfg.MySQL.Port == 13306`。假如你的 `.env` 里写了 `MYSQL_PORT=3307`，会怎样？同一个测试在你机器上失败、在同事机器上通过，这类问题叫什么？
3. 好的单元测试应该具备"可重复"的性质：结果只取决于代码，不取决于在谁的机器上、什么时候、以什么顺序运行。在这四份测试文件里，找出为"可重复"所做的努力。
4. 这个 `envKeys` 列表是手工维护的。将来 `config.go` 新增一个环境变量但忘了加到这里，会怎样？有没有更不容易遗漏的办法？

---

## 二、测试 HTTP

### Q6. `httptest.NewRecorder` 的原理 ⭐⭐ 🎯

**看代码**：`server_test.go`

```go
req := httptest.NewRequest(http.MethodGet, "/healthz", nil)
rec := httptest.NewRecorder()
srv.Engine().ServeHTTP(rec, req)
```

1. 这三行执行时，有没有监听端口？有没有建立 TCP 连接？
2. 它能工作，是因为 `ServeHTTP(w http.ResponseWriter, r *http.Request)` 的两个参数——一个是接口，一个是可以凭空构造的结构体。`rec` 是什么类型？它为什么能传给一个要求 `http.ResponseWriter` 的参数？
3. 回顾 gin Q1：这再次印证了 Gin 的 Engine 只是一个 `http.Handler`。测试直接调用 `ServeHTTP`，等于绕过了 `http.Server` 的哪些部分？
4. 所以这种测试**覆盖不到**哪些行为？以下哪些能测、哪些不能：
   - 中间件顺序
   - 统一响应格式
   - `ReadTimeout` / `WriteTimeout`
   - 优雅停机
   - 路由 404/405
5. 对于测不到的部分，`httptest` 还提供了 `NewServer`，它会真的在一个随机端口上启动服务。两者各自适合什么场景？
6. `Server` 特意暴露了一个 `Engine()` 方法"方便测试"。为了测试而增加公开方法，你怎么看？有没有别的办法？

---

### Q7. 假的依赖 ⭐⭐ 🎯

**看代码**：`server_test.go` 的 `TestHealthAndReadyz` 里用闭包伪造的 mysql/redis checker

1. 这个测试需要启动 Docker 吗？它是怎么模拟"Redis 挂了"的？
2. 能这样做的前提是什么？如果 `handleReadyz` 内部直接调用 `mysql.Ping(ctx, s.db)`（即 Server 持有真实的 `*gorm.DB`），这个测试还能这么写吗？回顾 go-basics Q9 第 3 问：**可测试性**和**依赖注入**是什么关系？
3. 测试里 `redisHealthy := true`，后面改成 `false`，闭包里读到的值随之变化。闭包捕获的是变量的**值**还是变量**本身**？
4. 这个测试验证了"checker 返回错误时，/readyz 返回 503"。它验证了"真实的 Redis 挂掉时，`rdb.Ping` 会返回错误"吗？这部分信心从哪来？（引出下一节）
5. 了解几个术语的区别：stub、mock、fake。这里的闭包更接近哪一个？

---

### Q8. 测试断言了什么，没断言什么？ ⭐⭐⭐

**看代码**：`server_test.go` 整个文件

好的测试不仅要"通过"，还要"在代码被改坏时失败"。针对下面每一种"改坏"，判断现有测试能否发现。**至少挑两个动手验证**（改坏 → 跑测试 → 还原）：

1. 把 `Recovery` 和 `AccessLog` 的注册顺序对调
2. `Fail` 的未知错误分支，不小心把 `err.Error()` 放进了 `Msg`
3. `RequestID` 中间件忘了写 `c.Request = c.Request.WithContext(ctx)`
4. `handleHealthz` 里加了一行检查 MySQL 的代码
5. `Recovery` 里漏掉了 `debug.Stack()`，日志里不再有堆栈
6. 去掉 `engine.HandleMethodNotAllowed = true`

对于发现不了的那些：是否值得补一个测试？怎么补？

这种"故意改坏代码看测试能不能发现"的思路有个正式名字，叫**变异测试（mutation testing）**。

---

## 三、集成测试

### Q9. 单元测试与集成测试 ⭐⭐ 🎯

**看代码**：`integration_test.go`；`Makefile` 的两个 test 目标

1. 用一句话分别定义"单元测试"和"集成测试"。本项目是按什么标准把二者分开的？
2. 两类测试在**速度**、**稳定性**、**能发现的问题类型**上各有什么特点？
3. "测试金字塔"主张哪种测试多、哪种少？为什么？
4. 对秒杀系统最核心的"不超卖"逻辑（阶段 2 的 `UPDATE ... SET stock = stock - 1 WHERE stock > 0`）：它的正确性依赖于 MySQL 的真实行为。用一个假的 repository 做单元测试，能证明"并发下不超卖"吗？这类逻辑应该主要靠哪种测试来保证？

---

### Q10. build tag ⭐⭐ 🎯

**看代码**：`integration_test.go` 第一行 `//go:build integration`

1. 这一行是注释还是指令？它对位置和格式有什么要求？（必须在 `package` 之前吗？后面要不要空一行？）
2. 执行 `go test ./...`（不带 `-tags`）时，编译器会怎么对待这个文件？
3. **动手**：先 `make docker-down`，再 `make test`。通过了吗？这就是把集成测试隔离开的价值——没装 Docker 的人也能跑单元测试。
4. `.golangci.yml` 里专门配置了 `build-tags: [integration]`。不配的话，这个文件里的 lint 问题会被报告吗？一个"不被编译、不被 lint、平时也不跑"的文件，过半年会变成什么样？
5. 除了 build tag，还有其他隔离慢测试的办法，例如 `testing.Short()` + `go test -short`，或者检测到没有环境变量时 `t.Skip()`。和 build tag 相比各有什么优劣？
6. 同一个 `internal/server` 目录里，`server_test.go` 写的是 `package server`，`integration_test.go` 写的是 `package server_test`。Go 允许一个目录里有两个包名吗？后者能访问 `server` 包里未导出的东西（如 `handleHealthz`）吗？这两种方式分别叫白盒测试和黑盒测试，各自适合测什么？

---

### Q11. 测试数据的隔离与清理 ⭐⭐⭐

**看代码**：`integration_test.go` 的 `cfg.MySQL.Database = "seckill_test"`、`cfg.Redis.DB = 1`、`t.Cleanup(...)`

1. 为什么强制把库名改成 `seckill_test`？如果某个集成测试的清理逻辑是 `DELETE FROM orders`，而它连的是你的开发库，会怎样？
2. 这个测试插入一行、最后用 `t.Cleanup` 删掉这一行。如果测试进程在中途被 `kill -9`，清理还会执行吗？数据库里会留下什么？下次运行受影响吗？
3. 对比几种让集成测试"互不干扰"的策略，各自的代价是什么：
   - 每个测试只清理自己创建的数据（当前做法）
   - 每个测试开始前 `TRUNCATE` 相关的表
   - 每个测试在一个事务里执行，结束时回滚
   - 每次测试启动一个全新的数据库容器（了解 `testcontainers-go` 这个名字）
4. "事务回滚"方案看起来最干净，但它不适用于阶段 2 的并发扣库存测试。为什么？（提示：并发的多个 goroutine 能共用一个事务吗？）
5. 测试里读取密码依赖环境变量，注释写着"不依赖相对路径的 config.yaml"（所以用了 `config.Load("")`）。`go test` 执行某个包的测试时，进程的**工作目录**是哪里？这就是为什么测试里用相对路径 `configs/config.yaml` 找不到文件。

---

### Q12. 测试缓存 ⭐⭐

1. **动手**：连续执行两次 `go test ./internal/config/`。第二次的输出末尾多了什么字样？
2. Go 根据什么判断"可以直接用上次的结果"？改了源码会重跑；改了测试读取的环境变量或文件呢？
3. 对集成测试来说：代码没变、环境变量没变，但**数据库里的数据或表结构变了**。Go 的缓存感知得到吗？这时测试显示 `ok (cached)` 可信吗？
4. 怎么强制不使用缓存？（`-count=1`）你认为 `Makefile` 的 `test-integration` 目标应该加上它吗？

---

## 四、race 检测与覆盖率

### Q13. 测试里的 `-race` ⭐⭐

**看代码**：`Makefile` 的 `go test -race -v ./...`

1. 现有的测试里，有没有哪个测试真的启动了多个 goroutine 并发访问 `Server`？如果没有，`-race` 在这些测试上能发现 `shuttingDown` 字段的并发问题吗？（回顾 go-concurrency Q11："没被执行到的竞态检测不出来"）
2. **动手**（和 go-concurrency Q9 的实验可以合并做）：写一个并发测试——启动 50 个 goroutine 同时请求 `/readyz`，同时另一个 goroutine 调用 `MarkShuttingDown()`，用 `sync.WaitGroup` 等它们结束。在当前代码下 `-race` 是否通过？
3. 阶段 2 的"防超卖"测试会是这种形式的放大版：N 个 goroutine 抢 M 件库存，最后断言成功数恰好等于 M。先想一想，这种测试的断言要检查哪几个数字才算完整？

---

### Q14. 覆盖率 ⭐⭐

1. **动手**：

   ```bash
   go test -coverprofile=cover.out ./...
   go tool cover -func=cover.out      # 每个函数的覆盖率
   go tool cover -html=cover.out      # 浏览器里逐行查看，红色是没覆盖到的
   ```

   哪些文件覆盖率最低？`main.go` 是多少？为什么？
2. 在 HTML 视图里找到 `config.go` 中标红的行。它们是什么情况下才会走到的分支？值得补测试吗？
3. `internal/infra/mysql` 和 `internal/infra/redis` 在单元测试下的覆盖率是多少？带上 `-tags=integration` 再测一次呢？
4. 覆盖率 100% 能说明代码没有 bug 吗？回到 Q8：一行代码"被执行过"和"它的行为被断言过"是一回事吗？
5. 你会给这个项目设定一个覆盖率目标吗？设多少？对哪些代码要求高、哪些可以放低？
