# 使用 Cookiecutter 创建 Nautobot App

在今天的挑战中，我们将使用 [Cookiecutter](https://github.com/cookiecutter/cookiecutter) 来创建我们自己的 Nautobot 应用。

> [!INFORMATION]
> Cookiecutter 是一个命令行工具，通过从预定义模板生成项目来简化新项目的创建过程。

Network to Code 于 2024 年初发布了 Nautobot App Cookiecutter [仓库](https://github.com/nautobot/cookiecutter-nautobot-app/tree/develop)，详见博文[《介绍 Cookiecutter 项目模板以支持网络自动化的 Nautobot App 开发》](https://networktocode.com/blog/introducing-cookiecutter-nautobot-app/)，我们将在今天的挑战中使用它来创建应用。

> [!TIP]
> 有关 Cookiecutter 的更多通用使用信息，请查阅 [Cookiecutter 快速入门](https://docs.nautobot.com/projects/cookiecutter-nautobot-app/en/latest/user/quick-start/#help)及其相关文档。

让我们开始吧。

## 环境搭建

请注意，在接下来的几天里，我们**不需要**启动 `nautobot-docker-compose` 下的 Nautobot 及相关 Docker 容器。应用开发环境包含一个与我们之前使用的类似的完整开发环境。

## Cookiecutter 示例

我们将按照 [Nautobot Cookiecutter 仓库 README](https://github.com/nautobot/cookiecutter-nautobot-app/blob/develop/README.md) 中的步骤来安装并创建应用：

```
@ericchou1 ➜ ~ $ pip install cookiecutter
...
Successfully installed Jinja2-3.1.6 MarkupSafe-3.0.2 arrow-1.3.0 binaryornot-0.4.4 chardet-5.2.0 cookiecutter-2.6.0 python-dateutil-2.9.0.post0 python-slugify-8.0.4 pyyaml-6.0.2 six-1.17.0 text-unidecode-1.3 types-python-dateutil-2.9.0.20241206
```

我们将为应用创建一个 `outputs` 目录：

```
@ericchou1 ➜ ~ $ mkdir outputs
```

正如 Cookiecutter 的说法，让我们来"烤一块饼干"：

> [!IMPORTANT]
> 在系统提示时，请使用你自己的信息，例如 GitHub 用户名、全名等。

```
@ericchou1 ➜ ~ $ cookiecutter \
  --output-dir=./outputs \
  --directory=nautobot-app \
  https://github.com/nautobot/cookiecutter-nautobot-app

  [1/18] codeowner_github_usernames (): <你的 GitHub 用户名> 
  [2/18] full_name (Network to Code, LLC): <你的全名>
  [3/18] email (info@networktocode.com): <你的邮箱地址>
  [4/18] github_org (nautobot): 
  [5/18] app_name (my_app): my_awesome_app
  [6/18] verbose_name (My Awesome App): My Awesome Application
  [7/18] app_slug (my-awesome-app): 
  [8/18] project_slug (nautobot-app-my-awesome-app): 
  [9/18] repo_url (https://github.com/nautobot/nautobot-app-my-awesome-app): 
  [10/18] base_url (my-awesome-app): 
  [11/18] min_nautobot_version (2.0.0): 
  [12/18] max_nautobot_version (2.9999): 
  [13/18] camel_name (MyAwesomeApp): 
  [14/18] project_short_description (My Awesome Application): 
  [15/18] Camel case name of the model class to be created, enter None if no model is needed (MyAwesomeAppExampleModel): 
  [16/18] Select open_source_license
    1 - Apache-2.0
    2 - Not open source
    Choose from [1/2] (1): 1
  [17/18] docs_base_url (https://docs.nautobot.com): 
  [18/18] docs_app_url (https://docs.nautobot.com/projects/my-awesome-app/en/latest): 

Congratulations! Your cookie has now been baked. It is located at /home/vscode/outputs/nautobot-app-my-awesome-app

⚠️⚠️ Before you start using your cookie you must run the following commands inside your cookie:

* poetry lock
* cp development/creds.example.env development/creds.env
* poetry install
* poetry shell
* invoke makemigrations
* invoke ruff --fix # this will ensure all python files are formatted correctly, may require `sudo chown -R $USER ./` as migrations may be owned by root

The file `creds.env` will be ignored by git and can be used to override default environment variables.
```

请注意 Cookie 烘焙完成后列出的启动开发环境的步骤。

以下是 `outputs` 目录的内容：

```
@ericchou1 ➜ ~ $ ls outputs/
nautobot-app-my-awesome-app
@ericchou1 ➜ ~ $ ls -lia outputs/
total 16
1346745 drwxr-xr-x 3 vscode vscode 4096 Feb 26 18:34 .
1206645 drwxr-x--- 1 vscode vscode 4096 Feb 26 18:34 ..
1347229 drwxr-xr-x 7 vscode vscode 4096 Feb 26 18:34 nautobot-app-my-awesome-app

@ericchou1 ➜ ~ $ cd outputs/nautobot-app-my-awesome-app/
@ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $ ls -l
total 80
drwxr-xr-x 2 vscode vscode  4096 Feb 26 18:34 changes
drwxr-xr-x 2 vscode vscode  4096 Feb 26 18:34 development
drwxr-xr-x 7 vscode vscode  4096 Feb 26 18:34 docs
-rw-r--r-- 1 vscode vscode   405 Feb 26 18:34 invoke.example.yml
-rw-r--r-- 1 vscode vscode   288 Feb 26 18:34 invoke.mysql.yml
-rw-r--r-- 1 vscode vscode   580 Feb 26 18:34 LICENSE
-rw-r--r-- 1 vscode vscode  4092 Feb 26 18:34 mkdocs.yml
drwxr-xr-x 6 vscode vscode  4096 Feb 26 18:34 my_awesome_app
-rw-r--r-- 1 vscode vscode  4870 Feb 26 18:34 pyproject.toml
-rw-r--r-- 1 vscode vscode  5108 Feb 26 18:34 README.md
-rw-r--r-- 1 vscode vscode 30961 Feb 26 18:34 tasks.py
```

切换到该目录：

```
@ericchou1 ➜ ~ $ cd outputs/nautobot-app-my-awesome-app/
@ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $
```

下一步，我们将启动应用的开发环境。

## 开发环境

该目录包含一个开发用的 Docker 环境：

```
@ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $ ls -l development/
total 48
-rw-r--r-- 1 vscode vscode 2678 Feb 26 18:34 app_config_schema.py
-rw-r--r-- 1 vscode vscode  992 Feb 26 18:34 creds.example.env
-rw-r--r-- 1 vscode vscode 1257 Feb 26 18:34 development.env
-rw-r--r-- 1 vscode vscode  150 Feb 26 18:34 development_mysql.env
-rw-r--r-- 1 vscode vscode 1430 Feb 26 18:34 docker-compose.base.yml
-rw-r--r-- 1 vscode vscode 2150 Feb 26 18:34 docker-compose.dev.yml
-rw-r--r-- 1 vscode vscode  934 Feb 26 18:34 docker-compose.mysql.yml
-rw-r--r-- 1 vscode vscode  500 Feb 26 18:34 docker-compose.postgres.yml
-rw-r--r-- 1 vscode vscode  295 Feb 26 18:34 docker-compose.redis.yml
-rw-r--r-- 1 vscode vscode 3123 Feb 26 18:34 Dockerfile
-rw-r--r-- 1 vscode vscode 4044 Feb 26 18:34 nautobot_config.py
-rw-r--r-- 1 vscode vscode 1303 Feb 26 18:34 towncrier_template.j2
```

我们将使用 `poetry`、`invoke` 及其他工具：

```
@ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $ poetry lock
Creating virtualenv my-awesome-app-TNUNvfeN-py3.10 in /home/vscode/.cache/pypoetry/virtualenvs
Updating dependencies
Resolving dependencies... (26.7s)

Writing lock file

@ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $ cp development/creds.example.env development/creds.env

@ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $ poetry install

@ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $ poetry shell

@ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $ invoke build
Building Nautobot with Python 3.11...
Running docker compose command "build"
#0 building with "default" instance using docker driver

#1 [nautobot internal] load build definition from Dockerfile
#1 transferring dockerfile: 3.16kB done
#1 DONE 0.0s
...

#13 naming to docker.io/my-awesome-app/nautobot:2.3.1-py3.11 done
#13 DONE 27.7s

#14 [nautobot] resolving provenance for metadata file
#14 DONE 0.0s

@ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $ invoke makemigrations
Running docker compose command "ps --services --filter status=running"
Running docker compose command "run --rm --entrypoint='nautobot-server makemigrations my_awesome_app' nautobot"
[+] Creating 4/4
 ✔ Network my-awesome-app_default         Created                                                                                                 0.1s 
 ✔ Volume "my-awesome-app_postgres_data"  Created                                      
 ...
```

最后，启动容器：

> [!TIP]
> 初次构建需要一些时间，请等到看到 `nautobot-1  | Starting development server at http://0.0.0.0:8080/` 消息后再进行下一步。

```
@ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $ invoke debug
Starting  in debug mode...
Running docker compose command "up"
 Network my-awesome-app_default  Creating
 Network my-awesome-app_default  Created
 Volume "my-awesome-app_postgres_data"  Creating
 Volume "my-awesome-app_postgres_data"  Created
 Container my-awesome-app-redis-1  Creating
 Container my-awesome-app-docs-1  Creating
 Container my-awesome-app-db-1  Creating
 Container my-awesome-app-docs-1  Created
 Container my-awesome-app-redis-1  Created
 Container my-awesome-app-db-1  Created
 Container my-awesome-app-nautobot-1  Creating
 Container my-awesome-app-nautobot-1  Created
 Container my-awesome-app-beat-1  Creating
 Container my-awesome-app-worker-1  Creating
 Container my-awesome-app-beat-1  Created
 Container my-awesome-app-worker-1  Created
Attaching to beat-1, db-1, docs-1, nautobot-1, redis-1, worker-1
redis-1     | 1:C 26 Feb 2025 18:58:23.513 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
redis-1     | 1:C 26 Feb 2025 18:58:23.513 # Redis version=6.2.17, bits=64, commit=00000000, modified=0, pid=1, just started
redis-1     | 1:C 26 Feb 2025 18:58:23.513 # Configuration loaded
redis-1     | 1:M 26 Feb 2025 18:58:23.514 * monotonic clock: POSIX clock_gettime
redis-1     | 1:M 26 Feb 2025 18:58:23.515 * Running mode=standalone, port=6379.
redis-1     | 1:M 26 Feb 2025 18:58:23.515 # Server initialized
...
```

关于 CSRF 设置还有一件事需要处理。这是由于 Codespace 的端口转发需要允许 localhost 来源。请勿在生产环境中执行此操作，这里仅用于实验：

```
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ docker exec -it -u root my-awesome-app-nautobot-1 bash

root@b02cb102c0b1:/source# ls
LICENSE  README.md  changes  development  dist  docs  invoke.example.yml  invoke.mysql.yml  mkdocs.yml  my_awesome_app  poetry.lock  pyproject.toml  tasks.py

root@b02cb102c0b1:/source# cd /opt/nautobot/

root@b02cb102c0b1:/opt/nautobot# apt update
root@b02cb102c0b1:/opt/nautobot# apt install -y vim

root@b02cb102c0b1:/opt/nautobot# vim nautobot_config.py 

CSRF_TRUSTED_ORIGINS = ["http://localhost:8080", "https://localhost:8080"] 
```

![csrf_trusted_origin](images/CSRF_Trusted_Origin.png)

现在我们可以在 `8080` 端口打开浏览器，使用 `admin/admin` 登录。注意，应用还附带了一个 [Django Debug Toolbar](https://django-debug-toolbar.readthedocs.io/en/latest/)，可以用 `Hide >>` 按钮将其隐藏：

![initial_home_page](images/initial_home_page.png)

导航到 `APPS -> Installed Apps` 部分，就能看到我们崭新的应用：

![installed_apps](images/installed_apps.png)

恭喜，这是一种快速启动 Nautobot 应用开发环境的绝妙方式，更不用说我们还学到了 Python 社区中的另一个实用工具！

在接下来的 3 天（到第 45 天），我们将继续使用这个 Cookiecutter 应用。除非你的免费 Codespace 存储配额即将用尽，否则建议停止 Codespace 实例，在接下来几天再回来继续使用。

## 第 42 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

请在你选择的社交媒体上发布今天挑战中构建的新应用截图，务必使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，这样我们可以分享你的进展！

在明天的挑战中，我们将更仔细地研究应用的文件结构。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+42+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 42 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
