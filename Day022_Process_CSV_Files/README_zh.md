# 处理上传的 CSV 文件

在今天的挑战中，我们将在昨天 ```FileVar``` 对象知识的基础上，进一步处理 CSV 文件，并利用其内容来操作 Nautobot 实例中的数据。

这是一个简单却强大的步骤，展示了如何使用外部信息与 Nautobot 数据模型进行交互。

## 环境配置

环境配置与 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md) 相同，以下是步骤摘要，如需详细背景说明请参阅该指南。

完整的 Nautobot 启动步骤如下。如果是重启 Codespace 实例，可以跳过 `invoke build` 和 `invoke db-import`，因为容器已经构建完成：
```
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

今天挑战的环境已配置完毕。同时，请确保已准备好第 21 天创建的 CSV 文件。

## 现有 Job

首先，让我们再看一下上次创建的 Job。这里没有新内容，只需确保已启用该 Job 且可以正常运行即可。
```
from nautobot.apps.jobs import Job, register_jobs, FileVar

class FileUpload(Job):
    class Meta:
        name = "CSV File Upload"
        description = "Please select a CSV file for upload"

    file = FileVar(
        description="CSV File to upload",
    )

    def run(self, file):
        
        contents = str(file.read())
        self.logger.info(f"File contents: {contents}")
        self.logger.info(f"Job didn't crash!")

        return "Great job!"


register_jobs(
    FileUpload,
)
```

## 更新后的 Job

当前 Job 读取文件后将内容发送至日志，让用户确认文件已被处理。但这本身并不太实用。我们希望 Job 能够读取数据、解析数据，并在 Nautobot 实例中创建新对象。我们知道文件是文本格式，且按 CSV（逗号分隔值）格式组织。因此，现在对 Job 进行如下改进：
```
from nautobot.apps.jobs import Job, register_jobs, FileVar
from nautobot.dcim.models import Device, Location, DeviceType
from nautobot.extras.models import Role, Status


class FileUpload(Job):
    class Meta:
        name = "CSV File Upload"
        description = "Please select a CSV file for upload"

    file = FileVar(
        description="CSV File to upload",
    )

    def run(self, file):
        
        contents = str(file.read())
        self.logger.info(f"File contents: {contents}")
        self.logger.info(f"Job didn't crash!")

        return "Great job!"


class FileUpload_2(Job):
    class Meta:
        name = "CSV File Upload and Process"
        description = "Please select a CSV file for upload"

    file = FileVar(
        description="CSV File to upload",
    )

    def run(self, file):

        file_contents = file.read().decode("utf-8")
        self.logger.info(file_contents)
        lines = file_contents.splitlines()
        self.logger.info(lines)

        self.logger.info("Parsing of the lines")

        for line in lines[1:]:
            device_name, role_name, model_name, location_name = line.split(",")
            self.logger.info(f"Name: {device_name}")
            self.logger.info(f"Role: {role_name}")
            self.logger.info(f"Device type: {model_name}")
            self.logger.info(f"Location: {location_name}")


            role = Role.objects.get(name=role_name)
            device_type = DeviceType.objects.get(model=model_name)
            location = Location.objects.get(name=location_name)
            status = Status.objects.get(name="Active")

            device = Device(
                name=device_name,
                device_type=device_type,
                location=location,
                status=status,
                role=role,
            )
            device.validated_save()


        return "Execution completed"


register_jobs(
    FileUpload,
    FileUpload_2,
)
```

新 Job 包含了从文件读取数据并创建设备所需的逻辑，所有改动均在 `run` 方法中实现。这里有几个新元素，让我们逐一解析。

首先，所有包含 `self.logger.info` 的行都可以安全删除。这些日志的作用是帮助您直观地了解 Job 的执行过程，以及它是如何逐行遍历文件的。如果文件包含数千乃至数百万行，这类日志可能会逐渐积累并影响性能。执行 Job 时请关注这些日志，它们有助于加深您对执行过程的理解。

以下两行代码读取文件内容，并确保使用正确的编码格式。将文件内容存储到 `file_contents` 变量后，我们需要逐行处理。`lines` 变量通过 `splitlines` 方法创建，是一个列表，其中每个元素是文件中的一行，在我们的示例中每行存储一台设备的信息。
```
file_contents = file.read().decode("utf-8")
lines = file_contents.splitlines()
```

将每台设备的数据作为 `lines` 列表的一个元素后，我们将遍历该数据结构，并为文件中的每一行在 Nautobot 中创建一台新设备。我们跳过第一行，因为它是表头，不包含实际数据。
```
for line in lines[1:]:
    ...
```

最核心的部分是实际创建设备的代码：
```
role = Role.objects.get(name=role_name)
device_type = DeviceType.objects.get(model=model_name)
location = Location.objects.get(name=location_name)
status = Status.objects.get(name="Active")

device = Device(
    name=device_name,
    device_type=device_type,
    location=location,
    status=status,
    role=role,
)
device.validated_save()
```

这里，我们首先使用文件中的字符串来查找与角色、设备类型、位置和状态对应的 Nautobot 对象。对于状态，我们决定统一使用 `Active`。接着，创建一个新对象并存储在 `device` 变量中。该变量持有对象实例，但要将其保存到数据库中，还需要调用 `.validated_save()` 方法。

请尝试执行该 Job，Job 完成后进入设备视图，确认两台新设备已成功创建。

以下是 Job 输出示例：

![job_result_1](images/job_result_1.png)

可以验证设备已成功创建：

![new_devices](images/new_devices.png)

恭喜完成第 22 天的挑战！

# 延伸思考

在尝试这个示例时，您可能会发现一些潜在的改进点。例如，如果想以不同的方式读取文件该怎么做？您可能还会注意到，如果再次运行该 Job，将会产生错误，因为设备已经存在。您希望它有不同的行为吗？您会做哪些改动？在后续的挑战中，您将探索部分这样的情况，但也欢迎您自行尝试这些改动。

## 第 22 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布新 Job 成功执行的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将了解与 Nautobot Jobs 相关的 HTML 模板。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+22+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 22 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
