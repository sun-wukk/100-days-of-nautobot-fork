# Django 入门（四）：添加表单

今天是四部曲的收官之战，我们将为用户添加投票表单。

本次挑战大致参照 [Django 教程第四部分](https://docs.djangoproject.com/en/5.1/intro/tutorial04/)，请参阅原文了解代码的更多细节。

## 代码示例

首先更新视图，添加 `vote`（投票）和 `results`（结果）两个函数视图。每个函数都会指定用于渲染最终页面的 HTML 模板：

```python
# polls/views.py
from django.shortcuts import render
from django.http import HttpResponseRedirect
from django.urls import reverse

from .models import Question, Choice

def index(request):
    latest_question_list = Question.objects.order_by('-pub_date')[:5]
    context = {'latest_question_list': latest_question_list}
    return render(request, 'polls/index.html', context)

def detail(request, question_id):
    question = Question.objects.get(pk=question_id)
    return render(request, 'polls/detail.html', {'question': question})

def vote(request, question_id):
    question = Question.objects.get(pk=question_id)
    try:
        selected_choice = question.choice_set.get(pk=request.POST['choice'])
    except (KeyError, Choice.DoesNotExist):
        return render(request, 'polls/detail.html', {
            'question': question,
            'error_message': "You didn't select a choice.",
        })
    else:
        selected_choice.votes += 1
        selected_choice.save()
        return HttpResponseRedirect(reverse('results', args=(question.id,)))

def results(request, question_id):
    question = Question.objects.get(pk=question_id)
    return render(request, 'polls/results.html', {'question': question})
```

修改 `detail.html`，加入带单选按钮的投票表单：

```html
<!DOCTYPE html>
<html>
<head>
    <title>{{ question.question_text }}</title>
</head>
<body>
    <h1>{{ question.question_text }}</h1>
    <form action="{% url 'vote' question.id %}" method="post">
        {% csrf_token %}
        {% for choice in question.choice_set.all %}
            <input type="radio" name="choice" id="choice{{ forloop.counter }}" value="{{ choice.id }}">
            <label for="choice{{ forloop.counter }}">{{ choice.choice_text }}</label><br>
        {% endfor %}
        <input type="submit" value="Vote">
    </form>
    {% if error_message %}
        <p><strong>{{ error_message }}</strong></p>
    {% endif %}
</body>
</html>
```

在 `polls/urls.py` 中添加新的 URL 路径：

```python
# polls/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('', views.index, name='index'),
    path('<int:question_id>/', views.detail, name='detail'),
    path('<int:question_id>/results/', views.results, name='results'),
    path('<int:question_id>/vote/', views.vote, name='vote'),
]
```

创建 `results` 结果页模板：

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
            <li>{{ choice.choice_text }} -- {{ choice.votes }} vote{{ choice.votes|pluralize }}</li>
        {% endfor %}
    </ul>
    <a href="{% url 'index' %}">Back to Polls</a>
</body>
</html>
```

更新 `admin.py`，支持在管理后台添加选项条目：

```python
# polls/admin.py
from django.contrib import admin
from .models import Question, Choice

admin.site.register(Question)
admin.site.register(Choice)
```

重启开发服务器：

```shell
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ python manage.py runserver 0.0.0.0:8080
```

现在可以在管理后台创建选项了。注意 Question 与 Choice 之间的关联关系——需要从下拉菜单中选择已有的问题：

![choice_1](images/choice_1.png)

各选项会列示在主页面上：

![choice_2](images/choice_2.png)

打开 `polls` 页面，开始投票：

![vote_1](images/vote_1.png)

投票完成后，页面会自动跳转到结果页：

![vote_2](images/result_1.png)

最终的目录结构如下：

```shell
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject $ pwd
/home/vscode/djangoproject

(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject $ tree .
.
├── mysite
│   ├── db.sqlite3
│   ├── manage.py
│   ├── mysite
│   │   ├── asgi.py
│   │   ├── __init__.py
│   │   ├── __pycache__
│   │   │   ├── __init__.cpython-310.pyc
│   │   │   ├── settings.cpython-310.pyc
│   │   │   ├── urls.cpython-310.pyc
│   │   │   └── wsgi.cpython-310.pyc
│   │   ├── settings.py
│   │   ├── urls.py
│   │   └── wsgi.py
│   └── polls
│       ├── admin.py
│       ├── apps.py
│       ├── __init__.py
│       ├── migrations
│       │   ├── 0001_initial.py
│       │   ├── __init__.py
│       │   └── __pycache__
│       │       ├── 0001_initial.cpython-310.pyc
│       │       └── __init__.cpython-310.pyc
│       ├── models.py
│       ├── __pycache__
│       │   ├── admin.cpython-310.pyc
│       │   ├── apps.cpython-310.pyc
│       │   ├── __init__.cpython-310.pyc
│       │   ├── models.cpython-310.pyc
│       │   ├── urls.cpython-310.pyc
│       │   └── views.cpython-310.pyc
│       ├── templates
│       │   └── polls
│       │       ├── detail.html
│       │       ├── index.html
│       │       └── results.html
│       ├── tests.py
│       ├── urls.py
│       └── views.py
└── pyproject.toml

9 directories, 32 files
```

恭喜完成这四部曲的全部挑战！

## 总结与思考

以下是 Django 与 Nautobot 的一些对比思考：

- 我们的 Django 应用无需 Celery、Redis 或 Beats，因为它不涉及异步任务的执行。
- 我们使用 `makemigrations` 和 `migrate` 提交数据库变更；而在前几天的 Nautobot 开发中，`invoke` 命令替我们处理了这些步骤及其他事项。
- 这个简单的 Django 应用使用 SQLite 作为数据库，整个数据库就是一个单文件；而在 Nautobot 的部署中，我们使用 PostgreSQL 容器作为数据库。
- 不难发现，`manage.py` 与 `nautobot-server` 的功能十分相似。
- `mysite` 相当于 Nautobot 本身，`polls` 则相当于各个 Nautobot App。
- 在这个简单的 Django 示例中，我们忽略了测试、安全及许多其他重要方面。
- 特别提醒：开发服务器仅供本地调试，不适用于生产环境。

接下来的几天，我们将把这些 Django 知识融会贯通，运用到 Nautobot App 的实际开发中。

## 第 49 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 Codespace 实例。

请在你选择的社交媒体上发布 Django 应用最终版本的截图，务必使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，这样我们可以分享你的进展！

明天的挑战，我们将把这些 Django 概念带入 Nautobot App 开发的实战环境。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+49+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 49 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
