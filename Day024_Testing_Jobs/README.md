# Job 测试

欢迎来到第 24 天！在今天的挑战中，我们将介绍一个重要的步骤：自动化测试代码。

如果您有软件开发经验，可能已经注意到，到目前为止我们还没有编写测试代码——而这在软件开发中往往是必不可少的环节。

测试的重要性体现在以下几个方面：

1. 编写新代码时，需要验证新代码是否按预期工作。
2. 引入新代码后，需要确保新代码没有破坏现有功能或影响预期行为。

## Nautobot Jobs 的测试

测试是一个重要话题，已有许多专著对其进行深入探讨。我们将在后续的挑战中重新审视这一话题，今天仅作概念性介绍。

由于 Nautobot 基于 Django 框架构建，Nautobot Jobs 可以通过 [Django 单元测试](https://docs.djangoproject.com/en/5.1/topics/testing/) 功能进行测试。

此外，[Testing Jobs](https://docs.nautobot.com/projects/core/en/stable/development/jobs/#testing-jobs) 文档中还介绍了一些专门用于测试 Jobs 的实用功能。

让我们先配置好环境。

## 环境配置

环境配置与 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md) 相同，以下是步骤摘要，如需详细背景说明请参阅该指南。

如果已停止 Codespace 环境，只需重新启动并按以下步骤运行 Nautobot，无需重建 Docker 实例或重新导入数据库：
```
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke debug
```

如果需要在 Codespace 中完全重建环境，请执行以下步骤：
```
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

今天的挑战无需使用 Containerlab。

## 创建文件

相信到现在，创建 Job 文件对您来说已经轻车熟路了。以下是连接到 Nautobot Docker 实例并在 `Jobs` 根目录下创建文件的步骤：
```
$ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash
root@c9e0fa2a45a0:/opt/nautobot# cd jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# pwd
/opt/nautobot/jobs
root@c9e0fa2a45a0:/opt/nautobot/jobs# touch job_hook_test.py
root@c9e0fa2a45a0:/opt/nautobot/jobs# chown nautobot:nautobot job_hook_test.py
```

今天挑战的环境已配置完毕。

## 执行现有测试

Nautobot 代码库中已有许多用于测试各项功能的软件测试文件。例如，[test_authentication.py](https://github.com/nautobot/nautobot/blob/develop/nautobot/core/tests/test_authentication.py) 用于测试外部认证功能。

我们可以执行该文件中编写的测试。第一步是 SSH 进入 Nautobot Docker 实例：
```
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash
root@8d0ac3752031:/opt/nautobot#
```

然后切换到 `jobs` 目录并执行测试：

> [!TIP]
> 测试可能需要一些时间，因为需要创建独立的测试数据库表。
```
root@8d0ac3752031:/opt/nautobot# cd jobs/
root@8d0ac3752031:/opt/nautobot/jobs# nautobot-server test nautobot.core.tests.test_authentication.ExternalAuthenticationTestCase
Using NautobotPerformanceTestRunner to run tests ...
Found 10 test(s).
Creating test database for alias 'default'...

    Checking for duplicate records ...

    Checking for duplicate records ...

    Checking for duplicate records ...

    Checking for duplicate records ...

>>> Finding and removing any invalid or dangling Note objects ...

>>> Removal completed. 


System check identified no issues (0 silenced).
..........
----------------------------------------------------------------------
Ran 10 tests in 0.918s

OK
Destroying test database for alias 'default'...
root@eba3a1d8b6ab:/opt/nautobot/jobs# 
```

我们可以按照相同的模式对之前创建的 Jobs 进行测试。

## 测试现有 Job

假设我们在 JOBS 根目录下有如下 `hello_job.py` 文件：
```
from nautobot.apps.jobs import Job, register_jobs, ObjectVar, StringVar, IntegerVar, FileVar
from nautobot.dcim.models.locations import Location
from nautobot.dcim.models.devices import Device
import requests


name = "Hello World Nautobot Jobs"

class HelloWorld(Job):

    class Meta:
        name = "Hello World"
        description = "Hello World for first Nautobot Jobs"

    def run(self):
        self.logger.debug("Hello, this is my first Nautobot Job.")


register_jobs(
    HelloWorld,
)
```

我们可以编写测试来验证日志消息。第一步是在 `nautobot` 目录下创建 `tests` 文件夹：
```
root@ee2753f052ae:/opt/nautobot/jobs# mkdir /opt/nautobot/tests
root@ee2753f052ae:/opt/nautobot/jobs# touch /opt/nautobot/tests/__init__.py
root@ee2753f052ae:/opt/nautobot/jobs# touch /opt/nautobot/tests/TestJobs_1.py
```

同时设置 `JOBS_ROOT` 环境变量：
```
root@ee2753f052ae:/opt/nautobot/jobs# export JOBS_ROOT="/opt/nautobot/jobs"
```

在 `TestJobs_1.py` 文件中编写以下代码：

> [!TIP]
> 本示例取自 [Testing Jobs](https://docs.nautobot.com/projects/core/en/stable/development/jobs/#testing-jobs) 文档，如有兴趣请参阅该文档获取更详细的说明。
```python 
from nautobot.apps.testing import run_job_for_testing, TransactionTestCase
from nautobot.extras.models import Job, JobLogEntry


class MyJobTestCase(TransactionTestCase):
    def test_my_job(self):
        # 测试 $JOBS_ROOT 中 "hello_job.py" 文件内的 "HelloWorld" Job
        # job = Job.objects.get(job_class_name="HelloWorld", module_name="hello_job", source="local")
        job = Job.objects.get(job_class_name="HelloWorld", module_name="hello_job")

        # 或者使用：job = Job.objects.get_for_class_path("local/my_job_file/MyJob")
        job_result = run_job_for_testing(job)

        # 检查运行 Job 所产生的日志
        log_entries = JobLogEntry.objects.filter(job_result=job_result)
        for log_entry in log_entries:
            self.assertEqual(log_entry.message, "Hello, this is my first Nautobot Job.")
```

测试的运行方式与之前相同：
```
root@ee2753f052ae:/opt/nautobot/tests# nautobot-server test TestJobs_1
Using NautobotPerformanceTestRunner to run tests ...
Found 1 test(s).
Creating test database for alias 'default'...
...
...
System check identified no issues (0 silenced).

F
======================================================================
FAIL: test_my_job (TestJobs_1.MyJobTestCase)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/opt/nautobot/tests/TestJobs_1.py", line 17, in test_my_job
    self.assertEqual(log_entry.message, "Hello, this is my first Nautobot Job.")
AssertionError: 'Running job' != 'Hello, this is my first Nautobot Job.'
- Running job
+ Hello, this is my first Nautobot Job.


----------------------------------------------------------------------
Ran 1 test in 2.593s

FAILED (failures=1)
Destroying test database for alias 'default'...
```

最后出现了 `AssertionError`，但目前这没有关系。更重要的是，我们已经成功运行了一个自定义测试。

编写测试有时感觉像是"额外"工作，因为它们不实现任何新功能，但它们是确保我们能快速发现代码问题的宝贵工具，能让我们高枕无忧。

## 第 24 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布新 Job 成功执行的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将深入了解 Nautobot CLI 工具 `nautobot-server`。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+24+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 24 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
