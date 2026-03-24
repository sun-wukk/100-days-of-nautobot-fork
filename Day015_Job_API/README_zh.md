# 使用 REST API 与 Nautobot Jobs 交互

Nautobot 使用 [Django REST Framework](https://www.django-rest-framework.org/) 提供 Web API，该 API 可用于触发 Nautobot Jobs。

在今天的挑战中，我们将使用 Web API 与 Nautobot Jobs 进行交互。

## 环境配置

环境配置与 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md) 相同，以下是步骤摘要，如需详细背景说明请参阅该指南。

> [!TIP]
> 如果您停止了 Codespace 环境后重新启动，发现 Docker 守护进程无法正常工作，请按照配置指南中的步骤重建环境。

按照以下步骤启动 Nautobot：
```
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

今天的挑战无需启动 Containerlab、Arista vEOS 镜像，也无需创建新文件。

今天挑战的环境已配置完毕。

## API 端点示例

首先需要找到 API 参考文档，链接位于 Nautobot 实例页面底部：

![Job_API_1](images/Job_API_1.png)

可以看到与 Jobs 相关的所有端点：

![Job_API_2](images/Job_API_2.png)

以 `/extras/job-results/` 为例，可以使用 `try it out` 按钮直接在浏览器中测试该端点：

![Job_API_3](images/Job_API_3.png)

找到执行按钮并查看结果：

![Job_API_4](images/Job_API_4.png)

## 轮到你了

现在轮到您了，请尝试使用 `curl` 或 `requests` 调用任意 API 端点。

关于如何获取 Token 并连接 REST API，可以参阅 [Nautobot 用户指南 - REST API](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/rest-api/authentication/#rest-api-authentication) 文档。

### curl
```sh
$ curl --help
Usage: curl [options...] <url>
 -d, --data <data>          HTTP POST data
 -f, --fail                 Fail silently (no output at all) on HTTP errors
 -h, --help <category>      Get help for commands
 -i, --include              Include protocol response headers in the output
 -o, --output <file>        Write to file instead of stdout
 -O, --remote-name          Write output to a file named as the remote file
 -s, --silent               Silent mode
 -T, --upload-file <file>   Transfer local FILE to destination
 -u, --user <user:password> Server user and password
 -A, --user-agent <name>    Send User-Agent <name> to server
 -v, --verbose              Make the operation more talkative
 -V, --version              Show version number and quit

This is not the full help, this menu is stripped into categories.
Use "--help category" to get an overview of all categories.
For all options use the manual or "--help all".
```

### python requests
```sh
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ python
Python 3.10.12 (main, Sep 11 2024, 15:47:36) [GCC 11.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import requests
>>> tokens = requests.get("http://localhost:8080/api/users/tokens", auth=("admin", "admin")).json()["results"]
>>> headers = { "Authorization": f"Token {tokens[0]["key"]}", "Accept": "application/json" }
>>> requests.get("http://localhost:8080/api/extras/job-results/?depth=1", headers=headers).json()
```

熟悉 [pynautobot](https://github.com/nautobot/pynautobot) 的用户也可以使用其 [API 端点类](https://github.com/nautobot/pynautobot?tab=readme-ov-file#jobs)。

发挥创意，期待看到您的成果！

## 第 15 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布 API 执行结果的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将了解如何按固定时间间隔调度 Job。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+15+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 15 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
