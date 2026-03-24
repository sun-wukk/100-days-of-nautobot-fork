# 页面面板和面板项

从今天的挑战开始，在接下来的几天里，我们将开始讨论 Nautobot 中的用户界面。

从之前的练习中，我们已经看到了模型、视图、模板和 URL 之间的关系。

以下是一个简化的工作流程：

1. 定义具有字段和关系的模型。
2. 创建引用模型和相关模板的视图。
3. 创建模板。
4. 创建指向视图的 URL。

Django 和 Nautobot 的当前趋势是减少过程中涉及的原始 HTML。这遵循了 `DRY（不要重复自己）` 原则，使表示更加一致。

我们将从面板和面板项开始讨论。

今天的主要目标是使用广泛使用的对象（面板和面板项）来说明如何使用我们到目前为止学到的东西并在我们的沙箱中遍历代码。我们还将进行一些trivial的更改来加强学习。

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

## 面板和面板项

我们在 Nautobot 主页上看到的第一件事是成组的 `面板`，其中包含 `面板项` 列表：

![panel_and_panel_items](images/panel_and_panel_items.png)

如果我们想对面板的外观进行一些更改，我们应该从哪里开始？以下是几个可以开始的地方。

## 查看代码

首先查看端点和相关的 `view` 代码。我们可以从 `nautobot -> core -> urls.py` 开始。我们可以看到根目录的 *urlpattern* 是 `HomeView`，它是从 `nautobot.core.views` 导入的：

```python nautobot.core.urls.py
from nautobot.core.views import (
    AboutView,
    CustomGraphQLView,
    get_file_with_authorization,
    HomeView,
    NautobotMetricsView,
    NautobotMetricsViewAuth,
    RenderJinjaView,
    SearchView,
    StaticMediaFailureView,
    ThemePreviewView,
    WorkerStatusView,
)
...
urlpatterns = [
    # Base views
    path("", HomeView.as_view(), name="home"),
    ...
]
```

当我们在 `nautobot.core` 下查看 `views` 文件夹时，我们没有看到与 `HomeView` 对应的单独文件。但是在查看 `__init__.py` 后，我们找到了 `HomeView` 的类视图：

```python nautobot.core.views.__init__.py
class HomeView(AccessMixin, TemplateView):
    template_name = "home.html"

    def render_additional_content(self, request, context, details):
        # 使用回调函数收集所有自定义数据。
        for key, data in details.get("custom_data", {}).items():
            if callable(data):
                context[key] = data(request)
            else:
                context[key] = data

        # 创建独立模板
        path = f'{details["template_path"]}{details["custom_template"]}'
        if os.path.isfile(path):
            with open(path, "r") as f:
                html = f.read()
        else:
            raise TemplateDoesNotExist(path)

        template = Template(html)

        additional_context = RequestContext(request, context)
        return template.render(additional_context)

    def get(self, request, *args, **kwargs):
        # 如果未认证，重定向用户到登录页面
        if not request.user.is_authenticated:
            return self.handle_no_permission()
        # 检查是否有新版本可用。（仅适用于 staff/superusers。）
        new_release = None
        if request.user.is_staff or request.user.is_superuser:
            latest_release, release_url = get_latest_release()
            if isinstance(latest_release, version.Version):
                current_version = version.parse(settings.VERSION)
                if latest_release > current_version:
                    new_release = {
                        "version": str(latest_release),
                        "url": release_url,
                    }

        context = self.get_context_data()
        context.update(
            {
                "search_form": SearchForm(),
                "new_release": new_release,
            }
        )

        # 循环遍历首页布局以收集所有额外数据并创建自定义面板。
        for panel_details in registry["homepage_layout"]["panels"].values():
            if panel_details.get("custom_template"):
                panel_details["rendered_html"] = self.render_additional_content(request, context, panel_details)

            else:
                for item_details in panel_details["items"].values():
                    if item_details.get("custom_template"):
                        item_details["rendered_html"] = self.render_additional_content(request, context, item_details)

                    elif item_details.get("model"):
                        # 如果有附加模型，收集对象计数。
                        item_details["count"] = item_details["model"].objects.restrict(request.user, "view").count()

                    elif item_details.get("items"):
                        # 收集分组对象的计数。
                        for group_item_details in item_details["items"].values():
                            if group_item_details.get("custom_template"):
                                group_item_details["rendered_html"] = self.render_additional_content(
                                    request, context, group_item_details
                                )
                            elif group_item_details.get("model"):
                                group_item_details["count"] = (
                                    group_item_details["model"].objects.restrict(request.user, "view").count()
                                )

        return self.render_to_response(context)
```

我们从行 `template_name = "home.html"` 知道正在渲染的模板。

## 示例 - HTML

让我们在 `core -> templates -> home.html` 文件中进行一个简单的更改。我们可以找到 `panel_name` 的 `div` 类并注释掉 `<string>` 标签，用红色的 `span style` 替换它：

![panel_name_1](images/panel_name_1.png)

```HTML
 <div class="panel-heading">
        <!-- <strong>{{ panel_name }}</strong> -->
        <span style="color: red;">{{ panel_name }}</span>
</div>
```

如果我们刷新主页，我们会看到面板名称从原来的黑色：

![panel_name_original](images/panel_name_original.png)

...变成红色：

![panel_name_red](images/panel_name_red.png)

这是我们如何在模板中进行更改以影响用户界面的一个简单示例。

## 示例 - 视图

我们也可以尝试从 `view` 的代码中进行更改。从代码中，我们可以看到在 `panel item` 中，模型计数是动态查询并返回的。

![panel_count_original](images/item_count_original.png)

当我们回到 `nautobot.core` 下 `views` 文件夹中的 `__init__.py`。我们可以进行一个简单的更改为静态值，在这个例子中，我们将其硬编码为 `100`：

![panel_count_code](images/item_count_code.png)

最终结果可以在页面上查看：

![panel_count_changed](images/item_count_changed.png)

正如伟大的 `老子` 所说：

**授人以鱼不如授人以渔。**

希望从今天挑战的步骤中，它打开了如果我们要进行模板和用户界面更改可以采取的步骤的大门。

恭喜完成第 71 天！

## 第 71 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布今天挑战中你所做更改的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将研究可搜索模型。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+71+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 71 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）