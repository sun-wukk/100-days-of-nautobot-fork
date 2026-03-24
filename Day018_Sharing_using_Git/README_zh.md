# 使用 Git 仓库共享 Jobs

Job 文件创建完成后，我们可能希望与他人共享，或建立一个集中化的存储位置，以便协作开发、追踪变更，并在新改动出现问题时回滚到之前的版本。

Nautobot 提供了 [Git 作为数据源](https://docs.nautobot.com/projects/core/en/stable/user-guide/feature-guides/git-data-source/) 功能，可通过 Git 仓库共享 Job 代码。

让我们在今天的练习中了解如何实现这一点。

## 环境配置

环境配置与 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md) 相同，以下是步骤摘要，如需详细背景说明请参阅该指南。

> [!TIP]
> 如果您停止了 Codespace 环境后重新启动，发现 Docker 守护进程无法正常工作，请按照配置指南中的步骤重建环境。如果已有实例在运行，只需启动 Poetry 环境并执行 `invoke debug` 即可。

按照以下步骤启动 Nautobot：
```
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

今天挑战的环境已配置完毕。

## Git 仓库结构

我为今天的练习创建了一个简单的 Git 仓库，地址为 [https://github.com/ericchou1/nautobot-jobs-test-repo](https://github.com/ericchou1/nautobot-jobs-test-repo)，仓库包含一个空的 `__init__.py` 文件和一个 `jobs` 目录：
```
# tree nautobot-jobs-test-repo/
nautobot-jobs-test-repo/
├── LICENSE
├── README.md
├── __init__.py
└── jobs
    ├── __init__.py
    └── hello_jobs.py
1 directory, 5 files
```

目录命名非常重要，因为 Nautobot 会查找名为 `jobs` 的文件夹。同时需要 `__init__.py` 文件来标识这是一个 Python 模块。

在 `jobs` 目录中，有两个文件：`__init__.py` 以及我们已经熟悉的 `hello_jobs.py`：
```
from nautobot.apps.jobs import Job

class HelloJobs(Job):
    class Meta: 
        name = "Hello Jobs from Git Repo"

    def run(self):
        self.logger.debug("This is from the Git repo.")
```

Job 的注册移至 `__init__.py` 文件中，在该文件中从 `hello_jobs.py` 导入类并进行注册：
```
# cat __init__.py 
from nautobot.apps.jobs import register_jobs
from .hello_jobs import HelloJobs

register_jobs(HelloJobs)
```

接下来需要在 Nautobot 中将此 Git 仓库注册为 Jobs 数据源。

## 注册 Git 数据源

可以在 "Extensibility -> Git Repositories" 下将 Git 仓库注册为数据源：

![git_data_source_1](images/git_data_source_1.png)

名称和 Slug 的填写较为直观，请确保使用正确的 URL，并将提供者选择为 `jobs`。

![git_data_source_2](images/git_data_source_2.png)

同步成功后，可以在 Jobs UI 中看到新出现的 Job：

![git_data_source_3](images/git_data_source_3.png)

如我们所知，新 Job 需要先启用才能运行。

今天的挑战到此结束，恭喜您坚持到了这里！

## 第 18 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布新 Job 成功执行的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将深入了解 Jobs 数据模型。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+18+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 18 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
