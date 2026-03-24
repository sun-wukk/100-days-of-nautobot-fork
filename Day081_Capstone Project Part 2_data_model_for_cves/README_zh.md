# 顶点项目第二部分。第 81 天：在 Nautobot 中创建 CVE 管理的数据模型

## 概述

在本节中，我们将在 Nautobot 中创建一个**数据模型来存储 CVE 信息**。虽然这可能不是长期 CVE 存储的**最佳**方法，但它为我们 Nautobot 插件提供了一个坚实的基础。

📌 **注意：** 在初始阶段，我们将使用 **Nautobot 内置的自定义字段**来存储 CVE。在后面的部分中，我们将构建更高级的**自定义数据模型**来存储和管理 CVE 数据。

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

## 定义 CVE 数据模型

对于 CVE 存储，我们将使用**字典的字典**结构，如下所示：

```json
{
    {{CVE_ID}}: {
        "cvss_base_score": {{CVE_SCORE}},
        "link": {{CVE_LINK}},
        "severity": {{CVE_SEVERITY}}
    }
}
```

### CVE 数据示例：
```json
{
    "CVE-2025-20169": {
        "cvss_base_score": 7.7,
        "link": "https://sec.cloudapps.cisco.com/security/center/content/CiscoSecurityAdvisory/cisco-sa-snmp-dos-sdxnSUcW",
        "severity": "High"
    },
    "CVE-2025-20172": {
        "cvss_base_score": 7.7,
        "link": "https://sec.cloudapps.cisco.com/security/center/content/CiscoSecurityAdvisory/cisco-sa-snmp-dos-sdxnSUcW",
        "severity": "High"
    }
}
```

## 为 CVE 创建 JSON 自定义字段

要在 Nautobot 中存储 CVE，我们需要创建一个类型为 **JSON** 的**自定义字段**。

### 步骤：
1. 导航到 **Extensibility > Custom Fields**。
2. 点击 **"Add Custom Field"**。
3. 设置以下值：
   - **Label:** `cves`
   - **Key:** `cves`
   - **Type:** `JSON`
   - **Content types**: `dcim|software version`
4. 点击 **Create** 保存字段。

![custom_field_creation_1](images/custom_field_creation_1.png)

![custom_field_creation_2](images/custom_field_creation_2.png)

## 添加平台

1. 导航到 **Devices > Platforms**。
2. 点击 **Add Platform**
3. 将 **Name** 设置为 `Cisco IOS-XE`

## 在 Nautobot 中添加软件版本

为了测试我们的 CVE 存储，我们将**创建一个新的软件版本**并填充 `cves` 字段。

### 步骤：
1. 导航到 **Devices > Software Versions**。
2. 点击 **"Add Software Version"**。
3. 设置以下字段：
   - **Platform:** `Cisco IOS-XE`
   - **Version:** `17.7.2`
   - **Status:** `Active`
4. 将示例 **CVE DATAs** 复制并粘贴到 `cves` 自定义字段中。
5. 点击 **Create** 保存软件版本。
6. 在 Cisco IOS-XE 软件版本 17.7.2 的 SoftwareVersion 详情视图中，你应该现在看到 CVE 列在 JSON 自定义字段中，如下图所示：
![cves_custom_field](images/cves_custom_field.png)

---

## 总结

在这一点上，我们已经：

✔ 创建了一个**JSON 自定义字段**来存储 CVE 数据。
✔ 添加了一个**软件版本**并用示例 CVE 信息填充它。

🔹 **下一步：**
在**第 82 天**，我们将开始开发一个 **Nautobot 插件**来扩展超出自定义字段的功能。

🚀 继续加油——你的 CVE 管理插件正在成型！

## 第 81 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。我们强烈建议你停止实例，**而不是**删除实例，直到我们在第 89 天完成整个顶点项目，因为这些天的内容是相互构建的。

继续在你选择的社交媒体上发布今天挑战中构建的新自定义字段的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将继续进行顶点项目。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+81+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 81 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）