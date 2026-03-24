# Nautobot 表单

在今天的挑战中，我们将讨论如何在 Nautobot 中使用表单。

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

## Nautobot 中的表单

与一般 HTML 一样，Nautobot 中的表单用于处理用户输入并在保存到数据库之前验证数据。Nautobot 扩展了 Django 表单以提供额外的功能和自定义选项。

表单代码主要包含在每个应用的 `forms.py` 文件中，例如 `nautobot -> examples -> example_app -> forms.py`：

![forms_1](images/forms_1.png)

与 DRY（不要重复自己）概念一样，许多共享的表单代码也可以追溯到 `nautobot -> extras -> forms`：

![forms_2](images/forms_2.png)

### 关键组件

1. **表单类**：在 Nautobot 中代表表单的类。
2. **字段**：代表各个输入元素的表单属性。
3. **Mixins**：为表单添加额外功能的类。

### 示例：使用表单

让我们尝试修改一个表单来加强我们的学习。

回到我们的 `example_app`，我们看到 `ExampleModelUIViewSet` 在 `views.py` 中包含 `forms.ExampleModelFilterForm` 的表单引用：

```python
from example_app import filters, forms, tables

class ExampleModelUIViewSet(views.NautobotUIViewSet):
    bulk_update_form_class = forms.ExampleModelBulkEditForm
    filterset_class = filters.ExampleModelFilterSet
    filterset_form_class = forms.ExampleModelFilterForm
    form_class = forms.ExampleModelForm
    queryset = ExampleModel.objects.all()
    serializer_class = serializers.ExampleModelSerializer
    table_class = tables.ExampleModelTable
...
```

让我们看一下 `forms.py` 中的代码片段 `ExampleModelFilterForm`：

```python
from django import forms

from nautobot.apps.constants import CHARFIELD_MAX_LENGTH
from nautobot.apps.forms import (
    BootstrapMixin,
    BulkEditForm,
    NautobotBulkEditForm,
    NautobotModelForm,
    TagsBulkEditFormMixin,
)

class ExampleModelFilterForm(BootstrapMixin, forms.Form):
    """用于 `ExampleModel` 对象的过滤/搜索表单。"""

    model = ExampleModel
    q = forms.CharField(required=False, label="搜索")
    name = forms.CharField(max_length=CHARFIELD_MAX_LENGTH, required=False)
    number = forms.IntegerField(required=False)
```

在 [第 67 天](../Day067_Nautobot_Views_2_Nautobot_UI_ViewSet/README.md) 的基础上，我们可以为 `UsefulLinkUIViewSet` 添加一个 `filter_form_class` 来使用单独的表单。

在第 67 天，我们为 `https://<url>/plugins/example-app/useful-links-2` 的 `NautobotUIViewSet` 映射了 URL。如果我们点击 `Filter` 按钮：

![filter_button](images/filter_button.png)

它会引导我们到一个通用表单：

![filter_form_1](images/filter_form_1.png)

让我们在 `forms.py` 中创建一个新的过滤表单：

```python
from example_app.models import AnotherExampleModel, ExampleModel, UsefulLink


class UsefullinkModelFilterForm(BootstrapMixin, forms.Form):
    """用于 `UsefullinkModel` 对象的过滤/搜索表单。"""

    model = UsefulLink
    q = forms.CharField(required=False, label="搜索")
    url = forms.CharField(max_length=CHARFIELD_MAX_LENGTH, required=False)
    description = forms.CharField(max_length=CHARFIELD_MAX_LENGTH, required=False)
```

我们可以在 `views.py` 中将新的过滤表单连接到 `UsefulLinkUIViewSet`：

```python
class UsefulLinkUIViewSet(views.NautobotUIViewSet):
    queryset = UsefulLink.objects.all()
    table_class = tables.UsefulLinkModelTable
    filterset_class = filters.UsefulLinkModelFilterSet
    filterset_form_class = forms.UsefullinkModelFilterForm  # 新增
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

通过这样做，当我们点击 `filter` 按钮时，我们可以使用自定义的表单：

![filter_form_2](images/filter_form_2.png)

值得注意的是，当我们处理表单时，通常需要编写 HTML 代码（以及 CSS 和可能的 JavaScript），但 Nautobot 允许我们使用 Python 对象来渲染 HTML 代码，使我们能够获得更清晰、一致的外观。

恭喜完成第 77 天！

## 第 77 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布今天挑战中新过滤表单的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将了解 Nautobot 表格。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+77+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 77 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）