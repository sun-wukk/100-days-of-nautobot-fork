# 实验场景 2

![scenario_2](images/scenario_2.png)

实验场景 2 与场景 1 类似，但有以下区别：

- 为环境预先克隆了一个新的 `nautobot` 仓库，克隆自 [Nautobot v2.5.4b1](https://github.com/nautobot/nautobot/)。
- 启动 `nautobot` 的步骤类似，但构建时间更长：

```
$ cd nautobot
$ poetry shell
$ poetry install
$ invoke build
（这一步需要耐心等待）
$ invoke debug
（这一步也需要耐心等待）
```

- 为了让 Codespace 正常工作，在 `development/dev.env` 中做了以下修改：

```
NAUTOBOT_ALLOWED_HOSTS=".localhost 127.0.0.1 [::1] *"
```

- 类似地，在 `development/nautobot_config.py` 中做了以下修改：

```
CORS_ALLOWED_ORIGINS = ["http://localhost:3000", "http://localhost:8080"]
CSRF_TRUSTED_ORIGINS = ["http://localhost:8080", "https://localhost:8080"]
```

- 开放了多个端口，`nautobot` 仍在 8080 端口：

![port_8080](images/port_8080.png)

- [Django 调试工具栏](https://django-debug-toolbar.readthedocs.io/en/latest/) 已启用：

![django_debug_toolbar](images/django_debug_toolbar.png)
