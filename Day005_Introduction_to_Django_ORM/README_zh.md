# 使用Nautobot shell来处理Django ORM

在今天的挑战中，我们将使用Nautobot shell来处理Django ORM。

Django ORM代表`对象关系映射`，它是Django web框架提供的一个功能，用于抽象代码和数据库之间的数据库层。通过使用Django ORM，我们只需要担心编写Python代码，而不需要编写数据库命令，在这种情况下是SQL查询。

它与Nautobot jobs有什么关系？与在Nautobot Jobs中编写的代码相比，使用Django ORM而不是原始SQL语言有几个优势：

- **简化**：它通过使用Python对象来简化数据库交互。该对象可以是一个站点、一条电路、一个IP地址，或任何其他我们可以用代码表示的东西。
- **易于使用**：如我们将看到的，我们可以通过Python类（称为模型）来定义数据库架构，这比管理数据库操作更直观。
- **内置**：Django ORM是已经内置在Django框架中的工具，我们可以利用所有开发工作来获得功能、安全性、代码一致性等。

列表还在继续，我们可以深入研究额外的安全性、面向对象的优势、数据库可移植性和许多其他优势。但我们现在将停止并继续一些例子，这样我们可以看到它如何工作。

> [!NOTE]
> 由于我们在今天的挑战中讨论Django ORM + Nautobot，我有时会发现准备[Django ORM查询文档](https://docs.djangoproject.com/en/5.1/topics/db/queries/)很有帮助，以查看哪些功能直接属于Django项目。

> [!NOTE]
> 如果你为今天的实验重新启动了停止的Codespace实例，可以跳过`invoke build`和`invoke db-import`的步骤，直接进行`invoke debug`来启动容器。

让我们启动代码空间环境。Codespace启动后，我们可以使用终端窗口来构建开发环境：

```
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

保持终端窗口打开，以便我们监控日志，使用第二个终端窗口继续执行以下步骤，记住要切换到正确的目录并启动poetry：

```
$ cd nautobot-docker-compose/
$ poetry shell
```

我们可以继续下一步。

## 使用nbshell

通过`invoke`为我们提供了一些命令快捷方式，我们可以使用`invoke --list`来查看它们：

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

我们对使用Nautobot交互式shell进行今天的挑战感兴趣，我们将使用`invoke nbshell`命令启动交互式shell：

```
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke nbshell

Running docker compose command "ps --services --filter status=running"
Running docker compose command "exec nautobot nautobot-server shell_plus"
# Shell Plus Model Imports
from constance.models import Constance
from django.contrib.admin.models import LogEntry
from django.contrib.auth.models import Group, Permission
from django.contrib.contenttypes.models import ContentType
from django.contrib.sessions.models import Session
from django_celery_beat.models import ClockedSchedule, CrontabSchedule, IntervalSchedule, PeriodicTask, PeriodicTasks, SolarSchedule
from django_celery_results.models import ChordCounter, GroupResult, TaskResult
from nautobot.circuits.models import Circuit, CircuitTermination, CircuitType, Provider, ProviderNetwork
from nautobot.cloud.models import CloudAccount, CloudNetwork, CloudNetworkPrefixAssignment, CloudResourceType, CloudService, CloudServiceNetworkAssignment
from nautobot.dcim.models.cables import Cable, CablePath
from nautobot.dcim.models.device_component_templates import ConsolePortTemplate, ConsoleServerPortTemplate, DeviceBayTemplate, FrontPortTemplate, InterfaceTemplate, ModuleBayTemplate, PowerOutletTemplate, PowerPortTemplate, RearPortTemplate
from nautobot.dcim.models.device_components import ConsolePort, ConsoleServerPort, DeviceBay, FrontPort, Interface, InterfaceRedundancyGroup, InterfaceRedundancyGroupAssociation, InventoryItem, ModuleBay, PowerOutlet, PowerPort, RearPort
from nautobot.dcim.models.devices import Controller, ControllerManagedDeviceGroup, Device, DeviceFamily, DeviceRedundancyGroup, DeviceType, DeviceTypeToSoftwareImageFile, Manufacturer, Module, ModuleType, Platform, SoftwareImageFile, SoftwareVersion, VirtualChassis
from nautobot.dcim.models.locations import Location, LocationType
from nautobot.dcim.models.power import PowerFeed, PowerPanel
from nautobot.dcim.models.racks import Rack, RackGroup, RackReservation
from nautobot.extras.models.change_logging import ObjectChange
from nautobot.extras.models.contacts import Contact, ContactAssociation, Team
from nautobot.extras.models.customfields import ComputedField, CustomField, CustomFieldChoice
from nautobot.extras.models.datasources import GitRepository
from nautobot.extras.models.groups import DynamicGroup, DynamicGroupMembership, StaticGroupAssociation
from nautobot.extras.models.jobs import Job, JobButton, JobHook, JobLogEntry, JobResult, ScheduledJob, ScheduledJobs
from nautobot.extras.models.metadata import MetadataChoice, MetadataType, ObjectMetadata
from nautobot.extras.models.models import ConfigContext, ConfigContextSchema, CustomLink, ExportTemplate, ExternalIntegration, FileAttachment, FileProxy, GraphQLQuery, HealthCheckTestModel, ImageAttachment, Note, SavedView, UserSavedViewAssociation, Webhook
from nautobot.extras.models.relationships import Relationship, RelationshipAssociation
from nautobot.extras.models.roles import Role
from nautobot.extras.models.secrets import Secret, SecretsGroup, SecretsGroupAssociation
from nautobot.extras.models.statuses import Status
from nautobot.extras.models.tags import Tag, TaggedItem
from nautobot.ipam.models import IPAddress, IPAddressToInterface, Namespace, Prefix, PrefixLocationAssignment, RIR, RouteTarget, Service, VLAN, VLANGroup, VLANLocationAssignment, VRF, VRFDeviceAssignment, VRFPrefixAssignment
from nautobot.tenancy.models import Tenant, TenantGroup
from nautobot.users.models import AdminGroup, ObjectPermission, Token, User
from nautobot.virtualization.models import Cluster, ClusterGroup, ClusterType, VMInterface, VirtualMachine
from silk.models import Profile, Request, Response, SQLQuery
from social_django.models import Association, Code, Nonce, Partial, UserSocialAuth
# Shell Plus Django Imports
from django.core.cache import cache
from django.conf import settings
from django.contrib.auth import get_user_model
from django.db import transaction
from django.db.models import Avg, Case, Count, F, Max, Min, Prefetch, Q, Sum, When
from django.utils import timezone
from django.urls import reverse
from django.db.models import Exists, OuterRef, Subquery
Python 3.8.19 (default, Sep  4 2024, 06:05:29) 
[GCC 12.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
(InteractiveConsole)
>>>
```

从nbshell命令输出中首先要注意的是它在nautobot docker容器中执行了`exec nautobot nautobot-server shell_plus`。如前所述，invoke命令的行为就像完整命令行命令的快捷方式。

> [!TIP] 
> 如果你对invoke命令配置的细节感兴趣，请查看`nautobbot-docker-compose`文件夹中的`tasks.py`文件。

我们许多人都熟悉在命令提示符中输入`python3`时的Python交互式shell。`nb_shell`类似于Python shell，但添加了Django和Nautobot的额外功能。

另一件要注意的事情是所有的`import`命令。如前所述，ORM允许我们用Python`model`类表示数据库对象。Nautobot有大量预定义的模型或数据库对象，如位置、用户、电源面板和机架。它们会自动为我们导入以节省时间。

> [!NOTE]
> 这些模型还包括不那么明显的数据库对象，例如权限、计算字段、配置上下文等。

学习Django ORM及其在Nautobot中的用法的最好方法是通过一些示例。所以让我们使用nbshell来尝试Django ORM吧？

我们知道已经填充了一些位置数据，我们可以简单地使用`objects.all()`查询来检索它们：

```
>>> Location.objects.all()
<LocationQuerySet [<Location: Baltimore>, <Location: Boston>, <Location: Chicago>, <Location: Columbus>, <Location: East Coast>, <Location: Indianapolis>, <Location: Jacksonville>, <Location: New York City>, <Location: New York HQ>, <Location: Philadelphia>, <Location: Richmond>, <Location: Washington, D.C.>]>
```

我们得到的是一个[Django QuerySet](https://docs.djangoproject.com/en/5.1/ref/models/querysets/#django.db.models.query.QuerySet)，它表示来自数据库的一个对象集合，在这个例子中是Location。为了使用它，我们通常将结果分配给一个可以迭代的变量：

```
>>> locations = Location.objects.all()
>>> for location in locations: 
...     print(location)
... 
Baltimore
Boston
Chicago
Columbus
East Coast
Indianapolis
Jacksonville
New York City
New York HQ
Philadelphia
Richmond
Washington, D.C.
```

我们也可以对返回的数据应用过滤器。由于我们有两个位置类型，`store`和`office`，我们可以通过这两个类型进行过滤：

> [!IMPORTANT] 
> 注意过滤器的语法是字段的名称(location_type)，后跟双下划线(`__`)，后跟字段的名称(name)。这是[Django QuerySet API约定](https://docs.djangoproject.com/en/5.1/ref/models/querysets/#queryset-api-reference)。

```
>>> office_locations = Location.objects.filter(location_type__name="Office")
>>> store_locations = Location.objects.filter(location_type__name="Store")
>>> 

>>> for office in office_locations:
...     print(office)
... 
New York HQ
 
>>> 
>>> for store in store_locations: 
...     print(store)
... 
Baltimore
Boston
Chicago
Columbus
Indianapolis
Jacksonville
New York City
Philadelphia
Richmond
Washington, D.C.
```

我们还可以进一步链接搜索过滤器：

```
>>> bos_store_locations = Location.objects.filter(location_type__name="Store").get(name="Boston")

>>> bos_store_locations.name
'Boston'

>>> bos_store_locations.created
datetime.datetime(2024, 9, 21, 20, 51, 49, 674314, tzinfo=datetime.timezone.utc)
```

我们可以使用更多的过滤器，我们可以查阅[Django查询API文档](https://docs.djangoproject.com/en/5.1/topics/db/queries/#retrieving-specific-objects-with-filters)以了解我们可以对`QuerySet`结果进行的其他方式过滤。

既然我们可以读取现有数据，让我们看看如何添加和更新模型数据。

## 添加、删除和更新数据

Django中通常使用两种方法来创建对象，`create()`和`get_or_create()`。

`create`方法将创建对象并返回该对象，而`get_or_create`方法将在对象存在时获取它，或在对象不存在时创建该对象，它还将返回第二个对象来指示对象是否被创建。

让我们看看创建的实际效果，让我们继续创建另一个店铺。从Web UI中我们可以看到，我们需要指定"位置类型"、"名称"、"状态"作为必填字段，以及可选的"父级"字段：

![location_createion_1](images/location_creation_1.png)

让我们创建一个名称为"Charlotte"的店铺，位置类型为"Store"，父级为"East Coast"，并将状态设置为active：

```
>>> charlotte = Location.objects.get_or_create(name="Charlotte", location_type="Store", parent="East Coast", status="Active")
Traceback (most recent call last):
  File "/usr/local/lib/python3.8/site-packages/django/db/models/fields/__init__.py", line 2688, in to_python
    return uuid.UUID(**{input_form: value})
  File "/usr/local/lib/python3.8/uuid.py", line 171, in __init__
    raise ValueError('badly formed hexadecimal UUID string')
ValueError: badly formed hexadecimal UUID string

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "<console>", line 1, in <module>
  ...
  File "/usr/local/lib/python3.8/site-packages/django/db/models/fields/__init__.py", line 2690, in to_python
    raise exceptions.ValidationError(
django.core.exceptions.ValidationError: ['"Store" is not a valid UUID.']
```

嗯...我们收到了"Store不是有效的UUID"的错误。事实证明，这些是作为输入所需的Python对象。

让我们通过获取必要的对象再试一次。这些对象中的每一个都由系统中的唯一`UUID`表示：

```
>>> location_type_store = LocationType.objects.get(name="Store")
>>> location_type_store.id
UUID('b84942cf-4145-49e0-b511-9ac62b79ac63')
>>> location_eastcoast = Location.objects.get(name="East Coast")
>>> location_eastcoast.id
UUID('108625f7-15de-4546-9521-4e04125468de')
>>> status_active = Status.objects.get(name="Active")
>>> status_active.id
UUID('023e4472-398a-4351-a82f-743e69085cc3')
```

我们可以使用`get_or_create`方法来创建位置，记住我们在`get_or_create`方法中提到返回两个对象，一个是对象本身(location)，第二个是对象是否被创建的状态(true或false)，这就是为什么我们为此命令分配两个变量的原因：

```
>>> location, created = Location.objects.get_or_create(name="Charlotte", location_type=location_type_store, parent=location_eastcoast, status=status_active)

>>> location
<Location: Charlotte>
>>> location.name
'Charlotte'
>>> location.validated_save()
>>> created
True
```

我们可以使用location查询来查看新创建的店铺，或者只是浏览Web UI：

![location_creation_2](images/location_creation_2.png)

这个位置在我们输入命令时在数据库中被创建。但如果你觉得在shell中如此轻松地创建数据库条目有点奇怪，你并不孤单。

Django提供的功能之一是在使用`validated_save()`方法保存前验证数据。它有助于捕获任何错误，以查看是否存在，比如重复的名称。这是一个很好的做法，在我们提交更改之前使用它。

让我们继续删除之前创建的站点：

```
>>> Location.objects.filter(name="Charlotte").delete()
(1, {'dcim.Location': 1})
```

然后重新创建并保存它：

```
>>> location, created = Location.objects.get_or_create(name="Charlotte", location_type=location_type_store, parent=location_eastcoast, status=status_active)
>>> location.validated_save()
```

我们也可以通过检索对象并更新它来更新对象。我们看到新位置没有描述：

![location_update_1](images/location_update_1.png)

我们可以用以下步骤更新描述：

```
>>> charlotte = Location.objects.get(name="Charlotte")
>>> charlotte.name
'Charlotte'
>>> charlotte.description
''
>>> charlotte.description = "New Site"
>>> charlotte.description
'New Site'
>>> charlotte.validated_save()
```

描述被添加到新站点：

![location_update_2](images/location_update_2.png)

如果你在想，"我为什么不直接在web界面中更新站点？"我不会责怪你。对于简单的操作，在web界面中做要容易得多。但是，如果我们需要通过脚本以编程方式添加多个对象，或者我们需要通过脚本查询信息，`Django QuerySet`是我们的朋友。

我们今天取得了很大进展，Django QuerySet绝对是一个强大的工具，我们将在未来的挑战中使用它。

## 第5天待办事项

记得在[https://github.com/codespaces/](https://github.com/codespaces/)上停止代码空间实例。

继续在你选择的任何社交媒体上发布你最喜欢的ORM查询的屏幕截图，确保你使用标签`#100DaysOfNautobot` `#JobsToBeDone`并标记`@networktocode`，这样我们可以分享你的进展！

在明天的挑战中，我们回到增强我们的Nautobot Jobs。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+5+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/) (Copy & Paste: I just completed Day 5 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot)
