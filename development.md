# FastAPI 项目 - 开发

## Docker Compose

* 使用 Docker Compose 启动本地技术栈：

```bash
docker compose watch
```

* 现在你可以打开浏览器并访问以下地址：

前端，使用 Docker 构建，根据路径处理路由：<http://localhost:5173>

后端，基于 OpenAPI 的 JSON Web API：<http://localhost:8000>

基于 Swagger UI 的自动交互式文档（来自 OpenAPI 后端）：<http://localhost:8000/docs>

Adminer，数据库 Web 管理界面：<http://localhost:8080>

Traefik UI，查看代理如何处理路由：<http://localhost:8090>

**注意**：首次启动技术栈时，可能需要等待一分钟左右才能就绪。此时后端正在等待数据库准备就绪并完成各项配置。你可以查看日志来监控进度。

查看日志，运行（在另一个终端中）：

```bash
docker compose logs
```

要查看特定服务的日志，加上服务名称，例如：

```bash
docker compose logs backend
```

## Mailcatcher

Mailcatcher 是一个简单的 SMTP 服务器，用于捕获本地开发期间后端发送的所有电子邮件。它不会发送真实的邮件，而是将其捕获并在 Web 界面中展示。

这非常适用于：

* 在开发期间测试邮件功能
* 验证邮件内容和格式
* 在不发送真实邮件的情况下调试邮件相关功能

使用 Docker Compose 本地运行时，后端已自动配置为使用 Mailcatcher（SMTP 端口为 1025）。所有捕获的邮件都可以在 <http://localhost:1080> 查看。

## 本地开发

Docker Compose 文件的配置使得每个服务都可以在 `localhost` 的不同端口上访问。

后端和前端使用的端口与其本地开发服务器的端口相同，后端的地址是 `http://localhost:8000`，前端的地址是 `http://localhost:5173`。

这样，你可以关闭某个 Docker Compose 服务并启动其本地开发服务器，一切仍然可以正常工作，因为它们使用的端口是相同的。

例如，你可以在 Docker Compose 中停止 `frontend` 服务，在另一个终端中运行：

```bash
docker compose stop frontend
```

然后启动本地前端开发服务器：

```bash
bun run dev
```

或者你可以停止 `backend` Docker Compose 服务：

```bash
docker compose stop backend
```

然后运行后端的本地开发服务器：

```bash
cd backend
fastapi dev app/main.py
```

## 使用 `localhost.tiangolo.com` 的 Docker Compose

当你启动 Docker Compose 技术栈时，默认使用 `localhost`，每个服务使用不同的端口（后端、前端、adminer 等）。

当你部署到生产环境（或预发布环境）时，每个服务会部署在不同的子域名下，例如后端使用 `api.example.com`，前端使用 `dashboard.example.com`。

在关于[部署](deployment.md)的指南中，你可以了解 Traefik，即已配置的代理。它是负责根据子域名将流量转发到各个服务的组件。

如果你想在本地测试一切是否正常工作，可以编辑本地的 `.env` 文件，将其修改为：

```dotenv
DOMAIN=localhost.tiangolo.com
```

Docker Compose 文件将使用它来配置服务的基础域名。

Traefik 将使用它把 `api.localhost.tiangolo.com` 的流量转发到后端，把 `dashboard.localhost.tiangolo.com` 的流量转发到前端。

域名 `localhost.tiangolo.com` 是一个特殊域名，它（及其所有子域名）被配置为指向 `127.0.0.1`。这样你就可以在本地开发中使用它。

修改完成后，重新运行：

```bash
docker compose watch
```

在部署时，例如在生产环境中，主要的 Traefik 是在 Docker Compose 文件之外配置的。对于本地开发，`compose.override.yml` 中包含了一个 Traefik，正是为了让你测试域名是否按预期工作，例如 `api.localhost.tiangolo.com` 和 `dashboard.localhost.tiangolo.com`。

## Docker Compose 文件和环境变量

有一个主 `compose.yml` 文件，包含适用于整个技术栈的所有配置，`docker compose` 会自动使用它。

此外还有一个 `compose.override.yml`，包含用于开发的覆盖配置，例如将源代码作为卷挂载。`docker compose` 会自动使用它来在 `compose.yml` 之上应用覆盖。

这些 Docker Compose 文件使用 `.env` 文件中的配置，将其作为环境变量注入到容器中。

它们还使用在调用 `docker compose` 命令之前在脚本中设置的环境变量中获取的一些额外配置。

修改变量后，请确保重启技术栈：

```bash
docker compose watch
```

## .env 文件

`.env` 文件包含了所有配置、生成的密钥和密码等。

根据你的工作流程，你可能希望将其从 Git 中排除，例如你的项目是公开的。在这种情况下，你需要确保为 CI 工具设置一种方式，以便在构建或部署项目时获取它。

一种方法是可以在 CI/CD 系统中添加每个环境变量，并更新 `compose.yml` 文件以读取特定的环境变量，而不是读取 `.env` 文件。

## Pre-commit 和代码检查

我们使用一个名为 [prek](https://prek.j178.dev/) 的工具（[Pre-commit](https://pre-commit.com/) 的现代替代品）进行代码检查和格式化。

安装后，它会在 git 提交之前自动运行。这样可以确保代码在提交之前就已经格式化和检查完毕。

你可以在项目根目录下找到一个包含配置的 `.pre-commit-config.yaml` 文件。

#### 安装 prek 使其自动运行

`prek` 已经是项目依赖的一部分。

在安装并可用 `prek` 工具之后，你需要在本地仓库中"安装"它，使其在每次提交之前自动运行。

使用 `uv`，你可以这样做（确保你在 `backend` 文件夹内）：

```bash
❯ uv run prek install -f
prek installed at `../.git/hooks/pre-commit`
```

`-f` 标志强制安装，以防之前已经安装过一个 `pre-commit` 钩子。

现在每当你尝试提交时，例如：

```bash
git commit
```

...prek 将运行并检查、格式化你即将提交的代码，并要求你在提交之前重新用 git 暂存（stage）那些代码。

然后你可以再次 `git add` 修改/修复的文件，之后就可以提交了。

#### 手动运行 prek 钩子

你也可以在所有文件上手动运行 `prek`，使用 `uv` 执行：

```bash
❯ uv run prek run --all-files
check for added large files..............................................Passed
check toml...............................................................Passed
check yaml...............................................................Passed
fix end of files.........................................................Passed
trim trailing whitespace.................................................Passed
ruff.....................................................................Passed
ruff-format..............................................................Passed
biome check..............................................................Passed
```

## URL

生产或预发布环境的 URL 使用相同的路径，但使用的是你自己的域名。

### 开发环境 URL

本地开发的开发环境 URL。

前端：<http://localhost:5173>

后端：<http://localhost:8000>

自动交互式文档（Swagger UI）：<http://localhost:8000/docs>

自动替代文档（ReDoc）：<http://localhost:8000/redoc>

Adminer：<http://localhost:8080>

Traefik UI：<http://localhost:8090>

MailCatcher：<http://localhost:1080>

### 配置了 `localhost.tiangolo.com` 的开发环境 URL

本地开发的开发环境 URL。

前端：<http://dashboard.localhost.tiangolo.com>

后端：<http://api.localhost.tiangolo.com>

自动交互式文档（Swagger UI）：<http://api.localhost.tiangolo.com/docs>

自动替代文档（ReDoc）：<http://api.localhost.tiangolo.com/redoc>

Adminer：<http://localhost.tiangolo.com:8080>

Traefik UI：<http://localhost.tiangolo.com:8090>

MailCatcher：<http://localhost.tiangolo.com:1080>
