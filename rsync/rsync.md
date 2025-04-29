# 数据备份

数据备份的前提是**该数据非常重要**。备份方式分2种：完全备份、增量备份。完全备份每次都是所有数据的拷贝，效率低；增量备份只同步新增的不同数据。常见的备份工具有2个：scp、rsync。

scp使用ssh协议进行网络拷贝，以全备的方式每一次拷贝都会覆盖旧数据，如果数据量比较大的情况下，scp的每次备份都会花费比较长的时间。如果需要进行数据备份的两台机器处于同一个局域网内，那么即便scp的全量备份会耗费较长时间也是可以接受的，但如果两台需要进行数据备份的机器需要通过互联网跨地域进行数据备份，首先，互联网的带宽可能更低、耗时更久，其次，互联网的稳定性肯定是没有局域网的稳定性好，数据量较大的全量备份很可能导致数据备份不完整

## 一、Rsync基本概述

Rsync是一款开源的快速增量备份工具，用于在不同的主机系统间进行同步，实现全备与增备。Rsync全称Remote Sync（远程同步），支持本地复制，或与其他SSH、Rsync主机同步，因此非常适合用于架构集中式备份或异地备份等应用

使用Rsync工具在备份数据的过程中，服务器分为两个角色，两个角色之间又存在两种数据同步方式

1. 角色
   - Rsync同步源：Rsync同步源又称为备份源，指的是数据所在的源服务器，负责响应来自Rsync客户机的同步操作请求
   - Rsync客户机：Rsync客户机又称为发起端，指的是数据最终要备份到的目标服务器，负责发起同步操作请求

2. 数据同步方式
   - 下行同步：下行同步也可以理解为定时同步，由Rsync客户机向Rsync同步源发起请求，下载Rsync同步源的数据
   - 上行同步：上行同步也可以理解为实时同步，由Rsync同步源主动上传数据到Rsync客户机

在大量服务器数据备份的场景下，单独使用某一种数据同步方式都不合适。上行同步即所有Rsync同步源推送本地数据到Rsync客户机，会导致数据同步缓慢，适合少量数据备份；下行同步即Rsync客户机拉取所有Rsync同步源上的数据，会导致Rsync客户机的开销加大。因此在大量设备、数据需要备份的场景下，建议两种数据传输方式结合使用

![大量服务器备份场景](https://www.z4a.net/images/2023/02/07/99355b44b24cab183fb4c6fb9f695a23.png)

![异地备份实现](https://www.z4a.net/images/2023/02/07/b4e1dc28ebd9f74533e3793b01cba7e2.png)

## 二、Rsync配置

### 配置Rsync同步源

某些情况下，Rsync工具会集成在linux系统中，本身不需要单独安装

1. 基本工具准备

   ```bash
   [root@node-1 ~]# yum install -y lrzsz vim rsync
   ```

2. 编辑rsyncd.conf配置文件

   ```bash
   [root@node-1 etc]# cp /etc/rsyncd.conf{,.bak}    # 对Rsyncd配置文件进行备份
   [root@node-1 etc]# vim /etc/rsyncd.conf
   uid = nobody    # Rsyncd守护程序的用户，缺省为nobody用户
   gid = nobody    # Rsyncd守护程序的组
   use chroot = yes    # 将Rsync客户机的权限禁锢在根目录下
   max connections = 4    # 设置最大连接数
   pid file = /var/run/rsyncd.pid    # 设置pid文件路径
   timeout = 900    # 超时时间
   ignore nonreadable = yes    # 设置强制Rsync同步源忽略用户没有访问权限的文件，当该参数设置为yes时，rsync在同步过程中会自动跳过所有Rsync客户机用户无权读取的文件或目录，避免因权限不足导致同步失败或中断
   dont compress = *.gz *.tgz *.zip *.z *.Z *.rpm *.deb *.bz2    # 设置文件是否压缩；以.gz、.tgz等后缀结尾的文件不用压缩，其他文件都进行压缩
   port = 873    # Rsyncd守护进程监听的端口。Rsyncd守护进程缺省监听873/tcp端口
   log file = /var/log/rsyncd.log    # 设置Rsyncd的日志文件
   address = 192.168.0.30    # Rsyncd守护进程监听的地址
   hosts allow = 192.168.0.0/24    # 允许指定主机进行数据同步
   
   [directory]    # 共享模块的名称。Rsync同步源向外发布的目录的名称，可自定义
   	path = /var/www/html/    # 共享模块的本地路径
   	comment = Remote Sync    # 对共享模块的说明，可不配置
   	read only = yes          # 设置只读权限
   	auth users = hebor       # 在共享模块中启用用户身份验证
   	secrets file = /etc/rsyncd_users.db    # 用户账号信息的存放路径
   ```

   `use chroot = yes`是一种安全措施，旨在将Rsync客户机的权限禁锢在根目录下，此根目录指的是Rsyncd守护程序的根目录，例如，将`/var/www/html/`目录设置为Rsyncd守护进程的根目录，即`/var/www/html/`目录下的数据同步到Rsync客户机，那么Rsync客户机就只能查看到此根目录下的文件。如果不启用此参数，那么Rsync客户机是有可能能够进入上级目录的，对于服务器而言非常不安全

   缺省情况下Rsyncd不配置身份验证，使用匿名身份信息进行数据同步。当Rsyncd配置身份验证时，其使用到的账户信息并不是系统账户，系统账户信息一般保存在`/etc/passwd`文件下，Rsyncd的账号信息可以通过新建secrets file文件来保存

3. 配置Rsyncd的独立的账号文件

   ```bash
   [root@node-1 etc]# vim /etc/rsyncd_users.db    # 配置Rsyncd的账户
   hebor:redhat
   [root@node-1 etc]# chmod 600 /etc/rsyncd_users.db    # 修改Rsyncd账号文件的权限，此步骤是必须要做的
   ```

   Rsyncd的secrets file文件采用“用户名:密码”的固定记录格式，每一行只能记录一个账户信息，Rsyncd的账户信息是独立的，不依赖于系统账号

4. 启用Rsyncd服务

   ```bash
   [root@node-1 etc]# rsync --daemon
   [root@node-1 ~]# setenforce 0    # 临时取消Selinux的限制
   [root@node-1 ~]# firewall-cmd --add-service=rsyncd --permanent    # 放行防火墙规则
   [root@node-1 ~]# firewall-cmd --reload
   ```

   通过`--daemon`选项使Rsyncd在后台监听服务端口

5. Rsyncd同步源创建测试的源数据文件

   ```bash
   [root@node-1 ~]# mkdir -p /var/www/html
   [root@node-1 ~]# touch /var/www/html/{1..10}.txt
   ```

6. 停止Rsyncd服务

   ```bash
   [root@node-1 ~]# kill -9 $(cat /var/run/rsyncd.pid)    # 通过杀死Rsyncd的进程来关闭Rsyncd服务
   [root@node-1 ~]# \rm /var/run/rsyncd.pid    # 删除Rsyncd服务产生的pid文件，避免下次启动Rsyncd服务时产生冲突
   ```

无论共享模块的本地路径是否存在，都可以正常启动Rsyncd服务，但这可能会出现数据同步有误的情况

### Rsync发起端命令语法

**Rsync命令语法**

```bash
rsync [选项] 数据原始位置 备份的目标位置
```

**Rsync常用选项**

- -a：归档模式，递归并保留对象属性，等同于`-rlptgoD`
- -v：显示同步过程的详细信息（verbose）
- -z：在传输文件时进行压缩（compress）；此参数结合Rsyncd配置文件中的`dont compress`参数来对传输过程中的指定数据进行压缩传输
- -H：保留硬链接文件
- -A：保留ACL属性信息
- --delete：删除备份的目标位置有，而数据原始位置没有的文件；此参数一般是用于需要实现两个主机之间的数据无差异同步的场景，缺省情况下rsync是增量备份工具，以下行同步为例，当Rsync同步源已经向Rsync客户机上同步了A、B、C三个数据文件时，此时Rsync同步源删除了文件A、新增了文件D，缺省情况下，Rsync同步源再次向Rsync客户机同步数据时，Rsync客户机上会保留文件A，并同步文件B、C、D。`--delete`选项的作用会使Rsync客户机也同步删除掉文件A
- --checksum：根据对象的校验来决定是否跳过文件

**Rsync同步源的两种表示方式**

```bash
格式1：用户名@主机地址::共享模块名
格式2：rsync://用户名@主机地址/共享模块名
```

```bash
[root@localhost ~]# rsync -avz hebor@192.168.0.30::directory /root

[root@localhost ~]# rsync -avz rsync://hebor@192.168.0.30/directory /root
```

#### Rsync操作示例

1. 本地传输示例

   ```bash
   # rsync本地拷贝类似于cp命令，本地拷贝语法：
   # Local:  rsync [OPTION...] SRC... [DEST]
   
   [root@node-2 ~]# rsync -avz ./anaconda-ks.cfg /opt/
   ```

2. 远程同步示例

   ```bash
   # rsync远程同步使用ssh协议实现网络传输，远程同步语法：
   # Pull: rsync [OPTION...] [USER@]HOST:SRC... [DEST]
   # Push: rsync [OPTION...] SRC... [USER@]HOST:DEST
   
   [root@node-2 ~]# rsync -avz /etc/passwd root@192.168.0.30:/tmp/	# 客户端推送文件到服务端
   [root@node-2 ~]# rsync -avz root@192.168.0.30:/etc/passwd ./		# 客户端从服务端拉取文件
   ```

   远程同步的缺陷在于，Rsync需要借助SSH协议远程同步数据，借助SSH协议同步数据时需要使用系统用户实现数据备份，如果使用超级用户root可能会引发安全风险，如果使用普通用户又可能导致权限不足

3. 守护进程示例

   ```bash
   # rsync守护进程语法：
   # Pull: rsync [OPTION...] [USER@]HOST::SRC... [DEST]
   # 	    rsync [OPTION...] rsync://[USER@]HOST[:PORT]/SRC... [DEST]
   # Push: rsync [OPTION...] SRC... [USER@]HOST::DEST
   # 	    rsync [OPTION...] SRC... rsync://[USER@]HOST[:PORT]/DEST
   
   [root@node-2 ~]# mkdir /mnt/backup
   [root@node-2 ~]# rsync -avzH hebor@192.168.0.30::directory /mnt/backup    # 此处会要求验证身份信息，此处身份信息是Rsyncd的账户hebor，而非系统用户
   
   # 增量备份；Rsync同步源新增文件，正常情况下，增量备份应该只备份3个文件
   [root@node-1 ~]# touch /var/www/html/{11,12}.txt
   [root@node-1 ~]# echo "aaa" > /var/www/html/1.txt
   
   [root@node-2 ~]# rsync -avz hebor@192.168.0.30::directory /mnt/backup
   ```

#### Rsync无差异同步

无差异同步：以某一台设备为主，其他设备的数据必须与主设备同步的数据完全一致。无论其他设备上的数据是否为新数据或旧数据，只要主设备上没有，同步之后其他设备上的差异数据也会被移除，同理，只要主设备上有的数据，也会同步到其他设备

```shell
# 在Rsync同步源的数据目录下新增测试文件，并测试Rsync客户机的数据同步
[root@node-1 ~]# cp /etc/services /var/www/html/

[root@node-2 ~]# rsync -avz hebor@192.168.0.30::directory /mnt/backup

# 在Rsync同步源的数据目录下删除测试文件，再次测试Rsync客户机的数据同步
[root@node-1 ~]# \rm /var/www/html/services

[root@node-2 ~]# rsync -avz hebor@192.168.0.30::directory /mnt/backup
[root@node-2 ~]# ls /mnt/backup    # 同步后查看Rsync客户机的备份目录，没有变化

# Rsync客户机使用无差异化同步，测试数据同步
[root@node-2 ~]# rsync -avz --delete hebor@192.168.0.30::directory /mnt/backup
[root@node-2 ~]# ls /mnt/backup    # 与Rsync同步源的数据完全同步
```

推送数据时以客户端为主，客户端数据会强制同步服务端数据；拉取数据时以服务端为主，服务端数据强制同步客户端

#### Rsync传输限速

内部架构中，为了避免rsync同步数据时占用过大的带宽，而导致用户体验变差，可能会考虑到Rsync传输限速

```bash
# 1. 在Rsync同步源上新建大容量文件
[root@node-1 ~]# dd if=/dev/zero of=/var/www/html/size.disk bs=1M count=100

# 2. 在Rsync客户机上同步数据
[root@node-2 ~]# rsync -avzP hebor@192.168.0.30::directory /mnt/backup
	-P：显示传输速率和进度信息

# 3. 在Rsync客户机上删除测试文件，并通过限速传输再次同步
[root@node-2 ~]# rm -f /mnt/backup/size.disk
[root@node-2 ~]# rsync -avzP --bwlimit=1 hebor@192.168.0.30::directory /mnt/backup
	--bwlimit：限制带宽，默认单位是MB
```

#### Rsync非交互同步

取消交互输入密码的步骤，是仅需要客户端做的配置，服务端不需要此配置

```bash
# 方式1：将虚拟用户的密码写入密码文件
[root@node-2 ~]# echo "redhat" > /etc/rsync.pass
[root@node-2 ~]# chmod 600 /etc/rsync.pass

# 通过指定密码文件位置来实现非交互式执行同步
[root@node-2 ~]# rsync -avzP --password-file=/etc/rsync.pass hebor@192.168.0.30::directory /mnt/backup/

# 方式2：设置Rsync密码的环境变量
[root@node-2 ~]# export RSYNC_PASSWORD="redhat"
[root@node-2 ~]# rsync -avzP --delete hebor@192.168.0.30::directory /mnt/backup    # 不再需要输入密码
```

**Rsync非交互定时同步示例**

```bash
[root@node-2 ~]# echo "redhat" > /etc/rsync.pass
[root@node-2 ~]# chmod 600 /etc/rsync.pass
[root@node-2 ~]# crontab -e
30 7 * * * /usr/bin/rsync -az --delete --password-file=/etc/rsync.pass hebor@192.168.0.30::directory /mnt/backup
[root@node-2 ~]# systemctl restart crond
[root@node-2 ~]# systemctl enable crond
```

非交互式定时同步无需关注同步详细信息，不使用`-v`参数

#### rsync的扩展选项

- --exclude=PATTERN：指定不需要传输同步的文件名

- --exclude-from=FILE：指定传输同步的黑名单文件，这个文件中记录的所有文件名都不会同步

- --partial：断点续传
- -P：关于`-P`选项，在rsync 3.1.2版本中，使用`--help`查看帮助手册时对`-P`的注解是`same as --partial --progress`，所以`-P`选项实际上包含了2部分功能：*断点续传*和*显示传输进度*

```shell
# Rsync客户机同步数据时，忽略directory模块下的所有9.txt文件
[root@node-2 ~]# rm -rf /mnt/backup/*    # 清空Rsync客户机的备份目录
[root@node-2 ~]# rsync -avzP --delete --exclude '9.txt' hebor@192.168.0.30::directory /mnt/backup

# 通配符使用：--exclude file*
# 黑名单使用：--exclude-from=/etc/rsync/exclude.list
```

注：

1. `--exclude`的路径必须是相对路径，不能是绝对路径，不可写为`/etc/passwd`
2. 系统会把文件和目录一视同仁，如果testuser是一个目录，同样不会复制
3. 如果想仅避开/backup/etc/目录下的passwd文件，可以这么写`etc/passwd`
4. 可以使用通配符排除不想复制的内容

使用`--exclude-from=FILE`选项时，命令行的写法建议使用绝对路径表明黑名单文件路径，单黑名单文件本身的内容仍与`--exclude`写法一致，需要使用相对路径

####  Rsyncd的第二种配置

**Rsyncd同步源的配置**

```shell
[root@node-1 ~]# rpm -qc rsync	# 查找rsync的配置文件
[root@node-1 ~]# vim /etc/rsyncd.conf
uid = rsync
gid = rsync
port = 873
fake super = yes
use chroot = no
max connections = 200
timeout = 600
ignore error
read only = false
list = false
auth users = rsync_backup    # 全局配置中的账户验证配置
secrets file = /etc/rsyncd_users.db
log file = /var/log/rsyncd.log

[backup]
       path = /backup
       comment = rsync_backup

[root@node-1 ~]# useradd rsync -M -s /sbin/nologin    # 根据rsync配置文件新建用户
[root@node-1 ~]# echo "rsync_backup:Huawei@123.com" >> /etc/rsyncd_users.db    # 新建虚拟用户密码文件
[root@node-1 ~]# chmod 600 /etc/rsyncd_users.db
[root@node-1 ~]# mkdir /backup
[root@node-1 ~]# chown -R rsync. /backup/    # 修正数据目录的权限

# 启动rsyncd服务并检测
[root@node-1 ~]# setenforce 0
[root@node-1 ~]# firewall-cmd --add-service=rsyncd --permanent
[root@node-1 ~]# firewall-cmd --reload
[root@node-1 ~]# systemctl enable --now rsyncd
[root@node-1 ~]# systemctl status rsyncd
[root@node-1 ~]# ss -lntp | grep "rsync"
```

**Rsyncd服务配置文件解析**

| rsyncd.conf 参数                 | 参数说明                                                     |
| -------------------------------- | ------------------------------------------------------------ |
| uid = rsync                      | Rsync服务所属用户。若此用户不存在，需要提前创建              |
| gid = rsync                      | Rsync服务所属组                                              |
| port = 873                       | 端口号可修改                                                 |
| fake super = yes                 | 无需让rsync以root身份运行，允许接收文件的完整属性            |
| use chroot = no                  | 如果为true，禁锢推送的数据至某个目录, 不允许跳出该目录。这是一种安全配置，因为大多数传输都在内网，所以不配也没关系 |
| max connections = 200            | 设置最大连接数，默认 0，意思无限制，负值为关闭这个模块       |
| timeout = 600                    | 默认为 0，表示 no timeout，建议 300-600（5-10 分钟）         |
| ignore error                     | 忽略 I/O 错误                                                |
| read only = false                | 指定客户端是否可以上传文件，默认对所有模块为 true，true 表示不可上传 |
| list = false                     | 是否允许客户端可以查看可用模块列表，默认为可以               |
| auth users = rsync_backup        | 定义虚拟用户，作为连接认证用户。用户不需要在本地系统中存在，默认为所有用户无密码访问 |
| secrets file = /etc/rsync.passwd | 指定用户名和密码存放的文件                                   |
| log file = /var/log/rsyncd.log   | rsync使用rsyslog记录日志                                     |
| [backup]                         | 定义模块信息，模块名称需用中括号扩起来，起名称没有特殊要求   |
| path                             | 这个模块中，daemon 使用的文件系统或目录，目录的权限要注意和配置文件中的权限一致，否则会遇到读写的问题 |
| comment                          | 模块注释信息                                                 |

配置文件中可以存在多个模块，模块属于局部配置，在上例配置文件中，除了`[backup]`模块以外的所有配置都是全局配置

**Rsync客户机检测**

```shell
# 客户端推送 /etc/ 目录下的所有内容到服务端
[root@node-2 ~]# rsync -avz /etc/ rsync_backup@192.168.0.30::backup

# 客户端推送 /etc 这个目录到服务端
[root@node-2 ~]# rsync -avz /etc rsync_backup@192.168.0.30::backup

# 客户端拉取服务端内容
[root@node-2 ~]# rsync -avz rsync_backup@192.168.0.30::backup /opt
```

Rsync传输文件的语法针对`/`比较严格，将`/etc/`改为`/etc`则表示推送`/etc`目录到Rsync同步源，而不是仅推送`/etc/`目录下的所有内容到Rsync同步源；使用守护进程方式启用Rsyncd服务时，无法再像单纯的远程传输一样指定传输路径，只能针对服务端的模块同步传输

Rsync守护进程服务的2个用户

- rsync_backup：虚拟用户，Rsync客户机通过该虚拟用户连接rsyncd服务，作为rsync连接认证用户，不需要在本地系统中存在，虚拟用户由配置选项[auth users]定义；虚拟用户的账号信息现需要存放在一个指定文件中。由配置选项[secrets file]定义
- rsync：系统用户，用于Rsync同步源启动rsyncd服务时的所属用户，主配置文件中的模块对应的path，必须以uid和gid的用户进行授权，同步数据时以此用户身份写入对应路径。即需要修改模块所指定的path路径的所属用户和所属组

## 三、Rsyncd实时同步

通过学习rsync工具的服务配置与命令语法，现在已经通过增量备份的方式减小备份的时间和带宽的损耗，且通过定时任务实现免交互式的周期性数据备份。一般场景下仅使用定时同步即可满足业务需求，然而面对高可用性、高可靠性场景时，定时同步数据备份无法满足业务需求，备份时间固定、延迟明显、实时性差都是定时同步的缺陷

当然，也可以通过高密度的定时同步来尽可能满足对数据同步的实时性要求，但是当Rsync同步源的数据长期没有产生变化时，密集的定时同步又是不必要的。在这种场景下，实时同步成为了更优解，实时同步解决了定时同步的不足，一旦Rsync同步源数据产生变化时，立即触发数据备份，Rsync同步源没有产生变化时则不执行备份

### inotify

实时同步即上行同步，要实现Rsync同步源数据发生变化时触发数据同步，那么首先需要解决如何监控到Rsync同步源数据是否产生了变化的问题，这就需要借助Linux内核的inotify机制来实现，inotify从内核版本2.6.13开始提供，可以用于监控文件系统的变动情况，并做出通知响应

实时同步的实现，实际上也就是inotify+sync的组合实现，inotify用于监控文件系统是否产生变动，当文件系统产生变动时通过inotify通知接口触发sync执行数据同步。inotify本身作为一个Linux内核的机制，管理员需要借助相应的软件工具才能对inotify进行配置

实时同步工具的选择有`sersync`、`inotify+rsync`。sersync是基于`rsync+inotify-tools`开发的工具，其强化了实时监控、文件过滤、简化配置等功能，帮助用户提高运行效率，节省时间和网络资源；inotify+rsync需要管理员借助inotify-tools实现inotify配置

[sersync工具](https://github.com/wsgzao/sersync)

> **注意：角色转换**
>
> 如果说前文的下行同步指的是备份服务器主动从业务服务器下载数据到本地进行备份，那么上行同步就是需要将两者身份互换，由业务服务器主动将数据转发到备份服务器进行备份。在下文中仍会以Rsync同步源、Rsync客户机标注，但需要注意上行同步方式下，业务服务器才是Rsync客户机

### rsync+inotity

#### 命令语法

调整inotify内核参数

- max_queue_events：监控事件队列大小

- max_user_instances：最大监控实例数量

- max_user_watches：每个实例例最大的监控文件数

  `max_user_instances`和`max_user_watches`参数的值取决于要监控的数据路径下有多少个文件，监控数值应大于监控目标的总文件数，监控数值小于监控目标的总文件数时，可能会出现监控不到文件变化的情况

  实验环境下需要同步的文件数量少，inotify的默认参数完全可以满足实验需求，三个内核参数都可以无需调整

inotify-tools辅助工具

- inotifywait：用于持续监控，实时输出结果

  ```bash
  inotifywait -mrq -e modify,create,move,delete
  	-m：持续进行监控
  	-r：递归监控数据目录下的所有子对象
  	-q：简化输出信息
  	-e：指定要监控的事件类型
  ```

- inotifywatch：用于短期监控，任务完成后再输出结果

  短期监控并不适用于实时同步场景，短期监控可能会在某一次的数据变化之后，被判断未任务完成，输出结果结束进程

#### 实时同步配置

1. 配合inotify内核参数监控文件系统

   ```bash
   [root@node-1 ~]# vim /etc/sysctl.d/inotify.conf
   fs.inotify.max_queued_events=16384
   fs.inotify.max_user_instances=1024
   fs.inotify.max_user_watches=1046384
   [root@node-1 ~]# sysctl -p    # 重新加载系统内核参数，使inotify配置立即生效
   ```

2. 配置inotify-tools工具

   ```bash
   [root@node-1 ~]# yum install -y epel-release    # inotify-tools软件包在EPEL源里
   [root@node-1 ~]# yum install -y inotify-tools
   [root@node-1 ~]# mkdir /data/    # 创建业务目录
   [root@node-1 ~]# inotifywait -mrq -e modify,create,delete,move /data/    # 监控业务目录；缺省情况下在前台监控
   ```

   此时，在ssh远程连接工具上新建一个node-1节点的连接，在新连接的`/data/`目录上执行文件操作，然后可以在当前连接上看到inotify的监控记录输出

3. 通过inotifywait触发rsync同步

   直接在命令行使用inotifywait命令不便于使用，且inotifywait也还未与rsync联动，因此综合考虑之下可以选择通过shell脚本的方式结合两个工具的使用，并实现实时同步

   ```bash
   [root@node-1 ~]# vim /opt/inotify_rsync.sh
   #!/bin/bash
   INOTIFY_CMD="/usr/bin/inotifywait -mrq -e modify,create,attrib,move,delete /data/"
   RSYNC_CMD="/usr/bin/rsync -azH --delete --password-file=/etc/rsync.pass /data/ hebor@192.168.0.31::data"
   $INOTIFY_CMD | while read DIRECTORY EVENT FILE    # 读取inotifywait输出的监控记录
   do
       if [ $(pgrep rsync | wc -l) -le 0 ]; then    # 如果rsync未在执行，则立即启动
           $RSYNC_CMD
       fi
   done
   [root@node-1 ~]# chmod +x /opt/inotify_rsync.sh    # 赋予脚本执行权限
   [root@node-1 ~]# vim /etc/rsync.pass
   redhat
   [root@node-1 ~]# chmod 600 /etc/rsync.pass
   ```

4. 配置node-2的Rsyncd服务

   ```bash
   [root@node-2 ~]# vim /etc/rsyncd.conf
   uid = nobody
   gid = nobody
   use chroot = yes
   max connections = 4
   pid file = /var/run/rsyncd.pid
   timeout = 900
   ignore noreadable = yes
   dont compress = *.gz *.tgz *.zip *.z *.Z *.rpm *.deb *.bz2
   port = 873
   address = 192.168.0.31
   log file = /var/log/rsyncd.log
   hosts allow = 192.168.0.0/24
   
   [data]
           path = /mnt/data/
           comment = inotify rsync
           read only = false
           auth users = hebor
           secrets file = /etc/rsyncd_users.db
   [root@node-2 ~]# mkdir /mnt/data/
   [root@node-2 ~]# chown -R nobody. /mnt/data    # 调整path路径授权
   [root@node-2 ~]# vim /etc/rsyncd_users.db
   hebor:redhat
   [root@node-2 ~]# chmod 600 /etc/rsyncd_users.db
   [root@node-2 ~]# rsync --daemon    # 启动Rsyncd守护进程
   [root@node-2 ~]# setenforce 0
   [root@node-2 ~]# firewall-cmd --add-service=rsyncd --permanent
   [root@node-2 ~]# firewall-cmd --reload
   ```

   实时同步的node-2主机的Rsyncd服务配置与“Rsyncd的第二种配置”相似，大部分的配置在“Rsyncd的第二种配置”小结中都有注解，但此处还是对数据目录的授权问题再进行着重注解

   首先Rsyncd服务配置文件中有两种用户，`uid`和`gid`所指定的是本地系统用户，本地系统用户的作用是作为Rsyncd服务的所属用户；`auth users`所指定的是虚拟用户，虚拟用户仅用于Rsync客户机向Rsync同步源发起请求时进行身份验证。当Rsync客户机使用虚拟用户账号向Rsync同步源发起并通过身份验证后，接下来要对Rsync同步源的模块中所指定的`path`路径进行读写操作，对`path`路径的读写操作是以Rsyncd服务的所属用户身份执行的动作

   此时在Rsyncd同步源上就可能出现权限问题，为了避免读写权限问题，需要确保Rsyncd服务的所属用户对`path`路径具备独写权限，或可以使用root用户作为Rsyncd服务的所属用户，root账户具备系统最高权限，能最大可能的避免出现权限问题

5. 验证并执行脚本

   ```bash
   # 在node-1上执行脚本文件中的rsync命令
   [root@node-1 ~]# /usr/bin/rsync -azH --delete --password-file=/etc/rsync.pass /data/ hebor@192.168.0.31::data
   
   # 在node-2上检查数据是否同步
   [root@node-2 ~]# ls /mnt/data
   
   # 在node-1上执行实时同步脚本
   [root@node-1 ~]# sh /opt/inotify_rsync.sh &    # 在后台执行脚本
   [root@node-1 ~]# jobs    # 查看后台进程
   ```

   至此，实时同步配置已经完成，但由于node-1脚本文件中对数据目录的属性权限（attrib）变动也做了监控，因此当node-1数据目录下的文件权限出现变更时，rsync也会想要将权限同步到node-2。例如，node-1上在`/data/`目录新建一个文件，缺省情况下该文件的所属用户和所属组都是root，node-1会想将该文件的权限也同步到node-2

   由于node-1上的文件操作都是直接用root账户执行的，而node-2的Rsyncd服务所属用户是nobody，因此，当node-1想要将文件所属账户同步为root时会出现权限不足的错误。node-1脚本同步的权限错误不会影响到正常的数据同步，但在node-2上所有数据的所属用户都是Rsyncd服务的所属用户，即nobody

### 实时同步实践

实现web上传文件，实则是写入NFS，当NFS存储新数据时触发实时同步操作，复制到备份服务器

| 角色       | 外网IP         | 内网IP           | 安装工具                           |
| ---------- | -------------- | ---------------- | ---------------------------------- |
| web01      | eth0:10.0.0.7  | eth1:172.16.1.7  | httpd、php                         |
| nfs-server | eth0:10.0.0.31 | eth1:172.16.1.31 | nfs-tuils、rsync、inotify、sersync |
| backup     | eth0:10.0.0.41 | eth1:172.16.1.41 | rsync-server                       |

1. WEB上传文件至NFS

```shell
# NFS角色配置
[root@nfs ~]# yum install -y nfs-utils
[root@nfs ~]# more /etc/exports
/data/	172.16.1.0/24(rw,sync,all_squash,anonuid=666,anongid=666)
[root@nfs ~]# groupadd -g 666 www
[root@nfs ~]# useradd -u 666 -g 666 www
[root@nfs ~]# mkdir /data
[root@nfs ~]# chown -R www.www /data

# WEB角色配置
[root@web01 ~]# yum install -y httpd php nfs-utils
[root@web01 ~]# systemctl start httpd
[root@web01 ~]# firewall-cmd --add-service=http --permanent
[root@web01 ~]# firewall-cmd --reload
[root@web01 ~]# mount -t nfs 172.16.1.31:/data /var/www/html/
[root@web01 ~]# wget https://dqunying2.jb51.net/201911/yuanma/upload_jb51.rar
[root@web01 ~]# unar upload_jb51.rar
[root@web01 ~]# mv /root/upload_jb51/upload.php /var/www/html/index.php
[root@web01 ~]# groupadd -g 666 www
[root@web01 ~]# useradd -u 666 -g 666 www
[root@web01 ~]# more /etc/httpd/conf/httpd.conf | egrep -v "^$|^.*#"
...
User www
Group www
...
```

WEB端的httpd服务，其所属用户和所属组必须修改，需要与NFS的用户保持一致，否则会出现WEB端挂载NFS后，数据写入报错的情况，系统用户直接写入NFS存储没有问题，但通过网页上传文件时会报错，权限拒绝；网页默认登录密码`danbaise.com`

PHP默认限制上传文件的大小为2MB，上传文件时也可能会出现文件大小超出限制报错的情况，具体情况根据日志进行解决

2. WEB和NFS的数据都备份到BACKUP的/backup目录

```shell
# BACKUP角色配置
[root@backup ~]# yum install -y rsync
[root@backup ~]# vim /etc/rsyncd.conf
[root@backup ~]# more /etc/rsyncd.conf
uid = www
gid = www
use chroot = no
max connections = 200
timeout = 900
ignore error
fake super = yes
port = 873
read only = false
list = false
auth users = rsync_backup
secrets file = /etc/rsync.passwd
log file = /var/log/rsyncd.log

[backup]
        path = /backup

[data]
        path = /data

[root@backup ~]# groupadd -g 666 www
[root@backup ~]# useradd -u 666 -g 666 -M www
[root@backup ~]# more /etc/rsync.passwd
rsync_backup:Huawei@123.com
[root@backup ~]# chmod 600 /etc/rsync.passwd
[root@backup ~]# mkdir /data
[root@backup ~]# mkdir /backup
[root@backup ~]# chown -R www.www /backup /data
[root@backup ~]# systemctl restart rsyncd

# 任意客户端执行rsync推送数据测试服务端
[root@nfs ~]# rsync -avzP /etc/sysconfig rsync_backup@backup::backup
```

3. NFS的数据实时同步到BACKUP的/data目录

监控NFS服务器上的/data/目录，如果发生变化就触发动作，动作就是执行一次数据同步

```shell
# NFS角色配置
[root@nfs ~]# wget https://github.com/wsgzao/sersync/raw/master/sersync2.5.4_64bit_binary_stable_final.tar.gz
[root@nfs ~]# tar -xzf sersync2.5.4_64bit_binary_stable_final.tar.gz
[root@nfs ~]# mv GNU-Linux-x86/ /usr/local/sersync
[root@nfs ~]# vim /usr/local/sersync/confxml.xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<head version="2.5">
    <host hostip="localhost" port="8008"></host>
    <debug start="false"/>  # 调试模式，默认不开启
    <fileSystem xfs="true"/>    # 文件系统xfs，根据系统文件系统选择是否开启
    <filter start="false">  # 不同步的过滤文件
        <exclude expression="(.*)\.svn"></exclude>
        <exclude expression="(.*)\.gz"></exclude>
        <exclude expression="^info/*"></exclude>
        <exclude expression="^static/*"></exclude>
    </filter>
    <inotify>   # 通知接口，监控目录产生的变化类型
        <delete start="true"/>  # 删除
        <createFolder start="true"/>    # 创建目录
        <createFile start="true"/>
        <closeWrite start="true"/>      # 关闭写
        <moveFrom start="true"/>
        <moveTo start="true"/>
        <attrib start="true"/>         # 属性
        <modify start="true"/>         # 修改
    </inotify>

    <sersync>   # 发生变化时触发的动作
        <localpath watch="/data">     # 被监控的本地目录
            <remote ip="172.16.1.41" name="data"/>  # 触发动作，推送到远程备份服务器的IP和模块
            <!--<remote ip="192.168.8.39" name="tongbu"/>-->
            <!--<remote ip="192.168.8.40" name="tongbu"/>-->
        </localpath>
        <rsync>     # 推送到备份服务器使用的命令
            <commonParams params="-az"/>     # 命令使用的参数
            <auth start="true" users="rsync_backup" passwordfile="/etc/rsync.passwd"/>  # 启用用户验证，并指明用于验证的账户文件
            <userDefinedPort start="false" port="874"/><!-- port=874 -->    # 是否监听端口
            <timeout start="true" time="100"/><!-- timeout=100 -->     # 超时时间
            <ssh start="false"/>    # 是否使用ssh协议，使用rsync守护进程时不使用ssh协议
        </rsync>
        <failLog path="/tmp/rsync_fail_log.sh" timeToExecute="60"/><!--default every 60mins execute once-->     # 错误日志路径，默认60分钟执行一次同步
        <crontab start="false" schedule="600"><!--600mins-->    # 定时任务设置
            <crontabfilter start="false">   # 定时任务过滤
                <exclude expression="*.php"></exclude>
                <exclude expression="info/*"></exclude>
            </crontabfilter>
        </crontab>
        <plugin start="false" name="command"/>      # 是否启用插件
    </sersync>

    <plugin name="command">     # 插件名称
        <param prefix="/bin/sh" suffix="" ignoreError="true"/>  <!--prefix /opt/tongbu/mmm.sh suffix-->
        <filter start="false">
            <include expression="(.*)\.php"/>
            <include expression="(.*)\.sh"/>
        </filter>
    </plugin>

    <plugin name="socket">  # 插件名称
        <localpath watch="/opt/tongbu">
            <deshost ip="192.168.138.20" port="8009"/>
        </localpath>
    </plugin>
    <plugin name="refreshCDN">  # 插件名称
        <localpath watch="/data0/htdocs/cms.xoyo.com/site/">    # 监控本地目录的静态文件
            <cdninfo domainname="ccms.chinacache.com" port="80" username="xxxx" passwd="xxxx"/>     # 同步推送到CDN节点
            <sendurl base="http://pic.xoyo.com/cms"/>
            <regexurl regex="false" match="cms.xoyo.com/site([/a-zA-Z0-9]*).xoyo.com/images"/>
        </localpath>
    </plugin>
</head>

[root@nfs ~]# vim /etc/rsync.passwd
Huawei@123.com
[root@nfs ~]# chmod 600 /etc/rsync.passwd
[root@nfs ~]# yum install -y inotify-tools
[root@nfs ~]# /usr/local/sersync/sersync2 -h    # 查看sersync的帮助手册
[root@nfs ~]# /usr/local/sersync/sersync2 -dro /usr/local/sersync/confxml.xml    # 按配置文件启动守护进程

# 守护进程启动后会输出具体执行的命令和参数，复制该命令手动再执行一次可以观察命令是否有问题
[root@nfs data]# cd /data && rsync -az -R --delete ./  --timeout=100 rsync_backup@172.16.1.41::data --password-file=/etc/rsync.passwd >/dev/null 2>&1    # 手动执行测试命令，观察命令是否执行成功
```

注：如果有多个目录需要监控并实时同步，*不能直接在.xml文件内增加`<localpath watch="/data">`多个目录*，**只能复制.xml配置文件，修改`<localpath watch="/data">`目录后，使用sersync2命令再启动一次新的配置文件；使用sersync2命令启动守护进程时，不要多次执行重复的命令，每执行一次sersync2守护进程，都会启动一个进程，即便命令是重复的。由于sersync没有专门的管理服务，所以只能通过类似pkill的命令停止多余的sersync2守护进程**

如果执行sersync2守护进程后，再执行测试命令发现报错，直接修改xml配置文件后，再执行一次测试命令即可，无需重复执行sersync2命令；**NFS的sersync的虚拟用户账户文件，只要写虚拟用户的密码，不要用户名，否则执行sersync同步数据到BACKUP会失败**

#### 实现数据平滑迁移

迁移NFS数据到BACKUP服务器，并将后续数据直接指向BACKUP服务器，在这个过程中还需要保证业务不中断

[![sersync业务平滑迁移](https://s1.ax1x.com/2022/12/10/zfnsOK.png)](https://imgse.com/i/zfnsOK)

1. 首先NFS的数据全部实时同步到BACKUP，实现数据的迁移，保证业务迁移时不会因为数据差异大产生较大的影响
2. BACKUP需要运行NFS上一样的业务环境，例如nfs服务
3. 在WEB上实现切换nfs服务端，卸载NFS的/data目录，挂载BACKUP服务的/data目录

```shell
# BACKUP角色配置
[root@backup ~]# groupadd -g 666 www
[root@backup ~]# useradd -u 666 -g 666 www
[root@backup ~]# yum install -y nfs-utils
[root@backup ~]# more /etc/exports
/data/  172.16.1.0/24(rw,sync,all_squash,anonuid=666,anongid=666)
[root@backup ~]# systemctl start nfs-server.service
[root@backup ~]# systemctl start rpcbind
[root@backup ~]# firewall-cmd --add-service=nfs --permanent
[root@backup ~]# firewall-cmd --add-service=mountd --permanent
[root@backup ~]# firewall-cmd --add-service=rpc-bind --permanent
[root@backup ~]# firewall-cmd reload

# WEB角色配置
[root@web01 ~]# umount /var/www/html && mount -t nfs 172.16.1.41:/data /var/www/html
```

## 四、Rsync备份案例

实现此案例需要3个角色

| 角色  | 外网（NAT）    | 内网（LAN）      | 主机名 |
| ----- | -------------- | ---------------- | ------ |
| WEB   | eth0:10.0.0.7  | eth1:172.16.1.7  | web01  |
| NFS   | eth0:10.0.0.31 | eth1:172.16.1.31 | nfs    |
| Rsync | eth0:10.0.0.41 | eth1:172.16.1.41 | backup |

**客户端要求**

1. 客户端提前准备存放备份数据的目录，目录规则如下：`/backup/nfs_172.16.1.31_2022-08-03`
2. 客户端在本地打包备份数据（系统配置文件、应用配置等）拷贝至`/backup/nfs_172.16.1.31_2022-08-03`
3. 客户端最后将备份的数据进行推送至backup服务器
4. 客户端每天凌晨1点定时执行该脚本
5. 客户端服务器本地保留最近7天数据，避免浪费磁盘空间

**服务端要求**

1. 服务端安装rsync，用于接收客户端推送的备份数据
2. 服务端需要每天校验客户端推送的数据是否完整
3. 服务端需要每天校验的结果通知给管理员
4. 服务端仅保留6个月的备份数据，其余的全部删除

#### NFS客户端操作步骤

```shell
# 1. 创建对应备份目录
mkdir -p /server/script	# 用于存放定时任务执行的脚本文件

# 2. 编辑脚本文件
vim /server/script/client_push_date.sh
#!/bin/bash
# filename: client_push_data.sh
# drscription: 用于数据上传

# 1.定义变量
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
Src=/backup
Host=$(hostname)
Addr=$(ifconfig eth1 | awk 'NR==2 {print $2}')
Date=$(date +%F)
Dest=${Host}_${Addr}_${Date}

# 2.创建目录
[ -d $Src/$Dest ] || mkdir -p $Src/$Dest	# 判断目录是否已存在，不存在时执行创建操作

# 3.备份文件
# 进入根目录并将所有要打包的绝对路径的根符号删除，是为了避免执行tar命令时提示去除根目录的信息
cd / && \
[ -f $Src/$Dest/system_file.tar.gz ] || tar -czf $Src/$Dest/system_file.tar.gz etc/fstab etc/passwd && \
[ -f $Src/$Dest/other_file.tar.gz ] || tar -czf $Src/$Dest/other_file.tar.gz var/spool/cron/ server/script/
# 每一次重新打包都会重新生成MD5校验码，所以执行文件判断

# 4.生成MD5校验码
# 如果不执行文件判断，每执行一次脚本都会再执行生成校验码操作，但只要压缩包文件属性未改变，生成的校验码也不会变
[ -f $Src/$Dest/md5_flag_${Date} ] || md5sum $Src/$Dest/*.tar.gz > $Src/$Dest/md5_flag_${Date}

# 5.本地推送到服务器
export RSYNC_PASSWORD=Huawei@123.com
rsync -avz $Src/$Dest rsync_backup@backup::backup

# 6.保留本地7天数据
find $Src/ -mtime +7 -type d  | xargs rm -rf
```

删除脚本中判断语句时，对脚本整体没有太大的影响，但反复执行脚本会出现一些系统提示，这对脚本实现的功能并没有影响，因为该脚本本身就是每天只执行一次。关于脚本的最后一个步骤，保留7天文件备份，可以通过修改日期的方式进行验证：

示例：手动验证日志文件是否正常保留

```shell
# 1.使用循环生成一个月的数据
for i in {1..30}; do date -s "2018/12/$i"; sh /server/script/client_push_data.sh; done

# 2.筛选出最近7天的数据
find /backup/ -mtime +7 -type d  | xargs rm -rf
# 反选7天以前所有的数据删除，注意，当前系统时间应该是2018/12/30，当天以及7天前的数据会被保留
```

最后，通过定时任务循环执行此脚本，注：定时任务设定成每分钟执行一次是为了测试定时任务是否能够正常执行。至此，客户端操作通过脚本已经全部实现

```shell
crontab -e
#crond02: rsync客户端推送
*/1 * * * *     /usr/bin/sh /server/script/client_push_data.sh
```

#### BACKUP服务端操作

```shell
# 1. 创建对应备份目录
mkdir /server/scripts -p

# 2. 编辑脚本文件
vim /server/scripts/check_client_data.sh
#!/bin/bash
# filename: check_client_data.sh
# drscription: 用于检查客户端备份数据

# 1.定义变量
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
Src=/backup
Date=$(date +%F)

# 2.使用MD5进行校验，并保存校验的结果
md5sum -c $Src/*_${Date}/md5_flag_${Date} > $Src/result_${Date}

# 3. 将保存的MD5校验结果发送给管理员
mail -s "Rsync MD5校验结果 ${Date}" 1234567890@qq.com < $Src/result_${Date}

# 4. 保留最近180天的数据
find $Src/ -mtime +180 -type d  | xargs rm -rf
```

示例：设置服务端定时任务

```shell
#crond01: Rsync数据同步校验
*/2 * * * *     /usr/bin/sh /server/scripts/check_client_data.sh
```

在使用邮箱发送邮件之前，需要先对邮箱进行配置：

```shell
yum install -y mailx
set from=1234567890@qq.com	# 填写发件人QQ邮箱
set smtp=smtps://smtp.qq.com:465
set smtp-auth-user=1234567890@qq.com
set smtp-auth-password=授权码	# 获取邮箱生成的客户端授权码
set smtp-auth=login
set ssl-verify=ignore
set nss-config-dir=/etc/pki/nssdb/

mail -s "QQ邮箱测试发送自己" 1015792427@qq.com < /backup/result_2022-11-24	# 测试能否手动发送邮件
```

最后进行整体测试：
	1. 删除客户端/backup/目录下的所有数据
	2. 删除服务端/backup/目录下的所有数据
	3. 通过修改时间测试定时任务是否正常执行
	4. 查看邮件和定时任务日志查看定时任务是否执行成功

小结：

1. 通过邮箱测试必须准备一个在线邮箱，还需要对邮箱进行开启SMTP功能获取授权码
2. 编写的脚本即便手动执行能够成功，但定时任务还是会因为环境变量的不同导致找不到命令，可以通过两种方式解决：脚本文件中所有命令都是用绝对路径 或 在脚本开头重新声明一下PATH
3. 脚本中大部分的条件判断语句都是可以删除的，不影响脚本执行的结果，但会出现很多不必要的系统提示，即便不删除这些判断语句，也会在邮箱中产生一些系统提示

#### 新增WEB客户端操作

新增WEB01客户端，模拟多台客户端同时向备份服务器推送备份数据

```shell
# 1. 初始化环境
yum install -y vim bash-completion rsync net-tools
vim /etc/hosts
172.16.1.7	web01
172.16.1.31	nfs
172.16.1.41	backup

rsync -avz root@nfs:/server /

# 2. 手动执行脚本测试
sh /server/script/client_push_data.sh
# 查看backup服务器是否已经存在多个客户端的MD5校验
```

#### Rsync备份思路

1. 定位需要备份的文件或目录
2. 规划用于保存备份数据的结构目录
3. 对备份数据进行打包压缩便于统一管理，并添加标记信息
4. 通过虚拟用户传输到备份服务器
5. 服务端对所有客户端的备份数据进行规划管理和数据验证
6. 服务端对备份数据的验证结果定时发送给管理员

#### Rsync小结

Rsync能够实现远程备份，但Rsync与备份之间并没有必然的关系，Rsync只是用于备份的某一个工具

再者，关于虚拟用户`rsync_backup`与进程用户`rsync`的区别，`rsync_bakcup`用户仅用于给客户端提供远程连接，`rsync`才能够决定是否往服务器中写入客户端推送的数据

Rsync服务故障解决思路

1. 测试备份服务器地址是否能够正常通信
2. 测试服务端的Rsync服务的端口是否正常
3. 检查客户端虚拟用户账号密码是否正确，以及密码文件的权限是否600
4. 检查服务端的数据存放目录属性、权限，以及该目录是否存在