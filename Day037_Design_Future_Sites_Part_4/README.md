# 设计未来站点（第四部分）

我们已完成为期 6 天的站点设计 Job 系列的一半进度。

## 从第 36 天停下的地方继续

省去冗长的介绍——让我们直接投入任务！

✅ 第 36 天：

    ✅ 创建关联关系

    ✅ 创建站点

    ✅ 分配 /16 前缀


✅ 第 37 天：
- [ ] 为每个角色创建并分配前缀
- [ ] 创建机架
- [ ] 建立机架与 VLAN 的关联关系

✅ 第 38 天：
- [ ] 创建设备
- [ ] 为关键接口分配 VLAN 和 IP
- [ ] 建立设备与 VLAN 的关联关系

✅ 第 39 天：
- [ ] 将线路连接到边缘设备
- [ ] 设备间布线

## 设计未来站点第四部分代码

如果您需要重新创建 Codespace 实例，请确保重新创建前一天挑战中的文件。
```shell
$ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash
root@c9e0fa2a45a0:/opt/nautobot# cd jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# pwd
/opt/nautobot/jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# touch create_site_job.py
root@c9e0fa2a45a0:/opt/nautobot/jobs# chown nautobot:nautobot create_site_job.py
```

## 操作步骤

今天我们将为"创建站点 Job"添加机架创建功能，因此需要导入额外的模型，并声明一些新的变量。
```python
from nautobot.dcim.models import Device, DeviceType, Manufacturer
from nautobot.dcim.models.racks import Rack
from nautobot.dcim.choices import RackTypeChoices

...

ROLE_PREFIX_SIZE = 18

RACK_HEIGHT = 48
RACK_WIDTH = 19
RACK_TYPE = RackTypeChoices.TYPE_4POST

DEVICE_ROLES = {
    "edge": {
        "per_rack": 1,
        "device_type": "DCS-7280CR2-60",
        "platform": "arista_eos",
        "rack_elevation": 32,
        "color": "ff9800",
        "interfaces": [
            ("peer", 2),
            ("leaf", 12),
            ("external", 8),
        ],
    },
    "leaf": {
        "per_rack": 3,
        "device_type": "DCS-7150S-24",
        "platform": "arista_eos",
        "rack_elevation": 38,
        "color": "3f51b5",
        "interfaces": [
            ("edge", 4),
            ("access", 20),
        ],
    },
}
```

在深入代码之前，我们先来了解 Nautobot 中的前缀（Prefix），特别是其层次结构。如需深入了解，请查阅 [Nautobot 官方文档](https://docs.nautobot.com/projects/core/en/stable/user-guide/core-data-model/ipam/prefix/#prefix-utilization-calculation)。

在昨天的挑战中，我们为站点分配了一个 /16。在我们的实验环境中，/8 和 /16 被作为 `"Container"`（容器）类型处理，我们将在此基础上创建类型为 `"Network"` 的子网。之后，我们可以使用这些子网创建 `"Pool"`（地址池），从而为设备和接口分配单独的 IP 地址。

![Prefix Hierarchy](images/prefix_hierarchy.png)

## 代码结构

让我们拆解代码，了解其应用方式：

1. 使用分配好的 /16 前缀（`pop_prefix`），通过 Python 内置的 `IPv4Network` 类创建更小的 /18 子网。
2. 遍历生成的子网，将每个子网分配给预定义的角色。
3. 创建实际的前缀对象，并关联到第 34 天建立的角色。
4. 注意，本项目中我们只创建了两个 VLAN（`server` 和 `mgmt`），因此只有这两个前缀会被分配 VLAN。

> [!TIP]
> 与所有 Python 代码一样，缩进非常重要！每次添加新"模块"时，请务必参照主 `run()` 方法检查缩进，以防报错。
```python
        # ----------------------------------------------------------------------------
        # 在 POP 中创建前缀并分配角色
        # ----------------------------------------------------------------------------
        
        site_subnets = IPv4Network(str(pop_prefix)).subnets(new_prefix=ROLE_PREFIX_SIZE)
        server_subnet = next(site_subnets)
        mgmt_subnet = next(site_subnets)
        loopback_subnet = next(site_subnets)
        p2p_subnet = next(site_subnets)

        # 为各角色分配新子网
        server_role = Role.objects.get(name="server")
        server_prefix, created = Prefix.objects.get_or_create(
            prefix=str(server_subnet),
            type="network",
            role=server_role,
            parent=pop_prefix,
            status=ACTIVE_STATUS,
            location=self.site,
            tenant=tenant,
            vlan=VLAN.objects.get(name=server_role)
        )
        self.logger.info(f"'{server_prefix}' assigned to '{server_role}'.")

        mgmt_role = Role.objects.get(name="mgmt")
        mgmt_prefix, created = Prefix.objects.get_or_create(
            prefix=str(mgmt_subnet),
            type="network",
            role=mgmt_role,
            parent=pop_prefix,
            status=ACTIVE_STATUS,
            location=self.site,
            tenant=tenant,
            vlan=VLAN.objects.get(name=mgmt_role)
        )
        self.logger.info(f"'{mgmt_prefix}' assigned to '{mgmt_role}'.")

        loopback_role = Role.objects.get(name="loopback")
        loopback_prefix, created = Prefix.objects.get_or_create(
            prefix=str(loopback_subnet),
            type="network",
            role=loopback_role,
            parent=pop_prefix,
            status=ACTIVE_STATUS,
            location=self.site,
            tenant=tenant
        )
        self.logger.info(f"'{loopback_prefix}' assigned to '{loopback_role}'.")

        p2p_role = Role.objects.get(name="p2p")
        p2p_prefix, created = Prefix.objects.get_or_create(
            prefix=str(p2p_subnet),
            type="network",
            role=p2p_role,
            parent=pop_prefix,
            status=ACTIVE_STATUS,
            location=self.site,
            tenant=tenant
        )
        self.logger.info(f"'{p2p_prefix}' assigned to '{p2p_role}'.") 
```

> [!TIP]
> 此时可以运行 Job 查看结果。在添加更多组件时，逐步测试代码是良好的实践习惯，能帮助定位问题所在。注意避免重复对象，如重名、重复前缀等。

接下来创建机架，并将其与我们之前创建的 VLAN 关联。这部分根据全局变量中声明的值来操作，相对直接。

此步骤将为我们在第 38 天创建的设备提供安置位置。

> [!TIP]
> 请记住 `global_device_counter`，明天创建设备时会用到它。
```python
        # ----------------------------------------------------------------------------
        # 创建机架
        # ----------------------------------------------------------------------------
        # 初始化全局计数器
        global_device_counter = {role: 1 for role in DEVICE_ROLES}  # 用于跟踪编号
        racks = []  # 存储已创建的机架以便后续迭代

        # 创建机架
        num_rack = 2  # 如果站点需要超过 2 个机架，可将此改为输入变量
        for num in range(1, num_rack + 1):
            rack_name = f"{site_code.upper()}-{100 + num}"
            rack, created = Rack.objects.get_or_create(
                name=rack_name,
                location=self.site,
                u_height=RACK_HEIGHT,
                width=RACK_WIDTH,
                type=RACK_TYPE,
                status=ACTIVE_STATUS,
                tenant=tenant,
            )
            racks.append(rack)
            self.logger.info(f"Successfully created {rack_name}.")

        # ---------------------------------------------------------------------------
        # 将 mgmt 和 server VLAN 与每个机架关联
        # ---------------------------------------------------------------------------
        mgmt_vlan = VLAN.objects.get(name="mgmt")
        server_vlan = VLAN.objects.get(name="server")
        for rack in racks:
            RelationshipAssociation.objects.get_or_create(
                relationship=rel_rack_vlan,
                source_type=ContentType.objects.get_for_model(Rack),
                source_id=rack.id,
                destination_type=ContentType.objects.get_for_model(VLAN),
                destination_id=mgmt_vlan.id
            )
            RelationshipAssociation.objects.get_or_create(
                relationship=rel_rack_vlan,
                source_type=ContentType.objects.get_for_model(Rack),
                source_id=rack.id,
                destination_type=ContentType.objects.get_for_model(VLAN),
                destination_id=server_vlan.id
            )
```

## 第 37 天最终代码
```python
"""用于创建 POP 类型新站点的 Job。"""

from itertools import product
import re

from django.contrib.contenttypes.models import ContentType
import yaml

from nautobot.dcim.models.device_component_templates import InterfaceTemplate
from nautobot.extras.models import Status
from nautobot.extras.models.roles import Role
from nautobot.ipam.models import Prefix, VLAN
from nautobot.tenancy.models import Tenant
from nautobot.extras.models.customfields import CustomField
from nautobot.dcim.models.device_components import Interface

####第36天####
from nautobot.apps.jobs import Job, ObjectVar, StringVar, register_jobs
from nautobot.dcim.models.locations import Location, LocationType
from ipaddress import IPv4Network
from nautobot.extras.models.relationships import Relationship, RelationshipAssociation
from nautobot.extras.choices import RelationshipTypeChoices

####第37天####
from nautobot.dcim.models.racks import Rack
from nautobot.dcim.choices import RackTypeChoices
from nautobot.dcim.models import Device, DeviceType, Manufacturer


name = "Data Population Jobs Collection"


PREFIX_ROLES = ["p2p", "loopback", "server", "mgmt", "pop"]
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

####第36天####
POP_PREFIX_SIZE = 16

####第37天####
ROLE_PREFIX_SIZE = 18
RACK_HEIGHT = 48
RACK_WIDTH = 19
RACK_TYPE = RackTypeChoices.TYPE_4POST
DEVICE_ROLES = {
    "edge": {
        "per_rack": 1,
        "device_type": "DCS-7280CR2-60",
        "platform": "arista_eos",
        "rack_elevation": 32,
        "color": "ff9800",
        "interfaces": [
            ("peer", 2),
            ("leaf", 12),
            ("external", 8),
        ],
    },
    "leaf": {
        "per_rack": 3,
        "device_type": "DCS-7150S-24",
        "platform": "arista_eos",
        "rack_elevation": 38,
        "color": "3f51b5",
        "interfaces": [
            ("edge", 4),
            ("access", 20),
        ],
    },
}


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
    ranges = [list(range(int(start), int(end) + 1)) for start, end in match]

    # 使用笛卡尔积生成名称
    expanded_names = []
    base_name = re.sub(r"\[[0-9]+-[0-9]+\]", "{}", pattern)

    for numbers in product(*ranges):
        expanded_names.append(base_name.format(*numbers))

    return expanded_names

####第37天####
def get_or_create_relationship(label, key, source_model, destination_model, rel_type):
    try:
        rel, created = Relationship.objects.get_or_create(
            key=key,
            defaults={
                "label": label,
                "source_type": ContentType.objects.get_for_model(source_model),
                "destination_type": ContentType.objects.get_for_model(destination_model),
                "type": rel_type,
            }
        )
        return rel
    except Exception as e:
        self.logger.error(f"Error creating relationship {label}: {e}")
        return Relationship.objects.get(key=key)  # 回退到已存在的关联关系


class CreatePop(Job):
    """用于创建 POP 类型新站点的 Job。"""
    
    ####第36天####
    # 接收用户输入的站点信息
    location_type = ObjectVar(
        model=LocationType,
        description = "Select location type for new site."
    )
    parent_site = ObjectVar(
        model=Location,
        required=False,
        description="Select an existing site to nest this site under. Site will be created as a Region if left blank.",
        label="Parent Site"
    )
    site_name = StringVar(description="Name of the new site", label="Site Name")
    site_facility = StringVar(description="Facility of the new site", label="Site Facility")
    
    site_code = StringVar(description="Enter Site Code as 2-letter state and 2-digit site ID e.g. NY01 for New York Store ID 01")
    tenant = ObjectVar(model=Tenant)


    class Meta:
        """CreatePop 的元数据。"""

        name = "Create a Point of Presence"
        description = """
        Create a new POP Site.
        A new /16 will automatically be allocated from the 'POP Global Pool' Prefix.
        """    
    ####第36天####    
    def run(self, location_type, site_name, site_facility, tenant, site_code, parent_site=None):
        """创建站点的主函数。"""
        # ----------------------------------------------------------------------------
        # 使用所有必需对象初始化数据库
        # ----------------------------------------------------------------------------
        create_prefix_roles(self.logger)
        create_tenant(self.logger)
        create_vlans(self.logger)
        create_device_types(self.logger)

        ####第37天####
        # ----------------------------------------------------------------------------
        # 创建关联关系
        # ----------------------------------------------------------------------------
        rel_device_vlan = get_or_create_relationship(
            "Device to VLAN", "device_to_vlan", Device, VLAN, RelationshipTypeChoices.TYPE_MANY_TO_MANY
        )
        rel_rack_vlan = get_or_create_relationship(
            "Rack to VLAN", "rack_to_vlan", Rack, VLAN, RelationshipTypeChoices.TYPE_MANY_TO_MANY
        )

        ####第36天####
        # ----------------------------------------------------------------------------
        # 创建站点
        # ----------------------------------------------------------------------------
        location_type_site, _ = LocationType.objects.get_or_create(name=location_type)
        self.site_name = site_name
        self.site_facility = site_facility
        self.site, created = Location.objects.get_or_create(
            name=site_name,
            location_type=LocationType.objects.get(name=location_type),
            facility=site_facility,
            status=ACTIVE_STATUS,
            parent=parent_site,  # 如果未提供则为 None
            tenant=tenant
        )
        
        if created:
            message = f"Site '{site_name}' created as a top level Region."
            if parent_site:
                message = f"Site '{site_name}' successfully nested under '{parent_site.name}'."
            self.site.validated_save()
            self.logger.info(message)

            pop_role = Role.objects.get(name="pop")
            self.logger.info(f"Assigning '{site_name}' as '{pop_role}' role.")

            # ----------------------------------------------------------------------------
            # 为此 POP 分配前缀
            # ----------------------------------------------------------------------------
        
            # 查找第一个尚未分配给站点的可用 /16 前缀
            pop_prefix = Prefix.objects.filter(
                type="container",
                prefix_length=POP_PREFIX_SIZE,
                status=ACTIVE_STATUS,
                location__isnull=True
            ).first()

            if pop_prefix:
                pop_prefix.location = self.site
                pop_prefix.validated_save()
                self.logger.info(f"Assigned {pop_prefix} to {site_name}.")
            else:
                self.logger.warning("No available /16 prefixes found. Creating a new /16.")
                top_level_prefix = Prefix.objects.filter(
                    type="container",
                    status=ACTIVE_STATUS,
                    prefix_length=8
                ).first()

                # 获取 /8 内的第一个可用前缀
                first_avail = top_level_prefix.get_first_available_prefix()

                if not first_avail:
                    raise Exception("No available subnets found within the /8 prefix.")

                # 遍历 /8 内所有可能的 /16 子网，找到第一个未分配的
                for candidate_prefix in IPv4Network(str(first_avail)).subnets(new_prefix=POP_PREFIX_SIZE):
                    if not Prefix.objects.filter(prefix=str(candidate_prefix)).exists():
                        pop_prefix, created = Prefix.objects.get_or_create(
                            prefix=str(candidate_prefix),
                            type="container",
                            location=self.site,
                            status=ACTIVE_STATUS,
                            role=pop_role
                        )
                        pop_prefix.validated_save()
                        self.logger.info(f"Allocated new '{pop_prefix}' for site '{site_name}'.")
                        break
                else:
                    raise Exception("No available /16 prefixes found within the /8 range.")
        else:
            self.logger.warning(f"Site '{site_name}' already exists.") 

        ####第37天####
        # ----------------------------------------------------------------------------
        # 在 POP 中创建前缀并分配角色
        # ----------------------------------------------------------------------------
        
        site_subnets = IPv4Network(str(pop_prefix)).subnets(new_prefix=ROLE_PREFIX_SIZE)
        server_subnet = next(site_subnets)
        mgmt_subnet = next(site_subnets)
        loopback_subnet = next(site_subnets)
        p2p_subnet = next(site_subnets)

        # 为各角色分配新子网
        server_role = Role.objects.get(name="server")
        server_prefix, created = Prefix.objects.get_or_create(
            prefix=str(server_subnet),
            type="network",
            role=server_role,
            parent=pop_prefix,
            status=ACTIVE_STATUS,
            location=self.site,
            tenant=tenant,
            vlan=VLAN.objects.get(name=server_role)
        )
        self.logger.info(f"'{server_prefix}' assigned to '{server_role}'.")

        mgmt_role = Role.objects.get(name="mgmt")
        mgmt_prefix, created = Prefix.objects.get_or_create(
            prefix=str(mgmt_subnet),
            type="network",
            role=mgmt_role,
            parent=pop_prefix,
            status=ACTIVE_STATUS,
            location=self.site,
            tenant=tenant,
            vlan=VLAN.objects.get(name=mgmt_role)
        )
        self.logger.info(f"'{mgmt_prefix}' assigned to '{mgmt_role}'.")

        loopback_role = Role.objects.get(name="loopback")
        loopback_prefix, created = Prefix.objects.get_or_create(
            prefix=str(loopback_subnet),
            type="network",
            role=loopback_role,
            parent=pop_prefix,
            status=ACTIVE_STATUS,
            location=self.site,
            tenant=tenant
        )
        self.logger.info(f"'{loopback_prefix}' assigned to '{loopback_role}'.")

        p2p_role = Role.objects.get(name="p2p")
        p2p_prefix, created = Prefix.objects.get_or_create(
            prefix=str(p2p_subnet),
            type="network",
            role=p2p_role,
            parent=pop_prefix,
            status=ACTIVE_STATUS,
            location=self.site,
            tenant=tenant
        )
        self.logger.info(f"'{p2p_prefix}' assigned to '{p2p_role}'.") 

        # ----------------------------------------------------------------------------
        # 创建机架
        # ----------------------------------------------------------------------------
        # 初始化全局计数器
        global_device_counter = {role: 1 for role in DEVICE_ROLES}  # 用于跟踪编号
        racks = []  # 存储已创建的机架以便后续迭代

        # 创建机架
        num_rack = 2  # 如果站点需要超过 2 个机架，可将此改为输入变量
        for num in range(1, num_rack + 1):
            rack_name = f"{site_code.upper()}-{100 + num}"
            rack, created = Rack.objects.get_or_create(
                name=rack_name,
                location=self.site,
                u_height=RACK_HEIGHT,
                width=RACK_WIDTH,
                type=RACK_TYPE,
                status=ACTIVE_STATUS,
                tenant=tenant,
            )
            racks.append(rack)
            self.logger.info(f"Successfully created {rack_name}.")

        # ---------------------------------------------------------------------------
        # 将 mgmt 和 server VLAN 与每个机架关联
        # ---------------------------------------------------------------------------
        mgmt_vlan = VLAN.objects.get(name="mgmt")
        server_vlan = VLAN.objects.get(name="server")
        for rack in racks:
            RelationshipAssociation.objects.get_or_create(
                relationship=rel_rack_vlan,
                source_type=ContentType.objects.get_for_model(Rack),
                source_id=rack.id,
                destination_type=ContentType.objects.get_for_model(VLAN),
                destination_id=mgmt_vlan.id
            )
            RelationshipAssociation.objects.get_or_create(
                relationship=rel_rack_vlan,
                source_type=ContentType.objects.get_for_model(Rack),
                source_id=rack.id,
                destination_type=ContentType.objects.get_for_model(VLAN),
                destination_id=server_vlan.id
            )
            
register_jobs(CreatePop)
```

猜猜下一步是什么？！

🚀 猜对了——运行我们的 Job 吧！🚀

> [!NOTE]
> 记得使用不同的站点名称以避免冲突。

![Input Site Info](images/create_site_day37_1.png)

在左侧导航到"IPAM -> Prefixes"查看前缀输出。

与第 36 天的 Job 相比，我们的前缀应该有更多详细信息。注意 /18 子网、角色和 VLAN 的正确分配，以及前缀在父站点子网下的正确嵌套关系（见"Parent Prefix"部分）。

![Prefix Result](images/create_site_day37_2.png)

![Prefix Detaisl](images/create_site_day37_3.png)

在左侧导航到"LOCATIONS -> Racks"查看新创建的机架。

查看机架名称，确认其与您的站点代码匹配。这正是标准化的价值所在——让我们能够应用自动化！

![Created Racks](images/create_site_day37_4.png)

最后，验证 `rack-to-vlan` 关联。在机架详情页面向下滚动，点击"Relationship -> VLANs"，可以看到机架已成功与我们创建的 VLAN 关联。

![Rack to VLAN](images/create_site_day37_5.png)

## 回顾

又是收获满满的一天！让我们庆祝并更新清单！🎉

✅ 第 36 天：

    ✅ 创建关联关系

    ✅ 创建站点

    ✅ 分配 /16 前缀


✅ 第 37 天：

    ✅ 为每个角色创建并分配前缀

    ✅ 创建机架

    ✅ 建立机架与 VLAN 的关联关系


✅ 第 38 天：
- [ ] 创建设备
- [ ] 为关键接口分配 VLAN 和 IP
- [ ] 建立设备与 VLAN 的关联关系

✅ 第 39 天：
- [ ] 将线路连接到边缘设备
- [ ] 设备间布线

## 第 37 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布成功创建前缀和机架的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

第 38 天，我们将着手创建设备并分配 VLAN。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+37+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 37 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
