# App 打包与发布（下篇）

欢迎回到应用发布流程的第二部分，让我们继续。

## 环境搭建

我们将使用 `nautobot-docker-compose` 实例来安装第 42 天创建、第 44 天导出的 App。

完整的容器启动说明见 `scenario 1`，请参阅 [scenario_1_setup](../Lab_Setup/scenario_1_setup/README.md) 复习搭建步骤。

以下是进入 Codespace 后的步骤摘要。如果环境是从之前的实验重启的且已执行过相关步骤，请跳过 `invoke build` 和 `invoke db-import`：

```
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

今天的挑战不需要 Containerlab。

## 安装示例

将 wheel 文件上传到 `nautobot-docker-compose` 目录：

![upload_wheel_file](images/upload_wheel_file.png)

然后使用 `docker cp` 将安装包复制到 Nautobot 容器的 `/tmp/` 目录：

```
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ docker cp my_awesome_app-0.1.0-py3-none-any.whl nautobot_docker_compose-nautobot-1:/tmp/
Successfully copied 17.4kB to nautobot_docker_compose-nautobot-1:/tmp/
```

进入 Nautobot 容器，用 `pip install` 安装该包：

```
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ docker exec -it -u root nautobot_docker_compose-nautobot-1 bash
root@48e06f355729:/opt/nautobot# cd /tmp
root@48e06f355729:/tmp# ls
dist  my_awesome_app-0.1.0-py3-none-any.whl

root@48e06f355729:/tmp# pip install my_awesome_app-0.1.0-py3-none-any.whl 
Processing ./my_awesome_app-0.1.0-py3-none-any.whl
Requirement already satisfied: nautobot<3.0.0,>=2.0.0 in /usr/local/lib/python3.8/site-packages (from my-awesome-app==0.1.0) (2.3.2)
Requirement already satisfied: Django<4.3.0,>=4.2.15 in /usr/local/lib/python3.8/site-packages (from nautobot<3.0.0,>=2.0.0->my-awesome-app==0.1.0) (4.2.15)
...
Installing collected packages: my-awesome-app
Successfully installed my-awesome-app-0.1.0

root@48e06f355729:/tmp# pip list | grep my-awesome
my-awesome-app                 0.1.0
```

在 `config -> nautobot_config.py` 的 `PLUGIN` 配置项中添加新 App：

![nautobot_config_1](images/nautobot_config_1.png)

此时会看到新 App 提示"未应用数据库迁移"的错误：

![nautobot_config_2](images/nautobot_config_2.png)

**不要停止**原有实例，另开一个终端执行 `invoke post-upgrade`：

> [!INFORMATION]
> 若停止实例后再重启，由于每个 Docker 实例都从全新状态启动，会出现找不到 App 的报错。

```
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke post-upgrade
Running docker compose command "ps --services --filter status=running"
Running docker compose command "exec nautobot nautobot-server post_upgrade"
Performing database migrations...
Operations to perform:
  Apply all migrations: admin, auth, circuits, cloud, constance, contenttypes, dcim, django_celery_beat, django_celery_results, extras, ipam, sessions, silk, social_django, taggit, tenancy, users, virtualization
Running migrations:
  No migrations to apply.
  Your models in app(s): 'my_awesome_app' have changes that are not yet reflected in a migration, and so won't be applied.
  Run 'manage.py makemigrations' to make new migrations, and then re-run 'manage.py migrate' to apply them.
```

也可以直接在 Nautobot 容器内用 `nautobot-server` 命令依次执行 `makemigrations` 和 `migrate`：

```
root@543d3084cedf:/opt/nautobot# nautobot-server makemigrations
Migrations for 'my_awesome_app':
  /usr/local/lib/python3.8/site-packages/my_awesome_app/migrations/0001_initial.py
    - Create model MyAwesomeAppExampleModel
root@543d3084cedf:/opt/nautobot# nautobot-server migrate
Operations to perform:
  Apply all migrations: admin, auth, circuits, cloud, constance, contenttypes, dcim, django_celery_beat, django_celery_results, extras, ipam, my_awesome_app, sessions, silk, social_django, taggit, tenancy, users, virtualization
Running migrations:
  No migrations to apply.
14:46:21.723 INFO    nautobot.extras.utils utils.py        refresh_job_model_from_job_class() :
  Refreshed Job "System Jobs: Export Object List" from <ExportObjectList>
14:46:21.732 INFO    nautobot.extras.utils utils.py        refresh_job_model_from_job_class() :
  Refreshed Job "System Jobs: Git Repository: Sync" from <GitRepositorySync>
14:46:21.738 INFO    nautobot.extras.utils utils.py        refresh_job_model_from_job_class() :
  Refreshed Job "System Jobs: Git Repository: Dry-Run" from <GitRepositoryDryRun>
14:46:21.750 INFO    nautobot.extras.utils utils.py        refresh_job_model_from_job_class() :
  Refreshed Job "System Jobs: Import Objects" from <ImportObjects>
14:46:21.756 INFO    nautobot.extras.utils utils.py        refresh_job_model_from_job_class() :
  Refreshed Job "System Jobs: Logs Cleanup" from <LogsCleanup>
14:46:21.763 INFO    nautobot.extras.utils utils.py        refresh_job_model_from_job_class() :
  Refreshed Job "System Jobs: Refresh Dynamic Group Caches" from <RefreshDynamicGroupCaches>
```

在 `Installed Apps` 页面，可以看到我们的新 App 已成功安装：

![new_app_installed](images/new_app_installed.png)

出色地完成了今天的挑战！

## 第 45 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 Codespace 实例。

请在你选择的社交媒体上发布新 App 安装到 Nautobot 实例的截图，务必使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，这样我们可以分享你的进展！

明天的挑战，我们将深入探索一些 Django 代码。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+45+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 45 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
