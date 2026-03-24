# 设计未来站点（第五部分）

好的，第五部分开始！

## 从第 37 天停下的地方继续

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

🔥 胜利就在眼前！冲吧！

## 设计未来站点第五部分代码

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

与前几天一样，我们从添加必要的导入语句开始。今天没有新的变量需要声明，但我们会使用现有的 `DEVICE_ROLES` 嵌套字典来确定创建设备时所需的属性，值得再次回顾一下。

> [!TIP]
> 还记得第 36 天的网络驱动映射步骤吗？这里 `"platform"` 非常关键。如果 Job 失败，请检查映射中是否设置了 `"arista_eos"`。
```python
from nautobot.dcim.models.device_components import Interface
from nautobot.dcim.choices import RackTypeChoices, InterfaceTypeChoices
from nautobot.ipam.models import Prefix, VLAN, IPAddress
...
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

今天的任务将把前几天的元素整合在一起，并引入新的内容。让我们逐节拆解。

首先，我们遍历已创建的机架，然后迭代 `DEVICE_ROLES` 中定义的数据，目标是在每个机架中按顺序系统性地创建 `"edge"` 和 `"leaf"` 设备。

还记得昨天创建机架时的全局计数器吗？它用于跟踪机架和设备的编号。此外，我们递增 `"position"` 以确保设备安装在唯一的插槽中。
```python
        # ----------------------------------------------------------------------------
        # 创建设备
        # ----------------------------------------------------------------------------
        self.devices = {}
        for rack in racks:  
            for role, data in DEVICE_ROLES.items():
                device_role, _ = Role.objects.get_or_create(
                    name=role,
                    color=data.get("color")
                )
                device_role.content_types.add(prefix_ct, vlan_ct)
                device_role.validated_save()
                self.logger.info(f"Created '{device_role}'")

                # 本机架中第一台设备的起始位置
                position = data.get("rack_elevation", 1)
                num_devices = data.get("per_rack") 

                for _ in range(num_devices):                
                    device_type = DeviceType.objects.get(model=data.get("device_type"))
                    device_name = f"{site_code}-{role}-{global_device_counter[role]:02}"
                    platform_obj = Platform.objects.get(network_driver=data.get("platform"))

                    device_obj, _ = Device.objects.get_or_create(
                        device_type=device_type, 
                        name=device_name,
                        location=self.site,
                        status=ACTIVE_STATUS,
                        role=device_role,
                        rack=rack,
                        platform=platform_obj,
                        position=position,
                        face="front",
                        tenant=tenant,
                    )
                    device_obj.save()
                    self.logger.info(f"Device {device_name} successfully created in {rack.name}")

                    # 将设备保存到清单中，供后续布线使用
                    self.devices[device_obj.name] = device_obj

                    position += 1
                    global_device_counter[role] += 1
```

接下来，我们在 `for` 循环中为每台创建的设备添加以下属性：

1. 从"loopback"前缀中找到第一个可用 IP 地址。
2. 基于该 IP 地址创建一个 IP 地址对象。
3. 在当前设备上创建 Loopback0 接口。
4. 将 IP 地址分配给创建的 Loopback 接口。
5. 将 Loopback0 的 IP 设置为设备的主 IPv4 地址，用于管理。
```python
                    # 分配 Loopback IP
                    loopback_prefix = Prefix.objects.get(
                        location=self.site,
                        role=loopback_role,
                    )

                    loopback_available_ip = loopback_prefix.get_first_available_ip()
                    
                    if not loopback_available_ip:
                        self.logger.error(f"No available IPs in prefix {loopback_prefix}")
                        return

                    loopback_ip, _ = IPAddress.objects.get_or_create(
                        address=str(loopback_available_ip),
                        status=ACTIVE_STATUS,
                        tenant=tenant,
                        dns_name=f"{role}-{global_device_counter[role]:02}.{site_code}.{tenant.description}"
                    )

                    loopback_ip.mask_length = 32  # L0 子网以 /32 分配，而非 /18
                    loopback_ip.save()

                    loopback_intf, _ = Interface.objects.get_or_create(
                        name="Loopback0", 
                        type=InterfaceTypeChoices.TYPE_VIRTUAL, 
                        device=device_obj, 
                        status=ACTIVE_STATUS,
                    )

                    loopback_intf.ip_addresses.add(loopback_ip)
                    loopback_intf.save()
                    
                    # 将 L0 IP 设置为设备的主 IPv4
                    device_obj.primary_ip4 = loopback_ip
                    device_obj.save()
                    self.logger.info(f"Created '{loopback_intf}' with '{loopback_ip}' and assigned to {device_name} as primary IP")

                    # 为接口分配角色
                    intfs = iter(Interface.objects.filter(device=device_obj))
                    for int_role, cnt in data.get("interfaces", []):
                        for _ in range(cnt):
                            intf = next(intfs, None)
                            if intf:
                                intf._custom_field_data = {"role": int_role}
                                intf.save()
```

在生产环境中，VLAN 通常存在于交换机（leaf）上。如果分配的角色为 `"leaf"`，以下部分将为设备创建对应属性：

1. 查询分配给 VLAN 角色的前缀（应为 /18）。
2. 从 /18 角色前缀中创建更小的 /24 子网，并设置类型为"pool"。
3. 从指定角色（"server"或"mgmt"）中找到第一个可用 IP 地址。
4. 在交换机上创建 VLAN 接口并分配 IP 地址。
```python
                    # 为 leaf 设备分配 VLAN
                    if role == "leaf":
                        for vlan_name, vlan_id in VLAN_INFO.items():                            
                            vlan_role = Role.objects.get(name=vlan_name)

                            vlan_block = Prefix.objects.filter(
                                location=self.site, 
                                status=ACTIVE_STATUS, 
                                role=vlan_role,
                            ).first()
                        
                            # 找到当前 VLAN 角色（server 或 mgmt）的下一个可用网络
                            first_avail = vlan_block.get_first_available_prefix()
                            subnet = list(first_avail.subnet(24))[0]
                            vlan_prefix, created = Prefix.objects.get_or_create(
                                prefix=str(subnet),
                                type="pool",
                                status=ACTIVE_STATUS,
                                role=vlan_role,
                                location=self.site,
                                tenant=tenant,
                                vlan=VLAN.objects.get(name=vlan_role)                                
                            )
                            
                            # 在 VLAN 接口上创建 IP 地址
                            vlan_ip, created = IPAddress.objects.get_or_create(
                                address=str(subnet[0]),
                                status=ACTIVE_STATUS,
                                tenant=tenant,
                                dns_name=f"ip-{str(subnet[0]).replace('.', '-')}.{vlan_name}.{site_code}.{tenant.description}",
                            )

                            intf_name = f"vlan{vlan_id}"
                            intf, created = Interface.objects.get_or_create(
                                name=intf_name, 
                                type=InterfaceTypeChoices.TYPE_VIRTUAL, 
                                device=device_obj,                                
                                status=ACTIVE_STATUS
                            )
                            intf.ip_addresses.add(vlan_ip)
                            intf.save()
```

最后，在 `for` 循环结束时，确保每台设备都关联了正确的 VLAN 和机架分配。
```python
                            # -----------------------------------------------------------------------
                            # 将 mgmt 和 server VLAN 与设备关联
                            # -----------------------------------------------------------------------
                            RelationshipAssociation.objects.get_or_create(
                                relationship=rel_device_vlan,
                                source_type=ContentType.objects.get_for_model(Device),
                                source_id=device_obj.id,
                                destination_type=ContentType.objects.get_for_model(VLAN),
                                destination_id=mgmt_vlan.id
                            )
                            RelationshipAssociation.objects.get_or_create(
                                relationship=rel_device_vlan,
                                source_type=ContentType.objects.get_for_model(Device),
                                source_id=device_obj.id,
                                destination_type=ContentType.objects.get_for_model(VLAN),
                                destination_id=server_vlan.id
                            )
```

## 第 38 天最终代码
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

####第38天####
from nautobot.dcim.models.device_components import Interface
from nautobot.dcim.choices import RackTypeChoices, InterfaceTypeChoices
from nautobot.ipam.models import Prefix, VLAN, IPAddress
from nautobot.dcim.models.devices import Device, DeviceType, Platform, Manufacturer


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

        # ----------------------------------------------------------------------------
        # 创建设备
        # ----------------------------------------------------------------------------
        self.devices = {}
        for rack in racks:  
            for role, data in DEVICE_ROLES.items():
                device_role, _ = Role.objects.get_or_create(
                    name=role,
                    color=data.get("color")
                )
                device_role.content_types.add(prefix_ct, vlan_ct)
                device_role.validated_save()
                self.logger.info(f"Created '{device_role}'")

                # 本机架中第一台设备的起始位置
                position = data.get("rack_elevation", 1)
                num_devices = data.get("per_rack") 

                for _ in range(num_devices):                
                    device_type = DeviceType.objects.get(model=data.get("device_type"))
                    device_name = f"{site_code}-{role}-{global_device_counter[role]:02}"
                    platform_obj = Platform.objects.get(network_driver=data.get("platform"))

                    device_obj, _ = Device.objects.get_or_create(
                        device_type=device_type, 
                        name=device_name,
                        location=self.site,
                        status=ACTIVE_STATUS,
                        role=device_role,
                        rack=rack,
                        platform=platform_obj,
                        position=position,
                        face="front",
                        tenant=tenant,
                    )
                    device_obj.save()
                    self.logger.info(f"Device {device_name} successfully created in {rack.name}")

                    # 将设备保存到清单中，供后续布线使用
                    self.devices[device_obj.name] = device_obj

                    position += 1
                    global_device_counter[role] += 1
        
                    # 分配 Loopback IP
                    loopback_prefix = Prefix.objects.get(
                        location=self.site,
                        role=loopback_role,
                    )

                    loopback_available_ip = loopback_prefix.get_first_available_ip()
                    
                    if not loopback_available_ip:
                        self.logger.error(f"No available IPs in prefix {loopback_prefix}")
                        return

                    loopback_ip, _ = IPAddress.objects.get_or_create(
                        address=str(loopback_available_ip),
                        status=ACTIVE_STATUS,
                        tenant=tenant,
                        dns_name=f"{role}-{global_device_counter[role]:02}.{site_code}.{tenant.description}"
                    )

                    loopback_ip.mask_length = 32  # L0 子网以 /32 分配，而非 /18
                    loopback_ip.save()

                    loopback_intf, _ = Interface.objects.get_or_create(
                        name="Loopback0", 
                        type=InterfaceTypeChoices.TYPE_VIRTUAL, 
                        device=device_obj, 
                        status=ACTIVE_STATUS,
                    )

                    loopback_intf.ip_addresses.add(loopback_ip)
                    loopback_intf.save()
                    
                    # 将 L0 IP 设置为设备的主 IPv4
                    device_obj.primary_ip4 = loopback_ip
                    device_obj.save()
                    self.logger.info(f"Created '{loopback_intf}' with '{loopback_ip}' and assigned to {device_name} as primary IP")

                    # 为接口分配角色
                    intfs = iter(Interface.objects.filter(device=device_obj))
                    for int_role, cnt in data.get("interfaces", []):
                        for _ in range(cnt):
                            intf = next(intfs, None)
                            if intf:
                                intf._custom_field_data = {"role": int_role}
                                intf.save()
                    
                    # 为 leaf 设备分配 VLAN
                    if role == "leaf":
                        for vlan_name, vlan_id in VLAN_INFO.items():                            
                            vlan_role = Role.objects.get(name=vlan_name)

                            vlan_block = Prefix.objects.filter(
                                location=self.site, 
                                status=ACTIVE_STATUS, 
                                role=vlan_role,
                            ).first()
                        
                            # 找到当前 VLAN 角色（server 或 mgmt）的下一个可用网络
                            first_avail = vlan_block.get_first_available_prefix()
                            subnet = list(first_avail.subnet(24))[0]
                            vlan_prefix, created = Prefix.objects.get_or_create(
                                prefix=str(subnet),
                                type="pool",
                                status=ACTIVE_STATUS,
                                role=vlan_role,
                                location=self.site,
                                tenant=tenant,
                                vlan=VLAN.objects.get(name=vlan_role)                                
                            )
                            
                            # 在 VLAN 接口上创建 IP 地址
                            vlan_ip, created = IPAddress.objects.get_or_create(
                                address=str(subnet[0]),
                                status=ACTIVE_STATUS,
                                tenant=tenant,
                                dns_name=f"ip-{str(subnet[0]).replace('.', '-')}.{vlan_name}.{site_code}.{tenant.description}",
                            )

                            intf_name = f"vlan{vlan_id}"
                            intf, created = Interface.objects.get_or_create(
                                name=intf_name, 
                                type=InterfaceTypeChoices.TYPE_VIRTUAL, 
                                device=device_obj,                                
                                status=ACTIVE_STATUS
                            )
                            intf.ip_addresses.add(vlan_ip)
                            intf.save()

                            # -----------------------------------------------------------------------
                            # 将 mgmt 和 server VLAN 与设备关联
                            # -----------------------------------------------------------------------
                            RelationshipAssociation.objects.get_or_create(
                                relationship=rel_device_vlan,
                                source_type=ContentType.objects.get_for_model(Device),
                                source_id=device_obj.id,
                                destination_type=ContentType.objects.get_for_model(VLAN),
                                destination_id=mgmt_vlan.id
                            )
                            RelationshipAssociation.objects.get_or_create(
                                relationship=rel_device_vlan,
                                source_type=ContentType.objects.get_for_model(Device),
                                source_id=device_obj.id,
                                destination_type=ContentType.objects.get_for_model(VLAN),
                                destination_id=server_vlan.id
                            )

register_jobs(CreatePop)
```

🚀 现在可以创建设备了！🚀

![Create Site Day 38](images/create_site_day38_1.png)

Job 完成后，让我们验证各项结果，确认执行无误！

前往"LOCATIONS -> Racks"验证机架和设备是否按预期创建。预期结果是 `{site_code}-101` 机架中应包含 edge-01 和 leaf-01 至 03 设备。

![Created Site Racks](images/create_site_day38_2.png)

`{site_code}-102` 机架中应包含 edge-02 和 leaf-04 至 06 设备。

![Created Site Racks](images/create_site_day38_3.png)

前缀情况如何？

可以看到，前缀已正确创建为带有所有正确属性的 /24 "pool"子网。

![Created Prefixes](images/create_site_day38_4.png)

检查设备，确认其具有所有预期属性。

![List Devices](images/create_site_day38_5.png)

点击其中一台设备，验证 Loopback0 IP、主 IPv4、角色、平台等关键信息。

![Device Details](images/create_site_day38_6.png)

最后，确认每台设备的 VLAN 关联正确。在设备详情页向下滚动至 Relationships -> VLANs，应该能看到两个关联条目。

![Device to VLAN](images/create_site_day38_7.png)

## 回顾

看到这些绿色的勾选标记，成就感油然而生！今天的内容确实很多，但绝对值得！

✅ 第 36 天：

    ✅ 创建关联关系

    ✅ 创建站点

    ✅ 分配 /16 前缀


✅ 第 37 天：

    ✅ 为每个角色创建并分配前缀

    ✅ 创建机架

    ✅ 建立机架与 VLAN 的关联关系


✅ 第 38 天：

    ✅ 创建设备

    ✅ 为关键接口分配 VLAN 和 IP

    ✅ 建立设备与 VLAN 的关联关系


✅ 第 39 天：
- [ ] 将线路连接到边缘设备
- [ ] 设备间布线

## 第 38 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布新 Job 成功执行的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

第 39 天是在 Nautobot 中构建站点的最后一天！明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+38+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 38 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
