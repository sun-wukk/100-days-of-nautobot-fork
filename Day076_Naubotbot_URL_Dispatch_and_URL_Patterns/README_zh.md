# URL 分派和 URL 模式

在今天的挑战中，我们将深入了解 Nautobot 中 URL 模式的配置以及相关的视图分派。

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

## URL 模式

Django 中的 URL 模式（因此在 Nautobot 中也是）用于将 URL 映射到视图。这允许应用程序用适当的视图和操作来响应不同的 URL。

我们已经有了一些模式方面的经验。回想一下在第 56 天和第 67 天，我们在应用级别配置了各种 URL 模式：

- [第 56 天。示例应用创建新 URL 路由](https://github.com/nautobot/100-days-of-nautobot/blob/main/Day056_Example_App_Creating_New_URL_Routing/README.md)
- [第 67 天。Nautobot 视图 2 Nautobot UI ViewSet](../Day067_Nautobot_Views_2_Nautobot_UI_ViewSet/README.md)

## URL 分派器

Django 中的 URL 分派器使用正则表达式来将请求的 URL 与 `urls.py` 文件中定义的一组模式进行匹配。

**关键组件：**

1. **URLconf 模块**：包含 URL 到视图映射的模块，这通常是 `urls.py`。
2. **Path 函数**：用于定义 URL 模式的函数。
3. **Include 函数**：用于包含其他 URL 模式的函数，如果我们需要将其传递给其他 `urls.py` 文件。
4. **Views**：处理请求并返回响应的函数或类（如果我们直接返回视图）。

让我们看一下顶级 `nautobot -> core -> urls.py`：

- 对于 `""`（空）、`about/`、`search/` 等路径，代码直接返回视图。
- 对于 `circuits/`、`cloud/`、`dcim/` 等路径，代码使用 `include()` 将它们传递给应用中其他 `urls.py` 文件。

![core_urls](images/core_urls.png)

但等一下，我们看到的 `example_app` 呢？代码不是引用文件，而是导入列表并在 `include()` 函数中引用列表：

```python
...
from nautobot.extras.plugins.urls import (
    apps_patterns,
    plugin_admin_patterns,
    plugin_patterns,
)
...
urlpatterns = [
    ...
    # Apps
    path("apps/", include((apps_patterns, "apps"))),
    path("plugins/", include((plugin_patterns, "plugins"))),
    ...
]
```

因为在我们的环境中 `debug` 被设置为 `true`，如果有一个不匹配的 URL，我们会看到一个错误页面，帮助我们看到匹配优先级的顺序：

![url_match_error](images/url_match_error.png)

正如页面上所述，URL 搜索模式按照优先级顺序搜索。如果你想了解更多关于 Django 如何处理请求 URL 模式的信息，可以在 [URL 分派器](https://docs.djangoproject.com/en/5.1/topics/http/urls/) 找到更多信息。

## URL 名称引用

你可能想知道 `path()` 函数中的 `name` 参数是什么。path() 函数中的"name"参数有几个重要目的：

1. **URL 反向解析**：

   - "name" 参数允许你为特定 URL 模式创建一个唯一标识符。
   - 这个标识符可以与 Django 的 `reverse()` 函数或 `{% url %}` 模板标签一起使用，以基于视图名称动态生成 URL，而不是硬编码 URL 路径。

2. **URL 命名空间**：

   - 为 URL 模式命名有助于通过提供命名空间 URL 的方式来避免具有多个应用的大型项目中的冲突。
   - 当包含来自不同应用的 URL 时，这特别有用。

3. **模板使用**：

   - 在 Django 模板中，你可以使用 `{% url %}` 模板标签以及 URL 名称来生成 URL。

## 路径转换器

URL 模式可能包含诸如 `<str:app>` 或 `<str:plugin>` 之类的表达式。它们被称为"路径转换器"，捕获传递给视图的关键字参数。它们遵循 `<converter:name>` 的语法，其中 `converter` 指定数据类型，`name` 是变量名。

让我们看 `nautobot -> extras -> plugins -> urls.py` 作为示例：

```python
...
path("installed-apps/<str:app>/", views.InstalledAppDetailView.as_view(), name="app_detail")
...
path("installed-plugins/<str:plugin>/", views.InstalledAppDetailView.as_view(), name="plugin_detail"),
...
```

`<str:app>` 意味着我们正在传递一个名为 `app` 的 `str` 变量给 `InstalledAppDetailView`：

![url_match_variable](images/url_match_variable.png)

这允许我们在不硬编码每个单一模式的情况下灵活地传递变量。

以上就是关于 URL 分派的内容。恭喜完成第 76 天！

## 第 76 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布今天挑战中我们查看的 URL 模式的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将学习 Nautobot 表单。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+76+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 76 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）