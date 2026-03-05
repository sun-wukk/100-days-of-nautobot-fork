# VLAN 变更 Job

在第 10 天的挑战中，我们成功使用 Nautobot Jobs 对 Nautobot 管理的设备执行了 'show' 命令。在今天的挑战中，我们将更进一步，执行能够进行运维变更的命令，即更改端口上的 VLAN。

在开始之前，让我们先准备好实验环境，今天的实验方式略有不同。

## 环境配置

环境配置与 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md) 相同，以下是步骤摘要，如需详细背景说明请参阅该指南。

> [!IMPORTANT]
> 如果您有足够的 Codespace 额度，或不介意付费以获得更好的体验，建议使用更多核心启动 Codespace 实例。建议从 `4核` 开始。

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

上传并准备 cEOS 镜像，然后启动 Containerlab：

> [!IMPORTANT]
> 请记得替换为与您下载版本匹配的版本号。

> [!WARNING]
> 在以下示例中，文件已解压，不含 `.zip` 扩展名。请根据实际情况添加 `.zip` 扩展名。
```
$ docker import cEOS64-lab-4.32.0F.tar ceos:4.32.0F
```

本实验只需要 BOS 设备：
```
$ cd clab/
$ sudo containerlab deploy --topo ceos-lab.clab.yml --node-filter bos-acc-01,bos-rtr-01
```

为今天的挑战创建文件，可以通过共享目录或直接在 Nautobot Docker 容器中操作：
```
$ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash
root@c9e0fa2a45a0:/opt/nautobot# cd jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# pwd
/opt/nautobot/jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# touch operation_jobs.py
root@c9e0fa2a45a0:/opt/nautobot/jobs# chown nautobot:nautobot operation_jobs.py
```

今天挑战的环境已配置完毕。

> [!NOTE]
> 今天的挑战是第 10 天挑战的延伸，请随时将第 10 天 Job 的最终版本复制粘贴过来后再继续。

下一步，我们将创建一个允许更改端口 VLAN 配置的脚本。

## VLAN 变更 Job

照例，先从导入开始：
```
import os

from django.conf import settings
from nautobot.apps.jobs import MultiChoiceVar, Job, ObjectVar, register_jobs, StringVar, IntegerVar
from nautobot.dcim.models.locations import Location
from nautobot.dcim.models.devices import Device
from nautobot.dcim.models.device_components import Interface
from netmiko import ConnectHandler
from nautobot.ipam.models import VLAN
from nautobot.apps.jobs import JobButtonReceiver
```

使用特殊全局变量 ```name``` 来指定 Jobs 的名称：
```
name = "Network Operations"
```

在 Job 的第一部分，我们希望实现以下功能：

1. 从现有位置中选择一个位置。
2. 从所选位置中选择一台设备。
3. 仅将以太网接口作为 VLAN 变更的选项（我们不希望在环回接口等接口上更改 VLAN）。

如何实现？我们可以使用 ```query_params``` 来限制范围：
```
class ChangeVLAN(Job):
    device_location = ObjectVar(model=Location, required=False)

    device = ObjectVar(
        model=Device,
        query_params={
            "location": "$device_location",
        },
    )

    interface = ObjectVar(
        model=Interface,
        query_params={
            "device_id": "$device",
            "name__ic": "Ethernet"
        }
    )
```

注意我们使用 ```"name__ic": "Ethernet"```，双下划线表示名称"包含"关键字"Ethernet"。

然后定义 ```IntegerVar``` 用于输入 VLAN 编号，以及包含元信息的 Meta 类：
```
    # 指定要配置的 VLAN 编号
    vlan = IntegerVar()
    
    
    class Meta:
        name = "Change VLAN for Port"
        description = "Change VLAN based on Selected Port."
```

在主 ```run()``` 方法中，执行常规检查后使用 Netmiko 执行命令：
```
    def run(self, device_location, device, interface, vlan):
        """执行设备检查的 run 方法。"""
        self.logger.info(f"Device: {device.name}, Interface: {interface}")

        # 验证设备是否已设置主 IP
        if device.primary_ip is None:
            self.logger.fatal("Device does not have a primary IP address set.")
            return

        # 验证设备是否已关联平台
        if device.platform is None:
            self.logger.fatal("Device does not have a platform set.")
            return

        # 检查设备驱动关联
        if device.platform.network_driver_mappings.get("netmiko") is None:
            self.logger.fatal("Device mapping for Netmiko is not present, please set.")
            return

        # 连接设备并获取输出 - 如果是模拟模式请注释掉此部分
        net_connect = ConnectHandler(
            device_type=device.platform.network_driver_mappings["netmiko"],
            host=device.primary_ip.host,  # 如果设备名称是 FQDN，也可以使用 device.name
            username="admin",
            password="admin",
        )

        # 平台与设备命令的简单映射
        COMMAND_MAP = {
            "cisco_nxos": [f"interface {interface}", f"switchport access vlan {vlan}"],
            "arista_eos": [f"interface {interface}", f"switchport access vlan {vlan}"],
        }

        commands = COMMAND_MAP[device.platform.network_driver_mappings.get("netmiko")]
        self.logger.info(f"This is the command: {commands}")
        # output = net_connect.send_command(commands)
        net_connect.enable()
        output = net_connect.send_config_set(commands)
        net_connect.save_config()
        net_connect.disconnect()
        self.logger.info(f"This is the output: {output}")

        # 如果未抛出异常，则配置已成功下发
        self.logger.info(
            interface, f"Successfully added to {interface.name} on {device.name}!"
        )
```

在 run 方法中，我们使用 COMMAND_MAP 来处理 "cisco_nxos" 和 "arista_eos" 之间潜在的命令差异：
```
        # 平台与设备命令的简单映射
        COMMAND_MAP = {
            "cisco_nxos": [f"interface {interface}", f"switchport access vlan {vlan}"],
            "arista_eos": [f"interface {interface}", f"switchport access vlan {vlan}"],
        }

        commands = COMMAND_MAP[device.platform.network_driver_mappings.get("netmiko")]
        self.logger.info(f"This is the command: {commands}")
```

最后一步是注册 Job：
```
register_jobs(
    ChangeVLAN,  
    CommandRunner,
)
```

启用 Job 并进行测试：

![change_vlan_1](images/change_vlan_1.png)

在结果页面上，可以看到命令已下发以及执行结果：

![change_vlan_2](images/change_vlan_2.png)

非常棒，我们可以通过 Nautobot 更改端口上的 VLAN 了！

让我们再做一个改进：与其使用 ```IntegerVar``` 手动输入 VLAN 编号，不如将 VLAN 的范围限定为已存在的 VLAN。

## 基于已有 VLAN 的 VLAN 变更

我们应该已经创建了 VLAN 10 和 VLAN 20，如有需要可以创建更多：

![create_vlans](images/create_vlans.png)

我们可以创建另一个 Job，将 VLAN 变更范围限定为已存在的 VLAN：
```
class ChangeVLAN_by_Function(Job):
    device_location = ObjectVar(model=Location, required=False)

    device = ObjectVar(
        model=Device,
        query_params={
            "location": "$device_location",
        },
    )

    interface = ObjectVar(
        model=Interface,
        query_params={
            "device_id": "$device",
            "name__ic": "Ethernet"
        }
    )

    vlan = ObjectVar(
        model=VLAN,
    )
    
    
    class Meta:
        name = "Change VLAN on Port by existing VLAN"
        description = "Change VLAN on Port by existing VLAN."

    def run(self, device_location, device, interface, vlan):
        """执行设备检查的 run 方法。"""
        self.logger.info(f"Device: {device.name}, Interface: {interface}")

        # 验证设备是否已设置主 IP
        if device.primary_ip is None:
            self.logger.fatal("Device does not have a primary IP address set.")
            return

        # 验证设备是否已关联平台
        if device.platform is None:
            self.logger.fatal("Device does not have a platform set.")
            return

        # 检查设备驱动关联
        if device.platform.network_driver_mappings.get("netmiko") is None:
            self.logger.fatal("Device mapping for Netmiko is not present, please set.")
            return

        # 连接设备并获取输出 - 如果是模拟模式请注释掉此部分
        net_connect = ConnectHandler(
            device_type=device.platform.network_driver_mappings["netmiko"],
            host=device.primary_ip.host,  # 如果设备名称是 FQDN，也可以使用 device.name
            username="admin",
            password="admin",
        )

        # 平台与设备命令的简单映射
        COMMAND_MAP = {
            "cisco_nxos": [f"interface {interface}", f"switchport access vlan {vlan.vid}"],
            "arista_eos": [f"interface {interface}", f"switchport access vlan {vlan.vid}"],
        }

        commands = COMMAND_MAP[device.platform.network_driver_mappings.get("netmiko")]
        self.logger.info(f"This is the command: {commands}")
        # output = net_connect.send_command(commands)
        net_connect.enable()
        output = net_connect.send_config_set(commands)
        net_connect.save_config()
        net_connect.disconnect()
        self.logger.info(f"This is the output: {output}")

        # 如果未抛出异常，则配置已成功下发
        self.logger.info(
            interface, f"Successfully added VLAN {vlan.name} to {interface.name} on {device.name}!"
        )


register_jobs(
    ChangeVLAN,  
    ChangeVLAN_by_Function,
    CommandRunner,
)
```

现在进入该 Job 时，下拉选项中只会显示已存在的 VLAN：

![vlan_based_on_functions](images/vlan_based_on_functions.png)

## 最终脚本

以下是 Job 文件的最终版本：
```
import os

from django.conf import settings
from nautobot.apps.jobs import MultiChoiceVar, Job, ObjectVar, register_jobs, StringVar, IntegerVar
from nautobot.dcim.models.locations import Location
from nautobot.dcim.models.devices import Device
from nautobot.dcim.models.device_components import Interface
from netmiko import ConnectHandler
from nautobot.ipam.models import VLAN
from nautobot.apps.jobs import JobButtonReceiver


name = "Network Operations"


COMMAND_CHOICES = (
    ("show ip interface brief", "show ip int bri"),
    ("show ip route", "show ip route"),
    ("show version", "show version"),
    ("show log", "show log"),
    ("show ip ospf neighbor", "show ip ospf neighbor"),
)


class CommandRunner(Job):
    device_location = ObjectVar(model=Location, required=False)

    device = ObjectVar(
        model=Device,
        query_params={
            "location": "$device_location",
        },
    )

    commands = MultiChoiceVar(choices=COMMAND_CHOICES)

    class Meta:
        name = "Command Runner"
        has_sensitive_variables = False
        description = "Command Runner"

    def run(self, device_location, device, commands):
        self.logger.info("Device name: %s", device.name)
    
        # 验证设备是否已设置主 IP
        if device.primary_ip is None:
            self.logger.fatal("Device does not have a primary IP address set.")
            return

        # 验证设备是否已关联平台
        if device.platform is None:
            self.logger.fatal("Device does not have a platform set.")
            return

        # 检查设备驱动关联
        if device.platform.network_driver_mappings.get("netmiko") is None:
            self.logger.fatal("Device mapping for Netmiko is not present, please set.")
            return

        # 连接设备并获取输出 - 如果是模拟模式请注释掉此部分
        net_connect = ConnectHandler(
            device_type=device.platform.network_driver_mappings["netmiko"],
            host=device.primary_ip.host,  # 如果设备名称是 FQDN，也可以使用 device.name
            # username=os.getenv("DEVICE_USERNAME"),  # 改为使用 user_name
            # password=os.getenv("DEVICE_PASSWORD"),
            username="admin",
            password="admin",
        )
        for command in commands:
            output = net_connect.send_command(
                command
            )  
            self.create_file(f"{device.name}-{command}.txt", output)

class ChangeVLAN(Job):
    device_location = ObjectVar(model=Location, required=False)

    device = ObjectVar(
        model=Device,
        query_params={
            "location": "$device_location",
        },
    )

    interface = ObjectVar(
        model=Interface,
        query_params={
            "device_id": "$device",
            "name__ic": "Ethernet"
        }
    )

    # 指定要配置的 VLAN 编号
    vlan = IntegerVar()
    
    
    class Meta:
        name = "Change VLAN for Port"
        description = "Change VLAN based on Selected Port."

    def run(self, device_location, device, interface, vlan):
        """执行设备检查的 run 方法。"""
        self.logger.info(f"Device: {device.name}, Interface: {interface}")

        # 验证设备是否已设置主 IP
        if device.primary_ip is None:
            self.logger.fatal("Device does not have a primary IP address set.")
            return

        # 验证设备是否已关联平台
        if device.platform is None:
            self.logger.fatal("Device does not have a platform set.")
            return

        # 检查设备驱动关联
        if device.platform.network_driver_mappings.get("netmiko") is None:
            self.logger.fatal("Device mapping for Netmiko is not present, please set.")
            return

        # 连接设备并获取输出 - 如果是模拟模式请注释掉此部分
        net_connect = ConnectHandler(
            device_type=device.platform.network_driver_mappings["netmiko"],
            host=device.primary_ip.host,  # 如果设备名称是 FQDN，也可以使用 device.name
            username="admin",
            password="admin",
        )

        # 平台与设备命令的简单映射
        COMMAND_MAP = {
            "cisco_nxos": [f"interface {interface}", f"switchport access vlan {vlan}"],
            "arista_eos": [f"interface {interface}", f"switchport access vlan {vlan}"],
        }

        commands = COMMAND_MAP[device.platform.network_driver_mappings.get("netmiko")]
        self.logger.info(f"This is the command: {commands}")
        # output = net_connect.send_command(commands)
        net_connect.enable()
        output = net_connect.send_config_set(commands)
        net_connect.save_config()
        net_connect.disconnect()
        self.logger.info(f"This is the output: {output}")

        # 如果未抛出异常，则配置已成功下发
        self.logger.info(
            interface, f"Successfully added to {interface.name} on {device.name}!"
        )
        

class ChangeVLAN_by_Function(Job):
    device_location = ObjectVar(model=Location, required=False)

    device = ObjectVar(
        model=Device,
        query_params={
            "location": "$device_location",
        },
    )

    interface = ObjectVar(
        model=Interface,
        query_params={
            "device_id": "$device",
            "name__ic": "Ethernet"
        }
    )

    vlan = ObjectVar(
        model=VLAN,
    )
    
    
    class Meta:
        name = "Change VLAN on Port by existing VLAN"
        description = "Change VLAN on Port by existing VLAN."

    def run(self, device_location, device, interface, vlan):
        """执行设备检查的 run 方法。"""
        self.logger.info(f"Device: {device.name}, Interface: {interface}")

        # 验证设备是否已设置主 IP
        if device.primary_ip is None:
            self.logger.fatal("Device does not have a primary IP address set.")
            return

        # 验证设备是否已关联平台
        if device.platform is None:
            self.logger.fatal("Device does not have a platform set.")
            return

        # 检查设备驱动关联
        if device.platform.network_driver_mappings.get("netmiko") is None:
            self.logger.fatal("Device mapping for Netmiko is not present, please set.")
            return

        # 连接设备并获取输出 - 如果是模拟模式请注释掉此部分
        net_connect = ConnectHandler(
            device_type=device.platform.network_driver_mappings["netmiko"],
            host=device.primary_ip.host,  # 如果设备名称是 FQDN，也可以使用 device.name
            username="admin",
            password="admin",
        )

        # 平台与设备命令的简单映射
        COMMAND_MAP = {
            "cisco_nxos": [f"interface {interface}", f"switchport access vlan {vlan.vid}"],
            "arista_eos": [f"interface {interface}", f"switchport access vlan {vlan.vid}"],
        }

        commands = COMMAND_MAP[device.platform.network_driver_mappings.get("netmiko")]
        self.logger.info(f"This is the command: {commands}")
        # output = net_connect.send_command(commands)
        net_connect.enable()
        output = net_connect.send_config_set(commands)
        net_connect.save_config()
        net_connect.disconnect()
        self.logger.info(f"This is the output: {output}")

        # 如果未抛出异常，则配置已成功下发
        self.logger.info(
            interface, f"Successfully added VLAN {vlan.name} to {interface.name} on {device.name}!"
        )


register_jobs(
    ChangeVLAN,  
    ChangeVLAN_by_Function,
    CommandRunner,
)
```

今天的挑战收获颇丰。让我们先告一段落，明天继续新的挑战。

## 第 11 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

您能想到对 VLAN 变更 Job 还可以做哪些改进吗？欢迎在社交媒体上发表您的想法，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将深入了解 Job Buttons。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+11+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 11 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
