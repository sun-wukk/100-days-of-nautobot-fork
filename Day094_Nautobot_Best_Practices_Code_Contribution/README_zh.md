# 第 94 天：Nautobot 代码贡献最佳实践

## 目标

欢迎来到第 94 天！今天我们将讨论为 Nautobot 项目贡献代码的最佳实践。

## Nautobot 中的社区贡献

Nautobot 作为由 [Network to Code (NTC)](https://www.networktocode.com) 支持的免费开源软件 (FOSS)，依靠社区驱动的开发而蓬勃发展。通过贡献与社区互动既受到鼓励也很有价值。事实上，"100 天 Nautobot"项目从一开始就是一个社区参与项目。

作为在过去 90 多天里一直致力于 Nautobot 代码工作的人，迈出贡献代码的下一步是一个很好的选择。这可以与你自己的项目结合进行。在开发项目时，你可能会发现错误、缺失的功能或实现预期目标的更好方法，它们都是向 Nautobot 项目贡献代码的绝佳切入点。

以下是向 Nautobot 进行有影响力的贡献的实用指南和最佳实践。

## 沟通平台

1. Slack：

    - 在 [Network to Code Slack](https://slack.networktocode.com/) 的 #nautobot 频道中进行实时讨论。
    - 限制对短暂主题的讨论，因为聊天历史不会永久保存。

2. GitHub：

    - 使用 [Nautobot GitHub 仓库](https://github.com/nautobot/) 进行功能请求或错误报告等更改。
    - 通过 GitHub 讨论参与一般询问，并在提交前完善功能请求。

## 错误报告流程

1. 检查版本：

    - 确保你正在最新稳定版本上运行，以避免报告已解决的问题。

2. 现有问题审查：

    - 检查 GitHub 问题列表中是否有针对你的错误的现有报告。如果找到，请在你的设置上对其影响进行反应和评论。

3. [提交问题](https://github.com/nautobot/nautobot/issues)：

    - 报告新错误时，包括：
        - **包括堆栈跟踪（如果有）**
        - 环境详情
        - 重现问题的步骤
        - 预期与实际行为
        - 屏幕截图和错误消息

![nautobot_issues](images/nautobot_issues_bug.png)

## 功能请求指南

1. 审查现有请求：

    - 阅读 GitHub 问题列表和讨论以获取类似的请求。如果适用，进行互动并添加理由。

2. 发起讨论：

    - 如果没有现有的讨论，在 GitHub 讨论中发起一个讨论，以在正式提交前验证和完善提案。

3. 提交功能请求：

    - 提供详细信息，例如：
        - 提议的功能
        - 用例
        - 数据库架构的必要更改（如果有）
        - 涉及的第三方库

## 拉取请求流程

请参考 [提交拉取请求](https://docs.nautobot.com/projects/core/en/stable/development/core/#submitting-pull-requests)。

1. 从一个问题开始：

    - 优先打开一个问题与维护者讨论想法，并避免冗余工作。

2. [Fork 仓库](https://github.com/nautobot/nautobot/blob/develop/nautobot/docs/development/core/getting-started.md#forking-the-repo)

    - 在开发 Nautobot 或其他项目时，在你自己的 fork 上进行本地开发是一个好主意。

3. 在功能分支上开发：

    - 使用描述性名称创建功能分支（例如 feature/add-new-feature）。

4. 编码实践：

    - 遵守编码标准，提供清晰的文档字符串，并添加有意义的注释。
    - 请参考 Nautobot [风格指南](https://github.com/nautobot/nautobot/blob/develop/nautobot/docs/development/core/style-guide.md) 和 [最佳实践](https://github.com/nautobot/nautobot/blob/develop/nautobot/docs/development/core/best-practices.md)。

5. 测试：

    - 为你的增强或修复开发单元测试，并确保所有测试通过。

6. 文档更新：

    - 编辑文档以合并新更改并遵循风格指南。

7. 变更日志：

    - 为你的更新起草变更日志。遵循现有格式，请参考文档 [创建变更日志片段](https://docs.nautobot.com/projects/core/en/stable/development/core/#creating-changelog-fragments)。

8. 打开拉取请求：

    - 彻底记录你的拉取请求，确保它包括测试并符合所有指南。

## 资源

- [Nautobot 文档](https://docs.nautobot.com/)
- [Nautobot GitHub 仓库](https://github.com/nautobot/nautobot)

## 第 94 天待办事项

你自己的项目进展如何，可以随意发布关于你自己项目的内容！

或者对于待办事项，阅读以下其中一项：Nautobot 问题、路线图、发布计划，并在你选择的社交媒体上对你有强烈感受的问题发表评论。一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将讨论 Nautobot 治理。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+jst+completed+Day+94+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 94 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）