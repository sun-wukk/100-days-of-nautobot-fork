# Nautobot Server 命令

Nautobot 内置了一个名为 `nautobot-server` 的命令行（CLI）管理工具，常用于执行各种常见的管理任务，是一个统一的操作入口。

> [!TIP]
> 如果您有 Django 开发经验，正如 [nautobot-server](https://docs.nautobot.com/projects/core/en/stable/user-guide/administration/tools/nautobot-server/) 文档所述，`nautobot-server` 的工作方式与 Django 项目的 `manage.py` 脚本完全相同，但在此基础上添加了 Nautobot 特有的功能。

实际上，我们在前几天已经以别名的形式使用了许多 CLI 命令。例如，在第 3 天的 `docker-compose.local.yml` 文件中，我们使用 `nautobot-server runserver 0.0.0.0:8080` 在 `nautobot` 容器中启动开发服务器。在第 5 天，当我们输入 `invoke nbshell` 时，输出信息告诉我们它实际上是在运行一个 `nautobot-server` 的 Docker Compose 封装命令，示例输出如下：
```
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke nbshell

...
Running docker compose command "exec nautobot nautobot-server shell_plus"
...
```

[nautobot-server](https://docs.nautobot.com/projects/core/en/stable/user-guide/administration/tools/nautobot-server/) 文档提供了所有可用选项的完整说明。在今天的挑战中，我们将介绍几个使用示例。

## 环境配置

环境配置与 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md) 相同，以下是步骤摘要，如需详细背景说明请参阅该指南。

> [!TIP]
> 我们在 [Lab Related Notes](../Lab_Setup/lab_related_notes/README.md) 中保存了各实验场景的实用技巧，可供参考。

如果已停止 Codespace 环境，只需重新启动并按以下步骤运行 Nautobot，无需重建 Docker 实例或重新导入数据库：
```bash
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke debug
```

如果需要在 Codespace 中完全重建环境，请执行以下步骤：
```bash
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

今天的挑战无需使用 Containerlab。

## Nautobot-Server 使用示例

首先进入 `nautobot` 容器的 Shell 环境：
```bash
@ericchou1 ➜ ~ $ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash
root@ee2753f052ae:/opt/nautobot#
```

首先，我们可以尝试使用 `nautobot-server runjob [module:class] -u [user]` 从命令行运行之前创建的 Job。假设已创建包含 `HelloWorldwithLogs` Job 的 `hello_job.py` 文件：
```
root@ee2753f052ae:/opt/nautobot# nautobot-server runjob hello_job.HelloWorldwithLogs -u admin
[23:47:37] Running hello_job.HelloWorldwithLogs...
        initialization: 0 debug, 1 info, 0 warning, 0 error, 0 critical
                info: Hello World with Logs: Running job
        post_run: 0 debug, 1 info, 0 warning, 0 error, 0 critical
                info: Job completed
        run: 1 debug, 1 info, 1 warning, 1 error, 1 critical
                info: This is an log of info type.
                debug: This is an log of debug type.
                warning: This is an log of warning type.
                error: This is an log of error type.
                critical: This is an log of critical type.
[23:47:38] hello_job.HelloWorldwithLogs: SUCCESS
[23:47:38] hello_job.HelloWorldwithLogs: Duration 0 minutes, 0.30 seconds
[23:47:38] Finished
```

可以使用 `nautobot-server help` 查看所有可用选项：
```
root@ee2753f052ae:/opt/nautobot# nautobot-server help

Type 'nautobot-server help <subcommand>' for help on a specific subcommand.

Available subcommands:

[auth]
    changepassword
    createsuperuser

[constance]
    constance

[contenttypes]
    remove_stale_contenttypes

[core]
    audit_dynamic_groups
    audit_graphql_queries
    celery
    generate_secret_key
    generate_test_data
    makemigrations
    migrate
    nbshell
...
```

要查看某个 `子命令` 的详细说明，可以使用 `nautobot-server help <子命令>`。例如，使用帮助菜单查看如何创建新的超级用户：
```
root@ee2753f052ae:/opt/nautobot# nautobot-server help createsuperuser
usage: nautobot-server createsuperuser [-h] [--username USERNAME] [--noinput] [--database DATABASE] [--email EMAIL] [--version] [-v {0,1,2,3}] [--settings SETTINGS]
                                       [--pythonpath PYTHONPATH] [--traceback] [--no-color] [--force-color] [--skip-checks]

Used to create a superuser.

optional arguments:
  -h, --help            show this help message and exit
  --username USERNAME   Specifies the login for the superuser.
  --noinput, --no-input
                        Tells Django to NOT prompt the user for input of any kind. You must use --username with --noinput, along with an option for any other required
                        field. Superusers created with --noinput will not be able to log in until they're given a valid password.
  --database DATABASE   Specifies the database to use. Default is "default".
  --email EMAIL         Specifies the email for the superuser.
  --version             Show program's version number and exit.
  -v {0,1,2,3}, --verbosity {0,1,2,3}
                        Verbosity level; 0=minimal output, 1=normal output, 2=verbose output, 3=very verbose output
  --settings SETTINGS   The Python path to a settings module, e.g. "myproject.settings.main". If this isn't provided, the DJANGO_SETTINGS_MODULE environment variable will
                        be used.
  --pythonpath PYTHONPATH
                        A directory to add to the Python path, e.g. "/home/djangoprojects/myproject".
  --traceback           Raise on CommandError exceptions.
  --no-color            Don't colorize the command output.
  --force-color         Force colorization of the command output.
  --skip-checks         Skip system checks.
```

接下来创建一个新的超级用户：
```
root@ee2753f052ae:/opt/nautobot# nautobot-server createsuperuser
Username: <用户名>
Email address: <邮箱>
Password: <密码>
Password (again): <密码>
Superuser created successfully.
```

用新账号在 Web UI 中登录试试吧！

## 第 25 天待办事项

恭喜完成了四分之一的挑战！

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

今天的任务是从文档中找到另一个命令并深入了解它。欢迎在社交媒体上分享您发现的有趣命令，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将使用 `PDB` 提升调试技能。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+25+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 25 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
