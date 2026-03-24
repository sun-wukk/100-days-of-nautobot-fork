# 顶点项目第十三部分。第 92 天：重构 CVE 加载器作业以填充新的 CVE 模型

## ✅ 目标

在**第 92 天**，我们将重构我们之前创建的 CVE 加载器作业，以将 CVE 数据存储到我们的**自定义 Nautobot 模型**（`CVE`）中，而不是存储在自定义 JSON 字段中。

到现在，你应该已经实现了 `CVE` 模型并将其集成到 Nautobot UI 中（参见第 91 天）。今天的重点是更新作业以正确使用此模型。

## 🛠️ 为什么要重构作业？

在原始实现中，CVE 存储在 JSON 自定义字段中，如下所示：

```json
{
    "CVE-2023-1234": {
        "cvss_base_score": 9.8,
        "link": "https://example.com/CVE-2023-1234",
        "severity": "Critical"
    }
}
```

现在我们有了用于 CVE 的适当模型，是时候将数据填充到 `CVE` 表中，并充分利用 Nautobot 的关系数据库支持和 UI 集成了。

## 🔄 重构作业代码

主要更改包括：

- 删除将数据添加到 JSON 自定义字段的逻辑。
- 为每个漏洞实例化一个 `CVE` 对象。
- 使用多对多关系将每个 CVE 链接到其关联的 `SoftwareVersion`。

## 📦 更新的作业代码

```python
"""
用于使用 NIST NVD API 管理 Nautobot 中 CVE 的模块。

此模块定义了一个用于将 CVE 加载到 Nautobot 数据库的作业。
"""

import requests

from nautobot.apps.jobs import BooleanVar, Job, ObjectVar, MultiObjectVar, register_jobs
from nautobot.extras.models import ExternalIntegration
from nautobot.dcim.models.devices import SoftwareVersion, SoftwareVersionQuerySet
from nautobot_software_cves.models import CVE

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
        self.logger.setLevel("DEBUG" if debug else "INFO")

        if not softwares:
            softwares = SoftwareVersion.objects.all()

        for software in softwares:
            self.logger.info("从 NIST NVD 数据库加载 CVE", extra={"object": software})

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
                response.raise_for_status()
                data = response.json()
                self.logger.debug(f"NVD 响应: {data}")
                vulnerabilities = data.get('vulnerabilities', [])
            except Exception as e:
                self.logger.error(f"意外错误: {e}")
                vulnerabilities = []

            for cve in vulnerabilities:
                cve_name = cve['cve']['id']
                self.logger.info(f"正在加载 CVE {cve_name}", extra={"object": software})

                cvss_versions = ['cvssMetricV32', 'cvssMetricV31', 'cvssMetricV30']
                cvss_version = next(version for version in cvss_versions if version in cve['cve']['metrics'])
                # software.custom_field_data['cves'][cve_name] = {
                #     "cvss_base_score": cve['cve']['metrics'][cvss_version][0]['cvssData']['baseScore'],
                #     "link": cve['cve']['references'][0]['url'],
                #     "severity": cve['cve']['metrics'][cvss_version][0]['cvssData']['baseSeverity'],
                # }
                # software.validated_save()
                cve_obj, created = CVE.objects.get_or_create(
                    name = cve_name,
                    defaults = {
                        'cvss' : cve['cve']['metrics'][cvss_version][0]['cvssData']['baseScore'],
                        'severity' : cve['cve']['metrics'][cvss_version][0]['cvssData']['baseSeverity'].capitalize(),
                        'link' : cve['cve']['references'][0]['url'],
                    }
                )
                cve_obj.affected_softwares.add(software)


jobs = [LoadCVEsJob]
register_jobs(*jobs)
```

---

## 🧪 验证步骤

实现更改后，让我们验证新模型是否正确填充：

1. 停止 nautobot 并运行 `invoke debug`。
2. 在 Nautobot UI 中导航到 **Jobs**。
3. 在 **CVE Tracking** 下找到 **Load Vulnerabilities** 作业。
   ![job_list](images/job_list.png)
4. 使用你选择的 **NIST NVD 外部集成**运行作业。
   ![run_job](images/run_job.png)
5. 作业完成后：
   - 在 Nautobot UI 中导航到 **CVE TRACKING → CVEs**。
   - 你应该看到已填充的 CVE 列表。
   - 点击任何 CVE 查看其详情（CVSS、严重性、链接、受影响的软件版本）。
  ![list_cves](images/list_cves.png)

## ✅ 结果

你现在已成功：

- 重构了 CVE 加载器作业以写入自定义模型。
- 使用正确的多对多关系将 CVE 链接到软件版本。
- 在 UI 中验证了更改。