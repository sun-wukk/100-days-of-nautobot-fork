# 对象作为变量

在今天的挑战中，我们将涉及编写Nautobot jobs时的另一个基础主题：[Nautobot Job变量](https://docs.nautobot.com/projects/core/en/stable/development/jobs/#variables)。

Job变量是一种方便的方式来接受Nautobot Web UI中的用户输入，该输入可以与后端中的数据相连接。然后我们可以将此值传递到`run()`方法中使用。

让我们重新启动我们停止的Codespace实验室环境或创建一个新的。

## 实验室环境设置

环境设置将与[实验室设置场景1](../Lab_Setup/scenario_1_setup/README.md)相同，以下是步骤的总结，如果需要详细背景，请参考指南。

> [!TIP]
> 如果你停止了Codespace环境并再次重新启动，但发现Docker守护程序停止工作，请按照设置指南中的步骤重建环境。

以下是启动Nautobot的步骤回顾：

```shell
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

我们已准备好创建新的job文件。

## Job文件创建

让我们为今天的挑战创建一个文件。我们可以通过共享目录或直接在Nautobot docker容器中执行此操作。

> [!TIP]
> 如果需要复习，请查看[第003天 Hello Jobs](../Day003_Hello_Jobs_Part_1/README.md)。

让我们首先在Nautobot docker容器下的`/opt/nautobot/jobs`目录中创建一个名为`Day6_Variable_Example`的文件（如果你愿意，可以通过共享目录创建）：

1. 在`/opt/nautobot/jobs`下创建文件。
2. 将文件所有权更改为`nautobot:nautobot`。

```shell
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash

root@32a27fa1f5a6:/opt/nautobot# cd jobs
root@32a27fa1f5a6:/opt/nautobot/jobs# touch Day6_Variable_Example.py

root@32a27fa1f5a6:/opt/nautobot/jobs# chown nautobot:nautobot Day6_Variable_Example.py 

root@32a27fa1f5a6:/opt/nautobot/jobs# ls -lia Day6_Variable_Example.py 
1618584 -rw-r--r-- 1 nautobot nautobot 0 Nov 10 15:07 Day6_Variable_Example.py
```

> [!IMPORTANT]
> 如果你在Docker容器目录中更改了文件所有权为```nautobot:nautobot```，请确保继续使用此方法修改文件。这有助于防止文件权限问题并确保一致性。

使用左侧的Docker扩展，我们可以右键单击文件在主窗口中编辑它：

![file_edit](images/file_edit.png)

就像在之前的日子里一样，我们将从设置基本要素开始今天的代码：我们的```import```语句、```class Meta```、```run()```方法和```register_jobs```。这确保我们的job得到正确的结构并准备好执行。

```python
from nautobot.apps.jobs import MultiChoiceVar, Job, ObjectVar, register_jobs, TextVar, IntegerVar

name = "Day 6 Variables"

class HelloVariables(Job):

    class Meta:

        name = "Hello Variables"
        description = "Jobs Variable Examples"

    def run(self):
        self.logger.debug("Testing Nautobot Variables.")

register_jobs(
    HelloVariables,
)
```

不要忘记在单独的终端窗口中执行`invoke post-upgrade`来注册job：

```shell
@ericchou1 ➜ ~ $ cd nautobot-docker-compose/
@ericchou1 ➜ ~/nautobot-docker-compose (main) $ poetry shell
nautobot-docker-compose (main) $ invoke post-upgrade
```

我们应该看到job显示在Jobs下：

![variable_job_1](images/variable_job_1.png)

启用Job：

![variable_job_2](images/variable_job_2.png)

然后运行它：

![variable_job_3](images/variable_job_3.png)

## 添加变量

注意我们为文件导入了一些额外的对象：

```python
from nautobot.apps.jobs import MultiChoiceVar, Job, ObjectVar, register_jobs, TextVar, IntegerVar
```

这允许我们向文件中添加`TextVar`和`IntegerVar`选项：

```python
from nautobot.apps.jobs import MultiChoiceVar, Job, ObjectVar, register_jobs, TextVar, IntegerVar

name = "Day 6 Variables"

class HelloVariables(Job):

    message = TextVar() 
    days = IntegerVar(
        default="10"
    )

    class Meta:
        name = "Hello Variables"
        description = "Jobs Variable Examples"

    def run(self, message, days):
        self.logger.debug(f"Please give the message: {message} in {days} days.")

register_jobs(
    HelloVariables,
)
```

注意我们需要将变量传递给`run()`方法，在这种情况下是"messages"和"days"，以便使用。

当我们再次尝试运行该job时，应该在运行job页面上看到其他字段：

![hello_variable_with_input_1](images/hello_variable_with_input_1.png)

如果我们为消息输入"Happy New Year!"，为Days输入"45"，我们应该看到以下结果：

![hello_variable_with_input_2](images/hello_variable_with_input_2.png)

让我们以(value, label)的形式为我们的输入添加一个[MultichoiceVar](https://docs.nautobot.com/projects/core/en/stable/development/jobs/#choicevar)值：

```python
class HelloVariables(Job):
    ...
    CHOICES = (
        ('h', 'Happy'),
        ('s', 'Sad'),
        ('e', 'Excited')
    )
    feelings = MultiChoiceVar(choices=CHOICES)
    ...
    def run(self, message, days, feelings):
        self.logger.debug(f"Please give the message: {message} in {days} days.")
        self.logger.info(f"I am feeling {feelings}!")

```

我们可以看到额外的选择出现在运行页面上：

![hello_variable_with_input_3](images/hello_variable_with_input_3.png)

注意我们在`run()`中收到的结果是值而不是标签：

![hello_variable_with_input_4](images/hello_variable_with_input_4.png)

如果我们退一步思考我们必须为所有以前的表单输入编写的所有HTML和Django代码，我们肯定可以欣赏Nautobot变量的简单性。

在下一个示例中，我们将看到如何使用`ObjectVar`将变量与我们的数据库模型相连接。

## 将ObjectVar与数据相连接

只需几行代码，`ObjectVar`就允许我们将代码与数据库对象相连接。我们将在未来的挑战中广泛使用它。

让我们看一个位置数据库的示例。我们需要先导入对象，然后我们可以使用`ObjectVar`在我们的代码中引用它：

```python
from nautobot.dcim.models.locations import Location 
...
class HelloVariables(Job):
    ...
    location = ObjectVar(model=Location)
    def run(self, message, days, feelings, location):
        ...
        self.logger.info(f"Pick a location: {location}")
```

就这样！现在我们可以提示用户从我们预定义的位置对象中选择一个位置：

![object_var_1](images/object_var_1.png)

结果如下：

![object_var_2](images/object_var_2.png)

## 最终代码

以下是今天挑战的最终代码：

```python
from nautobot.apps.jobs import MultiChoiceVar, Job, ObjectVar, register_jobs, TextVar, IntegerVar
from nautobot.dcim.models.locations import Location 

name = "Day 6 Variables"

class HelloVariables(Job):

    message = TextVar()
    days = IntegerVar(
        default="10"
    )

    CHOICES = (
        ('h', 'Happy'),
        ('s', 'Sad'),
        ('e', 'Excited')
    )
    feelings = MultiChoiceVar(choices=CHOICES)

    location = ObjectVar(model=Location)

    class Meta:
        name = "Hello Variables"
        description = "Jobs Variable Examples"

    def run(self, message, days, feelings, location):
        self.logger.debug(f"Please give the message: {message} in {days} days.")
        self.logger.info(f"I am feeling {feelings}!")
        self.logger.info(f"Pick a location: {location}")

register_jobs(
    HelloVariables,
)
```

记住我们也可以使用我们从第005天学到的过滤方法来过滤对象。

## 第6天待办事项

记得在[https://github.com/codespaces/](https://github.com/codespaces/)上停止并删除代码空间实例。

继续在你选择的任何社交媒体上发布job结果页面的屏幕截图，确保你使用标签`#100DaysOfNautobot` `#JobsToBeDone`并标记`@networktocode`，这样我们可以分享你的进展！

在明天的挑战中，我们将开始使用Nautobot jobs进行一些数据质量检查。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+6+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/) (Copy & Paste: I just completed Day 6 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot)
