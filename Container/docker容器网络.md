虚拟化网络
========
在实现容器技术需要隔离的六种资源中提到过，网络名称空间(NET)主要实现网络设备/协议栈的隔离，如果为一个namespaces单独分配一个物理网卡设备，那么其他namespaces就看不见这个物理网卡了，这个namespaces内部与外界的通信也是没有问题的，可以为每一个namespaces单独分配一个物理网卡设备以解决namespaces的网络通信问题，但如果namespaces的数量大于物理网卡设备的数量时如何解决，每一个namespaces内部的网络进程也需要通过网络进行通信又如何解决

虚拟网卡
-------
用纯软件的方式模拟网卡设备来使用，Linux内核支持二层/三层网络设备的模拟

### 虚拟二层网络通信
对于二层网络通信的模拟，利用Linux内核对二层虚拟设备的支持创建虚拟网卡接口，每一个虚拟网卡接口都是成对出现的，可以模拟为一根网线的两头，一头可以模拟插在主机上，另一头可以模拟插在交换机上，相当于将一个主机连接到交换机上。而Linux原生支持模拟二层虚拟网桥设备，那么将虚拟网卡设备的一头分配到namespaces，另一头分配到虚拟网桥，则相当于namespaces连接到网桥上，如果将多个namespaces连接到网桥，且配置为同一网段，那么不同的namespaces之间也能够相互通信了

### 虚拟三层网络通信
二层网络之间的通信通过虚拟网桥和虚拟网卡可以实现，不同网段之间的通信则需要通过三层路由来实现，而Linux内核本身则可以实现路由转发功能(通过iptables或打开内核的路由转发功能)，将内核的路由转发放置在一个单独的namespaces内模拟路由器，并使用虚拟网卡连接路由器和网桥(一个单臂路由模型)即可实现不同网段之间的namespaces通信 <br /> <br />

docker的网络通信中，每创建一个容器，都会创建一个虚拟网卡，一头放在容器内，另一头接在docker0网桥上
```shell
# brctl show    #查看网桥上关联的端口
```
docker0桥默认是一个NAT桥，每创建并启动一个容器时，会自动创建一个iptables规则。删除容器时也会自动将iptables规则删除

### ip命令操作网络名称空间
示例：创建网络名称空间
```shell
# ip netns add r1		创建两个netns
# ip netns add r2
# ip netns list			查看创建的netns
# ip netns exec r1 ip add	在netns里执行ip命令
```
注：使用ip命令模拟的网络名称空间，**除了网络名称空间是隔离的，其他的都是共享的**

示例：配置虚拟网卡接口
```shell
# ip link add name veth1.1 type veth peer name veth1.2			创建虚拟网卡接口，指定veth1.1的对端接口名称为veth1.2
# ip link show												查看虚拟网卡接口
# ip link set dev veth1.1 netns r1							将虚拟网卡的veth1.1端放至netns r1中
# ip netns exec r1 ip link set dev veth1.1 name eth0		将netns中的虚拟网卡接口改名为eth0
# ifconfig veth1.2 10.250.1.5/24 up							为物理设备上的veth1.2配置临时地址
# ip netns exec r1 ifconfig eth0 10.250.1.6/24 up			为netns内的veth1.1配置临时地址
# ping 10.250.1.6											测试物理设备与netns通信是否正常
```
补充：将物理机上的虚拟网卡的另一半放至netns r2中则变成r1与r2通信。将虚拟网卡接口移到netns中默认是不激活的。如果失误将虚拟网卡的两个接口都放至一个netns中了，则使用以下命令将接口移出
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
联盟式容器，将一部分namespaces隔离(User,Mounted,Pid)，UTS、IPC、NET则共享
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