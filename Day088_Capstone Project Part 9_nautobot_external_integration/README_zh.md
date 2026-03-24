# 顶点项目第九部分。第 88 天：在 Nautobot 中为 NIST NVD API 创建外部集成

## 目标

在**第 88 天**，我们将探索如何在 Nautobot 中创建**外部集成**以与[美国国家标准与技术研究院（NIST）国家漏洞数据库（NVD）API](https://nvd.nist.gov/developers/vulnerabilities) 交互。此集成将使 Nautobot 能够获取和利用与各种软件版本相关的漏洞数据。

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

## 理解 Nautobot 中的外部集成

Nautobot 中的**外部集成**充当集中配置，存储与外部系统交互所需的基本详细信息。这包括 URL、HTTP 方法、headers 和其他配置等信息。通过定义这些集成，Nautobot 可以与外部 API 或服务无缝通信。

有关概述，请参阅 [Nautobot 外部集成文档](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/externalintegration/)。

## 实现 NIST NVD API 集成

NIST NVD API 提供对大量漏洞数据的访问。将此 API 集成到 Nautobot 中可以增强漏洞管理和分析。值得注意的是，NVD API 提供无限制访问，无需认证凭证，简化了集成过程。

### 创建集成的步骤

1. **导航到外部集成：**
   - 在 Nautobot UI 中，转到 **EXTENSIBILITY** > **External Integrations**。

2. **添加新的外部集成：**
   - 点击 **Add External Integration**。

3. **配置集成详细信息：**
   - **Name:** `NIST NVD API`
   - **Remote URL:** `https://services.nvd.nist.gov/rest/json/cves/2.0`
   - **Verify SSL:** 启用以确保安全通信。
   - **Secrets Group:** 留空，因为不需要认证。
   - **Timeout:** 设置适当的超时值，例如 `30` 秒。
   - **Headers:** `{"Accept": "application/json","Content-Type": "application/json"}`
   - **HTTP Method:** `GET`
   - **CA file path:** 留空。
   - **Extra Config:** 留空。

4. **保存集成：**
   - 点击 **Create** 存储集成设置。

![nist_external_integration](images/external_integration.png)

### 利用集成

通过配置外部集成，Nautobot 现在可以与 NIST NVD API 交互。这可以在各种 Nautobot 作业或插件中利用，以获取和处理与特定软件版本相关的漏洞数据。

## 结论

通过在 Nautobot 中为 NIST NVD API 设置外部集成，我们为增强的漏洞管理奠定了基础。此集成促进了无缝检索和利用最新漏洞信息，有助于实现更安全、更明智的网络环境。

## 第 88 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。我们强烈建议你停止实例，**而不是**删除实例，直到我们在第 89 天完成整个顶点项目，因为这些天的内容是相互构建的。

继续在你选择的社交媒体上发布今天挑战中构建的新应用实例的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将继续进行顶点项目。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+88+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 88 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）