# 第 97 天：社区聚焦 - Byrn Baker

## 目标

在今天的挑战中，我们将聆听社区贡献者 Byrn Baker 的故事。我们将了解 Byrn 是如何接触 Nautobot 的，讨论他使用最多的功能，收集他对新手的建议，并讨论社区互动的最佳实践。

## 关于 Byrn Baker

Byrn 是一位拥有十多年电信网络自动化经验的工程师，从电缆技术人员起步，逐步晋升至 NOC 和工程部门的各种职位。

他对 Netbox（Nautobot 的前身）的最初接触是出于管理和跟踪大量" settlement-free peering "电路的需求，这些以前是用电子表格跟踪的。这展示了用于网络文档的结构化数据库的早期价值。

Byrn 向网络自动化和深入参与 Nautobot 的转变是由对比电子表格更强大的解决方案的需求以及他对实践学习的自然倾向所驱动的。

## 与 Byrn 的问答

1. 首先，你能告诉我们一些关于你的背景以及你是如何最初参与 Nautobot 的吗？

**Byrn**：好的，当然。我的背景主要是网络工程。我自 2000 年左右进入电信行业，做了大约 10 年的电缆技术人员，然后在 2010 年转入 NOC。

我在同一个提供商处转入 NOC 职位，并经历了各种 NOC 和工程职位。在 Nautobot 之前，我实际上使用 NetBox 进行文档编制。我注意到 Nautobot 并开始使用它是因为我想，"好吧，有什么区别呢？"我真的喜欢和感激 Nautobot 的是，它的全部目的是使其成为真相来源并将其集成到自动化中。这对我来说非常有意义。这让我开始走上这条路，我做了那个将 Nautobot 融入其中的 Ansible 研讨会。所以，那是我开始使用 Nautobot 的探索。

2. 你提到从对 Nautobot API 和自动化几乎零舒适度开始。你是如何学习如此新的东西的，尤其是编码背景有限的情况下？

**Byrn**：我的舒适度绝对为零。我来自网络背景，不是 Python 程序员。我是一个非常注重实践的人。这是我学习如何做事情的方式。我会坐在电脑前几个小时，输入不同的东西，试图看看什么有效，什么无效。我会输入它，按回车键，然后运行它看看会发生什么并阅读 traceback。这对我来说似乎是更好的学习方式。

我花了很长时间才理解这一点，但 traceback 不会对你撒谎；你只是需要理解它告诉你的内容。这是困难的部分。反复这样做，你最终会逐渐掌握那些东西。

3. 在此基础上，当你遇到困难或遇到你不能立即理解的 traceback 时，你的典型调试过程是什么？你是试图独自分析它，还是尽早去社区？

**Byrn**：当我学习研讨会的东西时，是数小时的头撞墙。我的主要方式仍然是做 - 坐在电脑前，输入东西，按回车，然后阅读 traceback 看看会发生什么。你看过足够多的 traceback，你可能会挑出最重要的部分并专注于它。但肯定有时候我就是不知道答案。有时我可以 Google，但它有时真的得不到你需要的答案。那是我去社区的时候。

4. 说到社区，你提到在 Nautobot Slack 频道问了很多问题。社区在你的学习过程中有多重要？你在那里发现的价值在其他地方得不到的是什么？

**Byrn**：社区非常庞大。如果你想要一个快速答案或至少被指向正确的方向，这是一个很好的方式。有时 Google 不能给你你需要的东西。而且，你不能害怕看起来很笨。我有时会问很多笨问题。当我第一次开始时，特别是像 nbshell 这样的东西，我不知道如何导航它，因为我全新接触 Python 和所有这些东西。我有大量大量的问题 - 我如何导航它，如何显示字段等等。我有时甚至不知道该去哪里找。社区帮助回答这些基本问题。

5. 对于在社区中提问的人，特别是当他们卡住时，你有什么有效提问的建议以获得最佳帮助吗？

**Byrn**：是的，我认为重要的是不要做一个混蛋。当你有问题时，我认为最好的是尽你所能解释你试图做什么，并提供你做过的但不工作的例子。这大多数时候非常有帮助。它帮助人们理解你在说什么，可能你只是走错了路。如果你不能提供例子或帮助某人理解你从你的角度如何看待它，就很难提供帮助。详细说明你如何得出结论也会有所帮助。不要省略重要的东西。提供多一点信息。如果你认为你发现了一个 bug 或有建议，不要只是在 Slack 中抱怨，去提交 GitHub 问题。记录它并提供反馈。

6. 你提到作为 Python 新手在 nbshell 方面遇到挑战。作为没有编码背景的人，在初次接触 Nautobot 和自动化时，还有其他你发现特别困难的特定障碍或概念吗？

**Byrn**：绝对是不了解 Python。像 nbshell 这样的工具，你需要理解 Python 才能导航它们，这很令人困惑。甚至阅读文档和查看示例也有学习曲线。理解 traceback 告诉你什么花了很长时间。

另一个例子是理解 API 交互 - 比如在 Nautobot 2.0 中，如果你试图分配一个不存在的地址，traceback 会告诉你它未找到。理解这个消息不是谎言，并意识到你必须在分配之前创建地址，需要理解工具如何工作，这与只是点击加号获取地址的"旧方式"不同。

7. 除了为自己学习，你一直非常愿意分享知识，制作视频并为 100 天 Nautobot 做贡献。你想分享你所学到的东西的动机是什么？

**Byrn**：我从其他人分享他们学到的东西中获得了太多。分享他们如何学习做事情的方式。这感觉是回馈社区的一种好方式。我决定分享的很多东西都是我在寻找的。非常具体的东西，而我找不到它。所以，我制作了那个视频系列，有点像"我希望我能找到这个"。我想，如果我在寻找它，可能还有其他人正在寻找非常相似的东西，也许这会帮助他们。真的就是这样。

8. 基于你的经验，你对那些 Nautobot 新手，可能也来自网络背景且 Python/Django 背景有限或没有的人，有什么关于如何学习这个项目的建议？

**Byrn**：对于一个非常初学者，我建议你有一个想法，然后看看你是否能使其一个小部分功能化。例如，想要用设备跟踪 OSPF 信息。也许你可以拿现有代码，比如来自另一个应用，添加到 models.py 中的一个字段，看看它是否能在 GUI 中显示。这是我采取的方法。这是在做小的调整，确保它运行，然后调整另一个部分。这是一个很好的方法。不要因为有人说它愚蠢就放弃。如果你认为它有价值，它可能确实有。

9. 将新工具或自动化实践引入组织有时会面临阻力。基于你自己的经验，你有什么建议如何让人们接受采用像 Nautobot 这样的工具，特别是开源的吗？

**Byrn**：选择一个小的、可行的。例如，可能是 100 天中涵盖的内容：更改 VLAN，添加 VLAN，关闭或打开端口。这些是我们每天做的非常简单的事情。你从那里开始，然后展示，"嘿，看，看我们能做什么"。展示你如何每次都能完美推送配置且没有错误。或者按一个按钮关闭/打开端口。然后你可以展示像在 GUI 中设置 RBAC 这样的东西，这样你就知道谁在登录并关闭了端口。你可以将配置细节抽象化给那些可能没有深入专业知识但足够知道按按钮的人。找到一个小项目，展示价值，然后在此基础上构建。向他们展示他们可能用键盘快速做一件事，但他们不能一键做一百件事。

10. 你提到在工作中使用 Nautobot 作为你的真相来源，管理许多设备和 VMs。你能详细说明你目前用它做的一些实际应用或"酷东西"吗？

**Byrn**：我们正在做一些非常酷的东西。`100 天 Nautobot` 有点推动我开始做一些验证工作。我们验证像操作系统与 Nautobot 中的匹配以及是否安装了所需的代理之类的东西。我们管理数千个 VMs 和数百个物理设备。我们使用自定义字段和其他方面来向 GUI 添加快速字段。我们刚开始使用作业按钮，我认为这是 Nautobot 2 的新功能。在 100 天中完成那一天，我就像，"哇，我们可以制作按钮！"现在我做了一些按钮来推送部署。我可以在 Nautobot 中构建所有东西，然后按一个按钮将其部署到生产环境中。这真的很酷。我正在将这些作业包装到我们的应用程序中。

## Byrn 的贡献

我们要感谢 Byrn 成为如此棒的贡献者和出色的内容创作者。

### YouTube 视频
- [了解如何利用 Nautobot 的 GraphQL 功能从你的 Ansible playbooks 动态查询 Nautobot](https://youtu.be/g4aMH_pGo0Q)
- [Nautobot 介绍/概述 - NTC 社区成员 Byrn Baker](https://youtu.be/NzwbVsewcV8)
- [了解如何利用 Nautobot 的 GraphQL 功能从你的 Ansible playbooks 动态查询 Nautobot](https://youtu.be/g4aMH_pGo0Q)
- [Nautobot 介绍/概述 - NTC 社区成员 Byrn Baker](https://youtu.be/NzwbVsewcV8)

### 对 100 天 Nautobot 挑战的贡献

Byrn Baker 为 100 天 Nautobot 挑战撰写了几天的任务：
- [第 23 天 作业模板](https://github.com/nautobot/100-days-of-nautobot/tree/main/Day023_Jobs_Templates)
- [第 27 天 作业的 URL 调度](https://github.com/nautobot/100-days-of-nautobot/tree/main/Day027_URL_Dispatch_for_Jobs)
- [第 33 天 将 Nautobot 与 Ansible 工作流集成](https://github.com/nautobot/100-days-of-nautobot/tree/main/Day033_Integrate_Nautobot_with_Ansible_Workflow)

### 博客

- [https://blog.byrnbaker.me/](https://blog.byrnbaker.me/)

### LinkedIn

- [在 LinkedIn 上联系 Byrn](https://www.linkedin.com/in/byrnbaker/)

## 第 97 天待办事项

探索 Byrn 的 YouTube 视频，查看他的博客，并在 LinkedIn 上与他联系。

在社交媒体上分享你的经验和见解，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将聚焦另一位社区成员 Dwayne Camacho。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+jst+completed+Day+97+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 97 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）