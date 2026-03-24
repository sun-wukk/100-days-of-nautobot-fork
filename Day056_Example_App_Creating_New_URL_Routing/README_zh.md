# 示例应用创建新 URL 路由

让我们添加一个 URL 模式来匹配视图，以便用户可以看到页面。

## urls.py

URL 文件位于与 `views.py` 同一级别，名为 `urls.py`：

```
@ericchou1 ➜ ~/nautobot/examples (develop) $ tree example_app/example_app/
example_app/example_app/
├── admin.py
...
├── models.py
├── navigation.py
├── templates
│   └── example_app
│       ├── anotherexamplemodel_retrieve.html
│       ├── config.html
│       ├── custom_job_form.html
│       ├── examplemodel_custom_action_get_all_example_model_names.html
│       ├── examplemodel_retrieve.html
│       ├── example_with_custom_template.html
│       ├── home.html
│       ├── inc
│       │   ├── item_example.html
│       │   └── panel_example.html
│       ├── panel.html
│       ├── tab_circuit_detail.html
│       ├── tab_device_detail_1.html
│       ├── tab_device_detail_2.html
│       └── useful_link_detail.html
├── urls.py
└── views.py

10 directories, 88 files
```

让我们添加一个 `useful-links/` 模式

```python url.py
from django.urls import path
...
urlpatterns = [
    ...
    path("useful-links/", views.UsefulLinkListView.as_view(), name="usefullinks_list"),
    ...
]
```

`name="usefullinks_list"` 为这个 URL 模式分配了一个名称，以便代码的其他部分引用，这样我们就不需要输入整个路径。

为了以防万一，这里是 `urls.py` 的完整内容：

```python urls.py
from django.templatetags.static import static
from django.urls import path
from django.views.generic import RedirectView

from nautobot.apps.urls import NautobotUIViewSetRouter

from example_app import views

app_name = "example_app"
router = NautobotUIViewSetRouter()
# ExampleModel 使用 ViewSet 注册
router.register("models", views.ExampleModelUIViewSet)
router.register("other-models", views.AnotherExampleModelUIViewSet)

urlpatterns = [
    path("", views.ExampleAppHomeView.as_view(), name="home"),
    path("useful-links/", views.UsefulLinkListView.as_view(), name="usefullinks_list"),
    path("config/", views.ExampleAppConfigView.as_view(), name="config"),
    path(
        "docs/",
        RedirectView.as_view(url=static("example_app/docs/index.html")),
        name="docs",
    ),
    # 仍然可以为使用 NautobotUIViewSet 的模型添加路由。
    path("circuits/<uuid:pk>/example-app-tab/", views.CircuitDetailAppTabView.as_view(), name="circuit_detail_tab"),
    path(
        "devices/<uuid:pk>/example-app-tab-1/",
        views.DeviceDetailAppTabOneView.as_view(),
        name="device_detail_tab_1",
    ),
    path(
        "devices/<uuid:pk>/example-app-tab-2/",
        views.DeviceDetailAppTabTwoView.as_view(),
        name="device_detail_tab_2",
    ),
    # 此 URL 定义在这里是为了测试 override_views 功能，该功能定义在
    # examples.example_app_with_view_override.example_app_with_view_override.views
    path("override-target/", views.ViewToBeOverridden.as_view(), name="view_to_be_overridden"),
    # 此 URL 定义在这里是为了测试 permission_classes 功能，该功能定义在 NautobotUIViewSetMixin 中
    path(
        "view-with-custom-permissions/",
        views.ViewWithCustomPermissions.as_view({"get": "list"}),
        name="view_with_custom_permissions",
    ),
]
urlpatterns += router.urls
```

现在我们可以看到页面了：

![final_view_1](images/final_view_1.png)

今天挑战的编码部分很简单。请用额外的时间查阅 Django 文档关于 [URL Dispatcher](https://docs.djangoproject.com/en/5.1/topics/http/urls/)。

然后回来看看 `<uuid:pk>` 部分会如何改变对 `views.DeviceDetailAppTabTwoView()` 函数的调用：

```python
    path(
        "devices/<uuid:pk>/example-app-tab-2/",
        views.DeviceDetailAppTabTwoView.as_view(),
        name="device_detail_tab_2",
    )
```

恭喜完成第 56 天！

## 第 56 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布今天挑战中关于 Django URL 调度所学到的内容，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将把这个 URL 模式添加到导航菜单中。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+56+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 56 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）