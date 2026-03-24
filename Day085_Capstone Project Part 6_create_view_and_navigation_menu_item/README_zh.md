# 顶点项目第六部分。CVE 管理 Nautobot 应用 - 第 85 天

## **目标**

在**第 85 天**，我们将把第 84 天的 **CVE 状态表**集成到 Nautobot 视图中。此视图将显示所有**软件版本**及其关联 CVE 的概览，使用户能够轻松识别易受攻击的版本。

为实现此目标，我们将：
1. **实现一个列表视图集**来管理视图逻辑。
2. **在 `urls.py` 中定义相应的 URL** 以使视图可访问。
3. **将视图添加到 Nautobot 导航栏**，确保用户能够轻松找到它。

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

### **1. 实现一个列表视图集**

Nautobot 提供了 **`ObjectListViewMixin`** 类，它简化了列表视图的实现。我们将创建一个名为 **`SoftwareCvesStatusViewSet`** 的新视图集，它使用第 84 天的 `CveStatusTable`。

在 **`views.py`** 中添加以下代码：

````python
from nautobot.apps import views
from nautobot.dcim.models import SoftwareVersion
from nautobot_software_cves.tables import CveStatusTable
from nautobot.dcim.filters import SoftwareVersionFilterSet

class SoftwareCvesStatusViewSet(views.ObjectListViewMixin):
    queryset = SoftwareVersion.objects.all()
    filterset_class = SoftwareVersionFilterSet
    table_class = CveStatusTable
````

#### **这如何工作**

- **queryset** 检索所有 `SoftwareVersion` 对象。
- **filterset_class** 允许过滤结果。
- **table_class** 指定使用 `CveStatusTable` 来呈现结果。

### **2. 在 `urls.py` 中添加相应的 URL**

为使此视图可访问，在 **`urls.py`** 中注册它。

````python
from django.urls import path
from django.views.generic import RedirectView
from django.templatetags.static import static
from nautobot.apps.urls import NautobotUIViewSetRouter
from nautobot_software_cves import views

router = NautobotUIViewSetRouter()
router.register("softwareversions", views.SoftwareCvesStatusViewSet)

urlpatterns = [
    path(
        "docs/",
        RedirectView.as_view(url=static("nautobot_software_cves/docs/index.html")),
        name="docs",
    ),
    path(
        "softwareversions/<uuid:pk>/cves/",
        views.SoftwareCvesView.as_view(),
        name="software_cves",
    ),
]

urlpatterns += router.urls
````

#### **这如何工作**

- 在 `softwareversions/` 端点下注册 **`SoftwareCvesStatusViewSet`**。
- 将 **router.urls** 添加到现有的 `urlpatterns`。

### **3. 将视图添加到 Nautobot 导航栏**

为使 **CVE 状态视图**易于访问，我们将把它添加到 **Devices → Software** 菜单部分。

在 **`nautobot_software_cves/`** 目录中，找到 **`navigation.py`** 文件或如果你在第 80 天第 9 步选择了 "None" 则创建它，并添加以下代码：

````python
from nautobot.apps.ui import NavMenuGroup, NavMenuItem, NavMenuTab

menu_items = (
    NavMenuTab(
        name="Devices",
        groups=(
            NavMenuGroup(
                name="Software",
                items=(
                    NavMenuItem(
                        # link="plugins:nautobot_software_cves:softwareversions", # 下面解释
                        link="plugins:nautobot_software_cves:softwareversion_list",
                        name="CVE Status",
                        permissions=["dcim.view_softwareversion"],
                    ),
                ),
            ),
        ),
    ),
)
````

#### **这如何工作**

- 在 **Devices → Software** 下添加一个**新菜单项**。
- 将菜单项链接到 `SoftwareCvesStatusViewSet` 视图。
- 确保只有具有 **`dcim.view_softwareversion`** 权限的用户才能访问菜单项。

#### **为什么在 `navigation.py` 中使用 `"softwareversion_list"` 而不是 `"softwareversions"`？**

当使用 `NautobotUIViewSetRouter` 注册 `SoftwareCvesStatusViewSet` 时：
```python
router = NautobotUIViewSetRouter()
router.register("softwareversions", views.SoftwareCvesStatusViewSet)
```
Nautobot 自动生成遵循 Django ViewSet 模式的**命名 URL**。

例如：
| **URL 模式** | **生成的名称** |
|----------------|-------------------|
| `/softwareversions/` | `plugins:nautobot_software_cves:softwareversion_list` |
| `/softwareversions/<uuid:pk>/` | `plugins:nautobot_software_cves:softwareversion` |
| `/softwareversions/add/` | `plugins:nautobot_software_cves:softwareversion_add` |

由于**列表视图**被命名为 **`softwareversion_list`**，你必须在 `navigation.py` 中引用此名称：

```python
NavMenuItem(
    link="plugins:nautobot_software_cves:softwareversion_list",
    name="CVE Status",
    permissions=["dcim.view_softwareversion"],
),
```

使用 `"softwareversions"` **将不起作用**，因为 Nautobot 期望的是 **ViewSet 生成的名称**。
要验证生成的名称，请运行：

```sh
nautobot-server show_urls | grep software-cves
```

### 验证

- 再次重启 invoke debug 之后。

- 要验证它是否正常工作，请导航到 Nautobot 实例的主菜单，点击 **Devices**，然后查找 **SOFTWARE** 子菜单。你应该会看到一个名为 **CVE Status** 的新项目。

![cves_navigation_menu](images/navigation_menu.png)

- 点击 **CVE Status** 菜单项，确认 **CVE Status 表**已显示。浏览器中的 URL 应该是 `/plugins/software-cves/softwareversions/`。

![cves_status_table](images/cve_status_table.png)

## **最终结果**

✅ **用户现在可以访问所有软件版本及其 CVE 状态的列表视图。**
✅ **视图已注册到 Nautobot 并可通过 URL 访问。**
✅ **CVE 状态视图现在是 Nautobot 导航菜单的一部分。**

通过此设置，Nautobot 用户可以轻松跟踪其网络中与不同软件版本关联的 CVE！

🚀 敬请期待**第 86 天**，在那里我们将创建一个用于软件 CVE 的 REST API 端点。

## 第 85 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。我们强烈建议你停止实例，**而不是**删除实例，直到我们在第 89 天完成整个顶点项目，因为这些天的内容是相互构建的。

继续在你选择的社交媒体上发布今天挑战中构建的新应用实例的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将继续进行顶点项目。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+85+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 85 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）