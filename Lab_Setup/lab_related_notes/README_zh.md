# 实验室相关笔记

本文档记录了与实验室设置相关的各种笔记。

## 启动更大的实例

当 Nautobot 容器和 Containerlab 同时运行时，即使只有少数实验室节点，我们也容易耗尽 CPU 资源。在大多数情况下，你可以简单地选择一个更大的实例来获得更好的体验：

![bigger_instance_1](images/bigger_instance_1.png)

## 过滤 Containerlab 节点

为了节省资源，有时我们可能希望启动比拓扑图中描述的更少的 Containerlab 节点。我们可以注释掉节点或使用命令行选项 `--node-filter`：

```
$ sudo containerlab deploy --topo ceos-lab.clab.yml --node-filter bos-acc-01,bos-rtr-01
```

## Containerlab 启动、销毁和重新配置

销毁 Containerlab 拓扑是一个好习惯，一旦你完成使用它。

```
(deploy) # containerlab deploy --topo ceos-lab.clab.yml
(destroy) # containerlab destroy --topo ceos-lab.clab.yml
```

但如果你忘记销毁实验室，下次尝试启动实验室时会遇到类似下面的错误：

```
Error: containers ["bos-acc-01" "bos-rtr-01"] already exist. Add '--reconfigure' flag to the deploy command to first remove the containers and then deploy the lab
```

如错误所述，解决方法是使用 `reconfigure` 标志：

```
# sudo containerlab deploy --reconfigure --topo ceos-lab.clab.yml
```

## 重建 Codespace

我们发现有时在 Codespace 实例手动停止或由于预设的超时期限停止后，Docker 守护进程会停止工作。

例如，这个实例在 3 天前停止了，我通过 "open in browser" 重新启动：

![rebuild_codespace_1](images/rebuild_codespace_1.png)

在终端中，docker 守护进程似乎已停止：

```
@ericchou1 ➜ ~ $ docker ps
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

如果发生这种情况，我们可以通过进入 "Settings -> Command Pallette" 重建 codespace：

![rebuild_codespace_2](images/rebuild_codespace_2.png)

然后输入 "rebuild" 并选择 "Codespaces: Rebuild Containers"：

![rebuild_codespace_3](images/rebuild_codespace_3.png)

选择 "Rebuild" 并继续：

![rebuild_codespace_4](images/rebuild_codespace_4.png)

重新加载窗口后，Docker 守护进程将重新运行：

```
@ericchou1 ➜ ~ $ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

@ericchou1 ➜ ~ $ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
c1ec31eb5944: Pull complete 
Digest: sha256:5b3cc85e16e3058003c13b7821318369dad01dac3dbb877aac3c28182255c724
Status: Downloaded newer image for 'hello-world:latest'

Hello from Docker!
This message shows that your installation appears to be working correctly.

This message shows that your installation appears to be working correctly.
To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/

```

## Job 文件创建和权限

Nautobot Jobs 需要安装在 [Job 开发者指南](https://docs.nautobot.com/projects/core/en/stable/development/jobs/#installing-jobs) 中指定的 `JOBS_ROOT` 路径下。

在我们使用 `nautobot-docker-compose` 的实验室中，我们能够在 `/jobs` 文件夹下创建作业文件，因为我们将目录映射到 docker 容器中的 `JOBS_ROOT`。这在[第三天](https://github.com/nautobot/100-days-of-nautobot/tree/main/Day003_Hello_Jobs_Part_1) "创建 Job 文件" 部分的 "选项 1" 中有说明。

但是，值得注意的是，同一部分中的 "选项 2" 更接近生产 `nautobot` 环境，我们需要在 `nautobot` 环境中的 `JOBS_ROOT` 下创建作业文件，在本例中是 nautobot docker 容器。如示例所述，创建作业文件并更改权限后，你需要按照这个[图示](https://github.com/nautobot/100-days-of-nautobot/blob/main/Day003_Hello_Jobs_Part_1/images/docker_access_1.png)在 docker 容器文件目录下打开文件。


## 重命名 Codespace 实例

默认情况下，codespace 名称是随机分配的。它们可以在启动后通过点击 ```...``` 选项并选择 "Rename" 来重命名：

![rename_spaces](images/rename_spaces.png)

我发现根据场景重命名它们很有帮助，例如 "scenario_1" 和 "scenario_2"。

## 数据库导入

为了最大限度地减少设置步骤，开发容器比 [nautobot-docker-compose](https://github.com/nautobot/nautobot-docker-compose) 中列出的步骤多做了一些工作。

- 主动复制环境文件：

```
cp environments/local.example.env environments/local.env
cp environments/creds.example.env environments/creds.env
```

- 在目录中预加载了 `nautobot.sql` 文件。`invoke db-import` 命令会查找 `nautobot.sql` 文件作为要加载的数据库文件：

```
# invoke --list
Available tasks:

  ...
  db-export              Export the database from the dev environment to nautobot.sql.
  db-import              Install the backup of Nautobot db into development environment.
  ...
```

在 "Lab_Setup -> database_files" 下有其他可用于实验室的数据库文件。

## PostCreate.sh 文件

在 `devcontainer.json` 文件中，我们使用 `postCreateCommand` 来指定 Codespace 启动后要运行的命令，在本例中是一个 shell 脚本 `postCreate.sh`：

```
{
    "name": "Lab Scenario 1",
    ...
    "postCreateCommand": "bash /workspaces/100-days-of-nautobot/postCreate.sh",
    ...
}
```

目前，该脚本克隆仓库并将 README.md 文件从仓库复制到根目录：

```
#!/bin/bash
git clone https://github.com/nautobot/100-days-of-nautobot.git
cp 100-days-of-nautobot/README.md .
```

## 连接到设备

在[第九天](../../Day009_Python_Script_to_Jobs_Part_1/README.md)和[第十天](../../Day010_Python_Script_to_Jobs_Part_2/README.md)中，我们允许 Job worker 连接到 Containerlab 中的 `Arista EOS` 设备。为了让初次体验更顺畅，这背后有几个移动部件。如果你连接设备时遇到问题，请检查以下内容：

- 网络设备的 IP 前缀是否存在，如需要请添加。

![ip_prefix](images/ip_prefix.png)

- 设备的 IP 地址在前缀内是否存在，如需要请添加。

![ip_addresses](images/ip_addresses.png)

- 设备接口分配了正确的 IP 地址，例如，`bos-acc-01` 的 `Management1` 接口应该在我的实验室中分配 `172.16.0.2/16`：

![bos-acc-01-management1_1](images/bos-acc-01-management1_1.png)
![bos-acc-01-management1_2](images/bos-acc-01-management1_2.png)

- 最后但同样重要的是，检查是否为平台指定了网络驱动程序。例如，Arista EOS 驱动程序应指定为 `arista_eos`：

![arista_eos_network_driver](images/arista_eos_network_driver.png)
