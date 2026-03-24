# Nautobot 过滤器

过滤器是 Nautobot 中的重要组件，它允许我们根据特定条件缩小数据集。它们在各种区域广泛使用，如 API、表单和 UI。

今天，我们将探索过滤器在 Nautobot 中的使用方式以及有效实现它们的最佳实践。

## 环境设置

我们将结合使用 [场景 2](../Lab_Setup/scenario_2_setup/README.md) 实验室、[https://demo.nautobot.com/](https://demo.nautobot.com/) 和 [Nautobot 文档](https://docs.nautobot.com/projects/core/en/latest/user-guide/core-data-model/overview/introduction/) 进行今天的挑战。

```
$ cd nautobot
$ poetry shell
$ poetry install
$ invoke build
（这个步骤需要耐心）
$ invoke debug
（这个步骤也需要耐心）
```

### 理解 FilterSet 类

以下示例取自 `example_app -> filters.py` 中的现有代码片段。

- **nautobot.apps.filters 类**：
  - 这些类定义了如何将过滤应用于查询或 `queryset`。
  - 通常存在于 `filters.py` 文件中。

```python
from nautobot.apps.filters import BaseFilterSet, SearchFilter

class UsefulLinkModelFilterSet(BaseFilterSet):
    """用于过滤有用链接模型对象的 API 过滤器。"""

    q = SearchFilter(
        filter_predicates={
            "url": "icontains",
            "description": "icontains",
        },
    )
    ...
    # 用于在描述中按多个关键词过滤的自定义过滤器方法
    def filter_description_keywords(self, queryset, name, value):
        keywords = value.split()
        for keyword in keywords:
            queryset = queryset.filter(description__icontains=keyword)
        return queryset
    ...
    # 用于不区分大小写 URL 过滤的自定义过滤器方法
    def filter_url_case_insensitive(self, queryset, name, value):
        return queryset.filter(url__icontains=value)
```

### 增强和自定义过滤器字段

- **增强过滤器**：
  - Nautobot 过滤器支持多个值，可以按关键词或值过滤记录。
  - 示例：`queryset = queryset.filter(description__icontains=keyword)`，`queryset.filter(url__icontains=value)`。

- **Django 过滤器**：
  - nautobot 过滤器允许应用 `django_filters` 的定制查询。
  - 示例：

```python
...
    # 用于在描述中按多个关键词过滤的自定义过滤器方法
    def filter_description_keywords(self, queryset, name, value):
        keywords = value.split()
        for keyword in keywords:
            queryset = queryset.filter(description__icontains=keyword)
        return queryset

    description_keywords = django_filters.CharFilter(method='filter_description_keywords', label='描述（关键词）')

    # 用于不区分大小写 URL 过滤的自定义过滤器方法
    def filter_url_case_insensitive(self, queryset, name, value):
        return queryset.filter(url__icontains=value)

    url_ci = django_filters.CharFilter(method='filter_url_case_insensitive', label='URL（不区分大小写）')
...
```

### 使用查找表达式

- **查找表达式**：
  - 过滤器使用各种查找表达式（`icontains`、`gt`、`lt`、`isnull` 等）来提供数据过滤的灵活性。
  - 这些表达式支持比较、模式匹配和空值检查。

### 实现过滤器的最佳实践

- **声明适当的查找表达式**：
  - 确保过滤器声明必要的查找表达式以满足你的过滤需求。

- **必要时使用 `exclude=True`**：
  - 如果查询需要排除某些条目，请使用 `exclude=True` 参数。

- **使用 `distinct=True` 获取唯一结果**：
  - 为确保唯一结果，请在过滤器中设置 `distinct=True`。

### 与 Django ORM 和 API 的集成

- **无缝集成**：
  - 过滤器与 Django 的 ORM 无缝集成，为 Django 开发者提供熟悉的语法。
  - 它们通过提供全面且用户友好的过滤能力来增强 API 体验。

## 代码示例

同样，让我们回顾一下我们的 `example_app` `UsefulLink` 模型视图，我们在那里使用 `filterset_class = filters.UsefulLinkModelFilterSet` 进行过滤：

```python
class UsefulLinkUIViewSet(views.NautobotUIViewSet):
    queryset = UsefulLink.objects.all()
    ...
    filterset_class = filters.UsefulLinkModelFilterSet
    ...
```

`filters.py` 中的初始搜索过滤器代码如下：

```python
class UsefulLinkModelFilterSet(BaseFilterSet):
    """用于过滤 usefullink 模型对象的 API 过滤器。"""

    q = SearchFilter(
        filter_predicates={
            "url": "icontains",
            "description": "icontains",
        },
    )

    class Meta:
        model = UsefulLink
        fields = [
            "url",
            "description",
        ]
```

我们可以添加更多过滤器来增强搜索功能：

```python
import django_filters

class UsefulLinkModelFilterSet(BaseFilterSet):
    """用于过滤有用链接模型对象的 API 过滤器。"""

    q = SearchFilter(
        filter_predicates={
            "url": "icontains",
            "description": "icontains",
        },
    )

    # 用于在描述中按多个关键词过滤的自定义过滤器方法
    def filter_description_keywords(self, queryset, name, value):
        keywords = value.split()
        for keyword in keywords:
            queryset = queryset.filter(description__icontains=keyword)
        return queryset

    description_keywords = django_filters.CharFilter(method='filter_description_keywords', label='描述（关键词）')

    # 用于不区分大小写 URL 过滤的自定义过滤器方法
    def filter_url_case_insensitive(self, queryset, name, value):
        return queryset.filter(url__icontains=value)

    url_ci = django_filters.CharFilter(method='filter_url_case_insensitive', label='URL（不区分大小写）')

    class Meta:
        model = UsefulLink
        fields = [
            "url",
            "description",
            "description_keywords",
            "url_ci",
        ]
```

现在，如果我们在点击 `Filter` 后查看 `Advanced` 表，会看到新的过滤器字段：

![new_filters](images/new_filters.png)

作为参考，以下是我们完成后 `example_app` 下完整的 `filters.py` 文件：

```python
from nautobot.apps.filters import BaseFilterSet, SearchFilter
import django_filters

from example_app.models import AnotherExampleModel, ExampleModel, UsefulLink


class UsefulLinkModelFilterSet(BaseFilterSet):
    """用于过滤有用链接模型对象的 API 过滤器。"""

    q = SearchFilter(
        filter_predicates={
            "url": "icontains",
            "description": "icontains",
        },
    )

    # 用于在描述中按多个关键词过滤的自定义过滤器方法
    def filter_description_keywords(self, queryset, name, value):
        keywords = value.split()
        for keyword in keywords:
            queryset = queryset.filter(description__icontains=keyword)
        return queryset

    description_keywords = django_filters.CharFilter(method='filter_description_keywords', label='描述（关键词）')

    # 用于不区分大小写 URL 过滤的自定义过滤器方法
    def filter_url_case_insensitive(self, queryset, name, value):
        return queryset.filter(url__icontains=value)

    url_ci = django_filters.CharFilter(method='filter_url_case_insensitive', label='URL（不区分大小写）')

    class Meta:
        model = UsefulLink
        fields = [
            "url",
            "description",
            "description_keywords",
            "url_ci",
        ]


class ExampleModelFilterSet(BaseFilterSet):
    """用于过滤示例模型对象的 API 过滤器。"""

    q = SearchFilter(
        filter_predicates={
            "name": "icontains",
            "number": "icontains",
        },
    )

    class Meta:
        model = ExampleModel
        fields = [
            "name",
            "number",
        ]


class AnotherExampleModelFilterSet(BaseFilterSet):
    """用于过滤另一个示例模型对象的 API 过滤器。"""

    q = SearchFilter(
        filter_predicates={
            "name": "icontains",
            "number": "icontains",
        },
    )

    class Meta:
        model = AnotherExampleModel
        fields = [
            "name",
            "number",
        ]
```

恭喜完成第 79 天！

## 第 79 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布今天挑战中新过滤器的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将开始一个新的多天-long  capstone 项目之旅。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+79+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 79 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）