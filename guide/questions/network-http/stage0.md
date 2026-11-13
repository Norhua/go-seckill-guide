# 网络与 HTTP · 阶段 0

> 涉及代码：`internal/server/server.go`、`internal/server/health.go`、`internal/server/errcode.go`、`cmd/api/main.go`、`configs/config.yaml`
> 用法说明见 [../README.md](../README.md)
>
> 这一组题大量依赖 `curl -v` 做实验。`-v` 会打印完整的请求头和响应头，是学 HTTP 最好的工具。

---

## 一、地址与端口

### Q1. `0.0.0.0`、`127.0.0.1`、`localhost` ⭐ 🎯

**看代码**：`config.go` 默认值里 server 的 `Host: "0.0.0.0"`，而 MySQL/Redis 的 `Host: "127.0.0.1"`

1. 一台机器通常有多个网络接口（回环、Wi-Fi、Docker 虚拟网卡……）。服务端监听 `127.0.0.1:8080` 和监听 `0.0.0.0:8080`，区别是什么？
2. **动手**：查出你电脑的局域网 IP（`ipconfig getifaddr en0`），分别在 `SERVER_HOST=127.0.0.1` 和 `SERVER_HOST=0.0.0.0` 下启动服务，用 `curl http://<局域网IP>:8080/healthz` 测试。哪种通，哪种不通？
3. 为什么同一份配置里，**服务端监听**用 `0.0.0.0`，而**客户端连接** MySQL 用 `127.0.0.1`？"连接到 `0.0.0.0`"这句话有意义吗？
4. 在咖啡店的公共 Wi-Fi 上跑一个监听 `0.0.0.0` 的开发服务，有什么风险？
5. `localhost` 和 `127.0.0.1` 完全等价吗？（提示：`localhost` 是个名字，需要解析；它还可能解析成 `::1`）

---

### Q2. 端口 ⭐

**看代码**：`config.go` 的 `Validate` 里 `c.Server.Port < 1 || c.Server.Port > 65535`

1. 为什么端口上限是 65535？这个数字从哪来？
2. 1024 以下的端口有什么特殊之处？为什么开发时习惯用 8080 而不是 80？
3. **动手**：开两个终端都执行 `make run`。第二个报什么错？这个错误最终是从 `main.go` 的哪个 `case` 分支冒出来的？
4. 校验规则不允许端口为 0。但在操作系统层面，监听端口 0 是合法的，它代表什么？这个特性在测试里很有用（`httptest.NewServer` 就是这么做的）。
5. 一条 TCP 连接由哪四个值唯一确定？所以"一个端口只能有一个连接"这个说法对吗？一台服务器的 8080 端口理论上能同时承载多少条连接，瓶颈在哪？

---

## 二、从 TCP 到 HTTP

### Q3. 一次 `curl` 背后 ⭐⭐ 🎯

**动手**：`curl -v http://localhost:8080/healthz`，对着输出回答：

1. `* Connected to localhost (127.0.0.1) port 8080` 这一行出现之前，客户端和服务端之间完成了什么？（TCP 三次握手，能说出三个包各是什么即可）
2. 以 `>` 开头的是请求，以 `<` 开头的是响应。一个 HTTP/1.1 请求由哪三部分组成？
3. 响应头里的 `X-Request-Id` 是谁加的？`Content-Type` 和 `Content-Length` 呢？
4. 握手这件事发生在你的 Go 代码里吗？当你的 handler 开始执行时，内核和 `net/http` 已经替你做完了哪些事？
5. 最后一行 `* Connection #0 to host localhost left intact` 是什么意思？连接没关，留着干什么？（引出 Q4）

**线索**：如果 TCP 三次握手/四次挥手完全陌生，推荐小林 coding《图解网络》的 TCP 篇，读"三次握手"和"四次挥手"两节即可。

---

### Q4. Keep-Alive 与 `IdleTimeout` ⭐⭐

**看代码**：`server.go` 的 `IdleTimeout: cfg.IdleTimeout`（默认 60s）

1. HTTP/1.0 每个请求都新建一条 TCP 连接。这样做在高并发下的代价是什么？HTTP/1.1 默认怎么改进的？
2. **动手**：`curl -v http://localhost:8080/healthz http://localhost:8080/readyz`（一条命令两个 URL）。第二个请求的输出里有一行 `Re-using existing connection`，说明了什么？
3. `IdleTimeout` 控制的是什么？设成 1 小时会怎样？设成 1 秒呢？各自的代价是什么？
4. 记住这个概念：**连接复用**。接下来 MySQL 连接池、Redis 连接池的设计动机和它一模一样。

---

### Q5. `http.Server` 的三个超时 ⭐⭐⭐ 🎯

**看代码**：`server.go` 的 `ReadTimeout`、`WriteTimeout`、`IdleTimeout`；《step8》里讲过"为什么不用 `r.Run()`"

1. 如果三个超时都不设（Gin 的 `r.Run()` 就是如此），一个恶意客户端建立连接后每 10 秒只发 1 个字节，会占用服务端什么资源？开 1 万个这样的连接呢？这种攻击叫什么名字？
2. 在一条时间线上画出：建立连接 → 读请求头 → 读请求体 → handler 执行 → 写响应 → 空闲等待下一个请求。把 `ReadTimeout`、`WriteTimeout`、`IdleTimeout` 各自覆盖的区间标上去。还有一个 `ReadHeaderTimeout`，它和 `ReadTimeout` 的区别是什么？
3. **动手**（重要）：`WriteTimeout` 是 10s。把 `handleDebugSlow` 的 sleep 临时改成 15s，然后 `curl -v localhost:8080/debug/slow`。
   - 客户端在第几秒看到了什么？是一个 HTTP 错误状态码吗？
   - 服务端日志里 `slow request finished` 还会打印吗？访问日志记录的状态码是多少？
4. 根据实验结果：`WriteTimeout` 到期后，handler 所在的 goroutine 被终止了吗？如果 handler 里是一条慢 SQL，它会停下来吗？
5. 所以要让"超时后真的停止工作"，光靠 `WriteTimeout` 不够，还需要什么机制配合？（回忆 context 的职责）

**线索**：Cloudflare 博客 *The complete guide to Go net/http timeouts*，里面有 Q2 要画的那张图。

---

## 三、HTTP 语义

### Q6. 状态码 ⭐⭐ 🎯

**看代码**：`errcode.go` 的全部错误码定义

1. 2xx、3xx、4xx、5xx 分别代表什么？4xx 和 5xx 最本质的区别是"谁的责任"——这个区别对**客户端要不要重试**、**你的告警要不要响**分别意味着什么？
2. `AccessLog` 对 `status >= 500` 打 Error、`>= 400` 打 Warn。为什么 4xx 不值得 Error 级别？
3. `ErrUnauthorized`（401）和 `ErrForbidden`（403）的区别是什么？"没登录"和"登录了但不是你的订单"分别对应哪个？
4. `ErrSoldOut` 被映射为 409 Conflict。"库存卖完"算不算一种"错误"？试着为以下三种设计各找一条理由，然后选一个你认同的：
   - A. HTTP 200 + 业务码 30002（很多国内项目的做法）
   - B. HTTP 409 + 业务码 30002（当前做法）
   - C. HTTP 200，`data` 里放 `{"success": false, "reason": "sold_out"}`
5. 延伸思考：秒杀开始后 99% 的请求都是"卖完了"。如果监控系统按"非 2xx 比例"报警，方案 B 会发生什么？这应该通过改状态码解决，还是改监控规则解决？
6. `ErrTooManyRequests`（429）和 `ErrServiceUnavailable`（503）都可以表示"现在处理不了"。从客户端视角，二者表达的含义有什么不同？有一个响应头可以告诉客户端"过多久再试"，叫什么？

---

### Q7. 为什么既有 HTTP 状态码，又有业务码？ ⭐⭐

**看代码**：`errcode.go` 的 `Code` 与 `HTTPStatus` 两个字段；`response.go` 的 `Response` 结构

1. HTTP 状态码只有几十个。`ErrSoldOut`、`ErrLimitExceeded`、`ErrOrderStateInvalid` 都是 409，前端怎么区分它们来显示不同的提示？
2. 为什么前端应该根据 `code` 做判断，而不是根据 `msg` 的文字内容？
3. 错误码分了号段（1xxxx 通用、3xxxx 秒杀……）。这有什么好处？如果系统将来拆成多个服务呢？
4. `Msg` 的注释写着"绝不能包含 SQL 报错等内部细节"。把 `Duplicate entry 'xxx' for key 'uk_request_id'` 这样的原始错误返回给客户端，攻击者能从中得到什么信息？

---

### Q8. 404 与 405 ⭐⭐

**看代码**：`server.go` 的 `engine.HandleMethodNotAllowed = true`、`NoRoute`、`NoMethod`

1. **动手**：`curl -i -X POST localhost:8080/healthz` 与 `curl -i localhost:8080/nope`，对比两者的状态码和响应体。
2. 把 `HandleMethodNotAllowed = true` 注释掉再试一次 POST。状态码变成了什么？哪个更准确地描述了"路径存在但方法不对"？
3. HTTP 规范要求 405 响应带一个响应头，告诉客户端该路径支持哪些方法。它叫什么？看看你的 405 响应里有没有。
4. GET 和 POST 在语义上的核心区别是什么？了解两个词：**安全（safe）**、**幂等（idempotent）**。"秒杀下单"接口该用哪个方法？它天然幂等吗？用户手抖点了两次会怎样？（阶段 2 的核心问题之一，先种个种子）

---

## 四、优雅停机

### Q9. `Shutdown` 到底做了哪几件事？ ⭐⭐⭐ 🎯

**看代码**：`server.go` 的 `Shutdown`；`main.go` 的停机流程

1. 不做优雅停机、直接让进程退出，正在处理中的请求会怎样？客户端看到的是什么？如果这个请求正执行到"Redis 库存已扣、订单还没写"，后果是什么？
2. 去读 `http.Server.Shutdown` 的文档注释，它按顺序做了三件事，分别是什么？
3. 文档里有一句："`Shutdown` 被调用后，`Serve`、`ListenAndServe` 会**立即**返回 `ErrServerClosed`。请确保程序不会就此退出，而是等待 `Shutdown` 返回。"对照 `main.go`：是哪段代码保证了这一点？假如把 `ser.Start()` 放在主 goroutine、把 `Shutdown` 放在另一个 goroutine，会掉进这个坑吗？
4. `shutdown_timeout` 是 10s。如果有个请求要 30s 才能完成，10s 到了之后 `Shutdown` 返回什么？那个请求的最终命运是什么？`Shutdown` 和 `Close` 的区别是什么？
5. 文档还说 `Shutdown` 不会等待某类连接（如 WebSocket）。是哪一类？
6. **动手**：`curl localhost:8080/debug/slow`，在它返回前 Ctrl+C。然后在另一个终端立刻 `curl localhost:8080/healthz`。新请求得到的是 503，还是 `Connection refused`？为什么？

---

### Q10. 停机的顺序为什么是这样？ ⭐⭐ 🎯

**看代码**：`main.go` 从 `ser.MarkShuttingDown()` 到函数结束

1. 当前的顺序是：标记停机 → 关 HTTP server（等在途请求）→ 关 Redis → 关 MySQL。如果先关 MySQL 再关 HTTP server，在途请求会遇到什么？
2. 总结一条通用原则：资源的关闭顺序和它们的**依赖方向**是什么关系？和初始化顺序又是什么关系？
3. `MarkShuttingDown()` 之后紧接着就调 `Shutdown()`，监听端口立刻被关了。那么"让 `/readyz` 返回 503"这一步，在当前实现里**有机会被任何人观察到吗**？
4. 在 K8s/负载均衡的环境里，从"`/readyz` 开始返回 503"到"负载均衡器真的不再转发流量"之间有延迟。生产环境通常在这两步之间加什么？（存档 step10 思考题 2 讲过，试着自己复述）

---

## 五、健康检查

### Q11. liveness 和 readiness 为什么要分开？ ⭐⭐ 🎯

**看代码**：`health.go` 的 `handleHealthz` 与 `handleReadyz`

1. `/healthz` 失败时，编排系统（如 K8s）会做什么？`/readyz` 失败时呢？
2. 假设把 MySQL 检查也放进 `/healthz`。某天 MySQL 抖动了 30 秒，你的 10 个服务实例会经历什么？MySQL 恢复之后又会发生什么？这种"一个故障引发更大故障"的现象叫什么？
3. **动手**：`make docker-stop s=redis`，然后分别 `curl -i` 两个端点。再 `make docker-start s=redis`，看多久恢复。
4. 观察上一步 `/readyz` 的响应体：`checks` 里出现了类似 `dial tcp 127.0.0.1:16379: connect: connection refused` 的内容。对比 Q7 第 4 问的原则，这算不算内部信息泄露？你会怎么处理？（提示：探针端点通常对谁开放？有没有折中方案？）

---

### Q12. readiness 检查本身的质量 ⭐⭐⭐

**看代码**：`handleReadyz` 的 `for name, checker := range s.checkers` 循环

1. 多个 checker 是**串行**执行的，共用同一个 2 秒超时的 ctx。如果 MySQL 的检查卡了 1.9 秒，轮到 Redis 时还剩多少时间？此时 Redis 明明是好的，报告里它会是什么状态？
2. 怎么改能让它们互不影响？（回到 go-concurrency，这是一个很小、很适合练手的并发改造）
3. 探针一般每隔几秒被调用一次。每次都真的去 ping MySQL 和 Redis，在平时没问题。但如果 `/readyz` 被暴露到了公网、被人高频调用呢？有什么廉价的保护办法？
4. `ctx` 是从 `c.Request.Context()` 派生的。如果探针的调用方等了 1 秒就断开连接，正在进行的 ping 会怎样？这是你想要的行为吗？
