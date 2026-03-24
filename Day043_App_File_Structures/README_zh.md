# Nautobot App 文件结构详解

今天的挑战，我们将深入探究 [Nautobot App 的结构](https://docs.nautobot.com/projects/core/en/stable/development/apps/api/setup/)。每个 Nautobot App 本质上都是一个独立的 Django 应用，包含数据库模型、URL 路由、HTML 模板和视图逻辑等核心组件。

App 开发者文档中列出了以下目录结构，并对各文件的用途做了简要说明：

```
app_name/
  - app_name/
    - __init__.py           # required
    - admin.py              # Django Admin Interface
    - api/
      - serializers.py      # REST API Model serializers
      - urls.py             # REST API URL patterns
      - views.py            # REST API view sets
    - banner.py             # Banners
    - custom_validators.py  # Custom Validators
    - datasources.py        # Loading Data from a Git Repository
    - filter_extensions.py  # Extending Filters
    - filters.py            # Filtersets for UI, REST API, and GraphQL Model Filtering
    - forms.py              # UI Forms and Filter Forms
    - graphql/
      - types.py            # GraphQL Type Objects
    - homepage.py           # Home Page Content
    - jinja_filters.py      # Jinja Filters
    - jobs.py               # Job classes
    - middleware.py         # Request/response middleware
    - migrations/
      - 0001_initial.py     # Database Models
    - models.py             # Database Models
    - navigation.py         # Navigation Menu Items
    - secrets.py            # Secret Providers
    - signals.py            # Signal Handler Functions
    - table_extensions.py   # Extending Tables
    - template_content.py   # Extending Core Templates
    - templates/
      - app_name/
        - *.html            # UI content templates
    - urls.py               # UI URL Patterns
    - views.py              # UI Views and any view override definitions
  - pyproject.toml          # *** REQUIRED *** - Project package definition
  - README.md
```

对于 Django 新手来说，这份清单初看可能令人望而生畏。好消息是，我们并不需要搞懂每一个文件才能推进开发。比如，如果不需要为 App 构建 REST API，完全可以忽略 `api/` 目录。

今天我们重点介绍几个最关键的文件。

## 环境搭建

从[第 42 天](../Day042_Baking_an_App_Cookie/README.md)重启 Codespace 实例，然后启动 App 开发环境：

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

让我们开始吧。

## 代码示例

我们的 Nautobot App 目录结构如下：

```
(my-awesome-app-py3.10) @ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $ tree .
.
├── changes
├── development
│   ├── app_config_schema.py
│   ├── creds.env
│   ├── creds.example.env
│   ├── development.env
│   ├── development_mysql.env
│   ├── docker-compose.base.yml
│   ├── docker-compose.dev.yml
│   ├── docker-compose.mysql.yml
│   ├── docker-compose.postgres.yml
│   ├── docker-compose.redis.yml
│   ├── Dockerfile
│   ├── nautobot_config.py
│   └── towncrier_template.j2
├── docs
│   ├── admin
│   │   ├── compatibility_matrix.md
│   │   ├── install.md
│   │   ├── release_notes
│   │   │   ├── index.md
│   │   │   └── version_1.0.md
│   │   ├── uninstall.md
│   │   └── upgrade.md
│   ├── assets
│   │   ├── extra.css
│   │   ├── favicon.ico
│   │   ├── nautobot_logo.png
│   │   ├── nautobot_logo.svg
│   │   ├── networktocode_bw.png
│   │   └── overrides
│   │       └── partials
│   │           └── copyright.html
│   ├── dev
│   │   ├── arch_decision.md
│   │   ├── code_reference
│   │   │   ├── api.md
│   │   │   ├── index.md
│   │   │   └── package.md
│   │   ├── contributing.md
│   │   ├── dev_environment.md
│   │   ├── extending.md
│   │   └── release_checklist.md
│   ├── images
│   │   └── icon-my-awesome-app.png
│   ├── index.md
│   ├── requirements.txt
│   └── user
│       ├── app_getting_started.md
│       ├── app_overview.md
│       ├── app_use_cases.md
│       ├── external_interactions.md
│       └── faq.md
├── invoke.example.yml
├── invoke.mysql.yml
├── LICENSE
├── mkdocs.yml
├── my_awesome_app
│   ├── api
│   │   ├── __init__.py
│   │   ├── __pycache__
│   │   │   ├── __init__.cpython-311.pyc
│   │   │   ├── serializers.cpython-311.pyc
│   │   │   ├── urls.cpython-311.pyc
│   │   │   └── views.cpython-311.pyc
│   │   ├── serializers.py
│   │   ├── urls.py
│   │   └── views.py
│   ├── app-config-schema.json
│   ├── filters.py
│   ├── forms.py
│   ├── __init__.py
│   ├── migrations
│   │   ├── 0001_initial.py
│   │   ├── __init__.py
│   │   └── __pycache__
│   │       ├── 0001_initial.cpython-311.pyc
│   │       └── __init__.cpython-311.pyc
│   ├── models.py
│   ├── navigation.py
│   ├── __pycache__
│   │   ├── filters.cpython-311.pyc
│   │   ├── forms.cpython-311.pyc
│   │   ├── __init__.cpython-311.pyc
│   │   ├── models.cpython-311.pyc
│   │   ├── navigation.cpython-311.pyc
│   │   ├── tables.cpython-311.pyc
│   │   ├── urls.cpython-311.pyc
│   │   └── views.cpython-311.pyc
│   ├── tables.py
│   ├── templates
│   │   └── my_awesome_app
│   │       └── myawesomeappexamplemodel_retrieve.html
│   ├── tests
│   │   ├── fixtures.py
│   │   ├── __init__.py
│   │   ├── test_api.py
│   │   ├── test_api_views.py
│   │   ├── test_basic.py
│   │   ├── test_filter_myawesomeappexamplemodel.py
│   │   ├── test_form_myawesomeappexamplemodel.py
│   │   ├── test_model_myawesomeappexamplemodel.py
│   │   └── test_views.py
│   ├── urls.py
│   └── views.py
├── poetry.lock
├── pyproject.toml
├── README.md
└── tasks.py

21 directories, 88 files
```

逐一解读：

- 根目录下的 `changes`、`development`、`docs` 等文件暂时可以忽略，重点关注 `my_awesome_app` 文件夹。
- 在 `my_awesome_app` 目录中，以下文件最值得关注：`models.py`、`navigation.py`、`templates/` 目录、`urls.py` 以及 `views.py`。

```
(my-awesome-app-py3.10) @ericchou1 ➜ ~/outputs/nautobot-app-my-awesome-app $ tree my_awesome_app/
my_awesome_app/
├── api
│   ├── __init__.py
│   ├── __pycache__
│   │   ├── __init__.cpython-311.pyc
│   │   ├── serializers.cpython-311.pyc
│   │   ├── urls.cpython-311.pyc
│   │   └── views.cpython-311.pyc
│   ├── serializers.py
│   ├── urls.py
│   └── views.py
├── app-config-schema.json
├── filters.py
├── forms.py
├── __init__.py
├── migrations
│   ├── 0001_initial.py
│   ├── __init__.py
│   └── __pycache__
│       ├── 0001_initial.cpython-311.pyc
│       └── __init__.cpython-311.pyc
├── models.py
├── navigation.py
├── __pycache__
│   ├── filters.cpython-311.pyc
│   ├── forms.cpython-311.pyc
│   ├── __init__.cpython-311.pyc
│   ├── models.cpython-311.pyc
│   ├── navigation.cpython-311.pyc
│   ├── tables.cpython-311.pyc
│   ├── urls.cpython-311.pyc
│   └── views.cpython-311.pyc
├── tables.py
├── templates
│   └── my_awesome_app
│       └── myawesomeappexamplemodel_retrieve.html
├── tests
│   ├── fixtures.py
│   ├── __init__.py
│   ├── test_api.py
│   ├── test_api_views.py
│   ├── test_basic.py
│   ├── test_filter_myawesomeappexamplemodel.py
│   ├── test_form_myawesomeappexamplemodel.py
│   ├── test_model_myawesomeappexamplemodel.py
│   └── test_views.py
├── urls.py
└── views.py

8 directories, 39 files
```

## 核心文件解析

### `models.py`
定义应用的数据库模型（即数据结构）。模型描述了数据在数据库中的存储方式——每个模型对应一张数据表，模型的每个属性对应表中的一列。

### `navigation.py`
定义应用的导航结构，控制 App 如何融入 Nautobot 的 UI，包括菜单项、链接及各导航元素的配置。

### `templates` 目录
存放用于渲染页面的 HTML 模板文件。模板定义了页面的结构与布局，并结合视图传入的数据动态生成 HTML 内容。

### `urls.py`
定义应用的 URL 路由，将 URL 规则映射到对应的视图，决定用户访问某个地址时执行哪段逻辑。

### `views.py`
定义应用的视图函数或视图类，负责处理请求、执行业务逻辑并返回响应，通常会将数据渲染到模板中呈现给用户。

其余文件的用途简要列举如下。

## 根目录

**changes**：用于追踪变更记录，如版本历史或变更日志。

**development**：开发环境相关配置文件目录。
- **app_config_schema.py**：应用配置的 Schema 定义。
- **creds.env**：凭据相关的环境变量（不提交至版本控制）。
- **creds.example.env**：凭据文件示例，供参考使用。
- **development.env**：开发环境的环境变量。
- **development_mysql.env**：使用 MySQL 时的开发环境变量。
- **docker-compose.base.yml**：Docker Compose 基础配置文件。
- **docker-compose.dev.yml**：开发环境专用的 Docker Compose 配置。
- **docker-compose.mysql.yml**：使用 MySQL 的 Docker Compose 配置。
- **docker-compose.postgres.yml**：使用 PostgreSQL 的 Docker Compose 配置。
- **docker-compose.redis.yml**：使用 Redis 的 Docker Compose 配置。
- **Dockerfile**：用于构建 Docker 镜像的 Dockerfile。
- **nautobot_config.py**：Nautobot 配置文件。
- **towncrier_template.j2**：使用 Towncrier 生成发布说明的 Jinja2 模板。

**docs**：应用文档目录。
- **admin**：运维管理员文档，包含兼容性说明、安装/卸载/升级指南及发布说明。
- **assets**：文档站点的静态资源，包含样式文件、图标和 Logo 等。
- **dev**：开发者文档，包含架构决策记录、代码参考、贡献指南、开发环境搭建说明、扩展指南及发布检查清单。
- **user**：用户文档，包含快速入门、功能概览、使用场景、外部交互说明及常见问题解答。

**invoke.example.yml**：Invoke 任务的示例配置。

**invoke.mysql.yml**：使用 MySQL 时的 Invoke 任务配置。

**LICENSE**：项目开源许可证文件。

**mkdocs.yml**：MkDocs 文档生成工具的配置文件。

**my_awesome_app**：应用主目录。
- **api/**：API 实现，包含序列化器、URL 路由和视图。
- **app-config-schema.json**：应用配置的 JSON Schema。
- **filters.py**：自定义过滤器。
- **forms.py**：Django 表单定义。
- **migrations/**：数据库迁移文件。
- **models.py**：数据库模型定义。
- **navigation.py**：导航菜单配置。
- **tables.py**：自定义表格。
- **templates/**：HTML 模板文件。
- **tests/**：测试用例，涵盖 API、视图、模型、表单、过滤器等各层的测试。
- **urls.py**：URL 路由配置。
- **views.py**：视图函数或视图类定义。

**poetry.lock**：Poetry 依赖管理器的锁定文件，确保依赖版本一致。

**pyproject.toml**：Poetry 及其他工具的项目配置文件。

**README.md**：项目说明文档。

**tasks.py**：Invoke 的任务定义文件。

---

理解每个文件的职责，是着手开发的第一步。恭喜你迈出了这一步！

说来也许出乎意料——我们刚刚生成的 App 已经是一个完整可发布的应用了。接下来的两天，我们将着手了解如何打包和分发它。

## 第 43 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 Codespace 实例。

请在你选择的社交媒体上发布今天查看的 App 目录结构截图，务必使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，这样我们可以分享你的进展！

明天的挑战，我们将正式开启 App 的发布之旅。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+43+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 43 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
