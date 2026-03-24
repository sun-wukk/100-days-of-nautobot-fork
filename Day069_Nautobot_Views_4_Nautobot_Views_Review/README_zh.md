# Nautobot 视图回顾

在第 66 天到第 68 天，我们深入研究了 Nautobot 视图的复杂性，主要关注类视图（CBVs）、`NautbotUIVewSet`、Nautobot Generic Views 和 `mixins`。很明显，我们只是触及了这些主题的表面。

在今天的挑战中，我们将对过去几天进行综合回顾。

## 关键概念

1. Django 类视图（CBVs）：CBVs 提供了一种结构化的方式来管理视图，强调代码重用性和利用现有方法的能力。
   - **关键 CBVs**：
     - TemplateView：渲染模板。
     - ListView：显示对象列表。
     - DetailView：显示单个对象的详情。
     - CreateView/UpdateView/DeleteView：管理对象生命周期。

2. Nautobot Generic Views：Nautobot 扩展了 Django CBVs 以满足其特定需求。
   - **常见的 Nautobot Generic Views**：
     - ObjectView/ObjectListView：用于对象的详情和列表视图。
     - ObjectEditView/ObjectDeleteView：用于编辑和删除操作。
     - 批量操作：使用 BulkEditView 和 BulkDeleteView 一次处理多个对象。

3. NautobotUIViewSet：视图集以连贯的方式组织相关视图。
   - **常见的 NautobotUIViewSet**：
     - DetailViewSet：显示单个对象的详细信息。
     - ListViewSet：处理对象列表。
     - EditViewSet：用于创建和编辑对象。
     - DestroyViewSet：管理对象删除。
     - BulkUpdateViewSet/BulkDestroyViewSet：实现批量操作以提高效率。

模型和视图是 Nautobot 应用的两个基础主题。*在今天挑战的剩余时间里，请从资源部分中选择一个你感兴趣的主题（或所有主题）并阅读更多。*

## 资源

- [最佳实践](https://docs.nautobot.com/projects/core/en/stable/development/core/best-practices/)
- [Generic Views](https://docs.nautobot.com/projects/core/en/stable/development/core/generic-views/)
- [NautobotUIViewSet](https://docs.nautobot.com/projects/core/en/stable/development/apps/api/views/nautobotuiviewset/)
- [Nautobot Apps Views 代码参考](https://docs.nautobot.com/projects/core/en/stable/code-reference/nautobot/apps/views/#nautobot.apps.views.BulkCreateView)

## 第 69 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布你从今天挑战中对 Nautobot 应用中不同视图的理解，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将讨论使用 CSS 进行 UI 样式设置。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+69+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 69 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）