# 示例应用创建新视图 - 第二部分

在今天的挑战中，我们将为新的数据模型创建一个新的视图。

## 新的 views.py 代码

正如昨天提到的，我们将保持视图非常简单。我们将导入数据模型，使用 Django 的 `ListView`，并用一个尚未创建的 HTML 模板来显示它，这个模板被创意性地命名为 `useful_link_detail.html`：

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

为了以防万一，这里是 `views.py` 文件的完整内容：

```python views.py
from django.shortcuts import HttpResponse, render
from django.utils.html import format_html
from rest_framework.decorators import action
from rest_framework.permissions import IsAdminUser, IsAuthenticated

from nautobot.apps import ui, views
from nautobot.circuits.models import Circuit
from nautobot.circuits.tables import CircuitTable
from nautobot.circuits.views import CircuitUIViewSet
from nautobot.core.ui.object_detail import TextPanel
from nautobot.dcim.models import Device

from example_app import filters, forms, tables
from example_app.api import serializers
from example_app.models import AnotherExampleModel, ExampleModel, UsefulLink

from django.views.generic import ListView


class UsefulLinkListView(ListView):
    model = UsefulLink
    template_name = "example_app/useful_link_detail.html"
    context_object_name = "useful_links"

    def get_queryset(self):
        return UsefulLink.objects.all()


class CircuitDetailAppTabView(views.ObjectView):
    """
    此视图的模板扩展了电路详情模板，
    适合作为电路详情页面的标签页显示。

    用作对象详情标签页内容渲染的视图必须
    始终继承自 nautobot.apps.views.ObjectView。
    """

    queryset = Circuit.objects.all()
    template_name = "example_app/tab_circuit_detail.html"


class DeviceDetailAppTabOneView(views.ObjectView):
    """
    此视图的模板扩展了设备详情模板，
    适合作为设备详情页面的标签页显示。

    用作对象详情标签页内容渲染的视图必须
    始终继承自 nautobot.apps.views.ObjectView。
    """

    queryset = Device.objects.all()
    template_name = "example_app/tab_device_detail_1.html"


class DeviceDetailAppTabTwoView(views.ObjectView):
    """
    与上面的 DeviceDetailAppTabOneView 视图相同，但使用不同的模板。
    """

    queryset = Device.objects.all()
    template_name = "example_app/tab_device_detail_2.html"


class ExampleAppHomeView(views.GenericView):
    def get(self, request):
        return render(request, "example_app/home.html")


class ExampleAppConfigView(views.GenericView):
    def get(self, request):
        """渲染此应用的应用配置页面。

        只是一个示例 - 实际上你可能需要根据应用的实际情况使用真实的配置数据（如果有的话）。
        """
        form = forms.ExampleAppConfigForm({"magic_word": "frobozz", "maximum_velocity": 300000})
        return render(request, "example_app/config.html", {"form": form})

    def post(self, request):
        """处理此应用的应用配置更改。

        这里没有实际实现。
        """
        form = forms.ExampleAppConfigForm({"magic_word": "frobozz", "maximum_velocity": 300000})
        return render(request, "example_app/config.html", {"form": form})


class ExampleModelUIViewSet(views.NautobotUIViewSet):
    bulk_update_form_class = forms.ExampleModelBulkEditForm
    filterset_class = filters.ExampleModelFilterSet
    filterset_form_class = forms.ExampleModelFilterForm
    form_class = forms.ExampleModelForm
    queryset = ExampleModel.objects.all()
    serializer_class = serializers.ExampleModelSerializer
    table_class = tables.ExampleModelTable
    object_detail_content = ui.ObjectDetailContent(
        panels=(
            ui.ObjectFieldsPanel(
                section=ui.SectionChoices.LEFT_HALF,
                weight=100,
                fields="__all__",
            ),
            # 一个动态派生的对象表，在 `get_extra_context()` 中创建
            ui.ObjectsTablePanel(
                section=ui.SectionChoices.RIGHT_HALF,
                weight=100,
                context_table_key="dynamic_table",
                max_display_count=3,
            ),
            # 一个具有静态定义列的非对象数据表
            ui.DataTablePanel(
                section=ui.SectionChoices.RIGHT_HALF,
                label="自定义表格 1 - 具有动态数据和硬编码列",
                weight=200,
                context_data_key="data_1",
                columns=["col_1", "col_2", "col_3"],
                column_headers=["列 1", "列 2", "列 3"],
            ),
            # 一个具有动态（渲染时）列的非对象数据表
            ui.DataTablePanel(
                section=ui.SectionChoices.FULL_WIDTH,
                label="自定义表格 2 - 具有动态数据和动态列",
                weight=100,
                context_data_key="data_2",
                context_columns_key="columns_2",
                context_column_headers_key="column_headers_2",
            ),
            ui.TextPanel(
                section=ui.SectionChoices.LEFT_HALF,
                label="带有 JSON 的文本面板",
                weight=300,
                context_field="text_panel_content",
                render_as=TextPanel.RenderOptions.JSON,
            ),
            ui.TextPanel(
                section=ui.SectionChoices.LEFT_HALF,
                label="带有 YAML 的文本面板",
                weight=300,
                context_field="text_panel_content",
                render_as=TextPanel.RenderOptions.YAML,
            ),
            ui.TextPanel(
                section=ui.SectionChoices.RIGHT_HALF,
                label="带有 PRE 标签用法的文本面板",
                weight=300,
                context_field="text_panel_code_content",
                render_as=TextPanel.RenderOptions.CODE,
            ),
        ),
    )

    def get_extra_context(self, request, instance):
        context = super().get_extra_context(request, instance)
        if self.action == "retrieve":
            # 为自定义面板添加动态对象表
            context["dynamic_table"] = CircuitTable(Circuit.objects.restrict(request.user, "view"))
            # 为对象详情视图自定义表添加非对象数据
            context["data_1"] = [
                # 因为上面定义的 DataTablePanel 指定了 `columns`，col_4 数据不会显示
                {"col_1": "value_1a", "col_2": "value_2", "col_3": "value_3", "col_4": "not shown"},
                # 演示空值和缺失列数据被安全/正确处理
                {"col_1": "value_1b", "col_2": None},
            ]
            # 一些用于渲染的任意数据
            # 动态指定此数据表的列和列标题，而不是在声明时
            context["columns_2"] = ["a", "e", "i", "o", "u"]
            context["column_headers_2"] = ["A", "E", "I", "O", "U"]
            context["data_2"] = [
                {
                    # 列值可以包含适当构造的 HTML
                    "a": format_html('<a href="https://en.wikipedia.org/wiki/{val}">{val}</a>', val="a"),
                    # 不当构造的 HTML 会在渲染时被适当转义
                    "e": '<a href="https://example.org/evil-link/e/">e</a>',
                    # Unicode 被正确处理
                    "i": "ℹ︎",  # noqa:RUF001 - 刻意的类字母 Unicode
                    "o": "º",
                    "u": "µ",
                },
                # 如上所述，不匹配特定 `columns` 条目的数据不会被渲染
                {"a": 97, "b": 98, "c": 99, "e": 101, "i": 105, "o": 111, "u": 17},
                {"a": "0x61", "b": "0x62", "c": "0x63", "e": "0x65", "i": "0x69", "o": "0x6f", "u": "0x75"},
                {
                    "u": 21 + instance.number,
                    "o": 15 + instance.number,
                    "i": 9 + instance.number,
                    "e": 5 + instance.number,
                    "a": 1 + instance.number,
                },
            ]
            # 为 TextPanel 添加数据
            context["text_panel_content"] = {
                "device_name": "Router1",
                "ip_address": "192.168.1.1",
                "subnet_mask": "255.255.255.0",
                "gateway": "192.168.1.254",
                "interfaces": [
                    {
                        "interface_name": "GigabitEthernet0/0",
                        "ip_address": "10.0.0.1",
                        "subnet_mask": "255.255.255.252",
                        "mac_address": "00:1A:2B:3C:4D:5E",
                    },
                ],
            }
            context["text_panel_code_content"] = 'import abc\nabc()\nprint("Hello world!")'

        return context

    @action(detail=False, name="All Names", methods=["get"], url_path="all-names", url_name="all_names")
    def all_names(self, request):
        """
        返回所有示例模型名称的列表。
        """
        all_example_models = self.get_queryset()
        return render(
            request,
            "example_app/examplemodel_custom_action_get_all_example_model_names.html",
            {"data": [model.name for model in all_example_models]},
        )


# 示例排除 BulkUpdateViewSet
class AnotherExampleModelUIViewSet(
    views.ObjectBulkDestroyViewMixin,
    views.ObjectBulkUpdateViewMixin,
    views.ObjectChangeLogViewMixin,
    views.ObjectNotesViewMixin,
    views.ObjectDestroyViewMixin,
    views.ObjectDetailViewMixin,
    views.ObjectEditViewMixin,
    views.ObjectListViewMixin,
):
    action_buttons = ["add", "export"]
    bulk_update_form_class = forms.AnotherExampleModelBulkEditForm
    filterset_class = filters.AnotherExampleModelFilterSet
    filterset_form_class = forms.AnotherExampleModelFilterForm
    create_form_class = forms.AnotherExampleModelCreateForm
    update_form_class = forms.AnotherExampleModelUpdateForm
    lookup_field = "pk"
    queryset = AnotherExampleModel.objects.all()
    serializer_class = serializers.AnotherExampleModelSerializer
    table_class = tables.AnotherExampleModelTable


class ViewToBeOverridden(views.GenericView):
    def get(self, request, *args, **kwargs):
        return HttpResponse("我是 example 应用中的一个视图，将被另一个应用覆盖。")


class ViewWithCustomPermissions(views.ObjectListViewMixin):
    permission_classes = [IsAuthenticated, IsAdminUser]
    filterset_class = filters.ExampleModelFilterSet
    queryset = ExampleModel.objects.all()
    serializer_class = serializers.ExampleModelSerializer
    table_class = tables.ExampleModelTable


override_views = {
    "circuits:circuit_list": CircuitUIViewSet.as_view({"get": "list"})  # 用于测试 override_views
}
```

今天的挑战代码很简单，请用剩余时间学习 `views.py` 文件，并参考 Nautobot 文档来了解代码的功能。

## 接下来的内容

快速提问，既然我们有了模型和视图，我们能在 Web UI 上看到任何内容吗？答案是不能。

"为什么不？"

嗯，首先，我们仍然需要一个 HTML 模板来显示链接。另外，我们应该使用哪个 URL 来查看该模板？这两个步骤就是我们接下来几天要做的事情。

## 第 54 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。