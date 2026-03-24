# 顶点项目第十一部分。第 90 天：为 CVE 创建自定义 Nautobot 模型

## 目标

在**第 90 天**，我们将通过创建自定义数据模型来存储 CVE（常见漏洞和披露）来增强我们的 Nautobot 插件。与使用 JSON 自定义字段相比，这种方法提供了更好的数据管理和集成。

## 为什么要从 JSON 自定义字段迁移到 Nautobot 数据模型？

以前，CVE 存储在 JSON 自定义字段中，如下所示：

```json
{
    "CVE-2023-1234": {
        "cvss_base_score": 9.8,
        "link": "https://example.com/CVE-2023-1234",
        "severity": "Critical"
    }
}
```

虽然这适用于基本用例，但它有几个局限性：

❌ 无关系完整性或内置查询
❌ 无法利用 Nautobot 的变更跟踪、过滤或对象权限
❌ 难以扩展或将 CVE 链接到其他模型

通过创建自定义模型，你将获得：

✅ 在 UI 和 API 中完全支持查询/过滤/搜索
✅ 与其他模型（例如 SoftwareVersion）的关联连接
✅ 更好的可见性、权限和可审计性

## 关系：CVE ↔ 软件版本

一个 CVE 可以影响多个软件版本，一个软件版本可以有多个 CVE。这是一个经典的多对多关系。

### 关系参考摘要：

- **一对一（1:1）**：表 A 中的每条记录在表 B 中有一条匹配的记录，反之亦然。
- **一对多（1:N）**：表 A 中的一条记录可以关联到 B 中的多条记录，但 B 中的每条记录只关联到 A 中的一条记录。
- **多对多（N:N）**：A 中的记录可以关联到 B 中的多条记录，反之亦然。

在我们的例子中：

- 一个 CVE → 影响多个软件版本
- 一个软件版本 → 可能受到多个 CVE 的影响

## 实现 CVE 模型

我们将定义一个与 `SoftwareVersion` 具有多对多关系的 `CVE` 模型。

### 代码实现

在 **`nautobot_software_cves/`** 目录中，找到 **`models.py`** 文件或如果你在第 80 天第 9 步选择了 "None" 则创建它，并插入以下代码：

在 `models.py` 中：

```python
from django.db import models

try:
    from nautobot.apps.constants import CHARFIELD_MAX_LENGTH
except ImportError:
    CHARFIELD_MAX_LENGTH = 255

from nautobot.apps.models import PrimaryModel
from nautobot.apps.choices import ChoiceSet

class CVESeverityChoices(ChoiceSet):
    """CVE 严重性级别的选项。"""

    CRITICAL = "Critical"
    HIGH = "High"
    MEDIUM = "Medium"
    LOW = "Low"
    NONE = "None"

    CHOICES = (
        (CRITICAL, CRITICAL),
        (HIGH, HIGH),
        (MEDIUM, MEDIUM),
        (LOW, LOW),
        (NONE, NONE),
    )

class CVE(PrimaryModel):
    """表示 CVE 的模型。"""

    name = models.CharField(max_length=CHARFIELD_MAX_LENGTH, unique=True)
    link = models.URLField()
    severity = models.CharField(
        max_length=CHARFIELD_MAX_LENGTH,
        choices=CVESeverityChoices,
        default=CVESeverityChoices.NONE
    )
    cvss = models.FloatField(null=True, blank=True, verbose_name="CVSS Base Score")
    affected_softwares = models.ManyToManyField(
        to="dcim.SoftwareVersion",
        related_name="corresponding_cves",
        blank=True
    )

    class Meta:
        verbose_name = "CVE"
        ordering = ("severity", "name")

    def __str__(self):
        return self.name
```

### 说明

- **PrimaryModel** 是 Nautobot 中用于全功能数据模型的类，支持大部分或全部 Nautobot 对基本 Django 功能的扩展。通常，这是你要用于表示网络中不同"事物"的任何数据模型的类。
- 注意，我们在 **name** 字段上使用 `unique=True` 来指定这在 Nautobot 数据库中必须是全局唯一的。
- **CVESeverityChoices**：为 severity 字段提供一致的值，使过滤和验证更容易。
- **name**：CVE ID（例如 "CVE-2023-1234"）存储在这里。
- **link**：指向官方 CVE 参考的 URL。
- **severity**：严重性级别，从预定义选项中选择。
- **cvss**：表示 CVSS 基本分数的浮点数。
- **affected_softwares**：与 `SoftwareVersion` 建立多对多关系。
  - `related_name` 属性允许使用 `corresponding_cves` 从 `SoftwareVersion` 反向查询关联的 `CVE` 实例。

- **Meta 选项**：
  - `verbose_name` 为模型提供人类可读的名称，用于管理界面。
  - `ordering` 指定查询结果的默认排序，首先按 `severity`，然后按 `name`。

这个结构化模型支持完整的 ORM 集成和关系，这是 JSON 字段所缺乏的。

## 应用迁移和验证

定义 `CVE` 模型后，你需要将更改应用到数据库架构。这涉及两个步骤：

### 1. 创建迁移文件

运行以下命令基于你的模型更改生成迁移文件：

```
$ invoke makemigrations
Running docker compose command "ps --services --filter status=running"
Running docker compose command "exec nautobot nautobot-server makemigrations nautobot_software_cves"
Migrations for 'nautobot_software_cves':
  nautobot_software_cves/migrations/0002_cve.py
    - Create model CVE
```

此命令分析你的模型并创建描述更新数据库架构所需更改的迁移文件。这些文件存储在应用 `migrations` 目录中。生成的文件 `nautobot_software_cves/migrations/0001_initial.py` 基本上是一组指示 Django 如何将你的 Python 代码转换为 SQL 的指令。

### 2. 应用迁移

创建迁移文件后，应用它们以更新你的数据库架构：

```
$ invoke migrate
Running docker compose command "ps --services --filter status=running"
Running docker compose command "exec nautobot nautobot-server migrate"
Operations to perform:
  Apply all migrations: admin, auth, circuits, cloud, constance, contenttypes, dcim, django_celery_beat, django_celery_results, extras, ipam, nautobot_software_cves, sessions, silk, social_django, taggit, tenancy, users, virtualization
Running migrations:
  Applying nautobot_software_cves.0002_cve... OK
12:53:08.839 INFO    nautobot.extras.utils utils.py        refresh_job_model_from_job_class() : Refreshed Job "System Jobs: Export Object List" from <ExportObjectList>
12:53:08.843 INFO    nautobot.extras.utils utils.py        refresh_job_model_from_job_class() : Refreshed Job "System Jobs: Git Repository: Sync" from <GitRepositorySync>
12:53:08.849 INFO    nautobot.extras.utils utils.py        refresh_job_model_from_job_class() : Refreshed Job "System Jobs: Git Repository: Dry-Run" from <GitRepositoryDryRun>
12:53:08.854 INFO    nautobot.extras.utils utils.py        refresh_job_model_from_job_class() : Refreshed Job "System Jobs: Import Objects" from <ImportObjects>
12:53:08.859 INFO    nautobot.extras.utils utils.py        refresh_job_model_from_job_class() : Refreshed Job "System Jobs: Logs Cleanup" from <LogsCleanup>
12:53:08.863 INFO    nautobot.extras.utils utils.py        refresh_job_model_from_job_class() : Refreshed Job "System Jobs: Refresh Dynamic Group Caches" from <RefreshDynamicGroupCaches>
12:53:08.867 INFO    nautobot.extras.utils utils.py        refresh_job_model_from_job_class() : Refreshed Job "CVE Tracking: Load Vulnerabilities" from <LoadCVEsJob>
```

此命令执行迁移文件中定义的 SQL 语句，根据需要创建或修改数据库表。

通过完成这些步骤，你的 `CVE` 模型将完全集成到 Nautobot 数据库中，允许你有效地管理 CVE 记录。

### 3. 验证

你可以使用 invoke nbshell 命令验证此模型现在已定义，如下所示，当然，你的数据库中实际上还没有特定的 CVE 记录：

```bash
❯ invoke nbshell
Running docker compose command "ps --services --filter status=running"
Running docker compose command "exec nautobot nautobot-server nbshell  "
# Django version 4.2.19
# Nautobot version 2.3.2
# Nautobot Software Cves version 0.1.0
Python 3.11.9 (main, Sep  4 2024, 07:49:21) [GCC 12.2.0]
Type 'copyright', 'credits' or 'license' for more information
IPython 8.12.3 -- An enhanced Interactive Python. Type '?' for help.
In [2]: from nautobot_software_cves.models import CVE
In [3]: CVE.objects.all()
Out[3]: <RestrictedQuerySet []>
```

## 最后的想法

从 JSON 自定义字段过渡到适当的 Nautobot 模型允许可扩展、健壮和可查询的解决方案，随你的网络自动化平台一起成长。

## 下一步

在接下来的几天中，我们将：

- 增强 Nautobot UI 以显示与软件版本关联的 CVE。
- 重构 CVE 加载器作业以填充这个新模型。

敬请期待第 91 天！