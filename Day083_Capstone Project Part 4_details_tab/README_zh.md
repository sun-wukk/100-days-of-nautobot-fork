# 顶点项目第四部分。CVE 管理 Nautobot 应用 - 第 83 天

## **目标**

在**第 82 天**，我们通过在右侧面板上显示 CVE 将 CVE 信息添加到了 **SoftwareVersion** 详情视图。然而，为了进一步增强可用性，我们现在将向 **SoftwareVersion** 视图添加一个**专用的 CVE 详情标签页**。

此标签页将允许用户在单独的视图中以结构化格式查看 CVE 数据，而不是仅仅在侧面板中。

## 环境设置

对于第 80-89 天的顶点项目，我们将使用我们一直使用的 [场景 2](../Lab_Setup/scenario_2_setup/README.md) 实验室和 Codespace。

假设我们建立在之前的进度上，我们需要使用 `poetry shell` 启用虚拟环境，并使用 `invoke debug` 启动环境：

```
@ericchou1 ➜ ~ $ cd nautobot-app-software-cves/
@ericchou1 ➜ ~/nautobot-app-software-cves $ poetry shell
(nautobot-software-cves-py3.10) @ericchou1 ➜ ~/nautobot-app-software-cves $ invoke debug
...
nautobot-1  | Django version 4.2.20, using settings 'nautobot_config'
nautobot-1  | Starting development server at http://0.0.0.0:8080/
nautobot-1  | Quit the server with CONTROL-C.
...
```

## **实施步骤**

### **1. 在 `template_content.py` 中添加 CVE 详情标签页**

我们扩展 `SoftwareVersionTemplateExtension` 类以向 **SoftwareVersion** 详情视图添加一个新的标签页。这是使用 `detail_tabs` 方法完成的。

- `detail_tabs` 为指定模型（`dcim.softwareversion`）向 Nautobot UI 添加**新标签页**。
- 它允许用户访问更详细的 CVE 信息而不使主视图显得杂乱。
- 该标签页将链接到一个我们接下来将定义的**专用视图**。

代码工作原理如下：
- 它在 SoftwareVersion 详情页面添加了一个 "CVEs" 标签页。
- 此标签页使用 reverse() 函数链接到 `/softwareversions/<uuid:pk>/cves/`。
- 在 Django 中，reverse() 通过解析相应的视图名称和可选参数从命名路由动态生成 URL。

#### 在 `template_content.py` 中插入以下代码。**right_page** 保持不变。

```python
"""Module to change object details view."""

from django.urls import reverse #new
from nautobot.apps.ui import TemplateExtension

class SoftwareVersionTemplateExtension(TemplateExtension):
    """Add CVE information to the Nautobot Software Version detail view."""

    model = "dcim.softwareversion"

    def right_page(self):
        """Add content on the right side of the view."""

    def detail_tabs(self): #new
        """Add a CVE details tab to the SoftwareVersion view."""
        return [
            {
                "title": "CVEs",
                "url": reverse(
                    "plugins:nautobot_software_cves:software_cves",
                    kwargs={"pk": self.context["object"].pk}
                ),
            },
        ]

# Register the template extension so Nautobot applies it
template_extensions = [SoftwareVersionTemplateExtension]
```

### **2. 在 `urls.py` 中定义 URL**

我们需要创建一个新 URL，因为上面添加的 **detail tab** 需要一个目标。此 URL 将映射到一个显示 SoftwareVersion 详细 CVE 数据的新**视图**。

带有名称 **software_cves** 的新 URL：
  - 将 `/softwareversions/<uuid:pk>/cves/` 映射到 `SoftwareCvesView`。
  - 确保正确的 SoftwareVersion 对象被传递到视图。

#### 在 `urls.py` 中插入以下代码来定义 URL：

```python
from django.urls import path
from nautobot_software_cves import views

urlpatterns = [
    path(
        "softwareversions/<uuid:pk>/cves/",
        views.SoftwareCvesView.as_view(),
        name="software_cves",
    ),
]
```

### **3. 在 `views.py` 中创建视图**

新视图将处理显示特定 SoftwareVersion 的 CVE 详情的请求。它从**自定义字段**中提取数据，并使用模板以有组织的方式呈现。

- 使用 `queryset` 检索 **SoftwareVersion** 对象。
- 在 `template_name` 属性中指定要呈现的模板。
- 使用 `get_extra_context` 将 **CVE 数据**传递给模板。

#### 在 **`nautobot_software_cves/`** 目录中，找到 `views.py` 或如果你在第 80 天第 9 步选择了 "None" 则创建它，并插入以下代码来定义新视图：

```python
from nautobot.apps import views
from nautobot.dcim.models import SoftwareVersion

class SoftwareCvesView(views.ObjectView):
    queryset = SoftwareVersion.objects.all()
    template_name = "nautobot_software_cves/software_cves.html"

    def get_extra_context(self, request, instance):
        return {"cves": instance.custom_field_data.get("cves", {})}
```

### **4. 为 CVE 详情创建模板**

- **SoftwareCvesView** 需要一个关联的 HTML 模板来呈现 CVE 详情。
- 此模板以用户友好的表格格式组织数据。

以下模板工作原理如下：
- 首先，它扩展了内置的 dcim/softwareversion_retrieve.html 模板，这是 SoftwareVersion 详情视图中每个标签页的基础模板。
- 在此模板中，我们只覆盖名为 content 的块，它定义标签页本身的 HTML 内容。
- 我们为 CVE 定义一个标题，然后是一个 CVE 条目表格，我们通过迭代自定义字段的 CVE 来填充它。

#### **创建 `templates/nautobot_software_cves/` 文件夹**

```
nautobot-app-software-cves/
├── nautobot_software_cves/
│   ├── templates/
│   │   ├── nautobot_software_cves/
│   │   │   ├── software_cves.html
```

#### 在 `software_cves.html` 中插入以下代码：

```html
{% extends 'dcim/softwareversion_retrieve.html' %}
{% block content %}
<div class="panel panel-default">
    <table class="table table-hover table-headings">
        <thead>
            <tr>
                <th>CVE Name</th>
                <th>CVSS Base Score</th>
                <th>Severity</th>
            </tr>
        </thead>
        {% for cve_name, cve_data in cves.items %}
        <tr>
            <td><a href="{{ cve_data.link }}">{{ cve_name }}</a></td>
            <td>{{ cve_data.cvss_base_score }}</td>
            <td>{{ cve_data.severity }}</td>
        </tr>
        {% endfor %}
    </table>
</div>
{% endblock content %}
```

### **验证**

- 再次重启 invoke debug 之后。
- 在 Cisco IOS-XE 版本 17.7.2 的 **SoftwareVersion** 详情视图中，一个新的 CVEs 标签页已被添加。点击此标签页将显示与此 **SoftwareVersion** 关联的 CVE 列表。
![cves_tab](images/cves_tab.png)

## **结论**

此增强通过以下方式改善了 Nautobot UI：
✅ 为每个 **SoftwareVersion** 添加了**专用 CVE 标签页**。
✅ 提供了 CVE 数据的**结构和详细视图**。
✅ 使用户更容易分析与其软件版本关联的安全漏洞。

在**第 84 天**，我们可以专注于**创建实际模板（`software_cves.html`）**以结构化表格格式显示 CVE。

如果需要任何修改，请告诉我！🚀

## 第 83 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。我们强烈建议你停止实例，**而不是**删除实例，直到我们在第 89 天完成整个顶点项目，因为这些天的内容是相互构建的。

继续在你选择的社交媒体上发布今天挑战中构建的 CVE 详情视图的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将继续进行顶点项目。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+83+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 83 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）