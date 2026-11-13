# Docker 与 Compose · 阶段 0

> 涉及代码：`deploy/docker-compose.yml`、`deploy/mysql/init.sql`、`.env` / `.env.example`、`Makefile` 的 `docker-*` 目标
> 用法说明见 [../README.md](../README.md)

---

## 一、基本概念

### Q1. 容器到底是什么？ ⭐⭐ 🎯

1. 你没有在 macOS 上安装 MySQL，却能连上一个 MySQL。它到底运行在哪里？
2. 镜像（image）和容器（container）是什么关系？用"类与对象"或"程序与进程"做个类比。
3. 容器和虚拟机的本质区别是什么？容器里的进程，在宿主机内核看来是什么？关键词：**namespace**（隔离）和 **cgroup**（限额），各用一句话说明它们负责什么。
4. Docker 的容器依赖 Linux 内核的特性，但你用的是 macOS。那 Docker Desktop 在 Mac 上是怎么跑起来的？
5. **动手**：`docker exec seckill-mysql ps aux`（如果镜像里没有 `ps`，换成 `docker top seckill-mysql`）。容器里有几个进程？PID 为 1 的是谁？

---

### Q2. 镜像标签 ⭐

**看代码**：`image: mysql:8.0` 和 `image: redis:7-alpine`

1. 如果写成 `image: mysql`（不带标签），拉下来的是哪个版本？半年后同事在新电脑上执行同样的 compose 文件，拉到的还是同一个版本吗？
2. `mysql:8.0` 就完全确定了吗？`8.0.36` 和 `8.0.40` 都匹配这个标签。要做到"完全可重复"，可以把标签写到多细？（了解一下 digest：`mysql@sha256:...`）
3. `redis:7-alpine` 的 `alpine` 是什么？它和不带 alpine 的镜像相比，好处和可能的坑各是什么？

---

## 二、网络

### Q3. 端口映射 ⭐⭐ 🎯

**看代码**：`ports: - "${MYSQL_PORT:-13306}:3306"`

1. 冒号左右两个数字分别是谁的端口？
2. 为什么宿主机一侧用 13306 这种"不标准"的端口，而不直接用 3306？
3. 容器内的 MySQL 监听的是容器自己的网络接口。从你的 Mac 发往 `127.0.0.1:13306` 的数据，是怎么到达容器里的 3306 的？
4. **动手**：`docker port seckill-mysql`，注意输出里的 `0.0.0.0:13306`。结合 network-http Q1：这意味着和你在同一局域网的人能连上你的 MySQL 吗？怎么写可以只允许本机访问？（提示：`ports` 支持 `"127.0.0.1:13306:3306"` 这种三段式写法）

---

### Q4. 容器之间怎么互相访问？ ⭐⭐⭐

现在 Go 服务跑在宿主机上，通过 `127.0.0.1:13306` 访问 MySQL。假设将来把 Go 服务也做成容器，放进同一个 compose 文件。

1. 在 Go 服务的容器里，`127.0.0.1` 指的是谁？还能用 `127.0.0.1:13306` 连到 MySQL 吗？
2. 那 `MYSQL_HOST` 和 `MYSQL_PORT` 应该分别配成什么？（提示：compose 会为每个项目创建一个网络，并提供基于**服务名**的 DNS）端口该用 13306 还是 3306？
3. 这就解释了为什么 `host` 和 `port` 必须是**可配置**的而不是写死在代码里——同一份代码，在"宿主机直跑"和"容器里跑"两种环境下，这两个值都不一样。
4. **动手**：`docker network ls` 找到本项目的网络，`docker network inspect <网络名>`，看看两个容器各自的内网 IP。然后验证服务名 DNS：`docker exec seckill-redis ping -c 1 mysql`。

---

## 三、数据

### Q5. 数据卷与 init 脚本 ⭐⭐ 🎯

**看代码**：

```yaml
volumes:
  - mysql_data:/var/lib/mysql
  - ./mysql/init.sql:/docker-entrypoint-initdb.d/01-init.sql:ro
```

1. 这两行是两种不同的挂载方式。第一行叫**命名卷（named volume）**，第二行叫**绑定挂载（bind mount）**。各自的数据实际存在哪里？各自适合什么用途？
2. 如果不挂 `mysql_data` 卷，`make docker-down && make docker-up` 之后数据还在吗？容器的文件系统在容器被删除时会怎样？
3. `make docker-stop`、`make docker-down`、`make docker-down-v` 三者分别删除了什么、保留了什么？填表：

   | 命令 | 容器 | 网络 | 数据卷 |
   |---|---|---|---|
   | `stop` | | | |
   | `down` | | | |
   | `down -v` | | | |

4. 现在回答 [mysql-gorm Q15](../mysql-gorm/stage0.md) 的实验：改了 `init.sql` 之后重新 `docker-up`，为什么新库没出现？`/docker-entrypoint-initdb.d/` 里的脚本在什么条件下才会执行？（去 Docker Hub 的 mysql 镜像说明页找 *Initializing a fresh instance* 一节）
5. 所以要让 `init.sql` 的修改生效，该用哪条命令？代价是什么？这也从另一个角度说明了：为什么"建表"不该放在 init 脚本里，而要用迁移工具？
6. 末尾的 `:ro` 是什么意思？去掉它有什么风险？
7. 文件名前缀 `01-` 有什么作用？

---

## 四、配置与健康检查

### Q6. 环境变量插值 ⭐⭐

**看代码**：`${MYSQL_ROOT_PASSWORD:?请在 .env 中配置 MYSQL_ROOT_PASSWORD}` 和 `${MYSQL_DATABASE:-seckill}`

1. `${VAR:-default}` 和 `${VAR:?message}` 分别是什么含义？这套语法是 compose 发明的，还是从别处借来的？
2. 为什么"数据库名"给了默认值，而"密码"选择了"没配就报错"？如果密码也给一个默认值（比如 `123456`），长期看会发生什么？
3. **动手**：临时把 `.env` 改个名，执行 `make docker-up`，看报错信息。
4. 这些 `${...}` 是在什么时候、由谁替换成真实值的——是容器里的 MySQL 进程，还是宿主机上的 `docker compose` 命令？**动手**：`docker compose --env-file .env -f deploy/docker-compose.yml config`，看渲染后的结果。
5. `Makefile` 里的 `COMPOSE` 显式写了 `--env-file .env`。如果不写，compose 默认会去哪个目录找 `.env`？（注意 compose 文件在 `deploy/` 下，而 `.env` 在项目根目录）

---

### Q7. `.env` 的三个读者 ⭐⭐

**看代码**：`.env.example`、`.gitignore`、`Makefile` 开头的 `-include .env` 和 `export`

1. `.env` 不提交到 git，`.env.example` 提交。两个文件各自的作用是什么？新同事 clone 仓库后第一步该做什么？
2. 万一有人不小心把带真实密码的 `.env` commit 并 push 了，之后再提交一次"删除 .env"够不够？为什么？正确的补救措施是什么？
3. 同一个 `.env` 文件被三个不同的程序读取：Docker Compose、Make、（间接地）你的 Go 进程。梳理一下 `make run` 时，`MYSQL_PASSWORD` 这个值是经过哪几步传到 `config.Load` 的 `os.LookupEnv` 手里的。
4. 这三个读者对 `.env` 的语法理解**并不完全一致**。如果密码是 `p@ss#word$1`，写成 `REDIS_PASSWORD=p@ss#word$1`，Make 会怎么理解 `#` 和 `$`？Compose 呢？（知道"有坑"即可。实践建议：开发环境的密码只用字母和数字）

---

### Q8. 健康检查与 `--wait` ⭐⭐⭐ 🎯

**看代码**：两个服务的 `healthcheck` 段；`Makefile` 的 `docker-up: $(COMPOSE) up -d --wait`

1. 容器"启动了（running）"和服务"可用了（healthy）"是一回事吗？MySQL 容器从启动到真正能接受连接，中间在做什么？大概要多久？
2. 不加 `--wait` 时，`docker compose up -d` 会在什么时刻返回？紧接着执行 `make migrate-up` 可能会遇到什么？
3. `interval`、`timeout`、`retries`、`start_period` 各自的含义是什么？`start_period: 15s` 期间检查失败算不算数？
4. MySQL 的检查命令特意写了 `-h 127.0.0.1` 而不是默认的 `localhost`。背景知识：MySQL 镜像首次初始化时，会先启动一个**只监听 Unix socket、不监听 TCP** 的临时实例来执行 init 脚本，完成后再重启为正式实例。而 `mysqladmin` 对 `localhost` 默认走 socket。据此推理：如果用 `localhost`，健康检查可能在什么时刻"过早地"报告健康？后果是什么？
5. **动手**：`docker inspect --format '{{json .State.Health}}' seckill-mysql`，看看最近几次检查的记录。
6. 把这里的 healthcheck 和你自己写的 `/readyz` 做个对比：两者解决的是不是同一类问题？

---

### Q9. 重启策略 ⭐

**看代码**：`restart: unless-stopped`

1. 可选值有 `no`、`always`、`on-failure`、`unless-stopped`。`always` 和 `unless-stopped` 的区别体现在什么时候？（提示：你手动 `stop` 了容器，然后重启了电脑）
2. 如果 MySQL 因为配置错误启动即崩溃，`unless-stopped` 会导致什么现象？怎么观察到？（`make docker-ps`、`make docker-logs s=mysql`）
3. `container_name: seckill-mysql` 固定了容器名。好处是命令好敲；代价是什么？（想想能不能在同一台机器上同时启动这个项目的两份副本）

---

### Q10. 开发环境 vs 生产环境 ⭐⭐

1. 这个 compose 文件只编排了 MySQL 和 Redis，Go 服务本身是用 `make run` 在宿主机上跑的。这种"依赖容器化、应用本机跑"的开发方式有什么好处？
2. 生产环境的 MySQL 会像这样用单个 Docker 容器 + 一个数据卷来运行吗？至少列举三项这个 compose 配置**没有**考虑的生产需求。
3. 阶段 3 会往 compose 里加消息队列，阶段 6 会加 Prometheus、Grafana。依赖越来越多之后，"一条 `make docker-up` 拉起全部环境"的价值体现在哪里？
