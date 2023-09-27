# 容器技术基础

在了解Docker的过程中，*容器*是一个避不开的话题。容器泛指任何可以用于容纳其他物品的工具，可以部分或完全封闭，被用于容纳、存储、传输物品，容器可以保护内容物品。容器本身被人类使用已经很长的历史了，其本身并不是一个新概念，而在Linux中，对于容器的了解需要从LXC开始（Linux Container）

## LXC

在学习LXC之前，需要先明确*虚拟化*与*容器*的关系，虚拟化目前主流的实现形式分为*主机级虚拟化*和*容器级虚拟化*

### 主机级虚拟化技术

主机级虚拟化需要模拟整个完整的硬件平台，在主机级虚拟化下又分为以下两种实现类型

- type-Ⅰ：直接在物理机上安装一个虚拟机管理器（Hypervisor） ---- esxi
- type-Ⅱ：在物理机在安装宿主机系统（Host OS），再在宿主机上那幢虚拟机管理器（VMM），在VMM上再创建虚拟机 ---- VMware，virtualbox

无论是哪种实现类型，主机级虚拟化的实现机制都是向用户提供一个完整的硬件平台，用户需要自己安装操作系统、应用程序。传统的虚拟化技术实现了资源的隔离使用，相同的业务应用可以运行在每一个虚拟机内，对于虚拟机而言，这个应用就是唯一的，在宿主机上也不会产生冲突，而实现这些功能的代价就是对宿主机的资源开销

[![虚拟化与容器的关系](https://s1.ax1x.com/2023/01/20/pSGMsxA.png)](https://imgse.com/i/pSGMsxA)

但是，当用户只有少量业务或只有单一业务需要部署时，虚拟化无疑就显得复杂、臃肿了。总所周知，实际产生价值的只有用户空间的业务应用，但用户空间又依赖内核环境。而虚拟化的业务应用运行更是需要经过二级调度，也就是需要经过“两层OS”，这对宿主机资源也会产生额外的损耗

那么现在需要追求的环境就是，既需要对用户空间进行隔离避免业务冲突、又需要缩减虚拟化结构的中间层以提高效率。如上图示例，缩减中间层后启用进程时，让进程启动运行在用户空间中，众多用户空间底层都是被同一个内核管理的，但是，进程运行时能够看到的边界，是自身所处用户空间的边界；综上所述，**每一个用户空间都被用于存放进程，为其提供运行环境，并对其进行隔离保护**，其本质就是容器

显然，用户空间级别隔离没有主机级别隔离的彻底，而且无论怎么对用户环境进行隔离，也一定会有一个特殊的用户空间，用于管理其他用户空间

### 容器级虚拟化技术

```shell
UTS：主机名域名         隔离内容：主机名和域名
IPC：进程通信专用通道    隔离内容：信息量、消息对列和共享内存
PID：用户进程           隔离内容：进程编号
Net：TCP/IP栈           隔离内容：网络设备、网络栈、端口等
Mount：文件系统         隔离内容：挂载点
User：用户ID            隔离：用户和用户组
```

内核在最初设计时只是为了支撑单个用户空间的运行，所以在内核级UTS、IPC、PID、Net、Mount、User这6组资源都是独立且只有1组，但是后来有需要运行jail、vserver（chroot）这种早期容器，所以6组资源开始能够在内核级直接切分为多个互相隔离的环境，这个环境就是名称空间。事实上，每一种资源只要在内核级能够直接切分为多个互相隔离的环境，都可以称为名称空间

在内核中，UTS是可以以名称空间为单位进行隔离的，这就意味着，可以在一个内核上创建出多个名称空间，每一个名称空间的UTS资源都相互隔离，所以每一个名称空间都可以有自己独有的主机名、域名；主机名、域名本身是内核级的，所以一台主机上只能有一个主机名，但是现在每一个名称空间都可以有自己的主机名，各个名称空间之间互不干扰，由于内核级的资源切分与隔离，这些名称空间也不会对真正的宿主机名称空间产生影响

为了支撑容器级技术的实现，Linux内核已经通过namespaces机制实现了这6种资源的隔离，也就是说内核级原生支持这6种资源的隔离；但最晚的User资源在内核3.8版本才开始被支持，所以CentOS6天然的就被排除在外了，这并不是说CentOS6就无法使用容器技术，只是在功能上会有缺失

在主机级虚拟化技术的基础上，拔除GuestOS内核，使每个GuestOS上的用户空间都从属于HostOS内核，这种技术就是容器级虚拟化

### CGroups（Control Groups）

解决资源隔离后，容器级虚拟化仍存在一个问题，使用主机级虚拟化，在创建虚拟机的时候就能够限制该虚拟机可使用的最大硬件资源，例如给该虚拟机分配几个CPU核心、多大内存。而容器级虚拟化，名称空间共用HostOS内核，那么如果某一个名称空间内的进程出现BUG，开始吞噬硬件资源，此时就需要内核级再实现一个功能，限制每一个用户空间中的进程的所有可用资源的总量，这个功能在内核级依靠CGroups机制实现

对于CGroups而言，它将系统资源分为多个组，然后将每个组内的资源量分配到各个用户空间的进程上

```shell
blkio：块设备 IO
cpu：CPU
cpuacct：CPU 资源使用报告
cpuset：多处理器平台上得 CPU 集合
devices：设备访问
freezer：挂起或恢复任务
memory：内存用量及报告
perf_event：对 cgroup 中的任务进行同一性能测试
net_cls：cgroup 中的任务创建的数据报文的类别标识符
```

可压缩型资源：例：CPU -- 进程需要 CPU 时如果没有，排列等待即可（挂起）

非可压缩性资源：例：内存 -- 进程需要内存时如果没有，宕机

### LXC（LinuX Container）

早期使用容器技术需要用户自己写代码系统调用，为了简化用户的使用，后来将需要使用容器技术的功能做成了一组工具，于是就产生了一个解决方案LXC，LXC是最早一批把完整的容器技术用一组简易使用的工具和模板实现的方案

LXC可以通过一个命令快速创建一个容器（用户空间），那么这个用户空间内就应该具备最基本的Linux目录结构、基本的应用程序，而创建这些基本环境就需要借助*模板（template）*，template的本质就是一组脚本，LXC工具快速创建一个名称空间实际上也就是给了模板一个执行环境，名称空间创建好之后模板会自动执行实现*安装过程*，这个安装过程实际上也就是模板中会指向将要创建的一类名称空间的系统发行版所属的仓库，从仓库中将各应用程序包下载到本地安装，生成一个新的名称空间，再使用chroot工具进入容器

**template模板的工作流程**

1. LXC工具创建一个名称空间，给了脚本一个执行环境
2. 容器创建好后自动执行模板（脚本）
3. 执行模板的过程中自动实现安装过程
4. 使用 chroot 工具进入容器

使用LXC工具快速创建的名称空间与VMware的虚拟机几乎无差别，但使用LXC依然存在一些问题，LXC对比虚拟机的优势在于，LXC能够让每一个用户空间中的进程直接使用宿主机的性能，中间基本没有额外开销，降低了宿主机的资源开销；虽然LXC工具简化了容器技术的使用，但是LXC比起虚拟机，其复杂程度并没有多少降低，隔离性也没有虚拟机好，除此以外还存在学习成本和使用成本的问题

1. 理解学习各种 LXC 的工具使用
2. 必要时需要自己定制模板
3. 数据迁移不便
4. **批量创建容器不便**

## Docker

由于LXC在大规模使用上仍然没有一个比较好的突破口，于是就产生了Docker，Docker更像是LXC的增强版，其本身也不是容器，而是容器的应用工具，容器是Linux内核的技术，Docker只是将容器技术的使用简化、普及到用户

早期的Docker就是LXC的二次封装，使用LXC作为容器管理引擎，只不过在创建容器时，不再使用模板安装操作系统和应用程序，而是使用镜像技术安装容器环境；镜像技术是指将一个操作系统用户空间所需要用到的所有组件事先编排好后，整体打包成一个文件，称之为镜像文件，镜像文件放在集中统一的一个仓库中，在互联网上有一个公共仓库*dockerhub.com*

早期的Docker操作容器时还是依赖LXC工具进行管理，但Docker创建容器时不会调用模板进行安装，而是连接到镜像仓库，下载匹配创建容器时需要的镜像，并基于此镜像创建容器

LXC是将一个容器当作一个用户空间进行使用，它可以像虚拟机一样，一个容器内可以运行多个进程，这就使得在LXC创建的容器内的管理几乎与虚拟机无差异。而Docker为了使容器更加易于管理，其理念是在一个容器内只运行一个主进程；那么，当容器内的进程出现故障时，LXC容器内的维护工具可以对一个用户空间内的多个进程进行追踪排查。而Docker，维护工具在哪个名称空间中，就只能对那个名称空间中的进程做调试，这就意味着每个Docker容器都需要自带维护工具，维护工具会以子进程的方式存在容器中，平时不会启动

Docker为开发带来了便利，却增加了运维的复杂度

1. 每个容器都是单独的进程，删除与否不影响其他进程
2. 极大降低了软件开发的成本和难度，真正实现了一次编译、到处运行
3. 实现批量创建容器
4. 每启动一个进程就需要新建一个容器，容器内所有的环境都是为进程准备，这也就占用了更多的空间
5. 不利于运维调试
6. 增加了运维环境管理的复杂度

Docker容器本身与主机并没有绑定关系，当容器的数据都挂载到外部存储上时，那么这个容器究竟存在于那个主机上就不重要了，Docker容器的本质就是一组镜像，只要数据还在，那么在任意一台主机上都可以以镜像的方式重启该容器。这种方式适用于单台主机的场景，而涉及到多台主机或集群的场景，就需要使用容器编排工具，常见的编排工具组合：

- machine+swarm+compose
- mesos+marathon
- kubernetes

docker 自主研发的容器引擎：libcontainer --> runC








# Docker基本用法

由上一小节可知，要想使用Linux容器，在内核级至少支持两种技术：namespaces和CGroups，然后再借助用户空间的一些工具，利用内核级所提供的技术，从而实现运行容器。而Docker通过镜像技术在简化容器运行的道路上更进一步，在Docker的主导下产生了OCI（Open Container Initiative）和OCF（Open Container Format）的标准

OCI的主要目的在于围绕容器格式和运行时制定一个开放的工业化标准，OCI的标准由2部分组成，runtime-spec（容器运行时标准）和image-spec（镜像格式标准），这是两种不同标准。再后来就产生了OCF格式，runC是OCF的重要实现之一，runC也是较新版本中Docker所使用的容器引擎

Docker专门提供了一个容纳容器镜像的站点[dockerhub](hub.docker.com)，使用Docker拉取或运行一个容器，默认都会从这个镜像仓库下载镜像

## Docker的架构

![Docker的架构](https://www.z4a.net/images/2023/02/01/Docker.png)

DOCKER_HOST实际上也就是Server端，整个Docker就是一个C/S架构的应用程序。无论是Client端或Server端，都由Docker这一个程序提供，同时Docker下面有很多子程序，其中一个子程序叫daemon，当宿主机运行Docker daemon程序时，就表示Docker在该主机上运行为守护进程

守护进程可以监听在某个Socket套接字上，Docker支持3种类型的Socket套接字：IPv4 Socket、IPv6 Socket、Unix Socket。处于安全考虑，Docker默认只提供本机的Unix Socket文件套接字，Unix Socket是本地文件套接字方式，它生成一个`/var/run/docker.sock`文件，用于本地进程之间的通信，本地套接字相比较网络套接字效率更高

DOCKER_HOST是真正运行容器的主机，在DOCKER_HOST上有两个非常重要的组成部分Containers（容器）和images（镜像），其中images来自Registry（镜像仓库）。Docker默认的Registry就是[dockerhub](hub.docker.com)，宿主机本地默认是没有镜像的，需要从Registry下载镜像

启动容器时基于本地镜像启动，这意味镜像也要在宿主机本地存储，但实际使用Docker时到底会运行什么容器事先无法评估，因此Registry用于专门存放海量镜像文件，需要用到什么容器，直接从Registry下载镜像到本地即可。下载镜像使用的协议是http或https，**默认必须使用https协议**

**小结**：所谓Docker_Host就是运行有Docker Daemon的主机，也是Docker Server端，接收Client端请求是通过http或https协议与Client交互，Client可以是远程主机，Docker Deamon收到创建或启动容器的命令后将在本地创建容器，一个Docker Daemon上可以创建多个容器，每个容器可能都运行不同的应用程序，而容器的启动需要基于镜像实现，如果Docker Daemon本地没有镜像，则会自动连接到Registry，从Registry获取镜像后存储在本地，本地用于存储镜像的空间必须是专用的文件系统（overlay2）

### 什么是Unix Socket

Unix Socket可以让一个程序通过类似处理一个文件的方式和另一个程序通信，这是一种进程间通信的方式（IPC）。当你在host上安装并且启动好docker，docker daemon会自动创建一个socket文件并且保存到`/var/run/docker.sock`路径，docker daemon监听着socket中即将到来的链接请求，当一个链接请求到来时，它会使用标准IO来读写数据

docker.sock是docker client和docker daemon在宿主机上进行通信的socket文件。可以直接call这个socket文件来拉取镜像、创建容器、启动容器等一系列操作。其实就是直接call docker daemon API，而不是通过docker client的方式去操控docker daemon

### TCP端口监听

Docker服务端开启端口监听`dockerd -H IP:PORT`, 客户端通过指定IP和端口访问服务端，这种方式就是网络套接字，客户端连接服务端使用的是http或https协议。通过这种方式, 任何人只要知道了你暴露的ip和端口就能随意访问你的docker服务了, 这是一件很危险的事, 因为docker的权限很高, 不法分子可以从这突破取得服务端宿主机的最高权限

### Registry

一个Registry至少具备2个功能：*镜像存储仓库*和*用户认证*，一般还会提供镜像索引功能，这个功能由另外的应用程序实现。一个Registry上又有很多“仓库”，Registry上的“仓库”又被称为repository，*通常一个repository只存放一个应用程序的镜像，且该repository仓库的名称就是应用程序的名称*

以nginx为例，nginx应用程序以镜像文件的方式存放在Registry的一个repository中，这个repository的名称就叫nginx，而nginx应用程序自身也存在不同的版本，这些不同版本的镜像文件统一都存放在nginx的repository仓库下，Docker通过tag（标签）来标识不同版本的nginx应用程序，下载镜像时如果未指明tag，默认会使用latest标签的镜像，代表最新版镜像

### 容器与镜像的关系

容器是动态的，它有自己的生命周期，镜像是静态的，一个镜像可以启动多个容器；两者的关系类似于Linux上的进程与程序之间的关系，程序本身是静态的，程序被执行时就产生了进程，每个进程都具备自己的生命周期，进程的启动依赖程序，正如容器的启动依赖镜像

### Docker安装

![Docker安装-1](https://www.z4a.net/images/2023/02/01/Docker-1.png)

![Docker安装-2](https://www.z4a.net/images/2023/02/01/Docker-2.png)

![Docker安装-3](https://www.z4a.net/images/2023/02/01/Docker-3.png)

![Docker安装-4](https://www.z4a.net/images/2023/02/01/Docker-4.png)

按照官网步骤安装即可。Docker安装完毕不需要经过任何修改就可以启动Docker服务，Docker的配置文件默认是不存在的，需要手动创建，实际上也就是配置镜像加速的过程。Docker镜像加速除了docker-cn，还可以使用阿里云和中科大提供的镜像加速服务。从示例的daemon.json文件中可以看出json语法是一个数组，这意味着可以通过逗号隔开添加多个镜像加速地址

```shell
vim /etc/docker/daemon.json
{
  "registry-mirrors":["https://registry.docker-cn.com"]
}
```

### Docker基本命令

1. 查看Docker的基本信息

```shell
docker version      # 查看Client和Server的版本信息和API版本
docker info     # 查看Docker更详细的环境信息。可以看到配置的registry信息
```

2. Docker常用命令

早期Docker与其子命令是直接使用的，随着版本的更新，Docker将部分子命令进行了分组，更便于容器的管理，为了兼容命令的用法，现在直接使用子命令或通过分组使用子命令都是可行的，但建议使用分组命令进行管理

```shell
docker search nginx     # 直接使用子命令查找nginx镜像
docker image pull nginx:1.14-alpine     # 使用分组image命令下载nginx镜像
docker pull busybox:latest      # 直接使用pull命令也能够下载镜像
docker image ls     # 查看本地的镜像
```

使用`docker search nginx`可以看到查出了很多nginx镜像仓库，但真正以nginx命名的只有一个镜像仓库，这个仓库被称为*顶级仓库*，其他所有带有`/`分隔符的仓库，例如`linuxserver/nginx`，都表示用户仓库。顶级仓库是Docker官方的，用户仓库只要注册就能够创建

3. 创建docker容器

Docker的理念是在一个容器内只运行一个主进程，而镜像作为启动容器的基石，每个镜像本身存在一个默认定义的要执行的程序，将镜像启动为容器时，如果没有指定要执行的程序，则启动镜像默认的程序；docker创建容器的子命令是create，但更为常见的子命令是run，`docker container run`能够直接创建并启动容器

```shell
docker container run -it --rm -d --name first_container --network bridge 
    -i：--interactive，交互式访问
    -t：--tty，分配一个伪终端，-it选项一般联用
    -d：--detach，后台运行
    --rm：停止容器时自动删除容器
    --name：为容器命名
    --network：指定容器网络
docker network ls   # 查看网络模型列表
```

创建容器时若未手动指定网络模型，docker默认会使用bridge桥接网络模型，桥接的网卡是docker0，docker0在安装docker时自动生成；各个容器间能够通过桥接网络模型进行互相通信，类似KVM的桥接网络，虚拟机之间能够通过桥接网络互相访问

```shell
# busybox本身也是一个工具集，它能够简单实现http服务
docker container exec -it first_container /bin/sh  # 进入创建的容器
/ # mkdir /data/html -p
/ # echo "busybox httpd server" > /data/html/index.html
/ # httpd -f -h /data/html/ # 将httpd程序运行在前台，并指定站点目录
docker container inspect first_container    # 查看容器的详细信息，可以看到容器的IP
curl 172.17.0.2 # 宿主机访问容器
```

进入容器后，通过ps命令查看当前进程可以看到，容器内PID为1的进程是sh，这也就是busybox镜像默认执行的主程序，通过exec进入容器后执行ps命令，可以看到除了默认的sh进程以外，还有一个/bin/sh进程，这是exec执行的命令；由于busybox镜像默认执行的/bin/sh进程，当使用exit退出容器时，该容器会停止运行，可以重新启动该容器，如果容器内运行的主进程是/bin/sh进程，使用`container start -ia`启动进程时能够直接进入容器

```shell
docker container run --name web1 -d nginx:1.14-alpine
docker container ls
```

通过ls命令可以看到nginx容器默认执行的程序是`nginx -g 'daemon off'`，该程序的意思是不执行在后台。这是因为任何执行在容器中的程序都不能在后台执行，否则会被docker认定为进程已停止，继而结束这个容器，这就会造成容器运行即停止的情况。**一个容器的主程序就是这个容器的骨架，如果主程序运行在后台，那么该容器将无意义**

此处概念容易与`docker container -d`选项混淆，`-d`选项表示将整个容器运行在系统的后台，`deamon off`表示容器内的主程序不能运行在容器后台








# docker镜像管理基础


Docker镜像含有容器所需要的文件系统及其内容，且采用**分层构建**机制，最底层为bootfs，其次为rootfs
- bootfs：用于系统引导的文件系统，包括bootloader和kernel，容器启动完成后会被卸载以节约内存资源
- rootfs：位于bootfs之上，表现为docker容器的根文件系统
	* 传统模式中，系统启动时，内核挂载rootfs时首先将其挂载为"只读"模式，完整性自检完成后重新挂载为读写模式
	* docker中，rootfs由内核挂载为"只读"模式，而后通过**联合挂载**技术额外挂载一个"可写"层

## 分层构建、联合挂载

Docker的镜像并不是传统的iso镜像，其每个镜像都分成了许多层，每一层都是只读层，使用镜像时就是将这些分层挂载到了一起，从外面看起来就像是一个镜像，每个低层镜像都是高层镜像的父镜像，启动容器时必须将所有层级镜像逐层启动。镜像的所有层都是只读层，但容器必定会产生数据，所以在创建容器时，只读层全部挂载完毕后还会在最顶层额外挂载一个读写层，这个读写层随着容器的消亡而消亡

以一个nginx镜像为例，假设一个nginx镜像由centos+nginx这2个镜像层组成，那么centos层必然会作为nginx层的下层镜像，也就是父镜像，同时它也处于整体镜像的最底层，启动nginx容器时，centos层和nginx层会联合挂载到一起，且从下层镜像开始往上逐层启动，启动镜像过程具备先后顺序

从dockerhub下载到本地的镜像占用的存储可能会比dockerhub上显示的要小一些，因为dockerhub上显示的存储大小是经过压缩过的，这样从dockerhub下载到本地时产生的流量会小一些。因为docker的镜像以分层的形式存在，那不同的镜像使用到相同的层时，再从dockerhub下载镜像，docker就不会再下载这些本地已有的层

位于下层的镜像称为父镜像，最底层的称为基础镜像(base image)，它通常用来供给一个系统的基本构成；注意，bootfs在容器启动时，一旦rootfs被引导完之后，bootfs会从内存中被移除，但不是被删除

#### Docker Registry分类

启动容器时，Docker Daemon会尝试从本地获取相关镜像，本地镜像不存在时将从Registry中下载该镜像保存到本地，默认情况下会从dockerhub下载镜像，如果想下载其他Registry的镜像，则拉取镜像时必须写明Registry服务器地址和相关镜像名

```shell
docker pull <registry>[:<port>]/[<namespace>/]<name>:<tag>
```

- Sponsor Registry：第三方的registry，供客户与Dockers社区使用
- Mirror Registry：第三方的registry，只让用户可用
- Vendor Registry：由发布Dokcer镜像的供应商提供的registry
- Private Registry：通过设有防火墙和额外的安全层的私有实体提供的registry

#### repository
Repository
- 由特定的docker镜像的所有迭代版本组成的镜像仓库
- 一个registry中可存在多个repository
	- repository可分为"顶层仓库"和"用户仓库"
	- 用户仓库名称格式为"用户名/仓库名"
- 每个仓库可以包含多个tag，每个tag对应一个镜像

### 制作镜像
制作镜像的三种方式：
* Dockerfile
* 基于容器制作
* Docker Hub automated builds

示例：基于容器制作镜像
```shell
docker container run --name box01 -it busybox:latest /bin/sh    #运行一个容器

#在容器中做出修改
/ # mkdir -p /data/html
/ # echo 'Busybox Server.' > /data/html/index.html

docker container commit -a "hebo1248@163.com" -p box01    #在不停止容器的前提下新开一个远程连接，以容器box为模板制作镜像
	-a：表示指定作者
	-p：表示制作镜像时短暂暂停容器的运行
docker image tag ff5fa57d01aa hebor/httpd:box01    #为镜像打标签
docker image tag hebor/httpd:box01 hebor/httpd:latest	#多标签
docker rmi hebor/httpd:latest    #删除标签
```

基于容器制作镜像时并不是制作了一个完整的镜像，而是将容器的读写层的修改内容单独制作成了一个镜像

注：
	1. 制作镜像时不指定仓库和标签，则此镜像仓库和tag都为空，这种镜像也被称为虚悬镜像
	2. tag选项也可以添加标签，可以为一个镜像打上多个标签，类似硬链接，删除标签时不会删除镜像

新镜像制作成功了，但是使用镜像启动容器时的默认命令却没有更改，使用以下命令查看默认启动的命令：

```shell
# docker inspect hebor/httpd:box01    #查看Cmd
```

commit选项自带了一个参数用于更改默认启动命令

```shell
docker commit -a "hebo1248@163.com" -c 'CMD ["/bin/httpd","-f","-h","/data/html"]' -p box01 hebor/httpd:box03
docker inspect hebor/httpd:box03	#查看默认命令
```

### 上传镜像

上传镜像时需要指定registry地址，如果没有指定，则默认使用dockerhub，也就是docker.io，上传镜像的前提是本地镜像标签要注意用户名和仓库名必须与dockerhub上的相对应。例如在dockerhub上创建了hebor/busybox名称空间，hebor表示用户名，每个人的用户名不一样，busybox表示仓库名，dockerhub上用户名是固定的，所以必须要先在dockerhub上创建一个busybox仓库，本地镜像才能上传成功 。如果本地镜像tag与dockerhub名称空间不匹配，则先修改本地要上传的镜像的tag后再上传


示例：上传镜像

```shell
docker login -u hebor	#默认登录dockerhub
docker tag hebor/httpd:box03 hebor/busybox:httpd-v0.1	#修改tag标签，要上传的镜像的tag标签必须严格符合镜像仓库创建的名称空间命名
docker push hebor/busybox:httpd-v0.1
```

这里使用了默认的dockerhub，所以完整的标签名应该是docker.io/hebor/busybox:httpd-v0.1，上传的完整路径也应该是

```shell
docker push docker.io/hebor/busybox:httpd-v0.1
```

如果使用registry不是dockerhub，那在打标签和上传镜像时就应该明确指出使用的registry地址，比如使用阿里云的registry

```shell
docker push registry.cn-hangzhou.aliyuncs.com/hebor/busybox:httpd-v0.1
```

上传镜像时如何不指定tag，那将会上传整个repository的镜像

### 镜像的导入和导出
示例：镜像打包
```shell
docker image save -o images.gz hebor/httpd:box01 hebor/httpd:box03    #将多个镜像打包到一起
```
注：使用image id打包，解包后时不带有仓库名和标签的，也就是说打开后是一个虚悬镜像。
也可以仅指定单个镜像打包，打包完成后使用scp或其他命令到其他主机上解包即可 

示例：镜像解包

```shell
# docker image load -i images.gz    #镜像解包
```

使用save打包镜像时仅打包了容器的读写层镜像，所以导入镜像之前，主机上需要下载好对应的base image，否则直接运行解包的镜像时还是会先pull基础镜像







# 虚拟化网络

在实现容器技术需要隔离的六种资源中提到过，网络名称空间(NET)主要实现网络设备/协议栈的隔离，如果为一个namespaces单独分配一个物理网卡设备，那么其他namespaces就看不见这个物理网卡了，这个namespaces内部与外界的通信也是没有问题的，可以为每一个namespaces单独分配一个物理网卡设备以解决namespaces的网络通信问题

但如果namespaces的数量大于物理网卡设备的数量时如何解决，每一个namespaces内部的网络进程也需要通过网络进行通信又如何解决，虚拟网卡技术能解决这些问题，用纯软件的方式模拟网卡设备来使用，Linux内核级支持二层/三层网络设备的模拟


## 虚拟二层网络通信

对于二层网络通信的模拟，利用Linux内核对二层虚拟设备的支持创建虚拟网卡接口，每一个虚拟网卡接口都是成对出现的，可以模拟为一根网线的两头，一头可以模拟插在主机上，另一头可以模拟插在交换机上，相当于将一个主机连接到交换机上。而Linux原生支持模拟二层虚拟网桥设备，那么将虚拟网卡设备的一头分配到namespaces，另一头分配到虚拟网桥，则相当于namespaces连接到网桥上，如果将多个namespaces连接到网桥，且配置为同一网段，那么不同的namespaces之间也能够相互通信了

## 虚拟三层网络通信

二层网络之间的通信通过虚拟网桥和虚拟网卡可以实现，不同网段之间的通信则需要通过三层路由来实现，而Linux内核本身则可以实现路由转发功能（通过iptables或打开内核的路由转发功能），将内核的路由转发放置在一个单独的namespaces内模拟路由器，并使用虚拟网卡连接路由器和网桥（一个单臂路由模型）即可实现不同网段之间的namespaces通信 

docker的网络通信中，每创建一个容器，都会创建一个虚拟网卡，一头放在容器内，另一头接在docker0网桥上。docker0桥默认是一个NAT桥，每创建并启动一个容器时，会自动创建一个iptables规则。删除容器时也会自动将iptables规则删除

```shell
# brctl show    #查看网桥上关联的端口
```

### ip命令操作网络名称空间

上述二层、三层网络通信都是基于网络名称空间进行理解，实际上在CentOS7系统中有自带的一个iproute工具包，这个包里有一个ip工具，在它的众多参数中存在一个netns，通过netns能够操作网络名称空间来模拟容器间通信。**使用ip命令管理网络名称空间时，只有网络名称空间是隔离的，其他名称空间都是共享的**


```shell
# 1.创建网络名称空间
[root@base ~]# ip netns help    #查看帮助手册
[root@base ~]# ip netns add r1  #创建两个netns
[root@base ~]# ip netns add r2
[root@base ~]# ip netns list    #查看创建的netns
[root@base ~]# ip netns exec r1 ifconfig -a   #在netns里执行命令查看网卡

# 2.配置虚拟网卡接口
[root@base ~]# ip link add name veth1.1 type veth peer name veth1.2
    #创建虚拟网卡接口对，指定veth1.1的对端接口名称为veth1.2，默认两个veth接口都在宿主机上且都未激活
[root@base ~]# ip link show     #查看虚拟网卡接口
[root@base ~]# ip link set dev veth1.1 netns r1     #设置虚拟网卡的veth1.1属于网络名称空间r1
[root@base ~]# ip netns exec r1 ip link set veth1.1 name eth0   #将netns中的虚拟网卡接口改名为eth0
[root@base ~]# ifconfig veth1.2 192.168.42.2/24 up  #为物理设备上的veth1.2配置临时地址
[root@base ~]# ip netns exec r1 ifconfig eth0 192.168.42.1/24 up    #为netns内的eth0配置临时地址
[root@base ~]# ping 192.168.42.1    #测试物理设备与netns通信是否正常

# 3.测试两个名称空间之间的通信
[root@base ~]# ip link set veth1.2 netns r2     #将虚拟网卡连接到r2
[root@base ~]# ip netns exec r2 ifconfig veth1.2 192.168.42.2/24 up
[root@base ~]# ip netns exec r2 ping 192.168.42.1
```

创建netns后如果没有给它指定网卡，那么在netns内就应该只有一个本地回环口lo。通过ip命令也能够创建虚拟网卡对，将虚拟网卡手动连接到名称空间中。将物理机上的虚拟网卡的另一半放至netns r2中则变成r1与r2通信。将虚拟网卡接口移到netns中默认是不激活的。如果失误将虚拟网卡的两个接口都放至一个netns中了，则使用以下命令将接口移出

```shell
# ip netns exec r1 ip link set dev veth1.1 netns r2
```

### docker容器端口映射{#index1}
默认情况下容器对外的网络通信是没有问题的，但如果想要由外向内访问容器，则需要将容器暴露出去。将容器端口与主机端口做映射，就相当于做DNAT，端口映射的方式有5种：
```shell
-p <containerPort>    #将指定的容器端口映射至主机所有地址的一个动态端口
-p <hostPort>:<containerPort>    #将容器端口映射至指定的主机端口
-p <ip>::<containerPort>    #将指定的容器端口映射至主机的某一个IP的动态端口
-p <ip>:<hostPort>:<containerPort>    #将容器端口映射至主机的某个IP的指定端口
	"动态端口"指随机端口，具体的映射使用docker port命令查看
-P 映射所有端口。这里的所有端口是指，构建镜像时要开放的所有端口。基于镜像启动容器时默认不会暴露端口。
```
查看端口映射[示例](#index2)


### docker的四种网络模型
示例：查看bridge网络详细信息
```shell
# docker network inspect bridge 
```

#### 1.closed container
封闭式容器，不为此容器的网络名称空间创建任何的网络设备，只有一个lo环回接口 <br />
示例：
```shell
# docker container run --name t1 --network none -it --rm -h t1.example.com busybox
	创建一个封闭式容器，并为此容器指定主机名，此容器创建完成后仅有一个lo口
```

#### 2.bridge container{#index2}
桥接式容器，通过nat的方式将容器和docker0网桥连接。为容器创建虚拟网卡，一半在容器中，一半接在docker0桥上
示例：包括[端口映射](#index1)
```shell
# docker container run --name t2 -it --rm --network bridge --dns 114.114.114.114 --dns-search example.com busybox
	创建一个桥接容器，指定dns服务器和域名
	--add-host centos.example.com:10.250.1.11    添加hosts文件解析条目

# docker container run --name t3 -d --network bridge -p 80  nginx
	将nginx服务的80端口映射到主机所有地址的随机高端口

# docker container run --name t4 -p 10.250.1.11::80 --rm nginx
	将nginx服务的80端口映射到主机指定地址的随机高端口
# docker port t4	查看端口映射
80/tcp -> 10.250.1.11:32769

# docker container run --name t5 -d -p 10.250.1.11:6000:80 --rm nginx
	指定地址和端口映射容器
```

#### 3.joined container
联盟式容器，将一部分namespaces隔离(User,Mounted,Pid)，UTS、IPC、NET则共享，在docker上表现为host网络
示例：创建两个联盟容器
```shell
# docker container run --name t6 -it --rm busybox
	先创建一个桥接容器
# docker container run --name t7 -it --rm --network container:t6 busybox
	创建容器指定网络是t6的容器网络。查看t6和t7的ip，此两个容器的ip共享
```

#### 4.open container
开放网络，直接共享物理机的namespaces
示例：共享主机网络
```shell
# docker container run --name t8 -it --rm --network host busybox
	共享主机网络启动容器。ifconfig查看网络与主机网络一致，启动http服务可直接通过主机地址访问
```

### 自定义docker网桥
更改docker0桥的网段，和其他的网络属性信息。创建新的docker桥，比如docker1

#### 更改docker0桥的属性
更改docker0桥的属性文件路径：/etc/docker/daemon.json
```shell
{
        "bip": "192.168.42.1/24",
        "mtu": 1500
        "dns": ["114.114.114.114","8.8.8.8"]
}
```
bip为核心选项，bridge ip，用于指定docker0桥本身的地址，其他选项可通过此地址计算得出

#### 容器的远程连接
docker0守护进程的C/S，默认仅监听Unix Socket格式的地址：/var/run/docker.sock。而Unix Socket文件只支持本地通信，如果想从docker0监听docker1的容器，默认是不可以的。docker客户端命令通信连接服务器时会用到**-H**选项，指定要连接的docker服务器，如果不指，默认指向/var/run/docker.sock文件。 <br />
如果想要docker服务器允许外部的连接，那就需要监听一个正常的tcp端口，在这里有很多教程会选择在/etc/docker/daemon.json文件中添加一行配置：
```shell
"hosts": ["tcp://0.0.0.0:2375","unix:///var/run/docker.sock"]
```
注：
```diff
- /etc/docker/daemon.json会被docker.service的配置文件覆盖，直接添加daemon.json不起作用，还可能导致docker服务起不来
```
此处应该编辑docker服务的配置文件：/lib/systemd/system/docker.servcie
```shell
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock
	更改此选项，为了防止出错，应注释原有选项后重新编辑
```
重启docker服务后，从其他安装了docker的主机可以访问本机的docker服务
示例：查看远程主机上的镜像
```shell
# docker -H 10.250.1.11:2375 images
```

#### 创建新的docker桥
docker支持的网络插件可以通过docker info命令查看Network项，目前docker Network项包括了6种：bridge、host、ipvlan、macvlan、null、overlay
示例：创建一个bridge网桥
```shell
# docker network create -d bridge --subnet "172.26.0.0/16" --gateway "172.26.0.1" mybr0
	-d：指定驱动类型
	--subnet：指定ipv4子网
	--gateway：网关
	mybr0：网桥名称

# docker network ls    查看网桥信息

# docker container run --name t9 -it --rm --network mybr0 busybox
	创建容器连接到自定义的网桥
```
ifconfig也能够看到mybr0的网桥，只不过名字是随机的，可以通过ip命令更改名称
