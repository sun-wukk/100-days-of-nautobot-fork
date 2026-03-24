# 示例应用创建新视图 - 第一部分

在第 52 天，我们创建了一个新的数据模型 `UsefulLinks` 并填充了一些数据。在这个挑战的下一部分中，我们将创建一个视图，将数据模型与我们将展示给用户的模板连接起来。

但首先，让我们看一下 example_app 的现有代码。

## 现有代码示例

由于 example_app 是为了提供应用开发的示例，`views.py` 文件中有大量注释来帮助我们理解代码。

同样，如果你有 GitHub Copilot，我发现它在解释代码方面做得相当不错。例如，我高亮了 `ExampleAppConfigView` 并让它解释：

![copilot_1](images/copilot_1.png)

有些视图可能很复杂，带有额外的上下文、动作、重写等。但其他视图可能很简单，比如 `CircuitDetailAppTabView` 和 `DeviceDetailAppTabOneView`，它们只是提供一个对象列表并将其传递给响应，然后指向特定的 HTML 模板：

```python views.py
class CircuitDetailAppTabView(views.ObjectView):
    """
    此视图的模板扩展了电路详情模板，
    适合作为电路详情页面的标签页显示。

    用作对象详情标签页内容渲染的视图必须
    始终继承自 nautobot.apps.views.ObjectView。
    """

    queryset = Circuit.objects.all()
    template_name = "example_app/tab_circuit_detail.html"


class DeviceDetailAppTabOneView(views.ObjectView):
    """
    此视图的模板扩展了设备详情模板，
    适合作为设备详情页面的标签页显示。

    用作对象详情标签页内容渲染的视图必须
    始终继承自 nautobot.apps.views.ObjectView。
    """

    queryset = Device.objects.all()
    template_name = "example_app/tab_device_detail_1.html"
```

我有一种感觉，对于我们简单的 `UsefulLinks` 表格，不需要任何复杂的东西。

再次查看 `Example Nautobot App` 页面的 `Views/URLs` 部分：

![views_url_1](images/views_url_1.png)

然后尝试一些 URL，并尝试关联视图、模型和 HTML 模板：

![views_url_2](images/views_url_2.png)

正如俗话说"量两次，切一次"，我们花在研究当前代码库上的时间会在建立理解方面带来长期收益。

明天回来，我们将创建我们自己的视图！

## 第 53 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布今天挑战中发现的不在我们提及范围内的 URL/视图/模板的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将为我们的数据模型创建一个视图。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+53+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 53 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）