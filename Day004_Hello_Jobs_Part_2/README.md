# Hello Jobs - 第二部分: 自定义和特性

在今天的挑战中，我们将继续从[第3天](https://github.com/networktocode-llc/100-days-of-nautobot-challenge/blob/main/Day003_Hello_Jobs_Part_1/README.md)处理```hello_jobs.py```。

让我们导航到[https://github.com/codespaces/](https://github.com/codespaces/)并重启代码空间实例。

![rebuild_codespace_1.png](../Lab_Setup/lab_related_notes/images/rebuild_codespace_1.png)

由于似乎存在一个bug，Docker守护程序在重启Codespace时不会自动启动。为了解决这个问题，我们需要重建Codespace。

```
@ericchou1 ➜ ~ $ docker ps
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

## (Note) 重建Codespace

由于这是前两天的系列挑战，我们建议在第一天结束时停止Codespace，然后在第二天重新启动同一实例，让我们花几分钟时间讨论重建的问题。对于未来的几天，我们将只在[实验室笔记](../Lab_Setup/lab_related_notes/README.md)中提及它（并在提及时也会提及），但不会花太多时间详细讲解。一定要收藏实验室笔记！

点击左下角的设置齿轮图标，选择"Command Palette"：

![codespace_rebuild_1](images/codespace_rebuild_1.png)

在命令面板中，输入"Codespace: Rebuild Container"并选择该选项进行重建。在下一页确认。

![codespace_rebuild_2](images/codespace_rebuild_2.png)

一旦Codespace正常工作，使用终端窗口启动Nautobot，就像我们在第3天所做的那样。

## Jobs Meta Class

如果仔细查看我们创建的Job，你会注意到没有描述，而且部分分组名称是带下划线的'Hello_jobs'。如果我们将其与其他现有的系统Job进行比较，可以看到系统Job既有描述，也有作业名称之间的空格：

![jobs_meta_1](images/jobs_meta_1.png)

我们如何改变这一点？让我们参考[Nautobot Jobs开发者指南](https://docs.nautobot.com/projects/core/en/stable/development/jobs/#module-metadata-attributes)的元数据属性部分。

从文档中，我们可以看到可以使用```name```的全局常量来自定义分组名称，以及使用```meta```类来定义特定于Job的名称。让我们进行以下更改：

```python
from nautobot.apps.jobs import Job, register_jobs

# new
name = "Hello World Nautobot Jobs"

class HelloJobs(Job):

    # new
    class Meta:
        name = "Hello Jobs"
        description = "Hello World for first Nautobot Jobs"

    def run(self):
        self.logger.debug("Hello, this is my first Nautobot Job.")
    
    
register_jobs(
    HelloJobs,
)
```

> [!IMPORTANT]
> Register jobs是一个重要的步骤，很多人，包括我自己，在最初接触Nautobot jobs时可能会忽略它。不要忘记最后那一行。

保存更改后，UI上没有任何反应。问题可能是什么？

记住，我们需要执行```invoke post-upgrade```使更改生效：

```
@ericchou1 ➜ ~ $ cd nautobot-docker-compose/
@ericchou1 ➜ ~/nautobot-docker-compose (main) $ poetry shell
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke post-upgrade
```

现在当我们返回Nautobot Jobs UI时，可以看到更改已被应用：

![jobs_meta_2](images/jobs_meta_2.png)

让我们在下一部分看看如何向Job添加更多日志记录。

## 添加更多日志记录

在第3天，我们看到了如何使用```self.logger.debug("Hello, this is my first Nautobot Job.")```在代码中记录进度。如果我们想记录具有不同严重级别的更多数据该怎么办？

同样，[Nautobot Jobs开发者指南](https://docs.nautobot.com/projects/core/en/stable/development/jobs/#logging)是我们的好帮手。在日志记录下，我们可以使用logger对象轻松做到这一点。

让我们在同一文件中创建第二个具有更多日志记录的Job：

```python
class HelloJobsWithLogs(Job):

    class Meta:
        name = "Hello Jobs with Logs"
        description = "Hello Jobs with different log types"

    def run(self):
        self.logger.info("This is an info type log.")
        self.logger.debug("This is a debug type log.")
        self.logger.warning("This is a warning type log.")
        self.logger.error("This is an error type log.")
        self.logger.critical("This is a critical type log.")
```

不要忘记注册新Job：

```python
register_jobs(
    HelloJobs,
    HelloJobsWithLogs,
)
```

此时文件应该是这样的：

```python
from nautobot.apps.jobs import Job, register_jobs

name = "Hello World Nautobot Jobs"

class HelloJobs(Job):

    class Meta:
        name = "Hello Jobs"
        description = "Hello World for first Nautobot Jobs"

    def run(self):
        self.logger.debug("Hello, this is my first Nautobot Job.")

class HelloJobsWithLogs(Job):

    class Meta:
        name = "Hello Jobs with Logs"
        description = "Hello Jobs with different log types"

    def run(self):
        self.logger.info("This is an info type log.")
        self.logger.debug("This is a debug type log.")
        self.logger.warning("This is a warning type log.")
        self.logger.error("This is an error type log.")
        self.logger.critical("This is a critical type log.")
       

register_jobs(
    HelloJobs,
    HelloJobsWithLogs,
)
```

执行```post_upgrade```后，我们将看到新Job出现在同一组下：

![jobs_logging_2](images/jobs_logging_2.png)

我们现在可以启用该Job并运行它。注意新的日志类型以及与严重级别相关的颜色：

![jobs_logging_3](images/jobs_logging_3.png)

在下一个Job中，我们将看到如何在Jobs中提供用户输入。

## 用户输入

我们将在未来的挑战中了解更多关于Django对象模型的信息。现在，让我们在下一个Job示例中添加一个简单的用户输入函数。

首先，让我们在导入语句中添加```StringVar```：

```python
from nautobot.apps.jobs import Job, register_jobs, StringVar
```

接下来，我们可以创建一个具有用户输入的新Job。请注意，我们使用```StringVar```创建一个变量```username```，然后将其作为属性传递给```run(self, username)```方法：

```python
class HelloJobsWithInputs(Job):
    
    username = StringVar()

    class Meta:
        name = "Hello Jobs with User Inputs"
        description = "Hello Jobs with Different User Inputs"

    def run(self, username):
        self.logger.info(f"Hello Jobs with {username}.")

register_jobs(
    ...
    HelloJobsWithInputs,
)
```

运行该Job一次后，我们会看到一个额外的字段：

![jobs_input_1](images/jobs_input_1.png)

结果页面将显示我们作为用户输入的内容：

![jobs_input_2](images/jobs_input_2.png)

这就是第4天挑战的全部内容！

> [!TIP] 
> 如果遇到任何问题，这里有一个[第4天视频演练](https://www.youtube.com/watch?v=qv32HJxdEDc&list=PLAaTeRWIM_wtqWt3yIKmIFnfEarsuxaYs&index=3)。

## 第4天待办事项

记得在[https://github.com/codespaces/](https://github.com/codespaces/)上停止并删除Codespace实例。

继续在你选择的任何社交媒体平台上发布新创建的Jobs的屏幕截图。确保你使用标签`#100DaysOfNautobot` `#JobsToBeDone`并标记`@networktocode`，以便我们可以分享你的进展！

在明天的挑战中，我们将深入研究Django ORM。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+4+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/) (复制并粘贴：I just completed Day 4 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot)
