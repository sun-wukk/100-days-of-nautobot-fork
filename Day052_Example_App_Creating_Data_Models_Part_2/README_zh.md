# 示例 App 数据模型开发（下篇）

今天的挑战是昨天工作的延续。

我们将创建一个简单的数据模型，包含描述（description）和链接地址（URL）两个字段。

## 代码示例

回顾一下 `ExampleModel` 的结构——它有一个 `@extras_features` 装饰器、一个 `Meta` 内部类，以及数据库字段本身的约束条件，例如 `CharField` 的 `max_length`：

```python
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
```

后续几天我们会逐一深入讲解这些内容。对于我们自己的数据模型，直接使用 Nautobot 的 `BaseModel`，只定义两个字段即可：

```python
# models.py
from django.db import models
from nautobot.core.models import BaseModel

class UsefulLink(BaseModel):
    url = models.URLField(unique=True)
    description = models.CharField(max_length=255)

    def __str__(self):
        return self.url
```

你是否好奇 `def __str__(self)` 在数据库类中的作用？如果你装了 GitHub Copilot，可以直接用自然语言提问（我觉得这功能真的很酷）：

![copilot_1](images/copilot_1.png)

以下是更新后 `models.py` 的完整内容：

```python
# models.py
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

class UsefulLink(BaseModel):
    url = models.URLField(unique=True)
    description = models.CharField(max_length=255)

    def __str__(self):
        return self.url
```

数据库结构变更后，需要执行 `makemigrations` 和 `migrate` 使其生效。我们直接在 nautobot 容器内操作：

```shell
root@c8032ee34216:/opt/nautobot# nautobot-server makemigrations
Migrations for 'example_app':
  /source/examples/example_app/example_app/migrations/0008_usefullink.py
    - Create model UsefulLink
root@c8032ee34216:/opt/nautobot# nautobot-server migrate
Operations to perform:
  Apply all migrations: admin, auth, circuits, cloud, constance, contenttypes, dcim, django_celery_beat, django_celery_results, example_app, extras, ipam, sessions, silk, social_django, taggit, tenancy, users, virtualization, wireless
Running migrations:
  Applying example_app.0008_usefullink... OK
22:19:13.100 INFO    nautobot.extras.utils utils.py        refresh_job_model_from_job_class() :
  Refreshed Job "System Jobs: Bulk Delete Objects" from <BulkDeleteObjects>
```

没有报错，是个好兆头。但如何向新数据表中添加数据呢？可以使用管理后台。

## 注册到管理后台

Django 和 Python 一样"开箱即用"，内置了管理后台。若要让新的数据模型出现在管理界面中，需要在 `example_app` 目录下的 `admin.py` 文件中进行注册：

![admin_1](images/admin_1.png)

以下是 `admin.py` 的完整内容。注意，我们导入了新的 `UsefulLink` 数据模型，并将其字段加入 `list_display`：

```python
# admin.py
from django.contrib import admin

from nautobot.apps.admin import NautobotModelAdmin

from example_app.models import ExampleModel, UsefulLink


@admin.register(ExampleModel)
class ExampleModelAdmin(NautobotModelAdmin):
    list_display = ("name", "number")

@admin.register(UsefulLink)
class UsefulLinkAdmin(admin.ModelAdmin):
    list_display = ('url', 'description')
    search_fields = ('url', 'description')
```

通过 `https://<url>/admin/` 登录管理后台：

![admin_panel_1](images/admin_panel_1.png)

然后添加一些常用链接：

![add_link_1](images/add_link_1.png)

![useful_links_list](images/useful_links_list.png)

我个人推荐添加 [Nautobot 用户指南](https://docs.nautobot.com/projects/core/en/stable/user-guide/) 和 [Nautobot 开发者指南](https://docs.nautobot.com/projects/core/en/stable/development/)这两个链接。

仅需寥寥几行代码，就能创建一个全新的数据模型并开始填充数据，是不是很酷？

## 第 52 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 Codespace 实例。

请在你选择的社交媒体上发布新添加的数据条目截图，务必使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，这样我们可以分享你的进展！

明天的挑战，我们将把这个数据模型与视图关联起来。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+52+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 52 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
