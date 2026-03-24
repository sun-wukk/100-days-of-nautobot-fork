# Jobs 结合示例 —— SSoT 与 DiffSync

在过去 19 天中，我们探索了 Nautobot Jobs 的各个方面，从创建、调度，到使用 Git 作为数据源，涵盖了大量内容。

我们已经走到了 Jobs 学习旅程的中间节点，此时适合稍作休整，退一步从整体视角来审视 Jobs 在全局中的定位。

## 从 Jobs 到 Apps

Jobs 非常适合实现战术性目标，例如将 Python 脚本转化为可共享、可定期执行的任务。然而，随着 Jobs 数量的增加，几乎总会到达一个需要对相似 Job 进行分组和整合的临界点。

这正是 Nautobot Apps 发挥作用的地方。我们将在 100 天挑战的后半段深入探讨 Nautobot Apps，但在这里，我们先以 Nautobot 单一事实来源（SSoT）为例，了解 Jobs 的逻辑分组方式。

## SSoT 与 DiffSync

[DiffSync](https://github.com/networktocode/diffsync) 是一个实用工具库，可用于比较和同步不同的数据集。

其主要使用场景是比较并同步多个数据源，如[此处](https://raw.githubusercontent.com/networktocode/diffsync/develop/docs/images/diffsync_components.png)所示：

![diff_sync_1](images/diff_sync_1.png)

这自然适合使用该库通过 Nautobot Jobs 来同步不同的数据源。正是基于这一需求，我们开始创建能够与 Meraki、ACI、IP Fabric 和 Infoblox 等数据源集成的"适配器"。

相关集成可以在 [nautobot-app-ssot/nautobot_ssot](https://github.com/nautobot/nautobot-app-ssot/tree/develop/nautobot_ssot) 中查看。可以注意到，每个集成都将 Nautobot Jobs 作为执行入口：

![ssot_1](images/ssot_1.png)

基于这些"适配器"，我们可以使用其他 Jobs 来执行实际的集成操作，例如 [nautobot ssot 示例 Job](https://github.com/nautobot/nautobot-app-ssot/blob/develop/nautobot_ssot/jobs/examples.py)：

![ssot_2](images/ssot_2.png)

可以看到，Job 可以相互嵌套叠加，并作为基础模块被整合进可发布为软件包的 App 中。

在明天的挑战中，我们将回归更多 Nautobot Job 的实践示例。

## 第 20 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上分享您对构建 SSoT、DiffSync 或 Jobs 的想法，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将了解如何通过 Jobs 上传和处理文件。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+20+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 20 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
