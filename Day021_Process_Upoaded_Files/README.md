# 处理上传的文件

如我们在前几天所见，用户向 Nautobot 提供信息的一种方式是使用 HTML 表单，这也是我们能够为 Nautobot Jobs 提供 IP 地址和位置等信息的方式。

另一种批量提供信息的方式是使用文件上传。当我们需要提供大量格式相似、仅有细微差异的信息时，这种方式尤为实用。

例如，可以参考 [Nautobot 设备上线 App](https://docs.nautobot.com/projects/device-onboarding/en/latest/)。该 App 旨在简化新设备接入 Nautobot 的流程，允许用户上传包含设备 IP 地址和位置信息的 CSV 文件，Nautobot 将据此连接设备并完成上线。

![device_onboarding_1](images/device_onboarding_1.png)

![device_onboarding_2](images/device_onboarding_2.png)

在今天的挑战中，我们将允许用户通过 [FileVar](https://docs.nautobot.com/projects/core/en/v2.3.9/development/jobs/#filevar) 上传自定义文件。

## 环境配置

环境配置与 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md) 相同，以下是步骤摘要，如需详细背景说明请参阅该指南。

如果您已停止并重新启动了 Codespace 实例，可以跳过 `invoke build` 和 `invoke db-import`，直接启动：
```
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke debug
```

否则，按照以下完整步骤在 Codespace 中启动 Nautobot：
```
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

今天挑战的环境已配置完毕。

## 数据文件

首先创建一个用于后续示例的简单文件。组织中的数据存储在文件中，或从其他系统导出为文件，这种情况十分常见。通常这些数据可能体量庞大且结构复杂，但我们先从简单的开始，后续可以随时上传更大的文件。

创建一个带有 CSV 扩展名的文本文件。CSV（逗号分隔值）文件是普通的文本文件，可以用您喜欢的文本编辑器打开。这类文件被广泛用于存储数据，我们也可以使用 TXT 等其他扩展名，但 CSV 会在某些编辑器中触发额外功能，因此我们选择使用 CSV。

文件内容：
```
name,role,model,location
sw-indianapolis,Switch,vEOS,Indianapolis
rt-indianapolis,Router,vEOS,Indianapolis
```

## 文件上传 Job

在 `jobs` 目录下创建名为 `file_upload.py` 的新文件，用于允许用户上传文本文件并读取其内容。

该 Job 流程十分简单，步骤如下：

1. 从 `nautobot.apps.jobs` 导入 `FileVar`。
2. 使用 `read()` 方法读取文件内容。
3. 记录输出日志。

以下是 `file_upload.py` 的内容：
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

> [!TIP]
> 别忘了执行 `invoke post-upgrade` 并在 GUI 中启用该 Job。

随着您不断深入并创建更复杂的 Job，您会发现有多种预定义变量类型可帮助您构建 Job 表单。可以通过以下[链接](https://docs.nautobot.com/projects/core/en/stable/development/jobs/?h=filevar#variables)查看当前支持的 Job 变量完整列表。

在我们的示例中，使用的是 `FileVar`。该变量允许您轻松接收用户上传的文件，并将文件传递给 Job 的 `run()` 方法。

Job 创建完成后，进入 Nautobot GUI 的 Job 菜单执行该 Job，记得先启用它。

以下是 Job 的执行日志：

![job_output_1](images/job_output_1.png)

注意在 `post_run` 阶段文件已被删除。文件加载到内存后即会被删除。

## （可选）文件类型检查

在我们的简单上传示例中，没有进行文件类型检查。在生产环境中，最好执行一些错误检查，以确保上传的文件类型符合预期。

## 第 21 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布新 Job 成功执行的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将在此基础上新增一个 Job，用于解析 CSV 文件并根据文件内容在 Nautobot 中创建对象。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+21+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 21 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
