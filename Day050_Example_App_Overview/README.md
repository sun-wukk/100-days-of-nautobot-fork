# 示例 App 概览

从现在起的第 50 至 59 天，我们将借助一个现成的示例 App，把前面所学的知识运用到 Nautobot 的实际开发环境中。

具体来说，我们将克隆 [Nautobot](https://github.com/nautobot/nautobot) 仓库，并使用其中的 [example_app](https://github.com/nautobot/nautobot/tree/develop/examples/example_app)。

与[第 42 天](../Day042_Baking_an_App_Cookie/README.md)类似，Nautobot 仓库自带一套包含多个 Docker 容器的开发环境，我们将直接使用它。

今天的挑战，我们将：

- 搭建开发环境，使用不同的开发容器镜像启动 App。
- 快速梳理 App 如何与 Nautobot 集成，了解完整的端到端工作流。

准备好了吗？我们开始吧。

## 环境搭建

第 50 至 59 天将使用[场景二（Scenario 2）](../Lab_Setup/scenario_2_setup/README.md)。该场景包含一个已预先克隆好的 Nautobot 目录：

![scenario_2](images/scenario_2.png)

> [!INFORMATION]
> 如果你好奇为何需要使用预克隆的目录，[场景二搭建说明](../Lab_Setup/scenario_2_setup/README.md)中列出了我们对 Codespace 环境所做的调整。

Codespace 启动后，可以看到 `nautobot` 目录：

![nautobot_repository](images/nautobot_repository.png)

> [!WARNING]
> **已知构建问题（[Issue #64](https://github.com/nautobot/100-days-of-nautobot/issues/64)）**：运行 `invoke build` 之前，请先对 Dockerfile 进行如下修复：
> ```bash
> cd ~/nautobot
> # 修复一：替换已弃用的 mime-support 包
> sed -i 's/mime-support/media-types mailcap/' docker/Dockerfile
> # 修复二：从 no-binary 列表中移除 xmlsec（兼容 lxml 5.x）
> sed -i 's/lxml,pyuwsgi,xmlsec/lxml,pyuwsgi/' docker/Dockerfile
> ```

启动开发容器的步骤与场景一基本一致，但由于"Nautobot 本体"的复杂度更高、依赖包更多，整个过程会耗时更长。

具体步骤如下：

```
$ cd /home/vscode/nautobot
# ⚠️ 运行 invoke build 前，请先按上方警告对 Dockerfile 进行修复
$ poetry shell
$ poetry install
...
（大量依赖包安装过程）
...
$ invoke build
...
（此步骤较慢，请耐心等待，可以去泡杯咖啡或茶）
...
$ invoke debug
...
（此步骤同样需要等待，再来一杯也无妨）
...
```

一切完成后，各容器会开放多个端口，选择我们熟悉的 8080 端口即可：

![port_8080](images/port_8080.png)

使用 `admin/admin` 作为用户名和密码登录。

登录后可以看到 [Django Debug Toolbar](https://django-debug-toolbar.readthedocs.io/en/latest/) 处于激活状态——这是一个非常强大的调试工具，但为了获得更宽阔的页面空间，我们先点击顶部的 `Hide>>` 按钮将其收起。

![django_debug_toolbar](images/django_debug_toolbar.png)

接下来，我们仔细看看这套环境。

## 示例 App 环境

以下是几个值得关注的新特性：

- 环境中新增了一个运行 [Selenium](https://www.selenium.dev/) 的 Docker 镜像，用于无界面（headless）的 Web UI 自动化测试。
- 当前 Nautobot 版本为 2.4。对我们的目的而言差异不大，但 2.3 与 2.4 之间确实存在一些变化。
- 如果想了解各服务对应的端口：

```
@ericchou1 ➜ ~ $ cd nautobot
@ericchou1 ➜ ~/nautobot (develop) $ poetry shell
@ericchou1 ➜ ~ $ docker ps
CONTAINER ID   IMAGE                                 COMMAND                  CREATED         STATUS                   PORTS                                                                                                          NAMES
495fd7e794fa   local/nautobot-dev:local-2.4-py3.12   "watchmedo auto-rest…"   9 minutes ago   Up 7 minutes (healthy)   8080/tcp                                                                                                       nautobot-2-4-celery_beat-1
ed2026b1fa1d   local/nautobot-dev:local-2.4-py3.12   "watchmedo auto-rest…"   9 minutes ago   Up 7 minutes (healthy)   0.0.0.0:6898->6898/tcp, :::6898->6898/tcp, 0.0.0.0:8081->8080/tcp, :::8081->8080/tcp                           nautobot-2-4-celery_worker-1
b3f7ed22a9e5   local/nautobot-dev:local-2.4-py3.12   "/docker-entrypoint.…"   9 minutes ago   Up 9 minutes (healthy)   0.0.0.0:6899->6899/tcp, :::6899->6899/tcp, 0.0.0.0:8080->8080/tcp, :::8080->8080/tcp                           nautobot-2-4-nautobot-1
dd26d3cf119d   redis:6-alpine                        "docker-entrypoint.s…"   9 minutes ago   Up 9 minutes (healthy)   6379/tcp                                                                                                       nautobot-2-4-redis-1
70e700f76299   postgres:13                           "docker-entrypoint.s…"   9 minutes ago   Up 9 minutes (healthy)   5432/tcp                                                                                                       nautobot-2-4-db-1
f6b2d7fe2883   selenium/standalone-firefox:4.27      "/opt/bin/entry_poin…"   9 minutes ago   Up 9 minutes             5900/tcp, 0.0.0.0:4444->4444/tcp, :::4444->4444/tcp, 0.0.0.0:15900->15900/tcp, :::15900->15900/tcp, 9000/tcp   nautobot-2-4-selenium-1
```

### 示例 App

在 `APPS -> Installed Apps` 页面，可以看到 `Example Nautobot App` 已预装其中：

![example_app_detail](images/example_app_detail.png)

进入 `nautobot` 容器，看看以下两个关键路径：

- 源代码映射在容器的 `/source` 目录下。
- Nautobot 安装于 `/opt/nautobot`。

```
@ericchou1 ➜ ~ $ docker exec -it -u root nautobot-2-4-nautobot-1 bash

root@b3f7ed22a9e5:/source# pwd
/source

root@b3f7ed22a9e5:/source# ls
CHANGELOG.md        LICENSE.txt  SECURITY.md  docker    git                 jobs        nautobot                 pyproject.toml  tasks.py
CODE_OF_CONDUCT.md  NOTICE       changes      docs      install.sh          media       nautobot.code-workspace  renovate.json   venv
CONTRIBUTING.md     README.md    development  examples  invoke.yml.example  mkdocs.yml  poetry.lock              scripts

root@b3f7ed22a9e5:/source# ls /opt/nautobot/
__pycache__  git  jobs  media  nautobot_config.py  static
```

### URL 路由分发

我们可以从 Python 包入手，顺着核心 `urls.py` 一路追踪，看到它如何将 URL 规则委托给 `example_app/urls.py` 处理：

```
root@b3f7ed22a9e5:/source# cat /usr/local/lib/python3.12/site-packages/nautobot.pth 
/source

root@b3f7ed22a9e5:/source# cat nautobot/core/urls.py 
...
from nautobot.extras.plugins.urls import (
    apps_patterns,
    plugin_admin_patterns,
    plugin_patterns,
)
...
urlpatterns = [
    ...
    path("plugins/", include((plugin_patterns, "plugins"))),
    ...
]


root@b3f7ed22a9e5:/source# cat examples/example_app/example_app/urls.py 
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
    # 对使用 NautobotUIViewSet 的模型，仍可额外添加路由
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
    # 此 URL 用于测试 override_views 功能，具体实现位于
    # examples.example_app_with_view_override.example_app_with_view_override.views
    path("override-target/", views.ViewToBeOverridden.as_view(), name="view_to_be_overridden"),
    # 此 URL 用于测试 NautobotUIViewSetMixin 中的 permission_classes 功能
    path(
        "view-with-custom-permissions/",
        views.ViewWithCustomPermissions.as_view({"get": "list"}),
        name="view_with_custom_permissions",
    ),
]
urlpatterns += router.urls
```

从 URL 规则中可以看到，`config/` 路径对应 `views.ExampleAppConfigView` 视图。从根路径算起，完整地址应为 `plugins/example-app/config/`：

![example_app_url_1](images/example_app_url_1.png)

来看看这个视图的代码：

```
root@b3f7ed22a9e5:/source# cat examples/example_app/example_app/views.py
...
class ExampleAppConfigView(views.GenericView):
    def get(self, request):
        """渲染此 App 的配置页面。
        
        仅作示例——实际使用时，你需要根据 App 的具体情况传入真实的配置数据。
        """
        form = forms.ExampleAppConfigForm({"magic_word": "frobozz", "maximum_velocity": 300000})
        return render(request, "example_app/config.html", {"form": form})

    def post(self, request):
        """处理此 App 的配置变更请求。
        
        此处未作实际实现。
        """
        form = forms.ExampleAppConfigForm({"magic_word": "frobozz", "maximum_velocity": 300000})
        return render(request, "example_app/config.html", {"form": form})
...
```

这段视图代码与我们之前学的写法有些不同——之前用的是 `def` 定义的函数视图，这里用的是 `class`（类视图）。这是为什么？简单来说，[函数视图](https://docs.djangoproject.com/en/5.1/topics/http/views/)和[类视图](https://docs.djangoproject.com/en/5.1/topics/class-based-views/)都是合法的视图实现方式，类视图是更新的写法，用更少的代码做更多的事，代价是引入了更多"Django 魔法"。

即使看不懂每一行代码，我们也能从视图中找到关键信息——它返回的模板是 `example_app/config.html`，让我们看看模板内容：

```
root@c8032ee34216:/source# cat examples/example_app/example_app/templates/example_app/config.html 
{% extends 'base.html' %}
{% load form_helpers %}

{% block content %}
    <form action="" method="post" enctype="multipart/form-data" class="form form-horizontal">
        {% csrf_token %}
        <div class="row">
            <div class="col-md-6 col-md-offset-3">
                <h3>{% block title %}Example App Configuration{% endblock title %}</h3>
                <p>
                    A Nautobot App can implement this page if it has various configuration options that make sense to
                    view and/or update via the web UI (as opposed to configuring them under <code>PLUGINS_CONFIG</code>
                    in <code>nautobot_config.py</code>). The below is just a simple example.
                </p>
                {% block form %}
                    <div class="panel panel-default">
                        <div class="panel-heading"><strong>Configuration Options</strong></div>
                        <div class="panel-body">
                            {% block form_fields %}
                                {% render_form form %}
                            {% endblock %}
                        </div>
                    </div>
                {% endblock form %}
            </div>
        </div>
        <div class="row">
            <div class="col-md-6 col-md-offset-3 text-right">
                <button type="submit" class="btn btn-primary">Update Configuration</button>
            </div>
        </div>
    </form>
{% endblock content %}
```

我们刚刚完整地走了一遍从初始 URL 到最终 HTML 模板的完整链路：**URL 路由 → 视图 → HTML 模板**。不过，数据库模型在哪里呢？回头看视图代码，有一行 `form = forms.ExampleAppConfigForm`，来瞧瞧：

```
root@c8032ee34216:/source# cat examples/example_app/example_app/forms.py 
...
class ExampleAppConfigForm(BootstrapMixin, forms.Form):
    """App 专属配置表单的示例。"""

    magic_word = forms.CharField()
    maximum_velocity = forms.IntegerField(help_text="Meters per second")
...
```

可以看到，这部分代码与数据库模型并无关联。但从 `forms.py` 的其他代码片段中可以发现，如果需要通过表单提交来修改数据库，相关逻辑就会写在这里。

今天的挑战证明了一点：即使不理解每一行代码，凭借对 Django 基本设计模式的认知，我们依然能对代码结构了然于胸。

最后一步，动手修改 HTML，看看效果。

## 修改顶部横幅

展开首页的 Django Debug Toolbar：

![debug_toolbar_1](images/debug_toolbar_1.png)

工具栏提供了 CPU 耗时、当前请求、SQL 查询数量等信息。在 `Templates` 一栏可以看到模板路径。找到生成横幅的模板：

![debug_toolbar_2](images/debug_toolbar_2.png)

找到后，加入一行 `<h1>` 标签，写上 `Hello Banners!`：

![banner_1](images/banner_1.png)

成了！用户登录后就能看到我们精心准备的欢迎语：

![banner_2](images/banner_2.png)

今天的挑战是理解 Nautobot 架构、学会利用现有框架开发新 App 的重要一步，收获满满！

## 第 50 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 Codespace 实例。如前所述，接下来几天我们还会继续使用同一个实例。

请在你选择的社交媒体上发布新横幅或今天任意步骤的截图，务必使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，这样我们可以分享你的进展！

明天的挑战，我们将在示例 App 中操作数据库模型。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+50+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 50 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
