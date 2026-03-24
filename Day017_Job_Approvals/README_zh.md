# Job 审批

Job 审批允许您创建需要他人审批才能执行的 Job，配置过程非常简单直接。

在今天的挑战中，我们将通过一个示例了解 Job 审批机制。

## 环境配置

环境配置与 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md) 相同，以下是步骤摘要，如需详细背景说明请参阅该指南。

> [!TIP]
> 如果您停止了 Codespace 环境后重新启动，发现 Docker 守护进程无法正常工作，请按照配置指南中的步骤重建环境。
>
> 此外，在已构建好容器且已导入数据库的重启环境中，只需执行 `poetry shell` 启动 Poetry 环境，再使用 `invoke debug` 启动容器即可。

按照以下步骤启动 Nautobot：
```sh
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

我们将以之前创建的 `hello_jobs.py` 文件为基础，如果 `/jobs` 目录中不存在该文件，请先创建：
```python
from nautobot.apps.jobs import Job, register_jobs

class HelloJobs(Job):
    def run(self):
        self.logger.debug("Hello, this is my first Nautobot Job.")

register_jobs(
    HelloJobs,
)
```

今天挑战的环境已配置完毕。

## 通过 UI 配置 Job 审批

可以通过 Web UI 的 `edit` 按钮设置 Job 属性以要求审批：

![job_approval_1](images/job_approval_1.png)

在 `properties` 部分可以设置 `approval required` 属性：

![job_approval_2](images/job_approval_2.png)

属性修改后，`Run Job Now` 按钮将变更为 `Request to Run Job Now`：

![job_approval_3](images/job_approval_3.png)

可以在审批队列中对 Job 进行 `Dry Run`（试运行）、`Approve`（批准）或 `Deny`（拒绝）操作：

![job_approval_4](images/job_approval_4.png)

接下来，让我们在 UI 中还原审批设置，并了解如何在代码中强制要求审批。

## 在代码中配置 Job 审批

在代码中为 Job 配置审批要求极为简单——只需在 `Meta` 类中添加 `approval_required = True` 即可！

我们将创建一个名为 `HelloJobsWithApproval` 的新 Job。由于需要审批的 Job *不能*包含敏感变量，我们同时将 `has_sensitive_variables` 设置为 `False`：
```python
...
class HelloJobsWithApproval(Job):
    class Meta: 
        name = "Hello World with Approval Required"
        approval_required = True
        has_sensitive_variables = False

    def run(self):
        self.logger.debug("Hello, this is my first Nautobot Job that requires approval.")

register_jobs(
    HelloJobs,
    HelloJobsWithApproval,
)
```

效果与通过 Web UI 修改配置完全相同。

## 延伸思考

由于今天的挑战相对简单，在此补充两个关于审批机制的思考题：

1. 您可以审批自己提交的 Job 吗？
2. 如何以及在哪里实现对象级别的审批权限？

欢迎在今天的成果分享中附上您对以上问题的解答。

## 第 17 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布今天挑战的解答，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将使用 Git 仓库来共享我们的 Job。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+17+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 17 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
