# 设计未来站点（第三部分）

准备好在前两天的基础上继续进阶了吗？我们出发吧！

## 环境配置

我们将继续从 [第 35 天](../Day035_Design_Future_Sites_Part_2/README.md) 的进度继续构建。

> [!IMPORTANT]
> 在 Nautobot 数据库中为 Arista EOS 配置网络驱动映射，是进入站点创建阶段前的最后一个关键步骤。前两天的重点是用生产环境中通常已存在的对象来配置我们的环境，现在还需要最后一步，确保所有内容都正确映射后再继续推进。

## 映射网络驱动

> [!NOTE]
> 当我们到达第 38 天实际创建设备时，`"platform"` 变量将指向此属性。

以下是映射网络驱动的快速参考说明。如需更多帮助，请参阅本挑战的 [第 10 天](../Day010_Python_Script_to_Jobs_Part_2)。

在左侧导航中进入"DEVICES -> Platforms"，编辑"Arista EOS"平台。

![Edit Platform](../Day010_Python_Script_to_Jobs_Part_2/images/driver_1.png)

将网络驱动选择为"arista_eos"。

![Edit Platform](../Day010_Python_Script_to_Jobs_Part_2/images/driver_2.png)

## 开始在 Nautobot 中构建站点！

激动人心的部分来了！

基础工作已经就绪——现在是时候创建站点了！在接下来几天里，我们将自动化在 Nautobot 中构建站点的全流程，涵盖所有核心组件：角色、前缀、子网、VLAN、机架、设备、布线等。

听起来很多？别担心！我们将其拆分为小而可管理的任务，以便专注于学习并稳步推进。

每天都在前一天的基础上构建，最终我们将拥有一个完整配置的站点！以下是计划：

✅ 第 36 天：
- [ ] 创建关联关系
- [ ] 创建站点
- [ ] 分配 /16 前缀

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

🔥 开始吧！

每天结束时，我们将回顾这份清单，看看完成了多少进度。进步的感觉很好，对吧？让我们投入其中！

## 设计未来站点第三部分代码

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

首先添加必要的导入语句并声明额外的变量。随着我们处理更多 Nautobot 对象，这里还会继续扩充。
```python
from nautobot.apps.jobs import Job, ObjectVar, StringVar, register_jobs
from nautobot.dcim.models.locations import Location, LocationType
from ipaddress import IPv4Network
from nautobot.extras.models.relationships import Relationship
from nautobot.extras.choices import RelationshipTypeChoices

...
POP_PREFIX_SIZE = 16
```

有时，我们需要在可信源中定义对象之间的自定义关联关系，以反映业务逻辑或其他非内置的连接关系。这正是 Relationships 功能的用武之地——它允许根据特定的网络或数据需求在对象间创建链接，供后续使用。如需深入了解 Nautobot 中的 `Relationships`，请查阅[官方文档](https://docs.nautobot.com/projects/core/en/stable/user-guide/feature-guides/relationships/?h=relationships)。

我们需要在代码顶层定义一个函数以便后续调用。在接下来几天中，我们将建立机架、设备和 VLAN 之间的关联关系，现在定义好函数会让后续代码引用更加方便。
```python
...
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
```

在 `run()` 方法中调用该函数，添加以下代码：
```python
        ...
        create_prefix_roles(self.logger)
        create_tenant(self.logger)
        create_vlans(self.logger)
        create_device_types(self.logger)

        # ----------------------------------------------------------------------------
        # 创建关联关系
        # ----------------------------------------------------------------------------
        rel_device_vlan = get_or_create_relationship(
            "Device to VLAN", "device_to_vlan", Device, VLAN, RelationshipTypeChoices.TYPE_MANY_TO_MANY
        )
        rel_rack_vlan = get_or_create_relationship(
            "Rack to VLAN", "rack_to_vlan", Rack, VLAN, RelationshipTypeChoices.TYPE_MANY_TO_MANY
        )
```

以下部分涉及创建站点的具体细节。我们将接受用户输入来定义新 POP 站点的参数，如站点名称、站点代码等。我们将在 `class CreatePop(Job)` 下添加此部分，允许自定义站点名称、区域和站点代码等详情。秉持自动化精神，标准化有助于保持结构清晰并确保命名规范的一致性！

同时，还需要修改 `run()` 方法的参数，加入接收到的输入数据。
```python
class CreatePop(Job):
    """用于创建 POP 类型新站点的 Job。"""

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
    # 将接收到的数据作为参数传递给 run() 方法
    def run(self, location_type, site_name, site_facility, tenant, site_code, parent_site=None):
```

> [!TIP]
> 我们希望代码保持模块化，因此后续所有任务都将追加在 `run()` 方法最后一项的下方，注意缩进！

以下代码块实际使用输入数据来创建 POP 站点的具体配置：
```python
        ...
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
```

最后，我们将在站点创建完成后为其分配 /16 前缀。在本实验环境中，我们使用分配给"East Coast"区域（作为父站点）的现有 /8 前缀。思路是首先查找可用的 /16 前缀，如果不存在，则将父前缀进一步划分为更小的 /16 并分配给新站点。

根据环境不同可以调整代码，但此方法确保我们实现了嵌套层次结构。
```python
            ...
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
```

## 第 36 天最终代码
```python
"""用于创建 POP 类型新站点的 Job。"""

from itertools import product
import re

from django.contrib.contenttypes.models import ContentType
import yaml

from nautobot.dcim.models import DeviceType, Manufacturer
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
from nautobot.extras.models.relationships import Relationship
from nautobot.extras.choices import RelationshipTypeChoices
from ipaddress import IPv4Network


name = "Data Population Jobs Collection"


####第36天####
POP_PREFIX_SIZE = 16

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

        ####第36天####
        # ----------------------------------------------------------------------------
        # 创建站点
        # ----------------------------------------------------------------------------
        location_type_site, _ = LocationType.objects.get_or_create(name=location_type)
        self.site_name = site_name
        self.site_facility = site_facility
        self.site, created = Location.objects.get_or_create(
            name = site_name,
            location_type = LocationType.objects.get(name=location_type),
            facility = site_facility,
            status = ACTIVE_STATUS,
            parent = parent_site,  # 如果未提供则为 None
            tenant = tenant
        )
        
        if created:
            message = f"Site '{site_name}' created as a top level Region."
            if parent_site:
                message = f"Site '{site_name}' successfully nested under '{parent_site.name}'."
            self.site.validated_save()
            self.logger.info(message)

            # ----------------------------------------------------------------------------
            # 为此 POP 分配前缀
            # ----------------------------------------------------------------------------
            pop_role = Role.objects.get(name="pop")
            self.logger.info(f"Assigning '{site_name}' as '{pop_role}' role.")

            # 查找第一个尚未分配给站点的可用 /16 前缀
            pop_prefix = Prefix.objects.filter(
                type="container",  # 确保是分配为容器类型的顶层子网
                prefix_length = POP_PREFIX_SIZE,
                status = ACTIVE_STATUS,
                location__isnull = True  # 确保尚未分配给其他站点
            ).first()

            if pop_prefix:
                # 将前缀分配给新站点
                pop_prefix.location = self.site
                pop_prefix.validated_save()
                self.logger.info(f"Assigned {pop_prefix} to {site_name}.")
            else:                 
                self.logger.warning("No available /16 prefixes found. Creating a new /16.")
                
                # 查找顶层 /8 前缀
                top_level_prefix = Prefix.objects.filter(
                    type = "container",  
                    status = ACTIVE_STATUS,
                    prefix_length = 8
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
                        self.logger.info(f"Allocated new'{pop_prefix}' for site '{site_name}'.")
                        break
                else:
                    raise Exception("No available /16 prefixes found within the /8 range.")        
        
        else:
            self.logger.warning(f"Site '{site_name}' already exists.") 
    
register_jobs(CreatePop)
```

🚀 我们现在可以运行 Job 了！🚀

![Create Site Input](images/create_site_day36_1.png)

> [!TIP]
> 在项目推进过程中，我们将使用站点所在美国州名与两位数店铺 ID 的组合作为站点名称，例如 NY01 代表纽约第 01 号门店。这只是为了方便学习，在生产环境中，您需要制定一套标准化命名策略，涵盖站点代码、设备名称、机架名称等。

以下是执行结果！导航至"ORGANIZATION -> LOCATIONS -> Locations"即可找到新创建的站点。

![Created Site](images/create_site_day36_2.png)

注意 /16 前缀已分配给您的站点，并嵌套在父站点的 /8 前缀下。

![Prefix result](images/create_site_day36_3.png)

> [!TIP]
> 如果遇到错误或冲突，请清理数据库，删除之前 Job 创建的对象。在挑战过程中，前期 Job 创建的对象有时可能导致代码报错。我们尝试通过验证器捕获所有错误，但根据您的环境不同，冲突仍可能发生。

## 回顾

内容相当丰富！但正如承诺的那样，让我们保持动力——是时候更新今日完成的清单了！🎉

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

## 第 36 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上分享过去三天的心得体会，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将进入第 37 天和第四部分的任务！明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+36+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 36 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
