# FastAPI 项目 - 部署

你可以使用 Docker Compose 将项目部署到远程服务器。

本项目需要你配置一个 Traefik 代理来处理与外部世界的通信和 HTTPS 证书。

你可以使用 CI/CD（持续集成和持续部署）系统来自动部署，项目中已有适用于 GitHub Actions 的配置。

但你还需要先完成一些配置。🤓

## 准备工作

* 准备一台可用的远程服务器。
* 配置你的域名 DNS 记录，指向你刚创建的服务器的 IP 地址。
* 为你的域名配置通配符子域名，这样你就可以为不同服务使用多个子域名，例如 `*.fastapi-project.example.com`。这对于访问不同组件非常有用，比如 `dashboard.fastapi-project.example.com`、`api.fastapi-project.example.com`、`traefik.fastapi-project.example.com`、`adminer.fastapi-project.example.com` 等。同样适用于 `staging` 环境，如 `dashboard.staging.fastapi-project.example.com`、`adminer.staging.fastapi-project.example.com` 等。
* 在远程服务器上安装并配置 [Docker](https://docs.docker.com/engine/install/)（Docker Engine，而非 Docker Desktop）。

## 公共 Traefik

我们需要一个 Traefik 代理来处理传入的连接和 HTTPS 证书。

以下步骤只需执行一次。

### Traefik Docker Compose

* 在远程服务器上创建一个目录来存放你的 Traefik Docker Compose 文件：

```bash
mkdir -p /root/code/traefik-public/
```

将 Traefik Docker Compose 文件复制到你的服务器。你可以在本地终端运行 `rsync` 命令来完成：

```bash
rsync -a compose.traefik.yml root@your-server.example.com:/root/code/traefik-public/
```

### Traefik 公共网络

这个 Traefik 需要有一个名为 `traefik-public` 的 Docker "公共网络"来与你的技术栈通信。

这样，将有一个单一的公共 Traefik 代理处理与外部世界的通信（HTTP 和 HTTPS），而在其背后，你可以有一个或多个使用不同域名的技术栈，即使它们都部署在同一台服务器上。

在远程服务器上运行以下命令，创建一个名为 `traefik-public` 的 Docker "公共网络"：

```bash
docker network create traefik-public
```

### Traefik 环境变量

Traefik Docker Compose 文件需要在启动之前在终端中设置一些环境变量。你可以在远程服务器上运行以下命令来完成。

* 创建 HTTP 基本认证的用户名，例如：

```bash
export USERNAME=admin
```

* 创建 HTTP 基本认证密码的环境变量，例如：

```bash
export PASSWORD=changethis
```

* 使用 openssl 生成 HTTP 基本认证的"哈希"版本密码，并将其存储到环境变量中：

```bash
export HASHED_PASSWORD=$(openssl passwd -apr1 $PASSWORD)
```

要验证哈希后的密码是否正确，可以打印出来查看：

```bash
echo $HASHED_PASSWORD
```

* 创建服务器域名的环境变量，例如：

```bash
export DOMAIN=fastapi-project.example.com
```

* 创建 Let's Encrypt 邮箱的环境变量，例如：

```bash
export EMAIL=admin@example.com
```

**注意**：你需要设置一个不同的邮箱，`@example.com` 的邮箱无法使用。

### 启动 Traefik Docker Compose

进入远程服务器中存放 Traefik Docker Compose 文件的目录：

```bash
cd /root/code/traefik-public/
```

现在环境变量已设置好，`compose.traefik.yml` 也已就位，你可以运行以下命令启动 Traefik Docker Compose：

```bash
docker compose -f compose.traefik.yml up -d
```

## 部署 FastAPI 项目

Traefik 配置好后，你就可以使用 Docker Compose 部署你的 FastAPI 项目了。

**注意**：你可能想先跳到关于使用 GitHub Actions 进行持续部署的章节。

## 复制代码

```bash
rsync -av --filter=":- .gitignore" ./ root@your-server.example.com:/root/code/app/
```

注意：`--filter=":- .gitignore"` 告诉 `rsync` 使用与 git 相同的规则，忽略被 git 忽略的文件，比如 Python 虚拟环境。

## 环境变量

你需要先设置一些环境变量。

### 生成密钥

`.env` 文件中的一些环境变量默认值为 `changethis`。

你必须将其替换为秘密密钥，可以运行以下命令来生成密钥：

```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

复制生成的内容，将其用作密码/秘密密钥。然后再次运行该命令生成另一个安全密钥。

### 必需的环境变量

设置 `ENVIRONMENT`，默认值为 `local`（用于开发），但部署到服务器时应设置为类似 `staging` 或 `production`：

```bash
export ENVIRONMENT=production
```

设置 `DOMAIN`，默认值为 `localhost`（用于开发），但部署时应使用你自己的域名，例如：

```bash
export DOMAIN=fastapi-project.example.com
```

将 `POSTGRES_PASSWORD` 设置为不同于 `changethis` 的值：

```bash
export POSTGRES_PASSWORD="changethis"
```

设置 `SECRET_KEY`，用于签名令牌：

```bash
export SECRET_KEY="changethis"
```

注意：你可以使用上面的 Python 命令来生成一个安全的秘密密钥。

将 `FIRST_SUPER_USER_PASSWORD` 设置为不同于 `changethis` 的值：

```bash
export FIRST_SUPERUSER_PASSWORD="changethis"
```

设置 `BACKEND_CORS_ORIGINS` 以包含你的域名：

```bash
export BACKEND_CORS_ORIGINS="https://dashboard.${DOMAIN?Variable not set},https://api.${DOMAIN?Variable not set}"
```

你还可以设置其他几个环境变量：

* `PROJECT_NAME`：项目名称，用于 API 文档和邮件中。
* `STACK_NAME`：用于 Docker Compose 标签和项目名称的技术栈名称，对于 `staging`、`production` 等环境应有所不同。你可以使用域名并将点号替换为短横线，例如 `fastapi-project-example-com` 和 `staging-fastapi-project-example-com`。
* `BACKEND_CORS_ORIGINS`：逗号分隔的允许的 CORS 来源列表。
* `FIRST_SUPERUSER`：第一个超级用户的邮箱，该超级用户将有权创建新用户。
* `SMTP_HOST`：用于发送邮件的 SMTP 服务器主机地址，由你的邮件服务提供商提供（例如 Mailgun、Sparkpost、Sendgrid 等）。
* `SMTP_USER`：用于发送邮件的 SMTP 服务器用户名。
* `SMTP_PASSWORD`：用于发送邮件的 SMTP 服务器密码。
* `EMAILS_FROM_EMAIL`：发送邮件所使用的邮箱账户。
* `POSTGRES_SERVER`：PostgreSQL 服务器的主机名。你可以保留默认值 `db`，由同一个 Docker Compose 提供。通常不需要更改，除非你使用的是第三方服务提供商。
* `POSTGRES_PORT`：PostgreSQL 服务器的端口。你可以保留默认值。通常不需要更改，除非你使用的是第三方服务提供商。
* `POSTGRES_USER`：Postgres 用户，你可以保留默认值。
* `POSTGRES_DB`：此应用程序使用的数据库名称。你可以保留默认值 `app`。
* `SENTRY_DSN`：Sentry 的 DSN（如果你在使用的话）。

## GitHub Actions 环境变量

还有一些仅由 GitHub Actions 使用的环境变量，你可以配置：

* `LATEST_CHANGES`：由 GitHub Action [latest-changes](https://github.com/tiangolo/latest-changes) 使用，根据合并的 PR 自动添加发布说明。它是一个个人访问令牌，详情请阅读相关文档。
* `SMOKESHOW_AUTH_KEY`：用于使用 [Smokeshow](https://github.com/samuelcolvin/smokeshow) 处理和发布代码覆盖率，按照其说明创建一个（免费的）Smokeshow 密钥。

### 使用 Docker Compose 部署

环境变量就位后，你可以使用 Docker Compose 进行部署：

```bash
cd /root/code/app/
docker compose -f compose.yml build
docker compose -f compose.yml up -d
```

对于生产环境，你通常不需要 `compose.override.yml` 中的覆盖配置，因此我们显式指定 `compose.yml` 作为要使用的文件。

## 持续部署（CD）

你可以使用 GitHub Actions 自动部署项目。😎

你可以配置多个环境的部署。

项目中已经配置了两个环境，`staging` 和 `production`。🚀

### 安装 GitHub Actions Runner

* 在远程服务器上，为 GitHub Actions 创建一个用户：

```bash
sudo adduser github
```

* 为 `github` 用户添加 Docker 权限：

```bash
sudo usermod -aG docker github
```

* 临时切换到 `github` 用户：

```bash
sudo su - github
```

* 进入 `github` 用户的 home 目录：

```bash
cd
```

* [按照官方指南安装 GitHub Action 自托管 runner](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/adding-self-hosted-runners#adding-a-self-hosted-runner-to-a-repository)。

* 当询问标签时，为环境添加一个标签，例如 `production`。你也可以稍后再添加标签。

安装完成后，指南会告诉你要运行一个命令来启动 runner。然而，一旦你终止该进程或与服务器的本地连接断开，它就会停止。

为了确保它在启动时运行并持续运行，你可以将其安装为一个服务。为此，退出 `github` 用户并回到 `root` 用户：

```bash
exit
```

执行后，你将回到之前的用户，并且所在的目录也属于该用户。

在能够进入 `github` 用户目录之前，你需要先成为 `root` 用户（你可能已经是了）：

```bash
sudo su
```

* 作为 `root` 用户，进入 `github` 用户 home 目录下的 `actions-runner` 目录：

```bash
cd /home/github/actions-runner
```

* 使用 `github` 用户将自托管 runner 安装为服务：

```bash
./svc.sh install github
```

* 启动服务：

```bash
./svc.sh start
```

* 检查服务状态：

```bash
./svc.sh status
```

你可以在官方指南中了解更多：[将自托管 runner 应用程序配置为服务](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/configuring-the-self-hosted-runner-application-as-a-service)。

### 配置 GitHub 环境

部署工作流为 `staging` 和 `production` 使用了 [GitHub 环境](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments)。这使得你可以配置环境特定的密钥、部署保护规则（例如必需的审核者、等待计时器）以及部署状态跟踪。

要配置它们，前往你仓库的 **Settings** > **Environments**，创建 `staging` 和 `production` 环境。

### 设置密钥

对于每个 GitHub 环境（`staging` 和 `production`），将所需的密钥配置为[环境密钥](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets#creating-secrets-for-an-environment)。环境密钥优于[仓库密钥](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets#creating-secrets-for-a-repository)，因为它们的作用范围限定在特定环境内，减少了暴露风险，并符合你配置的任何保护规则。

当前的 Github Actions 工作流需要以下密钥：

* `DOMAIN_PRODUCTION`
* `DOMAIN_STAGING`
* `STACK_NAME_PRODUCTION`
* `STACK_NAME_STAGING`
* `EMAILS_FROM_EMAIL`
* `FIRST_SUPERUSER`
* `FIRST_SUPERUSER_PASSWORD`
* `POSTGRES_PASSWORD`
* `SECRET_KEY`
* `LATEST_CHANGES`
* `SMOKESHOW_AUTH_KEY`

## GitHub Action 部署工作流

`.github/workflows` 目录中已有配置好的 GitHub Action 工作流，用于部署到对应环境（带有对应标签的 GitHub Actions runner）：

* `staging`：推送（或合并）到 `master` 分支后触发。
* `production`：发布 release 后触发。

两个工作流都与各自对应的 GitHub 环境关联，因此部署将在仓库的 **Environments** 部分可见，并会遵循你配置的任何保护规则。

如果你需要添加额外的环境，可以以这些工作流为起点。

## URL

将 `fastapi-project.example.com` 替换为你自己的域名。

### 主 Traefik 仪表盘

Traefik UI：`https://traefik.fastapi-project.example.com`

### 生产环境

前端：`https://dashboard.fastapi-project.example.com`

后端 API 文档：`https://api.fastapi-project.example.com/docs`

后端 API 基础 URL：`https://api.fastapi-project.example.com`

Adminer：`https://adminer.fastapi-project.example.com`

### 预发布环境

前端：`https://dashboard.staging.fastapi-project.example.com`

后端 API 文档：`https://api.staging.fastapi-project.example.com/docs`

后端 API 基础 URL：`https://api.staging.fastapi-project.example.com`

Adminer：`https://adminer.staging.fastapi-project.example.com`
