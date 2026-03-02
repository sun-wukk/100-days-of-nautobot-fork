# Nautobot Jobs概览

Nautobot 是一个网络自动化与编排平台，允许用户自动化网络任务和工作流。

使 Nautobot 成为自动化平台的关键功能之一是能够运行“Jobs”。每个Jobs都是用 Python 编写并存储在 Nautobot 中的可执行脚本或程序。

> [!NOTE]
> 如果你熟悉 NetBox，Nautobot Jobs 统一并取代了 NetBox 之前由“自定义脚本”和“报告”提供的功能。

Jobs 可以通过创建自定义自动化脚本来扩展 Nautobot 的功能，以执行诸如配置管理、数据验证、报告生成等任务。

在今天的挑战中，我们将在 [demo.nautobot.com](https://demo.nautobot.com/) 上执行一个预定义的 job，以熟悉 Jobs 的基本用法。我们还将了解在自动化任务中使用 Nautobot Jobs 的优势。

> [!TIP]
在单独的浏览器窗口中打开 [demo.nautobot.com](https://demo.nautobot.com/)，以便将说明并排显示（右键 -> 在新标签页中打开链接）

## Nautobot Jobs示例

1. 访问 [demo.nautobot.com](https://demo.nautobot.com/) 并使用用户名 ```demo``` 和密码 ```nautobot``` 登录：

![demo_nautobot_1](images/demo_nautobot_1.png)

2. 在左侧面板展开 JOBS 部分并点击 ```Jobs```：

如您所见，演示站点预装了来自不同 Nautobot 应用程序的各种Jobs以及系统Jobs。

![demo_nautobot_2](images/demo_nautobot_2.png)

3. 向下滚动到 `System Jobs`，并点击 `Export Object List` 旁的蓝色 Run 按钮：

![demo_nautobot_3](images/demo_nautobot_3.png)

> [!TIP]
> 默认情况下，新创建的Jobs未启用，这就是为什么有些Jobs显示为“灰显”。但此“导出对象列表（Export Object List）”Jobs默认是启用的。

4. 我们从“内容类型（Content Type）”下拉菜单中选择 `circuits|circuit`，其余选项保持默认，然后点击“立即运行（Run Job Now）”：

![demo_nautobot_4](images/demo_nautobot_4.png)

5. 系统会将你重定向到“Jobs结果：导出对象列表”页面，在那里你可以查看Jobs状态并查看Jobs执行日志。

该Jobs还会生成一个可供点击下载的文件输出：

![demo_nautobot_5](images/demo_nautobot_5.png)

下载该文件后，您会看到它是现有电路数据的 CSV 表示。要验证，请在屏幕左侧导航到“CIRCUITS -> Circuits”：

![demo_nautobot_6](images/demo_nautobot_6.png)

## Nautobot Jobs 的优势

您可能想知道，执行这个简单任务有什么特别之处？让我们指出一些在后台运行并使该任务能够执行的事项：

1. **异步执行**

该Jobs以异步、非阻塞的方式执行。如果你注意到，当我们点击“Run”时，会立即被重定向到结果页面。我们不必在可以再次与 Nautobot 交互之前等待Jobs结果返回。

这是因为 Nautobot 集成了带有分布式任务队列的 Celery 框架（https://docs.celeryq.dev/en/stable/getting-started/introduction.html）。有一个消息总线和一个代理，允许实现可扩展性，例如添加更多的 worker 来执行更多Jobs。

通过将我们的脚本转换为 Nautobot Jobs，我们也可以使用 Celery 分布式任务队列享受异步执行。

2. **责任追究**

在Jobs结果页面，我们可以看到执行该Jobs的用户以及其执行过程中生成的各种日志级别，从而提供责任追究和审计追踪。

3. **性能基准**

我们可以在Jobs执行中看到基本的性能基准，例如Jobs从开始到完成的持续时间。

4. **与现有 Nautobot 数据的交互**

该Jobs直接与 Nautobot 中已存在的数据进行交互。我们无需编写对 Nautobot 的外部 API 调用。该Jobs可以访问对象的数据以及对象之间的关系。

5. **其他好处**

使用 Nautobot Jobs还有许多其他好处，随着学习的深入我们会了解这些，例如调度Jobs、Jobs钩子、权限等。

## Nautobot Jobs文档

除了在 [demo.nautobot.com](https://demo.nautobot.com) 上执行示例Jobs外，今天还有一些任务：

1. 在演示站点上再执行几个Jobs，查看不同的选项并进行试验。
2. 阅读或至少浏览一下用户指南（[User Guide](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/jobs/)）中的“Nautobot Jobs”一节。
3. 熟悉《Jobs 开发者指南》(https://docs.nautobot.com/projects/core/en/stable/development/jobs/) 并将其加入收藏，这将是今后几天很有用的参考文档。

## 第 2 天 待办事项

现在把你刚完成的第 2 天任务连同“Jobs结果”页面的截图发布到你选择的社交媒体上，确保使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标注 `@networktocode`，这样我们就能分享你的进展，明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+2+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制并粘贴：我刚刚完成了“100 天 Nautobot”挑战的第 2 天，https://github.com/nautobot/100-days-of-nautobot！@networktocode #JobsToBeDone #100DaysOfNautobot）
# Nautobot 任务概述

Nautobot 是一个网络自动化和编排平台，允许用户自动化网络任务和工作流。

使 Nautobot 成为自动化平台的关键功能之一是能够运行"任务"。每个任务都是用 Python 编写的可执行脚本或程序，存储在 Nautobot 中。

> [!NOTE]
如果你熟悉 Netbox，Nautobot Jobs 统一并取代了 Netbox 之前提供的"自定义脚本"和"报告"功能。

Jobs 可以通过创建自定义自动化脚本来扩展 Nautobot 的功能，以执行配置管理、数据验证、报告生成等任务。

在今天的挑战中，我们将在 [demo.nautobot.com](https://demo.nautobot.com/) 上执行一个预定义的 job，以熟悉 Jobs 的基本用法。我们还将了解使用 Nautobot Jobs 进行自动化任务的优势。

> [!TIP]
打开一个单独的浏览器窗口访问 [demo.nautobot.com](https://demo.nautobot.com/)，以便并排查看说明（右键单击 -> 在新标签页中打开链接）

## Nautobot 任务示例

1. 导航到 [demo.nautobot.com](https://demo.nautobot.com/) 并使用用户名 ```demo``` 和密码 ```nautobot``` 登录：

![demo_nautobot_1](images/demo_nautobot_1.png)

2. 在左侧面板上，展开 JOBS 部分并点击 ```Jobs```：

如您所见，演示站点预装了来自不同 Nautobot 应用程序的各种Jobs以及系统Jobs。

![demo_nautobot_2](images/demo_nautobot_2.png)

3. 向下滚动到 `System Jobs`，然后点击 `Export Object List` 旁边的蓝色 Run 按钮：

![demo_nautobot_3](images/demo_nautobot_3.png)

>[!TIP]
> 默认情况下，新创建的Jobs是禁用的，这就是为什么其中一些Jobs显示为"灰显"。但这个"导出对象列表"Jobs默认是启用的。

4. 从`内容类型`下拉菜单中选择`circuits|circuit`，将所有选项保持为默认值，然后点击`立即运行Jobs`：

![demo_nautobot_4](images/demo_nautobot_4.png)

5. 您将被重定向到"任务结果：导出对象列表"页面，您可以在该页面查看任务的状态以及查看任务执行日志。

此任务还会生成一个文件输出，您可以点击下载：

![demo_nautobot_5](images/demo_nautobot_5.png)

下载文件后，您可以看到它是现有电路数据的 CSV 表示。要验证，请在屏幕左侧导航到"CIRCUITS -> Circuits"：

![demo_nautobot_6](images/demo_nautobot_6.png)

## Nautobot Jobs的优势

您可能想知道，执行这个简单的Jobs有什么特别之处？让我们指出一些在幕后工作以使Jobs能够运行的事项：

1. **异步执行**

该任务以异步、非阻塞的方式执行。如果你注意到了，一旦我们点击`Run`，我们就被重定向到结果页面。我们不必等待任务结果返回就可以再次与 Nautobot 交互。

这是因为 Nautobot 集成了 [Celery](https://docs.celeryq.dev/en/stable/getting-started/introduction.html) 框架和分布式任务队列。存在一个消息总线和代理，允许可扩展性，例如添加更多工作进程来执行更多任务。

通过将我们的脚本转换为 Nautobot Jobs，我们也可以使用 Celery 分布式任务队列享受异步执行。

2. **问责制**

在Jobs结果页面上，我们可以看到执行该Jobs的用户以及从其执行过程中生成的各种日志级别，提供了问责制和审计跟踪。

3. **性能基准**

我们可以在Jobs执行中看到基本的性能基准，例如Jobs从开始到完成的持续时间。

4. **与现有 Nautobot 数据的交互**

该任务直接与 Nautobot 中存在的数据进行交互。我们不必编写对 Nautobot 的外部 API 调用。该任务可以访问数据以及对象之间的关系。

5. **其他优势**

Nautobot 任务还有许多其他优势，我们将在后续学习中了解到，例如任务调度、任务钩子、权限等。

## Nautobot 任务文档

除了在 [demo.nautobot.com](https://demo.nautobot.com) 上执行示例任务外，今天还有以下几项任务：

1. 在演示站点上执行更多任务，查看不同的选项并进行实验。
2. 阅读或至少浏览 [用户指南](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/jobs/) 中的 `Nautobot 任务` 部分。
3. 熟悉[Jobs 开发者指南](https://docs.nautobot.com/projects/core/en/stable/development/jobs/)并将其加入书签，这将是一份方便的参考文档，供今后查阅。

## 第 2 天待办事项

继续在你选择的社交媒体上发布你新完成的第 2 天任务，并附上 Job Result 页面的截图，确保使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就能分享你的进度，明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+2+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制并粘贴：I just completed Day 2 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot)
