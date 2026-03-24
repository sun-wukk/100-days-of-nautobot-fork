# 第 96 天：Nautobot 测试和持续集成

## 目标

了解 Nautobot 当前测试和持续集成 (CI) 的实践。

请参考 [测试 Nautobot](https://docs.nautobot.com/projects/core/en/stable/development/core/testing/) 了解开发和维护 Nautobot 自动化测试套件的最佳实践。

## 在 Nautobot 中进行测试

测试是软件开发的关键部分。它有助于确保代码按预期工作，并防止引入错误。

Nautobot 遵循某些编写和运行测试的最佳实践。

### 关键组件

1. **单元测试**：测试代码的各个单元以确保它们按预期工作。
2. **集成测试**：测试不同单元代码之间的交互。
3. 通过 Selenium 对 UI/浏览器功能进行测试
4. 数据库迁移测试（升级场景）
5. REST API 和 GraphQL 模式生成和验证测试

### Nautobot 测试

1. **使用 Django 的测试框架**：
   - Nautobot 使用 Django 内置的测试框架。
   - 在每个应用的 `tests` 目录中编写测试。

2. **遵循 Arrange-Act-Assert 模式**：
   - Arrange：设置测试数据和环境。
   - Act：执行被测试的代码。
   - Assert：验证结果。

3. **使用工厂创建测试数据**：
   - 使用 `factory_boy` 等工厂库创建测试数据。
   - 这有助于保持测试的清洁和可维护性。

4. **为不同场景编写测试用例**：
   - 为典型和边缘情况编写测试。
   - 确保覆盖所有可能的场景。

### 示例：单元测试

以下是 Nautobot 中单元测试的示例：

1. **创建测试文件**：
   - 在你应用的 `tests` 目录中创建一个新文件 `test_models.py`。

```python name=tests/test_models.py
from nautobot.core.testing.TestCase import TestCase
from nautobot.dcim.models import Device

class DeviceTestCase(TestCase):
    def setUp(self):
        self.device = Device.objects.create(name="Test Device")

    def test_device_creation(self):
        self.assertEqual(self.device.name, "Test Device")
```

2. **运行测试**：
   - 使用 Nautobot 的 `nautobot-server` 运行测试。

```sh
nautobot-server test <app_name>.<TestCaseClass>.<test_method>
```

## Nautobot 中的持续集成 (CI)

持续集成 (CI) 是一种开发实践，开发人员频繁地将代码集成到共享仓库中。每次集成都通过自动化构建和测试进行验证，以早期检测问题。

Nautobot 使用 GitHub Actions 进行 CI。

![nautobot_actions](images/nautobot_actions.png)

## 资源

- [测试 Nautobot](https://docs.nautobot.com/projects/core/en/stable/development/core/testing/)
- [Django 测试文档](https://docs.djangoproject.com/en/stable/topics/testing/)
- [GitHub Actions 文档](https://docs.github.com/en/actions)
- [Nautobot 文档](https://docs.nautobot.com/)

## 第 96 天待办事项

你的项目进展如何？可以随意发布项目进展的内容。

你也可以在你选择的社交媒体上发布你从今天挑战中学到的关于测试的知识，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将突出介绍社区贡献者及其对新老手的建议。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+jst+completed+Day+96+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 96 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）