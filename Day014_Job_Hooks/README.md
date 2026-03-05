# Job Hooks 简介

[Job Hooks](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/jobs/jobhook/) 是 Nautobot 受 Webhook 启发而引入的概念。其核心思想是：当 Nautobot 中的数据发生变更时，自动触发一个 Job 并发出 API 调用。

在今天的挑战中，我们将创建一个简单的 Job Hook 示例。

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

本次挑战无需使用 Containerlab。

为今天的挑战创建文件，可以通过共享目录或直接在 Nautobot Docker 容器中操作：

![file_creation](images/file_creation.png)
```
$ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash
root@c9e0fa2a45a0:/opt/nautobot# cd jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# pwd
/opt/nautobot/jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# touch job_hook_test.py
root@c9e0fa2a45a0:/opt/nautobot/jobs# chown nautobot:nautobot job_hook_test.py
```

今天挑战的环境已配置完毕。

## Job Hook Receiver

与 Job Button Receiver 类似，第一步是创建 Job Hook Receiver。在 `job_hook_test.py` 文件中，编写以下内容：
```
from nautobot.apps.jobs import Job, register_jobs, JobHookReceiver
import requests 

name = "Job Hook Receivers"

class HelloWorldJobHook(JobHookReceiver):

    class Meta: 
        name = "This is my first Job Hook Receiver"
    
    def receive_job_hook(self, change, action, changed_object): 
        self.logger.info("Launching Job Hook Receiver.", extra={"object": changed_object})
        
        response = requests.get("https://httpbin.org/get")
        if response.status_code == 200:
            self.logger.info("Job Hook Launched.")

register_jobs(HelloWorldJobHook)
```

在 Receiver 中，我们使用 `receive_job_hook()` 方法响应数据对象的变更事件。注意传入的参数包括变更类型（创建/更新/删除）以及 `changed_object`（被变更的对象）。

创建 Job Hook Receiver 后需要执行 `post-upgrade`：
```
$ invoke post-upgrade
```

下一步将进行 Job Hook 的关联配置。

## 关联 Job Hook Receiver

与 Job Button Receiver 相同，可以使用筛选功能使 Job Hook Receiver 可见：

![job_hook_1](images/job_hook_1.png)

![job_hook_2](images/job_hook_2.png)

然后启用该 Job Hook：

![job_hook_3](images/job_hook_3.png)

点击 "Job Hooks" 旁边的 "+" 图标创建新的 Job Hook：

![job_hook_4](images/job_hook_4.png)

在本示例中，我们将其与 `dcim|device` 对象关联，这样每当设备发生创建/更新/删除事件时，该 Job Hook 都会被触发。

![job_hook_5](images/job_hook_5.png)

现在来测试我们全新的 Job Hook。

## 测试 Job Hook

通过创建一个测试设备来触发 Job Hook：

![job_hook_test_1](images/job_hook_test_1.png)

进入 "Job Results" 页面，应该可以看到 Job Hook 的执行结果：

![job_hook_test_2](images/job_hook_test_2.png)

非常好！希望您已经开始注意到 Nautobot Jobs 在结构和操作步骤上的规律性。

## 第 14 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布新 Job 成功执行的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将了解如何使用 Web API 与 Nautobot Jobs 进行交互。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+14+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 14 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
