# 工程实践 · 阶段 0

> 涉及代码：`internal/config/`、`internal/logger/`、`cmd/api/main.go`、`Makefile`、`.golangci.yml`、整体目录结构
> 用法说明见 [../README.md](../README.md)
>
> 这组题没有"高深"的技术，但面试官判断一个项目是"玩具"还是"认真做的"，很大程度上看的就是这些。

---

## 一、配置管理

### Q1. 配置的优先级 ⭐⭐ 🎯

**看代码**：`config.go` 的 `Load`：默认值 → YAML → 环境变量 → Validate

1. 为什么需要三层？各举一个"最适合放在这一层"的配置项：
   - 只放代码默认值就够的
   - 应该放 YAML 的
   - 必须走环境变量的
2. 为什么是"环境变量覆盖 YAML"而不是反过来？设想同一个编译产物要部署到开发、测试、生产三套环境。
3. `yaml.Unmarshal(data, cfg)` 是往一个**已经填好默认值**的结构体里解析。如果 YAML 里没写 `server.port`，解析之后 `Port` 是 0 还是 8080？这个行为是怎么实现"YAML 只需要写想改的项"的？
4. 了解一下 *The Twelve-Factor App* 的第三条 **Config**（12factor.net/config，一页纸）。它主张配置放在哪里？理由是什么？我们的做法和它相比是严格遵循、还是折中？

---

### Q2. 为什么密码不许写进 YAML？ ⭐⭐ 🎯

**看代码**：`Password string` 上的 `` `yaml:"-"` ``；`TestLoad_YAMLAndEnvOverride` 里的 `password_in_yaml_should_be_ignored`

1. `configs/config.yaml` 是提交到 git 的。如果允许在里面写密码，早晚会发生什么？
2. 代码用 `yaml:"-"` 从**机制上**禁止了这件事，而不是在 README 里写一句"请勿在 YAML 中填写密码"。"靠机制"和"靠约定"的区别是什么？在这个项目里再找一个"靠机制保证"的例子。
3. 环境变量就绝对安全了吗？在 Linux 上，谁能看到一个进程的环境变量？（`/proc/<pid>/environ`）生产环境更进一步的做法有哪些？（知道 K8s Secret、Vault 这类名词即可）
4. `applyEnvOverrides` 里，`MYSQL_ROOT_PASSWORD` 和 `MYSQL_PASSWORD` 都会写入同一个字段，后者优先。生产环境的应用应该用 root 账号连数据库吗？应用账号应该有哪些权限、不该有哪些权限（比如 `DROP`）？

---

### Q3. 启动时校验 ⭐⭐

**看代码**：`config.go` 的 `Validate`

1. **动手**：`SERVER_PORT=99999 LOG_LEVEL=verbose make run`。报错信息里有几条错误？如果是"发现一个就返回"的写法，运维要改几次、启动几次？
2. **动手**：`SERVER_PORT=abc make run`。这个错误是 `Validate` 报的，还是更早的环节报的？
3. 如果完全不做校验，`read_timeout: 0s` 会导致什么？（回顾 network-http Q5）这种错误是启动时暴露好，还是运行一周后被攻击时暴露好？
4. `Validate` 现在没有校验 Redis 的三个超时、MySQL 的 `ConnMaxLifetime`。它们为 0 或负数时各自意味着什么？你认为需要补吗？
5. 配置在启动时加载一次，之后不再变。有些系统支持"热更新"（不重启就改配置）。如果 `Config` 要支持热更新，结合 go-concurrency 的知识，读配置的代码会立刻面临什么问题？

---

### Q4. 把配置当参数传，而不是全局变量 ⭐⭐ 🎯

**看代码**：`main.go` 里 `cfg` 的流向：`mysql.New(ctx, cfg.MySQL)`、`server.New(cfg.Server, log)`

1. 另一种常见写法是搞一个全局变量 `config.Global`，任何地方直接读 `config.Global.MySQL.Host`。参考项目 `reference/miaosha` 是哪种风格？
2. `mysql.New` 只接收 `config.MySQLConfig`，而不是整个 `*Config`。这对"读这个函数的人"传达了什么信息？
3. 全局变量写法下，`server_test.go` 里的 `newTestServer` 要怎么给测试准备一份特殊的配置？两个需要不同配置的测试能并行跑吗？
4. `main.go` 的注释说它"只做组装和启动"。它是整个程序里唯一一个知道"所有组件怎么连起来"的地方。这种手法叫**依赖注入**（这里是手动的）。它和你在其他语言里见过的 DI 框架（如 Spring）目的相同吗？Go 社区为什么更偏好手写？

---

## 二、日志

### Q5. 为什么要"结构化"日志？ ⭐⭐ 🎯

**看代码**：`middleware.go` 的 `AccessLog`；`config.yaml` 的 `log.format`

1. **动手**：分别用 `LOG_FORMAT=text` 和 `LOG_FORMAT=json` 启动，各发几个请求，对比输出。
2. 假设要回答"过去一小时，耗时超过 500ms 的请求有哪些路径"。面对下面两种日志，各自怎么查？

   ```
   2026/11/13 12:00:00 GET /api/buy 耗时 612ms 状态 200
   ```
   ```json
   {"time":"...","msg":"http access","method":"GET","path":"/api/buy","status":200,"latency":612000000}
   ```

3. `log.Info("user " + userID + " bought item " + itemID)` 和 `log.Info("purchase", slog.String("user_id", userID), slog.String("item_id", itemID))`，后者除了便于检索，在**日志聚合**（把同类日志归到一起统计）上还有什么优势？
4. 本地开发适合 text，线上适合 json，为什么？
5. JSON 输出里 `latency` 是个很大的整数。它的单位是什么？日志平台的使用者看到会困惑吗？有什么改进办法？

---

### Q6. 日志级别 ⭐

**看代码**：`logger.go` 的 `parseLevel`；`AccessLog` 里按状态码选择级别

1. Debug、Info、Warn、Error 四个级别，各自应该在什么情况下使用？给每个级别举一个秒杀系统里的例子。
2. 一个实用的判断标准："这条日志出现时，需要有人半夜起来处理吗？"按这个标准，"用户参数填错了""库存卖完了""MySQL 连不上了"分别该是什么级别？
3. 线上通常设为 info。此时代码里的 `log.Debug("stock", slog.Any("detail", buildBigObject()))` 不会输出，但 `buildBigObject()` 会不会被执行？`slog` 提供了什么手段避免这种浪费？（回想 `LogValuer`，以及 `log.Enabled()`）
4. `parseLevel` 对无法识别的值悄悄回退到 Info。但 `Validate` 已经拒绝了非法的 level。这两处是重复吗？谁是第一道防线、谁是兜底？`logger_test.go` 直接用 `NewWithOptions` 时，会经过 `Validate` 吗？

---

### Q7. slog 的 Handler 设计 ⭐⭐⭐

**看代码**：`logger.go` 的 `contextHandler`

1. `slog.Logger` 是"前端"，`slog.Handler` 是"后端"。它们各自负责什么？为什么要拆开？
2. `contextHandler` 包了一层 `baseHandler`，自己只加一个字段然后转交。这种"包一层、加点功能、再委托"的模式叫什么？你在 Gin 中间件里见过同样的结构吗？
3. 另一种给日志带上 request_id 的办法：在中间件里 `l := log.With("request_id", id)`，把 `l` 存进 context，业务代码用 `logger.FromContext(ctx).Info(...)`。`logger.go` 里其实也准备了 `WithContext`/`FromContext`。比较两种方案：
   - 业务代码的写法各是什么样？
   - 如果有人忘了从 ctx 取 logger、直接用了全局的，各自会丢失 request_id 吗？
4. `contextHandler` 方案要求调用方使用 `log.InfoContext(ctx, ...)` 而不是 `log.Info(...)`。在 `main.go` 和 `health.go` 里找一找，有没有用了不带 Context 版本的地方？哪些是合理的（确实没有请求上下文），哪些是可以改进的？

---

### Q8. 日志输出到哪？ ⭐⭐

**看代码**：`logger.go` 的 `Output: os.Stdout`

1. 应用把日志写到标准输出，而不是自己写文件、自己做按天切割。那日志最终怎么落盘、怎么被收集？（12factor.net/logs）
2. 应用自己管理日志文件有哪些麻烦？（磁盘写满、切割、多实例、容器重启后文件还在吗）
3. `Options.Output` 是 `io.Writer` 接口而不是 `*os.File`。这个设计让测试可以怎么做？去 `server_test.go` 里看 `logBuf` 是怎么用的。这体现了"面向接口编程"的什么好处？
4. 高并发下，日志本身会成为性能瓶颈吗？秒杀高峰每秒几万请求、每个请求一条访问日志，写 stdout 是同步的还是异步的？（先留个印象，阶段 3 压测时会实际碰到）

---

## 三、项目结构

### Q9. 包的依赖方向 ⭐⭐ 🎯

**看代码**：各个包的 `import` 部分

1. 画出当前的包依赖图：`main`、`config`、`logger`、`server`、`infra/mysql`、`infra/redis`，谁 import 了谁。图里有环吗？
2. Go 禁止循环依赖（import cycle），编译直接报错。假如 `config` 想在加载时打一条日志而 import 了 `logger`，会发生什么？（`logger` 现在 import 了谁？）
3. 被依赖最多的包是哪个？一个"被很多包依赖"的包，应该尽量稳定还是可以频繁修改？它自己应该依赖很多东西吗？
4. `logger.New(cfg config.LogConfig, ...)` 让 logger 包依赖了 config 包，而 `logger.NewWithOptions(Options{...})` 不依赖。如果要把 logger 包抽出去给另一个项目用，哪个函数是障碍？
5. 阶段 1 会加入 `handler`、`service`、`repository`、`model` 四层。预先想一想：它们之间的依赖方向应该是怎样的？`repository` 能 import `handler` 吗？`service` 应该知道 `gin.Context` 的存在吗？

---

### Q10. 错误码放在 `server` 包里合适吗？ ⭐⭐⭐

**看代码**：`internal/server/errcode.go`（里面有 `ErrSoldOut`、`ErrOrderNotFound` 这些业务错误）

1. 阶段 1 之后，`service` 层在库存不足时要返回 `ErrSoldOut`。按现在的位置，`service` 包就得 import `server` 包。而 `server` 包将来要注册路由、调用 handler，handler 又调用 service……这条依赖链有什么问题？
2. `ErrorCode` 里有个 `HTTPStatus` 字段。service 层是纯业务逻辑，它应该知道"HTTP 状态码"这个概念吗？如果将来同一个 service 还要通过 gRPC 或消息队列被调用呢？
3. 至少想出两种调整方案，并说出各自的代价。例如：
   - 把错误码挪到一个独立的、不依赖任何人的包（如 `internal/errs`）
   - service 层只定义纯粹的业务错误，由 handler 层负责翻译成 HTTP 状态码
4. 这个问题现在不需要动手改，但请带着它进入阶段 1，它会在第一次写 service 时就冒出来。

---

## 四、构建工具

### Q11. Makefile ⭐⭐

**看代码**：`Makefile`

1. `.PHONY` 是干什么的？**动手**：在项目根目录 `touch test`（创建一个名为 `test` 的空文件），暂时删掉 `.PHONY` 里的 `test`，执行 `make test`。发生了什么？为什么？做完记得还原。
2. Makefile 里的 `=`、`:=`、`?=` 三种赋值有什么区别？`MYSQL_PORT ?= 13306` 用 `?=` 的用意是什么？（它让 `MYSQL_PORT=3307 make migrate-up` 这种临时覆盖成为可能吗？）
3. 开头的 `-include .env` 前面那个减号是什么意思？紧跟着的单独一行 `export` 起什么作用？没有它，`make run` 启动的 Go 进程还读得到 `REDIS_PASSWORD` 吗？
4. 命令行前面的 `@`（如 `@mkdir -p $(BIN_DIR)`）是什么作用？
5. `MIGRATE := $(HOME)/.go/bin/migrate` 把工具路径写死成了你自己机器上的位置。别人 clone 之后大概率会遇到什么？有哪些改进方式？（直接写 `migrate` 依赖 PATH；用 `go run` 带版本号执行；Go 1.24+ 的 `go tool`）
6. Makefile 的价值不在于它有多强大，而在于**把"怎么做"固化下来**。对比"README 里写一长串命令让人复制"和"`make test-integration`"，后者好在哪？

---

### Q12. 版本号写死在代码里 ⭐⭐

**看代码**：`main.go` 的 `const version = "v0.1.0"`

1. 每条日志都带着 `version` 字段。线上同时存在新旧两个版本的实例时（发布过程中），这个字段有什么用？
2. 但它是手写的常量，发版时忘了改怎么办？Go 支持在**编译时**把值注入变量：`go build -ldflags "-X main.version=$(git describe --tags)"`。要用这个功能，`version` 必须是 `const` 还是 `var`？
3. **动手**：把 `version` 改成 `var`，用上面的命令编译并运行，看日志里的 version 变成了什么。
4. 比起人工维护的 `v0.1.0`，带上 git commit hash 的版本号在排查线上问题时有什么优势？

---

### Q13. Lint 在防什么？ ⭐⭐

**看代码**：`.golangci.yml`

1. `gofmt`、`go vet`、`golangci-lint` 三者的定位有什么不同？
2. 启用的 5 个 linter 里，`errcheck` 排第一。**动手**：在任意位置写一行 `os.Remove("/tmp/x")`（不处理返回值），执行 `make lint`，读报告。Go 语言本身为什么不强制处理返回的 error？
3. `check-blank: true` 连 `_ = f()` 这种显式忽略都禁止。这么严格值得吗？遇到"确实不关心错误"的场景（比如某些 `defer f.Close()`）该怎么写？
4. `check-type-assertions: true` 防的是什么运行时错误？回到 go-basics Q10 第 4 问。
5. Lint 规则应该在什么时候执行才有约束力？只靠"大家记得跑 `make lint`"可靠吗？了解两个概念：**pre-commit hook** 和 **CI**。
6. `go vet` 里有一个检查叫 `lostcancel`。它检查的是什么？回到 go-basics Q11 第 4 问。

---

### Q14. 提交历史也是作品的一部分 ⭐

**看代码**：`git log --oneline`

1. 你的提交信息形如 `feat(server): ...`、`chore(lint): ...`、`test: ...`。这套格式叫什么规范？`feat`、`fix`、`chore`、`refactor`、`docs`、`test` 各自什么时候用？
2. "每完成一个 Step 提交一次"和"阶段 0 全做完后一次性提交"，对于将来的你、以及看你仓库的面试官，分别意味着什么？
3. `implement/` 是外层仓库的一个 git submodule。submodule 在外层仓库里记录的是什么——文件内容，还是一个 commit 的指针？这就是为什么外层 `git status` 会显示 `M implement`。
