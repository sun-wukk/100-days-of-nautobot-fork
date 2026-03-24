# 顶点项目第八部分。第 87 天：通过 API 从 NVD NIST 数据库检索 CVE 信息

## 目标

在**第 87 天**，我们将探索如何从美国国家标准与技术研究院（NIST）维护的国家漏洞数据库（NVD）检索常见漏洞和披露（CVE）信息。这包括：

1. 理解 **CVE 命名约定**。
2. 理解 **CPE 命名约定**。
3. 学习 NVD API 如何运作。
4. 演示如何检索 Cisco IOSXE 软件版本 17.7.2 的 CVE 数据。

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

## 1. 理解 CVE 命名约定

**CVE** 系统提供了识别和命名网络安全漏洞的标准化方法。每个 CVE 标识符遵循此格式：

```
CVE-YYYY-NNNNN
```

其中：
- `CVE` 表示常见漏洞和披露的前缀。
- `YYYY` 是 CVE ID 被分配或公开的年份。
- `NNNNN` 是唯一序列号。

*示例*：`CVE-2025-12345` 是 2025 年编目的第 12,345 个漏洞。

## 2. 理解 CPE 命名约定

**通用平台枚举（CPE）** 是一种标准化方法，用于描述和识别企业计算资产中存在的应用程序、操作系统和硬件设备的类别。CPE 名称由一个结构化字符串组成，包含多个组件以唯一标识产品。命名语法遵循此模式：

```
cpe:2.3:a:<vendor>:<product>:<version>:<update>:<edition>:<language>:<sw_edition>:<target_sw>:<target_hw>:<other>
```

- `cpe` 表示 CPE 前缀。
- `2.3` 指定 CPE 版本。
- `a` 表示部分，其中 `a` 代表应用程序，`o` 代表操作系统，`h` 代表硬件。
- `<vendor>` 是产品供应商的名称。
- `<product>` 是产品的名称。
- `<version>` 是产品的版本。
- `<update>` 指定更新信息。
- `<edition>` 提供版本信息。
- `<language>` 表示语言。
- `<sw_edition>`、`<target_sw>`、`<target_hw>` 和 `<other>` 提供额外的可选属性。

*示例*：Cisco IOS XE 版本 17.7.2 的 CPE 名称可能如下：

```
cpe:2.3:o:cisco:ios_xe:17.7.2:*:*:*:*:*:*:*
```

此 CPE 名称分解为：
- `cpe:2.3`：CPE 版本 2.3。
- `o`：操作系统。
- `cisco`：供应商。
- `ios_xe`：产品。
- `17.7.2`：版本。
- 其余字段是通配符（`*`），表示任意值。

## 3. NVD API 概述

NVD 提供了一个允许用户查询 CVE 数据的公共 API。主要特性包括：

- **基础 URL**：所有 API 端点都以 `https://services.nvd.nist.gov/rest/json/` 为前缀。
- **速率限制**：API 有速率限制；确保遵守以避免被阻止。
- **数据格式**：响应通常为 JSON 格式。
- **端点**：
  - `/cves/2.0`：检索 CVE 详情。
  - `/cpe/2.3`：检索 CPE 详情。

## 4. 检索 Cisco IOS XE 版本 17.7.2 的 CVE 数据

要查找与 Cisco IOS XE 版本 17.7.2 关联的 CVE：

1. **识别 CPE 名称**：如前所述，CPE 名称是：
   ```
   cpe:2.3:o:cisco:ios_xe:17.7.2:*:*:*:*:*:*:*
   ```

1. **使用 curl 发送 API 请求**：使用 `curl` 或 Postman 等工具向 URL `https://services.nvd.nist.gov/rest/json/cves/2.0` 发送 GET 请求。cpeName 应作为查询字符串参数包含在你的请求中。以下是使用 `curl` 的演示，但你也可以在 Postman 中测试相同的请求：

```bash
curl --location --get 'https://services.nvd.nist.gov/rest/json/cves/2.0' \
--data-urlencode 'cpeName=cpe:2.3:o:cisco:ios_xe:17.7.2:*:*:*:*:*:*:*' \
--header 'Content-Type: application/json' \
--header 'Accept: application/json'
```

1. **使用 python 发送 API 请求**：你也可以使用 Python 的 requests 库从 API 检索 CVE 数据，如下所示。

```python
import requests
import json
url = "https://services.nvd.nist.gov/rest/json/cves/2.0"
params = {"cpeName":"cpe:2.3:o:cisco:ios_xe:17.7.2:*:*:*:*:*:*:*"}
headers = {
  'Content-Type': 'application/json',
  'Accept': 'application/json'
}
response = requests.get(url=url, headers=headers, params=params)
print(response.text)
```

1. **处理响应**：API 将返回一个 JSON 对象，包含与指定 CPE 相关的 CVE 条目。提取相关信息，如 CVE ID、描述和严重等级。

*使用 `curl` 的示例*：

```bash
curl --location --get -s 'https://services.nvd.nist.gov/rest/json/cves/2.0' \
--data-urlencode 'cpeName=cpe:2.3:o:cisco:ios_xe:17.7.2:*:*:*:*:*:*:*' \
--header 'Content-Type: application/json' \
--header 'Accept: application/json' | jq .
```

此命令获取 CVE 数据并使用 `jq` 进行格式化以提高可读性。

*使用 `curl` 获取 id 为 **CVE-2024-20313** 的 CVE 数据*：

```bash
curl --location --get -s 'https://services.nvd.nist.gov/rest/json/cves/2.0' \
--data-urlencode 'cpeName=cpe:2.3:o:cisco:ios_xe:17.7.2:*:*:*:*:*:*:*' \
--header 'Content-Type: application/json' \
--header 'Accept: application/json' | jq '.vulnerabilities[] | select(.cve.id == "CVE-2024-20313")'
```

此命令获取 CVE 数据并使用 `jq` 提取 id 为 **CVE-2024-20313** 的 CVE 的数据。

*使用 `python` 获取 id 为 **CVE-2024-20313** 的 CVE 数据*：

```python
import requests
import json
url = "https://services.nvd.nist.gov/rest/json/cves/2.0"
params = {"cpeName":"cpe:2.3:o:cisco:ios_xe:17.7.2:*:*:*:*:*:*:*"}
headers = {
  'Content-Type': 'application/json',
  'Accept': 'application/json'
}
response = requests.get(url=url, headers=headers, params=params)
vulnerabilities = response.json()['vulnerabilities']
cve_data = next(cve for cve in vulnerabilities if cve['cve']['id']=='CVE-2024-20313')
print(cve_data)
```

## 最终结果

通过遵循这些步骤，你可以使用 NVD API 以编程方式检索和分析特定软件版本的 CVE 数据。此能力对于维护漏洞意识和增强组织内的网络安全措施至关重要。

## 第 87 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。我们强烈建议你停止实例，**而不是**删除实例，直到我们在第 89 天完成整个顶点项目，因为这些天的内容是相互构建的。

继续在你选择的社交媒体上发布今天挑战中构建的新应用实例的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将继续进行顶点项目。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+87+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 87 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）