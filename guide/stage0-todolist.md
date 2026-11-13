# 阶段 0 TodoList：工程骨架

> 编写日期：2026-11-05
> 对应路线图：`guide/roadmap.md` 阶段 0
> 本机环境：Go 1.26、Docker 29、Docker Compose v5（已确认）
> 阶段目标：搭好一个干净、可重复启动的工程底座。本阶段**不写任何秒杀业务**。

按顺序完成即可，后面的步骤依赖前面的产出。每一步都附有"完成标准"，达到后再进入下一步。

---

## Step 1：技术选型（先写下来再动手）

- [x] 在 `implement/README.md` 或 `implement/docs/decisions.md` 记录选型和理由，每项写一两句即可

| 领域 | 推荐 | 备选 | 说明 |
|---|---|---|---|
| HTTP 框架 | Gin | 标准库 `net/http`（1.22+ 已支持路由方法匹配）、Hertz | Gin 资料多，参考项目也用它 |
| 数据库访问 | `database/sql` + sqlx | GORM | 后续阶段大量依赖 `UPDATE ... WHERE status=?` 并检查 `RowsAffected`，手写 SQL 更直观，面试也更好讲。用 GORM 也可以，但要清楚它生成的 SQL |
| MySQL 驱动 | go-sql-driver/mysql | — | |
| Redis 客户端 | redis/go-redis v9 | — | |
| 日志 | 标准库 `log/slog` | zap | slog 零依赖，够用 |
| 配置 | YAML + 环境变量覆盖（koanf 或手写） | viper | viper 依赖较重；手写也就几十行 |
| 迁移 | golang-migrate | goose | 迁移文件用纯 SQL |
| 测试 | 标准库 testing + testify | — | 集成测试用 compose 中的测试库或 testcontainers-go |

**完成标准**：选型已记录，并且能说出每项为什么不用备选。

---

## Step 2：初始化仓库

- [X] 在 `implement/` 下 `git init`，之后每完成一个 Step 提交一次（commit 历史本身也是项目成果）
- [X] `go mod init <模块路径>`，比如 `github.com/<你的用户名>/go-seckill`
- [X] `.gitignore`：编译产物、`.env`、IDE 文件、本地数据卷目录
- [X] 建立目录骨架（空目录可先放 `.gitkeep`）：

```
implement/
├── cmd/
│   └── api/main.go          # 入口：只做组装和启动，不写业务
├── internal/
│   ├── config/              # 配置加载与校验
│   ├── server/              # HTTP server、路由、中间件
│   ├── handler/             # HTTP 处理器（阶段 1 起）
│   ├── service/             # 业务逻辑（阶段 1 起）
│   ├── repository/          # 数据访问（阶段 1 起）
│   ├── model/               # 领域模型（阶段 1 起）
│   └── infra/               # MySQL / Redis 等基础设施的客户端初始化
│       ├── mysql/
│       └── redis/
├── pkg/                     # 只放与本项目无关、真正可复用的代码，没有就空着
├── migrations/              # 版本化 SQL
├── configs/config.yaml
├── deploy/docker-compose.yml
├── .env.example
├── Makefile
└── README.md
```

**完成标准**：`go build ./...` 通过（`main.go` 里先放一个空的 `main`）。

**思考**：为什么大部分代码放 `internal/` 而不是 `pkg/`？（参考项目把 redis、mq 都放在 `pkg/`，你认为合理吗？）

---

## Step 3：docker-compose 启动依赖

- [X] `deploy/docker-compose.yml` 包含：
  - MySQL 8.x：指定字符集 `utf8mb4`，通过环境变量创建业务库 `seckill` 和测试库 `seckill_test`（或用初始化脚本）
  - Redis 7.x
- [X] 两个服务都配置 `healthcheck`
- [X] 数据挂载到命名卷，`docker compose down` 不丢数据，`down -v` 才清空
- [X] 端口映射用非默认端口（比如 13306、16379），避免和本机已有服务冲突
- [X] 密码等写在 `.env`（不提交）；提交 `.env.example` 作为模板

**完成标准**
- `docker compose -f deploy/docker-compose.yml up -d` 后，`docker compose ps` 显示两个服务都是 `healthy`
- 能用 `mysql` / `redis-cli` 客户端连上

---

## Step 4：配置模块 `internal/config`

- [X] 定义一个 `Config` 结构体，按模块分组：`Server`（端口、读写超时、停机超时）、`MySQL`（DSN 相关、连接池参数）、`Redis`（地址、密码、DB、连接池）、`Log`（级别、格式）
- [X] 加载顺序：代码默认值 → `configs/config.yaml` → 环境变量覆盖
- [X] 密码、密钥这类敏感项**只从环境变量读取**，YAML 中不出现
- [X] `Validate()`：必填项缺失或取值非法（端口越界、超时 ≤ 0 等）时启动直接失败，报错要说明是哪个字段
- [X] 单元测试：默认值生效、环境变量覆盖生效、缺少必填项时报错

**完成标准**：`go test ./internal/config/...` 通过。

**参考**：`reference/miaosha/backend/internal/config/config.go` 中环境变量覆盖的做法可以借鉴。

---

## Step 5：日志

- [X] 基于 `log/slog` 初始化全局 logger：开发环境输出 text，其他环境输出 JSON；级别可配置
- [X] 每条日志带上固定字段：`service=api`、`version`（可以先写死）
- [X] 约定：日志通过参数或 context 传递，**不要**在业务代码里到处直接用 `fmt.Println`

**完成标准**：启动时打印一条包含配置摘要的日志（**不能打印密码**）。

---

## Step 6：MySQL / Redis 客户端初始化 `internal/infra`

- [X] `mysql.New(cfg config.MySQLConfig) (*gorm.DB, func() error, error)`（或封装结构体提供 `Close() error`）：
  - 使用 GORM（`gorm.io/gorm` + `gorm.io/driver/mysql`）打开连接，从 `*gorm.DB` 获取底层 `*sql.DB`（`db.DB()`）
  - 设置连接池参数：`MaxOpenConns`、`MaxIdleConns`、`ConnMaxLifetime`、`ConnMaxIdleTime`（先用保守值，阶段 8 再调）
  - DSN 带上 `parseTime=true`、`loc`、`charset=utf8mb4`（已在 `config.MySQLConfig.DSN()` 中实现）
  - 启动时带超时地 `PingContext`，失败直接退出
- [X] `redis.New(cfg config.RedisConfig) (*redis.Client, error)`：同样设置连接池和超时，并带超时 `Ping` 一次
- [X] 两者都能在优雅停机时调用 `Close()` 释放连接
- [X] 不使用全局变量保存客户端，通过构造函数参数传递（依赖注入）

**完成标准**：服务启动时依赖不可用会**快速失败**，并给出清晰的错误信息；依赖可用时正常启动。

**思考**：`MaxOpenConns` 设成 2000（参考项目的值）有什么问题？MySQL 默认的 `max_connections` 是多少？

---

## Step 7：数据库迁移

- [X] 安装 golang-migrate CLI（或者把迁移嵌入程序，用 `embed` 加载）
- [X] 建立迁移文件命名规范：`000001_xxx.up.sql` / `000001_xxx.down.sql`
- [X] 写一个试验用的迁移（比如建一张 `ping_test` 表），验证 `up`、`down`、`version` 都能正常工作；验证后可以删掉或保留为第一个正式迁移
- [X] 在 Makefile 中封装 `migrate-up`、`migrate-down`、`migrate-new name=xxx`

**完成标准**：在空库上 `make migrate-up` 后表存在，`make migrate-down` 后表消失。

**注意**：
- 阶段 1 的业务表（用户、商品、活动、订单）在阶段 1 再设计，这一步只把迁移流程跑通。
- 虽然项目选用了 GORM（自带 `AutoMigrate`），但建表/改表（DDL）依然统一用 `golang-migrate` 管理版本化 SQL，GORM 只负责运行时的增删改查（DML/DQL）。

**思考**：GORM 自带 `db.AutoMigrate()`，为什么生产项目和本项目中仍然推荐用 `golang-migrate` 手写 SQL 迁移文件？（提示：从回滚能力、多实例启动时的 MySQL MDL 锁、以及结构体 Tag 复杂度三个角度思考）

---

## Step 8：HTTP 服务骨架 `internal/server`

- [X] 用 `http.Server` 包装 Gin engine，显式设置 `ReadTimeout`、`WriteTimeout`、`IdleTimeout`（不要直接用 `r.Run()`，它无法优雅停机）
- [X] 统一响应格式，例如 `{"code": 0, "msg": "ok", "data": ...}`，并提供 `Success` / `Fail` 工具函数
- [X] 错误码规范：定义一个错误码文件，按模块划分号段（通用 1xxxx、用户 2xxxx、秒杀 3xxxx……），业务错误与 HTTP 状态码的映射规则写清楚
- [X] 中间件（顺序要想清楚）：
  - Recovery：panic 时记录堆栈并返回统一格式的 500
  - RequestID：读取请求头 `X-Request-ID`，没有就生成；写入 context 和响应头
  - AccessLog：记录方法、路径、状态码、耗时、request_id
- [X] 让 handler 里 `slog` 打的日志能自动带上 request_id（通过 context 取）

**完成标准**：随便请求一个不存在的路由，返回统一格式的 404，且访问日志里有 request_id。

---

## Step 9：健康检查

- [X] `GET /healthz`（存活检查）：进程活着就返回 200，**不检查依赖**
- [X] `GET /readyz`（就绪检查）：带超时地检查 MySQL 和 Redis，返回每个依赖的状态；任一不可用返回 503

**完成标准**：`make docker-stop s=redis`（或 `docker compose --env-file .env -f deploy/docker-compose.yml stop redis`）后 `/readyz` 返回 503 并标明 Redis 不可用，`/healthz` 仍返回 200；`make docker-start s=redis` 后恢复。

**思考**：为什么要把 liveness 和 readiness 分开？如果 `/healthz` 也检查依赖，在 K8s 里会发生什么？

---

## Step 10：优雅停机

- [X] `main.go` 用 `signal.NotifyContext` 监听 `SIGINT` / `SIGTERM`
- [X] 收到信号后：
  1. 让 `/readyz` 返回 503（不再接收新流量）
  2. `server.Shutdown(ctx)`，带可配置的超时，等待在途请求完成
  3. **按初始化的逆序**关闭资源：HTTP → Redis → MySQL
  4. 打印停机日志后退出
- [X] 加一个临时调试接口（比如 `/debug/slow`，sleep 5 秒）来验证

**完成标准**：请求 `/debug/slow` 期间按 Ctrl+C，这个请求仍能正常返回，之后进程才退出；超过停机超时则强制退出并打印日志。

**为什么现在就做**：阶段 4 的 API 和 Worker 都依赖"停机前处理完在途任务"，骨架现在就要打对。

---

## Step 11：组装入口 `cmd/api/main.go`

- [X] `main` 只做这几件事：加载配置 → 初始化日志 → 初始化依赖 → 组装路由 → 启动 → 等待停机
- [X] 建议写一个 `run(ctx) error` 函数，由 `main` 调用并处理退出码，方便测试，也避免到处 `log.Fatal`
- [X] 不使用 `init()` 做隐式初始化

**完成标准**：`main.go` 控制在 100 行左右，读起来一眼能看出启动流程。

**参考**：对比 `reference/miaosha/backend/cmd/api/main.go`，注意它在启动时会清空 Redis、全量预热（R06），你的启动流程里**不要**有这类破坏性操作。

---

## Step 12：测试基础设施

- [X] 单元测试：config、统一响应、错误码映射、中间件（用 `httptest`）
- [X] 集成测试：
  - 用 build tag（如 `//go:build integration`）或环境变量和单元测试区分开
  - 连接 compose 中的 `seckill_test` 库，或使用 testcontainers-go
  - 至少写一个：跑完迁移后连库成功，`/readyz` 返回 200
- [X] 所有测试都能在 `-race` 下通过

**完成标准**：`make test` 只跑单元测试且不依赖 Docker；`make test-integration` 跑集成测试。

---

## Step 13：Makefile 与代码检查

- [X] Makefile 目标：
  - `docker-up` / `docker-down` / `docker-down-v` / `docker-ps` / `docker-stop` / `docker-start` / `docker-logs`：启停与查看依赖
  - `run`：本地启动 API
  - `build`：编译到 `bin/`
  - `test` / `test-integration`：都带 `-race`
  - `lint`：golangci-lint
  - `migrate-up` / `migrate-down` / `migrate-new`
- [X] `.golangci.yml`：先开启一组常用 linter（govet、errcheck、staticcheck、ineffassign、unused、gofmt / goimports），**特别是 `errcheck`**，参考项目里大量 `_ = xxx` 忽略错误的问题就靠它来防

**完成标准**：`make lint` 无告警。

---

## Step 14：README 与阶段收尾

- [X] README 写清：项目简介（一句话即可）、依赖、启动步骤、常用 make 命令、目录说明
- [X] 按 README 从零执行一遍：`make docker-down-v` → `make docker-up` → `make migrate-up` → `make run` → 访问 `/readyz`
- [X] 打一个 git tag：`stage-0`

---

## 阶段 0 总验收

- [X] 从空环境按 README 操作，5 分钟内能把服务跑起来
- [X] `/healthz`、`/readyz` 行为符合 Step 9 的描述
- [X] 依赖不可用时启动快速失败，报错清晰
- [X] Ctrl+C 能优雅停机，在途请求不被中断
- [X] `make test`、`make test-integration`、`make lint` 全部通过
- [X] 代码中没有全局可变状态、没有被忽略的 error、没有硬编码的密码

完成后，可以让 AI 助手 review `implement/` 的代码，再进入阶段 1。
