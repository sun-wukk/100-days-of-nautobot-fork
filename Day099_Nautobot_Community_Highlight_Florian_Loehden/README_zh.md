# 第 99 天：社区聚焦 - Florian Loehden

## 目标

今天我们将聚焦另一位社区贡献者 - Florian Loehden。Florian 一直持续参与 100 天 Nautobot 挑战并取得进展，特别值得一提的是他通过博客记录学习路径，通过帮助其他成员积极与社区互动，以及直接为项目代码做出贡献。

## 关于 Florian Loehden

Florian Loehden 目前在德国完成他的计算机科学硕士学位，同时在 DevOps 和网络自动化领域工作，他的公司在那里使用 NetBox 和 Nautobot 作为真相来源。

他通过同事和他的兴趣 - 在网络自动化硕士论文项目中集成 Nautobot - 参与了 Nautobot 项目和 `100 天 Nautobot` 挑战。

### 100 天 Nautobot 挑战

Florian Loehden 是 [100 天 Nautobot 挑战](https://github.com/nautobot/100-days-of-nautobot/blob/0ef713bca6111501313a42860919ea9369e1b356/README.md) 的关键贡献者，这是一项为社区成员提供的自主指导之旅，使用 Nautobot 构建强大而一致的网络自动化技能。该挑战专注于 Nautobot Jobs 和 Apps，利用 Nautobot 在执行网络自动化任务中的独特地位。

## 与 Florian 的问答

1. 你提到你通过同事了解这个项目，你们在工作中同时使用 Netbox 和 Nautobot 作为真相来源。那你是这样了解这个项目的吗？如果我们回到你刚开始接触这个项目和工具的时候，有什么挑战？我发现很多人不知道从哪里开始，这也是我们启动这个项目的原因之一。

Florian：谢谢你给我这个机会。我目前正处于计算机科学硕士学位学习的最后阶段，同时在德国 Logicalis Connected 工作。我们在 DevOps 和网络自动化方面做很多事情。由于我的同事，我了解了这个关于 Nautobot、Nautobot Jobs 以及如何利用 Nautobot 进行网络自动化的挑战。我遇到了这个挑战，所以为什么不通过这个来学习 Nautobot 呢？我早些时候想学习 Nautobot，因为我不太熟悉它。

我在网络自动化领域撰写我的硕士论文，我考虑过如何集成 Nautobot 或如何使其与 Nautobot 交互。我的同事们使用 Nautobot 并告诉我这是一个很棒的工具。我们在很多项目中使用它。有了这个，我想，好吧，我必须首先学习它，然后才能集成它。拥有一个网络真相来源很重要。我们在不同的项目中使用 Netbox 和 Nautobot。我基本上是将 Nautobot 作为一个文档工具来了解它的，用于记录我们的网络。我们的服务在哪台服务器上，哪个 IP 分配给哪台服务器，它始终是一种很好的方式来最终可视化这些。

2. 好吧，让我也许重新表述一下这个问题。那么，在 100 天 Nautobot 挑战之前，你的挑战深度或知识深度是多少？关于 Django 或 Python 你有什么程度的挑战？或者你的舒适度水平是什么？

Florian：在我开始挑战之前，我已经使用 Linux 并在 Django 和 Python 上构建了一个网站。挑战中关于 Django 的部分是一个很好的提醒，让我想起了那段时光。我做了教程并基于 Django 构建了一个系统。我为文档构建了一个系统，就像一个数据库前端。我知道 Netbox 和 Nautobot 可以做更多。我对 Django 有基本的理解。然而，作为更多做后端和 DevOps 而不是前端的人，我的知识就到这里为止。

3. 我想把你带回到 100 天 Nautobot 项目中，因为那是我们在这里的原因，我想问你一个问题... 对我来说最大的挑战实际上是如何在 codespace 中适应一切。就真相来源而言，你遇到的最大挑战是什么？

Florian：是的，最大的挑战之一是资源管理，我习惯了有比 Codespace 实例提供的更多资源。这是一次很好的学习。例如，我们在第一个环节中在 Nautobot 中使用 Ansible，并在容器中安装了 Ansible。

首先，我运行了所有的路由器容器，甚至比描述中说的还多，还有 nautobot 容器。然而，用这个，Ansible 的安装不可能，它崩溃了我的 codelabs 实例。所以我不得不首先只启动 Nautobot，安装 Ansible，然后只启动路由器容器（只有两个）然后用它们做任务。我也在论文中使用了 containerlab 和 Kubernetes，但从未遇到这些问题。我想我不得不重建环境 4 或 5 次。但这教会了我如何处理资源和构建环境。

4. 你还记得从 100 天 Nautobot 项目中遇到的其他挑战吗？你是如何克服它们的？

Florian：另一个大挑战是保持一致。这并不容易。我们都有限制我们时间的约束，有时候我也没办法完成挑战。重要的是再次坚持而不是放弃。邮件提醒帮助了我。也许我也因为它感到有点内疚，但跟踪我已经完成的日子和还没有完成的日子是好的。

另一个重要的挑战是遇到问题并理解为什么我遇到那些问题。故障排除过程很重要。有时候，这不是关于那天的任务，而是之前一两天的事情，重建它是过程的一部分。失败是过程的一部分。重要的是不要因此而气馁，而是深入了解为什么会出现问题并从中学习。这样，我们就能在最终变得更强。

5. 所以我也想提一下，你知道我们提到了一点关于你的贡献。所以在 100 天 Nautobot 之外，你持续在 LinkedIn 上发帖进行社交问责，你还写博客，我看到你在 Slack 上评论其他社区成员，回答其他人的问题，并鼓励其他人坚持完成这个过程。所以所以我想总结一下。你做了哪些贡献，你认为哪些影响了其他成员？

Florian：在我看来，互相帮助并一起前进是很棒的。当我知道其他人在做同样的事情时，我学到了很多，与他们讨论我们刚刚学到的东西是可能的。而且，有可能在像 Slack 频道这样的公共平台上联系，我们可以快速获得答案。

我的第一个贡献是修复 Twitter 和 LinkedIn 帖子中的 GitHub 链接。每次发帖时都要修改它确实很烦人，对于到那时为止发布的所有日子一次性修改它比每天修改一次更容易。这是第一次我能够为一个开源项目做出贡献，这是一个进入开源贡献世界的好方法。

6. 是的，完全正确？我认为这也是你提出的另一个好点，但我想进一步阐述。你可以为项目做贡献而不需要成为技术专家对吧？我认为这就是你所做的，我想给你荣誉，Florian 所做的是他意识到在我预先构建的 Twitter 和 LinkedIn 链接中有一些错误，但他能够为一堆这样的日子创建一个 poll 请求，并同时纠正它们。因此，你知道，你显示为一个贡献者，你知道，它都是公开文档化的，这建立了你自己的作品集。所以我认为这是我从人们身上看到的也许意想不到的好处，他们可能或可能没有与其他开源项目合作，他们能够做 poll 请求，他们经历合并过程、分支，以及只是与不同时区的人交流的整体沟通。

Florian：在这个挑战中，我学到的远不止使用 Nautobot 和 Nautobot Jobs 以及我能用它们做什么。Nautobot 的生态系统是巨大的，周围有很多东西。我也遇到了 Terraform Provider，了解它很有趣。

7. 在你的工作中或你在硕士学习中，你认为哪些功能或东西可能对 Nautobot 非常有帮助但今天还没有？或者只是学习日子的事情？

Florian：我来自某些领域的经验，所以有些步骤对我来说太小了。这取决于人们首先拥有的背景知识，因为其他任务我必须阅读很多并且挣扎过。但这也是其中的乐趣。我们学习新东西，但也刷新一些旧知识。

在我的情况下，我知道一些 Python，但不是网络方面的，所以我也需要在这方面建立，就像你那本书一样！

8. 是的。所以我想你知道，我们正处于对话的结尾。我从对话中收获很多。其中之一，这是我们在对话中刚刚出现的想法之一，是我们可以做一种评估，我们给你像这样的顺序测验，说如果你理解你知道 docker run 如果你理解 docker compose 然后这是你在 docker 方面的水平，如果你理解你知道某个水平的 Django 如果你在做那个 Django admin 那么这是你的水平，所以基于那个我们将给你这个图表和学习地图，贯穿每个步骤，但我不确定这是否太激进了，但从我们的对话来看。我认为这是我得到的。

Florian：我认为那是一个奇妙的想法，但相当有野心。如果我能贡献什么，我会很高兴和荣幸。

9. 所以在我们结束之前，我确实想问，你认为 Nautobot 可以解决从挑战中学到的其他挑战吗？所以我给你一个例子，因为你提到了文档对吧？所以 Netbox 和 Nautobot 总是可以记录你的 IPAM，你的 DSIM，以及你的其他东西。但有什么新的东西你在整个挑战中想到的吗，触发你认为 Nautobot 可以解决的新挑战？

Florian：对挑战的一个好的补充将是利用 nautobot 生态系统中的其他工具，比如用于基础设施配置的 Terraform provider 和通过 Nautobot 与网络交互。也可以有与虚拟机和客户端交互的部分。

10. 所以我想问你最后一个问题，你在 Nautobot 工作时最常用的工具或库是什么？

Florian：这也与我目前正在做的有关。我最近使用最多的是 also related，因为我的一个朋友为 GNS3 构建了一个 [terraform provider](https://github.com/NetOpsChic/terraform-provider-gns3)。这就是为什么我也最终想，好吧，为什么不？也利用 terraform 与 Nautobot 交互。因此，我遇到了 Nautobot 的 Go 库。

有一个抽象层很好，这样我们就可以使用库而不是向 Nautobot 发出 REST 调用，我们不必自己处理这一切。对于 pynautobot 的库也是一样的。它省去了查看不同的端点，或者更 Python 化，你可以更容易地猜测这个方法或函数是做什么的。

## Florian Loehden 的贡献

我们衷心感谢 Florian 对网络自动化社区和 100 天 Nautobot 挑战的宝贵贡献。

他总是持续在 LinkedIn 上发布最新动态，培养社交问责感，并且从不羞于分享他的知识。

### 博客文章

- [100 天 Nautobot](https://medium.com/@florian.loehden/100-days-of-nautobot-175915146de5)
- [100 天 Nautobot - 高级作业](https://medium.com/@florian.loehden/100-days-of-nautobot-advanced-jobs-c5433ffa1d00)
- [Nautobot 挑战 - 应用基础](https://medium.com/@florian.loehden/nautobot-challenge-the-basics-of-applications-4a4eab7a5d73)
- [100 天 Nautobot — 高级 Nautobot 应用](https://medium.com/@florian.loehden/100-days-of-nautobot-advanced-nautobot-apps-451a378fa30c)

### LinkedIn
- [在 LinkedIn 上联系 Florian](https://www.linkedin.com/in/florian-loehden/)

## 第 99 天待办事项

探索 Florian 的博客，在你选择的社交媒体上发布你学到的东西，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

明天，关于挑战的一些最终想法。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+jst+completed+Day+99+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 99 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）