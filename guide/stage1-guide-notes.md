# 阶段 1 指导要点（写给负责指导的 AI 助手）

> 编写日期：2026-11-13
> 配套文档：`guide/stage1-todolist.md`（学习者照着做的清单）、`guide/roadmap.md`
> **给学习者的提醒**：这份文档里有思考题的答案和参考建表语句。建议做完对应的 Step 再看。

---

## 0. 怎么用这份文档

学习者每次会带着一个 Step 编号来找你。先读 todolist 里对应的 Step，再读本文对应的小节。本文回答三个问题：这一步要讲清哪些概念，有哪些已经验证过的事实可以直接引用，思考题的要点是什么。

**学习者的情况**（来自 `AGENTS.md`）
- 刚开始学 Go，之前做过一个 CRUD 小项目，对 Go、Gin、MySQL 有基本了解
- 阶段 0 是在 AI 助手逐步指导下完成的，对话存档在 `guide/stage0-archive/`，可以参考当时的讲解深度

**指导时的约定**
- `implement/` 和 `reference/` 对 AI 助手只读。代码写在对话里，由学习者自己敲进去
- 凡是给出 GORM 代码，都要同时给出它生成的 SQL
- 篇幅小的原理直接讲；篇幅大的只引出概念，用问题引导学习者去查，或者推荐资料
- 讲之前先让学习者说出自己的预测，再揭晓结果
- 每个 Step 结束提醒提交一次。学习者的提交信息是 `feat(scope): 中文描述` 的风格

**本文里结论的可信度**
- 标 ✔ 的条目是实测结果。环境：MySQL 8.0.46（官方镜像，`utf8mb4_unicode_ci`，`+08:00`，默认的可重复读）、GORM v1.31.2、`gorm.io/driver/mysql` v1.6.0、`go-sql-driver/mysql` v1.8.1、Gin v1.12.0、validator v10.30.1、golang-jwt v5.3.1、`x/crypto` v0.48.0、Go 1.26.10、golangci-lint v2.15.0。都是在一次性的容器和临时目录里跑的，没有碰 `implement/` 和开发库
- 没有标 ✔ 的是常识性说明或设计建议，讲之前请自己判断

---

## 1. 阶段 0 留下的状态（2026-11-13 查看）

- `implement/` 在 tag `stage-0`（提交 `94e3f00`），工作区干净
- `make test`、`make lint` 通过；**`make test-integration` 稳定失败** ✔（见 Step 0）
- 数据库：`seckill_test` 在迁移版本 1，`ping_test` 里残留 1 行（id=1，2026-11-12 23:16 写入）。主库 `seckill` 的 `schema_migrations` 是空的，没有任何业务表，需要先 `make migrate-up`
- `guide/stage0-todolist.md` 的 Step 14 和总验收没有打勾，但 README 已完成、tag 已打。其中"`make test-integration` 通过"这一项实际不成立

---

## 2. 各 Step 的讲解要点

### Step 0：阶段 0 收尾

**集成测试为什么失败** ✔

`integration_test.go` 里，关闭 MySQL 和 Redis 用的是 `defer`（第 51、62 行），删除测试数据用的是 `t.Cleanup`（第 73 行）。执行顺序是：测试函数返回 → 它的 `defer` 按后进先出执行 → 然后才轮到 `t.Cleanup` 注册的函数（也是后进先出）。所以删除数据时连接池已经关了。

用一个最小例子复现过，输出顺序如下：

```
现在的写法:  test body done → db.Close() → cleanup failed: sql: database is closed
改成全用 t.Cleanup（先注册 Close，后注册删除）:  test body done → DELETE row ok → db.Close()
```

修法：资源的关闭也用 `t.Cleanup` 注册，在创建资源之后立刻注册。后注册的数据清理会先执行。

**为什么以前没暴露**：提交 `c6b6c2b` 之前，清理代码是 `_ = db.….Delete(…).Error`，错误被吞掉了。测试"通过"，但每跑一次就在测试库里留下一行。`errcheck` 的 `check-blank` 逼着处理这个错误之后，问题才显形。这是 `errcheck` 价值的一个现成例子，值得让学习者自己把因果说出来。

顺带可以提一句：`stage-0` 的 tag 是在这个测试失败的状态下打的，说明打 tag 前没有重新跑完整的检查。阶段 7 的 CI 就是为了不靠人记。

**其他两项**
- `/debug/slow` 在 `internal/server/server.go:73` 注册，`health.go:91` 实现
- README 4.1 的变量名和 Makefile 对不上（`MIGRATE_BIN` 对 `MIGRATE`）。golangci-lint v2 的安装路径是 `github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.x.y`

---

### Step 1：设计先行

这一步不写代码，重点是让学习者把每个决定和后面的阶段联系起来。九条决定的理由：

| # | 要点 |
|---|---|
| 1 单用户表 | 参考项目 `users`、`admin_users` 各自自增。管理员 Token 里的 `user_id=1` 拿到用户接口，中间件只看 `UserID` 不看角色（`middleware/jwt.go:13-59`），就被当成了 id=1 的普通用户。单表让 ID 不可能重复，再加上角色校验，是两道防线 |
| 2 库存归活动 | 参考项目的库存在 `goods.stock`，活动没有自己的库存。后台改商品库存直接覆盖绝对值（R11），多个活动共用一个池子时对账公式也说不清 |
| 3 状态只存人为状态 | "进行中"是时间的函数。存下来就需要有人在准点去改它，改晚了、改之后时间窗又被调整，都会出现两份事实不一致 |
| 4 发布即冻结 | 为阶段 3 准备的不变量。预热后 Redis 和 MySQL 各有一份库存，如果 MySQL 这份还能改，就得设计同步协议（R06 正是这里出的事） |
| 5 订单号是主键 | 少维护一棵唯一索引；重复消息直接表现为主键冲突。代价是需要自己的发号器（阶段 2），以及 JSON 里必须用字符串 |
| 6 整数分 | 浮点不能精确表示 0.1；DECIMAL 可以，但 Go 侧要引入 decimal 库。整数分最简单 |
| 7 状态从 1 开始 | ✔ GORM 会把没赋值的字段以 0 写进 INSERT，列的 DEFAULT 不生效。0 如果是合法状态，这个 bug 就是静默的 |
| 8 不建外键 | ✔ 见附录 C 第 3 条。另外不提供删除接口，也就不会出现指向不存在数据的行 |
| 9 订单号用字符串 | ✔ `7203685477580123456` 经过 float64 变成 `7203685477580123136`。2^53 = 9007199254740992 |

思考题 4 的答案：会多出一条"待支付"的订单，而且没有任何报错。

---

### Step 2：分层与错误处理

**要讲清的概念**
- Go 禁止循环 import，包的依赖必须是有向无环图。阶段 0 存档 `step8-http-server.md` 的结尾和问题库 `engineering/stage0.md` 的 Q10 已经铺垫过，可以直接接上
- 叶子包：被很多包依赖的包，自己不应该依赖内部的其他包
- 错误在层与层之间传递时的三个动作：**包装**（`fmt.Errorf("…: %w", err)` 加上下文）、**翻译**（repository 把 GORM 的错误换成自己的，service 把它换成业务错误码）、**映射**（`Fail` 把错误码变成 HTTP 响应）

**`errors.Is` 与 `WithMsg`** ✔

`errors.Is` 沿着 `Unwrap` 链逐个比较，默认用 `==`。`ErrorCode` 是指针，`==` 比的是地址；`WithMsg` 返回的是新对象，地址不同，所以结果是 `false`。如果链上的某个错误实现了 `Is(target error) bool`，`errors.Is` 会调用它。实测：

```
errors.Is(ErrInvalidParam.WithMsg("…"), ErrInvalidParam)             = false
errors.Is(fmt.Errorf("wrap: %w", ErrInvalidParam), ErrInvalidParam)  = true
加上按 Code 比较的 Is 方法后，第一种                                   = true
```

实现时对 `target` 做类型断言要带 `, ok`，否则过不了 lint 的 `check-type-assertions`。

**让未知错误进日志** ✔

现状：`response.go` 里未知错误被换成 `ErrInternal` 后直接丢弃，日志里没有任何痕迹。两种改法：

1. `Fail` 里调用 `c.Error(err)`，`AccessLog` 在 `c.Next()` 之后读 `c.Errors`。实测可行，好处是访问日志一行里同时有状态码和原因
2. `Fail` 里直接用 `logger.FromContext(ctx)` 打一条 ERROR

选第 1 种时有个坑：`c.Error` 的返回值类型是 `*gin.Error`，它实现了 `error`，所以 errcheck 会报错，写成 `_ = c.Error(err)` 也会报（`check-blank`）。实测在 `.golangci.yml` 里加下面的配置后通过：

```yaml
linters:
  settings:
    errcheck:
      exclude-functions:
        - (*github.com/gin-gonic/gin.Context).Error
```

**搬迁时会遇到的事**：`server_test.go` 里混着响应、错误码、中间件、健康检查四类测试，都依赖同一个 `newTestServer`，需要拆开。这是体会"测试也有依赖关系"的机会，不用替学习者拆。

**思考题 3**（service 该不该知道 HTTP 状态码）：推荐的做法是务实的折中，一处定义、少写一层映射；代价是 service 如果以后要被 gRPC 或消息消费者调用，`HTTPStatus` 就成了多余的东西。让学习者说出取舍即可，不必改。

---

### Step 3：GORM 配置与时区

**`TranslateError`** ✔

`gorm.io/driver/mysql` 的 `error_translator.go` 只翻译三个错误号：1062 → `gorm.ErrDuplicatedKey`，1451 和 1452 → `gorm.ErrForeignKeyViolated`。其他错误原样返回。

- 开启后：`errors.Is(err, gorm.ErrDuplicatedKey)` 为 `true`，错误文本是 `duplicated key not allowed`
- 不开启：要用 `errors.As(err, &mysqlErr)` 再判断 `mysqlErr.Number == 1062`
- 注意：翻译是**替换**，不是包装。翻译之后拿不到"撞的是哪个唯一键"。一张表有多个唯一键时要留意；本阶段每张表最多一个，没有影响

**slog logger** ✔

`logger.NewSlogLogger` 在 `gorm.io/gorm/logger/slog.go`。它的 `Trace` 逻辑：有错误（且不是被忽略的 not found）打 ERROR；超过慢阈值打 WARN；否则在 `LogLevel >= Info` 时打 INFO。它把 ctx 传给 `Handler().Handle(ctx, record)`，所以项目里 `contextHandler` 的 `request_id` 会自动带上。属性都在 `trace` 组下：`trace.duration`、`trace.sql`、`trace.rows`、`trace.error`。

实测输出（text 格式）：

```
level=INFO  msg="SQL executed" service=api trace.duration=0.349ms trace.sql="SELECT * FROM `users` WHERE username = ? LIMIT ?" trace.rows=1 request_id=req-abc-123
level=ERROR msg="SQL executed" service=api trace.duration=0.898ms trace.sql="INSERT INTO `users` (…) VALUES (?,?,?,?,?)" trace.rows=0 trace.error="duplicated key not allowed" request_id=req-abc-123
```

对照默认 logger 在同样场景下的输出：带 ANSI 颜色的纯文本，包含完整参数值（密码哈希可见），`record not found` 也按错误打印。

思考题 5（用户名重复打了 ERROR）：`Trace` 被调用时错误已经翻译过了（日志里是 `duplicated key not allowed`），所以可以写一个包住 `logger.Interface` 的类型，在 `Trace` 里对 `gorm.ErrDuplicatedKey` 降级。也可以接受现状。让学习者自己权衡。

**时区** ✔

三个层次：Go 进程的时区、MySQL 会话的时区、列类型（`DATETIME` 不带时区信息）。DSN 的 `loc` 只影响**驱动**：写入时把 `time.Time` 换算成 `loc` 时区的墙上时间字符串发出去，读取时把字符串按 `loc` 解释。它不改变会话的 `time_zone`。

实测：MySQL 是 `+08:00`，DSN 写 `loc=UTC`：

```
同一瞬间   库里 expire_at 的字面值 = 2026-11-13 09:39:44.953    MySQL 的 NOW(3) = 2026-11-13 17:39:44.963
SELECT expire_at <= NOW(3) - INTERVAL 1 HOUR   →  1      （数据库认为它一小时前就过期了）
用 loc=Asia/Shanghai 的连接读回来，与写入时刻相差 -8h
```

其他实测事实：
- DSN 里加 `time_zone=%27%2B08%3A00%27` 可以设置会话时区，`@@session.time_zone` 变成 `+08:00`。这样应用自己同时控制两边，不依赖服务器的默认值。`loc=UTC` 配 `time_zone='+00:00'` 同样可行
- `loc` 写了不存在的时区，`sql.Open` 直接返回 `unknown time zone …`
- 启动自检可行：`SELECT NOW(3)` 扫描进 `time.Time`，和 `time.Now()` 相减。两边一致时差值是 0s，`loc` 配错时差值是 -8h

思考题 3：Go 从系统的 zoneinfo 目录查时区规则。scratch、distroless 或没装 tzdata 的 alpine 镜像里没有这个目录，`time.LoadLocation` 会失败。解法是 `import _ "time/tzdata"`，把时区数据编进二进制。

思考题 4：UTC 的好处是不依赖部署地点、没有夏令时问题、不需要 tzdata；麻烦是直接看库里的数据要心算 8 小时，以及所有输入输出边界都要做转换。推荐东八区只是因为 docker-compose 已经这么配了，改动最小。

**时间精度** ✔（Step 5 的思考题 3 会用到）

`DATETIME(3)` 对多余的小数位是**四舍五入**，不是截断。写入 `12:00:00.9996`，读回来是 `12:00:01.000`。而 GORM 日志里显示的是 `'… 12:00:00.999'`——日志里的值是截断的，和实际存进去的不一样。GORM 自动填的 `created_at`、`updated_at` 已经是毫秒精度，内存里的值和库里一致。自己构造的时间（比如 `expire_at`）写库前先 `Truncate(time.Millisecond)`。

---

### Step 4：表结构与迁移

参考建表语句在附录 A，已经用 golang-migrate 对 MySQL 8.0.46 验证过 up 和 down ✔。学习者写完自己的版本后再拿出来对照，不要一开始就给。

**思考题要点**

1. 不建外键：✔ 附录 C 第 3 条是直接证据。补充理由：以后分库分表时外键没法跨库；迁移和删表的顺序受限。代价是数据库不再替你保证引用完整性，本项目靠"不提供删除接口 + service 校验"来保证
2. 显式的 `(user_id, id)`：✔ 见 Step 12 的实验数据。InnoDB 的二级索引确实带主键，等值查询加 `ORDER BY id` 时两种索引的计划相同。差别出现在 `id < ?` 的范围条件上：只有 `(user_id)` 时，优化器走了 `index_merge` 加 `filesort`；显式的联合索引走的是 `range` 加反向索引扫描。物理上显式写出来不多占空间
3. 索引的代价：每个二级索引是一棵独立的 B+ 树，插入一行要多维护一棵树、多写一份 redo。参考项目订单表有 7 个二级索引
4. ✔ 无符号减法越界报错 1690：`BIGINT UNSIGNED value is out of range in '(…available_stock - 1)'`。写在 `SET` 里和写在 `WHERE` 里都报这个错。正确的条件是 `available_stock >= ?`，库存不够时影响 0 行、不报错。可以告诉学习者：无符号列在这里是最后一道保险，代码写错了是报错而不是把库存扣成负数
5. CHECK 从 8.0.16 开始真正生效。之前的版本语法能通过，但会被忽略。违反时的错误号是 3819 ✔
6. ✔ 不需要 `DEFAULT`。GORM 把零值原样写进 INSERT，列的默认值用不上。实测：`status` 没赋值时，INSERT 里是 `status=0`，被 `chk_activities_status` 以 3819 拒绝

**其他可以讲的**
- `VARCHAR(32)` 的 32 是字符数。utf8mb4 下一个字符最多 4 字节
- 一个迁移文件只放一个 DDL（问题库 `mysql-gorm/stage0.md` 的 Q13 做过实验：MySQL 的 DDL 会隐式提交，中途失败会留下 dirty 状态）
- `ping_test` 的去留：推荐加一个新迁移把它删掉，同时把 readyz 的集成测试改成用 `users` 表。"已经执行过的迁移不改，用新迁移修正"是团队里的通行做法，这里正好练一次。直接改 `000001` 在这个项目里也行得通（没有部署过），让学习者自己选并说出理由
- ✔ 容器里的 `mysql` 客户端默认字符集是 latin1（`@@character_set_client` 等三个变量都是），中文会显示成 `?`。数据是对的，用 `HEX()` 能看到正确的 UTF-8 字节。加 `--default-character-set=utf8mb4` 即可

---

### Step 5：model、第一个 repository、测试地基

**GORM 的默认约定**
- 字段名转蛇形作为列名，结构体名转蛇形复数作为表名。四个结构体不写任何 tag 就能和附录 A 的表对上 ✔
- 名为 `ID` 的字段是主键；`CreatedAt`、`UpdatedAt` 在 `Create` 和 `Updates` 时自动填当前时间
- `Update`、`Updates` 自动带上 `updated_at` 的前提是模型里有 `UpdatedAt` 字段。`UpdateColumn`、`UpdateColumns` 不带。表上的 `ON UPDATE CURRENT_TIMESTAMP(3)` 是给这类情况和原生 SQL 兜底的
- 关联：`SeckillActivity` 上加一个 `Product *Product` 字段，配合已有的 `ProductID`，`Preload` 和 `Joins` 就能用。不需要数据库里有外键 ✔

**todolist 里那张"坑"表的实测细节** ✔

```
First(&x, 1)                 SELECT * FROM `users` WHERE `users`.`id` = 1 ORDER BY `users`.`id` LIMIT 1
Find(&xs) 查不到              err = nil, RowsAffected = 0, len = 0

Updates(Product{Name:"iPhone 2", Description:"", Price:0})
                             UPDATE `products` SET `name`='iPhone 2',`updated_at`='…' WHERE id = 1
Updates(map{…})              UPDATE `products` SET `description`='',`name`='iPhone 3',`price`=0,`updated_at`='…' WHERE id = 1
Select("name","description","price").Updates(Product{…})
                             UPDATE `products` SET `name`='iPhone 4',`description`='',`price`=0,`updated_at`='…' WHERE id = 1
Updates 忘了写 Where          err = WHERE conditions required

Save(&Product{ID: 9999, …})  先发 UPDATE … SET 所有列 … WHERE `id` = 9999（影响 0 行）
                             再发 INSERT … ON DUPLICATE KEY UPDATE …
                             → 凭空多出一行 id=9999
Save 一个只填了部分字段的对象   UPDATE 里带着 `created_at`='0000-00-00 00:00:00'，严格模式下报 1292

变量 x 的 ID 已经是 1，再 First(&x, 99999)
                             SELECT * FROM `users` WHERE `users`.`id` = 99999 AND `users`.`id` = 1 …

base := db.Model(&Product{}).Where("price > ?", 0)
base.Where("name = ?", "a").Find(&l1)
base.Where("name = ?", "b").Find(&l2)    第二条: … WHERE price > 0 AND name = 'a' AND name = 'b'
加上 .Session(&gorm.Session{}) 之后      第二条: … WHERE price > 0 AND name = 'b'

UpdateColumn("name", "same") 连续两次     第二次 RowsAffected = 0
Updates(map{"name": "same"})             RowsAffected = 1（因为 updated_at 变了）
DSN 加 clientFoundRows=true              相同的值也返回 1（语义变成"匹配到的行数"）
```

`clientFoundRows` 知道有这个开关即可，不建议现在打开。状态流转和扣库存的 `SET` 一定会改动列值，两种语义下结果相同。

**`Order.ID` 没赋值**（思考题 5）✔

| 写法 | `ID=0` 时 `Create` 的结果 |
|---|---|
| 字段不带 tag | GORM 不把 `id` 放进 INSERT，MySQL 报 1364 `Field 'id' doesn't have a default value` |
| `gorm:"primaryKey;autoIncrement:false"` | `id=0` 被写进 INSERT 并且**成功**，表里多出一个 0 号订单 |
| 同上，表上有 `CHECK (id > 0)` | 被 3819 拒绝 |

第二种情况的后果要讲透：第一笔这样的订单成功了，第二笔撞主键。到了阶段 4，主键冲突会被当成"重复消息，已经处理过"，于是静默丢单。所以推荐不加 tag，并保留 CHECK。显式给了 `ID` 时，不带 tag 也能正常插入。

**测试隔离**
- `go test ./...` 默认并行运行多个包的测试程序（`-p` 默认等于 CPU 核数）。两个包同时清表、写同一个库，结果是随机失败。`-p 1` 让包与包串行。`t.Parallel()` 管的是同一个包内的测试函数，是另一回事
- 事务回滚式的隔离：被测代码自己开事务并提交时不好用；并发测试更用不了——一个事务绑定在一条连接上，多个 goroutine 各用各的连接，互相看不到未提交的数据
- `TRUNCATE` 会隐式提交，并重置自增计数器

**时间比较**（思考题 3）：一层是上面说的四舍五入 ✔；另一层是 `time.Time` 用 `==` 时还比较了时区指针和单调时钟读数。应该用 `Equal`，或者断言两个时间之差小于某个值。

**哨兵错误的翻译**（思考题 4）：不换的话 service 要 `errors.Is(err, gorm.ErrRecordNotFound)`，就得 import `gorm.io/gorm`，Step 7 的那条依赖检查过不了。

---

### Step 6：密码与 Token

**bcrypt** ✔

```
cost=4   生成 ≈ 1.25ms    比对 ≈ 1.13ms
cost=10  生成 ≈ 49ms      比对 ≈ 46ms
cost=12  生成 ≈ 179ms     比对 ≈ 182ms
同一个密码两次哈希的结果不同
73 字节的密码 → bcrypt: password length exceeds 72 bytes（errors.Is(err, bcrypt.ErrPasswordTooLong) 为 true）
25 个汉字 = 75 字节 → 同样报错
密码不对 → errors.Is(err, bcrypt.ErrMismatchedHashAndPassword) 为 true
```

哈希串 `$2a$10$` 后面依次是 22 个字符的盐和 31 个字符的哈希值。盐就在串里，所以比对时能用同一个盐重算一遍。

要讲清的原理：密码哈希要的是"慢"，目标是让离线暴力破解的代价高到不划算；SHA-256 快，适合校验完整性，不适合存密码。cost 每加 1，耗时翻倍。

思考题 3 的展开：cost=10 时单核每秒约 20 次登录。这引出两个结论。一是登录之后必须用无状态的凭证，不能每个请求都回头验密码。二是阶段 2 压测前要预先生成 Token：造用户时用 `bcrypt.MinCost` 或者共用一个哈希，Token 直接用签发函数生成，不走登录接口。参考项目有 `test/tools/setup_users` 和 `gen_tokens`，是同一个思路。

思考题 4：✔ validator 的 `max=72` 数的是字符。72 个汉字通过了校验，但它是 216 字节，到 bcrypt 才报错。如果这个错误没有被 service 单独处理，客户端会收到 500。字节长度要在 service 里用 `len(password)` 检查。

**JWT** ✔

```
payload 可以直接 base64 解码: {"uid":7,"role":"user","exp":1794567350,"iat":1794563750}

                                              默认选项        加上对应的选项
正常                                          通过
已过期                                        拒绝  errors.Is(err, jwt.ErrTokenExpired)
没有 exp                                      通过  ←         WithExpirationRequired() → 拒绝（ErrTokenRequiredClaimMissing）
payload 被改（role 改成 admin）                拒绝  ErrTokenSignatureInvalid
alg=none                                      拒绝           
HS512 签发，密钥相同                           通过  ←         WithValidMethods(["HS256"]) → 拒绝
别的密钥签发                                   拒绝  ErrTokenSignatureInvalid
乱码、空串                                     拒绝  ErrTokenMalformed
issuer 不符                                   —              WithIssuer("…") → 拒绝（ErrTokenInvalidIssuer）
用 3 个字节的密钥签名                           成功，库不检查密钥长度
```

- 要讲清：JWT 是"签名"不是"加密"。HS256 是 HMAC-SHA256，签发和校验用同一个密钥
- 限定算法的意义：现在是 HS256 加 v5 默认拒绝 `none`，不限定也攻不进来。但如果将来换成 RS256，而 keyFunc 不看算法、一律返回同一个 key，攻击者就能把公钥当成 HMAC 密钥来伪造 Token。这是"算法混淆攻击"，限定算法是固定动作
- 密钥长度：RFC 7518 第 3.2 节要求 HS256 的密钥不短于 256 位。库不检查，所以在配置校验里检查
- 测过期不用 sleep：给签发方注入时钟，直接签一个 `exp` 在过去的 Token；或者解析时用 `jwt.WithTimeFunc`
- 用户 ID 放 `RegisteredClaims.Subject`（字符串），取出来再 `strconv.ParseUint`

思考题 5（无法吊销）：常见办法有三种。很短的有效期加刷新令牌；在 Redis 里维护黑名单（按 `jti`）；用户表存一个版本号，Token 里带着，每次比对。后两种都让鉴权重新依赖存储，等于放弃了无状态的好处。本阶段接受这个限制。

**配置改动会连带影响已有测试**：加了必填的 `JWT_SECRET` 之后，`config_test.go` 的 `setRequiredSecrets` 和集成测试里的 `config.Load("")` 都要跟着改。让学习者先预测哪些测试会挂。

---

### Step 7：注册与登录

这是分层第一次完整跑通，建议带着学习者自底向上走一遍：repository（Step 5 已有）→ service → handler → router → `main.go` 组装。每写一层就测一层。

**"先查后插"的竞态** ✔

20 个 goroutine 同时执行"先 `COUNT` 用户名，没有就 `INSERT`"：

```
有唯一索引的表   成功 1，被拒 19，其他错误 0，表里 1 行
没有唯一索引的表  成功 20，表里 20 行
```

参考项目的注册就是这个写法（`service/auth.go:59-74`）。用户名有唯一索引兜底；邮箱只有普通索引（`init.sql:96`），并发注册会出现重复邮箱。一定要把这件事和阶段 2 的超卖联系起来：都是"检查"和"写入"之间有空隙。

**排序规则** ✔

`utf8mb4_unicode_ci` 下：`'Alice' = 'alice'`、`'alice ' = 'alice'`、`'álice' = 'alice'` 都是 1。原因分别是不区分大小写、比较时忽略尾部空格（PAD SPACE）、不区分重音。实测往唯一索引里插入 `Alice`、`ALICE`、`alice `（带尾部空格）、`álice` 都报重复键；`WHERE username = 'ALICE'` 查出来的是 `alice`。

所以推荐在 service 里把用户名规范化成小写并限定字符集，让数据库的比较规则不影响业务。另一条路是把这一列的排序规则改成 `utf8mb4_bin`，可以作为延伸讨论。

**登录的时间差**（思考题 3）：用户不存在时不到 1ms 就返回，密码错误时要约 50ms。攻击者据此可以探测哪些用户名存在。抹平的办法是用户不存在时也拿一个预先算好的假哈希比对一次。属于可选的加固，讲清原理即可。

**gin 参数绑定** ✔

```
合法请求                           200
price=0（required）                400  validator.ValidationErrors  [Price:required]
price 缺失                         400  同上
price 是字符串 "100"                400  *json.UnmarshalTypeError
price 是 99.5                      400  *json.UnmarshalTypeError（cannot unmarshal number 99.5 … of type int64）
*int 字段缺失（required）           400  [Limit:required]
*int 字段为 0（required,gte=1）     400  [Limit:gte]         ← 用指针才能区分"没传"和"传了 0"
end_time 早于 start_time（gtfield） 400  [End:gtfield]
时间写成 "2026-11-13 20:00:00"      400  *time.ParseError    ← time.Time 只认 RFC 3339
用户名带下划线（alphanum）           400  [Username:alphanum]  ← alphanum 不含下划线，推荐的规则要自己写
72 个汉字的密码（max=72）           200
空请求体                           400  io.EOF
JSON 不完整                        400  io.ErrUnexpectedEOF
多出 "role":"admin"                200  被忽略
```

校验错误里的字段名默认是 Go 的字段名。想换成 JSON 里的名字，启动时注册一次 ✔：

```go
if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
    v.RegisterTagNameFunc(func(fld reflect.StructField) string {
        name := strings.SplitN(fld.Tag.Get("json"), ",", 2)[0]
        if name == "-" {
            return ""
        }
        return name
    })
}
```

注册之后 `fe.Field()` 返回的是 `seckill_price` 这样的名字。

**接口定义在使用方**：`UserService` 需要什么方法，就在 service 包里声明一个只含这些方法的接口。`*repository.UserRepository` 自动满足它，不需要显式声明。单元测试里用一个内存 map 实现这个接口，就能不连数据库地测"用户名重复""密码错误"这些分支。这个模式本阶段完整做一次就够，后面几个 service 可以直接用集成测试。

**参考项目的对照点**（行号已核对）
- `handler/auth.go:55-70`：每个 handler 用 `switch err` 手写一遍错误到状态码的映射，并且用 `==` 比较。service 一旦把错误包了一层，`==` 就不再匹配
- `service/auth.go:259`：`strings.Contains(err.Error(), "redis: nil")`，靠错误文本判断类型
- `service/auth.go:97`：`_ = redis.IncrementAdminUserCreated(…)`，忽略错误
- `repository/user.go:10-19`：空结构体加全局 `database.DB`

---

### Step 8：鉴权中间件与路由

**容易漏的一点**：中间件里判定失败后，写完响应必须 `c.Abort()`，否则后面的 handler 照样执行。项目里的 `Fail` 只写响应，不会 Abort。阶段 0 的 `Recovery` 是先 `c.Abort()` 再 `Fail` 的，可以让学习者去看那一段。

**身份放在哪里**
- 推荐放进请求的 context，做法和阶段 0 的 `RequestID` 中间件一样：`c.Request = c.Request.WithContext(newCtx)`。key 用不导出的空结构体类型
- 取身份的函数返回 `(Identity, bool)`。handler 拿到 `false` 时按错误处理，不要默默继续
- 参考项目 `handler/order.go:22` 的 `c.GetUint64("user_id")`：gin 的这类取值函数在 key 不存在时返回零值，不报错。路由漏挂中间件时 `userID` 就是 0，接下来按 `user_id = 0` 查询。这是思考题 3 要学习者自己发现的

**其他要点**
- 401 表示"不知道你是谁"，403 表示"知道你是谁，但你不能做这件事"
- `Auth` 挂在路由组上，自然排在三个全局中间件的里面，所以鉴权失败的请求也有访问日志
- R13 的具体位置：`middleware/jwt.go` 第 89 行，管理员中间件有 `if claims.Role != jwt.RoleAdmin`，普通用户的中间件（13~59 行）没有对应的判断
- 参考项目每个请求查一次 Redis（第 43 行）：换来了可以即时禁用用户；代价是鉴权依赖 Redis，而且 R17 指出 key 不存在时默认放行
- 鉴权矩阵适合写成表驱动测试。Token 用 Step 6 的签发函数现造

---

### Step 9：创建管理员

- 标准库的 `flag` 包加上对 `os.Args[1]` 的判断就够了，不需要引入命令行框架
- 密码从环境变量读最简单。交互式输入要用 `golang.org/x/term`，是新的依赖，可选
- 命令行参数会进 shell 的历史记录，进程运行期间 `ps` 也能看到
- 这一步的意义是让学习者体会 `cmd/` 下多个入口共用 `internal/` 的好处。阶段 4 的 Worker、阶段 2 的压测造数工具都是同样的结构

**参考项目的做法**（`service/admin.go:52-86`，在 `cmd/api/main.go:73` 调用）：每次启动按配置同步管理员——没有就建；密码和配置不一致或账号被禁用，就改成配置里的密码并重新启用。好处是不需要任何额外步骤。问题是：启动流程里有写库的副作用；一个被手动禁用的管理员会在下次重启时自动恢复；两个实例同时启动时，其中一个会在 `Create` 上撞唯一键，`EnsureAdminAccount` 返回错误。

---

### Step 10：商品管理

- 推荐把 `PUT` 定义成整体替换：请求体带全部可编辑字段。这样校验逻辑和新建完全相同，不必处理"只传了一部分字段"的情况。Step 11 的活动修改同理
- "修改不存在的 ID 返回 404"：影响行数为 0 时再按 ID 查一次。✔ 更新不存在的 ID 时 `err == nil`、`RowsAffected == 0`
- 思考题 2：用 `Updates(map)` 时 GORM 会带上新的 `updated_at`，所以内容完全相同通常也返回 1。例外是两次写入落在同一毫秒。换成 `UpdateColumn` 就不带 `updated_at`，内容相同时返回 0 ✔
- 思考题 3：`a, b := 0.1, 0.2; fmt.Println(a + b)` 输出 `0.30000000000000004`。直接写 `fmt.Println(0.1 + 0.2)` 输出 `0.3`，因为 Go 的常量表达式在编译期按任意精度计算。价格传 `99.5` 时绑定直接失败 ✔
- 分页参数 ✔：`form:"page,default=1" binding:"min=1"` 这种写法可用。`page=0` 和 `page_size=100000` 被校验拒绝；`page=abc` 是 `strconv` 的解析错误；`page=`（空值）取默认值
- 路径参数 ✔：`uri:"id" binding:"required,min=1"` 配 `ShouldBindUri`。`0`、`abc`、`-1`、超过 uint64 上限的数都会报错

---

### Step 11：秒杀活动管理

**核心概念：把判断条件放进 `UPDATE` 的 `WHERE` 里**

"查出来 → 判断 → 改 → 存回去"是四步，中间随时可能被别的请求插进来。"`UPDATE … WHERE id = ? AND status = ?`"是一步，数据库用行锁保证同一时刻只有一个事务能改这一行，后来的事务看到的是已经改过的状态，条件不再成立。

✔ 20 个 goroutine 同时对一个草稿执行发布：影响 1 行的恰好 1 个，影响 0 行的 19 个。

这一步要让学习者把"影响行数"当成操作成败的唯一依据。阶段 2 的扣库存（`WHERE available_stock >= ?`）和阶段 5 的支付、取消都是这个模式，这里先在一个不涉及钱的地方练熟。

**思考题要点**
1. "先查后 `Save`"：发布和编辑可以同时通过检查，后写的覆盖先写的。可能的结果是活动已经发布，但内容是另一个管理员刚保存的那一版，而发布的人并没有看过它
2. 影响 0 行之后的查询只用来决定返回 404 还是 409，不参与任何写操作。即使查的时候状态又变了，最坏也只是错误提示不够准确，数据不会出错。可以点明：**做决定**的那一步必须是原子的，**解释结果**的那一步不必
3. 推荐 `[start, end)`。Go 里写成：`now.Before(start)` 是未开始，`!now.Before(end)` 是已结束，其余是进行中。恰好等于 `start` 算进行中，恰好等于 `end` 算已结束
4. 库存 100，预热后卖了 30，Redis 里剩 70。这时 MySQL 的总量改成 50，Redis 应该变成多少？没有一个不出错的简单答案。所以发布后不让改，要调整只能走阶段 5 的增量接口
5. ✔ `Preload` 发 2 条：先查活动，再用 `IN` 一次查出商品。`Joins` 发 1 条 `LEFT JOIN`。循环里逐个查是 1 + N 条

   ```sql
   -- Preload("Product")
   SELECT * FROM `seckill_activities` WHERE status = 2 ORDER BY start_time ASC, id ASC LIMIT 20
   SELECT * FROM `products` WHERE `products`.`id` IN (1,2)
   -- Joins("Product")
   SELECT `seckill_activities`.`id`, …, `Product`.`id` AS `Product__id`, `Product`.`name` AS `Product__name`, …
   FROM `seckill_activities` LEFT JOIN `products` `Product` ON `seckill_activities`.`product_id` = `Product`.`id`
   WHERE seckill_activities.status = 2 ORDER BY seckill_activities.start_time ASC, seckill_activities.id ASC LIMIT 20
   ```

   学习者会注意到一个现象：日志里 `Preload` 的那条先打印出来。原因是主查询的日志要等整个回调链（包括 Preload）跑完才记录。用 `Joins` 时，`Where` 和 `Order` 里的列名要带表名
6. 以服务器时钟为准。客户端的时钟可能不准，也可以被用户改
7. `binding` tag 只能管单个字段和同一结构体内的字段比较（`gtfield`）；跨表的规则（秒杀价不高于商品原价）只能放 service；数据库 CHECK 管得了同一行内的关系（`end_time > start_time`），管不了跨表。CHECK 是最后一道防线，不应该是唯一的一道——它的报错（3819）对用户并不友好

**实现建议**
- 发布时"活动还没结束"的条件也可以放进 `WHERE`：`… AND status = 1 AND end_time > ?`，`?` 是 Go 传进去的 `now`
- 新建活动时"商品存在"是先查后插，但商品不会被删除，所以这个空隙没有危害。可以拿它和注册做对比：同样是先查后写，为什么这里没问题
- 时钟注入：service 里放一个 `now func() time.Time` 字段，构造时传 `time.Now`，测试里传一个返回固定时间的函数
- "下线"在本阶段只是一次状态流转。阶段 3 之后它还要让 Redis 那边停止接单，到时候再处理

---

### Step 12：订单查询

**越权访问**（思考题 1、2）：404 不透露"这个订单号存在"；403 等于告诉对方订单号是有效的。把 `user_id` 写进 `WHERE`，是让"不是你的"和"不存在"在数据库这一层就无法区分，不依赖代码里记得去比较。

✔ 两条查询生成的 SQL：

```sql
SELECT * FROM `orders` WHERE id = 7203685477580123456 AND user_id = 1 LIMIT 1    -- 本人：查到
SELECT * FROM `orders` WHERE id = 7203685477580123456 AND user_id = 2 LIMIT 1    -- 别人：record not found
```

**游标分页的实现**：`limit` 是 20 就查 21 条。拿到 21 条说明还有下一页，返回前 20 条，`next_cursor` 是第 20 条的订单号；不足 21 条就是最后一页。第一页没有 `cursor`，不加 `id < ?` 这个条件。

**实验的预期结果** ✔（数据用附录 B 的脚本生成：2 万单、200 个用户、订单号 100001~120000）

| 语句 | type | key | rows | Extra |
|---|---|---|---|---|
| ① `user_id = 77 ORDER BY id DESC LIMIT 21` | ref | idx_orders_user_id_id | 100 | Backward index scan |
| ② `user_id = 77 AND id < 115000 ORDER BY id DESC LIMIT 21` | range | idx_orders_user_id_id（key_len=16） | 75 | Using index condition; Backward index scan |
| ③ 同 ①，加 `IGNORE INDEX (idx_orders_user_id_id)` | index | PRIMARY | 21 | Using where; Backward index scan |
| ④ `user_id = 77 ORDER BY created_at DESC LIMIT 21` | ref | idx_orders_user_id_id | 100 | Using filesort |
| ⑤ `ORDER BY id DESC LIMIT 20 OFFSET 15000` | index | PRIMARY | 15020 | Backward index scan |
| 超时扫描 `status = 1 AND expire_at <= NOW(3) ORDER BY expire_at LIMIT 100` | range | idx_orders_status_expire_at | 275 | Using where; Using index |

实际读取的行数（`Handler_read_*`）：

| 语句 | 实际读的行 |
|---|---|
| ② 游标 | 20 |
| `user_id = 77 … LIMIT 21 OFFSET 60` | 80 |
| ③ 忽略索引 | **4132** |
| ⑤ 全表 `OFFSET 15000` | 15019 |
| 全表游标 `id < 105000 … LIMIT 20` | 19 |

解读要点：
- ③ 是这个实验里最有价值的一条。`EXPLAIN` 估的是 21 行，实际读了 4132 行。优化器的打算是"沿主键倒着走，凑够 21 条符合条件的就停"，估算时假定很快就能凑够；实际上每 200 行才有一行属于这个用户。由此得出：`EXPLAIN` 的 `rows` 是估算，要以实际读取量为准
- ④ 索引是按 `(user_id, id)` 排好序的，按 `id` 取不需要再排序。按 `created_at` 排序，索引帮不上，只能把这个用户的 100 行都取出来再排
- ⑤ `OFFSET` 的代价和偏移量成正比：数据库要先读出前面的 15000 行再丢掉
- 游标分页的代价：不能跳页，只能一页一页往下翻；排序的列必须唯一并且有索引。适合"我的订单"这种一直往下拉的列表，不适合需要显示总页数的后台表格

**进阶实验**（思考题 6）✔：把索引换成只有 `(user_id)` 一列之后，语句 ② 的计划变成：

```
type=index_merge  key=idx_orders_user_id,PRIMARY  Extra=Using intersect(idx_orders_user_id,PRIMARY); Using where; Using filesort
实际读取: Handler_read_key=76  Handler_read_next=75  Handler_read_rnd=75  Sort_rows=20
```

而语句 ① 的计划和联合索引时一样。结论：InnoDB 的二级索引带着主键，这对等值查询加按主键排序是够用的；但对主键上的范围条件，这个版本的优化器没有把隐含的主键当成索引的第二列来做范围扫描。提醒学习者：优化器的选择和版本、数据分布都有关系，结论要以自己机器上的 `EXPLAIN` 为准。

**订单号的 JSON 输出** ✔：`json:"id,string"` 输出 `"7203685477580123456"`。

**参考项目**：`repository/order_repo.go:80-95` 的 `FindByUserID` 没有 `LIMIT`，一次返回用户的全部订单；按 `create_time` 排序，而表上没有 `(user_id, create_time)` 的索引，对应上面的 ④。

---

### Step 13、14

- `.http` 文件是 VS Code 的 REST Client 和 JetBrains 系 IDE 都支持的格式，可以用变量把登录拿到的 Token 传给后面的请求。学习者用 Neovim，可能更愿意用 curl 脚本，都可以
- 端到端测试的结构和阶段 0 的 `integration_test.go` 相同，只是这次要组装完整的路由。组装代码最好能和 `main.go` 共用一个函数，避免测试里的组装和真实的组装不一致
- 打 tag 之前完整跑一遍三个 make 目标。阶段 0 的教训就在 Step 0

---

## 3. 范围守卫

下面这些事情本阶段不要引导学习者去做。它们要么属于后面的阶段，要么和秒杀主线无关。

| 不做 | 原因 |
|---|---|
| 邮箱验证码、图形验证码、找回密码、用户封禁 | 参考项目有，但和主线无关 |
| 刷新令牌、Token 黑名单 | 阶段 6 可选 |
| 订单号生成器（雪花算法） | 阶段 2。本阶段测试里手工指定订单号 |
| 下单、扣库存、支付、取消 | 阶段 2 和阶段 5 |
| 用 Redis 缓存活动信息 | 阶段 3 和阶段 6 |
| 事务管理器 | 阶段 2。本阶段没有跨表事务。可以提前问一句："阶段 2 要在一个事务里调用两个 repository，现在的写法支持吗？" |
| 泛型的通用 repository、依赖注入框架（wire、dig） | 手写构造函数和组装代码更利于理解 |
| `AutoMigrate`、软删除、GORM Hook | 违反 `roadmap.md` 1.4 的约定 |
| Swagger / OpenAPI 生成 | 可以放到阶段 8 的文档工作里 |

---

## 附录 A：参考建表语句 ✔

四个迁移文件的 up 部分。down 部分都是 `DROP TABLE IF EXISTS`。已经用 golang-migrate 在 MySQL 8.0.46 上验证过：从空库 up 到最新、逐个 down 回到 0、再 up，全程无报错。

```sql
-- 000002_create_users.up.sql
CREATE TABLE `users` (
    `id`            BIGINT UNSIGNED  NOT NULL AUTO_INCREMENT COMMENT '用户ID',
    `username`      VARCHAR(32)      NOT NULL COMMENT '登录名（小写字母、数字、下划线）',
    `password_hash` VARCHAR(255)     NOT NULL COMMENT '密码哈希（bcrypt）',
    `role`          TINYINT UNSIGNED NOT NULL COMMENT '角色: 1-普通用户 2-管理员',
    `created_at`    DATETIME(3)      NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
    `updated_at`    DATETIME(3)      NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_users_username` (`username`),
    CONSTRAINT `chk_users_role` CHECK (`role` IN (1, 2))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户表';
```

```sql
-- 000003_create_products.up.sql
CREATE TABLE `products` (
    `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '商品ID',
    `name`        VARCHAR(128)    NOT NULL COMMENT '商品名称',
    `description` VARCHAR(1024)   NOT NULL DEFAULT '' COMMENT '商品描述',
    `price`       BIGINT UNSIGNED NOT NULL COMMENT '原价（单位：分）',
    `created_at`  DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
    `updated_at`  DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间',
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='商品表';
```

```sql
-- 000004_create_seckill_activities.up.sql
CREATE TABLE `seckill_activities` (
    `id`              BIGINT UNSIGNED  NOT NULL AUTO_INCREMENT COMMENT '活动ID',
    `product_id`      BIGINT UNSIGNED  NOT NULL COMMENT '商品ID',
    `title`           VARCHAR(128)     NOT NULL COMMENT '活动标题',
    `seckill_price`   BIGINT UNSIGNED  NOT NULL COMMENT '秒杀价（单位：分）',
    `total_stock`     INT UNSIGNED     NOT NULL COMMENT '活动总库存（发布后不变，对账基准）',
    `available_stock` INT UNSIGNED     NOT NULL COMMENT '可售库存',
    `per_user_limit`  INT UNSIGNED     NOT NULL COMMENT '每人限购件数',
    `start_time`      DATETIME(3)      NOT NULL COMMENT '开始时间',
    `end_time`        DATETIME(3)      NOT NULL COMMENT '结束时间',
    `status`          TINYINT UNSIGNED NOT NULL COMMENT '状态: 1-草稿 2-已发布 3-已下线',
    `created_at`      DATETIME(3)      NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
    `updated_at`      DATETIME(3)      NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间',
    PRIMARY KEY (`id`),
    CONSTRAINT `chk_activities_status` CHECK (`status` IN (1, 2, 3)),
    CONSTRAINT `chk_activities_time_window` CHECK (`end_time` > `start_time`),
    CONSTRAINT `chk_activities_stock` CHECK (`available_stock` <= `total_stock`),
    CONSTRAINT `chk_activities_per_user_limit` CHECK (`per_user_limit` >= 1)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='秒杀活动表';
```

```sql
-- 000005_create_orders.up.sql
CREATE TABLE `orders` (
    `id`           BIGINT UNSIGNED  NOT NULL COMMENT '订单号（应用生成，不是自增）',
    `user_id`      BIGINT UNSIGNED  NOT NULL COMMENT '用户ID',
    `activity_id`  BIGINT UNSIGNED  NOT NULL COMMENT '活动ID',
    `product_id`   BIGINT UNSIGNED  NOT NULL COMMENT '商品ID',
    `product_name` VARCHAR(128)     NOT NULL COMMENT '下单时的商品名称快照',
    `quantity`     INT UNSIGNED     NOT NULL COMMENT '购买数量',
    `unit_price`   BIGINT UNSIGNED  NOT NULL COMMENT '下单时的成交单价快照（单位：分）',
    `total_amount` BIGINT UNSIGNED  NOT NULL COMMENT '应付总额（单位：分）',
    `status`       TINYINT UNSIGNED NOT NULL COMMENT '状态: 1-待支付 2-已支付 3-已取消',
    `expire_at`    DATETIME(3)      NOT NULL COMMENT '支付截止时间',
    `paid_at`      DATETIME(3)      NULL COMMENT '支付时间',
    `cancelled_at` DATETIME(3)      NULL COMMENT '取消时间',
    `created_at`   DATETIME(3)      NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
    `updated_at`   DATETIME(3)      NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间',
    PRIMARY KEY (`id`),
    KEY `idx_orders_user_id_id` (`user_id`, `id`),
    KEY `idx_orders_status_expire_at` (`status`, `expire_at`),
    CONSTRAINT `chk_orders_id` CHECK (`id` > 0),
    CONSTRAINT `chk_orders_status` CHECK (`status` IN (1, 2, 3)),
    CONSTRAINT `chk_orders_quantity` CHECK (`quantity` >= 1)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='订单表';
```

几处设计说明：
- 活动表没有二级索引。它只有几十到几百行，全表扫描比维护索引便宜。可以用来问学习者"这张表要不要给 `status` 建索引"
- `(activity_id, user_id)` 的索引没有建。它是为阶段 2 的限购校验准备的，而限购到底怎么做（查订单表，还是单独一张计数表）要到阶段 2 才定，到时候用迁移加

---

## 附录 B：造 2 万条订单的 SQL ✔

在 `seckill_test` 库里执行。生成 200 个用户、每人 100 单，订单号 100001~120000，其中 500 单是待支付。

```sql
SET SESSION cte_max_recursion_depth = 20000;

INSERT INTO orders (id, user_id, activity_id, product_id, product_name, quantity, unit_price, total_amount, status, expire_at)
WITH RECURSIVE seq (n) AS (
    SELECT 1 UNION ALL SELECT n + 1 FROM seq WHERE n < 20000
)
SELECT 100000 + n,                 -- 订单号
       (n * 7) % 200 + 1,          -- 200 个用户，每人 100 单
       n % 5 + 1,                  -- 5 个活动
       1, '测试商品', 1, 9900, 9900,
       CASE WHEN n % 40 = 0 THEN 1 WHEN n % 10 = 0 THEN 3 ELSE 2 END,
       NOW(3) + INTERVAL (n % 3600 - 1800) SECOND
FROM seq;

ANALYZE TABLE orders;
```

说明：递归 CTE 默认最多递归 1000 层，所以要先调大 `cte_max_recursion_depth`。`ANALYZE TABLE` 更新统计信息，不执行的话 `EXPLAIN` 的估算可能偏差很大。这是造测试数据的脚本，里面用 `NOW(3)` 不违反"业务代码不写 `NOW()`"的约定。实验做完用 `TRUNCATE TABLE orders` 清掉。

测量实际读取行数的方法（要在同一个会话里）：

```sql
FLUSH STATUS;
SELECT * FROM orders IGNORE INDEX (idx_orders_user_id_id) WHERE user_id = 77 ORDER BY id DESC LIMIT 21;
SHOW SESSION STATUS LIKE 'Handler_read%';
```

---

## 附录 C：为阶段 2 预留的实测结论 ✔

规划阶段 1 时顺带验证的。它们解释了阶段 1 的几个设计决定，也是规划阶段 2 的 todolist 时要用到的。环境同上，隔离级别是默认的可重复读。

**1. 事务里先做了一次普通 `SELECT`，限购校验会读到旧数据**

步骤：事务 1 `BEGIN`，（可选）先普通 `SELECT` 一次活动信息。事务 2 为同一个用户扣库存、插入订单、提交。然后事务 1 扣库存，再用普通 `SELECT` 统计该用户的已购数量。

```
事务 1 开头先做了普通 SELECT   → 看到该用户已购 = 0（真实已提交的是 1）
事务 1 的第一条语句就是 UPDATE → 看到该用户已购 = 1
```

原因：可重复读下，一致性视图是在事务里**第一次普通读**的时候建立的。之后的普通读都用这个视图，看不到别的事务后来提交的数据。`UPDATE` 是当前读，不受视图限制，所以库存不会扣错；但限购靠的是后面那次普通读。写法上的含义：下单事务的第一条语句必须是扣库存的 `UPDATE`，活动信息要在事务外面读，或者通过加锁的读来读。

**2. 对不存在的行 `SELECT … FOR UPDATE` 之后再 `INSERT`，不同用户之间会死锁**

步骤：两个事务分别对 `(activity_id=901, user_id=1)` 和 `(activity_id=901, user_id=2)` 执行 `SELECT COUNT(*) … FOR UPDATE`，这两行都还不存在。两个事务都立刻拿到了锁。然后各自 `INSERT` 自己的订单。

```
事务 1 INSERT: 等待约 300ms 后成功
事务 2 INSERT: Error 1213 (40001): Deadlock found when trying to get lock
```

原因：行不存在时 `FOR UPDATE` 加的是间隙锁。间隙锁之间互相兼容，所以两个事务都能加上；但它会阻塞别的事务往这个间隙里插入。两边都持有间隙锁、都想插入，形成死锁。含义："用悲观锁先查一下有没有买过，再插入"这条路走不通。

**3. 订单表对活动表建了物理外键时，"先插订单、再扣库存"会大面积死锁**

8 个 goroutine 各执行 30 次事务，都指向同一个父行：

| 子表 | 语句顺序 | 成功 | 死锁（1213） |
|---|---|---|---|
| 有外键 | 先插子表，再更新父表 | 39 | **201** |
| 有外键 | 先更新父表，再插子表 | 240 | 0 |
| 无外键 | 先插子表，再更新父表 | 240 | 0 |
| 无外键 | 先更新父表，再插子表 | 240 | 0 |

原因：有外键时，插入子表的行要给父表的对应行加共享锁。两个事务都拿到了共享锁，又都想升级成排他锁去更新父行，互相等待。

**4. GORM 默认给每次写操作包一层事务**

一次 `Create`：`Com_begin` +1、`Com_commit` +1、`Com_insert` +1。设置 `SkipDefaultTransaction: true` 之后：`Com_begin` +0、`Com_commit` +0、`Com_insert` +1。这是阶段 8 的一个优化项，测基线时要写明用的是哪种配置。

**5. 阶段 2 会用到的两种写法生成的 SQL**

```sql
-- db.Table("seckill_activities").Where("id = ? AND available_stock >= ?", 1, 2).
--    UpdateColumn("available_stock", gorm.Expr("available_stock - ?", 2))
UPDATE `seckill_activities` SET `available_stock`=available_stock - 2 WHERE id = 1 AND available_stock >= 2

-- db.Clauses(clause.Locking{Strength: "UPDATE"}).Take(&activity, 1)
SELECT * FROM `seckill_activities` WHERE `seckill_activities`.`id` = 1 LIMIT 1 FOR UPDATE
```

`UpdateColumn` 不会带上 `updated_at`。想带上就用 `Update`，并且用 `Model(&SeckillActivity{})` 而不是 `Table(…)`。

**6. 其他**
- 无符号列的减法越界报 1690，见 Step 4
- `Order.ID` 为 0 的问题，见 Step 5。阶段 2 引入发号器时要再检查一次
- bcrypt 的耗时决定了压测必须预先生成 Token，见 Step 6
