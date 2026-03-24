# Nautobot 数据库模型第四部分：迁移

我们一直在使用 `makemigrations` 和 `migrate` 命令将我们对数据模型所做的更改传播到数据库。

Django 的迁移框架被设计为灵活且强大的，允许我们随着时间的推移演变数据库 schema 而不丢失数据。

在今天的挑战中，我们将讨论数据库迁移。

## 环境设置

我们将结合使用 [场景 2](../Lab_Setup/scenario_2_setup/README.md) 实验室、[https://demo.nautobot.com/](https://demo.nautobot.com/) 和 [Nautobot 文档](https://docs.nautobot.com/projects/core/en/latest/user-guide/core-data-model/overview/introduction/) 进行今天的挑战。

```
$ cd nautobot
$ poetry shell
$ poetry install
$ invoke build
（这个步骤需要耐心）
$ invoke debug
（这个步骤也需要耐心）
```

## 迁移回滚

我们不需要经常这样做（所以我们希望如此），但如果需要，我们可以选择回滚到较旧的数据库 schema。

首先让我们查看迁移历史，注意所有更改的记录一直追溯到初始迁移，并按表分隔：

```shell
(nautobot-py3.10) @ericchou1 ➜ ~/nautobot (develop) $ docker exec -it nautobot-2-4-nautobot-1 bash

root@c8032ee34216:/source# nautobot-server showmigrations
admin
 [X] 0001_initial
 [X] 0002_logentry_remove_auto_add
 [X] 0003_logentry_add_action_flag_choices
auth
 [X] 0001_initial
...
dcim
 [X] 0001_initial_part_1
 [X] 0001_initial_part_2
...
 [X] 0068_asset
 [X] 0069_maintenanceschedule
...
```

我们可以看到最后两个 `dcim` 迁移是我们为 `asset` 和 `maintenanceschedule` 创建的两个数据库模型：

```shell
 [X] 0067_controllermanageddevicegroup_tenant
 [X] 0068_asset
 [X] 0069_maintenanceschedule
```

让我们回滚更改：

```shell
root@c8032ee34216:/source# nautobot-server migrate dcim 0067_controllermanageddevicegroup_tenant
Operations to perform:
  Target specific migration: 0067_controllermanageddevicegroup_tenant, from dcim
Running migrations:
  Rendering model states... DONE
  Unapplying dcim.0069_maintenanceschedule... OK
  Unapplying dcim.0068_asset... OK
...
```

我们可以验证更改是否确实被回滚了：

```shell
root@c8032ee34216:/source# nautobot-server showmigrations dcim
dcim
 [X] 0001_initial_part_1
 [X] 0001_initial_part_2
...
 [X] 0067_controllermanageddevicegroup_tenant
 [ ] 0068_asset
 [ ] 0069_maintenanceschedule
```

## 重新应用迁移

如果想要重新应用迁移，可以简单地运行：

```shell
root@c8032ee34216:/source# nautobot-server migrate
```

## 最佳实践

1. **经常提交迁移**：每次修改模型后都要创建迁移文件
2. **测试迁移**：在部署前先在测试环境中运行迁移
3. **回滚计划**：始终有一个回滚策略以防万一
4. **保持简洁**：尽量保持迁移文件小而专注

恭喜完成第 64 天！

## 第 64 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布你今天挑战中学到的关于数据库迁移的内容，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将讨论性能和可扩展性。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+64+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 64 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）