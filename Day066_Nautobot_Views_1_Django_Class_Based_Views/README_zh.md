# Django 类视图简介

Django 提供了两种主要的定义视图的方式：函数视图（FBV）和类视图（CBV）。每种方法都有自己的优势，适用于不同的场景。对于 Nautobot，类视图是主要使用的方式。

到目前为止，我们主要使用函数视图。在今天的挑战中，我们将介绍 Django 类视图的基础知识以及它们与函数视图的区别。

## 环境设置

对于今天的挑战，我们将把 [第 49 天](https://github.com/nautobot/100-days-of-nautobot/blob/main/Day049_Django_Example_Part_4_Add_Forms/README.md) 的 Python 视图从函数视图转换为类视图。

请重复 [第 46 天](https://github.com/nautobot/100-days-of-nautobot/blob/main/Day046_Django_Example_Part_1_Project_Setup_and_Creating_App/README.md) 到 [第 49 天](https://github.com/nautobot/100-days-of-nautobot/blob/main/Day049_Django_Example_Part_4_Add_Forms/README.md) 的步骤来创建 `polls` 应用。

## Django 函数视图回顾

让我们看一下 [第 49 天](https://github.com/nautobot/100-days-of-nautobot/blob/main/Day049_Django_Example_Part_4_Add_Forms/README.md) 使用 Python 函数的 `views.py` 文件（函数视图）：

```python polls.views.py
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

函数视图简单易懂。Python 函数至少将 `request` 对象作为参数，并返回带有渲染 HTML 模板的响应。对于 `detail()`、`vote()` 和 `results()` 视图，它们还额外接收 `questions_id` 作为参数。

默认情况下，`request.method` 被假定为 `GET`，如果是 `POST`、`PUT` 或 `DELETE` 方法，我们需要使用 `if/else` 语句添加额外代码。例如：

```python
from django.http import HttpResponse
from django.shortcuts import render

def my_view(request):
    if request.method == 'GET':
        # GET 请求的逻辑
        return render(request, 'my_template.html')
    elif request.method == 'POST':
        # POST 请求的逻辑
        return HttpResponse("Data received!")
    else:
         return HttpResponse("Method not allowed", status=405)
```

尽管函数视图很简单，但 Python 函数缺乏代码重用、继承等高级功能，对于更复杂的视图会变得复杂。

由于这些原因，Django 从大约 1.3 版本开始引入了一种新方法——[类视图](https://docs.djangoproject.com/en/5.1/topics/class-based-views/intro/)。

## Django 类视图

类视图提供了一种**面向对象（OO）**方法来组织你的视图代码。它们允许我们使用 Python 类而不是函数来构建视图，这可以带来更多可重用、可维护和可测试的代码。

在类视图中，Django 中的视图被表示为类。它们提供了一种在类中封装视图逻辑的方法，使得重用和扩展更容易。缺点是如果用户不熟悉面向对象编程，学习曲线可能很陡峭。

### 转换为类视图

我们可以将之前的函数视图转换为类视图（注意从 `django.views` 额外导入了 `generic`）。我们可以简单地添加新视图而不删除之前的函数视图：

```python
from django.shortcuts import render
from django.http import HttpResponseRedirect
from django.urls import reverse

from django.views import generic  # 新增

from .models import Question, Choice


class IndexView(generic.ListView):
    template_name = "polls/index.html"
    context_object_name = "latest_question_list"

    def get_queryset(self):
        """返回最后五个已发布的问题。"""
        return Question.objects.order_by("-pub_date")[:5]


class DetailView(generic.DetailView):
    model = Question
    template_name = "polls/detail.html"


class ResultsView(generic.DetailView):
    model = Question
    template_name = "polls/results.html"

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
```

我们将在 `urls.py` 中切换到新视图：

```python
from django.urls import path
from . import views

urlpatterns = [
    # path('', views.index, name='index'),
    # path('<int:question_id>/', views.detail, name='detail'),
    # path('<int:question_id>/results/', views.results, name='results'),
    # path('<int:question_id>/vote/', views.vote, name='vote'),
    path("", views.IndexView.as_view(), name="index"),
    path("<int:pk>/", views.DetailView.as_view(), name="detail"),
    path("<int:pk>/results/", views.ResultsView.as_view(), name="results"),
    path("<int:question_id>/vote/", views.vote, name="vote"),
]
```

启动 `polls` 应用以验证功能保持不变：

![polls_index](images/polls_index.png)

![polls_vote](images/polls_vote.png)

![polls_result](images/polls_result.png)

我们没有更改 `vote()` FBV，因为切换它不会节省更多工作，更多详情请看 [Django 教程第 4 部分](https://docs.djangoproject.com/en/5.1/intro/tutorial04/)。

### 类视图的优势

对于简单的视图可能不太明显，但类视图（CBV）为除了最简单视图之外的所有视图提供了相对于函数视图（FBV）的明显优势。

CBV 的一些优势：

1. **可重用性**：CBV 可以轻松地在应用程序的不同部分重用。你可以创建具有通用功能的基础类，并根据需要扩展它们。正如我们在示例中看到的，我们简单地为所有 CBV 使用 `django.views.generic`。

2. **可扩展性**：CBV 允许你使用继承来扩展和自定义视图。这使得通过组合和扩展更简单的视图来创建复杂视图变得容易。

3. **可维护性**：CBV 促进了一种更有组织和结构化的编写视图方式，使你的代码更容易阅读和维护。

我们在示例中使用了 `generic.ListView` 和 `generic.DetailView`，但 Django 还提供了许多其他开箱即用的类视图。

### Django 中常见的类视图

以下是 Django 中一些流行的类视图的总结：

1. **TemplateView**：渲染模板。它是用于渲染带有上下文的模板的简单视图。

2. **ListView**：渲染对象列表。它是用于显示来自*查询集*的对象列表的视图。

3. **DetailView**：渲染单个对象的详情页。用于显示单个对象的详细信息。

4. **CreateView**：渲染用于创建新对象的表单。它处理表单提交和对象创建。

5. **UpdateView**：渲染用于更新现有对象的表单。它处理表单提交和对象更新。

6. **DeleteView**：渲染用于删除对象的确认页。确认后处理对象删除。

### 资源

类视图可能需要一点时间适应刚接触 Django 的人，这就是为什么我们等到**第 66 天**才正式介绍它的原因。

从长远来看，CBV 相对于 FBV 提供的优势超过了学习曲线。对于包括 Nautobot 在内的新项目，类视图几乎总是默认选择。

以下是一些可帮助学习更多类视图的额外资源：

- [类视图简介](https://docs.djangoproject.com/en/5.1/topics/class-based-views/intro/)
- [内置类视图通用视图](https://docs.djangoproject.com/en/5.1/topics/class-based-views/generic-display/)

恭喜完成第 66 天！

## 第 66 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布今天挑战中新类视图的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将学习 Nautobot 的类视图。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+66+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 66 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）