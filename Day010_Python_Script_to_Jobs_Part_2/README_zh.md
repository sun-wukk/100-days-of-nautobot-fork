# Python 脚本转 Jobs - 第 2 部分

在第 009 天的挑战中，我们将 Arista cEOS 镜像上传到 Codespace 环境，启动了 Containerlab，并使用 Netmiko 库测试了从 Nautobot 到设备的连通性。

在今天的挑战中，我们将把 Netmiko 命令转化为可以从 UI 执行的 Nautobot Job。

> [!WARNING]
> 请注意，在 `2核` Codespace 机器上 CPU 占用率会很高，您需要等待 CPU 恢复正常，或者使用更大的机器类型启动 Codespace，例如 `4核`。

> [!NOTE]
> 请在 Codespace 中启动 Nautobot 环境，以下是简化版命令：
> ```
> $ cd nautobot-docker-compose/
> $ poetry shell
> $ invoke build
> $ invoke db-import
> $ invoke debug
> ```
> 我们还需要启动 Container Lab，以下是上传后导入 Arista cEOS 镜像的命令：
> ```
> docker import cEOS64-lab-4.32.0F.tar ceos:4.32.0F
> ```
> Containerlab 拓扑文件位于 clab 目录下，本次挑战只需要 BOS 设备，可以注释掉 NYC 设备或使用 `--node-filter` 命令：
> ```
> $ cd clab/
> $ sudo containerlab deploy --topo ceos-lab.clab.yml
> ```

如果出现类似 `Error: containers ["bos-acc-01" "bos-rtr-01"] already exist. Add '--reconfigure' flag to the deploy command to first remove the containers and then deploy the lab` 的错误，这是由于实验未通过 `containerlab destroy` 命令正常关闭导致的。请按照错误信息提示使用 `sudo containerlab deploy --reconfigure --topo ceos-lab.clab.yml`。

接下来，我们来创建今天挑战所需的文件。

## Operations Job 文件

我们将在 Nautobot Docker 实例的 ```/opt/nautobot/jobs``` 目录下创建名为 ```operation_jobs.py``` 的文件：
```
$ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash
root@c9e0fa2a45a0:/opt/nautobot# cd jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# pwd
/opt/nautobot/jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# touch operation_jobs.py
root@c9e0fa2a45a0:/opt/nautobot/jobs# chown nautobot:nautobot operation_jobs.py
```

文件创建后，可以在主面板中打开并开始编辑：

![operation_jobs_file_1](images/operation_jobs_file_1.png)

我们将逐步构建这个 Job。

## IP 前缀与地址分配

为了与设备通信，我们需要让 Nautobot 知道设备的主 IP 地址。需要通过 "IPAM -> Prefixes" 创建 IP 前缀列表：

![ip_address_1](images/ip_address_1.png)

创建 "172.17.0.0/16" 前缀并将状态设置为 "active"：

![ip_address_2](images/ip_address_2.png)

前缀创建后，可以在 "IPAM -> IP Address -> Add New IP address" 下创建单个 IP 地址。在 "Global" 命名空间中创建 "172.17.0.2/32" 和 "172.17.0.3/32"，类型为 "Network"，状态为 "Active"：

![ip_address_3](images/ip_address_3.png)

注意此时新地址尚未分配：

![ip_address_4](images/ip_address_4.png)

通过 "DEVICES -> Devices" 进入设备详情页，将 IP 分配给设备：

![devices_1](images/devices_1.png)

在设备详情页中，找到 Interfaces 部分和 Management1 接口：

![devices_2](images/devices_2.png)

点击 "Edit interface" 按钮编辑管理接口：

![devices_3](images/devices_3.png)

最后，向下滚动到 IP 地址部分，分配之前创建的 IP 地址：

![devices_4](images/devices_4.png)

如果该字段中已有预存的管理 IP 地址，请先删除再添加新创建的地址。

完成后，返回设备详情页，将该 IP 设置为 "Primary IPv4" 地址：

![devices_5](images/devices_5.png)

对所有需要操作的设备重复上述步骤。

## 映射网络驱动

还有一件事需要完成，即告知 Nautobot 为 Arista EOS 平台使用哪个网络驱动。这是一次性配置，所有网络平台都需要执行一次。

通过 "DEVICES -> Platforms" 编辑 "Arista EOS" 平台：

![driver_1](images/driver_1.png)

使用 "Show Configured Choices" 查看所有选项。Arista 的网络驱动名称为 "arista_eos"：

![driver_2](images/driver_2.png)

现在可以开始构建我们的 Operations 脚本了。

## 构建脚本

第一步是导入必要的数据库模型和库。此处导入的部分库将在后续使用，提前导入不影响程序运行：
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
```

将 Nautobot Jobs 的名称设置为 "Network Operations"：
```python
name = "Network Operations"
```

在这里指定允许用户选择的所有命令选项：
```python
COMMAND_CHOICES = (
    ("show ip interface brief", "show ip int bri"),
    ("show ip route", "show ip route"),
    ("show version", "show version"),
    ("show log", "show log"),
    ("show ip ospf neighbor", "show ip ospf neighbor"),
)
```

> [!TIP]
> `COMMAND_CHOICES` 中的每个元组包含两个元素：
>
> 第一个元素是通过 CLI 发送给目标设备的实际命令；如果语法不正确，Job 会成功执行，但输出中会显示错误信息。
>
> 第二个元素是命令在 Job 下拉菜单中的显示名称，可以自定义（例如用 `"OSPF neighbors"` 替代 `"show ip ospf neighbor"`），但应具有描述性，便于使用平台的网络工程师理解。
>
> 在本示例中，为了清晰起见，第二个元素直接使用了实际命令语法。

以下是脚本的核心部分，大部分验证逻辑已在前面介绍过。我们在验证之后添加了 Netmiko 连接函数，并将结果输出到文件中：
```python
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
```

不要忘记注册 Job：
```python
register_jobs(
    CommandRunner,
)
```

> [!IMPORTANT]
> 别忘了执行 POST 更新，使新 Job 出现在 Nautobot UI 中。请使用 `CTRL+C` 停止 Nautobot 容器，然后运行以下命令：
> ```
> invoke post-upgrade
> invoke debug
> ```
> 这将运行 Django 迁移以注册 **Command Runner** Job 并重启 Nautobot。

启用 Job 并进行测试：

![command_runner_1](images/command_runner_1.png)

执行后可以看到命令已成功执行，输出已保存到可供下载的文件中：

![command_runner_2](images/command_runner_2.png)

输出文件包含 "show version" 命令的执行结果：

![command_runner_3](images/command_runner_3.png)


## 最终脚本

以下是脚本的最终版本：
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



register_jobs(
    CommandRunner,
)
```

今天的挑战收获颇丰，让我们花点时间庆祝一下吧！

## 第 10 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布新 Job 成功执行的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将通过 VLAN 变更来增强我们的 Operations Job。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+10+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 10 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
