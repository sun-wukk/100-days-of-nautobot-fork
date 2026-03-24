# 顶点项目第五部分。CVE 管理 Nautobot 应用 - 第 84 天

## **目标**

在**第 84 天**，我们将介绍一个 CVE 的**总体状态视图**，为用户提供一个概览，显示哪些**软件版本**有关联的 CVE 以及每个版本包含多少个 CVE。此增强将帮助用户快速识别易受攻击的软件版本。

为实现此目标，我们将：
1. **创建一个新的数据表**来显示软件版本的 CVE 状态。
2. **在 `tables.py` 中定义表逻辑**。
3. **确保此表与 Nautobot 的 UI 集成**，使其可通过专用视图访问（将在下一步实现）。

---

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

### **1. 使用 `tables.py` 文件**

在 **`nautobot_software_cves/`** 目录中，找到 **`tables.py`** 文件或如果你在第 80 天第 9 步选择了 "None" 则创建它。

```
nautobot-app-software-cves/
├── nautobot_software_cves/
│   ├── __init__.py
│   ├── tables.py  # <-- 找到这个文件
```

此文件将定义 **CVE 状态表**的结构和行为。

### **2. 如果你在第 80 天第 9 步选择了 "None"，则在 `tables.py` 中插入以下代码**

```python
import django_tables2 as tables # 预先创建
from nautobot.apps.tables import BaseTable, ButtonsColumn, ToggleColumn # 预先创建
from django.urls import reverse
from django.utils.safestring import mark_safe
from nautobot.dcim.models import SoftwareVersion


class CveStatusTable(BaseTable):
    class Meta(BaseTable.Meta):
        model = SoftwareVersion
        default_columns = ["platform", "version", "cves_count"]

    platform = tables.Column(linkify=True)
    version = tables.Column(linkify=True)
    cves_count = tables.Column(
        verbose_name="CVEs Count",
        empty_values=(),
        orderable=False
    )

    def render_cves_count(self, value, record):
        cves = record.custom_field_data.get('cves', {})
        if cves:
            url = reverse(
                "plugins:nautobot_software_cves:software_cves",
                kwargs={"pk": record.pk}
            )
            return mark_safe(f'<a href="{url}">{len(cves)}</a>')
        return 0

```

如果你在第 80 天第 9 步没有选择 "None"，以下是带有预先存在代码和新代码的 `tables.py` 的最终版本：

```python
"""Tables for nautobot_software_cves."""

import django_tables2 as tables
from nautobot.apps.tables import BaseTable, ButtonsColumn, ToggleColumn
from django.urls import reverse
from nautobot_software_cves import models
from django.utils.safestring import mark_safe
from nautobot.dcim.models import SoftwareVersion


class NautobotSoftwareCvesExampleModelTable(BaseTable):
    # pylint: disable=R0903
    """Table for list view."""

    pk = ToggleColumn()
    name = tables.Column(linkify=True)
    actions = ButtonsColumn(
        models.NautobotSoftwareCvesExampleModel,
        # 修改每行默认操作按钮的选项：
        # buttons=("changelog", "edit", "delete"),
        # 修改操作按钮的 pk 的选项：
        pk_field="pk",
    )

    class Meta(BaseTable.Meta):
        """Meta attributes."""

        model = models.NautobotSoftwareCvesExampleModel
        fields = (
            "pk",
            "name",
            "description",
        )

        # 修改默认在列表视图中显示的列的选项：
        # default_columns = (
        #     "pk",
        #     "name",
        #     "description",
        # )


class CveStatusTable(BaseTable):
    class Meta(BaseTable.Meta):
        model = SoftwareVersion
        default_columns = ["platform", "version", "cves_count"]

    platform = tables.Column(linkify=True)
    version = tables.Column(linkify=True)
    cves_count = tables.Column(
        verbose_name="CVEs Count",
        empty_values=(),
        orderable=False
    )

    def render_cves_count(self, value, record):
        cves = record.custom_field_data.get('cves', {})
        if cves:
            url = reverse(
                "plugins:nautobot_software_cves:software_cves",
                kwargs={"pk": record.pk}
            )
            return mark_safe(f'<a href="{url}">{len(cves)}</a>')
        return 0

```

---

## **`CveStatusTable` 的工作原理**

### **1. 为软件版本定义一个表**

**`CveStatusTable`** 类扩展了 `BaseTable`，定义了一个表来显示软件版本及其 CVE 计数。

```python
class CveStatusTable(BaseTable):
```
- 此表将用于**视图**中，显示所有软件版本及其 CVE 信息的概览。

### **2. 指定模型和默认列**

```python
class Meta(BaseTable.Meta):
    model = SoftwareVersion
    default_columns = ["platform", "version", "cves_count"]
```
- 表链接到 `SoftwareVersion` 模型。
- **默认显示的列**为：
  - `platform`：软件平台（例如，Cisco IOS、Juniper JunOS 等）。
  - `version`：软件版本。
  - `cves_count`：与此版本关联的 CVE 数量。

### **3. 为平台和版本添加超链接**

```python
platform = tables.Column(linkify=True)
version = tables.Column(linkify=True)
```
- 使**平台**和**版本**可点击，以便用户可以直接导航到它们的详情。

### **4. `cves_count` 的自定义渲染方法**

```python
def render_cves_count(self, value, record):
    cves = record.custom_field_data.get('cves', {})
    if cves:
        url = reverse(
            "plugins:nautobot_software_cves:software_cves",
            kwargs={"pk": record.pk}
        )
        return mark_safe(f'<a href="{url}">{len(cves)}</a>')
    return 0
```
- 从**自定义字段**中检索 **CVE 数据**。
- 如果 CVE 存在：
  - 显示 **CVE 数量**。
  - 生成到专用 **CVE 详情视图**的**超链接**。
- 如果没有 CVE，则显示 `0`。

## **下一步**

- 现在我们有了 **CveStatusTable**，我们将创建一个**视图**来显示此表，并使其可通过 Nautobot 导航栏访问。
- 这将允许用户在一个集中视图中查看所有**软件版本**及其**CVE 计数**。

🚀 敬请期待**第 85 天**，在那里我们将把这个表集成到 Nautobot 视图中！

## 第 84 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。我们强烈建议你停止实例，**而不是**删除实例，直到我们在第 89 天完成整个顶点项目，因为这些天的内容是相互构建的。

继续在你选择的社交媒体上发布今天挑战中构建的新应用实例的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将继续进行顶点项目。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+84+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 84 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）