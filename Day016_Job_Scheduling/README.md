# Job 调度示例

Job 调度允许我们定期执行 Job，其功能类似于 `cron` 定时任务。让我们在今天的挑战中创建一个示例。

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

今天的挑战无需使用 Arista vEOS 镜像和 Containerlab。

今天挑战的环境已配置完毕。

## 创建 Job 文件

如果您保留了其他天的 Job 文件，可以直接使用。以下是我们在学习初期创建的 `hello_jobs.py` 文件：

![file_creation](images/file_creation.png)
```
from nautobot.apps.jobs import Job, register_jobs

class HelloJobs(Job):
    def run(self):
        self.logger.debug("Hello, this is my first Nautobot Job.")

register_jobs(
    HelloJobs,
)
```

如果这是一个新 Job，需要执行 `post-upgrade`：
```
$ invoke post-upgrade
```

如果该 Job 尚未启用，还需要先将其启用。

要使 Job 支持调度，不能包含敏感信息。在 Job 详情页（点击编辑按钮进入），需要覆盖 "Has sensitive variables" 的默认设置：

![job_scheduling_1](images/job_schedule_1.png)

修改完成后，在 Job 运行菜单中将出现调度选项：

![job_scheduling_2](images/job_schedule_2.png)

可以选择"在未来某一时间运行一次"，并指定日期和时间：

![job_scheduling_3](images/job_schedule_3.png)

Job 调度完成后，可以在 "Jobs -> Scheduled Jobs" 下查看：

![job_scheduling_4](images/job_schedule_4.png)

Job 调度是一个简单而强大的功能。如果您有时间，可以尝试使用现有的 "System Jobs" 中的 "Logs Cleanup"，或者了解 [Nautobot Golden Config App](https://docs.nautobot.com/projects/golden-config/en/latest/)，该应用使用 Job 调度来实现配置备份。

## 第 16 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布新 Job 成功调度的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将了解 Job 审批机制。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+16+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 16 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
