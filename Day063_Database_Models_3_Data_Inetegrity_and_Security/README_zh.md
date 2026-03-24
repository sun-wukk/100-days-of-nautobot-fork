# Nautobot 数据库模型第三部分：数据完整性和安全性

数据完整性和安全性是我们今天挑战的两个重点领域。

我们希望确保数据库中的数据保持准确、一致和可靠。在 Nautobot 中，我们可以通过强制字段约束、验证以及模型之间的某些关系来做到这一点。

Nautobot 的安全性涉及保护数据免受未经授权的访问，确保数据的机密性、完整性和可用性。

## 环境设置

对于今天的挑战，我们将使用：

- [场景 2](../Lab_Setup/scenario_2_setup/README.md) 实验室进行动手实践。
- [https://demo.nautobot.com/](https://demo.nautobot.com/) 作为实时演示环境。
- [Nautobot 文档](https://docs.nautobot.com/projects/core/en/latest/user-guide/core-data-model/overview/introduction/) 作为参考指南。

```
$ cd nautobot
$ poetry shell
$ poetry install
$ invoke build
（这个步骤需要耐心）
$ invoke debug
（这个步骤也需要耐心）
```

## 数据完整性

数据完整性确保数据库中的信息保持准确、一致和可靠。以下是 Nautobot 用来维护完整性的关键概念和实践：

### 1. 字段约束

字段约束在单个字段级别强制执行规则以确保有效数据。示例包括：
- `max_length` 用于指定 `CharField` 中的最大字符数。
- `unique=True` 用于必须具有唯一值的字段（例如序列号）。
- `null=True` 用于可以留空的字段。

---

### 2. 模型验证

模型内的自定义验证方法允许你强制执行超出基本字段约束的复杂验证规则。这些方法有助于确保你的数据符合业务逻辑和组织要求。

---

### 3. 数据库关系

数据库关系维护模型之间的引用完整性。示例包括：
- **`ForeignKey`**：将一条记录链接到另一个模型中的另一条记录（例如，将 `Asset` 与 `Device` 关联）。
- **`OneToOneField`**：确保记录之间的唯一一对一关系。
- **`ManyToManyField`**：将一个模型的多条记录链接到另一个模型的多条记录。

`ForeignKey` 中的 `on_delete` 参数确保删除引用对象时相关数据被正确处理。例如：
- `on_delete=models.CASCADE`：删除相关对象和所有相关记录。
- `on_delete=models.SET_NULL`：如果引用对象被删除，则将相关字段设置为 `NULL`。

---

### 示例：具有完整性功能的 Asset 模型

以下是这些概念在实际操作中的示例。回顾 [第 62 天](../Day062_Database_Models_2_Custom_Models/README.md) 的自定义模型，我们创建了以下与数据完整性概念相关的代码：

```python
class Asset(models.Model):
    device = models.ForeignKey(Device, on_delete=models.CASCADE)
    serial_number = models.CharField(max_length=100, unique=True)
    purchase_date = models.DateField()
    warranty_expiration = models.DateField()

    def __str__(self):
        return f"Asset {self.serial_number} for {self.device}"
```

`Asset` 模型强制执行：
- 使用 `ForeignKey` 和 `on_delete=models.CASCADE` 引用 `Device`。
- 每个资产具有唯一的序列号。
- `purchase_date` 和 `warranty_expiration` 使用正确的日期字段。

---

## 安全性

Nautobot 中的数据库安全性专注于保护数据免受未经授权的访问，并确保数据的机密性、完整性和可用性。以下是关键的安全措施：

### 1. 认证和授权

- 使用 Django 内置的身份验证系统来管理用户访问。
- 定义**用户角色**和**权限**来控制谁可以查看或修改数据。
- 利用 Nautobot 的 [对象权限](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/users/objectpermission/) 来限制对敏感字段的访问。

---

### 2. 字段级安全性

在 Nautobot 中可以应用对象级权限来限制对特定字段或对象的访问。这确保敏感数据只能被授权用户访问。

---

### 3. 审计日志

启用审计日志来跟踪对数据库所做的更改。这提供了谁做了更改、更改了什么以及何时发生的清晰记录。审计日志对于以下方面至关重要：
- 监控未经授权的更改。
- 符合监管要求。

---

### 4. 安全配置

许多安全设置在 `nautobot_config.py` 文件中配置。虽然开发环境可能会放宽一些安全设置，但确保生产环境：
- 使用强密码和 API 密钥。
- 强制使用 HTTPS。
- 定义适当的 CORS 和 CSRF 设置。
- 将敏感数据可见性限制为已认证用户。

`nautobot_config.py` 设置示例：
![nautobot_config_settings](images/nautobot_config_settings.png)

---

恭喜完成第 63 天！

## 第 63 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布你今天挑战中关于 Nautobot 数据完整性和安全性的想法，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将讨论数据库迁移。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+63+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 63 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）