# 顶点项目第三部分。CVE 管理 Nautobot 应用 - 第 82 天

## **目标**

尽管我们有一个存储 CVE 数据的自定义字段，但我们需要在 **SoftwareVersion** 详情视图中以更好的方式呈现它们，并带有指向软件供应商网站上实际 CVE 的超链接。

## 环境设置

对于第 80-89 天的顶点项目，我们将使用我们一直使用的 [场景 2](../Lab_Setup/scenario_2_setup/README.md) 实验室和 Codespace。

假设我们建立在之前的进度上，我们需要使用 `poetry shell` 启用虚拟环境，并使用 `invoke debug` 启动环境：

```
@ericchou1 ➜ ~ $ cd nautobot-app-software-cves/
@ericchou1 ➜ ~/nautobot-app-software-cves $ poetry shell
(nautobot-software-cves-py3.10) @ericchou1 ➜ ~/nautobot-app-software-cves $ invoke debug
...
nautobot-1  | Django version 4.2.20, using settings 'nautobot_config'
nautobot-1  | Starting development server at http://0.0.0.0:8080/
nautobot-1  | Quit the server with CONTROL-C.
...
```

## **实施步骤**

1. **创建一个 `template_content.py` 文件**
   - 在 Nautobot 插件中，`template_content.py` 用于自定义对象详情页面的 UI。
   - 它允许使用 `TemplateExtension` 添加额外内容来扩展现有视图。

```
nautobot-app-software-cves/
├── nautobot_software_cves/
│   ├── __init__.py
│   ├── tests/
│   ├── template_content.py
```

2. **创建 `SoftwareVersionTemplateExtension` 类**
   - 这个类将扩展 `TemplateExtension` 以将 CVE 相关内容注入到 SoftwareVersion 详情页面。

```python
"""Module to change object details view."""

from nautobot.apps.ui import TemplateExtension

class SoftwareVersionTemplateExtension(TemplateExtension):
```

3. **定义此模板扩展将要应用的模型**
   - 分配 `model` 属性到 **`dcim.softwareversion`**，确保扩展链接到 SoftwareVersion 对象。

```python
"""Module to change object details view."""

from nautobot.apps.ui import TemplateExtension

class SoftwareVersionTemplateExtension(TemplateExtension):

    model = "dcim.softwareversion"
```

4. **创建一个 `right_page` 方法**
   - 此方法定义显示在 **SoftwareVersion** 详情页面右侧的内容。
   - **SoftwareVersion** 对象可以使用 `self.context["object"]` 访问。
   - CVE 数据使用 ORM 从自定义字段 `cves` 中提取：`software_version.custom_field_data.get("cves", {})`。
   - HTML 输出通过创建一个 HTML 面板来显示 CVE 数据来构建：`<div class="panel panel-default">`。
   - 如果 CVE 存在，则生成带有超链接的无序列表（`<ul>`），`target='_blank'` 确保链接在新标签页中打开。
   - 如果没有找到 CVE，则显示一条消息。

```python
    def right_page(self):
        """Add content on the right side of the view."""
        # 获取作为模板上下文提供的对象；
        # 在这种情况下，是 SoftwareVersion 对象本身。
        software_version = self.context["object"]

        # 从 JSON 自定义字段获取 CVE：
        cves = software_version.custom_field_data.get("cves", {})

        # 构建包含此数据的 HTML
        output = """
            <div class="panel panel-default">
            <div class="panel-heading"><strong>CVEs</strong></div>
            <div class="panel-body">
        """

        # 根据可用数据添加列表条目：
        if cves:
            output += "<ul>"
            for cve_name, cve_data in cves.items():
                output += f"<li><a href='{cve_data['link']}' target='_blank'> {cve_name}</a></li>"
            output += "</ul>"
        else:
            output += "There are no CVEs for this Software Version."

        output += "</div></div>"
        return output
```

5. **注册模板扩展**
   - 注册类，以便 Nautobot 在渲染 **SoftwareVersion** 详情页面时应用模板扩展。

```python
template_extensions = [SoftwareVersionTemplateExtension]
```

### **验证**

- 再次重启 invoke debug 之后。
- 在 Cisco IOS-XE 软件版本 17.7.2 的 **SoftwareVersion** 详情视图中，你应该现在看到 CVE 列在右侧部分的面板中，如下图所示：
![cves_panel](images/cves_panel.png)

## **最终代码**

```python
"""Module to change object details view."""

from nautobot.apps.ui import TemplateExtension

class SoftwareVersionTemplateExtension(TemplateExtension):
    """Add CVE information to the Nautobot Software Version detail view."""

    model = "dcim.softwareversion"

    def right_page(self):
        """Add content on the right side of the view."""
        # 获取作为模板上下文提供的对象；
        # 在这种情况下，是 SoftwareVersion 对象本身。
        software_version = self.context["object"]

        # 从 JSON 自定义字段获取 CVE：
        cves = software_version.custom_field_data.get("cves", {})

        # 构建包含此数据的 HTML
        output = """
            <div class="panel panel-default">
            <div class="panel-heading"><strong>CVEs</strong></div>
            <div class="panel-body">
        """

        # 根据可用数据添加列表条目：
        if cves:
            output += "<ul>"
            for cve_name, cve_data in cves.items():
                output += f"<li><a href='{cve_data['link']}' target='_blank'> {cve_name}</a></li>"
            output += "</ul>"
        else:
            output += "There are no CVEs for this Software Version."

        output += "</div></div>"
        return output


# 注册模板扩展以便 Nautobot 应用它
template_extensions = [SoftwareVersionTemplateExtension]
```

## **结论**

此实现通过以下方式增强了 **SoftwareVersion** 详情页面：
✅ 以易于阅读的格式显示 CVE。
✅ 提供指向官方 CVE 参考的直接超链接。
✅ 使用 `TemplateExtension` 无缝集成到 Nautobot 的 UI 中。

这种方法通过使 CVE 数据更容易访问和视觉结构化来改善用户体验。

## 第 82 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。我们强烈建议你停止实例，**而不是**删除实例，直到我们在第 89 天完成整个顶点项目，因为这些天的内容是相互构建的。

继续在你选择的社交媒体上发布今天挑战中构建的新应用实例的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将继续进行顶点项目。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+82+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 82 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）