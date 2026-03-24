# 在 Jobs 中访问 Secrets

出于显而易见的原因，我们需要将 API 令牌、设备访问用户名和密码等机密信息作为 Secret 保存。在 Nautobot 中，Secrets 管理功能可以在 [Secrets and Security](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/secret/#secrets-and-security) 中找到。

Nautobot 中的 Secrets 管理涉及以下几个概念：

- **Secrets（密钥）**：Nautobot 中的 Secret 存储的是*如何检索密钥的引用*，而**非**密钥本身。
- **Secrets Group（密钥组）**：Secrets Group 包含一组密钥集合，可附加到设备或 Git 仓库等对象上。
- **Secrets providers（密钥提供者）**：[providers](https://github.com/nautobot/nautobot-app-secrets-providers) 是获取密钥实际值的来源，支持环境变量以及 `HashiCorp Vault`、`AWS Secrets Manager` 等第三方提供者。

> [!IMPORTANT]
> 务必注意，不要在 Job 日志中泄露密钥信息。

让我们来看一个在 Nautobot Jobs 中使用 Secrets 的示例。

## 环境配置

环境配置与 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md) 相同，以下是步骤摘要，如需详细背景说明请参阅该指南。

按照以下步骤启动 Nautobot，如果是重启已有实例且 `build` 和 `db-import` 已完成，可以跳过相应步骤：
```
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

上传并准备 `cEOS` 镜像，然后启动 Containerlab：
```
$ docker import cEOS64-lab-4.32.0F.tar ceos:4.32.0F
```

本实验只需要 `bos-acc-01` 设备：
```
$ cd ~/100-days-of-nautobot/clab/
$ sudo containerlab deploy --topo ceos-lab.clab.yml --node-filter bos-acc-01
```

今天挑战的环境已配置完毕。

## Command Runner Job

我们将以 [第 9 天的 Command Runner](https://github.com/nautobot/100-days-of-nautobot/blob/main/Day010_Python_Script_to_Jobs_Part_2/README.md) Job 为基础进行今天的挑战。

以下是该 Job 的内容供参考：
```python
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
            host=device.primary_ip.host,
            username="admin",
            password="admin",
        )
        for command in commands:
            output = net_connect.send_command(
                command
            )  
            self.create_file(f"{device.name}-{command}.txt", output)



register_jobs(
    CommandRunner,
)
```

在继续下一步之前，请确保该 Job 能够成功执行，如有需要请参阅 [第 9 天 Command Runner](https://github.com/nautobot/100-days-of-nautobot/blob/main/Day010_Python_Script_to_Jobs_Part_2/README.md)：

![command_runner_1](images/command_runner_1.png)
![command_runner_2](images/command_runner_2.png)

注意 Job 文件中的用户名和密码是硬编码的：
```python
        net_connect = ConnectHandler(
            device_type=device.platform.network_driver_mappings["netmiko"],
            host=device.primary_ip.host,
            username="admin",
            password="admin",
        )
```

在下一步中，我们将使用 `Nautobot Secrets` 替换这些硬编码的值。

## Nautobot Secrets

首先创建 `Secrets`。导航到 "Secrets -> '+'"，将密钥分别命名为 'ARISTA_USERNAME' 和 'ARISTA_PASSWORD'，并选择"环境变量"作为提供者：

![arista_username](images/arista_username.png)

![arista_password](images/arista_password.png)

接下来了解如何访问这些密钥。

## 访问 Secret 值

连接到 Nautobot 容器并设置两个环境变量：
```
$ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash

root@ee2753f052ae:/opt/nautobot# export ARISTA_USERNAME="admin"
root@ee2753f052ae:/opt/nautobot# export ARISTA_PASSWORD="admin"
```

启动 `nbshell` 并查看如何访问密钥的值：
```
root@ee2753f052ae:/opt/nautobot# nautobot-server nbshell
...
>>> from nautobot.extras.models.secrets import Secret
>>> username = Secret.objects.get(name="ARISTA_USERNAME").get_value()
>>> password = Secret.objects.get(name="ARISTA_PASSWORD").get_value()
>>> username
'admin'
>>> password
'admin'
```

下一步，在 `command_runner.py` Job 中使用这些密钥值。

## 在 Job 中使用 Secrets

由于 Job 通过 Nautobot Worker 以异步方式执行，添加新环境变量最简便的方式是修改 `creds.env` 文件：

![creds_env_1](images/creds_env_1.png)

需要重启 Nautobot 容器：
```
Ctrl+C
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke debug 
Starting Nautobot in debug mode...
Running docker compose command "up"
 Container nautobot_docker_compose-redis-1  Created
 Container nautobot_docker_compose-db-1  Created
 Container nautobot_docker_compose-nautobot-1  Created
 Container nautobot_docker_compose-celery_worker-1  Created
 Container nautobot_docker_compose-celery_beat-1  Created
Attaching to celery_beat-1, celery_worker-1, db-1, nautobot-1, redis-1
...
```

修改 `command_runner.py` 文件中的 `username` 和 `password`，改为使用 Secrets：
```
from nautobot.extras.models.secrets import Secret

...
        # 连接设备并获取输出 - 如果是模拟模式请注释掉此部分
        net_connect = ConnectHandler(
            device_type=device.platform.network_driver_mappings["netmiko"],
            host=device.primary_ip.host,  # 如果设备名称是 FQDN，也可以使用 device.name
            username=Secret.objects.get(name="ARISTA_USERNAME").get_value(),  
            password=Secret.objects.get(name="ARISTA_PASSWORD").get_value(),
        )
        for command in commands:
            output = net_connect.send_command(
                command
            )  
            self.create_file(f"{device.name}-{command}.txt", output)
...
```

执行结果与之前相同，但 Job 现在使用密钥值而非硬编码值：

![command_runner_3](images/command_runner_3.png)

## 最终 Job 文件

以下是最终版本的 `command_runner.py` 文件供参考：
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
from nautobot.extras.models.secrets import Secret


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
            username=Secret.objects.get(name="ARISTA_USERNAME").get_value(),  
            password=Secret.objects.get(name="ARISTA_PASSWORD").get_value(),
        )
        for command in commands:
            output = net_connect.send_command(
                command
            )  
            self.create_file(f"{device.name}-{command}.txt", output)



register_jobs(
    CommandRunner,
)
```

恭喜完成第 30 天的挑战！

## 第 30 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布使用 Nautobot Secret 成功执行 Job 的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将了解如何通过外部 API 调用来验证路由。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+30+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 30 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
