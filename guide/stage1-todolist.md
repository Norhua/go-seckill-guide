# 阶段 1 TodoList：基础业务域

> 编写日期：2026-11-13
> 对应路线图：`guide/roadmap.md` 阶段 1
> 起点：`implement/` 的 tag `stage-0`
> 阶段目标：把用户、商品、活动、订单四块基础业务和分层结构做对。本阶段**不写秒杀下单**，订单只建表和查询。

按顺序完成，后面的步骤依赖前面的产出。每个 Step 做完提交一次。

**几点说明**

- 文中的 SQL、报错信息和耗时数字，都是在和你相同版本的环境上实际跑出来的（MySQL 8.0.46、GORM v1.31.2、Gin v1.12.0、golang-jwt v5.3.1），可以直接对照你自己的日志
- 「思考」里的问题建议**先想、再动手**，它们的答案会影响你这一步怎么写
- 配套的 `stage1-guide-notes.md` 是写给 AI 助手的指导要点，里面有参考答案和参考建表语句。建议做完对应的 Step 再看
- 需要讲解时，把 Step 编号告诉 AI 助手即可

---

## 本阶段全貌

一个请求在阶段 1 结束时要经过的路：

```
HTTP 请求
  │
  ▼
middleware   RequestID → AccessLog → Recovery → Auth → RequireRole
  │
  ▼
handler      解析并校验请求参数 → 调 service → 写统一响应
  │
  ▼
service      业务规则（用户名能不能用、活动能不能发布……）
  │
  ▼
repository   GORM 读写 MySQL
```

| Step | 内容 | 性质 |
|---|---|---|
| 0 | 阶段 0 收尾 | 修补 |
| 1 | 设计先行 | 只写文档 |
| 2 | 分层与错误处理 | 改造阶段 0 的代码 |
| 3 | GORM 配置与时区 | 改造阶段 0 的代码 |
| 4 | 表结构与迁移 | 四张业务表 |
| 5 | model、第一个 repository、测试地基 | |
| 6 | 密码与 Token | |
| 7 | 注册与登录 | **第一条完整链路**，分层在这里第一次跑通 |
| 8 | 鉴权中间件与路由 | |
| 9 | 创建管理员的命令行工具 | |
| 10 | 商品管理 | |
| 11 | 秒杀活动管理 | 第一次写"条件更新 + 检查影响行数" |
| 12 | 订单查询 | |
| 13 | 端到端验收 | |
| 14 | 收尾 | |

Step 0~3 是在还阶段 0 的债，没有新功能，但后面每一步都踩在它们上面。Step 7 最费时间，做通之后 Step 10~12 都是同一个套路。

---

## Step 0：阶段 0 收尾

阶段 0 的代码有三处尾巴，先清掉。

- [ ] 修复集成测试。在 `stage-0` 上执行 `make test-integration`，每次都是这个结果：

  ```
  integration_test.go:75: cleanup ping_test record failed: sql: database is closed
  --- FAIL: TestIntegration_ReadyzAndMigration
  ```

  每失败一次，`seckill_test.ping_test` 里还会多留下一行。修好后把残留的行删掉
- [ ] 删除 `/debug/slow`（或者只在调试模式下注册）。它是验证优雅停机用的临时接口，不需要登录，调用一次就占住一个 goroutine 5 秒
- [ ] README 和 Makefile 对齐：README 4.1 让人修改 `MIGRATE_BIN` 和 `GOLANGCI_LINT_BIN`，而 Makefile 里的变量叫 `MIGRATE` 和 `GOLANGCI_LINT`；README 里 golangci-lint 的安装命令没有版本号，v2 的包路径也不对，应该是 `github.com/golangci/golangci-lint/v2/cmd/golangci-lint@<版本>`

**完成标准**：`make test`、`make test-integration`、`make lint` 全部通过；连续执行两次 `make test-integration`，`SELECT COUNT(*) FROM seckill_test.ping_test` 的结果不变。

**思考**
1. 测试函数里的 `defer` 和 `t.Cleanup` 注册的函数，谁先执行？多个 `t.Cleanup` 之间呢？对着 `integration_test.go` 第 51、62、73 行，把实际的执行顺序排出来。
2. 这个问题从测试写下来那天就存在，当时为什么是通过的？看 `git show c6b6c2b -- internal/server/integration_test.go`：`errcheck` 逼你去掉的那个 `_ =`，原来盖住了什么？

---

## Step 1：设计先行（不写代码）

- [ ] 在 `implement/docs/design.md`（文件名自定）写下本阶段的设计：数据模型、状态定义、接口清单，每个决定配一两句理由
- [ ] 下面两张表是推荐方案。可以不照做，但偏离的地方要写下理由。这份文档以后就是面试时讲"为什么这么设计"的底稿

**设计决定**

| # | 主题 | 推荐 | 和后面阶段的关系 |
|---|---|---|---|
| 1 | 账号 | 一张 `users` 表，用 `role` 区分普通用户和管理员 | 两类账号不可能出现相同的 ID（R13） |
| 2 | 库存 | 库存属于活动：`total_stock`（总量）和 `available_stock`（可售）。商品表没有库存 | 阶段 5 对账直接用：`total_stock = available_stock + 有效订单占用` |
| 3 | 活动状态 | 库里只存 草稿(1) / 已发布(2) / 已下线(3)。"未开始 / 进行中 / 已结束"在查询时用当前时间和时间窗算出来 | 不会出现"状态说进行中、时间说没开始"的矛盾 |
| 4 | 发布即冻结 | 只有草稿能编辑。发布后库存、限购、时间、价格都不能改 | 阶段 3 把库存预热进 Redis 之后，MySQL 这边的配置不会再变 |
| 5 | 订单号 | `orders.id` 就是订单号，由应用生成，不自增。本阶段不创建订单，测试里手工指定 | 阶段 4 靠它做消息幂等：重复消息 = 主键冲突 |
| 6 | 金额 | 整数，单位是分。Go 里用 `int64` | R16 |
| 7 | 状态值 | 从 1 开始编号，0 不是合法值 | Go 变量没赋值时是 0，不能让它恰好是一个合法状态 |
| 8 | 删除 | 不建物理外键，不做软删除，不提供任何删除接口 | 阶段 2 会看到外键在热点行上引发死锁 |
| 9 | 接口格式 | 前缀 `/api/v1`；时间用 RFC 3339（`2026-11-13T20:00:00+08:00`）；金额是整数分；订单号以**字符串**输出 | |

**接口清单**

| 分组 | 方法与路径 | 说明 |
|---|---|---|
| 公开 | `POST /api/v1/auth/register` | 注册，只能注册出普通用户 |
| 公开 | `POST /api/v1/auth/login` | 登录，返回 Token |
| 公开 | `GET /api/v1/activities` | 已发布的活动列表 |
| 公开 | `GET /api/v1/activities/:id` | 已发布的活动详情 |
| 用户 | `GET /api/v1/users/me` | 当前登录的用户 |
| 用户 | `GET /api/v1/orders` | 我的订单，游标分页 |
| 用户 | `GET /api/v1/orders/:id` | 我的订单详情 |
| 管理 | `POST /api/v1/admin/products` | 新建商品 |
| 管理 | `PUT /api/v1/admin/products/:id` | 修改商品 |
| 管理 | `GET /api/v1/admin/products`、`GET .../:id` | 商品列表、详情 |
| 管理 | `POST /api/v1/admin/activities` | 新建活动（草稿） |
| 管理 | `PUT /api/v1/admin/activities/:id` | 修改草稿 |
| 管理 | `POST /api/v1/admin/activities/:id/publish` | 发布 |
| 管理 | `POST /api/v1/admin/activities/:id/offline` | 下线 |
| 管理 | `GET /api/v1/admin/activities`、`GET .../:id` | 活动列表、详情（包含草稿和已下线） |

**完成标准**：设计文档写完，能回答下面的问题。

**思考**
1. （第 1 条）读 R13 和参考项目的 `scripts/init.sql`：`users` 和 `admin_users` 各自自增，所以 id=1 的管理员和 id=1 的用户同时存在。管理员的 Token 拿去调用户的订单接口，会被当成谁？
2. （第 3 条）如果把"进行中"存进数据库，谁负责在 20:00:00 整把它从"未开始"改过来？晚了 2 秒会怎样？
3. （第 5 条）一个 19 位的订单号作为数字放进 JSON，浏览器里的 JavaScript 读出来还是原来的数吗？算一下 2^53 是多少。
4. （第 7 条）假如订单状态定义成 `0-待支付 1-已支付 2-已取消`，代码某处忘了给 `Status` 赋值，数据库里会多出一条什么样的订单？

---

## Step 2：分层与错误处理（改造阶段 0 的代码）

阶段 0 的存档里留过一个伏笔：错误码放在 `internal/server` 包里，而 service 层需要返回这些错误。现在处理它。

阶段 1 结束时的目标结构（这一步只动前三个包，其余的在后面的 Step 里陆续出现）：

```
implement/internal/
├── errcode/        错误码。叶子包：不 import 任何内部包
├── response/       统一响应 Success / Fail
├── middleware/     全部 gin 中间件（阶段 0 的三个搬过来，Step 8 再加鉴权）
├── server/         http.Server 的生命周期、健康检查（已有）
├── model/          和表对应的结构体、状态常量（Step 5）
├── repository/     数据访问，GORM 只出现在这一层（Step 5）
├── auth/           密码哈希、Token 签发与校验（Step 6）
├── service/        业务规则（Step 7）
├── handler/        解析并校验请求 → 调 service → 写响应（Step 7）
└── router/         路由表：路径、handler、中间件怎么对应，只写在这一处（Step 7）
```

依赖方向只有一条规则：**上层可以 import 下层，反过来不行**。

```
cmd/api ─→ router ─→ handler ─→ service ─→ repository ─→ model
              └────→ middleware
errcode 和 model 在最底下，谁都可以 import 它们
```

- [ ] `errcode.go` 搬到 `internal/errcode`，`response.go` 搬到 `internal/response`，三个中间件搬到 `internal/middleware`，对应的测试跟着搬
- [ ] 给 `ErrorCode` 加一个 `Is(target error) bool` 方法，按 `Code` 比较
- [ ] 让未知错误进日志：`Fail` 收到不是 `*ErrorCode` 的错误时，对外照旧返回 `10000`，同时调用 `c.Error(err)` 把原始错误挂到 gin 的上下文上；`AccessLog` 在 `c.Next()` 之后把 `c.Errors` 一起打出来
  - 你的 lint 配置会把 `c.Error(err)` 和 `_ = c.Error(err)` 都报成 errcheck 问题（它的返回值类型实现了 `error`）。在 `.golangci.yml` 的 `errcheck` 设置里加 `exclude-functions: ["(*github.com/gin-gonic/gin.Context).Error"]` 即可
- [ ] 按号段补上本阶段用得到的错误码：用户名已存在、用户名或密码错误、Token 已过期、商品不存在、活动不存在、活动当前状态不允许此操作……商品用新号段 `5xxxx`

**完成标准**
- `make test`、`make lint` 通过
- 这条命令没有任何输出（errcode 是叶子包）：

  ```bash
  go list -f '{{join .Imports "\n"}}' ./internal/errcode | grep go-seckill/internal
  ```

- 有一个单测证明：handler 把 `fmt.Errorf("query user: %w", someErr)` 交给 `Fail` 后，响应体是 `{"code":10000,...}`，访问日志里能看到 `query user: ...` 的原文

**思考**
1. `errcode.ErrInvalidParam.WithMsg("用户名太短")` 得到的错误，拿去做 `errors.Is(err, errcode.ErrInvalidParam)`，现在的结果是 `false`。为什么？`errors.Is` 默认比较的是什么？加了 `Is` 方法之后，它是在什么时候被谁调用的？
2. 现在的 `Fail` 把未知错误一律变成 500，对客户端来说是对的。但如果数据库连不上，你打开日志能找到原因吗？在阶段 0 的代码里跟一遍，看这个 error 最后去了哪里。
3. `ErrorCode` 里带着 `HTTPStatus`。service 返回它，就等于 service 知道了 HTTP 的存在。另一种做法是 service 只返回纯业务错误，由 handler 负责翻译成状态码。两种做法各自的代价是什么？本项目推荐前一种，你同意吗？

**参考**：Go 官方博客《Working with Errors in Go 1.13》（`errors.Is` / `errors.As` / `%w` 的出处，不长）。

---

## Step 3：GORM 配置与时区

现在的 `mysql.New` 用的是 GORM 的默认配置。有两件事要在写业务之前改掉。

**第一件：日志和错误翻译**

- [ ] 打开 `TranslateError: true`
- [ ] 把 GORM 的日志接到项目的 slog 上：`logger.NewSlogLogger(你的 *slog.Logger, logger.Config{...})`
  - `LogLevel`：做成配置项（比如 `mysql.log_level`）。开发时调到 info，每一条 SQL 都会出现在日志里，这是你观察 GORM 最方便的办法
  - `SlowThreshold`：慢 SQL 阈值，先定 200ms
  - `ParameterizedQueries: true`：日志里的 SQL 只有 `?`，没有参数值
  - `IgnoreRecordNotFoundError: true`：查不到记录不当成错误

为什么要换。GORM 默认的 logger 直接往标准输出写带颜色的纯文本，不经过 slog，所以没有 `request_id`，混在 JSON 日志里是一行乱码。更要紧的是它在出错时会把参数值填进 SQL 一起打出来。注册时用户名重复，默认配置下日志是这样的：

```
Error 1062 (23000): Duplicate entry 'alice' for key 'users.uk_users_username'
[0.768ms] [rows:0] INSERT INTO `users` (`username`,`password_hash`,`role`,`created_at`,`updated_at`)
          VALUES ('alice','$2a$10$……',1,'2026-11-13 17:39:46.415','2026-11-13 17:39:46.415')
```

密码哈希进了日志。按上面的配置换掉之后，同一件事的日志是：

```
level=ERROR msg="SQL executed" service=api trace.duration=0.898ms
  trace.sql="INSERT INTO `users` (`username`,`password_hash`,`role`,`created_at`,`updated_at`) VALUES (?,?,?,?,?)"
  trace.rows=0 trace.error="duplicated key not allowed" request_id=req-abc-123
```

**第二件：时区**

- [ ] DSN 里的 `loc=Local` 换成写死的时区。推荐 `Asia/Shanghai`，和 docker-compose 里 MySQL 的 `+08:00` 保持一致。把这个约定写进设计文档
- [ ] 立一条规矩并遵守：判断"活动开始了没有""订单超时了没有"时，用 Go 里的 `now` 当参数传进 SQL，不在 SQL 里写 `NOW()`
- [ ] 可选：用 `go-sql-driver/mysql` 的 `mysql.Config{...}.FormatDSN()` 生成 DSN，替换手写的 `Sprintf`。顺带解决密码里带 `@`、`/` 时 DSN 解析出错的问题
- [ ] 可选：启动时比较 `SELECT NOW(3)` 和 `time.Now()`，相差超过几秒就打一条 WARN

**完成标准**
- 集成测试里有一条断言：用带 `request_id` 的 ctx 执行一条 SQL（日志级别 info），日志缓冲区里能找到同一个 `request_id` 和 `SQL executed`
- `make test-integration` 通过

**思考**
1. DSN 里的 `loc` 改变的是谁的行为：MySQL 服务器、这条连接的会话，还是 Go 这边的驱动？一个 `time.Time` 写进 `DATETIME` 列的时候，驱动把它变成了什么发给 MySQL？
2. 设想程序部署到一台 UTC 时区的机器上，DSN 还是 `loc=Local`，MySQL 还是东八区。此刻写入的 `expire_at` 和 MySQL 的 `NOW()` 相差多少？`WHERE expire_at <= NOW()` 会把一笔刚创建的订单判成什么？
3. `Asia/Shanghai` 这个名字对应的时区规则，Go 是从哪里查到的？程序放进一个极简的 Docker 镜像之后还查得到吗？（关键词：`time/tzdata`）
4. 全链路统一用 UTC 是另一种常见做法。和"全链路东八区"比，它的好处和麻烦各是什么？
5. 用户名重复是预期之内的业务结果，日志里却是一条 ERROR。要不要处理？能怎么处理？

---

## Step 4：表结构与迁移

四张表的列定义如下。建表语句自己写，写法沿用 `ping_test`（`ENGINE=InnoDB`、`utf8mb4`、每列带 `COMMENT`）。四张表都有 `created_at` 和 `updated_at`，类型 `DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3)`，其中 `updated_at` 再加 `ON UPDATE CURRENT_TIMESTAMP(3)`，下面不重复列出。

**users**

| 列 | 类型 | 说明 |
|---|---|---|
| `id` | `BIGINT UNSIGNED`，自增主键 | |
| `username` | `VARCHAR(32) NOT NULL` | 唯一索引 `uk_users_username` |
| `password_hash` | `VARCHAR(255) NOT NULL` | bcrypt 的结果是 60 个字符，留出换算法的余量 |
| `role` | `TINYINT UNSIGNED NOT NULL` | 1-普通用户 2-管理员。`CHECK (role IN (1, 2))` |

**products**

| 列 | 类型 | 说明 |
|---|---|---|
| `id` | `BIGINT UNSIGNED`，自增主键 | |
| `name` | `VARCHAR(128) NOT NULL` | |
| `description` | `VARCHAR(1024) NOT NULL DEFAULT ''` | |
| `price` | `BIGINT UNSIGNED NOT NULL` | 原价，单位分 |

**seckill_activities**

| 列 | 类型 | 说明 |
|---|---|---|
| `id` | `BIGINT UNSIGNED`，自增主键 | |
| `product_id` | `BIGINT UNSIGNED NOT NULL` | |
| `title` | `VARCHAR(128) NOT NULL` | |
| `seckill_price` | `BIGINT UNSIGNED NOT NULL` | 秒杀价，单位分 |
| `total_stock` | `INT UNSIGNED NOT NULL` | 活动总库存。发布后不变，是对账的基准 |
| `available_stock` | `INT UNSIGNED NOT NULL` | 可售库存。`CHECK (available_stock <= total_stock)` |
| `per_user_limit` | `INT UNSIGNED NOT NULL` | 每人限购件数。`CHECK (per_user_limit >= 1)` |
| `start_time`、`end_time` | `DATETIME(3) NOT NULL` | `CHECK (end_time > start_time)` |
| `status` | `TINYINT UNSIGNED NOT NULL` | 1-草稿 2-已发布 3-已下线。`CHECK (status IN (1, 2, 3))` |

**orders**

| 列 | 类型 | 说明 |
|---|---|---|
| `id` | `BIGINT UNSIGNED NOT NULL`，主键，**不自增** | 订单号。`CHECK (id > 0)` |
| `user_id`、`activity_id`、`product_id` | `BIGINT UNSIGNED NOT NULL` | |
| `product_name` | `VARCHAR(128) NOT NULL` | 下单时的商品名快照 |
| `quantity` | `INT UNSIGNED NOT NULL` | `CHECK (quantity >= 1)` |
| `unit_price` | `BIGINT UNSIGNED NOT NULL` | 下单时的成交单价快照，单位分 |
| `total_amount` | `BIGINT UNSIGNED NOT NULL` | 应付总额，单位分 |
| `status` | `TINYINT UNSIGNED NOT NULL` | 1-待支付 2-已支付 3-已取消。`CHECK (status IN (1, 2, 3))` |
| `expire_at` | `DATETIME(3) NOT NULL` | 支付截止时间 |
| `paid_at`、`cancelled_at` | `DATETIME(3) NULL` | |
| 索引 | `idx_orders_user_id_id (user_id, id)` | 用户的订单列表 |
| 索引 | `idx_orders_status_expire_at (status, expire_at)` | 阶段 5 的超时扫描 |

- [ ] 每张表一个迁移（`000002_create_users` ……），up 和 down 成对
- [ ] 决定 `ping_test` 的去留：留着，或者加一个新迁移把它删掉（集成测试要跟着改）。不要去修改已经存在的 `000001`
- [ ] `make migrate-up`，并确认 `seckill_test` 库也到了最新版本

**完成标准**
- 空库 `migrate-up` 后四张表都在；逐个 `migrate-down` 回到 0，再 `migrate-up`，全程无报错
- 手工插入一条违反 CHECK 的数据（比如 `end_time` 早于 `start_time`），被数据库拒绝，错误号是 3819

> 提示：在容器里用 `mysql` 客户端查中文，会显示成一串 `?`。数据没有坏，是容器里的客户端默认用了 latin1。连接时加 `--default-character-set=utf8mb4` 就正常了。

**思考**
1. 给订单表的 `user_id`、`activity_id`、`product_id` 建外键，能保证不出现"指向不存在的活动"的订单。这里为什么不建？先记住一个事实，阶段 2 你会亲手复现它的后果：有外键时，往子表插入一行，要给父表对应的那一行加共享锁。
2. `(user_id, id)` 这个索引是为哪条查询准备的？`id` 已经是主键，InnoDB 的二级索引本来就带着主键，为什么还要把它显式写进去？（Step 12 有实验）
3. 订单表现在只有两个二级索引，参考项目的订单表有 7 个（`scripts/init.sql` 77~83 行）。秒杀时每插入一笔订单，多一个索引意味着多做什么？"先建上，以后也许用得到"为什么不是好策略？
4. `available_stock` 是无符号整数。它等于 0 的时候执行 `SET available_stock = available_stock - 1`，结果是 -1、0，还是报错？`WHERE available_stock - 1 >= 0` 这种写法又会怎样？在 MySQL 里试一下。
5. CHECK 约束从 MySQL 的哪个版本才真正生效？在那之前写了会怎样？
6. `status` 列要不要写 `DEFAULT 1`？提示：GORM 插入时，结构体里没赋值的字段是"不出现在 INSERT 里"，还是"以 0 出现在 INSERT 里"？答案在 Step 5 的表里。

---

## Step 5：model、第一个 repository、测试地基

**model**

- [ ] `internal/model` 下四个结构体。字段名和列名按 GORM 的默认规则对应（`PasswordHash` ↔ `password_hash`），表名是结构体名的复数蛇形（`SeckillActivity` ↔ `seckill_activities`）。表结构由迁移文件管，所以结构体上基本不需要 `gorm:"..."` tag
- [ ] 不嵌入 `gorm.Model`（它自带软删除字段）
- [ ] 角色和状态用具名类型加常量，例如 `type ActivityStatus uint8`
- [ ] 类型约定：ID 用 `uint64`，金额用 `int64`，库存和数量用 `int`，可以为空的时间用 `*time.Time`

**repository**（先只做 `UserRepository`：创建、按用户名查、按 ID 查）

- [ ] 结构体里放 `*gorm.DB`，由构造函数传入。不用全局变量
- [ ] 每个方法的第一个参数是 `ctx`，方法里第一件事是 `r.db.WithContext(ctx)`
- [ ] 把 GORM 的错误翻译成自己的：`gorm.ErrRecordNotFound` → `repository.ErrNotFound`，`gorm.ErrDuplicatedKey` → `repository.ErrDuplicate`。其他错误用 `fmt.Errorf("…: %w", err)` 包一层再返回
- [ ] 每次查询都用一个**新声明的变量**接收结果

**测试地基**

- [ ] `internal/testutil`（带 `integration` build tag）：连接 `seckill_test`、清空指定表的辅助函数
- [ ] 每个集成测试在开头清空自己用到的表
- [ ] Makefile 的 `test-integration` 给 `go test` 加上 `-p 1`

**GORM 写法和它生成的 SQL**（本阶段会用到的都在这里，均为实测）

| 场景 | GORM | 生成的 SQL |
|---|---|---|
| 插入 | `db.Create(&u)` | ``INSERT INTO `users` (`username`,`password_hash`,`role`,`created_at`,`updated_at`) VALUES (?,?,?,?,?)``，自增 ID 从返回结果里回填，不会再发一条查询 |
| 按主键查 | `db.Take(&u, id)` | ``SELECT * FROM `users` WHERE `users`.`id` = ? LIMIT 1`` |
| 按条件查一条 | `db.Where("username = ?", name).Take(&u)` | ``SELECT * FROM `users` WHERE username = ? LIMIT 1`` |
| 同上，用 `First` | `db.First(&u, id)` | ``… WHERE `users`.`id` = ? ORDER BY `users`.`id` LIMIT 1`` |
| 部分更新 | `db.Model(&Product{}).Where("id = ?", id).Updates(map[string]any{…})` | ``UPDATE `products` SET `description`=?,`name`=?,`price`=?,`updated_at`=? WHERE id = ?`` |
| 条件更新 | `db.Model(&SeckillActivity{}).Where("id = ? AND status = ?", id, 1).Update("status", 2)` | ``UPDATE `seckill_activities` SET `status`=2,`updated_at`=? WHERE id = ? AND status = 1`` |
| 分页 | `db.Order("id DESC").Limit(20).Offset(40).Find(&list)` | ``SELECT * FROM `products` ORDER BY id DESC LIMIT 20 OFFSET 40`` |
| 计数 | `db.Model(&Product{}).Count(&n)` | ``SELECT count(*) FROM `products` `` |
| 游标分页 | `db.Where("user_id = ? AND id < ?", uid, cursor).Order("id DESC").Limit(21).Find(&list)` | ``SELECT * FROM `orders` WHERE user_id = ? AND id < ? ORDER BY id DESC LIMIT 21`` |

想看任意一条语句生成的 SQL，有两个办法：把 Step 3 的日志级别调到 info；或者用 `db.ToSQL(func(tx *gorm.DB) *gorm.DB { return tx.…… })`，它只生成不执行。

**GORM 的几个坑**（每一条都实测过，建议自己各试一次）

| 坑 | 现象 | 怎么办 |
|---|---|---|
| `First` 会排序 | 多一个 `ORDER BY 主键` | 按主键或唯一键查一条，用 `Take` |
| `Find` 查不到不报错 | `err == nil`，`RowsAffected == 0`，结果是零值 | 查一条用 `Take`，查不到会返回 `ErrRecordNotFound` |
| `Updates(结构体)` 跳过零值 | 想把描述清空、把价格改成 0，生成的 SQL 里根本没有这两列 | 用 `Updates(map)`，或者 `Select("列名", …).Updates(结构体)` |
| `Save` 不是"更新" | 它更新**所有**列；影响 0 行时还会再发一条 `INSERT … ON DUPLICATE KEY UPDATE` | 不用 `Save` |
| 零值照写不误 | 结构体里没赋值的 `Status` 以 0 写进 INSERT，列上的 `DEFAULT` 不起作用 | 状态从 1 开始 + CHECK 约束，让它报错而不是写进去 |
| 接收变量里有主键 | 变量里 `ID=1`，再用它接收 `Where("name = ?", …)` 的结果，SQL 变成 `… WHERE name = ? AND id = 1` | 每次查询用新变量 |
| 复用带条件的 `*gorm.DB` | `base := db.Where(a)`，第一次 `base.Where(b)`，第二次 `base.Where(c)`，第二次的 SQL 是 `a AND b AND c` | 要复用就写 `base := db.Where(a).Session(&gorm.Session{})` |
| `RowsAffected` 的含义 | MySQL 返回的是**实际改动**的行数。把一列更新成它原来的值，行存在，结果也是 0 | 记住它不等于"匹配到的行数" |

**完成标准**：`UserRepository` 的集成测试通过，覆盖三件事：创建后能按用户名查到；查一个不存在的用户名得到 `ErrNotFound`；插入重复的用户名得到 `ErrDuplicate`。

**思考**
1. `go test ./...` 默认会同时运行多个包的测试。两个包的测试都连 `seckill_test`，都在开头清表，会发生什么？`-p 1` 改变了什么？它和 `t.Parallel()` 管的是同一件事吗？
2. 另一种隔离办法：每个测试开一个事务，结束时回滚。它比清表快，但阶段 2 的并发测试用不了它，为什么？
3. 写入一个 `time.Now()` 再读出来，两个 `time.Time` 用 `==` 比较是 `false`。原因有两层：`DATETIME(3)` 只存到毫秒，多出来的位数 MySQL 是截掉还是四舍五入？`time.Time` 用 `==` 时除了时刻还比了什么？测试里应该怎么比？
4. repository 为什么要把 `gorm.ErrRecordNotFound` 换成自己的 `ErrNotFound`？如果不换，service 就得 import 哪个包？
5. `Order` 的 `ID` 忘了赋值就 `Create`，会发生什么？给它加上 `gorm:"primaryKey;autoIncrement:false"` 之后再试一次，结果有什么不同？表上的 `CHECK (id > 0)` 是在防哪一种情况？

**参考**：读参考项目的 `internal/repository/user.go` 第 10~19 行。`UserRepository` 是空结构体，直接用全局变量 `database.DB`。想一想：这样的 repository 怎么换成测试库？怎么让它的两次调用跑在同一个事务里？

---

## Step 6：密码与 Token（`internal/auth`）

- [ ] 密码：`Hash` 和 `Compare` 两个函数，基于 `golang.org/x/crypto/bcrypt`。cost 做成配置项，测试里用 `bcrypt.MinCost`
- [ ] Token：HS256 的 JWT。claims 里放用户 ID（`sub`）、角色、签发时间（`iat`）、过期时间（`exp`）、签发者（`iss`）
- [ ] 解析 Token 时三件事不能省：`jwt.WithValidMethods([]string{"HS256"})`、`jwt.WithExpirationRequired()`、`jwt.WithIssuer(…)`
- [ ] 配置：
  - `JWT_SECRET`：只从环境变量读，短于 32 字节启动失败，日志里脱敏
  - `auth.token_ttl`、`auth.issuer`、`auth.bcrypt_cost`
  - `.env.example` 同步更新
- [ ] 表驱动的单元测试，Token 至少覆盖：正常、已过期、payload 被改过、换了算法（HS512）、`alg=none`、没有 `exp`、别的密钥签的、乱码

本机实测的几个数字和事实，设计时用得上：

- bcrypt 算一次：cost=4 约 1ms，cost=10 约 50ms，cost=12 约 180ms。超过 72 字节的密码会返回 `ErrPasswordTooLong`
- golang-jwt 的默认行为：没有 `exp` 的 Token **能通过**校验；同一个密钥用 HS512 签出来的 Token **能通过**校验；`alg=none` 默认被拒绝；3 个字节的密钥也能签，库不检查密钥长度

**完成标准**：`go test ./internal/auth/...` 通过，并且测试里没有 `time.Sleep`。

**思考**
1. JWT 的内容是加密的吗？把你签出来的 Token 按 `.` 切开，第二段做 base64 解码看看。所以什么东西不能放进 Token？
2. 密码为什么不用"SHA-256 加盐"？bcrypt 的慢是缺点还是设计目标？同一个密码两次哈希的结果不一样，登录时是怎么比对上的？（看哈希串 `$2a$10$…` 的结构）
3. cost=10 时一个 CPU 核每秒只能算大约 20 次。秒杀开始前 10 万人同时登录会怎样？这和"登录之后用无状态 Token，下单接口鉴权不查库"的选择有什么关系？阶段 2 压测要准备 1 万个已登录的用户，该怎么办？
4. 请求参数上写 `binding:"max=72"` 能保证密码不超过 bcrypt 的上限吗？72 个汉字是多少字节？
5. Token 签发之后收不回来。用户改了密码，旧 Token 还能用多久？有哪些补救办法，各自的代价是什么？（本阶段接受这个限制，阶段 6 可选补上）
6. 为什么解析时要限定算法？查一下"JWT 算法混淆攻击"。
7. 测"Token 过期"，除了 `time.Sleep` 等它过期，还有什么办法？

**参考**：读参考项目的 `pkg/jwt/jwt.go` 第 65~68 行，对照上面"三件事不能省"，它省了哪几件？延伸阅读：OWASP 的 Password Storage Cheat Sheet；RFC 8725（JWT 最佳实践，只读第 3 节即可）。

---

## Step 7：注册与登录（第一条完整链路）

这一步把 handler → service → repository 第一次串起来，花的时间会比后面几步都长。

- [ ] `UserService` 依赖一个小接口，而不是具体的 `*UserRepository`。接口定义在 **service 包里**，只包含 service 用到的那几个方法
- [ ] 注册：
  - 用户名先规范化（去掉首尾空白、转小写），再校验。推荐规则：只允许小写字母、数字、下划线，长度 3~32
  - 密码长度按**字节**算，8~72
  - 直接插入，靠唯一索引判重：把 `repository.ErrDuplicate` 翻译成"用户名已存在"。**不要先查一遍再插**
  - 请求体不接受 `role`，注册出来的一律是普通用户
- [ ] 登录：用户不存在和密码错误，返回**同一个**错误码
- [ ] handler：请求和响应各有自己的结构体（DTO），不直接用 `model.User`。响应里不能有密码哈希
- [ ] 新建 `internal/router`，挂上 `POST /api/v1/auth/register` 和 `POST /api/v1/auth/login`
- [ ] 在 `cmd/api/main.go` 里组装：`*gorm.DB` → repository → service → handler → router。依赖都通过构造函数的参数传进去
- [ ] 测试：
  - service 的单元测试，用一个内存里的假实现代替 repository，不连数据库
  - 一个集成测试：20 个 goroutine 同时注册同一个用户名，恰好 1 个成功、19 个得到"用户名已存在"，没有其他错误

参数校验的几个实测行为，写 DTO 时用得上：

- `binding:"required"` 用在数字上，传 `0` 和没传一样会被拒绝。要区分"没传"和"传了 0"，字段得用指针
- `max=72` 数的是字符个数，不是字节数
- 请求体里多出来的字段（比如 `"role":"admin"`）会被悄悄忽略，前提是 DTO 里没有这个字段
- 校验失败的错误信息里，字段名默认是 Go 的字段名（`Username`），不是 JSON 里的名字（`username`）

**完成标准**
- 用 curl 注册、登录，拿到 Token；重复注册返回 HTTP 409 和对应的业务错误码
- 下面的命令没有任何输出（service 不直接依赖 gin 和 gorm）：

  ```bash
  go list -f '{{join .Imports "\n"}}' ./internal/service/... | grep -E 'gin-gonic|gorm.io'
  ```

**思考**
1. 读参考项目的 `internal/service/auth.go` 第 59~74 行：先 `FindByUsername`，没找到再 `Create`。两个请求同时注册同一个名字会怎样？用户名靠什么兜住了？邮箱呢？（看 `scripts/init.sql` 第 96 行，`idx_users_email` 是不是唯一索引）这种"先查、再根据结果写"的竞态，就是阶段 2 超卖问题的同一个形状。
2. 在 MySQL 里执行 `SELECT 'Alice' = 'alice', 'alice ' = 'alice', 'álice' = 'alice';`，三个结果都是 1。为什么？如果不限制用户名的字符集，会出什么问题？
3. 登录时，用户不存在可以立刻返回，密码错误要先算一次 bcrypt。两种情况的响应时间差了大约 50ms。攻击者能用这个差别做什么？怎么抹平它？
4. handler 里如果直接 `c.ShouldBindJSON(&user)`，其中 `user` 是 `model.User`，客户端在请求体里加一个 `"role": 2` 会怎样？
5. `UserService` 用的接口为什么定义在 service 包，而不是 repository 包？（Go 里常说的"接口属于使用方"）它给单元测试带来了什么？
6. 再读参考项目的 `internal/handler/auth.go` 第 55~70 行和 `internal/service/auth.go` 第 259 行。前者每个 handler 都手写一遍"哪个错误对应哪个状态码"，后者用 `strings.Contains(err.Error(), "redis: nil")` 判断错误类型。对照你 Step 2 的 `Fail` 和 `errors.Is`，它们各自的问题是什么？

---

## Step 8：鉴权中间件与路由

- [ ] `Auth()`：读取 `Authorization: Bearer <token>`，校验通过后把身份（用户 ID、角色）放进 `c.Request.Context()`；校验失败返回 401
- [ ] `RequireRole(role)`：身份里的角色不符，返回 403
- [ ] `internal/router`：三组路由。公开组不挂鉴权；用户组挂 `Auth` + `RequireRole(普通用户)`；管理组挂 `Auth` + `RequireRole(管理员)`
- [ ] 提供一个从 ctx 取身份的函数。handler 需要"当前用户是谁"时只从这里取，**不从请求参数里取**
- [ ] `GET /api/v1/users/me`
- [ ] 可选：让日志自动带上 `user_id`，做法和阶段 0 的 `request_id` 一样
- [ ] 表驱动的测试，覆盖鉴权矩阵：

| 情况 | 期望 |
|---|---|
| 没带 Token | 401 |
| `Authorization` 的格式不对 | 401 |
| Token 已过期 | 401 |
| 普通用户的 Token 访问管理接口 | 403 |
| 管理员的 Token 访问用户接口 | 403 |
| 角色相符 | 200 |

**完成标准**：上面的矩阵全部通过；所有路由都能在 `router` 包的一个文件里看全。

**思考**
1. 401 和 403 的区别是什么？"没登录"和"登录了但没权限"分别是哪一个？
2. 全局中间件的顺序是 RequestID → AccessLog → Recovery。`Auth` 应该排在 `AccessLog` 的里面还是外面？如果放在外面，鉴权失败的请求还有访问日志吗？
3. 读参考项目的 `internal/handler/order.go` 第 22 行：`userID := c.GetUint64("user_id")`。假如某条路由忘了挂鉴权中间件，这里拿到的 `userID` 是多少？会报错吗？接下来的查询会查出什么？你的"从 ctx 取身份"的函数应该怎么设计，才不会出现这种情况？
4. 参考项目的鉴权中间件每个请求都去 Redis 查一次用户有没有被禁用（`internal/middleware/jwt.go` 第 43 行）。这样做换来了什么，付出了什么？秒杀高峰时 Redis 抖一下，会发生什么？（R17）
5. 用自己的话把 R13 讲一遍：参考项目的 `JWTAuth`（13~59 行）和 `AdminJWTAuth`（61~99 行）差在哪一行？

---

## Step 9：创建管理员的命令行工具

管理员不能通过接口注册，用一个单独的小程序来建。

- [ ] `cmd/tool`（名字自定）下新建一个可执行程序，提供 `create-admin` 命令。它复用 `config`、`infra/mysql`、`auth`、`repository`
- [ ] 用户名从参数传入；密码从环境变量或者交互式输入读取，不放在命令行参数里
- [ ] 用户名已存在时给出明确提示，不覆盖已有账号
- [ ] Makefile 增加对应的目标

**完成标准**：建出来的管理员能登录，能访问 `/api/v1/admin/` 下的接口。

**思考**
1. 密码为什么不放在命令行参数里？（`history` 和 `ps` 分别能看到什么）
2. 参考项目的做法是 API 每次启动都执行 `EnsureAdminAccount`（`internal/service/admin.go` 第 52~86 行，在 `cmd/api/main.go` 第 73 行调用）：按配置里的账号密码创建管理员，密码对不上就改成配置里的。和单独的命令行工具比，各有什么好处和风险？
3. 这个工具和 API 是两个可执行程序，共用了哪些包？如果当初把数据库初始化的代码直接写在 `cmd/api/main.go` 里，现在会怎样？

---

## Step 10：商品管理

第一组完整的管理端接口，套路和 Step 7 相同：repository → service → handler → 路由 → 测试。

- [ ] 新建、修改、详情、列表四个接口，都在管理组下
- [ ] 校验：名称 1~128 个字符；价格是大于 0 的整数（分）；描述最长 1024
- [ ] 修改用 `Updates(map)` 或 `Select(…).Updates(…)`。修改一个不存在的 ID 返回 404
- [ ] 列表用 `page` / `page_size` 分页：`page` 从 1 开始，`page_size` 默认 20、最大 100
- [ ] 不提供删除接口

**完成标准**：集成测试覆盖：创建后能查到；把描述改成空字符串后，读出来确实是空的；修改不存在的 ID 返回 404；普通用户调用返回 403。

**思考**
1. 故意用 `Updates(结构体)` 把描述改成空字符串，打开 SQL 日志看生成的语句里有没有 `description`。
2. 修改时怎么判断"这个 ID 不存在"？`RowsAffected == 0` 一定意味着不存在吗？提交的内容和原来一模一样时会是多少？为什么用 `Updates(map)` 时通常不是 0？换成 `UpdateColumn` 呢？
3. 价格为什么不用 `float64`？在 Go 里运行 `a, b := 0.1, 0.2; fmt.Println(a + b)` 看看结果。再试试直接写 `fmt.Println(0.1 + 0.2)`，结果又不一样，为什么？客户端把价格传成 `99.5` 时，你的接口应该怎么反应？实际试一下。
4. `GET /admin/products?page=100000` 会生成 `OFFSET 1999980`。为了返回这 20 行，数据库实际要读多少行？（Step 12 会量出来）

---

## Step 11：秒杀活动管理

本阶段最重要的一步。这里第一次出现"带条件的更新 + 检查影响行数"，阶段 2 的扣库存和阶段 5 的订单状态机都是这个写法。

**管理端**

- [ ] 新建（状态为草稿）。业务校验放在 service：
  - 商品存在
  - `end_time` 晚于 `start_time`
  - 秒杀价大于 0，且不高于商品原价
  - `total_stock` ≥ 1；1 ≤ `per_user_limit` ≤ `total_stock`
  - `available_stock` 不由客户端传入，新建时等于 `total_stock`
- [ ] 修改：只有草稿能改。条件写进 SQL：`WHERE id = ? AND status = 草稿`。改 `total_stock` 时 `available_stock` 跟着改成同一个值
- [ ] 发布：草稿 → 已发布。`end_time` 已经过去的活动不能发布
- [ ] 下线：已发布 → 已下线
- [ ] 影响行数为 0 时要区分两种情况：活动不存在返回 404；活动存在但状态不允许，返回 409
- [ ] 列表和详情（能看到所有状态）

三个状态流转实测生成的 SQL：

```sql
-- 修改草稿
UPDATE `seckill_activities` SET `available_stock`=200,`per_user_limit`=2,`title`='双11 秒杀',`total_stock`=200,`updated_at`='…'
WHERE id = 1 AND status = 1;      -- 影响 1 行

-- 发布
UPDATE `seckill_activities` SET `status`=2,`updated_at`='…' WHERE id = 1 AND status = 1;   -- 影响 1 行
-- 再发布一次，或者发布之后再去改
UPDATE `seckill_activities` SET `status`=2,`updated_at`='…' WHERE id = 1 AND status = 1;   -- 影响 0 行

-- 下线
UPDATE `seckill_activities` SET `status`=3,`updated_at`='…' WHERE id = 1 AND status = 2;   -- 影响 1 行
```

**公开端**

- [ ] 列表和详情只返回已发布的活动。草稿和已下线的活动，详情接口返回 404
- [ ] 响应里带上商品名称和原价
- [ ] 响应里带上 `phase`（未开始 / 进行中 / 已结束），由当前时间和时间窗算出来；同时返回服务器的当前时间
- [ ] service 不直接调 `time.Now()`，而是持有一个可以替换的 `now func() time.Time`

**测试**

- [ ] 状态流转表：每个"当前状态 × 操作"的组合，结果要么成功，要么是 409
- [ ] 20 个 goroutine 同时发布同一个草稿，恰好 1 个成功
- [ ] `phase` 的边界：当前时间恰好等于 `start_time`、恰好等于 `end_time` 时分别是什么

**完成标准**：上面三组测试通过；已发布的活动用任何接口都改不了库存、限购、时间和价格。

**思考**
1. 如果发布写成"先查出活动，判断状态是草稿，把 `Status` 改成已发布，再 `Save`"，两个管理员一个点发布、一个点保存修改，会出现什么结果？和条件更新的写法比，差别在哪里？这正是 R04 和 R07 的形状。
2. 影响行数为 0 之后再查一次，用来区分 404 和 409。这次查询和刚才的 `UPDATE` 不是原子的，有没有可能判断错？判断错了后果严重吗？
3. 活动时间是 `[start, end)` 还是 `[start, end]`？20:00:00.000 整到达的请求算不算已经开始？定下来写进设计文档，阶段 2 和阶段 3 的 Lua 脚本都要和它一致。
4. 为什么发布之后不让改库存？设想阶段 3：库存 100 已经预热进 Redis 并卖出了 30 件，这时管理员把 `total_stock` 改成 50。（R06、R11）
5. 活动列表要带商品信息。`Preload("Product")` 和 `Joins("Product")` 各发几条 SQL？如果写成"查出 20 个活动，循环里逐个查商品"，一共是几条？打开 SQL 日志各试一次。
6. 页面上的倒计时应该以谁的时钟为准？这就是响应里要带服务器时间的原因。
7. "秒杀价不高于原价"这条规则，写在 `binding` tag 里、写在 service 里、写成数据库的 CHECK，哪些做得到？各自适合管什么样的规则？

**参考**：读参考项目的 `internal/model/seckill_activity.go` 和 `internal/service/activity.go`。它的 `status` 有"未开始 / 进行中 / 已结束 / 停用"四个值，同时还有 `start_time` 和 `end_time`。找一找是哪段代码在维护 `status`，它和时间窗不一致的时候以谁为准。

---

## Step 12：订单查询

本阶段不创建订单，测试数据由测试代码直接写进数据库。

- [ ] `OrderRepository`：`Create`（测试和阶段 2 用）、按"订单号 + 用户 ID"查一条、按用户 ID 游标分页查列表
- [ ] `GET /api/v1/orders/:id`：查询条件里同时带上订单号和当前用户的 ID。查不到就是 404，不区分"不存在"和"不是你的"
- [ ] `GET /api/v1/orders?cursor=&limit=`：按订单号倒序。`cursor` 是上一页最后一条的订单号。每次多查一条，用来判断还有没有下一页
- [ ] 订单号在 JSON 里输出为字符串（`json:"id,string"`）
- [ ] 可选：管理端的订单列表

**实验：索引和分页**（做完把结果记进设计文档）

- [ ] 在 `seckill_test` 库里造 2 万条订单：200 个用户，每人 100 单。可以用 GORM 的 `CreateInBatches` 写个小程序，也可以用 `stage1-guide-notes.md` 里的 SQL
- [ ] 对下面几条语句分别执行 `EXPLAIN`，记下 `type`、`key`、`rows`、`Extra`（语句里的用户 ID 和订单号按你造的数据调整）：

  ```sql
  SELECT * FROM orders WHERE user_id = 77 ORDER BY id DESC LIMIT 21;
  SELECT * FROM orders WHERE user_id = 77 AND id < 115000 ORDER BY id DESC LIMIT 21;
  SELECT * FROM orders IGNORE INDEX (idx_orders_user_id_id) WHERE user_id = 77 ORDER BY id DESC LIMIT 21;
  SELECT * FROM orders WHERE user_id = 77 ORDER BY created_at DESC LIMIT 21;
  SELECT * FROM orders ORDER BY id DESC LIMIT 20 OFFSET 15000;
  ```

- [ ] `EXPLAIN` 里的 `rows` 是估算值。想知道一条语句**实际**读了多少行，在同一个会话里依次执行 `FLUSH STATUS;`、要测的语句、`SHOW SESSION STATUS LIKE 'Handler_read%';`。用这个办法量一下第 3 条和第 5 条

**完成标准**
- 集成测试覆盖：用户只能看到自己的订单；用别人的订单号访问返回 404；连续翻页能不重不漏地取完所有订单
- 设计文档里有上面 5 条语句的 `EXPLAIN` 结果和你的解读

**思考**
1. 访问别人的订单，返回 404 还是 403？各自向对方透露了什么？
2. "先按订单号查出来，再比较 `order.UserID` 和当前用户"与"`WHERE id = ? AND user_id = ?`"，结果一样。为什么推荐后者？
3. 第 3 条语句的 `EXPLAIN` 显示 `rows=21`，实际却读了几千行。为什么估算差这么多？它是怎么执行的？
4. 第 4 条比第 1 条只换了排序的列，`Extra` 里多出了 `Using filesort`。为什么按 `id` 排序不需要额外排序，按 `created_at` 就需要？
5. 第 5 条为了返回 20 行读了多少行？游标分页为什么没有这个问题？它的代价是什么（想想"跳到第 50 页"）？
6. 进阶：把索引换成只有 `(user_id)` 一列，重新 `EXPLAIN` 第 2 条，看执行计划变成了什么。回到 Step 4 的思考题 2，现在能回答了吗？做完把索引改回来。
7. 订单里为什么要存一份 `product_name` 和 `unit_price`，而不是需要时用 `product_id` 去查？商品改名、改价之后，三个月前的订单应该显示什么？

**参考**：读参考项目的 `internal/repository/order_repo.go` 第 80~95 行。`FindByUserID` 一次返回用户的全部订单，按 `create_time` 排序。对照你刚做的实验，它有哪两个问题？延伸阅读：Use The Index, Luke 网站上关于"不用 OFFSET 的分页"的那一章。

---

## Step 13：端到端验收

- [ ] 一个 `.http` 文件（或 curl 脚本），按顺序走完：注册 → 登录 → 管理员登录 → 建商品 → 建活动 → 修改草稿 → 发布 → 公开列表和详情 → 查看我的订单
- [ ] 一个集成测试走同样的流程：`httptest` 加真实的 MySQL，从路由进去，一直到数据库
- [ ] 回头检查 Step 8 的鉴权矩阵，确认覆盖了本阶段新增的所有管理接口

**完成标准**：`.http` 文件从头到尾执行无误；`make test`、`make test-integration`、`make lint` 全部通过。

---

## Step 14：收尾

- [ ] 更新 README：目录结构、接口清单、新增的环境变量（`JWT_SECRET`）、怎么创建管理员
- [ ] 按 README 从零走一遍：`make docker-down-v` → `make docker-up` → `make migrate-up` → 创建管理员 → `make run` → 执行 `.http` 文件
- [ ] 打附注标签 `stage-1`

---

## 阶段 1 总验收

- [ ] 从空环境按 README 操作，能跑通"注册 → 登录 → 管理员建商品和活动 → 发布 → 公开查询"
- [ ] `make test`、`make test-integration`、`make lint` 全部通过，集成测试连续跑多次结果一致
- [ ] 依赖方向干净：`errcode` 不依赖任何内部包；`service` 不直接 import gin 和 gorm
- [ ] 没带 Token、角色不符、访问别人的订单，这三类请求都有自动化测试，并且都被拒绝
- [ ] 已发布的活动改不了库存、限购、时间和价格
- [ ] 日志里的每条 SQL 都带 `request_id`，并且没有参数值
- [ ] 代码里没有全局的数据库变量，没有 `Save`，没有被忽略的 error
- [ ] 设计文档记录了每个设计决定的理由，以及 Step 12 的实验结果

完成后可以让 AI 助手做三件事：review `implement/` 的代码；在 `guide/questions/` 下生成阶段 1 的问题库；生成阶段 1 的面试问答（问题和答案分成两个文件）。之后进入阶段 2。
