# 设计未来站点（第六部分）

哇，时间过得真快！我们现在来到了这个六部分系列的最终迭代。

## 设计未来站点第六部分代码

如果你需要创建一个新的 codespace 实例，请确保从上一个挑战中重新创建文件。
```shell
$ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash
root@c9e0fa2a45a0:/opt/nautobot# cd jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# pwd
/opt/nautobot/jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# touch create_site_job.py
root@c9e0fa2a45a0:/opt/nautobot/jobs# chown nautobot:nautobot create_site_job.py
```

我们终于到了最后一天。在这个练习的最后一天，我们将为站点创建线缆和电路，并将它们连接到设备上。在创建电路之前，我们需要先创建提供商（Provider）和电路类型（CircuitType）。这些本可以在前置条件部分完成，但那部分内容已经很多了，所以我们在这里添加。

在继续之前，让我们快速浏览一下今天的清单：

✅ 第36天：

    ✅ 创建关系

    ✅ 创建站点

    ✅ 分配 /16 前缀


✅ 第37天：

    ✅ 为每个角色创建并分配前缀

    ✅ 创建机架

    ✅ 建立机架与 VLAN 的关系


✅ 第38天：

    ✅ 创建设备

    ✅ 为关键接口分配 VLAN 和 IP

    ✅ 建立设备与 VLAN 的关系


✅ 第39天：
- [ ] 将电路连接到边缘设备
- [ ] 为设备之间连接线缆

## 操作说明

在这最后一天，我们将添加一些新的导入语句和一个新的常量。
```python
####DAY39####
from nautobot.dcim.models import Cable
from nautobot.circuits.models import Circuit, CircuitTermination, CircuitType, Provider

TRANSIT_PROVIDERS = ["Equinix", "Cologix", "CoreSite"]
```

这些导入与我们在本节中将要创建的对象相关，常量 `TRANSIT_PROVIDERS` 将允许我们向 Nautobot 中添加一些可用作电路提供商的提供商。

我们还需要更新所使用的 LocationType，使其包含 CircuitTermination 的 ContentType。我们将使用一个辅助方法来实现这一点，并在创建站点之前调用它。
```python
def update_location_type_for_circuit_termination(logger, location_type_obj):
    """
    确保给定的 LocationType 包含 CircuitTermination 的内容类型，
    允许该类型的位置与电路终端相关联。
    """
    from django.contrib.contenttypes.models import ContentType
    from nautobot.circuits.models import CircuitTermination

    ct = ContentType.objects.get_for_model(CircuitTermination)
    if ct not in location_type_obj.content_types.all():
        location_type_obj.content_types.add(ct)
        location_type_obj.validated_save()
        logger.info(f"LocationType '{location_type_obj}' 已更新，包含 CircuitTermination 内容类型。")
    else:
        logger.info(f"LocationType '{location_type_obj}' 已包含 CircuitTermination 内容类型。")
```

确保在"创建站点"部分下方的代码中添加对这个新方法的调用。
```python
        # ----------------------------------------------------------------------------
        # 创建站点
        # ----------------------------------------------------------------------------
        location_type_obj, created = LocationType.objects.get_or_create(name=location_type)
        update_location_type_for_circuit_termination(self.logger, location_type_obj)
```

完成这些步骤后，我们将开始创建线缆。
```python
        # ----------------------------------------------------------------------------
        # 线缆连接
        # ----------------------------------------------------------------------------
        # 连接边缘路由器
        edge_01 = self.devices.get(f"{site_code}-edge-01")
        edge_02 = self.devices.get(f"{site_code}-edge-02")

        # 获取每台边缘设备上具有 'peer' 自定义字段的接口
        peer_intfs_01 = iter(Interface.objects.filter(device=edge_01, _custom_field_data__role="peer"))
        peer_intfs_02 = iter(Interface.objects.filter(device=edge_02, _custom_field_data__role="peer"))

        for link in range(2):  # 创建 2 条对等链路
            self.create_p2p_link(next(peer_intfs_01), next(peer_intfs_02))

        # 连接边缘设备和叶交换机
        leaf_intfs_01 = iter(Interface.objects.filter(device=edge_01, _custom_field_data__role="leaf"))
        leaf_intfs_02 = iter(Interface.objects.filter(device=edge_02, _custom_field_data__role="leaf"))

        # 使用 DEVICE_ROLES 中定义的叶设备数量
        num_leaf = DEVICE_ROLES["leaf"]["per_rack"]  # 如有多个机架请相应调整

        for i in range(1, num_leaf + 1):
            leaf_name = f"{site_code}-leaf-{i:02}"
            leaf = self.devices.get(leaf_name)
            if not leaf:
                self.logger.error(f"未找到叶设备 {leaf_name}")
                continue

            edge_intfs = iter(Interface.objects.filter(device=leaf, _custom_field_data__role="edge"))

            # 创建两条线缆：从每台边缘设备连接到该叶设备的边缘接口。
            self.create_p2p_link(next(leaf_intfs_01), next(edge_intfs))
            self.create_p2p_link(next(leaf_intfs_02), next(edge_intfs))
```

我们首先获取边缘设备，并找到在自定义字段数据中定义了 `peer` 角色的接口。获取这些信息后，我们需要创建线缆并将其连接到设备。这里我们再次使用辅助函数来减少重复代码，即 `create_p2p_link` 方法。该方法接受来自两台设备各一个接口，创建一条线缆并将它们连接起来。
```python
    def create_p2p_link(self, interface_a, interface_b):
        """
        在两个接口之间创建点对点链路。
        根据站点的布线要求调整此逻辑。
        """
        
        # 获取每个接口的内容类型
        type_a = ContentType.objects.get_for_model(interface_a)
        type_b = ContentType.objects.get_for_model(interface_b)

        cable, created = Cable.objects.get_or_create(
            termination_a_type=type_a,
            termination_a_id=interface_a.pk,
            termination_b_type=type_b,
            termination_b_id=interface_b.pk,
            defaults={'status': ACTIVE_STATUS}
        )
        if created:
            self.logger.info(f"已在 {interface_a} 和 {interface_b} 之间创建线缆")
        else:
            self.logger.info(f"{interface_a} 和 {interface_b} 之间的线缆已存在")
```

根据机架中定义在 `DEVICE_ROLES` 常量里的叶设备数量，对边缘到叶设备的逻辑进行重复处理。所有设备完成布线后，接下来是创建和终止电路。

在开始创建电路之前，我们需要创建提供商和电路类型。我们已经添加了 `TRANSIT_PROVIDERS` 常量，因此创建提供商和电路类型所需的代码如下所示。
```python
        # ----------------------------------------------------------------------------
        # 创建电路提供商（如不存在）
        # ----------------------------------------------------------------------------
        for provider_name in TRANSIT_PROVIDERS:
            provider_obj, created = Provider.objects.get_or_create(
                name=provider_name,
            )
            if created:
                self.logger.info(f"已创建电路提供商：{provider_obj}")
            else:
                self.logger.info(f"电路提供商 {provider_obj} 已存在")

        # ----------------------------------------------------------------------------
        # 创建 CircuitType 'Transit'（如不存在）
        # ----------------------------------------------------------------------------
        circuit_type, ct_created = CircuitType.objects.get_or_create(
            name="Transit",
        )
        if ct_created:
            self.logger.info("已创建 CircuitType 'Transit'")
        else:
            self.logger.info("CircuitType 'Transit' 已存在")
```

此代码与我们在整个练习中用于创建对象的代码类似，含义不言而喻。

现在我们已经有了提供商和电路类型，可以添加站点创建的最后一段代码来创建并终止电路。
```python
        # ----------------------------------------------------------------------------
        # 创建电路并连接
        # ----------------------------------------------------------------------------
        external_intfs_01 = iter(Interface.objects.filter(device=edge_01, _custom_field_data__role="external"))
        external_intfs_02 = iter(Interface.objects.filter(device=edge_02, _custom_field_data__role="external"))

        for provider_name in TRANSIT_PROVIDERS:
            provider_obj = Provider.objects.get(name=provider_name)

            for intfs_list in [external_intfs_01, external_intfs_02]:
                intf = next(intfs_list)

                regex = re.compile("[^a-zA-Z0-9]")
                clean_name = regex.sub("", f"{site_code}{intf.device.name[-4:]}{intf.name[-4:]}")
                circuit_id = f"{provider_obj.name[0:3]}-{int(clean_name, 36)}"
                circuit, created = Circuit.objects.get_or_create(
                    cid=circuit_id,
                    circuit_type=circuit_type,
                    provider=provider_obj,
                    status=ACTIVE_STATUS,
                    tenant=tenant
                )

                self.logger.info(f"电路 {circuit_id} 创建成功：{circuit}")

                # 如果 A 侧已存在终端，先删除。
                if circuit.circuit_termination_a:
                    circuit.circuit_termination_a.delete()

                # 在 A 侧创建新的电路终端。
                ct = CircuitTermination(
                    circuit=circuit,
                    term_side="A",
                    location=self.site,
                )
                ct.validated_save()

                # 创建线缆将接口连接到电路终端。
                cable_status = Status.objects.get(name="Connected")
                intf_ct = ContentType.objects.get_for_model(intf)
                ct_ct = ContentType.objects.get_for_model(ct)
                cable, cable_created = Cable.objects.get_or_create(
                    termination_a_type=intf_ct,
                    termination_a_id=intf.pk,
                    termination_b_type=ct_ct,
                    termination_b_id=ct.pk,
                    defaults={'status': cable_status},
                )
                if cable_created:
                    self.logger.info(f"已创建连接 {intf} 和电路终端 {ct} 的线缆")
                else:
                    self.logger.info(f"连接 {intf} 和电路终端 {ct} 的线缆已存在")
```

我们首先查找具有 `external` 角色的接口，然后遍历 `TRANSIT_PROVIDERS` 列表，为每个提供商和外部接口创建一条电路。所有电路创建完成后，我们为每个 A 侧（即设备侧）创建电路终端。

最后，我们创建线缆将设备外部接口连接到各自的电路，作业成功完成。

项目的最终代码如下所示，供参考。

## 最终代码
```python
"""创建 POP 类型新站点的 Job。"""

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

####DAY36####
from nautobot.apps.jobs import Job, ObjectVar, StringVar, register_jobs
from nautobot.dcim.models.locations import Location, LocationType
from ipaddress import IPv4Network
from nautobot.extras.models.relationships import Relationship, RelationshipAssociation
from nautobot.extras.choices import RelationshipTypeChoices

####DAY37####
from nautobot.dcim.models.racks import Rack
from nautobot.dcim.choices import RackTypeChoices

####DAY38####
from nautobot.dcim.models.device_components import Interface
from nautobot.dcim.choices import RackTypeChoices, InterfaceTypeChoices
from nautobot.ipam.models import Prefix, VLAN, IPAddress
from nautobot.dcim.models.devices import Device, DeviceType, Platform, Manufacturer


####DAY39####
from nautobot.dcim.models import Cable
from nautobot.circuits.models import Circuit, CircuitTermination, CircuitType, Provider


name = "数据填充 Jobs 集合"


PREFIX_ROLES = ["p2p", "loopback", "server", "mgmt", "pop"]
TENANT_NAME = "Data Center"
ACTIVE_STATUS = Status.objects.get(name="Active")
# VLAN 定义：键也用于查找角色。
VLAN_INFO = {
    "server": 1000,
    "mgmt": 99,
}
CUSTOM_FIELDS = {
    "role": {"models": [Interface], "label": "Role"},
}

# 获取 Prefix 和 VLAN 模型的内容类型。
prefix_ct = ContentType.objects.get_for_model(Prefix)
vlan_ct = ContentType.objects.get_for_model(VLAN)

####DAY35####
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

####DAY36####
POP_PREFIX_SIZE = 16

####DAY37####
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


####DAY39####
TRANSIT_PROVIDERS = ["Equinix", "Cologix", "CoreSite"]

def create_prefix_roles(logger):
    """创建 PREFIX_ROLES 中定义的所有前缀角色，并为 IPAM Prefix 和 VLAN 添加内容类型。"""

    # 获取 Prefix 和 VLAN 模型的内容类型。
    for role in PREFIX_ROLES:
        role_obj, created = Role.objects.get_or_create(name=role)
        # 将 Prefix 和 VLAN 内容类型添加到角色。
        role_obj.content_types.add(prefix_ct, vlan_ct)
        role_obj.validated_save()
        logger.info(f"成功创建角色 {role}，包含 Prefix 和 VLAN 的内容类型。")


def create_tenant(logger):
    """创建 TENANT_NAME 中定义名称的租户。"""
    tenant_obj, _ = Tenant.objects.get_or_create(name=TENANT_NAME)
    tenant_obj.validated_save()
    logger.info(f"成功创建租户 {TENANT_NAME}。")

def create_vlans(logger):
    """创建 VLAN_INFO 中定义的预设 VLAN，并分配相应角色。"""

    for vlan_name, vlan_id in VLAN_INFO.items():
        # 根据 VLAN 名称检索相应角色。
        try:
            role_obj = Role.objects.get(name=vlan_name)
        except Role.DoesNotExist:
            logger.error(f"未找到角色 '{vlan_name}'。将创建没有角色的 VLAN。")
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
            logger.info(f"成功创建 VLAN '{vlan_name}'，ID 为 {vlan_id}。")
        else:
            logger.info(f"VLAN '{vlan_name}'（ID {vlan_id}）已存在。")

def create_custom_fields(logger):
    """创建 CUSTOM_FIELDS 中定义的所有关系。"""
    for cf_name, field in CUSTOM_FIELDS.items():
        try:
            cf = CustomField.objects.get(key=cf_name)
        except CustomField.DoesNotExist:
            cf = CustomField.objects.create(key=cf_name)
            if "label" in field:
                cf.label = field.get("label")
            cf.validated_save()
            logger.info(f"已创建自定义字段 '{cf_name}'")
        for model in field["models"]:
            ct = ContentType.objects.get_for_model(model)
            cf.content_types.add(ct)
            cf.validated_save()
            logger.info(f"已将内容类型 {ct} 添加到自定义字段 '{cf_name}'")

def create_device_types(logger):
    """
    从 YAML 定义创建 DeviceType 对象，并使用 InterfaceTemplate 添加接口。
    """

    for device_yaml in DEVICE_TYPES_YAML:
        data = yaml.safe_load(device_yaml)

        manufacturer_name = data.pop("manufacturer", None)
        if not manufacturer_name:
            logger.error("YAML 定义中未提供制造商。")
            continue
        manufacturer_obj, _ = Manufacturer.objects.get_or_create(name=manufacturer_name)

        model_name = data.pop("model", None)
        if not model_name:
            logger.error("制造商 %s 的 YAML 中未提供型号", manufacturer_name)
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
            logger.info(f"已创建 DeviceType：{device_type_obj}")
        else:
            logger.info(f"DeviceType 已存在：{device_type_obj}")

        # 使用 InterfaceTemplate 添加接口
        for iface in data.get("interfaces", []):
            pattern = iface.get("pattern")
            iface_type = iface.get("type")
            mgmt_only = iface.get("mgmt_only", False)

            if not pattern or not iface_type:
                logger.error(f"{model_name} 中的接口定义无效：{iface}")
                continue

            # 从范围模式生成接口
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
                    logger.info(f"已将接口 {iface_name}（{iface_type}）添加到 {model_name}")


def expand_interface_pattern(pattern):
    """
    将接口模式（如 'Ethernet[1-60]/[1-4]'）展开为实际名称。
    支持：
      - 单范围：Ethernet[1-24] -> Ethernet1, Ethernet2, ..., Ethernet24
      - 嵌套范围：Ethernet[1-60]/[1-4] -> Ethernet1/1, Ethernet1/2, ..., Ethernet60/4
    """
    match = re.findall(r"\[([0-9]+)-([0-9]+)\]", pattern)
    if not match:
        return [pattern]  # 无需展开，原样返回。

    # 转换为数字列表
    ranges = [list(range(int(start), int(end) + 1)) for start, end in match]

    # 使用笛卡尔积生成名称
    expanded_names = []
    base_name = re.sub(r"\[[0-9]+-[0-9]+\]", "{}", pattern)

    for numbers in product(*ranges):
        expanded_names.append(base_name.format(*numbers))

    return expanded_names


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
        self.logger.error(f"创建关系 {label} 时出错：{e}")
        return Relationship.objects.get(key=key)  # 回退到已有关系

####DAY39####
def update_location_type_for_circuit_termination(logger, location_type_obj):
    """
    确保给定的 LocationType 包含 CircuitTermination 的内容类型，
    允许该类型的位置与电路终端相关联。
    """
    from django.contrib.contenttypes.models import ContentType
    from nautobot.circuits.models import CircuitTermination

    ct = ContentType.objects.get_for_model(CircuitTermination)
    if ct not in location_type_obj.content_types.all():
        location_type_obj.content_types.add(ct)
        location_type_obj.validated_save()
        logger.info(f"LocationType '{location_type_obj}' 已更新，包含 CircuitTermination 内容类型。")
    else:
        logger.info(f"LocationType '{location_type_obj}' 已包含 CircuitTermination 内容类型。")

class CreatePop(Job):
    """创建 POP 类型新站点的 Job。"""
    ####DAY36####
    # 接收用户关于站点信息的输入
    location_type = ObjectVar(
    model=LocationType,
    description = "为新站点选择位置类型。"
    )
    parent_site = ObjectVar(
        model=Location,
        required=False,
        description="选择一个现有站点作为上级。若留空，站点将作为顶级 Region 创建。",
        label="上级站点"
    )
    site_name = StringVar(description="新站点的名称", label="站点名称")
    site_facility = StringVar(description="新站点的设施", label="站点设施")
    
    site_code = StringVar(description="输入站点代码，格式为 2 字母州名加 2 位站点编号，例如 NY01 表示纽约商店 ID 01")
    tenant = ObjectVar(model=Tenant)

    def create_p2p_link(self, interface_a, interface_b):
        """
        在两个接口之间创建点对点链路。
        根据站点的布线要求调整此逻辑。
        """
        
        # 获取每个接口的内容类型
        type_a = ContentType.objects.get_for_model(interface_a)
        type_b = ContentType.objects.get_for_model(interface_b)

        cable, created = Cable.objects.get_or_create(
            termination_a_type=type_a,
            termination_a_id=interface_a.pk,
            termination_b_type=type_b,
            termination_b_id=interface_b.pk,
            defaults={'status': ACTIVE_STATUS}
        )
        if created:
            self.logger.info(f"已在 {interface_a} 和 {interface_b} 之间创建线缆")
        else:
            self.logger.info(f"{interface_a} 和 {interface_b} 之间的线缆已存在")

    def run(self, location_type, site_name, site_facility, tenant, site_code, parent_site=None):
        """创建站点的主函数。"""
        # ----------------------------------------------------------------------------
        # 用所有必需的对象初始化数据库。
        # 我们将在接下来的几天中对此进行扩展。
        # ----------------------------------------------------------------------------
        create_prefix_roles(self.logger)
        create_tenant(self.logger)
        create_vlans(self.logger)
        create_custom_fields(self.logger)
        create_device_types(self.logger)

        # ----------------------------------------------------------------------------
        # 创建关系
        # ----------------------------------------------------------------------------
        rel_device_vlan = get_or_create_relationship(
            "Device to VLAN", "device_to_vlan", Device, VLAN, RelationshipTypeChoices.TYPE_MANY_TO_MANY
        )
        rel_rack_vlan = get_or_create_relationship(
            "Rack to VLAN", "rack_to_vlan", Rack, VLAN, RelationshipTypeChoices.TYPE_MANY_TO_MANY
        )

        # ----------------------------------------------------------------------------
        # 创建站点
        # ----------------------------------------------------------------------------
        location_type_obj, created = LocationType.objects.get_or_create(name=location_type)
        update_location_type_for_circuit_termination(self.logger, location_type_obj)
        self.site_name = site_name
        self.site_facility = site_facility
        self.site, created = Location.objects.get_or_create(
            name=site_name,
            location_type=LocationType.objects.get(name=location_type),
            facility=site_facility,
            status=ACTIVE_STATUS,
            parent=parent_site,  # 未提供时为 None
            tenant=tenant
        )
        
        if created:
            message = f"站点 '{site_name}' 已创建为顶级 Region。"
            if parent_site:
                message = f"站点 '{site_name}' 已成功嵌套在 '{parent_site.name}' 下。"
            self.site.validated_save()
            self.logger.info(message)

            pop_role = Role.objects.get(name="pop")
            self.logger.info(f"正在将 '{site_name}' 分配为 '{pop_role}' 角色。")

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
                self.logger.info(f"已将 {pop_prefix} 分配给 {site_name}。")
            else:
                self.logger.warning("未找到可用的 /16 前缀。正在创建新的 /16。")
                top_level_prefix = Prefix.objects.filter(
                    type="container",
                    status=ACTIVE_STATUS,
                    prefix_length=8
                ).first()

                # 获取 /8 内的第一个可用前缀
                first_avail = top_level_prefix.get_first_available_prefix()

                if not first_avail:
                    raise Exception("在 /8 前缀内未找到可用子网。")

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
                        self.logger.info(f"已为站点 '{site_name}' 分配新的 '{pop_prefix}'。")
                        break
                else:
                    raise Exception("在 /8 范围内未找到可用的 /16 前缀。")
        else:
            self.logger.warning(f"站点 '{site_name}' 已存在。") 

        ####DAY37####
        # ----------------------------------------------------------------------------
        # 在 POP 中创建前缀并分配给各角色
        # ----------------------------------------------------------------------------
        
        site_subnets = IPv4Network(str(pop_prefix)).subnets(new_prefix=ROLE_PREFIX_SIZE)
        server_subnet = next(site_subnets)
        mgmt_subnet = next(site_subnets)
        loopback_subnet = next(site_subnets)
        p2p_subnet = next(site_subnets)

        # 将新子网分配给角色
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
        self.logger.info(f"'{server_prefix}' 已分配给 '{server_role}'。")

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
        self.logger.info(f"'{mgmt_prefix}' 已分配给 '{mgmt_role}'。")

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
        self.logger.info(f"'{loopback_prefix}' 已分配给 '{loopback_role}'。")

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
        self.logger.info(f"'{p2p_prefix}' 已分配给 '{p2p_role}'。") 

        # ----------------------------------------------------------------------------
        # 创建机架
        # ----------------------------------------------------------------------------
        # 初始化全局计数器
        global_device_counter = {role: 1 for role in DEVICE_ROLES}  # 追踪编号
        racks = []  # 存储已创建的机架以便后续迭代

        # 创建机架
        num_rack = 2 # 如果站点需要超过 2 个机架，可将其修改为输入变量
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
            self.logger.info(f"成功创建 {rack_name}。")

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
        ####DAY38####
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
                self.logger.info(f"已创建 '{device_role}'")

                # 该机架中第一台设备的起始位置
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
                    self.logger.info(f"设备 {device_name} 已成功在 {rack.name} 中创建")

                    # 将设备保存到我们的清单中，以供后续布线使用
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
                        self.logger.error(f"前缀 {loopback_prefix} 中无可用 IP")
                        return

                    loopback_ip, _ = IPAddress.objects.get_or_create(
                        address=str(loopback_available_ip),
                        status=ACTIVE_STATUS,
                        tenant=tenant,
                        dns_name=f"{role}-{global_device_counter[role]:02}.{site_code}.{tenant.description}"
                    )

                    loopback_ip.mask_length = 32  # L0 子网以 /32 分配，而不是 /18
                    loopback_ip.save()

                    loopback_intf, _ = Interface.objects.get_or_create(
                        name="Loopback0", 
                        type=InterfaceTypeChoices.TYPE_VIRTUAL, 
                        device=device_obj, 
                        status=ACTIVE_STATUS,
                    )

                    loopback_intf.ip_addresses.add(loopback_ip)
                    loopback_intf.save()
                    
                    # 将 L0 IP 设为设备的主 IPv4
                    device_obj.primary_ip4 = loopback_ip
                    device_obj.save()
                    self.logger.info(f"已创建 '{loopback_intf}'，分配了 '{loopback_ip}'，并将其设为 {device_name} 的主 IP")

                    # 为接口分配角色
                    intfs = iter(Interface.objects.filter(device=device_obj))
                    for int_role, cnt in data.get("interfaces", []):
                        for _ in range(cnt):
                            intf = next(intfs, None)
                            if intf:
                                intf._custom_field_data = {"role": int_role}
                                intf.save()

                    # 为叶设备分配 VLAN
                    if role == "leaf":
                        for vlan_name, vlan_id in VLAN_INFO.items():                            
                            vlan_role = Role.objects.get(name=vlan_name)

                            vlan_block = Prefix.objects.filter(
                                location=self.site, 
                                status=ACTIVE_STATUS, 
                                role=vlan_role,
                            ).first()
                        
                            # 查找当前 VLAN 角色（如 server 或 mgmt）的下一个可用网络
                            first_avail = vlan_block.get_first_available_prefix()
                            subnet = list(first_avail.subnet(24))[0]
                            vlan_prefix, created = Prefix.objects.get_or_create(
                                prefix=str(subnet),
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
        ####DAY39####
        # ----------------------------------------------------------------------------
        # 线缆连接
        # ----------------------------------------------------------------------------
        # 连接边缘路由器
        edge_01 = self.devices.get(f"{site_code}-edge-01")
        edge_02 = self.devices.get(f"{site_code}-edge-02")

        # 获取每台边缘设备上具有 'peer' 自定义字段的接口
        peer_intfs_01 = iter(Interface.objects.filter(device=edge_01, _custom_field_data__role="peer"))
        peer_intfs_02 = iter(Interface.objects.filter(device=edge_02, _custom_field_data__role="peer"))

        for link in range(2):  # 创建 2 条对等链路
            self.create_p2p_link(next(peer_intfs_01), next(peer_intfs_02))

        # 连接边缘设备和叶交换机
        leaf_intfs_01 = iter(Interface.objects.filter(device=edge_01, _custom_field_data__role="leaf"))
        leaf_intfs_02 = iter(Interface.objects.filter(device=edge_02, _custom_field_data__role="leaf"))

        # 使用 DEVICE_ROLES 中定义的叶设备数量
        num_leaf = DEVICE_ROLES["leaf"]["per_rack"]  # 如有多个机架请相应调整

        for i in range(1, num_leaf + 1):
            leaf_name = f"{site_code}-leaf-{i:02}"
            leaf = self.devices.get(leaf_name)
            if not leaf:
                self.logger.error(f"未找到叶设备 {leaf_name}")
                continue

            edge_intfs = iter(Interface.objects.filter(device=leaf, _custom_field_data__role="edge"))

            # 创建两条线缆：从每台边缘设备连接到该叶设备的边缘接口。
            self.create_p2p_link(next(leaf_intfs_01), next(edge_intfs))
            self.create_p2p_link(next(leaf_intfs_02), next(edge_intfs))

        # ----------------------------------------------------------------------------
        # 创建电路提供商（如不存在）
        # ----------------------------------------------------------------------------
        for provider_name in TRANSIT_PROVIDERS:
            provider_obj, created = Provider.objects.get_or_create(
                name=provider_name,
            )
            if created:
                self.logger.info(f"已创建电路提供商：{provider_obj}")
            else:
                self.logger.info(f"电路提供商 {provider_obj} 已存在")

        # ----------------------------------------------------------------------------
        # 创建 CircuitType 'Transit'（如不存在）
        # ----------------------------------------------------------------------------
        circuit_type, ct_created = CircuitType.objects.get_or_create(
            name="Transit",
        )
        if ct_created:
            self.logger.info("已创建 CircuitType 'Transit'")
        else:
            self.logger.info("CircuitType 'Transit' 已存在")

        # ----------------------------------------------------------------------------
        # 创建电路并连接
        # ----------------------------------------------------------------------------
        external_intfs_01 = iter(Interface.objects.filter(device=edge_01, _custom_field_data__role="external"))
        external_intfs_02 = iter(Interface.objects.filter(device=edge_02, _custom_field_data__role="external"))

        for provider_name in TRANSIT_PROVIDERS:
            # 获取提供商
            provider_obj = Provider.objects.get(name=provider_name)

            for intfs_list in [external_intfs_01, external_intfs_02]:
                intf = next(intfs_list)

                regex = re.compile("[^a-zA-Z0-9]")
                clean_name = regex.sub("", f"{site_code}{intf.device.name[-4:]}{intf.name[-4:]}")
                circuit_id = f"{provider_obj.name[0:3]}-{int(clean_name, 36)}"
                circuit, created = Circuit.objects.get_or_create(
                    cid=circuit_id,
                    circuit_type=circuit_type,
                    provider=provider_obj,
                    status=ACTIVE_STATUS,
                    tenant=tenant
                )

                self.logger.info(f"电路 {circuit_id} 创建成功：{circuit}")

                # 如果 A 侧已存在终端，先删除。
                if circuit.circuit_termination_a:
                    circuit.circuit_termination_a.delete()

                # 在 A 侧创建新的电路终端。
                ct = CircuitTermination(
                    circuit=circuit,
                    term_side="A",
                    location=self.site,
                )
                ct.validated_save()

                # 创建线缆将接口连接到电路终端。
                cable_status = Status.objects.get(name="Connected")
                intf_ct = ContentType.objects.get_for_model(intf)
                ct_ct = ContentType.objects.get_for_model(ct)
                cable, cable_created = Cable.objects.get_or_create(
                    termination_a_type=intf_ct,
                    termination_a_id=intf.pk,
                    termination_b_type=ct_ct,
                    termination_b_id=ct.pk,
                    defaults={'status': cable_status},
                )
                if cable_created:
                    self.logger.info(f"已创建连接 {intf} 和电路终端 {ct} 的线缆")
                else:
                    self.logger.info(f"连接 {intf} 和电路终端 {ct} 的线缆已存在")

register_jobs(CreatePop)
```

🚀现在，让我们期待已久的时刻到来了……启动我们的 Job，看看结果吧！🚀

首先，让我们导航到"CIRCUITS->Circuits"来检查电路，应该看到如下内容：

![电路](images/create_site_day39_1.png)

这些电路基于我们创建的提供商。

![电路](images/create_site_day39_2.png)

我们还需要验证电路是否被分配为"Transit"类型。可以导航到"CIRCUITS->Circuit Types->Transit"来确认。

![Transit 电路](images/create_site_day39_3.png)

最后……最后一个要素是线缆连接！在这里，我们需要确保 `edge` 设备已与 `Circuits` 和 `leaf` 设备建立了连接。

![线缆连接](images/create_site_day39_4.png)


现在来更新我们的清单并庆祝吧！

🎉 🎉 恭喜完成这项漫长的练习 🎉 🎉

✅ 第36天：

    ✅ 创建关系

    ✅ 创建站点

    ✅ 分配 /16 前缀


✅ 第37天：

    ✅ 为每个角色创建并分配前缀

    ✅ 创建机架

    ✅ 建立机架与 VLAN 的关系


✅ 第38天：

    ✅ 创建设备

    ✅ 为关键接口分配 VLAN 和 IP

    ✅ 建立设备与 VLAN 的关系


✅ 第39天：

    ✅ 将电路连接到边缘设备

    ✅ 为设备之间连接线缆


## 第39天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

请在你选择的社交媒体上发布新 Job 成功执行的截图，务必使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，这样我们可以分享你的进展！

在明天的挑战中，我们将进行回顾并展望 100 Days of Nautobot 挑战后续几天的内容。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+39+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 39 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
