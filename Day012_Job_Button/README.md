# Job Button - 第 1 部分

到目前为止，我们一直通过 Web 界面来启动 Job。有时候，直接从对象所在页面启动 Job 会更加方便。

例如，我们可能有一个用于重启设备接口端口的 Python 脚本。将其转换为 Nautobot Job 后，与其从 Jobs UI 触发该脚本，不如直接从接口页面触发 Job。我们可以通过 Job Button 来实现这一点。

![job_button_1](images/job_button_1.png)

创建 Job Button 分为两步：

第一步：创建 Job Button Receiver。
第二步：将其与 Job Button 关联。

在今天的挑战中，我们将创建一个简单的 Job Button。在明天的挑战中，我们将在此基础上创建端口重启 Job Button。

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

为今天的挑战创建文件，可以通过共享目录或直接在 Nautobot Docker 容器中操作：

![file_creation.png](images/file_creation.png)
```
$ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash
root@c9e0fa2a45a0:/opt/nautobot# cd jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# pwd
/opt/nautobot/jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# touch hello_world_job_button.py
root@c9e0fa2a45a0:/opt/nautobot/jobs# chown nautobot:nautobot hello_world_job_button.py
```

今天挑战的环境已配置完毕。

## 第一个 Job Button Receiver

我们要做的第一件事是创建一个 Job 文件，已在上一步环境配置中完成。

> [!NOTE]
> 本示例摘自 [Network Automation with Nautobot](https://www.packtpub.com/en-us/product/network-automation-with-nautobot-9781837634514a) 一书的第 11 章。

以下是 ```hello_world_job_button.py``` 的内容：
```
# hello_world_job_button.py

from nautobot.apps.jobs import Job, register_jobs, JobButtonReceiver


name = "Job Button Receivers"

class HelloWorldJobButton(JobButtonReceiver):

    class Meta: 
        name = "This is my first JobButton Receiver"
    
    def receive_job_button(self, obj): 
        self.logger.info("This is my first Nautobot Job Button.", extra={"object": obj})
        self.logger.info("This is my first Nautobot Job Button.", extra={"object": obj.name})
        self.logger.info("This is my first Nautobot Job Button.", extra={"object": obj.status})
        self.logger.info("This is my first Nautobot Job Button.", extra={"object": obj.role})

register_jobs(HelloWorldJobButton)
```

到目前为止，我们应该已经熟悉创建和注册 Job 的整体结构了。不过在这个文件中，我们创建的是 ```Job Button Receiver``` 而非普通 Job。其中 ```obj``` 是一个通用术语，表示我们将该 Receiver 关联到的任意对象。

创建 Job 后需要执行 `post-upgrade`：
```
$ invoke post-upgrade
```

现在可以将 Job Receiver 与对象进行关联了。

## 关联 Job Receiver

默认情况下，Job Button Receiver 不会显示在 Job UI 中。我们需要点击 Job 菜单中的筛选按钮：

![job_button_3](images/job_button_3.png)

然后在 "is job button receiver" 选项中选择 ```Yes```，再点击 "Apply"：

![job_button_4](images/job_button_4.png)

Job 显示出来后，还需要将其启用：

![job_button_2](images/job_button_2.png)

可以通过 Job 菜单中的 "+" 图标创建新的 Job Button：

![job_button_5](images/job_button_5.png)

将 Job Button 与 ```dcim|device``` 对象关联，并按如下所示填写其余选项：

![job_button_6](images/job_button_6.png)

创建完成后即可运行。

## 测试 Job Button

导航到任意设备页面，可以看到右上角出现了一个按钮：

![job_button_7](images/job_button_7.png)

确认执行后，页面会显示一个查看结果的链接：

![job_button_8](images/job_button_8.png)

点击链接可以查看 Job 日志。回顾一下，我们在日志消息中使用了 ```obj.name```、```obj.status``` 和 ```obj.role```，可以验证它们与当前对象的信息是否一致：

![job_button_9](images/job_button_9.png)

恭喜，成功完成了第一个 Job Button 的创建和运行！

## 第 12 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布新 Job Button 成功执行的截图，记得使用标签 `#100DaysOfNautobot` `#100DON` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将继续乘势而上，创建一个实用的 Job Button。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+12+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 12 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
