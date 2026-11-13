# User:
---
我正在第0阶段，指导我完成Step 14
---
# Gemini 4 Argon (High):
---
**Step 14** 是阶段 0（工程骨架）的收尾步骤。这一步的目标是：**把工程使用方式文档化，用“全新环境”视角做一次端到端验收，并用 Git Tag 定格阶段 0 的里程碑基线。**

你可以按以下三个步骤在 `implement/` 下完成：

---

### 第一步：完善 [README.md](file:///Users/amadeus/work-space/learning/go-seckill/implement/README.md)

目前的 [README.md](file:///Users/amadeus/work-space/learning/go-seckill/implement/README.md) 只有两行简介。一份合格的工程 README 应该让任何拿到仓库的人（包括几个月后的你自己、或者看你 GitHub 仓库的面试官）在 **5 分钟内不看源码就能把服务和所有测试跑起来**，同时把 Step 1 的技术选型也沉淀进去。

> [!TIP]
> **容易遗漏的细节**：新克隆仓库时是没有 `.env` 文件的（因为被 [.gitignore](file:///Users/amadeus/work-space/learning/go-seckill/implement/.gitignore#L40-L43) 忽略了），而 [Makefile](file:///Users/amadeus/work-space/learning/go-seckill/implement/Makefile) 和 [docker-compose.yml](file:///Users/amadeus/work-space/learning/go-seckill/implement/deploy/docker-compose.yml) 都强依赖 `.env` 中的密码变量。因此**快速开始的第一步必须是 `cp .env.example .env`**。

你可以参考以下结构完善 [implement/README.md](file:///Users/amadeus/work-space/learning/go-seckill/implement/README.md)：

````markdown
# Go-Seckill 高并发秒杀系统

基于 Go 构建的高并发秒杀系统，采用“先保证数据一致性、再渐进式演进性能”的工程路线（同步事务 → Redis Lua 预占 → MQ 异步削峰 → 状态机与可靠补偿）。

## 1. 技术选型

| 领域 | 选型 | 说明 |
| :--- | :--- | :--- |
| HTTP 框架 | Gin | 路由性能好、中间件生态成熟 |
| 数据库访问 | GORM (`gorm.io/gorm`) | 负责运行时 DML/DQL；配合底层 `*sql.DB` 精细控制连接池与条件更新 `RowsAffected` |
| 数据库 | MySQL 8.0 | 核心交易数据与库存最终一致性落库（`utf8mb4`） |
| 缓存 / 预占 | Redis 7 (`go-redis/v9`) | 后续阶段用于活动预热、Lua 原子预占库存与限流 |
| 结构化日志 | 标准库 `log/slog` | 零第三方依赖，支持 Context 透传 `request_id` 与 JSON/Text 双格式 |
| 配置管理 | `gopkg.in/yaml.v3` + 环境变量 | 默认值 → YAML 配置文件 → 环境变量覆盖，敏感密码仅从环境变量读取 |
| 数据库迁移 | `golang-migrate` | 使用版本化纯 SQL 管理 DDL，避免生产环境 `AutoMigrate` 带来的锁表与回滚风险 |
| 测试与质量 | `testing` + `golangci-lint` | 单元测试与集成测试分离（`//go:build integration`），全链路开启 `-race` 与 `errcheck` |

## 2. 目录结构

```text
implement/
├── cmd/
│   └── api/main.go              # API 进程入口：组装依赖、启动服务与优雅停机
├── configs/
│   └── config.yaml              # 默认非敏感配置
├── deploy/
│   ├── docker-compose.yml       # 本地开发依赖（MySQL 8.0 / Redis 7）
│   └── mysql/init.sql           # 容器首次初始化脚本（创建 seckill_test 测试库）
├── internal/
│   ├── config/                  # 配置加载、环境变量覆盖与合法性校验
│   ├── infra/                   # 基础设施客户端初始化（MySQL / Redis）
│   ├── logger/                  # slog 结构化日志封装与 Context 注入
│   └── server/                  # HTTP Server、路由、中间件、统一响应与健康检查
├── migrations/                  # golang-migrate 版本化 SQL 迁移文件
├── .env.example                 # 环境变量模板（含敏感配置）
├── .golangci.yml                # 静态代码检查配置
└── Makefile                     # 常用工程化命令封装
```

## 3. 环境依赖

- **Go**: 1.26+
- **Docker** & **Docker Compose**: v2+
- **golang-migrate CLI**: `go install -tags 'mysql' github.com/golang-migrate/migrate/v4/cmd/migrate@latest`
- **golangci-lint**: v2+

## 4. 快速开始（5 分钟从零启动）

### 4.1 准备环境变量
```bash
cp .env.example .env
```

### 4.2 启动基础设施（MySQL & Redis）
```bash
make docker-up
```
> 该命令会自动等待容器 `healthcheck` 变为 `healthy`（默认映射端口：MySQL `13306`，Redis `16379`）。

### 4.3 执行数据库迁移
```bash
make migrate-up
```

### 4.4 启动 API 服务
```bash
make run
```

### 4.5 验证服务状态
另开一个终端执行：
```bash
# 存活检查（Liveness）
curl -i http://127.0.0.1:8080/healthz

# 就绪检查（Readiness，检查 MySQL 与 Redis 连通性）
curl -i http://127.0.0.1:8080/readyz
```

## 5. 常用 Make 命令

| 命令 | 说明 |
| :--- | :--- |
| `make run` | 本地启动 API 服务（自动加载 `.env`） |
| `make build` | 编译二进制文件到 `bin/api` |
| `make docker-up` | 启动 MySQL 和 Redis 容器并等待健康检查通过 |
| `make docker-down` | 停止并移除容器（保留命名卷数据） |
| `make docker-down-v` | 停止容器并**清空数据卷**（重置全新环境） |
| `make docker-ps` | 查看依赖容器运行与健康状态 |
| `make docker-stop s=redis` | 停止单个服务（用于故障演练） |
| `make docker-start s=redis` | 恢复启动单个服务 |
| `make migrate-new name=xxx` | 创建新的 `.up.sql` / `.down.sql` 迁移文件 |
| `make migrate-up` | 执行所有未应用的向上迁移 |
| `make migrate-down` | 回滚最后一次迁移（每次仅回滚 1 步） |
| `make migrate-version` | 查看当前数据库迁移版本 |
| `make test` | 运行所有单元测试（开启 `-race`，不依赖 Docker） |
| `make test-integration` | 对 `seckill_test` 执行迁移并运行集成测试 |
| `make lint` | 运行 `golangci-lint` 静态代码检查 |
| `make fmt` | 使用 `gofmt` / `goimports` 自动格式化代码 |
````

---

### 第二步：按 README 从零执行一遍全流程验证

写完 `README.md` 后，在 `implement/` 目录下按顺序跑一遍“清空数据卷 → 从头启动 → 迁移 → 测通接口与测试”的完整闭环，确保没有任何隐式依赖遗漏：

```bash
cd implement

# 1. 彻底销毁容器和命名卷（模拟一台刚 clone 下代码的新机器）
make docker-down-v

# 2. 重新拉起 MySQL 和 Redis（同时触发 deploy/mysql/init.sql 自动创建 seckill_test 库）
make docker-up

# 3. 执行数据库迁移
make migrate-up

# 4. 跑一遍单元测试、集成测试和静态检查，确保全绿
make test
make test-integration
make lint

# 5. 启动服务并验证 /readyz
make run
```
服务启动后，在另一个终端请求 `curl -i http://127.0.0.1:8080/readyz` 确认返回 `200 OK`，然后回到运行服务的终端按 `Ctrl+C` 确认优雅停机正常退出。

> 💡 **顺手清理提示**：我注意到 `implement/deploy/data/` 下有两个空的 `mysql` 和 `redis` 子目录（之前尝试本地目录挂载时留下的，现在 [docker-compose.yml](file:///Users/amadeus/work-space/learning/go-seckill/implement/deploy/docker-compose.yml#L58-L60) 已经改用命名卷 `mysql_data` 和 `redis_data` 了），可以顺手删掉 `implement/deploy/data` 空目录保持仓库干净。

---

### 第三步：提交代码并打上 `stage-0` 标签

#### 📘 概念小科普：Git 轻量标签 vs 附注标签（Annotated Tag）

在 Git 中打标签有两种方式：
1. **轻量标签（Lightweight Tag）**：`git tag stage-0`
   - **底层实现**：它只是在 `.git/refs/tags/stage-0` 下写了一个包含 40 位 Commit SHA-1 哈希值的文件，相当于一个“不会移动的分支指针”，不包含任何额外信息。
2. **附注标签（Annotated Tag，推荐）**：`git tag -a stage-0 -m "..."`
   - **底层实现**：Git 会在对象数据库（`.git/objects/`）中真正创建一个独立的 **Tag 对象（Tag Object）**。这个对象里不仅记录了它指向的 Commit SHA-1，还完整保存了**打标签者的姓名与邮箱、打标签的时间戳、以及一段里程碑说明信息（message）**。
   - **工程实践**：在正式项目和版本发布（如 `v1.0.0` 或我们的阶段里程碑 `stage-0`）中，**一律推荐使用附注标签（`-a`）**，这样用 `git show stage-0` 查看时能清楚看到该阶段的交付说明与时间点。

在 `implement/` 下执行以下命令完成收尾提交与打标：

```bash
# 1. 提交 README 改动
git add README.md
git commit -m "docs: 完善 README 文档，完成阶段 0 工程骨架收尾"

# 2. 创建附注标签 stage-0
git tag -a stage-0 -m "Stage 0: 完成工程骨架（配置、日志、MySQL/Redis 客户端、迁移、HTTP 服务、健康检查、优雅停机与测试基础设施）"

# 3. 验证标签信息
git show stage-0 --stat
```

---

完成以上三步后告诉我，我会为你：
1. 更新 [stage0-todolist.md](file:///Users/amadeus/work-space/learning/go-seckill/guide/stage0-todolist.md) 的最终进度；
2. 对 `implement/` 整个阶段 0 的代码进行一次全面的 **Code Review**（输出报告到 `guide/`）；
3. 基于阶段 0 涉及的核心知识点，在 `guide/` 下生成 **阶段 0 面试 Q&A 文档**（按规则将问题与答案拆分为两份独立的 `.md` 文件供你自测）。
---
