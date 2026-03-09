# 设计未来站点（第二部分）

今天是为期 6 天的站点设计 Job 系列的第二天。

## 设计未来站点第二部分代码

如果您需要重新创建 Codespace 实例，请确保重新创建前一天挑战中的文件。
```shell
$ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash
root@c9e0fa2a45a0:/opt/nautobot# cd jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# pwd
/opt/nautobot/jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# touch create_site_job.py
root@c9e0fa2a45a0:/opt/nautobot/jobs# chown nautobot:nautobot create_site_job.py
```

以下是昨天完成的 Job 代码：
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

## 操作步骤

我们将通过添加必要的导入语句开始对现有代码的修改：
```python
from itertools import product
import re
import yaml
from nautobot.dcim.models import DeviceType, Manufacturer
from nautobot.dcim.models.device_component_templates import InterfaceTemplate
```

如昨天所述，今天的重点是设备类型（DeviceType）。以下是我们将使用的数据，定义为常量 `DEVICE_TYPES_YAML`。

这些设备类型的 YAML 内容来自社区维护的 [Nautobot 设备类型库](https://github.com/nautobot/devicetype-library)，其中包含大量设备类型，建议前往查看所有可用的设备类型。
```python
DEVICE_TYPES_YAML = [
    """
    manufacturer: Arista
    model: DCS-7280CR2-60
    part_number: DCS-7280CR2-60
    u_height: 1
    is_full_depth: false
    comments: '[Arista 7280R Data Sheet](https://www.arista.com/assets/data/pdf/Datasheets/7280R-DataSheet.pdf)'
    interfaces:
        - pattern: "Ethernet[1-60]/[1-4]"
          type: 100gbase-x-qsfp28
        - pattern: "Management1"
          type: 1000base-t
          mgmt_only: true
    """,
    """
    manufacturer: Arista
    model: DCS-7150S-24
    part_number: DCS-7150S-24
    u_height: 1
    is_full_depth: false
    comments: '[Arista 7150 Data Sheet](https://www.arista.com/assets/data/pdf/Datasheets/7150S_Datasheet.pdf)'
    interfaces:
        - pattern: "Ethernet[1-24]"
          type: 10gbase-x-sfpp
        - pattern: "Management1"
          type: 1000base-t
          mgmt_only: true
    """,
]
```

> [!NOTE]
> 我们为设备类型添加了接口，这里使用了来自 `device_component_templates` 的 `InterfaceTemplate` 模型，同时需要一个名为 `expand_interface_pattern` 的辅助函数，用于处理 `- pattern: "Ethernet[1-60]/[1-4]"` 这样的简短接口命名规范。

以下方法展示了如何使用正则表达式匹配接口模式，从而通过模式匹配批量生成大量接口，而无需在设备类型 YAML 中逐一指定：
```python
def expand_interface_pattern(pattern):
    """
    将接口模式（如 'Ethernet[1-60]/[1-4]'）展开为实际接口名称列表。
    
    支持以下格式：
      - 单一范围：Ethernet[1-24] -> Ethernet1, Ethernet2, ..., Ethernet24
      - 嵌套范围：Ethernet[1-60]/[1-4] -> Ethernet1/1, Ethernet1/2, ..., Ethernet60/4
    """
    match = re.findall(r"\[([0-9]+)-([0-9]+)\]", pattern)
    if not match:
        return [pattern]  # 无需展开，直接返回

    # 转换为数字列表
    try:
        ranges = [list(range(int(start), int(end) + 1)) for start, end in match]
    except ValueError:
        raise ValueError(f"Invalid range in pattern: {pattern}")

    # 生成含占位符的基础名称
    base_name = re.sub(r"\[[0-9]+-[0-9]+\]", "{}", pattern, count=len(ranges))

    # 使用笛卡尔积展开
    return [base_name.format(*nums) for nums in product(*ranges)]
```

我们新增的另一个方法是 `create_device_types`，它也会调用 `expand_interface_pattern` 方法：
```python
def create_device_types(logger):
    """
    从 YAML 定义创建 DeviceType 对象，并使用 InterfaceTemplate 添加接口。
    """

    for device_yaml in DEVICE_TYPES_YAML:
        data = yaml.safe_load(device_yaml)

        manufacturer_name = data.pop("manufacturer", None)
        if not manufacturer_name:
            logger.error("Manufacturer not provided in YAML definition.")
            continue
        manufacturer_obj, _ = Manufacturer.objects.get_or_create(name=manufacturer_name)

        model_name = data.pop("model", None)
        if not model_name:
            logger.error("Model not provided in YAML for manufacturer %s", manufacturer_name)
            continue

        # 创建 DeviceType
        device_type_defaults = {
            k: data[k] for k in ["part_number", "u_height", "is_full_depth", "comments"] if k in data
        }
        device_type_obj, created = DeviceType.objects.get_or_create(
            manufacturer=manufacturer_obj,
            model=model_name,
            defaults=device_type_defaults,
        )

        if created:
            device_type_obj.validated_save()
            logger.info(f"DeviceType created: {device_type_obj}")
        else:
            logger.info(f"DeviceType already exists: {device_type_obj}")

        # 使用 InterfaceTemplate 添加接口
        for iface in data.get("interfaces", []):
            pattern = iface.get("pattern")
            iface_type = iface.get("type")
            mgmt_only = iface.get("mgmt_only", False)

            if not pattern or not iface_type:
                logger.error(f"Invalid interface definition in {model_name}: {iface}")
                continue

            # 从范围模式生成接口名称
            interface_names = expand_interface_pattern(pattern)
            for iface_name in interface_names:
                interface_template, created = InterfaceTemplate.objects.get_or_create(
                    device_type=device_type_obj,
                    name=iface_name,
                    defaults={
                        "type": iface_type,
                        "mgmt_only": mgmt_only,
                    },
                )
                if created:
                    logger.info(f"Added interface {iface_name} ({iface_type}) to {model_name}")
```

该方法负责创建设备类型，并通过 Nautobot 的 `InterfaceTemplate` 类添加接口，确保从该设备类型创建的任何设备都会带有我们为该设备类型指定的接口。

方法的第一部分读取 `DEVICE_TYPES_YAML` 中指定的 YAML 数据，然后检查厂商（Manufacturer）是否已存在于 Nautobot 中，不存在则创建。厂商是设备类型的必填字段，必须在数据库中存在才能创建设备类型。
```python
    for device_yaml in DEVICE_TYPES_YAML:
        data = yaml.safe_load(device_yaml)

        manufacturer_name = data.pop("manufacturer", None)
        if not manufacturer_name:
            logger.error("Manufacturer not provided in YAML definition.")
            continue
        manufacturer_obj, _ = Manufacturer.objects.get_or_create(name=manufacturer_name)
```

接下来，在尝试创建新设备类型之前，将 YAML 文件中除厂商和型号以外的所有数据存入 defaults。如果设备类型已存在，则跳过并继续。
```python
        model_name = data.pop("model", None)
        if not model_name:
            logger.error("Model not provided in YAML for manufacturer %s", manufacturer_name)
            continue

        # 创建 DeviceType
        device_type_defaults = {
            k: data[k] for k in ["part_number", "u_height", "is_full_depth", "comments"] if k in data
        }
        device_type_obj, created = DeviceType.objects.get_or_create(
            manufacturer=manufacturer_obj,
            model=model_name,
            defaults=device_type_defaults,
        )

        if created:
            device_type_obj.validated_save()
            logger.info(f"DeviceType created: {device_type_obj}")
        else:
            logger.info(f"DeviceType already exists: {device_type_obj}")
```

最后，直接创建接口，或者如果使用了上述模式，则调用 `expand_interface_pattern` 方法批量生成：
```python
        for iface in data.get("interfaces", []):
            pattern = iface.get("pattern")
            iface_type = iface.get("type")
            mgmt_only = iface.get("mgmt_only", False)

            if not pattern or not iface_type:
                logger.error(f"Invalid interface definition in {model_name}: {iface}")
                continue

            # 从范围模式生成接口名称
            interface_names = expand_interface_pattern(pattern)
            for iface_name in interface_names:
                interface_template, created = InterfaceTemplate.objects.get_or_create(
                    device_type=device_type_obj,
                    name=iface_name,
                    defaults={
                        "type": iface_type,
                        "mgmt_only": mgmt_only,
                    },
                )
                if created:
                    logger.info(f"Added interface {iface_name} ({iface_type}) to {model_name}")
```

以下是前两天完成的最终代码，我们还没有开始站点部分的工作。

## 最终代码
```python
"""用于创建 POP 类型新站点的 Job。"""

from django.contrib.contenttypes.models import ContentType

from nautobot.apps.jobs import Job, register_jobs
from nautobot.dcim.models import DeviceType, Manufacturer
from nautobot.dcim.models.device_component_templates import InterfaceTemplate
from nautobot.extras.models import Status
from nautobot.extras.models.roles import Role
from nautobot.ipam.models import Prefix, VLAN
from nautobot.tenancy.models import Tenant
from nautobot.dcim.models.device_components import Interface
from nautobot.extras.models.customfields import CustomField

####第35天####
from itertools import product
import re
import yaml
from nautobot.dcim.models import DeviceType, Manufacturer
from nautobot.dcim.models.device_component_templates import InterfaceTemplate

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

####第35天####
DEVICE_TYPES_YAML = [
    """
    manufacturer: Arista
    model: DCS-7280CR2-60
    part_number: DCS-7280CR2-60
    u_height: 1
    is_full_depth: false
    comments: '[Arista 7280R Data Sheet](https://www.arista.com/assets/data/pdf/Datasheets/7280R-DataSheet.pdf)'
    interfaces:
        - pattern: "Ethernet[1-60]/[1-4]"
          type: 100gbase-x-qsfp28
        - pattern: "Management1"
          type: 1000base-t
          mgmt_only: true
    """,
    """
    manufacturer: Arista
    model: DCS-7150S-24
    part_number: DCS-7150S-24
    u_height: 1
    is_full_depth: false
    comments: '[Arista 7150 Data Sheet](https://www.arista.com/assets/data/pdf/Datasheets/7150S_Datasheet.pdf)'
    interfaces:
        - pattern: "Ethernet[1-24]"
          type: 10gbase-x-sfpp
        - pattern: "Management1"
          type: 1000base-t
          mgmt_only: true
    """,
]


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

####第35天####
def create_device_types(logger):
    """
    从 YAML 定义创建 DeviceType 对象，并使用 InterfaceTemplate 添加接口。
    """

    for device_yaml in DEVICE_TYPES_YAML:
        data = yaml.safe_load(device_yaml)

        manufacturer_name = data.pop("manufacturer", None)
        if not manufacturer_name:
            logger.error("Manufacturer not provided in YAML definition.")
            continue
        manufacturer_obj, _ = Manufacturer.objects.get_or_create(name=manufacturer_name)

        model_name = data.pop("model", None)
        if not model_name:
            logger.error("Model not provided in YAML for manufacturer %s", manufacturer_name)
            continue

        # 创建 DeviceType
        device_type_defaults = {
            k: data[k] for k in ["part_number", "u_height", "is_full_depth", "comments"] if k in data
        }
        device_type_obj, created = DeviceType.objects.get_or_create(
            manufacturer=manufacturer_obj,
            model=model_name,
            defaults=device_type_defaults,
        )

        if created:
            device_type_obj.validated_save()
            logger.info(f"DeviceType created: {device_type_obj}")
        else:
            logger.info(f"DeviceType already exists: {device_type_obj}")

        # 使用 InterfaceTemplate 添加接口
        for iface in data.get("interfaces", []):
            pattern = iface.get("pattern")
            iface_type = iface.get("type")
            mgmt_only = iface.get("mgmt_only", False)

            if not pattern or not iface_type:
                logger.error(f"Invalid interface definition in {model_name}: {iface}")
                continue

            # 从范围模式生成接口名称
            interface_names = expand_interface_pattern(pattern)
            for iface_name in interface_names:
                interface_template, created = InterfaceTemplate.objects.get_or_create(
                    device_type=device_type_obj,
                    name=iface_name,
                    defaults={
                        "type": iface_type,
                        "mgmt_only": mgmt_only,
                    },
                )
                if created:
                    logger.info(f"Added interface {iface_name} ({iface_type}) to {model_name}")

def expand_interface_pattern(pattern):
    """
    将接口模式（如 'Ethernet[1-60]/[1-4]'）展开为实际接口名称列表。
    
    支持以下格式：
      - 单一范围：Ethernet[1-24] -> Ethernet1, Ethernet2, ..., Ethernet24
      - 嵌套范围：Ethernet[1-60]/[1-4] -> Ethernet1/1, Ethernet1/2, ..., Ethernet60/4
    """
    match = re.findall(r"\[([0-9]+)-([0-9]+)\]", pattern)
    if not match:
        return [pattern]  # 无需展开，直接返回

    # 转换为数字列表
    try:
        ranges = [list(range(int(start), int(end) + 1)) for start, end in match]
    except ValueError:
        raise ValueError(f"Invalid range in pattern: {pattern}")

    # 生成含占位符的基础名称
    base_name = re.sub(r"\[[0-9]+-[0-9]+\]", "{}", pattern, count=len(ranges))

    # 使用笛卡尔积展开
    return [base_name.format(*nums) for nums in product(*ranges)]

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
        ####第35天####
        create_device_types(self.logger)


register_jobs(CreatePop)
```

以下是更新后代码的 Job 执行结果：

![day_35_job_result](images/day_35_job_result.png)

## 第 35 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布 Job 成功执行的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将通过添加站点和前缀创建功能来进一步增强站点创建 Job。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+35+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 35 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
