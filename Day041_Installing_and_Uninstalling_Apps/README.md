# 安装与卸载 Nautobot Apps

[Nautobot Apps 概览](https://docs.nautobot.com/projects/core/en/stable/apps/)页面是了解一些热门 Nautobot 应用文档的好起点。

另一个查看不断扩展的应用列表的地方是 [Nautobot 应用生态页面](https://networktocode.com/nautobot/nautobot-apps/)。

在今天的挑战中，我们将了解如何从 [PyPI](https://pypi.org/) 安装和卸载现有的 Nautobot 应用。

## 在 demo.nautobot.com 上探索 Apps

[https://demo.nautobot.com/](https://demo.nautobot.com/) 提供的功能之一，是可以在 `Apps -> Installed Apps` 下看到预安装在 Nautobot 实例上的一些热门应用：

![installed_apps_demo](images/installed_apps_demo.png)

这是一种以低风险方式试用一些应用的好方法。在 `Apps -> Apps Marketplace` 下，你可以找到各应用的文档链接和简要描述：

![app_marketplace](images/app_marketplace.png)

但如果你想深入了解这些应用的内部细节，我们需要在自己的 Nautobot 实例上安装它们。

## 环境搭建

我们将使用 `scenario 1` 进行实验，请参阅 [scenario_1_setup](../Lab_Setup/scenario_1_setup/README.md) 复习搭建步骤。

以下是进入 Codespace 后的步骤摘要。如果环境是从之前的实验重启的且已执行过相关步骤，请跳过 `invoke build` 和 `invoke db-import`：

```
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

今天的挑战不需要 Containerlab。

## 安装 Golden Config App

查看我们的实例，目前没有任何已安装的应用：

![installed_apps_1](images/installed_apps_1.png)

我们可以参考 [Nautobot Golden Config 安装指南](https://docs.nautobot.com/projects/golden-config/en/latest/admin/install/#install-guide)文档进行安装。

连接到 nautobot 容器：

```
$ docker exec -it nautobot_docker_compose-nautobot-1 bash
```

使用 `pip install` 安装软件包：

```
nautobot@acbbb4e1fc95:~$ pip install nautobot-golden-config
Defaulting to user installation because normal site-packages is not writeable
Collecting nautobot-golden-config
  Downloading nautobot_golden_config-2.3.0-py3-none-any.whl.metadata (8.9 kB)
Collecting deepdiff!=6.0,!=6.1,<8,>=5.5.0 (from nautobot-golden-config)
  Downloading deepdiff-7.0.1-py3-none-any.whl.metadata (6.8 kB)
...
Successfully installed anyio-4.5.2 contourpy-1.1.1 cycler-0.12.1 deepdiff-7.0.1 django-pivot-1.9.0 fonttools-4.56.0 hier-config-2.3.1 httpcore-1.0.7 httpx-0.27.0 importlib-metadata-4.13.0 kiwisolver-1.4.7 matplotlib-3.7.5 mypy_extensions-1.0.0 nautobot-capacity-metrics-3.1.1 nautobot-golden-config-2.3.0 nautobot-plugin-nornir-2.2.0 nornir-3.4.1 nornir-jinja2-0.2.0 nornir-napalm-0.5.0 nornir-nautobot-3.2.0 nornir-netmiko-1.0.1 nornir-utils-0.2.0 numpy-1.24.4 ordered-set-4.1.0 packaging-23.2 pynautobot-2.4.2 ruamel.yaml-0.18.10 ruamel.yaml.clib-0.2.8 setuptools-75.3.0 types-pyyaml-6.0.12.20241230 urllib3-2.2.3 xmldiff-2.7.0
```

如指南中所述，我们还需要将应用添加到 `nautobot_config.py` 中的 `PLUGIN` 列表和 `PLUGIN_CONFIG` 中。

在 `nautobot-docker-compose` 目录下找到 `nautobot_config.py`：

![nautobot_config_1](images/nautobot_config_1.png)

然后复制粘贴以下配置：

```
...
# 启用已安装的插件。将每个插件的名称添加到列表中。
# PLUGINS = ["nautobot_example_plugin"]
PLUGINS = ["nautobot_plugin_nornir", "nautobot_golden_config"]


# 插件配置设置。这些设置由用户可能已安装的各种插件使用。
# 字典中的每个键是已安装插件的名称，其值是该插件的配置字典。
PLUGINS_CONFIG = {
    "nautobot_example_plugin": {},
    "nautobot_plugin_nornir": {
        "nornir_settings": {
            "credentials": "nautobot_plugin_nornir.plugins.credentials.env_vars.CredentialsEnvVars",
            "runner": {
                "plugin": "threaded",
                "options": {
                    "num_workers": 20,
                },
            },
        },
    },
    "nautobot_golden_config": {
        "per_feature_bar_width": 0.15,
        "per_feature_width": 13,
        "per_feature_height": 4,
        "enable_backup": True,
        "enable_compliance": True,
        "enable_intended": True,
        "enable_sotagg": True,
        "enable_plan": True,
        "enable_deploy": True,
        "enable_postprocessing": False,
        "sot_agg_transposer": None,
        "postprocessing_callables": [],
        "postprocessing_subscribed": [],
        "jinja_env": {
            "undefined": "jinja2.StrictUndefined",
            "trim_blocks": True,
            "lstrip_blocks": False,
        },
        # "default_deploy_status": "Not Approved",
        # "get_custom_compliance": "my.custom_compliance.func"
    },
}
...
```

Nautobot 实例应会自动检测到配置变更并重新启动。

`Golden Config` 现在应出现在导航栏和 `Installed Apps` 中：

![golden_config_1](images/golden_config_1.png)

我们还需要执行 `invoke post-upgrade` 以确保数据库表已正确创建：

```
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke post-upgrade
```

可以展开导航菜单随意点击查看：

![golden_config_2](images/golden_config_2.png)

## 卸载 Golden Config App

通常，在生产环境中卸载应用时需要格外谨慎，因为这涉及数据库回滚。但在我们的沙箱环境中，可以安全地执行这些步骤。

我们可以参考[卸载说明](https://docs.nautobot.com/projects/golden-config/en/latest/admin/uninstall/)，使用 `nautobot-server migrate nautobot_golden_config zero` 和 `pip3 uninstall nautobot-golden-config` 来卸载应用。

同时应删除 `nautobot_config.py` 中之前添加的配置。

首先回滚该应用特定的数据库迁移：

```
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ docker exec -it nautobot_docker_compose-nautobot-1 bash

nautobot@acbbb4e1fc95:~$ nautobot-server migrate nautobot_golden_config zero


22:12:32.910 WARNING nautobot.core.api.routers routers.py                   get_api_root_view() :
  Something has changed an OrderedDefaultRouter's APIRootView attribute to a custom class. Please verify that class GoldenConfigRootView implements appropriate authentication controls.
Operations to perform:
  Unapply all migrations: nautobot_golden_config
Running migrations:
  Rendering model states...
...
```

然后卸载应用：

```
nautobot@acbbb4e1fc95:~$ pip3 uninstall nautobot-golden-config
Found existing installation: nautobot-golden-config 2.3.0
Uninstalling nautobot-golden-config-2.3.0:
  Would remove:
    /opt/nautobot/.local/lib/python3.8/site-packages/nautobot_golden_config-2.3.0.dist-info/*
    /opt/nautobot/.local/lib/python3.8/site-packages/nautobot_golden_config/*
Proceed (Y/n)? y
  Successfully uninstalled nautobot-golden-config-2.3.0
```

就这样，Golden Config 应用已被移除：

![installed_apps_2](images/installed_apps_2.png)

出色地完成了今天的挑战！

## 第 41 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

请在你选择的社交媒体上分享你对安装和卸载 Nautobot 应用的看法，务必使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，这样我们可以分享你的进展！

在明天的挑战中，我们将"烤一块饼干"。不用担心，不需要任何烹饪技能。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+41+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 41 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
