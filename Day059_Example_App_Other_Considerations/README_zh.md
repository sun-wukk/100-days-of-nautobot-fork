# 示例应用其他注意事项

得益于现有的环境，我们能够相对快速地组装一个功能性的 Nautobot 应用。看起来可能不算快，但想想我们需要从头开始构建管理界面、导航栏或数据库交互的时间。我们甚至还没有考虑 Redis 数据库、Celery worker 和 Nautobot 系统中涉及的其他组件。

然而，仍有许多我们可以添加的功能和我们没有机会涵盖的主题。

今天的挑战是找一个你感兴趣的主题，花一个小时阅读文档并在沙盒中尝试。

我会在下面提供一些值得探索的想法，但不要让这个列表阻止你探索你感兴趣的其他主题。

## Django Debug Toolbar

[Django Debug Toolbar](https://django-debug-toolbar.readthedocs.io/en/latest/) 是调查 Django 内部工作原理的*最佳*工具之一。即使你不是"调试"，它也是"看到" Django 如何工作的好方法。

## Django Rest Framework

[Django Rest Framework](https://www.django-rest-framework.org/) 是一个强大且灵活的用于构建 Web API 的工具包。[Nautobot API](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/rest-api/overview/) 就是基于它构建的。

看一下 `example_app` API 文件夹，看看是否...

![api_folder](images/api_folder.png)

## 添加 HTML CSS

我们可以通过 [CSS](https://www.w3schools.com/css/) 修改应用中页面的外观和感觉。改变 HTML 页面中的一个小东西，比如背景颜色，可以是理解 Nautobot 模板继承的一步。

## 添加 JavaScript 功能

类似于 HTML CSS，一旦我们能改变页面的静态样式，为什么不进一步添加一些交互性呢？\[提示：\] 这需要一些 [JavaScript](https://www.javascript.com/) 来实现。

## 第 59 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止（和删除）codespace 实例。

继续在你选择的社交媒体上发布你决定更深入学习的主题，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将快速回顾"第三部分"并展望"第四部分"的内容。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+59+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 59 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）