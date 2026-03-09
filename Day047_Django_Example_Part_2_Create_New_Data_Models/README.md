# Django 入门（二）：创建数据模型

在昨天的基础上，今天我们将为 `polls` 应用创建两个新的数据库模型。

## 代码示例

我们将创建 `Question`（问题）和 `Choice`（选项）两个模型。[Django 教程第二部分](https://docs.djangoproject.com/en/5.1/intro/tutorial02/)对各字段含义及模型间关系有详尽的讲解，请参阅原文以获取更多信息。

`polls/models.py` 文件内容如下：

```python
# polls/models.py
from django.db import models

class Question(models.Model):
    question_text = models.CharField(max_length=200)
    pub_date = models.DateTimeField('date published')

    def __str__(self):
        return self.question_text

class Choice(models.Model):
    question = models.ForeignKey(Question, on_delete=models.CASCADE)
    choice_text = models.CharField(max_length=200)
    votes = models.IntegerField(default=0)

    def __str__(self):
        return self.choice_text
```

Django 提供了 `makemigrations` 和 `migrate` 两个命令，用于将数据库结构的变更落地生效。关于这两个命令的具体作用，请参阅 [Django 教程第二部分](https://docs.djangoproject.com/en/5.1/intro/tutorial02/)。

这里我们直接执行迁移，使变更生效：

```
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ python manage.py makemigrations polls
Migrations for 'polls':
  polls/migrations/0001_initial.py
    + Create model Question
    + Create model Choice
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, polls, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying auth.0012_alter_user_first_name_max_length... OK
  Applying polls.0001_initial... OK
  Applying sessions.0001_initial... OK
```

Django 内置了一套管理后台，可以直接管理数据库条目，但我们需要先在 `polls/admin.py` 中注册数据库模型：

```python
# polls/admin.py
from django.contrib import admin
from .models import Question

admin.site.register(Question)
```

为项目创建超级管理员账户，用户名和密码均设为 "admin"：

```
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ python manage.py createsuperuser
Username (leave blank to use 'vscode'): admin
Email address: admin@admin.com
Password: 
Password (again): 
The password is too similar to the username.
This password is too short. It must contain at least 8 characters.
This password is too common.
Bypass password validation and create user anyway? [y/N]: y
Superuser created successfully.
```

重新启动开发服务器：

```
(djangoproject-py3.10) @ericchou1 ➜ ~/djangoproject/mysite $ python manage.py runserver 0.0.0.0:8080
```

在 URL 末尾加上 `/admin` 即可访问管理后台：

![admin_1](images/admin_1.png)

点击 `+ Add` 按钮可以创建新问题：

![admin_2](images/admin_2.png)

页面上提供了文字、日期和时间输入字段，可以用 `Today` 和 `Now` 按钮快速填入当前日期和时间：

![admin_3](images/admin_3.png)

出色地完成了今天的挑战！明天我们将创建新的视图和模板。

## 第 47 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 Codespace 实例。

请在你选择的社交媒体上发布在管理后台操作 polls 数据库条目的截图，务必使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，这样我们可以分享你的进展！

明天的挑战，我们将创建新的视图和 HTML 模板。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+47+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 47 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
