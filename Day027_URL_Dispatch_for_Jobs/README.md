# Nautobot 中的 URL 分发与视图

每个 Web 应用程序都需要一种机制来将传入的 HTTP 请求路由到相应的代码段。在 Nautobot 中，这一功能通过 Django 强大的 URL 路由系统实现。当请求到达时，Django 会查阅其 URL 配置（通常定义在一个或多个 urls.py 文件中）来确定处理该请求的合适视图。URL 定义与业务逻辑之间的清晰分离是 URL 分发设计模式的基本原则，有助于构建高度可维护的代码库。

在 Nautobot 中，URL 分发不仅限于提供静态页面或处理 API 端点，它在平台的可扩展性方面也发挥着至关重要的作用，尤其是在其动态插件生态系统中。开发者可以定义直接映射到视图的自定义 URL 模式，从而创建能够与 Nautobot 的模型、Job 及其他核心组件无缝交互的新页面。

您可能还记得第 23 天，我们直接修改了 Nautobot 核心，添加了一个用于显示结果表格的附加选项卡。今天，我们将创建自己的 Nautobot 应用来覆盖用于显示 Job 输出的核心视图。虽然这种方式与之前的实现目标相似，但它引入了一种灵活的方法，可应用于各种功能增强场景。例如，您可以利用这一技术在特定设备的选项卡中集成 Grafana 图表等。

## 环境配置

环境配置与 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md) 相同，以下是步骤摘要，如需详细背景说明请参阅该指南。

> [!TIP]
> 如果您停止了 Codespace 环境后重新启动，发现 Docker 守护进程无法正常工作，请按照配置指南中的步骤重建环境。

> [!TIP]
> 请确保从 ```nautobot-docker-compose/environments/docker-compose.local.yml``` 文件中移除第 23 天创建的模板映射，确保使用原始的 Nautobot 文件后再继续操作。

同样，我们将在 `nautobot-docker-compose/environments/docker-compose.local.yml` 中添加一个卷挂载。我们将使用插件示例，它应该已经位于您的 `nautobot-docker-compose` 目录下的 `plugins` 文件夹中。
`````yaml
---
services:
  nautobot:
    command: "nautobot-server runserver 0.0.0.0:8080"
    ports:
      - "8080:8080"
    volumes:
      - "../config/nautobot_config.py:/opt/nautobot/nautobot_config.py"
      - "../jobs:/opt/nautobot/jobs"
      - "../plugins/plugin_example/nautobot_example_plugin:/usr/local/lib/python3.8/site-packages/nautobot_example_plugin"
    healthcheck:
      interval: "30s"
      timeout: "10s"
      start_period: "60s"
      retries: 3
      test: ["CMD", "true"]  # Due to layering, disable: true won't work. Instead, change the test
  celery_worker:
    volumes:
      - "../config/nautobot_config.py:/opt/nautobot/nautobot_config.py"
      - "../jobs:/opt/nautobot/jobs"
      - "../plugins/plugin_example/nautobot_example_plugin:/usr/local/lib/python3.8/site-packages/nautobot_example_plugin"
`````

### 启动 Nautobot
`````sh
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke debug
`````

## URL 与视图的作用

### `urls.py`

为了更好地理解 Nautobot 如何处理路由，从容器中打开 Nautobot Shell 并导入 Nautobot 核心 URL。这一过程会显示 Nautobot 某个 `urls.py` 文件的位置，从而可以查看其内容：
`````sh
root@eba3a1d8b6ab:/opt/nautobot# nautobot-server nbshell
>>> import nautobot.core.urls
>>> print(nautobot.core.urls.__file__)
/usr/local/lib/python3.8/site-packages/nautobot/core/urls.py
>>> exit()
`````

该文件对应 [Nautobot GitHub 仓库](https://github.com/nautobot/nautobot/blob/develop/nautobot/core/urls.py) 中的同名文件。

现在聚焦于 `urls.py` 文件的特定部分：
`````python
urlpatterns = [
    path("circuits/", include("nautobot.circuits.urls")),
    path("cloud/", include("nautobot.cloud.urls")),
    path("dcim/", include("nautobot.dcim.urls")),
    path("extras/", include("nautobot.extras.urls")),
    path("ipam/", include("nautobot.ipam.urls")),
    path("tenancy/", include("nautobot.tenancy.urls")),
]
`````

这些 URL 模式将请求引导至您在 Nautobot 应用中点击链接时屏幕上渲染的相应视图。每个路径对应一组模型、视图和 API，实际上各自构成一个独立的 Nautobot 应用。

### views.py

URL 模式决定请求的去向，而视图负责处理请求并生成相应的响应。在 Django（以及 Nautobot）中，视图封装了与模型交互、执行数据处理以及渲染模板或返回 API 响应的业务逻辑。
`````python
class JobResultView(generic.ObjectView):
    """
    Display a JobResult and its Job data.
    """

    queryset = JobResult.objects.prefetch_related("job_model", "user")
    template_name = "extras/jobresult.html"

    def get_extra_context(self, request, instance):
        associated_record = None
        job_class = None
        if instance.job_model is not None:
            job_class = instance.job_model.job_class

        return {
            "job": job_class,
            "associated_record": associated_record,
            "result": instance,
            **super().get_extra_context(request, instance),
        }
`````

在此示例中：

**GET 请求**：当用户访问某个 JobResult 的 URL 时，视图使用预配置的查询集（同时预取关联的 Job 模型和用户数据）从数据库中检索相应对象，然后渲染 extras/jobresult.html 模板以显示 JobResult 详情。在此过程中，`get_extra_context` 方法被调用以扩充模板上下文，该方法检查 JobResult 实例是否关联了 Job 模型，若有则提取对应的 Job 类，并将这些信息（连同尚未使用的关联记录和 JobResult 本身）添加到提供给模板的上下文中。

**POST 请求**：该视图仅用于展示信息，不包含任何处理 POST 请求的逻辑。与该视图的所有交互均为只读操作，专注于渲染详细的 Job 结果数据，而非处理用户输入或表单提交。

这种设计确保了当用户访问 JobResult 页面时，能够获得包含相关 Job 数据的完整视图，同时通过 URL 路由与视图逻辑的清晰分离来保证代码质量。

## 自定义插件的构建模块

今天，我们将创建一个映射到视图的 URL，该视图专门用于与 VerifyHostname Job 交互。您也可以创建用于执行定期网络配置检查或数据聚合的 Job，视图可以简单到只呈现一个带触发 Job 按钮的页面，也可以复杂到处理表单提交和 Job 状态更新。

> [!TIP]
> 以下所有文件均应在 `nautobot-docker-compose/plugins/plugin_example/nautobot_example_plugin` 目录下创建。该目录同时映射到 Nautobot 和 Nautobot-worker 容器，并放置于 Python 的 site-packages 目录中，其方式与 PIP 安装文件类似。这样的配置使我们能够像通过 PIP 安装了该插件一样启动 Nautobot 容器。

### 构建自定义 Nautobot 插件应用

在 `plugins/plugin_example/nautobot_example_plugin` 目录下，我们将创建一组文件作为插件的构建模块，包括插件配置、URL 映射、自定义视图、Job 定义以及用于渲染页面的模板。以下是这些文件及其作用的详细说明：

![vscode_explorer](images/vscode_explorer.png)

1. `__init__.py`

该文件通过定义插件配置来初始化并向 Nautobot 注册插件。

- **插件配置**：`ExampleConfig` 类（继承自 `PluginConfig`）定义了插件的名称、版本、作者、基础 URL 等核心元数据，告知 Nautobot 如何将插件集成到系统中。

- **注册**：通过 `config = ExampleConfig` 赋值，Nautobot 会在启动时自动检测并加载您的插件。
`````python
from nautobot.extras.plugins import PluginConfig

class ExampleConfig(PluginConfig):
    """Plugin configuration for the nautobot_example plugin."""

    name = "nautobot_example_plugin"
    verbose_name = "Simple Project for Example"
    version = "0.1.1"
    author = "Network to Code"
    description = ""
    base_url = "example"
    required_settings = []
    default_settings = {}
    caching_config = {}

config = ExampleConfig  # pylint: disable=invalid-name
`````

2. `urls.py`

该文件定义插件的 URL 映射。

- **URL 映射**：将特定 URL 模式（`verifyhostname-results/<uuid:pk>/`）映射到自定义视图（`CustomJobResultView`）。当用户访问该 URL 时，请求将被定向到负责显示 Job 结果的视图。
`````python
from django.urls import path
from nautobot_example_plugin.views import CustomJobResultView

urlpatterns = [
    path("verifyhostname-results/<uuid:pk>/", CustomJobResultView.as_view(), name="custom_job_result"),
]
`````

3. `views.py`

该文件通过扩展 Nautobot 内置的 Job 结果视图来自定义数据的呈现方式。

- **自定义 Job 结果视图**：`CustomJobResultView` 类继承自 `JobResultView`（已处理权限验证和基本对象渲染），将默认模板替换为自定义模板（`customized_jobresult.html`），并扩展上下文数据以包含额外变量（如 Job 结果和自定义消息）。

- **视图覆盖注册**：`override_views` 字典指示 Nautobot 使用此自定义视图替代默认视图，确保您的修改应用于 Job 结果展示页面。
`````python
from nautobot.extras.views import JobResultView  # 导入内置视图

class CustomJobResultView(JobResultView):
    """
    该视图自定义 Nautobot 内置的 JobResultView。
    由于 JobResultView 已实现 ObjectPermissionRequiredMixin，
    无需在此重复引入。
    """
    template_name = "nautobot_example_plugin/customized_jobresult.html"

    def get_context_data(self, **kwargs):
        # 调用父类实现以获取默认上下文
        context = super().get_context_data(**kwargs)
        # JobResult 对象在上下文中以 'object' 形式存在
        job_result = context.get("object")
        if job_result and job_result.result:
            context["results"] = job_result.result.get("results", [])
        else:
            context["results"] = []
        # 在此添加其他上下文变量
        context["custom_message"] = "This is my custom job result view."
        return context

# 该字典告知 Nautobot 对 Job 结果使用自定义视图
override_views = {
    "extras:jobresult": CustomJobResultView.as_view(),
}
`````

4. `jobs/__init__.py` 和 `jobs/verify_hostnames.py`

这两个文件定义插件将执行的自定义 Job。

- **包初始化**：`jobs/__init__.py` 将 jobs 目录标记为 Python 包，使 Nautobot 能够发现并导入 Job 模块。
`````python
from nautobot.core.celery import register_jobs
from .verify_hostnames import VerifyHostnameJob

jobs = [VerifyHostnameJob]
register_jobs(*jobs)
`````

- **自定义 Job 实现**：在 `jobs/verify_hostnames.py` 中，`VerifyHostnameJob` 类定义了一个检查设备主机名是否符合指定模式的 Job。它使用 `ObjectVar` 选择包含设备的位置，遍历该位置的每台设备，并记录每台设备主机名是否通过正则模式匹配检查。此外，Job 还使用 Django 的 `reverse` 函数构建指向详细结果视图的 URL，并返回包含 Job 结果和该 URL 的字典，供自定义视图渲染详细结果页面使用。
`````python
from nautobot.apps.jobs import Job, ObjectVar
from nautobot.dcim.models.locations import Location
from nautobot.dcim.models.devices import Device
import re
from django.urls import reverse

HOSTNAME_PATTERN = re.compile(r"[a-z0-1]+\-[a-z]+\-\d+\.infra\.valuemart\.com")

name = "Data Quality Custom Jobs Collection"

class VerifyHostnameJob(Job):
    location_to_check = ObjectVar(
        model=Location,
        query_params={"has_devices": True},
    )

    class Meta:
        name = "Verify Hostname Pattern For Selected Locations Plugin Job"
        description = "Checks all devices at the designated location for hostname pattern conformity"

    def run(self, location_to_check):
        results = []
        for device in Device.objects.filter(location=location_to_check):
            hostname = device.name
            compliance_status = "PASS" if HOSTNAME_PATTERN.match(hostname) else "FAIL"

            if compliance_status == "PASS":
                self.logger.info(f"{hostname} is compliant.", extra={"object": device})
            else:
                self.logger.error(f"{hostname} does NOT match the hostname pattern.", extra={"object": device})

            results.append({
                "hostname": hostname,
                "device_id": device.id,
                "status": compliance_status,
                "device_url": device.get_absolute_url(),
            })
        
        link_url = reverse("plugins:nautobot_example_plugin:custom_job_result", args=[str(self.job_result.id)])
        self.logger.info(f'<a href="{link_url}" target="_blank">View Detailed Results</a>')

        return {"results": results, "redirect_url": link_url}
`````

5. 模板文件

模板负责渲染显示插件输出的 HTML 页面。

- **`templates/nautobot_example_plugin/customized_jobresult.html`**：该模板继承自 Nautobot 的通用对象详情模板，为 Job 结果页面提供自定义布局，定义了多个块（面包屑导航、按钮、内容区域、选项卡和 JavaScript）来组织页面结构。模板使用 `CustomJobResultView` 提供的上下文（包括自定义消息和 Job 结果）来呈现 Job 执行的详细视图。

- **`templates/nautobot_example_plugin/inc/hostname_check_results.html`**：该局部模板被包含在主 Job 结果模板中，通过遍历 Job 的结果数据显示主机名检查结果表格。此外，它还包含一个按钮和配套的 JavaScript 代码，用于将结果导出为 CSV 文件，方便用户下载和分析 Job 输出。
`````templates/nautobot_example_plugin/customized_jobresult.html```:
````html
{% extends 'generic/object_detail.html' %}
{% load helpers %}
{% load custom_links %}
{% load form_helpers %}
{% load log_levels %}
{% load plugins %}
{% load static %}
{% load buttons %}

{% block breadcrumbs %}
    <li><a href="{% url 'extras:jobresult_list' %}">Job Results</a></li>
    {% if result.job_model is not None %}
        <li>{{ result.job_model.grouping }}</li>
        <li><a href="{% url 'extras:jobresult_list' %}?job_model={{ result.job_model.name }}">
            {{ result.job_model }}
        </a></li>
    {% elif associated_record %}
        {% if associated_record.name %}
            <li><a href="{% url 'extras:jobresult_list' %}?name={{ associated_record.name|urlencode }}">
                {{ associated_record.name }}
            </a></li>
        {% else %}
            <li>{{ associated_record }}</li>
        {% endif %}
    {% elif job %}
        <li><a href="{% url 'extras:jobresult_list' %}?name={{ job.class_path|urlencode }}">
            {{ job.class_path }}
        </a></li>
    {% else %}
        <li>{{ result.name }}</li>
    {% endif %}
    <li>{{ result.created }}</li>
{% endblock breadcrumbs %}

{% block buttons %}
    {% if perms.extras.run_job %}
        {% if result.job_model and result.task_kwargs %}
            <a href="{% url 'extras:job_run' pk=result.job_model.pk %}?kwargs_from_job_result={{ result.pk }}"
               class="btn btn-success">
                <span class="mdi mdi-repeat" aria-hidden="true"></span> Re-Run
            </a>
        {% elif result.job_model is not None %}
            <a href="{% url 'extras:job_run' pk=result.job_model.pk %}"
               class="btn btn-primary">
                <span class="mdi mdi-play" aria-hidden="true"></span> Run
            </a>
        {% endif %}
    {% endif %}
    <a href="{% url 'extras-api:joblogentry-list' %}?job_result={{ result.pk }}&format=csv" class="btn btn-success">
        <span class="mdi mdi-database-export" aria-hidden="true"></span> Export
    </a>
    {{ block.super }}
{% endblock buttons %}

{% block title %}
    Job Result:
    {% if result.job_model is not None %}
        {{ result.job_model }}
    {% elif associated_record %}
        {{ associated_record }}
    {% elif job %}
        {{ job }}
    {% else %}
        {{ result.name }}
    {% endif %}
{% endblock %}

{% block extra_nav_tabs %}
    {% if result.data.output %}
        <li role="presentation">
            <a href="#output" role="tab" data-toggle="tab">Output</a>
        </li>
    {% endif %}
    {% if result.result.results %}
        <li role="presentation">
            <a href="#hostname-check" role="tab" data-toggle="tab">Hostname Check</a>
        </li>
    {% endif %}
{% endblock %}

{% block content_full_width_page %}
    {% include 'extras/inc/jobresult.html' with result=result log_table=log_table %}
{% endblock content_full_width_page %}

{% block advanced_content_left_page %}
    <div class="panel panel-default">
        <div class="panel-heading">
            <strong>Job Keyword Arguments</strong>
        </div>
        <div class="panel-body">
            {% include 'extras/inc/json_data.html' with data=result.task_kwargs format="json" %}
        </div>
    </div>
    <div class="panel panel-default">
        <div class="panel-heading">
            <strong>Job Positional Arguments</strong>
        </div>
        <div class="panel-body">
            {% include 'extras/inc/json_data.html' with data=result.task_args format="json" %}
        </div>
    </div>
    <div class="panel panel-default">
        <div class="panel-heading">
            <strong>Job Celery Keyword Arguments</strong>
        </div>
        <div class="panel-body">
            {% include 'extras/inc/json_data.html' with data=result.celery_kwargs format="json" %}
        </div>
    </div>
{% endblock advanced_content_left_page %}
{% block advanced_content_right_page %}
    <div class="panel panel-default">
        <div class="panel-heading">
            <strong>Worker</strong>
        </div>
        <table class="table table-hover panel-body attr-table">
            <tbody>
                <tr>
                    <td>Worker Hostname</td>
                    <td>{{ result.worker }}</td>
                </tr>
                <tr>
                    <td>Queue</td>
                    <td>{{ result.celery_kwargs.queue}}</td>
                </tr>
                <tr>
                    <td>Task Name</td>
                    <td>{{ result.task_name }}</td>
                </tr>
                <tr>
                    <td>Meta</td>
                    <td>{% include 'extras/inc/json_data.html' with data=result.meta format="json" %}</td>
                </tr>
            </tbody>
        </table>
    </div>
    <div class="panel panel-default">
        <div class="panel-heading">
            <strong>Traceback</strong>
        </div>
        <div class="panel-body">
            {% include 'extras/inc/json_data.html' with data=result.traceback format="python" %}
        </div>
    </div>
{% endblock advanced_content_right_page %}
{% block extra_tab_content %}
    {% if result.data.output %}
        <div role="tabpanel" class="tab-pane" id="output">
            <pre>{{ result.data.output }}</pre>
        </div>
    {% endif %}
    {% if result.result.results %}
        <div role="tabpanel" class="tab-pane" id="hostname-check">
            {% include 'nautobot_example_plugin/inc/hostname_check_results.html' %}
        </div>
    {% endif %}
{% endblock extra_tab_content %}


{% block javascript %}
    {{ block.super }}
    {% include 'extras/inc/jobresult_js.html' with result=result %}
    <script src="{% versioned_static 'js/tableconfig.js' %}"></script>
    <script src="{% versioned_static 'js/log_level_filtering.js' %}"></script>
{% endblock %}
````
````templates/nautobot_example_plugin/inc/hostname_check_results.html```:
```html
{% load custom_links %}

{% if result.result.results %}
    <h1>Hostname Check Results Table</h1>
    <!-- 添加带 id "hostname-check" 的容器 -->
    <div id="hostname-check">
        <table class="table table-hover">
            <thead>
                <tr>
                    <th>Hostname</th>
                    <th>Status</th>
                </tr>
            </thead>
            <tbody>
                {% for item in result.result.results %}
                <tr>
                    <td>
                        <a href="/dcim/devices/{{ item.device_id }}/?tab=main">
                            {{ item.hostname }}
                        </a>
                    </td>
                    <td>
                        {% if item.status == "PASS" %}
                            <span class="label label-success">{{ item.status }}</span>
                        {% else %}
                            <span class="label label-danger">{{ item.status }}</span>
                        {% endif %}
                    </td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
    </div>
    <button id="export-results" class="btn btn-primary">Export Results</button>
{% endif %}

<script>
    document.addEventListener('DOMContentLoaded', function() {
        document.getElementById('export-results').addEventListener('click', function() {
            // 由于表格现在包含在 #hostname-check 中，此选择器可正常工作
            var table = document.querySelector('#hostname-check table');
            if (!table) {
                console.error("Table not found!");
                return;
            }
            var rows = table.querySelectorAll('tr');
            
            var csv = [];
            for (var i = 0; i < rows.length; i++) {
                var row = [], cols = rows[i].querySelectorAll('td, th');
                
                for (var j = 0; j < cols.length; j++) {
                    // 清理单元格内容以适配 CSV 格式
                    var data = cols[j].innerText.replace(/(\r\n|\n|\r)/gm, '').replace(/(\s\s)/gm, ' ');
                    // 转义引号
                    data = data.replace(/"/g, '""');
                    // 必要时添加引号
                    if (data.search(/("|,|\n)/g) >= 0) data = '"' + data + '"';
                    row.push(data);
                }
                csv.push(row.join(','));
            }
            
            // 下载文件
            var csvString = csv.join('\n');
            var a = document.createElement('a');
            // 使用正确的 CSV MIME 类型
            a.href = 'data:text/csv;charset=utf-8,' + encodeURIComponent(csvString);
            a.target = '_blank';
            a.download = 'hostname_check_results.csv';
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
        });
    });
</script>
```

所有文件就绪后，需要重启容器以便 Nautobot 注册新插件。

### 停止容器
```bash
nautobot-1       |   "GET /extras/jobs/ HTTP/1.1" 200 255502
nautobot-1       | 02:56:33.623 INFO    django.server :
nautobot-1       |   "GET /template.css HTTP/1.1" 200 3984
Gracefully stopping... (press Ctrl+C again to force)
 Container nautobot_docker_compose-celery_beat-1  Stopping
 Container nautobot_docker_compose-celery_worker-1  Stopping
 Container nautobot_docker_compose-celery_worker-1  Stopped
 Container nautobot_docker_compose-celery_beat-1  Stopped
 Container nautobot_docker_compose-nautobot-1  Stopping
 Container nautobot_docker_compose-nautobot-1  Stopped
 Container nautobot_docker_compose-db-1  Stopping
 Container nautobot_docker_compose-redis-1  Stopping
 Container nautobot_docker_compose-db-1  Stopped
 Container nautobot_docker_compose-redis-1  Stopped
canceled
```

### 更新 `config/nautobot_config.py`

添加插件时，插件名称必须写入 `nautobot_config.py` 文件，以便向 Nautobot 注册插件并确保其正常运行。

![nautobot_config](images/nautobot_config.png)
```python
"""Nautobot development configuration file."""

# pylint: disable=invalid-envvar-default
import os
import sys

from nautobot.core.settings import *  # noqa: F403  # pylint: disable=wildcard-import,unused-wildcard-import
from nautobot.core.settings_funcs import is_truthy, parse_redis_connection

# Debug
DEBUG = is_truthy(os.getenv("NAUTOBOT_DEBUG", False))

TESTING = len(sys.argv) > 1 and sys.argv[1] == "test"

# Logging
LOG_LEVEL = "DEBUG" if DEBUG else "INFO"

# Redis
# Redis Cacheops
CACHEOPS_REDIS = parse_redis_connection(redis_database=1)

# 启用已安装的插件，将每个插件名称添加到列表中
PLUGINS = ["nautobot_example_plugin"]

# 插件配置设置
PLUGINS_CONFIG = {
    "nautobot_example_plugin": {},
}

CSRF_TRUSTED_ORIGINS = ["http://localhost:8080", "https://localhost:8080"]
```

### 重建容器
```bash
 ➜ ~/nautobot-docker-compose (main) $ invoke build
Building Nautobot 2.3.2 with Python 3.8...
Running docker compose command "build"
#0 building with "default" instance using docker driver
#1 [nautobot internal] load build definition from Dockerfile
#1 transferring dockerfile: 2.32kB done
#1 WARN: FromAsCasing: 'as' and 'FROM' keywords' casing do not match (line 3)
#1 WARN: FromAsCasing: 'as' and 'FROM' keywords' casing do not match (line 14)
#1 WARN: FromAsCasing: 'as' and 'FROM' keywords' casing do not match (line 54)
#1 DONE 0.1s
#2 [nautobot internal] load metadata for ghcr.io/nautobot/nautobot:2.3.2-py3.8
```

### 启动容器
```bash
$ invoke debug
Starting Nautobot in debug mode...
Running docker compose command "up"
 Container nautobot_docker_compose-redis-1  Created
 Container nautobot_docker_compose-db-1  Created
 Container nautobot_docker_compose-nautobot-1  Created
 Container nautobot_docker_compose-celery_beat-1  Created
 Container nautobot_docker_compose-celery_worker-1  Created
```

### 验证插件安装

在 VSCode 中打开第二个终端窗口，验证文件已成功复制到容器中：
```bash
$ cd nautobot-docker-compose/
$ invoke cli
Running docker compose command "ps --services --filter status=running"
Running docker compose command "exec nautobot bash"
nautobot@51c9bdd1c291:~$ ls /usr/local/lib/python3.8/site-packages/nautobot_example_plugin/
__init__.py  jobs  templates  urls.py  views.py
```

### 在 Nautobot 中验证

在左侧导航栏中找到 **Apps** 部分，展开后点击 **Installed Apps**。

![installed-apps](images/installed-apps.jpg)

点击 **Sample Project for Example**，可以看到该插件配置的功能详情，包括：
- 指向 `jobs` 目录中 Job 的链接。
- 在 `urls.py` 中创建的自定义 URL。
- 在 `views.py` 中配置的**核心视图覆盖（Core View Overrides）**，将核心 Job 结果页面替换为我们的自定义 HTML 模板。

![installed-app-details](images/installed-app-details.jpg)

### 运行 Job

点击 **Verify Hostname Pattern For Selected Locations Plugin Job** 链接，导航到该插件新增的 Job 页面，选择站点并运行。

![new-job-run](images/new-job-run.jpg)

### 查看 Job 结果

在 **Job Result** 页面，您将看到：
- 一个 **Hostname Check** 选项卡。
- 日志输出中的 **View Detailed Results** 链接。

点击该链接将跳转到同一 Job 结果页面，但使用的是自定义 URL。**Hostname Check** 选项卡将显示第 23 天创建的表格输出。

![hostname-check-tab](images/hostname-check-tab.jpg)

### Jobs 列表更新

在 **Jobs** 列表中，您将注意到：
- 新增了一个名为"Data Quality Custom Jobs Collection"的 **Job Group**。
- 已注册的 Job **Verify Hostname Pattern For Selected Locations Plugin Job**。

这些信息定义在 `VerifyHostnameJob` 类中：
```python
name = "Data Quality Custom Jobs Collection"
```

以及 `Meta` 类中：
```python
class Meta:
    name = "Verify Hostname Pattern For Selected Locations Plugin Job"
    description = "Checks all devices at the designated location for hostname pattern conformity"
```

![job-list](images/job-list.jpg)

## 总结

该自定义插件由以下组件构成：

- **配置（`__init__.py`）**：向 Nautobot 注册插件并设置核心元数据。
- **URL 映射（`urls.py`）**：定义从自定义 URL 到插件视图的路由。
- **自定义视图（`views.py`）**：扩展 Nautobot 内置视图以自定义 Job 结果展示。
- **Job 定义（`jobs/verify_hostnames.py`）**：实现主机名验证逻辑并记录结果。
- **模板（`customized_jobresult.html` 和 `hostname_check_results.html`）**：渲染 Job 结果展示的 UI，包括结果表格和 CSV 导出功能。

这种模块化结构确保了与 Nautobot 的无缝集成，同时保持了代码的清晰性和可扩展性。

## 第 27 天待办事项

记得在 [GitHub Codespaces](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布新 Job 成功执行的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将探讨 Nautobot Job 日志保留机制。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+27+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 27 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
````
