# 顶点项目第十部分。第 89 天：通过插件将 CVE 加载器作业集成到 Nautobot

## **目标**

在**第 89 天**，我们将建立在第 88 天与 NIST NVD 的外部集成基础上，并展示如何：

1. **通过插件将作业集成到 Nautobot**。
2. **开发一个作业**，使用 NIST NVD API 检索所选软件版本的 CVE。
3. **运行作业**以获取特定软件版本（`Cisco IOS XE 17.7.2`）的 CVE。
4. **验证结果**在 Nautobot UI 中。

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

## **实施步骤**

### **1. 在 `jobs.py` 中定义作业**

在你的 Nautobot 插件文件夹（`nautobot_software_cves/`）中，创建一个名为 `jobs.py` 的文件。这是我们将定义 CVE 加载器作业的地方。

```python
"""
用于使用 NIST NVD API 管理 Nautobot 中 CVE 的模块。

此模块定义了一个用于将 CVE 加载到 Nautobot 数据库的作业。
"""

import requests

from nautobot.apps.jobs import BooleanVar, Job, ObjectVar, MultiObjectVar, register_jobs
from nautobot.extras.models import ExternalIntegration
from nautobot.dcim.models.devices import SoftwareVersion, SoftwareVersionQuerySet

name = "CVE Tracking"


class LoadCVEsJob(Job):
    nist_external_integration = ObjectVar(
        model=ExternalIntegration,
        label="NIST NVD API",
        description="CVE portal of NIST NVD database 的外部集成",
        required=True,
    )
    softwares = MultiObjectVar(
        model=SoftwareVersion,
        label="Softwares",
        description="为所选软件加载 CVE。",
        required=False
    )
    debug = BooleanVar(description="启用以获取更详细的调试日志")

    class Meta:
        """Device State Diff - Compare Data Job 的 Meta 对象。"""

        name = "Load Vulnerabilities"
        description = "Load Vulnerabilities"
        has_sensitive_variables = False
        hidden = False

    def run(
        self,
        nist_external_integration: ExternalIntegration = None,
        softwares: SoftwareVersionQuerySet = None,
        debug: bool = False,
    ):

        # 设置日志级别
        self.logger.setLevel("DEBUG" if debug else "INFO")

        # 如果用户未选择任何软件，则定义 SoftwareVersion 查询集
        if not softwares:
            softwares = SoftwareVersion.objects.all()

        # 循环遍历所选软件以加载 CVE
        for software in softwares:
            self.logger.info("从 NIST NVD 数据库加载 CVE", extra={"object": software})

            # 从 NIST NVD API 检索软件漏洞
            software_version = software.version
            cpe_name = f"cpe:2.3:o:cisco:ios_xe:{software_version}:*:*:*:*:*:*:*"
            url = nist_external_integration.remote_url
            params = {"cpeName": cpe_name}
            headers = nist_external_integration.headers
            timeout = nist_external_integration.timeout
            http_method = nist_external_integration.http_method
            try:
                response = requests.request(
                    method=http_method,
                    url=url,
                    headers=headers,
                    params=params,
                    timeout=timeout
                )
                # 对 4xx/5xx 响应引发异常
                response.raise_for_status()
                # 解析 JSON 响应并获取漏洞
                data = response.json()
                self.logger.debug(f"NVD 响应: {data}")
                vulnerabilities = data.get('vulnerabilities', [])
            except Exception as e:
                self.logger.error(f"意外错误: {e}")
                vulnerabilities = []

            # 如果自定义字段数据为 None，则初始化
            if not software.custom_field_data['cves']:
                software.custom_field_data['cves'] = {}

            # 循环遍历漏洞并将它们加载到 Nautobot
            for cve in vulnerabilities:
                cve_name = cve['cve']['id']
                self.logger.info(f"正在加载 CVE {cve_name}", extra={"object": software})
                cvss_versions = ['cvssMetricV32', 'cvssMetricV31', 'cvssMetricV30']
                cvss_version = next(version for version in cvss_versions if version in cve['cve']['metrics'])
                software.custom_field_data['cves'][cve_name] = {
                     "cvss_base_score": cve['cve']['metrics'][cvss_version][0]['cvssData']['baseScore'],
                     "link": cve['cve']['references'][0]['url'],
                     "severity": cve['cve']['metrics'][cvss_version][0]['cvssData']['baseSeverity'],
                 }
                software.validated_save()


jobs = [LoadCVEsJob]
register_jobs(*jobs)
```

### **2. 在 Nautobot UI 中运行作业**

一旦作业被定义且你的插件已安装/重新加载：

1. 使用命令 `invoke stop` 停止你的 nautobot 实例
2. 运行 `invoke post-upgrade` 以将新作业注册到 Nautobot。
3. 在 Nautobot UI 中导航到 **Jobs**。
4. 在 **CVE Tracking** 部分下找到该作业。
5. 通过点击左侧的 **Edit** 按钮并选择 `Enabled` 选项来启用作业。
   ![edit_job](images/edit_job.png)
   ![enable_job](images/enable_job.png)
6. 返回作业页面。
6. 选择你在 **第 88 天** 创建的 **`NIST NVD API` 外部集成**。
7. 从列表中选择软件版本 `17.7.2`。
8. 运行作业。
   ![run_job](images/run_job.png)
9. 观察作业结果。
![job_result](images/job_result.png)

### **3. 验证结果**

作业完成后：

- 导航到 **Devices > Software Versions**。
- 找到 **Cisco IOS XE 17.7.2** 的条目。
- 点击它进入 **SoftwareVersion 详情视图**。
- 你现在应该看到 **CVE 自定义字段**已填充了详细信息，例如：
  - **CVE ID**
  - **CVSS 基本分数**
  - **严重性**
  - **参考链接**
- 点击 **CVEs 标签页**并验证为此特定软件版本列出的漏洞。
  ![software_cve](images/software_cves.png)
- 然后，从主菜单导航到 Devices > SOFTWARE > CVE Status 并验证 CVE 也显示在那里
  ![cve_status](images/cve_status.png)

### **✅ 最终结果**

- Nautobot 现在包括一个**通过插件集成的作业**，用于从 NIST NVD API 加载 CVE 数据。
- CVE 使用 Nautobot 的自定义字段直接绑定到特定的**软件版本**。
- 你现在可以定期或手动运行此作业以保持漏洞数据更新！

📌 **专业提示：** 你可以使用 Nautobot 内置的预定作业功能自动执行作业。

## 第 89 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。我们强烈建议你停止实例，**而不是**删除实例，直到我们在第 89 天完成整个顶点项目，因为这些天的内容是相互构建的。

继续在你选择的社交媒体上发布今天挑战中构建的新应用实例的截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将继续进行顶点项目。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+89+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 89 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）