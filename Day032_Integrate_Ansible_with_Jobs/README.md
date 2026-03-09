# 将 Ansible 与 Jobs 集成

在今天的挑战中，我们将在 Nautobot Job 中调用一个简单的 Ansible Playbook。这个示例并非用于生产环境，而是为了展示这两款主流工具之间的互操作性。

可以看到，我们的 Job 配置正在逐步变得更加复杂，这令人兴奋！

## 环境配置

环境配置与 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md) 相同，以下是步骤摘要，如需详细背景说明请参阅该指南。

按照以下步骤启动 Nautobot，如果是重启已有实例且 `build` 和 `db-import` 已完成，可以跳过相应步骤：
```
$ cd nautobot-docker-compose/
$ poetry shell
$ invoke build
$ invoke db-import
$ invoke debug
```

上传并准备 cEOS 镜像，然后启动 Containerlab：
```
$ docker import cEOS64-lab-4.32.0F.tar ceos:4.32.0F
```

本实验只需要 `bos-acc-01` 设备：
```
$ cd ~/100-days-of-nautobot/clab/
$ sudo containerlab deploy --topo ceos-lab.clab.yml --node-filter bos-acc-01
```

今天挑战的环境已配置完毕。

## Ansible 配置

默认情况下，Ansible 并未安装在我们的 Docker 容器中。需要分别在 `nautobot-worker` 和 `nautobot` 上安装。

打开两个终端窗口，分别连接到 `nautobot-worker` 和 `nautobot`：
```
$ docker exec -u root -it nautobot_docker_compose-celery_worker-1 bash
$ docker exec -u root -it nautobot_docker_compose-nautobot-1 bash
```

在两个容器中分别执行以下步骤安装 Ansible：
```shell
root@0936589bc72d:/opt/nautobot# apt update
root@0936589bc72d:/opt/nautobot# apt install -y software-properties-common
root@0936589bc72d:/opt/nautobot# apt-get install -y python3-software-properties
root@0936589bc72d:/opt/nautobot# echo "deb http://ppa.launchpad.net/ansible/ansible/ubuntu focal main" > /etc/apt/sources.list.d/ansible.list
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 93C4A3FD7BB9C367
apt-get update
root@0936589bc72d:/opt/nautobot# apt-get install -y ansible vim python3-paramiko
root@771b55abc34a:/opt/nautobot# export ANSIBLE_HOST_KEY_CHECKING=False

root@0936589bc72d:/opt/nautobot# ansible --version
ansible [core 2.14.18]
  config file = None
  configured module search path = ['/root/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  ansible collection location = /root/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.11.2 (main, Nov 30 2024, 21:22:50) [GCC 12.2.0] (/usr/bin/python3)
  jinja version = 3.1.2
  libyaml = True
```

现在可以开始构建 Nautobot Job 了。

## 包含 Ansible Playbook 的 Nautobot Job

在 `/opt/nautobot/jobs` 目录中创建 `hello_world.yml` Playbook：
```
root@0936589bc72d:/opt/nautobot/jobs# cat hello_world.yml 
---
- name: Hello World Playbook
  hosts: all
  gather_facts: no
  tasks:
    - name: Print Hello World
      debug:
        msg: "Hello World from {{ inventory_hostname }}"
```

该 Playbook 会打印 `inventory_hostname`，以确认我们将所选设备正确传递给了 Playbook。

我们可以编写一个 Nautobot Job `hello_ansible_test.py`，允许用户从 Nautobot 中选择设备，并将相关信息写入一个临时文件，该文件随后将作为 Ansible 的动态 Inventory 文件使用。

Ansible Playbook 通过 Python 的 `subprocess` 调用。

Nautobot Job 内容如下：
```python 
from nautobot.apps.jobs import MultiChoiceVar, MultiObjectVar, Job, ObjectVar, register_jobs, StringVar, IntegerVar
from nautobot.dcim.models.devices import Device
from nautobot.dcim.models.locations import Location
import subprocess
import json 

name = "Operations with Ansible"

class HelloAnsible(Job):
    devices = MultiObjectVar(
        model=Device,
    )

    class Meta:
        name = "Ansible Hello World"
        description = "A job to call Ansible playbook."

    def run(self, devices):
        inventory = {"all": {"hosts": {}}}
        # 收集 Inventory 信息
        for device in devices:
            ip_address = str(device.primary_ip).split('/')[0] 
            inventory["all"]["hosts"][device.name] = {
                "ansible_host": ip_address,
                "ansible_user": "admin",  
                "ansible_password": "admin",
                "ansible_connection": "network_cli", 
                "ansible_network_os": "eos",
                "ansible_become": True,
                "ansible_become_method": "enable"
            }

        # 将 Inventory 写入临时文件
        inventory_file = "/tmp/inventory.json"
        with open(inventory_file, "w") as f:
            json.dump(inventory, f)

        # 运行 Ansible Playbook
        device = str(device.primary_ip).split('/')[0]
        result = subprocess.run(
            ["ansible-playbook", "-i", inventory_file, "/opt/nautobot/jobs/hello_world.yml"],
            capture_output=True,
            text=True,
        )

        if result.returncode != 0:
            self.logger.fatal(f"Ansible playbook failed: {result.stderr}")
            return

        self.logger.info(f"Successfully run ansible playbook, {result}.")


register_jobs(
    HelloAnsible,
)
```

启用 Job 后，可以执行并观察结果：

![ansible_hello_world_1](images/ansible_hello_world_1.png)

在"Job Result"页面中，可以看到 Ansible Playbook 的输出，包括所选设备的主机名：

![ansible_hello_world_2](images/ansible_hello_world_2.png)

## Ansible Playbook 故障排查

由于大部分 Nautobot Job 配置在前几天已有详细介绍，今天挑战中遇到的问题很可能来自 Ansible 本身。

建议先直接在 Nautobot 主机上测试运行 Ansible，确认其可以正常执行。

以下是几个需要检查的事项。

1. 确认 Inventory 文件内容正确：
```
root@771b55abc34a:/opt/nautobot/jobs# cat /tmp/inventory.json 
{"all": {"hosts": {"bos-acc-01.infra.valuemart.com": {"ansible_host": "172.17.0.2", "ansible_user": "admin", "ansible_password": "admin", "ansible_connection": "network_cli", "ansible_network_os": "eos", "ansible_become": true, "ansible_become_method": "enable"}}}}
```

2. 检查 Ansible Playbook 内容：
```
root@771b55abc34a:/opt/nautobot/jobs# cat hello_world.yml 
---
- name: Hello World Playbook
  hosts: all
  gather_facts: no
  tasks:
    - name: Print Hello World
      debug:
        msg: "Hello World from {{ inventory_hostname }}"
```

3. 在本地直接执行 Ansible Playbook：
```
root@771b55abc34a:/opt/nautobot/jobs# ansible-playbook -i /tmp/inventory.json hello_world.yml 

PLAY [Hello World Playbook] *************************************************************************************************************************

TASK [Print Hello World] ****************************************************************************************************************************
[WARNING]: ansible-pylibssh not installed, falling back to paramiko
ok: [bos-acc-01.infra.valuemart.com] => {
    "msg": "Hello World from bos-acc-01.infra.valuemart.com"
}

PLAY RECAP ******************************************************************************************************************************************
bos-acc-01.infra.valuemart.com : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```

恭喜完成第 032 天的挑战，您表现得非常出色！

## 第 32 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布成功调用 Ansible Playbook 的 Job 执行截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+32+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 32 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
