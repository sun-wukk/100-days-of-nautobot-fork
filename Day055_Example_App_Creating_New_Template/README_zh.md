# 示例应用创建新模板

和之前一样，在今天的挑战中，我们将创建一个简单的 HTML 模板来向用户展示响应内容。

## HTML 模板

回顾一下第 54 天我们离开的地方，我们使用以下视图代码来连接 `UsefulLink` 模型，并指定 `example_app/useful_link_detail.html` 作为我们将用来渲染结果的模板：

```python views.py
from example_app.models import AnotherExampleModel, ExampleModel, UsefulLink

from django.views.generic import ListView


class UsefulLinkListView(ListView):
    model = UsefulLink
    template_name = "example_app/useful_link_detail.html"
    context_object_name = "useful_links"

    def get_queryset(self):
        return UsefulLink.objects.all()
```

虽然代码中指定的模板很简单，但目录结构常常是困惑的来源。`example_app` 顶级结构下的模板文件 `useful_link_detail.html` 如下：

```
example_app
  example_app
    templates
      example_app
        useful_link_detail.html
```

![template_1](images/template_1.png)

这是 Django 项目的常见模式，用于组织模板以避免名称冲突，并明确模板属于哪个应用。

简而言之，Django 期望所有模板都在应用代码的 `template/` 文件夹中。由于我们指定了 `example_app/useful_link_detail.html` 作为目标，所以有一个额外的 `example_app` 文件夹。

"为什么名称要冗余？"

这是一个好问题。虽然 `useful_link_detail.html` 这个名称很有表达力，但想一下一些通用名称如 `index.html` 或 `item_list.html`，它们可能不清楚属于哪个应用。这不是硬性规则，但我们只是在这里介绍这种常见模式。

好了，解释得够多了，这里是模板的代码：

```html template.html
{% extends "base.html" %}

{% block content %}
<h1>Useful Links</h1>
<ul>
    {% for link in useful_links %}
    <li><a href="{{ link.url }}">{{ link.description }}</a></li>
    {% endfor %}
</ul>
{% endblock %}
```

如果你以前使用过 Jinja 或其他模板语言，你会认出这种模式。这行 `{% extends "base.html" %}` 继承自 `base.html` 模板，而 `{% block content %}` 开始一个名为 `content` 的块。

下面的代码遍历变量 `useful_links` 中的对象：

```html
<ul>
    {% for link in useful_links %}
    <li><a href="{{ link.url }}">{{ link.description }}</a></li>
    {% endfor %}
</ul>
```

我们怎么知道传递给模板的变量名为 `useful_links`？这是在 `views.py` 的 `context_object_name` 中指定的：

```python
class UsefulLinkListView(ListView):
    model = UsefulLink
    template_name = "example_app/useful_link_detail.html"
    context_object_name = "useful_links"

    def get_queryset(self):
        return UsefulLink.objects.all()
```

好了，我们快完成了，明天我们将为此模板添加 URL 路由。

## 第 55 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体平台上发布今天挑战中的模板代码截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将为我们的应用添加 URL 路由。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+55+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 55 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）