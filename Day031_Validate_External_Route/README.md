# 通过 API 调用设备验证外部路由

在之前的挑战中，我们已经接触过应用程序编程接口（API）。今天，我们将继续探索 Nautobot 的多样性，通过向外部网络设备发起 API 调用，检查某条远程路由是否存在于其路由表中。

在本次挑战中，我们将使用 `requests` 库——一个功能强大且易用的 Python HTTP 请求工具，它让与 API 的交互变得无缝顺畅，轻松实现数据的发送和接收。如果您不熟悉 `requests` 或想深入了解，请参阅 [Requests: HTTP for Humans™](https://requests.readthedocs.io/en/latest/)。

让我们开始配置今天挑战的环境吧！

## 环境配置

环境配置与 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md) 类似，以下是简要步骤摘要，如需更多细节请参阅该指南。

> [!NOTE]
> 如果您停止了 Codespace 环境后重新启动，发现 Docker 守护进程无法正常工作，请按照配置指南中的步骤重建环境。
```shell
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

> [!NOTE]
> 如果只是从前几天重新启动 Codespace 和 Nautobot，可以跳过 ```invoke build``` 和 ```invoke db-import```。

今天的挑战还需要用到 cEOS 虚拟实验室。以下是部署拓扑的快速步骤，如需虚拟实验室配置方面的帮助，请参阅挑战的[第 9 天](../Day009_Python_Script_to_Jobs_Part_1/README.md)。

> [!TIP]
> 此步骤为可选！到目前为止，我们一直只使用波士顿设备启动 Containerlab 拓扑。如果想用更多设备测试 Job，可以在 `ceos-lab.clab.yml` 文件中启用纽约设备，但记得为设备添加 IP 地址并分配主 IP！如需帮助请参阅[第 10 天](../Day010_Python_Script_to_jobs_Part_2/README.md)。同时记得使用更大规格的 Codespace 实例以获得更多硬件资源。
```yml
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

    nyc-acc-01:
      kind: "ceos"
      startup-config: "startup-configs/nyc-acc-01.conf"

    nyc-rtr-01:
      kind: "ceos"
      startup-config: "startup-configs/nyc-rtr-01.conf"

  links:
    - endpoints: ["bos-acc-01:eth1", "bos-rtr-01:eth1"]
    - endpoints: ["bos-acc-01:eth2", "nyc-rtr-01:eth2"]
    - endpoints: ["bos-rtr-01:eth2", "nyc-acc-01:eth2"]
    - endpoints: ["nyc-acc-01:eth1", "nyc-rtr-01:eth1"]
```

如果只想使用波士顿设备，请继续执行以下步骤：
```bash
➜ ~ $ docker import cEOS64-lab-4.32.0F.tar ceos:4.32.0F
➜ ~ $ cd 100-days-of-nautobot/clab/
➜ ~/100-days-of-nautobot/clab (main) $ sudo containerlab deploy --topo ceos-lab.clab.yml --node-filter bos-acc-01,bos-rtr-01
```

🌟 我们的环境已就绪，可以开始下一个挑战了！🌟

## 操作步骤

首先，为今天的挑战创建 Job 文件。
```bash
$ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash
root@c9e0fa2a45a0:/opt/nautobot# cd jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# pwd
/opt/nautobot/jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# touch remote_route_api.py
root@c9e0fa2a45a0:/opt/nautobot/jobs# chown nautobot:nautobot remote_route_api.py
```

文件创建后，在左侧 Docker 控制台中打开文件并进行编辑。

![Create File](images/remote_route_1.png)

首先导入 Job 所需的所有模块，包括 ```Job```、```register_jobs``` 和 ```ObjectVar```。由于我们需要确定特定设备和位置的远程路由，还需导入 ```Device``` 和 ```Location``` 来访问现有数据模型。
```python
import requests
from nautobot.apps.jobs import Job, ObjectVar, register_jobs
from nautobot.dcim.models import Device, Location
```

接下来，声明所有变量和 Meta 类用于 Job 描述：
```python
name = "API Requests"


class RemoteRouteAPI(Job):
    class Meta:
        name = "Remote Route API"
        has_sensitive_variables = False
        description = "Make API calls to retrieve routing table from a device using the requests library"

    # 定义设备位置和设备选择的 ObjectVar
    device_location = ObjectVar(
        model=Location, 
        required=False
    )
    
    device = ObjectVar(
        model=Device,
        query_params={
            "location": "$device_location",
        },
    )
```

正如我们所学，Nautobot 依赖 ```run()``` 方法执行 Job，并将设备及其位置作为参数传入。此外，在 ```run()``` 方法中，我们将在 Job 执行前加入日志记录和数据验证，以提升功能性。
```python
    def run(self, device, device_location):
        self.logger.info(f"Checking all routes for {device.name}.")

        # 验证设备是否有主 IP
        if device.primary_ip is None:
            self.logger.fatal(f"Device '{device.name}' does not have a primary IP address set.")
            return

        # 验证设备是否关联了平台
        if device.platform is None:
            self.logger.fatal(f"Device '{device.name}' does not have a platform set.")
            return
```

如果您熟悉 ```requests``` 库，就知道发起 API 调用时需要传入几个基本元素：

1. 由 ```url``` 表示的 API 端点。
2. 包含实际命令的 ```payload```（负载），数据结构根据设备要求而定。
3. 允许远程执行设备命令的有效 ```auth```（认证）凭据。

要验证网络设备上的远程路由，需要在负载中发送 ```show ip route``` 命令（或根据平台使用等效命令）。

> [!TIP]
> 如需了解更多 API 构建方式，请参阅 [ARISTA](https://arista.my.site.com/AristaCommunity/s/article/arista-eapi-101?utm_source=chatgpt.com) 和 [CISCO](https://www.cisco.com/c/en/us/td/docs/routers/csr1000/software/restapi/restapi/RESTAPIipinterface.html?utm_source=chatgpt.com) 官方文档。

> [!NOTE]
> API 调用方式可能因环境而异。在本实验中，只需向 API 端点发送基本认证信息和负载即可。
```python
        # 使用设备的主 IP 地址构造 API URL
        url = f"https://{str(device.primary_ip).split('/')[0]}/command-api"
        
        # 根据设备平台确定命令
        command_map = {
            "cisco_ios": "show ip route",
            "arista_eos": "show ip route",
            "juniper_junos": "show route"
        }
        
        platform_name = device.platform.network_driver
        cmd = command_map[platform_name]        
             
        # 根据设备类型定义 API 调用负载
        payload = {
            "jsonrpc": "2.0",
            "method": "runCmds",
            "params": {
                "version": 1,
                "cmds": [cmd],  # 命令必须以列表形式传入
                "format": "json"
            },
            "id": 1
        }

        # 设置基本认证
        auth = ("admin", "admin")  # 如有需要请替换为实际凭据
```

现在使用 ```requests``` 库发起实际的 API 请求。

注意我们使用 ```requests.post``` 作为请求方法。在其他场景中，根据不同操作，可能需要替换为 ```.get```、```post```、```put``` 或 ```delete``` 来执行基本的增删改查（CRUD）操作。

最后，添加连接错误处理，以便了解设备是否可达，帮助我们尽早发现问题并优雅地处理异常！

> [!IMPORTANT]
> 温馨提醒！😊 完成 Job 编写后，别忘了最后一步——使用 `register_jobs()` 注册 Job，然后运行 ```invoke post-upgrade```。
```python
        # 发起 API 调用
        try:
            response = requests.post(url, json=payload, auth=auth, verify=False) 
            response.raise_for_status()  # 对 HTTP 错误抛出异常

            # 解析 JSON 响应并以日志形式输出
            route_data = response.json()
            self.logger.info(f"Routing table from {device.name}:\n{route_data}")

            return route_data  # 可根据需要修改为返回特定数据

        # 将错误信息同时作为返回值和日志条目输出
        except requests.exceptions.RequestException as e:
            self.logger.fatal(f"Error connecting to {device.name}. Device unreachable.")
            raise Exception(f"Error connecting to {device.name}: {e}")

# Nautobot 识别 Job 的必要步骤
register_jobs(
    RemoteRouteAPI
)
```

## 完整代码

到目前为止，完整代码应如下所示：
```python
import requests
from nautobot.apps.jobs import Job, ObjectVar, register_jobs
from nautobot.dcim.models import Device, Location


name = "API Requests"


class RemoteRouteAPI(Job):
    class Meta:
        name = "Remote Route API"
        has_sensitive_variables = False
        description = "Make API calls to retrieve routing table from a device using the requests library"

    # 定义设备位置和设备选择的 ObjectVar
    device_location = ObjectVar(
        model=Location, 
        required=False
    )
    
    device = ObjectVar(
        model=Device,
        query_params={
            "location": "$device_location",
        },
    )

    def run(self, device_location, device):
        self.logger.info(f"Checking all routes for {device.name}.")

        # 验证设备是否有主 IP
        if device.primary_ip is None:
            self.logger.fatal(f"Device '{device.name}' does not have a primary IP address set.")
            return

        # 验证设备是否关联了平台
        if device.platform is None:
            self.logger.fatal(f"Device '{device.name}' does not have a platform set.")
            return

        # 构造 API URL
        url = f"https://{str(device.primary_ip).split('/')[0]}/command-api"
        
        # 根据设备平台确定命令
        command_map = {
            "cisco_ios": "show ip route",
            "arista_eos": "show ip route",
            "juniper_junos": "show route"
        }
        
        platform_name = device.platform.network_driver
        cmd = command_map[platform_name]        
             
        # 根据设备类型定义 API 调用负载
        payload = {
            "jsonrpc": "2.0",
            "method": "runCmds",
            "params": {
                "version": 1,
                "cmds": [cmd],  # 命令必须以列表形式传入
                "format": "json"
            },
            "id": 1
        }

        # 设置基本认证
        auth = ("admin", "admin")  # 如有需要请替换为实际凭据

        # 发起 API 调用
        try:
            response = requests.post(url, json=payload, auth=auth, verify=False) 
            response.raise_for_status()  # 对 HTTP 错误抛出异常

            # 解析 JSON 响应并以日志形式输出
            route_data = response.json()
            self.logger.info(f"Routing table from {device.name}:\n{route_data}")

            return route_data  # 可根据需要修改为返回特定数据

        # 将错误信息同时作为返回值和日志条目输出
        except requests.exceptions.RequestException as e:
            self.logger.fatal(f"Error connecting to {device.name}. Device unreachable.")
            raise Exception(f"Error connecting to {device.name}: {e}")


# Nautobot 识别 Job 的必要步骤
register_jobs(
    RemoteRouteAPI
)
```

现在可以运行 Job 了！🚀

![Remote Route Job Without Target IP](images/remote_route_2.png)

以下是执行结果！

![Remote Route Job Result With All Routes](images/remote_route_3.png)

> [!TIP]
> 如果连接设备时遇到问题，请参阅[第 9 天](../Day009_Python_Script_to_Jobs_Part_1/README.md)和[第 10 天](../Day010_Python_Script_to_Jobs_Part_2/README.md)的步骤，包括：
> 1. BOS 设备的 IP 前缀配置。
> 2. 设备 IP 地址是否已创建并分配给设备。
> 3. 设备的主 IP 是否已分配。
> 4. 是否在 Devices -> Platforms -> Arista EOS -> （编辑平台）-> Network Drivers 中指定了 `arista_eos`。

太棒了！到目前为止，我们已经成功连接到远程设备，发送了携带正确负载的 API 请求，并从其路由表中获取了数据。

但作为网络工程师，我们经常需要检查设备路由表中是否存在某个特定 IP 地址或路由——这在故障排查等场景中非常有用。那么，如何将目标 IP 地址或子网作为输入参数添加到我们的 Job 中呢？

要实现这一点，需要对代码做一些调整！

✅ 首先，在导入语句中添加 `StringVar`。

✅ 其次，使用 `StringVar` 引入一个输入变量，用于接收特定的目标 IP 或子网。

✅ 然后，修改负载，在 API 负载中加入目标 IP（如果有提供）。

✅ 最后，更新日志语句，使输出更加友好和信息丰富。

让我们开始吧！
```python
...
# 在导入语句中添加 StringVar
from nautobot.apps.jobs import Job, ObjectVar, StringVar, register_jobs


    ...
    # 要求用户输入目标 IP
    target_ip = StringVar(
        description = "Enter destination IP or remote route. Shows all routes available if left blank.", 
        required = False
    )

    ...
    # 日志语句区分是查询特定路由还是查询所有路由
    def run(self, device_location, device, target_ip):
        if target_ip:
            self.logger.info(f"Checking if {device.name} has a route to {target_ip}.")
        else:
            self.logger.info(f"Checking all routes for {device.name}.")
```

> [!NOTE]
> 注意我们将 `target_ip` 设置为 ```required = False``` 以提供灵活性，这样无论用户是否指定特定路由，Job 都能正常运行。

修改负载只需添加一个 if/else 语句。如果提供了目标 IP，代码会将其追加到基础命令 ```show ip route``` 之后；如果该字段留空，则只发送基础命令到设备。
```python
        ...        
        base_cmd = command_map.get(platform_name) 

        # 如果提供了目标 IP，则将其追加到 "show ip route" 命令
        if target_ip:
            cmd = f"{base_cmd} {target_ip}"
        else:
            cmd = base_cmd
             
        # 根据设备类型构造 API 调用负载
        payload = {
            "jsonrpc": "2.0",
            "method": "runCmds",
            "params": {
                "version": 1,
                "cmds": [cmd],  # 命令必须以列表形式传入
                "format": "json"
            },
            "id": 1
        }    
```

包含所有修改的最终完整代码如下：
```python
import requests
from nautobot.apps.jobs import Job, ObjectVar, StringVar, register_jobs
from nautobot.dcim.models import Device, Location


name = "API Requests"


class RemoteRouteAPI(Job):
    class Meta:
        name = "Remote Route API"
        has_sensitive_variables = False
        description = "Make API calls to retrieve routing table from a device using the requests library"

    # 定义设备位置和设备选择的 ObjectVar
    device_location = ObjectVar(
        model=Location, 
        required=False
    )
    
    device = ObjectVar(
        model=Device,
        query_params={
            "location": "$device_location",
        },
    )

    target_ip = StringVar(
        description = "Enter destination IP or remote route. Shows all routes available if left blank.", 
        required = False
    )

    def run(self, device_location, device, target_ip):
        if target_ip:
            self.logger.info(f"Checking if {device.name} has a route to {target_ip}.")
        else:
            self.logger.info(f"Checking all routes for {device.name}.")

        # 验证设备是否有主 IP
        if device.primary_ip is None:
            self.logger.fatal(f"Device '{device.name}' does not have a primary IP address set.")
            return

        # 验证设备是否关联了平台
        if device.platform is None:
            self.logger.fatal(f"Device '{device.name}' does not have a platform set.")
            return

        # 构造 API URL
        url = f"https://{str(device.primary_ip).split('/')[0]}/command-api"
        
        # 根据设备平台确定命令
        command_map = {
            "cisco_ios": "show ip route",
            "arista_eos": "show ip route",
            "juniper_junos": "show route"
        }
        
        platform_name = device.platform.network_driver
        base_cmd = command_map.get(platform_name)

        # 如果提供了目标 IP，则将其追加到 "show ip route" 命令
        if target_ip:
            cmd = f"{base_cmd} {target_ip}"
        else:
            cmd = base_cmd
             
        # 根据设备类型构造 API 调用负载
        payload = {
            "jsonrpc": "2.0",
            "method": "runCmds",
            "params": {
                "version": 1,
                "cmds": [cmd],  # 命令必须以列表形式传入
                "format": "json"
            },
            "id": 1
        }    
        
        # 设置基本认证
        auth = ("admin", "admin")  # 如有需要请替换为实际凭据

        # 发起 API 调用
        try:
            response = requests.post(url, json=payload, auth=auth, verify=False) 
            response.raise_for_status()  # 对 HTTP 错误抛出异常

            # 解析 JSON 响应并以日志形式输出
            route_data = response.json()
            self.logger.info(f"Routing table from {device.name}:\n{route_data}")

            return route_data  # 可根据需要修改为返回特定数据

        # 将错误信息同时作为返回值和日志条目输出
        except requests.exceptions.RequestException as e:
            self.logger.fatal(f"Error connecting to {device.name}. Device unreachable.")
            raise Exception(f"Error connecting to {device.name}: {e}")


# Nautobot 识别 Job 的必要步骤
register_jobs(
    RemoteRouteAPI
)
```

运行我们改进后的 Job 吧！🚀

![Remote Route Job With Target IP](images/remote_route_4.png)

如果目标路由存在于路由表中，您应该能在 ```{routes}``` 下看到相应条目（或类似内容）。

![Remote Route Job Result With Specific Route](images/remote_route_5.png)

如果路由不存在，预期输出是什么样的？让我们用一个不存在的目标地址运行 Job，看看输出结果。

![Remote Route Job With Non-existent Target IP](images/remote_route_6.png)

如果目标地址不在路由表中，我们将看到空的 ```{routes}```。

![Remote Route Job Result for Non-existent Route](images/remote_route_7.png)

## 附加挑战

作为额外挑战，不妨尝试修改代码以返回不同的信息内容？由于接收到的数据为 JSON 格式，可以尝试修改逻辑来解析数据并提取特定关注项。例如，检查某些路由是否存在，并返回一个布尔值来表示其是否存在。

## 第 31 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布新 Job 成功执行的截图，告诉我们您是如何修改代码以满足其他需求的！记得使用标签 `#100DaysOfNautobot` 和 `#JobsToBeDone`，并 @ `@networktocode`，让我们一起分享您的进展！

在下一个挑战中，我们将通过更多 API 调用从设备中获取常见漏洞与暴露（CVE）信息。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+31+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 31 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
