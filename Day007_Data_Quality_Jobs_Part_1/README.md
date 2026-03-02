# 数据质量Jobs

在今天的挑战中，我们将创建两个新的jobs来检查我们数据的质量。

## 实验室环境设置

环境设置将与[实验室设置场景1](../Lab_Setup/scenario_1_setup/README.md)相同，以下是步骤的总结，如果需要详细背景，请参考指南。

> [!TIP]
> 如果你停止了Codespace环境并再次重新启动，但发现Docker守护程序停止工作，请按照设置指南中的步骤重建环境。

以下是启动Nautobot的步骤回顾：

```
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

我们已准备好创建新的job文件。

## Job文件创建

让我们在nautobot docker容器的`/opt/nautobot/jobs`下创建一个名为`data_quality_jobs.py`的新job（可以自由使用[Hello Jobs](../Day003_Hello_Jobs_Part_1/README.md)中指定的选项1）：

```
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash

root@32a27fa1f5a6:/opt/nautobot# cd jobs

root@32a27fa1f5a6:/opt/nautobot/jobs# touch data_quality_jobs.py

root@32a27fa1f5a6:/opt/nautobot/jobs# chown nautobot:nautobot data_quality_jobs.py
```

我们将使用此文件来处理今天挑战的其余部分。

## 验证平台已定义

让我们继续在`data_quality_jobs.py`中填充以下代码，之后我们将讨论文件中我们可能熟悉的部分：

```python
from nautobot.apps.jobs import MultiChoiceVar, Job, ObjectVar, register_jobs, StringVar, IntegerVar
from nautobot.dcim.models.locations import Location
from nautobot.dcim.models.devices import Device

name = "Data Quality Jobs Collection"


class VerifyPlatform(Job):

    location_to_check = ObjectVar(
        model=Location,
    )

    class Meta:
        name = "Check Platform is defined"
        has_sensitive_variables = False
        description = "Check Platform is defined for devices in selected location"

    def run(self, location_to_check):
        device_query = Device.objects.filter(location=location_to_check)

        for device in device_query:
            self.logger.info(
                "Checking the device %s for Platform specified.",
                device.name,
                extra={"object": device},
            )

            # Verify that the device has a platform set
            if device.platform is None:
                self.logger.fatal(f"{device} does not have platform set.")
                return

            else:
                self.logger.debug(
                    "Device %s is of the platform: %s",
                    device.name,
                    device.platform,
                    extra={"object": device},
                )

register_jobs(
    VerifyPlatform
)
```

我们首先注意到`Meta`类中的额外属性。它们是相当不言自明的，但我们也可以参考[文档](https://docs.nautobot.com/projects/core/en/stable/development/jobs/#class-metadata-attributes)以获得更详细的信息：

```python
    class Meta:
        name = "Check Platform is defined"
        has_sensitive_variables = False
        description = "Check Platform is defined for devices in selected location"
```

我们还可以看到我们使用`ObjectVar`来选择位置，然后使用对象过滤器来过滤所选位置内的设备：

```python
class VerifyPlatform(Job):

    location_to_check = ObjectVar(
        model=Location,
    )
    ...
    def run(self, location_to_check):
        device_query = Device.objects.filter(location=location_to_check)
    ...
```

当我们循环遍历设备时，在日志中，我们使用`extra={"object": device}`来在日志中提供指向设备的链接：

```python
        for device in device_query:
            self.logger.info(
                "Checking the device %s for Platform specified.",
                device.name,
                extra={"object": device},
            )
```

> [!TIP]
> 不要忘记通过打开新的终端窗口并运行`invoke post-upgrade`命令来注册job。

一旦我们启用并运行了job，我们可以从位置列表中选择Boston：

![platform_check_1](images/platform_check_1.png)

然后我们可以看到该位置中具有平台的设备列表：

![platform_check_2](images/platform_check_2.png)

## 查询参数

但是，我们注意到一些位置没有设备，例如Baltimore。当我们为Baltimore运行该job时，它将成功，但由于没有设备，结果将为空：

![platform_check_3](images/platform_check_3.png)

我们可以使用的是添加一个`query_params`属性，以仅限制有设备的位置显示在列表中：

```python
class VerifyPlatform(Job):

    location_to_check = ObjectVar(
        model=Location,
        query_params={
            "has_devices": True,
        }
    )
```

现在，在Job运行页面上，只有有设备的位置才会显示：

![platform_check_4](images/platform_check_4.png)

`query_params`是一个非常强大的功能，我们可以使用REST API端点中可用的任何参数。例如，我们如何找出我们是否可以使用"has_devices"参数来仅限制有设备的位置？我们可以使用API文档。

让我们通过向下滚动到页面页脚并单击API链接来转到API文档：

![query_params_1](images/query_params_1.png)

然后我们可以向下滚动到位置端点：

![query_params_2](images/query_params_2.png)

然后我们可以看到有一个"has_devices"布尔参数，我们可以使用：

![query_params_3](images/query_params_3.png)

我们可以遵循相同的代码模式来检查我们数据的更多参数。

## 验证其他参数

遵循相同的模式，我们可以检查其他字段，例如序列号、主IP等。

由于这些额外字段中没有添加新的逻辑，代码将在此处作为参考列出：

```python
from nautobot.apps.jobs import MultiChoiceVar, Job, ObjectVar, register_jobs, StringVar, IntegerVar
from nautobot.dcim.models.locations import Location
from nautobot.dcim.models.devices import Device

name = "Data Quality Jobs Collection"

class VerifySerialNumber(Job):

    location_to_check = ObjectVar(
        model=Location,
        query_params={
            "has_devices": True,
        }
    )

    class Meta:
        name = "Check Serial Numbers"
        has_sensitive_variables = False
        description = "Check serial numbers exist for devices in the selected location"

    def run(self, location_to_check):
        device_query = Device.objects.filter(location=location_to_check)

        for device in device_query:
            self.logger.info(
                "Checking the device %s for a serial number.",
                device.name,
                extra={"object": device},
            )
            if device.serial == "":
                self.logger.error(
                    "Device %s does not have serial number defined.",
                    device.name,
                    extra={"object": device},
                )
            else:
                self.logger.debug(
                    "Device %s has serial number: %s",
                    device.name,
                    device.serial,
                    extra={"object": device},
                )


class VerifyPrimaryIP(Job):

    location_to_check = ObjectVar(
        model=Location,
        query_params={
            "has_devices": True,
        }
    )

    class Meta:
        name = "Verify Device has at selected location has Primary IP configured"
        has_sensitive_variables = False
        description = "Check Device at selected location Primary IP configured"

    def run(self, location_to_check):
        device_query = Device.objects.filter(location=location_to_check)

        for device in device_query:
            self.logger.info(
                "Checking the device %s for Primary IP.",
                device.name,
                extra={"object": device},
            )

            # Verify that the device has a primary IP
            if device.primary_ip is None:
                self.logger.fatal(f"{device} does not have a primary IP address configured.")
                return

            else:
                self.logger.debug(
                    "Device %s has primary IP: %s",
                    device.name,
                    device.primary_ip,
                    extra={"object": device},
                )


class VerifyPlatform(Job):

    location_to_check = ObjectVar(
        model=Location,
        query_params={
            "has_devices": True,
        }
    )

    class Meta:
        name = "Check Platform is defined"
        has_sensitive_variables = False
        description = "Check Platform is defined for devices in selected location"

    def run(self, location_to_check):
        device_query = Device.objects.filter(location=location_to_check)

        for device in device_query:
            self.logger.info(
                "Checking the device %s for Platform specified.",
                device.name,
                extra={"object": device},
            )

            # Verify that the device has a platform set
            if device.platform is None:
                self.logger.fatal(f"{device} does not have platform set.")
                return

            else:
                self.logger.debug(
                    "Device %s is of the platform: %s",
                    device.name,
                    device.platform,
                    extra={"object": device},
                )

register_jobs(
    VerifySerialNumber,
    VerifyPrimaryIP,
    VerifyPlatform
)
```

请确保从终端使用`invoke post-upgrade`注册我们的两个新jobs，以便它们显示在Nautobot中。

启用jobs后，我们可以运行它们并查看结果。在这种情况下，Boston设备确实缺少序列号：

![other_checks_1](images/other_checks_1.png)

我们在今天的挑战中做了很多，事实上我们只需几行代码就可以访问job中的数据库模型，这对我们的自动化工作真的大有裨益。

> [!TIP] 
> 如果遇到任何问题，这里有一个[第7天视频演练](https://www.youtube.com/watch?v=LwLOjt9j1SI&list=PLAaTeRWIM_wtqWt3yIKmIFnfEarsuxaYs&index=2)。

## 第7天待办事项

记得在[https://github.com/codespaces/](https://github.com/codespaces/)上停止代码空间实例。

继续在你选择的任何社交媒体上发布新创建的jobs的屏幕截图，确保你使用标签`#100DaysOfNautobot` `#JobsToBeDone`并标记`@networktocode`，这样我们可以分享你的进展！

在明天的挑战中，我们将继续构建我们的数据质量jobs，明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+7+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/) (Copy & Paste: I just completed Day 7 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot)
