# 顶点项目第十二部分。第 91 天：将 CVE 模型集成到 Nautobot UI

## 目标

今天我们将 `CVE` 模型集成到 Nautobot UI 中，以启用完整的 CRUD 操作、列表、过滤、查看以及与 `SoftwareVersion` 对象的链接。

到今天结束时，你将拥有：
- CVE 的列表和详情视图
- 过滤和表单支持
- `SoftwareVersion` 对象下的 CVE 标签页
- CVE API 和导航菜单条目

由于我们的 CVE 模型是一个成熟的 Nautobot 数据模型，我们可以使用 Nautobot 的许多内置功能来定义使用它们的 UI。要获得完整功能，你需要为以下内容编写代码：数据表、过滤该数据的代码、编辑该数据的表单、显示该数据的模板，以及显示所有这些内容的视图。

## 1. 实现表

**文件**: `nautobot_software_cves/tables.py`

```python
import django_tables2 as tables
from nautobot.apps.tables import BaseTable, ButtonsColumn, ToggleColumn, TagColumn
from django.urls import reverse
from nautobot_software_cves import models

class CveStatusTable(BaseTable):
    # 不变，此处省略


class CVETable(BaseTable):
    """用于在 UI 中列出 CVE 的表。"""

    model = models.CVE
    pk = ToggleColumn()
    name = tables.Column(linkify=True)  # 允许点击 CVE 名称
    link = tables.TemplateColumn(
        template_code="""{% if record.link %}
            <a href="{{ record.link }}" target="_blank" data-toggle="tooltip" data-placement="left" title="{{ record.link }}">
                <span class="mdi mdi-open-in-new"></span>
            </a>
        {% else %}
            —
        {% endif %}""",
        verbose_name="Link",
    )
    actions = ButtonsColumn(models.CVE, buttons=("changelog", "edit", "delete"))
    tags = TagColumn(url_name="plugins:nautobot_software_cves:cve_list")

    class Meta(BaseTable.Meta):
        model = models.CVE
        fields = ("pk", "name", "link", "severity", "cvss", "affected_softwares", "actions")
        default_columns = ("pk", "name", "link", "severity", "cvss", "actions")
```

**说明**：
此表定义了 CVE 在列表视图中的显示方式。
- `ToggleColumn` 允许批量操作。
- 在 name 列上使用 `linkify=True` 使此列中的文本自动链接到相应对象的详情视图。
- `TemplateColumn` 格式化 CVE 链接使其美观。
- `ButtonsColumn` 添加用于变更日志/编辑/删除的 UI 按钮。
- `actions` 列提供表中各记录的快速编辑和删除按钮。
- `Meta 类`定义表适用于哪个模型，`fields` 属性定义默认情况下列的渲染顺序。通常，`pk` 列应始终是第一列，`actions` 列应是最后一列。
- 注意，我们没有在 **CVE 模型**上定义 `tags` 字段；这是从我们用作基类的 **PrimaryModel** 自动带到模型的，但我们确实需要明确将其包含在表中。

## 2. 实现过滤器集

**文件**: `nautobot_software_cves/filters.py`（如果你在第 80 天第 9 步选择了 "None" 则创建此文件）

```python
import django_filters
from nautobot.apps.filters import NautobotFilterSet, SearchFilter, TagFilter
from nautobot_software_cves.models import CVE

class CVEFilterSet(NautobotFilterSet):
    class Meta:
        model = CVE
        fields = ["name", "cvss"]

    q = SearchFilter(filter_predicates={"name": "icontains"})
    cvss__gte = django_filters.NumberFilter(field_name="cvss", lookup_expr="gte")
    cvss__lte = django_filters.NumberFilter(field_name="cvss", lookup_expr="lte")
    tags = TagFilter()
```

**说明**：
过滤器集可用于控制 UI 和 API 中数据的过滤。
`CVEFilterSet` 允许用户按名称搜索和过滤 CVE，或按 CVSS 基本分数范围过滤。

通过使用 **NautobotFilterSet**，自动支持按 **CVEs** 存在的任何 **关系** 和 **自定义字段** 过滤 CVE 表。此外，我们定义了自由文本搜索 CVE 表的工作方式（仅对 `name` 字段进行不区分大小写的搜索），并声明了按分配标签过滤的能力。

## 3. 实现表单

**文件**: `nautobot_software_cves/forms.py`（如果你在第 80 天第 9 步选择了 "None" 则创建此文件）

```python
from django import forms
from nautobot.apps.forms import NautobotModelForm, DynamicModelMultipleChoiceField
from nautobot.dcim.models import SoftwareVersion
from nautobot_software_cves.models import CVE, CVESeverityChoices

class CVEForm(NautobotModelForm):
    """用于创建和编辑 CVE 的表单。"""

    severity = forms.ChoiceField(choices=CVESeverityChoices.CHOICES, label="Severity", required=False)
    affected_softwares = DynamicModelMultipleChoiceField(queryset=SoftwareVersion.objects.all(), required=False)

    class Meta:
        model = CVE
        fields = "__all__"
```

**说明**：
表单在 UI 中用于添加或编辑 CVE。
- severity 是下拉菜单，受影响的软件版本可以动态链接。
- 我们再次使用全功能类 `NautobotModelForm` 用于 CVE 模型。
- 我们对 affected_softwares 使用 `DynamicModelMultipleChoiceField`；此类允许 Nautobot 动态填充这些字段的选项，而不是在最初渲染页面时预填充所有可用选项，从而使创建/编辑视图加载得更快。

## 4. 实现模板

**文件**: `nautobot_software_cves/templates/nautobot_software_cves/cve_retrieve.html`

```html
{% extends 'generic/object_retrieve.html' %}
{% load helpers %}

{% block content_left_page %}
<div class="panel panel-default">
    <div class="panel-heading">
        <strong>CVE</strong>
    </div>
    <table class="table table-hover panel-body attr-table">
        <tr style="font-weight: bold">
            <td>Name</td>
            <td>{{ object.name }}</td>
        </tr>
        <tr>
            <td>Link</td>
            <td><a href="{{ object.link }}" target="_blank">{{ object.link }}</td>
        </tr>
        <tr>
            <td>Severity</td>
            <td>{% if object.severity %} {{ object.severity }} {% else %} &mdash; {% endif %}</td>
        </tr>
        <tr>
            <td>CVSS Base Score</td>
            <td>{% if object.cvss %} {{ object.cvss }} {% else %} &mdash; {% endif %}</td>
        </tr>
        <tr>
            <td>Affected Softwares</td>
            <td>
                {% for affected_software in object.affected_softwares.all %}
                {{ affected_software|hyperlinked_object }}{% if not forloop.last %}, {% endif %}
                {% endfor %}
            </td>
        </tr>
    </table>
</div>
{% endblock content_left_page %}
```

**说明**：
自定义每个 CVE 的对象详情视图，显示所有相关信息的简单表格。
对于我们 CVE 模型的大多数基本视图，我们实际上根本不需要编写模板。Nautobot 的内置通用模板会做得很好。唯一的例外是详情视图，每个模型需要一个模板来处理渲染其独特字段。

在这里，我们扩展了通用 `generic/object_retrieve.html` 并加载了 Nautobot helpers 模块提供的附加模板标签集，其中包括 `hyperlinked_object` 标签，我们用它来自动超链接到关联 `SoftwareVersion` 对象的详情视图。
就像我们之前做的那样，我们没有覆盖整个模板，而只是覆盖了 `content_left_page` 块，它定义了渲染模板的左侧。

## 5. 实现序列化器

**文件**: `nautobot_software_cves/serializers.py`（如果你在第 80 天第 9 步选择了 "None" 则创建此文件）

```python
from nautobot.apps.api import NautobotModelSerializer
from nautobot_software_cves.models import CVE

class CVESerializer(NautobotModelSerializer):
    class Meta:
        model = CVE
        fields = ["__all__"]
```

**说明**：
序列化器通过 REST API 暴露 CVE 模型，但它也在 CVE 视图中使用。
序列化器是 django-rest-framework 可以轻松将 Django 模型转换为其 JSON 表示的方式，反之亦然。Nautobot 进一步定义了提供 Nautobot 特定功能自己的序列化器基类。

对于大多数基本数据模型，在 Nautobot 中创建 REST API 序列化器需要做的就是：
- 声明序列化器适用于哪个模型。每个模型都需要。
- 声明 fields = ["__all__"]，意思是"序列化此模型定义的所有字段"。只有在你想要序列化超出模型数据库字段的额外数据，或者想要明确排除某些字段（如可能包含敏感数据的字段）时，才需要更改此设置。

## 6. 实现视图

**文件**: `nautobot_software_cves/views.py`

```python
from nautobot.apps import views
from nautobot.dcim.models import SoftwareVersion
from nautobot.dcim.filters import SoftwareVersionFilterSet
from nautobot_software_cves.tables import CveStatusTable
from nautobot_software_cves import filters, forms, models, tables
from nautobot_software_cves import serializers as software_cves_serializers


class SoftwareCvesView(views.ObjectView):
# 不变，此处省略

class SoftwareCvesStatusViewSet(views.ObjectListViewMixin):
# 不变，此处省略

class CVEUIViewSet(views.NautobotUIViewSet):
    filterset_class = filters.CVEFilterSet
    form_class = forms.CVEForm
    lookup_field = "pk"
    queryset = models.CVE.objects.all()
    serializer_class = software_cves_serializers.CVESerializer
    table_class = tables.CVETable
```

**说明**：
`CVEUIViewSet` 将所有 UI 组件（表单、表、过滤器）绑定在一起。

## 7. 实现 URL

**文件**: `nautobot_software_cves/urls.py`

```python
from django.urls import path
from django.views.generic import RedirectView
from django.templatetags.static import static
from nautobot.apps.urls import NautobotUIViewSetRouter
from nautobot_software_cves import views

router = NautobotUIViewSetRouter()
router.register("softwareversions", views.SoftwareCvesStatusViewSet)
router.register("cves", views.CVEUIViewSet) # CVE 模型的新路由

urlpatterns = [
    path(
        "docs/",
        RedirectView.as_view(
            url=static("nautobot_software_cves/docs/index.html")
        ),
        name="docs"
    ),
    path(
        "softwareversions/<uuid:pk>/cves/",
        views.SoftwareCvesView.as_view(),
        name="software_cves",
    ),
]

urlpatterns += router.urls
```

**说明**：
这为 CVE 设置了路由。

## 8. 实现导航菜单

**文件**: `nautobot_software_cves/navigation.py`

```python
from nautobot.apps.ui import NavMenuGroup, NavMenuItem, NavMenuTab

menu_items = (
    NavMenuTab(
        name="Devices",
        groups=(
            NavMenuGroup(
                name="Software",
                items=(
                    NavMenuItem(
                        link="plugins:nautobot_software_cves:softwareversion_list",
                        name="CVE Status",
                        permissions=["dcim.view_softwareversion"],
                    ),
                ),
            ),
        ),
    ),
    NavMenuTab(
        name="CVE Tracking",
        groups=(
            NavMenuGroup(
                name="CVEs",
                items=(
                    NavMenuItem(
                        link="plugins:nautobot_software_cves:cve_list",
                        name="CVEs",
                        permissions=["nautobot_software_cves.cve"],
                    ),
                ),
            ),
        ),
    ),
)
```

**说明**：
创建一个名为 CVE Tracking 的新菜单项，用于快速导航到 CVE。

## ✅ 验证步骤

实现上述所有内容后：

1. 停止 Nautobot 并运行 invoke debug
2. 在 UI 中导航到 `CVE TRACKING > CVEs`。
   ![cve_nav_menu](images/cve_nav_menu.png)
1. 点击 **Add CVE** 创建一个新的 CVE 条目。
   - 为 name（CVE-2024-20510）、link（https://sec.cloudapps.cisco.com/security/center/content/CiscoSecurityAdvisory/cisco-sa-c9800-cwa-acl-nPSbHSnA）、severity（Medium）、CVSS score（4.7）提供值，并将其链接到一个或多个 `SoftwareVersions`（Cisco IOS-XE - 17.7.2）。
   ![add_cve_1](images/add_cve_1.png)
   ![add_cve_2](images/add_cve_2.png)
2. 保存后：
   - 转到 **CVEs** 查看你的新条目。
   - 点击它打开 **CVE 详情视图**。
   - 确认每个字段的值正确。
   ![cve_list](images/cve_list.png)
   ![cve_details](images/cve_details.png)

这就是第 91 天的全部内容！你现在有了一个功能完整的 CVE 管理界面，集成到 Nautobot 的 UI 和 API 中。