# User:
---
我正在第0阶段的第二步，请你为我提供一个合适的.gitignore
---
# Gemini 4 Argon (High):
---
结合 [stage0-todolist.md](file:///Users/amadeus/work-space/learning/go-seckill/guide/stage0-todolist.md#L31-L60) 中后续步骤的规划（例如 `Makefile` 编译输出到 `bin/`、使用 `.env` 存放敏感配置并保留 `.env.example`、单元测试与覆盖率文件等），你可以将以下内容直接复制到 [implement/.gitignore](file:///Users/amadeus/work-space/learning/go-seckill/implement/.gitignore) 中：

```gitignore
# =========================
# 1. Go 编译产物与测试文件
# =========================
bin/
dist/
*.exe
*.exe~
*.dll
*.so
*.dylib

# go test 编译生成的二进制与覆盖率报告
*.test
*.out
coverage.txt
coverage.html

# Go workspace 文件（本地多模块调试用，通常不提交）
go.work
go.work.sum

# =========================
# 2. 环境变量与本地私有配置
# =========================
.env
.env.local
.env.*.local
!.env.example

# 如果后续有本地覆盖配置（如 config.local.yaml）可一并忽略
configs/*.local.yaml
configs/*.local.yml

# =========================
# 3. 本地数据卷、日志与临时目录
# =========================
data/
logs/
*.log
tmp/

# =========================
# 4. IDE 与编辑器配置
# =========================
.idea/
.vscode/
*.swp
*.swo
*~

# =========================
# 5. 操作系统生成文件
# =========================
.DS_Store
Thumbs.db
```

### 重点说明：
1. **`bin/`**：对应 Step 13 中 `make build` 编译产物的存放目录，避免将二进制文件提交到 Git 仓库。
2. **`.env` 与 `!.env.example`**：忽略包含数据库/Redis 密码的 `.env` 文件，同时通过 `!.env.example` 确保配置模板不会被误忽略（对应 Step 3）。
3. **`configs/*.local.yaml`**：方便你后续在本地调试时新建 `configs/config.local.yaml` 而不污染仓库中的默认 `configs/config.yaml`。
---

# User:
---
接下来告诉我如何完成deploy/docker-compose.yml
---
# Gemini 4 Argon (High):
---
在 [stage0-todolist.md](file:///Users/amadeus/work-space/learning/go-seckill/guide/stage0-todolist.md#L67-L80) 的 **Step 3** 中，我们需要完成三个文件：`.env.example`（及本地 `.env`）、`deploy/mysql/init.sql` 和 `deploy/docker-compose.yml`。

> [!TIP]
> 在开始 Step 3 前，别忘了确认 Step 2 的目录骨架和空的 `cmd/api/main.go` 已经建好并通过 `go build ./...`。

---

### 1. 创建 `.env.example` 和 `.env`

在 `implement/.env.example` 中写入环境变量模板（此文件提交到 Git），然后复制一份为 `implement/.env`（此文件已被 `.gitignore` 忽略）：

```dotenv
# MySQL 配置
MYSQL_PORT=13306
MYSQL_ROOT_PASSWORD=seckill_root_123
MYSQL_DATABASE=seckill

# Redis 配置
REDIS_PORT=16379
REDIS_PASSWORD=seckill_redis_123
```

---

### 2. 创建数据库初始化脚本 `deploy/mysql/init.sql`

官方 `mysql:8.0` 镜像的 `MYSQL_DATABASE` 环境变量**只能创建一个数据库**。为了同时创建业务库 `seckill` 和后续集成测试用的 `seckill_test`（Step 12 会用到），我们在 `implement/deploy/mysql/init.sql` 中放入初始化 SQL：

```sql
CREATE DATABASE IF NOT EXISTS `seckill` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE IF NOT EXISTS `seckill_test` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```
> MySQL 容器首次启动（数据卷为空）时，会自动执行挂载到 `/docker-entrypoint-initdb.d/` 下的 `.sql` 文件。

---

### 3. 编写 `deploy/docker-compose.yml`

在 `implement/deploy/docker-compose.yml` 中写入以下内容：

```yaml
name: go-seckill

services:
  mysql:
    image: mysql:8.0
    container_name: seckill-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:?请在 .env 中配置 MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE:-seckill}
      TZ: Asia/Shanghai
    ports:
      - "${MYSQL_PORT:-13306}:3306"
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --default-time-zone=+08:00
    volumes:
      - mysql_data:/var/lib/mysql
      - ./mysql/init.sql:/docker-entrypoint-initdb.d/01-init.sql:ro
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "-uroot", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 15s

  redis:
    image: redis:7-alpine
    container_name: seckill-redis
    restart: unless-stopped
    ports:
      - "${REDIS_PORT:-16379}:6379"
    command:
      - redis-server
      - --appendonly
      - "yes"
      - --requirepass
      - ${REDIS_PASSWORD:?请在 .env 中配置 REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 5s

volumes:
  mysql_data:
  redis_data:
```

#### 相比参考项目（`reference/miaosha`）的改进点：
1. **密码强制校验（Fail-Fast）**：使用 `${VAR:?报错信息}` 语法，如果忘记创建 `.env` 或未配置密码，启动时直接报错退出，而不是像参考项目那样回退到硬编码的弱密码 `root123`。
2. **补齐 Redis 健康检查與持久化**：参考项目的 Redis 没有配 `healthcheck`，这里启用了 AOF 持久化（`--appendonly yes`）并配置了 `redis-cli ping` 健康检查。
3. **命名数据卷（Named Volumes）**：底部声明了 `mysql_data` 和 `redis_data`，执行 `docker compose down` 不会丢失数据，只有执行 `docker compose down -v` 才会清空数据卷（重跑 `init.sql`）。

---

### 4. 启动与验证（注意 `.env` 路径细节）

因为 `docker-compose.yml` 放在子目录 `deploy/` 下，而 `.env` 在 `implement/` 根目录，在 `implement/` 目录下执行命令时**需要显式加上 `--env-file .env`**（后续我们会把它封装进 `Makefile` 的 `make up` 中）：

```bash
# 1. 根据模板生成本地 .env
cp .env.example .env

# 2. 启动容器
docker compose --env-file .env -f deploy/docker-compose.yml up -d

# 3. 查看状态（等待几秒后两个服务 STATUS 都应包含 (healthy)）
docker compose --env-file .env -f deploy/docker-compose.yml ps
```

验证连接是否正常：
```bash
# 验证 MySQL 里的 seckill 和 seckill_test 库是否都已创建
docker exec -it seckill-mysql mysql -uroot -pseckill_root_123 -e "SHOW DATABASES;"

# 验证 Redis 连接
docker exec -it seckill-redis redis-cli -a seckill_redis_123 ping
```
---
