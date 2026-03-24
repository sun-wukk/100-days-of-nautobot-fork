# 了解 Job 数据模型

在前几天的挑战中，即使重启容器后，我们仍然能够查看 Nautobot Jobs 的执行结果。这说明结果数据被持久化存储在了数据模型的永久存储位置中。

Job 数据模型提供了描述 Job 元数据的数据库表示，同时也是存储 Job 执行结果的地方。

[https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/jobs/models/](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/jobs/models/)

在今天的挑战中，我们将快速浏览 Job 数据模型的各个方面。今天的动手实践内容较少，更多的是"知道它在哪里"，以便在需要时能找到更多信息。

## 环境配置

环境配置与 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md) 相同，以下是步骤摘要，如需详细背景说明请参阅该指南。

> [!TIP]
> 如果您停止了 Codespace 环境后重新启动，发现 Docker 守护进程无法正常工作，请按照配置指南中的步骤重建环境。如果已有实例在运行，只需启动 Poetry 环境并执行 `invoke debug` 即可。

按照以下步骤启动 Nautobot：
```
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

今天挑战的环境已配置完毕。

## Job 数据模型

Job 的数据库表示位于 Job 数据模型中。Job 数据模型同时也是 JobResult、ScheduledJob 等其他数据模型的锚点。

> [!TIP]
> 查阅 [Job 数据模型文档](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/jobs/models/) 了解更多信息。

如下所示，我们可以在 UI 中修改 Job 的各项属性，例如名称或分组：

![job_model_1](images/job_model_1.png)

也可以覆盖 Job 的多个属性：

![job_model_2](images/job_model_2.png)

回想一下，我们在 Job 审批挑战中已经尝试过覆盖"需要审批"属性。

另一个有时需要调整的选项是 Job 执行的"时间限制"。我们知道，有时网络设备响应较慢，或者 Job 需要处理大量设备，适当调整时间限制可以作为一种临时解决方案来确保 Job 正常执行。

## Nautobot Shell

我们也可以通过 Nautobot Shell 来查看数据模型：
```
$ invoke nbshell
...
from nautobot.extras.models.jobs import Job, JobButton, JobHook, JobLogEntry, JobResult, ScheduledJob
...
>>> j = Job.objects.all()
>>> j
<JobQuerySet [<Job: Check Serial Numbers>, <Job: Command Runner>, <Job: Verify Hostname Pattern For Existing Locations>, <Job: Thsi is my first JobButton Receiver>, <Job: This is my first JobButton Receiver>, <Job: This is my first Job Hook Receiver>, <Job: Update Serial Number JobButton Receiver.>, <Job: Export Object List>, <Job: Git Repository: Dry-Run>, <Job: Git Repository: Sync>, <Job: Import Objects>, <Job: Logs Cleanup>, <Job: Refresh Dynamic Group Caches>, <Job: Verify Hostname Pattern For New York City>, <Job: HelloWorld>, <Job: Hello World with Approval Required>, <Job: Hello Jobs from Git Repo>, <Job: Bounce Interface ports>]>
>>>
>>> j1 = Job.objects.first()
>>> j1.
Display all 120 possibilities? (y or n)
j1.DoesNotExist(                            j1.get_computed_fields_grouping(            j1.job_task
j1.Meta(                                    j1.get_computed_fields_grouping_advanced(   j1.jobbutton_set(
j1.MultipleObjectsReturned(                 j1.get_computed_fields_grouping_basic(      j1.last_updated
j1.adelete(                                 j1.get_constraints(                         j1.latest_result
j1.approval_required                        j1.get_custom_field_groupings(              j1.module_name
j1.approval_required_override               j1.get_custom_field_groupings_advanced(     j1.name
j1.arefresh_from_db(                        j1.get_custom_field_groupings_basic(        j1.name_override
j1.asave(                                   j1.get_custom_fields(                       j1.natural_key(
j1.associated_contacts(                     j1.get_custom_fields_advanced(              j1.natural_key_args_to_kwargs(
j1.associated_object_metadata(              j1.get_custom_fields_basic(                 j1.natural_key_field_lookups
j1.associations                             j1.get_deferred_fields(                     j1.natural_slug
j1.cf                                       j1.get_dynamic_groups_url(                  j1.notes
j1.check(                                   j1.get_notes_url(                           j1.objects
j1.class_path                               j1.get_relationships(                       j1.pk
j1.clean(                                   j1.get_relationships_data(                  j1.prepare_database_save(
j1.clean_fields(                            j1.get_relationships_data_advanced_fields(  j1.present_in_database
j1.composite_key                            j1.get_relationships_data_basic_fields(     j1.read_only
j1.created                                  j1.git_repository                           j1.refresh_from_db(
j1.csv_natural_key_field_lookups(           j1.grouping                                 j1.required_related_objects_errors(
j1.custom_field_data                        j1.grouping_override                        j1.runnable
j1.date_error_message(                      j1.has_computed_fields(                     j1.save(
j1.delete(                                  j1.has_computed_fields_advanced(            j1.save_base(
j1.description                              j1.has_computed_fields_basic(               j1.scheduled_jobs(
j1.description_first_line                   j1.has_sensitive_variables                  j1.serializable_value(
j1.description_override                     j1.has_sensitive_variables_override         j1.soft_time_limit
j1.destination_for_associations(            j1.hidden                                   j1.soft_time_limit_override
j1.documentation_static_path                j1.hidden_override                          j1.source_for_associations(
j1.dryrun_default                           j1.id                                       j1.static_group_association_set(
j1.dryrun_default_override                  j1.installed                                j1.supports_dryrun
j1.dynamic_groups                           j1.is_cloud_resource_type_model             j1.tagged_items(
j1.dynamic_groups_cached                    j1.is_contact_associable_model              j1.tags
j1.dynamic_groups_list                      j1.is_dynamic_group_associable_model        j1.task_queues
j1.dynamic_groups_list_cached               j1.is_job_button_receiver                   j1.task_queues_override
j1.enabled                                  j1.is_job_hook_receiver                     j1.time_limit
j1.from_db(                                 j1.is_metadata_associable_model             j1.time_limit_override
j1.full_clean(                              j1.is_saved_view_model                      j1.to_objectchange(
j1.get_absolute_url(                        j1.job_class                                j1.unique_error_message(
j1.get_changelog_url(                       j1.job_class_name                           j1.validate_constraints(
j1.get_computed_field(                      j1.job_hooks(                               j1.validate_unique(
j1.get_computed_fields(                     j1.job_results(                             j1.validated_save(
>>>
```

今天的挑战侧重于 Nautobot Job 数据模型的背景知识，而非动手实践。作为初学者，我们应该对修改未直接暴露给我们的核心数据模型保持谨慎。

随着我们对 Nautobot Jobs 的逐渐熟悉，在排查问题时可能需要重新审视这些字段。

## 第 19 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布您对 Job 数据模型执行的任意 `queryset` 截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将了解单一事实来源（SSoT）应用与 DiffSync 库。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+19+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 19 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
