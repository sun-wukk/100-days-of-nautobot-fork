# 第 95 天：Nautobot 治理最佳实践

## 目标

了解 Nautobot 治理的最佳实践。

- 版本控制
- 发布管理
- 依赖管理
- 许可

## Nautobot 中的治理

Nautobot 是一个由 [Network to Code](https://www.networktocode.com) 赞助的基于社区的免费开源软件 (FOSS) 项目。Nautobot 核心团队负责项目提交代码的方向和执行。以下是 Nautobot 治理的一些关键指南和最佳实践：

### 治理结构

1. **核心团队职责**：
   - 核心团队负责 Nautobot 项目的整体方向和执行。
   - 他们审查和合并拉取请求（合并），管理发布，并确保项目的健康和进展。

2. **社区参与**：
   - Nautobot 鼓励社区贡献和参与。
   - 用户可以提交 GitHub 问题以获取功能请求和错误报告。
   - 通过 GitHub 讨论和 Slack 频道欢迎讨论和反馈。

### 项目结构

Nautobot 组件被安排成称为 _apps_ 的功能子部分。每个应用包含与特定功能相关的模型、视图和模板：

- `core` - Nautobot 核心应用功能、基类 和工具。
- `circuits`：通信电路和提供商
- `dcim`：数据中心基础设施管理（位置、机架和设备）
- `extras`：核心数据模型的扩展功能和增强。
- `ipam`：IP 地址管理（VRF、前缀、IP 地址和 VLAN）
- `tenancy`：可以将 Nautobot 对象分配给的租户（如客户）
- `users`：身份验证和用户偏好
- `virtualization`：虚拟机和集群
- `cloud`：云模型适配
- `wireless`：无线模型和适配

### 发布管理

1. **版本控制**：
   - Nautobot 遵循语义版本控制（SemVer）策略，版本格式为 `X.Y.Z`。
   - 主要（`X`）、次要（`Y`）和补丁（`Z`）版本表示变化的性质。

2. **发布计划**：
   - 每年一个主要版本
   - 每年三个次要版本
   - 至少每两周发布一次补丁版本，或根据需要更频繁

3. **维护版本 (LTM)**：
   - 上一主要版本系列的最后一个次要版本被视为维护或长期维护 (LTM) 版本。
   - 在下一个维护版本可用之前积极维护。

4. **弃用策略**：
   - 提供弃用警告和通知以协助过渡。

### 沟通

也请参考 `社区参与` 部分。

1. **Slack**：
   - [Network to Code Slack 上的 `#nautobot`](http://slack.networktocode.com/) 用于快速聊天和讨论。

2. **GitHub**：
   - [GitHub 问题](https://github.com/nautobot/nautobot/issues) 用于功能请求、错误报告和重大更改。
   - [GitHub 讨论](https://github.com/nautobot/nautobot/discussions) 用于一般讨论和支持问题。

### 许可

Nautobot 根据 Apache 2.0 许可证发布，这是一个宽松的开源许可证。这意味着：

- 使用自由：你可以自由使用、修改和分发软件。
- 商业使用：许可证允许你在商业产品中无限制地使用 Nautobot。
- 归属：任何修改后代码的分发必须包括原始版权声明和许可证副本。
- 专利授予：许可证为贡献者向用户提供专利权的明确授予。

- 社区贡献：作为开源项目，Nautobot 依靠社区贡献而蓬勃发展。用户可以参与开发、报告问题和讨论新功能。

- 合规要求：用户必须遵守 Apache 2.0 许可证规定的条款，例如包括对源代码所做任何更改的通知。

## 第 95 天待办事项

你的项目进展如何？请随意发布关于项目进展的内容。

你也可以在你选择的社交媒体上发布你对 Nautobot 治理的想法，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将讨论 Nautobot 测试和持续集成。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+jst+completed+Day+95+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 95 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）