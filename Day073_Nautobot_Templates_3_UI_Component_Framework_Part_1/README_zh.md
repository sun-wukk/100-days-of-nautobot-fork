# Nautobot UI 组件框架 - 第一部分

[Nautobot UI 组件框架](https://docs.nautobot.com/projects/core/en/stable/development/core/ui-component-framework/) 在 Nautobot v2.4 中引入，它改变了 Nautobot 在应用中创建对象视图的方式。我们不是编写 HTML 模板，而是使用 Python 对象来*声明* UI 结构。这使我们能够拥有更一致、更易维护和更响应式的界面。

**为什么要使用 UI 框架？**
- 减少开发时间
- 设计模式的一致性
- 可重用组件
- 可扩展和可定制

在今天和明天的挑战中，我们将尝试了解 Nautobot UI 框架、它的组件以及如何使用它来创建对象详情视图。

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

## 使用 UI 组件框架的示例应用

在 [第 50 天](https://github.com/nautobot/100-days-of-nautobot/tree/main/Day050_Example_App_Overview) 到 [第 59 天](https://github.com/nautobot/100-days-of-nautobot/tree/main/Day059_Example_App_Other_Considerations) 的日子里，我们使用 `nautobot` 仓库中的 `Example App` 向应用添加有用的链接（模型、视图、URL 和导航）。让我们看看如何使用新的 UI 组件来实现相同的目标。

对于 `models.py`，我们将重用我们在 [第 52 天](https://github.com/nautobot/100-days-of-nautobot/blob/main/Day052_Example_App_Creating_Data_Models_Part_2/README.md) 创建的相同 `UsefulLink` 模型。如果需要重新创建它，请记住使用 `makemigrations` 和 `migrate` 来更新数据库。

以下是 `nautobot -> examples -> example_app -> models.py` 下模型的代码片段：

```python models.py
from django.db import models

class UsefulLink(BaseModel):
    url = models.URLField(unique=True)
    description = models.CharField(max_length=255)

    def __str__(self):
        return self.url
```

下一步是为模型创建 `views`。这是我们可以开始使用 Nautobot UI 组件框架的地方。我们可以将以下代码添加到 `example_app` 下的 `views.py` 文件中：

```python file=views.py
from nautobot.apps.ui import ObjectDetailContent, SectionChoices, ObjectFieldsPanel

class UsefulLinkUIViewSet(views.NautobotUIViewSet):
    queryset = UsefulLink.objects.all()

    object_detail_content = ObjectDetailContent(
        panels=[
            ObjectFieldsPanel(
                weight=100,
                section=SectionChoices.LEFT_HALF,
                fields=[
                    "url",
                    "description",
                ],
            )
        ]
    )
```

我们怎么知道在代码中使用 `object_detail_content` 和 `ObjectDetailContent`？这是阅读 [基础设置](https://docs.nautobot.com/projects/core/en/stable/development/core/ui-component-framework/#basic-setup) 文档和阅读 `example_app -> views.py` 文件中 `ExampleModelUIViewSet` 的代码片段相结合的。

> [!提示]
> [Panels](https://docs.nautobot.com/projects/core/en/stable/development/core/ui-component-framework/#panels) 是 UI 的主要构建块，还有 [Tabs](https://docs.nautobot.com/projects/core/en/stable/development/core/ui-component-framework/#panels) 和 [Buttons](https://docs.nautobot.com/projects/core/en/stable/development/core/ui-component-framework/#buttons)

与其他 `NautobotUIVIewSet` 视图一样，我们将在 `urls.py` 中使用 `NautobotUIViewSetRouter` 注册新的 `viewset`。注意我们使用 `useful-link-2` 作为端点（回想一下之前的 URL 是 `useful-link`）：

```python file=urls.py
app_name = "example_app"
router = NautobotUIViewSetRouter()
...
router.register("useful-links-2", views.UsefulLinkUIViewSet)
```

注意在 `views.py` 代码中我们没有指定要渲染的确切 HTML 模板，这就是重点，HTML 基础模板是*隐含的*。

我们现在可以将浏览器指向 `https://<name>.app.github.dev/plugins/example-app/useful-links-2/` 来查看页面。

哎呀，有一个错误：

![table_class_error](images/table_class_error.png)

原来，`tables` 和 `filtersets`（我们将在未来几天讨论）的组件与 Nautobot UI 组件框架紧密耦合，我们需要在这里包含它们。幸运的是，它们不难理解，即使这是我们第一次看到它们的用法。

让我们将以下代码添加到 `nautobot -> examples -> example_app -> tables.py` 中：

```python file=tables.py
from example_app.models import AnotherExampleModel, ExampleModel, UsefulLink

class UsefulLinkModelTable(BaseTable):
    """Table for list view of `UsefulLink` objects."""

    pk = ToggleColumn()
    name = tables.LinkColumn()
    actions = ButtonsColumn(ExampleModel)

    class Meta(BaseTable.Meta):
        model = UsefulLink
        fields = ["url", "description"]
```

将以下代码添加到 `nautobot -> examples -> example_app -> filters.py` 中：

```python file=filters.py
from example_app.models import AnotherExampleModel, ExampleModel, UsefulLink

class UsefulLinkModelFilterSet(BaseFilterSet):
    """API filter for filtering usefullink model objects."""

    q = SearchFilter(
        filter_predicates={
            "url": "icontains",
            "description": "icontains",
        },
    )

    class Meta:
        model = UsefulLink
        fields = [
            "url",
            "description",
        ]
```

我们现在可以将它们包含在 `views.py` 代码中：

```python views.py
class UsefulLinkUIViewSet(views.NautobotUIViewSet):
    queryset = UsefulLink.objects.all()
    table_class = tables.UsefulLinkModelTable  # 新增
    filterset_class = filters.UsefulLinkModelFilterSet  # 新增
    object_detail_content = ObjectDetailContent(
        panels=[
            ObjectFieldsPanel(
                weight=100,
                section=SectionChoices.LEFT_HALF,
                fields=[
                    "url",
                    "description",
                ],
            )
        ]
    )
```

我们可以导航回 `useful-link-2` 页面并查看显示的页面：

![usefull-links-2](images/usefull-links-2.png)

我们无法直接在页面中执行编辑链接等操作，但我们至少可以体验一下 Nautobot UI 组件框架的强大功能。

恭喜完成第 73 天！

## 资源

- [Nautobot UI 组件框架](https://docs.nautobot.com/projects/core/en/stable/development/core/ui-component-framework/)
- [nautobot.apps.ui](https://docs.nautobot.com/projects/core/en/stable/code-reference/nautobot/apps/ui/)

## 第 73 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布今天挑战中使用 Nautobot UI 组件框架渲染的新页面的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将更深入地研究 Nautobot UI 组件框架。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+73+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 73 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）