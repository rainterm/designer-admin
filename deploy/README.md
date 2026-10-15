# Designer Admin — 生产环境 Docker 部署

数据库与应用**分开**部署；应用在**本机用 JDK / Maven / Node 打包**后，再打成运行镜像。

## 目录

```
deploy/
├── install-env.sh          # 检查并安装 Docker / JDK / Maven / Node
├── deploy-db.sh            # 仅部署 MySQL + Redis
├── deploy-minio.sh         # 本地 MinIO（S3 兼容对象存储，可选）
├── deploy-app.sh           # 本地打包 + 构建镜像 + 启动应用
├── stop.sh / restart.sh / logs.sh
├── .env.example
├── docker-compose.db.yml
├── docker-compose.app.yml
├── build/                  # 本地打包产物暂存（自动生成）
├── server/Dockerfile
├── web/Dockerfile          # 管理后台 admin-web
└── nginx/default.conf      # 前端反代（/admin-api、/app-api、/captcha）
```

> **关于 `/captcha/`**：滑块验证码的端点是 AJ-Captcha 插件写死的 `/captcha/get`、`/captcha/check`，
> 挂在**根路径**上（不在 `/admin-api` 前缀下），所以 `nginx/default.conf` 里单独有一条
> `location /captcha/`（限流桶 `admin_captcha`）。少了它，请求会落到 `location /` 的
> `try_files` 上，拿到一个 HTML 而不是 JSON —— 症状是「验证码加载失败」而不是 404，
> 排查时容易走弯路。后端 dev 的 Vite `server.proxy` 里也补了同样一条。

## 推荐流程

```bash
cd designer-admin/deploy

# 1. 检查环境，缺什么装什么（Linux 建议 sudo）
./install-env.sh
# 仅检查不安装：
./install-env.sh --check

# 2. 配置
cp .env.example .env && vim .env

# 3. 启动数据库
./deploy-db.sh

# 4. 初始化数据库表（首次 / 重置时执行；破坏性操作，请勿对有数据的库重复执行）
(cd .. && bash sql/init/mysql/00-run-init.sh)

# 5. 打包并部署应用
./deploy-app.sh
```

`./install-env.sh` 会检查并按需安装：

| 组件 | 要求 |
|---|---|
| Docker + Compose | 运行容器 |
| JDK 25 (Temurin) | 后端编译 |
| Maven | 后端打包 |
| Node.js 18+（默认装 22） | 前端构建 |

支持 macOS（Homebrew）与 Ubuntu/Debian/RHEL 系。

| 入口 | 默认端口 | 说明 |
|---|---|---|
| 管理后台 | `APP_PORT=80` | admin-web，账号 `admin / admin123` |
| 后端直连 | `SERVER_PORT=8080` | 一般不必对外暴露 |

## 使用外部数据库

配置 `.env` 中 `DB_*` / `REDIS_*` 后只执行 `./deploy-app.sh`。建表请执行 `sql/init/mysql/00-run-init.sh`（按 `_base` → `system` / `demo` 分目录导入）；脚本支持交互输入，也可用 `DB_HOST/DB_PORT/DB_USER/DB_PASSWORD/DB_NAME` 环境变量免交互。

## 常用命令

| 命令 | 说明 |
|---|---|
| `./install-env.sh` | 检查并安装环境 |
| `./deploy-db.sh` | 启动 MySQL、Redis（不自动建表） |
| `./deploy-minio.sh` | 启动本地 MinIO（S3 API `9000` / Console `9001`，可选） |
| `./deploy-app.sh` | 本地打包并部署应用 |
| `./deploy-app.sh --skip-build` | 跳过打包，用已有 `deploy/build` |
| `./restart.sh app` / `./stop.sh app` | 重启 / 停止应用 |
| `./logs.sh server` | 后端日志（容器 stdout，受 compose `logging` 限制） |

## 日志

- **文件日志**：容器内 `/app/logs/designer-server.log`，挂载在 `server_logs` 卷（`docker volume inspect` 查看位置）。按「天 + 单文件 10MB」滚动，默认保留 `maxHistory=7` 天 / 总量上限 `totalSizeCap=1GB`。
- **容器 stdout**：`./logs.sh server` 看到的是 Docker 收集的标准输出（应用控制台日志 + 启动报错），由 `docker-compose.app.yml` 的 `logging.options.max-size=50m` / `max-file=5` 上限约束，最多约 250MB。
- **调整保留策略**：改后端配置 `solon.logging.appender.file.maxFileSize` / `maxHistory` / `totalSizeCap`，或用环境变量 `LOG_PATH` 覆盖文件路径（默认 `logs/designer-server`）。

## 架构

```
./install-env.sh  →  Docker + JDK + Maven + Node
本机打包：
  mvn -pl designer-server -am package
  designer-ui/admin-web → pnpm run build
产物同步到 deploy/build/{server,web}
Docker 运行镜像（JRE / Nginx）→ MySQL / Redis
```

## 注意事项

1. 修改 `.env` 示例密码；本机库时 `DB_PASSWORD` 与 `MYSQL_ROOT_PASSWORD` 一致。
2. macOS 安装 Docker Desktop 后需手动打开一次。
3. 日常发版只跑 `./deploy-app.sh`，不必动数据库。
4. 数据库表需手动初始化（`bash sql/init/mysql/00-run-init.sh`）；该脚本为**破坏性**（`DROP TABLE` 重建 + 插入种子数据），仅在首次或需要重置时执行，勿对有数据的库重复运行。
