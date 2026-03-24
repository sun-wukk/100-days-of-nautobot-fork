# Jobs HTML 模板视图

在过去 22 天积累的基础上，让我们深入了解构成 Nautobot 的 HTML 模板，并感受 Nautobot 的高度可扩展性——它真正做到了"只要你能想到，就能构建出来"。今天我们将创建一个自定义 HTML 模板，将之前某个验证 Job 的数据以美观的表格形式呈现，并支持导出为 CSV 文件。

今天将涵盖以下内容：
- 如何通过新增"主机名检查"选项卡来扩展 Nautobot 内置的 jobresult.html 模板，展示更新后的 Job（VerifyHostname）返回的自定义数据。
- 修改 Job 以返回 JSON 数据结构。
- 通过 Docker 卷挂载新的本地模板文件。
- 添加 HTML/JavaScript 来展示主机名验证结果，并支持可选的导出功能。

> [!TIP]
> 直接修改 Nautobot 文件并不是最佳实践，因为每次升级都会覆盖这些改动，长期维护也不理想。Nautobot 应用（App）更适合用于此类定制化需求，后续将会介绍。我们希望这种方式有助于理解 Nautobot 框架的结构。

## 实验环境配置

环境配置与 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md) 相同，以下是步骤摘要，如需详细背景说明请参阅该指南。

以下是启动 Nautobot 的步骤回顾。如果是重启之前已停止的 Codespace 实例，可以跳过 `invoke build` 和 `invoke db-import`：
```
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

现在可以开始对 jobresult.html 进行定制了。

## 页面模板

什么是页面模板？让我们直接看 [Nautobot 文档](https://docs.nautobot.com/projects/core/en/stable/development/core/templates/) 的说明：*"Nautobot 提供了多种页面模板，在保持页面风格与应用整体一致的同时，提供了极大的灵活性。"*

这使我们能够构建简单的 Jinja2 模板，复用 Nautobot 中已有的 HTML 结构，确保所有新增内容保持统一的视觉风格。

让我们查看容器中的 `jobresult.html` 文件，可以直接通过命令行访问 Docker 中的这些文件：
```bash
$ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash
$ ls /usr/local/lib/python3.8/site-packages/nautobot/extras/templates/extras/job*
/usr/local/lib/python3.8/site-packages/nautobot/extras/templates/extras/job.html
/usr/local/lib/python3.8/site-packages/nautobot/extras/templates/extras/job_approval_confirmation.html
/usr/local/lib/python3.8/site-packages/nautobot/extras/templates/extras/job_approval_request.html
/usr/local/lib/python3.8/site-packages/nautobot/extras/templates/extras/job_bulk_edit.html
/usr/local/lib/python3.8/site-packages/nautobot/extras/templates/extras/job_detail.html
/usr/local/lib/python3.8/site-packages/nautobot/extras/templates/extras/job_edit.html
/usr/local/lib/python3.8/site-packages/nautobot/extras/templates/extras/job_list.html
/usr/local/lib/python3.8/site-packages/nautobot/extras/templates/extras/jobbutton_retrieve.html
/usr/local/lib/python3.8/site-packages/nautobot/extras/templates/extras/jobhook.html
/usr/local/lib/python3.8/site-packages/nautobot/extras/templates/extras/jobresult.html
```
![docker_container](images/docker-container-view.jpg)

也可以使用 VSCode 打开该文件。在侧边栏中找到 Docker 鲸鱼图标并点击，您将看到 nautobot-docker-compose 中所有正在运行的容器。找到名为 ```nautobot-docker-compose-nautobot-1``` 的容器，按照以下路径逐级展开文件夹：```/usr/local/lib/python3.8/site-packages/nautobot/extras/templates/extras/jobresult.html```。

![docker_container_folder_view](images/jobresult-file-in-container.jpg)

右键点击 `jobresult.html`，选择打开。现在可以在 VSCode 中查看该文件，这将使后续操作更加便捷。

> [!TIP]
> 也可以在[这里](https://github.com/nautobot/nautobot/blob/develop/nautobot/extras/templates/extras/jobresult.html)直接查看 `jobresult.html` 的代码。

## 更新主机名验证 Job

在第 7 天和第 8 天，我们创建了多个 Job 用于验证序列号、IP 地址、平台配置以及主机名格式。这些 Job 的文件名为 ```data_quality_jobs.py```。打开该文件，我们需要做一个小调整，以便为表格提供数据。

文件底部应有如下代码：
```python
class VerifyHostname(Job):
    location_to_check = ObjectVar(
        model=Location,
        query_params={
            "has_devices": True,
        }
    )
    
    class Meta:

        name = "Verify Hostname Pattern For Selected Locations"
        description = "Checks all devices at Designated Location for hostname pattern conformation"

    def run(self, location_to_check):
        """Run method for executing the checks on the devices."""

        # Iterate through each Device object, limited to just the location of choice.
        for device in Device.objects.filter(location=location_to_check):
            hostname = device.name
            self.logger.info(
                f"Checking device hostname compliance: {hostname}",
                extra={"object": device},
            )
            # Check if the hostname matches the expected pattern
            if HOSTNAME_PATTERN.match(hostname):
                self.logger.info(f"{hostname} configured hostname is correct.")
                # Skip to next iteration of the list
                continue

            # Mark the Device as failed in the job results
            self.logger.error(f"{hostname} does Not Match Hostname Pattern.")
```

我们将对 run 函数进行若干修改，添加一个简单的 JSON 数据结构。return 语句将使这些数据在后续 HTML 模板中可供迭代访问。
```python
class VerifyHostname(Job):
    location_to_check = ObjectVar(
        model=Location,
        query_params={
            "has_devices": True,
        }
    )
    
    class Meta:

        name = "Verify Hostname Pattern For Selected Locations"
        description = "Checks all devices at Designated Location for hostname pattern conformation"

    def run(self, location_to_check):
        """Run method for executing the checks on the devices."""
        results = []

        # Iterate through each Device object, limited to just the location of choice.
        for device in Device.objects.filter(location=location_to_check):
            hostname = device.name
            device_id = device.id
            self.logger.info(
                f"Checking device hostname compliance: {hostname}",
                extra={"object": device},
            )
            # Check if the hostname matches the expected pattern
            if HOSTNAME_PATTERN.match(hostname):
                self.logger.info(f"{hostname} configured hostname is correct.")
                status = "PASS"
            else:
                # Mark the Device as failed in the job results
                self.logger.error(f"{hostname} does Not Match Hostname Pattern.")
                status = "FAIL"

            results.append({
                "hostname": hostname,
                "device_id": device_id,
                "status": status,
            })

        # Return the results as part of the job result
        return {"results": results}
```

## 添加 HTML 模板

我们需要新建几个文件，并将其映射到一直使用的 `nautobot-docker-compose` Docker 容器中。

切换回 VSCode 文件资源管理器，在 ```nautobot-docker-compose``` 目录下新建一个名为 ```templates``` 的文件夹。

在该文件夹中创建两个新文件：
1. customized_jobresult.html
2. hostname_check_results.html

在 ```environments``` 目录下打开 ```docker-compose-local.yml``` 文件，内容应如下所示：
```yaml
---
services:
  nautobot:
    command: "nautobot-server runserver 0.0.0.0:8080"
    ports:
      - "8080:8080"
    volumes:
      - "../config/nautobot_config.py:/opt/nautobot/nautobot_config.py"
      - "../jobs:/opt/nautobot/jobs"
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
```

我们将在 Nautobot 服务的 volumes 列表中新增两行，用于在容器启动时将本地模板文件映射到容器内的相应目录。

更新后的配置如下所示：

> [!TIP]
> 注意 volumes 列表中新增的两行。
```yaml
---
services:
  nautobot:
    command: "nautobot-server runserver 0.0.0.0:8080"
    ports:
      - "8080:8080"
    volumes:
      - "../config/nautobot_config.py:/opt/nautobot/nautobot_config.py"
      - "../jobs:/opt/nautobot/jobs"
      - "../templates/hostname_check_results.html:/usr/local/lib/python3.8/site-packages/nautobot/extras/templates/extras/inc/hostname_check_results.html"
      - "../templates/customized_jobresult.html:/usr/local/lib/python3.8/site-packages/nautobot/extras/templates/extras/jobresult.html"
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
```

完成这些准备工作后，让我们仔细研究一下在 VSCode 中打开的 jobresult.html 文件。

该模板专门用于以标签页形式呈现单个 `JobResult` 的所有相关信息（如日志、参数、报错栈和输出），以便用户查阅。

> [!TIP]
> 以下是对该模板的简要说明，目前可能不需要了解所有细节：
> 1. **页面结构与布局**
>    - 继承 generic/object_detail.html，复用 Nautobot 的标准样式和布局。
>    - 定义多个内容"块"（breadcrumbs、buttons、content_full_width_page、advanced_content_left_page、advanced_content_right_page 等），用于组织 Job 结果数据的展示方式。
> 2. **面包屑导航**
>    - 在页面顶部渲染面包屑导航，根据 Job 是否关联 job_model、associated_record 或自定义 job class path 动态调整。
> 3. **按钮与操作**
>    - 展示以下按钮：
>      - 重新运行或运行（根据用户权限及 Job 是否支持重运行/直接运行条件显示）。
>      - 导出（提供以 CSV 格式下载 Job 日志条目的链接）。
> 4. **标题**
>    - 根据是否存在 job_model、associated_record 或 job，动态设置页面标题。
> 5. **附加数据选项卡**
>    - Job 完成后会渲染"Job Result"和"Advanced"两个选项卡。
> 6. **Job Result 选项卡**
>    - 使用包含的局部模板（extras/inc/jobresult.html）渲染 Job 结果数据的主体内容，包括日志。
> 7. **Advanced 选项卡：左侧面板**
>    - 以 JSON 格式展示：
>      - 任务关键字参数（result.task_kwargs）
>      - 任务位置参数（result.task_args）
>      - Celery 关键字参数（result.celery_kwargs）
> 8. **Advanced 选项卡：右侧面板**
>    - 展示工作进程信息，包括 worker 主机名、队列、任务名称和 Job 元数据 JSON。
>    - 展示报错栈信息（如果 Job 失败或遇到错误）。
> 9. **附加选项卡内容**
>    - 在 ```<pre>``` 块中渲染 Job 的标准输出（如有）。
> 10. **JavaScript**
>     - 包含用于处理表格配置和 Job 日志级别过滤的脚本。

在本示例中，我们主要关注文件中的"Tabs（选项卡）"和"Additional Tab Contents（附加选项卡内容）"部分。这两部分代码允许我们在 Job 结果页面中添加一个新选项卡，在 Job 完成后渲染展示。

### customized_jobresult.html —— 添加选项卡

我们将新增一个名为"Hostname Check"的第三个选项卡。查看下方的"extra_nav_tabs"代码块，它会判断 Job 是否有结果数据，若有则显示 Job Result 选项卡。
```jinja
{% block extra_nav_tabs %}
    {% if result.data.output %}
        <li role="presentation">
            <a href="#output" role="tab" data-toggle="tab">Output</a>
        </li>
    {% endif %}
{% endblock %}
```

我们将沿用此格式，但不再检查 Job 自身的结果数据，而是判断更新后代码返回的"results"是否为真值。Job 返回的数据可以通过 ```result.result.results``` 访问，其中"results"是我们返回的 JSON 结构的键名。
```jinja
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
```

这样，只有当 Job 返回了 Results JSON 结构时，"Hostname Check"选项卡才会显示，其他 Job 不会出现此选项卡。

接下来关注文件底部的 ```extra_tab_content``` 块。该块定义了页面各附加选项卡（通过 id 值引用）的内容。下方代码块判断 result.data.output 是否为真值，并渲染相应的选项卡内容。
```jinja
{% block extra_tab_content %}
    {% if result.data.output %}
        <div role="tabpanel" class="tab-pane" id="output">
            <pre>{{ result.data.output }}</pre>
        </div>
    {% endif %}
{% endblock extra_tab_content %}
```

同样地，我们判断 ```result.result.results``` 是否为真值并渲染 hostname-check 选项卡：
```jinja
{% block extra_tab_content %}
    {% if result.data.output %}
        <div role="tabpanel" class="tab-pane" id="output">
            <pre>{{ result.data.output }}</pre>
        </div>
    {% endif %}
    {% if result.result.results %}
        <div role="tabpanel" class="tab-pane" id="hostname-check">
            {% include 'extras/inc/hostname_check_results.html' %}
        </div>
    {% endif %}
{% endblock extra_tab_content %}
```

添加这两个代码块后，```customized_jobresult.html``` 的完整内容应如下所示：
```html
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
            {% include 'extras/inc/hostname_check_results.html' %}
        </div>
    {% endif %}
{% endblock extra_tab_content %}


{% block javascript %}
    {{ block.super }}
    {% include 'extras/inc/jobresult_js.html' with result=result %}
    <script src="{% versioned_static 'js/tableconfig.js' %}"></script>
    <script src="{% versioned_static 'js/log_level_filtering.js' %}"></script>
{% endblock %}
```

了解了 ```jobresult.html``` 需要调整的内容后，将上述模板复制到我们新建的 ```customized_jobresult.html``` 文件中，注意其中已更新的 ```extra_nav_tabs``` 块和 ```extra_tab_content``` 块。

### hostname_check_results.html —— 扩展基础模板

上一步中我们引用了一个尚不存在的文件 ```extras/inc/hostname_check_results.html```，这正是我们在 ```nautobot-docker-compose/template``` 目录中创建的文件，并在 ```extra_tab_content``` 块中链接到了相应位置。

> [!TIP]
> 以这种方式链接文件，可以在本地编辑文件后立即在容器中看到变更，无需手动移动文件。

打开 ```nautobot-docker-compose/template``` 目录下的 ```hostname_check_results.html``` 文件，添加以下内容：
```jinja2
{% if result.result.results %}
    <h1>Hostname Check Results Table</h1>
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
{% endif %}
```

这是一个非常简洁的模板，以两列（Hostname 和 Status）表格格式展示输出，使用 `label-success` 和 `label-danger` 样式类分别以绿色和红色标示通过（PASS）或失败（FAIL）的结果。

完成上述操作后，停止容器后重新启动，可以在终端中按 ```Ctrl+C``` 停止调试进程。
```bash
nautobot-1       |   "GET /extras/job-results/2f21d935-942d-4f11-b19d-b50d4562263d/?tab=main HTTP/1.1" 200 215322
nautobot-1       | 17:25:05.031 INFO    django.server :
nautobot-1       |   "GET /template.css HTTP/1.1" 200 3984
nautobot-1       | 17:25:05.083 INFO    django.server :
nautobot-1       |   "GET /extras/job-results/2f21d935-942d-4f11-b19d-b50d4562263d/log-table/ HTTP/1.1" 200 5852
Gracefully stopping... (press Ctrl+C again to force)
 Container nautobot_docker_compose-celery_worker-1  Stopping
 Container nautobot_docker_compose-celery_beat-1  Stopping
 Container nautobot_docker_compose-celery_beat-1  Stopped
 Container nautobot_docker_compose-celery_worker-1  Stopped
 Container nautobot_docker_compose-nautobot-1  Stopping
 Container nautobot_docker_compose-nautobot-1  Stopped
 Container nautobot_docker_compose-redis-1  Stopping
 Container nautobot_docker_compose-db-1  Stopping
 Container nautobot_docker_compose-db-1  Stopped
 Container nautobot_docker_compose-redis-1  Stopped
canceled
(nautobot-docker-compose-py3.10) bbaker4@ByrnsGameRig:~/github-projects/100-days-of-nautobot/nautobot-docker-compose$ 
```

容器完全停止后，使用 ```invoke build debug``` 重新启动所有服务。
```bash
$ invoke build debug
Building Nautobot 2.3.2 with Python 3.8...
Running docker compose command "build"
 Service nautobot  Building
...
nautobot-1       | System check identified 5 issues (0 silenced).
nautobot-1       | 17:30:23.437 INFO    django.utils.autoreload :
nautobot-1       |   Watching for file changes with StatReloader
nautobot-1       | Performing system checks...
nautobot-1       | 
nautobot-1       | System check identified no issues (0 silenced).
nautobot-1       | 17:30:24.636 INFO    nautobot             __init__.py                              setup() :
nautobot-1       |   Nautobot initialized!
nautobot-1       | January 31, 2025 - 17:30:24
nautobot-1       | Django version 4.2.16, using settings 'nautobot_config'
nautobot-1       | Starting development server at http://0.0.0.0:8080/
nautobot-1       | Quit the server with CONTROL-C.
```

进入 Jobs 菜单，选择任意位置运行 ```Verify Hostname Pattern For Selected Locations``` Job。执行完成后，结果页面应出现第三个名为"Hostname Check"的选项卡。

![Job Results](images/job-results.jpg)

在"Hostname Check"选项卡中，可以看到主机名验证结果的数据表格。

![Hostname Table](images/hostname_check.jpg)

如果想在模板底部添加导出按钮，可以在 `hostname_check_results.html` 模板中添加 CSS 代码来支持 CSV 导出功能。

在 ```</table>``` 标签下方，在"IF"判断块内添加以下内容：
```
    </table>
        <button id="export-results" class="btn btn-primary">Export Results</button>
```

然后在 ```{% endif %}``` 标签下方添加以下 JavaScript：
```JavaScript
<script>
    document.addEventListener('DOMContentLoaded', function() {
        document.getElementById('export-results').addEventListener('click', function() {
            var table = document.querySelector('#hostname-check table');
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
            var a         = document.createElement('a');
            a.href        = 'data:attachment/csv,' +  encodeURIComponent(csvString);
            a.target      = '_blank';
            a.download    = 'hostname_check_results.csv';
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
        });
    });
</script>
```

刷新页面或重新运行 Job。在"Hostname Check"选项卡中，现在应出现一个"Export"按钮，点击后可下载表格的 CSV 版本。

![Hostname Export](images/hostname_check_with_export.jpg)

## 第 23 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布新 Job 成功执行的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将深入探讨 Job 测试。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+23+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 23 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
