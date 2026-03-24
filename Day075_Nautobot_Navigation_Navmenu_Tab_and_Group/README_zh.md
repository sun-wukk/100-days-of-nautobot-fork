# Nautobot 导航

在 [第 57 天](https://github.com/nautobot/100-days-of-nautobot/blob/main/Day057_Example_App_Creating_Navigation/README.md) 中，我们看到了如何为视图添加导航菜单项。

在今天的挑战中，我们将学习更多关于导航菜单的知识。特别是，如何填充、修改和添加导航菜单。

我们将参考核心和应用开发者文档中解释的概念：

- [填充导航菜单](https://docs.nautobot.com/projects/core/en/stable/development/core/navigation-menu/)
- [添加导航菜单项](https://docs.nautobot.com/projects/core/en/stable/development/apps/api/ui-extensions/navigation/)

让我们开始吧。

## 环境设置

我们将结合使用 [场景 2](../Lab_Setup/scenario_2_setup/README.md) 实验室、[https://demo.nautobot.com/](https://demo.nautobot.com/) 和 [Nautobot 文档](https://docs.nautobot.com/projects/core/en/latest/user-guide/core-data-model/overview/introduction/) 进行今天的挑战。

```
$ cd nautobot
$ poetry shell
$ poetry install
$ invoke build
（这个步骤需要耐心）
$ invoke debug
（这个步骤也需要耐心）
```

## Nautobot 中的 NavMenuTab 和 NavMenuGroup

`NavMenuTab` 和 `NavMenuGroup` 类用于组织 Nautobot 中的导航菜单。选项卡代表顶级导航元素，而组代表选项卡内各项的集合。

1. **NavMenuTab 类**：代表顶级导航选项卡。
2. **NavMenuGroup 类**：代表导航选项卡内各项的组。
3. **NavMenuItem 类**：代表导航组内的各个项。
4. **Registry**：用于注册 `NavMenuTab`、`NavMenuGroup` 和 `NavMenuItem` 实例。

为了比较，我们可以看到 `circuits` 应用和 `example_app` 在导航中的层次差异：

![navigation_1](images/navigation_1.png)

如果我们比较 `circuits` 和 `example_app` 之间的 `navigations.py` 文件，我们可以看到 `circuits` 有自己的 `NavMenuTab`，而 `example_app` 链接在 `APPs` `NavMenuTab` 部分下：

![navigation_2](images/navigation_2.png)

## 权重

定义对象显示位置的主要方式是使用 `weight` 属性。

例如，我们可以将 `circuits` 的 `weight` 从 `500` 更改为 `100`（较低的值意味着较高的优先级）：

![navigation_weight_1](images/navigation_weight_1.png)

`circuits` 对象现在列在顶部：

![navigation_weight_2](images/navigation_weight_2.png)

同样的事情可以在 `NavMenuItems` 内部完成。例如，我们可以将 `example_app` 的 `Useful Links` 菜单项的权重更改为 `500`，而所有其他菜单项更改为 `100`：

![navigation_weight_3](images/navigation_weight_3.png)

相应的顺序将列出 `Useful Links` 在其他对象下方：

![navigation_weight_4](images/navigation_weight_4.png)

## 添加新的 NavMenuTab

让我们看看如何添加 `NavMenuTab` 项。我们可以在 `navigations.py` 的 `menu_items` 列表末尾附加以下代码：

![navigation_3](images/navigation_3.png)

```python
NavMenuTab(
    name="100 Days of Nautobot Tab",
    groups=(
        NavMenuGroup(
            name="100 Days of Nautobot Group",
            weight=150,
            items=(
                NavMenuItem(
                    link="plugins:example_app:examplemodel_list",
                    name="100 Days of Nautobot Model",
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
```

我们将看到新的 `NavMenuTab` 对象及其关联的 `NavMenuGroup` 和 `NavMenuItem`：

![navigation_4](images/navigation_4.png)

导航菜单在其使用上很简单，但有时需要一点练习才能使东西按照你想要的方式出现。

恭喜完成第 75 天！

## 资源

- [填充导航菜单](https://docs.nautobot.com/projects/core/en/stable/development/core/navigation-menu/)
- [添加导航菜单项](https://docs.nautobot.com/projects/core/en/stable/development/apps/api/ui-extensions/navigation/)

## 第 75 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布今天挑战中你创建的新导航菜单的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将再次查看 URL 分派。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+75+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 75 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）