# 可搜索模型

在今天的挑战中，我们将研究 Nautobot 中的可搜索模型和相关概念。

"可搜索模型"是什么意思？在主页上，我们看到有两个位置可以搜索数据模型：

![search_1](images/search_1.png)

对于今天的挑战，它与模板无关，但是，它适合整体展示主题以及我们如何向用户呈现数据。

如果我们使用与 [第 71 天](../Day071_Nautobot_Templates_1_Panel_and_Panel_Items/README.md) 相同的方法查看 `urls.py` 和 `views.py`，我们可以看到还有一个我们可以访问的 `SearchView`：

![search_2](images/search_2.png)

让我们运用我们的知识更深入地研究搜索。

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

## 创建位置类型和位置

在 `Organization -> Location Types` 下，让我们创建两个位置类型"数据中心"和"办公室"：

![location_types](images/location_types.png)

在 `Organization -> Locations` 下，我们可以创建两个位置 `BOS1` 和 `NYC1`，对两者都使用 `数据中心` 作为位置类型，状态为 `Active`：

![locations](images/locations.png)

如果我们使用搜索功能搜索 `BOS1`，我们会看到位置的结果：

![search_result_1](images/search_result_1.png)

但是，如果我们搜索"数据中心"或"办公室"，则不会有结果：

![search_result_2](images/search_result_2.png)

为什么会这样？

如果我们使用 `All Objects` 的下拉菜单，我们可以在 `DCIM` 组下看到 `locations` 在搜索列表中，但 `location types` 不在：

![searchable_models](images/searchable_models.png)

我们如何在可搜索模型列表中包含 `location types`？我们可以利用我们目前的知识并在代码中追踪它。

## 视图

首先需要查看相关视图。在我们看到的 `SearchView` 注释部分中有注释：

```python
if form.is_valid():
    # 构建 (app_label, modelname) 元组列表，代表全局搜索中包含的所有模型，
    # 基于每个应用定义的 `app_config.searchable_models` 列表（如果有）
    searchable_models = []
    for app_config in apps.get_app_configs():
        if hasattr(app_config, "searchable_models"):
            searchable_models += [(app_config.label, modelname) for modelname in app_config.searchable_models]
```

![searchable_models_2](images/searchable_models_2.png)

我们知道列表是从每个应用中定义的 `app_config.searchable_models` 构建的。

## 添加到列表

我们在哪里可以找到 `app_config.searchable_models`？我们可以直接在 `core -> apps -> __init__.py` 中添加吗？让我们试试：

![searchable_model_3](images/searchable_model_3.png)

哎呀，它抛出了错误：

![searchable_model_error_1](images/searchable_model_error_1.png)

让我们将其从列表中移除。

更仔细地查看注释，我们记得注释指出 `基于每个应用定义的 app_config.searchable_models 列表（如果有）`。

我们知道模型是在 `DCIM` 应用中定义的，在摸索一番后，我们可以在 `apps.py` 中看到类 `DCIMConfig` 继承自 `NautobotConfig`，它有一个我们可以添加到的 `searchable_models` 列表：

![searchable_models_4](images/searchable_models_4.png)

将 `locationtype` 添加到列表后，开发服务器将重新加载，我们可以看到 `location types` 现在是一个可搜索模型：

![searchable_models_5](images/searchable_models_5.png)

现在如果搜索"数据中心"或"办公室"，将返回结果：

![searchable_models_6](images/searchable_models_6.png)

随着我们进一步深入 nautobot 应用开发旅程，文档将始终不够。我们可能需要开始养成使用现有知识、阅读代码、实验和在公开 Slack 频道提问的习惯。希望这个挑战能让你对这个过程有一些 taste。

恭喜完成第 72 天！

## 第 72 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布今天挑战中新可搜索模型列表的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将研究一个新的 UI 组件框架。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+72+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 72 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）