# 示例应用创建新 Job

Nautobot Jobs 是 Nautobot 生态系统的核心力量。它们可以是独立的（就像我们在挑战前两个部分构建的那样），也可以捆绑到 Nautobot 应用中。

在今天的挑战中，我们将在我们一直在开发的 example_app 中放入一个小的"hello world"类型的 job，只是为了展示如何在 Nautobot Apps 中包含 jobs。

## Job 代码

似乎很久以前我们对 Nautobot Jobs 一无所知，但如果我们回想一下最初构建的几个 jobs，我们写了一个简单的 job 来记录一些输出：

```python jobs.py
class TestingAppHelloJobsWithLogs(Job):

    class Meta:
        name = "Testing App Hello Jobs with Logs"
        description = "Testing App Hello Jobs with different log types"

    def run(self):
        self.logger.info("This is an info type log in Testing App.")
        self.logger.debug("This is a debug type log in Testing App.")
        self.logger.warning("This is a warning type log in Testing App.")
        self.logger.error("This is an error type log in Testing App.")
        self.logger.critical("This is a critical type log in Testing App.")
...
jobs = (
    ...
    TestingAppHelloJobsWithLogs,
)
```

让我们找到 `example_app` 目录中的 `jobs.py` 文件并将代码添加到文件末尾，别忘了在 `jobs` 列表中注册新 job：

![jobs_python_file_1](images/jobs_python_file_1.png)

这里是完整的 `jobs.py` 文件：

```python jobs.py
import sys
import time

from django.conf import settings
from django.db import transaction
from django.forms import widgets

from nautobot.apps.jobs import (
    BooleanVar,
    ChoiceVar,
    DryRunVar,
    FileVar,
    IntegerVar,
    IPAddressVar,
    IPAddressWithMaskVar,
    IPNetworkVar,
    Job,
    JobButtonReceiver,
    JobHookReceiver,
    JSONVar,
    MultiChoiceVar,
    MultiObjectVar,
    ObjectVar,
    register_jobs,
    StringVar,
    TextVar,
)
from nautobot.dcim.models import Device, Location, LocationType
from nautobot.extras.choices import ObjectChangeActionChoices
from nautobot.extras.jobs import get_task_logger
from nautobot.extras.models import Status

name = "Example App jobs"  # 包含此文件中定义的所有 Jobs 的"分组"。


class ExampleEverythingJob(Job):
    """一个尽可能展示更多 Job 功能的示例 Job。"""

    class Meta:
        """描述和配置此 Job 的元类属性。"""

        name = "Example Job of Everything"
        description = """\
这是一个尽可能多地展示 Job 特性的 Job。

- 所有魔法方法（before_start、run、on_success、on_failure、after_return）。
- 所有 Meta 属性。
- 所有 ScriptVariable 类型的 Job 输入。
- 等等。"""
        approval_required = True  # 默认值：False
        # 设置为 True 使另一个用户在任何运行此 Job 之前必须批准
        dryrun_default = False  # 默认值：False
        # 设置为 True 使 Job 执行默认进入"dry-run"模式
        field_order = []  # 默认值：未指定（空列表）
        # 输入变量在 UI 中出现的顺序
        # 如果未指定，将按声明顺序出现
        has_sensitive_variables = False  # 默认值：True
        # 设置为 False 以声明此 Job 的变量是非敏感的
        # 也就是说，它们可以保存到数据库并对任何可以查看此 Job 结果的用户可见
        hidden = False  # 默认值：False
        # 设置为 True 使 Job 默认不在 UI 中显示
        read_only = False  # 默认值：False
        # 设置为 True 以声明此 Job 不会对 Nautobot 数据或更广泛的环境进行任何更改
        soft_time_limit = 1000  # 默认值：None（遵循全局配置）
        # Celery 自动尝试优雅地结束 Job（通过引发 SoftTimeLimitExceeded 异常）之前的秒数
        task_queues = []  # 默认值：[]
        # 此 Job 想要路由到的任务队列名称列表
        # 通常你会在数据库中配置它，而不是将其硬编码到 Job 本身
        template_name = ""  # 默认值：""
        # 目前仅适用于 App 的功能，因为只有 Apps 可以提供额外的 Django 模板文件
        # 在 UI 中显示此 Job 输入的 Django 模板的相对路径
        time_limit = 2000  # 默认值：None（遵循全局配置）
        # Celery 自动尝试终止 Job（通过杀死 Celery worker 进程）之前的秒数
        is_singleton = False  # 默认值：False
        # 如果设置为 `True`，则阻止 Job 同时运行两次
        # 任何重复的 Job 实例都将出现特定于单例的错误消息

    # 运行此 Job 时请求的输入变量定义
    should_fail = BooleanVar(description="Check this box to force this Job to fail")
    string_input = StringVar(
        # 所有 ScriptVariable 类型的常见可选参数：
        default="Hello World!",
        description="Say hello to somebody...",
        label="Brief String Input",
        required=True,  # 所有输入变量的默认值是 True，除非另有指定
        widget=widgets.TextInput,  # StringVar 的默认小部件
        # StringVar 特有的额外可选参数
        min_length=5,  # 最少字符数
        max_length=255,  # 最大字符数
        regex=r"^Hello .*$",  # 任何提供值必须匹配的正则表达式
    )
    text_input = TextVar(
        default="Lorem ipsum...",
        description="You could type a whole essay here.",
        label="Longer Text Input",
        required=True,
    )
    integer_input = IntegerVar(
        default=0,
        min_value=-100,
        max_value=100,
    )
    choice_input = ChoiceVar(
        description="Select a single value from the given choices",
        choices=(
            ("#ff0000", "Red"),  # 值，显示字符串
            ("#00ff00", "Green"),
            ("#0000ff", "Blue"),
        ),
    )
    multiple_choice_input = MultiChoiceVar(
        description="Select any number of values from the given choices",
        choices=(
            ("#ff0000", "Red"),
            ("#00ff00", "Green"),
            ("#0000ff", "Blue"),
        ),
        required=False,
    )
    location_type_input = ObjectVar(
        description="Select a single object instance",
        model=LocationType,
    )
    object_input = ObjectVar(
        description="Select a single object instance from an API endpoint filtered by the previous selection",
        model=Location,
        display_field="name",
        query_params={"location_type": "$location_type_input"},
        null_option="(none)",
        required=False,
    )
    multiple_object_input = MultiObjectVar(
        description="Select any number of object instances",
        model=Status,
        required=False,
    )
    file_input = FileVar(
        required=False,
        description="Upload a file here",
    )
    ip_address_input = IPAddressVar(
        label="IP Address",
        description="An IPv4 or IPv6 address, without a netmask",
        default="10.0.0.1",
    )
    ip_address_with_mask_input = IPAddressWithMaskVar(
        label="IP Address with Mask",
        description="An IPv4 or IPv6 address plus netmask",
        default="10.0.0.1/24",
    )
    ip_network_input = IPNetworkVar(
        label="IP Network",
        description="An IPv4 or IPv6 network with mask",
        min_prefix_length=8,
        max_prefix_length=24,
        default="10.0.0.0/24",
    )
    json_input = JSONVar(
        label="JSON Data",
        description="A JSON value that can be parsed by Nautobot",
        default={},
    )
    # 名称 `dryrun` 是重要的！
    dryrun = DryRunVar(description="Set to True to run in dry-run mode, bypassing approval requirements")

    def before_start(self, task_id, args, kwargs):
        """
        在启动 Job run() 方法之前调用。

        如果引发任何未处理的异常，将不会调用 run()，而是直接继续调用 on_failure()。

        Args:
            task_id (str): 为 Celery 兼容性存在，大多数情况下可以忽略。
            args (list): 为 Celery 兼容性存在，大多数情况下可以忽略。
            kwargs (dict): 将传入 run() 的关键字参数。
        """
        self.logger.info("Before start! The provided kwargs are `%s`", kwargs)

    def run(self, **kwargs):  # pylint:disable=arguments-differ
        """
        任何 Job 的主要工作函数。

        如果在不引发任何异常的情况下返回，则被视为"成功"；任何引发且未处理的异常都将被视为"失败"。

        Args:
            **kwargs (dict): 对应于此 Job 定义的 ScriptVariables 的参数。
                对于任何标记为可选的变量，请务必定义默认值！

        Returns:
            data (any): 将保存到 JobResult 的数据。**必须**可以序列化为 JSON。
        """
        self.logger.info("Running!")
        for key, value in kwargs.items():
            self.logger.info("For kwarg %s, the provided value was `%s` `%s`", key, type(value), value)

        # 运行 Job 时会自动设置一些相关的实例属性：
        self.logger.debug("The user who requested this Job to run is %s", self.user.username)
        self.logger.debug("The JobResult for this Job has id %s", self.job_result.id)

        # 由于 Jobs 是 Python 类，你可以定义任何你喜欢的自定义方法并自己调用它们：
        self.demonstrate_logging(kwargs)

        # create_file() 辅助方法接受文件名和字符串或字节串作为输入
        # 用户后续可以下载创建的文件
        self.create_file("example.txt", "\n".join([kwargs["string_input"], kwargs["text_input"]]))

        if kwargs["should_fail"]:
            raise RuntimeError("This unhandled exception will cause the Job to fail")
        # else:
        return {"my job result": {"The return value on success can contain any": ["JSON", "serializable", "data"]}}

    def demonstrate_logging(self, kwargs):
        """不是魔法方法 - 这是一个从 run() 手动调用的自定义 Python 函数。"""
        self.logger.debug("This is a simple debug message, with **Markdown** formatting.")
        self.logger.info(
            "This is an info message, with an associated database object",
            extra={"object": kwargs["location_type_input"]},
        )
        self.logger.success(
            "You can use logger.success() to set the log level to `SUCCESS`.", extra={"grouping": "post_run"}
        )
        self.logger.warning(
            "You can specify a custom grouping for messages, but do so with consideration.",
            extra={"grouping": "warning messages"},
        )
        self.logger.error("Note that *logging* an error does _not_ automatically cause the job to fail.")
        self.logger.exception("Any supported Python log level can be logged but not all are automatically colorized.")

    def on_success(self, retval, task_id, args, kwargs):
        """
        如果 before_start() 和 run() 都没有引发任何异常，则调用。

        Args:
            retval (any): run() 返回的值（如果有的话）。
            task_id (str): 为 Celery 兼容性存在，大多数情况下可以忽略。
            args (list): 为 Celery 兼容性存在，大多数情况下可以忽略。
            kwargs (dict): 传入 run() 的关键字参数。
        """
        self.logger.info("Success! The retval is `%s`, kwargs were `%s`", retval, kwargs)

    def on_failure(self, exc, task_id, args, kwargs, einfo):
        """
        如果 before_start() 或 run() 引发了未处理的异常，则调用。

        Args:
            exc (Exception): 引发的异常。
            task_id (str): 为 Celery 兼容性存在，大多数情况下可以忽略。
            args (list): 为 Celery 兼容性存在，大多数情况下可以忽略。
            kwargs (dict): 传入（或将要传入）run() 的关键字参数。
            einfo (any): 为 Celery 兼容性存在，大多数情况下可以忽略。
        """
        self.logger.error("Failure! The exception is `%s`, kwargs were `%s`", exc, kwargs)

    def after_return(self, status, retval, task_id, args, kwargs, einfo):
        """
        无论 Job 的成功或失败，在 on_success() 和 on_failure() 之后调用。

        Args:
            status (JobResultStatusChoices): 要么是 `STATUS_SUCCESS` 要么是 `STATUS_FAILURE`。
            retval (any): run() 的返回值（成功时）或引发的 `Exception`（失败时）。
            task_id (str): 为 Celery 兼容性存在，大多数情况下可以忽略。
            args (list): 为 Celery 兼容性存在，大多数情况下可以忽略。
            kwargs (dict): 传入 run() 的关键字参数。
            einfo (any): 为 Celery 兼容性存在，大多数情况下可以忽略。
        """
        self.logger.info(
            "After return! The status is `%s`, retval is `%s`, Job kwargs were `%s`", status, retval, kwargs
        )


class ExampleDryRunJob(Job):
    dryrun = DryRunVar()

    class Meta:
        approval_required = True
        has_sensitive_variables = False
        description = "Example job to remove serial number on all devices, supports dryrun mode."

    def run(self, dryrun):  # pylint:disable=arguments-differ
        try:
            with transaction.atomic():
                devices_with_serial = Device.objects.exclude(serial="")
                log_msg = "Removing serial on %s devices."
                if dryrun:
                    log_msg += " (DRYRUN)"
                self.logger.info(log_msg, devices_with_serial.count())
                for device in devices_with_serial:
                    if not dryrun:
                        device.serial = ""
                        device.save()
        except Exception:
            self.logger.error("%s failed. Database changes rolled back.", self.__class__.__name__)
            raise
        self.logger.success("We can use the success log level to indicate success.")
        # 确保 get_task_logger 也能使用 success
        logger = get_task_logger(__name__)
        logger.success("We can also use the success log level in get_task_logger.")


class ExampleJob(Job):
    some_json_data = JSONVar(label="JSON", description="Example JSONVar for a job.", default={})

    # 指定 template_name 以覆盖默认的 job 调度模板
    template_name = "example_app/example_with_custom_template.html"

    class Meta:
        name = "Example job, does nothing"
        description = """
            Markdown Formatting

            *This is italicized*
        """

    def run(self, some_json_data):  # pylint:disable=arguments-differ
        # some_json_data 作为 Python 对象（例如字典）传入 run 方法
        pass


class ExampleCustomFormJob(Job):
    """此 job 提供一个简单的 HTML 表单，而不是动态生成的表单。

    Nautobot `Job` 基类提供了一种机制来自动生成一个表单
    以在运行视图中显示。这涵盖了大多数显示 job
    输入表单的用例。但是，有时需要提供一些定制
    在视图中。这个特定的 job 将简单地用不同的元素替换表单元素。

    # Nautobot jobs 使用脚本变量来生成动态表单，但
    # 此表单也用于处理提交的数据。因此，为了
    # 将数据提交到已提交的 job，必须使用脚本变量，并且提交的
    # 表单字段名称必须与这些脚本变量名称匹配。只要
    # 表单字段名称匹配并包含与 `id_{form_field_name}` 匹配的 id，那么
    # 一切都会按预期工作。
    #
    # 这里的 `StringVar` 将显示一个简单的输入字段，但我们将在模板中
    # 显示一个文本区域。
    """
    custom_job_data = StringVar(label="Input Data", description="Some input data", default="Lorem Ipsum")

    # 指定 template_name 以覆盖默认的 job 调度模板
    template_name = "example_app/custom_job_form.html"

    class Meta:
        name = "Custom form."
        has_sensitive_variables = False

    def run(self, custom_job_data):  # pylint:disable=arguments-differ
        """运行 job。"""
        self.logger.debug("Data is %s", custom_job_data)


class ExampleHiddenJob(Job):
    class Meta:
        hidden = True
        name = "Example hidden job"
        description = "I should not show in the UI!"

    def run(self):  # pylint:disable=arguments-differ
        pass


class ExampleLoggingJob(Job):
    interval = IntegerVar(default=4, description="The time in seconds to sleep.")

    class Meta:
        name = "Example logging job."
        description = "I log stuff to demonstrate how UI logging works."
        task_queues = [
            settings.CELERY_TASK_DEFAULT_QUEUE,
            "priority",
            "bulk",
        ]

    def run(self, interval):  # pylint:disable=arguments-differ
        self.logger.debug("Running for %s seconds.", interval)
        for step in range(1, interval + 1):
            time.sleep(1)
            self.logger.info("Step %s", step)
            print(f"stdout logging for step {step}, task: {self.request.id}")
            print(f"stderr logging for step {step}, task: {self.request.id}", file=sys.stderr)
        self.logger.critical(
            "This log message will not be logged to the database but will be logged to the console.",
            extra={"skip_db_logging": True},
        )
        self.logger.info("Success", extra={"object": self.job_model, "grouping": "job_run_success"})
        return f"Ran for {interval} seconds"


class ExampleFileInputOutputJob(Job):
    input_file = FileVar(description="Text file to transform")

    class Meta:
        name = "Example File Input/Output job"
        description = "Takes a file as input and reverses its line order, creating a new file as output."

    def run(self, input_file):  # pylint:disable=arguments-differ
        # 注意 input_file 总是以二进制模式打开，所以我们需要将其解码为 str
        text = input_file.read().decode("utf-8")
        output = "\n".join(reversed(text.split("\n")))
        # create_file(filename, content) 可以接受 str 或 bytes 作为内容
        self.create_file("output.txt", output)


class ExampleJobHookReceiver(JobHookReceiver):
    class Meta:
        name = "Example job hook receiver"
        description = "Validate changes to object serial field"

    def receive_job_hook(self, change, action, changed_object):
        # 在删除操作时返回
        if action == ObjectChangeActionChoices.ACTION_DELETE:
            return

        # 记录差异输出
        snapshots = change.get_snapshots()
        self.logger.info("DIFF: %s", snapshots["differences"])

        # 验证对 serial 字段的更改
        if "serial" in snapshots["differences"]["added"]:
            old_serial = snapshots["differences"]["removed"]["serial"]
            new_serial = snapshots["differences"]["added"]["serial"]
            self.logger.info("%s serial has been changed from %s to %s", changed_object, old_serial, new_serial)

            # 检查新的 serial 是否有效，必要时恢复
            if not self.validate_serial(new_serial):
                changed_object.serial = old_serial
                changed_object.save()
                self.logger.info("%s serial %s was not valid. Reverted to %s", changed_object, new_serial, old_serial)

            self.logger.info("Serial validation completed for %s", changed_object)

    def validate_serial(self, serial):
        # 添加验证 serial 的业务逻辑
        return False


class ExampleSimpleJobButtonReceiver(JobButtonReceiver):
    class Meta:
        name = "Example Simple Job Button Receiver"

    def receive_job_button(self, obj):
        self.logger.info("Running Job Button Receiver.", extra={"object": obj})
        # 在这里添加 job 逻辑


class ExampleComplexJobButtonReceiver(JobButtonReceiver):
    class Meta:
        name = "Example Complex Job Button Receiver"

    def _run_location_job(self, obj):
        self.logger.info("Running Location Job Button Receiver.", extra={"object": obj})
        # 运行 Location Job 函数

    def _run_device_job(self, obj):
        self.logger.info("Running Device Job Button Receiver.", extra={"object": obj})
        # 运行 Device Job 函数

    def receive_job_button(self, obj):
        user = self.user
        if isinstance(obj, Location):
            if not user.has_perm("dcim.add_location"):
                self.logger.error("User '%s' does not have permission to add a Location.", user, extra={"object": obj})
            else:
                self._run_location_job(obj)
        elif isinstance(obj, Device):
            if not user.has_perm("dcim.add_device"):
                self.logger.error("User '%s' does not have permission to add a Device.", user, extra={"object": obj})
            else:
                self._run_device_job(obj)
        else:
            self.logger.error("Unable to run Job Button for type %s.", type(obj).__name__, extra={"object": obj})


class ExampleSingletonJob(Job):
    class Meta:
        name = "Example job, only one can run at any given time."
        is_singleton = True

    def run(self, *args, **kwargs):
        time.sleep(60)

class TestingAppHelloJobsWithLogs(Job):

    class Meta:
        name = "Testing App Hello Jobs with Logs"
        description = "Testing App Hello Jobs with different log types"

    def run(self):
        self.logger.info("This is an info type log in Testing App.")
        self.logger.debug("This is a debug type log in Testing App.")
        self.logger.warning("This is a warning type log in Testing App.")
        self.logger.error("This is an error type log in Testing App.")
        self.logger.critical("This is a critical type log in Testing App.")

jobs = (
    ExampleEverythingJob,
    ExampleDryRunJob,
    ExampleJob,
    ExampleCustomFormJob,
    ExampleHiddenJob,
    ExampleLoggingJob,
    ExampleFileInputOutputJob,
    ExampleJobHookReceiver,
    ExampleSimpleJobButtonReceiver,
    ExampleComplexJobButtonReceiver,
    ExampleSingletonJob,
    TestingAppHelloJobsWithLogs,
)
register_jobs(*jobs)
```

记得执行 `invoke post-upgrade` 以允许新 job 被注册。完成后，新 job 应该出现在 `JOBS` 菜单下：

![jobs_1](images/jobs_1.png)

启用 Job 并运行它，我们应该熟悉 Job 结果页面：

![jobs_2](images/jobs_2.png)

恭喜完成我们使用 Nautobot 仓库 `example_app` 进行 App 开发的 9 部分系列的最后一程！

## 第 58 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布带有新 app 的 job 结果截图，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将涵盖过去几天没有机会涵盖的一些主题。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+58+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 58 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）