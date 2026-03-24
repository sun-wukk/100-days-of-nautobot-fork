# App 打包与发布（上篇）

在典型的应用开发流程中，开发完成后需要一套可靠的分发机制。[第 41 天](../Day041_Installing_and_Uninstalling_Apps/README.md)我们已经见识过，一旦应用上架 PyPI，安装起来有多简单——但如何将应用打包成可以上传到 PyPI 的格式呢？这正是第 44 天（今天）和第 45 天（明天）要解决的问题。

整体脉络如下：

- 第 42 天：用 Cookiecutter 生成 Nautobot App 脚手架。
- 第 43 天：理解 App 结构，着手开发。
- 第 44 天（今天）：将 App 打包成 wheel 文件，可本地使用或上传至 PyPI。
- 第 45 天（明天）：将此 App 安装到另一个 Nautobot 实例。

准备好了吗？我们开始吧。

## 环境搭建

从[第 43 天](../Day042_Baking_an_App_Cookie/README.md)重启 Codespace 实例，然后启动 App 开发环境：

```
@ericchou1 ➜ ~ $ cd outputs/nautobot-app-my-awesome-app/
@ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $ poetry shell
Spawning shell within /home/vscode/.cache/pypoetry/virtualenvs/my-awesome-app-TNUNvfeN-py3.10
@ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $ . /home/vscode/.cache/pypoetry/virtualenvs/my-awesome-app-TNUNvfeN-py3.10/bin/activate
(my-awesome-app-py3.10) @ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $

(my-awesome-app-py3.10) @ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $ invoke debug
Starting  in debug mode...
Running docker compose command "up"
 Container my-awesome-app-redis-1  Created
 Container my-awesome-app-db-1  Created
 Container my-awesome-app-nautobot-1  Created
...
nautobot-1  | Django version 4.2.20, using settings 'nautobot_config'
nautobot-1  | Starting development server at http://0.0.0.0:8080/
nautobot-1  | Quit the server with CONTROL-C.
nautobot-1  | 
```

一切就绪，可以开始打包了。

## 操作示例

`pyproject.toml` 中保存了我们在初始化时填写的项目信息，包括版本号、描述、作者等：

```
@ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $ cat pyproject.toml 
[tool.poetry]
name = "my-awesome-app"
version = "0.1.0"
description = "My Awesome Application"
authors = ["Eric Chou <email@domain.com>"]
license = "Apache-2.0"
readme = "README.md"
homepage = "https://github.com/nautobot/nautobot-app-my-awesome-app"
repository = "https://github.com/nautobot/nautobot-app-my-awesome-app"
documentation = "https://docs.nautobot.com/projects/my-awesome-app/en/latest/"
keywords = ["nautobot", "nautobot-app", "nautobot-plugin"]
classifiers = [
    "Intended Audience :: Developers",
    "Development Status :: 5 - Production/Stable",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.8",
    "Programming Language :: Python :: 3.9",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
]
packages = [
    { include = "my_awesome_app" },
]
include = [
    # Poetry 默认会排除 .gitignore 中列出的文件
    "my_awesome_app/static/my_awesome_app/docs/**/*",
]

[tool.poetry.dependencies]
python = ">=3.8,<3.13"
# 仅用于本地开发
nautobot = "^2.0.0"

[tool.poetry.group.dev.dependencies]
coverage = "*"
django-debug-toolbar = "*"
invoke = "*"
ipython = "*"
pylint = "*"
pylint-django = "*"
pylint-nautobot = "*"
ruff = "0.5.5"
yamllint = "*"
toml = "*"
Markdown = "*"
# 渲染自定义 markdown，用于版本新增/变更/移除说明
markdown-version-annotations = "1.0.1"
# 将文档渲染为 HTML
mkdocs = "1.6.0"
# MkDocs Material 主题
mkdocs-material = "9.5.32"
# 从源码自动生成文档（供 MkDocs 使用）
mkdocstrings = "0.25.2"
mkdocstrings-python = "1.10.8"
mkdocs-autorefs = "1.2.0"
griffe = "1.1.1"
towncrier = "~23.6.0"
to-json-schema = "*"
jsonschema = "*"

[tool.poetry.extras]
all = [
]

[tool.pylint.master]
# 引入 pylint_django 插件，避免对 Django 代码模式产生误报
load-plugins = "pylint_django, pylint_nautobot"
ignore = ".venv"

[tool.pylint.basic]
# 私有方法、test_ 开头的函数及内部 Meta 类无需 docstring
no-docstring-rgx = "^(_|test_|Meta$)"

[tool.pylint.messages_control]
disable = """,
    line-too-long
"""

[tool.pylint.miscellaneous]
# 不将 TODO 标记为错误，允许提交含待办事项的代码
notes = """,
    FIXME,
    XXX,
    """

[tool.pylint-nautobot]
supported_nautobot_versions = [
    "2.0.0"
]

[tool.ruff]
line-length = 120
target-version = "py38"

[tool.ruff.lint]
select = [
    "D",  # pydocstyle
    "F", "E", "W",  # flake8
    "S",  # bandit
    "I",  # isort
]
ignore = [
    # 警告：`one-blank-line-before-class`（D203）与 `no-blank-line-before-class`（D211）互相冲突
    "D203", # 类 docstring 前需要一个空行

    # D212 在 google 约定中默认启用，若 docstring 写法如下会报错：
    # """
    # docstring 内容写在引号后的下一行，而非与引号同行。
    # """
    # 经过讨论，我们认为这是合理的风格选择。
    "D212", # 多行 docstring 摘要应从第一行开始
    "D213", # 多行 docstring 摘要应从第二行开始

    # 当前代码库中会产生大量问题
    "D401", # docstring 首行应使用祈使语气
    "D407", # 节标题后缺少虚线分隔
    "D416", # 节名称以冒号结尾
    "E501", # 行长度超限
]

[tool.ruff.lint.pydocstyle]
convention = "google"

[tool.ruff.lint.per-file-ignores]
"my_awesome_app/migrations/*" = [
    "D",
]
"my_awesome_app/tests/*" = [
    "D",
    "S"
]

[build-system]
requires = ["poetry_core>=1.0.0"]
build-backend = "poetry.core.masonry.api"

[tool.towncrier]
package = "my_awesome_app"
directory = "changes"
filename = "docs/admin/release_notes/version_X.Y.md"
template = "development/towncrier_template.j2"
start_string = "<!-- towncrier release notes start -->"
issue_format = "[#{issue}](https://github.com/nautobot/nautobot-app-my-awesome-app/issues/{issue})"

[[tool.towncrier.type]]
directory = "security"
name = "Security"
showcontent = true

[[tool.towncrier.type]]
directory = "added"
name = "Added"
showcontent = true

[[tool.towncrier.type]]
directory = "changed"
name = "Changed"
showcontent = true

[[tool.towncrier.type]]
directory = "deprecated"
name = "Deprecated"
showcontent = true

[[tool.towncrier.type]]
directory = "removed"
name = "Removed"
showcontent = true

[[tool.towncrier.type]]
directory = "fixed"
name = "Fixed"
showcontent = true

[[tool.towncrier.type]]
directory = "dependencies"
name = "Dependencies"
showcontent = true

[[tool.towncrier.type]]
directory = "documentation"
name = "Documentation"
showcontent = true

[[tool.towncrier.type]]
directory = "housekeeping"
name = "Housekeeping"
showcontent = true
```

执行 `invoke generate-packages` 命令，即可生成 wheel 文件——这是 Python 社区通用的一体化安装包格式：

```
(my-awesome-app-py3.10) @ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $ invoke generate-packages
Running docker compose command "ps --services --filter status=running"
Running docker compose command "exec nautobot poetry build"
Skipping virtualenv creation, as specified in config file.
Building my-awesome-app (0.1.0)
  - Building sdist
  - Built my_awesome_app-0.1.0.tar.gz
  - Building wheel
  - Built my_awesome_app-0.1.0-py3-none-any.whl
```

生成的 `my_awesome_app-0.1.0-py3-none-any.whl` 和 `my_awesome_app-0.1.0.tar.gz` 文件会保存在 `dist/` 目录下：

```
(my-awesome-app-py3.10) @ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $ ls
changes  development  dist  docs  invoke.example.yml  invoke.mysql.yml  LICENSE  mkdocs.yml  my_awesome_app  poetry.lock  pyproject.toml  README.md  tasks.py
(my-awesome-app-py3.10) @ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $ ls dist/
my_awesome_app-0.1.0-py3-none-any.whl  my_awesome_app-0.1.0.tar.gz
(my-awesome-app-py3.10) @ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $
```

右键点击 wheel 文件，将其下载到本机的某个位置。明天我们将把它安装到一个独立的 Nautobot 实例上。

![download_wheel](images/download_wheel.png)

## 第 44 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 Codespace 实例。

请在你选择的社交媒体上发布今天生成的 `dist/` 目录截图，务必使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，这样我们可以分享你的进展！

明天的挑战，我们将正式安装这个包。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+44+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 44 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
