# Python脚本转Job - 第1部分 Netmiko `show command` Python脚本

在今天的挑战中，我们将把学到的关于Nautobot Jobs的知识应用到网络实验室设备中。

在许多未来的挑战中，我们需要有一个网络实验室供Nautobot实例与之交互。今天挑战的目标之一是使用[Containerlab](https://containerlab.dev/)在Codespace中设置一个实验室环境。

> [!IMPORTANT]
> 如果你有足够的Codespace积分或不介意支付费用以获得更好的体验，你应该使用更多核心来启动Codespace实例。多少？我会从`4核心`开始。

![codespace_machine_types](images/codespace_machine_types.png)

正如我们在上面的`重要`部分中所述，如果我们向Codespace环境添加更多容器，我们可能会在启动过程中发现高CPU使用率。我已经使用`2核心`机器类型成功完成了实验室，但在CPU使用率正常化之前确实花了一些时间。

## 实验室环境设置

实验室环境设置的Nautobot部分将与[实验室设置场景1](../Lab_Setup/scenario_1_setup/README.md)相同。以下是步骤的总结。如果需要详细背景，请参考指南。

> [!TIP]
> 如果你停止了Codespace环境并再次重新启动，但发现Docker守护程序已停止工作，请按照设置指南中的步骤重建环境。

以下是启动Nautobot的步骤回顾：

```
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

让我们设置网络实验室。

## Containerlab

[Containerlab](https://containerlab.dev/)是使用网络操作系统容器构建虚拟实验室的一种方式。

作为我们设置的一部分，Containerlab可执行文件已经被安装。我们可以运行`version`命令作为测试安装是否成功的方式：

```
@ericchou1 ➜ ~ $ containerlab version
  ____ ___  _   _ _____  _    ___ _   _ _____ ____  _       _     
 / ___/ _ \| \ | |_   _|/ \  |_ _| \ | | ____|  _ \| | __ _| |__  
| |  | | | |  \| | | | / _ \  | ||  \| |  _| | |_) | |/ _` | '_ \ 
| |__| |_| | |\  | | |/ ___ \ | || |\  | |___|  _ <| | (_| | |_) |
 \____\___/|_| \_| |_/_/   \_\___|_| \_|_____|_| \_\_|\__,_|_.__/ 

    version: 0.57.3
     commit: 8c357f5a
       date: 2024-09-21T12:26:37Z
     source: https://github.com/srl-labs/containerlab
 rel. notes: https://containerlab.dev/rn/0.57/#0573
 ```

在下一步中，我们将下载要在我们的实验室中使用的Arista cEOS镜像。

## Arista cEOS

[Arista Networks](https://www.arista.com/en/)提供对其容器化EOS系统的免费下载，可与Containerlab一起使用。注册是免费的，但你需要使用商业电子邮件地址。

注册后，你可以通过"Support -> Software Download"下载镜像：

![arista_1](images/arista_1.png)

> [!TIP]
> 请也参阅[实验室设置场景1](../Lab_Setup/scenario_1_setup/README.md)以获取有关如何下载Arista cEOS镜像的说明

选择一个"cEOS Lab"软件镜像，在屏幕截图中，我们显示```cEOS64-lab-4-32.0F.tar.xz```：

![arista_2](images/arista_2.png)

一旦软件镜像被下载，右键单击Explorer区域并选择上传镜像：

![arista_3](images/arista_3.png)

根据你的互联网速度，上传时间可能需要几分钟。镜像上传后，使用以下命令将镜像导入Docker：

> [!IMPORTANT]
> 记住用匹配你下载的版本的版本号替换。

> [!WARNING]
> 在下面的示例中，文件已经解压缩，它没有`.xz`文件扩展名。如果需要，请包括扩展名。

```
docker import cEOS64-lab-4.32.0F.tar ceos:4.32.0F
```

现在我们已准备好启动我们的实验室！

## 启动Containerlab

我们在```100-days-of-nautobot```目录下准备了一个```clab```目录，其中包含实验室拓扑和启动配置：

```
@ericchou1 ➜ ~/100-days-of-nautobot-challenge (main) $ cd clab/
@ericchou1 ➜ ~/100-days-of-nautobot-challenge/clab (main) $ ls
ceos-lab.clab.yml  startup-configs
```

对于这个初始测试，我们可以注释掉`NYC`设备，仅启动`BOS`设备以节省时间和资源。

> [!IMPORTANT]
> 如果与本文档中使用的不同，请使用上一步中使用的相同镜像名称！

```yaml
---
name: "ceos-lab"
prefix: ""

mgmt:
  network: "bridge"

topology:
  kinds:
    ceos:
      image: "ceos:4.32.0F"

  nodes:
    bos-acc-01:
      kind: "ceos"
      startup-config: "startup-configs/bos-acc-01.conf"

    bos-rtr-01:
      kind: "ceos"
      startup-config: "startup-configs/bos-rtr-01.conf"

    # nyc-acc-01:
    #   kind: "ceos"
    #   startup-config: "startup-configs/nyc-acc-01.conf"

    # nyc-rtr-01:
    #   kind: "ceos"
    #   startup-config: "startup-configs/nyc-rtr-01.conf"

  links:
    - endpoints: ["bos-acc-01:eth1", "bos-rtr-01:eth1"]
    # - endpoints: ["bos-acc-01:eth2", "nyc-rtr-01:eth2"]
    # - endpoints: ["bos-rtr-01:eth2", "nyc-acc-01:eth2"]
    # - endpoints: ["nyc-acc-01:eth1", "nyc-rtr-01:eth1"]
```

> [!TIP]
> 作为替代方案，我们也可以在启动期间使用`--node-filter`命令：
> 
> ```
> $ cd clab/
> $ sudo containerlab deploy --topo ceos-lab.clab.yml --node-filter bos-acc-01,bos-rtr-01
> ```

让我们继续启动实验室：

```
@ericchou1 ➜ ~/100-days-of-nautobot-challenge/clab (main) $ sudo containerlab deploy --topo ceos-lab.clab.yml 
INFO[0000] Containerlab v0.57.3 started                 
INFO[0000] Parsing & checking topology file: ceos-lab.clab.yml 
WARN[0000] Unable to init module loader: stat /lib/modules/6.5.0-1025-azure/modules.dep: no such file or directory. Skipping... 
INFO[0000] Creating lab directory: /home/vscode/100-days-of-nautobot-challenge/clab/clab-ceos-lab 
INFO[0000] Creating container: "bos-acc-01"             
INFO[0000] Creating container: "bos-rtr-01"             
INFO[0001] Running postdeploy actions for Arista cEOS 'bos-rtr-01' node 
...
INFO[0120] Adding ssh config for containerlab nodes     
INFO[0121] 🎉 New containerlab version 0.59.0 is available! Release notes: https://containerlab.dev/rn/0.59/
Run 'containerlab version upgrade' to upgrade or go check other installation options at https://containerlab.dev/install/ 
+---+------------+--------------+--------------+------+---------+---------------+--------------+
| # |    Name    | Container ID |    Image     | Kind |  State  | IPv4 Address  | IPv6 Address |
+---+------------+--------------+--------------+------+---------+---------------+--------------+
| 1 | bos-acc-01 | a7a817e93157 | ceos:4.32.0F | ceos | running | 172.17.0.2/16 | N/A          |
| 2 | bos-rtr-01 | 67cdc82c7f46 | ceos:4.32.0F | ceos | running | 172.17.0.3/16 | N/A          |
+---+------------+--------------+--------------+------+---------+---------------+--------------+
```

注意显示的IP地址；在下一步中，我们将使用一个简单的Netmiko脚本来执行```show```命令。

## Netmiko Python脚本

Nautobot利用Kirk Byers创建的多厂商、开源[Netmiko](https://pynet.twb-tech.com/blog/netmiko-python-library.html)库与网络设备交互。因此，它已经包含在我们的Nautobot实例中。

让我们继续在Nautobot容器上使用```nbshell```尝试```netmiko```：

```
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke nbshell
```

我们可以遵循[Netmiko入门](https://pynet.twb-tech.com/blog/netmiko-python-library.html)部分所示的简单示例：

```
>>> from netmiko import ConnectHandler
>>> net_connect = ConnectHandler( 
...     device_type="arista_eos", 
...     host="172.17.0.2", 
...     username="admin", 
...     password="admin", 
... )

>>> net_connect
<netmiko.arista.arista.AristaSSH object at 0x7abfa8bbf610>

>>> net_connect.find_prompt()
'ceos-01>'
```

太棒了！我们可以与设备通信。让我们从设备获取```show version```输出：

```
>>> output = net_connect.send_command("show version")
>>> print(output)
Arista cEOSLab
Hardware version: 
Serial number: 7579A68E1C5B921AA01EA5A60E12FCD5
Hardware MAC address: 001c.7314.b2a8
System MAC address: 001c.7314.b2a8

Software image version: 4.32.0F-36401836.4320F (engineering build)
Architecture: x86_64
Internal build version: 4.32.0F-36401836.4320F
Internal build ID: e97bbe15-478c-45d1-84fa-332db23aef84
Image format version: 1.0
Image optimization: None

cEOS tools version: (unknown)
Kernel version: 6.5.0-1025-azure

Uptime: 4 minutes
Total memory: 8119864 kB
Free memory: 1232352 kB
```

我会让你重复对另一个设备```172.17.0.3```的相同步骤以测试实验室可达性。

在你完成今天的工作后，使用```destroy```命令关闭Containerlab：

```
$ cd clab/
$ sudo containerlab destroy --topo ceos-lab.clab.yml 
```

## 第9天待办事项

记得在[https://github.com/codespaces/](https://github.com/codespaces/)上停止代码空间实例。

继续在你选择的任何社交媒体上发布Netmiko show命令的输出，确保你使用标签`#100DaysOfNautobot` `#JobsToBeDone`并标记`@networktocode`，这样我们可以分享你的进展！

在明天的挑战中，我们将这个脚本集成到我们的Nautobot Job中。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+9+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/) (Copy & Paste: I just completed Day 9 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot)
