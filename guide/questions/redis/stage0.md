# Redis · 阶段 0

> 涉及代码：`internal/infra/redis/redis.go`、`internal/config/config.go`（`RedisConfig`）、`deploy/docker-compose.yml` 的 redis 服务
> 用法说明见 [../README.md](../README.md)
>
> 阶段 0 对 Redis 只做了"连上并 Ping 通"。但从阶段 3 起，Redis 是整个秒杀系统的核心（库存预扣就在这里），所以这组题有一半是在为后面铺路。
>
> 实验准备：进入 Redis 命令行
> `docker exec -it seckill-redis redis-cli -a <你的 REDIS_PASSWORD>`

---

## 一、Redis 是什么

### Q1. 为什么秒杀需要 Redis？ ⭐ 🎯

1. Redis 的数据主要存在哪里？MySQL 的数据主要存在哪里？这个差别带来的读写速度差距大约是几个数量级？
2. 既然 Redis 这么快，为什么不把订单、用户这些数据全放 Redis，不要 MySQL 了？列举 Redis 做不到或做不好的事。
3. 所以在秒杀系统里，Redis 和 MySQL 各自扮演什么角色？谁是"最终事实来源（source of truth）"？
4. 这种"两个存储各存一份"的架构必然带来一个问题，叫什么？（它是阶段 3、阶段 5 的主线）

---

### Q2. 单线程为什么还这么快？ ⭐⭐ 🎯

1. Redis 执行命令的核心是单线程的。"单线程"听起来很慢，为什么 Redis 仍能每秒处理十万级请求？至少说出两个原因。
2. 单线程执行命令带来一个对秒杀极其重要的性质：**两条命令不可能同时执行**。那么 `DECR stock` 这一条命令需要加锁吗？
3. 但是"先 `GET stock` 判断大于 0，再 `DECR stock`"是**两条**命令。两个客户端的这两条命令可能怎样交错？结果是什么？（这就是超卖。先想想有哪些办法能把"判断 + 扣减"变成不可分割的一步，阶段 3 会用到）
4. 单线程也有代价。**动手**：在 redis-cli 里执行一段空转的 Lua 脚本来模拟一条"慢命令"（循环次数按机器快慢调整，让它跑 2～4 秒）：

   ```
   EVAL "local i=0 while i<300000000 do i=i+1 end return i" 0
   ```

   同时在另一个终端反复 `curl localhost:8080/readyz`。发生了什么？这说明线上为什么严禁对大数据量执行 `KEYS *` 这类命令？

**线索**：搜索"Redis 为什么快 IO 多路复用"。

---

## 二、客户端

### Q3. Ping 的时候线路上传了什么？ ⭐⭐

**看代码**：`redis.go` 的 `client.Ping(pingCtx).Err()`

1. Redis 的通信协议叫 RESP，是纯文本的。**动手**（不用任何 Redis 客户端，直接用 TCP 工具）：

   ```bash
   printf 'AUTH 你的密码\r\nPING\r\n' | nc localhost 16379
   ```

   你看到了什么？`+OK`、`+PONG` 开头的 `+` 是什么意思？
2. `goredis.NewClient(...)` 被调用时，会立刻建立连接吗？和 `sql.Open` 的行为像不像？所以后面那次 `Ping` 的意义是什么？
3. go-redis 在一条**新连接**上发第一条业务命令之前，会自动先发几条"准备"命令（认证、选库等）。**动手**：在 redis-cli 里执行 `MONITOR`，然后重启你的 Go 服务，观察都收到了哪些命令。
4. `client.Ping(ctx)` 返回的不是 `error` 而是一个 `*StatusCmd`，要再调 `.Err()`。go-redis 为什么这样设计 API？（想想 `Get` 命令：它需要同时返回"值"和"错误"，还要支持 `.Int()`、`.Result()` 等多种取法）

---

### Q4. 连接池参数 ⭐⭐⭐

**看代码**：`config.yaml` 的 `pool_size: 50`、`min_idle_conns: 10`

1. Redis 服务端是单线程的，那客户端为什么还需要 50 条连接？一条连接上，前一个请求的响应没回来之前，能发下一个请求吗？
2. 对比 MySQL 连接池的四个参数，go-redis 的 `PoolSize`、`MinIdleConns` 各自和 MySQL 的哪个参数最接近？`MinIdleConns` 这种"预热"在什么时刻最有价值？（想想秒杀开始的那一秒）
3. 50 条连接全被占用时，新的请求会等待。等多久？go-redis 有个 `PoolTimeout` 参数我们没有配置，去文档里查它的默认值是怎么算出来的。
4. **动手**：服务启动后，在 redis-cli 里 `CLIENT LIST`，数一数来自应用的连接数。和 `min_idle_conns` 对得上吗？

---

### Q5. 超时之后，命令到底执行了没有？ ⭐⭐⭐ 🎯

**看代码**：`config.yaml` 的 `dial_timeout: 3s`、`read_timeout: 2s`、`write_timeout: 2s`

1. 三个超时分别管哪个阶段？
2. 关键思考：客户端发出 `DECR stock`，然后等了 2 秒没收到响应，`read_timeout` 触发，客户端拿到一个超时错误。此时 Redis 服务端那边，这条 `DECR` 是**执行了**、**没执行**，还是**无法确定**？
3. 如果是"无法确定"，业务代码该怎么办？
   - 当作失败，告诉用户"没抢到"——如果实际上扣成功了，会怎样？
   - 当作失败并重试一次——如果实际上扣成功了，会怎样？
4. 这类"不知道对方到底做没做"的问题，是分布式系统里最根本的难题之一。解决它的通用思路有两个关键词：**幂等**和**对账**。先记住这两个词，阶段 3、阶段 5 会展开。
5. 2 秒的读超时对秒杀接口来说是长还是短？一个请求如果在 Redis 上卡 2 秒，它占着的那个 HTTP 连接和 goroutine 也被卡 2 秒。高并发下这会造成什么连锁反应？

---

### Q6. 认证与密码 ⭐⭐

**看代码**：`docker-compose.yml` 的 `--requirepass`；`config.go` 里 Redis 的 `Password` 带着 `yaml:"-"`

1. Redis 默认是没有密码的。历史上大量 Redis 实例因为"无密码 + 监听公网"被入侵，攻击者通常能做到什么程度？（搜索"Redis 未授权访问"）
2. **动手**：`docker exec seckill-redis redis-cli -a 你的密码 ping`，注意输出里的 Warning。它在提醒什么？在一台多人共用的服务器上，别人用 `ps aux` 能看到什么？
3. 同理，compose 里 `--requirepass ${REDIS_PASSWORD}` 把密码放进了容器的启动命令。**动手**：`docker inspect seckill-redis | grep -i -A8 '"Cmd"'`。对本地开发环境这可以接受吗？生产环境通常怎么做？
4. Redis 6 引入了 ACL，可以创建多个用户并限制各自能执行的命令。如果要给"只读的监控程序"单独发一个账号，这有什么好处？

---

## 三、服务端

### Q7. DB 编号 ⭐⭐

**看代码**：`config.yaml` 的 `db: 0`；`integration_test.go` 的 `cfg.Redis.DB = 1`

1. Redis 默认有 16 个编号的 DB。集成测试用 DB 1 来和 DB 0 隔离。**动手**：在 redis-cli 里 `SET a 1`，`SELECT 1`，`GET a`，体会一下隔离。
2. 这种隔离有多"硬"？DB 0 里一条很慢的命令会不会卡住 DB 1 的请求？`FLUSHALL` 会清掉哪些 DB？
3. Redis Cluster（集群模式）支持多 DB 吗？所以如果生产环境用集群，"用 DB 编号区分环境/业务"的做法还成立吗？
4. 更通用的隔离方式是 **key 前缀**（如 `seckill:stock:{activityID}`）。设计一下：秒杀库存、用户已购标记这两类数据，你会怎么命名 key？一个好的 key 命名规范应该包含哪些部分？

---

### Q8. 持久化：重启后数据还在吗？ ⭐⭐⭐ 🎯

**看代码**：`docker-compose.yml` 的 `--appendonly yes` 和 `redis_data:/data`

1. Redis 数据在内存里，进程重启就没了——所以需要持久化。有 RDB 和 AOF 两种方式，各自的原理用一句话概括是什么？
2. **动手**：`SET hello world`，然后 `make docker-stop s=redis && make docker-start s=redis`，`GET hello` 还在吗？再进容器 `ls /data`，看到了什么文件？
3. AOF 有个关键参数 `appendfsync`，默认值是 `everysec`。**动手**：`CONFIG GET appendfsync`。它意味着机器突然断电时，最多丢失多久的数据？
4. 把这个结论放到秒杀场景：Redis 里的库存已经扣到 0，随后 Redis 宕机重启，丢了最后 1 秒的写入。恢复后的库存会比真实值**多**还是**少**？会导致什么业务后果？
5. 改成 `appendfsync always` 就能不丢，为什么大家通常不这么配？
6. 所以结论是：**Redis 里的库存能不能作为最终依据？**最后一道防线应该设在哪里？（提示：回到 Q1 第 3 问。阶段 2 用 MySQL 的 `UPDATE ... WHERE stock > 0` 防超卖，到了阶段 3 加上 Redis 之后，那条 SQL 还要不要保留？）

---

### Q9. 内存满了怎么办？ ⭐⭐

1. **动手**：`CONFIG GET maxmemory` 和 `CONFIG GET maxmemory-policy`。当前的值是什么含义？
2. 如果 Redis 被当作"缓存"用，内存满时淘汰一些 key 是合理的。常见的淘汰策略有哪几种？
3. 但如果 Redis 里存的是**库存计数**，它被淘汰了会怎样？所以存放"不能丢的数据"的 Redis 和存放"纯缓存"的 Redis，在淘汰策略上应该有什么不同？它们适合共用同一个实例吗？
4. key 可以设置过期时间（TTL）。秒杀活动结束后，那些库存 key、用户已购标记 key 如果不设过期也不清理，长期会怎样？

---

### Q10. 关闭顺序与 `Close` ⭐

**看代码**：`redis.go` 里 ping 失败时的 `client.Close()`；`main.go` 里的 `defer rdb.Close()`

1. `redis.New` 在 ping 失败时返回错误之前，先把 client 关掉了。不关会泄露什么？调用方拿到的是 `nil, err`，它自己还有机会去关吗？
2. 由此总结一条规则：**构造函数失败时，谁负责清理已经申请到一半的资源？**
3. `mysql.New` 遵守了同样的规则吗？对照着看一遍。
