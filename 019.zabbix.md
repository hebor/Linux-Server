# zabbix

## 应用监控

应用监控的操作流程基本类似，下述以nginx、php为例。首先需要明确的一点是，任何应用服务都一定会有应用自身的状态信息，例如nginx的stub_status模块、php的status页面，只要应用服务自身的status信息能够输出，那就一定能对其监控

### nginx应用监控

监控思路：

1. 端口是否正常
2. stub_status的7种状态
    1. 安装nginx
    2. 开启stub_status
    3. 获取每个状态的指标
    4. 将获取到的状态做成监控项
    5. 在web界面创建模板->创建监控项->创建触发器

关于nginx的安装与stub_status模块的使用此处不提及，仅以zabbix的操作为重点，但stub_status页面完全是可以仅允许本地回环地址访问的，因为zabbix agent的系统取值都是在本地主机执行，数据采集完成后发送给zabbix server，所以无需考虑zabbix server如何获取到nginx的状态信息的问题

```shell
# 访问nginx状态页面
[root@web01 ~]# curl http://127.0.0.1/status
Active connections: 1 
server accepts handled requests
 7 7 7 
Reading: 0 Writing: 1 Waiting: 0

# 获取每个状态的指标（此处以活跃链接状态为例）
[root@web01 ~]# curl -s http://127.0.0.1/status | awk '/^Active/ {print $NF}'
    accepts curl -s http://127.0.0.1/status | awk 'NR==3 {print $1}'
    handled curl -s http://127.0.0.1/status | awk 'NR==3 {print $2}'
    requests    curl -s http://127.0.0.1/status | awk 'NR==3 {print $3}'

# agent自定义nginx的监控项
[root@web01 ~]# vim /etc/zabbix/zabbix_agentd.d/nginx.conf
UserParameter=nginx.active,curl -s http://127.0.0.1/status | awk '/^Active/ {print $NF}'
UserParameter=nginx.accepts,curl -s http://127.0.0.1/status | awk 'NR==3 {print $1}'
UserParameter=nginx.handled,curl -s http://127.0.0.1/status | awk 'NR==3 {print $2}'
UserParameter=nginx.requests,curl -s http://127.0.0.1/status | awk 'NR==3 {print $3}'

# agent测试自定义的监控项是否能够正常取值
[root@web01 ~]# zabbix_agentd -p
[root@web01 ~]# systemctl restart zabbix-agent.service  # 能够正常取值后即重启服务，否则服务端获取不到值

# server测试能否获取到agent的自定义监控项的值
[root@zabbix-server ~]# zabbix_get -s 172.16.1.7 -k nginx.active
```

#### zabbix server创建nginx应用监控

确认agent客户端和server服务端能够获取到自定义的监控项的值之后，需要在zabbix前端再定义监控模板应用

`配置`->`模板`->`创建模板`->填写新建模板信息->搜索新建的模板->`监控项`->`创建监控项`->编辑监控项信息

![nginx应用监控-1](https://www.z4a.net/images/2023/04/12/nginx-1.png)
![nginx应用监控-2](https://www.z4a.net/images/2023/04/12/nginx-2.png)
![nginx应用监控-3](https://www.z4a.net/images/2023/04/12/nginx-3.png)


第一个监控项编辑完成后，后续的几个监控项直接克隆修改即可。可基于监控项设置触发器：`触发器`->`创建触发器`->填写触发器信息；此处以nginx的监听端口为例创建触发器，检测nginx的存活状态

![nginx应用监控-4](https://www.z4a.net/images/2023/04/12/nginx-4.png)
![nginx应用监控-5](https://www.z4a.net/images/2023/04/12/nginx-5.png)

每个监控项都能够设置*查看值*，这个查看值用于*触发器*的信号接收，当查看值达到设置的阈值后触发信号，执行触发器设置的动作。模板创建完成后仍跟主机没有任何关联，所以还需要手动指定为主机添加模板：`配置`->`主机`->点击目标主机->`模板`->选择链接新模板->勾选新建的nginx状态模板->`更新`

### php-fpm应用监控

1. 启用php-fpm的状态
    
    ```shell
    # 1.启用php-fpm状态页
    [root@web01 ~]# vim /usr/local/php/etc/php-fpm.d/www.conf
    pm.status_path = /phpstatus

    # 2.配置nginx展示php-fpm状态页
    [root@web01 ~]# vim /usr/local/nginx/conf.d/stub.example.com.conf
    location /phpstatus {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
    ```

2. 根据php-fpm状态，提取需要的指标

    ```shell
    # i.编辑zabbix配置文件
    [root@web01 ~]# vim /etc/zabbix/zabbix_agentd.d/phpfpm_status.conf
    UserParameter=phpfpm_status[*],/bin/bash /etc/zabbix/zabbix_agentd.d/phpfpm_status.sh "$1"

    # ii.编辑脚本文件
    [root@web01 ~]# vim /etc/zabbix/zabbix_agentd.d/phpfpm_status.sh
    !#/bin/bash

    # 1.定义变量名
    php_status_file=/tmp/php_status.log
    php_status_comm=$1

    # 2.将php状态结果输出到文件中，减少重复查看php状态时，curl请求带来过多的连接和请求数
    /usr/bin/curl -s "http://127.0.0.1/phpstatus" > $php_status_file

    # 3.提取监控指标值
    case $phpfpm_status_comm in
        accepted_conn)
            awk '/^accepted conn:/ {print $NF}' $phpfpm_status_file
            ;;
        listen_queue)
            awk '/^listen queue:/ {print $NF}' $phpfpm_status_file
            ;;
        max_listen_queue)
            awk '/^max listen queue:/ {print $NF}' $phpfpm_status_file
            ;;
        idle_processes)
            awk '/^idle processes:/ {print $NF}' $phpfpm_status_file
            ;;
        active_processes)
            awk '/^active processes:/ {print $NF}' $phpfpm_status_file
            ;;
        total_processes)
            awk '/^total processes:/ {print $NF}' $phpfpm_status_file
            ;;
        *)
            echo $"USAGE:$phpfpm_status_comm {accepted_conn|listen_queue|max_listen_queue|idle_processes|active_processes|total_p
    rocesses}"
    esac
    [root@web01 ~]# chmod +x /etc/zabbix/zabbix_agentd.d/phpfpm_status.sh   # 不加执行权限时，zabbix执行会出错
    [root@web01 ~]# chown zabbix.zabbix /tmp/php_status.log 
    ```

3. 将系统提取到的指标配置为监控项

    ```shell
    # 1.agent测试自定义监控文件是否生效
    [root@web01 ~]# zabbix_agentd -p    # 查看监控项一定会出现脚本参数使用提示，因为默认测试是没有给脚本带上参数的
    [root@web01 ~]# systemctl restart zabbix-agent.service

    # 2.server测试自定义监控文件是否生效
    [root@zabbix-server ~]# zabbix_get -s 172.16.1.7 -k phpfpm_status[accepted_conn]
    ```

4. zabbix web界面创建模板

    创建模板的过程与nginx的过程一样，只不过监控项的键值条件需要一些修改

![php应用监控-1](https://www.z4a.net/images/2023/04/15/php-1.png)
![php应用监控-2](https://www.z4a.net/images/2023/04/15/php-2.png)
![php应用监控-3](https://www.z4a.net/images/2023/04/15/php-3.png)

在php优化章节中有设置过php的初始进程数是32，监测php活跃进程数的触发器，当php的活跃连接数平均3分钟数超过300个活跃链接时报警

#### 以脚本实现监控

```shell
[root@web02 ~]# vim /etc/zabbix/zabbix_agentd.d/phpfpm_status.conf
UserParameter=phpfpm_status[*],/bin/bash /etc/zabbix/scripts/phpfpm_status.sh "$1"

cat /etc/zabbix/scripts/phpfpm_status.sh
# 1.定义变量
PHPFPM_COMMAND=$1
PHPFPM_PORT=80

# 2.声明函数
accepted_conn(){
    /usr/bin/curl -s "http://127.0.0.1:"$PHPFPM_PORT"/phpstatus" | awk '/^accepted conn:/ {print $NF}'
}

# 3.条件判断
case $PHPFPM_COMMAND in
    accepted_conn)
        accepted_conn;
        ;;
esac
```

phpfpm_status.conf文件的配置应从左往右看，首先假设整个脚本配置已经完成了，那么在zabbix web创建监控项时，需要输入一个监控指标。此时假设在zabbi web中以`accepted conn`作为指标，则`accepted conn`这个值就会传到phpfpm_status[accepted conn]，继而再传到`$1`。`$1`的值就代表要传给脚本的值，也就是说`accepted conn`将会作为参数传给脚本，此时就需要查看脚本内容具体实现什么功能

脚本中需要接收的参数是`accepted_conn`而不是`accepted conn`，这是因为从zabbix web输入监控指标的键值时，`accepted_conn`下划线的方式更容易被大众接受。示例脚本中只给出了`accepted conn`的函数和判断，存在多个条件时多添加几个函数和对应的判断即可

### redis应用监控

1. db01节点安装agent

    ```shell
    [root@db01 ~]# rpm -ivh https://mirrors.aliyun.com/zabbix/zabbix/5.0/rhel/7/x86_64/zabbix-agent-5.0.33-1.el7.x86_64.rpm?spm=a2c6h.25603864.0.0.497f2e2fCR7vTV
    [root@db01 ~]# vim /etc/zabbix/zabbix_agentd.conf
    Server=172.16.1.71
    [root@db01 ~]# systemctl start zabbix-agent.service 
    [root@db01 ~]# systemctl enable zabbix-agent.service
    ```

2. 获取redis的状态

```shell
[root@db01 ~]# redis-cli -h 127.0.0.1 -p 6379 info
```

3. 根据redis状态配置监控指标

```shell
[root@db01 ~]# vim /etc/zabbix/zabbix_agentd.d/redis.conf
UserParameter=redis.connected_clients,redis-cli -h 127.0.0.1 -p 6379 info | grep "connected_clients" | awk -F ":" '{print $NF}'
[root@db01 ~]# zabbix_agentd -p     # 本地监测自定义监控项是否生效
[root@db01 ~]# systemctl restart zabbix-agent.service
```

4. server检测是否能够获取到监控项

```shell
[root@zabbix-server ~]# zabbix_get -s 172.16.1.51 -k redis.connected_clients
```

5. zabbix web新建模板配置监控项

## web监测

zabbix可以对多个网站进行可用性监控，通过设置一些响应条件来验证网站是否能够对外提供正常访问，**如果要使用Web监控，必须编译安装时加入cURL（libcurl）库支持**，因为zabbix的网站验证基于curl命令，使用curl进行监测的站点不能有验证码

### web场景监测概述

1. web网站中什么时动态网站，什么是静态网站

静态网站：纯静态网站就是服务器的源代码和客户端的源代码一致
动态网站：例如`<?php phpinfo() ?>`，每次用户请求的内容都是在内存中动态生成的，动态网站支持账号验证、用户交互

2. 用户访问web时，session和cookie是如何工作的

用户首次访问动态网站时不会携带cookie信息，服务端响应客户端请求时会为客户端浏览器分配一个唯一的session ID，客户端会将该session ID存储至浏览器的cookie中

3. 模拟登录网站

```shell
# 使用curl命令模拟登录zabbix服务器，获取cookie并将cookie保存至本地cook文件中
[root@db01 ~]# curl -L -c cook -b cook 'http://10.0.0.71/index.php'
    -L：指定http请求跟随服务器的重定向
    -c：保存cookie到cook文件
    -b：向服务器发送cookie
# 再次访问携带cook信息，同时使用-d参数携带账户密码，模拟登录
[root@db01 ~]# curl -L -c cook -b cook -d 'name=Admin&password=zabbix&autologin=1&enter=Sign+in' 'http://10.0.0.71/index.php'
    -d：发送POST请求的数据体
```

#### web监测示例

注：示例中只是针对zabbix server主机制作一个web监测项演示，严谨一些一定是针对模板制作web监测项

`配置`->`主机`->`web监测`->`创建web场景`

![web监测示例-1](https://www.z4a.net/images/2023/04/16/web-1.png)

检测间隔时间是web监测的所有步骤的执行时间总和，如果某些步骤的执行时间过长导致总体步骤的执行时间超时，则zabbix会重新开始执行步骤，这将会导致web监测返回警告

![web监测示例-2](https://www.z4a.net/images/2023/04/16/web-2.png)

监测的站点是动态站点时可能会在URL地址上附带一些属性参数，通过`分析`可以将这些属性参数解析成查询字段

![web监测示例-3](https://www.z4a.net/images/2023/04/16/web-3.png)

### web监测zabbix server页面

`配置`->`主机`->`web监测`->`创建web场景`

![wen监测zabbix_server](https://www.z4a.net/images/2023/04/17/wenzabbix_server.png)

此处只是展示了一下如何在创建web监测项的时候添加变量，即便此处未添加变量，在添加步骤的过程中仍可以再添加变量

1. 检查zabbix web首页是否能否访问成功

![访问zabbix首页](https://www.z4a.net/images/2023/04/17/zabbix.png)

2. 登录zabbix

    注意，Zabbix前端在登录时使用JavaScript重定向，**因此首先必须登录，只有在下一步的步骤中，才能检查登录功能**。此外，登录步骤必须使用完整的URL以获取index.php文件。还要注意我们如何使用正则表达式的变量语法获取`{sid}`变量（会话 ID）的内容：`regex:name="csrf-token" content="([0-9a-z]{16})"`，在退出zabbix需要向zabbix server端声明自身的会话ID

![登录zabbix](https://www.z4a.net/images/2023/04/17/zabbix35be9391a983ace0.png)

3. 检查登录是否成功

针对zabbix web的登录校验，需要检查一个仅在登录后可见的字符串，例如Administration（管理）。根据zabbix web设置的语言的不同，这个校验字符串也需要即时进行修改，否则可能出现提取不到字符串而告警的情况，例如原本使用英文语言，转变为中文后没有即时修改校验字符串

![检查zabbix是否登录成功](https://www.z4a.net/images/2023/04/17/zabbix7a357b03f46eb395.png)

4. zabbix登出

zabbix server会将用户的cookie信息写入数据库，而每一次的监测访问都要登陆zabbix web，这样会对数据库造成较大开销，zabbix数据库将被大量打开的会话记录占用资源，所以每次监测后都需要退出登录

![zabbix登出字段](https://www.z4a.net/images/2023/04/17/zabbixe01f81cfa432f73d.png)

![zabbix登出](https://www.z4a.net/images/2023/04/17/zabbix7faf235b80c40be1.png)

5. 检查zabbix是否登出成功

![检查zabbix是否登出成功](https://www.z4a.net/images/2023/04/17/zabbix1cc65addf883679e.png)

#### web监测wordpress

待补充。整个过程步骤与zabbix server基本一致，仍需注意退出时的唯一ID号

## 自动化监控

### 自动发现

自动发现别称网络发现，Zabbix定期检测网络发现规则中定义的IP范围，并为每个规则单独配置检查的频次，一个发现规则始终由单一发现进程处理，IP范围主机不会被分拆到多个发现进程处理。自动发现规则的更新间隔时长必须要特别注意，如果网段较大的情况下需要给足充分的时间，由于自动发现是zabbix server主动去扫描网段，在时间不够的情况下会导致zabbix server的discovery进程跑满，且无法扫描到全部地址，这也是自动发现的弊端，它可能会导致zabbix server端性能下降并出现报警

对于zabbix server的discovery进程性能的问题，除了修改更新间隔时间，也可以通过修改zabbix配置文件进行优化

```shell
[root@zabbix-server ~]# vim /etc/zabbix/zabbix_server.conf
StartDiscoverers=20
```

`配置`->`自动发现`->`创建发现规则`->编辑自动发现规则

![创建自动发现规则](https://www.z4a.net/images/2023/04/17/76c9ce053b353e851ff010287b75f267.png)

定义zabbix网络扫描到主机后需要执行什么动作：`配置`->`动作`->切换`发现动作`->`创建动作`->编辑动作信息

![自动发现动作-1](https://www.z4a.net/images/2023/04/17/-1.png)

![自动发现操作](https://www.z4a.net/images/2023/04/17/7d2db21135c2434566b0bc919940b6ea.png)

```
自动发现主机IP：{DISCOVERY.DEVICE.IPAADDRESS}
客户端名称：{DISCOVERY.SERVICE.NAME}
客户端端口：{DISCOVERY.SERVICE.PORT}
客户端状态：{DISCOVERY.SERVICE.STATUS}
```

配置自动发现动作后删除原有的web主机，等待zabbix自动发现

[zabbix内建宏](https://www.zabbix.com/documentation/5.0/zh/manual/appendix/macros/supported_by_location)

### 自动注册

自动注册需要修改agent端配置文件的3个选项，如果没有在zabbix_agentd.conf中特别定义Hostname, 则服务器将使用agent的系统主机名命名主机，修改配置文件后需要重启agent使其生效

```shell
[root@web01 ~]# vim /etc/zabbix/zabbix_agentd.conf
Server=172.16.1.71
ServerActive=172.16.1.71    # 开启自动注册
Hostname=web01
[root@web01 ~]# systemctl restart zabbix-agent.service
```

zabbix server无需进行配置，在web上新建自动注册的动作即可。如果想要区分监控Linux主机和Windows主机，需要使用主机元数据条件进行区分，且还需要在agent上添加一些参数，此处所有主机都是Linux，所以不做展示。

[自动发现示例](https://www.zabbix.com/documentation/5.0/zh/manual/discovery/auto_registration)

![自动注册-1](https://www.z4a.net/images/2023/04/17/-1a0b576e0eafff165.png)
![自动注册-2](https://www.z4a.net/images/2023/04/17/227b08cab82f94bea.png)

```shell
Auto registration:{HOST.HOST}
Host name: {HOST.HOST}
Host IP: {HOST.IP}
Agent port: {HOST.PORT}
```

![自动注册-3](https://www.z4a.net/images/2023/04/17/-3.png)

### zabbix主被模式区别

1. 被动模式：zabbix server轮询监测zabbix agent，应用在网络发现规则
2. 主动模式：zabbix agent主动上报数据到zabbix server，应用在自动注册规则；当Queue存在大量延迟监控项或当监控主机超过300+时，建议使用主动模式

zabbix默认是被动模式，如果被动模式需要获取100个监控项的值，则Server需要向Agent获取100次数据，每次获取一个监控项的值，如果在大量监控项的场景下，这种方式会导致Server性能下降；如果主动模式需要获取100个监控项的值，Server会将需要的监控项的值生成一个清单发送给Agent，Agent采集完成后一次将所有数据发送给Server

![zabbix被动监测模式](https://www.z4a.net/images/2023/04/17/zabbix735422e85c9f30f3.png)

#### 调整zabbix为主动模式

1. 修改agent配置文件（在自动注册中已经调整过agent的配置）
2. 全克隆zabbix模板为Active

`配置`->`主机`->点击模板->`全克隆`->标记模板名称->`添加`

![zabbix主动监测模式-1](https://www.z4a.net/images/2023/04/17/zabbix-1.png)
![zabbix主动监测模式-2](https://www.z4a.net/images/2023/04/17/zabbix-2.png)
![zabbix主动监测模式-3](https://www.z4a.net/images/2023/04/17/zabbix-3.png)

`监控项`->全选监控项->`批量更新`->勾选类型->选择zabbix客户端(主动式)->`更新`

![zabbix主动监测模式-4](https://www.z4a.net/images/2023/04/17/zabbix-40c0d95ea272edf56.png)
![zabbix主动监测模式-5](https://www.z4a.net/images/2023/04/17/zabbix-5.png)

注：以上这种直接批量修改zabbix默认模板的方式在5.0已经不适用了，以`Template OS Linux by Zabbix agent`模板为例，这个模板下的所有监控项几乎都是链接模板，直接修改此模板的类型为zabbix客户端(主动式)会发现，zabbix web提示监控项已修改成功，但再查看该模板的监控项类型时，类型仍是zabbix客户端，没有发生改变。如果一定要修改zabbix的模板模板，只能究其根源找到原始的监控项，并更新其为主动式才会生效

#### 自动化监控小结

网络发现（被动模式）

1. 扫描效率低
2. 容易漏扫描
3. 如果因为某一台主机的未知原因，导致在该主机上花费时间较长，可能导致后面的主机没有扫描到就要开始重新扫描

自动注册（主动模式）

1. agent自动注册到server，根据不同的主机名称关联不同的模板
2. 效率高

主动模式和被动模式的区别（监控项）

被动模式由server向agent轮询查询监控项的值，如果有100个监控项，则需要100此数据响应
主动模式由agent主动向server发送所有数据，如果有100个监控项，agent会一次返回给server