# Jobs 中的保留属性名称

回想学习各种编程语言的经历，我记得 [Java](https://en.wikipedia.org/wiki/Java_(programming_language)) 既难学又枯燥，而 [Python](https://en.wikipedia.org/wiki/Python_(programming_language)) 则让我爱不释手。在这两种语言上，许多人都有相似的感受。

原因或许各有不同，但对我而言，Python 区别于其他语言的关键在于它能让你更快地"做成事情"。"Hello World"只需要一两行代码，而借助海量的现有库，从交换机获取 `show version` 输出也不过是更进一步的事。

然而，到了某个阶段，我不得不回过头来学习更多"枯燥的东西"才能继续进步。面向对象编程、动态类型检查、垃圾回收等话题并不让人兴奋，但彼时我已深深着迷。当你已经领略到这门工具的强大之处，再学这些枯燥的话题也就不那么难熬了。

希望在过去 28 天的挑战中，我们已经让您充分体验到了 Nautobot Jobs 的魅力与乐趣。今天的挑战会略显枯燥，但却是必要的。

我们将从学习 Nautobot Job 类中的[保留属性名称](https://docs.nautobot.com/projects/core/en/stable/development/jobs/#reserved-attribute-names)开始。

## 环境配置

今天的挑战无需启动 Codespace 实验室，除非您想边实操边对照阅读。如有需要，请参阅 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md) 启动实验环境。

## 保留属性名称

Job 类有许多属性和方法被用作保留名称，这与 [Python 关键字](https://realpython.com/python-keywords/) 的概念类似——不应将其用作自定义名称。试想一下，如果将 `for` 或 `def` 用作变量名或函数名，会造成多大的混乱。

以下是不应使用的[特殊方法名称](https://docs.nautobot.com/projects/core/en/stable/development/jobs/#special-methods)：

- `before_start()`
- `run()`
- `on_success()`
- `on_failure()`
- `after_return()`

以下是需要了解的[元数据属性](https://docs.nautobot.com/projects/core/en/stable/development/jobs/#special-methods)：

- name
- description
- approval_required
- dryrun_default
- field_order
- has_sensitive_variables
- hidden
- is_singleton
- read_only
- soft_time_limit
- task_queues
- template_name
- time_limit

随着版本更新，该列表可能会有所增减。无需死记硬背，只需了解它们的存在，并将相关页面收藏为参考资料即可。

今天的挑战较为简短，请利用剩余时间浏览 [Jobs 开发者指南](https://docs.nautobot.com/projects/core/en/stable/development/jobs/)（如果您还没有读过的话）。

## 第 29 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上分享您从阅读 [Jobs 开发者指南](https://docs.nautobot.com/projects/core/en/stable/development/jobs/) 中学到的内容，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将学习如何在 Nautobot Jobs 中使用 Nautobot Secret Groups。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+29+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 29 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
