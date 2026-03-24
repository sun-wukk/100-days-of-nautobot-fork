# Django 入门（一）：搭建项目与创建应用

在本次挑战中，我们多次提到 Nautobot 是基于 Python Django 框架构建的。正如我们所见，使用 Nautobot 并不需要掌握 Django。但随着我们在 Nautobot App 开发之路上走得越来越深，具备一定的 Django 基础将大有裨益，有助于理解 Django 与 Nautobot 之间的内在联系。

Django 以其详尽全面的文档著称于世——毕竟，来自一家屡获殊荣的新闻机构的 Web 框架，文档怎么能不扎实？:)

我们将官方的七部曲 [Django 教程](https://docs.djangoproject.com/en/5.1/intro/tutorial01/)浓缩成四个章节，安排在第 46 至 49 天，旨在帮助大家掌握 Django 主要组件的基本概念，从而加深对 Nautobot 的理解。

> [!IMPORTANT]
> 这是一个精简版教程，节奏较快，重点在于快速搭出一个可运行的应用，以便直观感受 Django 的各个核心组成部分。[Django 官方教程](https://docs.djangoproject.com/en/5.1/intro/tutorial01/)对细节有非常出色的讲解，并附有丰富的延伸资料，如需深入了解请参阅原文。

今天的挑战，我们将搭建一个 Django 项目并在其中创建一个应用。

具体步骤如下：

- 创建虚拟环境
- 创建 Django 项目
- 创建应用并将其注册到项目中
- 编写初始视图，并将用户路由到该视图

让我们开始吧。

## 环境搭建

我们将继续使用一直以来的 Codespace 环境，无需启动 `nautobot-docker-compose` 容器，后续所有必要步骤都会在这几天中逐一介绍。

## 代码示例

从默认主目录 `/home/vscode` 出发，创建一个存放新项目的目录，并用 `poetry init` 初始化虚拟环境：

```
@ericchou1 ➜ ~ $ mkdir djangoproject
@ericchou1 ➜ ~ $ cd djangoproject/
@ericchou1 ➜ ~/djangoproject $ poetry init

This command will guide you through creating your pyproject.toml config.

Package name [djangoproject]:  
Version [0.1.0]:  
Description []:  
Author [Name <email>, n to skip]:  
License []:  
Compatible Python versions [^3.10]:  

Would you like to define your main dependencies interactively? (yes/no) [yes] 
You can specify a package in the following forms:
  - A single name (requests): this will search for matches on PyPI
  - A name and a constraint (requests@^2.23.0)
  - A git url (git+https://github.com/python-poetry/poetry.git)
  - A git url with a revision (git+https://github.com/python-poetry/poetry.git#develop)
  - A file path (../my-package/my-package.whl)
  - A directory (../my-package/)
  - A url (https://example.com/packages/my-package-0.1.0.tar.gz)

Package to add or search for (leave blank to skip): 

Would you like to define your development dependencies interactively? (yes/no) [yes] 
Package to add or search for (leave blank to skip): 

Generated file

[tool.poetry]
name = "djangoproject"
version = "0.1.0"
description = ""
authors = ["Name <email>"]
readme = "README.md"

[tool.poetry.dependencies]
python = "^3.10"


[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"


Do you confirm generation? (yes/no) [yes] 
```

创建完成后，用 `poetry shell` 激活虚拟环境：

```
@ericchou1 ➜ ~/djangoproject $ poetry shell
Creating virtualenv djangoproject-jP4IF3vC-py3.10 in /home/vscode/.cache/pypoetry/virtualenvs
Spawning shell within /home/vscode/.cache/pypoetry/virtualenvs/djangoproject-jP4IF3vC-py3.10
```

用 `pip` 安装 Django，再用 `django-admin` 命令初始化一个新的 Django 项目，然后进入项目目录：

```
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject $ pip install django
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject $ django-admin --version
5.1.7

(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject $ django-admin startproject mysite

(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject $ cd mysite/
```

此时已可以启动开发服务器，查看基本的首页效果：

```
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ python manage.py runserver 0.0.0.0:8080
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).

You have 18 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, sessions.
Run 'python manage.py migrate' to apply them.
March 07, 2025 - 14:54:44
Django version 5.1.7, using settings 'mysite.settings'
Starting development server at http://0.0.0.0:8080/
Quit the server with CONTROL-C.
```

![django_hello_world_1](images/django_hello_world_1.png)

按 `CONTROL-C` 停止服务器。

项目目录（本例中为 `mysite`）包含项目级别的工具，其中 `manage.py` 可用于执行各类项目级任务，例如启动开发服务器、创建管理员账户等。

仔细观察会发现，目录内有一个与项目同名的子目录 `mysite`，其中包含控制配置和顶层 URL 路由的文件：

```
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ pwd
/home/vscode/djangoproject/mysite
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ tree mysite/
mysite/
├── asgi.py
├── __init__.py
├── __pycache__
│   ├── __init__.cpython-310.pyc
│   ├── settings.cpython-310.pyc
│   ├── urls.cpython-310.pyc
│   └── wsgi.cpython-310.pyc
├── settings.py
├── urls.py
└── wsgi.py

1 directory, 9 files
```

Django 的组织方式是将每个应用独立成一个文件夹。对于简单的应用来说，这看似繁琐，但从长远来看，这种"约定大于配置"的思路非常有利于关注点分离。

用 `manage.py` 创建一个名为 `polls` 的新应用：

```
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ python manage.py startapp polls
```

至此，目录结构如下：`djangoproject` 存放虚拟环境文件，子目录 `djangoproject/mysite` 包含项目级配置，`djangoproject/mysite/mysite` 是站点入口，`djangoproject/mysite/polls` 则包含 `polls` 应用的所有文件：

```
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ tree .
.
├── db.sqlite3
├── manage.py
├── mysite
│   ├── asgi.py
│   ├── __init__.py
│   ├── __pycache__
│   │   ├── __init__.cpython-310.pyc
│   │   ├── settings.cpython-310.pyc
│   │   ├── urls.cpython-310.pyc
│   │   └── wsgi.cpython-310.pyc
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
└── polls
    ├── admin.py
    ├── apps.py
    ├── __init__.py
    ├── migrations
    │   └── __init__.py
    ├── models.py
    ├── tests.py
    └── views.py

4 directories, 18 files
```

在项目级配置文件 `djangoproject/mysite/mysite/settings.py` 中，将新应用添加到已安装应用列表：

```
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ vim mysite/settings.py

# Application definition

INSTALLED_APPS = [
    'django.contrib.admin',
    ...
    'polls',
]
```

在主项目的 `urls.py` 中，将所有 `polls/` 路径的请求转发给 `polls` 应用内的 `urls.py` 处理：

```
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ cat mysite/urls.py 

from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path('polls/', include('polls.urls')),
    path('admin/', admin.site.urls),
]
```

在 `polls` 应用下新建 `urls.py` 文件：

```
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ touch polls/urls.py
```

在该文件中，用空字符串 `''` 表示根路径，将其指向 `views` 文件中名为 `index` 的视图（稍后创建）：

```
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ cat polls/urls.py 
from django.urls import path
from . import views

urlpatterns = [
    path('', views.index, name='index'),
]
```

在 `polls/views.py` 中，用简单的 `HttpResponse` 实现 `index` 视图：

```
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ cat polls/views.py 
from django.http import HttpResponse

def index(request):
    return HttpResponse("Hello, world. You're at the polls index.")
```

再次启动开发服务器，查看刚创建的视图：

```
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ python manage.py runserver 0.0.0.0:8080
```

访问 `polls/` 路径，新视图如期呈现：

![django_polls_index_1](images/django_polls_index_1.png)

看似平淡无奇，却值得小小庆祝——我们刚刚完整地创建了第一个 Django 项目，在其中添加了一个新应用，并让用户能够访问它。

请停止 Codespace，但**不要删除**，接下来几天我们还会继续在这个项目上开展工作。

## 第 46 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 Codespace 实例。

请在你选择的社交媒体上发布今天创建的新应用截图，务必使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，这样我们可以分享你的进展！

明天的挑战，我们将创建新的数据库模型。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+46+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 46 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
