# 示例应用创建导航

我们几乎完成了向 Nautobot `example_app` 添加"有用链接"的部分。最后一步是将 URL 链接添加到导航菜单中。

## Nautobot 导航菜单

在 Nautobot 中，核心应用程序和 Apps 都可以通过在应用的 `navigation.py` 文件中的 `menu_items` 添加代码来出现在导航菜单中。

> [!提示]
> Nautobot 文档 [填充导航菜单](https://docs.nautobot.com/projects/core/en/stable/development/core/navigation-menu/) 是关于这个主题的很好的资源。

记住我们使用 `name` 作为 url，我们可以在 `NavMenuItem` 中使用它：

```python navigation.py
menu_items = (
    NavMenuTab(
        ...
                    NavMenuItem(
                        link="plugins:example_app:usefullinks_list",
                        name="Useful Links",
                        permissions=["example_app.view_examplemodel"],
                    ),
        ...
    )
)
```

这是 `navigation.py` 文件的内容：

```python navigation.py
from nautobot.apps.ui import (
    NavMenuAddButton,
    NavMenuGroup,
    NavMenuItem,
    NavMenuTab,
)

menu_items = (
    NavMenuTab(
        name="Apps",
        groups=(
            NavMenuGroup(
                name="Example Nautobot App",
                weight=100,
                items=(
                    NavMenuItem(
                        link="plugins:example_app:examplemodel_list",
                        name="Example Models",
                        permissions=["example_app.view_examplemodel"],
                        buttons=(
                            NavMenuAddButton(
                                link="plugins:example_app:examplemodel_add",
                                permissions=[
                                    "example_app.add_examplemodel",
                                ],
                            ),
                        ),
                    ),
                    NavMenuItem(
                        link="plugins:example_app:examplemodel_list",
                        name="Example Models filtered",
                        permissions=["example_app.view_examplemodel"],
                        query_params={"number": "100"},
                    ),
                    NavMenuItem(
                        link="plugins:example_app:anotherexamplemodel_list",
                        name="Another Example Models",
                        permissions=["example_app.view_anotherexamplemodel"],
                        buttons=(
                            NavMenuAddButton(
                                link="plugins:example_app:anotherexamplemodel_add",
                                permissions=[
                                    "example_app.add_anotherexamplemodel",
                                ],
                            ),
                        ),
                    ),
                    NavMenuItem(
                        link="plugins:example_app:usefullinks_list",
                        name="Useful Links",
                        permissions=["example_app.view_examplemodel"],
                    ),
                ),
            ),
        ),
    ),
    NavMenuTab(
        name="Example Menu",
        weight=150,
        groups=(
            NavMenuGroup(
                name="Example Group 1",
                weight=100,
                items=(
                    NavMenuItem(
                        link="plugins:example_app:examplemodel_list",
                        name="Example Models",
                        permissions=["example_app.view_examplemodel"],
                        buttons=(
                            NavMenuAddButton(
                                link="plugins:example_app:examplemodel_add",
                                permissions=[
                                    "example_app.add_examplemodel",
                                ],
                            ),
                        ),
                    ),
                ),
            ),
        ),
    ),
    NavMenuTab(
        name="Circuits",
        groups=(
            NavMenuGroup(
                name="Example Circuit Group",
                weight=150,
                items=(
                    NavMenuItem(
                        link="plugins:example_app:examplemodel_list",
                        name="Example Models",
                        permissions=["example_app.view_examplemodel"],
                        buttons=(
                            NavMenuAddButton(
                                link="plugins:example_app:examplemodel_add",
                                permissions=[
                                    "example_app.add_examplemodel",
                                ],
                            ),
                        ),
                    ),
                ),
            ),
        ),
    ),
)
```

我们的链接现在作为一级公民出现在导航菜单上：

![navigation_1](images/navigation_1.png)

尝试使用文档中指定的不同选项。例如，如果我们将链接的权重改为 100：

```
                    NavMenuItem(
                        link="plugins:example_app:usefullinks_list",
                        name="Useful Links",
                        weight=100,
                        permissions=["example_app.view_examplemodel"],
                    ),
```

新链接现在出现在 `Example Nautobot App` 组中的其他链接之前。

![navigation_2](images/navigation_2.png)

尝试更改导航菜单的其他选项，这会很有趣！

## 第 57 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布新导航菜单的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将向我们的应用添加一个作业。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+57+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 57 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）