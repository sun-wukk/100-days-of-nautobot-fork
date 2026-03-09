# 修改 Job 日志保留策略

在今天的挑战中，我们有两个目标：

1. 探索 `JobLogEntry` 数据模型，以便操作 Job 日志条目。
2. 建立一套处理 Nautobot Job 与数据库模型交互的工作模式。

换言之，今天的挑战将手把手带您完成以下步骤。

> [!TIP]
> 日志保留策略也可以在配置文件中设置，如有兴趣请参阅 [NAUTOBOT_CHANGELOG_RETENTION](https://docs.nautobot.com/projects/core/en/stable/user-guide/administration/configuration/settings/#changelog_retention)。

让我们开始吧。

## 环境配置

环境配置与 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md) 相同，如需详细步骤请参阅该指南。

## 数据模型探索

我们知道 Job 结果日志条目存储在数据库中，但可能不清楚需要操作哪个数据库模型。可以使用 `nbshell` 来探索数据模型：

```
@ericchou1 ➜ ~ $ cd nautobot-docker-compose/
@ericchou1 ➜ ~/nautobot-docker-compose (main) $ poetry shell
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke nbshell
```

首先导入所有数据模型进行初步探索：

```
>>> from nautobot.extras.models import *
>>> dir()
['AdminGroup', 'Association', 'Avg', 'Cable', 'CablePath', 'Case', 'ChangeLoggedModel', 'ChordCounter', 'Circuit', 'CircuitTermination', 'CircuitType', 'ClockedSchedule', 'CloudAccount', 'CloudNetwork', 'CloudNetworkPrefixAssignment', 'CloudResourceType', 'CloudService', 'CloudServiceNetworkAssignment', 'Cluster', 'ClusterGroup', 'ClusterType', 'Code', 'ComputedField', 'ConfigContext', 'ConfigContextModel', 'ConfigContextSchema', 'ConsolePort', 'ConsolePortTemplate', 'ConsoleServerPort', 'ConsoleServerPortTemplate', 'Constance', 'Contact', 'ContactAssociation', 'ContactMixin', 'ContentType', 'Controller', 'ControllerManagedDeviceGroup', 'Count', 'CrontabSchedule', 'CustomField', 'CustomFieldChoice', 'CustomFieldModel', 'CustomLink', 'Device', 'DeviceBay', 'DeviceBayTemplate', 'DeviceFamily', 'DeviceRedundancyGroup', 'DeviceType', 'DeviceTypeToSoftwareImageFile', 'DynamicGroup', 'DynamicGroupMembership', 'DynamicGroupMixin', 'DynamicGroupsModelMixin', 'Exists', 'ExportTemplate', 'ExternalIntegration', 'F', 'FileAttachment', 'FileProxy', 'FrontPort', 'FrontPortTemplate', 'GitRepository', 'GraphQLQuery', 'Group', 'GroupResult', 'HealthCheckTestModel', 'IPAddress', 'IPAddressToInterface', 'ImageAttachment', 'Interface', 'InterfaceRedundancyGroup', 'InterfaceRedundancyGroupAssociation', 'InterfaceTemplate', 'IntervalSchedule', 'InventoryItem', 'Job', 'JobButton', 'JobHook', 'JobLogEntry', 'JobResult', 'Location', 'LocationType', 'LogEntry', 'Manufacturer', 'Max', 'MetadataChoice', 'MetadataType', 'Min', 'Module', 'ModuleBay', 'ModuleBayTemplate', 'ModuleType', 'Namespace', 'Nonce', 'Note', 'ObjectChange', 'ObjectMetadata', 'ObjectPermission', 'OuterRef', 'Partial', 'PeriodicTask', 'PeriodicTasks', 'Permission', 'Platform', 'PowerFeed', 'PowerOutlet', 'PowerOutletTemplate', 'PowerPanel', 'PowerPort', 'PowerPortTemplate', 'Prefetch', 'Prefix', 'PrefixLocationAssignment', 'Profile', 'Provider', 'ProviderNetwork', 'Q', 'RIR', 'Rack', 'RackGroup', 'RackReservation', 'RearPort', 'RearPortTemplate', 'Relationship', 'RelationshipAssociation', 'RelationshipModel', 'Request', 'Response', 'Role', 'RoleField', 'RouteTarget', 'SQLQuery', 'SavedView', 'SavedViewMixin', 'ScheduledJob', 'ScheduledJobs', 'Secret', 'SecretsGroup', 'SecretsGroupAssociation', 'Service', 'Session', 'SoftwareImageFile', 'SoftwareVersion', 'SolarSchedule', 'StaticGroupAssociation', 'Status', 'StatusField', 'StatusModel', 'Subquery', 'Sum', 'Tag', 'TaggedItem', 'TaskResult', 'Team', 'Tenant', 'TenantGroup', 'Token', 'User', 'UserSavedViewAssociation', 'UserSocialAuth', 'VLAN', 'VLANGroup', 'VLANLocationAssignment', 'VMInterface', 'VRF', 'VRFDeviceAssignment', 'VRFPrefixAssignment', 'VirtualChassis', 'VirtualMachine', 'Webhook', 'When', '_', '__builtins__', 'cache', 'deleted_count', 'get_user_model', 'info_log_entries', 'log', 'reverse', 'settings', 'timezone', 'transaction']
```

从模型列表中，`Job` 和 `JobLogEntry` 看起来很有价值。让我们进一步了解：

```
>>> from nautobot.extras.models import Job, JobLogEntry
>>> log = JobLogEntry.objects.first()

# 在句点后按 Tab 键查看可用选项
>>> log.
log.DoesNotExist(                      log.full_clean(                        log.message
log.Meta(                              log.get_absolute_url(                  log.natural_key(
log.MultipleObjectsReturned(           log.get_constraints(                   log.natural_key_args_to_kwargs(
log.absolute_url                       log.get_deferred_fields(               log.natural_key_field_lookups
log.adelete(                           log.get_log_level_display(             log.natural_slug
log.arefresh_from_db(                  log.get_next_by_created(               log.objects
log.asave(                             log.get_previous_by_created(           log.pk
log.associated_object_metadata(        log.grouping                           log.prepare_database_save(
log.check(                             log.id                                 log.present_in_database
log.clean(                             log.is_cloud_resource_type_model       log.refresh_from_db(
log.clean_fields(                      log.is_contact_associable_model        log.save(
log.composite_key                      log.is_dynamic_group_associable_model  log.save_base(
log.created                            log.is_metadata_associable_model       log.serializable_value(
log.csv_natural_key_field_lookups(     log.is_saved_view_model                log.unique_error_message(
log.date_error_message(                log.job_result                         log.validate_constraints(
log.delete(                            log.job_result_id                      log.validate_unique(
log.documentation_static_path          log.log_level                          log.validated_save(
log.from_db(                           log.log_object                         
>>> 

>>> dir(log)
['DoesNotExist', 'Meta', 'MultipleObjectsReturned', '__class__', ...]
```

可以筛选 `info` 级别的日志条目（您的输出内容会有所不同）：

```
>>> info_log_entries = JobLogEntry.objects.filter(log_level="info")

>>> info_log_entries
<RestrictedQuerySet [<JobLogEntry: Running job>, <JobLogEntry: Checking device hostname compliance: bos-acc-01.infra.valuemart.com>, <JobLogEntry: bos-acc-01.infra.valuemart.com configured hostname is correct.>, <JobLogEntry: Checking device hostname compliance: bos-rtr-01.infra.valuemart.com>, <JobLogEntry: bos-rtr-01.infra.valuemart.com configured hostname is correct.>, <JobLogEntry: Job completed>, <JobLogEntry: Running job>, <JobLogEntry: Job completed>, <JobLogEntry: Running job>, <JobLogEntry: This is an log of info type.>, <JobLogEntry: Running job>, <JobLogEntry: This is an log of info type.>, <JobLogEntry: Running job>, <JobLogEntry: Running job>]>
>>>

>>> info_log_entries.count()
14
```

可以看到，目前有 14 条与 `info` 级别匹配的 `JobLogEntries`。让我们尝试删除它们：

```
>>> deleted_count, _ = info_log_entries.delete()
>>> deleted_count
14
>>> info_log_entries = JobLogEntry.objects.filter(log_level="info")
>>> info_log_entries.count()
0
```

现在我们已经掌握了构建新 Job 所需的一切，让它自动完成这些操作。

## 创建 Job

我们可以将在 `nbshell` 中实验的命令整合为一个名为 `delete_info_log.py` 的新 Job。该脚本没有引入新内容，只是将我们在 `nbshell` 中执行的命令代码化：

```python
from nautobot.apps.jobs import Job, register_jobs
from nautobot.extras.models import JobLogEntry


class DeleteInfoLogEntries(Job):
    class Meta:
        name = "Delete Information Level Log Entries"
        description = "A job to delete all information level log entries."

    def run(self):
        # 筛选 info 级别的日志条目
        info_log_entries = JobLogEntry.objects.filter(log_level="info")

        # 记录待删除条目的数量
        info_entries_count = info_log_entries.count()
        self.logger.debug(f"Found {info_entries_count} information level log entries to delete.")

        # 删除筛选到的日志条目
        deleted_count, _ = info_log_entries.delete()

        # 记录删除结果
        self.logger.debug(f"Deleted {deleted_count} information level log entries.")


register_jobs(
    DeleteInfoLogEntries,
)
```

启用 Job 后，可以运行它来删除其他 `info` 级别的 Job 日志条目：

![jog_log_delete_1](images/jog_log_delete_1.png)

值得一提的是，Nautobot 内置了一个名为 `Logs Cleanup` 的系统 Job，如果只需要进行基础清理，可以直接使用：

![log_cleanup_job_1](images/log_cleanup_job_1.png)

## 与数据模型交互的通用步骤

让我们总结今天挑战中与 Nautobot 数据库模型交互的步骤：

1. 找到需要操作的数据模型。
2. 对模型条目进行实验和探索。
3. 将操作步骤代码化为脚本或 Job。

恭喜完成第 028 天的挑战！

## 第 28 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布新的"删除 info 级别日志"Job 成功执行的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将了解 Nautobot Jobs 的保留属性名称。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+28+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 28 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
