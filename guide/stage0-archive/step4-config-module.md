# User:
---
我正在第0阶，指导我完成Step 4
---
# Gemini 4 Argon (High):
---
下面是 **阶段 0 - Step 4：配置模块 (`internal/config`)** 的实现指南。

> [!TIP]
> 我注意到 [implement/](file:///Users/amadeus/work-space/learning/go-seckill/implement) 下目前还没有创建 `internal/` 和 `configs/` 目录，在开始写代码前可以先创建好对应目录。

---

### 一、对比参考项目：我们要改进什么？

你可以先看一眼参考项目的 [`config.go`](file:///Users/amadeus/work-space/learning/go-seckill/reference/miaosha/backend/internal/config/config.go) 和 [`config.yaml`](file:///Users/amadeus/work-space/learning/go-seckill/reference/miaosha/backend/configs/config.yaml)，它在工程实践上有几个明显问题，我们在实现时直接修正：

1. **去掉包级全局变量 `var Cfg *Config`**（参考项目第 148 行）：全局变量会导致单元测试之间互相污染，后续我们统一通过函数参数（依赖注入）传递 `*config.Config`。
2. **敏感配置在结构体上加 `yaml:"-"`**：参考项目的 `config.yaml` 里直接写了 `password: root123`，且漏掉了 `REDIS_PASSWORD`。我们在结构体字段上标注 `yaml:"-"`，**从机制上阻止 `yaml.Unmarshal` 读取密码**，即使有人误把密码写进 YAML 也会被忽略，强制只从环境变量加载。
3. **时间配置直接使用 `time.Duration`**：参考项目混用了 `int`（如 `ConnMaxLifetimeSec`）和 `string`（如 `CodeTTL`）。`gopkg.in/yaml.v3` 原生支持将 `"5s"`、`"10m"` 解析为 `time.Duration`，直接传给 `http.Server`、GORM `sql.DB` 和 `go-redis` 更安全直观。
4. **按阶段演进（YAGNI）**：阶段 0 只定义 `Server`、`MySQL`、`Redis`、`Log` 四个模块，不在第 0 阶段塞入 Kafka、雪花算法、邮件等尚未用到的配置。

---

### 二、实现步骤

#### 1. 创建目录并安装 YAML 依赖

在 `implement/` 目录下执行：

```bash
mkdir -p internal/config configs cmd/api
go get gopkg.in/yaml.v3
```

---

#### 2. 编写 `internal/config/config.go`

按 **“代码默认值 → YAML 文件覆盖 → 环境变量覆盖 → `Validate()` 校验”** 的四步流水线组织代码：

##### （1）结构体定义与 `DSN()` 方法

```go
package config

import (
	"errors"
	"fmt"
	"os"
	"strconv"
	"strings"
	"time"

	"gopkg.in/yaml.v3"
)

type Config struct {
	Server ServerConfig `yaml:"server"`
	MySQL  MySQLConfig  `yaml:"mysql"`
	Redis  RedisConfig  `yaml:"redis"`
	Log    LogConfig    `yaml:"log"`
}

type ServerConfig struct {
	Host            string        `yaml:"host"`
	Port            int           `yaml:"port"`
	ReadTimeout     time.Duration `yaml:"read_timeout"`
	WriteTimeout    time.Duration `yaml:"write_timeout"`
	IdleTimeout     time.Duration `yaml:"idle_timeout"`
	ShutdownTimeout time.Duration `yaml:"shutdown_timeout"`
}

func (s ServerConfig) Addr() string {
	return fmt.Sprintf("%s:%d", s.Host, s.Port)
}

type MySQLConfig struct {
	Host            string        `yaml:"host"`
	Port            int           `yaml:"port"`
	Username        string        `yaml:"username"`
	Password        string        `yaml:"-"` // 敏感信息：禁止从 YAML 读取，只从环境变量注入
	Database        string        `yaml:"database"`
	MaxOpenConns    int           `yaml:"max_open_conns"`
	MaxIdleConns    int           `yaml:"max_idle_conns"`
	ConnMaxLifetime time.Duration `yaml:"conn_max_lifetime"`
	ConnMaxIdleTime time.Duration `yaml:"conn_max_idle_time"`
}

// DSN 生成供 GORM mysql.Open 使用的连接串
func (m MySQLConfig) DSN() string {
	return fmt.Sprintf("%s:%s@tcp(%s:%d)/%s?charset=utf8mb4&parseTime=True&loc=Local",
		m.Username, m.Password, m.Host, m.Port, m.Database,
	)
}

type RedisConfig struct {
	Host         string        `yaml:"host"`
	Port         int           `yaml:"port"`
	Password     string        `yaml:"-"` // 敏感信息：禁止从 YAML 读取，只从环境变量注入
	DB           int           `yaml:"db"`
	PoolSize     int           `yaml:"pool_size"`
	MinIdleConns int           `yaml:"min_idle_conns"`
	DialTimeout  time.Duration `yaml:"dial_timeout"`
	ReadTimeout  time.Duration `yaml:"read_timeout"`
	WriteTimeout time.Duration `yaml:"write_timeout"`
}

func (r RedisConfig) Addr() string {
	return fmt.Sprintf("%s:%d", r.Host, r.Port)
}

type LogConfig struct {
	Level  string `yaml:"level"`  // debug, info, warn, error
	Format string `yaml:"format"` // text, json
}
```

> [!NOTE]
> 这里把 `Redis` 拆成 `Host` + `Port` 并提供 `Addr()` 方法，是为了和你目前 [`.env.example`](file:///Users/amadeus/work-space/learning/go-seckill/implement/.env.example) 中的 `REDIS_PORT=16379` 保持一致，避免一边配 `REDIS_PORT` 一边又配 `REDIS_ADDR` 造成混乱。

---

##### （2）默认值与三层加载逻辑

在同一个文件中继续实现 `defaultConfig()`、`Load(path string)` 和 `applyEnvOverrides(cfg *Config)`：

```go
func defaultConfig() *Config {
	return &Config{
		Server: ServerConfig{
			Host:            "0.0.0.0",
			Port:            8080,
			ReadTimeout:     5 * time.Second,
			WriteTimeout:    10 * time.Second,
			IdleTimeout:     60 * time.Second,
			ShutdownTimeout: 10 * time.Second,
		},
		MySQL: MySQLConfig{
			Host:            "127.0.0.1",
			Port:            13306,
			Username:        "root",
			Database:        "seckill",
			MaxOpenConns:    50,
			MaxIdleConns:    25,
			ConnMaxLifetime: time.Hour,
			ConnMaxIdleTime: 10 * time.Minute,
		},
		Redis: RedisConfig{
			Host:         "127.0.0.1",
			Port:         16379,
			DB:           0,
			PoolSize:     50,
			MinIdleConns: 10,
			DialTimeout:  3 * time.Second,
			ReadTimeout:  2 * time.Second,
			WriteTimeout: 2 * time.Second,
		},
		Log: LogConfig{
			Level:  "info",
			Format: "text",
		},
	}
}

// Load 按「代码默认值 → YAML 文件 → 环境变量覆盖 → Validate」顺序加载配置。
// 当 path 为空字符串时，跳过 YAML 文件读取（方便纯环境变量启动或单测）。
func Load(path string) (*Config, error) {
	cfg := defaultConfig()

	if path != "" {
		data, err := os.ReadFile(path)
		if err != nil {
			return nil, fmt.Errorf("read config file %q: %w", path, err)
		}
		if err := yaml.Unmarshal(data, cfg); err != nil {
			return nil, fmt.Errorf("unmarshal config file %q: %w", path, err)
		}
	}

	if err := applyEnvOverrides(cfg); err != nil {
		return nil, err
	}

	if err := cfg.Validate(); err != nil {
		return nil, fmt.Errorf("invalid config: %w", err)
	}

	return cfg, nil
}
```

对于 `applyEnvOverrides(cfg *Config)`，你可以参考 [`reference/miaosha/backend/internal/config/config.go`](file:///Users/amadeus/work-space/learning/go-seckill/reference/miaosha/backend/internal/config/config.go#L169-L309) 的辅助函数写法，支持：
- `setStringFromEnv(key string, target *string)`
- `setIntFromEnv(key string, target *int)`
- `setDurationFromEnv(key string, target *time.Duration)`（内部用 `time.ParseDuration`）

环境变量映射建议如下（与你的 [`.env.example`](file:///Users/amadeus/work-space/learning/go-seckill/implement/.env.example) 对齐）：
- **Server**：`SERVER_HOST`、`SERVER_PORT`、`SERVER_READ_TIMEOUT`、`SERVER_WRITE_TIMEOUT`、`SERVER_IDLE_TIMEOUT`、`SERVER_SHUTDOWN_TIMEOUT`
- **MySQL**：`MYSQL_HOST`、`MYSQL_PORT`、`MYSQL_USER`、`MYSQL_DATABASE`
  - 密码：优先读 `MYSQL_PASSWORD`，如果未设置也可兼容读取 `MYSQL_ROOT_PASSWORD`（或者在 `.env` 中明确加上 `MYSQL_PASSWORD`）
- **Redis**：`REDIS_HOST`、`REDIS_PORT`、`REDIS_PASSWORD`、`REDIS_DB`
- **Log**：`LOG_LEVEL`、`LOG_FORMAT`

---

##### （3）`Validate()` 校验逻辑

要求**报错必须指明具体字段名**。可以使用标准库 `errors.Join` 把所有不合法的字段一次性收集起来返回：

```go
func (c *Config) Validate() error {
	var errs []error

	// 1. Server 校验
	if c.Server.Port < 1 || c.Server.Port > 65535 {
		errs = append(errs, fmt.Errorf("server.port must be between 1 and 65535, got %d", c.Server.Port))
	}
	if c.Server.ReadTimeout <= 0 {
		errs = append(errs, errors.New("server.read_timeout must be > 0"))
	}
	if c.Server.WriteTimeout <= 0 {
		errs = append(errs, errors.New("server.write_timeout must be > 0"))
	}
	if c.Server.IdleTimeout <= 0 {
		errs = append(errs, errors.New("server.idle_timeout must be > 0"))
	}
	if c.Server.ShutdownTimeout <= 0 {
		errs = append(errs, errors.New("server.shutdown_timeout must be > 0"))
	}

	// 2. MySQL 校验
	if strings.TrimSpace(c.MySQL.Host) == "" {
		errs = append(errs, errors.New("mysql.host is required"))
	}
	if c.MySQL.Port < 1 || c.MySQL.Port > 65535 {
		errs = append(errs, fmt.Errorf("mysql.port must be between 1 and 65535, got %d", c.MySQL.Port))
	}
	if strings.TrimSpace(c.MySQL.Username) == "" {
		errs = append(errs, errors.New("mysql.username is required"))
	}
	if strings.TrimSpace(c.MySQL.Password) == "" {
		errs = append(errs, errors.New("mysql.password (env MYSQL_PASSWORD) is required"))
	}
	if strings.TrimSpace(c.MySQL.Database) == "" {
		errs = append(errs, errors.New("mysql.database is required"))
	}
	if c.MySQL.MaxOpenConns <= 0 {
		errs = append(errs, errors.New("mysql.max_open_conns must be > 0"))
	}
	if c.MySQL.MaxIdleConns < 0 || c.MySQL.MaxIdleConns > c.MySQL.MaxOpenConns {
		errs = append(errs, errors.New("mysql.max_idle_conns must be between 0 and max_open_conns"))
	}

	// 3. Redis 校验
	if strings.TrimSpace(c.Redis.Host) == "" {
		errs = append(errs, errors.New("redis.host is required"))
	}
	if c.Redis.Port < 1 || c.Redis.Port > 65535 {
		errs = append(errs, fmt.Errorf("redis.port must be between 1 and 65535, got %d", c.Redis.Port))
	}
	if strings.TrimSpace(c.Redis.Password) == "" {
		errs = append(errs, errors.New("redis.password (env REDIS_PASSWORD) is required"))
	}
	if c.Redis.DB < 0 {
		errs = append(errs, errors.New("redis.db must be >= 0"))
	}
	if c.Redis.PoolSize <= 0 {
		errs = append(errs, errors.New("redis.pool_size must be > 0"))
	}

	// 4. Log 校验
	switch strings.ToLower(c.Log.Level) {
	case "debug", "info", "warn", "error":
	default:
		errs = append(errs, fmt.Errorf("log.level must be one of [debug, info, warn, error], got %q", c.Log.Level))
	}
	switch strings.ToLower(c.Log.Format) {
	case "text", "json":
	default:
		errs = append(errs, fmt.Errorf("log.format must be one of [text, json], got %q", c.Log.Format))
	}

	return errors.Join(errs...)
}
```

---

#### 3. 编写 `configs/config.yaml`

在 `implement/configs/config.yaml` 中放入非敏感配置（**不要出现 `password` 字段**）：

```yaml
server:
  host: "0.0.0.0"
  port: 8080
  read_timeout: 5s
  write_timeout: 10s
  idle_timeout: 60s
  shutdown_timeout: 10s

mysql:
  host: "127.0.0.1"
  port: 13306
  username: "root"
  database: "seckill"
  max_open_conns: 50
  max_idle_conns: 25
  conn_max_lifetime: 1h
  conn_max_idle_time: 10m

redis:
  host: "127.0.0.1"
  port: 16379
  db: 0
  pool_size: 50
  min_idle_conns: 10
  dial_timeout: 3s
  read_timeout: 2s
  write_timeout: 2s

log:
  level: "info"
  format: "text"
```

---

#### 4. 编写单元测试 `internal/config/config_test.go`

在测试中使用 Go 标准库的 `t.Setenv(key, val)` 设置环境变量（测试结束后会自动清理还原），并用 `t.TempDir()` 创建临时 YAML 文件。至少覆盖以下 4 个测试用例：

1. **`TestLoad_Defaults`**：`Load("")` 不传 YAML 文件，只通过 `t.Setenv` 设置 `MYSQL_PASSWORD` 和 `REDIS_PASSWORD`，断言默认值（如端口 `8080`、超时 `5s`）生效。
2. **`TestLoad_YAMLAndEnvOverride`**：
   - 在临时 YAML 文件里写 `server.port: 9000`、`mysql.database: yaml_db`，甚至故意写一行 `mysql.password: leaked_in_yaml`。
   - 同时用 `t.Setenv("SERVER_PORT", "9090")` 覆盖端口。
   - 断言最终生效的是：`Port == 9090`（环境变量优先于 YAML）、`Database == "yaml_db"`（YAML 优先于默认值），且如果没设 `MYSQL_PASSWORD` 环境变量，YAML 里的 `leaked_in_yaml` 不会生效（被 `yaml:"-"` 忽略）。
3. **`TestLoad_MissingRequiredSecret`**：将 `MYSQL_PASSWORD` 设为空，调用 `Load("")`，断言返回 `error != nil` 且错误信息包含 `"mysql.password"`。
4. **`TestValidate_InvalidValues`**：构造端口越界（如 `Port: 70000`）、超时为负数（`ReadTimeout: -1s`）、非法日志级别（`Level: "trace"`），断言 `Validate()` 返回错误并包含对应字段名。

---

### 三、完成标准自检

写完后在 `implement/` 下运行：

```bash
go test -v -race ./internal/config/...
```

全部 PASS 后，别忘了把 [guide/stage0-todolist.md](file:///Users/amadeus/work-space/learning/go-seckill/guide/stage0-todolist.md#L82-L93) 中 Step 4 的复选框勾上，并在 `implement/` 下提交一次 Git commit。完成后告诉我，我帮你 review 代码或继续指导 Step 5！
---
# User:
---
applyEnvOverrides应该怎么写
---
# Gemini 4 Argon (High):
---
`applyEnvOverrides` 的核心思路是：**只覆盖环境变量中真正设置了的项；没设置的保持原值（即 YAML 或默认值）不变；如果环境变量格式非法（比如端口写了 `"abc"`），直接报错并指出是哪个环境变量**。

结合你现在的 [`internal/config/config.go`](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/config/config.go)，可以分两步完成：

### 1. 在 `import` 中补充 `"strconv"`

```go
import (
	"errors"
	"fmt"
	"os"
	"strconv" // <-- 新增：用于把环境变量字符串转成 int
	"strings"
	"time"

	"gopkg.in/yaml.v3"
)
```

### 2. 实现 `applyEnvOverrides` 与 3 个类型解析辅助函数

把下面这段代码加到 [`internal/config/config.go`](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/config/config.go) 末尾即可：

```go
func applyEnvOverrides(cfg *Config) error {
	var errs []error

	// 1. Server
	setStringFromEnv("SERVER_HOST", &cfg.Server.Host)
	errs = append(errs,
		setIntFromEnv("SERVER_PORT", &cfg.Server.Port),
		setDurationFromEnv("SERVER_READ_TIMEOUT", &cfg.Server.ReadTimeout),
		setDurationFromEnv("SERVER_WRITE_TIMEOUT", &cfg.Server.WriteTimeout),
		setDurationFromEnv("SERVER_IDLE_TIMEOUT", &cfg.Server.IdleTimeout),
		setDurationFromEnv("SERVER_SHUTDOWN_TIMEOUT", &cfg.Server.ShutdownTimeout),
	)

	// 2. MySQL
	setStringFromEnv("MYSQL_HOST", &cfg.MySQL.Host)
	setStringFromEnv("MYSQL_USER", &cfg.MySQL.Username)
	setStringFromEnv("MYSQL_DATABASE", &cfg.MySQL.Database)
	// 优先读 MYSQL_PASSWORD；若未设置，兼容读取 docker-compose 用的 MYSQL_ROOT_PASSWORD
	setStringFromEnv("MYSQL_ROOT_PASSWORD", &cfg.MySQL.Password)
	setStringFromEnv("MYSQL_PASSWORD", &cfg.MySQL.Password)
	errs = append(errs,
		setIntFromEnv("MYSQL_PORT", &cfg.MySQL.Port),
		setIntFromEnv("MYSQL_MAX_OPEN_CONNS", &cfg.MySQL.MaxOpenConns),
		setIntFromEnv("MYSQL_MAX_IDLE_CONNS", &cfg.MySQL.MaxIdleConns),
		setDurationFromEnv("MYSQL_CONN_MAX_LIFETIME", &cfg.MySQL.ConnMaxLifetime),
		setDurationFromEnv("MYSQL_CONN_MAX_IDLE_TIME", &cfg.MySQL.ConnMaxIdleTime),
	)

	// 3. Redis
	setStringFromEnv("REDIS_HOST", &cfg.Redis.Host)
	setStringFromEnv("REDIS_PASSWORD", &cfg.Redis.Password)
	errs = append(errs,
		setIntFromEnv("REDIS_PORT", &cfg.Redis.Port),
		setIntFromEnv("REDIS_DB", &cfg.Redis.DB),
		setIntFromEnv("REDIS_POOL_SIZE", &cfg.Redis.PoolSize),
		setIntFromEnv("REDIS_MIN_IDLE_CONNS", &cfg.Redis.MinIdleConns),
		setDurationFromEnv("REDIS_DIAL_TIMEOUT", &cfg.Redis.DialTimeout),
		setDurationFromEnv("REDIS_READ_TIMEOUT", &cfg.Redis.ReadTimeout),
		setDurationFromEnv("REDIS_WRITE_TIMEOUT", &cfg.Redis.WriteTimeout),
	)

	// 4. Log
	setStringFromEnv("LOG_LEVEL", &cfg.Log.Level)
	setStringFromEnv("LOG_FORMAT", &cfg.Log.Format)

	return errors.Join(errs...)
}

// setStringFromEnv 当环境变量存在且非空时，覆盖目标字符串
func setStringFromEnv(name string, target *string) {
	if v, ok := os.LookupEnv(name); ok {
		if trimmed := strings.TrimSpace(v); trimmed != "" {
			*target = trimmed
		}
	}
}

// setIntFromEnv 当环境变量存在且非空时，解析为 int 并覆盖目标值
func setIntFromEnv(name string, target *int) error {
	v, ok := os.LookupEnv(name)
	if !ok {
		return nil
	}
	trimmed := strings.TrimSpace(v)
	if trimmed == "" {
		return nil
	}
	n, err := strconv.Atoi(trimmed)
	if err != nil {
		return fmt.Errorf("invalid env %s=%q: %w", name, v, err)
	}
	*target = n
	return nil
}

// setDurationFromEnv 当环境变量存在且非空时，按 "5s"、"10m" 格式解析为 time.Duration
func setDurationFromEnv(name string, target *time.Duration) error {
	v, ok := os.LookupEnv(name)
	if !ok {
		return nil
	}
	trimmed := strings.TrimSpace(v)
	if trimmed == "" {
		return nil
	}
	d, err := time.ParseDuration(trimmed)
	if err != nil {
		return fmt.Errorf("invalid env %s=%q: %w", name, v, err)
	}
	*target = d
	return nil
}
```

### 为什么这样写？

1. **`errors.Join(errs...)` 简化错误处理**：`errors.Join` 会自动忽略 `nil` 的 error，只有当某个 `setIntFromEnv` / `setDurationFromEnv` 转换失败时才会返回错误，省去了参考项目里满屏的 `if err != nil { return err }`。
2. **`os.LookupEnv` vs `os.Getenv`**：如果环境变量没设或设为空串（比如 `SERVER_PORT=""`），保持 YAML 或代码默认值不变，不会误把 `int` 覆盖成 `0`。
3. **兼容 [`.env.example`](file:///Users/amadeus/work-space/learning/go-seckill/implement/.env.example)**：先读 `MYSQL_ROOT_PASSWORD` 再读 `MYSQL_PASSWORD`，这样你本地直接加载 `.env` 时不用重复配两遍密码，而在生产或测试环境单独指定 `MYSQL_PASSWORD` 时以 `MYSQL_PASSWORD` 为准。
---
# User:
---
给我完整的单元测试代码
---
# Gemini 4 Argon (High):
---
下面是完整的单元测试代码，直接保存为 `implement/internal/config/config_test.go` 即可（仅依赖 Go 标准库，无需额外安装第三方测试库）：

```go
package config

import (
	"os"
	"path/filepath"
	"strings"
	"testing"
	"time"
)

// setRequiredSecrets 统一设置必填的敏感环境变量，并清空可能干扰测试的宿主机环境变量
func setRequiredSecrets(t *testing.T) {
	t.Helper()
	t.Setenv("MYSQL_ROOT_PASSWORD", "")
	t.Setenv("MYSQL_PASSWORD", "test_mysql_pwd")
	t.Setenv("REDIS_PASSWORD", "test_redis_pwd")
}

// 1. 验证代码默认值生效
func TestLoad_Defaults(t *testing.T) {
	setRequiredSecrets(t)

	cfg, err := Load("")
	if err != nil {
		t.Fatalf("Load(\"\") unexpected error: %v", err)
	}

	if cfg.Server.Port != 8080 {
		t.Errorf("expected default server.port=8080, got %d", cfg.Server.Port)
	}
	if cfg.Server.ReadTimeout != 5*time.Second {
		t.Errorf("expected default server.read_timeout=5s, got %v", cfg.Server.ReadTimeout)
	}
	if cfg.MySQL.Port != 13306 {
		t.Errorf("expected default mysql.port=13306, got %d", cfg.MySQL.Port)
	}
	if cfg.MySQL.Password != "test_mysql_pwd" {
		t.Errorf("expected mysql.password from env, got %q", cfg.MySQL.Password)
	}
	if cfg.Redis.Port != 16379 {
		t.Errorf("expected default redis.port=16379, got %d", cfg.Redis.Port)
	}
	if cfg.Log.Level != "info" || cfg.Log.Format != "text" {
		t.Errorf("unexpected default log config: %+v", cfg.Log)
	}
}

// 2. 验证加载优先级：默认值 < YAML < 环境变量，且 YAML 中的密码会被忽略
func TestLoad_YAMLAndEnvOverride(t *testing.T) {
	setRequiredSecrets(t)

	yamlContent := `
server:
  port: 9000
  read_timeout: 8s
mysql:
  host: "10.0.0.1"
  database: "seckill_yaml"
  password: "password_in_yaml_should_be_ignored"
redis:
  db: 2
  password: "redis_yaml_pwd_should_be_ignored"
log:
  level: "debug"
  format: "json"
`
	tmpFile := filepath.Join(t.TempDir(), "config.yaml")
	if err := os.WriteFile(tmpFile, []byte(yamlContent), 0o600); err != nil {
		t.Fatalf("write temp yaml failed: %v", err)
	}

	// 用环境变量覆盖 YAML 中的 server.port 和 log.level
	t.Setenv("SERVER_PORT", "9090")
	t.Setenv("LOG_LEVEL", "warn")

	cfg, err := Load(tmpFile)
	if err != nil {
		t.Fatalf("Load(%q) unexpected error: %v", tmpFile, err)
	}

	// 环境变量覆盖了 YAML (9000 -> 9090, debug -> warn)
	if cfg.Server.Port != 9090 {
		t.Errorf("expected env override server.port=9090, got %d", cfg.Server.Port)
	}
	if cfg.Log.Level != "warn" {
		t.Errorf("expected env override log.level=warn, got %q", cfg.Log.Level)
	}

	// YAML 覆盖了代码默认值
	if cfg.Server.ReadTimeout != 8*time.Second {
		t.Errorf("expected yaml server.read_timeout=8s, got %v", cfg.Server.ReadTimeout)
	}
	if cfg.MySQL.Host != "10.0.0.1" || cfg.MySQL.Database != "seckill_yaml" {
		t.Errorf("expected yaml mysql host/db override, got host=%q db=%q", cfg.MySQL.Host, cfg.MySQL.Database)
	}
	if cfg.Redis.DB != 2 {
		t.Errorf("expected yaml redis.db=2, got %d", cfg.Redis.DB)
	}
	if cfg.Log.Format != "json" {
		t.Errorf("expected yaml log.format=json, got %q", cfg.Log.Format)
	}

	// 未在 YAML 和环境变量中出现的字段，保留代码默认值
	if cfg.MySQL.MaxOpenConns != 50 {
		t.Errorf("expected default mysql.max_open_conns=50, got %d", cfg.MySQL.MaxOpenConns)
	}
}

// 3. 验证即使 YAML 写了 password，只要环境变量没提供密码，仍然会报错（证明 yaml:"-" 生效）
func TestLoad_MissingRequiredSecret(t *testing.T) {
	t.Setenv("MYSQL_ROOT_PASSWORD", "")
	t.Setenv("MYSQL_PASSWORD", "")
	t.Setenv("REDIS_PASSWORD", "")

	yamlContent := `
mysql:
  password: "leaked_yaml_password"
redis:
  password: "leaked_yaml_password"
`
	tmpFile := filepath.Join(t.TempDir(), "config.yaml")
	if err := os.WriteFile(tmpFile, []byte(yamlContent), 0o600); err != nil {
		t.Fatalf("write temp yaml failed: %v", err)
	}

	_, err := Load(tmpFile)
	if err == nil {
		t.Fatal("expected error when MYSQL_PASSWORD and REDIS_PASSWORD are missing, got nil")
	}

	errMsg := err.Error()
	if !strings.Contains(errMsg, "mysql.password") {
		t.Errorf("expected error to mention 'mysql.password', got: %v", errMsg)
	}
	if !strings.Contains(errMsg, "redis.password") {
		t.Errorf("expected error to mention 'redis.password', got: %v", errMsg)
	}
}

// 4. 验证环境变量类型格式非法时报错
func TestLoad_InvalidEnvFormat(t *testing.T) {
	setRequiredSecrets(t)
	t.Setenv("SERVER_PORT", "not_a_number")

	_, err := Load("")
	if err == nil {
		t.Fatal("expected error for invalid SERVER_PORT, got nil")
	}
	if !strings.Contains(err.Error(), "SERVER_PORT") {
		t.Errorf("expected error to mention 'SERVER_PORT', got: %v", err)
	}
}

// 5. 验证 Validate() 对非法取值的拦截与字段提示
func TestValidate_InvalidFields(t *testing.T) {
	cfg := defaultConfig()
	cfg.MySQL.Password = "pwd"
	cfg.Redis.Password = "pwd"

	// 人为构造多处非法配置
	cfg.Server.Port = 70000          // 端口越界
	cfg.Server.ReadTimeout = 0       // 超时 <= 0
	cfg.MySQL.MaxIdleConns = 100     // Idle > Open (50)
	cfg.Redis.DB = -1                // DB < 0
	cfg.Log.Level = "verbose"        // 非法日志级别
	cfg.Log.Format = "xml"           // 非法日志格式

	err := cfg.Validate()
	if err == nil {
		t.Fatal("expected validation error, got nil")
	}

	errMsg := err.Error()
	expectedFields := []string{
		"server.port",
		"server.read_timeout",
		"mysql.max_idle_conns",
		"redis.db",
		"log.level",
		"log.format",
	}
	for _, field := range expectedFields {
		if !strings.Contains(errMsg, field) {
			t.Errorf("expected validation error to contain field %q, got:\n%s", field, errMsg)
		}
	}
}

// 6. 验证 Addr() 和 DSN() 辅助方法格式正确
func TestHelperMethods(t *testing.T) {
	cfg := defaultConfig()
	cfg.MySQL.Password = "secret123"

	if got := cfg.Server.Addr(); got != "0.0.0.0:8080" {
		t.Errorf("Server.Addr() = %q, want %q", got, "0.0.0.0:8080")
	}
	if got := cfg.Redis.Addr(); got != "127.0.0.1:16379" {
		t.Errorf("Redis.Addr() = %q, want %q", got, "127.0.0.1:16379")
	}
	wantDSN := "root:secret123@tcp(127.0.0.1:13306)/seckill?charset=utf8mb4&parseTime=True&loc=Local"
	if got := cfg.MySQL.DSN(); got != wantDSN {
		t.Errorf("MySQL.DSN() = %q, want %q", got, wantDSN)
	}
}
```

### 运行测试

把上面的文件保存到 `implement/internal/config/config_test.go`（别忘了先把上一条回复中的 `applyEnvOverrides` 加进 [`config.go`](file:///Users/amadeus/work-space/learning/go-seckill/implement/internal/config/config.go)），然后在 `implement/` 目录下执行：

```bash
go test -v -race ./internal/config/...
```

6 个测试全部 `PASS` 后，Step 4 就完成了！完成后随时告诉我，我们继续进入 **Step 5：日志**。
---
# User:
---
完成这一步后我该如何创建git commit
---
# Gemini 3.8 Flash (High):
---
刚刚运行了测试，`internal/config` 下的 6 个单元测试已全部通过（包含竞态检测 `-race`）！

目前在 [implement/](file:///Users/amadeus/work-space/learning/go-seckill/implement) 目录下，除了 Step 4 的配置模块外，还包含前面 Step 2（工程骨架）和 Step 3（Docker 编排）尚未提交的文件。

为了让 Git 提交记录规范、像一个成熟开源项目（面试时查看 commit 历史会很加分），推荐遵循 **[Conventional Commits](https://www.conventionalcommits.org/)** 规范。你可以根据需要选择 **拆分提交（推荐）** 或 **一次性提交**：

### 方案 A：分步提交（推荐，提交历史最清晰）

把基础设施与配置模块分开，形成两个清晰的逻辑提交：

#### 第一步：提交 Step 2 & Step 3 的基础设施与 Docker 编排
```bash
cd implement

# 1. 暂存 Step 2 & 3 相关文件
git add .gitignore .env.example deploy/

# 2. 提交
git commit -m "feat(infra): setup docker-compose for mysql and redis with healthcheck"
```

#### 第二步：提交 Step 4 配置模块与单元测试
```bash
# 1. 暂存 Step 4 相关文件
git add go.mod go.sum configs/ internal/config/

# 2. 提交（包含多行详细说明）
git commit -m "feat(config): implement configuration loading, validation and tests

- Support Code Defaults -> YAML -> Env override order
- Exclude sensitive secrets from YAML via yaml:\"-\" tag
- Add Validate() method with descriptive field errors
- Cover default, override, missing secret and validation cases in unit tests"
```

### 方案 B：一次性提交当前所有改动

如果你想合并为一个阶段性提交，可以执行：

```bash
cd implement

# 1. 查看暂存状态，确认 .env 敏感文件没有被误添加（应该被 .gitignore 忽略）
git status

# 2. 添加所有变更文件
git add .

# 3. 提交
git commit -m "feat(core): setup docker infra and internal/config module

- Add docker-compose for MySQL 8 and Redis 7 with healthchecks
- Implement config loading with default -> YAML -> env hierarchy
- Enforce secret-injection via environment variables
- Add config validation and complete unit tests"
```

### 提交后的检查与推送

```bash
# 查看提交历史
git log --oneline -n 3

# 推送到远端仓库（如果有配置远程仓库）
git push origin main
```

完成提交后，我们就可以直接进入 **Step 5：基于 `log/slog` 的结构化日志模块** 了！准备好时告诉我即可。
---
