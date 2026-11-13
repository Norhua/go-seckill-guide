# MySQL 与 GORM · 阶段 0

> 涉及代码：`internal/infra/mysql/mysql.go`、`internal/config/config.go`（`MySQLConfig`、`DSN()`）、`migrations/`、`deploy/mysql/init.sql`、`internal/server/integration_test.go`
> 用法说明见 [../README.md](../README.md)
>
> 实验准备：进入 MySQL 命令行
> `docker exec -it seckill-mysql mysql -uroot -p`

---

## 一、连接

### Q1. 一条 MySQL 连接有多"贵"？ ⭐⭐ 🎯

1. 应用和 MySQL 之间建立一条可用的连接，要经过哪些步骤？（TCP 握手之后，MySQL 协议层还要做什么？）
2. 一条 MySQL 连接在**同一时刻**能执行几条 SQL？所以 100 个并发请求如果只有 1 条连接会怎样？
3. 在 MySQL 服务端，每条连接对应什么资源（线程？内存？）。
4. 综合 1～3：为什么既不能"每个请求新建一条连接"，也不能"所有请求共用一条连接"？中间方案叫什么？
5. 你在 network-http Q4 里见过同样的思路，是哪个？

---

### Q2. DSN 逐段拆解 ⭐⭐

**看代码**：`config.go` 的 `DSN()`

```
root:密码@tcp(127.0.0.1:13306)/seckill?charset=utf8mb4&parseTime=True&loc=Local
```

1. 说出每一段的含义。`tcp(...)` 这个位置还能写别的什么协议？
2. 去掉 `parseTime=True` 会怎样？**动手**：去掉后跑 `make test-integration`，读报错信息。它说明 MySQL 驱动默认把 `DATETIME` 列读成什么 Go 类型？
3. `loc=Local` 的 `Local` 指的是谁的"本地"——MySQL 服务器的，还是 Go 进程的？
4. 假如密码里包含 `@` 或 `/`，这个 DSN 还能被正确解析吗？`Makefile` 里 `MIGRATE_DB_URL`（`mysql://` 开头的 URL 格式）呢？两种格式对特殊字符的处理规则一样吗？
5. `DSN()` 返回的字符串包含明文密码。如果有人顺手写了 `log.Info("dsn", cfg.MySQL.DSN())` 会怎样？`LogValue` 的脱敏保护得了这种情况吗？

---

### Q3. `sql.DB` 不是一条连接 ⭐⭐⭐ 🎯

**看代码**：`mysql.go` 的 `sqlDB, err := db.DB()` 及其后的 4 个 `Set...`

1. `*gorm.DB`、`*sql.DB`、`go-sql-driver/mysql` 三者是什么关系？画一个三层的图，标出每层的职责。
2. 标准库 `sql.Open()` 被调用时会真的去连数据库吗？所以如果只 `Open` 不 `Ping`，密码配错了要到什么时候才发现？
3. 那 `gorm.Open()` 呢？它内部有没有替你 Ping？（线索：查 `gorm.Config` 的 `DisableAutomaticPing` 字段）既然如此，`mysql.go` 里再做一次带超时的 `PingContext` 是多余的吗？它比 GORM 自带的那次多了什么？
4. "启动时发现依赖不可用就立刻退出"叫 Fail-Fast。反面做法是"先启动，等请求来了再报错"。在有负载均衡和自动重启的部署环境里，为什么前者更好？

---

### Q4. 连接池的四个参数 ⭐⭐⭐ 🎯

**看代码**：`config.yaml` 的 `max_open_conns: 50`、`max_idle_conns: 25`、`conn_max_lifetime: 1h`、`conn_max_idle_time: 10m`

1. 用自己的话分别解释这四个参数。
2. 50 条连接都在忙的时候，第 51 个请求会怎样——报错、排队、还是新建连接？如果是排队，它最多等多久？由什么决定？
3. `Validate` 禁止 `max_idle_conns > max_open_conns`。如果不拦，`database/sql` 自己会怎么处理？
4. `MaxIdleConns` 设得太小（比如 2，这正是 `database/sql` 的默认值）而并发很高时，会出现什么现象？（提示：用完的连接放不回池子里……）
5. MySQL 服务端会主动断开空闲太久的连接（`wait_timeout`，**动手**：`SHOW VARIABLES LIKE 'wait_timeout';`）。如果应用这边连接池里的连接"活"得比服务端允许的更久，会发生什么？`ConnMaxLifetime` 应该比 `wait_timeout` 大还是小？
6. **动手**：`SHOW VARIABLES LIKE 'max_connections';`。如果你部署 4 个应用实例，每个 `max_open_conns=50`，加起来会超吗？所以"连接池上限"应该站在**谁**的角度来算？

**线索**：Alex Edwards 的文章 *Configuring sql.DB for Better Performance*；Go 官方文档 *Managing connections*。

---

### Q5. 连接池越大越好吗？ ⭐⭐⭐ 🎯

（存档 step6 讲过 `MaxOpenConns: 2000` 是反模式。不看存档，自己推一遍。）

1. MySQL 所在机器有 8 个 CPU 核。同时执行 2000 条 SQL 和同时执行 50 条 SQL，哪种情况下**总吞吐量**更高？为什么？
2. 连接数远超 CPU 核数时，数据库把时间花在了什么上？
3. **动手观察连接池**：启动服务后，在 MySQL 里执行 `SHOW PROCESSLIST;`，数一数来自应用的连接有几条。然后用 `for i in $(seq 200); do curl -s localhost:8080/readyz > /dev/null & done; wait` 打一波并发，再看一次。数量怎么变的？10 分钟后（`conn_max_idle_time`）再看呢？
4. `sqlDB.Stats()` 能返回连接池的运行指标。其中 `WaitCount` 和 `WaitDuration` 如果持续增长，说明什么？（阶段 6 会把它们接进监控）

**线索**：HikariCP 的 wiki *About Pool Sizing*（Java 连接池的文档，但道理通用，很短）。

---

## 二、GORM 生成的 SQL

### Q6. 猜 SQL ⭐⭐ 🎯

**看代码**：`integration_test.go` 里的三个 GORM 调用

```go
db.WithContext(ctx).Create(&record)                    // (a)
db.WithContext(ctx).First(&got, record.ID)             // (b)
db.WithContext(ctx).Delete(&pingTestRecord{}, record.ID) // (c)
```

1. **先自己写出**这三行各自对应的 SQL，再往下看。
2. **动手验证**：把 `mysql.go` 里的 `logger.Warn` 临时改成 `logger.Info`，跑 `make test-integration`，对照输出。
3. 对 (a)：`record.ID` 在 `Create` 之前是 0，之后变成了真实的 ID。GORM 是怎么拿到这个值的？它多发了一条查询吗？
4. 对 (b)：你只是想"按主键查一条"，GORM 生成的 SQL 里多了一个 `ORDER BY`。为什么 `First` 要排序？有一个不排序的兄弟方法，叫什么？按主键查时这个 `ORDER BY` 有性能影响吗？
5. 对 (b)：如果记录不存在，`First` 返回什么错误？`Find` 呢？这个差别是 GORM 新手的常见困惑，记下来。
6. `pingTestRecord` 没有声明表名时，GORM 会猜表名是什么？所以才需要 `TableName()` 方法。

<details>
<summary>核对：GORM 实际生成的 SQL（先自己猜完再展开）</summary>

```sql
-- (a) Create
INSERT INTO `ping_test` (`message`,`created_at`) VALUES ('integration_test_ping','2026-11-13 12:00:00.000');
-- 自增 ID 通过 MySQL 协议返回的 LastInsertId 回填，不会额外发 SELECT

-- (b) First
SELECT * FROM `ping_test` WHERE `ping_test`.`id` = 1 ORDER BY `ping_test`.`id` LIMIT 1;

-- (c) Delete
DELETE FROM `ping_test` WHERE `ping_test`.`id` = 1;
```

注意：日志里看到的是 GORM 把参数填进去之后的"展示版"。实际发给驱动的是带 `?` 占位符的语句加参数列表，见 Q8。

</details>

---

### Q7. 一次 `Create` 发了几条语句？ ⭐⭐⭐ 🎯

1. GORM 默认会把每次写操作（Create/Update/Delete）包在一个事务里。也就是说 Q6 的 (a) 实际发给 MySQL 的是哪三条语句？
2. 对一条单独的 `INSERT`，这个事务有必要吗？（InnoDB 在 `autocommit=1` 时，单条语句本身是不是原子的？）
3. GORM 提供了一个配置项关掉这个行为，叫什么？官方文档声称关掉后能带来多少性能提升？
4. 秒杀场景下每个下单请求都要写库。多出来的 `BEGIN` 和 `COMMIT` 意味着多几次网络往返？阶段 3 压测时，这会是一个可以拿出来讲的优化点。

---

### Q8. 占位符 `?` 与预处理语句 ⭐⭐⭐

1. 用 `?` 占位符而不是字符串拼接来传参，防的是什么攻击？举一个用字符串拼接时能被攻破的例子。
2. 带 `?` 的 SQL 发给 MySQL 有两种方式：
   - 服务端预处理：先 `PREPARE`，再 `EXECUTE`，最后 `CLOSE`
   - 客户端插值：驱动在本地把参数安全地转义并填进 SQL，一次发出

   `go-sql-driver/mysql` 默认用哪种？一条带参数的查询因此要几次网络往返？
3. DSN 里有个参数 `interpolateParams=true` 可以切换到另一种方式。它的收益是什么？驱动文档里提到它在哪些字符集下不安全？我们用的 `utf8mb4` 属于哪类？
4. GORM 也有一个 `PrepareStmt: true` 选项，它的思路是"预处理一次，后面反复用"。它和上一问的方案分别适合什么场景？

（这题偏深，阶段 0 知道"一条查询可能不止一次往返"就够了，阶段 3 性能优化时再回来。）

---

### Q9. `WithContext(ctx)` 做了什么？ ⭐⭐

**看代码**：`integration_test.go` 里每次调用都带 `db.WithContext(ctx)`

1. 不写 `WithContext` 直接 `db.Create(&record)` 能运行吗？那它用的是什么 context？
2. 带了 ctx 之后，如果 ctx 超时或被取消，正在执行的 SQL 会怎样？驱动是怎么让 MySQL 停下来的——是发了一条"取消"命令，还是直接断开连接？
3. 回忆 network-http Q5：`WriteTimeout` 到期后 handler 不会停。现在你知道让 SQL 停下来需要什么了——从 handler 到 repository，每一层该怎么传 ctx？
4. `db.WithContext(ctx)` 返回的是新对象还是修改了全局的 `db`？如果是后者，并发请求之间会互相干扰吗？

---

## 三、表结构

### Q10. 逐列读懂建表语句 ⭐⭐ 🎯

**看代码**：`migrations/000001_init_ping_test.up.sql`

1. `BIGINT UNSIGNED`：`INT` 的上限大约是 21 亿。订单表用 `INT` 做主键，什么时候会出事？`UNSIGNED` 换来了什么？Go 里对应什么类型？
2. `NOT NULL DEFAULT ''`：为什么很多规范要求"字段尽量 NOT NULL"？`NULL` 在 Go 里要怎么接收（`string` 能装 `NULL` 吗）？`WHERE message != 'a'` 能查出 `message` 为 `NULL` 的行吗？
3. `DATETIME(3)` 的 `(3)` 是什么意思？不写会怎样？对一个需要按时间精确排序的秒杀系统，精度不够会有什么问题？
4. `DATETIME` 和 `TIMESTAMP` 的区别：存储范围、是否带时区转换。记住"2038 年问题"这个关键词。
5. `ENGINE=InnoDB`：另一个著名的引擎是 MyISAM。秒杀系统必须用 InnoDB 的两个决定性理由是什么？（提示：事务、行级锁）
6. `utf8mb4`：MySQL 里的 `utf8` 为什么**不是**真正的 UTF-8？存一个 emoji 会怎样？
7. `COLLATE=utf8mb4_unicode_ci` 的 `ci` 是什么意思？**动手**：`SELECT 'A' = 'a';` 结果是什么？如果将来有个字段存的是区分大小写的令牌/兑换码，这会造成什么问题？

---

### Q11. 时区的三个层次 ⭐⭐⭐

**看代码**：`docker-compose.yml` 的 `TZ: Asia/Shanghai`、`--default-time-zone=+08:00`；DSN 的 `loc=Local`；建表语句的 `DEFAULT CURRENT_TIMESTAMP(3)`

1. 一个时间值从 Go 写入 MySQL、再读回 Go，涉及三个"时区设置"：Go 进程的本地时区、MySQL 会话的时区、列类型本身（`DATETIME` 不带时区）。分别是在哪里配置的？
2. `DATETIME` 列存 `2026-11-13 20:00:00`，它记录了"这是哪个时区的 20 点"吗？
3. 设想：MySQL 容器是东八区，而 Go 服务被部署到一台 UTC 时区的服务器上（`loc=Local` 此时等于 UTC）。Go 写入一个 `time.Now()`，然后由 MySQL 的 `CURRENT_TIMESTAMP` 写入另一行。这两行的时间能直接比较吗？差多少？
4. 秒杀的核心判断是"现在是否在活动时间内"。如果应用服务器时区和数据库时区不一致，会出什么事故？业界常见的两种规避做法是什么？（全链路统一 UTC；或者用整数时间戳）

---

## 四、迁移

### Q12. 为什么不用 `AutoMigrate`？ ⭐⭐ 🎯

（存档 step7 讲过，自己复述。）

1. GORM 的 `AutoMigrate` 能做什么？它**不会**做什么（删列？改列类型？）？
2. 多个应用实例同时启动、同时执行 `AutoMigrate` 会怎样？
3. "表结构变更"和"应用代码发布"应该是同一个动作还是两个独立的动作？为什么？
4. 用版本化 SQL 文件的方式，表结构的每一次变化都留在 git 历史里。这对团队协作和线上排障的价值是什么？

---

### Q13. golang-migrate 的工作机制 ⭐⭐⭐

**看代码**：`migrations/` 目录、`Makefile` 的 `migrate-*` 目标

1. **动手**：`make migrate-up` 之后，在 MySQL 里 `SHOW TABLES;`。多出来的那张表是什么？`SELECT * FROM schema_migrations;` 里的两列各代表什么？
2. `migrate` 工具怎么知道哪些迁移已经执行过、哪些还没有？
3. **动手制造一次失败**：`make migrate-new name=broken`，在 up 文件里写两条语句——第一条是合法的 `CREATE TABLE t1 (id INT);`，第二条是有语法错误的 SQL。执行 `make migrate-up`，然后查 `schema_migrations` 和 `SHOW TABLES;`。
   - `dirty` 变成了什么？
   - `t1` 表存在吗？
   - 再执行一次 `make migrate-up` 会怎样？
4. 为什么第一条语句没有被"回滚"？MySQL 的 DDL 语句（`CREATE TABLE`、`ALTER TABLE`）和事务是什么关系？关键词：**隐式提交（implicit commit）**。
5. 怎么从 dirty 状态里恢复？（查 `migrate force` 命令）做完实验记得清理：手动 `DROP TABLE t1`、删掉 broken 迁移文件。
6. 由此得出一条实践原则：一个迁移文件里应该放多少个 DDL 变更？

---

### Q14. 迁移文件里的几个选择 ⭐⭐

**看代码**：up 文件的 `CREATE TABLE IF NOT EXISTS`；`Makefile` 的 `down 1` 和 `?multiStatements=true`

1. 迁移文件里的 `IF NOT EXISTS` 是把双刃剑。假设线上已经存在一张**结构不同**的同名表，带 `IF NOT EXISTS` 的迁移会报错还是"成功"？哪种结果更危险？
2. `migrate-down` 写的是 `down 1` 而不是 `down`。不带数字的 `down` 会做什么？
3. 线上环境里，`down` 迁移真的能随便执行吗？如果 up 是"加一列"，down 是"删这列"，执行 down 的时候那一列里已经有的数据怎么办？
4. `multiStatements=true` 让一次请求可以执行多条用分号隔开的语句。迁移工具需要它，但**应用的 DSN 里没有加它**。如果应用 DSN 也打开这个选项，会放大哪种攻击的危害？
5. 已经在线上执行过的迁移文件，发现写错了，能直接改文件内容吗？正确做法是什么？

---

### Q15. 两个"初始化"的区别 ⭐⭐

**看代码**：`deploy/mysql/init.sql`（建库）与 `migrations/`（建表）

1. 为什么"建数据库"放在 Docker 的 init 脚本里，而"建表"放在 migrations 里？它们的执行时机、执行者、执行次数分别有什么不同？
2. `seckill_test` 库为什么要单独建一个，而不是测试也用 `seckill` 库？
3. **动手**：在 `init.sql` 末尾加一行 `CREATE DATABASE IF NOT EXISTS seckill_demo;`，然后 `make docker-down && make docker-up`，再 `SHOW DATABASES;`。新库出现了吗？为什么？（答案在 [docker Q5](../docker/stage0.md)）
