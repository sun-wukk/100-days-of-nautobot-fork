# 具有用户输入的数据质量Jobs

在今天的job中，我们将从第007天的数据质量job基础上进行构建。今天挑战的主要学习目标是说明由于Nautobot Jobs基于Python，我们能够利用广泛的Python库来帮助编写Nautobot Jobs。

> [!NOTE] 
> 请在Codespace中启动Nautobot环境，这是一个精简版本的命令：
> ```
> $ cd nautobot-docker-compose/
> $ poetry shell
> $ invoke build
> $ invoke db-import
> $ invoke debug
> ```

提醒一下，在第007天我们在nautobot docker容器的```/opt/nautobot/jobs```下创建了一个名为```data_quality_jobs.py```的文件。以下是[第7天](../Day007_Data_Quality_Jobs_Part_1/README.md)的文件创建和文件内容的重复：

```
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash

root@32a27fa1f5a6:/opt/nautobot/jobs# touch data_quality_jobs.py
root@32a27fa1f5a6:/opt/nautobot/jobs# chown nautobot:nautobot data_quality_jobs.py
```

让我们继续粘贴从第007天开始的代码：

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

我们可以继续进行今天挑战的第一个新job。

## 验证位置中的主机名模式

假设我们想确保库存中的所有设备都符合命名标准。我们的主机名应该始终遵循以下模式：

```
<city airport code>-<device role>-<device number>.infra.valuemart.com
```

我们首先可以做的是构造一个我们将使用的正则表达式模式。学习、构建和验证正则表达式模式的好来源是[https://regex101.com/](https://regex101.com/)。

![regex101](images/regex101.png)

我们可以使用我们构建的正则表达式，并使用Python RE模块来构造一个匹配模式：

```python
import re
...

HOSTNAME_PATTERN = re.compile(r"[a-z0-1]+\-[a-z]+\-\d+\.infra\.valuemart\.com")

```

我们可以遵循相同的模式来选择一个位置，然后限制要检查的设备在该位置内：

```python
class VerifyHostname(Job):
    location_to_check = ObjectVar(
        model=Location,
        query_params={
            "has_devices": True,
        }
    )
    
    class Meta:

        name = "Verify Hostname Pattern For Selected Locations"
        description = "Checks all devices at Designated Location for hostname pattern conformation"

    def run(self, location_to_check):
        """Run method for executing the checks on the devices."""

        # Iterate through each Device object, limited to just the location of choice.
        for device in Device.objects.filter(location=location_to_check):
            hostname = device.name
            self.logger.info(
                f"Checking device hostname compliance: {hostname}",
                extra={"object": device},
            )
            # Check if the hostname matches the expected pattern
            if HOSTNAME_PATTERN.match(hostname):
                self.logger.info(f"{hostname} configured hostname is correct.")
                # Skip to next iteration of the list
                continue

            # Mark the Device as failed in the job results
            self.logger.error(f"{hostname} does Not Match Hostname Pattern.")
```

记住要注册job：

```python
register_jobs(
    VerifyHostname,
    VerifySerialNumber,
    VerifyPrimaryIP,
    VerifyPlatform
)
```

我们需要在单独的终端窗口中使用```invoke post-upgrade```注册新job：

```
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke post-upgrade
```

启用后，我们可以运行该job并检查结果：

![hostname_check_1](images/hostname_check_1.png)

这只是使用我们熟悉的普通Python代码来增强jobs功能的一个简单示例。正如我们稍后将看到的，我们将使用Python库，例如```csv```来读取CSV文件、```json```来解码和编码JSON正文，以及各种API库来进行出站调用。

> [!TIP] 
> 如果遇到任何问题，这里有一个[第8天视频演练](https://www.youtube.com/watch?v=XD0uBRh_4Y4&list=PLAaTeRWIM_wtqWt3yIKmIFnfEarsuxaYs&index=1)。


## 第8天待办事项

记得在[https://github.com/codespaces/](https://github.com/codespaces/)上停止并删除代码空间实例。

继续在你选择的任何社交媒体平台上发布主机名检查job成功的屏幕截图，确保你使用标签`#100DaysOfNautobot` `#JobsToBeDone`并标记`@networktocode`，这样我们可以分享你的进展！

在明天的挑战中，我们将回到增强我们的Nautobot Jobs。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+8+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/) (Copy & Paste: I just completed Day 8 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot)
