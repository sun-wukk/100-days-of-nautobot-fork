# 设置开发环境

在今天的挑战中，我们将为接下来的日子设置我们的开发环境。

有几个组件：

- [GitHub Codespace](https://github.com/features/codespaces) 是由 GitHub 提供的一个简单的、免费层级可用的开发环境。GitHub Codespace [free tier](https://docs.github.com/en/billing/managing-billing-for-your-products/managing-billing-for-github-codespaces/about-billing-for-github-codespaces#monthly-included-storage-and-core-hours-for-personal-accounts)每月包括 120 小时的核心小时数，相当于每月 60 小时的最小 2 核环境。

- 默认情况下，当你启动 GitHub Codespace 时，开发环境的界面是 [VisualStudio Code](https://code.visualstudio.com/), 这是一个完整的集成开发环境（IDE）。作为一个 IDE，VSCode 提供了许多功能和扩展，我们在完成挑战时将使用这些功能和扩展。
  
- 我们已经预先构建了一个镜像，其中包含此存储库和启动完全功能的 Nautobot 实例所需的所有组件，可在 Codespace 中使用。


启动我们的实验室有两个步骤：

1. 启动您的个人 GitHub Codespace。
2. 在 Codespace 中启动 Nautobot 和必要的组件。

让我们从使用正确的选项启动 Codespace 开始。

> [!TIP]
> 我们在学习过程中学到了一些技巧和窍门，我们保留了一份运行中的 [Lab Notes](https://github.com/nautobot/100-days-of-nautobot/blob/main/Lab_Setup/lab_related_notes/README.md)。如果您遇到问题，请随时参考这份笔记。

## 启动 Codespace

如果你是 GitHub Codespace 的新手，我们建议观看以下视频来了解启动 Codespace 的步骤。但是，如果你已经对 Codespace 有所了解，可以跳过视频，直接查看截图说明以快速参考。

[Video: Setting Up Your 100 Days of Nautobot Development Environment](https://www.youtube.com/watch?v=i1K_zBz0Ny0)

按照以下步骤启动 Codespace:

1. 点击绿色的 Code 按钮
2. 选择 Codespaces
3. 点击"..."选项
4. 选择"New with Options"

![Codespace_Screenshot_1.png](images/Codespace_Screenshot_1.png)

在下一个界面上，点击"Dev container configuration"的下拉菜单，选择"Lab Scenario 1"，然后点击"Create Codespace"：

> [!提示]
> 当你第一次启动 Codespace 时，它可能会提示你在基于浏览器的 Visual Studio Code 或启动桌面版本之间进行选择，请选择基于浏览器的版本以与屏幕截图保持一致，但如果你愿意，也可以随意选择桌面版本。

![Codespace_Screenshot_2.png](images/Codespace_Screenshot_2.png)

Codespace 将开始启动，您可以随时点击"Building Codespace"来查看创建日志并监控进度：

![Codespace_Screenshot_3.png](images/Codespace_Screenshot_3.png)

Codespace 设置完成后，您将拥有一个基于浏览器的开发环境，包含以下部分：

1. 资源管理器窗口：这是你可以选择不同文件的地方，即 Day001、Day002 等文件夹。
2. 终端窗口：还会有一个终端窗口，我们可以通过它与 Codespace 代码进行交互。
3. 在资源管理器窗口中，展开 "100-Days-of-nautobot-challenge" 文件夹和 "Day001_Development_Setup" 子文件夹，右键单击 README.md 文件并选择 "Open Preview"： 

    ![Codespace_Screenshot_4.png](images/Codespace_Screenshot_4.png)

4. 我们将大量使用终端窗口，有时会同时打开多个终端窗口。```+``` 是我们可以添加更多终端窗口的地方。

![Codespace_Screenshot_5.png](images/Codespace_Screenshot_5.png)

继续启动 Codespace，启动完成后，回到  ```Day001_Development_Setup``` README.md 文件继续学习，我们会在这里等你。

## 启动 Nautobot 和必要的组件

Codespace 中包含来自 [nautobot-docker-compose](https://github.com/nautobot/nautobot-docker-compose/) 仓库的代码。我们的 Codespace 是通过 docker-in-docker 功能启动的，这允许我们在容器中运行 Nautobot 以及必要的组件。

以下指令将在终端窗口中输入。

> [!提示]
> 如果你尝试将命令复制并粘贴到 Codespace 终端窗口，它会在第一次时要求权限。请允许它。

- 切换目录到 nautobot docker-compose 代码所在的位置：

```shell
@ericchou1 ➜ ~ $ cd nautobot-docker-compose/
```

- 我们已经安装了 [poetry](https://python-poetry.org/) 虚拟环境，所以我们只需要启用该环境：

```shell
@ericchou1 ➜ ~/nautobot-docker-compose (main) $ poetry shell
Spawning shell within /home/vscode/.cache/pypoetry/virtualenvs/nautobot-docker-compose-70lkLMMl-py3.10
@ericchou1 ➜ ~/nautobot-docker-compose (main) $ . /home/vscode/.cache/pypoetry/virtualenvs/nautobot-docker-compose-70lkLMMl-py3.10/bin/activate
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $
```

- 我们将使用 [Invoke](https://www.pyinvoke.org/) 来处理面向 shell 的子进程，用于 CLI 可调用的任务。第一步是构建 docker 镜像，如果这是你第一次构建，会花费一些时间，请确保等待直到看到末尾的 "DONE" 消息以及返回给用户的终端提示符。

```shell
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke build
Building Nautobot 2.3.2 with Python 3.8...
Running docker compose command "build"
#0 building with "default" instance using docker driver

#1 [nautobot internal] load build definition from Dockerfile
#1 transferring dockerfile: 2.32kB done
#1 WARN: FromAsCasing: 'as' and 'FROM' keywords' casing do not match (line 3)
#1 WARN: FromAsCasing: 'as' and 'FROM' keywords' casing do not match (line 14)
#1 WARN: FromAsCasing: 'as' and 'FROM' keywords' casing do not match (line 54)
#1 DONE 0.0s
... 
<skip> 
...
#2 [nautobot auth] nautobot/nautobot-dev:pull token for ghcr.io
#2 DONE 0.0s
#24 [nautobot] resolving provenance for metadata file
#24 DONE 0.0s
```

- 成功的 ```invoke build``` 将输出一个 Nautobot 的 docker 镜像，可以使用 ```docker images```检查。如果没有看到任何镜像，请重新尝试 invoke 过程，或者在  [Nautobot Slack](https://networktocode.slack.com/archives/C01UJ9ZQZ3D)中请求帮助。

```shell
(nautobot-docker-compose-py3.10) @user ➜ ~/nautobot-docker-compose (main) $ docker images
REPOSITORY                         TAG         IMAGE ID       CREATED         SIZE
yourrepo/nautobot-docker-compose   local       aae299e33a72   2 minutes ago   1.11GB
(nautobot-docker-compose-py3.10) @user ➜ ~/nautobot-docker-compose (main) $ 
```

- 我们现在已准备好导入初始数据集：

```shell
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke db-import
Importing Database into Development...

Starting Postgres for DB import...

Running docker compose command "up -d db"
 db Pulling 
 43c4264eed91 Pulling fs layer 
...
<skip>
...
 55d20525b40e Waiting 
 82052d0672a9 Downloading [====================================>              ]     720B/984B
 82052d0672a9 Downloading [==================================================>]     984B/984B
 82052d0672a9 Download complete 
 ...
 <skip>
 ...
 Network nautobot_docker_compose_default  Creating
 Network nautobot_docker_compose_default  Created
...
<skip>
...
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
```

- 一个成功的 ```invoke db-import``` 完成，将输出一个 ```postgres``` 镜像和一个运行/健康的 ```nautobot_docker_compose-db-1``` 容器。如果这两者都不可见，请尝试再次执行 invoke 过程和/或在 [Nautobot Slack](https://networktocode.slack.com/archives/C01UJ9ZQZ3D)中请求帮助。

```shell
(nautobot-docker-compose-py3.10) @user ➜ ~/nautobot-docker-compose (main) $ docker images
REPOSITORY                         TAG         IMAGE ID       CREATED         SIZE
yourrepo/nautobot-docker-compose   local       aae299e33a72   2 minutes ago   1.11GB
postgres                           13-alpine   844163899fc2   7 weeks ago     268MB
(nautobot-docker-compose-py3.10) @user ➜ ~/nautobot-docker-compose (main) $ 

(nautobot-docker-compose-py3.10) @user ➜ ~/nautobot-docker-compose (main) $ docker ps
CONTAINER ID   IMAGE                COMMAND                  CREATED          STATUS                    PORTS      NAMES
bde9c3850113   postgres:13-alpine   "docker-entrypoint.s…"   31 seconds ago   Up 30 seconds (healthy)   5432/tcp   nautobot_docker_compose-db-1
```

- 现在我们可以使用 ```invoke debug```启动 nautobot 容器。这将以调试模式启动 Nautobot 并在屏幕上显示所有消息：

> [!提示]
> 等待直到您看到 ```Starting development server at http://0.0.0.0:8080/``` 的消息后再继续下一步：

```shell
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke debug
Starting Nautobot in debug mode...
Running docker compose command "up"
 redis Pulling 
 43c4264eed91 Already exists 
 54346cffc29b Pulling fs layer 
 2866ca214a5e Pulling fs layer 
 ee16541feddb Pulling fs layer 
 d14ed515876d Pulling fs layer 
 cf7b98d3ba3c Pulling fs layer 
 4f4fb700ef54 Pulling fs layer 
 c4e0a3f69d20 Pulling fs layer 
 ...
 <skip>
 ...
nautobot-1       | October 16, 2024 - 20:18:55
nautobot-1       | Django version 4.2.16, using settings 'nautobot_config'
nautobot-1       | Starting development server at http://0.0.0.0:8080/
nautobot-1       | Quit the server with CONTROL-C.
nautobot-1       | 
```

一旦 Nautobot 启动，我们可以转到转发端口，将鼠标悬停在"转发地址"上，然后单击地球图标以打开单独的浏览器窗口：

![Codespace_Screenshot_6.png](images/Codespace_Screenshot_6.png)

新窗口应该会将你导向转发的端口，在那里可以访问 Nautobot UI。默认登录凭证是用户名 ```admin``` 和密码 ```admin```, 这是我们在初始数据集中包含的管理员用户：

![Codespace_Screenshot_7.png](images/Codespace_Screenshot_7.png)

> [!提示]
> 我知道这不是最安全的用户名和密码，根据浏览器的不同，你可能会收到警告提示。该端口和实例 不 对公众开放。


我们现在在 Codespace 中有一个正常运行的 Nautobot 实例。

## 停止 Nautobot 和必要的组件

- 让我们回到终端窗口，使用 ```Ctl+C``` 来终止 Nautobot 实例：

```shell
...
...
nautobot-1       | 20:25:48.342 INFO    django.server :
nautobot-1       |   "GET /static/img/favicon.ico?version=2.3.2 HTTP/1.1" 200 15086
redis-1          | 1:M 16 Oct 2024 20:33:15.250 * 100 changes in 300 seconds. Saving...
redis-1          | 1:M 16 Oct 2024 20:33:15.251 * Background saving started by pid 14
redis-1          | 14:C 16 Oct 2024 20:33:15.253 * DB saved on disk
redis-1          | 14:C 16 Oct 2024 20:33:15.254 * RDB: 0 MB of memory used by copy-on-write
redis-1          | 1:M 16 Oct 2024 20:33:15.351 * Background saving terminated with success
Gracefully stopping... (press Ctrl+C again to force)
 Container nautobot_docker_compose-celery_worker-1  Stopping
 Container nautobot_docker_compose-celery_beat-1  Stopping
 Container nautobot_docker_compose-celery_beat-1  Stopped
 Container nautobot_docker_compose-celery_worker-1  Stopped
 Container nautobot_docker_compose-nautobot-1  Stopping
 Container nautobot_docker_compose-nautobot-1  Stopped
 Container nautobot_docker_compose-db-1  Stopping
 Container nautobot_docker_compose-redis-1  Stopping
 Container nautobot_docker_compose-redis-1  Stopped
 Container nautobot_docker_compose-db-1  Stopped
canceled
```

- 你可以使用 ```docker ps``` 确认所有容器已成功终止：:

```shell
(nautobot-docker-compose-py3.10) @user ➜ ~/nautobot-docker-compose (main) $ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
(nautobot-docker-compose-py3.10) @user ➜ ~/nautobot-docker-compose (main) $ 
```

- 让我们继续停止 Codespace，因为当我们不使用它时，我们不想产生不必要的费用。我们将导航到您的 [Codespace](https://github.com/codespaces) 设置并停止 Codespace:

![Codespace_Screenshot_8.png](images/Codespace_Screenshot_8.png)

> [!提示]
> 您可以选择删除 Codespace，但是如果您这样做，您将需要重复本课程中的步骤。我的偏好是只 停止 Codespace，除非您的使用额度不足，您可以在 [此处](https://github.com/codespaces)检查。

这就是第 1 天的全部内容，恭喜您创建了一个实验环境！

## 第 1 天待办事项

继续在你选择的社交媒体上发布你在 Codespace 中新启动的 Nautobot 的截图，确保使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`， 这样我们就可以庆祝并分享你的进度，明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+1+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/) (复制并粘贴：我刚刚完成了 100 Days of Nautobot 的第 1 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot)
