# Nautobot 数据库模型第二部分：自定义数据模型

在某些时候，我们会想要创建自己的数据模型。自定义数据模型提供以下几个优点：

- **定制化和灵活性**：自定义数据模型允许你定制数据库 schema 以满足应用程序的特定需求。这种灵活性使你能够定义对你的用例独特的字段、关系和行为。

- **数据完整性和验证**：通过定义自己的数据模型，你可以在数据库级别强制执行数据完整性和验证规则（我们将在明天的挑战中讨论数据完整性）。这确保了存储在数据库中的数据是一致的，并符合定义的约束。

- **优化查询**：自定义数据模型允许你优化数据库查询以提高性能（我们将在 [第 65 天](../Day065_Database_Models_5_Performance_and_Scalability/README.md) 讨论性能）。你可以定义索引、约束和关系，使数据检索更高效。

- **增强功能**：通过自定义数据模型，你可以为模型添加自定义方法和属性，增强应用程序的功能。这允许你将业务逻辑封装在模型本身中。

- **与 Nautobot 数据模型的无缝集成**：也许最大的优势是，如果你已经在 Nautobot 中有数据，自定义数据模型可以与 Nautobot 现有数据模型（如 `devices` 和 `circuits`）无缝集成。

其他优点包括安全性、可管理性和安全性。

自定义数据模型可以作为自定义应用的一部分创建，也可以作为我们可以与 Nautobot Jobs 一起使用的独立数据模型。

让我们看几个自定义数据模型的例子。请记住，这些模型仅用于说明目的，其中许多已经是现有 Nautobot 应用和核心模型的一部分。

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

## 数据库关系

当我们对现实世界的对象建模时，我们必须考虑的重要方面之一是对象之间的关系。

在 Django 中，数据库关系使用模型中的特定字段类型定义，这些类型表示不同表之间的关系。以下是主要类型的数据库关系及其对应的 Django ORM 命令：

> [!信息]
> 这些仅用于说明目的，无需输入它们。

1. 一对多关系（ForeignKey）
一对多关系是指一个表中的记录可以与另一个表中的多条记录相关联。这在 Django 中用 *ForeignKey* 表示。

这是一个经典的作者和书籍示例，一个"作者"可以有多本"书"。

> [!信息]
> 这些很容易理解，无需创建，但可以在 Codespace 环境中自由创建。

```python
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=200)

    def __str__(self):
        return self.name

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)

    def __str__(self):
        return self.title
```

在 Django 中，*ForeignKey* 用于在两个模型之间创建多对一关系。这意味着带有 *ForeignKey* 字段的模型的每个实例都与引用模型的一个实例相关，但引用模型可以与带有 *ForeignKey* 的模型的多个实例相关。在我们的示例中，Book 是作者的 *多*，所以我们在 Book 模型上使用 *ForeignKey*。

*on_delete* 参数指定了删除引用对象（在这种情况下是 Author）时的行为。在我们的示例中，*CASCADE* 意味着当引用对象被删除时，我们也会删除有外键指向它的对象。

创建作者及其书籍的相应示例：

```python
author = Author.objects.create(name="J.K. Rowling")
book1 = Book.objects.create(title="Harry Potter and the Philosopher's Stone", author=author)
book2 = Book.objects.create(title="Harry Potter and the Chamber of Secrets", author=author)

# 检索作者的书
books = Book.objects.filter(author=author)
```

2. 多对多关系（ManyToManyField）
多对多关系是指一个表中的记录可以与另一个表中的多条记录相关联，反之亦然。这在 Django 中用 *ManyToManyField* 表示。

这是（多）课程对（多）学生的示例：

```python
from django.db import models

class Student(models.Model):
    name = models.CharField(max_length=200)

    def __str__(self):
        return self.name

class Course(models.Model):
    title = models.CharField(max_length=200)
    students = models.ManyToManyField(Student)

    def __str__(self):
        return self.title
```

创建学生和课程：

```python
student1 = Student.objects.create(name="Alice")
student2 = Student.objects.create(name="Bob")
course = Course.objects.create(title="Django 101")
course.students.add(student1, student2)
```

检索课程和学生：

```python
courses = student1.course_set.all()
students = course.students.all()
```

3. 一对一关系（OneToOneField）
一对一关系是指一个表中的记录正好与另一个表中的一条记录相关联。这在 Django 中用 *OneToOneField* 表示。

```python
from django.db import models

class User(models.Model):
    username = models.CharField(max_length=200)

    def __str__(self):
        return self.username

class Profile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    bio = models.TextField()

    def __str__(self):
        return self.user.username
```

ORM 命令：

```python
user = User.objects.create(username="john_doe")
profile = Profile.objects.create(user=user, bio="Software Developer")

profile = user.profile

user = profile.user
```

这些关系允许你以反映实体之间现实世界关系的方式构建数据库，而 Django 的 ORM 使使用 Python 代码与这些关系交互变得容易。

让我们在 Nautobot 的网络工程上下文中尝试一个数据模型。

## Asset 模型

我们可以尝试创建一个 **Asset 模型**。它将是一个用于跟踪物理资产的模型，其中每个资产都与特定设备相关联。

```python
from django.db import models

class Asset(models.Model):
    device = models.ForeignKey(Device, on_delete=models.CASCADE)
    serial_number = models.CharField(max_length=100, unique=True)
    purchase_date = models.DateField()
    warranty_expiration = models.DateField()

    def __str__(self):
        return f"Asset {self.serial_number} for {self.device}"
```

我们可以将上述代码放在 `nautobot -> dcim -> models -> devices.py` 下：

![asset_model](images/asset_model_1.png)

我们需要进行数据库迁移：

```shell
(nautobot-py3.10) @ericchou1 ➜ ~/nautobot (develop) $ invoke makemigrations

(nautobot-py3.10) @ericchou1 ➜ ~/nautobot (develop) $ invoke migrate

...
```

让我们再创建一个自定义模型。

## Maintenance Schedule 模型

**Maintenance Schedule 模型**：一个用于安排和跟踪 location 内设备维护活动的模型。

```python
from django.db import models

class MaintenanceSchedule(models.Model):
    device = models.ForeignKey(Device, on_delete=models.CASCADE)
    maintenance_date = models.DateField()
    description = models.TextField()

    def __str__(self):
        return f"Maintenance for {self.device} on {self.maintenance_date}"
```

迁移数据库：

```shell
(nautobot-py3.10) @ericchou1 ➜ ~/nautobot (develop) $ invoke makemigrations
Running docker compose command "exec nautobot nautobot-server makemigrations"
...
Migrations for 'dcim':
  nautobot/dcim/migrations/0069_maintenanceschedule.py
    - Create model MaintenanceSchedule

(nautobot-py3.10) @ericchou1 ➜ ~/nautobot (develop) $ invoke migrate
...
Applying dcim.0069_maintenanceschedule... OK
...
```

让我们验证自定义模型是否已创建。

## 验证

我们可以使用 `nbshell` 验证模型是否已创建：

```shell
(nautobot-py3.10) @ericchou1 ➜ ~/nautobot (develop) $ invoke nbshell
...
>>> from nautobot.dcim.models import *
>>> dir()
['AdminGroup', 'AnotherExampleModel', 'Asset', 'Association', ..., 'MaintenanceSchedule', 'Manufacturer', ...
>>>
```

![model_list](images/model_list.png)

在 Nautobot 中创建自定义模型是我们定制 Nautobot 的主要步骤之一。干得好！

## 第 62 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 上停止 codespace 实例。

继续在你选择的社交媒体上发布你今天挑战中创建自定义数据模型的步骤，一定要使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并标记 `@networktocode`，这样我们就可以分享你的进度！

在明天的挑战中，我们将讨论数据完整性和安全性。明天见！

[X/Twitter](https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+62+of+the+100+days+of+nautobot+challenge+!&hashtags=100DaysOfNautobot,JobsToBeDone)

[LinkedIn](https://www.linkedin.com/) （复制粘贴：我刚刚完成了 100 天 Nautobot 挑战的第 62 天，https://github.com/nautobot/100-days-of-nautobot，挑战！@networktocode #JobsToBeDone #100DaysOfNautobot）