# Nautobot 表格

Nautobot 中的表格用于以表格格式显示数据列表。Nautobot 扩展了 Django 表格以提供额外的功能和显示选项。

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

### 关键组件

以下是 Nautobot 表格的主要组件。

1. **表格类**：在 Nautobot 中代表表格的类。
2. **列**：代表各个列的表格属性。

### 示例：使用 HTML 的表格

让我们回顾一下 [第 68 天](../Day068_Nautobot_Views_3_Nautobot_Views/README.md) 中我们第一次使用 `ListView` 来处理 `UsefulLink` 模型的第一个视图时的内容，当时在 `views.py` 中使用了 `UsefulLinkListView`：

```python
class UsefulLinkListView(ListView):
    model = UsefulLink
    template_name = "example_app/useful_link_detail.html"
    context_object_name = "useful_links"

    def get_queryset(self):
        return UsefulLink.objects.all()
```

在该视图中，我们引用了一个 HTML 页面，当用户访问内容时会渲染该视图。我们使用无序列表 `<ul>` 在 `useful_link_detail.html` 中循环遍历链接：

```html
{% extends "base.html" %}

{% block content %}
<h1>有用的链接</h1>
<ul>
    {% for link in useful_links %}
    <li><a href="{{ link.url }}">{{ link.description }}</a></li>
    {% endfor %}
</ul>
{% endblock %}
```

这是该视图的截图：

![table_html_1](images/table_html_1.png)

如果我们想把列表改成更易呈现的表格视图，我们可以使用 `table head <thead>`、`table row <tr>`、`table head <th>` 和一些其他表格相关的 HTML 格式标签来修改 `useful_link_detail.html` 中的 HTML 代码：

```html
{% extends "base.html" %}

{% block content %}
<h1>有用的链接</h1>
<table border="1" style="border-collapse: collapse; width: 100%;">
    <thead style="background-color: #f2f2f2;">
        <tr>
            <th>描述</th>
            <th>URL</th>
        </tr>
    </thead>
    <tbody>
        {% for link in useful_links %}
        <tr>
            <td>{{ link.description }}</td>
            <td><a href="{{ link.url }}">{{ link.url }}</a></td>
        </tr>
        {% endfor %}
    </tbody>
</table>
{% endblock %}
```

最终结果是一个更好的表格视图。

![table_html_2](images/table_html_2.png)

为了制作一个简单的表格，需要很多 HTML 代码。我们需要更多的样式来使其类似于其他 Nautobot HTML 页面。此外，为了使它们看起来相似，我们将在所有 HTML 页面上重复相同的 HTML 代码模式。

### 示例：使用 Python 的 Nautobot 表格

让我们将其与引用相同模型并使用 `table_class = tables.UsefulLinkModelTable` 的 `NautobotUIViewSet` 进行比较：

> [!信息]
> 我们不会从之前的日子做任何额外的更改，只是重新检查 `views.py` 中的代码。

```python
class UsefulLinkUIViewSet(views.NautobotUIViewSet):
    queryset = UsefulLink.objects.all()
    table_class = tables.UsefulLinkModelTable
    ...
```

`UsefulLinkModelTable` 继承自 `BaseTable`，其中 Meta 类指示模型和 `tables.py` 中的字段：

```python
class UsefulLinkModelTable(BaseTable):
    """用于 `UsefulLink` 对象列表视图的表格。"""

    pk = ToggleColumn()
    name = tables.LinkColumn()
    actions = ButtonsColumn(UsefulLink)

    class Meta(BaseTable.Meta):
        model = UsefulLink
        fields = ["url", "description"]
```

结果是格式更好的表格，具有正确的样式。最棒的是大部分代码已经为你完成了：

![table_python_1](images/table_python_1.png)

这就是使用 Python 代码处理表格的主要思想。我们可以重用已经完成的代码*并且*它为我们提供了一致的呈现。

恭喜完成第 78 天！

## 第 78 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布你关于今天挑战中使用表格的想法，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将使用过滤器。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+78+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 78 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）