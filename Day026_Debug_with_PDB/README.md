# 使用 PDB 调试 Jobs

`pdb` 是 Python 调试器，一个用于调试 Python 程序的内置模块。它允许我们设置断点、单步执行代码、检查变量，并以交互方式求值表达式。

在今天的挑战中，我们将了解如何使用 `pdb` 来调试 Nautobot Jobs。

## 环境配置

环境配置与 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md) 相同，以下是步骤摘要，如需详细背景说明请参阅该指南。

> [!TIP]
> 我们在 [Lab Related Notes](../Lab_Setup/lab_related_notes/README.md) 中保存了各实验场景的实用技巧，可供参考。

如果已停止 Codespace 环境，只需重新启动并按以下步骤操作，无需重建 Docker 实例或重新导入数据库：
```bash
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke debug
```

如果需要在 Codespace 中完全重建环境，请执行以下步骤：
```bash
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

今天的挑战无需使用 Containerlab。

## Python PDB 示例

在将 `pdb` 用于 Nautobot Jobs 之前，先通过一个简单的 Python 示例了解 `pdb` 的基本用法。

进入 Nautobot 容器 Shell：
```bash
@ericchou1 ➜ ~ $ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash
root@ee2753f052ae:/opt/nautobot# 
```

创建一个简单的 Python 文件，这里我在容器中安装并使用了 `vim`，您也可以使用任何顺手的文件创建和编辑方式：
```bash
root@ee2753f052ae:/opt/nautobot# apt update
root@ee2753f052ae:/opt/nautobot# apt install vim
```

以下是 `pdb_example.py` 的内容及执行结果：
```bash
root@ee2753f052ae:/opt/nautobot# cat pdb_example.py 

def my_function():
    x = 1
    y = 2
    z = x + y
    print(z)

my_function()

root@ee2753f052ae:/opt/nautobot# python pdb_example.py 
3
```

在文件开头添加 `import pdb`，并在打印 `z` 的结果之前通过 `pdb.set_trace()` 插入断点：
```bash
root@ee2753f052ae:/opt/nautobot# cat pdb_example.py 
import pdb

def my_function():
    x = 1
    y = 2
    z = x + y
    pdb.set_trace()  # 执行将在此处暂停
    print(z)

my_function()
```

这次执行文件时，程序将进入 `(Pdb)` Shell。我们输入 `print(x)` 打印 x 的值，输入 `print(y)` 打印 y 的值，然后按 `n` 键（next）继续执行。屏幕上显示如下：
```bash
root@ee2753f052ae:/opt/nautobot# python pdb_example.py 
> /opt/nautobot/pdb_example.py(8)my_function()
-> print(z)
(Pdb) print(x)
1
(Pdb) print(y)
2
(Pdb) n
3
--Return--
> /opt/nautobot/pdb_example.py(8)my_function()->None
-> print(z)
(Pdb) exit()
```

以下是 PDB 中用于控制程序流程的常用命令：

- n（next）：继续执行，直到当前函数的下一行。
- s（step）：执行当前行，并在第一个可能的时机停止。
- c（continue）：继续执行，直到遇到断点。
- l（list）：显示当前行附近的源代码。
- p（print）：求值并打印表达式。
- q（quit）：退出调试器并终止程序。

删除测试文件：
```bash
root@ee2753f052ae:/opt/nautobot# rm pdb_example.py 
```

接下来看一个在 `Nautobot Jobs` 中使用 PDB 的示例。

## 使用 PDB 调试 Jobs 示例

假设我们有如下 Job 文件：
```bash
from nautobot.apps.jobs import Job, register_jobs, ObjectVar, StringVar, IntegerVar, FileVar

name = "Hello World Nautobot Jobs"

class HelloWorldwithLogs(Job):

    class Meta:
        name = "Hello World with Logs"
        description = "Hello World with different log types"

    def run(self):
        self.logger.info("This is an log of info type.")
        self.logger.debug("This is an log of debug type.")
        self.logger.warning("This is an log of warning type.")
        self.logger.error("This is an log of error type.")
        self.logger.critical("This is an log of critical type.")


register_jobs(
    HelloWorldwithLogs,
)
```

可以通过 CLI 执行该 Job：
```bash
root@ee2753f052ae:/opt/nautobot# nautobot-server runjob hello_job.HelloWorldwithLogs -u admin

[02:19:12] Running hello_job.HelloWorldwithLogs...
```

在 `run()` 方法下创建一些变量并插入断点：
```bash
from nautobot.apps.jobs import Job, register_jobs, ObjectVar, StringVar, IntegerVar, FileVar
import pdb 

name = "Hello World Nautobot Jobs"

class HelloWorldwithLogs(Job):

    class Meta:
        name = "Hello World with Logs"
        description = "Hello World with different log types"

    def run(self):
        x = 1
        y = 2
        name = "Eric"
        self.logger.info("This is an log of info type.")
        self.logger.debug("This is an log of debug type.")
        self.logger.warning("This is an log of warning type.")
        pdb.set_trace()
        self.logger.error("This is an log of error type.")
        self.logger.critical("This is an log of critical type.")

register_jobs(
    HelloWorldwithLogs,
)
```

执行 Job 时需要添加 `--local` 标志：
```bash
root@ee2753f052ae:/opt/nautobot# nautobot-server runjob hello_job.HelloWorldwithLogs -u admin --local
[02:23:06] Running hello_job.HelloWorldwithLogs...
02:23:06.508 DEBUG   nautobot.extras.jobs jobs.py                                run_job() :
  Running job hello_job.HelloWorldwithLogs
```

切换到该 Job 的"Job Result"页面，可以看到 Job 处于"pending"（等待中）状态，输出显示我们正处于 `(Pdb)` 环境中：

![job_pdb_1](images/job_pdb_1.png)

接下来按照排查问题的思路进行操作：
```bash
print(x)
print(y)
print(name)
exit()
```

执行结果将显示在"Job Results"的日志中：

![job_pdb_2](images/job_pdb_2.png)

PDB 初看起来可能有些陌生，感觉投入多收获少。但当我们需要在 Job 的执行上下文中进行调试时，它是最经得起时间检验的利器之一。

## 第 26 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布 `PDB` 成功执行结果的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将深入了解 URL 分发机制。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+26+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 26 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
