# 示例 App 数据模型开发（上篇）

在接下来的示例 App 开发中，我们将新增一个"常用链接"页面，作为快速参考工具。到第 59 天结束时，我们最终将实现以下功能：

- 新建一个"常用链接"数据库模型。
- 通过管理后台向该模型添加数据。
- 新建一个整合了新数据模型的视图。
- 新建一个用于展示这些链接的 HTML 模板。
- 为新模板配置新的 URL 路由。
- 在 example_app 中添加指向新 URL 的导航链接。

![final_version_1](images/final_version_1.png)

我们将通过管理后台添加链接数据：

![final_version_1](images/final_version_admin_panel.png)

与大多数新应用开发一样，我们从数据库模型开始。

## models.py

我们知道示例 App 中需要修改的文件大概叫 `models.py`，在 Nautobot 仓库中找一找，可以找到这个 [models.py](https://github.com/nautobot/nautobot/blob/develop/examples/example_app/example_app/models.py) 文件。来在我们的环境里定位它。

用 `docker ps` 查看正在运行的容器：

```
@ericchou1 ➜ ~ $ docker ps
CONTAINER ID   IMAGE                                 COMMAND                  CREATED      STATUS                       PORTS                                                                                                          NAMES
6d89d5e18d29   local/nautobot-dev:local-2.4-py3.12   "watchmedo auto-rest…"   3 days ago   Up About an hour (healthy)   8080/tcp                                                                                                       nautobot-2-4-celery_beat-1
ed1202e56658   local/nautobot-dev:local-2.4-py3.12   "watchmedo auto-rest…"   3 days ago   Up About an hour (healthy)   0.0.0.0:6898->6898/tcp, :::6898->6898/tcp, 0.0.0.0:8081->8080/tcp, :::8081->8080/tcp                           nautobot-2-4-celery_worker-1
c8032ee34216   local/nautobot-dev:local-2.4-py3.12   "/docker-entrypoint.…"   3 days ago   Up About an hour (healthy)   0.0.0.0:6899->6899/tcp, :::6899->6899/tcp, 0.0.0.0:8080->8080/tcp, :::8080->8080/tcp                           nautobot-2-4-nautobot-1
8dfdf6d852e1   selenium/standalone-firefox:4.27      "/opt/bin/entry_poin…"   3 days ago   Up About an hour             5900/tcp, 0.0.0.0:4444->4444/tcp, :::4444->4444/tcp, 0.0.0.0:15900->15900/tcp, :::15900->15900/tcp, 9000/tcp   nautobot-2-4-selenium-1
a4251f9acfae   redis:6-alpine                        "docker-entrypoint.s…"   3 days ago   Up About an hour (healthy)   6379/tcp                                                                                                       nautobot-2-4-redis-1
d5e11fb88308   postgres:13                           "docker-entrypoint.s…"   3 days ago   Up About an hour (healthy)   5432/tcp                                                                                                       nautobot-2-4-db-1
```

进入 nautobot 容器，查看模型文件：

```
@ericchou1 ➜ ~ $ docker exec -it nautobot-2-4-nautobot-1 bash

root@c8032ee34216:/source# cat examples/example_app/example_app/models.py 
from django.db import models
from nautobot.core.models import BaseModel 
from nautobot.apps.constants import CHARFIELD_MAX_LENGTH
from nautobot.apps.models import extras_features, OrganizationalModel


@extras_features(
    "custom_links",
    "custom_validators",
    "export_templates",
    "graphql",
    "webhooks",
)
class ExampleModel(OrganizationalModel):
    name = models.CharField(max_length=CHARFIELD_MAX_LENGTH, help_text="The name of this Example.", unique=True)
    number = models.IntegerField(default=100, help_text="The number of this Example.")

    class Meta:
        ordering = ["name"]

    def __str__(self):
        return f"{self.name} - {self.number}"


@extras_features(
    "custom_validators",
    "export_templates",
    # "graphql"，此处未指定，因为该模型有自定义类型，详见 example_app.graphql.types
    "webhooks",
    "relationships",  # 在此显式声明以避免冲突：https://github.com/nautobot/nautobot/issues/3592
)
class AnotherExampleModel(OrganizationalModel):
    name = models.CharField(max_length=CHARFIELD_MAX_LENGTH, unique=True)
    number = models.IntegerField(default=100)

    # 默认情况下，natural key 仅为 "name"（因为它是唯一字段），但我们可以覆盖这一行为：
    natural_key_field_names = ["name", "number"]

    class Meta:
        ordering = ["name"]
```

文件中定义了两个模型：`ExampleModel` 和 `AnotherExampleModel`。我们可以在 Web UI 的两处位置验证这一点。

第一处是 `Installed Apps -> Example Nautobot App` 页面，其中的 `Data Models` 区块列出了所有数据模型：

![example_app_models](images/example_app_models.png)

另一处是导航菜单，其中有一个 `Example Models` 链接，点击后可以看到示例数据的列表视图：

![example_app_models_2](images/example_app_models_2.png)

在我们动手添加自己的数据模型时，请记住这两处位置。

最后一步是在 VSCode 资源管理器中找到这个文件：

![models_file](images/models_file.png)

恭喜完成今天的任务！找对文件、知道去哪里找，看似微不足道，但相信我，对于一个已有多年开发历史的大型软件项目而言，这绝非易事！

## 第 51 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 Codespace 实例。

请在你选择的社交媒体上发布今天任意步骤的截图，务必使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，这样我们可以分享你的进展！

明天的挑战，我们将正式创建新的数据库模型。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+51+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 51 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
