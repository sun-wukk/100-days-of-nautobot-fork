# 将 Nautobot 与 Ansible 工作流集成

在过去 32 天里，我们深入探讨了 Nautobot Jobs，并重点关注 Python（这是网络工程师的重要技能）。然而，许多组织已将自动化标准化为 Ansible。Network to Code 为 Nautobot 维护了一个出色的 [Ansible 库](https://github.com/nautobot/nautobot-ansible)，其中包含管理所有模型的模块，包括组和权限等管理功能。

今天，我们将设置 Ansible Playbook 来管理容器实验室设备，并以 Nautobot 作为唯一可信源（Source of Truth）。从 Ansible 角度来看，最强大的功能之一是能够通过 Nautobot 动态处理组变量和主机变量，使我们无需在 Ansible 服务器上单独维护一套组变量和主机变量文件，同时保持 Nautobot 作为唯一可信源。

## 环境配置

环境配置遵循 [Lab Setup Scenario 1](../Lab_Setup/scenario_1_setup/README.md)，以下是步骤摘要，如需详细背景说明请参阅该指南。

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

## 将 Nautobot 用作 Ansible 动态 Inventory

今天挑战的环境已配置完毕。我们将使用 Nautobot 作为动态 Inventory，并创建几个 Playbook 来修改 `bos-acc-01` 上的配置。

首先，设置 Inventory 文件以使用 Nautobot 的 Inventory 模块。

> [!TIP]
> 若要熟悉 Nautobot Collection 中的所有 Ansible 模块，请访问 Git 仓库并查看 `plugins` 文件夹。在 `inventory` 文件夹中可以找到 `gql_inventory.py`，其中提供了设置 Inventory 文件的示例。

该模块使用默认的 GraphQL 查询，从 Nautobot 中拉取设备和虚拟机。我们可以过滤结果并从查询中排除虚拟机，这在管理大量设备和虚拟机时非常有用。

我们使用其中一个示例进行测试。可以在查询中包含 API Token，但作为最佳实践，应避免将其写入 Playbook 或任何可能暴露的文件。我们有两种方式：

1. 创建名为 `NAUTOBOT_TOKEN` 的环境变量。
2. 使用 Ansible Vault 将 Token 存储在加密文件中。

我们采用环境变量方式。API Token 可以在 Nautobot 应用的 ADMIN --> Profile --> API Tokens 中找到。

![API_TOKEN](images/API_Token.png)

在文件末尾添加以下内容：
```
export NAUTOBOT_TOKEN="<在此处填入 ADMIN Profile 中的 Token>"
```

### BASH 操作说明

如果您修改了 `.bashrc` 文件，请执行以下操作：打开新终端并运行以下命令：
```bash
$ source ~/.bashrc
$ echo $NAUTOBOT_TOKEN
```

### ZSH 操作说明

如果您修改了 `.zshrc` 文件，请执行以下操作：打开新终端并运行以下命令：
```bash
$ zsh ~/.bashrc
$ echo $NAUTOBOT_TOKEN
```

`echo` 命令应在终端中显示您的 Nautobot API Token。接下来，我们来配置运行 Ansible Playbook 的环境。我们将创建一个 Python 虚拟环境，与用于 `nautobot-docker-compose` 的 Poetry 环境保持隔离。
```bash
$ mkdir nautobot-ansible-example
$ cd nautobot-ansible-example
$ sudo apt update
$ sudo apt install python3-venv -y
$ python3 -m venv .ansbile
$ source .ansbile/bin/activate
$ pip3 install ansible pynautobot paramiko netutils
$ ansible-galaxy collection install networktocode.nautobot
$ ansible-galaxy collection install arista.eos --force
```

创建新文件 `inventory.yml` 并添加以下内容：
```yaml
---
plugin: networktocode.nautobot.gql_inventory
api_endpoint: http://localhost:8080
query:
  devices:
    tags: name
    serial:
    tenant: name
    location:
      name:
      contact_name:
      description:
      parent: name
  virtual_machines:
    tags: name
    tenant: name
```

同时创建 `ansible.cfg` 文件：
```ini
[defaults]
host_key_checking = False

COLLECTIONS_PATHS = ./ansible_collections
```

查看 Ansible 的 Inventory 内容，运行：
```bash
ansible-inventory -v --list -i inventory.yml --yaml
```

您应该看到类似如下的输出：
```yaml
all:
  children:
    ungrouped:
      hosts:
        bos-acc-01.infra.valuemart.com:
          ansible_host: 172.17.0.2
          ansible_network_os: arista.eos.eos
          location:
            contact_name: ''
            description: ''
            name: Boston
            parent:
              name: East Coast
          name: bos-acc-01.infra.valuemart.com
          platform:
            napalm_driver: eos
          primary_ip4:
            host: 172.17.0.2
        bos-rtr-01.infra.valuemart.com:
          ansible_host: 172.17.0.3
          ansible_network_os: arista.eos.eos
          location:
            contact_name: ''
            description: ''
            name: Boston
            parent:
              name: East Coast
          name: bos-rtr-01.infra.valuemart.com
          platform:
            napalm_driver: eos
          primary_ip4:
            host: 172.17.0.3
        nyc-acc-01.infra.valuemart.com:
          ansible_host: nyc-acc-01.infra.valuemart.com
          ansible_network_os: arista.eos.eos
          location:
            contact_name: ''
            description: ''
            name: New York City
            parent:
              name: East Coast
          name: nyc-acc-01.infra.valuemart.com
          platform:
            napalm_driver: eos
        nyc-rtr-01.infra.valuemart.com:
          ansible_host: nyc-rtr-01.infra.valuemart.com
          ansible_network_os: arista.eos.eos
          location:
            contact_name: ''
            description: ''
            name: New York City
            parent:
              name: East Coast
          name: nyc-rtr-01.infra.valuemart.com
          platform:
            napalm_driver: eos
```

如果您和我一样以 YAML 格式管理 Ansible Inventory，会发现这个输出结构非常熟悉。Nautobot 中设置的主 IP 被转换为 `ansible_host`，`ansible_network_os` 则使用已配置平台的 Network Driver Mappings。由于我们没有指定任何分组，这个动态 Inventory 中没有组信息。让我们更新 `inventory.yml` 文件，使用 `location.name` 添加 `group_by` 子句：
```yaml
---
plugin: networktocode.nautobot.gql_inventory
api_endpoint: http://localhost:8080
query:
  devices:
    tags: name
    serial:
    tenant: name
    role: name
    location:
      name:
      contact_name:
      description:
      parent: name
  virtual_machines:
    tags: name
    tenant: name
group_by:
  - location.name
```

> [!TIP]
> 要使用 `group_by`，请确保相关字段已包含在查询中。

现在输出应该类似如下，设备按位置分组：
```yaml
all:
  children:
    Boston:
      hosts:
        bos-acc-01.infra.valuemart.com:
          ansible_host: 172.17.0.2
          ansible_network_os: arista.eos.eos
          location:
            contact_name: ''
            description: ''
            name: Boston
            parent:
              name: East Coast
          name: bos-acc-01.infra.valuemart.com
          platform:
            napalm_driver: eos
          primary_ip4:
            host: 172.17.0.2
          role:
            name: Switch
        bos-rtr-01.infra.valuemart.com:
          ansible_host: 172.17.0.3
          ansible_network_os: arista.eos.eos
          location:
            contact_name: ''
            description: ''
            name: Boston
            parent:
              name: East Coast
          name: bos-rtr-01.infra.valuemart.com
          platform:
            napalm_driver: eos
          primary_ip4:
            host: 172.17.0.3
          role:
            name: Router
    New York City:
      hosts:
        nyc-acc-01.infra.valuemart.com:
          ansible_host: nyc-acc-01.infra.valuemart.com
          ansible_network_os: arista.eos.eos
          location:
            contact_name: ''
            description: ''
            name: New York City
            parent:
              name: East Coast
          name: nyc-acc-01.infra.valuemart.com
          platform:
            napalm_driver: eos
          role:
            name: Switch
        nyc-rtr-01.infra.valuemart.com:
          ansible_host: nyc-rtr-01.infra.valuemart.com
          ansible_network_os: arista.eos.eos
          location:
            contact_name: ''
            description: ''
            name: New York City
            parent:
              name: East Coast
          name: nyc-rtr-01.infra.valuemart.com
          platform:
            napalm_driver: eos
          role:
            name: Router
```

这种动态 Inventory 为 Nautobot 开辟了令人兴奋的可能性——不再需要将组变量或主机变量存储在本地或 Git 中的静态文件里，现在可以直接用 Nautobot 存储这些信息并动态获取！

您可能会问："那么不对应 Nautobot 现有对象的变量怎么办？"这是个好问题！Nautobot 为此提供了解决方案。在 Extensibility 菜单的"AUTOMATION"下，有一个"Config Context"部分。

![config_context](images/config_context.png)

> [!TIP]
> 您也可以从 Git 仓库同步 Config Contexts。

在这里，您可以根据位置、角色、设备类型、集群组等条件，为设备分配 JSON 格式的配置数据。为 Arista vEOS 设备类型创建一个新的 Config Context，内容如下：
```json
{
    "snmp": {
        "community": [
            {
                "name": "networktocode",
                "role": "ro"
            },
            {
                "name": "secure",
                "role": "rw"
            }
        ]
    }
}
```

![snmp_data](images/snmp_data.png)

再次更新 `inventory.yml`，在查询中加入 `config_context`：
```yaml
---
plugin: networktocode.nautobot.gql_inventory
api_endpoint: http://localhost:8080
query:
  devices:
    tags: name
    serial:
    tenant: name
    role: name
    location:
      name:
      contact_name:
      description:
      parent: name
    config_context:
  virtual_machines:
    tags: name
    tenant: name
group_by:
  - location.name
```

注意 Inventory 中新增的 `config_context` 字段。由于这是 Inventory 文件的一部分，您可以像访问其他 Inventory 条目一样在 Ansible 中使用这些变量：
```yaml
all:
  children:
    Boston:
      hosts:
        bos-acc-01.infra.valuemart.com:
          ansible_host: 172.17.0.2
          ansible_network_os: arista.eos.eos
          config_context:
            snmp:
              community:
              - name: networktocode
                role: ro
              - name: secure
                role: rw
          location:
            contact_name: ''
            description: ''
            name: Boston
            parent:
              name: East Coast
          name: bos-acc-01.infra.valuemart.com
          platform:
            napalm_driver: eos
          primary_ip4:
            host: 172.17.0.2
          role:
            name: Switch
        bos-rtr-01.infra.valuemart.com:
          ansible_host: 172.17.0.3
          ansible_network_os: arista.eos.eos
          config_context:
            snmp:
              community:
              - name: networktocode
                role: ro
              - name: secure
                role: rw
          location:
            contact_name: ''
            description: ''
            name: Boston
            parent:
              name: East Coast
          name: bos-rtr-01.infra.valuemart.com
          platform:
            napalm_driver: eos
          primary_ip4:
            host: 172.17.0.3
          role:
            name: Router
    New York City:
      hosts:
        nyc-acc-01.infra.valuemart.com:
          ansible_host: nyc-acc-01.infra.valuemart.com
          ansible_network_os: arista.eos.eos
          config_context:
            snmp:
              community:
              - name: networktocode
                role: ro
              - name: secure
                role: rw
          location:
            contact_name: ''
            description: ''
            name: New York City
            parent:
              name: East Coast
          name: nyc-acc-01.infra.valuemart.com
          platform:
            napalm_driver: eos
          role:
            name: Switch
        nyc-rtr-01.infra.valuemart.com:
          ansible_host: nyc-rtr-01.infra.valuemart.com
          ansible_network_os: arista.eos.eos
          config_context:
            snmp:
              community:
              - name: networktocode
                role: ro
              - name: secure
                role: rw
          location:
            contact_name: ''
            description: ''
            name: New York City
            parent:
              name: East Coast
          name: nyc-rtr-01.infra.valuemart.com
          platform:
            napalm_driver: eos
          role:
            name: Router
```

在更好地理解了 Nautobot 动态 Inventory 及其与 Ansible 的结合方式后，我们来创建一个 Ansible Playbook，将 SNMP 配置推送到波士顿的 Arista 设备。

创建新文件 `pb.snmp_update.yml` 并添加以下内容：
```yaml
---
- name: Configure SNMP settings on Arista vEOS
  hosts: Boston
  gather_facts: no
  connection: network_cli
  collections:
    - arista.eos

  tasks:
    - name: Configure SNMP Community strings
      become: true
      arista.eos.eos_snmp_server:
        config:
          communities:
            - name: "{{ item.name }}"
              ro: "{% if item.role == 'ro' %}True{% else %}False{% endif %}"
              rw: "{% if item.role == 'rw' %}True{% else %}False{% endif %}"
        state: merged
      loop: "{{ hostvars[inventory_hostname].config_context.snmp.community }}"
```

先检查一台波士顿设备，确认当前没有任何 SNMP 配置：
```bash
$ ssh bos-acc-01
Warning: Permanently added 'bos-acc-01' (ED25519) to the list of known hosts.
(admin@bos-acc-01) Password: 
Last login: Tue Feb 18 23:16:28 2025 from 172.17.0.1
ceos-01>en
ceos-01#sh run | i snmp
ceos-01#
ceos-01#
```

此任务使用 `config_context` 中的 SNMP 设置和 Arista EOS Ansible 模块，将配置应用到波士顿的路由器和交换机。运行以下命令执行 Playbook：
```bash
ansible-playbook pb.snmp_update.yml -i inventory.yml -u admin -k
```

运行 Playbook 后，终端应显示类似如下的输出：
```bash
$ ansible-playbook pb.snmp_update.yml -i inventory.yml -u admin -k
[DEPRECATION WARNING]: [defaults]collections_paths option, does not fit var naming standard, use the singular form collections_path instead. This feature will be removed from 
ansible-core in version 2.19. Deprecation warnings can be disabled by setting deprecation_warnings=False in ansible.cfg.
SSH password: 
 [ERROR]: Mapping ansible_host requires primary_ip6.host or primary_ip4.host as part of the query.
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details

PLAY [Configure SNMP settings on Arista vEOS] ***************************************************************************************************************************************

TASK [Configure SNMP Community strings] *********************************************************************************************************************************************
[WARNING]: ansible-pylibssh not installed, falling back to paramiko
changed: [bos-rtr-01.infra.valuemart.com] => (item={'name': 'networktocode', 'role': 'ro'})
changed: [bos-acc-01.infra.valuemart.com] => (item={'name': 'networktocode', 'role': 'ro'})
changed: [bos-rtr-01.infra.valuemart.com] => (item={'name': 'secure', 'role': 'rw'})
changed: [bos-acc-01.infra.valuemart.com] => (item={'name': 'secure', 'role': 'rw'})

PLAY RECAP **************************************************************************************************************************************************************************
bos-acc-01.infra.valuemart.com : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
bos-rtr-01.infra.valuemart.com : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```

Playbook 执行完成后，再次检查运行配置以确认变更已生效：
```bash
ceos-01#
ceos-01#sh run | i snmp
snmp-server community networktocode ro
snmp-server community secure rw
ceos-01#
```

## 总结

在本次挑战中，我们成功将 Nautobot 与 Ansible 集成，以 Nautobot 作为动态 Inventory 和唯一可信源来管理容器实验室设备。我们完成了环境配置，设置了包含组变量和主机变量的动态 Inventory，并利用 Nautobot 的 Config Context 存储自定义 SNMP 设置。最后，我们创建并运行了 Ansible Playbook，将 SNMP 配置应用到波士顿的 Arista 设备，并验证了运行配置中的变更。这一工作流程消除了对静态变量文件的依赖，简化了自动化操作，并将所有内容集中管理在 Nautobot 中。掌握了这些技能，您已准备好探索更强大的自动化可能性！

## 第 33 天待办事项

记得在 [https://github.com/codespaces/](https://github.com/codespaces/) 停止 Codespace 实例。

欢迎在社交媒体上发布新 Job 成功执行的截图，记得使用标签 `#100DaysOfNautobot` `#JobsToBeDone` 并 @ `@networktocode`，让我们一起分享您的进展！

在明天的挑战中，我们将通过构建未来站点来提升 Nautobot Job 技能。明天见！

[X/Twitter](<https://twitter.com/intent/tweet?url=https://github.com/nautobot/100-days-of-nautobot&text=I+just+completed+Day+33+of+the+100+days+of+nautobot+!&hashtags=100DaysOfNautobot,JobsToBeDone>)

[LinkedIn](https://www.linkedin.com/)（复制粘贴：I just completed Day 33 of 100 Days of Nautobot, https://github.com/nautobot/100-days-of-nautobot, challenge! @networktocode #JobsToBeDone #100DaysOfNautobot）
