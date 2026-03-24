# Nautobot 数据库模型第一部分：理解 Nautobot 的核心数据模型

在接下来的几天里，我们将熟悉 Nautobot 的部分数据模型。正如 [核心数据模型概述](https://docs.nautobot.com/projects/core/en/latest/user-guide/core-data-model/overview/introduction/) 中所述，Nautobot 核心中有近 200 个模型，这还不包括来自 Nautobot Apps 的任何模型。我们不可能在一天的挑战中涵盖所有模型。但我们可以重点介绍最常见的模型，学习它们的使用模式，了解在哪里找到文档，并将它们作为其他模型的快速参考。

正如 [Nautobot 基础模型简化图](https://docs.nautobot.com/projects/core/en/latest/media/models/model_simple.png) 所示，核心数据模型包括：

- DCIM：Location、Device、DeviceType、Manufacturer、Platform、Interface、Cable 等。
- Circuits：Circuits、Provider、CircuitTermination 等。
- IPAM：IPAddress、Prefix、Namespace、VLAN 等。

![nautobot_fundamental_models_simplified](images/nautobot_fundamental_models_simplified.png)

对于不熟悉的人，可以从 [UML 版本](https://docs.nautobot.com/projects/core/en/latest/user-guide/core-data-model/overview/introduction/#fundamental-model-uml) 中获取更多详细信息：

![nautobot_fundamental_models_uml](images/nautobot_fundamental_models_uml.png)

在今天的挑战中，我们的目标是使用各种工具来查找有关 Nautobot 特定核心数据模型及其与其他数据模型关系的信息。

## 环境设置

我们将结合使用 [场景 2](../Lab_Setup/scenario_2_setup/README.md) 实验室、[https://demo.nautobot.com/](https://demo.nautobot.com/) 和 [Nautobot 文档](https://docs.nautobot.com/projects/core/en/latest/user-guide/core-data-model/overview/introduction/) 进行今天的挑战。

```
$ cd nautobot
$ poetry shell
$ poetry install
$ invoke build
（这个步骤需要耐心）
$ invoke debug
（这个步骤也需要耐心）
```

## Device 模型示例

作为网络工程师，管理网络设备是我们工作的主要领域之一。假设我们好奇设备如何在 Nautobot 中表示及其与其他数据模型的关联。我们该怎么做？

有不同的方法，学习方式高度依赖于你的背景和经验水平。下面我提供一些学习和探索的想法。

### 演示站点

[Nautobot 演示站点](https://demo.nautobot.com/) 提供了旨在复制真实世界场景的预填充数据。

我们可以在 `/dcim/devices/` 下的 [Devices](https://demo.nautobot.com/dcim/devices/) 中看到表示，这提供了第一个线索：设备模型是 DCIM 数据模型组的一部分。

![devices_list](images/devices_list.png)

如果我们点击任何详情，它将带我们进入设备的详情视图：

![device_detail](images/device_detail.png)

从这个视图中，我们可以有根据地猜测设备有诸如名称、序列号、资产标签和其他字段。

由于我们可以点击 Location、Device Types 和 Device Family 等字段，我们也可以合理地猜测这些是与设备数据模型有关联的其他数据模型。

### 文档

事实上，我们可以通过阅读 [基础模型 UML](https://docs.nautobot.com/projects/core/en/latest/user-guide/core-data-model/overview/introduction/#fundamental-model-uml) 来消除一些猜测：

![device_model_UML](images/device_model_UML.png)

[设备](https://docs.nautobot.com/projects/core/en/latest/user-guide/core-data-model/dcim/device/) 文档确实提供了更多关于该模型的信息，例如必需的赋值：

```
每个设备必须被分配一个 location、device role 和 operational status，并且可以选择性地被分配到一个 location 内的 rack。平台、序列号和资产标签可以可选地分配给每个设备。

设备名称在同一个 location 内必须唯一，除非该设备已被分配给租户。设备也可以没有名称。
```

下一步，我们将尝试查看与 `dcim.devices` 数据模型相关的 Nautobot 源代码。

### 源代码

我们可以启动我们的 Codespace 并查看源代码。我们看到模型定义在 `nautobot -> dcim -> models -> devices.py`：

![codespace_1](images/codespace_1.png)

我们可以看到这个文件中定义了许多模型，如 `Manufacturer`、`DeviceFamily`、`DeviceType`、`Platform`，当然还有 `Device`：

```python
@extras_features(
    "custom_links",
    "custom_validators",
    "export_templates",
    "graphql",
    "locations",
    "statuses",
    "webhooks",
)
class Device(PrimaryModel, ConfigContextModel):
    """
    A Device represents a piece of physical hardware. Each Device is assigned a DeviceType,
    Role, and (optionally) a Platform. Device names are not required, however if one is set it must be unique.

    Each Device must be assigned to a Location, and optionally to a Rack within that.
    Associating a device with a particular rack face or unit is optional (for example, vertically mounted PDUs
    do not consume rack units).

    When a new Device is created, console/power/interface/device bay components are created along with it as dictated
    by the component templates assigned to its DeviceType. Components can also be added, modified, or deleted after the
    creation of a Device.
    """

    device_type = models.ForeignKey(to="dcim.DeviceType", on_delete=models.PROTECT, related_name="devices")
    status = StatusField(blank=False, null=False)
    role = RoleField(blank=False, null=False)
    tenant = models.ForeignKey(
        to="tenancy.Tenant",
        on_delete=models.PROTECT,
        related_name="devices",
        blank=True,
        null=True,
    )
    platform = models.ForeignKey(
        to="dcim.Platform",
        on_delete=models.SET_NULL,
        related_name="devices",
        blank=True,
        null=True,
    )
    name = models.CharField(
        max_length=CHARFIELD_MAX_LENGTH,
        blank=True,
        null=True,
        db_index=True,
    )
    serial = models.CharField(max_length=CHARFIELD_MAX_LENGTH, blank=True, verbose_name="Serial number", db_index=True)
    asset_tag = models.CharField(
        max_length=CHARFIELD_MAX_LENGTH,
        blank=True,
        null=True,
        unique=True,
        verbose_name="Asset tag",
        help_text="A unique tag used to identify this device",
    )
    location = models.ForeignKey(
        to="dcim.Location",
        on_delete=models.PROTECT,
        related_name="devices",
    )
    rack = models.ForeignKey(
        to="dcim.Rack",
        on_delete=models.PROTECT,
        related_name="devices",
        blank=True,
        null=True,
    )
    # ... 更多字段
```

字段的详细说明超出了今天挑战的范围，[Django Models](https://docs.djangoproject.com/en/5.1/topics/db/models/) 文档很好地提供了字段、字段类型、关系等的详细描述。

### 使用 nbshell

继续创建一个具有必要赋值（如 location、device role 和 status）的设备。然后尝试使用 `nbshell` 查询设备并探索其与其他数据模型的关系：

```shell
$ docker exec -it nautobot-2-4-nautobot-1 bash

root@c8032ee34216:/source# nautobot-server nbshell
...

# 检索所有设备
>>> devices = Device.objects.all()

# 遍历设备
>>> for device in devices:
...     print(device.name, device.device_type, device.device_role, device.site)
...

# 按名称检索特定设备
device = Device.objects.get(name="DeviceName")
print(device.name, device.device_type, device.device_role, device.site)

# 按 site 筛选设备
site_devices = Device.objects.filter(site__name="SiteName")
for device in site_devices:
    print(device.name, device.device_type, device.device_role, device.site)
```

恭喜完成第 61 天，这是 Nautobot 模型的 5 天第一天，敬请期待更多内容！

## 第 61 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布今天挑战中学到的东西，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将学习更多关于创建自定义 Nautobot 数据模型的知识。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+61+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 61 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）