# Job Button - 第 2 部分：接口端口重启器

端口重启（即将端口先关闭再开启）是网络工程师最常见的操作之一。正如我们在昨天的挑战中所提到的，我们可以将这项任务"代码化"，并通过 Nautobot Job 来执行此操作。

一种方式是创建一个普通 Job，利用我们已学过的技术提供菜单来选择目标设备和端口。但更好的方案是使用昨天学习的 Job Button，将其直接绑定到接口对象上。

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

上传并准备 cEOS 镜像，然后启动 Containerlab：
```
$ docker import cEOS64-lab-4.32.0F.tar ceos:4.32.0F
```

本实验只需要 BOS 设备：
```
$ cd clab/
$ sudo containerlab deploy --topo ceos-lab.clab.yml --node-filter bos-acc-01,bos-rtr-01
```

为今天的挑战创建文件，可以通过共享目录或直接在 Nautobot Docker 容器中操作，文件命名为 `port_bounce_job_button.py`：

![file_creation](images/file_creation.png)
```
$ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash
root@c9e0fa2a45a0:/opt/nautobot# cd jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# pwd
/opt/nautobot/jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# touch port_bounce_job_button.py
root@c9e0fa2a45a0:/opt/nautobot/jobs# chown nautobot:nautobot port_bounce_job_button.py
```

今天挑战的环境已配置完毕。

## Job Button Receiver

与之前一样，我们需要在上一步创建的文件中编写 Receiver：
```
from nautobot.apps.jobs import Job, register_jobs, JobButtonReceiver
from netmiko import ConnectHandler


class PortBouncerButton(JobButtonReceiver):

    """通过 Netmiko 和 Job Button 重启端口。"""
   
    class Meta:
        name = "Bounce Interface ports"
        has_sensitive_variables = False
        description = "Bounce Interface Port"

    def receive_job_button(self, obj):
        self.logger.info("Running job button receiver.", extra={"object": obj})
        if obj.device.primary_ip is None:
            self.logger.fatal("Device does not have a primary IP address set.")
            return

        if obj.device.platform is None:
            self.logger.fatal("Device does not have a platform set.")
            return

        if obj.device.platform.network_driver_mappings.get("netmiko") is None:
            self.logger.fatal("Device mapping for Netmiko is not present, please set.")
            return

        
        # 连接设备并获取输出 - 如果是模拟模式请注释掉此部分
        net_connect = ConnectHandler(
            device_type=obj.device.platform.network_driver_mappings["netmiko"],
            host=obj.device.primary_ip.host,  # 如果设备名称是 FQDN，也可以使用 device.name
            username="admin",
            password="admin",
        )

        # 平台与设备命令的简单映射
        COMMAND_MAP = {
            "cisco_nxos": [f"interface {obj}", f"shut", f"no shut"],
            "arista_eos": [f"interface {obj}", f"shut", f"no shut",],
        }

        commands = COMMAND_MAP[obj.device.platform.network_driver_mappings.get("netmiko")]
        self.logger.info(f"This is the command: {commands}")
        net_connect.enable()
        net_connect.send_config_set(commands)
        net_connect.disconnect()

        self.logger.info(f"Successfully bounced port {obj} on {obj.device}!")
 

register_jobs(
    PortBouncerButton,
)
```

我们添加了几行代码，用于检查主 IP、驱动映射以及设备平台。然后使用 `netmiko` 驱动连接设备并执行命令。

注意，虽然 `cisco_nxos` 和 `arista_eos` 的端口重启命令相同，我们仍将它们分开配置。这样为将来支持使用不同命令集的其他平台预留了扩展空间。

创建 Job 后需要执行 `post-upgrade`：
```
$ invoke post-upgrade
```

接下来将 Job Button 与 Receiver 进行关联。

## 关联 Job Button

与 [第 12 天](../Day012_Job_Button/README.md) 相同，需要完成以下步骤：

1. 如果尚未操作，在 Jobs UI 中使用 `filter` 使 Job Button 可见。
2. 启用该 Job。
3. 通过 "+" 图标创建新按钮。

将此 Job 与 `dcim|interface` 对象关联：

![job_button_1](images/job_button_1.png)

现在可以进行测试了！

## 测试 Job Button

新按钮将出现在接口页面的右上角：

![job_button_2](images/job_button_2.png)

点击按钮并确认执行：

![job_button_3](images/job_button_3.png)

确认后将显示查看 Job 结果的链接：

![job_button_4](images/job_button_4.png)

之后可以像查看普通 Job 日志一样查看执行结果。

## 第 13 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布新 Job 成功执行的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将深入了解 Job Hooks。到时见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+13+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 13 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
