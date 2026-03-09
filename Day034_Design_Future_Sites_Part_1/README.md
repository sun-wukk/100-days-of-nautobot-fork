# 设计未来站点（第一部分）

恭喜完成 33 天的 Nautobot Job 挑战！现在我们已经掌握了应对更大规模 Job 所需的全部工具。今天标志着为期 6 天系列的开始，我们将迭代地构建一个用于设计未来站点的 Job。

准备好了吗？让我们从配置环境开始。

## 环境配置

环境配置与 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md) 相同，以下是步骤摘要，如需详细背景说明请参阅该指南。

> [!TIP]
> 如果您停止了 Codespace 环境后重新启动，发现 Docker 守护进程无法正常工作，请按照配置指南中的步骤重建环境。

按照以下步骤启动 Nautobot：
```
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

## 设计未来站点第一部分代码

为今天的挑战创建文件，可以通过共享目录或直接在 Nautobot Docker 容器中操作：
```shell
$ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash
root@c9e0fa2a45a0:/opt/nautobot# cd jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# pwd
/opt/nautobot/jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# touch create_site_job.py
root@c9e0fa2a45a0:/opt/nautobot/jobs# chown nautobot:nautobot create_site_job.py
```

今天挑战的环境已配置完毕。

## 操作步骤

本挑战跨越多个练习，涵盖第 34 至 39 天。每天我们都会在前一天代码的基础上进行扩展，逐步完善自动化功能。

今天，我们将首先创建必要的前置条件，这些条件是创建包含所有机架和设备的新站点所必需的。

正如前几课所见，Nautobot 中的某些对象有依赖关系，必须先创建依赖对象才能创建目标对象。例如，如果想在创建前缀时为其添加角色（ROLE），该角色必须已在 Nautobot 中存在。

为了充分配置本项目的环境，我们将创建前缀角色、新的租户（Tenant）、VLAN 和设备类型（DeviceType）。设备类型将留到第 35 天创建。今天我们需要用到以下数据：
```python
PREFIX_ROLES = ["p2p", "loopback", "server", "mgmt", "pop"]
TENANT_NAME = "Data Center"
VLAN_INFO = {
    "server": 1000,
    "mgmt": 99,
}
CUSTOM_FIELDS = {
    "role": {"models": [Interface], "label": "Role"},
}
# 获取 Prefix 和 VLAN 模型的内容类型
prefix_ct = ContentType.objects.get_for_model(Prefix)
vlan_ct = ContentType.objects.get_for_model(VLAN)
```

逐节解析这些数据。第一个常量是将在 Nautobot 中创建的角色列表，我们需要 `p2p`、`loopback`、`server`、`mgmt` 和 `pop` 这些角色，以便将其附加到我们创建的对象（如 VLAN、PREFIX 和 IP 地址）上。

第二个常量 `TENANT_NAME` 用于指定租户名称，POP 站点中的对象将归属于该租户。

`VLAN_INFO` 是一个键值字典，指定了将为站点创建的 VLAN 名称和 ID。

最后，Prefix 和 VLAN 的内容类型将允许我们在后续构建站点时与已创建的对象进行交互。

## 创建前缀角色

我们将实现的第一个方法是根据 `PREFIX_ROLES` 中的字符串创建角色。请注意 `create_prefix_roles` 方法中的 `ContentType` 导入和相关代码段——我们确保在后续构建站点时，这些角色可以应用于 PREFIX 和 VLAN 对象。
```python
"""用于创建 POP 类型新站点（支持可选父站点）的 Job。"""
from django.contrib.contenttypes.models import ContentType

from nautobot.apps.jobs import Job, register_jobs
from nautobot.extras.models.roles import Role
from nautobot.ipam.models import Prefix, VLAN

name = "Data Population Jobs Collection"

PREFIX_ROLES = ["p2p", "loopback", "server", "mgmt", "pop"]

def create_prefix_roles(logger):
    """创建 PREFIX_ROLES 中定义的所有前缀角色，并为其添加 IPAM Prefix 和 VLAN 的内容类型。"""

    for role in PREFIX_ROLES:
        role_obj, created = Role.objects.get_or_create(name=role)
        # 为角色添加 Prefix 和 VLAN 内容类型
        role_obj.content_types.add(prefix_ct, vlan_ct)
        role_obj.validated_save()
        logger.info(f"Successfully created role {role} with content types for Prefix and VLAN.")

class CreatePop(Job):
    """用于创建 POP 类型新站点的 Job。"""

    class Meta:
        """CreatePop 的元数据。"""

        name = "Create a Point of Presence"
        description = """
        Create a new Site of Type POP.
        A new /16 will automatically be allocated from the 'POP Global Pool' Prefix.
        """

    def run(self):
        """创建站点的主函数。"""
        # ----------------------------------------------------------------------------
        # 使用所有必需对象初始化数据库。
        # 我们将在接下来的几天中逐步扩展。
        # ----------------------------------------------------------------------------
        create_prefix_roles(self.logger)

register_jobs(CreatePop)
```

您可以随时运行这段代码，也可以等到最后再运行。我们全程使用 `get_or_create` 方法，因此即使对象已存在也不会报错。

接下来，我们添加创建租户的部分。

`create_tenant` 方法非常简单，只需使用 `TENANT_NAME` 常量中定义的 `Data Center` 字符串，创建一个具有该名称的租户对象，供后续练习使用。
```python
"""用于创建 POP 类型新站点（支持可选父站点）的 Job。"""
from django.contrib.contenttypes.models import ContentType

from nautobot.apps.jobs import Job, register_jobs
from nautobot.extras.models.roles import Role
from nautobot.ipam.models import Prefix, VLAN
from nautobot.tenancy.models import Tenant

name = "Data Population Jobs Collection"

PREFIX_ROLES = ["p2p", "loopback", "server", "mgmt", "pop"]
TENANT_NAME = "Data Center"

def create_prefix_roles(logger):
    """创建 PREFIX_ROLES 中定义的所有前缀角色，并为其添加 IPAM Prefix 和 VLAN 的内容类型。"""

    # 获取 Prefix 和 VLAN 模型的内容类型
    prefix_ct = ContentType.objects.get_for_model(Prefix)
    vlan_ct = ContentType.objects.get_for_model(VLAN)

    for role in PREFIX_ROLES:
        role_obj, created = Role.objects.get_or_create(name=role)
        # 为角色添加 Prefix 和 VLAN 内容类型
        role_obj.content_types.add(prefix_ct, vlan_ct)
        role_obj.validated_save()
        logger.info(f"Successfully created role {role} with content types for Prefix and VLAN.")

def create_tenant(logger):
    """使用 TENANT_NAME 定义的名称创建租户。"""
    tenant_obj, _ = Tenant.objects.get_or_create(name=TENANT_NAME)
    tenant_obj.validated_save()
    logger.info(f"Successfully created Tenant {TENANT_NAME}.")

class CreatePop(Job):
    """用于创建 POP 类型新站点的 Job。"""

    class Meta:
        """CreatePop 的元数据。"""

        name = "Create a Point of Presence"
        description = """
        Create a new Site of Type POP.
        A new /16 will automatically be allocated from the 'POP Global Pool' Prefix.
        """

    def run(self):
        """创建站点的主函数。"""
        # ----------------------------------------------------------------------------
        # 使用所有必需对象初始化数据库。
        # 我们将在接下来的几天中逐步扩展。
        # ----------------------------------------------------------------------------
        create_prefix_roles(self.logger)
        create_tenant(self.logger)

register_jobs(CreatePop)
```

接下来，我们将创建 VLAN 并为每个 VLAN 分配相应的角色。

我们还新增了一个名为 `ACTIVE_STATUS` 的常量，用于获取 Nautobot Extras 中的"Active"状态，以便在构建站点时为对象分配状态属性。`create_vlans` 方法中的 VLAN 创建就需要用到这个状态。

注意，我们在此处为每个 VLAN 获取对应的角色名称：
```python
ACTIVE_STATUS = Status.objects.get(name="Active")
...
role_obj = Role.objects.get(name=vlan_name)
```

如果找到了与 VLAN 名称匹配的角色，则将该角色分配给 VLAN；否则，不分配角色而直接创建。这样确保了在创建 VLAN 时能够添加我们之前创建的角色。
```python
def create_vlans(logger):
    """创建 VLAN_INFO 中定义的预设 VLAN，并分配相应角色。"""

    for vlan_name, vlan_id in VLAN_INFO.items():
        # 根据 VLAN 名称获取对应角色
        try:
            role_obj = Role.objects.get(name=vlan_name)
        except Role.DoesNotExist:
            logger.error(f"Role '{vlan_name}' not found. VLAN will be created without a role.")
            role_obj = None

        defaults = {"name": vlan_name, "status": ACTIVE_STATUS}
        if role_obj:
            defaults["role"] = role_obj

        vlan_obj, created = VLAN.objects.get_or_create(
            vid=vlan_id,
            defaults=defaults,
        )
        if created:
            vlan_obj.validated_save()
            logger.info(f"Successfully created VLAN '{vlan_name}' with ID {vlan_id}.")
        else:
            logger.info(f"VLAN '{vlan_name}' with ID {vlan_id} already exists.")
```

最后，我们将创建自定义字段，用于为接口分配角色。这一步对于后续几天创建接口间的连接和布线至关重要。注意，为使自定义字段方法正常工作，需要额外添加两个导入语句。
```python
from nautobot.dcim.models.device_components import Interface
from nautobot.extras.models.customfields import CustomField

...
def create_custom_fields(logger):
    """创建 CUSTOM_FIELDS 中定义的所有关联关系。"""
    for cf_name, field in CUSTOM_FIELDS.items():
        try:
            cf = CustomField.objects.get(key=cf_name)
        except CustomField.DoesNotExist:
            cf = CustomField.objects.create(key=cf_name)
            if "label" in field:
                cf.label = field.get("label")
            cf.validated_save()
            logger.info(f"Created custom field '{cf_name}'")
        for model in field["models"]:
            ct = ContentType.objects.get_for_model(model)
            cf.content_types.add(ct)
            cf.validated_save()
            logger.info(f"Added content type {ct} to custom field '{cf_name}'")
```

## 最终代码
```python
"""用于创建 POP 类型新站点的 Job。"""

from django.contrib.contenttypes.models import ContentType

from nautobot.apps.jobs import Job, register_jobs
from nautobot.extras.models.roles import Role
from nautobot.ipam.models import Prefix, VLAN
from nautobot.tenancy.models import Tenant
from nautobot.extras.models import Status
from nautobot.dcim.models.device_components import Interface
from nautobot.extras.models.customfields import CustomField

name = "Data Population Jobs Collection"


PREFIX_ROLES = ["p2p", "loopback", "server", "mgmt", "pop"]
POP_PREFIX_SIZE = 16
TENANT_NAME = "Data Center"
ACTIVE_STATUS = Status.objects.get(name="Active")
# VLAN 定义：键名同时用于查找对应角色
VLAN_INFO = {
    "server": 1000,
    "mgmt": 99,
}
CUSTOM_FIELDS = {
    "role": {"models": [Interface], "label": "Role"},
}
# 获取 Prefix 和 VLAN 模型的内容类型
prefix_ct = ContentType.objects.get_for_model(Prefix)
vlan_ct = ContentType.objects.get_for_model(VLAN)

def create_prefix_roles(logger):
    """创建 PREFIX_ROLES 中定义的所有前缀角色，并为其添加 IPAM Prefix 和 VLAN 的内容类型。"""    

    for role in PREFIX_ROLES:
        role_obj, created = Role.objects.get_or_create(name=role)
        # 为角色添加 Prefix 和 VLAN 内容类型
        role_obj.content_types.add(prefix_ct, vlan_ct)
        role_obj.validated_save()
        logger.info(f"Successfully created role {role} with content types for Prefix and VLAN.")


def create_tenant(logger):
    """使用 TENANT_NAME 定义的名称创建租户。"""
    tenant_obj, _ = Tenant.objects.get_or_create(name=TENANT_NAME)
    tenant_obj.validated_save()
    logger.info(f"Successfully created Tenant {TENANT_NAME}.")


def create_vlans(logger):
    """创建 VLAN_INFO 中定义的预设 VLAN，并分配相应角色。"""

    for vlan_name, vlan_id in VLAN_INFO.items():
        # 根据 VLAN 名称获取对应角色
        try:
            role_obj = Role.objects.get(name=vlan_name)
        except Role.DoesNotExist:
            logger.error(f"Role '{vlan_name}' not found. VLAN will be created without a role.")
            role_obj = None

        defaults = {"name": vlan_name, "status": ACTIVE_STATUS}
        if role_obj:
            defaults["role"] = role_obj

        vlan_obj, created = VLAN.objects.get_or_create(
            vid=vlan_id,
            defaults=defaults,
        )
        if created:
            vlan_obj.validated_save()
            logger.info(f"Successfully created VLAN '{vlan_name}' with ID {vlan_id}.")
        else:
            logger.info(f"VLAN '{vlan_name}' with ID {vlan_id} already exists.")

def create_custom_fields(logger):
    """创建 CUSTOM_FIELDS 中定义的所有关联关系。"""
    for cf_name, field in CUSTOM_FIELDS.items():
        try:
            cf = CustomField.objects.get(key=cf_name)
        except CustomField.DoesNotExist:
            cf = CustomField.objects.create(key=cf_name)
            if "label" in field:
                cf.label = field.get("label")
            cf.validated_save()
            logger.info(f"Created custom field '{cf_name}'")
        for model in field["models"]:
            ct = ContentType.objects.get_for_model(model)
            cf.content_types.add(ct)
            cf.validated_save()
            logger.info(f"Added content type {ct} to custom field '{cf_name}'")

class CreatePop(Job):
    """用于创建 POP 类型新站点的 Job。"""

    class Meta:
        """CreatePop 的元数据。"""

        name = "Create a Point of Presence"
        description = """
        Create a new Site of Type POP.
        A new /16 will automatically be allocated from the 'POP Global Pool' Prefix.
        """

    def run(self):
        """创建站点的主函数。"""
        # ----------------------------------------------------------------------------
        # 使用所有必需对象初始化数据库。
        # 我们将在接下来的几天中逐步扩展。
        # ----------------------------------------------------------------------------
        create_prefix_roles(self.logger)
        create_tenant(self.logger)
        create_vlans(self.logger)
        create_custom_fields(self.logger)


register_jobs(CreatePop)
```

> [!TIP]
> 别忘了运行 `invoke post-upgrade` 来注册 Job。

启用 Job 后，可以运行以查看结果：

![day_34_job_result](images/day_34_job_result.png)

明天我们将继续完善前置条件，为 Job 添加设备类型（DeviceType）配置。

## 第 34 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。由于这是一个为期 6 天的系列，如果您删除了实例，则需要重新完成前几天的代码。建议只停止实例，明天再重新启动。

欢迎在社交媒体上发布新建 Job 成功执行的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将通过添加设备类型创建功能来增强站点创建 Job。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+34+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 34 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
