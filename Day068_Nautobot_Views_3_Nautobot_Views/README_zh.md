# Nautobot 应用视图

我们在 [第 67 天](../Day067_Nautobot_Views_2_Nautobot_UI_ViewSet/README.md) 中看到了 [NautobotUIViewtSet](https://docs.nautobot.com/projects/core/en/stable/development/apps/api/views/nautobotuiviewset/) 的例子。但*NautobotUIViewSet* 在 Nautobot 中是相对较新引入的（在 Nautobot 1.4 中引入）。

在 `NautobotUIViewSet` 之前，应用视图通常从 [nautobot.apps.views](https://github.com/nautobot/nautobot/blob/develop/nautobot/apps/views.py) 导入 `generic` 视图。

与它们的 `NautobotUIViewSet` 类似，`generic` 视图及其相关的 `mixins` 同样能够处理常见操作，如显示列表、处理表单和执行数据库操作。它们仍然被广泛使用在代码库中，在某些场景下仍然有优势。

> [!提示]
> 我们将使用术语 `generic` 视图来区分它们与 `NautobotUIVewSet`，但它们**不是** Django generic 视图。这些视图来自 `nautobot.core.views`。我们也可以将它们称为 `Nautobot Generic Views`。

在今天的挑战中，我们将深入了解 generic 视图。

## 环境设置

我们将结合使用 [场景 2](../Lab_Setup/scenario_2_setup/README.md) 实验室、[https://demo.nautobot.com/](https://demo.nautobot.com/) 和 [Nautbot 文档](https://docs.nautobot.com/projects/core/en/latest/user-guide/core-data-model/overview/introduction/) 进行今天的挑战。

```
$ cd nautobot
$ poetry shell
$ poetry install
$ invoke build
（这个步骤需要耐心）
$ invoke debug
（这个步骤也需要耐心）
```

### `nautobot.apps.views` 中的关键视图

完整的视图列表可以从 [views.py](https://github.com/nautobot/nautobot/blob/develop/nautobot/apps/views.py) 中获取：

```
from nautobot.core.views.generic import (
    BulkComponentCreateView,
    BulkCreateView,
    BulkDeleteView,
    BulkEditView,
    BulkImportView,  # 3.0 TODO: deprecated, will be removed in 3.0
    BulkRenameView,
    ComponentCreateView,
    GenericView,
    ObjectDeleteView,
    ObjectEditView,
    ObjectImportView,
    ObjectListView,
    ObjectView,
)
from nautobot.core.views.mixins import (
    AdminRequiredMixin,
    ContentTypePermissionRequiredMixin,
    GetReturnURLMixin,
    NautobotViewSetMixin,
    ObjectBulkCreateViewMixin,  # 3.0 TODO: deprecated, will be removed in 3.0
    ObjectBulkDestroyViewMixin,
    ObjectBulkUpdateViewMixin,
    ObjectChangeLogViewMixin,
    ObjectDestroyViewMixin,
    ObjectDetailViewMixin,
    ObjectEditViewMixin,
    ObjectListViewMixin,
    ObjectNotesViewMixin,
    ObjectPermissionRequiredMixin,
)
```

下面列出了一些常见的视图，更多详情请参考 [nautobot.apps.views 代码参考](https://docs.nautobot.com/projects/core/en/stable/code-reference/nautobot/apps/views/)：

1. **ObjectView**：显示单个对象的详细信息。
2. **ObjectListView**：显示对象列表。
3. **ObjectEditView**：处理单个对象的编辑。
4. **ObjectDeleteView**：处理单个对象的删除。
6. **BulkCreateView**：处理多个对象的批量创建。
6. **BulkEditView**：处理多个对象的批量编辑。
7. **BulkDeleteView**：处理多个对象的批量删除。

### 示例：使用 ObjectEditView

让我们尝试追踪其中一个 generic 视图。我们可以查看 [Circuits App views.py](https://github.com/nautobot/nautobot/blob/develop/nautobot/circuits/views.py )：

```python
...
from nautobot.core.views import generic, mixins as view_mixins
...

class CircuitSwapTerminations(generic.ObjectEditView):
    """
    交换电路的 A 和 Z  termination。
    """

    queryset = Circuit.objects.all()

    def get(self, request, *args, **kwargs):
        circuit = get_object_or_404(self.queryset, pk=kwargs["pk"])
        form = ConfirmationForm()

        # Circuit 必须至少有一个 termination 才能交换
        if not circuit.circuit_termination_a and not circuit.circuit_termination_z:
            messages.error(
                request,
                f"No terminations have been defined for circuit {circuit}.",
            )
            return redirect("circuits:circuit", pk=circuit.pk)

        return render(
            request,
            "circuits/circuit_terminations_swap.html",
            {
                "circuit": circuit,
                "circuit_termination_a": circuit.circuit_termination_a,
                "circuit_termination_z": circuit.circuit_termination_z,
                "form": form,
                "panel_class": "default",
                "button_class": "primary",
                "return_url": circuit.get_absolute_url(),
            },
        )

    def post(self, request, *args, **kwargs):
        circuit = get_object_or_404(self.queryset, pk=kwargs["pk"])
        form = ConfirmationForm(request.POST)

        if form.is_valid():
            circuit_termination_a = CircuitTermination.objects.filter(
                circuit=circuit, term_side=CircuitTerminationSideChoices.SIDE_A
            ).first()
            circuit_termination_z = CircuitTermination.objects.filter(
                circuit=circuit, term_side=CircuitTerminationSideChoices.SIDE_Z
            ).first()

            if circuit_termination_a and circuit_termination_z:
                # 使用占位符以避免 (circuit, term_side) 唯一约束上的 IntegrityError
                with transaction.atomic():
                    circuit_termination_a.term_side = "_"
                    circuit_termination_a.save()
                    circuit_termination_z.term_side = "A"
                    circuit_termination_z.save()
                    circuit_termination_a.term_side = "Z"
                    circuit_termination_a.save()
            elif circuit_termination_a:
                circuit_termination_a.term_side = "Z"
                circuit_termination_a.save()
            else:
                circuit_termination_z.term_side = "A"
                circuit_termination_z.save()

            messages.success(request, f"Swapped terminations for circuit {circuit}.")
            return redirect("circuits:circuit", pk=circuit.pk)

        return render(
            request,
            "circuits/circuit_terminations_swap.html",
            {
                "circuit": circuit,
                "circuit_termination_a": circuit.circuit_termination_a,
                "circuit_termination_z": circuit.circuit_termination_z,
                "form": form,
                "panel_class": "default",
                "button_class": "primary",
                "return_url": circuit.get_absolute_url(),
            },
        )
```

由于我们还没有介绍 `form`，目前我们可以忽略一些关于 Django forms 的行。但从代码片段中我们已经可以了解到几点：

1. `CircuitSwapTerminations` 是一个继承自 `generic.ObjectEditView` 的类视图，用于单个对象。
2. `queryset` 获取所有 `Circuit` 对象。
3. 支持两种 HTTP 方法：`get` 和 `post`。
4. 每个方法执行操作并使用 `circuits` 文件夹下的模板渲染响应。

随着我们在未来的挑战中前进，请记住我们介绍的 `views`，并在需要时参考它们。

## 资源

- [Nautobot App 开发者指南 Nautobot Generic Views](https://docs.nautobot.com/projects/core/en/stable/development/apps/api/views/nautobot-generic-views/)
- [Nautobot Core 开发者指南 Generic Views](https://docs.nautobot.com/projects/core/en/stable/development/core/generic-views/)
- [nautobot.apps.views 代码参考](https://docs.nautobot.com/projects/core/en/stable/code-reference/nautobot/apps/views/)

## 第 68 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布今天挑战中某个应用 Nautobot generic 视图的另一个例子，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将回顾我们学到的关于 Nautobot 视图的内容。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+68+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 68 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）