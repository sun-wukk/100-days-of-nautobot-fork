# Nautobot UI 样式

Nautobot 的主要 UI 样式基于 [Bootstrap 3](https://getbootstrap.com/docs/3.4/)，这是最流行的用于开发响应式用户界面的 HTML、CSS 和 JavaScript 框架之一。

但是，从版本 2.1.0 开始，UI 使用了 Nautobot 特定的定制 Bootstrap 主题，存储在 [nautobot-bootstrap GitHub 仓库](https://github.com/nautobot/nautobot-bootstrap/)，然后在主 `nautobot` 仓库中进一步定制。

> [!提示]
> 我们将只介绍与 Nautobot 实现相关的 Bootstrap 3 基础知识。要了解 Bootstrap 能如何用于网站样式，请查看 [Bootstrap Expo](https://expo.getbootstrap.com/)

主题文件是安装并在 HTML 模板中引用的文件。

## 主题文件

- `nautobot/project-static/bootstrap-3.4.1-dist/css/`：基础的 Nautobot 主题 Bootstrap CSS 定义，直接从 `nautobot-bootstrap` 编译而来。这些文件不应手动编辑，只能从 `nautobot-bootstrap` 重新编译并原样复制到 nautobot 中。
- `nautobot/project-static/css/base.css`：对 Nautobot 基础 CSS 主题的覆盖和扩展。可以根据需要编辑。
- `nautobot/project-static/css/dark.css`：专门针对"暗色模式"主题的额外覆盖和扩展。可以根据需要编辑。

让我们查看这些文件并尝试不同的样式。

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

## 主题预览

在我们的开发环境中，我们将 `settings.DEBUG` 设置为 `True`。因此，我们可以导航到 `/theme-preview` 来获取展示许多 Nautobot UI 元素的模板视图。

![theme_preview](images/theme_preview.png)

静态文件位于 `nautobot -> project-static` 文件夹下：

![project_static](images/project_static.png)

让我们做一些无害但可见的更改。

## 示例

只是为了说明，让我们做一些不会改变永久设置的无害更改。

我们可以在 `nautobot -> project-static -> css -> base.css` 下看到 `footer` 设置：

![footer_1](images/footer_1.png)

我们可以右键点击任何 Nautobot 页面，在本例中是 `/theme-preview/`，然后选择 `检查`：

![inspect](images/inspect.png)

找到 `.footer` 部分，我们可以看到与之匹配的代码片段的 `base.css?ve*`。我们可以进行可以立即在页面上查看的更改。在下面的示例中，背景颜色更改为 `红色`，高度添加了 `480px`：

![footer_2](images/footer_2.png)

更改会立即显示在页面上。

一旦我们完成实验，只需关闭检查框并刷新页面。没有永久更改发生，但现在我们知道如果决定进行样式更改需要修改哪些行。

## 资源

`CSS` 中的 `C` 代表 `层叠`，它包括不同的特异性、继承和 [层叠](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_cascade/Cascade)，后者指定了来源和重要性的顺序。许多关于 CSS 的完整书籍已经被编写。

关于 Django 处理静态文件的方式还有额外的复杂性，例如它的 `collectstatic` 命令将不同的静态文件收集到一个位置以提高效率。

在今天的挑战中，我们专注于文件的位置，这样如果我们需要进行样式更改就知道在哪里查找。

要进一步学习，请查看下面的额外资源。每个下面的链接都会引向更多的资源，如果有兴趣的话。

- [Bootstrap UI 文档](https://docs.nautobot.com/projects/core/en/stable/development/core/bootstrap-ui/)
- [Nautobot Bootstrap 仓库](https://github.com/nautobot/nautobot-bootstrap/)
- [Django 静态文件](https://www.w3schools.com/django/django_collect_static_files.php)

恭喜完成第 70 天！

## 第 70 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布今天挑战中你所做更改的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将进一步深入研究 Nautobot 模板。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+70+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 70 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）