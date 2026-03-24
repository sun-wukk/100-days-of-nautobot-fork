# 顶点项目第七部分。CVE 管理 Nautobot 应用 - 第 86 天

## **目标**

在**第 86 天**，我们将通过 REST API 端点暴露每个单独**软件版本**的 CVE 信息。这确保了 UI 中可见的数据也可以通过编程方式访问，符合 API 优先的最佳实践。

为实现此目标，我们将：
1. **实现一个自定义 API 视图**来为特定的 `SoftwareVersion` 提供 CVE 数据。
2. **在 `urls.py` 中定义相应的 API URL**。
3. **验证 API** 可访问并返回预期的 CVE 数据。

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

### **1. 实现一个自定义 API 视图**

由于我们没有使用专用数据模型来存储 CVE，因此我们不会使用 `ModelSerializer`。相反，我们将直接从每个 `SoftwareVersion` 的 `custom_field_data` 中返回 `cves` json 数据。

如果你在第 80 天的第 9 步选择了 'None'，则在 **`nautobot_software_cves/`** 目录下创建一个名为 `api` 的文件夹，然后在里面创建文件 `views.py`、`urls.py` 和一个空的 `__init__.py`。

在 **`nautobot_software_cves/api/views.py`** 中添加以下代码：

````python
from django.shortcuts import get_object_or_404
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView

from nautobot.dcim.models import SoftwareVersion


class SoftwareVersionCVEsView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request, pk=None, format=None):
        software_version = get_object_or_404(
            SoftwareVersion.objects.restrict(self.request.user, "view"), pk=pk
        )
        custom_field_data = software_version.custom_field_data
        return Response({"cves": custom_field_data.get("cves", {})})
````

#### **这如何工作**

- 使用 Nautobot 的权限系统对请求进行身份验证。
- 通过其 UUID（`pk`）检索 `SoftwareVersion` 对象。
- 从 `custom_field_data` 中提取 `cves`。
- 将 `cves` 作为 JSON 响应返回。

### **2. 在 `nautobot_software_cves/api/urls.py` 中添加相应的 URL**

为使视图可通过 REST 端点访问，请在插件的 **API URL 配置**中注册它。

>[!提示]
> 删除或注释掉现有的 `urlpatterns` 如下所示。

在 **`nautobot_software_cves/api/urls.py`** 中添加：

````python
from django.urls import path
from nautobot_software_cves.api.views import SoftwareVersionCVEsView

urlpatterns = [
    path(
        "softwareversions/<uuid:pk>/cves/",
        SoftwareVersionCVEsView.as_view(),
        name="software_cves",
    ),
]

# urlpatterns = router.urls
````

#### **这如何工作**

- 在 `/api/plugins/software-cves/` 下定义一个新的 URL 模式。
- `pk` 指的是 `SoftwareVersion` 对象的 UUID。
- 该端点以 JSON 格式返回 `cves`。

### **3. 项目文件结构**

完成上述步骤后，你的插件目录结构应如下所示：

````text
nautobot_software_cves/
├── __init__.py
├── api
│   ├── __init__.py
│   ├── urls.py
│   └── views.py
├── navigation.py
├── tables.py
├── template_content.py
├── templates
│   └── nautobot_software_cves
│       └── software_cves.html
├── tests
│   ├── __init__.py
│   └── test_basic.py
├── urls.py
└── views.py
````

### **验证**

要测试 API 端点：

1. 在 Nautobot UI 中导航到**软件版本**对象的详情页面。
2. 转到**高级**标签页以复制**UUID**。
![software_version_uuid](images/software_version_uuid.png)
1. 再次重启 invoke debug 后，使用 Nautobot 内置的 OpenAPI（Swagger）UI。导航到 http://<your-server-ip>:8080/api/docs/ 并向下滚动到 plugins 标题，你应该会看到新的 REST API 端点已列出：
![software_cves_api_1](images/software_cves_api_1.png)
1. 点击此端点展开它，然后点击 Try it out 按钮。将先前获得的软件版本 PK 输入到 id 字段中并点击 Execute。你应该会看到包含为此软件版本定义的 CVE 列表的预期响应：
1. ![software_cves_api_2](images/software_cves_api_2.png)

## **最终结果**

✅ **用户现在可以通过 REST API 检索给定软件版本的 CVE 数据。**
✅ **API 端点遵守 Nautobot 的权限模型和 URL 约定。**
✅ **实现直接从软件版本的 `custom_field_data` 返回结构化 JSON。**

此添加为第三方集成、自动化或仪表板以编程方式消费 CVE 数据提供了基础。

🚀 敬请期待**第 87 天**，在那里我们将探索如何使用 NIST NVD 数据库获取 CVE。

## 第 86 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。我们强烈建议你停止实例，**而不是**删除实例，直到我们在第 89 天完成整个顶点项目，因为这些天的内容是相互构建的。

继续在你选择的社交媒体上发布今天挑战中构建的新应用实例的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将继续进行顶点项目。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+86+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 86 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）