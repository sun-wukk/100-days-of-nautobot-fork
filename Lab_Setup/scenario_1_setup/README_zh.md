# 实验场景 1

实验场景 1 是我们在[前言](../../Day000_Preamble/README.md)中介绍的基础场景，设置步骤在[第一天说明](../../Day001_Development_Setup/README.md)中。

为了更好地说明，我们将在下面总结开发环境的设置。如需更详细的说明，请参阅[第一天说明](../../Day001_Development_Setup/README.md)。

## 启动 Codespace

如果你对 GitHub Codespace 完全陌生，可以观看下面的设置视频：

[视频：设置您的 100 天 Nautobot 开发环境](https://www.youtube.com/watch?v=gW3Qq0yssLE)

按照以下步骤启动 Codespace：

1. 点击绿色的 Code 按钮
2. 选择 Codespaces
3. 点击 "..." 选项
4. 选择 "New with Options"

![Codespace_Screenshot_1.png](../../Day001_Development_Setup/images/Codespace_Screenshot_1.png)

在下一个屏幕上，点击 "Dev container configuration" 的下拉菜单，选择 "Lab Scenario 1"，然后点击 "Create Codespace"：

> [!提示]
> 首次启动 Codespace 时，它可能会提示你选择基于浏览器的 Visual Studio Code 或桌面版本，选择基于浏览器的版本以与截图保持一致，但如果你愿意，也可以选择桌面版本。

![Codespace_Screenshot_2.png](../../Day001_Development_Setup/images/Codespace_Screenshot_2.png)

Codespace 将开始启动，可以点击 "Building Codespace" 查看创建日志并监控进度：

![Codespace_Screenshot_3.png](../../Day001_Development_Setup/images/Codespace_Screenshot_3.png)

Codespace 完成设置后，你将拥有一个基于浏览器的开发环境，包含以下部分：

1. 资源管理器窗口：在这里你可以选择不同的文件，即 Day001、Day002 等文件夹。
2. 终端窗口：也会有一个可以与 Codespace 交互的终端窗口。
3. 在资源管理器窗口中，展开 "100-Days-of-nautobot-challenge" 文件夹和 "Day001_Development_Setup" 子文件夹，右键点击 ```README.md``` 文件并选择 "Open Preview"：

![Codespace_Screenshot_4.png](../../Day001_Development_Setup/images/Codespace_Screenshot_4.png)

4. 我们会经常使用终端窗口，有时会同时打开很多终端窗口。```+``` 是我们可以添加更多终端窗口的地方。

![Codespace_Screenshot_5.png](../../Day001_Development_Setup/images/Codespace_Screenshot_5.png)

继续启动 Codespace，启动后，这个仓库将被克隆到环境中。

## 启动 Nautobot 和必要的组件

Codespace 中包含来自 [nautobot-docker-compose](https://github.com/nautobot/nautobot-docker-compose/) 仓库的代码。我们的 Codespace 启用了 docker-in-docker 功能，允许我们在容器中运行 Nautobot 以及必要的组件。

以下指令将在终端窗口中输入。

- 切换到 nautobot docker-compose 代码所在目录：
```
@ericchou1 ➜ ~ $ cd nautobot-docker-compose/
```
- 我们已经安装了 [poetry](https://python-poetry.org/) 虚拟环境，所以只需要启用环境：

```
@ericchou1 ➜ ~/nautobot-docker-compose (main) $ poetry shell
Spawning shell within /home/vscode/.cache/pypoetry/virtualenvs/nautobot-docker-compose-70lkLMMl-py3.10
@ericchou1 ➜ ~/nautobot-docker-compose (main) $ . /home/vscode/.cache/pypoetry/virtualenvs/nautobot-docker-compose-70lkLMMl-py3.10/bin/activate
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $
```

- 我们将使用 [Invoke](https://www.pyinvoke.org/) 作为面向 shell 的子进程命令行任务工具。

> [!重要]
> 下面的步骤展示了如何从头开始构建环境；如果你从之前构建的、已经导入了数据库的 docker-compose 环境重新启动，只需使用 `invoke debug` 启动 nautobot 容器。

- 第一步是构建 docker 镜像，如果你第一次构建，需要一些时间，确保等到最后看到 "DONE" 消息并返回用户终端提示符。

```
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke build
Building Nautobot 2.3.2 with Python 3.8...
Running docker compose command "build"
#0 building with "default" instance using docker driver

#1 [nautobot internal] load build definition from Dockerfile
#1 transferring dockerfile: 2.32kB done
#1 WARN: FromAsCasing: 'as' and 'FROM' keywords' casing do not match (line 3)
#1 WARN: FromAsCasing: 'as' and 'FROM' keywords' casing do not match (line 14)
#1 WARN: FromAsCasing: 'as' and 'FROM' keywords' casing do not match (line 54)
#1 DONE 0.0s
...
<skip>
...
#2 [nautobot auth] nautobot/nautobot-dev:pull token for ghcr.io
#2 DONE 0.0s
#24 [nautobot] resolving provenance for metadata file
#24 DONE 0.0s
```

- 现在我们可以导入初始数据集：

```
$ source db_import_prep.sh
$ invoke db-import
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke db-import
Importing Database into Development...

Starting Postgres for DB import...

Running docker compose command "up -d db"
 db Pulling
 43c4264eed91 Pulling fs layer
...
<skip>
...
 55d20525b40e Waiting
 82052d0672a9 Downloading [====================================>              ]     720B/984B
 82052d0672a9 Downloading [==================================================>]     984B/984B
 82052d0672a9 Download complete
 ...
 <skip>
 ...
 Network nautobot_docker_compose_default  Creating
 Network nautobot_docker_compose_default  Created
...
<skip>
...
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
```

- 现在我们可以使用 ```invoke debug``` 启动 nautobot 容器。这将以调试模式启动 Nautobot 并在屏幕上显示所有消息：

> [!提示]
> 等到看到 ```Starting development server at http://0.0.0.0:8080/``` 的消息后再进行下一步：

```
(nautobot-docker-compose-py3.10) @ericchou1 ➜ ~/nautobot-docker-compose (main) $ invoke debug
Starting Nautobot in debug mode...
Running docker compose command "up"
 redis Pulling
 43c4264eed91 Already exists
 54346cffc29b Pulling fs layer
 2866ca214a5e Pulling fs layer
 ee16541feddb Pulling fs layer
 d14ed515876d Pulling fs layer
 cf7b98d3ba3c Pulling fs layer
 4f4fb700ef54 Pulling fs layer
 c4e0a3f69d20 Pulling fs layer
 ...
 <skip>
 ...
nautobot-1       | October 16, 2024 - 20:18:55
nautobot-1       | Django version 4.2.16, using settings 'nautobot_config'
nautobot-1       | Starting development server at http://0.0.0.0:8080/
nautobot-1       | Quit the server with CONTROL-C.
nautobot-1       |
```

Nautobot 启动后，我们可以转到转发端口，将鼠标悬停在 "Forwarded Address" 上，点击地球图标在新浏览器窗口中打开：

![Codespace_Screenshot_6.png](../../Day001_Development_Setup/images/Codespace_Screenshot_6.png)

新窗口将指向转发端口，可以访问 Nautobot UI。默认登录凭据是用户名 ```admin``` 和密码 ```admin```，这是我们在初始数据集中包含的管理员用户：

![Codespace_Screenshot_7.png](../../Day001_Development_Setup/images/Codespace_Screenshot_7.png)

现在我们在 Codespace 中有了一个可用的 Nautobot 实例。让我们回到终端窗口，使用 ```Ctl+C``` 终止 Nautobot 实例：

```
...
...
nautobot-1       | 20:25:48.342 INFO    django.server :
nautobot-1       |   "GET /static/img/favicon.ico?version=2.3.2 HTTP/1.1" 200 15086
redis-1          | 1:M 16 Oct 2024 20:33:15.250 * 100 changes in 300 seconds. Saving...
redis-1          | 1:M 16 Oct 2024 20:33:15.251 * Background saving started by pid 14
redis-1          | 14:C 16 Oct 2024 20:33:15.253 * DB saved on disk
redis-1          | 14:C 16 Oct 2024 20:33:15.254 * RDB: 0 MB of memory used by copy-on-write
redis-1          | 1:M 16 Oct 2024 20:33:15.351 * Background saving terminated with success
Gracefully stopping... (press Ctrl+C again to force)
 Container nautobot_docker_compose-celery_worker-1  Stopping
 Container nautobot_docker_compose-celery_beat-1  Stopping
 Container nautobot_docker_compose-celery_beat-1  Stopped
 Container nautobot_docker_compose-celery_worker-1  Stopped
 Container nautobot_docker_compose-nautobot-1  Stopping
 Container nautobot_docker_compose-nautobot-1  Stopped
 Container nautobot_docker_compose-db-1  Stopping
 Container nautobot_docker_compose-redis-1  Stopping
 Container nautobot_docker_compose-redis-1  Stopped
 Container nautobot_docker_compose-db-1  Stopped
canceled
```

## 启动 Containerlab（可选）

[Containerlab](https://containerlab.dev/) 是一个基于 docker 的网络实验室环境，我们将用它来测试网络设备。

网络实验室不是所有挑战都需要的，但这里提供了必要的信息以备需要时使用。如需更详细的说明，请参阅[实验 9 说明](../../Day009_Python_Script_to_Jobs_Part_1/README.md)。

Containerlab 已安装在我们的环境中，我们可以检查已安装的版本：

```
@ericchou1 ➜ ~ $ containerlab version
  ____ ___  _   _ _____  _    ___ _   _ _____ ____  _       _
 / ___/ _ \| \ | |_   _|/ \  |_ _| \ | | ____|  _ \| | __ _| |__  
| |  | | | |  \| | | | / _ \  | ||  \| |  _| | |_) | |/ _` | '_ \
| |__| |_| | |\  | | |/ ___ \ | || |\  | |___|  _ <| | (_| | |_) |
 \____\___/|_| \_| |_/_/   \_\___|_| \_|_____|_| \_\_|\__,_|_.__/ 

    version: 0.57.3
     commit: 8c357f5a
       date: 2024-09-21T12:26:37Z
     source: https://github.com/srl-labs/containerlab
 rel. notes: https://containerlab.dev/rn/0.57/#0573
```

我们在 ```100-days-of-nautobot-challenge/clab``` 下提供了一个实验室拓扑，可以看一下：

```
$ cd 100-days-of-nautobot-challenge/
$ cd clab/
$ cat ceos-lab.clab.yml 
---
name: "ceos-lab"
prefix: ""

mgmt:
  # network: "network-lab"
  # ipv4-subnet: "172.24.78.0/24"
  network: "bridge"

topology:
  kinds:
    ceos:
      image: "ceos:4.32.0F"

  nodes:
    bos-acc-01:
      kind: "ceos"
      startup-config: "startup-configs/bos-acc-01.conf"

    bos-rtr-01:
      kind: "ceos"
      startup-config: "startup-configs/bos-rtr-01.conf"

    nyc-acc-01:
      kind: "ceos"
      startup-config: "startup-configs/nyc-acc-01.conf"

    nyc-rtr-01:
      kind: "ceos"
      startup-config: "startup-configs/nyc-rtr-01.conf"

  links:
    - endpoints: ["bos-acc-01:eth1", "bos-rtr-01:eth1"]
    - endpoints: ["bos-acc-01:eth2", "nyc-rtr-01:eth2"]
    - endpoints: ["bos-rtr-01:eth2", "nyc-acc-01:eth2"]
    - endpoints: ["nyc-acc-01:eth1", "nyc-rtr-01:eth1"]
```

我们需要按照拓扑文件中的指示上传 cEOS 镜像。

## 下载并上传 cEOS 镜像

[Arista Networks](https://www.arista.com/en/) 提供免费的容器化 EOS 系统下载，可与 Containerlab 一起使用。注册是免费的，但需要使用商务邮箱地址。

注册后，我们可以通过 "Support -> Software Download" 下载镜像：

![arista_1](../../Day009_Python_Script_to_Jobs_Part_1/images/arista_1.png)

我们应该选择 "cEOS Lab" 软件镜像之一，截图示例中显示的是 ```cEOS64-lab-4-32.0F.tar.xz```：

![arista_2](../../Day009_Python_Script_to_Jobs_Part_1/images/arista_2.png)

下载完成后，我们可以在资源管理器区域右键点击，选择上传镜像：

![arista_3](../../Day009_Python_Script_to_Jobs_Part_1/images/arista_3.png)

根据网络速度，上传可能需要一到两分钟。镜像上传后，可以使用以下命令将其导入 docker：

> [!重要]
> 请记住将版本和位置替换为下载的版本和文件位置。

```
$ docker import ../Lab_Setup/cEOS64-lab-4.32.0F.tar ceos:4.32.0F

sha256:ff28abebb338b16656c0c86e01940e97a3b26de4b6c66873daebdb941cd4f4e2
```

我们可以在下一步启动实验室。

## 启动 Containerlab

我们可以使用有限数量的设备启动实验室，例如，只使用波士顿设备通过 ``--node-filter 选项``：

```
$ sudo containerlab deploy --topo ceos-lab.clab.yml --node-filter bos-acc-01,bos-rtr-01
INFO[0000] Containerlab v0.57.3 started                 
INFO[0000] Applying node filter: ["bos-acc-01" "bos-rtr-01"] 
INFO[0000] Parsing & checking topology file: ceos-lab.clab.yml 
WARN[0000] Unable to init module loader: stat /lib/modules/6.5.0-1025-azure/modules.dep: no such file or directory. Skipping... 
INFO[0000] Creating lab directory: /home/vscode/100-days-of-nautobot-challenge/clab/clab-ceos-lab 
INFO[0000] Creating container: "bos-acc-01"             
INFO[0000] Creating container: "bos-rtr-01"             
INFO[0001] Running postdeploy actions for Arista cEOS 'bos-rtr-01' node 
INFO[0001] Created link: bos-acc-01:eth1 <--> bos-rtr-01:eth1 
INFO[0001] Running postdeploy actions for Arista cEOS 'bos-acc-01' node 
...
INFO[0093] 🎉 New containerlab version 0.60.1 is available! Release notes: https://containerlab.dev/rn/0.60/#0601
Run 'containerlab version upgrade' to upgrade or go check other installation options at https://containerlab.dev/install/ 
+---+------------+--------------+--------------+------+---------+---------------+--------------+
| # |    Name    | Container ID |    Image     | Kind |  State  | IPv4 Address  | IPv6 Address |
+---+------------+--------------+--------------+------+---------+---------------+--------------+
| 1 | bos-acc-01 | 575cba7b555b | ceos:4.32.0F | ceos | running | 172.17.0.3/16 | N/A          |
| 2 | bos-rtr-01 | 940e991b876a | ceos:4.32.0F | ceos | running | 172.17.0.2/16 | N/A          |
+---+------------+--------------+--------------+------+---------+---------------+--------------+
```

网络设备的默认用户名和密码都是 ```admin```：

```
$ ssh admin@172.17.0.3
The authenticity of host '172.17.0.3 (172.17.0.3)' can't be established.
ED25519 key fingerprint is SHA256:aXFI/vMIdoc3mdWegaZATVj6uUmjUYtJHRnB1PiSSCs.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '172.17.0.3' (ED25519) to the list of known hosts.
(admin@172.17.0.3) Password: 
ceos-01>sh ver
Arista cEOSLab
Hardware version: 
Serial number: CECCE43E5AAA6355D646272A9F91AE27
Hardware MAC address: 001c.73a0.2d78
System MAC address: 001c.73a0.2d78

Software image version: 4.32.0F-36401836.4320F (engineering build)
Architecture: x86_64
Internal build version: 4.32.0F-36401836.4320F
Internal build ID: e97bbe15-478c-45d1-84fa-332db23aef84
Image format version: 1.0
Image optimization: None

cEOS tools version: (unknown)
Kernel version: 6.5.0-1025-azure

Uptime: 4 minutes
Total memory: 8119864 kB
Free memory: 2728160 kB

ceos-01>exit
Connection to 172.17.0.3 closed.
```

我们可以继续进行需要网络设备测试的挑战。

完成后，可以使用 ```destroy``` 命令关闭实验室：

```
$ sudo containerlab destroy --topo ceos-lab.clab.yml --node-filter bos-acc-01,bos-rtr-01
INFO[0000] Applying node filter: ["bos-acc-01" "bos-rtr-01"] 
INFO[0000] Parsing & checking topology file: ceos-lab.clab.yml 
INFO[0000] Destroying lab: ceos-lab                     
INFO[0002] Removed container: bos-rtr-01                
INFO[0002] Removed container: bos-acc-01                
INFO[0002] Removing containerlab host entries from /etc/hosts file 
INFO[0002] Removing ssh config for containerlab nodes   
```

完成手头的挑战后，我们可以停止或删除 Codespace 实例。

## 停止或删除 Codespace

让我们停止 Codespace，因为我们不想在不使用时产生不必要的费用。我们将导航到你的 [Codespace](https://github.com/codespaces) 设置并停止 Codespace：

![Codespace_Screenshot_8.png]../../Day001_Development_Setup/(images/Codespace_Screenshot_8.png)

> [!提示]
> 你也可以选择删除 Codespace，但如果你这样做了，就需要重复本课程中的步骤。

恭喜，我们现在准备好继续基于实验场景 1 的课程了！
