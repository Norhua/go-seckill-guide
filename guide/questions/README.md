# 启发性问题库

这个目录按**知识方面**组织问题，每个方面一个子目录，每个阶段一个文件（`stage0.md`、`stage1.md`……）。
问题全部从你在 `implement/` 里亲手写下的代码出发，目的是把"照着写出来了"变成"我能讲清楚为什么"。

## 目录

| 目录 | 方面 | 阶段 0 涉及的内容 |
|---|---|---|
| [go-basics/](go-basics/stage0.md) | Go 语言基础 | error 体系、接口与嵌入、方法接收者、defer/panic/recover、struct tag、包与模块 |
| [go-concurrency/](go-concurrency/stage0.md) | Go 并发 | goroutine、channel、select、context、atomic、信号处理、竞态 |
| [network-http/](network-http/stage0.md) | 网络与 HTTP | 监听地址与端口、TCP 连接、HTTP 状态码、`http.Server` 超时、优雅停机、探针 |
| [gin/](gin/stage0.md) | Gin 框架 | Engine 与 `net/http` 的关系、中间件洋葱模型、`gin.Context`、统一响应 |
| [mysql-gorm/](mysql-gorm/stage0.md) | MySQL 与 GORM | DSN、连接池、GORM 与 `database/sql`、生成的 SQL、建表、迁移 |
| [redis/](redis/stage0.md) | Redis | 客户端连接池、超时、DB 编号、持久化、认证 |
| [docker/](docker/stage0.md) | Docker 与 Compose | 端口映射、数据卷、健康检查、环境变量插值、初始化脚本 |
| [engineering/](engineering/stage0.md) | 工程实践 | 配置管理、结构化日志、目录结构、Makefile、lint |
| [testing/](testing/stage0.md) | 测试 | 单元/集成测试、`httptest`、build tag、race 检测、测试隔离 |

## 建议的顺序

1. `go-basics` → `go-concurrency`：后面所有内容都建立在这两个之上
2. `network-http` → `gin`：先懂标准库和协议，再看框架做了什么
3. `mysql-gorm` → `redis` → `docker`
4. `engineering` → `testing`

不需要一口气做完。每个文件 10 多道题，一天一个文件比较合适。

## 怎么用

每道题的结构：

- **看代码**：先打开指到的文件和位置，重新读一遍那几行
- **问题链**：由浅入深的几个小问题。前面的问题是后面的台阶，按顺序想
- **动手**：一个几分钟就能做完的小实验。**先写下你的预测，再跑**——预测错了的地方就是你真正学到东西的地方
- **线索**：关键词或推荐阅读，卡住时再看

难度标记：⭐ 基础概念　⭐⭐ 需要理解原理　⭐⭐⭐ 需要读源码或文档深挖　🎯 面试高频

几条建议：

- 实验请在 `implement/` 里开一个临时分支做（`git switch -c playground`），做完丢弃即可，不要污染 `main`
- 文件里**故意不放答案**（GORM 生成的 SQL 例外，放在折叠块里供你核对）。把你的回答写下来，然后让 AI 助手 review；实在卡住，可以让 AI 助手只针对某一题讲解
- 有些题目指出了当前代码里**真实存在的小问题**。发现它们、判断要不要修，本身就是练习的一部分；值得修的可以记下来带到阶段 1
- 标了 🎯 的题目，试着脱稿用 1 分钟讲出来，这就是面试时的状态
