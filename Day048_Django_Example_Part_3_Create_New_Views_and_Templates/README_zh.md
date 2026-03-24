# Django 入门（三）：创建视图与模板

今天，我们将为过去两天搭建的 `polls` 应用创建新的视图和模板。

本次挑战大致参照 Django 官方文档的[教程第三部分](https://docs.djangoproject.com/en/5.1/intro/tutorial03/)，该页面对视图、URL 匹配模式、查询集及相关代码有更详尽的讲解，建议参阅原文深入理解。

> [!IMPORTANT]
> 这是一个精简版教程，节奏较快，重点在于快速搭出一个可运行的应用，以便直观感受 Django 的各个核心组成部分。[Django 官方教程](https://docs.djangoproject.com/en/5.1/intro/tutorial01/)对细节有非常出色的讲解，并附有丰富的延伸资料，如需深入了解请参阅原文。

## 代码示例

按如下内容修改 `polls/views.py`，关于 Django 查询集与上下文的详细说明，请参阅[教程第三部分](https://docs.djangoproject.com/en/5.1/intro/tutorial03/)：

```
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ cat polls/views.py 
from django.shortcuts import render
from .models import Question

def index(request):
    latest_question_list = Question.objects.order_by('-pub_date')[:5]
    context = {'latest_question_list': latest_question_list}
    return render(request, 'polls/index.html', context)

def detail(request, question_id):
    question = Question.objects.get(pk=question_id)
    return render(request, 'polls/detail.html', {'question': question})
```

更新 `polls/urls.py`，将详情视图的路由接入其中：

```
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ cat polls/urls.py 
from django.urls import path
from . import views

urlpatterns = [
    path('', views.index, name='index'),
    path('<int:question_id>/', views.detail, name='detail'),
]
```

观察视图代码可以发现，它依赖 `polls/index.html` 和 `polls/detail.html` 这两个 HTML 模板。

模板的默认存放位置往往让初学者感到困惑。Django 默认在应用目录下查找 `templates` 子目录，再按指定路径寻找模板文件。由于我们指定的路径是 `polls/<文件名>`，因此需要创建 `polls/templates/polls` 目录，并在其中放置 `index.html` 和 `detail.html`：

```
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ mkdir -p polls/templates/polls
```

为了更直观地展示，以下是 `polls/` 目录的完整结构：

```shell
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ tree polls/
polls/
├── admin.py
├── apps.py
├── __init__.py
├── migrations
│   ├── 0001_initial.py
│   ├── __init__.py
│   └── __pycache__
│       ├── 0001_initial.cpython-310.pyc
│       └── __init__.cpython-310.pyc
├── models.py
├── __pycache__
│   ├── admin.cpython-310.pyc
│   ├── apps.cpython-310.pyc
│   ├── __init__.cpython-310.pyc
│   ├── models.cpython-310.pyc
│   ├── urls.cpython-310.pyc
│   └── views.cpython-310.pyc
├── templates
│   └── polls
│       ├── detail.html
│       └── index.html
├── tests.py
├── urls.py
└── views.py

5 directories, 19 files
```

`index.html` 代码如下：

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Polls Index</title>
</head>
<body>
    <h1>Polls</h1>
    <ul>
        {% for question in latest_question_list %}
            <li><a href="{% url 'detail' question.id %}">{{ question.question_text }}</a></li>
        {% endfor %}
    </ul>
</body>
</html>
```

`detail.html` 代码如下：

```html
<!DOCTYPE html>
<html>
<head>
    <title>{{ question.question_text }}</title>
</head>
<body>
    <h1>{{ question.question_text }}</h1>
    <ul>
        {% for choice in question.choice_set.all %}
            <li>{{ choice.choice_text }}</li>
        {% endfor %}
    </ul>
</body>
</html>
```

重新启动开发服务器：

```
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ python manage.py runserver 0.0.0.0:8080
```

这是 `polls` 应用的首页：

![polls_1](images/polls_1.png)

点击某个问题，可以查看该问题的详情：

![polls_2](images/polls_2.png)

目前投票统计功能尚未实现，我们将在明天的挑战中补上这一部分。

## 第 48 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 Codespace 实例。

请在你选择的社交媒体上发布今天创建的新页面截图，务必使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，这样我们可以分享你的进展！

明天的挑战，我们将完成这个 Django 示例应用的收尾工作。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+48+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 48 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
