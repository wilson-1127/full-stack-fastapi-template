# CLAUDE.md

本文件为 Claude Code（claude.ai/code）在此仓库中工作时提供指导。

## 项目概览

全栈 Web 应用模板：**FastAPI**（Python）后端 + **React**（TypeScript）前端，使用 **Docker Compose** 编排。PostgreSQL 通过 SQLModel ORM 访问，JWT 认证，基于邮件的密码找回，Traefik 反向代理，前端使用 Tailwind CSS + shadcn/ui。

## 整体架构

```
                          ┌──────────────────────────────────────────┐
                          │              Internet / 客户端            │
                          └──────────┬───────────────────────────────┘
                                     │
                                     ▼
                          ┌──────────────────────────────────────────┐
                          │     Traefik 反向代理 (compose.yml)        │
                          │  • api.${DOMAIN}      → backend:8000     │
                          │  • dashboard.${DOMAIN} → frontend:80      │
                          │  • adminer.${DOMAIN}  → adminer:8080      │
                          └──────┬────────────────────┬──────────────┘
                                 │                    │
                   ┌─────────────▼──┐          ┌──────▼──────────────┐
                   │   Frontend     │          │      Backend        │
                   │  (React + TS)  │          │  (FastAPI + Python) │
                   │   Port: 80     │          │    Port: 8000       │
                   │                │          │                     │
                   │ ┌────────────┐ │          │  ┌───────────────┐  │
                   │ │ TanStack   │ │  HTTP    │  │  api_router   │  │
                   │ │ Router     │ │◄───────►│  │  /api/v1       │  │
                   │ │ + Query    │ │  REST    │  │                │  │
                   │ └────────────┘ │          │  │ • login        │  │
                   │                │          │  │ • users CRUD   │  │
                   │ ┌────────────┐ │          │  │ • items CRUD   │  │
                   │ │ 自动生成    │ │          │  │ • utils        │  │
                   │ │ API 客户端  │ │          │  │ • private      │  │
                   │ └────────────┘ │          │  └───────┬───────┘  │
                   │                │          │          │          │
                   │ ┌────────────┐ │          │  ┌───────▼───────┐  │
                   │ │ shadcn/ui  │ │          │  │   CRUD 层     │  │
                   │ │ Tailwind   │ │          │  │ (纯函数)      │  │
                   │ │ React Hook │ │          │  └───────┬───────┘  │
                   │ │ Form+Zod   │ │          │          │          │
                   │ └────────────┘ │          │  ┌───────▼───────┐  │
                   └────────────────┘          │  │  SQLModel ORM │  │
                                               │  │  (models.py)  │  │
                                               │  └───────┬───────┘  │
                                               │          │          │
                                               │  ┌───────▼───────┐  │
                                               │  │  SQLAlchemy   │  │
                                               │  │  Engine       │  │
                                               │  └───────────────┘  │
                                               └─────────┬───────────┘
                                                         │
                                                         │ SQL
                                                         ▼
                                               ┌──────────────────┐
                                               │   PostgreSQL 18  │
                                               │    (db 服务)     │
                                               │    Port: 5432    │
                                               └──────────────────┘

辅助服务（仅本地开发 compose.override.yml）:
  ┌─────────────────┐     ┌──────────────────────┐
  │  Adminer        │     │  Mailcatcher         │
  │  DB 管理界面    │     │  邮件调试工具         │
  │  Port: 8080     │     │  Web UI Port: 1080   │
  └─────────────────┘     └──────────────────────┘

CI/CD (GitHub Actions):
  ┌─────────────────┬──────────────────┬───────────────────┬──────────────┐
  │ test-backend    │ test-docker-     │ playwright        │ pre-commit   │
  │ (pytest)        │ compose (集成)   │ (E2E 前端)        │ (lint/format)│
  └─────────────────┴──────────────────┴───────────────────┴──────────────┘
                                          │
                              ┌───────────┴───────────┐
                              ▼                       ▼
                    ┌──────────────────┐   ┌──────────────────┐
                    │ deploy-staging   │   │ deploy-production│
                    │ (自托管 runner)  │   │ (自托管 runner)  │
                    └──────────────────┘   └──────────────────┘
```

## 开发命令

### 启动完整技术栈

```bash
docker compose watch
```

启动所有服务：后端（端口 8000）、前端（端口 5173）、PostgreSQL、Adminer（端口 8080）、Mailcatcher（端口 1080）。

### 本地开发（不通过 Docker 运行单个服务）

停止某个 Docker 服务并在本地原生运行以加快迭代速度：

```bash
# 前端（先停止 Docker 服务：docker compose stop frontend）
bun install
bun run dev

# 安装依赖环境
uv sync

# 后端（先停止 Docker 服务：docker compose stop backend）
cd backend
source .venv/bin/activate
fastapi dev app/main.py       # ✅ 开发模式，热重载
# 或
uvicorn app.main:app --reload  # ✅ 等价方式
# 或
fastapi run app/main.py        # ✅ 生产模式，多 worker
```

### 后端测试

```bash
# 运行全部后端测试（构建全新 Docker 栈，运行，然后清理）
bash ./scripts/test.sh

# 在已运行的栈中执行测试，可传递额外的 pytest 参数
docker compose exec backend bash scripts/tests-start.sh -x

# 运行单个测试文件
docker compose exec backend bash scripts/tests-start.sh backend/tests/api/routes/test_users.py
```

### 前端测试（Playwright E2E）

```bash
docker compose up -d --wait backend
bunx playwright test
bunx playwright test --ui          # 交互式 UI 模式
docker compose down -v             # 清理测试数据
```

### Lint 与格式化

```bash
# 手动运行所有 pre-commit 检查
uv run prek run --all-files

# 仅后端
uv run ruff check --fix
uv run ruff format
uv run mypy backend/app
uv run ty check backend/app

# 仅前端
bun run lint          # biome check --write --unsafe
```

### 代码生成

```bash
# 从后端 OpenAPI schema 重新生成前端 API 客户端
bash ./scripts/generate-client.sh
```

### 数据库迁移（Alembic）

```bash
docker compose exec backend bash
alembic revision --autogenerate -m "为 User 模型添加 last_name 列"
alembic upgrade head
```

## 架构

### 后端（`backend/app/`）

包管理器：**uv**（根目录 `pyproject.toml` 声明了 uv workspace，`backend` 为成员）。

FastAPI 应用在 `backend/app/main.py` 中创建。路由树：

```
app (main.py)
 └── api_router (api/main.py, 前缀: /api/v1)
      ├── login.router      — POST /login/access-token、POST /password-recovery、POST /reset-password
      ├── users.router      — 用户的 GET/POST/PATCH/DELETE（CRUD）、/me、/me/password
      ├── items.router      — 当前用户拥有的 items 的 CRUD
      ├── utils.router      — GET /utils/health-check、POST /test-email
      └── private.router    — POST /private/users（仅在 ENVIRONMENT=local 时启用，绕过认证）
```

关键架构模式：

- **模型**（`models.py`）：SQLModel 表（`User`、`Item`）与 Pydantic 请求/响应 schema（`UserCreate`、`UserPublic`、`ItemCreate`、`ItemPublic`、`Token`、`Message` 等）定义在同一文件中。
- **CRUD**（`crud.py`）：纯函数，以 `session: Session` 作为首个关键字参数。`authenticate()` 通过用户不存在时对 dummy hash 进行密码验证来实现时序攻击安全的密码校验。
- **依赖注入**（`api/deps.py`）：`SessionDep`（DB 会话）、`TokenDep`（OAuth2 bearer token）、`CurrentUser`（通过 JWT 解码获取当前用户）、`get_current_active_superuser`。
- **配置**（`core/config.py`）：单个 `Settings` 实例从 `../.env`（backend/ 的上一级目录）读取配置。Pydantic Settings，使用 computed fields 计算 `SQLALCHEMY_DATABASE_URI` 和 `all_cors_origins`。
- **安全**（`core/security.py`）：使用 `pwdlib`，Argon2 为首选 + bcrypt 为后备。密码哈希在登录时会自动升级（如有需要）。
- **数据库初始化**（`core/db.py`）：从 settings 创建 engine，`init_db()` 创建首个超级用户。
- **邮件**（`utils.py`）：Jinja2 模板位于 `email-templates/build/`，MJML 源文件位于 `email-templates/src/`。`send_email()` 使用 `emails` 库。

### 前端（`frontend/src/`）

包管理器：**Bun**（根目录 `package.json` 声明了 bun workspace，`frontend` 为成员）。构建工具：Vite + SWC。

```
src/
 ├── client/          — 自动生成的 OpenAPI 客户端（运行 generate-client.sh 更新）
 ├── components/
 │    ├── ui/         — shadcn/ui 基础组件（button、input、dialog、dropdown-menu、sidebar 等）
 │    ├── Admin/      — 管理面板（编辑/删除用户、编辑 items）
 │    ├── Common/     — 共享组件：DeleteAlert、ErrorPage、GenericDialog 等
 │    ├── Items/      — Item CRUD 相关组件
 │    ├── Sidebar/    — 应用侧边栏及用户菜单
 │    └── UserSettings/ — 用户资料与密码表单
 ├── routes/
 │    ├── __root.tsx  — 根路由包装器
 │    ├── _layout.tsx — 认证后布局（侧边栏 + 内容区域）
 │    ├── _layout/    — index、admin、items、settings 页面
 │    ├── login.tsx、signup.tsx、recover-password.tsx、reset-password.tsx
 │    └── routeTree.gen.ts — 自动生成的路由树（由 @tanstack/router-plugin 生成）
 ├── hooks/           — useAuth、useMobile、useCopyToClipboard、useCustomToast
 └── lib/             — cn() 工具函数（clsx + tailwind-merge）
```

核心依赖：TanStack Router（文件路由）、TanStack Query（服务端状态）、React Hook Form + Zod（表单）、Axios（HTTP，由生成的客户端使用）、shadcn/ui（基于 Radix 原语的组件库）、Tailwind CSS v4。

### Docker Compose 服务（compose.yml）

- **db**：PostgreSQL 18，带健康检查
- **prestart**：一次性容器，在后端启动前运行数据库迁移和初始数据
- **backend**：FastAPI，端口 8000，依赖 db（健康） + prestart（成功完成）
- **frontend**：Nginx 提供构建后的 React 应用，端口 80
- **adminer**：数据库 Web 管理界面，端口 8080
- Traefik 标签按子域名路由：`api.${DOMAIN}` → 后端、`dashboard.${DOMAIN}` → 前端、`adminer.${DOMAIN}` → adminer
- `compose.override.yml` 添加开发便利功能（卷挂载、热重载、Mailcatcher）

### CI/CD（GitHub Actions）

- `test-backend.yml`：运行后端 pytest 测试套件
- `test-docker-compose.yml`：全栈集成测试
- `playwright.yml`：前端 E2E 测试
- `pre-commit.yml`：Lint/格式化检查
- `deploy-staging.yml` / `deploy-production.yml`：部署至自托管 runner
- `smokeshow.yml`：覆盖率报告

## Pre-commit Hooks

使用 **prek**（pre-commit 的精神继承者）。配置位于 `.pre-commit-config.yaml`：
- ruff check + format（Python）
- mypy + ty（Python 类型检查）
- biome check（前端）
- generate-frontend-sdk（后端变更时自动重新生成客户端）
- add-release-date（更新 release-notes.md）
- zizmor（GitHub Actions 安全 lint）

安装命令：`cd backend && uv run prek install -f`
