# Nautobot UI 组件框架 - 第二部分

我们在 [第 73 天](../Day073_Nautobot_Templates_3_UI_Component_Framework_Part_1/README.md) 中看到了 Nautobot UI 组件框架的基本示例。对于刚接触 Django 的人来说，一个常见的反馈是使用框架从 0 到 1 的初始学习曲线。为了开始，我们必须学习创建应用、视图、URL 路由和一些 HTML，只是为了在浏览器中显示一个简单的 `hello world`。

这种反馈是 100% 有效的，也是绝对正确的，但好消息是从 1 到 2 变得更容易了，因为我们已经提前完成了学习努力。当我们熟悉模式后，事情会变得越来越容易。

这对于 Nautobot UI 组件框架也是如此。最初需要一点学习才能使事情运转，然而，随着我们更多地使用框架，工作变得越来越容易。

在今天的挑战中，我们将继续学习 Nautobot UI 组件框架的第二部分。我们将介绍一些常见的组件及其用法。

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

## 代码示例

在我们的 Codespace 环境中，已经有一个 `ExampleModelUIViewSet` 在 `example_app` 中，可以用作今天剩余讨论的参考：

```python file=views.py

class ExampleModelUIViewSet(views.NautobotUIViewSet):
    bulk_update_form_class = forms.ExampleModelBulkEditForm
    filterset_class = filters.ExampleModelFilterSet
    filterset_form_class = forms.ExampleModelFilterForm
    form_class = forms.ExampleModelForm
    queryset = ExampleModel.objects.all()
    serializer_class = serializers.ExampleModelSerializer
    table_class = tables.ExampleModelTable
    object_detail_content = ui.ObjectDetailContent(
        panels=(
            ui.ObjectFieldsPanel(
                section=ui.SectionChoices.LEFT_HALF,
                weight=100,
                fields="__all__",
            ),
            # 一个从 `get_extra_context()` 动态派生的对象表
            ui.ObjectsTablePanel(
                section=ui.SectionChoices.RIGHT_HALF,
                weight=100,
                context_table_key="dynamic_table",
                max_display_count=3,
            ),
            # 一个具有静态定义列的非对象数据表
            ui.DataTablePanel(
                section=ui.SectionChoices.RIGHT_HALF,
                label="自定义表格 1 - 具有动态数据和硬编码列",
                weight=200,
                context_data_key="data_1",
                columns=["col_1", "col_2", "col_3"],
                column_headers=["列 1", "列 2", "列 3"],
            ),
            # 一个具有动态（渲染时）列的非对象数据表
            ui.DataTablePanel(
                section=ui.SectionChoices.FULL_WIDTH,
                label="自定义表格 2 - 具有动态数据和动态列",
                weight=100,
                context_data_key="data_2",
                context_columns_key="columns_2",
                context_column_headers_key="column_headers_2",
            ),
            ui.TextPanel(
                section=ui.SectionChoices.LEFT_HALF,
                label="带 JSON 的文本面板",
                weight=300,
                context_field="text_panel_content",
                render_as=TextPanel.RenderOptions.JSON,
            ),
            ui.TextPanel(
                section=ui.SectionChoices.LEFT_HALF,
                label="带 YAML 的文本面板",
                weight=300,
                context_field="text_panel_content",
                render_as=TextPanel.RenderOptions.YAML,
            ),
            ui.TextPanel(
                section=ui.SectionChoices.RIGHT_HALF,
                label="带 PRE 标签使用的文本面板",
                weight=300,
                context_field="text_panel_code_content",
                render_as=TextPanel.RenderOptions.CODE,
            ),
        ),
    )

    def get_extra_context(self, request, instance):
        context = super().get_extra_context(request, instance)
        if self.action == "retrieve":
            # 为自定义面板添加动态对象表
            context["dynamic_table"] = CircuitTable(Circuit.objects.restrict(request.user, "view"))
            # 为对象详情视图自定义表格添加非对象数据
            context["data_1"] = [
                # 因为上面定义的 DataTablePanel 指定了 `columns`，col_4 数据不会出现
                {"col_1": "value_1a", "col_2": "value_2", "col_3": "value_3", "col_4": "not shown"},
                # 演示空和缺失列数据被安全/正确处理
                {"col_1": "value_1b", "col_2": None},
            ]
            # 一些额外的任意数据用于渲染
            # 动态为此数据表指定列和列头，而不是在声明时
            context["columns_2"] = ["a", "e", "i", "o", "u"]
            context["column_headers_2"] = ["A", "E", "I", "O", "U"]
            context["data_2"] = [
                {
                    # 列值可以包含适当构造的 HTML
                    "a": format_html('<a href="https://en.wikipedia.org/wiki/{val}">{val}</a>', val="a"),
                    # 不适当构造的 HTML 在渲染时被适当转义
                    "e": '<a href="https://example.org/evil-link/e/">e</a>',
                    # Unicode 被正确处理
                    "i": "ℹ︎",  # noqa:RUF001 - 故意使用的类unicode字母
                    "o": "º",
                    "u": "µ",
                },
                # 如上，不匹配特定 `columns` 条目的数据不会被渲染
                {"a": 97, "b": 98, "c": 99, "e": 101, "i": 105, "o": 111, "u": 17},
                {"a": "0x61", "b": "0x62", "c": "0x63", "e": "0x65", "i": "0x69", "o": "0x6f", "u": "0x75"},
                {
                    "u": 21 + instance.number,
                    "o": 15 + instance.number,
                    "i": 9 + instance.number,
                    "e": 5 + instance.number,
                    "a": 1 + instance.number,
                },
            ]
            # 为 TextPanel 添加数据
            context["text_panel_content"] = {
                "device_name": "Router1",
                "ip_address": "192.168.1.1",
                "subnet_mask": "255.255.255.0",
                "gateway": "192.168.1.254",
                "interfaces": [
                    {
                        "interface_name": "GigabitEthernet0/0",
                        "ip_address": "10.0.0.1",
                        "subnet_mask": "255.255.255.252",
                        "mac_address": "00:1A:2B:3C:4D:5E",
                    },
                ],
            }
            context["text_panel_code_content"] = 'import abc\nabc()\nprint("Hello world!")'

        return context

    @action(detail=False, name="All Names", methods=["get"], url_path="all-names", url_name="all_names")
    def all_names(self, request):
        """
        返回所有示例模型名称的列表。
        """
        all_example_models = self.get_queryset()
        return render(
            request,
            "example_app/examplemodel_custom_action_get_all_example_model_names.html",
            {"data": [model.name for model in all_example_models]},
        )

```

让我们从回顾核心概念开始。

## 核心概念

- **ObjectDetailContent 定义**
  - 为对象详情视图定义 `object_detail_content` 属性。
  - 配置特定数据模型的额外标签、面板和额外按钮。

- **标签（Tabs）**
  - 主要的 UI 构建块，允许不同的页面内容。
  - 支持客户端切换和不同视图渲染。

- **面板（Panels）**
  - 包含特定内容，定位在标签中的部分内。

- **按钮（Buttons）**
  - 添加到 `ObjectDetailContent` 以提供额外功能。

面板是 UI 组件的主要容器，让我们看一些面板类型。

#### 面板类型

- **基础面板（Base Panel）**
  - 作为显示面板的基类。

  ```python
  from nautobot.apps.ui import Panel, SectionChoices

  Panel(
     weight=100,
     section=SectionChoices.FULL_WIDTH,
     label="面板头部",
  )
  ```

- **对象字段面板（ObjectFieldsPanel）**
  - 以表格格式自动渲染对象属性。

  ```python
  from nautobot.apps.ui import ObjectFieldsPanel, SectionChoices

  ObjectFieldsPanel(
     weight=100,
     section=SectionChoices.LEFT_HALF,
     label="对象字段面板",
     context_object_key="obj",
  )
  ```

- **键值表格面板（KeyValueTablePanel）**
  - 以两列表格格式显示数据。

  ```python
  from nautobot.apps.ui import KeyValueTablePanel

  KeyValueTablePanel(
      weight=100,
      data={
          "speed": "1000000",
          "notes": "**重要**"
      },
  )
  ```

- **分组键值表格面板（GroupedKeyValueTablePanel）**
  - 将数据组织成可折叠的组。

  ```python
  from nautobot.apps.ui import GroupedKeyValueTablePanel, SectionChoices

  GroupedKeyValueTablePanel(
      weight=300,
      section=SectionChoices.FULL_WIDTH,
      label="分组信息",
      body_id="network-details",
      data={
          "网络": {
              "VLAN": "100",
              "IP 范围": "192.168.1.0/24"
          },
          "物理": {
              "位置": "机架 A1",
              "高度": "2U"
          },
          "": {
              "备注": "重要信息"
          }
      },
  )
  ```

- **统计面板（StatsPanel）**
  - 显示带有指向过滤视图的可点击链接的统计信息。

  ```python
  from nautobot.apps.ui import StatsPanel, SectionChoices

  StatsPanel(
      weight=700,
      section=SectionChoices.RIGHT_HALF,
      label="统计",
      filter_name="location",
      related_models=[
          Device,
          (Circuit, "circuit_terminations__location__in"),
          (VirtualMachine, "cluster__location__in")
      ],
  )
  ```

- **对象文本面板和文本面板（ObjectTextPanel and TextPanel）**
  - 以各种格式（Markdown、JSON 等）显示对象属性和上下文文本。

  ```python
  from nautobot.apps.ui import ObjectTextPanel, SectionChoices

  ObjectTextPanel(
     weight=500,
     section=SectionChoices.FULL_WIDTH,
     label="描述",
     object_field="description",
     render_as=ObjectTextPanel.RenderOptions.MARKDOWN,
     render_placeholder=True,
  )
  ```

- **数据表格面板（DataTablePanel）**
  - 直接从字典渲染表格数据。

  ```python
  from nautobot.apps.ui import DataTablePanel

  DataTablePanel(
     weight=100,
     context_data_key="data",
     columns=["one", "two", "three"],
     column_headers=["One", "Two", "Three"]
  )
  ```

- **对象表格面板（ObjectsTablePanel）**
  - 渲染 Django 模型对象的表格，具有广泛的定制性。

  ```python
  from nautobot.apps.ui import ObjectsTablePanel, SectionChoices

  ObjectsTablePanel(
      weight=100,
      section=SectionChoices.RIGHT_HALF,
      table_class=ExampleTable,
      table_filter="example_name",
      table_title="示例表格",
  )
  ```

#### 按钮类型

- **按钮（Button）**
  - 对象详情视图中的单个按钮。

  ```python
  from nautobot.apps.ui import Button

  Button(
      weight=100,
      label="检查密钥",
      icon="mdi-test-tube",
      javascript_template_path="extras/secret_check.js",
      attributes={"onClick": "checkSecret()"},
  )
  ```

- **下拉按钮（DropdownButton）**
  - 包含渲染为下拉菜单的其他按钮。

  ```python
  from nautobot.apps.ui import DropdownButton, Button, ButtonColorChoices

  DropdownButton(
      weight=100,
      color=ButtonColorChoices.BLUE,
      label="添加组件",
      icon="mdi-plus-thick",
      required_permissions=["dcim.change_device"],
      children=(
          Button(
              weight=100,
              link_name="dcim:device_consoleports_add",
              label="控制台端口",
              icon="mdi-console",
              required_permissions=["dcim.add_consoleport"],
          ),
          Button(
              weight=200,
              link_name="dcim:device_consoleserverports_add",
              label="控制台服务器端口",
              icon="mdi-console-network-outline",
              required_permissions=["dcim.add_consoleserverport"],
          ),
      ),
  )
  ```

## 最佳实践

1. **面板组织**
   - 使用一致的权重，对相关信息进行分组，考虑屏幕尺寸。

2. **性能**
   - 只选择需要的字段，使用 `select_related` 或 `prefetch_related`。

3. **用户体验**
   - 提供清晰的标签，使用一致的模式，正确处理错误。

4. **维护**
   - 记录转换，更新相关模型列表，使用有意义的 `body_id` 值。

#### 故障排除

- **面板不出现**：验证部分选择，面板权重，数据可用性。
- **性能问题**：审查查询复杂性，优化字段选择，检查数据库索引。
- **布局问题**：验证部分分配，审查权重顺序，检查响应行为。
- **值渲染**：验证转换函数，检查数据类型，确认模板路径。

## 资源

- [迁移到 UI 组件框架](https://docs.nautobot.com/projects/core/en/stable/development/apps/migration/ui-component-framework/)
- [nautobot.apps.ui](https://docs.nautobot.com/projects/core/en/stable/code-reference/nautobot/apps/ui/)

## 第 74 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体平台上发布关于今天挑战中学到的内容的帖子，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将更深入地研究 Nautobot 应用导航菜单。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+74+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 74 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）