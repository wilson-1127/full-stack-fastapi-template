# FastAPI 项目 - 后端

## 环境要求

* [Docker](https://www.docker.com/)。
* [uv](https://docs.astral.sh/uv/) 用于 Python 包和环境管理。

## Docker Compose

按照 [../development.md](../development.md) 中的指南，使用 Docker Compose 启动本地开发环境。

## 常规工作流程

默认情况下，依赖项使用 [uv](https://docs.astral.sh/uv/) 管理，前往那里了解并安装它。

在 `./backend/` 目录下，你可以通过以下命令安装所有依赖：

```console
$ uv sync
```

然后你可以通过以下命令激活虚拟环境：

```console
$ source .venv/bin/activate
```

确保你的编辑器使用的是正确的 Python 虚拟环境，解释器路径为 `backend/.venv/bin/python`。

在 `./backend/app/models.py` 中修改或添加用于数据和 SQL 表的 SQLModel 模型，在 `./backend/app/api/` 中添加 API 端点，在 `./backend/app/crud.py` 中添加 CRUD（创建、读取、更新、删除）工具函数。

## VS Code

项目中已有相应的配置，可以通过 VS Code 调试器运行后端，这样你就可以使用断点、暂停并探索变量等。

同时也已经配置好，你可以通过 VS Code 的 Python 测试选项卡运行测试。

## Docker Compose 覆盖

在开发期间，你可以在 `compose.override.yml` 文件中更改只会影响本地开发环境的 Docker Compose 设置。

对该文件的更改只会影响本地开发环境，不会影响生产环境。因此，你可以添加有助于开发工作流程的"临时"更改。

例如，包含后端代码的目录会在 Docker 容器中同步，将你更改的代码实时复制到容器内的目录中。这样你就可以立即测试更改，而无需重新构建 Docker 镜像。这应该仅在开发期间这样做，对于生产环境，你应该使用最新版本的后端代码构建 Docker 镜像。但在开发期间，它允许你非常快速地迭代。

还有一个命令覆盖，运行 `fastapi run --reload` 而不是默认的 `fastapi run`。它启动一个单服务器进程（与生产环境中的多进程不同），并在代码更改时重新加载进程。请记住，如果你有语法错误并保存了 Python 文件，它将会崩溃并退出，容器也会停止。之后，你可以修复错误并重新运行来重启容器：

```console
$ docker compose watch
```

还有一个被注释掉的 `command` 覆盖，你可以取消注释并将默认的命令注释掉。它使后端容器运行一个"什么都不做"的进程，但保持容器存活。这样你就可以进入正在运行的容器内部并执行命令，例如使用 Python 解释器来测试已安装的依赖项，或启动在检测到更改时自动重载的开发服务器。

要通过 `bash` 会话进入容器内部，你可以启动技术栈：

```console
$ docker compose watch
```

然后在另一个终端中，`exec` 到正在运行的容器内部：

```console
$ docker compose exec backend bash
```

你应该会看到类似如下的输出：

```console
root@7f2607af31c3:/app#
```

这表示你正在容器的 `bash` 会话中，作为 `root` 用户，位于 `/app` 目录下。该目录内有一个名为 "app" 的目录，你的代码就存放在容器的 `/app/app` 中。

在那里你可以使用 `fastapi run --reload` 命令来运行调试用的实时重载服务器。

```console
$ fastapi run --reload app/main.py
```

...它会像这样显示：

```console
root@7f2607af31c3:/app# fastapi run --reload app/main.py
```

然后按回车键。这将运行实时重载服务器，当检测到代码更改时自动重新加载。

然而，如果它没有检测到更改而是检测到语法错误，它就会因错误而停止。但由于容器仍然存活且你处于 Bash 会话中，你可以在修复错误后快速重新运行相同的命令（按"上箭头"和"回车"）来重启它。

...上述细节正是让容器保持空闲状态，然后在 Bash 会话中让它运行实时重载服务器的实用性所在。

## 后端测试

运行后端测试：

```console
$ bash ./scripts/test.sh
```

测试使用 Pytest 运行，你可以在 `./backend/tests/` 中修改和添加测试。

如果你使用 GitHub Actions，测试将会自动运行。

### 在运行中的技术栈中测试

如果你的技术栈已经启动，你只想运行测试，可以使用：

```bash
docker compose exec backend bash scripts/tests-start.sh
```

该 `/app/scripts/tests-start.sh` 脚本在确保技术栈的其余部分正在运行后调用 `pytest`。如果你需要向 `pytest` 传递额外的参数，可以将它们传递给该命令，参数会被转发。

例如，在遇到第一个错误时停止：

```bash
docker compose exec backend bash scripts/tests-start.sh -x
```

### 测试覆盖率

运行测试时，会生成 `htmlcov/index.html` 文件，你可以在浏览器中打开它来查看测试覆盖率。

## 数据库迁移

由于在本地开发期间，你的应用目录作为卷挂载在容器内，你也可以在容器内使用 `alembic` 命令运行迁移，迁移代码将保留在你的应用目录中（而不是仅存在于容器内部）。这样你就可以将其添加到 git 仓库中。

确保你每次更改模型时都创建该模型的"版本（revision）"，并使用该版本"升级（upgrade）"你的数据库。因为这将更新数据库中的表。否则，你的应用程序将会出错。

* 在后端容器中启动交互式会话：

```console
$ docker compose exec backend bash
```

* Alembic 已经配置好，可以从 `./backend/app/models.py` 导入你的 SQLModel 模型。

* 更改模型后（例如添加一列），在容器内创建一个数据库迁移版本，例如：

```console
$ alembic revision --autogenerate -m "为 User 模型添加 last_name 列"
```

* 将在 alembic 目录中生成的文件提交到 git 仓库。

* 创建迁移版本后，在数据库中运行迁移（这才是真正更改数据库的操作）：

```console
$ alembic upgrade head
```

如果你完全不想使用迁移功能，可以取消 `./backend/app/core/db.py` 文件中以下行的注释：

```python
SQLModel.metadata.create_all(engine)
```

并注释掉 `scripts/prestart.sh` 文件中包含以下内容的行：

```console
$ alembic upgrade head
```

如果你不想从默认模型开始，而是想从头删除/修改它们，而没有任何以前的迁移版本，你可以删除 `./backend/app/alembic/versions/` 下的迁移版本文件（`.py` Python 文件）。然后按照上述方式创建第一个迁移。

## 邮件模板

邮件模板位于 `./backend/app/email-templates/` 目录中。这里有两个目录：`build` 和 `src`。`src` 目录包含用于构建最终邮件模板的源文件。`build` 目录包含应用程序使用的最终邮件模板。

在继续之前，请确保你已在 VS Code 中安装了 [MJML 扩展](https://github.com/mjmlio/vscode-mjml)。

安装 MJML 扩展后，你可以在 `src` 目录中创建新的邮件模板。创建新的邮件模板并在编辑器中打开 `.mjml` 文件后，使用 `Ctrl+Shift+P` 打开命令面板，搜索 `MJML: Export to HTML`。这将把 `.mjml` 文件转换为 `.html` 文件，然后你就可以将其保存到 build 目录中。
