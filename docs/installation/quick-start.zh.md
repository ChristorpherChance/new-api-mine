# nеw-аρi 快速上手安装指南

本文档介绍如何在本地或服务器上安装、配置并启动 **nеw-аρi**（QuаntumΝоuѕ）项目。涵盖 Docker Compose、Docker 单容器、源码编译、前端开发模式四种部署方式，并附环境变量说明与首次登录清单。

---

## 一、项目概览

**nеw-аρi** 是一个基于 Go 的 AI API 网关 / 代理，聚合 40+ 上游 LLM 提供商（OpenAI、Claude、Gemini、Azure、AWS Bedrock 等），统一接口对外提供，并内置用户管理、计费、限流、管理后台。

| 类别 | 内容 |
|------|------|
| 后端 | Go 1.25.1+、Gin、GORM v2 |
| 前端 | 双主题：`web/default`（React 19 + Rsbuild + Radix UI + Tailwind）、`web/classic`（React 18 + Vite + Semi UI）|
| 数据库 | SQLite / MySQL ≥ 5.7.8 / PostgreSQL ≥ 9.6（任选其一）|
| 缓存 | Redis（可选，推荐）+ 内存缓存 |
| 默认端口 | `3000` |
| 首次管理员账号 | 用户名 `root`，密码 `123456`（首次启动自动创建，**务必登录后立即修改**）|
| 前端包管理器 | **Bun**（推荐，优于 npm/yarn/pnpm）|

---

## 二、部署方式一：Docker Compose（推荐用于生产）

适用场景：希望开箱即用，并附带 PostgreSQL（或 MySQL）+ Redis 的完整生产部署。

### 1. 克隆仓库

```bash
git clone https://github.com/QuantumNous/new-api.git
cd new-api
```

### 2. 修改默认密码

编辑 `docker-compose.yml`，把所有 `123456`（PostgreSQL/MySQL 密码、Redis 密码等）替换成强密码，并同步更新 `SQL_DSN`、`REDIS_CONN_STRING`。

> **重要**：默认配置仅用于演示，**生产环境必须**更换全部默认凭证。

### 3. 启动服务

```bash
docker-compose up -d
```

### 4. 验证

- 浏览器访问 `http://<服务器IP>:3000`
- 使用 `root` / `123456` 登录，**立即**修改密码

### 5. 常用命令

```bash
docker-compose logs -f new-api    # 查看日志
docker-compose down               # 停止
docker-compose pull && docker-compose up -d   # 升级
```

---

## 三、部署方式二：Docker 单容器（最快上手，SQLite）

适用场景：个人使用 / 小流量 / 快速试用，无需独立数据库。

```bash
docker run --name new-api -d --restart always \
  -p 3000:3000 \
  -e TZ=Asia/Shanghai \
  -v ./data:/data \
  calciumion/new-api:latest
```

数据将持久化到当前目录下的 `./data/`（含 SQLite 数据库 `one-api.db`、日志等）。

如需使用 MySQL/PostgreSQL，追加 `-e SQL_DSN=...`，例如：

```bash
-e SQL_DSN="root:password@tcp(host:3306)/new-api"
```

---

## 四、部署方式三：源码编译

适用场景：需要二次开发、定制构建，或没有 Docker 环境的服务器。

### 1. 环境要求

- Go **1.25.1** 及以上
- Bun（前端构建）— 安装：`curl -fsSL https://bun.sh/install | bash`
- Git

### 2. 克隆并编译前端

```bash
git clone https://github.com/QuantumNous/new-api.git
cd new-api

# 编译 default 主题
cd web/default
bun install
bun run build
cd ../..

# 编译 classic 主题
cd web/classic
bun install
bun run build
cd ../..
```

> **关键**：`main.go` 中使用 `//go:embed web/default/dist` 和 `//go:embed web/classic/dist` 嵌入前端资源，因此**两个主题的 `dist` 目录都必须存在**，否则 Go 编译会失败。

### 3. 编译后端

```bash
go mod download
go build -ldflags "-s -w -X 'github.com/QuantumNous/new-api/common.Version=$(cat VERSION)'" -o new-api
```

### 4. 启动

```bash
# 默认 3000 端口、当前目录写入 SQLite
./new-api

# 指定端口与日志目录
./new-api --port 8080 --log-dir ./logs
```

可用启动参数：

| 参数 | 说明 |
|------|------|
| `--port <int>` | 监听端口（默认 3000）|
| `--log-dir <path>` | 日志目录（默认空，输出到标准输出）|
| `--version` | 打印版本信息后退出 |
| `--help` | 帮助 |

---

## 五、部署方式四：前端开发模式（HMR）

适用场景：调试前端，需要保留 React 热更新。

### 1. 启动后端（来自源码 / Docker）

```bash
docker compose -f docker-compose.dev.yml up -d
```

`docker-compose.dev.yml` 使用 `Dockerfile.dev`，跳过前端打包（注入占位 `dist/index.html`）以保证 `//go:embed` 通过。后端依然在 **3000** 端口运行。

### 2. 启动前端开发服务器

```bash
cd web/default
bun install
bun run dev
```

前端默认运行在 **3001** 端口，自动把 `/api`、`/mj`、`/pg` 代理到 `http://localhost:3000`（见 `web/default/rsbuild.config.ts`）。

浏览器访问 `http://localhost:3001` 即可使用带 HMR 的前端。

> 如果后端 URL 不是 `http://localhost:3000`，导出环境变量 `VITE_REACT_APP_SERVER_URL=https://<your-host>` 后再 `bun run dev`。

---

## 六、关键环境变量

完整列表见 `.env.example`。下表为最常用项：

| 变量 | 说明 | 默认/示例 |
|------|------|-----------|
| `PORT` | 监听端口 | `3000` |
| `SQL_DSN` | 数据库连接串。**留空则使用 SQLite** | MySQL: `user:pass@tcp(host:3306)/new-api`；PostgreSQL: `postgres://user:pass@host:5432/new-api?sslmode=disable` |
| `SQLITE_PATH` | SQLite 文件路径 | `./data/one-api.db` |
| `REDIS_CONN_STRING` | Redis 连接串 | `redis://default:password@host:6379/0` |
| `SESSION_SECRET` | Session 加密密钥（**必填且不能为字面量 `random_string`**）| 自定义随机字符串 |
| `CRYPTO_SECRET` | 字段级加密密钥（用于上游凭证等）| 自定义随机字符串 |
| `MEMORY_CACHE_ENABLED` | 启用内存缓存 | `true`（启用 Redis 时自动开启）|
| `BATCH_UPDATE_ENABLED` | 批量写入用量数据，多节点时建议开启 | `false` |
| `NODE_TYPE` | 节点类型：`master` / `slave` | `master` |
| `NODE_NAME` | 节点名称（多节点部署用）| 自定义 |
| `ERROR_LOG_ENABLED` | 持久化错误日志 | `false` |
| `ENABLE_PPROF` | 启用 pprof（监听 `:8005`）| `false` |
| `DEBUG` | Debug 模式 | `false` |
| `GIN_MODE` | `release` / `debug` | `release` |
| `STREAMING_TIMEOUT` | 流式响应超时（秒）| `60` |
| `CHANNEL_UPDATE_FREQUENCY` | 渠道自动更新频率（分钟）| 不设则关闭 |
| `TZ` | 时区 | `Asia/Shanghai` |

也可以创建项目根目录下的 `.env` 文件，启动时会通过 `godotenv.Load(".env")` 自动加载。

---

## 七、首次登录后必做清单

1. 使用 `root` / `123456` 登录后，**第一时间在「个人设置」中修改密码**。
2. 进入「渠道管理」，添加上游 LLM 提供商（OpenAI、Claude、Gemini 等），填入 API Key。
3. 在「令牌管理」中创建应用使用的访问令牌（Token）。
4. 在「系统设置」中按需开启：
   - 自动重试与失败迁移
   - 用量统计、计费倍率
   - 渠道自动测速 / 失效屏蔽
5. 检查 `docker-compose.yml` 与 `.env` 中的所有默认密码（数据库、Redis、`SESSION_SECRET`、`CRYPTO_SECRET`），全部改为强密码。
6. 如需多节点部署，将后续节点的 `NODE_TYPE=slave`，并指向同一数据库与 Redis。

---

## 八、客户端使用

部署完成后，客户端将原有的 OpenAI/Claude/Gemini base URL 替换为：

```
http://<your-host>:3000
```

将 API Key 替换为后台「令牌管理」生成的令牌，即可像调用原生 API 一样使用。

兼容路径示例：

| 协议 | 路径 |
|------|------|
| OpenAI 兼容 | `/v1/chat/completions`、`/v1/embeddings`、`/v1/images/generations` 等 |
| Claude 兼容 | `/v1/messages` |
| Gemini 兼容 | `/v1beta/models/...` |
| Midjourney | `/mj/...` |
| 任务类（Suno、Kling 等）| `/v1/...` |

---

## 九、常见问题

- **`go build` 报 `pattern web/default/dist: no matching files found`**：前端 `dist` 未生成。先执行第四节第 2 步的两次 `bun run build`。
- **首次登录显示密码错误**：确保使用全小写的 `root` 与 `123456`；如忘记修改后的密码，可参考官方文档的密码重置流程。
- **多节点部署时数据不同步**：必须配置同一个 Redis 与同一个数据库，并确保只有一个 `NODE_TYPE=master`。
- **流式响应被截断**：增大 `STREAMING_TIMEOUT`，并确认反向代理（Nginx 等）已禁用响应缓冲。

---

## 十、参考链接

- 官方文档：<https://docs.newapi.pro>
- 仓库主页：<https://github.com/QuantumNous/new-api>
- Issue 反馈：<https://github.com/QuantumNous/new-api/issues>
- 宝塔面板部署：见 `docs/installation/BT.md`

---

> 本文为快速上手指南。如需进阶配置（多节点、监控、定制主题、二次开发等），请参阅项目根目录的 `README.md`、`AGENTS.md` 与 `CLAUDE.md`。
