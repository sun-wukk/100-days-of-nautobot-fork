# Hello Jobs - 第 1 部分：创建 Jobs

欢迎来到 `#100DaysOfNautobot` 挑战的第 3 天！现在是时候为 Retail-r-Us 创建一些 jobs 了。

对于我们的第一个 job，我们将创建一个传统的"Hello World"类型的 job。它将是一个小型且功能有限的 Python 脚本，但是一个完全可用的 Nautobot job。这将帮助我们在继续进行更复杂的 jobs 之前，了解各个部分如何协同工作。

准备好了吗？让我们开始吧！

## 在 Codespaces 中启动 Nautobot 实例

> [!TIP]
> 如果您正在从第 1 天重新启动现有的 Codespace，可以跳过本部分。但请注意，有时在重新启动后 Docker 守护程序停止工作时，我们需要[重建 Codespace](https://github.com/nautobot/100-days-of-nautobot/blob/main/Lab_Setup/lab_related_notes/README.md#rebuild-codespace)。

> [!NOTE]
> 本部分包含大量来自第 1 天的重复信息，您可以快速浏览。

让我们回顾一下第 1 天开发设置的步骤。
我们需要通过"Code -> "..." -> New with Options"来启动一个 codespace 实例，以选择一个实验场景：

![codespaces_screenshot_1](images/codespaces_screenshot_1.png)

让我们选择"Lab Scenario 1"：

![codespaces_screenshot_2](images/codespaces_screenshot_2.png)

就像我们在第 1 天所做的那样，我们可以使用以下命令启动 Nautobot 实例以及所有其他必要的组件：

1. 导航到 `nautobot-docker-compose` 目录。
2. 启动 `poetry shell`。
3. 使用 `invoke build` 构建容器。
4. 使用 `invoke db-import` 导入初始数据。
5. 使用 `invoke debug` 启动 Nautobot。

以下是示例输出：

```
@ericchou1 ➜ ~ $ cd nautobot-docker-compose/

@ericchou1 ➜ ~/nautobot-docker-compose (main) $ poetry shell

(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke build

(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke db-import

(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke debug

Starting Nautobot in debug mode...
Running docker compose command "up"
 Container nautobot_docker_compose-redis-1  Created
 Container nautobot_docker_compose-db-1  Created
 Container nautobot_docker_compose-nautobot-1  Created
 Container nautobot_docker_compose-celery_beat-1  Created
 Container nautobot_docker_compose-celery_worker-1  Created
Attaching to celery_beat-1, celery_worker-1, db-1, nautobot-1, redis-1
redis-1          | 1:C 17 Oct 2024 12:06:43.191 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
redis-1          | 1:C 17 Oct 2024 12:06:43.191 # Redis version=6.2.16, bits=64, commit=00000000, modified=0, pid=1, just started
redis-1          | 1:C 17 Oct 2024 12:06:43.191 # Configuration loaded
redis-1          | 1:M 17 Oct 2024 12:06:43.192 * monotonic clock: POSIX clock_gettime
redis-1          | 1:M 17 Oct 2024 12:06:43.216 * Running mode=standalone, port=6379.
...
<skip>

```
 
让我们保持这个终端窗口打开，这样我们可以观察由不同容器生成的所有后续消息。

接下来，我们将使用转发的端口在单独的浏览器窗口中打开 Nautobot。为此，请转到 ```PORTS``` 并点击地球图标。

![port_forwarding_1](images/port_forwarding_1.png)

> [!NOTE]
> 使用 admin/admin 作为用户名和密码。

现在我们已经在调试模式下运行 Nautobot，并打开了浏览器窗口来访问 Nautobot UI。

## 创建Job文件

我们现在准备使用 Python 创建第一个Job文件。**有两个选项可供选择，请选择其中一个。** 如果你是 Nautobot Jobs 的新手，这对我们大多数人来说可能都是这样，我建议从选项 1 开始。但不用担心——也可以看一下选项 2 以了解其背后的结构。

### 选项 1. 在 Jobs 文件夹中创建文件

在 ```nautobot-docker-compose``` 文件夹下，找到 ```jobs``` 文件夹，右键单击创建新文件，并将其命名为 ```hello_jobs.py```：

![create_job_in_folder_1](images/create_job_in_folder_1.png)

我们可以双击该文件并在编辑器窗口中打开：

![create_job_in_folder_2](images/create_job_in_folder_2.png)

此选项之所以有效，是因为 ```docker-compose.local.yml``` 文件将卷映射到 ```nautobot``` 容器：

``
---
services:
  nautobot:
    command: "nautobot-server runserver 0.0.0.0:8080"
    ports:
      - "8080:8080"
    volumes:
      - "../config/nautobot_config.py:/opt/nautobot/nautobot_config.py"
      - "../jobs:/opt/nautobot/jobs"
    healthcheck:
      interval: "30s"
      timeout: "10s"
      start_period: "60s"
      retries: 3
      test: ["CMD", "true"]  # Due to layering, disable: true won't work. Instead, change the test
  celery_worker:
    volumes:
      - "../config/nautobot_config.py:/opt/nautobot/nautobot_config.py"
      - "../jobs:/opt/nautobot/jobs"
```

如果您已通过选项 1 创建了文件，可以阅读选项 2 以更好地理解作业文件结构。

### 选项 2. 在 Docker 容器中创建文件

使用选项 2，我们将直接在 ```nautobot``` 容器中创建作业文件。我们演示此选项是因为它遵循 [Jobs Developer Guide](https://docs.nautobot.com/projects/core/en/stable/development/jobs/#installing-jobs) 中用于安装作业的方法。

让我们回到终端部分。在保持第一个 ```invoke debug``` 窗口打开的情况下，点击 ```+``` 符号添加一个单独的终端窗口：

![start_new_terminal_1](images/start_new_terminal_1.png)


> [!TIP]
> 通过保持第一个终端窗口打开并处于调试模式，我们将能够看到系统级消息，这对我们的学习过程很有帮助。

让我们看看哪些 docker 容器正在运行：

```
@ericchou1 ➜ ~ $ docker ps
CONTAINER ID   IMAGE                                    COMMAND                  CREATED              STATUS                        PORTS                                                 NAMES
0674568846da   yourrepo/nautobot-docker-compose:local   "sh -c 'nautobot-ser…"   About a minute ago   Up About a minute (healthy)   8080/tcp, 8443/tcp                                    nautobot_docker_compose-celery_worker-1
50c2738fbded   yourrepo/nautobot-docker-compose:local   "sh -c 'nautobot-ser…"   About a minute ago   Up About a minute             8080/tcp, 8443/tcp                                    nautobot_docker_compose-celery_beat-1
15a80b83b587   yourrepo/nautobot-docker-compose:local   "/docker-entrypoint.…"   About a minute ago   Up About a minute (healthy)   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 8443/tcp   nautobot_docker_compose-nautobot-1
fd292402488a   redis:6-alpine                           "docker-entrypoint.s…"   About a minute ago   Up About a minute             6379/tcp                                              nautobot_docker_compose-redis-1
5075768319ae   postgres:13-alpine                       "docker-entrypoint.s…"   About a minute ago   Up About a minute (healthy)   5432/tcp                                              nautobot_docker_compose-db-1
@ericchou1 ➜ ~ $ 

```

我们可以从终端窗口使用 ```docker exec``` 附加到 Nautobot docker 镜像。

让我们以 root 身份附加到 nautobot 容器，导航到 ```/opt/nautobot/jobs``` 文件夹，然后创建一个 ```hello_jobs.py``` 文件：

```
@ericchou1 ➜ ~ $ docker exec -it -u root nautobot_docker_compose-nautobot-1 bash

root@196e7f7abedd:/opt/nautobot# cd jobs
root@196e7f7abedd:/opt/nautobot/jobs# touch hello_jobs.py
```

> [!IMPORTANT] 
文件的位置非常重要，这是 Nautobot 查找作业文件的地方。同时使用 `chown` 设置文件权限也很关键。

我们需要将文件的所有者和组更改为 `nautobot`：


```
root@196e7f7abedd:/opt/nautobot/jobs# ls -lia hello_jobs.py 
1487781 -rw-r--r-- 1 root root 0 Oct 17 12:38 hello_jobs.py

root@196e7f7abedd:/opt/nautobot/jobs# chown nautobot:nautobot hello_jobs.py 

root@196e7f7abedd:/opt/nautobot/jobs# ls -lia hello_jobs.py 

1487781 -rw-r--r-- 1 nautobot nautobot 0 Oct 17 12:38 hello_jobs.py
```

我们可以直接在终端中编辑该文件。但是，请记住 Visual Studio Code 是一个功能完整的 IDE。通过使用 Docker 扩展，我们可以进行更直观的编辑。

我们可以点击 docker 扩展符号，找到 Nautobot 容器：

![docker_access_1](images/docker_access_1.png)

一旦我们在 `/opt/nautobot/jobs` 下找到该文件，将其高亮显示并选择 `open` 选项在查看器区域中打开：

![docker_access_2](images/docker_access_2.png)

打开文件后，我们可以开始向作业添加组件。

## Hello Jobs

无论您使用选项 1 还是选项 2，您现在都可以修改 Python 文件来指定作业的详细信息。

首先，我们需要从 ```nautobot.apps.jobs``` 模块导入必要的对象和方法：

```
from nautobot.apps.jobs import Job, register_jobs
```

```Job``` 是一个对象，我们将在自己的 Job 类中继承它。编程中的继承允许我们定义一个类，该类继承另一个预先创建的对象的所有方法和属性。

我们现在可以使用 ```run()``` 方法定义自己的 Jobs 对象来存放我们的代码：

```
class HelloJobs(Job):

    def run(self):
        self.logger.debug("Hello, this is my first Nautobot Job.")
```

最后，我们需要向 Nautobot 注册我们的任务：

```
register_jobs(
    HelloJobs,
)
```

> [!IMPORTANT]
> 注册作业是一个重要步骤，许多人（包括我自己）在初次接触 Nautobot 作业时可能会忽略。

以下是完整文件的样子：

```python
from nautobot.apps.jobs import Job, register_jobs

class HelloJobs(Job):

    def run(self):
        self.logger.debug("Hello, this is my first Nautobot Job.")

register_jobs(
    HelloJobs,
)
```

确保我们在继续下一步之前保存文件。

## 注册并运行作业

为了使作业可用，我们需要通过注册来告知 Nautobot。还记得那个方便地为我们构建和启动所有组件的 ```invoke debug``` 命令吗？事实证明还有一个用于 ```post_upgrade``` 的 ```invoke``` 命令。

> [!IMPORTANT]
使用 `invoke post-upgrade` 注册任务是许多初次接触 Nautobot Jobs 的工程师可能遗漏的另一个关键步骤。请记住执行此步骤。

打开（另一个）终端窗口并启动 poetry shell：

```
@ericchou1 ➜ ~ $ cd nautobot-docker-compose/
@ericchou1 ➜ ~/nautobot-docker-compose (main) $ poetry shell
```

我们可以使用 ```invoke --list``` 来查看所有可用的 CLI 命令：

```
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke --list
Available tasks:

  build                  Build Nautobot docker image.
  cli                    Launch a bash shell inside the running Nautobot container.
  createsuperuser        Create a new Nautobot superuser account (default: "admin"), will prompt for password.
  db-export              Export the database from the dev environment to nautobot.sql.
  db-import              Install the backup of Nautobot db into development environment.
  debug                  Start Nautobot and its dependencies in debug mode.
  destroy                Destroy all containers and volumes.
  import-nautobot-data   Import nautobot_data.json.
  migrate                Perform migrate operation in Django.
  nbshell                Launch an interactive nbshell session.
  post-upgrade           Nautobot common post-upgrade operations using a single entrypoint.
  restart                Gracefully restart all containers.
  start                  Start Nautobot and its dependencies in detached mode.
  stop                   Stop Nautobot and its dependencies.

```

让我们执行一个 ```invoke post-upgrade```：

```
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke post-upgrade
Running docker compose command "ps --services --filter status=running"
Running docker compose command "exec nautobot nautobot-server post_upgrade"
Performing database migrations...
...
20:00:52.673 INFO    nautobot.extras.utils utils.py        refresh_job_model_from_job_class() :
  Created Job "hello_jobs: HelloJobs" from <HelloJobs>

Generating cable paths...
Found no missing circuit termination paths; skipping
Found no missing console port paths; skipping
Found no missing console server port paths; skipping
Found no missing interface paths; skipping
Found no missing power feed paths; skipping
Found no missing power outlet paths; skipping
Found no missing power port paths; skipping
Finished.

Collecting static files...

0 static files copied to '/opt/nautobot/static', 1156 unmodified.

Removing stale content types...

Removing expired sessions...
...

Refreshing _content_type cache
CONTENT_TYPE_CACHE_TIMEOUT is set to 0; skipping cache refresh

Refreshing dynamic group member caches...
Refreshing DynamicGroup member caches...
```

如果我们回到 Nautobot UI 上的 `JOBS` 部分，应该能看到新创建的作业：

![hello_jobs_1](images/hello_jobs_1.png)

记住所有新工作默认是禁用的吗？我们需要通过点击编辑按钮来编辑工作：

![hello_jobs_2](images/hello_jobs_2.png)

向下滚动到启用复选框：

![hello_jobs_3](images/hello_jobs_3.png)

然后更新该作业：

![hello_jobs_4](images/hello_jobs_4.png)

现在我们可以通过点击 ```RUN``` 按钮来运行该任务：

![hello_jobs_5](images/hello_jobs_5.png)

然后点击 ```Run Job Now```：

![hello_jobs_6](images/hello_jobs_6.png)

这是你应该看到的结果：

![hello_jobs_7](images/hello_jobs_7.png)

就这样，我们的第一个任务已经运行了！

> [!TIP]
> 如果遇到任何问题，这是 [第 3 天视频演练](https://www.youtube.com/watch?v=M6lgj6VhWoA&list=PLAaTeRWIM_wtqWt3yIKmIFnfEarsuxaYs&index=4)。
> [第 3 天的旧版已弃用视频](https://www.youtube.com/watch?v=ogh9jMTTa7I)。

## 第 3 天待办事项

由于我们明天将继续在同一个文件上工作，让我们继续在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止代码空间实例，但不要删除它。

继续在你选择的社交媒体上发布新创建的任务结果的截图。确保使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将探索如何进一步自定义该任务。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+3+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制并粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 3 天，https://github.com/nautobot/100-days-of-nautobot @networktocode #JobsToBeDone #100DaysOfNautobot）
