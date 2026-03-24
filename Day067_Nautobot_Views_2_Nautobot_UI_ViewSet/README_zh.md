# Nautobot 类视图

Nautobot 利用类视图来提供更有组织和更易维护的方式处理 HTTP 请求。在 Nautobot 中，类视图被广泛用于处理各种操作，如显示列表、处理表单和执行数据库操作。

在今天的挑战中，我们将介绍 Nautobot 对 Nautobot 应用使用类视图的方法。特别是，我们将讨论 `NautobotUIViewSetRouter` 和 `NautobotUIViewSet`，在今天的挑战中涉及的主题时，以下相关文档可能会有帮助：

- [NautobotUIVewSetRouter](https://docs.nautobot.com/projects/core/en/stable/development/apps/api/views/nautobotuiviewsetrouter/)
- [NautobotUIViewSet](https://docs.nautobot.com/projects/core/en/stable/development/apps/api/views/nautobotuiviewset/)

## 环境设置

今天的挑战没有动手操作。请随意结合使用 [场景 2](../Lab_Setup/scenario_2_setup/README.md) 实验室和 [Nautbot 文档](https://docs.nautobot.com/projects/core/en/stable/development/apps/) 来尝试今天的挑战代码。

```
$ cd nautobot
$ poetry shell
$ poetry install
$ invoke build
（这个步骤需要耐心）
$ invoke debug
（这个步骤也需要耐心）
```

### NautobotUIViewSetRouter

Nautobot 通过 `NautobotUIViewSet` 及其组件 `Mixins` 广泛使用类视图。

> [!提示]
> [Django 中的 Mixins](https://docs.djangoproject.com/en/5.1/topics/class-based-views/mixins/) 是一种多重继承形式，允许在多个类之间重用代码。

`NautobotUIViewSetRouter` 是在 [Nautobot Apps urls.py](https://github.com/nautobot/nautobot/blob/develop/nautobot/apps/urls.py) 中创建和注册 URL 模式的关键组件：

```
"""Utilities for apps to implement URL routing."""

from nautobot.core.views.routers import NautobotUIViewSetRouter

__all__ = ("NautobotUIViewSetRouter",)
```

然后在 [https://github.com/nautobot/nautobot/blob/develop/nautobot/core/views/routers.py](https://github.com/nautobot/nautobot/blob/develop/nautobot/core/views/routers.py) 中匹配模式。匹配的 URL 模式将被定向到应用的标准视图或 `NautobotUIViewSet`。

正如 [App 开发者指南 -> 视图 -> NautobotUIViewSetRouter](https://docs.nautobot.com/projects/core/en/stable/development/apps/api/views/nautobotuiviewsetrouter/) 中所解释的，我们需要使用 `NautobotUIViewSetRouter` 注册应用的模型和视图。

以下是一个 Nautobot 应用的理论 `urls.py` 文件：

```python
from django.urls import path

from nautobot.apps.urls import NautobotUIViewSetRouter
from your_app import views


router = NautobotUIViewSetRouter()
router.register("yourappmodel", views.YourAppModelUIViewSet)

urlpatterns = [
    # 不遵循 `NautobotUIViewSetRouter` 模式的额外 URL 在这里。
    # changelog, notes 等。
    ...
    path(
        "yourappmodels/<uuid:pk>/changelog/",
        ObjectChangeLogView.as_view(),
        name="yourappmodel_changelog",
        kwargs={"model": yourappmodel},
    ),
    path(
        "yourappmodels/<uuid:pk>/notes/",
        ObjectNotesView.as_view(),
        name="yourappmodel_notes",
        kwargs={"model": yourappmodel},
    ),
    ...
]
urlpatterns += router.urls
```

对于应用 `urls.py` 的示例，我们可以查看 [nautobot -> circuits -> urls.py](https://github.com/nautobot/nautobot/blob/develop/nautobot/circuits/urls.py)：

```python
from django.urls import path

from nautobot.core.views.routers import NautobotUIViewSetRouter
from nautobot.dcim.views import CableCreateView, PathTraceView

from . import views
from .models import CircuitTermination

app_name = "circuits"
router = NautobotUIViewSetRouter()
router.register("providers", views.ProviderUIViewSet)
router.register("provider-networks", views.ProviderNetworkUIViewSet)
router.register("circuit-types", views.CircuitTypeUIViewSet)
router.register("circuits", views.CircuitUIViewSet)
router.register("circuit-terminations", views.CircuitTerminationUIViewSet)

urlpatterns = [
    path(
        "circuits/<uuid:pk>/terminations/swap/",
        views.CircuitSwapTerminations.as_view(),
        name="circuit_terminations_swap",
    ),
    path(
        "circuits/<uuid:circuit>/terminations/add/",
        views.CircuitTerminationUIViewSet.as_view({"get": "create", "post": "create"}),
        name="circuittermination_add",
    ),
    path(
        "circuit-terminations/<uuid:termination_a_id>/connect/<str:termination_b_type>/",
        CableCreateView.as_view(),
        name="circuittermination_connect",
        kwargs={"termination_a_type": CircuitTermination},
    ),
    path(
        "circuit-terminations/<uuid:pk>/trace/",
        PathTraceView.as_view(),
        name="circuittermination_trace",
        kwargs={"model": CircuitTermination},
    ),
]
urlpatterns += router.urls
```

我们可以在 [demo.nautobot.com](https://demo.nautobot.com/) 上尝试 URL 映射：

![circuit_providers](images/circuit_providers.png)

在下一节中，我们将看看 `NautobotUIViewSet` 提供的一些视图及其相关的 `Mixins`。

### NautobotUIViewSet Mixins

[NautobotUIViewSet](https://docs.nautobot.com/projects/core/en/stable/development/apps/api/views/nautobotuiviewset/) 在 Nautobot 1.4 中引入，旨在节省应用开发者的和时间。

`NautobotUIViewSet` 视图可以从 [Nautobot -> Apps -> Views.py](https://github.com/nautobot/nautobot/blob/develop/nautobot/apps/views.py) 中获取。

一些常见的 `NautobotUIViewSet` `Mixins`：

1. **ObjectListViewMixin**：显示模型的列表视图。
2. **ObjectDetailViewMixin**：显示单个对象的详细视图。
3. **ObjectEditViewMixin**：模型的编辑视图。
4. **ObjectDestroyViewMixin**：模型的删除视图。
5. **ObjectBuildUpdateViewMixin**：模型的批量编辑视图。
6. **ObjectBulkDestroyViewMixin**：模型的批量删除视图。

与 URL 映射相关，我们可以查看 `views.ProviderUIViewSet`：

```python
class ProviderUIViewSet(NautobotUIViewSet):
    bulk_update_form_class = forms.ProviderBulkEditForm
    filterset_class = filters.ProviderFilterSet
    filterset_form_class = forms.ProviderFilterForm
    form_class = forms.ProviderForm
    queryset = Provider.objects.all()
    serializer_class = serializers.ProviderSerializer
    table_class = tables.ProviderTable
    object_detail_content = ObjectDetailContent(
        panels=(
            ObjectFieldsPanel(
                section=SectionChoices.LEFT_HALF,
                weight=100,
                fields="__all__",
            ),
            ObjectsTablePanel(
                weight=200,
                table_class=tables.CircuitTable,
                table_filter="provider",
                section=SectionChoices.FULL_WIDTH,
                exclude_columns=["provider"],
            ),
        ),
    )
```

在这个示例中，通过几行代码，大部分数据库操作被卸载到 `NautobotUIViewSet`。

## 资源

- [NautobotUIVewSetRouter](https://docs.nautobot.com/projects/core/en/stable/development/apps/api/views/nautobotuiviewsetrouter/)
- [NautobotUIViewSet](https://docs.nautobot.com/projects/core/en/stable/development/apps/api/views/nautobotuiviewset/)

恭喜完成第 67 天！

## 第 67 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布关于 `NautobotUIViewSet` 以及你从今天挑战中学到的内容，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将看看 Nautobot 中提供的其他视图。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+67+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 67 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）