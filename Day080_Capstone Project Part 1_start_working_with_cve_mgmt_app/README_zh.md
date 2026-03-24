# 顶点项目第一部分。概述：Nautobot 软件 CVE 管理应用

欢迎来到 100 天 Nautobot 之旅的最后阶段！在接下来的十天（第 80-92 天），我们将把到目前为止所学到的一切整合起来，创建一个顶点项目：Nautobot 软件 CVE 管理应用。我们将引导你完成整个过程。到本阶段结束时，你将拥有一个功能完整的 Nautobot 应用来展示你的创造力和技术实力。让我们开始这段旅程，一起构建一些令人惊叹的东西！

## 应用的目的

**注意：** 此应用仅供**演示和学习目的**使用。如需全面的解决方案，请考虑使用官方的 [Nautobot 设备生命周期管理应用](https://docs.nautobot.com/projects/device-lifecycle/en/latest/)，它提供了更强大的 CVE 跟踪和软件生命周期管理方法。

**Nautobot 软件 CVE 管理应用**旨在通过将软件常见漏洞和披露（**CVEs**）直接集成到 Nautobot 中来增强安全意识。此应用提供对存储在 Nautobot 中的软件已知漏洞的可见性，帮助团队了解其网络环境中的安全风险。

## 环境设置

对于第 80-92 天的顶点项目，我们将使用我们一直使用的 [场景 2](../Lab_Setup/scenario_2_setup/README.md) 实验室和 Codespace。

正如我们在 [第 42 天](https://github.com/nautobot/100-days-of-nautobot/tree/main/Day042_Baking_an_App_Cookie) 中看到的，当我们"烘焙一个应用 cookie"时，我们将有一个完全功能化、自包含的 Nautobot 应用开发环境。

### **关键目标：**

- 自动检索并关联**已发布的 CVE** 与存储在 Nautobot 中的软件记录。
- 实现软件漏洞的**高效跟踪**。
- 利用 Nautobot 现有的数据模型以避免初期的不必要的复杂性。

## 开发方法

### **步骤 1：烘焙 Cookie（生成 Nautobot 应用）**

开发此应用的第一步是**烘焙一个 cookie**，这意味着使用官方 Cookiecutter 模板生成一个新的 Nautobot 应用。这建立了一个标准化的项目结构，使应用的开发和维护更容易。

### **步骤 2：探索 Nautobot 开发者 API**

对于初始实现，我们**不会**创建自定义模型。相反，我们将：

- 使用 Nautobot 内置的数据模型来填充 CVE 信息。
- 专注于使用现有 Nautobot 结构**获取、存储和显示 CVE 数据**。
- 确保未来可以无缝地引入增强功能，例如自动化工作流或与外部漏洞数据库的集成。

通过遵循这种方法，我们保持了一个轻量级和高效的解决方案，同时利用了 Nautobot 的核心能力。未来的迭代可能会根据不断发展的需求引入自定义模型和附加功能。

# 使用 Cookiecutter 启动 Nautobot 应用

Cookiecutter 是一个旨在从模板生成项目的工具，结合了预定义结构和用户提供的输入。在这种情况下，我们将使用 **Nautobot Cookiecutter 模板**来高效地创建一个新的 Nautobot 应用。

我们将使用的 Cookiecutter 模板由 Nautobot 提供在：
🔗 [Nautobot Cookiecutter 模板](https://github.com/nautobot/cookiecutter-nautobot-app)

## 为什么要使用 Cookiecutter？

- **标准化结构** – 确保跨 Nautobot 应用的最佳实践和一致性。
- **自动化设置** – 通过生成样板代码减少手动设置时间。
- **可定制** – 允许你根据项目特定需求定制你的应用。

## 它是如何工作的

1. **Cookiecutter 从 GitHub 下载 Nautobot 模板**。
2. **系统将提示你输入信息**，如你的应用名称、作者详细信息和项目元数据。
3. **该工具生成一个新项目**，其中包含结构化目录和必要文件。

通过遵循本指南，你将学习如何使用 Cookiecutter 快速搭建 Nautobot 应用并开始开发。

# 生成 Nautobot 应用

我们需要首先安装 Cookiecutter：

```bash
@ericchou1 ➜ ~ $ python3 -m pip install --user cookiecutter
...
Successfully installed Jinja2-3.1.6 MarkupSafe-3.0.2 arrow-1.3.0 binaryornot-0.4.4 chardet-5.2.0 cookiecutter-2.6.0 python-dateutil-2.9.0.post0 python-slugify-8.0.4 pyyaml-6.0.2 six-1.17.0 text-unidecode-1.3 types-python-dateutil-2.9.0.20241206
```

要使用 Cookiecutter 生成一个新的 Nautobot 应用，请在用户主目录中运行以下命令：

```bash
cookiecutter https://github.com/nautobot/cookiecutter-nautobot-app.git --directory=nautobot-app
```

Cookiecutter 将提示你输入各种配置细节。如果需要，你可以填写你的 GitHub 账户用户名，或者直接按回车接受默认值。这只是用于填充项目模板中的一些空白；它不会将数据发送到 GitHub 或任何类似的地方。

1. **GitHub 用户名（可选）：**
   ```
   codeowner_github_usernames ():
   ```
   你可以输入你的 GitHub 用户名或留空。这是可选的，仅用于生成文件中的代码所有权元数据。如果不需要，请留空。

2. **全名：**
   ```
   full_name (Network to Code, LLC):
   ```
   输入你的名字或你的组织名称。

3. **电子邮件地址：**
   ```
   email (info@networktocode.com):
   ```
   输入你的电子邮件地址。

4. **GitHub 组织：**
   ```
   github_org (nautobot):
   ```
   这是此应用将要发布到的 GitHub 组织，如果你决定稍后发布的话。

5. **应用名称：**
   ```
   app_name (my_app): nautobot_software_cves
   ```
   在这里，我们建议你现在输入 nautobot_software_cves 作为应用名称，或者你可以使用任何你想要的名称。这将被明智地用于自定义应用模板剩余部分的默认值，如下所示：

6. **额外命名参数：**
   ```
   verbose_name (Nautobot Software Cves):
   app_slug (nautobot-software-cves):
   project_slug (nautobot-app-nautobot-software-cves): nautobot-app-software-cves
   repo_url (https://github.com/nautobot/nautobot-app-software-cves):
   base_url (nautobot-software-cves): software-cves
   ```

7. **Nautobot 版本支持：**
   ```
   min_nautobot_version (2.0.0): 2.3.2
   max_nautobot_version (2.9999):
   ```
   指定你的应用将支持的 Nautobot 版本。

8. **项目元数据：**
   ```
   camel_name (NautobotSoftwareCves):
   project_short_description (Nautobot Software Cves):
   ```

9. **模型类（可选）：**
   ```
   model_class_name (NautobotSoftwareCvesExampleModel): None
   ```
   如果你想要生成一个示例模型类，请输入其名称。否则，输入 `None` 跳过此步骤。现在选择 None。

10. **开源许可证：**
    ```
    Select open_source_license
    1 - Apache-2.0
    2 - Not open source
    Choose from [1/2] (1): 1
    ```
    为你的项目选择适当的许可证。

11. **文档 URL：**
    ```
    docs_base_url (https://docs.nautobot.com):
    docs_app_url (https://docs.nautobot.com/projects/nautobot-software-cves/en/latest):
    ```
    这些决定了你应用的文档将在哪里托管。

# 探索 Nautobot 应用文件夹结构

生成 Nautobot 应用后，让我们探索其目录结构并理解每个文件和文件夹的用途。

## 导航到你的应用目录

更改到新创建的应用目录：

```bash
cd nautobot-app-software-cves
```

现在，使用 `tree` 命令查看项目结构：

```bash
tree .
```

## 项目结构概述

```
nautobot-app-software-cves/
├── LICENSE
├── README.md
├── changes/
├── development/
│   ├── Dockerfile
│   ├── creds.example.env
│   ├── development.env
│   ├── development_mysql.env
│   ├── docker-compose.base.yml
│   ├── docker-compose.dev.yml
│   ├── docker-compose.mysql.yml
│   ├── docker-compose.postgres.yml
│   ├── docker-compose.redis.yml
│   ├── nautobot_config.py
│   ├── towncrier_template.j2
├── docs/
│   ├── admin/
│   ├── assets/
│   ├── dev/
│   ├── images/
│   │   └── icon-nautobot-software-cves.png
│   ├── index.md
│   ├── requirements.txt
│   ├── user/
├── invoke.example.yml
├── invoke.mysql.yml
├── mkdocs.yml
├── nautobot_software_cves/
│   ├── __init__.py
│   ├── tests/
│   │   ├── __init__.py
│   │   ├── test_api.py
│   │   ├── test_basic.py
├── pyproject.toml
└── tasks.py
```

### 文件夹和文件描述

#### **顶级文件**
- **LICENSE** – 指定软件许可证（默认为 Apache-2.0）。
- **README.md** – 项目 README 文件的通用模板。
- **changes/** – 用于跟踪项目的更改，通过 [towncrier](https://towncrier.readthedocs.io/en/stable/) 管理。

#### **开发环境（`development/`）**
此目录包含与开发环境相关的文件，主要用于基于 Docker 的设置：
- **Dockerfile** – 定义应用的容器设置。
- **docker-compose.\*.yml** – 针对不同数据库后端的各种 Docker Compose 配置。
- **nautobot_config.py** – 开发特定的 Nautobot 配置（不用于生产）。

#### **文档（`docs/`）**
此目录为编写文档提供了一个起点：
- **admin/** – 安装和管理指南的样板。
- **dev/** – 记录开发过程的模板。
- **user/** – 用户面向的文档模板。
- **index.md** – 应用文档的着陆页。
- **requirements.txt** – 定义渲染文档所需的依赖项。

#### **配置和自动化文件**
- **invoke.example.yml / invoke.mysql.yml** – 用于自动化常见任务的 Invoke 配置文件的示例。
- **mkdocs.yml** – [MkDocs](https://www.mkdocs.org/) 的配置，用于生成项目文档。

#### **应用源代码（`nautobot_software_cves/`）**
- **`__init__.py`** – 将目录标记为 Python 模块。
- **tests/** – 包含示例单元测试（`test_api.py`、`test_basic.py`）。

#### **项目元数据和构建系统**
- **pyproject.toml** – 将应用定义为 Python 包，包括依赖项和构建说明。
- **tasks.py** – 可以使用 [Invoke](https://www.pyinvoke.org/) 执行的预定义任务。

## 总结

此结构为开发、测试和记录你的 Nautobot 应用提供了坚实的基础。虽然某些文件可能会随着时间演变，但理解此布局将帮助你有效地管理和扩展你的应用。

## 完成设置

Cookiecutter 完成项目生成后，按照以下步骤完成设置：

```bash
cd nautobot-app-software-cves
poetry lock
cp development/creds.example.env development/creds.env
cp invoke.example.yml invoke.yaml
```

🔹 在继续之前，**请确保你的 `invoke.yaml` 中的 `nautobot_ver` 与 `pyproject.toml` 中定义的 Nautobot 版本匹配**。

```
(nautobot-software-cves-py3.10) @ericchou1 ➜ ~/nautobot-app-software-cves $ cat pyproject.toml
...
[tool.poetry.dependencies]
python = ">=3.8,<3.13"
# Used for local development
nautobot = "^2.3.2"

(nautobot-software-cves-py3.10) @ericchou1 ➜ ~/nautobot-app-software-cves $ cat invoke.yaml
---
nautobot_software_cves:
  nautobot_ver: "2.3.2"
  python_ver: "3.11"
  # local: false
  # compose_dir: "/full/path/to/nautobot-app-software-cves/development"
...
```

如果它们不匹配，你可以使用以下命令更新 `invoke.yaml` 中的正确版本：

```bash
$ nautobot_version=$(awk -F'"' '/nautobot =/ {print $2}' pyproject.toml | tr -d '\n' | tr -d '*^')
$ sed -i "s/nautobot_ver: \".*\"/nautobot_ver: \"$nautobot_version\"/" invoke.yaml
```

最后，我们可以运行 `poetry` 和 `invoke` 命令来构建和启动 nautobot：

```bash
poetry shell
poetry install
invoke build # 这个步骤需要耐心
invoke makemigrations
invoke debug # 这个步骤也需要耐心
```

让我们保持终端窗口打开，以便我们可以看到不同容器生成的所有未来消息。

关于 CSRF 设置还有一件事我们需要做（就像在第 47 天所做的那样），这是由于在 Codespace 中设置的端口转发。

使用单独的终端窗口附加到 nautobot docker 实例并进行以下更改。不要在生产环境中这样做，我们只会在实验室中这样做：

```
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ docker exec -it -u root nautobot-software-cves-nautobot-1 bash

root@1335a55d0eb1:/source# ls
LICENSE  README.md  changes  development  dist  docs  invoke.example.yml  invoke.mysql.yml  mkdocs.yml  my_awesome_app  poetry.lock  pyproject.toml  tasks.py

root@1335a55d0eb1:/source# cd /opt/nautobot/

root@1335a55d0eb1:/opt/nautobot# apt update
root@1335a55d0eb1:/opt/nautobot# apt install -y vim

root@1335a55d0eb1:/opt/nautobot# vim nautobot_config.py

CSRF_TRUSTED_ORIGINS = ["http://localhost:8080", "https://localhost:8080"]
```
![csrf_trusted_origin](images/CSRF_Trusted_Origin.png)

接下来，我们将在单独的浏览器窗口中使用转发的端口打开 Nautobot。为此，请转到 ```PORTS``` 并点击地球图标。

![forward_port](images/forward_port.png)

> [!注意]
> 使用 admin/admin 作为用户名和密码。

我们现在在调试模式下运行 Nautobot，并打开一个访问 Nautobot UI 的浏览器窗口。

![nautobot](images/nautobot.png)

恭喜完成顶点项目第一部分！

## 第 80 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。我们强烈建议你停止实例，**而不是**删除实例，直到我们在第 89 天完成整个顶点项目，因为这些天的内容是相互构建的。

继续在你选择的社交媒体上发布今天挑战中构建的新应用实例的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将继续进行顶点项目。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+80+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 80 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）